---
id: decisions
title: Decisions
sidebar_label: Decisions
---

# Decisions

The decision store records every evaluation that Memintel performs — triggered or not. Each record is immutable: written once, never modified. Calibrations, guardrails updates, and task changes all create new versions — historical decisions are untouched.

---

## List Decisions

```
GET /decisions
```

Returns a paginated list of decision records. Filter by entity, condition, outcome, or time range.

### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `entity_id` | string | Optional | Filter to decisions for a specific entity. |
| `condition_id` | string | Optional | Filter to decisions for a specific condition. |
| `condition_version` | string | Optional | Filter to a specific condition version. Requires `condition_id`. |
| `outcome` | enum | Optional | `triggered` \| `not_triggered`. |
| `from` | datetime | Optional | ISO 8601 UTC. Start of time range (inclusive). |
| `to` | datetime | Optional | ISO 8601 UTC. End of time range (inclusive). |
| `limit` | integer | Optional | Default `50`. Maximum `200`. |
| `cursor` | string | Optional | Pagination cursor from previous response. |

### Response

| Parameter | Type | Description |
|---|---|---|
| `items` | array | Array of decision records. See [Decision Record](#decision-record) below. |
| `has_more` | boolean | Whether more results are available. |
| `next_cursor` | string \| null | Pass as `cursor` on the next request. |

### Response Codes

| Status | Description |
|---|---|
| **200** | Decision list. |
| **400** | Invalid filter parameters. |
| **401** | Unauthorised. |

### TypeScript Example

```typescript
// All triggered decisions for an account in the last 30 days
const decisions = await client.decisions.list({
  entityId: "account_xyz789",
  outcome: "triggered",
  from: "2025-10-01T00:00:00Z",
  to: "2025-10-31T23:59:59Z",
});

decisions.items.forEach(d => {
  console.log(d.decision_id, d.outcome, d.evaluated_at);
});
```

---

## Get Decision

```
GET /decisions/{decision_id}
```

Returns the full decision record for a specific decision.

### Path Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `decision_id` | string | **Required** | Unique decision identifier. |

### Response — Decision Record {#decision-record}

| Parameter | Type | Description |
|---|---|---|
| `decision_id` | string | Unique identifier for this decision. |
| `condition_id` | string | The condition that evaluated this decision. |
| `condition_version` | string | The specific condition version that was active. |
| `concept_result` | object | The computed concept value that was evaluated (`value`, `type`). |
| `input_primitives` | object | The raw primitive values that drove the concept, keyed by primitive ID. |
| `signal_errors` | array | Which signal fetches failed during evaluation, with error details — distinguishes connector failure from legitimate null. |
| `threshold_applied` | number \| null | The exact parameter value in effect at decision time. `null` for `equals` and `composite` strategies. |
| `strategy` | string | The strategy type applied (`threshold`, `percentile`, `z_score`, `change`, `equals`, `composite`). |
| `outcome` | enum | `triggered` \| `not_triggered`. |
| `action_id` | string \| null | The action that was taken (if triggered). |
| `entity_id` | string | The entity this decision relates to (pseudonymised). |
| `evaluated_at` | datetime | ISO 8601 UTC timestamp of evaluation. |
| `ir_hash` | string | SHA-256 hash of the execution graph — machine-verifiable proof that the logic was unchanged. |

### Response Codes

| Status | Description |
|---|---|
| **200** | Decision record. |
| **401** | Unauthorised. |
| **404** | Decision not found. |

### TypeScript Example

```typescript
const decision = await client.decisions.get("dec_abc123");

console.log(decision.outcome);           // "triggered"
console.log(decision.threshold_applied); // 0.35
console.log(decision.input_primitives);  // { "account.active_user_rate_30d": 0.29, ... }
console.log(decision.ir_hash);           // "sha256:7f3a9c..."
```

---

## Explain Decision

```
POST /decisions/explain
```

Returns a plain-English explanation of a specific decision: the Result (Rₜ) that was evaluated, the strategy and parameters applied, whether the condition fired, and the contribution of each input signal.

This endpoint is grounded entirely in the stored decision record — it does not re-evaluate anything. The explanation is safe to include in audit documentation.

**Boolean strategies** (`threshold`, `percentile`, `change`, `z_score`, `composite`): `decision` is `true`/`false`. `threshold_applied` contains the numeric cutoff compared against the Result (Rₜ).

**Equals strategy:** `decision` is the matched label string. `label_matched` contains it explicitly. `threshold_applied` is `null`.

### Request Body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `condition_id` | string | **Required** | Fully qualified condition identifier. |
| `condition_version` | string | **Required** | Explicit condition version. |
| `entity` | string | **Required** | Entity the Decision (Aₜ) was made for. |
| `timestamp` | datetime | **Required** | ISO 8601 UTC. Retrieves the stored Decision (Aₜ) at this timestamp. |

### Response — DecisionExplanation

| Parameter | Type | Required | Description |
|---|---|---|---|
| `decision_type` | enum | Always | `boolean` \| `categorical`. |
| `decision` | boolean\|string | Always | Decision (Aₜ) value: `true`/`false` or matched label. |
| `concept_value` | number\|string | Always | The Result (Rₜ) that was evaluated. |
| `strategy_type` | enum | Always | `threshold` \| `percentile` \| `change` \| `z_score` \| `equals` \| `composite`. |
| `threshold_applied` | number | Conditional | Numeric cutoff. Present for `threshold`, `percentile`, `change`, `z_score`. `null` for `equals`, `composite`. |
| `label_matched` | string | Conditional | Matched label for `equals` strategy. `null` if no match. |
| `drivers` | array | Always | `signal`, `contribution`, `value` per input signal. |

### Response Codes

| Status | Description |
|---|---|
| **200** | DecisionExplanation. |
| **401** | Unauthorised. |
| **404** | Decision not found. |

### TypeScript Example

```typescript
// Boolean decision
const exp = await client.decisions.explain({
  conditionId: "org.high_churn",
  conditionVersion: "1.0",
  entity: "user_abc123",
  timestamp: "2024-03-15T09:00:00Z",
});

console.log(exp.decision_type);      // "boolean"
console.log(exp.decision);           // true  (Aₜ)
console.log(exp.concept_value);      // 0.87  (Rₜ)
console.log(exp.threshold_applied);  // 0.8

// Categorical decision (equals strategy)
const cat = await client.decisions.explain({
  conditionId: "org.segment_check",
  conditionVersion: "1.0",
  entity: "user_abc123",
  timestamp: "2024-03-15T09:00:00Z",
});

console.log(cat.decision_type);      // "categorical"
console.log(cat.decision);           // "high_risk"
console.log(cat.threshold_applied);  // null
```
