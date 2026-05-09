# 3 — Missing in Skills

> Cross-reference of every skill I (Clawdia) actually invoked during the Fabric Roadshow run, and the **extra context I had to discover the hard way** because the skill itself didn't spell it out. Use this when you (future you, or the next Copilot run) want to know which gotchas already happened so you don't burn another 2 hours rediscovering them.
>
> Skills referenced are the ones in `C:\Users\revandam\.agents\skills\` — Microsoft `skills-for-fabric` family + RuiRomano's PowerBI skills + the OpenSpec / Squad CLIs.

---

## 1. `e2e-medallion-architecture`

The skill nails Bronze/Silver/Gold theory and points at `notebook-api-operations.md` for `.ipynb` validation. Real-run gaps:

- **Schema-enabled lakehouses change every relative path.** When the lakehouse has schemas turned on, Delta tables live under `Tables/dbo/<name>` not `Tables/<name>`. The skill mentions schemas exist but never says *"if you used schema-enabled, ABFSS reads must inject `/dbo/`"*. Cost me one full Silver re-run.
- **Spark cannot fetch from arbitrary HTTPS.** Buried, but should be a sticky note: stage to `Files/` with `notebookutils.fs.cp` from the source lakehouse first. AdventureWorks shortcut → cross-lakehouse copy is the cleanest path; the skill could show that pattern.
- **Default-lakehouse binding requires *both* id and name** in `metadata.dependencies.lakehouse` of the `.ipynb` *and* in the `executionData.configuration.defaultLakehouse` of `RunNotebook`. The skill says "set lakehouse" — neither shape is shown side by side.
- **Run order discipline.** The skill says "Bronze → Silver → Gold sequentially." It should also say *poll each LRO until `status=Completed` (not just submitted) before kicking the next one* — otherwise you happily get `[TABLE OR VIEW NOT FOUND]` because Silver fired before Bronze finished writing.

## 2. `spark-authoring-cli` / `notebook-api-operations.md`

- **`outputs: []` and `execution_count: null` on every code cell** is in the skill. ✅ kept that.
- **What's missing:** the `.platform` file is *also* required next to the `.ipynb` in the item folder, with `metadata.type=Notebook` and a stable `logicalId`. Without it, `updateDefinition` accepts the payload but the notebook silently has no display name in the workspace tree.
- The skill never tells you that `RunNotebook` returns `202 Accepted` with `Location` header to poll, but the *body* is empty — your code must read the header, not parse the body.

## 3. `powerbi-authoring-cli` (Direct Lake / TMDL)

This is where the most pain lived.

- **TABS vs spaces.** TMDL parser is whitespace-sensitive. Mixing two spaces with a tab character anywhere in a `column` block silently drops the column. Skill says "TMDL is YAML-like" — that is misleading; YAML accepts spaces, TMDL does not.
- **`Sql.Database()` second argument is the SQL endpoint GUID, not the lakehouse GUID.** The skill calls it "the SQL endpoint connection string" without showing where to find the GUID (`GET /lakehouses/{id}` → `properties.sqlEndpointProperties.id`). I lost ~15 min on a 401 because I pasted the lakehouse id.
- **No manual `lineageTag`.** If you write your own TMDL, leave `lineageTag` out — Fabric assigns one on first deploy. Adding one yourself causes "lineage tag already exists" on second deploy.
- **Direct Lake framing is async and silent.** After deploying the model you must POST `…/refreshes { "type": "full", "objects": [...] }` and *poll* — `executeQueries` will return empty rows for ~30s after deploy and there is no error message. Skill should say: "framing is mandatory before validation, allow 60s."
- **Display names with spaces** must be in TMDL `name: 'Sum Order Value'` AND surface identically in PBIR — never use `sourceColumn` to bridge a rename, the model will validate but the report will throw `Cannot find column`.

## 4. `powerbi-report-authoring` (PBIR)

> ⚠️ **Run 0054-1902 corrections:** earlier guidance in this section was wrong on almost every shape. The text below is the *verified-working* version after deploying a 2-page report with 6 visuals. The golden references are:
>
> - **Shell shapes (`.platform`, `definition.pbir`, `version.json`, `report.json`, `pages.json`, `page.json`)** → `microsoft/semantic-link-labs/src/sempy_labs/report/_bpareporttemplate/`
> - **`visual.json` shapes** → `ibarrau/PowerBi-code/PowerBi Reports/Pbip Pruebas/AnalisisAutomotor.Report/`

### Deploy in two passes, not one

The path that actually works is:

1. **Pass 1 — empty shell** (no `visuals/` folder). 5 files: `.platform`, `definition.pbir`, `definition/version.json`, `definition/report.json`, `definition/pages/pages.json` + one `page.json` per page. POST to `/v1/workspaces/{ws}/items` with `type=Report`. This locks in the report id and proves your schema URLs.
2. **Pass 2 — visuals** via `POST /v1/workspaces/{ws}/items/{id}/updateDefinition`. Send the full definition (including the shell files again) plus all `visuals/<name>/visual.json` files. This is where you'll see real-world failures cheaply, because the report already exists.

Trying to deploy everything in one POST means every visual schema bug looks identical to a shell schema bug, and you burn LRO round-trips bisecting.

### Correct schema URLs (copy these verbatim)

| File | `$schema` URL | Value |
|---|---|---|
| `definition/version.json` | `…/fabric/item/report/definition/versionMetadata/1.0.0/schema.json` | `"version": "2.0.0"` (must match `^[1-9]\d*\.(0\|[1-9]\d*)\.0$` — semver ending in `.0`) |
| `definition/report.json` | `…/fabric/item/report/definition/report/1.1.0/schema.json` | object-shaped (see below) |
| `definition/pages/pages.json` | `…/fabric/item/report/definition/pagesMetadata/1.0.0/schema.json` | `pageOrder` + `activePageName` |
| `definition/pages/<page>/page.json` | `…/fabric/item/report/definition/page/1.1.0/schema.json` | `displayOption` is **string** `"FitToPage"`, not number `1` |
| `definition/pages/<page>/visuals/<v>/visual.json` | `…/fabric/item/report/definition/visualContainer/`**`1.4.0`**`/schema.json` | NOT `1.0.0` — the older version silently rejects half the visualTypes |
| `definition.pbir` | **no `$schema`** | uses `version: "4.0"` |

### `definition.pbir` — the connection block (Direct Lake)

Earlier guidance said "byConnection only allows connectionString" — that is **wrong**. The shape that works for a Direct Lake report bound to a deployed semantic model:

```json
{
  "version": "4.0",
  "datasetReference": {
    "byPath": null,
    "byConnection": {
      "connectionString": null,
      "pbiServiceModelId": null,
      "pbiModelVirtualServerName": "sobe_wowvirtualserver",
      "pbiModelDatabaseName": "<SEMANTIC_MODEL_ID_GUID>",
      "name": "EntityDataSource",
      "connectionType": "pbiServiceXmlaStyleLive"
    }
  }
}
```

The model id goes in `pbiModelDatabaseName`. `pbiServiceModelId` stays `null`. `pbiModelVirtualServerName` is the literal string `"sobe_wowvirtualserver"`.

### `report.json` — object-shaped, no stringified `config`

The "stringified `config` blob" pattern is dead. Real shape (post-`1.1.0`):

```json
{
  "$schema": ".../report/1.1.0/schema.json",
  "themeCollection": {
    "baseTheme": {
      "name": "CY24SU06",
      "reportVersionAtImport": "5.56",
      "type": "SharedResources"
    }
  },
  "layoutOptimization": "None",
  "resourcePackages": [
    { "name": "SharedResources", "type": "SharedResources",
      "items": [{ "name": "CY24SU06", "path": "BaseThemes/CY24SU06.json", "type": "BaseTheme" }] }
  ],
  "settings": {
    "useStylableVisualContainerHeader": true,
    "defaultDrillFilterOtherVisuals": true,
    "allowChangeFilterTypes": true,
    "useEnhancedTooltips": true,
    "useDefaultAggregateDisplayName": true
  }
}
```

- `reportVersionAtImport` is a **single string** (e.g. `"5.56"`) nested inside `themeCollection.baseTheme`. The earlier "three-string root object" claim was wrong.
- The `CY24SU06` theme file must actually exist at `StaticResources/SharedResources/BaseThemes/CY24SU06.json`. Grab the bytes from the sempy-labs template — it's ~13KB.

### `visual.json` — shape that works

```json
{
  "$schema": ".../visualContainer/1.4.0/schema.json",
  "name": "<unique-folder-name>",
  "position": { "x": 20, "y": 20, "z": 1000, "width": 260, "height": 120, "tabOrder": 1000 },
  "visual": {
    "visualType": "card",
    "query": {
      "queryState": {
        "Values": {
          "projections": [{
            "field": { "Measure": {
              "Expression": { "SourceRef": { "Entity": "fact_sales_order" } },
              "Property": "Sum Order Value"
            }},
            "queryRef": "fact_sales_order.Sum Order Value"
          }]
        }
      }
    }
  }
}
```

- **`nativeQueryRef` is OPTIONAL.** Earlier guidance said it was required — false. Real samples (e.g. `ibarrau/PowerBi-code`) omit it. Adding it doesn't break, omitting it doesn't break.
- For a hierarchy level use `field.HierarchyLevel.Expression.Hierarchy` with `Hierarchy: "Category Hierarchy"` and `Level: "Master Category"` — the names must match what's in TMDL.
- `clusteredBarChart` / `clusteredColumnChart` / `donutChart` / `slicer` / `pivotTable` / `card` / `filledMap` are the verified visualTypes for a basic dashboard.

### Sanity-check column / measure names BEFORE writing visuals

Reading the local TMDL is faster and more reliable than `INFO.COLUMNS()` over `executeQueries` (which fails on Direct Lake until framing settles). Specifically check for:

- Spaces in column names (must be quoted in TMDL: `column 'Product Name'`).
- Hierarchy-level names (these are the *level* names, not the column names).
- Measure casing.

A wrong column name = silently empty visual, not an error. Catch it locally.

### What's still true from the old guidance

- `definition.pbir` is in the *root* of the report folder, not under `definition/` — yes, two namespaces in one item.
- `clusteredBarChart` not `barChart`. ✅
- Forbidden top-level keys in `page.json` — `config`, `ordinal`. ✅ (`filters` is fine to omit, no need to send `[]`.)
- Page `displayName` is the friendly name; `name` is the URL slug.

## 5. `skills-for-fabric:FabricAdmin` (Bob's tools)

- **OneLake shortcuts** to AdventureWorks: the skill shows the POST shape but not that the **target lakehouse must already have schemas enabled at create time** for shortcuts to land under `Tables/dbo/...`. Switching to schema mode after the fact does not migrate existing shortcuts.
- **Capacity must be `Active`** before workspace assignment will succeed. The skill assumes someone else verified this.

## 6. `skills-for-fabric:FabricDataEngineer` (Pipelines)

- **Schedule body needs Windows TZ, not IANA.** `W. Europe Standard Time` ✅ — `Europe/Amsterdam` ❌. Skill is silent.
- **`interval` is minutes** (1440 = daily). Skill shows `frequency: "Daily"` which works for one shape and not another.
- **Pipeline / schedule POSTs are `201` synchronous, not `202` LRO** — different polling model than every other Items API endpoint. Easy to over-engineer the polling code.
- **`startDateTime`** must be future ISO with NO `Z` and NO offset. Adding `Z` returns `400 Invalid datetime`.

## 7. RuiRomano `powerbi:powerbi-developer` agent

Excellent at TMDL/PBIR mechanics. What's not in its system prompt:

- It assumes the model is already deployed when you hand it report work — it does not deploy semantic models. Use `powerbi-authoring-cli` first.
- Ask it to commit as a specific identity if you care: it will use the local git config otherwise.
- It bills LROs as "Succeeded" once the operation returns 200, but for SemanticModel the **framing** is a separate refresh op — tell it explicitly to wait for that too.

## 8. `pbicli` (https://github.com/powerbi-cli/powerbi-cli)

Installed mid-run. Lessons:

- Wraps **Power BI REST API only** (`api.powerbi.com`). Does **not** touch `api.fabric.microsoft.com` — so it cannot create lakehouses, notebooks, pipelines, or reports/models in the new Fabric Items API shape.
- DAX flag is `--dax`, NOT `--query` (which is JMESPath on the response). Easy 5-min trap.
- In a CLI subshell with redirected stdout (i.e. the way Copilot CLI runs it) the `dataset query` command **silently swallows output** even on success. `--output-file` was supposed to fix it but didn't reliably write the file. Fall back to `az rest` for validation queries.
- `pbicli pipeline` is **PBI Deployment Pipelines** (dev→test→prod release flow), NOT Fabric Data Pipelines. Different beast.
- `pbicli import pbix` accepts only legacy PBIX, not PBIR/enhanced.
- **Where it does shine:** `dataset list / show / refresh / takeover`, `report rebind`, `workspace user add`. Saves ~3-6 minutes versus hand-rolling REST.

## 9. `fab` CLI (Microsoft `ms-fabric-cli`)

Was on the prereq list, used sparingly.

- `fab login` and `fab ls` work great, but `fab import` for PBIR / TMDL has the same TTY-swallow problem as pbicli — sometimes succeeds, sometimes returns no output, sometimes 500s. `az rest` against the Items API was the reliable fallback for every persona.
- Workspace path syntax `<workspace>.Workspace/<item>.<Type>` is undocumented in the skill but mandatory.

## 10. Fabric **Data Agent** (item type `DataAgent`)

New in run 0054-1902. Bonus persona after Scoop. Skill coverage: zero — it's all from `fabric-mcp-server` `docs_item-definitions --workload-type dataAgent`.

- **Item type** is `DataAgent` (PascalCase) on `POST /v1/workspaces/{ws}/items`. LRO 202 → poll → `/result` returns the agent id.
- **Parts live under `Files/Config/`** (not `definition/`):
  - `Files/Config/data_agent.json` → `{ "$schema": "2.1.0" }` — that's literally the whole top-level config.
  - `Files/Config/draft/stage_config.json` → `{ "$schema": "1.0.0", "aiInstructions": "<the long prompt>" }`. **This is the entire system prompt** for the agent. Spend your time here.
  - `Files/Config/draft/<dataSourceType>-<dataSourceName>/datasource.json` → binds the agent to a model/lakehouse/warehouse. Type enum uses **snake_case** (`semantic_model`, `lakehouse_tables`, `data_warehouse`, `kusto`, etc.). `artifactId` + `workspaceId` point at the bound resource.
  - `Files/Config/draft/<dataSourceType>-<dataSourceName>/fewshots.json` → `{ "$schema": "1.0.0", "fewShots": [{ "id": "<guid>", "question": "...", "query": "..." }] }`. For `semantic_model` data sources the `query` is DAX (`EVALUATE …`). Generate fresh GUIDs per shot.
- **Draft vs published.** A fresh deploy is `draft` only — the agent won't answer end-user questions until you click *Publish* in the Service (or duplicate the same files into `Files/Config/published/` + add `Files/Config/publish_info.json`). Leaving it in draft is the right default for a code-deployed agent — give the user a chance to read your `aiInstructions` first.
- **What to put in `aiInstructions`.** Reusable template that worked: (1) one-paragraph identity + binding statement, (2) a **model cheat sheet** listing every table, key column, hierarchy and measure with a one-line description, (3) **known totals** for sanity-check (e.g. "fact_sales_order has 542 rows / total $708,690.15") so the agent can detect bad joins, (4) numbered **answering rules** (always use measures not raw `SUM`, currency format, time-intelligence default, top-N default, refusal-to-fabricate, PII guard), (5) **style** rules (concise, units, end every reply with a `Try next:` follow-up).
- **Don't ask the agent to write back.** Always include "do not write back to the model, lakehouse or pipeline" as a hard limit. The DataAgent runtime can call tools that mutate; the prompt is your only guardrail.

## 11. Geographic visuals (`filledMap`, `map`) and `dataCategory`

Came up on the second iteration of the report.

- **The `filledMap` / `map` visual will not geocode without `dataCategory` set on the geo columns.** A model deployed without categories renders the map as a single grey bubble.
- Set the categories on the dim_geography (or equivalent) columns:
  - `City` → `City`
  - `StateProvince` → `StateOrProvince` (note Pascal joining, no space)
  - `CountryRegion` → `Country`
  - `PostalCode` → `PostalCode`
- **Easiest path** is XMLA via `powerbi-modeling-mcp` `column_operations` `Update` with `dataCategory` per column. Metadata-only change, takes ~1 second, **no refresh required** — Direct Lake picks it up on the next query.
- The `filledMap` visual.json query has two projections:
  - `Category` → list of geo columns (Country first, then State, then City — that's the drill order)
  - `Y` → the size measure (e.g. `[Sum Order Value]`)
- AdventureWorks SalesLT geocodes cleanly to US/CA/UK/AU/DE/FR; expect ~6 country bubbles, drillable to ~30 states.

## 12. `openspec` + `squad`

- `squad hire` is a stub (v0.9.1). Charters must be hand-authored — every `4_killer_prompt` run will hit this until hire works.
- OpenSpec `validate --strict` rejects bullet-only requirements: every requirement *must* have at least one `#### Scenario:` block with `WHEN/THEN`. Skill examples are correct, but the linter error message ("missing scenarios") doesn't tell you the heading level it expects.

---

## TL;DR — top 8 things every skill should pin to its README

1. **Schema-enabled lakehouses inject `/dbo/` everywhere.** Decide at create time, never retrofit.
2. **TMDL = TABS only.** No manual `lineageTag`. `Sql.Database()` wants the **SQL endpoint GUID**.
3. **PBIR shell shapes come from `microsoft/semantic-link-labs/_bpareporttemplate`.** Visual.json shapes come from `ibarrau/PowerBi-code`. Schema URLs: `versionMetadata/1.0.0`, `report/1.1.0`, `pagesMetadata/1.0.0`, `page/1.1.0`, **`visualContainer/1.4.0`**. `version.json` value is **semver `X.Y.0`** (e.g. `2.0.0`).
4. **Deploy reports in two passes:** empty shell first (5 files) → then `/updateDefinition` with visuals. Bisecting visual bugs against a non-existent report id is misery.
5. **Direct Lake `definition.pbir`** uses `byConnection.pbiModelDatabaseName = <model-guid>` + `connectionType: "pbiServiceXmlaStyleLive"` + `pbiModelVirtualServerName: "sobe_wowvirtualserver"`. No `$schema` on this file.
6. **Pipeline schedules** want Windows TZ name + minute-interval + naked ISO datetime.
7. **`pbicli` is Power BI only.** For anything Fabric (lakehouse / notebook / pipeline / new-shape report / Data Agent) → `az rest` against `api.fabric.microsoft.com`.
8. **Maps need `dataCategory` set on geo columns** *before* the map visual will render. One XMLA `column_operations Update`, no refresh needed.
