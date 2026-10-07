# Backend Security

Security bugs are the ones that make the news. Most are not exotic — they're a
missing authorization check, an unvalidated input, or a secret in a log. This
reference covers the high-frequency backend risks and a pre-ship checklist.
Security is a property of the whole change, not a phase you bolt on; but doing a
deliberate pass before shipping catches the routine mistakes.

## Table of contents

- [AuthN vs AuthZ](#authn-vs-authz)
- [The injection family](#the-injection-family)
- [Input validation and output encoding](#input-validation-and-output-encoding)
- [Secrets management](#secrets-management)
- [Sensitive data handling](#sensitive-data-handling)
- [Rate limiting and abuse](#rate-limiting-and-abuse)
- [Common backend-specific pitfalls](#common-backend-specific-pitfalls)
- [Pre-ship security checklist](#pre-ship-security-checklist)

## AuthN vs AuthZ

Two different questions, and conflating them is a top cause of breaches:

- **Authentication (authN):** *who are you?* Verified once, usually by
  middleware, producing an authenticated principal (user/service).
- **Authorization (authZ):** *are you allowed to do this specific thing to this
  specific resource?* Must be checked **on every operation**, using the
  authenticated identity.

The classic failure is **IDOR / broken object-level authorization**: the
endpoint authenticates the user but then acts on `GET /invoices/{id}` without
checking that *this* user owns *that* invoice. Authenticating is not authorizing.
Every handler that touches a resource must verify the caller's right to it —
scope the query to the principal (`WHERE id = ? AND owner_id = ?`) rather than
fetching by ID and hoping.

For multi-tenant systems, tenant isolation is the master invariant: it should be
structurally difficult to query without a tenant filter. Consider enforcing it
at the data-access layer so no individual query can forget it.

## The injection family

All injection is the same root cause: **untrusted data interpreted as code or
commands.** The fix is always to separate data from the instruction.

- **SQL injection** — never concatenate input into SQL. Use **parameterized
  queries / prepared statements**, always, with no exceptions. ORMs
  parameterize by default; the moment you drop to raw SQL, the discipline is on
  you.
- **Command injection** — don't build shell command strings from input. Use APIs
  that take an argument array and don't invoke a shell; avoid shelling out at
  all if you can.
- **NoSQL / query injection** — object-shaped inputs can smuggle operators
  (`{"$gt": ""}`); validate types and don't pass raw request objects into
  queries.
- **Template / SSTI, LDAP, XML (XXE)** — same principle: parse safely, disable
  dangerous features (external entities in XML parsers), never interpolate input
  into an interpreter.

## Input validation and output encoding

- **Validate all input at the boundary** against an allowlist (what's
  permitted), not a denylist (what's forbidden) — you can't enumerate every bad
  value. Check type, range, length, format.
- **Validate redirects and URLs.** An open redirect or a server-side request to a
  user-supplied URL (**SSRF**) lets an attacker reach internal services. Allowlist
  destinations; block internal IP ranges and metadata endpoints.
- **Encode/escape on output** for the destination context — even for a backend
  emitting HTML, JSON, or CSV — so data can't become markup or a formula.
- **Guard deserialization.** Never deserialize untrusted data into arbitrary
  types with a format that can instantiate classes or run code; prefer
  data-only formats and explicit schemas.

## Secrets management

- **Secrets never live in code or version control** — not API keys, DB
  passwords, signing keys, tokens. Scanning history for leaked keys is a routine
  attacker move.
- **Load secrets from the environment or a secrets manager** (Vault, cloud
  secret store) injected at runtime.
- **Never log secrets**, and be careful they don't ride along in a serialized
  request object, an exception message, or a debug dump.
- **Rotate on exposure**, and design so rotation is possible without a code
  change.

## Sensitive data handling

- **Hash passwords** with a slow, salted, purpose-built algorithm (bcrypt,
  scrypt, Argon2) — never plain, never fast general-purpose hashes like SHA-256
  alone.
- **Encrypt sensitive data** in transit (TLS everywhere, including
  service-to-service) and at rest where required.
- **Minimize and redact PII/secrets in logs and errors.** Full card numbers,
  government IDs, auth tokens, and passwords must never appear in a log line or
  an error response. Redact at the logging layer so it can't slip through.
- **Return generic errors to clients**; keep the diagnostic detail server-side
  behind a request ID. "Invalid username or password" (not "no such user")
  avoids leaking which accounts exist.

## Rate limiting and abuse

- **Rate-limit public and auth endpoints** to blunt brute-force, credential
  stuffing, and scraping. Login and password-reset especially.
- **Bound resource use per request** — max page size, max payload size, query
  timeouts — so one caller can't exhaust the service.
- **Return `429`** with a `Retry-After` when limiting, so well-behaved clients
  back off.

## Common backend-specific pitfalls

- **Mass assignment** — binding a whole request body onto a model lets a caller
  set fields they shouldn't (`is_admin`, `balance`). Bind an explicit allowlist
  of fields.
- **Missing authorization on "internal" endpoints** — admin routes, health/debug
  endpoints, and internal APIs still need protection; "no one knows the URL" is
  not access control.
- **Trusting client-supplied authority** — never trust a `role`, `user_id`, or
  `price` from the request body; derive authority from the authenticated
  session/token server-side.
- **TOCTOU / race conditions in checks** — "check then act" on balances or quotas
  must be atomic (see `database.md`), or a race defeats the check.
- **Verbose errors in production** — stack traces and framework debug pages leak
  internals; disable them outside development.

## Pre-ship security checklist

- [ ] Every resource operation checks **object-level authorization**, not just
      authentication.
- [ ] All queries are **parameterized**; no string-built SQL/commands.
- [ ] Input is **validated at the boundary** with an allowlist.
- [ ] No **secrets** in code, config committed to git, or logs.
- [ ] **Passwords hashed** with bcrypt/scrypt/Argon2; sensitive data encrypted.
- [ ] **No PII/secrets** in logs or client-facing error messages.
- [ ] Client errors are **generic**; details stay server-side with a request ID.
- [ ] **Mass assignment** guarded — only intended fields are bindable.
- [ ] Authority (role, ownership, price) derived **server-side**, never trusted
      from the client.
- [ ] Public/auth endpoints **rate-limited**; request size and page size bounded.
- [ ] User-supplied URLs/redirects validated against **SSRF** and open redirect.
