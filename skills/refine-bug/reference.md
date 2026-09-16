# refine-bug reference

Load from SKILL.md when drafting the Description or implementer handoff.

## HTML Description template

Replace placeholders. Keep valid HTML ADO renders (`<h2>`, `<p>`, `<ul>`, `<ol>`, `<li>`, `<code>`, `<strong>`). Do not use Markdown inside Description.

```html
<h2>Summary</h2>
<p><!-- One paragraph: symptom + user impact --></p>

<h2>Environment</h2>
<ul>
  <li><strong>Surface:</strong> <!-- e.g. Portal local http://localhost:1010 --></li>
  <li><strong>Gateway/API:</strong> <!-- e.g. http://localhost:5202 --></li>
  <li><strong>Branch:</strong> <!-- e.g. Edison/306969-build + file: linked @pxp/transactionsriskscreening --></li>
  <li><strong>Build/run mode:</strong> <!-- ng serve vs Docker dev-latest --></li>
</ul>

<h2>Steps to reproduce</h2>
<ol>
  <li><!-- step --></li>
</ol>

<h2>Expected</h2>
<p><!-- testable expected behaviour --></p>

<h2>Actual</h2>
<p><!-- testable actual behaviour --></p>

<h2>Artifacts</h2>
<ul>
  <li><strong>Screenshot:</strong> <!-- session path; QA to attach in ADO Discussion --></li>
  <li><strong>Route/URL:</strong> <code><!-- path --></code></li>
  <li><strong>Logs:</strong> <!-- redacted excerpt or unknown --></li>
</ul>

<h2>Technical analysis</h2>
<p><strong>Root cause:</strong> <!-- evidenced statement --></p>
<ul>
  <li><strong>Repo:</strong> <!-- e.g. Pxp.Unity.Portal --></li>
  <li><strong>Files:</strong> <code><!-- path --></code></li>
  <li><strong>Mechanism:</strong> <!-- e.g. missing i18n keys in Portal en.json --></li>
</ul>
<p><strong>Suggested fix:</strong></p>
<ul>
  <li><!-- minimal concrete fix --></li>
</ul>
<p><strong>Comparison / regression:</strong> <!-- e.g. BIN Blacklist on same page renders correctly --></p>

<h2>Verification</h2>
<ol>
  <li><!-- how to confirm fix --></li>
</ol>

<h2>Unknowns</h2>
<ul>
  <li><!-- explicit gaps --></li>
</ul>
```

## Discussion comment (after create/update)

Plain text or simple HTML for `Add-WorkItemComment.ps1`:

```text
refine-bug: enriched Description with technical analysis from QA session <date>.
QA action: attach screenshot(s) to this Discussion thread if not already present.
Implementer: see Description → Technical analysis and Verification.
```

## Implementer prompt snippet

Give the user this block after refine-bug completes:

```text
Implement Bug <id> on <repo>, branch <feature-branch>, target integration branch <integration-branch>.

Bug is refined — read Description in ADO; do not re-triage. Root cause and suggested fix are in Technical analysis. Verify per Verification section (local Portal :1010 / gateway :5202 as stated).
```

Customize `<repo>`, branches, and ports from the bug pack.

## Parent PBI resolution

1. User names PBI ID → use directly.
2. User names Feature ID → `Get-WorkItemChildren.ps1 -Id <featureId>`, match by title/scope (UI bug → UI PBI).
3. Conversation context (e.g. PBI 309171 Card Blacklist UI) → confirm with QA pick.
4. Still ambiguous → **stop**; do not create Bug without parent.

## Invoke examples

**Enrich existing:**

```text
refine-bug 309212
```

**Create from QA session:**

```text
refine-bug

Card Blacklist modal shows translation keys on Portal localhost:1010.
Screenshot in chat. Parent PBI 309171, Feature 306969.
```

**Draft only:**

```text
refine-bug 309212 — draft only, no ADO write yet
```

## Bug fields (Unity / PXP Scrum - Unity)

Re-query if create fails.

| UI label | Reference | refine-bug |
|---|---|---|
| Title | `System.Title` | Required |
| Description | `System.Description` | Set on create if required; **not shown** on the Bug Details form |
| Repro Steps | `Microsoft.VSTS.TCM.ReproSteps` | **Required visible pack** (HTML template above) |
| System Info | `Microsoft.VSTS.TCM.SystemInfo` | Environment HTML |
| Acceptance Criteria | `Microsoft.VSTS.Common.AcceptanceCriteria` | Verification checkbox HTML |
| Severity | `Microsoft.VSTS.Common.Severity` | From Gate A |
| Value Area | `Microsoft.VSTS.Common.ValueArea` | `Business` or `Architectural` per README |
| Area / Iteration | `System.AreaPath` / `System.IterationPath` | Inherit parent PBI unless QA overrides |

There is no `Custom.TechnicalImplementationDetails` on Bug — put implementer detail in **Repro Steps** (and keep a copy in `System.Description` if the create path already wrote it).

## Do not

- Paste Markdown into `System.Description` or Repro Steps.
- Create a Bug without `-ParentId` (parent PBI).
- Claim root cause without source or session evidence.
- Store PAN/secrets in Description or comments.
- Use `review-bugbot` or `fix-bug` workflow as a substitute for this skill.
