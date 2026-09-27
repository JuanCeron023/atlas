# 03. Temporal Data, Time Windows & Boundary Mathematics

> **Time Window Modeling, Half-Open Intervals, Timezone Localization, and Rolling Aggregation Invariants.**

---

## 1. Non-Standard & Extended Operating Windows (Past Midnight)

### The Failure Mode
In 24/7 operations, logistics, shifts, transport, and hospitality, an operational "work day" or "shift" frequently does not end at calendar midnight (`23:59:59`). Instead, operations extend past midnight into the early morning of the next calendar day (e.g. `06:00` until `03:00` of the following day—a 27-hour operational window).

When software parses these operational times using standard library time parsers:
```go
// ANTI-PATTERN: Fails when hours extend past 23:59
t, err := time.Parse("15:04", "25:30") // err: parsing time "25:30": hour out of range [0, 23]
```
Standard ISO 8601 and RFC 3339 parsers reject hours $\ge 24$. Systems fail with unhandled 400 errors or misallocate post-midnight operations into the wrong logical business shift.

### Recommended Pattern: Explicit Logical Window Representation
1. **Decouple Logical Window from Calendar Date**: Store the pair:
   - `LogicalDate`: Date string formatted as `YYYY-MM-DD` representing the logical operational day.
   - `MinuteOfShift`: Integer representing total minutes from shift start (e.g. `0` to `1620`), or a duration offset.

```go
package temporal

import (
	"fmt"
	"strconv"
	"strings"
)

type ShiftTime struct {
	Hour   int
	Minute int
}

// ParseExtendedTime parses strings formatted as "HH:MM", permitting hours up to 28:00
func ParseExtendedTime(s string) (ShiftTime, int, error) {
	parts := strings.Split(s, ":")
	if len(parts) != 2 {
		return ShiftTime{}, 0, fmt.Errorf("invalid time format: %s", s)
	}

	h, err := strconv.Atoi(parts[0])
	if err != nil || h < 0 || h > 28 {
		return ShiftTime{}, 0, fmt.Errorf("hour out of range [0, 28]: %d", h)
	}

	m, err := strconv.Atoi(parts[1])
	if err != nil || m < 0 || m > 59 {
		return ShiftTime{}, 0, fmt.Errorf("minute out of range [0, 59]: %d", m)
	}

	totalMinutes := (h * 60) + m
	return ShiftTime{Hour: h, Minute: m}, totalMinutes, nil
}
```

---

## 2. Timezone Localization vs Server UTC Boundaries

### The Failure Mode
A backend service running in an AWS/GCP region configured with UTC evaluates active alerts or queries shifts using `time.Now().UTC()`.
- If an resource is located in a UTC-5 timezone, at 01:00 AM local time, the server UTC time is 06:00 AM on the **next calendar day**.
- A query asking for "today's metrics" resolves to tomorrow's window.
- Operations that should be recorded against the current shift are incorrectly partitioned into the future.

### Recommended Pattern: Local Timezone Fetch & Midday Anchoring
1. **Always calculate in the resource's local timezone**: Every resource or facility must have an authoritative timezone attribute (e.g. `America/New_York`, `Europe/London`).
2. **Anchor reference queries to midday UTC**: When requesting date-scoped operational data from an upstream service, anchor the query timestamp to `12:00:00 UTC` or `15:00:00 UTC` of the logical date. Midday UTC guarantees that the timestamp falls within the operational day across worldwide timezones, eliminating boundary jitter.

```go
func GetOperatingReferenceDate(logicalDate string) (time.Time, error) {
	// logicalDate formatted as "YYYY-MM-DD"
	t, err := time.Parse("2006-01-02", logicalDate)
	if err != nil {
		return time.Time{}, err
	}
	// Anchor at 12:00 UTC to remain safely within the operational day across timezones
	return time.Date(t.Year(), t.Month(), t.Day(), 12, 0, 0, 0, time.UTC), nil
}
```

---

## 3. Half-Open Interval Math: The Inclusive Boundary Delete Bug

### The Failure Mode
An operational segment is modified, adjusting its start time from `08:00` to `08:15`. A cleanup routine runs to purge invalidated pre-calculated buckets:
```javascript
// BUG: Using $lte on the upper boundary
db.buckets.deleteMany({
    start_time: { $gte: "08:00", $lte: "08:15" }
})
```
- Because `$lte` (less than or equal) was used, the bucket starting at **`08:15`** is deleted!
- But `08:15` is the first valid bucket of the **new segment**!
- As a consequence, capacity counts for the day temporarily drop until an audit worker recalculates the missing unit.

### The Invariant: Always Use Half-Open Intervals $[start, end)$
In continuous time-series data, time intervals must strictly follow half-open semantics:

$$I = [t_{\text{start}}, t_{\text{end}}) \implies t \in \{ t \mid t_{\text{start}} \le t < t_{\text{end}} \}$$

```javascript
// CORE INVARIANT: Upper boundary MUST use $lt
db.buckets.deleteMany({
    start_time: { 
        $gte: "08:00", 
        $lt:  "08:15"   // Exclusive: preserves bucket starting at 08:15
    }
})
```

---

## 4. Rolling Window Aggregations Across Discontinuous Segments

### The Failure Mode
A system computes a "Rolling 60-Minute Sum" (e.g. transaction throughput over the preceding hour).
An resource operates in split shifts:
- Shift 1: 08:00 to 12:00.
- Gap: 12:00 to 14:00 (offline).
- Shift 2: 14:00 to 20:00.

**Two critical failure modes emerge:**
1. **Resetting at Boundaries**: If Shift 2 is evaluated independently, the rolling sum at 14:15 only includes 15 minutes of data, resetting historical context.
2. **Index-Based vs Time-Aware Lookback**: If the calculator implements lookback by indexing 60 elements back in an array (`slice[i-60]`), the 2-hour offline gap is ignored! The calculation grabs measurements from 11:30, treating them as if they occurred 30 minutes ago rather than 2.5 hours ago!

### Recommended Pattern: Time-Aware Rolling Flattening
1. Flatten all operational segments into a unified chronological time axis.
2. Pad operational gaps with explicit `0` count entries.
3. Compute the window using a timestamp delta ($\Delta t = 60\text{m}$), never array indices.

```go
type TimeMetric struct {
	Timestamp time.Time
	Value     float64
}

// ComputeRollingSum calculates time-aware rolling window with gap padding
func ComputeRollingSum(timeline []TimeMetric, windowDuration time.Duration) []float64 {
	results := make([]float64, len(timeline))
	left := 0
	currentSum := 0.0

	for right := 0; right < len(timeline); right++ {
		currentSum += timeline[right].Value

		// Evict entries that fall outside the time window
		for timeline[right].Timestamp.Sub(timeline[left].Timestamp) >= windowDuration {
			currentSum -= timeline[left].Value
			left++
		}
		results[right] = currentSum
	}
	return results
}
```

---

## 5. Deterministic Secondary Sorting on Database Timelines

### The Failure Mode
An aggregation pipeline sorts events to construct a timeline:
```javascript
db.events.aggregate([
    { $sort: { "timestamp": 1 } }
])
```
When two events have the exact same timestamp (e.g. batch ingestion at millisecond precision), document storage engines **do not guarantee deterministic ordering**.
On subsequent queries or across replica set secondaries, the returned order flips, causing nondeterministic timeline reconstruction, sporadic target recalculations, and test flakiness.

### The Invariant: Always Include a Unique Tie-Breaker Key
```javascript
// CORE INVARIANT: Always append a unique identifier to sorts
db.events.aggregate([
    { $sort: { "timestamp": 1, "_id": 1 } }
])
```

---

## 6. Inclusive Expiration Boundaries

### The Failure Mode
A configuration or access rule has:
- `effectiveDate: "2026-09-01"`
- `expirationDate: "2026-09-27"`

A query checking if the rule is active on `2026-09-27` evaluates:
```go
// ANTI-PATTERN: Strict inequality on expiration date!
if today.After(rule.EffectiveDate) && today.Before(rule.ExpirationDate) { ... }
```
`today.Before("2026-09-27")` returns `false` on the same day! The rule expires a full 24 hours early.

### The Invariant: Inclusive Same-Day Validity
In domain configurations, expiration dates are **inclusive** of the final day:

$$\text{Active}(t) \iff \text{effectiveDate} \le t \land (\text{expirationDate} = \text{null} \lor \text{expirationDate} \ge t)$$

```go
func IsConfigActive(targetDate, effectiveDate string, expirationDate *string) bool {
	if targetDate < effectiveDate {
		return false
	}
	if expirationDate != nil && *expirationDate != "" {
		if targetDate > *expirationDate {
			return false
		}
	}
	return true
}
```

---

## 🔍 Agent Action Checklist

- [ ] Does time parsing support hours $\ge 24$ when handling non-standard shifts or extended windows?
- [ ] Are date-scoped lookups calculated in the resource's local timezone or anchored to midpoint UTC?
- [ ] Are interval deletions using half-open `$lt` upper bounds instead of `$lte`?
- [ ] Do database timeline sorts include a deterministic secondary tie-breaker (`_id: 1`)?
- [ ] Are expiration date checks inclusive of the final boundary date (`expirationDate >= targetDate`)?
- [ ] Are rolling window aggregations time-aware rather than array-index-based?



