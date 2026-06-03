# Node.js Senior Engineer Interview Prep

> Authored from the perspective of a Senior Staff Engineer & experienced technical interviewer.
> Format per question: **Asked → Weak answer trap → Strong senior answer → Follow-ups → What the interviewer is really testing.**

---

## PART 1 — Core JavaScript

---

### Q1. Walk me through exactly what happens when JavaScript executes this code:

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');
```

**Weak answer (junior trap):** "1, 2, 3, 4 because setTimeout runs later."

**Strong senior answer:**
Output: `1 → 4 → 3 → 2`

Execution trace:
1. `console.log('1')` — synchronous, runs immediately on call stack.
2. `setTimeout(fn, 0)` — Web API schedules callback in the **macrotask queue** (task queue).
3. `Promise.resolve().then(fn)` — `.then` callback queued in **microtask queue** (PromiseJobs).
4. `console.log('4')` — synchronous, runs immediately.
5. Call stack is now empty. Event loop checks **microtask queue first** (always drained completely before next macrotask). Runs `console.log('3')`.
6. Microtask queue empty. Event loop picks next macrotask: `console.log('2')`.

**Key distinction:**
- Microtask queue: Promise callbacks, `queueMicrotask()`, `MutationObserver` — drained **entirely** after each task.
- Macrotask queue: `setTimeout`, `setInterval`, `setImmediate`, I/O callbacks — one per event loop iteration.

**Follow-up 1:** What if there are 1000 chained `.then()` calls — does that block the event loop?
> Yes — microtasks are drained completely before the next macrotask. A pathological chain of microtasks can starve I/O callbacks. This is a real production concern.

**Follow-up 2:** Where does `queueMicrotask()` fit vs `process.nextTick()`?
> `process.nextTick()` runs **before** the microtask queue — it has its own "nextTick queue" that is drained before Promise microtasks. Priority: nextTick → microtasks → macrotasks.

**What the interviewer tests:** Whether you understand the event loop is not just "async vs sync" but a precise queue-ordering system.

---

### Q2. Explain closures. Give a real production use case — not a counter example.

**Weak answer:** "A closure is a function that remembers its outer scope."

**Strong senior answer:**
A closure is a function that **retains a live reference** to the variables in its lexical scope, even after the outer function has returned. The inner function closes over the variable bindings, not the values.

**Production use cases:**

**1. Module pattern / encapsulation (pre-ES modules):**
```javascript
const createRateLimiter = (limit, windowMs) => {
    const requests = new Map();  // closed over — private state

    return (userId) => {
        const now = Date.now();
        const userRequests = requests.get(userId) || [];
        const valid = userRequests.filter(t => now - t < windowMs);
        if (valid.length >= limit) return false;
        requests.set(userId, [...valid, now]);
        return true;
    };
};

const limiter = createRateLimiter(100, 60000);
// `requests` is not accessible outside — true encapsulation
```

**2. Memoization / caching:**
```javascript
const memoize = (fn) => {
    const cache = new Map();
    return (...args) => {
        const key = JSON.stringify(args);
        if (cache.has(key)) return cache.get(key);
        const result = fn(...args);
        cache.set(key, result);
        return result;
    };
};
```

**Classic closure trap:**
```javascript
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 0);  // prints 3, 3, 3
}
// Fix: use let (block scope) or IIFE
for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 0);  // prints 0, 1, 2
}
```

**Follow-up:** How can closures cause memory leaks in Node.js?
> If a closure references a large object (e.g., a request body, DB result set) and the closure is stored in a long-lived structure (event emitter callback, global cache), the GC cannot collect the object. The closure keeps the reference alive indefinitely.

---

### Q3. What is the difference between `==` and `===`? When would you actually use `==`?

**Strong senior answer:**
`===` is strict equality — no type coercion. `==` uses abstract equality — applies type coercion rules defined in the spec.

In practice: **never use `==` in production code** except:
- `value == null` is the idiomatic way to check for `null || undefined` simultaneously (safe because `null == undefined` is `true` but `null == 0` is `false`).

```javascript
// Checks for both null and undefined
if (user.role == null) { /* handle missing role */ }
// Equivalent to: user.role === null || user.role === undefined
```

**Common coercion traps:**
```javascript
0 == false     // true
'' == false    // true
null == 0      // false (!)
NaN == NaN     // false (only value not equal to itself — use Number.isNaN())
[] == false    // true
[] == ![]      // true (both coerce to 0)
```

---

### Q4. Explain `this` — what is it in different contexts and how do arrow functions change it?

**Strong senior answer:**
`this` is determined at **call time** (not definition time) for regular functions, and at **definition time** for arrow functions.

| Context | `this` value |
|---|---|
| Global scope (non-strict) | `globalThis` / `window` |
| Global scope (strict mode) | `undefined` |
| Method call: `obj.fn()` | `obj` |
| Regular function call: `fn()` | `globalThis` (non-strict) / `undefined` (strict) |
| `new Fn()` | new instance |
| Arrow function | Lexical `this` from enclosing scope |
| `call/apply/bind` | Explicitly set |

**Production scenario where this matters:**
```javascript
class PaymentService {
    constructor() {
        this.processor = 'stripe';
    }

    // BUG: 'this' is lost when passed as callback
    process(amount) {
        return fetch('/pay').then(function() {
            console.log(this.processor);  // undefined — 'this' is global/undefined
        });
    }

    // FIX: arrow function
    processFixed(amount) {
        return fetch('/pay').then(() => {
            console.log(this.processor);  // 'stripe' — lexical this
        });
    }
}
```

**Follow-up:** When does `bind` not work?
> `bind` has no effect on arrow functions — their `this` is fixed at definition and cannot be overridden by `bind`, `call`, or `apply`.

---

### Q5. What are Promises? Explain the states and common pitfalls.

**Strong senior answer:**

A Promise is an object representing the eventual completion or failure of an async operation. States:
- **Pending** — initial state.
- **Fulfilled** — operation succeeded, value available.
- **Rejected** — operation failed, reason available.
- Once settled (fulfilled or rejected), **immutable** — cannot change state.

**Critical pitfalls:**

**1. Unhandled rejection (production killer):**
```javascript
// BAD: rejection is silently swallowed
someAsyncFn().then(result => {
    // if this throws, no .catch()
    JSON.parse(result);
});

// GOOD
someAsyncFn()
    .then(result => JSON.parse(result))
    .catch(err => logger.error(err));

// In Node.js, listen globally:
process.on('unhandledRejection', (reason, promise) => {
    logger.error('Unhandled rejection', reason);
    process.exit(1); // recommended in production
});
```

**2. Promise.all vs Promise.allSettled:**
```javascript
// Promise.all — fails fast on first rejection
const [user, account] = await Promise.all([getUser(id), getAccount(id)]);

// Promise.allSettled — waits for all, reports each result
const results = await Promise.allSettled([getUser(id), getAccount(id)]);
results.forEach(r => {
    if (r.status === 'rejected') logger.error(r.reason);
});
```

**3. Promise.race vs Promise.any:**
```javascript
// race — first settled (fulfilled OR rejected)
const result = await Promise.race([fetchPrimary(), timeout(5000)]);

// any (ES2021) — first fulfilled, ignores rejections unless ALL reject
const result = await Promise.any([fetchReplica1(), fetchReplica2()]);
```

**Follow-up:** What is Promise chaining and how does error propagation work?
> Returning a rejected promise or throwing inside `.then()` propagates to the next `.catch()` in the chain, skipping intermediate `.then()` handlers. This is the Promise equivalent of try-catch-finally.

---

### Q6. Explain Async/Await — what does it compile to? What are common mistakes?

**Strong senior answer:**

`async/await` is syntactic sugar over Promises. An `async` function always returns a Promise. `await` pauses execution of the `async` function (not the event loop) until the Promise settles.

**Common senior-level mistakes:**

**1. Sequential awaits when parallel is possible:**
```javascript
// SLOW: ~2000ms (sequential)
const user = await getUser(id);      // 1000ms
const orders = await getOrders(id);  // 1000ms

// FAST: ~1000ms (parallel)
const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);
```

**2. await inside forEach (does NOT work):**
```javascript
// BUG: forEach doesn't await async callbacks
const ids = [1, 2, 3];
ids.forEach(async (id) => {
    await processOrder(id);  // fires but not awaited
});
// All 3 run concurrently without any control

// FIX: for...of (sequential) or Promise.all (parallel)
for (const id of ids) {
    await processOrder(id);  // sequential, awaited
}
```

**3. Error handling — async functions swallow errors:**
```javascript
// Each async IIFE must handle its own errors
async function handler(req, res) {
    try {
        const result = await processPayment(req.body);
        res.json(result);
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
}
```

**4. Async in constructors:**
> Constructors cannot be async. Use a static factory method: `static async create()` that calls `new` and then awaits initialization.

---

### Q7. What is hoisting? Explain with `var`, `let`, `const`, and function declarations.

**Strong senior answer:**

Hoisting is the JS engine's behavior of processing declarations during the compilation phase before execution.

| Declaration | Hoisted? | Initialized? | TDZ? |
|---|---|---|---|
| `var` | Yes | `undefined` | No |
| `let` | Yes | No | Yes |
| `const` | Yes | No | Yes |
| `function` declaration | Yes | Full definition | No |
| `function` expression (`var fn = function`) | var hoisted, function is not | `undefined` | No |

**TDZ (Temporal Dead Zone):** `let`/`const` are hoisted but not initialized. Accessing them before declaration throws `ReferenceError`.

```javascript
console.log(x);  // undefined (var hoisting)
var x = 5;

console.log(y);  // ReferenceError: Cannot access 'y' before initialization
let y = 5;

fn();  // works — function declaration fully hoisted
function fn() { return 1; }

fn2();  // TypeError: fn2 is not a function
var fn2 = function() { return 1; }
```

---

### Q8. Explain Prototypes and the Prototype Chain. How does `class` relate?

**Strong senior answer:**

Every JavaScript object has an internal `[[Prototype]]` link to another object (or `null`). When you access a property, JS walks the chain until it finds it or reaches `null`.

```javascript
const animal = { breathe() { return 'breathing'; } };
const dog = Object.create(animal);
dog.bark = function() { return 'woof'; };

dog.bark();     // found on dog
dog.breathe();  // not on dog → walks to animal → found

dog.__proto__ === animal;                    // true
Object.getPrototypeOf(dog) === animal;      // true (preferred over __proto__)
```

**ES6 `class` is syntactic sugar:**
```javascript
class Animal {
    constructor(name) { this.name = name; }
    speak() { return `${this.name} speaks`; }
}
class Dog extends Animal {
    bark() { return 'woof'; }
}
// Dog.prototype.__proto__ === Animal.prototype
```

**Production relevance:** Understanding prototypes matters for:
- Performance: methods on prototype are shared; methods in constructor create a new function per instance.
- Mixin patterns: `Object.assign(Target.prototype, mixin)`.
- Detecting own properties: `obj.hasOwnProperty('key')` vs `'key' in obj` (chain).

---

### Q9. Explain memory leaks in Node.js. How do you detect and fix them?

**Strong senior answer:**

**Common sources:**

1. **Global variables accumulating data:**
```javascript
const cache = {};  // global, never cleared
app.post('/data', (req, res) => {
    cache[req.body.id] = req.body;  // grows forever
});
```

2. **Event emitter listeners not removed:**
```javascript
// Each request adds a listener, never removed
emitter.on('data', handler);
// Fix:
emitter.once('data', handler);  // auto-removes after firing
// or
emitter.removeListener('data', handler);
```

3. **Closures holding references:**
```javascript
function leak() {
    const largeBuffer = Buffer.alloc(10 * 1024 * 1024); // 10MB
    return () => largeBuffer.length;  // closure keeps buffer alive
}
const fns = [];
setInterval(() => fns.push(leak()), 1000);  // memory grows indefinitely
```

4. **Timers and intervals not cleared:**
```javascript
const interval = setInterval(() => {}, 1000);
// If interval is never cleared, its callback closure stays in memory
clearInterval(interval);  // always clean up
```

5. **Circular references (less of an issue with modern GC, but streams/custom objects).**

**Detection:**
```bash
node --inspect app.js
# Open Chrome → chrome://inspect → Memory tab → Take heap snapshots
# Compare before/after load → look for growing retainer counts

# CLI tools
node --expose-gc app.js  # expose gc() for manual GC in tests
clinic doctor -- node app.js   # 0x, clinic flame, clinic heap
```

**Fix pattern:** Use `WeakMap`/`WeakSet` for caches where keys are objects — entries are GC'd when the key is no longer referenced.

---

## PART 2 — Node.js Internals

---

### Q10. Explain the Node.js Event Loop phases in order.

**Strong senior answer:**

Node.js event loop (libuv) has 6 phases, executed in order:

```
   ┌────────────────────────┐
   │        timers          │  setTimeout, setInterval callbacks
   │  ────────────────────  │
   │     pending callbacks  │  I/O callbacks deferred from prev iteration
   │  ────────────────────  │
   │       idle, prepare    │  internal use
   │  ────────────────────  │
   │          poll          │  retrieve new I/O events; execute I/O callbacks
   │  ────────────────────  │
   │          check         │  setImmediate callbacks
   │  ────────────────────  │
   │     close callbacks    │  socket.on('close', ...)
   └────────────────────────┘
```

**Between each phase:** `process.nextTick` queue and microtask (Promise) queue are fully drained.

**Priority order:** `process.nextTick` > Promise microtasks > macrotasks (phase callbacks)

**Practical scenario:**
```javascript
setImmediate(() => console.log('setImmediate'));
setTimeout(() => console.log('setTimeout'), 0);
// Output order is non-deterministic when both scheduled from main module
// BUT inside an I/O callback, setImmediate ALWAYS runs before setTimeout
fs.readFile('file', () => {
    setImmediate(() => console.log('immediate'));  // always first
    setTimeout(() => console.log('timeout'), 0);  // always second
});
```

**Follow-up:** What is the poll phase and what makes Node.js block?
> The poll phase blocks waiting for I/O if the callback queue is empty and no timers are pending. If a timer has expired, it proceeds. This is where Node.js "waits" efficiently — no spinning.

---

### Q11. What is libuv and what does it do?

**Strong senior answer:**

libuv is a C library that provides Node.js with:
1. **Event loop** — the core async execution mechanism.
2. **Thread pool** (default 4 threads, configurable via `UV_THREADPOOL_SIZE`) — handles operations that don't have async OS syscalls: file system I/O, DNS resolution (`dns.lookup`), crypto (`bcrypt`), zlib.
3. **Async I/O** — wraps OS-level async APIs: `epoll` (Linux), `kqueue` (macOS), `IOCP` (Windows) for network I/O, pipes, and some file operations.

**What goes to thread pool vs OS async:**

| Thread Pool (blocking, libuv manages) | OS Async (truly non-blocking) |
|---|---|
| `fs.*` file operations | Network TCP/UDP sockets |
| `dns.lookup()` | `dns.resolve()` |
| `crypto.pbkdf2`, `crypto.randomBytes` | Pipes, signals |
| `zlib` compression | |

**Critical for seniors:** Saturating the thread pool (4 concurrent bcrypt calls) blocks ALL subsequent thread pool operations (file reads etc.). Fix: `UV_THREADPOOL_SIZE=16` or use a dedicated worker thread.

---

### Q12. When would you use Worker Threads vs Cluster mode vs Child Processes?

**Strong senior answer:**

| | Worker Threads | Cluster | Child Process |
|---|---|---|---|
| Use case | CPU-bound tasks in same process | Scale across CPU cores for HTTP | Isolated processes, external programs |
| Memory sharing | Yes (SharedArrayBuffer) | No | No |
| Communication | postMessage, SharedArrayBuffer | IPC via `cluster.worker.send` | IPC, stdin/stdout/stderr |
| Overhead | Low (same process) | Medium (fork) | High (fork + exec) |
| Crash isolation | No (crashes main process) | Yes | Yes |

**Worker Threads — CPU-bound in Node.js:**
```javascript
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
    const worker = new Worker(__filename, { workerData: { n: 40 } });
    worker.on('message', result => console.log(result));
} else {
    const result = fibonacci(workerData.n);  // runs on separate thread
    parentPort.postMessage(result);
}
```

**Cluster — scale HTTP server across all CPU cores:**
```javascript
const cluster = require('cluster');
const os = require('os');

if (cluster.isPrimary) {
    os.cpus().forEach(() => cluster.fork());
    cluster.on('exit', (worker) => cluster.fork());  // auto-restart
} else {
    require('./app').listen(3000);
}
```

**Follow-up:** What is the difference between Cluster and PM2 cluster mode?
> PM2 cluster mode is a managed abstraction over Node.js cluster. PM2 adds: process monitoring, log aggregation, zero-downtime reload (`pm2 reload`), environment management, and startup script generation. Under the hood it uses `cluster.fork()`.

---

### Q13. What are Node.js Streams? Explain backpressure and how to handle it.

**Strong senior answer:**

Streams are abstract interfaces for working with sequential data. Types:
- **Readable** — source of data (fs.createReadStream, http.IncomingMessage)
- **Writable** — destination (fs.createWriteStream, http.ServerResponse)
- **Duplex** — both (TCP socket)
- **Transform** — duplex that transforms data (zlib.createGzip)

**Why streams matter:** Process large files without loading into memory.
```javascript
// Without streams — loads entire file into memory → OOM on large files
const data = fs.readFileSync('huge.csv');
res.send(data);

// With streams — memory-efficient, starts sending immediately
fs.createReadStream('huge.csv').pipe(res);
```

**Backpressure** — the mechanism that prevents a fast readable from overwhelming a slow writable:
```javascript
// Manual backpressure handling
readable.on('data', (chunk) => {
    const canContinue = writable.write(chunk);
    if (!canContinue) {
        readable.pause();                        // stop reading
        writable.once('drain', () => {
            readable.resume();                   // resume when writable is ready
        });
    }
});

// pipe() handles backpressure automatically
readable.pipe(writable);

// pipeline() — preferred (handles cleanup on error)
const { pipeline } = require('stream/promises');
await pipeline(
    fs.createReadStream('input.csv'),
    csvTransform,
    fs.createWriteStream('output.json')
);
```

**Production scenario:** Streaming a 5GB file upload to S3 via AWS SDK:
```javascript
app.post('/upload', (req, res) => {
    const upload = s3.upload({ Bucket, Key, Body: req });  // req is a Readable stream
    upload.promise().then(() => res.json({ ok: true }));
});
```

---

### Q14. How would you handle CPU-bound work in Node.js without blocking the event loop?

**Strong senior answer:**

Options ranked by fit:

1. **Worker Threads** — same process, SharedArrayBuffer for data sharing, low overhead. Best for: in-process CPU work (image processing, PDF generation, encryption).

2. **Child Process (`child_process.fork`)** — separate Node.js process. Best for: isolated work with full Node.js environment, crash isolation.

3. **Offload to a queue (Kafka/Bull/BullMQ)** — for long-running jobs. Node.js API enqueues work; separate worker processes consume. Best for: async job processing (report generation, email sending, data processing).

4. **C++ Addon / N-API** — for ultra-performance native extensions. Rarely needed.

5. **External service** — Python/Go service for ML inference, FFmpeg for video.

**Production pattern (BullMQ):**
```javascript
// API layer — non-blocking
app.post('/reports', async (req, res) => {
    const job = await reportQueue.add('generate', { userId: req.user.id });
    res.json({ jobId: job.id });
});

// Separate worker process
const worker = new Worker('reportQueue', async (job) => {
    return generateReport(job.data.userId);  // CPU-heavy, separate process
});
```

---

## PART 3 — Backend Engineering

---

### Q15. Design a JWT authentication system. What are its weaknesses and how do you mitigate them?

**Strong senior answer:**

**Basic flow:**
```
Login → Server issues JWT (access token, short TTL) + refresh token (long TTL, httpOnly cookie)
Request → Client sends JWT in Authorization: Bearer header
Server → Verifies signature, extracts claims, no DB lookup needed
Refresh → Client uses refresh token to get new access token
```

**Weaknesses and mitigations:**

| Weakness | Mitigation |
|---|---|
| Can't invalidate before expiry | Maintain a token blocklist in Redis; check on each request |
| Stolen access token | Short TTL (5–15 min); use HTTPS only |
| Stolen refresh token | Rotate refresh tokens on use; detect reuse (refresh token family) |
| Large payload → bigger headers | Keep claims minimal (userId, roles only); avoid PII in token |
| Algorithm confusion (RS256 vs HS256) | Always verify `alg` in header; explicitly specify algorithm in verify() |

**Refresh token rotation (security):**
```javascript
// On refresh: issue new refresh token, invalidate old one
async function refreshTokens(oldRefreshToken) {
    const payload = verifyRefreshToken(oldRefreshToken);

    // Detect reuse attack: if token already used, revoke entire family
    const tokenRecord = await db.refreshTokens.findOne({ token: oldRefreshToken });
    if (!tokenRecord || tokenRecord.used) {
        await db.refreshTokens.deleteMany({ family: payload.family });
        throw new Error('Refresh token reuse detected');
    }

    await db.refreshTokens.update({ token: oldRefreshToken }, { used: true });
    const newRefreshToken = issueRefreshToken({ ...payload, family: payload.family });
    return { accessToken: issueAccessToken(payload), refreshToken: newRefreshToken };
}
```

**Follow-up:** Where do you store tokens on the client?
> Refresh tokens: `httpOnly; Secure; SameSite=Strict` cookie — not accessible via JS, protected against XSS. Access tokens: in-memory (JS variable) — shorter-lived, not persisted. Storing in `localStorage` is acceptable for SPAs where XSS risk is managed, but `httpOnly` cookie is safer.

---

### Q16. How do you prevent duplicate payments in a distributed system?

**This is a key FinTech question given your background.**

**Strong senior answer:**

**Idempotency keys:**
```javascript
// Client sends a unique idempotency key per payment attempt
POST /payments
Headers: Idempotency-Key: uuid-v4-generated-by-client
Body: { amount: 100, currency: 'BDT', to: 'acc_123' }

// Server implementation
async function processPayment(req) {
    const key = req.headers['idempotency-key'];
    if (!key) throw new BadRequestError('Idempotency-Key required');

    // Check if we've seen this key before
    const cached = await redis.get(`idem:${key}`);
    if (cached) return JSON.parse(cached);  // return same response

    // Acquire distributed lock to prevent concurrent duplicate processing
    const lock = await redlock.acquire([`lock:payment:${key}`], 30000);
    try {
        const result = await executePayment(req.body);
        // Store result for 24 hours (or whatever window makes sense)
        await redis.setex(`idem:${key}`, 86400, JSON.stringify(result));
        return result;
    } finally {
        await lock.release();
    }
}
```

**Database-level deduplication (belt and suspenders):**
```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE,  -- DB-level uniqueness guarantee
    status VARCHAR(50),
    ...
);
```

**Outbox pattern** — prevent double-publishing to Kafka after DB write:
```
1. In same DB transaction: INSERT payment + INSERT outbox_event
2. Outbox worker: read unpublished events → publish to Kafka → mark published
3. Exactly-once delivery: even if step 2 crashes, re-run is safe (idempotent)
```

---

### Q17. Walk me through how you'd implement rate limiting in a Node.js service at scale.

**Strong senior answer:**

**Layer 1 — Nginx (network level):** Reject clearly abusive traffic before it hits Node. (`limit_req_zone` per IP).

**Layer 2 — Application level (Redis sliding window):**
```javascript
const rateLimit = async (userId, limit = 100, windowSec = 60) => {
    const key = `rl:${userId}:${Math.floor(Date.now() / 1000 / windowSec)}`;
    const count = await redis.incr(key);
    if (count === 1) await redis.expire(key, windowSec);
    return count <= limit;
};

// Middleware
app.use(async (req, res, next) => {
    const allowed = await rateLimit(req.user.id);
    if (!allowed) return res.status(429).json({
        error: 'Rate limit exceeded',
        retryAfter: 60
    });
    next();
});
```

**Token bucket (smoother, handles bursts):**
```javascript
// Using Redis + Lua for atomic token bucket
const tokenBucketScript = `
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local tokens = tonumber(redis.call('hget', key, 'tokens') or capacity)
local last = tonumber(redis.call('hget', key, 'last') or now)
local elapsed = now - last
local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)
if new_tokens >= 1 then
    redis.call('hmset', key, 'tokens', new_tokens - 1, 'last', now)
    return 1
end
return 0
`;
```

**Headers to include in response:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1700000060
Retry-After: 30   (only on 429)
```

---

### Q18. How do you handle errors in a production Node.js Express application?

**Strong senior answer:**

**Operational errors vs programmer errors:**
- **Operational:** expected failures (DB timeout, validation error, 404) — handle gracefully.
- **Programmer:** unexpected bugs (TypeError, ReferenceError) — crash fast, let process manager restart.

```javascript
// Custom error class hierarchy
class AppError extends Error {
    constructor(message, statusCode, code) {
        super(message);
        this.statusCode = statusCode;
        this.code = code;
        this.isOperational = true;
        Error.captureStackTrace(this, this.constructor);
    }
}

class NotFoundError extends AppError {
    constructor(resource) {
        super(`${resource} not found`, 404, 'NOT_FOUND');
    }
}

// Async wrapper (avoids try-catch in every route)
const asyncHandler = (fn) => (req, res, next) =>
    Promise.resolve(fn(req, res, next)).catch(next);

// Global Express error handler (must have 4 params)
app.use((err, req, res, next) => {
    const { statusCode = 500, message, code, isOperational } = err;

    logger.error({ err, requestId: req.id }, 'Request error');

    if (!isOperational) {
        // Programmer error — alert on-call, then crash
        process.exit(1);
    }

    res.status(statusCode).json({
        error: { code, message }
    });
});

// Unhandled errors
process.on('unhandledRejection', (reason) => {
    logger.error(reason, 'Unhandled rejection');
    process.exit(1);
});
process.on('uncaughtException', (err) => {
    logger.error(err, 'Uncaught exception');
    process.exit(1);
});
```

---

## PART 4 — Database (PostgreSQL)

---

### Q19. Explain PostgreSQL transaction isolation levels and when you'd use each.

**Strong senior answer:**

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Use Case |
|---|---|---|---|---|
| READ UNCOMMITTED | Possible | Possible | Possible | Not used in PG (same as READ COMMITTED) |
| READ COMMITTED (default) | No | Possible | Possible | Most reads, simple writes |
| REPEATABLE READ | No | No | Possible (PG: No) | Reports, balance calculations |
| SERIALIZABLE | No | No | No | Financial transactions, inventory |

**Production example — race condition in wallet debit:**
```sql
-- Without proper isolation: two concurrent requests both read balance=100,
-- both subtract 80, both succeed → balance goes to -60

-- FIX 1: SELECT FOR UPDATE (pessimistic locking)
BEGIN;
SELECT balance FROM wallets WHERE id = $1 FOR UPDATE;  -- row-level lock
UPDATE wallets SET balance = balance - 80 WHERE id = $1 AND balance >= 80;
COMMIT;

-- FIX 2: Optimistic locking with version column
UPDATE wallets
SET balance = balance - 80, version = version + 1
WHERE id = $1 AND balance >= 80 AND version = $2;
-- Check rows affected: if 0, retry

-- FIX 3: SERIALIZABLE isolation
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- PG will abort conflicting transactions (be prepared to retry)
```

**Follow-up:** What is MVCC and how does PostgreSQL implement it?
> MVCC (Multi-Version Concurrency Control) — readers never block writers, writers never block readers. PG stores multiple versions of each row (tuples) with `xmin`/`xmax` transaction IDs. Each transaction sees a snapshot of the DB at its start time. `VACUUM` cleans up dead tuples.

---

### Q20. How do you diagnose and fix N+1 queries in a Node.js app?

**Strong senior answer:**

N+1: 1 query to fetch N items, then N queries to fetch related data = N+1 total queries.

```javascript
// N+1 in ORM (Sequelize/TypeORM)
const orders = await Order.findAll();                    // 1 query
for (const order of orders) {
    order.user = await User.findByPk(order.userId);      // N queries
}

// FIX 1: Eager loading (JOIN)
const orders = await Order.findAll({ include: User });

// FIX 2: Dataloader (batching) — especially in GraphQL
const userLoader = new DataLoader(async (ids) => {
    const users = await User.findAll({ where: { id: ids } });
    return ids.map(id => users.find(u => u.id === id));
});
// Multiple calls in same tick → batched into one query
const user = await userLoader.load(order.userId);

// FIX 3: Raw SQL with JOIN
SELECT o.*, u.name FROM orders o JOIN users u ON u.id = o.user_id;
```

**Detection:**
- `pg` query logging: log all queries, look for repeated patterns.
- APM tools (Datadog, New Relic) — trace DB calls per request.
- `EXPLAIN ANALYZE` — look for sequential scans on related tables.

---

### Q21. Explain database indexing — types, when to use, and pitfalls.

**Strong senior answer:**

**B-Tree index (default):** Ordered tree. Good for equality, range queries, ORDER BY.
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Composite index (column order matters!)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- Useful for: WHERE user_id = ? AND status = ?
-- NOT useful for: WHERE status = ? (only leading column used)
```

**Partial index:** Only indexes rows matching a condition — smaller, faster:
```sql
CREATE INDEX idx_pending_orders ON orders(created_at)
WHERE status = 'pending';
-- Only 5% of rows? 95% smaller index
```

**Index on expression:**
```sql
CREATE INDEX idx_lower_email ON users(LOWER(email));
-- Supports: WHERE LOWER(email) = 'user@example.com'
```

**Pitfalls:**
1. **Too many indexes** — each index slows INSERT/UPDATE/DELETE. Write-heavy tables: be selective.
2. **Low-cardinality columns** (boolean, status enum) — index often not used; partial index or composite with high-cardinality column.
3. **`LIKE '%term%'` ignores B-tree index** — use GIN + `pg_trgm` for text search.
4. **Index bloat** — after heavy deletes, run `REINDEX` or use `pg_repack`.

```sql
-- Always check if index is being used
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 123;
-- Look for: Index Scan vs Seq Scan
-- Look for: Buffers: shared hit (cache) vs read (disk)
```

---

## PART 5 — Distributed Systems

---

### Q22. Explain the Outbox Pattern. Why is it necessary in event-driven systems?

**Strong senior answer:**

**The dual-write problem:**
```javascript
// UNSAFE: Two separate operations — atomicity not guaranteed
await db.insertOrder(order);          // succeeds
await kafka.publish('order.created'); // crashes → event never published
// Result: DB has order, Kafka doesn't → consumers never process it
```

**Outbox pattern:**
```javascript
// SAFE: Single DB transaction writes data + event atomically
await db.transaction(async (trx) => {
    await trx('orders').insert(order);
    await trx('outbox_events').insert({
        id: uuid(),
        aggregate_type: 'Order',
        aggregate_id: order.id,
        event_type: 'order.created',
        payload: JSON.stringify(order),
        published: false,
        created_at: new Date()
    });
});
// Outbox publisher (separate process/CDC):
// 1. Poll/listen for unpublished events
// 2. Publish to Kafka
// 3. Mark as published
```

**Two publisher approaches:**
1. **Polling publisher** — query `WHERE published = false` every second. Simple but adds DB load.
2. **CDC (Change Data Capture)** — Debezium reads PostgreSQL WAL (write-ahead log) and publishes to Kafka. No polling, real-time, zero additional DB load. Industry standard.

**Follow-up:** How do you ensure exactly-once delivery with the outbox pattern?
> The outbox guarantees **at-least-once** delivery (publisher may retry on crash). Consumers must be **idempotent** — check if event was already processed using event ID before acting. Combine: `INSERT ... ON CONFLICT DO NOTHING` in the consumer's processed_events table.

---

### Q23. Explain the SAGA pattern. Choreography vs Orchestration — when to use each.

**Strong senior answer:**

SAGA = sequence of local transactions where each step publishes an event/message, and compensating transactions roll back on failure. Replaces distributed ACID transactions.

**Choreography (event-driven):**
```
OrderService → publishes OrderCreated
  PaymentService listens → processes payment → publishes PaymentCompleted
    InventoryService listens → reserves stock → publishes StockReserved
      ShippingService listens → ships order → publishes OrderShipped

On failure:
  InventoryService fails → publishes StockReservationFailed
    PaymentService listens → refunds payment → publishes PaymentRefunded
      OrderService listens → marks order cancelled
```
**Pros:** Loose coupling, no central coordinator. **Cons:** Hard to track overall saga state; circular event chains become unmaintainable.

**Orchestration (central coordinator):**
```
SagaOrchestrator:
  1. Call PaymentService → success
  2. Call InventoryService → failure
  3. Call PaymentService.compensate() → refund
  4. Mark saga failed
```
**Pros:** Single place to see saga state; explicit flow. **Cons:** Orchestrator is a new service; potential bottleneck.

**Rule of thumb:**
- < 3 services, simple flow → choreography.
- Complex multi-step workflows, need visibility into state → orchestration.
- FinTech payment flows → orchestration (need audit trail of each step).

---

### Q24. How would you handle a Kafka consumer falling behind (consumer lag)?

**Strong senior answer:**

**Diagnosis first:**
```bash
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group payment-processor-group
# Look at: LAG column — difference between latest offset and consumer offset
```

**Root causes and fixes:**

1. **Consumer processing is too slow (CPU/DB bottleneck):**
   - Profile the processing function — is there a slow DB query, external API call?
   - Add indexes, batch DB writes, async non-critical steps.

2. **Not enough consumer instances:**
   - Increase consumers (up to partition count — each consumer owns ≥1 partition).
   - Increase partition count (requires recreation or careful online expansion).

3. **Poison pill message (one message blocks the consumer):**
   ```javascript
   consumer.run({
       eachMessage: async ({ message }) => {
           try {
               await processMessage(message);
           } catch (err) {
               // Move to Dead Letter Queue instead of blocking
               await producer.send({ topic: 'payments.DLQ', messages: [message] });
               logger.error({ message, err }, 'Moved to DLQ');
           }
       }
   });
   ```

4. **Batch processing instead of one-by-one:**
   ```javascript
   consumer.run({
       eachBatch: async ({ batch }) => {
           const messages = batch.messages;
           await db.bulkInsert(messages.map(m => JSON.parse(m.value)));
           // Commit offset after entire batch
       }
   });
   ```

5. **Temporary: reset offset to latest** (skip caught-up processing, used carefully):
   ```bash
   kafka-consumer-groups.sh --reset-offsets --to-latest --execute
   ```

**Monitoring alert:** Set alert on consumer lag > N messages or lag growing for > 5 minutes.

---

### Q25. What is a Circuit Breaker pattern? Implement it in Node.js.

**Strong senior answer:**

Circuit breaker prevents cascading failures. When a downstream service is failing, stop calling it to give it time to recover.

**States:**
- **Closed** — normal operation, requests pass through.
- **Open** — downstream is failing; requests fail immediately (fast fail).
- **Half-Open** — trial period; allow one request through to test recovery.

```javascript
class CircuitBreaker {
    constructor(fn, { failureThreshold = 5, timeout = 60000, successThreshold = 2 } = {}) {
        this.fn = fn;
        this.failureThreshold = failureThreshold;
        this.timeout = timeout;
        this.successThreshold = successThreshold;
        this.state = 'CLOSED';
        this.failures = 0;
        this.successes = 0;
        this.nextAttempt = null;
    }

    async call(...args) {
        if (this.state === 'OPEN') {
            if (Date.now() < this.nextAttempt) {
                throw new Error('Circuit breaker OPEN — service unavailable');
            }
            this.state = 'HALF_OPEN';
        }

        try {
            const result = await this.fn(...args);
            this.onSuccess();
            return result;
        } catch (err) {
            this.onFailure();
            throw err;
        }
    }

    onSuccess() {
        this.failures = 0;
        if (this.state === 'HALF_OPEN') {
            this.successes++;
            if (this.successes >= this.successThreshold) {
                this.state = 'CLOSED';
                this.successes = 0;
            }
        }
    }

    onFailure() {
        this.failures++;
        if (this.failures >= this.failureThreshold) {
            this.state = 'OPEN';
            this.nextAttempt = Date.now() + this.timeout;
        }
    }
}

// Usage
const breaker = new CircuitBreaker(
    (id) => paymentGateway.charge(id),
    { failureThreshold: 5, timeout: 30000 }
);
```

**Production:** Use `opossum` library instead of rolling your own.

---

## PART 6 — Production Scenarios

---

### Q26. A Node.js service's memory is growing over 24 hours and eventually crashes. How do you investigate?

**Strong senior answer — systematic approach:**

**Step 1: Confirm it's a leak vs expected growth.**
```bash
# Monitor memory over time
watch -n 5 'ps -o pid,rss,vsz -p $(pgrep node)'
# Or add to app:
setInterval(() => {
    const { rss, heapUsed, heapTotal } = process.memoryUsage();
    logger.info({ rss, heapUsed, heapTotal }, 'memory');
}, 30000);
```

**Step 2: Narrow to leak category.**
- `heapUsed` growing → JS heap leak (closures, event listeners, global cache).
- `rss` growing but `heapUsed` stable → native/Buffer leak (Buffers are off-heap).
- External memory growing → native addon or native module.

**Step 3: Heap snapshot comparison.**
```javascript
// Enable heap snapshot via SIGUSR2
process.on('SIGUSR2', () => {
    const { writeHeapSnapshot } = require('v8');
    writeHeapSnapshot();
});
// kill -USR2 <pid>  →  heapdump-<timestamp>.heapsnapshot
// Open in Chrome DevTools → Memory → Load snapshot → Compare
```

**Step 4: Look for retainer chains.**
In Chrome Memory tab: "Retainer" shows what's keeping objects alive.

**Common findings:**
- Growing `Array` or `Map` in retainer chain → unbounded cache.
- `EventEmitter` with hundreds of listeners → forgotten `.removeListener()`.
- `setTimeout`/`setInterval` closures holding large objects.

**Step 5: Fix and validate.**
- Add WeakRef/WeakMap for caches.
- Remove event listeners when done.
- Set `maxListeners` on emitters; Node.js warns at 11 by default.
- Use `--max-old-space-size` as a safety ceiling, not a cure.

---

### Q27. API latency suddenly spiked from 50ms to 2000ms. Walk me through your investigation.

**Strong senior answer:**

**Step 1: Determine scope.**
- All endpoints or specific ones? (routing issue vs global)
- All regions or one? (infrastructure issue)
- Gradual or sudden? (deploy, traffic spike, or external dependency)

```bash
# Check recent deployments
git log --oneline -10

# Check current error rate
curl -s 'https://metrics/api?query=error_rate' | jq

# Check p50/p95/p99 latency breakdown
```

**Step 2: Identify bottleneck layer.**
```
Client → Load Balancer → Nginx → Node.js → PostgreSQL/Redis/Kafka/External API

# Check each layer:
# 1. Nginx access log timing
# 2. Node.js APM trace (Datadog/NewRelic) — which function is slow?
# 3. DB: pg_stat_activity — long-running queries?
# 4. Redis: SLOWLOG GET 10
# 5. External APIs: response time from logs
```

**Step 3: Common culprits.**
- **DB query regression** — new code path with missing index, EXPLAIN ANALYZE.
- **Connection pool exhaustion** — all connections in use; requests queue up.
  ```javascript
  // pg pool config
  const pool = new Pool({ max: 20, idleTimeoutMillis: 30000 });
  pool.on('connect', () => logger.debug('New client connected'));
  // Monitor: pool.totalCount, pool.idleCount, pool.waitingCount
  ```
- **Memory pressure → GC pauses** — check `--trace-gc` flag.
- **Event loop lag** — synchronous code blocking event loop.
  ```javascript
  // Monitor event loop lag
  const lag = require('event-loop-lag')(1000);
  setInterval(() => logger.info({ lag: lag() }, 'event loop lag'), 5000);
  ```
- **Redis latency** — `LATENCY HISTORY` + check if `KEYS *` is being called.
- **External service slow** — circuit breaker would have helped; add timeout.

**Step 4: Mitigate while investigating.**
- Route traffic to previous deployment (feature flag or rollback).
- Increase DB connection pool size (if that's the bottleneck).
- Add timeout to slow external calls so they fail fast.

---

### Q28. How do you prevent race conditions in a concurrent Node.js payment system?

**Strong senior answer:**

Race condition example: Two simultaneous debit requests for the same wallet, both read balance=100, both approve, balance goes to -60.

**Prevention strategies:**

**1. Database-level pessimistic locking:**
```sql
BEGIN;
SELECT balance FROM wallets WHERE id = $1 FOR UPDATE;  -- locks row
-- No other transaction can read-for-update or update this row until COMMIT
UPDATE wallets SET balance = balance - $2 WHERE id = $1 AND balance >= $2;
COMMIT;
```

**2. Atomic DB operations (no read-modify-write):**
```sql
UPDATE wallets
SET balance = balance - $2
WHERE id = $1 AND balance >= $2
RETURNING balance;
-- If 0 rows updated → insufficient balance (atomic check-and-update)
```

**3. Distributed lock (Redis Redlock) for cross-service scenarios:**
```javascript
const lock = await redlock.acquire([`wallet:${walletId}`], 5000);
try {
    const wallet = await db.getWallet(walletId);
    if (wallet.balance < amount) throw new InsufficientFundsError();
    await db.debitWallet(walletId, amount);
} finally {
    await lock.release();
}
```

**4. Queue per entity (serialized processing):**
- Route all operations for `wallet_id=X` to the same Kafka partition (using walletId as partition key).
- Single consumer processes operations serially per wallet — no concurrent access.
- Scales horizontally by partition count.

**Rule of thumb for FinTech:** Use DB-level `FOR UPDATE` or atomic updates for single-service scenarios. Use distributed locks or Kafka partitioning for cross-service scenarios.

---

## PART 7 — System Design Questions

---

### Q29. Design a payment service for a FinTech platform (given your background)

**Strong senior answer structure:**

**Requirements clarification:**
- Scale: 20M BDT/day → ~230 transactions/second.
- Consistency: strong (financial data, no eventual consistency for balances).
- Idempotency: required (retries must not double-charge).
- Audit: complete transaction history required.

**Architecture:**
```
Client → API Gateway (Nginx) → Payment Service (Node.js)
                                     ↓
                             Idempotency Check (Redis)
                                     ↓
                          Distributed Lock (Redlock)
                                     ↓
                         PostgreSQL (SERIALIZABLE txn)
                           ┌─────────────────────┐
                           │ UPDATE wallet        │
                           │ INSERT transaction   │
                           │ INSERT outbox_event  │
                           └─────────────────────┘
                                     ↓
                    Outbox Publisher → Kafka → Notification Service
                                            → Analytics Service
                                            → Reconciliation Service
```

**Key design decisions:**

1. **Idempotency keys** — client-generated UUID per payment attempt; stored in Redis (24h TTL).
2. **Optimistic vs pessimistic locking** — use `SELECT FOR UPDATE` on wallet row within a SERIALIZABLE transaction.
3. **Event sourcing** — every state change appended to `payment_events` table (never update, only insert) for complete audit trail.
4. **Reconciliation** — nightly batch job compares payment_events with bank statements; alert on mismatch.
5. **Circuit breaker** — for external payment gateway (bKash, Nagad); fallback to queue.

---

### Q30. Common follow-up pattern: "What would you change if traffic grew 100x?"

**Strong senior answer:**

- **Read replicas** for PostgreSQL — analytical queries, reporting, wallet reads (with acceptable stale read).
- **CQRS** — separate read model (denormalized, Redis or read replica) from write model (PostgreSQL).
- **Kafka partitioning by wallet ID** — serialize operations per wallet, horizontally scalable.
- **Caching wallet balances in Redis** — cache-aside with short TTL; invalidate on write.
- **Database sharding** — shard by user ID or wallet ID when single Postgres can't keep up.
- **Event sourcing + event store** — replace PostgreSQL for transaction log with an optimized event store.

---

## Quick-Reference Summary

### JavaScript gotchas interviewers love
```javascript
typeof null === 'object'      // true — historical bug
NaN === NaN                   // false — use Number.isNaN()
0.1 + 0.2 === 0.3            // false — floating point; use toFixed or integer math
[] + [] === ''               // true — coercion
[] + {} === '[object Object]' // true
{} + [] === 0                // true (parsed as block + unary +)
```

### Node.js event loop cheat sheet
```
Order of execution:
1. Synchronous code (call stack)
2. process.nextTick() callbacks
3. Promise microtasks (.then, .catch)
4. setTimeout / setInterval callbacks
5. I/O callbacks
6. setImmediate callbacks
7. close callbacks
```

### PostgreSQL explain output — what to look for
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
-- Seq Scan → missing index or low selectivity
-- Index Scan → good, using index
-- Hash Join → joining large tables (ok if small hash)
-- Nested Loop → ok for small datasets, bad for large
-- "rows=X (actual rows=Y)" large difference → stale statistics (run ANALYZE)
-- "Buffers: shared hit=X read=Y" — hit=RAM, read=disk; high read = needs caching
```

### Production checklist for Node.js services
- [ ] Global error handlers (`unhandledRejection`, `uncaughtException`)
- [ ] Request IDs on every log line (correlation ID middleware)
- [ ] Health check endpoint (`/health`, `/ready`)
- [ ] Graceful shutdown (drain connections on SIGTERM)
- [ ] Connection pool configured with max, timeout, idle timeout
- [ ] Circuit breakers on all external HTTP/gRPC calls
- [ ] Idempotency keys on all write APIs
- [ ] `process.nextTick` queue and Promise queue monitored for starvation
- [ ] `UV_THREADPOOL_SIZE` increased if using heavy bcrypt/crypto
- [ ] Heap snapshot capability in production (SIGUSR2 handler)
