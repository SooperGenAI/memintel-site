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

## How Primitives Work — Two Steps

Setting up a primitive requires two separate steps. Both are required. Neither replaces the other.

| Step | Where | What it does | Who does it |
|---|---|---|---|
| **1. Config file mapping** | `memintel_config.yaml` | Tells the runtime *where* to fetch the primitive's value — which database, which query, which REST endpoint | Data engineer |
| **2. Type registry** | `POST /registry/definitions` | Tells the compiler and runtime *what type* the primitive is — float, int, boolean, etc. | Data engineer |

The primitive name must match in both places. The registry owns the type. The config owns the data source.

:::warning Both steps are required
A primitive registered via the API but missing from the config file will fail at execution time — the runtime has no data source to fetch from. A primitive in the config but not registered via the API will fail at compile time — the compiler does not know its type.
:::

---

## Step 1 — Config File Mapping

The `memintel_config.yaml` file maps each primitive name to its data source. This is the data plumbing — it tells the runtime where to fetch each value at execution time.

```yaml
# memintel_config.yaml

primitives:
  account.active_user_rate_30d:
    connector: postgres.analytics
    query: >
      SELECT (active_users::float / total_seats)
      FROM account_metrics
      WHERE account_id = :entity_id
      AND recorded_at <= :as_of
      ORDER BY recorded_at DESC LIMIT 1

  account.days_to_renewal:
    connector: postgres.accounts
    query: >
      SELECT EXTRACT(DAY FROM (renewal_date - :as_of::date))::int
      FROM subscriptions
      WHERE account_id = :entity_id
      AND status = 'active' LIMIT 1

  account.payment_failed_flag:
    connector: rest.billing_api
    path: /accounts/:entity_id/payment-status
    field: payment_failed
```

Each entry maps a primitive name to a connector (defined in the `connectors:` section) and a query or endpoint. The `:entity_id` and `:as_of` parameters are injected by the runtime at execution time — `:as_of` ensures deterministic, point-in-time fetching.

Your data engineer writes and maintains these entries. You provide the primitive names and descriptions; they handle the SQL and connector wiring.

---

## Step 2 — Type Registry

Once the config file is updated and the server has restarted, register each primitive's type schema via the API:

```bash
curl -X POST https://api.memsdl.ai/v1/registry/definitions \
  -H "X-Elevated-Key: your-elevated-key" \
  -H "Content-Type: application/json" \
  -d '{
    "primitive_id": "account.active_user_rate_30d",
    "type": "float",
    "namespace": "org",
    "missing_data_policy": "null",
    "label": "Active user rate (30d)",
    "description": "Ratio of active users to total licensed seats in last 30 days, 0-1"
  }'
```

This is what the compiler uses to validate concepts and select evaluation strategies. A primitive must be registered here before any monitoring task can reference it.

:::note Elevated key required
`POST /registry/definitions` is a privileged operation — it requires the `X-Elevated-Key` header.
:::

---

## The Primitive Name

The primitive name is the single identifier that must match across both the config file and the registry. Use the format `entity.signal_name` — lowercase, dot-separated, underscores between words.

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

---

## Types

The type tells the system what kind of values this primitive produces and which evaluation strategies are available for it.

| Type | What it means | Example signals |
|---|---|---|
| `float` | A decimal number, usually between 0 and 1 or a ratio | Sentiment score, engagement rate, LTV ratio |
| `int` | A whole number | Days since login, count of events, number of calls |
| `boolean` | True or false only | Payment failed flag, OIG exclusion flag, license active |
| `categorical` | One value from a fixed list | Risk tier (low/medium/high), status (active/paused/closed) |
| `time_series<float>` | A sequence of decimal values over time | Error rate over last hour, DSCR over last 4 quarters |
| `time_series<int>` | A sequence of whole numbers over time | Daily transaction count over last 30 days |

**When to use `time_series` vs a plain number:**

Use a time series when you want the system to be able to detect **trends and trajectories** — not just the current value. For example:

- `borrower.dscr` (type: `float`) — the current DSCR value right now
- `borrower.dscr_trend_4q` (type: `time_series<float>`) — the DSCR across the last 4 quarters, enabling detection of a declining trend

If a user might say "alert me when X is declining" or "alert me when X is trending upward", register a time series variant.

**Nullable signals:**

If a signal sometimes has no value — for example, a sentiment score for a customer who has never sent an email — use a nullable type:

```json
{
  "primitive_id": "deal.last_call_sentiment",
  "type": "float?",
  "missing_data_policy": "null",
  "description": "Sentiment score from the last call recording — null if no calls"
}
```

---

## One Signal Per Primitive

The most important design rule: **each primitive measures exactly one thing**.

If you find yourself writing "and" or "or" in a description, you are probably trying to combine two signals into one primitive. Split them.

```json
// Wrong — two signals combined
{ "primitive_id": "deal.engagement_and_sentiment", "type": "float" }

// Right — two separate primitives
{ "primitive_id": "deal.engagement_score", "type": "float" }
{ "primitive_id": "deal.sentiment_score", "type": "float" }
```

The system combines primitives into concepts automatically. Your job is to provide the raw signals, not pre-combine them.

---

## Internal vs External Signals

One of Memintel's key capabilities is evaluating **your internal data against external signals** — regulatory changes, market data, peer benchmarks. Both types are registered as primitives in exactly the same way.

```json
// Internal signal — your own data
{
  "primitive_id": "filing.deprecated_tag_count",
  "type": "int",
  "description": "Number of XBRL tags in this draft that are deprecated in the new taxonomy"
}

// External signal — regulatory feed
{
  "primitive_id": "taxonomy.tag_deprecated_flag",
  "type": "boolean",
  "description": "True if this tag is deprecated in the current SEC GAAP taxonomy version"
}

// External signal — peer benchmark
{
  "primitive_id": "provider.peer_deviation_percentile",
  "type": "float",
  "description": "This provider's billing deviation percentile within their specialty peer group, 0-100"
}
```

Register external signals the same way as internal ones. Your data engineer connects them to the appropriate external data source in the config file.

---

## Complete Examples by Domain

### SaaS Churn Detection

**Config file entries:**
```yaml
primitives:
  user.days_since_last_login:
    connector: postgres.auth
    query: >
      SELECT EXTRACT(DAY FROM (:as_of::date - last_login_at::date))::int
      FROM users WHERE user_id = :entity_id

  account.active_user_rate_30d:
    connector: postgres.analytics
    query: >
      SELECT (active_users::float / total_seats)
      FROM account_metrics
      WHERE account_id = :entity_id AND recorded_at <= :as_of
      ORDER BY recorded_at DESC LIMIT 1

  account.payment_failed_flag:
    connector: rest.billing_api
    path: /accounts/:entity_id/payment-status
    field: payment_failed
```

**API registrations:**
```bash
# Register each primitive
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "user.days_since_last_login", "type": "int", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "account.active_user_rate_30d", "type": "float", "missing_data_policy": "null"}'
curl -X POST .../v1/registry/definitions -d '{"primitive_id": "account.payment_failed_flag", "type": "boolean", "missing_data_policy": "null"}'
```

### Credit Risk Monitoring

```json
{ "primitive_id": "borrower.dscr", "type": "float", "description": "Debt service coverage ratio — EBITDA divided by total debt service. Below 1.0 means insufficient cash flow." }
{ "primitive_id": "borrower.dscr_trend_4q", "type": "time_series<float>", "description": "DSCR across last 4 quarters, oldest to newest — enables declining trend detection" }
{ "primitive_id": "borrower.leverage_ratio", "type": "float", "description": "Total debt divided by EBITDA" }
{ "primitive_id": "loan.covenant_headroom_pct", "type": "float", "description": "Distance to nearest covenant threshold as a percentage — negative means breach" }
{ "primitive_id": "borrower.management_sentiment_score", "type": "float?", "description": "LLM-extracted sentiment from most recent management commentary, 0-1 — null if unavailable" }
```

### Clinical Trial Safety

```json
{ "primitive_id": "patient.ae_severity_score", "type": "float", "description": "Composite adverse event severity score based on MedDRA grades, 0-1" }
{ "primitive_id": "patient.ae_relatedness_signal", "type": "float", "description": "LLM-extracted probability that the most recent AE is related to the study drug, 0-1" }
{ "primitive_id": "patient.sae_count_30d", "type": "int", "description": "Number of serious adverse events in the last 30 days" }
{ "primitive_id": "compound.fda_class_safety_alert_flag", "type": "boolean", "description": "True if FDA has issued a safety communication for this compound class in the last 90 days" }
```

---

## Working with Your Data Engineer

For each primitive, the workflow is:

1. **You define the signal** — agree on the name, what it measures, and its value range. Write a clear description.
2. **Data engineer adds the config entry** — maps the primitive to its data source with the appropriate query or REST call. Server restarts to load the new config.
3. **Data engineer registers the type** — calls `POST /registry/definitions` with the primitive's type, namespace, and missing data policy.
4. **You verify** — create a test monitoring task using the primitive and confirm the results make sense.

A clear description makes steps 2 and 3 much faster. The better you describe what the signal measures and what its value range means, the less back-and-forth is needed.

---

## Common Mistakes

**Forgetting to restart after config changes.** Changes to `memintel_config.yaml` require a server restart to take effect. If you add a primitive to the config but don't restart, the runtime still uses the old config.

**Registering via API before restarting.** Register the type via `POST /registry/definitions` after the server has restarted with the new config — not before. The runtime needs the config entry to be live before it can execute concepts that use the primitive.

**Name mismatch between config and registry.** The primitive name in `memintel_config.yaml` and the `primitive_id` in `POST /registry/definitions` must match exactly — case-sensitive, including dots and underscores.

**Defining concepts as primitives.** "Deal health score", "customer risk level", "account engagement" — these are concepts the compiler derives from primitive signals. Register the underlying signals instead.

**Forgetting to mark nullable signals.** If a signal sometimes has no value, declare it as `type?` and set `missing_data_policy: "null"`. An unexpected null on a non-nullable primitive causes evaluation errors.

**Missing time-series variants.** If you want to detect trends — "declining over the last 4 quarters", "increasing over the last 8 weeks" — register a `time_series<float>` or `time_series<int>` variant. The plain `float` version only tells you the current value.
