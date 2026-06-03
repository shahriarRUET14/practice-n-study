# Concurrency Problems and Solution Approaches (Senior Notes)

Concurrency issues appear when multiple threads/processes/services access shared state at the same time.  
In distributed systems, this expands to multiple nodes, retries, network delays, and partial failures.

## 1) Common Concurrency Problems

### 1.1 Race Condition
- **What happens:** Two or more operations interleave in unexpected order.
- **Example:** Two requests update the same account balance; one update is lost.
- **Impact:** Wrong business state, non-deterministic bugs.

### 1.2 Lost Update
- **What happens:** Read-modify-write without coordination; later write overrides earlier write.
- **Example:** Inventory count decremented concurrently, only one decrement persists.

### 1.3 Dirty Read / Non-repeatable Read / Phantom Read
- **What happens:** Transaction isolation is too weak for workload.
- **Example:** Reporting query sees uncommitted or shifting data.

### 1.4 Deadlock
- **What happens:** Two transactions hold locks each other needs.
- **Example:** Txn A locks row X then waits row Y; Txn B locks row Y then waits row X.

### 1.5 Livelock / Starvation
- **What happens:** System keeps retrying/yielding but no useful progress.
- **Example:** High-priority tasks permanently delay low-priority tasks.

### 1.6 Distributed Duplication
- **What happens:** Retries, network timeout, or at-least-once delivery execute same command multiple times.
- **Example:** Payment API retries create double charge.

### 1.7 Out-of-Order Events
- **What happens:** Event B arrives before Event A due to partitioning/retry.
- **Example:** "OrderCancelled" processed before "OrderCreated".

---

## 2) Solution Patterns (When to Use)

### 2.1 Pessimistic Locking
- **Use when:** Strong correctness is mandatory and contention is moderate.
- **How:** `SELECT ... FOR UPDATE`, distributed lock (carefully).
- **Trade-off:** Lower throughput, lock waiting, deadlock risk.

### 2.2 Optimistic Concurrency Control (OCC)
- **Use when:** Contention is low-to-medium, high read ratio.
- **How:** Version column / ETag / compare-and-swap (CAS).
- **Trade-off:** Conflicts cause retries.

### 2.3 Atomic Database Operations
- **Use when:** Problem can be expressed as single atomic statement.
- **How:** `UPDATE ... SET count = count - 1 WHERE id = ? AND count > 0`.
- **Trade-off:** Keeps logic in SQL; best for counters/inventory.

### 2.4 Idempotency Keys
- **Use when:** External calls may retry (payments, order creation).
- **How:** Client sends key; server stores result by key and returns same response on retry.
- **Trade-off:** Need key storage and expiry policy.

### 2.5 Queue + Single Writer / Partitioned Ownership
- **Use when:** High-write contention for same entity.
- **How:** Route same entity key to same partition/consumer.
- **Trade-off:** Added latency; requires partition strategy.

### 2.6 Saga / Compensating Transactions
- **Use when:** Multi-service workflow cannot be one ACID transaction.
- **How:** Orchestrated/choreographed steps with rollback actions.
- **Trade-off:** Eventual consistency; compensation complexity.

### 2.7 Transactional Outbox + Inbox
- **Use when:** Need consistency between DB state and event publishing.
- **How:** Write business row + outbox row in one DB transaction; async publisher emits events.
- **Trade-off:** More moving parts, but removes dual-write inconsistency.

### 2.8 Stronger Isolation (Selective)
- **Use when:** Financial, ledger, and strict integrity flows.
- **How:** Repeatable read / serializable for critical transactions only.
- **Trade-off:** Throughput drop under contention.

---

## 3) Practical Decision Framework

1. **Classify data criticality**
    - Money/ledger: correctness first.
    - Analytics counters: eventual consistency acceptable.
2. **Measure contention hot spots**
    - Identify entities with frequent concurrent updates.
3. **Pick default strategy**
    - Low contention -> OCC + retry.
    - High contention -> partition ownership / queue / single-writer.
4. **Design retry safely**
    - Exponential backoff + jitter.
    - Max retry cap + dead-letter queue.
5. **Guarantee deduplication**
    - Idempotency keys for commands; event consumer dedupe store.
6. **Observe and verify**
    - Track conflict rate, lock wait time, deadlocks, duplicate execution rate.

---

## 4) Anti-Patterns to Avoid

- Blind read-modify-write with no lock or version check.
- Distributed lock as first choice for every problem.
- Infinite retries without jitter/cap.
- Dual-write (DB + message broker) without outbox pattern.
- Global serializable isolation for all queries.

---

## 5) Interview/Design Talk Track (Short)

"I start by classifying consistency requirements per operation, then quantify contention.  
For low contention, I prefer optimistic concurrency with version checks and bounded retries.  
For hot keys, I shift to partitioned ownership or single-writer via queue.  
For distributed workflows, I use idempotency keys, outbox/inbox, and saga compensation.  
Finally, I monitor conflict/lock/duplicate metrics and tune isolation only where business-critical."

## Application Performance - Slow performance
### There is three reason
** Inefficient slow processing **
- Badly written code or library
- External Dependencies
  ** Serial resources acceess **
- Database Access
- Shared Files: Only one file can access a file at a time.
- Serial Execution in code processing synchronous operation.
  ** Limited Resource Capacity **
- CPU, Memory, Network, Disk I/O
- Thread pool exhaustion, connection pool exhaustion, etc.
### Performance Principles
* **Efficiency**: Reduce Response time on a single request.
    * Efficient resource utilizations (io, cpu)
    * Efficient Logic (algorithm, DB query)
    * Efficient Data storage (Data structure, DB schema)
    * **Caching L1 (in-memory), L2 (distributed cache), L3 (CDN)**


* **Concurrency**: Handle multiple requests at the same time.
    * Hardware must have concurrency support
* **Capacity**: Scale to handle more load.
    * CPU, Memory, Network, Disk I/O

### Objectives of Performance Principles
* **Reduce Latency**: Time taken to process a single request.
* **Maximize Throughput**: Number of requests processed per unit time.
* **Database throughput is often lower**: We need a queue to handle more requests.

### Performance Measurement Metrics
* **Latency**: Time taken to process a single request (ms)... 1 ms is best 500 ms considered to be worst there is no actual standard it depends on business
* **Throughput**: Number of requests processed per unit time (req/s).
* **Resource Utilization**: CPU, Memory, Network, Disk I/O usage (%).
* **Error Rate**: Percentage of failed requests (%).

## What is tail latency?
    Tail latency refers to the response times of the slowest request within a certain set.
* Tail Latency is an indication of queuing of requests. Gets worse with higher load.
* Average latency hides the effects of tail latency.
* Tail latency is important for user experience and system reliability.
* Tail latency can be caused by resource contention, garbage collection.


# Performance Optimization Strategies

## Serial Request Latency

## Network Latency - Application
TCP Connections: 
* Connection Pool(DB-> App Server): Reusing existing connections instead creating new connection on every call.
* Persistent Connections(Client -> API Gateway): 
* SSL Session Caching :


Data Transfer:
* Session / Data Caching
* Static Data Caching (CDN)
* Data Format & Compression
* Binary Data - gRPC: 

Additional Considerations:
* Minimise HTTP requests
* Optimize DNS resolution: This is critical
* Monitor and analyze


## Memory Access Latency
* Finite Heap Memory:
* Large Heap Memory:
* GC(Garbage Collection) Algorithms:
* Finite Buffer Memory (on DB)

## Minimize Memory Access Latency
1. Avoid Memory Bloat: Reduce the overall amount of data your application holds in memory at any given time.
2. Weak / Soft References: Use weak or soft references for objects that can be recreated or are not critical to keep in memory.
3. Multiple Smaller Processes:  Spliting your application into multiple smaller processes can help reduce memory access latency by allowing each process to manage its own memory more efficiently.
4. Garbase Collection Algorithms: Choosing an efficient GC algorithm and tuning its parameters. 
5. Batch Processes: Process data in batches instead of single items specially in database access to reduce the number of memory accesses and improve cache locality.
6. Constantly run with main process
7. Database Normalization
8. Computer Over Storage
9. Don't Store Unnecessary Data

## Disk Access Latency
* Web Content Files
* Logging
* DB Disk Access(Data Files, Index Files, Transaction Logs): Important. 

## Minimize Disk Access Latency
1. Sequential and Batch IO 
2. Asynchronous Logging: Asynchronous logging can help minimize disk access latency by allowing the application to continue processing requests without waiting for the logging operation to complete. This can be achieved using a separate logging thread or an asynchronous logging library.
3. Web Content Caching
4. Reverse Proxy / Load Balancer
5. Separate hosting of dynamic and static content : separate servers for dynamic and static content can help minimize disk access latency by allowing each server to optimize its resources for the specific type of content it serves. Static content can be served from a cache or CDN, while dynamic content can be processed on a server optimized for application logic and database access.
6. Page Cache, Zero Copy: Page caching can help minimize disk access latency by keeping frequently accessed data in memory, reducing the need to read from disk. Zero-copy techniques can further reduce latency by allowing data to be transferred directly between the disk and the network without passing through the application layer.
7. Querry Optimization
8. Data Caching
9. Query Optimization
10. Schema Optimization
11. Highest IOPS, RAID, SSD Disks
* **Know your hardware well and optimize for it.**

## CPU Latency
* Inecfficient algorithms(Developer can cause this)
* Context Swtiching
 ## Minimize CPU Latency
1. Efficient Algorithms
2. Efficient Queries
3. Batch / Async I/O
4. Single Threaded Model
5. Thread Pool Size
6. Multi Process in Virtual Environment

## Latency Cost idea need a bit details : need a bit details about this topic

## Concurrent Request Processing: need a bit details techniques and approaches













SRE: Site Reliability Engineering (SRE) is a discipline that incorporates aspects of software engineering and applies them to infrastructure and operations problems. The main goals of SRE are to create scalable and highly reliable software systems. SRE teams are responsible for ensuring the availability, performance, and reliability of services.



To learn : there is a seminar or webinar on Hotstar about throughpuput management





