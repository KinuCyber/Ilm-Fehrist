# Ilm Fehrist - System Philosophy

This document is the single source of truth for application philosophy.
Read DESIGN.md for the implement of the following philosophy as core
logic and database schema.

## Entities

There are four entity types. Each is distinct in what it represents, how it progresses, and what it contains.

| Type    | Represents                          | Progression Unit                          | Example               |
|---------|-------------------------------------|-------------------------------------------|-----------------------|
| Book    | A readable file on disk             | Reading position (pages)                  | TCP/IP Illustrated    |
| Course  | A structured study unit             | Children (Books) + Objectives             | Networking 101        |
| Product | A deliverable with versioned output | Children (Courses) + Objectives           | Gnosyn Phase 1 (PoC)  |
| Project | A long-term dleiverable             | Children (Products, Courses) + Objectives | Project Gnosyn        |

A Project is realized when all its children and Objectives complete. Project phases (Gnosyn Phase 1, Phase 2...) are Products within the Project. A Product is a quick, deliverable result of Courses, with ongoing versioned iterations. A Product can be deprecated before completion. A Project can be abandoned before completion.


## Entities & Relations

**Entities**

There are four item types, conceptually arranged as:
```
Project --> Product --> Course --> Book
```

**Relations**

Valid edge relationships are:
```
Book    -> Course
Course  -> Product
Course  -> Project
Product -> Project
```
Invalid edge relationships are:
```
Book    -> Product
Book    -> Project
```

## Graph, Not Tree

The structure is a directed graph. A single Book can belong to multiple Courses. A single Course can belong to multiple Products or Projects. Progress propagates upward through all parents when any child updates.

Cycles are forbidden. Enforced in application logic, not SQL.

## Progress

Progress is stored on each item as a float (0.0–1.0), updated on change, not computed at query time.

### Per Type

**Book**
```
progress = current_page / total_pages
```
Set only by updating the reading position. Never set directly.

**Course**
```
progress = weighted avg of child Books + Objectives
```

**Product**
```
progress = weighted avg of child Courses + Objectives
```

**Project**
```
progress = weighted avg of child Products + child Courses + Objectives
```

### Weights

Weight for any child (edge or Objective) under a given parent is always:
```
weight = 1 / (edge_children_count + top_level_objectives_count)
```
Never set manually. Recomputed automatically whenever a child or top-level Objective is added to or removed from a parent. Weights always sum to 1.0 for any parent.

For sub-Objectives, weight is computed identically but scoped to siblings under the same parent Objective:
```
weight = 1 / sibling_objectives_count
```

### Propagation

When any item's progress changes, it propagates upward recursively through all ancestors via the edges table. When any Objective's progress changes, it propagates upward through its parent Objective chain (if any), then into its parent item, then upward through the graph from there.

## Objectives

Objectives are the shared progression unit across Course, Product, and Project. They represent work that either augments or falls outside the Book hierarchy: implementation, marketing, networking, distribution, or any named sub-task within those.

### Properties

- Progress on a leaf Objective is set manually (0.0–1.0)
- Progress on a parent Objective is derived from its sub-Objectives
- Objectives carry a weight, auto-managed identically to edge weights
- Completing an Objective triggers the same upward propagation as completing a child item
- Objectives have their own status: `inactive | active | completed | abandoned`
- Objectives can have an optional due date

### Sub-Objectives

Objectives are recursive. A top-level Objective can contain sub-Objectives, whose progress rolls up into the parent Objective automatically. This allows coarse tasks like "Implementation" or "Marketing" to be broken into finer-grained trackable pieces when needed.

**Discipline rule:** An Objective should only be broken into sub-Objectives if its sub-tasks are so different in nature that a single progress estimate would be meaningless. If you can honestly say "this is about 40% done," keep it atomic. Sub-Objectives should be kept as shallow as possible — one level deep unless there is a genuine reason to go further. Decomposition is not a substitute for progress.

### Parentage

Every Objective belongs to exactly one parent — either an item (Course, Product, or Project) or another Objective. Never both, never neither. This is enforced at the schema level.

## Bookmarks vs Reading Position

These are two fully independent concepts with no overlap.

### Bookmarks
- Multiple per Book
- Each has a page number, optional label, optional note
- Intended for future reading targets, chapter markers, points of interest
- Have **zero effect** on progress, ever

### Reading Position
- Exactly one per Book, always
- The **only** thing that triggers progress recomputation
- Stored as an upsert — duplicates are structurally impossible
- **Forward-only:** reading position cannot move backwards
- Every update is the definitive statement of how far a Book has been read

## Books & Filesystem

- The filesystem is the single source of truth. The database is a queryable index built on top of it, never a replacement for it.
- Files are never stored in the database — only their relative path and metadata.
- A crawler reconciles disk state against DB state on demand.

### Crawler Logic

| Disk    | DB      | Action                                        |
|---------|---------|-----------------------------------------------|
| EXISTS  | MISSING | INSERT as new item                            |
| EXISTS  | EXISTS  | Check hash — UPDATE path if moved             |
| MISSING | EXISTS  | Mark status as `missing` (never DELETE)       |

- File identity uses MD5 of the first 64KB. Detects moves and renames without reading entire files, and preserves all metadata, tags, and reading history when files are reorganized.
- Removed files are marked `missing`, not deleted. No metadata is ever lost due to a filesystem change.

## Statuses

**Book:** `unread | reading | completed | missing`

**Course:** `inactive | active | completed`

**Product:** `inactive | active | completed | deprecated`

**Project:** `inactive | active | completed | abandoned`

**Objective:** `inactive | active | completed | abandoned`

## Versioning (Products)

Versions are a property of a Product, not separate items. A Product maintains a current version and a version history. Each version entry can optionally hold a planned and actual release date — designed to accommodate future deadline tracking without enforcing it prematurely.

## Tags

Tags are independent entities. Any item of any type can have multiple tags. Tags enable cross-cutting queries that the hierarchy alone cannot express — for example, all unread cryptography material regardless of which Course or Product it belongs to.

## Database Conventions
 
- All write operations follow a try/except/rollback pattern. On failure, changes since the last commit are undone and the error is re-raised with context.
- All read operations (SELECT only) require no transaction handling — they have no side effects.
- The database connection (`conn`) is opened once at program startup via `open_db()`, which applies required pragmas (`foreign_keys = ON`, `journal_mode = WAL`), and passed into every function that needs it. It is never opened inside individual functions.
- Schema migrations are versioned and applied in order. A `schema_migrations` table tracks which have run. The filesystem of migration files is the source of truth for schema history.
 
## Graph Traversal
 
The graph structure supports upward traversal from any item to all of its ancestors. This is used internally for progress propagation and is also available as a query for debugging and future UI work — for example, tracing which Projects a given Book ultimately contributes to. Traversal is implemented as a recursive CTE in SQL or an iterative depth-first walk in Python.
 
## What Is Still Undefined

- Project-specific metadata beyond deadline (if any)
- Whether reading position history is kept (currently only the latest position is stored)
- Maximum enforced depth of sub-Objective nesting, if any
