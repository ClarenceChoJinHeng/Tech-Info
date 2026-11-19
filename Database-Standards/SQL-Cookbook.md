SQL Cookbook
============

> Ready-to-use recipes for questions that come up again and again. Read **[General Principles](/Database-Standards/General-Principles)** first. Every recipe assumes parameters for user input, and a look at the plan with `EXPLAIN` before you ship it on a big table.
> Written for **PostgreSQL**. Where SQL Server or MySQL needs something different, the recipe says so in one line.

* * *

## At a Glance

| I need to… | Recipe |
| --- | --- |
| [Get the top 3 rows for each customer](#top-n-per-group) | **`row_number()` in a subquery, or `LATERAL` with `LIMIT`**<br/><sub>`LATERAL` wins when there's an index and each group is big.</sub> |
| [Get only the latest row for each customer](#latest-row-per-customer) | **`DISTINCT ON (customer_id)`**<br/><sub>Postgres-only shortcut. Elsewhere, `row_number() = 1`.</sub> |
| [Find rows that have no match](#rows-with-no-match) | **`NOT EXISTS`, never `NOT IN`**<br/><sub>One NULL in a `NOT IN` list makes the whole query return nothing.</sub> |
| [Pick random rows](#random-sample) | **`ORDER BY random()` on small tables, `TABLESAMPLE` on big ones**<br/><sub>`ORDER BY random()` sorts the entire table first.</sub> |
| [Find duplicates](#find-duplicates) | **`GROUP BY` the duplicate key, `HAVING count(*) > 1`**<br/><sub>`array_agg(id)` shows you exactly which rows they are.</sub> |
| [Delete duplicates and keep one](#delete-duplicates-and-keep-one) | **`row_number()` per duplicate key, delete everything after the first**<br/><sub>Then add a unique index so it can't happen again.</sub> |
| [Running totals and moving averages](#running-totals-and-moving-averages) | **`sum(...) OVER (ORDER BY ...)`**<br/><sub>Use an explicit `ROWS` frame for moving windows.</sub> |
| [Show each row as a percent of the total](#percent-of-total) | **`x / sum(x) OVER ()`**<br/><sub>One pass, no second query for the total.</sub> |
| [Rank rows](#ranking) | **`row_number`, `rank` or `dense_rank`**<br/><sub>They only differ in how they treat ties.</sub> |
| [Compare with last month](#compare-with-the-previous-row) | **`lag()` and `lead()`**<br/><sub>Reads the previous or next row without a self-join.</sub> |
| [Median and percentiles](#median-and-percentiles) | **`percentile_cont(0.5) WITHIN GROUP (ORDER BY ...)`**<br/><sub>The average lies when a few values are huge. The median doesn't.</sub> |
| [Group by day in the clinic's time zone](#group-by-day-in-local-time) | **`AT TIME ZONE` before you truncate**<br/><sub>Otherwise "today" ends at midnight UTC.</sub> |
| [Show zero for days with no data](#fill-missing-dates-for-a-chart) | **`generate_series` of dates, `LEFT JOIN` the data**<br/><sub>Charts get every day, not just the days something happened.</sub> |
| [Find streaks of consecutive days](#streaks-of-consecutive-days) | **Date minus `row_number()` groups each streak**<br/><sub>The classic "gaps and islands" trick.</sub> |
| [Find or prevent double bookings](#overlapping-time-ranges) | **`a.start < b.end AND b.start < a.end`, or an exclusion constraint**<br/><sub>The constraint makes a double booking impossible, not just detectable.</sub> |
| [Count by status in one row](#count-by-status-in-one-row) | **`count(*) FILTER (WHERE ...)`**<br/><sub>One scan, one row per group, one column per status.</sub> |
| [Walk a tree or hierarchy](#tree-and-hierarchy-queries) | **`WITH RECURSIVE`, with a cycle guard**<br/><sub>Categories, org charts, threaded comments.</sub> |
| [Turn rows into a comma-separated list](#comma-separated-lists) | **`string_agg(name, ', ' ORDER BY name)`**<br/><sub>Always give it an `ORDER BY`, or the order is random.</sub> |
| [Build JSON for an API in SQL](#build-json-for-an-api) | **`jsonb_build_object` and `jsonb_agg`**<br/><sub>One query returns the invoice with its lines nested inside.</sub> |
| [Insert, or update if it exists](#insert-or-update) | **`INSERT ... ON CONFLICT ... DO UPDATE`**<br/><sub>Atomic. Select-then-insert isn't.</sub> |
| [Get back the rows I just changed](#get-back-what-you-changed) | **`RETURNING`**<br/><sub>No second query to find out what you inserted, updated or deleted.</sub> |
| [Insert, or get the existing id](#insert-or-get-the-existing-id) | **`ON CONFLICT DO NOTHING RETURNING id`, then fall back to a `SELECT`**<br/><sub>Handles the "already exists" case in one round trip.</sub> |
| [Divide without crashing](#safe-division) | **`x / NULLIF(y, 0)`**<br/><sub>Division by zero becomes NULL instead of an error.</sub> |
| [Search ignoring case](#case-insensitive-search) | **Index `lower(column)`, or `pg_trgm` for "contains"**<br/><sub>`ILIKE '%x%'` needs a trigram index to be fast.</sub> |
| [Search across several columns](#search-across-several-columns) | **One generated search column with one index**<br/><sub>Instead of an `OR` across columns that no index can serve.</sub> |

* * *

## Concepts you'll meet on this page

### Window function

A function over rows related to the current row that, unlike `GROUP BY`, **doesn't collapse them**. In `row_number() OVER (PARTITION BY customer_id ORDER BY created_at DESC)`, `PARTITION BY` is "per group" and `ORDER BY` is "in this order".

### CTE

A **Common Table Expression**: a named subquery written up front with `WITH name AS (...)`, so a long query reads top to bottom. `WITH RECURSIVE` lets it refer to itself, which is how you walk trees.

### LATERAL

A subquery in `FROM` that can refer to columns of earlier tables: "for each customer, run this little query", usually with `ORDER BY ... LIMIT` inside.

### The example schema

The clinic booking app: `customers`, `invoices`, `invoice_lines`, `products`, `categories`, `payments`, and `appointments (id, customer_id, practitioner_id, starts_at, ends_at, status)`. Keys are `bigint`, timestamps `timestamptz`, money `numeric(12,2)`.

* * *

## Picking rows

### Top N per group

**Problem:** the 3 most recent invoices for each customer.

```sql
-- Option 1: number the rows per customer, keep the first 3
SELECT customer_id, id, number, total, created_at
FROM (
    SELECT i.*,
           row_number() OVER (PARTITION BY i.customer_id
                              ORDER BY i.created_at DESC, i.id DESC) AS rn
    FROM   invoices AS i
) AS ranked
WHERE  rn <= 3
ORDER  BY customer_id, rn;

-- Option 2: for each customer, fetch its 3 newest invoices
SELECT c.id AS customer_id, c.name, recent.number, recent.total, recent.created_at
FROM   customers AS c
CROSS  JOIN LATERAL (
    SELECT i.number, i.total, i.created_at
    FROM   invoices AS i
    WHERE  i.customer_id = c.id
    ORDER  BY i.created_at DESC, i.id DESC
    LIMIT  3
) AS recent;

-- what makes option 2 fast: each customer's lookup is a short index read
CREATE INDEX invoices_customer_id_created_at_idx
    ON invoices (customer_id, created_at DESC, id DESC);
```

**Why it works:** option 1 numbers every invoice in one read; option 2 does one tiny index lookup per customer, so it wins when customers have many invoices and the index exists. Use `LEFT JOIN LATERAL (...) AS recent ON true` to keep customers with no invoices.

> 💡 Always add a tiebreaker (`id DESC`) to the `ORDER BY`. Otherwise two invoices with the same `created_at` come back in a different order each run, and paging breaks.

**Other engines:** SQL Server uses `CROSS APPLY` / `OUTER APPLY` with `TOP (3)` instead of `LATERAL`. MySQL supports `row_number()` from 8.0 and `LATERAL` from 8.0.14.

### Latest row per customer

**Problem:** only the newest invoice for each customer.

```sql
SELECT DISTINCT ON (customer_id)
       customer_id, id, number, total, created_at
FROM   invoices
ORDER  BY customer_id, created_at DESC, id DESC;
```

**Why it works:** `DISTINCT ON (customer_id)` keeps the **first** row of each `customer_id` in `ORDER BY` order, so the `ORDER BY` must start with the same columns.

**Other engines:** `DISTINCT ON` is Postgres-only. Use option 1 from [Top N per group](#top-n-per-group) with `rn = 1`.

### Rows with no match

**Problem:** customers who have never booked an appointment.

```sql
-- ✅
SELECT c.id, c.name
FROM   customers AS c
WHERE  NOT EXISTS (
    SELECT 1
    FROM   appointments AS a
    WHERE  a.customer_id = c.id
);

-- ❌ if any appointments.customer_id is NULL, this returns no rows at all
SELECT id, name
FROM   customers
WHERE  id NOT IN (SELECT customer_id FROM appointments);
```

**Why it works:** `x NOT IN (1, 2, NULL)` is never true, because comparing with NULL gives "unknown". `NOT EXISTS` has no such trap and runs as an efficient anti-join; `LEFT JOIN ... WHERE a.id IS NULL` is fine too.

**Other engines:** same everywhere.

### Random sample

**Problem:** 20 random customers to spot-check, or roughly 1% of a big table.

```sql
-- small table or a one-off: simple, but it sorts the whole table first
SELECT id, name FROM customers ORDER BY random() LIMIT 20;

-- big table: about 1% of rows, reading only about 1% of the table's pages
SELECT id, name FROM customers TABLESAMPLE SYSTEM (1);

-- big table, more evenly random: about 1% of rows, reads the whole table but doesn't sort it
SELECT id, name FROM customers TABLESAMPLE BERNOULLI (1) REPEATABLE (42);
```

**Why it works:** `SYSTEM` picks whole pages (fast, but rows stored together come together); `BERNOULLI` picks each row independently. `REPEATABLE (seed)` returns the same sample every time, for rerunning a check.

**Other engines:** SQL Server: `TABLESAMPLE (1 PERCENT)` or `ORDER BY NEWID()`. MySQL: `ORDER BY RAND() LIMIT 20`, with the same full-sort cost.

* * *

## Duplicates

### Find duplicates

**Problem:** customers entered twice with the same email.

```sql
SELECT lower(email)              AS email,
       count(*)                  AS copies,
       array_agg(id ORDER BY id) AS ids
FROM   customers
GROUP  BY lower(email)
HAVING count(*) > 1
ORDER  BY copies DESC;
```

**Why it works:** group by whatever makes two rows "the same" (here, email ignoring case) and keep groups with more than one row; `array_agg(id)` lists the ids to inspect.

**Other engines:** SQL Server: `STRING_AGG(CAST(id AS varchar(20)), ',')`. MySQL: `GROUP_CONCAT(id ORDER BY id)`.

### Delete duplicates and keep one

**Problem:** remove duplicate customers, keeping the oldest record of each.

```sql
BEGIN;

DELETE FROM customers
WHERE  id IN (
    SELECT id
    FROM (
        SELECT id,
               row_number() OVER (PARTITION BY lower(email) ORDER BY id) AS rn
        FROM   customers
    ) AS numbered
    WHERE  rn > 1
);
-- check the row count matches what "Find duplicates" told you, then:
COMMIT;

-- stop it happening again
CREATE UNIQUE INDEX customers_email_lower_key ON customers (lower(email));
```

**Why it works:** `row_number()` numbers each email's rows oldest first, and everything after row 1 is deleted.

> ⚠️ If other tables point at the duplicates (`invoices.customer_id`, `appointments.customer_id`), move those references to the row you're keeping **first**, in the same transaction. Otherwise the foreign key blocks the delete, or `ON DELETE CASCADE` silently deletes the duplicate's invoices with it.

**A table with no key at all** (a raw import, for example) can use `ctid`, the row's physical location:

```sql
DELETE FROM import_rows AS a
USING  import_rows AS b
WHERE  a.external_ref = b.external_ref
  AND  a.ctid > b.ctid;
```

> 📝 `ctid` changes when a row is updated or the table is rewritten. It's safe within one statement like this one. Never store it or use it as an id.

**Other engines:** SQL Server can delete straight through the CTE: `WITH d AS (SELECT ROW_NUMBER() OVER (...) AS rn FROM customers) DELETE FROM d WHERE rn > 1;`. MySQL 8: `DELETE c FROM customers c JOIN (subquery with ROW_NUMBER) d ON d.id = c.id WHERE d.rn > 1;`.

* * *

## Running numbers

### Running totals and moving averages

**Problem:** daily revenue, the running total so far, and a 7-day moving average.

```sql
SELECT paid_at::date                                   AS day,
       sum(amount)                                     AS revenue,
       sum(sum(amount)) OVER (ORDER BY paid_at::date)  AS running_total,
       round(avg(sum(amount)) OVER (ORDER BY paid_at::date
                                    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS avg_7_days
FROM   payments
GROUP  BY paid_at::date
ORDER  BY day;
```

**Why it works:** `GROUP BY` makes one row per day, then the window functions run over those rows: `sum(sum(amount))` is "the window sum of the daily sums", and `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` is "this row and the 6 before it".

> ⚠️ "The 6 rows before" is only "the 6 days before" when every day has a row. Combine with [Fill missing dates for a chart](#fill-missing-dates-for-a-chart) if some days have no payments.

> ⚠️ Without a frame, an ordered window defaults to `RANGE`, which treats rows with the same `ORDER BY` value as one step. For a per-row running total where values can tie, write `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` explicitly.

> 📝 `paid_at::date` uses the session's time zone. For a clinic's local days, see [Group by day in local time](#group-by-day-in-local-time).

**Other engines:** same syntax in SQL Server (2012+) and MySQL (8.0+).

### Percent of total

**Problem:** each category's share of revenue.

```sql
SELECT p.category_id,
       sum(l.quantity * l.unit_price) AS revenue,
       round(100.0 * sum(l.quantity * l.unit_price)
                   / NULLIF(sum(sum(l.quantity * l.unit_price)) OVER (), 0), 1) AS pct_of_total
FROM   invoice_lines AS l
JOIN   products      AS p ON p.id = l.product_id
GROUP  BY p.category_id
ORDER  BY revenue DESC;
```

**Why it works:** an empty `OVER ()` means "across all rows of the result", so the grand total comes in the same pass. `NULLIF(..., 0)` avoids division by zero on an empty period.

**Other engines:** same in SQL Server and MySQL 8.0+.

### Ranking

**Problem:** rank practitioners by appointments this month.

```sql
SELECT practitioner_id,
       count(*)                                     AS appointments,
       row_number() OVER (ORDER BY count(*) DESC)   AS row_number,
       rank()       OVER (ORDER BY count(*) DESC)   AS rank,
       dense_rank() OVER (ORDER BY count(*) DESC)   AS dense_rank
FROM   appointments
WHERE  starts_at >= date_trunc('month', now())
GROUP  BY practitioner_id;
```

| appointments | `row_number` | `rank` | `dense_rank` |
| --- | --- | --- | --- |
| 100 | 1 | 1 | 1 |
| 90 | 2 | 2 | 2 |
| 90 | 3 | 2 | 2 |
| 80 | 4 | 4 | 3 |

**Which one:** `row_number` for exactly N rows, `rank` for "joint second, nobody third" like a sports table, `dense_rank` for "the top 3 distinct values".

**Other engines:** same in SQL Server and MySQL 8.0+.

### Compare with the previous row

**Problem:** monthly revenue, last month's revenue, and the growth between them.

```sql
WITH monthly AS (
    SELECT date_trunc('month', paid_at) AS month,
           sum(amount)                  AS revenue
    FROM   payments
    GROUP  BY date_trunc('month', paid_at)
)
SELECT month::date                 AS month,
       revenue,
       lag(revenue) OVER w         AS previous_month,
       round(100.0 * (revenue - lag(revenue) OVER w)
                   / NULLIF(lag(revenue) OVER w, 0), 1) AS growth_pct
FROM   monthly
WINDOW w AS (ORDER BY month)
ORDER  BY month;
```

**Why it works:** `lag(revenue)` reads the previous row in window order (`lead()` the next; `lag(revenue, 12)` the same month last year), and the first month gets NULL. `WINDOW` names the window once instead of repeating it.

**Other engines:** `LAG`/`LEAD` work in SQL Server and MySQL 8.0+; the named `WINDOW` clause needs SQL Server 2022 or MySQL 8.0 (on older SQL Server, repeat `OVER (ORDER BY month)`).

### Median and percentiles

**Problem:** the typical invoice value, and the value 90% of invoices stay under.

```sql
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY total) AS median_total,
       percentile_cont(0.9) WITHIN GROUP (ORDER BY total) AS p90_total,
       percentile_disc(0.5) WITHIN GROUP (ORDER BY total) AS median_real_invoice
FROM   invoices
WHERE  status = 'paid';
```

**Why it works:** `percentile_cont` interpolates between the two middle values (returning `double precision`); `percentile_disc` returns an actual value from the data. Add `GROUP BY customer_id` for a median per customer, or use `percentile_cont(ARRAY[0.5, 0.9, 0.99])` for several percentiles in one pass.

> 💡 Use the median for "typical" values like invoice size or page load time. One huge invoice drags the average up but barely moves the median.

**Other engines:** SQL Server only has `PERCENTILE_CONT` as a window function: `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total) OVER ()`, with `DISTINCT` or `TOP 1`. MySQL has no percentile function: number the rows with `row_number()` and pick the middle one.

* * *

## Dates and time

### Group by day in local time

**Problem:** appointments per day in October, counted in the clinic's time zone, not UTC.

```sql
SELECT (a.starts_at AT TIME ZONE 'Europe/London')::date AS clinic_day,
       count(*)                                          AS appointments
FROM   appointments AS a
WHERE  a.starts_at >= timestamptz '2026-10-01 00:00 Europe/London'
  AND  a.starts_at <  timestamptz '2026-11-01 00:00 Europe/London'
GROUP  BY clinic_day
ORDER  BY clinic_day;
```

**Why it works:** `AT TIME ZONE 'Europe/London'` turns the stored instant into London wall-clock time before `::date` takes the day, while the `WHERE` compares the raw column against two instants so the `starts_at` index still works. Use IANA zone names (`Europe/London`), never fixed offsets like `+01`, so daylight saving is handled.

- **Weeks or months:** `date_trunc('week', a.starts_at AT TIME ZONE 'Europe/London')`. Weeks start on Monday.
- **Several clinics:** store each clinic's zone in a column and use `AT TIME ZONE c.time_zone`.

**Other engines:** SQL Server 2016+: `AT TIME ZONE 'GMT Standard Time'` (Windows zone names). MySQL: `CONVERT_TZ(starts_at, 'UTC', 'Europe/London')`, which needs the server's time zone tables loaded.

### Fill missing dates for a chart

**Problem:** revenue for every day of October, with 0 for days with no payments.

```sql
SELECT d.day::date                  AS day,
       coalesce(sum(p.amount), 0)   AS revenue
FROM   generate_series(date '2026-10-01', date '2026-10-31', interval '1 day') AS d(day)
LEFT   JOIN payments AS p
       ON  p.paid_at >= d.day
       AND p.paid_at <  d.day + interval '1 day'
GROUP  BY d.day
ORDER  BY d.day;
```

**Why it works:** `generate_series` makes one row per day, `LEFT JOIN` keeps every day, and `coalesce` turns "no payments" into 0. Joining on a range, not `paid_at::date`, lets the `paid_at` index work.

> 📝 Each generated day starts at midnight **in the session's time zone**. Run `SET TIME ZONE 'Europe/London';` first if the days should follow a clinic's calendar.

**Other engines:** SQL Server 2022 has `GENERATE_SERIES(0, 30)` for numbers, so use `DATEADD(day, value, '2026-10-01')`. Older SQL Server and MySQL use a recursive CTE, or a permanent calendar table.

### Streaks of consecutive days

**Problem:** for each practitioner, runs of consecutive days with at least one appointment.

```sql
WITH days AS (
    SELECT DISTINCT practitioner_id, starts_at::date AS day
    FROM   appointments
),
grouped AS (
    SELECT practitioner_id,
           day,
           day - (row_number() OVER (PARTITION BY practitioner_id ORDER BY day))::int AS streak_key
    FROM   days
)
SELECT practitioner_id,
       min(day) AS streak_start,
       max(day) AS streak_end,
       count(*) AS days_in_a_row
FROM   grouped
GROUP  BY practitioner_id, streak_key
ORDER  BY days_in_a_row DESC;
```

**Why it works:** within a streak, date and row number both go up by 1, so `day - row_number` stays constant; after a gap the date jumps further, giving a new key. This pattern is called **gaps and islands**.

**Other engines:** SQL Server: `DATEADD(day, -ROW_NUMBER() OVER (...), day)`. MySQL: `DATE_SUB(day, INTERVAL ROW_NUMBER() OVER (...) DAY)`.

### Overlapping time ranges

**Problem:** find double-booked practitioners, then make double bookings impossible.

```sql
-- find them: two ranges overlap when each starts before the other ends
SELECT a.id, b.id AS clashes_with, a.practitioner_id, a.starts_at, a.ends_at
FROM   appointments AS a
JOIN   appointments AS b
       ON  b.practitioner_id = a.practitioner_id
       AND b.id > a.id                       -- each pair once, never a row with itself
       AND a.starts_at < b.ends_at
       AND b.starts_at < a.ends_at
WHERE  a.status <> 'cancelled'
  AND  b.status <> 'cancelled';

-- prevent them: the database rejects any overlapping booking for the same practitioner
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE appointments
    ADD CONSTRAINT appointments_no_double_booking
    EXCLUDE USING gist (
        practitioner_id WITH =,
        tstzrange(starts_at, ends_at) WITH &&
    )
    WHERE (status <> 'cancelled');
```

**Why it works:** `a.start < b.end AND b.start < a.end` is the complete overlap test, including one booking inside another. The exclusion constraint applies it on every write, so of two simultaneous requests for the same slot the second fails with error `23P01`; `tstzrange` is half-open (`[start, end)`), so 10:00–10:30 and 10:30–11:00 don't clash.

**Other engines:** the overlap test works everywhere. SQL Server and MySQL have no exclusion constraints, so prevent double bookings with a locking transaction instead. See [Transactions and Concurrency](/Database-Standards/Transactions-and-Concurrency).

* * *

## Reshaping results

### Count by status in one row

**Problem:** per customer, the number of invoices in each status, as columns.

```sql
SELECT customer_id,
       count(*)                                                   AS invoices,
       count(*) FILTER (WHERE status = 'paid')                    AS paid,
       count(*) FILTER (WHERE status = 'sent' AND due_date < current_date) AS overdue,
       coalesce(sum(total) FILTER (WHERE status = 'paid'), 0)     AS paid_total
FROM   invoices
GROUP  BY customer_id;
```

**Why it works:** `FILTER (WHERE ...)` makes each aggregate count only matching rows, so one scan produces every column. It's also how you build a pivot with a fixed set of columns.

**Other engines:** SQL Server and MySQL use `SUM(CASE WHEN status = 'paid' THEN 1 ELSE 0 END)` (works in Postgres too, but `FILTER` reads better).

### Tree and hierarchy queries

**Problem:** the whole category tree, with each category's depth and full breadcrumb.

```sql
WITH RECURSIVE tree AS (
    -- start: the top-level categories
    SELECT id, parent_id, name,
           1           AS depth,
           ARRAY[id]   AS path,
           name::text  AS breadcrumb
    FROM   categories
    WHERE  parent_id IS NULL

    UNION ALL

    -- step: the children of every row found so far
    SELECT c.id, c.parent_id, c.name,
           t.depth + 1,
           t.path || c.id,
           t.breadcrumb || ' > ' || c.name
    FROM   categories AS c
    JOIN   tree       AS t ON c.parent_id = t.id
    WHERE  NOT c.id = ANY (t.path)        -- cycle guard: never revisit a category
)
SELECT id, depth, breadcrumb
FROM   tree
ORDER  BY path;
```

**Why it works:** the first `SELECT` seeds the result, then the second reruns on the previous round's new rows until it adds none. `path` records the route, so a loop in bad data (A's parent is B, B's parent is A) stops instead of running forever.

- **One subtree:** start with `WHERE id = 42` instead of `parent_id IS NULL`.
- **Ancestors of a category:** flip the join to `ON c.id = t.parent_id`.
- **Postgres 14+:** the `CYCLE` clause does the cycle guard for you: put `CYCLE id SET is_cycle USING cycle_path` after the closing bracket of the CTE.

**Other engines:** SQL Server omits `RECURSIVE` and stops after 100 levels unless you add `OPTION (MAXRECURSION n)`. MySQL 8.0+ supports `WITH RECURSIVE`, limited by `cte_max_recursion_depth` (1000 by default).

### Comma separated lists

**Problem:** each invoice with its product names in one column.

```sql
SELECT i.id,
       i.number,
       string_agg(DISTINCT p.name, ', ' ORDER BY p.name) AS products
FROM   invoices      AS i
JOIN   invoice_lines AS l ON l.invoice_id = i.id
JOIN   products      AS p ON p.id = l.product_id
GROUP  BY i.id;
```

**Why it works:** `string_agg` joins a group's values with a separator; without an inner `ORDER BY` the order follows the plan and can change between runs. Grouping by the primary key `i.id` alone is enough, because `i.number` is then one value per group.

**Other engines:** SQL Server 2017+: `STRING_AGG(p.name, ', ') WITHIN GROUP (ORDER BY p.name)`. MySQL: `GROUP_CONCAT(DISTINCT p.name ORDER BY p.name SEPARATOR ', ')`, which silently cuts the result at 1024 characters unless you raise `group_concat_max_len`.

### Build JSON for an API

**Problem:** return an invoice with its lines nested inside, as one JSON document.

```sql
SELECT jsonb_build_object(
           'id',     i.id,
           'number', i.number,
           'total',  i.total,
           'lines',  coalesce(
                         jsonb_agg(
                             jsonb_build_object(
                                 'product',   p.name,
                                 'quantity',  l.quantity,
                                 'unitPrice', l.unit_price
                             ) ORDER BY l.id
                         ) FILTER (WHERE l.id IS NOT NULL),
                         '[]'::jsonb)
       ) AS invoice
FROM   invoices           AS i
LEFT   JOIN invoice_lines AS l ON l.invoice_id = i.id
LEFT   JOIN products      AS p ON p.id = l.product_id
WHERE  i.id = 48213
GROUP  BY i.id;
```

**Why it works:** `jsonb_agg` collects the lines into an array and `jsonb_build_object` wraps them with the invoice fields. `FILTER` plus `coalesce` returns `[]` instead of `[null]` for an invoice with no lines.

> 💡 Use this for exports and heavy read endpoints, where it replaces several queries. For ordinary endpoints, shaping the response in the app (Prisma `include`, a Laravel API Resource) is easier to maintain.

> ⚠️ `numeric` becomes a JSON number, which JavaScript parses as floating point. For money that must stay exact on the client, send it as a string: `'total', i.total::text`.

**Other engines:** SQL Server: `FOR JSON PATH`. MySQL 8: `JSON_OBJECT` and `JSON_ARRAYAGG`.

* * *

## Writing data

### Insert or update

**Problem:** add stock for a product, creating the stock row if it doesn't exist yet.

```sql
-- ✅ one atomic statement; needs a unique constraint or index on product_id
INSERT INTO product_stock (product_id, quantity, updated_at)
VALUES (17, 5, now())
ON CONFLICT (product_id)
DO UPDATE SET quantity   = product_stock.quantity + EXCLUDED.quantity,
              updated_at = now();
```

```ts
// ❌ select, then insert or update: two requests at once both see "no row" and both insert
const row = await repo.findOneBy({ productId: 17 });
if (row) await repo.update({ productId: 17 }, { quantity: row.quantity + 5 });
else await repo.insert({ productId: 17, quantity: 5 });
```

**Why it works:** `ON CONFLICT` checks and writes in one step, protected by the unique index; `EXCLUDED` is the row you tried to insert. Use `DO NOTHING` to skip duplicates without changing anything.

| Tool | Upsert |
| --- | --- |
| Prisma | `upsert`. It only runs as a native `ON CONFLICT` when Prisma's "database upsert" conditions are met (one unique field in `where`, the same value in `create`, no nested writes). Otherwise it's a read then a write. Check the query log. |
| TypeORM | `repo.upsert(values, ['productId'])` runs `ON CONFLICT`. |
| Laravel | `Model::upsert($rows, uniqueBy: ['product_id'], update: ['quantity'])`. |
| EF Core | No upsert API. Use `ExecuteSqlAsync` with the SQL above, or Dapper. |

> 📝 Postgres 15 added `MERGE`. For "insert or update" under concurrent traffic, `ON CONFLICT` is still safer, because `MERGE` can fail with a unique violation when two sessions insert the same key at the same moment.

**Other engines:** MySQL: `INSERT ... ON DUPLICATE KEY UPDATE quantity = quantity + new.quantity`, with `AS new` after `VALUES (...)` (8.0.19+). SQL Server: `UPDATE ... WITH (UPDLOCK, HOLDLOCK)`, then `INSERT` if `@@ROWCOUNT = 0`, in a transaction; plain `MERGE` there has known concurrency problems without those hints.

### Get back what you changed

**Problem:** the id of a row you just inserted, or exactly which rows an update or delete touched.

```sql
-- the new id and defaults, without a second query
INSERT INTO invoices (customer_id, number, total)
VALUES (42, 'INV-2026-0193', 250.00)
RETURNING id, status, created_at;

-- which invoices just became overdue
UPDATE invoices
SET    status = 'overdue'
WHERE  status = 'sent'
  AND  due_date < current_date
RETURNING id, number;

-- move old rows to an archive table in one statement
WITH moved AS (
    DELETE FROM payments
    WHERE  paid_at < now() - interval '7 years'
    RETURNING *
)
INSERT INTO payments_archive
SELECT * FROM moved;

-- Postgres 18+: values from before and after an update
UPDATE invoices
SET    total = round(total * 1.1, 2)
WHERE  id = 48213
RETURNING old.total AS total_before, new.total AS total_after;
```

**Why it works:** `RETURNING` hands back the rows the statement wrote, including generated ids and defaults, in the same round trip.

**Other engines:** SQL Server: `OUTPUT inserted.id` / `OUTPUT deleted.*`. MySQL has no `RETURNING`; use `LAST_INSERT_ID()` after an insert.

### Insert or get the existing id

**Problem:** find a customer by email, creating them if needed, and get the id either way.

```sql
WITH inserted AS (
    INSERT INTO customers (email, name)
    VALUES ('ana@example.com', 'Ana Lopez')
    ON CONFLICT (email) DO NOTHING
    RETURNING id
)
SELECT id FROM inserted
UNION ALL
SELECT id FROM customers WHERE email = 'ana@example.com'
LIMIT 1;
```

**Why it works:** a successful insert returns the new id via `RETURNING`; if the email exists, `DO NOTHING` returns nothing and the second `SELECT` finds the existing row.

> ⚠️ In the rare case where another transaction inserts the same email at that exact moment, this can return no row. Retry once when it comes back empty. If you'd rather never retry, `ON CONFLICT (email) DO UPDATE SET email = EXCLUDED.email RETURNING id` always returns the id, at the cost of writing a new row version every time.

**Other engines:** MySQL: `INSERT ... ON DUPLICATE KEY UPDATE id = LAST_INSERT_ID(id)`, then `SELECT LAST_INSERT_ID()`. SQL Server: the locked `UPDATE`-then-`INSERT` pattern from [Insert or update](#insert-or-update) with `OUTPUT`.

* * *

## Small but useful

### Safe division

```sql
-- ✅ no-show rate; NULL instead of an error when there were no appointments
SELECT practitioner_id,
       round(100.0 * count(*) FILTER (WHERE status = 'no_show')
                   / NULLIF(count(*), 0), 1) AS no_show_pct
FROM   appointments
GROUP  BY practitioner_id;

-- ❌ integer divided by integer is an integer: 7 / 2 = 3, and 1 / 4 = 0
SELECT 7 / 2;          -- 3
SELECT 7 / 2.0;        -- 3.5000000000000000
SELECT 7::numeric / 2; -- 3.5000000000000000
```

**Why it works:** `NULLIF(x, 0)` returns NULL when `x` is 0, and anything divided by NULL is NULL; wrap it in `coalesce(..., 0)` to show 0 instead.

**Other engines:** integer division behaves the same in SQL Server. MySQL's `/` always returns a decimal, and `DIV` is its integer division.

### Case insensitive search

```sql
-- exact match ignoring case: index the expression you search on
CREATE INDEX customers_email_lower_idx ON customers (lower(email));
SELECT id FROM customers WHERE lower(email) = lower('Ana@Example.com');

-- "contains" ignoring case: a trigram index makes ILIKE '%...%' fast
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX customers_name_trgm_idx ON customers USING gin (name gin_trgm_ops);
SELECT id, name FROM customers WHERE name ILIKE '%lopez%';
```

- The `WHERE` must use exactly the indexed expression. An index on `lower(email)` does nothing for `WHERE email ILIKE ...`.
- A `citext` column makes plain `=` case-insensitive. See [PostgreSQL](/Database-Standards/PostgreSQL).
- Escape `%` and `_` in user input before putting it into a `LIKE` pattern, or a search for `50%` matches everything starting with `50`.

**Other engines:** SQL Server and MySQL usually default to case-insensitive collations (`..._CI_...`, `utf8mb4_0900_ai_ci`), so `=` already ignores case. Check the column's collation before adding `LOWER()`, because wrapping the column can stop an index being used.

### Search across several columns

**Problem:** one search box that matches a customer's name, email or phone.

```sql
-- one column that holds everything you search, kept up to date by Postgres
ALTER TABLE customers
    ADD COLUMN search_text text
    GENERATED ALWAYS AS (
        coalesce(name, '') || ' ' || coalesce(email, '') || ' ' || coalesce(phone, '')
    ) STORED;

CREATE INDEX customers_search_text_trgm_idx
    ON customers USING gin (search_text gin_trgm_ops);

-- ✅ one condition, one index
SELECT id, name, email
FROM   customers
WHERE  search_text ILIKE '%' || $1 || '%'
LIMIT  20;

-- ❌ three conditions joined by OR: usually a full table scan
SELECT id, name, email
FROM   customers
WHERE  name ILIKE '%' || $1 || '%'
   OR  email ILIKE '%' || $1 || '%'
   OR  phone ILIKE '%' || $1 || '%';
```

**Why it works:** the generated column is recomputed on every write, and the trigram index makes "contains" fast. Use `||` with `coalesce`, not `concat_ws`, because a generated column needs functions Postgres marks as immutable.

> 💡 For searching words in longer text (descriptions, notes, articles), with stemming and ranking, use full-text search instead. See [PostgreSQL](/Database-Standards/PostgreSQL).

**Other engines:** SQL Server: a computed column, or full-text indexing for longer text. MySQL: a `FULLTEXT` index across the columns with `MATCH ... AGAINST`.

* * *

## Do / Don't

Each row links to the recipe with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Add a tiebreaker column to every `ORDER BY` used for ranking or paging | Order by a column that can tie and hope the order is stable | [Top N per group](#top-n-per-group) |
| `NOT EXISTS` for "rows with no match" | `NOT IN (SELECT ...)` on a column that can be NULL | [Rows with no match](#rows-with-no-match) |
| `TABLESAMPLE` for samples from big tables | `ORDER BY random()` on millions of rows | [Random sample](#random-sample) |
| Add a unique index after cleaning up duplicates | Clean up duplicates and leave the door open | [Delete duplicates](#delete-duplicates-and-keep-one) |
| Spell out `ROWS BETWEEN` for moving windows | Rely on the default `RANGE` frame when values can tie | [Running totals](#running-totals-and-moving-averages) |
| Convert to the local time zone before truncating to a day | Group a `timestamptz` by day in UTC for a local business | [Local time](#group-by-day-in-local-time) |
| Filter on the raw timestamp column with a range | Wrap the column in a function in `WHERE` | [Local time](#group-by-day-in-local-time) |
| Prevent double bookings with an exclusion constraint | Rely on a check in app code before inserting | [Overlapping ranges](#overlapping-time-ranges) |
| Guard recursive queries against cycles | Assume the tree data has no loops | [Trees](#tree-and-hierarchy-queries) |
| `ORDER BY` inside `string_agg` and `jsonb_agg` | Expect aggregated lists to come back in a stable order | [Comma separated lists](#comma-separated-lists) |
| `INSERT ... ON CONFLICT` for upserts | Select, then insert or update | [Insert or update](#insert-or-update) |
| `RETURNING` to get ids and changed rows | Run a second query to find what you just wrote | [Get back what you changed](#get-back-what-you-changed) |
| `NULLIF(divisor, 0)` and a numeric operand | Divide integers and lose the decimals, or crash on zero | [Safe division](#safe-division) |
| Index the exact expression you search on | `ILIKE '%x%'` or `OR` across columns with no matching index | [Case insensitive search](#case-insensitive-search) |

* * *

## Further reading

1.  **PostgreSQL Documentation — Window Functions tutorial**
    [https://www.postgresql.org/docs/current/tutorial-window.html](https://www.postgresql.org/docs/current/tutorial-window.html)
2.  **PostgreSQL Documentation — WITH Queries (Common Table Expressions)**
    [https://www.postgresql.org/docs/current/queries-with.html](https://www.postgresql.org/docs/current/queries-with.html)
3.  **PostgreSQL Documentation — Aggregate Functions** (`string_agg`, `jsonb_agg`, `percentile_cont`)
    [https://www.postgresql.org/docs/current/functions-aggregate.html](https://www.postgresql.org/docs/current/functions-aggregate.html)
4.  **PostgreSQL Documentation — INSERT** (`ON CONFLICT` and `RETURNING`)
    [https://www.postgresql.org/docs/current/sql-insert.html](https://www.postgresql.org/docs/current/sql-insert.html)
5.  **PostgreSQL Documentation — Range Types** (including exclusion constraints)
    [https://www.postgresql.org/docs/current/rangetypes.html](https://www.postgresql.org/docs/current/rangetypes.html)
6.  **Modern SQL** — which SQL features each database supports, with examples
    [https://modern-sql.com/](https://modern-sql.com/)

* * *

## Related pages

- [Writing Queries](/Database-Standards/Writing-Queries) — the rules behind these recipes: filters that use an index, joins, CTEs, pagination.
- [Indexing](/Database-Standards/Indexing) — the indexes that make these recipes fast on real data.
- [PostgreSQL](/Database-Standards/PostgreSQL) — JSONB, full-text search, ranges and the extensions used here.
- [General Principles](/Database-Standards/General-Principles#touching-production-by-hand) — before you run a cleanup recipe on production.
