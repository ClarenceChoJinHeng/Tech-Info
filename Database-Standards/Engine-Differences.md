Engine Differences
==================

> Read **[General Principles](/Database-Standards/General-Principles)** first. This page lists behaviour that **silently differs** between PostgreSQL, SQL Server and MySQL: SQL that looks the same but returns a different result, fails on only one engine, or hits a limit on only one. Plain syntax differences (`LIMIT` vs `TOP`) are in the translation tables on the [SQL Server](/Database-Standards/SQL-Server) and [MySQL](/Database-Standards/MySQL) pages. MongoDB and Redis aren't SQL, so they have their own pages.

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [GROUP BY](#group-by) | **Group by, or aggregate, every column you select**<br/><sub>Postgres and MySQL accept shortcuts SQL Server rejects. MySQL with strict mode off returns a random row's value.</sub> |
| [DISTINCT and ORDER BY](#distinct-with-order-by) | **With DISTINCT, sort only by columns you select**<br/><sub>All three reject it today. Code that "worked on MySQL" was written for a version older than 5.7.</sub> |
| [Row order](#row-order) | **Only the outermost ORDER BY guarantees order**<br/><sub>Not a view's, not a subquery's. SQL Server refuses ORDER BY in a view.</sub> |
| [NULLs](#nulls) | **NULLs sort, concatenate and stay unique differently on each engine**<br/><sub>Spell it out with CASE, CONCAT_WS and COALESCE.</sub> |
| [Text](#text-comparison) | **Postgres compares text exactly. SQL Server and MySQL usually don't.**<br/><sub>Case, accents and trailing spaces all differ by default.</sub> |
| [Numbers](#numbers-and-booleans) | **Cast before dividing or averaging**<br/><sub>`7 / 2` is 3 on Postgres and SQL Server, 3.5 on MySQL.</sub> |
| [Writes](#writes-and-transactions) | **MySQL commits on every CREATE, ALTER or DROP**<br/><sub>A migration that fails halfway can't roll back there.</sub> |
| [Limits](#limits) | **SQL Server allows 2,100 parameters and 100 recursion levels**<br/><sub>Big IN lists and deep trees fail there and work elsewhere.</sub> |
| [Portable SQL](#writing-sql-that-behaves-the-same-everywhere) | **Test on the same engine and major version as production**<br/><sub>Never SQLite, or another engine, standing in for the real one.</sub> |

* * *

## Grouping and sorting

### GROUP BY

| Behaviour | PostgreSQL | SQL Server | MySQL |
| --- | --- | --- | --- |
| Selecting a column that's neither grouped nor aggregated | Error, **unless** it depends on a grouped primary key (9.1+) | Error 8120, always | Error with `ONLY_FULL_GROUP_BY` (default since 5.7.5), which also allows columns that depend on the grouped key |
| `ONLY_FULL_GROUP_BY` turned off | — | — | **Runs**, returning a value from whichever row MySQL picks |

```sql
-- Postgres ✅  MySQL ✅  SQL Server ❌ Msg 8120 (c.name isn't grouped)
SELECT c.id, c.name, count(*) FROM customers AS c
JOIN invoices AS i ON i.customer_id = c.id
GROUP BY c.id;

-- ✅ all three
... GROUP BY c.id, c.name;
```

> ⚠️ Laravel's `'strict' => false` turns `ONLY_FULL_GROUP_BY` off. Queries that "work" there return arbitrary values. Keep `strict` on.

### DISTINCT with ORDER BY

| Behaviour | PostgreSQL | SQL Server | MySQL |
| --- | --- | --- | --- |
| `SELECT DISTINCT a ... ORDER BY b`, `b` not selected | Error: *ORDER BY expressions must appear in select list* | Error 145 | Error 3065 on 5.7+. Older versions sorted by an arbitrary row's value |

> ⚠️ Decide which value to sort by and say it: `GROUP BY name ORDER BY min(created_at)`.

### Row order

| Behaviour | PostgreSQL | SQL Server | MySQL |
| --- | --- | --- | --- |
| No `ORDER BY` | Any order | Any order | Any order |
| `ORDER BY` in a view, subquery or CTE | Allowed, but the outer query needn't keep it | **Error 1033** unless `TOP`, `OFFSET` or `FOR XML` is used | Allowed, but may be ignored without `LIMIT` |

> ⚠️ "It always came back in insertion order" stops being true when the table grows or the plan changes. Sort the outermost query, with a unique tie-breaker: `ORDER BY created_at, id`.

* * *

## NULLs

| Behaviour | PostgreSQL | SQL Server | MySQL |
| --- | --- | --- | --- |
| `ORDER BY x` ascending | NULLs **last** | NULLs **first** | NULLs **first** |
| `NULLS FIRST` / `NULLS LAST` | Supported | Not supported | Not supported |
| NULLs in a `UNIQUE` column | Many (`NULLS NOT DISTINCT`, 15+, allows one) | **Only one** | Many |
| `CONCAT('Dr ', NULL)` | `'Dr '` | `'Dr '` | **NULL** |
| `'Dr ' \|\| NULL` | NULL | NULL (2025 and Azure SQL only) | `\|\|` means **OR** unless `PIPES_AS_CONCAT` |
| `'Dr ' + NULL` | Error | NULL | Numeric addition |
| Null-safe equality | `IS NOT DISTINCT FROM` | `IS NOT DISTINCT FROM` (2022+) | `<=>` |
| `''` vs NULL | Different | Different | Different |

```sql
-- NULLs last, the same on all three
ORDER BY CASE WHEN paid_at IS NULL THEN 1 ELSE 0 END, paid_at

-- SQL Server: optional unique column
CREATE UNIQUE INDEX UX_Customers_Phone ON Customers (Phone) WHERE Phone IS NOT NULL;
```

> ⚠️ `ORDER BY paid_at DESC` puts unpaid invoices at the **top** on Postgres only. A second customer without a phone fails to save on SQL Server only. `CONCAT` of a name with a NULL middle name is NULL on MySQL only; use `CONCAT_WS(' ', ...)`, which skips NULLs on all three.

* * *

## Text comparison

| Behaviour | PostgreSQL | SQL Server | MySQL |
| --- | --- | --- | --- |
| `'Ali' = 'ali'` | false | true (default `_CI_AS` collation) | true (default `utf8mb4_0900_ai_ci`) |
| `'José' = 'jose'` | false | false | **true** |
| `'abc' = 'abc '` | false | **true**: `=` ignores trailing spaces | false with `_0900_` collations; **true** with `utf8mb4_unicode_ci` (Laravel's default) |
| Characters / bytes | `length` / `octet_length` | `LEN` (ignores trailing spaces) / `DATALENGTH` | `CHAR_LENGTH` / **`LENGTH` is bytes** |
| `"paid"` | A column name | A column name | A **string** (unless `ANSI_QUOTES`) |
| Unquoted `Invoices` | Folded to `invoices` | Collation decides (case-insensitive by default) | Table names case-sensitive on Linux, not on Windows or macOS |
| `LIKE` wildcards | `%` `_` | `%` `_` **and** `[a-c]` | `%` `_` |
| `LIKE` default escape | Backslash | **None** | Backslash |
| `LIKE` case | Sensitive (`ILIKE` isn't) | Collation decides | Collation decides |

> ⚠️ A `UNIQUE` email lets `Ali@clinic.com` and `ali@clinic.com` both sign up on Postgres; store emails lower-cased or use `citext`. `FROM Invoices` works on a Windows laptop and fails on a Linux MySQL server. User search text containing `%`, `_` or `[` needs escaping; `LIKE ? ESCAPE '!'` works the same on all three.

* * *

## Numbers and booleans

| Behaviour | PostgreSQL | SQL Server | MySQL |
| --- | --- | --- | --- |
| `7 / 2` | `3` | `3` | `3.5000` |
| `AVG` of an integer column (1, 2) | `1.5` | **`1`** | `1.5000` |
| `SELECT 1 / 0` | Error | Error 8134 | **NULL**, with a warning |
| Boolean type | `boolean` | `bit` | `TINYINT(1)`: holds -128 to 127 |
| `WHERE is_active` | Works | **Syntax error** (Msg 4145): use `= 1` | Works: any non-zero is true |
| `SELECT a = b` | `true` / `false` | **Silently means `b AS a`** (alias syntax), not a comparison: use `CASE` | `1` / `0` |

> ⚠️ `paid_count / total_count` is 0 on Postgres and SQL Server. Cast one side to `decimal` first, and divide by `NULLIF(x, 0)` so all three return NULL instead of an error.

* * *

## Writes and transactions

| Behaviour | PostgreSQL | SQL Server | MySQL |
| --- | --- | --- | --- |
| `CREATE` / `ALTER` / `DROP` in a transaction | Rolls back (not `CREATE INDEX CONCURRENTLY`) | Rolls back (not `ALTER DATABASE`) | **Commits immediately**, including your open transaction |
| Value too long for the column | Error | Error (names the column since 2019). **Silent truncation** when assigning to a variable or casting | Error in strict mode (default). **Silent truncation** without it |
| Default isolation | Read Committed | Read Committed **with read locks**, unless `READ_COMMITTED_SNAPSHOT` (on by default in Azure SQL) | **Repeatable Read**, with gap locks |
| `UPDATE`/`DELETE` with a subquery on the same table | Works | Works | **Error 1093** |
| `LIMIT` inside `IN (subquery)` | Works | `TOP` works | **Error**: not supported |
| Limit rows in `UPDATE`/`DELETE` | Not supported; use a subquery | `TOP (n)`, arbitrary rows | `LIMIT n`, single-table only |
| New id from an insert | `RETURNING id` | `OUTPUT inserted.Id` (fails without `INTO` on a table with triggers), or `SCOPE_IDENTITY()`. **Not `@@IDENTITY`**: it returns a trigger's insert | `LAST_INSERT_ID()`: per connection, and the **first** id of a multi-row insert |
| Identity gaps | Rollbacks, `ON CONFLICT`, a small jump after a crash | Rollbacks, **jumps of up to 1,000 (`int`) or 10,000 (`bigint`)** after an unexpected restart (`IDENTITY_CACHE = OFF` stops it, 2017+) | Rollbacks, `ON DUPLICATE KEY` and `INSERT IGNORE`. **Before 8.0** the counter reset on restart and could reuse ids |
| `now()` and equivalents | Transaction start (`clock_timestamp()` for the actual time) | Statement start | `NOW()`: statement start. `SYSDATE()`: actual time |
| Session time zone changes stored values? | `timestamptz` display only | No | **`TIMESTAMP` is converted**; `DATETIME` isn't |

> ⚠️ A MySQL migration with three `ALTER TABLE`s that fails on the third leaves two applied, so keep one change per migration there. No engine gives gapless ids: use a counter row for invoice numbers. The same code can block on SQL Server, deadlock on MySQL and run fine on Postgres; see [Transactions and Concurrency](/Database-Standards/Transactions-and-Concurrency). More on triggers and `@@IDENTITY`: [Logic in the Database](/Database-Standards/Logic-in-the-Database).

* * *

## Limits

| Limit | PostgreSQL | SQL Server | MySQL |
| --- | --- | --- | --- |
| Name length | 63 bytes, **silently truncated** | 128, error | 64, error |
| Parameters per statement | 65,535 | **2,100** | 65,535 |
| Recursive CTE keyword | `WITH RECURSIVE` | `WITH` (`RECURSIVE` is an error) | `WITH RECURSIVE` (8.0+) |
| Recursion depth | No limit | **100**, then error 530: `OPTION (MAXRECURSION n)` | 1,000 (`cte_max_recursion_depth`) |
| Placeholders in raw SQL | `$1` (node-postgres) | `@name` | `?` |

> ⚠️ Dapper expands `IN @Ids` to one parameter per item, so 2,500 ids fail on SQL Server; use a table-valued parameter or `OPENJSON`, or `= ANY(@Ids)` on Postgres. A 12-column multi-row insert fits 175 rows per statement on SQL Server. Two long index names sharing their first 63 bytes collide on Postgres.

* * *

## Writing SQL that behaves the same everywhere

- `ORDER BY` on the outermost query, with a unique tie-breaker.
- Group by or aggregate every selected column.
- Spell out NULLs: `COALESCE`, `CONCAT_WS`, `NULLIF`, `CASE WHEN x IS NULL`.
- Cast explicitly before dividing, averaging or comparing different types.
- Single quotes for strings. Write table names exactly as created.
- Decide case sensitivity in the schema, not per query.
- Parameterize everything, and pass big lists as one array or JSON parameter.
- Test on the production engine and major version, in Docker locally.

* * *

## Do / Don't

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Group by or aggregate every selected column | Rely on the primary-key shortcut or MySQL with strict off | [GROUP BY](#group-by) |
| Sort DISTINCT results by a chosen aggregate | Sort by a column that isn't selected | [DISTINCT](#distinct-with-order-by) |
| `ORDER BY` on the outermost query | Rely on a view's or subquery's order | [Row order](#row-order) |
| Filtered unique index for optional unique columns on SQL Server | Plain `UNIQUE` on a nullable SQL Server column | [NULLs](#nulls) |
| `CONCAT_WS` or `COALESCE` around nullable parts | `CONCAT` with nullable columns on MySQL | [NULLs](#nulls) |
| Lower-case or `citext` emails on Postgres | Assume `'Ali' = 'ali'` everywhere | [Text](#text-comparison) |
| Escape `%`, `_`, `[` in user search text | Pass user input straight into `LIKE` | [Text](#text-comparison) |
| Cast before dividing, `NULLIF` the divisor | Trust `a / b` or `AVG(int)` | [Numbers](#numbers-and-booleans) |
| One schema change per MySQL migration | Several `ALTER TABLE`s in one MySQL migration | [Writes](#writes-and-transactions) |
| `SCOPE_IDENTITY()` or `OUTPUT` on SQL Server | `@@IDENTITY` | [Writes](#writes-and-transactions) |
| A counter row for gapless numbers | Identity values as invoice numbers | [Writes](#writes-and-transactions) |
| Array, JSON or table-valued parameters for big lists | A 3,000-item `IN` list on SQL Server | [Limits](#limits) |

* * *

## Further reading

1.  **PostgreSQL Documentation — SELECT**
    [https://www.postgresql.org/docs/current/sql-select.html](https://www.postgresql.org/docs/current/sql-select.html)
2.  **MySQL 8.4 Reference Manual — MySQL Handling of GROUP BY**
    [https://dev.mysql.com/doc/refman/8.4/en/group-by-handling.html](https://dev.mysql.com/doc/refman/8.4/en/group-by-handling.html)
3.  **MySQL 8.4 Reference Manual — Server SQL Modes**
    [https://dev.mysql.com/doc/refman/8.4/en/sql-mode.html](https://dev.mysql.com/doc/refman/8.4/en/sql-mode.html)
4.  **MySQL 8.4 Reference Manual — Identifier Case Sensitivity**
    [https://dev.mysql.com/doc/refman/8.4/en/identifier-case-sensitivity.html](https://dev.mysql.com/doc/refman/8.4/en/identifier-case-sensitivity.html)
5.  **SQL Server — ORDER BY clause (Transact-SQL)**
    [https://learn.microsoft.com/en-us/sql/t-sql/queries/select-order-by-clause-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-order-by-clause-transact-sql)
6.  **SQL Server — Collation and Unicode support**
    [https://learn.microsoft.com/en-us/sql/relational-databases/collations/collation-and-unicode-support](https://learn.microsoft.com/en-us/sql/relational-databases/collations/collation-and-unicode-support)
7.  **Modern SQL** — how standard SQL features behave across engines
    [https://modern-sql.com/](https://modern-sql.com/)

* * *

## Related pages

- [SQL Server](/Database-Standards/SQL-Server) and [MySQL](/Database-Standards/MySQL) — syntax translation from Postgres, and each engine's own rules.
- [PostgreSQL](/Database-Standards/PostgreSQL) — our default engine's gotchas.
- [Writing Queries](/Database-Standards/Writing-Queries) — NULL comparisons, pagination and sorting in depth.
- [Transactions and Concurrency](/Database-Standards/Transactions-and-Concurrency) — isolation and locking on each engine.
