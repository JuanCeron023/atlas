# 02. Event-Driven Streaming, Partitioning & Idempotency

> **Message Partitioning Invariants, Deduplication Patterns, and Out-of-Order Failure Modes in Event Streams and Message Queues.**

---

## 1. Stream Partition Key Design & Per-Resource FIFO Guarantees

### The Failure Mode
In distributed stream architectures (Kafka, AWS Kinesis, Apache Pulsar), messages are distributed across shards/partitions based on their **Partition Key**.

When systems use coarse partition keys such as an event type name or a static string:
- All events across thousands of distinct resources hash to a single partition.
- That single partition becomes an extreme bottleneck (`ThroughputExceeded`), while other partitions remain completely idle.
- Conversely, if partition keys are randomized (`UUID`), events for the **same resource** land on different partitions. Distributed consumers process them concurrently without ordering guarantees: an `UPDATE` or `DELETED` event may be processed before its corresponding `CREATED` event!

### Recommended Partition Key Format
To ensure strict FIFO ordering per resource while distributing load evenly across all available stream partitions, always use a **domain-composite partition key**:

$$\text{PartitionKey} = \text{ResourceType} + \text{"\_"} + \text{ResourceID}$$

```go
// Example: order_event_65ecb2c24e6e3aa8224faa24
partitionKey := fmt.Sprintf("%s_%s", domainType, ResourceID.Hex())
```

### Why this works:
1. **Per-Resource Ordering**: All lifecycle transitions for a specific resource will hash to the exact same partition, guaranteeing sequential consumption.
2. **Cross-Resource Sharding**: distinct resources hash across different partitions, maximizing horizontal scaling across the consumer cluster.

---

## 2. Poller-Level State Coalescing & Split-Batch Reordering

### The Failure Mode
A state scheduler emits two state events for the same resource at the exact same transition timestamp:
1. `STATE_COMPLETED` for Phase 1 (ending at $T$).
2. `STATE_ACTIVE` for Phase 2 (starting at $T$).

A polling service queries due events and pushes a batch of 100 records into the stream in sequence: `[..., COMPLETED, ACTIVE, ...]`.

Downstream, a stream consumer Lambda or worker pool pulls with a batch size of 10.
- The `COMPLETED` event for Resource A falls at index 9 (Batch 1).
- The `ACTIVE` event for Resource A falls at index 10 (Batch 2).
- Batch 1 executes first, writing `status = COMPLETED` to the database and triggering downstream completion notifications.
- Batch 2 executes milliseconds later, updating `status = ACTIVE`.
- **The Bug**: Downstream systems receive a false-alarm completion notification, or historical audit logs show an erroneous transient state dip.

### Root Cause
Consumer-side deduplication (e.g. an in-memory map where newer states override older ones) **only works within the scope of a single batch invocation**. When a batch boundary splits the pair across batches, consumer-side logic cannot observe both events simultaneously.

### Recommended Pattern: Producer-Side State Coalescing
When a state transition is instantaneous at timestamp $T$, the **producer/poller** must coalesce or deduplicate mutually exclusive events before publishing to the stream:

```go
// CoalesceStateTransitions eliminates transient completion events if an active event exists at the same timestamp
func CoalesceStateTransitions(events []ResourceEvent) []ResourceEvent {
	type key struct {
		ResourceID  string
		timestamp int64
	}

	hasActive := make(map[key]bool)
	for _, ev := range events {
		if ev.State == "ACTIVE" {
			hasActive[key{ev.ResourceID, ev.Timestamp.Unix()}] = true
		}
	}

	coalesced := make([]ResourceEvent, 0, len(events))
	for _, ev := range events {
		k := key{ev.ResourceID, ev.Timestamp.Unix()}
		// Suppress transient transition event if an active event starts at the identical second
		if ev.State == "COMPLETED" && hasActive[k] {
			continue
		}
		coalesced = append(coalesced, ev)
	}
	return coalesced
}
```

---

## 3. Shard Error Handling: Partial Batch Item Failures

### The Failure Mode
A stream consumer receives a batch of 100 records. 99 records process successfully, but record #42 encounters a malformed payload or unhandled edge case and returns an error.

By default, the stream worker runtime returns an error for the entire batch. The broker then **re-delivers all 100 records** from the beginning:
- The 99 successful records are executed over and over again, generating duplicate side-effects and wasting compute.
- The stream iterator age spikes to hours, causing cascading lag across all consumers on that shard.

### Recommended Pattern: Selective Batch Item Failures
Always configure stream consumers with partial batch failure reporting (e.g. AWS Lambda `ReportBatchItemFailures` or Kafka manual offset commits per record):

```go
type BatchResponse struct {
	BatchItemFailures []BatchItemFailure `json:"batchItemFailures"`
}

type BatchItemFailure struct {
	ItemIdentifier string `json:"itemIdentifier"`
}

func Handler(ctx context.Context, batchEvent StreamBatchEvent) (BatchResponse, error) {
	var failures []BatchItemFailure

	for _, record := range batchEvent.Records {
		err := processRecord(ctx, record)
		if err != nil {
			// Record the sequence number of ONLY the failed record
			failures = append(failures, BatchItemFailure{
				ItemIdentifier: record.SequenceNumber,
			})
		}
	}

	// Returning nil error allows the runtime to checkpoint all records EXCEPT failed ones
	return BatchResponse{BatchItemFailures: failures}, nil
}
```

---

## 4. Distributed Idempotency: Composite Keys

### The Failure Mode
In distributed systems, message brokers provide **at-least-once delivery**. A message may be delivered multiple times due to consumer restarts, network timeouts, or broker partition rebalancing.

If a consumer checks idempotency solely using `MessageID`, fan-out processing breaks:
- A telemetry event from a shared IoT sensor maps to **two distinct business resources**.
- Resource 1 processes the message and marks `MessageID = "msg_123"` as processed in the database.
- Resource 2 is evaluated next, sees `"msg_123"` already present in the database, and **silently drops the event**! Resource 2 never receives its telemetry.

### Recommended Pattern: Composite Idempotency Keys
Always define idempotency around the tuple of **`MessageID + DomainResourceID`**:

$$\text{IdempotencyKey} = \text{MessageID} + \text{ResourceID}$$

```go
func (p *EventProcessor) Process(ctx context.Context, ev Event) error {
	// Atomic check-and-insert using composite key
	query := bson.M{
		"messageId": ev.MessageID,
		"ResourceID":  ev.ResourceID,
	}
	update := bson.M{
		"$setOnInsert": bson.M{
			"messageId":   ev.MessageID,
			"ResourceID":    ev.ResourceID,
			"payload":     ev.Payload,
			"processedAt": time.Now(),
		},
	}
	opts := options.Update().SetUpsert(true)

	result, err := p.db.Collection("processed_events").UpdateOne(ctx, query, update, opts)
	if err != nil {
		return err
	}
	if result.UpsertedCount == 0 {
		// Event was already processed for this specific resource; skip idempotently
		return nil
	}

	return p.applyBusinessLogic(ctx, ev)
}
```

---

## 5. Poison Pill Handling & Stale Backlog Draining

### The Failure Mode
Due to a downstream database downtime, a stream accumulates a backlog of 500,000 records over 12 hours. When the database recovers, the consumer attempts to process all 12 hours of historical records sequentially.
- The records represent transient real-time metrics (e.g. live queue sizes, real-time vehicle velocities).
- Processing stale data from 10 hours ago overwrites current operational state with obsolete numbers.
- Draining the backlog takes 8 hours, during which live metrics are delayed.

### Recommended Pattern: Age-Based Stale Record Discarding
For transient real-time metrics, configure a maximum freshness threshold:

```go
const MaxRecordAge = 60 * time.Minute

func ProcessMetric(ctx context.Context, record StreamRecord) error {
	recordAge := time.Since(record.Timestamp)
	if recordAge > MaxRecordAge {
		// Log metric and fast-forward; do not run expensive computations on obsolete data
		metrics.Increment("stale_records_discarded")
		return nil
	}

	return computeLiveMetric(ctx, record)
}
```

---

## 6. Message Transport Envelope Unwrapping

### The Failure Mode
A service consumes a queue (e.g. SQS) subscribed to a publish-subscribe topic (e.g. SNS).
- Locally, unit tests feed direct JSON payloads into the handler: `{"ResourceID": "123", "value": 42}`.
- in practice, the topic wraps the message in a transport notification envelope:
  ```json
  {
    "Type": "Notification",
    "MessageId": "...",
    "TopicArn": "...",
    "Message": "{\"ResourceID\": \"123\", \"value\": 42}"
  }
  ```
- The queue consumer attempts to parse the envelope directly into the domain struct. The parser fails, and messages accumulate in the Dead Letter Queue (DLQ) unnoticed.

### Recommended Pattern: Transparent Envelope Unwrapping
```go
type TransportEnvelope struct {
	Type    string `json:"Type"`
	Message string `json:"Message"`
}

func ExtractPayload(rawBody []byte) ([]byte, error) {
	var env TransportEnvelope
	// Inspect if payload is wrapped in a broker transport envelope
	if err := json.Unmarshal(rawBody, &env); err == nil && env.Type == "Notification" {
		return []byte(env.Message), nil
	}
	// Fall back to direct body
	return rawBody, nil
}
```

---

## 🔍 Agent Action Checklist

- [ ] Does the stream partition key format guarantee Per-Resource FIFO? Format: `{domainType}_{ResourceID}`.
- [ ] Is partial batch failure reporting enabled in consumer handlers?
- [ ] Is downstream idempotency guarded by a composite key (`MessageID + ResourceID`)?
- [ ] Are instantaneous state handovers at the same timestamp coalesced at the producer?
- [ ] Does the consumer unwrap transport notification envelopes?
- [ ] Is there an age threshold to prevent stale backlogs from overwriting live operational data?



