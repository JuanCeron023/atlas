# 05. Caching Layers, Edge Proxies & Reverse Gateways

> **Edge Caching Mechanics, Reverse Proxies, Cache Stampede Prevention, and HTTP Header Invariants.**

---

## 1. The POST Request Body-Drop Hazard in Reverse Proxies

### The Failure Mode
In microservice architectures, an API Gateway or reverse caching layer (e.g. Varnish Cache, Nginx, Envoy) sits between internal services to accelerate read traffic.

A client service configures an HTTP client helper that automatically injects default caching headers:
`Cache-Control: public, max-age=60`

When the client executes an HTTP `POST` request to ingest or mutate data:
- The request reaches the reverse proxy.
- Standard HTTP specification semantics (RFC 7234, RFC 9111) dictate that caching layers cache responses to safe, idempotent methods (`GET` and `HEAD`).
- When proxies encounter caching headers on a `POST` request, internal buffering routines may **strip or drop the HTTP request body** before forwarding to the upstream service!
- The upstream service receives an empty POST body (`Content-Length: 0`) and crashes or returns `HTTP 400 Bad Request: Empty Payload`.

### The Invariant: Never Attach Cache Headers to Mutating Requests
**Rule**: `Cache-Control` headers must strictly be omitted from all mutating HTTP methods (`POST`, `PUT`, `DELETE`, `PATCH`).

```go
func (c *Client) Do(req *http.Request) (*http.Response, error) {
	// Only attach Cache-Control headers to idempotent read requests
	if req.Method == http.MethodGet || req.Method == http.MethodHead {
		req.Header.Set("Cache-Control", "public, max-age=30")
	} else {
		// Explicitly ensure no caching headers leak into mutating methods
		req.Header.Del("Cache-Control")
	}
	return c.httpClient.Do(req)
}
```

---

## 2. Caching Complex Read Queries via POST (`X-Cache-Key`)

### The Challenge
Some read queries require large JSON query filters (e.g. filtering across 500 resource IDs and multi-field date ranges) that exceed standard URL length limits (2,048 characters), forcing the API design to use `POST /v1/query`.

Because proxies ignore or bypass `POST` requests by default, these heavy read endpoints bypass the cache and hit the database directly.

### Recommended Pattern: Custom Hash Header (`X-Cache-Key`)
When caching is required on read-only POST queries:
1. The client hashes the request body (e.g. SHA-256) and sets a custom header:
   `X-Cache-Key: query_sha256_<hash>`
2. The proxy configuration is configured to include the custom header in its cache lookup routine:

```vcl
sub vcl_recv {
    if (req.method == "POST" && req.http.X-Cache-Key) {
        // Force proxy to evaluate cache lookup for this specific POST query
        return (hash);
    }
}

sub vcl_hash {
    if (req.http.X-Cache-Key) {
        hash_data(req.http.X-Cache-Key);
        return (lookup);
    }
}
```

---

## 3. Backend-to-Backend Service Caching vs Public CDN Caching

### Architectural Separation
Edge caching for internal microservices differs fundamentally from public user-facing CDNs:

```
Internal Microservices (WITH Cache Layer):
  Service A → Gateway Proxy → Cache Layer → Upstream Service → Database

Public UI / Frontend:
  Client SPA → Public CDN (Static Assets Only) → API Gateway → Direct Services
```

### Key Differences:
1. **Client Session & Auth Tokens**: Backend-to-backend caching intentionally **ignores user cookies and Authorization tokens** in cache keys when serving shared catalog data (e.g. master configs, resource definitions). Treating user tokens as cache keys would cause a 0% cache hit rate.
2. **Error Responses (4xx / 5xx)**: Gateway proxies must **never cache error responses**. If an upstream service returns 500 or 503, set `TTL = 0` immediately to ensure transient service failures do not poison the cache for healthy retries.

---

## 4. Multi-Tier Layered Caching Architecture

### The Stampede Hazard (Thundering Herd)
When a cached key expires on a high-throughput endpoint (1,000 req/sec), all 1,000 concurrent requests immediately miss the cache and hammer the downstream database simultaneously.

### The Invariant: Multi-Tier Cache with Singleflight
Combine an in-memory LRU cache, a bounded TTL, and `singleflight`:

```go
package cache

import (
	"context"
	"time"

	"github.com/hashicorp/golang-lru/v2/expirable"
	"golang.org/x/sync/singleflight"
)

type MultiTierCache[T any] struct {
	lru   *expirable.LRU[string, T]
	group singleflight.Group
}

func NewMultiTierCache[T any](size int, ttl time.Duration) *MultiTierCache[T] {
	return &MultiTierCache[T]{
		lru: expirable.NewLRU[string, T](size, nil, ttl),
	}
}

func (c *MultiTierCache[T]) GetOrFetch(
	ctx context.Context,
	key string,
	fetchFn func(ctx context.Context) (T, error),
) (T, error) {
	// Tier 1: Local In-Memory LRU
	if val, ok := c.lru.Get(key); ok {
		return val, nil
	}

	// Tier 2: Singleflight Coalescing (Only 1 worker hits the DB/remote cache)
	detachedCtx := context.WithoutCancel(ctx)
	val, err, _ := c.group.Do(key, func() (any, error) {
		res, err := fetchFn(detachedCtx)
		if err != nil {
			return nil, err
		}
		c.lru.Add(key, res)
		return res, nil
	})

	var zero T
	if err != nil {
		return zero, err
	}
	return val.(T), nil
}
```

---

## 5. Soft Invalidation vs Hard Deletion

### The Failure Mode
When an resource or configuration item changes, naive caching systems execute `DELETE FROM cache WHERE id = ...`.
- Downstream reporting engines running a historical audit query mid-calculation suddenly find missing records.
- Historical aggregations fail with `NotFoundException`.

### Recommended Pattern: Soft Deprecation with State Flags
Instead of physically deleting cached entries:
1. Set `deprecated: true` and record `deprecatedDateTime: now`.
2. Active read queries filter strictly on `{ deprecated: false }`.
3. Historical audit queries and state-reconciliation routines can still read the deprecated version to observe pre-change state.
4. Physical cleanup is deferred to an asynchronous TTL purge worker (`purgeDateTime: now + 30 days`).

---

## 🔍 Agent Action Checklist

- [ ] Are `Cache-Control` headers completely excluded from all mutating requests (`POST`, `PUT`, `DELETE`)?
- [ ] If a `POST` request is cached, does it supply an explicit, hashed `X-Cache-Key`?
- [ ] Does the proxy or in-memory cache bypass/set `TTL = 0` on 4xx/5xx responses?
- [ ] Is high-concurrency cache fetching protected by `singleflight` to prevent thundering herds?
- [ ] Are invalidations performed via soft deprecation rather than hard deletes where audit trails are required?



