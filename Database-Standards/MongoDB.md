MongoDB
=======

> **MongoDB is a NoSQL document database**, and **not** our default: new projects start on [PostgreSQL](/Database-Standards/PostgreSQL). Read [General Principles](/Database-Standards/General-Principles) first. Most of it still applies here.

* * *

## What it is and when to use it

| | |
| --- | --- |
| **What it is** | A **NoSQL document database**. Each record is a document (BSON, a binary form of JSON) that can hold nested objects and arrays. Documents in one collection don't have to share a shape. |
| **Use it for** | Data that is mostly **self-contained documents, read and written as a whole**, whose shape really varies from record to record: CMS content blocks, form submissions with client-defined fields, event and audit payloads, product catalogues where every category has different attributes. Also when the client already runs MongoDB. |
| **Don't use it for** | Data with relationships you need to join on (patients ↔ appointments ↔ invoices), money and ledgers, reporting across many entities, or "we don't want to design a schema yet". A `jsonb` column in Postgres covers most "flexible data" needs without a second database. |
| **Versions** | **8.0 or newer** for anything new (8.0 is supported until 31 October 2029; 9.0 is the current major release). **7.0 reaches end of life on 31 August 2027**, so plan the upgrade. The in-between "rapid releases" (8.1, 8.2, 8.3) are supported on Atlas only. Check the [MongoDB lifecycle page](https://www.mongodb.com/legal/support-policy/lifecycles) before you pick. |

> ⚠️ Choosing MongoDB as a project's main database is an exception to our default. Agree it with the tech lead and write down why in the README. See [How to Use This Wiki → Document every exception](/How-to-Use-This-Wiki#document-every-exception).

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [Postgres JSONB or MongoDB](#postgres-jsonb-or-mongodb) | **Try a JSONB column in Postgres first**<br/><sub>MongoDB earns its place only when most of the data is documents read and written as a whole.</sub> |
| [Modeling](#design-for-how-the-data-is-read) | **Design documents around how the app reads them**<br/><sub>Start from the screens and queries, not from an entity diagram.</sub> |
| [Embed or reference](#embed-or-reference) | **Embed what's read together and bounded, reference what's shared or unbounded**<br/><sub>A document has a hard 16 MB limit, and a big document is slow long before that.</sub> |
| [Unbounded arrays](#avoid-unbounded-arrays) | **No array that grows forever**<br/><sub>Cap it with `$slice`, or move the items into their own collection.</sub> |
| [Validation](#validate-with-a-json-schema) | **Every collection gets a `$jsonSchema` validator**<br/><sub>"Schemaless" means the database won't stop bad data unless you tell it to.</sub> |
| [Indexes](#index-every-query-you-run) | **Every query pattern has an index, compound indexes in ESR order**<br/><sub>Equality fields, then Sort fields, then Range fields.</sub> |
| [Explain](#read-the-explain-output) | **A `COLLSCAN` on a real query is a bug**<br/><sub>Check `explain("executionStats")` for any query on a growing collection.</sub> |
| [Pagination](#paginate-by-range-not-skip) | **Paginate by range on an indexed field, never a large `skip()`**<br/><sub>`skip(10000)` still reads 10,000 documents before returning anything.</sub> |
| [Atomic updates](#prefer-single-document-atomic-updates) | **Update operators on one document before reaching for a transaction**<br/><sub>`$inc`, `$set` and `$push` on a single document are atomic on their own.</sub> |
| [Transactions](#transactions-when-you-need-them) | **Short, small, and only on a replica set**<br/><sub>They work, but they cost more than in a relational database and time out after 60 seconds by default.</sub> |
| [Operator injection](#operator-injection) | **Validate the shape of every input, never pass `req.body` as a filter**<br/><sub>`{ "$ne": null }` in a JSON body turns "find this token" into "find any token".</sub> |
| [Server security](#lock-down-the-server) | **Auth on, never reachable from the internet, TLS between app and database**<br/><sub>Open MongoDB servers are found and wiped by bots within hours.</sub> |
| [Schema changes](#schema-changes-and-migrations) | **Version documents and run changes as committed migration scripts**<br/><sub>No schema doesn't mean no migrations. It means the old shapes stay around until you move them.</sub> |

* * *

## Concepts you'll meet on this page

### Document and collection

A **document** is one record: a JSON-like object with fields, nested objects and arrays, with a unique `_id`. A **collection** is a group of documents, roughly a SQL table.

### BSON and ObjectId

**BSON** is MongoDB's storage format, adding types JSON lacks (dates, 64-bit integers, decimals, `ObjectId`). An **ObjectId** is the default `_id`: 12 bytes, unique, roughly sorted by creation time because it starts with a timestamp.

> 📝 Anyone who sees an ObjectId can tell roughly when the record was created. Usually fine. If it isn't, use another public ID.

### Replica set

Three or more servers holding the same data (one primary, the rest secondaries) with automatic failover. **Transactions and change streams need a replica set**, so local development runs a one-node replica set.

### Aggregation pipeline

MongoDB's equivalent of a SQL query with `GROUP BY`, `JOIN` and computed columns: a list of stages (`$match`, `$group`, `$sort`, `$lookup`...) that documents flow through in order.

* * *

## When to use MongoDB

### Postgres JSONB or MongoDB

| Situation | Choose |
| --- | --- |
| A relational app with a few flexible fields (custom form answers, settings, integration payloads) | **Postgres** with a `jsonb` column |
| You need joins, foreign keys, reporting across entities, or money | **Postgres** |
| Most of the data is documents read and written whole, with shapes that really vary, and few relationships between them | **MongoDB** |
| Very high write volume of independent documents that has to scale out across servers (sharding) | **MongoDB**, agreed with the tech lead |
| The client already runs MongoDB and the project extends that system | **MongoDB** |

> 💡 If most queries need `$lookup`, the data is relational. That's a sign the project wanted Postgres.

* * *

## Data modeling

### Design for how the data is read

Start from the queries, not a normalized design: list the screens and API calls, see which data each needs together, and shape documents so the most frequent reads touch **one document**. In the clinic app the busiest read is "today's appointments for this clinic, with the patient's name", so the appointment carries the patient's name:

```js
// appointments collection
{
  _id: ObjectId("6720f1c2a1b2c3d4e5f60789"),
  clinicId: ObjectId("6720e9aa1b2c3d4e5f601234"),
  // extended reference: only the fields lists show
  patient: { _id: ObjectId("6720ea001b2c3d4e5f605678"), name: "Amina Khan", phone: "+92 300 1234567" },
  startsAt: ISODate("2026-11-10T09:30:00Z"), durationMinutes: 20, status: "booked", schemaVersion: 2
}
```

The full patient record stays in `patients`. When a patient changes their name, update both: decide up front which fields are copied and who keeps them in step.

### Embed or reference

| Embed it in the parent document when... | Reference it by `_id` when... |
| --- | --- |
| It's always read together with the parent | It's often read on its own |
| It belongs to one parent only | It's shared by many parents (one clinic, many appointments) |
| The number of items is small and **bounded** (a patient's 2–3 phone numbers) | The number of items can grow without limit (a clinic's appointments) |
| It changes along with the parent | It changes independently and often |

> ⚠️ **A document can't be bigger than 16 MB.** That's a hard limit, not a guideline, and a document gets slow to read, write and replicate long before it gets there.

### Avoid unbounded arrays

```js
// ❌ every appointment pushed into the clinic document: it grows forever and every read loads all of it
{ _id: clinicId, name: "Gulberg Clinic", appointments: [ /* 40,000 items and counting */ ] }
// ✅ appointments are their own collection, pointing at the clinic
{ _id: appointmentId, clinicId: clinicId, startsAt: ISODate("2026-11-10T09:30:00Z"), ... }
// ✅ a short list inside a document is fine if capped as you write: keep only the 20 most recent notes
db.patients.updateOne({ _id: patientId }, { $push: { recentNotes: { $each: [note], $slice: -20 } } });
```

### Validate with a JSON Schema

Add a `$jsonSchema` validator to every collection, so a bug or a script can't write a document the app can't read. ODM schemas (Mongoose, the C# classes) only protect writes that go through the app. The validator protects all of them.

```js
db.createCollection("appointments", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["clinicId", "patient", "startsAt", "status", "schemaVersion"],
    properties: {
      clinicId:        { bsonType: "objectId" },
      patient:         { bsonType: "object", required: ["_id", "name"],
                         properties: { _id: { bsonType: "objectId" }, name: { bsonType: "string", minLength: 1 } } },
      startsAt:        { bsonType: "date" },
      durationMinutes: { bsonType: "int", minimum: 5, maximum: 240 },
      status:          { enum: ["booked", "checked_in", "completed", "cancelled"] },
      schemaVersion:   { bsonType: "int" }
    }
  } },
  validationLevel: "strict",     // check every insert and update
  validationAction: "error"      // reject, don't just log
});
```

To change the validator on an existing collection, run `db.runCommand({ collMod: "appointments", validator: { ... } })` from a migration.

> 💡 On a collection with old documents, `validationLevel: "moderate"` only checks documents that already pass. Use it while you migrate the old ones, then switch to `"strict"`.

### Schema design patterns

| Pattern | Problem it solves | Clinic app example |
| --- | --- | --- |
| **Extended reference** | Avoid a `$lookup` for fields you always show | Copy the patient's name and phone into each appointment |
| **Subset** | A document is big but most reads only need a small part | Keep the 20 most recent notes on the patient, the full history in its own collection |
| **Computed** | The same calculation runs on every read | Store `appointmentsThisMonth` on the clinic, updated with `$inc` on each booking |
| **Bucket** | Huge numbers of tiny time-based records | One document per device per hour of readings. For pure time-series data, use a **time series collection**, which buckets for you. |
| **Outlier** | A few documents break the normal shape | Flag the rare clinic with 10,000 practitioners (`hasOverflow: true`) and store the extra in an overflow collection |
| **Schema versioning** | Documents of different shapes live side by side during a change | `schemaVersion: 2` on every document. See [Schema changes](#schema-changes-and-migrations). |

* * *

## Indexes and queries

### Index every query you run

Every query on a growing collection needs a supporting index, created in a migration script, not by hand in Compass or the Atlas UI.

```js
db.appointments.createIndex({ clinicId: 1, startsAt: 1 });
db.patients.createIndex({ clinicId: 1, email: 1 }, { unique: true });   // also enforces uniqueness
```

### Compound indexes follow the ESR rule

Order fields **Equality** (matched exactly, `clinicId: X`), then **Sort** (fields in `sort()`), then **Range** (`$gt`, `$lt`, `$in` with many values, regex...).

```js
// "Patients aged 65+ at this clinic, by surname"
db.patients.find({ clinicId: X, age: { $gte: 65 } }).sort({ lastName: 1 });
// ✅ Equality (clinicId), Sort (lastName), Range (age): reads in order, no in-memory sort
db.patients.createIndex({ clinicId: 1, lastName: 1, age: 1 });
// ❌ Range before Sort: results come back out of order and MongoDB sorts them in memory
db.patients.createIndex({ clinicId: 1, age: 1, lastName: 1 });
```

Left-to-right applies as in SQL: `{ clinicId: 1, lastName: 1 }` helps queries on `clinicId` alone, not on `lastName` alone.

### Other index types

| Index | Use it for | Example |
| --- | --- | --- |
| **Multikey** | Querying inside an array. Created automatically when the indexed field is an array. A compound index can include at most one array field. | `{ "tags": 1 }` |
| **TTL** | Deleting documents automatically after a time: sessions, one-time codes, temporary bookings | `createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })` removes each document once its `expiresAt` date has passed |
| **Unique** | Enforcing "no duplicates", like a SQL UNIQUE constraint | `{ clinicId: 1, email: 1 }, { unique: true }` |
| **Partial** | Indexing only the documents a query cares about, so the index stays small | `{ startsAt: 1 }, { partialFilterExpression: { status: "booked" } }` |
| **Text** | Basic word search on a self-hosted server. Only one text index per collection. | `{ title: "text", body: "text" }` |
| **Atlas Search** | Real search (relevance, typo tolerance, autocomplete) on Atlas | Defined as a search index in Atlas, queried with `$search` |

> 📝 TTL cleanup runs about once a minute, so an expired document can stay readable briefly. If that matters, also filter on `expiresAt` in the query.

### Read the explain output

```js
db.appointments.find({ clinicId: X, startsAt: { $gte: ISODate("2026-11-10T00:00:00Z") } })
  .sort({ startsAt: 1 }).explain("executionStats");
```

| Look at | Good | Bad |
| --- | --- | --- |
| Winning plan stage | `IXSCAN` (used an index) | `COLLSCAN` (read the whole collection) |
| `totalDocsExamined` vs `nReturned` | Close to each other | Examined is many times returned: the index isn't selective enough |
| A `SORT` stage | Absent | Present: the sort happens in memory. Fix the index order (ESR). |

On Atlas, the **Performance Advisor** suggests missing indexes from real traffic and the **Query Profiler** shows slow queries. Self-hosted, the server logs every operation slower than 100 ms by default.

### Aggregation pipelines

- **`$match` first**, so the pipeline can use an index and later stages handle fewer documents.
- **`$project` early** to drop fields you don't need.
- **`$lookup` is a join, and it costs like one.** Fine for an occasional report. In a hot path, the document should usually carry the data (extended reference).

```js
// Appointments per status for one clinic in November
db.appointments.aggregate([
  { $match: { clinicId: X, startsAt: { $gte: ISODate("2026-11-01T00:00:00Z"), $lt: ISODate("2026-12-01T00:00:00Z") } } },
  { $group: { _id: "$status", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]);
```

### Paginate by range not skip

```js
// ❌ reads and throws away 10,000 documents to show page 501
db.appointments.find({ clinicId: X }).sort({ _id: 1 }).skip(10000).limit(20);
// ✅ next page starts after the last _id the client saw (index on { clinicId: 1, _id: 1 })
db.appointments.find({ clinicId: X, _id: { $gt: lastSeenId } }).sort({ _id: 1 }).limit(20);
```

When sorting by a non-unique field like `startsAt`, add `_id` as a tie-breaker in both the sort and the index, and continue from the pair: `{ $or: [ { startsAt: { $gt: last.startsAt } }, { startsAt: last.startsAt, _id: { $gt: last._id } } ] }`. The SQL version is on [Writing Queries](/Database-Standards/Writing-Queries).

* * *

## Writes and transactions

### Prefer single-document atomic updates

A write to one document, nested fields included, is always atomic. Use update operators instead of read, change in code, write back.

```js
// ❌ read-modify-write: two bookings at the same moment both see 1 slot left, both book it
const slot = await slots.findOne({ _id: slotId });
if (slot.remaining > 0) {
  await slots.updateOne({ _id: slotId }, { $set: { remaining: slot.remaining - 1 } });
}
// ✅ the condition and the change happen in one atomic step
const result = await slots.updateOne({ _id: slotId, remaining: { $gt: 0 } }, { $inc: { remaining: -1 } });
if (result.modifiedCount === 0) throw new ConflictException('Slot is full');
```

### Transactions when you need them

Multi-document ACID transactions exist (replica sets since 4.0, sharded clusters since 4.2), but:

- They need a **replica set** or sharded cluster, never a standalone server.
- A transaction is aborted if it runs longer than **60 seconds** by default.
- They cost more than single-document writes. MongoDB's guidance: keep them small (no more than 1,000 documents modified in one).

```ts
// withTransaction retries automatically on transient errors
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    const slot = await slots.updateOne({ _id: slotId, remaining: { $gt: 0 } }, { $inc: { remaining: -1 } }, { session });
    if (slot.modifiedCount === 0) throw new ConflictException('Slot is full');   // aborts the transaction
    await appointments.insertOne(appointment, { session });
  });
} finally {
  await session.endSession();
}
```

> 💡 Frequent multi-document transactions across collections are a sign of relational data. Revisit [Postgres JSONB or MongoDB](#postgres-jsonb-or-mongodb).

### Write concern and read concern

- **Write concern** (how many servers must confirm a write): the default `w: "majority"` (since 5.0) survives the primary failing. Don't lower it to `w: 1` for data you care about, and never use `w: 0`.
- **Read concern `"majority"`** returns only data that can't be rolled back. Use it for reads that drive decisions (balances, slot counts).
- **Secondary reads** (`readPreference: "secondaryPreferred"`) spread load but can return slightly old data. Not for read-after-write flows like "save, then show the saved record".

* * *

## Security

### Operator injection

MongoDB filters are objects, so if user input reaches a filter as an object instead of a string, the user can send query operators: MongoDB's version of SQL injection.

```ts
// ❌ the body is used as-is. Sending { "token": { "$ne": null } } matches the first user with any reset token
const user = await User.findOne({ resetToken: req.body.token });
// ✅ validate the shape first: token must be a string, so an operator object is rejected with a 400
const { token } = z.object({ token: z.string().min(32) }).parse(req.body);
const user = await User.findOne({ resetToken: token });
```

- **Validate every input's type** at the boundary (Zod or class-validator in NestJS, form requests in Laravel, typed models in .NET). In PHP, `?token[$ne]=x` arrives as an **array**; Laravel validation with `'token' => ['required', 'string']` blocks it.
- **Never pass `req.body` or `req.query` straight into a filter**, and never let user input build `$where`, `$expr` or `$function`.
- With Mongoose, also turn on `mongoose.set('sanitizeFilter', true)`, which wraps filter values in `$eq`. Keep Mongoose patched: versions before 8.22.1 (and 9.1.6 on the 9.x line) could be bypassed with `$nor` (CVE-2026-42334).

### Lock down the server

Atlas does most of this by default. Self-hosted, check every item:

```yaml
# /etc/mongod.conf
security:
  authorization: enabled                  # no access without a user and password
net:
  bindIp: 127.0.0.1,10.0.1.12             # loopback and the private network only
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb.pem
```

- **Never expose port 27017 to the internet.** Firewall it to the app servers. On Atlas, use the IP access list or a private endpoint, never `0.0.0.0/0`.
- **One database user per app** with the `readWrite` role on its own database only. Reporting tools get `read`. Nobody uses an admin user from the app. Users, roles and credentials: see [Access and Roles](/Database-Standards/Access-and-Roles).

* * *

## Drivers and ODMs in our stacks

| Stack | Use | Notes |
| --- | --- | --- |
| NestJS | **Mongoose** via `@nestjs/mongoose` | The default. Schemas, validation and hooks in code. Turn on `sanitizeFilter`. |
| NestJS | Native `mongodb` driver | Scripts, migrations and anything Mongoose gets in the way of. |
| NestJS | Prisma | **Don't start a MongoDB project on Prisma.** Prisma ORM 7 doesn't support MongoDB at the time of writing; MongoDB projects have to stay on Prisma 6. Check the [Prisma MongoDB docs](https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/mongodb) for the current state. |
| Laravel | **`mongodb/laravel-mongodb`** | MongoDB's official Laravel integration (formerly `jenssegers/mongodb`). Needs the `mongodb` PHP extension. Models extend `MongoDB\Laravel\Eloquent\Model`, and most of Eloquent and the query builder work as usual. |
| .NET | **`MongoDB.Driver`** (official C# driver) | The default. LINQ or the builders API, typed classes per collection. |
| .NET | `MongoDB.EntityFrameworkCore` | MongoDB's official EF Core provider (GA since 2024). Fine if the team wants EF Core patterns, but it supports a subset of EF Core features, so drop to the driver for anything advanced. |

Whatever the stack: create **one client per application** and reuse it. The client holds a connection pool, so creating one per request exhausts the server's connections.

* * *

## Schema changes and migrations

MongoDB won't stop you deploying code that expects a new shape while old documents keep the old one, so every change needs a plan for existing data:

1. **Add a `schemaVersion` field** to every document.
2. **Make the code read both versions** (old and new) and write only the new one.
3. **Migrate the old documents** with a committed script, in batches.
4. **Remove the old-version code** once nothing has the old version left, and tighten the validator.

[migrate-mongo](https://www.npmjs.com/package/migrate-mongo) gives MongoDB the same "numbered scripts, applied once, tracked in the database" workflow as SQL migrations:

```bash
npx migrate-mongo create split-patient-name   # creates a timestamped up/down script
npx migrate-mongo up                          # applies every script not yet applied
npx migrate-mongo status
```

```js
// migrations/20261105-split-patient-name.js
module.exports = {
  async up(db) {
    await db.collection('patients').updateMany({ schemaVersion: 1 }, [   // pipeline update: compute new fields from existing ones
      { $set: { firstName: { $arrayElemAt: [{ $split: ['$name', ' '] }, 0] }, schemaVersion: 2 } }
    ]);
  },
  async down(db) {
    await db.collection('patients').updateMany({ schemaVersion: 2 }, { $set: { schemaVersion: 1 }, $unset: { firstName: '' } });
  },
};
```

Index and validator (`collMod`) changes go in migrations too. For millions of documents, update in batches instead of one `updateMany`. See [Large Data and Bulk Operations](/Database-Standards/Large-Data-and-Bulk-Operations).

* * *

## Running MongoDB

| | Atlas (managed) | Self-hosted |
| --- | --- | --- |
| Default for us | **Yes**, unless the client hosts it | When the client requires on-premises or their own servers |
| Backups | Cloud backups with point-in-time restore, turned on per cluster | Your job: `mongodump` for small databases, filesystem snapshots or Percona Backup for MongoDB for real ones |
| High availability | A replica set out of the box | Run at least a **three-member replica set**, never a standalone in production |
| Upgrades and patches | Handled by Atlas | Your job, on a schedule |

Backup strategy and restore drills: [Backups and Recovery](/Database-Standards/Backups-and-Recovery). Replica sets, failover and what to monitor: [Replication and Monitoring](/Database-Standards/Replication-and-Monitoring).

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Try a Postgres `jsonb` column first | Pick MongoDB to avoid designing a schema | [Postgres JSONB or MongoDB](#postgres-jsonb-or-mongodb) |
| Shape documents around the app's reads | Copy a normalized SQL design into collections | [Modeling](#design-for-how-the-data-is-read) |
| Embed small, bounded, read-together data | Embed lists that grow forever | [Embed or reference](#embed-or-reference) |
| Cap arrays with `$push` + `$each` + `$slice` | Let an array grow towards the 16 MB limit | [Unbounded arrays](#avoid-unbounded-arrays) |
| Add a `$jsonSchema` validator to every collection | Rely on the ODM alone to keep data valid | [Validation](#validate-with-a-json-schema) |
| Order compound index fields Equality, Sort, Range | Put range fields before sort fields | [ESR](#compound-indexes-follow-the-esr-rule) |
| Check `explain("executionStats")` for new queries | Ship a query that does a `COLLSCAN` | [Explain](#read-the-explain-output) |
| Paginate from the last seen `_id` | Use large `skip()` values | [Pagination](#paginate-by-range-not-skip) |
| Use `$inc` / `$set` with a condition in the filter | Read, change in code, write back | [Atomic updates](#prefer-single-document-atomic-updates) |
| Keep transactions small and use `withTransaction` | Use transactions everywhere, or on a standalone server | [Transactions](#transactions-when-you-need-them) |
| Validate input types, enable `sanitizeFilter` | Pass `req.body` straight into a filter | [Operator injection](#operator-injection) |
| Enable auth, bind to private IPs, use TLS | Expose port 27017 or allow `0.0.0.0/0` on Atlas | [Server security](#lock-down-the-server) |
| Version documents and migrate with committed scripts | Change shapes in code and hope old documents don't show up | [Schema changes](#schema-changes-and-migrations) |

* * *

## Further reading

1.  **MongoDB Manual — Data Modeling**
    [https://www.mongodb.com/docs/manual/data-modeling/](https://www.mongodb.com/docs/manual/data-modeling/)
2.  **MongoDB Manual — Schema Validation**
    [https://www.mongodb.com/docs/manual/core/schema-validation/](https://www.mongodb.com/docs/manual/core/schema-validation/)
3.  **MongoDB Manual — The ESR (Equality, Sort, Range) Guideline**
    [https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/)
4.  **MongoDB Manual — Production Considerations for Transactions**
    [https://www.mongodb.com/docs/manual/core/transactions-production-consideration/](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/)
5.  **MongoDB Manual — Security Checklist**
    [https://www.mongodb.com/docs/manual/administration/security-checklist/](https://www.mongodb.com/docs/manual/administration/security-checklist/)
6.  **MongoDB Blog — Building with Patterns: A Summary**
    [https://www.mongodb.com/blog/post/building-with-patterns-a-summary](https://www.mongodb.com/blog/post/building-with-patterns-a-summary)
7.  **MongoDB Software Lifecycle Schedules**
    [https://www.mongodb.com/legal/support-policy/lifecycles](https://www.mongodb.com/legal/support-policy/lifecycles)

* * *

## Related pages

- [Database Standards → Which database?](/Database-Standards#which-database) — where MongoDB fits next to Postgres and Redis.
- [PostgreSQL](/Database-Standards/PostgreSQL) — the default, including `jsonb` for flexible data.
- [Redis](/Database-Standards/Redis) — the other NoSQL store we use, for caching and queues.
- [Security → Secure Coding](/Security/Secure-Coding) — injection from the security angle.
