# Architecture

## Overview

The Notification Prioritization Engine (NPE) is built as a pipeline of six loosely-coupled services backed by two shared data stores. Events flow from ingestion through deduplication, optional AI enrichment, rule-based decision, and finally delivery or deferral.

---

## Component Map

```
┌─────────────────────────────────────────────────────────────┐
│                        PRODUCERS                            │
│   Messages · Reminders · Alerts · Promos · System Events   │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP POST /v1/notifications/submit
                         ▼
              ┌─────────────────────┐
              │    API Gateway      │  Rate limiting, auth,
              │    / Ingress        │  schema validation,
              └─────────┬───────────┘  normalization
                        │
                        ▼
              ┌─────────────────────┐
              │   Dedup Service     │  Bloom filter (fast path)
              │                     │  Redis hash (slow path)
              └─────────┬───────────┘  SimHash near-dup check
                        │
              ┌─────────┴───────────┐
              │  Enrichment Service │  User prefs lookup
              │  (async, optional)  │  AI intent scoring
              └─────────┬───────────┘  30ms timeout → fallback
                        │
                        ▼
              ┌─────────────────────┐
              │   Decision Engine   │  P0–P8 rule evaluation
              │   (stateless)       │  Composite score calc
              └──┬──────┬───────┬───┘  Reason chain assembly
                 │      │       │
               NOW     LATER   NEVER
                 │      │       │
                 ▼      ▼       ▼
        ┌──────────┐ ┌──────┐ ┌──────────┐
        │ Delivery │ │Sched-│ │  Audit   │
        │Dispatcher│ │ uler │ │  Logger  │
        └──────────┘ └──────┘ └──────────┘
            │            │
       push/email/    Redis sorted
       SMS/in-app     set (deferred)
                          │
                     re-enters pipeline
                     at deferUntil time
```

---

## Services

### 1. API Gateway / Ingress
- Authenticates callers via service-to-service JWT
- Validates event schema and rejects malformed events with 422
- Normalizes fields (e.g., coerces priority_hint to enum, timestamps to UTC ISO8601)
- Rate limits per source service to prevent ingestion storms
- **Latency budget**: < 5ms

### 2. Dedup Service
- Maintains a Redis Bloom filter for O(1) first-pass dedup check
- On bloom hit → confirms with Redis hash lookup (TTL = dedup window, default 30min)
- Generates canonical hash when `dedupe_key` is absent: `SHA-256(user_id + event_type + source + normalize(title))`
- Near-duplicate detection via SimHash 64-bit fingerprint (Hamming distance ≤ 3)
- **Latency budget**: < 5ms

### 3. Enrichment Service
- Fetches user quiet-hours window and channel preferences from user store
- Queries AI model for intent score (0.0–1.0)
- Retrieves sliding-window notification counters from Redis
- Hard timeout: **30ms**. On timeout or error → proceeds with `ai_intent_score = 0.5`, `ai_used = false`
- **Latency budget**: < 20ms (parallel with preference lookup)

### 4. Decision Engine
- Stateless rule evaluator — all state lives in Redis and Postgres
- Loads compiled rule set from in-memory cache (refreshed every 30s from Postgres)
- Evaluates rules in strict priority order (P0–P8)
- Emits a `Decision` object: `{ outcome, reasons[], score, defer_until?, channel_overrides }`
- **Latency budget**: < 15ms

### 5. Scheduler Service
- Receives LATER decisions with a `defer_until` timestamp
- Stores deferred events in a Redis sorted set keyed by epoch timestamp
- Background poller (tick every 5s) promotes due events back into the pipeline
- Checks `expires_at` before re-submission — expired events become NEVER at delivery time
- **Latency budget**: background; < 1s polling lag

### 6. Delivery Dispatcher
- Fans out to appropriate channels: push, email, SMS, in-app
- Retry logic: 3 attempts with exponential backoff (max 30s total)
- Records `delivered_at` timestamp in DecisionRecord on success
- On 3 failures → marks DELIVERY_FAILED, enqueues to dead-letter, alerts on-call
- **Latency budget**: < 30ms per channel attempt

### 7. Audit Logger
- Appends every DecisionRecord asynchronously (non-blocking to main path)
- Writes to append-only Postgres table (no UPDATE/DELETE permitted)
- Includes: outcome, reasons[], score, rule_version, ai_used, decided_at, delivered_at

---

## Data Stores

| Store | Used For | Notes |
|-------|----------|-------|
| **Redis** | Dedup bloom filter, dedup hash index, sliding-window counters, per-type cooldowns, deferred event queue, circuit breaker state | Redis Sentinel or Cluster for HA |
| **PostgreSQL** | NotificationRule config, DecisionRecord audit log, user preferences | Standby replica for HA; read from cache in fallback |

---

## Deployment Topology Notes

- Decision Engine is stateless → horizontally scalable; run N replicas behind load balancer
- Enrichment Service scales independently; AI model can be a sidecar or external microservice
- Scheduler Service runs as a single-leader pod (use distributed lock in Redis) to prevent duplicate deliveries
- Bloom filter is shared across all Decision Engine replicas via Redis — single source of truth
- Rules cache is local per Decision Engine pod, refreshed on TTL (30s) or on rule update webhook

---

## Performance Targets

| Metric | Target |
|--------|--------|
| End-to-end decision latency P50 | < 50ms |
| End-to-end decision latency P99 | < 200ms |
| Throughput | 5,000+ events/sec (horizontally scalable) |
| Dedup false-positive rate | < 0.1% |
| Scheduler polling lag | < 1 second |
