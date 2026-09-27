# 08. Observability, Root-Cause Analysis & Triaging

> **Distributed Tracing, Structured Logging Safety, Error Masking Prevention, and the RCA Playbook.**

---

## 1. End-to-End Tracing & Correlation ID Propagation

### The Failure Mode
A client application experiences an intermittent `500 Internal Server Error` on an API request.
When engineers inspect the logs of Gateway Service A, they see:
`[ERROR] request failed: upstream service returned error`
Service A calls Service B, which calls Service C, which queries the database.
Because no Correlation ID was passed in headers:
- There is no reliable way to correlate the error in Service A with logs in Service B or C.
- Engineers are forced to guess based on imprecise timestamps across unsynchronized server clocks, wasting hours during critical debugging sessions.

### The Invariant: Mandatory Correlation ID Injection
Every inbound HTTP request or message stream must extract or generate a Correlation ID (e.g. `X-Correlation-ID` or OpenTelemetry `traceparent`) and propagate it across all downstream HTTP clients, gRPC calls, and messaging envelopes.

```go
func TracingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		corrID := r.Header.Get("X-Correlation-ID")
		if corrID == "" {
			corrID = uuid.New().String()
		}

		ctx := context.WithValue(r.Context(), "correlation_id", corrID)
		w.Header().Set("X-Correlation-ID", corrID)

		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

---

## 2. Panic Logging: The Masked Stack Trace Catastrophe

### The Failure Mode
An asynchronous concurrency runner handles background worker goroutines:
```go
// ANTI-PATTERN: Masks the stack trace!
defer func() {
    if r := recover(); r != nil {
        logger.Error("AsyncRunner: run error", "err", fmt.Sprintf("recovered: %v", r))
    }
}()
```
When a worker encounters a nil pointer dereference, the application log contains:
`AsyncRunner: run error | err: recovered: runtime error: invalid memory address or nil pointer dereference`

**The Impact**:
- The log contains **no file name**, **no function name**, and **no line number**.
- Because the fan-out spawned 1,000+ goroutines across dozens of source files, engineers cannot determine which struct was uninitialized.
- The bug remains unresolvable in practice.

### Recommended Pattern: Explicit `debug.Stack()` Logging
Always format recovered panics with `debug.Stack()`:

```go
package recovery

import (
	"context"
	"log/slog"
	"runtime/debug"
)

func RecoverWithStack(ctx context.Context, logger *slog.Logger, opName string) {
	if r := recover(); r != nil {
		stackTrace := string(debug.Stack())
		logger.ErrorContext(ctx, "Panic recovered in operation",
			slog.String("operation", opName),
			slog.Any("panic", r),
			slog.String("stack_trace", stackTrace),
		)
	}
}
```

---

## 3. Structured Logging Safety: Reserved Attribute Collisions

### The Failure Mode
A developer uses Go's structured logger `log/slog` to record an incoming event:
```go
// ANTI-PATTERN: 'source' is a reserved key in slog when AddSource is true!
slog.Info("processing event", slog.String("source", event.Source))
```
In JSON log handlers that configure `slog.HandlerOptions{AddSource: true}`, the key `"source"` is a **reserved built-in attribute** used by the Go runtime to output source code file/line locations (`{"function": "...", "file": "...", "line": 42}`).
When the application attempts to serialize the log line, the duplicate attribute causes a **panic in the logging framework itself**, crashing the service!

### The Invariant: Domain-Prefixed Attribute Names
Never use generic un-prefixed keys like `source`, `time`, or `level` in structured logging:
- Use `eventSource` or `dataSource` instead of `source`.
- Use `timestamp` or `occurredAt` instead of `time`.

---

## 4. Misleading User Error Masking (Upstream Service Failure vs Client Data)

### The Failure Mode
A client uploads a batch containing 500 records.
The ingestion service executes:
```go
// Step 1: Upstream dependency call to validate dependencies
metadata, err := dependencyClient.GetMetadataBatch(ctx, resourceIDs)
if err != nil {
    // ANTI-PATTERN: Blaming the client's data for an upstream crash!
    for _, record := range records {
        record.AddError("Could not validate record, please check input data")
    }
    return recordErrors
}
```
The upstream dependency service crashed with a 500 internal error.
The user interface displays:
`Row 1: Invalid input data`
`Row 2: Invalid input data`
...
`Row 500: Invalid input data`

**The Chaos**:
- The user spends hours editing and re-formatting valid data, believing their payload is corrupt.
- On-call engineers waste time hunting for input syntax bugs.
- The actual failure was a transient upstream microservice crash!

### The Invariant: Error Origin Integrity
Never transform an upstream service error (500, 503, timeout) into a client input validation error (400):
- If an upstream dependency fails: return `HTTP 502/503 "Upstream Dependency Unavailable, please retry"`.
- Only return field validation errors if the input data itself violates structural or business rules.

---

## 5. The Universal RCA Triage Playbook

When investigating critical system anomalies, follow this battle-tested triage workflow:

```
[Phase 1: Boundary Isolation]  ──►  [Phase 2: Timeline Correlation]  ──►  [Phase 3: Root Mechanism Proving]
Identify First Failing Timestamp     Align Sibling Service Logs            Differentiate Root Cause from Symptoms
```

### Phase 1: Boundary Isolation
1. **Identify the exact millisecond of the first failure**: Do not look at logs across an hour window. Zoom into a tight $\pm 10$-second window around the initial error.
2. **Filter by HTTP Status Code**: Look for the exact moment metrics transitioned from `HTTP 200` to `HTTP 500/502/504`.

### Phase 2: Timeline Correlation
1. **Cross-Service Contamination Analysis**:
   Query distributed log aggregators for shared cancellations and panics:
   ```
   "context canceled" OR "panic" OR "deadline exceeded"
   ```
2. **Identify Leader vs Followers**:
   Look for the request that entered first and died on an exact round timeout (e.g. `duration: 15005ms`). This was the leader that timed out, aborting the shared context. Requests dying at the exact same millisecond with short durations (`1ms`, `18ms`) are poisoned followers.

### Phase 3: Root Mechanism Proving
1. **Differentiate Cause from Symptom**:
   - *Symptom*: Client file upload returns validation error on all rows.
   - *Intermediate*: Upstream batch validation endpoint returned 500.
   - *Root Cause*: Shared singleflight leader timed out at 15s $\rightarrow$ canceled context propagated to worker pool $\rightarrow$ unhandled nil dereference in un-cancelled worker routine.

---

## 🔍 Agent Action Checklist

- [ ] Is `X-Correlation-ID` propagated through all outbound HTTP and messaging headers?
- [ ] Do all goroutine worker pool `recover()` blocks output `debug.Stack()`?
- [ ] Are structured logging attributes checked to avoid colliding with `slog` builtins (e.g. `source`)?
- [ ] Do batch validation endpoints clearly distinguish upstream service failures from user data validation errors?
- [ ] When diagnosing intermittent failures, did you verify whether the fix branch was actually merged to `main`?



