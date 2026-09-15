# PostgreSQL — Internals, Advanced Indexing, Scan Strategies & MVCC
### Presenter Script & Summary Guide

> **Audience:** Mid-Senior to Staff/Principal Engineers
> **Goal:** By the end of this talk, everyone in the room should be able to look at a slow query, understand *why* PostgreSQL made the plan it did, and know exactly which lever to pull to fix it.

---

## Opening (Slide 1–2)

Start by setting expectations. This is not a "how to use PostgreSQL" talk — it is a "how PostgreSQL works under the hood" talk. The difference matters because you can only tune what you understand.

Walk through the four parts quickly:
- **Part 1** gives us the physical foundation — pages, memory, and writes.
- **Part 2** covers every index type and when each one is the right tool.
- **Part 3** explains how the planner *chooses* between scan strategies, and how to read `EXPLAIN`.
- **Part 4** goes deep on MVCC — the hidden cost behind every update, and how to manage it.

---

## Part 1 — PostgreSQL Internals & Architecture

### 1.1 The 8 kB Page (Slides 4–5)

Everything in PostgreSQL — every table row, every index entry — lives inside an **8 kB page** (also called a block). This is the smallest unit PostgreSQL reads from or writes to disk. You cannot read half a page; you always pay the full 8 kB.

**Inside a page, from top to bottom:**

```
+------------------------------------------------+
|  Page Header  (24 bytes)                       |
|   LSN · checksum · pd_lower · pd_upper         |
+------------------------------------------------+
|  Line Pointer Array  (Item IDs)                |
|   [1] -> offset, length, flags                 |
|   [2] -> offset, length, flags                 |
|   [3] -> DEAD / UNUSED                         |
+------------------------------------------------+
|              Free Space                        |
+------------------------------------------------+
|  Tuple Data  (rows, bottom-up)                 |
|   HeapTupleHeader | Column values              |
+------------------------------------------------+
|  Special Space  (B-Tree sibling pointers, …)   |
+------------------------------------------------+
```

The critical design insight is the **two-direction growth**: Line Pointers grow downward from the header; Tuples grow upward from the bottom. Free space sits in the middle. When they meet, the page is full.

**The most important fact about updates:** In PostgreSQL, `UPDATE` is not an in-place modification. It is a `DELETE + INSERT`. The old row is marked dead (its `xmax` is set), and the new row is appended at the bottom of the page — or on a new page if there is no room. The old row stays on disk until `VACUUM` cleans it up.

This is not a bug; it is the intentional price of MVCC concurrency. But it has deep consequences for performance.

**Line Pointers never move.** Even if a tuple is physically shifted inside the page during a defragmentation operation, its line-pointer index (1, 2, 3…) stays the same. This is why indexes can safely store `ctid = (page 42, offset 3)` — the offset is a pointer into the line pointer array, not a raw byte offset.

---

### 1.2 MVCC — Hidden Columns on Every Row (Slide 6)

Every single row in PostgreSQL carries three invisible system columns you never see in `SELECT *`:

| Column | Meaning |
|--------|---------|
| `xmin` | Transaction ID (XID) that **inserted** this row |
| `xmax` | XID that **deleted or updated** this row. `0` means the row is alive |
| `ctid`  | Physical location `(block #, offset #)` — always points to the **newest version** |

When your query runs, PostgreSQL takes a **snapshot** of the current transaction state. For every tuple it encounters during a scan, it checks: "Is `xmin` committed and visible to my snapshot? Is `xmax` zero or not yet visible to me?" Only if both conditions pass does the row appear in your results.

**The write amplification problem:** When you `UPDATE` a row, the new version gets a new physical address — a new `ctid`. Every index that points to the old `ctid` must now be updated to point to the new one. If you have 5 indexes on a table, one `UPDATE` causes 5 index page writes plus 1 heap page write. This is write amplification, and it is a key driver of I/O pressure on write-heavy systems.

---

### 1.3 Memory Architecture — Shared vs Local (Slide 7)

PostgreSQL memory is split into two completely separate worlds:

**Shared Memory (global, all connections share it)**

- **Shared Buffers** — the database's L1 cache. When PostgreSQL reads a page from disk, it lands here. Subsequent reads of the same page are served from RAM. Uses a Clock-Sweep algorithm (a variation of LRU). Rule of thumb: set to ~25% of total RAM.
- **WAL Buffers** — a temporary staging area for Write-Ahead Log records before they are flushed to disk.
- **Lock Space** — tracks who holds which lock on which object.

**Local Memory (per backend process — every connection has its own)**

- **`work_mem`** — memory available for in-memory sorts and hash joins. **Critical warning: this is per operation, not per query and not per connection.** A single query with 5 joins can allocate `5 × work_mem` simultaneously.
- **`maintenance_work_mem`** — used by `VACUUM`, `CREATE INDEX`, `REINDEX`. Safe to set much higher than `work_mem`.
- **`temp_buffers`** — for temporary tables within a session.

---

### 1.4 The Life of a Write — WAL & Checkpoints (Slide 8)

This is one of the most misunderstood mechanisms in PostgreSQL. Walk through it step by step:

1. **Backend receives `INSERT`** — it locks the target 8 kB page in Shared Buffers.
2. **WAL Buffer write (memory)** — the change is written to the WAL Buffer in RAM. The page in Shared Buffers is now marked *dirty*.
3. **`COMMIT`** — the WAL Buffer is flushed to the WAL Segment file on disk via `fsync`. **At this point, "success" is returned to the client. The actual data page is still only in RAM — it has NOT been written to the heap file yet.**
4. **Checkpoint (background, every 5–15 minutes)** — the Checkpointer process wakes up, flushes all dirty pages from Shared Buffers to the actual data files, and marks the WAL as recyclable.

**The key insight:** PostgreSQL separates *durability* from *persistence*. The WAL on disk is enough to guarantee durability — if the server crashes, PostgreSQL replays the WAL to reconstruct any unflushed dirty pages. The heap file is just an optimized cache of the authoritative WAL log.

---

### 1.5 Real-World War Story — The work_mem OOM Kill (Slide 9)

This is a real incident worth telling in full because it is a classic mistake:

> A developer set `work_mem = 128 MB` globally because the server had 64 GB of RAM. During a busy period with 200 active connections, each running complex reports with multiple sorts, the kernel OOM killer fired and crashed the primary database.

**The math that explains it:**
```
200 connections × 2 complex sorts each × 128 MB = ~50 GB
+ OS overhead + Shared Buffers ≈ total RAM exhausted → CRASH
```

**The fix:** Keep global `work_mem` conservative — something like 16 MB. For the reporting service that actually needs more:
```sql
ALTER ROLE report_bot SET work_mem = '1GB';
```

Only the `report_bot` role gets the high allocation. Regular application connections stay safe.

---

## Part 2 — Advanced Indexing

### 2.1 Index Types Overview (Slide 11)

Before diving into each type, establish the mental model: an index is a separate data structure that trades disk space and write performance for faster reads. There is no free lunch — every index you add slows down writes. The question is always: *does the read benefit justify the write cost?*

PostgreSQL offers six index types:

| Index | Best For |
|-------|----------|
| **B-Tree** | Equality, ranges, ORDER BY, LIMIT — the universal default |
| **GIN** | Composite values: JSONB, arrays, full-text search |
| **GiST** | Geometric data, range types, trigram similarity |
| **BRIN** | Massive append-only tables with natural sort order |
| **Hash** | Pure equality lookups only |
| **SP-GiST** | Space-partitioned data: IP addresses, URLs, points |

---

### 2.2 B-Tree — The Default (Slides 12–13)

The B-Tree is a balanced, multi-level structure of 8 kB pages organized into a hierarchy:

```
          [ Root Page ]
         /      |      \
  [Branch]  [Branch]  [Branch]    ← Keys + Pointers
   / | \    / | \    / | \
[Leaf][Leaf][Leaf][Leaf][Leaf]    ← Keys + ctid pointers (sorted)
  |
  +-- Doubly-linked list  ←  enables forward AND backward scans
```

**Why B-Trees stay shallow:** A single 8 kB page holds roughly 100 pointers (fan-out). A 3-level tree can index 100³ = 1,000,000 rows. A 4-level tree handles 100,000,000 rows. In practice, most B-Trees in production are only 3–5 levels deep, even for billions of rows. Finding a specific row requires just 3–5 I/O operations, and most of those are already cached in Shared Buffers.

**Write amplification — Page Splits:** When a B-Tree leaf page fills up, PostgreSQL must split it into two. The parent branch must then be updated to point to the new page. If the branch is also full, it splits too, cascading up to the root. If you have 10 indexes on a table, a single `INSERT` could trigger 10 different B-Tree page splits across disk. This is why having too many indexes on a high-write table is a real problem.

**Fillfactor:** The antidote to excessive page splits. Setting `fillfactor = 90` leaves 10% empty space in every leaf page, delaying splits on write-heavy workloads. This also directly enables HOT updates (covered in Part 4).

**The free ORDER BY:** Because B-Tree leaves are sorted and doubly-linked, walking the tree from left-to-right gives you rows in ascending order for free. `ORDER BY col DESC` walks right-to-left. No sort operation needed in the query plan — this is enormous for pagination APIs.

---

### 2.3 GIN — Generalized Inverted Index (Slides 14–15)

GIN is fundamentally different from B-Tree. A B-Tree maps one key to many rows. GIN maps one *element* to many rows. It is designed for data where one row contains multiple "keys" — like a JSONB document with dozens of fields, or a text field with hundreds of words.

**Internal structure:**
```
Entry Tree (a B-Tree over individual keys/words/elements)
  Key: "database"  →  Posting List: [TID(1,3), TID(45,7), TID(900,2)]
  Key: "postgres"  →  Posting Tree: [TID(1,3), TID(12,5), TID(55,1)...]
```

For each unique element in the indexed data, GIN maintains a *posting list* (or posting tree if it grows large) of all row locations containing that element.

**Use cases:** `JSONB` (`@>`, `?`, `?|`), `arrays` (`@>`, `<@`, `&&`), full-text search (`@@`).

**The write performance trap:** Inserting one row into a table with a GIN index may require updating 50 or more posting lists — one for each word in a text field or key in a JSONB document. This makes GIN index maintenance significantly more expensive than B-Tree. PostgreSQL buffers new writes into a *Pending List* in memory (`fastupdate = on` by default) and flushes them in bulk. But at 10,000 inserts/second on a large JSONB table, GIN can become the bottleneck — and it competes with your data for Shared Buffer cache space.

---

### 2.4 GIN vs GiST Trade-off (Slide 16)

The question of when to use GIN vs GiST comes down to one fundamental difference:

| | GIN | GiST |
|-|-----|------|
| Precision | **Exact** (non-lossy) | Often **lossy** (bounding predicate) |
| Heap recheck | Never needed | Always needed |
| Read speed | Extremely fast | Slower |
| Write speed | Very slow | Faster |
| Index size | Large | Smaller |
| Best for | JSONB, arrays, tsvector | Geometry, ranges, trigram ILIKE |

GiST uses "bounding predicates" — each internal node approximates everything in its subtree (like a bounding box for spatial data). This means it can produce false positives, requiring a heap recheck. In `EXPLAIN`, you will see `Rows Removed by Index Recheck` when GiST is being lossy.

**Rule of thumb:** If you need to search JSON or arrays and read performance is paramount → GIN. If you need spatial queries, range overlaps, or trigram similarity (`ILIKE '%pattern%'`) → GiST.

---

### 2.5 BRIN — The Secret Weapon for Big Data (Slides 17–18)

BRIN (Block Range INdex) is radically different from every other index. Instead of indexing individual rows, it divides the entire heap file into ranges of 128 pages (~1 MB each) and stores just two values per range: the **minimum** and **maximum** value found in that block.

```
Range 0  (pages   0–127):  min = 2025-01-01, max = 2025-01-05
Range 1  (pages 128–255):  min = 2025-01-06, max = 2025-01-10
Range 2  (pages 256–383):  min = 2025-01-11, max = 2025-01-15

Query: WHERE created_at = '2025-01-07'
  Range 0 → does not overlap → SKIP (128 pages saved!)
  Range 1 → overlaps         → scan + recheck
  Range 2 → does not overlap → SKIP
```

**The size advantage is staggering:** A BRIN index on a 1 TB append-only table is approximately 10 MB. A B-Tree index on the same column would be around 500 GB. Even if BRIN is 2× slower than B-Tree for queries, it saves 499.99 GB of expensive NVMe storage and Shared Buffer cache.

**When it works — and when it fails:**
- ✅ Works perfectly on **append-only, time-ordered data**: logs, IoT sensor readings, event streams — data that is naturally written in chronological order, so each block range covers a narrow time window.
- ❌ Completely useless for **random data** like UUIDs: every 128-page range will have `min ≈ 0` and `max ≈ infinity`, so no ranges can ever be skipped.

**Real-world result:** A 5 TB time-series table replaced an 800 GB B-Tree index with a 200 MB BRIN index. Inserts became 5–10× faster; range queries became 3–20× faster.

---

### 2.6 Hash Index (Slide 19)

Hash indexes are simple: compute a 32-bit hash of the column value, locate a bucket, store the TID. Average lookup is O(1).

**Use when:** You have a very high-cardinality column (UUIDs, long strings) and your queries are exclusively equality (`=`) lookups. The Hash index will be smaller than a B-Tree because it stores only the 4-byte hash code, not the original value.

**Do not use when:** You need range scans, ORDER BY, or you're on PostgreSQL < 10 (Hash indexes were only made crash-safe in PG 10). In most other situations, B-Tree is safer and more versatile.

---

### 2.7 Partial Indexes — Index Only What You Need (Slide 20)

A partial index adds a `WHERE` clause to the index definition. PostgreSQL only includes rows matching that predicate. The index is physically smaller, faster to scan, and cheaper to maintain on writes.

**Classic patterns:**
```sql
-- Job queue: 10 M rows, 99% are 'completed' — don't index them
CREATE INDEX idx_jobs_pending ON jobs (created_at)
WHERE status = 'pending';
-- Result: index drops from 2 GB → 5 MB

-- Soft deletes
CREATE INDEX idx_users_active ON users (email)
WHERE deleted_at IS NULL;

-- Partial UNIQUE constraint (powerful!)
CREATE UNIQUE INDEX idx_users_active_email
ON users (email)
WHERE is_deleted = false AND status = 'active';
-- Allows multiple soft-deleted rows with same email, enforces uniqueness on live ones
```

**Always ask during code review:** "Can this index be partial? What percentage of rows will actually be queried through this index?"

---

### 2.8 Expression (Functional) Indexes (Slide 21)

If your query filters on a *transformed* value — `lower(email)`, `date_trunc('day', created_at)`, `profile ->> 'email'` — a standard B-Tree on the raw column is useless. PostgreSQL must compute the function for every row it scans.

The solution is to **index the expression itself**:

```sql
CREATE INDEX idx_users_lower_email ON users ( (lower(email)) );
-- Now this query is an Index Scan:
SELECT * FROM users WHERE lower(email) = 'john.doe@example.com';
```

PostgreSQL pre-computes the expression on every insert and update, storing the result in the index. At query time, no recomputation is needed.

**Requirement:** The function must be `IMMUTABLE` (same input always produces same output, no side effects). PostgreSQL enforces this — you cannot index a `VOLATILE` or `STABLE` function.

**Combining partial + expression** is one of the most powerful patterns available:
```sql
CREATE INDEX idx_active_lower_username
ON users ( (lower(username)) )
WHERE is_active = true AND deleted_at IS NULL;
```

---

### 2.9 Composite Indexes — The Left-to-Right Rule (Slide 22)

A composite index on `(a, b, c)` is like a phone book sorted first by last name, then first name within each last name group. You can look up "Smith, John" or all "Smiths", but you cannot efficiently look up all "Johns" without knowing the last name.

**The planner can use the index for:**
- `WHERE a = ?`
- `WHERE a = ? AND b = ?`
- `WHERE a = ? AND b > ?` — equality on prefix, range on next column
- `ORDER BY a, b, c` — free sort from index traversal

**The planner CANNOT efficiently use it for:**
- `WHERE b = ?` — skips the leading column
- `WHERE c = ?` — even worse

**Column order guideline:** Put equality columns first (prefer higher cardinality). Put the range column last.

**The redundant index trap** — always check during PR review:
```
Index 1: (last_name, first_name)   ← already covers last_name prefix
Index 2: (last_name)               ← completely redundant, delete it
```
Every redundant index wastes disk space and adds write overhead for zero read benefit.

---

## Part 3 — Scan Strategies & Query Planning

This part answers the question every engineer eventually asks: *"I created an index — why isn't PostgreSQL using it?"*

### 3.1 The Four Scan Strategies at a Glance (Slide 24)

Before the planner can choose a scan strategy, it estimates **selectivity**: the fraction of rows that match your `WHERE` clause. That estimate, combined with cost calculations, determines which path wins.

| Strategy | Selectivity Sweet Spot | EXPLAIN Node |
|----------|----------------------|--------------|
| **Sequential Scan** | > ~20% of table, or small tables | `Seq Scan` |
| **Index Scan** | < ~1% of table | `Index Scan` |
| **Bitmap Index Scan** | ~1–20% of table, OR queries | `Bitmap Heap Scan` + `Bitmap Index Scan` |
| **Index-Only Scan** | < ~1%, all columns in index, VM all-visible | `Index Only Scan` |

---

### 3.2 Sequential Scan — When the "Dumb" Scan Wins (Slide 25)

A Sequential Scan reads every 8 kB page in the heap file from page 0 to the last page, checking every tuple against the `WHERE` condition. It is O(N pages).

For 1 million rows: ~8,334 pages × 8 kB = **~65 MB of I/O** to find one row. This sounds terrible — and for point lookups it is. But for queries that need a large fraction of the table, it wins because:

1. **Sequential I/O is fast.** The OS can prefetch pages in order. Reading 8,334 pages sequentially is much faster than 8,334 random jumps.
2. **An index would be slower here.** If `WHERE status = 'active'` matches 90% of the table, an index would send PostgreSQL on 900,000 random heap page jumps, visiting most of the same pages as the sequential scan but in an expensive random order.

**Key EXPLAIN signals for Seq Scan:**
- `Rows Removed by Filter: 999,999` → one row returned from a million scanned — a strong hint that an index would help.
- `Buffers: shared hit = 8334` → every page touched.

**NVMe note:** On modern NVMe SSDs the sequential vs random I/O gap narrows significantly. Set `random_page_cost = 1.1` to tell the planner that random I/O is barely more expensive than sequential — it will start preferring index scans more aggressively.

---

### 3.3 Index Scan — Random I/O Per Row (Slide 26)

An Index Scan traverses the B-Tree to find the matching `ctid`, then jumps to that exact location in the heap to fetch the row, then returns to the index for the next pointer, and so on.

```
B-Tree leaf                     Heap (table)
  key=500  ctid=(42,3)  ──────►  page 42   (1 random I/O)
  key=501  ctid=(99,1)  ──────►  page 99   (1 random I/O)
  key=502  ctid=(7,5)   ──────►  page 7    (1 random I/O)
```

**The hidden cost:** 1,000 matching rows = up to 1,000 random heap page fetches. PostgreSQL accounts for this using `random_page_cost` (default 4.0) — meaning each random fetch is assumed to cost 4× a sequential page read. This is why the planner will choose a Sequential Scan over an Index Scan when the result set is moderately large.

**Index Scan wins when:** selectivity is very low (< ~1%) — few enough random jumps that the total I/O is less than a full sequential scan.

---

### 3.4 Bitmap Index Scan — The I/O Optimizer (Slides 27–28)

When the result set is too large for an Index Scan (too many random I/Os) but too small to justify a Sequential Scan, PostgreSQL uses the **two-phase Bitmap approach**:

**Phase 1 — Build a bitmap (index side):**
Scan the B-Tree (or GIN/BRIN) and collect all matching `ctid`s into an in-memory bitmap organized by page number:
```
page 7   → offsets [2, 5, 9]
page 42  → offsets [1, 3]
page 99  → offsets [7]
```

**Phase 2 — Heap scan (heap side):**
Sort the pages and visit each one exactly once, in order:
```
Visit page  7 once → check offsets 2, 5, 9
Visit page 42 once → check offsets 1, 3
Visit page 99 once → check offset 7
```

This converts random I/O into nearly-sequential I/O. Each heap page is read at most once.

**LOSSY mode — the `work_mem` trap:** The bitmap is built in `work_mem`. If there are too many matching rows, the bitmap exceeds `work_mem` and switches to LOSSY mode: instead of tracking individual row offsets, it only tracks which pages contain matches. Every row on those pages must then be rechecked. In `EXPLAIN`, you will see `Rows Removed by Index Recheck` — that is your signal that `work_mem` is too small.

**BitmapOr / BitmapAnd:** The real power of Bitmap scans comes when you have OR conditions. `WHERE email = ? OR username = ?` becomes two separate Bitmap Index Scans whose results are merged with `BitmapOr` — giving you efficient use of two separate single-column indexes for a query that a composite index could not serve.

---

### 3.5 Index-Only Scan — The Performance Holy Grail (Slides 29–30)

The Index-Only Scan is what every performance-conscious engineer aims for on hot read paths: the query is answered entirely from the index leaves, without ever touching the heap file.

**How it works:**
```
Normal Index Scan:
  B-Tree leaf → ctid → heap page → visibility check → return row

Index-Only Scan:
  B-Tree leaf → check Visibility Map → if all-visible → return from index!
```

**The Visibility Map (VM)** is a companion file with one bit per heap page. When a page's bit is set to "all-visible", it means every tuple on that page is old enough to be visible to all current transactions — so PostgreSQL can skip the heap entirely and trust the data in the index.

**Two requirements (both must hold):**
1. Every column in `SELECT` and `WHERE` must be present in the index.
2. The Visibility Map marks the relevant pages as "all-visible".

**Creating a Covering Index:**
```sql
CREATE INDEX idx_user_email ON users(email) INCLUDE (user_id, name);
-- Now this is an Index-Only Scan:
SELECT user_id, name FROM users WHERE email = 'alice@example.com';
```

The `INCLUDE` columns are stored in leaf pages but not in the tree keys — they do not affect sort order but allow Index-Only Scans for queries that need those columns.

**Diagnosing health:**
```
Heap Fetches: 0       ← perfect Index-Only Scan
Heap Fetches: 50000   ← VM is stale, run VACUUM ANALYZE
```

**HOT + IOS synergy:** HOT updates keep new tuple versions on the same page. VACUUM can mark that page "all-visible" quickly. Non-HOT updates scatter dead tuples across many pages, resetting the VM bits and forcing IOS to fall back to heap fetches.

---

### 3.6 The Query Planner — How it Picks a Plan (Slides 31–33)

Before any query executes, it passes through five stages:

```
Parser → Analyzer → Rewriter → Planner → Executor
```

The **Planner** is the brain. It generates multiple candidate execution paths and calculates the *cost* of each one. Cost is not measured in seconds — it is a dimensionless unit where `1.0 = cost of reading one 8 kB page sequentially from disk`.

**The cost formula:**
```
Total Cost  =  I/O Cost  +  CPU Cost
I/O Cost    =  pages × seq_page_cost   (default: 1.0)
CPU Cost    =  tuples × cpu_tuple_cost (default: 0.01)
```

**The critical parameter: `random_page_cost`** (default 4.0)

This single parameter has more influence on plan choice than almost anything else. It tells the planner: "a random disk read costs 4× a sequential read." This was accurate for spinning hard drives. On modern NVMe SSDs, the ratio is closer to 1.1–1.2.

**Tuning for modern hardware:**
```
random_page_cost = 1.1   # SSD / NVMe — use indexes much more aggressively
effective_cache_size = 48GB  # 75% of 64 GB RAM — planner assumes index pages are in OS cache
```

**Plan stability risk — the "flip-flopping plan":** When the estimated cost of a Sequential Scan and an Index Scan are nearly equal (e.g., 1000.41 vs 1000.42), a tiny change in table statistics after an `ANALYZE` can flip the plan from one to the other overnight. This is how you get a query that was fast for months and suddenly becomes slow after routine maintenance.

**Reading `EXPLAIN (ANALYZE, BUFFERS)` like a Staff Engineer:**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 123 AND order_date > '2025-01-01';

-- Output:
Index Scan using idx_orders_cust_date on orders
  (cost=0.56..42.10 rows=15 width=64)
  (actual time=0.031..0.089 rows=12 loops=1)
  Buffers: shared hit=5 read=1
```

What each field tells you:

| Field | What it means | Red flag if... |
|-------|--------------|----------------|
| `cost=0.56..42.10` | Startup cost .. total cost (planner estimate) | Very high relative to table size |
| `rows=15` vs `actual rows=12` | Estimate vs reality | They diverge by 10×+ → stale statistics |
| `shared hit=5` | Pages served from RAM (Shared Buffers) | Low ratio of hit vs read |
| `read=1` | Pages fetched from disk | High → cold cache or index too large |
| `loops=1000` | Inner node of Nested Loop ran 1000 times | Danger: watch for this in complex joins |

**If `rows estimate` diverges widely from `actual rows` → run `ANALYZE` immediately.** The planner is making decisions based on a stale picture of your data.

**Operator switches (debugging only — never leave in production):**
```sql
SET enable_seqscan = off;     -- force the planner away from Seq Scan
SET enable_indexscan = off;   -- force away from Index Scan
SET enable_bitmapscan = off;  -- force away from Bitmap Scan
```
Use these to understand *why* the planner chose what it did, then re-enable.

---

### 3.7 Scan Strategy Decision Tree (Slide 34)

Summarize the planner's logic into a mental model everyone can remember:

```
Selectivity = estimated matching rows / total rows

> ~20%  OR  small table
    → Sequential Scan  (sequential I/O is cheaper)

< ~1%  AND  all needed columns in index  AND  VM all-visible
    → Index-Only Scan  (zero heap I/O — best case)

< ~1%  AND  index exists  (VM not all-visible)
    → Index Scan  (few random I/Os — still fast)

1–20%  OR  OR query needing multi-index merge
    → Bitmap Index Scan  (sort ctids → sequential-ish heap reads)
```

> **These thresholds are not fixed.** They shift based on `random_page_cost`, table size, Shared Buffer cache pressure, and statistics accuracy. The planner does the math; we set up the conditions for success.

---

## Part 4 — MVCC Deep Dive: HOT Updates & Index Bloat

### 4.1 MVCC — The Hidden Tax Revisited (Slide 36)

Now that we understand pages and scans, we can fully appreciate the cost of MVCC:

- `INSERT` → new tuple with `xmin = current txid`, `xmax = 0`
- `UPDATE` → new tuple (new `xmin`) + old tuple gets `xmax = current txid`
- `DELETE` → old tuple gets `xmax = current txid`

Dead tuples — those with `xmax` set — are not immediately removed. They persist on disk until `VACUUM` comes along. This means:

1. **Table bloat**: The heap file grows even if the logical table size stays the same.
2. **Index bloat**: Every index entry for a dead tuple also becomes dead, but `VACUUM` cannot shrink the index pages — it can only mark entries as reusable.
3. **Scan overhead**: Both Sequential Scans and Index Scans must still inspect dead tuples and verify their visibility, adding CPU work.

---

### 4.2 HOT Updates — The Optimization That Changes Everything (Slides 37–38)

HOT (Heap-Only Tuple) is one of the most important optimizations in PostgreSQL. When a HOT update occurs, **no indexes are touched at all**.

**Normal UPDATE (non-HOT):**
```
All indexes  →  updated to point to new ctid  ← expensive
LP1 (old)                LP2 (new, different page or offset)
```

**HOT UPDATE:**
```
Index       →  still points to LP1  ← UNCHANGED
LP1 (old)  ──ctid chain──►  LP2 (new, same page)
```

The index still points to the old line pointer. When an index scan follows that pointer and finds LP1 is dead, it follows the `ctid` chain to find the current version — all within the same heap page. No index write occurred.

**Two strict requirements:**
1. The new tuple must fit on the **same 8 kB page** as the old one (free space must exist).
2. The updated column must **not** be part of any index.

**`fillfactor` is the key enabler.** If you set `fillfactor = 80` on a table, every page is only 80% full after writes. That 20% headroom is exactly where HOT update's new tuples land.

```sql
CREATE TABLE orders (...) WITH (fillfactor = 80);
ALTER TABLE orders SET (fillfactor = 80);  -- for existing tables
```

**What kills HOT:**
- You indexed a column that gets updated frequently — e.g., `last_login_at`, `view_count`, `status`.
- The page is full because `fillfactor = 100` (the default) leaves no room.

**Measuring HOT health:**
```sql
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(n_tup_hot_upd::numeric / nullif(n_tup_upd,0) * 100, 1) AS hot_pct
FROM pg_stat_user_tables ORDER BY n_tup_upd DESC;
```
- `hot_pct` near 100% → HOT is working well, minimal index bloat.
- `hot_pct` near 0% → every update is writing to all indexes → bloat accumulating.

---

### 4.3 Index Bloat — The Ghost in the Machine (Slides 39–40)

Even with VACUUM running regularly, indexes silently accumulate bloat:

- Every non-HOT UPDATE creates a new index entry and marks the old one dead.
- `VACUUM` can prune dead entries but **cannot reclaim the space** — B-Tree pages remain the same size, just partially empty.
- Over time: fragmented pages, half-empty leaves, a taller tree, more I/O for every scan.

**Normal bloat levels:** 30–70% in active B-Trees is considered acceptable. Above 100–200% — the index is so fragmented it needs a rebuild.

**The compounding effect:** A bloated 500 GB index must all fit in Shared Buffers to be efficient. If it doesn't, PostgreSQL starts swapping index pages from disk on every query — evicting your actual data from cache. This degrades the entire cluster, not just queries on that table.

**Detection with `pgstattuple`:**
```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstatindex('idx_orders_customer_date');
-- Look at avg_leaf_density
-- 80-90% → healthy
-- 20-30% → index is 70-80% empty air → rebuild needed
```

**Fix — `REINDEX CONCURRENTLY` (PG 12+):**
```sql
REINDEX INDEX CONCURRENTLY idx_orders_customer_date;
```
This builds a fresh, compact index in the background without taking any table lock. Reads and writes continue normally during the rebuild.

---

### 4.4 Transaction Isolation & MVCC (Slide 41)

Three isolation levels, each trading consistency guarantees for concurrency:

| Level | Snapshot taken | Risk |
|-------|---------------|------|
| **Read Committed** (default) | Per statement | Non-repeatable reads |
| **Repeatable Read** | At transaction start | Write skew (serialization anomalies) |
| **Serializable** | Full serial emulation | Serialization errors (must retry) |

**The double-booking problem** illustrates why isolation level matters:

```
T1: SELECT rooms WHERE date = '2025-06-01' AND status = 'available'  → room 101
T2: SELECT rooms WHERE date = '2025-06-01' AND status = 'available'  → room 101
T1: INSERT booking (room 101, user A)   ← success
T2: INSERT booking (room 101, user B)   ← success too! Overbooking!
```

- **Fix A (Pessimistic):** `SELECT ... FOR UPDATE` — locks the rows at read time. Prevents the race but creates serialization bottlenecks under high concurrent load.
- **Fix B (Senior approach):** Use `SERIALIZABLE` isolation. PostgreSQL detects the dependency conflict and throws `ERROR: could not serialize access`. Your application catches it and retries. This accepts that ~1% of transactions will retry in exchange for no locking overhead.

---

### 4.5 Operations — Preventing XID Wraparound (Slide 42)

This is the one scenario that can **shut down your production database with no warning** if you ignore it.

**What is it?** PostgreSQL uses 32-bit Transaction IDs (XIDs). At 2 billion XIDs, the counter wraps around — tuples that look "in the future" from the new counter's perspective become invisible. PostgreSQL prevents this data corruption by shutting the database down hard at 2 billion.

**Monitor it:**
```sql
SELECT max(age(datfrozenxid)) FROM pg_database;
-- Alert at > 1.5 billion — gives you headroom to respond
```

**Root cause of slow `VACUUM` progress:** A long-running transaction holds an open snapshot. `AUTOVACUUM` cannot advance `relfrozenxid` past that snapshot, so the XID age keeps climbing.

**Fix checklist:**
1. Identify and kill the long-running transaction (`pg_stat_activity`).
2. Increase `autovacuum_vacuum_cost_limit` to let VACUUM run faster.
3. Manually run `VACUUM FREEZE` on tables with the highest age.

**`pg_stat_statements`** — mention this briefly as the single most valuable production tool:
- Which query consumes the most cumulative execution time?
- Which query has the highest standard deviation in execution time (inconsistent, potentially flip-flopping plan)?

---

## Closing — The Principal Engineer Checklist (Slide 43)

Bring it all together with the four-category checklist that should be running in the back of every engineer's mind when designing or reviewing database-touching code:

### Internals
- Know the 8 kB page layout — header, line pointers, tuples, special space.
- Understand the WAL → dirty page → checkpoint flow. Know *why* the client gets success before the page hits disk.
- Keep `work_mem` globally conservative; raise per-role for specific workloads.

### Indexing
- **B-Tree:** Left-to-right column order. Set `fillfactor < 100` on write-heavy tables. Use `INCLUDE` for covering indexes. Eliminate redundant indexes.
- **GIN:** Great for JSONB/arrays reads; expensive writes. Watch for buffer cache competition.
- **BRIN:** The right choice for large, append-only, time-ordered tables.
- **Partial indexes:** Always ask "can this be smaller?" before creating a full-table index.

### Scan Strategies
- Sequential Scan wins when selectivity is high; don't fight the planner blindly.
- Index Scan for < 1%; Bitmap for 1–20%; Index-Only Scan as the design goal for hot read paths.
- On NVMe: set `random_page_cost = 1.1`. Set `effective_cache_size` to 50–75% of RAM.
- `EXPLAIN (ANALYZE, BUFFERS)` is your truth. A large gap between estimated and actual rows = run `ANALYZE`.

### MVCC
- Design schemas to maximize HOT: keep frequently-updated columns out of indexes.
- Monitor `n_tup_hot_upd` ratio. Monitor `avg_leaf_density`.
- When index bloat exceeds 100–200%: `REINDEX INDEX CONCURRENTLY`.
- Watch XID age. Keep autovacuum healthy. Kill long-running idle transactions.

---

## References (Slide 44)

**Books (recommended reading order):**
1. *The Internals of PostgreSQL* — Hironobu Suzuki *(free online — read chapters 1–3 first)*
2. *PostgreSQL High Performance* — Gregory Smith
3. *Designing Data-Intensive Applications* — Martin Kleppmann *(Chapters 3 & 7 for transaction theory)*
4. *The Art of PostgreSQL* — Dimitri Fontaine

**Online:**
- Use The Index, Luke — `markuswinand.com` *(best free resource on index internals)*
- PostgreSQL Documentation: Chapter 73 (Page Layout), Chapter 14 (Performance Tips), Chapter 26 (Backup and Restore)

**Hands-on labs to try:**
1. Compare Seq Scan vs Index Scan cost by varying `random_page_cost` from 4.0 to 1.1.
2. Run `EXPLAIN BUFFERS`; force plan changes with `SET enable_seqscan = off`.
3. Generate a LOSSY Bitmap Scan by reducing `work_mem` to 64kB before a wide query.
4. Generate index bloat via a mass `UPDATE`; measure with `pgstatindex`; fix with `REINDEX CONCURRENTLY`.
5. Compare B-Tree vs BRIN index size on a 10 M-row time-series table — the size difference will be eye-opening.

---

*Presentation covers 44 slides across 4 parts. Estimated delivery time: 60–90 minutes with Q&A.*
