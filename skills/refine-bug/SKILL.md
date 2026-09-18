---
name: refine-bug
description: >-
  Refines or creates a PXP Unity Azure DevOps Bug
  (https://dev.azure.com/pxphq/Unity) with implementation-ready technical
  detail for human QA. Investigates codebase evidence, captures reproduction
  steps and session artifacts, asks clarifying pickable questions, then creates
  a new Bug linked to the correct parent PBI or updates an existing Bug
  Description in ADO. Use when the user says refine-bug, enrich a bug, file a
  QA bug, triage bug <id>, or wants an implementer-ready bug without
  re-triage (for example Bug 309212).
---

# Refine Bug

QA refinement for a **Bug** on the PXP Unity Azure DevOps board (https://dev.azure.com/pxphq/Unity): gather evidence from session context and source, ask the engineer running this session pickable questions, then **create or update** the ADO work item with a full HTML Description an implementer can execute without re-triaging.

This is **not** `fix-bug` (implementation). This is **not** `refine-feature` (Feature → PBI breakdown). Do **not** open a PR or change product code unless the user explicitly asks to fix the bug in the same turn.

State the role before proceeding: **QA triage / ADO authoring** (investigation + work-item write).

## When to use

- Human QA found a defect and wants it filed or enriched for an implementer.
- User names a Bug ID to enrich (`refine-bug 309212`).
- User describes a defect with no Bug yet (`refine-bug` + repro context).
- After local testing (Portal, APIs, screenshots) when technical root cause is partially known.

## Hard rules

- Do not invent APIs, file paths, root causes, or environments. Unevidenced items are `unknown` + follow-up (`GLB-012`).
- **Stop before any ADO write.** Ask pickable questions; **recommended option first**. After answers, rewrite the bug pack from those choices.
- **Description is HTML** in ADO — not Markdown. On **Bug**, the Details form does **not** show `System.Description`. Put the full pack in **`Microsoft.VSTS.TCM.ReproSteps`** (UI: Repro Steps). Also fill **`Microsoft.VSTS.TCM.SystemInfo`** (environment) and **`Microsoft.VSTS.Common.AcceptanceCriteria`** (verification checkboxes). Still set `System.Description` if create tooling requires it, but never treat that as visible. Comments may be plain text or simple HTML.
- Never log, echo, persist, or attach PAN, CVV, secrets, tokens, or raw cardholder data (`GLB-040`). Redact from logs and screenshots in write-ups.
- Resolve current `ado/*` script names from `Pxp.Unity.Agents.Tooling/ado/README.md` at time of use. Prefer those scripts; in a qualifying ADR-0023 §1a MCP session, Azure DevOps MCP is allowed for work-item read/update (`ADR-0025`).
- Link every new Bug to a **parent PBI** (`-ParentId`). Do not create orphan Bugs. If the parent PBI is unclear, stop at Gate A and ask.
- Do not set Bug `Done` or move to `In Progress` unless the user directs that separately.
- After discovering a work item is another team's `AreaPath`, do not keep writing it unless the user named that ID (`ADO-012`).
- **Artifacts:** cite screenshot/file paths from the session; tell QA to attach images to the Bug **Discussion** panel in ADO (agents cannot upload attachments via sanctioned CLI today).

## Inputs

At least one of:

| Input | Use |
|---|---|
| Bug work-item ID or URL | Enrich existing item |
| Repro narrative + screenshot(s) | Create or match existing |
| Parent PBI / Feature ID | Resolve hierarchy |
| Session context (local URLs, branches, builds) | Environment + technical detail |

Load when available:

1. Existing Bug (if ID given): `Get-WorkItem.ps1 -Id <id>`.
2. Parent PBI and Feature: `Get-WorkItem.ps1`, `Get-WorkItemChildren.ps1`.
3. Conversation artifacts: screenshots, terminal output, URLs, correlation IDs.
4. Inspected source in affected repos — grep/read, not guesses.
5. Unity references when present: `inventory/repositories.json`, `architecture/SERVICE_MAP.md`, parent Feature/Epic constraints.

If the ID exists but is not type **Bug**, stop and ask (may be a PBI mislabeled in chat).

## Workflow

Copy and track:

```
refine-bug:
- [ ] Load Bug (if any), parent PBI/Feature, session artifacts, source evidence
- [ ] Investigate: localize repo, files, root cause (evidenced)
- [ ] Gate A — QA clarifying questions — STOP for picks
- [ ] Draft bug pack (HTML Description) — no ADO writes
- [ ] Gate B — approve bug pack — STOP for picks
- [ ] Create Bug (New-Task.ps1 -Type Bug) OR update existing Description
- [ ] Add discussion comment with artifact checklist for QA
- [ ] Return Bug ID/URL, parent PBI, implementer prompt snippet
```

### Investigate (before Gate A)

From context + source, determine:

- **Symptom** (what the user sees).
- **Expected vs actual** (testable).
- **Environment** (Portal URL/port, gateway, branch, Docker vs local serve, linked `file:` packages).
- **Repro steps** (numbered, minimal).
- **Affected bounded context / repo(s)** from `repositories.json` + routing.
- **Root cause** with **file paths and line refs** only when evidenced in source.
- **Suggested fix** (concrete, minimal scope).
- **Regression?** Compare with sibling feature (e.g. BIN Blacklist works, Card Blacklist does not).
- **Out of scope / unknowns.**

Do not skip investigation when a Bug ID already exists — enrich with new evidence.

### Gate A — QA clarifying questions (no ADO writes)

Use `AskQuestion` when available. **Recommended option first.** Minimum topics:

1. **Parent PBI** — which PBI owns this UI/API slice? (list candidates from Feature children if known)
2. **Severity** — `1 - Critical` / `2 - High` / `3 - Medium` / `4 - Low`
3. **Environment** — local dev / shared QA / other
4. **Regression** — never worked vs worked before
5. **Create vs update** — new Bug vs enrich existing ID
6. **Artifacts** — screenshot attached to ADO? (yes/no — remind to attach manually)

Add feature-specific picks only when genuinely unresolved.

**Stop.** Do not draft the final Description until picks are answered.

### Draft bug pack (no ADO writes)

Use the HTML template in [reference.md](reference.md). Sections required:

| Section | Content |
|---|---|
| Summary | One paragraph |
| Environment | URLs, branches, ports, package link mode |
| Steps to reproduce | Ordered list |
| Expected / Actual | Testable |
| Artifacts | Screenshot paths, logs (redacted), URLs |
| Technical analysis | Root cause, files, suggested fix, repos, branches |
| Verification | How implementer/QA confirms fix |
| Unknowns | Explicit gaps |

Title pattern: `[Area] Short symptom` — e.g. `Card Blacklist modal shows translation keys instead of labels`.

### Gate B — approve bug pack (no ADO writes)

Show:

- Proposed **Title**
- Parent **PBI ID** (+ Feature if known)
- **Severity**, **Area**, **Iteration** (inherit from parent PBI unless QA picked otherwise)
- Full **Description** preview (HTML)
- **Implementer prompt** one-liner (see reference.md)

Ask:

1. Approve as drafted vs edit title/severity (recommended: approve)
2. Create new vs update existing Bug ID
3. Proceed to ADO write now vs draft-only

**Stop.** After picks, apply edits.

### Create or update ADO (only after Gate B)

**New Bug** — `New-Task.ps1`. Pass the HTML pack as `-Description` **and** the same HTML (or the Summary+repro+analysis pack) via `-Fields` `'Microsoft.VSTS.TCM.ReproSteps'` so it appears on the Bug Details form. Also set SystemInfo and AcceptanceCriteria:

```powershell
./ado/New-Task.ps1 `
  -Type Bug `
  -Title "<title>" `
  -ParentId <pbiId> `
  -AreaPath "Unity\Edison" `
  -IterationPath "<from parent PBI or Gate B>" `
  -Description "<html pack>" `
  -Fields @{
    'Microsoft.VSTS.Common.Severity' = '<severity>'
    'Microsoft.VSTS.Common.ValueArea' = 'Business'
    'Microsoft.VSTS.TCM.ReproSteps' = '<html pack>'
    'Microsoft.VSTS.TCM.SystemInfo' = '<html environment>'
    'Microsoft.VSTS.Common.AcceptanceCriteria' = '<checkbox html verification>'
  }
```

Multi-line HTML in `-Fields` must go through `New-AdoCliFileArg` (already used by `New-AdoWorkItem` for values containing newlines).

Resolve `Value Area` allowed values for Bug from `ado/README.md` if create fails.

**Existing Bug** — update the **visible** Bug fields (PXP Scrum - Unity form: Repro Steps, not Description). **Do not** pass multi-line HTML inline to `--fields` on Windows PowerShell. Use `New-AdoCliFileArg`:

```powershell
Import-Module (Join-Path $ToolingRoot 'shared/Ado.psm1') -Force
$reproFileArg = New-AdoCliFileArg -Value $htmlPack
try {
  Invoke-AzCli @(
    'boards', 'work-item', 'update',
    '--id', '<bugId>',
    '--org', 'https://dev.azure.com/pxphq',
    '--fields', "Microsoft.VSTS.TCM.ReproSteps=$($reproFileArg.Arg)",
    '--output', 'json'
  ) | Out-Null
} finally {
  Remove-Item $reproFileArg.Path -Force -ErrorAction SilentlyContinue
}
```

Optional same-call `--fields` for `Microsoft.VSTS.TCM.SystemInfo` and `Microsoft.VSTS.Common.AcceptanceCriteria` (use `@file` for any value containing newlines or `"`).

Optional title/severity in the same call as additional `--fields` pairs (use `@file` for any value containing newlines or `"`).

Prefer a single Repro Steps replace that merges prior useful content with new technical detail. Add a **Discussion** comment via `Add-WorkItemComment.ps1` (or REST `wit/comments` with no-BOM JSON if the script hits a BOM decode error) noting what changed and reminding QA to attach screenshots.

Do not implement the fix in this skill unless the user explicitly requests it in the same message.

## Output to the user

- Bug **ID and URL**
- Parent **PBI ID/URL**
- Summary of evidence added (root cause, files, fix hint)
- **Implementer prompt** (copy-paste ready)
- Reminder to attach screenshots in ADO Discussion if not yet attached
- Remaining `unknown` items

## Additional resources

- HTML Description template, implementer prompt template, example invoke: [reference.md](reference.md)
