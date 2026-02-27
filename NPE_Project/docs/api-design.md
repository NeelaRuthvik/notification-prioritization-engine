# API Design

## Overview

The NPE exposes 5 REST endpoints. All endpoints:
- Accept and return `application/json`
- Authenticate via service-to-service **JWT** in `Authorization: Bearer <token>` header
- Return standard error shape on failure: `{ "error": { "code": "...", "message": "..." } }`

**Base URL:** `https://npe.internal/v1`

---

## Endpoint Index

| # | Method | Path | Purpose |
|---|--------|------|---------|
| 1 | POST | `/notifications/submit` | Submit a notification event; receive decision synchronously |
| 2 | GET | `/notifications/decision/{event_id}` | Retrieve stored decision record (audit/support) |
| 3 | GET | `/users/{user_id}/notification-state` | Get current user notification state |
| 4 | PUT | `/rules/{rule_id}` | Create or update a routing rule |
| 5 | POST | `/notifications/batch-status` | Poll delivery status for up to 100 event IDs |

---

## 1. POST `/notifications/submit`

Submit a notification event to the engine. Returns the decision synchronously. Idempotent on `event_id`.

### Request Body

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "user_id": "usr_abc123",
  "event_type": "MESSAGE",
  "title": "You have a new message from Alice",
  "message": "Hey, can we sync tomorrow?",
  "source": "messaging-service",
  "priority_hint": "HIGH",
  "channel": ["push", "in_app"],
  "timestamp": "2026-02-25T14:32:00Z",
  "expires_at": "2026-02-25T20:00:00Z",
  "dedupe_key": "msg-thread-9821-usr-abc123",
  "metadata": {
    "thread_id": "thread_9821",
    "sender_id": "usr_xyz456"
  }
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `event_id` | ✅ | UUID; caller-supplied for idempotency |
| `user_id` | ✅ | Target user |
| `event_type` | ✅ | Enum: MESSAGE, REMINDER, ALERT, PROMO, SYSTEM, UPDATE, SECURITY |
| `title` | ✅ | Max 120 chars |
| `source` | ✅ | Originating service name |
| `channel` | ✅ | Non-empty array: push, email, sms, in_app |
| `timestamp` | ✅ | ISO8601 UTC |
| `message` | ❌ | Max 1000 chars |
| `priority_hint` | ❌ | CRITICAL / HIGH / MEDIUM / LOW |
| `expires_at` | ❌ | If past on receipt → immediate NEVER |
| `dedupe_key` | ❌ | Caller hint; absent = canonical hash used |
| `metadata` | ❌ | Arbitrary JSON ≤ 4KB |

### Responses

**200 OK — Decision made**
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "decision_id": "d1e2f3a4-...",
  "outcome": "NOW",
  "reasons": ["DEFAULT_PASS"],
  "score": 0.74,
  "defer_until": null,
  "ai_used": true,
  "decided_at": "2026-02-25T14:32:00.042Z"
}
```

**200 OK — Deferred**
```json
{
  "event_id": "...",
  "decision_id": "...",
  "outcome": "LATER",
  "reasons": ["QUIET_HOURS"],
  "score": 0.61,
  "defer_until": "2026-02-26T08:03:17Z",
  "ai_used": true,
  "decided_at": "..."
}
```

**200 OK — Suppressed**
```json
{
  "event_id": "...",
  "decision_id": "...",
  "outcome": "NEVER",
  "reasons": ["DEDUP_EXACT"],
  "score": null,
  "defer_until": null,
  "ai_used": false,
  "decided_at": "..."
}
```

**409 Conflict — Already processed (idempotent replay)**
```json
{
  "event_id": "...",
  "decision_id": "...",
  "outcome": "NOW",
  "reasons": ["IDEMPOTENT_REPLAY"],
  "message": "Event already processed; returning cached decision"
}
```

**422 Unprocessable Entity**
```json
{
  "error": {
    "code": "VALIDATION_FAILURE",
    "message": "Field 'channel' must be a non-empty array",
    "fields": ["channel"]
  }
}
```

**503 Service Degraded** *(fallback mode active)*
```json
{
  "event_id": "...",
  "decision_id": "...",
  "outcome": "NOW",
  "reasons": ["DEFAULT_PASS", "AI_FALLBACK", "REDIS_FALLBACK"],
  "score": 0.50,
  "fallback_mode": true,
  "decided_at": "..."
}
```

---

## 2. GET `/notifications/decision/{event_id}`

Retrieve the stored decision record for a previously submitted event. Used for audit, support, and debugging.

### Path Parameters

| Param | Type | Required |
|-------|------|----------|
| `event_id` | UUID | ✅ |

### Response 200

```json
{
  "decision_id": "d1e2f3a4-...",
  "event_id": "550e8400-...",
  "user_id": "usr_abc123",
  "outcome": "NOW",
  "reasons": ["DEFAULT_PASS"],
  "score": 0.74,
  "ai_used": true,
  "ai_score": 0.82,
  "rule_version": "2.1.4",
  "defer_until": null,
  "defer_count": 0,
  "decided_at": "2026-02-25T14:32:00.042Z",
  "delivered_at": "2026-02-25T14:32:00.119Z",
  "delivery_channel": "push",
  "delivery_status": "DELIVERED"
}
```

**404 Not Found**
```json
{ "error": { "code": "NOT_FOUND", "message": "No decision found for event_id" } }
```

---

## 3. GET `/users/{user_id}/notification-state`

Retrieve the current notification state for a user. Useful for support tooling, user-facing preference explanations, and debugging suppression behavior.

### Response 200

```json
{
  "user_id": "usr_abc123",
  "window_counts": {
    "last_5m": 1,
    "last_1h": 7,
    "last_24h": 22
  },
  "fatigue_caps": {
    "5m_cap": 3,
    "1h_cap": 10,
    "24h_cap": 30
  },
  "active_suppressions": [
    {
      "event_type": "PROMO",
      "reason": "COOLDOWN_ACTIVE",
      "expires_at": "2026-02-25T15:30:00Z"
    }
  ],
  "incident_mode_active": false,
  "quiet_hours": {
    "enabled": true,
    "start": "22:00",
    "end": "08:00",
    "timezone": "Asia/Kolkata"
  },
  "opted_out_channels": ["sms"],
  "opted_out_event_types": ["PROMO"],
  "pending_deferred_count": 2
}
```

---

## 4. PUT `/rules/{rule_id}`

Create or update a notification routing rule. Changes take effect within 30 seconds (in-memory cache TTL refresh). **No deployment required.**

### Request Body

```json
{
  "name": "Suppress promos during quiet hours",
  "description": "Defer all PROMO events that arrive outside business hours to next 8am",
  "priority": 45,
  "conditions": {
    "op": "AND",
    "rules": [
      { "field": "event_type", "operator": "eq", "value": "PROMO" },
      { "field": "priority_hint", "operator": "in", "value": ["LOW", null] }
    ]
  },
  "action": {
    "outcome": "LATER",
    "defer_strategy": "next_morning",
    "channel_override": null,
    "reason_code": "PROMO_DEFERRED_QUIET"
  },
  "enabled": true
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `name` | ✅ | Human-readable label |
| `priority` | ✅ | Unique integer; lower = evaluated first |
| `conditions` | ✅ | JSONB predicate tree |
| `action` | ✅ | JSONB outcome block |
| `enabled` | ✅ | false = disable without deleting |
| `description` | ❌ | Optional explanation |

### Response 200

```json
{
  "rule_id": "rule_abc123",
  "name": "Suppress promos during quiet hours",
  "priority": 45,
  "version": "1.0.0",
  "enabled": true,
  "created_at": "2026-02-25T14:00:00Z",
  "updated_at": "2026-02-25T14:00:00Z",
  "message": "Rule saved. Will take effect within 30 seconds."
}
```

**400 Bad Request**
```json
{
  "error": {
    "code": "INVALID_RULE",
    "message": "Condition field 'event_type' uses unsupported operator 'regex'",
    "fields": ["conditions.rules[0].operator"]
  }
}
```

**409 Conflict** *(priority already taken)*
```json
{
  "error": {
    "code": "PRIORITY_CONFLICT",
    "message": "Priority 45 is already assigned to rule 'rule_xyz'. Choose a unique priority value."
  }
}
```

---

## 5. POST `/notifications/batch-status`

Poll delivery status and decision outcomes for up to 100 event IDs in a single call. Designed for dashboards, reporting, and bulk support lookups.

### Request Body

```json
{
  "event_ids": [
    "550e8400-e29b-41d4-a716-446655440000",
    "661f9511-f30c-52e5-b827-557766551111"
  ]
}
```

| Field | Required | Constraints |
|-------|----------|-------------|
| `event_ids` | ✅ | Array of UUIDs; max 100 items |

### Response 200

```json
{
  "results": [
    {
      "event_id": "550e8400-...",
      "outcome": "NOW",
      "delivery_status": "DELIVERED",
      "delivered_at": "2026-02-25T14:32:00.119Z",
      "reasons": ["DEFAULT_PASS"]
    },
    {
      "event_id": "661f9511-...",
      "outcome": "NEVER",
      "delivery_status": "SUPPRESSED",
      "delivered_at": null,
      "reasons": ["DEDUP_EXACT"]
    }
  ],
  "not_found": [],
  "total": 2
}
```

---

## Error Code Reference

| HTTP Status | Code | Meaning |
|-------------|------|---------|
| 400 | `VALIDATION_FAILURE` | Request schema invalid |
| 400 | `INVALID_RULE` | Rule conditions/action schema invalid |
| 401 | `UNAUTHORIZED` | Missing or invalid JWT |
| 403 | `FORBIDDEN` | Valid JWT but insufficient scope |
| 404 | `NOT_FOUND` | Resource not found |
| 409 | `DUPLICATE_EVENT` | Idempotent replay of already-processed event |
| 409 | `PRIORITY_CONFLICT` | Rule priority number already in use |
| 422 | `INVALID_ENUM` | Unrecognized enum value |
| 429 | `RATE_LIMITED` | Ingestion rate limit exceeded |
| 503 | `DEGRADED` | Fallback mode active; decision made with reduced accuracy |
