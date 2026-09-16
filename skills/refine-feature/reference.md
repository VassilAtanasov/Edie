# refine-feature reference

Load from SKILL.md only when drafting fields or the architecture pack.

## Architecture pack (Gate A)

Use this shape. Cite Feature / Epic / `architecture/*` / source. Mark `unknown` instead of guessing.

```markdown
# Architecture — Feature #<id> <title>

## Reconciliation
- Confirmed:
- Changed (Feature or Epic vs evidence):
- Gaps:
- Conflicts:
- Assumptions:
- Open questions:

## Scope vs Epic
- Verdict: in-POC | follow-on under this Epic | wrong parent
- Recommended pick: …

## Bounded contexts and repos
| Concern | Owner (service/repo) | Evidence |
|---|---|---|
| Persist | | |
| Enforce (pay-in / other) | | |
| Operator UI | | library templates + **Portal i18n host** |
| Identity / token / PCI | | |

## Peer calls
Logical names via IServiceRepository. No host:port.

## Contracts and APIs
Only what source or an accepted ADR shows. Capability grain.

## Security
- PCI / secrets / class-1: yes/no + why
- security-review-agent: required before implementation? yes/no

## Picks for the user (recommended first)
1. …
```

## PBI fields (Unity / PXP Scrum - Unity)

Re-query type metadata if create fails; do not treat this table as forever-true.

| UI label | Reference | This skill |
|---|---|---|
| Title | `System.Title` | Required |
| Description | `System.Description` | Required HTML user story; not Scope |
| Scope | `Custom.Scope` | Required; discrete bullets |
| Story Purpose | `Custom.StoryPurpose` | Required; 1–2 sentences |
| Acceptance Criteria | `Microsoft.VSTS.Common.AcceptanceCriteria` | Required; covers every Scope line |
| Assumptions | `Custom.Assumptions` | Optional; genuine only |
| Technical Implementation Details | `Custom.TechnicalImplementationDetails` | **Populate** (skill-directed) |
| Already Deployed to Production | `Custom.AlreadyDeployedtoProduction` | false unless evidenced |
| Priority / Value Area | process defaults | Usually leave default |

`create-pbi` tooling: `New-ProductBacklogItem.ps1` with `-FeatureId`. Plain `-AcceptanceCriteria` / `-Assumptions` lines auto-wrap to `<div><span>[ ]</span><span> … </span> </div>`. Description must be HTML (`<p>`, `<ul>`, `<br>`), not Markdown.

Do not paste Scope into Description. AC is derived from Scope; every Scope line maps to ≥1 AC line before create.

**Portal-hosted library UI:** add a Scope line such as “Portal English (and existing sibling locales) include the new `transaction-risk-screening…` keys” and an AC that the Internal Risk Screening surface shows labels, not dotted keys. Copy source of truth is the library showcase `en.json`, not the package dist.

## Task fields

`New-Task.ps1 -ParentId <pbiId> -Title …` optional `-Description` (HTML), `-AreaPath`, `-IterationPath`. One implementable slice per Task. Evidence from the parent PBI’s TechnicalImplementationDetails + source.

## Invoke example

User:

```text
refine-feature 306969
```

Agent: load Feature 306969, Epic 306966, sibling Features, architecture/*, Gate A (PCI card identity, Epic BIN-only vs Card Blacklist follow-on), stop for picks, then Gate B spec pack, then create.

## Do not

- Emit Tasks during a PRD story-map-only request (use this skill instead when the user wants PBIs **and** Tasks).
- Create a new Feature when the user already gave a Feature ID.
- Target default/release branches when the Epic names a POC branch.
- Store PAN in Portal or in work-item text.
