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

## Spring Boot vs Spring Framework

### Quick Comparison

| Aspect | Spring Framework | Spring Boot |
|--------|------------------|-------------|
| **Purpose** | Core framework (DI, AOP, MVC) | Opinionated framework on top of Spring |
| **Configuration** | XML or Java config required | Auto-configuration (convention over configuration) |
| **Dependencies** | Manual dependency management | Starter dependencies (pre-configured) |
| **Embedded Server** | Not included | Tomcat/Jetty included |
| **Setup Time** | More configuration needed | Minimal configuration |
| **Production Ready** | Manual setup required | Built-in features (actuator, metrics) |
| **Deployment** | WAR file to external server | JAR file (standalone) |
| **Learning Curve** | Steeper | Easier |

---

## IoC Container (Inversion of Control)

### What is IoC Container?

**IoC Container** manages object creation, lifecycle, and dependency injection.

**Key Concepts:**
- **Inversion of Control**: Framework controls object creation (not developer)
- **Dependency Injection**: Dependencies provided by container
- **Bean**: Object managed by IoC container

### Dependency Injection

**Types of DI:**

**1. Constructor Injection (Recommended)**
```java
@Service
public class UserService {
    private final UserRepository repository;
    
    // Constructor injection
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

**2. Setter Injection**
```java
@Service
public class UserService {
    private UserRepository repository;
    
    @Autowired
    public void setRepository(UserRepository repository) {
        this.repository = repository;
    }
}
```

**3. Field Injection (Not Recommended)**
```java
@Service
public class UserService {
    @Autowired
    private UserRepository repository;
}
```

### Autowiring

**Autowiring Modes:**

**1. By Type (Default)**
```java
@Autowired
private UserRepository repository; // Matches by type
```

**2. By Name**
```java
@Autowired
@Qualifier("userRepositoryImpl")
private UserRepository repository; // Matches by bean name
```

**3. By Constructor (Recommended)**
```java
public UserService(UserRepository repository) {
    // Automatically autowired
}
```

**Autowiring Resolution:**
1. Match by type
2. If multiple candidates → use @Qualifier
3. If no match → throw exception

### Bean Lifecycle

**Bean Creation Phases:**

```
1. Instantiation (Constructor called)
   ↓
2. Populate Properties (@Autowired, @Value)
   ↓
3. BeanNameAware.setBeanName()
   ↓
4. BeanFactoryAware.setBeanFactory()
   ↓
5. ApplicationContextAware.setApplicationContext()
   ↓
6. @PostConstruct method
   ↓
7. InitializingBean.afterPropertiesSet()
   ↓
8. Custom init method (@Bean(initMethod = "init"))
   ↓
9. Bean Ready (In Use)
   ↓
10. @PreDestroy method
   ↓
11. DisposableBean.destroy()
   ↓
12. Custom destroy method (@Bean(destroyMethod = "cleanup"))
```

**Example:**
```java
@Component
public class UserService implements InitializingBean, DisposableBean {
    
    @PostConstruct
    public void init() {
        // Custom initialization
    }
    
    @Override
    public void afterPropertiesSet() {
        // InitializingBean method
    }
    
    @PreDestroy
    public void cleanup() {
        // Custom cleanup
    }
    
    @Override
    public void destroy() {
        // DisposableBean method
    }
}
```

### Bean Declaration Ways

**1. Component Scanning (@Component, @Service, @Repository, @Controller)**
```java
@Component
public class UserService { }

@Service
public class UserService { }

@Repository
public class UserRepository { }

@Controller
public class UserController { }
```

**2. @Bean Method (in @Configuration class)**
```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}
```

**3. @Import (Import other configurations)**
```java
@Configuration
@Import({DatabaseConfig.class, SecurityConfig.class})
public class AppConfig { }
```

**4. XML Configuration (Legacy)**
```xml
<bean id="userService" class="com.example.UserService"/>
```

**5. Programmatic Registration**
```java
@Configuration
public class AppConfig {
    @Bean
    public BeanDefinitionRegistryPostProcessor customBean() {
        return registry -> {
            // Register beans programmatically
        };
    }
}
```

### Bean Scopes

| Scope | Description | Use Case |
|-------|-------------|----------|
| **singleton** | One instance per container (default) | Stateless services |
| **prototype** | New instance each time | Stateful beans |
| **request** | One per HTTP request | Web controllers |
| **session** | One per HTTP session | User session data |
| **application** | One per ServletContext | Web application context |

---

## Reactive Programming in Spring

### What is Reactive Programming?

**Reactive Programming** = Asynchronous, non-blocking, event-driven programming

**Key Principles:**
- **Non-blocking**: Threads don't wait for I/O
- **Asynchronous**: Operations execute concurrently
- **Backpressure**: Handle data flow rate
- **Event-driven**: React to events/streams

### Spring WebFlux (Reactive)

**Traditional (Blocking):**
```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        // Blocking call - thread waits
        return userRepository.findById(id);
    }
}
```

**Reactive (Non-blocking):**
```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        // Non-blocking - thread free for other requests
        return userRepository.findById(id);
    }
}
```

### Reactive Types

**1. Mono (0 or 1 element)**
```java
Mono<String> mono = Mono.just("Hello");
Mono<User> user = userRepository.findById(id);
```

**2. Flux (0 to N elements)**
```java
Flux<String> flux = Flux.just("A", "B", "C");
Flux<User> users = userRepository.findAll();
```

### Multithreading in Spring Reactive

**Threading Model:**

**1. Event Loop (Non-blocking)**
```
Request → Event Loop Thread
         ↓
    Non-blocking I/O
         ↓
    Callback executed
         ↓
    Response sent
```

**2. Thread Pool (Blocking operations)**
```java
// Switch to thread pool for blocking operations
Mono.fromCallable(() -> {
    // Blocking operation
    return blockingDatabaseCall();
})
.subscribeOn(Schedulers.boundedElastic());
```

**Schedulers:**

| Scheduler | Use Case |
|-----------|----------|
| **Schedulers.immediate()** | Current thread |
| **Schedulers.single()** | Single worker thread |
| **Schedulers.parallel()** | Parallel execution |
| **Schedulers.boundedElastic()** | Blocking operations |
| **Schedulers.elastic()** | Unbounded thread pool |

**Example:**
```java
@RestController
public class UserController {
    
    @GetMapping("/users/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        // Non-blocking reactive call
        return userRepository.findById(id)
            .subscribeOn(Schedulers.boundedElastic());
    }
    
    @GetMapping("/users")
    public Flux<User> getAllUsers() {
        // Non-blocking stream
        return userRepository.findAll()
            .delayElements(Duration.ofMillis(100));
    }
}
```

### Reactive vs Traditional

| Aspect | Traditional (Blocking) | Reactive (Non-blocking) |
|--------|----------------------|------------------------|
| **Thread Model** | One thread per request | Event loop (few threads) |
| **Blocking** | Thread waits for I/O | Thread continues other work |
| **Scalability** | Limited by threads | High concurrency |
| **Resource Usage** | High (many threads) | Low (few threads) |
| **Complexity** | Simpler | More complex |
| **Use Case** | CRUD, simple APIs | High throughput, streaming |

### Interview Answer

**Q: What is IoC Container and Dependency Injection?**

**Answer:**
"IoC (Inversion of Control) Container manages object lifecycle and dependencies. Instead of creating objects manually, the container:
- Creates objects (beans)
- Manages dependencies
- Controls lifecycle

**Dependency Injection** is providing dependencies to objects rather than objects creating them. Types:
- Constructor injection (recommended)
- Setter injection
- Field injection

**Autowiring** automatically injects dependencies by type or name. Spring resolves dependencies and injects them automatically."

**Q: Explain Bean Lifecycle.**

**Answer:**
"Bean lifecycle phases:
1. **Instantiation**: Constructor called
2. **Property Population**: @Autowired, @Value injected
3. **Aware Interfaces**: BeanNameAware, BeanFactoryAware
4. **@PostConstruct**: Custom initialization
5. **InitializingBean**: afterPropertiesSet()
6. **Bean Ready**: Available for use
7. **@PreDestroy**: Cleanup before destruction
8. **DisposableBean**: destroy() method

Bean can be declared via: @Component, @Bean, @Import, XML, or programmatic registration."

**Q: What is Reactive Programming in Spring?**

**Answer:**
"Reactive Programming is asynchronous, non-blocking, event-driven programming.

**Spring WebFlux** provides reactive support:
- **Mono**: 0 or 1 element
- **Flux**: 0 to N elements
- **Non-blocking**: Threads don't wait for I/O
- **Event loop**: Few threads handle many requests

**Multithreading:**
- Event loop for non-blocking operations
- Thread pools (Schedulers) for blocking operations
- High concurrency with fewer threads

**Use when**: High throughput, streaming data, many concurrent requests."

### Key Takeaways

**IoC Container:**
- Manages object creation and dependencies
- Dependency Injection: Constructor (best), Setter, Field
- Autowiring: By type or @Qualifier by name
- Bean Lifecycle: Instantiation → Property Population → Initialization → Ready → Destruction

**Reactive Programming:**
- Non-blocking, asynchronous, event-driven
- Mono (0-1 element), Flux (0-N elements)
- Event loop model (few threads, high concurrency)
- Use for high throughput, streaming, many concurrent requests

---
