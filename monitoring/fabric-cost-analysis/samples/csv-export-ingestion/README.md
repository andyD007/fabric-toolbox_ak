# CSV FOCUS export ingestion (reuses FCA's native downstream stack)

[`Load_Focus_CSV_Sample.ipynb`](./Load_Focus_CSV_Sample.ipynb) loads a FOCUS-schema Cost
Management export delivered as **CSV**, for the case of a **manually downloaded and
uploaded** export: one or more `part_N_NNNN.csv` files sitting directly in a Lakehouse
`Files/` folder, no date-based subfolder structure. It reads everything under that folder and
overwrites the `focus` Delta table each run — simplest and safest for an occasional manual
refresh, since there's no per-period folder to detect or merge against.

If your export instead lands automatically via a OneLake shortcut to a scheduled export's
storage container (the recurring, dated-folder layout FCA's own `01_Load_Focus.Notebook`
assumes), that needs different, folder/date-aware logic — say the word if you want that
variant built out too; this notebook intentionally doesn't try to handle both layouts at once.

## Why this instead of the API notebooks

The Cost Management **Query API** is throttled at the subscription/billing-account level,
**shared across every caller** — including any scheduled exports already running in the
tenant. Live API polling competes with them for the same quota and can end up persistently
rate-limited (`429`, sometimes surfacing as a spurious `401`) regardless of how well the
calling code backs off. Reading an already-exported file instead avoids the Query API
entirely.

## What this notebook is

Loads CSV into a `focus_staging` table, then overwrites `focus`. Once `focus` is populated,
**FCA's existing `01_Load_Focus_Fabric`, `FCA_Core_SM`, and `FCA_Core_Report` all run on top
of it with no changes** — this only replaces the very first ingestion step.

## Before your first real run

1. Upload your CSV file(s) into a Lakehouse `Files/` folder and set `rawSourcePath` to that
   folder (a folder, not a specific file).
2. Run the notebook's Step 0 cell against one of the uploaded files to confirm your export's
   actual `ServiceName` **and** `ChargeDescription` values. FCA's own
   `01_Load_Focus_Fabric.Notebook` hardcodes `ServiceName = 'Microsoft.Fabric'` — if your
   export mixes Fabric in with other services (Databricks, disk, compute, etc., as many
   exports do), `ServiceName` alone won't isolate Fabric cleanly, and `ChargeDescription`
   (e.g. values like `Fabric` / `OneLake`) is the more precise filter — Step 0 shows the exact
   strings your export uses so the filter in `01_Load_Focus_Fabric.Notebook` can be swapped to
   match them precisely (case/spacing varies by tenant, so don't assume the exact spelling
   without checking).
3. **Refresh workflow**: each time you want updated numbers, replace the file(s) under
   `rawSourcePath` with a fresh export before re-running — leaving an old file alongside a new
   one will double-count, since this reads everything present under that folder.
