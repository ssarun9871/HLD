# URL Shortener — System Design

## 1. Functional Requirements

- User provides a long URL → system returns a short URL
- User hits short URL → system redirects to original long URL (HTTP 301/302)
- Custom aliases support (optional: user can choose their short URL)
- URL expiration — configurable TTL (default 1 year, max 5 years)
- Analytics tracking — clicks, geolocation, referrer (stretch goal)

**Out of scope:**
- User authentication (public service)
- URL validation/malware scanning

---

## 2. Non-Functional Requirements

- **Low latency:** Redirect < 10ms (P99)
- **High availability:** 99.99% uptime (read path critical, write path can tolerate brief downtime)
- **Read-heavy system:** 100:1 read/write ratio
- **Scalability:** Handle billions of URLs
- **Collision-free:** Short URLs must be unique and not easily guessable

---

## 3. Capacity Estimation

### Assumptions
- 100M new URLs per day
- Read/Write ratio: 100:1
- URL stored for 5 years
- Average URL mapping size: 500 bytes (short_url + long_url + metadata)

### Traffic
```
Write RPS  = 100M / 86400 ≈ 1,200 req/sec
Read RPS   = 1,200 × 100 = 120,000 req/sec
Peak RPS   = 120K × 2 = 240K req/sec (2x safety margin)
```

### Storage
```
Per day    = 100M × 500 bytes = 50 GB/day
5 years    = 50 GB × 365 × 5 = 91,250 GB ≈ 91 TB
```

### Bandwidth
```
Write BW   = 50 GB/day ÷ 100K sec ≈ 0.5 MB/sec
Read BW    = 0.5 MB/sec × 100 = 50 MB/sec
Total      ≈ 50 MB/sec
```

### Cache
```
80/20 rule — 20% URLs generate 80% traffic
Cache size = 91 TB × 20% ≈ 18 TB
Daily hot set ≈ 10 GB (fits in Redis easily)
```

---

## 4. API Design

### Create Short URL
```http
POST /api/v1/shorten
Content-Type: application/json

Request:
{
  "long_url": "https://example.com/very/long/path",
  "custom_alias": "promo2024",  // optional
  "ttl_days": 365               // optional
}

Response:
{
  "short_url": "https://short.ly/abc123",
  "long_url": "https://example.com/very/long/path",
  "created_at": "2024-04-18T10:30:00Z",
  "expires_at": "2025-04-18T10:30:00Z"
}
```

### Redirect
```http
GET /{short_code}

Response:
HTTP/1.1 301 Moved Permanently
Location: https://example.com/very/long/path
```

---

## 5. High Level Design

```
┌─────────┐          ┌─────────────┐         ┌──────────┐
│ Client  │──────────│  API Gateway │─────────│  LB      │
└─────────┘          └─────────────┘         └──────────┘
                                                    │
                    ┌───────────────────────────────┼──────────────┐
                    │                               │              │
              ┌─────▼─────┐                   ┌─────▼─────┐  ┌────▼─────┐
              │  Write    │                   │   Read    │  │  Read    │
              │  Service  │                   │  Service  │  │  Service │
              └─────┬─────┘                   └─────┬─────┘  └────┬─────┘
                    │                               │              │
                    │                         ┌─────▼──────────────▼─────┐
                    │                         │    Redis Cache (R/W)     │
                    │                         │  TTL-based eviction      │
                    │                         └─────┬──────────────┬─────┘
                    │                               │              │
              ┌─────▼─────────────────────────┐     │  Cache Miss  │
              │   PostgreSQL (Primary)        │◄────┘              │
              │   - url_mappings table        │                    │
              │   - Indexes on short_code     │                    │
              └─────┬─────────────────────────┘                    │
                    │                                              │
              ┌─────▼──────────────────────────┐                   │
              │  PostgreSQL (Read Replicas)    │◄──────────────────┘
              │  - Async replication           │
              └────────────────────────────────┘
```

---

## 6. Database Schema

```sql
CREATE TABLE url_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- Internal ID
    short_code VARCHAR(10) UNIQUE NOT NULL,         -- Base62(UUIDv7)
    long_url TEXT NOT NULL,
    custom_alias BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    click_count BIGINT DEFAULT 0
);
```

**Why PostgreSQL over NoSQL?**
- ACID guarantees for uniqueness constraint on `short_code`
- Strong indexing on short_code lookup (B-tree index)
- Relational integrity if we add analytics tables later
- URL shortener is read-heavy but not write-intensive enough to need Cassandra-level writes

---

## 7. Short URL Generation Strategy

### Base62 Encoding
```
Use sequential UUID (UUIDv7) → convert to Base62 → truncate to 7 chars

Characters: [a-zA-Z0-9] = 62 chars
Length = 7 chars → 62^7 = 3.5 trillion unique URLs

Example:
UUIDv7: 01875d4c-9a3f-7000-8f3e-9d2c4b1a5e6f
Base62 (full): 0A3fG9xKl2mN8pQrS4tUvW (22 chars)
Truncated: 0A3fG9x (7 chars)
Short URL: https://short.ly/0A3fG9x 
```

**Pros:**
- Time-ordered (recent URLs cluster together in DB indexes)
- Native UUID support in PostgreSQL and Cassandra
- 128-bit collision resistance (astronomically low)
- No single point of failure

**Cons:**
- Truncation to 7 chars introduces small collision risk (~0.0014% at 100M/day)
- Requires collision detection and retry logic
- First characters predictable (time-based prefix)

**Security Note:**
- Time-based prefix makes recent URLs semi-predictable
- If security critical, use full 22-char Base62 (zero collision risk)
- Or hash the UUIDv7 before Base62 conversion to randomize prefix

---


## 8. Key Design Decisions

### Caching Strategy
```
Redis Cluster (Master-Replica setup)
- Key: short_code
- Value: { long_url, expires_at }
- TTL: Same as URL expiration
- Cache-aside pattern
- Eviction: TTL-based + LRU for memory limits
```

**Read Flow:**
```
1. Check Redis → if hit, return long_url
2. If miss → Query PostgreSQL read replica
3. Write to Redis with TTL
4. Return long_url
```

**Write Flow:**
```
1. Generate short_code (UUIDv7 → Base62)
2. Insert into PostgreSQL primary
3. Write to Redis (async, fire-and-forget)
4. Return short_url to client
```

---

### Redirect: 301 vs 302
```
301 Permanent Redirect:
- Browser caches the redirect
- Subsequent visits don't hit our server
- Can't track analytics accurately

302 Temporary Redirect (Recommended):
- Every visit hits our server
- Accurate click tracking
- Slight latency overhead (mitigated by caching)
```

**Choice:** Use **302** for analytics tracking. If analytics not needed, use **301** to reduce server load.

---

### Custom Aliases
```
User requests: short.ly/promo2024

Flow:
1. Check if "promo2024" exists in DB
2. If exists → return error "Alias already taken"
3. If available → insert with custom_alias=true flag
4. Return short URL
```

**Collision Handling:**
- Custom aliases are first-come-first-serve
- No retry logic (user must choose different alias)

---

### URL Expiration & Cleanup
```
Passive Cleanup (Lazy Deletion):
- On read, check expires_at
- If expired → return 404
- Don't delete immediately (avoid write overhead)

Active Cleanup (Background Job):
- Cron job runs daily
- DELETE FROM url_mappings WHERE expires_at < NOW()
- Batch delete in chunks to avoid locking
```

---

## 9. Scaling Considerations

### Database Sharding
```
Shard by short_code (consistent hashing)
- Total URLs: 91 TB / 500 bytes = ~180B records
- 10 shards → 18B records per shard
- Each shard: ~9 TB
```

**Shard Key:** `hash(short_code) % num_shards`

---

### Read Replicas
```
1 Primary (Writes)
5 Read Replicas (Reads)
Read/Write split at application layer
```

---

### CDN for Static Content
```
Cache redirect responses at CDN edge
- For popular URLs (e.g., marketing campaigns)
- TTL: 1 hour
- Reduces origin server load by 90%
```

---

### Rate Limiting
```
Per IP: 100 requests/minute (write API)
No rate limit on read (redirect) API
Implementation: Redis + Token Bucket algorithm
```

---

## 10. Monitoring & Observability

### Metrics (Prometheus)
- `url_shortener_write_requests_total`
- `url_shortener_redirect_requests_total`
- `url_shortener_cache_hit_rate`
- `url_shortener_redirect_latency_p99`

### Alerts
- Cache hit rate < 80%
- P99 latency > 50ms
- Write failures > 1%
- DB replication lag > 10 seconds

---

## 11. Security Considerations

### Anti-Abuse
- Rate limiting on write API (prevent spam)
- CAPTCHA for high-volume IPs
- Blocklist for malicious domains (phishing/malware)

### Non-Guessable URLs
- Use UUIDv7 instead of auto-increment
- Makes enumeration attacks harder

### HTTPS Only
- Force HTTPS redirects
- Prevent man-in-the-middle attacks

---

## 12. Follow-Up Questions & Answers

**Q: How do you handle a sudden traffic spike (e.g., viral campaign)?**
A: Auto-scaling + CDN caching. If a URL goes viral, CDN will absorb 90% of traffic. Backend scales horizontally based on CPU/memory metrics.

**Q: What if Redis goes down?**
A: Fallback to PostgreSQL read replicas. Latency increases from 5ms → 20ms, but system remains available. Redis recovery from snapshot + AOF.

**Q: Can you support analytics (clicks, geolocation)?**
A: Yes. On redirect, async publish event to Kafka → stream to analytics DB (ClickHouse/BigQuery). Doesn't block redirect path.

**Q: How do you handle URL updates (change long URL for existing short URL)?**
A: Not supported by design (immutable URLs). If needed, invalidate cache + update DB, but breaks 301 cached redirects in browsers.

---