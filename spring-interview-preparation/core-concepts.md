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

