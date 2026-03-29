---
id: admin-governance-overview
title: Governance
sidebar_label: Overview
---

# Governance

Memintel's deterministic architecture makes governance a first-class property of the system — not an add-on. Because every decision flows through a fixed, versioned pipeline, the system can answer questions that LLM-based decision systems fundamentally cannot:

- **What rule was active when this decision was made?**
- **Why did this condition fire — what values drove it?**
- **Has this threshold ever changed, and if so, why?**
- **Can we prove that the decision logic was unchanged between these two dates?**

This section covers how to use Memintel's governance capabilities in practice.

---

## What Memintel Records

Every decision produces a permanent, immutable record containing:

| Field | What it captures |
|---|---|
| `decision_id` | Unique identifier for this decision |
| `task_id` + `task_version` | Which task produced this decision, at which version |
| `condition_id` + `condition_version` | Which condition evaluated it, at which version |
| `concept_result` | The computed concept value that was evaluated |
| `input_primitives` | The raw primitive values that drove the concept |
| `threshold_applied` | The exact parameter value in effect at decision time |
| `outcome` | `triggered` or `not_triggered` |
| `action_id` | Which action was taken (if triggered) |
| `entity_id` | The entity this decision relates to (pseudonymised) |
| `evaluated_at` | ISO 8601 timestamp |
| `ir_hash` | Hash of the execution graph — proves the logic was unchanged |

Nothing in this record is ever modified. Calibrations, guardrails updates, and task changes all create new versions — the historical record is untouched.

---

## The Governance Stack

Memintel's governance capabilities are organised into four layers:

### 1 — Decision Traceability
Every decision is queryable by `decision_id`. You can retrieve the full decision record, replay the evaluation with the original inputs, and inspect the exact logic that was applied. See [Decision Traceability](/docs/admin-guide/admin-decision-traceability).

### 2 — Version History and Immutability
Conditions, guardrails, and application context are all versioned. Every change creates a new version — nothing is ever overwritten. You can retrieve any historical version and compare it against the current state. See [Version History and Immutability](/docs/admin-guide/admin-version-history).

### 3 — Audit Trail
Every system change — guardrails updates, calibrations, task rebinds — is logged with a timestamp, the actor who made the change, and an optional note. The audit log is append-only. See [Audit Trail](/docs/admin-guide/admin-audit-trail).

### 4 — Data Lineage
For any decision, you can trace the full chain: which primitive values were fetched, how the concept was computed from them, which condition evaluated the concept, and which action was taken. See [Data Lineage](/docs/admin-guide/admin-data-lineage).

---

## Regulatory Coverage

Memintel's governance model maps to the requirements of several major regulatory regimes. See [Regulatory Readiness](/docs/admin-guide/admin-regulatory-readiness) for a full breakdown covering:

- **FDA 21 CFR Part 11** — electronic records and audit trails for clinical and life sciences
- **SOC 2 Type II** — change management, availability, and processing integrity
- **GDPR** — data subject rights, pseudonymisation, and the right to erasure
- **Basel III / DORA** — model risk management and operational resilience for financial services

---

## Pages in This Section

- [Decision Traceability](/docs/admin-guide/admin-decision-traceability)
- [Version History and Immutability](/docs/admin-guide/admin-version-history)
- [Audit Trail](/docs/admin-guide/admin-audit-trail)
- [Data Lineage](/docs/admin-guide/admin-data-lineage)
- [Regulatory Readiness](/docs/admin-guide/admin-regulatory-readiness)
