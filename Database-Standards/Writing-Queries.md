Writing Queries
===============

> Read **[General Principles](/Database-Standards/General-Principles)** first. This page is how to write queries that return the right rows and let the database use its indexes.
> Examples are PostgreSQL. Where SQL Server or MySQL differ, the section says so.

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [Columns](#select-only-what-you-need) | **Name the columns you need, never `SELECT *` in code**<br/><sub>Less data over the wire, and a new column can't silently change your result.</sub> |
| [Index-friendly filters](#keep-filters-index-friendly) | **Never wrap the filtered column in a function or cast**<br/><sub>`created_at >= '2026-01-01'` can use the index. `EXTRACT(YEAR FROM created_at) = 2026` can't.</sub> |
| [Type mismatches](#match-the-column-type) | **Compare a column with a value of the same type**<br/><sub>An implicit conversion on the column side turns an index seek into a full scan.</sub> |
| [NULL](#null-in-comparisons) | **`IS NULL`, never `= NULL`, and never `NOT IN` over a nullable list**<br/><sub>One NULL in a `NOT IN` list makes the whole query return nothing.</sub> |
| [Joins](#joins) | **Join on keys, let the planner choose the order**<br/><sub>"Join the small table first" is a myth for Postgres, SQL Server and MySQL.</sub> |
| [EXISTS and IN](#exists-in-and-not-exists) | **`IN` or `EXISTS` for "has any", `NOT EXISTS` for "has none"**<br/><sub>Postgres plans `IN` and `EXISTS` the same way. `NOT IN` is the one that bites.</sub> |
| [OR and UNION](#or-or-union) | **Write `OR`, check the plan, switch to `UNION` only if it helps**<br/><sub>Postgres can combine two indexes for one `OR` on its own.</sub> |
| [CTEs and subqueries](#ctes-subqueries-and-temp-tables) | **CTEs for readability, temp tables for big reused intermediate results**<br/><sub>Since Postgres 12, a simple CTE costs the same as the subquery it replaces.</sub> |
| [N+1](#the-n-plus-one-problem) | **Load related rows with the list, not once per item**<br/><sub>50 invoices should be 2 queries, not 51.</sub> |
| [Pagination](#pagination) | **Keyset pagination for long or infinite lists**<br/><sub>`OFFSET 100000` still reads and throws away 100,000 rows.</sub> |
| [Counting](#counting-rows) | **Don't `COUNT(*)` a big table on every page load**<br/><sub>Fetch one extra row to know if there's a next page.</sub> |
| [Text search](#searching-text) | **`LIKE 'abc%'` can use an index, `LIKE '%abc%'` needs a trigram index**<br/><sub>Or full-text search, when you're searching words in prose.</sub> |
| [Useful features](#sql-features-worth-knowing) | **FILTER, window functions, LATERAL, RETURNING, ON CONFLICT**<br/><sub>Each one replaces a second query or a loop in app code.</sub> |

* * *

## Concepts you'll meet on this page

### Sargable

"Search ARGument ABLE": a condition the database can answer by **seeking** into an index instead of reading every row. The index stores `created_at`, not the year computed from it, so `EXTRACT(YEAR FROM created_at) = 2026` isn't sargable.

### Semi-join and anti-join

A **semi-join** keeps rows with **at least one** match in another table (`EXISTS`, `IN`). An **anti-join** keeps rows with **no** match (`NOT EXISTS`).

### Planner

Turns your SQL into a [query plan](/Database-Standards/General-Principles#query-plan) by estimating the cost of many ways to run it, from table statistics, and picking the cheapest. You describe **what**; the planner decides **how**.

* * *

## Filtering

### Select only what you need

```sql
-- ❌ every column: more I/O, memory and network, and the result changes shape when someone adds a column
SELECT * FROM invoices WHERE customer_id = 42;

-- ✅ only what the screen needs
SELECT id, number, total, status, due_date
FROM   invoices
WHERE  customer_id = 42;
```

In the ORM: Prisma `select: { id: true, number: true }`, TypeORM `select: { id: true, number: true }` or `.select([...])`, Eloquent `->select('id', 'number')` or `->get(['id', 'number'])`, EF Core `.Select(i => new InvoiceSummary(...))`. With Dapper you write the column list yourself.

> 💡 Fewer columns also makes an **index-only scan** possible: if the index holds every column the query needs, the table isn't read at all. See [Indexing](/Database-Standards/Indexing).

### Keep filters index friendly

Put the column **alone** on one side and do any calculation on the value side.

| ❌ Not sargable | ✅ Sargable |
| --- | --- |
| `WHERE EXTRACT(YEAR FROM created_at) = 2026` | `WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'` |
| `WHERE created_at::date = '2026-03-15'` | `WHERE created_at >= '2026-03-15' AND created_at < '2026-03-16'` |
| `WHERE total * 1.2 > 1000` | `WHERE total > 1000 / 1.2` |
| `WHERE COALESCE(cancelled_at, now()) > $1` | `WHERE cancelled_at > $1 OR cancelled_at IS NULL` |
| `WHERE lower(email) = 'john@example.com'` | The same query is fine **if** there's an index on `lower(email)` (an expression index) |

> 💡 **Use half-open ranges for dates and times:** `>= start AND < next_start`. `BETWEEN '2026-01-01' AND '2026-12-31'` on a timestamp silently drops everything after midnight on 31 December.

> 📝 On `timestamptz` columns, a literal like `'2026-03-15'` means midnight **in the session's time zone**. For a user's local day, compute the UTC range in the app and pass both ends as parameters.

### Match the column type

If the database converts the **column** to match the value, the column's index can't be used.

| Engine | The trap | Fix |
| --- | --- | --- |
| PostgreSQL | `WHERE id = 42.0` or a parameter typed `numeric` against an `integer`/`bigint` column: Postgres casts the column to `numeric`, and the index is skipped | Pass an integer. Postgres refuses most other mismatches (e.g. `text = integer`) with an error instead. |
| MySQL | `WHERE phone = 5551234` on a `VARCHAR` column: every row's `phone` is converted to a number | Quote it: `WHERE phone = '5551234'` |
| SQL Server | An `nvarchar` parameter against a `varchar` column causes `CONVERT_IMPLICIT` on the column and a scan | Match the parameter type to the column |

> ⚠️ **Dapper on SQL Server sends C# strings as `nvarchar` by default.** Against a `varchar` column, every lookup scans the table. Pass `new DbString { Value = code, IsAnsi = true, Length = 20 }` for `varchar` columns, or make the columns `nvarchar`.

### NULL in comparisons

Any comparison with `NULL` is unknown, and `WHERE` only keeps rows where the condition is **true**. (The `NOT IN` trap is [below](#exists-in-and-not-exists).)

```sql
-- ❌ always returns no rows, even when cancelled_at is NULL
SELECT id FROM appointments WHERE cancelled_at = NULL;

-- ✅
SELECT id FROM appointments WHERE cancelled_at IS NULL;

-- ❌ status <> 'paid' is unknown when status is NULL, so those rows silently drop out
SELECT id FROM invoices WHERE status <> 'paid';

-- ✅ treats NULL as "a different value"
SELECT id FROM invoices WHERE status IS DISTINCT FROM 'paid';
```

| Engine | Null-safe "not equal" | Null-safe "equal" |
| --- | --- | --- |
| PostgreSQL | `IS DISTINCT FROM` | `IS NOT DISTINCT FROM` |
| SQL Server 2022+ | `IS DISTINCT FROM` | `IS NOT DISTINCT FROM` |
| MySQL | `NOT (a <=> b)` | `a <=> b` |

* * *

## Joins

### Join on keys

Join primary key to foreign key, with an index on the foreign key column (Postgres and SQL Server don't add one, see [Schema Design](/Database-Standards/Schema-Design)).

```sql
-- ❌ joins on a name: no index, not unique, and two customers called "Ali Khan" get mixed up
SELECT o.id, o.total, c.name
FROM   orders AS o
JOIN   customers AS c ON c.name = o.customer_name
WHERE  o.created_at >= '2026-01-01';

-- ✅ joins on the key, selects only what it needs
SELECT o.id, o.total, c.name
FROM   orders AS o
JOIN   customers AS c ON c.id = o.customer_id
WHERE  o.created_at >= '2026-01-01'
  AND  o.status = 'paid';
```

No need to "filter early" by moving the `WHERE` into a CTE or subquery: the planner already applies filters before the join when it can. That rewrite changes readability, not speed.

### The join order myth

"Join the smaller table first" dates from databases that joined in the order written. Postgres, SQL Server and MySQL have **cost-based planners** that reorder joins from table statistics, so `FROM` order usually makes no difference.

> 📝 **Postgres limit:** the planner fully reorders up to 8 joined tables (`join_collapse_limit` and `from_collapse_limit`, default 8). Past that, it keeps the order you wrote for the rest, and from 12 tables (`geqo_threshold`) it switches to a faster, approximate search. A query joining that many tables usually should be split or turned into a view anyway.

When the planner does pick a bad order, the cause is almost always **stale or missing statistics** (`ANALYZE` the table). See [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics).

### DISTINCT is a smell

Needing `DISTINCT` to remove duplicates usually means a join is multiplying rows.

```sql
-- ❌ one row per matching invoice, then DISTINCT throws the duplicates away again
SELECT DISTINCT c.id, c.name
FROM   customers AS c
JOIN   invoices AS i ON i.customer_id = c.id
WHERE  i.status = 'overdue';

-- ✅ asks the real question: customers that have at least one overdue invoice
SELECT c.id, c.name
FROM   customers AS c
WHERE  EXISTS (
    SELECT 1 FROM invoices AS i
    WHERE  i.customer_id = c.id
      AND  i.status = 'overdue'
);
```

* * *

## Subqueries

### EXISTS IN and NOT EXISTS

For "has at least one", `IN (subquery)` and `EXISTS` get the same semi-join plan on Postgres. Pick whichever reads better.

```sql
-- both fine
SELECT id, name FROM customers WHERE id IN (SELECT customer_id FROM invoices WHERE status = 'overdue');
SELECT id, name FROM customers c WHERE EXISTS (SELECT 1 FROM invoices i WHERE i.customer_id = c.id AND i.status = 'overdue');
```

For "has **none**", always `NOT EXISTS`:

```sql
-- ❌ if ANY invoices.customer_id is NULL, this returns zero rows. Not fewer: zero.
SELECT id, name FROM customers
WHERE  id NOT IN (SELECT customer_id FROM invoices);

-- ✅ correct with or without NULLs, and planned as an anti-join
SELECT c.id, c.name FROM customers AS c
WHERE  NOT EXISTS (SELECT 1 FROM invoices AS i WHERE i.customer_id = c.id);
```

Why: `id NOT IN (1, 2, NULL)` means `id <> 1 AND id <> 2 AND id <> NULL`, and `id <> NULL` is never true. Postgres also can't turn `NOT IN (subquery)` into an anti-join, so it's slower even without NULLs.

> 📝 `LEFT JOIN invoices i ON ... WHERE i.id IS NULL` is also a correct anti-join. `NOT EXISTS` says what you mean more directly.

### Correlated subqueries

A subquery that refers to the outer row can run once per row.

```sql
-- ❌ the COUNT runs again for every customer
SELECT c.name
FROM   customers AS c
WHERE  (SELECT COUNT(*) FROM orders AS o WHERE o.customer_id = c.id) > 10;

-- ✅ count once per customer, in one pass over orders
SELECT c.name
FROM   customers AS c
JOIN  (SELECT customer_id
       FROM   orders
       GROUP  BY customer_id
       HAVING COUNT(*) > 10) AS busy ON busy.customer_id = c.id;
```

`EXISTS` subqueries are correlated too, but fine: the planner turns them into joins.

### OR or UNION

Rewriting `OR` as `UNION` is usually unnecessary on Postgres: with an index on each column, the planner combines them with a **BitmapOr**.

```sql
-- start with this; with indexes on role and on status, Postgres can use both
SELECT id, email FROM users WHERE role = 'admin' OR status = 'locked';
```

Rewrite only if `EXPLAIN` shows a sequential scan on a big table **and** the rewrite measurably helps:

```sql
SELECT id, email FROM users WHERE role = 'admin'
UNION
SELECT id, email FROM users WHERE status = 'locked';
```

- `UNION` removes duplicates, at the cost of a sort or hash of the whole result. Needed here: an admin who is also locked would otherwise appear twice.
- `UNION ALL` skips that work. Use it whenever the halves can't overlap, or duplicates don't matter.

### CTEs subqueries and temp tables

| | CTE (`WITH ...`) | Subquery | Temp table |
| --- | --- | --- | --- |
| Lives for | One statement | One statement | The session (or transaction) |
| Stored? | Usually inlined into the query (Postgres 12+) | Inlined | Yes, written out as a real table |
| Can be indexed | No | No | Yes |
| Use it for | Breaking a long query into named, readable steps; recursion | A one-off value or filter | A big intermediate result used by several later statements |

**CTEs in Postgres 12+:** a non-recursive, side-effect-free CTE referenced **once** is inlined like a subquery, so it costs nothing. One referenced **more than once** is materialized (computed once and stored) by default. Override either way:

```sql
WITH overdue AS MATERIALIZED (      -- compute once, reuse the result
    SELECT customer_id, SUM(total) AS owed
    FROM   invoices
    WHERE  status = 'overdue'
    GROUP  BY customer_id
)
SELECT c.name, o.owed
FROM   customers AS c
JOIN   overdue AS o ON o.customer_id = c.id;
```

`NOT MATERIALIZED` inlines it even when referenced twice. Before Postgres 12, **every** CTE was materialized: that's where "CTEs are slow" comes from.

| Engine | How a CTE runs |
| --- | --- |
| PostgreSQL 12+ | As above. `MATERIALIZED` / `NOT MATERIALIZED` override it. |
| SQL Server | Always inlined. A CTE referenced twice **runs twice**, and there's no hint to change that. If it's expensive and reused, use a temp table. |
| MySQL 8.0+ | The optimizer merges or materializes it. The `MERGE` / `NO_MERGE` optimizer hints override it. |

Name each CTE after what it holds (`overdue_invoices`, not `cte1`). Chaining several is fine if each reads as a single step.

#### Recursive CTEs

```sql
WITH RECURSIVE reports AS (
    SELECT id, manager_id, name, 1 AS depth          -- seed: runs once
    FROM   staff
    WHERE  id = 42
    UNION ALL
    SELECT s.id, s.manager_id, s.name, r.depth + 1   -- step: repeats on the rows the last round added
    FROM   staff   AS s
    JOIN   reports AS r ON s.manager_id = r.id
    WHERE  r.depth < 20                              -- stop condition
)
SELECT id, name, depth
FROM   reports;
```

- **Always give it a stop condition** (depth limit or cycle check). Bad data with a loop (A manages B, B manages A) otherwise runs until the query is killed. On Postgres 14+, `) CYCLE id SET is_cycle USING path` does the cycle check for you.
- **`UNION ALL`, not `UNION`**, unless you need duplicates removed: `UNION` de-duplicates on every round.
- The full tree recipe, with breadcrumbs, is in the [SQL Cookbook](/Database-Standards/SQL-Cookbook).

| Engine | Keyword | Recursion limit |
| --- | --- | --- |
| PostgreSQL | `WITH RECURSIVE` | None. Runs until done, or until `statement_timeout`. |
| SQL Server | Plain `WITH` (`RECURSIVE` is a syntax error) | 100 levels by default, then an error. `OPTION (MAXRECURSION n)` raises it (0 = no limit). |
| MySQL 8.0+ | `WITH RECURSIVE` | `cte_max_recursion_depth`, 1000 by default. |

#### Writing with CTEs

Postgres only: a CTE can `INSERT`, `UPDATE` or `DELETE` and pass rows on with `RETURNING`: one statement, one round trip, atomic.

```sql
-- create an invoice and its lines in one statement
WITH new_invoice AS (
    INSERT INTO invoices (customer_id, number, total)
    VALUES ($1, $2, $3)
    RETURNING id
)
INSERT INTO invoice_lines (invoice_id, description, amount)
SELECT     n.id, l.description, l.amount
FROM       new_invoice AS n
CROSS JOIN unnest($4::text[], $5::numeric[]) AS l (description, amount);
```

> ⚠️ Every part of the statement runs against the **same snapshot**. One part can't see another part's changes, except through `RETURNING`. Never modify the same row in two parts: Postgres doesn't define which change wins.

SQL Server and MySQL can't modify data inside a CTE: use separate statements in a transaction. On SQL Server, `OUTPUT ... INTO` passes rows from one statement to the next.

**Temp tables** pay off when a large intermediate result is queried several times, e.g. in a reporting script:

```sql
CREATE TEMP TABLE active_order_counts AS
SELECT customer_id, COUNT(*) AS order_count
FROM   orders
WHERE  status = 'active'
GROUP  BY customer_id;

CREATE INDEX ON active_order_counts (customer_id);
ANALYZE active_order_counts;   -- autovacuum never analyzes temp tables; without this the planner is guessing
```

| Engine | Temp table |
| --- | --- |
| PostgreSQL | `CREATE TEMP TABLE`, dropped at the end of the session (or the transaction, with `ON COMMIT DROP`) |
| SQL Server | `#name` (session) or `##name` (global), stored in `tempdb`. `SELECT ... INTO #name` creates one. |
| MySQL | `CREATE TEMPORARY TABLE` |

* * *

## The N plus one problem

```ts
// ❌ 1 query for the invoices + 1 per invoice for its customer = 51 queries for 50 invoices
const invoices = await prisma.invoice.findMany({ take: 50 });
for (const inv of invoices) {
  inv.customer = await prisma.customer.findUnique({ where: { id: inv.customerId } });
}

// ✅ the related rows come with the list
const invoices = await prisma.invoice.findMany({
  take: 50,
  include: { customer: true },
});
```

| Tool | Load related rows with the list | Watch out for |
| --- | --- | --- |
| Prisma | `include: { customer: true }`, or `select` with the relation | A `findUnique` inside a loop |
| TypeORM | `find({ relations: { customer: true } })` or `.leftJoinAndSelect('invoice.customer', 'customer')` | Lazy relations (`Promise<Customer>`) load on access, one query each |
| Eloquent | `Invoice::with('customer')->get()` | Touching `$invoice->customer` in a Blade loop without `with()` |
| EF Core | `.Include(i => i.Customer)`; add `.AsSplitQuery()` when including several collections | Lazy-loading proxies. Several collection `Include`s in one query multiply rows (a "cartesian explosion"). |
| Dapper | One query with a `JOIN` and multi-mapping, or `QueryMultiple` | A query inside a `foreach` |

> 💡 **Laravel can refuse lazy loading in development**, so N+1 shows up as an error instead of a slow page. In `AppServiceProvider::boot()`:
> ```php
> Model::preventLazyLoading(! $this->app->isProduction());
> ```

Dapper, both ways:

```csharp
// one row per invoice, with its customer mapped onto it
var invoices = await conn.QueryAsync<Invoice, Customer, Invoice>(
    """
    SELECT i.id, i.number, i.total, c.id, c.name
    FROM   invoices AS i
    JOIN   customers AS c ON c.id = i.customer_id
    WHERE  i.status = @Status
    """,
    (invoice, customer) => { invoice.Customer = customer; return invoice; },
    new { Status = "sent" },
    splitOn: "id");   // the second "id" column starts the Customer

// an invoice and its lines: two result sets, one round trip
using var multi = await conn.QueryMultipleAsync(
    """
    SELECT id, number, total FROM invoices WHERE id = @Id;
    SELECT id, quantity, unit_price FROM invoice_lines WHERE invoice_id = @Id;
    """,
    new { Id = id });
var invoice = await multi.ReadSingleAsync<Invoice>();
invoice.Lines = (await multi.ReadAsync<InvoiceLine>()).ToList();
```

* * *

## Pagination

`LIMIT 20 OFFSET 100000` finds, sorts and **discards** 100,000 rows before returning 20: page 1 is fast, page 5,000 isn't. Rows inserted or deleted between requests also shift pages, so users see duplicates or miss rows.

**Keyset (seek) pagination** continues from where the last page ended:

```sql
-- first page
SELECT id, number, total, created_at
FROM   invoices
ORDER  BY created_at DESC, id DESC
LIMIT  20;

-- next page: pass the created_at and id of the last row you received
SELECT id, number, total, created_at
FROM   invoices
WHERE  (created_at, id) < ($1, $2)
ORDER  BY created_at DESC, id DESC
LIMIT  20;
```

```sql
-- the index that makes every page as fast as the first
CREATE INDEX ix_invoices_created_at_id ON invoices (created_at DESC, id DESC);
```

- The `ORDER BY` must be **unique**, so add `id` as the tie-breaker. Otherwise two invoices created in the same microsecond get skipped or repeated.
- `(created_at, id) < ($1, $2)` is a row-value comparison: `created_at` first, `id` only on a tie, and the index serves both.

| Use `OFFSET` | Use keyset |
| --- | --- |
| Admin tables with numbered pages and small data | Feeds, infinite scroll, APIs, exports, anything that grows |
| Users really need to jump to page 37 | "Next" and "previous" are enough |

| Engine | Keyset condition |
| --- | --- |
| PostgreSQL, MySQL 8 | `WHERE (created_at, id) < (?, ?)` |
| SQL Server | No row-value comparison: `WHERE created_at < @c OR (created_at = @c AND id < @id)` |

> 💡 Prisma's cursor pagination (`cursor: { id }, skip: 1`) works the same way when you sort by a unique column. The opaque cursor you return to the client can simply be the last row's `created_at` and `id`, base64-encoded.

### Counting rows

On Postgres, `SELECT COUNT(*) FROM invoices` scans the table (or an index), because each transaction can see a different set of rows. On millions of rows, that's slow on every page load.

- **"Is there a next page?"** Fetch `LIMIT 21` and show 20. If the 21st exists, there's a next page.
- **A rough total** ("about 1.2 million") is in the table statistics:
  ```sql
  SELECT reltuples::bigint AS estimated_rows FROM pg_class WHERE oid = 'invoices'::regclass;
  ```
- An exact count of a **filtered** set (`WHERE customer_id = 42`) is fine when an index narrows it down.

> 📝 `COUNT(*)` counts rows. `COUNT(column)` counts non-NULL values of that column. `COUNT(1)` is not faster than `COUNT(*)`.

* * *

## Searching text

| Search | Index that helps | Notes |
| --- | --- | --- |
| Exact: `email = $1` | Normal B-tree | |
| Prefix: `number LIKE 'INV-2026%'` | B-tree, **with** `text_pattern_ops` or a `"C"` collation on Postgres | A normal B-tree on a non-C collation can't serve `LIKE` |
| Contains: `name ILIKE '%khan%'` | Trigram index (`pg_trgm` extension, GIN) | A B-tree can never help a leading `%` |
| Words in prose: product descriptions, notes | Full-text search (`tsvector` + GIN) | Handles stemming ("running" finds "run") and ranking |

```sql
-- ❌ cuts the column up, so no index on order_code can be used
SELECT id FROM orders WHERE left(order_code, 3) = 'INV';

-- ✅ a prefix LIKE, with an index that supports it
CREATE INDEX ix_orders_order_code_pattern ON orders (order_code text_pattern_ops);
SELECT id FROM orders WHERE order_code LIKE 'INV%';
```

Creating trigram and full-text indexes: [Indexing](/Database-Standards/Indexing) and [PostgreSQL](/Database-Standards/PostgreSQL).

* * *

## SQL features worth knowing

### Conditional aggregates with FILTER

```sql
-- ✅ one pass over the table, several counts
SELECT customer_id,
       COUNT(*)                                 AS invoices,
       COUNT(*) FILTER (WHERE status = 'paid')  AS paid,
       SUM(total) FILTER (WHERE status = 'overdue') AS overdue_amount
FROM   invoices
GROUP  BY customer_id;
```

`FILTER` is Postgres. On SQL Server and MySQL, write `SUM(CASE WHEN status = 'paid' THEN 1 ELSE 0 END)`.

### Window functions

Compute across related rows **without** collapsing them the way `GROUP BY` does.

```sql
SELECT id,
       customer_id,
       total,
       SUM(total)   OVER (PARTITION BY customer_id ORDER BY created_at) AS running_total,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS recency,
       LAG(total)   OVER (PARTITION BY customer_id ORDER BY created_at) AS previous_total
FROM   invoices;
```

`ROW_NUMBER`, `RANK`, `LAG`/`LEAD`, and running `SUM`/`AVG` cover most reporting needs, on Postgres, SQL Server and MySQL 8+. Recipes: [SQL Cookbook](/Database-Standards/SQL-Cookbook).

### LATERAL for top N per row

```sql
-- each customer with their three most recent invoices
SELECT c.name, recent.number, recent.total
FROM   customers AS c
CROSS  JOIN LATERAL (
    SELECT number, total
    FROM   invoices AS i
    WHERE  i.customer_id = c.id
    ORDER  BY i.created_at DESC
    LIMIT  3
) AS recent;
```

SQL Server spells it `CROSS APPLY` (or `OUTER APPLY` to keep customers with no invoices). MySQL supports `LATERAL` from 8.0.14.

### RETURNING

Get back what you just wrote, in the same statement:

```sql
INSERT INTO invoices (customer_id, number, total)
VALUES ($1, $2, $3)
RETURNING id, public_id, created_at;

UPDATE invoices SET status = 'void' WHERE id = $1 RETURNING id, status, updated_at;
```

| Engine | Equivalent |
| --- | --- |
| PostgreSQL | `RETURNING` on `INSERT`, `UPDATE`, `DELETE` (and `MERGE` from Postgres 17) |
| SQL Server | `OUTPUT inserted.id, inserted.created_at` |
| MySQL | None. `LAST_INSERT_ID()` for the new key, then a `SELECT`. |

### Upsert with ON CONFLICT

Insert, or update if the row exists, in one atomic statement:

```sql
INSERT INTO product_stock (product_id, warehouse_id, quantity)
VALUES ($1, $2, $3)
ON CONFLICT (product_id, warehouse_id)
DO UPDATE SET quantity   = product_stock.quantity + EXCLUDED.quantity,
              updated_at = now();
```

- The conflict target needs a **unique constraint or unique index** on exactly those columns.
- `EXCLUDED` is the row you tried to insert. `DO NOTHING` skips it instead, useful for idempotent imports.

| Engine | Upsert |
| --- | --- |
| PostgreSQL | `INSERT ... ON CONFLICT` (9.5+); `MERGE` exists from 15 but `ON CONFLICT` is the safer choice under concurrency |
| MySQL | `INSERT ... ON DUPLICATE KEY UPDATE quantity = quantity + new.quantity` with a row alias (`VALUES (...) AS new`, 8.0.19+) |
| SQL Server | No `ON CONFLICT`. `MERGE` needs `WITH (HOLDLOCK)` to be safe under concurrency, or use `UPDATE` then `INSERT` inside a transaction with `UPDLOCK, HOLDLOCK` |

Why app-side "check, then insert" isn't safe, and locking under concurrent writes: [Transactions and Concurrency](/Database-Standards/Transactions-and-Concurrency).

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Select the columns you need | `SELECT *` in application code | [Columns](#select-only-what-you-need) |
| Half-open ranges on the raw column | Functions or casts on the filtered column | [Index-friendly filters](#keep-filters-index-friendly) |
| Match parameter types to column types | Let Dapper send `nvarchar` to a `varchar` column | [Type mismatches](#match-the-column-type) |
| `IS NULL`, `IS DISTINCT FROM` | `= NULL`, or `<>` on a nullable column | [NULL](#null-in-comparisons) |
| Join on keys with an index on the foreign key | Join on names or other non-key text | [Joins](#join-on-keys) |
| Trust the planner's join order, `ANALYZE` when it's wrong | Reorder tables by size | [Join order](#the-join-order-myth) |
| `EXISTS` for "has any" | `JOIN` + `DISTINCT` to remove duplicates | [DISTINCT](#distinct-is-a-smell) |
| `NOT EXISTS` for "has none" | `NOT IN (subquery)` | [NOT EXISTS](#exists-in-and-not-exists) |
| `GROUP BY` once, then join | A correlated `COUNT(*)` per row | [Correlated subqueries](#correlated-subqueries) |
| `OR`, then `UNION` only if the plan proves it | Rewrite every `OR` as `UNION` | [OR or UNION](#or-or-union) |
| CTEs for readability, temp tables + `ANALYZE` for big reused results | Assume CTEs are slow (pre-Postgres 12 advice) | [CTEs](#ctes-subqueries-and-temp-tables) |
| Give every recursive CTE a stop condition | Trust the data never to contain a loop | [Recursive CTEs](#recursive-ctes) |
| `include`, `relations`, `with()`, `Include()`, Dapper multi-mapping | A query inside a loop | [N+1](#the-n-plus-one-problem) |
| Keyset pagination with a unique sort and a matching index | Deep `OFFSET` on a growing table | [Pagination](#pagination) |
| `LIMIT n + 1` to detect a next page | `COUNT(*)` a big table on every request | [Counting](#counting-rows) |
| Trigram or full-text index for "contains" searches | `LIKE '%term%'` on a big table with no index | [Text search](#searching-text) |
| `ON CONFLICT` for upserts | Check-then-insert in app code | [Upsert](#upsert-with-on-conflict) |

* * *

## Further reading

1.  **Use The Index, Luke — Obfuscated Conditions** (functions, casts and other filters that hide the column from the index)
    [https://use-the-index-luke.com/sql/where-clause/obfuscation](https://use-the-index-luke.com/sql/where-clause/obfuscation)
2.  **Use The Index, Luke — No Offset** (keyset pagination)
    [https://use-the-index-luke.com/no-offset](https://use-the-index-luke.com/no-offset)
3.  **PostgreSQL Documentation — WITH Queries (Common Table Expressions)**
    [https://www.postgresql.org/docs/current/queries-with.html](https://www.postgresql.org/docs/current/queries-with.html)
4.  **PostgreSQL Documentation — Subquery Expressions** (EXISTS, IN, NOT IN and NULLs)
    [https://www.postgresql.org/docs/current/functions-subquery.html](https://www.postgresql.org/docs/current/functions-subquery.html)
5.  **PostgreSQL Documentation — INSERT, including ON CONFLICT**
    [https://www.postgresql.org/docs/current/sql-insert.html](https://www.postgresql.org/docs/current/sql-insert.html)
6.  **Prisma — Relation queries** (`include` and nested reads)
    [https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries](https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries)
7.  **EF Core — Single vs. Split Queries**
    [https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries)
8.  **Laravel — Eloquent eager loading**
    [https://laravel.com/docs/eloquent-relationships#eager-loading](https://laravel.com/docs/eloquent-relationships#eager-loading)

* * *

## Related pages

- [Indexing](/Database-Standards/Indexing) — the indexes these queries rely on.
- [Performance and Diagnostics](/Database-Standards/Performance-and-Diagnostics) — reading `EXPLAIN` to prove a rewrite helped.
- [SQL Cookbook](/Database-Standards/SQL-Cookbook) — ready-made queries for common reporting problems.
- [General Principles → Always parameterize](/Database-Standards/General-Principles#always-parameterize) — every value in these examples goes in as a parameter.
