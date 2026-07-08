# Contract-First Observability (OBS-01 / OBS-02, RFC-005)

> **Status:** shipped in `waffle-commons/telemetry` + `waffle-commons/telemetry-otel` (+ contracts) since the beta5 cycle (RFC-005, AXE5).
> **Companion pages:** [telemetry reference](../reference/telemetry.md) · [telemetry-otel reference](../reference/telemetry-otel.md) · [How to: Enable Telemetry and Metrics](../how-to/enable-telemetry-and-metrics.md).

This page explains *why* observability in Waffle looks the way it does. For exact signatures, use the references.

## The constraint that shapes everything: the SDK never enters the core

The defining decision of RFC-005 is negative: **no observability SDK is allowed inside the framework core**. A tracing SDK is a large, transitively-heavy dependency with its own release cadence; pulling OpenTelemetry into `waffle-commons/waffle`, `routing`, `security`, `data`, or `http-client` would couple every application to that graph whether it traces or not, and would breach the `mago guard` perimeter (a component may depend only on `waffle-commons/contracts`, plus `utils`). So instead of importing an SDK, the core imports an *interface*.

Every instrumentation point in the framework — the request root span, the outbound HTTP span, the per-query database span — is written against a contract that lives in `waffle-commons/contracts`:

- `Waffle\Commons\Contracts\Telemetry\TracerInterface` — starts spans, exposes the active context;
- `Waffle\Commons\Contracts\Telemetry\SpanInterface` / `SpanContextInterface` — a timed unit of work and its W3C identity;
- `Waffle\Commons\Contracts\Telemetry\TextMapPropagatorInterface` — injects/extracts trace context across HTTP headers;
- `Waffle\Commons\Contracts\Telemetry\Metrics\MetricsRegistryInterface` — counters, gauges, histograms.

Each of these contracts ships with an **inert default that does nothing**: `NullTracer` returns a `NullSpan` whose mutators are empty methods, `NullSpanContext` is the all-zero W3C context, `NullTextMapPropagator` injects and extracts nothing, and `NullMetricsRegistry` records nothing. Because these defaults are the wiring default, instrumentation sprinkled through the hot path costs next to nothing until a real backend is bound. You pay for observability only when you opt in.

## Two backends, one perimeter discipline

There are exactly two implementation packages, and each one is the *sole* importer of its heavy dependency:

- **`waffle-commons/telemetry-otel`** is the only package in the ecosystem that imports `open-telemetry/api` and `open-telemetry/sdk`. It binds the tracing contracts to OpenTelemetry: `OtelTracer`, `OtelSpan`, `OtelSpanContext`, and the `W3CTraceContextPropagator`. Nothing else in the monorepo references an `OpenTelemetry\*` symbol, so the SDK is quarantined to this one boundary.
- **`waffle-commons/telemetry`** is deliberately **SDK-free**: it requires only `waffle-commons/contracts`. It is a from-scratch Prometheus implementation — a `MetricsRegistry`, a set of stateless collectors, a `PrometheusExporter` that renders the text exposition format by hand, and the middleware/decorators that wire metrics into a request.

This split is why metrics and tracing are independent choices. An application can scrape Prometheus metrics with zero SDK dependency (the metrics component alone), turn on distributed tracing by adding the OTel bridge, or do both — and the core never changes either way.

## Tracing: a single trace that crosses service boundaries

A trace is only useful if it follows a request from one service into the next. Waffle achieves end-to-end traces by sharing **one tracer instance** across the three places that open spans, and by speaking the W3C Trace Context wire format at the edges:

1. **Inbound.** `TracingMiddleware` opens the request root span (`http.request`, `SpanKind::Server`). If the inbound request carries a `traceparent` header, the middleware hands it to the propagator's `extract()` and parents the root span on that remote context — so this service *continues* the caller's trace instead of starting a fresh one.
2. **Database.** Each RFC-022 repository emits a `waffle.db.query` `SpanKind::Client` span around its read operations, parented (through the tracer's active context) on the request root. This is **native**, not bolted-on: the `data` component carries its own `QueryTracer` and a `withTracer()` wither on every repository, defaulting to the no-op tracer.
3. **Outbound.** The PSR-18 HTTP client opens an `http.client.request` `SpanKind::Client` span and **injects** the active context's `traceparent` (and `tracestate`) onto the outbound request via the propagator. The downstream service's own `TracingMiddleware` extracts it, and the trace continues.

Because all three use the same injected `TracerInterface`, and because OTel owns the active-context stack inside `OtelTracer`, the spans nest correctly into one tree without any of them holding per-request state of their own.

### Why the propagation is W3C, and why extract delegates to the SDK

The `traceparent` format (`00-<32-hex-trace>-<16-hex-span>-<2-hex-flags>`) is the W3C standard every modern collector understands. `W3CTraceContextPropagator::inject()` serialises the Waffle context through its own `toTraceparent()` — a small, auditable string operation — and only writes a header when the context `isValid()`. `extract()`, conversely, delegates to OpenTelemetry's own audited `TraceContextPropagator` for parsing and validation, because parsing untrusted inbound headers is exactly where you want a battle-tested implementation rather than a hand-rolled one.

## Metrics: stateless collectors over shared memory

The hard problem with metrics under a resident worker is that a counter must *accumulate* across requests, but the worker-mode statelessness mandate forbids cumulative state on the worker heap (that is precisely the kind of growth `igor-php` audits against). Waffle resolves the tension by keeping every cumulative value **out of the worker heap entirely**, in APCu shared memory:

- `MetricsRegistry` (the `MetricsRegistryInterface` implementation) writes every `increment` / `gauge` / `observe` into a `MetricStoreInterface`; the production binding `ApcuMetricStore` keeps those values in APCu, so a single `/waffle-metrics` scrape aggregates the contributions of **every worker on the instance**, and the registry instance itself stays `final readonly` with no mutable field.
- The **collectors** are stateless by a different mechanism: rather than accumulate, they sample live values at scrape time. `MemoryCollector` reads `memory_get_usage()` / `memory_get_peak_usage()`, `GcCollector` reads `gc_status()`, and `PoolUtilizationCollector` reads a bound `PoolStatsInterface`. They hold no state because they read the world fresh on every `collect()`.

`observe()` is stored as a Prometheus-style summary — a `<name>_sum` counter plus a `<name>_count` counter — so a downstream mean (e.g. average request duration) is derivable without keeping a histogram in memory. The metric descriptor (name, type, labels) is JSON-encoded into the store key and reconstructed at export time, so the registry needs no schema registry of its own.

When APCu is unavailable the wiring falls back to the contract `NullMetricsRegistry`, so a missing extension downgrades gracefully to no-op cumulative metrics rather than failing the boot — the live gauges (memory, GC, pool) still render, because those collectors don't need the store.

## The metrics endpoint is fail-closed by design

`/waffle-metrics` exposes operational internals — memory curves, GC pressure, request rates — so it is treated as a sensitive surface, not a public one. `MetricsMiddleware` is **fail-closed**: a request to the path is answered only when it presents the configured bearer token (compared with `hash_equals()`, so the check is constant-time and timing-safe) **or** originates from an allow-listed `REMOTE_ADDR`. Anything else receives a `404` — not a `401` or `403` — so the endpoint's very existence is never confirmed to an unauthorised caller (the same information-hiding discipline as AXE0 `LEAK-03`). Every other path passes straight through untouched.

The default wiring in the template apps allow-lists only `127.0.0.1` and `::1` with no bearer token, so out of the box the endpoint is reachable from the local host only; granting a remote Prometheus a bearer token is a deliberate, explicit act.

## Worker safety: the only mutable state is external

The whole telemetry surface passes the `igor-php` worker-mode audit with zero findings, and it does so structurally:

- the tracing adapters (`OtelTracer`, `OtelSpan`, `OtelSpanContext`, the propagator) are `final readonly` and delegate the active-context stack to OTel, holding none of their own;
- the metrics registry, collectors, exporter, middlewares, and decorators are all `final readonly` with no accumulating field;
- the *only* cumulative state — the counters — lives in APCu, outside the worker heap, where it both survives across requests (as a counter must) and cannot leak request A's data into request B.

`Coerce`, a tiny type-narrowing helper, exists so the inherently-`mixed` returns of `apcu_fetch()` and `json_decode()` are narrowed with `is_*()` guards rather than cast — keeping the analyzer clean without a single suppression, per the zero-baseline mandate.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
