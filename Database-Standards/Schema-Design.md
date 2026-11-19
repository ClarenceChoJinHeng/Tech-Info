[🏠 Home](../README.md) › [Database Standards](README.md) › **Schema Design**

# Schema Design

> Read **[General Principles](General-Principles.md)** first. This page is the detail behind its naming, time, money and constraint rules.
> Examples are PostgreSQL. Where SQL Server or MySQL differ, the section says so.

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [Naming](#naming) | **snake_case, plural tables, singular columns, one convention per database**<br/><sub>PascalCase is fine on EF Core and existing projects. Mixing the two never is.</sub> |
| [ORM mapping](#mapping-names-in-the-orm) | **Let the ORM map code names to database names**<br/><sub>Prisma `@map`, a TypeORM naming strategy, or EF Core's snake_case convention. Code stays idiomatic, SQL stays quote-free.</sub> |
| [Primary keys](#primary-keys) | **`bigint` identity inside, UUIDv7 for anything that leaves the system**<br/><sub>Sequential IDs in a URL let anyone count your invoices and guess the next one.</sub> |
| [Data types](#data-types) | **Pick the type that matches the meaning, not the one that fits**<br/><sub>`text` for strings, `numeric` for money, `timestamptz` for moments, `jsonb` only for genuinely loose data.</sub> |
| [Fixed value sets](#status-columns-and-fixed-value-sets) | **CHECK for a short fixed list, a lookup table when values carry data**<br/><sub>Postgres enums are hard to change later.</sub> |
| [NULL](#null) | **NOT NULL unless "no value" is a real state**<br/><sub>Never use `''` or `0` to mean "unknown".</sub> |
| [Constraints](#constraints) | **Every rule you can state about the data becomes a constraint**<br/><sub>Including "two appointments can't overlap", which Postgres can enforce for you.</sub> |
| [Foreign keys](#foreign-keys-and-on-delete) | **Index every foreign key, choose ON DELETE on purpose**<br/><sub>CASCADE only for children that can't exist without their parent. Watch EF Core's cascade default.</sub> |
| [Normalization](#normalize-first-then-denormalize-on-purpose) | **Each fact stored once, denormalize only deliberately**<br/><sub>A snapshot of a price at sale time is not duplication. A copied customer name is.</sub> |
| [Soft delete](#soft-delete) | **Only where you need undo or history, with a partial unique index**<br/><sub>Every query has to remember to filter it, so let the ORM do that.</sub> |
| [Audit columns](#audit-columns) | **`created_at` and `updated_at` on every table**<br/><sub>`updated_at` needs a trigger or the ORM, it doesn't maintain itself.</sub> |
| [Many-to-many](#many-to-many-join-tables) | **A join table with a composite primary key**<br/><sub>Never a comma-separated list of IDs in a column.</sub> |
| [Generated columns](#generated-columns) | **Let the database compute derived values**<br/><sub>The value can't drift from the columns it's computed from.</sub> |

* * *

## Naming

### The rules

| Thing | Rule | ✅ | ❌ |
| --- | --- | --- | --- |
| Tables | Plural, snake_case | `invoices`, `invoice_lines` | `Invoice`, `tbl_invoice`, `invoiceLine` |
| Columns | Singular, snake_case, no table prefix | `name`, `total`, `status` | `invoice_name`, `InvoiceTotal`, `sts` |
| Primary key | `id` | `id` | `invoice_id` on the invoices table, `pk` |
| Foreign keys | Singular table name + `_id` | `customer_id`, `invoice_id` | `customer`, `cust_fk`, `customerID` |
| Two keys to the same table | Say the role | `billed_to_customer_id`, `created_by_user_id` | `customer_id2` |
| Booleans | Reads as a yes/no question | `is_active`, `has_insurance` | `active_flag`, `status_bool` |
| Moments in time | Past tense + `_at` | `created_at`, `paid_at`, `cancelled_at` | `create_time`, `paid` |
| Calendar dates | `_date` (or `_on`, one per project) | `due_date`, `birth_date` | `due`, `dob` |
| Join tables | Both tables, in the order you'd say it | `user_roles`, `product_categories` | `mapping`, `link_table` |

> 💡 **A timestamp often beats a boolean.** `paid_at timestamptz NULL` tells you *whether* and *when*. `is_paid boolean` loses the "when" for good.

> 📝 **Plural table names also dodge reserved words.** `user`, `order` and `group` need quoting. `users`, `orders` and `groups` don't.

### Constraint and index names

In hand-written SQL, name constraints and indexes yourself, so an error names the rule that failed: `violates check constraint "ck_invoices_total_not_negative"` tells you far more than `"invoices_check1"`.

| Kind | Pattern | Example |
| --- | --- | --- |
| Primary key | `pk_<table>` | `pk_invoices` |
| Foreign key | `fk_<table>_<column>` | `fk_invoices_customer_id` |
| Unique | `uq_<table>_<columns>` | `uq_users_email` |
| Check | `ck_<table>_<rule>` | `ck_invoices_total_not_negative` |
| Index | `ix_<table>_<columns>` | `ix_invoices_customer_id_created_at` |

When the ORM generates the names (Prisma's `invoices_customer_id_fkey`, EF Core's `FK_Invoices_Customers_CustomerId`, Laravel's `invoices_customer_id_foreign`), **keep the ORM's pattern**. Don't rename them by hand.

> ⚠️ Postgres silently **truncates** names longer than 63 bytes. Two long index names that share their first 63 characters collide. SQL Server allows 128 characters, MySQL 64.

### Snake case or Pascal case

| Situation | Convention |
| --- | --- |
| New project on Postgres or MySQL | **snake_case** |
| .NET with EF Core, especially on SQL Server | **PascalCase** is acceptable (`Invoices`, `CustomerId`), matching EF Core's defaults |
| An existing project | **Whatever it already uses**, for every new table too |

Why snake_case on Postgres: unquoted names fold to lowercase, so a PascalCase table needs `"Invoice"`, quoted, in every raw query. See [General Principles → One naming convention per database](General-Principles.md#one-naming-convention-per-database).

### Mapping names in the ORM

```prisma
// Prisma — PascalCase model, camelCase fields, snake_case in the database
model Invoice {
  id         BigInt   @id @default(autoincrement())
  customerId BigInt   @map("customer_id")
  dueDate    DateTime @map("due_date") @db.Date
  createdAt  DateTime @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt  DateTime @updatedAt @map("updated_at") @db.Timestamptz(6)

  customer Customer @relation(fields: [customerId], references: [id], onDelete: Restrict)

  @@map("invoices")
}
```

> ⚠️ **Prisma's `DateTime` is `timestamp(3)` (without time zone) on Postgres unless you say otherwise.** Add `@db.Timestamptz(6)` to every moment-in-time field, and `@db.Date` to calendar dates.

```ts
// TypeORM — one setting maps every entity and column (package: typeorm-naming-strategies)
import { SnakeNamingStrategy } from 'typeorm-naming-strategies';

export const dataSource = new DataSource({
  type: 'postgres',
  namingStrategy: new SnakeNamingStrategy(),
  // ...
});
```

```csharp
// EF Core on Postgres — package: EFCore.NamingConventions
options.UseNpgsql(connectionString)
       .UseSnakeCaseNamingConvention();
```

- **Laravel:** already plural snake_case tables and columns. Nothing to configure.
- **Dapper:** maps by name, so alias in SQL (`customer_id AS CustomerId`) or set `Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true` once at startup.

* * *

## Primary keys

### The default

```sql
-- ✅ every table, unless there's a reason below
id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

| Choice | Why |
| --- | --- |
| `bigint`, not `integer` | `integer` runs out at about 2.1 billion. Changing a key's type later rewrites the table and every foreign key pointing at it. |
| `IDENTITY`, not `serial` | SQL standard (Postgres 10+). `serial` creates a loose sequence with separate permissions and ownership that drifts out of step. |
| `GENERATED ALWAYS` | Rejects a hand-written `id`, so nobody collides with the sequence. Imports that must keep IDs use `INSERT ... OVERRIDING SYSTEM VALUE`. |

> 📝 **Don't fight the ORM's key setup.** Prisma's `@default(autoincrement())` creates `bigserial`/`serial`, and Npgsql for EF Core creates `GENERATED BY DEFAULT AS IDENTITY`. Both are fine on projects managed through that ORM. What matters is `bigint`.

### IDs that leave the system

Never expose a sequential ID (URLs, API responses, emails, QR codes, file names):

- **Enumeration:** if `/invoices/1041` works, someone will try `/invoices/1040`. One missed access check becomes a data leak.
- **Business information:** invoice 1041 this week and 1104 next week tells a competitor how many you issue.

Add a **UUIDv7** next to the internal key:

```sql
CREATE TABLE invoices (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,   -- joins, foreign keys
    public_id  uuid NOT NULL DEFAULT uuidv7(),                    -- URLs and API responses
    -- ...
    CONSTRAINT uq_invoices_public_id UNIQUE (public_id)
);
```

| Where the UUID comes from | How |
| --- | --- |
| PostgreSQL 18+ | `DEFAULT uuidv7()` (built in) |
| PostgreSQL 17 and older | Generate in the app and insert it |
| Node | `import { v7 as uuidv7 } from 'uuid'` (package `uuid`, version 10 or later) |
| .NET 9+ | `Guid.CreateVersion7()` |

**Why v7, not v4?** v7 starts with a timestamp, so new values land at the end of the index like a sequence. Random v4 values land anywhere, splitting pages and pushing hot pages out of memory as the table grows.

> 📝 A UUIDv7 reveals roughly when the row was created. Fine for invoices and bookings. If creation time is sensitive, use a v4 for that column.

**UUID as the primary key itself** only when rows are created outside the database and synced in (offline mobile apps), or merged from several databases. Otherwise `bigint` + `public_id` keeps joins and foreign keys at 8 bytes instead of 16.

| Engine | UUID type | Watch out for |
| --- | --- | --- |
| PostgreSQL | `uuid` | Nothing. Store it as `uuid`, never `text`. |
| SQL Server | `uniqueidentifier` | Sorted by its **last** bytes, so a UUIDv7 does **not** arrive in order. Keep `bigint IDENTITY` as the clustered key, the UUID in a non-clustered unique index. |
| MySQL | `BINARY(16)` | InnoDB stores rows in primary-key order, so a random primary key hurts it more than most. `CHAR(36)` works but takes more than twice the space. |

* * *

## Data types

| Purpose | PostgreSQL | SQL Server | MySQL | Notes |
| --- | --- | --- | --- | --- |
| Text | `text` | `nvarchar(n)` | `VARCHAR(n)` with `utf8mb4` | On Postgres, `text` and `varchar(n)` perform the same. Length `CHECK` only when the limit is a business rule. |
| Whole numbers | `integer`, `bigint` | `int`, `bigint` | `INT`, `BIGINT` | `smallint` rarely saves enough to be worth a future overflow. |
| Money, exact decimals | `numeric(12,2)` | `decimal(12,2)` | `DECIMAL(12,2)` | Or `bigint` cents. Never float. |
| Yes / no | `boolean` | `bit` | `BOOLEAN` (stored as `TINYINT(1)`) | `NOT NULL` with a default. A nullable boolean has three states. |
| Moment in time | `timestamptz` | `datetimeoffset`, or `datetime2` always in UTC | `DATETIME` always in UTC | See [General Principles → Store time in UTC](General-Principles.md#store-time-in-utc). |
| Calendar date | `date` | `date` | `DATE` | Birthdays, due dates, holidays. |
| Duration | `interval` | `int` seconds | `INT` seconds | |
| UUID | `uuid` | `uniqueidentifier` | `BINARY(16)` | See [Primary keys](#ids-that-leave-the-system). |
| Loose structured data | `jsonb` | `nvarchar(max)` with an `ISJSON` check, or the native `json` type on SQL Server 2025 and Azure SQL | `JSON` | See below. |
| Email | `text`, unique on `lower(email)` | `nvarchar(320)` | `VARCHAR(320)` | Case-insensitive uniqueness, see [Constraints](#constraints). |
| IP address | `inet` | `varchar(45)` | `VARBINARY(16)` or `VARCHAR(45)` | |
| Files, images | Don't | Don't | Don't | Object storage (S3, Azure Blob); keep the key and metadata in the table. |

> ⚠️ **Avoid `char(n)` on Postgres.** It pads values with spaces to the full length. Use `text` with a `CHECK (char_length(code) = 3)` for fixed-length codes like a currency.

### JSON columns

`jsonb` is for data whose shape really varies per row (webhook payloads, per-product attributes, user preferences), **not** a way to avoid designing a schema.

| Put it in `jsonb` | Make it a real column |
| --- | --- |
| You store it and show it back, but rarely filter on it | You filter, sort or join on it |
| Its keys differ from row to row | Every row has it |
| It comes from an outside system and you don't control its shape | It needs a foreign key, NOT NULL or CHECK |

- `jsonb`, not `json`: stored parsed, GIN-indexable, faster to query. Plain `json` only if you must keep the exact original text, whitespace and key order included.
- Filtering on the same JSON key in most queries? Promote it to a column.

### Status columns and fixed value sets

| Option | Use when | Example |
| --- | --- | --- |
| `text` + `CHECK` | A short list that only changes with a code release | `status text NOT NULL CHECK (status IN ('draft', 'sent', 'paid', 'void'))` |
| Lookup table + foreign key | Values carry extra data (label, sort order, colour), or admins add values | `appointment_types (id, name, duration_minutes)` |
| Postgres `enum` type | The list will practically never change, and you want the type to document it | `CREATE TYPE payment_method AS ENUM ('card', 'bank_transfer', 'cash');` |

> ⚠️ **Postgres enums are easy to grow and hard to shrink.** `ALTER TYPE ... ADD VALUE` works, but a value can never be removed. Prisma `enum`s become Postgres enum types, so this applies to them too. A `CHECK` constraint can be dropped and recreated in one migration.

* * *

## NULL

`NULL` means "no value" or "unknown". It is not `''`, not `0` and not `false`.

- **Default to `NOT NULL`.** Allow NULL only when "no value" is a real state: `paid_at` before payment, `cancelled_at` on a booking that wasn't cancelled.
- **No sentinel values** (`''`, `0`, `'1900-01-01'`, `-1`) for "unknown". Every query must know the magic value, and aggregates count it as real data.
- **Avoid nullable booleans** (three states). If you really need three, use a `text` column with a `CHECK` and name them.

```sql
-- ❌ '' for "no phone" — now every report has to remember to exclude ''
phone text NOT NULL DEFAULT ''

-- ✅ no phone is NULL, and COUNT(phone) counts only real ones
phone text NULL
```

**NULLs in a UNIQUE constraint, per engine:**

| Engine | Several rows with NULL in a unique column? |
| --- | --- |
| PostgreSQL | Allowed. Postgres 15+ can forbid it with `UNIQUE NULLS NOT DISTINCT`. |
| SQL Server | **Only one NULL allowed.** For "unique when present", use a filtered unique index (`WHERE col IS NOT NULL`). |
| MySQL | Allowed. |

NULL in `WHERE`, `NOT IN` and comparisons: [Writing Queries](Writing-Queries.md).

* * *

## Constraints

A constraint is checked on every write, from every app, script and console.

```sql
CREATE TABLE appointments (
    id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    patient_id   bigint      NOT NULL REFERENCES patients (id),
    doctor_id    bigint      NOT NULL REFERENCES doctors (id),
    starts_at    timestamptz NOT NULL,
    ends_at      timestamptz NOT NULL,
    status       text        NOT NULL DEFAULT 'booked',
    created_at   timestamptz NOT NULL DEFAULT now(),
    updated_at   timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT ck_appointments_status    CHECK (status IN ('booked', 'completed', 'cancelled', 'no_show')),
    CONSTRAINT ck_appointments_time_order CHECK (ends_at > starts_at)
);
```

| Constraint | Use it for |
| --- | --- |
| `PRIMARY KEY` | Every table has one. |
| `NOT NULL` | Every column that must always have a value. |
| `UNIQUE` | Anything that must not repeat: `email`, `invoice.number`, `(clinic_id, room_name)`. |
| `CHECK` | Ranges, allowed values, and rules across columns in the same row (`ends_at > starts_at`). |
| `FOREIGN KEY` | Every reference to another table. See below. |
| `EXCLUDE` (Postgres) | "No two rows may overlap", e.g. double-booked appointments. |

**Case-insensitive uniqueness**, e.g. emails:

```sql
-- ✅ John@Example.com and john@example.com count as the same address
CREATE UNIQUE INDEX uq_users_email ON users (lower(email));
```

Queries must then compare `lower(email) = lower($1)` to use the index. Or use the `citext` extension (a case-insensitive text type), so a plain `UNIQUE` works.

**Stop overlapping bookings in the database itself** (Postgres only). Unlike an app-side "is this slot free?" check, this holds however close together two requests arrive:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;   -- lets the = on doctor_id work in a GiST index

ALTER TABLE appointments
    ADD CONSTRAINT ex_appointments_no_double_booking
    EXCLUDE USING gist (
        doctor_id WITH =,
        tstzrange(starts_at, ends_at) WITH &&
    )
    WHERE (status <> 'cancelled');
```

### Foreign keys and ON DELETE

> ⚠️ **Postgres and SQL Server do not index foreign key columns for you** (MySQL does). Without one, deleting a customer and listing "invoices for this customer" both scan the whole invoices table. Add one for every foreign key:
> ```sql
> CREATE INDEX ix_invoices_customer_id ON invoices (customer_id);
> ```

| `ON DELETE` | Means | Use it for |
| --- | --- | --- |
| `NO ACTION` / `RESTRICT` (default) | The delete fails while children exist | Most references: you can't delete a customer who has invoices |
| `CASCADE` | Children are deleted with the parent | Children that can't exist on their own: `invoice_lines` of an invoice, rows in a join table |
| `SET NULL` | The reference becomes NULL | Optional references: `assigned_user_id` when that user is deleted |

> 📝 `NO ACTION` and `RESTRICT` behave the same unless the constraint is `DEFERRABLE`. `NO ACTION` can wait until the end of the transaction. `RESTRICT` can't.

| Tool | Default when you don't say | How to set it |
| --- | --- | --- |
| Prisma | `Restrict` for required relations, `SetNull` for optional ones | `@relation(..., onDelete: Restrict)` |
| TypeORM | `NO ACTION` | `@ManyToOne(() => Customer, { onDelete: 'RESTRICT' })` |
| Laravel | No action | `->constrained()->restrictOnDelete()`, `->cascadeOnDelete()`, `->nullOnDelete()` |
| EF Core | **`Cascade` for required relationships** | `.OnDelete(DeleteBehavior.Restrict)` |

> ⚠️ **EF Core cascades deletes on required relationships by default.** Deleting one customer quietly deletes every invoice, payment and line under them. Set `DeleteBehavior.Restrict` explicitly unless the child really is owned by the parent.

* * *

## Normalize first then denormalize on purpose

Store each fact **once**. Two copies will eventually disagree.

```sql
-- ❌ a list in a column: can't join it, can't index it, can't stop a typo'd ID
tag_ids text   -- '3,17,42'

-- ❌ the customer's name copied into every invoice: rename the customer and old invoices disagree
invoices (id, customer_id, customer_name, customer_email, ...)

-- ✅ reference the customer, join when you need the name
invoices (id, customer_id, ...)
```

**A snapshot is not duplication.** Some values must be frozen at the moment something happened:

| Snapshot (correct) | Why |
| --- | --- |
| `invoice_lines.unit_price` | The product's price changes next month. The invoice must not. |
| `invoices.billing_address` | The customer moves. Last year's invoice keeps the old address. |
| `orders.tax_rate` | Tax rates change. |

**Deliberate denormalization** (a cached total, a counter, a reporting table) is fine when a measured performance problem calls for it:

- Document which column is the copy and where the source of truth is.
- Keep **one** place that updates it (a trigger, or one service method), never "remember to update it everywhere".
- Prefer a [generated column](#generated-columns) or a materialized view when it can do the job.

* * *

## Soft delete

```sql
deleted_at timestamptz NULL   -- NULL = live, a timestamp = deleted (and when)
```

**Use it when** users need an undo, or you must keep history: cancelled bookings, archived customers, removed products that old orders still point to.

**Don't use it for** personal data someone asked you to erase (the row is still stored: that needs a real delete or anonymization), high-churn tables like sessions, carts, tokens and logs (they grow forever and every query wades through dead rows), or join tables (remove the row).

**Unique constraints must ignore deleted rows**, or a deleted user's email can never be registered again:

```sql
-- ✅ unique among live rows only (a partial unique index)
CREATE UNIQUE INDEX uq_users_email_live ON users (lower(email)) WHERE deleted_at IS NULL;
```

SQL Server calls this a filtered index (same `WHERE`). MySQL has no partial indexes: put the unique index on a generated column that is NULL for deleted rows.

**Every query must exclude deleted rows**, so let the framework add the filter:

| Tool | How |
| --- | --- |
| Laravel | `use SoftDeletes;` on the model, `$table->softDeletes();` in the migration |
| TypeORM | `@DeleteDateColumn()` on the entity, then `softDelete()` / `softRemove()`. `find` skips deleted rows. |
| EF Core | A global query filter: `modelBuilder.Entity<Customer>().HasQueryFilter(c => c.DeletedAt == null);` |
| Prisma | No built-in support. Filter explicitly, or add it once with a Prisma Client extension. |
| Dapper, raw SQL | `WHERE deleted_at IS NULL`, every time. Consider a view (`live_customers`) to query instead. |

* * *

## Audit columns

```sql
created_at timestamptz NOT NULL DEFAULT now(),
updated_at timestamptz NOT NULL DEFAULT now()
```

A default only applies on insert, so something has to set `updated_at` on every update:

| Who sets it | Covers | Misses |
| --- | --- | --- |
| Prisma `@updatedAt` | Updates made through Prisma Client | `$executeRaw`, other apps, console fixes |
| TypeORM `@UpdateDateColumn()` | Updates through TypeORM | Raw queries |
| Laravel `$table->timestamps()` | Eloquent model saves and Eloquent query `update()` | `DB::table(...)->update()` |
| EF Core `SaveChanges` override or interceptor | Updates through that `DbContext` | Dapper and raw SQL |
| A Postgres trigger | **Every** update, from anywhere | Nothing |

When more than one thing writes to the table (EF Core plus Dapper, scripts, console fixes), use the trigger. When triggers are and aren't a good idea: [Logic in the Database](Logic-in-the-Database.md).

```sql
CREATE FUNCTION set_updated_at() RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
    NEW.updated_at := now();
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_invoices_set_updated_at
    BEFORE UPDATE ON invoices
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

> 💡 `created_by_user_id` / `updated_by_user_id` tell you who touched a row last. The **full history** of every change (common for FI clients) needs an audit table or log, not more columns.

* * *

## Many-to-many join tables

```sql
-- ✅ one row per pairing; the composite key stops duplicates
CREATE TABLE user_roles (
    user_id  bigint NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    role_id  bigint NOT NULL REFERENCES roles (id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

-- the primary key covers lookups by user_id; add one for the other direction
CREATE INDEX ix_user_roles_role_id ON user_roles (role_id);
```

A pairing with its own data (`assigned_at`, `quantity`, `price`) is an entity: give it a real name, and its own `id` if other tables will point at it (`order_items`, not `order_products`).

* * *

## Generated columns

```sql
CREATE TABLE invoice_lines (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    invoice_id  bigint        NOT NULL REFERENCES invoices (id) ON DELETE CASCADE,
    quantity    integer       NOT NULL CHECK (quantity > 0),
    unit_price  numeric(12,2) NOT NULL CHECK (unit_price >= 0),
    line_total  numeric(12,2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);
```

- `STORED` (Postgres 12+) computes on write, so it can be indexed. Postgres 18 adds `VIRTUAL` (computed on read) and makes it the default when you specify neither.
- Only same-row columns and immutable functions: no other tables, no `now()`.
- SQL Server: computed columns (`AS (quantity * unit_price) PERSISTED`). MySQL 5.7+: `GENERATED ALWAYS AS (...) VIRTUAL | STORED`.

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Plural snake_case tables, singular columns, `<table>_id` foreign keys | Prefix tables with `tbl_`, or mix naming styles | [Naming](#the-rules) |
| Name constraints and indexes in hand-written SQL | Leave Postgres to call it `invoices_check1` | [Constraint and index names](#constraint-and-index-names) |
| Map names with `@map`, a TypeORM naming strategy, or `UseSnakeCaseNamingConvention()` | Quote PascalCase tables in every Postgres query | [ORM mapping](#mapping-names-in-the-orm) |
| `@db.Timestamptz(6)` on Prisma DateTime fields | Accept Prisma's default `timestamp(3)` | [ORM mapping](#mapping-names-in-the-orm) |
| `bigint GENERATED ALWAYS AS IDENTITY` | `integer` or `serial` keys on new tables | [Primary keys](#the-default) |
| A UUIDv7 `public_id` for anything in a URL or API | Expose sequential IDs | [IDs that leave the system](#ids-that-leave-the-system) |
| `text`, `numeric`, `timestamptz`, `date`, `uuid` | `char(n)`, float money, `timestamp`, UUIDs as text | [Data types](#data-types) |
| `jsonb` for loose data you rarely filter on | JSON for data you filter, join or constrain | [JSON columns](#json-columns) |
| `CHECK` or a lookup table for status values | A Postgres enum for a list that will change | [Fixed value sets](#status-columns-and-fixed-value-sets) |
| NOT NULL by default, NULL for a real "no value" | `''`, `0` or `1900-01-01` as "unknown" | [NULL](#null) |
| Index every foreign key, set ON DELETE on purpose | Rely on EF Core's cascade default | [Foreign keys](#foreign-keys-and-on-delete) |
| An exclusion constraint for "no overlapping bookings" | An app-side availability check alone | [Constraints](#constraints) |
| Snapshot prices and addresses on the invoice | Copy a customer's name into every invoice | [Normalization](#normalize-first-then-denormalize-on-purpose) |
| Partial unique index with soft delete | A plain UNIQUE that blocks re-registration | [Soft delete](#soft-delete) |
| A trigger for `updated_at` when several tools write | Assume `DEFAULT now()` keeps `updated_at` current | [Audit columns](#audit-columns) |
| A join table with a composite primary key | `tag_ids text` with comma-separated IDs | [Many-to-many](#many-to-many-join-tables) |

* * *

## Further reading

1.  **PostgreSQL Wiki — Don't Do This**
    [https://wiki.postgresql.org/wiki/Don't_Do_This](https://wiki.postgresql.org/wiki/Don't_Do_This)
2.  **PostgreSQL Documentation — Data Types**
    [https://www.postgresql.org/docs/current/datatype.html](https://www.postgresql.org/docs/current/datatype.html)
3.  **PostgreSQL Documentation — Constraints**
    [https://www.postgresql.org/docs/current/ddl-constraints.html](https://www.postgresql.org/docs/current/ddl-constraints.html)
4.  **PostgreSQL Documentation — Generated Columns**
    [https://www.postgresql.org/docs/current/ddl-generated-columns.html](https://www.postgresql.org/docs/current/ddl-generated-columns.html)
5.  **RFC 9562 — Universally Unique IDentifiers (UUIDs)**, including UUIDv7
    [https://www.rfc-editor.org/rfc/rfc9562](https://www.rfc-editor.org/rfc/rfc9562)
6.  **Prisma — Database mapping** (`@map`, `@@map`, constraint names)
    [https://www.prisma.io/docs/orm/prisma-schema/data-model/database-mapping](https://www.prisma.io/docs/orm/prisma-schema/data-model/database-mapping)
7.  **EF Core — Cascade Delete**
    [https://learn.microsoft.com/en-us/ef/core/saving/cascade-delete](https://learn.microsoft.com/en-us/ef/core/saving/cascade-delete)
8.  **EFCore.NamingConventions**
    [https://github.com/efcore/EFCore.NamingConventions](https://github.com/efcore/EFCore.NamingConventions)

* * *

## Related pages

- [General Principles](General-Principles.md) — the rules this page expands on.
- [Indexing](Indexing.md) — which indexes to add beyond primary and foreign keys.
- [Migrations and Schema Changes](Migrations-and-Schema-Changes.md) — how to change any of this on a live table.
- [PostgreSQL](PostgreSQL.md) — JSONB, arrays, enums and extensions in more depth.

---

[⬅ Back to Database Standards](README.md)
