# Monitoring

## Overview

The NPE monitoring strategy has three layers:
1. **Real-time metrics** — latency, throughput, outcome ratios
2. **Operational dashboards** — live system health and rule impact views
3. **Audit & explainability** — per-decision traceability and user-facing reason logs

---

## Key Performance Indicators (KPIs)

| Metric | Type | Alert: Warning | Alert: Critical | What It Signals |
|--------|------|---------------|-----------------|-----------------|
| `decision_latency_p50` | Histogram | > 50ms | > 100ms | Median decision speed |
| `decision_latency_p99` | Histogram | > 150ms | > 300ms | Tail latency / SLA risk |
| `throughput_events_per_sec` | Counter | Drop > 20% in 1min | Drop > 50% | Ingestion health |
| `outcome_ratio_now` | Gauge (%) | < 40% | < 25% | Over-suppression risk |
| `outcome_ratio_later` | Gauge (%) | > 40% | > 60% | Accumulating defer backlog |
| `outcome_ratio_never` | Gauge (%) | > 55% | > 70% | Aggressive suppression / rule misconfiguration |
| `dedup_hit_rate` | Counter | > 20%/min | > 40%/min | Upstream retry storm or key recycling |
| `near_dedup_hit_rate` | Counter | > 10%/min | — | Excessive semantic duplicates |
| `ai_fallback_rate` | Gauge (%) | > 5% | > 20% | AI enrichment service degradation |
| `fatigue_cap_breach_count` | Counter | Spike > 2σ | — | Noisy source or misconfigured cap |
| `delivery_failure_rate` | Gauge (%) | > 1% | > 5% | Channel delivery health |
| `critical_notifications_suppressed` | Counter | **Any > 0** | **Any > 0** | Must always be zero (except expired) |
| `scheduler_queue_depth` | Gauge | > 5,000 | > 20,000 | Deferred notification backlog growing |
| `scheduler_overdue_count` | Counter | > 100/min | > 500/min | Scheduler falling behind |
| `bloom_filter_fill_ratio` | Gauge (%) | > 60% | > 75% | Bloom filter needs rotation |
| `rules_cache_age_seconds` | Gauge | > 60s | > 120s | Rules cache not refreshing |

---

## Dashboards

### 1. Decision Outcomes Dashboard (Real-Time)
**Refresh:** 10 seconds  
**Audience:** Product, Engineering, Operations

- Stacked time-series: NOW / LATER / NEVER ratio over last 24h
- Drill-down by `event_type`, `source`, `priority_hint`
- Top 10 suppression reasons (by volume, last 1h)
- Top 10 noisiest user_ids (by notification volume, last 1h)
- Active incident suppression windows

### 2. System Health Dashboard
**Refresh:** 5 seconds  
**Audience:** On-call Engineering

- Decision latency P50 / P99 / P99.9 (last 5min, 1h, 24h)
- Throughput (events/sec, rolling 1min)
- Circuit breaker states: AI service, Redis, Postgres
- AI fallback rate (rolling 5min)
- Delivery failure rate per channel
- Bloom filter fill ratio
- Scheduler queue depth and overdue count

### 3. Rule Impact Dashboard
**Audience:** Product / Operations (post-rule-change review)

- For any given time window: outcome distribution before vs after
- Per-rule match count and outcome breakdown
- Rule version history and change log
- "Dry run" mode: simulate a rule change against last 1h of events without applying it

### 4. User Notification Health View
**Audience:** Support tooling / CX team

- Per `user_id`: 24h notification timeline (received / deferred / suppressed)
- Active suppressions and their expiry
- Opted-out channels and event types
- Pending deferred count
- Quiet hours configuration
- Reason code history for last 50 decisions

---

## Alerting

All alerts route to PagerDuty. Severity levels:

| Severity | Response SLA | Examples |
|----------|-------------|---------|
| **P1 (Critical)** | Immediate page | `critical_notifications_suppressed > 0`, `decision_latency_p99 > 500ms`, `throughput drop > 50%` |
| **P2 (High)** | 15-minute response | `delivery_failure_rate > 5%`, `scheduler_queue_depth > 20k`, `outcome_ratio_never > 70%` |
| **P3 (Medium)** | Next business hour | `ai_fallback_rate > 20%`, `bloom_filter_fill_ratio > 75%`, `rules_cache_age > 120s` |
| **P4 (Low)** | Weekly review | Sustained `dedup_hit_rate > 20%`, `fatigue_cap_breach` spikes |

---

## Audit & Explainability

### DecisionRecord Audit Log

Every decision is written to an **append-only** Postgres table with:
- Full `reasons[]` array (ordered, human-readable codes)
- `rule_version` snapshot — enables "what would have happened under old rules?" replay
- `ai_used` and `ai_score` — transparency on AI influence
- `decided_at` and `delivered_at` — end-to-end latency tracking

### Audit Log Access Patterns

```sql
-- All decisions for a user in the last 24h
SELECT * FROM decision_records
WHERE user_id = 'usr_abc123'
  AND decided_at > NOW() - INTERVAL '24 hours'
ORDER BY decided_at DESC;

-- All suppressed CRITICAL events (should always return 0 except expired)
SELECT * FROM decision_records
WHERE outcome = 'NEVER'
  AND 'CRITICAL_OVERRIDE' != ANY(reasons)
  AND event_type = 'SECURITY';

-- Suppression reason breakdown for last 1h
SELECT unnest(reasons) AS reason, COUNT(*) AS count
FROM decision_records
WHERE decided_at > NOW() - INTERVAL '1 hour'
  AND outcome = 'NEVER'
GROUP BY reason
ORDER BY count DESC;
```

### User-Facing Explainability

The `reasons[]` array is surfaced in:
- **Notification preferences UI**: "Why didn't I receive this?" tooltip
- **Support tooling**: CX agents can look up any notification by event_id
- **API response**: included in every `POST /notifications/submit` response

---

## Reason Code Reference (Complete)

| Code | Outcome | Stage | Description |
|------|---------|-------|-------------|
| `EXPIRED` | NEVER | P0 | `expires_at` was in the past at decision time |
| `CRITICAL_OVERRIDE` | NOW | P1 | CRITICAL or SECURITY category — all suppression bypassed |
| `DEDUP_EXACT` | NEVER | P2 | Exact canonical hash match within dedup window |
| `DEDUP_NEAR` | NEVER | P2 | SimHash fingerprint Hamming distance ≤ threshold |
| `USER_OPT_OUT` | NEVER | P3 | User opted out globally or for this channel/type |
| `FATIGUE_CAP_5M` | LATER | P4 | 5-minute sliding window cap reached |
| `FATIGUE_CAP_1H` | LATER / NEVER | P4 | Hourly sliding window cap reached |
| `FATIGUE_CAP_24H` | LATER / NEVER | P4 | Daily sliding window cap reached |
| `QUIET_HOURS` | LATER | P5 | Arrival during user-configured quiet window |
| `COOLDOWN_ACTIVE` | LATER | P5 | Per-type cooldown period not yet expired |
| `DIGEST_ELIGIBLE` | LATER | P6 | Queued for digest batching delivery |
| `CHANNEL_CAP` | LATER | P6 | Channel-specific daily cap reached; demoted or deferred |
| `INCIDENT_SUPPRESSION` | NEVER | P6 | Active incident window suppresses PROMO/LOW events |
| `SCORE_BELOW_THRESHOLD` | NEVER | P7 | Composite score < 0.30 |
| `SCORE_DEFER` | LATER | P7 | Composite score in range 0.30–0.65 |
| `FORCED_DELIVERY` | NOW | Anti-starvation | Event previously deferred 2+ times; forced through |
| `AI_FALLBACK` | *(flag)* | Enrichment | AI service unavailable; neutral score (0.5) used |
| `REDIS_FALLBACK` | *(flag)* | Dedup/counters | Redis unavailable; dedup and caps skipped |
| `DEFAULT_PASS` | NOW | P8 | No suppression rule matched; delivered by default |
| `IDEMPOTENT_REPLAY` | *(flag)* | Gateway | Already-processed event_id; returning cached decision |
| `DELIVERY_FAILED` | *(flag)* | Dispatcher | Decision was NOW but channel delivery failed after 3 retries |

---

## Operational Runbooks

### High `outcome_ratio_never` Alert
1. Check "Top 10 suppression reasons" panel
2. If `DEDUP_EXACT` dominating → check for upstream retry storm
3. If `FATIGUE_CAP` dominating → review cap thresholds or noisy source
4. If `USER_OPT_OUT` dominating → expected; check opt-out volume trend

### `critical_notifications_suppressed > 0` Alert (P1)
1. **Immediately** query audit log for suppressed CRITICAL events
2. Check if suppression reason is `EXPIRED` (expected) or anything else (incident)
3. If unexpected → escalate; check Decision Engine rule config for P1 bypass
4. Replay affected events if `expires_at` not passed

### Scheduler Overdue Alert
1. Check scheduler pod health
2. Check Redis connection from scheduler
3. If Redis degraded → deferred events are in Redis list; will replay on recovery
4. If scheduler pod crashed → Kubernetes restarts automatically; verify restart succeeded
