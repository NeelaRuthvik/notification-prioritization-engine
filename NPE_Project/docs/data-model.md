# Data Model

## Entity Overview

```
NotificationEvent  ──(1:1)──▶  DecisionRecord
                                     │
                              (references)
                                     │
                              NotificationRule (used at decision time; version snapshotted)

UserNotificationState  (Redis, per-user TTL keys — referenced during evaluation)
```

---

## 1. NotificationEvent

**Storage:** In-memory during pipeline; persisted to Postgres for audit if needed.  
**Purpose:** Canonical normalized input to the engine.

| Field | Type | Required | Constraints | Notes |
|-------|------|----------|-------------|-------|
| `event_id` | UUID | ✅ | Unique | System-generated on ingestion; idempotency key |
| `user_id` | String | ✅ | Max 128 chars | Target user |
| `event_type` | Enum | ✅ | See below | Notification category |
| `title` | String | ✅ | Max 120 chars | Short notification title |
| `message` | String | ❌ | Max 1000 chars | Body text |
| `source` | String | ✅ | Max 64 chars | Originating service or feature name |
| `priority_hint` | Enum | ❌ | See below | Caller-supplied urgency hint |
| `channel` | Enum[] | ✅ | Non-empty array | Preferred delivery channels |
| `timestamp` | ISO8601 | ✅ | UTC | Event creation time |
| `expires_at` | ISO8601 | ❌ | UTC; must be > timestamp | Discard if undelivered past this time |
| `dedupe_key` | String | ❌ | Max 256 chars | Caller-provided dedup hint; may be absent or unreliable |
| `metadata` | JSON | ❌ | Max 4KB | Arbitrary context: `{ incident: true, thread_id: "...", }` |

**event_type enum values:**
`MESSAGE`, `REMINDER`, `ALERT`, `PROMO`, `SYSTEM`, `UPDATE`, `SECURITY`, `DIGEST`

**priority_hint enum values:**
`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`

**channel enum values:**
`push`, `email`, `sms`, `in_app`

---

## 2. DecisionRecord

**Storage:** PostgreSQL — append-only (no UPDATE/DELETE).  
**Purpose:** Full audit trail of every decision made by the engine.

| Field | Type | Notes |
|-------|------|-------|
| `decision_id` | UUID | Primary key, system-generated |
| `event_id` | UUID | FK → NotificationEvent (or inline) |
| `user_id` | String | Denormalized for fast per-user queries |
| `outcome` | Enum | `NOW` / `LATER` / `NEVER` |
| `reasons` | String[] | Ordered list of reason codes e.g. `["FATIGUE_CAP_1H","QUIET_HOURS"]` |
| `score` | Float | Composite priority score; null if short-circuited at P0–P3 |
| `defer_until` | ISO8601 | Populated when outcome = LATER |
| `defer_count` | Integer | How many times this event has been deferred (max 2 before forced delivery) |
| `rule_version` | String | Snapshot of rules config version active at decision time |
| `ai_used` | Boolean | Whether AI enrichment contributed to this decision |
| `ai_score` | Float | Raw AI intent score; null if ai_used = false |
| `decided_at` | ISO8601 | Timestamp of decision |
| `delivered_at` | ISO8601 | Null until confirmed delivery; updated by Dispatcher |
| `delivery_channel` | Enum | Actual channel used for delivery |
| `delivery_status` | Enum | `PENDING` / `DELIVERED` / `DELIVERY_FAILED` / `SUPPRESSED` |

**Indexes:**
- `(user_id, decided_at DESC)` — per-user history queries
- `(event_id)` — idempotency lookup
- `(outcome, decided_at)` — reporting / dashboard
- `(delivery_status)` — operations / support queries

---

## 3. NotificationRule

**Storage:** PostgreSQL.  
**Purpose:** Human-configurable routing rules, evaluated by the Decision Engine.

| Field | Type | Notes |
|-------|------|-------|
| `rule_id` | UUID | Primary key |
| `name` | String | Human-readable label |
| `description` | String | Optional explanation of intent |
| `priority` | Integer | Unique; lower = evaluated first; enforced by DB unique constraint |
| `conditions` | JSONB | Predicate tree — see schema below |
| `action` | JSONB | Outcome block — see schema below |
| `enabled` | Boolean | Toggle without deletion |
| `version` | String | Semantic version e.g. `"1.3.0"` |
| `created_by` | String | Operator identity |
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

All keys are TTL-based. No persistent storage needed — counters naturally expire.

| Key Pattern | Value Type | TTL | Purpose |
|-------------|-----------|-----|---------|
| `npe:count:{user_id}:5m` | Sorted Set (timestamps) | 5 minutes | Sliding 5-min notification counter |
| `npe:count:{user_id}:1h` | Sorted Set (timestamps) | 1 hour | Sliding 1-hour counter |
| `npe:count:{user_id}:24h` | Sorted Set (timestamps) | 24 hours | Sliding 24-hour counter |
| `npe:dedup:hash:{sha256}` | String `"1"` | Configurable (default 30min) | Exact dedup presence flag |
| `npe:dedup:simhash:{fp}` | String `"1"` | Configurable (default 30min) | Near-dup fingerprint flag |
| `npe:suppress:{user_id}:{event_type}` | ISO8601 string | Per-type cooldown duration | Per-type cooldown expiry marker |
| `npe:incident:{user_id}` | String `"1"` | Incident TTL (default 2h) | Active incident suppression flag |
| `npe:defer:{event_id}` | JSON string (full event) | Until `expires_at` | Deferred event payload storage |
| `npe:defer_queue` | Sorted Set (score = epoch) | Persistent (rolling) | Deferred delivery schedule |
| `npe:bloom` | Bloom filter | Persistent (rotated at 70% fill) | Fast O(1) first-pass dedup check |

---

## 5. Bloom Filter Management

- **Library:** Redis `BF.ADD` / `BF.EXISTS` (RedisBloom module)
- **Parameters:** Tuned for 0.1% false-positive rate at 10M daily unique keys
- **Rotation:** At 70% fill ratio, a new filter is initialized. Old filter remains read-only for the remaining TTL overlap window (30 minutes)
- **Monitoring:** `BF.INFO` exposes current fill ratio; alert threshold at 60%

---

## Schema Diagram (Simplified)

```
NotificationEvent
  event_id (PK)
  user_id
  event_type
  title, message, source
  priority_hint
  channel[]
  timestamp, expires_at
  dedupe_key, metadata
        │
        │ 1:1
        ▼
DecisionRecord
  decision_id (PK)
  event_id (FK)
  user_id
  outcome
  reasons[]
  score, ai_score, ai_used
  defer_until, defer_count
  rule_version
  decided_at, delivered_at
  delivery_status

NotificationRule
  rule_id (PK)
  priority (UNIQUE)
  conditions (JSONB)
  action (JSONB)
  enabled, version
  created_at, updated_at
```
