# Stored Procedures

*A deep-dive walkthrough of stored procedures — covering parameters and output parameters, plan caching and recompilation specific to procedures (and how this connects directly to this series' Execution Plans guide's parameter-sniffing discussion), error handling with TRY/CATCH and THROW, the genuine security benefits of permission abstraction, temp tables vs. table variables inside a procedure body, the stored-procedures-vs-dynamic-SQL-vs-ORM debate argued honestly from both sides, and how EF Core calls into procedures when raw LINQ-to-Entities isn't the right tool.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [What a Stored Procedure Actually Is](#1-what-a-stored-procedure-actually-is)
3. [Parameters: Input, Output, and Defaults](#2-parameters-input-output-and-defaults)
4. [Return Values vs. Output Parameters vs. Result Sets](#3-return-values-vs-output-parameters-vs-result-sets)
5. [Plan Caching for Stored Procedures](#4-plan-caching-for-stored-procedures)
6. [Recompilation: sp_recompile, WITH RECOMPILE, OPTION (RECOMPILE)](#5-recompilation-sp_recompile-with-recompile-option-recompile)
7. [Error Handling: TRY/CATCH and THROW](#6-error-handling-trycatch-and-throw)
8. [Temp Tables vs. Table Variables Inside a Procedure](#7-temp-tables-vs-table-variables-inside-a-procedure)
9. [Security: Permission Abstraction](#8-security-permission-abstraction)
10. [Stored Procedures and SQL Injection](#9-stored-procedures-and-sql-injection)
11. [Stored Procedures vs. Dynamic SQL vs. ORM-Generated SQL](#10-stored-procedures-vs-dynamic-sql-vs-orm-generated-sql)
12. [Calling Stored Procedures from EF Core](#11-calling-stored-procedures-from-ef-core)
13. [Versioning and Deployment](#12-versioning-and-deployment)
14. [Common Pitfalls](#13-common-pitfalls)
15. [Quick Reference Table](#quick-reference-table)
16. [Conclusion](#conclusion)

---

## Introduction

A stored procedure is a named, precompiled batch of SQL stored inside the database itself — parameterized, callable by name, and capable of containing genuine procedural logic (conditionals, loops, error handling) alongside ordinary queries. This guide goes deep on the mechanics that matter most in practice: how a procedure's plan is cached and why that interacts directly with this series' Execution Plans guide's parameter-sniffing discussion, the genuine, specific security benefit procedures provide beyond "they prevent SQL injection" (a claim worth examining precisely, since parameterized ad hoc queries prevent injection too), and an honest treatment of the stored-procedures-vs-ORM debate that doesn't pretend one side is simply correct.

```plaintext
CREATE PROCEDURE GetOrdersByCustomer
    @CustomerId INT
AS
BEGIN
    SELECT Id, OrderDate, Total FROM Orders WHERE CustomerId = @CustomerId;
END;

EXEC GetOrdersByCustomer @CustomerId = 42;
-- compiled ONCE (per Section 4), called REPEATEDLY — the exact mechanism
-- this series' Execution Plans guide's Section 10 plan-caching discussion describes
```

---

## 1. What a Stored Procedure Actually Is

### A named, persisted batch of T-SQL, living inside the database alongside tables and indexes

```sql
CREATE PROCEDURE UpdateOrderStatus
    @OrderId INT,
    @NewStatus NVARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;  -- suppresses the "N rows affected" message — a genuine, standard performance/noise reduction
    UPDATE Orders SET Status = @NewStatus WHERE Id = @OrderId;
END;
```

A procedure is created once (via `CREATE PROCEDURE`) and persists as a database object, exactly like a table or an index — it's called by name (`EXEC`), with parameters supplied at the call site, and its body can contain not just a single `SELECT`/`UPDATE` but genuine control flow (`IF`/`WHILE`), local variables, temp tables (Section 7), and structured error handling (Section 6) — a meaningfully richer programming surface than a single ad hoc query.

### `SET NOCOUNT ON`: a small, nearly-universal convention worth understanding why it exists

```plaintext
Without it, EVERY statement inside the procedure sends a "(N rows
  affected)" message back to the CALLING application — extra, usually
  unwanted network traffic and client-side processing overhead,
  multiplied across every statement in the procedure. SETTING IT ON
  suppresses this entirely, which is why it appears at the top of
  nearly every well-written procedure.
```

---

## 2. Parameters: Input, Output, and Defaults

### Input parameters: the standard way callers pass data in

```sql
CREATE PROCEDURE GetOrdersByStatus
    @Status NVARCHAR(20) = 'Pending'   -- a DEFAULT value — @Status is OPTIONAL at the call site
AS
BEGIN
    SELECT * FROM Orders WHERE Status = @Status;
END;

EXEC GetOrdersByStatus;                  -- uses the default, 'Pending'
EXEC GetOrdersByStatus @Status = 'Shipped';  -- overrides it explicitly
```

Parameters are declared with a type (exactly like a C# method's typed parameters) and, optionally, a default value — this directly parallels this series' Delegates guide's discussion of method signatures, just expressed in T-SQL rather than C#.

### Output parameters: returning a scalar value back through a parameter, not a result set

```sql
CREATE PROCEDURE CreateOrder
    @CustomerId INT,
    @Total DECIMAL(10,2),
    @NewOrderId INT OUTPUT   -- the OUTPUT keyword — this parameter carries a value BACK to the caller
AS
BEGIN
    INSERT INTO Orders (CustomerId, Total) VALUES (@CustomerId, @Total);
    SET @NewOrderId = SCOPE_IDENTITY();  -- the just-inserted row's auto-generated Id
END;
```

```sql
DECLARE @orderId INT;
EXEC CreateOrder @CustomerId = 42, @Total = 99.99, @NewOrderId = @orderId OUTPUT;
SELECT @orderId;  -- the CALLER must ALSO specify OUTPUT at the call site, or the value won't flow back
```

Worth noting precisely: `OUTPUT` must be specified both in the procedure's own parameter declaration *and* again at the call site — omitting it at the call site is a genuinely common mistake that silently discards the output value rather than raising any error, since without it the parameter is simply treated as input-only.

---

## 3. Return Values vs. Output Parameters vs. Result Sets

### Three genuinely distinct ways a procedure communicates back to its caller

```sql
CREATE PROCEDURE ValidateOrder
    @OrderId INT
AS
BEGIN
    IF NOT EXISTS (SELECT 1 FROM Orders WHERE Id = @OrderId)
        RETURN -1;    -- RETURN: a single, INTEGER-only status code — conventionally used for success/failure
    RETURN 0;
END;
```

```sql
DECLARE @result INT;
EXEC @result = ValidateOrder @OrderId = 42;  -- captures the RETURN value
```

`RETURN` is specifically, exclusively for a single integer status code — conventionally `0` for success, non-zero for various failure categories — and is structurally distinct from both `OUTPUT` parameters (Section 2, which can carry any data type, not just integers) and an actual `SELECT`-produced result set (the normal way a procedure returns rows of data to be consumed as a table-shaped result). Conflating these three — trying to `RETURN` a genuine data value, say — is a real, common source of confusion for developers newer to T-SQL specifically.

---

## 4. Plan Caching for Stored Procedures

### A procedure's plan is compiled once, on first execution, and reused — exactly the mechanism this series' Execution Plans guide's Section 10 describes

```plaintext
The FIRST time a procedure is EXECUTED (not when it's CREATED — creation
  alone doesn't compile a plan), SQL Server compiles and CACHES a plan
  for its entire body — every SUBSEQUENT call reuses that SAME cached
  plan, skipping the compilation cost entirely, which is precisely the
  performance benefit this series' Execution Plans guide's Section 10
  attributes to plan caching generally.
```

This is worth restating specifically for procedures because it's the textbook, canonical example of plan caching in action — a procedure's *entire name* (not just one query's text) is the cache key, which means the whole body's plan is reused as a unit across every call with different parameter values.

### Why this makes procedures the textbook example of parameter sniffing

```plaintext
Per this series' Execution Plans guide's Section 9: a procedure called
  FIRST with a highly selective @Status value produces a plan optimized
  for THAT selectivity, cached, and then reused — correctly or NOT —
  for every SUBSEQUENT call, including ones with a genuinely different,
  LOW-selectivity value. This is EXACTLY what that guide's Section 9
  describes, and stored procedures are the single most common real-world
  context where parameter sniffing actually surfaces as a production issue.
```

This connection is worth making explicit, since "stored procedures are slow sometimes, for no obvious reason" is a genuinely common complaint whose root cause, traced back far enough, is almost always exactly this — not a flaw in the procedure itself, but a direct, specific consequence of the same plan-reuse mechanism that makes procedures efficient in the first place.

---

## 5. Recompilation: sp_recompile, WITH RECOMPILE, OPTION (RECOMPILE)

### Three different scopes of "force a fresh plan," each with a genuinely different trade-off

```sql
-- Forces recompilation on the PROCEDURE'S NEXT CALL, ONE TIME —
-- after that, normal caching resumes
EXEC sp_recompile 'GetOrdersByStatus';

-- Forces EVERY SINGLE EXECUTION of this procedure to recompile — the plan
-- is NEVER cached or reused at all for this procedure
CREATE PROCEDURE GetOrdersByStatus
    @Status NVARCHAR(20)
WITH RECOMPILE
AS
BEGIN
    SELECT * FROM Orders WHERE Status = @Status;
END;

-- Forces recompilation of ONLY this ONE STATEMENT within the procedure,
-- leaving the REST of the procedure's plan cached and reused normally
SELECT * FROM Orders WHERE Status = @Status OPTION (RECOMPILE);
```

This directly extends this series' Execution Plans guide's Section 9 mitigation discussion — `sp_recompile` is a one-time, administrative nudge (useful right after a significant data change or statistics update); `WITH RECOMPILE` on the procedure definition is the broadest, most expensive option, paying full compilation cost on literally every call, appropriate only when a procedure's performance genuinely varies so unpredictably by parameter that caching provides no net benefit at all; `OPTION (RECOMPILE)` on a single statement is the most surgical, letting you isolate the fix to precisely the one query inside a larger procedure that actually suffers from parameter sniffing, leaving the rest of the procedure's plan caching fully intact.

---

## 6. Error Handling: TRY/CATCH and THROW

### Structured error handling, directly paralleling this series' Exception Handling guide's C# discipline

```sql
CREATE PROCEDURE TransferFunds
    @FromAccountId INT, @ToAccountId INT, @Amount DECIMAL(10,2)
AS
BEGIN
    SET XACT_ABORT ON;   -- per this series' Transactions guide's Section 3
    BEGIN TRY
        BEGIN TRANSACTION;
            UPDATE Accounts SET Balance = Balance - @Amount WHERE Id = @FromAccountId;
            UPDATE Accounts SET Balance = Balance + @Amount WHERE Id = @ToAccountId;
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;   -- re-raises the ORIGINAL error, preserving its message/number — exactly
                 --  this series' C# guides' bare `throw;` preserving the original stack trace
    END CATCH;
END;
```

This is genuinely the same discipline this series' Exception Handling guide's Section 1 and this series' Transactions guide's Section 3 both establish, just expressed in T-SQL — `TRY`/`CATCH` catches an error, `ROLLBACK` undoes any partial work (guarded by `@@TRANCOUNT` to avoid rolling back a transaction that was never actually opened, or was already closed), and bare `THROW` re-raises the original error rather than masking it with a new, less informative one.

### Custom, application-meaningful errors with THROW

```sql
IF @Amount <= 0
    THROW 50001, 'Transfer amount must be positive.', 1;
```

`THROW` with explicit arguments raises a custom error with your own error number (user-defined errors conventionally start at 50000+), message, and state — this is the direct T-SQL equivalent of this series' Exception Handling guide's Section 2 custom exception hierarchy, giving calling application code (via `SqlException.Number`, exactly as this series' Deadlocks guide's Section 13 checks for error 1205) something specific and meaningful to catch and handle, rather than a generic, undifferentiated failure.

---

## 7. Temp Tables vs. Table Variables Inside a Procedure

### Temp tables (`#Table`): genuinely similar to a real table, with real statistics

```sql
CREATE TABLE #RecentOrders (OrderId INT, Total DECIMAL(10,2));
INSERT INTO #RecentOrders SELECT Id, Total FROM Orders WHERE OrderDate >= DATEADD(DAY, -7, GETDATE());
-- the optimizer maintains genuine STATISTICS on #RecentOrders, per this series'
-- Execution Plans guide's Section 7 — cardinality estimation for queries
-- AGAINST it is generally ACCURATE
```

A local temp table lives in `tempdb`, is scoped to the current session (or the specific procedure call, for a table created and used entirely within one procedure), and — critically — the optimizer maintains real statistics on it, exactly as it would for a permanent table, which generally produces well-estimated, well-optimized plans for subsequent queries against it.

### Table variables (`@Table`): lighter weight, but historically with weaker statistics

```sql
DECLARE @RecentOrders TABLE (OrderId INT, Total DECIMAL(10,2));
INSERT INTO @RecentOrders SELECT Id, Total FROM Orders WHERE OrderDate >= DATEADD(DAY, -7, GETDATE());
```

Table variables have historically carried a well-known limitation: the optimizer, in older SQL Server versions, assumed a fixed, low row-count estimate for them regardless of their actual content (no real statistics were maintained), which could produce badly mismatched plans for queries against a table variable holding genuinely many rows — directly analogous to this series' Execution Plans guide's Section 8 estimated-vs-actual divergence problem, just arising from the table variable's own design rather than stale statistics. Modern SQL Server versions (with "deferred compilation" for table variables) have substantially improved this, but it's worth knowing the historical limitation and verifying actual behavior on your specific SQL Server version via an execution plan rather than assuming either structure is unconditionally better.

### The practical guidance

```plaintext
For GENUINELY SMALL, short-lived intermediate sets: table variables are
  a reasonable, lightweight default. For LARGER sets, or ones a complex
  QUERY will be built against where estimation accuracy genuinely
  matters: temp tables, with their real statistics, are the safer choice.
```

---

## 8. Security: Permission Abstraction

### Granting EXECUTE on a procedure, without granting any direct table access at all

```sql
GRANT EXECUTE ON UpdateOrderStatus TO ApplicationRole;
-- ApplicationRole can CALL this procedure, and the procedure's OWN
-- permissions (as its OWNER, typically dbo) govern what IT can do to
-- the Orders table — WITHOUT ApplicationRole needing ANY direct
-- SELECT/UPDATE/INSERT/DELETE permission on Orders itself
```

This is the genuine, specific security benefit stored procedures provide that's worth stating precisely: via SQL Server's **ownership chaining**, a procedure's caller doesn't need direct permissions on the underlying tables at all — they only need `EXECUTE` permission on the procedure, and the procedure (owned by, say, `dbo`) carries its own authority to touch the tables it references. This lets you grant an application's database login *extremely* narrow access — "you may call these specific, named procedures, and nothing else" — rather than broad `SELECT`/`UPDATE` rights across entire tables.

### Why this is a genuinely different, additional benefit from "prevents SQL injection"

```plaintext
This is worth stating precisely because the two are OFTEN conflated:
  injection prevention (Section 9) comes from PARAMETERIZATION, which
  parameterized AD HOC queries ALSO provide — it is NOT unique to
  stored procedures. PERMISSION ABSTRACTION (this section) IS
  genuinely, structurally specific to stored procedures (and views) —
  an ad hoc parameterized query from application code still requires
  the application's database login to have DIRECT table permissions,
  which a procedure-only access model does not.
```

This distinction is worth getting precisely right, since the "stored procedures are more secure" claim is often stated vaguely, conflating two genuinely separate benefits — one (injection prevention) that parameterized ad hoc SQL provides equally well, and one (permission abstraction) that is a real, distinct advantage specific to procedures and views.

---

## 9. Stored Procedures and SQL Injection

### Parameters are the actual mechanism — not the fact that it's a "stored" procedure

```sql
-- SAFE — @Status is a genuine PARAMETER, never concatenated into SQL text
CREATE PROCEDURE GetOrdersByStatus @Status NVARCHAR(20) AS
    SELECT * FROM Orders WHERE Status = @Status;
```

```sql
-- ❌ STILL VULNERABLE — building a dynamic string INSIDE a procedure is
--    JUST as injectable as doing it in application code; being "inside
--    a stored procedure" provides NO protection on its own
CREATE PROCEDURE GetOrdersByColumn @ColumnName NVARCHAR(50), @Value NVARCHAR(50) AS
BEGIN
    DECLARE @sql NVARCHAR(MAX) = 'SELECT * FROM Orders WHERE ' + @ColumnName + ' = ''' + @Value + '''';
    EXEC(@sql);  -- @Value concatenated directly into the string — genuinely injectable
END;
```

This is worth stating as directly as possible, since it's a genuinely common misconception: a stored procedure is **not** inherently immune to SQL injection — the protection comes specifically from using *parameters* rather than string concatenation, and a procedure that builds and executes dynamic SQL internally via string concatenation (Section 11 covers when this is legitimately necessary) is exactly as vulnerable as application code doing the same thing. `sp_executesql` with its own parameterization (rather than bare `EXEC` on a concatenated string) is the correct way to build necessary dynamic SQL safely, even from within a procedure.

---

## 10. Stored Procedures vs. Dynamic SQL vs. ORM-Generated SQL

### The case for stored procedures

```plaintext
- Permission abstraction (Section 8) — a genuine, distinct security benefit
- Plan caching keyed on a STABLE name, independent of exactly how
  calling application code happens to be written
- A clear, centralized place for complex, multi-statement business
  logic that's awkward to express as pure LINQ-to-Entities
- Can be altered WITHOUT redeploying the application, for logic that
  lives entirely in the database tier
- DBA-reviewable, independent of application code review processes
```

### The case for ORM-generated SQL (this series' EF Core guide) or well-structured dynamic SQL

```plaintext
- Business logic lives in ONE place (the application, typically C#)
  rather than split across two different languages/deployment pipelines
- Easier to UNIT TEST with ordinary application-level testing tools,
  versus needing database-specific testing infrastructure
- Schema and query logic can EVOLVE TOGETHER, tracked in the SAME
  source control history and the SAME migration workflow (this series'
  EF Core guide's Section 11)
- LINQ's type safety (this series' LINQ and IEnumerable/IQueryable
  guides) catches certain classes of error at COMPILE time that T-SQL,
  evaluated only at runtime/deployment, cannot
- Avoids the operational overhead of a SEPARATE deployment/versioning
  process for database-resident code (Section 12)
```

### Why this is a genuine, legitimate architectural trade-off, not a settled debate

```plaintext
Both sides of this list are REAL, substantive advantages, not strawmen
  — the right answer depends genuinely on team structure (a dedicated
  DBA team versus a single full-stack team), the complexity and
  business-criticality of the logic involved, and organizational
  deployment practices. Many real, successful systems use BOTH
  deliberately: ORM-generated queries for ordinary CRUD, and stored
  procedures specifically for complex, multi-statement, performance-
  critical, or security-sensitive operations.
```

This is worth stating as honestly as the rest of this guide treats every other trade-off — reaching for stored procedures universally, or avoiding them universally on ORM-purity grounds, are both overcorrections; the deliberate, hybrid approach (procedures for the specific cases where their genuine advantages matter, ORM for everything else) is what most mature, pragmatic systems actually converge on.

---

## 11. Calling Stored Procedures from EF Core

### `FromSqlInterpolated`, for a procedure returning a result set EF Core can map to an entity

```csharp
var orders = await context.Orders
    .FromSqlInterpolated($"EXEC GetOrdersByCustomer @CustomerId = {customerId}")
    .ToListAsync();
```

This is precisely this series' EF Core guide's Section 13 raw-SQL mechanism, applied to a procedure call instead of an inline query — still safely, automatically parameterized (per Section 9's point about interpolation specifically), and still returning genuine, tracked (or no-tracking, per that guide's Section 8) `Order` entities.

### Output parameters and non-entity-shaped results: dropping to ADO.NET directly

```csharp
await using var connection = context.Database.GetDbConnection();
await connection.OpenAsync();
await using var command = connection.CreateCommand();
command.CommandText = "CreateOrder";
command.CommandType = CommandType.StoredProcedure;
command.Parameters.Add(new SqlParameter("@CustomerId", customerId));
var outputParam = new SqlParameter("@NewOrderId", SqlDbType.Int) { Direction = ParameterDirection.Output };
command.Parameters.Add(outputParam);
await command.ExecuteNonQueryAsync();
int newOrderId = (int)outputParam.Value;
```

For a procedure using `OUTPUT` parameters (Section 2) or producing a result shape EF Core's entity-mapping model can't directly represent, dropping to the underlying `DbConnection`/`DbCommand` (which `context.Database.GetDbConnection()` exposes) is the correct, supported approach — this operates entirely at the ADO.NET level, outside LINQ-to-Entities, but still through the same `DbContext`-managed connection, keeping it consistent with whatever transaction scope (this series' EF Core guide's Section 12) is currently active.

---

## 12. Versioning and Deployment

### Why procedures need the SAME deliberate deployment discipline as schema migrations

```plaintext
Per this series' EF Core guide's Section 11: schema changes (and,
  equally, PROCEDURE definition changes) deserve a controlled,
  reviewed, explicit deployment step — ALTERING a procedure directly on
  a production server, outside of source control and a deployment
  pipeline, creates exactly the same kind of drift-between-environments
  risk that guide warns about for ad hoc schema changes.
```

### Keeping procedure definitions in source control, alongside migrations

```plaintext
A genuinely disciplined approach versions EVERY CREATE/ALTER PROCEDURE
  script in source control, applied through the SAME migration/
  deployment pipeline as table schema changes — rather than treating
  procedures as a separately-managed, DBA-only artifact living purely
  inside the database with no corresponding tracked history.
```

This connects directly to this series' EF Core guide's migration discipline — a procedure is, after all, just another piece of the database's schema, and treating it with the same version-control and deliberate-rollout rigor is what prevents the specific, common failure mode of a procedure silently diverging between a developer's local database, a staging environment, and production.

---

## 13. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Forgetting `OUTPUT` at the call site, even though the procedure declares it | The output value is silently discarded — no error is raised | Always specify `OUTPUT` both in the procedure definition and at every call site (Section 2) |
| Using `RETURN` to try to pass back a genuine data value | `RETURN` only supports a single integer; anything else is silently truncated or produces unexpected behavior | Use `OUTPUT` parameters or a result set for actual data; reserve `RETURN` for integer status codes (Section 3) |
| Assuming a stored procedure's performance is stable regardless of the first parameter value it was called with | Plan caching means the first call's plan is reused for every subsequent call, including ones with very different selectivity | Recognize parameter sniffing (Section 4-5) as the likely cause of input-dependent slowness, and apply the appropriately-scoped recompilation fix |
| Building dynamic SQL inside a procedure via string concatenation | Just as injectable as doing the same thing in application code — "it's a stored procedure" provides no protection by itself | Use `sp_executesql` with genuine parameters for any dynamic SQL that's truly necessary (Section 9) |
| Assuming stored procedures are inherently more secure against injection than parameterized application code | Injection protection comes from parameterization specifically, which ad hoc parameterized queries provide equally well | Credit procedures for their genuine, distinct advantage — permission abstraction — not injection prevention, which isn't unique to them (Section 8-9) |
| Treating table variables as a universal, lightweight substitute for temp tables | Historically weaker cardinality estimation can produce badly mismatched plans for larger result sets | Use temp tables when query-plan estimation accuracy against the intermediate set genuinely matters (Section 7) |
| Editing a stored procedure directly in production, outside source control | Creates schema/procedure drift between environments, identical to the ad hoc schema change risk this series' EF Core guide warns against | Version every procedure definition in source control, deployed through the same controlled pipeline as schema migrations (Section 12) |
| Picking a side in the stored-procedures-vs-ORM debate dogmatically, for every situation | Both approaches have genuine, substantive advantages suited to different cases; neither is universally correct | Use each deliberately — ORM for ordinary CRUD, procedures for complex/security-sensitive/performance-critical operations (Section 10) |

---

## Quick Reference Table

| Concept | Syntax | Purpose |
|---|---|---|
| Define a procedure | `CREATE PROCEDURE Name @Param TYPE AS BEGIN ... END` | A named, parameterized, persisted batch of T-SQL |
| Output parameter | `@Param TYPE OUTPUT` (both sides) | Returns a scalar value back to the caller |
| Status code | `RETURN n;` | A single integer, conventionally success/failure |
| One-time recompile | `EXEC sp_recompile 'Name';` | Forces a fresh plan on the next call only |
| Always recompile | `WITH RECOMPILE` on the procedure | No plan caching at all for this procedure |
| Recompile one statement | `OPTION (RECOMPILE)` | Fixes parameter sniffing for a single query within a larger procedure |
| Structured error handling | `BEGIN TRY ... END TRY BEGIN CATCH ... THROW; END CATCH` | Mirrors this series' Exception Handling guide's discipline in T-SQL |
| Custom error | `THROW 50001, 'message', 1;` | A meaningful, application-specific error number and message |
| Permission abstraction | `GRANT EXECUTE ON Name TO Role;` | Callers need no direct table access, only execute rights |
| EF Core result-mapped call | `.FromSqlInterpolated($"EXEC Name @P = {value}")` | Maps a procedure's result set to tracked/no-tracking entities |

---

## Conclusion

A stored procedure's real value rests on two genuinely distinct pillars this guide treats as precisely as this series treats everything else: plan caching that makes repeated calls fast (at the direct, well-understood cost of parameter-sniffing risk, exactly this series' Execution Plans guide's Section 9 concern applied to its most common real-world context), and permission abstraction that lets an application's database access be scoped to "may call these specific, named operations" rather than broad table-level rights — a genuine security benefit distinct from, and not a substitute for, the parameterization that actually prevents SQL injection regardless of whether that parameterization happens inside a procedure or an ad hoc query.

The stored-procedures-versus-ORM question deserves the honest, two-sided treatment this guide gives it, because both positions rest on real, substantive engineering trade-offs rather than one side simply being outdated or misguided — and the practical resolution most mature systems reach, using each deliberately where its own specific strengths matter most, is itself the most useful takeaway this guide offers: understanding procedures this precisely is what lets you make that choice as a genuine architectural decision, exactly the way this whole series treats every other trade-off it covers.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the procedure-was-blazing-fast-for-months-then-suddenly-slow-for-no-code-change-reason story that made the plan-caching-meets-parameter-sniffing connection click far better than any isolated explanation of either concept alone.*
