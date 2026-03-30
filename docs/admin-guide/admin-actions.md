---
id: admin-actions
title: Step 4 — Actions
sidebar_label: Step 4 — Actions
---

# Step 4 — Actions

An action is what happens when a monitoring condition fires. It is the delivery mechanism — where does the alert go, and in what form?

You register a library of available actions via the API. When monitoring tasks are created, they bind to a specific action. You can have as many actions as you need — a Slack alert for one team, a webhook to your CRM for another, a workflow trigger for a third.

---

## How Actions Are Registered

Actions are registered via `POST /actions` and stored in the database. There is no actions section in `memintel_config.yaml`.

```bash
curl -X POST https://api.memsdl.ai/v1/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -H "Content-Type: application/json" \
  -d '{
    "action_id": "slack_customer_success",
    "version": "v1",
    "config": {
      "type": "notification",
      "channel": "slack-cs-alerts",
      "message_template": "Churn risk alert for {entity} — active user rate dropped to {decision_value}"
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

## The Action Structure

Every action definition has four required fields:

### action_id — the action's name

A unique identifier for this action. Make it descriptive — it's what your team sees when tasks are created.

```
slack_customer_success
pagerduty_critical
crm_churn_webhook
workflow_credit_review
```

### version — the action version

Actions are versioned and immutable. Once registered, an `(action_id, version)` pair is permanent. To update an action, register a new version.

```json
"version": "v1"
```

### config — the action type and delivery parameters

The config object defines the action type and how it delivers. The `type` field is the discriminator — see [Four Action Types](#four-action-types) below.

### trigger — when the action fires

The trigger defines when the action fires and which condition it is bound to:

```json
"trigger": {
  "fire_on": "true",
  "condition_id": "cond_churn_risk",
  "condition_version": "v1"
}
```

**`fire_on` options:**

| Value | When the action fires |
|---|---|
| `"true"` | When the condition evaluates to true (or the matched label for `equals` strategy) |
| `"false"` | When the condition evaluates to false |
| `"any"` | On every evaluation, regardless of outcome |

---

## Four Action Types

### notification — push alerts to a channel

Use for Slack messages, PagerDuty, or any named notification channel. Memintel sends the message to the configured channel.

```json
{
  "type": "notification",
  "channel": "slack-cs-alerts",
  "message_template": "Churn risk: {entity} active rate {decision_value}"
}
```

| Field | Required | Description |
|---|---|---|
| `type` | Yes | `"notification"` |
| `channel` | Yes | Named notification channel |
| `message_template` | No | Message format string — omit for default decision summary |

### webhook — HTTP POST to an external system

Use for CRMs, ticketing systems, incident management tools, or any system with an API endpoint.

```json
{
  "type": "webhook",
  "endpoint": "https://myapp.com/hooks/alert",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer ${CRM_WEBHOOK_SECRET}"
  },
  "payload_template": {
    "entity": "{entity}",
    "alert_type": "churn_risk"
  }
}
```

| Field | Required | Description |
|---|---|---|
| `type` | Yes | `"webhook"` |
| `endpoint` | Yes | Target URL |
| `method` | No | HTTP method — defaults to `"POST"` |
| `headers` | No | HTTP headers — use `${ENV_VAR}` for secrets |
| `payload_template` | No | Request body template — omit for default decision payload |

:::warning No automatic retries
Actions are best-effort — at-most-once per evaluation. A failed action is recorded as `failed` in the decision record; the pipeline returns HTTP 200 regardless. Design your downstream systems to be idempotent.
:::

### workflow — trigger an external workflow engine

Use to trigger a registered workflow in a connected workflow engine.

```json
{
  "type": "workflow",
  "workflow_id": "credit_review_workflow",
  "input_mapping": {
    "borrower": "entity",
    "dscr_value": "decision_value"
  }
}
```

| Field | Required | Description |
|---|---|---|
| `type` | Yes | `"workflow"` |
| `workflow_id` | Yes | ID of the registered workflow to trigger |
| `input_mapping` | No | Maps workflow input parameter names to decision fields — omit to forward the full decision payload |

### register — write the decision result back to a primitive

Use to close the loop — write a decision outcome back into the Memintel primitive registry so future concept evaluations can consume it.

```json
{
  "type": "register",
  "primitive_id": "account.churn_risk_flag",
  "entity_field": "entity"
}
```

| Field | Required | Description |
|---|---|---|
| `type` | Yes | `"register"` |
| `primitive_id` | Yes | The primitive to update (format: `namespace.field`) |
| `entity_field` | No | Which field from the decision carries the entity ID — defaults to `"entity"` |

---

## Using Environment Variables for Secrets

Never put real API keys, webhook secrets, or tokens directly in an action config. Use `${VARIABLE_NAME}` references instead — the server resolves these from environment variables at startup.

```json
// Wrong — secret hardcoded
"headers": { "Authorization": "Bearer sk-real-secret-token" }

// Right — secret stored as environment variable
"headers": { "Authorization": "Bearer ${CRM_WEBHOOK_SECRET}" }
```

Ask your data engineer to set the corresponding environment variables on the server.

---

## Testing an Action

To verify an action before go-live, trigger it directly for a test entity:

```bash
curl -X POST https://api.memsdl.ai/v1/actions/slack_customer_success/trigger \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "version": "v1",
    "entity": "test_entity_001",
    "dry_run": true
  }'
```

With `dry_run: true`, the action is simulated without making any real HTTP calls or external side effects. The response shows `status: "would_trigger"`.

Remove `dry_run` to trigger the action for real against the test entity.

---

## Complete Examples

### SaaS Platform

```bash
# Slack alert — customer success team
curl -X POST .../v1/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -d '{
  "action_id": "slack_cs_team",
  "version": "v1",
  "config": { "type": "notification", "channel": "slack-cs-alerts" },
  "trigger": { "fire_on": "true", "condition_id": "cond_churn_risk", "condition_version": "v1" },
  "namespace": "org"
}'

# CRM webhook — trigger outreach sequence
curl -X POST .../v1/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -d '{
  "action_id": "crm_churn_webhook",
  "version": "v1",
  "config": {
    "type": "webhook",
    "endpoint": "https://crm.myapp.com/hooks/churn",
    "headers": { "Authorization": "Bearer ${CRM_SECRET}" }
  },
  "trigger": { "fire_on": "true", "condition_id": "cond_churn_risk", "condition_version": "v1" },
  "namespace": "org"
}'

# Write-back — update churn risk flag primitive
curl -X POST .../v1/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -d '{
  "action_id": "write_churn_flag",
  "version": "v1",
  "config": { "type": "register", "primitive_id": "account.churn_risk_flag" },
  "trigger": { "fire_on": "any", "condition_id": "cond_churn_risk", "condition_version": "v1" },
  "namespace": "org"
}'
```

### Financial Services / Compliance

```bash
# Compliance notification
curl -X POST .../v1/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -d '{
  "action_id": "compliance_alert",
  "version": "v1",
  "config": { "type": "notification", "channel": "slack-compliance" },
  "trigger": { "fire_on": "true", "condition_id": "cond_aml_signal", "condition_version": "v1" },
  "namespace": "org"
}'

# Compliance system webhook — high priority
curl -X POST .../v1/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -d '{
  "action_id": "compliance_system_webhook",
  "version": "v1",
  "config": {
    "type": "webhook",
    "endpoint": "https://compliance.myapp.com/hooks/aml",
    "headers": {
      "Authorization": "Bearer ${COMPLIANCE_SECRET}",
      "X-Alert-Priority": "high"
    }
  },
  "trigger": { "fire_on": "true", "condition_id": "cond_aml_signal", "condition_version": "v1" },
  "namespace": "org"
}'
```

### Clinical Trials / Healthcare

```bash
# Safety alert — medical monitor
curl -X POST .../v1/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -d '{
  "action_id": "safety_alert",
  "version": "v1",
  "config": { "type": "notification", "channel": "slack-medical-monitor" },
  "trigger": { "fire_on": "true", "condition_id": "cond_ae_severity", "condition_version": "v1" },
  "namespace": "org"
}'

# Pharmacovigilance system webhook
curl -X POST .../v1/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -d '{
  "action_id": "safety_system_webhook",
  "version": "v1",
  "config": {
    "type": "webhook",
    "endpoint": "https://safety.myapp.com/hooks/ae-alert",
    "headers": { "Authorization": "Bearer ${SAFETY_SECRET}" }
  },
  "trigger": { "fire_on": "true", "condition_id": "cond_ae_severity", "condition_version": "v1" },
  "namespace": "org"
}'
```

### DevOps / SRE

```bash
# PagerDuty — critical SLO breach
curl -X POST .../v1/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -d '{
  "action_id": "pagerduty_critical",
  "version": "v1",
  "config": {
    "type": "webhook",
    "endpoint": "https://events.pagerduty.com/v2/enqueue",
    "headers": { "Authorization": "Token token=${PAGERDUTY_KEY}" }
  },
  "trigger": { "fire_on": "true", "condition_id": "cond_slo_breach", "condition_version": "v1" },
  "namespace": "org"
}'

# CI/CD deployment block workflow
curl -X POST .../v1/actions \
  -H "X-Elevated-Key: your-elevated-key" \
  -d '{
  "action_id": "block_deployment",
  "version": "v1",
  "config": {
    "type": "workflow",
    "workflow_id": "deployment_block_workflow",
    "input_mapping": { "service": "entity", "risk_score": "decision_value" }
  },
  "trigger": { "fire_on": "true", "condition_id": "cond_deployment_risk", "condition_version": "v1" },
  "namespace": "org"
}'
```

---

## Common Mistakes

**Using `log_only` as an action type.** This type does not exist. If you want a decision recorded without any external delivery, use a `webhook` pointing to an internal audit endpoint, or rely on the decision record that Memintel creates for every evaluation automatically.

**Expecting automatic retries.** Actions are at-most-once. If a webhook fails, it is recorded as `failed` but not retried. Design your downstream systems to handle this — use idempotency keys based on `(condition_id, condition_version, entity, timestamp)`.

**Putting secrets directly in the config.** Use `${ENV_VAR}` references for all credentials. Ask your data engineer to set the corresponding environment variables on the server.

**Not testing actions before go-live.** Use `POST /actions/{id}/trigger` with `dry_run: true` to verify each action is reachable and configured correctly before users start creating tasks.

**Using one action for everything.** A critical safety signal and a low-priority early warning should not go to the same channel. Register separate actions for different priority levels and condition types.
