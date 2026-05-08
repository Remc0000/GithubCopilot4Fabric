# GitHub Copilot CLI — Initial Setup Guide

Everything you need to install and configure before using [GitHub Copilot CLI](https://githubnext.com/projects/copilot-cli) effectively. Use this guide after a fresh laptop install or to onboard new team members.

---

## Prerequisites

### 1. Azure CLI

The Azure CLI lets Copilot CLI show your active Azure account and interact with Azure resources.

**Install:** <https://learn.microsoft.com/cli/azure/install-azure-cli>

```powershell
# Check if installed
az --version

# Upgrade to latest
az upgrade --yes

# Sign in
az login
```

---

### 2. GitHub CLI

The GitHub CLI enables Copilot CLI to show your GitHub identity and interact with repos, issues, and PRs.

**Install:** <https://cli.github.com/>

```powershell
# Check if installed
gh --version

# Upgrade via winget (Windows)
winget upgrade --id GitHub.cli

# Sign in
gh auth login
```

---

### 3. Node.js

Node.js 18+ is required for several tools below. If you don't have it yet:

**Install:** <https://nodejs.org/>

```powershell
node --version
```

---

### 4. uv (Python package manager)

uv is needed to install Spec Kit. It's a fast alternative to pip/pipx.

**Install:** <https://docs.astral.sh/uv/>

```powershell
uv --version
```

---

## Tools

### 5. AI Engineering Fluency CLI

Tracks your GitHub Copilot token usage across editors (VS Code, Copilot CLI, OpenCode, etc.).

**Repo:** <https://github.com/rajbos/ai-engineering-fluency>
**npm:** [@rajbos/ai-engineering-fluency](https://www.npmjs.com/package/@rajbos/ai-engineering-fluency)

```powershell
# Install globally
npm install -g @rajbos/ai-engineering-fluency

# Check / upgrade
ai-engineering-fluency --version
npm install -g @rajbos/ai-engineering-fluency@latest

# Example: show token usage
ai-engineering-fluency usage
```

---

### 6. OpenSpec

A lightweight spec framework for AI-assisted development. Helps you agree on what to build before writing code.

**Repo:** <https://github.com/Fission-AI/OpenSpec>
**npm:** [@fission-ai/openspec](https://www.npmjs.com/package/@fission-ai/openspec)

```powershell
# Install globally
npm install -g @fission-ai/openspec@latest

# Check / upgrade
openspec --version
npm install -g @fission-ai/openspec@latest

# Initialize in a project
cd your-project
openspec init
```

---

### 7. Spec Kit (Specify CLI)

GitHub's spec-driven development toolkit. Focuses on product scenarios and predictable outcomes.

**Repo:** <https://github.com/github/spec-kit>

> **Note:** Install from GitHub directly — not from PyPI.

```powershell
# Install a specific release (recommended — replace vX.Y.Z with latest tag)
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@vX.Y.Z

# Or install latest from main
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git

# Check version
specify version

# Upgrade (use --force to overwrite)
uv tool install specify-cli --force --from git+https://github.com/github/spec-kit.git@vX.Y.Z

# Initialize in a project
specify init . --integration copilot
```

Find the latest release tag at: <https://github.com/github/spec-kit/releases>

---

### 8. Handy (Speech-to-Text)

A free, open-source, offline speech-to-text desktop app. Press a shortcut, speak, and your words are typed into any text field.

**Repo:** <https://github.com/cjpais/Handy>
**Website:** <https://handy.computer>

```powershell
# Install via winget (Windows)
winget install cjpais.Handy

# Check if installed
winget list --id cjpais.Handy

# Upgrade
winget upgrade --id cjpais.Handy
```

After installing, launch Handy and configure:
- Microphone permissions
- Accessibility permissions
- Your preferred keyboard shortcut (default: hold Space to record)

---

## Custom Statusline

The custom statusline shows useful context at the bottom of Copilot CLI — Azure account, GitHub user, token usage, active specs, and more.

**Repo:** <https://github.com/pascalvanderheiden/copilot-cli-custom-statusline>

```text
space hold to record · ↻ 21:57:15 · az: user@example.com · gh: username · tokens<30d: 898.9K · squad: repo · openspec: change 47%
```

### Install the statusline (Windows)

1. **Enable the statusline** in Copilot CLI:

   ```text
   /statusline
   ```

   Enable the custom statusline when prompted.

2. **Copy the script** to your `.copilot` folder:

   ```powershell
   New-Item -ItemType Directory -Force "$env:USERPROFILE\.copilot"

   # Download the latest script
   Invoke-WebRequest -Uri "https://raw.githubusercontent.com/pascalvanderheiden/copilot-cli-custom-statusline/main/statusline.ps1" `
     -OutFile "$env:USERPROFILE\.copilot\statusline.ps1"
   ```

3. **Configure** `%USERPROFILE%\.copilot\settings.json` — add the `statusLine` block:

   ```json
   {
     "statusLine": {
       "type": "command",
       "command": "powershell.exe -NoProfile -ExecutionPolicy Bypass -File %USERPROFILE%\\.copilot\\statusline.ps1"
     }
   }
   ```

4. **Test** the script:

   ```powershell
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File "$env:USERPROFILE\.copilot\statusline.ps1"
   ```

5. **Restart** Copilot CLI to see the statusline.

### Statusline segments

| Segment | What it shows |
| --- | --- |
| `space hold to record` | Handy voice-recording reminder |
| `↻ HH:MM:SS` | Time the statusline was refreshed |
| `az: ...` | Signed-in Azure CLI account |
| `gh: ...` | Active GitHub CLI account |
| `tokens<30d: ...` | Last 30 days token usage |
| `subtasks: ...` | Running Copilot subagents |
| `squad: ...` | Active Squad/AI-team context |
| `openspec: ...` | OpenSpec change progress |
| `spec-kit: ...` | Spec Kit task progress |

Segments only appear when their related tool is installed and authenticated.

---

## Quick Check Script

Run this to verify everything is installed:

```powershell
Write-Host "=== GitHub Copilot CLI Prerequisites ===" -ForegroundColor Cyan

Write-Host "`n[Azure CLI]" -ForegroundColor Yellow
az --version 2>$null | Select-Object -First 1

Write-Host "`n[GitHub CLI]" -ForegroundColor Yellow
gh --version 2>$null

Write-Host "`n[Node.js]" -ForegroundColor Yellow
node --version 2>$null

Write-Host "`n[uv]" -ForegroundColor Yellow
uv --version 2>$null

Write-Host "`n=== Tools ===" -ForegroundColor Cyan

Write-Host "`n[AI Engineering Fluency]" -ForegroundColor Yellow
ai-engineering-fluency --version 2>$null

Write-Host "`n[OpenSpec]" -ForegroundColor Yellow
openspec --version 2>$null

Write-Host "`n[Spec Kit]" -ForegroundColor Yellow
specify version 2>$null

Write-Host "`n[Handy]" -ForegroundColor Yellow
winget list --id cjpais.Handy 2>$null

Write-Host "`n[Custom Statusline]" -ForegroundColor Yellow
if (Test-Path "$env:USERPROFILE\.copilot\statusline.ps1") {
    Write-Host "statusline.ps1 found"
} else {
    Write-Host "statusline.ps1 NOT found" -ForegroundColor Red
}
```

---

## License

MIT
