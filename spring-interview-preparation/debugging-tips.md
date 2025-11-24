# Debugging & Troubleshooting

## Quick Reference Snippets

---

## Logging Configuration

```yaml
logging:
  level:
    com.example: DEBUG
    org.springframework: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
```

---

## Common Debugging Commands

```bash
# Actuator endpoints
/actuator/health
/actuator/info
/actuator/metrics
/actuator/env
```

---

## Exception Handling

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(CustomException.class)
    public ResponseEntity<ErrorResponse> handleException(CustomException e) {
        // Error handling
    }
}
```

---

