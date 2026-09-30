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

Human reviews and merges PR !C, then:

`build-pbi 312486 --close` → confirms PR !C completed into `<defaultBranch>` → PBI `Done`.
