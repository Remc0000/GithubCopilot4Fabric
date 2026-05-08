# 🎤 The Fabric Roadshow Killer Prompt

> Copy everything between the `--- BEGIN ---` and `--- END ---` markers,
> paste into GitHub Copilot CLI, hit enter, lean back, enjoy the show.

---

## --- BEGIN ---

Hi GHCP! 👋 We're live at the **Fabric Roadshow 2026** and you're the demo.
No pressure. The lights are on you. Bob the Builder is watching.

Build me an **end-to-end Microsoft Fabric analytics solution** — from
mirrored database to a published Power BI report — and please do it
**without the rookie mistakes you made last time** (yes, I kept notes,
they're in `C:\Users\revandam\GHCP4Fab\3_extra_input.md` — read them first,
I'm begging you).

### 🎯 The mission

Analyse **sales orders by Product Category × City × State** with
**Sum / Avg / Max** order value and **IQR-based outlier detection**.

### 🧰 What you have

- Mirrored DB **`RvDSQL`** in workspace **`SalesLT.Workspace`**
  (AdventureWorksLT — but redacted: no StateProvince, no LineTotal, no
  Name. You'll have to be clever. Postal-code prefix → state. Compute
  LineTotal as `OrderQty * UnitPrice * (1 - Discount)`.)
- Trial capacity **`Trial-Remco`**
- Skills: **fab CLI**, **skills-for-fabric**, **kpbray/power-bi-agent-skills**
- Identity: commit as **`Remc0000`** — NOT `Remc0123`. I'm watching. 👀
  ```
  git -c user.name='Remc0000' -c user.email='223556219+Copilot@users.noreply.github.com' commit ...
  ```

### 🏗️ The squad (announce who's doing what, live)

| Agent | Role | Tools |
|---|---|---|
| 👷 **Bob** | Tech lead, orchestrates the build | skills-for-fabric, fab CLI |
| 🚜 **Muck** | Bulldozes data into the warehouse | T-SQL, sqldw-authoring-cli |
| 🏗️ **Scoop** | Scoops up the semantic model in TMDL | powerbi-authoring-cli |
| 🚧 **Roley** | Rolls out the Power BI report (PBIR) | kpbray report-visuals |
| 🏎️ **Lofty** | Lifts heavy specs, writes OpenSpec | openspec |
| 🚒 **Dizzy** | Mixes everything in CI / GitHub | gh CLI |

### 📋 The plan (in this order, no improvising)

1. **Lofty** — write an **OpenSpec** proposal for `add-orders-analytics`,
   validate `--strict` (remember: SHALL/MUST goes in the requirement
   **body**, not just the heading), commit to a fresh GitHub repo
   **`FabricRoadshow`**.
2. **Bob** — `fab` create workspace **`Fabric Roadshow 2`**, assign to
   `Trial-Remco`. Create **3 lakehouses** (bronze/silver/gold) and
   **OneLake shortcuts** to `RvDSQL`.
3. **Muck** — skip the PySpark notebook. It will fail. It always fails.
   Go **straight to a T-SQL Warehouse** (`gold_wh`) with a star schema
   (fact_sales_order, dim_product_category, dim_geography, dim_state,
   dim_date). Cast everything `varchar` — Fabric DW doesn't do nvarchar.
4. **Scoop** — author **TMDL** for a **Direct Lake** semantic model
   `OrdersAnalytics`. Rules of the house:
   - **TABS, not spaces.** Verify 0x09. I'll know.
   - No manual `lineageTag` on new objects.
   - Relationships use `fromColumn` / `toColumn`, **not** `fromTable`.
   - Display names with spaces (`'Category Name'`, `'State Name'`) —
     visuals and DAX reference the **display name**, not `sourceColumn`.
     Get this wrong and 11 visuals scream "Can't display". Don't ask
     how I know.
5. **Scoop** — deploy via `fab import`, then **trigger a Direct Lake
   framing refresh** (`POST .../refreshes {"type":"Full"}`) before any
   DAX. Validate: total ≈ **$708,690**, **32 orders**, **64 outliers**.
6. **Roley** — author the **PBIR enhanced format** that Fabric *actually*
   accepts (the kpbray 1.0.0 schemas don't — learned that the hard way):
   - `definition.pbir` → `definitionProperties/2.0.0`, `version: "4.0"`
   - `definition/version.json` → `versionMetadata/1.0.0`, `"2.0.0"` ⚠️ required
   - `definition/report.json` → `report/3.0.0`, object `reportVersionAtImport`
   - `definition/pages/<id>/visuals/<n>/visual.json` → `visualContainer/2.4.0`
   - `byConnection` with `connectionString` containing `semanticModelId=<guid>`
   - **Every projection needs `nativeQueryRef`** or visuals go dark.
   - Visual types: `clusteredBarChart`, `clusteredColumnChart`, `donutChart`,
     `card`, `tableEx`, `slicer`. (Not `barChart`. Don't try.)
7. **Dizzy** — commit & push everything to
   `https://github.com/Remc0000/FabricRoadshow` as `Remc0000`.

### 🎬 Showmanship rules

- **Announce each agent** before they work. ("🚜 Muck rolling in…")
- When something fails, **don't silently retry 14 times**. Tell us, pivot,
  move on. The audience is here for the journey, not the stack traces.
- **No PowerBI MCP cache traps** — use the `executeQueries` REST API for
  fresh Direct Lake models.
- **API audiences** (memorise, do not mix):
  - Items API → `https://api.fabric.microsoft.com`
  - Datasets API → `https://analysis.windows.net/powerbi/api`
  - OneLake DFS → `https://storage.azure.com/`

### 🏁 The finish line

When done, print:
1. An **ASCII finish flag** 🏁
2. The **GitHub Copilot banner** in ASCII
3. The **total elapsed time** (target: beat the previous `1h 14m`,
   because last time you discovered the PBIR format the hard way and
   we'd all like our lunch break back)
4. Links to: the GitHub repo, the workspace, the published report

### 🙏 One last thing

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
