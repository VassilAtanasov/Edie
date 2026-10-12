---
name: build-pbi
description: >-
  Builds one PXP Unity Azure DevOps Product Backlog Item
  (https://dev.azure.com/pxphq/Unity) and all its child Tasks with the Unity
  agents: one Task at a time on its own branch and PR into a PBI branch,
  independent code/security review per task PR, a human-directed squash merge
  per Task, one QA pass against the PBI acceptance criteria, then one
  ready-for-review PR from the PBI branch to the default branch so a human
  reviews a single PR. Moves Task and PBI states per ADO-010/011. Use when the
  user says build-pbi, build a PBI, implement a PBI with its Tasks, or names a
  PBI ID to implement (for example PBI 312486).
disable-model-invocation: true
---

# Build PBI

Edie orchestrator for one PXP Unity PBI (https://dev.azure.com/pxphq/Unity). **Not** a Unity catalog role. **Not** `/autopilot`.

Announce before acting (`[GLB-070]`): session is **build-pbi**; each repo slice adopts the primary resolved from `Pxp.Unity.Agents.Policy/orchestration/routing.json` at run time (never cached here), and reviews run in fresh sessions of the resolved lead reviewer and any change-class reviewers.

The one PR a human reviews is `<PBI branch>` → inventory `defaultBranch`. This skill **never** completes, auto-completes or pushes to a default, release or environment branch (`[GLB-003]`, `[GLB-024]`).

## When to use

The user names a PBI (ID or URL) that is refined, has child Tasks, and should be built end to end. Companion to `refine-pbi` (architecture detail).

## Hard rules

- **Freshness gate `[GLB-071]` / ADR-0033.** Run `Test-PolicySnapshotFreshness.ps1` (see [reference.md](reference.md#freshness-gate)) before: the first ADO write, each Task start, each PR open, each merge, each state change. Only `InSync` proceeds. Any known-stale verdict: stop and ask the user to regenerate with `New-CursorWorkspace.ps1`. `Unreachable`: proceed only when the user directs that one action; add the ADR-0033 D5 disclosure line to the PR description.
- **ADO only through Tooling `ado/*` scripts** (`[ADO-005]`–`[ADO-007]`). Resolve names and parameters from `Pxp.Unity.Agents.Tooling/ado/README.md` at time of use. No `az boards`, `az repos` or REST calls by hand.
- **Routing at run time** from `routing.json` steps RT-101 to RT-107. No repo-type table in this skill.
- **One repository per PR** (`[GLB-022]`). A multi-repo PBI gets one PBI branch, one set of task PRs and one final PR per repo.
- **Human QA Tasks are not built.** A child Task whose title starts with `Human QA:` (filed by `refine-pbi`) belongs to a person. Never start, implement, re-state or close it; list it in the ledger as excluded and in the report as remaining human work. The agent QA pass in step 4 does not replace it.
- **One Task at a time**, in Task ID order unless the user gave an order. The next Task starts only after the previous task PR is merged into the PBI branch.
- **Merge authority `[GLB-024]`.** Each task PR merge is asked for individually, naming PR id, title, source, target and method. A blanket "go" given earlier is **not** direction (ADR-0033 D5.3: per-action, never standing). No local `git merge` fallback. Never `--bypass-policy`.
- **Target guard.** Before any `Complete-PullRequest.ps1` call, read the PR with `Get-PullRequest.ps1` and refuse unless `Target` is exactly the PBI branch, and is not the inventory `defaultBranch`, `master`, `main`, `development`, `prod` or `releases/*`. The script itself does not check this.
- **PBI `Done` only per `[ADO-010]`**: after the final PR is merged to the default branch by a human. This skill leaves the PBI `Committed`.
- **Independent review.** The session that wrote the code never records its own verdict (SOD-002). Each reviewer is a new subagent with no authoring context.
- **Run folder.** Keep run state in `<workspace root>/.edie/build-pbi/<PBIId>/` ([reference.md](reference.md#run-folder)). Write `ledger.md` at every checkpoint: after the load, after each PR opens, after each verdict, after each merge, after QA, after the final PR. The folder is a working copy; Azure DevOps and `origin` are the authority. It is never inside a repository clone and never holds PAN, secrets or tokens.
- **Merge direction does not carry across sessions.** A yes recorded in the ledger for a PR that is still active is not direction (`[GLB-024]`). Ask again.
- **Reports go to files.** Every subagent writes its full report to `reports/` in the run folder and hands back only the fixed block in [reference.md](reference.md#handback). Do not ask a subagent to repeat its report in chat. Read a report file only when: a verdict raises a question for the user, two reviewers disagree, a loop limit is reached, or the user asks.
- **Conditions file.** Carried review conditions and decisions live in `conditions.md` in the run folder ([reference.md](reference.md#conditions-file)). The orchestrator is its only writer: it appends each `NEW-CONDITIONS` line from a handback and each decision the user or an approver gives, and updates a row's status from `CONDITIONS` / `CONDITIONS-CHECKED`. Briefs name the file; they never restate its rows.
- **Follow-up changes go through the PBI branch.** After the final PR is open, every further change is a follow-up PR into the PBI branch with the same reviews and the same per-PR merge direction ([Follow-up](#follow-up---followup)). Never commit to the PBI branch directly.
- **Other people's threads.** Do not reply on, resolve or close a thread someone else opened on a PR unless the user asks for that reply. Never reply on an analyzer's or bot's thread: fix the code. Threads this skill opened (review records, QA records) may be closed by it.
- Tests ship with the behaviour (implement workflow step 6). Never skip tests to go green.
- Never log, echo or persist PAN or secrets (`[GLB-040]`/`[GLB-041]`). BIN prefixes only.
- **Stop** the slice on ADR-0004 **class 1** (crypto/secrets/HSM/auth internals) and **class 4** (prod desired state). Report.
- Do not create or edit PBIs or Tasks (fields). Do not close the Feature or Epic.
- PowerShell: confirm `$PSVersionTable.PSVersion.Major` is 7 (`[ENV-002]`).
- **Host-app i18n:** if a Task adds or changes ngx-translate keys in a frontend package, the UI is not done until Portal `src/assets/i18n/en.json` (and sibling locales that already have the parent block) contain the keys. That is a required `Pxp.Unity.Portal` slice. If no Task covers it, stop and report the missing Task.

## Preconditions

Stop and ask if any fails.

1. PowerShell 7; `az login` valid; `Get-WorkItem.ps1 -Id <PBIId>` succeeds.
2. Freshness gate is `InSync`.
3. Unity roles are available as subagents. Look for `.claude/agents/<role>.md` in the workspace root. If absent, say once: "Workspace has no Claude bindings; regenerate with `New-CursorWorkspace.ps1 -EmitClaudeBindings` (ADR-0030). Falling back to general-purpose subagents that read `Pxp.Unity.Agents.Policy/agents/<role>.md`." Then continue with the fallback.
4. PBI bar:
   - Type is Product Backlog Item (or a Bug the user named).
   - State is not `Done`, `Removed` or `Rejected`.
   - `Microsoft.VSTS.Common.AcceptanceCriteria` has real intent (no `TBC`).
   - If `Custom.Scope` is populated, every Scope line has at least one AC line (`[ADO-001]`).
   - At least one child Task that is not a `Human QA:` Task (`[ADO-002]`). If none: stop and suggest `refine-pbi` or the `create-task` procedure. This skill does not create Tasks.
   - Area path is in the user's scope, or the user named this ID (`[ADO-012]`).

## Inputs

Required: PBI ID.

Optional: Task order; PBI branch name; repos to include; `--resume` (see [Resume](#resume---resume)); `--tasks-per-session <n>` (default 3; 0 = no limit); `--followup <source>` (see [Follow-up](#follow-up---followup)), where `<source>` is who raised findings on the final PR: a reviewer's name, `sonar`, or a short phrase; `--close` (see [Close](#close---close)).

Load, in order:

1. PBI via `Get-WorkItem.ps1`; Tasks via `Get-WorkItemChildren.ps1 -Id <PBIId> -MaxResults 100`; each Task via `Get-WorkItem.ps1`.
2. Owning repos from PBI Scope, Technical Implementation Details, the Task descriptions (`refine-pbi` writes the detail there), an older `refine-pbi` comment if present, and the source. Not guessed. Map each Task to its repo.
3. Per repo: `repositoryType` and `defaultBranch` from `Pxp.Unity.Agents.Policy/inventory/repositories.json` (`[GLB-002]`; never local git).
4. Current state names from `Get-WorkItem.ps1` output / `ado/README.md`. Do not hard-code beyond the snapshot below.
5. Save what subagents need to `inputs/` in the run folder, once: `acceptance-criteria.txt`, `scope.txt`, `tid.txt` (Technical Implementation Details and any older `refine-pbi` comment), and `task-<TaskId>.txt` per Task (title and full description). Briefs give these paths, not the text. Then write the first `ledger.md` and an empty `conditions.md`.

## Routing

Per repo slice, apply `routing.json` `resolution.steps` ([reference.md](reference.md#routing)):

- RT-101/102: `repoTypes` row → `primary`, `supporting`, `leadReviewer`. `unknown` or unlisted type → apply `resolution.unknownType` / `unlistedType` (triage by `architecture-agent`; do not implement).
- RT-104/105: evaluate `changeClasses` triggers and `overrides` against the Task's intended change; union reviewers, human approver and gate. Apply `carveOuts` (for example embedded `<Repo>.Web/**` Angular libraries route to `frontend-software-agent`).
- RT-103: announce role and basis with rule ids.
- Security review is required for the task PR when any matched class or override lists `security-review-agent`, the repo is an always-on security repo in `routing.md`, or the diff touches auth, crypto, secrets, PAN or card data.

## Workflow

Copy and track:

```
build-pbi <PBIId>:
- [ ] Preconditions + freshness gate
- [ ] Load PBI, Tasks, repos; resolve routing per repo
- [ ] Run folder: inputs/, ledger.md, conditions.md
- [ ] PBI → Committed if [ADO-009] bar met (else leave, comment why)
- [ ] Per repo: PBI branch from origin/<defaultBranch>
- [ ] For each Task, sequentially:
  - [ ] Task → In Progress, AssignedTo = session user ([ADO-011])
  - [ ] Task branch from current origin/<PBI branch>
  - [ ] Primary implements Task + tests; commit AB#<TaskId>; push
  - [ ] Task PR → PBI branch; link Task + PBI
  - [ ] Code review (+ security if routed) in fresh sessions
  - [ ] Rework (max 3 loops) until PASS
  - [ ] Ask user to direct this PR's squash merge
  - [ ] Target guard; Complete-PullRequest -HumanDirected -Squash -DeleteSourceBranch
  - [ ] Confirm squash commit on origin/<PBI branch>; Task → Done
  - [ ] Checkpoint ledger.md; stop if --tasks-per-session reached
- [ ] QA on PBI branch against every AC line (max 3 loops)
- [ ] Final PR PBI branch → defaultBranch (ready, linked, test plan)
- [ ] Comment PBI; PBI stays Committed
- [ ] Report
```

### 1. PBI state at start

If the PBI is below `Committed` and the `[ADO-009]` bar holds (real AC, Scope covered, Tasks exist, blockers resolved or recorded): `Set-WorkItemState.ps1 -Id <PBIId> -State Committed`. Otherwise leave its state and continue only if the user confirms; record why on the PBI with `Add-WorkItemComment.ps1`.

### 2. PBI branch

Per repo, once:

- Name: `feature/<PBIId>-<slug>`, or the squad prefix when the area path maps to a known squad (`[GLB-031]` lists `Spirit/`, `Prime/`, `Saigon/`, `Exodia/`, `Lambda/`, `Panda/`; `Unity\Edison` uses `Edison/`), e.g. `Edison/<PBIId>-<slug>`. `<slug>` is 2–5 lowercase words from the PBI title. Never a name implying a production change (`[GLB-032]`).
- If `origin/<PBI branch>` exists: check out and fast-forward only. Never rebase shared history.
- Else: `New-Branch.ps1 -Repo <repo> -Branch <PBI branch> -BaseRef <defaultBranch>`, then fetch and check out locally.

### 3. Per Task

For each Task (Task ID order unless the user ordered them):

1. **Start.** `Set-WorkItemState.ps1 -Id <TaskId> -State "In Progress" -AssignedTo <session user>` (`[ADO-011]`; if the identity cannot be resolved, stop and ask).
2. **Branch.** `git fetch origin` then create `<PBI branch>-<TaskId>` from `origin/<PBI branch>`. It must include every earlier Task's squash commit.
3. **Implement.** Dispatch the resolved **primary** (subagent `<primary>` or fallback, prompt in [reference.md](reference.md#implementer-prompt)). It follows `orchestration/workflows/implement-backend-feature.md` or `implement-frontend-feature.md` as routing selects, except that the PR target is the PBI branch. Scope is that Task only; the PBI AC is the test oracle; tests in the same change. The primary also writes the task PR description to `pr/<TaskId>.md` (under 4,000 characters, the Azure DevOps limit) and its report to `reports/<TaskId>-impl-r<round>.md`, and hands back the implementer block.
4. **Commit and push.** Subject `AB#<TaskId> <imperative intent>`; body `Parent PBI AB#<PBIId>.` plus a test note. Stage only this Task's files. `git push -u origin <task branch>`.
5. **PR.** `New-PullRequest.ps1 -Repo <repo> -Source <task branch> -Target <PBI branch> -Title "AB#<TaskId> <intent>" -Description (Get-Content -Raw <run folder>/pr/<TaskId>.md)`. The description is the file the primary wrote, per [reference.md](reference.md#task-pr-description) (`[ADO-004]`); open the PR from the file without recomposing it. Name PRs `PR !<id>`, never `#<id>` (`[ADO-013]`).
6. **Link.** `Set-WorkItemPullRequestLink.ps1 -WorkItemId <TaskId>,<PBIId> -PullRequestId <id>`; verify both sides (`link-task-to-pull-request` procedure).
7. **Review.** Launch the reviews in **one** message, in parallel, each a fresh subagent that did not write the code:
   - **Code**: `<leadReviewer>` (e.g. `backend-quality-agent`), following `orchestration/workflows/review-pull-request.md` on the task PR diff (`origin/<PBI branch>...<task branch>`). May add tests; may not change behaviour. `VERDICT: PASS | REWORK`.
   - **Security** (only if routed): `security-review-agent`. `VERDICT: PASS | FAIL | STOP-CLASS-1`.
   - Each reviewer writes its report to `reports/<TaskId>-<role>-r<round>.md` and records its own verdict on the PR with `New-PullRequestThread.ps1` (`review_record: …`, at most 1,500 characters: verdict, head SHA, must-fix lines, conditions). It hands back the reviewer block with the thread id. The orchestrator does not rewrite verdicts into threads.
   - Record each verdict, its head SHA and its round in `ledger.md`; append `NEW-CONDITIONS` lines to `conditions.md`.
8. **Rework.** On `REWORK` / `FAIL`: dispatch the primary with the **paths** of the failing reports, not their text. It fixes on the same task branch with a new commit `AB#<TaskId> Address review: …`, push, re-run every routed review (new commits reset the review). After **3** loops without pass, stop the PBI: leave the Task `In Progress`, comment the blocker on Task and PBI, report. `STOP-CLASS-1` stops immediately.
9. **Direct the merge.** First check that no row in `conditions.md` that applies to this Task is `open`: an open row is rework, or a question for the user if the primary reported it not applicable. Tick the review lines in `pr/<TaskId>.md` and apply it with `Set-PullRequestDescription.ps1`. Then ask the user, and wait:
   > Merge task PR !<id> "AB#<TaskId> <intent>" from `<task branch>` into `<PBI branch>` by squash? Code review PASS, security <PASS|not routed>.
   Only an explicit yes for **this** PR is direction. If the user declines: stop, leave the Task `In Progress`, report.
10. **Merge.** Freshness gate. `Get-PullRequest.ps1 -Id <id>`: check the target guard and that the gate verdict is not BLOCKED. Then `Complete-PullRequest.ps1 -Id <id> -HumanDirected -Squash -DeleteSourceBranch`. If the script refuses (gate BLOCKED, required reviewers, policies): stop and report the blocker. No fallback.
11. **Confirm and close the Task.** `git fetch origin`; confirm `origin/<PBI branch>` has the squash commit. Then `Set-WorkItemState.ps1 -Id <TaskId> -State Done`. Record the squash SHA for the final PR.
12. **Checkpoint.** Rewrite `ledger.md`. If `--tasks-per-session` Tasks have merged in this session and more remain, stop here and tell the user: "Checkpoint written after Task <TaskId>. Start a new session and run `build-pbi <PBIId> --resume` to continue with Task <next>." Otherwise continue.

A Task spanning repos is split into one slice per repo, each committed as `AB#<TaskId>`. The Task goes `Done` only when every slice is merged.

### 4. QA

After every Task of the repo slice is merged, launch one fresh QA subagent (the resolved `<leadReviewer>` role in QA mode, prompt in [reference.md](reference.md#qa-prompt)) on `origin/<PBI branch>`. It runs the app the way the repo documents and checks **every** AC line: `PASS | FAIL | UNVERIFIABLE`. `VERDICT: VERIFIED` only if every line passes. `UNVERIFIABLE` never counts as pass. Its report goes to `reports/qa-r<round>.md`; it posts nothing itself. Record the result and its head SHA in `ledger.md`.

On `NOT VERIFIED`: the primary, given the QA report's path, fixes on a new branch `<PBI branch>-qa-<n>` from `origin/<PBI branch>`, commit `AB#<TaskId> Address QA: …` (the Task whose scope owns the defect, else `AB#<PBIId>`), task PR → PBI branch, code review, then **per-PR merge direction** as in step 3.9–3.10. Re-run QA. After 3 QA loops: stop, comment, report. Tasks already `Done` stay `Done`; the PBI stays `Committed`.

### 5. Final PR

Freshness gate. Then:

1. `New-PullRequest.ps1 -Repo <repo> -Source <PBI branch> -Target <defaultBranch> -Title "AB#<PBIId> <PBI title>" -Description <file>`, **not** draft. Description per [reference.md](reference.md#final-pr-description).
2. `Set-WorkItemPullRequestLink.ps1 -WorkItemId <PBIId>,<TaskIds…> -PullRequestId <id>`.
3. Name the routed human gates and reviewers in the description (change class). Required reviewers come from branch policy.
4. Rows in `conditions.md` that apply to `final PR` go into the description's Unresolved section; rows that apply to `approver` go into "For the approver to confirm". A row still `open` for a Task blocks the final PR. Keep the description in `pr/final.md`.
5. Post the QA result (the criteria table from `reports/qa-r<round>.md`) as a thread on this PR and close it.
6. Never complete, never arm auto-complete on this PR.

### 6. PBI comment

`Add-WorkItemComment.ps1 -Id <PBIId>` with: repo, PBI branch, final PR link (`PR !<id>`), Task → task PR → squash SHA table, QA result, and "PBI stays Committed until the final PR merges to <defaultBranch> ([ADO-010])". HTML, no secrets.

## Resume (`--resume`)

`build-pbi <PBIId> --resume` continues a build in a new session.

1. Preconditions and freshness gate, as for a new run. Resolve routing again; never take it from the ledger.
2. Read `ledger.md` and `conditions.md`. Do not read `reports/`.
3. Reconcile each Task against Azure DevOps and `origin` ([reference.md](reference.md#resume-reconcile)). Where the ledger disagrees, Azure DevOps and `origin` win; correct the ledger and say what changed.
4. If the run folder is missing (another machine, or deleted): rebuild `inputs/` and `ledger.md` from the PBI, the Task states, the PRs linked to each Task, and `git log origin/<PBI branch>`. `conditions.md` cannot be rebuilt without a Tooling script that lists PR threads; stop and ask the user whether to continue without the carried conditions.
5. Tell the user the resume point in one line per Task (done, or its stage) and continue from the first Task that is not done. Stages after the Tasks (QA, final PR, PBI comment) resume the same way.
6. A PR that was waiting for merge direction is asked about again.

## Follow-up (`--followup`)

`build-pbi <PBIId> --followup <source>`, run while the final PR is active, for findings raised on it (a reviewer's comments, analyzer findings).

1. **Start.** Preconditions, freshness gate, then resume reconcile. The final PR must be `active`. If it is completed or abandoned, stop: the change needs its own work item.
2. **Collect.** Read the final PR's threads from `<source>` and save them unedited to `inputs/followup-<n>-findings.md`. Use a Tooling `ado/*` script; if none lists PR threads, ask the user to paste the findings or to approve the read. Do not paste them into chat.
3. **Classify.** One row per finding in `pr/followup-<n>-plan.md` ([reference.md](reference.md#follow-up-plan)): `fix`, `settled` (a Decisions row already answers it), `decide` (the implementation would fork), `separate item` (outside this PBI), or `not a change` (the reviewer accepted it as built).
4. **Stop on scope.** If a finding changes an acceptance criterion or the PBI's Scope, stop and report it. This skill does not edit PBI fields.
5. **Agree the plan.** Show the user the counts per class and every `decide` row as a question. Wait. Record each answer, and each ruling the reviewer gave, as a Decisions row in `conditions.md`, marked by who decided; a ruling that changes an earlier decision supersedes it.
6. **Build.** Branch `<PBI branch>-fu-<n>` from `origin/<PBI branch>`. Resolve routing again on the intended change: a follow-up can hit a change class no Task did. Then run workflow steps 3.3 to 3.11 with these differences:
   - commits are `AB#<PBIId> Address <source>: <intent>`, one commit per group of findings when the user asked for ordered commits;
   - the PR links the PBI only;
   - no Task changes state. Tasks stay `Done`.
   Class 1 or class 4 stops the follow-up, as for a Task.
7. **QA again.** Launch QA on the new head of the PBI branch, unless every routed reviewer's handback says `BEHAVIOUR: unchanged`. In that case keep the earlier QA result and say so in the final PR ("VERIFIED at <sha>; <sha> reviewed as no behaviour change"). A `NOT VERIFIED` result follows workflow step 4.
8. **Bring the final PR up to date.** Rewrite `pr/final.md` and apply it with `Set-PullRequestDescription.ps1`: a row for the follow-up PR, the QA line with its SHA, and the `approver` rows from `conditions.md`. If QA ran, post it as a thread and close it. Add a PBI comment with the follow-up row. Update `ledger.md`.
9. **Report.** Findings fixed, findings left and why, what the reviewer still has to confirm. Say that new commits may have reset votes on the final PR. Never complete the final PR and never arm auto-complete.

Limits: three review loops per follow-up PR, as for a Task. Follow-ups are numbered and can repeat while the final PR is active.

## Close (`--close`)

`build-pbi <PBIId> --close`, run after the human merged the final PR:

1. Freshness gate.
2. `Get-PullRequest.ps1` on the final PR: status must be `completed` into the inventory `defaultBranch`.
3. Every child Task is `Done` or descoped with a recorded reason. This includes each `Human QA:` Task: if one is not `Done`, stop and say a person still has to run it; never set it `Done`.
4. QA recorded every AC line `PASS` (from the PBI comment); if not, stop.
5. `Set-WorkItemState.ps1 -Id <PBIId> -State Done`; comment the merge commit.

## Output to the user

- Per repo: PBI branch and final PR link.
- Table: Task, task PR, squash SHA, review verdicts, state.
- QA table: each AC line and result.
- Stopped or skipped items (class 1/4, rework limit, declined merge, gate blocked) and what each needs.
- Remaining human work: run each `Human QA:` Task; review and merge the final PR, then `build-pbi <PBIId> --close`. If findings come back on the final PR: `build-pbi <PBIId> --followup <source>`.
- Take the tables from `ledger.md`; do not rebuild them from the conversation.

## Additional resources

- Freshness, routing, prompts, PR and comment templates: [reference.md](reference.md)
- Worked example: [examples.md](examples.md)
