# System Scalability Summary

## Core Concepts

**Scaling vs. Performance:**
- _Scalability is a subset of performance_
- Scaling increases *throughput* (concurrent requests), Performance improves latency
- Performance improves response speed; scalability enables handling more simultaneous users

## Two Scaling Approaches

**Vertical Scaling (Scale-Up):**
- Add resources (CPU, RAM, storage) to a single machine
- Advantages: Simple, cost-effective for small workloads, fast internal communication
- Disadvantages: Hardware limits Can not extend to infinity, downtime during upgrades, high upfront costs, single point of failure
- Best for: Predictable workloads, monolithic applications, Small to medium traffic.

**Horizontal Scaling (Scale-Out):**
- Add multiple smaller machines (nodes) to distribute load
- Advantages: No hardware limits, cost-effective growth, fault tolerance, dynamic scaling, handles unpredictable workloads
- Requires: Load balancing to distribute traffic across nodes
- Challenges: Complexity in coordination, data consistency, deployment management

## Key Takeaways from salable systems lecture:

1. Scaling solves throughput limitations but requires separate performance optimization for latency
2. Vertical scaling hits hardware limits; horizontal scaling is practically unlimited
3. Horizontal scaling demands load balancing and introduces operational complexity
4. Choose vertical for stable, predictable systems; horizontal for dynamic, growing systems


## How to Achieve Horizontal Scaling
- Load Balancing
- Clustering
- Containerization
- Auto Scaling
- 
- Use **multiple servers** instead of one bigger server.
- Put a **load balancer** in front to distribute traffic.
- Keep application servers **stateless** so any request can go to any node.
- Store persistent data in external systems like **databases, caches, or object storage**.
- Use **replication** for availability and **sharding** for large datasets.
- Use **container orchestration** like Kubernetes for deployment, scaling, and failover.
- For monoliths, split into **modules** or **microservices** when scaling becomes difficult.
- Prefer **shared-nothing design** where components are independent.

### Key interview points

- Horizontal scaling means **adding more machines**, not upgrading one machine.
- It improves **throughput** and **fault tolerance**.
- It is harder than vertical scaling because of:
    - state management
    - data consistency
    - deployment coordination
    - routing complexity
- Stateless services are much easier to scale horizontally.
- Stateful systems like databases need special techniques such as **replication, sharding, and indexing**.

### Suggested interview answer

> Horizontal scaling is achieved by adding more servers and distributing requests through a load balancer. To make this work well, application instances should be stateless, while persistent data should be stored externally in databases or storage systems. For reliability and scale, databases often use replication or sharding, and orchestration tools like Kubernetes help manage deployment and failover. The main challenge is maintaining consistency and coordinating multiple nodes.

### Suggested improvement to the tutorial section

Replace the current text with a structure like:

1. **Definition**
2. **How it works**
3. **Why stateless apps scale better**
4. **Challenges in monolithic systems**
5. **Common solutions**
6. **Interview takeaway**

If needed, I can rewrite the selected section into a cleaner interview-ready version.

[00:54:28] **Horizontal scaling introduces complexity**, including:

- Managing multiple instances.
- Coordinating inter-node communication.
- Ensuring data consistency.
- Handling deployment and updates across many servers.

Despite complexity, it is essential for scalable, maintainable systems.

[00:57:29] Horizontal scaling is ideal for **dynamic and unpredictable workloads**, such as peak traffic during live events or e-commerce spikes. Cloud environments facilitate easy provisioning and de-provisioning of instances to match demand.

[01:00:52] The lecture addresses practical concerns such as practicing load balancing on real servers, and confirms that database scaling is a separate, complex topic distinct from application scaling.

[01:03:48] **Replication** is explained as creating clones of data or services to improve availability and load distribution. Replicas must stay synchronized to be effective.

[01:06:37] The principle of **decentralization and independence** is introduced as a scalability foundation:

- Systems should not rely on a single central component.
- Responsibilities and functions must be distributed among multiple components.
- Components operate independently, reducing bottlenecks and improving fault tolerance.

[01:11:14] Decentralization leads to:

- **Improved scalability** by distributing load.
- **Fault tolerance**: failure of one component does not bring down the system.
- **Disaster resilience** through backups and multi-region deployment.
- **Flexibility** to add or remove servers dynamically.

[01:15:22] **Independence** means components are **loosely coupled**, allowing:

- Independent development, deployment, scaling.
- Isolation of concerns, minimizing impact of changes.
- Easier maintenance and parallel development.

[01:17:02] This leads to **service-oriented or microservices architecture**, where components are independent services communicating over APIs.

[01:20:41] The lecture transitions to **Load Balancers**, their types, and functionality:

- Load balancers can be **software or hardware**.
- They distribute incoming network traffic across multiple servers to optimize resource use and enhance availability.
- Commonly deployed for web servers, application servers, databases, and more.
- They support algorithms like **Round Robin** and **Weighted Round Robin**.
- Perform **health checks** to avoid routing traffic to unavailable servers.
- Support **session persistence (sticky sessions)**.
- Handle SSL/TLS termination and offloading.
- Enable global load balancing across geographic regions.
- Provide rate limiting and traffic shaping.
- Log and monitor traffic.

[01:34:32] Load balancers are used in diverse contexts including web servers, mail servers, VoIP, DNS, streaming, and hybrid cloud environments.

[01:44:56] The lecture explains **Reverse Proxy**:

- Positioned between clients and web servers.
- Forwards client requests to backend servers and returns responses.
- Enhances security by hiding backend IPs.
- Performs SSL termination, caching, compression.
- Manages authentication and authorization.
- Provides firewall capabilities.
- Supports content rewriting and request transformation.
- Enables session management.
- Often implemented via NGINX or HAProxy.
- Shares many features with load balancers but focuses more on request forwarding and security.

[01:57:15] The relationship between **Load Balancer, Reverse Proxy, and API Gateway** is clarified:

- All can distribute load and forward requests.
- API Gateways specialize in **API management**, including authentication, authorization, request/response transformations, rate limiting, analytics, and versioning.
- API Gateways act as a **centralized entry point** for multiple microservices or APIs.
- Reverse proxies emphasize security and request forwarding.
- Load balancers focus on distributing traffic to ensure optimal resource use.

[02:10:50] API Gateways can:

- Function as load balancers and reverse proxies.
- Handle complex API routing.
- Support service discovery.
- Provide centralized security, caching, and analytics.
- Enable versioning and transformation of API requests/responses.
- Integrate with cloud platforms (AWS, Azure) or open-source solutions (Kong).

[02:31:11] The lecture concludes that system design involves choosing and combining these components flexibly according to needs. API Gateways typically serve as a **single point of API management**, with load balancers and reverse proxies supporting scalability and security.

---

### Summary Table: Scaling Types and Characteristics

| Aspect                     | Vertical Scaling                       | Horizontal Scaling                        |
|----------------------------|--------------------------------------|------------------------------------------|
| Approach                   | Add resources to a single server     | Add more servers (nodes)                  |
| Hardware Dependency        | High                                 | Moderate to low                           |
| Downtime for Upgrades      | Yes                                  | No (can be zero-downtime with orchestration) |
| Cost                       | High upfront                        | Pay-as-you-grow (cloud-friendly)         |
| Fault Tolerance            | Single point of failure              | High (redundancy via multiple nodes)     |
| Scalability Limit          | Hardware capacity                    | Practically unlimited                     |
| Complexity                 | Low                                 | High (coordination, data consistency)    |
| Deployment                 | Simple                              | Complex (requires orchestration)          |
| Suitable for               | Predictable, stable workloads       | Dynamic, unpredictable workloads          |

---

### Load Balancer Key Functions

- Distributes incoming traffic across multiple servers.
- Supports algorithms: Round Robin, Weighted Round Robin.
- Performs health checks, removes unhealthy servers.
- Manages session persistence (sticky sessions).
- SSL/TLS termination and offloading.
- Rate limiting and traffic shaping.
- Supports global load balancing (multi-region).
- Provides logging and monitoring.

---

### Reverse Proxy Features

- Forwards client requests to backend servers.
- Hides backend server details (IP addresses).
- SSL termination and encryption management.
- Caching and compression.
- Authentication and authorization enforcement.
- Request rewriting and transformation.
- Acts as a firewall.
- Enables session management.
- Often implemented by NGINX, HAProxy.

---

### API Gateway Characteristics

- Centralized entry point for multiple APIs/microservices.
- Handles authentication, authorization, rate limiting.
- Supports request/response transformation.
- Provides analytics and logging.
- Manages API versioning.
- Can act as load balancer and reverse proxy.
- Integrates with cloud platforms and open-source tools.

---

### Key Insights

- **Scaling is essential to handle load beyond hardware limits despite performance optimization.**
- **Vertical scaling is limited and costly with downtime and single points of failure.**
- **Horizontal scaling offers fault tolerance, flexibility, and cost-effectiveness but introduces complexity.**
- **Monolithic applications are hard to scale horizontally without refactoring into modular or microservices architectures.**
- **Stateless applications are easier to scale horizontally; stateful components like databases require advanced techniques (replication, sharding).**
- **Load balancers, reverse proxies, and API gateways are complementary tools essential for scalable, secure, and manageable systems.**
- **API Gateways provide specialized API management capabilities beyond load balancing and proxying.**
- **Decentralization and independence of components are fundamental principles for building scalable distributed systems.**

---

This comprehensive lecture covered the theoretical foundations and practical components of application scalability, focusing on vertical and horizontal scaling, their trade-offs, and the roles of load balancers, reverse proxies, and API gateways in modern system architectures.