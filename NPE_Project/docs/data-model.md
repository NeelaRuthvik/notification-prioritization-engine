---
# Data Model

This document describes the core entities used in the Notification Prioritization Engine (NPE).

---

# 1️⃣ Entity Overview

```mermaid
graph TD

NotificationEvent -->|1:1| DecisionRecord
DecisionRecord -->|references (version snapshotted at decision time)| NotificationRule
DecisionRecord -. reads state .-> UserNotificationState[(UserNotificationState<br/>Redis - per-user TTL keys)]
---

## 1. NotificationEvent

**Storage:** Ephemeral in-memory object during pipeline execution; serialized into the `DecisionRecord` payload for asynchronous audit storage.

**Purpose:** Canonical normalized input to the engine.

| Field | Type | Required | Constraints | Notes |
| --- | --- | --- | --- | --- |
| `event_id` | UUID | ✅ | Unique | System-generated or caller-supplied; idempotency key |
| `user_id` | String | ✅ | Max 128 chars | Target user identifier |
| `event_type` | Enum | ✅ | See below | Notification category |
| `title` | String | ✅ | Max 120 chars | Short notification title |
| `message` | String | ❌ | Max 1000 chars | Body text |
| `source` | String | ✅ | Max 64 chars | Originating service or feature name |
| `priority_hint` | Enum | ❌ | See below | Caller-supplied urgency hint |
| `channel` | Enum[] | ✅ | Non-empty array | Preferred delivery channels |
| `timestamp` | ISO8601 | ✅ | UTC | Event creation time |
| `expires_at` | ISO8601 | ❌ | UTC; must be > timestamp | Discard if undelivered past this time |
| `dedupe_key` | String | ❌ | Max 256 chars | Caller-provided dedup hint; may be absent or unreliable |
| `metadata` | JSONB | ❌ | Max 4KB | Arbitrary context: `{ incident: true, thread_id: "..." }` |

**event_type enum values:**
`MESSAGE`, `REMINDER`, `ALERT`, `PROMO`, `SYSTEM`, `UPDATE`, `SECURITY`, `DIGEST`

**priority_hint enum values:**
`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`

**channel enum values:**
`push`, `email`, `sms`, `in_app`

---

## 2. DecisionRecord

**Storage:** PostgreSQL — append-only. **Table is time-partitioned (e.g., daily or weekly) to allow efficient bulk deletion of old audit logs.** **Purpose:** Full audit trail of every decision made by the engine for observability and support.

| Field | Type | Notes |
| --- | --- | --- |
| `decision_id` | UUID | Primary key, system-generated |
| `event_id` | UUID | Indexed for idempotency / support lookups |
| `user_id` | String | Denormalized for fast per-user history queries |
| `outcome` | Enum | `NOW` / `LATER` / `NEVER` |
| `reasons` | String[] | Ordered list of reason codes e.g. `["FATIGUE_CAP_1H","QUIET_HOURS"]` |
| `matched_rule_id` | UUID | FK → NotificationRule (null if fallback/AI decided) |
| `score` | Float | Composite priority score; null if short-circuited |
| `defer_until` | ISO8601 | Populated when outcome = LATER |
| `defer_count` | Integer | How many times this event has been deferred (max 2) |
| `rule_version` | String | Snapshot of rules config version active at decision time |
| `ai_used` | Boolean | Whether AI enrichment contributed to this decision |
| `ai_score` | Float | Raw AI intent score; null if ai_used = false |
| `decided_at` | ISO8601 | Timestamp of decision (Partition Key) |
| `delivered_at` | ISO8601 | Null until confirmed delivery; updated asynchronously |
| `delivery_channel` | Enum | Actual channel used for delivery |
| `delivery_status` | Enum | `PENDING` / `DELIVERED` / `DELIVERY_FAILED` / `SUPPRESSED` |

**Indexes:**

* `(user_id, decided_at DESC)` — per-user history queries
* `(event_id)` — unique constraint & exact lookup
* `(delivery_status)` — operations / dead-letter queries

---

## 3. NotificationRule

**Storage:** PostgreSQL. Cached in-memory by Decision Engine instances.

**Purpose:** Human-configurable routing rules, evaluated dynamically.

| Field | Type | Notes |
| --- | --- | --- |
| `rule_id` | UUID | Primary key |
| `name` | String | Human-readable label |
| `description` | String | Optional explanation of intent |
| `priority` | Integer | Unique; lower = evaluated first; enforced by DB unique constraint |
| `conditions` | JSONB | Predicate tree — see schema below |
| `action` | JSONB | Outcome block — see schema below |
| `enabled` | Boolean | Toggle without deletion |
| `version` | String | Semantic version e.g. `"1.3.0"` |
| `updated_by` | String | Operator identity (for compliance/audit) |
| `created_at` | ISO8601 | Creation timestamp |
| `updated_at` | ISO8601 | Last modification timestamp |

### Conditions Schema (JSONB)

```json
{
  "op": "AND",
  "rules": [
    { "field": "event_type", "operator": "eq", "value": "PROMO" },
    { "field": "priority_hint", "operator": "in", "value": ["LOW", null] }
  ]
}

```

Supported operators: `eq`, `neq`, `in`, `not_in`, `gt`, `lt`, `gte`, `lte`, `exists`, `not_exists`

Logical combinators: `AND`, `OR`, `NOT`

### Action Schema (JSONB)

```json
{
  "outcome": "LATER",
  "defer_strategy": "next_morning",
  "channel_override": null,
  "reason_code": "PROMO_DEFER_QUIET"
}

```

`defer_strategy` values: `fixed_delay_15m`, `next_hour`, `next_morning`, `next_quiet_end`, `digest_window`

---

## 4. UserNotificationState (Redis)

All keys are TTL-based. No persistent storage is needed for these execution states.

| Key Pattern | Value Type | TTL | Purpose |
| --- | --- | --- | --- |
| `npe:idem:{event_id}` | String | 24 hours | Fast API Gateway idempotency check |
| `npe:count:{user_id}:5m` | Sorted Set (timestamps) | 5 minutes | Sliding 5-min notification counter |
| `npe:count:{user_id}:1h` | Sorted Set (timestamps) | 1 hour | Sliding 1-hour counter |
| `npe:count:{user_id}:24h` | Sorted Set (timestamps) | 24 hours | Sliding 24-hour counter |
| `npe:dedup:hash:{sha256}` | String `"1"` | Configurable (24h) | Exact dedup presence flag |
| `npe:dedup:simhash:{fp}` | String `"1"` | Configurable (24h) | Near-dup fingerprint flag |
| `npe:suppress:{user_id}:{type}` | String | Cooldown duration | Per-type cooldown expiry marker |
| `npe:defer:{event_id}` | JSON string (event) | Until `expires_at` | Deferred event payload storage |
| `npe:defer_queue` | Sorted Set (score = epoch) | Persistent | Deferred delivery schedule |
| `npe:bloom` | Bloom filter | Rotated | Fast O(1) first-pass dedup check |

---

## 5. Bloom Filter Management

* **Library:** Redis `BF.ADD` / `BF.EXISTS` (RedisBloom module)
* **Parameters:** Tuned for a 0.01% false-positive rate at expected daily volume.
* **Rotation:** At 70% capacity, a new filter is initialized. The old filter remains active for read-only checks for the duration of the deduplication window (e.g., 24 hours) to prevent state loss during rotation.
* **Monitoring:** `BF.INFO` exposes current capacity; alerts trigger at 60%.

---

## Schema Diagram (Simplified)

```text
NotificationEvent (In-Memory Payload)
  event_id
  user_id
  event_type
  title, message, source
  priority_hint
  channel[]
  timestamp, expires_at
  dedupe_key, metadata
        │
        │ Serialized into
        ▼
DecisionRecord (Postgres - Partitioned)
  decision_id (PK)
  event_id (IDX)
  user_id (IDX)
  outcome
  reasons[]
  matched_rule_id (FK)
  score, ai_score, ai_used
  defer_until, defer_count
  rule_version
  decided_at (Partition Key)
  delivered_at
  delivery_status

NotificationRule (Postgres - Cached)
  rule_id (PK)
  priority (UNIQUE)
  conditions (JSONB)
  action (JSONB)
  enabled, version
  created_at, updated_at, updated_by

```

---

What section of the solution would you like to review next? (If you are pasting text, making sure it's copied as "plain text" without formatting can help avoid those parsing errors).
