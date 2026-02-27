# 🔔 Notification Prioritization Engine (NPE)

> **Cyepro Solutions — Round 1 AI-Native Solution Crafting Test**  
> Submitted: February 2026

---

## Overview

The **Notification Prioritization Engine** is an intelligent routing system that makes a real-time decision for every incoming notification event:

| Decision | Meaning |
|----------|---------|
| 🟢 **NOW** | Deliver immediately to configured channel(s) |
| 🟡 **LATER** | Defer to a calculated future time, then re-deliver |
| 🔴 **NEVER** | Suppress entirely — logged but never delivered |

The engine handles thousands of notification events per minute, prevents alert fatigue, eliminates duplicates, respects user preferences, and remains fully auditable — all while degrading safely when AI or dependent services are unavailable.

---

## Project Structure

```
/docs
   architecture.md        → System components, data flow, topology
   decision-logic.md      → NOW / LATER / NEVER classification strategy
   data-model.md          → Entities, schemas, Redis key patterns
   api-design.md          → 5 REST endpoint contracts
   monitoring.md          → Metrics, dashboards, alerting, audit
   fallback-strategy.md   → Failure modes and safe degradation
/diagrams
   architecture.png       → High-level system component diagram
   flowchart.png          → Decision engine evaluation flowchart
README.md                 → This file
```

---

## Core Design Principles

1. **Speed** — Sub-50ms median decision latency
2. **Explainability** — Every decision logged with an ordered `reasons[]` array
3. **Safety** — AI enrichment is optional; deterministic rules always run
4. **Configurability** — Business rules stored as data (no deployment for rule changes)
5. **Reliability** — Critical notifications can never be silently lost

---

## Quick Reference: Decision Flow

```
Incoming Event
     │
     ▼
[API Gateway]  ──── Validate & normalize
     │
     ▼
[Dedup Service]  ── Bloom filter → Redis hash check
     │                  └── DUPLICATE? → NEVER (DEDUP_EXACT / DEDUP_NEAR)
     ▼
[Enrichment]  ───── User prefs + AI intent score (30ms timeout)
     │
     ▼
[Decision Engine]  ─ P0→P8 rule evaluation
     │
     ├── NOW   → Delivery Dispatcher → push/email/SMS/in-app
     ├── LATER → Scheduler Service  → re-enter at deferUntil
     └── NEVER → Audit Log only
```

---

## Key Capabilities

- **Exact + Near-Duplicate Suppression** via SHA-256 canonical hash + SimHash fingerprinting
- **Multi-layer Fatigue Prevention**: sliding window caps (5m / 1h / 24h), per-type cooldowns, digest batching, quiet hours
- **Critical Override Path**: CRITICAL/SECURITY events bypass all suppression — delivered NOW always
- **Human-Configurable Rules**: JSONB rule store in Postgres; changes take effect within 30 seconds
- **Full Audit Trail**: append-only `DecisionRecord` table with reason codes, scores, rule version snapshot
- **Circuit Breakers** on all external dependencies with defined fallback behavior

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| API Gateway | REST / JWT auth |
| Dedup | Redis (Bloom filter + Hash) |
| Enrichment | AI inference service (optional) |
| Decision Engine | Stateless rule evaluator |
| Rules Store | PostgreSQL (JSONB) |
| Scheduler | Redis Sorted Set + background worker |
| Audit Log | PostgreSQL (append-only) |
| Monitoring | Metrics dashboard + alerting |

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | Components, data flow, deployment topology |
| [Decision Logic](docs/decision-logic.md) | Classification strategy, scoring, conflict resolution |
| [Data Model](docs/data-model.md) | Entities, schemas, Redis key patterns |
| [API Design](docs/api-design.md) | 5 endpoint contracts with request/response shapes |
| [Monitoring](docs/monitoring.md) | KPIs, dashboards, audit, reason code reference |
| [Fallback Strategy](docs/fallback-strategy.md) | Failure modes, circuit breakers, safe degradation |

---

*For questions or review, refer to the individual docs above or the system diagrams in `/diagrams`.*
