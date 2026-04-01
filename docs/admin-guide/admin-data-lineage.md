---
id: admin-data-lineage
title: Data Lineage
sidebar_label: Data Lineage
---

# Data Lineage

For any decision Memintel makes, you can trace the complete chain from raw data to outcome: which primitive values were fetched, how they were combined into a concept, which condition evaluated that concept, and which action was taken. This chain is the data lineage of the decision.

---

:::note Planned — Lineage API in development
The endpoints documented on this page are planned and not yet implemented. The data they will return — primitive values, computation graphs, and DAG structures — is captured internally during evaluation. API access to this data is on the roadmap. The `input_primitives` and `signal_errors` fields in the decision record provide primitive-level lineage today via `GET /decisions/{id}`.
:::

## The Lineage Chain

Every Memintel decision flows through a fixed, deterministic pipeline:

```
Primitive values  →  Concept computation  →  Condition evaluation  →  Action
```

Each stage is recorded in the decision record and queryable independently:

| Stage | What it produces | Recorded as |
|---|---|---|
| **Primitive fetch** | Raw signal values for the entity at evaluation time | `input_primitives` in decision record |
| **Concept computation** | Derived meaning from primitives | `concept_result` in decision record |
| **Condition evaluation** | Significance judgment — triggered or not | `outcome`, `threshold_applied` in decision record |
| **Action** | Delivery of the alert or webhook | `action_id` in decision record |

---

## Retrieving Full Lineage

For any decision, retrieve the complete lineage:

```bash
curl https://api.memsdl.ai/v1/decisions/dec_abc123/lineage \
  -H "X-API-Key: your-api-key"
```

```json
{
  "decision_id": "dec_abc123",
  "entity_id": "ent_pseudonymised_001",
  "evaluated_at": "2025-11-14T09:23:41Z",
  "lineage": {
    "primitives": {
      "account.active_user_rate_30d": {
        "value": 0.29,
        "fetched_from": "activity_pipeline",
        "fetched_at": "2025-11-14T09:23:39Z"
      },
      "account.days_to_renewal": {
        "value": 47,
        "fetched_from": "billing_pipeline",
        "fetched_at": "2025-11-14T09:23:39Z"
      }
    },
    "concept": {
      "concept_id": "concept_churn_risk",
      "computation": "weighted_composite(account.active_user_rate_30d, account.days_to_renewal)",
      "result": 0.29,
      "ir_hash": "sha256:7f3a9c..."
    },
    "condition": {
      "condition_id": "cond_xyz456",
      "condition_version": "v1",
      "strategy": "threshold",
      "threshold": 0.35,
      "direction": "below",
      "outcome": "triggered"
    },
    "action": {
      "action_id": "slack_customer_success",
      "action_type": "notification",
      "channel": "slack",
      "delivered_at": "2025-11-14T09:23:42Z",
      "delivery_status": "success"
    }
  }
}
```

---

## Primitive-Level Lineage

You can trace any primitive value back to its source:

```bash
curl "https://api.memsdl.ai/v1/decisions/dec_abc123/lineage/primitives" \
  -H "X-API-Key: your-api-key"
```

The response shows each primitive, its value, which data pipeline provided it, and when it was fetched. This is the bridge between Memintel's decision record and your upstream data systems — enabling you to trace a decision all the way back to the source database record.

---

## Concept Computation Lineage

The concept computation shows exactly how primitive values were combined into the concept result:

```bash
curl "https://api.memsdl.ai/v1/decisions/dec_abc123/lineage/concept" \
  -H "X-API-Key: your-api-key"
```

```json
{
  "concept_id": "concept_churn_risk",
  "computation_graph": {
    "nodes": [
      { "id": "n1", "type": "primitive", "primitive_id": "account.active_user_rate_30d", "value": 0.29 },
      { "id": "n2", "type": "primitive", "primitive_id": "account.days_to_renewal", "value": 47 },
      { "id": "n3", "type": "transform", "operation": "normalise", "input": "n2", "output": 0.53 },
      { "id": "n4", "type": "aggregate", "operation": "weighted_mean", "inputs": ["n1", "n3"], "weights": [0.7, 0.3], "output": 0.29 }
    ],
    "result": 0.29
  }
}
```

This computation graph is deterministic — the same inputs always produce the same result through the same graph. The `ir_hash` in the decision record is a hash of this graph.

---

## DAG Mapping

For complex deployments with multiple tasks sharing primitives and concepts, you can retrieve the full directed acyclic graph (DAG) of dependencies:

```bash
curl https://api.memsdl.ai/v1/tasks/task_ghi789/dag \
  -H "X-API-Key: your-api-key"
```

The DAG shows:
- Which primitives feed into which concepts
- Which concepts are evaluated by which conditions
- Which conditions are bound to which tasks
- Which tasks trigger which actions

This is particularly useful for impact analysis — understanding which tasks and decisions are affected by a change to a primitive or concept.

---

## Impact Analysis

Before changing a primitive definition or data source, understand what depends on it:

```bash
# What would be affected by changing this primitive?
curl https://api.memsdl.ai/v1/primitives/account.active_user_rate_30d/impact \
  -H "X-API-Key: your-api-key"
```

```json
{
  "primitive_id": "account.active_user_rate_30d",
  "depends_on_this": {
    "concepts": ["concept_churn_risk", "concept_account_health"],
    "conditions": ["cond_xyz456", "cond_health_001"],
    "tasks": ["task_ghi789", "task_health_monitor"],
    "decisions_last_30d": 847
  }
}
```

---

## Domain Examples

### Healthcare — Payor/Provider Billing Audit

A healthcare payor is auditing a claim decision. The auditor needs to see exactly which data values drove the billing anomaly alert.

```bash
curl https://api.memsdl.ai/v1/decisions/dec_billing_042/lineage \
  -H "X-API-Key: your-api-key"
```

The lineage shows:
- The provider's billing deviation percentile (from the benchmarking pipeline)
- The peer group used for comparison
- The concept computation that derived the anomaly score
- The condition threshold that triggered the alert

This complete chain — from raw billing data to alert — is the evidence trail for the audit.

### XBRL Filing Compliance

A financial reporting team needs to demonstrate that a deprecated XBRL tag warning was triggered by the correct taxonomy data.

```bash
curl https://api.memsdl.ai/v1/decisions/dec_xbrl_007/lineage/primitives \
  -H "X-API-Key: your-api-key"
```

The primitive lineage shows exactly which taxonomy feed version provided the `taxonomy.tag_deprecated_flag` value, and when it was fetched — proving the alert was based on the correct regulatory data at the time of filing.

### AML Transaction Monitoring

A compliance officer needs to demonstrate to a regulator that a transaction flagged for AML review was identified by a documented, validated rule — not an opaque model.

```bash
curl https://api.memsdl.ai/v1/decisions/dec_aml_118/lineage \
  -H "X-API-Key: your-api-key"
```

The full lineage shows the exact transaction amount, the customer's 90-day average, the ratio computed by the concept, the threshold applied, and the condition version — providing the documented, auditable basis for the referral that regulators require.

### DevOps — Incident Post-Mortem

After a production incident triggered by a deployment block, the SRE team needs to understand exactly what signals drove the decision to block the deployment.

```bash
curl https://api.memsdl.ai/v1/decisions/dec_deploy_block_033/lineage \
  -H "X-API-Key: your-api-key"
```

The lineage shows the exact error rate, latency percentile, and change rate values that were present at the time of the decision — providing the factual basis for the post-mortem.

---

## Common Mistakes

**Not tracing data quality issues through the lineage.** When a decision looks wrong, the first step is to check the primitive values in the lineage. Many apparent decision errors are actually data pipeline issues — a missing value, a stale fetch, or a calculation error upstream.

**Assuming the concept computation is a black box.** The computation graph is fully inspectable. If a concept result looks unexpected, retrieve the computation lineage and check each node.

**Not running impact analysis before changing data sources.** Changing a primitive's data source or calculation logic affects every concept, condition, and task downstream. Always run the impact analysis before making changes to understand the full scope.
