# 🔗 Simple URL Shortener — System Design

A scalable system design for a **URL Shortener** that converts long URLs into short, unique URLs and redirects users efficiently using caching, sharding, and horizontally scalable services.

## 📌 Overview

The system provides two main operations:

* **Create a short URL** from a long URL.
* **Redirect** users from the short URL to the original long URL.

The design focuses on high availability, low redirect latency, scalability, data durability, and safe retries.

---

## 🏗️ Architecture

The system follows this high-level flow:

```text
Client
  │
  ▼
DNS + CDN
  │
  ▼
WAF + Rate Limiter
  │
  ▼
L7 Load Balancer
  │
  ├───────────────┐
  ▼               ▼
Create Service   Redirect Service
  │               │
  ▼               ▼
Key Generator    Redis Cluster
  │               │
  ▼               ▼
Shard Router ◄────┘
  │
  ▼
Sharded URL Mapping Store
```

### Main Components

| Component          | Responsibility                                    |
| ------------------ | ------------------------------------------------- |
| DNS + CDN          | Routes traffic and caches hot redirect responses  |
| WAF + Rate Limiter | TLS, DDoS protection, validation and quotas       |
| L7 Load Balancer   | Routes create and redirect requests               |
| Create Service     | Validates URLs and creates mappings               |
| Redirect Service   | Looks up codes and returns HTTP 302               |
| Key Generator      | Generates unique 8-character Base62 codes         |
| Redis Cluster      | Caches `code → long URL` mappings                 |
| Shard Router       | Routes database requests using consistent hashing |
| URL Mapping Store  | Durable storage for URL mappings                  |

---

## ⚙️ Functional Requirements

1. Accept and validate an HTTP/HTTPS long URL.
2. Generate a unique, opaque **8-character code**.
3. Persist the `code → long URL` mapping before returning success.
4. Return the complete short URL.
5. Redirect known codes using **HTTP 302**.
6. Return **404 Not Found** for unknown codes.
7. Support `Idempotency-Key` to make creation retries safe.

---

## 🚀 Non-Functional Requirements

* Redirect p99 latency below **100 ms** on a regional cache hit.
* Creation p99 latency below **300 ms**.
* Target **99.99% redirect availability**.
* No acknowledged mapping should be lost.
* Immediate read-after-write consistency after URL creation.
* Horizontally scalable stateless services.
* TLS and rate limiting.
* DDoS and abuse protection.
* Monitoring, metrics and distributed tracing.
* Backups and tested failover.

---

## 🔌 API Endpoints

### Create Short URL

```http
POST /v1/urls
```

#### Request

```http
Idempotency-Key: optional-key
Content-Type: application/json
```

```json
{
  "longUrl": "https://example.com/path"
}
```

#### Response

```http
HTTP/1.1 201 Created
```

```json
{
  "code": "g7K2pQ9x",
  "shortUrl": "https://sho.rt/g7K2pQ9x"
}
```

### Possible Errors

```text
400 Bad Request  → Invalid URL
413 Payload Too Large
429 Too Many Requests
503 Service Unavailable
```

---

### Redirect

```http
GET /{code}
```

Example:

```http
GET /g7K2pQ9x
```

Response:

```http
HTTP/1.1 302 Found
Location: https://example.com/path
Cache-Control: public, max-age=300
```

Unknown code:

```http
404 Not Found
```

Disabled or expired URL:

```http
410 Gone
```

---

### Optional HEAD Request

```http
HEAD /{code}
```

Can be used as a lightweight existence or redirect check.

---

## 🗄️ Database Design

### URL_MAPPING

```text
URL_MAPPING
────────────────────────
code          CHAR(8)       PRIMARY KEY
long_url      TEXT          NOT NULL
created_at    TIMESTAMP     NOT NULL
status        SMALLINT      DEFAULT 1
url_hash      BINARY        optional
```

### Important Invariant

```text
One short code → One long URL
```

The database is sharded using:

```text
hash(code)
```

---

### IDEMPOTENCY_RECORD

```text
IDEMPOTENCY_RECORD
────────────────────────
idempotency_key   VARCHAR     PRIMARY KEY
code              CHAR(8)
request_hash      BINARY
created_at        TIMESTAMP
expires_at        TIMESTAMP
```

This prevents duplicate short URLs when a client retries the same creation request.

---

### Key Pool

The key generator uses **8-character Base62 codes**.

```text
unused → leased → used
```

The system uses:

* Atomic batch leases
* `UNIQUE(code)` constraint
* Pre-generated codes

The total theoretical key space is:

```text
62⁸ ≈ 218 trillion keys
```

---

## ⚡ Caching Strategy

Redis stores frequently accessed mappings:

```text
shortCode → longURL
```

Example:

```text
g7K2pQ9x → https://example.com/path
```

### Redis Features

* Replicated Redis cluster
* TTL with jitter
* Cache-aside lookup
* Short negative caching
* Cache warming after URL creation

Target:

```text
Cache hit rate ≥ 95%
```

If Redis fails, the Redirect Service falls back to the durable database.

---

## 🔄 Create URL Flow

```text
Client
   │
   ▼
POST /v1/urls
   │
   ▼
WAF + Rate Limiter
   │
   ▼
Load Balancer
   │
   ▼
Create Service
   │
   ├── Validate URL
   │
   ├── Check Idempotency-Key
   │
   ├── Obtain unique code
   │
   ├── Commit mapping to database
   │
   ├── Warm Redis cache
   │
   ▼
201 Created
```

The important ordering is:

```text
Generate code
     ↓
Commit mapping
     ↓
Warm Redis
     ↓
Return 201
```

This ensures an acknowledged URL mapping is durable before success is returned.

---

## 🔀 Redirect Flow

```text
Client
   │
   ▼
GET /{code}
   │
   ▼
DNS + CDN
   │
   ├── Cache Hit ──→ 302 Redirect
   │
   ▼
WAF + Rate Limiter
   │
   ▼
Load Balancer
   │
   ▼
Redirect Service
   │
   ▼
Redis
   │
   ├── Cache Hit ──→ 302 Redirect
   │
   ▼
Shard Router
   │
   ▼
URL Mapping Store
   │
   ▼
Return 302
```

The redirect path is optimized for reads because redirects are expected to greatly outnumber URL creations.

---

## 📊 Capacity Estimation

### Traffic Assumptions

```text
100M new URLs / month
100 redirects per URL
10× peak factor
```

### Average Traffic

```text
Writes ≈ 39 requests/sec
Reads  ≈ 3.9K requests/sec
```

### Peak Traffic

```text
Writes ≈ 390 requests/sec
Reads  ≈ 38.6K requests/sec
```

---

## 💾 Storage Estimation

For 5 years:

```text
100M URLs/month × 60 months
= 6 billion mappings
```

Assuming approximately **500 bytes per mapping**:

```text
Raw storage ≈ 3 TB
```

With 3 copies:

```text
≈ 9 TB
```

With approximately 30% additional headroom:

```text
Planned storage ≈ 12 TB
```

---

## 🧠 Redis Capacity

Assuming:

```text
100M hot mappings
≈ 50 GB raw data
```

With Redis overhead:

```text
≈ 80–100 GB
```

The cache should be distributed across multiple Redis nodes.

---

## 🖥️ Server Estimation

Assuming one redirect server handles approximately:

```text
5K RPS
```

At peak:

```text
38.6K / 5K ≈ 8 nodes
```

Deploying around:

```text
10 redirect nodes per region
```

provides additional failover and scaling capacity.

For the create path:

```text
≈ 3 nodes per region
```

Database:

```text
8 logical shards × 3 copies
```

---

## 🔐 Reliability & Security

### Security

* HTTPS/TLS
* WAF
* DDoS protection
* Rate limiting
* URL validation
* Abuse protection
* Request quotas

### Reliability

* Database replication
* Redis replication
* Multiple service instances
* Health checks
* Database backups
* Tested failover
* Horizontal scaling

---

## 📈 Scalability

The architecture is designed to scale horizontally.

### Application Layer

Both services are stateless:

```text
Create Service
   ├── Instance 1
   ├── Instance 2
   ├── Instance 3
   └── ...
```

and:

```text
Redirect Service
   ├── Instance 1
   ├── Instance 2
   ├── Instance 3
   └── ...
```

More instances can be added as traffic increases.

### Database Layer

URL mappings are distributed using:

```text
consistent hash(code)
```

This allows the database to scale by adding more shards.

---

## 🧩 Design Decisions

### Why Base62?

Base62 uses:

```text
A-Z
a-z
0-9
```

An 8-character Base62 code provides approximately:

```text
62⁸ ≈ 218 trillion combinations
```

This gives a very large key space while keeping URLs short.

### Why Redis?

Redirect traffic is read-heavy. Redis allows frequently accessed URLs to be served without hitting the database.

### Why Sharding?

With billions of mappings, a single database can become a bottleneck. Sharding distributes the data and workload across multiple database partitions.

### Why HTTP 302?

The system uses temporary redirects so the redirect behavior can remain controlled by the service and CDN rather than permanently caching a `301` redirect.

### Why Idempotency-Key?

Network failures can cause clients to retry a successful request. The idempotency record allows the server to return the same result instead of creating another short URL.

---

## 📌 Core Design Principle

The most important behavior of the system is:

```text
CREATE

Client
  ↓
Validate URL
  ↓
Get unique code
  ↓
Persist mapping
  ↓
Warm Redis
  ↓
Return 201


REDIRECT

Client
  ↓
CDN
  ↓
Redis
  ↓
Database on cache miss
  ↓
Return 302
```

**The create path prioritizes durability and correctness, while the redirect path prioritizes low latency and high availability.**

---

## 🗺️ System Design Diagram

The repository contains the complete **draw.io system design diagram** showing:

* Functional and non-functional requirements
* Capacity estimation
* API design
* Database structure
* Production architecture
* Create flow
* Redirect flow
* Redis caching
* Database sharding
* Key generation
* Reliability and scalability considerations

---

## 📁 Project Structure

```text
.
├── simple-url-shortener-system-design.drawio
└── README.md
```

---

## 🎯 Learning Objectives

This system design demonstrates practical concepts including:

* URL shortening
* REST API design
* Load balancing
* CDN caching
* Redis caching
* Database sharding
* Consistent hashing
* Idempotency
* Horizontal scaling
* Capacity estimation
* High availability
* Fault tolerance
* Rate limiting
* System reliability

---

## ⭐ Summary

This URL shortener is designed around a **read-heavy workload** where redirects significantly outnumber URL creations.

The key architectural choices are:

```text
CDN
 ↓
Redis
 ↓
Sharded Database
```

for fast redirects, combined with:

```text
Unique Base62 Key Generation
+
Idempotency
+
Database Replication
```

for reliable URL creation.

The result is a system capable of handling **billions of URL mappings** while maintaining low redirect latency and high availability.
