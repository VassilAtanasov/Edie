---
name: refine-feature
description: >-
  Refines an existing Azure DevOps Feature into implementation-ready Product
  Backlog Items and Tasks. Performs technical architecture analysis against the
  parent Epic and architecture/source evidence, then specifies and creates all
  PBIs (ADO-001) and child Tasks (ADO-002) with Scope, AC, and technical
  implementation detail. Use when the user says refine-feature, refine a
  Feature, specify all PBIs of a Feature, break down a Feature for
  implementation, or names a Feature ID to fully specify (for example Feature
  306969).
---

# Refine Feature

Team refinement for an **existing Feature**: architecture analysis, then PBIs, then Tasks. This is **not** `prd-story-mapping-agent` (ADR-0011 stops at stories and forbids tasks). It **sequences** `architecture-agent` analysis with `create-pbi` and `create-task`.

State the role before proceeding: **architecture-agent** for Gate A; **ADO authoring** (`ADO-001`/`ADO-002`) for Gates B–C.

## When to use

User names a Feature ID and wants every PBI and Task specified for implementation. Do **not** create a Feature or Epic. Do **not** open a PR. Do **not** move items to `Committed` / `In Progress` unless the user directs that separately.

## Hard rules

- Do not invent product, design, or implementation facts. Unevidenced items are `unknown` + follow-up (`GLB-012`).
- Stop before any ADO **write**. Ask pickable questions; **recommended option first**. After answers, rewrite the pack from those choices.
- Description and AC are **HTML** in ADO work-item fields, not Markdown. Pass plain AC/Scope lines to sanctioned scripts (they wrap checkbox HTML).
- Never log, echo, or store PAN / cardholder data (`GLB-040`). PCI / tokenization / inter-service auth → flag `security-review-agent` + ADR-0004 class 1; do not treat those stories as `Ready` until decided.
- Do not conflate `TransactionsRiskScreening` (real-time) with `RiskManagement` (analytics), or `SmartRouting` with `CardRouting`.
- Resolve current `ado/*` script names from `Pxp.Unity.Agents.Tooling/ado/README.md` at time of use. Prefer those scripts; in a qualifying ADR-0023 §1a MCP session, Azure DevOps MCP is allowed for work-item create/update (`ADR-0025`).
- Parent every PBI to the **given Feature ID**. Parent every Task to its PBI. No orphans.
- Skip create for any story still `Needs Decision` / `Blocked`.
- After discovering a work item is another team's `AreaPath`, do not keep writing it unless the user named that ID (`ADO-012`).
- **Host-app i18n (Portal + ngx-translate libraries):** Angular libraries (`@pxp/*`) do **not** ship translations into Portal. Portal loads `Pxp.Unity.Portal.Web/src/assets/i18n/*.json` only. A Feature that adds operator UI in a frontend-package **must** include a Portal Scope/AC (or a dedicated Portal Task) to copy every new key from the library showcase `en.json` into Portal locales. Missing this produced Bug 309212 (Card Blacklist keys) and the same symptom for Country Blacklist (`transaction-risk-screening.risk-checks.country-blacklist.*`). Showcase-only strings are not enough.

## Inputs

Required: Feature work-item ID (or a URL containing it).

Load, in order:

1. The Feature (title, description, area, iteration, parent).
2. The parent **Epic** (in/out of scope, POC/branch constraints, success criteria).
3. Sibling Features under that Epic (do not duplicate their scope).
4. Existing children of this Feature (reuse; do not recreate covered PBIs).
5. Unity reference materials when the workspace has them: `Pxp.Unity.Agents.Policy/architecture/SERVICE_MAP.md`, `SYSTEM_CONTEXT.md`, `BIG_PICTURE.md`, `EVENT_FLOW.md`, accepted ADRs, `inventory/repositories.json`.
6. Inspected source in the owning repos the Epic/Feature name — not guessed APIs.

If the Feature ID is missing or is not type Feature, stop and ask.

## Workflow

Copy and track:

```
refine-feature:
- [ ] Load Feature, Epic, siblings, existing children, architecture/source
- [ ] Gate A: architecture analysis — STOP for picks
- [ ] Draft story map + PBI fields + Tasks (no ADO writes)
- [ ] Gate B: present suggested PBIs clearly, then STOP for spec-pack picks
- [ ] Create approved PBIs (create-pbi / ADO-001)
- [ ] Create approved Tasks (create-task / ADO-002)
- [ ] Return IDs, URLs, Scope→AC matrix, leftover unknowns
```

### Gate A — architecture (no ADO writes)

Produce the analysis in [reference.md](reference.md) § Architecture pack. Reconcile Feature vs Epic vs architecture vs source explicitly: confirmed / changed / gap / conflict / assumption / open question. Never silently overwrite the Feature or Epic.

Must cover:

- Scope verdict vs Epic (in-POC / follow-on / wrong parent).
- Owning bounded context and repos; who persists, who enforces, who presents UI; if UI is a library hosted by Portal, name **both** the library repo **and** Portal i18n as distinct deliverables.
- Peer calls via `IServiceRepository` (no hardcoded URLs).
- Data ownership (`ARCH-004`); token/PCI boundary if cards or PII.
- Contracts/events/APIs **evidenced in source**; capability grain only — no invented class names, file paths, or migrations.
- Inheritance / owner scope if the Epic uses merchant group / merchant / site.
- Security-review flags.

Then **stop**. Pickable questions for every unresolved product/design choice (recommended first). Do not continue until the user picks.

### Draft the spec pack (still no writes)

From Gate A picks, draft **vertical** user stories under this Feature only (prefer sibling Feature slice shape when it already has PBIs).

Each story:

| Field | Rule |
|---|---|
| Title | Outcome slice, not "implement Feature X" |
| Description | HTML `As a … I want … so that …` — narrative, **not** a paste of Scope |
| Story Purpose | 1–2 sentences why it matters |
| Scope | Discrete testable bullets, one concern per line |
| AC | ≥1 testable line per Scope item (GIVEN/WHEN/THEN). Self-check coverage |
| Assumptions | Only genuine leftovers; omit if none |
| TechnicalImplementationDetails | **Required in this skill.** Architecture facts for **this slice**: owning service, identity/data, API/event, PCI boundary, peers, POC branch if the Epic names one. No invented files/classes |
| Readiness | `Ready` / `Near Ready` / `Needs Decision` / `Blocked` |
| Traceability | Feature section + Epic constraint + architecture pick |

Each `Ready` / `Near Ready` story gets Tasks: one **implementable slice** each (persist+API, Portal UI, Card enforce, tests for that slice). Not "frontend / backend / tests" as vague buckets. Not a restatement of the PBI title.

When the slice adds ngx-translate keys in a library, include a **Portal i18n Task** (or a Scope line on the Portal UI PBI) whose AC is: operator UI shows English labels, not raw keys; Portal `en.json` contains the same block as the library showcase. Follow existing sibling keys (e.g. `bin-blacklist` / `card-blacklist`) for locale-file pattern; other locales are `unknown` unless those files already exist.

### Gate B — approve spec pack (no ADO writes)

**Order is mandatory.** The artifact being approved is the suggested PBI pack. The user must see that pack before any Gate B question.

1. **Present the suggested PBIs first** in the user-visible message (not only inside question labels). Use [reference.md](reference.md) § Gate B presentation. Include:
   - Feature → PBI → Task tree (numbered)
   - For **each** suggested PBI: title, story purpose, Scope bullets, AC, Tasks, readiness, traceability
   - Scope→AC matrix
   - Existing children that will be reused (not recreated)
2. **Then** ask pickable questions. `AskQuestion` is allowed only after the pack is already in that same message:

   1. Story list: accept vs edit (recommended: accept Gate A–aligned list).
   2. Deposit/payout / extra slices: in vs follow-on (recommended: match Epic unless user already overrode).
   3. Iteration: inherit Feature vs leave unset (recommended: inherit).
   4. Create now vs draft-only (recommended: create now after this pick).
   5. Any Task the user wants dropped or split.

Hard rules for this gate:

- Do not open questions until the suggested PBI list is fully rendered.
- Do not compress the pack into option labels or a one-line summary (“accept this 4-story pack”).
- Questions refer to the pack above; they do not replace it.

**Stop.** After picks, rewrite the pack from those choices. Remaining gaps stay `unknown`.

### Create (only after Gate B)

For each approved `Ready` / `Near Ready` story, follow the workspace `create-pbi` skill (`ADO-001`):

- `-FeatureId` = this Feature.
- `-Title`, `-Description` (HTML), `-Scope`, `-StoryPurpose`, `-AcceptanceCriteria`.
- `-Assumptions` only if genuine.
- `-Fields` for `Custom.TechnicalImplementationDetails` (this skill directs it) and `Custom.AlreadyDeployedtoProduction` = false unless evidenced otherwise.
- Area/iteration from Gate B.

Then for each approved Task, follow `create-task` (`ADO-002`): `-ParentId` = new PBI ID, title + short HTML description, same area.

If `create-pbi` / `create-task` skills exist in the workspace, execute them rather than hand-rolling `az boards`. On script failure, report the blocker; do not invent a REST fallback unless the user directs MCP under ADR-0023 §1a.

Do not set Task `In Progress`. If the user later asks: `AssignedTo` = directing human (Vassil: `vassil.atanasov@pxp.io`) in the same state call (`ADO-011`).

## Output to the user

- Architecture verdict (short) + picks applied.
- Table: PBI ID/URL, title, readiness, child Task IDs/URLs.
- Scope→AC coverage per PBI.
- Stories skipped (`Needs Decision` / `Blocked`) and who must unblock (`architecture-agent` / `security-review-agent`).
- Epic constraints that affect later PRs (e.g. POC target branch).

## Additional resources

- Field catalog, architecture pack template, and invoke example: [reference.md](reference.md)
