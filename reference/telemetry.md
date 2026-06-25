# Telemetry Reference (`waffle-commons/telemetry`)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *SDK-free Prometheus metrics + tracing wiring (OBS-02, RFC-005, AXE5)*
> **Requires:** PHP 8.5+. Depends only on `waffle-commons/contracts`.
> **Suggests:** `ext-apcu` (with `apc.enable_cli` under the worker SAPI) for cumulative counters; without it the wiring falls back to the contract `NullMetricsRegistry`.

The SDK-free half of Waffle's observability stack: a Prometheus metrics implementation written from scratch (no OpenTelemetry, no third-party metrics library), the fail-closed `/waffle-metrics` scrape endpoint, the per-request tracing/metrics middleware, and the cache/repository instrumentation decorators. Distributed tracing is bound separately by the [`waffle-commons/telemetry-otel`](telemetry-otel.md) bridge; the tracing contracts and their no-op defaults live in [contracts](contracts.md).

For the architectural reasoning (why contract-first, why the SDK never enters the core, why metrics live in APCu), see [Explanation: Contract-First Observability](../explanation/observability-telemetry.md).

## Tracing contracts (in `waffle-commons/contracts`)

The framework core depends only on these interfaces and their inert defaults — they are documented here because they are the surface the telemetry components implement and the apps wire.

### `Waffle\Commons\Contracts\Telemetry\TracerInterface`

```php
public function startSpan(
    string $name,
    SpanKind $kind = SpanKind::Internal,
    ?SpanContextInterface $parent = null,
): SpanInterface;
public function currentContext(): ?SpanContextInterface;
```

When `$parent` is null the current active context is used as the parent; otherwise the span starts from the supplied (typically remote) context. The default `Waffle\Commons\Contracts\Telemetry\NullTracer` returns a `NullSpan` and never has an active context.

### `Waffle\Commons\Contracts\Telemetry\SpanInterface`

```php
public function setAttribute(string $key, string|int|float|bool $value): void; // OTel semantic-convention key
public function recordException(Throwable $exception): void;                    // does NOT end the span
public function setStatus(SpanStatus $status): void;
public function context(): SpanContextInterface;
public function end(): void;                                                    // idempotent
```

The idiom is `try { … } finally { $span->end(); }`. Default `NullSpan` — every method is a no-op.

### `Waffle\Commons\Contracts\Telemetry\SpanContextInterface`

The immutable W3C identity of a span: `traceId(): string` (32-hex), `spanId(): string` (16-hex), `traceFlags(): int`, `traceState(): string`, `isValid(): bool`, and `toTraceparent(): string` (`00-<trace>-<span>-<flags>`). Default `NullSpanContext` is the all-zero, never-valid context.

### `Waffle\Commons\Contracts\Telemetry\TextMapPropagatorInterface`

```php
/** @param array<string, string> $carrier  @param-out array<string, string> $carrier */
public function inject(SpanContextInterface $context, array &$carrier): void;
/** @param array<string, string> $carrier */
public function extract(array $carrier): ?SpanContextInterface;
```

Default `NullTextMapPropagator` injects nothing and extracts nothing. The real W3C propagator ships in [telemetry-otel](telemetry-otel.md).

### Enums

- `Waffle\Commons\Contracts\Telemetry\Enum\SpanKind: string` — `Internal 'internal'` (default), `Server 'server'`, `Client 'client'`, `Producer 'producer'`, `Consumer 'consumer'`.
- `Waffle\Commons\Contracts\Telemetry\Enum\SpanStatus: string` — `Unset 'unset'` (default), `Ok 'ok'`, `Error 'error'`.

## Metrics contracts (in `waffle-commons/contracts`)

### `Waffle\Commons\Contracts\Telemetry\Metrics\MetricsRegistryInterface`

```php
/** @param array<string, string> $labels */
public function increment(string $name, float $value = 1.0, array $labels = []): void; // monotonic counter
public function observe(string $name, float $value, array $labels = []): void;          // histogram/summary observation
public function gauge(string $name, float $value, array $labels = []): void;            // absolute point-in-time value
```

Implementations MUST keep cumulative state out of the resident worker heap (e.g. APCu). Default `Waffle\Commons\Contracts\Telemetry\Metrics\NullMetricsRegistry` — records nothing.

### `Waffle\Commons\Contracts\Telemetry\Metrics\MetricsCollectorInterface`

```php
/** @return iterable<MetricSample> */
public function collect(): iterable;
```

A scrape-time source of samples. Collectors are stateless: they read live values when `collect()` is called.

### `Waffle\Commons\Contracts\Telemetry\Metrics\MetricSample`

`final readonly` — one export-ready reading:

```php
public function __construct(
    public string $name,
    public MetricType $type,
    public float $value,
    public array $labels = [],     // array<string, string>
    public string $help = '',
);
```

### `Waffle\Commons\Contracts\Telemetry\Metrics\PoolStatsInterface`

Live connection-pool utilisation, sampled for the endpoint: `activeLeases(): int`, `idle(): int`, `capacity(): int`. Until a real pool is bound, the collector reports zeros.

### `Waffle\Commons\Contracts\Telemetry\Metrics\Enum\MetricType: string`

`Counter 'counter'` · `Gauge 'gauge'` · `Histogram 'histogram'`.

## Metrics registry — `Waffle\Commons\Telemetry\Metric\MetricsRegistry`

`final readonly` implementation of **both** `MetricsRegistryInterface` and `MetricsCollectorInterface`.

```php
public function __construct(private MetricStoreInterface $store);

public function increment(string $name, float $value = 1.0, array $labels = []): void; // store->add(Counter key)
public function gauge(string $name, float $value, array $labels = []): void;            // store->set(Gauge key)
public function observe(string $name, float $value, array $labels = []): void;          // <name>_sum + <name>_count counters
public function collect(): iterable;                                                    // store snapshot → MetricSamples
```

- Stateless: every cumulative value lives in the injected `MetricStoreInterface`, never on the worker heap, so one scrape reflects every worker.
- `observe()` is recorded as a **summary** — a `<name>_sum` (Counter) plus a `<name>_count` (Counter) — so a mean is derivable downstream.
- The metric descriptor (name, type, labels) is JSON-encoded into the store key (labels `ksort`ed for a stable key), so it can be faithfully reconstructed during `collect()`; a malformed key is silently dropped.

## Metric store — `Waffle\Commons\Telemetry\Metric\MetricStoreInterface`

```php
public function add(string $key, float $delta): void; // absent key treated as 0
public function set(string $key, float $value): void;
/** @return array<string, float> */
public function snapshot(): array;
```

Shared, worker-safe storage (opaque key → float). The production binding keeps values in instance-shared memory, never on the resident worker heap, so counters survive across requests.

### `Waffle\Commons\Telemetry\Metric\ApcuMetricStore`

`final readonly` APCu binding. Counters live in the host's shared memory; a small `__index__` key tracks the live key set so `snapshot()` needs no `APCUIterator`.

```php
public function __construct(private string $prefix = 'wfl_metric:');
```

- Requires `ext-apcu` (with `apc.enable_cli` under the worker SAPI). When APCu is unavailable, the app wiring should fall back to `NullMetricsRegistry`.
- `add()` is a read-modify-write (intentionally non-atomic — metrics are approximate); the only instance field is the immutable prefix, so the store is stateless from the worker's view.

## Collectors (`Waffle\Commons\Telemetry\Collector\…`)

All `final readonly`, all `MetricsCollectorInterface`. They sample live values at scrape time and hold no state.

| Collector | Constructor | Emitted samples |
| :--- | :--- | :--- |
| `MemoryCollector` | `()` | `waffle_memory_usage_bytes` (Gauge), `waffle_memory_peak_bytes` (Gauge) — real memory via `memory_get_usage(true)` / `memory_get_peak_usage(true)` |
| `GcCollector` | `()` | `waffle_gc_runs_total` (Counter), `waffle_gc_collected_total` (Counter), `waffle_gc_roots` (Gauge) — from `gc_status()` |
| `PoolUtilizationCollector` | `(?PoolStatsInterface $pool = null)` | `waffle_db_pool_active` (Gauge), `waffle_db_pool_idle` (Gauge), `waffle_db_pool_capacity` (Gauge) — zeros until a pool is bound |

The request middleware also feeds dynamic series into the registry (which is itself a collector):

- `waffle_http_requests_total` (Counter, labels `method`, `status`);
- `waffle_http_request_duration_seconds` summary (`_sum` + `_count`, label `method`).

## Exporter — `Waffle\Commons\Telemetry\Exporter\PrometheusExporter`

`final readonly`. Renders the samples of a set of collectors into the Prometheus text exposition format (v0.0.4) — no SDK required.

```php
public const string CONTENT_TYPE = 'text/plain; version=0.0.4; charset=utf-8';

/** @param iterable<MetricsCollectorInterface> $collectors */
public function __construct(private iterable $collectors);

public function render(): string;
```

`# HELP` / `# TYPE` are emitted once per metric name; labels are escaped; values are rendered in fixed notation with trailing zeros trimmed (`42.0` → `42`, `0.012000` → `0.012`). Stateless.

## Metrics endpoint middleware — `Waffle\Commons\Telemetry\Middleware\MetricsMiddleware`

`final readonly`, PSR-15. Serves the scrape endpoint; **fail-closed**.

```php
public const string PATH = '/waffle-metrics';

/** @param list<string> $allowedIps  Exact REMOTE_ADDR values permitted to scrape. */
public function __construct(
    private PrometheusExporter $exporter,
    private ResponseFactoryInterface $responseFactory,   // PSR-17
    private StreamFactoryInterface $streamFactory,       // PSR-17
    #[\SensitiveParameter] private ?string $bearerToken = null,
    private array $allowedIps = [],
);
```

- A request to `/waffle-metrics` is answered (`200`, `Content-Type: text/plain; version=0.0.4`) only when it presents `Authorization: Bearer <token>` (compared with `hash_equals()`) **or** comes from an allow-listed `REMOTE_ADDR`. Otherwise it returns **`404`** — the endpoint's existence is never revealed (mirrors AXE0 `LEAK-03`).
- With both `bearerToken === null` and `allowedIps === []`, no request is ever authorised — fail-closed by construction.
- Every other path passes straight through to `$handler`. Stateless.

## Tracing/metrics middleware — `Waffle\Commons\Telemetry\Middleware\TracingMiddleware`

`final readonly`, PSR-15. Opens the per-request root span and records request count + duration.

```php
public function __construct(
    private TracerInterface $tracer = new NullTracer(),
    private MetricsRegistryInterface $metrics = new NullMetricsRegistry(),
    private TextMapPropagatorInterface $propagator = new NullTextMapPropagator(),
);
```

- Opens an `http.request` span (`SpanKind::Server`), tagged `http.request.method` and `url.path`. If the inbound request carries a `traceparent` header it is handed to `propagator->extract()` and used as the parent, so this service continues the caller's distributed trace.
- On the way out it sets `http.response.status_code` and a `SpanStatus` (`Error` for `>= 500`, else `Ok`); on a thrown error it records the exception, sets `Error`, and re-throws — the span is always `end()`ed in `finally`.
- Records `waffle_http_requests_total{method,status}` and observes `waffle_http_request_duration_seconds{method}`. Safe to install with the no-op defaults; cumulative state lives in the registry's shared store.

## Cache instrumentation — `Waffle\Commons\Telemetry\Cache\MeteredCache`

`final readonly` decorator over any `Waffle\Commons\Contracts\Cache\CacheInterface` (Waffle/PSR-16), recording cache hit/miss counters for the hit-ratio.

```php
public function __construct(
    private CacheInterface $inner,
    private MetricsRegistryInterface $metrics = new NullMetricsRegistry(),
);
```

`get()` judges hit vs miss with `has()` (so the stored value is never captured into a `mixed` local), incrementing `waffle_cache_hits_total` or `waffle_cache_misses_total`; every other PSR-16 operation delegates untouched. Stateless.

## Repository instrumentation — `Waffle\Commons\Telemetry\Repository\TracingRepositoryDecorator`

`final readonly` decorator over any RFC-022 `RepositoryInterface<T>`, emitting a `waffle.db.query` `SpanKind::Client` span around each read (`find`, `findOne`, `stream`).

```php
/**
 * @template T of object
 * @implements RepositoryInterface<T>
 * @param RepositoryInterface<T> $inner
 */
public function __construct(
    private RepositoryInterface $inner,
    private TracerInterface $tracer = new NullTracer(),
    private string $system = 'sql',
);
```

Each span is tagged `db.system` and `db.operation`; a failure records the exception, sets `SpanStatus::Error`, and re-throws (the span is always `end()`ed). Stateless and holds no record state.

> **Note — native vs decorator:** the `data` component already traces its repositories *natively* via its own `QueryTracer` + a `withTracer(TracerInterface)` wither on every repository (see the [data reference](data.md)), so `TracingRepositoryDecorator` is the alternative for wrapping a repository you cannot reconfigure — both emit the same `waffle.db.query` CLIENT span.

## Support — `Waffle\Commons\Telemetry\Support\Coerce`

A tiny static type-coercion helper for the inherently-`mixed` returns of APCu and `json_decode()`. Each method narrows with `is_*()` (or `array_filter` with a typed predicate) so callers never assign or cast `mixed` directly — keeping the analyzer clean without suppressions: `toFloat()`, `toString()`, `toArray()`, `toStringList()`, `toStringMap()`.

## Worker-safety contract

Every class in the component is `final readonly` with no accumulating field, and the only cumulative state — the metric counters — lives in APCu, outside the worker heap. The component passes the `igor-php` worker-mode audit with zero findings.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
