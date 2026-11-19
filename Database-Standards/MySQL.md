MySQL
=====

> Read **[General Principles](/Database-Standards/General-Principles)** first. The topic pages are written for PostgreSQL; this page is what's **different** on MySQL (or MariaDB).
> **When we use MySQL at all:** see [Database Standards → Which database?](/Database-Standards#which-database).

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [When we use it](#when-we-use-it) | **Existing Laravel/PHP projects, and hosting where it's the only option**<br/><sub>A new project where Postgres is available uses Postgres.</sub> |
| [Versions](#versions) | **MySQL 8.4 LTS. Anything still on 8.0 plans its upgrade now**<br/><sub>MySQL 8.0 reached end of life in April 2026: no more security fixes.</sub> |
| [MariaDB](#mariadb-is-not-mysql) | **Treat MariaDB as a different database, not a drop-in MySQL**<br/><sub>JSON, collations, upsert syntax and replication have all diverged.</sub> |
| [Storage engine](#always-innodb) | **InnoDB for every table**<br/><sub>It's the only engine with transactions, row locks, foreign keys and crash recovery.</sub> |
| [Character set](#utf8mb4-and-collation) | **`utf8mb4`, never `utf8`**<br/><sub>MySQL's `utf8` is a 3-byte subset that can't store emoji or some names.</sub> |
| [SQL mode](#strict-sql-mode) | **Strict mode stays on. Fix the query, not the mode**<br/><sub>Without it, MySQL silently truncates and rounds bad data instead of rejecting it.</sub> |
| [Translating from Postgres](#translating-from-postgres) | **Backticks, `AUTO_INCREMENT`, no `RETURNING`, `ON DUPLICATE KEY UPDATE`**<br/><sub>The table below covers what doesn't map one-to-one.</sub> |
| [Primary keys](#the-primary-key-is-the-table) | **Small, ever-increasing `BIGINT UNSIGNED` primary key**<br/><sub>InnoDB stores rows in primary-key order, and every secondary index carries a copy of the key.</sub> |
| [Time](#dates-and-times) | **`DATETIME` in UTC. `TIMESTAMP` stops working in 2038**<br/><sub>Laravel's `timestamps()` creates `TIMESTAMP` columns; know which dates might pass 2038.</sub> |
| [Locking](#isolation-and-gap-locks) | **Know that REPEATABLE READ takes gap locks**<br/><sub>Two "insert if missing" requests can deadlock each other on a row that doesn't exist yet.</sub> |
| [Schema changes](#changing-tables-online) | **DDL commits immediately. Say `ALGORITHM=INSTANT` or `INPLACE` explicitly**<br/><sub>A migration that fails half-way can't roll back, and a silent table copy can lock writes for minutes.</sub> |
| [Diagnostics](#finding-slow-queries) | **Slow query log plus `EXPLAIN ANALYZE`**<br/><sub>The log tells you which queries; the plan tells you why.</sub> |
| [Backups](#backups) | **Consistent dump or physical backup, plus binary logs for point-in-time recovery**<br/><sub>A nightly dump alone can lose a whole day of data.</sub> |

* * *

## When we use it

**Not** our default. Only for an existing **Laravel/PHP** application already on MySQL, or when the client's hosting only offers MySQL or MariaDB (common on shared and cPanel hosting). A **new** project uses PostgreSQL whenever the hosting allows, Laravel ones included: Laravel supports Postgres fully.

> 📝 **Naming:** snake_case for everything (`invoice_lines`, `customer_id`, `created_at`), which Laravel's conventions produce. See [General Principles → One naming convention per database](/Database-Standards/General-Principles#one-naming-convention-per-database).

* * *

## Versions

| Version | Status | What to do |
| --- | --- | --- |
| **MySQL 8.4** | **LTS** (long-term support), supported until 2032 | Use it for anything new or upgraded. |
| **MySQL 9.x** | Innovation releases: new features, each supported only until the next | Not for client production. |
| **MySQL 8.0** | **End of life since April 2026.** No more security fixes from Oracle | Plan the upgrade to 8.4. Cloud providers' paid extended support is time-limited and not a plan. |
| **MySQL 5.7 and older** | Long out of support | Flag it to the client as a security risk. |

> ⚠️ **Upgrading 8.0 → 8.4:** the old `mysql_native_password` authentication plugin is disabled by default in 8.4, and users still on it can't log in. Move app users to `caching_sha2_password` (supported by current PHP `mysqlnd`, Node and .NET drivers) before upgrading, and test the app's connection in staging.

Many features on this page need a minimum 8.0 point release; 8.4 has all of them.

### MariaDB is not MySQL

A 2009 fork of MySQL that has drifted apart since. Most simple SQL works on both, but don't assume a MySQL answer applies to MariaDB.

| Area | MySQL 8.x | MariaDB |
| --- | --- | --- |
| `JSON` type | Native binary JSON type | Alias for `LONGTEXT` with a validity check |
| Default `utf8mb4` collation | `utf8mb4_0900_ai_ci` | Different collation names; `utf8mb4_0900_*` collations aren't available everywhere |
| Upsert using the new row's values | Row alias: `... AS new ON DUPLICATE KEY UPDATE x = new.x` (8.0.19+) | `VALUES(x)` |
| `RETURNING` | Not supported | `INSERT ... RETURNING` and `DELETE ... RETURNING` |
| Sequences | No | `CREATE SEQUENCE` |
| Replication | MySQL GTIDs | Its own GTID format; not interchangeable with MySQL's |

Check which you're on with `SELECT VERSION();` (MariaDB includes `MariaDB` in the string). Test on the same engine and major version as production: a dump from one doesn't always restore cleanly into the other.

* * *

## Setup that matters

### Always InnoDB

Every table uses **InnoDB**, the default and the only engine with transactions, row-level locking, foreign keys and crash recovery. A `MyISAM` table on an old project has none of those; converting it (`ALTER TABLE ... ENGINE=InnoDB`) rebuilds the table, so plan it.

```sql
-- find any table that isn't InnoDB
SELECT table_schema, table_name, engine
FROM   information_schema.tables
WHERE  table_schema = DATABASE() AND engine <> 'InnoDB';
```

### utf8mb4 and collation

MySQL's `utf8` is really **`utf8mb3`**: at most 3 bytes per character, so no emoji or some CJK characters. Inserts fail or, without strict mode, cut the text off. **Always `utf8mb4`** for the database, every table and the connection.

```sql
CREATE DATABASE billing CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;   -- ✅
CREATE DATABASE billing CHARACTER SET utf8;   -- ❌ 3-byte "utf8": the alias for utf8mb3, deprecated
```

**The collation decides how text compares.** MySQL 8's default, `utf8mb4_0900_ai_ci`, is **accent-insensitive (`ai`) and case-insensitive (`ci`)**: `'José' = 'jose'` and `'ALI' = 'ali'` are both true (more in [Engine Differences → Text](/Database-Standards/Engine-Differences#text-comparison)).

- A UNIQUE index on `email` treats `Ali@Example.com` and `ali@example.com` as duplicates, usually what you want. It also treats `resume` and `résumé` as the same: for codes, tokens or anything that must match exactly, use a binary collation (`utf8mb4_bin`) or a `BINARY`/`VARBINARY` type on that column.
- Keep **one collation across the whole database**. Joining columns with different collations either errors ("Illegal mix of collations") or can't use an index.

### Strict SQL mode

MySQL 8's default `sql_mode` includes `STRICT_TRANS_TABLES` and `ONLY_FULL_GROUP_BY`. With strict mode, bad data is **rejected with an error**. Without it, MySQL "fixes" it silently with only a warning: strings truncated, out-of-range numbers clamped, invalid dates zeroed.

```sql
INSERT INTO customers (name) VALUES (REPEAT('x', 300));   -- name is VARCHAR(200)
-- strict:     ERROR 1406: Data too long for column 'name'
-- not strict: saved with the name cut to 200 characters, and only a warning
```

`ONLY_FULL_GROUP_BY` rejects selecting a column that's neither grouped nor aggregated, which MySQL would otherwise fill from an arbitrary row ([Engine Differences → GROUP BY](/Database-Standards/Engine-Differences#group-by)). Say which row you mean:

```sql
-- ❌ SELECT customer_id, number, MAX(total) FROM invoices GROUP BY customer_id;  -- which invoice's number?
-- ✅ the latest invoice per customer
SELECT customer_id, number, total
FROM (
    SELECT customer_id, number, total,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
    FROM   invoices
) AS ranked
WHERE rn = 1;
```

> ⚠️ **Never turn strict mode or `ONLY_FULL_GROUP_BY` off to make a query "work".** The error is pointing at a real bug. Laravel's `'strict' => true` in `config/database.php` (the default) turns these modes on for the connection; leave it on.

* * *

## Translating from Postgres

| Postgres | MySQL | Watch out for |
| --- | --- | --- |
| `"identifier"` | `` `identifier` `` | Double quotes are string literals in MySQL unless `ANSI_QUOTES` is set |
| `bigint GENERATED ALWAYS AS IDENTITY` | `BIGINT UNSIGNED AUTO_INCREMENT` | Laravel's `$table->id()` creates exactly this. Gaps in the sequence are normal |
| `INSERT ... RETURNING id` | No `RETURNING`; use `LAST_INSERT_ID()` | Eloquent and the drivers do this for you |
| `INSERT ... ON CONFLICT (key) DO UPDATE` | `INSERT ... ON DUPLICATE KEY UPDATE` | Fires on a conflict with **any** unique index, not one you choose |
| `'a' \|\| 'b'` | `CONCAT('a', 'b')` | `\|\|` means logical OR in MySQL by default |
| `ILIKE` | `LIKE` | Case sensitivity comes from the collation (`_ci` = insensitive) |
| `boolean` | `BOOLEAN` | Just an alias for `TINYINT(1)`; it can hold any value from -128 to 127 |
| `jsonb` | `JSON` | Index a JSON field through a generated column or a functional index (below) |
| `timestamptz` | `DATETIME(3)` written in UTC | See [Dates and times](#dates-and-times) |
| `text` with no limit | `VARCHAR(n)` or `TEXT` | A `TEXT` column can only be indexed on a prefix |
| `string_agg(x, ', ')` | `GROUP_CONCAT(x SEPARATOR ', ')` | Silently truncated at `group_concat_max_len`, **1024 bytes** by default |
| Partial index `WHERE deleted_at IS NULL` | Not supported | Index the column normally, or use a generated column |
| `DISTINCT ON (...)` | Not supported | `ROW_NUMBER()` in a subquery, as in [Strict SQL mode](#strict-sql-mode) |
| `FULL OUTER JOIN` | Not supported | `LEFT JOIN ... UNION ... RIGHT JOIN` |
| `generate_series(...)` | Not supported | A recursive CTE |
| CTEs, window functions | Supported | MySQL 8.0+ |
| `CHECK` constraints | Enforced | Only since MySQL 8.0.16. Older versions parse them and ignore them |
| `INTERSECT`, `EXCEPT` | Supported | MySQL 8.0.31+ |
| Transactional DDL | **No.** `CREATE`, `ALTER`, `DROP` commit immediately | See [Changing tables online](#changing-tables-online) |
| `UPDATE ... LIMIT` | Not in Postgres | MySQL allows `LIMIT` on single-table `UPDATE` and `DELETE`, which is handy for batching |

### Upserts

Use the row-alias syntax (MySQL 8.0.19+); the older `VALUES(column)` form is deprecated.

```sql
-- ✅ MySQL 8.0.19+: "new" refers to the row you tried to insert
INSERT INTO customer_balances (customer_id, balance)
VALUES (42, 100.00) AS new
ON DUPLICATE KEY UPDATE balance = customer_balances.balance + new.balance;

-- ❌ deprecated since 8.0.20 (still the syntax on MariaDB)
INSERT INTO customer_balances (customer_id, balance) VALUES (42, 100.00)
ON DUPLICATE KEY UPDATE balance = balance + VALUES(balance);
```

> ⚠️ `ON DUPLICATE KEY UPDATE` triggers on a duplicate in **any** unique index on the table, so with more than one unique key a conflict on the "wrong" one updates a row you didn't expect. Keep upsert targets to tables with a single unique key, or check first. Laravel's `Model::upsert($rows, uniqueBy: [...], update: [...])` generates this statement, and on MySQL the `uniqueBy` columns are ignored: every unique index counts.

### JSON columns

`JSON` stores validated, binary JSON. To filter on a field quickly, index it through a **generated column**:

```sql
ALTER TABLE invoices
    ADD COLUMN source VARCHAR(30)
        GENERATED ALWAYS AS (metadata ->> '$.source') VIRTUAL,
    ADD INDEX idx_invoices_source (source);

SELECT id, number FROM invoices WHERE source = 'stripe';   -- ✅ uses the index
```

MySQL 8.0.13+ also supports functional indexes directly, and 8.0.17+ multi-valued indexes for JSON arrays (`MEMBER OF`). The generated column is the clearest for the team to read.

* * *

## Design differences

### The primary key is the table

InnoDB stores each table **in primary-key order** (a clustered index), and **every secondary index stores the primary key** as its pointer to the row. So **keep the primary key small** (a 36-character UUID string is copied into every secondary index) and **ever-increasing** (random keys insert into random pages, splitting pages and fragmenting the table).

```sql
-- ✅ small, increasing primary key; public UUID in its own unique column
CREATE TABLE invoices (
    id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    public_id    BINARY(16)      NOT NULL,
    customer_id  BIGINT UNSIGNED NOT NULL,
    number       VARCHAR(30)     NOT NULL,
    total        DECIMAL(12,2)   NOT NULL,
    created_at   DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    PRIMARY KEY (id),
    UNIQUE KEY uq_invoices_public_id (public_id),
    UNIQUE KEY uq_invoices_number (number),
    KEY idx_invoices_customer_id (customer_id),
    CONSTRAINT fk_invoices_customer FOREIGN KEY (customer_id) REFERENCES customers (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- ❌ a random UUID string as the primary key: 36+ bytes in every index, and page splits on every insert
CREATE TABLE invoices (id CHAR(36) NOT NULL PRIMARY KEY, ...);
```

**Storing UUIDs:**

- No UUID type: `BINARY(16)` takes 16 bytes, `CHAR(36)` 36 or more.
- MySQL's `UUID()` generates **version 1** UUIDs. To make them sort by time, store with `UUID_TO_BIN(uuid, 1)` (the swap flag) and read with `BIN_TO_UUID(col, 1)`.
- **UUIDv7**: generate it in the app. It's already time-ordered, so store it **without** the swap flag: `UUID_TO_BIN(uuid)`.
- Laravel's `$table->uuid()` creates a `CHAR(36)` column, and the `HasUuids` trait generates ordered UUIDs: fine for a public ID in a secondary unique column. `BINARY(16)` is only worth the extra casting on very large tables.

> 📝 Foreign keys need an index on the referencing column. InnoDB creates one if you don't, but name it yourself so migrations stay readable. See [Indexing](/Database-Standards/Indexing).

### Dates and times

| Type | Range | Time zone behaviour | Use for |
| --- | --- | --- | --- |
| `DATETIME(3)` | Years 1000–9999 | None; stores exactly what you write | Moments in time, **always written in UTC** |
| `TIMESTAMP` | Up to **2038-01-19** | Converted from the session time zone to UTC on write, and back on read | Avoid for new columns |
| `DATE` | Years 1000–9999 | None | Calendar dates: due dates, birthdays |

- Set the app and connection time zone to UTC so `NOW()` and `CURRENT_TIMESTAMP` are UTC. In Laravel: `'timezone' => 'UTC'` in `config/app.php`, and `'timezone' => '+00:00'` on the MySQL connection in `config/database.php`. `UTC_TIMESTAMP()` returns UTC regardless of session time zone.
- Laravel's `$table->timestamps()` creates `TIMESTAMP` columns on MySQL: fine for `created_at` and `updated_at`. For dates that can be far in the future (expiry dates, appointments years ahead, contract end dates), use `$table->dateTime(...)`.

* * *

## Locking and changes

### Isolation and gap locks

InnoDB defaults to **REPEATABLE READ** (Postgres and SQL Server: READ COMMITTED). Plain reads see a consistent snapshot. Locking reads and writes (`SELECT ... FOR UPDATE`, `UPDATE`, `DELETE`) also lock the **gaps** between index entries (next-key locks), so nobody can insert into the range they scanned. That causes the classic MySQL deadlock:

```sql
-- Sessions A and B both run this at once for the same new email:
START TRANSACTION;
SELECT id FROM users WHERE email = 'new@example.com' FOR UPDATE;   -- no row: both get a gap lock
INSERT INTO users (email, name) VALUES ('new@example.com', 'New');  -- each waits for the other's gap lock
-- → ERROR 1213: Deadlock found when trying to get lock
```

- Let a **UNIQUE constraint** decide and catch the duplicate-key error (or use `INSERT ... ON DUPLICATE KEY UPDATE`), instead of check-then-insert. See [General Principles → Constraints belong in the database](/Database-Standards/General-Principles#constraints-belong-in-the-database).
- Keep transactions short, and access tables in the same order everywhere.
- **Retry on deadlock.** In Laravel, `DB::transaction(fn () => ..., attempts: 3)` retries the whole closure.
- Some teams run OLTP apps at `READ COMMITTED`, which disables most gap locking. Agree that with the tech lead: it changes what repeated reads inside a transaction see.

See the last deadlock with `SHOW ENGINE INNODB STATUS\G` (section *LATEST DETECTED DEADLOCK*); set `innodb_print_all_deadlocks = ON` to log every one.

### Changing tables online

**MySQL DDL is not transactional** ([Engine Differences → Writes](/Database-Standards/Engine-Differences#writes-and-transactions)): if a Laravel migration with three `ALTER`s fails on the second, the first stays applied and `migrate:rollback` won't know. Keep **one schema change per migration** where you can. State the algorithm, so MySQL errors instead of silently copying the whole table:

```sql
-- ✅ adding a column: INSTANT changes metadata only (8.0.12+; any column position since 8.0.29)
ALTER TABLE invoices ADD COLUMN po_number VARCHAR(50) NULL, ALGORITHM=INSTANT;
-- ✅ adding an index: built in place while reads and writes continue
ALTER TABLE invoices ADD INDEX idx_invoices_created_at (created_at), ALGORITHM=INPLACE, LOCK=NONE;
-- ❌ no algorithm stated: if INSTANT/INPLACE isn't possible, MySQL copies the table and blocks writes
ALTER TABLE invoices MODIFY total DECIMAL(14,2) NOT NULL;
```

> ⚠️ **Metadata locks.** Even an instant `ALTER` needs a brief exclusive metadata lock. If a long query or an idle open transaction is using the table, the `ALTER` waits and **every query behind it queues up too**: the app stalls. Set a short wait on the migration session (`SET SESSION lock_wait_timeout = 5;`) and retry if it times out.

For big tables where the change needs a table copy (e.g. a column type change), use an online schema-change tool that builds a shadow copy and swaps it in: **gh-ost** (GitHub; reads the binary log, no triggers) or **pt-online-schema-change** (Percona Toolkit; uses triggers). Both are covered in [Migrations and Schema Changes](/Database-Standards/Migrations-and-Schema-Changes), with the general expand-and-contract approach.

* * *

## Finding slow queries

### Slow query log

On for every production database: it records every query slower than `long_query_time`. `mysqldumpslow` or Percona's `pt-query-digest` group the log by query shape, showing the worst queries instead of thousands of lines; managed services' performance dashboards usually do the same.

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;     -- seconds
-- make it permanent in my.cnf (or the cloud provider's parameter group)
```

### Performance Schema and the sys schema

Without the log, the `sys` schema summarises statement statistics:

```sql
-- statements with the highest total latency
SELECT query, exec_count, total_latency, rows_examined_avg FROM sys.statement_analysis LIMIT 10;
-- indexes that are never used (candidates to drop, after checking a full business cycle)
SELECT * FROM sys.schema_unused_indexes WHERE object_schema = DATABASE();
```

### EXPLAIN ANALYZE

`EXPLAIN` shows the plan. `EXPLAIN ANALYZE` (MySQL 8.0.18+) shows actual rows and timings per step, and, as on Postgres, **actually executes** the statement.

```sql
EXPLAIN ANALYZE
SELECT id, number, total FROM invoices WHERE customer_id = 42 AND status = 'paid';
```

| In `EXPLAIN` output | Means |
| --- | --- |
| `type: ALL` | Full table scan |
| `key: NULL` | No index used |
| `rows` far above what the query returns | The index isn't selective enough, or the wrong one was chosen |
| `Using filesort` / `Using temporary` | Sorting or grouping without a suitable index. Fine on small sets, slow on big ones |

Reading plans in general: [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics).

* * *

## Backups

Full strategy and restore testing: [Backups and Recovery](/Database-Standards/Backups-and-Recovery). MySQL specifics:

| Tool | Type | Use it for |
| --- | --- | --- |
| `mysqldump --single-transaction` | Logical (SQL text), consistent for InnoDB without locking | Small and medium databases, moving data between versions |
| **MySQL Shell** `util.dumpInstance()` / `util.loadDump()` | Logical, parallel, compressed | Larger databases. Much faster than `mysqldump` both ways |
| **Percona XtraBackup** | Physical (copies data files) while the server runs | Large databases where a logical restore would take too long |
| **Binary logs** | Every change since the last backup | **Point-in-time recovery**: restore the last backup, then replay the binlog with `mysqlbinlog` up to just before the mistake |
| Managed services | Automated snapshots plus binlog-based point-in-time restore | Turn on, set retention, and test a restore |

```bash
# ✅ consistent logical backup of one database, without locking InnoDB tables
mysqldump --single-transaction --routines --triggers --events \
  --set-gtid-purged=OFF -u backup_user -p billing | gzip > billing_$(date +%F).sql.gz
```

- Keep binary logs (`binlog_expire_logs_seconds`) at least as long as the gap between full backups, or point-in-time recovery has holes.
- `mysqlpump` was deprecated in 8.0 and removed in 8.4: use MySQL Shell's dump utilities.
- Use a dedicated backup user with only the privileges the tool needs. See [Access and Roles](/Database-Standards/Access-and-Roles).

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Run MySQL 8.4 LTS and plan upgrades off 8.0 | Stay on an end-of-life version | [Versions](#versions) |
| Test on the same engine (MySQL or MariaDB) and version as production | Assume MariaDB behaves like MySQL | [MariaDB](#mariadb-is-not-mysql) |
| InnoDB for every table | Keep MyISAM tables | [InnoDB](#always-innodb) |
| `utf8mb4` with one collation across the database | `utf8` (utf8mb3), or mixed collations | [utf8mb4](#utf8mb4-and-collation) |
| Keep strict mode and `ONLY_FULL_GROUP_BY` on | Turn a mode off to make an error go away | [Strict mode](#strict-sql-mode) |
| Row-alias upsert syntax, on tables with one unique key | `VALUES()` in `ON DUPLICATE KEY UPDATE` | [Upserts](#upserts) |
| Index JSON fields through a generated column | Filter on `JSON_EXTRACT` in a WHERE with no index | [JSON columns](#json-columns) |
| `BIGINT UNSIGNED AUTO_INCREMENT` key, UUID in a unique column | A `CHAR(36)` random UUID as the primary key | [Primary keys](#the-primary-key-is-the-table) |
| `DATETIME` in UTC, `DATE` for calendar dates | `TIMESTAMP` for dates that can pass 2038 | [Dates and times](#dates-and-times) |
| Let a UNIQUE constraint catch duplicates, retry on deadlock | `SELECT ... FOR UPDATE` then `INSERT` | [Gap locks](#isolation-and-gap-locks) |
| One schema change per migration, `ALGORITHM` stated | Several `ALTER`s that can fail half-way | [Online changes](#changing-tables-online) |
| `lock_wait_timeout` on migration sessions | Let an `ALTER` queue behind a long query and stall the app | [Online changes](#changing-tables-online) |
| Slow query log on in production, `EXPLAIN ANALYZE` to investigate | Guess which query is slow | [Finding slow queries](#finding-slow-queries) |
| Backups plus binary logs for point-in-time recovery | A nightly dump as the only protection | [Backups](#backups) |

* * *

## Further reading

1.  **MySQL 8.4 Reference Manual**
    [https://dev.mysql.com/doc/refman/8.4/en/](https://dev.mysql.com/doc/refman/8.4/en/)
2.  **MySQL 8.4 — Server SQL Modes**
    [https://dev.mysql.com/doc/refman/8.4/en/sql-mode.html](https://dev.mysql.com/doc/refman/8.4/en/sql-mode.html)
3.  **MySQL 8.4 — The utf8mb4 Character Set**
    [https://dev.mysql.com/doc/refman/8.4/en/charset-unicode-utf8mb4.html](https://dev.mysql.com/doc/refman/8.4/en/charset-unicode-utf8mb4.html)
4.  **MySQL 8.4 — InnoDB Locking**
    [https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
5.  **MySQL 8.4 — Online DDL Operations**
    [https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html)
6.  **MySQL 8.4 — INSERT ... ON DUPLICATE KEY UPDATE**
    [https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html](https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html)
7.  **gh-ost — GitHub's online schema migration tool for MySQL**
    [https://github.com/github/gh-ost](https://github.com/github/gh-ost)
8.  **Percona Toolkit — pt-online-schema-change**
    [https://docs.percona.com/percona-toolkit/pt-online-schema-change.html](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
9.  **endoflife.date — MySQL release support dates**
    [https://endoflife.date/mysql](https://endoflife.date/mysql)

* * *

## Related pages

- [Database Standards → Which database?](/Database-Standards#which-database) — when MySQL is the right call instead of Postgres.
- [General Principles](/Database-Standards/General-Principles) — the rules that apply on every engine.
- [Engine Differences](/Database-Standards/Engine-Differences) — behaviour that silently differs from Postgres and SQL Server.
- [Coding Standards → Backend — Laravel (PHP)](/Coding-Standards/Backend-Laravel-PHP) — where Eloquent code sits in a Laravel project.
- [Migrations and Schema Changes](/Database-Standards/Migrations-and-Schema-Changes) — Laravel migrations and changing live tables safely.
- [Backups and Recovery](/Database-Standards/Backups-and-Recovery) — the backup strategy across all engines.
- [Access and Roles](/Database-Standards/Access-and-Roles) — database users and least privilege.
