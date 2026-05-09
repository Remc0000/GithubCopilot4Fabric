# 3 — Extra Input: Lessons for the Fabric Roadshow E2E demo

Reference for building **OrdersAnalytics** end-to-end (source lakehouse →
bronze/silver/gold lakehouses → Direct Lake semantic model → published
Power BI report) from the GitHub Copilot CLI, using `fab` CLI +
`skills-for-fabric` (microsoft) + `powerbi-agentic-plugins` (RuiRomano).

Read this **before** starting a run. Each section is what bit us; obey
the rules and you'll finish inside the 30-minute target.

---

## 1. Source: `SalesLT` lakehouse, full AdventureWorksLT

The source is the **`SalesLT` lakehouse** in workspace **`SalesLT`**.
Tables sit under the **`SalesLT` schema** (so the OneLake path is
`Tables/SalesLT/<TableName>`, and the SQL endpoint sees them as
`SalesLT.<TableName>`). All standard AdventureWorksLT columns are
present — `Name`, `StateProvince`, `LineTotal`, `TotalDue`, the lot.

| Table | Notes |
|---|---|
| `SalesLT.Address` | `City`, `StateProvince`, `CountryRegion`, `PostalCode` |
| `SalesLT.Product` / `ProductCategory` / `ProductModel` | `Name`, `ParentProductCategoryID` |
| `SalesLT.SalesOrderDetail` | `LineTotal` already computed in source — read it, don't recompute it |
| `SalesLT.SalesOrderHeader` | `TotalDue`, `SalesOrderNumber`, etc. |

**Read the data as-is.** No postal-code → state lookups, no hardcoded
category-name dictionaries, no derived `LineTotal`. If you find yourself
hardcoding values that already live in source, stop and re-read the
schema.

### Product Category hierarchy

`ProductCategory` is self-referencing through `ParentProductCategoryID`.
The 4 top-level rows (`Bikes`, `Components`, `Clothing`, `Accessories`)
have `ParentProductCategoryID IS NULL`; every other row has a parent.
Materialise the hierarchy in silver:

```python
top = pcat.alias('p').join(
    pcat.alias('t'),
    F.col('p.ParentProductCategoryID') == F.col('t.ProductCategoryID'),
    'left'
).select(
    F.col('p.ProductCategoryID'),
    F.col('p.Name').alias('CategoryName'),
    F.col('p.ParentProductCategoryID'),
    F.coalesce(F.col('t.Name'), F.col('p.Name')).alias('TopCategoryName'),
)
```

Carry both `CategoryName` and `TopCategoryName` into `dim_product_category`,
then expose them as a 2-level user hierarchy in the semantic model
(see §4).

---

## 2. Three-notebook medallion (Bronze / Silver / Gold)

The `e2e-medallion-architecture` skill standard is **one notebook per
layer**, not a single mega-notebook. Smaller notebooks fail faster and
isolate where the failure was.

```
fabric/notebooks/
  01_bronze_ingest.Notebook/   # default lakehouse: bronze_lh, raw read of source shortcuts
  02_silver_clean.Notebook/    # default lakehouse: silver_lh, clean + dedupe + category hierarchy
  03_gold_star.Notebook/       # default lakehouse: gold_lh, star schema + IQR outlier flag + validation prints
```

Per notebook:

- **Author as `.ipynb`** (JSON). The cell-delimited `.py` format breaks
  on CRLF round-trips through git/REST.
- **`.platform`** file at the notebook root: `gitIntegration/platformProperties/2.0.0`,
  `type=Notebook`.
- **Default-lakehouse `# META` block** in the leading cell — required for
  `spark.read.format("delta").load(abfss://...)` to authenticate without
  prompting:
  ```python
  # META   "dependencies": {
  # META     "lakehouse": {
  # META       "default_lakehouse": "<lakehouseGuid>",
  # META       "default_lakehouse_name": "<layer>_lh",
  # META       "default_lakehouse_workspace_id": "<wsGuid>"
  # META     }
  # META   }
  ```
- **Read shortcuts via ABFSS directly** — no need to register them in the
  SQL endpoint first. Schema-aware shortcuts to the source lakehouse live
  under the `SalesLT` schema:
  ```python
  abfss://{WS_ID}@onelake.dfs.fabric.microsoft.com/{BRONZE_ID}/Tables/SalesLT/SalesOrderDetail
  ```
- **Validate inside the notebook** with `print(f"TOTAL_ORDER_VALUE={...}")`
  checkpoints. Spark job logs (`/jobs/instances/{id}/snapshot|logs|sparkUiLink`)
  all 404/401 from REST, so prints in the failure-reason payload are your
  only signal.
- **Use markdown cells generously** to explain WHAT and WHY — the audience
  reads them on screen.

### Run them in order via the Items jobs API

```http
POST https://api.fabric.microsoft.com/v1/workspaces/{ws}/items/{notebookId}/jobs/instances?jobType=RunNotebook
→ 202 + Location header /jobs/instances/{jobId}
GET  https://api.fabric.microsoft.com/v1/workspaces/{ws}/items/{notebookId}/jobs/instances/{jobId}
→ poll until Completed | Failed | Cancelled | Deduped
```

Audience: `https://api.fabric.microsoft.com`. Gate each layer on the
previous returning `Completed`.

### SQL endpoint discovery is asynchronous

After a notebook writes Delta to `Tables/<name>`, the lakehouse SQL
endpoint takes ~30–60 s to surface the new tables in
`INFORMATION_SCHEMA.TABLES`. Validate inside the notebook (where ABFSS
paths are immediately visible), or back off ~60 s before any
cross-database SQL query.

---

## 3. Item deployment: REST, not `fab import`

`fab import` from the GitHub Copilot CLI subshell crashes with:

```
No Windows console found. Are you running cmd.exe?
```

This affects notebooks, semantic models, and reports. There is no
`--non-interactive` flag. **Use the REST API instead** for every item type:

| Operation | REST call |
|---|---|
| Create | `POST /v1/workspaces/{ws}/items` with `{displayName, type, definition:{parts:[{path,payload(b64),payloadType:"InlineBase64"}]}}` |
| Update | `POST /v1/workspaces/{ws}/items/{id}/updateDefinition` with same `definition` shape |
| Get    | `GET  /v1/workspaces/{ws}/items/{id}` |

Audience for all: `https://api.fabric.microsoft.com`.

### Other `fab` CLI quirks (when it does work)

| Quirk | Fix |
|---|---|
| Notice spam | `$env:FAB_DISABLE_NOTICES='1'` (still noisy though) |
| `fab ls Tables/` lists shortcuts as `<name>.Shortcut` on schema-less lakehouse | Filter in PowerShell |
| Re-import = update | Message: "An item with the same name exists / Importing (update)…" |

---

## 4. TMDL semantic model — Direct Lake on a lakehouse

- ✅ **TABS, not spaces.** Verify with hex dump (`0x09`).
- ❌ Do **not** add manual `lineageTag` to expressions — auto-generated,
  conflicts.
- ❌ Do **not** put `ref table X` / `ref cultureInfo en-US` in `model.tmdl`
  if you also include the per-table files — fab/Fabric handles refs
  itself; explicit refs break parsing.
- ❌ TMDL `relationship` does **not** use `fromTable`/`toTable`. Only:
  ```
  relationship rel_name
      fromColumn: factTable.FKColumn
      toColumn:   dimTable.PKColumn
  ```
- ✅ Direct Lake expression — the second arg of `Sql.Database()` MUST be
  the **SQL endpoint GUID** (item type `SQLAnalyticsEndpoint`,
  auto-created next to the lakehouse), NOT the lakehouse GUID. Wrong
  one deploys but framing fails:
  ```
  expression DatabaseQuery =
      let database = Sql.Database("<sqlEndpointHost>", "<sqlEndpointGuid>")
      in database
  ```
  Resolve the SQL endpoint GUID after creating the lakehouse:
  ```bash
  az rest --resource https://api.fabric.microsoft.com \
    --url "https://api.fabric.microsoft.com/v1/workspaces/{ws}/items?type=SQLAnalyticsEndpoint" \
    --query "value[?displayName=='<lakehouseName>'].id"
  ```
  Per table: `partition X = entity` with `mode: directLake`,
  `entityName: <table>`, `schemaName: dbo`, `expressionSource: DatabaseQuery`.
- ✅ **Display name (with spaces and quoting) is what visuals/DAX reference,
  NOT `sourceColumn`.** Get this wrong and every visual returns blanks:
  ```
  column 'Category Name'         -- ← reference THIS in PBIR (Property: "Category Name")
      sourceColumn: CategoryName  -- ← NOT THIS
  ```
- ✅ **User hierarchy** for the Product Category Top → Sub drill:
  ```
  hierarchy 'Category Hierarchy'
      level 'Top Category Name'
          column: 'Top Category Name'
      level 'Category Name'
          column: 'Category Name'
  ```
  Hangs off `dim_product_category`. Reference the hierarchy by its
  display name from PBIR visuals.

---

## 5. PBIR enhanced report format

Use the **RuiRomano `powerbi@powerbi-agentic-plugins`** plugin — its
templates target the schemas Fabric service expects.

| File | Schema |
|---|---|
| `definition.pbir` | `definitionProperties/2.0.0`, `"version": "4.0"` |
| `definition/version.json` | `versionMetadata/1.0.0`, `"version": "2.0.0"` (required) |
| `definition/report.json` | `report/3.0.0`. `themeCollection.baseTheme` MUST have `name`, `reportVersionAtImport` (object), `type` |
| `definition/pages/pages.json` | `pagesMetadata/1.0.0`, `pageOrder` + `activePageName` |
| `definition/pages/<id>/page.json` | `page/2.0.0` |
| `definition/pages/<id>/visuals/<n>/visual.json` | `visualContainer/2.4.0` (one file per visual) |

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

- Schema 2.0.0 only allows `connectionString`. Older `pbiServiceModelId` /
  `pbiModelVirtualServerName` / `pbiModelDatabaseName` / `name` /
  `connectionType` are rejected.
- `byPath` only works in PBIP local dev — Fabric import errors with
  `Definition includes byPath; switch to byConnection`.
- `semanticModelId=<guid>` is **mandatory**, otherwise
  `InvalidConnectionInformation`.

### `themeCollection.baseTheme` shape

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

Without `nativeQueryRef` the report imports but every visual shows
"Can't display the visual".

### Visual types that work

- `card`, `tableEx`, `slicer`, `donutChart`
- `clusteredBarChart` (NOT `barChart`)
- `clusteredColumnChart` (NOT `columnChart`)
- `pivotTable` / `tableEx` with a hierarchy in Rows for Top → Sub drill

### Strip unsupported fields before POST

Items API rejects visuals carrying stray keys outside the
`visualContainer/2.4.0` schema (we hit it on `config` and `ordinal`).
Error is generic (`InvalidDefinition`), no field pointer. Validate every
`visual.json` against the schema, or strip unknown keys from a
higher-level template, before deploying.

---

## 6. Direct Lake — frame before query

The first `executeQueries` after deploy/update returns `ArtifactNotFound`
until a framing refresh runs:

```http
POST https://api.powerbi.com/v1.0/myorg/groups/{ws}/datasets/{id}/refreshes
Body: {"type":"Full"}
```

Status comes back as `refreshType: DirectLakeFraming`, completes in
seconds. Then DAX works.

---

## 7. Fabric / Power BI API audiences

| Operation | `--resource` |
|---|---|
| Items API (semantic models, reports, getDefinition, updateDefinition, jobs) | `https://api.fabric.microsoft.com` |
| Datasets API (refresh, executeQueries, datasources, permissions) | `https://analysis.windows.net/powerbi/api` |
| OneLake DFS (file CRUD via `curl`) | `https://storage.azure.com/` |
| SQL endpoint via pyodbc | `https://database.windows.net/` |

Wrong audience = silent 401. Always pass `--resource` explicitly to
`az rest`.

### Use REST, not the PowerBIQuery MCP tool, for fresh models

The `PowerBIQuery-ExecuteQuery` MCP tool caches the workspace's model
list and can return `ArtifactNotFound` for hours after a deploy, while
`executeQueries` REST API on the same GUID works instantly. **For
freshly deployed models, always go REST.**

---

## 8. Source-vs-model validation

Don't ship without confirming the gold fact matches source totals. The
source lakehouse already has `LineTotal` so the truth query is trivial:

```sql
SELECT SUM(LineTotal) AS Total, COUNT(DISTINCT SalesOrderID) AS Orders
FROM SalesLT.SalesOrderDetail;
-- Expect: 708690.15  /  32
```

Compare against the model via `executeQueries` (`EVALUATE { [Sum Order
Value] }` and `EVALUATE { [Order Count] }`). Delta should be ~0
(floating-point noise only). If the model under-reports, the most
likely cause is an INNER join in gold dropping rows whose dimension
key has no match — switch suspect joins to LEFT and `coalesce` the
display column to `'Unknown'`.

---

## 9. OpenSpec

- `openspec validate --strict` requires **SHALL / MUST in the requirement
  body text**, not just in the heading.
- Every requirement also needs **at least one `#### Scenario:` block**,
  otherwise `--strict` fails with `must include at least one scenario`.
  Add a placeholder scenario as soon as you create the requirement
  heading.
- `openspec init` is **interactive** — in a non-TTY context it blocks on
  a tool-selection prompt. Sending `{enter}` twice picks defaults and
  unblocks.

---

## 10. Squad — use real agents, not narrative role-play

Spin up a real Squad in the project root so the audience sees actual
hand-offs (and you get `squad cost` afterwards):

```powershell
squad init                                 # markdown-only, default
squad hire --name Bob   --role tech-lead
squad hire --name Muck  --role data-engineer
squad hire --name Scoop --role semantic-model-author
squad hire --name Roley --role report-author
squad hire --name Lofty --role spec-author
squad hire --name Dizzy --role devops
```

Open a second terminal during the demo:

```powershell
squad status
Get-Content .squad\orchestration.log -Wait -Tail 20
while ($true) { Clear-Host; squad cost --all; Start-Sleep 5 }
```

Dispatch every plan step to the matching agent; announce hand-offs in
chat (`👷 Bob, take it…` / `✅ done — over to 🚜 Muck`).

---

## 11. IQR outliers — don't chase magic numbers

At order grain (32 rows × 4 categories), `1.5 × IQR` always yields
**0 outliers** because the spread inside each category is too small.

Pick one and move on:

- (a) Compute `IsOutlier` at line grain and aggregate `OutlierLineCount`
  into the fact, OR
- (b) Keep order grain and accept `Outlier Count = 0`.

Validate that the measure **returns a number**, not that it equals a
specific count.

---

## 12. Live-demo time budget (target 30 min total)

| Stage | Owner | Cap |
|---|---|---|
| Pre-flight (auth, repo reset, squad init) | Clawdia | 2 min |
| OpenSpec proposal | 🏎️ Lofty | 2 min |
| Workspace + 3 lakehouses + shortcuts | 👷 Bob | 4 min |
| 3 medallion notebooks authored + run | 🚜 Muck | 8 min |
| TMDL Direct Lake model + framing + DAX validation | 🏗️ Scoop | 7 min |
| PBIR enhanced report deployed | 🚧 Roley | 5 min |
| Final commit & push | 🚒 Dizzy | 2 min |

When a stage hits its cap: **stop polishing and move on**. Audience >
perfection.

---

## 13. Don't try `gh repo delete` in the Copilot CLI shell

`gh auth refresh -h github.com -s delete_repo` triggers a device-flow
login that blocks indefinitely (no browser, no way to enter the device
code). For a clean slate, **reset the repo by force-pushing an orphan
commit** — same effect, no interactive auth:

```powershell
Remove-Item .git -Recurse -Force
git init -b main -q
git add .
git -c user.name='Remc0000' -c user.email='223556219+Copilot@users.noreply.github.com' commit -q -m "chore: reset"
git remote add origin https://github.com/Remc0000/FabricRoadshow.git
git push -f -u origin main
```
