# Build PBI — Sonar rules (Unity C# / NUnit)

`svc_unity_sonar` posts every analyzer finding on a line a PR adds or changes as a PR thread, info level included. `dotnet build` does not print all of them. Code written in a build-pbi Task avoids the rules below, and the implementer checks the new lines before handing back.

Applies when the change adds or edits `.cs` files. Hits on lines the change did not add or edit are left alone.

## Rules to write against

| Rule | Avoid | Do instead |
|---|---|---|
| NUnit2045 | Two or more independent `Assert.That` calls in a row outside a scope. Applies to private assert helper methods too, not only `[Test]` bodies. | `using (Assert.EnterMultipleScope()) { ... }`. A helper with two or more independent asserts opens its own scope (scopes nest). Asserts where one depends on another (`Is.Not.Null`, then a dereference) stay outside the scope. |
| NUnit2056 | `Assert.Multiple(() => { ... })` and `Assert.MultipleAsync`, even where older repo code still uses them. | `using (Assert.EnterMultipleScope())`. |
| NUnit1028 | A public member that is not a test, such as a `TestCaseSource` or `ValueSource` method or field. | Make it `private static`. |
| CA1861 | `new[] { ... }` literals passed as arguments, including `Is.EqualTo(new[] { ... })`. | A named `private static readonly T[]` field. |
| CA1859 | A private or local helper that returns an interface type. | Return the concrete type (`string[]`, not `IReadOnlyList<string>`). |
| S2692 | `IndexOf(...) > 0`. Sonar-only, not visible in a local build. | State the intent directly, for example `!s.StartsWith(t, StringComparison.Ordinal) && s.Contains(t, StringComparison.Ordinal)`. |
| S3267 | A `foreach` (or `for` over a collection) whose whole body is one `if` on the loop variable, including a side-effecting filter such as `if (set.Add(x))`. Sonar-only, not visible in a local build. | Filter before the loop with `.Where(...)`. For a de-duplicating add, use `.Distinct(comparer)` / `.DistinctBy(key, comparer)` then `.Where(x => !held.Contains(x))`, so the predicate has no side effect. Enumerate the submitted sequence, not a `HashSet`, when order matters (a `HashSet`'s enumeration order is not guaranteed). Never enumerate the collection the loop body mutates. |

## Check before handing back

A clean local build does not prove a clean Sonar scan. The local NUnit.Analyzers (4.15.0) does not flag NUnit2045 inside non-`[Test]` helpers, and S2692 and S3267 are Sonar-only (the repositories do not reference SonarAnalyzer.CSharp).

1. Build each changed project with its own SARIF log, outside the repository clone (for example `<RUN_FOLDER>/sarif/`):
   `dotnet build <proj> --no-incremental --no-dependencies -p:ErrorLog=<RUN_FOLDER>/sarif/<proj>.sarif`
2. List the results on added lines of `git diff -U0 origin/<PBI_BRANCH>...HEAD`. Fix every hit on a new line.
3. Read the new lines for the rules the build misses: ungrouped asserts in test helpers (NUnit2045), and the Sonar-only rules `IndexOf(...) > 0` (S2692) and a loop whose whole body is one `if` on the loop variable (S3267).
4. Do not commit analyzer config used for the check (for example a temporary `.editorconfig` that raises severities).
5. Say in the report which projects were checked and that no hit remains on a new line.

## Review

The code reviewer checks the table on added lines of the whole task diff (`origin/<PBI_BRANCH>...<TASK_BRANCH>`), not only the last commit. Each hit is a must-fix with `file:line` and the rule id.

The code reviewer reads every added loop for S3267. No local build or SARIF log shows it, so reading the new lines is the only check before Sonar.

## Findings on a PR

- Change only the code. Never post on, reply to or resolve a `svc_unity_sonar` thread.
- On a build-pbi final PR, Sonar findings go in through `build-pbi <PBIId> --followup sonar`. A finding that matches a rule above is class `fix`.
- A Sonar finding for a rule not in this table is fixed the same way. Report it so the table can be extended.
