# Telemetry OTel Bridge Reference (`waffle-commons/telemetry-otel`)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *The sole OpenTelemetry-SDK importer; binds the tracing contracts (OBS-01, RFC-005, AXE5)*
> **Requires:** PHP 8.5+, `open-telemetry/api` (`^1.0`), `open-telemetry/sdk` (`^1.0`). Depends on `waffle-commons/contracts`.

The OpenTelemetry binding for Waffle's tracing contracts. This is the **only** package in the ecosystem that imports an `OpenTelemetry\*` symbol — the framework core and the demo apps only ever see `Waffle\Commons\Contracts\Telemetry\TracerInterface` and friends, so the SDK is quarantined to this perimeter (the `mago guard` rule that keeps every other component on `contracts` alone). It is **opt-in**: an application adds it as a dependency only when it wants distributed tracing; metrics (Prometheus) are a separate, SDK-free choice in [`waffle-commons/telemetry`](telemetry.md).

For the architectural reasoning (why the SDK never enters the core, how one shared tracer yields one end-to-end trace), see [Explanation: Contract-First Observability](../explanation/observability-telemetry.md).

## Tracer adapter — `Waffle\Commons\TelemetryOtel\Trace\OtelTracer`

`final readonly` implementation of `Waffle\Commons\Contracts\Telemetry\TracerInterface`.

```php
public function __construct(private OtelTracerInterface $tracer); // OpenTelemetry\API\Trace\TracerInterface

public function startSpan(string $name, SpanKind $kind = SpanKind::Internal, ?SpanContextInterface $parent = null): SpanInterface;
public function currentContext(): ?SpanContextInterface;
```

- `startSpan()` builds an OTel span (an empty name is replaced with `'span'`), mapping the Waffle `SpanKind` onto `OtelSpanKind::KIND_*` (`Internal`/`Server`/`Client`/`Producer`/`Consumer`). The span is **activated** on start, so nested spans and `currentContext()` compose through OTel's context stack. The returned `OtelSpan` carries the active scope.
- When an explicit `$parent` context is supplied (e.g. one extracted from an inbound W3C `traceparent`), the new span continues that remote trace — rebuilt via `SpanContext::createFromRemoteParent()` from the parent's trace id, span id, flags, and (optional) tracestate. Otherwise the span parents off OTel's current active context.
- `currentContext()` returns an `OtelSpanContext` wrapping OTel's current span context, or `null` when that context is invalid.
- **Stateless:** OTel owns the active-context stack, so this wrapper carries no per-request state and is safe across resident-worker requests.

## Span adapter — `Waffle\Commons\TelemetryOtel\Trace\OtelSpan`

`final readonly` implementation of `Waffle\Commons\Contracts\Telemetry\SpanInterface`.

```php
public function __construct(
    private OtelSpanInterface $span,   // OpenTelemetry\API\Trace\SpanInterface
    private ScopeInterface $scope,     // OpenTelemetry\Context\ScopeInterface — the active scope
);
```

- `setAttribute()` forwards to the OTel span (an empty key is ignored).
- `recordException()` forwards the throwable; `setStatus()` maps `SpanStatus::{Ok,Error,Unset}` onto `StatusCode::STATUS_{OK,ERROR,UNSET}`.
- `context()` returns an `OtelSpanContext`.
- `end()` **detaches the active scope, then ends the span** — in that order — so the context stack unwinds cleanly.

## Span-context adapter — `Waffle\Commons\TelemetryOtel\Trace\OtelSpanContext`

`final readonly` implementation of `Waffle\Commons\Contracts\Telemetry\SpanContextInterface`, wrapping `OpenTelemetry\API\Trace\SpanContextInterface`.

```php
public function __construct(private OtelSpanContextInterface $context);

public function traceId(): string;       // OTel getTraceId()
public function spanId(): string;        // OTel getSpanId()
public function traceFlags(): int;       // OTel getTraceFlags()
public function traceState(): string;    // '' when the OTel tracestate is null
public function isValid(): bool;         // OTel isValid()
public function toTraceparent(): string; // '00-<trace>-<span>-<2-hex-flags>'
```

## W3C propagator — `Waffle\Commons\TelemetryOtel\Propagation\W3CTraceContextPropagator`

`final readonly` implementation of `Waffle\Commons\Contracts\Telemetry\TextMapPropagatorInterface`.

```php
/** @param array<string, string> $carrier  @param-out array<string, string> $carrier */
public function inject(SpanContextInterface $context, array &$carrier): void;
/** @param array<string, string> $carrier */
public function extract(array $carrier): ?SpanContextInterface;
```

- `inject()` writes nothing when the context is invalid; otherwise it sets `traceparent` from the context's own `toTraceparent()`, and `tracestate` only when the trace state is non-empty.
- `extract()` delegates to OpenTelemetry's audited `TraceContextPropagator` (via `ArrayAccessGetterSetter`) for robust parsing and validation, returning an `OtelSpanContext` only when the parsed context is valid (else `null`).

This is the propagator the HTTP client uses to inject `traceparent` on outbound calls and that `TracingMiddleware` uses to extract it on inbound requests — so a trace continues across service boundaries.

## Factory — `Waffle\Commons\TelemetryOtel\Factory\OtelTracerFactory`

Assembles a ready-to-use `OtelTracer` from the OpenTelemetry SDK, keeping every `OpenTelemetry\SDK\*` reference inside this perimeter.

```php
public static function create(string $serviceName, SpanExporterInterface $exporter): OtelTracer;
public static function console(string $serviceName): OtelTracer;
```

- `create()` tags every span with `service.name = $serviceName` (an OTel `ResourceInfo`) and exports through the supplied span exporter (e.g. an OTLP exporter in production, an in-memory one in tests). It uses a **`SimpleSpanProcessor`** on purpose: each span is exported synchronously on `end()`, which keeps a FrankenPHP worker's stdout deterministic (no background batch flush to lose). The tracer is obtained from a `TracerProvider` named `waffle-commons/telemetry-otel`.
- `console()` is the local/dev convenience: it exports spans as JSON to `php://stdout` (a `ConsoleSpanExporter` over a `stream` transport), so a container's logs become a zero-dependency span collector — `docker logs <service>`. This is exactly what the template apps wire when `WAFFLE_TRACING=otel`.

## Wiring (template apps)

The workspace app builds the bridge behind an environment toggle (the skeleton ships the no-op default and a comment showing the swap):

```php
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

The **one** `$tracer` is then injected into the HTTP client, the `TracingMiddleware`, and any tracing repositories — so the outbound span, the request root span, and the database spans all join the same end-to-end trace. See [How to: Enable Telemetry and Metrics](../how-to/enable-telemetry-and-metrics.md).

## Worker-safety contract

Every adapter is `final readonly` and delegates the active-context stack to the OpenTelemetry context, holding no per-request state of its own; the component passes the `igor-php` worker-mode audit with zero findings.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
