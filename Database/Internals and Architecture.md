## How Seniors Really Learn PostgreSQL Internals (My Suggestion)

Instead of “reading internals”, we learn it in **4 mental layers**:

1. **Process & Memory Model** (How PostgreSQL runs)
2. **Write Path** (What happens when you INSERT/UPDATE)
3. **Read Path** (What happens when you SELECT)
4. **Background Workers & Housekeeping** (Why performance degrades over time)

If you understand these 4 layers, you understand PostgreSQL internals better than 80% of engineers.

---

# Layer 1 — Process & Memory Model (CRITICAL)

### What PostgreSQL *Actually Is*

PostgreSQL is **not a server with threads**.

👉 It is a **process manager**.

```
Client
  |
  v
Postmaster
  |
  +-- Backend Process (1 per connection)
  +-- Backend Process
  +-- Backend Process
```

### Why This Matters (Senior Insight)

* 1 connection = 1 OS process
* Too many connections = context switching + memory blowup
* This is why:

  * PgBouncer exists
  * Connection pooling is mandatory at scale

### Memory Areas (Simplified but Accurate)

| Memory Area          | Purpose                          |
| -------------------- | -------------------------------- |
| shared_buffers       | PostgreSQL’s main cache          |
| work_mem             | Per-operation (sort, hash)       |
| maintenance_work_mem | VACUUM, CREATE INDEX và ALTER TABLE ADD FOREIGN KEY             |
| OS Page Cache        | Often bigger than shared_buffers |

👉 **Senior rule**: PostgreSQL trusts the OS heavily.

---

# Layer 2 — Write Path (INSERT / UPDATE / DELETE)

This is where most misunderstandings happen.

### Step-by-step: INSERT

## The Step-by-Step Write Flow

### 1. The Request (Backend Process)

When a client sends a write query, a **Backend Process** is assigned to handle it. The process first finds the specific "page" (an 8KB block of data) that needs to be modified.

### 2. Shared Buffers & "Dirtying" Pages

* If the required page isn't already in the **Shared Buffer Pool** (RAM), the backend fetches it from the disk.
* The change is applied to the page in memory.
* This page is now marked as **"Dirty,"** meaning its in-memory version is newer than the version on the disk.

### 3. Write-Ahead Logging (WAL)

Before the transaction can be considered "committed," PostgreSQL must ensure the change is durable.

* A record of the change is created in the **WAL Buffers**.
* When you issue a `COMMIT`, the **WAL Writer** process flushes these logs from memory to the **WAL Files** (located in `pg_wal`) on the disk.
* **Crucial Concept:** This is a sequential write, which is extremely fast. Once the WAL is on disk, the database guarantees the data is safe, even if the power cuts out a millisecond later.

### 4. Background Writing (Asynchronous)

While the user gets a "Success" message after the WAL is flushed, the actual data files (the `.db` files) still have the old data.

* The **Background Writer** process periodically wakes up and moves some dirty pages from the Shared Buffers to the disk.
* This "trickles" the writes to the disk so that the system doesn't get overwhelmed.

### 5. The Checkpoint

A **Checkpoint** is a periodic event where PostgreSQL ensures all dirty pages in memory are synchronized with the data files on disk.

* The **Checkpointer** process flushes all remaining dirty buffers to the data files.
* Once finished, it marks a point in the WAL. Any WAL files older than this checkpoint can now be recycled or deleted because the data files are now up-to-date.


### Example Scenario

**Query:** `UPDATE accounts SET balance = 1000 WHERE user_id = 5;`

---

## The Visual Flow of a Write

### Step 1: Loading & Dirtying (In-Memory)

* **The Search:** The backend process looks in the **Shared Buffers** for the page containing `user_id = 5`. If it’s not there, it pulls it from the **Data File** (Disk) into the **Shared Buffers** (RAM).
* **The Change:** The process modifies the balance to `1000` inside the RAM.
* **Status:** This page is now **"Dirty"** (Memory \neq Disk).

### Step 2: The "Safety Net" (WAL)

* **Log Creation:** Before the change is "official," a small record of this update is placed in the **WAL Buffer**.
* **The Commit:** When you hit "Enter" and the transaction commits, the **WAL Writer** immediately pushes that record to the **WAL File** on disk.
* **The Result:** The database tells you "Update Successful." Even though the actual table file hasn't changed yet, the record is safe in the log.

### Step 3: The Background Sync

* **Lazy Writing:** The **Background Writer** notices the dirty page and, when the disk isn't too busy, it writes the updated page back to the **Data File**.

### Step 4: The Checkpoint (The Final Sync)

* **The Deadline:** Every few minutes (controlled by `checkpoint_timeout`), the **Checkpointer** forces **all** dirty pages to be written to the Data Files.
* **Cleanup:** It then creates a "Checkpoint" mark in the WAL, signaling that everything prior to this point is now safely stored in the main data files.

---

### Golden Rule

> **WAL is written before data pages**

This guarantees **crash safety**.

👉 **Would you like to dive deeper into how Checkpoints are tuned to prevent disk I/O spikes?**

### UPDATE is NOT an update

To understand the **UPDATE** path in detail, you have to look at how PostgreSQL handles "versions" of data. Unlike some databases that overwrite the old data in place, PostgreSQL uses a method called **MVCC (Multi-Version Concurrency Control)**.

When you run an `UPDATE`, PostgreSQL doesn't actually change the old row—it creates a **new** one.

---

## The Detailed "UPDATE" Flow

Imagine we are updating a user's age from **25** to **26**.

### 1. The Search & Lock

* The Backend process finds the page containing the row.
* It places a "Lock" on that specific row so other transactions don't try to update the same record at the exact same time.

### 2. Creating the New Version (The "Insert")

* Instead of erasing "25," PostgreSQL marks the old row as **expired**.
* It then creates a **brand new version** of the row with the value "26" in the same (or a new) page.
* Both versions exist on the disk for a short time.

### 3. Writing the WAL (The Journal)

* A record of this change (Old location \rightarrow New location) is written to the **WAL Buffer**.
* Upon `COMMIT`, this log is flushed to the **WAL File** on disk.

### 4. Pointer Update

* The database updates the **Indexes** (like a Primary Key index) to point to the new version of the row.

---

## Visualization of the Table Page

Before the update, your data page looks like this:
`[ Row 1 (Age: 25) ]`

After the `UPDATE` (but before cleanup), the page looks like this:
`[ Row 1 (Age: 25 - DEAD) ]  [ Row 2 (Age: 26 - LIVE) ]`

---

## Why does it do this?

1. **Consistency:** If another user is running a long report while you are updating, they can keep reading the "Old" version (Age 25) without being blocked. They see a consistent snapshot of the data.
2. **Speed:** Appending a new row is often faster than finding and precisely modifying bits of an old row.

---

## The Aftermath: VACUUM

Because `UPDATE` leaves "Dead" rows behind (often called **Bloat**), PostgreSQL has a background process called **Autovacuum**.

* **The Goal:** To find those "DEAD" rows (like the old Age 25 record) and mark that space as "Available" for future inserts.
* **The Result:** If you don't vacuum, your database files will keep growing forever, even if you aren't adding new data!

---

PostgreSQL does this instead:

```
OLD ROW (dead)
NEW ROW (live)
```

👉 This is **MVCC**

### Consequences

* Tables grow
* Indexes grow
* VACUUM is mandatory

---

# Layer 3 — Read Path (SELECT)
The **Read Path** in PostgreSQL is the journey a request takes to fetch data from the disk and deliver it to your screen. Like the Write Path, it is designed to avoid the "slow" disk as much as possible by using layers of memory.

---

## The Read Flow: Layer by Layer

PostgreSQL uses a **Dual-Cache** strategy. It doesn't just use its own memory; it "collaborates" with your Operating System (Linux/Windows) to keep data close.

### 1. Step 1: The Request (Backend Process)

When you run a `SELECT`, your dedicated backend process calculates which **Pages** (8KB blocks) it needs. It doesn't look for "rows" yet; it looks for the blocks that *contain* those rows.

### 2. Step 2: The "L1 Cache" (Shared Buffers)

The process first checks the **Shared Buffer Pool** (PostgreSQL’s private RAM).

* **Cache Hit:** If the page is there, the process reads it instantly. This is the fastest possible result.
* **Cache Miss:** If the page is missing, the process must "request" it from the Operating System.

### 3. Step 3: The "L2 Cache" (OS Page Cache)

The OS (Linux) has its own memory area called the **Page Cache**.

* **OS Hit:** If another process (or a previous query) recently read that file, the OS already has it in RAM. It copies the page into PostgreSQL's Shared Buffers. This is still very fast (RAM-to-RAM).
* **OS Miss:** If neither has it, we hit the **"I/O Wall."**

### 4. Step 4: Physical Disk I/O

The OS finally goes to the physical storage (SSD/HDD). It reads the 8KB block, places it in the **OS Cache**, and then delivers a copy to the **Shared Buffers**.

* This is the slowest part of the path (can be 100x–1000x slower than RAM).

---

## Example: The "Hot" vs "Cold" Read

Imagine you have a table of **Products**.

| Scenario | What happens? | Speed |
| --- | --- | --- |
| **Cold Read** (First time) | Page is not in RAM. Postgres goes to Disk \rightarrow OS Cache \rightarrow Shared Buffers. | **Slow** |
| **Warm Read** (Second time) | Page was kicked out of Shared Buffers but is still in **OS Cache**. | **Fast** |
| **Hot Read** (Repeat query) | Page is sitting directly in **Shared Buffers**. | **Blazing Fast** |

---

## The "Clock Sweep" (How Postgres chooses what to keep)

Since memory is limited, Postgres must decide which pages to "kick out" to make room for new ones. It uses a **Clock Sweep** algorithm:

1. Each page in memory has a **Usage Count** (0 to 5).
2. A "pointer" (like a clock hand) moves around the buffers.
3. Every time a page is touched, its count goes up.
4. When the clock hand passes a page, it *decreases* the count.
5. If the count hits **0**, that page is evicted (deleted from RAM) to make space for a new page.

---

## How to see this in action

You can actually see the Read Path working by using `EXPLAIN (ANALYZE, BUFFERS)`.

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM accounts WHERE id = 100;

```

**Look for these in the output:**

* **`shared hit`**: Found in PostgreSQL Shared Buffers (L1).
* **`shared read`**: Had to ask the OS (L2 or Disk).

Would you like me to show you how to calculate your "Cache Hit Ratio" to see if your server has enough RAM for your data?

---

# Layer 4 — Background Workers (Why Things Break Over Time)

### Key Workers

| Worker            | Role               |
| ----------------- | ------------------ |
| Checkpointer      | Flush dirty pages  |
| Background writer | Smooth disk writes |
| Autovacuum        | Clean dead tuples  |
| WAL writer        | Flush WAL          |

### Autovacuum (Most Ignored, Most Important)

Autovacuum:

* Removes dead rows
* Updates statistics
* Prevents transaction ID wraparound

👉 **Turning it off is a career-ending mistake**

---

# PostgreSQL Architecture — One Picture in Your Head

Think of PostgreSQL as:

> **An append-only system with garbage collection**

* Writes append new versions
* Reads filter visibility
* Vacuum cleans old versions

---

## What I Recommend You Do (Concrete Plan)

### Step 1 — Read the *Right* Docs (Not All Docs)

Read these **in order**:

1. PostgreSQL docs:

   * “Architecture Overview”
   * “Database Physical Storage”
2. Wiki:

   * MVCC
   * Autovacuum

Do **not** read source code yet.

---

### Step 2 — Observe Internals Live (Hands-on)

Run these in psql:

```sql
SELECT pid, state, query
FROM pg_stat_activity;
```

```sql
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables;
```

```sql
SELECT * FROM pg_locks;
```

👉 This connects theory to reality.

---

### Step 3 — Senior Mental Models (Memorize These)

* PostgreSQL is **process-based**
* UPDATE = INSERT + DELETE
* WAL first, data later
* VACUUM is garbage collection
* Reads don’t block writes (usually)

If these feel *obvious*, you’re leveling up.

---

## Optional (Advanced, but Powerful)

If you want to go deeper:

* Page layout (8KB pages)
* FSM / VM (Free Space Map / Visibility Map)
* HOT updates

I’ll only go here once the fundamentals are solid.

---

## Next Mentoring Question (Important)

To tailor the next deep dive, tell me **one thing**:

👉 Do you want to understand PostgreSQL internals for:

1. Performance tuning
2. Debugging production incidents
3. Database design at scale
4. Senior/Staff interviews

Pick **one** — this is how seniors focus.
