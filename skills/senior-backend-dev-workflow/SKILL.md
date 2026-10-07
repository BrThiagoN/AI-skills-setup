---
name: senior-backend-dev-workflow
description: >-
  A disciplined workflow for building, changing, and reviewing backend services
  the way a senior backend engineer would. Use this skill whenever the task
  involves server-side code — REST/GraphQL/gRPC APIs, database schemas or
  migrations, background jobs, queues, authentication/authorization, caching,
  service-to-service calls, or any change to a backend system's behavior or
  data. Trigger it even when the user just says "add an endpoint", "fix this
  bug", "write a service", "add a column", or "make this faster" without naming
  a process — the value of the skill is applying the full rigor (explore →
  design → implement → test → harden → review) rather than jumping straight to
  code. Especially reach for it on anything touching persistence, money, auth,
  concurrency, or public API contracts, where a naive change is likely to be
  subtly wrong or unsafe.
---

# Senior Backend Dev Workflow

The difference between a junior and a senior backend change is rarely the code
that gets written — it's everything *around* it: understanding the real
requirement, respecting the existing system, anticipating failure, and leaving
the codebase safer than you found it. This skill encodes that discipline as a
repeatable workflow so that every backend change ships with the right questions
already answered.

Backend code has a specific character that makes carefulness pay off: it holds
**shared, persistent state**, it runs **concurrently**, it fails **partially**
(the network is down for *some* callers, the DB commits *some* rows), and its
**contracts outlive its code** (clients depend on your API shape long after you
forget it). Most production incidents trace back to one of those four
realities. The workflow below is organized to surface them before they surface
you.

## The workflow at a glance

Work through these phases in order. Early phases are cheap and prevent expensive
mistakes later — don't skip them to "save time," because the time you save is
usually borrowed against a production incident.

1. **Understand** — pin down the real requirement and its constraints.
2. **Explore** — learn how this codebase already does things before adding to it.
3. **Design** — decide the contract, the data model, and the failure behavior.
4. **Implement** — write the change to match the system, defensively.
5. **Test** — prove it works and prove it fails safely.
6. **Harden** — security, performance, observability, operability.
7. **Review** — self-review as an adversary before asking anyone else to.

You don't need heavy ceremony for a one-line fix — scale the effort to the
blast radius. A typo in a log line needs phases 4–5. A new payment endpoint
needs all seven, thoroughly. The judgment about how much rigor to apply *is*
the senior skill; the sections below tell you what "thorough" looks like so you
can dial it in deliberately.

---

## Phase 1 — Understand the requirement

The most expensive backend bugs are built correctly to the wrong spec. Before
touching code, make sure you can answer:

- **What is the actual behavior wanted**, in terms of inputs and outputs? Get a
  concrete example: "given request X, the system should do Y and return Z."
- **Who calls this, and what do they already depend on?** A change is only
  "backward compatible" relative to real consumers. If you can't name them,
  find out before you break them.
- **What are the invariants that must never break?** Money must balance. A user
  can't see another tenant's data. An order can't ship twice. Write these down —
  they become your test assertions and your review checklist.
- **What's the expected scale and latency?** "A few rows per day" and "50k
  writes per second" call for completely different designs. Don't guess; ask or
  measure.

If the requirement is ambiguous on any of these and the answer changes the
design, ask the user a focused question rather than guessing. A thirty-second
clarification beats a day of rework. But don't sandbag — if a sensible default
is obvious and reversible, state your assumption and proceed.

---

## Phase 2 — Explore the existing system

New code should look like it was written by the person who wrote the code around
it. Before implementing, learn the local conventions — they encode decisions you
don't want to re-litigate accidentally.

Look for and match:

- **Layering and structure.** Where do HTTP handlers, business logic, and data
  access live? Is there a service layer, a repository pattern, plain functions?
  Follow the existing seams instead of inventing new ones.
- **Error handling idiom.** Does the codebase return error values, throw typed
  exceptions, use a `Result` type? Mismatched error handling is a common source
  of swallowed failures.
- **How the database is accessed.** ORM, query builder, raw SQL? How are
  transactions started and committed? How are migrations written and run?
- **Validation and serialization.** Is there a schema layer (e.g. a DTO /
  validation library) that all input passes through? New endpoints should use
  it, not hand-roll parsing.
- **Auth.** How is the caller authenticated and authorized? There's almost
  always an existing middleware or decorator — use it; don't reimplement auth.
- **Config and secrets.** How are environment-specific values and credentials
  loaded? Never hardcode what the codebase already parameterizes.
- **Tests.** Find the nearest existing test to what you're building and mirror
  its structure, fixtures, and helpers.

The output of this phase is a short mental (or written) model: "handlers in
`X`, business logic in `Y`, DB access via `Z`, errors are `W`, tests live in
`V`." If the codebase is large or unfamiliar, delegate this survey to a search
agent so you get the map without reading every file yourself.

---

## Phase 3 — Design the change

Design is where senior judgment concentrates. For anything non-trivial, decide
these before writing code. `references/api-design.md` and
`references/database.md` go deeper; the essentials:

### The contract

- **API shape is forever-ish.** Adding a field is easy; removing or renaming one
  breaks clients. Design request/response shapes so they can grow: prefer
  additive evolution, avoid leaking internal enums/IDs you'll want to change,
  and think about how a client paginates, retries, and handles partial results.
- **Make operations idempotent where the caller might retry.** Networks drop
  responses, so clients retry. If "create payment" runs twice, you must not
  charge twice. Idempotency keys, natural unique constraints, or upserts are how
  you get there. This is one of the highest-leverage backend habits.
- **Design the error responses, not just the happy path.** What status code and
  body does a validation failure, a missing resource, a conflict, and a
  downstream outage produce? Consistent, machine-readable errors are part of the
  contract.

### The data model

- **Model the invariants into the schema.** Uniqueness, foreign keys, non-null,
  and check constraints let the database enforce correctness that application
  code will eventually forget to. A constraint is a test that runs on every
  write, forever.
- **Plan the migration as a rollout, not an event.** Schema changes must be safe
  to deploy while the old code is still running (they usually deploy before it).
  Adding a nullable column or a new table is safe; renaming/dropping/backfilling
  needs a multi-step expand-migrate-contract sequence. See
  `references/database.md`.
- **Think about how it's queried.** The right index turns a table scan into a
  lookup. Design the access pattern and the index together, not after the
  `slow query` alert fires.

### The failure behavior

- **Enumerate what can fail and decide what happens.** The DB call times out,
  the queue is full, the third-party API returns 500, two requests race for the
  same row. For each, decide: retry, fail the request, degrade, or compensate.
  Undecided failure modes become 3 a.m. pages.
- **Get transaction boundaries right.** What must commit atomically? Don't hold
  a transaction open across a slow network call. Don't do non-transactional side
  effects (send email, call an API) inside a DB transaction that might roll
  back — use the outbox pattern or commit first.

If several designs are viable and the tradeoff is real, briefly lay out the
options and recommend one rather than silently picking. If it's clear, just
proceed.

---

## Phase 4 — Implement

Now write the code — to match the system (Phase 2) and the design (Phase 3).
Senior implementation habits:

- **Validate input at the boundary.** Everything from outside — request bodies,
  query params, queue messages, third-party responses — is untrusted until
  validated. Reject early with a clear error rather than letting a bad value
  flow into business logic.
- **Fail loudly, then handle deliberately.** Don't swallow exceptions or ignore
  error returns. Either handle a failure meaningfully or let it propagate to a
  layer that can. A bare `catch` that logs and continues hides the bug you'll be
  paged for.
- **Keep side effects at the edges and make them idempotent.** Pure business
  logic in the middle, I/O at the boundaries, is easier to test and reason
  about.
- **Don't log secrets or PII.** Tokens, passwords, full card numbers, and
  personal data don't belong in logs. Redact at the logging layer.
- **Leave the code readable at the altitude of the surrounding code.** Match the
  naming, the comment density, and the abstractions already in use. Clever code
  that only you understand is a liability in a service five people maintain.

Concurrency deserves special care: if two requests can touch the same row,
decide your consistency strategy explicitly (optimistic version column,
`SELECT ... FOR UPDATE`, a unique constraint that turns a race into a caught
error). "It works when I test it alone" is not evidence it's correct under load.

---

## Phase 5 — Test

Tests are how you prove the change is correct *and* keep it correct as the code
evolves. Aim your effort at behavior and risk, not at a coverage number.

- **Test the invariants from Phase 1.** These are the assertions that matter
  most: money balances, tenants stay isolated, the operation is idempotent.
- **Test the failure paths, not just the happy path.** The happy path usually
  works; bugs live in the edges — empty inputs, duplicates, concurrent writes,
  downstream timeouts, partial failures. A test that a retried request doesn't
  double-charge is worth ten that check the 200 response.
- **Prefer the test level that gives the most confidence per unit of
  maintenance.** Fast unit tests for logic; integration tests (real DB, real
  migrations) for anything involving persistence or transactions, because that's
  where the subtle bugs are and mocks hide them. Match whatever the codebase
  already does.
- **Make tests deterministic.** No reliance on wall-clock time, random ordering,
  or shared mutable state between tests. Flaky tests get ignored, and ignored
  tests protect nothing.
- **Actually run them and read the output.** Report real results. If something
  fails or you skipped a case, say so plainly.

See `references/testing.md` for patterns on integration tests, test data,
concurrency testing, and testing external dependencies.

---

## Phase 6 — Harden

This is the phase juniors skip and seniors treat as non-negotiable for anything
user-facing. Walk the checklist; each item links to deeper guidance.

- **Security.** Authorization checked on every path (not just authentication).
  Input validated and queries parameterized (no SQL/command injection). Secrets
  out of code and logs. No sensitive data in error messages. See
  `references/security.md`.
- **Performance.** No N+1 queries. Pagination on any list that can grow.
  Appropriate indexes. Bounded resource use (no unbounded `IN` clauses, no
  loading a whole table into memory). Caching where it genuinely helps, with a
  correct invalidation story. See `references/database.md`.
- **Observability.** Structured logs with correlation/request IDs so a single
  request can be traced. Metrics on the things you'd want during an incident
  (rate, errors, duration). Errors carry enough context to diagnose without a
  repro. See `references/observability.md`.
- **Operability.** Safe to deploy (migration ordering, backward compatibility,
  feature flags for risky changes). Safe to roll back. Timeouts and retries with
  backoff on every outbound call so one slow dependency can't exhaust your
  service.

Not every item applies to every change — an internal script doesn't need
metrics — but *deciding* each is out of scope is different from forgetting it.

---

## Phase 7 — Review as an adversary

Before you call it done, review your own diff as if you were trying to find the
bug that gets you paged. `references/code-review.md` has the full checklist;
the mindset:

- **Re-read the diff top to bottom** as a reviewer, not the author. The author
  sees intent; the reviewer sees what the code actually says.
- **Hunt the classic backend failure modes:** unhandled error, missing auth
  check, N+1 query, non-idempotent retry, unsafe migration, race condition,
  swallowed exception, unbounded query, leaked secret, missing input validation.
- **Check the tests test the risk**, not just the mechanics. Would these tests
  actually fail if the code were wrong?
- **Confirm the invariants from Phase 1 still hold** across the whole change.

Then summarize honestly: what changed, what you verified and how, what you
deliberately left out, and any risk the reviewer should focus on. Faithful
reporting — including "I couldn't test X" — is part of the craft.

---

## Reference files

Read these when the phase that points to them applies. They exist so this file
stays scannable while the depth is there when you need it.

- `references/api-design.md` — Designing REST/GraphQL/gRPC contracts that evolve
  safely: versioning, pagination, idempotency, error formats, compatibility.
- `references/database.md` — Data modeling, safe migrations (expand-contract),
  indexing, transactions, isolation levels, N+1 avoidance, connection pooling.
- `references/testing.md` — Backend testing strategy: the test pyramid for
  services, integration tests with real databases, testing concurrency, test
  data, and handling external dependencies.
- `references/security.md` — AuthN vs authZ, the injection family, secrets
  management, sensitive-data handling, rate limiting, and a pre-ship checklist.
- `references/observability.md` — Structured logging, metrics, tracing,
  correlation IDs, health checks, and what to instrument for incident response.
- `references/code-review.md` — A concrete self-review checklist of the backend
  bugs most worth catching before anyone else sees the diff.
