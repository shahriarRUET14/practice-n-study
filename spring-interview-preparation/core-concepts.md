# Core Spring Boot Concepts

## Quick Reference Snippets

> **Note:** Detailed explanations available in chat. This file contains key snippets only.

---

## Spring Boot Auto-Configuration

```java
@SpringBootApplication
// Equivalent to:
// @Configuration + @EnableAutoConfiguration + @ComponentScan
```

---

## Dependency Injection Types

```java
// Constructor Injection (Recommended)
@RequiredArgsConstructor
public class UserService {
    private final UserRepository repository;
}

// Field Injection (Not recommended)
@Autowired
private UserRepository repository;

// Setter Injection
@Autowired
public void setRepository(UserRepository repository) {
    this.repository = repository;
}
```

---

## Bean Scopes

```java
@Scope("singleton")  // Default
@Scope("prototype")
@Scope("request")   // Web
@Scope("session")   // Web
```

---

## Configuration Properties

```java
@ConfigurationProperties(prefix = "app")
@Data
public class AppConfig {
    private String name;
    private int timeout;
}
```

---

## Common Spring Annotations: Quick Reference

### Core Annotations (Most Used)

**@Component / @Service / @Repository / @Controller**
- **Purpose:** Mark classes as Spring-managed beans
- **Difference:** Semantic (Service = business logic, Repository = data access, Controller = web layer)

**@Autowired**
- **Purpose:** Dependency injection
- **Usage:** Field, constructor, or setter injection

**@Value**
- **Purpose:** Inject property values
- **Example:** `@Value("${app.name}")` or `@Value("#{systemProperties['user.name']}")`

**@Transactional**
- **Purpose:** Declare transactional methods
- **Critical:** Manages database transactions

### Web/MVC Annotations (REST APIs)

**@RestController**
- **Purpose:** REST controller (returns JSON/XML)
- **Equivalent:** @Controller + @ResponseBody

**@GetMapping / @PostMapping / @PutMapping / @DeleteMapping**
- **Purpose:** Map HTTP methods to handler methods
- **Shortcuts for:** @RequestMapping

**@PathVariable**
- **Purpose:** Extract URL path variables
- **Example:** `/users/{id}` → `@PathVariable Long id`

**@RequestParam**
- **Purpose:** Extract query parameters
- **Example:** `?name=John` → `@RequestParam String name`

**@RequestBody**
- **Purpose:** Bind HTTP request body to object
- **Converts:** JSON → Java object

**@ResponseBody**
- **Purpose:** Convert return value to HTTP response
- **Converts:** Java object → JSON

### Data/JPA Annotations

**@Entity / @Table / @Id / @GeneratedValue**
- **Purpose:** JPA entity mapping
- **Maps:** Java class to database table

**@OneToMany / @ManyToOne / @OneToOne / @ManyToMany**
- **Purpose:** Define relationships between entities
- **Critical for:** ORM

**@Transactional**
- **Purpose:** Database transaction management
- **Options:** Propagation, isolation, timeout

### Validation Annotations

**@Valid / @NotNull / @NotEmpty / @NotBlank / @Size / @Email**
- **Purpose:** Bean validation
- **Validates:** Input data

### Configuration Annotations

**@ConfigurationProperties**
- **Purpose:** Type-safe configuration binding
- **Binds:** application.properties to POJO

**@Profile**
- **Purpose:** Environment-specific configuration
- **Example:** `@Profile("dev")`, `@Profile("prod")`

### AOP Annotations

**@Aspect / @Before / @After / @Around**
- **Purpose:** Aspect-oriented programming
- **Cross-cutting concerns:** Logging, security, transactions

### Caching Annotations

**@Cacheable / @CacheEvict / @CachePut**
- **Purpose:** Method result caching
- **Improves:** Performance

### Scheduling Annotations

**@Scheduled**
- **Purpose:** Scheduled task execution
- **Example:** `@Scheduled(fixedRate = 5000)`

### Messaging Annotations

**@KafkaListener / @RabbitListener**
- **Purpose:** Listen to message queues
- **Event-driven:** Architecture

### Most Important Annotations for Interview

**Top 10:**
1. **@SpringBootApplication** - Main application entry
2. **@RestController** - REST API controllers
3. **@Autowired** - Dependency injection
4. **@Service / @Repository** - Layer stereotypes
5. **@Transactional** - Transaction management
6. **@GetMapping / @PostMapping** - HTTP method mapping
7. **@PathVariable / @RequestParam** - Request parameter binding
8. **@RequestBody / @ResponseBody** - Request/response handling
9. **@Entity / @Id** - JPA entity mapping
10. **@Valid** - Input validation

### Interview Tips

**Q: What's the difference between @Component, @Service, and @Repository?**

**Answer:**
"All three are specializations of @Component:
- **@Component**: Generic Spring-managed bean
- **@Service**: Business logic layer (semantic)
- **@Repository**: Data access layer (enables exception translation)
- **@Controller**: Web layer (handles HTTP requests)

Functionally similar, but provide semantic meaning and enable specific features."

**Q: When to use @Autowired vs Constructor Injection?**

**Answer:**
"Constructor injection is recommended because:
- Immutable dependencies (final fields)
- Easier testing (can pass mocks)
- Fail-fast (null check at construction)
- No reflection needed

Field injection (@Autowired on field) is not recommended:
- Hard to test
- Hidden dependencies
- Can't make fields final"

**Q: What does @Transactional do?**

**Answer:**
"@Transactional manages database transactions:
- Wraps method in transaction
- Commits on success, rolls back on exception
- Configurable: propagation, isolation, timeout, readOnly
- Works with JPA/Hibernate

Example:
```java
@Transactional(rollbackFor = Exception.class)
public void transferMoney(Account from, Account to, BigDecimal amount) {
    // All DB operations in one transaction
}
```"

---

