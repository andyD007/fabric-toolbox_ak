# CSV FOCUS export ingestion (reuses FCA's native downstream stack)

If your Cost Management export is already configured (FOCUS schema, delivered as **CSV**
rather than the Parquet FCA's `01_Load_Focus.Notebook` expects), use
[`Load_Focus_CSV_Sample.ipynb`](./Load_Focus_CSV_Sample.ipynb) instead of the live-API
notebooks in [`../notebook-api-ingestion/`](../notebook-api-ingestion/).

## Why this instead of the API notebooks

The Cost Management **Query API** is throttled at the subscription/billing-account level,
**shared across every caller** — including your existing scheduled exports. If those exports
are already running, live API polling competes with them for the same quota and can end up
persistently rate-limited (`429`, sometimes surfacing as a spurious `401`) regardless of how
well the calling code backs off.

Reading the *exported files* instead avoids the Query API entirely — it's just a OneLake
shortcut + a Spark read, identical in spirit to what FCA's export+pipeline path already does
for Parquet FOCUS exports.

## What this notebook is

FCA's own `01_Load_Focus.Notebook`, unchanged except for the file format: same wildcard/
month-folder path-detection logic, same `focus_staging` → `focus` Delta table target. Once
`focus` is populated, **FCA's existing `01_Load_Focus_Fabric`, `FCA_Core_SM`, and
`FCA_Core_Report` all run on top of it with no changes** — this only replaces the very first
ingestion step, not anything downstream.

## Before your first real run

1. Set up a OneLake shortcut pointing at your CSV export's storage container (same steps as
   [`../../Deploy.md`](../../Deploy.md) section 1.1, just pointed at this export instead of a
   FOCUS Parquet one).
2. Run the notebook's Step 0 cell against one known CSV file to confirm your export's actual
   `ServiceName` value — FOCUS exports have been observed using both `"Microsoft Fabric"` and
   `"Microsoft.Fabric"` depending on version/tenant. FCA's own `01_Load_Focus_Fabric.Notebook`
   hardcodes `ServiceName = 'Microsoft.Fabric'` — update that filter if your export uses the
   other form.
3. Confirm your export's folder layout matches one of the two patterns FCA's path-detection
   already handles (`YYYY/MM/...` or `YYYYMMDD-YYYYMMDD/...`) — if your export uses a
   different structure, the regexes in the notebook's Step 1 cell need adjusting to match.
