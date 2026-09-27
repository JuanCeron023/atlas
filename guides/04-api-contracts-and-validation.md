# 04. API Contracts, Validation & Data Modeling

> **Validation Pitfalls, Pointer Types vs Value Types, Nullable Semantics, and Gateway Entry Gate Design.**

---

## 1. Go Validator Zero-Value Trap: `validate:"required"`

### The Failure Mode
In Go microservices utilizing struct validation tags (e.g. `go-playground/validator`), engineers define request DTOs:
```go
// ANTI-PATTERN: Rejects legitimate 0 values!
type ConfigDTO struct {
    OffsetSeconds int32 `json:"offsetSeconds" validate:"required"`
    IsActive      bool  `json:"isActive"      validate:"required"`
}
```
When a client sends a valid request with an offset of zero (`{"offsetSeconds": 0, "isActive": false}`):
- The validator library treats Go's **zero-value** (`0` for integers, `""` for strings, `false` for booleans) as **missing / empty**!
- The API responds with `HTTP 400 Bad Request: Key: 'ConfigDTO.OffsetSeconds' Error:Field validation for 'OffsetSeconds' failed on the 'required' tag`.
- Clients are forced to send arbitrary non-zero numbers to bypass validation, corrupting business logic.

### Recommended Pattern: Pointer Types for Zero-Able Fields
Change scalar value types to **pointer types** (`*int32`, `*bool`, `*float64`).
When using a pointer:
1. `validate:"required"` checks whether the pointer itself is `nil` (i.e. whether the field was omitted from JSON).
2. If the field is present with value `0` or `false`, the pointer is non-nil (`&0`), and validation succeeds!

```go
// CORE INVARIANT: Use pointers for numeric and boolean fields that can legitimately be zero
type ConfigDTO struct {
    OffsetSeconds *int32 `json:"offsetSeconds" validate:"required"`
    IsActive      *bool  `json:"isActive"      validate:"required"`
}
```

---

## 2. Nullable Semantics: Distinguishing `null` from `0`

### The Failure Mode
An operational metrics service tracks telemetry and queue depths.
- During inactive maintenance hours, no measurements are taken.
- During active hours, a device is running, but the queue is empty (0 items).

If the backend models this field as a scalar value type (`int`):
```go
type MetricResponse struct {
    QueueDepth int `json:"queueDepth"` // Omitempty or default 0
}
```
In both cases, JSON outputs `{"queueDepth": 0}`.
Downstream systems cannot determine whether the reading represents an active queue with 0 items, or an offline device with no data. In telemetry and reporting, this causes false SLA calculations.

### Recommended Pattern: Explicit Tri-State Representation
Use pointer types to differentiate between:
- `nil` $\rightarrow$ Missing / No data / Gap in observation (`null` in JSON).
- `&0` $\rightarrow$ Active observation measuring zero (`0` in JSON).
- `&N` $\rightarrow$ Active observation measuring $N$ ($N$ in JSON).

```go
type MetricResponse struct {
    QueueDepth *int `json:"queueDepth"` // null when inactive, 0 when empty
}
```

---

## 3. URL Parameter Colon Encoding Breaking Map Lookups

### The Failure Mode
A service exposes an endpoint where clients query time intervals:
`GET /v1/metrics?interval=09:00`

1. HTTP client libraries URL-encode parameters by default, transforming `09:00` into `09%3A00`.
2. The server framework extracts the query string into a hash map without decoding:
   `intervalMap["09%3A00"]`
3. Downstream code queries the map using the literal interval key `map["09:00"]`.
4. Lookup fails silently, returning `nil` or empty responses.

### Invariant: Gateway Decoding & Clean Contract Types
- Prefer integer minutes from epoch/midnight (e.g. `interval=540`) or standard ISO formats for URL query parameters.
- If string time keys (`HH:MM`) are used, always enforce URL-decode middleware before handler dispatch.

---

## 4. Partial Updates (PATCH): Field Erasure

### The Failure Mode
An API receives a PATCH request to update a single user property:
`PATCH /v1/users/123 {"role": "ADMIN"}`

The handler unmarshals the request into a model struct:
```go
type UserUpdateDTO struct {
    Role        *string `json:"role"`
    Preferences *string `json:"preferences"` // omitted in JSON -> nil
}
```
The repository blindly applies the update to the database:
```go
// ANTI-PATTERN: Overwrites existing preferences with nil!
db.users.UpdateOne(ctx, filter, bson.M{
    "$set": bson.M{
        "role": dto.Role,
        "preferences": dto.Preferences, // Sets null in DB!
    },
})
```
Any field omitted in the incoming request is overwritten with `null` in the database, silently erasing existing data!

### Recommended Pattern: Dynamic Partial Update Map
Only add fields to the `$set` document if their pointers are explicitly non-nil:

```go
func BuildPartialUpdate(dto UserUpdateDTO) bson.M {
	setDoc := bson.M{}
	if dto.Role != nil {
		setDoc["role"] = *dto.Role
	}
	if dto.Preferences != nil {
		setDoc["preferences"] = *dto.Preferences
	}
	return bson.M{"$set": setDoc}
}
```

---

## 5. Entry Gate Asymmetry: Preconditions Starving Direct Paths

### The Failure Mode
A processing pipeline supports two ways to resolve an resource:
1. **Hierarchical Path (90% of traffic)**: Message $\rightarrow$ Channel $\rightarrow$ Partition $\rightarrow$ resource.
2. **Direct Path (10% of traffic)**: Message $\rightarrow$ Direct resource ID.

A shared ingestion adapter executes:
```go
// GATE 0 (Database Query):
filter := bson.M{"channelId": bson.M{"$exists": true, "$ne": []}} // DROPS ALL DIRECT PATH EVENTS!

// GATE A (Validator):
if len(event.ChannelId) == 0 {
    return errors.New("channelId is empty") // FAILS ALL DIRECT PATH EVENTS!
}
```
Because the shared entry gates enforce preconditions valid only for Path 1, **all events for Path 2 are silently discarded before ever reaching the resolver!**
No error logs appear, and metrics show empty zeros.

### Recommended Pattern: Gate Symmetry
A shared entry point must never front-load a prerequisite that applies to only a subset of supported event models:

```go
// CORE INVARIANT: Gate must permit alternative documented identifiers
if len(event.ChannelId) == 0 && event.DirectResourceId.IsZero() {
    return errors.New("event must provide either channelId or DirectResourceId")
}
```

---

## 6. Batch Endpoints vs N+1 HTTP Fan-Outs

### The Failure Mode
A backend job processes 400 resources. For each resource, it makes a single HTTP call to fetch configuration:
$$\text{HTTP Calls} = 400 \text{ calls} \times 80\text{ms} = 32 \text{ seconds}$$
Under load, connection pools saturate, container execution timeouts trigger, and reverse proxies return 504.

### Recommended Pattern: Batch by Range
Replace individual resource calls with a single batch endpoint accepting a slice of IDs:
`POST /v1/configs/batch {"resourceIDs": ["..."], "dateRange": "..."}`
- Total network round-trips: **1 HTTP call**.
- Total latency: **~120ms** (a 99.6% reduction).

---

## 🔍 Agent Action Checklist

- [ ] Do struct validation tags on `0`-accepting fields use pointer types (`*int32`, `*bool`)?
- [ ] Does the data contract distinguish between `null` (no data) and `0` (zero value)?
- [ ] Does partial update (PATCH) code check for non-nil before setting fields in the database?
- [ ] Do shared validation gates accommodate all legitimate resolution paths without silent starvation?
- [ ] Are high-volume external calls batched rather than executed in N+1 loops?



