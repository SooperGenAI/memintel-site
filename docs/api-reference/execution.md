---
id: execution
title: Execution
sidebar_label: Execution
---

# Execution

Fully deterministic — no LLM involved. The same entity, concept version, condition version, and timestamp always produce the same Result (Rₜ) and Decision (Aₜ).

---

## Execute Full Pipeline

```
POST /evaluate/full
```

Runs concept execution (ψ), condition evaluation (φ), and action triggering (α) in one atomic request. Returns a `FullPipelineResult` containing the Result (Rₜ) and Decision (Aₜ).

**Determinism:** Provide a `timestamp` for fully deterministic, reproducible execution. Omit for snapshot mode — current data, not guaranteed reproducible.

:::tip dry_run
Pass `dry_run: true` to simulate without firing actions. `decision.actions_triggered[].status` will be `"would_trigger"`.
:::

### Request Body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `concept_id` / `concept_version` | string | **Required** | Pinned concept reference. |
| `condition_id` / `condition_version` | string | **Required** | Pinned condition reference. |
| `entity` | string | **Required** | Entity to evaluate. |
| `timestamp` | datetime | Optional | ISO 8601 UTC. Provides deterministic execution. |
| `explain` | boolean | Optional | Default `false`. Include `Explanation` in Result (Rₜ). |
| `dry_run` | boolean | Optional | Default `false`. Simulate without firing actions. |

### Response — FullPipelineResult

| Parameter | Type | Required | Description |
|---|---|---|---|
| `result` | Result (Rₜ) | Always | Concept execution output. |
| `result.value` | number\|bool\|string | Always | Computed output. Type matches declared output type. |
| `result.deterministic` | boolean | Always | `true` when `timestamp` was provided. |
| `decision` | Decision (Aₜ) | Always | Condition evaluation output. |
| `decision.value` | boolean\|string | Always | `true`/`false` for boolean strategies; matched label for `equals`. |
| `decision.actions_triggered` | array | Always | `action_id`, `status`, `payload_sent`, `error?` per action. |

### Response Codes

| Status | Description |
|---|---|
| **200** | Pipeline executed. Action failures return 200 with per-action status. |
| **401** | Unauthorised. |
| **404** | Concept, condition, or entity not found. |
| **408** | Execution timed out (30s). |
| **422** | Execution failed — missing data or null propagation. |
| **429** | Rate limit. |

### TypeScript Example

```typescript
const pipeline = await client.evaluateFull({
  conceptId: "org.churn_risk",
  conceptVersion: "1.2",
  conditionId: "org.high_churn",
  conditionVersion: "1.0",
  entity: "user_abc123",
  timestamp: new Date().toISOString(),
  explain: true,
});

console.log(pipeline.result.value);        // 0.87  (Result Rₜ)
console.log(pipeline.result.deterministic); // true
console.log(pipeline.decision.value);       // true  (Decision Aₜ)

pipeline.decision.actions_triggered.forEach(a =>
  console.log(a.action_id, a.status)
);
```

---

## Evaluate a Condition with Inline Data (Test Only)

```
POST /execute/static
```

Evaluates a registered condition using caller-supplied primitive values. No real data connectors or `memintel_config.yaml` entries needed. Use for local testing and smoke testing new conditions.

:::warning Test only
This endpoint bypasses the production data pipeline entirely. Do not use it in production workflows.
:::

### Request Body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `condition_id` | string | **Required** | The condition to evaluate. |
| `condition_version` | string | **Required** | The specific condition version. |
| `entity` | string | **Required** | Entity identifier. |
| `data` | object | **Required** | Inline primitive values — `{ primitive_name: value }`. |

### Response — DecisionValue

| Parameter | Type | Description |
|---|---|---|
| `value` | boolean\|string | The decision outcome. `true`/`false` for boolean strategies; matched label for `equals`. |
| `decision_type` | enum | `boolean` \| `categorical`. |
| `strategy` | string | The strategy that was applied. |
| `threshold_applied` | number\|null | The parameter value used. `null` for `equals` and `composite`. |
| `reason` | string\|null | `null_input`, `insufficient_history`, or `history_unavailable` — present when the strategy could not fully evaluate. |
| `history_count` | integer\|null | Number of historical results available when `reason` is `insufficient_history`. |

### Response Codes

| Status | Description |
|---|---|
| **200** | DecisionValue returned. |
| **404** | Condition not found. |
| **422** | Execution failed — missing data or type mismatch. |

### TypeScript Example

```typescript
const result = await client.execute.static({
  conditionId: "cond_churn_risk",
  conditionVersion: "v1",
  entity: "account_xyz789",
  data: {
    "account.active_user_rate_30d": 0.29,
    "account.days_to_renewal": 47,
  },
});

console.log(result.value);             // true
console.log(result.decision_type);     // "boolean"
console.log(result.threshold_applied); // 0.35
```
