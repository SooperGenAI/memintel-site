---
id: admin-audit-trail
title: Audit Trail
sidebar_label: Audit Trail
---

# Audit Trail

Memintel maintains an append-only audit log of every system event — decisions, configuration changes, calibrations, task rebinds, and guardrails updates. No event is ever deleted or modified. The audit log is the authoritative record of everything that has happened in the system.

---

## What the Audit Log Records

The audit log captures two categories of events:

### Decision Events
Every evaluation — triggered or not — produces an audit event:

| Event | When it occurs |
|---|---|
| `decision.triggered` | A condition evaluated and fired |
| `decision.not_triggered` | A condition evaluated and did not fire |
| `decision.error` | Evaluation failed (data missing, type mismatch, etc.) |

### System Change Events
Every configuration change produces an audit event:

| Event | When it occurs |
|---|---|
| `condition.registered` | A new condition was registered |
| `condition.calibrated` | A calibration recommendation was generated |
| `condition.calibration_applied` | A calibration was applied, creating a new version |
| `guardrails.updated` | Guardrails were updated via API |
| `guardrails.loaded` | Guardrails were loaded from file on startup |
| `context.updated` | Application context was updated |
| `task.created` | A new task was created |
| `task.rebound` | A task was rebound to a new condition version |
| `task.deleted` | A task was deleted |
| `feedback.submitted` | Feedback was submitted against a decision |

---

## Querying the Audit Log

```bash
# All events in a time range
curl "https://api.memsdl.ai/v1/audit?from=2025-11-01&to=2025-11-30" \
  -H "X-API-Key: your-api-key"

# All events for a specific condition
curl "https://api.memsdl.ai/v1/audit?resource_id=cond_xyz456" \
  -H "X-API-Key: your-api-key"

# All system change events (no decision events)
curl "https://api.memsdl.ai/v1/audit?category=system_change" \
  -H "X-API-Key: your-api-key"

# All events by actor
curl "https://api.memsdl.ai/v1/audit?actor=admin" \
  -H "X-API-Key: your-api-key"
```

---

## Audit Event Structure

Each audit event contains:

```json
{
  "audit_id": "aud_mno654",
  "event_type": "condition.calibration_applied",
  "occurred_at": "2025-11-14T14:32:00Z",
  "actor": "admin",
  "resource_type": "condition",
  "resource_id": "cond_xyz456",
  "detail": {
    "from_version": "v1",
    "to_version": "v2",
    "calibration_token": "cal_tok_def456",
    "change_note": "Reduced threshold — enterprise accounts have lower login rates due to SSO",
    "threshold_before": 0.35,
    "threshold_after": 0.28
  }
}
```

---

## The Calibration Audit Chain

Calibration produces a chain of linked audit events that together tell the full story of a threshold change:

```
feedback.submitted  →  condition.calibrated  →  condition.calibration_applied  →  task.rebound
```

You can retrieve the full chain for any calibration token:

```bash
curl https://api.memsdl.ai/v1/audit/chain/cal_tok_def456 \
  -H "X-API-Key: your-api-key"
```

```json
{
  "calibration_token": "cal_tok_def456",
  "chain": [
    {
      "event_type": "feedback.submitted",
      "occurred_at": "2025-11-01T10:00:00Z",
      "detail": { "decision_id": "dec_001", "feedback_type": "false_positive" }
    },
    {
      "event_type": "feedback.submitted",
      "occurred_at": "2025-11-07T14:22:00Z",
      "detail": { "decision_id": "dec_007", "feedback_type": "false_positive" }
    },
    {
      "event_type": "condition.calibrated",
      "occurred_at": "2025-11-14T14:00:00Z",
      "detail": { "recommended_value": 0.28, "feedback_count": 5 }
    },
    {
      "event_type": "condition.calibration_applied",
      "occurred_at": "2025-11-14T14:32:00Z",
      "detail": { "from_version": "v1", "to_version": "v2", "threshold_after": 0.28 }
    },
    {
      "event_type": "task.rebound",
      "occurred_at": "2025-11-14T14:45:00Z",
      "detail": { "task_id": "task_ghi789", "from_version": "v1", "to_version": "v2" }
    }
  ]
}
```

This chain is the complete audit record of the calibration — from the first feedback event to the task rebind.

---

## Exporting Audit Logs

For compliance reporting, you can export the full audit log for a time period:

```bash
curl "https://api.memsdl.ai/v1/audit/export?from=2025-01-01&to=2025-12-31&format=json" \
  -H "X-API-Key: your-api-key" \
  -o audit_2025.json
```

Supported formats: `json`, `csv`.

---

## Domain Examples

### FDA 21 CFR Part 11 — Audit Trail for Clinical Systems

FDA regulations require that computer systems used in clinical trials maintain audit trails that capture the date and time of operator entries and changes to electronic records — including who made the change and when.

The Memintel audit log satisfies this requirement for all decision and configuration events. For a clinical trial inspection:

```bash
# Export all audit events for the trial period
curl "https://api.memsdl.ai/v1/audit/export?from=2025-01-01&to=2025-06-30&format=csv" \
  -H "X-API-Key: your-api-key" \
  -o trial_audit_log.csv
```

The export includes every decision event, every threshold change, every calibration, and every feedback submission — with timestamps and actor attribution throughout.

Key regulatory mapping:
- **§ 11.10(e)** — Use of audit trails to record the date and time of operator entries → `occurred_at` on every event
- **§ 11.10(f)** — Use of operational system checks → `decision.error` events capture evaluation failures
- **§ 11.300** — Controls for identification codes and passwords → `actor` field on every event

### SOC 2 — Processing Integrity and Change Management

SOC 2 Type II auditors require evidence that system processing is complete, accurate, and authorised — and that changes to processing logic are controlled.

The audit log provides:
- Evidence of continuous processing (`decision.triggered` and `decision.not_triggered` events throughout the audit period)
- Evidence that configuration changes were authorised and documented (`actor`, `change_note` on all system change events)
- Evidence that no unauthorised changes occurred (append-only log with no deletions or modifications)

### GDPR — Right of Access and Erasure

Under GDPR, data subjects have the right to know what automated decisions were made about them and on what basis.

The audit log supports this in two ways:

**Right of access:** Retrieve all decisions for a pseudonymised entity and provide the explanation:

```bash
curl "https://api.memsdl.ai/v1/decisions?entity_id=ent_pseudonymised_001" \
  -H "X-API-Key: your-api-key"
```

For each decision, `POST /decisions/{id}/explain` produces a plain-English explanation of why the decision was made — suitable for a GDPR subject access response.

**Right to erasure:** Memintel uses `entity_id` pseudonymisation. The mapping between real identifiers and pseudonymised `entity_id` values is maintained in a separate `entity_id_map` table. To comply with an erasure request, delete the entry from `entity_id_map`. The decision records in the audit log retain the pseudonymised `entity_id` — the audit integrity is preserved, but the record can no longer be linked to the individual.

:::warning
Deleting from `entity_id_map` is irreversible. Before proceeding, confirm that retaining the pseudonymised audit record is consistent with your legal basis for processing and your data retention policy.
:::

### Financial Services — Operational Resilience (DORA)

Under the EU Digital Operational Resilience Act (DORA), financial entities must maintain comprehensive logs of ICT-related incidents and the behaviour of critical systems.

```bash
# Retrieve all decision errors and system change events for DORA incident reporting
curl "https://api.memsdl.ai/v1/audit?event_type=decision.error&from=2025-01-01&to=2025-12-31" \
  -H "X-API-Key: your-api-key"
```

The audit log provides the continuous evidence of system behaviour required by DORA Article 9 (ICT risk management) and Article 10 (detection of anomalous activities).

---

## Common Mistakes

**Not using the `change_note` field.** The audit log records what changed. Your change notes record why. Both are needed for a complete audit trail. Build a team practice of always filling in change notes for guardrails updates, calibrations, and task rebinds.

**Treating `decision.not_triggered` events as unimportant.** Regulators and auditors often want to see evidence that the system was monitoring continuously — not just that it fired alerts. The `not_triggered` events are that evidence.

**Exporting only triggered decisions for compliance reports.** Export the full audit log, not just triggered events. A complete log demonstrates continuous operation; a filtered log raises questions about what was omitted.
