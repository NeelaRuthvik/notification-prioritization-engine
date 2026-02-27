# Decision Logic

## Classification Outcomes

Every notification event is classified into exactly one outcome:

| Outcome | Symbol | Meaning |
|---------|--------|---------|
| **NOW** | 🟢 | Deliver immediately to configured channel(s) |
| **LATER** | 🟡 | Defer to a calculated or configured future time |
| **NEVER** | 🔴 | Suppress entirely — logged but never delivered |

---

## Rule Evaluation Pipeline (P0 → P8)

Rules are evaluated in strict priority order. **First matching rule wins.** Lower priority number = evaluated first.

```
┌────┬──────────────────────┬──────────────────────────────────────────────────┬──────────────┐
│ P# │ Stage                │ Logic                                            │ Outcome      │
├────┼──────────────────────┼──────────────────────────────────────────────────┼──────────────┤
│ P0 │ Hard Expiry Check    │ expires_at < now()                               │ NEVER        │
│ P1 │ Critical Override    │ category = CRITICAL or SECURITY                  │ NOW (bypass) │
│ P2 │ Dedup Guard          │ Exact hash match OR near-dup fingerprint match   │ NEVER        │
│ P3 │ User Opt-Out         │ User globally or per-channel unsubscribed        │ NEVER        │
│ P4 │ Fatigue Cap          │ User hit sliding-window count threshold          │ LATER/NEVER  │
│ P5 │ Quiet Hours          │ Current time within user quiet window            │ LATER        │
│ P6 │ Channel Rules        │ Channel-specific cap reached (e.g. SMS 3/day)   │ LATER/NOW*   │
│ P7 │ Priority Scoring     │ Composite score → threshold routing              │ NOW/LATER/NEVER │
│ P8 │ Default              │ No earlier rule matched                          │ NOW          │
└────┴──────────────────────┴──────────────────────────────────────────────────┴──────────────┘
* Channel demoted to alternate channel if available
```

---

## P1: Critical Override (Hard Bypass)

When `priority_hint = CRITICAL` or `event_type = SECURITY`:

- **Bypass**: Dedup (except exact `event_id` check), fatigue caps, quiet hours, all P3–P7 rules
- **Always outcome**: NOW
- **Only suppressible by**: expired `expires_at` (P0) or account-level ban
- **Rationale**: Security and critical events must never be silently lost due to noise-reduction logic

---

## P4: Fatigue Cap Detail

| Window | Default Cap | Action on Breach |
|--------|-------------|-----------------|
| 5 minutes | 3 notifications | LATER — defer 15 minutes |
| 1 hour | 10 notifications | LATER — defer to next hour boundary |
| 24 hours | 30 total | NEVER for PROMO/LOW; LATER for MEDIUM+ |

Counters are Redis sliding-window sorted sets (by timestamp). Atomic `ZADD` + `ZCOUNT` keeps them accurate under concurrent load.

---

## P7: Composite Priority Score

When no earlier rule has resolved the outcome, a composite score `S` is computed:

```
S = (w1 × priority_hint_score)
  + (w2 × ai_intent_score)
  + (w3 × event_type_base_weight)
  + (w4 × recency_factor)
```

### Default Weights

| Weight | Value | Factor |
|--------|-------|--------|
| w1 | 0.35 | priority_hint_score |
| w2 | 0.25 | ai_intent_score |
| w3 | 0.30 | event_type_base_weight |
| w4 | 0.10 | recency_factor |

### Score Mappings

**priority_hint_score:**

| priority_hint | Score |
|---------------|-------|
| CRITICAL | 1.0 |
| HIGH | 0.8 |
| MEDIUM | 0.5 |
| LOW | 0.2 |
| (absent) | 0.3 |

**ai_intent_score:** 0.0–1.0 from Enrichment Service. Falls back to `0.5` if AI unavailable.

**event_type_base_weight:** Configurable per `event_type` in rules store. Example defaults:

| event_type | Base Weight |
|------------|-------------|
| SECURITY | 1.0 |
| MESSAGE | 0.75 |
| REMINDER | 0.6 |
| SYSTEM | 0.55 |
| UPDATE | 0.4 |
| PROMO | 0.15 |

**recency_factor:** Decays as the user receives more notifications recently:
```
recency_factor = 1.0 − (min(recent_count_1h, 10) / 10)
```
At 0 recent notifications → 1.0. At 10+ recent notifications → 0.0.

### Threshold Routing

| Score Range | Outcome |
|-------------|---------|
| S ≥ 0.65 | NOW |
| 0.30 ≤ S < 0.65 | LATER |
| S < 0.30 | NEVER |

Thresholds are configurable per `event_type` in the rules store.

---

## Conflict Resolution

Some notifications are simultaneously urgent AND potentially noisy (e.g., high-frequency alerts during a system incident). Resolution rules:

| Scenario | Resolution |
|----------|-----------|
| CRITICAL + fatigue cap exceeded | NOW — fatigue cap does not apply to CRITICAL |
| HIGH + fatigue cap exceeded | LATER (not NEVER); max 1 re-queue attempt |
| HIGH deferred twice already | Promoted to NOW with reason `FORCED_DELIVERY` |
| PROMO during active incident window | NEVER with reason `INCIDENT_SUPPRESSION` |
| LOW during incident window | NEVER with reason `INCIDENT_SUPPRESSION` |
| HIGH during quiet hours | LATER (reschedule to quiet_end + jitter) |
| CRITICAL during quiet hours | NOW — quiet hours do not apply |

### Forced Delivery (Anti-Starvation)

To prevent high-priority events from being permanently deferred:

```
if defer_count >= 2 AND priority >= HIGH:
    outcome = NOW
    reasons.append("FORCED_DELIVERY")
```

---

## Defer Strategy

When outcome is LATER, `defer_until` is calculated based on the reason:

| Reason | Defer Until |
|--------|-------------|
| QUIET_HOURS | next `quiet_end` boundary + random jitter (0–5 min) |
| FATIGUE_CAP_5M | now + 15 minutes |
| FATIGUE_CAP_1H | next hour boundary |
| FATIGUE_CAP_24H | next day 8am (user local time) |
| COOLDOWN_ACTIVE | cooldown key TTL expiry |
| DIGEST_ELIGIBLE | next digest delivery window |

**Jitter** on quiet hours wake-up is critical to prevent thundering herd when thousands of users' quiet windows end simultaneously.

---

## Incident Suppression Mode

When a `SYSTEM` event with `metadata.incident = true` is received:

1. Engine enters **incident suppression mode** for affected `user_id` (or globally if `metadata.scope = global`)
2. All `PROMO` and `LOW` events → NEVER with reason `INCIDENT_SUPPRESSION`
3. Mode expires after configurable TTL (default: 2 hours) OR on receipt of recovery system event
4. `HIGH`, `MEDIUM`, `CRITICAL` events continue routing normally

---

## Reason Code Quick Reference

| Code | Outcome | Triggered By |
|------|---------|-------------|
| `EXPIRED` | NEVER | P0: expires_at passed |
| `CRITICAL_OVERRIDE` | NOW | P1: CRITICAL/SECURITY category |
| `DEDUP_EXACT` | NEVER | P2: exact hash match |
| `DEDUP_NEAR` | NEVER | P2: SimHash Hamming ≤ 3 |
| `USER_OPT_OUT` | NEVER | P3: user unsubscribed |
| `FATIGUE_CAP_5M` | LATER | P4: 5-minute cap |
| `FATIGUE_CAP_1H` | LATER/NEVER | P4: hourly cap |
| `FATIGUE_CAP_24H` | LATER/NEVER | P4: daily cap |
| `QUIET_HOURS` | LATER | P5: quiet window active |
| `COOLDOWN_ACTIVE` | LATER | P5/P6: per-type cooldown |
| `DIGEST_ELIGIBLE` | LATER | P6: batching eligible |
| `INCIDENT_SUPPRESSION` | NEVER | P6: active incident |
| `SCORE_BELOW_THRESHOLD` | NEVER | P7: score < 0.30 |
| `SCORE_DEFER` | LATER | P7: score 0.30–0.65 |
| `FORCED_DELIVERY` | NOW | Anti-starvation: 2+ defers |
| `DEFAULT_PASS` | NOW | P8: no rule matched |
