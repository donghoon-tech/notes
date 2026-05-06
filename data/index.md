# Index

## Table of Contents
1. [Data Structures and Physical Organization](#data-structures-and-physical-organization)
2. [Composite Indexes and Selectivity](#composite-indexes-and-selectivity)
3. [Optimizer Mechanics and Execution Plans](#optimizer-mechanics-and-execution-plans)
4. [Indexing Anti-patterns and SARGability](#indexing-anti-patterns-and-sargability)
5. [Locks and Index Interactions](#locks-and-index-interactions)
6. [Write Performance and Maintenance](#write-performance-and-maintenance)
7. [Advanced Indexing and Partitioning](#advanced-indexing-and-partitioning)
8. [Operations and Monitoring](#operations-and-monitoring)
9. [Common Misconceptions](#common-misconceptions)

---

## Data Structures and Physical Organization

### B-Tree vs. B+Tree
Most RDBMS utilize B+Tree structures rather than standard B-Trees. In a B+Tree, internal nodes only store keys to guide the search, while all actual data (or pointers to data) reside in leaf nodes. Leaf nodes are linked as a doubly-linked list.
- **Why B+Tree?** It provides superior range scan performance due to the linked leaf nodes and offers a higher branching factor (fan-out) since internal nodes don't store data, reducing the tree height and disk I/O.

### Clustered vs. Non-Clustered Indexes
- **Clustered Index:** Defines the physical order of data in the table. In MySQL's InnoDB, the Primary Key (PK) is the clustered index. The leaf nodes contain the actual row data.
- **Non-Clustered (Secondary) Index:** Stores the index key and a pointer to the clustered index key.
- **Implication:** In InnoDB, every secondary index lookup requires a second traversal of the clustered index (Bookmark Lookup/Key Lookup) to retrieve the full row, unless it is a covering index.

---

## Composite Indexes and Selectivity

### The Leftmost Prefix Rule
A composite index is sorted sequentially from the first column. The query must filter by the leading columns to utilize the index. If a middle column is skipped, subsequent columns cannot use the index for searching.

Assuming an index on `(A, B, C)`:

| Query Condition | Index Usage | Note |
| :--- | :--- | :--- |
| `WHERE A = 1` | Uses A | |
| `WHERE A = 1 AND B = 2` | Uses A, B | |
| `WHERE A = 1 AND B = 2 AND C = 3`| Uses A, B, C | Fully utilized |
| `WHERE A = 1 AND C = 3` | Uses A only | `C` cannot use the index |
| `WHERE B = 2` | Cannot use | Missing leftmost prefix `A` |

### Column Ordering and Cardinality
The essence of index design is maximizing filtering efficiency. While high cardinality columns should generally come first, the actual query pattern (filtering + sorting) dictates the final design.

**Practical Example: Order List Query**
```sql
SELECT * FROM orders
WHERE sender_id = 123 AND status = 'PAID'
ORDER BY created_at DESC;
```
**Optimal Index:** `(sender_id, created_at)`
- **Why `sender_id` first?** It isolates a specific user. One user's orders represent a tiny fraction of the whole table (High Cardinality). It filters out the most data immediately.
- **Why exclude `status`?** Low cardinality. `sender_id` already narrows the scope significantly; adding `status` provides minimal benefit while increasing write and storage costs.
- **Why `created_at` second?** To eliminate the `filesort` operation for `ORDER BY`. Because the index is already sorted by `created_at` within each `sender_id`, the sorting is essentially free.

### Covering Indexes
A covering index includes all columns required for the `SELECT`, `WHERE`, `GROUP BY`, and `ORDER BY` clauses.
- **Mechanism:** It prevents the "Secondary Index -> Clustered Index" double lookup. The database retrieves the necessary data directly from the secondary index leaf nodes, significantly reducing random disk I/O.
- **Verification:** In the `EXPLAIN` output, a covering index is confirmed when the `Extra` column displays `Using index`. Note that this is distinctly different from `type: index`, which implies an index full scan.

---

## Optimizer Mechanics and Execution Plans

### Index Condition Pushdown (ICP)
ICP is an engine optimization where the evaluation of index column conditions is pushed down to the storage engine *before* performing the table lookup (Clustered Index lookup).
- **Core Concept:** In the order list example above, the engine scans the index for `sender_id = 123`, then retrieves the row to check `status = 'PAID'`. If `status` were in the index, ICP would evaluate it using the index data alone, reducing unnecessary table lookups.
- **Warning:** ICP is not a justification for adding low-cardinality columns to an index. If cardinality is low, the reduction in table lookups is marginal, and the index bloat is not worth the write penalty.

### Cost-Based Optimization (CBO) and Thresholds
The optimizer uses table statistics to estimate the I/O cost of access paths.
- **The 20-30% Rule:** The optimizer typically abandons an index and resorts to a Full Table Scan if it estimates that the query will retrieve more than approximately 20-30% of the table's rows. At this threshold, the overhead of random I/O required for secondary index lookups exceeds the cost of sequential multi-block reads used in a full scan.

---

## Indexing Anti-patterns and SARGability

To utilize an index, the predicate must be SARGable (Searchable Arguments).

- **Functions/Operations:** `WHERE YEAR(created_at) = 2024` disables the index. Rewrite as `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'`.
- **Implicit Type Conversion:** Comparing a `VARCHAR` column with an integer forces type casting, invalidating the index.
- **Leading Wildcards:** `LIKE '%keyword'` cannot use an index. `LIKE 'keyword%'` can.
- **OR Conditions:** Splitting a composite index with `OR` usually breaks index usage. Rewrite as `UNION ALL` or rely on Index Merge if appropriate.

---

## Locks and Index Interactions

In InnoDB, row-level locking is intrinsically tied to indexes. It uses Next-Key Locks (Record Lock + Gap Lock) on index records to manage concurrency.
- **Unindexed `UPDATE`/`DELETE` Hazard:** If a modifying query filters by an unindexed column, the storage engine cannot isolate the specific rows. Consequently, it traverses the clustered index, effectively placing locks on every row in the table (a full table lock). This is a primary cause of severe concurrency bottlenecks and deadlocks in production environments.

---

## Write Performance and Maintenance

Every `INSERT`, `UPDATE`, or `DELETE` requires updating all relevant indexes.
- **The Cost of Indexing:** Excessive indexing severely degrades write performance. Each new row requires finding the correct page in the B+Tree and inserting the key, potentially causing page splits.
- **Maintenance:** Index fragmentation occurs as data is modified. Tools like `pt-online-schema-change` or native Online DDL allow adding/rebuilding indexes without locking the table.

---

## Advanced Indexing and Partitioning

### Partitioning Strategy
- **Partition Pruning:** Queries targeting partitioned tables must include the partition key in the `WHERE` clause. Without it, the engine scans all partitions, negating the benefits of partitioning.
- **Local vs. Global Indexes:**
    - **Local Index:** The index is partitioned exactly like the table. Highly efficient for partition maintenance (e.g., dropping older partitions) but requires the partition key in the query to avoid scanning indexes across all partitions.
    - **Global Index:** A single index covering all partitions. Essential for enforcing unique constraints across the entire table, but severely impacts performance during partition operations, as the global tree must be heavily modified.

### Specialized Indexes
- **Partial Index:** Indexes only a subset of rows (e.g., `WHERE deleted_at IS NULL`).
- **Descending Index:** Critical for multi-column indexes with mixed sort directions (e.g., `a ASC, b DESC`).
- **Full-Text Index:** Uses an inverted index for complex text searching.
- **R-Tree:** Used for spatial (GIS) data.
- **Bitmap Index:** Ideal for OLAP environments with low-cardinality columns, but unsuitable for OLTP due to heavy write locking.

---

## Operations and Monitoring

### Troubleshooting Workflow
1. Identify slow queries via Slow Query Logs.
2. Run `EXPLAIN ANALYZE` to check execution plans. Look for `type: ALL` (Full Scan) or `Using filesort`.
3. Check for stale statistics and run `ANALYZE TABLE` if the optimizer makes poor choices.

### Safe Index Removal Process
Removing an index blindly based on assumptions can cause immediate production incidents.
1. **Verification:** Query `sys.schema_unused_indexes` or `performance_schema.table_io_waits_summary_by_index_usage` to find indexes with zero or near-zero usage over a meaningful uptime period.
2. **Shadow Testing (Invisible Indexes):** In modern RDBMS (e.g., MySQL 8.0+), alter the index to `INVISIBLE`. This hides it from the optimizer without dropping the physical data structure.
3. **Monitoring:** Monitor query performance and slow logs. If a regression occurs, the index can be made `VISIBLE` instantly without the massive I/O cost of rebuilding it.
4. **Drop:** Only execute `DROP INDEX` after a sufficient observation period confirms no negative impact.

---

## Common Misconceptions

- **"High cardinality columns must ALWAYS go first."**
    - *Reality:* You must analyze the specific query pattern. If `ORDER BY` is fixed, placing the sorting column strategically might save more CPU than filtering a few extra rows. Equality conditions (`=`) are generally the strongest filters and should precede range conditions (`>`, `<`).
- **"More indexes equal better performance."**
    - *Reality:* Every index is a tax on write operations. On write-heavy tables, redundant or marginally useful indexes cause severe `INSERT/UPDATE` bottlenecks.
- **"Every column in the WHERE clause should be indexed."**
    - *Reality:* Indexing low-cardinality columns (`status`, `gender`, `is_active`) individually is almost always useless. The optimizer will likely choose a Full Table Scan anyway, as sequential disk reads are faster than the random I/O required by traversing a non-selective index.