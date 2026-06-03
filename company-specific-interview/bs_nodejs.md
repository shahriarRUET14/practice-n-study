# BS Backend Developer Interview Prep

Nodejs, queue management, Kafka, cache(redis), nginx, db

## 1) Role snapshot

**Target role:** Backend Developer / Sr. Backend Developer (Software Architect)

**Core expectation:** Design and operate backend systems that are scalable, reliable, observable, and easy to evolve.

**What this interview is likely checking:**
- System design thinking, especially for high-throughput backend services
- Practical experience with Node.js, queues, Kafka, Redis, Nginx, and databases
- Fault tolerance, incident handling, and production debugging
- Distributed systems fundamentals
- Communication clarity and trade-off reasoning

## 2) Main topics to revise

- **Node.js**: event loop, async patterns, performance, memory, error handling
- **Queue management**: retries, backpressure, delayed jobs, dead-letter queues
- **Kafka**: partitions, consumer groups, offsets, ordering, replay, delivery semantics
- **Redis**: caching strategy, TTL, invalidation, locks, rate limiting, pub/sub
- **Nginx**: reverse proxy, load balancing, TLS termination, rate limiting, buffering
- **Databases**: MySQL, Cassandra, indexing, transactions, replication, consistency
- **AWS**: EC2, ECS/EKS, S3, RDS, ElastiCache, SQS/SNS, IAM, CloudWatch
- **Docker/Kubernetes**: containers, deployment, scaling, probes, resource limits, rollouts
- **Observability**: logs, metrics, traces, ELK, alerting, dashboards, root-cause analysis
- **Architecture**: microservices, event-driven design, idempotency, sagas, resilience

## 3) What to emphasize in answers

When you answer, try to show:

1. **Business impact** - why the choice matters
2. **Technical trade-offs** - what you gain and what you lose
3. **Failure handling** - what happens when something breaks
4. **Operational readiness** - how you monitor and recover
5. **Scalability** - how the design grows with traffic and data

## 4) Fault-tolerant financial system design points

If asked how to design a scalable and fault-tolerant financial system, include these points:

- **Strong correctness first**: money movement must be consistent and auditable
- **Idempotency**: every payment/transfer request should be safely retryable
- **Transactional boundaries**: use DB transactions where correctness is required
- **Outbox pattern**: persist domain change and event together to avoid lost events
- **Saga / compensation**: for multi-service workflows, define rollback or compensation steps
- **Retries with limits**: retry transient failures, but use exponential backoff and max attempts
- **Dead-letter queue**: isolate poison messages for manual inspection
- **Exactly-once illusion**: design for at-least-once delivery and deduplicate in consumers
- **Reconciliation jobs**: compare source-of-truth records with downstream systems
- **Audit trail**: keep immutable logs of all critical operations
- **Multi-AZ / multi-instance deployment**: remove single points of failure
- **Circuit breakers and timeouts**: prevent cascading failures
- **Graceful degradation**: non-critical features should fail without breaking payments
- **Observability**: logs, metrics, tracing, and alerting on business KPIs
- **Security**: encryption, IAM least privilege, secret management, tamper resistance

## 5) System design talking points for this stack

### Node.js
- Use asynchronous I/O properly
- Avoid blocking the event loop with CPU-heavy work
- Offload long tasks to queues or worker processes
- Handle errors centrally and return meaningful responses

### Kafka / queue management
- Use partitions to scale consumers
- Preserve ordering only where needed
- Use consumer groups for horizontal scaling
- Apply idempotent consumer logic
- Define retry, timeout, and DLQ strategy

### Redis
- Use as cache, distributed lock, session store, or rate limiter
- Set TTL intentionally
- Plan cache invalidation strategy
- Avoid using Redis as the only source of truth for critical data

### MySQL / Cassandra
- MySQL for transactional consistency
- Cassandra for high write throughput and wide-column access patterns
- Know when denormalization is acceptable
- Be ready to explain indexing, replication, and consistency trade-offs

### Nginx and infra
- Nginx can handle reverse proxying, TLS termination, and rate limiting
- Kubernetes should provide deployment resilience, autoscaling, and health checks
- AWS should support high availability with managed services where practical

## 6) Interview questions with answer guidance

### A. Node.js

**Q1. Why is Node.js suitable for backend services?**
- Event-driven, non-blocking I/O
- Good for I/O-heavy APIs and real-time systems
- Strong ecosystem and fast development velocity
- Mention that it is not ideal for heavy CPU-bound workloads without offloading

**Q2. What can block the Node.js event loop?**
- Synchronous CPU-heavy code
- Large JSON operations
- Expensive loops or crypto/compression on the main thread
- Fix by using workers, queues, or moving work to another service

**Q3. How do you handle errors in a Node.js service?**
- Validate input early
- Use centralized error middleware
- Return proper HTTP status codes
- Log with correlation IDs
- Avoid exposing internal stack traces to clients

### B. Queue management

**Q4. Why use a queue in backend architecture?**
- Decouple producers and consumers
- Smooth traffic spikes
- Improve resilience and asynchronous processing
- Allow retries and dead-letter handling

**Q5. How do you make queue processing reliable?**
- Acknowledge only after successful processing
- Make consumers idempotent
- Use retries with backoff
- Send failed jobs to DLQ
- Monitor lag and failure rates

### C. Kafka

**Q6. What is Kafka used for?**
- Event streaming and decoupled communication
- High-throughput event processing
- Replayable event history

**Q7. How do partitions help Kafka scale?**
- Data is split across partitions
- Consumers in a group can process partitions in parallel
- More partitions usually means more throughput, but ordering is only guaranteed within a partition

**Q8. How do you handle duplicate messages in Kafka consumers?**
- Use unique event IDs
- Store processed message IDs or transaction markers
- Make processing idempotent

### D. Redis

**Q9. When should Redis be used as a cache?**
- For frequently read, expensive-to-fetch data
- For data where slight staleness is acceptable
- Use TTL and invalidation rules

**Q10. What are common Redis pitfalls?**
- Using it as the source of truth
- Missing TTLs
- Cache stampede
- Poor key design
- Memory pressure without eviction strategy

### E. Databases

**Q11. How do you choose between MySQL and Cassandra?**
- MySQL for relational, transactional consistency and joins
- Cassandra for distributed scale and high write throughput
- Choose based on access pattern, consistency needs, and query model

**Q12. What is your approach to database scaling?**
- Optimize queries and indexing first
- Add read replicas if reads dominate
- Partition or shard when needed
- Use caching and asynchronous workflows to reduce pressure

### F. AWS, Docker, Kubernetes

**Q13. What do containers solve?**
- Consistent runtime across environments
- Easier deployment and scaling
- Isolation of dependencies

**Q14. Why is Kubernetes useful in production?**
- Self-healing, rolling deployments, autoscaling
- Service discovery and health probes
- Better orchestration for microservices

**Q15. What AWS services would you typically use for a backend platform?**
- EC2 or EKS for compute
- RDS for relational data
- ElastiCache for Redis
- S3 for object storage
- CloudWatch for metrics/logs
- IAM for access control

### G. Nginx and edge concerns

**Q16. What is Nginx used for in backend systems?**
- Reverse proxy
- Load balancing
- TLS termination
- Rate limiting
- Request buffering and routing

### H. Observability and production support

**Q17. How do you debug a production incident?**
- Identify impact and scope
- Check metrics, logs, and traces
- Look for recent deployments or config changes
- Isolate whether it is app, DB, queue, or infrastructure related
- Mitigate first, then do root-cause analysis

**Q18. What should be monitored in a financial backend?**
- Error rate
- Latency
- Queue lag
- DB saturation
- Payment success/failure rate
- Reconciliation mismatch rate
- Alert on business-level anomalies, not just infra metrics

### I. System design and architecture

**Q19. How would you design a scalable and fault-tolerant payment workflow?**
- API receives request and validates it
- Persist transaction state with an idempotency key
- Publish event reliably using outbox pattern
- Process async steps via queue/Kafka
- Retry transient failures with backoff
- Use DLQ for failures needing manual review
- Reconcile records periodically
- Ensure auditability and traceability end to end

**Q20. How do microservices stay reliable in a distributed system?**
- Clear service boundaries
- Timeouts and retries
- Circuit breakers
- Idempotency
- Observability
- Graceful degradation
- Strong deployment and rollback strategy

## 7) Quick revision checklist before the interview

- Can I explain the event loop clearly?
- Can I describe queue retry and DLQ strategy?
- Can I explain Kafka partitions and consumer groups?
- Can I compare Redis cache vs source of truth?
- Can I discuss MySQL vs Cassandra trade-offs?
- Can I describe Nginx, Kubernetes, and AWS in one architecture?
- Can I explain how to build a fault-tolerant payment or ledger flow?
- Can I walk through a production incident and RCA clearly?

## 8) Short answer pattern to use in the interview

Try this structure:

**Situation -> Design choice -> Trade-off -> Failure handling -> Monitoring**

Example:
- “For payment events, I would use an idempotent API, durable storage, and async event processing. I would choose at-least-once delivery with deduplication because it is safer than pretending exactly-once. I would add retries, DLQ, reconciliation, and tracing so failures are visible and recoverable.”

## 9) Personal notes

- Be ready to talk about your own production experience
- Mention incidents you handled and what you learned
- If you have architecture decisions from past projects, explain the trade-offs honestly
- Keep answers structured and concise
