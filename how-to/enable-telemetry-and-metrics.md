# How to Enable Telemetry and Metrics

Waffle's observability ships in two independent, opt-in packages (RFC-005). `waffle-commons/telemetry` is an SDK-free Prometheus implementation: it exposes a fail-closed `/waffle-metrics` scrape endpoint and records request/cache/database metrics. `waffle-commons/telemetry-otel` is the OpenTelemetry bridge for distributed tracing — the only package that pulls in the OTel SDK. The framework core depends on neither; it speaks only the tracing/metrics contracts, whose no-op defaults make instrumentation free until you wire a real backend. This recipe enables both.

The template apps already wire all of this — what follows is the wiring you would write yourself (or read when adapting theirs).

## 1. Add the components

Metrics alone need only the SDK-free package:

```bash
composer require waffle-commons/telemetry
```

Add the OTel bridge **only if you want distributed tracing**:

```bash
composer require waffle-commons/telemetry-otel
```

For cumulative counters (request totals, cache hits), enable APCu — including under the worker SAPI:

```ini
; php.ini / a conf.d drop-in
apc.enabled=1
apc.enable_cli=1
```

Without APCu the wiring falls back to the no-op `NullMetricsRegistry`: live gauges (memory, GC, pool) still render, but counters stay flat.

## 2. Wire the metrics registry and the `/waffle-metrics` endpoint

Register a shared `MetricsRegistryInterface`, then install `MetricsMiddleware` early in the pipeline (it short-circuits `/waffle-metrics` before the application pipeline and applies its own fail-closed security):

```php
use Waffle\Commons\Contracts\Telemetry\Metrics\MetricsCollectorInterface;
use Waffle\Commons\Contracts\Telemetry\Metrics\MetricsRegistryInterface;
use Waffle\Commons\Contracts\Telemetry\Metrics\NullMetricsRegistry;
use Waffle\Commons\Telemetry\Collector\GcCollector;
use Waffle\Commons\Telemetry\Collector\MemoryCollector;
use Waffle\Commons\Telemetry\Collector\PoolUtilizationCollector;
use Waffle\Commons\Telemetry\Exporter\PrometheusExporter;
use Waffle\Commons\Telemetry\Metric\ApcuMetricStore;
use Waffle\Commons\Telemetry\Metric\MetricsRegistry;
use Waffle\Commons\Telemetry\Middleware\MetricsMiddleware;

$metricsRegistry = apcu_enabled()
    ? new MetricsRegistry(new ApcuMetricStore())
    : new NullMetricsRegistry();
$container->set(MetricsRegistryInterface::class, $metricsRegistry);

// Stateless collectors sampled at scrape time; the registry is also a collector
// (it exports the request/cache counters it has accumulated in APCu).
$collectors = [new MemoryCollector(), new GcCollector(), new PoolUtilizationCollector()];
if ($metricsRegistry instanceof MetricsCollectorInterface) {
    $collectors[] = $metricsRegistry;
}

$stack->add(middleware: new MetricsMiddleware(
    new PrometheusExporter($collectors),
    $responseFactory,   // PSR-17 ResponseFactoryInterface
    $streamFactory,     // PSR-17 StreamFactoryInterface
    null,               // bearer token — null = no token auth
    ['127.0.0.1', '::1'],
));
```

`/waffle-metrics` is **fail-closed**: a scrape is served only when it presents `Authorization: Bearer <token>` (constant-time `hash_equals`) **or** comes from an allow-listed `REMOTE_ADDR`. Anything else gets a `404`, so the endpoint's existence is never confirmed to an unauthorised caller. With this wiring, only the local host can scrape. To allow a remote Prometheus, pass a bearer token instead of (or in addition to) IPs:

```php
new MetricsMiddleware($exporter, $responseFactory, $streamFactory, $bearerToken, []);
```

## 3. Record request metrics with `TracingMiddleware`

Install `TracingMiddleware` so every request records `waffle_http_requests_total{method,status}` and the `waffle_http_request_duration_seconds` summary — and opens the per-request root span (a no-op until you wire a real tracer in step 5):

```php
use Waffle\Commons\Telemetry\Middleware\TracingMiddleware;

$stack->add(middleware: new TracingMiddleware($tracer, $metricsRegistry, $tracePropagator));
```

## 4. Scrape the endpoint

```bash
# from the local host (IP allow-list), or with a bearer token from anywhere
curl -s http://127.0.0.1:8080/waffle-metrics
curl -s -H 'Authorization: Bearer <token>' https://your-app/waffle-metrics
```

You get Prometheus text exposition (v0.0.4), e.g.:

```
# HELP waffle_memory_usage_bytes Current real memory used by the PHP worker, in bytes.
# TYPE waffle_memory_usage_bytes gauge
waffle_memory_usage_bytes 12582912
# TYPE waffle_http_requests_total counter
waffle_http_requests_total{method="GET",status="200"} 41
# TYPE waffle_gc_runs_total counter
waffle_gc_runs_total 7
```

Point a Prometheus `scrape_config` at the path:

```yaml
scrape_configs:
  - job_name: 'waffle'
    metrics_path: /waffle-metrics
    authorization:
      credentials: '<bearer-token>'   # omit if scraping from an allow-listed IP
    static_configs:
      - targets: ['your-app:8080']
```

## 5. Turn on distributed tracing (OTel)

Tracing is off by default — the no-op `NullTracer` adds ~zero overhead and emits no headers. Build the OTel bridge behind an environment toggle and inject the **one** tracer everywhere spans are opened:

```php
use Waffle\Commons\Contracts\Telemetry\NullTextMapPropagator;
use Waffle\Commons\Contracts\Telemetry\NullTracer;
use Waffle\Commons\Contracts\Telemetry\TracerInterface;
use Waffle\Commons\TelemetryOtel\Factory\OtelTracerFactory;
use Waffle\Commons\TelemetryOtel\Propagation\W3CTraceContextPropagator;

$tracer = new NullTracer();
$tracePropagator = new NullTextMapPropagator();
if (getenv('WAFFLE_TRACING') === 'otel') {
    $otelService = getenv('WAFFLE_SERVICE_NAME');
    $tracer = OtelTracerFactory::console(
        is_string($otelService) && $otelService !== '' ? $otelService : 'waffle',
    );
    $tracePropagator = new W3CTraceContextPropagator();
}
$container->set(TracerInterface::class, $tracer);
```

| Environment variable | Effect |
| :--- | :--- |
| `WAFFLE_TRACING=otel` | Mounts the OpenTelemetry bridge (`OtelTracerFactory::console`) + the W3C propagator. Any other value keeps the no-op default. |
| `WAFFLE_SERVICE_NAME` | The `service.name` tag on every span (defaults to `waffle`). |

`OtelTracerFactory::console()` exports each span as JSON to `php://stdout`, so a container's logs become a zero-dependency span collector (`docker logs <service>`). For production, build the tracer with `OtelTracerFactory::create($serviceName, $exporter)` and pass an OTLP exporter instead.

### Share one tracer for an end-to-end trace

The same `$tracer` must reach all three span sources so they nest into a single trace:

```php
// outbound HTTP client — injects `traceparent` onto outbound requests
$httpClient = new Client($responseFactory, $streamFactory, $ssrfGuard, $tracer, $tracePropagator);

// per-request root span (already wired in step 3)
$stack->add(middleware: new TracingMiddleware($tracer, $metricsRegistry, $tracePropagator));

// native DB query spans — every RFC-022 repository has a withTracer() wither
$users = (new SQLRepository($pool, UserRow::class, $compiler))->withTracer($tracer);
```

With this, an inbound request whose `traceparent` is extracted by `TracingMiddleware`, the `waffle.db.query` spans from the repositories, and the `http.client.request` span (which injects `traceparent` downstream) all join one distributed trace. The workspace app ships a two-service `waffle-upstream` → `waffle-downstream` demo (`WAFFLE_TRACING=otel`, `WAFFLE_SERVICE_NAME` per service) that proves the trace crosses the service boundary over `docker logs`.

## 6. (Optional) instrument the cache

Wrap any Waffle/PSR-16 cache to feed the cache-hit-ratio metrics:

```php
use Waffle\Commons\Telemetry\Cache\MeteredCache;

$cache = new MeteredCache($innerCache, $metricsRegistry);
// get() now records waffle_cache_hits_total / waffle_cache_misses_total
```

## How it works

- **Contract-first:** the core imports only the contracts; the OTel SDK lives solely in `telemetry-otel`, and the metrics package is SDK-free. See [Explanation: Contract-First Observability](../explanation/observability-telemetry.md).
- **Worker-safe metrics:** cumulative counters live in APCu shared memory (`ApcuMetricStore`), never on the worker heap, so one scrape aggregates every worker and `igor-php` stays clean.
- **W3C propagation:** outbound calls inject `traceparent`; `TracingMiddleware` extracts it inbound, so a trace continues across services.

See the [telemetry reference](../reference/telemetry.md) and the [telemetry-otel reference](../reference/telemetry-otel.md) for the full API surface.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
