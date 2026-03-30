---
id: admin-calibration
title: Calibration
sidebar_label: Calibration
---

# Calibration

Calibration is how Memintel improves over time. When a condition fires at the wrong threshold — too sensitive, not sensitive enough — you record that as feedback, generate a calibrated recommendation, and bind it to a new condition version. The old condition is never touched. Every calibration creates a permanent, auditable record.

This page explains when and why to calibrate, how the process works end-to-end, and what calibration looks like across different domains.

---

## Why Calibration Exists

When you first define a condition, the threshold is an informed estimate. It is based on domain knowledge and the parameter priors in your guardrails file — but it has not been tested against real outcomes.

Over time, the system generates decisions. Some of those decisions will be wrong:

- A deal flagged as at-risk that closed successfully
- A patient adverse event that triggered an alert when the clinical team already knew about it
- A credit covenant that fired two months before the actual stress event

Each of these is information. Calibration turns that information into a better condition.

---

## The Three Feedback Types

Every piece of feedback falls into one of three categories:

| Type | Meaning | What it tells the system |
|---|---|---|
| `correct` | The condition fired and the outcome confirmed it was right | Threshold is in the right range |
| `false_positive` | The condition fired but the outcome showed it was wrong | Threshold is too sensitive — needs to be tighter |
| `false_negative` | The condition did not fire but the outcome showed it should have | Threshold is not sensitive enough — needs to be looser |

---

## How Calibration Works — End to End

Calibration is a four-step process. Each step is a separate API call.

### Step 1 — Submit Feedback

When you observe a wrong outcome, record it against the condition that produced it:

```bash
curl -X POST https://api.memsdl.ai/v1/feedback/decision \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "condition_id": "cond_churn_risk",
    "condition_version": "v1",
    "entity": "account_abc123",
    "feedback": "false_positive"
  }'
```

**Required fields:**

| Field | Description |
|---|---|
| `condition_id` | The condition that produced the decision |
| `condition_version` | The specific version of the condition |
| `entity` | The entity the decision was made about |
| `feedback` | `"correct"`, `"false_positive"`, or `"false_negative"` |

You can submit multiple feedback records against the same condition over time. The more feedback you accumulate, the more reliable the calibration recommendation will be.

### Step 2 — Request a Calibration Recommendation

When you have enough feedback to act on, call the calibrate endpoint:

```bash
curl -X POST https://api.memsdl.ai/v1/conditions/calibrate \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "condition_id": "cond_churn_risk",
    "condition_version": "v1"
  }'
```

Both `condition_id` and `condition_version` are required — calibration is always against a specific version.

The system analyses the accumulated feedback and returns a recommendation:

```json
{
  "status": "recommendation_ready",
  "current_params": { "threshold": 0.35, "direction": "below" },
  "recommended_params": { "threshold": 0.28, "direction": "below" },
  "calibration_token": "cal_tok_def456",
  "impact": {
    "false_positive_rate_before": 0.18,
    "false_positive_rate_after": 0.09,
    "feedback_count": 5
  }
}
```

The `calibration_token` is a signed reference to this specific recommendation. It expires and cannot be reused once applied.

### Step 3 — Review the Recommendation

Before applying, review the recommendation. Check that:

- The direction makes sense (tightening vs loosening)
- The magnitude is reasonable for your domain
- The impact metrics (false positive rate change, feedback count) justify the change

You are not obligated to apply every recommendation. If the suggested change looks wrong — for example, if the false positives were caused by a data quality issue rather than a threshold problem — discard the token and fix the underlying cause first.

### Step 4 — Apply the Calibration

If the recommendation looks right, apply it by providing both the calibration token and the new version identifier:

```bash
curl -X POST https://api.memsdl.ai/v1/conditions/apply-calibration \
  -H "X-Elevated-Key: your-elevated-key" \
  -H "Content-Type: application/json" \
  -d '{
    "token": "cal_tok_def456",
    "new_version": "v2"
  }'
```

This creates a new version of the condition with the calibrated parameters. The existing condition version is unchanged and remains in the audit log. Any tasks currently bound to the old version continue using it until explicitly rebound.

:::note Apply-calibration requires the elevated key
`POST /conditions/apply-calibration` is a privileged operation — it requires the `X-Elevated-Key` header.
:::

### Step 5 — Rebind Tasks to the New Version

After calibration, tasks still point to the old condition version. To update them, explicitly rebind each task to the new version:

```bash
curl -X PATCH https://api.memsdl.ai/v1/tasks/task_ghi012 \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "condition_version": "v2"
  }'
```

This is intentional — rebinding is an explicit action that you control. It means a calibration change never silently affects running tasks.

---

## The Immutability Guarantee

Calibration never mutates an existing condition. Every calibration creates a new version:

```
cond_churn_risk v1  →  threshold: 0.35  (original)
cond_churn_risk v2  →  threshold: 0.28  (calibrated)
```

Both versions remain in the system. You can query either version, see which tasks are bound to each, and understand exactly when and why each version was created.

This means:
- **Full audit trail** — every threshold change has a timestamp, a calibration token, and the feedback that drove it
- **Safe rollback** — if a calibrated version produces worse results, rebind tasks back to v1
- **No surprises** — a calibration never changes the behaviour of a running task without your explicit rebind

---

## When to Calibrate

Calibration is most valuable when you see a consistent pattern — not a single outlier. A single false positive might be noise. Five false positives in the same direction over 30 days is a signal.

**Signs it is time to calibrate:**

- Your team is regularly dismissing alerts as not actionable
- Outcomes that should have triggered alerts are being caught late or missed entirely
- After a significant change in the underlying domain (a product launch, a market shift, a regulatory update)
- After deploying to a new customer segment or geography with different baseline behaviour

**Signs to investigate the data first, not calibrate:**

- False positives that trace back to a missing or incorrect data value
- A sudden spike in alerts that coincides with a data pipeline failure
- Conflicting feedback — some users marking the same condition as correct, others as false positive

---

## Domain Examples

### SaaS Churn Detection

**Scenario:** The `account.active_user_rate_30d` condition is set to fire at 0.35. Over the past quarter, the customer success team has flagged 8 decisions as false positives — accounts that received churn-risk alerts but renewed without any intervention.

**Action:**
```bash
# Submit feedback for each false positive
curl -X POST .../v1/feedback/decision -d '{
  "condition_id": "cond_churn_risk",
  "condition_version": "v1",
  "entity": "account_enterprise_001",
  "feedback": "false_positive"
}'
# ... repeat for each false positive

# Request calibration
curl -X POST .../v1/conditions/calibrate \
  -H "X-API-Key: your-api-key" \
  -d '{"condition_id": "cond_churn_risk", "condition_version": "v1"}'
```

The recommendation comes back suggesting 0.28. Review: enterprise accounts tend to have lower login rates due to SSO and API usage. Apply and rebind.

---

### Credit Risk Monitoring

**Scenario:** A `borrower.dscr` condition fires when DSCR drops below 1.20. The credit team has identified 3 false negatives — borrowers who were flagged as healthy but subsequently breached covenant.

**Action:** Submit `false_negative` feedback for each. The calibration recommendation raises the threshold to 1.22. The credit committee reviews and agrees — the extra headroom is appropriate given current market volatility.

---

### Clinical Trial Safety

**Scenario:** A patient adverse event severity condition is generating alerts that the medical monitor consistently classifies as expected toxicity rather than drug-related signals — 12 `false_positive` records over 6 weeks.

**Action:** Submit feedback, request calibration. Review the recommendation carefully before applying — in a clinical context, loosening a safety threshold requires sign-off from the medical monitor and should be documented in the trial master file alongside the `calibration_token` for regulatory inspection readiness.

---

### DevOps / SRE

**Scenario:** An error rate condition fires at 0.5% but the SRE team has reconfigured the load balancer, changing the baseline traffic pattern. The last 2 weeks of feedback are almost entirely `false_positive`.

**Action:** Submit feedback, request calibration. Apply and rebind. The calibration token provides the audit record of why the threshold shifted.

---

## Common Mistakes

**Calibrating after a single outlier.** One false positive is not a pattern. Accumulate at least 3–5 consistent feedback records before requesting a calibration recommendation.

**Applying without reviewing.** The calibration recommendation is a suggestion, not an instruction. Always review the `recommended_params` and `impact` before applying — especially for safety-critical conditions.

**Forgetting to rebind tasks.** Applying a calibration creates a new condition version but does not automatically update running tasks. If you skip the rebind step, tasks continue using the old threshold.

**Calibrating instead of fixing data quality.** If false positives are caused by a faulty data source, calibrating the condition papers over the real problem. Investigate data quality first.

**Omitting `condition_version` from the calibrate request.** Both `condition_id` and `condition_version` are required. Calibration is always against a specific, immutable version — not "the latest."
