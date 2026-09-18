---
name: build-feature
description: >-
  Drains a named PXP Unity Azure DevOps Feature
  (https://dev.azure.com/pxphq/Unity): implements each specified PBI and its
  Tasks one by one with tests, runs independent code/security/QA reviews,
  merges only onto a long-lived Feature integration branch, and closes those
  work items. Orchestrates Unity implement/review workflows and repo-type
  agents, reusing their reference and knowledge base. Use when the user says
  build-feature, build a Feature, drain a Feature, implement feature, implement
  all PBIs of a Feature, or names a Feature ID to build onto a feature branch
  (for example Feature 306968).
disable-model-invocation: true
---

# Build Feature

Edie orchestrator for PXP Unity Features and PBIs (https://dev.azure.com/pxphq/Unity). **Not** a Unity catalog role. **Not** `/autopilot`.
Does **not** merge to `development` / `main` / `prod` / `master` / `releases/*`.

Announce before acting: session is **build-feature**; each PBI adopts the
repo-type **primary** from `Pxp.Unity.Agents.Policy/orchestration/routing.md`,
then independent reviewers (`backend-quality-agent` / `frontend-quality-agent`,
`security-review-agent`, QA). Resolve routing at session time — do not cache names
into git files.

Invoking this skill **is** in-session human direction (`[GLB-024]`) **only** for
completing PRs whose **target is the Feature integration branch**. Never pass
`-HumanDirected` for a PR targeting a default/release/environment branch.

## When to use

User names a Feature ID (or URL) and wants every **specified** child PBI built
onto one integration branch per repo, then those PBIs/Tasks closed. Companion
to `refine-feature` (specify) then this skill (build).

## Hard rules

- One PBI at a time, one **repository** per change (`[GLB-022]`). Split multi-repo PBIs into sequential repo slices.
- **One commit per Task** on the PBI branch — never squash Task slices into a single PBI commit. If the PBI has no Tasks, one commit for the whole PBI is allowed.
- Tests ship with the behavior (implement workflows step 6). Do not skip tests to go green.
- Never log, echo, or persist PAN / secrets (`[GLB-040]`/`[GLB-041]`).
- Never push to default/release/environment branches; never deploy (`[GLB-003]`).
- **Stop** (do not implement) ADR-0004 **class 1** (crypto/secrets/HSM/auth internals) and **class 4** (prod desired-state). Report and continue the rest of the queue if possible.
- Class 2/5/3: still implement onto the integration branch; **do not** complete a PR to the default branch. Flag the human gate for later integration.
- Skip `Rejected` / `Removed` / `Done` PBIs. Skip items that fail the specified bar (see below). Skip other-team `AreaPath` unless the user named that ID (`[ADO-012]`).
- Do **not** run `prd-story-mapping-agent` or `refine-feature`. Do not create PBIs/Tasks.
- Do **not** close the Feature or Epic.
- Circuit breaker: **3 consecutive** failed PBI cycles → stop the run and summarize.
- Resolve `ado/*` script names from `Pxp.Unity.Agents.Tooling/ado/README.md` at time of use. Prefer those scripts; MCP work-item updates only in a qualifying ADR-0023 §1a session (`ADR-0025`).
- PowerShell: confirm `$PSVersionTable.PSVersion.Major`; on 5.x use `;` not `&&` (`ENV-002`).
- **Host-app i18n:** If a Task adds or changes ngx-translate keys in a frontend-package (e.g. `@pxp/transactionsriskscreening`), the UI is **not done** until Portal `src/assets/i18n/en.json` (and sibling locale files that already have the parent block) contain those keys. Library showcase `en.json` is the copy source; the package dist is **not** loaded by Portal. This is a required `Pxp.Unity.Portal` slice (`[GLB-022]`), not an optional extra — Bug 309212 and Country Blacklist raw keys (`*.country-blacklist.*`) are this gap. Do not close the Portal-UI PBI while the running Portal shows dotted keys.

## Specified bar (drainable PBI)

All of:

1. Type is Product Backlog Item (or Bug the user included).
2. State is not `Rejected`, `Removed`, or `Done`.
3. `Microsoft.VSTS.Common.AcceptanceCriteria` has real intent — no `TBC` / placeholder-only.
4. If `Custom.Scope` is populated, every Scope line has ≥1 AC line (`ADO-001` coverage).
5. Area is in session scope (or user named the ID).

`Committed` is sufficient but **not** required. Unspecified PBIs: comment why skipped; do not specify them.

## Inputs

Required: Feature work-item ID.

Optional: integration branch name (default `Edison/<FeatureId>-build` using the Feature's squad prefix if the area is a known squad, else `feature/<FeatureId>-build`); PBI order; repos to include/exclude.

Load, in order:

1. Feature (title, area, parent, description).
2. Children via `Get-WorkItemChildren.ps1 -Id <FeatureId> -MaxResults 100`.
3. Each drainable PBI's fields + its Task children (same script on the PBI).
4. `inventory/repositories.json` for `repositoryType` + `defaultBranch` (never trust local Git for the default).
5. Owning repos from PBI Scope / TechnicalImplementationDetails / inspected source — not guessed.

If the ID is missing or not a Feature, stop and ask.

## Workflow

Copy and track:

```
build-feature:
- [ ] Load Feature, children, specified-bar filter
- [ ] Per repo: ensure integration branch from inventory defaultBranch
- [ ] For each drainable PBI (lowest ID unless user ordered):
  - [ ] Split by repo if needed
  - [ ] Cut PBI branch from integration; adopt primary software agent
  - [ ] For each child Task (or one slice if no Tasks):
    - [ ] Set Task In Progress; implement that Task slice + tests
    - [ ] Commit **only that Task's files** with `AB#<TaskId>` (see below)
    - [ ] Push PBI branch after each Task commit (or after the last Task before review)
  - [ ] Independent code review (quality agent) — diff = all Task commits on PBI branch
  - [ ] Independent security review
  - [ ] Independent QA review
  - [ ] Rework until reviews pass or skip after 3 loops (rework commits also per Task)
  - [ ] Merge PBI branch into integration (preserve per-Task commits; no squash)
  - [ ] Close Tasks then PBI; comment with PR/commit SHAs per Task
- [ ] Summarize; Feature stays open for human → defaultBranch
```

### Integration branch

Per repo, once:

1. `defaultBranch` from `inventory/repositories.json`.
2. If `origin/<integration>` exists: checkout, fast-forward from origin only (no rebase of shared history).
3. Else create from `origin/<defaultBranch>` with `New-Branch.ps1 -Repo <repo> -Branch <integration> -BaseRef <defaultBranch>` (or equivalent git) and push `-u`.

PBI work branch: `<integration>` + `-<PBIId>` (or `-<PBIId>-<repo-slug>`). Cut from **current integration**, not from default.

### Per-Task commits

Each **Task** is one implementation slice and **one git commit** on the PBI branch. Do not land multiple Tasks in one commit or batch-unrelated PBI work into a single commit.

**Per Task (in Task ID order):**

1. `Set-WorkItemState.ps1 -Id <TaskId> -State "In Progress" -AssignedTo <session-user>` (`ADO-011`). Use the Azure DevOps identity of the engineer running this session.
2. Implement **only** what that Task's title/scope requires; parent PBI AC remains the test oracle. If the Task is library UI with new translate keys and no Portal i18n Task exists, **do not skip Portal** — implement Portal `en.json` as the next repo slice of the same PBI (or stop and report the missing Task rather than shipping unreadable UI).
3. Stage **only** files touched for this Task. If a file spans Tasks, finish the earlier Task's portion first and commit before continuing.
4. Commit with imperative message, **Task id in the subject**:
   - `AB#<TaskId> <short intent>` (e.g. `AB#309160 Add CardBlacklistEntries DACPAC table`)
   - Body may reference parent `AB#<PBIId>` when helpful.
5. Push the PBI branch (`git push origin <pbi-branch>`). Repeat for the next Task.

**If the PBI has no Tasks:** one commit `AB#<PBIId> …` for the full slice is allowed.

**Rework after review:** fix on the PBI branch; commit per Task affected (`AB#<TaskId> …` or `AB#<TaskId> Address review: …`). Do not amend prior Task commits unless that commit was created in the **same** session, has **not** been pushed, and `[GLB-024]` merge to integration has not happened yet.

**Merge to integration:** use merge (not squash) so each Task commit stays visible on `<integration>`. PR description lists Task ids with their commit SHAs.

### Implement (reuse Unity workflows)

Read the matching file and execute it, **except** branch/PR target:

| Repo type (`repositories.json`) | Workflow | Primary | Quality |
|---|---|---|---|
| `backend-service`, `worker-service`, `nuget-package` | `orchestration/workflows/implement-backend-feature.md` | `backend-software-agent` | `backend-quality-agent` |
| `frontend-application`, `frontend-package`, `sdk` | `orchestration/workflows/implement-frontend-feature.md` | `frontend-software-agent` | `frontend-quality-agent` |
| other | stop and ask; do not guess |

Also follow workspace skills `create-change-pr` (description/test-plan/AB# rules) and `link-task-to-pull-request`, with **Target = integration branch**.

Follow **Per-Task commits** above: implement and commit Task-by-Task before reviews. Tests for a Task ship in that Task's commit (implement workflows step 6). No live sandbox PANs.

### Independent reviews (parallel Task subagents)

After **all** Tasks for the PBI repo slice are committed on the PBI branch, launch **three** `Task` subagents in **one** message, `run_in_background: false`. They must **not** have written the code. Prompts: [reference.md](reference.md). The diff is the full PBI branch vs `<integration>` (all Task commits).

1. **Code** — `subagent_type: generalPurpose`. Instruct: operate as the **quality** agent for this repo type; read `agents/<quality>.md` and `orchestration/workflows/review-pull-request.md`; read-only except adding tests; verdict `PASS` / `REWORK`.
2. **Security** — prefer `subagent_type: security-review` (workspace `review-security` skill shape: `Diff: branch changes`, `Base Branch: <integration>`). If that type is unavailable: `generalPurpose` reading `agents/security-review-agent.md`.
3. **QA** — `subagent_type: generalPurpose`. Independent of code review. Exercise the **running** app against each AC (Ivan `qa-verifier` shape in [reference.md](reference.md)). `VERDICT: VERIFIED` or `NOT VERIFIED`.

Base the diff on the **integration branch**, not `development`, when the PBI branch was cut from integration.

**REWORK** / **NOT VERIFIED** / security **fail**: adopt the software agent, fix, re-run all three. After 3 loops, skip the PBI (leave In Progress / Committed), comment the blocker, count a circuit-breaker failure.

**PASS + VERIFIED + security pass** (or security `N/A` with evidence the diff is outside class 1 / PCI / secrets): proceed to merge.

### Merge onto the integration branch

1. `New-PullRequest.ps1 -Repo <repo> -Source <pbi-branch> -Target <integration> -Title "AB#<PBIId> …"` with test plan + `AB#<PBIId>` and **each** `AB#<TaskId>` with its commit SHA. `[ADO-013]`: never `PR #<n>` in ADO markdown. Do **not** squash commits on merge.
2. `Set-WorkItemPullRequestLink.ps1` for the PBI and each Task.
3. Complete with `Complete-PullRequest.ps1 -Id <pr> -HumanDirected -DeleteSourceBranch` **only if** target is the integration branch.
4. If completion refuses (policies/reviewers on an unprotected branch): local `git checkout <integration>`; `git merge --no-ff <pbi-branch>`; `git push origin <integration>`. Do **not** `--bypass-policy`. Do **not** retarget to default.
5. Confirm `origin/<integration>` contains the commits. Do not mark items Done until that is true.

### Close linked items (integration-branch deviation)

Unity `[ADO-010]` ties PBI `Done` to merge on the **default** branch. **This skill** closes work when it is on the **integration** branch so the board matches the drain. Record that in the PBI comment.

1. Each implemented Task → `Done` (`Set-WorkItemState.ps1`).
2. PBI → `Done` only if every child Task is `Done` or descoped with a comment, AC intended to be satisfied, and commits are on `origin/<integration>`.
3. Comment Feature + PBI with repo, integration branch, PR id/URL (`PR !<id>` in ADO comments).
4. Leave Feature open. Human later PRs `<integration>` → inventory `defaultBranch`.

## Output to the user

- Integration branch per repo.
- Table: PBI, Tasks (with commit SHA each), repo, PR, merge result, state.
- Skipped / class-1/4 stops / circuit-breaker.
- Remaining human work: PR integration branch → default; ADR-0004 gates still apply there.

## Additional resources

- Review/QA prompts, merge fallback, invoke example: [reference.md](reference.md)
- Example Feature 306968: [examples.md](examples.md)
