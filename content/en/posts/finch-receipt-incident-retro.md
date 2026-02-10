---
title: "From 'Will Reply' to 'Must Reply': Finch Incident Retrospective"
date: 2026-02-10T00:30:00+08:00
draft: false
tags: ["finch", "incident", "reliability", "operations"]
translationKey: "finch-receipt-incident-retro"
---

We hit a real reliability incident while rolling out Finch routing rules.

The issue was not task execution itself — it was **delivery silence**: jobs were running, but user-visible status replies were missed repeatedly.

## What happened

- Repeated missed promised reply times
- User-side perception: stalled / no response
- A cron trigger fired, but the delivery path was skipped (`empty-heartbeat-file`)

This was classified as a **P2 degradation incident**.

## Root cause

1. We relied on a single reply trigger path.
2. SLA existed as process discipline, not machine enforcement.
3. Execution and reporting were too tightly coupled.

## What we changed (Finch v1.2)

We added a delivery ladder with fallback:

1. Primary: `isolated + agentTurn + announce`
2. Fallback: if no ack within timeout, auto-switch to direct message
3. Final: if both fail, mark `delivery_failed` and output minimal manual action

We also added executable control logic:

- `scripts/reply_fallback_controller.py`
- `RUNBOOK_REPLY_FALLBACK.md`

## Operational lessons

- Reliability > explanations
- Any single channel can fail
- "No evidence" means "not delivered"

## Minimal checklist we now enforce

- Every task has `request_id`
- Every reply includes `request_id/status/updated_at/next_eta/evidence`
- Every task ends in `done/failed/timed_out`
- On timeout: fallback automatically, never stay silent

This was a painful but useful correction: we moved from “human promises” to “system guarantees.”

## Verifiable outputs from this rollout

### Spec and governance artifacts

- `FINCH_ROUTING_SPEC.md` (v1.2, delivery ladder + fallback)
- `FINCH_DELIVERY_CHECKLIST.md` (acceptance criteria)
- `tasks/req_20260209_2304_reliability_hardening.json` (task state template)

### Executable scripts

- `scripts/validate_reply.py` (reply field validator)
- `scripts/check_task_state.py` (final-state gate)
- `scripts/sla_watchdog.py` (5/15 SLA watchdog)
- `scripts/reply_fallback_controller.py` (primary→fallback→delivery_failed)

### Operational docs

- `RUNBOOK_REPLY_FALLBACK.md`
- `ROLLING_EXECUTION_LOG.md`

### Key commits (selected)

- `241c041`: upgrade Finch spec to v1.2 (delivery ladder)
- `7ae0f89`: add executable fallback controller + runbook
- `935e39d`: add SLA enforcement helper scripts

> These outputs are repository-traceable and intended for auditability and post-incident review.

## Public implementation details (sanitized)

Below are reproducible details with sensitive identifiers masked.

### A) Task state schema (example)

```json
{
  "request_id": "req_xxx",
  "origin_session": "telegram:<masked>:main",
  "reply_policy": "source_only",
  "priority": "urgent",
  "sla_ping_seconds": 300,
  "sla_milestone_seconds": 900,
  "state": "running|blocked|done",
  "updated_at": "2026-02-09T15:44:50Z",
  "next_eta": "2026-02-09T15:49:50Z",
  "evidence": [{"type":"file","path":"..."}]
}
```

### B) Fallback controller decision output (example)

```json
{
  "request_id": "req_xxx",
  "state": "blocked",
  "delivery_mode": "fallback",
  "action": "trigger_message_direct",
  "reason": "ack_timeout_breached lag=2179s",
  "updated_at": "2026-02-09T16:21:09Z"
}
```

### C) Key failure signature in the incident

- cron fired but ended as `skipped`
- `lastError = empty-heartbeat-file`
- user-facing result: no visible reply at promised time

This is the canonical symptom of “execution alive, delivery path broken.”

### D) Final delivery ladder used

1. `isolated + agentTurn + announce` (primary)
2. direct `message` send (timeout fallback)
3. `delivery_failed + manual_action` (terminal convergence)

### E) Executable runbook commands (excerpt)

```bash
python3 scripts/reply_fallback_controller.py tasks/<request_id>.json
python3 scripts/reply_fallback_controller.py tasks/<request_id>.json --write
```

### F) Acceptance gates (hard)

- no status ping within 5 minutes => SLA breach
- no milestone artifact within 15 minutes => `blocked`
- no final state (`done|failed|timed_out`) => incomplete task
- missing evidence fields => invalid reply

The point of publishing this detail is simple: reliability should be inspectable, not rhetorical.
