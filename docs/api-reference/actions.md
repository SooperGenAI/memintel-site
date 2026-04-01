---
id: actions
title: Actions
sidebar_label: Actions
---

# Actions

Actions define what happens when a condition fires — Slack notifications, webhook calls, workflow triggers, or write-back to a primitive. Actions are registered independently and bound to conditions via the `trigger` config.

Actions are immutable once registered. To update an action, register a new version.

---

## Register an Action

```
POST /actions
```

Registers a new action. Requires the elevated key.

### Request Body — ActionDefinition

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action_id` | string | **Required** | Unique identifier for this action. |
| `version` | string | **Required** | Version string (e.g. `"v1"`). Once registered, `(action_id, version)` is permanent. |
| `config` | ActionConfig | **Required** | Action type and delivery parameters. See [Action Types](#action-types) below. |
| `trigger` | TriggerConfig | **Required** | When the action fires and which condition it is bound to. |
| `namespace` | string | **Required** | Organisation namespace (e.g. `"org"`). |

### TriggerConfig

| Parameter | Type | Required | Description |
|---|---|---|---|
| `fire_on` | enum | **Required** | `"true"` \| `"false"` \| `"any"`. When to fire relative to the condition decision. |
| `condition_id` | string | **Required** | The condition this action is bound to. |
| `condition_version` | string | **Required** | The specific condition version. |

**`fire_on` values:**

| Value | When the action fires |
|---|---|
| `"true"` | When the condition evaluates to true (or the matched label for `equals` strategy) |
| `"false"` | When the condition evaluates to false |
| `"any"` | On every evaluation, regardless of outcome |

### Response Codes

| Status | Description |
|---|---|
| **200** | Action registered. |
| **400** | Invalid request body or unknown action type. |
| **401** | Unauthorised. |
| **403** | Elevated key required. |
| **409** | `(action_id, version)` already exists. |

### TypeScript Example

```typescript
await client.actions.register({
  actionId: "slack_cs_alert",
  version: "v1",
  config: {
    type: "notification",
    channel: "slack-customer-success",
    messageTemplate: "Churn risk: {entity} active rate {decision_value}",
  },
  trigger: {
    fireOn: "true",
    conditionId: "cond_churn_risk",
    conditionVersion: "v1",
  },
  namespace: "org",
});
```

---

## List Actions

```
GET /actions
```

Returns a paginated list of registered actions for a namespace.

### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `namespace` | string | Optional | Filter by namespace. Defaults to `"org"`. |
| `limit` | integer | Optional | Default `50`. Maximum `200`. |
| `cursor` | string | Optional | Pagination cursor from previous response. |

### Response

| Parameter | Type | Description |
|---|---|---|
| `items` | array | Array of `ActionDefinition` records. |
| `has_more` | boolean | Whether more results are available. |
| `next_cursor` | string \| null | Pass as `cursor` on the next request. |
| `total_count` | integer | Total number of registered actions in this namespace. |

### Response Codes

| Status | Description |
|---|---|
| **200** | Action list. |
| **401** | Unauthorised. |

### TypeScript Example

```typescript
const actions = await client.actions.list({ namespace: "org" });

actions.items.forEach(a => {
  console.log(a.action_id, a.version, a.config.type);
});
```

---

## Trigger an Action

```
POST /actions/{action_id}/trigger
```

Triggers a registered action directly for a given entity, bypassing the full pipeline. Use for testing action configuration before go-live. Supports dry run mode.

### Path Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action_id` | string | **Required** | The action to trigger. |

### Request Body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `version` | string | **Required** | The action version to trigger. |
| `entity` | string | **Required** | The entity to use when constructing the action payload. |
| `timestamp` | datetime | Optional | ISO 8601 UTC. Used in the action payload. |
| `dry_run` | boolean | Optional | Default `false`. Simulate without making external calls. Returns `status: "would_trigger"`. |

### Response — ActionResult

| Parameter | Type | Description |
|---|---|---|
| `action_id` | string | The triggered action. |
| `action_version` | string | The version triggered. |
| `status` | enum | `triggered` \| `skipped` \| `failed` \| `would_trigger` |
| `payload_sent` | object \| null | The actual payload delivered (populated when triggered). |
| `error` | object \| null | Error details (populated when `status: "failed"`). |

### Response Codes

| Status | Description |
|---|---|
| **200** | Action triggered, skipped, or simulated. |
| **401** | Unauthorised. |
| **404** | Action not found. |

### TypeScript Example

```typescript
// Dry run — simulate without external calls
const result = await client.actions.trigger("slack_cs_alert", {
  version: "v1",
  entity: "account_xyz789",
  dryRun: true,
});

console.log(result.status); // "would_trigger"

// Real trigger
const live = await client.actions.trigger("slack_cs_alert", {
  version: "v1",
  entity: "account_xyz789",
});

console.log(live.status); // "triggered"
```

---

## Action Types {#action-types}

The `config` object is a discriminated union keyed on `type`.

### notification

Push alert to a named notification channel (Slack, PagerDuty, etc.).

```json
{
  "type": "notification",
  "channel": "slack-customer-success",
  "message_template": "Churn risk: {entity} active rate {decision_value}"
}
```

| Field | Required | Description |
|---|---|---|
| `channel` | **Required** | Named notification channel. |
| `message_template` | Optional | Format string. Omit for default decision summary. |

### webhook

HTTP POST to an external endpoint.

```json
{
  "type": "webhook",
  "endpoint": "https://myapp.com/hooks/alert",
  "method": "POST",
  "headers": { "Authorization": "Bearer ${CRM_SECRET}" },
  "payload_template": { "entity": "{entity}" }
}
```

| Field | Required | Description |
|---|---|---|
| `endpoint` | **Required** | Target URL. |
| `method` | Optional | HTTP method. Default `"POST"`. |
| `headers` | Optional | HTTP headers. Use `${ENV_VAR}` for secrets. |
| `payload_template` | Optional | Request body template. Omit for default decision payload. |

:::warning At-most-once delivery
Actions are best-effort — no automatic retry. A failed webhook is recorded as `failed` in the decision record; the pipeline returns HTTP 200 regardless. Design downstream systems to be idempotent.
:::

### workflow

Trigger a registered workflow engine.

```json
{
  "type": "workflow",
  "workflow_id": "credit_review_workflow",
  "input_mapping": { "borrower": "entity", "score": "decision_value" }
}
```

| Field | Required | Description |
|---|---|---|
| `workflow_id` | **Required** | ID of the registered workflow. |
| `input_mapping` | Optional | Maps workflow input names to decision fields. Omit to forward the full decision payload. |

### register

Write the decision result back to a primitive (closed-loop feedback).

```json
{
  "type": "register",
  "primitive_id": "account.churn_risk_flag",
  "entity_field": "entity"
}
```

| Field | Required | Description |
|---|---|---|
| `primitive_id` | **Required** | The primitive to update (`namespace.field` format). |
| `entity_field` | Optional | Which decision field carries the entity ID. Default `"entity"`. |
