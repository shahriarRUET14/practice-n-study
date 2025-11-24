# Advanced Spring Boot Topics

## Quick Reference Snippets

---

## Transaction Management

### ACID Transactions (Single Database)

```java
@Transactional(propagation = Propagation.REQUIRED)
@Transactional(isolation = Isolation.READ_COMMITTED)
@Transactional(timeout = 30)
@Transactional(rollbackFor = Exception.class)
```

### Distributed Transactions: SAGA Pattern

**Problem:** ACID transactions don't work across microservices (different databases)

**Solution:** SAGA Pattern - Sequence of local transactions with compensation

**Two Approaches:**

**1. Choreography (Event-Driven)**
```java
// Each service publishes events
@Service
public class OrderService {
    
    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
        // Publish event
        kafkaTemplate.send("order-created", new OrderCreatedEvent(order));
    }
}

@Service
public class PaymentService {
    
    @KafkaListener(topics = "order-created")
    @Transactional
    public void processPayment(OrderCreatedEvent event) {
        try {
            paymentRepository.charge(event.getOrderId(), event.getAmount());
            kafkaTemplate.send("payment-completed", new PaymentCompletedEvent(event));
        } catch (Exception e) {
            // Compensate
            kafkaTemplate.send("payment-failed", new PaymentFailedEvent(event));
        }
    }
}
```

**2. Orchestration (Centralized)**
```java
// Orchestrator coordinates all steps
@Service
public class OrderOrchestrator {
    
    public void createOrder(OrderRequest request) {
        SagaTransaction saga = new SagaTransaction();
        
        try {
            // Step 1: Create Order
            Order order = orderService.createOrder(request);
            saga.addStep("create-order", order);
            
            // Step 2: Process Payment
            Payment payment = paymentService.charge(order);
            saga.addStep("charge-payment", payment);
            
            // Step 3: Reserve Inventory
            Inventory inventory = inventoryService.reserve(order);
            saga.addStep("reserve-inventory", inventory);
            
            saga.complete();
        } catch (Exception e) {
            // Compensate in reverse order
            saga.compensate();
        }
    }
    
    private void compensate(SagaTransaction saga) {
        // Reverse order
        if (saga.hasStep("reserve-inventory")) {
            inventoryService.release(saga.getStep("reserve-inventory"));
        }
        if (saga.hasStep("charge-payment")) {
            paymentService.refund(saga.getStep("charge-payment"));
        }
        if (saga.hasStep("create-order")) {
            orderService.cancel(saga.getStep("create-order"));
        }
    }
}
```

**SAGA Principles:**
- Each step is a local ACID transaction
- If any step fails, compensate previous steps
- Compensation must be idempotent
- No locks across services (eventual consistency)

**Use Cases:**
- Order processing across services
- Payment workflows
- Multi-step business processes
- Distributed data updates

---

## AOP - Aspect-Oriented Programming

```java
@Aspect
@Component
public class LoggingAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        // Advice logic
    }
}
```

---

## Spring Security

```java
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        // Security configuration
    }
}
```

---

## Custom Auto-Configuration

```java
@Configuration
@ConditionalOnClass(SomeClass.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
    // Auto-configuration logic
}
```

---

## Messaging Systems: Kafka vs RabbitMQ

### Kafka - Event Streaming Platform
```java
// Producer
@Autowired
private KafkaTemplate<String, String> kafkaTemplate;

public void publishEvent(String topic, String message) {
    kafkaTemplate.send(topic, message);
}

// Consumer
@KafkaListener(topics = "my-topic", groupId = "my-group")
public void consume(String message) {
    // Process message
}
```

**Use Cases:**
- High-throughput event streaming
- Event sourcing & CQRS
- Real-time analytics
- Log aggregation
- Microservices event-driven architecture

### RabbitMQ - Message Broker
```java
@Autowired
private RabbitTemplate rabbitTemplate;

public void sendMessage(String exchange, String routingKey, Object message) {
    rabbitTemplate.convertAndSend(exchange, routingKey, message);
}
```

**Use Cases:**
- Request/response patterns
- Task queues
- Complex routing (direct, topic, fanout, headers)
- Priority queues
- Dead letter queues

### Key Differences Summary

| Feature | Kafka | RabbitMQ |
|---------|-------|----------|
| **Model** | Event streaming (log-based) | Message broker (queue-based) |
| **Message Retention** | Configurable (days/weeks) | Deleted after consumption |
| **Throughput** | Very high (millions/sec) | High (thousands/sec) |
| **Latency** | Higher (batching) | Lower (immediate) |
| **Ordering** | Per-partition guaranteed | Per-queue guaranteed |
| **Multiple Consumers** | Native (consumer groups) | Multiple queues needed |
| **Replay** | Yes (historical events) | No (messages deleted) |
| **Routing** | Simple (topics/partitions) | Complex (exchanges) |
| **Use Case** | Event streaming, analytics | Task queues, RPC |

### Why Use Kafka in Microservices?

**Answer:**
Kafka is ideal for microservices because it supports event-driven architecture where services communicate asynchronously through events. It provides:

1. **Decoupling**: Services don't need to know about each other
2. **Scalability**: Handles millions of events per second
3. **Event Replay**: New services can consume historical events
4. **Multiple Consumers**: Same event can be consumed by analytics, notifications, audit services
5. **Ordering**: Maintains event order per partition
6. **Durability**: Events are persisted, providing audit trail

**In MFS system:** We use Kafka for transaction events, user activity streams, and inter-service communication where multiple services need to react to the same events.

### Key Takeaway for Interview

**Kafka excels in event-driven microservices where you need:**
- High throughput (millions of events/sec)
- Multiple consumers of the same events
- Event replay and historical data access
- Event sourcing and audit trail
- Decoupled service communication

**RabbitMQ is better for:**
- Task queues and job processing
- Request/response patterns
- Complex routing requirements
- Low-latency immediate delivery

---

## Redis: Use Cases & Implementation

### Redis Configuration in Spring Boot

```java
// Configuration
@Configuration
@EnableCaching
public class RedisConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setDefaultSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}

// Usage as Cache
@Service
public class UserService {
    @Cacheable(value = "users", key = "#id")
    public User getUser(Long id) {
        return userRepository.findById(id);
    }
    
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
}

// Direct Redis Operations
@Autowired
private RedisTemplate<String, Object> redisTemplate;

public void storeData(String key, Object value) {
    redisTemplate.opsForValue().set(key, value, Duration.ofMinutes(10));
}
```

### Redis Use Cases

**1. Cache (Primary Use)**
- Session storage
- Frequently accessed data
- API response caching
- Database query result caching

**2. Database (Secondary)**
- Real-time analytics
- Leaderboards
- Counters and statistics
- Time-series data

**3. Message Broker (Pub/Sub)**
- Real-time notifications
- Event broadcasting
- Chat applications
- Real-time dashboards

### Why Use Redis?

1. **Ultra-fast Performance**: In-memory storage, sub-millisecond latency
2. **Rich Data Structures**: Strings, Lists, Sets, Sorted Sets, Hashes, Bitmaps
3. **Persistence Options**: RDB snapshots, AOF (Append Only File)
4. **High Availability**: Redis Sentinel, Redis Cluster
5. **Atomic Operations**: Transactions, Lua scripting
6. **Pub/Sub**: Real-time messaging capabilities


### Key Takeaway for Interview

**Redis in MFS:**
- **Primary Role**: Fast access cache for frequently used data
- **Why Redis**: Sub-millisecond latency, rich data structures, high availability
- **Without Redis**: Direct database queries → slower response times, higher DB load

**Redis + Kafka Combination:**
- **Kafka**: Handles async event publishing/consuming (decoupling, scalability)
- **Redis**: Provides instant data access (performance, reduced DB load)
- **Together**: Event-driven architecture with fast data retrieval

**Alternatives:**
- Caching: Caffeine, EhCache (in-process) or Memcached (distributed)
- Database: PostgreSQL, MongoDB (persistent storage)
- Message Broker: RabbitMQ (if not using Kafka)

---

## Distributed Transactions: SAGA Pattern

### Problem: ACID Transactions in Microservices

**Challenge:**
- ACID transactions require 2PC (Two-Phase Commit)
- 2PC doesn't scale in microservices (blocking, single point of failure)
- Each service has its own database
- Cannot use `@Transactional` across services

**Example Problem:**
```java
// This DOESN'T work across services
@Transactional
public void processOrder(Order order) {
    orderService.save(order);        // Service 1, DB 1
    paymentService.charge(order);     // Service 2, DB 2
    inventoryService.reserve(order);  // Service 3, DB 3
    // If step 3 fails, how to rollback step 1 and 2?
}
```

### Solution: SAGA Pattern

**SAGA = Sequence of local transactions with compensation**

**Key Principles:**
1. Each step is a **local ACID transaction** (within one service)
2. If any step fails, **compensate** previous steps (reverse operations)
3. **No distributed locks** (eventual consistency)
4. Compensation must be **idempotent** (safe to retry)

### SAGA Implementation Patterns

**1. Choreography (Event-Driven) - Decentralized**

```java
// Order Service
@Service
public class OrderService {
    
    @Transactional
    public void createOrder(Order order) {
        order.setStatus(OrderStatus.PENDING);
        orderRepository.save(order);
        
        // Publish event
        kafkaTemplate.send("order-created", 
            new OrderCreatedEvent(order.getId(), order.getAmount()));
    }
    
    @KafkaListener(topics = "order-failed")
    @Transactional
    public void cancelOrder(OrderFailedEvent event) {
        Order order = orderRepository.findById(event.getOrderId())
            .orElseThrow();
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
    }
}

// Payment Service
@Service
public class PaymentService {
    
    @KafkaListener(topics = "order-created")
    @Transactional
    public void processPayment(OrderCreatedEvent event) {
        try {
            Payment payment = new Payment();
            payment.setOrderId(event.getOrderId());
            payment.setAmount(event.getAmount());
            payment.setStatus(PaymentStatus.PROCESSING);
            paymentRepository.save(payment);
            
            // Charge customer
            chargeCustomer(payment);
            
            payment.setStatus(PaymentStatus.COMPLETED);
            paymentRepository.save(payment);
            
            // Publish success event
            kafkaTemplate.send("payment-completed", 
                new PaymentCompletedEvent(event.getOrderId()));
        } catch (Exception e) {
            // Publish failure event
            kafkaTemplate.send("payment-failed", 
                new PaymentFailedEvent(event.getOrderId()));
        }
    }
    
    @KafkaListener(topics = "order-failed")
    @Transactional
    public void refundPayment(OrderFailedEvent event) {
        Payment payment = paymentRepository.findByOrderId(event.getOrderId());
        if (payment != null && payment.getStatus() == PaymentStatus.COMPLETED) {
            refundCustomer(payment);
            payment.setStatus(PaymentStatus.REFUNDED);
            paymentRepository.save(payment);
        }
    }
}

// Inventory Service
@Service
public class InventoryService {
    
    @KafkaListener(topics = "payment-completed")
    @Transactional
    public void reserveInventory(PaymentCompletedEvent event) {
        try {
            Inventory inventory = inventoryRepository.findByOrderId(event.getOrderId());
            inventory.setStatus(InventoryStatus.RESERVED);
            inventoryRepository.save(inventory);
            
            kafkaTemplate.send("inventory-reserved", 
                new InventoryReservedEvent(event.getOrderId()));
        } catch (Exception e) {
            kafkaTemplate.send("inventory-failed", 
                new InventoryFailedEvent(event.getOrderId()));
        }
    }
    
    @KafkaListener(topics = "order-failed")
    @Transactional
    public void releaseInventory(OrderFailedEvent event) {
        Inventory inventory = inventoryRepository.findByOrderId(event.getOrderId());
        if (inventory != null && inventory.getStatus() == InventoryStatus.RESERVED) {
            inventory.setStatus(InventoryStatus.AVAILABLE);
            inventoryRepository.save(inventory);
        }
    }
}
```

**Flow:**
```
Order Created → Payment Processing → Inventory Reservation
     ↓              ↓                      ↓
  (Success)    (Success)            (Success/Failure)
     ↓              ↓                      ↓
  (If fails)   (Refund)            (Release)
```

**2. Orchestration (Centralized) - Recommended for Complex Flows**

```java
// Saga Orchestrator
@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private SagaStateRepository sagaStateRepository;
    
    public void executeOrderSaga(OrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        SagaState sagaState = new SagaState(sagaId, SagaStatus.IN_PROGRESS);
        sagaStateRepository.save(sagaState);
        
        try {
            // Step 1: Create Order
            Order order = orderService.createOrder(request);
            sagaState.addStep("create-order", order.getId(), StepStatus.COMPLETED);
            sagaStateRepository.save(sagaState);
            
            // Step 2: Process Payment
            Payment payment = paymentService.charge(order.getId(), order.getAmount());
            sagaState.addStep("charge-payment", payment.getId(), StepStatus.COMPLETED);
            sagaStateRepository.save(sagaState);
            
            // Step 3: Reserve Inventory
            Inventory inventory = inventoryService.reserve(order.getId());
            sagaState.addStep("reserve-inventory", inventory.getId(), StepStatus.COMPLETED);
            sagaStateRepository.save(sagaState);
            
            // All steps completed
            sagaState.setStatus(SagaStatus.COMPLETED);
            sagaStateRepository.save(sagaState);
            
        } catch (Exception e) {
            // Compensate in reverse order
            compensate(sagaState);
            throw new SagaException("Order saga failed", e);
        }
    }
    
    private void compensate(SagaState sagaState) {
        sagaState.setStatus(SagaStatus.COMPENSATING);
        sagaStateRepository.save(sagaState);
        
        // Compensate in reverse order
        List<SagaStep> steps = sagaState.getSteps();
        Collections.reverse(steps);
        
        for (SagaStep step : steps) {
            if (step.getStatus() == StepStatus.COMPLETED) {
                try {
                    switch (step.getName()) {
                        case "reserve-inventory":
                            inventoryService.release(step.getEntityId());
                            break;
                        case "charge-payment":
                            paymentService.refund(step.getEntityId());
                            break;
                        case "create-order":
                            orderService.cancel(step.getEntityId());
                            break;
                    }
                    step.setStatus(StepStatus.COMPENSATED);
                } catch (Exception e) {
                    // Log and continue compensation
                    step.setStatus(StepStatus.COMPENSATION_FAILED);
                }
            }
        }
        
        sagaState.setStatus(SagaStatus.COMPENSATED);
        sagaStateRepository.save(sagaState);
    }
}
```

### Choreography vs Orchestration

| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| **Control** | Decentralized (events) | Centralized (orchestrator) |
| **Complexity** | Simple flows | Complex flows |
| **Coupling** | Services know events | Services know orchestrator |
| **Visibility** | Hard to track | Easy to track |
| **Failure Handling** | Distributed | Centralized |
| **Best For** | Simple, few services | Complex, many services |

### SAGA Best Practices

**1. Idempotency**
```java
// Compensation must be idempotent
public void refundPayment(String paymentId) {
    Payment payment = paymentRepository.findById(paymentId);
    if (payment.getStatus() != PaymentStatus.REFUNDED) {
        // Only refund if not already refunded
        refundCustomer(payment);
        payment.setStatus(PaymentStatus.REFUNDED);
        paymentRepository.save(payment);
    }
}
```

**2. Saga State Management**
```java
// Store saga state for recovery
@Entity
public class SagaState {
    private String sagaId;
    private SagaStatus status;
    private List<SagaStep> steps;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

**3. Timeout Handling**
```java
// Set timeouts for each step
@Scheduled(fixedDelay = 60000) // Check every minute
public void checkSagaTimeouts() {
    List<SagaState> stuckSagas = sagaStateRepository
        .findByStatusAndUpdatedAtBefore(
            SagaStatus.IN_PROGRESS,
            LocalDateTime.now().minusMinutes(10)
        );
    
    for (SagaState saga : stuckSagas) {
        compensate(saga);
    }
}
```

**4. Event Ordering**
```java
// Use Kafka partitions for ordering
kafkaTemplate.send("order-events", orderId, event);
// Same orderId → same partition → ordered events
```

### Interview Answer: SAGA Pattern

**Q: How do you handle transactions in a distributed system?**

**Answer:**
"In microservices, we can't use traditional ACID transactions across services because each service has its own database. We use the **SAGA Pattern** to manage distributed transactions.

**SAGA Pattern:**
- Sequence of local ACID transactions (each within one service)
- If any step fails, compensate previous steps (reverse operations)
- No distributed locks (eventual consistency)
- Compensation must be idempotent

**Two Approaches:**

**1. Choreography (Event-Driven):**
- Services communicate via events (Kafka)
- Each service listens to events and performs its step
- If failure, publishes compensation event
- Decentralized, good for simple flows

**Example:**
```
Order Created → Payment Processing → Inventory Reservation
If inventory fails → Compensation events flow back
```

**2. Orchestration (Centralized):**
- Orchestrator coordinates all steps
- Maintains saga state
- If failure, orchestrator triggers compensation
- Centralized, good for complex flows

**Example:**
```
Orchestrator:
  1. Create Order
  2. Charge Payment
  3. Reserve Inventory
If step 3 fails → Compensate in reverse: Release Inventory → Refund Payment → Cancel Order
```

**Key Principles:**
- Each step is a local transaction
- Compensation must be idempotent
- Store saga state for recovery
- Handle timeouts
- Use event ordering (Kafka partitions)

**In MFS:** We use orchestration for complex payment flows where we need visibility and centralized control."

### Key Takeaway for Interview

**SAGA Pattern for Distributed Transactions:**
- **Problem**: ACID transactions don't work across microservices
- **Solution**: Sequence of local transactions with compensation
- **Approaches**: Choreography (events) or Orchestration (centralized)
- **Principles**: Idempotent compensation, no distributed locks, eventual consistency
- **Use Cases**: Order processing, payment workflows, multi-step processes

**When to Use:**
- Choreography: Simple flows, few services, event-driven architecture
- Orchestration: Complex flows, many services, need visibility and control

---

## MFS SAGA Implementation: Transaction & Commission Management

### Use Case 1: Transaction Management SAGA

**Scenario:** Money transfer between two accounts

**Services Involved:**
1. Transaction Service
2. Account Service (Sender)
3. Account Service (Receiver)
4. Notification Service
5. Audit Service

**SAGA Steps (Orchestration):**

```
[1] Create Transaction Record
    Service: Transaction Service
    Action: Save transaction with status PENDING
    Compensation: Delete transaction record
    
[2] Debit Sender Account
    Service: Account Service
    Action: Deduct amount from sender balance
    Compensation: Credit back to sender account
    
[3] Credit Receiver Account
    Service: Account Service
    Action: Add amount to receiver balance
    Compensation: Debit from receiver account
    
[4] Update Transaction Status
    Service: Transaction Service
    Action: Mark transaction as COMPLETED
    Compensation: Mark as FAILED
    
[5] Send Notifications (Async)
    Service: Notification Service
    Action: Send SMS/Email to both parties
    Compensation: Send failure notification
    
[6] Audit Log (Async)
    Service: Audit Service
    Action: Log transaction for compliance
    No compensation needed (read-only)
```

**Implementation:**

```java
@Service
public class TransactionSagaOrchestrator {
    
    public void executeTransaction(TransactionRequest request) {
        String sagaId = UUID.randomUUID().toString();
        SagaState saga = new SagaState(sagaId, SagaStatus.IN_PROGRESS);
        
        try {
            // Step 1: Create Transaction
            Transaction tx = transactionService.create(request);
            saga.addStep("create-transaction", tx.getId(), StepStatus.COMPLETED);
            
            // Step 2: Debit Sender
            Account sender = accountService.debit(request.getFromAccount(), request.getAmount());
            saga.addStep("debit-sender", sender.getId(), StepStatus.COMPLETED);
            
            // Step 3: Credit Receiver
            Account receiver = accountService.credit(request.getToAccount(), request.getAmount());
            saga.addStep("credit-receiver", receiver.getId(), StepStatus.COMPLETED);
            
            // Step 4: Update Transaction
            transactionService.complete(tx.getId());
            saga.addStep("update-transaction", tx.getId(), StepStatus.COMPLETED);
            
            // Step 5 & 6: Async (no compensation needed)
            notificationService.sendSuccessNotification(tx);
            auditService.logTransaction(tx);
            
            saga.setStatus(SagaStatus.COMPLETED);
            
        } catch (Exception e) {
            compensate(saga); // Reverse in order: 4→3→2→1
            throw new TransactionException("Transaction failed", e);
        }
    }
    
    private void compensate(SagaState saga) {
        // Reverse order: 4 → 3 → 2 → 1
        if (saga.hasStep("update-transaction")) {
            transactionService.fail(saga.getStep("update-transaction").getEntityId());
        }
        if (saga.hasStep("credit-receiver")) {
            accountService.debit(saga.getStep("credit-receiver").getEntityId(), amount);
        }
        if (saga.hasStep("debit-sender")) {
            accountService.credit(saga.getStep("debit-sender").getEntityId(), amount);
        }
        if (saga.hasStep("create-transaction")) {
            transactionService.delete(saga.getStep("create-transaction").getEntityId());
        }
    }
}
```

### Use Case 2: Commission Management SAGA (Near Real-Time)

**Scenario:** Calculate and distribute commission after successful transaction

**Services Involved:**
1. Commission Service
2. Account Service (Agent/Merchant)
3. Settlement Service
4. Reporting Service

**SAGA Steps (Choreography - Event-Driven):**

```
Transaction Completed Event (from Transaction SAGA)
    ↓
[1] Calculate Commission
    Service: Commission Service
    Action: Calculate commission based on transaction amount & rate
    Event: commission-calculated
    
[2] Credit Agent Account
    Service: Account Service
    Action: Add commission to agent/merchant balance
    Event: commission-credited
    
[3] Update Settlement
    Service: Settlement Service
    Action: Record commission for daily settlement
    Event: settlement-updated
    
[4] Update Reporting (Async)
    Service: Reporting Service
    Action: Update commission reports
    No compensation needed
```

**Implementation:**

```java
// Commission Service
@Service
public class CommissionService {
    
    @KafkaListener(topics = "transaction-completed", groupId = "commission-group")
    @Transactional
    public void calculateCommission(TransactionCompletedEvent event) {
        try {
            // Calculate commission
            Commission commission = new Commission();
            commission.setTransactionId(event.getTransactionId());
            commission.setAmount(calculateCommissionAmount(event));
            commission.setStatus(CommissionStatus.CALCULATED);
            commissionRepository.save(commission);
            
            // Publish event
            kafkaTemplate.send("commission-calculated", 
                new CommissionCalculatedEvent(commission));
        } catch (Exception e) {
            kafkaTemplate.send("commission-failed", 
                new CommissionFailedEvent(event.getTransactionId()));
        }
    }
    
    @KafkaListener(topics = "commission-credit-failed")
    @Transactional
    public void compensateCommission(CommissionCreditFailedEvent event) {
        // Mark commission as failed
        Commission commission = commissionRepository
            .findByTransactionId(event.getTransactionId());
        commission.setStatus(CommissionStatus.FAILED);
        commissionRepository.save(commission);
    }
}

// Account Service
@Service
public class AccountService {
    
    @KafkaListener(topics = "commission-calculated", groupId = "account-group")
    @Transactional
    public void creditCommission(CommissionCalculatedEvent event) {
        try {
            Commission commission = commissionRepository
                .findById(event.getCommissionId()).orElseThrow();
            
            // Credit agent account
            Account agentAccount = accountRepository
                .findByAgentId(commission.getAgentId());
            agentAccount.credit(commission.getAmount());
            accountRepository.save(agentAccount);
            
            commission.setStatus(CommissionStatus.CREDITED);
            commissionRepository.save(commission);
            
            // Publish event
            kafkaTemplate.send("commission-credited", 
                new CommissionCreditedEvent(commission));
        } catch (Exception e) {
            kafkaTemplate.send("commission-credit-failed", 
                new CommissionCreditFailedEvent(event.getCommissionId()));
        }
    }
    
    @KafkaListener(topics = "commission-credit-failed")
    @Transactional
    public void compensateCredit(CommissionCreditFailedEvent event) {
        // Debit back if already credited
        Commission commission = commissionRepository.findById(event.getCommissionId())
            .orElseThrow();
        
        if (commission.getStatus() == CommissionStatus.CREDITED) {
            Account agentAccount = accountRepository
                .findByAgentId(commission.getAgentId());
            agentAccount.debit(commission.getAmount());
            accountRepository.save(agentAccount);
        }
    }
}

// Settlement Service
@Service
public class SettlementService {
    
    @KafkaListener(topics = "commission-credited", groupId = "settlement-group")
    @Transactional
    public void updateSettlement(CommissionCreditedEvent event) {
        Commission commission = commissionRepository
            .findById(event.getCommissionId()).orElseThrow();
        
        // Update daily settlement
        Settlement settlement = settlementRepository
            .findByDateAndAgentId(LocalDate.now(), commission.getAgentId());
        settlement.addCommission(commission.getAmount());
        settlementRepository.save(settlement);
        
        kafkaTemplate.send("settlement-updated", 
            new SettlementUpdatedEvent(settlement));
    }
}
```

### MFS SAGA Flow Diagram

```
Transaction Management SAGA:
┌─────────────────────────────────────────┐
│ 1. Create Transaction (PENDING)        │
│ 2. Debit Sender Account                 │
│ 3. Credit Receiver Account              │
│ 4. Update Transaction (COMPLETED)       │
│ 5. Send Notifications (Async)            │
│ 6. Audit Log (Async)                     │
└─────────────────────────────────────────┘
              │
              ▼ (Publish Event)
┌─────────────────────────────────────────┐
│ Commission Management SAGA (Event-Driven)│
│ 1. Calculate Commission                 │
│ 2. Credit Agent Account                  │
│ 3. Update Settlement                     │
│ 4. Update Reporting (Async)              │
└─────────────────────────────────────────┘
```

### Key Design Decisions

**1. Transaction SAGA: Orchestration**
- **Why**: Need strict order, visibility, centralized control
- **Critical**: Money movement must be atomic
- **Compensation**: Must reverse in exact order

**2. Commission SAGA: Choreography**
- **Why**: Near real-time, less critical, event-driven
- **Async**: Can be processed asynchronously
- **Compensation**: Event-based rollback

**3. Idempotency**
```java
// Transaction idempotency
public void debit(String accountId, BigDecimal amount) {
    Account account = accountRepository.findById(accountId);
    if (account.getBalance().compareTo(amount) >= 0) {
        account.debit(amount);
        accountRepository.save(account);
    }
    // If already debited, do nothing
}

// Commission idempotency
public void creditCommission(String commissionId) {
    Commission commission = commissionRepository.findById(commissionId);
    if (commission.getStatus() != CommissionStatus.CREDITED) {
        // Credit only if not already credited
        creditAgent(commission);
    }
}
```

**4. Event Ordering (Kafka)**
```java
// Use transactionId as partition key for ordering
kafkaTemplate.send("transaction-events", transactionId, event);
// Same transactionId → same partition → ordered processing
```

### Interview Questions & Answers

**Q1: How do you handle transaction management in MFS using SAGA?**

**Answer:**
"In MFS, we use **SAGA Orchestration** for transaction management because money movement requires strict ordering and visibility.

**Transaction SAGA Flow:**
1. Create transaction record (PENDING)
2. Debit sender account
3. Credit receiver account
4. Update transaction (COMPLETED)
5. Send notifications (async)
6. Audit log (async)

**Why Orchestration:**
- Need centralized control for money movement
- Strict ordering is critical
- Easy to track and debug
- Centralized compensation handling

**Compensation:**
If any step fails, we compensate in reverse order:
- Step 4 fails → Mark transaction as FAILED
- Step 3 fails → Debit receiver (reverse credit)
- Step 2 fails → Credit sender (reverse debit)
- Step 1 fails → Delete transaction record

**Key Features:**
- Idempotent operations (safe to retry)
- Saga state stored for recovery
- Timeout handling for stuck transactions
- Event publishing for downstream services"

**Q2: How do you handle commission management in near real-time?**

**Answer:**
"For commission management, we use **SAGA Choreography** (event-driven) because it's less critical and needs to be near real-time.

**Commission SAGA Flow (Event-Driven):**
1. Listen to `transaction-completed` event
2. Calculate commission (Commission Service)
3. Credit agent account (Account Service)
4. Update settlement (Settlement Service)
5. Update reporting (Reporting Service - async)

**Why Choreography:**
- Near real-time processing
- Less critical than money movement
- Decoupled services
- Scales well with high transaction volume

**Compensation:**
If commission credit fails:
- Publish `commission-credit-failed` event
- Commission Service marks commission as FAILED
- Account Service debits back if already credited

**Key Features:**
- Event-driven (Kafka)
- Idempotent compensation
- Async processing for reporting
- Partition key (transactionId) for ordering"

**Q3: What happens if commission calculation fails after transaction is completed?**

**Answer:**
"Commission calculation failure doesn't affect the main transaction because:
1. **Transaction SAGA is independent** - Once completed, it's final
2. **Commission is separate** - Calculated asynchronously
3. **Compensation strategy**:
   - If commission calculation fails → Mark as FAILED, retry later
   - If commission credit fails → Debit back if already credited
   - Settlement and reporting are eventually consistent

**Recovery:**
- Failed commissions are logged
- Batch job retries failed commissions
- Manual intervention for critical cases
- Audit trail for all commission operations"

### Key Takeaways for Interview

**MFS SAGA Pattern:**
- **Transaction Management**: Orchestration (strict order, money movement)
- **Commission Management**: Choreography (near real-time, event-driven)
- **Idempotency**: All operations must be idempotent
- **Compensation**: Reverse operations in exact order
- **Event Ordering**: Use Kafka partition keys (transactionId)
- **Recovery**: Saga state stored, timeout handling, retry mechanisms

**Design Principles:**
- **Critical paths** (money) → Orchestration
- **Non-critical paths** (commission) → Choreography
- **Async operations** → No compensation needed
- **Idempotency** → Safe to retry
- **Event ordering** → Partition keys for consistency

---

