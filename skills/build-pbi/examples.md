# Example: PBI 312486

Illustrative run for PBI 312486. The repo, Task IDs, titles and states below are placeholders; read the real ones with `Get-WorkItem.ps1 -Id 312486` and `Get-WorkItemChildren.ps1 -Id 312486` before relying on them.

Invoke: `build-pbi 312486`

## Resolved at run time

| Item | Value |
|---|---|
| Repo | from PBI Scope; confirm in source |
| `repositoryType`, `defaultBranch` | from `inventory/repositories.json` |
| Primary / lead reviewer | from `routing.json` `repoTypes` (e.g. `backend-software-agent` / `backend-quality-agent` for `backend-service`) |
| Change class | from ADR-0004; e.g. class 5 (public API) unions in its reviewers and human gate; security routed if auth is touched |
| PBI branch | `<Area>/312486-<pbi-slug>` |

## Sequence

```text
PBI 312486            → Committed (ADO-009 bar met)
Task <T1>             → In Progress, assigned to you
  branch  <Area>/312486-<pbi-slug>-<T1> from origin/PBI branch
  commit  AB#<T1> <task title>
  PR !A   → <Area>/312486-<pbi-slug>
  review  lead reviewer PASS; security-review-agent PASS (if routed)
  ask     "Merge task PR !A … by squash?"  → yes
  merge   Complete-PullRequest -HumanDirected -Squash -DeleteSourceBranch
Task <T1>             → Done
Task <T2>             → In Progress
  branch  …-<T2> from origin/PBI branch (includes <T1> squash)
  commit  AB#<T2> <task title>
  PR !B   → review REWORK (missing test) → fix → PASS
  ask     "Merge task PR !B … by squash?"  → yes
Task <T2>             → Done
QA on PBI branch      → every AC line PASS → VERIFIED
Final PR !C           <Area>/312486-<pbi-slug> → <defaultBranch>, ready for review
PBI 312486            stays Committed; comment with Task/PR/SHA table
```

Every subagent above writes its report under `.edie/build-pbi/312486/reports/` and hands back a short block; the ledger and conditions files sit beside it.

### Checkpoint and resume

With the default `--tasks-per-session 3`, a PBI with more Tasks stops after the third merge:

```text
Task <T3>             → Done
checkpoint            ledger.md written
stop                  "Start a new session and run build-pbi 312486 --resume to continue with Task <T4>."
```

New session, `build-pbi 312486 --resume`:

```text
freshness InSync; routing resolved again
reconcile             <T1>–<T3> merged; <T4> not started
Task <T4>             → In Progress …
```

### Follow-up on the final PR

A reviewer comments on PR !C. `build-pbi 312486 --followup "<reviewer name>"`:

```text
collect               threads → inputs/followup-1-findings.md
plan                  5 fix, 1 decide, 1 separate item, 3 not a change
ask                   the one `decide` row → answer recorded as D-00n
branch                <Area>/312486-<pbi-slug>-fu-1
commit                AB#312486 Address <reviewer name>: <intent>
PR !D                 → PBI branch; reviews PASS; "Merge PR !D … by squash?" → yes
QA                    re-run on the new head → VERIFIED
PR !C                 description updated; QA thread posted and closed
```

Human reviews and merges PR !C, then:

`build-pbi 312486 --close` → confirms PR !C completed into `<defaultBranch>` → PBI `Done`.
