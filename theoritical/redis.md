# Redis Advanced Interview Questions & Answers

---

## Most Frequently Asked Redis Interview Questions

> Quick-reference section covering the questions that come up most often in job interviews.

### Q1. What is Redis? Why is it so fast?
Redis is an **in-memory key-value store**. It's fast because:
- All data lives in RAM (no disk I/O on reads/writes).
- Single-threaded event loop eliminates locking overhead.
- Simple data structures with O(1) or O(log N) operations.
- Efficient serialization protocol (RESP).

---

### Q2. What are the main use cases of Redis?
- **Caching** — database query results, API responses, computed values.
- **Session store** — user sessions with TTL.
- **Rate limiting** — counter per user/IP per time window.
- **Leaderboards** — sorted sets for real-time rankings.
- **Pub/Sub messaging** — real-time notifications, chat.
- **Distributed locks** — coordination across services.
- **Job queues** — lists as FIFO queues; streams for durable queues.
- **Real-time analytics** — HyperLogLog for unique counts, bitmaps for flags.

---

### Q3. What is the difference between Redis and Memcached?

| Feature | Redis | Memcached |
|---|---|---|
| Data structures | Rich (strings, hashes, lists, sets, sorted sets…) | Strings only |
| Persistence | Yes (RDB, AOF) | No |
| Replication | Yes | No |
| Pub/Sub | Yes | No |
| Transactions | Yes (MULTI/EXEC) | No |
| Lua scripting | Yes | No |
| Clustering | Yes (native) | Client-side sharding |
| Multi-threading | I/O threads (v6+) | Multi-threaded |

**Rule of thumb:** Use Memcached for simple, high-throughput string caching with multiple CPU cores; use Redis for everything else.

---

### Q4. What happens when Redis runs out of memory?
Depends on `maxmemory-policy`:
- `noeviction` — returns error on write commands (default if maxmemory not set).
- `allkeys-lru` — evicts least recently used key from all keys.
- `volatile-lru` — evicts LRU key only among keys with a TTL.
- `allkeys-lfu` — evicts least frequently used (Redis 4.0+).

Best practice for a cache: use `allkeys-lru` or `allkeys-lfu`.

---

### Q5. What is the difference between SET with NX/XX and SETNX/SETEX?
- `SETNX key value` — set only if key does **not** exist (deprecated).
- `SETEX key seconds value` — set with TTL in seconds (deprecated).
- `SET key value NX PX 30000` — atomic set-if-not-exists with millisecond TTL (preferred).

The modern `SET` command with options (`NX`, `XX`, `EX`, `PX`, `EXAT`, `PXAT`, `KEEPTTL`, `GET`) replaces all older variants atomically in a single command.

---

### Q6. How do you cache an object in Redis and handle cache invalidation?
```python
# Cache-aside pattern
key = f"user:{user_id}"
cached = redis.get(key)
if cached:
    return deserialize(cached)

user = db.find_user(user_id)
redis.setex(key, 3600, serialize(user))  # TTL = 1 hour
return user

# Invalidation on update
def update_user(user_id, data):
    db.update_user(user_id, data)
    redis.delete(f"user:{user_id}")  # evict stale cache
```

Strategies:
- **TTL-based expiry** — simplest, accepts eventual consistency.
- **Event-driven invalidation** — delete/update cache on write.
- **Cache versioning** — embed a version in the key (`user:42:v3`).

---

### Q7. Explain TTL and how to set/check it.
```
SET session:abc "data" EX 1800      # expires in 1800 seconds
TTL session:abc                      # returns remaining seconds (-1 = no TTL, -2 = missing)
PTTL session:abc                     # remaining milliseconds
PERSIST session:abc                  # remove TTL (make permanent)
EXPIRE session:abc 3600              # reset TTL
```

---

### Q8. What is the difference between DEL and UNLINK?
- `DEL key` — synchronously deletes the key; blocks Redis while freeing memory (slow for large keys).
- `UNLINK key` — marks the key as deleted instantly (O(1)) and frees memory **asynchronously** in a background thread.

Always prefer `UNLINK` for large keys (big hashes, long lists, large sorted sets) to avoid blocking.

---

### Q9. What is SCAN and why should you use it instead of KEYS?
- `KEYS pattern` — returns all matching keys in one blocking call. **Never use in production** on large datasets; it blocks all other clients.
- `SCAN cursor MATCH pattern COUNT 100` — iterates incrementally using a cursor, returning a small batch per call. Non-blocking, safe for production.

```python
cursor = 0
while True:
    cursor, keys = redis.scan(cursor, match="session:*", count=100)
    for k in keys:
        process(k)
    if cursor == 0:
        break
```

---

### Q10. How does Redis replication work at a high level?
1. Replica connects to master and sends `PSYNC`.
2. Master forks, generates an RDB snapshot, and sends it to the replica.
3. Replica loads the RDB, then master streams buffered and new write commands.
4. Replication is **asynchronous** — master doesn't wait for replica acknowledgement by default.
5. Use `WAIT numreplicas timeout` to get a synchronous durability guarantee.

---

### Q11. What is the difference between Redis Sentinel and Redis Cluster?
- **Sentinel** — HA for a **single master** with replicas. Monitors, detects failure, and promotes a replica. No data sharding.
- **Cluster** — horizontal scaling across **multiple masters** using hash slot sharding (16,384 slots). Each master has replicas for HA. Use when data doesn't fit on one node.

---

### Q12. How would you implement a simple distributed lock in Redis?
```python
import uuid, redis

r = redis.Redis()
lock_key = "lock:order:42"
lock_val = str(uuid.uuid4())

# Acquire: set only if not exists, with expiry
acquired = r.set(lock_key, lock_val, nx=True, px=10000)

# Release: delete only if we own it (Lua for atomicity)
release_script = """
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
"""
r.eval(release_script, 1, lock_key, lock_val)
```
Key points: unique value prevents another client from releasing your lock; `PX` ensures the lock auto-expires if the holder crashes.

---

### Q13. What is a Redis pipeline and what problem does it solve?
Without pipelining, each command waits for a round-trip (RTT). With a pipeline, multiple commands are sent in one TCP write and responses are read in one batch — eliminates RTT per command.

```python
pipe = redis.pipeline(transaction=False)
for i in range(1000):
    pipe.incr(f"counter:{i}")
pipe.execute()  # single round-trip
```
Not atomic (unlike MULTI/EXEC). Use for bulk independent operations.

---

### Q14. How do you handle session management in Redis?
```
SET session:{token} {serialized_user_data} EX 1800
GET session:{token}
DEL session:{token}   # logout
EXPIRE session:{token} 1800  # sliding expiry on activity
```
- Use a random, cryptographically secure token as the key.
- Store minimal data (user ID, roles) — not sensitive secrets.
- Refresh TTL on each request for sliding session expiry.
- On logout, `DEL` the key immediately.

---

### Q15. When would you NOT use Redis?
- **Primary durable data store** — Redis is not a replacement for a relational DB when you need strong ACID guarantees.
- **Large datasets exceeding RAM** — everything must fit in memory (unless using Redis on Flash / enterprise tiers).
- **Complex relational queries** — no joins, no SQL aggregations.
- **Compliance/audit logs** — while AOF provides durability, it's not designed as an immutable audit trail.
- **Long-running analytical workloads** — blocking operations impact latency-sensitive workloads.

---

## 1. What is Redis and how does it differ from a traditional RDBMS?

Redis (Remote Dictionary Server) is an in-memory, key-value data store that supports rich data structures (strings, hashes, lists, sets, sorted sets, bitmaps, hyperloglogs, streams, geospatial indexes). Unlike an RDBMS:

| Aspect | Redis | RDBMS |
|---|---|---|
| Storage | Primarily in-memory (optional persistence) | Disk-based |
| Data model | Key-value with complex structures | Relational tables |
| Query language | Commands (GET, SET, ZADD…) | SQL |
| Latency | Sub-millisecond | Milliseconds to seconds |
| Durability | Configurable (RDB/AOF) | ACID by default |
| Use cases | Cache, pub/sub, leaderboards, sessions | Transactional, relational data |

---

## 2. Explain Redis persistence options: RDB vs AOF

### RDB (Redis Database Snapshot)
- Saves point-in-time snapshots of data to disk at configured intervals (`SAVE 900 1` = after 900s if ≥1 key changed).
- **Pros:** Compact file, fast restart, minimal I/O impact.
- **Cons:** Data loss between snapshots; blocking `BGSAVE` forks a child process (can be expensive on large datasets).

### AOF (Append-Only File)
- Logs every write command. On restart, replays the log.
- Configurable `fsync` policy:
  - `always` — fsync after every write (safest, slowest)
  - `everysec` — fsync every second (good balance, ≤1s data loss)
  - `no` — OS decides (fastest, most data loss risk)
- **Pros:** Much lower data loss window; more durable.
- **Cons:** Larger file size; slower restart on huge AOF (mitigated by AOF rewrite).

### Hybrid (Redis 4.0+)
`aof-use-rdb-preamble yes` — AOF starts with an RDB snapshot then appends incremental changes. Best of both worlds.

---

## 3. How does Redis replication work?

Redis uses **asynchronous master-replica replication**:

1. Replica connects and sends `PSYNC <replicationID> <offset>`.
2. If master can do a **partial resync** (replica offset still in replication backlog), it sends only the delta.
3. Otherwise, master forks, generates an RDB snapshot (`BGSAVE`), and streams it to the replica, followed by buffered write commands (full resync).
4. After initial sync, master streams every write command to replicas in real time.

Key points:
- Replication is non-blocking on the master.
- `min-replicas-to-write` and `min-replicas-max-lag` can enforce write guarantees.
- Replicas are read-only by default (`replica-read-only yes`).

---

## 4. What is Redis Sentinel and what problem does it solve?

Redis Sentinel provides **high availability** for a master-replica setup:
- Monitors master and replicas.
- Performs **automatic failover**: promotes a replica to master when the master fails.
- Acts as a **configuration provider** — clients query Sentinel to discover the current master address.

Failover process:
1. A Sentinel marks the master `SDOWN` (subjectively down) when it fails health checks.
2. If a quorum of Sentinels agrees, the master becomes `ODOWN` (objectively down).
3. A leader Sentinel is elected (via Raft-like algorithm), selects the best replica (lowest replication lag, highest priority), and promotes it.
4. Other replicas and Sentinels are reconfigured to follow the new master.

Minimum recommended: **3 Sentinel processes** to form a quorum of 2.

---

## 5. Explain Redis Cluster and how it handles data sharding

Redis Cluster partitions data across multiple nodes using **hash slots** (16,384 total).

- Each key is assigned a slot: `CRC16(key) % 16384`
- Each master node owns a range of hash slots.
- Cluster requires at least **3 master nodes** (with 3 replicas for HA = 6 nodes minimum).

### Hash Tags
`{user}.name` and `{user}.age` share the same slot because only `user` is hashed — allows multi-key operations on related keys.

### Resharding
- Slots are migrated between nodes live using `CLUSTER SETSLOT` + `MIGRATE`.
- Clients receive `MOVED` (redirect permanently) or `ASK` (redirect for this request only) responses during migration.

### Cluster vs Sentinel

| Feature | Sentinel | Cluster |
|---|---|---|
| Sharding | No (single shard) | Yes (16384 slots) |
| HA | Yes | Yes (replica per master) |
| Multi-key ops | Full support | Only within same slot |
| Complexity | Lower | Higher |

---

## 6. What are the eviction policies in Redis?

Configured via `maxmemory-policy`:

| Policy | Behavior |
|---|---|
| `noeviction` | Return error when memory limit reached |
| `allkeys-lru` | Evict least recently used keys from all keys |
| `volatile-lru` | LRU eviction among keys with TTL set |
| `allkeys-lfu` | Evict least frequently used (Redis 4.0+) |
| `volatile-lfu` | LFU among keys with TTL |
| `allkeys-random` | Random eviction from all keys |
| `volatile-random` | Random among keys with TTL |
| `volatile-ttl` | Evict key with shortest TTL first |

Redis uses an **approximated LRU/LFU** — it samples `maxmemory-samples` random keys and evicts the best candidate, avoiding a full LRU linked list.

---

## 7. How does Redis handle transactions with MULTI/EXEC?

```
MULTI
SET key1 val1
INCR counter
EXEC
```

- Commands between `MULTI` and `EXEC` are **queued**, not executed immediately.
- `EXEC` runs all queued commands atomically (no other client's commands interleave).
- `DISCARD` aborts the transaction.

**Limitations:**
- No rollback on runtime errors (e.g., calling `INCR` on a string) — other commands still execute.
- Syntax errors during queuing cause the whole transaction to be discarded.
- Not true ACID — no isolation between reads inside the transaction.

### WATCH (Optimistic Locking)
```
WATCH balance
val = GET balance
MULTI
SET balance (val - 100)
EXEC  -- returns nil if balance was modified since WATCH
```
`WATCH` implements **optimistic locking**: if any watched key changes before `EXEC`, the transaction is aborted (returns `nil`).

---

## 8. What is a Redis Pipeline and when should you use it?

A pipeline batches multiple commands into a single network round-trip:

```python
pipe = redis.pipeline()
pipe.set('k1', 'v1')
pipe.set('k2', 'v2')
pipe.get('k1')
pipe.execute()  # sends all at once
```

- Reduces **round-trip time (RTT)** overhead.
- Does **not** guarantee atomicity (unlike MULTI/EXEC).
- Use when sending many independent commands — bulk inserts, batch reads.
- Benchmark shows 10x+ throughput improvement over individual commands on high-latency connections.

---

## 9. Explain Redis Pub/Sub vs Streams

### Pub/Sub
- Publisher sends messages to a **channel**; all current subscribers receive it.
- **Fire and forget** — no message persistence, no consumer groups.
- Subscribers who connect after publishing miss prior messages.
- Use case: real-time notifications, chat.

```
SUBSCRIBE news
PUBLISH news "Breaking!"
```

### Redis Streams (Redis 5.0+)
- Append-only log of entries with auto-generated IDs (`timestamp-sequence`).
- Messages are **persisted** and can be replayed.
- Supports **consumer groups** — multiple consumers can process the same stream with acknowledgement (`XACK`), pending entries list (PEL), and message replay on failure.

```
XADD orders * product "book" qty 2
XREAD COUNT 10 STREAMS orders 0
XGROUP CREATE orders grp1 $
XREADGROUP GROUP grp1 consumer1 COUNT 5 STREAMS orders >
XACK orders grp1 <id>
```

| Feature | Pub/Sub | Streams |
|---|---|---|
| Persistence | No | Yes |
| Consumer groups | No | Yes |
| Message replay | No | Yes |
| Backpressure | No | Yes (MAXLEN) |

---

## 10. How would you implement a distributed lock with Redis?

### Simple SETNX (not safe for distributed systems)
```
SET lock:resource client_id NX PX 30000
```

### Redlock Algorithm (multi-node, by Antirez)
1. Get current time `T1`.
2. Try to acquire lock on **N/2+1 (majority)** independent Redis nodes with the same key, random value, and TTL.
3. Lock is valid if acquired on majority nodes AND total elapsed time < TTL.
4. To release, send `DEL` only if the value matches (use Lua script for atomicity):

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**Controversies:** Martin Kleppmann argues Redlock is unsafe under clock drift + process pauses; recommends fencing tokens for strong safety. Use Redlock for efficiency (avoid duplicate work), not correctness (where data integrity depends on it).

---

## 11. What is Redis Lua scripting and why is it useful?

Lua scripts run **atomically** on the server — no other command executes while a script runs.

```lua
-- Atomic compare-and-delete
local val = redis.call('GET', KEYS[1])
if val == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end
return 0
```

```
EVAL "script" numkeys key1 key2 arg1 arg2
EVALSHA sha numkeys ...   -- use cached script
SCRIPT LOAD "script"      -- preload, returns SHA
```

Benefits:
- Atomicity without MULTI/EXEC limitations (can use `if` logic).
- Reduces round trips.
- `EVALSHA` avoids re-sending the script each time.

Caution: long-running scripts block Redis; use `redis.breakpoint()` and `redis.debug()` with `redis-cli --ldb` for debugging.

---

## 12. How do you implement a rate limiter in Redis?

### Fixed Window Counter
```lua
local key = "rate:" .. user_id .. ":" .. math.floor(time/60)
local count = redis.call("INCR", key)
redis.call("EXPIRE", key, 60)
if count > limit then return 0 end
return 1
```

### Sliding Window Log
Store timestamps of requests in a sorted set:
```
ZADD rate:{user} now now
ZREMRANGEBYSCORE rate:{user} 0 (now - window)
count = ZCARD rate:{user}
EXPIRE rate:{user} window
```

### Token Bucket / Sliding Window Counter
Use a sorted set with `ZRANGEBYSCORE` + `ZADD` in a Lua script for atomic operation. Libraries like `redis-cell` (Redis module) implement the GCRA algorithm natively via `CL.THROTTLE`.

---

## 13. What data structures does Redis provide and their use cases?

| Structure | Commands | Use Cases |
|---|---|---|
| String | GET/SET/INCR/APPEND | Counters, sessions, simple cache |
| Hash | HGET/HSET/HMGET | User profiles, objects |
| List | LPUSH/RPOP/LRANGE | Queues, activity feeds, stacks |
| Set | SADD/SMEMBERS/SINTER | Unique visitors, tags, friend lists |
| Sorted Set | ZADD/ZRANGE/ZRANGEBYSCORE | Leaderboards, priority queues, rate limiting |
| Bitmap | SETBIT/GETBIT/BITCOUNT | Feature flags, daily active users |
| HyperLogLog | PFADD/PFCOUNT | Approximate unique counts (12KB max memory) |
| Geospatial | GEOADD/GEODIST/GEORADIUS | Location-based search |
| Stream | XADD/XREAD/XGROUP | Event sourcing, message queues |

---

## 14. Explain Redis memory optimization techniques

1. **Use appropriate data structures** — a Hash with `ziplist` encoding stores small objects more efficiently than individual string keys.
2. **ziplist / listpack thresholds** — keep hash/list/zset sizes below `hash-max-ziplist-entries` (128) and `hash-max-ziplist-value` (64 bytes) to use compact encoding.
3. **Key naming** — shorter key names reduce overhead at scale.
4. **TTL** — always set expiry on cache keys to prevent unbounded growth.
5. **Object encoding** — `OBJECT ENCODING key` reveals current encoding; `int`, `embstr`, `ziplist`, `skiplist`, `hashtable`, etc.
6. **Redis modules** — `RedisBloom` for bloom filters (space-efficient membership tests), `RedisTimeSeries` for time-series data.
7. **maxmemory + eviction** — configure `maxmemory` and a suitable eviction policy.
8. **`DEBUG OBJECT key`** — shows serialized length and encoding for tuning.
9. **Lazy freeing** — `lazyfree-lazy-eviction yes` frees memory asynchronously to avoid blocking.

---

## 15. How does Redis handle concurrency? Is it single-threaded?

Redis command execution is **single-threaded** (one thread handles the event loop and command processing), which avoids locking overhead and race conditions within the server.

**Multi-threading in Redis 6.0+:**
- **I/O threads** handle network reads/writes in parallel (configured via `io-threads`).
- Command execution itself remains single-threaded.
- Background threads handle: AOF fsync, lazy deletion, RDB snapshots (fork), cluster bus.

**Implications:**
- Long-running commands (`KEYS *`, large `SORT`, heavy Lua scripts) block all other clients.
- Use `SCAN` instead of `KEYS` for non-blocking key iteration.
- `UNLINK` instead of `DEL` for async key deletion.

---

## 16. What is the difference between EXPIRE, PEXPIRE, EXPIREAT, and PEXPIREAT?

| Command | Unit | Type |
|---|---|---|
| `EXPIRE key seconds` | Seconds | Relative |
| `PEXPIRE key milliseconds` | Milliseconds | Relative |
| `EXPIREAT key unix-timestamp` | Seconds (Unix) | Absolute |
| `PEXPIREAT key unix-timestamp-ms` | Milliseconds (Unix) | Absolute |

- `TTL key` / `PTTL key` — returns remaining time (-1 = no expiry, -2 = key not found).
- `PERSIST key` — removes TTL, making the key permanent.
- Redis 7.0 adds `EXAT`, `PXAT`, `KEEPTTL` options directly in `SET`.

---

## 17. How would you design a leaderboard using Redis?

Use a **Sorted Set** where the score is the user's points:

```
ZADD leaderboard 1500 "alice"
ZADD leaderboard 2300 "bob"
ZADD leaderboard 1800 "charlie"

# Top 10 (highest score first)
ZREVRANGE leaderboard 0 9 WITHSCORES

# Rank of a user (0-indexed)
ZREVRANK leaderboard "alice"

# Score of a user
ZSCORE leaderboard "alice"

# Users in score range
ZRANGEBYSCORE leaderboard 1000 2000 WITHSCORES

# Increment score atomically
ZINCRBY leaderboard 100 "alice"
```

For **time-windowed leaderboards** (daily/weekly), use separate sorted sets per time window with TTLs, or use Streams to replay events into a fresh sorted set.

---

## 18. Explain Redis keyspace notifications

Redis can publish events to Pub/Sub channels when keys are modified/expired:

Enable in config:
```
notify-keyspace-events "KEA"
```

- `K` — Keyspace events (channel: `__keyspace@<db>__:<key>`)
- `E` — Keyevent events (channel: `__keyevent@<db>__:<event>`)
- `g` — Generic (DEL, EXPIRE…)
- `x` — Expired events
- `l` — List events
- `z` — Sorted set events
- `A` — Alias for `g$lzxet`

```
SUBSCRIBE __keyevent@0__:expired
```

Use case: trigger cleanup logic when a session key expires, implement TTL-based workflows.

---

## 19. What are Redis modules and give examples?

Redis modules extend Redis with custom data structures and commands via a C API:

| Module | Purpose |
|---|---|
| **RedisJSON** | Native JSON storage and querying (`JSON.GET`, `JSON.SET`) |
| **RediSearch** | Full-text search, secondary indexing, aggregations |
| **RedisGraph** | Graph database (Cypher queries) |
| **RedisBloom** | Bloom filter, Cuckoo filter, Count-Min Sketch, Top-K |
| **RedisTimeSeries** | Time-series data with downsampling and aggregation |
| **RedisAI** | Run ML models (TensorFlow, PyTorch, ONNX) inside Redis |
| **RedisGears** | Serverless data processing, event-driven pipelines |

Modules are loaded with:
```
MODULE LOAD /path/to/module.so
```
Or in redis.conf: `loadmodule /path/to/module.so`

---

## 20. How do you monitor and debug Redis performance?

### Key commands
```
INFO all              -- comprehensive server stats
INFO memory           -- memory usage breakdown
INFO replication      -- master/replica status
INFO stats            -- commands processed, keyspace hits/misses
MONITOR               -- real-time command stream (dev only, impacts perf)
SLOWLOG GET 10        -- last 10 slow commands (>slowlog-log-slower-than µs)
CLIENT LIST           -- connected clients
DEBUG SLEEP 0         -- test latency
LATENCY HISTORY event -- latency samples for an event
```

### Metrics to watch
- **Memory:** `used_memory`, `mem_fragmentation_ratio` (>1.5 indicates fragmentation)
- **Hit rate:** `keyspace_hits / (keyspace_hits + keyspace_misses)` — target >99%
- **Evictions:** `evicted_keys` — if non-zero, `maxmemory` is too low
- **Connections:** `connected_clients`, `rejected_connections`
- **Replication lag:** `master_repl_offset - slave_repl_offset`

### Tools
- `redis-cli --stat` — rolling stats
- `redis-cli --bigkeys` — find memory-heavy keys
- `redis-cli --hotkeys` — find frequently accessed keys (requires `maxmemory-policy allkeys-lfu`)
- `redis-cli --memkeys` — memory usage per key
- Prometheus + redis_exporter — production monitoring

---

## 21. What is the WAIT command and when would you use it?

```
WAIT numreplicas timeout
```

Blocks the client until `numreplicas` replicas acknowledge receiving all write commands issued before `WAIT`, or until `timeout` milliseconds pass. Returns the number of replicas that acknowledged.

Use case: ensure a critical write has propagated before proceeding (e.g., confirming a payment record is replicated before returning a response). Provides a stronger durability guarantee than fire-and-forget replication.

---

## 22. Explain the difference between SCAN, HSCAN, SSCAN, and ZSCAN

All use **cursor-based iteration** — non-blocking alternative to `KEYS`/`HGETALL`/`SMEMBERS`/`ZRANGE`:

```
SCAN cursor [MATCH pattern] [COUNT count] [TYPE type]
```

- Returns a new cursor (0 = iteration complete) and a batch of keys.
- `COUNT` is a hint, not a guarantee — actual returned count may vary.
- Does **not** guarantee no duplicates if keys are added/removed during iteration.
- `TYPE` filter (Redis 6.0) limits results to a specific data type.

Full iteration pattern:
```python
cursor = 0
while True:
    cursor, keys = redis.scan(cursor, match="user:*", count=100)
    process(keys)
    if cursor == 0:
        break
```

---

## 23. How does Redis handle data serialization?

Redis stores data as **binary-safe strings** (byte arrays up to 512MB). Serialization is the client's responsibility:

- **JSON** — human-readable, widely supported; use RedisJSON module for partial updates.
- **MessagePack** — binary, more compact than JSON (~30% smaller).
- **Protocol Buffers** — schema-enforced, efficient for structured data.
- **Java serialization / Kryo** — JVM ecosystems.
- **CBOR / Avro** — schema-based binary formats.

Best practice: use a compact binary format for high-throughput caches; use JSON/RedisJSON when you need query support or human readability.

---

## 24. What is the Redis Object Encoding and why does it matter?

Redis automatically selects the most memory-efficient internal encoding:

| Type | Encoding | Condition |
|---|---|---|
| String | `int` | 64-bit integer |
| String | `embstr` | ≤44 bytes |
| String | `raw` | >44 bytes |
| List | `listpack` | Small/short elements |
| List | `quicklist` | Large |
| Hash | `listpack` | ≤128 entries, values ≤64 bytes |
| Hash | `hashtable` | Exceeds limits |
| Set | `listpack` | ≤128 integers |
| Set | `intset` | All integers, ≤512 |
| Set | `hashtable` | Otherwise |
| Sorted Set | `listpack` | ≤128 entries, values ≤64 bytes |
| Sorted Set | `skiplist` | Exceeds limits |

Staying within the compact encoding thresholds (`hash-max-listpack-entries`, etc.) can reduce memory by 2–10x.

---

## 25. Describe a cache-aside (lazy loading) vs write-through caching pattern

### Cache-Aside (Lazy Loading)
1. Application checks cache — **cache hit**: return data.
2. **Cache miss**: fetch from DB, write to cache with TTL, return data.

```python
val = redis.get(key)
if val is None:
    val = db.query(key)
    redis.setex(key, ttl, val)
return val
```

- Pros: only cached data that is actually requested; resilient to cache failure.
- Cons: cold start penalty; potential **thundering herd** (many misses hitting DB simultaneously).

### Write-Through
- Write to cache **and** DB on every write.
- Cache is always up to date; no stale reads.
- Cons: write latency; cache filled with data that may never be read.

### Write-Behind (Write-Back)
- Write to cache immediately; asynchronously flush to DB.
- Very low write latency; risk of data loss if cache crashes before flush.

### Thundering Herd Mitigation
- **Mutex/lock** on cache miss — only one request populates the cache.
- **Probabilistic early expiration** — refresh slightly before TTL expires.
- **Stale-while-revalidate** — serve stale data while refreshing in background.

---