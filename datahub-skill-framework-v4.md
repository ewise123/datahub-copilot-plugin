# DataHub Copilot Plugin
## Architecture, Distribution, and Maintenance (v4)

---

## 1. What Changed from v3

v3 designed a two-tier system: routing rules in `.instructions.md` dispatch to specialist agents, which then read upstream and platform skills. Testing revealed a fundamental problem — Copilot doesn't reliably follow the routing layer. When a developer says "I need to do Dagster work," Copilot finds the skill with the best keyword match and reads it directly, bypassing both the routing table and the agent.

The agent layer was adding complexity that Copilot actively fights against. v4 drops agents and routing entirely.

**The framework is now a plugin** — an installable, versioned package of curated skills and reference knowledge that ships DataHub expertise to every developer's IDE. The "code" is markdown files. The "runtime" is Copilot's skill discovery. The "package manager" is a setup script, an ADO pipeline, and a drift scanner. A developer runs `datahub-update`, restarts VS Code, and Copilot knows how DataHub works. They don't need to know about skill directories, reference files, or how the content is structured.

Further testing confirmed that Copilot reads multiple skills in a single interaction when a task spans domains — eliminating the last argument for an agent or orchestration layer. The simplest possible architecture is also the correct one.

This makes the framework simpler to build and maintain, but shifts the critical challenge to **distribution and freshness** — keeping every developer's plugin current is now the entire operational surface area.

---

## 2. What We've Tested and Confirmed

| Assumption | Test | Result |
|-----------|------|--------|
| Skills auto-discovered from `~/.copilot/skills/` with nested reference subdirectories | Created test skill with nested references containing a unique phrase not in any training data | ✅ Copilot returned the planted phrase — skill discovery with nested refs works |
| User-level agents roam across workspaces | Agent in `prompts/` tested across different ADO-hosted workspaces | ✅ Works regardless of which workspace is open |
| Copilot reads skills based on keyword matching without routing | Installed dagster-expert skill, told Copilot "I need to do dagster work" with no routing table in place | ✅ Copilot matched keywords directly to skill and read it — confirms skills-only architecture works |
| Copilot reads multiple skills in a single interaction | Installed datahub-dlt and dagster-expert skills, prompted: "Create a new dlt source for ServiceNow and wire it up as a Dagster asset on a daily schedule" | ✅ Copilot identified both domains, read both skill files, and combined knowledge from both. No agent or orchestration layer needed. |
| Copilot respects skill constraints in unfamiliar workspace | Same multi-skill test run in a non-DataHub repo | ✅ Copilot attempted to find expected workspace files (sources/p8e-data-source-*/, pyproject.toml) and correctly identified they were missing — constraints are being followed |

The multi-skill test is the most important validation. It confirms that Copilot natively handles cross-domain tasks without agents, routing tables, or orchestration — the simplest possible architecture works.

---

## 3. The Problem This Solves

Developers working in the DataHub platform need to understand how 15 repositories, three deployment paths, and a dozen tools connect together. Today, that knowledge lives in people's heads, scattered docs, and tribal knowledge. When a developer needs to add a new dlt source, fix a Dagster deploy, or create an infrastructure stack, they spend time figuring out which repos to touch, which templates to follow, and which patterns to use.

This plugin puts that knowledge directly into the developer's IDE, scoped to exactly what they're working on. Install the plugin, and Copilot becomes a DataHub expert.

---

## 4. Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  USER LEVEL (distributed via central repo)                              │
│                                                                         │
│  .instructions.md            — slim DataHub mental model (read on every │
│  Location: prompts/            interaction): deployment lanes, datalake  │
│                                layers, repo count. ~150 words.          │
│                                                                         │
│  Skills + References                                                    │
│  Location: C:\Users\{user}\.copilot\skills\                           │
│                                                                         │
│  datahub-dagster/          — Dagster + DataHub orchestration patterns    │
│    SKILL.md                — complete skill: knowledge + constraints     │
│    references/             — supporting detail files                     │
│                                                                         │
│  datahub-dlt/              — dlt + DataHub ingestion patterns            │
│    SKILL.md                                                             │
│    references/                                                          │
│                                                                         │
│  datahub-dbt/              — dbt + DataHub transformation patterns       │
│    SKILL.md                                                             │
│    references/                                                          │
│                                                                         │
│  datahub-infra/            — Terragrunt/OpenTofu + DataHub IaC          │
│    SKILL.md                                                             │
│    references/                                                          │
│                                                                         │
│  datahub-k8s/              — K8s/AKS + DataHub deployment               │
│    SKILL.md                                                             │
│    references/                                                          │
│                                                                         │
│  datahub-contracts/        — ODCS data contracts                        │
│    SKILL.md                                                             │
│    references/                                                          │
│                                                                         │
│  datahub-platform/         — Cross-cutting reference files (NOT a skill) │
│    references/             — no SKILL.md; domain skills read these by   │
│      architecture.md         path when they need platform context       │
│      datalake-layers.md                                                 │
│      deployment-k8s.md                                                  │
│      deployment-dagster.md                                              │
│      deployment-infra.md                                                │
│      template-map.md                                                    │
│      pr-workflow.md                                                     │
│                                                                         │
│  datahub-version.json      — installed version + timestamp              │
├─────────────────────────────────────────────────────────────────────────┤
│  WORKSPACE LEVEL (per-repo, version controlled in ADO)                  │
│  Location: {repo}/.github/                                              │
│                                                                         │
│  copilot-instructions.md   — light repo context: identity, standards,   │
│                              file structure, what not to do              │
└─────────────────────────────────────────────────────────────────────────┘
```

### Three Layers of Context

**Layer 1: .instructions.md (every interaction)**
A slim (~150 word) DataHub mental model that Copilot reads on every interaction. Gives Copilot the 30-second orientation: DataHub has 15 repos across three deployment lanes, the datalake has three layers, here's the vocabulary. This is NOT a routing table — it's ambient context so that any Copilot interaction has baseline DataHub awareness.

```markdown
# DataHub Platform Context

DataHub is a data platform with 15 repos across three deployment lanes:
- **Data pipelines:** dlt (ingestion → Raw) + dbt (transform → Prep/Prod),
  orchestrated by Dagster, deployed to Dagster Cloud
- **Services:** APIs and tools deployed to AKS via Helm + FluxCD
- **Infrastructure:** Azure resources provisioned via Terragrunt/OpenTofu

The datalake has three layers in Databricks Unity Catalog:
- **Raw:** landing zone for dlt-ingested API data
- **Prep:** dbt-transformed staging models
- **Prod:** dbt mart models, consumer-facing
```

Note: the `.instructions.md` intentionally does NOT point to the platform reference directory. Including a path like "detailed files are at ~/.copilot/skills/datahub-platform/references/" would invite Copilot to proactively read those files on every interaction. Instead, only domain skills contain those paths, so platform references are only loaded when a matching skill asks for them.

**Layer 2: Domain skills (matched by task keywords)**
Curated skills that Copilot discovers and reads when keywords match. Each skill contains upstream tool knowledge filtered through DataHub conventions, constraints, approach patterns, and output guidance.

**Layer 3: Per-repo instructions (workspace-scoped)**
Light, stable facts about the specific repo the developer has open. File structure, coding standards, what not to do.

### How It Works

1. Developer opens any DataHub repo in VS Code
2. Copilot reads `.instructions.md` (user-level) — gets the DataHub mental model
3. Copilot reads `.github/copilot-instructions.md` (workspace-level) — gets repo-specific context
4. Developer describes a task in Copilot Chat
5. Copilot matches task keywords to a domain skill and reads it
6. The skill provides: domain knowledge, DataHub-specific constraints, workspace files to examine, approach, and output guidance
7. The skill may direct Copilot to read specific platform reference files for cross-cutting context
8. Developer reviews the result

### What the Developer Experiences

Developer types: "I need to add a new dlt source for ServiceNow with incidents and change requests"

Copilot automatically:
- Matches "dlt source" → reads `datahub-dlt` skill
- Gets upstream dlt knowledge (API patterns, @dlt.source conventions) already filtered for DataHub
- Gets platform context (Raw layer conventions, repo structure)
- Reads existing sources in workspace (p8e-data-source-* patterns)
- Returns scaffolded source package with all files, tests, and next steps

No routing layer, no agent delegation, no `@` tags. The developer describes work, Copilot finds the skill.

---

## 5. Skill Design

### Structure

Every skill follows the same internal structure. This is critical — the skill must carry everything that was previously split across agent definitions, upstream skills, and platform skills.

```markdown
---
name: datahub-{domain}
description: >
  {Keyword-rich description that matches how developers describe tasks.
  This is the primary matching surface — get the keywords right.}
---

# DataHub {Domain} Expert

You are a {domain} expert specialized for the DataHub platform.

## CRITICAL: Read reference files before answering. Never answer from memory.

Read from this skill's references/ directory for detailed knowledge.
Read from ~/.copilot/skills/datahub-platform/references/ for cross-cutting
platform context when the task involves deployment, architecture, or datalake layers.

## Constraints

- {What NOT to do — prevents hallucinated patterns}
- {File paths and patterns that must be followed}
- {Integration boundaries — what not to break}
- ALWAYS read existing code in the workspace before generating new code.
  Match the patterns you find.

## Workspace Files to Examine

- {Specific paths in the repo to read for current patterns}

## Approach

1. {Step-by-step procedure for common tasks}
2. {Including which reference files to read}
3. {Including which workspace files to examine}
4. {Verification and testing steps}

## Output Guidance

When completing a task, include:
- Summary of what was done
- Files changed with purpose
- Testing commands and expected results
- Next steps for the developer
```

### Why This Structure Works

The skill description is the keyword magnet — it's what Copilot matches on. The body is the execution context — constraints, approach, output format. This is the same structure the v3 agents had, but living inside the skill that Copilot naturally gravitates toward.

### Curating Upstream Knowledge

Each skill bakes in upstream tool knowledge rather than pointing to a separate upstream skill directory. This prevents the problem we observed — Copilot finding a raw upstream skill and using it without DataHub context.

How to curate:
1. Start with the vendor's upstream skill content
2. Remove anything that contradicts DataHub conventions
3. Add DataHub-specific constraints and patterns
4. Put detailed reference material in the skill's `references/` directory
5. The SKILL.md points to references but contains the constraints and approach directly

When upstream tools release updates, the relevant skill's references get updated — but always filtered through DataHub conventions first.

### Skill Sizing

Copilot's ability to follow instructions degrades as content length increases. Skills that are too large get partially read, and constraints buried deep in the file may be ignored.

**Guidelines:**
- **SKILL.md:** Under 300 lines. This is the constraints, approach, and output guidance — the most important content. Keep it tight.
- **Individual reference files:** Under 500 lines each. If a reference file grows beyond this, split it into more specific files and have the skill point to only the relevant ones per task type.
- **Total skill size** (SKILL.md + all references): Be aware that Copilot likely won't read everything in a single interaction. The skill's Approach section should direct Copilot to specific reference files per task type rather than saying "read all references."
- **Front-load the critical stuff.** Constraints and "what not to do" go at the top of SKILL.md. If Copilot only reads the first half, make sure the guardrails are in it.

### Cross-Cutting Platform Knowledge

Platform knowledge is NOT a skill. It lives in two places:

1. **`.instructions.md`** — the slim mental model (~150 words) that Copilot reads on every interaction. This gives baseline orientation without consuming attention on irrelevant detail.

2. **`~/.copilot/skills/datahub-platform/references/`** — detailed reference files that domain skills point to by path when they need deeper platform context. There is no SKILL.md here — these files are only read when a domain skill explicitly directs Copilot to them.

Domain skills reference platform files like this:

```
When the task involves deployment configuration, read:
~/.copilot/skills/datahub-platform/references/deployment-dagster.md

When creating assets that land in Raw/Prep/Prod, read:
~/.copilot/skills/datahub-platform/references/datalake-layers.md
```

This prevents datahub-platform from competing with domain skills for keyword matches. Copilot never picks up "architecture" or "deployment" and lands on a generic platform overview instead of the domain skill that can actually help.

---

## 6. Per-Repo Instructions

**Location:** `{repo}/.github/copilot-instructions.md`
**Maintained by:** Authored in central repo, distributed to working repos

Light, stable facts. 200-400 words. Covers repo identity, file structure, coding standards, what not to do, deployment lane. Volatile knowledge lives in skills, not here.

These provide workspace context that skills can't — what repo the developer is currently in, what the file structure looks like, what standards apply locally.

---

## 7. The Central Repo

### Purpose

Single source of truth for all skills and repo instruction files. Developers don't work in this repo — it's maintained by tech leads and distributed to developer machines automatically.

### Structure

```
datahub-copilot/                        (ADO repo)
│
├── README.md                           # What this is, how to set up
├── MAINTENANCE.md                      # How to update skills
├── version.json                        # Version metadata (see Distribution)
│
├── setup.ps1                           # PowerShell setup/update script
├── scan.ps1                            # Drift scanner (see Keeping Content Fresh)
│
├── prompts/                            # → AppData\Roaming\Code\User\prompts\
│   └── .instructions.md                # Slim DataHub mental model
│
├── skills/                             # → .copilot\skills\
│   ├── datahub-dagster/
│   │   ├── SKILL.md
│   │   ├── UPSTREAM-SOURCE.md          # Tracks vendor skill version/date
│   │   ├── skill-metadata.json         # Tracked versions for drift scanner
│   │   └── references/
│   │
│   ├── datahub-dlt/
│   │   ├── SKILL.md
│   │   ├── UPSTREAM-SOURCE.md
│   │   ├── skill-metadata.json
│   │   └── references/
│   │
│   ├── datahub-dbt/
│   │   ├── SKILL.md
│   │   ├── UPSTREAM-SOURCE.md
│   │   ├── skill-metadata.json
│   │   └── references/
│   │
│   ├── datahub-infra/
│   │   ├── SKILL.md
│   │   ├── UPSTREAM-SOURCE.md
│   │   ├── skill-metadata.json
│   │   └── references/
│   │
│   ├── datahub-k8s/
│   │   ├── SKILL.md
│   │   └── references/
│   │
│   ├── datahub-contracts/
│   │   ├── SKILL.md
│   │   └── references/
│   │
│   └── datahub-platform/              # NOT a skill — no SKILL.md
│       └── references/                # Read by domain skills via path
│           ├── architecture.md
│           ├── datalake-layers.md
│           ├── deployment-k8s.md
│           ├── deployment-dagster.md
│           ├── deployment-infra.md
│           ├── template-map.md
│           └── pr-workflow.md
│
├── repo-instructions/                  # Source for each repo's copilot-instructions.md
│   ├── data-api.md
│   ├── datahub-data-platform-repo.md
│   ├── data-transformation.md
│   └── ... (one per repo)
│
├── scan-config.json                    # Drift scanner configuration
│
└── tests/                              # Example prompts + expected outputs for QA
    ├── dlt-new-source.md
    ├── dbt-new-model.md
    └── infra-new-stack.md
```

---

## 8. Distribution — The Hard Problem

The framework is only as good as the stalest developer machine. If skills are out of date, developers get wrong answers with high confidence — worse than no framework at all.

### Design Principles

1. **Updates must be automatic by default.** Relying on developers to remember `git pull && .\setup.ps1` will fail within weeks.
2. **Manual update must remain possible.** For air-gapped environments, new machines, or troubleshooting.
3. **Every developer interaction should have a freshness check.** If skills are stale, the developer should know before trusting the output.
4. **Distribution failures must be visible.** Silent staleness is the worst outcome.

### Version Tracking

The plugin uses semantic versioning. Treat it like shipping software — every merge to main gets a version bump.

The central repo maintains `version.json`:

```json
{
  "version": "2.1.0",
  "updated": "2026-03-10T14:30:00Z",
  "changelog": "Updated datahub-dagster for dg CLI v0.28, added branch catalog isolation docs"
}
```

**Versioning rules:**
- **Major** (3.0.0): Breaking changes — skill restructured, file paths moved, conventions changed in ways that invalidate previous skill output
- **Minor** (2.1.0): New skill added, significant content update to existing skill, new platform reference file
- **Patch** (2.0.1): Typo fixes, minor clarifications, wording improvements

The changelog matters. When a developer sees the staleness warning and runs `datahub-update`, the changelog in the notification tells them whether to care. "Updated datahub-dagster for dg CLI v0.28, added branch catalog isolation docs" is useful. "Various improvements" is not.

This file is copied to `~/.copilot/skills/datahub-version.json` on every install/update. Skills reference it for staleness warnings.

### Distribution Layers

Three layers, each catching what the previous one misses:

#### Layer 1: ADO Pipeline — Automated Push Notification

An ADO pipeline in the central repo triggers on merge to `main`. It doesn't push files to developer machines directly (that requires machine access), but it **notifies developers that an update is available**.

Options (choose based on team communication patterns):

**Option A: Teams/Slack webhook**
Pipeline posts to a team channel: "DataHub Copilot skills updated to v2.1.0 — run `datahub-update` to install. Changes: updated datahub-dagster for dg CLI v0.28."

**Option B: ADO dashboard widget**
Pipeline updates a shared dashboard showing current version, last update date, and changelog. Developers check this as part of their workflow.

**Option C: Email notification**
Pipeline sends email to the team distribution list with version and changelog.

All three are simple ADO pipeline tasks. Option A is recommended — it's the most visible and requires the least effort to check.

#### Layer 2: VS Code Task — One-Command Update

Developers run updates via a VS Code task or terminal alias. The setup script registers a command alias on first install:

```powershell
# setup.ps1 registers this alias in the user's PowerShell profile
# After first setup, developer just runs:
datahub-update
```

The `datahub-update` command:

```powershell
# datahub-update function (added to PowerShell profile by setup.ps1)
function datahub-update {
    $RepoPath = "$env:USERPROFILE\datahub-copilot"

    if (-not (Test-Path $RepoPath)) {
        Write-Host "First-time setup: cloning datahub-copilot..." -ForegroundColor Yellow
        git clone https://dev.azure.com/protectivetfsprod/DataHub/_git/datahub-copilot $RepoPath
    }

    Push-Location $RepoPath
    git pull --ff-only
    if ($LASTEXITCODE -ne 0) {
        Write-Host "ERROR: git pull failed. Check your network/VPN connection." -ForegroundColor Red
        Pop-Location
        return
    }
    & "$RepoPath\setup.ps1"
    Pop-Location
}
```

#### Layer 3: Staleness Warning — Skills Self-Report

Every skill includes a staleness check instruction at the top of its body:

```markdown
## Version Check

Before answering, read ~/.copilot/skills/datahub-version.json.
If the "updated" timestamp is more than 14 days old, prepend your response with:

> ⚠️ **Your DataHub skills may be outdated (last updated {date}).**
> Run `datahub-update` in your terminal to get the latest version.

Then continue answering normally.
```

This is the safety net. Even if a developer ignores notifications and never runs the update command, they'll see a warning in every Copilot response once skills go stale. The warning is mild enough to not block work but persistent enough to create pressure to update.

### Setup Script (setup.ps1)

```powershell
# DataHub Copilot Plugin Setup
# Installs curated skills, platform context, and registers update command

$ErrorActionPreference = "Stop"

$SkillsDir = "$env:USERPROFILE\.copilot\skills"
$PromptsDir = "$env:APPDATA\Code\User\prompts"
$ScriptDir = Split-Path -Parent $MyInvocation.MyCommand.Path
$ProfilePath = $PROFILE.CurrentUserCurrentHost

Write-Host "DataHub Copilot Plugin Setup" -ForegroundColor Cyan
Write-Host "=============================" -ForegroundColor Cyan

# Create directories
New-Item -ItemType Directory -Path $SkillsDir -Force | Out-Null
New-Item -ItemType Directory -Path $PromptsDir -Force | Out-Null

# Clean previous install (remove old/renamed skills)
# Only removes datahub-* directories and datahub-version.json — leaves non-DataHub skills untouched
Write-Host "Cleaning previous DataHub skills..." -ForegroundColor Yellow
Get-ChildItem -Path $SkillsDir -Directory -Filter "datahub-*" -ErrorAction SilentlyContinue | Remove-Item -Recurse -Force
Remove-Item "$SkillsDir\datahub-version.json" -Force -ErrorAction SilentlyContinue

# Copy skills and platform references
Write-Host "Installing skills..." -ForegroundColor Yellow
Copy-Item "$ScriptDir\skills\*" -Destination $SkillsDir -Force -Recurse

# Copy .instructions.md (DataHub mental model)
Write-Host "Installing platform context..." -ForegroundColor Yellow
Copy-Item "$ScriptDir\prompts\.instructions.md" -Destination "$PromptsDir\.instructions.md" -Force

# Write version file
Copy-Item "$ScriptDir\version.json" -Destination "$SkillsDir\datahub-version.json" -Force

# Register datahub-update command (idempotent)
$UpdateFunc = @'

# DataHub Copilot Plugin Update
function datahub-update {
    $RepoPath = "$env:USERPROFILE\datahub-copilot"
    if (-not (Test-Path $RepoPath)) {
        Write-Host "First-time setup: cloning datahub-copilot..." -ForegroundColor Yellow
        git clone https://dev.azure.com/protectivetfsprod/DataHub/_git/datahub-copilot $RepoPath
    }
    Push-Location $RepoPath
    git pull --ff-only
    if ($LASTEXITCODE -ne 0) {
        Write-Host "ERROR: git pull failed." -ForegroundColor Red
        Pop-Location
        return
    }
    & "$RepoPath\setup.ps1"
    Pop-Location
}
'@

if (-not (Test-Path $ProfilePath)) {
    New-Item -ItemType File -Path $ProfilePath -Force | Out-Null
}
$ProfileContent = Get-Content $ProfilePath -Raw -ErrorAction SilentlyContinue
if ($ProfileContent -notmatch 'function datahub-update') {
    Add-Content -Path $ProfilePath -Value $UpdateFunc
    Write-Host "Registered 'datahub-update' command." -ForegroundColor Yellow
} else {
    Write-Host "'datahub-update' command already registered." -ForegroundColor Gray
}

Write-Host ""
Write-Host "Setup complete!" -ForegroundColor Green
Write-Host "  Skills:           $SkillsDir" -ForegroundColor Gray
Write-Host "  Platform context: $PromptsDir\.instructions.md" -ForegroundColor Gray
Write-Host ""
Write-Host "To update in the future, run: datahub-update" -ForegroundColor Yellow
Write-Host "Restart VS Code to activate." -ForegroundColor Yellow
```

### Distribution Flow Summary

```
Tech lead merges skill update to main
        │
        ▼
ADO pipeline fires ──► Team notification (Teams/Slack/email)
                              │
                              ▼
                    Developer sees notification
                              │
                              ▼
                    Developer runs: datahub-update
                              │
                              ▼
                    git pull + setup.ps1 copies skills
                              │
                              ▼
                    VS Code restart activates new skills

If developer misses notification:
        │
        ▼
Skills go stale (>14 days)
        │
        ▼
Every Copilot response shows ⚠️ staleness warning
        │
        ▼
Developer runs: datahub-update
```

---

## 9. Keeping Content Fresh — The Drift Scanner

The plugin is a snapshot of how DataHub works at a point in time. The 15 working repos keep evolving — dependencies get bumped, file structures change, new repos appear. The drift scanner is a scheduled ADO pipeline that checks whether reality has drifted from what the skills describe, and produces a report telling the tech lead what needs attention.

### What the Scanner Checks

**Dependency versions:** Parses `pyproject.toml`, `package.json`, and similar files across repos. Compares pinned versions of key libraries (dlt, dagster, dbt-core, etc.) against what the skills document. Flags mismatches.

Example output: `datahub-data-platform-repo bumped dlt from 1.14.1 to 1.16.0 — datahub-dlt skill references 1.14.1`

**Key file changes:** Tracks modification dates of files that skills reference — `deploy.yaml`, pipeline YAML, `pyproject.toml`, directory structures. If a file changed since the last skill update, it might mean the skill's instructions are stale.

Example output: `data-api/.azuredevops/pipelines/deploy.yaml modified 2026-03-08 — last skill update 2026-02-20`

**File structure drift:** Compares the actual directory tree of each repo against what `copilot-instructions.md` describes. Flags new top-level directories, missing expected directories, or renamed paths.

Example output: `datahub-data-platform-repo has new directory sources/p8e-data-source-jira/ not covered by any skill reference`

**New or removed repos:** Checks the ADO project for repos that don't have a corresponding `copilot-instructions.md` in the central repo, or instructions that reference repos that no longer exist.

Example output: `New repo dp-infra-monitoring detected — no copilot-instructions.md exists`

### What the Scanner Does NOT Do

The scanner does not update skills. It produces a drift report. A human reads the report, decides what matters, updates the relevant skills, and merges. The scanner tells you *something changed* — it can't tell you *how to update the skill*.

### Skill Metadata for Version Tracking

For the scanner to compare "what the skill thinks the version is" against "what the repo actually has," each skill with upstream dependencies maintains a `skill-metadata.json`:

```json
{
  "tracked_versions": {
    "dlt": "1.14.1",
    "dagster-dlt": "0.24.0"
  },
  "last_curated": "2026-03-01"
}
```

The scanner reads this file alongside `scan-config.json` to produce version drift reports. This is more reliable than parsing version numbers out of markdown.

### Configuration (scan-config.json)

```json
{
  "repos": [
    {
      "name": "datahub-data-platform-repo",
      "ado_project": "DataHub",
      "tracked_dependencies": {
        "pyproject.toml": ["dlt", "dagster", "dagster-dlt", "dagster-dbt"]
      },
      "tracked_files": [
        ".azuredevops/pipelines/deploy.yaml",
        "pyproject.toml"
      ],
      "expected_structure": [
        "p8e_data_platform/",
        "sources/",
        "dbt/",
        ".azuredevops/pipelines/"
      ],
      "related_skills": ["datahub-dlt", "datahub-dagster", "datahub-dbt"]
    },
    {
      "name": "data-api",
      "ado_project": "DataHub",
      "tracked_dependencies": {
        "package.json": ["express", "@azure/identity"]
      },
      "tracked_files": [
        ".azuredevops/pipelines/deploy.yaml",
        "charts/"
      ],
      "expected_structure": [
        "src/",
        "charts/",
        ".azuredevops/pipelines/"
      ],
      "related_skills": ["datahub-k8s"]
    }
  ],
  "scan_schedule": "weekly",
  "notification_channel": "teams_webhook_url"
}
```

### Pipeline Design

The scanner runs as a scheduled ADO pipeline (weekly recommended, bi-weekly acceptable). It:

1. Clones or fetches the latest from all 15 repos (read-only)
2. Reads `scan-config.json` for what to check per repo
3. Reads `version.json` for the last skill update timestamp
4. Runs checks: dependency versions, file modifications since last update, structure diff, repo existence
5. Produces a drift report (markdown format)
6. Posts the report to the team channel (same webhook as update notifications)
7. If zero drift detected, posts a short "all clear" confirmation so the team knows the scan ran

### Example Drift Report

```markdown
# DataHub Plugin Drift Report — 2026-03-12

## 🔴 Action Required

**datahub-data-platform-repo**
- dlt version bumped: 1.14.1 → 1.16.0 (affects datahub-dlt skill)
- New source directory: sources/p8e-data-source-jira/ (not in any skill reference)

**data-api**
- deploy.yaml modified 2026-03-08 (last skill update: 2026-02-20)

## 🟡 Worth Checking

**dp-infra-networking**
- pyproject.toml modified 2026-03-10 (may be routine dependency bump)

## ✅ No Drift

data-transformation, runway, runway-provisioner, argocd,
dp-service-catalog, data-contracts, build-templates,
dg-build-templates, fivetran-connections, dp-infra-fivetran,
data-projen, protective-life-tables
```

### Drift → Update Flow

```
Weekly scan runs
      │
      ▼
Drift report posted to team channel
      │
      ▼
Tech lead reviews report
      │
      ├─ No action needed ──► Done
      │
      └─ Skill needs update ──► Edit skill in central repo
                                      │
                                      ▼
                                Bump version.json, merge to main
                                      │
                                      ▼
                                Normal distribution kicks in
                                (notification → datahub-update → VS Code restart)
```

---

## 10. Per-Repo Instruction Distribution

Repo instruction files live in the central repo under `repo-instructions/`. They need to reach each working repo's `.github/copilot-instructions.md`.

### Phase 1 (Manual)

When instructions change, tech lead copies the file to the relevant repo and commits. Acceptable for 15 repos if changes are infrequent (these files contain stable facts).

### Phase 2 (Automated)

ADO pipeline in the central repo detects changes to `repo-instructions/*.md` on merge to main. For each changed file, the pipeline uses the Azure DevOps REST API to create a PR against the corresponding working repo updating `.github/copilot-instructions.md`.

This is lower priority than skill distribution because repo instructions change rarely and contain stable facts.

---

## 11. Skill Inventory

| Skill | Domain | Key Knowledge | Curated From |
|-------|--------|---------------|--------------|
| datahub-dagster | Orchestration | Dagster API, dg CLI, assets, sensors, deploy.yaml chain, branch deployments | Dagster upstream skill + DataHub deploy conventions |
| datahub-dlt | API ingestion | dlt API, @dlt.source/@dlt.resource, incremental loading, Dagster-dlt bridge | dlthub upstream + DataHub source patterns |
| datahub-dbt | Transformation | dbt CLI, models, sources, testing, Raw→Prep→Prod layering | dbt upstream + DataHub datalake conventions |
| datahub-infra | Infrastructure | Terragrunt, OpenTofu, Azure providers, dp-service-catalog patterns | Terraform upstream + DataHub IaC patterns |
| datahub-k8s | Deployment | AKS, Helm, FluxCD, runway, container deployment | DataHub deployment-k8s conventions |
| datahub-contracts | Data contracts | ODCS spec, schema validation, contract-first development | Tech lead's existing ODCS work |

**Not a skill but part of the plugin:**

| Component | Type | Contents |
|-----------|------|----------|
| .instructions.md | Ambient context | Slim DataHub mental model — read on every Copilot interaction |
| datahub-platform/references/ | Reference files | Architecture, datalake layers, deployment paths, template map, PR workflows — read by domain skills via path |

---

## 12. Known Risks and Mitigations

### Risk 1: Developers Don't Update

**Problem:** Notifications ignored, `datahub-update` never run.
**Mitigation:** Three-layer defense. Layer 1 (notification) is ignorable. Layer 2 (command) requires action. Layer 3 (staleness warning in every Copilot response) is inescapable — the developer sees it every time they use Copilot. Social pressure also helps: if one developer's Copilot keeps showing warnings while others don't, they'll update.

### Risk 2: Upstream Skill Drift

**Problem:** Vendor releases a new version of their tool, curated skill has outdated patterns.
**Mitigation:** Each skill has an UPSTREAM-SOURCE.md tracking the vendor version it was curated from. Maintenance cadence: check upstream sources monthly. When a vendor releases a significant update, the relevant skill gets re-curated and pushed via the normal distribution flow. The drift scanner also flags dependency version bumps in working repos.

### Risk 3: Skill Keyword Collisions

**Problem:** Developer asks about something that matches multiple skills, Copilot picks the wrong one.
**Mitigation:** Skill descriptions should have distinct keyword spaces with minimal overlap. If collision is unavoidable (e.g., "deploy" matches datahub-dagster, datahub-k8s, and datahub-infra), include disambiguation in the description: "Use when deploying to Dagster Cloud" vs "Use when deploying to AKS."

### Risk 4: ~~Multi-Domain Tasks~~ — RESOLVED

**Original concern:** Developer's task spans two skills and Copilot only reads one.
**Test result:** Confirmed that Copilot reads multiple skills in a single interaction without any orchestration layer. When prompted with a task spanning dlt and Dagster, Copilot independently identified both domains, read both skill files, and combined the knowledge. No agents, routing, or explicit skill pointers needed. See Section 2 for test details.

### Risk 5: Curated Knowledge Goes Stale Internally

**Problem:** DataHub conventions change (new repo structure, new deployment path) but skills aren't updated.
**Mitigation:** Tie skill updates to the same sprint process that changes conventions. If a tech lead changes a deployment path, the corresponding skill update is part of the same work item. The drift scanner catches cases where the process discipline fails — it flags file structure changes and dependency bumps that suggest skills may need updating. MAINTENANCE.md documents which skills need updating for each type of platform change.

### Risk 6: Copilot Ignores Skill Constraints

**Problem:** Copilot reads the skill but doesn't follow constraints (generates code outside the expected patterns).
**Mitigation:** Make constraints concrete and specific — file paths, not principles. "ONLY create dlt sources inside sources/p8e-data-source-*/" is enforceable. "Follow DataHub conventions" is not. Front-load constraints at the top of SKILL.md — if Copilot only reads the first half, the guardrails should be in it. Test with example prompts and document expected vs actual behavior in tests/.

### Risk 7: Staleness Warning Unreliable

**Problem:** The staleness check instruction in each skill asks Copilot to read `datahub-version.json` and warn if stale. But Copilot may skip this preamble step and go straight to answering — same class of problem as the v3 routing table.
**Mitigation:** Test staleness warning reliability explicitly in Phase 1. If it fires inconsistently, accept that inline warnings are best-effort and rely on the notification layer (Teams/Slack) and the `datahub-update` command as the primary freshness mechanisms. Don't design around the staleness warning being 100% reliable.

### Risk 8: .instructions.md Bleeds into Non-DataHub Work

**Problem:** `.instructions.md` lives at the user level and is read on every Copilot interaction, including non-DataHub repos. A developer working on a React frontend gets "DataHub is a data platform with 15 repos" injected into context.
**Mitigation:** The content is ~150 words of ambient context — small enough that it's unlikely to confuse Copilot or degrade non-DataHub interactions. If the team works exclusively on DataHub repos, this is a non-issue. If developers split time across DataHub and non-DataHub projects, monitor whether the bleed-through causes confusion and consider making the content conditional (e.g., "If the current workspace is a DataHub repo, apply the following context..."). Test this in Phase 1.

### Risk 9: New Developer Onboarding

**Problem:** New developer starts, doesn't have skills installed, nobody tells them.
**Mitigation:** Add `datahub-copilot` setup to the onboarding checklist. The `datahub-update` command handles first-time clone + install in one step — no separate clone step needed.

### Risk 10: Drift Scanner Requires Cross-Repo Access

**Problem:** The scanner pipeline needs read access to all 15 repos. Enterprise ADO environments often scope permissions per team.
**Mitigation:** Create a dedicated service connection or build identity with read-only access to all DataHub repos. Document this in the central repo's README and MAINTENANCE.md. The scanner never writes to working repos — read-only access is sufficient.

---

## 13. Maintenance

### Adding a New Skill

1. Create `skills/{name}/SKILL.md` following the standard structure
2. Add `references/` directory with supporting material
3. If curating from upstream, add `UPSTREAM-SOURCE.md` with source details
4. Add the new skill's repos to `scan-config.json` so the drift scanner covers them
5. Add example prompts and expected outputs to `tests/`
6. Test with at least two developers on real tasks
7. Merge to main — ADO pipeline notifies team

### Updating an Existing Skill

1. Edit skill files in the central repo
2. Update `version.json` (bump version, update timestamp and changelog)
3. Merge to main — notification goes out, developers run `datahub-update`

### Responding to a Drift Report

1. Review the weekly drift report
2. For each flagged item, decide: does this affect what the skill tells developers to do?
3. If yes, update the skill and/or platform reference files
4. If no (e.g., a routine dependency bump with no API changes), no action needed
5. If a new repo appeared, author a `copilot-instructions.md` and add it to `scan-config.json`

### Monthly Maintenance Checklist

- [ ] Check upstream sources for major version changes (Dagster, dbt, dlt, Databricks, Terraform)
- [ ] Review accumulated drift reports for patterns (same thing flagged repeatedly = systemic gap)
- [ ] Review test results — run example prompts against current skills
- [ ] Check if any DataHub platform conventions have changed
- [ ] Update UPSTREAM-SOURCE.md for any re-curated skills
- [ ] Bump version.json and merge

---

## 14. Rollout Plan

### Phase 1: Foundation (Week 1-2)

- [ ] Create central repo with directory structure
- [ ] Build setup.ps1 with PowerShell profile integration
- [ ] Write .instructions.md (slim DataHub mental model)
- [ ] Create version.json and staleness check pattern
- [ ] Build datahub-platform reference files (architecture, datalake layers, deployment paths)
- [ ] Build datahub-dlt skill (curate from upstream + DataHub conventions)
- [ ] Write copilot-instructions.md for datahub-data-platform-repo
- [ ] Test full chain: install → use skill → get DataHub-specific answer
- [ ] Verify .instructions.md is read on every interaction (test with generic DataHub question)
- [ ] Verify skill reads platform reference files by path
- [ ] Test staleness warning fires after 14 days (and note reliability — see Risk 7)
- [ ] Test .instructions.md bleed-through: open a non-DataHub repo and verify DataHub context doesn't confuse Copilot (see Risk 8)
- [ ] Deploy to 2-3 developers on real sprint tasks

### Phase 2: Dagster + dbt + Distribution (Week 3-4)

- [ ] Build datahub-dagster skill (curate from tech lead's upstream skill)
- [ ] Build datahub-dbt skill (curate from upstream)
- [ ] Write copilot-instructions.md for data-transformation
- [ ] Set up ADO pipeline for team notifications on merge to main
- [ ] Build scan-config.json for initial repos
- [ ] Build and test drift scanner pipeline (scan.ps1 + scheduled ADO pipeline)
- [ ] Test with full data engineering team

### Phase 3: Infra + K8s (Week 5-6)

- [ ] Build datahub-k8s and datahub-infra skills
- [ ] Write copilot-instructions.md for remaining repos
- [ ] Expand scan-config.json to cover all repos
- [ ] Set up ADO pipeline for repo instruction distribution (Phase 2 automation)

### Phase 4: Full Coverage (Week 7-8)

- [ ] Adopt datahub-contracts skill from tech lead's ODCS work
- [ ] Complete copilot-instructions.md for all 15 repos
- [ ] Write MAINTENANCE.md
- [ ] Run full test suite across all skills
- [ ] Verify drift scanner covers all repos and produces actionable reports
- [ ] Onboarding documentation finalized

---

## 15. Coordination with Tech Lead's Existing Work

The tech lead has already built:
- Dagster upstream skill (curate into datahub-dagster)
- ODCS agent (adapt substance into datahub-contracts skill)
- Various custom skills with nested reference files

**Approach:** Frame the central repo as scaling her work across all 15 repos. Specifically:
- Her Dagster upstream skill → curate into datahub-dagster, filtering through DataHub conventions
- Her ODCS agent → extract the knowledge and constraints into datahub-contracts skill format
- Her custom DataHub-specific content → evaluate for datahub-platform references
- Coordinate before Phase 1 to align on structure and avoid duplicate effort

---

## 16. What We Dropped from v3 (and Why)

| v3 Component | Status | Reason |
|--------------|--------|--------|
| .instructions.md as routing table | **Repurposed** as ambient context | Copilot doesn't reliably read it before matching skills. Now carries a slim DataHub mental model instead of routing rules — provides baseline context on every interaction without trying to control dispatch. |
| Specialist agents (.agent.md) | Dropped | Copilot skips agents when skills match keywords directly. Agent value (constraints, approach, output format) now lives inside skills. |
| runSubagent delegation | Dropped | Unnecessary when there's no agent to delegate to. |
| Orchestrator agent for multi-skill tasks | Dropped | Testing confirmed Copilot reads multiple skills natively in a single interaction. No orchestration layer needed. |
| Separate upstream skill directories | Dropped | Raw upstream skills compete with curated skills for Copilot's attention. Upstream knowledge is now curated into domain skills directly. |
| user-level prompts/ directory for agents | **Repurposed** for .instructions.md | No longer holds agent files. Holds only the slim .instructions.md platform context file. |
| datahub-platform as a skill | **Demoted** to reference directory | A platform skill with keywords like "architecture" and "deployment" would attract false matches. Now a plain reference directory with no SKILL.md — only read when domain skills explicitly point to it. |
