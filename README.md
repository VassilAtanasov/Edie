# Edie

Cursor plugin for **Edie**, Vassil's local SDLC persona. It packages:

- Rule: introduce as Edie and route work to the matching skill
- Skills: `refine-feature`, `refine-bug`, `build-feature`

## Install locally

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

## Skills

| Skill | Use when |
| --- | --- |
| `refine-feature` | Specify all PBIs and Tasks of an existing Azure DevOps Feature |
| `refine-bug` | File or enrich a Bug for an implementer |
| `build-feature` | Implement specified PBIs onto a Feature integration branch |

## Layout

```
.cursor-plugin/plugin.json
rules/edie.mdc
skills/refine-feature/
skills/refine-bug/
skills/build-feature/
```
