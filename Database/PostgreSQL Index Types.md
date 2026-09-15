# PostgreSQL Index Type: From Senior to Principal

> **Goal:** Move beyond "which index do I add" to "why does this index behave the way it does, what does it cost, and how do I prove it's the right one." Every section pairs the mental model with the internals that justify it.

## Table of Contents

1. [Mental Model: What an Index Actually Is](#1-mental-model-what-an-index-actually-is)
2. [Access Method Framework & Operator Classes](#2-access-method-framework--operator-classes)
3. [B-Tree — The Default Workhorse](#3-b-tree--the-default-workhorse)
4. [Hash Index](#4-hash-index)
5. [GiST (Generalized Search Tree)](#5-gist-generalized-search-tree)
6. [SP-GiST (Space-Partitioned GiST)](#6-sp-gist-space-partitioned-gist)
7. [GIN (Generalized Inverted Index)](#7-gin-generalized-inverted-index)
8. [BRIN (Block Range Index)](#8-brin-block-range-index)
9. [Clustered vs Non-Clustered Indexes](#9-clustered-vs-non-clustered-indexes)
10. [Composite (Multicolumn) Indexes & Column Ordering](#10-composite-multicolumn-indexes--column-ordering)
11. [Covering Indexes (`INCLUDE`) & Index-Only Scans](#11-covering-indexes-include--index-only-scans)
12. [Partial Indexes](#12-partial-indexes)
13. [Expression Indexes](#13-expression-indexes)
14. [Unique Indexes & Constraints](#14-unique-indexes--constraints)
15. [Extension-Backed Index Types](#15-extension-backed-index-types)
16. [A Practical Process for Evaluating & Selecting an Index](#16-a-practical-process-for-evaluating--selecting-an-index)
17. [Build & Maintenance: Locking, Bloat, REINDEX](#17-build--maintenance-locking-bloat-reindex)
18. [Planner Internals: Cost Model & Selectivity](#18-planner-internals-cost-model--selectivity)
19. [Indexes at Scale: Partitioning, Parallelism, Replication](#19-indexes-at-scale-partitioning-parallelism-replication)
20. [Monitoring & Diagnostics](#20-monitoring--diagnostics)
21. [Anti-Patterns & Principal-Level Judgment Calls](#21-anti-patterns--principal-level-judgment-calls)
22. [Quick Reference Tables](#22-quick-reference-tables)
23. [Composite Index Storage & Query Mechanics — A Worked Walkthrough](#23-composite-index-storage--query-mechanics--a-worked-walkthrough)

---

## 1. Mental Model: What an Index Actually Is

A PostgreSQL index is **not a shortcut into the table** — it is a separate, independently-stored data structure containing (typically) two things per entry:

- A **key** — the indexed column value(s), or a derived value (hash, expression result, tsvector, etc.)
- A **pointer** — a `ctid` (`(block, offset)`) into the heap, or in some engines a pointer to another index page

Consequences that matter at Principal level:

- **An index is a second copy of data.** Every `INSERT`/`UPDATE`/`DELETE` that touches an indexed column pays a write in the index *in addition to* the heap. N indexes on a table means up to N+1 page writes per mutating statement (see [§17](#17-build--maintenance-locking-bloat-reindex) for HOT as the exception).
- **An index does not know if its pointer is still valid.** Because of MVCC, a `ctid` in an index may point to a dead tuple. The heap visibility check (`xmin`/`xmax` against your snapshot) is unavoidable — *unless* the Visibility Map says the page is all-visible, which is what makes Index-Only Scans possible ([§11](#11-covering-indexes-include--index-only-scans)).
- **"Index" is a family, not one structure.** PostgreSQL exposes a pluggable **access method (AM)** interface. B-Tree is the default AM, but GiST, SP-GiST, GIN, BRIN, and Hash are all first-class citizens with completely different storage layouts and completely different questions they answer well.

The Principal-level habit: before reaching for `CREATE INDEX`, ask **"what predicate, in what shape, over what data distribution?"** — the answer picks the AM, not intuition.

---

## 2. Access Method Framework & Operator Classes

Every index is bound to an **operator class (opclass)**, which tells the AM which operators (`=`, `<`, `&&`, `<->`, …) it can accelerate for a given type. This is why `CREATE INDEX ... USING gin (tags)` works for `jsonb` but you need `jsonb_path_ops` opclass to get smaller, faster indexes for `@>` containment queries specifically.

```sql
-- Default opclass
CREATE INDEX idx_tags ON docs USING gin (tags);

-- Specialized opclass: smaller index, faster containment, no key-existence (?, ?|, ?&) support
CREATE INDEX idx_tags_path ON docs USING gin (tags jsonb_path_ops);
```

Key mental model: **the opclass is the contract between your query's WHERE clause and the index's ability to help.** `EXPLAIN` will silently ignore an index if the operator in your predicate isn't covered by the index's opclass — this is the #1 cause of "I have an index but it's not being used."

List available AMs and opclasses:

```sql
SELECT amname FROM pg_am;
SELECT opcname, amname FROM pg_opclass oc JOIN pg_am am ON am.oid = oc.opcmethod;
```

---

## 3. B-Tree — The Default Workhorse

### 3.1 Structure

A balanced tree of fixed-size (8 kB) pages. Internal pages hold `(key, downlink)` pairs; leaf pages hold `(key, ctid)` pairs and are doubly-linked (left/right sibling pointers) to support efficient range and reverse scans without re-descending the tree.

```
              [ 50 ]
            /        \
       [20|35]      [65|80]
       /  |  \        /  |  \
   leaf leaf leaf  leaf leaf leaf  <-- linked list, supports ORDER BY asc/desc
```

### 3.2 What it accelerates

`=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `ORDER BY` (both directions, for free — no sort node needed), `IS NULL`, and as of PG 9.6+ **multi-column skip-scans-adjacent** behavior for leading-column-agnostic equality on some patterns (see [§10](#10-composite-multicolumn-indexes--column-ordering)).

### 3.3 Deduplication (PG 13+)

Since PG13, B-Tree leaf pages compress runs of identical keys into a single key with a list of TIDs (a "posting list"), similar in spirit to GIN. This dramatically shrinks indexes with many duplicate values (e.g., a `status` column with 5 distinct values over 100M rows) — but it is **only safe** with the default opclasses (no non-deterministic collations); PostgreSQL disables it automatically when unsafe.

### 3.4 Nulls

B-Tree indexes **do** store NULLs by default (sorted as highest by default, controlled by `NULLS FIRST/LAST`). This matters because `WHERE col IS NULL` *can* use a plain B-Tree — a common misconception is that you need a partial index for this. You still might want one if NULLs are rare (see [§12](#12-partial-indexes)).

### 3.5 Skip Scan (PG 18+)

Historically, a composite index `(a, b)` was useless for `WHERE b = 5` alone — the planner had to scan the whole index or ignore it. PostgreSQL 18 introduced **B-Tree Skip Scan**, letting the planner jump between distinct values of the leading column `a` even when `a` isn't in the predicate, when `a` has low cardinality. This is a genuine internals shift — re-evaluate long-held "leading column must be in the WHERE clause" rules on 18+, but don't rely on it as a substitute for correct column ordering on earlier versions or high-cardinality leading columns.

### 3.6 When B-Tree is the wrong tool

- Equality on a huge composite type where you only ever query prefixes → fine, still B-Tree.
- Containment (`@>`, `<@`), full-text, fuzzy match, geometric overlap → wrong tool, see GIN/GiST.
- Append-only, naturally correlated columns (timestamps, IDs) at huge scale where index size dominates → consider BRIN instead ([§8](#8-brin-block-range-index)).

---

## 4. Hash Index

### 4.1 Structure

A hash table of the indexed value, bucketed into pages. Stores `(hash(value), ctid)` — **not the value itself** — so it can *only* answer `=`. No ordering, no range queries, no `ORDER BY` support.

### 4.2 Why it's rarely chosen

For years Hash indexes weren't WAL-logged (unsafe on replicas/crash recovery) and weren't measurably faster than B-Tree for equality due to B-Tree's excellent cache behavior on integer/text equality. **As of PG 10, Hash indexes are WAL-logged and crash-safe**, and they *can* be meaningfully smaller and faster for pure high-cardinality equality lookups on wide keys (e.g., long text/UUID equality) because the index stores only a 4-byte hash, not the full value.

### 4.3 When to actually use it

- Equality-only lookups on **wide keys** (long strings, JSON blobs hashed for dedup) where you never need ordering or range.
- You've benchmarked and confirmed B-Tree isn't smaller/faster — don't reach for Hash speculatively; B-Tree's flexibility (it can always do `=` plus everything else) usually wins unless key width is the bottleneck.

```sql
CREATE INDEX idx_payload_hash ON events USING hash (payload_digest);
```

Caveat: hash collisions mean the AM still must re-check the actual heap tuple, so Hash gives you smaller index size, not a fundamentally different lookup cost class.

---

## 5. GiST (Generalized Search Tree)

### 5.1 Structure

Not one algorithm — a **framework** for building tree-based indexes where each node is a "lossy" bounding predicate about everything beneath it (think R-tree bounding boxes). Every leaf's ancestors must be a superset ("consistent") of the leaf's data, which is why GiST supports overlapping/nested searches that a strict, ordered B-Tree cannot.

### 5.2 What it accelerates

- **Geometric types**: `box`, `circle`, `polygon` — `&&` (overlaps), `<->` (KNN distance/"nearest neighbor")
- **PostGIS** geography/geometry — spatial indexing is built entirely on GiST
- **Range types** (`int4range`, `tsrange`, etc.) — overlap, containment, adjacency
- **`btree_gist` extension** — lets you put ordinary scalar types into a GiST index so they can be combined with range/geometric columns in a single **exclusion constraint**
- **Full-text search** (`tsvector`) — GiST is an option, though GIN is usually preferred for read-heavy FTS (see [§7](#7-gin-generalized-inverted-index))
- **Nearest-neighbor (`ORDER BY col <-> point`)** — GiST natively supports KNN-style ordered traversal; this is the mechanism behind "find the 10 closest stores to this point" queries.

### 5.3 Killer feature: Exclusion Constraints

GiST is the only built-in AM that enforces **"no two rows may overlap"** at the constraint level — the classic use case is preventing double-booking:

```sql
CREATE EXTENSION btree_gist;

CREATE TABLE room_booking (
  room_id   int,
  during    tstzrange,
  EXCLUDE USING gist (room_id WITH =, during WITH &&)
);
```

This single constraint replaces an entire class of application-level "check for conflicts then insert" race conditions — it's enforced atomically by the database.

### 5.4 Cost profile

GiST is lossy by construction for some opclasses — a bounding box may include a "false positive" row that the executor must re-check (`Recheck Cond` in `EXPLAIN`). Build time is generally slower than B-Tree for the same cardinality, and it doesn't compress as aggressively as GIN for high-duplication data.

---

## 6. SP-GiST (Space-Partitioned GiST)

### 6.1 Structure

Also a framework, but for **non-balanced, disjoint partition trees** — think quad-trees, k-d trees, radix (prefix) trees — instead of GiST's balanced, overlapping bounding-box hierarchy. Each node partitions space into non-overlapping regions, so it excels when data is **naturally clustered/hierarchical and unevenly distributed**.

### 6.2 What it accelerates

- **IP address / CIDR data** (`inet`, `cidr`) — radix-tree-style prefix matching, much faster than GiST for `<<` (contained by network)
- **Text prefix search** — `text_ops` SP-GiST opclass out-performs B-Tree for `LIKE 'prefix%'` on highly-skewed string data
- **Geometric points** (not boxes) — k-d tree partitioning, faster inserts than GiST for point-only workloads
- **Phone numbers, IP ranges, taxonomies** — anything hierarchical/prefix-shaped

### 6.3 SP-GiST vs GiST — the actual decision

| | GiST | SP-GiST |
|---|---|---|
| Tree shape | Balanced, can overlap | Unbalanced, disjoint partitions |
| Best for | Overlapping ranges/regions | Hierarchical/prefix, skewed data |
| Insert cost | Higher (rebalancing) | Lower for skewed distributions |
| Example type | `box`, `tsrange` | `inet`, point clustering |

Most teams never touch SP-GiST directly — it matters when the data itself is prefix/hierarchical shaped (network ranges are the most common real-world case).

---

## 7. GIN (Generalized Inverted Index)

### 7.1 Structure

An **inverted index**: for a composite value (array, jsonb, tsvector), GIN indexes each *component* (each array element, each JSON key/value, each lexeme) separately, mapping `component → list of ctids`. This is the same structural idea as a search-engine inverted index.

```
'cat'   -> [ctid1, ctid5, ctid9]
'dog'   -> [ctid2, ctid5]
'bird'  -> [ctid9]
```

### 7.2 What it accelerates

- **Full-text search** (`tsvector @@ tsquery`) — the standard-library FTS solution
- **`jsonb` containment** (`@>`, `?`, `?|`, `?&`) — index every key/value pair
- **Array containment/overlap** (`@>`, `<@`, `&&`)
- **`pg_trgm` trigram search** — see [§15](#15-extension-backed-index-types) for fuzzy `LIKE '%term%'` and similarity search

### 7.3 The write-cost tradeoff (Principal-level knowledge)

GIN indexes are **expensive to update** — a single row change can touch many inverted-list entries (every lexeme in a document, every array element). PostgreSQL mitigates this with the **GIN Pending List**: new entries are appended to an unsorted, unindexed staging area (`gin_pending_list_limit`, default 4MB) and merged into the main structure lazily (on `VACUUM`, or when the pending list fills). This trades read-latency (queries must also scan the pending list) for dramatically better write throughput.

```sql
-- Force pending-list flush if you need consistent read latency after a bulk load
SELECT gin_clean_pending_list('idx_docs_fts');
```

For write-heavy tables, either tune `gin_pending_list_limit` up (fewer flushes, larger pending-list read cost) or run periodic `VACUUM` to control it — don't just leave the default and be surprised by read-latency variance.

### 7.4 `jsonb_ops` vs `jsonb_path_ops`

| | `jsonb_ops` (default) | `jsonb_path_ops` |
|---|---|---|
| Index size | Larger (~2-3x) | Smaller |
| Supports `@>` | Yes | Yes |
| Supports `?`, `?|`, `?&` (key existence) | Yes | **No** |
| Lookup speed for `@>` | Slower (more false positives) | Faster |

Rule: if you only ever do containment queries (`data @> '{"status":"active"}'`), use `jsonb_path_ops` — smaller and faster. Only use the default if you need bare key-existence operators too.

### 7.5 Multicolumn GIN is not leftmost-prefix-bound

Unlike B-Tree, a multicolumn GIN index does **not** require the first indexed column to appear in the query — every column contributes independently searchable entries, and the AM combines the resulting candidate bitmaps. See [§10.4](#104-composite-index-behavior-differs-by-access-method) for the full comparison across AMs.

---

## 8. BRIN (Block Range Index)

### 8.1 Structure

The structural opposite of GIN: instead of one entry per row, BRIN stores **one summary per physical block range** (default 128 pages = 1MB), recording e.g. `MIN`/`MAX` for that range. It is a **lossy, tiny** index — often 1000x smaller than an equivalent B-Tree.

```
Block range 0-127:   min=2024-01-01, max=2024-01-03
Block range 128-255: min=2024-01-03, max=2024-01-05
...
```

### 8.2 The one condition that makes or breaks BRIN

BRIN only helps if the indexed column is **physically correlated with insertion order** — the classic case is an append-only `created_at` or auto-increment `id` on a table that's never `UPDATE`d out of order. Check correlation before creating one:

```sql
SELECT attname, correlation FROM pg_stats
WHERE tablename = 'events' AND attname = 'created_at';
-- correlation near 1.0 (or -1.0) = BRIN will be effective
-- correlation near 0 = BRIN will scan almost every range = useless
```

If correlation is poor (e.g., a `status` column that gets updated randomly across the table over time), BRIN degrades toward a full table scan because nearly every block range's min/max will overlap the query predicate. A one-time `CLUSTER` on a correlated index can restore this correlation — see [§9.4](#94-when-clustered-like-behavior-actually-matters).

### 8.3 Why Principal engineers reach for it

- Multi-terabyte, append-only fact tables (time-series, event logs, audit trails) where a B-Tree on `created_at` would itself be tens of GB and mostly redundant with physical layout.
- Near-zero write overhead — updating a BRIN summary is far cheaper than maintaining a B-Tree, since only the *block range's* summary changes, not a per-row entry.
- Trade precision for size: BRIN scans always **recheck** against the heap (it's lossy — it tells you "row might be in this 1MB range," not "this exact row"), so it's a poor fit for highly selective point lookups, but excellent for range scans over huge correlated tables.

```sql
CREATE INDEX idx_events_brin ON events USING brin (created_at) WITH (pages_per_range = 32);
```

Lowering `pages_per_range` trades index size for precision — smaller ranges mean less "wasted" scanning per summary, at the cost of a bigger (though still tiny relative to B-Tree) index.

---

## 9. Clustered vs Non-Clustered Indexes

### 9.1 PostgreSQL tables are heap-organized by default

In SQL Server or MySQL/InnoDB, a **clustered index** (usually the primary key) *is* the physical storage of the table — the index's leaf level directly contains the row data, so there is no separate "table file." Every other index on that table is, by contrast, a **non-clustered (secondary) index**: a separate structure of `(key, pointer-to-clustered-row)`.

**PostgreSQL has no true clustered index.** Every table is a **heap**: rows live in a plain heap file in insertion/free-space order, completely independent of any index. Every PostgreSQL index — including the primary key — is structurally what other databases would call "non-clustered": a separate `(key, ctid)` structure pointing back into the heap.

```
Other RDBMS (clustered PK):        PostgreSQL:
┌─────────────────────┐            ┌───────────┐      ┌────────────────┐
│ PK B-Tree            │           │ Heap file │◄─────│ PK B-Tree      │
│  leaf = actual row   │           │ (rows in  │ ctid │ leaf = (key,   │
│  data                 │           │ insert    │      │  ctid)         │
└─────────────────────┘            │ order)    │      └────────────────┘
Secondary index → PK value          └───────────┘      ┌────────────────┐
                                                  ◄──────│ Secondary index│
                                                    ctid │ leaf = (key,   │
                                                          │  ctid)         │
                                                          └────────────────┘
```

### 9.2 Why this is a deliberate MVCC tradeoff, not a limitation

Because an `UPDATE` in PostgreSQL creates a new tuple version at a (potentially) new physical location ([§17.2](#172-hot-heap-only-tuple-updates--the-exception-to-every-index-pays-on-every-write)), a true always-clustered structure would require rewriting the clustering index's physical layout on every update. Heap-organization decouples logical row identity from physical placement, which is precisely what keeps MVCC's "old versions stay put until vacuumed" model cheap. The cost of this design is structural: **every** PostgreSQL index lookup, even by primary key, is index-scan → `ctid` → heap-fetch, unless an Index-Only Scan applies ([§11](#11-covering-indexes-include--index-only-scans)).

### 9.3 `CLUSTER` — a one-time, non-maintained physical reorder

PostgreSQL does offer a command that produces clustered-like locality, but only as a **snapshot operation**, not an ongoing property:

```sql
CLUSTER orders USING idx_orders_created_at;
```

This physically rewrites the entire heap so row order matches the given index's key order — improving sequential I/O locality and OS readahead for range-scan-heavy access patterns, and (usefully) improving BRIN's `correlation` statistic on that column.

**Critical caveat: `CLUSTER` does not keep the table clustered.** New `INSERT`s and relocated `UPDATE`s go back to being placed by free-space availability, not by clustering key — the physical ordering decays continuously after the command runs. To keep it effective you must either:
- Re-run `CLUSTER` periodically (it takes an **`ACCESS EXCLUSIVE`** lock — blocks *all* reads and writes for the full rewrite, unlike `CREATE INDEX CONCURRENTLY`), or
- Use the community `pg_repack` extension, which achieves the same physical reordering without the long exclusive lock.

```sql
-- Check which index (if any) a table was last clustered on
SELECT c.relname AS table, i.relname AS clustered_on
FROM pg_index idx
JOIN pg_class c ON c.oid = idx.indrelid
JOIN pg_class i ON i.oid = idx.indexrelid
WHERE idx.indisclustered;
```

### 9.4 When "clustered-like" behavior actually matters

- **Range-scan-heavy analytical/reporting tables** where physical locality with the dominant access pattern meaningfully cuts I/O — e.g., a multi-tenant table always queried `WHERE tenant_id = X ORDER BY created_at`: clustering by `(tenant_id, created_at)` co-locates each tenant's rows on disk.
- **Rescuing BRIN effectiveness**: if a naturally-append-only column has drifted out of correlation (e.g., after a bulk `UPDATE` or restore), `CLUSTER`-ing on it restores the correlation BRIN depends on ([§8.2](#82-the-one-condition-that-makes-or-breaks-brin)).
- **Rarely worth it for OLTP write-heavy tables** — the `ACCESS EXCLUSIVE` lock cost plus continuous decay make it a reporting/warehouse-table tool, not a general-purpose one. Most OLTP tables should just stay heap-organized and lean on covering indexes ([§11](#11-covering-indexes-include--index-only-scans)) instead.

### 9.5 Quick comparison

| | True Clustered Index (SQL Server / InnoDB) | PostgreSQL |
|---|---|---|
| Table storage | *Is* the index's leaf level | Always a separate heap |
| Index → row lookup | None needed (data lives in the index) | Always `ctid` → heap fetch, unless Index-Only Scan |
| Maintained automatically | Yes, kept in sync on every write | No — `CLUSTER` is a one-time snapshot that decays |
| Cost of "reclustering" | Amortized into every write | Full-table rewrite + `ACCESS EXCLUSIVE` lock (or `pg_repack`) |
| One per table, by definition | Yes | N/A — `CLUSTER` can retarget to any index, but only one physical order is true at a time |

---

## 10. Composite (Multicolumn) Indexes & Column Ordering

### 10.1 Terminology

"Composite index" and "multicolumn index" are the same thing: a single index built over **two or more columns (or expressions)**, as opposed to a single-column index. PostgreSQL supports composite indexes on B-Tree, GiST, GIN, SP-GiST, and BRIN (up to 32 columns by default for B-Tree/GIN).

### 10.2 The leftmost-prefix rule (B-Tree)

A B-Tree composite index `(a, b, c)` can efficiently serve:
- `WHERE a = ?`
- `WHERE a = ? AND b = ?`
- `WHERE a = ? AND b = ? AND c = ?`
- `WHERE a = ? AND b > ?` (range on the *last* used column)

It generally **cannot** efficiently serve `WHERE b = ?` alone (pre-PG18 Skip Scan, [§3.5](#35-skip-scan-pg-18)) or `WHERE a = ? AND c = ?` skipping `b` for equality (it *can* use `a`, but `c` isn't index-searchable until `b` is pinned). This rule is specific to B-Tree's physically-sorted, concatenated-key structure — see [§10.4](#104-composite-index-behavior-differs-by-access-method) for how other AMs differ.

### 10.3 Ordering heuristics, in priority order

1. **Equality columns before range columns.** `(status, created_at)` beats `(created_at, status)` for `WHERE status = 'x' AND created_at > y` — the range predicate should be last so the B-Tree can still narrow by equality first, then scan a contiguous range.
2. **Higher selectivity first, among equality columns** — put the column that eliminates the most rows leftmost, all else equal. But equality-before-range (#1) always wins when the two rules conflict.
3. **Match `ORDER BY`.** If the query does `WHERE status = 'x' ORDER BY created_at DESC`, ordering the index `(status, created_at DESC)` lets the planner skip a separate `Sort` node entirely.
4. **Consider query frequency, not just one query.** A composite index built for query A might be uselessly ordered for query B — sometimes two narrower indexes beat one "clever" composite one. Don't over-fit a single index to a single query if the table serves varied access patterns.

### 10.4 Composite index behavior differs by access method

This is the point most engineers miss: the leftmost-prefix rule is a **B-Tree property**, not a universal one.

| AM | Composite key behavior | Column order matters? |
|---|---|---|
| B-Tree | Concatenated, physically sorted key | Yes — strict leftmost-prefix rule |
| GIN | Each column's entries indexed independently ([§7.5](#75-multicolumn-gin-is-not-leftmost-prefix-bound)); results combined via bitmap AND/OR | No — any subset of indexed columns can drive a search |
| GiST | Per-column `consistent` checks chained; later columns only narrow if earlier ones are checkable, but there's no strict single-key ordering like B-Tree | Partially — put the most selective/most commonly-queried column first as a heuristic, not a hard rule |
| BRIN | One summary tuple per block range covering all columns | Order doesn't affect correctness; effectiveness still depends on each column's own physical correlation |

Principal-level implication: don't blindly port "equality columns first" thinking onto a composite GIN or BRIN index — verify with `EXPLAIN` what each AM actually does with your specific predicate combination.

### 10.5 Redundant index elimination

`(a)` is fully redundant once `(a, b)` exists for equality-only B-Tree workloads — the composite index serves single-column `a` lookups too (leftmost prefix). Principal-level hygiene: periodically audit for this via `pg_stat_user_indexes` ([§20.3](#203-duplicateredundant-indexes)) — redundant indexes are pure write-amplification with zero read benefit.

---

## 11. Covering Indexes (`INCLUDE`) & Index-Only Scans

### 11.1 The mechanism

An **Index-Only Scan** answers a query entirely from the index, never touching the heap — *if and only if*:

1. Every column the query needs (`SELECT` list + `WHERE`) is present in the index, and
2. The heap page is marked **all-visible** in the **Visibility Map (VM)**, so PostgreSQL doesn't need to check `xmin`/`xmax` against the heap tuple.

```sql
CREATE INDEX idx_orders_covering ON orders (customer_id) INCLUDE (order_date, total);
```

`INCLUDE` columns are **not part of the key** — they aren't sorted, can't be used in `WHERE`/`ORDER BY`, and don't participate in uniqueness for `UNIQUE` indexes, but they ride along in the leaf page so the heap lookup can be skipped.

### 11.2 Why the Visibility Map is the real gatekeeper

A well-designed covering index on a table with heavy churn can still fall back to regular Index Scans if `VACUUM` isn't keeping the VM up to date. This is the single most common reason "I added a covering index and `EXPLAIN` still shows heap fetches" — check `Heap Fetches` in `EXPLAIN (ANALYZE, BUFFERS)`:

```
Index Only Scan using idx_orders_covering on orders
  Heap Fetches: 15234   <-- high number = VM is stale, tune autovacuum
```

High `Heap Fetches` on a supposedly covering index → the table needs more aggressive autovacuum (lower `autovacuum_vacuum_scale_factor`), not a different index.

### 11.3 When to use `INCLUDE` vs adding to the key

Add a column to `INCLUDE` (not the key) when you need it in the `SELECT` list but never filter/sort by it — this keeps the B-Tree smaller and avoids unnecessary dedup/comparison overhead on a column that adds no selectivity.

---

## 12. Partial Indexes

An index with a `WHERE` clause, indexing only a subset of rows:

```sql
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';
```

### 12.1 Why this is a Principal-level lever, not a minor optimization

- **Size**: if `pending` is 0.1% of a 500M-row table, this index is ~1000x smaller than a full index — faster to build, faster to scan, faster to maintain, trivially fits in cache.
- **Planner must prove applicability**: the partial index is only used when the query's `WHERE` clause **provably implies** the index predicate. `WHERE status = 'pending' AND created_at > now() - interval '1 day'` matches; `WHERE status IN ('pending','shipped')` does **not**, even though `pending` is technically a subset — the planner won't do arbitrary logical inference, it does a fairly literal implication check.
- **Common patterns**: soft-delete tables (`WHERE deleted_at IS NULL`), state machines (`WHERE status = 'active'`), multi-tenant hot tenants, unique constraints over a subset (`CREATE UNIQUE INDEX ... WHERE active`).

### 12.2 Partial unique constraint (a pattern many engineers don't know exists)

```sql
-- Only one *active* row per user allowed; historical/inactive rows can repeat
CREATE UNIQUE INDEX idx_one_active_session ON sessions (user_id) WHERE active;
```

This replaces application-level "check then insert" races for a very common business rule ("only one active X per Y") with a single atomic database guarantee.

---

## 13. Expression Indexes

Index the **result of a function/expression**, not a raw column:

```sql
CREATE INDEX idx_lower_email ON users (lower(email));
-- Now this uses the index:
SELECT * FROM users WHERE lower(email) = 'alice@example.com';
```

### 13.1 The rule that trips people up

The query's expression must **textually match** (after parse-tree normalization) the indexed expression. `WHERE email = lower('Alice@Example.com')` will **not** use `idx_lower_email` — the index is on `lower(email)`, not `email`. This is a frequent silent-miss cause; always verify with `EXPLAIN`.

### 13.2 Common uses

- Case-insensitive lookups (`lower(col)`) — often better than `citext` for indexing purposes since it's explicit and doesn't change the column's comparison semantics everywhere.
- Indexing into JSON: `CREATE INDEX ON events ((payload->>'user_id'))`.
- Extracting date parts for reporting: `CREATE INDEX ON logs (date_trunc('day', created_at))`.
- Computed/derived business keys where you don't want a generated column.

### 13.3 Cost note

Expression indexes require re-evaluating the expression on every `INSERT`/`UPDATE` to the underlying column(s) — factor that CPU cost in for expensive expressions (e.g., heavy regex or JSON parsing) on high-write tables.

---

## 14. Unique Indexes & Constraints

A `UNIQUE` constraint **is** implemented as a unique B-Tree index — there's no separate mechanism. Key nuances:

- **NULLs are not equal to each other** by default (standard SQL semantics) — `UNIQUE (email)` allows unlimited rows with `email IS NULL`. PG 15+ added `NULLS NOT DISTINCT` if you want NULLs to collide too:
  ```sql
  CREATE UNIQUE INDEX idx_email_unique ON users (email) NULLS NOT DISTINCT;
  ```
- **Concurrency**: uniqueness is enforced at insert/update time via an index-level check plus, when needed, waiting on conflicting in-flight transactions — this is why high-contention unique columns (e.g., a global sequence-like "slug" generator under heavy concurrent insert) can become a serialization point.
- **Deferred constraints**: `UNIQUE ... DEFERRABLE INITIALLY DEFERRED` lets you violate uniqueness *within* a transaction (useful for swap/reorder operations) as long as it's resolved by `COMMIT`.
- **No cross-partition global uniqueness by default** — see [§19.1](#191-partitioned-tables).

---

## 15. Extension-Backed Index Types

### 15.1 `pg_trgm` — Trigram Fuzzy/Substring Search

Breaks text into 3-character trigrams and indexes them via GIN or GiST, enabling:
- `LIKE '%term%'` (unanchored substring — impossible to accelerate with plain B-Tree)
- `%` similarity operator and `similarity()` function for fuzzy matching
- `ORDER BY col <-> 'query'` (GiST only) for "closest match" ranking

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_name_trgm ON products USING gin (name gin_trgm_ops);
SELECT * FROM products WHERE name % 'wireles mouse';  -- typo-tolerant
```

GIN is generally preferred for read-heavy trigram search (faster lookup, larger index); GiST for write-heavy or when you need KNN-style similarity ordering.

### 15.2 `pgvector` — Similarity Search for Embeddings

The standard for AI/ML workloads in PostgreSQL. Two index types, both **approximate nearest neighbor (ANN)** — trading exactness for speed:

| | IVFFlat | HNSW |
|---|---|---|
| Structure | Clusters vectors into `lists`, searches nearest clusters | Multi-layer navigable small-world graph |
| Build time | Faster | Slower |
| Query speed/recall | Good, needs tuning `lists`/`probes` | Generally better recall at same latency |
| Needs training data present at build | Yes (clusters computed from existing rows) | No |
| Typical choice | Bulk-loaded, less write-heavy | Default recommendation for most workloads |

```sql
CREATE EXTENSION vector;
CREATE INDEX idx_embed_hnsw ON docs USING hnsw (embedding vector_cosine_ops);
```

Principal-level point: **ANN indexes are inherently lossy** — recall (percentage of true nearest neighbors actually returned) is a tunable tradeoff (`ef_search` for HNSW, `probes` for IVFFlat), not a fixed guarantee. Benchmark recall against your actual data distribution before shipping.

### 15.3 `RUM` — GIN + Position/Ranking Awareness

An out-of-core extension that extends GIN to also store lexeme *positions*, enabling `tsvector <=> tsquery` ranked ordering to use the index directly (plain GIN can filter by `@@` but must compute `ts_rank` for every matching row post-filter, since it doesn't store enough to order by rank). Worth knowing exists for FTS-heavy systems where `ORDER BY ts_rank(...) LIMIT 10` is a hot path — plain GIN can't push the `LIMIT` down without RUM.

### 15.4 `bloom` — Multi-Column Equality on Wide, Rarely-Filtered-Together Columns

A Bloom-filter index: cheap to maintain, small, supports equality on many columns in one index without needing a specific column order (unlike B-Tree's leftmost-prefix rule) — at the cost of being probabilistic (false positives require a recheck, no false negatives). Niche, but useful for ad-hoc equality filtering across, e.g., a wide events table with a dozen low-selectivity dimension columns where you can't justify a dozen separate B-Trees.

---

## 16. A Practical Process for Evaluating & Selecting an Index

Picking an AM by predicate shape ([§16.4](#164-step-4--map-predicate--data-shape-to-access-method)) is only one step in a larger process. This is the workflow to actually run when deciding whether — and what — to index, rather than guessing from experience alone.

### 16.1 Step 1 — Establish the workload signal, don't guess

Never start from intuition about "what's probably slow." Pull the real signal:

```sql
SELECT query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

`total_exec_time` (calls × mean latency) — not `mean_exec_time` alone — tells you actual production impact. A query that's slow but runs once a day matters less than a merely-mediocre query running 10,000 times/minute. For each candidate, capture the **baseline plan** before touching anything:

```sql
EXPLAIN (ANALYZE, BUFFERS) <query>;
```

### 16.2 Step 2 — Characterize the predicate shape

For every `WHERE`, `JOIN ON`, and `ORDER BY` clause in the candidate query, classify the operator:
- Equality / range / `IN` → B-Tree territory
- Containment / overlap (`@>`, `&&`) → GIN or GiST territory
- Fuzzy / substring / similarity → `pg_trgm`
- Spatial / nearest-neighbor → GiST or `pgvector`
- Prefix / hierarchical (CIDR, radix-shaped) → SP-GiST

Note which predicates **co-occur** in the same query — that determines whether a composite index is a candidate, and in what column order ([§10.3](#103-ordering-heuristics-in-priority-order)).

### 16.3 Step 3 — Characterize the data, not just the query

The same predicate shape can call for a different index depending on the data itself:

```sql
SELECT attname, n_distinct, correlation, most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

- **Cardinality/selectivity** (`n_distinct`) — low-cardinality columns rarely benefit from a plain index; consider a partial index on the rare values instead ([§12](#12-partial-indexes)).
- **Skew** (`most_common_vals`/`most_common_freqs`) — a column that's 95% one value and 5% everything else usually wants a partial index on the 5%, not a full index the planner will ignore for the common case anyway.
- **Physical correlation** (`correlation`) — near ±1.0 opens up BRIN as a viable, far cheaper option ([§8.2](#82-the-one-condition-that-makes-or-breaks-brin)); near 0, BRIN is off the table unless you're willing to `CLUSTER` first ([§9.4](#94-when-clustered-like-behavior-actually-matters)).
- **Write frequency on the candidate columns** — check `pg_stat_user_tables.n_tup_upd`/`n_tup_ins`. High-write columns push toward cheaper index shapes (partial, BRIN, smaller B-Tree) and away from expensive ones (GIN over large documents, wide GiST).

### 16.4 Step 4 — Map predicate + data shape to access method

```
Is the predicate a simple =, <, >, BETWEEN, or ORDER BY on a scalar column?
 └─ Yes → B-Tree (default choice, stop here unless key is very wide and equality-only)
 └─ No, keep going ↓

Is it equality-only on a very wide key (long text/JSON), never range/sort?
 └─ Yes → consider Hash (benchmark vs B-Tree first)

Is it containment/overlap on array, jsonb, or full-text search?
 └─ Yes → GIN (use jsonb_path_ops if only doing @>)

Is it geometric overlap, range-type overlap, KNN distance, or an exclusion constraint?
 └─ Yes → GiST

Is it hierarchical/prefix data with skewed distribution (IP/CIDR, radix-shaped)?
 └─ Yes → SP-GiST

Is the table huge, append-only, and the column correlated with physical row order?
 └─ Yes → BRIN (verify pg_stats.correlation first; CLUSTER once if it's drifted)

Is it fuzzy text match / substring / typo tolerance?
 └─ Yes → pg_trgm (GIN or GiST)

Is it vector similarity search (embeddings)?
 └─ Yes → pgvector (HNSW by default, IVFFlat for bulk-load-then-mostly-read)

Does the query only touch a small, well-defined subset of rows repeatedly?
 └─ Yes → layer a partial index on top of whichever AM above

Does the query need columns beyond the filter/sort columns?
 └─ Yes → add INCLUDE columns for an index-only scan
```

### 16.5 Step 5 — Prototype at production-representative scale

Never trust a decision made on a dev database with 1% of production's row count or a synthetic, unskewed distribution. Build the candidate index against a production-scale copy (a replica, an anonymized snapshot, or a generated dataset that matches real cardinality and skew), then compare `EXPLAIN (ANALYZE, BUFFERS)` before/after:

- Estimated `rows` vs `actual rows` — a large mismatch means the decision is being made on bad statistics, fix `ANALYZE`/`default_statistics_target` first.
- `Buffers: shared hit` vs `read` — confirms whether the new index actually fits in cache at real scale.
- Wall-clock `actual time` — the number that ultimately matters.

### 16.6 Step 6 — Price the write side, not just the read side

Every index has to justify itself against build time, on-disk size, and ongoing write overhead — not just query speedup. Run a write-heavy benchmark (`pgbench` custom script, or a replay of real traffic) with the candidate index present vs. absent, and measure the latency/throughput delta on `INSERT`/`UPDATE`. Pay particular attention to:

- Whether the new index disables HOT updates on previously-HOT-eligible columns ([§17.2](#172-hot-heap-only-tuple-updates--the-exception-to-every-index-pays-on-every-write)).
- For GIN/GiST: pending-list flush behavior under sustained write load, not just steady-state read latency ([§7.3](#73-the-write-cost-tradeoff-principal-level-knowledge)).

### 16.7 Step 7 — Roll out safely

- Build with `CREATE INDEX CONCURRENTLY` in production; verify `pg_index.indisvalid` afterward and retry on failure ([§17.1](#171-create-index-locking)).
- Ship inside a monitored window — watch replica lag (index builds generate WAL), lock contention, and confirm via `EXPLAIN` in production that the target query's plan actually changed as predicted.

### 16.8 Step 8 — Monitor and prune, indefinitely

Index decisions are not "set and forget." Add to periodic review:
- Unused-index check ([§20.1](#201-unused-indexes-pure-write-tax-no-read-benefit))
- Bloat check ([§20.2](#202-bloat-estimation))
- Redundant-index check ([§20.3](#203-duplicateredundant-indexes))

Workload and data distribution shift over time — a correct index decision today can become dead weight in a year as access patterns, data skew, or PostgreSQL version capabilities (e.g., Skip Scan) change the calculus.

---

## 17. Build & Maintenance: Locking, Bloat, REINDEX

### 17.1 `CREATE INDEX` locking

Plain `CREATE INDEX` takes `SHARE` lock on the table — blocks writes (`INSERT`/`UPDATE`/`DELETE`) for the full build duration. On a large production table this can mean minutes to hours of write-blocking.

```sql
CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);
```

`CONCURRENTLY` avoids the write lock by doing two table scans and a wait-for-transactions phase, at the cost of: 2-3x slower build, cannot run inside a transaction block, and **can fail and leave an `INVALID` index behind** — always check `pg_index.indisvalid` after, and `DROP INDEX` + retry if it failed (don't just leave a phantom invalid index, it still costs write overhead with zero read benefit).

```sql
SELECT indexrelid::regclass, indisvalid FROM pg_index WHERE NOT indisvalid;
```

### 17.2 HOT (Heap-Only Tuple) updates — the exception to "every index pays on every write"

If an `UPDATE` doesn't change **any indexed column**, and there's free space in the same heap page, PostgreSQL can perform a **HOT update**: the new tuple version is chained within the same page via a redirect, and **no index entries need to be added at all**. This is why "don't index columns you don't need to" isn't just about index size — it directly determines whether routine updates can use the cheap HOT path or must pay full index-maintenance cost on every index for every update.

```sql
-- Check HOT update ratio — low ratio on a write-heavy table suggests
-- either too many indexes, or heap pages too full (low fillfactor headroom) for HOT chaining
SELECT n_tup_upd, n_tup_hot_upd, round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd,0), 1) AS hot_pct
FROM pg_stat_user_tables WHERE relname = 'orders';
```

Lowering `fillfactor` on a heavily-updated table (e.g., `fillfactor = 80`) deliberately leaves free space per page specifically to keep HOT updates possible for longer between `VACUUM`s.

### 17.3 Index bloat

Bloat is dead space from old versions (B-Tree pages fragmented by deletes/updates that VACUUM hasn't reclaimed/recycled, or that can't be compacted back to the OS). Left unchecked it degrades both index size (cache pressure) and scan speed (more pages to traverse for the same logical data).

```sql
REINDEX INDEX CONCURRENTLY idx_orders_status;   -- PG12+, avoids the exclusive lock
```

`REINDEX CONCURRENTLY` rebuilds from scratch without blocking reads/writes (briefly blocks at the final swap), and is the standard remediation — routine `VACUUM` reclaims *some* space for reuse but doesn't shrink the on-disk index file; only a rebuild does that.

### 17.4 `autovacuum` tuning is index tuning

Stale statistics (`ANALYZE`) lead the planner to bad index-vs-seqscan decisions; stale visibility maps defeat index-only scans ([§11.2](#112-why-the-visibility-map-is-the-real-gatekeeper)); accumulated dead tuples bloat every index on the table. On high-churn tables, per-table autovacuum tuning (`autovacuum_vacuum_scale_factor`, `autovacuum_analyze_scale_factor`) is often a bigger lever than any single index change.

---

## 18. Planner Internals: Cost Model & Selectivity

### 18.1 Why the planner sometimes "refuses" to use your index

The planner picks the plan with the lowest **estimated total cost**, computed from `pg_stats` (most-common values, histogram, null fraction, correlation) — not from actually running the query. Two failure modes:

- **Stale/wrong statistics** → bad cardinality estimate → wrong plan. Fix: `ANALYZE`, or increase `default_statistics_target` / per-column `SET STATISTICS` for skewed columns.
- **Low selectivity** → if a predicate matches, say, 30% of rows, a sequential scan (large sequential I/O) genuinely can beat an index scan (many small random I/Os) — this is **correct planner behavior**, not a bug, especially on spinning disks or even on SSD once random I/O volume gets high enough relative to `random_page_cost`.

### 18.2 `random_page_cost` — the single setting that most changes index usage

Defaults to `4.0` (tuned historically for spinning disks, where random reads are ~4x sequential). On SSD/NVMe-backed systems (essentially all modern cloud databases), lowering this to `1.1`–`1.5` makes the planner's cost model reflect reality and often flips borderline seqscan-vs-indexscan decisions toward correctly using available indexes.

```sql
ALTER SYSTEM SET random_page_cost = 1.1;  -- for SSD-backed storage
```

Don't chase individual "why isn't my index used" cases with per-query hacks before checking this global setting is sane for your storage.

### 18.3 Reading `EXPLAIN (ANALYZE, BUFFERS)` for index decisions

```
Index Scan using idx_orders_status on orders  (cost=0.43..8.45 rows=12 width=97)
  (actual time=0.02..0.05 rows=11 loops=1)
  Buffers: shared hit=4
```

- `rows=12` (estimate) vs `actual rows=11` — close match means stats are trustworthy; a 100x mismatch means fix statistics before anything else.
- `Buffers: shared hit` vs `read` — `hit` means served from shared_buffers (fast); high `read` means physical I/O, worth knowing if the index even fits in cache.
- **`Index Scan` vs `Index Only Scan`** — see [§11](#11-covering-indexes-include--index-only-scans) for what determines which one you get and why it matters.

---

## 19. Indexes at Scale: Partitioning, Parallelism, Replication

### 19.1 Partitioned tables

Indexes on a partitioned (parent) table are **not physical objects** — each partition gets its own local index, and `CREATE INDEX` on the parent creates matching indexes on all partitions (and future ones). Principal-level implication: a "global unique index" across all partitions **does not exist** in core PostgreSQL — uniqueness can only be enforced per-partition unless the partition key is part of the unique key. This is a common surprise when partitioning an existing table that had a global uniqueness guarantee.

### 19.2 Parallel index scans

B-Tree supports **Parallel Index Scan** and **Parallel Bitmap Heap Scan** — multiple workers cooperatively walk the index/heap. Controlled by `max_parallel_workers_per_gather` and the planner's cost estimate of whether parallelism pays off (small result sets won't trigger it — worker startup overhead dominates). GIN/GiST have more limited parallel support historically; check current version's release notes if this matters for a specific AM.

### 19.3 Replication considerations

- All index types discussed here (as of PG10+) are WAL-logged and safely replicate to physical standbys.
- **Logical replication** replicates data, not index definitions automatically in older setups — but modern `pg_dump`/subscription workflows do carry index DDL; the operational risk is usually forgetting to build indexes on a newly-initialized logical replica *before* cutover, not a fundamental limitation.
- **Hot standby query load**: read replicas benefit from the *same* index design as the primary — a common mistake is over-indexing the primary for OLTP writes while under-indexing for the analytical/reporting queries actually routed to replicas. Index strategy should be workload-aware per role, even on the same schema.

---

## 20. Monitoring & Diagnostics

### 20.1 Unused indexes (pure write-tax, no read benefit)

```sql
SELECT s.relname AS table, s.indexrelname AS index, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan = 0 AND NOT i.indisunique  -- keep unique/PK indexes regardless of scan count
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

Caveat: on a freshly-restarted server or a replica with different query routing, `idx_scan = 0` may just mean "not enough uptime yet," not "truly unused" — check `pg_stat_user_indexes` reset time / server uptime before dropping anything.

### 20.2 Bloat estimation

```sql
SELECT schemaname, relname, indexrelname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;
```

For precise bloat (not just size), use the community `pgstattuple` extension (`pgstatindex()`) rather than eyeballing raw size — raw size alone can't distinguish "large because busy and legitimate" from "large because bloated."

### 20.3 Duplicate/redundant indexes

```sql
SELECT indrelid::regclass AS table, array_agg(indexrelid::regclass) AS redundant_indexes
FROM pg_index
GROUP BY indrelid, indkey
HAVING count(*) > 1;
```

### 20.4 Index-vs-heap size ratio, a smell test

If total index size on a table meaningfully exceeds heap size, it's worth an audit — not automatically wrong (wide tables with many legitimate access patterns exist), but frequently a sign of speculative or redundant indexing accumulated over time without pruning.

---

## 21. Anti-Patterns & Principal-Level Judgment Calls

- **Indexing every foreign key "just in case."** FK columns used only for the FK constraint's own lookup (checking parent exists) don't need an index for that purpose alone — index them when you actually query/join on them, or when you need fast `ON DELETE CASCADE` performance on the child side.
- **Adding a B-Tree on a low-cardinality column expecting it to help equality filters alone.** A `status` column with 3 values won't get picked by the planner over a seqscan/bitmap scan unless paired with other predicates or made partial ([§12](#12-partial-indexes)).
- **Reaching for GIN/GiST before checking if a simple B-Tree with the right expression/partial predicate solves it.** Exotic AMs have real build/write cost; don't pay it for a problem B-Tree already solves.
- **Ignoring `random_page_cost` and blaming the planner.** ([§18.2](#182-random_page_cost--the-single-setting-that-most-changes-index-usage))
- **Treating `CONCURRENTLY` as free.** It's slower and can leave `INVALID` indexes on failure — always verify, never fire-and-forget on a migration pipeline without a validity check step.
- **Running `CLUSTER` on an OLTP table expecting it to stay clustered.** It's a one-time snapshot that decays immediately under continued writes ([§9.3](#93-cluster--a-one-time-non-maintained-physical-reorder)) — it is not a substitute for an ongoing index strategy.
- **Not re-evaluating index strategy after a major version upgrade.** Skip Scan (PG18), dedup (PG13), `NULLS NOT DISTINCT` (PG15) — index design decisions made under PG11 assumptions can be actively suboptimal on newer versions.
- **Benchmarking index choice on a small/synthetic dataset.** Selectivity, correlation (BRIN), and cache-fit behavior only show their true character at production-representative scale and data skew — a decision that looks obviously correct on 10K rows can invert at 500M. This is precisely why [§16.5](#165-step-5--prototype-at-production-representative-scale) insists on scale-representative prototyping.
- **Forgetting indexes are schema, not an implementation detail hidden from reviewers.** Every index is a tradeoff the whole team inherits (write latency, storage, `VACUUM` load) — treat `CREATE INDEX` with the same review rigor as a schema migration, because it is one.

---

## 22. Quick Reference Tables

### 22.1 Access method summary

| AM | Best for | Supports ORDER BY | Lossy (needs recheck) | Typical extension |
|---|---|---|---|---|
| B-Tree | Equality, range, sort | Yes | No | core |
| Hash | Equality only, wide keys | No | No | core |
| GiST | Overlap, KNN, exclusion constraints | Yes (KNN `<->`) | Sometimes | core, PostGIS |
| SP-GiST | Prefix/hierarchical, skewed data | Partial | Sometimes | core |
| GIN | Containment, full-text, arrays | No | No (exact) | core, pg_trgm |
| BRIN | Huge correlated append-only data | No | Yes (always) | core |

### 22.2 Clustered vs composite, at a glance

| Concept | PostgreSQL reality |
|---|---|
| Clustered index | Doesn't exist natively — every index is "non-clustered"; `CLUSTER` gives a one-time, decaying approximation ([§9](#9-clustered-vs-non-clustered-indexes)) |
| Composite index | Same as "multicolumn index"; behavior (leftmost-prefix or not) depends on the AM, not a universal rule ([§10.4](#104-composite-index-behavior-differs-by-access-method)) |

### 22.3 Storage/behavior cheat sheet

| Concern | Lever |
|---|---|
| Index-only scan not happening | Check `Heap Fetches` in `EXPLAIN`; tune autovacuum for VM freshness |
| Write amplification too high | Check for redundant indexes; consider HOT via lower `fillfactor` |
| Planner ignoring a good index | Check `ANALYZE` freshness, `random_page_cost`, opclass match |
| Index too large | Consider partial index, BRIN (if correlated), or dedup-eligible B-Tree |
| Slow `CREATE INDEX` blocking writes | Use `CREATE INDEX CONCURRENTLY` |
| Bloated index | `REINDEX INDEX CONCURRENTLY` |
| Poor physical locality for range scans / BRIN | One-time `CLUSTER` (or `pg_repack` for an online-equivalent) |
| Fuzzy/typo-tolerant search | `pg_trgm` + GIN |
| Embedding similarity search | `pgvector` HNSW |
| "No double-booking" business rule | GiST exclusion constraint |
| Global replica read load pattern differs from primary | Index per role, not just per schema |

---

## 23. Composite Index Storage & Query Mechanics — A Worked Walkthrough

[§10](#10-composite-multicolumn-indexes--column-ordering) states the leftmost-prefix rule as a rule. This section shows *why* it's true, down to the physical layout — and walks through what actually happens on disk for a read and for a write. Everything here is specific to **B-Tree** (the default AM); GIN/GiST/BRIN store composite keys differently ([§10.4](#104-composite-index-behavior-differs-by-access-method)).

### 23.1 A composite key is one concatenated, sortable value — not three separate columns

When you write:

```sql
CREATE INDEX idx_orders_composite ON orders (customer_id, status, created_at);
```

PostgreSQL does not build three indexes and intersect them. It builds **one B-Tree whose key is the tuple `(customer_id, status, created_at)`**, compared the same way you'd sort a spreadsheet with three sort levels: sort by `customer_id` first; for rows where `customer_id` ties, sort by `status`; for rows where both tie, sort by `created_at`. Every leaf entry is `(customer_id, status, created_at, ctid)`, physically stored as one comparable unit.

### 23.2 Worked example: sample data and its leaf-page layout

```sql
CREATE TABLE orders (
  id           bigint PRIMARY KEY,
  customer_id  int,
  status       text,
  created_at   timestamptz
);

CREATE INDEX idx_orders_composite ON orders (customer_id, status, created_at);
```

Sample rows (unordered, as they might physically sit in the heap):

| id | customer_id | status    | created_at |
|----|-------------|-----------|------------|
| 1  | 5           | pending   | 09:00 |
| 2  | 5           | shipped   | 08:00 |
| 3  | 5           | pending   | 10:00 |
| 4  | 7           | pending   | 09:30 |
| 5  | 5           | cancelled | 07:00 |
| 6  | 7           | shipped   | 11:00 |

The **index leaf page** stores these same rows, but physically reordered by the full composite key — `customer_id` first, then `status` (alphabetically), then `created_at`:

```
Leaf page (sorted order):
┌──────────────┬────────────┬────────────┬──────┐
│ customer_id  │ status     │ created_at │ ctid │
├──────────────┼────────────┼────────────┼──────┤
│ 5            │ cancelled  │ 07:00      │ →id5 │
│ 5            │ pending    │ 09:00      │ →id1 │  ← customer_id=5 block
│ 5            │ pending    │ 10:00      │ →id3 │
│ 5            │ shipped    │ 08:00      │ →id2 │
│ 7            │ pending    │ 09:30      │ →id4 │  ← customer_id=7 block
│ 7            │ shipped    │ 11:00      │ →id6 │
└──────────────┴────────────┴────────────┴──────┘
```

Notice: within `customer_id = 5`, entries are grouped further by `status`, and only *within* a `(customer_id, status)` pair are they sorted by `created_at`. This nesting is the entire mechanism behind the leftmost-prefix rule — it's not a policy PostgreSQL enforces, it's a direct consequence of how a multi-key sort physically arranges data.

### 23.3 How a query traverses this structure

**Query A — all three columns, leading to trailing:**
```sql
WHERE customer_id = 5 AND status = 'pending' AND created_at > '09:30'
```
The tree descent uses `customer_id = 5` to jump straight to that block, then `status = 'pending'` narrows to the two-row sub-range within it, then `created_at > '09:30'` bounds the scan to just the `10:00` row. All three predicates become **Index Cond** — they directly shrink which pages/entries get visited. This is the fastest possible case: an O(log n) descent plus a tiny, tight scan.

**Query B — leading column only:**
```sql
WHERE customer_id = 5
```
Descend to the start of the `customer_id = 5` block, scan forward until `customer_id` changes to `7`. Still efficient — bounded on both sides — even though `status`/`created_at` aren't filtered, because the *block itself* is contiguous.

**Query C — skips the leading column:**
```sql
WHERE status = 'pending'
```
Look at the leaf layout above: rows with `status = 'pending'` (`id1`, `id3`, `id4`) are **not adjacent** — they're scattered across the `customer_id = 5` and `customer_id = 7` blocks. There is no single contiguous range to jump to, so (pre-PG18, or when Skip Scan doesn't kick in) PostgreSQL must walk the *entire* index checking every entry's `status` — no better than a full index scan, and the planner will usually prefer a sequential scan instead. This is the leftmost-prefix rule made visible: **the physical layout, not an arbitrary restriction, is what makes this query un-indexable by this key order.**

**Query D — leading column + trailing column, middle column skipped:**
```sql
WHERE customer_id = 5 AND created_at > '08:30'
```
`customer_id = 5` narrows the descent to that block (**Index Cond**) — but *within* that block, entries are sorted by `status` first, so `created_at` values are **not** contiguous (`09:00`, `10:00`, `08:00` in leaf order — not increasing). PostgreSQL can't binary-search further; it must walk every entry in the `customer_id = 5` block and test `created_at > '08:30'` on each one as a **Filter**, not an Index Cond. It's still much cheaper than Query C (bounded to one customer's rows, not the whole table), but strictly worse than Query A.

### 23.4 Reading the difference in `EXPLAIN`

```
Index Scan using idx_orders_composite on orders
  Index Cond: (customer_id = 5)
  Filter: (created_at > '08:30')
  Rows Removed by Filter: 1
```

**`Index Cond`** = used to physically bound the B-Tree traversal — cheap, no wasted row visits. **`Filter`** = evaluated after fetching the entry from the index page but did *not* narrow which pages/entries were visited — every entry in the bounded range still gets pulled and tested. `Rows Removed by Filter` > 0 is your empirical signal that a column in your composite key isn't earning its position — worth re-checking against [§10.3](#103-ordering-heuristics-in-priority-order)'s ordering heuristics.

### 23.5 What happens physically on `INSERT`

1. PostgreSQL assembles the new row's indexed columns into the composite key, e.g. `(5, 'pending', '11:30')`.
2. It descends the B-Tree from the root, using key comparison at each internal page to choose which child pointer to follow — exactly the same comparison logic as the query traversal above.
3. The leaf page it lands on gets the new `(key, ctid)` entry inserted at the correct **sorted position** (not appended — it must stay sorted for future binary search to work).
4. **If the leaf page is full**, PostgreSQL performs a **page split**: roughly half the entries move to a newly allocated page, and a separator key is pushed up into the parent page. If the parent is also full, the split cascades upward — in the rare worst case, all the way to the root, which increases the tree's height by one level. This is why bulk-loading a table and then building the index (`CREATE INDEX` after `COPY`) is far cheaper than inserting rows one at a time into a pre-existing index — the former builds the tree bottom-up in one pass; the latter pays for splits incrementally.
5. **Deduplication (PG13+, [§3.3](#33-deduplication-pg-13)):** if the new key is identical to its neighbor's key (same `customer_id`, `status`, and `created_at`), PostgreSQL may merge it into a shared posting list instead of storing the full key twice.

### 23.6 What happens physically on `UPDATE`

This is the step most engineers get wrong intuitively. Two cases:

**Case 1 — none of the composite index's columns change (e.g., you update an un-indexed `notes` column) and there's free space on the heap page.** This qualifies for a **HOT update** ([§17.2](#172-hot-heap-only-tuple-updates--the-exception-to-every-index-pays-on-every-write)): a new row version is chained within the same heap page, and **the index is not touched at all** — no descent, no new entry, nothing.

**Case 2 — any indexed column changes (e.g., `status` goes from `'pending'` to `'shipped'`).** The composite key's sort position depends on *all* its columns, so changing even one means the row's correct location in the tree has changed — potentially to a completely different leaf page. PostgreSQL cannot update the entry "in place"; it must:
- Leave the old index entry as-is for now (it still correctly points to the old, now-dead heap tuple version — needed for any transaction whose snapshot still sees that old version).
- Perform a fresh **insert** of the new key `(5, 'shipped', '11:30')` at its new sorted position, pointing to the new heap tuple's `ctid` — following the exact same descent-and-possibly-split process as [§23.5](#235-what-happens-physically-on-insert).
- The old, now-orphaned entry becomes garbage, later marked `LP_DEAD` opportunistically during scans, and physically reclaimed by the next `VACUUM`.

**The practical consequence for column ordering:** if the **leading** column of a composite index is the one that changes frequently (e.g., a `status` column that cycles through many states), every such update relocates the entry to a potentially distant, unrelated part of the tree — worse cache locality, more random I/O, more page splits spread across the index. If the volatile column is placed **last** in the key instead, an update that only changes it still requires a delete+insert (Case 2 always does), but the new position is typically near the old one — same or an adjacent leaf page — which is cheaper in practice. This is a write-locality reason to order composite keys carefully, independent of the read-side ordering heuristics in [§10.3](#103-ordering-heuristics-in-priority-order).

### 23.7 What happens physically on `DELETE`

`DELETE` doesn't touch the index immediately either. The heap tuple's `xmax` is set to the deleting transaction's ID — the row is logically dead but physically present, because other transactions with an older snapshot may still need to see it (MVCC). The index entry pointing to it is left alone until:
- An index scan opportunistically notices the pointed-to tuple is dead-to-everyone and marks the index entry `LP_DEAD`, or
- `VACUUM` runs its index cleanup pass, which removes entries whose heap tuple is confirmed dead to all possible snapshots, then reclaims the corresponding heap space.

This is why a burst of deletes doesn't shrink an index right away — the space is only reclaimed for reuse (not returned to the OS) once `VACUUM` catches up, and only a `REINDEX` ([§17.3](#173-index-bloat)) actually shrinks the on-disk file.

### 23.8 Summary: the full lifecycle in one table

| Operation | Index touched? | What happens |
|---|---|---|
| `INSERT` | Always | Descend tree by full composite key → insert at sorted position in leaf → possible page split |
| `UPDATE`, no indexed column changed, room on page | No (HOT) | New tuple chained in-page; index entries untouched |
| `UPDATE`, an indexed column changed | Yes | Old entry left as-is (later reclaimed); new entry inserted at its new sorted position — same cost/path as an `INSERT` |
| `DELETE` | Deferred | `xmax` set on heap tuple; index entry marked dead lazily (scan or `VACUUM`), physically removed by `VACUUM` |
| Bulk load then `CREATE INDEX` | N/A | Tree built bottom-up in one sorted pass — far cheaper than incremental inserts into a pre-existing index |

---

## Further Reading

- PostgreSQL official docs: [Index Types](https://www.postgresql.org/docs/current/indexes-types.html), [GIN](https://www.postgresql.org/docs/current/gin.html), [GiST](https://www.postgresql.org/docs/current/gist.html), [BRIN](https://www.postgresql.org/docs/current/brin.html), [CLUSTER](https://www.postgresql.org/docs/current/sql-cluster.html)
- `pg_trgm`, `btree_gist`, `pgvector`, `pgstattuple`, `pg_repack` extension docs
- Companion doc in this repo: `PostgreSQL_Presentation_Script.md` (internals, MVCC, page structure, planner scan strategies)
