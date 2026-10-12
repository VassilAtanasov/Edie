# Build PBI — reference

## Paths

Unity workspace root (adjust if the clone root differs):

- Policy: `Pxp.Unity.Agents.Policy/`
- Routing: `Pxp.Unity.Agents.Policy/orchestration/routing.json` (readable view `routing.md`)
- Inventory: `Pxp.Unity.Agents.Policy/inventory/repositories.json`
- Scripts: `Pxp.Unity.Agents.Tooling/ado/` (confirm names in `ado/README.md`)
- Freshness: `Pxp.Unity.Agents.Tooling/workspace-setup/Test-PolicySnapshotFreshness.ps1`

- Run folder: `.edie/build-pbi/<PBIId>/` at the workspace root (see [Run folder](#run-folder))

`AssignedTo`: Azure DevOps identity of the engineer running the session.

## Run folder

```
<workspace root>/.edie/build-pbi/<PBIId>/
  ledger.md            run state, rewritten at each checkpoint
  conditions.md        carried conditions and decisions
  inputs/              saved once: acceptance-criteria.txt, scope.txt, tid.txt, task-<TaskId>.txt,
                       followup-<n>-findings.md
  reports/             <TaskId>-<role>-r<round>.md, fu<n>-<role>-r<round>.md, qa-r<round>.md
  pr/                  <TaskId>.md, followup-<n>.md, followup-<n>-plan.md, final.md
```

- It is a working copy, not authority. Azure DevOps and `origin` decide; the folder is corrected to match them.
- No PAN, secrets or tokens (`[GLB-040]`/`[GLB-041]`). Reports quote file and line, not secret values.
- It is never committed and never placed inside a repository clone.
- Routing is not stored as a decision. It is resolved again in every session; the ledger only notes what was resolved, for the report.

Ledger shape (rewrite the whole file at each checkpoint):

```markdown
# build-pbi <PBIId>: ledger

Updated: <UTC time>   Policy: InSync @ <sha>
Repo: <repo>   Default branch: <defaultBranch>   PBI branch: <PBI_BRANCH> @ <sha>
Routing (resolved this session, for the report only): primary <role>, lead <role>, classes <list>
Order: <TaskId>, <TaskId>, …   Excluded: <TaskIds and why>

| Task | State | Branch | PR | Head | Code | Other reviews | Round | Squash | Stage |
|---|---|---|---|---|---|---|---|---|---|
| <TaskId> | Done | (deleted) | !<id> | <sha> | PASS | security PASS | 2 | <sha> | merged |
| <TaskId> | In Progress | …-<TaskId> | !<id> | <sha> | PASS | security REWORK | 1 | - | rework |

QA: <not run | VERIFIED n/m @ sha | NOT VERIFIED @ sha, round n>
Final PR: <none | !id, status>
Follow-up PRs into the PBI branch: <none | n: source, !id, squash sha>
Waiting on the user: <question, or none>
```

`Stage` is one of: `not started`, `implementing`, `pr open`, `in review`, `rework`, `awaiting merge direction`, `merged`.

## Resume reconcile

| Azure DevOps and origin show | Stage | Next action |
|---|---|---|
| Task `Done`, PR completed, `AB#<TaskId>` commit on `origin/<PBI branch>` | merged | skip |
| PR completed, commit on the PBI branch, Task not `Done` | merged | set the Task `Done` |
| PR active, a verdict from every routed reviewer at the PR's head SHA, all PASS | awaiting merge direction | ask again |
| PR active, any verdict REWORK or FAIL at the head SHA | rework | dispatch the primary with the report paths |
| PR active, a verdict missing or recorded at an older SHA | in review | run the missing reviews |
| Task `In Progress`, task branch on origin, no PR | implementing | inspect the branch; finish, or open the PR |
| Task `In Progress`, no branch | not started | create the branch |
| Task `To Do` | not started | start the Task |
| PR abandoned | (stop) | ask the user |

A verdict counts only if the ledger records the head SHA it was given at and that SHA is still the PR's head.

## Handback

Every subagent ends with exactly the block for its role, at most 200 words, and nothing after it. The full report is in the file named in `REPORT`.

Implementer:

```text
RESULT: DONE | BLOCKED | STOP-CLASS-<n>
HEAD: <commit sha>
TESTS: <project>: <passed>/<failed>; …
REPORT: <path>
PR-DESCRIPTION: <path>
CONDITIONS: <id> met | <id> not applicable: <reason> | none
DEVIATIONS: <one line each, or none>
QUESTIONS: <only what a human must decide, one line each, or none>
```

Reviewer (code, security, any change-class reviewer):

```text
VERDICT: PASS | REWORK | FAIL | STOP-CLASS-1
HEAD: <sha reviewed>
REPORT: <path>
THREAD: <review_record thread id>
MUST-FIX: <n>. <file:line> <one line>   (or none)
NEW-CONDITIONS: <applies to: TaskId | all | final PR | approver> <one line>   (or none)
CONDITIONS-CHECKED: <id> met | <id> not met   (or none)
BEHAVIOUR: unchanged | changed: <one line>
QUESTIONS: <or none>
```

`BEHAVIOUR: unchanged` only when no request, response, stored value, log level or error outcome differs after the change.

QA:

```text
VERDICT: VERIFIED | NOT VERIFIED
HEAD: <sha>
REPORT: <path>
CRITERIA: <n> PASS, <n> FAIL, <n> UNVERIFIABLE
FAILED: <criterion number> <one line>   (or none)
NOT-EXERCISED: <one line each, or none>
```

## Conditions file

```markdown
# build-pbi <PBIId>: conditions and decisions

## Conditions
| Id | From | Applies to | Condition | Status |
|---|---|---|---|---|
| C-001 | PR !<id> security r1 | <TaskId> | <one line> | met @ <sha> (code r1) |
| C-002 | PR !<id> architecture r2 | approver | <one line> | open |

## Decisions
| Id | Question | Answer | By | Date |
|---|---|---|---|---|
| D-001 | <one line> | <one line> | directing engineer | <date> |
```

- `Applies to` is a Task id, `all`, `final PR` or `approver`.
- `Status` is `open`, `met @ <sha> (<who confirmed>)`, `not applicable (<who agreed>)` or `dropped by D-<n>`.
- One line per row. Detail stays in the report the row came from.
- A decision that changes an earlier one adds a new row and marks the old one `superseded by D-<n>`.
- Only the orchestrator writes this file.

## Freshness gate

```powershell
$fresh = & ./Pxp.Unity.Agents.Tooling/workspace-setup/Test-PolicySnapshotFreshness.ps1 `
    -WorkspaceRoot <workspace root> -PolicyBranch main -Json | ConvertFrom-Json
$LASTEXITCODE   # 0 = InSync, 2 = Unreachable, other = stale
```

| Verdict | Action |
|---|---|
| `InSync` | Proceed. |
| `BehindRegenerate`, `BehindPull`, `Diverged`, `Inconsistent`, `NotGenerated`, `PolicyRepoMissing` | Stop. Show the JSON `Remediation` command (for `BehindRegenerate`: `pwsh -File <tooling>/workspace-setup/New-CursorWorkspace.ps1 -SkipClone <root>`; add `-EmitClaudeBindings` for Claude subagents) and ask the user to run it. |
| `Unreachable` | Stop unless the user directs this one action. Then add to the PR description: `Policy freshness: Unreachable at <UTC time>; proceeded on human direction ([GLB-071], ADR-0033 D5).` |

Run it at every unit-of-work boundary: before the first ADO write, each Task start, each PR open, each merge, each state change.

## Routing

Read `routing.json` once per repo slice and walk `resolution.steps`:

| Step | Do |
|---|---|
| RT-101 | `repositoryType` for the repo from `inventory/repositories.json`. |
| RT-102 | Matching `repoTypes[].outcome`: `primary`, `supporting`, `leadReviewer`. Unknown/unlisted type → `resolution.unknownType` / `unlistedType`. |
| RT-103 | Announce: "Adopting `<primary>` for `<repo>` (`<repositoryType>`, routing.json `<RT-2xx>`, [GLB-070])." |
| RT-104 | Evaluate each `changeClasses[]` trigger against the Task's change; union reviewers, human approver, gate. |
| RT-105 | Evaluate `overrides[]`; union reviewers; force `requiresChangeClasses`; settle conflicts by `resolution.conflict`. Apply `carveOuts[]`. |
| RT-106 | Triage-only outcome → stop, hand to lead reviewer. Else branch + PR + evidence to lead reviewer. |
| RT-107 | Merge only by human-directed completion once all gates pass. |

`separationOfDuty` (SOD-001 human author ≠ approver; SOD-002 authoring session ≠ reviewing session) applies to every verdict.

Subagent type: `<role>` when `.claude/agents/<role>.md` exists; else `general-purpose` with "Operate as `<role>`. Read `Pxp.Unity.Agents.Policy/agents/<role>.md` first."

## Implementer prompt

```text
Operate as <PRIMARY>. Read Pxp.Unity.Agents.Policy/agents/<primary>.md and
Pxp.Unity.Agents.Policy/orchestration/workflows/<implement-workflow>.md.
Follow the workflow except: the branch already exists and the PR target is
<PBI_BRANCH>, not the default branch. Do not open the PR; the orchestrator does.

Repository: <ABS_REPO>   Branch: <TASK_BRANCH> (checked out, from origin/<PBI_BRANCH>)
Task AB#<TaskId>: <title>   Parent PBI AB#<PBIId>
Run folder: <RUN_FOLDER>
Read from it: inputs/task-<TaskId>.txt, inputs/acceptance-criteria.txt (test oracle),
inputs/scope.txt, inputs/tid.txt, conditions.md.
Conditions: every open row that applies to Task <TaskId> or to "all" is in scope.
Every row under Decisions is settled; do not reopen it.

Implement only this Task's scope. Tests ship in the same change.
Change only this repository. Never log PAN or secrets.
If the change hits ADR-0004 class 1 or 4, stop and say so.
Commit: "AB#<TaskId> <imperative intent>" with body "Parent PBI AB#<PBIId>."

Write the task PR description to pr/<TaskId>.md, under 4,000 characters, in this shape:
<Task PR description template>
Write your full report to reports/<TaskId>-impl-r<round>.md: files changed, tests added
and their result, anything unknown. Never put PAN, secrets or tokens in it.
End with the implementer handback block and nothing else. Do not repeat the report.
<Handback: implementer block>
```

Rework round, add:

```text
Round <n>. Fix the must-fix items in: <report paths>. New commit
"AB#<TaskId> Address review: <intent>". Update pr/<TaskId>.md if the evidence changed.
```

## Code-review prompt

```text
You did not write this code. Operate as <LEAD_REVIEWER>.
Read Pxp.Unity.Agents.Policy/agents/<lead-reviewer>.md and
Pxp.Unity.Agents.Policy/orchestration/workflows/review-pull-request.md.

Repository: <ABS_REPO>
PR: !<id>  Source: <TASK_BRANCH>  Target: <PBI_BRANCH>
Diff: git diff origin/<PBI_BRANCH>...<TASK_BRANCH>

Task AB#<TaskId>: <title>   Parent PBI AB#<PBIId>
Run folder: <RUN_FOLDER>
Read from it: inputs/task-<TaskId>.txt, inputs/acceptance-criteria.txt, conditions.md.

Hunt: correctness, Task scope gaps, missing or hollow tests, contract breaks,
secrets/PAN, change-class triggers. Ignore formatter noise.
You may add tests; do not change product behaviour. Say if you added tests.
Check the open conditions that apply to Task <TaskId> or to "all". Put anything a
later Task, the final PR or the approver must honour in NEW-CONDITIONS, one line each.

Verdict is PASS or REWORK; every must-fix item has file:line evidence.
<Reviewer closing lines>
```

Reviewer closing lines (code, security and change-class reviewers):

```text
Write your full report to <RUN_FOLDER>/reports/<TaskId>-<role>-r<round>.md.
Never put PAN, secrets or tokens in it.
Record your verdict yourself, in a comment that starts "review_record:" and stays
under 1,500 characters (verdict, head SHA, must-fix lines, new conditions):
  New-PullRequestThread.ps1 -Repo <repo> -PullRequestId <id> -Comment (Get-Content -Raw <file>)
End with the reviewer handback block and nothing else. Do not repeat the report.
<Handback: reviewer block>
```

Round 2 or later, add:

```text
Round <n>. The earlier report is <path>. Check its must-fix items and anything the
new commits touch.
```

## Security-review prompt

```text
You did not write this code. Operate as security-review-agent.
Read Pxp.Unity.Agents.Policy/agents/security-review-agent.md.

Repository: <ABS_REPO>
Diff: git diff origin/<PBI_BRANCH>...<TASK_BRANCH>
Routed because: <change class / override / always-on repo / diff touches X>

Class 1 (crypto, secrets, HSM, inter-service auth internals) = STOP.
PAN or secrets in the diff = FAIL. BIN prefixes only is fine.

Read <RUN_FOLDER>/conditions.md and check the open rows that apply to Task <TaskId> or to "all".

Verdict is PASS, FAIL or STOP-CLASS-1, with mitigations and evidence in the report.
<Reviewer closing lines>
```

## QA prompt

```text
You did not write this code. Operate as <LEAD_REVIEWER> in QA mode: an
independent verifier. Do not review source for style; exercise the running app.

PBI AB#<PBIId> acceptance criteria: <RUN_FOLDER>/inputs/acceptance-criteria.txt
(quote each line in your report).
Repository: <ABS_REPO> on origin/<PBI_BRANCH>.
How to run: repo README / AGENTS.md / docs. Do not invent hosts or ports.
Tools: HTTP client, browser tools, or the repo's test host, as evidenced.

For EACH criterion: PASS | FAIL | UNVERIFIABLE — criterion — what you did.
Never mark UNVERIFIABLE as PASS. Never install new tooling; missing tool =
UNVERIFIABLE. Never log PAN.

Operator UI (Portal / ngx-translate): FAIL any label criterion if the host app
(Portal localhost:1010) shows dotted keys instead of text.

VERIFIED only if every line is PASS.
Write your full report to <RUN_FOLDER>/reports/qa-r<round>.md, with one table row per
criterion and a list of what you did not exercise. Post nothing to Azure DevOps.
End with the QA handback block and nothing else. Do not repeat the report.
<Handback: QA block>
```

If the app cannot start, that is `NOT VERIFIED` and counts as a QA loop.

## Follow-up plan

`pr/followup-<n>-plan.md`, one row per finding:

```markdown
# Follow-up <n> on PR !<final id>: findings from <source>

| # | Thread | Finding (one line) | Class | Action or question |
|---|---|---|---|---|
| 1 | <thread id> | <finding> | fix | <what changes> |
| 2 | <thread id> | <finding> | decide | <the question for the user> |
| 3 | <thread id> | <finding> | separate item | <who raises it, where> |
| 4 | <thread id> | <finding> | not a change | Accepted as built |
```

Classes: `fix`, `settled`, `decide`, `separate item`, `not a change`.

Follow-up implementer, add to the implementer prompt (in place of the Task lines):

```text
This is follow-up <n> on the PBI's final PR, not a Task.
Findings: <RUN_FOLDER>/inputs/followup-<n>-findings.md
Plan: <RUN_FOLDER>/pr/followup-<n>-plan.md. Implement rows <numbers>. Leave every other row alone.
A test whose expectation changes must trace to a plan row; name the row in your report.
Commit: "AB#<PBIId> Address <source>: <intent>". PR description: pr/followup-<n>.md.
Report: reports/fu<n>-impl-r<round>.md.
```

Follow-up reviewers, add:

```text
This is follow-up <n>. Plan: <RUN_FOLDER>/pr/followup-<n>-plan.md.
Check each `fix` row is done and nothing outside the plan changed.
A changed test expectation with no plan row behind it is a must-fix.
Report: reports/fu<n>-<role>-r<round>.md.
```

## Merge direction question

Ask exactly one PR per question:

```text
Merge task PR !<id> "AB#<TaskId> <intent>"
  from <TASK_BRANCH> into <PBI_BRANCH> by squash?
  Code review: PASS (<reviewer>). Security: <PASS | not routed>.
  Target guard: OK (not the default or a release branch).
```

Proceed only on an explicit yes that refers to this PR.

## Target guard

```powershell
./Pxp.Unity.Agents.Tooling/ado/Get-PullRequest.ps1 -Id <id>
# Read "Target:" (refs/heads/<name>) and the GLB-023 gate verdict.
```

Refuse when the target is not exactly `refs/heads/<PBI_BRANCH>`, or equals the inventory `defaultBranch`, `main`, `master`, `development`, `prod`, or starts with `releases/`. Refuse when the verdict is BLOCKED.

## Task PR description

```markdown
AB#<TaskId> — <Task title>. Parent PBI AB#<PBIId>.

## Intent
<what this Task changes and why, 2–4 sentences>

## Evidence
<files, tests, commands run and results>

## Guidelines and ADRs
<rule ids / ADRs that apply>

## Reviewers
Lead: <leadReviewer>. Additional: <change-class reviewers or none>.

## Unresolved
<unknowns or none>

## Test plan
- [ ] Unit/integration tests added for <behaviour> pass
- [ ] Code review verdict PASS recorded
- [ ] <security review PASS recorded | security not routed>
```

## Final PR description

```markdown
AB#<PBIId> — <PBI title>. Tasks: AB#<TaskId>, AB#<TaskId>, …

## Intent
<plain prose summary of what the PBI builds>

## Tasks
| Task | Task PR | Squash commit | Review |
|---|---|---|---|
| AB#<TaskId> <title> | [PR !<id>](<url>) | <sha> | PASS / security PASS |
| Follow-up <n>: <source> | PR !<id> | <sha> | PASS |

## Changes from <source> (PR !<id>)
<only after a follow-up: one line per group of `fix` rows>

## QA against acceptance criteria
<VERIFIED at <sha>: n of m criteria PASS. Per-criterion results are in a thread on this PR.>
<after a follow-up with no QA re-run: "<sha> reviewed as no behaviour change">

## For the approver to confirm
<`approver` rows from conditions.md, numbered; or none>

## Gates
Change class: <n or none>. Human approver: <role>. Reviewers: <list>.
Policy freshness: <InSync @ sha | disclosure line>.

## Unresolved
<or none>

## Test plan
- [ ] All task PRs reviewed and squash-merged into <PBI_BRANCH>
- [ ] QA VERIFIED for every acceptance criterion
- [ ] <change-class human gate items>
```

In ADO markdown name PRs `PR !<id>`; in repository Markdown use the full link `[PR !<id>](<url>)` (`[ADO-013]`).

## PBI comment (HTML)

```html
<p><b>build-pbi</b>: <repo> — PBI branch <code><PBI_BRANCH></code>, final PR !<id> to <code><defaultBranch></code>.</p>
<table><tr><th>Task</th><th>Task PR</th><th>Squash</th><th>State</th></tr>
<tr><td>AB#<TaskId></td><td>PR !<id></td><td><sha></td><td>Done</td></tr></table>
<p>QA: <VERIFIED | NOT VERIFIED>, <n>/<m> criteria PASS.</p>
<p>PBI stays Committed until PR !<id> merges to <defaultBranch> ([ADO-010]). Close with <code>build-pbi <PBIId> --close</code>.</p>
```

Pass long comments through a file variable, never secrets on the command line.

## State snapshot

`PXP Scrum - Unity` (point-in-time, confirm with `Get-WorkItem.ps1`):

| Item | States used here |
|---|---|
| Task | `To Do` → `In Progress` → `Done` |
| PBI | `New`/`Approved` → `Committed` → `Done` (Done only via `--close`) |

## Stop conditions

| Condition | State left | Report |
|---|---|---|
| Freshness not InSync | unchanged | verdict + regenerate command |
| No Tasks / AC `TBC` | unchanged | suggest create-task |
| Class 1 or 4 | Task In Progress | class and evidence |
| 3 review or QA loops | Task In Progress, PBI Committed | must-fix list |
| User declines a merge | Task In Progress | PR link |
| `Complete-PullRequest` refuses | Task In Progress | gate blocker |
| `--tasks-per-session` reached | Tasks merged so far Done | checkpoint and the `--resume` command |
| Run folder missing on `--resume` | unchanged | ask whether to continue without carried conditions |
| Final PR completed or abandoned at `--followup` | unchanged | needs its own work item |
| A follow-up finding changes AC or Scope | unchanged | the finding and the criterion it touches |
