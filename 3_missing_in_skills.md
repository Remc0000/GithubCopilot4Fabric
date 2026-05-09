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

The skill knows PBIR exists. It does not know schema v2 broke half the things v1 docs say.

- **`definition.pbir`** schema URL has *no* `/definition/` segment, but every file inside `definition/` *does* have it. Two different schema namespaces in the same folder.
- **`byConnection` only allows `connectionString`** in v2. Adding `pbiServiceModelId` or `pbiModelDatabaseName` (which the v1 sample shows) makes the LRO fail with `unknown property` — but the error blob comes back `{}` so you have to bisect.
- **`reportVersionAtImport`** must be `{ "report": "5.54.0", "page": "5.54.0", "visual": "5.54.0" }` — three semver strings. Old samples show `{ major: 5, minor: 54 }` which is rejected.
- **Visual types renamed.** `barChart` / `columnChart` → `clusteredBarChart` / `clusteredColumnChart`. Skill lists "barChart" in examples; current schema rejects it.
- **Every projection needs `queryRef` AND `nativeQueryRef`.** `queryRef` is `Table.Field`, `nativeQueryRef` is the display name. Missing either → "Can't display the visual" with no diagnostics.
- **Forbidden top-level keys in `page.json`:** `filters` (must be omitted, not `[]`), `config`, `ordinal`. The skill validator should grep for these.

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

## 10. `openspec` + `squad`

- `squad hire` is a stub (v0.9.1). Charters must be hand-authored — every `4_killer_prompt` run will hit this until hire works.
- OpenSpec `validate --strict` rejects bullet-only requirements: every requirement *must* have at least one `#### Scenario:` block with `WHEN/THEN`. Skill examples are correct, but the linter error message ("missing scenarios") doesn't tell you the heading level it expects.

---

## TL;DR — top 5 things every skill should pin to its README

1. **Schema-enabled lakehouses inject `/dbo/` everywhere.** Decide at create time, never retrofit.
2. **TMDL = TABS only.** No manual `lineageTag`. `Sql.Database()` wants the **SQL endpoint GUID**.
3. **PBIR v2** has two schema namespaces, three-string `reportVersionAtImport`, and forbidden keys (`config`, `ordinal`, `filters`).
4. **Pipeline schedules** want Windows TZ name + minute-interval + naked ISO datetime.
5. **`pbicli` is Power BI only.** For anything Fabric (lakehouse / notebook / pipeline / new-shape report) → `az rest` against `api.fabric.microsoft.com`.
