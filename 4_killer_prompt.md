# 🎤 The Fabric Roadshow Killer Prompt

> Copy everything between the `--- BEGIN ---` and `--- END ---` markers,
> paste into GitHub Copilot CLI, hit enter, lean back, enjoy the show.

---

## --- BEGIN ---

Hi Clawdia! 👋 We're live at the **Fabric Roadshow 2026** and you're the demo.
No pressure. The lights are on you. Bob the Builder is watching.

**Don't delete prior demos.** Generate a slug at t = 0 (see §0) and use
slug-suffixed names for the workspace (`Fabric Roadshow_<slug>`) and
GitHub repo (`FabricRoadshow_<slug>`). Saves ~60 s, keeps prior runs
intact for A/B compare, and avoids the device-flow prompt that
`gh repo delete` triggers.

Build me an **end-to-end Microsoft Fabric analytics solution** — from a
source lakehouse to a published Power BI report — and please do it
**without the rookie mistakes you made last time** (yes, I kept notes,
they're in `C:\Users\revandam\GHCP4Fab\3_extra_input.md` — read them first,
I'm begging you).

---

## 0. Pre-flight (one PowerShell block, run once at t = 0)

Run this *first*, before anything else. It sets encoding (avoids the
``[UnexpectedError] charmap`` from ``fab``), generates the slug, prints
the targets, pre-warms the four audience tokens **in parallel**, and
scaffolds the squad. After this block returns, t = 0 starts.

```powershell
[Console]::OutputEncoding = [Text.Encoding]::UTF8
$env:PYTHONIOENCODING     = 'utf-8'

$slug      = "$(Get-Random -Min 1000 -Max 9999)-$((Get-Date).ToString('HHmm'))"
$WORKSPACE = "Fabric Roadshow_$slug"
$REPO      = "FabricRoadshow_$slug"
$REPO_PATH = "C:\Users\revandam\$REPO"

Write-Host "🐾 slug=$slug | ws='$WORKSPACE' | repo=$REPO" -ForegroundColor Cyan

# Pre-warm 4 audience tokens in parallel (≈ 4 s total instead of ~16 s serial)
$audiences = @(
  'https://api.fabric.microsoft.com',
  'https://analysis.windows.net/powerbi/api',
  'https://storage.azure.com/',
  'https://database.windows.net/'
)
$audiences | ForEach-Object -Parallel {
  az account get-access-token --resource $_ --query expiresOn -o tsv | Out-Null
} -ThrottleLimit 4

# Squad scaffold (init works; hire is a stub — author charters by hand)
New-Item -ItemType Directory -Path $REPO_PATH -Force | Out-Null
Set-Location $REPO_PATH
git init -q; squad init | Out-Null
"slug=$slug`nworkspace=$WORKSPACE`nrepo=$REPO" |
  Set-Content .squad\identity\now.md -Encoding UTF8
```

After this block:

- `$WORKSPACE`, `$REPO`, `$slug` are set in the session and printed.
- Four Azure tokens are warm in cache (next REST call is instant).
- `.squad/` is initialised; `now.md` records this run's identity.
- `gh repo create` and `fab create workspace` MUST use `$WORKSPACE` /
  `$REPO`. Quote them — they contain underscores **and** the workspace
  name has a space.

---

## 1. Functional requirements (what we're delivering)

### 1.1 The mission

Analyse **sales orders by Product Category × City × State** with
**Sum / Avg / Max** order value and **IQR-based outlier detection**.
Surface a **Product Category hierarchy** (Top Category → Sub-Category)
the audience can drill through.

### 1.2 The data — single source, full schema

Source is the **`SalesLT` lakehouse** in workspace **`SalesLT`** —
full AdventureWorksLT, 10 base tables under the **`SalesLT` schema**:
`Address`, `Customer`, `CustomerAddress`, `Product`, `ProductCategory`,
`ProductDescription`, `ProductModel`, `ProductModelProductDescription`,
`SalesOrderDetail`, `SalesOrderHeader`. Every column you'd expect is
there: `ProductCategory.Name`, `ProductCategory.ParentProductCategoryID`,
`Address.StateProvince`, `Address.CountryRegion`, `Product.Name`,
`SalesOrderDetail.LineTotal`, `SalesOrderHeader.TotalDue`. **No
workarounds, no redaction, no hardcoded lookups.** Read the data as it is.

### 1.3 Star schema (gold)

| Table | Type | Notes |
|---|---|---|
| `fact_sales_order` | Fact, order grain | One row per `SalesOrderID`; `OrderValue`, `OrderQty`, `IsOutlier`, FK columns |
| `dim_product_category` | Dimension | Carries both `CategoryName` and `TopCategoryName` to drive the hierarchy |
| `dim_geography` | Dimension | City + State |
| `dim_state` | Dimension | State name (from `Address.StateProvince`, **no postal-code derivations**) |
| `dim_date` | Dimension | Standard date dim |

### 1.4 Product Category hierarchy

`ProductCategory` is self-referencing through `ParentProductCategoryID`
(top-level rows have `ParentProductCategoryID IS NULL`). Materialise the
hierarchy in silver by self-joining `ProductCategory`:

```
ProductCategoryID  CategoryName       ParentProductCategoryID  TopCategoryName
1                  Bikes              NULL                     Bikes
5                  Mountain Bikes     1                        Bikes
6                  Road Bikes         1                        Bikes
...
```

Top-level rows get `TopCategoryName = CategoryName` so the column is
never null. Expose it in the semantic model as a 2-level user hierarchy
(see §2.4).

### 1.5 Lineage and audit columns (every layer)

Every table in **every layer** — bronze, silver, gold — **must** carry
these three columns so we can trace any record back to source and tell
when it landed:

| Column | Type | Meaning |
|---|---|---|
| `SourceID` | int | FK into `silver_sources` (registry lives in silver, FK referenced from all layers) |
| `IngestionDate` | timestamp | First-write timestamp (set on insert only) |
| `LastUpdateDt` | timestamp | Last-write timestamp (bumped on every update) |

Plus a dedicated reference table `silver_sources` — registry of every
system feeding the platform:

- `SourceID` (int, PK)
- `SourceName` (string) — friendly name, e.g. `SalesLT`
- `SourceSystem` (string) — kind of system, e.g. `Lakehouse`,
  `Mirrored Azure SQL`, `OneLake Shortcut`
- `SourceLocation` (string) — workspace + lakehouse / table path that
  the data came from
- `IngestionDate`, `LastUpdateDt` — same semantics as on the data tables

For this demo there's exactly one source row, but every fact/dim/bronze
row still has to point at it via `SourceID`. The `silver_sources` table
is the single registry; bronze and gold reference its `SourceID` values
without duplicating the registry. It is **not** surfaced in the
semantic model — audit metadata only.

### 1.6 Incremental load (every layer)

Every layer supports **incremental loads**, not full-refresh. Re-running
the whole pipeline twice in a row must produce zero updates on the
second run. Use the lineage columns as the bookkeeping:

- **Bronze** — MERGE from the source lakehouse shortcuts into bronze
  Delta tables, keyed on each source table's natural PK, watermarked on
  source `ModifiedDate`:
  - `WHEN MATCHED AND source.ModifiedDate > target.LastUpdateDt`
    → update data columns + bump `LastUpdateDt = current_timestamp()`,
    keep `IngestionDate` and `SourceID` unchanged.
  - `WHEN NOT MATCHED` → insert with `IngestionDate = LastUpdateDt =
    current_timestamp()`, stamp `SourceID = 1`.
- **Silver** — MERGE from bronze using the same pattern. Watermark on
  `bronze.LastUpdateDt > silver.LastUpdateDt` so silver only re-runs
  rows that bronze just touched. The materialised
  `silver_product_category` is rebuilt from the (now small) silver
  category table on each run — it's a derivation, not a copy.
  `silver_sources` is upserted on `SourceID` (idempotent).
- **Gold** — MERGE from silver into the star schema. Each dim is keyed
  on its natural key (`ProductCategoryID`, `City+State`, `StateName`,
  date), `fact_sales_order` is keyed on `SalesOrderID`. Watermark on
  `silver.LastUpdateDt > gold.LastUpdateDt`. **Recompute the IQR
  boundaries across the full fact** after each MERGE (IQR is a window
  function over the whole dataset, not per-row), then update only the
  `IsOutlier` column on the affected rows.

The first run is, by definition, "incremental from empty" — every row
is `NOT MATCHED` and gets inserted. Validate by re-running every
notebook back-to-back: each one must report `inserted=0, updated=0` on
the second pass. Print those counters at the end of each notebook so
the audience sees idempotency live.

### 1.7 Nightly orchestration — Data Factory pipeline

IQR rule on `OrderValue`, computed at order grain or line grain — your
call. At order grain (32 rows × 4 categories), `1.5 × IQR` will yield
**0 outliers**. Don't chase a magic number; pick one approach and
validate that the measure returns a number.

### 1.9 The deliverable

- GitHub repo `Remc0000/FabricRoadshow` containing the OpenSpec, the
  3 medallion notebooks, the TMDL semantic model, the PBIR report,
  and the `nightly_refresh` pipeline definition, all committed as
  `Remc0000`.
- Fabric workspace `Fabric Roadshow` on `Trial-Remco` capacity, holding
  3 lakehouses (`bronze_lh`, `silver_lh`, `gold_lh`), a Direct Lake
  semantic model `OrdersAnalytics`, a published report
  `OrdersAnalytics_Report`, and a scheduled pipeline `nightly_refresh`.
- Source-vs-model validation: model `[Sum Order Value] ≈ $708,690.15`
  and `[Order Count] = 32`, matching `SUM(SalesLT.SalesOrderDetail.LineTotal)`
  in source.
- Idempotency check: the silver notebook re-run reports `inserted=0,
  updated=0` when source hasn't changed.

---

## 2. Technical requirements (how to build it)

### 2.1 Tooling

- **`fab` CLI** for workspace + items administration
- **`skills-for-fabric`** (microsoft) — already installed at
  `~/.copilot/installed-plugins/fabric-collection/skills-for-fabric`
- **`powerbi-agentic-plugins`** (RuiRomano) — `powerbi@powerbi-agentic-plugins`
  and `fabric@powerbi-agentic-plugins`. Already installed. Use these
  for ALL TMDL semantic-model work, PBIR report authoring, and DAX.
- Identity for every commit: **`Remc0000`** I'm watching. 👀
  ```
  git -c user.name='Remc0000' -c user.email='223556219+Copilot@users.noreply.github.com' commit ...
  ```

### 2.1.1 Skill-and-agent matrix — NON-NEGOTIABLE

This is the part Clawdia skipped last run. **Every stage MUST be
driven through the matching skill or agent**, not by hand-rolling REST
calls or one-off scripts. Hand-rolling is faster in the moment and
costs you the documented best practices the skills exist to enforce.

| Stage | Owner | MUST invoke (in this order, fall back only if first fails) |
|---|---|---|
| Pre-flight, workspace, capacity, lakehouses, shortcuts | 👷 Bob | 1. ``skills-for-fabric:FabricAdmin`` agent for workspace + capacity assignment <br>2. ``fabric-mcp-server-core_create-item`` for each lakehouse (with ``enableSchemas: true``) <br>3. ``fabric-mcp-server-onelake_*`` for shortcut creation and verification <br>4. ``fab`` CLI only as fallback when an MCP tool errors |
| Medallion notebook authoring + run | 🚜 Muck | 1. **Invoke the ``e2e-medallion-architecture`` skill FIRST** to get the canonical Bronze/Silver/Gold scaffold and Spark configs per layer <br>2. ``skills-for-fabric:FabricDataEngineer`` agent to orchestrate the cross-workload flow <br>3. ``spark-authoring-cli`` skill for notebook deploy + run <br>4. Hand-rolled REST only if all of the above fail |
| TMDL Direct Lake semantic model | 🏗️ Scoop | 1. Delegate to the ``powerbi:powerbi-developer`` agent (RuiRomano) <br>2. That agent drives the ``powerbi-modeling-mcp-*`` tools (model_operations, table_operations, relationship_operations, measure_operations, user_hierarchy_operations) <br>3. ``powerbi-semantic-model-authoring`` skill for any gaps |
| DAX validation | 🏗️ Scoop | 1. ``PowerBIQuery-ExecuteQuery`` for one-off reads (note: caches model lists for fresh deploys — use REST ``executeQueries`` if model just deployed) <br>2. ``powerbi-modeling-mcp-dax_query_operations`` for validate-and-execute |
| PBIR enhanced report | 🚧 Roley | 1. Delegate to the ``powerbi:powerbi-developer`` agent <br>2. That agent uses the ``powerbi-report-authoring`` skill (RuiRomano patterns) <br>3. Deploy via ``fabric-mcp-server-core_create-item`` |
| OpenSpec proposal | 🏎️ Lofty | 1. ``openspec`` CLI directly (no skill wrapper exists) — but lean on the ``microsoft-skill-creator`` patterns for SHALL/MUST wording and ``#### Scenario:`` blocks |
| Pipeline + schedule | 🚒 Dizzy | 1. ``skills-for-fabric:FabricDataEngineer`` to author the chain <br>2. ``fabric-mcp-server-core_create-item`` (type=DataPipeline) <br>3. REST for ``/jobs/Pipeline/schedules`` (no MCP wrapper) |

**Audit rule** — at the finish flag, the agent MUST list which skills
and MCP tool families were actually called. If the only entries are
``fab`` and ``urllib.request``, that's a fail — re-run.

**Forbidden pattern**: hand-writing ``urllib.request`` deploy scripts
when ``fabric-mcp-server-core_create-item`` exists. The MCP tool
handles auth, polling, retries, and emits structured results — your
hand-rolled retry helper does not.

**The skills are not optional decoration.** They encode the lessons
from prior demos. Skipping them is how Clawdia ends up re-discovering
the schema-lowercase bug or the ``/v1/`` path or the Windows TZ name
on stage. Use the skills, save the surprises for the audience.

### 2.2 Workspace + lakehouses (👷 Bob)

- `fab` create workspace **`Fabric Roadshow`**, assign to `Trial-Remco`.
- Create 3 lakehouses: `bronze_lh`, `silver_lh`, `gold_lh`.
- In `bronze_lh`, create **OneLake shortcuts** to the `SalesLT` lakehouse
  Delta tables under `Tables/SalesLT/<TableName>` for the 10 base tables.
  Views (`vGetAllCategories`, etc.) are not required.

### 2.3 Three-notebook medallion (🚜 Muck)

This is the **`e2e-medallion-architecture`** skill standard — please
use it. **THREE separate notebooks**, not one mega-notebook:

- **`01_bronze_ingest.Notebook`** (default lakehouse `bronze_lh`)
  - Source shortcuts under `Tables/SalesLT/<TableName>` are the input.
  - **MERGE-load** each source table into a bronze Delta table keyed on
    the source's natural PK, watermarked on `source.ModifiedDate >
    target.LastUpdateDt` (see §1.6). Stamp every bronze table with
    `SourceID`, `IngestionDate`, `LastUpdateDt` (§1.5).
  - Schema preserved; only light typing. Print `inserted=…, updated=…`
    counters per table.

- **`02_silver_clean.Notebook`** (default lakehouse `silver_lh`)
  - Read from bronze (NOT from the source shortcuts — bronze is the
    contract).
  - Clean + dedupe; project the columns each downstream layer needs.
  - **Read every name, code, and amount straight from bronze** —
    `ProductCategory.Name`, `Address.StateProvince`,
    `SalesOrderDetail.LineTotal`, etc. No derivations of values that
    already exist in source.
  - **Build `silver_sources`** with one row for the `SalesLT` lakehouse
    (`SourceID = 1`, `SourceName = 'SalesLT'`,
    `SourceSystem = 'Lakehouse'`, `SourceLocation` =
    `SalesLT/SalesLT.Lakehouse/Tables/SalesLT`, plus the audit
    timestamps). MERGE on `SourceID` so re-runs are idempotent.
  - **MERGE-load** each cleaned table from bronze, keyed on natural PK,
    watermarked on `bronze.LastUpdateDt > silver.LastUpdateDt`. Stamp
    every silver table with `SourceID`, `IngestionDate`, `LastUpdateDt`
    (§1.5).
  - Materialise `silver_product_category` by self-joining the cleaned
    `silver_product_category_raw` (or whatever you name the cleaned
    `ProductCategory`) on `ParentProductCategoryID = ProductCategoryID`,
    keeping both `CategoryName` and `TopCategoryName` (top-level rows
    coalesce `TopCategoryName = CategoryName`). Rebuild it on each run
    — it's a small derivation.
  - Print `inserted=…, updated=…` counters per table.

- **`03_gold_star.Notebook`** (default lakehouse `gold_lh`)
  - Build the star schema (`fact_sales_order`, `dim_product_category`,
    `dim_geography`, `dim_state`, `dim_date`).
  - `dim_product_category` carries `CategoryName` AND `TopCategoryName`
    so the model can hang the Top → Sub hierarchy off it.
  - `dim_state` comes from `Address.StateProvince` (no postal-code hacks).
  - **MERGE-load** every dim and the fact, keyed on natural keys
    (`ProductCategoryID`, `City+State`, `StateName`, date,
    `SalesOrderID`), watermarked on `silver.LastUpdateDt >
    gold.LastUpdateDt` (§1.6). Carry `SourceID`, `IngestionDate`,
    `LastUpdateDt` end-to-end. Do NOT surface `silver_sources` in
    gold or in the semantic model — it's audit metadata.
  - After the fact MERGE, **recompute IQR boundaries across the full
    fact** and update `IsOutlier` (IQR is a dataset-wide window
    function; per-row recompute is wrong).
  - Validate totals with `print()` checkpoints; print `inserted=…,
    updated=…` counters. Write to `gold_lh.Tables.*`.

For each notebook: author as `.ipynb` (NOT cell-delimited `.py` — CRLF
round-trip breaks the parser). Save under
`fabric/notebooks/<nn>_<name>.Notebook/`. Deploy via REST
`POST /v1/workspaces/{ws}/items` — `fab import` CRASHES inside this CLI
session (`No Windows console found`). Run them in order via the Items
jobs API (Bronze → Silver → Gold) and gate each step on the previous
job reporting `Completed`.

### 2.4 Direct Lake semantic model (🏗️ Scoop)

Author **TMDL** for `OrdersAnalytics`. Rules of the house:

- **TABS, not spaces.** Verify 0x09. I'll know.
- No manual `lineageTag` on new objects.
- Relationships use `fromColumn` / `toColumn`, **not** `fromTable`.
- Display names with spaces (`'Category Name'`, `'Top Category Name'`,
  `'State Name'`) — visuals and DAX reference the **display name**, not
  `sourceColumn`. Get this wrong and 11 visuals scream "Can't display".
- On `dim_product_category`, define a **user hierarchy**:
  ```
  hierarchy 'Category Hierarchy'
      level 'Top Category Name'
          column: 'Top Category Name'
      level 'Category Name'
          column: 'Category Name'
  ```
  so visuals can drill from top categories down to leaf categories
  without DAX gymnastics.
- Hide the lineage columns (`SourceID`, `IngestionDate`, `LastUpdateDt`)
  from the report layer (`isHidden: true`) — they ride along for audit
  but shouldn't clutter the field list.

Deploy via REST `POST /v1/workspaces/{ws}/items` (NOT `fab import`).
Use the **SQL endpoint GUID**, not the lakehouse GUID, in
`Sql.Database()` — wrong one fails framing. Trigger a Direct Lake
**framing refresh** (`POST .../refreshes {"type":"Full"}`) before any
DAX. Validate via REST `executeQueries` (NOT the PowerBIQuery MCP tool —
it caches stale model lists): total ≈ **$708,690**, **32 orders**, and
the hierarchy returns values at both levels.

### 2.5 PBIR enhanced report (🚧 Roley)

Use the RuiRomano `powerbi@powerbi-agentic-plugins` plugin. Its
templates target the schemas Fabric service expects:

- `definition.pbir` → `definitionProperties/2.0.0`, `version: "4.0"`
- `definition/version.json` → `versionMetadata/1.0.0`, `"2.0.0"` ⚠️ required
- `definition/report.json` → `report/3.0.0`, object `reportVersionAtImport`
- `definition/pages/<id>/visuals/<n>/visual.json` → `visualContainer/2.4.0`
- `byConnection` with `connectionString` containing `semanticModelId=<guid>`
- **Every projection needs `nativeQueryRef`** or visuals go dark.
- Visual types: `clusteredBarChart`, `clusteredColumnChart`, `donutChart`,
  `card`, `tableEx`, `slicer`. (Not `barChart`. Don't try.)
- Include at least **one matrix or drill-down bar** that uses the
  `Category Hierarchy` so the audience sees the Top → Sub drill.

### 2.6 OpenSpec (🏎️ Lofty)

Write a proposal for `add-orders-analytics`, validate `--strict`
(SHALL/MUST goes in the requirement **body**, not just the heading;
every requirement needs at least one `#### Scenario:` block), commit
to a fresh GitHub repo `FabricRoadshow`.

### 2.7 Nightly pipeline (🚒 Dizzy)

Author a Fabric Data Pipeline `nightly_refresh` (Items API
`type: DataPipeline`) with three Notebook activities chained on success:
`Bronze → Silver → Gold`. Add a final Web activity (or notebook cell)
that POSTs `{"type":"Full"}` to the `OrdersAnalytics` model's
`/refreshes` endpoint so Direct Lake re-frames after gold completes.
Attach a **daily schedule** at 02:00 Europe/Amsterdam.

Save the pipeline definition under
`fabric/pipelines/nightly_refresh.DataPipeline/` (mirror the platform
+ pipeline-content layout used by the notebooks). Deploy via REST
`POST /v1/workspaces/{ws}/items` with `type: DataPipeline` — same
`definition.parts` envelope as notebooks, `payloadType: InlineBase64`.

### 2.8 Final commit & push (🚒 Dizzy)

Push everything to `https://github.com/Remc0000/FabricRoadshow` as
`Remc0000`.

### 2.9 API audiences (memorise — do not mix)

| Operation | Audience |
|---|---|
| Items API (semantic models, reports, getDefinition, jobs) | `https://api.fabric.microsoft.com` |
| Datasets API (refresh, executeQueries) | `https://analysis.windows.net/powerbi/api` |
| OneLake DFS | `https://storage.azure.com/` |
| SQL endpoint via pyodbc | `https://database.windows.net/` |

Wrong audience = silent 401.

### 2.10 Fabric REST URL pin

The Fabric Items API base path is **`/v1/`**, not `/v1.0/`. Last run
the pipeline deploy 404'd because `/v1.0/` was used. Same convention
holds for `getDefinition`, `updateDefinition`, `jobs/instances`,
`shortcuts`, and `jobs/Pipeline/schedules`. The Power BI Datasets API
base path is `https://api.powerbi.com/v1.0/myorg/...` — that *is*
`v1.0`. Mix them up and you'll lose 5 minutes to confusion.

### 2.11 Schedule body — Windows time-zone names

The pipeline schedule endpoint
`POST /v1/workspaces/{ws}/items/{pid}/jobs/Pipeline/schedules` requires
a **Windows time-zone identifier**, not an IANA one:

```json
{
  "enabled": true,
  "configuration": {
    "type": "Cron",
    "interval": 1440,
    "startDateTime": "<future-ISO-without-Z>",
    "endDateTime":   "2099-12-31T23:59:59",
    "localTimeZoneId": "W. Europe Standard Time"
  }
}
```

`Europe/Amsterdam` will be rejected. `interval` is in **minutes**
(1440 = daily). `startDateTime` must be a future timestamp, no `Z`,
no offset.

### 2.12 Shared helpers module — write once, reuse three times

Last run I re-implemented `request()` / `token()` / `b64()` three
times (notebook deploy, pipeline deploy, schedule). Don't. Drop a
single ``scripts/_fabric.py`` first; every later script imports it.

```python
# scripts/_fabric.py — single source of truth for REST plumbing
import base64, json, os, ssl, subprocess, sys, time
import urllib.request, urllib.error

_TOKENS = {}        # audience -> (token, expires_epoch)

def token(aud="https://api.fabric.microsoft.com"):
    t = _TOKENS.get(aud)
    if t and t[1] - time.time() > 120:
        return t[0]
    out = subprocess.run(
        f'az account get-access-token --resource {aud}',
        capture_output=True, text=True, check=True, shell=True)
    j = json.loads(out.stdout)
    exp = time.mktime(time.strptime(j["expiresOn"], "%Y-%m-%d %H:%M:%S.%f"))
    _TOKENS[aud] = (j["accessToken"], exp)
    return j["accessToken"]

def request(method, url, body=None, aud="https://api.fabric.microsoft.com",
            extra_headers=None, timeout=120):
    headers = {"Authorization": f"Bearer {token(aud)}",
               "Content-Type": "application/json"}
    if extra_headers: headers.update(extra_headers)
    data = json.dumps(body).encode() if body is not None else None
    req = urllib.request.Request(url, data=data, headers=headers, method=method)
    ctx = ssl.create_default_context()
    last = None
    for attempt in range(3):
        try:
            with urllib.request.urlopen(req, timeout=timeout, context=ctx) as r:
                return r.status, dict(r.headers), r.read().decode()
        except urllib.error.HTTPError as e:
            return e.code, dict(e.headers or {}), e.read().decode()
        except Exception as e:
            last = e; time.sleep(2 ** attempt)
    raise last

def b64_file(path):
    with open(path, "rb") as f:
        return base64.b64encode(f.read()).decode()

def wait_op(op_url, aud="https://api.fabric.microsoft.com", every=3):
    while True:
        s, h, t = request("GET", op_url, aud=aud)
        body = json.loads(t) if t else {}
        st = body.get("status", "?")
        if st in ("Succeeded", "Failed"): return s, body
        time.sleep(every)
```

Subprocess pin: `az` from Python on Windows **needs `shell=True`** —
the bare-list form fails with `FileNotFoundError [WinError 2]`.

---

## 3. The squad — use real agents, **and** use them in parallel

| Agent | Role | Tools |
|---|---|---|
| 👷 **Bob** | Tech lead, orchestrates the build | skills-for-fabric, fab CLI |
| 🚜 **Muck** | Bulldozes data through bronze/silver/gold | PySpark notebooks, skills-for-fabric |
| 🏗️ **Scoop** | Scoops up the semantic model in TMDL | RuiRomano `powerbi` plugin |
| 🚧 **Roley** | Rolls out the Power BI report (PBIR) | RuiRomano `powerbi` plugin |
| 🏎️ **Lofty** | Lifts heavy specs, writes OpenSpec | openspec |
| 🚒 **Dizzy** | Mixes everything in CI / GitHub | gh CLI |

### 3.1 What squad v0.9.1 actually does (read this first)

Reality check from the last run: **`squad hire` is a stub** — it prints
`(full implementation pending)` and exits. So Clawdia, do NOT skip this
section by typing `squad hire` six times and assuming agents got hired.
They didn't. Last time I narrated hand-offs but only fired two real
parallel background agents and serialised the rest. Don't repeat that.

What works in v0.9.1 today:

| Command | Status |
|---|---|
| `squad init` | ✅ creates `.squad/` with templates, identity, log, config |
| `squad roles` | ✅ list built-in role catalogue (use as charter inspiration) |
| `squad cast` | ✅ shows current session cast |
| `squad cost --all` | ✅ token-usage breakdown — perfect for the second screen |
| `squad triage` / `squad loop` | ✅ scan for work, Ralph-mode work loop |
| `squad delegate <squad> <desc>` | ✅ create work in another squad |
| `squad status` / `squad doctor` | ✅ diagnostics |
| `squad hire` | 🟥 **stub** — prints "implementation pending" |

### 3.2 Realistic squad recipe (do this, not the stub)

```powershell
# 1. Bootstrap (always works)
squad init

# 2. Hand-author charters because hire is stubbed.
#    One folder per persona, identical to what hire would produce.
$personas = @(
  @{name='Bob';   role='tech-lead';              emoji='👷'},
  @{name='Muck';  role='data-engineer';          emoji='🚜'},
  @{name='Scoop'; role='semantic-model-author';  emoji='🏗️'},
  @{name='Roley'; role='report-author';          emoji='🚧'},
  @{name='Lofty'; role='spec-author';            emoji='🏎️'},
  @{name='Dizzy'; role='devops';                 emoji='🚒'}
)
foreach ($p in $personas) {
  $dir = ".squad/agents/$($p.name.ToLower())"
  New-Item -ItemType Directory -Path $dir -Force | Out-Null
  @"
# $($p.emoji) $($p.name) — $($p.role)

Charter: see role catalogue (`squad roles --search $($p.role)`).
Owns: <fill from §2 of killer prompt>
Reports to: 👷 Bob.
"@ | Set-Content "$dir/charter.md" -Encoding UTF8
  "" | Set-Content "$dir/history.md" -Encoding UTF8
}
```

### 3.3 Make the agents *real* — fan out background `task` agents

Every persona MUST be driven by a real **background `task` agent** that
reads its charter and reports back. Drop the narrative-only hand-offs.

**Critical parallelism rule** — last run cost 70+ min by serialising.
This run, fire **multiple background `task` agents in ONE tool call**
at every fan-out point:

- **t = 0** (before any Fabric work): fire 3 background agents
  simultaneously — preflight cleanup (Clawdia direct), 🏎️ **Lofty**
  (OpenSpec), 👷 **Bob** (workspace + 3 lakehouses + 10 shortcuts).
- **As soon as 🚜 Muck's gold notebook reports `Completed`**: fire
  3 more background agents in ONE tool call — 🏗️ **Scoop** (TMDL),
  🚧 **Roley** (PBIR shell, with placeholder `semanticModelId` to be
  swapped at deploy), 🚒 **Dizzy** (pipeline JSON, model ID
  placeholder). NEVER wait for Scoop before launching Roley/Dizzy —
  they're independent except for a 30-second model-ID swap at the
  very end.
- The two long poles in the run are Scoop (~12 min, REST polling for
  async deploy + framing + DAX) and Roley (~20 min, lots of visual
  JSON). They MUST overlap or you blow past 30 min.

Each background agent's prompt MUST end with: *"Append a line to
`.squad/orchestration.log` when you start and when you finish: format
`<ISO-timestamp> <emoji> <name> <state> <one-line-summary>`."* That
log file is what powers the second-window tail.

### 3.4 Don't destroy — version with a unique slug

Last run wasted ~60s deleting the `Fabric Roadshow` workspace and
force-pushing the GitHub repo to an empty orphan, plus risked the
device-flow prompt that `gh repo delete` triggers. Don't.

At t = 0 generate a short slug (`brave-otter`, `2026-05-09-1042`, etc.)
and use it everywhere:

```powershell
$slug = "$(Get-Random -Minimum 1000 -Maximum 9999)-$((Get-Date).ToString('HHmm'))"
$WORKSPACE = "Fabric Roadshow_$slug"
$REPO      = "FabricRoadshow_$slug"
"slug=$slug`nworkspace=$WORKSPACE`nrepo=$REPO" |
  Set-Content .squad\identity\now.md -Encoding UTF8
```

Print the slug + workspace + repo URL **at the very start**. Prior
demos stay intact for A/B compare. No destructive cleanup, no prompts.

### 3.5 Audience-facing live screen

Open a second PowerShell window so the audience sees real squad
activity (not just chat narration):

```powershell
# Pane 1: live orchestration log (every agent appends start/finish)
Get-Content .squad\orchestration.log -Wait -Tail 30

# Pane 2: live token spend per agent
while ($true) { Clear-Host; squad cost --all; Start-Sleep 5 }

# Pane 3: which charter / history files just got touched
while ($true) {
  Clear-Host
  Get-ChildItem -Recurse -Force -Filter "*.md" .squad\agents |
    Select-Object FullName, LastWriteTime |
    Sort-Object LastWriteTime -Descending | Select-Object -First 10
  Start-Sleep 5
}
```

Announce hand-offs in chat (`👷 Bob, take it…` / `✅ Bob done — over to
🚜 Muck`). Combined with the three live panes, the audience sees both
narration and proof — and because every persona is a real background
`task` agent appending to the log, the proof is real too.

---

## 4. Time budget per stage (target total ≤ 35 min)

| Stage | Owner | Max time |
|---|---|---|
| Pre-flight (auth, repo reset, squad init) | Clawdia | 2 min |
| OpenSpec proposal | 🏎️ Lofty | 2 min |
| Workspace + 3 lakehouses + shortcuts | 👷 Bob | 4 min |
| 3 medallion notebooks (Bronze/Silver/Gold) authored + run | 🚜 Muck | 9 min |
| TMDL Direct Lake model + framing + DAX validation | 🏗️ Scoop | 7 min |
| PBIR enhanced report deployed | 🚧 Roley | 5 min |
| `nightly_refresh` pipeline + schedule | 🚒 Dizzy | 4 min |
| Final commit & push | 🚒 Dizzy | 2 min |
| **Total** | | **35 min** |

If you hit the cap on any stage, **stop polishing** and move on with
what works. Audience > perfection.

---

## 5. Showmanship rules

- **Announce each agent** before they work. ("🚜 Muck rolling in…")
- When something fails, **don't silently retry 14 times**. Tell us, pivot,
  move on. The audience is here for the journey, not the stack traces.
- **No PowerBI MCP cache traps** — use the `executeQueries` REST API for
  fresh Direct Lake models.

---

## 5.5 Smoothness checklist & abort criteria

If by minute X the milestone hasn't fired, **pivot — don't keep
polishing**:

| By minute | Milestone | If missing → |
|---|---|---|
| 2 | Pre-flight green, slug printed, 4 tokens warm | abort, fix auth |
| 6 | Workspace + 3 schema-enabled lakehouses + 10 shortcuts visible | skip shortcuts, hard-code ABFSS paths |
| 10 | Bronze notebook deployed (run can finish later) | drop to single-table demo |
| 18 | Gold notebook `Completed` ($708,690.15 / 32) | use seed data, skip outlier IQR |
| 22 | TMDL deployed + framing returned 202 | skip hierarchy drill, single bar chart |
| 28 | PBIR report `Succeeded` | swap to a 1-card report, keep moving |
| 32 | Pipeline + schedule created | print schedule cron in chat as backup |
| 35 | Final commit + push | apologise on stage and finish flag anyway 🏁 |

**Three things I will check on you live:**

1. ``.squad/orchestration.log`` has ≥ 12 timestamped lines (start +
   finish per persona). If it has ≤ 6, the squad framing is cosmetic.
2. The fan-out at t = 0 is **one tool call** firing 3 background
   ``task`` agents (preflight, Lofty, Bob). Same at gold-Completed
   (Scoop, Roley, Dizzy). Sequential is a fail.
3. Workspace + repo carry the slug. Anything called plain
   ``Fabric Roadshow`` / ``FabricRoadshow`` means you ignored §0.

---

## 6. The finish line

When done, print:
1. An **ASCII finish flag** 🏁
2. The **GitHub Copilot banner** in ASCII
3. The **total elapsed time** (target: **Max** `30min` otherwise people are already gone)
4. Links to: the GitHub repo, the workspace, the published report

If anything goes sideways live on stage, take a breath, blame Bob, and
keep building. **Can we fix it? Yes we can.** 🔨

— Remco

## --- END ---

---

## How to use this in the demo

1. Open GitHub Copilot CLI in `C:\Users\revandam\`
2. Paste the block between `--- BEGIN ---` and `--- END ---`
3. Pretend to sip coffee while the squad does the work
4. Take credit at the finish flag 🏁
