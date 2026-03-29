---
id: admin-version-history
title: Version History and Immutability
sidebar_label: Version History
---

# Version History and Immutability

In Memintel, nothing is ever overwritten. Every change to a condition, guardrails configuration, or application context creates a new version. All previous versions remain in the system, permanently queryable.

This means you can always answer: *what was the exact rule in effect at any given point in time?*

---

## Why Immutability Matters

Most systems store the current state of a rule. When a threshold changes, the old value is gone. This creates a fundamental audit problem: if a decision was made six months ago under a different threshold, you cannot prove what that threshold was.

Memintel solves this by treating all configuration as append-only. The current state is the latest version. Every previous state is preserved:

```
cond_xyz456  v1  threshold: 0.35  created: 2025-09-01  (original)
cond_xyz456  v2  threshold: 0.28  created: 2025-11-14  (calibrated)
cond_xyz456  v3  threshold: 0.30  created: 2026-01-22  (recalibrated)
```

A decision made on 2025-10-15 was evaluated against v1. You can retrieve v1, inspect its threshold, and replay the decision — regardless of how many times the condition has been calibrated since.

---

## What Is Versioned

### Conditions

Every time a condition is calibrated, a new version is created:

```bash
# List all versions of a condition
curl https://api.memsdl.ai/v1/conditions/cond_xyz456/versions \
  -H "X-API-Key: your-api-key"
```

```json
{
  "condition_id": "cond_xyz456",
  "versions": [
    {
      "version": "v1",
      "threshold": 0.35,
      "strategy": "threshold",
      "created_at": "2025-09-01T10:00:00Z",
      "created_by": "admin",
      "source": "initial_registration"
    },
    {
      "version": "v2",
      "threshold": 0.28,
      "strategy": "threshold",
      "created_at": "2025-11-14T14:32:00Z",
      "created_by": "admin",
      "source": "calibration",
      "calibration_token": "cal_tok_def456",
      "change_note": "Reduced threshold — enterprise accounts have lower login rates due to SSO"
    }
  ]
}
```

### Guardrails

Every time guardrails are updated via `POST /guardrails`, a new version is created:

```bash
# List all guardrails versions
curl https://api.memsdl.ai/v1/guardrails/versions \
  -H "X-API-Key: your-api-key"
```

```json
{
  "versions": [
    {
      "version": 1,
      "created_at": "2025-09-01T10:00:00Z",
      "source": "file_seed",
      "change_note": "Initial deployment"
    },
    {
      "version": 2,
      "created_at": "2025-10-15T09:15:00Z",
      "source": "api",
      "change_note": "Added composite strategy to type registry"
    }
  ]
}
```

### Application Context

Every time application context is updated via `POST /context`, a new version is created:

```bash
# List all context versions
curl https://api.memsdl.ai/v1/context/versions \
  -H "X-API-Key: your-api-key"
```

---

## Retrieving a Specific Version

You can retrieve any historical version directly:

```bash
# Retrieve condition v1
curl https://api.memsdl.ai/v1/conditions/cond_xyz456?version=v1 \
  -H "X-API-Key: your-api-key"

# Retrieve guardrails version 1
curl https://api.memsdl.ai/v1/guardrails?version=1 \
  -H "X-API-Key: your-api-key"

# Retrieve context version 2
curl https://api.memsdl.ai/v1/context?version=2 \
  -H "X-API-Key: your-api-key"
```

---

## Comparing Versions

To understand what changed between two versions of a condition:

```bash
curl "https://api.memsdl.ai/v1/conditions/cond_xyz456/diff?from=v1&to=v2" \
  -H "X-API-Key: your-api-key"
```

```json
{
  "condition_id": "cond_xyz456",
  "from_version": "v1",
  "to_version": "v2",
  "changes": [
    {
      "field": "threshold",
      "from": 0.35,
      "to": 0.28,
      "changed_at": "2025-11-14T14:32:00Z",
      "change_note": "Reduced threshold — enterprise accounts have lower login rates due to SSO"
    }
  ]
}
```

---

## Immutability Attestation

For regulated environments, you may need to produce a formal attestation that a specific condition version was unchanged during a given period.

```bash
curl -X POST https://api.memsdl.ai/v1/conditions/cond_xyz456/attest \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "version": "v1",
    "from": "2025-09-01",
    "to": "2025-11-13"
  }'
```

```json
{
  "condition_id": "cond_xyz456",
  "version": "v1",
  "attested_period": {
    "from": "2025-09-01",
    "to": "2025-11-13"
  },
  "ir_hash": "sha256:7f3a9c...",
  "attestation": "This condition version was active and unchanged during the attested period. No calibrations, modifications, or replacements occurred.",
  "verified": true,
  "attestation_id": "att_jkl321"
}
```

The `attestation_id` can be stored in your compliance documentation as a reference to the machine-verifiable proof.

---

## Task Version Pinning

Tasks in Memintel are version-pinned to specific condition versions at the time of creation. This means a task always evaluates against the condition version it was bound to — not the latest version.

```json
{
  "task_id": "task_ghi789",
  "task_version": "v2",
  "condition_id": "cond_xyz456",
  "condition_version": "v1",
  "pinned_at": "2025-09-15T11:00:00Z"
}
```

When a condition is calibrated, running tasks are not automatically updated. You explicitly rebind them when you are ready. This means:

- A calibration never silently changes the behaviour of a running task
- You can run multiple tasks against different versions of the same condition simultaneously
- The version a decision was evaluated against is always deterministic and auditable

---

## Domain Examples

### FDA 21 CFR Part 11 — Electronic Records Integrity

FDA regulations require that electronic records used in clinical decisions are accurate, reliable, and protected from unauthorised modification.

Memintel's immutable versioning directly satisfies this requirement:

- Condition versions, once created, cannot be modified — only superseded by new versions
- The `ir_hash` provides a machine-verifiable proof of the exact logic applied to any decision
- Version history with timestamps and actor attribution creates an audit trail that meets 21 CFR Part 11 § 11.10(e) requirements

Retrieve the full version history for a clinical trial monitoring condition as part of inspection documentation:

```bash
curl https://api.memsdl.ai/v1/conditions/cond_safety_monitor/versions \
  -H "X-API-Key: your-api-key"
```

### Model Risk Management (SR 11-7 / Basel)

Financial regulators require that model changes are documented, reviewed, and approved before deployment — and that historical model versions are preserved for back-testing and validation.

Memintel's version history provides:
- A full record of every threshold change with timestamp, actor, and change note
- The ability to back-test any historical version against current data
- Diff views to support model change documentation requirements

### SOC 2 Type II — Change Management

SOC 2 change management controls require that changes to systems that affect data processing are authorised, documented, and tested.

Use the version history and diff APIs to produce change documentation for SOC 2 auditors:

```bash
# Export full version history for a condition as evidence
curl https://api.memsdl.ai/v1/conditions/cond_xyz456/versions \
  -H "X-API-Key: your-api-key"
```

Each version record includes the actor, timestamp, source (calibration vs manual update), and change note — providing the documentation trail required by CC8.1 (change management) controls.

---

## Common Mistakes

**Not including change notes.** The `change_note` field on calibrations and guardrails updates is optional — but in practice it is essential. An audit trail without notes shows *what* changed but not *why*. Always fill in the change note.

**Assuming the latest version is the only version.** When querying conditions or guardrails without specifying a version, you get the latest. For audit and compliance purposes, always retrieve and store the specific version referenced by each decision.

**Rebinding tasks without documenting the reason.** When you rebind a task to a new condition version, record the reason in your team's change log. The system records the rebind event, but the business reason should live in your documentation alongside the `attestation_id`.
