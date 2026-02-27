# Fallback Strategy

## Philosophy

The NPE is designed around one core safety guarantee:

> **A CRITICAL or SECURITY notification must never be silently lost, regardless of what fails.**

Every other design decision in this document is subordinate to that guarantee. For non-critical notifications, the fallback preference order is: **degrade gracefully → reduce accuracy → skip optional steps → log the gap**.

---

## Dependency Map

```
┌─────────────────────────────────────────────────────────────────┐
│                        NPE Decision Engine                      │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────────┐  │
│  │ AI Enrichment│   │    Redis     │   │ Postgres (Rules)   │  │
│  │  Service     │   │  (Dedup,     │   │ (Rules, Audit)     │  │
│  │  (optional)  │   │  Counters,   │   │                    │  │
│  │              │   │  Queue)      │   │                    │  │
│  └──────┬───────┘   └──────┬───────┘   └─────────┬──────────┘  │
│         │                  │                     │             │
│     OPTIONAL           IMPORTANT             IMPORTANT         │
│     30ms timeout       HA required           HA + cache        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Failure Mode Table

| Failure | Detection | Immediate Fallback | Degraded Behavior | Recovery |
|---------|-----------|-------------------|-------------------|----------|
| **AI Enrichment slow** | HTTP response > 30ms | Proceed with `ai_intent_score = 0.5` | Decision made without AI signal; `ai_used = false` in audit | Circuit breaker half-opens after 10s; resumes on first success |
| **AI Enrichment down** | Circuit breaker open | Same as above | Same as above | Circuit breaker monitors health endpoint; auto-resumes |
| **Redis unavailable** | Connection error on any Redis op | Skip dedup + counters; log `REDIS_FALLBACK` reason | Duplicates may pass through; fatigue caps disabled; response includes `fallback_mode: true` | Redis Sentinel/Cluster auto-failover; ops alert via P2 |
| **Redis partially degraded** | Timeout on specific key ops | Per-operation timeout (5ms); skip that operation only | Subset of safeguards disabled; logged per decision | Automatic on Redis recovery |
| **Postgres (Rules) unavailable** | Query timeout > 100ms | Serve from in-memory rules cache (last-good snapshot) | Rules up to 60s stale; no new rule changes take effect | Postgres HA standby replica; cache refreshes on reconnect |
| **Rules cache stale > 120s** | Cache age monitor | Alert P3; continue serving stale cache | Rules may not reflect recent changes | Ops intervention; force cache refresh endpoint |
| **Scheduler Service down** | Kubernetes health probe failure | LATER decisions queued in Redis list (`npe:defer_fallback_queue`) with full event payload | Deferred events accumulate; delivered after scheduler recovers | Kubernetes restarts pod automatically; Redis list replayed |
| **Delivery Dispatcher failure** | HTTP 5xx / timeout per channel | Retry with exponential backoff: 3 attempts, max 30s total | After 3 failures → `DELIVERY_FAILED` in audit; on-call alerted | Dead-letter queue (DLQ) captures failed events for forensics and manual replay |
| **Entire NPE unavailable** | All health checks failing | Emergency bypass path: callers may deliver CRITICAL events directly via documented emergency channel config | Non-critical notifications queued at source | Restore from last checkpoint; replay events from durable queue |

---

## Circuit Breaker Configuration

All external calls use a circuit breaker with the **half-open** pattern:

```
State Machine:
  CLOSED ──[5 failures in 10s]──▶ OPEN ──[10s timeout]──▶ HALF-OPEN
     ▲                                                          │
     └────────────[1 successful probe]────────────────────────┘
                                              │
                              [probe fails]───▶ OPEN (restart timer)
```

| Dependency | Timeout | Failure Threshold | Open Duration |
|------------|---------|-------------------|---------------|
| AI Enrichment | 30ms | 5 failures / 10s | 10s |
| Redis | 5ms per op | 5 failures / 10s | 5s |
| Postgres (Rules) | 100ms | 3 failures / 10s | 15s |
| Delivery channels | 2000ms | 5 failures / 60s | 30s |

Circuit breaker state is stored in Redis and shared across all Decision Engine replicas. Circuit state changes are logged and surfaced in the monitoring dashboard.

---

## Critical Notification Protection

CRITICAL and SECURITY notifications follow a **hard bypass path** that cannot be disabled by any runtime failure:

```
Incoming CRITICAL event
        │
        ▼
   [P0] expires_at check
        │ (only this check applies)
        ▼
   [Delivery Dispatcher]
        │
        ├── attempt 1: push
        ├── attempt 2: email (if push fails)
        ├── attempt 3: SMS (if email fails)
        └── attempt 4: in-app (last resort)
        │
        ▼
   If ALL channels fail → DELIVERY_FAILED + P1 alert + DLQ
```

Rules that explicitly bypass for CRITICAL:
- ❌ Dedup guard (only `event_id` idempotency check applied)
- ❌ User fatigue caps
- ❌ Quiet hours
- ❌ Per-type cooldowns
- ❌ Incident suppression mode
- ❌ Composite score threshold
- ✅ `expires_at` check (P0) — only expiry can suppress a CRITICAL event

---

## Fallback Decision Transparency

When the engine makes a decision in fallback mode, the response and audit record always reflect this:

```json
{
  "outcome": "NOW",
  "reasons": ["DEFAULT_PASS", "AI_FALLBACK", "REDIS_FALLBACK"],
  "fallback_mode": true,
  "ai_used": false,
  "score": 0.50
}
```

`fallback_mode: true` in the API response allows callers to implement their own secondary logic if desired.

---

## Data Durability Guarantees

| Scenario | What Is Preserved |
|----------|------------------|
| AI service down | All decisions still made; accuracy slightly reduced |
| Redis down | All decisions still made; dedup + caps skipped; decisions logged |
| Postgres (Rules) down | Decisions made with cached rules; no decisions lost |
| Scheduler crashes | Deferred events preserved in Redis list; no loss on restart |
| Dispatcher fails after 3 retries | Event captured in DLQ; available for manual replay |
| Decision Engine pod crashes | In-flight decision may be lost; upstream retries on `event_id` are idempotent |

---

## Failure Runbooks

### AI Service Circuit Open
```
1. Check AI service health endpoint
2. Verify fallback rate in monitoring dashboard (should show NEUTRAL_SCORE decisions)
3. If AI service recovering → circuit half-opens automatically after 10s
4. If AI service fully down → fallback mode continues indefinitely; notifications still delivered
5. No manual intervention needed for < 30 min outage
```

### Redis Full Outage
```
1. IMMEDIATE: Check Redis Sentinel / Cluster status
2. Verify NPE is logging REDIS_FALLBACK reason in decisions
3. Expect: duplicate notifications may arrive (dedup disabled)
4. Expect: fatigue caps not enforced (users may receive more notifications than normal)
5. DO NOT manually suppress — this creates data loss risk
6. On Redis recovery: counters reset; dedup window restarts from now
7. Post-incident: audit for any unusual notification spikes during outage window
```

### Scheduler Crash
```
1. Kubernetes should auto-restart within 30s
2. Verify restart succeeded via pod health check
3. On restart: scheduler reads npe:defer_queue sorted set and resumes
4. Check for overdue events: query npe:defer_queue for scores < now()
5. If large backlog: scheduler processes in priority order (CRITICAL first)
```

### Mass Delivery Failure (channel outage)
```
1. Check channel-specific delivery failure rate in dashboard
2. Identify failing channel (push / email / SMS)
3. Dispatcher automatically retries on alternate channels for CRITICAL events
4. For non-critical: events in DLQ; hold for manual replay when channel recovers
5. Notify affected upstream services if SLA breach imminent
```
