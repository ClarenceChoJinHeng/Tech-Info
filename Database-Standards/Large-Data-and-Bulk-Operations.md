Large Data and Bulk Operations
==============================

> Read **[General Principles](/Database-Standards/General-Principles)** first — this page picks up where [think in sets, not loops](/Database-Standards/General-Principles#think-in-sets-not-loops) leaves off: when the set is so big that one statement causes problems of its own.
> Examples are PostgreSQL. SQL Server and MySQL are covered in [Bulk work in other engines](#bulk-work-in-other-engines).

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [Why batch](#why-one-huge-statement-hurts) | **Past about a hundred thousand rows, change data in batches**<br/><sub>One giant statement holds locks for minutes, floods the WAL, lags replicas, and rolls everything back if it fails at the end.</sub> |
| [Batching](#batch-by-primary-key) | **A few thousand rows per batch, each committed on its own, walked by primary key**<br/><sub>Short locks, steady progress, and other queries get a turn between batches.</sub> |
| [Resumable jobs](#a-resumable-batch-job) | **A batch job can be stopped and restarted without redoing or skipping rows**<br/><sub>Filter on "not done yet", or remember the last ID processed.</sub> |
| [Walking a table](#walking-a-huge-table-from-app-code) | **Keyset (`WHERE id > last`), never OFFSET, to page through a big table**<br/><sub>OFFSET reads and throws away every row before the page, so it gets slower each page.</sub> |
| [Bulk inserts](#loading-data-in-bulk) | **Multi-row inserts for thousands, COPY for millions**<br/><sub>One row per statement is the slowest way to load data there is.</sub> |
| [ORM entities](#bulk-tools-in-each-stack) | **No entity-per-row loops for big jobs**<br/><sub>Use the tool's set-based methods (`updateMany`, `ExecuteUpdate`, `chunkById`) or plain SQL.</sub> |
| [Partitioning](#partitioning-big-tables) | **Partition by date only for very large tables queried and expired by date**<br/><sub>Dropping a month's partition is instant. Deleting a month of rows is not.</sub> |
| [Archiving](#archiving-old-data) | **Move old rows out in batches, inside one transaction per batch**<br/><sub>A row is in the live table or the archive, never both and never neither.</sub> |
| [Heavy reports](#heavy-reports-with-materialized-views) | **Precompute expensive reports in a materialized view and refresh it concurrently**<br/><sub>The dashboard reads a small table instead of aggregating millions of rows per page load.</sub> |
| [Cleaning up](#cleaning-up-after-big-deletes) | **`TRUNCATE` to empty a table, `VACUUM ANALYZE` after a huge delete**<br/><sub>Deleted rows stay on disk until vacuum clears them.</sub> |

* * *

## Concepts you'll meet on this page

### Batch

A slice of a big job ("the next 5,000 rows"), small enough to finish in a second or two, run as its own transaction.

### WAL (write-ahead log)

Postgres writes every change to the WAL first; replicas and point-in-time backups replay it. SQL Server's equivalent is the **transaction log**.

### Keyset iteration

Walking a table by remembering where you stopped (`WHERE id > 48000 ORDER BY id LIMIT 1000`) instead of counting rows to skip (`OFFSET 48000`).

### Dead rows

Old row versions left on disk by Postgres `UPDATE` and `DELETE` until `VACUUM` cleans them up. A big delete leaves a lot of them.

### Partition

A big table split into smaller physical tables (one per month, for example) that queries still see as one table.

* * *

## Why one huge statement hurts

```sql
-- ❌ 12 million expired sessions in one statement
DELETE FROM sessions WHERE expires_at < now() - interval '30 days';
```

| Problem | What actually happens |
| --- | --- |
| **Long locks** | Every deleted row stays locked until the statement commits. Anything that touches those rows waits. |
| **WAL flood** | Gigabytes of WAL in one go: disk fills up, backups grow, and replicas fall minutes behind. |
| **All or nothing** | If it fails at row 11 million (a timeout, a deploy, a lost connection), all of it rolls back, and the rollback takes time too. |
| **No progress** | You can't tell how far it's got, and you can't pause it. |
| **Dead rows** | 12 million dead rows for autovacuum to clean up at once. |

Batching fixes all five, for big `UPDATE`s (backfills) and `INSERT ... SELECT`s too.

> 📝 **Where's the line?** No exact number. Under about a hundred thousand rows on a lightly used table, one statement is usually fine. Above that, or on a table the app writes to constantly, batch it. If in doubt, batch.

* * *

## Batching updates and deletes

### Batch by primary key

Postgres's `DELETE` and `UPDATE` have no `LIMIT`, so pick the batch's IDs in a subquery:

```sql
-- ✅ one batch: run it in a loop until it deletes 0 rows
DELETE FROM sessions
WHERE id IN (
    SELECT id
    FROM   sessions
    WHERE  expires_at < now() - interval '30 days'
    ORDER  BY id
    LIMIT  5000
);

-- ✅ one batch of a backfill
WITH batch AS (
    SELECT id
    FROM   customers
    WHERE  mobile_phone IS NULL
      AND  phone IS NOT NULL
    ORDER  BY id
    LIMIT  5000
)
UPDATE customers AS c
SET    mobile_phone = c.phone
FROM   batch
WHERE  c.id = batch.id;
```

- **Batch size:** start around 1,000–10,000 rows, aiming for well under a second each. Measure, then adjust.
- **Commit each batch** (autocommit, or its own short transaction). Never loop inside one big transaction: that's the giant statement again.
- **Pause between batches** (100–500 ms) so app queries, autovacuum and replicas keep up.
- **The filter must use an index.** Without one on `expires_at`, every batch scans the whole table. Check with `EXPLAIN` (see [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics)).

### A resumable batch job

A batch job must be safe to stop and start again, either by:

1. **Filtering on "not done yet"**, like `mobile_phone IS NULL` above, so a restart carries on.
2. **Remembering the last ID processed**, when "done" can't be read from the row itself.

```ts
// ✅ keyset-driven backfill: resumable, logs progress, gives the database room to breathe
const BATCH_SIZE = 5000;
let lastId = BigInt(process.env.START_AFTER_ID ?? '0');   // resume point after a restart

for (;;) {
  // 1. the next slice of IDs, by primary key
  const batch = await prisma.$queryRaw<{ id: bigint }[]>`
    SELECT id FROM customers
    WHERE  id > ${lastId}
    ORDER  BY id
    LIMIT  ${BATCH_SIZE}`;
  if (batch.length === 0) break;                 // past the end of the table

  // 2. change only the rows in that slice that still need it
  const ids = batch.map((r) => r.id);
  const updated = await prisma.$executeRaw`
    UPDATE customers
    SET    mobile_phone = phone
    WHERE  id = ANY(${ids})
      AND  mobile_phone IS NULL`;

  lastId = ids[ids.length - 1];
  logger.info({ lastId: lastId.toString(), updated }, 'backfill batch done');
  await new Promise((r) => setTimeout(r, 200));
}
```

> 💡 Log the last ID after every batch, so a dead job restarts with `START_AFTER_ID` instead of from the beginning.

### Batching in each tool

| Tool | Set-based or batched way | ❌ Avoid for big jobs |
| --- | --- | --- |
| Prisma | `updateMany` / `deleteMany` for one statement; `$executeRaw` in a loop for batches | `findMany` everything, then `update` each row |
| TypeORM | `createQueryBuilder().update()...execute()` or `.delete()`; raw `query()` in a loop for batches | `find()` everything, change entities, `save()` them |
| Laravel | `chunkById(1000, fn)` or `lazyById()` to walk rows; a query-builder `update()` or `delete()` per chunk | `chunk()` when you're changing the column you filter on: it skips rows. `Model::all()` on a big table. |
| EF Core 7+ | `ExecuteUpdateAsync` / `ExecuteDeleteAsync` (one statement, no entities loaded); `ExecuteSqlAsync` in a loop for batches | Loading entities, changing them, `SaveChanges()` |
| Dapper | Plain SQL; `ExecuteAsync` returns the rows affected, so loop until it's 0 | `ExecuteAsync(sql, listOfObjects)`: it runs the statement once per item |

```php
// Laravel: ✅ walk by ID, one UPDATE per chunk
Customer::whereNull('mobile_phone')
    ->chunkById(5000, function ($customers) {
        Customer::whereIn('id', $customers->pluck('id'))
            ->update(['mobile_phone' => DB::raw('phone')]);
    });
```

```csharp
// EF Core 7+: ✅ one DELETE statement, nothing loaded into memory
await db.Sessions
    .Where(s => s.ExpiresAt < cutoff)
    .ExecuteDeleteAsync();

// ✅ batched with raw SQL: loop until a batch deletes nothing
int deleted;
do
{
    deleted = await db.Database.ExecuteSqlAsync($"""
        DELETE FROM sessions
        WHERE id IN (SELECT id FROM sessions WHERE expires_at < {cutoff} ORDER BY id LIMIT 5000)
        """);
    await Task.Delay(200);
} while (deleted > 0);
```

* * *

## Walking a huge table from app code

For jobs that process every row in app code (an email per customer, an export), page by key, not by offset.

```sql
-- ❌ page 2,000: reads 2,000,000 rows and throws away all but the last 1,000
SELECT id, email FROM customers ORDER BY id LIMIT 1000 OFFSET 1999000;

-- ✅ every page costs the same: jump straight to where the last page ended
SELECT id, email FROM customers WHERE id > $1 ORDER BY id LIMIT 1000;
```

Offset paging also skips or repeats rows if anything is inserted or deleted mid-walk; keyset doesn't. For user-facing lists, see [Writing Queries](/Database-Standards/Writing-Queries).

* * *

## Loading data in bulk

| Rows | Use |
| --- | --- |
| Up to a few thousand | One multi-row `INSERT` (or the ORM's bulk insert) |
| Tens of thousands and up | Multi-row inserts in chunks, or `COPY` |
| Millions, or a CSV file | `COPY` into a staging table, then `INSERT ... SELECT` into the real one |

### Multi-row inserts

```sql
-- ❌ 3 statements, 3 round trips (and in a loop: 10,000 statements, 10,000 round trips)
INSERT INTO products (sku, name, price) VALUES ('A-1', 'Desk lamp', 24.00);
INSERT INTO products (sku, name, price) VALUES ('A-2', 'Desk mat', 12.50);
INSERT INTO products (sku, name, price) VALUES ('A-3', 'Monitor arm', 89.00);

-- ✅ one statement
INSERT INTO products (sku, name, price) VALUES
    ('A-1', 'Desk lamp',   24.00),
    ('A-2', 'Desk mat',    12.50),
    ('A-3', 'Monitor arm', 89.00);
```

> ⚠️ A single Postgres statement can have at most **65,535 bind parameters**. 10,000 rows × 8 columns is 80,000 and fails. Chunk so that rows × columns stays well under the limit, e.g. 1,000–5,000 rows per statement.

### COPY

The fastest way to get rows into Postgres: no per-row statement parsing, and the data streams in.

```sql
-- server-side: the app streams the data over the connection
COPY products (sku, name, price) FROM STDIN WITH (FORMAT csv, HEADER true);

-- from psql, reading a file on YOUR machine (\copy, not COPY)
\copy products (sku, name, price) FROM 'products.csv' WITH (FORMAT csv, HEADER true)
```

- Plain `COPY ... FROM 'file'` reads a file on the **database server** and needs special privileges. From a laptop or app, use `\copy` or `FROM STDIN`.
- `COPY` is all or nothing: one bad row fails the whole load. On Postgres 17+, `ON_ERROR ignore` skips rows that fail type conversion. Otherwise, load into a staging table with `text` columns, then clean and convert in SQL.

**COPY into a staging table, then merge** is the standard import pattern:

```sql
CREATE TEMP TABLE products_import (sku text, name text, price text);

-- \copy products_import FROM 'products.csv' WITH (FORMAT csv, HEADER true)

INSERT INTO products (sku, name, price)
SELECT sku, name, price::numeric(12,2)
FROM   products_import
WHERE  sku IS NOT NULL
ON CONFLICT (sku) DO UPDATE
SET    name = EXCLUDED.name,
       price = EXCLUDED.price;
```

### Bulk upserts with unnest

Pass **one array per column** and `unnest` them instead of building a giant `VALUES` list. It's three parameters whatever the row count, so the 65,535 limit doesn't apply.

```sql
INSERT INTO products (sku, name, price)
SELECT *
FROM   unnest($1::text[], $2::text[], $3::numeric[])
ON CONFLICT (sku) DO UPDATE
SET    name  = EXCLUDED.name,
       price = EXCLUDED.price;
```

### Bulk tools in each stack

```ts
// Prisma: ✅ one INSERT; skipDuplicates adds ON CONFLICT DO NOTHING (not on SQL Server)
await prisma.product.createMany({ data: products, skipDuplicates: true });

// TypeORM: ✅ insert in chunks (insert() skips the select-per-entity that save() does)
for (let i = 0; i < products.length; i += 2000) {
  await productRepo.insert(products.slice(i, i + 2000));
}

// node-postgres + pg-copy-streams: ✅ stream a CSV straight into COPY
import { pipeline } from 'node:stream/promises';
import { createReadStream } from 'node:fs';
import { from as copyFrom } from 'pg-copy-streams';

const client = await pool.connect();
try {
  const ingest = client.query(
    copyFrom('COPY products_import (sku, name, price) FROM STDIN WITH (FORMAT csv, HEADER true)'),
  );
  await pipeline(createReadStream('products.csv'), ingest);
} finally {
  client.release();
}
```

```php
// Laravel: ✅ multi-row insert and upsert, chunked
foreach (array_chunk($rows, 2000) as $chunk) {
    Product::upsert($chunk, ['sku'], ['name', 'price']);   // rows, unique-by columns, columns to update
}
```

```csharp
// Npgsql binary COPY: ✅ the fastest way to load from .NET
await using var writer = await conn.BeginBinaryImportAsync(
    "COPY products (sku, name, price) FROM STDIN (FORMAT BINARY)");

foreach (var p in products)
{
    await writer.StartRowAsync();
    await writer.WriteAsync(p.Sku, NpgsqlDbType.Text);
    await writer.WriteAsync(p.Name, NpgsqlDbType.Text);
    await writer.WriteAsync(p.Price, NpgsqlDbType.Numeric);
}
await writer.CompleteAsync();   // without this, nothing is saved

// Dapper + Npgsql: ✅ arrays go through as Postgres arrays, so unnest works
await conn.ExecuteAsync("""
    INSERT INTO products (sku, name, price)
    SELECT * FROM unnest(@Skus, @Names, @Prices)
    ON CONFLICT (sku) DO UPDATE SET name = EXCLUDED.name, price = EXCLUDED.price
    """,
    new { Skus = skus, Names = names, Prices = prices });   // string[], string[], decimal[]
```

> 💡 **First load into an empty table?** Create indexes and foreign keys after the data is in. Building an index once is much faster than updating it a million times.

* * *

## Partitioning big tables

### When partitioning pays off

Partitioning adds work: every unique key must include the partition column, and someone has to create next month's partition before next month starts.

| Partition when | Don't partition when |
| --- | --- |
| The table is very large (tens of millions of rows and up) **and** growing steadily | It's big but a good index already makes the queries fast |
| Almost every query filters on the same column, usually a date | Queries rarely filter on the partition column. Every query would have to check every partition. |
| Old data is deleted or archived by age (logs, audit events, readings) | You just want it to "go faster". Check [Indexing](/Database-Standards/Indexing) and the query plan first. |

### Range partitioning by date

```sql
CREATE TABLE audit_events (
    id           bigint GENERATED ALWAYS AS IDENTITY,
    occurred_at  timestamptz NOT NULL,
    user_id      bigint,
    action       text NOT NULL,
    payload      jsonb,
    PRIMARY KEY (id, occurred_at)          -- must include the partition column
) PARTITION BY RANGE (occurred_at);

CREATE TABLE audit_events_2026_11 PARTITION OF audit_events
    FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');
CREATE TABLE audit_events_2026_12 PARTITION OF audit_events
    FOR VALUES FROM ('2026-12-01') TO ('2027-01-01');

CREATE INDEX ON audit_events (user_id, occurred_at);   -- created on every partition
```

- **Partition pruning:** `WHERE occurred_at >= '2026-11-01'` reads only the matching partitions. A query without the partition column reads all of them.
- **Create partitions ahead of time.** An insert with no matching partition fails. Use a scheduled job or [pg_partman](https://github.com/pgpartman/pg_partman) to keep the next few months ready.
- Identity columns on a partitioned table need **Postgres 17+**. On older versions use `bigserial` for the `id`.

### Dropping old data by partition

```sql
-- ❌ a month of audit events: millions of row deletes, WAL, dead rows, vacuum
DELETE FROM audit_events WHERE occurred_at < '2025-12-01';

-- ✅ instant: detach the old month, then drop it (or dump it to cold storage first)
ALTER TABLE audit_events DETACH PARTITION audit_events_2025_11 CONCURRENTLY;
DROP TABLE audit_events_2025_11;
```

`DETACH PARTITION ... CONCURRENTLY` (Postgres 14+) doesn't block queries on the parent. It can't run inside a transaction, or on a table with a default partition.

* * *

## Archiving old data

Move rows that must be kept but not live to an archive table (or cold storage) in batches. Copy and delete **in the same statement or transaction**, so a row is never in both places, or neither.

```sql
CREATE TABLE orders_archive (LIKE orders INCLUDING DEFAULTS);
ALTER TABLE orders_archive ADD PRIMARY KEY (id);

-- ✅ one batch: delete from orders and insert into the archive in one atomic statement
WITH moved AS (
    DELETE FROM orders
    WHERE id IN (
        SELECT id
        FROM   orders
        WHERE  status IN ('delivered', 'cancelled')
          AND  created_at < now() - interval '2 years'
        ORDER  BY id
        LIMIT  5000
    )
    RETURNING *
)
INSERT INTO orders_archive
SELECT * FROM moved;
```

- Child rows first: archive `order_items` for those orders in the same transaction, or the foreign key stops the delete.
- `LIKE orders` copies the column order, which is why `SELECT *` lines up. If the tables drift apart, list the columns explicitly.
- For large time-based data, [partitioning](#partitioning-big-tables) turns archiving into detaching a partition.

* * *

## Heavy reports with materialized views

A **materialized view** stores an expensive aggregate's result, refreshed on a schedule, so a dashboard doesn't aggregate millions of rows on every page load.

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT date_trunc('month', paid_at) AS month,
       sum(amount)                  AS revenue,
       count(*)                     AS payment_count
FROM   payments
GROUP  BY 1;

CREATE UNIQUE INDEX ON monthly_revenue (month);   -- required for CONCURRENTLY

-- ✅ readers keep seeing the old data while it refreshes
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

- Without `CONCURRENTLY`, a refresh locks the view and every reader waits.
- Data is only as fresh as the last refresh. Show "updated at" on the dashboard, and schedule the refresh (an app scheduler, or `pg_cron` where available).

* * *

## Cleaning up after big deletes

### VACUUM and ANALYZE

After deleting or updating a large share of a table:

```sql
VACUUM (ANALYZE) sessions;
```

- `VACUUM` marks dead rows' space as reusable; `ANALYZE` refreshes the now-wrong planner statistics. Autovacuum does both eventually, but running it yourself stops the planner working from stale numbers meanwhile.
- Plain `VACUUM` doesn't return space to the OS, and doesn't need to: the table reuses it. **Avoid `VACUUM FULL` on a live table.** It rewrites the table under an ACCESS EXCLUSIVE lock. If you genuinely need the disk space back, use `pg_repack`.

### TRUNCATE versus DELETE

| | `DELETE FROM t` | `TRUNCATE t` |
| --- | --- | --- |
| Speed on a big table | Slow: row by row, WAL for each | Instant |
| Can filter with WHERE | Yes | No, it empties the whole table |
| Lock | Row locks | ACCESS EXCLUSIVE: nobody can read the table until it commits |
| Row triggers | Fire | Don't fire |
| Table referenced by a foreign key | Works if no child rows exist | Fails unless the child tables are truncated too (`CASCADE`) |
| Resets the identity counter | No | With `RESTART IDENTITY` |

Both can be rolled back in Postgres. `TRUNCATE` is for emptying staging, import and scratch tables, never for "delete most of it".

* * *

## Bulk work in other engines

| Engine | Batching | Bulk loading | Partitioning |
| --- | --- | --- | --- |
| **SQL Server** | `DELETE TOP (4000) FROM dbo.sessions WHERE expires_at < @cutoff;` in a `WHILE` loop until `@@ROWCOUNT = 0`. Keep batches under ~5,000 rows: around 5,000 locks in one statement, SQL Server escalates to a **table lock**. In the FULL recovery model, the transaction log only shrinks after log backups, so a huge job can fill the disk. | `SqlBulkCopy` from .NET, `BULK INSERT` or `bcp` for files | Partition functions and schemes. `ALTER TABLE ... SWITCH PARTITION` moves a whole partition out instantly. |
| **MySQL** | `DELETE ... ORDER BY id LIMIT 5000` works directly (no subquery needed) | `LOAD DATA [LOCAL] INFILE` (`local_infile` must be enabled on both client and server) | Partitioned InnoDB tables **can't have foreign keys**, and every unique key must include the partition columns. |

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Batch big updates and deletes, committing each batch | Run one statement over millions of rows | [Why batch](#why-one-huge-statement-hurts) |
| Pick batch IDs with `ORDER BY id LIMIT n` in a subquery | Loop inside one huge transaction | [Batch by primary key](#batch-by-primary-key) |
| Make batch jobs resumable, and log progress | Start over from row 1 after every failure | [Resumable jobs](#a-resumable-batch-job) |
| Use `updateMany`, `ExecuteUpdate`, `chunkById` or plain SQL | Load entities one by one and save them | [Each tool](#batching-in-each-tool) |
| Walk big tables by key (`WHERE id > $1`) | Page with OFFSET | [Walking a table](#walking-a-huge-table-from-app-code) |
| Multi-row inserts chunked under 65,535 parameters | One INSERT per row, or one giant VALUES list | [Multi-row inserts](#multi-row-inserts) |
| `COPY` into a staging table, then merge | Import a million-row CSV with an ORM loop | [COPY](#copy) |
| Send arrays and `unnest` them for bulk upserts | Build huge SQL strings by hand | [unnest](#bulk-upserts-with-unnest) |
| Partition only very large, date-driven tables | Partition to fix a missing index | [Partitioning](#when-partitioning-pays-off) |
| Detach and drop old partitions | `DELETE` a month of rows from a partitioned table | [Dropping by partition](#dropping-old-data-by-partition) |
| Move rows to the archive with `DELETE ... RETURNING` in one statement | Copy in one job and delete in another | [Archiving](#archiving-old-data) |
| Precompute heavy reports and `REFRESH ... CONCURRENTLY` | Aggregate millions of rows on every dashboard load | [Materialized views](#heavy-reports-with-materialized-views) |
| `VACUUM (ANALYZE)` after a big delete | `VACUUM FULL` on a live table | [VACUUM](#vacuum-and-analyze) |

* * *

## Further reading

1.  **PostgreSQL Documentation — Populating a Database** (fast bulk loading)
    [https://www.postgresql.org/docs/current/populate.html](https://www.postgresql.org/docs/current/populate.html)
2.  **PostgreSQL Documentation — COPY**
    [https://www.postgresql.org/docs/current/sql-copy.html](https://www.postgresql.org/docs/current/sql-copy.html)
3.  **PostgreSQL Documentation — Table Partitioning**
    [https://www.postgresql.org/docs/current/ddl-partitioning.html](https://www.postgresql.org/docs/current/ddl-partitioning.html)
4.  **PostgreSQL Documentation — Routine Vacuuming**
    [https://www.postgresql.org/docs/current/routine-vacuuming.html](https://www.postgresql.org/docs/current/routine-vacuuming.html)
5.  **Use The Index, Luke — Paging Through Results** (why OFFSET gets slow)
    [https://use-the-index-luke.com/no-offset](https://use-the-index-luke.com/no-offset)
6.  **Microsoft Learn — ExecuteUpdate and ExecuteDelete (EF Core)**
    [https://learn.microsoft.com/en-us/ef/core/saving/execute-insert-update-delete](https://learn.microsoft.com/en-us/ef/core/saving/execute-insert-update-delete)
7.  **Npgsql — COPY**
    [https://www.npgsql.org/doc/copy.html](https://www.npgsql.org/doc/copy.html)
8.  **Laravel — Eloquent: Chunking Results**
    [https://laravel.com/docs/eloquent#chunking-results](https://laravel.com/docs/eloquent#chunking-results)
9.  **pg_partman** — automatic creation and retention of time-based partitions
    [https://github.com/pgpartman/pg_partman](https://github.com/pgpartman/pg_partman)

* * *

## Related pages

- [General Principles](/Database-Standards/General-Principles) — think in sets, and the rules for touching production by hand.
- [Migrations and Schema Changes](/Database-Standards/Migrations-and-Schema-Changes) — backfills as part of expand and contract.
- [Indexing](/Database-Standards/Indexing) — the index every batch filter needs.
- [Writing Queries](/Database-Standards/Writing-Queries) — keyset pagination for user-facing lists.
- [Replication and Monitoring](/Database-Standards/Replication-and-Monitoring) — watching replica lag while a big job runs.
