# SQL Complex Query & Senior Interview Preparation Guide
### Designed for 14+ years of experience
**Primary stack:** SQL Server 2019/2022, T-SQL, OLTP, data warehousing, ETL, APIs and microservices

---

## How to use this guide

At 14+ years, interviews are not only testing whether you can write a query. They are testing whether you can:

- Translate ambiguous business requirements into a correct data model and query.
- Explain correctness, performance, concurrency and operational trade-offs.
- Diagnose production problems using evidence rather than guesswork.
- Design for scale, availability, security, maintainability and cost.
- Review other engineers' SQL and improve the engineering system around them.
- Communicate risk and make pragmatic decisions under incomplete information.

For every complex SQL answer, explain:

1. **Semantics:** What result should be returned, including nulls, duplicates and edge cases?
2. **Access path:** Which indexes, partitions, joins and filters will be used?
3. **Data volume:** How does the solution behave at 1 million, 100 million and 1 billion rows?
4. **Concurrency:** What happens when reads and writes happen at the same time?
5. **Operations:** How will it be monitored, deployed, rolled back and maintained?

> Senior rule: never claim that a query is fast without stating the data volume, selectivity, indexes, execution plan and measurement method.

---

# TABLE OF CONTENTS

**PART 1 - Senior SQL foundations**

1.1 Correctness before cleverness · 1.2 Logical query processing · 1.3 Relational thinking · 1.4 NULL and three-valued logic · 1.5 Set-based design · 1.6 OLTP versus analytical SQL

**PART 2 - Complex query patterns**

2.1 Top-N per group · 2.2 Gaps and islands · 2.3 Running totals and windows · 2.4 Latest row per entity · 2.5 Consecutive events · 2.6 Hierarchies · 2.7 Recursive CTEs · 2.8 Relational division · 2.9 Dynamic pivots · 2.10 Temporal queries · 2.11 Change detection · 2.12 Deduplication

**PART 3 - Query design and performance**

3.1 SARGability · 3.2 Join strategy · 3.3 Cardinality estimation · 3.4 Execution plans · 3.5 Index design · 3.6 Covering and filtered indexes · 3.7 Statistics · 3.8 Parameter sniffing · 3.9 Temp tables versus table variables · 3.10 CTEs and materialization · 3.11 Pagination · 3.12 Batch processing

**PART 4 - Transactions and concurrency**

4.1 ACID · 4.2 Isolation levels · 4.3 Blocking and deadlocks · 4.4 Optimistic concurrency · 4.5 Retry and idempotency · 4.6 Queue patterns · 4.7 Transaction boundaries

**PART 5 - Data architecture**

5.1 Normalization and denormalization · 5.2 Partitioning · 5.3 OLTP schema design · 5.4 Star schema · 5.5 Slowly changing dimensions · 5.6 CDC and event-driven data · 5.7 Multi-tenancy · 5.8 Security and governance

**PART 6 - Production troubleshooting**

6.1 Query regression · 6.2 CPU and memory pressure · 6.3 Blocking incidents · 6.4 TempDB · 6.5 Storage and log growth · 6.6 Availability and recovery · 6.7 Observability

**PART 7 - Senior interview questions**

7.1 Complex SQL questions · 7.2 Performance questions · 7.3 Architecture questions · 7.4 Leadership and ownership questions · 7.5 Coding exercises · 7.6 Interview answer framework

**PART 8 - Final revision checklist**

8.1 Query checklist · 8.2 Production checklist · 8.3 Questions to ask the interviewer

---

# PART 1 - SENIOR SQL FOUNDATIONS

## 1.1 Correctness before cleverness

A query is correct only when it handles the business definition, not merely when it returns rows.

Before writing SQL, clarify:

- What is the grain of the result: one row per customer, order, day or event?
- Should records with no matching child rows appear?
- Are duplicate business events possible?
- Which timestamp and timezone define the business day?
- Are soft-deleted, cancelled or unapproved records included?
- Are amounts gross, net, tax-inclusive or currency-normalized?
- What should happen when a value is unknown (`NULL`)?

Example: “Top customers by revenue” is incomplete. A senior engineer asks whether revenue means paid invoices, order value, refunds excluded, and whether customers are ranked globally or within each region.

## 1.2 Logical query processing order

SQL is written in one order but logically processed in another:

```text
FROM / JOIN
WHERE
GROUP BY
HAVING
SELECT
DISTINCT
ORDER BY
OFFSET / FETCH
```

This explains common errors:

- A `WHERE` predicate on the right side of a `LEFT JOIN` can turn it into an inner join.
- A `SELECT` alias usually cannot be referenced in the same `SELECT` list.
- Filtering after aggregation requires `HAVING`, or an outer query.
- `TOP` without `ORDER BY` is not deterministic.

## 1.3 NULL and three-valued logic

`NULL` means unknown or missing; it is not zero, an empty string or false.

```sql
-- Wrong when the intention is to find missing values
WHERE Discount <> 0

-- Correct
WHERE Discount IS NULL OR Discount <> 0
```

Important interview points:

- `NULL = NULL` is not true; it evaluates to unknown.
- `COUNT(*)` counts rows; `COUNT(column)` ignores nulls.
- `NOT IN` can return no rows when its subquery contains `NULL`; prefer `NOT EXISTS` when nullability is possible.
- Use `COALESCE` only when replacing unknown with a business-valid default, not just to hide data-quality problems.

## 1.4 Set-based design

Prefer a set-based operation over a cursor or row-by-row loop. If procedural processing is required, make it explicit and bounded.

```sql
-- Set-based update
UPDATE o
SET    o.Status = 'Overdue'
FROM   dbo.Orders AS o
WHERE  o.Status = 'Open'
AND    o.DueDate < SYSUTCDATETIME();
```

For millions of rows, even a set-based update may create excessive locks and transaction-log growth. Use controlled batches, a suitable index and an observable retry strategy.

## 1.5 OLTP versus analytical SQL

| Concern | OLTP | Analytical / reporting |
|---|---|---|
| Grain | Individual business transaction | Aggregate or dimensional fact |
| Design | Normalized | Star/snowflake or semantic model |
| Workload | Short, selective reads/writes | Large scans and aggregations |
| Indexing | Narrow access-path indexes | Columnstore, partitioning, aggregates |
| Consistency | Immediate transactional correctness | Defined refresh/latency contract |
| Concurrency | Many concurrent sessions | Fewer, heavier queries |

Do not solve a reporting workload by adding dozens of indexes to a transactional table without considering write cost, blocking and storage.

---

# PART 2 - COMPLEX QUERY PATTERNS

The examples use these logical tables:

```text
Customers(CustomerId, TenantId, Name, Region, CreatedAt)
Orders(OrderId, CustomerId, OrderDate, Status, TotalAmount, UpdatedAt)
OrderItems(OrderId, ProductId, Quantity, UnitPrice)
Events(EventId, EntityId, EventType, EventTime, Payload)
Employee(EmployeeId, ManagerId, Name, DepartmentId)
```

## 2.1 Top-N per group

Question: return the three most expensive completed orders for each customer.

```sql
WITH RankedOrders AS
(
    SELECT  o.CustomerId,
            o.OrderId,
            o.TotalAmount,
            o.OrderDate,
            ROW_NUMBER() OVER
            (
                PARTITION BY o.CustomerId
                ORDER BY o.TotalAmount DESC, o.OrderId DESC
            ) AS rn
    FROM dbo.Orders AS o
    WHERE o.Status = 'Completed'
)
SELECT CustomerId, OrderId, TotalAmount, OrderDate
FROM RankedOrders
WHERE rn <= 3;
```

Use `ROW_NUMBER` when exactly N rows are required, `RANK` when ties should share a rank with gaps, and `DENSE_RANK` when ties should share a rank without gaps.

## 2.2 Latest row per entity

Question: get the current status for every order from an append-only status history.

```sql
WITH LatestStatus AS
(
    SELECT  h.OrderId,
            h.Status,
            h.ChangedAt,
            ROW_NUMBER() OVER
            (
                PARTITION BY h.OrderId
                ORDER BY h.ChangedAt DESC, h.StatusHistoryId DESC
            ) AS rn
    FROM dbo.OrderStatusHistory AS h
)
SELECT OrderId, Status, ChangedAt
FROM LatestStatus
WHERE rn = 1;
```

The tie-breaker is essential. Ordering only by a timestamp is nondeterministic when two changes have the same timestamp.

## 2.3 Running totals and moving windows

```sql
SELECT  CustomerId,
        OrderDate,
        TotalAmount,
        SUM(TotalAmount) OVER
        (
            PARTITION BY CustomerId
            ORDER BY OrderDate, OrderId
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS LifetimeAmount,
        AVG(TotalAmount) OVER
        (
            PARTITION BY CustomerId
            ORDER BY OrderDate, OrderId
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS SevenOrderAverage
FROM dbo.Orders
WHERE Status = 'Completed';
```

Specify `ROWS` when physical rows are intended. The default frame can behave differently with duplicate ordering values.

## 2.4 Gaps and islands

Question: find consecutive days on which a customer placed an order.

```sql
WITH DistinctDays AS
(
    SELECT DISTINCT CustomerId, CAST(OrderDate AS date) AS OrderDay
    FROM dbo.Orders
    WHERE Status = 'Completed'
), Numbered AS
(
    SELECT CustomerId,
           OrderDay,
           DATEADD(day,
                   -ROW_NUMBER() OVER
                    (PARTITION BY CustomerId ORDER BY OrderDay),
                   OrderDay) AS IslandKey
    FROM DistinctDays
)
SELECT CustomerId,
       MIN(OrderDay) AS StartDay,
       MAX(OrderDay) AS EndDay,
       COUNT(*) AS ConsecutiveDays
FROM Numbered
GROUP BY CustomerId, IslandKey;
```

The normalized date minus row number is constant within each consecutive island. This pattern also applies to login streaks, machine uptime and subscription periods.

## 2.5 Gaps in a sequence

```sql
WITH Numbered AS
(
    SELECT Id, LEAD(Id) OVER (ORDER BY Id) AS NextId
    FROM dbo.Invoice
)
SELECT Id + 1 AS MissingStart,
       NextId - 1 AS MissingEnd
FROM Numbered
WHERE NextId > Id + 1;
```

Clarify whether gaps are actually defects. In distributed systems and rolled-back transactions, gaps in surrogate keys are normally expected.

## 2.6 Relational division

Question: find customers who bought **every** product in a required product set.

```sql
SELECT c.CustomerId
FROM dbo.Customers AS c
WHERE NOT EXISTS
(
    SELECT 1
    FROM dbo.RequiredProducts AS r
    WHERE NOT EXISTS
    (
        SELECT 1
        FROM dbo.OrderItems AS oi
        JOIN dbo.Orders AS o ON o.OrderId = oi.OrderId
        WHERE o.CustomerId = c.CustomerId
          AND o.Status = 'Completed'
          AND oi.ProductId = r.ProductId
    )
);
```

This “there does not exist a required item that the customer does not have” pattern is safer than a naïve count when duplicates or missing values exist.

## 2.7 Hierarchies

```sql
WITH Org AS
(
    SELECT EmployeeId, ManagerId, Name, 0 AS Depth,
           CAST(Name AS nvarchar(max)) AS Path
    FROM dbo.Employee
    WHERE EmployeeId = @RootEmployeeId

    UNION ALL

    SELECT e.EmployeeId, e.ManagerId, e.Name, o.Depth + 1,
           CONCAT(o.Path, N' / ', e.Name)
    FROM dbo.Employee AS e
    JOIN Org AS o ON e.ManagerId = o.EmployeeId
    WHERE o.Depth < @MaxDepth
)
SELECT EmployeeId, ManagerId, Name, Depth, Path
FROM Org
OPTION (MAXRECURSION 100);
```

Production considerations:

- Protect against cycles and define a maximum depth.
- Index `Employee.ManagerId`.
- Decide whether the hierarchy is read frequently enough to justify a closure table or materialized path.
- Use an iterative or graph-based design when the hierarchy is very large or changes frequently.

## 2.8 Dynamic pivot

Prefer conditional aggregation when columns are known:

```sql
SELECT CustomerId,
       SUM(CASE WHEN Status = 'Paid'    THEN TotalAmount ELSE 0 END) AS PaidAmount,
       SUM(CASE WHEN Status = 'Pending' THEN TotalAmount ELSE 0 END) AS PendingAmount,
       SUM(CASE WHEN Status = 'Failed'  THEN TotalAmount ELSE 0 END) AS FailedAmount
FROM dbo.Orders
GROUP BY CustomerId;
```

For dynamic columns, use `QUOTENAME` for identifiers, `sp_executesql` for values, and an allow-list of permitted columns. Never concatenate raw user input into dynamic SQL.

## 2.9 Change detection

For synchronizing two sets, use a null-safe comparison. `EXCEPT` is useful for set differences, but remember that it removes duplicates.

```sql
SELECT CustomerId, Name, Region
FROM dbo.CustomerSnapshot
EXCEPT
SELECT CustomerId, Name, Region
FROM dbo.CustomerSource;
```

For high-volume synchronization, compare a stable row hash only as an optimization and still handle hash collisions according to the business risk. Use CDC, Change Tracking or an outbox when change history and reliable delivery matter.

## 2.10 Deduplication with deterministic retention

```sql
WITH Duplicates AS
(
    SELECT  CustomerId,
            Email,
            CustomerRecordId,
            ROW_NUMBER() OVER
            (
                PARTITION BY NormalizedEmail
                ORDER BY IsVerified DESC, UpdatedAt DESC, CustomerRecordId
            ) AS rn
    FROM dbo.Customer
)
SELECT *
FROM Duplicates
WHERE rn > 1;
```

Do not delete duplicates before defining the survivor rule, checking foreign keys, auditing the merge and considering concurrent inserts. Add a unique constraint or filtered unique index after the data is corrected.

## 2.11 Temporal and as-of queries

For system-versioned temporal tables, an as-of query can reconstruct what the database knew at a point in time:

```sql
SELECT OrderId, Status, TotalAmount
FROM dbo.Orders
FOR SYSTEM_TIME AS OF @AsOfUtc
WHERE OrderId = @OrderId;
```

Explain the difference between **valid time** (when a fact is true in the business world) and **transaction time** (when the database recorded it). Audits often need both.

---

# PART 3 - QUERY DESIGN AND PERFORMANCE

## 3.1 SARGability

A predicate is SARGable when the optimizer can use an index search rather than applying a function to every row.

```sql
-- Usually non-SARGable
WHERE CONVERT(date, OrderDate) = @OrderDate;

-- SARGable half-open range
WHERE OrderDate >= @OrderDate
  AND OrderDate < DATEADD(day, 1, @OrderDate);
```

Other common problems:

- `WHERE ISNULL(Status, 'Open') = @Status`.
- Leading wildcard: `LIKE '%abc'`.
- Implicit conversion caused by mismatched data types.
- Arithmetic or functions applied to the indexed column.
- Optional-parameter patterns that force one plan to serve incompatible selectivities.

## 3.2 Join strategy

Know the conceptual difference:

- **Nested loops:** good when the outer input is small and the inner side is indexed.
- **Hash join:** good for large unsorted inputs; requires memory and may spill.
- **Merge join:** efficient for sorted inputs; benefits from compatible indexes.

Do not force a join type merely because it appears in an execution plan. Identify why the optimizer selected it and whether estimates, indexes or statistics are wrong.

## 3.3 Execution-plan diagnosis

Read plans from the root operator backward to the data access, then compare **estimated rows** with **actual rows**.

High-value clues:

- Large estimate/actual mismatch: stale statistics, skew, correlation or parameter sensitivity.
- Key lookup executed thousands or millions of times: consider a covering index or a different access path.
- Sort or hash spill: insufficient memory grant, poor estimate or avoidable wide rows.
- Implicit conversion: data type mismatch may prevent an index seek.
- Eager spool: repeated work or Halloween protection; investigate before removing it.
- High logical reads: the query is touching too many pages even if elapsed time is currently acceptable.

Measure with `SET STATISTICS IO, TIME ON`, Query Store, actual execution plans and wait statistics. Client duration alone is not enough.

## 3.4 Index design

Design an index from a workload, not from a table diagram.

For a common query:

```sql
SELECT OrderId, OrderDate, TotalAmount
FROM dbo.Orders
WHERE CustomerId = @CustomerId
  AND Status = 'Completed'
ORDER BY OrderDate DESC;
```

A possible index is:

```sql
CREATE INDEX IX_Orders_Customer_Status_OrderDate
ON dbo.Orders(CustomerId, Status, OrderDate DESC)
INCLUDE (TotalAmount);
```

Trade-offs to explain:

- Key order affects seekability and ordering.
- Included columns reduce lookups but increase storage and write cost.
- Every index slows inserts, updates and deletes that touch its columns.
- A filtered index can be excellent for a stable selective predicate, but it must match the query and workload.
- Too many overlapping indexes increase maintenance and confuse tuning.

## 3.5 Parameter sniffing

SQL Server may compile a plan using the first parameter values it sees. That plan can be excellent for a large customer and poor for a small customer, or vice versa.

Investigation and remedies depend on evidence:

- Query Store plan history and runtime statistics.
- Update statistics or improve the data distribution model.
- `OPTION (RECOMPILE)` for infrequent, highly variable queries.
- `OPTIMIZE FOR` when a stable representative value is justified.
- Query Store hints or Parameter Sensitive Plan optimization where supported.
- Dynamic SQL with `sp_executesql` when optional predicates need separate shapes.

Do not use `RECOMPILE` everywhere; it trades plan reuse for compilation CPU and can hide a modeling or indexing problem.

## 3.6 Temp tables versus table variables

| Choice | Prefer when | Main trade-off |
|---|---|---|
| `#temp` table | Large intermediate result, multiple statements, indexes/statistics needed | TempDB usage and creation overhead |
| Table variable | Small, short-lived set with simple logic | Historically weaker cardinality estimates; fewer optimizer options |
| CTE | Readability and one logical statement | Not automatically materialized or cached |
| Permanent staging table | Reusable, auditable or very large staging process | Lifecycle and cleanup responsibility |

A CTE is a query expression, not a performance feature by itself. If referenced repeatedly, it may be re-evaluated.

## 3.7 Pagination

`OFFSET/FETCH` is convenient but can become expensive for deep pages because the engine still processes and discards earlier rows.

For high-volume feeds, prefer keyset pagination:

```sql
SELECT TOP (@PageSize)
       OrderId, OrderDate, TotalAmount
FROM dbo.Orders
WHERE (OrderDate < @LastOrderDate)
   OR (OrderDate = @LastOrderDate AND OrderId < @LastOrderId)
ORDER BY OrderDate DESC, OrderId DESC;
```

The ordering must be deterministic and backed by a compatible index.

## 3.8 Batch processing

For large maintenance jobs, use bounded batches and a stable key range:

```sql
WHILE 1 = 1
BEGIN
    ;WITH Batch AS
    (
        SELECT TOP (5000) OrderId
        FROM dbo.Orders
        WHERE Status = 'Archived'
        ORDER BY OrderId
    )
    DELETE o
    FROM dbo.Orders AS o
    JOIN Batch AS b ON b.OrderId = o.OrderId;

    IF @@ROWCOUNT = 0 BREAK;
END;
```

Add monitoring, retry behavior and a cancellation mechanism. A batch is not automatically safe if it still holds locks for too long or runs inside one outer transaction.

## 3.9 SQL Server high availability, recovery and operational boundaries

Senior interviews often move from query tuning to data safety. Distinguish these concerns:

| Concern | Typical mechanisms | Interview point |
|---|---|---|
| High availability | Availability Groups, failover cluster instances, platform-managed replicas | Keeps service available; it is not a backup |
| Disaster recovery | Full, differential and log backups; restore to another region/site | Prove the restore process against the required RPO/RTO |
| Read scaling | Readable secondary, replica or separate read model | Define replica-lag and read-after-write behavior |
| Data retention | Partition switching, archival tables/storage, controlled purge | Deletion must respect legal hold and audit needs |
| Deployment safety | Expand-contract schema changes, compatibility window, rollback plan | Old and new application versions may run concurrently |

### Backup and restore example

```sql
BACKUP DATABASE Claims
TO DISK = 'D:\\sqlbackups\\Claims_full.bak'
WITH COMPRESSION, CHECKSUM, INIT;

BACKUP LOG Claims
TO DISK = 'D:\\sqlbackups\\Claims_2026-11-13_1200.trn'
WITH CHECKSUM;
```

The commands are only examples; production paths, encryption, retention, permissions and backup tooling are environment-specific. A backup that has never been restored is not evidence of recoverability. Test point-in-time restore and measure the elapsed recovery time.

### Common recovery questions

- What is the recovery point objective (maximum acceptable data loss)?
- What is the recovery time objective (maximum acceptable outage)?
- Are log backups succeeding and is the log reuse wait understood?
- Can the team restore a single database without breaking dependent services?
- How are connection strings, DNS, secrets and application failover coordinated?
- What is the consistency behavior during failover and replica lag?

---

# PART 4 - TRANSACTIONS AND CONCURRENCY

## 4.1 ACID in practical terms

- **Atomicity:** all required changes commit or none do.
- **Consistency:** constraints and business invariants remain valid.
- **Isolation:** concurrent transactions do not observe invalid intermediate state according to the chosen isolation level.
- **Durability:** committed data survives a crash according to the recovery design.

At senior level, connect ACID to the business invariant. For example: “A payment approval and ledger entry must commit together, but sending an email should happen through an outbox after commit.”

## 4.2 Isolation levels

| Level | Prevents | Cost / risk |
|---|---|---|
| Read Uncommitted | Almost nothing | Dirty, non-repeatable and phantom reads |
| Read Committed | Dirty reads | Blocking unless row versioning is used |
| Read Committed Snapshot | Dirty reads with versions | Version-store pressure in TempDB |
| Repeatable Read | Changes to rows already read | More locks and blocking |
| Snapshot | Consistent transaction view | Version-store and update-conflict considerations |
| Serializable | Phantoms for the range | Highest blocking/deadlock risk |

Do not use `NOLOCK` as a general performance fix. It can return dirty, missing, duplicated or internally inconsistent results.

## 4.3 Deadlocks

A deadlock is a cycle of waiting transactions, not simply “a slow query.” Capture the deadlock graph and identify:

- Resources and indexes involved.
- Lock order in each transaction.
- Whether transactions are larger than necessary.
- Missing indexes causing scans and broad locks.
- Inconsistent access order across code paths.

Fixes include consistent lock ordering, smaller transaction scope, appropriate indexes, reducing user interaction inside transactions and bounded retry with jitter for safe operations. The database will choose a victim; the application must handle the error when retry is safe.

## 4.4 Optimistic concurrency

Use a version column (`rowversion` in SQL Server) or an explicit version number to prevent lost updates:

```sql
UPDATE dbo.Customer
SET    Name = @Name,
       UpdatedAt = SYSUTCDATETIME()
WHERE  CustomerId = @CustomerId
AND    RowVersion = @OriginalRowVersion;

IF @@ROWCOUNT = 0
    THROW 50001, 'The record was changed by another user.', 1;
```

Returning zero rows is a domain conflict, not automatically a server error.

## 4.5 Idempotency and retries

Retries are common in distributed systems. For a payment or order command:

- Accept an idempotency key.
- Store the key with the resulting business outcome under a unique constraint.
- Perform the state change and idempotency record in one transaction.
- Return the original outcome for a repeated request.
- Retry only transient errors, not validation failures or authorization failures.

---

# PART 5 - DATA ARCHITECTURE

## 5.1 Normalization and denormalization

Normalize to protect integrity and reduce update anomalies. Denormalize only for a measured read requirement, and define the synchronization owner.

A senior answer should mention alternatives: indexed views, summary tables, cache, read replicas, columnstore, materialized aggregates or a separate analytical store.

## 5.2 Partitioning

Partitioning can help with manageability, sliding-window retention and partition elimination. It does not automatically make every query faster.

Choose a partition key aligned with common access and retention patterns, commonly a date. Verify that predicates are compatible with the partitioning expression and that indexes, statistics and maintenance are partition-aware.

## 5.3 Slowly changing dimensions

- **Type 1:** overwrite; no history.
- **Type 2:** add a new version with effective dates and current flag.
- **Type 3:** preserve selected previous values.

For Type 2, enforce non-overlapping validity periods and define how late-arriving facts are handled.

## 5.4 Multi-tenancy

Every tenant-scoped query must enforce tenant isolation in the database access path, not merely in an API controller.

Options include:

- Shared tables with `TenantId` and row-level security.
- Schema per tenant.
- Database per tenant.

Compare isolation, operational cost, noisy-neighbor risk, backup/restore requirements and tenant-specific scaling.

## 5.5 Security and governance

Use least-privilege database principals, separate read/write permissions where practical, parameterized SQL, auditing for sensitive data, encryption in transit and at rest, and controlled access to production data.

Never treat obfuscation or application-only filtering as tenant isolation. The database should provide a second enforcement boundary for high-risk data.

---

# PART 6 - PRODUCTION TROUBLESHOOTING

## 6.1 A disciplined incident workflow

1. Define the symptom: latency, error rate, CPU, blocking, deadlocks or incorrect results.
2. Establish when it started and what changed: deployment, statistics, data volume, index, compatibility level or workload.
3. Identify the affected query, plan, wait type and resource.
4. Compare a good baseline with the failing period.
5. Apply the smallest safe mitigation first.
6. Validate with measurements, then implement the durable fix.
7. Document the root cause, detection gap and prevention action.

## 6.2 Useful SQL Server evidence

- Query Store: plan history, duration, CPU, logical reads and regressions.
- Actual execution plans: row estimates, spills, lookups and scans.
- DMVs: active requests, waits, cached plans and index usage.
- Extended Events: deadlocks, statement completion, errors and spills.
- SQL Agent / monitoring: job failures, backups, log growth and storage.
- Application telemetry: correlation id, endpoint, parameter shape and timeout.

Use DMVs as current-state evidence, not as permanent history. Query Store and external telemetry preserve the trend.

## 6.3 Common production scenarios

### Query suddenly became slow

Check plan regression, parameter sensitivity, changed statistics, data skew, blocking and resource pressure before adding an index. A query can be slow because it is waiting, not because it is consuming CPU.

### CPU is high but one query is not obvious

Group Query Store data by query hash, inspect parallelism and compilation, correlate with wait statistics and check whether a deployment created many ad hoc plans.

### Transaction log is growing

Check long-running transactions, log reuse wait, backup chain, replication/CDC, large deletes and open sessions. Do not repeatedly shrink the log as a routine fix.

### TempDB is under pressure

Investigate version-store consumers, spills, large sorts/hashes, temporary objects and allocation contention. Fix the workload and configure TempDB appropriately; simply adding files is not a substitute for diagnosis.

---

# PART 7 - SENIOR INTERVIEW QUESTIONS

## 7.1 Complex SQL questions

### Q1. Find the second-highest salary without using `TOP 1` twice.

Discuss ties. If “second distinct salary” is required:

```sql
SELECT Salary
FROM
(
    SELECT Salary,
           DENSE_RANK() OVER (ORDER BY Salary DESC) AS SalaryRank
    FROM dbo.Employee
) AS x
WHERE SalaryRank = 2;
```

### Q2. Difference between `WHERE` and `HAVING`?

`WHERE` filters rows before grouping; `HAVING` filters groups after aggregation. Mention that pushing safe predicates into `WHERE` can reduce rows earlier, but do not change semantics.

### Q3. `EXISTS` versus `IN` versus a join?

Use `EXISTS` when testing existence and avoid accidental row multiplication. `IN` can be equivalent for non-null sets, but null semantics matter. A join returns combined rows and may duplicate the outer row. Validate with the actual plan rather than relying on folklore.

### Q4. Why can a `LEFT JOIN` behave like an inner join?

Because a predicate on the nullable right-side columns in `WHERE` removes the null-extended rows. Move the predicate into the `ON` clause when unmatched left rows must remain.

### Q5. How do you find duplicate records?

Use `GROUP BY ... HAVING COUNT(*) > 1` to identify duplicate keys, then `ROW_NUMBER` to select a deterministic survivor. Explain cleanup, foreign-key impact, concurrency and prevention with a constraint.

### Q6. How would you return the latest record per customer?

Use `ROW_NUMBER` partitioned by customer and ordered by timestamp plus a unique tie-breaker. Explain why `MAX(timestamp)` joined back can return multiple rows on ties.

### Q7. How do you delete duplicate rows safely?

First write a `SELECT` that proves the survivor set. Copy or audit affected rows, delete in a transaction or controlled batches, validate foreign keys, and add a unique constraint afterward. Never run an unreviewed `DELETE` based only on a visual result.

### Q8. How would you design a query for a dashboard that times out?

Define the dashboard freshness requirement, capture the query and plan, reduce the data set early, verify joins and grain, add workload-specific indexes or aggregates, consider columnstore/partitioning, and separate analytical workload from OLTP if needed.

## 7.2 Performance questions

### Q9. An index exists, but SQL Server scans. Why?

Possible causes include low selectivity, stale statistics, implicit conversion, a function on the column, non-SARGable predicate, incompatible collation, residual predicates, large lookup cost, parameter sensitivity or a cheaper scan for the estimated workload.

### Q10. What is parameter sniffing and how do you fix it?

Explain compilation using initial parameter values, prove it with plan/runtime variation, then choose a targeted remedy: statistics, query rewrite, recompile, optimize-for, Query Store hint or parameter-sensitive optimization. Mention trade-offs.

### Q11. What is the difference between a seek and a scan?

A seek navigates to qualifying portions of an index; a scan reads much or all of an access structure. A scan is not inherently bad: scanning a small table or a large fraction of a table may be optimal.

### Q12. What does a high estimated cost mean?

It is an optimizer estimate used to compare plans, not elapsed milliseconds and not proof of actual resource consumption. Compare actual rows, logical reads, CPU, duration and waits.

### Q13. How do you tune a query without changing its result?

Freeze a representative test case, capture baseline results and metrics, compare execution plans and results after each change, test nulls/duplicates/edge dates, and deploy through a reversible change. Performance is invalid if semantics change.

## 7.3 Architecture questions

### Q14. Design a high-volume order system.

Cover transaction boundaries, keys, constraints, status transitions, idempotency, indexing, partitioning/retention, read models, outbox events, retries, audit history, security, backup/restore and operational ownership.

### Q15. When would you use a read replica?

For read scaling or workload isolation when replica lag is acceptable. Define which reads require primary consistency, how routing works, how lag is monitored and what happens during failover. A replica does not solve inefficient queries.

### Q16. When should data move out of SQL Server?

When a workload has different scale or access characteristics: search, analytics, object storage, time series or caching. Preserve a clear source of truth, delivery guarantees, reconciliation and data ownership.

### Q17. How do you migrate a 2 TB table with minimal downtime?

Use an online or expand-contract approach: add compatible structures, backfill in batches, capture changes, validate counts/checksums, switch reads/writes behind a controlled boundary, then retire the old structure. Discuss log, lock, rollback and observability risks.

### Q18. How do you prevent cross-tenant data leakage?

Enforce tenant context at the database access layer, use row-level security or equivalent defense in depth, require tenant predicates and indexes, test negative cases, audit access, and make cross-tenant support access explicit and time-bound.

## 7.4 Leadership and ownership questions

### Q19. Tell me about a production database incident.

Use this structure: business impact, detection, immediate containment, evidence, root cause, durable fix, prevention and what you personally changed. Avoid blaming an individual; demonstrate learning and control improvement.

### Q20. How do you review SQL from another engineer?

Check correctness and grain first, then security, data access, indexes, plan risk, transaction scope, error handling, maintainability, test coverage and operational behavior. Explain the reason for each comment and offer a simpler alternative.

### Q21. How do you handle disagreement with an architect or product owner?

Frame alternatives with measurable consequences: latency, correctness, delivery time, cost, operational risk and reversibility. Run a focused experiment where useful, document the decision and commit once the decision is made.

### Q22. How do you improve SQL quality across a team?

Introduce standards, query review checklists, migration discipline, Query Store baselines, automated tests, performance budgets, observability, mentoring and incident learning. Senior impact is measured by the team and system, not only by individual query output.

## 7.5 Coding exercises to practise

1. Top three products by revenue per region, including ties.
2. Customers with purchases in three consecutive months.
3. Detect overlapping validity periods for the same contract.
4. Find the longest login streak per user.
5. Return a bill of materials tree with depth and path.
6. Reconcile two transaction sources and classify missing, changed and duplicate rows.
7. Allocate payments to invoices using a deterministic oldest-first rule.
8. Implement keyset pagination with a stable cursor.
9. Identify sessions causing blocking and the blocked query.
10. Design an idempotent batch job that can resume after failure.

## 7.6 Interview answer framework

For any SQL design or tuning question, answer in this order:

```text
Clarify the requirement and result grain
  -> State assumptions and edge cases
  -> Write the simplest correct query
  -> Explain indexes and expected access path
  -> Discuss scale, concurrency and failure modes
  -> Describe measurement and validation
  -> Explain alternatives and trade-offs
```

This structure demonstrates senior judgment even when the interviewer changes the schema or adds a constraint.

---

# PART 8 - FINAL REVISION CHECKLIST

## 8.1 Query checklist

- [ ] Is the result grain explicit?
- [ ] Are nulls, duplicates, ties and timezones handled?
- [ ] Are joins guaranteed not to multiply rows unexpectedly?
- [ ] Is ordering deterministic?
- [ ] Are predicates SARGable and data types aligned?
- [ ] Is the query parameterized?
- [ ] Does the plan match expected row counts?
- [ ] Are indexes justified by the workload?
- [ ] Is pagination scalable?
- [ ] Are large writes batched and observable?

## 8.2 Production checklist

- [ ] Query Store and baseline metrics are available.
- [ ] Deadlock and blocking diagnostics are configured.
- [ ] Backup, restore and disaster-recovery objectives are tested.
- [ ] Long-running transactions and log growth are monitored.
- [ ] Tenant and sensitive-data access is audited.
- [ ] Schema changes are backward compatible and reversible.
- [ ] Retry behavior is bounded and idempotent.
- [ ] Permissions follow least privilege.
- [ ] Data quality and reconciliation checks exist.
- [ ] Operational ownership is clear.

## 8.3 Questions to ask the interviewer

- What SQL Server version, compatibility level and deployment model are used?
- What are the largest tables and the most demanding query patterns?
- Is the workload primarily OLTP, reporting or mixed?
- How are query regressions and blocking detected?
- What are the RPO/RTO and availability expectations?
- How are schema changes, indexes and data migrations reviewed?
- What authority does this role have to improve standards and architecture?
- What would success look like after six months?

---

## Closing advice

At 14+ years, syntax is table stakes. Make your answers show that you protect correctness, understand the optimizer, design for concurrency, operate databases safely and help teams make better decisions. The strongest senior response usually includes a query, an execution-plan hypothesis, a measurement plan, a failure mode and a clear trade-off.
