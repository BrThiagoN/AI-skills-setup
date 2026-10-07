# Backend Testing

The point of tests is confidence: that the change is correct now, and that a
future change that breaks it gets caught before production. Optimize for
**confidence per unit of maintenance**, not for a coverage percentage — 100%
coverage of getters proves nothing; one integration test that a retried payment
doesn't double-charge protects real money.

## Table of contents

- [The service test pyramid](#the-service-test-pyramid)
- [What to test](#what-to-test)
- [Integration tests with a real database](#integration-tests-with-a-real-database)
- [Testing external dependencies](#testing-external-dependencies)
- [Testing concurrency and idempotency](#testing-concurrency-and-idempotency)
- [Test data](#test-data)
- [Keeping tests trustworthy](#keeping-tests-trustworthy)

## The service test pyramid

- **Unit tests** — pure business logic with no I/O. Fast, plentiful, run on
  every save. Test the branchy logic: pricing rules, state transitions,
  validation.
- **Integration tests** — the code plus its real collaborators, especially the
  database and its real migrations. Fewer, slower, but they catch the bugs that
  matter most in backend code: wrong SQL, bad transaction scope, migration
  mistakes, serialization errors. **Do not skimp here** — mocking the database
  hides exactly the class of bug persistence code is prone to.
- **End-to-end / contract tests** — the running service over HTTP, or a contract
  test against a consumer's expectations. Few and high-value; they verify the
  contract you promised.

The backend-specific adjustment to the classic pyramid: weight integration tests
more heavily than a frontend project would, because so much backend risk lives
in the database and transaction layer.

## What to test

Aim tests at risk and behavior:

- **The invariants.** The things that must never break (balances reconcile,
  tenant isolation holds, no double-ship). These are your highest-priority
  assertions.
- **The failure paths.** Empty input, missing record, duplicate submit,
  malformed payload, downstream timeout, permission denied. Bugs cluster at the
  edges; the happy path usually works on the first try.
- **The boundaries.** Off-by-one on pagination, zero and max values, unicode and
  very long strings, timezone edges, empty collections.
- **The contract.** Correct status codes and error shapes for each condition,
  not just the 200.

Don't test framework internals or the language's standard library — test *your*
logic and *your* wiring.

## Integration tests with a real database

The gold standard for data-access code is a test that runs against a real
instance of the same engine you use in production (not an in-memory substitute,
which has different SQL semantics).

- **Spin up a real database** — an ephemeral container (e.g. Testcontainers) or a
  dedicated test instance. Run your **actual migrations** against it so the test
  also validates the schema.
- **Isolate tests from each other.** Wrap each test in a transaction that rolls
  back, or truncate/reset between tests. Shared leftover state is the #1 cause of
  tests that pass alone and fail together.
- **Assert on the database, not just the return value.** After "create order,"
  query the row and check it — that catches serialization and persistence bugs a
  return-value check misses.

## Testing external dependencies

Third-party APIs and other services shouldn't be called for real in tests
(slow, flaky, rate-limited, sometimes charges money). Fake them at a boundary
you control:

- **Prefer faking at the HTTP/transport edge** (a stub server or recorded
  responses) over mocking your own client class, so you still exercise your
  serialization and error handling.
- **Test how you handle *their* failures** — timeouts, 500s, malformed bodies,
  slow responses — not just their success case. Your resilience code is only
  proven if a test exercises it.
- **Contract-test the integration** against the real dependency periodically (or
  against its published schema), so a change on their side that your fake doesn't
  reflect still gets noticed.

## Testing concurrency and idempotency

These are the tests juniors omit and the bugs that page seniors.

- **Idempotency:** submit the same operation twice (same idempotency key or same
  natural key) and assert exactly one effect — one row, one charge. This is
  often a plain sequential test and enormously valuable.
- **Concurrency:** fire N parallel requests at the same resource (transfer from
  the same account, claim the same seat) and assert the invariant holds — total
  is conserved, only one claim succeeds. Even a modest parallel test surfaces
  missing locks that no single-threaded test can.

## Test data

- **Build test data with factories/builders**, not sprawling fixtures. A helper
  that creates a valid order with overridable fields keeps tests readable and
  survives schema changes in one place.
- **Make each test set up exactly what it needs** and no more, so the test
  documents its own preconditions and doesn't depend on hidden global state.
- **Use realistic values.** `"test"` in every field hides bugs that real names,
  unicode, and long strings expose.

## Keeping tests trustworthy

A test suite is only worth its maintenance if people trust it. Protect that:

- **Deterministic, always.** No dependence on wall-clock time (inject a clock),
  random values (seed them), network to the internet, or inter-test ordering.
- **Flaky tests are worse than no tests** — they train people to ignore red.
  Fix or delete a flaky test immediately; don't retry-loop around it.
- **A test should fail for one clear reason.** When it goes red, the name and
  assertion should point at the cause.
- **Run them and read the output.** Report actual results honestly — if a test
  fails or you couldn't run a category, say so; don't claim green you didn't
  see.
