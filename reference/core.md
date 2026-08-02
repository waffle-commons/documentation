# Core Reference (`waffle-commons/waffle`)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *Adds typed kernel lifecycle events + interface-based response conversion (ARCH-04/05)*

The framework kernel. Orchestrates the PSR-15 middleware stack, dispatches lifecycle events, and resolves controllers via the container. The kernel itself stays agnostic of routing, security, logging, and HTTP — every concrete dependency is injected.

## The Kernel

`Waffle\Kernel` extends `Waffle\Abstract\AbstractKernel`, which implements `Waffle\Commons\Contracts\Core\KernelInterface`.

```php
namespace Waffle\Abstract;

abstract class AbstractKernel implements KernelInterface, TerminableInterface
{
    protected string $environment = Constant::ENV_PROD;
    protected bool $booted = false;

    protected(set) ?System $system = null;                          // asymmetric visibility
    protected ?EventDispatcherInterface $dispatcher = null;

    public function __construct(
        public protected(set) ConfigInterface $config,
        public protected(set) ContainerInterface $container,
        protected SecurityInterface $security,
        protected(set) MiddlewareStackInterface $middlewareStack,
        protected LoggerInterface $logger = new NullLogger(),
    );
}
```

## Constructor injection (ARCH-03)

Every **required** collaborator — config, container, security, middleware stack — is a mandatory constructor parameter, so a half-built kernel is unrepresentable. The previous nullable fields + `set*()` setters + `validateState()` temporal-coupling machinery are gone. The PSR-3 logger defaults to `NullLogger`.

The **one** optional collaborator keeps a boot-time setter (marked `#[WorkerSafe(scope: 'boot-time')]`); every lifecycle hook no-ops when it is absent:

```php
public function setEventDispatcher(EventDispatcherInterface $dispatcher): void;
```

## Lifecycle

### `boot(): static`

Initializes the environment (`APP_ENV`, environment string) and flips the `$booted` guard. Idempotent — calling it twice is a no-op.

### `configure(): void`

Runs once after `boot()` (guarded by `$booted`). Scans the `waffle.paths.services` / `waffle.paths.controllers` config directories via `ContainerFactory`. Builds the `System` binding. Registers a default `ControllerDispatcher` under `RequestHandlerInterface` **only when that slot is empty** — the lookup is `has()`-gated and idempotent, so a pre-registered terminal handler is left untouched. Calls `$container->lock()` if available, then hands the locked container to `CompiledContainerLoader` — under `WAFFLE_AOT=1` with a valid artifact it swaps in the reflection-free `CompiledContainer`; on any miss the runtime container is returned unchanged (RFC-019 mandatory fallback).

### `handle(ServerRequestInterface): ResponseInterface`

The request hot-path:

1. Calls `boot()->configure()` lazily if not yet booted.
2. Dispatches `RequestReceivedEvent`. Listeners may swap the request via the returned event instance.
3. **Resolves** the terminal handler from the container under `Psr\Http\Server\RequestHandlerInterface` (type-checked) and runs the middleware stack against it — there is no hard-coded `new ControllerDispatcher(...)` on the hot path, so an app can pre-register its own terminal handler (Beta-1 Phase 1 decoupling).
4. Dispatches `ResponseGeneratedEvent`. Listeners may swap the response.
5. Returns the response.

### `terminate(ServerRequestInterface, ResponseInterface): void`

Called by `WaffleRuntime` after the response has been emitted. Dispatches `TerminateEvent` for post-response async work (audit logging, fire-and-forget tasks).

### `reset(): void`

Called between FrankenPHP worker requests. Calls `$container->reset()`, then drains the logger when it implements `ResettableInterface` (so buffered log entries never bleed across worker requests).

## Lifecycle events

All three live in `Waffle\Event\*`. As of Beta-1 (leftover-purge §2) they expose state via PHP 8.5 asymmetric visibility (`public private(set)`) — read with property access, replace with the immutable `with*()` factories. The legacy `get*()` getters have been removed.

| Event | When | Stoppable | Read | Mutate |
| :--- | :--- | :--- | :--- | :--- |
| `RequestReceivedEvent` | Before the middleware pipeline runs. | No | `$event->request` | `$event->withRequest($r)` |
| `ResponseGeneratedEvent` | After the pipeline returns. | No | `$event->response` | `$event->withResponse($r)` |
| `TerminateEvent` | After the response is emitted. | No | `$event->request`, `$event->response` | — (immutable, post-emit) |
| `ControllerArgumentsResolvedEvent` | Between argument resolution and controller invocation. | No | `$event->request`, `$event->controller`, `$event->method`, `$event->arguments` | — |

## Controller plumbing

| Class | Role |
| :--- | :--- |
| `Waffle\Handler\ControllerDispatcher` | Terminal PSR-15 handler. Resolves `_controller` + `_method` + `_route_params` from the request attributes and invokes the controller method. |
| `Waffle\Handler\ControllerArgumentResolver` | Hydrates the controller method's arguments. Detects `#[Dto]` on a parameter's type and instantiates it from the parsed body — validation happens inside the DTO's Property Hooks (RFC-011). Beta-1 hardening: each body value is **pre-validated** against the constructor parameter's declared type (`assertAssignable()` — scalars, unions, and nullability) and a mismatch (e.g. a string for `int $age`) becomes a field-level `422` carrying the offending `field` and **no** `previous` chain, so a native `\TypeError` can never reach the catch. Property Hook failures during construction are then unified: typed `ValidationExceptionInterface` bubbles verbatim (preserving `field`); a plain `\InvalidArgumentException` is rewrapped as a `422` with `previous` chained. The framework never catches `\Error` subclasses. **Beta6 audit hardening:** a route path parameter typed `int`/`float`/`bool` (e.g. `orders/{id}` → `show(int $id)`) is validated *before* casting, mirroring the DTO path — `int` via `FILTER_VALIDATE_INT` (rejects out-of-range digit strings instead of silently aliasing to `PHP_INT_MAX`), `float` via `is_numeric()` **and** `is_finite()` (rejects overflow-to-`INF` strings like `"1e400"`), `bool` via an explicit `FILTER_VALIDATE_BOOLEAN` allow-list. Any mismatch is the same field-level `422`, not a silent `(int) 'abc'` → `0`-style normalization. |
| `Waffle\Handler\ControllerResponseConverter` | Converts a controller's scalar / array return into a PSR-7 `ResponseInterface`. A bare `string` return is now **escaped by default** (`htmlspecialchars($result, ENT_QUOTES \| ENT_SUBSTITUTE, 'UTF-8')`) before being written as `text/html` — the actual injection-prevention mechanism (Beta6 audit hardening). A controller that deliberately wants unescaped HTML returns a `Waffle\Handler\RawHtml` value object instead, which is written verbatim. Every `text/html` response still carries `Content-Security-Policy: default-src 'self'; form-action 'self'; base-uri 'self'` + `X-Content-Type-Options: nosniff` as **defense-in-depth**, not the injection-prevention mechanism itself — the CSP alone was never sufficient (it blocks script execution but not `form-action`/meta-refresh-based markup injection), which is why the escaping was added. The CSP is configurable via the `$stringResponseCsp` constructor parameter. |
| `Waffle\Core\BaseController` | Default `BaseControllerInterface` implementation; provides `jsonResponse()` and similar helpers. |
| `Waffle\Abstract\AbstractController` | Abstract base that user controllers may extend. |

## Exceptions

- `RouteNotFoundException` — extends `WaffleException`, implements `RouteNotFoundExceptionInterface`. Rendered as RFC 7807 `404`.
- `ValidationException` — extends `WaffleException`, implements `ValidationExceptionInterface`. Rendered as RFC 7807 `422`, with the optional `getField()` surfaced into the payload.
- `RenderingException` — extends `WaffleException`; generic rendering failures.
- `InvalidConfigurationException` — extends `\Exception` directly, implements `InvalidConfigurationExceptionInterface`; raised when a configuration value is missing or has an invalid type.

The `ErrorHandlerMiddleware` translates each via interface-matching, so application exceptions opt into the right HTTP status by implementing the corresponding contract interface.
