# 🧠 Today I Learned — PostgreSQL Performance Tuning & Index Internals

While dealing with data loading delays and timeouts during execution of large queries,  
I整理ed how PostgreSQL actually executes queries, why they become slow,  
and what role indexes play in all of this.

---

## 🚨 Problem Summary

- An API that fetches a certain list was timing out.
- The execution plan showed that one of the subqueries was running as a **Full Sequential Scan**.
- This subquery was comparing a key column (something like `order_number`) in the `WHERE` clause,  
  but since no effective index was being used, it ended up repeatedly scanning the entire table.

---

## ⚙️ Root Cause Summary

1️⃣ **No index on the key column**  
   - The key column had no index, **or**  
     even if an index existed, PostgreSQL couldn’t use it effectively.

2️⃣ **Column type mismatch (`varchar` vs `text`)**  
   - When comparing different data types, PostgreSQL automatically casts one side (e.g. `::text`).  
   - In that case, the index no longer matches the original data type definition and is effectively **ignored**.

3️⃣ **Correlated subquery structure**  
   - The subquery was executed **once per row** of the main table.  
   - Example: if the parent table has 1,000 rows, the subquery also runs 1,000 times.  
   - Without an index, that means scanning the entire child table every time → time complexity approaches O(N²).

---

## 🧩 How PostgreSQL Executes JOINs and WHERE Clauses

### 1. Basic Flow

When PostgreSQL performs a JOIN, it typically chooses one of the following strategies:

| Strategy           | Description                                               | Typical Use Case                             |
|--------------------|-----------------------------------------------------------|----------------------------------------------|
| **Nested Loop Join** | For each row on one side, look up matching rows on the other | Small table joined to a larger, indexed table |
| **Hash Join**      | Build an in-memory hash table from one side, scan the other | Both sides large, but can fit in memory      |
| **Merge Join**     | Sort both inputs and merge them in order                  | Both inputs sorted or can be sorted cheaply  |

---

### 2. Nested Loop Join — How It Actually Works

```sql
SELECT *
FROM orders o
LEFT JOIN notes n
  ON n.order_number = o.order_number;
```

1️⃣ Read one row from `orders`.  
2️⃣ Use that row’s `order_number` to search in `notes`.  
3️⃣ If matching rows are found, combine them into the JOIN result.  
4️⃣ Move on to the next row.

In other words:

> The engine “reads one row from one table, then performs a ‘lookup’ in the other table for every single row.”

---

### 3. With vs Without Indexes

| Situation       | What PostgreSQL Does                           | Result        |
|-----------------|------------------------------------------------|---------------|
| No index        | Scans the entire table with the `WHERE` filter | Full Seq Scan |
| Index available | Uses the sorted index tree to find matches     | Index Scan    |

For example, if the parent table has 1,000 rows and the child table has 50,000 rows:

- **Without an index:**  
  1,000 × 50,000 = 50,000,000 comparisons → huge 🔥  
- **With an index:**  
  1,000 × `O(log N)` index lookups → hundreds or thousands of times faster ⚡

---

## 🔍 What Is an Index and How Does It Work?

### 1. What is an Index?

> An index is a separate, sorted data structure used to find rows quickly.  
> PostgreSQL primarily uses **B-Tree** indexes.

So instead of scanning the entire table,  
PostgreSQL can navigate the sorted index tree and find what it needs in **O(log N)** time.

---

### 2. Why Indexes Are Fast

- An index is managed separately from the main table as a **sorted key → pointer** structure.
- When a query needs rows that match a condition,  
  it can use the index to jump directly to the relevant rows instead of scanning everything.

---

### 3. Pros and Cons of Indexes

| Pros                                             | Cons                                                |
|--------------------------------------------------|-----------------------------------------------------|
| Much faster `SELECT` / `JOIN` / `ORDER BY`       | Slower `INSERT` / `UPDATE` / `DELETE` (index maintenance) |
| Lower sorting cost                               | Extra disk usage                                   |
| Avoid unnecessary full table scans               | Too many indexes can complicate query planning     |

---

## 🧠 Common Reasons Why Indexes Are Not Used

| Reason                    | Explanation                                                 |
|---------------------------|-------------------------------------------------------------|
| **Type mismatch**         | Comparing `varchar` vs `text` can use different operators; indexes may be ignored |
| **Functions / casts**     | `col::text`, `lower(col)`, etc. prevent using plain column indexes |
| **Wildcard patterns**     | `LIKE '%abc'` can’t use a normal B-Tree index              |
| **Overly broad conditions** | If most rows match, PostgreSQL may prefer a full scan         |
| **Stale statistics**      | Without fresh stats, the planner makes bad decisions → run `ANALYZE` |
| **Collation mismatch**    | Different locale settings may make ordered comparisons incompatible with an index |

---

## ⚙️ Types of Indexes in PostgreSQL

| Type       | Characteristics                               | Typical Use Case                      |
|------------|-----------------------------------------------|---------------------------------------|
| **B-Tree** | Default. Supports `=`, `<`, `>`, `ORDER BY`   | Most equality and range queries       |
| **GIN**    | Inverted index for JSONB, arrays, full-text   | `@>`, `?`, text search on JSON/arrays |
| **GiST**   | Generalized search tree                      | Spatial, ranges, similarity queries   |
| **BRIN**   | Block range index, very small and coarse      | Huge, naturally ordered tables (e.g. logs) |
| **Hash**   | Equality comparison only                      | Rarely used; B-Tree usually better    |

---

## 🧱 Example Index Designs

### 1️⃣ Single-column Index

```sql
CREATE INDEX idx_notes_order_number
  ON notes (order_number);
```

- Speeds up `WHERE order_number = ?`

---

### 2️⃣ Composite Index (Filter + Sort)

```sql
CREATE INDEX idx_notes_order_number_created
  ON notes (order_number, created_at DESC);
```

- Optimizes queries like:  
  `WHERE order_number = ? ORDER BY created_at DESC`
- Handles both the filtering and the sorting in one index.

---

### 3️⃣ Partial Index

```sql
CREATE INDEX idx_notes_active
  ON notes (order_number)
  WHERE deleted_at IS NULL;
```

- Only indexes “active” rows, reducing index size and overhead.
- Great when queries almost always use a certain condition.

---

### 4️⃣ Covering Index (INCLUDE)

```sql
CREATE INDEX idx_notes_cover
  ON notes (order_number)
  INCLUDE (created_at, user_id);
```

- Allows some queries to be served entirely from the index  
  (Index Only Scan), without touching the main table.

---

## 🔎 How to Analyze Performance: EXPLAIN / ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM notes
WHERE order_number = 'ORD123';
```

### Important fields

| Field              | Meaning                                  |
|--------------------|------------------------------------------|
| **Seq Scan**       | Full table scan                          |
| **Index Scan**     | Uses an index                            |
| **Bitmap Index Scan** | Hybrid approach for combining conditions |
| **cost**           | Estimated execution cost                 |
| **actual time**    | Real measured execution time             |
| **rows / loops**   | Rows processed and loop counts           |

---

## 🧩 Practical Index Tuning Checklist

1️⃣ **Unify data types**  
   - Make sure the types of columns you compare (`text`, `varchar`, etc.) are identical.  
   - Mismatched types can invalidate index usage.

2️⃣ **Add necessary indexes**  
   - Create indexes on columns frequently used in `WHERE` and `JOIN`.  
   - Use composite indexes if sorting is also common.

3️⃣ **Remove casts / functions from indexed columns in `WHERE`**  
   - Avoid `col::text`, `lower(col)`, etc. on indexed columns in conditions.  
   - If you must use them, consider an expression index instead.

4️⃣ **Refresh statistics**

   ```sql
   ANALYZE notes;
   ```

5️⃣ **Re-check the plan**  
   - Use `EXPLAIN ANALYZE` and confirm that `Seq Scan` has turned into `Index Scan` where you expect it.

---

## ⚡ Example of Performance Difference

| Situation    | Execution Plan                                | Expected Performance |
|-------------|-----------------------------------------------|----------------------|
| No index    | `Seq Scan (cost=0.00..1300.00)`               | Seconds              |
| With index  | `Index Scan using idx_notes_order_number_created` | Milliseconds         |

---

## 🚀 Conclusion

The core issues in the original problem were:

1️⃣ **No usable index or type mismatches preventing index usage**, and  
2️⃣ **A correlated subquery structure that repeatedly scanned the same table**.

The main fixes:

```sql
-- (1) Unify data type
ALTER TABLE notes
  ALTER COLUMN order_number TYPE text;

-- (2) Create index
CREATE INDEX idx_notes_order_number_created
  ON notes (order_number, created_at DESC);
```

Once the execution plan switches to `Index Scan`,  
query performance can easily improve by an order of magnitude or more 🚀

---

## 🧭 Key Takeaways

| Topic                  | Summary                                                            |
|------------------------|--------------------------------------------------------------------|
| JOIN behavior          | Repeatedly looks up matching rows from the other table per row     |
| Role of indexes        | Turn lookups from O(N) into O(log N)                              |
| Type consistency       | `varchar` vs `text` mismatches can kill index usage               |
| Correlated subqueries  | Can cause massive performance issues when combined with full scans |
| Reading EXPLAIN plans  | Essential for finding bottlenecks                                 |
| Composite indexes      | Can optimize both filtering and sorting                           |
| Partial / expression indexes | Useful for conditional or function-based queries                   |
| Statistics (ANALYZE)   | Critical for the planner to make good decisions                   |

---

## ✅ Final Insight

> The heart of query optimization is “making the database read less data.”  
> Indexes are one of the most powerful tools to achieve that,  
> but they can be completely neutralized by type mismatches  
> or poorly structured queries.

This whole debugging process reinforced that:

- Index design is not an optional afterthought, but a core part of schema design.  
- `EXPLAIN ANALYZE` is your best friend for understanding what’s really happening.  
- Good performance comes from thinking about data types, key columns, and access patterns together.
