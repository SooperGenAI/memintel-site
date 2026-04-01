---
id: quickstart
title: Quickstart
sidebar_label: Quickstart (5 min)
---

# Quickstart — 5 Minutes to Your First Decision

Build your first deterministic monitoring task — from registering a primitive to executing a full decision pipeline.

**Goal:** Define what to monitor → compile intent into a condition → execute deterministically → get a reproducible decision.

---

## What You're Building

```
Primitive (signal) → Task (intent) → Condition (compiled) → Execute → Decision
```

**Example:** You want to alert when an account's active user rate drops below a threshold. You describe that intent in plain English. Memintel compiles it into a deterministic condition. Every evaluation is reproducible and auditable.

---

## Before You Start

You need a running Memintel server. See [Self-Hosting](/docs/intro/self-hosting) or [Local Setup](/docs/intro/local-setup) to get one running.

You'll need:
- Your server URL (e.g. `http://localhost:8000`)
- Your `MEMINTEL_ELEVATED_KEY` (for registry operations)

---

## Step 1 — Define Application Context (Recommended)

Give the LLM compiler domain knowledge before creating tasks. This produces more accurate conditions from the first request.

```bash
curl -X POST http://localhost:8000/context/context \
  -H "Content-Type: application/json" \
  -d '{
    "domain": {
      "description": "B2B SaaS platform monitoring customer health and churn risk",
      "entities": [
        { "name": "account", "description": "company-level subscription — the billing unit" },
        { "name": "user", "description": "individual platform user within an account" }
      ],
      "decisions": ["churn_risk", "expansion_opportunity"]
    },
    "calibration_bias": {
      "false_negative_cost": "high",
      "false_positive_cost": "medium"
    }
  }'
```

Response: `{ "context_id": "...", "version": "v1", "is_active": true }`

:::tip
Skipping this step is valid — the system works without context. But compiled conditions will be more generic and may need more calibration cycles.
:::

---

## Step 2 — Register a Primitive

Register the signal you want to monitor. This tells the compiler what data is available and what type it is.

```bash
curl -X POST http://localhost:8000/registry/definitions \
  -H "X-Elevated-Key: your-elevated-key" \
  -H "Content-Type: application/json" \
  -d '{
    "primitive_id": "account.active_user_rate_30d",
    "type": "float",
    "namespace": "org",
    "missing_data_policy": "null"
  }'
```

Response: `{ "primitive_id": "account.active_user_rate_30d", "type": "float", ... }`

This primitive represents the ratio of active users to total licensed seats over the last 30 days (0–1). Your data engineer connects it to the actual data source in `memintel_config.yaml`.

---

## Step 3 — Register an Action

Define what happens when a condition fires.

```bash
curl -X POST http://localhost:8000/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -H "Content-Type: application/json" \
  -d '{
    "action_id": "slack_cs_alert",
    "version": "v1",
    "config": {
      "type": "notification",
      "channel": "slack-customer-success"
    },
    "trigger": {
      "fire_on": "true",
      "condition_id": "cond_churn_risk",
      "condition_version": "v1"
    },
    "namespace": "org"
  }'
```

---

## Step 4 — Create a Task

Describe your monitoring intent in plain English. The LLM compiler — constrained by your guardrails and context — compiles this into a deterministic condition.

```bash
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "intent": "Alert me when churn risk is high — active user rate drops below 35%",
    "entity_scope": "account",
    "delivery": {
      "type": "webhook",
      "endpoint": "https://myapp.com/hooks/alert"
    }
  }'
```

Response includes the compiled condition:

```json
{
  "task_id": "task_abc123",
  "condition_id": "cond_churn_risk",
  "condition_version": "v1",
  "context_version": "v1",
  "context_warning": null
}
```

The `context_warning: null` confirms the task was compiled with active domain context. The condition is now immutable — its strategy, parameters, and logic are fixed.

:::note How compilation works
You expressed intent in natural language. The LLM resolved it within your guardrails constraints into a deterministic threshold condition: `account.active_user_rate_30d < 0.35`. No LLM is involved in any subsequent evaluation — only the compiled condition.
:::

---

## Step 5 — Execute the Full Pipeline

Run the complete ψ → φ → α pipeline for an entity. Provide a timestamp to make the result deterministic and cacheable.

```bash
curl -X POST http://localhost:8000/evaluate/full \
  -H "Content-Type: application/json" \
  -d '{
    "concept_id": "<concept_id from task response>",
    "concept_version": "<concept_version from task response>",
    "condition_id": "cond_churn_risk",
    "condition_version": "v1",
    "entity": "account_xyz789",
    "timestamp": "2025-11-14T09:00:00Z"
  }'
```

Response:

```json
{
  "result": {
    "value": 0.29,
    "deterministic": true
  },
  "decision": {
    "value": true,
    "strategy": "threshold",
    "threshold_applied": 0.35,
    "direction": "below"
  },
  "actions_triggered": [
    { "action_id": "slack_cs_alert", "status": "triggered" }
  ]
}
```

`result.deterministic: true` confirms this evaluation is cached and reproducible — run the same call again with the same parameters and you get the identical result.

---

## Step 6 — Verify Determinism

Run the same call three times. All three must return identical `result.value` and `decision.value`:

```bash
for i in 1 2 3; do
  curl -s -X POST http://localhost:8000/evaluate/full \
    -H "Content-Type: application/json" \
    -d '{ ...same payload... }' | jq '.result.value'
done
# 0.29
# 0.29
# 0.29
```

This is the core property: same inputs, same guardrails, same decision — every time.

---

## What Just Happened

```
You described intent → LLM compiled (once) → Condition locked
         ↓
Primitive value fetched → Concept computed → Condition evaluated → Action fired
         ↓
Deterministic, auditable, reproducible decision
```

**Without Memintel:**
```python
# LLM decides at runtime — non-deterministic, not auditable
if llm.assess("is this account at risk?"):
    trigger_alert()
```

**With Memintel:**
```
Plain English intent → Compiled once by LLM → Evaluated deterministically forever
```

The LLM is used exactly once, at task creation, within guardrails constraints. Every subsequent evaluation is pure computation — no LLM, no drift, full reproducibility.

---

## Next Steps

- [Application Context](/docs/intro/application-context) — improve compilation accuracy with domain knowledge
- [Admin Guide](/docs/admin-guide/admin-overview) — configure guardrails, primitives, and actions
- [API Reference](/docs/api-reference/overview) — full endpoint documentation
- [Case Studies](/docs/tutorials/deal-intelligence) — domain-specific examples
