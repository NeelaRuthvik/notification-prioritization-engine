---

# Architecture

## Overview

The Notification Prioritization Engine (NPE) is built as a pipeline of highly cohesive, loosely coupled services backed by Redis and PostgreSQL. The system decouples the *decision* phase (which is synchronous and low-latency) from the *delivery* phase (which is asynchronous), ensuring the API remains highly responsive even if downstream channels degrade.

---

## Component Map

```text
┌─────────────────────────────────────────────────────────────┐
│                        PRODUCERS                            │
│   Messages · Reminders · Alerts · Promos · System Events    │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP POST /v1/notifications/submit
                         ▼
              ┌─────────────────────┐
              │    API Gateway      │  Rate limiting, auth,
              │     / Ingress       │  schema validation,
              └─────────┬───────────┘  idempotency check
                        │
                        ▼
              ┌─────────────────────┐
              │   Dedup Service     │  Bloom filter (O(1) pass)
              │                     │  Redis Hash (exact match)
              └─────────┬───────────┘  SimHash (near-match)
                        │
              ┌─────────┴───────────┐  Parallel Scatter-Gather:
              │ Enrichment Service  │  - User pref lookup
              │    (Concurrent)     │  - Fast ML intent scoring
              └─────────┬───────────┘  - Redis counter fetch
                        │
                        ▼
              ┌─────────────────────┐
              │   Decision Engine   │  In-memory AST rules
              │     (Stateless)     │  Composite score calc
              └──┬──────┬───────┬───┘  Synchronous API Return
                 │      │       │
               NOW    LATER   NEVER ────────┐
                 │      │       │           │
                 ▼      ▼       ▼           ▼
        ┌──────────┐ ┌──────┐ ┌──────────┐ ┌──────────┐
        │ Delivery │ │Sched-│ │ Dead     │ │  Audit   │
        │ Queue    │ │ uler │ │ Letter   │ │  Logger  │
        └────┬─────┘ └──────┘ └──────────┘ └──────────┘
             │          │                        ▲
        ┌────▼─────┐ Redis ZSET                  │
        │ Delivery │    │                        │
        │Dispatcher│    │ (re-enters pipeline    │
        └────┬─────┘    │  at deferUntil time)   │
             │          │                        │
        push/email/ ────┴────────────────────────┘
        SMS/in-app     (Async Status Updates)

```

---

## Services

### 1. API Gateway / Ingress

* Authenticates callers via service-to-service JWT.
* Validates event schema and rejects malformed events (422).
* Checks `event_id` against a short-lived Redis cache for strict idempotent replays (returns cached 200 OK immediately if matched).
* Normalizes fields (e.g., coerces `priority_hint` to enum, timestamps to UTC ISO8601).
* **Latency budget**: < 5ms

### 2. Dedup Service

* Maintains a Redis Bloom filter for an O(1) first-pass dedup check.
* On bloom hit → confirms with Redis hash lookup (TTL = dedup window, default 24h).
* Generates canonical hash when `dedupe_key` is absent: `SHA-256(user_id + event_type + source + normalize(title))`.
* Near-duplicate detection via SimHash 64-bit fingerprint against a rolling window of recent user messages (Hamming distance ≤ 3).
* **Latency budget**: < 5ms

### 3. Enrichment Service

* Executes a parallel scatter-gather pattern to minimize latency.
* Fetches user quiet-hours window and channel preferences from the Postgres replica (or local cache).
* Queries a lightweight ML classifier (e.g., XGBoost or small BERT) for intent score (0.0–1.0).
* Retrieves sliding-window notification counters from Redis.
* Hard timeout: **30ms**. On timeout or failure → circuit breaks and proceeds with `ai_intent_score = null`, `ai_used = false`.
* **Latency budget**: < 35ms total

### 4. Decision Engine

* Stateless rule evaluator — acts as the brain of the pipeline.
* Compiles rules into an in-memory Abstract Syntax Tree (AST) for microsecond evaluation (refreshed every 30s from Postgres).
* Evaluates rules in strict priority order.
* Returns the `Decision` object to the API Gateway to close the synchronous HTTP request, while simultaneously routing the payload to the appropriate downstream async queue (Delivery or Scheduler).
* **Latency budget**: < 5ms

### 5. Scheduler Service

* Receives LATER decisions with a `defer_until` timestamp.
* Stores deferred events in a Redis Sorted Set (ZSET) keyed by epoch timestamp.
* High-frequency worker processes use `ZRANGEBYSCORE` to poll for due events and push them back into the Ingress queue.
* Checks `expires_at` before re-submission — expired events are routed directly to the Audit Logger as NEVER.
* **Latency budget**: Background process; < 500ms delivery lag.

### 6. Delivery Dispatcher

* Consumes from the asynchronous Delivery Queue.
* Fans out to appropriate external provider APIs (APNs, FCM, SendGrid, Twilio).
* Retry logic: 3 attempts with exponential backoff (max 30s total jitter).
* On success, fires an async event to the Audit Logger to append `delivered_at`.
* On 3 failures → marks `DELIVERY_FAILED`, enqueues to dead-letter, and updates metrics.
* **Latency budget**: Async; dependent on external vendor APIs.

### 7. Audit Logger

* Consumes decision and delivery events from a high-throughput message bus (or Redis Streams).
* Batches inserts (e.g., every 500ms or 1000 records) to an append-only Postgres table to prevent database IO bottlenecks.
* Includes: outcome, reasons[], score, rule_version, ai_used, decided_at, delivered_at.

---

## Data Stores

| Store | Used For | Notes |
| --- | --- | --- |
| **Redis (Cluster)** | Dedup bloom filter, exact-match hashes, sliding-window fatigue counters, deferred event ZSETs, idempotency cache. | Partitioned by `user_id` to ensure operations for a single user map to the same shard. |
| **PostgreSQL** | NotificationRule config, DecisionRecord audit log, persistent user preferences. | Standby replica for HA. Read-heavy operations (rules, prefs) are heavily cached in-memory. |

---

## Deployment Topology Notes

* **Decision Engine & API:** Horizontally scalable; run N replicas behind an internal load balancer. Completely stateless.
* **Enrichment Service:** Scales independently. The AI model is deployed as a sidecar container via gRPC to avoid network hop latency, ensuring the 30ms timeout is achievable.
* **Scheduler Service:** Runs as a multi-worker deployment utilizing Redis distributed locks (Redlock) or consumer groups to ensure a deferred event is only processed by one worker.
* **Rules Cache:** Local per Decision Engine pod, refreshed via an asynchronous background thread to prevent latency spikes during cache misses.

---

## Performance Targets

| Metric | Target | Notes |
| --- | --- | --- |
| API Sync Latency P50 | < 30ms | Time to return decision to caller. |
| API Sync Latency P99 | < 75ms | Accounts for AI model tail latencies. |
| Throughput | 10,000+ events/sec | Bound only by Redis cluster shard capacity. |
| Dedup False-Positive Rate | < 0.01% | Tuned via Bloom filter sizing. |
| Scheduler Polling Lag | < 500ms | Time from `defer_until` to actual dispatch. |

---

Would you like to move on to refining the "Minimal Data Model" or the "Decision Logic / AI Fallback" strategy next?
