---
id: admin-decision-traceability
title: Decision Traceability
sidebar_label: Decision Traceability
---

# Decision Traceability

Every decision Memintel makes is permanently recorded and fully queryable. You can retrieve the exact inputs, the exact logic, and the exact outcome for any decision — at any point in the future.

---

## What Is a Decision Record?

When a condition evaluates a concept, Memintel creates a decision record. This record is immutable — it is written once and never modified. It captures everything needed to understand, reproduce, and audit the decision:

```json
{
  "decision_id": "dec_abc123",
  "task_id": "task_ghi789",
  "task_version": "v2",
  "condition_id": "cond_xyz456",
  "condition_version": "v1",
  "entity_id": "ent_pseudonymised_001",
  "evaluated_at": "2025-11-14T09:23:41Z",
  "outcome": "triggered",
  "concept_result": {
    "value": 0.29,
    "type": "float"
  },
  "input_primitives": {
    "account.active_user_rate_30d": 0.29,
    "account.days_to_renewal": 47,
    "account.payment_failed_flag": false
  },
  "threshold_applied": 0.35,
  "strategy": "threshold",
  "direction": "below",
  "action_id": "slack_customer_success",
  "ir_hash": "sha256:7f3a9c..."
}
```

---

## Querying a Decision

Retrieve a decision by its `decision_id`:

```bash
curl https://api.memsdl.ai/v1/decisions/dec_abc123 \
  -H "X-API-Key: your-api-key"
```

The response includes the full decision record as shown above.

---

## Listing Decisions

Query decisions by entity, task, condition, or time range:

```bash
# All decisions for a specific entity
curl "https://api.memsdl.ai/v1/decisions?entity_id=ent_pseudonymised_001" \
  -H "X-API-Key: your-api-key"

# All triggered decisions for a task in a time range
curl "https://api.memsdl.ai/v1/decisions?task_id=task_ghi789&outcome=triggered&from=2025-11-01&to=2025-11-30" \
  -H "X-API-Key: your-api-key"

# All decisions for a condition version
curl "https://api.memsdl.ai/v1/decisions?condition_id=cond_xyz456&condition_version=v1" \
  -H "X-API-Key: your-api-key"
```

---

## Explaining a Decision

For any decision, you can request a plain-English explanation of why it fired — or why it did not fire:

```bash
curl -X POST https://api.memsdl.ai/v1/decisions/dec_abc123/explain \
  -H "X-API-Key: your-api-key"
```

Response:

```json
{
  "decision_id": "dec_abc123",
  "explanation": "The active user rate for this account was 0.29 — below the threshold of 0.35. The condition fired and triggered the customer success Slack alert.",
  "primitive_contributions": [
    {
      "primitive_id": "account.active_user_rate_30d",
      "value": 0.29,
      "contribution": "primary — this value fell below threshold"
    }
  ]
}
```

This explanation is grounded entirely in the stored decision record — it does not re-evaluate anything. It is safe to include in audit documentation.

---

## Replaying a Decision

You can replay any past decision to verify that the same inputs produce the same outcome:

```bash
curl -X POST https://api.memsdl.ai/v1/decisions/dec_abc123/replay \
  -H "X-API-Key: your-api-key"
```

The replay uses the original inputs and the original condition version — not the current state of either. If the replay produces a different outcome, it means something in the execution environment has changed in a way that should be investigated.

The `ir_hash` field in the decision record is a deterministic hash of the execution graph at the time of the decision. The replay verifies this hash before executing — if the graph has been tampered with, the replay will fail with a `409 ir_hash_mismatch` error.

---

## The ir_hash and Tamper Evidence

The `ir_hash` is a SHA-256 hash of the compiled execution graph. It is computed at compile time and stored in the decision record at evaluation time.

Because the execution graph is deterministic and immutable, the same condition version always produces the same `ir_hash`. This means:

- **Tamper detection** — if the execution graph is modified after the fact, the hash will not match and replay will fail
- **Cross-environment verification** — you can verify that the same logic was applied in staging and production by comparing `ir_hash` values
- **Regulatory attestation** — the `ir_hash` is machine-verifiable proof that the decision logic was unchanged between two dates

```bash
# Verify the ir_hash of a decision
curl https://api.memsdl.ai/v1/decisions/dec_abc123/verify \
  -H "X-API-Key: your-api-key"
```

```json
{
  "decision_id": "dec_abc123",
  "ir_hash": "sha256:7f3a9c...",
  "verified": true,
  "condition_id": "cond_xyz456",
  "condition_version": "v1"
}
```

---

## Domain Examples

### Clinical Trials — Adverse Event Traceability

A DSMB (Data Safety Monitoring Board) requests documentation of every adverse event alert generated during the trial period and the exact logic that produced each one.

```bash
# Retrieve all triggered decisions for the safety monitoring task
curl "https://api.memsdl.ai/v1/decisions?task_id=task_safety_monitor&outcome=triggered&from=2025-01-01&to=2025-06-30" \
  -H "X-API-Key: your-api-key"
```

For each decision, the DSMB can see:
- The exact AE severity score and relatedness signal that drove the alert
- The threshold that was active at the time
- The `ir_hash` proving the evaluation logic was unchanged throughout the trial

### Financial Services — Credit Decision Documentation

A regulator requests documentation of the basis for a credit early warning that triggered a covenant review.

```bash
curl https://api.memsdl.ai/v1/decisions/dec_credit_001/explain \
  -H "X-API-Key: your-api-key"
```

The explanation provides a plain-English account of which DSCR value triggered the alert and what threshold was applied — suitable for inclusion in the loan file.

### SaaS — Customer Dispute Resolution

A customer disputes a churn-risk alert that triggered an automated account review. The customer success team retrieves the decision record to explain exactly what signal drove it.

```bash
curl https://api.memsdl.ai/v1/decisions/dec_churn_042 \
  -H "X-API-Key: your-api-key"
```

The record shows the exact active user rate at the time, confirming the alert was correct. The team can share this with the customer as documentation.

---

## Common Mistakes

**Relying on current condition state to explain a past decision.** Always retrieve the decision record directly — do not re-evaluate with the current condition, which may have been calibrated since.

**Not storing `decision_id` in your downstream systems.** If you trigger a webhook or create a CRM record from a decision, store the `decision_id` alongside it. This makes it trivial to look up the full decision record later.

**Assuming `not_triggered` decisions are not recorded.** They are. Every evaluation produces a record, whether or not the condition fired. This is important for demonstrating that the system was monitoring continuously.
