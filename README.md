# 🏛️ Atlas: Universal Software Engineering Lessons & Best Practices

> **Compendium of Core Architectural Principles, Concurrency Patterns, and Distributed Systems Design Lessons.**  
> Designed as an authoritative technical knowledge base and universal engineering playbook for AI agents and software engineers.

---

## 🎯 Purpose & Scope

**Atlas** is a domain-agnostic, educational repository of **fundamental software engineering lessons**. It synthesizes battle-tested lessons on designing, implementing, and debugging resilient, highly concurrent, and scalable backend architectures.

Any autonomous agent (such as `forge` agents) or software engineer designing microservices, event streams, or database schemas can consult Atlas as a universal best-practices reference.

---

## 🧭 Navigation Matrix

| Engineering Domain | Problems & Core Patterns Addressed | Guide |
|---|---|---|
| **Concurrency, Runtimes & Threads** | Context decoupling in shared in-flight calls, container CFS CPU quotas, admission control (load shedding), lock contention | [`01-concurrency-and-runtime-traps.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/01-concurrency-and-runtime-traps.md) |
| **Event Streaming & Idempotency** | Composite partition keys, per-resource FIFO order preservation, split-batch boundary reordering, deduplication | [`02-event-driven-streaming-and-idempotency.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/02-event-driven-streaming-and-idempotency.md) |
| **Temporal Data & Time Windows** | Extended continuous shifts, local timezone calculation vs UTC, half-open interval math $[start, end)$, rolling windows | [`03-temporal-data-and-time-windows.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/03-temporal-data-and-time-windows.md) |
| **API Contracts & Validation** | Pointer types for zero values (`0`/`false`), `null` vs `0` semantics, field erasure in partial updates, validator symmetry | [`04-api-contracts-and-validation.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/04-api-contracts-and-validation.md) |
| **Caching Layers & Edge Proxies** | HTTP method semantics and cache headers on mutating POSTs, cache stampede prevention, soft deprecation | [`05-caching-layers-and-edge-proxies.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/05-caching-layers-and-edge-proxies.md) |
| **Database Transactions & Storage** | Transaction boundaries and network I/O exclusion, atomic upserts vs check-then-insert, strict typing in filters | [`06-database-transactions-and-queries.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/06-database-transactions-and-queries.md) |
| **State Machines & Pipelines** | Precedence of manual human overrides over automated rules, Draft/Staged/Live lifecycles, 4-stage streaming pipelines | [`07-state-machines-and-streaming-pipelines.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/07-state-machines-and-streaming-pipelines.md) |
| **Observability & Diagnostics** | Correlation ID propagation, stack trace preservation in async recoveries, error origin integrity | [`08-observability-rca-and-triaging.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/08-observability-rca-and-triaging.md) |

---

## ⚡ The 10 Core Engineering Principles

1. **Decouple Contexts in Shared In-Flight Operations**: Never execute shared deduplicated work (such as `singleflight`) under the cancellation context of a single caller. If the leader disconnects, waiting followers must not be aborted in cascade.
2. **Align Language Runtimes with Container CPU Limits**: Runtimes reading physical host cores assign excessive threads, causing CFS CPU throttling. Synchronize runtime concurrency with actual cgroup limits (`automaxprocs`).
3. **Exclude All Network I/O from Database Transactions**: Pre-fetch external dependencies outside the transaction. Keep transaction closures reserved exclusively for fast, local atomic writes to prevent connection starvation and timeouts.
4. **Prefer Atomic Upserts Over Check-Then-Insert**: Querying for a record and then inserting it creates race condition collisions under concurrency. Always rely on atomic database-level upserts with unique constraints.
5. **Preserve FIFO Ordering with Composite Partition Keys**: In distributed event streams, route all state transitions for a given resource to the same partition using `{resourceType}_{resourceId}` to guarantee strict sequential delivery.
6. **Differentiate Absence of Data from Numerical Zero**: In contracts and models, use pointer types (`*int`, `*bool`). A value of `0` or `false` is a valid observation; `nil` or `null` represents missing information.
7. **Use Half-Open Intervals $[start, end)$ for Continuous Timelines**: When segmenting or querying continuous temporal slices, always enforce strict inequality (`<`) on the upper bound to prevent deleting or double-counting adjacent boundaries.
8. **Maintain Symmetry Across Ingress Validation Preconditions**: A shared entry validator must never enforce preconditions that apply to only one code path, starving valid alternative documented flows.
9. **Never Attach Cache Headers to Mutating HTTP Requests**: Intermediate reverse proxies can discard request bodies on `POST` or `PUT` requests if caching headers are present. Caching must be strictly reserved for idempotent reads.
10. **Preserve Stack Traces in Async Panic Handlers**: Any concurrency routine that captures panics (`recover`) must explicitly record the full call stack trace (`debug.Stack()`) so root causes can always be diagnosed.

---

## 🤖 Usage for Autonomous Agents

To review the protocol and verification checklist on how autonomous AI agents should consult and apply Atlas lessons, see [AGENT_INSTRUCTIONS.md](file:///c:/Users/juanc/Downloads/AISkills/atlas/AGENT_INSTRUCTIONS.md).
