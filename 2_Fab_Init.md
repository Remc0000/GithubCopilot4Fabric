# Microsoft Fabric — Copilot CLI Setup Guide

Everything you need on top of [`1_GHCP_Init.md`](1_GHCP_Init.md) to use **GitHub Copilot CLI with Microsoft Fabric**.

This guide installs:

1. **Python 3.13** (side-by-side with newer Python versions)
2. **Fabric CLI** (`fab`)
3. **fabric-cicd** (Python package for Fabric CI/CD)
4. **skills-for-fabric** (Copilot CLI plugin with Fabric skills)

> **TL;DR — "install 2_Fab_Init.md"**
> Run the [Quick Install Script](#quick-install-script) at the bottom of this file. It is idempotent — re-running it only installs what's missing or out of date.

---

## Prerequisites

- Windows with PowerShell
- [GitHub Copilot CLI](https://githubnext.com/projects/copilot-cli) installed and signed in (see `1_GHCP_Init.md`)
- `git` available on PATH
- Azure CLI signed in (`az login`) — needed for Fabric REST API calls later

---

## 1. Python 3.13 (required for fabric-cicd)

The `fabric-cicd` package requires Python `>=3.9, <3.14`. If you only have Python 3.14+, install 3.13 alongside it using the Python launcher (`py`).

```powershell
# Check current Python
python --version

# Check if 3.13 is already available via py launcher
py -3.13 --version

# If not installed, install it (Python launcher 3.10+):
py install 3.13
```

After installation:

```powershell
py -3.13 --version   # should print: Python 3.13.x
```

---

## 2. Fabric CLI (`fab`)

The Fabric CLI is the official command-line tool for Microsoft Fabric.

**Docs:** <https://microsoft.github.io/fabric-cli/>

```powershell
# Check if installed
fab --version

# Install / upgrade (uses your default Python; 3.10+ is fine)
pip install --upgrade ms-fabric-cli

# Sign in
fab auth login
```

---

## 3. fabric-cicd (Python package)

`fabric-cicd` automates Fabric workspace deployments (notebooks, pipelines, semantic models, etc.) from a Git repo. Install it under **Python 3.13** because it does not yet support 3.14.

**Docs:** <https://microsoft.github.io/fabric-cicd/>

```powershell
# Install / upgrade
py -3.13 -m pip install --upgrade fabric-cicd

# Verify
py -3.13 -m pip show fabric-cicd | Select-String "Version"
```

When you use it in a script, invoke Python explicitly:

```powershell
py -3.13 your_deploy_script.py
```

---

## 4. skills-for-fabric (Copilot CLI plugin)

Adds Microsoft Fabric skills to Copilot CLI: SQL Warehouse, Spark/Lakehouse, Power BI, Eventhouse/KQL, Eventstream, Dataflows Gen2, search, migrations, and end-to-end medallion architecture.

**Repo:** <https://github.com/microsoft/skills-for-fabric>### First-time install (inside Copilot CLI)

```text
/plugin marketplace add microsoft/skills-for-fabric
/plugin install fabric-skills@fabric-collection
```

Or install a focused bundle:

```text
/plugin install fabric-authoring@fabric-collection
/plugin install fabric-consumption@fabric-collection
/plugin install fabric-operations@fabric-collection
```

### Update an already-installed plugin

Inside Copilot CLI:

```text
/plugin update skills-for-fabric@fabric-collection
```

Or update directly via Git (works because the installed plugin folder is a Git clone):

```powershell
$pluginDir = "$env:USERPROFILE\.copilot\installed-plugins\fabric-collection\skills-for-fabric"
if (Test-Path $pluginDir) {
    Push-Location $pluginDir
    git pull --ff-only
    Pop-Location
}
```

Then **restart Copilot CLI** so the new skills are loaded.

### Check current version

```powershell
$pkg = "$env:USERPROFILE\.copilot\installed-plugins\fabric-collection\skills-for-fabric\package.json"
if (Test-Path $pkg) { (Get-Content $pkg | ConvertFrom-Json).version }
```

---

## 5. powerbi-agentic-plugins (RuiRomano) — REQUIRED for Power BI / PBIR work

Adds two Copilot plugins that target the schemas Fabric service actually accepts today (`definitionProperties/2.0.0`, `report/3.0.0`, `visualContainer/2.4.0`, `version 4.0`). Use these for any TMDL semantic-model authoring, PBIR report authoring, DAX, or workspace navigation.

**Repo:** <https://github.com/RuiRomano/powerbi-agentic-plugins>

> ⚠ **Do NOT use `kpbray/power-bi-agent-skills` anymore.** Its
> `report-visuals` 1.0.0 schemas are rejected by Fabric service, and
> its `pbip-project` skeleton conflicts with the layout we now use.
> Two roadshow runs lost ~20 min each fighting this. RuiRomano's
> templates are correct first try.

### Install (inside Copilot CLI)

```text
/plugin marketplace add RuiRomano/powerbi-agentic-plugins
/plugin install powerbi@powerbi-agentic-plugins
/plugin install fabric@powerbi-agentic-plugins
```

Or from the host shell:

```powershell
copilot plugin marketplace add RuiRomano/powerbi-agentic-plugins
copilot plugin install powerbi@powerbi-agentic-plugins
copilot plugin install fabric@powerbi-agentic-plugins
```

After install, **restart Copilot CLI** so the skills load.

### What each plugin gives you

| Plugin | Use it for |
|---|---|
| `powerbi@powerbi-agentic-plugins` | Semantic-model authoring (TMDL), PBIR enhanced-format report authoring, DAX query/measure authoring, modelling best practices |
| `fabric@powerbi-agentic-plugins`  | Workspace navigation, item import/export via REST, Fabric/Power BI REST helpers, OneLake file operations |

### Check installed versions

```powershell
copilot plugin list | Select-String "powerbi-agentic-plugins"
```

---

## Quick Install Script

Save this block as `install-fabric.ps1` (or paste straight into PowerShell). It is idempotent — safe to re-run.

```powershell
Write-Host "=== Microsoft Fabric — Copilot CLI setup ===" -ForegroundColor Cyan

# 1. Python 3.13 ---------------------------------------------------------------
Write-Host "`n[1/4] Python 3.13" -ForegroundColor Yellow
$py313 = & py -3.13 --version 2>$null
if (-not $py313) {
    Write-Host "Installing Python 3.13 via py launcher..."
    py install 3.13
} else {
    Write-Host "Found: $py313"
}

# 2. Fabric CLI ----------------------------------------------------------------
Write-Host "`n[2/4] Fabric CLI (fab)" -ForegroundColor Yellow
$fabVer = & fab --version 2>$null | Select-String "fab version" | Select-Object -First 1
if (-not $fabVer) {
    Write-Host "Installing ms-fabric-cli..."
    pip install --upgrade ms-fabric-cli
} else {
    Write-Host "Found: $fabVer"
    Write-Host "Upgrading ms-fabric-cli..."
    pip install --upgrade ms-fabric-cli --quiet
}

# 3. fabric-cicd (Python 3.13) -------------------------------------------------
Write-Host "`n[3/4] fabric-cicd (Python 3.13)" -ForegroundColor Yellow
py -3.13 -m pip install --upgrade fabric-cicd --quiet
py -3.13 -m pip show fabric-cicd | Select-String "Version"

# 4. skills-for-fabric ---------------------------------------------------------
Write-Host "`n[4/4] skills-for-fabric Copilot plugin" -ForegroundColor Yellow
$pluginDir = "$env:USERPROFILE\.copilot\installed-plugins\fabric-collection\skills-for-fabric"
if (Test-Path $pluginDir) {
    Push-Location $pluginDir
    git pull --ff-only
    $localVer = (Get-Content package.json | ConvertFrom-Json).version
    Write-Host "skills-for-fabric is at v$localVer"
    Pop-Location
} else {
    Write-Host "Plugin not yet installed. In Copilot CLI run:" -ForegroundColor Magenta
    Write-Host "  /plugin marketplace add microsoft/skills-for-fabric"
    Write-Host "  /plugin install fabric-skills@fabric-collection"
}

# 5. powerbi-agentic-plugins (RuiRomano) --------------------------------------
Write-Host "`n[5/5] powerbi-agentic-plugins (RuiRomano)" -ForegroundColor Yellow
$installed = (& copilot plugin list 2>$null) -join "`n"
if ($installed -match "powerbi-agentic-plugins") {
    Write-Host "Already installed:"
    $installed -split "`n" | Select-String "powerbi-agentic-plugins"
} else {
    Write-Host "Installing RuiRomano plugins..."
    copilot plugin marketplace add RuiRomano/powerbi-agentic-plugins | Out-Null
    copilot plugin install powerbi@powerbi-agentic-plugins
    copilot plugin install fabric@powerbi-agentic-plugins
}

Write-Host "`n=== Done. Restart Copilot CLI to pick up plugin changes. ===" -ForegroundColor Green
```

---

## Verify

```powershell
python --version
py -3.13 --version
fab --version
py -3.13 -m pip show fabric-cicd | Select-String "Version"
$pkg = "$env:USERPROFILE\.copilot\installed-plugins\fabric-collection\skills-for-fabric\package.json"
if (Test-Path $pkg) { "skills-for-fabric: v$((Get-Content $pkg | ConvertFrom-Json).version)" } else { "skills-for-fabric: not installed" }
copilot plugin list | Select-String "powerbi-agentic-plugins"
```

---

## Troubleshooting

**`fabric-cicd` fails with `Could not find a version that satisfies the requirement`**
Your default `pip` is using Python 3.14+. Install with `py -3.13 -m pip install fabric-cicd` instead.

**New Fabric skills don't show up in Copilot CLI after update**
Restart the Copilot CLI process — skills are loaded at session start.

**`fab auth login` fails**
Make sure `az login` succeeded first and that your account has access to a Fabric workspace.

---

## License

MIT
