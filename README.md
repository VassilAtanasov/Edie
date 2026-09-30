# Edie

Cursor and Claude Code plugin for **Edie**. Edie automates the Unity Agents framework for long-running multi-task sessions. Edie orchestrates Unity agents, reusing their reference and knowledge base.

Edie operates on PXP Unity projects in Azure DevOps — Features and PBIs at https://dev.azure.com/pxphq/Unity.

It packages:

- Persona: introduce as Edie and route work to the matching skill
- Skills: `refine-pbi`, `refine-bug`, `build-pbi`
- Disabled: `refine-feature`, `build-feature` (kept in `disabled-skills/`, not loaded)

The same `skills/` tree is loaded by both hosts. Cursor reads `rules/edie.mdc`. Claude Code reads `agents/edie.md` and, when the plugin is enabled, uses Edie as the session agent.

## Install in Cursor

Cursor loads Cursor Plugins from `~/.cursor/plugins/local/<plugin-name>/`.

From PowerShell:

```powershell
$src = "C:\git\va\Edie"
$dst = Join-Path $env:USERPROFILE ".cursor\plugins\local\edie"
New-Item -ItemType Directory -Force -Path (Split-Path $dst) | Out-Null
if (Test-Path $dst) { Remove-Item -Recurse -Force $dst }
Copy-Item -Recurse $src $dst
```

Then:

1. Settings → enable **Include third-party Plugins, Skills, and other configs**.
2. On a Team/Enterprise plan, an admin may need **Allow Local Plugin Imports**.
3. Command Palette → **Developer: Reload Window**.
4. Confirm `edie` appears under Customize, with the three skills and the Edie rule.

Do not symlink the git folder into `plugins/local` — Cursor currently ignores those links. Copy (or recopy after you change files).

## Install in Claude Code

This repo is both the **marketplace** (`.claude-plugin/marketplace.json`) and the **plugin** (`.claude-plugin/plugin.json`, `skills/`, `agents/`).

### From GitHub

In Claude Code:

```text
/plugin marketplace add VassilAtanasov/Edie
/plugin install edie@edie
/reload-plugins
```

Or from a shell:

```powershell
claude plugin marketplace add VassilAtanasov/Edie
claude plugin install edie@edie
```

### From a local clone

```text
/plugin marketplace add C:\git\va\Edie
/plugin install edie@edie
/reload-plugins
```

Or from a shell:

```powershell
claude plugin marketplace add C:\git\va\Edie
claude plugin install edie@edie
```

To load this checkout for one session without installing:

```powershell
claude --plugin-dir C:\git\va\Edie
```

After install, skills appear as `/edie:refine-pbi`, `/edie:refine-bug` and `/edie:build-pbi`. `build-pbi` is user-invoked only (`disable-model-invocation`).

Validate before publishing a change:

```powershell
claude plugin validate C:\git\va\Edie --strict
```

## Skills

| Skill | Use when |
| --- | --- |
| `refine-pbi` | Give an implementation agent the architecture and coding detail for one PBI, as an HTML Discussion comment |
| `refine-bug` | File or enrich a Bug for an implementer |
| `build-pbi` | Implement one PBI and its Tasks via reviewed task PRs into a PBI branch, leaving one PR for a human |

`refine-feature` and `build-feature` are **disabled** while they have known issues. Their files are kept in `disabled-skills/`, outside the `skills/` folder both hosts load, so they cannot be invoked. See `disabled-skills/README.md` to re-enable one.

## Layout

```
.cursor-plugin/plugin.json
.cursor-plugin/marketplace.json
.claude-plugin/plugin.json
.claude-plugin/marketplace.json
agents/edie.md
settings.json
rules/edie.mdc
skills/refine-pbi/
skills/refine-bug/
skills/build-pbi/
disabled-skills/refine-feature/
disabled-skills/build-feature/
```
