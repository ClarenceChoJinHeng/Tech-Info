[🏠 Home](../README.md) › [Database Standards](README.md) › **General Principles**

# General Principles

> Applies to every database we use. Engine-specific rules live on the [PostgreSQL](PostgreSQL.md), [SQL Server](SQL-Server.md), [MySQL](MySQL.md), [MongoDB](MongoDB.md) and [Redis](Redis.md) pages.
> **Which ORM each stack uses, and how it's set up:** [Tech Stack → Backend Stack](../Tech-Stack/Backend-Stack.md#database-and-orm). This page is how to work with the database day to day.

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [Default engine](#postgres-unless-told-otherwise) | **PostgreSQL, unless the client or spec says otherwise**<br/><sub>Any other engine for the main database is a documented exception with a reason.</sub> |
| [Constraints](#constraints-belong-in-the-database) | **NOT NULL, foreign keys, UNIQUE and CHECK live in the schema**<br/><sub>App validation stops honest mistakes. Constraints also stop race conditions, scripts and the next app.</sub> |
| [Think in sets](#think-in-sets-not-loops) | **One query for many rows, not one query per row**<br/><sub>1,000 round trips are always slower than one statement that touches 1,000 rows.</sub> |
| [ORM or raw SQL](#orm-or-raw-sql) | **ORM by default, raw SQL when the ORM gets in the way**<br/><sub>Prisma or TypeORM, Eloquent, EF Core. Dapper on .NET when you want to write the SQL yourself. One tool owns migrations.</sub> |
| [Parameters](#always-parameterize) | **Values go in as parameters, never pasted into the SQL string**<br/><sub>String-building a query is how SQL injection happens, with or without an ORM.</sub> |
| [Naming](#one-naming-convention-per-database) | **One naming convention per database, never mixed**<br/><sub>snake_case by default, PascalCase where the ORM expects it. The details are in Schema Design.</sub> |
| [Time](#store-time-in-utc) | **Timestamps with a time zone, stored in UTC**<br/><sub>Convert to local time at the edge, when you show it to a person.</sub> |
| [Money](#money-is-never-a-float) | **Exact decimals or integer cents, never float**<br/><sub>A float can't store 0.10 exactly, and the rounding error ends up on an invoice.</sub> |
| [Schema changes](#every-schema-change-is-a-migration) | **Every change is a committed migration**<br/><sub>Nobody runs ALTER TABLE by hand on a shared database.</sub> |
| [Performance](#measure-before-you-optimize) | **Look at the query plan before adding an index or rewriting a query**<br/><sub>A guess about what's slow is wrong more often than it's right.</sub> |
| [Transactions](#keep-transactions-short) | **Short transactions, no network calls inside them**<br/><sub>A transaction waiting on an HTTP call holds its locks the whole time.</sub> |
| [Production](#touching-production-by-hand) | **Hand-run writes go inside a transaction, row count checked before COMMIT**<br/><sub>ROLLBACK is free. Restoring from a backup is not.</sub> |
| [Real data](#production-data-stays-in-production) | **Production data never lands on a laptop unmasked**<br/><sub>Use seed data, or an anonymized copy when you need realistic volume.</sub> |
| [SQL style](#sql-style-and-linting) | **Uppercase keywords, explicit JOINs, one clause per line**<br/><sub>SQLFluff checks hand-written SQL, so review is about logic instead of layout.</sub> |

* * *

## Concepts you'll meet on these pages

Skip this if you've worked with relational databases before. If you haven't, the rest of the section assumes these.

### Constraint

A rule the database itself enforces on every write, whoever makes it: `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK`. If a write breaks the rule, the database rejects it with an error instead of storing bad data.

### Transaction

A group of statements that either **all** happen or **none** do. If anything fails part-way, `ROLLBACK` undoes everything since `BEGIN`, as if none of it ran.

### Index

A separate, sorted structure that lets the database find rows without reading the whole table, like the index at the back of a book. Faster reads, but every write has to update it too. See [Indexing](Indexing.md).

### Query plan

The database's step-by-step recipe for running your query: which index to use, which table to read first, how to join them. `EXPLAIN` shows it to you. It's the first thing to look at when a query is slow. See [Performance and Diagnostics](Performance-and-Diagnostics.md).

### Migration

A file, committed with your code, that changes the schema: add a table, add a column, create an index. Every environment runs the same migrations in the same order, so they all end up with the same schema.

### N+1 queries

Loading a list with one query, then running one more query **per item** to fetch something related. 50 invoices become 51 queries. It's the most common performance bug an ORM makes easy. See [Writing Queries](Writing-Queries.md).

* * *

## Choosing and designing

### Postgres unless told otherwise

Every new project uses **PostgreSQL** as its main database. Use something else only when:

- the client or the spec requires a specific engine (common for FI clients on SQL Server), or
- the project extends an existing system that already runs on another engine.

Redis and MongoDB are not alternatives to Postgres. They sit **next to** it for specific jobs. The [Database Standards](README.md#which-database) landing page has the decision chart and what each engine is for.

> ⚠️ If the main database isn't Postgres, write down why in the project README. See How to Use This Wiki → *Document every exception* *(page coming later)*.

### Constraints belong in the database

Validate in the app **and** constrain in the schema. App validation gives the user a friendly message. The constraint is what actually guarantees the data is right, because the app isn't the only thing that writes to the database: so do migrations, seed scripts, admin tools, a second service, and two requests running at the same time.

```sql
-- ✅ the schema says what a valid invoice is
CREATE TABLE invoices (
    id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id  bigint        NOT NULL REFERENCES customers (id),
    number       text          NOT NULL UNIQUE,
    status       text          NOT NULL DEFAULT 'draft'
                               CHECK (status IN ('draft', 'sent', 'paid', 'void')),
    total        numeric(12,2) NOT NULL CHECK (total >= 0),
    created_at   timestamptz   NOT NULL DEFAULT now()
);

-- ❌ every column nullable, nothing checked: the app is the only thing standing between you and bad data
CREATE TABLE invoices (
    id           serial PRIMARY KEY,
    customer_id  int,
    number       varchar(255),
    status       varchar(255),
    total        float,
    created_at   timestamp
);
```

The classic case app validation can't cover:

```ts
// ❌ check-then-insert: two requests arrive at the same moment, both see "not taken", both insert
const exists = await prisma.invoice.findUnique({ where: { number } });
if (!exists) await prisma.invoice.create({ data: { number, ... } });

// ✅ let the UNIQUE constraint decide, and turn its error into a friendly message
try {
  await prisma.invoice.create({ data: { number, ... } });
} catch (e) {
  if (e.code === 'P2002') throw new ConflictException('Invoice number already used');
  throw e;
}
```

> 💡 Name the columns and types in the schema the way you'd want to read them in a report a year from now. How to choose each type, key and constraint is in [Schema Design](Schema-Design.md).

### One naming convention per database

We have projects in **snake_case** and projects in **PascalCase**, and both are fine. **Mixing them in one database is not.**

| Situation | Convention |
| --- | --- |
| New project on Postgres or MySQL | **snake_case**: `invoice_lines`, `customer_id`, `created_at` |
| .NET / EF Core, especially on SQL Server | **PascalCase** is acceptable: `InvoiceLines`, `CustomerId`, matching EF Core's defaults |
| Existing project | **Whatever it already uses.** Don't introduce the other style for new tables. |

> ⚠️ **Postgres folds unquoted names to lowercase.** A table created as `"Invoice"` (Prisma's default for a model called `Invoice`) has to be written with quotes in every raw query, forever:
> ```sql
> SELECT customerId FROM Invoice;        -- ERROR: relation "invoice" does not exist
> SELECT "customerId" FROM "Invoice";    -- works, but every query now needs the quotes
> ```
> That's why snake_case is the default on Postgres. On a NestJS project, keep PascalCase models in Prisma and map them with `@@map("invoices")` / `@map("customer_id")`.

The full naming rules (plural tables, key and index names, booleans) are in [Schema Design](Schema-Design.md).

### Store time in UTC

Store a moment in time as a **timestamp with a time zone**, in UTC. Convert it to the user's local time only when you show it to them.

```sql
-- ✅ an absolute instant: Postgres stores it as UTC and converts on the way out
created_at  timestamptz NOT NULL DEFAULT now()

-- ❌ no time zone: "2026-03-29 02:30" means a different instant on every server and every laptop
created_at  timestamp
```

| Engine | Use for a moment in time |
| --- | --- |
| PostgreSQL | `timestamptz` |
| SQL Server | `datetimeoffset`, or `datetime2` that is always written in UTC |
| MySQL | `DATETIME` that is always written in UTC (`TIMESTAMP` stops working in 2038) |

> 📝 Not everything is a moment. A birthday or an invoice due date is a calendar **date**, so store it as `date`. Converting a date-only value through time zones is how a due date ends up one day early.

### Money is never a float

Store money as an **exact decimal** (`numeric(12,2)`) or as **integer minor units** (`bigint` cents). Pick one per project and stick to it.

```sql
SELECT 0.1::float8  + 0.2::float8;    -- 0.30000000000000004
SELECT 0.1::numeric + 0.2::numeric;   -- 0.3
```

- Never `real`, `float` or `double precision` for money.
- Avoid Postgres's `money` type too. Its output depends on the server's locale settings.
- If a project handles more than one currency, store the ISO 4217 code next to the amount (`amount numeric(12,2), currency text NOT NULL CHECK (currency ~ '^[A-Z]{3}$')`). Not `char(3)`: Postgres pads `char(n)` with spaces, which causes surprises in comparisons.

* * *

## Writing queries

### Think in sets, not loops

A database is built to work on many rows in one statement. Each query from your app pays for a network round trip, so a loop of queries is slow even when every single query is fast.

```ts
// ❌ one UPDATE per invoice: 500 overdue invoices = 500 round trips
for (const inv of overdue) {
  await prisma.invoice.update({ where: { id: inv.id }, data: { status: 'overdue' } });
}

// ✅ one statement, and the database finds the rows itself
await prisma.invoice.updateMany({
  where: { status: 'sent', dueDate: { lt: new Date() } },
  data: { status: 'overdue' },
});
```

```sql
-- the same thing in SQL
UPDATE invoices
SET    status = 'overdue'
WHERE  status = 'sent'
  AND  due_date < current_date;
```

The same applies to reads. Load related data with the list instead of once per item: Prisma `include`, TypeORM `relations` or `leftJoinAndSelect`, Laravel `with()`, EF Core `Include()`. With Dapper you write the `JOIN` yourself, so there's no lazy loading to trip over, but a query inside a `foreach` is still N+1. See [Writing Queries](Writing-Queries.md) for N+1 in detail.

> 💡 If the number of rows is huge (hundreds of thousands or more), one statement can lock too much for too long. That's when you batch. See [Large Data and Bulk Operations](Large-Data-and-Bulk-Operations.md).

### ORM or raw SQL

#### The data-access tools we use

| Stack | Tool | What it is | Where we use it |
| --- | --- | --- | --- |
| NestJS | **Prisma** | ORM. One `schema.prisma` file drives migrations and fully typed queries. | The default for new NestJS projects. See [Tech Stack → Backend Stack](../Tech-Stack/Backend-Stack.md#database-and-orm). |
| NestJS | **TypeORM** | ORM. Entity classes with decorators, plus a query builder. | Existing NestJS projects built on it. Keep using it there rather than mixing in Prisma. |
| Laravel | **Eloquent** + Query Builder | ORM, with `DB::` for raw SQL. | Every Laravel project. |
| .NET | **EF Core** | ORM. LINQ queries, change tracking, migrations. | Most writes and all schema migrations on .NET projects. |
| .NET | **Dapper** | Micro-ORM. You write the SQL, and Dapper maps the rows to objects. Nothing generated, nothing tracked. | Reads, reports and tuned queries where EF Core's generated SQL gets in the way. Often in the same project as EF Core, on the same database. |

> ⚠️ **One tool owns the schema.** When a project uses EF Core and Dapper together, EF Core migrations still own every schema change. Dapper only runs queries. On a Dapper-only project, use numbered `.sql` migration scripts (see [Every schema change is a migration](#every-schema-change-is-a-migration)).

> ⚠️ **TypeORM: never `synchronize: true` outside your own machine.** It changes the database schema to match your entities every time the app starts, and it will drop a column it no longer sees, data and all. Shared environments use migrations only.

#### Which one for this query?

Use the ORM by default. Switch to raw SQL (Dapper on .NET, `$queryRaw` in Prisma, `query()` or the query builder in TypeORM, `DB::select` in Laravel) when the ORM version is harder to read or slower than the SQL would be.

| Use the ORM for | Use raw SQL for |
| --- | --- |
| Create, read, update, delete on one entity | Reports: `GROUP BY`, window functions, several joins |
| Simple filters, sorting, paging | Bulk updates and upserts across many rows |
| Loading related records | A query you've tuned with `EXPLAIN` and need to keep exactly as written |
| Anything that has to stay in step with the schema's types | Engine features the ORM doesn't expose: JSONB operators, `LATERAL`, recursive CTEs |

Wherever raw SQL lives, it follows the same rules as any other code:

- It sits in the data layer (repository or service), **never in a controller**.
- It's parameterized (see the next section).
- It has a test that runs it against a **real database**, not a mock. A mock always agrees with the test.

### Always parameterize

Values from users, URLs, files or other systems go into the query as **parameters**. The database receives the SQL and the values separately, so a value can never be run as SQL.

```ts
// Prisma
// ✅ the tagged template turns ${email} into a parameter
await prisma.$queryRaw`SELECT id FROM users WHERE email = ${email}`;
// ❌ string-built: email = "x' OR '1'='1" returns every user
await prisma.$queryRawUnsafe(`SELECT id FROM users WHERE email = '${email}'`);

// TypeORM
// ✅ named parameter in the query builder
await repo.createQueryBuilder('u').where('u.email = :email', { email }).getOne();
// ✅ positional parameter in a raw query
await dataSource.query('SELECT id FROM users WHERE email = $1', [email]);
// ❌ the value is pasted into the WHERE string
await repo.createQueryBuilder('u').where(`u.email = '${email}'`).getOne();

// node-postgres
// ✅
await pool.query('SELECT id FROM users WHERE email = $1', [email]);
```

```php
// Laravel
// ✅
DB::select('select id from users where email = ?', [$email]);
// ❌
DB::select("select id from users where email = '$email'");
```

```csharp
// EF Core
// ✅ FromSql turns the interpolated value into a parameter
db.Users.FromSql($"SELECT * FROM users WHERE email = {email}");
// ❌ FromSqlRaw with concatenation pastes the value straight in
db.Users.FromSqlRaw("SELECT * FROM users WHERE email = '" + email + "'");

// Dapper
// ✅ @Email is a parameter, filled from the anonymous object
await conn.QuerySingleOrDefaultAsync<User>(
    "SELECT id, email, name FROM users WHERE email = @Email",
    new { Email = email });
// ❌ string interpolation: Dapper sends whatever string you built
await conn.QueryAsync<User>($"SELECT id, email, name FROM users WHERE email = '{email}'");
```

> ⚠️ **Dapper makes this mistake easy.** In EF Core, `FromSql($"...")` turns `{email}` into a parameter. In Dapper, `$"..."` is just a C# string, and `{email}` is pasted straight into the SQL. Always pass values through the parameter object.

**Table and column names can't be parameters.** When a user picks the sort column, map their input to an allowlist instead of inserting it into the SQL:

```ts
// ✅ the user chooses a key; only names you wrote can ever reach the SQL
const SORTABLE = { date: 'created_at', total: 'total', number: 'number' } as const;
const column = SORTABLE[query.sort as keyof typeof SORTABLE] ?? 'created_at';
```

More on injection: Security → Secure Coding *(page coming later)*.

* * *

## Changes and production

### Every schema change is a migration

Make every change to tables, columns, indexes or constraints in a migration file, and commit it with the code that needs it. Never make the change by hand in a GUI tool or console on a shared database: the other environments never get it, and nobody can tell what changed or when.

| Stack | Create a migration | Apply it in other environments |
| --- | --- | --- |
| NestJS + Prisma | `pnpm prisma migrate dev --name add-invoice-due-date` | `pnpm prisma migrate deploy` |
| NestJS + TypeORM | `pnpm typeorm migration:generate src/migrations/AddInvoiceDueDate -d src/data-source.ts` | `pnpm typeorm migration:run -d src/data-source.ts` |
| Laravel | `php artisan make:migration add_due_date_to_invoices` | `php artisan migrate --force` |
| .NET + EF Core (with or without Dapper) | `dotnet ef migrations add AddInvoiceDueDate` | `dotnet ef migrations script --idempotent`, reviewed, then applied by the pipeline |
| .NET + Dapper only | A new numbered script, e.g. `Scripts/0042_add_invoice_due_date.sql` | [DbUp](https://dbup.readthedocs.io/) runs every script that hasn't been applied yet, in order |

> 📝 On TypeORM projects, `typeorm` in the commands above is a `package.json` script pointing at `typeorm-ts-node-commonjs`, so the CLI can read the TypeScript data source. Read every generated migration before committing it. TypeORM sometimes drops and recreates a column where a rename was intended.

Some changes that look harmless lock a big table for minutes: adding a `NOT NULL` column, changing a type, creating an index the normal way. [Migrations and Schema Changes](Migrations-and-Schema-Changes.md) shows how to make each of those safely on a live table.

### Measure before you optimize

When a query is slow, look at its plan before you add an index or rewrite it. The plan tells you where the time goes. A guess usually doesn't.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, number, total
FROM   invoices
WHERE  customer_id = 42
  AND  status = 'paid';
```

> ⚠️ `EXPLAIN ANALYZE` **actually runs** the statement. To look at an `UPDATE` or `DELETE`'s plan without changing anything, wrap it:
> ```sql
> BEGIN;
> EXPLAIN ANALYZE DELETE FROM sessions WHERE expires_at < now();
> ROLLBACK;
> ```

How to read the output and find the slowest queries on a running system: [Performance and Diagnostics](Performance-and-Diagnostics.md).

### Keep transactions short

Use a transaction whenever several writes must succeed or fail together. Keep it to the database work only. While a transaction is open, the rows it has written stay locked, and every other request that needs them waits.

```ts
// ❌ the order row stays locked for as long as the payment provider takes to answer
await prisma.$transaction(async (tx) => {
  await tx.order.update({ where: { id }, data: { status: 'paying' } });
  await paymentProvider.charge(order);          // 2–30 seconds over the network
  await tx.order.update({ where: { id }, data: { status: 'paid' } });
});

// ✅ short transactions around the writes, the network call between them
await prisma.order.update({ where: { id }, data: { status: 'paying' } });
const charge = await paymentProvider.charge(order, { idempotencyKey: order.id });
await prisma.$transaction([
  prisma.order.update({ where: { id }, data: { status: 'paid' } }),
  prisma.payment.create({ data: { orderId: id, chargeId: charge.id } }),
]);
```

Isolation levels, deadlocks, and how to stop two requests overwriting each other: [Transactions and Concurrency](Transactions-and-Concurrency.md).

### Touching production by hand

Sometimes a fix has to be run by hand against production. When it does:

1. **Write it as a `SELECT` first.** Check that it returns exactly the rows you mean to change.
2. **Run the write inside a transaction**, and check the row count before you commit.
3. **Have someone else look at it** before you run it. Paste it in the ticket or PR so there's a record.
4. **Know when the last backup ran** before you start.

```sql
BEGIN;

UPDATE invoices
SET    status = 'void'
WHERE  id = 48213;
-- psql prints: UPDATE 1     ← you expected 1 row. If it says UPDATE 48213, ROLLBACK.

SELECT id, status FROM invoices WHERE id = 48213;   -- check it looks right

COMMIT;   -- or ROLLBACK;
```

| Engine | Start a transaction | Watch out for |
| --- | --- | --- |
| PostgreSQL | `BEGIN;` | psql autocommits every statement you run outside a transaction |
| SQL Server | `BEGIN TRAN;` | SSMS shows "(n rows affected)": read it before `COMMIT` |
| MySQL | `START TRANSACTION;` | Autocommit is on by default, and DDL (`ALTER`, `DROP`) commits immediately even inside a transaction |

> 💡 **In DataGrip or DBeaver:** switch production connections to **manual commit**, and give them a red colour. Most "I ran it on the wrong database" stories start with a production tab that looked just like the local one.

If the fix is anything more than a few rows, it should be a reviewed migration or script in the repo, not something typed into a console.

### Production data stays in production

Never copy a production dump onto a laptop or into a shared dev or staging database as-is. It contains real people's personal data, and laptops get lost.

- **Local and dev:** seed data and factories (Prisma seed, Laravel factories and seeders, EF Core `HasData` or a seed script).
- **When you need realistic volume or a real bug's data:** an anonymized copy, with names, emails, phone numbers, addresses and payment details replaced. Agree it with the tech lead first.

See Security *(page coming later)* for how staging and personal data are protected.

* * *

## Tooling

### SQL style and linting

Raw SQL in the repo (migrations you write by hand, reports, scripts) follows one style:

- **Uppercase keywords**, lowercase identifiers: `SELECT`, `FROM`, `WHERE`.
- **One clause per line**, so a diff shows exactly which clause changed.
- **Explicit `JOIN ... ON`**, never a comma between tables in `FROM`.
- **Meaningful table aliases** (`i` for invoices is fine, `a`, `b`, `c` is not), and every column qualified once there's a join.
- **Name every column you select.** `SELECT *` only in a quick console query, never in code.

```sql
-- ❌
select * from invoices i, customers c where i.customer_id=c.id and i.status='paid'

-- ✅
SELECT i.number,
       i.total,
       c.name AS customer_name
FROM   invoices AS i
JOIN   customers AS c ON c.id = i.customer_id
WHERE  i.status = 'paid';
```

**[SQLFluff](https://docs.sqlfluff.com/)** lints and formats SQL files for Postgres, SQL Server (`tsql`) and MySQL. Add a `.sqlfluff` at the repo root:

```ini
[sqlfluff]
dialect = postgres
max_line_length = 120

[sqlfluff:rules:capitalisation.keywords]
capitalisation_policy = upper
```

```bash
sqlfluff lint sql/            # report problems
sqlfluff fix sql/             # fix what it can automatically
```

- Run it in the same pre-commit hook as the rest of the project's linters. See [Coding Standards → Linting and formatting](../Coding-Standards/General-Principles.md#linting-and-formatting).
- Add migrations generated by the ORM (Prisma's `migration.sql`, for example) to `.sqlfluffignore`. They're machine-written and never edited by hand.

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Start every new project on PostgreSQL | Pick another engine without a written reason | [Default engine](#postgres-unless-told-otherwise) |
| Put NOT NULL, FK, UNIQUE and CHECK constraints in the schema | Rely on app validation alone | [Constraints](#constraints-belong-in-the-database) |
| Let a UNIQUE constraint catch duplicates | Check-then-insert | [Constraints](#constraints-belong-in-the-database) |
| Update or load many rows in one statement | Loop over rows, one query each | [Think in sets](#think-in-sets-not-loops) |
| Keep raw SQL in the data layer, tested against a real database | Put raw SQL in controllers | [ORM or raw SQL](#orm-or-raw-sql) |
| Let one tool own migrations (EF Core alongside Dapper, migrations in TypeORM) | Use TypeORM `synchronize: true` on a shared database | [ORM or raw SQL](#orm-or-raw-sql) |
| Pass every value as a parameter, allowlist column names | Build SQL with string concatenation or interpolation, including C# `$"..."` in Dapper | [Parameters](#always-parameterize) |
| Use one naming convention per database | Mix `snake_case` and `PascalCase` tables in one schema | [Naming](#one-naming-convention-per-database) |
| `timestamptz` in UTC, `date` for calendar dates | `timestamp` without a time zone | [Time](#store-time-in-utc) |
| `numeric(12,2)` or integer cents for money | `float`, `double` or Postgres `money` | [Money](#money-is-never-a-float) |
| Commit a migration for every schema change | Change a shared database by hand | [Migrations](#every-schema-change-is-a-migration) |
| Read the query plan before optimizing | Add indexes on a hunch | [Performance](#measure-before-you-optimize) |
| Keep transactions to the database writes | Call an API or wait on a user inside a transaction | [Transactions](#keep-transactions-short) |
| SELECT first, then write inside a transaction, check the row count | Run a bare UPDATE or DELETE on production | [Production](#touching-production-by-hand) |
| Use seed data or an anonymized copy locally | Copy a production dump to your laptop | [Real data](#production-data-stays-in-production) |

* * *

## Further reading

1.  **PostgreSQL Wiki — Don't Do This** (timestamps, money, naming and more)
    [https://wiki.postgresql.org/wiki/Don't_Do_This](https://wiki.postgresql.org/wiki/Don't_Do_This)
2.  **PostgreSQL Documentation — Constraints**
    [https://www.postgresql.org/docs/current/ddl-constraints.html](https://www.postgresql.org/docs/current/ddl-constraints.html)
3.  **Use The Index, Luke** — a free guide to SQL performance for developers
    [https://use-the-index-luke.com/](https://use-the-index-luke.com/)
4.  **OWASP — SQL Injection Prevention Cheat Sheet**
    [https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
5.  **SQLFluff documentation**
    [https://docs.sqlfluff.com/](https://docs.sqlfluff.com/)

* * *

## Related pages

- [Schema Design](Schema-Design.md) — the detail behind naming, keys, types and constraints.
- [Writing Queries](Writing-Queries.md) — the next page to read.
- [Coding Standards → Backend — NestJS](../Coding-Standards/Backend-NestJS.md#database-changes), [Laravel](../Coding-Standards/Backend-Laravel-PHP.md), [.NET](../Coding-Standards/Backend-DotNet-CSharp.md) — where the database code sits in each stack.

---

[⬅ Back to Database Standards](README.md)
