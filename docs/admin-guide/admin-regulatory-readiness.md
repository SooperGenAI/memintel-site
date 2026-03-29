---
id: admin-regulatory-readiness
title: Regulatory Readiness
sidebar_label: Regulatory Readiness
---

# Regulatory Readiness

Memintel's governance architecture — immutable versioning, append-only audit logs, full decision traceability, and deterministic replay — maps directly to the requirements of major regulatory frameworks. This page covers what each framework requires and how Memintel satisfies it.

---

## FDA 21 CFR Part 11 — Electronic Records and Audit Trails

**Applicable to:** Clinical trials, pharmaceutical manufacturing, medical devices, life sciences.

21 CFR Part 11 governs electronic records and electronic signatures used in FDA-regulated activities. It requires that systems maintain accurate, reliable records with audit trails that capture who did what and when.

### Key Requirements and How Memintel Satisfies Them

| Requirement | Regulation | How Memintel satisfies it |
|---|---|---|
| Audit trails must record date/time of entries and changes | § 11.10(e) | Every audit event has an `occurred_at` ISO 8601 timestamp |
| Audit trails must record the identity of the operator | § 11.10(e) | Every audit event has an `actor` field |
| Audit trails must be computer-generated and not modifiable | § 11.10(e) | Audit log is append-only — no event can be modified or deleted |
| System must protect records from modification | § 11.10(c) | Conditions, guardrails, and decisions are immutable once written |
| System must generate accurate copies of records | § 11.10(b) | Decision records and version history are fully exportable |
| Audit trail must be available for FDA inspection | § 11.10(e) | Full audit log export via API in JSON or CSV |

### Practical Guidance

**For clinical trial monitoring:**

Retrieve and export the full audit trail for the trial period as part of trial master file documentation:

```bash
curl "https://api.memsdl.ai/v1/audit/export?from=2025-01-01&to=2025-06-30&format=csv" \
  -H "X-API-Key: your-api-key" \
  -o trial_audit_log_2025.csv
```

For each safety signal decision, use the `ir_hash` and immutability attestation to demonstrate that the evaluation logic was unchanged throughout the trial:

```bash
curl -X POST https://api.memsdl.ai/v1/conditions/cond_safety_monitor/attest \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "version": "v1", "from": "2025-01-01", "to": "2025-06-30" }'
```

Store the returned `attestation_id` in the trial master file as a reference to the machine-verifiable proof.

**For DSMB review packets:**

Include the full decision lineage for each safety signal — showing the exact adverse event severity score, relatedness probability, and threshold that produced each alert:

```bash
curl https://api.memsdl.ai/v1/decisions/dec_safety_001/lineage \
  -H "X-API-Key: your-api-key"
```

---

## SOC 2 Type II — Trust Services Criteria

**Applicable to:** SaaS platforms, B2B software, any organisation storing or processing customer data.

SOC 2 Type II requires that an organisation's controls are not just designed correctly but operate effectively over time — typically a 12-month observation period. The relevant trust service criteria for decision systems are:

### Security (CC6) — Logical Access and Change Management

| Criterion | Requirement | How Memintel satisfies it |
|---|---|---|
| CC6.1 | Restrict logical access to protect against threats | Elevated key required for privileged operations (compile, calibrate, guardrails update) |
| CC6.8 | Prevent or detect unauthorised changes | Immutable versioning — every change creates a new version, no in-place edits |
| CC8.1 | Authorise and document changes to infrastructure | Actor + timestamp + change note on every system change event |

### Availability (A1) — System Availability

| Criterion | Requirement | How Memintel satisfies it |
|---|---|---|
| A1.2 | Monitor environmental conditions | `decision.error` events in audit log capture evaluation failures |
| A1.3 | Recover from environmental failures | Version history enables rollback to known-good condition versions |

### Processing Integrity (PI1) — Complete and Accurate Processing

| Criterion | Requirement | How Memintel satisfies it |
|---|---|---|
| PI1.1 | Processing is complete, accurate, timely | Every evaluation produces a decision record — `not_triggered` events prove continuous operation |
| PI1.4 | Output is complete and accurate | `ir_hash` provides machine-verifiable proof that evaluation logic was applied correctly |

### Practical Guidance

For a SOC 2 Type II audit, provide the auditor with:

1. **Change management evidence** — export the system change audit log for the observation period, showing all condition calibrations, guardrails updates, and task rebinds with actors, timestamps, and change notes
2. **Processing integrity evidence** — export decision counts by day for the observation period, showing continuous operation
3. **Access control evidence** — demonstrate that privileged operations require the elevated key

```bash
# Change management evidence
curl "https://api.memsdl.ai/v1/audit?category=system_change&from=2025-01-01&to=2025-12-31" \
  -H "X-API-Key: your-api-key"

# Processing integrity evidence — decisions per day
curl "https://api.memsdl.ai/v1/decisions/summary?group_by=day&from=2025-01-01&to=2025-12-31" \
  -H "X-API-Key: your-api-key"
```

---

## GDPR — General Data Protection Regulation

**Applicable to:** Any organisation processing personal data of EU residents.

GDPR introduces several rights that affect automated decision systems. The most relevant are the right of access (Article 15), the right to explanation for automated decisions (Article 22), and the right to erasure (Article 17).

### Right of Access (Article 15)

Data subjects have the right to know what decisions were made about them and on what basis.

Retrieve all decisions for a data subject using their pseudonymised `entity_id`:

```bash
curl "https://api.memsdl.ai/v1/decisions?entity_id=ent_pseudonymised_001&from=2025-01-01&to=2025-12-31" \
  -H "X-API-Key: your-api-key"
```

For each decision, produce a plain-English explanation:

```bash
curl -X POST https://api.memsdl.ai/v1/decisions/dec_abc123/explain \
  -H "X-API-Key: your-api-key"
```

The explanation is grounded in the stored decision record — not re-evaluated — making it a reliable basis for a subject access response.

### Right to Explanation for Automated Decisions (Article 22)

Where automated decisions produce legal or similarly significant effects, data subjects have the right to meaningful information about the logic involved.

The decision explanation API provides exactly this — a plain-English account of the logic, the inputs, and the outcome, grounded in the immutable decision record.

### Right to Erasure (Article 17)

Memintel uses `entity_id` pseudonymisation. Real identifiers are stored in a separate `entity_id_map` table, not in the decision records themselves.

To fulfil an erasure request:

1. Look up the data subject's `entity_id` in your `entity_id_map`
2. Delete the entry from `entity_id_map`

The pseudonymised decision records in the audit log are retained — audit integrity is preserved, but the records can no longer be linked to the individual.

:::warning
This process is irreversible. Once the `entity_id_map` entry is deleted, the pseudonymised records cannot be re-linked to the individual. Ensure this is consistent with your legal basis for retaining audit records before proceeding.
:::

### Data Minimisation (Article 5)

Memintel only stores the primitive values that were actually used in a decision — not a full snapshot of the entity's data. The `input_primitives` field in the decision record contains only the signals that fed into the concept computation for that specific decision.

---

## Basel III / DORA — Financial Services

**Applicable to:** Banks, investment firms, insurers, and other regulated financial entities in the EU and UK.

### Model Risk Management (SR 11-7 / EBA Guidelines)

Supervisory Guidance on Model Risk Management (US SR 11-7 and its European equivalents) requires that models used in credit, risk, and other financial decisions are documented, validated, and controlled.

| Requirement | How Memintel satisfies it |
|---|---|
| Model documentation | Condition version history with full parameter records |
| Model change control | Immutable versioning — every calibration creates a new version with actor, timestamp, and change note |
| Model validation | Attestation API provides machine-verifiable proof of model state during any period |
| Back-testing | Historical condition versions can be applied to current data for back-testing |

```bash
# Retrieve the complete history of a credit risk condition for model documentation
curl https://api.memsdl.ai/v1/conditions/cond_credit_dscr/versions \
  -H "X-API-Key: your-api-key"

# Produce attestation for a model validation period
curl -X POST https://api.memsdl.ai/v1/conditions/cond_credit_dscr/attest \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "version": "v2", "from": "2025-01-01", "to": "2025-12-31" }'
```

### Digital Operational Resilience Act (DORA — EU 2022/2554)

DORA requires financial entities to maintain comprehensive ICT risk management frameworks, including logging and monitoring of critical systems.

Key DORA obligations and how Memintel satisfies them:

| DORA Article | Obligation | How Memintel satisfies it |
|---|---|---|
| Article 9 | ICT risk management — detect, manage, and monitor ICT risks | `decision.error` and system change events in audit log |
| Article 10 | Detection of anomalous activities | Continuous decision records with `not_triggered` events proving monitoring was active |
| Article 11 | Response and recovery | Version history enables rollback to known-good condition versions |
| Article 17 | ICT-related incident reporting | Full audit log export for incident documentation |

```bash
# Export ICT event log for DORA incident report
curl "https://api.memsdl.ai/v1/audit?from=2025-11-01&to=2025-11-30&format=json" \
  -H "X-API-Key: your-api-key" \
  -o dora_incident_log.json
```

---

## Building a Compliance Documentation Package

For any regulatory inspection or audit, the following set of exports provides a complete compliance documentation package:

```bash
# 1. Full audit log for the period
curl "https://api.memsdl.ai/v1/audit/export?from=2025-01-01&to=2025-12-31&format=csv" \
  -H "X-API-Key: your-api-key" -o audit_log.csv

# 2. Version history for all active conditions
curl https://api.memsdl.ai/v1/conditions/versions/summary \
  -H "X-API-Key: your-api-key" -o condition_versions.json

# 3. Guardrails version history
curl https://api.memsdl.ai/v1/guardrails/versions \
  -H "X-API-Key: your-api-key" -o guardrails_versions.json

# 4. Decision summary by period
curl "https://api.memsdl.ai/v1/decisions/summary?group_by=month&from=2025-01-01&to=2025-12-31" \
  -H "X-API-Key: your-api-key" -o decision_summary.json

# 5. Immutability attestations for key conditions
curl -X POST https://api.memsdl.ai/v1/conditions/cond_xyz456/attest \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{ "from": "2025-01-01", "to": "2025-12-31" }' -o attestation.json
```

Keep these exports, along with your change notes and calibration audit chains, as the core of your regulatory evidence file.

---

## Common Mistakes

**Not filling in change notes.** Every calibration and guardrails update allows a `change_note`. In a regulated environment, this note is the difference between a well-documented change and an unexplained entry in the audit log. Make change notes mandatory in your team processes.

**Assuming pseudonymisation equals anonymisation under GDPR.** Pseudonymised data is still personal data under GDPR if the `entity_id_map` exists and can re-link the records. Treat all Memintel decision data as personal data unless the `entity_id_map` entry has been deleted.

**Exporting audit logs only when asked.** For regulated environments, set up a scheduled export of the audit log to a secure, long-term storage location. Do not rely on the ability to export on demand — systems change, and a gap in the audit log is difficult to explain to a regulator.

**Not storing `attestation_id` references.** When you generate an immutability attestation, store the `attestation_id` in your compliance documentation. It is a reference to a machine-verifiable proof — if you need to produce it later, you can retrieve it by ID rather than regenerating it.
