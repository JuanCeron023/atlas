# 06. Database Transactions, Queries & Storage Safety

> **Transaction Boundaries, Atomic Upserts, Typing Hazards, and Query Safety in High-Volume Storage Engines.**

---

## 1. Network I/O Inside Database Transactions

### The Failure Mode
A service updates an resource's configuration. To update state atomically, a developer wraps the logic inside a database transaction:

```go
// ANTI-PATTERN: Network I/O inside a DB transaction closure!
err := session.WithTransaction(ctx, func(sessCtx mongo.SessionContext) (any, error) {
    // 1. HTTP calls to external microservices (auth, permissions, metadata)
    metadata, err := metaClient.GetMetadata(sessCtx, ResourceID) // ~150ms HTTP call
    rules, err := rulesClient.GetRules(sessCtx, ResourceID)       // ~200ms HTTP call
    
    // 2. Perform write
    return repo.SaveConfig(sessCtx, ResourceID, metadata, rules)
})
```

When 10 concurrent requests arrive:
- Each transaction holds a database connection and an open transaction session while making remote HTTP calls.
- If one external HTTP call experiences a minor latency hiccup (e.g. 2–5 seconds):
- The database server transaction lifetime limit (typically 60–120s, or client-side transaction timeouts of 5–10s) expires.
- The storage engine aborts the transaction server-side.
- When the code finally attempts the write, the database responds with:
  `Command failed with error (NoSuchTransaction / TransactionAborted): Transaction has been aborted`.
- Connection pools saturate and cascading 500 errors propagate across the application.

### The Invariant: Pre-Fetch Everything, Transaction ONLY Writes
**Rule**: Database transactions must be purely computational and transactional. **Never execute an HTTP call, message publish, or remote RPC inside a database transaction closure.**

```go
// 1. PRE-FETCH ALL DATA OUTSIDE TRANSACTION (Parallelized if possible)
metadata, err := metaClient.GetMetadata(ctx, ResourceID)
if err != nil {
    return err
}
rules, err := rulesClient.GetRules(ctx, ResourceID)
if err != nil {
    return err
}

// 2. EXECUTE TRANSACTION PURELY FOR ATOMIC WRITES
return session.WithTransaction(ctx, func(sessCtx mongo.SessionContext) (any, error) {
    if err := repo.ArchivePreviousConfig(sessCtx, ResourceID); err != nil {
        return nil, err
    }
    return nil, repo.InsertNewConfig(sessCtx, ResourceID, metadata, rules)
})
```

---

## 2. Check-Then-Insert vs Atomic Upserts

### The Failure Mode
A worker creates resource definitions (roles, permissions, counter settings). It uses a "check-then-insert" pattern:

```go
// ANTI-PATTERN: Inherent race condition under concurrency!
existing, err := repo.FindByName(ctx, name)
if err == nil && existing != nil {
    return nil // already exists
}
return repo.Insert(ctx, newResource) // DUPLICATE KEY ERROR!
```
Under concurrent invocations (e.g. multiple container instances bootstrapping or processing stream events simultaneously):
1. Both workers execute `FindByName` at the exact same millisecond. Both see that the record does not exist.
2. Both proceed to `Insert`.
3. The first insert succeeds; the second insert crashes with a `DuplicateKeyException`.
4. If no unique database index was configured, **duplicate records are silently created**, corrupting domain integrity.

### Recommended Pattern: Atomic Upsert
Replace check-then-insert with an atomic database upsert (`findAndModify` or `UpdateOne` with `upsert: true`):

```go
func UpsertResource(ctx context.Context, col *mongo.Collection, resource Resource) error {
	filter := bson.M{"code": resource.Code}
	update := bson.M{
		"$setOnInsert": bson.M{
			"code":      resource.Code,
			"createdAt": time.Now(),
		},
		"$set": bson.M{
			"name":      resource.Name,
			"updatedAt": time.Now(),
		},
	}
	opts := options.Update().SetUpsert(true)

	_, err := col.UpdateOne(ctx, filter, update, opts)
	return err
}
```

---

## 3. Strict Type Safety in Document Filters

### The Failure Mode
In document databases (MongoDB, DynamoDB, PostgreSQL JSONB), identifier fields are often stored as binary or object types (e.g. `primitive.ObjectID` in MongoDB).

A developer constructs a filter using a raw string representation:
```go
// ANTI-PATTERN: String vs native binary/ObjectID type mismatch!
filter := bson.M{"ResourceID": ResourceID.Hex()} // Querying as string "6637..."
```
Because the storage engine is strictly typed:
$$\text{ObjectID}("6637...") \ne \text{String}("6637...")$$
- The query matches **zero documents**, even though a document with that visual ID exists!
- Worse, in some repository builders, passing an un-parsable type caused the filter builder to silently fall back to an empty filter `{}`:
  `col.Find(ctx, bson.M{})`
- The query executes a full-collection scan and returns **all documents in the entire database**, causing memory exhaustion and data leaks!

### Recommended Pattern: Strict BSON Type Conversion
Always validate and convert string IDs to native types before querying:

```go
func ByResourceID(idStr string) (bson.M, error) {
	objID, err := primitive.ObjectIDFromHex(idStr)
	if err != nil {
		return nil, fmt.Errorf("invalid ObjectID hex: %w", err)
	}
	return bson.M{"ResourceID": objID}, nil
}
```

---

## 4. Unexported Filter Getters: The Empty Filter Disaster

### The Failure Mode
A custom query filter struct is defined:
```go
type QueryFilter struct {
    ResourceID primitive.ObjectID
    filter   bson.M
}
```
The repository code executes:
```go
// BUG: filter.Get() was never called or field was unexported!
cursor, err := col.Find(ctx, qf.filter) 
```
Because `qf.filter` was an uninitialized nil map, the driver serialized it to `{}` (match all). 
The query executed an unindexed full collection scan and returned every document in the database!

### Invariant: Unit Test Filter Cardinality
Always write a unit test verifying that applying a query filter **reduces result cardinality** compared to an unfiltered query.

---

## 5. Mandatory Soft-Delete Filtering

### The Failure Mode
In multi-tenant or auditing databases, records are soft-deleted via `{ "deleted": true }`.

Engineers write custom queries:
```javascript
// BUG: Omits soft-delete check!
db.configs.find({ "tenantId": id, "status": "ACTIVE" })
```
Deleted records are returned to clients, causing phantom configurations, state corruption, and inconsistent calculations.

### The Invariant: Global Soft-Delete Exclusion
Every repository query must systematically enforce soft-delete filtering:

```go
func WithActiveOnly(filter bson.M) bson.M {
	filter["deleted"] = bson.M{"$ne": true}
	return filter
}
```

---

## 🔍 Agent Action Checklist

- [ ] Are any external network calls (HTTP, RPC, messaging) inside a DB transaction closure? If so, extract them outside!
- [ ] Are record insertions using atomic upserts (`UpdateOne` with `upsert: true`) rather than check-then-insert?
- [ ] Are database ID fields queried with their native binary/ObjectID types rather than raw strings?
- [ ] Do custom query filters guarantee non-empty BSON representations?
- [ ] Do all queries explicitly filter `{ deleted: { $ne: true } }`?



