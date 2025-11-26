# Architecture Patterns & Best Practices

## Quick Reference Snippets

---

## Layered Architecture

```
Controller Layer → Service Layer → Repository Layer → Database
```

---

## Design Patterns in Spring

```java
// Singleton Pattern (Default bean scope)
@Component
public class SingletonBean { }

// Factory Pattern
@Configuration
public class BeanFactory {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}

// Proxy Pattern (AOP)
@Transactional
public void method() { }
```

---

## Microservices Architecture

### Architecture Pattern: API Gateway Pattern

**Our Architecture:**
```
Client → Spring Cloud Gateway → Route Locator (Redis/PostgreSQL) 
      → Authorization Filter → Rate Limiter → Target Service (K8s)
```

**Components:**
1. **API Gateway** (Spring Cloud Gateway): Single entry point, reactive Netty server
2. **Service Discovery**: Kubernetes service discovery
3. **Route Locator**: Custom implementation using Redis cache + PostgreSQL
4. **Authorization**: Multi-level authorization (Public, Aggregator, External, Customer)
5. **Rate Limiting**: Request rate limiter with Redis
6. **Microservices**: Backend services running in Kubernetes

### Complete Request Flow

**Example Request:** `GET /api/v1/payment/transaction/123`

```
[1] Gateway Receives Request
    ↓
[2] Route Locator (ApiRouteManagerServiceImpl)
    - Check Redis: API_ROUTE:* (cached routes)
    - If miss → Load from PostgreSQL: gateway.API_ROUTE
    - Match path: '/api/v1/payment/**' → URI: 'http://172.25.2.2:32020'
    ↓
[3] Authorization Filter (Order: 1 - Highest Priority)
    ↓
[4] Request Inspection (RequestInspectionService)
    ├─ Step 4.1: Check Public API (PUBLIC_API_ACCESS)
    │  └─ If public → ✅ Allow directly
    │
    ├─ Step 4.2: Check Aggregator API (AGGREGATOR_API_ACCESS)
    │  └─ Validate JWT from 'token' header → ✅ Allow
    │
    ├─ Step 4.3: Check External API (EXTERNAL_API_ACCESS)
    │  └─ Validate encrypted token → ✅ Allow
    │
    └─ Step 4.4: Customer Authorization
       ├─ 4.4.1: Validate JWT Token (Authorization header)
       ├─ 4.4.2: Check Blocked APIs (USER_API_NOT_ACCESS)
       ├─ 4.4.3: Decrypt Session Data (Session-Data header)
       ├─ 4.4.4: Validate Timestamp
       ├─ 4.4.5: Validate Device Identity (DEVICE_IDENTIFIER)
       ├─ 4.4.6: Check API Access (USER_API_ACCESS)
       ├─ 4.4.7: Merchant Portal Permissions
       ├─ 4.4.8: Validate Session in Redis
       └─ 4.4.9: Create CurrentUserContext → Add to headers
    ↓
[5] Rate Limiting (RequestRateLimiter)
    - Check Redis: API_LIMITER:*
    - If miss → Load from PostgreSQL: gateway.API_LIMITER
    - Check request count within threshold
    - If exceeded → ❌ 429 Too Many Requests
    ↓
[6] Route Forwarding
    - Forward to target service: http://172.25.2.2:32020
    - Preserve headers (including Current-User-Context)
    ↓
[7] Target Service Processes Request
    ↓
[8] Response Returned to Client
```

### Implementation Details

**Route Locator (Custom):**
```java
// ApiRouteManagerServiceImpl
public class ApiRouteManagerServiceImpl {
    public List<Route> getRoutes() {
        // 1. Check Redis cache: API_ROUTE:*
        List<ApiRoute> routes = apiRouteService.findApiRoutesRedis();
        
        // 2. If cache miss → Load from PostgreSQL
        // Table: gateway.API_ROUTE
        // Columns: id, path, method, uri
        // Example: path='/api/v1/payment/**', uri='http://172.25.2.2:32020'
        
        // 3. Match request path against route patterns
        return buildRoutes(routes);
    }
}
```

**Authorization Filter:**
```java
@Order(1) // Highest priority
public class AuthorizationFilter implements GatewayFilter {
    
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 1. SecurityInterceptor.applySecurityIntercept()
        // 2. RequestInspectionService.inspectRequest()
        
        // Multi-level checks:
        // - Public API (PUBLIC_API_ACCESS from Redis/PostgreSQL)
        // - Aggregator API (AGGREGATOR_API_ACCESS + JWT validation)
        // - External API (EXTERNAL_API_ACCESS + encrypted token)
        // - Customer Authorization (JWT + Session + Device + Permissions)
        
        // Create CurrentUserContext → Add to headers
        return chain.filter(exchange);
    }
}
```

**Rate Limiting:**
```java
// RequestRateLimiter
public class RequestRateLimiter {
    // Redis Hash: API_LIMITER:*
    // If miss → PostgreSQL: gateway.API_LIMITER
    // Check: request count within threshold for TTL period
    // If exceeded → 429 Too Many Requests
}
```

### Service Discovery: Kubernetes

**Kubernetes Service Discovery:**
- Services registered as Kubernetes Services
- DNS-based discovery: `service-name.namespace.svc.cluster.local`
- Load balancing handled by Kubernetes Service
- Direct IP routing: `http://172.25.2.2:32020` (from API_ROUTE table)

### Interview Answer: Architecture & Request Flow

**Q: Which architecture are you using?**

**Answer:**
"We're using the **API Gateway Pattern** with Spring Cloud Gateway. Our architecture consists of:

1. **API Gateway** (Spring Cloud Gateway) - Reactive Netty server, single entry point
2. **Service Discovery** (Kubernetes) - Services registered as K8s Services
3. **Custom Route Locator** - Dynamic routing from Redis cache + PostgreSQL
4. **Multi-level Authorization** - Public, Aggregator, External, Customer authorization
5. **Rate Limiting** - Request rate limiter with Redis
6. **Microservices** - Backend services in Kubernetes pods

This provides:
- Single entry point with centralized security
- Dynamic routing configuration
- Multi-tenant authorization support
- Rate limiting and protection
- Kubernetes-native service discovery"

**Q: How does a request land to a particular service?**

**Answer:**
"Requests follow this flow:

1. **Client sends request** to Gateway (e.g., `GET /api/v1/payment/transaction/123`)

2. **Route Locator** finds target service:
   - Check Redis cache: `API_ROUTE:*`
   - If cache miss → Load from PostgreSQL: `gateway.API_ROUTE`
   - Match path pattern: `/api/v1/payment/**` → URI: `http://172.25.2.2:32020`

3. **Authorization Filter** (highest priority):
   - Check Public API → If public, allow
   - Check Aggregator API → Validate JWT from 'token' header
   - Check External API → Validate encrypted token
   - Customer Authorization:
     * Validate JWT token (Authorization header)
     * Check blocked APIs
     * Decrypt Session-Data header
     * Validate device identity
     * Check API access permissions
     * Validate session in Redis
   - Create `CurrentUserContext` → Add to headers

4. **Rate Limiting**:
   - Check Redis: `API_LIMITER:*`
   - Validate request count within threshold
   - If exceeded → 429 Too Many Requests

5. **Route Forwarding**:
   - Forward to target service URI (from route config)
   - Preserve all headers including `CurrentUserContext`

6. **Target Service** processes request and returns response

7. **Response** flows back through gateway to client

**Example:**
```
Client → Gateway (/api/v1/payment/transaction/123)
      → Route Locator (Redis/PostgreSQL) → URI: http://172.25.2.2:32020
      → Authorization (JWT + Session + Device validation)
      → Rate Limiter (check threshold)
      → Payment Service (172.25.2.2:32020)
      → Response back to client
```

**Key Points:**
- Routes stored in Redis (cache) + PostgreSQL (source of truth)
- Multi-level authorization for different client types
- Kubernetes service discovery for backend services
- Rate limiting protects against abuse

---

## Inter-Service Communication: User Context Management

### Current Approach: Base64 Encoded Header

**Implementation:**
```java
// Gateway creates CurrentUserContext
public class CurrentUserContext {
    private String userIdentity;
    private String userType;
    private String userStatus;
    private String userLevel;
    private String phoneNo;
    private String clientId;
    private String groupId;
    private String deviceId;
    // ... other fields
}

// Serialize to JSON, then Base64 encode
String json = objectMapper.writeValueAsString(context);
String base64Context = Base64.getEncoder().encodeToString(json.getBytes());

// Add to header
exchange.getRequest().mutate()
    .header("Current-User-Context", base64Context)
    .build();
```

**Service receives and decodes:**
```java
@RestController
public class PaymentController {
    
    @GetMapping("/transaction/{id}")
    public Transaction getTransaction(
            @PathVariable Long id,
            @RequestHeader("Current-User-Context") String contextHeader) {
        
        // Decode Base64
        String json = new String(Base64.getDecoder().decode(contextHeader));
        CurrentUserContext context = objectMapper.readValue(json, CurrentUserContext.class);
        
        // Use context
        String userId = context.getUserIdentity();
        // Process request...
    }
}
```

### Pros & Cons of Current Approach

**Pros:**
- ✅ Simple implementation
- ✅ Single header, easy to forward
- ✅ Contains all user info in one place
- ✅ No additional service calls needed

**Cons:**
- ❌ Header size limit (8KB typical)
- ❌ Base64 encoding increases size (~33% overhead)
- ❌ No validation/verification by receiving service
- ❌ Context can be tampered with
- ❌ Not standardized across services
- ❌ Hard to version/evolve

### Better Approaches

**1. JWT Token with Claims (Recommended)**

```java
// Gateway creates JWT with user claims
public String createUserContextJWT(CurrentUserContext context) {
    return Jwts.builder()
        .setSubject(context.getUserIdentity())
        .claim("userType", context.getUserType())
        .claim("userStatus", context.getUserStatus())
        .claim("userLevel", context.getUserLevel())
        .claim("phoneNo", context.getPhoneNo())
        .claim("clientId", context.getClientId())
        .claim("groupId", context.getGroupId())
        .claim("deviceId", context.getDeviceId())
        .setIssuedAt(new Date())
        .setExpiration(new Date(System.currentTimeMillis() + 3600000)) // 1 hour
        .signWith(SignatureAlgorithm.HS256, jwtSecretKey)
        .compact();
}

// Add to header
exchange.getRequest().mutate()
    .header("X-User-Context", jwtToken)
    .build();
```

**Service validates and extracts:**
```java
@RestController
public class PaymentController {
    
    @GetMapping("/transaction/{id}")
    public Transaction getTransaction(
            @PathVariable Long id,
            @RequestHeader("X-User-Context") String contextToken) {
        
        // Validate JWT
        Claims claims = Jwts.parser()
            .setSigningKey(jwtSecretKey)
            .parseClaimsJws(contextToken)
            .getBody();
        
        // Extract user info
        String userId = claims.getSubject();
        String userType = claims.get("userType", String.class);
        // Process request...
    }
}
```

**Benefits:**
- ✅ Tamper-proof (signed)
- ✅ Expiration support
- ✅ Standard format
- ✅ Can be validated by any service
- ✅ Smaller than Base64 JSON

**2. Service-to-Service Token (OAuth2 Client Credentials)**

```java
// Gateway gets service token
@Service
public class ServiceTokenService {
    
    public String getServiceToken() {
        // Request token from Auth Service
        OAuth2AccessToken token = oauth2RestTemplate
            .getAccessToken(clientCredentials, "service-to-service");
        return token.getValue();
    }
}

// Add both user context and service token
exchange.getRequest().mutate()
    .header("X-User-Context", userContextJWT)
    .header("Authorization", "Bearer " + serviceToken)
    .build();
```

**3. Context Service Pattern**

```java
// Gateway stores context in Redis with short-lived key
String contextKey = UUID.randomUUID().toString();
redisTemplate.opsForValue().set(
    "context:" + contextKey, 
    context, 
    Duration.ofMinutes(5)
);

// Pass only key in header
exchange.getRequest().mutate()
    .header("X-Context-Key", contextKey)
    .build();
```

**Service retrieves:**
```java
@GetMapping("/transaction/{id}")
public Transaction getTransaction(
        @PathVariable Long id,
        @RequestHeader("X-Context-Key") String contextKey) {
    
    // Get from Redis
    CurrentUserContext context = (CurrentUserContext) 
        redisTemplate.opsForValue().get("context:" + contextKey);
    
    // Use context
}
```

**4. Distributed Tracing Context (OpenTelemetry/Jaeger)**

```java
// Use distributed tracing context
Span span = tracer.nextSpan()
    .tag("user.id", context.getUserIdentity())
    .tag("user.type", context.getUserType())
    .start();

// Context automatically propagated via trace headers
// Services can access from trace context
```

### Recommended Approach: Hybrid (JWT + Context Service)

**Best Practice:**
```java
// Gateway Implementation
public class UserContextManager {
    
    public void addUserContext(ServerWebExchange exchange, CurrentUserContext context) {
        // 1. Create lightweight JWT with essential claims
        String jwt = createUserContextJWT(context);
        
        // 2. Store full context in Redis (if needed for complex scenarios)
        String contextKey = UUID.randomUUID().toString();
        redisTemplate.opsForValue().set(
            "context:" + contextKey,
            context,
            Duration.ofMinutes(5)
        );
        
        // 3. Add both to headers
        exchange.getRequest().mutate()
            .header("X-User-Context", jwt)  // Essential info in JWT
            .header("X-Context-Key", contextKey)  // Full context key (optional)
            .build();
    }
}
```

**Service Implementation:**
```java
@Component
public class UserContextResolver {
    
    public CurrentUserContext resolve(HttpServletRequest request) {
        String jwt = request.getHeader("X-User-Context");
        
        if (jwt != null) {
            // Validate and extract from JWT (fast path)
            Claims claims = validateJWT(jwt);
            return extractContextFromClaims(claims);
        }
        
        // Fallback: Get from Redis using context key
        String contextKey = request.getHeader("X-Context-Key");
        if (contextKey != null) {
            return (CurrentUserContext) redisTemplate
                .opsForValue().get("context:" + contextKey);
        }
        
        throw new UnauthorizedException("User context not found");
    }
}

// Usage in Controller
@GetMapping("/transaction/{id}")
public Transaction getTransaction(
        @PathVariable Long id,
        HttpServletRequest request) {
    
    CurrentUserContext context = userContextResolver.resolve(request);
    // Use context...
}
```

### Comparison Table

| Approach | Security | Performance | Scalability | Complexity |
|----------|----------|-------------|-------------|------------|
| **Base64 Header** | Low | High | Low | Low |
| **JWT Token** | High | High | High | Medium |
| **Context Service** | Medium | Medium | High | High |
| **Hybrid (JWT + Redis)** | High | High | High | Medium |

### Interview Answer: User Context Management

**Q: How do you manage user context in inter-service communication?**

**Answer:**
"Currently, we use a **Base64-encoded JSON header** (`Current-User-Context`) that contains all user information. The gateway creates this context after authentication and forwards it to backend services.

**Current Flow:**
1. Gateway validates JWT and extracts user info
2. Creates `CurrentUserContext` object with user details
3. Serializes to JSON, Base64 encodes
4. Adds to header: `Current-User-Context`
5. Services decode and extract user info

**Limitations:**
- Header size constraints
- No tamper protection
- Base64 overhead (~33% size increase)

**Better Approach (Recommended):**
We're moving to a **JWT-based approach** where:
- Gateway creates a signed JWT with user claims
- Services validate JWT signature (tamper-proof)
- Smaller size, standard format
- Can include expiration
- Works well with distributed systems

**Alternative:**
For complex scenarios, we could use a **Context Service pattern**:
- Store full context in Redis with short-lived key
- Pass only key in header
- Services retrieve from Redis
- Better for large contexts, but adds Redis dependency

**Best Practice:**
Hybrid approach - JWT for essential info, Redis key for full context when needed."

---

## Microservices vs Monolithic: Comparison

### Quick Overview

| Aspect | Monolithic | Microservices |
|--------|-----------|---------------|
| **Structure** | Single deployable unit | Multiple independent services |
| **Codebase** | Single codebase | Multiple codebases |
| **Database** | Shared database | Database per service |
| **Technology** | Single stack | Multiple stacks allowed |
| **Communication** | In-process calls | Network calls (HTTP/gRPC) |
| **Scaling** | Scale entire app | Scale individual services |
| **Deployment** | Single deployment | Independent deployments |

### Comprehensive Comparison

| Aspect | Monolithic | Microservices |
|--------|-----------|---------------|
| **Development** | Fast (initial), simple | Slower (setup), complex |
| **Debugging** | Simple (local) | Complex (distributed) |
| **Testing** | Unit + integration | + Contract + E2E tests |
| **Performance** | Fast (no network) | Slower (network latency) |
| **Scalability** | Scale all or nothing | Scale per service |
| **Cost** | Lower (fewer resources) | Higher (more infrastructure) |
| **Team Size** | Small teams (1-5) | Large teams (10+) |
| **Time to Market** | Faster (initial) | Slower (initial setup) |
| **Fault Isolation** | Failure affects all | Isolated to service |
| **Transaction** | ACID transactions | SAGA pattern |
| **Consistency** | Strong | Eventual |
| **Monitoring** | Single logs | Distributed tracing |
| **Service Discovery** | Not needed | Required |

### Pros and Cons

**Monolithic:**
- ✅ Simple, fast initial dev, better performance, lower cost, ACID transactions
- ❌ Scaling limitations, tech lock-in, single point of failure, slower as grows

**Microservices:**
- ✅ Independent scaling, fault isolation, tech diversity, independent deployment
- ❌ Complexity, network latency, distributed transactions, higher cost

### When to Use

**Monolithic:** Small team, simple app, fast time to market, limited budget, strong consistency needed

**Microservices:** Large team, complex app, different scaling needs, tech diversity, independent deployment

### Interview Answer

**Q: Microservices vs Monolithic - when to choose which?**

**Answer:**
"**Choose Microservices when:**
- Large team (10+ developers)
- Different scaling needs per component
- Technology diversity required
- Independent deployment needed
- High availability critical

**Choose Monolithic when:**
- Small team (1-5 developers)
- Simple application
- Fast time to market
- Limited budget
- Strong consistency required

**Best Practice:** Start with monolithic, migrate to microservices when team grows or complexity increases."

---

