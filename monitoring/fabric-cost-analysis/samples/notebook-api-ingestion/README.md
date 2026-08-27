# Notebook + API cost ingestion (sample)

This folder adds a **notebook-only** alternative to FCA's default ingestion path
(Azure Cost Management **export** → Data Lake Gen2 → OneLake shortcut → `01_Load_Focus*`
notebooks, driven by the `Load FCA E2E` pipeline).

[`Cost_API_Ingestion_Sample.ipynb`](./Cost_API_Ingestion_Sample.ipynb) calls the
**Azure Cost Management Query API** and the **Fabric Admin REST API** directly from a
Fabric notebook, using the AAD identity that runs the notebook (your own sign-in, or a
service principal when scheduled) — no separate storage account or export configuration
required.

## Why you'd use this instead of the full FCA pipeline

| | Export + pipeline (FCA default) | Notebook + API (this sample) |
|---|---|---|
| Setup | ADLS Gen2 account, Cost Management export, OneLake shortcut, `Load FCA E2E` pipeline | One notebook, no extra storage |
| Schema | Full **FOCUS 1.0** | Reduced Cost Management shape (Cost, MeterCategory, ResourceId, …) |
| History | Up to 1 year via chunked export | Bounded by Cost Management API limits (~13 months) |
| Freshness | Export cadence (daily/monthly) | On demand, any time you run/schedule the notebook |
| Multi-subscription | Native (shortcut over multiple export folders) | Loop over `scopes` in the notebook |
| Best for | Ongoing production FinOps reporting at scale | Prototyping, small estates, or environments where export/storage provisioning isn't an option |

## How auth maps to "your admin ID"

`notebookutils.credentials.getToken(...)` requests a token for **whoever is running the
notebook** — no client secret is embedded. Two audiences are used:

- `https://management.azure.com/` — Azure Resource Manager, for the Cost Management
  Query API. The caller needs **Cost Management Reader** (or **Billing Reader**) on each
  scope queried.
- `https://api.fabric.microsoft.com/` — Fabric REST **Admin API**, used to resolve
  capacity IDs to friendly names. The caller needs the **Fabric Administrator** role, and
  [admin API access must be enabled for the tenant](https://learn.microsoft.com/en-us/fabric/admin/tenant-settings-index#developer-settings).

When this notebook is scheduled (via the notebook's own **Schedule**, or a Data Pipeline
**Notebook activity**), the identity is whichever principal owns that schedule/pipeline —
give that principal the same two roles above instead of relying on an interactive sign-in.

## Growing this into a full dashboard

1. Run the notebook on a schedule to keep `cost_fabric_api` and `dim_fabric_capacities`
   fresh (append/clean pattern shown in the notebook, mirroring `01_Load_Focus_Fabric`).
2. Add a calendar dimension (FCA's `01_Load_Calendar` notebook can be reused as-is).
3. Build a Direct Lake semantic model over `cost_fabric_api` + `dim_fabric_capacities` +
   calendar, and a Power BI report — FCA's `FCA_Core_SM` / `FCA_Core_Report` pages
   (Home, Summary, Capacity Usage, Cost Detail) are a good structural reference even
   though the source columns here are narrower than full FOCUS.
4. If/when you need full FOCUS coverage, longer history, or many subscriptions, switch
   to FCA's native export-based ingestion documented in [`../../Deploy.md`](../../Deploy.md).
