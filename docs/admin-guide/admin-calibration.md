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

You submit feedback against a specific decision — identified by its `decision_id` — using `POST /feedback`.

---

## How Calibration Works — End to End

Calibration is a four-step process. Each step is a separate API call.

### Step 1 — Submit Feedback

When you observe a wrong outcome, record it:

```bash
curl -X POST https://api.memsdl.ai/v1/feedback \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "decision_id": "dec_abc123",
    "feedback_type": "false_positive",
    "note": "Deal closed successfully — threshold too aggressive for enterprise accounts"
  }'
```

You can submit multiple feedback records against the same condition over time. The more feedback you accumulate, the more reliable the calibration recommendation will be.

### Step 2 — Request a Calibration Recommendation

When you have enough feedback to act on, call the calibrate endpoint for the condition:

```bash
curl -X POST https://api.memsdl.ai/v1/conditions/calibrate \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "condition_id": "cond_xyz789"
  }'
```

The system analyses the accumulated feedback and returns a calibration recommendation — a suggested new parameter value and a `calibration_token`:

```json
{
  "condition_id": "cond_xyz789",
  "current_value": 0.35,
  "recommended_value": 0.28,
  "calibration_token": "cal_tok_def456",
  "reasoning": "5 false positives in last 30 days — threshold adjusted downward to reduce sensitivity"
}
```

The `calibration_token` is a signed reference to this specific recommendation. It expires and cannot be reused once applied.

### Step 3 — Review the Recommendation

Before applying, review the recommendation. Check that:

- The direction makes sense (tightening vs loosening)
- The magnitude is reasonable for your domain
- The reasoning matches what you observed

You are not obligated to apply every recommendation. If the suggested change looks wrong — for example, if the false positives were caused by a data quality issue rather than a threshold problem — discard the token and fix the underlying cause first.

### Step 4 — Apply the Calibration

If the recommendation looks right, apply it:

```bash
curl -X POST https://api.memsdl.ai/v1/conditions/apply-calibration \
  -H "X-Elevated-Key: your-elevated-key" \
  -H "Content-Type: application/json" \
  -d '{
    "calibration_token": "cal_tok_def456"
  }'
```

This creates a new version of the condition with the calibrated parameter. The existing condition version is unchanged and remains in the audit log. Any tasks currently bound to the old version continue using it until explicitly rebound.

:::note Apply-calibration requires the elevated key
`POST /conditions/apply-calibration` is a privileged operation — it modifies shared registry state. It requires the `X-Elevated-Key` header, the same key used for compile and register endpoints.
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
cond_xyz789 v1  →  threshold: 0.35  (original)
cond_xyz789 v2  →  threshold: 0.28  (calibrated)
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

**Scenario:** The `account.active_user_rate_30d` condition is set to fire at 0.35 (35% active users). Over the past quarter, the customer success team has flagged 8 decisions as false positives — accounts that received churn-risk alerts but renewed without any intervention.

**Action:** Submit `false_positive` feedback for each of the 8 decisions. Call `POST /conditions/calibrate` for the condition. The recommendation comes back suggesting 0.28. Review: the team agrees enterprise accounts tend to have lower login rates due to SSO and API usage that doesn't register in the activity pipeline. Apply the calibration and rebind affected tasks.

---

### Credit Risk Monitoring

**Scenario:** A `borrower.dscr` condition fires when DSCR drops below 1.20. The credit team has identified 3 false negatives — borrowers who were flagged as healthy but subsequently breached covenant at 1.15.

**Action:** Submit `false_negative` feedback for each. The calibration recommendation raises the threshold to 1.22. Before applying, the credit committee reviews and agrees — the extra headroom is appropriate given current market volatility. Apply and rebind.

---

### Clinical Trial Safety

**Scenario:** A patient adverse event severity condition is generating alerts that the medical monitor consistently classifies as expected toxicity rather than drug-related signals. The feedback pattern is 12 `false_positive` records over 6 weeks.

**Action:** Submit feedback, request calibration. Review the recommendation carefully before applying — in a clinical context, loosening a safety threshold requires sign-off from the medical monitor and should be documented in the trial master file alongside the calibration token for regulatory inspection readiness.

---

### DevOps / SRE

**Scenario:** An error rate condition fires at 0.5% but the SRE team has reconfigured the load balancer, changing the baseline traffic pattern. What was previously a signal of real degradation is now normal operating noise. The last 2 weeks of feedback are almost entirely `false_positive`.

**Action:** Submit feedback, request calibration. The recommendation adjusts the threshold upward. Apply and rebind. Note the infrastructure change in the calibration note field so the audit log captures the reason for the shift.

---

## Common Mistakes

**Calibrating after a single outlier.** One false positive is not a pattern. Accumulate at least 3–5 consistent feedback records before requesting a calibration recommendation. A single data point produces an unreliable suggestion.

**Applying without reviewing.** The calibration recommendation is a suggestion, not an instruction. Always review the direction and magnitude before applying — especially for safety-critical conditions.

**Forgetting to rebind tasks.** Applying a calibration creates a new condition version but does not automatically update running tasks. If you skip the rebind step, tasks continue using the old threshold and nothing changes in practice.

**Calibrating instead of fixing data quality.** If false positives are caused by a faulty data source, calibrating the condition papers over the real problem. Investigate data quality first.

**Not noting the reason.** The `note` field in `POST /feedback` and the `change_note` field in `POST /conditions/apply-calibration` are your audit trail. Fill them in — especially for regulated domains where you may need to explain threshold changes to auditors or regulators.
