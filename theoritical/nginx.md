# Nginx — High-Level Brief & Interview Prep (Node.js Focus)

> Tailored for a backend engineer with microservices experience (Spring Boot / FinTech) transitioning into or interviewing for Node.js roles.

---

## High-Level Brief

**Nginx** (engine-x) is a high-performance **web server, reverse proxy, load balancer, and HTTP cache**. In Node.js ecosystems it almost always sits **in front of** the Node process as a reverse proxy.

```
Internet → Nginx (:80/:443) → Node.js app (:3000)
```

**Why not expose Node.js directly?**
- Node's HTTP server is single-threaded per core — not optimized for SSL handshakes, static files, or slow clients.
- Nginx handles these efficiently in C, freeing Node to do application logic.
- Single entry point for multiple services (API gateway pattern — same thing you do with 24+ Spring Boot services behind a gateway).

**Core roles in a Node.js deployment:**

| Role | What Nginx does |
|---|---|
| Reverse proxy | Forwards requests to Node, hides internal port |
| Load balancer | Distributes traffic across Node instances/pods |
| SSL termination | Handles HTTPS; Node receives plain HTTP internally |
| Static file server | Serves `/public` assets without hitting Node |
| Rate limiter | Protects Node from abuse at the network layer |
| Gzip / Brotli | Compresses responses before sending to client |
| WebSocket proxy | Upgrades HTTP connection to WS and tunnels it |
| Health check gateway | Only routes to healthy upstreams |

---

## Core nginx.conf Structure

```nginx
# /etc/nginx/nginx.conf (simplified)

events {
    worker_connections 1024;  # per worker process
}

http {
    # Upstream = your Node.js instances (same concept as Spring Boot service pool)
    upstream node_app {
        least_conn;                    # load balancing algorithm
        server 127.0.0.1:3000;
        server 127.0.0.1:3001;
        keepalive 32;                  # reuse connections to Node
    }

    server {
        listen 80;
        server_name example.com;
        return 301 https://$host$request_uri;  # force HTTPS
    }

    server {
        listen 443 ssl;
        server_name example.com;

        ssl_certificate     /etc/ssl/cert.pem;
        ssl_certificate_key /etc/ssl/key.pem;

        # Static files — Node never sees these requests
        location /static/ {
            root /var/www;
            expires 30d;
            add_header Cache-Control "public";
        }

        # Proxy everything else to Node
        location / {
            proxy_pass http://node_app;

            # Critical headers — Node reads these to get real client IP
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_http_version 1.1;
            proxy_set_header Upgrade    $http_upgrade;   # WebSocket support
            proxy_set_header Connection "upgrade";

            proxy_read_timeout  60s;
            proxy_connect_timeout 10s;
        }
    }
}
```

---

## Most Frequently Asked Questions (Node.js Interviews)

---

### Q1. Why do we use Nginx in front of a Node.js application?

Node.js is great at application logic but not optimized for:
- SSL/TLS termination (expensive crypto in JS)
- Serving thousands of slow/idle HTTP connections (C10K problem)
- Serving static files (disk I/O)
- DDoS / rate limiting at scale

Nginx handles all of these in C, asynchronously, with very low memory per connection. It also lets you:
- Run multiple Node processes and load balance them (since Node is single-threaded per core).
- Deploy zero-downtime updates by swapping upstreams.
- Centralize logging, SSL certs, and security headers.

*Relate to your experience:* same pattern as putting an API gateway or load balancer in front of your 24 Spring Boot microservices.

---

### Q2. What is a reverse proxy and how does it differ from a forward proxy?

| | Forward Proxy | Reverse Proxy |
|---|---|---|
| Sits in front of | Clients | Servers |
| Client knows? | Yes (configured) | No (transparent) |
| Use case | Bypass restrictions, anonymize client | Load balance, SSL termination, hide server |
| Example | Corporate proxy | Nginx in front of Node |

Nginx as a reverse proxy: client talks to Nginx, Nginx talks to Node. Client never knows Node's port/IP.

---

### Q3. How does Nginx load balance multiple Node.js instances?

```nginx
upstream node_app {
    # Algorithms:
    # round_robin (default) — requests distributed evenly
    # least_conn             — to the instance with fewest active connections
    # ip_hash               — same client always goes to same instance (sticky session)
    # hash $request_uri     — same URL always goes to same instance

    least_conn;
    server 127.0.0.1:3000 weight=3;  # gets 3x traffic
    server 127.0.0.1:3001 weight=1;
    server 127.0.0.1:3002 backup;    # only used when others fail
}
```

**Why this matters for Node.js:**
- Node.js is single-threaded, so you run one process per CPU core (via PM2 cluster or multiple containers).
- Nginx distributes traffic across those processes.
- `least_conn` is recommended over round-robin for APIs with varying response times.

**`ip_hash` caveat:** breaks if you have many users behind NAT (all get same IP → same server). Prefer sticky sessions via Redis-backed sessions instead.

---

### Q4. How does Nginx handle WebSocket connections with Node.js?

WebSockets require HTTP Upgrade. Without special config, Nginx drops the connection.

```nginx
location /socket.io/ {
    proxy_pass http://node_app;
    proxy_http_version 1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host       $host;
    proxy_read_timeout 3600s;   # keep WS alive; default 60s would kill it
}
```

- `proxy_http_version 1.1` — HTTP/1.0 doesn't support keep-alive/upgrade.
- `Connection "upgrade"` — tells Node this is a protocol upgrade, not a normal request.
- Long `proxy_read_timeout` — prevents Nginx from killing idle WebSocket connections.

---

### Q5. What is SSL termination and why terminate at Nginx instead of Node?

**SSL termination** = decrypting HTTPS traffic and forwarding plain HTTP to the backend.

```
Client → HTTPS → Nginx (decrypts) → HTTP → Node.js
```

Reasons to terminate at Nginx:
1. **Performance** — OpenSSL in C is far faster than Node's TLS.
2. **Cert management** — one place to renew/rotate certs (Let's Encrypt / Certbot).
3. **Simplified Node code** — Node just handles plain HTTP, no `https.createServer()`.
4. **Internal network trust** — Node behind Nginx on the same host/VPC doesn't need its own cert.

In Node.js, check `req.headers['x-forwarded-proto'] === 'https'` to know the original request was HTTPS.

---

### Q6. How do you get the real client IP in Node.js when behind Nginx?

Nginx adds headers before forwarding:
```nginx
proxy_set_header X-Real-IP       $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

In Node.js (Express):
```javascript
// Trust proxy — required for req.ip to work correctly
app.set('trust proxy', 1);

app.get('/', (req, res) => {
    const ip = req.ip;                          // uses X-Forwarded-For
    const rawIp = req.headers['x-real-ip'];    // direct header
});
```

`trust proxy` tells Express to use `X-Forwarded-For` instead of the socket's IP (which would be Nginx's IP, 127.0.0.1).

**Security note:** Only trust `X-Forwarded-For` from Nginx (your trusted proxy). Without `trust proxy`, a malicious client could spoof the header.

---

### Q7. How do you implement rate limiting in Nginx for a Node.js API?

```nginx
http {
    # Define a rate limit zone: 10MB memory, keyed by client IP, 10 req/sec
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            # burst=20: allow up to 20 queued requests above the rate
            # nodelay: don't queue them — return 429 immediately if burst exceeded
            proxy_pass http://node_app;
        }
    }
}
```

- **Rate limiting at Nginx** is much more efficient than in Node — rejects before Node process is even touched.
- Combine with application-level rate limiting (express-rate-limit + Redis) for per-user/token limits.
- `$binary_remote_addr` uses 4 bytes (vs 7-15 for `$remote_addr`) — saves memory in the zone.

---

### Q8. How does Nginx serve static files and why offload this from Node?

```nginx
location /static/ {
    root /var/www/myapp;
    # File served from: /var/www/myapp/static/...
    gzip_static on;          # serve pre-compressed .gz files
    expires 1y;              # aggressive caching for hashed filenames
    add_header Cache-Control "public, immutable";
}

# Or alias for SPA (React/Next.js build output)
location / {
    root /var/www/myapp/build;
    try_files $uri $uri/ /index.html;  # client-side routing fallback
}
```

**Why not let Node serve static files?**
- Node reads file, allocates Buffer, writes to socket — synchronous disk I/O blocks the event loop.
- Nginx uses `sendfile()` syscall — kernel copies file directly to socket, zero copy, no userspace allocation.
- For a FinTech app serving thousands of users, this is significant: static files saturating Node = slower API responses.

---

### Q9. What are proxy timeouts and why do they matter for Node.js APIs?

```nginx
location /api/ {
    proxy_connect_timeout  10s;   # time to establish connection to Node
    proxy_send_timeout     30s;   # time to send request to Node
    proxy_read_timeout     60s;   # time to wait for Node's response
}
```

- If a Node endpoint does heavy DB work (e.g., a reconciliation query), `proxy_read_timeout` must be long enough or Nginx sends a 504 to the client before Node replies.
- For long-polling or SSE (Server-Sent Events), set `proxy_read_timeout` to a large value (e.g., `3600s`).
- `proxy_connect_timeout` should be short — if Node is down, fail fast.

---

### Q10. What is the difference between `location /` and `location /api/`? How does location matching work?

Nginx uses priority-based matching:

| Modifier | Example | Priority | Description |
|---|---|---|---|
| `=` | `location = /ping` | 1st (exact) | Exact match only |
| `^~` | `location ^~ /static/` | 2nd | Prefix match, stop regex search |
| `~` | `location ~ \.php$` | 3rd | Case-sensitive regex |
| `~*` | `location ~* \.jpg$` | 3rd | Case-insensitive regex |
| (none) | `location /api/` | 4th | Longest prefix match |

```nginx
location = /health {
    return 200 "ok";          # handled entirely by Nginx, Node not hit
}

location /static/ {
    root /var/www;            # serve files directly
}

location / {
    proxy_pass http://node_app;  # everything else → Node
}
```

---

### Q11. How do you configure Nginx for a Node.js microservices setup?

*This is directly relevant to your 24-service architecture.*

```nginx
upstream auth_service    { server auth:3001; }
upstream payment_service { server payment:3002; }
upstream wallet_service  { server wallet:3003; }

server {
    listen 443 ssl;

    location /api/auth/    { proxy_pass http://auth_service; }
    location /api/payment/ { proxy_pass http://payment_service; }
    location /api/wallet/  { proxy_pass http://wallet_service; }

    # Or use path rewriting
    location /api/payment/ {
        rewrite ^/api/payment/(.*)$ /$1 break;  # strip prefix before forwarding
        proxy_pass http://payment_service;
    }
}
```

Nginx acts as an **API gateway** — single entry point, routes to appropriate microservice. Same pattern as a Spring Cloud Gateway or Kong in a Spring Boot world.

---

### Q12. What are upstream keepalive connections and why are they important?

```nginx
upstream node_app {
    server 127.0.0.1:3000;
    keepalive 32;  # keep 32 idle connections open to Node
}

location / {
    proxy_http_version 1.1;
    proxy_set_header Connection "";  # clear Connection header to enable keepalive
    proxy_pass http://node_app;
}
```

Without keepalive: every request creates a new TCP connection to Node (3-way handshake each time).
With keepalive: connections are reused → lower latency, less CPU on both Nginx and Node.

For high-throughput FinTech APIs (your 20M+ BDT daily transactions scenario), this is critical.

---

### Q13. How does Nginx gzip compression work with Node.js?

```nginx
http {
    gzip on;
    gzip_types text/plain application/json application/javascript text/css;
    gzip_min_length 1024;     # only compress responses > 1KB
    gzip_comp_level 6;        # 1 (fast) to 9 (best compression); 6 is sweet spot
    gzip_vary on;             # add Vary: Accept-Encoding header
    gzip_proxied any;         # compress even proxied responses
}
```

- Node.js's `compression` middleware also gzips, but it runs in JS — slower than Nginx's C implementation.
- If using Nginx gzip, disable Node-level compression to avoid double compression.
- For static assets: pre-compress with `gzip_static on` and ship `.js.gz` files alongside `.js`.

---

### Q14. What security headers should you add in Nginx for a Node.js app?

```nginx
server {
    add_header X-Frame-Options           "SAMEORIGIN"         always;
    add_header X-Content-Type-Options    "nosniff"            always;
    add_header X-XSS-Protection          "1; mode=block"      always;
    add_header Referrer-Policy           "strict-origin"      always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header Content-Security-Policy   "default-src 'self'" always;

    # Hide Nginx version from error pages
    server_tokens off;
}
```

Adding these at Nginx means every response from every upstream (Node, or any service) gets them automatically — no need to set in Express for each route.

---

### Q15. How do you do zero-downtime deployment of a Node.js app with Nginx?

**Strategy 1: Reload Nginx config (no connection drop)**
```bash
nginx -t && nginx -s reload   # test config, then graceful reload
```
Nginx gracefully finishes existing requests then applies new config. Use when only changing routes/upstreams.

**Strategy 2: Blue-Green with upstream swap**
1. New Node version starts on `:3001`, old running on `:3000`.
2. Health check passes on `:3001`.
3. Update upstream to point to `:3001`, reload Nginx.
4. Drain `:3000`, shut it down.

```nginx
upstream node_app {
    server 127.0.0.1:3001;          # new version
    server 127.0.0.1:3000 down;     # mark old as down (graceful)
}
```

**Strategy 3: PM2 + Nginx**
```bash
pm2 reload app --update-env   # PM2 restarts workers one by one; Nginx keepalive handles in-flight requests
```

---

### Q16. How does `try_files` work? Common use case with React/Next.js

```nginx
location / {
    root /var/www/app/build;
    try_files $uri $uri/ /index.html;
}
```

- Try to serve `$uri` as a file (e.g., `/logo.png`).
- Try `$uri/` as a directory with index (e.g., `/about/index.html`).
- Fall back to `/index.html` — lets React Router handle the route on the client side.

Without this: refreshing `/dashboard` returns 404 because Nginx looks for `/dashboard` as a file, which doesn't exist.

---

### Q17. What is `proxy_buffering` and should you disable it for Node.js SSE/streaming?

```nginx
location /stream/ {
    proxy_pass http://node_app;
    proxy_buffering off;           # REQUIRED for SSE / chunked streaming
    proxy_cache off;
    proxy_set_header Cache-Control "no-cache";
    proxy_read_timeout 3600s;
}
```

By default `proxy_buffering on` — Nginx buffers the entire response before sending to client. This **breaks** Server-Sent Events and streaming responses because data never reaches the client until Node closes the connection.

Disable buffering for:
- SSE (Server-Sent Events)
- Chunked transfer encoding
- Long-polling endpoints
- Real-time log streaming

---

## Quick-Reference Cheat Sheet

```
nginx -t               # test config syntax
nginx -s reload        # graceful reload (zero downtime)
nginx -s stop          # fast shutdown
nginx -s quit          # graceful shutdown
tail -f /var/log/nginx/error.log
tail -f /var/log/nginx/access.log

# Check which config file is loaded
nginx -T | grep "configuration file"
```

**Common HTTP status codes from Nginx:**
- `502 Bad Gateway` — Nginx reached Node but got an invalid response (Node crashed/restarting).
- `504 Gateway Timeout` — Node took longer than `proxy_read_timeout`.
- `503 Service Unavailable` — all upstream servers are down or at capacity.
- `499 Client Closed Request` — client disconnected before Node responded (check slow endpoints).

---

## How This Fits Your FinTech Microservices Background

| Your Spring Boot Experience | Nginx Equivalent |
|---|---|
| Spring Cloud Gateway / API Gateway | Nginx upstream + location routing |
| Ribbon load balancer | Nginx upstream with least_conn |
| SSL via cert in Spring app | SSL termination at Nginx |
| Spring Security rate limiting | `limit_req_zone` in Nginx |
| Eureka health checks | Nginx `health_check` (nginx plus) / passive checks |
| Docker service names in compose | Upstream `server service-name:port` |

In a Node.js microservices interview, knowing Nginx means you can explain the full request lifecycle from browser → Nginx → Node service → DB and back — which is exactly the kind of system design depth a senior engineer is expected to have.