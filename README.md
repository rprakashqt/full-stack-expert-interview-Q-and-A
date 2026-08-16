# full-stack-expert-interview-Q-and-A
# SQL Server 2022 Programming Interview Questions and Answers

An expert-level, scenario-based guide for senior developer, database developer, technical lead, and architect interviews. The questions focus on the SQL Server topics commonly tested in large product and consulting companies: query tuning, indexing, execution plans, concurrency, transactions, T-SQL problem solving, security, and SQL Server 2022 features.

> **Interview note:** Do not memorize only the query. Explain the trade-off, expected execution plan, concurrency behavior, and how you would prove the improvement with production-like data.

## Contents

1. [Practice schema](#practice-schema)
2. [Query design and performance](#query-design-and-performance)
3. [Advanced T-SQL problems](#advanced-t-sql-problems)
4. [Transactions and concurrency](#transactions-and-concurrency)
5. [Programmability and data features](#programmability-and-data-features)
6. [SQL Server 2022 topics](#sql-server-2022-topics)
7. [Security, troubleshooting, and design](#security-troubleshooting-and-design)
8. [Legacy SQL Server to Azure SQL migration scenarios](#legacy-sql-server-to-azure-sql-migration-scenarios)
9. [Rapid-fire questions](#rapid-fire-questions)
10. [Interview evaluation checklist](#interview-evaluation-checklist)
11. [Official references](#official-references)

## Practice schema

The answers use the following simplified sales schema. Run it in a disposable SQL Server 2022 database if you want to test the examples.

```sql
ALTER DATABASE CURRENT SET COMPATIBILITY_LEVEL = 160;
GO

CREATE TABLE dbo.Customers
(
    CustomerId int IDENTITY(1,1) NOT NULL
        CONSTRAINT PK_Customers PRIMARY KEY,
    Email varchar(320) NOT NULL,
    RegionCode char(2) NOT NULL,
    CreatedAt datetime2(3) NOT NULL
        CONSTRAINT DF_Customers_CreatedAt DEFAULT SYSUTCDATETIME(),
    IsActive bit NOT NULL
        CONSTRAINT DF_Customers_IsActive DEFAULT 1,
    Version rowversion NOT NULL,
    CONSTRAINT UQ_Customers_Email UNIQUE (Email)
);

CREATE TABLE dbo.Orders
(
    OrderId bigint IDENTITY(1,1) NOT NULL
        CONSTRAINT PK_Orders PRIMARY KEY,
    CustomerId int NOT NULL,
    OrderDate datetime2(3) NOT NULL,
    Status varchar(20) NOT NULL,
    TotalAmount decimal(19,4) NOT NULL,
    Metadata nvarchar(max) NULL,
    CONSTRAINT FK_Orders_Customers
        FOREIGN KEY (CustomerId) REFERENCES dbo.Customers(CustomerId),
    CONSTRAINT CK_Orders_TotalAmount CHECK (TotalAmount >= 0),
    CONSTRAINT CK_Orders_Metadata_IsJson
        CHECK (Metadata IS NULL OR ISJSON(Metadata) = 1)
);

CREATE TABLE dbo.OrderItems
(
    OrderId bigint NOT NULL,
    LineNo int NOT NULL,
    ProductId int NOT NULL,
    Quantity int NOT NULL,
    UnitPrice decimal(19,4) NOT NULL,
    CONSTRAINT PK_OrderItems PRIMARY KEY (OrderId, LineNo),
    CONSTRAINT FK_OrderItems_Orders
        FOREIGN KEY (OrderId) REFERENCES dbo.Orders(OrderId),
    CONSTRAINT CK_OrderItems_Quantity CHECK (Quantity > 0),
    CONSTRAINT CK_OrderItems_UnitPrice CHECK (UnitPrice >= 0)
);

CREATE INDEX IX_Orders_CustomerId_OrderDate
ON dbo.Orders (CustomerId, OrderDate DESC)
INCLUDE (Status, TotalAmount);
GO
```

## Query design and performance

### 1. What makes a predicate SARGable, and why does it matter?

**Answer:** A SARGable predicate lets SQL Server use an index seek to locate a range of keys. Applying a function or implicit conversion to an indexed column often forces SQL Server to evaluate rows after scanning a larger part of the index.

```sql
-- Non-SARGable: transforms every OrderDate value.
SELECT OrderId, CustomerId, TotalAmount
FROM dbo.Orders
WHERE CAST(OrderDate AS date) = '2026-08-17';

-- SARGable: a half-open range handles every time precision safely.
SELECT OrderId, CustomerId, TotalAmount
FROM dbo.Orders
WHERE OrderDate >= '2026-08-17'
  AND OrderDate <  '2026-08-18';
```

Also watch for mismatched parameter types. Comparing an `nvarchar` parameter with an indexed `varchar` column can introduce a conversion and change the access method.

### 2. Explain clustered, nonclustered, covering, and filtered indexes.

**Answer:**

- A **clustered index** defines the table's leaf-level row order. A table can have one.
- A **nonclustered index** has its own key order and points to the base row. A table can have many.
- A **covering index** contains every column needed by a query as key or included columns, avoiding key lookups.
- A **filtered index** stores only rows matching a stable predicate, making it smaller and cheaper for selective workloads.

```sql
CREATE INDEX IX_Orders_Open_Customer_Date
ON dbo.Orders (CustomerId, OrderDate DESC)
INCLUDE (OrderId, TotalAmount)
WHERE Status = 'OPEN';
```

Do not blindly include every selected column. Wider indexes increase storage, memory use, transaction-log volume, and write cost.

### 3. How do you choose the key order of a composite index?

**Answer:** Start from actual query patterns, not a blanket “most selective column first” rule. Equality predicates usually come first, followed by range and ordering columns. Key order must also support joins and `ORDER BY` requirements.

For this query, `(CustomerId, OrderDate DESC)` is useful because `CustomerId` is an equality predicate and `OrderDate` supplies both the range and output order:

```sql
SELECT TOP (20) OrderId, OrderDate, TotalAmount
FROM dbo.Orders
WHERE CustomerId = @CustomerId
  AND OrderDate >= @FromDate
ORDER BY OrderDate DESC;
```

An index on `(OrderDate, CustomerId)` cannot efficiently seek to one customer's rows across a broad date range. Validate the choice with representative data and the actual execution plan.

### 4. An index exists, but SQL Server scans instead of seeking. Why?

**Answer:** A scan may be correct. Common reasons are:

- The predicate returns a large percentage of the table.
- The predicate is non-SARGable or contains an implicit conversion.
- Statistics are stale or do not describe correlated/skewed data well.
- The useful column is not the leading index key.
- Repeated key lookups cost more than a scan.
- A cached plan was compiled for a very different parameter value.

Inspect **actual rows versus estimated rows**, warnings, residual predicates, lookups, spills, memory grants, and wait statistics. Operator cost percentages are optimizer estimates, not measured elapsed time.

```sql
SET STATISTICS IO, TIME ON;

SELECT OrderId, CustomerId, OrderDate, TotalAmount
FROM dbo.Orders
WHERE CustomerId = @CustomerId;

SET STATISTICS IO, TIME OFF;
```

### 5. What is the difference between estimated and actual execution plans?

**Answer:** An estimated plan is compiled without executing the query. An actual plan includes runtime metrics such as actual row counts and spill information, but obtaining it executes the statement and adds profiling overhead. Large estimated-versus-actual row differences often point to statistics, parameter sensitivity, non-SARGable expressions, or cardinality-estimation assumptions.

For a production incident, correlate the plan with Query Store, duration, CPU, logical reads, memory grant, blocking, and waits. Never tune from the graphical plan alone.

### 6. How do statistics affect plan quality?

**Answer:** Statistics contain distribution information used to estimate row counts. Poor estimates can cause the wrong join type, access path, memory grant, or degree of parallelism.

```sql
SELECT
    s.name,
    STATS_DATE(s.object_id, s.stats_id) AS LastUpdated,
    sp.rows,
    sp.rows_sampled,
    sp.modification_counter
FROM sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) AS sp
WHERE s.object_id = OBJECT_ID(N'dbo.Orders');

UPDATE STATISTICS dbo.Orders IX_Orders_CustomerId_OrderDate
WITH FULLSCAN;
```

`FULLSCAN` is not a universal maintenance policy; it can be expensive. Prefer automatic statistics plus targeted intervention based on evidence.

### 7. What is parameter sniffing? How do you solve it?

**Answer:** On compilation, SQL Server can use the current parameter values to estimate cardinality and cache a plan. That is normally beneficial. It becomes a problem when skew means that no single plan performs well for all values.

Possible responses, after proving the problem, include:

- Let SQL Server 2022 Parameter Sensitive Plan optimization maintain multiple variants.
- Improve indexes or statistics.
- Split materially different workloads into different query paths.
- Use Query Store to force a known stable plan or apply a hint without changing code.
- Use `OPTION (RECOMPILE)` only when compile cost and CPU are acceptable.
- Use `OPTIMIZE FOR` only when a representative value is stable.

```sql
CREATE OR ALTER PROCEDURE dbo.GetOrdersByCustomer
    @CustomerId int
AS
BEGIN
    SET NOCOUNT ON;

    SELECT OrderId, OrderDate, Status, TotalAmount
    FROM dbo.Orders
    WHERE CustomerId = @CustomerId;
END;
GO
```

Local-variable tricks and disabling parameter sniffing merely replace a sniffed estimate with a generic estimate; they are not automatic fixes.

### 8. CTE, temporary table, table variable, or derived table—which should you use?

**Answer:** A nonrecursive CTE and a derived table primarily improve query expression; they are generally not materialized just because they are named. A temporary table materializes an intermediate result, can have indexes and statistics, and creates a phase boundary for compilation. A table variable has a narrower scope and fewer management features; deferred compilation improves estimates, but it is still not a universal replacement for a temporary table.

```sql
SELECT CustomerId, SUM(TotalAmount) AS Revenue
INTO #CustomerRevenue
FROM dbo.Orders
WHERE OrderDate >= @FromDate
GROUP BY CustomerId;

CREATE UNIQUE CLUSTERED INDEX CX_CustomerRevenue
    ON #CustomerRevenue(CustomerId);

SELECT c.Email, r.Revenue
FROM #CustomerRevenue AS r
JOIN dbo.Customers AS c ON c.CustomerId = r.CustomerId
WHERE r.Revenue >= @MinimumRevenue;
```

Use a temporary table when reused data, indexing, or better phase-specific cardinality justifies tempdb cost.

### 9. Why is keyset pagination usually better than a large `OFFSET`?

**Answer:** `OFFSET` still makes SQL Server walk or sort past earlier rows, so deep pages become progressively expensive. Keyset pagination seeks after the last stable key from the previous page.

```sql
CREATE INDEX IX_Orders_OrderDate_OrderId
ON dbo.Orders (OrderDate DESC, OrderId DESC)
INCLUDE (CustomerId, Status, TotalAmount);

SELECT TOP (@PageSize)
    OrderId, CustomerId, OrderDate, Status, TotalAmount
FROM dbo.Orders
WHERE OrderDate < @LastOrderDate
   OR (OrderDate = @LastOrderDate AND OrderId < @LastOrderId)
ORDER BY OrderDate DESC, OrderId DESC;
```

Keyset pagination does not naturally support jumping directly to arbitrary page numbers; that is the trade-off.

### 10. When should you use `EXISTS` instead of `JOIN` or `IN`?

**Answer:** Use `EXISTS` for a semi-join when the requirement is only to test whether a related row exists. A normal join can multiply parent rows. `NOT EXISTS` also has safe null semantics, unlike `NOT IN` when the subquery can return `NULL`.

```sql
SELECT c.CustomerId, c.Email
FROM dbo.Customers AS c
WHERE EXISTS
(
    SELECT 1
    FROM dbo.Orders AS o
    WHERE o.CustomerId = c.CustomerId
      AND o.Status = 'OPEN'
);
```

### 11. Why can `SELECT *` hurt performance and maintainability?

**Answer:** It increases I/O and network payload, can prevent narrow covering indexes, increases memory grants, exposes newly added columns unexpectedly, and makes contracts fragile. It is especially damaging with large object columns. Select only required columns at application boundaries.

### 12. How do you tune a slow query systematically?

**Answer:** Use a measurement-driven sequence:

1. Capture the exact query, parameters, runtime, frequency, resource use, blocking, and regression time.
2. Use Query Store to compare plans and runtime history.
3. Reproduce with representative data and session settings.
4. Inspect the actual plan and `STATISTICS IO, TIME`.
5. Fix correctness and SARGability before adding indexes or hints.
6. Check estimates, statistics, indexes, spills, implicit conversions, lookups, parallelism, and parameter sensitivity.
7. Change one thing, retest under concurrency, and assess write/storage impact.
8. Keep a rollback path and monitor after release.

## Advanced T-SQL problems

### 13. Return the three highest-value orders per customer.

**Answer:** Rank within each customer and include a deterministic tie-breaker.

```sql
WITH RankedOrders AS
(
    SELECT
        o.OrderId,
        o.CustomerId,
        o.TotalAmount,
        o.OrderDate,
        ROW_NUMBER() OVER
        (
            PARTITION BY o.CustomerId
            ORDER BY o.TotalAmount DESC, o.OrderId DESC
        ) AS rn
    FROM dbo.Orders AS o
)
SELECT OrderId, CustomerId, TotalAmount, OrderDate
FROM RankedOrders
WHERE rn <= 3;
```

Use `DENSE_RANK` instead if all tied values must be returned, accepting that more than three rows may result.

### 14. Calculate a running total and a seven-order moving average.

```sql
SELECT
    CustomerId,
    OrderId,
    OrderDate,
    TotalAmount,
    SUM(TotalAmount) OVER
    (
        PARTITION BY CustomerId
        ORDER BY OrderDate, OrderId
        ROWS UNBOUNDED PRECEDING
    ) AS RunningTotal,
    AVG(TotalAmount) OVER
    (
        PARTITION BY CustomerId
        ORDER BY OrderDate, OrderId
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS MovingAverageOfLast7Orders
FROM dbo.Orders;
```

**Expert detail:** Specify `ROWS` explicitly. The default frame for an ordered aggregate can have peer-group behavior that surprises candidates when sort values are duplicated.

### 15. Solve a gaps-and-islands problem for consecutive login dates.

Assume `dbo.UserLogins(UserId int, LoginDate date)` can contain multiple logins per day.

```sql
WITH DistinctDays AS
(
    SELECT DISTINCT UserId, LoginDate
    FROM dbo.UserLogins
), GroupedDays AS
(
    SELECT
        UserId,
        LoginDate,
        DATEADD
        (
            day,
            -CONVERT(int, ROW_NUMBER() OVER
                (PARTITION BY UserId ORDER BY LoginDate)),
            LoginDate
        ) AS IslandKey
    FROM DistinctDays
)
SELECT
    UserId,
    MIN(LoginDate) AS StartDate,
    MAX(LoginDate) AS EndDate,
    COUNT(*) AS ConsecutiveDays
FROM GroupedDays
GROUP BY UserId, IslandKey
ORDER BY UserId, StartDate;
```

Consecutive dates minus consecutive row numbers produce a constant key for each island.

### 16. Delete duplicate rows while retaining the newest one.

Assume duplicates are defined by `CustomerId`, `OrderDate`, and `TotalAmount`.

```sql
WITH Duplicates AS
(
    SELECT
        OrderId,
        ROW_NUMBER() OVER
        (
            PARTITION BY CustomerId, OrderDate, TotalAmount
            ORDER BY OrderId DESC
        ) AS rn
    FROM dbo.Orders
)
DELETE FROM Duplicates
WHERE rn > 1;
```

In production, first run the CTE as a `SELECT`, take a recovery-safe backup or archive, process large sets in batches, and add a unique constraint or index that prevents recurrence.

### 17. Find the median order amount.

```sql
SELECT DISTINCT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY TotalAmount)
        OVER () AS MedianOrderAmount
FROM dbo.Orders;
```

`PERCENTILE_CONT` can interpolate and therefore may return a value not present in the table. Discuss whether the business instead wants a discrete percentile.

### 18. What is the difference between `CROSS APPLY` and `OUTER APPLY`?

**Answer:** `APPLY` invokes a correlated table expression for each left row. `CROSS APPLY` removes left rows with no right-side result; `OUTER APPLY` retains them with null right-side columns.

```sql
SELECT
    c.CustomerId,
    c.Email,
    lastOrder.OrderId,
    lastOrder.OrderDate
FROM dbo.Customers AS c
OUTER APPLY
(
    SELECT TOP (1) o.OrderId, o.OrderDate
    FROM dbo.Orders AS o
    WHERE o.CustomerId = c.CustomerId
    ORDER BY o.OrderDate DESC, o.OrderId DESC
) AS lastOrder;
```

With the practice index, this can become an efficient seek per customer. For very large outer inputs, compare it with a window-function solution.

### 19. How do you write safe dynamic SQL?

**Answer:** Parameterize data values with `sys.sp_executesql`. Identifiers cannot be parameters, so allow-list them and wrap them with `QUOTENAME`. Never concatenate untrusted values into executable SQL.

```sql
DECLARE @SortColumn sysname = N'OrderDate';
DECLARE @sql nvarchar(max);

IF @SortColumn NOT IN (N'OrderDate', N'TotalAmount', N'OrderId')
    THROW 50001, 'Invalid sort column.', 1;

SET @sql = N'
    SELECT OrderId, CustomerId, OrderDate, TotalAmount
    FROM dbo.Orders
    WHERE CustomerId = @CustomerId
    ORDER BY ' + QUOTENAME(@SortColumn) + N' DESC;';

EXEC sys.sp_executesql
    @sql,
    N'@CustomerId int',
    @CustomerId = @CustomerId;
```

### 20. Write a recursive hierarchy query and guard against runaway recursion.

Assume `dbo.Employees(EmployeeId, ManagerId, EmployeeName)`.

```sql
WITH Organization AS
(
    SELECT EmployeeId, ManagerId, EmployeeName, 0 AS Depth
    FROM dbo.Employees
    WHERE EmployeeId = @RootEmployeeId

    UNION ALL

    SELECT e.EmployeeId, e.ManagerId, e.EmployeeName, o.Depth + 1
    FROM dbo.Employees AS e
    JOIN Organization AS o ON e.ManagerId = o.EmployeeId
    WHERE o.Depth < 100
)
SELECT EmployeeId, ManagerId, EmployeeName, Depth
FROM Organization
OPTION (MAXRECURSION 100);
```

A depth limit does not identify a cycle; robust hierarchy design also prevents or detects cycles during writes.

## Transactions and concurrency

### 21. Explain SQL Server isolation levels with anomalies.

| Isolation level | Key behavior |
|---|---|
| `READ UNCOMMITTED` | Allows dirty reads and does not guarantee a stable or complete view. |
| `READ COMMITTED` | Prevents dirty reads; values can change between statements. Locking or row-versioning behavior depends on database configuration. |
| `REPEATABLE READ` | Holds read locks to transaction end, preventing changes to rows already read; new matching rows can still appear. |
| `SNAPSHOT` | Reads a transactionally consistent version as of transaction start; update conflicts are possible. |
| `SERIALIZABLE` | Uses the strongest locking semantics, including key-range protection for qualifying predicates. |

`READ_COMMITTED_SNAPSHOT` (RCSI) is a database option, not a separate syntax-level isolation value. It changes `READ COMMITTED` to use row versions, generally reducing reader-writer blocking while increasing tempdb/version-store usage.

```sql
ALTER DATABASE InterviewDb
SET READ_COMMITTED_SNAPSHOT ON
WITH ROLLBACK IMMEDIATE;
```

This database-wide change requires workload testing; `ROLLBACK IMMEDIATE` disconnects active sessions and should not be used casually.

### 22. Why is `NOLOCK` not a general performance solution?

**Answer:** `NOLOCK` means `READ UNCOMMITTED`. It can read uncommitted data, miss rows, read rows twice, and return combinations that never existed as a committed state. It also does not eliminate every lock, because compilation and metadata access can still acquire locks.

For correct read-heavy concurrency, consider short transactions, suitable indexes, RCSI, or snapshot isolation. Choose based on correctness requirements and measured contention.

### 23. How do you make a stored procedure transaction-safe?

```sql
CREATE OR ALTER PROCEDURE dbo.CreateOrder
    @CustomerId int,
    @OrderDate datetime2(3),
    @Status varchar(20),
    @TotalAmount decimal(19,4),
    @OrderId bigint OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        INSERT dbo.Orders(CustomerId, OrderDate, Status, TotalAmount)
        VALUES (@CustomerId, @OrderDate, @Status, @TotalAmount);

        SET @OrderId = CONVERT(bigint, SCOPE_IDENTITY());

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
GO
```

`XACT_ABORT ON` rolls back most runtime-error transactions; `TRY/CATCH` allows cleanup; parameter types match their columns; and `THROW` preserves the original error details. If the procedure can participate in an outer transaction, use a transaction-ownership/savepoint pattern rather than blindly rolling back the caller's work.

### 24. How do deadlocks occur, and how do you prevent them?

**Answer:** A deadlock is a cycle in which sessions each hold a resource the other needs. SQL Server selects a victim and returns error 1205.

Prevent or reduce deadlocks by:

- Accessing tables and rows in a consistent order.
- Keeping transactions small and avoiding user/network waits inside them.
- Adding indexes that reduce the rows and lock footprint touched.
- Using an appropriate isolation model.
- Handling error 1205 with bounded retry and jitter in the application.

Capture and inspect the `xml_deadlock_report` from the `system_health` Extended Events session. Do not treat `MAXDOP 1`, `NOLOCK`, or disabling lock escalation as universal fixes.

### 25. Implement optimistic concurrency with `rowversion`.

```sql
UPDATE dbo.Customers
SET Email = @NewEmail
WHERE CustomerId = @CustomerId
  AND Version = @OriginalVersion;

IF @@ROWCOUNT = 0
    THROW 50002, 'The row was changed or deleted by another user.', 1;
```

Return the new `Version` value to the client after a successful update. `rowversion` is an incrementing binary database value; it is not a date/time and should not be presented as one.

### 26. How do you perform a concurrency-safe insert-if-missing operation?

**Answer:** First enforce the invariant with a unique constraint. Then use locking or handle a duplicate-key exception. A prior `IF NOT EXISTS` without suitable locking has a race condition.

```sql
SET XACT_ABORT ON;

BEGIN TRANSACTION;

IF NOT EXISTS
(
    SELECT 1
    FROM dbo.Customers WITH (UPDLOCK, HOLDLOCK)
    WHERE Email = @Email
)
BEGIN
    INSERT dbo.Customers(Email, RegionCode)
    VALUES (@Email, @RegionCode);
END;

COMMIT TRANSACTION;
```

The unique constraint on `Email` remains the final guarantee. Be prepared to explain locking, error handling, and why a complex `MERGE` is not automatically safer.

### 27. Design a table-backed worker queue without double-processing rows.

Assume `dbo.WorkQueue(WorkId, Status, EnqueuedAt, StartedAt, WorkerId)` and a suitable index beginning with `(Status, EnqueuedAt, WorkId)`.

```sql
BEGIN TRANSACTION;

;WITH NextItem AS
(
    SELECT TOP (1) *
    FROM dbo.WorkQueue WITH (UPDLOCK, READPAST, ROWLOCK)
    WHERE Status = 'READY'
    ORDER BY EnqueuedAt, WorkId
)
UPDATE NextItem
SET Status = 'RUNNING',
    StartedAt = SYSUTCDATETIME(),
    WorkerId = @WorkerId
OUTPUT inserted.WorkId;

COMMIT TRANSACTION;
```

`UPDLOCK` reserves the selected row, while `READPAST` lets competing workers skip locked rows. `ROWLOCK` is a preference, not a guarantee. A production queue also needs leases/timeouts, retries, poison-message handling, idempotency, monitoring, and recovery for abandoned work.

## Programmability and data features

### 28. How do you parse and validate JSON in SQL Server?

```sql
DECLARE @payload nvarchar(max) = N'
{
  "customerId": 42,
  "items": [
    {"productId": 101, "quantity": 2, "unitPrice": 19.95},
    {"productId": 205, "quantity": 1, "unitPrice": 49.00}
  ]
}';

IF ISJSON(@payload) <> 1
    THROW 50003, 'Invalid JSON.', 1;

SELECT
    JSON_VALUE(@payload, '$.customerId') AS CustomerId,
    i.ProductId,
    i.Quantity,
    i.UnitPrice
FROM OPENJSON(@payload, '$.items')
WITH
(
    ProductId int '$.productId',
    Quantity int '$.quantity',
    UnitPrice decimal(19,4) '$.unitPrice'
) AS i;
```

For frequently filtered JSON properties, expose a deterministic computed column and index it:

```sql
ALTER TABLE dbo.Orders
ADD Channel AS CONVERT(varchar(20), JSON_VALUE(Metadata, '$.channel'));

CREATE INDEX IX_Orders_Channel ON dbo.Orders(Channel);
```

### 29. What are system-versioned temporal tables used for?

**Answer:** A temporal table automatically retains previous row versions and supports time-travel queries. It is useful for audit history, point-in-time analysis, and accidental-change investigation, but retention, storage, indexing, privacy deletion, and schema changes still require design.

```sql
CREATE TABLE dbo.ProductPrices
(
    ProductId int NOT NULL CONSTRAINT PK_ProductPrices PRIMARY KEY,
    Price decimal(19,4) NOT NULL,
    ValidFrom datetime2 GENERATED ALWAYS AS ROW START NOT NULL,
    ValidTo datetime2 GENERATED ALWAYS AS ROW END NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
)
WITH
(
    SYSTEM_VERSIONING = ON
    (
        HISTORY_TABLE = dbo.ProductPricesHistory
    )
);

SELECT ProductId, Price, ValidFrom, ValidTo
FROM dbo.ProductPrices
FOR SYSTEM_TIME AS OF '2026-08-17T10:00:00';
```

### 30. Stored procedure, view, inline TVF, scalar UDF, or trigger?

**Answer:**

- Use a **stored procedure** for an explicit operation, multiple statements, transactions, controlled permissions, and one or more result sets.
- Use a **view** for a reusable relational projection with no parameters.
- Use an **inline table-valued function** for a parameterized relational expression that the optimizer can expand into the calling query.
- Use a **scalar UDF** sparingly in row-heavy queries. Inlining can help eligible functions, but verify the actual plan and compatibility behavior.
- Use a **trigger** only for rules that must execute transparently with writes. Triggers must handle multi-row operations and add hidden coupling.

### 31. Why is row-by-row processing usually slow, and what replaces it?

**Answer:** Cursors and loops repeatedly pay statement, locking, and logging overhead and limit optimizer choices. Prefer set-based joins, window functions, grouped updates, and batch operations. Cursors can still be appropriate for inherently sequential or administrative work; use the smallest-scope cursor options and measure.

```sql
UPDATE c
SET IsActive = 0
FROM dbo.Customers AS c
WHERE c.IsActive = 1
  AND NOT EXISTS
  (
      SELECT 1
      FROM dbo.Orders AS o
      WHERE o.CustomerId = c.CustomerId
        AND o.OrderDate >= DATEADD(year, -2, SYSUTCDATETIME())
  );
```

### 32. How do you delete or update millions of rows safely?

**Answer:** Use deterministic batches supported by an index, commit between batches, throttle if needed, and monitor log reuse, replication/availability lag, blocking, and lock escalation.

```sql
DECLARE @Rows int = 1;

WHILE @Rows > 0
BEGIN
    DELETE TOP (5000)
    FROM dbo.Orders
    WHERE OrderDate < @CutoffDate;

    SET @Rows = @@ROWCOUNT;
END;
```

For predictable progress, select keys in order inside a CTE and delete through it. If removing entire aligned time ranges, partition switching can be dramatically cheaper than row-by-row deletion.

### 33. When do partitioning and columnstore indexes help?

**Answer:** Partitioning is primarily a manageability feature for very large tables: sliding-window load/archive, partition-level maintenance, and elimination when predicates align with the partitioning column. It is not automatic query acceleration and does not replace good indexes.

A clustered columnstore index is strong for large analytical scans and aggregations because it uses columnar storage, compression, segment elimination, and batch-mode execution. It is usually not the first choice for singleton lookups or heavily updated narrow OLTP rows. Hybrid designs may use nonclustered rowstore indexes over columnstore or nonclustered columnstore over rowstore, depending on workload.

## SQL Server 2022 topics

### 34. What is Parameter Sensitive Plan optimization?

**Answer:** SQL Server 2022 can keep multiple active cached plans for one eligible parameterized statement when data distribution is skewed. A dispatcher expression routes runtime values to query variants representing different cardinality ranges.

Requirements and limitations to mention:

- SQL Server 2022 or later and database compatibility level 160.
- The SQL Server 2022 implementation targets eligible equality predicates.
- Statistics quality and appropriate indexes still matter.
- Query Store helps expose dispatcher and variant relationships.

```sql
ALTER DATABASE CURRENT SET COMPATIBILITY_LEVEL = 160;

SELECT OrderId, OrderDate, TotalAmount
FROM dbo.Orders
WHERE CustomerId = @CustomerId;
```

PSP is an optimization for parameter sensitivity, not permission to ignore poor query or schema design.

### 35. What are Query Store hints, and when would you use them?

**Answer:** Query Store hints apply supported query hints to a captured `query_id` without changing application SQL. They are useful for an urgent, reversible correction when application code cannot be released immediately.

```sql
EXEC sys.sp_query_store_set_hints
    @query_id = 42,
    @query_hints = N'OPTION (MAXDOP 4)';

SELECT query_id, query_hint_text, last_query_hint_failure_reason_desc
FROM sys.query_store_query_hints
WHERE query_id = 42;

EXEC sys.sp_query_store_clear_hints @query_id = 42;
```

Use hints only after root-cause analysis. Reevaluate them after data distribution, compatibility level, indexes, or application behavior changes.

### 36. Name important Intelligent Query Processing improvements in SQL Server 2022.

**Answer:** Strong answers include:

- Parameter Sensitive Plan optimization.
- Persisted and percentile memory grant feedback.
- Degree-of-parallelism feedback.
- Cardinality-estimation feedback.
- Optimized plan forcing through Query Store.

Several depend on Query Store and/or compatibility level 160. Explain the problem each feature solves rather than merely listing names. Query Store is enabled by default for newly created SQL Server 2022 databases, but upgraded databases must be checked explicitly.

### 37. Demonstrate useful SQL Server 2022 T-SQL additions.

```sql
SELECT value
FROM GENERATE_SERIES(1, 10, 1);

SELECT DATETRUNC(month, OrderDate) AS OrderMonth,
       SUM(TotalAmount) AS Revenue
FROM dbo.Orders
GROUP BY DATETRUNC(month, OrderDate);

SELECT OrderId,
       LEAST(TotalAmount, CONVERT(decimal(19,4), 1000)) AS CappedAmount,
       GREATEST(TotalAmount, CONVERT(decimal(19,4), 0)) AS NonNegativeAmount
FROM dbo.Orders;

SELECT value, ordinal
FROM STRING_SPLIT('critical,high,normal', ',', 1)
ORDER BY ordinal;
```

Always verify version and compatibility-level requirements before using newer syntax in a migration.

## Security, troubleshooting, and design

### 38. How do TDE, Always Encrypted, dynamic data masking, and row-level security differ?

| Feature | Primary purpose | Important limitation |
|---|---|---|
| Transparent Data Encryption (TDE) | Encrypt database, log, and backup files at rest. | Data is decrypted inside the engine; it does not protect against authorized queries. |
| Always Encrypted | Keep selected sensitive values encrypted from the database engine using client-held keys. | Query operations and driver/application design are constrained; enclaves expand supported operations. |
| Dynamic data masking | Obscure displayed values for nonprivileged users. | It is not encryption and is not a security boundary by itself. |
| Row-level security (RLS) | Apply a server-side predicate that controls which rows a principal can access. | Predicate design can affect performance and must avoid context/ownership mistakes. |

Use least privilege, separate application identities, parameterized SQL, auditing, protected backups/keys, and layered controls.

### 39. How do you diagnose blocking in a live system?

**Answer:** Identify the head blocker, its transaction age, statement, wait type, isolation level, and locks before deciding whether to terminate anything.

```sql
SELECT
    r.session_id,
    r.blocking_session_id,
    r.status,
    r.wait_type,
    r.wait_time,
    r.wait_resource,
    r.open_transaction_count,
    DB_NAME(r.database_id) AS DatabaseName,
    SUBSTRING
    (
        t.text,
        (r.statement_start_offset / 2) + 1,
        ((CASE r.statement_end_offset
              WHEN -1 THEN DATALENGTH(t.text)
              ELSE r.statement_end_offset
          END - r.statement_start_offset) / 2) + 1
    ) AS RunningStatement
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.session_id <> @@SPID;
```

Killing a blocker can cause a long rollback and does not fix the root cause. Common fixes are shorter transactions, missing indexes, consistent access order, corrected isolation, and removing application think-time from transactions.

### 40. How do you find expensive queries without relying only on the plan cache?

**Answer:** Prefer Query Store for durable, per-database history and regression analysis. Plan-cache DMVs are useful for recent aggregate evidence but reset on restart, recompile, eviction, and cache clearing.

```sql
SELECT TOP (20)
    qs.execution_count,
    qs.total_worker_time / 1000.0 AS TotalCpuMs,
    qs.total_elapsed_time / 1000.0 AS TotalElapsedMs,
    qs.total_logical_reads,
    (qs.total_worker_time / NULLIF(qs.execution_count, 0)) / 1000.0
        AS AvgCpuMs,
    SUBSTRING
    (
        st.text,
        (qs.statement_start_offset / 2) + 1,
        ((CASE qs.statement_end_offset
              WHEN -1 THEN DATALENGTH(st.text)
              ELSE qs.statement_end_offset
          END - qs.statement_start_offset) / 2) + 1
    ) AS StatementText
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
ORDER BY qs.total_worker_time DESC;
```

Rank by the metric matching the incident: total CPU for server load, average latency for slow requests, logical reads for inefficient access, or execution count for a cheap query called excessively.

### 41. How do you design an idempotent API/database write?

**Answer:** Accept an idempotency key, store it under a unique constraint in the same transaction as the business write, and return the original outcome for retries. Do not rely on “check then insert” without concurrency protection.

```sql
CREATE TABLE dbo.ApiRequests
(
    IdempotencyKey uniqueidentifier NOT NULL
        CONSTRAINT PK_ApiRequests PRIMARY KEY,
    OrderId bigint NULL,
    CreatedAt datetime2(3) NOT NULL
        CONSTRAINT DF_ApiRequests_CreatedAt DEFAULT SYSUTCDATETIME()
);
```

The transaction should establish ownership of the key, create the order once, record the result, and handle duplicate-key races. Define behavior for a retry that reuses a key with a different payload, commonly by storing and comparing a request hash.

### 42. Normalize or denormalize?

**Answer:** Normalize OLTP data to enforce dependencies, reduce update anomalies, and make invariants explicit. Denormalize only for a measured read or integration requirement, with a clear owner and consistency mechanism. Indexed views, persisted computed columns, summary tables, caches, and analytical replicas are controlled forms of denormalization. “Joins are slow” is not sufficient evidence.

## Legacy SQL Server to Azure SQL migration scenarios

“Azure SQL Server” is often used informally, but an expert should first identify the exact target:

- **Azure SQL Database** is a database-scoped PaaS service for modern applications, with options such as single databases, elastic pools, serverless, Business Critical, and Hyperscale.
- **Azure SQL Managed Instance** is an instance-scoped PaaS service with high SQL Server compatibility, including SQL Server Agent and cross-database capabilities.
- **SQL Server on Azure Virtual Machines** provides operating-system access and the highest lift-and-shift compatibility, but the customer retains more administrative responsibility.

### 43. Scenario: How would you choose the correct Azure SQL target for a legacy application?

The application uses SQL Server Agent, linked servers, CLR, cross-database transactions, and a vendor component that writes files to the local server.

**Answer:** Do not choose from database size alone. Inventory instance-level dependencies, operating-system dependencies, latency, availability requirements, licensing, operational ownership, and willingness to refactor.

| Requirement | Likely target and reasoning |
|---|---|
| Modern database-scoped application that can remove instance dependencies | Azure SQL Database offers the most managed experience. |
| SQL Agent, cross-database queries, Service Broker, or other instance features must remain | Azure SQL Managed Instance usually minimizes refactoring. |
| File-system access, unsupported drivers/extensions, full OS control, or exact SQL Server compatibility is mandatory | SQL Server on Azure VM is the safer initial target. |
| Many independently scalable tenant databases | Azure SQL Database with elastic pools is a strong candidate. |

In this scenario, test Managed Instance compatibility first. If the vendor truly requires local OS/file access or an unsupported component, use SQL Server on Azure VM as the initial landing zone and create a later modernization plan. Never force Azure SQL Database when the required refactoring exceeds the project's risk or schedule.

### 44. Scenario: Plan the migration of a 2 TB SQL Server 2008 R2 database with only 15 minutes of downtime.

**Answer:** Treat this as a program with discovery, remediation, rehearsal, migration, and optimization phases:

1. Inventory databases, logins, jobs, linked servers, SSIS packages, CLR, Service Broker, replication, certificates, encryption keys, and external integrations.
2. Capture at least one representative business cycle of CPU, I/O, waits, throughput, data growth, transaction-log generation, query latency, and peak concurrency.
3. Run a current migration-readiness assessment through SQL Server enabled by Azure Arc, Azure Migrate, or the migration experience in current SSMS. Resolve blockers and use measured sizing recommendations.
4. Select the target. Managed Instance is often the lowest-refactoring PaaS destination for a legacy instance; Azure SQL Database requires database-scoped compatibility.
5. Rehearse schema, data, security, job, and application migrations in a production-like environment.
6. Use an online or continuously synchronized path that supports the selected source and target. SQL Server 2008 R2 cannot use the Managed Instance link, which requires SQL Server 2016 or later. For a Managed Instance target, evaluate Azure DMS online migration; alternatively, upgrade the source to a supported intermediate version before using the link. Confirm the current support matrix in a rehearsal rather than assuming that every online method supports every version.
7. Run repeated cutover rehearsals and measure final synchronization plus application reconnection time.
8. At cutover, stop source writes, drain jobs and queues, synchronize, validate, redirect clients, and run smoke tests.
9. Keep the source protected and read-only for the agreed rollback window.

A 15-minute objective cannot be guaranteed until a full-volume rehearsal proves it. Also define the rollback point: once the target accepts new writes, returning to the source requires reverse synchronization or reconciliation, not merely changing a connection string.

### 45. Scenario: The assessment reports hundreds of compatibility findings. How do you prioritize them?

**Answer:** Classify findings by impact and dependency:

- **Migration blockers:** Unsupported features or syntax that prevent schema deployment or workload execution.
- **Correctness risks:** Changed behavior, collation differences, date/number conversions, isolation assumptions, and security mapping.
- **Performance risks:** Non-SARGable SQL, old cardinality assumptions, hints, oversized indexes, and chatty data access.
- **Operational gaps:** Agent jobs, backups, alerts, linked servers, Database Mail, maintenance scripts, and monitoring.
- **Cleanup:** Unused objects and deprecated syntax that do not block the current move.

Prioritize by business criticality and runtime evidence, not raw finding count. Map every finding to an owning application, remediation, test, and target release. A feature reported in an unused stored procedure is lower risk than a silent semantic change in the payment path.

### 46. Scenario: Should you migrate and raise the database compatibility level in the same release?

**Answer:** Usually separate the physical/platform migration from the optimizer-behavior change. Move first at a supported compatibility level, establish correctness and performance, capture a Query Store baseline, and then raise compatibility in a separately tested change.

```sql
SELECT name, compatibility_level
FROM sys.databases
WHERE name = DB_NAME();

ALTER DATABASE CURRENT SET QUERY_STORE = ON
(
    OPERATION_MODE = READ_WRITE,
    QUERY_CAPTURE_MODE = AUTO
);

-- Perform only after workload testing and a regression plan.
ALTER DATABASE CURRENT SET COMPATIBILITY_LEVEL = 160;
```

Test application behavior, estimates, plans, and critical queries after the change. SQL Server 2022/Azure SQL features such as Parameter Sensitive Plan optimization may require compatibility level 160. If the target does not support the source level, the upgrade cannot be separated completely; increase testing and remediate before cutover.

### 47. Scenario: A legacy procedure joins four databases with three-part names. The target is Azure SQL Database.

```sql
SELECT c.CustomerId, o.OrderId, l.CreditLimit
FROM CRM.dbo.Customers AS c
JOIN Sales.dbo.Orders AS o ON o.CustomerId = c.CustomerId
JOIN Finance.dbo.CreditLimits AS l ON l.CustomerId = c.CustomerId;
```

**Answer:** Azure SQL Database is database-scoped and does not provide transparent SQL Server-style cross-database queries and transactions. Choose deliberately among these options:

- Consolidate tightly coupled databases into one database and separate domains with schemas.
- Copy or synchronize the small reference data needed locally.
- Use elastic query/external tables for supported read scenarios, accepting distributed-query latency and limitations.
- Move orchestration to the application or a data-integration service and design compensating behavior instead of assuming one ACID transaction.
- Select Managed Instance when cross-database behavior is fundamental and refactoring is not justified.

Do not replace a local join with multiple row-by-row network calls. Redesign the operation around bulk requests, local projections, or bounded service calls and test failure behavior between steps.

### 48. Scenario: What happens to SQL Server Agent jobs during migration?

**Answer:** Jobs do not automatically become a correct cloud operating model.

- On **Managed Instance**, SQL Server Agent is available, but proxy accounts, file paths, CmdExec/PowerShell steps, Database Mail, credentials, owners, and external endpoints still require review.
- On **Azure SQL Database**, use Elastic Jobs for scheduled T-SQL across databases or services such as Azure Automation, Functions, Logic Apps, or Data Factory according to the workload.

For every job, document its schedule, dependencies, credentials, retry behavior, alerting, duration, overlap policy, and idempotency. Disable the source schedule before enabling the target schedule, otherwise both environments can process the same work.

### 49. Scenario: How do you prove that source and target data match before cutover?

**Answer:** Use layered reconciliation rather than trusting a single row count or `CHECKSUM`:

```sql
SELECT
    COUNT_BIG(*) AS OrderCount,
    MIN(OrderId) AS MinimumOrderId,
    MAX(OrderId) AS MaximumOrderId,
    SUM(TotalAmount) AS TotalOrderAmount,
    MIN(OrderDate) AS FirstOrderDate,
    MAX(OrderDate) AS LastOrderDate
FROM dbo.Orders
WHERE OrderDate >= @ValidationStart
  AND OrderDate <  @ValidationEnd;
```

Compare source and target by deterministic key ranges or business periods. Also validate:

- Critical business totals and state transitions.
- Row counts per tenant/status/date partition, not only whole-table counts.
- Sampled or chunked cryptographic hashes based on canonical representations for high-risk data.
- Foreign-key orphans, nullability, unique constraints, identity/sequence positions, and temporal history.
- Permissions, users, procedures, functions, triggers, jobs, and configuration.
- New changes captured during continuous synchronization.

Checksums can collide, floating-point/string formatting can differ, and tables continue changing during an online migration. Record the validation watermark and repeat the delta checks after writes stop.

### 50. Scenario: How do you size Azure SQL without copying the old server's CPU and RAM?

**Answer:** Size from the workload, not the hardware label. Collect peak and percentile CPU, memory pressure, IOPS, throughput, storage latency, log generation rate, database size/growth, tempdb usage, active sessions/workers, and query latency over representative peak periods.

Then:

1. Use migration assessment recommendations as a starting point.
2. Load a production-like copy and replay or simulate realistic concurrency.
3. Compare General Purpose, Business Critical, and Hyperscale characteristics where applicable.
4. Measure application-to-database network latency in the intended topology.
5. Load test background jobs, index maintenance, failovers, and peak traffic together.
6. Select headroom and scaling thresholds tied to an SLO, not average utilization.

Azure metrics such as CPU, data I/O, log I/O, workers, sessions, and storage should be correlated with Query Store. Scaling can relieve resource saturation, but it does not correct an inefficient query or an excessively chatty application.

### 51. Scenario: A procedure is fast on-premises but slow after migration. What do you investigate first?

**Answer:** Compare evidence across environments:

- Exact parameters, result size, session `SET` options, compatibility level, and database-scoped configuration.
- Actual plans, estimated versus actual rows, join choices, spills, memory grants, and parallelism.
- Statistics age/sample after the bulk data load and index/schema parity.
- Query Store runtime history and plan changes.
- Parameter sensitivity and data skew.
- CPU, data I/O, log I/O, worker/session limits, tempdb pressure, and service-tier characteristics.
- Application-region placement, connection setup, result transfer, and number of round trips.

```sql
SELECT
    q.query_id,
    p.plan_id,
    rs.avg_duration,
    rs.avg_cpu_time,
    rs.avg_logical_io_reads,
    rs.count_executions,
    rsi.start_time,
    rsi.end_time
FROM sys.query_store_query AS q
JOIN sys.query_store_plan AS p
  ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats AS rs
  ON rs.plan_id = p.plan_id
JOIN sys.query_store_runtime_stats_interval AS rsi
  ON rsi.runtime_stats_interval_id = rs.runtime_stats_interval_id
WHERE q.query_id = @QueryId
ORDER BY rsi.start_time DESC;
```

Do not immediately force the old plan: a plan suited to different hardware, cardinality, or service characteristics can be worse in Azure. Force or hint only after controlled comparison and retain a rollback path.

### 52. Scenario: Azure SQL periodically drops connections during maintenance. How should the application change?

**Answer:** Brief connection interruptions and transient faults are normal in cloud systems. Use a current driver, open connections late, close them promptly, and implement bounded retries with exponential backoff and jitter for errors classified as transient.

```text
Server=tcp:example.database.windows.net,1433;
Database=Sales;
Encrypt=True;
TrustServerCertificate=False;
Connect Timeout=30;
ConnectRetryCount=3;
ConnectRetryInterval=10;
```

Connection-string retry options help connection resiliency; they are not permission to replay every failed command. For command/transaction retries:

- Open a fresh connection before retrying.
- Retry the complete transaction, not an arbitrary statement from its middle.
- Make writes idempotent because the client can lose the response after the server committed.
- Bound attempts and elapsed time; apply jitter to avoid a retry storm.
- Do not retry authentication, syntax, constraint, or permanent business errors as if they were transient.
- Log the operation ID, attempt, error number, and final outcome.

Use the idempotency-key pattern from question 41 for APIs that can create financial or order records.

### 53. Scenario: Replace a SQL username and password embedded in application configuration.

**Answer:** For an Azure-hosted application, prefer a managed identity with Microsoft Entra authentication. Configure the Azure SQL logical server/instance for Entra authentication, create a contained database user for the identity, and grant only required permissions.

```sql
CREATE USER [orders-api-prod] FROM EXTERNAL PROVIDER;

GRANT EXECUTE ON SCHEMA::api TO [orders-api-prod];
GRANT SELECT ON SCHEMA::reporting TO [orders-api-prod];
```

Example connection-string intent for a supported current driver:

```text
Server=tcp:example.database.windows.net,1433;
Database=Sales;
Authentication=Active Directory Managed Identity;
Encrypt=True;
TrustServerCertificate=False;
```

Do not grant `db_owner` merely to make deployment pass. Separate migration/deployment identity from runtime identity, test token acquisition and connection-pool behavior, and retain an audited emergency-access procedure.

### 54. Scenario: The application makes 5,000 single-row calls per request. It becomes slow after moving to Azure.

**Answer:** The local network previously hid a chatty design. Reduce round trips using set-based stored procedures, table-valued parameters, bulk copy, and appropriately sized batches.

```sql
CREATE TYPE dbo.OrderItemInput AS TABLE
(
    LineNo int NOT NULL PRIMARY KEY,
    ProductId int NOT NULL,
    Quantity int NOT NULL,
    UnitPrice decimal(19,4) NOT NULL
);
GO

CREATE OR ALTER PROCEDURE dbo.AddOrderItems
    @OrderId bigint,
    @Items dbo.OrderItemInput READONLY
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    INSERT dbo.OrderItems(OrderId, LineNo, ProductId, Quantity, UnitPrice)
    SELECT @OrderId, LineNo, ProductId, Quantity, UnitPrice
    FROM @Items;
END;
GO
```

Table-valued parameters do not maintain column statistics. For a very large or skewed input, copy it into a temporary table, add the useful index, and then join. Measure batch size against latency, memory, log rate, locking, timeouts, and retry cost.

### 55. Scenario: A nightly import overwhelms the Azure SQL log rate and blocks OLTP users.

**Answer:** Diagnose whether the constraint is log I/O, data I/O, CPU, locking, or worker exhaustion. Then combine workload and service changes:

- Stage data with bulk-loading APIs instead of individual inserts.
- Load in bounded batches and avoid one enormous transaction.
- Insert in clustered-key order where practical.
- Remove or defer only provably unnecessary secondary-index work; preserve correctness constraints.
- Separate staging from user-facing tables and switch/merge in controlled steps.
- Schedule or throttle against the OLTP SLO.
- Temporarily scale the target only when tests show that the tier change raises the actual bottleneck.
- Monitor replication/geo-replica lag and retry cost.

```sql
WHILE 1 = 1
BEGIN
    ;WITH Batch AS
    (
        SELECT TOP (5000) *
        FROM dbo.ImportStage
        WHERE ProcessedAt IS NULL
        ORDER BY StageId
    )
    UPDATE Batch
    SET ProcessedAt = SYSUTCDATETIME();

    IF @@ROWCOUNT = 0 BREAK;
END;
```

The example illustrates deterministic batching; a real import should atomically claim and apply each batch and be restartable without duplicating business rows.

### 56. Scenario: Modernize legacy data types and time handling during migration.

**Answer:** Inventory semantics before mechanically changing types:

- Replace `text`, `ntext`, and `image` with `(n)varchar(max)` and `varbinary(max)` as appropriate.
- Prefer `datetime2` over `datetime` for new designs because of range and precision.
- Store an absolute event instant in UTC; retain the source time-zone identifier when future local-time reconstruction matters. Use `datetimeoffset` when the original offset is part of the business value.
- Replace approximate `float` with `decimal(p,s)` for values requiring exact arithmetic.
- Validate `money` scale/range against explicit `decimal` business rules.
- Review `timestamp`; in SQL Server it is a deprecated synonym for `rowversion`, not date/time.
- Check collation effects on comparison, sorting, case/accent sensitivity, and temporary objects.

```sql
SELECT
    EventId,
    LegacyLocalTime,
    LegacyLocalTime AT TIME ZONE 'India Standard Time' AS EventWithOffset,
    (LegacyLocalTime AT TIME ZONE 'India Standard Time')
        AT TIME ZONE 'UTC' AS EventUtc
FROM dbo.LegacyEvents;
```

Time-zone conversion requires a known source zone and special testing around daylight-saving gaps and overlaps. Never infer historical time zone from a server's current setting.

### 57. Scenario: How do you handle unsupported server features in Azure SQL Database?

**Answer:** Build a feature-remediation map rather than blindly translating syntax.

| Legacy dependency | Possible Azure SQL Database response |
|---|---|
| SQL Server Agent | Elastic Jobs or an external Azure scheduler/orchestrator. |
| CLR or `xp_cmdshell` | Move logic into a secured application, Function, or worker service. |
| Local file `BULK INSERT` | Use supported Azure Storage access or a data-integration/bulk-copy path. |
| Linked server | API/data pipeline, local projection, elastic query where supported, or select Managed Instance. |
| Cross-database transaction | Consolidate data, redesign the consistency boundary, or select Managed Instance. |
| Server login dependency | Contained database users and Microsoft Entra identities. |
| Profiler-based monitoring | Extended Events, Query Store, Azure monitoring, and diagnostics. |

The correct response may be to change the target. A feature that is central to hundreds of critical procedures can make Managed Instance less risky than a rushed rewrite for Azure SQL Database.

### 58. Scenario: Design a multi-tenant database model after migration.

**Answer:** Evaluate isolation, scale, noisy-neighbor risk, tenant-specific restore, schema deployment, residency, encryption, and cost.

- **Shared database/shared schema:** Lowest database count and simple aggregate queries; every key/index/security predicate must include `TenantId`, and one tenant can affect others.
- **Database per tenant:** Strong isolation, independent scale/restore, and placement; requires fleet automation. Azure SQL elastic pools can share resources across many databases.
- **Sharded groups of tenants:** Middle ground, but requires a shard catalog, routing, rebalancing, and cross-shard reporting strategy.

For a shared model, enforce tenant access in multiple layers. Row-level security can provide database-side defense, but it does not replace correct application authorization and tenant-aware unique/index keys.

```sql
CREATE INDEX IX_Orders_Tenant_Customer_Date
ON dbo.Orders(TenantId, CustomerId, OrderDate DESC)
INCLUDE (Status, TotalAmount);
```

The example assumes the migrated schema adds `TenantId`. Avoid one table per tenant; it creates object explosion and difficult deployments.

### 59. Scenario: Built-in high availability exists. Why does the application still need a disaster-recovery design?

**Answer:** Local high availability does not by itself satisfy recovery from regional outage, accidental deletion, malicious changes, or application corruption. Define business RTO and RPO, then select backups/retention, zone redundancy, geo-replication or failover groups, and a tested recovery process.

For failover groups:

- Connect applications through the read-write listener rather than a region-specific server name.
- Expect connection interruption and use bounded retry logic.
- Decide customer-managed versus automatic failover based on data-loss and control requirements.
- Deploy logins/users, firewall/private DNS, managed identities, keys, alerts, and jobs in the recovery environment.
- Test failover and failback with the complete application, not only a successful SQL connection.

Asynchronous geo-replication can imply data loss during forced failover. The business, not only the DBA, must approve that RPO.

### 60. Scenario: What should a production migration cutover and rollback runbook contain?

**Answer:** Make the runbook executable, timed, owned, and rehearsed:

1. Entry criteria: assessment approved, load/failover tests passed, target synchronized, validation within tolerance, backups and observability ready.
2. Named decision owner, communication channel, checkpoints, stop conditions, and maximum outage clock.
3. Freeze schema deployments; pause source jobs, integrations, and writers in a defined order.
4. Record source watermark and final synchronization state.
5. Run reconciliation and security/configuration checks.
6. Switch endpoints/configuration through a reversible deployment mechanism.
7. Run technical smoke tests and business transaction tests.
8. Monitor error rate, latency percentiles, throughput, resource saturation, blocking, and queues against explicit thresholds.
9. Declare success only after the observation period and business-owner sign-off.
10. If a stop condition is breached, execute the predetermined rollback or roll-forward procedure.

The rollback section must state how target writes are handled. Before target writes, rollback may be an endpoint reversal. After target writes, it needs reverse replication, dual-write reconciliation, or an accepted loss window. “Restore the old backup” is not a complete rollback plan.

## Rapid-fire questions

### `DELETE` vs `TRUNCATE` vs `DROP`

- `DELETE` removes qualifying rows, supports a `WHERE` clause, fires delete triggers, and logs row changes.
- `TRUNCATE TABLE` deallocates data pages, removes all rows, resets an identity seed, takes a schema modification lock, and has restrictions such as referenced foreign keys.
- `DROP TABLE` removes the table object, its data, indexes, constraints, and permissions.

Both `DELETE` and `TRUNCATE` can be rolled back when executed in an explicit SQL Server transaction. “TRUNCATE cannot be rolled back” is a common incorrect interview answer.

### `UNION` vs `UNION ALL`

`UNION ALL` concatenates results and preserves duplicates. `UNION` removes duplicates, commonly requiring an additional sort or hash step. Use `UNION ALL` unless distinctness is a real requirement.

### `WHERE` vs `HAVING`

`WHERE` filters input rows before grouping; `HAVING` filters groups after aggregation. Push nonaggregate filters to `WHERE` so fewer rows reach the aggregate.

### `COUNT(*)`, `COUNT(column)`, and `COUNT_BIG(*)`

`COUNT(*)` counts rows, `COUNT(column)` counts non-null values, and `COUNT_BIG` returns `bigint` for counts that may exceed `int`.

### `IDENTITY` vs `SEQUENCE`

`IDENTITY` belongs to one table and generates a value during insert. A `SEQUENCE` is a separate object that can serve multiple tables and can allocate values before an insert. Neither promises gap-free numbering; rollbacks, caching, and failures can create gaps.

### Primary key vs unique constraint

Both enforce uniqueness and normally create a unique index. A table has one primary key but can have multiple unique constraints. A primary-key column is non-null; SQL Server unique constraints allow a single `NULL` per nullable key combination under their normal semantics.

### `SCOPE_IDENTITY()` vs `@@IDENTITY`

`SCOPE_IDENTITY()` returns the last identity generated in the current scope. `@@IDENTITY` can return an identity generated by a trigger in another scope. For multi-row inserts, prefer the `OUTPUT inserted.Id` clause.

### `COALESCE` vs `ISNULL`

`ISNULL` is SQL Server-specific and returns the type/length determined primarily by its first argument. `COALESCE` is standard SQL, accepts multiple expressions, and follows `CASE`-style data type precedence. They can also differ in nullability metadata and evaluation behavior, so they are not universally interchangeable.

### `RAISERROR` vs `THROW`

Use `THROW` for new T-SQL error handling. It preserves the original error when used without arguments in a `CATCH` block and honors `SET XACT_ABORT`; `RAISERROR` has legacy formatting/severity capabilities but does not behave identically.

### Heap vs clustered table

A heap has no clustered index. It can be useful for specific staging/loading patterns, but updates can create forwarded records and nonclustered indexes use a row identifier. A clustered table is not automatically optimal either; choose a narrow, stable, preferably increasing clustered key based on workload.

### Can index fragmentation explain every slow query?

No. Investigate waits, reads, estimates, plans, blocking, memory grants, statistics, and parameter sensitivity first. Routine rebuilds can consume log, I/O, CPU, and availability-replica bandwidth without addressing the actual bottleneck.

## Interview evaluation checklist

A strong expert answer should demonstrate all of the following:

- **Correctness:** Handles nulls, duplicates, ties, time boundaries, overflow, and multi-row operations.
- **Performance reasoning:** Discusses SARGability, indexes, cardinality, memory, I/O, and expected plan shape.
- **Concurrency reasoning:** Identifies race conditions, isolation behavior, deadlocks, transaction scope, and retry strategy.
- **Operational awareness:** Uses Query Store, plans, waits, monitoring, rollback plans, and production-safe batching.
- **Security:** Uses parameters, least privilege, protected secrets/keys, and layered data protection.
- **Trade-offs:** Avoids absolute claims such as “always use a temp table,” “never use a cursor,” or “NOLOCK makes queries faster.”
- **SQL Server 2022 awareness:** Knows compatibility level 160, PSP optimization, Query Store hints, IQP feedback, and new T-SQL functions.
- **Azure migration judgment:** Selects the correct Azure SQL target, finds compatibility blockers, proves performance and data integrity, designs resilient access, and rehearses cutover and rollback.

## Official references

- [What's new in SQL Server 2022](https://learn.microsoft.com/sql/sql-server/what-s-new-in-sql-server-2022)
- [Parameter Sensitive Plan optimization](https://learn.microsoft.com/sql/relational-databases/performance/parameter-sensitive-plan-optimization)
- [Query Store hints](https://learn.microsoft.com/sql/relational-databases/performance/query-store-hints)
- [Monitor performance with Query Store](https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)
- [Execution plans](https://learn.microsoft.com/sql/relational-databases/performance/execution-plans)
- [Transaction locking and row versioning](https://learn.microsoft.com/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide)
- [`OPENJSON`](https://learn.microsoft.com/sql/t-sql/functions/openjson-transact-sql)
- [`GENERATE_SERIES`](https://learn.microsoft.com/sql/t-sql/functions/generate-series-transact-sql)
- [Choose among Azure SQL Database, Managed Instance, and SQL Server on Azure VM](https://learn.microsoft.com/azure/azure-sql/azure-sql-iaas-vs-paas-what-is-overview)
- [SQL Server to Azure SQL Database migration guide](https://learn.microsoft.com/azure/azure-sql/database/migrate-to-database-from-sql-server)
- [Azure Database Migration Service](https://learn.microsoft.com/azure/dms/)
- [Azure SQL Database migration assessment rules](https://learn.microsoft.com/data-migration/sql-server/database/assessment-rules)
- [Managed Instance link](https://learn.microsoft.com/azure/azure-sql/managed-instance/managed-instance-link-feature-overview)
- [Azure SQL transient-fault resiliency](https://learn.microsoft.com/azure/reliability/reliability-sql-database)
- [Microsoft Entra authentication for Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-overview)
- [Elastic Jobs for Azure SQL Database](https://learn.microsoft.com/azure/azure-sql/database/elastic-jobs-tsql-create-manage)

---

These questions represent high-frequency interview themes, not a claim that every company uses the same question bank. Interviewers usually change the schema and constraints to test reasoning rather than memorization.
