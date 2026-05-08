# 3 — Extra Input: Learnings from the Fabric Roadshow 2026 E2E demo

Session: building OrdersAnalytics (workspace `Fabric Roadshow 2`) end-to-end from a redacted mirrored database to a published Power BI report — purely from the GitHub Copilot CLI, using `fab` CLI + skills-for-fabric + kpbray/power-bi-agent-skills.

---

## 1. The mirrored database was heavily redacted

`RvDSQL` (AdventureWorksLT mirror) does NOT expose the full schema. Missing columns we expected:

| Table | Missing columns | Workaround |
|---|---|---|
| `SalesLT.Address` | `StateProvince`, `CountryRegion` | Derive state from `PostalCode` prefix lookup table |
| `SalesLT.Product` / `ProductCategory` | `Name` | Hardcoded category names from public AdventureWorksLT taxonomy |
| `SalesLT.SalesOrderDetail` | `LineTotal` | Compute as `OrderQty * UnitPrice * (1 - UnitPriceDiscount)` |
| `SalesLT.SalesOrderHeader` | `TotalDue` (and likely `SubTotal`/`TaxAmt`/`Freight` are the *only* money columns) | Aggregate the computed `LineTotal` from `SalesOrderDetail` up to header grain — do NOT reference `TotalDue`, Spark notebooks die mid-run with `System_Cancelled_Session_Statements_Failed` and no useful diagnostics. Confirmed in Roadshow take 2 (2026-05-08 16:xx). |

➡ This is exactly why the prompt said *"state is not in the data, so add this"*. Always inspect the mirrored schema first (`sys.columns` from the SQL endpoint) — don't trust the source schema docs.


## 2. Spark notebooks DO work — the prior session was wrong to skip them

Update from the 2026-05-08 session: a PySpark medallion notebook ran end-to-end on Trial-Remco in **86 seconds** (Bronze read → Silver clean → Gold star schema → validation), producing the same totals as the T-SQL Warehouse build (`$708,690.15` / 32 orders / 64 IQR outliers). The earlier "Spark is a black hole" lesson was overstated — what actually matters is **how you author and submit the notebook**.

### What worked

- **Author as `.py` source, not `.ipynb`.** Fabric notebooks deploy via `fab import` with `--format .py` (note the leading dot — `-f .py` does NOT work, the shorthand fails with `unknown shorthand flag: .py`; you must use the long form `--format .py`).
- **Folder layout the importer expects:**
  ```
  notebooks/01_medallion_build.Notebook/
      .platform                  # gitIntegration/platformProperties/2.0.0, type=Notebook
      notebook-content.py        # # Fabric notebook source  +  # CELL ******** delimiters
  ```
- **`# CELL ********************` and `# MARKDOWN ********************` are the delimiters** between cells. Each cell ends with a `# METADATA ********************` block (typically just `# META {}`).
- **The default-lakehouse block in the leading `# META` is mandatory** for `spark.read.format("delta").load(abfss://...)` calls to authenticate without prompting:
  ```python
  # META   "dependencies": {
  # META     "lakehouse": {
  # META       "default_lakehouse": "<lakehouseGuid>",
  # META       "default_lakehouse_name": "gold_lh",
  # META       "default_lakehouse_workspace_id": "<wsGuid>"
  # META     }
  # META   }
  ```
- **Read mirrored shortcuts directly via ABFSS** — no need to register them in the lakehouse SQL endpoint first:
  ```python
  abfss://{WS_ID}@onelake.dfs.fabric.microsoft.com/{BRONZE_ID}/Tables/SalesOrderDetail
  ```
  `spark.read.format("delta").load(...)` resolves shortcuts transparently.

### Triggering a run from the CLI

The Items API job endpoint works without any extra permissions and returns a Location header you can poll:
```bash
POST https://api.fabric.microsoft.com/v1/workspaces/{ws}/items/{notebookId}/jobs/instances?jobType=RunNotebook
# returns 202 + Location header pointing at /jobs/instances/{jobId}
GET  https://api.fabric.microsoft.com/v1/workspaces/{ws}/items/{notebookId}/jobs/instances/{jobId}
# poll status until Completed | Failed | Cancelled | Deduped
```
Audience: `https://api.fabric.microsoft.com` (Items API, not Datasets).

### Lakehouse SQL endpoint discovery is asynchronous

After the notebook writes Delta to `Tables/<name>` via ABFSS, the lakehouse SQL endpoint takes ~30–60 seconds to surface the new tables in `INFORMATION_SCHEMA.TABLES`. Querying `gold_lh.dbo.fact_sales_order` from the warehouse immediately after the job completes returned `Invalid object name`; the same query succeeded a minute later. **Add a small backoff before you validate cross-database from the warehouse** — or validate inside the notebook itself, where the freshly written paths are guaranteed visible.

### When Spark really would have been a trap

The original lesson — Spark diagnostics are unreachable via the Jobs REST API — still holds: `/jobs/instances/{id}/snapshot`, `/logs`, `/sparkUiLink` all 404 or 401. So **if a notebook fails, the only practical signal you get from the CLI is the terminal `status` and `failureReason`**. Mitigation: write the notebook with explicit `print(...)` checkpoints between cells so the failure reason includes a recognisable string, and keep the T-SQL Warehouse path available as a fallback.


## 3. Fabric Warehouse T-SQL gotchas

- ❌ `nvarchar` is NOT supported in DW. Use `varchar(n)` everywhere.
- ❌ `DATENAME(...)` returns `nvarchar` → wrap in `CAST(... AS varchar(20))`.
- ✅ Cross-database queries within the same workspace work natively (`<lakehouse>.dbo.<table>`).
- ✅ `PERCENTILE_CONT(...) WITHIN GROUP (ORDER BY ...) OVER ()` works for IQR outlier detection.

## 4. AAD authentication for SQL via Python

```python
import struct, pyodbc, subprocess, json
tok = json.loads(subprocess.check_output(
    ["az","account","get-access-token","--resource","https://database.windows.net/"]))["accessToken"]
buf = tok.encode("utf-16-le")
token_struct = struct.pack(f"=i{len(buf)}s", len(buf), buf)
SQL_COPT_SS_ACCESS_TOKEN = 1256
conn = pyodbc.connect(connstr, attrs_before={SQL_COPT_SS_ACCESS_TOKEN: token_struct})
```

## 5. `fab` CLI quirks

| Quirk | Fix |
|---|---|
| Notebook import demands `--format .py` (with leading dot!) | `fab import ... -f .py` |
| Notice spam | `$env:FAB_DISABLE_NOTICES='1'` (still noisy though) |
| `fab ls Tables/` on schema-less lakehouse lists shortcuts as `<name>.Shortcut` | Filter in PowerShell |
| Re-import = update | "An item with the same name exists / Importing (update)…" |

## 6. TMDL semantic model — what bit me

- ✅ **TABS, not spaces**, in TMDL. Verify with hex dump (`0x09`).
- ❌ Don't add manual `lineageTag` to expressions — auto-generated; conflicts.
- ❌ Don't put `ref table X` and `ref cultureInfo en-US` in `model.tmdl` if you also include the per-table files — fab/Fabric handles refs itself, the explicit refs broke parsing.
- ❌ TMDL `relationship` does NOT use `fromTable` / `toTable`. Only:
  ```
  relationship rel_name
      fromColumn: factTable.FKColumn
      toColumn:   dimTable.PKColumn
  ```
- ✅ Direct Lake on a Warehouse: `expression DatabaseQuery = let ... Sql.Database("<sqlEndpoint>", "<dbName>") ... in database` then `partition X = entity / mode: directLake / entityName / schemaName: dbo / expressionSource: DatabaseQuery`.
- ✅ **The TMDL display name (with spaces and quoting) is what visuals/DAX reference, NOT `sourceColumn`.** This is what made every visual return blanks initially:
  ```
  column 'Category Name'        -- ← reference THIS in PBIR (Property: "Category Name")
      sourceColumn: CategoryName -- ← NOT THIS
  ```

## 7. PBIR enhanced report format — what actually works in Fabric service

The `kpbray/power-bi-agent-skills` `report-visuals` skill uses the older 1.0.0 schema URLs with inline `visuals[]`. **Fabric service rejects this.** The actually-working enhanced format (verified against `microsoft/BCApps`):

| File | Content |
|---|---|
| `definition.pbir` | `definitionProperties/2.0.0` schema, **`"version": "4.0"`** (NOT `"1.0"`) |
| `definition/version.json` | `versionMetadata/1.0.0` schema, `"version": "2.0.0"` — **required** |
| `definition/report.json` | `report/3.0.0` schema. `themeCollection.baseTheme` MUST have `name`, `reportVersionAtImport` (object, not string!), `type` |
| `definition/pages/pages.json` | `pagesMetadata/1.0.0`, `pageOrder` + `activePageName` |
| `definition/pages/<id>/page.json` | `page/2.0.0` |
| `definition/pages/<id>/visuals/<name>/visual.json` | `visualContainer/2.4.0` — **one file per visual**, NOT inline |

### `definition.pbir` byConnection minimum

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definitionProperties/2.0.0/schema.json",
  "version": "4.0",
  "datasetReference": {
    "byConnection": {
      "connectionString": "Data Source=powerbi://api.powerbi.com/v1.0/myorg/<WorkspaceName>;Initial Catalog=<ModelName>;Integrated Security=ClaimsToken;semanticModelId=<modelGuid>"
    }
  }
}
```

- Schema 2.0.0 only allows `connectionString` — older `pbiServiceModelId`/`pbiModelVirtualServerName`/`pbiModelDatabaseName`/`name`/`connectionType` properties are rejected.
- `byPath` only works in PBIP local dev — Fabric import errors with `Definition includes byPath; switch to byConnection`.
- `semanticModelId=<guid>` in the connection string is **mandatory**, otherwise `InvalidConnectionInformation`.

### `themeCollection.baseTheme` shape that works

```json
"baseTheme": {
  "name": "CY24SU10",
  "reportVersionAtImport": {"visual": "1.8.23", "report": "2.0.23", "page": "1.3.23"},
  "type": "SharedResources"
}
```

### Visual projection — `nativeQueryRef` is required

```json
{
  "field": {"Measure": {"Expression": {"SourceRef": {"Entity": "fact_sales_order"}}, "Property": "Sum Order Value"}},
  "queryRef":       "fact_sales_order.Sum Order Value",
  "nativeQueryRef": "Sum Order Value",
  "active": true
}
```

Without `nativeQueryRef` the report imports but every visual shows "Can't display the visual".

### Visual type names that work

- `card`, `tableEx`, `slicer`, `donutChart`
- `clusteredBarChart` (NOT `barChart`)
- `clusteredColumnChart` (NOT `columnChart`)

## 8. Fabric API audiences — easy to get wrong

| Operation | `--resource` |
|---|---|
| Items API (semantic models, reports, getDefinition, updateDefinition) | `https://api.fabric.microsoft.com` |
| Datasets API (refresh, executeQueries, datasources, permissions) | `https://analysis.windows.net/powerbi/api` |
| OneLake DFS (file CRUD via `curl`) | `https://storage.azure.com/` |
| SQL endpoint via pyodbc | `https://database.windows.net/` |

Wrong audience = silent 401. Always pass `--resource` explicitly to `az rest`.

## 9. Direct Lake — frame before query

After deploying or updating a Direct Lake model, the first `executeQueries` returns `ArtifactNotFound` until a "framing" refresh runs:

```bash
POST /v1.0/myorg/groups/{ws}/datasets/{id}/refreshes
Body: {"type":"Full"}
```

Status comes back as `refreshType: DirectLakeFraming`, completes in seconds. Then DAX works.

## 10. PowerBIQuery MCP tool caches model list

After deploying a brand-new semantic model, the `PowerBIQuery-ExecuteQuery` MCP tool returned `ArtifactNotFound` for hours, while `executeQueries` REST API on the same GUID worked instantly. Lesson: when validating a freshly deployed model, **use the REST API directly**, not the MCP tool.

## 11. OpenSpec gotcha

`openspec validate --strict` requires **SHALL / MUST in the requirement body text**, not just in the heading. Easy to miss — fail fast by validating after every edit.

## 12. Squad-as-skills mapping that worked

| Bob the Builder character | Skill / tool |
|---|---|
| 🧱 Bob | `fab CLI` workspace + capacity ops |
| 🪡 Wendy | `semantic-model` + `dax` (kpbray) |
| 🛠️ Scoop | `pbip-project` (kpbray) folder/pointer files |
| 🚜 Muck | `fab import` for everything |
| 🌀 Dizzy | `sqldw-authoring-cli` (build_warehouse_v2.py) |
| 🛞 Roley | OpenSpec proposal + repo init |
| 🦅 Lofty | `report-visuals` (kpbray) PBIR authoring |
| 🦘 Travis | git/GitHub commits as `Remc0000` |

## 13. Things to add to a future skill

A `power-bi-report-cli` skill is missing from skills-for-fabric. The closest is `report-visuals` from kpbray, but its 1.0.0 schemas don't match what Fabric service currently accepts. A skill that codifies the **2.4.0 visualContainer / 3.0.0 report / 4.0 definition.pbir** combo with `nativeQueryRef` examples would have saved ~20 minutes of trial-and-error in this session.

## 14. Roadshow take 2 (2026-05-08) — additional lessons

Re-ran the same E2E build live on stage, workspace `Fabric Roadshow` on `Trial-Remco`. Total elapsed **1h 03m** — faster than take 1 (1h 14m) thanks to these notes, but still well over the 30-minute target. New findings:

### 14.1 `fab import` from the GitHub Copilot CLI session crashes

Every `fab import …` call from inside the Copilot CLI shell exits with:

```
No Windows console found. Are you running cmd.exe?
```

This affected **notebook**, **semantic model**, and **report** imports. The `fab` CLI seems to require an interactive Windows console handle that the Copilot CLI subshell does not provide. There is no `--non-interactive` flag that fixes it.

➡ **Reliable workaround for ALL item imports: use the Items REST API directly.**

| Operation | REST call | Audience |
|---|---|---|
| Create new item | `POST /v1/workspaces/{ws}/items` with `{displayName, type, definition:{parts:[{path,payload(b64),payloadType:"InlineBase64"}]}}` | `https://api.fabric.microsoft.com` |
| Update existing item | `POST /v1/workspaces/{ws}/items/{id}/updateDefinition` with same `definition` shape | `https://api.fabric.microsoft.com` |
| Get item | `GET /v1/workspaces/{ws}/items/{id}` | `https://api.fabric.microsoft.com` |

Every artefact (notebook, TMDL semantic model, PBIR report) deployed cleanly via this route. Codify this in `skills-for-fabric` rather than relying on `fab import`.

### 14.2 Notebook source format: `.ipynb` beats cell-delimited `.py` over REST

Muck saved a `notebook-content.py` with the documented `# CELL ********` delimiters. Round-tripping through git / REST mangled CRLF line endings and the importer rejected the cells. Switching to a real `notebook-content.ipynb` (JSON) parsed reliably first try. Keep the `.py` as a human-readable mirror, but **deploy the `.ipynb`**.

### 14.3 Direct Lake `Sql.Database()` — use the SQL endpoint GUID, not the lakehouse GUID

TMDL expression:

```
expression DatabaseQuery =
    let database = Sql.Database("<sqlEndpointHost>", "<???>")
    in database
```

If `<???>` is the **lakehouse GUID** (e.g. `gold_lh.id`), the model deploys but the **first framing refresh fails** (`Refresh failed`, no useful detail). If it is the **SQL endpoint GUID** (the item type `SQLAnalyticsEndpoint` that Fabric auto-creates next to the lakehouse), framing succeeds and DAX works.

➡ Always resolve the SQL endpoint id explicitly:

```bash
az rest --resource https://api.fabric.microsoft.com \
  --url "https://api.fabric.microsoft.com/v1/workspaces/{ws}/items?type=SQLAnalyticsEndpoint" \
  --query "value[?displayName=='<lakehouseName>'].id"
```

(Type may also list as `SQLEndpoint` depending on tenant — query both.)

### 14.4 IQR outlier detection at order grain on a 32-row dataset = always 0 outliers

The original prompt's target of **64 outliers** was based on **line-level** IQR (542 detail rows). When the fact is aggregated to **order grain (32 rows × 4 categories)**, `1.5 × IQR` is wider than the max within every category, so `IsOutlier = TRUE` for nothing.

➡ For meaningful outlier counts at order grain, either (a) widen the partition (year × category instead of category alone), (b) use a tighter multiplier (1.0 × IQR or z-score > 1), or (c) keep outlier detection at line grain and aggregate `OutlierLineCount` up. Document the chosen rule in the spec — don't hard-code a target number that depends on grain.

### 14.5 PBIR REST deploy — strip unsupported visual fields

Fabric Items API rejects `definition.pbir` reports if any visual JSON contains stray properties not in the `visualContainer/2.4.0` schema (we hit it on `config` and `ordinal`). Error is generic (`InvalidDefinition`), no field pointer. Mitigation: validate every `visual.json` against the published 2.4.0 schema before POSTing, or strip unknown keys after generating from a higher-level template.

### 14.6 `gh repo delete` needs the `delete_repo` OAuth scope

`gh auth refresh -h github.com -s delete_repo` triggers a device-flow login that **blocks indefinitely** in the Copilot CLI shell (no browser, no way to enter the device code). For live demos, **don't try to delete the repo**: instead reset to a clean state with `git init` + `git push -f origin main` of an orphan commit. Same effect, no interactive auth.

### 14.7 OpenSpec — every requirement needs ≥1 scenario, not just SHALL/MUST in the body

Building on lesson 11: even a perfectly worded SHALL requirement fails `--strict` validation with `must include at least one scenario` if you forgot the `#### Scenario:` block. Add a placeholder scenario as soon as you create the requirement heading; flesh it out later.

### 14.8 `openspec init` is interactive

Running `openspec init` in a non-TTY context blocks on a tool-selection prompt. Sending `{enter}` twice picks the defaults and unblocks. Worth a `--yes` / `--defaults` flag upstream.

---

**Total demo time:** 1h 14m (take 1) → 1h 03m (take 2). Still need to break the 30-minute target — biggest remaining wins:
1. Skip Spark, build the warehouse path in parallel as a fallback (saves ~5 min when Spark hits a redacted column)
2. Pre-baked TMDL skeleton + display-name mapping (saves ~10 min)
3. Pre-baked PBIR skeleton with the 2.4.0 visual templates (saves ~10 min)

**Final repos:**
- Take 1: https://github.com/Remc0000/FabricRoadshow (now reset for take 2)
- Take 2: https://github.com/Remc0000/FabricRoadshow

**Final workspace:** `Fabric Roadshow` on `Trial-Remco.Capacity`
