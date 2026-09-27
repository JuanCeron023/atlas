# 🤖 Atlas: Autonomous Agent Protocol & Engineering Verification

> **Guidelines for AI Agents (Forge, Subagents, and Code Assistants) on Consulting and Applying Atlas Lessons and Invariants.**

---

## 🎯 When to Consult Atlas

As an autonomous agent, you MUST consult the corresponding Atlas guide whenever designing, implementing, or reviewing code in the following areas:

1. **System & Architecture Design**:
   - Concurrency, worker pools, or runtime configuration $\rightarrow$ Consult [`01-concurrency-and-runtime-traps.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/01-concurrency-and-runtime-traps.md).
   - Event-driven streams, message queues, or distributed deduplication $\rightarrow$ Consult [`02-event-driven-streaming-and-idempotency.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/02-event-driven-streaming-and-idempotency.md) and [`07-state-machines-and-streaming-pipelines.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/07-state-machines-and-streaming-pipelines.md).
   - Database operations, schema design, or transactions $\rightarrow$ Consult [`06-database-transactions-and-queries.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/06-database-transactions-and-queries.md).
   - Caching layers and reverse proxies $\rightarrow$ Consult [`05-caching-layers-and-edge-proxies.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/05-caching-layers-and-edge-proxies.md).

2. **Code Implementation & Refactoring**:
   - API contracts, DTOs, and input validation $\rightarrow$ Consult [`04-api-contracts-and-validation.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/04-api-contracts-and-validation.md).
   - Date handling, timezones, and continuous time windows $\rightarrow$ Consult [`03-temporal-data-and-time-windows.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/03-temporal-data-and-time-windows.md).
   - Async routines, goroutines, or retry loops $\rightarrow$ Consult [`01-concurrency-and-runtime-traps.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/01-concurrency-and-runtime-traps.md).

3. **Diagnostics & Root Cause Analysis**:
   - Investigating intermittent errors (500, 502, 504) or latency degradation $\rightarrow$ Consult [`01-concurrency-and-runtime-traps.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/01-concurrency-and-runtime-traps.md) and [`08-observability-rca-and-triaging.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/08-observability-rca-and-triaging.md).
   - Investigating duplicate messages or queue accumulations $\rightarrow$ Consult [`02-event-driven-streaming-and-idempotency.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/02-event-driven-streaming-and-idempotency.md).

---

## 📋 Agent Verification Checklist

Before approving an architecture, completing code generation, or finalizing an implementation plan, verify that these core invariants are met:

### 1. Concurrency & Runtime Integrity
- [ ] Do shared deduplicated calls (`singleflight`) decouple caller cancellation using `context.WithoutCancel(ctx)`?
- [ ] Does the containerized Go service synchronize runtime threads with container cgroup CPU limits (e.g. `_ "go.uber.org/automaxprocs"`)?
- [ ] Are sleeps, delays, and retries context-aware (`select { case <-time.After(): case <-ctx.Done(): }`), avoiding rigid un-cancellable blocking?
- [ ] Do all routines that capture async panics preserve the full call stack trace (`debug.Stack()`)?
- [ ] Are coarse-grained global locks avoided when operations target independent resource keys?

### 2. Database & Persistence Safety
- [ ] Is all external network I/O (HTTP, RPC) completely excluded from database transaction closures?
- [ ] Are record creations using atomic database upserts (`ReplaceOne` with `upsert: true`) rather than check-then-insert logic?
- [ ] Do queries against soft-deleted collections systematically filter inactive records (`{ deleted: { $ne: true } }`)?
- [ ] Do query filter types match the native storage engine data types?

### 3. Messaging & Streams
- [ ] Do stream partition keys incorporate the resource ID (`{resourceType}_{resourceId}`) to guarantee per-resource FIFO order?
- [ ] Are partial batch item failures supported, avoiding full-batch retry stampedes?
- [ ] Is consumer idempotency validated against a composite key (`MessageID + ResourceID`)?
- [ ] Are simultaneous state transitions at the same timestamp coalesced at the producer to prevent split-batch reordering races?

### 4. Temporal Data & Windows
- [ ] Do continuous temporal segment cleanups and window queries use half-open intervals ($[start, end)$ with strict `<` on upper bounds)?
- [ ] Do database timeline sorts include a deterministic secondary tie-breaker key to guarantee consistent ordering?
- [ ] Are validity and expiration range checks inclusive of the final boundary date?
- [ ] Are rolling window aggregations time-aware rather than array-index-based?

### 5. API Contracts & Intermediate Layers
- [ ] Do numeric or boolean fields that can legitimately be zero or false use pointer types (`*int`, `*bool`) in validated structs?
- [ ] Does the data contract distinguish clearly between absence of information (`null`) and a measurement of value zero (`0`)?
- [ ] Are `Cache-Control` headers completely omitted from mutating HTTP requests (`POST`, `PUT`, `DELETE`)?
- [ ] Do shared ingress validation gates accommodate all legitimate resolution paths without silent starvation?
