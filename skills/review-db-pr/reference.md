# Review DB PR — reference

Rule ids here are local to this skill. `DATA-*` are Policy rules (`guidelines/data-persistence-and-migrations.md`), cited, not restated as authority. `DBR-*` are the DB Scripts Reviewers' rules, learned from their PR comments (TRS PRs 113110, 114757); they are not yet in Policy, which marks column conventions `UNKNOWN` (`[DATA-005]`). Proposing them to Policy as `DATA-*` rules is a follow-up; when that lands, cite the Policy ids and drop the copies here.

Borrowed ideas: `hmohamed01/SQL-Expert` (`sql-expert/SKILL.md`: SARGable predicates, implicit conversion, dynamic SQL, transactions) and `dotnet/skills` `optimizing-ef-core-queries` (MIT: evidence gating, EF anti-patterns).

## Severity

- **must-fix**: breaks a `MANDATORY` rule or a `DBR` rule, or risks data loss, a failed deploy, or PAN exposure.
- **should-fix**: a correctness or performance risk with evidence (code path, missing index, plan, query count).
- **note**: alternatives, style, existing drift nearby, anything not verified.

## A. Unity rules

| Id | Sev | Rule |
|---|---|---|
| DATA-001 | must | Schema changes go through the service's `.Db` DACPAC project. EF `Migrations/` or runtime `CREATE`/`ALTER` that diverge from the DACPAC are must-fix. |
| DATA-002 | must | A service uses only its own `AppDb`: no three-part names to another database, synonyms, linked servers or cross-service connection strings. |
| DATA-003 | must | `Pxp.Unity.Db` or cross-domain schema/ETL: class-3 gate and a named per-domain owner in the PR description, else `STOP-CLASS-3`. |
| DATA-004 | must | Sensitive fields use `[Encrypted]` / `Pxp.Unity.Encryption`; card search uses a fingerprint hash; no raw PAN column. Route to `security-review-agent`. |
| DATA-005 | should | Timestamps in UTC; audit columns where the table family has them. |
| DATA-006 | should | PostgreSQL only where there is precedent (Toggle, Reporting). |
| DBR-001 | must | **Own identity PK.** First column `[<TableSingular>ID] [TRowID] IDENTITY(1,1) NOT NULL`, `CONSTRAINT [PK_<Table>] PRIMARY KEY CLUSTERED`. Never a parent's id as the PK (shared-key one-to-one); for one-to-one keep the parent id as an FK with `CREATE UNIQUE NONCLUSTERED INDEX [IX_<Table>_<ParentID>]`. Children reference this table's id, not the grandparent's. Seeded lookup tables keep a non-identity `<Name>ID`. |
| DBR-002 | must | **FK column = parent key name, exactly.** `RestrictionStatusID`, not `Status`; `RiskCheckTypeID`, not `CheckTypeID`. No role or domain names on FK columns; keep the domain name on the C# property with `HasColumnName`. |
| DBR-003 | must | **Every FK has an unfiltered index whose first column is the FK column.** A composite index counts only if it leads with it; a filtered index, or one led by another column (for example `TenantId`), does not. Low-cardinality lookup FKs get `IX_<Table>_<Column>` too. Names are `IX_<Table>_<Columns>`; a renamed column renames its indexes. |
| DBR-004 | must | **Date defaults use `SYSUTCDATETIME()`.** `CreatedOn`/`UpdatedOn` are `[TDateTime]` (`DATETIME2(7)`) with `CONSTRAINT [DF_<Table>_<Column>] DEFAULT (SYSUTCDATETIME())`; `GETUTCDATE()` (returns `datetime`) is must-fix even when an older table uses it. |

## B. DACPAC deployment safety

| Id | Sev | Rule |
|---|---|---|
| SQL-D1 | must | A renamed column or table has a `.refactorlog` entry; otherwise the deploy drops and recreates it and loses data. |
| SQL-D2 | must | A new `NOT NULL` column on an existing table has a default, or the PR states the table is empty in every environment. |
| SQL-D3 | must | A narrowed type or length, or a dropped column or table, comes with a stated data plan; `BlockOnPossibleDataLoss` must not be turned off to get past it. |
| SQL-D4 | should | A changed PK or clustered index on a large table is called out (rebuild cost, locking). |
| SQL-D5 | must | Pre/post-deployment scripts are idempotent and re-runnable (`IF NOT EXISTS`, `MERGE` for seeds). |
| SQL-D6 | must | A new lookup table that an FK references has its seed rows in post-deployment. |
| SQL-D7 | should | New objects are part of the `.sqlproj` (check when the project lists files instead of globbing). |

## C. T-SQL quality and security

| Id | Sev | Rule |
|---|---|---|
| SQL-Q1 | should | SARGable predicates: no function or arithmetic on an indexed column (`YEAR(col)`, `CAST(col …)`, `col + 1`); use half-open ranges. |
| SQL-Q2 | should | Parameter and column types match (`nvarchar` vs `varchar`, `int` vs `bigint`), so no implicit conversion. |
| SQL-Q3 | should | Types: repo UDTs (`TRowID`, `TDateTime`) where the repo has them; explicit lengths and precision; no `NVARCHAR(MAX)` for keys or indexed columns; `DECIMAL(p,s)` for amounts. |
| SQL-Q4 | should | NULL semantics: no `= NULL`; no `NOT IN` over a nullable subquery. |
| SQL-Q5 | should | Writing procedures use `SET XACT_ABORT ON` and `TRY…CATCH` with rollback; no transaction left open on an error path. |
| SQL-Q6 | should | Composite index order: equality columns first, then range or sort columns; `INCLUDE` only for a shown covering need; mention the write cost. |
| SQL-Q7 | note | No `SELECT *` in views or procedures; no `NOLOCK` without a stated reason. |
| SQL-Q8 | should | Version-specific syntax (for example `GREATEST`, `GENERATE_SERIES` need 2022) matches the target SQL Server version; if that version is not evidenced, mark `unknown` and ask. Verify exact syntax against learn.microsoft.com, not memory. |
| SQL-S1 | must | Dynamic SQL uses `sp_executesql` with parameters and `QUOTENAME` for identifiers; never concatenated input. |
| SQL-S2 | must | No logins, users, grants or role changes in a service's scripts (route to `platform-engineer-agent` / `security-review-agent`). |
| SQL-S3 | should | Tenant-scoped tables carry the tenant key their table family uses, and queries filter by it. |

## D. EF Core

EF is used in Unity for encryption converters and a few stores; it never owns the schema (`DATA-001`). Its "add the index and generate a migration" fix does not apply here: indexes go in the `.Db` scripts.

| Id | Sev | Rule |
|---|---|---|
| EF-01 | should | Filters, projections and aggregates stay on `IQueryable` until execution (no `ToList()` then `Where`); `AnyAsync` for existence; results are bounded or paged. A repository returning a deferred `IEnumerable` is not enumerated twice. |
| EF-02 | should | Index-compatible predicates (as SQL-Q1); an index does not fix a wrapped column. |
| EF-03 | note | `AsNoTracking` for read-only entity graphs; not needed for scalar projections; entities inside DTOs are still tracked. |
| EF-04 | must | No N+1: no navigation access inside loops; no lazy-loading proxies in services. |
| EF-05 | should | Sibling collection `Include`s: project instead, or `AsSplitQuery` with a unique order. |
| EF-06 | should | Paging has a unique, stable order (`ThenBy`, not a second `OrderBy`); keyset paging for deep pages. |
| EF-07 | must | The model matches the DACPAC: `ToTable`, `HasColumnName`, `HasIndex`, types, lengths and nullability equal the scripts. |
| EF-08 | should | No `SaveChanges` per row in repositories or loops; unit-of-work and outbox boundaries kept (see the user's memory `unit-of-work-failure-paths`). |
| EF-09 | should | `ExecuteUpdate`/`ExecuteDelete`: tracked entities go stale, concurrency tokens are not checked, and each call is its own statement; share the transaction when it must be atomic. |
| EF-10 | must | One operation at a time per `DbContext`: no `Task.WhenAll` over a shared context, no singleton or static context. |
| EF-11 | must | Raw SQL only via the interpolated `FromSql`/`ExecuteSql` overloads; never `FromSqlRaw`/`ExecuteSqlRaw` with concatenated input. |
| EF-12 | note | `EF.CompileQuery`, `EF.Parameter`, `EF.Constant` only with workload evidence. |

## E. Dapper

Dapper with `IDbConnection` is the main Unity data-access style.

| Id | Sev | Rule |
|---|---|---|
| DAP-1 | must | Values go in as parameters (anonymous object or `DynamicParameters`); no interpolated or concatenated SQL. |
| DAP-2 | should | `DbString { IsAnsi = true }` or `DbType.AnsiString` for `varchar` columns, so the `nvarchar` default does not cause SQL-Q2. |
| DAP-3 | must | Column names in SQL strings match the DACPAC after renames (a `DBR-002` rename silently breaks Dapper mapping). |
| DAP-4 | should | Large reads are paged or use `buffered: false`; `commandTimeout` only with a reason. |
| DAP-5 | should | Connection and transaction come from the unit of work; no connection opened per row. |

## F. Consistency and tests

| Id | Sev | Rule |
|---|---|---|
| CON-1 | must | The same column names, types and indexes in the `.sql`, EF configuration, Dapper SQL and test schemas (SQLite copies). |
| CON-2 | should | New tables are added to the repo's script-level tests where they exist (TRS: `DbProjectScripts.ForeignKeysOf` / `IndexLeadingColumnsOf`; tests `ForeignKeyColumns_CarryTheNameOfTheParentTablesKey` and `ForeignKeys_AreCoveredByAnIndexThatStartsWithTheirColumn` in `RiskCheckEntryMappingTests` take `TestCase` lists). |
| CON-3 | must | Behaviour change ships with tests at the repo's existing level; none weakened or deleted (`[HND-025]`). |
| CON-4 | note | DDL sketched in a design record or Task description follows the `DBR` rules too. |

## Verify query

Run on the scratch catalog after deploy. Every row should read `ok` and name an index.

```sql
SELECT t.name AS [Table], c.name AS FkColumn, pc.name AS ParentKey,
       CASE WHEN c.name = pc.name THEN 'ok' ELSE 'DBR-002' END AS NameCheck,
       ISNULL(ix.name, 'DBR-003 missing') AS CoveringIndex
FROM sys.foreign_key_columns fkc
JOIN sys.tables  t  ON t.object_id = fkc.parent_object_id
JOIN sys.columns c  ON c.object_id = fkc.parent_object_id AND c.column_id = fkc.parent_column_id
JOIN sys.columns pc ON pc.object_id = fkc.referenced_object_id AND pc.column_id = fkc.referenced_column_id
OUTER APPLY (SELECT TOP (1) i.name
             FROM sys.indexes i
             JOIN sys.index_columns ic ON ic.object_id = i.object_id AND ic.index_id = i.index_id
             WHERE i.object_id = fkc.parent_object_id AND ic.key_ordinal = 1
               AND ic.column_id = fkc.parent_column_id AND i.has_filter = 0) ix
ORDER BY t.name, c.name;
```

`GETUTCDATE()` defaults (DBR-004):

```sql
SELECT OBJECT_NAME(dc.parent_object_id) AS [Table], dc.name, dc.definition
FROM sys.default_constraints dc
WHERE dc.definition LIKE '%getutcdate%' OR dc.definition LIKE '%getdate%';
```

## Report format

```
VERDICT: PASS | REWORK | STOP-CLASS-3
PR: !<id>  head: <sha>  repo: <repo> (<repoType>)  round: <n>
Roles: <lead reviewer> [+ data-engineer-agent] [+ security-review-agent]
DB files: schema <n>, EF <n>, Dapper <n>, tests <n>
Runtime verify: done on SCRATCH_<Repo>_<PR> | not run

| # | Sev | Rule | File:line | Finding | Fix |
|---|-----|------|-----------|---------|-----|

Sound patterns seen: <one line each>
Existing drift nearby (not this PR's): <table — rule>
Not verified: <one line each>
Gates: DB Scripts Reviewers (branch policy) · class 3 <yes|no> · security <yes|no>
Threads: <finding #> → <thread id>   (after posting)
```

`PASS` only with no must-fix finding.

## Thread format

One thread per finding, at most 1,000 characters:

```
<Sev> <Rule>: <file>:<line>
<What is wrong, one or two sentences.>
Fix: <the change, with a one-line example when it helps>
(Edie review-db-pr, round <n>, head <short sha>)
```

Summary thread:

```
review_record: review-db-pr round <n> on <short sha>: <VERDICT>
must-fix <n>, should-fix <n>, notes <n> (notes not posted)
Must-fix: <rule> <file:line>; …
Gates: DB Scripts Reviewers · class 3 <yes|no> · security <yes|no>
```

Notes are not posted as threads; they stay in the report.
