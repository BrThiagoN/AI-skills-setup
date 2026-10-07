# API Design

APIs are contracts. Code is cheap to change; a contract that other teams and
external clients depend on is expensive, because you can't see or fix all the
callers. Design the contract to **grow additively** so you rarely have to break
it.

## Table of contents

- [Compatibility: the golden rule](#compatibility-the-golden-rule)
- [Versioning](#versioning)
- [Resource and payload design](#resource-and-payload-design)
- [Pagination](#pagination)
- [Idempotency](#idempotency)
- [Error responses](#error-responses)
- [Validation](#validation)
- [GraphQL notes](#graphql-notes)
- [gRPC / protobuf notes](#grpc--protobuf-notes)

## Compatibility: the golden rule

A change is **backward compatible** if every existing client keeps working
without modification. Safe (additive) changes:

- Adding a new optional request field (with a sensible default).
- Adding a new field to a response — *if clients ignore unknown fields*, which
  well-behaved ones do.
- Adding a new endpoint, a new enum value clients already tolerate, or a new
  optional query parameter.

Breaking changes (require a version bump or a migration plan):

- Removing or renaming a field, endpoint, or parameter.
- Changing a field's type, format, or units (seconds → milliseconds is a classic
  outage).
- Making an optional field required, or tightening validation on existing input.
- Changing default behavior, status codes, or error shapes clients branch on.
- Adding a new enum value that old clients *don't* tolerate (breaks strict
  parsers).

When in doubt, assume a client somewhere depends on the current behavior — in a
public API, someone always does.

## Versioning

Pick a strategy and apply it consistently:

- **URI versioning** (`/v1/orders`) — most visible, easiest to route and cache,
  most common for public REST.
- **Header versioning** (`Accept: application/vnd.api.v2+json`) — keeps URLs
  stable, less discoverable.

Version at a coarse granularity (the whole API or a large surface), not per
endpoint, or you'll drown in combinations. Prefer to **avoid new versions** by
evolving additively; a new major version means maintaining two code paths and
migrating every client. When you must, keep the old version working with a clear
deprecation window and communicated sunset date.

## Resource and payload design

- **Name resources as nouns, use HTTP verbs for actions.** `POST /orders`,
  `GET /orders/{id}`, `PATCH /orders/{id}`. Reserve verb-y RPC-style routes
  (`/orders/{id}/cancel`) for genuine state transitions that don't map cleanly
  to CRUD.
- **Don't leak internal representation.** Exposing raw database column names,
  auto-increment primary keys, or internal enum spellings welds your storage to
  your contract. Map to a stable external shape. Opaque IDs (UUIDs or encoded
  tokens) also avoid leaking row counts and enabling enumeration attacks.
- **Use the right status codes.** `200` success, `201` created, `202` accepted
  (async), `204` no content, `400` bad input, `401` unauthenticated, `403`
  unauthorized, `404` not found, `409` conflict, `422` semantic validation
  failure, `429` rate limited, `500` server error, `503` unavailable. Clients
  branch on these; use them accurately.
- **Be consistent across the whole API** — field naming (camelCase vs
  snake_case), date formats (prefer RFC 3339 / ISO 8601 UTC), money
  representation (integer minor units or decimal strings, never floats).

## Pagination

Any list endpoint that can grow **must** paginate — an unbounded list is a
latent outage the day the table gets big.

- **Cursor (keyset) pagination** is the robust default: the client passes an
  opaque cursor pointing at the last item seen, and you query
  `WHERE (sort_key) > cursor ORDER BY sort_key LIMIT n`. Stable under concurrent
  inserts and stays fast at any offset.
- **Offset/limit** (`?offset=1000&limit=50`) is simple but degrades — the DB
  scans and discards `offset` rows every page — and skips/duplicates items when
  the underlying data changes between pages. Fine for small, stable datasets;
  avoid for large or fast-changing ones.

Always return enough for the client to continue (a `next_cursor` or
`has_more`), and enforce a **maximum page size** server-side so a client can't
request a million rows at once.

## Idempotency

Because clients retry on network failure, any non-idempotent write can execute
twice. `GET`, `PUT`, and `DELETE` are idempotent by definition; `POST` is the
danger.

Make retryable writes safe with one of:

- **Idempotency keys.** The client sends a unique key (e.g.
  `Idempotency-Key` header) per logical operation; you store it with the result
  and return the stored result on replay instead of re-executing. Standard for
  payment-like APIs.
- **Natural unique constraints.** A unique index on
  `(customer_id, external_order_id)` turns a duplicate create into a caught
  conflict instead of a second row.
- **Upserts.** `INSERT ... ON CONFLICT DO UPDATE` when "create or overwrite" is
  the correct semantic.

State this in the contract so clients know retrying is safe.

## Error responses

Errors are part of the API. Return a **consistent, machine-readable** shape, for
example:

```json
{
  "error": {
    "code": "insufficient_funds",
    "message": "Account balance is below the requested amount.",
    "details": [{ "field": "amount", "issue": "exceeds_balance" }],
    "request_id": "req_01H..."
  }
}
```

- **`code`** is a stable, documented string clients can branch on — don't make
  them regex the human `message`.
- **`message`** is for humans/logs; safe to change.
- Include a **`request_id`** that also appears in your logs, so a user reporting
  an error gives you the exact needle.
- **Never leak internals** — stack traces, SQL, internal hostnames, or secrets —
  in error bodies. Log those server-side; return a generic message plus the
  request ID.

## Validation

Validate every field at the boundary and reject invalid input with `400`/`422`
and a specific message naming the offending field. Prefer a schema/DTO layer
(e.g. a validation library) over hand-rolled `if` checks — it's exhaustive,
self-documenting, and consistent. Validate types, ranges, lengths, formats, and
cross-field rules ("`end_date` must be after `start_date`"). Fail on unknown
fields for internal APIs if you want strictness; tolerate them for public ones
to allow forward compatibility — decide deliberately.

## GraphQL notes

- **Deprecate, don't delete** fields (`@deprecated(reason: ...)`); clients may
  still select them.
- **Guard against expensive queries** — depth limits, complexity scoring,
  pagination on list fields — or a single query can table-scan your database.
- **The N+1 problem is acute** in resolvers; batch with a dataloader so
  resolving 100 items doesn't fire 100 queries.
- Authorize at the field/resolver level, not just the endpoint — a single query
  can reach across many resources.

## gRPC / protobuf notes

- **Never reuse or renumber field tags**; that's what breaks wire
  compatibility. Add new fields with new tag numbers, and reserve retired ones.
- New fields must be optional-friendly — proto3 scalars can't distinguish unset
  from zero unless you use `optional` or wrappers; pick deliberately for fields
  where "absent" differs from "zero".
- Adding a field or a new RPC is compatible; removing/renaming a field or
  changing its type is not.
