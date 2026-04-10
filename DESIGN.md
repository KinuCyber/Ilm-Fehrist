# Ilm Fehrist - System Design

This document is the single source of truth for database schema and core logic.
Read PHILOSOPHY.md for the reasoning behind these decisions.

---

## 1. Database Setup

```sql
-- Enable foreign key enforcement (off by default in SQLite)
PRAGMA foreign_keys = ON;

-- WAL mode for better concurrent read performance
PRAGMA journal_mode = WAL;
```

Table creation order matters due to foreign key dependencies:

```
items
|-- book_meta
|   |-- reading_position
|   `-- bookmarks
|-- course_meta
|-- product_meta
|   `-- product_versions
|-- project_meta
|-- objectives
|-- edges
|-- tags
`-- item_tags
```

---

## 2. Items

The base table for all four entity types. Every node in the graph is a row here.

```sql
CREATE TABLE items (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    type        TEXT NOT NULL CHECK(type IN ('book', 'course', 'product', 'project')),
    title       TEXT NOT NULL,
    status      TEXT NOT NULL DEFAULT 'inactive',
    progress    REAL NOT NULL DEFAULT 0.0 CHECK(progress BETWEEN 0.0 AND 1.0),
    added_at    DATETIME DEFAULT CURRENT_TIMESTAMP,

    CHECK(
        -- Books
        (type = 'book'    AND status IN ('unread', 'reading', 'completed', 'missing')) OR
        -- Courses
        (type = 'course'  AND status IN ('inactive', 'active', 'completed')) OR
        -- Products
        (type = 'product' AND status IN ('inactive', 'active', 'completed', 'deprecated')) OR
        -- Projects
        (type = 'project' AND status IN ('inactive', 'active', 'completed', 'abandoned'))
    )
);
```

**Operations**

```python
def create_item(conn, type, title, status=None):
    """
    Insert a new item. Status defaults per type if not provided.
    Returns the new item's id.
    """
    defaults = {
        'book':    'unread',
        'course':  'inactive',
        'product': 'inactive',
        'project': 'inactive',
    }
    status = status or defaults[type]

    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO items (type, title, status)
        VALUES (?, ?, ?)
    """, (type, title, status))
    conn.commit()
    return cursor.lastrowid


def update_item_status(conn, item_id, status):
    """
    Manually update an item's status.
    Validity of status for the item's type is enforced by the DB CHECK constraint.
    """
    cursor = conn.cursor()
    cursor.execute(
        "UPDATE items SET status = ? WHERE id = ?",
        (status, item_id)
    )
    conn.commit()
```

---

## 3. Books

### Schema

```sql
CREATE TABLE book_meta (
    item_id         INTEGER PRIMARY KEY REFERENCES items(id) ON DELETE CASCADE,
    filename        TEXT,
    path            TEXT UNIQUE,        -- relative to library root
    author          TEXT,
    year            INTEGER,
    file_hash       TEXT,               -- MD5 of first 64KB
    total_pages     INTEGER NOT NULL DEFAULT 1,
    CHECK(total_pages >= 1)
);

-- Single source of truth for reading progress.
-- One row per book, always. Never append - always upsert.
CREATE TABLE reading_position (
    book_id         INTEGER PRIMARY KEY REFERENCES items(id) ON DELETE CASCADE,
    current_page    INTEGER NOT NULL DEFAULT 0,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    CHECK(current_page >= 0)
);

-- Pure annotations. No effect on progress.
CREATE TABLE bookmarks (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    book_id     INTEGER NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    page        INTEGER NOT NULL,
    label       TEXT,
    note        TEXT,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### Operations

```python
def create_book(conn, title, path, filename, total_pages,
                author=None, year=None, file_hash=None):
    """
    Create a book item + its metadata + an initial reading position.
    Returns the new item's id.
    """
    item_id = create_item(conn, 'book', title, status='unread')

    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO book_meta (item_id, filename, path, author, year, file_hash, total_pages)
        VALUES (?, ?, ?, ?, ?, ?, ?)
    """, (item_id, filename, path, author, year, file_hash, total_pages))

    cursor.execute("""
        INSERT INTO reading_position (book_id, current_page)
        VALUES (?, 0)
    """, (item_id,))

    conn.commit()
    return item_id


def update_reading_position(conn, book_id, page):
    """
    Advance the reading position for a book.
    Forward-only: raises if page is behind current position.
    Triggers progress recomputation upward through the graph.
    """
    cursor = conn.cursor()

    cursor.execute("""
        SELECT bm.total_pages, rp.current_page
        FROM book_meta bm
        JOIN reading_position rp ON rp.book_id = bm.item_id
        WHERE bm.item_id = ?
    """, (book_id,))
    row = cursor.fetchone()
    if not row:
        raise ValueError(f"No book found with id {book_id}")

    total_pages, current_page = row

    if page < current_page:
        raise ValueError(
            f"Cannot move reading position backwards "
            f"(current: {current_page}, given: {page})"
        )
    if page > total_pages:
        raise ValueError(
            f"Page {page} exceeds total pages ({total_pages})"
        )

    # Upsert — one row per book, always
    cursor.execute("""
        INSERT INTO reading_position (book_id, current_page, updated_at)
        VALUES (?, ?, CURRENT_TIMESTAMP)
        ON CONFLICT(book_id) DO UPDATE SET
            current_page = excluded.current_page,
            updated_at   = excluded.updated_at
    """, (book_id, page))

    # Update book's progress directly from pages
    progress = page / total_pages
    cursor.execute(
        "UPDATE items SET progress = ? WHERE id = ?",
        (progress, book_id)
    )

    # Update status based on progress
    if progress == 0.0:
        status = 'unread'
    elif progress < 1.0:
        status = 'reading'
    else:
        status = 'completed'
    cursor.execute(
        "UPDATE items SET status = ? WHERE id = ?",
        (status, book_id)
    )

    conn.commit()

    # Propagate upward through all ancestor items
    update_progress_upward(conn, book_id)


def add_bookmark(conn, book_id, page, label=None, note=None):
    """
    Add a bookmark annotation to a book.
    Has absolutely no effect on progress or reading position.
    """
    cursor = conn.cursor()

    cursor.execute(
        "SELECT total_pages FROM book_meta WHERE item_id = ?", (book_id,)
    )
    row = cursor.fetchone()
    if not row:
        raise ValueError(f"No book found with id {book_id}")

    total_pages = row[0]
    if not (0 <= page <= total_pages):
        raise ValueError(f"Page {page} out of range (0–{total_pages})")

    cursor.execute("""
        INSERT INTO bookmarks (book_id, page, label, note)
        VALUES (?, ?, ?, ?)
    """, (book_id, page, label, note))
    conn.commit()


def get_bookmarks(conn, book_id):
    """
    Return all bookmarks for a book, annotated with distance
    from current reading position. Negative = already passed.
    """
    cursor = conn.cursor()
    cursor.execute("""
        SELECT
            b.id,
            b.page,
            b.label,
            b.note,
            b.page - rp.current_page AS pages_away,
            b.created_at
        FROM bookmarks b
        JOIN reading_position rp ON rp.book_id = b.book_id
        WHERE b.book_id = ?
        ORDER BY b.page
    """, (book_id,))
    return cursor.fetchall()
```

---

## 4. Courses

```sql
CREATE TABLE course_meta (
    item_id     INTEGER PRIMARY KEY REFERENCES items(id) ON DELETE CASCADE,
    source      TEXT    -- 'MIT OCW', 'self-study', 'Coursera', etc.
);
```

```python
def create_course(conn, title, source=None):
    """Create a course item + its metadata. Returns new item id."""
    item_id = create_item(conn, 'course', title)

    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO course_meta (item_id, source) VALUES (?, ?)",
        (item_id, source)
    )
    conn.commit()
    return item_id
```

Courses have no progression unit beyond Books and Objectives.
Progress is entirely derived - never set directly on a Course.

---

## 5. Products

```sql
CREATE TABLE product_meta (
    item_id         INTEGER PRIMARY KEY REFERENCES items(id) ON DELETE CASCADE,
    current_version TEXT NOT NULL DEFAULT 'v0.1'
);

CREATE TABLE product_versions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id      INTEGER NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    version         TEXT NOT NULL,
    planned_date    DATE,           -- nullable until deadlines are needed
    released_date   DATE,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(product_id, version)
);
```

```python
def create_product(conn, title, initial_version='v0.1'):
    """Create a product item + metadata + initial version record. Returns new item id."""
    item_id = create_item(conn, 'product', title)

    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO product_meta (item_id, current_version) VALUES (?, ?)",
        (item_id, initial_version)
    )
    cursor.execute("""
        INSERT INTO product_versions (product_id, version)
        VALUES (?, ?)
    """, (item_id, initial_version))
    conn.commit()
    return item_id


def release_product_version(conn, product_id, new_version, planned_date=None):
    """
    Mark current version as released (set released_date to today),
    then begin a new version.
    """
    cursor = conn.cursor()

    # Close out current version
    cursor.execute("""
        UPDATE product_versions
        SET released_date = DATE('now')
        WHERE product_id = ? AND released_date IS NULL
    """, (product_id,))

    # Record new version
    cursor.execute("""
        INSERT INTO product_versions (product_id, version, planned_date)
        VALUES (?, ?, ?)
    """, (product_id, new_version, planned_date))

    cursor.execute("""
        UPDATE product_meta SET current_version = ? WHERE item_id = ?
    """, (new_version, product_id))

    conn.commit()
```

---

## 6. Projects

```sql
CREATE TABLE project_meta (
    item_id     INTEGER PRIMARY KEY REFERENCES items(id) ON DELETE CASCADE,
    deadline    DATE            -- nullable; projects are milestone-driven by default
);
```

```python
def create_project(conn, title, deadline=None):
    """Create a project item + metadata. Returns new item id."""
    item_id = create_item(conn, 'project', title)

    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO project_meta (item_id, deadline) VALUES (?, ?)",
        (item_id, deadline)
    )
    conn.commit()
    return item_id
```

---

## 7. Objectives

Objectives are the shared progression unit for Courses, Products, and Projects.
They are recursive: an Objective can contain sub-Objectives.

### Schema

```sql
CREATE TABLE objectives (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    item_id             INTEGER REFERENCES items(id) ON DELETE CASCADE,
    parent_objective_id INTEGER REFERENCES objectives(id) ON DELETE CASCADE,
    label               TEXT NOT NULL,
    progress            REAL NOT NULL DEFAULT 0.0 CHECK(progress BETWEEN 0.0 AND 1.0),
    weight              REAL NOT NULL DEFAULT 1.0,  -- auto-managed, never set manually
    status              TEXT NOT NULL DEFAULT 'inactive'
                        CHECK(status IN ('inactive', 'active', 'completed', 'abandoned')),
    due_date            DATE,
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,

    -- Exactly one parent: either an item or an objective. Never both, never neither.
    CHECK(
        (item_id IS NOT NULL AND parent_objective_id IS NULL) OR
        (item_id IS NULL     AND parent_objective_id IS NOT NULL)
    )
);
```

### Operations

```python
def add_objective(conn, label, item_id=None, parent_objective_id=None, due_date=None):
    """
    Add an Objective to an item or as a sub-Objective of another Objective.
    Exactly one of item_id or parent_objective_id must be provided.
    Returns the new objective's id.
    """
    if (item_id is None) == (parent_objective_id is None):
        raise ValueError(
            "Provide exactly one of item_id or parent_objective_id, not both or neither."
        )

    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO objectives (item_id, parent_objective_id, label, due_date)
        VALUES (?, ?, ?, ?)
    """, (item_id, parent_objective_id, label, due_date))
    conn.commit()

    # Recompute weights for the parent scope
    if parent_objective_id is not None:
        recompute_weights(conn, parent_objective_id=parent_objective_id)
    else:
        recompute_weights(conn, parent_id=item_id)


def update_objective_progress(conn, objective_id, progress):
    """
    Set progress on a leaf Objective manually.
    Raises if the Objective has children (it is not a leaf).
    Triggers upward propagation.
    """
    if not (0.0 <= progress <= 1.0):
        raise ValueError("Progress must be between 0.0 and 1.0")

    cursor = conn.cursor()

    # Ensure this is a leaf (no children)
    cursor.execute(
        "SELECT COUNT(*) FROM objectives WHERE parent_objective_id = ?",
        (objective_id,)
    )
    if cursor.fetchone()[0] > 0:
        raise ValueError(
            "Cannot set progress on a non-leaf Objective. "
            "Update its sub-Objectives instead."
        )

    cursor.execute(
        "UPDATE objectives SET progress = ? WHERE id = ?",
        (progress, objective_id)
    )
    conn.commit()

    _propagate_objective_upward(conn, objective_id)


def _recompute_objective_from_children(conn, objective_id):
    """
    Recompute a parent Objective's progress from its sub-Objectives.
    Internal -- called during propagation, not directly.
    """
    cursor = conn.cursor()
    cursor.execute("""
        SELECT progress, weight FROM objectives
        WHERE parent_objective_id = ?
    """, (objective_id,))
    children = cursor.fetchall()

    if not children:
        return  # Leaf -- do not overwrite manually set progress

    progress = sum(p * w for p, w in children)
    cursor.execute(
        "UPDATE objectives SET progress = ? WHERE id = ?",
        (progress, objective_id)
    )
    conn.commit()


def _propagate_objective_upward(conn, objective_id):
    """
    Walk upward from an Objective through its parent chain,
    recomputing each parent, until reaching an item.
    Then triggers item-level upward propagation.
    """
    cursor = conn.cursor()
    cursor.execute("""
        SELECT item_id, parent_objective_id FROM objectives WHERE id = ?
    """, (objective_id,))
    item_id, parent_obj_id = cursor.fetchone()

    if parent_obj_id is not None:
        # Parent is another Objective — recompute it, then continue upward
        _recompute_objective_from_children(conn, parent_obj_id)
        _propagate_objective_upward(conn, parent_obj_id)
    else:
        # Reached an item — recompute it and propagate through the graph
        recompute_item_progress(conn, item_id)
        update_progress_upward(conn, item_id)
```

---

## 8. Edges (Graph)

### Schema

```sql
CREATE TABLE edges (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    parent_id       INTEGER NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    child_id        INTEGER NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    weight          REAL NOT NULL DEFAULT 1.0,  -- auto-managed, never set manually
    UNIQUE(parent_id, child_id),
    CHECK(parent_id != child_id)
);
```

### Valid Type Combinations

```python
VALID_EDGES = {
    ('book',    'course'),
    ('course',  'product'),
    ('course',  'project'),
    ('product', 'project'),
}
```

### Operations

```python
def add_edge(conn, parent_id, child_id):
    """
    Add a directed edge (parent contains child).
    Validates type combination and checks for cycles before inserting.
    """
    cursor = conn.cursor()

    # Fetch types
    cursor.execute("SELECT type FROM items WHERE id = ?", (parent_id,))
    parent_type = cursor.fetchone()
    cursor.execute("SELECT type FROM items WHERE id = ?", (child_id,))
    child_type = cursor.fetchone()

    if not parent_type or not child_type:
        raise ValueError("One or both item ids do not exist.")

    parent_type = parent_type[0]
    child_type  = child_type[0]

    if (child_type, parent_type) not in VALID_EDGES:
        raise ValueError(
            f"Invalid edge: {child_type} -> {parent_type} is not a permitted relationship."
        )

    # Cycle check: would adding this edge create a path from child back to parent?
    if _would_create_cycle(conn, parent_id, child_id):
        raise ValueError(
            f"Adding edge {child_id} -> {parent_id} would create a cycle."
        )

    cursor.execute("""
        INSERT INTO edges (parent_id, child_id) VALUES (?, ?)
    """, (parent_id, child_id))
    conn.commit()

    # Recompute weights for all children under this parent
    recompute_weights(conn, parent_id=parent_id)

    # Recompute parent's progress to account for new child
    recompute_item_progress(conn, parent_id)
    update_progress_upward(conn, parent_id)


def remove_edge(conn, parent_id, child_id):
    """Remove a directed edge and rebalance weights."""
    cursor = conn.cursor()
    cursor.execute(
        "DELETE FROM edges WHERE parent_id = ? AND child_id = ?",
        (parent_id, child_id)
    )
    conn.commit()

    recompute_weights(conn, parent_id=parent_id)
    recompute_item_progress(conn, parent_id)
    update_progress_upward(conn, parent_id)


def _would_create_cycle(conn, parent_id, child_id):
    """
    Return True if adding parent -> child would create a cycle.
    Walks all descendants of child_id to check if parent_id appears.
    """
    visited = set()
    stack = [child_id]
    cursor = conn.cursor()

    while stack:
        node = stack.pop()
        if node == parent_id:
            return True
        if node in visited:
            continue
        visited.add(node)
        cursor.execute(
            "SELECT child_id FROM edges WHERE parent_id = ?", (node,)
        )
        stack.extend(row[0] for row in cursor.fetchall())

    return False
```

---

## 9. Weight Recomputation

Weights always reflect `1 / total_children` for a given parent scope.
Called whenever a child (edge or Objective) is added or removed.

```python
def recompute_weights(conn, parent_id=None, parent_objective_id=None):
    """
    Recompute weights for a given parent scope.
    Exactly one of parent_id or parent_objective_id must be provided.
    """
    if (parent_id is None) == (parent_objective_id is None):
        raise ValueError(
            "Provide exactly one of parent_id or parent_objective_id."
        )

    cursor = conn.cursor()

    if parent_objective_id is not None:
        # Scope: sub-Objectives of an Objective
        cursor.execute(
            "SELECT COUNT(*) FROM objectives WHERE parent_objective_id = ?",
            (parent_objective_id,)
        )
        total = cursor.fetchone()[0]
        if total == 0:
            return
        cursor.execute(
            "UPDATE objectives SET weight = ? WHERE parent_objective_id = ?",
            (1.0 / total, parent_objective_id)
        )

    else:
        # Scope: edge children + top-level Objectives of an item
        cursor.execute(
            "SELECT COUNT(*) FROM edges WHERE parent_id = ?", (parent_id,)
        )
        edge_count = cursor.fetchone()[0]

        cursor.execute(
            "SELECT COUNT(*) FROM objectives WHERE item_id = ?", (parent_id,)
        )
        obj_count = cursor.fetchone()[0]

        total = edge_count + obj_count
        if total == 0:
            return

        weight = 1.0 / total
        cursor.execute(
            "UPDATE edges SET weight = ? WHERE parent_id = ?",
            (weight, parent_id)
        )
        cursor.execute(
            "UPDATE objectives SET weight = ? WHERE item_id = ?",
            (weight, parent_id)
        )

    conn.commit()
```

---

## 10. Progress Propagation

```python
def recompute_item_progress(conn, item_id):
    """
    Recompute and store progress for a non-book item from its children.
    Books compute their own progress via reading position — never call this on a Book.
    """
    cursor = conn.cursor()

    cursor.execute("SELECT type FROM items WHERE id = ?", (item_id,))
    item_type = cursor.fetchone()[0]

    if item_type == 'book':
        return  # Books are never recomputed this way

    # Collect progress from graph children
    cursor.execute("""
        SELECT i.progress, e.weight
        FROM edges e
        JOIN items i ON i.id = e.child_id
        WHERE e.parent_id = ?
    """, (item_id,))
    components = list(cursor.fetchall())

    # Courses, Products, and Projects also include Objectives
    if item_type in ('course', 'product', 'project'):
        cursor.execute("""
            SELECT progress, weight FROM objectives
            WHERE item_id = ?
        """, (item_id,))
        components += cursor.fetchall()

    if not components:
        return  # No children yet — leave progress unchanged

    progress = sum(p * w for p, w in components)

    cursor.execute(
        "UPDATE items SET progress = ? WHERE id = ?",
        (progress, item_id)
    )
    conn.commit()


def update_progress_upward(conn, item_id):
    """
    Recompute progress for all ancestors of item_id.
    Called after any change that affects an item's progress.
    """
    cursor = conn.cursor()
    cursor.execute(
        "SELECT parent_id FROM edges WHERE child_id = ?", (item_id,)
    )
    parents = cursor.fetchall()

    for (parent_id,) in parents:
        recompute_item_progress(conn, parent_id)
        update_progress_upward(conn, parent_id)
```

---

## 11. Tags

```sql
CREATE TABLE tags (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    name    TEXT NOT NULL UNIQUE
);

CREATE TABLE item_tags (
    item_id INTEGER NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    tag_id  INTEGER NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (item_id, tag_id)
);
```

```python
def add_tag(conn, item_id, tag_name):
    """
    Tag an item. Creates the tag if it does not already exist.
    """
    cursor = conn.cursor()

    # Insert tag if missing
    cursor.execute(
        "INSERT OR IGNORE INTO tags (name) VALUES (?)", (tag_name,)
    )
    cursor.execute(
        "SELECT id FROM tags WHERE name = ?", (tag_name,)
    )
    tag_id = cursor.fetchone()[0]

    cursor.execute(
        "INSERT OR IGNORE INTO item_tags (item_id, tag_id) VALUES (?, ?)",
        (item_id, tag_id)
    )
    conn.commit()


def remove_tag(conn, item_id, tag_name):
    """Remove a tag from an item. Does not delete the tag itself."""
    cursor = conn.cursor()
    cursor.execute("""
        DELETE FROM item_tags
        WHERE item_id = ?
        AND tag_id = (SELECT id FROM tags WHERE name = ?)
    """, (item_id, tag_name))
    conn.commit()


def get_items_by_tag(conn, tag_name, item_type=None):
    """
    Return all items with a given tag.
    Optionally filter by item type.
    """
    cursor = conn.cursor()
    query = """
        SELECT i.id, i.type, i.title, i.status, i.progress
        FROM items i
        JOIN item_tags it ON it.item_id = i.id
        JOIN tags t ON t.id = it.tag_id
        WHERE t.name = ?
    """
    params = [tag_name]
    if item_type:
        query += " AND i.type = ?"
        params.append(item_type)

    cursor.execute(query, params)
    return cursor.fetchall()
```

---

## 12. Crawler

The crawler reconciles filesystem state against the database.
Run on demand -> never automatically. The filesystem is always the source of truth.

```python
import os
import hashlib
from pathlib import Path

LIBRARY_ROOT = Path("/your/library/root")
SUPPORTED_EXTENSIONS = {'.pdf', '.epub'}


def hash_file(path):
    """MD5 of first 64KB. Fast identity check without reading entire file."""
    h = hashlib.md5()
    with open(path, 'rb') as f:
        h.update(f.read(65536))
    return h.hexdigest()


def crawl(conn):
    """
    Walk LIBRARY_ROOT. For each supported file:
    - If not in DB: insert as new book (title defaults to filename)
    - If in DB by hash but different path: update path (file was moved)
    - If in DB and path matches: no action needed
    For each DB book not found on disk:
    - Mark status as 'missing'
    """
    cursor = conn.cursor()

    # Step 1: collect all files on disk
    disk_files = {}
    for root, dirs, files in os.walk(LIBRARY_ROOT):
        for filename in files:
            ext = Path(filename).suffix.lower()
            if ext not in SUPPORTED_EXTENSIONS:
                continue
            abs_path  = Path(root) / filename
            rel_path  = str(abs_path.relative_to(LIBRARY_ROOT))
            file_hash = hash_file(abs_path)
            disk_files[rel_path] = {
                'filename': filename,
                'path':     rel_path,
                'hash':     file_hash,
            }

    # Step 2: collect all books in DB
    cursor.execute("SELECT i.id, bm.path, bm.file_hash FROM items i JOIN book_meta bm ON bm.item_id = i.id")
    db_books = {
        row[1]: {'id': row[0], 'hash': row[2]}
        for row in cursor.fetchall()
    }

    disk_paths = set(disk_files.keys())
    db_paths   = set(db_books.keys())

    # Step 3: new or moved files
    for path in disk_paths - db_paths:
        f = disk_files[path]

        # Check if this hash exists under a different path (moved file)
        cursor.execute(
            "SELECT i.id, bm.path FROM book_meta bm JOIN items i ON i.id = bm.item_id WHERE bm.file_hash = ?",
            (f['hash'],)
        )
        existing = cursor.fetchone()

        if existing:
            item_id, old_path = existing
            print(f"  MOVED:  {old_path}  →  {path}")
            cursor.execute(
                "UPDATE book_meta SET path = ?, filename = ? WHERE item_id = ?",
                (path, f['filename'], item_id)
            )
            # Clear 'missing' status if it was marked as such
            cursor.execute(
                "UPDATE items SET status = 'unread' WHERE id = ? AND status = 'missing'",
                (item_id,)
            )
        else:
            print(f"  NEW:    {path}")
            item_id = create_item(conn, 'book', f['filename'], status='unread')
            cursor.execute("""
                INSERT INTO book_meta (item_id, filename, path, file_hash)
                VALUES (?, ?, ?, ?)
            """, (item_id, f['filename'], path, f['hash']))
            cursor.execute(
                "INSERT INTO reading_position (book_id, current_page) VALUES (?, 0)",
                (item_id,)
            )

    # Step 4: missing files
    for path in db_paths - disk_paths:
        print(f"  MISSING: {path}")
        cursor.execute(
            "UPDATE items SET status = 'missing' WHERE id = ("
            "  SELECT item_id FROM book_meta WHERE path = ?"
            ")", (path,)
        )

    conn.commit()
    print("Crawl complete.")
```

---

## Useful Queries

```sql
-- All books currently being read
SELECT i.title, bm.path, rp.current_page, bm.total_pages,
       ROUND(i.progress * 100, 1) AS percent
FROM items i
JOIN book_meta bm ON bm.item_id = i.id
JOIN reading_position rp ON rp.book_id = i.id
WHERE i.status = 'reading'
ORDER BY i.progress DESC;

-- All unread books tagged 'cryptography'
SELECT i.title, bm.path
FROM items i
JOIN book_meta bm ON bm.item_id = i.id
JOIN item_tags it ON it.item_id = i.id
JOIN tags t ON t.id = it.tag_id
WHERE t.name = 'cryptography' AND i.status = 'unread';

-- Overall progress of every project
SELECT i.title, ROUND(i.progress * 100, 1) AS percent, i.status
FROM items i
WHERE i.type = 'project'
ORDER BY i.progress DESC;

-- All objectives for a product, with progress
SELECT label, ROUND(progress * 100, 1) AS percent, status, due_date
FROM objectives
WHERE item_id = ?
ORDER BY status, due_date;

-- Books that belong to multiple courses (shared nodes in the graph)
SELECT i.title, COUNT(e.parent_id) AS course_count
FROM items i
JOIN edges e ON e.child_id = i.id
WHERE i.type = 'book'
GROUP BY i.id
HAVING course_count > 1;
```
