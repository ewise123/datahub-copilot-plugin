# Phase 0: Plugin Validation Checklist

ADO Repo: https://dev.azure.com/SSAAIAccelerator/DataHub%20Copilot%20Test

## Prerequisites

- [ ] VS Code Insiders or latest stable with Agent Plugins preview
- [ ] `chat.plugins.enabled: true` in VS Code settings
- [ ] GitHub Copilot extension active

## Test 1: ADO HTTPS Git URL as Marketplace Source

Add to VS Code `settings.json`:
```json
"chat.plugins.marketplaces": [
    "https://dev.azure.com/SSAAIAccelerator/DataHub%20Copilot%20Test/_git/datahub-copilot-plugin"
]
```

- [ ] Does VS Code fetch the marketplace feed without errors?
- [ ] Does the "datahub" plugin appear when searching `@agentPlugins`?
- [ ] If ADO URL fails, try alternate formats:
  - `https://SSAAIAccelerator@dev.azure.com/SSAAIAccelerator/DataHub%20Copilot%20Test/_git/datahub-copilot-plugin`
  - With `.git` suffix
- [ ] Document: auth prompt? PAT required? credential manager integration?

**Fallback if ADO doesn't work:**
Use `chat.plugins.paths` pointing to local clone:
```json
"chat.plugins.paths": [
    "/path/to/local/datahub-copilot-plugin"
]
```

## Test 2: Plugin Installation

- [ ] Plugin installs from Extensions sidebar
- [ ] No errors in VS Code Output > GitHub Copilot Chat
- [ ] Installed plugin shows correct name and description

## Test 3: Skill Discovery

Open a workspace and ask Copilot:
> "I need to create a new dlt source for ServiceNow"

- [ ] Copilot matches keywords and reads the datahub-dlt skill
- [ ] Copilot reads the skill's references/ directory
- [ ] Response follows DataHub conventions (p8e-data-source-* pattern, Key Vault auth, etc.)

## Test 4: .instructions.md Bundling

- [ ] Does Copilot read the .instructions.md from the plugin?
- [ ] Test: ask "What deployment lanes does DataHub have?" — should answer from ambient context
- [ ] If .instructions.md is NOT picked up, plan preamble fallback (bake into each SKILL.md)

## Test 5: Plugin Update Propagation

1. Make a trivial change to SKILL.md (add a comment)
2. Bump version in plugin.json and marketplace.json
3. Push to ADO
4. Check: does VS Code detect the update?
- [ ] Update notification appears
- [ ] Updated skill content is used after update

## Test 6: Copilot CLI Diagnostics

```bash
# If Copilot CLI is available, test for better error messages
copilot plugin list
copilot plugin install ./plugins/datahub
```

- [ ] CLI provides useful error messages for manifest issues
- [ ] CLI confirms skill is loaded

## Results

| Test | Pass/Fail | Notes |
|------|-----------|-------|
| ADO URL as marketplace source | | |
| Plugin installation | | |
| Skill discovery | | |
| .instructions.md bundling | | |
| Plugin update propagation | | |
| Copilot CLI diagnostics | | |

## Issues Discovered

(Document any issues, workarounds, or limitations here)
