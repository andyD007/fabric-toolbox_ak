# Notebook + API cost dashboard (enhanced, chargeback-focused)

This folder is a **notebook-only** alternative to FCA's default ingestion path
(Azure Cost Management **export** → Data Lake Gen2 → OneLake shortcut → `01_Load_Focus*`
notebooks, driven by the `Load FCA E2E` pipeline), purpose-built around four questions FCA's
stock report doesn't directly answer:

1. **Capacity-level chargeback** — what did each capacity (and, with the extension in
   [Chargeback allocation](#chargeback-allocation-capacity--workspace) below, each workspace) actually cost?
2. **Throttling** — when are we consuming more CU than the capacity's SKU allows, and is that
   what's driving cost up?
3. **Budget vs actual** — are we tracking against a defined budget, and by how much?
4. **Utilization** — how much of each capacity's CU pool is actually being used?

None of this reuses `FCA_Core_Report`/`FCA_Core_SM` — it's a from-scratch model, built to be
extended rather than cloned.

## The three ingestion notebooks

| Notebook | Source | Lands into | Answers |
|---|---|---|---|
| [`Cost_API_Ingestion_Sample.ipynb`](./Cost_API_Ingestion_Sample.ipynb) | Azure **Cost Management Query API** (`management.azure.com`) + Fabric **Admin REST API** | `cost_fabric_api`, `dim_fabric_capacities` | Chargeback — $ cost per capacity/resource/day |
| [`Utilization_Throttling_Ingestion_Sample.ipynb`](./Utilization_Throttling_Ingestion_Sample.ipynb) | **Fabric Capacity Metrics app** semantic model, queried via **DAX** (`sempy.fabric.evaluate_dax`, the SemPy wrapper over the Power BI/Fabric "Execute Queries" REST API / XMLA endpoint) | `capacity_utilization` | Throttling + utilization — CU%, rejection %, carry-over, per capacity/timepoint |
| [`Budget_vs_Actual_Ingestion_Sample.ipynb`](./Budget_vs_Actual_Ingestion_Sample.ipynb) | Azure **Consumption Budgets API** (`Microsoft.Consumption/budgets`) | `budgets` | Budget vs actual — current/forecast spend against a defined budget |

**Why throttling/utilization can't come from Cost Management at all**: CU%, throttling
minutes, and carry-over/overage are runtime capacity telemetry — they only exist in the
**Fabric Capacity Metrics app**'s own semantic model, not in any billing API. The only
supported way to pull them programmatically is a DAX query against that model (via the
Execute Queries REST API or the XMLA endpoint) — there's no plain REST endpoint for it, which
is why that notebook uses `sempy.fabric.evaluate_dax` instead of `requests`.

## How auth maps to "your admin ID"

`notebookutils.credentials.getToken(...)` (cost/budget notebooks) and `sempy.fabric.evaluate_dax`
(utilization notebook) both run as **whoever is executing the notebook** — no client secret
embedded. Required roles, by notebook:

| Notebook | Audience | Role needed |
|---|---|---|
| Cost | `management.azure.com` | **Cost Management Reader** (or **Billing Reader**) on each scope |
| Cost (enrichment) | `api.fabric.microsoft.com` | **Fabric Administrator**, tenant admin APIs enabled |
| Utilization/Throttling | Metrics app semantic model | **Viewer** on the workspace hosting the Fabric Capacity Metrics app |
| Budget | `management.azure.com` | **Cost Management Reader** on each scope |

When any of these notebooks is scheduled (notebook **Schedule**, or a Data Pipeline
**Notebook activity**), the identity is whichever principal owns that schedule/pipeline —
grant that principal the same roles instead of relying on interactive sign-in.

## Enhanced star schema

```
dim_calendar ─────────┐
dim_fabric_capacities ─┼─→ fact_cost_fabric        (from cost_fabric_api)
                       ├─→ fact_capacity_utilization (from capacity_utilization)
                       └─→ fact_budgets              (from budgets)
```

- **`dim_fabric_capacities`** — CapacityId, CapacityName, Sku, Region (from the Cost notebook's
  Fabric Admin API call)
- **`dim_calendar`** — reuse FCA's `01_Load_Calendar` notebook as-is
- **`fact_cost_fabric`** — daily $ cost per capacity/resource/meter
- **`fact_capacity_utilization`** — per-capacity, per-timepoint CU%, `IsThrottled`,
  `IsOverCULimit`, carry-over
- **`fact_budgets`** — per-scope budget amount, current spend, forecast spend, variance,
  appended over time so budget-vs-actual can be trended, not just viewed as a snapshot

Join key across all three facts is `CapacityId` (+ date/timepoint); `dim_fabric_capacities` is
the conformed dimension tying cost, utilization, and (indirectly, via scope) budget together.

## Chargeback allocation (capacity → workspace)

Azure only bills at the **capacity resource** level (e.g. one F64 SKU) — it has no idea how
much of that capacity a given workspace or item consumed. To charge that dollar cost back to
individual workspaces/teams, allocate proportionally by CU share:

```
workspace_cost($) = capacity_cost($) × ( workspace_CU_consumed / total_capacity_CU_consumed )
```

`capacity_cost($)` comes from `fact_cost_fabric`; `total_capacity_CU_consumed` comes from
`fact_capacity_utilization`. The per-*workspace* CU breakdown isn't in the Timepoints table
this sample pulls — it's in the Metrics app's `Items` table (per-item/workspace CU). Extending
`Utilization_Throttling_Ingestion_Sample.ipynb` to also pull `Items` (see FUAM's
`02_Transfer_CapacityMetricData_ItemKind_Unit.Notebook` / `03_..._ItemOperation_Unit.Notebook`
for the DAX shape) gives you the numerator for this formula. Flagging this as the natural next
build step rather than including it now, since it's a bigger DAX surface — say the word and
I'll add it.

## Correlating throttling with cost spikes

`fact_capacity_utilization.IsThrottled` and `CumulativeCUUsagePct` are the "why did cost go
up" explanation: a capacity running near/over its CU limit either throttles (rejects/delays
operations — visible immediately as `IsThrottled = true`) or, if autoscale/pay-as-you-go
overage is enabled, burns extra billed CU (visible as a `fact_cost_fabric` spike on the same
`CapacityId`/date). Plot both on the same timeline per capacity to make that link visible.

## Growing this into a full dashboard

1. Schedule all three notebooks (own **Schedule**, or one Data Pipeline chaining them, mirroring
   FCA's `Load FCA E2E` pattern with `FromMonth`/`ToMonth`-style parameters).
2. Add `dim_calendar` (reuse FCA's `01_Load_Calendar` notebook).
3. Build a Direct Lake semantic model over the three fact tables + the two dimensions, and a
   Power BI report with (at minimum) a **Chargeback** page (cost by capacity/workspace),
   **Throttling** page (CU% + rejection % over time, per capacity), and **Budget** page
   (actual vs budget vs forecast, per scope) — a genuinely different report from FCA's, built
   around these four questions rather than FOCUS.
4. If/when you need full FOCUS coverage, longer history, or many subscriptions for the cost
   side specifically, switch that one piece to FCA's native export-based ingestion documented
   in [`../../Deploy.md`](../../Deploy.md) — the utilization and budget notebooks stay as-is
   either way.
