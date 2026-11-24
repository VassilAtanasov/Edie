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
| S3776 | A method whose cognitive complexity is over 15: nested `if`s and loops, `&&`/`\|\|` chains, `continue`/`break`. Sonar-only, not visible in a local build. | Extract inner branches into named private methods, and use guard clauses (early `return`/`continue`) instead of nesting. |
| S3218 | A member of a nested type (method, record property) with the same name as a member of the outer class. Sonar-only, not visible in a local build. | Rename the inner member (for example `ReadRun`, `RowFailure`). |
| S6605 | `.Any(predicate)` on a `List<T>` or an array. | `list.Exists(predicate)` / `Array.Exists(array, predicate)`. |
| S6603 | `.All(predicate)` on a `List<T>` or an array. | `list.TrueForAll(predicate)` / `Array.TrueForAll(array, predicate)`. |
| S3236 | Passing the `paramName` argument to a guard that takes it via `[CallerArgumentExpression]`, for example `ArgumentException.ThrowIfNullOrWhiteSpace(subject.OwnerId, nameof(subject))`. | Omit the argument. If a different name must be reported, write an explicit `if` and `throw new ArgumentException(..., nameof(subject))`. |
| S3398 | A `private static` method in the outer class that only a nested class uses. Sonar-only, not visible in a local build. | Move the method into the nested class. |
| S3358 | A nested ternary (`a ? b : c ? d : e`). Sonar-only, not visible in a local build. | An `if`/`return` chain or a `switch` expression. |
| S3871 | A non-`public` exception type. Sonar-only, not visible in a local build. | Make the exception type `public`. |
| S2219 | A constant pattern against a named constant, `x is SomeConst` (observed: `result.Error.Code is UploadErrors.UploadRowLimitExceededErrorCode`; Sonar's message "Use a 'null' check instead" does not fully explain it). Sonar-only, not visible in a local build. | `x == SomeConst`. |

## Check before handing back

A clean local build does not prove a clean Sonar scan. The local NUnit.Analyzers (4.15.0) does not flag NUnit2045 inside non-`[Test]` helpers, and S2692, S3267, S3776, S3218, S3398, S3358, S3871 and S2219 are Sonar-only (the repositories do not reference SonarAnalyzer.CSharp). S6605 and S6603 may not show locally either; treat them as Sonar-only.

1. Build each changed project with its own SARIF log, outside the repository clone (for example `<RUN_FOLDER>/sarif/`):
   `dotnet build <proj> --no-incremental --no-dependencies -p:ErrorLog=<RUN_FOLDER>/sarif/<proj>.sarif`
2. List the results on added lines of `git diff -U0 origin/<PBI_BRANCH>...HEAD`. Fix every hit on a new line.
3. Read the new lines for the rules the build misses: ungrouped asserts in test helpers (NUnit2045), and the Sonar-only rules `IndexOf(...) > 0` (S2692), a loop whose whole body is one `if` on the loop variable (S3267), deeply nested or branchy methods (S3776), nested-type members named like outer members (S3218), outer `private static` methods used only by a nested class (S3398), nested ternaries (S3358), non-public exception types (S3871), `x is SomeConst` (S2219), and `.Any(pred)`/`.All(pred)` on a `List<T>` or array (S6605/S6603).
4. Do not commit analyzer config used for the check (for example a temporary `.editorconfig` that raises severities).
5. Say in the report which projects were checked and that no hit remains on a new line.

## Review

The code reviewer checks the table on added lines of the whole task diff (`origin/<PBI_BRANCH>...<TASK_BRANCH>`), not only the last commit. Each hit is a must-fix with `file:line` and the rule id.

The code reviewer reads every added loop for S3267, and the added lines for the other Sonar-only rules: S3776, S3218, S3398, S3358, S3871, S2219, S6605 and S6603. No local build or SARIF log shows them, so reading the new lines is the only check before Sonar.

## Findings on a PR

- Change only the code. Never post on, reply to or resolve a `svc_unity_sonar` thread.
- On a build-pbi final PR, Sonar findings go in through `build-pbi <PBIId> --followup sonar`. A finding that matches a rule above is class `fix`.
- A Sonar finding for a rule not in this table is fixed the same way. Report it so the table can be extended.
