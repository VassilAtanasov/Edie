# Example: PBI 306980

Illustrative run. PBI 306980 "Gateway Portal BIN APIs" under Feature 306968 (area `Unity\Edison`), with Tasks 306983 and 306984 as listed in the build-feature example. Confirm current titles, states and repos with `Get-WorkItem.ps1` before relying on them.

Read-only check on 2026-09-30: `Get-WorkItemChildren.ps1 -Id 306980` returned no child work items, so a real run today stops at the PBI bar ("at least one child Task") and suggests `refine-feature` / `create-task`. The same check found the `C:\Unity` workspace `BehindRegenerate` (73 commits), so the freshness gate would stop first.

Invoke: `build-pbi 306980`

## Resolved at run time

| Item | Value |
|---|---|
| Repo | `Pxp.Unity.Api.Gateway` (from PBI Scope; confirm in source) |
| `repositoryType`, `defaultBranch` | from `inventory/repositories.json` |
| Primary / lead reviewer | from `routing.json` `repoTypes` (e.g. `backend-software-agent` / `backend-quality-agent` for `backend-service`) |
| Change class | class 5 (public API) if the routes are merchant-facing → reviewers and human gate unioned in; security routed if auth is touched |
| PBI branch | `Edison/306980-gateway-portal-bin-apis` |

## Sequence

```text
PBI 306980            → Committed (ADO-009 bar met)
Task 306983           → In Progress, assigned to you
  branch  Edison/306980-gateway-portal-bin-apis-306983 from origin/PBI branch
  commit  AB#306983 Add BIN blacklist read endpoints
  PR !A   → Edison/306980-gateway-portal-bin-apis
  review  backend-quality-agent PASS; security-review-agent PASS
  ask     "Merge task PR !A … by squash?"  → yes
  merge   Complete-PullRequest -HumanDirected -Squash -DeleteSourceBranch
Task 306983           → Done
Task 306984           → In Progress
  branch  …-306984 from origin/PBI branch (includes 306983 squash)
  commit  AB#306984 Add BIN blacklist write endpoints and tests
  PR !B   → review REWORK (missing 404 test) → fix → PASS
  ask     "Merge task PR !B … by squash?"  → yes
Task 306984           → Done
QA on PBI branch      → every AC line PASS → VERIFIED
Final PR !C           Edison/306980-gateway-portal-bin-apis → <defaultBranch>, ready for review
PBI 306980            stays Committed; comment with Task/PR/SHA table
```

Human reviews and merges PR !C, then:

`build-pbi 306980 --close` → confirms PR !C completed into `<defaultBranch>` → PBI `Done`.
