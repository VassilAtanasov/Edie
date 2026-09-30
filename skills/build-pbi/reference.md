# Build PBI — reference

## Paths

Unity workspace root (adjust if the clone root differs):

- Policy: `Pxp.Unity.Agents.Policy/`
- Routing: `Pxp.Unity.Agents.Policy/orchestration/routing.json` (readable view `routing.md`)
- Inventory: `Pxp.Unity.Agents.Policy/inventory/repositories.json`
- Scripts: `Pxp.Unity.Agents.Tooling/ado/` (confirm names in `ado/README.md`)
- Freshness: `Pxp.Unity.Agents.Tooling/workspace-setup/Test-PolicySnapshotFreshness.ps1`

`AssignedTo`: Azure DevOps identity of the engineer running the session.

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
Task AB#<TaskId>: <title>
<Task description>
Parent PBI AB#<PBIId> acceptance criteria (test oracle):
<AC lines>
Technical detail (refine-pbi comment / TID), if any:
<text>

Implement only this Task's scope. Tests ship in the same change.
Change only this repository. Never log PAN or secrets.
If the change hits ADR-0004 class 1 or 4, stop and say so.
Commit: "AB#<TaskId> <imperative intent>" with body "Parent PBI AB#<PBIId>."
Report: files changed, tests added and their result, anything unknown.
```

## Code-review prompt

```text
You did not write this code. Operate as <LEAD_REVIEWER>.
Read Pxp.Unity.Agents.Policy/agents/<lead-reviewer>.md and
Pxp.Unity.Agents.Policy/orchestration/workflows/review-pull-request.md.

Repository: <ABS_REPO>
PR: !<id>  Source: <TASK_BRANCH>  Target: <PBI_BRANCH>
Diff: git diff origin/<PBI_BRANCH>...<TASK_BRANCH>

Task AB#<TaskId>: <title and description>
Parent PBI AB#<PBIId> acceptance criteria:
<AC lines>

Hunt: correctness, Task scope gaps, missing or hollow tests, contract breaks,
secrets/PAN, change-class triggers. Ignore formatter noise.
You may add tests; do not change product behaviour. Say if you added tests.

End with exactly:
VERDICT: PASS
or
VERDICT: REWORK
and a numbered list of must-fix items with file:line evidence.
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

End with VERDICT: PASS | FAIL | STOP-CLASS-1, with mitigations and evidence.
```

## QA prompt

```text
You did not write this code. Operate as <LEAD_REVIEWER> in QA mode: an
independent verifier. Do not review source for style; exercise the running app.

PBI AB#<PBIId> acceptance criteria (quote each line).
Repository: <ABS_REPO> on origin/<PBI_BRANCH>.
How to run: repo README / AGENTS.md / docs. Do not invent hosts or ports.
Tools: HTTP client, browser tools, or the repo's test host, as evidenced.

For EACH criterion: PASS | FAIL | UNVERIFIABLE — criterion — what you did.
Never mark UNVERIFIABLE as PASS. Never install new tooling; missing tool =
UNVERIFIABLE. Never log PAN.

Operator UI (Portal / ngx-translate): FAIL any label criterion if the host app
(Portal localhost:1010) shows dotted keys instead of text.

End with VERDICT: VERIFIED (every line PASS) or VERDICT: NOT VERIFIED.
```

If the app cannot start, that is `NOT VERIFIED` and counts as a QA loop.

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

## QA against acceptance criteria
| Criterion | Result | Evidence |
|---|---|---|

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
| No Tasks / AC `TBC` | unchanged | suggest refine-feature / create-task |
| Class 1 or 4 | Task In Progress | class and evidence |
| 3 review or QA loops | Task In Progress, PBI Committed | must-fix list |
| User declines a merge | Task In Progress | PR link |
| `Complete-PullRequest` refuses | Task In Progress | gate blocker |
