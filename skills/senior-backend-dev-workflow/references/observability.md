# Observability

You can't fix what you can't see. Observability is what turns a 3 a.m. page from
"the service is broken, no idea why" into "requests to endpoint X are timing out
on the payments DB call, here's the trace." Instrument for the incident you
haven't had yet — the goal is to diagnose a problem you didn't anticipate,
without a reproduction.

The three pillars — **logs, metrics, traces** — answer different questions:
metrics tell you *something is wrong* (and alert you), traces tell you *where*
in a request's path, logs tell you *what exactly* happened.

## Table of contents

- [Structured logging](#structured-logging)
- [Correlation / request IDs](#correlation--request-ids)
- [What to log (and what not to)](#what-to-log-and-what-not-to)
- [Metrics](#metrics)
- [Distributed tracing](#distributed-tracing)
- [Errors with context](#errors-with-context)
- [Health checks](#health-checks)
- [Instrumentation checklist](#instrumentation-checklist)

## Structured logging

Log **structured key-value events (JSON)**, not interpolated prose. Compare:

```
Bad:  log("User " + id + " failed to place order: " + err)
Good: log.error("order_place_failed", user_id=id, order_id=oid,
                error=err.code, duration_ms=dt)
```

Structured logs are queryable — you can ask "all `order_place_failed` events for
`user_id=42` in the last hour" instead of grepping free text. Standardize field
names across the service (`user_id`, not `uid`/`userId`/`user`) so queries work
everywhere.

Use **log levels** meaningfully: `ERROR` = something needs attention, `WARN` =
suspicious but handled, `INFO` = notable business events, `DEBUG` = diagnostic
detail off by default in production. If everything is `ERROR`, nothing is.

## Correlation / request IDs

The single most useful move in a distributed system: attach a **correlation ID**
to every request at the edge and thread it through every log line, downstream
call, and queue message that request touches. Then one ID reconstructs the
entire path across services.

- Generate it at the entry point (or accept an inbound `X-Request-ID` /
  trace header from an upstream caller).
- Put it in a request-scoped context so every log automatically includes it.
- Propagate it on outbound HTTP/gRPC calls and into async jobs, so the async
  work is still tied to the request that spawned it.
- Return it to the client (in the response or error body) so a user's bug report
  carries the exact needle.

## What to log (and what not to)

Log:

- Request start/end with method, route, status, and duration.
- Notable business events (order placed, payment captured, account locked).
- Every handled error, with its cause and enough context to act.
- Boundaries of external calls (which dependency, latency, outcome).

Do **not** log:

- **Secrets or credentials** — tokens, passwords, keys, session IDs.
- **PII beyond what's needed** — full card numbers, government IDs, personal
  data. Redact at the logging layer so it can't leak by accident.
- **High-cardinality noise at INFO** — logging every row of a big loop drowns
  the signal and costs money. Sample or aggregate.

## Metrics

Metrics are cheap, aggregated numbers you can alert on. At minimum capture the
**RED** signals per endpoint/dependency:

- **Rate** — requests per second.
- **Errors** — error rate (and by type/status).
- **Duration** — latency distribution. Track **percentiles (p50/p95/p99)**, not
  averages — an average hides the slow tail where users actually suffer.

For resources, the **USE** signals (Utilization, Saturation, Errors) apply —
e.g. DB connection pool usage, queue depth. Add business metrics that reveal
health faster than infra does (checkout success rate, signups/min). **Alert on
symptoms users feel** (error rate, latency, queue backlog), not on every
internal blip, or alert fatigue sets in and real pages get ignored.

Keep metric label cardinality bounded — never label a metric with a user ID,
request ID, or raw URL; it explodes the time-series database.

## Distributed tracing

When a request crosses services, a **trace** ties the spans together so you can
see where the time and the failure went. Propagate trace context (W3C
`traceparent` / OpenTelemetry) across service boundaries. A trace turns "the
request was slow" into "480ms of the 500ms was in the inventory service's DB
query." Adopt OpenTelemetry rather than a proprietary format so you're not
locked to one backend.

## Errors with context

An error is only useful if it tells you enough to act without reproducing it.
When logging or wrapping an error, include the **operation, the inputs that
matter, and the cause** — and preserve the chain (wrap, don't replace, the
original) so you keep the stack/root cause. `"error occurred"` is useless;
`"charge_card failed for order_id=123: gateway timeout after 30s"` is
actionable. Distinguish *expected* errors (validation failed → not an alert)
from *unexpected* ones (DB unreachable → alert) so noise doesn't bury signal.

## Health checks

Expose endpoints the platform uses to route and heal:

- **Liveness** — is the process alive? (If not, restart it.) Keep it trivial.
- **Readiness** — can it serve traffic right now? Check critical dependencies
  (DB reachable, migrations applied). A failing readiness check pulls the
  instance out of the load balancer without killing it.

Keep health checks cheap and don't let them cascade — a readiness check that
does heavy work becomes its own outage under load.

## Instrumentation checklist

- [ ] Logs are **structured** with consistent field names and sane levels.
- [ ] A **correlation ID** threads through every log, downstream call, and job,
      and is returned to the client.
- [ ] **No secrets or excess PII** in logs.
- [ ] **RED metrics** (rate, errors, duration with percentiles) per endpoint and
      dependency.
- [ ] **Traces** propagate across service boundaries.
- [ ] Errors carry **operation + inputs + cause**, and preserve the chain.
- [ ] **Liveness and readiness** endpoints exist and are cheap.
- [ ] Alerts fire on **user-visible symptoms**, tuned to avoid fatigue.
