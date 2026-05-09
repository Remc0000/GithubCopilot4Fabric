# 3 — New Skill: `fabric-roadshow-orchestrator`

> Everything I (Clawdia) learned during the run that is **not currently captured** in any of the skills, MCP servers, or the Squad/OpenSpec CLIs. Drop this in `~/.agents/skills/fabric-roadshow-orchestrator/SKILL.md` and the next Copilot run won't repeat my mistakes.

---

```yaml
name: fabric-roadshow-orchestrator
description: >
  Orchestrate an end-to-end Microsoft Fabric demo (workspace + medallion lakehouse +
  Direct Lake semantic model + PBIR report + nightly pipeline + GitHub repo) under a
  slug-suffixed workspace using a 6-persona background-agent squad. Use when the user
  wants to: (1) run the "killer prompt", (2) provision a self-contained Fabric demo,
  (3) coordinate Bob/Muck/Scoop/Roley/Lofty/Dizzy personas, (4) gate Direct Lake model
  authoring on medallion completion. Triggers: "killer prompt", "roadshow", "squad run",
  "fabric e2e demo", "6 personas", "slug workspace".
```

---

## When to use

- The user says "run the killer prompt" or hands you a roadshow-shaped spec (one workspace, three lakehouses, a Direct Lake model, a report, a pipeline).
- The work fans out into >3 independent streams that benefit from background agents (charter-driven personas).
- A short slug (`####-HHMM`) suffix is required so two people can run the demo in parallel without colliding on workspace names.

## When NOT to use

- A single-purpose ask ("just create a notebook") — use the underlying skill directly.
- Production migrations — this orchestrator optimises for time-to-green-light, not bulletproofing.

---

## Core architecture

```
Clawdia (you)
  ├─► Pre-flight (slug, tokens, scaffold) — you do this yourself, no agent
  ├─► Fan-out #1 (parallel): Lofty, Bob, preflight-verify
  ├─► Gate: Bob.workspace_id present
  ├─► Muck (medallion) — sequential, gates the rest
  ├─► Gate: Muck.gold_status == "Completed"
  ├─► Fan-out #2 (parallel): Scoop, Roley-PhaseA, Dizzy-PhaseA
  ├─► Gate: Scoop.model_id present
  ├─► Phase B messages to idle Roley + Dizzy with real MODEL_ID
  └─► Final commit + push + finish flag
```

Every persona writes a `<name>-output.json` and appends to `.squad/orchestration.log` (≥12 lines is the self-grading threshold).

---

## The 6 personas (charter cheat-sheet)

| Persona | Emoji | Primary skill | What they own |
|---|---|---|---|
| **Lofty** | 📜 | `openspec` CLI | Proposal under `openspec/changes/<slug>/`. Validates `--strict`. |
| **Bob** | 🏗️ | `skills-for-fabric:FabricAdmin` | Workspace + 3 schema-enabled lakehouses + OneLake shortcuts to source. |
| **Muck** | 🚜 | `skills-for-fabric:FabricDataEngineer` (`e2e-medallion-architecture`) | 3 medallion notebooks; deploys + runs them; validates gold rowcounts. |
| **Scoop** | 🚂 | `powerbi-authoring-cli` + `powerbi:powerbi-developer` | TMDL Direct Lake model; framing refresh; DAX measure validation. |
| **Roley** | 🛣️ | `powerbi-report-authoring` + `powerbi:powerbi-developer` | PBIR enhanced report. **Two phases** — Phase A authors with `{{MODEL_ID_PLACEHOLDER}}`, Phase B swaps + deploys. |
| **Dizzy** | 🚧 | `skills-for-fabric:FabricDataEngineer` (Pipelines) + `gh` CLI | Nightly_refresh pipeline + schedule; final GitHub repo push. |

> **`squad hire` is a stub.** Hand-author each `.squad/agents/<name>/charter.md`. The "agent" is a background `task` that reads the charter and appends to `.squad/orchestration.log`.

---

## Pre-flight checklist (do these yourself, never delegate)

```powershell
$OutputEncoding = [Text.Encoding]::UTF8
[Console]::OutputEncoding = [Text.Encoding]::UTF8
$env:PYTHONIOENCODING = "utf-8"

# Slug is random4-HHMM (e.g. 9155-1501) — keep both repo and workspace names tied to it
$SLUG = "{0:D4}-{1}" -f (Get-Random -Max 10000), (Get-Date -Format "HHmm")
$WORKSPACE = "Fabric Roadshow_$SLUG"   # NOTE the SPACE *and* underscore — quote everywhere
$REPO      = "FabricRoadshow_$SLUG"

# Pre-warm 4 token audiences in parallel — they all need to be cached before agents run
$audiences = @(
  "https://api.fabric.microsoft.com",
  "https://analysis.windows.net/powerbi/api",
  "https://storage.azure.com/",
  "https://database.windows.net/"
)
$audiences | ForEach-Object -Parallel { az account get-access-token --resource $_ --query accessToken -o tsv | Out-Null }
```

Persist `slug.txt` to `~/.copilot/session-state/<id>/files/` so a resumed session doesn't re-roll the slug.

---

## The orchestration sequence (verbatim)

### Fan-out #1 — three agents in ONE tool call

Independent. Fire them together.

```
task: explore             → preflight-verify   (verifies source workspace + AdventureWorks LH)
task: general-purpose     → lofty-openspec     (writes + validates OpenSpec change)
task: skills-for-fabric:FabricAdmin → bob-workspace
```

Wait until **Bob** has emitted `bob-output.json` with `workspace_id` and the 3 lakehouse ids. The other two can keep running in the background.

### Sequential — Muck

```
task: skills-for-fabric:FabricDataEngineer → muck-medallion
```

Charter must include:
- "Use the e2e-medallion-architecture skill."
- "Schema-enabled lakehouses → ABFSS paths use `Tables/dbo/`."
- "Run notebooks sequentially; poll each LRO to `Completed`."
- "Validate gold: `SUM(Order Value)` ≈ `<spec value>` and `COUNT(orders) = <N>`."

Gate on `muck-output.json.gold_status == "Completed"` AND row count matches.

### Fan-out #2 — three agents in ONE tool call (Phase A)

```
task: powerbi:powerbi-architect / authoring-cli  → scoop-tmdl       (deploy + frame model)
task: powerbi:powerbi-developer                  → roley-pbir       (PHASE A — placeholder)
task: skills-for-fabric:FabricDataEngineer       → dizzy-pipeline   (PHASE A — placeholder)
```

`{{MODEL_ID_PLACEHOLDER}}` is the convention. Roley + Dizzy author all artifacts with that string and then go **idle** — they do not deploy yet.

### Phase B — wake Roley + Dizzy with the real MODEL_ID

When Scoop is done:

```
write_agent → roley-pbir   "Phase B GO. MODEL_ID = <guid>"
write_agent → dizzy-pipeline "Phase B GO. MODEL_ID = <guid>"
```

They swap, deploy, commit, append to log.

### Final

You (Clawdia) do the final commit + `gh repo create … --public --source=. --push` + finish flag.

---

## Hard timeouts (enforce these in charters)

| Persona | Soft cap | Hard cap |
|---|---|---|
| Lofty | 3 min | 5 min |
| Bob | 5 min | 8 min |
| Muck | 10 min | 15 min |
| **Scoop** | **7 min** | **10 min** ← currently blows past, biggest risk |
| Roley A | 5 min | 7 min |
| Roley B | 3 min | 5 min |
| Dizzy A+B | 5 min | 8 min |

> **Scoop reality check:** Direct Lake framing is async. If `executeQueries` returns empty rows for >60s after deploy, kill the agent and re-frame manually with `POST /datasets/<id>/refreshes` + `objects:[{table:"...",partition:"..."}]`. Do NOT let the agent loop on `executeQueries` — that is what cost me 2 hours.

---

## Validation that is non-negotiable

```bash
# After Muck:
sqlcmd -S <gold_sql_endpoint> -d <ws_id> -G -Q "SELECT COUNT(*), SUM(order_value) FROM gold.fact_sales_order"

# After Scoop (and Roley):
az rest --resource https://analysis.windows.net/powerbi/api \
  --method POST \
  --url "https://api.powerbi.com/v1.0/myorg/groups/$WS/datasets/$MODEL_ID/executeQueries" \
  --body @dax.json
```

Why `az rest` and not `pbicli dataset query`? Because the latter silently swallows stdout in a non-TTY shell. Confirmed across multiple attempts.

---

## Anti-patterns I personally fell into

1. **Re-running validation when the answer didn't match the spec.** If gold returns $956k but spec says $708k, *log the discrepancy and move on* — do not re-author Muck inside Scoop's window. Flag it for the next demo.
2. **Trying `pbicli` for Fabric Items API operations.** It cannot. `az rest` is the only universal hammer.
3. **Polling `getDefinition` to confirm `updateDefinition` worked.** `updateDefinition` LRO `Succeeded` is sufficient. `getDefinition` is itself an async LRO and adds 30-90s of pure latency per item.
4. **Letting agents own the final push.** They commit to local — but the GitHub repo create + push is way faster done by you in 2 lines, and you avoid charter-drift.

---

## Self-grading rule (`§7` of the killer prompt)

```
.squad/orchestration.log line count ≥ 12   → squad-mode credit ✅
each persona output JSON exists            → completeness ✅
gold row count + sum match spec ±5%        → data-correctness ✅
report URL renders cards (not error)       → BI ✅
schedule next-run < 24h from now           → ops ✅
```

Anything else (perf, theme polish, accessibility) is bonus.

---

## Companion files

- `3_missing_in_skills.md` — gotchas per skill that the skills themselves don't document.
- `3_extra_input.md` — the original 47KB lessons file (do not modify, consult when blocked).
- `4_killer_prompt.md` — the actual demo script.

---

## One-liner cat seal of approval

```
 /\_/\
( o.o )   "if it looks like a 30-min demo, budget 90."
 > ^ <
```
