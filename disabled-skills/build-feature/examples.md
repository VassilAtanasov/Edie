# Example: Feature 306968

Title: Internal Risk Screening - BIN Blacklist. Area: `Unity\Edison`. Parent Epic: 306966.

Invoke: `build-feature 306968`

Suggested integration branch: `Edison/306968-build`.

| PBI | State (as of 2026-09-14) | Drain? | Repos (from Scope, confirm in source) |
|---|---|---|---|
| 306979 Persist Internal BIN in TRS | New | yes if AC/Scope hold | `Pxp.Unity.TransactionsRiskScreening` |
| 306980 Gateway Portal BIN APIs | New | yes | `Pxp.Unity.Gateway` |
| 306981 Portal UI | New | yes after 306980 APIs exist on the Gateway integration branch | `Pxp.Unity.Portal` → frontend workflow |
| 306982 Evaluate BIN + Card refuse | Committed | yes; **two** sequential slices | TRS then `Pxp.Unity.Card` |
| 308611 Gateway Sessions before 3DS | Committed | yes; do not change `Pxp.Unity.ThreeDSecureAuthentication` | TRS evaluate (if not already) then Gateway Sessions |
| 308548 Checkout 3DS / 4 repos | Rejected | **skip** | — |

Order: 306979 → 306980 → 306982 (TRS then Card) → 308611 → 306981.

**Commits:** each Task under a PBI gets its own commit on `<integration>-<PBIId>` (e.g. `AB#306980` PBI with Tasks `AB#306983`, `AB#306984` → two commits before merge to `Edison/306968-build`). PBIs with no Tasks: one `AB#<PBIId>` commit.

Class 2 (TRS Api.Contracts + Card ExternalContracts): land on integration branches; do **not** complete PRs to `development`. Human integrates later.

PCI: BIN prefixes only; fail security review if PAN appears.

After the drain, user opens per-repo PR `Edison/306968-build` → inventory `defaultBranch` and applies ADR-0004 class 2 reviewers.
