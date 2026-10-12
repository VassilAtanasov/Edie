---
name: refine-pbi
description: >-
  Prepares one existing PXP Unity Azure DevOps Product Backlog Item
  (https://dev.azure.com/pxphq/Unity) so an implementation agent can fulfill
  its requirements and match its acceptance criteria. Investigates
  Pxp.Unity.Agents.Reference, accepted Policy architecture, and the owning
  product code, then files the PBI's child Tasks: one per implementation slice,
  each with a description that carries the architecture and coding detail for
  the acceptance criteria it covers, plus at least one human QA Task. Shows the
  Tasks for review and creates them only on the user's yes. Asks pickable
  questions only when a choice would change the implementation. Posts no
  Discussion comment. Does not change PBI fields or state, or existing Tasks.
  Use when the user says refine-pbi, refine a PBI, or names a PBI ID to
  implement from (for example PBI 310618).
---

# Refine PBI

The purpose of this skill is to give the **implementation agent** the technical architecture and coding details it needs to fulfill this PBI's requirements and match its acceptance criteria.

The only write is the PBI's new child Tasks. Each Task's description is the implementation agent's plan for that slice, so it does not need a second architecture pass; one more Task tells a person what to check by hand. This skill posts **no** Discussion comment and does **not** create PBIs.

Announce **Edie**, then adopt **architecture-agent** for the analysis (`Pxp.Unity.Agents.Policy/agents/architecture-agent.md`, routing from `orchestration/routing.md` at use time). Name the repo-type primary the implementer should adopt.

## When to use

The user names a PBI ID (or a work-item URL containing it) and wants that PBI made implementable. Do **not** create a Feature or PBI. Do **not** open a PR. Do **not** change product code.

## Hard rules

- Write for the implementation agent. Every acceptance criterion gets the architecture and the coding detail that make its THEN true, in the Task that covers it, or an explicit unknown that blocks it.
- Coding detail cites inspected files, types, endpoints, and patterns. Do not invent class names, routes, or migrations. Where a seam exists, name it. Where it does not, name the repo and the capability, and mark the new file or type `unknown`.
- Do not invent product or design facts. Unevidenced items are `unknown` plus a follow-up (`GLB-012`).
- **New Tasks only.** Do not post a Discussion comment. Do not change the PBI's Title, Description, Scope, Story Purpose, Acceptance Criteria, Technical Implementation Details, Assumptions, state, assignment, area, or iteration. Do not edit, re-parent, re-state or delete an existing Task.
- **Every Task this skill creates has a Description.** Never call `New-Task.ps1` without `-Description`. The description is HTML from the templates in [reference.md](reference.md#task-descriptions) and is enough to implement the slice without any other document.
- **At least one human QA Task.** After this skill runs, the PBI has a Task whose title starts with `Human QA:`, listing every acceptance criterion as a check a person runs by hand. If one already exists, do not create another. An agent never implements or closes it.
- **Tasks are created only after the user reviews them.** Show the full list of Tasks to be created and wait for an explicit yes. No Task is created before that answer, and a yes covers only the list that was shown.
- **Freshness gate `[GLB-071]`** before the first write: run `Test-PolicySnapshotFreshness.ps1`; only `InSync` proceeds. On a stale verdict, stop and ask the user to regenerate the workspace.
- Decide from evidence when the answer is clear. Ask only when the unanswered choice would make the implementer build the wrong thing or miss an acceptance criterion.
- Never log, echo, or store PAN / cardholder data (`GLB-040`).
- Do not conflate `TransactionsRiskScreening` with `RiskManagement`, or `SmartRouting` with `CardRouting`.
- Resolve `ado/*` script names from `Pxp.Unity.Agents.Tooling/ado/README.md` at use time. Prefer those scripts.
- If the PBI was discovered as someone else's item and the user did not name that ID, do not file Tasks (`ADO-012`). Naming the PBI ID counts as naming it.
- **Portal i18n:** a library UI acceptance criterion that adds ngx-translate keys includes the Portal locale change in that criterion's coding detail (Bug 309212). Portal loads `Pxp.Unity.Portal.Web/src/assets/i18n/*.json` only. Do not file that slice as a separate Task; it belongs to the Task that adds the keys.

## Inputs

Required: PBI ID or URL. If it is missing, or the work item is not a Product Backlog Item, stop and ask.

Load, in order:

1. The PBI requirements: title, description, Scope, acceptance criteria, story purpose, technical implementation details, assumptions, parent, area and iteration. `Get-WorkItem.ps1` prints a short summary and strips Description HTML. Read Scope, AC, and parent from `az boards work-item show` JSON when the summary omits them. See [reference.md](reference.md).
2. Parent **Feature**, then parent **Epic**, including POC branch constraints. Sibling PBIs under that Feature (do not implement their scope here).
3. Existing child Tasks (`Get-WorkItemChildren.ps1`), with their descriptions. A slice an existing Task covers is not filed again. Note whether a `Human QA:` Task exists.
4. Existing Discussion comments, read-only. An earlier `refine-pbi` comment (older runs posted one) is evidence of decisions already taken; cite it, do not edit it. If that read fails, say so and continue without it.
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
- [ ] Split the map into slices in build order; draft one Task per unfiled slice, and one Human QA Task
- [ ] Show the Tasks for review; create them only on the user's yes
- [ ] Return PBI URL, Task ids, verdict, decisions, unknowns
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

One question round. After the picks, fold them into the coding detail and go on to the Task drafts in the same turn.

Verdict `Ready` means every acceptance criterion has enough architecture and coding detail to implement and to show the THEN. `Near Ready` means a leftover unknown does not change that approach. `Needs Decision` or `Blocked` means at least one acceptance criterion cannot be met accurately until the named follow-up is resolved.

### Draft the Tasks

1. **Slices.** Split the work into slices in build order. Every acceptance criterion is covered by at least one slice. Drop every slice an existing child Task already covers. When the verdict is `Needs Decision` or `Blocked`, hold back every slice that depends on a blocked acceptance criterion, and say which and why.
2. **Implementation Task.** Title is the slice's imperative intent, without the PBI id. The description, from the implementation template, carries: the goal, the repo and primary agent, the architecture that constrains the slice, and for each acceptance criterion it covers the criterion text, the coding detail with its evidenced seam, and the test or check that shows the THEN. Then what it depends on, what it leaves alone, the decisions taken with their citations, and the unknowns. Architecture shared by several slices is repeated in each, briefly: a Task is read on its own.
3. **Human QA Task.** If no child Task's title starts with `Human QA:`, draft one: `Human QA: <PBI title>`. Its description, from the QA template, lists **every** acceptance criterion as a numbered check with the steps and the expected result, and names the environment and the test data needed. Test data never includes a PAN; card data is a BIN prefix or a named test card from the repo's own fixtures. One Task covers all criteria; draft a second only when the criteria are checked in two separate operator UIs or environments.
4. **Review file.** Write the drafts to the review file ([reference.md](reference.md#review-file)): verdict, then each Task in creation order with its title and full description, the human QA Task last, then held-back slices.

### Review, then create

Show in chat the verdict, a table (number, title, acceptance criteria covered) and the review file link, then ask once (`AskQuestion`): create these Tasks, or change the list first. Wait.

- **Yes:** create exactly the drafts shown.
- **Changes asked for:** apply them, show the changed list, and ask again. The human QA Task and the description on every Task are not optional; if the user asks to drop either, say so and keep it.
- **No:** create nothing. Report that no Tasks were filed and where the drafts are.

Creating follows the `create-task` procedure (`Pxp.Unity.Agents.Policy/orchestration/procedures/create-task.md`) with `New-Task.ps1`; confirm its parameters in `ado/README.md`.

1. Freshness gate.
2. Create the implementation Tasks in build order, one call at a time, so their ids ascend in that order: `build-pbi` builds in Task id order. Create the human QA Task last.
3. Each call: `-Title`, `-ParentId <PBIId>`, `-AreaPath` and `-IterationPath` copied from the PBI, `-Description` taken from the review file.
4. Leave the Tasks unassigned in their initial state. Do not set state or `AssignedTo`.
5. Confirm with `Get-WorkItemChildren.ps1 -Id <PBIId>` that each new Task is under the PBI. If a create fails, stop; report the Tasks already created and the slices left. Do not retry with a hand-rolled call.

A later run files only slices that no existing Task covers, and never a second `Human QA:` Task. It does not rewrite the description of a Task an earlier run created; if the new analysis changes a slice already filed, say so in the output and leave the Task as it is.

## Output to the user

- PBI id and URL (`https://dev.azure.com/pxphq/Unity/_workitems/edit/<id>`).
- Verdict: whether the implementation agent can meet every acceptance criterion from the Tasks.
- Tasks created: id, title, and the acceptance criteria each covers; the `Human QA:` Task id (created now, or the one that already existed).
- Slices held back and why. Decisions applied, and any acceptance criterion still blocked.
- Existing Tasks with an empty description, if any: listed, not edited.
- If the user declined: no Tasks were filed, and the review file's path.
- Explicitly: no comment was posted; PBI fields and state, and existing Tasks, were left unchanged.

## Additional resources

- Task description templates, review file, field read, comment list: [reference.md](reference.md)
