# Upgrade Plan — C# / .NET / Databases / Architecture

> A personal study plan. No time estimates — you set those.
> The one rule that doesn't get broken: **after every topic, go `grep` your own repo and find where it lives.**
> Study that never touches your code doesn't stick.

---

## How deep is deep — the marker on every line

**Go deep where being wrong is silent. Skim where the compiler stops you.**

| Mark | Means | Test |
|---|---|---|
| 🔴 | **Dive.** Read it properly, write the code, break it on purpose | Getting it wrong compiles, runs, and produces **wrong data or wrong behaviour** — or it's a decision you make every week |
| ⚪ | **General knowledge.** Know it exists, know when it would matter, move on | Getting it wrong is a **compiler error**, or it's a performance concern in code whose cost is dominated by a database round-trip and an HTTP call |

A ⚪ line is one paragraph and one sentence written in your own words. That's it. Roughly a third of this plan is ⚪ — that third is where your time comes from.

**Two exceptions, where ⚪ is still wrong:** anything touching **money** and anything touching **security**. There you go deep even when the failure is loud, because a loud failure at the wrong moment is still a failed payment.

---

## Part 1 — Topics

### Scope 1 · C# in depth

#### 1.1 Type system & memory
- 🔴 Stack vs heap; value vs reference semantics — aliasing is the silent one: two names, one object
- ⚪ Boxing — where it happens silently (interfaces on structs, LINQ over value types)
- ⚪ `struct` / `readonly struct` / `ref struct`; defensive copies — know that `ref struct` is stack-only and why `Span<T>` needs it; you won't write one
- 🔴 Nullable value types vs nullable reference types; what `!` actually does (nothing at runtime) — a `!` over a genuinely null value is silent until it isn't
- 🔴 **Equality**: `==` vs `.Equals` vs `IEquatable<T>`; the `GetHashCode` contract; what breaks when you override one without the other
  - 🔎 In your code: `BaseEntity<TId>` overrides **neither** `Equals` nor `GetHashCode`, and its constructor assigns a fresh `Guid.CreateVersion7()` to every instance — so the `HashSet<Domain.Property>` in `AddSelectedPropertiesRequest` falls back to reference equality and deduplicates nothing
  - 🔎 Also in your code: `Entity` in the same file (`Domain/Common/Contracts/BaseEntity.cs`) is a full textbook implementation — `Equals` + `GetHashCode` + `==`/`!=` + the `IsTransient()` guard. It is **dead code**, nothing derives from it. Read it as the reference implementation, and work out why the transient guard has to be there

#### 1.2 Delegates, lambdas, closures
- ⚪ Delegates as types; multicast; the invocation list
- ⚪ `Func` / `Action` / `Predicate`
- 🔴 Closures: capture by reference not by value; the loop-variable trap (the display class and its cost is ⚪ — the capture semantics are not)
- 🔴 **Expression trees vs delegates** — why EF's `Where(x => ...)` is an `Expression`, not a `Func`
  - 🔎 In your code: the commit `project plan activation fix the query, projection now in memory`

#### 1.3 Generics & variance
- 🔴 Constraints: `class` / `struct` / `new()` / `notnull` — you write generic handlers and repositories, so you choose these
- ⚪ Covariance and contravariance (`out` / `in`) — enough to know why `IEnumerable<T>` is covariant and `List<T>` isn't. Getting it wrong doesn't compile
- ⚪ Type inference; open vs closed generics
- 🔴 How the DI container resolves `IEnumerable<IHandler>` — you depend on this, and a handler that isn't registered fails as "nothing happened"
  - 🔎 In your code: the marker interfaces in `DataSynchronizationWithDari`

#### 1.4 Records, tuples, pattern matching
- 🔴 `record` vs `record struct`; generated value equality; `with`; deconstruction — value equality is silent when you didn't want it
- 🔴 Why records are excellent for requests/DTOs and dangerous for entities
- ⚪ `ValueTuple` vs `Tuple`; named elements
- ⚪ Property patterns, `is { Count: > 0 }`, switch expressions — syntax. **Except exhaustiveness**, which is 🔴: a switch expression that silently falls through to an unhandled case is a runtime surprise

#### 1.5 Operators & conversion semantics
- 🔴 Precedence traps: `??` against `+` and against `?:`
  - 🔎 In your code: `DescriptionEn ?? "" + "\n\n" + DescriptionAr` — the Arabic description is silently dropped
- ⚪ Compound assignment semantics and the implicit conversions inside them
- 🔴 `implicit` / `explicit` operator overloading — an implicit conversion is invisible at the call site, which is exactly why it needs to be deliberate
  - 🔎 In your code: this is what makes `return errors;` compile in your `Result` type

#### 1.6 Collections & LINQ semantics
> The whole topic is 🔴 except the last line. Every item here fails silently, and every one of them has already produced a bug in your code.
- 🔴 Deferred vs immediate execution; the multiple-enumeration problem
- 🔴 `IEnumerable` vs `IQueryable` — the moment a query stops being a query and starts being a loop over the whole table
- 🔴 `First` / `FirstOrDefault` / `Single` / `SingleOrDefault` — what each one means and when each throws
- 🔴 `Except` / `ExceptBy` / `Distinct` / `DistinctBy` — key selectors and what distinctness is actually guaranteed
- ⚪ Complexity of `List` / `HashSet` / `Dictionary` and when to reach for each
  - 🔎 In your code: the `FirstOrDefault(...).X` pattern, recurring from August 2025 to today

#### 1.7 Exceptions
- 🔴 When to throw vs when to return a `Result` — a decision you make in every handler (the *cost* of an exception is ⚪)
- ⚪ `using` / `IDisposable` / `IAsyncDisposable`; dispose ordering
- ⚪ Exception filters (`when`)
- 🔴 Exceptions in `async void` and unobserved tasks — these disappear, and a swallowed failure in a background job is invisible until the data is wrong

#### 1.8 OOP — inheritance & polymorphism
> Numbered last, studied second. It sits on 1.1 (reference semantics) and everything else in this scope leans on it.

**Inheritance mechanics**
- ⚪ Base and derived; what a derived object actually is in memory
- 🔴 Constructor chaining: `base(...)`, and the exact order of field initializers vs constructor bodies
- 🔴 **Calling a virtual method from a constructor** — the classic trap: the override runs before the derived fields are initialized
- 🔴 Access modifiers as a design decision: `private` / `protected` / `internal` / `private protected` — what each one says about who is allowed to extend you
- ⚪ `sealed`, and why "sealed until there's a reason" is a defensible default

**Polymorphism**
- 🔴 `virtual` / `override` / `abstract` / `new` — and what each does **at the call site**, not in the declaration
- 🔴 Method hiding (`new`) vs overriding: the same call, two different methods, decided by the *declared* type — the definition of a silent bug
- ⚪ Virtual dispatch: how the runtime picks the implementation, and what it costs
- 🔴 Overload resolution (compile time) vs override resolution (run time) — the difference that explains most "why did it call that one" bugs
- 🔴 The `object` virtuals: `ToString` / `Equals` / `GetHashCode` — overriding them is polymorphism, and 1.1 is where it bites

**Abstract classes vs interfaces**
- 🔴 What only an abstract class can do (state, constructors, protected members, a partial implementation)
- 🔴 What only an interface can do (multiple, variance, structural role)
- ⚪ Explicit interface implementation, and why you'd hide a member from the class's own API
- ⚪ Default interface methods — what they actually solve, and why they are not "abstract classes now"

**Design consequences — the part that matters more than the syntax. All 🔴.**
- 🔴 **Template method**: a base class that defines the flow and leaves holes for the derived class to fill
  - 🔎 In your code: `ApplicationActionService` is exactly this — `protected virtual Execute`, `ExecuteCancel`, `ApplicationIsComplete`, `CheckAgainstClosedApp` are the holes, and every service's action service fills the ones it cares about
- 🔴 **Liskov substitution** — the L you already know, stated concretely: a derived class that throws where the base returns, or narrows what the base accepted, breaks every caller that holds a base reference
- 🔴 **The fragile base class problem**: adding a method to a base class can break derived classes that never changed
- 🔴 **Composition over inheritance** — when a base class is the wrong tool and a dependency is the right one
- 🔴 Inheritance for **reuse** vs inheritance for **substitution** — only the second one is what inheritance is for
  - 🔎 In your code: `BaseEntity` / `BaseAuditableEntity` is reuse; `CustomException` → `NotFoundException` / `ConflictException` / `ForbiddenException` is substitution; `BaseOffPlanContractEventHandler` is both, and worth reading with this distinction in hand

---

### Scope 2 · Async & Concurrency

#### 2.1 Async fundamentals
- 🔴 What `async/await` compiles into — a state machine, not a thread
- ⚪ `Task` vs `ValueTask`, and when `ValueTask` actually matters (it doesn't, in your code)
- 🔴 `SynchronizationContext`; why `ConfigureAwait(false)` exists and why it's a no-op in ASP.NET Core
  - 🔎 In your code: `.IC()` is everywhere — know exactly what it buys you
- 🔴 Sync-over-async deadlocks (`.Result` / `.Wait()`) — the failure is a hang, with no exception and no log line
- ⚪ `async void`; and what an `async` method with no `await` actually costs — one rule to memorize, not a dive

#### 2.2 Task composition
- 🔴 `WhenAll` / `WhenAny`; what happens to the second exception — you lose failures here and never see them
- 🔴 `CancellationToken`: cooperative cancellation, linked tokens, and `CancellationToken.None` as a deliberate choice
- ⚪ `TaskCompletionSource`
- ⚪ `Task.Run` — know the one rule: don't offload in a web app to "make it faster"
  - 🔎 In your code: the `Task.WhenAll` in `InitiatePropertyBookingRegistrationRequest` and in `AddApplicationParties`

#### 2.3 Concurrency hazards
> All 🔴, no exceptions. This is the money path, and every failure mode here is data that is quietly wrong.
- 🔴 `DbContext` is not thread-safe — parallel repository calls on a scoped context
- 🔴 Race conditions: lost update, write skew, check-then-act
- 🔴 Optimistic concurrency (rowversion) vs pessimistic (`FOR UPDATE`)
- 🔴 Idempotency keys; at-least-once delivery; why exactly-once is a lie
- 🔴 Compensating actions and the **outbox pattern**
  - 🔎 In your code: the manual/auto race in Outflow, and the document-delete race in Inflow

#### 2.4 Background work
- ⚪ `IHostedService` / `BackgroundService`
- 🔴 Hangfire: how jobs are serialized, retry behaviour, and what makes a job resumable and idempotent — a job that half-ran is your known open item
- 🔴 Continuations instead of one large job
  - 🔎 In your code: `OutflowAdrecFeesAutoApplicationsService`

---

### Scope 3 · Database

> The scope with the fewest ⚪ lines in the plan, and that's deliberate: it's your largest gap, you work on money, and a database gets almost everything wrong **silently**.

#### 3.1 Transactions & isolation
- 🔴 ACID; isolation levels; what InnoDB defaults to
- 🔴 The anomalies: dirty read, non-repeatable read, phantom, lost update, write skew
- 🔴 Transaction duration and why long ones hurt
- 🔴 Coordinating work across a database and an external API
  - 🔎 In your code: `ClearSheetData` deletes a document from DARI inside an operation with no transaction around it

#### 3.2 Locking
- 🔴 Row locks, gap locks, next-key locks (InnoDB-specific)
- 🔴 `SELECT ... FOR UPDATE` / `FOR SHARE`
- 🔴 Deadlocks: how they arise, how the engine resolves them, and how operation ordering prevents them

#### 3.3 Indexing & query performance
- ⚪ B-tree structure — the shape, not the internals. **But clustered vs secondary indexes in InnoDB is 🔴**: it decides what every secondary index costs you
- 🔴 Composite index column order and the leftmost-prefix rule
- 🔴 Covering indexes
- 🔴 `EXPLAIN` — how to read it: `type`, `key`, `rows`, `Extra` (filesort / temporary)
- 🔴 Selectivity and cardinality; when an index is ignored

#### 3.4 Schema & integrity
- ⚪ Normalization, and deliberate denormalization — you already do this by instinct
- 🔴 Constraints as correctness: unique, foreign key, check — pushing invariants down into the database
- 🔴 FK delete behaviours and what each one means
- 🔴 Surrogate vs natural keys
  - 🔎 In your code: `Property` carries both `Id` (surrogate) and `DariPropertyId` (natural), and the two get conflated
- 🔴 **Migrations against live data**: additive → backfill → switch, and why a destructive migration never happens in one step

#### 3.5 EF Core
- 🔴 The change tracker: how it works, `AsNoTracking`, tracked vs untracked entities
- 🔴 `ExecuteUpdate` / `ExecuteDelete` bypass the tracker — and what that implies
- 🔴 Query translation: client vs server evaluation
- 🔴 `Include` / `ThenInclude`, split queries, the N+1 problem
- 🔴 `SaveChanges` boundaries and its transaction behaviour
  - 🔎 In your code: `OffPlanContractRepository` calls `SaveChanges` from inside a repository
- ⚪ Concurrency tokens, owned types, value converters — know they exist; reach for them when a case turns up

---

### Scope 4 · .NET Infrastructure

> The one scope that cannot be studied without building a project from scratch. Most of it is ⚪ **to read** and 🔴 **to build** — the marks below are about reading.

#### 4.1 Hosting & startup
- ⚪ The Generic Host and `WebApplicationBuilder`; the startup sequence
- 🔴 `IHostedService` lifecycle; graceful shutdown — you run background work, and a job killed mid-write is your problem

#### 4.2 Dependency Injection
- 🔴 The three lifetimes; the captive dependency problem — a singleton holding a scoped `DbContext` is a silent, intermittent disaster
- ⚪ Registering open generics; `IEnumerable<T>` resolution; keyed services
- ⚪ Validation on build — turn it on, that's the whole lesson

#### 4.3 Configuration & Options
- 🔴 Configuration providers and precedence (appsettings, environment variables, user secrets) — "it works locally, not in QA" lives here
- 🔴 `IOptions` vs `IOptionsSnapshot` vs `IOptionsMonitor` — picking the wrong one means a config change that never takes effect
- ⚪ Options validation

#### 4.4 Middleware pipeline
- 🔴 The standard order, and why it is that order — a misordered pipeline authenticates after authorizing, and nothing errors
- ⚪ Writing middleware; short-circuiting
- ⚪ Exception-handling middleware and `ProblemDetails`

#### 4.5 AuthN / AuthZ
> Security. The ⚪/🔴 filter is suspended here — go deep on all of it.
- 🔴 Authentication schemes; JWT validation parameters (issuer, audience, lifetime, signing key) — a validation parameter left off accepts tokens it shouldn't, with no error anywhere
- 🔴 `ClaimsPrincipal` and claims
- 🔴 Policies, requirements, handlers — **build these yourself**
  - 🔎 In your code: `ApplicationAccessHandler` and the `application-read` policy — you use them without having built them, and that handler **fails open**
- ⚪ What a `DelegatingHandler` does
  - 🔎 In your code: there are two in `src/Infrastructure/DariServices`

#### 4.6 Cross-cutting
- ⚪ `ILogger`, structured logging, scopes, providers; wiring Serilog
- 🔴 `IHttpClientFactory`: named and typed clients, delegating handlers, resilience (Polly) — socket exhaustion and a missing retry policy both look like "DARI is slow today"
- ⚪ Health checks
- ⚪ Model binding, validation, filters; controllers vs minimal APIs

---

### Scope 5 · Testing

> Not a standalone scope — it attaches to the project from day one.

#### 5.1 Foundations
- ⚪ Test types: unit / integration / contract / end-to-end — what each one buys
- ⚪ Arrange-Act-Assert; one behaviour per test; naming
- 🔴 What is worth testing (business rules, boundaries) and what isn't — the judgement call that decides whether a test suite is an asset or a tax

#### 5.2 Unit testing
- ⚪ xUnit: `Fact` / `Theory` / `InlineData` / `MemberData`; fixtures; collection fixtures
- ⚪ FluentAssertions
- 🔴 Test doubles: stub vs mock vs fake (NSubstitute / Moq) — mock the wrong thing and the test passes forever regardless of the code
- 🔴 Designing for testability without wrecking the design

#### 5.3 Integration testing
- ⚪ `WebApplicationFactory` and `TestServer`
- 🔴 A real database vs in-memory vs Testcontainers — and why EF in-memory lies to you
- 🔴 Test data setup and teardown; deterministic tests — a flaky test is worse than no test, because it trains you to ignore red
- ⚪ Faking external systems: `HttpMessageHandler` stubs, WireMock
  - 🔎 In your code: `IntegrationTests/Bootstrapping/` already exists — read it before you start

#### 5.4 Practical
- ⚪ Testing time (`ITimeProvider` — you already have it)
- 🔴 Testing background jobs
- 🔴 What a regression test is for: lock the bug, not the code

---

### Scope 6 · Architecture

#### 6.1 Consolidate what you already do
- ⚪ Layered / clean architecture; dependency direction; why Core doesn't reference Infrastructure — you already do this correctly; this is putting names on it
- 🔴 CQRS with MediatR — what it actually is, and what it isn't
- 🔴 Aggregate boundaries; why your entities are anemic and what a rich domain model looks like

#### 6.2 Genuinely different styles
- 🔴 Event-driven: brokers, pub/sub, at-least-once delivery, ordering, idempotent consumers
- 🔴 **Outbox pattern** ← directly applicable to `DataSynchronizationWithDari`
- 🔴 **Saga / process manager** for long-running workflows ← directly applicable to `ApplicationWorkflowBlueprint`
- ⚪ Caching and invalidation strategies — until you have a measured cache problem, this is vocabulary
- 🔴 API design: versioning, and **idempotent endpoints** — the second one you need now, not later

#### 6.3 Distributed systems basics
> All 🔴. You already run a distributed system across TAS, DARI, ELMS and Aurora; you just haven't called it that.
- 🔴 Failure modes: timeouts, partial failure, retry storms
- 🔴 Retry + backoff + jitter; circuit breaker
- 🔴 At-most-once vs at-least-once vs exactly-once
- 🔴 Strong vs eventual consistency
  - 🔎 In your code: the TAS↔DARI link is eventually consistent whether you designed for it or not

---

### Outside the scopes · Design Patterns
⚪ **Skim, not a scope.** You already apply factory, specification, strategy and marker-interface fan-out correctly. Read the names, map them onto what you already do, and move on. A weekend.

---
---

## Part 2 — How the plan runs

### The principle: two tracks, one sequence

**One scope at a time, alone** → knowledge that is never applied decays. You'd finish C# and have forgotten half of it before reaching infrastructure.

**Everything mixed** → context switching, and you end up with more of the breadth you already have and none of the depth.

The answer: **one deep topic at a time, in a fixed order, plus one project that grows underneath the whole thing.**

```
Track 1 (deep)     : the numbered sequence below — one topic at a time, in order
Track 2 (applied)  : one project, from day one to the last day, growing with each block
```

The sequence below is the plan. Not "scope by scope" — **topic by topic**, with the project move that goes with each one. Testing is not a block at the end; it enters at step 6 and never leaves.

---

### The sequence

Five blocks. Inside a block the order matters. Between blocks it matters more.

#### Block A — C# · the project is a console app / small library

| # | Topic | The project move |
|---|---|---|
| 1 | **1.1** Type system & memory | Write a value type and a reference type that behave differently on purpose. Override `Equals`/`GetHashCode` and put both in a `HashSet` |
| 2 | **1.8** OOP — inheritance & polymorphism | Build a small template-method base class with two derived classes. Then write the `new` vs `override` pair and call both through a base reference. Then call a virtual method from a constructor and watch it read an uninitialized field |
| 3 | **1.2** Delegates, lambdas, closures | Write the loop-variable capture bug on purpose, then fix it. Build one `Expression<Func<T,bool>>` by hand |
| 4 | **1.6** Collections & LINQ semantics | Enumerate one `IEnumerable` twice and watch it re-run. Write `Except` / `DistinctBy` with a key selector and prove what "distinct" kept |
| 5 | **1.4** Records, tuples, pattern matching | Convert your DTOs to records; try to make an entity a record and write down why it goes wrong |
| 6 | **5.1** Testing foundations | The project gets a test project — now, while it's small. Three tests, AAA, named properly |
| 7 | **1.3** Generics & variance | A generic repository or handler with constraints. Make one `IEnumerable<T>` assignment compile and the `List<T>` one fail |
| 8 | **5.2** Unit testing | xUnit `Theory`, one fake, one stub. Test the logic from steps 1–7 |
| 9 | **1.5** Operators & conversion semantics | Build a tiny `Result<T>` with an implicit operator, so `return errors;` compiles for you too |
| 10 | **1.7** Exceptions | Decide in writing: which failures throw, which return a `Result` — and apply it across the project. Give the project its own exception hierarchy, deriving properly |

**Block A checkpoint — two things, both concrete:**
1. Explain why the `HashSet<Property>` in `AddSelectedPropertiesRequest` deduplicates nothing, and fix it correctly.
2. Open `ApplicationActionService`, and explain why each `protected virtual` member is virtual — what a derived service is meant to change, and what happens to the flow when it doesn't.

---

#### Block B — Database · the project becomes a Web API + EF Core + a real MySQL

| # | Topic | The project move |
|---|---|---|
| 11 | **3.4** Schema & integrity | Design the schema before writing a line of C#. Every invariant you can push into a constraint, push |
| 12 | **3.5** EF Core | Map it. Read what EF generates. Break it deliberately: a missing `Include`, a tracked entity you didn't expect |
| 13 | **3.1** Transactions & isolation | Two connections, one row. Reproduce a lost update and a non-repeatable read yourself |
| 14 | **3.2** Locking | `SELECT ... FOR UPDATE` on both connections in the wrong order. Cause a deadlock and read the engine's message |
| 15 | **3.3** Indexing & query performance | Load ~100k rows. `EXPLAIN` the slow query, add the index, `EXPLAIN` again, write the two numbers down |
| 16 | **5.3** Integration testing | Real database, not in-memory. `WebApplicationFactory`, deterministic setup and teardown |

**Block B checkpoint:** take the slowest query in TAS, read its `EXPLAIN`, add an index, prove the difference with numbers.

> Why the database comes second, before async: it pays off in your day job immediately, and steps 13–14 are what make step 19 comprehensible. A race condition without isolation levels is just a word.

---

#### Block C — Async & Concurrency · the project gets a background job and an external call

| # | Topic | The project move |
|---|---|---|
| 17 | **2.1** Async fundamentals | Cause a sync-over-async deadlock, then remove it. Explain what `.IC()` buys you in one paragraph |
| 18 | **2.2** Task composition | `WhenAll` over real calls. Make two of them throw and find the second exception |
| 19 | **2.3** Concurrency hazards | Run the same request twice concurrently and corrupt your own data. Then add the idempotency key that stops it |
| 20 | **2.4** Background work | A Hangfire (or `BackgroundService`) job that is resumable: it can die mid-run and restart without redoing work |
| 21 | **5.4** Practical testing | Test the job. Test time via `ITimeProvider`. Write one regression test for the bug from step 19 |

**Block C checkpoint:** fix the manual/auto race in Outflow, and write an outbox for the data sync.

---

#### Block D — .NET Infrastructure · the project is rebuilt from empty, by hand, no template

> This is the block that cannot be read. `dotnet new web` and nothing else — no scaffolding, no copying from TAS.

| # | Topic | The project move |
|---|---|---|
| 22 | **4.1** Hosting & startup | Start from an empty `Program.cs`. Add a hosted service and shut it down gracefully |
| 23 | **4.2** Dependency Injection | Register the three lifetimes and cause a captive dependency on purpose. Turn on validate-on-build |
| 24 | **4.3** Configuration & Options | Three providers, one key, and prove which one wins. `IOptionsSnapshot` vs `IOptions` — observe the difference |
| 25 | **4.4** Middleware pipeline | Write two middlewares. Reorder them and break something. Add `ProblemDetails` handling |
| 26 | **4.5** AuthN / AuthZ | JWT end to end, then **your own policy + requirement + handler** — the thing `ApplicationAccessHandler` does, written by you |
| 27 | **4.6** Cross-cutting | Serilog with scopes, a typed `HttpClient` with a delegating handler and a retry policy, health checks |

**Block D checkpoint:** the project runs with real auth, and you can explain the middleware order and the reason for it, without opening the file.

---

#### Block E — Architecture · the project stops growing and starts being redesigned

| # | Topic | The project move |
|---|---|---|
| 28 | **Design Patterns** (skim) | A weekend. Map the names onto what you already do — and onto step 2, since most of them are inheritance and composition wearing a name |
| 29 | **6.1** Consolidate what you already do | Write down, in your own words, what CQRS is and what it isn't. Make one anemic entity rich |
| 30 | **6.2** Genuinely different styles | Implement the outbox properly in the project. Then a saga for a two-step workflow |
| 31 | **6.3** Distributed systems basics | Add retry + backoff + jitter and a circuit breaker to the external call from step 17 |

**Block E checkpoint:** redesign a TAS subsystem in a different style, with the **why** and the **trade-offs** written down.

---

### Moving from one step to the next

A step is done when all three are true:

1. **The grep rule** — you found it in TAS. The 🔎 markers in Part 1 are the starting points.
2. **The explain rule** — you can explain it with nothing open in front of you.
3. **The writing rule** — five lines in your own words. Not copied.

The three rules apply to 🔴 lines. A ⚪ line is done when you can say in one sentence what it is and when it would matter — no grep, no project code, no five lines. Treating a ⚪ line like a 🔴 one is the main way this plan overruns.

And two rules about the sequence itself:

4. **Never two steps at once.** One.
5. **Documentation is a reference, not a curriculum.** Go to it for the step you're on; don't read it front to back.

Jumping ahead is allowed in one direction only: if work forces a topic on you early, take it early — then come back and do it properly rather than skipping it twice.

---

### External checkpoints — outside your own bubble

The block checkpoints above are self-marked, and self-marking has a ceiling. At least one of these:

- An interview you don't need. Not to take a job — to find out where you actually stand.
- A contribution to an open-source .NET repo, so someone who doesn't know you reviews your code.
- Rebuild part of TAS from scratch outside the repo, and compare.

---

### If time gets tight (the minimum)

If work consumes you and only part of the sequence fits, keep these steps and drop the rest:

**13 → 14 → 1 → 2 → 3 → 4 → 16 → Block D.**

That is: transactions and locking first (your most dangerous gap, and you work on money), then equality, OOP, closures and LINQ semantics (they produce real bugs in your code today, and step 2 is what every design conversation stands on), then two tests against the existing fixture in `IntegrationTests/`, then the from-scratch project.

Everything else waits.
