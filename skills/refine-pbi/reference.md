# refine-pbi reference

Load from SKILL.md when reading the PBI, drafting the Tasks, or creating them.

The Task descriptions are the implementation agent's technical plan for this PBI. They exist so that agent can fulfill the requirements and match the acceptance criteria. Do not post a Discussion comment, and do not also write `docs/plans/<PBI-id>.md`.

## Fields the summary printer drops

`Get-WorkItem.ps1` prints type, title, state, assignee, area, iteration, and the first 20 lines of Description with tags removed. It does not print Scope, acceptance criteria, story purpose, technical implementation details, assumptions, or parent.

Read those from the same verb the script uses:

```powershell
az boards work-item show --id <id> --org https://dev.azure.com/pxphq --output json
```

| Need | Field |
|---|---|
| Description | `System.Description` |
| Scope | `Custom.Scope` |
| Acceptance criteria | `Microsoft.VSTS.Common.AcceptanceCriteria` |
| Story purpose | `Custom.StoryPurpose` |
| Technical implementation details | `Custom.TechnicalImplementationDetails` |
| Assumptions | `Custom.Assumptions` |
| Area, iteration | `System.AreaPath`, `System.IterationPath` |
| Parent | `System.Parent` when present, otherwise the Hierarchy-Reverse relation |

If parent is absent from the payload, parent is `unknown`. Stop and ask which Feature owns the PBI. Do not update the work item to find out.

## Prior comments

Read-only. Older runs of this skill posted a Discussion comment; it is evidence of decisions already taken.

Re-read `ado/README.md` first. If it still has no list script, list Discussion comments with the `wit/comments` resource:

```powershell
az devops invoke --area wit --resource comments --route-parameters project=Unity workItemId=<id> --http-method GET --api-version 7.1-preview --org https://dev.azure.com/pxphq --output json
```

Use whichever array the payload actually has (`comments` is the documented one). A prior refine-pbi comment is one whose `text` contains `refine-pbi`; the latest by `createdDate` is the one to cite, by its `id`. Do not invent a comment deep link.

If this call fails, say so in the output and continue without prior comments. Do not claim there were none.

## Task descriptions

Task descriptions are HTML, passed to `New-Task.ps1 -Description`. Escape `&`, `<`, and `>` in titles and evidence quotes. No Markdown headings, no fences. Coding names only files, types, endpoints, and patterns found in source. A new type that does not exist yet is `unknown`, with the repo and the capability named. No PAN, secrets or tokens.

```powershell
$desc = @'
<p><b>Goal:</b> …</p>
'@
& '<Pxp.Unity.Agents.Tooling>/ado/New-Task.ps1' -Title '<imperative intent>' -ParentId <PBIId> `
    -AreaPath '<PBI area path>' -IterationPath '<PBI iteration path>' -Description $desc
```

### Implementation Task

Repeat the acceptance-criterion block once per criterion the slice covers. Pair each one with the Scope line or Reference requirement id it fulfills.

```html
<p><b>Goal:</b> {what this slice delivers, one or two sentences}. Parent PBI {id}: {title}.</p>
<p><b>Implement in:</b> {repo} ({repositoryType}). Primary agent: {from routing.md}. {POC branch from the Epic, or none.}</p>

<h3>Architecture</h3>
<ul>
  <li><b>Persist:</b> {owner}. {evidence}</li>
  <li><b>Enforce:</b> {owner}. {evidence}</li>
  <li><b>Operator UI:</b> {library and Portal, or none}. {evidence}</li>
  <li><b>Peers:</b> {logical names via IServiceRepository, or none}</li>
  <li><b>Data / PCI:</b> {boundary, or not in scope}</li>
  <li><b>Contracts:</b> {only names evidenced in source or an accepted ADR}</li>
  <li><b>Security review:</b> {not required | required before implementation, because …}</li>
</ul>

<h3>Acceptance criteria covered</h3>

<h4>AC {n} — {short name}</h4>
<p><b>Requirement:</b> {Scope line, or REQ id}.</p>
<p><b>Acceptance criterion:</b> GIVEN {context} WHEN {action} THEN {result}.</p>
<p><b>Coding:</b> Extend {evidenced file, type, endpoint, or pattern}. Implement {the behavior that makes the THEN true}.</p>
<p><b>Show the criterion:</b> {the test or check that proves the THEN}.</p>
<p><b>Blocked by:</b> {unknown, or None.}</p>

<h3>Boundaries</h3>
<p><b>Depends on:</b> {earlier slice by title, or None}.</p>
<p><b>Leave alone:</b> {sibling slice or sibling PBI scope}.</p>

<h3>Decisions taken</h3>
<ul>
  <li>{decision} — {citation: AN-nn, clarification default, ADR, architecture page, or file}</li>
</ul>

<h3>Unknowns</h3>
<ul>
  <li>{unknown} — {which acceptance criterion it affects, and the follow-up}</li>
</ul>

<h3>Evidence</h3>
<ul>
  <li>{Reference path, Policy path, or repo file} — {which criterion it settles}</li>
</ul>
```

Keep only the Architecture lines that constrain this slice. When Decisions taken or Unknowns is empty, one item: `None.`

A slice that adds ngx-translate keys in an Angular library includes, in that criterion's coding detail, copying those keys into Portal `en.json` (and sibling locales that already exist). The THEN is labels on the operator UI, not raw keys.

### Human QA Task

Title `Human QA: {PBI title}`.

```html
<p><b>Goal:</b> A person confirms by hand that PBI {id} meets every acceptance criterion. Not for an agent: build-pbi does not implement or close this Task.</p>
<p><b>When:</b> after the implementation Tasks are merged to the PBI branch and it is deployed to {environment, or unknown}.</p>
<p><b>Needs:</b> {environment, accounts or roles, test data}. No real card numbers; use {named test fixture or BIN prefix}.</p>
<ol>
  <li><b>AC 1 — {short name}.</b> GIVEN {context} WHEN {action} THEN {result}.<br>Steps: {what to do, where}. Expected: {what to see}.</li>
  <li><b>AC 2 — {short name}.</b> …</li>
</ol>
<p><b>Record:</b> PASS or FAIL per criterion as a comment on this Task; a FAIL becomes a Bug linked to PBI {id}.</p>
```

Every acceptance criterion on the PBI appears once in the list. Steps name only screens, endpoints and data that exist in source or in Reference; a step that cannot be evidenced is written `unknown` with what is missing.

## Review file

`<workspace root>/.edie/refine-pbi/<PBIId>/tasks-draft.md`, outside every repository clone:

```markdown
# refine-pbi · PBI {id} · {yyyy-mm-dd}

Verdict: {Ready | Near Ready | Needs Decision | Blocked}. {one sentence}
Existing Tasks: {ids and titles, or none}. Earlier refine-pbi comment: {#id, or none}.

## 1. {Task title}
Covers: AC {n, n}
{the HTML description, exactly as it will be created}

## {last}. Human QA: {PBI title}
{the HTML description}

## Held back
{slice — the blocked criterion and the follow-up; or None}
```

Create the Tasks from this file after the user's yes, so what is created is what was shown. If the user asks for changes, rewrite the file before asking again.

## Example

User: `refine-pbi 310618`

Load PBI 310618, its acceptance criteria, its Feature and Epic, sibling PBIs, child Tasks, and any earlier comment whose text contains `refine-pbi`. Search Reference for `310618` and the parent Feature id. Read the owning repo from `inventory/repositories.json` and the code that repo already has for this behavior. For each acceptance criterion, work out the architecture and the coding seam that make it true. Ask only if that would fork the implementation. Draft one Task per slice that no existing Task covers, each with its description, and the `Human QA:` Task if the PBI has none. Show the drafts and ask; create them on a yes. Post no comment. Leave the PBI form and its existing Tasks as they were.
