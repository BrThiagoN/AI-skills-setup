# Databases: Modeling, Migrations, and Query Safety

The database is where backend mistakes become permanent. Application bugs get
redeployed; corrupted or lost data often can't be recovered. Treat schema and
query decisions with corresponding care.

## Table of contents

- [Data modeling](#data-modeling)
- [Constraints are enforced correctness](#constraints-are-enforced-correctness)
- [Safe migrations: expand-contract](#safe-migrations-expand-contract)
- [Indexing](#indexing)
- [Transactions and isolation](#transactions-and-isolation)
- [Concurrency and races](#concurrency-and-races)
- [The N+1 problem](#the-n1-problem)
- [Connection pooling and limits](#connection-pooling-and-limits)
- [Query safety checklist](#query-safety-checklist)

## Data modeling

- **Model to the invariants, then to the queries.** Start from what must always
  be true (every order has a customer; an email is unique) and encode that
  structurally. Then shape tables and indexes around how the data is actually
  read.
- **Normalize by default, denormalize deliberately.** Normalized data has one
  source of truth and can't drift. Denormalize (duplicate data for read speed)
  only when a measured read pattern demands it, and own the consistency cost you
  just took on.
- **Choose keys thoughtfully.** Surrogate keys (UUID/auto-increment) decouple
  identity from mutable business data. UUIDs avoid enumeration and let clients
  generate IDs, but random UUIDs fragment B-tree indexes — prefer time-ordered
  variants (UUIDv7/ULID) for high-insert tables.
- **Store money as integer minor units or fixed-point decimal**, never floating
  point. `0.1 + 0.2 != 0.3` is a real bug in a ledger.
- **Store timestamps in UTC** with a timezone-aware type; convert at the edges.

## Constraints are enforced correctness

A database constraint is a test that runs on every write for the life of the
system — far more reliable than application checks, which each new code path can
forget. Use them:

- `NOT NULL` for anything that must be present.
- `UNIQUE` for natural keys and idempotency (`(customer_id, external_ref)`).
- `FOREIGN KEY` to prevent orphaned rows; choose `ON DELETE` behavior
  deliberately (`RESTRICT`, `CASCADE`, `SET NULL`).
- `CHECK` for domain rules (`amount >= 0`, `status IN (...)`).

A unique constraint also converts a race condition into a clean, catchable error
— two concurrent inserts, one wins, the other gets a violation you handle.

## Safe migrations: expand-contract

Schema changes deploy while the **old application code is still running** (during
a rolling deploy, both versions run at once). So a migration must be compatible
with the code both before and after it. The safe pattern for any non-trivial
change is **expand → migrate → contract**, across multiple deploys:

**Adding a column the app will write:**
1. **Expand:** add the column as nullable (or with a default). Old code ignores
   it; new code can start writing it. Safe.
2. **Migrate:** backfill existing rows in batches (not one giant `UPDATE` that
   locks the table); deploy code that reads the new column.
3. **Contract:** once every row is populated and all code depends on it, add
   `NOT NULL`.

**Renaming a column** (never rename in place — it breaks the running old code):
1. Add the new column; write to **both** old and new from the app.
2. Backfill the new column from the old.
3. Switch reads to the new column.
4. Stop writing the old column; drop it in a later deploy.

**Operations that lock or rewrite the table** are the ones that cause outages —
adding a non-nullable column with a default on old MySQL/Postgres versions,
adding an index without `CONCURRENTLY` (Postgres), changing a column type. Know
your engine's locking behavior; on Postgres use `CREATE INDEX CONCURRENTLY` and
add constraints as `NOT VALID` then `VALIDATE` separately to avoid long locks.

**Every migration needs a rollback path.** Prefer changes that are safe to roll
back at any step. Dropping or destructive changes should trail the code change by
a full deploy cycle so a rollback never lands on missing columns.

## Indexing

An index turns an O(n) table scan into an O(log n) lookup — and slows writes
slightly and costs storage. Index the columns you filter, join, and sort on.

- **Match the index to the query.** A composite index `(a, b)` serves
  `WHERE a = ? AND b = ?` and `WHERE a = ?`, but not `WHERE b = ?` alone (column
  order matters — leftmost prefix).
- **Index foreign keys** you join on; many engines don't do it automatically.
- **Covering indexes** (including the selected columns) let the DB answer from
  the index alone, skipping the table read.
- **Read the query plan** (`EXPLAIN ANALYZE`) rather than guessing. "Seq Scan"
  on a big table under a hot query is a red flag.
- Don't over-index: every index is write amplification. Remove unused ones.

## Transactions and isolation

- **A transaction is your atomicity boundary** — all-or-nothing. Wrap
  multi-statement invariants (debit one account, credit another) in one
  transaction so a crash can't leave them half-applied.
- **Keep transactions short.** Never hold one open across a slow network call or
  user think-time; it holds locks and exhausts the connection pool.
- **Know your isolation level.** `READ COMMITTED` (common default) still allows
  non-repeatable reads and lost updates on read-modify-write. For
  check-then-act logic (reserve inventory, allocate a seat), you need explicit
  locking or `SERIALIZABLE`, and must be ready to retry on serialization
  failures.
- **Don't put non-transactional side effects inside a DB transaction.** Sending
  an email or calling an API mid-transaction means a rollback can't undo the
  email, and a commit failure after the call leaves you inconsistent. Commit
  first, or use the **transactional outbox**: write the intent to an `outbox`
  table in the same transaction, and a separate process delivers it.

## Concurrency and races

Two requests hitting the same row is the norm, not the exception. Choose a
strategy:

- **Optimistic locking:** add a `version` column; update with
  `WHERE id = ? AND version = ?` and bump it. If zero rows change, someone else
  won — reload and retry or fail. Great when conflicts are rare.
- **Pessimistic locking:** `SELECT ... FOR UPDATE` locks the row for the
  transaction. Use when conflicts are common or the critical section is short.
- **Let the database arbitrate:** a `UNIQUE` constraint or an atomic
  `UPDATE ... SET balance = balance - ? WHERE balance >= ?` (checking the
  affected-row count) avoids read-modify-write races entirely — often the
  simplest correct option.

The anti-pattern to avoid: read a value into the app, compute, write it back.
Between the read and the write, another request changed it, and one update is
silently lost.

## The N+1 problem

Fetching a list, then firing one more query per item, is the most common backend
performance bug. Loading 100 orders and then their customer one-by-one is 101
queries. Fix by:

- **Eager loading / joins** — fetch the related data in the original query.
- **Batch loading** — collect the IDs and issue one `WHERE id IN (...)`.
- ORMs make N+1 easy to write by accident (lazy relations in a loop) — inspect
  the queries your ORM actually emits, don't trust the code to look efficient.

## Connection pooling and limits

Database connections are a scarce, expensive resource. Every service should use a
**bounded connection pool**, sized to the database's capacity — not the app's.
Ten app instances with a pool of 50 each is 500 connections; most databases fall
over well before that. Set pool size, acquisition timeout, and max connection
lifetime deliberately. A slow query holding a connection while the pool is
exhausted is how one bad endpoint takes down the whole service.

## Query safety checklist

Before shipping data-access code:

- [ ] **Parameterized queries only** — never string-concatenate user input into
  SQL. This is the SQL-injection line; there is no acceptable exception.
- [ ] Every list query is **paginated and bounded**.
- [ ] No **N+1** — related data is batched or joined.
- [ ] Queries hitting large tables use an **index** (checked via `EXPLAIN`).
- [ ] Multi-statement invariants are wrapped in a **transaction** with the right
      scope.
- [ ] Read-modify-write on shared rows uses **locking or an atomic update**.
- [ ] No unbounded `IN (...)` from a caller-supplied list.
- [ ] The migration is **expand-contract safe** and has a rollback.
