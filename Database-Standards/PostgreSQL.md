PostgreSQL
==========

> **Our default database for every new project.** Read **[General Principles](/Database-Standards/General-Principles)** first. This page covers only what's specific to Postgres.
> **How Postgres is set up in the NestJS stack:** [Tech Stack → Backend Stack](/Tech-Stack/Backend-Stack).

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [Why Postgres](#why-postgres-is-our-default) | **PostgreSQL for every new project unless told otherwise**<br/><sub>Free, standards-compliant, available on every cloud, and it covers JSON, search and ranges without a second database.</sub> |
| [Versions](#versions) | **Run a supported major version, and apply minor releases promptly**<br/><sub>Each major is supported for 5 years. Minor releases are security and bug fixes only.</sub> |
| [MVCC and VACUUM](#mvcc-and-vacuum) | **Never turn autovacuum off, never leave a transaction open**<br/><sub>Every update leaves an old row version behind. VACUUM cleans them up, if nothing is holding them.</sub> |
| [Data types](#data-types-worth-knowing) | **`text`, `timestamptz`, `numeric`, `bigint` identity, `jsonb`**<br/><sub>Use Postgres's types instead of squeezing everything into varchar.</sub> |
| [Enums](#enums-or-a-lookup-table) | **Postgres enums only for value lists that never shrink**<br/><sub>You can add a value, but you can't remove one.</sub> |
| [JSONB](#working-with-jsonb) | **JSONB for data whose shape varies, columns for data you filter and join on**<br/><sub>A GIN index makes containment queries fast. It can't make a JSON foreign key.</sub> |
| [Full-text search](#full-text-search) | **A generated `tsvector` column with a GIN index**<br/><sub>Good enough for most in-app search. Typo tolerance and facets need a search engine.</sub> |
| [Extensions](#extensions-we-use) | **Check the managed provider supports an extension before you design around it**<br/><sub>Most common ones are everywhere. Some aren't.</sub> |
| [Schemas](#schemas-and-search-path) | **Only the database owner creates objects in a schema**<br/><sub>Postgres 15 made this the default for `public`. Older versions need one REVOKE.</sub> |
| [Settings](#settings-worth-knowing) | **Set `statement_timeout` and `idle_in_transaction_session_timeout` for the app user**<br/><sub>A runaway query or a forgotten transaction should end itself, not lock the table.</sub> |
| [psql](#psql-tips) | **`\d+`, `\x auto`, `\timing`, and `ON_ERROR_ROLLBACK`**<br/><sub>A few settings in `~/.psqlrc` make the console far nicer and safer.</sub> |
| [Gotchas](#common-gotchas) | **Foreign keys aren't indexed for you, `now()` doesn't move, `count(*)` isn't free**<br/><sub>The surprises people coming from other engines run into most.</sub> |
| [Local setup](#running-postgres-locally) | **Docker image pinned to production's major version**<br/><sub>A data directory from one major version won't start on another.</sub> |

* * *

## Why Postgres is our default

| Reason | What it means for a project |
| --- | --- |
| **Free, and managed everywhere** | No licence or per-core cost, no locked editions. Managed with backups and failover on AWS RDS and Aurora, Azure Database for PostgreSQL (Flexible Server), Google Cloud SQL, Supabase, Neon. |
| **Close to the SQL standard** | Skills transfer; the planner handles window functions, CTEs, `LATERAL`, `FILTER` well. |
| **More than tables** | `jsonb`, full-text search, ranges, arrays, plus extensions (trigram, PostGIS, pgvector for AI). Often no second database needed. |
| **Concurrency and safe migrations** | Readers and writers never block each other (MVCC, below). `ALTER TABLE` is transactional, so a failed migration rolls back cleanly. MySQL can't do that. |

* * *

## Versions

| As of November 2026 | |
| --- | --- |
| Release cycle | One major a year, each supported **5 years**. Minor releases (18.1, 18.2, …) at least quarterly: bug and security fixes only, never new features |
| Supported majors | 18, 17, 16, 15, 14 |
| Ending soon | **14 reaches end of life on 12 November 2026** |
| Coming | **19** is in release-candidate testing |
| Always check | [postgresql.org/support/versioning](https://www.postgresql.org/support/versioning/) |

- **New projects:** the newest major your provider offers with **at least one minor release** (18 today). Move to 19 at 19.1 or 19.2, once the provider supports it.
- **Minor releases:** apply promptly (no behaviour changes, often security fixes). On managed services, enable automatic minor upgrades in a maintenance window.
- **Major upgrades:** plan before end of life. Upgrade a restored copy in staging, run the tests, then production (managed: one-click; self-hosted: `pg_upgrade`). Near-zero downtime on a big database: logical replication into the new version, then switch over.

> ⚠️ Managed providers keep old versions running past end of life, but often **charge for it** (AWS RDS Extended Support). An old version costs money as well as security.

* * *

## MVCC and VACUUM

`UPDATE` writes a **new row version** and marks the old one dead; `DELETE` only marks it dead. With this **MVCC** (multi-version concurrency control) each transaction sees the versions that existed when it started, so readers and writers don't block each other. **Autovacuum** clears dead versions unless an open transaction might still need them, so one transaction open for hours (or "idle in transaction") lets them pile up database-wide: **bloat**, bigger tables and indexes, slower queries. Spotting bloat and a lagging vacuum: [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics).

- **Never turn autovacuum off.** If it can't keep up on a busy table, tune it to run more often there.
- **Keep transactions short**, and set `idle_in_transaction_session_timeout` ([Settings](#settings-worth-knowing)).
- **A huge one-off `UPDATE` can double a table's size** until vacuum catches up. Batch it: [Large Data and Bulk Operations](/Database-Standards/Large-Data-and-Bulk-Operations).

* * *

## Data types worth knowing

| For | Use | Not | Why |
| --- | --- | --- | --- |
| Text | `text` | `varchar(255)` by habit | Same performance. Add `CHECK (length(x) <= n)` only for a real business limit. |
| Case-insensitive text (emails) | `citext` (extension) or a `lower(x)` unique index | `varchar` and hoping | `citext` makes `=` and `UNIQUE` ignore case. |
| A moment in time | `timestamptz` | `timestamp` | [General Principles → Store time in UTC](/Database-Standards/General-Principles#store-time-in-utc). |
| A calendar date | `date` | `timestamptz` at midnight | A due date isn't a moment. |
| Money | `numeric(12,2)` or `bigint` cents | `float`, `money` | [General Principles → Money is never a float](/Database-Standards/General-Principles#money-is-never-a-float). |
| Primary key | `bigint GENERATED ALWAYS AS IDENTITY` | `serial`, `int` | [Identity columns and UUIDs](#identity-columns-and-uuids). |
| Flexible, nested data | `jsonb` | `json`, or JSON in `text` | Stored parsed, so indexable and fast. `json` keeps the exact input text, which you almost never need. |
| A short list read with its row | `text[]`, `bigint[]` | A comma-separated string | Indexable and queryable. See [Arrays](#arrays). |
| A time window or a number span | `tstzrange`, `daterange`, `numrange` | Two columns with no rule linking them | Overlap operators; can power an exclusion constraint. |
| An IP address or network | `inet`, `cidr` | `text` | Validated on insert; "is this IP in this network" queries. |
| True / false | `boolean` | `smallint`, `char(1)` | |

### Arrays

For a short list read with its row, like a product's tags. Use a separate table when items are things in their own right: you join on them, they need a foreign key (impossible on array elements), or they have attributes.

```sql
ALTER TABLE products ADD COLUMN tags text[] NOT NULL DEFAULT '{}';
CREATE INDEX products_tags_idx ON products USING gin (tags);
SELECT id, name FROM products WHERE tags @> ARRAY['vegan'];    -- has the tag 'vegan'
```

### Enums or a lookup table

```sql
CREATE TYPE appointment_status AS ENUM ('booked', 'confirmed', 'completed', 'cancelled');
ALTER TYPE appointment_status ADD VALUE 'no_show';                 -- ✅ adding is easy
ALTER TYPE appointment_status RENAME VALUE 'booked' TO 'pending';  -- ✅ renaming works
-- ❌ there is no DROP VALUE: removing one means creating a new type and converting the column
```

| Option | Use it when |
| --- | --- |
| **Postgres `ENUM`** | A short list that will only ever grow: statuses in a fixed workflow. |
| **`text` + `CHECK (status IN (...))`** | The list may shrink or change. Changing a `CHECK` is one migration. |
| **Lookup table + foreign key** | Users or admins manage the list, or each value has extra data (label, colour, sort order). |

> 📝 A Prisma `enum` on Postgres creates a real Postgres enum type, with the same "can't remove a value" limit. So does TypeORM's `enum` column by default.

### Identity columns and UUIDs

```sql
id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY      -- ✅ internal key: compact, fast, ordered
public_id uuid NOT NULL DEFAULT uuidv7() UNIQUE         -- ✅ public id (URLs, other systems): Postgres 18+
id serial PRIMARY KEY                                   -- ❌ old style: a plain sequence with looser rules
```

- **`GENERATED ALWAYS`** rejects a supplied id, preventing the classic "imported rows collide with the sequence" bug. A one-off import that must keep ids uses `INSERT ... OVERRIDING SYSTEM VALUE`.
- **`uuidv7()`** (Postgres 18+) is time-ordered, keeping indexes compact like a sequence. `gen_random_uuid()` (built in since 13) is random v4 and scatters index inserts. On 17 or older, generate v7 in the app. Which key when: [Schema Design](/Database-Standards/Schema-Design).

> ⚠️ Identity and sequence values have **gaps**: a rolled-back insert still uses up its number. Never use them where gaps aren't allowed, like invoice numbers in some tax systems; generate those separately, in a transaction.

### Generated columns

Computed from other columns of the same row on every write:

```sql
ALTER TABLE invoice_lines ADD COLUMN line_total numeric(12,2) GENERATED ALWAYS AS (quantity * unit_price) STORED;
```

- `STORED` (Postgres 12+) is saved on write, so it can be indexed. Postgres 18 added `VIRTUAL` (computed when read) as the **default** when you write neither keyword: write `STORED` explicitly to index the column.
- Only immutable functions are allowed: `||` works, `concat_ws()` doesn't.

* * *

## Working with JSONB

| Operator | Means | Example |
| --- | --- | --- |
| `->` | Get a field, as `jsonb` | `settings -> 'notifications'` |
| `->>` | Get a field, as `text` | `settings ->> 'language'` |
| `#>>` | Get a nested path, as `text` | `settings #>> '{notifications,email}'` |
| `@>` | Contains | `settings @> '{"language": "en"}'` |
| `?` | Has this top-level key | `settings ? 'beta'` |

```sql
-- ✅ GIN index for "contains" queries on the whole document
CREATE INDEX customers_settings_idx ON customers USING gin (settings jsonb_path_ops);
SELECT id FROM customers WHERE settings @> '{"language": "en"}';

-- ✅ one field you filter on a lot: a normal B-tree index on just that field
CREATE INDEX customers_settings_language_idx ON customers ((settings ->> 'language'));
SELECT id FROM customers WHERE settings ->> 'language' = 'en';

-- update one key without rewriting the rest in app code
UPDATE customers SET settings = jsonb_set(settings, '{notifications,email}', 'false') WHERE id = 42;
```

- `jsonb_path_ops` indexes are smaller and faster for `@>` but don't support `?`; use the default `jsonb_ops` for key-exists queries.
- **Don't put relational data in JSONB.** Anything you regularly filter, join, sort or sum on, or that other rows refer to, is a real column with a type and constraints. JSONB is for parts whose shape really varies: settings, form answers, an external API payload.
- Updating any part rewrites the whole value: split a large, often-changing document into columns or rows.

* * *

## Full-text search

Word search in longer text (descriptions, notes, articles), with stemming ("cooling" matches "cooled") and ranking.

```sql
ALTER TABLE products
    ADD COLUMN search tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(description, '')), 'B')
    ) STORED;
CREATE INDEX products_search_idx ON products USING gin (search);

-- websearch_to_tsquery understands what people type: quotes, OR, and -word
SELECT id, name, ts_rank(search, query) AS rank
FROM   products,
       websearch_to_tsquery('english', 'cooling pad -laptop') AS query
WHERE  search @@ query
ORDER  BY rank DESC
LIMIT  20;
```

- **Weights** (`A` name, `B` description) rank a name match higher. **Pick the language** (`'english'`) to match the content: it decides stemming and which words to ignore.
- For "contains these letters" search on short fields (names, emails, codes), use `pg_trgm` instead. See [SQL Cookbook](/Database-Standards/SQL-Cookbook).

> 💡 Move to a search engine (Meilisearch, Typesense, OpenSearch) when you need typo tolerance, faceted filters across millions of rows, or per-language relevance tuning. Until then, Postgres saves running and syncing a second system.

* * *

## Extensions we use

Enable extensions in a migration so every environment has them: `CREATE EXTENSION IF NOT EXISTS pg_trgm;`.

| Extension | What for | Notes |
| --- | --- | --- |
| `pg_stat_statements` | Per-query stats: calls, total and average time | First tool for finding slow queries. Needs `shared_preload_libraries` (managed providers usually set it). See [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics). |
| `pg_trgm` | Fast `LIKE` / `ILIKE '%...%'` and similarity search | GIN index with `gin_trgm_ops`. |
| `citext` | Case-insensitive text type | Emails, usernames. |
| `btree_gist` | Lets normal columns join range columns in an exclusion constraint | For the "no double booking" constraint in the [SQL Cookbook](/Database-Standards/SQL-Cookbook). |
| `unaccent` | Accent-insensitive search (`café` = `cafe`) | With full-text or trigram search. |
| `postgis` | Geospatial: distance, "within this area" | Anything location-based. |
| `vector` (pgvector) | Embeddings and similarity search | AI and semantic search. |
| `pg_partman` | Creates and drops time-based partitions automatically | See [Large Data and Bulk Operations](/Database-Standards/Large-Data-and-Bulk-Operations). |
| `pgcrypto` | Cryptographic functions | Not needed for UUIDs (`gen_random_uuid()` is built in since Postgres 13). Hash passwords in the app, not in SQL. |

> ⚠️ Managed providers only allow extensions from their own list, which differs between AWS, Azure, Google Cloud, Supabase and Neon. Check it **before** designing a feature around an extension.

* * *

## Schemas and search path

A **schema** is a namespace inside a database (`public.invoices`, `reporting.daily_revenue`); the **search path** decides which schemas an unprefixed name is looked up in.

- Most projects only need `public`. Add schemas for a clear reason, like a `reporting` schema with different permissions.
- **Postgres 15+:** only the database owner can create objects in `public`, closing a hole where any user could create a function shadowing one you meant to call. **Postgres 14 or older:** do it yourself, once per database: `REVOKE CREATE ON SCHEMA public FROM PUBLIC;`
- Prisma takes the schema from the connection string (`?schema=app`); TypeORM, Laravel and EF Core have a `schema` option in connection or model config.
- Which users and roles to create: [Access and Roles](/Database-Standards/Access-and-Roles).

* * *

## Settings worth knowing

| Setting | What it does | Our default |
| --- | --- | --- |
| `statement_timeout` | Cancels a statement running longer than this | `30s` for the app user. Reports and migrations set their own. |
| `lock_timeout` | Stops waiting for a lock after this long | `5s` in migrations, so a blocked `ALTER TABLE` fails instead of queueing every query behind it. See [Migrations and Schema Changes](/Database-Standards/Migrations-and-Schema-Changes). |
| `idle_in_transaction_session_timeout` | Ends a session idle inside an open transaction | `60s`. Stops a forgotten transaction blocking vacuum all day. |
| `transaction_timeout` | Ends a transaction running longer than this (Postgres 17+) | Optional, if long transactions keep recurring. |
| `log_min_duration_statement` | Logs statements slower than this | `500ms` to `1s`, depending on the app. |
| `work_mem` | Memory **each** sort or hash step may use before spilling to disk (default 4MB) | Raise per reporting session (`SET work_mem = '64MB'`), not globally: a complex query uses it several times over, on every connection. |
| `random_page_cost` | Planner's cost of a random disk read (default 4.0) | About `1.1` on SSD, which all managed services use. |
| `shared_buffers` | Postgres's own cache | Self-hosted only: about 25% of RAM to start. Managed services set it. |

Set per-session limits on the app's role, so every connection gets them:

```sql
ALTER ROLE app_user SET statement_timeout = '30s';
ALTER ROLE app_user SET idle_in_transaction_session_timeout = '60s';
```

* * *

## psql tips

| Command | What it does |
| --- | --- |
| `\dt` / `\di` / `\dx` | List tables / indexes / installed extensions |
| `\d+ invoices` | A table's columns, types, indexes, constraints and size |
| `\x auto` | Show wide rows vertically when they don't fit |
| `\timing on` | Print each query's duration |
| `\e` | Edit the last query in your editor; runs on save and quit |
| `\watch 5` | Re-run the last query every 5 seconds |
| `\copy invoices TO 'invoices.csv' CSV HEADER` | Export or import a file on **your** machine |
| `\conninfo` | Which database, host and user you're on. Check before anything risky. |

> 📝 `COPY` (no backslash) reads and writes files **on the database server** and needs special privileges. `\copy` streams through psql to and from your machine, which is almost always what you want.

A safer, more readable `~/.psqlrc`. `ON_ERROR_ROLLBACK interactive` makes a typo inside a transaction undo only that statement, not the whole transaction; `\pset null '∅'` shows NULL distinctly from an empty string.

```
\set ON_ERROR_ROLLBACK interactive
\set HISTFILE ~/.psql_history- :DBNAME
\pset null '∅'
\x auto
\timing on
```

When running a script, use `\set ON_ERROR_STOP on` (or `psql -v ON_ERROR_STOP=1 -f script.sql`) so it stops at the first error.

* * *

## Common gotchas

| Gotcha | What happens | What to do |
| --- | --- | --- |
| **Unquoted names are lowercased** | `"Invoice"` and `Invoice` are different tables | snake_case. See [General Principles → Naming](/Database-Standards/General-Principles#one-naming-convention-per-database). |
| **Foreign keys aren't indexed automatically** | Only the referenced primary key is. Joins and cascading deletes on `invoices.customer_id` scan the whole table. | Index every foreign key column. See [Indexing](/Database-Standards/Indexing). |
| **NULLs don't clash in a UNIQUE constraint** | Two rows with `email = NULL` are both allowed | `NOT NULL`, or `UNIQUE NULLS NOT DISTINCT` (Postgres 15+). |
| **Integer division** | `SELECT 1 / 4` returns `0` | `1::numeric / 4`. See the [SQL Cookbook](/Database-Standards/SQL-Cookbook). |
| **`now()` doesn't move** | It's the **transaction** start time, the same for every row and statement | `clock_timestamp()` for the actual time, e.g. timing steps in a long script. |
| **`timestamp` vs `timestamptz`** | No time zone: the same value means different instants on different servers | `timestamptz`, always. |
| **`count(*)` is slow on big tables** | MVCC makes Postgres check every row's visibility, reading the whole table | For "about how many": `SELECT reltuples::bigint FROM pg_class WHERE oid = 'invoices'::regclass;` |
| **`OFFSET` gets slower every page** | `OFFSET 100000` reads and discards 100,000 rows | Keyset pagination. See [Writing Queries](/Database-Standards/Writing-Queries). |
| **`serial` instead of identity** | Sequence permissions and ownership managed separately; manual ids collide later | `bigint GENERATED ALWAYS AS IDENTITY`. |
| **`ALTER TABLE` waits for a lock** | Even a fast `ALTER` queues behind a long query, and every new query queues behind it | `SET lock_timeout` in migrations. See [Migrations and Schema Changes](/Database-Standards/Migrations-and-Schema-Changes). |

* * *

## Running Postgres locally

Pin the Docker image to **production's major version**, never `latest` or an unversioned tag: one major's data directory won't start under another.

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:18-alpine          # ✅ same major as production
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql     # Postgres 18+ images: mount the parent folder

volumes:
  pgdata:
```

> ⚠️ **The volume path changed with the Postgres 18 Docker image:** `/var/lib/postgresql` from 18 on, `/var/lib/postgresql/data` for 17 and older. The wrong path either won't start or silently starts with an empty database on every restart.
>
> ⚠️ Changing the image's major version (17 → 18) doesn't upgrade the data. Dump and restore into a fresh volume, or wipe it and re-run migrations and seeds.

Full compose setup with the API: [Tech Stack → Backend Stack](/Tech-Stack/Backend-Stack).

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Run a supported major and apply minor releases promptly | Sit on a version past end of life | [Versions](#versions) |
| Keep transactions short and set `idle_in_transaction_session_timeout` | Turn off autovacuum, or leave sessions idle in a transaction | [MVCC and VACUUM](#mvcc-and-vacuum) |
| `text`, `timestamptz`, `numeric`, `jsonb` | `varchar(255)` by habit, `timestamp`, `float`, `json` | [Data types](#data-types-worth-knowing) |
| `bigint GENERATED ALWAYS AS IDENTITY`, `uuidv7()` for public ids | `serial`, random UUIDs as the primary key of a busy table | [Identity and UUIDs](#identity-columns-and-uuids) |
| `CHECK` or a lookup table for lists that change | A Postgres enum for a list that may lose values | [Enums](#enums-or-a-lookup-table) |
| JSONB for data whose shape varies, with a GIN or expression index | JSONB for values you join, filter and sum on every day | [JSONB](#working-with-jsonb) |
| A generated `tsvector` column with a GIN index | `ILIKE '%word%'` across long text columns | [Full-text search](#full-text-search) |
| Enable extensions in a migration, after checking the provider supports them | Design a feature around an extension production can't install | [Extensions](#extensions-we-use) |
| Set `statement_timeout` and `lock_timeout` | Raise `work_mem` globally to fix one slow report | [Settings](#settings-worth-knowing) |
| `\copy` to export to your machine | `COPY ... TO` a server path | [psql tips](#psql-tips) |
| Index every foreign key column | Assume Postgres created one | [Gotchas](#common-gotchas) |
| Pin the Docker image to production's major version | `postgres:latest` | [Local setup](#running-postgres-locally) |

* * *

## Further reading

1.  **PostgreSQL — Versioning Policy** (supported versions and end-of-life dates)
    [https://www.postgresql.org/support/versioning/](https://www.postgresql.org/support/versioning/)
2.  **PostgreSQL Documentation — Routine Vacuuming**
    [https://www.postgresql.org/docs/current/routine-vacuuming.html](https://www.postgresql.org/docs/current/routine-vacuuming.html)
3.  **PostgreSQL Documentation — JSON Types** (including JSONB indexing)
    [https://www.postgresql.org/docs/current/datatype-json.html](https://www.postgresql.org/docs/current/datatype-json.html)
4.  **PostgreSQL Documentation — Full Text Search**
    [https://www.postgresql.org/docs/current/textsearch.html](https://www.postgresql.org/docs/current/textsearch.html)
5.  **PostgreSQL Documentation — Client Connection Defaults** (`statement_timeout`, `lock_timeout`, idle timeouts)
    [https://www.postgresql.org/docs/current/runtime-config-client.html](https://www.postgresql.org/docs/current/runtime-config-client.html)
6.  **PostgreSQL Documentation — psql**
    [https://www.postgresql.org/docs/current/app-psql.html](https://www.postgresql.org/docs/current/app-psql.html)
7.  **PostgreSQL Wiki — Don't Do This**
    [https://wiki.postgresql.org/wiki/Don't_Do_This](https://wiki.postgresql.org/wiki/Don't_Do_This)
8.  **Docker Official Image — postgres** (tags, environment variables, data directory)
    [https://hub.docker.com/_/postgres](https://hub.docker.com/_/postgres)

* * *

## Related pages

- [Database Standards → Which database?](/Database-Standards#which-database) — when Postgres is the right choice, and what sits next to it.
- [Indexing](/Database-Standards/Indexing) and [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics) — making Postgres queries fast and keeping them fast.
- [Backups and Recovery](/Database-Standards/Backups-and-Recovery) and [Replication and Monitoring](/Database-Standards/Replication-and-Monitoring) — running Postgres in production.
- [SQL Cookbook](/Database-Standards/SQL-Cookbook) — Postgres recipes for everyday questions.
- [Engine Differences](/Database-Standards/Engine-Differences) — what changes when a query written for Postgres has to run on SQL Server or MySQL.
