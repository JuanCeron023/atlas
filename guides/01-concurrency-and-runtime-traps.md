# 01. Concurrency, Runtime Traps & Container Sizing

> **Failure Modes, Root Causes, and Engineering Lessons for High-Concurrency Microservices and Container Runtimes.**

---

## 1. Singleflight Context Cancellation Cascading Failures

### The Failure Mode
In high-throughput microservices, in-flight deduplication (e.g. Go's `singleflight.Group`) is standard practice to collapse concurrent duplicate reads (e.g. database lookups, configuration fetches, remote API queries) into a single shared execution.

When multiple concurrent requests query the same resource key simultaneously:
1. Request A (the leader) initiates the singleflight call under its own HTTP request `context.Context`.
2. Requests B, C, and D (followers) join the flight and await Request A's completion.
3. If Request A times out at the client/gateway boundary (e.g. reverse proxy or load balancer 15s timeout) or the client abruptly disconnects, Request A's context is **canceled**.
4. The shared execution is aborted with `context canceled`.
5. **The Cascade**: All waiting followers (B, C, and D) receive the `context canceled` error—even if their own request deadlines had plenty of remaining time!
6. If the singleflight function was populating state, subsequent processing in follower routines receives uninitialized `nil` pointers, triggering panics (`nil pointer dereference`).

### Root Cause
`singleflight.Group.Do` executes the provided function synchronously within the goroutine of the **first caller**. If that closure accepts or inherits the first caller's context directly, the work is strictly coupled to the leader's lifecycle.

Furthermore, in Go, an interface holding a typed nil or zero-value struct is not nil:
```go
// PITFALL: If result is a concrete struct, val is NOT nil even on error!
val, err, _ := group.Do(key, func() (any, error) {
    res, err := doWork(ctx)
    return res, err // if res is MyStruct{}, val != nil!
})
if val == nil && err != nil { // BUG: condition fails! err is dropped!
    return err
}
```

### Recommended Pattern: Caller Context Decoupling
To prevent leader cancellation from poisoning follower requests:
1. In Go 1.21+, use `context.WithoutCancel(ctx)` to detach the cancellation lifecycle while preserving context values (e.g., OpenTelemetry tracing spans, correlation IDs, logger tags).
2. Apply an independent, bounded timeout specifically for the shared operation.

```go
package singleflightutil

import (
	"context"
	"time"

	"golang.org/x/sync/singleflight"
)

type SafeGroup[T any] struct {
	group singleflight.Group
}

// DoSafe executes fn with context cancellation detached from the leader.
// It applies a bounded fallback timeout to ensure the flight cannot hang indefinitely.
func (g *SafeGroup[T]) DoSafe(
	ctx context.Context,
	key string,
	timeout time.Duration,
	fn func(detachedCtx context.Context) (T, error),
) (T, error) {
	// Detach from caller cancellation, but preserve trace & correlation values
	detachedCtx := context.WithoutCancel(ctx)
	flightCtx, cancel := context.WithTimeout(detachedCtx, timeout)

	val, err, _ := g.group.Do(key, func() (any, error) {
		defer cancel()
		return fn(flightCtx)
	})

	var zero T
	if err != nil {
		return zero, err
	}
	return val.(T), nil
}
```

---

## 2. Container CFS Quotas & CFS CPU Throttling

### The Failure Mode
A Go service deployed in a container environment (Docker, Kubernetes, AWS ECS) with a resource limit of `1 vCPU` and `2 GB RAM` exhibits severe latency spikes and sporadic 504 timeouts, even when average CPU utilization is reported at only 20–35%.

### Root Cause
1. **Host-Aware vs Container-Aware Runtime**: The Go runtime defaults `GOMAXPROCS` to `runtime.NumCPU()`. In virtualized cloud environments, `runtime.NumCPU()` queries the underlying **host node**, which often has 32, 64, or 96 physical cores.
2. **CFS Throttling**: The Linux Completely Fair Scheduler (CFS) enforces CPU limits over a quota period (typically 100ms). If a Go application with 64 OS worker threads bursts work, all 64 threads consume CPU slices simultaneously. The 100ms quota is exhausted in the first 10ms of the period!
3. The kernel immediately **throttles** the container for the remaining 90ms (`nr_throttled`). During this period, the application cannot respond to incoming packets, causing health check failures and request timeouts.

### Recommended Pattern: AutoMaxProcs
Always import `go.uber.org/automaxprocs` in containerized service entrypoints (`main.go`). It reads container cgroups (`/sys/fs/cgroup/cpu`) and automatically sets `GOMAXPROCS` to match the container's true CPU quota.

```go
package main

import (
	"log/slog"

	_ "go.uber.org/automaxprocs" // Automatically sets GOMAXPROCS to cgroup quota
)

func main() {
	slog.Info("Service starting with container-aware GOMAXPROCS")
	// ...
}
```

---

## 3. Load Shedding vs Internal Mutex Throttling

### The Failure Mode
Under sudden traffic surges, engineers often place concurrency limiters (e.g. buffered channels or mutexes) around internal business logic to protect downstream dependencies.

Under high load:
1. 500 requests enter the service per second.
2. The internal throttler allows 10 concurrent requests; the other 490 requests block waiting on a mutex or channel.
3. In-flight memory balloons because each blocked request holds its HTTP connection, socket buffers, and goroutine stack.
4. Load balancer health checks hit the service, get queued behind blocked requests, and time out.
5. The load balancer marks all target containers unhealthy, deregisters them, and the entire cluster collapses with `502 Bad Gateway`.

### Recommended Pattern: Early Admission Load Shedding
Never throttle deep in the call stack. Throttle at the **HTTP admission layer** using early load shedding:
- If capacity is exhausted, immediately reject excess requests with `HTTP 503 Service Unavailable` or `HTTP 429 Too Many Requests`.
- Include `Retry-After: <seconds>` headers.
- Never block incoming requests indefinitely.

```go
// ConcurrencyLimiter middleware sheds load at request admission
func ConcurrencyLimiter(maxConcurrent int) func(http.Handler) http.Handler {
	sem := make(chan struct{}, maxConcurrent)

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Health check endpoint MUST always bypass load shedding
			if r.URL.Path == "/healthz" || r.URL.Path == "/live" {
				next.ServeHTTP(w, r)
				return
			}

			select {
			case sem <- struct{}{}:
				defer func() { <-sem }()
				next.ServeHTTP(w, r)
			default:
				// Load shed immediately: do not buffer in memory
				w.Header().Set("Retry-After", "3")
				http.Error(w, "Service Overloaded (Load Shedding)", http.StatusServiceUnavailable)
			}
		})
	}
}
```

---

## 4. Shared Mutex Serialization Across Independent Keys

### The Failure Mode
A service manages state across hundreds of independent resource partitions or resources. A developer wraps database operations in a shared `sync.Mutex` on the service struct to "prevent concurrent DB conflicts."

When 4+ independent resources are modified concurrently:
1. Each operation takes ~3 seconds to complete its database calls.
2. Because of the shared mutex, all 4 operations run **sequentially** instead of in parallel.
3. Total duration stretches to `12s` (exceeding the health check threshold of the load balancer).
4. The load balancer deregisters the container as unresponsive.

### Rule of Architecture
**Never use a coarse-grained global mutex over independent partition keys.**
- If operations target different resource IDs, tenant IDs, or partition keys, they must execute concurrently.
- If synchronization is required, use **keyed sharded locks** (striped locks) or leverage database-level atomic operations.

---

## 5. Context-Aware Sleep & Retry Operations

### The Failure Mode
A retry loop uses `time.Sleep(delay)`. During application shutdown or when a client cancels a request, the goroutine remains stuck sleeping for the full delay (e.g. 5–10 seconds), leaking goroutines and preventing graceful pod termination.

### Recommended Pattern:
Always make sleeps context-aware using `select`:

```go
func SleepContext(ctx context.Context, d time.Duration) error {
	timer := time.NewTimer(d)
	defer timer.Stop()

	select {
	case <-timer.C:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

---

## 6. Panic Recovery in Worker Pools & Fan-Outs

### The Failure Mode
A worker pool recovers from panics using a generic `recover()` block:
```go
// ANTI-PATTERN: Masks the stack trace!
defer func() {
    if r := recover(); r != nil {
        log.Printf("recovered: %v", r) // Output: "recovered: runtime error: nil pointer dereference"
    }
}()
```
When a nil pointer panic occurs inside a fan-out of 1,000+ goroutines, the log only shows `"recovered: nil pointer dereference"`. There is **no file name, no line number, and no stack trace**. The root cause remains unprovable.

### Recommended Pattern: Panic Recovery with Stack Preservation
```go
package async

import (
	"context"
	"fmt"
	"log/slog"
	"runtime/debug"
)

// SafeGo runs a worker goroutine with full panic recovery and stack trace capture.
func SafeGo(ctx context.Context, logger *slog.Logger, fn func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				stack := string(debug.Stack())
				logger.ErrorContext(ctx, "Goroutine recovered from panic",
					slog.Any("panic", r),
					slog.String("stack_trace", stack),
				)
			}
		}()
		fn()
	}()
}
```

---

## 🔍 Agent Action Checklist

- [ ] Does any shared deduplicator (`singleflight`) pass the caller's context into the flight? If so, refactor with `context.WithoutCancel(ctx)`.
- [ ] Is `go.uber.org/automaxprocs` included in the container entrypoint?
- [ ] Are retry delays and backoffs cancellable via `select` on `ctx.Done()`?
- [ ] Does `recover()` capture `debug.Stack()`?
- [ ] Are global mutexes serializing operations on distinct resource IDs? If so, remove or stripe the locks.



