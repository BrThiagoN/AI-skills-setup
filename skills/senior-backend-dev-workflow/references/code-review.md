# Self-Review Checklist

Before you call a backend change done — and before anyone else reviews it — read
your own diff as an adversary trying to find the bug that pages you. The author
knows what the code *meant*; the reviewer sees what it *says*. Deliberately
switch hats.

Work through the diff top to bottom once for understanding, then again against
this checklist. Not every item applies to every change; the value is in
*deciding* each, so nothing important gets skipped by default.

## Correctness

- [ ] Does the change actually do what the requirement asked — with a concrete
      input/output example in mind, not just in the abstract?
- [ ] Are the **invariants** (from Phase 1) still true across the whole change?
- [ ] **Edge cases:** empty input, null/absent values, zero, negative, maximum,
      duplicates, very large collections, unicode/long strings.
- [ ] **Off-by-one and boundary** conditions in pagination, ranges, and loops.
- [ ] Are all **branches reachable and correct**, including the `else` you didn't
      think much about?

## Error handling

- [ ] Is **every error return / exception handled or deliberately propagated** —
      no bare `catch` that logs and continues past a real failure?
- [ ] Do failures **leave state consistent** (no half-applied multi-step
      change)?
- [ ] Are downstream/external call failures (timeout, 500, malformed response)
      handled, not assumed away?
- [ ] Do error messages avoid leaking internals to the client while logging
      enough server-side to diagnose?

## Data and concurrency

- [ ] **Parameterized queries only** — no user input concatenated into SQL.
- [ ] **No N+1** — related data is batched or joined.
- [ ] Every growable list query is **paginated and bounded**.
- [ ] **Transaction scope** is correct — atomic where it must be, not held across
      slow calls.
- [ ] **Read-modify-write on shared rows** is race-safe (locking or atomic
      update), not "read into app, compute, write back."
- [ ] The **migration is expand-contract safe**, deployable alongside old code,
      and reversible.

## Idempotency and retries

- [ ] Can this operation run **twice** (client retry, at-least-once queue) without
      duplicating an effect? If it's a write that a client might retry, is it
      idempotent?
- [ ] Do outbound calls have **timeouts and bounded retries with backoff**, so a
      slow dependency can't hang or storm?

## Security

- [ ] **Authorization** checked on this specific resource for this specific
      caller — not just authentication (watch for IDOR).
- [ ] **Input validated** at the boundary against an allowlist.
- [ ] No **secrets** in the diff or logs; no **PII** in logs or client errors.
- [ ] Authority (role, price, ownership, tenant) derived **server-side**, not
      trusted from the request.
- [ ] No **mass assignment** — only intended fields are bindable.

## Observability

- [ ] Failures are **logged with enough context** (operation, key inputs, cause)
      to diagnose without a repro.
- [ ] The **correlation ID** flows through new log lines and downstream calls.
- [ ] New failure modes worth alerting on are **measurable**.

## Tests

- [ ] Do the tests cover the **invariants and failure paths**, not just the happy
      path?
- [ ] Would these tests **actually fail** if the code were wrong? (A test that
      passes against a broken implementation is theater.)
- [ ] Is there a test for the **idempotency/concurrency** behavior if this change
      touches shared state?
- [ ] Are the tests **deterministic** (no clock, randomness, network, or
      inter-test coupling)?
- [ ] Did you **run them and read the output**?

## Fit and clarity

- [ ] Does the code **match the surrounding conventions** — layering, error
      idiom, naming, comment density?
- [ ] Is anything **clever that should be obvious**? Optimize for the next
      maintainer, not for brevity.
- [ ] Is there **dead code, a leftover debug log, or a commented-out block** to
      remove?
- [ ] Did you **remove scope creep** — unrelated changes that belong in their own
      diff?

## The honest summary

Finish by writing, for yourself and the reviewer:

- **What changed** and why, in a sentence or two.
- **What you verified and how** — which tests you ran and what you observed.
- **What you deliberately left out** of scope.
- **Where the risk is** — the part of the diff a reviewer should look at hardest.

Reporting faithfully — including "I couldn't test X" or "this migration needs a
careful rollout" — is part of the job, not an admission of weakness. The senior
move is surfacing the risk, not hiding it.
