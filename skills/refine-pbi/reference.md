# refine-pbi reference

Load from SKILL.md when reading the PBI, listing comments, or posting.

The comment is the implementation agent's technical plan for this PBI. It exists so that agent can fulfill the requirements and match the acceptance criteria. Do not also write `docs/plans/<PBI-id>.md`.

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
| Parent | `System.Parent` when present, otherwise the Hierarchy-Reverse relation |

If parent is absent from the payload, parent is `unknown`. Stop and ask which Feature owns the PBI. Do not update the work item to find out.

## Prior comments

Re-read `ado/README.md` first. If it still has no list script, list Discussion comments with the same `wit/comments` resource `Add-WorkItemComment.ps1` posts to:

```powershell
az devops invoke --area wit --resource comments --route-parameters project=Unity workItemId=<id> --http-method GET --api-version 7.1-preview --org https://dev.azure.com/pxphq --output json
```

Use whichever array the payload actually has (`comments` is the documented one). A prior refine-pbi comment is one whose `text` contains `refine-pbi`. The latest by `createdDate` is the one to cite. Copy its `id` and `url`. Do not invent a comment deep link.

If this call fails, stop and report the error. Do not post.

## HTML comment

ADO Discussion renders this as HTML. Escape `&`, `<`, and `>` in titles and evidence quotes. No Markdown headings, no fences.

Verdict is one of: `Ready`, `Near Ready`, `Needs Decision`, `Blocked`.

Repeat the acceptance-criterion block once per criterion on the PBI. Pair each one with the Scope line or Reference requirement id it fulfills. Coding names only files, types, endpoints, and patterns found in source. A new type that does not exist yet is `unknown`, with the repo and the capability named.

```html
<h2>refine-pbi · PBI {id} · {yyyy-mm-dd}</h2>
<p><b>For the implementation agent.</b> Use this comment to fulfill the requirements and match the acceptance criteria. {title}</p>
<p><b>Verdict:</b> {Ready | Near Ready | Needs Decision | Blocked}. {Whether every acceptance criterion can be implemented from this comment.}</p>
<p><b>Implement in:</b> {repo} ({repositoryType}). Primary agent: {from routing.md}. {POC branch from the Epic, or none.}</p>
<p><b>Previous refine-pbi comment:</b> none | #{id} ({yyyy-mm-dd}, {url from the payload}). Read this comment. The earlier comment is unchanged.</p>
<p>Title, Description, Scope, acceptance criteria, technical details, state, and child Tasks were not changed.</p>

<h3>Shared architecture</h3>
<ul>
  <li><b>Persist:</b> {owner}. {evidence}</li>
  <li><b>Enforce:</b> {owner}. {evidence}</li>
  <li><b>Operator UI:</b> {library and Portal, or none}. {evidence}</li>
  <li><b>Peers:</b> {logical names via IServiceRepository, or none}</li>
  <li><b>Data / PCI:</b> {boundary, or not in scope}</li>
  <li><b>Contracts:</b> {only names evidenced in source or an accepted ADR}</li>
  <li><b>Security review:</b> {not required | required before implementation, because …}</li>
</ul>

<h3>Acceptance criteria</h3>

<h4>AC 1 — {short name}</h4>
<p><b>Requirement:</b> {Scope line, or REQ id}. {Matches the PBI field. | PBI field differs: {what}. Fields were left as written.}</p>
<p><b>Acceptance criterion:</b> GIVEN {context} WHEN {action} THEN {result}.</p>
<p><b>Architecture:</b> {the boundary, data, and peers this criterion depends on}. {citation}</p>
<p><b>Coding:</b> Extend {evidenced file, type, endpoint, or pattern} in {repo}. Implement {the behavior that makes the THEN true}. Leave {sibling scope} unchanged. {Already filed as Task {id}. | Not filed.}</p>
<p><b>Show the criterion:</b> {the test or check that proves the THEN}.</p>
<p><b>Blocked by:</b> {unknown, or None.}</p>

<h3>Evidence</h3>
<ul>
  <li>{Reference path, Policy path, or repo file} — {which criterion it settles}</li>
</ul>

<h3>Decisions taken</h3>
<ul>
  <li>{decision} — {citation: AN-nn, clarification default, ADR, architecture page, or file}</li>
</ul>

<h3>Unknowns</h3>
<ul>
  <li>{unknown} — {which acceptance criterion it blocks, and the follow-up}</li>
</ul>
```

When Decisions taken or Unknowns is empty, one item: `None.`

An acceptance criterion that adds ngx-translate keys in an Angular library includes, in that criterion's coding detail, copying those keys into Portal `en.json` (and sibling locales that already exist). The THEN is labels on the operator UI, not raw keys.

## Example

User: `refine-pbi 310618`

Load PBI 310618, its acceptance criteria, its Feature and Epic, sibling PBIs, child Tasks, and any comment whose text contains `refine-pbi`. Search Reference for `310618` and the parent Feature id. Read the owning repo from `inventory/repositories.json` and the code that repo already has for this behavior. For each acceptance criterion, write the architecture and the coding seam that make it true. Ask only if that would fork the implementation. Post the HTML comment. Leave the PBI form and its Tasks as they were.
