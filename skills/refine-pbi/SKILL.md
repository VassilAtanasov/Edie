---
name: refine-pbi
description: >-
  Prepares one existing PXP Unity Azure DevOps Product Backlog Item
  (https://dev.azure.com/pxphq/Unity) so an implementation agent can fulfill
  its requirements and match its acceptance criteria. Investigates
  Pxp.Unity.Agents.Reference, accepted Policy architecture, and the owning
  product code, then posts a human-readable HTML Discussion comment with the
  architecture and coding details for each acceptance criterion. Asks pickable
  questions only when a choice would change the implementation. Does not change
  PBI fields, state, or child Tasks. Use when the user says refine-pbi, refine
  a PBI, or names a PBI ID to implement from (for example PBI 310618).
---

# Refine PBI

The purpose of this skill is to give the **implementation agent** the technical architecture and coding details it needs to fulfill this PBI's requirements and match its acceptance criteria.

The only write is one HTML Discussion comment. The implementation agent reads that comment and does not need a second architecture pass. This does **not** create PBIs or Tasks, and a comment does not by itself give the PBI the child Tasks `build-pbi` needs.

Announce **Edie**, then adopt **architecture-agent** for the analysis (`Pxp.Unity.Agents.Policy/agents/architecture-agent.md`, routing from `orchestration/routing.md` at use time). Name the repo-type primary the implementer should adopt. The ADO write is comment-only.

## When to use

The user names a PBI ID (or a work-item URL containing it) and wants that PBI made implementable. Do **not** create a Feature, PBI, or Task. Do **not** open a PR. Do **not** change product code.

## Hard rules

- Write for the implementation agent. Every acceptance criterion gets the architecture and the coding detail that make its THEN true, or an explicit unknown that blocks it.
- Coding detail cites inspected files, types, endpoints, and patterns. Do not invent class names, routes, or migrations. Where a seam exists, name it. Where it does not, name the repo and the capability, and mark the new file or type `unknown`.
- Do not invent product or design facts. Unevidenced items are `unknown` plus a follow-up (`GLB-012`).
- **Comment only.** Do not change Title, Description, Scope, Story Purpose, Acceptance Criteria, Technical Implementation Details, Assumptions, state, assignment, area, iteration, or child Tasks.
- Decide from evidence when the answer is clear. Ask only when the unanswered choice would make the implementer build the wrong thing or miss an acceptance criterion. Do **not** ask for approval to post.
- The comment is **HTML**, using the template in [reference.md](reference.md). Not Markdown.
- A later run **adds a new comment** and cites the previous `refine-pbi` comment. Do not edit or delete the old one.
- Never log, echo, or store PAN / cardholder data (`GLB-040`).
- Do not conflate `TransactionsRiskScreening` with `RiskManagement`, or `SmartRouting` with `CardRouting`.
- Resolve `ado/*` script names from `Pxp.Unity.Agents.Tooling/ado/README.md` at use time. Prefer those scripts.
- If the PBI was discovered as someone else's item and the user did not name that ID, do not comment (`ADO-012`). Naming the PBI ID counts as naming it.
- **Portal i18n:** a library UI acceptance criterion that adds ngx-translate keys includes the Portal locale change in that criterion's coding detail (Bug 309212). Portal loads `Pxp.Unity.Portal.Web/src/assets/i18n/*.json` only. Do not file that slice as a Task.

## Inputs

Required: PBI ID or URL. If it is missing, or the work item is not a Product Backlog Item, stop and ask.

Load, in order:

1. The PBI requirements: title, description, Scope, acceptance criteria, story purpose, technical implementation details, assumptions, parent. `Get-WorkItem.ps1` prints a short summary and strips Description HTML. Read Scope, AC, and parent from `az boards work-item show` JSON when the summary omits them. See [reference.md](reference.md).
2. Parent **Feature**, then parent **Epic**, including POC branch constraints. Sibling PBIs under that Feature (do not implement their scope here).
3. Existing child Tasks (`Get-WorkItemChildren.ps1`). Point the coding detail at a Task that already covers a slice. Do not recreate Tasks.
4. Existing Discussion comments. Find earlier `refine-pbi` comments before drafting. If that read fails, stop. Do not claim there was no earlier comment.
5. `Pxp.Unity.Agents.Reference`: search the PBI id, the Feature id, and distinctive title words. Read the matching initiative's requirements, clarifications, `decisions.md`, architecture options, and ADR drafts, plus `domains/` when present.
6. Accepted Policy: `architecture/SERVICE_MAP.md`, `SYSTEM_CONTEXT.md`, `BIG_PICTURE.md`, `EVENT_FLOW.md`, accepted ADRs, `inventory/repositories.json`.
7. Source in the repos the inventory names. This is where coding details come from: the existing seam the implementer extends. No guessed APIs or file paths.

Authority when sources disagree: a bound Reference decision (answer id, locked or promoted decision) and an accepted Policy ADR outrank an unbound clarification. Inspected code outranks a draft. Record the conflict instead of silently picking.

## Workflow

```
refine-pbi:
- [ ] Load PBI requirements and AC, Feature, Epic, siblings, child Tasks, prior comments
- [ ] Investigate Reference, Policy architecture, and the code the implementer will change
- [ ] Map every AC to architecture + coding detail; ask only if that map would fork
- [ ] Post one HTML comment the implementation agent can execute
- [ ] Return PBI URL, comment id, verdict, decisions, unknowns
```

### Decide, or ask

**Decide and cite** when any of these settles it:

- Reference `decisions.md` binds an answer, or a decision is locked / promoted / accepted.
- An accepted Policy ADR or `architecture/*` page states it.
- Inspected source implements one behavior and the PBI does not contradict it.
- A Reference clarification recommends one default and neither Policy nor source contradicts it. Cite it as that default, not as a locked decision.
- The PBI, Feature, or Epic text states it and nothing above contradicts it.

**Ask** (`AskQuestion`, recommended option first) only when the implementer would otherwise miss an acceptance criterion or build the wrong behavior:

- Two sources disagree on what an acceptance criterion requires, or on the owning repo or boundary.
- A product or design choice has no evidenced default, and the coding detail would have to pick it (which repo persists, which existing endpoint or check to extend, which channel is in scope).
- PCI, tokenization, or inter-service auth is in scope and the boundary is not in Reference or code. Include "leave unknown" as a choice. Do not invent the boundary.

A missing class name for code that does not exist yet is **not** a question. Name the repo and the capability, and put the new type under Unknowns.

One question round. After the picks, fold them into the acceptance-criterion coding detail and post in the same turn. Do not ask whether to post.

Verdict `Ready` means every acceptance criterion has enough architecture and coding detail to implement and to show the THEN. `Near Ready` means a leftover unknown does not change that approach. `Needs Decision` or `Blocked` means at least one acceptance criterion cannot be met accurately until the named follow-up is resolved.

### Post the comment

Build the HTML template in [reference.md](reference.md). Lead with the implementation map: each acceptance criterion, the requirement it fulfills, the architecture that constrains it, and the coding detail that satisfies it. Shared architecture is stated once. Proposed task slices stay text inside the comment.

Header is exactly an `h2` whose text starts with `refine-pbi`, then the PBI id and the ISO date. Point at the latest earlier `refine-pbi` comment id and its `url` from the comments payload. If there is none, write `none`.

Post with `Add-WorkItemComment.ps1` and a PowerShell here-string. The script accepts simple HTML and does not take a Markdown format flag. Do not send Markdown. Do not open a second comments client.

```powershell
$html = @'
<h2>refine-pbi · PBI {id} · {yyyy-mm-dd}</h2>
'@
& '<Pxp.Unity.Agents.Tooling>/ado/Add-WorkItemComment.ps1' -Id 310618 -Comment $html
```

Use the real checkout path and the real HTML. The script prints the new comment id.

## Output to the user

- PBI id and URL (`https://dev.azure.com/pxphq/Unity/_workitems/edit/<id>`).
- New comment id, and the previous `refine-pbi` comment id when there was one.
- Verdict: whether the implementation agent can meet every acceptance criterion from the comment.
- Decisions applied, and any acceptance criterion still blocked.
- Explicitly: fields, state, and child Tasks were left unchanged.

## Additional resources

- HTML template, field read, comment list: [reference.md](reference.md)
