# Pipeline Reference (`waffle-commons/pipeline`)

> **Release:** `0.1.0-beta6`
> **PSR Compliance:** PSR-15 (`Psr\Http\Server\MiddlewareInterface`, `RequestHandlerInterface`)

The PSR-15 middleware stack that runs every request through the kernel. The stack locks itself the moment it is converted to a handler, so middleware order cannot be tampered with mid-request.

## `MiddlewareStack`

```php
namespace Waffle\Commons\Pipeline;

final class MiddlewareStack implements MiddlewareStackInterface
{
    public private(set) array $middlewares = [];   // PHP 8.5 asymmetric visibility

    public function add(MiddlewareInterface $middleware): static;       // fluent
    public function prepend(MiddlewareInterface $middleware): static;   // fluent
    public function getMiddlewares(): array;
    public function createHandler(RequestHandlerInterface $fallbackHandler): RequestHandlerInterface;
}
```

`public private(set)` on `$middlewares` exposes the list for read-only inspection (tests, debug pages) while restricting mutation to the `add()` / `prepend()` methods.

### Locking semantics

`createHandler()` flips the internal `$locked = true` flag and returns a `Waffle\Commons\Pipeline\RequestHandler`. Any subsequent `add()` / `prepend()` raises:

```
RuntimeException("MiddlewareStack is locked and cannot be modified during request processing.")
```

This guarantees that middleware order cannot change while a request is in flight (important under FrankenPHP worker mode where the stack is shared across requests).

## Built-in middleware

### `Waffle\Commons\Pipeline\CoreRoutingMiddleware`

```php
final readonly class CoreRoutingMiddleware implements MiddlewareInterface
{
    public function __construct(
        private RouterInterface $router,
        private ?ResponseFactoryInterface $responseFactory = null,  // PSR-17; enables auto-OPTIONS
    );
}
```

Routes the request by calling `RouterInterface::matchRequest()` (which returns the immutable `MatchedRoute` DTO since Beta-1 — see [`reference/routing`](routing.md#matchedroute-dto)), then exposes the result on the request attributes:

- `_classname` — FQCN of the resolved controller class.
- `_method` — method name to invoke.
- `_arguments` — `array<string, mixed>` of per-argument metadata from `#[Argument]`.
- `_path` — original route path pattern.
- `_name` — route name from `#[Route(name: ...)]`.
- `_params` — `array<string, mixed>` of `{placeholder}` values extracted from the URI. Named by `Constant::ATTR_PARAMS` (added in 0.1.0-beta7).

When no route matches, raises `Waffle\Commons\Contracts\Routing\Exception\RouteNotFoundException` (the concrete `final` class added in Beta-1, which implements `RouteNotFoundExceptionInterface` and is rendered as RFC 7807 `404` by `JsonErrorRenderer`). Beta-0 raised a generic `RuntimeException` here and produced a `500`; that mapping has been fixed.

**OPTIONS preflight.** When constructed with a PSR-17 `$responseFactory` (the default app wiring passes one), an `OPTIONS` request to a known path that declares no explicit `OPTIONS` handler is answered with `204 No Content` + the `Allow` header the router computed — intercepting the `MethodNotAllowedException` here so a routine preflight never reaches the logger. A real method mismatch (any non-OPTIONS verb), or OPTIONS when no factory is wired, re-throws the `405` for `ErrorHandlerMiddleware` to render. See [routing → HTTP Method Filtering](routing.md#http-method-filtering--route-overloading).

### `Waffle\Commons\Pipeline\Middleware\TrustedHostMiddleware`

```php
final readonly class TrustedHostMiddleware implements MiddlewareInterface
{
    /** @param list<string> $trustedHosts */
    public function __construct(array $trustedHosts);
}
```

Compares `$request->getUri()->getHost()` (lowercased) against the configured allowlist (lowercased at construction). An empty allowlist disables the check (DEV-only convenience). Missing or untrusted Host raises `\InvalidArgumentException`, which the error handler renders as RFC 7807 `400`.

The exception message does **not** include the allowlist contents — to avoid information disclosure.

### `Waffle\Commons\Pipeline\Middleware\SecureHeadersMiddleware`

```php
final readonly class SecureHeadersMiddleware implements MiddlewareInterface
{
    public function __construct(
        private string $contentSecurityPolicy = "default-src 'self'",
    );
}
```

Adds the baseline security response headers to every outgoing response: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, and `Content-Security-Policy` (the policy string is the single constructor parameter). Note that bare-string controller responses carry their own stricter CSP, set by `ControllerResponseConverter` (`default-src 'self'; form-action 'self'; base-uri 'self'`) — see [core.md](core.md).

## Orchestrator probes (OPS-01/02)

*Added in `0.1.0-beta7` (OPS-01/02, RFC-023).* Guide:
[How to: Run on Kubernetes](../how-to/run-on-kubernetes.md).

### `Waffle\Commons\Pipeline\Middleware\ProbeMiddleware`

```php
final readonly class ProbeMiddleware implements MiddlewareInterface
{
    public const string LIVENESS_PATH = '/healthz';
    public const string READINESS_PATH = '/readyz';

    /** @param list<string> $allowedIps Exact `REMOTE_ADDR` values that get the per-probe report. */
    public function __construct(
        private ReadinessProbe $readiness,
        private ResponseFactoryInterface $responseFactory,
        private StreamFactoryInterface $streamFactory,
        #[\SensitiveParameter]
        private ?string $bearerToken = null,
        private array $allowedIps = [],
    ) {}
}
```

- `/healthz` → `200 {"status":"ok"}`, always: liveness never checks a dependency.
- `/readyz` → runs the `ReadinessProbe`; `200 {"status":"ok"}` when ready, `503 {"status":"unavailable"}`
  otherwise. A request carrying the bearer token, or from an allow-listed `REMOTE_ADDR`, gets the detailed
  report instead: `status`, `draining`, `checks` (each with `status`, `critical`, `duration_ms`, `detail`)
  and `missing`.
- Responses are `application/json` with `Cache-Control: no-store`. Every other path passes through.
- Mount it before `TrustedHostMiddleware` (kubelet probes carry the pod IP as `Host`) and before the rate
  limiter.

### `Waffle\Commons\Pipeline\Health\ReadinessProbe`

```php
final readonly class ReadinessProbe
{
    public const string DURATION_METRIC = 'waffle_health_check_duration_seconds';
    public const string UP_METRIC = 'waffle_health_check_up';

    /**
     * @param list<HealthCheckInterface> $checks      The registered probes; their names must be unique.
     * @param list<string>               $required    The names in `waffle.health.required`.
     * @param string|null                $drainMarker The file whose presence means "draining"
     *                                                (`waffle.health.drain_marker`); null disables it.
     *
     * @throws InvalidArgumentException When two probes share a name.
     */
    public function __construct(
        private array $checks = [],
        private array $required = [],
        private ?MetricsRegistryInterface $metrics = null,
        private ?string $drainMarker = null,
    );

    public function run(): ReadinessReport;
}
```

- While the drain marker exists, `run()` reports not ready (`draining: true`) without running any probe
  (OPS-02): the pod's `preStop` hook creates the file before SIGTERM.
- Otherwise every probe runs, timed here: `waffle_health_check_duration_seconds{check}` and
  `waffle_health_check_up{check}`. A probe that throws counts as down.
- A probe is critical when it says so (`isCritical()`) or when `$required` names it; a required name with no
  registered probe is reported in `missing` and makes the instance not ready. Nothing is cached between two
  calls.

The probes shipped by components are `Data\Health\ConnectionPoolHealthCheck` (`db.pool`, critical),
`Cache\Health\CacheHealthCheck` (`cache`, not critical) and `Queue\Health\StreamsHealthCheck` (`queue`,
critical); the contract is in [contracts.md](contracts.md#health-checks-ops-01).

## Conventional ordering (Beta-1)

```php
$stack = (new MiddlewareStack())
    ->add(new ErrorHandlerMiddleware($renderer, $logger))               // 1. outermost (catches everything)
    ->add(new TrustedHostMiddleware($config->getArray('waffle.trusted_hosts', []) ?? []))
    ->add(new AnonymousSessionMiddleware())                             // 3. issues WAFFLE_SID cookie + _anon_sid attr
    ->add(new CoreRoutingMiddleware($router, $responseFactory))         // 4. resolves _classname / _method; auto-answers OPTIONS
    ->add(new CsrfMiddleware($csrfTokenManager))                        // 5. validates #[RequiresCsrfToken] using _anon_sid
    ->add(new SecurityMiddleware($secureContainer, $logger))            // 6. fail-closed ABAC analysis
                                                                        //    optional 3rd arg (Beta-6): a SubjectResolverInterface that hydrates
                                                                        //    the domain subject passed to voters (IDOR/ownership rules)
    ->add(new SecureHeadersMiddleware())                                // 7. innermost — adds defensive response headers
;

$handler = $stack->createHandler($controllerDispatcher);
$response = $handler->handle($request);
```

`AnonymousSessionMiddleware` must run before `CsrfMiddleware` — the CSRF HMAC binds to the SID it publishes. `CoreRoutingMiddleware` must run before `CsrfMiddleware` and `SecurityMiddleware` — both read the `_classname`/`_method` attributes it sets. After `createHandler()`, the stack is locked. The terminal `$controllerDispatcher` is invoked only if no middleware short-circuits with its own response.
