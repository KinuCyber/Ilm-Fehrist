# Ilm Fehrist (IF)

This is just a personal project I've developed through AI assistance.
Mostly to help me keep track of my books, progress in courses related
to the books, such as Lamport's paper for Gnosyn

**Entities & Hierarchy**

There are four entity types, arranged in a strict composition hierarchy:

```
Project
`-- Product
    `-- Course
        `-- Book
```

Valid edge relationships are:
```
book    -> course
course  -> product
course  -> project
product -> project
```
Invalid edge relationships are:
```
book    -> product
book    -> project
```

---

**Graph, Not Hierarchy**

The structure is a directed graph, not a strict tree. A single Book can belong to multiple Courses. A single Course can belong to multiple Products or Projects. Progress propagates upward through all parents.

---

**Progress**

- Progress is stored on the item itself as a float (0.0–1.0), not computed at query time.
- For Books, progress is derived purely from `current_page / total_pages`.
- For Courses, Products, and Projects, progress is the weighted average of direct children's progress.
- Edge weight is always `1 / number_of_children` for a given parent — never set manually.
- Weights are recomputed automatically whenever a child is added or removed from a parent.
- When a Book's reading position updates, progress propagates upward recursively through all ancestors.

---

**Books & Filesystem**

- The filesystem is the source of truth. The database is a queryable index built on top of it.
- Files are never stored in the database — only their path and metadata.
- A crawler reconciles disk state against DB state, handling four cases: new, moved, removed, unchanged.
- File hashing (MD5 of first 64KB) detects moved or renamed files, preserving all metadata and tags.
- Removed files are marked `missing` rather than deleted, so metadata is not lost.

---

**Bookmarks vs Reading Position**

These are two distinct, fully independent concepts:

- **Bookmarks** are pure annotations. Multiple per book. Each has a page, optional label, and optional note. They have zero effect on progress. Intended for marking future reading targets, chapter starts, points of interest.
- **Reading Position** is the single source of truth for actual progress. One per book, always. It is the only thing that triggers progress recomputation. Stored as an upsert so duplicates are structurally impossible.

---

**Tags**

Tags are independent entities. Any item of any type can have multiple tags. Tags enable cross-cutting queries that the hierarchy alone cannot express (e.g. all unread cryptography material regardless of which course it belongs to).

---

**What Is Still Undefined**

- Product-specific metadata and progression unit (milestone equivalent for Products)
- Project-specific metadata and progression unit
- Whether reading position can move backwards, or is forward-only

---

For any inquiries: **contact@kinu.uk**

Ilm Fehrist - علم فہرست

&copy; Kinu Cyber

Released under GNU GPL-3.0.
