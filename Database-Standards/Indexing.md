Indexing
========

> Read **[General Principles](/Database-Standards/General-Principles)** first. This page covers which index to create for which query, and how to prove it's being used.
> Examples are PostgreSQL. SQL Server and MySQL differences are in [Other engines](#other-engines).

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [Cost](#what-an-index-costs) | **Every index speeds up some reads and slows down every write**<br/><sub>An index has to earn its place. "Just in case" indexes are pure cost.</sub> |
| [Foreign keys](#index-every-foreign-key) | **Index every foreign key column yourself**<br/><sub>Postgres does not do it for you. Joins and cascading deletes scan the whole table without it.</sub> |
| [When to index](#when-to-index-and-when-not-to) | **Index the columns your real queries filter, join and sort on**<br/><sub>Start from the query, never from the table.</sub> |
| [Composite order](#composite-indexes-and-column-order) | **Equality columns first, then the range or sort column**<br/><sub>A composite index is sorted left to right, like a phone book by last name then first name.</sub> |
| [Sorting](#indexes-that-match-order-by) | **An index can hand back rows already sorted**<br/><sub>Match the index to `ORDER BY` and `LIMIT` queries stop sorting the whole table.</sub> |
| [Covering](#covering-indexes-with-include) | **`INCLUDE` the extra columns a hot query reads**<br/><sub>The query is answered from the index alone, without visiting the table.</sub> |
| [Partial](#partial-indexes) | **Index only the rows you query**<br/><sub>`WHERE deleted_at IS NULL` keeps the index small and makes "unique among active rows" possible.</sub> |
| [Expressions](#expression-indexes) | **Index `lower(email)` if you query `lower(email)`**<br/><sub>A function on a column hides it from a plain index.</sub> |
| [Index types](#beyond-b-tree) | **B-tree by default. GIN, GiST and BRIN for the jobs B-tree can't do**<br/><sub>JSONB, full-text, `LIKE '%x%'`, ranges and huge time-ordered tables.</sub> |
| [Live tables](#creating-indexes-on-live-tables) | **`CREATE INDEX CONCURRENTLY` on any table that's in use**<br/><sub>A plain `CREATE INDEX` blocks every write to the table until it finishes.</sub> |
| [Housekeeping](#finding-unused-and-missing-indexes) | **Check for unused, duplicate and missing indexes regularly**<br/><sub>The statistics views tell you which indexes nobody uses and which tables get scanned.</sub> |

* * *

## Concepts you'll meet on this page

### Selectivity

How much an index narrows things down. `email` (one value, one row) is highly selective; `status` with four values isn't. Indexes help most on selective conditions.

### Sequential scan

Reading every row of the table. Often the fastest option on a small table; a problem on millions of rows when the query returns a handful.

### Index-only scan

Answering the query from the index alone, without visiting the table. Only possible when the index contains every column the query needs.

### Write amplification

Every `INSERT`, and every `UPDATE` that changes an indexed column, must update each relevant index too. Ten indexes can make writes several times slower.

* * *

## What an index costs

An index is a second, sorted copy of some of your data:

| Cost | What it means |
| --- | --- |
| **Slower writes** | Each index is updated on every insert, and on updates that change its columns. |
| **Storage and memory** | Indexes take disk space and compete with table data for the cache. On busy tables they're often bigger than the table. |
| **Fewer HOT updates** | Postgres can update a row without touching any index (a *heap-only tuple* update) only if no changed column is indexed. Index a constantly updated column like `last_seen_at` and every update touches every index. |

So for every index ask: **which query is this for, and how often does it run?**

* * *

## When to index and when not to

### Index every foreign key

Postgres automatically indexes a `PRIMARY KEY` and every `UNIQUE` constraint, but **not** a foreign key.

```sql
-- the foreign key on its own: no index on invoice_lines.invoice_id
ALTER TABLE invoice_lines
    ADD CONSTRAINT invoice_lines_invoice_id_fkey
    FOREIGN KEY (invoice_id) REFERENCES invoices (id) ON DELETE CASCADE;

-- ✅ add it yourself
CREATE INDEX invoice_lines_invoice_id_idx ON invoice_lines (invoice_id);
```

Without it, loading an invoice's lines scans all of `invoice_lines`, and so does deleting one invoice (Postgres has to find the lines to cascade to, or check that none exist).

| Tool | Creates the FK index for you? |
| --- | --- |
| EF Core | **Yes**, by convention, for every foreign key |
| Prisma | No. Add `@@index([invoiceId])` |
| TypeORM | No. Add `@Index()` on the column |
| Laravel `foreignId()->constrained()` | No on Postgres. Add `->index()` (MySQL creates one automatically) |
| Dapper / hand-written SQL | No |

Foreign keys with no index on their first column:

```sql
SELECT c.conrelid::regclass AS table_name,
       c.conname            AS foreign_key,
       a.attname            AS column_name
FROM   pg_constraint AS c
JOIN   pg_attribute  AS a ON a.attrelid = c.conrelid AND a.attnum = c.conkey[1]
WHERE  c.contype = 'f'
  AND  NOT EXISTS (
         SELECT 1
         FROM   pg_index AS i
         WHERE  i.indrelid = c.conrelid
           AND  i.indkey[0] = c.conkey[1]
       );
```

> 📝 This checks the first column of each foreign key, which covers the usual single-column case.

### Good candidates

| Index it | Example |
| --- | --- |
| Foreign keys | `invoices.customer_id`, `invoice_lines.invoice_id` |
| Columns in a selective `WHERE` | `users.email`, `orders.reference` |
| Columns you `JOIN` on that aren't keys already | `payments.provider_charge_id` |
| `ORDER BY` columns of paged lists | `invoices (customer_id, created_at DESC)` |
| Columns behind a business rule "must be unique" | `invoices.number`, as a `UNIQUE` constraint |

### Leave it out

| Don't index | Why |
| --- | --- |
| Small tables (a few thousand rows) | A sequential scan is already fast. The planner will often ignore the index anyway. |
| A low-selectivity column on its own (`status`, `is_active`) | If a value matches a big share of the table, reading the table is faster. Use it as the second column of a composite, or as a [partial index](#partial-indexes) for the rare value. |
| Write-heavy tables with few reads (logs, events) | You pay on every insert for queries that rarely run. |
| A column that's already the leading column of another index | `(customer_id)` is redundant next to `(customer_id, created_at)`. |
| Every column, "just in case" | Each one slows down every write, for a query that may never exist. |

* * *

## Composite indexes and column order

A composite index is sorted by its first column, then by the second within that, and so on. Column order decides which queries it can serve.

```sql
CREATE INDEX invoices_customer_id_status_created_at_idx
    ON invoices (customer_id, status, created_at);
```

| Query filters on | Uses the index well? |
| --- | --- |
| `customer_id = 42` | ✅ Leftmost column |
| `customer_id = 42 AND status = 'paid'` | ✅ Leftmost two |
| `customer_id = 42 AND status = 'paid' AND created_at >= '2026-01-01'` | ✅ All three |
| `customer_id = 42 AND created_at >= '2026-01-01'` | ⚠️ Partly. Finds the customer, then has to step through every status |
| `status = 'paid'` | ❌ Not efficiently. The leading column is missing |

**The rule: equality columns first, then the one range or sort column.** Columns after a range (`>=`, `<`, `BETWEEN`) or sort column can't narrow the search any further.

```sql
-- query: one customer's paid invoices in a date range
WHERE customer_id = 42 AND status = 'paid' AND created_at >= now() - interval '90 days'

-- ✅ equality, equality, range
CREATE INDEX ... ON invoices (customer_id, status, created_at);

-- ❌ range column first: every row in the date range is scanned, for every customer
CREATE INDEX ... ON invoices (created_at, customer_id, status);
```

> 📝 **PostgreSQL 18 added B-tree skip scan:** it can sometimes use a composite index without the leading column, by jumping through each distinct value of it. It only pays off when that column has few distinct values, and the planner decides when. Don't design around it. Put the columns in the right order.

> 💡 **Correcting the old guidance:** the order of conditions in a `WHERE` clause doesn't matter. `WHERE status = 'paid' AND customer_id = 42` uses the same index as the reverse. Only the column order in the **index** matters.

* * *

## Indexes that match ORDER BY

A B-tree is already sorted, so it can return rows in order and let `LIMIT` stop early, with no sort step.

```sql
-- the latest 20 invoices for a customer
SELECT id, number, total, created_at
FROM   invoices
WHERE  customer_id = 42
ORDER  BY created_at DESC
LIMIT  20;

-- ✅ reads 20 index entries and stops
CREATE INDEX invoices_customer_id_created_at_idx ON invoices (customer_id, created_at DESC);
```

- A single-column index can be read **backwards**, so `(created_at)` serves both `ASC` and `DESC`.
- **Mixed directions** (`ORDER BY priority DESC, created_at ASC`) need an index declared with those directions.
- Postgres sorts `NULL`s **last** in `ASC` and **first** in `DESC`. If the query says `NULLS LAST` with `DESC`, declare the index the same way: `(due_date DESC NULLS LAST)`.

This is also what makes keyset pagination fast. See [Writing Queries](/Database-Standards/Writing-Queries).

* * *

## Covering indexes with INCLUDE

If a query reads only columns in the index, Postgres can use an **index-only scan** and skip the table. Since PostgreSQL 11, `INCLUDE` adds columns to the index entries without making them part of the sort key:

```sql
-- the query: a customer's invoice list, showing number and total
SELECT number, total
FROM   invoices
WHERE  customer_id = 42;

-- ✅ search by customer_id, carry number and total along
CREATE INDEX invoices_customer_id_covering_idx
    ON invoices (customer_id) INCLUDE (number, total);
```

**Correcting the old guidance:** the old document listed every column in the key, `(customer_id, order_date, total_amount)`. That works, but makes the key bigger and sorts on columns that are only read. Use `INCLUDE` for columns you read but don't filter or sort on.

> ⚠️ An index-only scan only skips the table for pages that **VACUUM** has marked all-visible. On a table with many recent updates, the plan says "Index Only Scan" but still shows many `Heap Fetches`. Autovacuum normally keeps up. See [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics).

Keep covering indexes for **hot, read-heavy queries** (dashboards, list screens, reports): every included column makes the index bigger.

* * *

## Partial indexes

A partial index only contains rows matching its `WHERE`: smaller, faster to update, and able to express rules a normal index can't.

```sql
-- soft-deleted rows are never queried, so leave them out of the index
CREATE INDEX invoices_customer_id_active_idx
    ON invoices (customer_id)
    WHERE deleted_at IS NULL;

-- the rare value of a low-selectivity column: 2% of orders are pending
CREATE INDEX orders_pending_created_at_idx
    ON orders (created_at)
    WHERE status = 'pending';
```

**Unique among active rows** is the most useful pattern. A plain `UNIQUE (email)` stops a user re-registering after their account was soft-deleted; a partial unique index doesn't:

```sql
CREATE UNIQUE INDEX users_email_active_key
    ON users (lower(email))
    WHERE deleted_at IS NULL;
```

> ⚠️ The planner only uses a partial index when it can prove the query's `WHERE` implies the index's `WHERE`. `WHERE deleted_at IS NULL AND customer_id = 42` matches. A query that leaves out `deleted_at IS NULL` doesn't use it.

* * *

## Expression indexes

A function around a column hides it from a plain index. Either rewrite the condition so the column stands alone (see [Writing Queries](/Database-Standards/Writing-Queries)), or index the expression.

```sql
-- the query: case-insensitive login
SELECT id FROM users WHERE lower(email) = lower($1);

-- ❌ an index on email is not used for lower(email)
CREATE INDEX users_email_idx ON users (email);

-- ✅ index exactly the expression the query uses
CREATE INDEX users_lower_email_idx ON users (lower(email));
```

The query must use the **same expression**: `lower(email)` won't use an index on `upper(email)`.

> 💡 **Alternative for case-insensitive text: `citext`.** A column type that compares case-insensitively, so a plain index and a plain `UNIQUE` just work:
> ```sql
> CREATE EXTENSION IF NOT EXISTS citext;
> ALTER TABLE users ALTER COLUMN email TYPE citext;
> ```
> Pick one approach per project. Mixing `lower()` in some queries and `citext` in others is how duplicates slip through.

* * *

## Beyond B-tree

B-tree is the default and right for `=`, `<`, `>`, `BETWEEN`, `IN`, `ORDER BY` and prefix `LIKE 'abc%'`.

| Type | Use it for | Example |
| --- | --- | --- |
| **B-tree** (default) | Equality, ranges, sorting | Almost everything |
| **GIN** | Values that contain many items: JSONB, arrays, full-text, trigrams | `metadata @> '{"source":"web"}'`, `tags @> ARRAY['urgent']` |
| **GiST** | Ranges, geometry, "overlaps" and nearest-neighbour searches | No two appointments overlap |
| **BRIN** | Huge tables where rows are physically in time order | Append-only event logs, audit trails |
| **Hash** | Equality only | Rarely worth it. B-tree does the same job and more. |

### Full-text search

**Correcting the old guidance:** `MATCH ... AGAINST` (MySQL) and `CONTAINS` (SQL Server) don't exist in Postgres. Use built-in full-text search with `tsvector` and a GIN index:

```sql
ALTER TABLE products
    ADD COLUMN search tsvector
    GENERATED ALWAYS AS (
        to_tsvector('english', coalesce(name, '') || ' ' || coalesce(description, ''))
    ) STORED;

CREATE INDEX products_search_idx ON products USING gin (search);

SELECT id, name
FROM   products
WHERE  search @@ websearch_to_tsquery('english', 'laptop cooling pad');
```

Generated columns need PostgreSQL 12+. `websearch_to_tsquery` accepts what users type into a search box, including quotes and `-word`.

### LIKE with a leading wildcard

A B-tree can't help `LIKE '%cooling%'`, because the match could start anywhere. `pg_trgm` can:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX customers_name_trgm_idx ON customers USING gin (name gin_trgm_ops);

-- ✅ both can now use the index
SELECT id, name FROM customers WHERE name ILIKE '%smith%';
SELECT id, name FROM customers WHERE name % 'jon smith';   -- similarity: catches typos
```

Full-text search for words in documents; trigrams for "contains" search on short fields like names, codes and emails.

### No overlapping appointments

A GiST **exclusion constraint** makes the database refuse overlapping bookings for the same doctor. No race condition in the app can get past it.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;   -- lets GiST handle the plain "=" on doctor_id

ALTER TABLE appointments
    ADD CONSTRAINT appointments_no_overlap
    EXCLUDE USING gist (
        doctor_id WITH =,
        tstzrange(starts_at, ends_at) WITH &&
    );
```

`tstzrange` includes the start and excludes the end by default, so 10:00–10:30 and 10:30–11:00 don't conflict.

### BRIN for huge, time-ordered tables

BRIN stores only the minimum and maximum value per block of the table: tiny (often under 1% of a B-tree), but only useful when the column's order matches the rows' physical order, typically `created_at` on an append-only table.

```sql
CREATE INDEX audit_log_created_at_brin ON audit_log USING brin (created_at);
```

Heavy updates or deletes break the physical order and BRIN stops helping. Use a B-tree there.

* * *

## Unique indexes and unique constraints

In Postgres both create the same unique B-tree index. Prefer the **constraint** for plain business rules: foreign keys and `ON CONFLICT (...)` refer to it, and it reads as a rule in the schema.

```sql
-- ✅ a business rule: invoice numbers are unique
ALTER TABLE invoices ADD CONSTRAINT invoices_number_key UNIQUE (number);
```

Use a **unique index** when a constraint can't express it: a `WHERE` (partial) or an expression like `lower(email)`.

> 📝 Postgres treats `NULL`s as distinct by default, so a `UNIQUE` column can hold many `NULL`s. Since PostgreSQL 15, `UNIQUE NULLS NOT DISTINCT` allows only one.

* * *

## Creating indexes on live tables

A plain `CREATE INDEX` blocks every `INSERT`, `UPDATE` and `DELETE` until it finishes: on a big table, minutes of failed or queued requests. On any table in use, build it concurrently:

```sql
-- ✅ writes carry on while the index builds
CREATE INDEX CONCURRENTLY invoices_customer_id_created_at_idx
    ON invoices (customer_id, created_at DESC);
```

1. **It can't run inside a transaction block.** Most migration tools wrap each migration in one, so turn that off for this migration (table below).
2. **If it fails, it leaves an invalid index behind**: not used for reads, but still slowing writes. Find it, drop it, and rerun the migration:
   ```sql
   SELECT indexrelid::regclass AS index_name
   FROM   pg_index
   WHERE  NOT indisvalid;

   DROP INDEX CONCURRENTLY invoices_customer_id_created_at_idx;
   ```
3. **It's slower than a normal build**: it scans the table twice and waits for running transactions to finish. That's the trade for not blocking anyone.

| Tool | How to run it outside a transaction |
| --- | --- |
| Prisma | `prisma migrate dev --create-only`, then make `CREATE INDEX CONCURRENTLY ...` the **only statement** in that `migration.sql`. A file with several statements is sent as one implicit transaction. |
| TypeORM | Set `transaction = false` on the migration class, and run migrations with `--transaction each` |
| Laravel | Set `public $withinTransaction = false;` on the migration and use `DB::statement('CREATE INDEX CONCURRENTLY ...')` |
| EF Core | `migrationBuilder.Sql("CREATE INDEX CONCURRENTLY ...", suppressTransaction: true);` |
| DbUp | Run that script without a transaction |

How this fits into a zero-downtime release: [Migrations and Schema Changes](/Database-Standards/Migrations-and-Schema-Changes).

* * *

## Defining indexes in each tool

```prisma
// Prisma
model Invoice {
  id         BigInt   @id @default(autoincrement())
  customerId BigInt   @map("customer_id")
  number     String   @unique
  status     String
  createdAt  DateTime @default(now()) @map("created_at") @db.Timestamptz

  @@index([customerId])                        // the FK index
  @@index([customerId, status, createdAt])     // composite
  @@map("invoices")
}
```

```ts
// TypeORM
@Entity('invoices')
@Index(['customerId', 'status', 'createdAt'])
@Index('invoices_number_active_key', ['number'], { unique: true, where: '"deleted_at" IS NULL' })
export class Invoice {
  @Index()                              // the FK index
  @Column({ name: 'customer_id', type: 'bigint' })
  customerId: string;
  // ...
}
```

```php
// Laravel
Schema::table('invoices', function (Blueprint $table) {
    $table->foreignId('customer_id')->index()->constrained();   // ->index() matters on Postgres
    $table->index(['customer_id', 'status', 'created_at']);
    $table->unique('number');
});
```

```csharp
// EF Core (Npgsql)
modelBuilder.Entity<Invoice>(e =>
{
    e.HasIndex(i => new { i.CustomerId, i.Status, i.CreatedAt });
    e.HasIndex(i => i.Number).IsUnique().HasFilter("deleted_at IS NULL");
    e.HasIndex(i => i.CustomerId).IncludeProperties(i => new { i.Number, i.Total });
});

modelBuilder.Entity<Customer>()
    .HasIndex(c => c.Name)
    .HasMethod("gin")
    .HasOperators("gin_trgm_ops");   // needs the pg_trgm extension (HasPostgresExtension("pg_trgm"))
```

> ⚠️ **Prisma, partial and expression indexes:** Prisma 7.4 added partial indexes (`where:` on `@@index` / `@@unique`) as the `partialIndexes` preview feature. On older versions, and for expression indexes, write the index as raw SQL in a `--create-only` migration. Prisma doesn't know about indexes it can't see in the schema, so read every migration it generates afterwards and remove any `DROP INDEX` for them.

* * *

## Finding unused and missing indexes

### Unused indexes

```sql
SELECT s.relname                                      AS table_name,
       s.indexrelname                                 AS index_name,
       s.idx_scan                                     AS times_used,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM   pg_stat_user_indexes AS s
JOIN   pg_index             AS i ON i.indexrelid = s.indexrelid
WHERE  s.idx_scan = 0
  AND  NOT i.indisunique          -- unique indexes enforce rules even if never read
ORDER  BY pg_relation_size(s.indexrelid) DESC;
```

Before dropping anything it lists:

- **Check how long statistics have been collecting:** `SELECT stats_reset FROM pg_stat_database WHERE datname = current_database();` After a reset last week, a monthly report's index shows zero uses.
- **Check every replica.** Counters are per server; an index unused on the primary may be what the read replica's reports depend on.
- Since PostgreSQL 16, `pg_stat_user_indexes.last_idx_scan` shows when each index was last used.

### Duplicate and redundant indexes

Look for two indexes on the same columns, and for a single-column index that's also the leading column of a composite: `(customer_id)` next to `(customer_id, created_at)` is pure write cost.

### Tables that need an index

```sql
SELECT relname      AS table_name,
       seq_scan,
       seq_tup_read,
       idx_scan,
       n_live_tup   AS approx_rows
FROM   pg_stat_user_tables
WHERE  n_live_tup > 100000
ORDER  BY seq_tup_read DESC
LIMIT  20;
```

A big table with many sequential scans and a huge `seq_tup_read` is being read end to end, repeatedly. Find the responsible query with `pg_stat_statements` (see [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics)) and index for it.

### Index bloat

Indexes on tables with heavy updates and deletes accumulate dead space. Since PostgreSQL 12 you can rebuild one without blocking writes (like `CREATE INDEX CONCURRENTLY`, not inside a transaction block):

```sql
REINDEX INDEX CONCURRENTLY invoices_customer_id_created_at_idx;
```

* * *

## Other engines

| | PostgreSQL | SQL Server | MySQL (InnoDB) |
| --- | --- | --- | --- |
| Table storage | Heap. Every index points to the row. | Usually a **clustered index**, normally the primary key: the table *is* that index. Other indexes are **nonclustered**. | Always clustered on the **primary key**. Every secondary index stores the PK value, so a wide PK makes every index bigger. |
| FK indexes | Create them yourself | Create them yourself | Created automatically if missing |
| Covering columns | `INCLUDE (...)` | `INCLUDE (...)` | No `INCLUDE`. Add the columns to the key; secondary indexes already carry the PK |
| Partial indexes | `WHERE ...` | **Filtered index**: `WHERE ...`, with restrictions on the expressions allowed | Not supported |
| Expression indexes | `(lower(email))` | Index a computed column | Functional key parts since **8.0.13**: `((lower(email)))` |
| Build without blocking writes | `CREATE INDEX CONCURRENTLY` | `WITH (ONLINE = ON)`, Enterprise edition and Azure SQL only | Online DDL, the default for most index changes: `ALGORITHM=INPLACE, LOCK=NONE` |
| Test dropping an index | — | Disable it (`ALTER INDEX ... DISABLE`) | **Invisible index** (8.0+): `ALTER TABLE ... ALTER INDEX ... INVISIBLE` |
| Long text columns | Index whole values, or a trigram index | Key size limits apply | **Prefix index** on the first N characters: `(description(50))` |

Engine-specific detail: [SQL Server](/Database-Standards/SQL-Server), [MySQL](/Database-Standards/MySQL).

* * *

## Do / Don't

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Index every foreign key column | Assume Postgres indexed it for you | [Foreign keys](#index-every-foreign-key) |
| Start from a real query, then design its index | Index every column "just in case" | [When to index](#when-to-index-and-when-not-to) |
| Equality columns first, then range or sort | Lead a composite with a range column | [Composite order](#composite-indexes-and-column-order) |
| Match the index to `ORDER BY ... LIMIT` | Sort the whole table to return 20 rows | [Sorting](#indexes-that-match-order-by) |
| `INCLUDE` read-only columns for hot queries | Add read-only columns to the sort key | [Covering](#covering-indexes-with-include) |
| Partial unique index for "unique among active rows" | Plain `UNIQUE` on a soft-deleted table | [Partial](#partial-indexes) |
| Index the exact expression the query uses, or use `citext` | Expect an index on `email` to serve `lower(email)` | [Expressions](#expression-indexes) |
| GIN for JSONB, full-text and trigram search | `LIKE '%x%'` on a big table with no trigram index | [Index types](#beyond-b-tree) |
| Exclusion constraint for "no overlapping bookings" | Check overlaps only in app code | [No overlapping appointments](#no-overlapping-appointments) |
| `CREATE INDEX CONCURRENTLY` on live tables, outside a transaction | Plain `CREATE INDEX` on a busy production table | [Live tables](#creating-indexes-on-live-tables) |
| Check stats age and replicas before dropping an "unused" index | Drop an index because it shows zero uses on one server | [Housekeeping](#finding-unused-and-missing-indexes) |

* * *

## Further reading

1.  **PostgreSQL Documentation — Indexes** (types, multicolumn, partial, expression, index-only scans)
    [https://www.postgresql.org/docs/current/indexes.html](https://www.postgresql.org/docs/current/indexes.html)
2.  **PostgreSQL Documentation — CREATE INDEX** (including `CONCURRENTLY`)
    [https://www.postgresql.org/docs/current/sql-createindex.html](https://www.postgresql.org/docs/current/sql-createindex.html)
3.  **PostgreSQL Documentation — pg_trgm**
    [https://www.postgresql.org/docs/current/pgtrgm.html](https://www.postgresql.org/docs/current/pgtrgm.html)
4.  **PostgreSQL Documentation — Full Text Search**
    [https://www.postgresql.org/docs/current/textsearch.html](https://www.postgresql.org/docs/current/textsearch.html)
5.  **Use The Index, Luke** — how indexes work, for developers, across engines
    [https://use-the-index-luke.com/](https://use-the-index-luke.com/)
6.  **Npgsql EF Core provider — Indexes** (`IncludeProperties`, `HasMethod`, `HasOperators`)
    [https://www.npgsql.org/efcore/modeling/indexes.html](https://www.npgsql.org/efcore/modeling/indexes.html)

* * *

## Related pages

- [Writing Queries](/Database-Standards/Writing-Queries) — writing conditions that can actually use these indexes.
- [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics) — proving an index is used, with `EXPLAIN`.
- [Migrations and Schema Changes](/Database-Standards/Migrations-and-Schema-Changes) — shipping a new index without downtime.
