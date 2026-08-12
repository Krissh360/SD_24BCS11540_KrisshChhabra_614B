# URL Shortener — System Design

A scalable URL shortener system designed to convert long URLs into compact, unique URLs and redirect users to the original destination with low latency.

This system design is based on the requirements, estimations, and architecture described in the provided system-design document.

---

## 1. Problem Statement

The goal is to design a URL shortening service that allows users to:

* Create an account and log in.
* Convert long URLs into short URLs.
* View the history of their shortened URLs.
* Redirect users from a short URL to the original URL.
* Optionally create custom short URLs.

The system should be highly available, responsive, consistent, and capable of handling a very large number of redirects with low latency.

---

## 2. Requirements

### Functional Requirements

1. **Account Creation**

   * Users should be able to register and log in.

2. **Convert Long URL to Short URL**

   * Users can submit a long URL and receive a unique shortened URL.

3. **URL History**

   * Authenticated users can view the URLs they have previously shortened.

4. **URL Redirection**

   * When a user accesses a shortened URL, they should be redirected to the original URL.

5. **Custom URL Creation**

   * Users may optionally choose their own custom short identifier.

These requirements are specified in the original system-design document.

### Non-Functional Requirements

* **Unique shortened URLs**
* **High availability**
* **Responsive system**
* **Low latency**
* **Consistency**

The system should continue to work reliably as traffic and the number of stored URLs increase.

---

## 3. Capacity Estimation

The system is designed around the following assumptions:

* **10 million daily users**
* Each user shortens approximately **2 URLs per day**
* Approximately **20 million URLs are shortened per day**
* Approximately **2 billion redirects per day**

This makes the system heavily read/redirect oriented compared with URL creation.

### URL Creation

```text
Daily users                  = 10 million
URLs shortened per user/day  = 2
---------------------------------------
URLs shortened per day       = 20 million
```

### Redirect Traffic

The design estimates:

```text
20 million URL creations/day
              ↓
~2 billion redirects/day
```

Therefore, the redirect path is the most critical path from a performance perspective.

---

## 4. Storage Estimation

Each URL record contains metadata such as:

* Created timestamp
* Expiration information
* User ID
* Original URL
* Short URL identifier

The design assumes approximately **300 bytes per URL record**.

### Daily Storage

```text
20 million URLs × 300 bytes

≈ 6 GB/day
```

### Yearly Storage

```text
6 GB × 365 days

≈ 2.2 TB/year
```

Therefore, the system requires approximately **2.2 TB of raw storage per year**, excluding additional requirements such as replication, indexes, backups, and operational overhead.

---

## 5. Cache Estimation

Since the system handles approximately 2 billion redirects per day, caching is critical.

The design assumes approximately **20 million hot URLs**.

```text
20 million hot URLs × 300 bytes

≈ 6 GB
```

The recommended cache capacity is therefore approximately:

```text
8–10 GB
```

This provides additional headroom beyond the estimated 6 GB working set.

---

## 6. High-Level Architecture

The architecture consists of:

```text
                     +----------------+
                     |      User      |
                     +-------+--------+
                             |
                             v
                    +------------------+
                    | API / Entry Layer|
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | Load Balancer /  |
                    | Traffic Layer    |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | Application      |
                    | Servers          |
                    +--------+---------+
                             |
                  +----------+----------+
                  |                     |
                  v                     v
        +------------------+   +------------------+
        | Short URL        |   | Redirection      |
        | Creator          |   | Service          |
        +--------+---------+   +--------+---------+
                 |                      |
                 v                      v
        +------------------+   +------------------+
        | Persistent       |   | Cache            |
        | Database         |   +--------+---------+
        +------------------+            |
                                        |
                                        v
                               +------------------+
                               | Persistent       |
                               | Database         |
                               +------------------+
```

The original PDF's architecture shows the user request passing through an API/application layer and then reaching separate short-URL creation and redirection components backed by storage/cache.

---

## 7. URL Shortening Flow

When a user wants to shorten a URL:

```text
Client
  |
  v
API / Entry Layer
  |
  v
Application Server
  |
  v
Short URL Creator
  |
  +--> Generate unique short ID
  |
  +--> Store mapping
  |
  v
Return Short URL
```

Example:

```text
Original URL:

https://example.com/some/very/long/path

                    ↓

Short URL:

https://short.example/aZ91xK
```

The database stores the relationship:

```text
short_id  →  original_url
```

---

## 8. URL Redirection Flow

Redirection is the most frequently executed operation in the system.

```text
User
  |
  v
Short URL
  |
  v
API / Entry Layer
  |
  v
Redirection Service
  |
  v
Check Cache
  |
  +---- Cache HIT ----> Original URL
  |                         |
  |                         v
  |                     Redirect
  |
  +---- Cache MISS ---> Database
                            |
                            v
                       Original URL
                            |
                            v
                       Update Cache
                            |
                            v
                         Redirect
```

Because the system expects approximately **2 billion redirects per day**, the database should not be accessed for every redirect.

The cache should handle the majority of frequently accessed URLs.

---

## 9. Short URL Generation

A short URL requires a unique identifier.

A common approach is to generate a compact **Base62** identifier using:

```text
a-z
A-Z
0-9
```

Example:

```text
https://short.example/aZ91xK
```

The short identifier can then be used as the lookup key:

```text
aZ91xK → https://example.com/original/url
```

### Uniqueness

The system must guarantee that two different URLs do not receive the same short identifier.

Possible approaches include:

* Database uniqueness constraints
* Distributed ID generation
* Counter-based ID generation
* Snowflake-style IDs followed by Base62 encoding
* Collision detection and retry

A uniqueness constraint at the database level provides an additional layer of protection.

---

## 10. Data Model

A simplified URL mapping can look like:

```text
URLMapping
--------------------------------
short_id       : String
original_url   : String
user_id        : String
created_at     : Timestamp
expires_at     : Timestamp
```

Example:

```json
{
  "short_id": "aZ91xK",
  "original_url": "https://example.com/very/long/url",
  "user_id": "user123",
  "created_at": "2026-08-12T12:00:00Z",
  "expires_at": null
}
```

The original design specifically identifies created timestamp, expiration, and user ID as URL metadata.

---

## 11. Caching Strategy

Caching is one of the most important parts of this architecture.

### Cache Key

```text
short_id
```

### Cache Value

```text
original_url
```

For example:

```text
Key:
aZ91xK

Value:
https://example.com/very/long/url
```

### Cache Read Strategy

1. Receive the short URL.
2. Extract the short identifier.
3. Check the cache.
4. If the value exists, redirect immediately.
5. If the value does not exist, query the database.
6. Store the result in the cache.
7. Redirect the user.

This is commonly referred to as a **cache-aside** approach.

---

## 12. Scalability

The system should support horizontal scaling because of its large traffic volume.

### Application Servers

Application servers should ideally be stateless.

```text
                    Load Balancer
                   /      |      \
                  /       |       \
                 v        v        v
              Server 1  Server 2  Server 3
```

Any server should be able to process a request.

This allows additional application servers to be added when traffic increases.

### Database

The estimated raw storage requirement is approximately:

```text
2.2 TB/year
```

Additional capacity will be required for:

* Replication
* Indexes
* Backups
* Temporary data
* Growth
* Operational overhead

---

## 13. Availability

Availability is one of the major non-functional requirements.

To avoid single points of failure:

* Run multiple application instances.
* Use replicated databases.
* Use redundant cache nodes.
* Use health checks.
* Automatically replace unhealthy instances.
* Use load balancing.
* Keep application servers stateless.
* Maintain database backups.

A failure of one application server should not cause the entire URL shortener to become unavailable.

---

## 14. Consistency

The most important piece of data in the system is the mapping:

```text
short_id → original_url
```

Once a short URL has been created, it should always resolve to the correct destination.

### URL Creation

The system should:

1. Generate a unique short ID.
2. Store the mapping in persistent storage.
3. Confirm that the mapping was successfully stored.
4. Return the short URL to the user.

### Redirection

The system should:

1. Check the cache.
2. If the cache contains the mapping, use it.
3. Otherwise, query persistent storage.
4. Populate the cache.
5. Redirect the user.

Persistent storage should remain the source of truth.

---

## 15. Custom Short URLs

Custom URLs are an optional feature.

For example:

```text
https://short.example/my-profile
```

instead of:

```text
https://short.example/aZ91xK
```

The system should validate:

* Alias format
* Alias length
* Reserved words
* Duplicate aliases
* User authorization

A uniqueness constraint should ensure that two users cannot claim the same alias.

---

## 16. URL History

Authenticated users should be able to see their previously shortened URLs.

Example endpoint:

```http
GET /api/v1/urls
```

Example response:

```json
[
  {
    "shortUrl": "https://short.example/aZ91xK",
    "originalUrl": "https://example.com/",
    "createdAt": "2026-08-12T12:00:00Z",
    "expiresAt": null
  }
]
```

Pagination should be used for users with a large number of shortened URLs.

Example:

```http
GET /api/v1/urls?page=2&limit=20
```

---

## 17. API Design

A possible REST API:

| Method | Endpoint                | Description              |
| ------ | ----------------------- | ------------------------ |
| `POST` | `/api/v1/urls`          | Create a short URL       |
| `GET`  | `/{shortId}`            | Redirect to original URL |
| `GET`  | `/api/v1/urls`          | Get user's URL history   |
| `POST` | `/api/v1/auth/register` | Register a user          |
| `POST` | `/api/v1/auth/login`    | Login                    |

### Create Short URL

Request:

```http
POST /api/v1/urls
```

```json
{
  "originalUrl": "https://example.com/a/very/long/url"
}
```

Response:

```json
{
  "shortUrl": "https://short.example/aZ91xK"
}
```

### Redirect

```http
GET /aZ91xK
```

Response:

```http
HTTP/1.1 302 Found
Location: https://example.com/a/very/long/url
```

---

## 18. Performance Considerations

The most important performance characteristic is the extremely high redirect-to-write ratio.

```text
URL creations/day  ≈ 20 million
Redirects/day      ≈ 2 billion
```

Therefore, the system should prioritize the redirect path.

### Important optimizations

* Cache hot URLs.
* Keep application servers stateless.
* Scale application servers horizontally.
* Minimize database access during redirects.
* Use efficient indexing on `short_id`.
* Monitor cache hit rate.
* Monitor P95/P99 redirect latency.
* Keep the redirect service lightweight.

---

## 19. Security and Abuse Prevention

For a production system, additional security controls should be implemented.

### Authentication

Users should authenticate before accessing their URL history or managing their URLs.

### Authorization

A user should only be able to modify or delete URLs that belong to them.

### Rate Limiting

Limit URL creation requests to prevent:

* Spam
* Abuse
* Automated attacks
* Excessive resource consumption

### URL Validation

Submitted URLs should be validated before storing them.

### Malicious URLs

The system should consider mechanisms for detecting or disabling:

* Phishing URLs
* Malware destinations
* Spam URLs
* Abusive content

### Reserved Short IDs

Certain identifiers should be reserved:

```text
admin
login
api
register
health
metrics
```

---

## 20. Observability

The system should expose metrics and logs to make failures and performance problems easy to identify.

### Traffic Metrics

* URL creation requests/sec
* Redirect requests/sec
* Requests per endpoint

### Latency Metrics

* P50 latency
* P95 latency
* P99 latency
* Database latency
* Cache latency

### Cache Metrics

* Cache hit rate
* Cache miss rate
* Cache memory usage
* Eviction rate

### Reliability Metrics

* Error rate
* Database availability
* Application health
* Failed redirects

---

## 21. Key Design Decisions

| Decision                      | Reason                            |
| ----------------------------- | --------------------------------- |
| Stateless application servers | Enables horizontal scaling        |
| Cache original URLs           | Reduces database load             |
| Unique short IDs              | Prevents URL collisions           |
| Persistent URL mapping        | Provides durability               |
| Separate redirect path        | Optimizes the read-heavy workload |
| Pagination for URL history    | Avoids large responses            |
| Replicated infrastructure     | Improves availability             |
| Cache-aside strategy          | Keeps database as source of truth |

---

## 22. Capacity Summary

| Metric                      |    Estimate |
| --------------------------- | ----------: |
| Daily users                 |  10 million |
| URLs/user/day               |           2 |
| New URLs/day                |  20 million |
| Redirects/day               |  ~2 billion |
| Metadata/URL                |  ~300 bytes |
| Storage/day                 |       ~6 GB |
| Storage/year                |     ~2.2 TB |
| Hot URLs                    | ~20 million |
| Estimated cache requirement |       ~6 GB |
| Recommended cache capacity  |    ~8–10 GB |

These estimates are taken directly from the capacity calculations in the provided system-design document.

---

## 23. Future Improvements

The basic architecture can be extended with:

* Analytics and click tracking
* Custom domains
* QR code generation
* URL expiration
* Link deletion
* Link editing
* Geographic analytics
* Device/browser analytics
* Abuse detection
* Rate limiting
* Distributed ID generation
* Database sharding
* Multi-region deployment
* CDN integration

These features can be added without changing the core concept of:

```text
Short URL
    ↓
Short ID
    ↓
Cache
    ↓
Persistent Mapping
    ↓
Original URL
```

---

## 24. Final Architecture Summary

The URL shortener is fundamentally a **read-heavy distributed system**.

The core flow is:

```text
                URL Creation

User
  ↓
API Layer
  ↓
Application Server
  ↓
Short URL Creator
  ↓
Database
  ↓
Short URL
```

And the dominant flow is:

```text
                URL Redirection

User
  ↓
Short URL
  ↓
API Layer
  ↓
Redirection Service
  ↓
Cache
  ↓
Original URL
  ↓
Redirect
```

The key design priorities are:

1. Generate unique short identifiers.
2. Store URL mappings reliably.
3. Cache frequently accessed URLs.
4. Keep the application tier stateless.
5. Scale horizontally.
6. Minimize database access on redirects.
7. Maintain high availability.
8. Maintain consistency between short URLs and their original destinations.

With approximately **20 million URL creations and 2 billion redirects per day**, caching and horizontal scalability are the most important architectural considerations.
