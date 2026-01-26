# PostgreSQL Indexing: From Junior to Principal

## Table of Contents

### 1️⃣ Fundamentals (Junior → Mid)

- [Lesson 1: The Bare Metal — Sequential Scans](#lesson-1-the-bare-metal--sequential-scans)
- [Lesson 2: PostgreSQL B-Tree Index — From First Principles to Internals](#lesson-2-postgresql-b-tree-index--from-first-principles-to-internals)
- [Lesson 3: Selectivity, Cardinality & The Stats Collector](#lesson-3-selectivity-cardinality--the-stats-collector)
- [Lesson 4: Index Scan vs. Bitmap Index Scan vs. Index Only Scan](#lesson-4-index-scan-vs-bitmap-index-scan-vs-index-only-scan)
- [Lesson 5: How WHERE, ORDER BY, and GROUP BY Influence Index Choice](#lesson-5-how-where-order-by-and-group-by-influence-index-choice)

### 2️⃣ Intermediate (Senior Level)

- [Lesson 6: Deep Dive into GIN & GiST](#lesson-6-deep-dive-into-gin--gist)
- [Lesson 7: BRIN, Hash, and SP-GiST](#lesson-7-brin-hash-and-sp-gist)
- [Lesson 8: Multi-column, Partial, and Expression Indexes](#lesson-8-multi-column-partial-and-expression-indexes)

### 3️⃣ Advanced (Staff Level)

- [Lesson 9: PostgreSQL Internal Structure (Pages, Tuples, Pointers)](#lesson-9-postgresql-internal-structure-pages-tuples-pointers)
- [Lesson 10: Advanced - MVCC, HOT Updates, and Index Bloat](#lesson-10-advanced---mvcc-hot-updates-and-index-bloat)

### 4️⃣ Expert / Principal Level

- [Lesson 11: Query Planner Internals & Cost Estimation](#lesson-11-query-planner-internals--cost-estimation)
- [Lesson 12: High-scale Indexing Strategies (Multi-tenant & Time-series)](#lesson-12-high-scale-indexing-strategies-multi-tenant--time-series)

---

# Lesson 1: The Bare Metal — Sequential Scans

Before we talk about B-Trees or GIN indexes, we must understand the "default" behavior. If Postgres doesn't have an index, it performs a Sequential Scan (often seen as Seq Scan in plans).

## To understand why this is slow (or sometimes surprisingly fast), we need to look at how data is stored.

## 1. Internal Storage: Pages and Blocks

PostgreSQL does not store rows individually. It organizes data into Pages (also called Blocks), which are typically 8KB in size.

### The Storage Hierarchy:

1. Relation (Table): A collection of files on disk.
2. Page (8KB): The unit of I/O. When Postgres reads a row, it must load the entire 8KB page into memory (the Buffer Cache).
3. Tuple (Row): The actual data stored within the page.

### Heap Page(Page):

#### Structure:

Important insights:

- Line pointers grow top → down
- Tuple data grows bottom → up
- Free space is in the middle

```
+--------------------------------------------------+
| Page Header                                      |
|  - LSN (WAL): (WAL position).                    |
|  - checksum                                      |
|  - flags : (PD_ALL_VISIBLE, etc.)                |
|  - free space pointers                           |
+--------------------------------------------------+
| Line Pointer Array (ItemIdData[])                |
|  [1] -> tuple offset, length, flags              |
|  [2] -> tuple offset, length, flags              |
|  [3] -> DEAD / UNUSED                            |
+--------------------------------------------------+
|                                                  |
|                Free Space                        |
|                                                  |
+--------------------------------------------------+
| Tuple Data (rows stored from bottom upward)      |
|  - HeapTupleHeader                               |
|  - Column values                                 |
+--------------------------------------------------+
```

#### Line Pointers (ItemIdData):

Each line pointer is 4 bytes:
| Field | Meaning |
| ------ | ------------------------ |
| offset | Byte offset to tuple |
| length | Tuple length |
| flags | NORMAL / DEAD / REDIRECT |

Why this matters:

- Indexes point to (block, offset_number)
- Offset number = line pointer index, not byte offset
- VACUUM can:
  - Mark DEAD
  - Redirect
  - Reuse slots without changing index entries

[!]PostgreSQL trades extra indirection for massive concurrency and MVCC flexibility.

#### Tuple Header Internals:

Each tuple begins with:
| Field | Purpose |
| ----------- | ----------------------- |
| xmin | Creating transaction |
| xmax | Deleting transaction |
| infomask | Visibility flags |
| infomask2 | HOT, lock info |
| ctid | Pointer to next version |
| null bitmap | Column NULLs |
| data | Column values |

---

## 2. How a Sequential Scan Works

When you run SELECT \* FROM users WHERE age = 30 without an index:

1. The Storage Manager starts at Page 0.
2. It loads Page 0 into the Buffer Cache.
3. It iterates through every Tuple in that page, checking the WHERE condition.
4. It moves to Page 1, Page 2, and so on, until the end of the file.

[!IMPORTANT] A Sequential Scan is an O(N) operation, where N is the number of pages. It is "dumb" because it must discard data only after reading it from disk and bringing it into CPU registers.

## 3. Real SQL Example: The Cost of "No Index"

Let's look at a table with 1 million rows.

```sql
-- Setup
CREATE TABLE large_table AS
SELECT generate_series(1, 1000000) AS id, md5(random()::text) AS val;

-- Query without index
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table WHERE id = 500000;
```

### Potential Output Analysis:

```text
QUERY PLAN
---------------------------------------------------------------------------------------------
Seq Scan on large_table  (cost=0.00..18334.00 rows=1 width=37) (actual time=45.123..88.456 rows=1 loops=1)
  Filter: (id = 500000)
  Rows Removed by Filter: 999999
  Buffers: shared hit=8334
Planning Time: 0.123 ms
Execution Time: 88.500 ms
```

### Reading like a Staff Engineer:

- cost=0.00..18334.00: The first number is the startup cost. The second is the estimated total cost based on I/O + CPU.
- Buffers: shared hit=8334: This is critical. It tells us Postgres had to read 8,334 pages to find one row. Since each page is 8KB, it processed ~65MB of data to find a single integer.
- Rows Removed by Filter: 999999: This is the "waste" metric. High numbers here relative to the result set are a major red flag.

---

## 4. Staff/Principal-Level Insights

### When is a Seq Scan "Better" than an Index?

A common junior mistake is thinking "Indexes are always faster." If you are querying SELECT \* FROM users WHERE status = 'active' and 90% of the table is 'active', an index would actually be slower.

Why?

- Sequential I/O (Seq Scan) is much faster for hardware than Random I/O (Index Scan).
- If the database knows it's going to hit almost every page anyway, it's faster to just stream the file linearly than to jump back and forth between the Index (one file) and the Heap (another file).

### The CPU vs. I/O Trade-off:

In modern systems with NVMe SSDs, the bottleneck is often the CPU cost of evaluating the Filter rather than the disk I/O. Postgres spends a lot of cycles decoding the tuples within the page.

---

## 🧭 Thinking Question

If a table has 1,000,000 rows but they all fit into one single 8KB page (impossible, but imagine), would a Sequential Scan be slow? Why or why not?

```text
Short answer:
👉 No, a Sequential Scan would be extremely fast in that (imaginary) scenario.

* Sequential Scan cost is dominated by I/O (page reads), not row count.
* PostgreSQL scans pages, not rows.

What really happens in a Sequential Scan

Normally:
1. PostgreSQL reads data page by page
2. Each page is 8 KB
3. Each page may contain many tuples
4. Cost ≈ number of pages read + per-tuple CPU cost

Your hypothetical scenario
* 1,000,000 rows
* All rows fit into 1 single 8 KB page
* Sequential Scan behavior:
  * Read 1 page
  * Iterate over 1,000,000 tuples in memory

I/O cost:
  * Exactly one page read
  * Likely already in shared buffers
  * Disk cost ≈ near zero

CPU cost:
  * Looping over 1M tuples in memory
  * Very fast compared to disk I/O
  * Modern CPUs handle this easily

➡️ Result: Blazing fast sequential scan

Compare with an Index Scan in this scenario
  * An index scan would be worse:
  * Read index pages
  * Read heap page (same page)
  * Extra pointer chasing
  * Extra random access overhead

➡️ Index scan adds work without reducing I/O.
```

## When you're ready, tell me your thoughts on the question, and we'll move to B-Trees.

# Lesson 2: PostgreSQL B-Tree Index — From First Principles to Internals

In Lesson 1, we saw that a Sequential Scan is O(N). To find one row in a million, we might read 65MB. A B-Tree (Balanced Tree) changes the game to O(log N).

## To a Staff Engineer, a B-Tree isn't just a "fast search"; it's a multi-level map of sorted pointers.

## 1. First Principles: The Power of Sorting

If I give you a phone book (remember those?) and ask you to find "Zyxel", you don't start at page 1. You jump to the middle, then the end. This is Binary Search.

A B-Tree is essentially a persistent, disk-optimized version of Binary Search.

## 2. The Internal Structure

Postgres B-Trees are made of 8KB pages, just like tables. But they are organized into a hierarchy:

```
       [ Root Page ]
      /      |      \
 [Branch] [Branch] [Branch]  <-- Internal Pages
  / | \    / | \    / | \
[Leaf][Leaf][Leaf][Leaf][Leaf] <-- Leaf Pages (Sorted)
```

### A. Root & Branch Pages (The Map)

These pages don't contain your data (like username or email). They contain:

- Keys: The value you indexed (e.g., id = 500).
- Pointers: The address of the next page down in the tree.

### B. Leaf Pages (The Destination)

This is the bottom layer. These pages contain:

- Keys: The actual data values in sorted order.
- CTID (Item Pointer): The physical address of the row in the table (Heap). It looks like (BlockNumber, Offset).

---

## 3. Why B-Trees are "Shallow" (The Fan-out)

A single 8KB page can hold hundreds of pointers. This is called Fan-out.

If a page holds 100 pointers, a 3-level tree can index 1,000,000 items ($100^3$).
A 4-level tree can index 100,000,000 items.
[!TIP] This is why most B-Trees in production are only 3 to 5 levels deep, even for billions of rows. Finding a specific row requires only 3–5 I/O operations (and most of those are cached in RAM).

---

## 4. Real SQL Example: Seeing the B-Tree in Action

```sql
-- Create an index
CREATE INDEX idx_large_table_id ON large_table(id);

-- Query WITH index
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table WHERE id = 500000;
```

Expected Output Analysis:

```text
QUERY PLAN
---------------------------------------------------------------------------------------------
Index Scan using idx_large_table_id on large_table  (cost=0.42..8.44 rows=1 width=37)
  Index Cond: (id = 500000)
  Buffers: shared hit=4
```

Reading like a Staff Engineer:

- shared hit=4: Compare this to the 8,334 buffers from Lesson 1. We found the row by reading only 4 pages (likely: 1 Root, 1 Branch, 1 Leaf, and 1 Heap page). This is a 2,000x improvement.
- Index Cond: This means the tree was traversed. If you see Filter under an index scan, it's a sign of a bad index or a complex condition.

---

## 5. Staff/Principal-Level Insights

### Write Amplification & Page Splits

The "B" in B-Tree stands for Balanced. Every leaf must be at the same depth. When you insert a row into a "full" leaf page:

Postgres must split that page into two.
It must then update the parent (branch) page to point to the new page.
If the branch is full, it splits too.

The Cost: Inserting into a table with 10 indexes is significantly slower because one INSERT might trigger 10 different B-Tree page splits across the disk.

### The "Fillfactor" Optimization

For high-write tables, we often set a fillfactor < 100 (e.g., 90). This leaves 10% empty space in each B-Tree page for future inserts, delaying expensive page splits.

---

## 6. Practice Exercises 🏋️

### 1. Check Tree Depth: Use the pgstatindex extension (CREATE EXTENSION pgstattuple;) to check the tree_level of an index.

```sql
SELECT * FROM pgstatindex('idx_large_table_id');
```

### 2. Order Matters: Research why B-Trees are great for WHERE x > 10 AND x < 20 (Range Scans) compared to Hash indexes.

---

## 🧭 Thinking Question

In the EXPLAIN output above, we saw Buffers: shared hit=4. If a B-Tree has a depth of 3, why are there 4 hits?

```text
Because the EXPLAIN Buffers count includes both index pages and heap pages.

A B-tree depth of 3 explains index page reads only, but an Index Scan also reads at least one heap page.
So:
    3 index pages (root + internal + leaf)
    + 1 heap page
    = 4 shared buffer hits
```

Depth = number of index levels from root to leaf:

```
Level 1: Root page
Level 2: Internal page
Level 3: Leaf page
```

Post your answer or any questions, and we'll move to Lesson 3: Selectivity & Cardinality.

---

# Lesson 3: Selectivity, Cardinality & The Stats Collector

In Lesson 2, we saw how fast B-Trees are. But here is the Staff-level truth: Postgres often ignores your indexes.

To understand why, we have to look into the "Brain" of the database: the Query Planner and the Statistics Collector.

---

## 1. The Core Concepts: Selectivity & Cardinality

### Selectivity (The "Ratio")

Selectivity is the percentage of rows that match your filter.

- Formula: (Rows matching filter) / (Total rows)
- Low Selectivity (Good for Index): Finding 1 custom ID in 1,000,000 (0.0001%).
- High Selectivity (Bad for Index): Finding all "Active" users where 90% are active.

### Cardinality (The "Uniqueness")

Cardinality is the number of distinct values in a column.

- High Cardinality: UUID, Email, ID. (Great for indexing).
- Low Cardinality: Gender, Status, True/False. (Terrible for standard B-Trees).

---

## 2. The Statistics Collector: How Postgres "Guesses"

Postgres doesn't count rows in real-time when you run a query (that would be too slow). Instead, it looks at samples stored in pg_statistic (viewed via pg_stats).

### What's inside pg_stats?

1. n_distinct: Estimated number of unique values.
2. most_common_vals (MCV): The values that appear most frequently.
3. most_common_freqs (MCF): How often those values appear.
4. histogram_bounds: A distribution of values that aren't in the MCV list.

### Real SQL Example: Looking inside the Brain

```sql
-- See statistics for our table
SELECT
    column_name,
    n_distinct,
    most_common_vals,
    most_common_freqs
FROM pg_stats
WHERE tablename = 'large_table';
```

---

## 3. The Role of ANALYZE

Statistics are only updated when you run ANALYZE (or when autovacuum triggers it).

[!WARNING] Stale Statistics are the #1 cause of slow queries in production. If you delete 90% of a table but the stats haven't updated, the planner will still think the table is huge and choose bad plans.

---

## 4. Staff/Principal-Level Insights

### The "Default" Selectivity

When Postgres doesn't have statistics (new column, no analyze), it uses hardcoded defaults. For example:

- column = 'value' defaults to 0.5% selectivity.
- column > 'value' defaults to 33.3% selectivity.
  As a Staff Engineer, if you see an EXPLAIN plan that estimates exactly rows=10 or rows=1000 on a multi-million row table, it's a huge hint that stats are missing.

### Correlated Statistics

Postgres stats are usually calculated per column. It doesn't know that city='San Francisco' and zip_code='94105' are highly correlated. This often leads the planner to underestimate rows. Solution: In advanced cases, we use Extended Statistics (CREATE STATISTICS).

---

## 5. Practice Exercises 🏋️

1. Verify Stats Update: Create a table, insert 10,000 rows, and check pg_stats. Then delete half the rows and check pg_stats before and after running ANALYZE.

```text
PostgreSQL không biết dữ liệu đã thay đổi cho đến khi ANALYZE chạy.

Planner:
Không đếm row thật
Chỉ tin vào thống kê gần nhất

❗ Vì vậy:
Autovacuum bị lag → kế hoạch sai
DELETE / UPDATE nhiều → phải chú ý ANALYZE
```

2. The "Impossible" Filter: Query a column with a value that doesn't exist. Look at EXPLAIN. Does Postgres estimate 0 or 1? (Hint: It almost never estimates 0).

```text
PostgreSQL cố tình không bao giờ ước tính 0 rows.
Lý do thiết kế (cực kỳ quan trọng)
1️⃣ Thống kê chỉ là mẫu (sampling)
ANALYZE không scan toàn bảng
Nó lấy mẫu
Giá trị “không thấy trong mẫu” ≠ “không tồn tại”
Planner không được phép chắc chắn

2️⃣ Giá trị có thể xuất hiện trong tương lai
Concurrent INSERT
Transaction chưa commit
Partition khác
HOT update
Planner không có quyền giả định “impossible”

3️⃣ Ước tính 0 sẽ phá planner
Nếu planner tin là 0:
Có thể loại bỏ entire plan branch
Join order sai
Nested loop bị tối ưu hóa sai
Execution plan trở nên không an toàn
➡️ PostgreSQL chọn an toàn > chính xác tuyệt đối

PostgreSQL làm gì thay thế?
Planner dùng minimum selectivity
Nên:
Bảng 10.000 rows → ước tính ≈ 1 row
Bảng 1.000.000 rows → ≈ 1 row
📌 1 là con số “an toàn”

Tổng kết ngắn gọn:
Planner chỉ tin thống kê, không tin dữ liệu thật
Postgres gần như không bao giờ ước tính 0 rows — vì an toàn

=> PostgreSQL tối ưu cho tính đúng và ổn định, không phải cho sự thông minh tuyệt đối.
```

---

### 🧭 Thinking Question

Imagine a column status where 99% of rows are PROCESSED and 1% are PENDING. If you have a B-Tree index on status, will Postgres use it for WHERE status = 'PROCESSED'? What about WHERE status = 'PENDING'? Why?

#### Answer

| Query                              | Will Postgres use the B-tree index? | Why                                   |
| ---------------------------------- | ----------------------------------- | ------------------------------------- |
| `WHERE status = 'PROCESSED'` (99%) | ❌ **Usually NO**                   | Too many rows → too many heap fetches |
| `WHERE status = 'PENDING'` (1%)    | ✅ **Yes**                          | Highly selective → few heap fetches   |

```
Case 1: status = 'PROCESSED' (99%)
What an Index Scan would do
  1. Traverse B-tree (cheap)
  2. Find almost every index entry
  3. Follow millions of CTIDs
  4. Fetch almost all heap pages, randomly

That means:
  * Random I/O
  * Many heap page visits
  * Poor cache locality

What a Sequential Scan does
  1. Read heap pages sequentially
  2. Filter rows in memory

Sequential I/O is:
  * Cheaper
  * Prefetch-friendly
  * Cache-friendly

Planner’s conclusion
    Index Scan cost  >>  Seq Scan cost
👉 Sequential Scan wins
```

```
Case 2: status = 'PENDING' (1%)
Index Scan behavior
  1. Traverse B-tree
  2. Find few index entries
  3. Fetch very few heap pages
Cost is:
  * Few random reads
  * Much less work than scanning whole table

Planner’s conclusion
    Index Scan cost  <<  Seq Scan cost
👉 Index Scan wins
```

## Post your answer, and we'll move to Lesson 4: Index Scan vs Bitmap Scan vs Index Only Scan.

# Lesson 4: Index Scan vs. Bitmap Index Scan vs. Index Only Scan

Once Postgres decides an index is worth using (thanks to the statistics we learned in Lesson 3), it has to choose how to use it. There are three primary mechanisms, and as a Staff Engineer, you must know exactly which one is happening and why.

---

## 1. Index Scan (The Bread and Butter)

This is the simplest way.

- Process: Postgres traverses the B-Tree to find a pointer, jumps to the Table (Heap) to get the row, goes back to the index for the next pointer, jumps back to the table, and so on.
- When it's used: When the result set is very small (low selectivity).
- The Pitfall: If there are 1,000 matches, Postgres might perform 1,000 random I/O jumps. Random I/O is the most expensive operation in a database.

## 2. Bitmap Index Scan (The "I/O Optimizer")

If Postgres needs to fetch a few thousand rows, jumping back and forth is too slow. It switches to a Bitmap Scan.

- Process:

1. Bitmap Index Scan: It scans the index and marks the location of rows in a "Bitmap" (a map of which pages contain data).
2. Bitmap Heap Scan: It sorts those locations by page number and visits each table page only once in physical order.

- Why it's better: It converts random I/O into sequential-ish I/O. It's much faster for mechanical disks and even helps NVMe SSDs via read-ahead.

```text
QUERY PLAN
-------------------------------------------------------------------------
Bitmap Heap Scan on large_table  (cost=12.43..567.89 rows=1000)
  Recheck Cond: (status = 'PENDING')
  ->  Bitmap Index Scan on idx_status (cost=0.00..12.34 rows=1000)
        Index Cond: (status = 'PENDING')
```

---

## 3. Index Only Scan (The Performance Holy Grail)

This is what every Principal Engineer aims for in high-performance paths.

- Process: The query is answered entirely using the data inside the B-Tree. Postgres never touches the table (Heap).
- Requirement: Every column in your SELECT and WHERE clauses must be included in the index.

### The Catch: The Visibility Map

Wait! B-Tree indexes don't store MVCC information (which transaction can see which row). So how does Postgres know if the row in the index is actually visible to your query?

- Solution: It checks the Visibility Map.
- If the visibility map says a page is "all visible" (meaning all rows in that page are old enough for everyone to see), it skips the Heap.
- If the page has recently modified rows, it must jump to the Heap to check visibility.
  [!IMPORTANT] If your Index Only Scan shows a high number of Heap Fetches in EXPLAIN ANALYZE, it means your VACUUM isn't running often enough to update the Visibility Map!

---

## 4. Staff/Principal-Level Insights

### The "Overhead" of Bitmap Scans

A Bitmap Scan requires memory (work_mem). If the bitmap is too large to fit in memory, it becomes "Lossy"—instead of tracking individual rows, it tracks entire pages. This forces Postgres to re-check Every row in those pages, adding CPU overhead.

### Strategic Covering Indexes

You can force an Index Only Scan by adding columns to an index that aren't for searching, but just for data retrieval, using the INCLUDE keyword:

```sql
CREATE INDEX idx_user_email_id ON users(email) INCLUDE (user_id);
-- Now (SELECT user_id FROM users WHERE email = '...') is an Index Only Scan!
```

---

## 🧭 Thinking Question

If you have a query SELECT \* FROM users WHERE age = 30, is it possible to ever get an Index Only Scan? Why or why not?

## When you're ready, we'll head into Lesson 5: How WHERE, ORDER BY, and JOIN influence index choice.

## 5. Practice Exercises 🏋️

- The INCLUDE Test: Create two indexes: one standard and one with an INCLUDE column. Run a SELECT that only asks for that column and see if it switches from Index Scan to Index Only Scan.
- The Vacuum Proof: Perform a massive UPDATE on a table. Run an Index Only Scan and look for Heap Fetches. Then run VACUUM ANALYZE and run the query again. The fetches should drop to 0.

---

# Lesson 5: How WHERE, ORDER BY, and GROUP BY Influence Index Choice

In previous lessons, we focused on finding rows. But a Staff Engineer knows that Indexes are also for Sorting. Because B-Trees are kept in sorted order, they can completely eliminate the need for expensive memory-resident sorts.

---

## 1. The "Free" Order By

When you run SELECT \* FROM users ORDER BY created_at, Postgres has two choices:

1. The Hard Way: Read the whole table and sort it in memory (or on disk if it's large). You will see a Sort node in your plan.
2. The Smart Way: Walk the B-Tree index on created_at from left to right. The rows come out sorted "for free."

### Forward vs. Backward Scans

B-Trees are doubly-linked lists at the leaf level. This means Postgres can read them forward or backward with equal efficiency.

- ORDER BY created_at ASC -> Index Scan
- ORDER BY created_at DESC -> Index Scan Backward

---

## 2. The "Top-N" Optimization (LIMIT)

This is where indexes shine in production APIs. If you have 10 million rows but only want the 10 newest ones:

- Without Index: Postgres must read and sort all 10 million rows just to find the top 10. O(N log N).
- With Index: Postgres jumps to the rightmost leaf of the B-Tree and reads 10 items. O(log N).

```text
QUERY PLAN
---------------------------------------------------------------------------------
Limit  (cost=0.42..1.56 rows=10 width=37)
  ->  Index Scan Backward on idx_created_at (cost=0.42..45678.00 rows=10000000)
```

---

## 3. Group By: Hash vs. Group Aggregate

When you group data (GROUP BY category), Postgres chooses between:

1. HashAggregate: It builds a Hash Table in memory (work_mem). Great for many small groups.
2. GroupAggregate: It expects the data to be already sorted.
   If you have an index on the grouping column, Postgres can use GroupAggregate which uses very little memory and is extremely fast for huge datasets because it just processes the sorted stream.

---

## 4. Multi-clause Interaction

What happens if you have both? WHERE status = 'active' ORDER BY created_at. To get an Index Scan (no sort), you need a Composite Index on (status, created_at).

- If you index (created_at, status), Postgres can't use the index for both. It would have to scan the whole tree for the ORDER BY.
- If you index (status, created_at), Postgres finds all 'active' rows (which are grouped together in the tree) and notices they are already sorted by created_at within that group. Victory.

---

## 5. Staff/Principal-Level Insights

### The "Price" of the Index Scan

The Optimizer is smart. It knows that while an Index Scan avoids a sort, it might involve Random I/O (jumping to the Heap). If the Optimizer thinks it will have to read 50% of the table anyway, it might choose a Seq Scan + Sort because Sequential I/O + Memory Sort is often faster than Random I/O.

### Avoid ORDER BY random()

You've probably seen this for "Featured Products." It is a performance killer because it forces a full table scan and a full sort every single time.

---

## 6. Practice Exercises 🏋️

1. The Sort Battle: Create a table with 100k rows. Run a query with ORDER BY id (assuming id has an index). Then run it with ORDER BY (id + 0). The + 0 breaks the index usage. Compare the plans.
2. Limit Magic: Run a query with a heavy ORDER BY and no index, once with LIMIT 10 and once without. Look at the Sort Method. Is it quicksort or top-N heapsort?

## 🧭 Thinking Question

You have an index on (last_name, first_name). Will this index help with ORDER BY first_name? What about WHERE last_name = 'Smith' ORDER BY first_name?

Post your answer, and we'll move to Lesson 6: Intermediate - Deep Dive into GIN & GiST.

---

# Lesson 6: Deep Dive into GIN & GiST

Everything we've learned so far has been about B-Trees. But B-Trees have a weakness: they are designed to find one value or a range of values.

What if you want to find "any row that contains the word 'Postgres' inside a 1MB text field"? Or "any row where a JSONB object contains the key 'admin'"? For this, we need Non-B-Tree indexes.

---

## 1. GIN (Generalized Inverted Index)

GIN works by breaking down indexed values into atomic "keys" and storing inverted mappings: key → list of row locations (TID: tuple IDs).

GIN (Generalized Inverted Index) = index cho 1 row → nhiều values
Khác B-Tree:

- B-Tree: 1 key → nhiều rows
- GIN: 1 value → nhiều rows

### The Internal Mechanism: Postings Lists

- Instead of a tree where each leaf points to one row, a GIN index has an entry for every distinct "item" (word, array element, JSON key).
- That entry points to a Postings List: a list of all row IDs (CTIDs/Tuple ID) containing that item.

Entry tree (B-Tree-like):

- Key = element (vd: "postgres")
- Value = pointer đến posting list / posting tree

Posting list / Posting tree:

- Danh sách TID (row locations)
- Nếu nhỏ → lưu inline
- Nếu lớn → chuyển sang posting tree

```text
Item: "database" -> [CTID 1(block, offset), CTID 45(block, offset), CTID 900(block, offset)]
Item: "postgres" -> [CTID 1(block, offset), CTID 12(block, offset), CTID 55(block, offset)]
```

#### Posting list:

Posting list = list of TID(Tuple ID/CTID) đại diện cho những row chứa cùng một index key
Mỗi (block, offset) = vị trí row trong heap.

Posting list trong PostgreSQL được lưu như thế nào?
Khi số lượng TID ít
PostgreSQL lưu trực tiếp posting list: - Inline - Nằm ngay trong index entry

```text
Entry Tree node:
---------------------------------
Key: "postgres"
TIDs: [T1, T2, T3, T4]
---------------------------------
```

#### Posting tree:

Posting tree = một B-Tree riêng dùng để lưu rất nhiều TID cho cùng một key

```
Key → pointer → Posting Tree

Entry Tree
-----------
"postgres"
     |
     v
Posting Tree
-------------
[TID1, TID2, TID3]
       |
     [TID4, TID5]

```

### When to use GIN:

- JSONB: Search for keys or values.
- Arrays: Find rows containing specific elements.
- Full-Text Search: Searching keywords in long text.

| Data type  | Operator                |         |
| ---------- | ----------------------- | ------- |
| `array`    | `@>`, `<@`, `&&`        |         |
| `jsonb`    | `@>`, `?`, `?           | `, `?&` |
| `tsvector` | `@@` (full-text search) |         |

[!WARNING] Write Speed: GIN indexes are very slow to update. Because inserting one row might require updating 50 different postings lists (one for each word in the text). Postgres uses a "Pending List" to buffer these updates, which is later flushed to the main index.

### ASCII diagram of GIN structure:

```
GIN Index Overview
+-------------------+
| Metapage          |  <-- Root info, version, etc.
+-------------------+
| Entry Tree (B-Tree over Keys)
| +---------------+
| | Internal Node |  <-- Keys & Down Pointers
| +---------------+
| | Leaf Node     |
| | Key1 | Category | Posting Ptr --> Posting List or Tree
| | Key2 | ...     |
| +---------------+
+-------------------+
Posting List (Small, Inline):
[ TID1, TID2, TID3 ... ]  <-- Compressed, sorted tuple IDs

Posting Tree (Large Lists, Separate B-Tree):
+---------------+
| Root Node     |  <-- Points to leaves with TIDs
+---------------+
| Leaf: [TID chunk] |
+---------------+

Pending List (Temp, per key):
Unsorted new TIDs, flushed to main on threshold/VACUUM
```

---

## 2. GiST (Generalized Search Tree)

GiST is not an inverted index like GIN. Instead, it's a balanced search tree framework (similar in spirit to an R-tree for spatial data) where:

- Each internal node stores a predicate (often called a "bounding predicate" or "union") that conservatively approximates (covers) all the data in its subtree.
- The predicate is typically something that can overlap: e.g., a bounding box for geometry, a range union for ranges, or a merged text signature for full-text.
- Leaf nodes store the actual indexed values + TID (ctid pointer to heap tuple).
- Overlaps are allowed between sibling nodes — this is the key difference from B-Tree (strict non-overlapping ranges) or GIN (exact inverted lists).

### The Internal Mechanism & Page Structure

GiST uses standard PostgreSQL 8 KB slotted pages, but with important differences:

- Page types:
  - Root / Internal pages: Contain down-links (pointers to child pages) + union predicates (one per child).
  - Leaf pages: Contain actual indexed entries (compressed if compress method exists) + TID.

- Opaque data (in page header — see GISTPageOpaqueData in source):
  - NSN (Next Sequence Number) — for detecting concurrent page splits safely (very clever concurrency trick).
  - Flags (leaf/internal, deleted, etc.).
  - Right-link / parent-link for navigation & vacuum.

- Insertion & Splitting:

1. Traverse tree using consistent → find path where new value should go (choosing path with smallest penalty increase).
2. At leaf: if full → split using picksplit method (operator-class specific!).
3. picksplit decides how to divide entries into left/right → produces two new union predicates.
4. Insert new downlink + union into parent → may cascade split up to root.
5. Important asymmetry: Inserts can enlarge union predicates (to minimize penalty), but deletes never shrink them unless page split or REINDEX/VACUUM FULL occurs → index quality degrades over time on update-heavy workloads.

```text
          Root (union = big BOX covering everything)
               /          \
   Internal (BOX A overlaps BOX B a bit)    Internal (BOX C)
      /     \                                 |
Leaf1     Leaf2                            Leaf3
(points in A)  (points in B)               (points in C)
```

### When to use GiST:

- Geometric Data (PostGIS): Finding points within a radius.
- Range Types: Finding overlapping dates or prices.
- Trigram Search: ILIKE '%pattern%' using the pg_trgm extension.

---

## 3. GIN vs. GiST: The Staff-Level Trade-off

| Feature              | GIN (Generalized Inverted Index)              | GiST (Generalized Search Tree)            |
| -------------------- | --------------------------------------------- | ----------------------------------------- |
| Search Speed         | 🚀 Extremely fast for exact matches           | 🐢 Slower due to signature-based checks   |
| Build / Write Speed  | ❌ Very slow (expensive index maintenance)    | ✅ Faster writes and updates              |
| Index Precision      | Exact (non-lossy)                             | Often lossy                               |
| Recheck Required     | ❌ No                                         | ✅ Yes (heap recheck needed)              |
| EXPLAIN Indicator    | No recheck                                    | `Rows Removed by Index Recheck`           |
| Best Use Case        | Many items per row (JSONB, arrays, full-text) | Complex shapes, ranges, similarity search |
| Storage Size         | Large                                         | Smaller                                   |
| Operator Flexibility | Limited to supported operators                | Very flexible (custom operators)          |
| Typical Index Types  | `jsonb_ops`, `jsonb_path_ops`, `tsvector`     | Range, geometric, KNN, extensions         |

### Staff/Principal Insights: Recheck Conditions

GiST is often lossy. This means the index might say "This page might have your data." In EXPLAIN, you will see Rows Removed by Index Recheck. This means Postgres found a potential match in the index but had to verify the actual row in the Heap because the index signature wasn't precise enough.

---

## 4. Real SQL Example: JSONB Indexing

```sql
-- Create a table with JSONB
CREATE TABLE logs (data jsonb);

-- Index for @> (contains) operator
CREATE INDEX idx_logs_data ON logs USING GIN (data);

-- This query is now lightning fast
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM logs WHERE data @> '{"status": "error"}';
```

## 4. Real SQL Example: GiST index

```sql
CREATE TABLE places (
    id SERIAL PRIMARY KEY,
    name TEXT,
    loc GEOMETRY(POINT)
);

CREATE INDEX idx_places_loc ON places USING GIST (loc);

-- Insert data
INSERT INTO places (name, loc) VALUES
    ('Eiffel Tower', ST_Point(2.2945, 48.8584)),
    ('Louvre',       ST_Point(2.3376, 48.8606));

-- Overlap query (&& = overlaps)
EXPLAIN ANALYZE
SELECT * FROM places
WHERE loc && ST_MakeEnvelope(2.29, 48.85, 2.30, 48.86);
```

---

## 5. Practice Exercises 🏋️

1. The Fastupdate Test: Check the size of your GIN index. Perform a large batch of inserts and see how gin_pending_list_limit affects performance.
2. The Trigram Trick: Install pg_trgm and compare a LIKE query performance using a standard B-Tree vs a GiST index.

---

### 🧭 Thinking Question

In a high-traffic logging system where you insert 10,000 JSON rows per second, would you choose a GIN index on the whole JSON body? Why or why not? (Think about the write performance we discussed).

## Post your answer, and we'll move to Lesson 7: BRIN, Hash, and SP-GiST.

# Lesson 7: BRIN, Hash, and SP-GiST

In Lesson 6, we covered GIN and GiST for complex data. Today, we look at the last three major index types in PostgreSQL. One of these (BRIN) is a secret weapon for Big Data and Time-Series systems.

---

## 1. BRIN (Block Range Index)

BRIN is fundamentally different from B-Tree. Instead of pointing to individual rows, it points to a range of pages (e.g., 128 pages at a time).

BRIN (Block Range INdex) exploits exactly this: instead of indexing every row or every key, it divides the table into fixed-size ranges of consecutive heap pages (default: 128 pages ≈ 1 MB of data) and stores only summary statistics about each range — most commonly min and max values of the indexed column(s).

### The Internal Mechanism: Min/Max

For every 128 pages (1MB of data), BRIN stores only two values: the minimum and maximum value found in that range.
Process: When you query WHERE created_at = '2023-01-01', Postgres checks the BRIN index. It skips any 1MB block where your value is not between the Min and Max.

BRIN index pages (also 8 KB):

- Metapage (block 0): version, pages_per_range, lastRevmapPage, etc.
- Revmap pages (reverse range map): map block range IDs → physical index page numbers where summaries live. This allows BRIN to grow the summary pages without rewriting the whole index.
- Summary pages: each tuple is one block range summary. For minmax strategy (default):
  - Range start block number
  - Min value
  - Max value
  - (optional) nulls bitmap, etc.

```text
Block 1 (Pages 0-127): Min: 2023-01-01, Max: 2023-01-05 -> SEARCH HERE
Block 2 (Pages 128-255): Min: 2023-01-06, Max: 2023-01-10 -> SKIP
```

### Scan:

How a scan works:

1. Query planner sees range condition (e.g., timestamp > '2025-01-01')
2. BRIN scan looks at summaries
3. If query range doesn't overlap [min,max] of a block range → skip the entire 128 pages (huge win!)
4. If overlaps → add all pages in that range to a bitmap
5. Bitmap Heap Scan reads only those heap pages → filters rows with real condition (recheck)

### SQL Examples

```
CREATE TABLE measurements (
    measured_at TIMESTAMPTZ NOT NULL,
    sensor_id   INT NOT NULL,
    value       DOUBLE PRECISION
);

CREATE INDEX idx_measurements_time_brin ON measurements
USING BRIN (measured_at)
WITH (pages_per_range = 32);
```

### When to use BRIN:

- Append-only tables: Where data is naturally sorted (e.g., logs, events, metrics).
- Gigantic tables: BRIN indexes are thousands of times smaller than B-Trees. An index on a 1TB table might be only 10MB!
  [!CAUTION] Random Data: If your data is not sorted (e.g., a random UUID), BRIN is useless because every 1MB block will likely have a Min of 0 and a Max of Infinity.

```
BRIN Index
----------
Range 0–127   → min=10   max=99
Range 128–255 → min=100  max=199
Range 256–383 → min=200  max=299
```

### Common Mistakes & Pitfalls

- Using BRIN on low-correlation columns → min/max spans almost whole table → no skipping → worse than Seq Scan.
- Forgetting to reindex after bulk updates or heavy out-of-order inserts → summaries become inaccurate (use REINDEX INDEX CONCURRENTLY).
- Default pages_per_range=128 too coarse for your data → too many false positives. Tune down to 32–64 for better skipping (but index grows).
- Expecting point lookups (=) → BRIN is for range conditions only (>, <, BETWEEN).
- Not vacuuming → dead tuples still affect heap reads even if BRIN skips ranges.

Real-world postmortem: Time-series table grew to 5 TB. B-Tree on timestamp bloated to 800 GB and slowed inserts. Switched to BRIN → index shrunk to ~200 MB, inserts 5–10× faster, range queries 3–20× faster (depending on date spread).

---

## 2. Hash Indexes

Hash indexes are simple. They take your value, run it through a hash function, and point directly to the row.
From first principles: A hash table maps arbitrary keys to buckets via a hash function. Ideal average-case lookup is O(1): compute hash → go directly to bucket → check for match.

PostgreSQL's hash index applies this to disk-persistent storage:

- Compute a **32-bit hash** of the column value (using type-specific `hash` function, e.g., `hash_any()` for most).
- Use that hash to locate a **bucket**.
- Buckets contain lists of **TID** (ctid = heap tuple pointers) where the hash matches.
- **No actual column value** stored — only the 4-byte hash code → very compact for long values (UUIDs, long strings, URLs).
- **Lossy by design**: hash collisions mean you must recheck the real value on the heap (even if hashes match).

Result: Extremely fast **direct access** to buckets (no tree descent like B-Tree), potentially fewer logical I/O for huge tables.

But: Only supports `=` operator. No ranges, no ordering, no `<`, `>`, `LIKE`, etc. — planner ignores hash index for anything else.

### Internal Mechanism & Page Structure:

PostgreSQL uses **linear hashing** (dynamic, incremental growth) + chained buckets.

Key structures (from official docs, chapter on Hash Indexes):

- **Meta page** (page 0): Control info — version, bucket count, high/low mask for linear hashing, split pointer, etc.
- **Primary bucket pages**: Fixed initial buckets (start with 2, grow via splitting).
- **Overflow pages**: When a primary bucket overflows, chain additional pages.
- **Bitmap pages**: Track free/reusable overflow pages (after deletes/vacuum).

**Linear hashing growth** (avoids full rehash):

- Maintains a **split pointer** and **level** (number of bits used).
- Buckets before split pointer use `level+1` bits; after use `level` bits.
- When load factor > threshold (~75%), split **one bucket** at a time → insert new primary bucket, redistribute entries from old bucket.
- This keeps growth gradual, no big pauses.

**Insertion flow**:

1. Compute 32-bit hash of value.
2. Mask hash to current bucket count → find primary bucket.
3. If collision (same hash, different value), chain overflow page.
4. Append TID + 4-byte hash code to bucket/overflow chain.

**Lookup flow** (for `WHERE col = 'value'`):

1. Compute hash of 'value'.
2. Find bucket number.
3. Scan bucket chain, compare stored 4-byte hash (fast reject most collisions).
4. For matching hashes → fetch heap tuple → recheck real value (lossy scan).
5. Return matching rows.

**ASCII diagram** (simplified):

```
Meta Page
+---------------------------------+
| bucket_cnt | split_ptr | level  |
+---------------------------------+

Bucket Directory (logical, via masks)
Bucket 0 → Primary Page 5
Bucket 1 → Primary Page 7 + Overflow Page 12
Bucket 2 → Primary Page 3
...

Primary Bucket Page (example)
+---------------------------------+
| hash_code1 | TID (ctid)         |
| hash_code2 | TID                |
| ...                            |
+---------------------------------+
          ↓ (overflow chain)
Overflow Page
+---------------------------------+
| more hash_code | TID            |
+---------------------------------+
```

- Buckets chain via page links.
- Vacuum cleans dead TIDs, reuses overflow pages via bitmap.

### SQL Examples

```sql
CREATE TABLE sessions (
    session_id   UUID PRIMARY KEY,
    user_id      BIGINT,
    data         JSONB
);

-- Hash index on UUID (long value → hash index smaller than B-Tree)
CREATE INDEX idx_sessions_hash ON sessions USING HASH (session_id);

-- Typical lookup
EXPLAIN ANALYZE
SELECT * FROM sessions WHERE session_id = '550e8400-e29b-41d4-a716-446655440000';
```

### The Trade-off:

- Pros: For a simple = equality check, Hash indexes can be slightly faster than B-Trees.
- Cons:
  - They cannot do range scans (>, <).
  - They cannot do sorted results (ORDER BY).
  - They were only made "crash-safe" in Postgres 10, so many legacy systems avoid them.

---

## 3. SP-GiST (Space-Partitioned GiST)

GiST allows **overlapping** bounding predicates in sibling nodes — this flexibility handles arbitrary data (e.g., heavily overlapping polygons) but leads to:

- Union predicates growing over time (never shrinking on delete).
- More false positives → more rechecks → slower queries as the index ages.

SP-GiST flips this: it enforces **non-overlapping, disjoint partitions** of the search space at every level.

- The "space" can be geometric (2D plane → quadtree), multi-dimensional (k-d tree), prefix-based (radix tree for strings/inet), or any domain that can be recursively divided without overlap.
- Each internal node defines a **prefix** or **rule** that splits its children into mutually exclusive regions.
- No overlaps → **no need for conservative unions** → fewer false positives → fewer rechecks → better pruning.

### The Internal Mechanism: Prefixes

SP-GiST uses 8 KB pages like other indexes, but with a very different layout optimized for **long pointer chains mapped to few disk pages** (high fanout on disk).

Key differences from GiST:

- **No union predicates** — instead, each node has a **prefix value** (or label) that defines how the space is partitioned.
- Nodes can be:
  - **Inner nodes** — contain **label** (prefix) + pointers to child nodes.
  - **Leaf nodes** — contain actual indexed values + TID (ctid).
- **Nulls & placeholders** — special handling for sparse trees (e.g., many empty partitions).
- **Concurrency** — uses a similar NSN (Next Sequence Number) scheme as GiST for safe splits without full locks.
- **Node-to-page mapping** — SP-GiST packs many logical tree nodes into one disk page (high fanout) to minimize I/O even for deep trees (e.g., radix trees with depth 32+ for IPv6).

Common implementations (built-in opclasses):

- **quadtree** — for points (geometry/geography) — recursively divides plane into 4 quadrants.
- **kd-tree** — multi-dimensional points.
- **radix tree (prefix tree)** — for text, inet/cidr, any prefixable type.
- **text** — for LIKE 'prefix%' queries (very efficient).

**How a scan works** (e.g., point lookup or nearest-neighbor):

1. Start at root.
2. At each inner node: use **choose** function to pick the correct child partition (based on query value).
3. Descend until leaf → exact match or collect candidates.
4. Because partitions are **disjoint** → at most one path per lookup (no branching like GiST) → fewer pages touched.
5. For range/nearest: may visit multiple subtrees, but still no overlap waste.

ASCII approximation of SP-GiST quadtree (point data):

```
Root (whole plane)
   ├── NW quadrant (label: NW) ──> child node
   ├── NE quadrant              ──> ...
   ├── SW
   └── SE

Each quadrant node:
   ├── sub-NW ──> deeper partition
   └── ...     (no overlap with siblings)
```

For radix tree (e.g., IP prefixes):

```
Root
   ├── '192' prefix ──> node
   │     ├── '.168' ──> ...
   └── '10' prefix ──> ...
```

#### 3. SQL Examples

Built-in point indexing with quadtree SP-GiST:

```sql
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name TEXT,
    coord POINT NOT NULL
);

CREATE INDEX idx_locations_coord_spgist ON locations
USING SPGIST (coord point_ops);

-- Insert points (well-distributed)
INSERT INTO locations (name, coord) VALUES
    ('Eiffel', '(2.2945,48.8584)'),
    ('Statue Liberty', '(-74.0445,40.6892)');

-- Nearest neighbor or containment
EXPLAIN ANALYZE
SELECT * FROM locations
ORDER BY coord <-> '(2.3,48.86)'::point
LIMIT 5;
```

Typical plan:

```
Limit  (actual time=0.015..0.016 rows=1 loops=1)
  ->  Index Scan using idx_locations_coord_spgist on locations  (cost=... rows=... actual ...)
        Order By: (coord <-> '(2.3,48.86)'::point)
```

For prefix search (radix):

```sql
CREATE INDEX idx_inet_spgist ON access_logs USING SPGIST (ip_address inet_ops);
```

### When to use SP-GiST:

- URLs and File Paths: Where many entries share the same prefix.
- IP Addresses: CIDR ranges.
- Clustered Points: Where data is densely packed in some areas and sparse in others.

---

## 4. Staff/Principal-Level Insights

### BRIN "Lossiness"

BRIN is always lossy. It tells you "Your data might be in this 1MB block." Postgres must then perform a Sequential Scan of those 128 pages. This is why BRIN is usually combined with a Bitmap Heap Scan.

### The Storage/Performance Balance

As a Staff Engineer, you don't use indexes just to make things fast; you use them to manage resources. If you have a 10TB logging table, a B-Tree on timestamp might take 500GB of disk and RAM. A BRIN index takes 50MB. Even if BRIN is 2x slower than B-Tree, it saves 499.95GB of expensive NVMe storage. That is a win.

---

## 5. Practice Exercises 🏋️

1. Size Comparison: Create a table with 10 million rows sorted by a date. Create a B-Tree and a BRIN index on the date. Use \di+ in psql to compare their sizes.
2. The Fragmentation Test: Create a BRIN index. Then UPDATE random rows throughout the table (messing up the sort order). Run a query and see if the performance degrades. (Hint: Use VACUUM to fix it).

---

### 🧭 Thinking Question

You have a logs table that is 5TB. You only ever query it by request_id (a random UUID). Can you use a BRIN index for this? Why or why not?

Post your answer, and we'll move to Lesson 8: Composite Indexes & Column Order Rules.

---

# Lesson 8: Multi-column, Partial, and Expression Indexes

## Until now, we've mostly indexed single columns. But real-world queries are complex. A Staff Engineer knows how to tailor an index so perfectly to a query that the database barely has to work.

## 1. Multi-column (Composite) Indexes

A composite index is an index on (col1, col2, col3).

A composite index on (last_name, first_name) is like a phone book sorted first by last_name, then within each last_name group, by first_name.
PostgreSQL stores the index entries in **lexicographical order** — the leftmost column is the most significant "digit."
Key principle: **The leftmost prefix rule** (still mostly true in 2026, with one big exception we'll cover).
The planner can use the index efficiently only for **prefixes** of the indexed columns.

### The "Left-to-Right" Rule (Crucial)

Imagine a phone book sorted by (Last Name, First Name).

- Can you find "Smith, John"? Yes.
- Can you find all "Smiths"? Yes.
- Can you find all "Johns" without knowing the last name? No. (You'd have to scan the whole book).
  Postgres Rule: A composite index can be used if the query filters on a prefix of the indexed columns.

- Index on `(a, b, c)` can help with:
  - `WHERE a = ?`
  - `WHERE a = ? AND b = ?`
  - `WHERE a = ? AND b = ? AND c = ?`
  - `WHERE a = ? AND b > ?` (equality on prefix + range on next)
  - `ORDER BY a, b, c` (index-only sort)
- But **not** efficiently for:
  - `WHERE b = ?` (skips leading column → usually Seq Scan or Bitmap Index Scan fallback)
  - `WHERE c = ?` (even worse)

### Internal Mechanism

In a B-tree composite index:

- Each index tuple contains **all** indexed columns + TID (ctid).
- Keys are compared component-wise: first compare column 1, if equal → column 2, etc.
- Leaf pages store sorted tuples → efficient range scans within a prefix.

#### 3. SQL Examples

```sql
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT      NOT NULL,
    order_date  TIMESTAMPTZ NOT NULL,
    status      VARCHAR(20) NOT NULL,  -- 'pending', 'shipped', 'delivered'
    amount      NUMERIC
);

-- Populate with realistic data (customer_id high cardinality, status low)
```

Common composite indexes:

```sql
-- Good: equality + range
CREATE INDEX idx_orders_cust_date ON orders (customer_id, order_date);

-- Good: equality on low-card + range (status first if status often filtered alone)
CREATE INDEX idx_orders_status_date ON orders (status, order_date);

-- Allows: WHERE customer_id = 123 AND order_date > '2026-01-01'
EXPLAIN ANALYZE SELECT * FROM orders
WHERE customer_id = 123 AND order_date >= '2026-01-01';
-- → Index Scan using idx_orders_cust_date

-- Same index supports: WHERE customer_id = 123 ORDER BY order_date
-- (index provides sort order)

-- Bad order example
CREATE INDEX idx_bad_date_cust ON orders (order_date, customer_id);
-- WHERE customer_id = 123 → cannot use efficiently (no prefix)
```

### B-tree composite index structure

Root Page:

```
+----------------------------------+
| key = (20, 'PAID', 2024-01-01)   | → child page A
| key = (50, 'PENDING', 2024-02-01)| → child page B
+----------------------------------+
```

Internal Page:

```
+----------------------------------+
| (10, 'PAID', 2023-12-01) → P1    |
| (10, 'PENDING', 2024-01-01) → P2 |
| (15, 'PAID', 2024-01-05) → P3    |
+----------------------------------+
```

Leaf Page:

```
+---------------------------------------------------+
| (10, 'PAID', 2024-01-01) → (block 120, off 3)     |
| (10, 'PAID', 2024-01-02) → (block 121, off 7)     |
| (10, 'PENDING', 2024-01-03) → (block 130, off 1)  |
| (11, 'PAID', 2024-01-01) → (block 200, off 4)     |
+---------------------------------------------------+
```

Index tuple layout (physical):

```
+----------------------------------+
| IndexTupleHeader                 |
| - size                           |
| - null bitmap                    |
+----------------------------------+
| user_id                          |
| status                           |
| created_at                       |
| heap TID (block, offset)         |
+----------------------------------+
```

---

## 2. Partial Indexes

Why index the whole table if you only care about a tiny fraction of it? CREATE INDEX idx_active_users ON users(id) WHERE status = 'active';

### Concept Explanation (First Principles)

A normal index duplicates **every row** in the table (or the indexed columns).  
But in real workloads:

- Many tables have **skewed distributions** — e.g., 85% of orders are "completed", only 5% are "pending".
- Queries almost always filter on the **interesting minority** (e.g., pending orders, active users, non-deleted records).
- Indexing the majority adds bloat: larger index → slower scans, more I/O, higher memory usage, slower INSERT/UPDATE/DELETE (because index must be maintained for every row).

**Partial index solution**: Add a `WHERE` clause to the `CREATE INDEX` statement. PostgreSQL builds and maintains entries **only** for rows matching that predicate.

- Index is smaller (often 5–20% the size of a full index).
- Updates/inserts/deletes skip index maintenance when row doesn't match predicate → faster writes.
- Queries matching the predicate can use a tiny, dense index → faster lookups.
- Queries **not** matching the predicate fall back to Seq Scan or other indexes — planner is smart about this.

Key restriction: The query's `WHERE` clause must be **implied by** (or at least compatible with) the index's predicate for the planner to consider it. PostgreSQL won't use a partial index if it can't prove the condition is satisfied.

### Internal Mechanism

Internally, a partial index is a normal B-tree (or GIN/GiST/BRIN) with an extra **predicate check**:

- During INSERT/UPDATE: Before adding/updating index entry, evaluate the partial predicate. If false → skip index operation.
- During VACUUM: Dead tuples in partial indexes are cleaned normally.
- Planner: When seeing a query, it checks if the query qualifiers **imply** the index predicate (via constraint exclusion logic). If yes → partial index is a candidate (often cheaper due to smaller size).
- Stats: Partial indexes have their own statistics in `pg_statistic` — `ANALYZE` treats them separately.

No extra page types or structures — just fewer tuples in the B-tree leaves.

### SQL Examples

Classic use cases:

**1. Index only "active" or "pending" records**

```sql
CREATE INDEX idx_orders_pending ON orders (customer_id, order_date)
WHERE status = 'pending';
```

Query:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'pending' AND customer_id = 123 AND order_date > now() - interval '7 days';
```

→ Uses the tiny partial index (maybe 2–5% of full index size).

**2. Unique constraint on subset** (very powerful!)

```sql
-- Allow multiple soft-deleted users with same email, but enforce unique active emails
CREATE UNIQUE INDEX idx_users_active_email_unique
ON users (email)
WHERE is_deleted = false AND status = 'active';
```

This enforces business rules without blocking deleted/inactive records.

**3. Index only non-null or recent data**

```sql
CREATE INDEX idx_logs_recent_errors ON logs (error_code, logged_at)
WHERE logged_at > CURRENT_DATE - INTERVAL '30 days'
  AND level = 'ERROR';
```

Great for monitoring dashboards — index stays small as old logs age out (but see caveats below).

**4. Partial on expression**

```sql
CREATE INDEX idx_users_phone_normalized
ON users (lower(phone))
WHERE phone IS NOT NULL;
```

### When to use:

- Soft Deletes: WHERE deleted_at IS NULL.
- Queues: WHERE processed = false.
- Top Tier: WHERE total_spend > 10000.
  [!TIP] Performance & Size: A partial index is smaller (less RAM) and faster to update (Postgres skips the index update if the new row doesn't meet the WHERE clause).

---

## 3. Expression (Functional) Indexes

By default, an index on email won't help if you query WHERE LOWER(email) = 'user@example.com'.

Solution: Index the expression itself. CREATE INDEX idx_user_lower_email ON users (LOWER(email));

### Concept Explanation (First Principles)

PostgreSQL indexes normally store **raw column values**.  
But many queries filter or sort on **transformed** or **derived** values:

- Case-insensitive search → `lower(email)`
- Date truncation → `date_trunc('day', created_at)`
- Computed fields → `quantity * price`
- JSONB extraction → `(data ->> 'email')`
- Prefix normalization → `left(phone, 3)`

Without an expression index, PostgreSQL must:

1. Read every qualifying row from the heap
2. Compute the expression for each row
3. Apply the filter

This kills performance on large tables — especially if the function is expensive or non-sargable.

**Expression index solution**: Store the **pre-computed result** of the expression in the index itself.  
During index maintenance (INSERT/UPDATE), PostgreSQL computes the expression once and stores it.  
During query execution, the planner sees `WHERE indexed_expression = constant` → uses the index like any normal index. No recomputation needed at query time.

**Key requirement**: The function(s) must be marked **IMMUTABLE** (always return the same output for the same input, no side effects, no dependency on current time/database state).  
PostgreSQL enforces this — if you try to index a STABLE or VOLATILE function, creation fails.

### Internal Mechanism

- Syntax: `CREATE INDEX ... ON table ( (expression) )`
- The parentheses around the expression are **required** when it's not a plain column name.
- The index stores the computed value + TID (like any B-tree/GIN/etc.).
- Maintenance cost: Expression recomputed on every INSERT and **non-HOT UPDATE** (HOT skips if indexed columns unchanged).
- Query time: Expression **not** recomputed → planner treats it as a simple equality/range on a stored column.
- Statistics: PostgreSQL collects stats on the **expression values** (via `ANALYZE`), so selectivity estimates are accurate.
- Can be combined with: partial indexes, multi-column, INCLUDE, different index types (B-tree most common, but GIN on expressions possible for arrays/JSON).

### SQL Examples

Classic real-world patterns:

**1. Case-insensitive email/username search**

```sql
CREATE INDEX idx_users_lower_email ON users ( (lower(email)) );

-- Query uses the index automatically
EXPLAIN ANALYZE
SELECT * FROM users WHERE lower(email) = 'john.doe@example.com';
```

Plan:

```
Index Scan using idx_users_lower_email on users  (actual time=0.015..0.020 rows=1 loops=1)
  Index Cond: (lower(email) = 'john.doe@example.com'::text)
```

**2. Date truncation (common in reporting)**

```sql
CREATE INDEX idx_orders_day ON orders ( (date_trunc('day', order_date)) );

SELECT COUNT(*) FROM orders
WHERE date_trunc('day', order_date) = '2026-01-15';
```

→ Fast daily aggregates without full scan.

**3. Computed value (total price)**

```sql
CREATE INDEX idx_line_items_total ON line_items ( (quantity * unit_price) );

-- Range scan possible
SELECT * FROM line_items
WHERE quantity * unit_price > 1000;
```

**4. JSONB key extraction (very common in 2026 apps)**

```sql
CREATE INDEX idx_users_profile_email ON users ( ((profile ->> 'email')) )
WHERE profile ? 'email';   -- optional partial for safety

-- Or with expression + operator class for GIN if needed
CREATE INDEX idx_users_data_gin ON users USING GIN ( (profile -> 'tags') );
```

**5. Partial + expression combo (powerful!)**

```sql
CREATE INDEX idx_active_lower_username
ON users ( (lower(username)) )
WHERE is_active = true AND deleted_at IS NULL;
```

### The Price:

Postgres must calculate the function result for every INSERT and UPDATE. Use these for expensive but common transformations.

---

## 4. Staff/Principal-Level Insights

### Index Intersection vs. Composite

A common question: "Should I have two indexes on (A) and (B), or one composite index on (A, B)?"

Two Indexes: Postgres can use a Bitmap Index Scan to combine them (Lesson 4). It's flexible.
One Composite: It is much faster for queries that use both, and it allows for Index Only Scans (Lesson 4).
Staff Decision: If (A, B) is a high-frequency performance-critical path, use Composite. If they are used independently and rarely together, two single indexes are better.

### Redundant Indexes

As a Principal Engineer reviewing a PR, look for:

- Index 1: (last_name, first_name)
- Index 2: (last_name)
  Index 2 is redundant! Index 1 already covers the prefix last_name. Delete Index 2 to save disk space and write overhead.

---

## 5. Practice Exercises 🏋️

1. The Prefix Test: Create an index on (tenant_id, user_id). Run a query filtering only by user_id. Does it use the index? (Check for Index Scan vs Seq Scan).
2. The Partial Savings: Create a table where 99% of rows are inactive. Create a partial index for active rows. Compare the size of this index vs a full index on the same column.

---

### 🧭 Thinking Question

You have an index on (created_at, status). Your query is: SELECT \* FROM orders WHERE created_at > '2024-01-01' AND status = 'shipped'. Will the status part of the index be used effectively? (Think about how B-Trees are sorted).

## Post your answer, and we'll move to Lesson 9: Advanced - PostgreSQL Internal Structure (Pages, Tuples, Pointers).

# Lesson 9: PostgreSQL Internal Structure (Pages, Tuples, Pointers)

Welcome to the Staff Level. To truly master indexing, you must stop looking at tables as "rows" and start seeing them as byte arrays on disk.

## In Lesson 2, I mentioned the Ctid (Item Pointer). Today, we look at exactly where that pointer goes.

## 1. The 8KB Page Layout

Every file in a Postgres database is a sequence of 8KB pages. A page is laid out like a "sandwich":

```text
+----------------+
| Page Header    | (24 bytes: checksum, pointers to free space)
+----------------+
| Item Pointers  | (4 bytes each: [Offset, Length])
| [1] [2] [3]    | (Grows downwards)
+----------------+
| Free Space     | (Where new data goes)
+----------------+
| Tuples (Data)  | (Grows upwards)
|     [3] [2] [1]|
+----------------+
| Special Space  | (Used by indexes like B-Tree for sibling pointers)
+----------------+
```

### Why this design?

## Efficiency: The Item Pointers at the top act as a map. They never move. Even if a Tuple is moved within the page to defragment space, its index ID (1, 2, 3) stays the same.

## 2. The Item Pointer (Tid / Ctid)

When an index finds a match, it returns a Ctid like (42, 3).

- 42: Go to the 42nd page of the file.
- 3: Look at the 3rd Item Pointer in that page.
- That pointer then tells the CPU exactly what byte offset to start reading the data.

---

## 3. Data Alignment & Padding (Staff-Level Performance)

Postgres aligns data on 8-byte boundaries. This means the order of your columns in a table affects how much disk space you use.

Mỗi kiểu dữ liệu có alignment requirement
Example:
| Type | Size | Alignment |
| ------ | ------- | ------------------ |
| `int2` | 2 bytes | 2-byte aligned |
| `int4` | 4 bytes | 4-byte aligned |
| `int8` | 8 bytes | **8-byte aligned** |

Rule:
Giá trị phải bắt đầu ở offset là bội số của alignment

```sql
-- Bad Design (Wasteful)
CREATE TABLE wasteful (
    id int2,      -- 2 bytes
    val int8,    -- 8 bytes (Needs 6 bytes of padding before it)
    status int2   -- 2 bytes
);
=> offset 0-1: int2 id
=> offset 2-9: padding (val int8 cần 8-byte alignment Offset hiện tại = 2 và Bội số gần nhất của (8,2) là 8 => **PostgreSQL phải chèn padding để đảm bảo rằng giá trị val bắt đầu ở offset là bội số của alignment**)
=> offset 10-17: int8 val
=> offset 18-19: int2 status
-- Each row wastes ~6 bytes. At 1 Billion rows, this is 6GB of wasted RAM/Disk.

-- Good Design (Compact)
CREATE TABLE compact (
    val int8,    -- 8 bytes
    id int2,      -- 2 bytes
    status int2   -- 2 bytes
);
```

## [!IMPORTANT] Index Impact: This alignment also applies to your B-Tree leaf pages! A poorly ordered composite index can be significantly "fatter" than a well-ordered one.

4. Seeing the Bytes: pageinspect
   You don't have to take my word for it. Postgres provides an extension to look at the raw binary data.

```sql
CREATE EXTENSION pageinspect;

-- Look at the header of the first page of our 'large_table'
SELECT * FROM page_header(get_raw_page('large_table', 0));

-- Look at the individual items (tuples) in that page
SELECT * FROM heap_page_items(get_raw_page('large_table', 0));
```

---

## 5. Staff/Principal-Level Insights

### The Tuple Header Overhead

Every single row in Postgres has a 23-byte header.

- If you have a table with only one boolean column (1 byte), the physical size of the row is at least 24 bytes (Header + Data).
- This is why Thin Tables (many rows, few columns) often have much higher index overhead relative to data than Fat Tables.

---

## 6. Practice Exercises 🏋️

1. The Padding Test: Create the wasteful and compact tables from the example above. Insert 1 million rows into each. Use pg_total_relation_size() to see the difference.
2. Page Inspector: Use page_header on one of your indexes. Notice how it differs from a Heap page (e.g., look for special space).

---

### 🧭 Thinking Question

In the Page Layout diagram, the Item Pointers grow down and the Tuples grow up. What happens when they meet in the middle? (Don't overthink it, but think about what Postgres has to do internally to prevent corruption).

## Post your answer, and we'll move to Lesson 10: Advanced - MVCC, HOT Updates, and Index Bloat.

# Lesson 10: MVCC, HOT Updates, and Index Bloat

To answer the Lesson 9 question: When the pointers and tuples meet in the middle, the Free Space is gone. Postgres cannot fit more data into that 8KB page and must find (or create) a new one.

## Today, we cover the "Hidden Tax" of PostgreSQL: MVCC. This is where many Senior Engineers get stuck, and where Staff Engineers save the company millions in infrastructure costs.

## 1. MVCC & The "Shadow" Row

PostgreSQL uses Multi-Version Concurrency Control (MVCC). When you UPDATE a row, Postgres doesn't change the data in place (like MySQL/InnoDB sometimes does). It marks the old row as dead and inserts a brand-new row (tuple) at a new Ctid.

### The Index Problem:

## If you have 5 indexes on a table and you update one column (even an unindexed one), all 5 indexes must be updated to point to the new Ctid. This results in massive write amplification.

### Concept Explanation: MVCC Basic

PostgreSQL never overwrites or deletes rows in place.  
Instead:

- **INSERT** → new tuple with `t_xmin = current txid`, `t_xmax = 0`.
- **UPDATE** → new tuple (new `t_xmin = current txid`) + old tuple gets `t_xmax = current txid` (marked dead for future txs).
- **DELETE** → old tuple gets `t_xmax = current txid`.

Readers (SELECT) see only versions visible to their snapshot (based on txids).  
Writers never block readers → high concurrency, but creates **dead tuples** (old versions) → **bloat**.

Visibility check (during scan):

- Tuple visible if: `t_xmin` committed & before snapshot AND (`t_xmax = 0` OR `t_xmax` aborted OR after snapshot).

This is why Seq Scan must visit every tuple → check visibility.  
Indexes point to ctids → must follow to heap for visibility.

## 2. HOT (Heap-Only Tuples) Optimization

Postgres engineers realized this was a nightmare, so they invented HOT.
HOT Update xảy ra khi UPDATE không thay đổi bất kỳ cột nào nằm trong index

### The Requirements for HOT:

1. The updated column must not be part of any index.
2. There must be enough Free Space on the same 8KB page for the new tuple.

### Internal Mechanism: HOT Updates (Heap-Only Tuple)

HOT is PostgreSQL's most important MVCC optimization (introduced in 8.3).

**Normal UPDATE flow (non-HOT)**:

1. Create new tuple on (possibly) different page.
2. Set old tuple's `t_xmax` + `t_ctid` points to new tuple.
3. Update **every index** to point to new ctid → expensive if many indexes.

**HOT UPDATE conditions** (all must be true):

- New tuple fits on the **same heap page** as old one (enough free space → fillfactor helps!).
- Update does **not change any indexed column**.
- No TOAST changes that move data out-of-line.

**HOT flow**:

1. Create new tuple on same page.
2. Old tuple: `t_xmax` set, `t_ctid` points to new tuple → creates **HOT chain**.
3. **Indexes unchanged** — still point to old ctid.
4. During index scan: follow chain via `t_ctid` → reach newest visible version.
5. During VACUUM: prune chain → redirect line pointer or remove dead entries.

**Result**: No index writes → 5–10× faster updates, far less index bloat.

**HOT chain pruning**:

- Opportunistic during index scans (if chain visible).
- Systematic during VACUUM → removes dead chain links, marks redirect pointers.

### How it works:

- Instead of updating the index, Postgres creates a HOT Chain inside the heap page.
- Item Pointer 1 points to Tuple A (Old). Tuple A has a hidden flag saying "Go to Tuple B (New)".
- The Index still points to Item Pointer 1. It never changes.
  [!TIP] This is why Staff Engineers often recommend keeping your most frequently updated columns out of indexes. If you update an indexed column, you kill the HOT optimization for that row.

### How HOT Interacts with Visibility Map & Index-Only Scans

Visibility Map (VM) — per-page bitmap (1 bit per heap page):

- **all-visible** bit: all tuples on page visible to everyone (no recent changes).
- **all-frozen** bit: no wraparound risk (txids frozen).

Index-Only Scan (IOS):

1. Find index entry → get ctid.
2. Check VM for page → if all-visible → return data from index (no heap fetch!).
3. If not → heap fetch → check visibility → possibly follow HOT chain.

HOT + VM synergy:

- Frequent HOT updates keep changes on same page → easier for VACUUM to mark page all-visible.
- Pruned HOT chains → fewer dead tuples → VM stays set longer.
- Result: more true Index-Only Scans → huge I/O savings.

If indexed columns updated often → no HOT → dead tuples spread → VM rarely set → IOS falls back to heap fetches → performance hit.

### Benefits of HOT Updates:

HOT là tối ưu sống còn để:

- Giảm index bloat
- Giảm write amplification
- Tăng update performance

---

## 3. Index Bloat: The Ghost in the Machine

**Table bloat** — dead tuples + free space gaps (mitigated by VACUUM + fillfactor).

**Index bloat** — much harder:

- Each UPDATE (non-HOT) → new index entry + old entry becomes dead.
- VACUUM prunes dead index entries but **does not shrink pages** or return space (B-tree keeps structure).
- Result: fragmented pages, half-empty, taller trees → slower lookups, more I/O, worse caching.
- Common triggers:
  - Frequent UPDATE/DELETE on indexed columns (no HOT).
  - High-cardinality indexes (many unique entries).
  - Bulk loads + deletes (e.g., time-series without partitioning).

**Typical bloat levels**:

- 30–70% "normal" in active B-trees.
- > 100–200% → rebuild needed.

---

4. Detection and Mitigation
   Detecting Bloat
   You can't see bloat with \di+. You need specialized queries or the pgstattuple extension.

```sql
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstatindex('idx_large_table_id');
-- Look at 'avg_leaf_density'.
-- If it's 20% or 30%, your index is 70-80% empty air (Bloat).
```

Fixing Bloat: REINDEX CONCURRENTLY
In the past, you had to lock the table to fix bloat. Now we use: REINDEX INDEX CONCURRENTLY idx_my_heavy_index; This builds a fresh index in the background without blocking reads or writes.

---

## 5. Staff/Principal-Level Insights

### The "Fillfactor" Strategy

In Lesson 2, we mentioned fillfactor. Now you know why it matters. If you set a table's fillfactor to 90, you leave 10% free space in every page. This gives HOT updates a place to land, significantly reducing index maintenance overhead.

### Visibility Map and Indexing

## Remember Lesson 4? Index-only scans depend on the Visibility Map. If you have high bloat and infrequent VACUUM, your Index-Only scans will revert to Heap fetches, destroying your performance.

## 6. Practice Exercises 🏋️

1. Observe HOT: Create a table, insert a row, and check pg_stat_user_tables. Update an unindexed column 100 times. Does n_tup_hot_upd increase? Now add an index to that column and repeat. What happens?
2. Bloat Generation: Perform a massive DELETE (e.g., 50% of the table). Check pgstatindex. Does the index shrink? (Spoiler: No).

---

### 🧭 Thinking Question

If you have a table where you constantly update a last_login_at column, and you decide to index it to find "recently active users," what is the architectural trade-off you are making?

## Post your answer, and we'll move to Lesson 11: Expert - Query Planner Internals & Cost estimation.

# Lesson 11: Query Planner Internals & Cost Estimation

In Lesson 10, we saw the physical cost of updates. But how does Postgres know which path costs more? Why did it choose a Seq Scan with cost 18334.00 instead of an Index Scan?

## Today, we look at the Math of the Optimizer.

## 1. The Query Lifecycle

Before a query runs, it goes through five stages:

1. Parser: Checks syntax.
2. Analyzer: Checks if tables/columns exist.
3. Rewriter: Handles Views and Rules.
4. Planner (The Brain): Generates multiple "Paths" and calculates the cost of each.
5. Executor: Actually runs the cheapest path.

---

## 2. The Cost Model (The Math)

Postgres doesn't use "seconds" to measure cost. It uses an arbitrary unit where 1.0 is the cost of reading a single 8KB page sequentially from disk.

### The Basic Formula:

Total Cost = (I/O Cost) + (CPU Cost)

- I/O Cost: number of pages × seq_page_cost (1.0).
- CPU Cost: number of tuples × cpu*tuple_cost (0.01).
  Example: A 1,000-page table with 10,000 rows. Cost = (1000 * 1.0) + (10000 \_ 0.01) = 1,100.0

---

## 3. The "Random I/O" Penalty

This is the most important setting for a Staff Engineer.

- seq_page_cost: 1.0 (Default)
- random_page_cost: 4.0 (Default)

Postgres assumes that reading from an index (Random I/O) is 4x more expensive than a Sequential Scan. This was true for spinning hard drives (HDDs).

## [!IMPORTANT] Modern Flash/SSD Tuning: On modern NVMe SSDs, the random I/O penalty is almost zero. Most Staff Engineers set random_page_cost = 1.1 or 1.1. This encourages the planner to use indexes more aggressively for larger result sets.

## 4. Path Selection & The GEQO

For a simple SELECT, there are only a few paths. But for a 10-table join, there are millions of possible combinations.

Dynamic Programming: For small joins (< 12 tables), Postgres checks every possibility.
GEQO (Genetic Query Optimizer): For complex joins, Postgres uses a genetic algorithm to "evolve" a good-enough plan because checking every path would take longer than the query itself.

---

## 5. Staff/Principal-Level Insights

### Plan Stability & "Lying" Statistics

As a Principal Engineer, you will encounter the "Flip-Flopping Plan". This happens when the cost of an Index Scan and a Seq Scan are almost identical (e.g., 1000.41 vs 1000.42). A tiny change in statistics might cause the database to switch plans, causing a sudden performance spike.

### Tuning the "Brain"

Beyond just indexes, you can tune how the planner thinks:

- effective_cache_size: Tells the planner how much memory is available for caching. If you set this higher, the planner will more likely assume index pages are in RAM and choose index scans.

---

## 6. Practice Exercises 🏋️

1. The Cost Manipulation: Run EXPLAIN on a query that uses an index. Then run SET random_page_cost = 100; and run the query again. Does it switch to a Seq Scan?
2. The Cache Effect: Compare the plan output of a query with SET effective_cache_size = '1GB'; vs SET effective_cache_size = '64GB';. (You may need a large table/complex query to see the transition).

---

### 🧭 Thinking Question

## If you set random_page_cost = 0.1 (lower than seq_page_cost), what kind of "bad" behavior might the database start showing? (Think about why Sequential Scans exist even for small tables).

# Lesson 12: High-scale Indexing Strategies (Multi-tenant & Time-series)

## We have reached the peak of the mountain. At the Principal level, you aren't just choosing between B-Tree and GIN; you are designing Architectural Strategies for systems that store petabytes of data and handle millions of tenants.

## 1. Multi-tenant SaaS Indexing

In a SaaS application (like Slack or Notion), every query usually starts with WHERE tenant_id = ?.

### Strategy A: The Sharded (Partitioned) Index

If you use Declarative Partitioning by tenant_id, each tenant's data lives in its own table.

- Pros: Each index is tiny and fits in RAM. Deleting a tenant is a DROP TABLE (instant).
- Cons: Management overhead increases as you reach thousands of partitions.

### Strategy B: The Big Table (Composite Index)

If everyone is in one big table, you must use a composite index: (tenant_id, user_id, ...).

- The "Data Locality" Benefit: Because B-Trees are sorted, all data for tenant_id=123 is physically located right next to each other in the index. This results in incredibly fast cache hits for that tenant.

---

## 2. Time-series Data: The "BRIN + Partitioning" Combo

Time-series data (logs, IoT metrics) is usually:

1. Append-only (nearly sorted by time).
2. Queried by time ranges.
3. Massive.

### The Staff-level Pattern:

- Partition by Month/Day: To keep the working set small.
- BRIN on created_at: Since data is sorted, BRIN achieves 99% of B-Tree performance at 0.1% of the disk cost (Lesson 7).
- Partition Pruning: Postgres can skip entire partitions (tables) if the WHERE clause doesn't match the partition range.

---

## 3. Read vs. Write Trade-offs (The Infrastructure Budget)

As a Principal Engineer, you must justify every index.

- Index Count Limit: In a high-write OLTP system, having more than 5-7 indexes on a single table usually starts causing visible lock contention and disk I/O pressure during peak traffic.
- The RAM Trap: If your indexes grow larger than your server's RAM (Working Set), your database will "fall off a cliff" as it starts swapping index pages from disk instead of RAM.

---

4. Real-world Post-mortem: The "Shared Buffer" Crisis
   Scenario: A company added a GIN index to a 1TB JSONB table.

- The Failure: Every INSERT now had to update the GIN index. GIN updates are random I/O.
- The Result: The random I/O from the GIN index updates evicted the "normal" data from the cache. Slowing down every query in the system, even those that didn't use the JSON table.
- Lesson learned: Large indexes don't just use disk; they compete for the same Buffer Cache (RAM) as your data.

---

## 5. How Staff Engineers Review Index PRs

When you review a junior/senior's PR, ask these 4 questions:

1. Is it redundant? (Does a composite index already cover this?)
2. Can it be partial? (Do we only need subset X?)
3. Is the column order correct? (High cardinality first? Range last?)
4. What is the HOT impact? (Is this a frequently updated column?)

---

## 6. Practice Exercise: The Final Boss 🐉

Design an indexing strategy for a Global Payment Gateway that:

- Stores 1 Billion transactions.
- Must query by transaction_id (UUID).
- Must generate reports by merchant_id and month.
- Must search customer_name using "fuzzy search" (e.g. jo matches John).
  Tell me your index design for these three requirements.

---
