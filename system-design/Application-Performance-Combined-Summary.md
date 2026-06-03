# Application performance — combined summary

**Sources:** AI notes from the same video (generated) + your handwritten notes (`Application Performance.md`).  
**Goal:** One short reference for **why** systems slow down and **what** to change first.

---

## 1. The situation (context)

- Traffic grows → a **monolith** can feel slow before you “need microservices.”
- Quick wins: more CPU/RAM, better code paths, **caching** — but **only after** you see *where* time and queues grow.
- Scaling servers without fixing bottlenecks often **moves** the queue, not **removes** it.

---

## 2. Deep core idea: performance is mostly about **queues**

Under load, work waits in lines: HTTP accept queue, thread pool, DB connection pool, disk scheduler, lock waits, GC pauses.

**Three root causes of queue buildup** (same idea, two wordings merged):

| Cause | Plain meaning | Tiny example |
|--------|----------------|---------------|
| **Slow work** | Each job takes too long | N+1 queries, heavy JSON, cold algorithm |
| **Serial access** | Only one consumer at a time | One row hot-updated by everyone; sync file access |
| **Small pipe** | Not enough CPU/RAM/IO/net | 100 Mbps link, tiny DB pool, saturated CPU |

**Analogy:** A single cashier (serial), slow scanning (inefficient), or too few lanes (limited capacity) — all create a **line**; user sees **latency**.

---

## 3. What “good” means — metrics (watch all four)

| Metric | What it is | Note |
|--------|------------|------|
| **Latency** | Time for one request (often ms) | Business sets targets; 100–200 ms “snappy” is common talk; **average lies** |
| **Tail latency** | Slow end of the distribution (p95/p99) | **Queues show up here first**; GC spikes, lock wait, one slow dependency |
| **Throughput** | Jobs per second | App can be “fast” per request but still choke on total load |
| **Errors + saturation** | Fail %, CPU/RAM/disk/net use | Errors often mean timeouts → hidden queues |

**Compare:** Two systems both **average** 50 ms; one has **p99** 200 ms, the other 2 s. Users hit the tail — optimize **p99**, not only the mean.

**Throughput vs DB:** App layer can accept more than the DB can commit; then you **queue** (in app or broker) by design — still a capacity/ordering problem.

---

## 4. Three levers (from the video) — how they map to metrics

| Lever | Aim | Typical moves |
|--------|-----|----------------|
| **Efficiency** | Shorter time **per** request | Better queries, less data moved, cache hits, fewer round trips |
| **Concurrency** | More requests **in parallel** | Threads/async, DB pool size, horizontal replicas — needs **safe** concurrency |
| **Capacity** | Bigger pipes | CPU/RAM, SSD/IOPS, bandwidth, pool sizes |

They **trade off**: more concurrency can raise CPU and **tail latency** (contention, GC); more caching can raise memory. Tune as a **system**, not one knob.

---

## 5. Layer checklist — where latency hides

### 5.1 Network (app level)

- **TCP/TLS:** New connection per call = extra RTT + handshakes → **reuse** (pools to DB, keep-alive / persistent connections client → gateway).
- **SSL session caching:** fewer full TLS setups on repeat clients.
- **Data:** fewer HTTP calls, smaller payloads, **compression** where it wins; **binary** (e.g. gRPC) vs fat JSON when it fits.
- **Edge:** **CDN** for static assets; cache session-ish data where safe.
- **DNS:** slow DNS = invisible latency — monitor and fix TTL/resolvers.

**vs serial misuse:** Chatty micro-calls over the network = **many** round trips = **queue of waits**.

### 5.2 Memory and GC

- Big heap → longer **GC** pauses → **tail latency**.
- **Bloat:** hold less in RAM; weak/soft refs only where the runtime supports it and you understand lifecycle.
- **Split processes:** sometimes several smaller JVMs/processes beat one giant heap (team/runtime dependent).
- **Batch work:** fewer trips to DB = better locality and less churn.
- **DB side:** finite buffers — huge reads can pressure DB memory + disks.

### 5.3 Disk and logging

- Logs on **sync** disk path = **queue behind disk**.
- Prefer **async logging**, batch/sequential IO, SSD + enough **IOPS**, RAID where relevant.
- **Separate** static (CDN/cache) vs dynamic app paths.
- **Page cache / zero-copy:** less copying from disk → socket on hot paths.
- **DB:** indexes, query shape, schema for read/write pattern; avoid unnecessary columns (your notes + generated align here).

### 5.4 CPU

- Bad algorithms and hot loops.
- **Context switching** and **locks:** too many threads or coarse locks → CPU spent on scheduling/wait, not business work.
- **Thread pool size:** too small → queue; too large → contention and GC.
- **Batch/async IO** off the critical thread.

### 5.5 “Know the hardware”

Rough memory hierarchy idea (orders of magnitude): **L1 ≪ RAM ≪ SSD ≪ HDD**. Design so hot data stays “up” the pyramid (cache in RAM, CDN at edge, sequential disk).

---

## 6. Caching stack (your L1 / L2 / L3 framing)

| Level | Role | Example |
|-------|------|---------|
| **L1** | In-process / in-memory | Local cache for reference data |
| **L2** | Shared across app instances | Redis/Memcached |
| **L3** | Close to user | CDN for static |

**Rule:** Cache **read-heavy**, **tolerably stale** data; invalidate or TTL with eyes open on **correctness**.

---

## 7. Concurrency and performance (short bridge)

Your longer note file starts with **correctness** patterns (locks, OCC, idempotency, outbox). For **performance**:

- **Pessimistic locks** and **high isolation** often **lower throughput** and add **wait** → watch lock time and deadlocks.
- **Hot keys:** many writers on one row = **serial** bottleneck → partition ownership, queue per key, or atomic SQL updates when possible.
- **Retries without idempotency** → duplicate work and **worse** queues; **backoff + jitter + cap** as in your framework.

Use the full concurrency doc when the problem is **wrong data**; use this summary when the problem is **time in queue**.

---

## 8. Practical workflow (timeline)

1. **Observe:** metrics + logs + user complaints; chart **p95/p99**, errors, saturation.
2. **Find the queue:** which pool, which dependency, which lock, which disk?
3. **Classify cause:** slow work vs serial vs small pipe.
4. **Fix:** efficiency first (cheapest wins), then concurrency model, then capacity.
5. **Verify:** load test; confirm **tail** improved, not only average.

---

## 9. Anti-patterns (quick)

- Scale hardware only while **one** query still does 1000 round trips.
- Trust **average** latency only.
- Unlimited threads / unbounded queues (memory + tail blowups).
- Dual-write DB + broker without **outbox** (correctness + retry storms).
- Global **serializable** for everything “just to be safe.”

---

## 10. One-liner you can say in an interview

“I treat slowness as **queues**: inefficient work, **serialized** resources, or **under-sized** pipes. I measure **tail latency** and saturation, fix the biggest queue with **efficiency → safe concurrency → capacity**, and use **caching and connection reuse** to cut repeat cost.”

---

## 11. Follow-ups (from your scratch notes)

- **SRE:** owns reliability, performance, and capacity practice in many orgs — same metrics, stricter SLOs.
- **Deeper “concurrent request processing”** and **latency cost breakdowns:** expand in a separate note when you revisit that part of the video.
- **Hotstar-style throughput management:** optional deep dive webinar — good for streaming-scale patterns.

---

## Glossary (minimal)

| Term | Short meaning |
|------|----------------|
| **Latency** | Time to complete one request |
| **Tail latency** | High percentiles (p99); worst user experience |
| **Throughput** | Requests completed per second |
| **Queue buildup** | Work waiting because something is slow, serial, or full |
| **Connection pool** | Reuse DB/API connections instead of reconnecting every time |
| **CDN** | Cache static content near users |

---

*End of combined summary.*
