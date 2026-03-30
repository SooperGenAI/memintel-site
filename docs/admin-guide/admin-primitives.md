---
id: admin-primitives
title: Step 1 — Primitives
sidebar_label: Step 1 — Primitives
---

# Step 1 — Primitives

A primitive is a single signal that you want Memintel to be able to monitor. Think of primitives as the **vocabulary of measurable things** in your domain — the building blocks that all monitoring tasks are made from.

Before you can create a monitoring task for "deal engagement" or "patient adverse event severity", those concepts need to be broken down into their underlying measurable signals. Those signals are your primitives.

:::note The key distinction
**Primitives are raw, observable facts. They are not interpretations.**

- ✓ "Days since last email reply" — a primitive (directly measurable)
- ✗ "Deal health" — not a primitive (an interpretation of multiple signals)
- ✓ "Transaction amount divided by customer 90-day average" — a primitive (computable)
- ✗ "Transaction risk" — not a primitive (a concept derived from multiple signals)

The compiler derives concepts from primitives. Your job is to define the primitives.
:::

---

## How Primitives Are Registered

Primitives are registered via the API using `POST /registry/definitions`. This is the only registration step required — there is no primitives section in `memintel_config.yaml`.

```bash
curl -X POST https://api.memsdl.ai/v1/registry/definitions \
  -H "X-Elevated-Key: your-elevated-key" \
  -H "Content-Type: application/json" \
  -d '{
    "primitive_id": "account.active_user_rate_30d",
    "type": "float",
    "namespace": "org",
    "missing_data_policy": "null"
  }'
```

:::note Elevated key required
`POST /registry/definitions` is a privileged operation — it requires the `X-Elevated-Key` header.
:::

Once registered, the primitive is available for use in monitoring tasks. The compiler validates that any concept referencing a primitive uses a compatible type and strategy. Tasks cannot be created against unregistered primitives.

The data engineer separately configures how each primitive's value is fetched — which database, which query, which REST endpoint — via the `connectors:` section of `memintel_config.yaml`. Both must be in place before a monitoring task can execute.

---

## The Four Registration Fields

### primitive_id — the signal's name

A unique identifier for this signal. Use the format `entity.signal_name` — lowercase, with a dot separating the entity type from the signal name, and underscores between words.

```
account.active_user_rate_30d
customer.days_since_last_login
borrower.debt_service_coverage_ratio
patient.adverse_event_severity_score
service.error_rate_5m
```

:::tip Naming convention
The part before the dot is the **entity type** — the thing being measured (account, customer, patient, deal).
The part after the dot is the **signal name** — what is being measured, often including a time window.
This makes the registry easy to browse as it grows.
:::

### type — what kind of data it contains

The type tells the system what kind of values this signal produces and which evaluation strategies are available for it.

| Type | What it means | Supported strategies |
|---|---|---|
| `float` | A decimal number, ratio, or score | threshold, percentile, z_score, change |
| `int` | A whole number | threshold, percentile, change |
| `categorical` | One value from a fixed set of labels | equals |
| `time_series<float>` | A sequence of decimal values over time | z_score, change, percentile |
| `time_series<int>` | A sequence of whole numbers over time | z_score, change, percentile |

:::warning Boolean primitives
`boolean` is not a usable primitive type in the current implementation — no condition strategy can evaluate a boolean primitive directly. Use one of these alternatives instead:

- **`categorical` with `labels: ["true", "false"]`** — use the `equals` strategy to check for a specific value
- **`int` (0 for false, 1 for true)** — use the `threshold` strategy

Example: register `account.payment_failed_flag` as `categorical` with labels `["true", "false"]`, not as `boolean`.
:::

**When to use `time_series` vs a plain number:**

Use a time series when you want the system to detect **trends and trajectories** — not just the current value. For example:

- `borrower.dscr` (type: `float`) — the current DSCR value right now
- `borrower.dscr_trend_4q` (type: `time_series<float>`) — the DSCR across the last 4 quarters, enabling detection of a declining trend

If a user might say "alert me when X is declining" or "alert me when X is trending upward", register a time series variant alongside the scalar.

### namespace — which organisation owns this primitive

The namespace scopes the primitive to your organisation. For most deployments this is set once and reused across all primitives.

```json
"namespace": "org"
```

### missing_data_policy — what happens when there is no value

Defines how the system behaves when a primitive cannot return a value for a given entity at a given time.

| Policy | Behaviour |
|---|---|
| `"null"` | Return null — the concept evaluation receives a null input |
| `"zero"` | Return zero — treat missing data as the zero value for this type |
| `"error"` | Raise an evaluation error — the decision record shows a data error |

Use `"null"` for signals that are legitimately absent (a customer with no calls, a borrower with no commentary). Use `"error"` for signals that should always be present and whose absence indicates a data pipeline problem.

---

## Optional: Labels for Categorical Primitives

For `categorical` primitives, specify the set of valid values using the `labels` field:

```bash
curl -X POST https://api.memsdl.ai/v1/registry/definitions \
  -H "X-Elevated-Key: your-elevated-key" \
  -H "Content-Type: application/json" \
  -d '{
    "primitive_id": "account.plan_tier",
    "type": "categorical",
    "namespace": "org",
    "missing_data_policy": "null",
    "labels": ["starter", "growth", "enterprise"]
  }'
```

Labels constrain what values the `equals` strategy can be evaluated against. A condition checking `account.plan_tier equals "premium"` would be rejected at compile time if "premium" is not in the registered label set.

For boolean-like signals, use two labels:

```json
{
  "primitive_id": "account.payment_failed_flag",
  "type": "categorical",
  "namespace": "org",
  "missing_data_policy": "null",
  "labels": ["true", "false"]
}
```

---

## Nullable Signals

If a signal sometimes has no value, append `?` to the type and set `missing_data_policy` to `"null"`:

```json
{
  "primitive_id": "deal.last_call_sentiment",
  "type": "float?",
  "namespace": "org",
  "missing_data_policy": "null"
}
```

An unexpected null on a non-nullable primitive causes an evaluation error. Always mark signals that can legitimately be absent as nullable.

---

## One Signal Per Primitive

The most important design rule: **each primitive measures exactly one thing**.

If you find yourself wanting "engagement and sentiment" in a single primitive, split them:

```bash
# Wrong — two signals in one
{ "primitive_id": "deal.engagement_and_sentiment", "type": "float" }

# Right — two separate primitives
{ "primitive_id": "deal.engagement_score", "type": "float" }
{ "primitive_id": "deal.sentiment_score", "type": "float" }
```

The system combines primitives into concepts automatically. Your job is to provide the raw signals, not pre-combine them.

---

## Internal vs External Signals

One of Memintel's key capabilities is evaluating **your internal data against external signals** — regulatory changes, market data, peer benchmarks. Both types are registered as primitives in exactly the same way.

```bash
# Internal signal — your own data
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "filing.deprecated_tag_count", "type": "int", "namespace": "org", "missing_data_policy": "zero"}'

# External signal — regulatory feed (boolean-like → categorical)
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "taxonomy.tag_deprecated_flag", "type": "categorical", "namespace": "org", "missing_data_policy": "null", "labels": ["true", "false"]}'

# External signal — peer benchmark
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "provider.peer_deviation_percentile", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
```

Your data engineer connects each primitive to its data source via `memintel_config.yaml`. The primitive registry records the type and policy; the connector configuration records where to fetch the value.

---

## Domain Examples

### SaaS Churn Detection

```bash
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "user.days_since_last_login", "type": "int", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "user.core_actions_30d", "type": "int", "namespace": "org", "missing_data_policy": "zero"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "user.session_frequency_trend_8w", "type": "time_series<float>", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "account.active_user_rate_30d", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "account.seat_utilization_rate", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "account.days_to_renewal", "type": "int", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "account.payment_failed_flag", "type": "categorical", "namespace": "org", "missing_data_policy": "null", "labels": ["true", "false"]}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "account.nps_score", "type": "float?", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "account.support_ticket_rate_30d", "type": "float", "namespace": "org", "missing_data_policy": "zero"}'
```

### Credit Risk Monitoring

```bash
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "borrower.dscr", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "borrower.dscr_trend_4q", "type": "time_series<float>", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "borrower.leverage_ratio", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "borrower.management_sentiment_score", "type": "float?", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "loan.covenant_headroom_pct", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "loan.days_since_financial_submission", "type": "int", "namespace": "org", "missing_data_policy": "null"}'
```

### Clinical Trial Safety

```bash
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "patient.ae_severity_score", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "patient.ae_relatedness_signal", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "patient.ae_relatedness_confidence", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "patient.sae_count_30d", "type": "int", "namespace": "org", "missing_data_policy": "zero"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "trial.treatment_vs_comparator_ratio", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "trial.stopping_rule_proximity_score", "type": "float", "namespace": "org", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "compound.fda_class_safety_alert_flag", "type": "categorical", "namespace": "org", "missing_data_policy": "null", "labels": ["true", "false"]}'
```

---

## Working with Your Data Engineer

For each primitive, the workflow is:

1. **You define the signal** — agree on the name, what it measures, its type, and its value range.
2. **Register the type** — call `POST /registry/definitions` with the correct type, namespace, and missing data policy.
3. **Data engineer connects the data source** — adds the connector mapping to `memintel_config.yaml` so the runtime knows how to fetch the value at execution time. This requires a server restart.
4. **You verify** — create a test monitoring task using the primitive and confirm results make sense.

Steps 2 and 3 can happen in either order, but both must be complete before a monitoring task can execute successfully.

---

## Common Mistakes

**Adding a `primitives:` section to `memintel_config.yaml`.** This section does not exist in the current implementation and will be silently ignored or cause a validation error. Primitives are registered exclusively via `POST /registry/definitions`.

**Using `boolean` as the primitive type.** No condition strategy can evaluate a boolean primitive. Use `categorical` with `labels: ["true", "false"]` and the `equals` strategy instead.

**Using wrong field names in the API.** The endpoint accepts `primitive_id`, `type`, `namespace`, `missing_data_policy`, and `labels`. Fields like `id`, `source`, `entity`, and `description` do not exist in the current model.

**Registering a primitive but forgetting the connector mapping.** The registry records the type; the connector configuration in `memintel_config.yaml` records where to fetch the value. Without a matching connector mapping, task execution will fail with a data resolution error.

**Defining concepts as primitives.** "Deal health score", "customer risk level", "account engagement" — these are concepts the compiler derives from primitive signals. Register the underlying signals instead.

**Forgetting to mark nullable signals.** If a signal sometimes has no value, declare it as `type?` and set `missing_data_policy: "null"`. An unexpected null on a non-nullable primitive causes evaluation errors.

**Missing time-series variants.** If you want to detect trends — "declining over the last 4 quarters", "increasing over the last 8 weeks" — register a `time_series<float>` or `time_series<int>` variant. The plain `float` version only tells you the current value.
