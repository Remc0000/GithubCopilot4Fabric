# 🎤 The Fabric Roadshow Killer Prompt

> Copy everything between the `--- BEGIN ---` and `--- END ---` markers,
> paste into GitHub Copilot CLI, hit enter, lean back, enjoy the show.

---

## --- BEGIN ---

Hi Clawdia! 👋 We're live at the **Fabric Roadshow 2026** and you're the demo.
No pressure. The lights are on you. Bob the Builder is watching.

Before starting, make sure you delete earlier demo's. So delete the FabricRoadshow project in github and start clean.
Also delete the Fabric Roadshow workspace

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
- Skills (use these — and ONLY these — for Power BI / Fabric work):
  - **`fab` CLI** for workspace + items administration
  - **`skills-for-fabric`** (microsoft) — already installed at
    `~/.copilot/installed-plugins/fabric-collection/skills-for-fabric`
  - **`powerbi-agentic-plugins`** (RuiRomano) — `powerbi@powerbi-agentic-plugins`
    and `fabric@powerbi-agentic-plugins`. Already installed. Use these
    for ALL TMDL semantic-model work, PBIR report authoring, and DAX.
- Identity: commit as **`Remc0000`** I'm watching. 👀
  ```
  git -c user.name='Remc0000' -c user.email='223556219+Copilot@users.noreply.github.com' commit ...
  ```

### 🏗️ The squad (announce who's doing what, live)

| Agent | Role | Tools |
|---|---|---|
| 👷 **Bob** | Tech lead, orchestrates the build | skills-for-fabric, fab CLI |
| 🚜 **Muck** | Bulldozes data through bronze/silver/gold | PySpark notebooks, skills-for-fabric |
| 🏗️ **Scoop** | Scoops up the semantic model in TMDL | RuiRomano `powerbi` plugin |
| 🚧 **Roley** | Rolls out the Power BI report (PBIR) | RuiRomano `powerbi` plugin |
| 🏎️ **Lofty** | Lifts heavy specs, writes OpenSpec | openspec |
| 🚒 **Dizzy** | Mixes everything in CI / GitHub | gh CLI |

### 👥 Use a real Squad — don't just role-play

Before you start the plan below, **initialise a Squad** in the project
and **hire each agent above as a real squad member**, then **delegate
every task in the plan to the matching squad agent**. Don't fake the
hand-offs with narration only — actually use `squad` so I can see who
did what in `squad cost` afterwards.

```powershell
# In the FabricRoadshow project root:
squad init                           # markdown-only, default
squad hire --name Bob   --role tech-lead
squad hire --name Muck  --role data-engineer
squad hire --name Scoop --role semantic-model-author
squad hire --name Roley --role report-author
squad hire --name Lofty --role spec-author
squad hire --name Dizzy --role devops
```

Then for each step below, dispatch the work to the named squad agent
(via `squad` interactive shell or by invoking the agent persona). The
plan steps already say which agent owns each step — keep that mapping.

### 📺 How to watch the squad work (open a 2nd terminal)

Open a second PowerShell window during the demo and keep these running
so the audience can see who's doing what in real time:

```powershell
# 1. Status — which squad is active, where is it rooted
squad status

# 2. Live progress — tail the orchestration log as agents post updates
Get-ChildItem .squad -Recurse -Filter *.log -ErrorAction SilentlyContinue
Get-Content .squad\orchestration.log -Wait -Tail 20    # adjust path if different

# 3. Per-agent token cost (refresh every few seconds)
while ($true) { Clear-Host; squad cost --all; Start-Sleep 5 }

# 4. The work the squad is creating, on the file system
Get-ChildItem -Recurse -Force -Filter "*.md" .squad\agents | Select-Object FullName, LastWriteTime |
  Sort-Object LastWriteTime -Descending | Select-Object -First 10
```

Also, **Clawdia must announce hand-offs in chat** when she dispatches
work to a squad agent (`👷 Bob, take it…`) and again when the agent
reports back (`✅ Bob done — over to 🚜 Muck`). Combined with the live
terminals above, the audience sees both narration and proof.

### ⏱️ Time budget per stage (target total ≤ 30 min)

| Stage | Owner | Max time |
|---|---|---|
| Pre-flight (auth, repo reset, squad init) | Clawdia | 2 min |
| OpenSpec proposal | 🏎️ Lofty | 2 min |
| Workspace + 3 lakehouses + shortcuts | 👷 Bob | 4 min |
| 3 medallion notebooks (Bronze/Silver/Gold) authored + run | 🚜 Muck | 8 min |
| TMDL Direct Lake model + framing + DAX validation | 🏗️ Scoop | 7 min |
| PBIR enhanced report deployed | 🚧 Roley | 5 min |
| Final commit & push | 🚒 Dizzy | 2 min |
| **Total** | | **30 min** |

If you hit the cap on any stage, **stop polishing** and move on with
what works. Audience > perfection.

### 📋 The plan (in this order, no improvising)

1. **Lofty** — write an **OpenSpec** proposal for `add-orders-analytics`,
   validate `--strict` (remember: SHALL/MUST goes in the requirement
   **body**, not just the heading), commit to a fresh GitHub repo
   **`FabricRoadshow`**.
2. **Bob** — `fab` create workspace **`Fabric Roadshow`**, assign to
   `Trial-Remco`. Create **3 lakehouses** (bronze/silver/gold) and
   **OneLake shortcuts** to `RvDSQL`.
3. **Muck** — Build a **classic 3-notebook medallion** (this is the
   standard for the **`e2e-medallion-architecture`** skill, please use
   it). I want **THREE separate notebooks**, not one mega-notebook:
   - `01_bronze_ingest.Notebook` — read mirrored shortcuts from `bronze_lh`
     (raw copy, schema preservation, light typing only). Land Bronze
     Delta tables in `bronze_lh.Tables.<name>` (or read shortcuts in
     place if that's cleaner — your call, document why).
   - `02_silver_clean.Notebook` — clean, dedupe, derive `LineTotal =
     OrderQty * UnitPrice * (1 - UnitPriceDiscount)`, derive
     `StateName` from `PostalCode` prefix, hardcode category names.
     Write to `silver_lh.Tables.*`.
   - `03_gold_star.Notebook` — build the star schema (`fact_sales_order`,
     `dim_product_category`, `dim_geography`, `dim_state`, `dim_date`),
     compute `IsOutlier` (IQR rule — see note below), validate totals
     with `print()` checkpoints. Write to `gold_lh.Tables.*`.

   For each notebook:
   - Use Markdown cells generously to explain WHAT and WHY (the audience
     reads them on screen).
   - Default lakehouse in the leading `# META` block matches the layer
     it writes to (bronze_lh / silver_lh / gold_lh respectively).
   - Author as `.ipynb` (NOT cell-delimited `.py` — CRLF round-trip
     breaks the parser). Save under `fabric/notebooks/<nn>_<name>.Notebook/`.
   - Deploy via REST `POST /v1/workspaces/{ws}/items` — `fab import`
     CRASHES inside this CLI session (`No Windows console found`).
   - Run them in order via the Items jobs API (Bronze → Silver → Gold)
     and gate each step on the previous job reporting `Completed`.

   ⚠ **Outlier note:** at order grain (32 rows) `1.5 × IQR` produces
   ZERO outliers. Don't chase the magic number `64`. Either (a) compute
   `IsOutlier` at line grain and aggregate `OutlierLineCount` into the
   fact, or (b) keep order-grain and accept `Outlier Count = 0`. Pick
   one and move on — don't burn 5 minutes trying to "fix" the count.
4. **Scoop** — author **TMDL** for a **Direct Lake** semantic model
   `OrdersAnalytics`. Rules of the house:
   - **TABS, not spaces.** Verify 0x09. I'll know.
   - No manual `lineageTag` on new objects.
   - Relationships use `fromColumn` / `toColumn`, **not** `fromTable`.
   - Display names with spaces (`'Category Name'`, `'State Name'`) —
     visuals and DAX reference the **display name**, not `sourceColumn`.
     Get this wrong and 11 visuals scream "Can't display". Don't ask
     how I know.
5. **Scoop** — deploy via REST `POST /v1/workspaces/{ws}/items` (NOT
   `fab import` — it crashes inside this CLI session). Use the **SQL
   endpoint GUID**, not the lakehouse GUID, in `Sql.Database()` — wrong
   one fails framing. Trigger a Direct Lake **framing refresh** (`POST
   .../refreshes {"type":"Full"}`) before any DAX. Validate via REST
   `executeQueries` (NOT the PowerBIQuery MCP tool — it caches stale
   model lists): total ≈ **$708,690**, **32 orders**. (Outlier count
   target depends on the choice Muck made in step 3; just confirm the
   measure returns a number, don't insist on `64`.)
6. **Roley** — author the **PBIR enhanced format** using the RuiRomano
   `powerbi@powerbi-agentic-plugins` plugin. Its templates target the
   schemas Fabric service expects:
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
3. The **total elapsed time** (target: **Max** `30min` otherwise people are already gone)
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
