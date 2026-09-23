# GitHub Copilot CLI - Initial Setup Guide

Everything you need to install and configure before using
[GitHub Copilot CLI](https://docs.github.com/en/copilot/concepts/agents/copilot-cli/about-copilot-cli)
for Microsoft Fabric development. Use this guide after a fresh Windows laptop
install or to onboard new team members.

The commands below install the latest stable release unless a specific version
is intentionally required. Python is pinned to the 3.13 release line to match
the current Fabric Runtime guidance in
[skills-for-fabric](https://github.com/microsoft/skills-for-fabric).

---

## Prerequisites

### 1. PowerShell 7

GitHub Copilot CLI requires PowerShell 6 or later on Windows. PowerShell 7
(`pwsh`) is also recommended by the Windows custom statusline.

**Install:** <https://learn.microsoft.com/powershell/scripting/install/installing-powershell-on-windows>

```powershell
# Install the latest stable PowerShell 7
winget install --exact --id Microsoft.PowerShell

# Check / upgrade
pwsh --version
winget upgrade --exact --id Microsoft.PowerShell
```

Restart the terminal after installing PowerShell for the first time.

---

### 2. GitHub Copilot CLI

Copilot CLI requires an active GitHub Copilot subscription. Organization and
enterprise policies must also allow Copilot CLI.

**Install documentation:** <https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/install-copilot-cli>

```powershell
# Install the latest stable release
winget install --exact --id GitHub.Copilot

# Check / upgrade
copilot --version
winget upgrade --exact --id GitHub.Copilot

# Recommended interactive authentication
copilot login
```

You can also start `copilot` and run `/login`. Copilot CLI may use an
authenticated GitHub CLI session as a fallback, but its own OAuth login is the
recommended interactive method.

---

### 3. Azure CLI

The Azure CLI authenticates Fabric and Azure operations. The custom statusline
can also use it to show the active Azure account.

**Install:** <https://learn.microsoft.com/cli/azure/install-azure-cli>

```powershell
# Install the latest stable release
winget install --exact --id Microsoft.AzureCLI

# Check / upgrade
az --version
az upgrade --yes

# Sign in
az login
az account show --output table
```

---

### 4. GitHub CLI

The GitHub CLI provides repository, issue, and pull-request commands. Copilot
CLI can use its authenticated token as a fallback, and the custom statusline
uses it to show the active GitHub identity.

**Install:** <https://cli.github.com/>

```powershell
# Install the latest stable release
winget install --exact --id GitHub.cli

# Check / upgrade
gh --version
winget upgrade --exact --id GitHub.cli

# Sign in
gh auth login
gh auth status
```

---

### 5. Git

Git is required to clone repositories, for the `skills-for-fabric` Copilot CLI
plugin (installed as a Git checkout), and by `2_Fab_Init.md`.

**Install:** <https://git-scm.com/downloads/win>

```powershell
# Install the latest stable release
winget install --exact --id Git.Git

# Check / upgrade
git --version
winget upgrade --exact --id Git.Git
```

Restart the terminal after installing Git for the first time so `git` is
picked up on `PATH`.

---

### 6. Node.js

The npm installation method for Copilot CLI requires Node.js 22 or later.
Use the latest LTS release; do not install the end-of-life Node.js 18 or 20
release lines.

**Release status:** <https://nodejs.org/en/about/previous-releases>

```powershell
# Install the latest Node.js LTS release
winget install --exact --id OpenJS.NodeJS.LTS

# Check / upgrade
node --version
npm --version
winget upgrade --exact --id OpenJS.NodeJS.LTS
```

If WinGet reports that another Node.js package is already installed, remove or
upgrade that installation rather than keeping multiple unmanaged copies.

---

### 7. Python 3.13

Use Python 3.13 for this Fabric development setup. It matches the current
Python baseline documented for Fabric Runtime in the
[skills-for-fabric data-engineering guidance](https://github.com/microsoft/skills-for-fabric/blob/main/skills/spark-cli/references/authoring/resources/data-engineering-patterns.md).

```powershell
# Install the latest patch release in the Python 3.13 line
winget install --exact --id Python.Python.3.13

# Verify Python 3.13 specifically, even if other versions are installed
py -3.13 --version

# Upgrade to the latest available Python 3.13 patch
winget upgrade --exact --id Python.Python.3.13
```

Use `py -3.13` when a command must run against this exact Python release.

---

### 8. uv (Python package manager)

uv is used to install Spec Kit in an isolated tool environment.

**Install:** <https://docs.astral.sh/uv/getting-started/installation/>

```powershell
# Install the latest stable release
winget install --exact --id astral-sh.uv

# Check / upgrade
uv --version
winget upgrade --exact --id astral-sh.uv
```

---

## Tools

### 9. AI Engineering Fluency CLI

Tracks GitHub Copilot token usage across editors and command-line tools.

**Repo:** <https://github.com/rajbos/ai-engineering-fluency>

**npm:** <https://www.npmjs.com/package/@rajbos/ai-engineering-fluency>

```powershell
# Install or upgrade to the latest stable release
npm install --global @rajbos/ai-engineering-fluency@latest

# Check version and show usage
ai-engineering-fluency --version
ai-engineering-fluency usage
```

---

### 10. OpenSpec

A lightweight spec framework for AI-assisted development. It helps teams agree
on what to build before writing code.

**Repo:** <https://github.com/Fission-AI/OpenSpec>

**npm:** <https://www.npmjs.com/package/@fission-ai/openspec>

```powershell
# Install or upgrade to the latest stable release
npm install --global @fission-ai/openspec@latest

# Check version
openspec --version

# Initialize in a project
Set-Location .\your-project
openspec init
```

---

### 11. Spec Kit (Specify CLI)

GitHub's spec-driven development toolkit. The current recommended installation
uses the published `specify-cli` package.

**Repo:** <https://github.com/github/spec-kit>

**Quick start:** <https://github.com/github/spec-kit/blob/main/docs/quickstart.md>

```powershell
# Install the latest stable release from the package registry
uv tool install specify-cli

# Check version
specify version

# Upgrade an existing installation
uv tool upgrade specify-cli

# Initialize in the current project for GitHub Copilot
specify init . --integration copilot
```

For reproducible environments, pin an exact published release instead of
installing the development branch:

```powershell
# Replace X.Y.Z with a release from the Spec Kit releases page
uv tool install "specify-cli==X.Y.Z"
```

Releases: <https://github.com/github/spec-kit/releases>

---

### 12. Handy (speech-to-text)

A free, open-source, offline speech-to-text desktop app. Press a shortcut,
speak, and the words are typed into the active text field.

**Repo:** <https://github.com/cjpais/Handy>

**Website:** <https://handy.computer>

```powershell
# Install the latest release available through WinGet
winget install --exact --id cjpais.Handy

# Check / upgrade
winget list --exact --id cjpais.Handy
winget upgrade --exact --id cjpais.Handy
```

The WinGet package is community maintained rather than maintained by the Handy
developers. After installing, launch Handy and configure:

- Microphone permissions
- Accessibility permissions
- Your preferred keyboard shortcut

---

## MCP servers

[Model Context Protocol](https://docs.github.com/en/copilot/concepts/context/mcp)
(MCP) servers give Copilot CLI extra tools and up-to-date context. Add these
two for Fabric development: the GitHub MCP server (repository, issue, and pull
request context) and the Microsoft Learn MCP server (live Microsoft/Fabric
documentation search).

**Docs:** <https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-mcp-servers>

### GitHub MCP server

The GitHub MCP server is built into Copilot CLI and available by default with
no configuration. Only add it explicitly if you need a customized toolset
(for example, a limited set of toolsets or read-only mode) via the
Docker-based server instead of the built-in one.

**Repo:** <https://github.com/github/github-mcp-server>

```powershell
# Optional: add the Docker-based GitHub MCP server instead of the built-in one
copilot mcp add github --env GITHUB_PERSONAL_ACCESS_TOKEN=YOUR_GITHUB_PAT -- docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server
```

### Microsoft Learn MCP server

The Microsoft Learn MCP server is a free, public, remote HTTP server. No
authentication is required. It exposes `microsoft_docs_search`,
`microsoft_docs_fetch`, and `microsoft_code_sample_search`, which help Copilot
CLI ground answers in current Microsoft Learn and Fabric documentation instead
of relying on training data.

**Docs:** <https://learn.microsoft.com/training/support/mcp>

```powershell
# Add the remote Microsoft Learn MCP server
copilot mcp add --transport http microsoft-learn https://learn.microsoft.com/api/mcp
```

### Verify

```powershell
copilot mcp list
```

Or, inside an interactive `copilot` session, run `/mcp show`.

---

## Custom statusline

The third-party custom statusline shows useful context at the bottom of Copilot
CLI, including context-window usage, Azure account, GitHub user, token usage,
and active specs.

**Repo:** <https://github.com/pascalvanderheiden/copilot-cli-custom-statusline>

**Windows instructions:** <https://github.com/pascalvanderheiden/copilot-cli-custom-statusline/blob/main/windows/README.md>

```text
space hold to record | 21:57:15 | ctx: 42.1k/200k 21% | az: user@example.com | gh: username | openspec: change 47%
```

> [!IMPORTANT]
> This setup downloads and automatically executes third-party scripts whenever
> Copilot CLI renders the statusline. Review both files before enabling them.
> Using `main` downloads the newest code but is not reproducible; replace
> `main` with a reviewed commit SHA when a stable, auditable installation is
> more important than receiving updates immediately.

### Install the statusline on Windows

1. Download both required Windows files:

   ```powershell
   $destination = "$env:USERPROFILE\.copilot"
   $revision = "main" # Replace with a reviewed commit SHA to pin the code
   $baseUrl = "https://raw.githubusercontent.com/pascalvanderheiden/copilot-cli-custom-statusline/$revision/windows"

   New-Item -ItemType Directory -Force -Path $destination | Out-Null
   Invoke-WebRequest "$baseUrl/statusline.cmd" -OutFile "$destination\statusline.cmd"
   Invoke-WebRequest "$baseUrl/statusline.ps1" -OutFile "$destination\statusline.ps1"

   # Review the downloaded scripts before enabling automatic execution
   Get-Content "$destination\statusline.cmd"
   Get-Content "$destination\statusline.ps1"
   Get-FileHash "$destination\statusline.cmd", "$destination\statusline.ps1"
   ```

2. Edit `%USERPROFILE%\.copilot\settings.json` and merge all three required
   blocks into any existing settings. Replace `<your-user>` with your Windows
   profile folder name:

   ```json
   {
     "experimental": true,
     "statusLine": {
       "type": "command",
       "command": "C:\\Users\\<your-user>\\.copilot\\statusline.cmd",
       "padding": 1
     },
     "feature_flags": {
       "enabled": ["STATUS_LINE"]
     }
   }
   ```

   Do not overwrite unrelated settings that are already in the file.

3. Test the wrapper with a sample payload:

   ```powershell
   '{"cwd":"C:\\Users\\you","context_window":{"current_context_tokens":42137,"displayed_context_limit":200000,"current_context_used_percentage":21.07},"cost":{"total_duration_ms":125300,"total_lines_added":17,"total_lines_removed":4}}' |
     & "$env:USERPROFILE\.copilot\statusline.cmd"
   ```

4. In Copilot CLI, run `/statusline`, select **Custom**, and then run `/restart`.

If the wrapper reports that `pwsh` is not available, install PowerShell 7 as
described above. The upstream Windows README contains additional diagnostics
for blank output, encoding problems, and slow statusline segments.

### Statusline segments

| Segment | What it shows |
| --- | --- |
| `space hold to record` | Handy voice-recording reminder |
| `HH:MM:SS` | Time the statusline was refreshed |
| `ctx: ...` | Current Copilot context-window usage |
| `az: ...` | Signed-in Azure CLI account |
| `gh: ...` | Active GitHub CLI account |
| `tokens<30d: ...` | Last 30 days token usage |
| `subtasks: ...` | Running Copilot subagents |
| `squad: ...` | Active Squad/AI-team context |
| `openspec: ...` | OpenSpec change progress |
| `spec-kit: ...` | Spec Kit task progress |

Segments only appear when their related tool is installed, configured, and
authenticated.

---

## Quick check script

This script reports missing tools and verifies the required Node.js and Python
release lines. It does not change the machine.

```powershell
$checks = @(
    @{ Name = "PowerShell 7"; Command = "pwsh"; Arguments = @("--version") }
    @{ Name = "GitHub Copilot CLI"; Command = "copilot"; Arguments = @("--version") }
    @{ Name = "Azure CLI"; Command = "az"; Arguments = @("--version") }
    @{ Name = "GitHub CLI"; Command = "gh"; Arguments = @("--version") }
    @{ Name = "Git"; Command = "git"; Arguments = @("--version") }
    @{ Name = "Node.js"; Command = "node"; Arguments = @("--version") }
    @{ Name = "npm"; Command = "npm"; Arguments = @("--version") }
    @{ Name = "uv"; Command = "uv"; Arguments = @("--version") }
    @{ Name = "AI Engineering Fluency"; Command = "ai-engineering-fluency"; Arguments = @("--version") }
    @{ Name = "OpenSpec"; Command = "openspec"; Arguments = @("--version") }
    @{ Name = "Spec Kit"; Command = "specify"; Arguments = @("version") }
)

foreach ($check in $checks) {
    Write-Host "`n[$($check.Name)]" -ForegroundColor Yellow
    if (Get-Command $check.Command -ErrorAction SilentlyContinue) {
        $commandArguments = $check.Arguments
        & $check.Command @commandArguments 2>$null | Select-Object -First 3
    } else {
        Write-Host "$($check.Command) NOT found" -ForegroundColor Red
    }
}

Write-Host "`n[Required versions]" -ForegroundColor Yellow

$nodeVersion = if (Get-Command node -ErrorAction SilentlyContinue) {
    [version]((node --version).TrimStart("v"))
}
if (-not $nodeVersion -or $nodeVersion.Major -lt 22) {
    Write-Host "Node.js 22+ is required" -ForegroundColor Red
} else {
    Write-Host "Node.js $nodeVersion is supported"
}

$pythonVersion = if (Get-Command py -ErrorAction SilentlyContinue) {
    $rawPythonVersion = py -3.13 --version 2>&1
    if ($LASTEXITCODE -eq 0 -and $rawPythonVersion -match "Python (\d+\.\d+\.\d+)") {
        [version]$Matches[1]
    }
}
if (-not $pythonVersion -or $pythonVersion.Major -ne 3 -or $pythonVersion.Minor -ne 13) {
    Write-Host "Python 3.13 is required for this Fabric setup" -ForegroundColor Red
} else {
    Write-Host "Python $pythonVersion is installed"
}

Write-Host "`n[Handy]" -ForegroundColor Yellow
winget list --exact --id cjpais.Handy 2>$null

Write-Host "`n[Custom statusline]" -ForegroundColor Yellow
$statuslineFiles = @(
    "$env:USERPROFILE\.copilot\statusline.cmd"
    "$env:USERPROFILE\.copilot\statusline.ps1"
)
$missingStatuslineFiles = $statuslineFiles | Where-Object { -not (Test-Path $_) }
if ($missingStatuslineFiles) {
    $missingStatuslineFiles | ForEach-Object {
        Write-Host "$_ NOT found" -ForegroundColor Red
    }
} else {
    Write-Host "Windows statusline files found"
}
```

---

## Keeping tools current

Run these commands periodically:

```powershell
winget upgrade --exact --id Microsoft.PowerShell
winget upgrade --exact --id GitHub.Copilot
winget upgrade --exact --id Microsoft.AzureCLI
winget upgrade --exact --id GitHub.cli
winget upgrade --exact --id Git.Git
winget upgrade --exact --id OpenJS.NodeJS.LTS
winget upgrade --exact --id Python.Python.3.13
winget upgrade --exact --id astral-sh.uv
winget upgrade --exact --id cjpais.Handy

npm install --global @rajbos/ai-engineering-fluency@latest
npm install --global @fission-ai/openspec@latest
uv tool upgrade specify-cli
```

WinGet, npm, and uv can only install versions currently available from their
configured sources. A command succeeding therefore means "latest available
from that source," not necessarily that every upstream project published its
newest release to that source immediately.

---

## License

MIT
