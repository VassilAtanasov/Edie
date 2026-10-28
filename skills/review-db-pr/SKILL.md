---
name: review-db-pr
description: >-
  Reviews the database part of one PXP Unity Azure DevOps pull request
  (https://dev.azure.com/pxphq/Unity): DACPAC .sql/.sqlproj scripts in a
  service's .Db project or Pxp.Unity.Db, EF Core model configuration and
  queries, and Dapper repositories. Checks the change against the Unity data
  rules, the DB Scripts Reviewers' table rules, DACPAC deployment safety, T-SQL
  quality and EF Core / Dapper query anti-patterns, shows evidence-cited
  findings, posts one PR thread per finding on the user's yes, and offers a fix
  branch into the PR's source branch on the user's yes. Use when the user says
  review-db-pr, review the DB changes in PR <id>, or wants .sql changes checked
  before pushing (for example review-db-pr 114757).
---

# Review DB PR

Edie review of the **database part** of one PR, so the `DB Scripts Reviewers` gate and the class-3 data gate are not where these problems are first met. **Not** a Unity catalog role, and **not** a replacement for the human `DB Scripts Reviewers` approval, `security-review-agent`, or the ADR-0004 class-3 gate (`[DATA-003]`).

Announce before acting (`[GLB-070]`): Edie, session **review-db-pr**, operating as the lead reviewer resolved from `Pxp.Unity.Agents.Policy/orchestration/routing.json` for the PR's repo type (for example `backend-quality-agent` for `backend-service`), with `data-engineer-agent` added when the diff touches `Pxp.Unity.Db` or cross-domain data, and `security-review-agent` when it touches encrypted or card columns. Resolve at run time; never cache routing here.

## When to use

- The user names a PR to review for DB changes (`review-db-pr 114757`).
- Before the first push of a branch with `.sql` changes (`review-db-pr --local`).
- build-pbi step 7 runs this checklist as its DB reviewer when a task diff has DB files ([Inside build-pbi](#inside-build-pbi)).

Not for: designing the schema (refine-pbi), tuning a live database, or stores other than SQL Server.

## Hard rules

- **Findings first, writes on a yes.** Show the findings table in chat. Post PR threads only after the user says yes to posting, and open a fix branch only after a separate yes to fixing. Each yes covers that PR and that round only.
- **Evidence or it is a note.** Every finding names `file:line` in the diff and a rule id from [reference.md](reference.md) (`DATA-*`, `DBR-*`, `SQL-*`, `EF-*`, `DAP-*`, `CON-*`) or a Policy citation. A finding without both is a `note`.
- **Evidence gating.** A missing `HasIndex` does not prove a missing index: read the `.Db` scripts. A load-mutate loop with one `SaveChanges` is not N+1. `AsNoTracking` matters only for tracked entity results. If nothing is wrong, say so; do not pad.
- **An existing table is not a licence** (`DBR-001`). Flag the new code even when an older table has the same shape; name the older table as existing drift in a note.
- **Unknown stays unknown** (`[GLB-012]`, `[DATA-005]`). Where a convention is not in reference.md, read the repo's existing `.Db` tables first; ask the user only if they do not settle it.
- **Never echo PAN, secrets, keys or connection strings** found in the diff (`[GLB-040]`). Cite `file:line` only; route to `security-review-agent`.
- **Other people's threads.** Never reply on, resolve or close a thread this skill did not open. Never reply on `svc_unity_sonar` or another bot's thread.
- **Never deploy** a DACPAC to a shared environment. A scratch local catalog is allowed only on the user's yes ([Verify](#4-verify-optional)).
- **Never merge**, arm auto-complete, or push to a default, release or environment branch (`[GLB-003]`, `[GLB-024]`).
- Resolve `ado/*` script names from `Pxp.Unity.Agents.Tooling/ado/README.md` at time of use.

## Inputs

| Input | Use |
|---|---|
| PR id or URL | `Get-PullRequest.ps1 -Id <id> *>&1 \| Out-String` → repo, source, target, head SHA, status |
| `--local` | working tree and branch diff against `origin/<defaultBranch>` from `inventory/repositories.json` |
| `--round <n>` | re-review after fixes; read the earlier report and check its findings plus the new commits |
| optional PBI/Task id | intent and AC, via `Get-WorkItem.ps1` |

If the PR is not `active`, stop and say so.

## Workflow

Copy and track:

```
review-db-pr <PR>:
- [ ] Resolve repo, type, default branch, roles; announce
- [ ] Select the DB file set (stop if empty)
- [ ] Read the context around each changed object
- [ ] Check A–F from reference.md
- [ ] (optional, on yes) Build + scratch-catalog verify
- [ ] Report file + findings table in chat
- [ ] (on yes) Post one thread per finding
- [ ] (on yes) Fix branch → PR into the source branch; re-review the delta
```

### 1. Resolve

Repo, repo type and `defaultBranch` from `Pxp.Unity.Agents.Policy/inventory/repositories.json` (`[GLB-002]`). Lead reviewer, change classes and overrides from `routing.json` (RT-101..105). Class 3 applies when the diff touches `Pxp.Unity.Db` or cross-domain schema/ETL. State the role and its basis.

Run folder: `.edie/review-db-pr/<PR>/` at the Unity workspace root, beside build-pbi's run folders (`--local`: `.edie/review-db-pr/local-<branch>/`). Never commit it.

### 2. Select the DB file set

From `git diff --name-status origin/<target>...origin/<source>` (`--local`: `origin/<defaultBranch>...HEAD` plus uncommitted changes):

- **Schema:** `**/*.Db/**`, `**/*.sql`, `**/*.sqlproj`, `*.refactorlog`, publish profiles, `**/Migrations/**`.
- **EF Core:** `DbContext` types, `IEntityTypeConfiguration<>`, `OnModelCreating`, LINQ over `DbSet<>`, `FromSql*`/`ExecuteSql*`/`ExecuteUpdate`/`ExecuteDelete`.
- **Dapper/ADO:** `Query*`/`Execute*` calls, `CommandDefinition`, SQL string literals, embedded `.sql` resources.
- **Tests:** script-level tests (for example TRS `DbProjectScripts`), SQLite or in-memory schema copies.

Empty set: report "no DB changes" and stop.

### 3. Read the context, then check

Read each new or changed table script in full, the parent tables its FKs reference, the EF configuration and Dapper SQL for the same tables, and the repo's script tests. Then apply [reference.md](reference.md) in order:

- **A. Unity rules:** `DATA-001..006`, `DBR-001..004`.
- **B. DACPAC deployment safety:** `SQL-D*`.
- **C. T-SQL quality and security:** `SQL-Q*`, `SQL-S*`.
- **D. EF Core:** `EF-*`, only when EF is in the set.
- **E. Dapper:** `DAP-*`, only when Dapper is in the set.
- **F. Consistency and tests:** `CON-*`.

Apply every `DBR` rule to **every** table and column in the change; the human reviewer flags one example per rule.

### 4. Verify (optional)

Ask once: "Build the `.sqlproj` and deploy to a scratch local catalog to check FK names and index coverage?" On yes: build, deploy to a new local catalog named `SCRATCH_<Repo>_<PR>`, run the [verify query](reference.md#verify-query), then drop the catalog. Known local pitfalls are in the user's memory (`local-trs-db-branch-era-rows`). On no or failure, record "not verified at runtime" in the report.

### 5. Report

Write `<run folder>/report-r<round>.md` in the [report format](reference.md#report-format), then show the user the verdict line and the findings table. `VERDICT: PASS | REWORK | STOP-CLASS-3`. `STOP-CLASS-3` when a `Pxp.Unity.Db` or cross-domain change has no named per-domain owner or class-3 approver in the PR description.

Then ask, and wait:

> Post these <n> findings to PR !<id> as one thread each? (yes / only must-fix / no)

### 6. Post (on yes)

One thread per finding with `New-PullRequestThread.ps1 -Repo <repo> -PullRequestId <id> -Comment <text>` in the [thread format](reference.md#thread-format), plus one summary thread that starts `review_record:` (verdict, head SHA, counts, under 1,500 characters). Record each thread id in the report. The sanctioned script opens PR-level threads; it has no file or line anchor today, so the first line of each thread names `file:line`. Do not hand-roll `az rest` calls to anchor them.

On a later round, close the threads this skill opened whose finding is fixed (`Set-PullRequestThreadStatus.ps1 -Status fixed`), and leave the rest active.

### 7. Fix (on a separate yes)

Ask, and wait:

> Open a fix branch for the <n> must-fix (and <m> should-fix) findings, with a PR into `<source branch>`?

On yes:

1. `Get-PullRequest.ps1 -Id <id> *>&1 | Out-String`: if the PR is no longer `active`, stop; offer a new small PR against the default branch instead.
2. `git fetch origin`; create `<source branch>-dbfix-<n>` from `origin/<source branch>`.
3. Dispatch the resolved **primary** (for example `backend-software-agent`) as a fresh subagent with the report path. It fixes only the listed findings, updates EF configuration, Dapper SQL and test schemas to match (`CON-1`), adds or extends script tests (`CON-2`), and builds and runs the repo's tests. The reviewing session does not write the fix.
4. Commit with subject `AB#<id> Fix DB review findings: <short list>`, using the work-item id the source PR's commits use, and push with `git push -u origin <fix branch>`. If the push output says `[new branch]` for a branch that should already exist, stop.
5. `New-PullRequest.ps1 -Repo <repo> -Source <fix branch> -Target <source branch> -Title "DB review fixes for PR !<id>" -Description <file>` (under 4,000 characters: findings fixed, rule ids, test result).
6. Re-review the fix PR from step 2 with `--round <n+1>` in a fresh subagent, and report both PR links. Merging the fix PR is the user's, or `[GLB-024]` on their explicit direction for that PR.

## Inside build-pbi

When build-pbi step 7 finds DB files in the task diff, it launches one more fresh reviewer with the prompt in build-pbi's reference.md (DB-review prompt). That reviewer applies this skill's [reference.md](reference.md) A–F to the task PR diff, does not ask the user, posts nothing but its own `review_record:` thread, and hands back the build-pbi reviewer block. Fixes go through build-pbi's rework loop, not step 7 here.

## Stop conditions

- Findings shown and the user has answered the post and fix questions, or declined them.
- The PR is not active, the DB file set is empty, or `STOP-CLASS-3` (report the missing owner or gate).
- Two `MANDATORY` rules conflict, or an `unknown` convention decides the verdict: ask the user.
