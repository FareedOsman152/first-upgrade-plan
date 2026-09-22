# Upgrade Plan — C# / .NET / Databases / Architecture

> A personal study plan. No time estimates — you set those.
> The one rule that doesn't get broken: **after every topic, go `grep` your own repo and find where it lives.**
> Study that never touches your code doesn't stick.

---

## Part 1 — Topics

### Scope 1 · C# in depth

#### 1.1 Type system & memory
- Stack vs heap; value vs reference semantics
- Boxing — where it happens silently (interfaces on structs, LINQ over value types)
- `struct` / `readonly struct` / `ref struct`; defensive copies
- Nullable value types vs nullable reference types; what `!` actually does (nothing at runtime)
- **Equality**: `==` vs `.Equals` vs `IEquatable<T>`; the `GetHashCode` contract; what breaks when you override one without the other
  - 🔎 In your code: `BaseEntity` implements equality on `Id`, so the `HashSet<Property>` in `AddSelectedPropertiesRequest` deduplicates nothing

#### 1.2 Delegates, lambdas, closures
- Delegates as types; multicast; the invocation list
- `Func` / `Action` / `Predicate`
- Closures: capture by reference not by value; the loop-variable trap; the display class and its cost
- **Expression trees vs delegates** — why EF's `Where(x => ...)` is an `Expression`, not a `Func`
  - 🔎 In your code: the commit `project plan activation fix the query, projection now in memory`

#### 1.3 Generics & variance
- Constraints: `class` / `struct` / `new()` / `notnull`
- Covariance and contravariance (`out` / `in`) — why `IEnumerable<T>` is covariant and `List<T>` isn't
- Type inference; open vs closed generics
- How the DI container resolves `IEnumerable<IHandler>`
  - 🔎 In your code: the marker interfaces in `DataSynchronizationWithDari`

#### 1.4 Records, tuples, pattern matching
- `record` vs `record struct`; generated value equality; `with`; deconstruction
- Why records are excellent for requests/DTOs and dangerous for entities
- `ValueTuple` vs `Tuple`; named elements
- Property patterns, `is { Count: > 0 }`, switch expressions, exhaustiveness

#### 1.5 Operators & conversion semantics
- Precedence traps: `??` against `+` and against `?:`
  - 🔎 In your code: `DescriptionEn ?? "" + "\n\n" + DescriptionAr` — the Arabic description is silently dropped
- Compound assignment semantics and the implicit conversions inside them
- `implicit` / `explicit` operator overloading
  - 🔎 In your code: this is what makes `return errors;` compile in your `Result` type

#### 1.6 Collections & LINQ semantics
- Deferred vs immediate execution; the multiple-enumeration problem
- `IEnumerable` vs `IQueryable`
- `First` / `FirstOrDefault` / `Single` / `SingleOrDefault` — what each one means and when each throws
- `Except` / `ExceptBy` / `Distinct` / `DistinctBy` — key selectors and what distinctness is actually guaranteed
- Complexity of `List` / `HashSet` / `Dictionary` and when to reach for each
  - 🔎 In your code: the `FirstOrDefault(...).X` pattern, recurring from August 2025 to today

#### 1.7 Exceptions
- The cost of an exception; when to throw vs when to return a `Result`
- `using` / `IDisposable` / `IAsyncDisposable`; dispose ordering
- Exception filters (`when`)
- Exceptions in `async void` and unobserved tasks

---

### Scope 2 · Async & Concurrency

#### 2.1 Async fundamentals
- What `async/await` compiles into — a state machine, not a thread
- `Task` vs `ValueTask`, and when `ValueTask` actually matters
- `SynchronizationContext`; why `ConfigureAwait(false)` exists and why it's a no-op in ASP.NET Core
  - 🔎 In your code: `.IC()` is everywhere — know exactly what it buys you
- Sync-over-async deadlocks (`.Result` / `.Wait()`)
- `async void`; and what an `async` method with no `await` actually costs

#### 2.2 Task composition
- `WhenAll` / `WhenAny`; what happens to the second exception
- `CancellationToken`: cooperative cancellation, linked tokens, and `CancellationToken.None` as a deliberate choice
- `TaskCompletionSource`
- `Task.Run` — when offloading is right and when it's wrong in a web app
  - 🔎 In your code: the `Task.WhenAll` in `InitiatePropertyBookingRegistrationRequest` and in `AddApplicationParties`

#### 2.3 Concurrency hazards
- `DbContext` is not thread-safe — parallel repository calls on a scoped context
- Race conditions: lost update, write skew, check-then-act
- Optimistic concurrency (rowversion) vs pessimistic (`FOR UPDATE`)
- Idempotency keys; at-least-once delivery; why exactly-once is a lie
- Compensating actions and the **outbox pattern**
  - 🔎 In your code: the manual/auto race in Outflow, and the document-delete race in Inflow

#### 2.4 Background work
- `IHostedService` / `BackgroundService`
- Hangfire: how jobs are serialized, retry behaviour, and what makes a job resumable and idempotent
- Continuations instead of one large job
  - 🔎 In your code: `OutflowAdrecFeesAutoApplicationsService`

---

### Scope 3 · Database

#### 3.1 Transactions & isolation
- ACID; isolation levels; what InnoDB defaults to
- The anomalies: dirty read, non-repeatable read, phantom, lost update, write skew
- Transaction duration and why long ones hurt
- Coordinating work across a database and an external API
  - 🔎 In your code: `ClearSheetData` deletes a document from DARI inside an operation with no transaction around it

#### 3.2 Locking
- Row locks, gap locks, next-key locks (InnoDB-specific)
- `SELECT ... FOR UPDATE` / `FOR SHARE`
- Deadlocks: how they arise, how the engine resolves them, and how operation ordering prevents them

#### 3.3 Indexing & query performance
- B-tree structure; clustered vs secondary indexes in InnoDB
- Composite index column order and the leftmost-prefix rule
- Covering indexes
- `EXPLAIN` — how to read it: `type`, `key`, `rows`, `Extra` (filesort / temporary)
- Selectivity and cardinality; when an index is ignored

#### 3.4 Schema & integrity
- Normalization, and deliberate denormalization
- Constraints as correctness: unique, foreign key, check — pushing invariants down into the database
- FK delete behaviours and what each one means
- Surrogate vs natural keys
  - 🔎 In your code: `Property` carries both `Id` (surrogate) and `DariPropertyId` (natural), and the two get conflated
- **Migrations against live data**: additive → backfill → switch, and why a destructive migration never happens in one step

#### 3.5 EF Core
- The change tracker: how it works, `AsNoTracking`, tracked vs untracked entities
- `ExecuteUpdate` / `ExecuteDelete` bypass the tracker — and what that implies
- Query translation: client vs server evaluation
- `Include` / `ThenInclude`, split queries, the N+1 problem
- `SaveChanges` boundaries and its transaction behaviour
  - 🔎 In your code: `OffPlanContractRepository` calls `SaveChanges` from inside a repository
- Concurrency tokens, owned types, value converters

---

### Scope 4 · .NET Infrastructure

> The one scope that cannot be studied without building a project from scratch.

#### 4.1 Hosting & startup
- The Generic Host and `WebApplicationBuilder`; the startup sequence
- `IHostedService` lifecycle; graceful shutdown

#### 4.2 Dependency Injection
- The three lifetimes; the captive dependency problem
- Registering open generics; `IEnumerable<T>` resolution; keyed services
- Validation on build

#### 4.3 Configuration & Options
- Configuration providers and precedence (appsettings, environment variables, user secrets)
- `IOptions` vs `IOptionsSnapshot` vs `IOptionsMonitor`
- Options validation

#### 4.4 Middleware pipeline
- The standard order, and why it is that order
- Writing middleware; short-circuiting
- Exception-handling middleware and `ProblemDetails`

#### 4.5 AuthN / AuthZ
- Authentication schemes; JWT validation parameters (issuer, audience, lifetime, signing key)
- `ClaimsPrincipal` and claims
- Policies, requirements, handlers — **build these yourself**
  - 🔎 In your code: `ApplicationAccessHandler` and the `application-read` policy — you use them without having built them
- What a `DelegatingHandler` does
  - 🔎 In your code: there are two in `src/Infrastructure/DariServices`

#### 4.6 Cross-cutting
- `ILogger`, structured logging, scopes, providers; wiring Serilog
- `IHttpClientFactory`: named and typed clients, delegating handlers, resilience (Polly)
- Health checks
- Model binding, validation, filters; controllers vs minimal APIs

---

### Scope 5 · Testing

> Not a standalone scope — it attaches to the project from day one.

#### 5.1 Foundations
- Test types: unit / integration / contract / end-to-end — what each one buys
- Arrange-Act-Assert; one behaviour per test; naming
- What is worth testing (business rules, boundaries) and what isn't

#### 5.2 Unit testing
- xUnit: `Fact` / `Theory` / `InlineData` / `MemberData`; fixtures; collection fixtures
- FluentAssertions
- Test doubles: stub vs mock vs fake (NSubstitute / Moq)
- Designing for testability without wrecking the design

#### 5.3 Integration testing
- `WebApplicationFactory` and `TestServer`
- A real database vs in-memory vs Testcontainers — and why EF in-memory lies to you
- Test data setup and teardown; deterministic tests
- Faking external systems: `HttpMessageHandler` stubs, WireMock
  - 🔎 In your code: `IntegrationTests/Bootstrapping/` already exists — read it before you start

#### 5.4 Practical
- Testing time (`ITimeProvider` — you already have it)
- Testing background jobs
- What a regression test is for: lock the bug, not the code

---

### Scope 6 · Architecture

#### 6.1 Consolidate what you already do
- Layered / clean architecture; dependency direction; why Core doesn't reference Infrastructure
- CQRS with MediatR — what it actually is, and what it isn't
- Aggregate boundaries; why your entities are anemic and what a rich domain model looks like

#### 6.2 Genuinely different styles
- Event-driven: brokers, pub/sub, at-least-once delivery, ordering, idempotent consumers
- **Outbox pattern** ← directly applicable to `DataSynchronizationWithDari`
- **Saga / process manager** for long-running workflows ← directly applicable to `ApplicationWorkflowBlueprint`
- Caching and invalidation strategies
- API design: versioning, idempotent endpoints

#### 6.3 Distributed systems basics
- Failure modes: timeouts, partial failure, retry storms
- Retry + backoff + jitter; circuit breaker
- At-most-once vs at-least-once vs exactly-once
- Strong vs eventual consistency
  - 🔎 In your code: the TAS↔DARI link is eventually consistent whether you designed for it or not

---

### Outside the scopes · Design Patterns
**Skim, not a scope.** You already apply factory, specification, strategy and marker-interface fan-out correctly. Read the names, map them onto what you already do, and move on. A weekend.

---
---

## Part 2 — How the plan runs

### The principle: two tracks in parallel, not one scope at a time

**One scope at a time, alone** → knowledge that is never applied decays. You'd finish C# and have forgotten half of it before reaching infrastructure.

**Everything mixed** → context switching, and you end up with more of the breadth you already have and none of the depth.

The answer: **one deep scope at a time, plus an applied track running continuously.**

```
Track 1 (deep)     : one scope at a time, in order
Track 2 (applied)  : the from-scratch project — running from day one to the last day
```

The applied track is what makes the deep track stick. Everything you learn goes into it.

---

### Order of the deep track

| # | Scope | Why here |
|---|---|---|
| 1 | **C#** | Everything after it stands on it. You can't understand the EF change tracker without reference semantics, or async without closures |
| 2 | **Database** | Pays off in your day job **immediately**. You write code that moves money through MySQL every day |
| 3 | **Async & Concurrency** | Needs C# done (state machines, closures) and needs the database (transactions, locking) to make sense of race conditions |
| 4 | **.NET Infrastructure** | Where the project gets serious. Currently your weakest area |
| 5 | **Architecture** | Last. It won't pay off until everything beneath it is solid — otherwise it's just vocabulary |

**Testing** is not a numbered item — it rides along with the project from day one and grows with every scope.

---

### The applied track: the project

**One** project that grows with you, not a new project per scope.

| Stage | What the project becomes |
|---|---|
| With C# | A console app or small library — where you exercise equality, closures and generics by hand |
| With Database | A simple Web API + EF + a real MySQL. Migrations, indexes, `EXPLAIN` on your own queries |
| With Async | A background job, a call to an external API, and a small outbox |
| With Infrastructure | **Rebuilt from empty, by hand, no template**: JWT auth, a middleware pipeline you can explain in order, DI, options, logging, health checks |
| With Architecture | Redesign a piece of TAS (the data sync, for example) in a different style, and write down why |

---

### Operating rules

1. **The grep rule** — after every topic, find it in TAS. The 🔎 markers above are your starting points.
2. **The explain rule** — a topic isn't finished until you can explain it to someone without opening anything.
3. **The writing rule** — five lines in your own words per topic. Not copied.
4. **Documentation is a reference, not a curriculum** — go to it when you need it; don't read it front to back.
5. **Never two deep scopes at once.** One.

---

### Checkpoints (critical — without these the plan fails)

This plan has no natural feedback loop. You have to build one deliberately — exactly like you did in TAS when the senior moved on.

**After each scope — something concrete, not a feeling:**

| After | The checkpoint |
|---|---|
| C# | Explain why the `HashSet<Property>` in Inflow deduplicates nothing, and fix it correctly |
| Database | Take the slowest query in TAS, read its `EXPLAIN`, add an index, and prove the difference with numbers |
| Async | Fix the manual/auto race in Outflow, and write an outbox for the data sync |
| Infrastructure | The project runs with real auth, and you can explain the middleware order and the reason for it |
| Architecture | Redesign a TAS subsystem in a different style, with the **why** and the **trade-offs** written down |

**External checkpoints — outside your own bubble:**
- An interview you don't need. Not to take a job — to find out where you actually stand.
- Or a contribution to an open-source .NET repo, so someone who doesn't know you reviews your code.
- Or rebuild part of TAS from scratch outside the repo, and compare.

---

### If time gets tight (the minimum)

If work consumes you and only one thing fits, take them in this order:

1. **Database: transactions + isolation + locking** — your most dangerous gap right now, and you work on money.
2. **C#: equality + closures + LINQ semantics** — these produce real bugs in your code today.
3. **Testing: write two tests** against the existing fixture in `IntegrationTests/`.
4. **Infrastructure: the from-scratch project.**

Everything else waits.
