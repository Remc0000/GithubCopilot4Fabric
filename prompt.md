# 🛠️ The "Sales-by-Category-with-a-Heatmap" Demo Prompt

> *aka "the one Remco runs on stage to prove that GitHub Copilot on Fabric isn't a slide deck."* 🐱
>
> If you (Copilot) read this and sigh, congratulations — you have feelings. Now build the report.

---

## 0. Pre-requisites — read these BEFORE you touch a tool

You **must** ingest these two files into context first. They are not optional. They are the difference between a 35-minute green-light demo and a 2-hour saga where Clawdia learns what `lineageTag` does:

1. 📄 `C:\Users\revandam\GHCP4Fab\3_missing_in_skills.md` — every gotcha per skill, hard-won.
2. 📄 `C:\Users\revandam\GHCP4Fab\3_new_skill.md` — the orchestration recipe + persona timeouts.

If either file is missing, **stop** and ask the user. Don't improvise the orchestrator from scratch — that path is paved with empty `executeQueries` responses and tears.

### Tooling you are required to use

| Tool / family | Why |
|---|---|
| 🐝 **Squad** (`.squad/`, charters, background `task` agents) | 6-persona orchestration. `squad hire` is a stub — hand-author the charters. |
| 📜 **OpenSpec** | Lofty writes the proposal. `openspec validate --strict` is a hard gate. |
| 🧰 **`skills-for-fabric`** family | Bob (Admin), Muck + Dizzy (DataEngineer). |
| ⭐ **`e2e-medallion-architecture`** (THE most important skill) | Muck lives here. Bronze→Silver→Gold, schema-enabled, Direct Lake-ready. |
| 🎨 **RuiRomano `powerbi:` skills** (`powerbi-architect`, `powerbi-developer`) | Scoop (model) + Roley (report). |
| 🪄 **`pbicli`** (https://github.com/powerbi-cli/powerbi-cli) | Power BI plane only — dataset list/refresh, report rebind. **Don't** try to deploy with it. |
| 🧱 **`fab` CLI** | Quick `fab ls` and login sanity. Fall back to `az rest` whenever it goes silent. |

> 🐈 **Identity rule:** every git commit must be `Remc0000 <223556219+Copilot@users.noreply.github.com>`. No exceptions. Don't let Roley sneak in as `clawdia@meow`.

---

## 1. Functional Requirements — what Remco actually wants to show on stage

Build a **Sales Analysis** Power BI report with the following:

### 1.1 Pages

- **Page 1 — "Category Overview"** 🛒
  - Two big KPI **cards** at the top: total revenue + order count. Make them look like they cost money. (They didn't. It's AdventureWorksLT.)
  - A **clustered bar chart** of revenue by **Category** (drill-up to **Master Category**).
  - A **donut** of orders by Master Category — because at least one C-suite member in the room loves a donut.
  - A **slicer** on Master Category to keep the demo interactive.

- **Page 2 — "The Heatmap"** 🔥
  - A **matrix / heatmap visual** with **Category on rows × Month-of-Year on columns**, conditional-formatted by `Sum Order Value`.
  - Optional drill: rows can drop into **Product**.
  - This is the money shot. If the heatmap doesn't render in <2 seconds, blame Direct Lake and demo it again.

### 1.2 Non-functional whispers

- Direct Lake (no import). If anyone asks "is this live?" the answer is "yes, technically more live than your last all-hands."
- Theme `CY24SU10`. Dark text, light bg. We're not cosplaying a Bloomberg terminal.
- Workspace name: `Fabric Roadshow_<slug>`. Slug = `RAND4-HHMM`.

---

## 2. Technical Requirements — the bits that break demos when you skip them

### 2.1 The Category dimension (read this **twice**)

The whole point of this demo is that **GitHub Copilot can model a real two-level hierarchy properly.** Don't flatten it. Don't fake it.

- Build `dim_product_category` with **two levels**:
  - `MasterCategoryKey` + `MasterCategoryName` (e.g. *Bikes*, *Components*, *Clothing*, *Accessories*)
  - `CategoryKey` + `CategoryName` (e.g. *Mountain Bikes*, *Road Bikes*, *Helmets*, *Jerseys*)
- In TMDL, expose a **user hierarchy** named `Category Hierarchy` with two levels: `Master Category` → `Category`.
- The fact table joins to `CategoryKey` only. The hierarchy does the navigation.
- In AdventureWorksLT this is `SalesLT.ProductCategory` — note `ParentProductCategoryID` is the self-join. Master = rows where `ParentProductCategoryID IS NULL`. Children = rows where it IS NOT NULL. **Resolve this in Silver, not in DAX.** If Copilot tries to `IF(BLANK())` the parent in DAX, slap its paws.

### 2.2 Power BI report scope

- Direct Lake semantic model **`OrdersAnalytics`**, deployed to the workspace, framed before the report goes live.
- Report **`OrdersAnalytics_Report`**, PBIR enhanced format, two pages as above.
- Validate with one DAX call via `az rest /executeQueries`:
  ```
  EVALUATE ROW(
    "Sum",        [Sum Order Value],
    "Count",      [Order Count],
    "Heatmap OK", IF(COUNTROWS(SUMMARIZE(fact_sales_order, dim_product_category[CategoryName], dim_date[MonthName])) >= 12, "yes", "no")
  )
  ```
- If `pbicli dataset query` returns empty stdout (it will, in this shell), don't waste 20 minutes — go straight to `az rest`. The pain of others has already been paid; see `3_missing_in_skills.md §8`.

### 2.3 Source tables you must analyse (medallion-side)

Source = **AdventureWorksLT** lakehouse exposed via OneLake shortcuts. The tables Bob shortcuts and Muck consumes:

| Source table | Used for | Notes |
|---|---|---|
| `SalesLT.SalesOrderHeader` | Order grain in Silver | Don't use `TotalDue` for the fact's `OrderValue` — that includes tax/freight. We use line-level revenue. |
| `SalesLT.SalesOrderDetail` | Line items → fact `LineTotal` aggregated to order or kept at line | **`SUM(LineTotal)` is the canonical revenue.** Last demo run used `TotalDue` and got $956k instead of $708k — Remco noticed in front of 40 people. Don't be that demo. |
| `SalesLT.Product` | Product dim | Join to `ProductCategory`. |
| `SalesLT.ProductCategory` | The two-level Category dim | **Self-join**. Master = parent IS NULL. Resolve in Silver. |
| `SalesLT.Customer` + `SalesLT.Address` | Geography (state, country) for the bar/column charts on Page 1 | Optional but cheap. |

### 2.4 Medallion table contract

Gold (consumed by Direct Lake) must include:

- **`fact_sales_order`** at line grain: `OrderKey`, `LineKey`, `DateKey`, `CustomerKey`, `ProductKey`, `CategoryKey`, `OrderValue` (= `LineTotal`), `Quantity`.
- **`dim_product_category`**: `CategoryKey`, `CategoryName`, `MasterCategoryKey`, `MasterCategoryName`. *Already flattened.*
- **`dim_date`**: contiguous date range, with `Date`, `Year`, `Quarter`, `Month`, `MonthName`, `MonthNumber`, `DayOfWeek`. Mark as date table.
- **`dim_product`**, **`dim_customer`**, **`dim_geography`**: the obvious shapes.

Schema-enabled lakehouses → tables land at `Tables/dbo/<name>`. Don't forget the `/dbo/`. (Yes, this gotcha is in `3_missing_in_skills.md §1`. Read it twice.)

### 2.5 Pipeline

`nightly_refresh` Data Pipeline running **daily at 02:00 W. Europe Standard Time** (Windows TZ name, not IANA). Activities: Bronze → Silver → Gold → SemanticModel refresh. Smoke-run it once after deploy, and accept the `202` synchronously. Schedule body uses `interval: 1440` minutes.

---

## 3. Acceptance criteria (Remco's stage checklist)

- [ ] Workspace `Fabric Roadshow_<slug>` exists, capacity attached, 3 lakehouses, schemas on. 🏗️
- [ ] Bronze, Silver, Gold notebooks all `Completed`. Gold row count and `SUM(LineTotal)` match the OpenSpec proposal. 🚜
- [ ] Direct Lake `OrdersAnalytics` model framed, hierarchy `Category Hierarchy` works in DAX. 🚂
- [ ] Report `OrdersAnalytics_Report` opens in the service with **both pages** rendering — cards, bar, donut, slicer, AND the heatmap. 🛣️
- [ ] Pipeline + schedule deployed; smoke run kicked. 🚧
- [ ] OpenSpec proposal validates `--strict`. 📜
- [ ] `.squad/orchestration.log` ≥ 12 lines. 🐝
- [ ] GitHub repo `Remc0000/FabricRoadshow_<slug>` exists, public, has all artifacts. ⬆️
- [ ] Final ASCII finish flag with elapsed time + report URL printed to chat. 🏁

---

## 4. The vibe 😼

This is a **demo**, not a SOC2 audit. Optimise for:

1. **Greens on the validation steps.** Anything off by ±5% → flag in the finish report, don't re-author. Save the heroic re-runs for staging.
2. **Wall-clock under 35 minutes.** If Scoop blows past 10 minutes on framing, kill it and run the framing call yourself in 1 line of `az rest`.
3. **Funny commit messages are encouraged.** "fix: ProductCategory was lying about its parents" gets a laugh in the room.
4. **Cats welcome.** A small ASCII cat in the finish flag is canon.

```
       /\_/\
      ( ^.^ )    "Direct Lake go brrr."
       > ^ <
```

Now go. The audience is loading their LinkedIn. ⏱️
