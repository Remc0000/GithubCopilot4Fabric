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

➡ This is exactly why the prompt said *"state is not in the data, so add this"*. Always inspect the mirrored schema first (`sys.columns` from the SQL endpoint) — don't trust the source schema docs.

## 2. Spark notebook diagnostics are a black hole

A PySpark medallion notebook failed twice with `System_Cancelled_Session_Statements_Failed`. Tried REST endpoints to retrieve cell-level errors:

- `/jobs/instances/{id}/snapshot` → 404
- `/jobs/instances/{id}/logs` → unauthorized
- `/jobs/instances/{id}/sparkUiLink` → 404

➡ **Pivoted to a Fabric T-SQL Warehouse** — built the entire star schema with cross-database queries (`bronze_lh.dbo.<shortcut>` from `gold_wh`). Far more debuggable, no Spark session needed, ran in seconds.

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

---

**Total demo time:** 1h 14m (from `Hi GHCP!` to working visuals on a published report)
**Final repo:** https://github.com/Remc0000/FabricRoadshow
**Final workspace:** `Fabric Roadshow 2` on `Trial-Remco.Capacity`
