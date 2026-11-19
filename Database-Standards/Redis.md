Redis
=====

> **Redis is a NoSQL, in-memory key-value store**: extremely fast, and **not** where your important data lives. It sits **next to** [PostgreSQL](/Database-Standards/PostgreSQL), never instead of it. Read [General Principles](/Database-Standards/General-Principles) first.

* * *

## What it is and when to use it

| | |
| --- | --- |
| **What it is** | A **NoSQL in-memory key-value store**. Every value lives under a key, in RAM, as one of a handful of data structures. Reads and writes take well under a millisecond. Data can be saved to disk, but Redis is built for speed first. |
| **Use it for** | **Caching** slow or frequent reads, **sessions**, **rate limiting**, **job queues** (BullMQ, Laravel queues), short-lived **locks**, **pub/sub** notifications, **leaderboards** and counters, one-time codes with an expiry. |
| **Don't use it for** | The **only copy** of data you can't afford to lose (patients, appointments, payments), anything you need to query by more than its key, relationships and reports, or large values (files, big JSON blobs). Treat everything in Redis as something you could rebuild. |
| **Versions** | **Redis 7.x or 8.x**, or **Valkey**. Some commands on this page need 7.0+ (noted where they appear). |

> 📝 **Redis or Valkey?** In 2024 Redis moved from the open-source BSD license to source-available licenses (RSALv2 / SSPLv1). **Valkey** was forked from Redis 7.2 under the Linux Foundation with a BSD license, and cloud providers such as AWS ElastiCache now offer it. **Redis 8 added AGPLv3** as a third licensing option. As a service our app connects to, either works the same, and the commands and client libraries on this page apply to both. Use whichever the hosting provider offers, and ask the tech lead if a project ships Redis inside something we distribute.

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [Pick the structure](#pick-the-right-data-structure) | **Choose the data structure that matches the job**<br/><sub>A sorted set for a leaderboard, a hash for an object, a stream for a durable queue.</sub> |
| [Key names](#name-keys-consistently) | **`app:entity:id`, with an environment prefix on a shared instance**<br/><sub>Readable in a debugger, and you can find or delete a group of keys safely.</sub> |
| [Expiry](#every-cache-key-has-a-ttl) | **Every cache key gets a TTL**<br/><sub>A key without an expiry lives until someone remembers to delete it, which is never.</sub> |
| [Caching](#cache-aside) | **Cache-aside: read Redis, fall back to the database, write back with a TTL**<br/><sub>The database stays the source of truth. Redis only remembers answers.</sub> |
| [Invalidation](#invalidating-the-cache) | **Delete the key when the data changes, and let the TTL catch what you miss**<br/><sub>A short TTL is the safety net for every invalidation bug.</sub> |
| [Stampedes](#preventing-a-cache-stampede) | **Add jitter to TTLs and let only one request rebuild a hot key**<br/><sub>Otherwise a popular key expiring sends every request to the database at once.</sub> |
| [Rate limits](#rate-limiting) | **`INCR` with an expiry set atomically**<br/><sub>A counter whose expiry never got set blocks that user forever.</sub> |
| [Queues](#job-queues) | **BullMQ, Laravel queues or streams, on an instance with `noeviction`**<br/><sub>An eviction policy that deletes queue keys silently loses jobs.</sub> |
| [Locks](#locks) | **`SET NX PX` with a unique token, released only by its owner**<br/><sub>A Redis lock is for avoiding duplicate work. For correctness, lock in the database.</sub> |
| [Dangerous commands](#never-block-the-server) | **`SCAN`, never `KEYS *`, on a real instance**<br/><sub>Redis runs commands one at a time, so one slow command stalls every client.</sub> |
| [Memory](#memory-and-eviction) | **Set `maxmemory` and the right eviction policy for what the instance is for**<br/><sub>Cache instances evict. Queue and session instances must not.</sub> |
| [Security](#lock-it-down) | **Never reachable from the internet, ACL users, TLS**<br/><sub>An open Redis is one of the most commonly exploited services on the internet.</sub> |

* * *

## Concepts you'll meet on this page

### Key and value

Everything is stored under a **key**, a string you choose like `clinic:appointment:123`. No tables, no queries: you read and write by key.

### TTL

**Time to live** before Redis deletes a key by itself. Set it with `EX` (seconds) or `PX` (milliseconds) when writing, or `EXPIRE` afterwards. `TTL key` shows what's left; `-1` means "never expires".

### Eviction

What Redis does at its memory limit (`maxmemory`): reject new writes, or delete keys to make room. The **eviction policy** decides which, if any.

### Saving to disk

Snapshots or an append-only log let data survive a restart. That reduces data loss but doesn't make Redis a primary database. See [Persistence](#persistence).

* * *

## Using Redis well

### Pick the right data structure

| Structure | What it holds | Use it for | Key commands |
| --- | --- | --- | --- |
| **String** | One value (text, number, serialized JSON) | Cache entries, counters, one-time codes, flags | `SET`, `GET`, `INCR`, `MGET` |
| **Hash** | A small object of field → value | A session, a user's preferences, fields you update one at a time | `HSET`, `HGET`, `HGETALL`, `HINCRBY` |
| **List** | An ordered list, push and pop at either end | Simple "latest N" lists | `LPUSH`, `LRANGE`, `LTRIM` |
| **Set** | Unique, unordered members | "Has this user already done X?", tags | `SADD`, `SISMEMBER`, `SCARD` |
| **Sorted set** | Unique members, each with a score, kept in order | Leaderboards, sliding-window rate limits, scheduled items by time | `ZADD`, `ZRANGE`, `ZREMRANGEBYSCORE`, `ZCARD` |
| **Stream** | An append-only log of entries, with consumer groups and acknowledgements | Durable event processing where messages must not be lost | `XADD`, `XREADGROUP`, `XACK` |
| **Pub/Sub** (not stored) | Messages broadcast to whoever is listening right now | Live notifications, cache-clear signals between app servers | `PUBLISH`, `SUBSCRIBE` |

> ⚠️ **Pub/Sub doesn't store anything.** A subscriber that's disconnected, restarting or slow simply misses the message. If every message must be processed, use a stream or a job queue.

### Name keys consistently

Colon-separated, general to specific: **`app:entity:id[:field]`**.

```
clinic:appointment:123              # one cached appointment
clinic:clinic:42:slots:2026-11-10   # a clinic's free slots for a day
clinic:ratelimit:login:203.0.113.7  # login attempts from one IP
clinic:session:9f1c2e...            # a session
```

- **Shared instance between environments or apps?** Add a prefix (`staging:clinic:appointment:123`), set once in the client (`keyPrefix` in ioredis, `prefix` in Laravel's `config/database.php` redis options, `WithKeyPrefix` in StackExchange.Redis).
- Don't rely on numbered databases (`SELECT 1`) to separate apps. They aren't supported in Redis Cluster, and many managed services only offer database 0.

### Every cache key has a TTL

```bash
# ✅ expires in 5 minutes, whatever happens to the code that wrote it
SET clinic:appointment:123 '{"id":123,"status":"booked"}' EX 300
# ❌ lives forever: stale data, and memory that only grows
SET clinic:appointment:123 '{"id":123,"status":"booked"}'
```

Pick the TTL by "how stale is acceptable?": seconds for availability, minutes for clinic details, hours for reference data that almost never changes.

* * *

## Common use cases

### Cache-aside

```ts
// NestJS with ioredis
async getClinic(id: number): Promise<Clinic> {
  const key = `clinic:clinic:${id}`;
  const cached = await this.redis.get(key);
  if (cached) return JSON.parse(cached);
  const clinic = await this.clinics.findById(id);                 // the database is the source of truth
  const ttl = 600 + Math.floor(Math.random() * 60);               // 10 minutes plus jitter
  await this.redis.set(key, JSON.stringify(clinic), 'EX', ttl);
  return clinic;
}
```

```php
// Laravel: remember() does the same in one call. The TTL is in seconds.
$clinic = Cache::remember("clinic:{$id}", 600, fn () => Clinic::findOrFail($id));
```

```csharp
// .NET: HybridCache (in-memory + Redis) handles the miss, the write-back and stampede protection
var clinic = await hybridCache.GetOrCreateAsync(
    $"clinic:clinic:{id}",
    async ct => await clinicRepository.GetByIdAsync(id, ct),
    new HybridCacheEntryOptions { Expiration = TimeSpan.FromMinutes(10) });
```

> 💡 Cache **what's expensive or frequent**, not everything. A Postgres primary-key lookup already takes about a millisecond; caching it adds a second system to keep correct for almost no gain.

### Invalidating the cache

| Strategy | How | Use when |
| --- | --- | --- |
| **TTL only** | Let the key expire | A little staleness is fine (clinic opening hours) |
| **Delete on write** | `DEL` the key right after the database write commits | Users expect to see their own change immediately |
| **Versioned keys** | Put a version in the key (`clinic:v2:clinic:42`), bump it to drop a whole group at once | Many related keys change together, or the cached shape changes on deploy |

```ts
// ✅ write the database first, then delete the cache. The next read rebuilds it.
await this.clinics.update(id, dto);
await this.redis.del(`clinic:clinic:${id}`);
// ❌ updating the cached copy yourself: two concurrent updates can leave Redis holding the older one
await this.redis.set(`clinic:clinic:${id}`, JSON.stringify({ ...cachedClinic, ...dto }));
```

Keep the TTL even when you delete on write: it fixes the cases your invalidation code misses.

### Preventing a cache stampede

When a popular key expires, every request that misses hits the database at once. Three fixes, from simplest:

1. **Jitter**: add a random few seconds to every TTL (as in the cache-aside example), so keys written together don't expire together.
2. **Single rebuild**: only one request recomputes the value while the rest wait or get the old value. .NET `HybridCache` does this for you. In Laravel, wrap the rebuild in `Cache::lock(...)`.
3. **Stale-while-revalidate**: serve the old value while it's refreshed in the background. Laravel's `Cache::flexible('key', [fresh, stale], fn)` does this in recent Laravel versions.

### Sessions

Use Redis for sessions when the app runs on more than one server, so a user isn't logged out when a request lands on a different one. Laravel: `SESSION_DRIVER=redis`. ASP.NET Core: `AddStackExchangeRedisCache` plus `AddSession`. The session's TTL must match the session lifetime, and the instance must not evict keys (see [Memory and eviction](#memory-and-eviction)).

### Rate limiting

**Fixed window** (simple, good enough for most endpoints): one counter per key per window.

```bash
# login attempts from one IP, 10 per minute
INCR   clinic:ratelimit:login:203.0.113.7
EXPIRE clinic:ratelimit:login:203.0.113.7 60 NX    # NX: only set the expiry if it has none (Redis 7.0+)
```

Send both in one `MULTI`/`EXEC`: a crash between them would leave a counter with no expiry, blocking that IP forever. Without `NX`, every request pushes the window back and a busy client is never let through.

**Sliding window** (smoother, no burst at the window edge): store each request's timestamp in a sorted set.

```bash
MULTI
ZREMRANGEBYSCORE clinic:ratelimit:api:user:77 0 1762340000000     # drop entries older than the window
ZADD             clinic:ratelimit:api:user:77 1762340060000 req-8f3a  # this request (unique member)
ZCARD            clinic:ratelimit:api:user:77                     # requests still in the window
PEXPIRE          clinic:ratelimit:api:user:77 60000
EXEC
```

> 💡 Before hand-writing either, check the framework: `@nestjs/throttler` with a Redis storage, Laravel's `RateLimiter` (which uses the cache store), and ASP.NET Core's built-in rate limiting middleware.

### Job queues

Background jobs (reminder SMS, emails, reports) use the stack's own queue library, backed by Redis:

| Stack | Library | Notes |
| --- | --- | --- |
| NestJS | **BullMQ** via `@nestjs/bullmq` | Retries, delays, scheduled jobs. Uses ioredis. Worker connections need `maxRetriesPerRequest: null`. |
| Laravel | **Queues** with the `redis` driver, run by **Horizon** | Horizon adds a dashboard, metrics and worker balancing. |
| .NET | A queue library with a Redis backend, or **Redis Streams** directly through StackExchange.Redis | Streams give consumer groups and acknowledgements, so a crashed worker's messages are picked up again. |

> ⚠️ **A queue instance must use `maxmemory-policy noeviction`.** BullMQ's own production guide says it can't work correctly otherwise: an evicting policy deletes queue keys and jobs disappear without an error. The BullMQ docs also note that **ElastiCache Serverless** can't be used with BullMQ for this reason.

### Locks

A Redis lock stops **duplicate work**: two servers sending the same reminder batch, two clicks starting the same export.

```bash
# acquire: only if nobody holds it, with a unique token and an expiry
SET lock:clinic:reminders:2026-11-10 "a1f9c0e2-host-3" NX PX 30000
```

```lua
-- release: delete only if it's still OUR lock (the token matches)
if redis.call("GET", KEYS[1]) == ARGV[1] then return redis.call("DEL", KEYS[1]) else return 0 end
```

- **Always set an expiry**, so a crashed holder doesn't keep the lock forever.
- **Never release with a plain `DEL`.** If your work ran past the expiry, someone else now holds the lock, and `DEL` removes theirs.
- Frameworks have this built in: Laravel `Cache::lock('reminders', 30)->get(fn () => ...)`, and libraries such as `redlock` for Node and `RedLock.net` for .NET.

> ⚠️ **A Redis lock is for efficiency, not correctness.** A process can pause (garbage collection, a slow network) past the expiry and carry on believing it holds the lock; even the multi-server Redlock algorithm doesn't fully fix this. When two holders would corrupt data (double-booking a slot, charging twice), lock in the database instead (`SELECT ... FOR UPDATE`, a unique constraint, or Postgres advisory locks). See [Transactions and Concurrency](/Database-Standards/Transactions-and-Concurrency).

### Leaderboards and counters

```bash
ZINCRBY clinic:leaderboard:reviews:2026-11 1 "clinic:42"     # add a point
ZRANGE  clinic:leaderboard:reviews:2026-11 0 9 REV WITHSCORES # top 10 (REV needs Redis 6.2+)
```

A sorted set stays ordered on every write, so "top 10" and "what rank is clinic 42?" (`ZREVRANK`) are instant at any size.

* * *

## Running Redis safely

### Never block the server

A command that touches millions of keys stalls **every** other client until it finishes.

```bash
# ❌ walks every key in one go: production stalls for seconds
KEYS clinic:session:*
# ✅ walks the keyspace in small batches; repeat with the returned cursor until it's 0
SCAN 0 MATCH clinic:session:* COUNT 1000
```

- **`FLUSHALL`, `FLUSHDB`, `KEYS` and `DEBUG`**: block them for the app's user with ACLs.
- **Big keys** (a hash with millions of fields, a 50 MB string) make every read slow, and deleting one blocks too. `UNLINK` instead of `DEL` frees it in the background. Find them with `redis-cli --bigkeys` or `--memkeys`.
- **Hot keys**: one key read by every request concentrates load on one server in a cluster. `redis-cli --hotkeys` finds them (it needs an LFU eviction policy).
- **Batch round trips**: `MGET` or a pipeline instead of 50 separate `GET`s, the same [think in sets](/Database-Standards/General-Principles#think-in-sets-not-loops) rule as SQL.

### Memory and eviction

Always set `maxmemory`, and pick the policy by what the instance is for:

| Instance used for | `maxmemory-policy` | Why |
| --- | --- | --- |
| **Cache only** | `allkeys-lru` or `allkeys-lfu` | When memory is full, drop the least recently / least frequently used keys. Losing a cache entry is fine. |
| **Queues, sessions, locks, rate limits** | `noeviction` (the default) | Never delete data silently. Writes fail with an out-of-memory error instead, which you'll see and fix. |
| **Mixed** | Don't. Use **separate instances** (or at least separate databases on a managed service that supports it). | One policy can't be right for both. |

> 💡 Monitor memory use and the `evicted_keys` count (from `INFO`). Evictions on a cache instance are normal. Any on a `noeviction` instance means something is wrong.

### Persistence

| Mode | What it does | Data you can lose on a crash |
| --- | --- | --- |
| **None** | Memory only | Everything. Fine for a pure cache. |
| **RDB** | Snapshots to disk at intervals | Everything since the last snapshot (often minutes) |
| **AOF** | Logs every write; `appendfsync everysec` is the usual setting | About one second of writes |
| **RDB + AOF** | Both | About one second, with faster restarts |

Queues and sessions need **AOF** on, so a restart doesn't drop every job and log everyone out. Managed services expose this as a setting or a "durable" tier (for example **Amazon MemoryDB**). Backup strategy: [Backups and Recovery](/Database-Standards/Backups-and-Recovery).

### Lock it down

- **Never reachable from the internet.** Bind to private interfaces and firewall port 6379 to the app servers. Leave `protected-mode` on.
- **ACL users** (Redis 6.0+): one user per app with only the commands and key patterns it needs, and dangerous commands blocked. Don't share the `default` user.
- **TLS** between the app and Redis whenever traffic leaves the server. On managed services, turn the setting on.
- Credentials come from config or a secrets manager, never the repo. See [Access and Roles](/Database-Standards/Access-and-Roles).

```bash
# an app user: its own keys only, no admin or dangerous commands
ACL SETUSER clinic-app on >S3cret-from-vault ~clinic:* &clinic:* +@all -@admin -@dangerous
```

* * *

## Clients in our stacks

| Stack | Client | Notes |
| --- | --- | --- |
| NestJS | **ioredis** | The usual choice, and what BullMQ uses. The official `redis` (node-redis) client is fine too. Don't mix both in one app. |
| NestJS caching | **`@nestjs/cache-manager`** with a Keyv Redis store (`@keyv/redis`) | Current versions use Keyv stores, so older guides using `cache-manager-redis-store` no longer apply. TTLs here are in **milliseconds**. |
| Laravel | **phpredis** (PHP extension) or **predis** (pure PHP package) | phpredis is faster and Laravel's default (`REDIS_CLIENT=phpredis`). Use the `Cache` facade for caching and `Redis` for raw commands. |
| .NET | **StackExchange.Redis** | Create **one `ConnectionMultiplexer` for the whole app** (a singleton) and reuse it. One per request exhausts connections. |
| .NET caching | **`HybridCache`** (`Microsoft.Extensions.Caching.Hybrid`, GA since 2025) or `IDistributedCache` | HybridCache combines an in-memory cache with Redis and prevents stampedes. `IDistributedCache` via `AddStackExchangeRedisCache` is the simpler, older option. |

```csharp
// ✅ registered once, shared everywhere
builder.Services.AddSingleton<IConnectionMultiplexer>(
    ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")!));
// ❌ a new connection for every request
using var redis = ConnectionMultiplexer.Connect(connectionString);
```

### Managed Redis

| Provider | Service | Notes |
| --- | --- | --- |
| AWS | **ElastiCache** (Valkey or Redis OSS), **MemoryDB** | MemoryDB is the durable option. Check BullMQ compatibility before choosing ElastiCache Serverless. |
| Azure | **Azure Managed Redis** | Azure Cache for Redis is being retired: the Enterprise tiers by March 2027, Basic/Standard/Premium on 30 September 2028, and new caches can no longer be created. New projects use Azure Managed Redis. |
| Google Cloud | **Memorystore** (Valkey or Redis) | |
| Any cloud | **Redis Cloud**, **Upstash** | Upstash is serverless with per-request pricing and an HTTP API, handy for low-traffic and serverless apps. |

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Keep the real data in the database, rebuildable data in Redis | Store the only copy of patients, bookings or payments in Redis | [When to use it](#what-it-is-and-when-to-use-it) |
| Pick the structure that fits: sorted set, hash, stream | Store everything as JSON strings | [Data structures](#pick-the-right-data-structure) |
| Name keys `app:entity:id`, prefix shared instances | Use ad hoc key names, or numbered databases to separate apps | [Key names](#name-keys-consistently) |
| Give every cache key a TTL, with jitter | Write cache keys that never expire | [Expiry](#every-cache-key-has-a-ttl) |
| Delete the key after the database write commits | Patch the cached copy by hand | [Invalidation](#invalidating-the-cache) |
| Let one request rebuild a hot key | Let every miss hit the database at once | [Stampedes](#preventing-a-cache-stampede) |
| `INCR` + `EXPIRE ... NX` in one pipeline | `INCR` with no expiry, or an `EXPIRE` that resets every request | [Rate limits](#rate-limiting) |
| Run queues on a `noeviction` instance | Put queues on an `allkeys-lru` cache instance | [Queues](#job-queues) |
| `SET NX PX` with a token, release by Lua compare | Release a lock with a plain `DEL`, or trust it for correctness | [Locks](#locks) |
| `SCAN`, `UNLINK`, `MGET`, pipelines | `KEYS *`, deleting big keys with `DEL`, 50 separate `GET`s | [Blocking](#never-block-the-server) |
| Set `maxmemory` and the right policy per instance | Mix cache and queue data on one instance | [Memory](#memory-and-eviction) |
| AOF on for queues and sessions | Assume Redis keeps data across a restart | [Persistence](#persistence) |
| Private network, ACL users, TLS | Expose port 6379 or use the `default` user from the app | [Security](#lock-it-down) |
| One shared connection or multiplexer per app | Open a connection per request | [Clients](#clients-in-our-stacks) |

* * *

## Further reading

1.  **Redis Docs — Key eviction**
    [https://redis.io/docs/latest/develop/reference/eviction/](https://redis.io/docs/latest/develop/reference/eviction/)
2.  **Redis Docs — Persistence**
    [https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
3.  **Redis Docs — Security**
    [https://redis.io/docs/latest/operate/oss_and_stack/management/security/](https://redis.io/docs/latest/operate/oss_and_stack/management/security/)
4.  **Redis Docs — Distributed Locks with Redis**
    [https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
5.  **Martin Kleppmann — How to do distributed locking** (why a Redis lock isn't enough for correctness)
    [https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
6.  **BullMQ — Going to production**
    [https://docs.bullmq.io/guide/going-to-production](https://docs.bullmq.io/guide/going-to-production)
7.  **Microsoft Learn — HybridCache library in ASP.NET Core**
    [https://learn.microsoft.com/aspnet/core/performance/caching/hybrid](https://learn.microsoft.com/aspnet/core/performance/caching/hybrid)
8.  **Valkey** — the Linux Foundation fork of Redis
    [https://valkey.io/](https://valkey.io/)

* * *

## Related pages

- [Database Standards → Which database?](/Database-Standards#which-database) — where Redis fits next to Postgres and MongoDB.
- [MongoDB](/Database-Standards/MongoDB) — the other NoSQL store we use.
- [Tech Stack → Backend Stack](/Tech-Stack/Backend-Stack) — how each stack is set up.
- [Security](/Security) — network protection and secrets for every service, Redis included.
