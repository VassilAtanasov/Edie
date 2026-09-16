# Build Feature — reference

## Tooling paths

Unity workspace (adjust if the clone root differs):

- Policy: `Pxp.Unity.Agents.Policy/`
- Scripts: `Pxp.Unity.Agents.Tooling/ado/`
- Inventory: `Pxp.Unity.Agents.Policy/inventory/repositories.json`

Confirm script names in `Pxp.Unity.Agents.Tooling/ado/README.md` before invoking.

Directing human for `AssignedTo`: `vassil.atanasov@pxp.io`.

## Code-review subagent prompt

```text
You did not write this code. Operate as <QUALITY_AGENT>.
Read Pxp.Unity.Agents.Policy/agents/<quality-agent>.md and
Pxp.Unity.Agents.Policy/orchestration/workflows/review-pull-request.md.

Full repository path: <ABS_REPO>
Diff: branch changes
Base branch: <INTEGRATION_BRANCH>   (not development unless they are the same)

Work item: AB#<PBI> — acceptance criteria attached below.
Hunt: correctness, AC gaps, missing/hollow tests, secrets/PAN, contract/PCI.
Do not comment on formatter/lint noise.
You may add tests; do not change product behavior. If you add tests, say so.

End with exactly:
VERDICT: PASS
or
VERDICT: REWORK
and a numbered list of must-fix items.
```

## Security-review subagent

Prefer Cursor `security-review` with:

```text
Full Repository Path: <ABS_REPO>
Diff: branch changes
Base Branch: <INTEGRATION_BRANCH>
Custom Instructions: Unity security-review-agent. Read
Pxp.Unity.Agents.Policy/agents/security-review-agent.md.
This merge target is a Feature integration branch, not production.
Flag class 1 (crypto/secrets/HSM/inter-service auth internals) as STOP.
PAN/secrets in diff = FAIL. BIN prefixes only is OK.
End with VERDICT: PASS | FAIL | N/A | STOP-CLASS-1
```

## QA subagent prompt

```text
You did not write this code. You are an independent QA verifier.
Do not review source for correctness; exercise the running application.

Work item AB#<PBI> acceptance criteria (quote each line).
Repo: <ABS_REPO> on branch <PBI_BRANCH>.
How to run: repo README / AGENTS.md / docs — do not invent hosts.
QA tooling: HTTP client, browser tools, or the repo's test host as evidenced.

For EACH acceptance criterion: PASS | FAIL | UNVERIFIABLE — criterion — what you did.
Persistence: batch writes, one restart, then confirm.
Never mark UNVERIFIABLE as PASS.
Never install new QA tooling; missing tool = UNVERIFIABLE.
Never log PAN.

Operator UI (Portal / ngx-translate): FAIL any AC that covers labels if the browser shows dotted keys (e.g. `transaction-risk-screening.risk-checks.country-blacklist.page-title`) instead of English. Check the host app (`localhost:1010` Portal), not only the library showcase. Sibling controls that already have Portal i18n (e.g. BIN Blacklist) are the control.

End with VERDICT: VERIFIED or VERDICT: NOT VERIFIED.
```

If the app cannot start, `NOT VERIFIED` counts as a failed review loop.

## Per-Task commit convention

One **Task** → one **commit** on the PBI branch. Never combine Tasks into one commit.

```text
Subject: AB#<TaskId> <imperative intent>
Body (optional): Parent PBI AB#<PBIId>. <why / test note>
```

Example sequence on `Edison/306969-build-309159`:

```text
AB#309160 Add CardBlacklistEntries DACPAC table
AB#309161 Add Card blacklist domain and application commands
AB#309162 Expose card-blacklist API routes and Reqnroll tests
```

Stage only files for the current Task. Push the PBI branch after each commit (or before review after the last Task).

**Rework:** new commits per affected Task — do not rewrite pushed Task history.

**Merge:** preserve commits (merge, not squash). PR description lists each Task id + SHA.

## Merge fallback (PowerShell)

Integration branch already checked out remotely as `origin/<integration>`:

```powershell
git fetch origin
git checkout <integration>
git merge --no-ff <pbi-branch> -m "Merge AB#<PBI> into <integration>"
git push origin <integration>
```

Never `push` to the inventory `defaultBranch`. Never `--bypass-policy`.

## Close sequence

```powershell
# Task
.\Set-WorkItemState.ps1 -Id <TaskId> -State "Done"
# PBI after all tasks Done and origin/integration contains the merge
.\Set-WorkItemState.ps1 -Id <PbiId> -State "Done"
.\Add-WorkItemComment.ps1 -Id <PbiId> - (comment file: branch + PR !id + per-Task commit SHAs + personal-skill close vs ADO-010)
```

Use the comment-file pattern from Tooling (no secrets in the command line).

## Invoke

User: `build-feature 306968`

Agent: this skill → Feature 306968 → drain specified children onto `Edison/306968-build` (area `Unity\Edison`).
