SQL Server
==========

> Read **[General Principles](/Database-Standards/General-Principles)** first. The topic pages are written for PostgreSQL; this page is what's **different** on SQL Server.
> **When we use SQL Server at all:** see [Database Standards → Which database?](/Database-Standards#which-database).

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [When we use it](#when-we-use-it) | **When the client mandates it, typically FI clients on .NET**<br/><sub>New projects with no client requirement use PostgreSQL.</sub> |
| [Versions](#versions-and-hosting) | **SQL Server 2022 or 2025, or Azure SQL, at the latest compatibility level**<br/><sub>Many newer features only switch on at the matching compatibility level, not just the version.</sub> |
| [Translating from Postgres](#translating-from-postgres) | **`TOP`/`OFFSET FETCH`, `OUTPUT`, `+` for strings, `bit`, `datetime2`**<br/><sub>Most of T-SQL maps one-to-one. The table below covers the cases that trip people up.</sub> |
| [Clustered key](#the-clustered-index-is-the-table) | **`bigint IDENTITY` clustered primary key, public GUID in its own unique column**<br/><sub>The clustered key is the physical order of the table. A random key splits pages on every insert.</sub> |
| [Text and collation](#text-types-and-collation) | **`nvarchar` by default, never `text`/`ntext`, and know the collation is case-insensitive**<br/><sub>`'ABC' = 'abc'` is true on a default install, including in UNIQUE constraints.</sub> |
| [Implicit conversions](#implicit-conversions) | **Parameter types match column types**<br/><sub>An `nvarchar` parameter against a `varchar` column can turn an index seek into a full scan.</sub> |
| [Parameter sniffing](#parameter-sniffing) | **Know the symptom, fix it with Query Store, not by flushing the plan cache**<br/><sub>"Fast for one customer, slow for another" is usually one cached plan reused for very different values.</sub> |
| [Isolation](#isolation-and-blocking) | **Turn on `READ_COMMITTED_SNAPSHOT`, never sprinkle `NOLOCK`**<br/><sub>Readers stop blocking writers without reading uncommitted, missing or duplicate rows.</sub> |
| [Upserts](#upserts-and-merge) | **`UPDATE` then `INSERT` with `UPDLOCK, SERIALIZABLE`, inside a transaction**<br/><sub>`MERGE` has a history of bugs and still races without `HOLDLOCK`.</sub> |
| [Stored procedures](#stored-procedures) | **Queries live in the app by default. Procedures only with a reason, and always in migrations**<br/><sub>A procedure edited by hand in production is a schema change nobody can see.</sub> |
| [EF Core and Dapper](#ef-core-and-dapper-on-sql-server) | **Retry on transient errors, `rowversion` for concurrency, `varchar` parameters typed as ANSI**<br/><sub>Azure SQL drops connections during failover. Your code should shrug that off.</sub> |
| [Diagnostics](#finding-slow-queries) | **Query Store first, then the actual execution plan**<br/><sub>Query Store shows what was slow yesterday. The plan shows why.</sub> |
| [Backups](#backups) | **FULL recovery model with log backups, and `COPY_ONLY` for ad-hoc backups**<br/><sub>Point-in-time restore needs an unbroken chain of log backups.</sub> |

* * *

## When we use it

**Not** our default. Only when the client mandates it (common with **FI clients**: banks and insurers whose infrastructure, licensing and DBAs are built around it), or the project extends an existing system on it. Almost always **.NET** with **EF Core**, often plus **Dapper**: see [Coding Standards → Backend — .NET / C#](/Coding-Standards/Backend-DotNet-CSharp).

> 📝 **Naming:** EF Core's default PascalCase (`dbo.Invoices`, `CustomerId`) is acceptable on SQL Server, and this page uses it. If the project already uses snake_case, keep it. Never mix the two in one database. See [General Principles → One naming convention per database](/Database-Standards/General-Principles#one-naming-convention-per-database).

* * *

## Versions and hosting

| Option | What it is | Notes |
| --- | --- | --- |
| **SQL Server 2025** (17.x) | Latest, generally available since November 2025 | Our pick for a new install when the client lets us choose. Adds a native `json` type, regular-expression functions and vector support. |
| **SQL Server 2022** (16.x) | Previous release, very common on client servers | Fully fine. Query Store on by default for new databases. |
| **SQL Server 2019 and older** | Still on client estates | Check every feature you use exists there. 2016 and older are out of Microsoft support: flag them to the client. |
| **Azure SQL Database** | Fully managed single database (PaaS) | Backups, patching and high availability handled. No SQL Agent, no cross-database queries. |
| **Azure SQL Managed Instance** | Managed, close to full SQL Server compatibility | When the client needs Agent jobs, cross-database queries or an easy lift from on-premises. |

> ⚠️ **Version and compatibility level are different things.** A database restored onto SQL Server 2022 can still run at an old compatibility level, leaving `GENERATE_SERIES`, `GREATEST`/`LEAST`, parameter-sensitive plans and other features off. Check it:
> ```sql
> SELECT name, compatibility_level FROM sys.databases;
> -- 170 = SQL Server 2025, 160 = 2022, 150 = 2019
> ```

### Tools

| Tool | Use it for |
| --- | --- |
| **SSMS** (SQL Server Management Studio, Windows) | The full toolset: execution plans, Query Store reports, Agent jobs, backups. |
| **VS Code + MSSQL extension** | Cross-platform querying and database projects. Replaces Azure Data Studio, and opens its database projects. |
| **DataGrip / DBeaver** | If you already use them for Postgres. |

> 📝 **Azure Data Studio was retired on 28 February 2026.** Don't install it on a new machine.

* * *

## Translating from Postgres

| Postgres | SQL Server | Watch out for |
| --- | --- | --- |
| `LIMIT 10` | `SELECT TOP (10) ...` | |
| `LIMIT 10 OFFSET 20` | `OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY` | Only valid with an `ORDER BY` |
| `bigint GENERATED ALWAYS AS IDENTITY` | `bigint IDENTITY(1,1)` | |
| `gen_random_uuid()` | `NEWID()` (random) or `NEWSEQUENTIALID()` (ordered) | `NEWSEQUENTIALID()` only works as a column `DEFAULT` |
| `'a' \|\| 'b'` | `'a' + 'b'` or `CONCAT('a', 'b')` | `+` returns NULL if either side is NULL; `CONCAT` treats NULL as `''` |
| `COALESCE(x, 0)` | `COALESCE(x, 0)` | Prefer it over `ISNULL`, which takes the first argument's type and can silently truncate |
| `now()` | `SYSUTCDATETIME()` / `SYSDATETIMEOFFSET()` | `GETDATE()` is server-local time. `GETUTCDATE()` returns the older `datetime` type |
| `timestamptz` | `datetimeoffset`, or `datetime2` always written in UTC | Avoid `datetime`: it rounds to about 3 ms and has a 1753 minimum date |
| `text` | `nvarchar(n)` / `nvarchar(max)` | `text` and `ntext` are deprecated |
| `boolean` | `bit` | No `TRUE`/`FALSE` literals; use `1`/`0`. A `bit` can't be used directly as a condition |
| `numeric(12,2)` | `decimal(12,2)` | Avoid the `money` type: it loses precision in division and multiplication |
| `INSERT ... RETURNING id` | `INSERT ... OUTPUT inserted.Id VALUES (...)` | Also works on `UPDATE` (`deleted.` and `inserted.`) and `DELETE` |
| `INSERT ... ON CONFLICT` | No direct equivalent | See [Upserts and MERGE](#upserts-and-merge) |
| `string_agg(name, ', ')` | `STRING_AGG(Name, ', ') WITHIN GROUP (ORDER BY Name)` | SQL Server 2017+ |
| `"Identifier"` | `[Identifier]` | Double quotes also work while `QUOTED_IDENTIFIER` is ON, which is the default |
| `ILIKE` | `LIKE` | Case sensitivity comes from the collation, not the operator |
| Partial index `WHERE ...` | Filtered index `WHERE ...` | Queries with parameters may not be able to use a filtered index |
| `generate_series(1, 10)` | `GENERATE_SERIES(1, 10)` | SQL Server 2022+, compatibility level 160 |
| `jsonb` | `nvarchar(max)` + `JSON_VALUE` / `OPENJSON`, or the native `json` type | `json` type needs SQL Server 2025 or Azure SQL |

> ⚠️ **`GO` is not T-SQL.** It's a batch separator for SSMS and `sqlcmd`. A script containing `GO` sent through EF Core, Dapper or ADO.NET fails with a syntax error. Split it into separate commands, or run it with a tool that understands `GO`.

* * *

## Design differences

### The clustered index is the table

Rows are stored in clustered-key order (by default the primary key), and every nonclustered index points back to the row via that key. A good clustered key is **narrow, unique, never changes, and always increasing**: `bigint IDENTITY` is all four. A random GUID fails the last two: each insert lands on a random page, splits it, and fragments the index.

```sql
-- ✅ narrow, ever-increasing clustered key; the public ID is a separate unique column
CREATE TABLE dbo.Invoices (
    Id          bigint IDENTITY(1,1) NOT NULL,
    PublicId    uniqueidentifier NOT NULL
                CONSTRAINT DF_Invoices_PublicId DEFAULT NEWID(),
    CustomerId  bigint        NOT NULL,
    Number      nvarchar(30)  NOT NULL,
    Total       decimal(12,2) NOT NULL,
    CreatedAt   datetime2(3)  NOT NULL
                CONSTRAINT DF_Invoices_CreatedAt DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_Invoices PRIMARY KEY CLUSTERED (Id),
    CONSTRAINT UQ_Invoices_PublicId UNIQUE NONCLUSTERED (PublicId),
    CONSTRAINT UQ_Invoices_Number UNIQUE NONCLUSTERED (Number),
    CONSTRAINT FK_Invoices_Customers FOREIGN KEY (CustomerId) REFERENCES dbo.Customers (Id)
);

-- ❌ random GUID as the clustered key: page splits on every insert, and every
--    nonclustered index carries a 16-byte key instead of 8
CREATE TABLE dbo.Invoices (Id uniqueidentifier NOT NULL DEFAULT NEWID() PRIMARY KEY, ...);
```

> ⚠️ **UUIDv7 doesn't fix this on SQL Server.** It compares `uniqueidentifier` values starting with the **last** group of bytes, so a v7 GUID (e.g. .NET's `Guid.CreateVersion7()`) still lands in effectively random places, unlike on Postgres. (Microsoft doesn't document the order; this comes from published testing.) Keep the `bigint IDENTITY` clustered key; a `NEWID()` or v7 public GUID is fine in a unique nonclustered column, as above.

> 📝 Index foreign key columns yourself (`CustomerId` above): SQL Server doesn't, same as Postgres. See [Indexing](/Database-Standards/Indexing).

### Text types and collation

- **`nvarchar` stores Unicode** (anything a person types). `varchar` stores single-byte text in the collation's code page, unless the column uses a `_UTF8` collation (SQL Server 2019+). Pick one per project: mixing them causes [implicit conversions](#implicit-conversions).
- Give every column a real maximum length. `nvarchar(max)` only for long free text; it can't be an index key column.
- **The collation decides how text compares and sorts.** A default (US English) install uses `SQL_Latin1_General_CP1_CI_AS`, where `CI` means **case-insensitive**: `N'Ali@Example.com' = N'ali@example.com'` is true, and a UNIQUE constraint on Email treats them as duplicates. Usually right for emails and usernames.
- For exact comparison (tokens, codes, hashes), give that column a binary or case-sensitive collation. Don't change the server or database collation after data exists: existing columns keep the old one.

### Implicit conversions

With the default `SQL_` collations, an **`nvarchar` parameter against a `varchar` column** converts the column on every row, turning an index seek into a scan. Windows collations usually still seek, less efficiently.

```sql
-- Column: Customers.Email varchar(200), indexed
-- ✅ parameter typed as varchar → index seek
DECLARE @Email varchar(200) = 'ali@example.com';
SELECT Id FROM dbo.Customers WHERE Email = @Email;

-- ❌ parameter typed as nvarchar (what .NET sends for a string by default) → CONVERT_IMPLICIT on the column
DECLARE @Email nvarchar(200) = N'ali@example.com';
SELECT Id FROM dbo.Customers WHERE Email = @Email;
```

The giveaway is a `CONVERT_IMPLICIT` warning (yellow triangle) on the plan operator. EF Core and Dapper fixes: [EF Core and Dapper on SQL Server](#ef-core-and-dapper-on-sql-server).

* * *

## Writing T-SQL

### Parameter sniffing

The plan built for the first call's parameter values is **cached** for every later call, and with very uneven data one plan doesn't suit every value. **Symptom:** "instant for most customers, 40 seconds for one big one", or "fine yesterday, nothing changed, now slow": the plan was recompiled with an unusual value and stuck.

| Fix | When to use it |
| --- | --- |
| **Parameter Sensitive Plan optimization** | SQL Server 2022+, compatibility level 160+ (check it). On by default: caches several plans per query when an equality predicate has very uneven data. |
| **Query Store plan forcing** | A good plan exists: force it in Query Store, no code change. |
| `OPTION (RECOMPILE)` | Rarely run queries (a report, an admin search): a fresh plan costs less than a bad one. Not for queries running hundreds of times a second. |
| `OPTION (OPTIMIZE FOR (@CustomerId = 42))` or `OPTIMIZE FOR UNKNOWN` | You know which value gives a plan good enough for everyone. |

```sql
-- ✅ a rarely run report: recompile each time, so each customer gets a fitting plan
SELECT i.Number, i.Total, i.CreatedAt FROM dbo.Invoices AS i
WHERE  i.CustomerId = @CustomerId AND i.CreatedAt >= @From
ORDER  BY i.CreatedAt DESC
OPTION (RECOMPILE);
```

> ⚠️ **Don't fix it with `DBCC FREEPROCCACHE`.** Clearing the plan cache on production recompiles everything, spikes CPU, and only works until the next unlucky compile.

### Isolation and blocking

The default **locking** `READ COMMITTED` makes readers wait for writers' locks, so under load reports and APIs block each other. **Our default: turn on `READ_COMMITTED_SNAPSHOT` (RCSI)**, so readers see the last committed row version instead of waiting, as on Postgres. Azure SQL Database has it on by default.

```sql
-- Once per database, in a maintenance window: it needs a moment with no other connections.
ALTER DATABASE CURRENT SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
SELECT name, is_read_committed_snapshot_on FROM sys.databases;   -- check it
```

> ⚠️ **RCSI changes read-then-write code.** "Read the balance, then update it" can now read a value about to change. Lock the row you'll update (`WITH (UPDLOCK)`), or do the arithmetic inside the `UPDATE`. Row versions live in `tempdb` (or in the database with Accelerated Database Recovery): expect a little extra I/O.

**`NOLOCK` is not a performance fix.** `WITH (NOLOCK)` (= `READ UNCOMMITTED`) reads data that may be rolled back a second later, and during page splits can **skip rows or return a row twice**. An occasionally wrong report is worse than a slow one.

```sql
-- ❌ fast and occasionally wrong: dirty, missing or duplicated rows
SELECT SUM(Total) FROM dbo.Invoices WITH (NOLOCK) WHERE CustomerId = @CustomerId;
-- ✅ with RCSI on, a plain read doesn't block and only sees committed data
SELECT SUM(Total) FROM dbo.Invoices WHERE CustomerId = @CustomerId;
```

Deadlocks, isolation levels in general and optimistic locking: [Transactions and Concurrency](/Database-Standards/Transactions-and-Concurrency).

### Upserts and MERGE

There's no `ON CONFLICT`. `MERGE` has a long list of reported bugs, and even when it works it's **not** safe against two sessions inserting the same key at once without `HOLDLOCK`. Use update-then-insert in a transaction, with locking hints that stop the race:

```sql
-- ✅ safe under concurrency: the range lock stops a second session inserting the same key
SET XACT_ABORT ON;
BEGIN TRAN;
UPDATE dbo.CustomerBalances WITH (UPDLOCK, SERIALIZABLE)
SET    Balance = Balance + @Amount
WHERE  CustomerId = @CustomerId;
IF @@ROWCOUNT = 0
    INSERT dbo.CustomerBalances (CustomerId, Balance)
    VALUES (@CustomerId, @Amount);
COMMIT;

-- ❌ two sessions can both see "not matched" and both insert → duplicate-key error
MERGE dbo.CustomerBalances AS t
USING (SELECT @CustomerId AS CustomerId, @Amount AS Amount) AS s
ON t.CustomerId = s.CustomerId
WHEN MATCHED THEN UPDATE SET Balance = t.Balance + s.Amount
WHEN NOT MATCHED THEN INSERT (CustomerId, Balance) VALUES (s.CustomerId, s.Amount);
```

Existing `MERGE` code needs `WITH (HOLDLOCK)` on the target, a statement-ending `;`, and testing under concurrent load.

> 💡 `SET XACT_ABORT ON` makes any runtime error roll back the whole transaction; without it, some errors abort only the failing statement and the transaction carries on half-done. Put it at the top of every procedure or script that opens a transaction.

### Temp tables and table variables

| | `#Temp` table | `@Table` variable |
| --- | --- | --- |
| Statistics | Yes, so the optimizer knows the row count | No column statistics |
| Indexes | Any, created after the table | Only inline PRIMARY KEY, UNIQUE or INDEX in the declaration |
| Rolled back by `ROLLBACK` | Yes | No: contents survive a rollback |
| Use for | Intermediate results over a few hundred rows, joined or filtered again | Small sets: a handful of IDs, a lookup list |

```sql
-- ✅ big intermediate result: a temp table gets statistics and can be indexed
SELECT CustomerId, COUNT(*) AS PaidInvoices
INTO   #PaidCounts
FROM   dbo.Invoices
WHERE  Status = 'paid'
GROUP  BY CustomerId;
CREATE CLUSTERED INDEX IX_PaidCounts_CustomerId ON #PaidCounts (CustomerId);
SELECT c.Name, p.PaidInvoices FROM dbo.Customers AS c JOIN #PaidCounts AS p ON p.CustomerId = c.Id;
DROP TABLE IF EXISTS #PaidCounts;
```

> 📝 Since SQL Server 2019 (compatibility level 150), table variables get a real row count at first compile instead of a guess of 1 row, removing much of their old trap. Temp tables are still safer for anything large.

### Stored procedures

Our default on .NET is queries in the app (EF Core or Dapper), versioned, reviewed and tested with the code. A procedure needs a reason: the client's DBAs require all access through procedures (common at FI clients: the app login only has `EXECUTE`), a heavy set-based batch job is clearer and faster as one procedure than many round trips, or the logic must run inside the database (e.g. a SQL Agent job). General rules for procedures, functions and triggers: [Logic in the Database](/Database-Standards/Logic-in-the-Database).

- Create and change it **in a migration**, with `CREATE OR ALTER PROCEDURE`. Never edit it by hand in SSMS on a shared database.
- Start with `SET NOCOUNT ON;` (no "rows affected" message per statement) and `SET XACT_ABORT ON;`.
- Keep business rules the app also needs in the app. Duplicated logic drifts apart.

```sql
CREATE OR ALTER PROCEDURE dbo.Invoices_MarkOverdue
    @AsOf date
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    UPDATE dbo.Invoices
    SET    Status = 'overdue'
    WHERE  Status = 'sent'
      AND  DueDate < @AsOf;
END;
```

* * *

## EF Core and Dapper on SQL Server

### Retry on transient errors

Azure SQL (and any SQL Server behind a failover group) briefly drops connections during failover and maintenance. Turn on EF Core's retries:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sql => sql.EnableRetryOnFailure()));
```

> ⚠️ With retries on, a transaction you start yourself must run inside the execution strategy (retried as one unit), or EF Core throws `InvalidOperationException`:
> ```csharp
> var strategy = db.Database.CreateExecutionStrategy();
> await strategy.ExecuteAsync(async () =>
> {
>     await using var tx = await db.Database.BeginTransactionAsync();
>     // ...writes...
>     await db.SaveChangesAsync();
>     await tx.CommitAsync();
> });
> ```

Dapper has no built-in retry. On Azure SQL, wrap calls in a retry policy (e.g. Polly) that retries only transient error numbers, and only operations safe to run twice.

### Optimistic concurrency with rowversion

A `rowversion` column changes on every update, so EF Core can detect that someone changed the row since you loaded it:

```csharp
public class Invoice
{
    public long Id { get; set; }
    public decimal Total { get; set; }
    [Timestamp]                       // maps to rowversion
    public byte[] RowVersion { get; set; } = default!;
}
// SaveChanges throws DbUpdateConcurrencyException if the row changed since it was read
```

With Dapper, check it yourself: `UPDATE ... WHERE Id = @Id AND RowVersion = @RowVersion`, and treat 0 rows affected as a conflict.

### Match parameter types to columns

.NET sends strings as `nvarchar` unless told otherwise, causing [implicit conversions](#implicit-conversions) on `varchar` columns.

```csharp
// EF Core: tell the model the column is varchar, and EF sends varchar parameters
modelBuilder.Entity<Customer>()
    .Property(c => c.Email)
    .IsUnicode(false)
    .HasMaxLength(200);

// Dapper: ✅ type the parameter as ANSI (varchar) with the column's length
// (❌ a plain `new { Email = email }` is sent as nvarchar(4000) → CONVERT_IMPLICIT and a scan)
var customer = await conn.QuerySingleOrDefaultAsync<Customer>(
    "SELECT Id, Email, Name FROM dbo.Customers WHERE Email = @Email",
    new { Email = new DbString { Value = email, IsAnsi = true, Length = 200 } });
```

> 💡 If a Dapper project's string columns are **all** `varchar`, set it once at startup: `SqlMapper.AddTypeMap(typeof(string), DbType.AnsiString);`. If they're `nvarchar`, the default is already correct.

### Other things to know

- **EF Core GUID keys** are generated sequentially client-side on SQL Server, avoiding most page splits. We still prefer a `bigint` clustered key ([above](#the-clustered-index-is-the-table)).
- **See the SQL EF Core sends:** `query.ToQueryString()`, or log `Microsoft.EntityFrameworkCore.Database.Command`.
- **Bulk work:** EF Core 7+ `ExecuteUpdateAsync` / `ExecuteDeleteAsync` run one set-based statement without loading entities; `SqlBulkCopy` loads large volumes. See [Large Data and Bulk Operations](/Database-Standards/Large-Data-and-Bulk-Operations).

* * *

## Finding slow queries

### Query Store

Records every query's plans and runtime stats in the database: what was slow yesterday, and with which plan. On by default for new databases on SQL Server 2022+ and in Azure SQL; on older versions, `ALTER DATABASE CURRENT SET QUERY_STORE = ON (OPERATION_MODE = READ_WRITE);`. In SSMS, under the database → **Query Store**, start with **Top Resource Consuming Queries** and **Regressed Queries**; you can force a known good plan from the same report.

### Execution plans and statistics

```sql
SET STATISTICS IO, TIME ON;    -- logical reads and CPU per statement, in the Messages tab
-- In SSMS, Ctrl+M includes the ACTUAL execution plan, then run the query
SELECT ...;
```

| Sign in the plan | Usually means |
| --- | --- |
| **Index Scan / Clustered Index Scan** on a big table, for a query that returns a few rows | Missing index, or a [non-SARGable filter](/Database-Standards/Writing-Queries) |
| **Key Lookup** with a high execution count | The index is missing columns; add them with `INCLUDE (...)` |
| **Yellow warning triangle** | Implicit conversion, spill to `tempdb`, or missing statistics |
| **Estimated rows** wildly different from **actual rows** | Out-of-date statistics, or parameter sniffing |

### Useful DMVs

```sql
-- Top 20 statements by total CPU since the plan was cached
SELECT TOP (20)
       qs.execution_count,
       qs.total_worker_time / qs.execution_count AS avg_cpu_microseconds,
       qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
       SUBSTRING(st.text, (qs.statement_start_offset / 2) + 1,
                 ((CASE qs.statement_end_offset
                       WHEN -1 THEN DATALENGTH(st.text)
                       ELSE qs.statement_end_offset
                   END - qs.statement_start_offset) / 2) + 1) AS statement_text
FROM   sys.dm_exec_query_stats AS qs
CROSS  APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
ORDER  BY qs.total_worker_time DESC;
```

| View | Shows |
| --- | --- |
| `sys.dm_exec_requests` | What's running now, and what it's waiting on or blocked by (`blocking_session_id`) |
| `sys.dm_db_index_usage_stats` | Which indexes are read, and which are only written (candidates to drop) |
| `sys.dm_db_missing_index_details` | Indexes the optimizer wished it had. Hints to investigate, not a to-do list |

> 💡 Two free community tools most SQL Server DBAs use: **sp_WhoIsActive** (what's running and blocking now) and Brent Ozar's **First Responder Kit** (`sp_Blitz`, `sp_BlitzCache`, `sp_BlitzIndex`). Ask before installing anything on a client's server.

Reading plans in general: [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics).

* * *

## Backups

Full strategy and restore testing: [Backups and Recovery](/Database-Standards/Backups-and-Recovery). SQL Server specifics:

| Point | Detail |
| --- | --- |
| **Recovery model** | `FULL` for production: point-in-time restore needs it, plus regular **transaction log backups**. `SIMPLE` restores only to the last full or differential. |
| **Backup types** | Full (weekly or daily), differential (daily), log (every 5–15 minutes, depending on how much data the client can afford to lose). |
| **Log growth** | In `FULL` with no log backups, the log grows until the disk is full: without them, the database isn't really in `FULL`. |
| **Ad-hoc backups** | `BACKUP DATABASE ... WITH COPY_ONLY` before a risky change, so you don't reset the base the scheduled differentials depend on. |
| **Integrity** | `WITH CHECKSUM` on every backup, `DBCC CHECKDB` regularly. `RESTORE VERIFYONLY` checks the file is readable; only a real restore proves it works. |
| **Scheduling** | SQL Agent jobs. Ola Hallengren's free Maintenance Solution is the de facto standard for backup, integrity-check and index jobs. |
| **Azure SQL Database** | Automatic full, differential and log backups; point-in-time restore 7 days by default, configurable up to 35. Long-term retention is a separate setting. |

```sql
-- ✅ ad-hoc safety backup before a risky change
BACKUP DATABASE Billing
TO DISK = N'D:\Backups\Billing_before_migration_0042.bak'
WITH COPY_ONLY, CHECKSUM, COMPRESSION;
```

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Run at the latest compatibility level the client allows | Assume the version alone turns new features on | [Versions](#versions-and-hosting) |
| Use VS Code with the MSSQL extension, or SSMS | Install Azure Data Studio (retired) | [Tools](#tools) |
| `bigint IDENTITY` clustered key, GUID in a unique nonclustered column | A random or v7 GUID as the clustered primary key | [Clustered key](#the-clustered-index-is-the-table) |
| `datetime2` in UTC or `datetimeoffset`, `decimal` for money | `datetime`, `money`, `text`, `ntext` | [Translating from Postgres](#translating-from-postgres) |
| Know your collation is case-insensitive | Assume `'ABC' <> 'abc'` | [Text and collation](#text-types-and-collation) |
| Match parameter types to column types | Send `nvarchar` parameters to `varchar` columns | [Implicit conversions](#implicit-conversions) |
| Fix parameter sniffing with Query Store, PSP or `RECOMPILE` on rare queries | `DBCC FREEPROCCACHE` on production | [Parameter sniffing](#parameter-sniffing) |
| Turn on `READ_COMMITTED_SNAPSHOT` | Add `WITH (NOLOCK)` to make blocking go away | [Isolation](#isolation-and-blocking) |
| `UPDATE` then `INSERT` with `UPDLOCK, SERIALIZABLE` | `MERGE` without `HOLDLOCK` for upserts | [Upserts](#upserts-and-merge) |
| `SET XACT_ABORT ON` in anything that opens a transaction | Assume an error rolls the whole transaction back | [Upserts](#upserts-and-merge) |
| Temp tables for large intermediate results | Table variables holding thousands of rows | [Temp tables](#temp-tables-and-table-variables) |
| Procedures in migrations with `CREATE OR ALTER` | Edit a procedure by hand on a shared server | [Stored procedures](#stored-procedures) |
| `EnableRetryOnFailure` and the execution strategy for transactions | Let a failover surface as a 500 error | [Retry](#retry-on-transient-errors) |
| `rowversion` for optimistic concurrency | Last write silently wins | [rowversion](#optimistic-concurrency-with-rowversion) |
| Query Store, then the actual plan | Guess which query is slow | [Finding slow queries](#finding-slow-queries) |
| `FULL` recovery with scheduled log backups, `COPY_ONLY` for ad-hoc | `FULL` recovery with no log backups | [Backups](#backups) |
| Strip `GO` before sending a script from code | Send a script with `GO` through EF Core or Dapper | [Translating from Postgres](#translating-from-postgres) |

* * *

## Further reading

1.  **Microsoft Learn — What's new in SQL Server 2025**
    [https://learn.microsoft.com/en-us/sql/sql-server/what-s-new-in-sql-server-2025](https://learn.microsoft.com/en-us/sql/sql-server/what-s-new-in-sql-server-2025)
2.  **Microsoft Learn — Monitor performance by using the Query Store**
    [https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store](https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)
3.  **Microsoft Learn — Parameter Sensitive Plan optimization**
    [https://learn.microsoft.com/en-us/sql/relational-databases/performance/parameter-sensitive-plan-optimization](https://learn.microsoft.com/en-us/sql/relational-databases/performance/parameter-sensitive-plan-optimization)
4.  **Microsoft Learn — SET TRANSACTION ISOLATION LEVEL**
    [https://learn.microsoft.com/en-us/sql/t-sql/statements/set-transaction-isolation-level-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-transaction-isolation-level-transact-sql)
5.  **Microsoft Learn — EF Core connection resiliency**
    [https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency)
6.  **Microsoft Learn — What's happening to Azure Data Studio**
    [https://learn.microsoft.com/en-us/azure-data-studio/whats-happening-azure-data-studio](https://learn.microsoft.com/en-us/azure-data-studio/whats-happening-azure-data-studio)
7.  **MSSQLTips (Aaron Bertrand) — Use caution with SQL Server's MERGE statement**
    [https://www.mssqltips.com/sqlservertip/3074/use-caution-with-sql-servers-merge-statement/](https://www.mssqltips.com/sqlservertip/3074/use-caution-with-sql-servers-merge-statement/)
8.  **dbi services — Why UUIDv7 does not reduce fragmentation in SQL Server**
    [https://www.dbi-services.com/blog/why-uuidv7-does-not-reduce-fragmentation-in-sql-server/](https://www.dbi-services.com/blog/why-uuidv7-does-not-reduce-fragmentation-in-sql-server/)
9.  **Ola Hallengren — SQL Server Maintenance Solution**
    [https://ola.hallengren.com/](https://ola.hallengren.com/)

* * *

## Related pages

- [Database Standards → Which database?](/Database-Standards#which-database) — when SQL Server is the right call instead of Postgres.
- [General Principles](/Database-Standards/General-Principles) — the rules that apply on every engine.
- [Engine Differences](/Database-Standards/Engine-Differences) — behaviour that silently differs from Postgres and MySQL.
- [Coding Standards → Backend — .NET / C#](/Coding-Standards/Backend-DotNet-CSharp) — where EF Core and Dapper code sits in a .NET project.
- [Migrations and Schema Changes](/Database-Standards/Migrations-and-Schema-Changes) — EF Core migrations and DbUp, and changing live tables safely.
- [Backups and Recovery](/Database-Standards/Backups-and-Recovery) — the backup strategy across all engines.
- [Access and Roles](/Database-Standards/Access-and-Roles) — logins, users and least privilege.
