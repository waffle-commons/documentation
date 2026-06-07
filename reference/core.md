# Core Reference (`waffle-commons/waffle`)

> **Release:** `0.1.0-beta3` &nbsp;|&nbsp; *No behavioural changes since Beta-1*

The framework kernel. Orchestrates the PSR-15 middleware stack, dispatches lifecycle events, and resolves controllers via the container. The kernel itself stays agnostic of routing, security, logging, and HTTP — every concrete dependency is injected.

## The Kernel

`Waffle\Kernel` extends `Waffle\Abstract\AbstractKernel`, which implements `Waffle\Commons\Contracts\Core\KernelInterface`.

```php
namespace Waffle\Abstract;

abstract class AbstractKernel implements KernelInterface
{
    protected string $environment = Constant::ENV_PROD;
    protected bool $booted = false;

    public ?ConfigInterface $config = null;
    public ?ContainerInterface $container = null;
    protected ?SecurityInterface $security = null;
    protected ?EventDispatcherInterface $dispatcher = null;

    protected(set) ?System $system = null;                          // asymmetric visibility
    protected(set) ?MiddlewareStackInterface $middlewareStack = null;

    public function __construct(protected LoggerInterface $logger = new NullLogger());
}
```

## Setter API

The kernel uses **setter injection** for its dependencies. Verbatim from `AbstractKernel`:

```php
public function setContainerImplementation(PsrContainerInterface $container): void;
public function setConfiguration(ConfigInterface $config): void;
public function setSecurity(SecurityInterface $security): void;
public function setMiddlewareStack(MiddlewareStackInterface $stack): void;
public function setEventDispatcher(EventDispatcherInterface $dispatcher): void;
```

The PSR-3 logger is passed via the constructor and stored as `protected LoggerInterface $logger`; default is `NullLogger`.

## Lifecycle

### `boot(): static`

Initializes the environment (`APP_ENV`, environment string) and flips the `$booted` guard. Idempotent — calling it twice is a no-op.

### `configure(): void`

Runs once after `boot()`. Validates that `ConfigInterface`, `SecurityInterface`, and a PSR-11 container were injected. Builds the `System` binding. Registers a default `ControllerDispatcher` under `RequestHandlerInterface` **only when that slot is empty** — the lookup is `has()`-gated and idempotent, so a pre-registered terminal handler is left untouched. Optionally calls `$container->lock()` if available.

### `handle(ServerRequestInterface): ResponseInterface`

The request hot-path:

1. Calls `boot()->configure()` lazily if not yet booted.
2. `validateState()` — raises if the middleware stack / container / system isn't set up.
3. Dispatches `RequestReceivedEvent`. Listeners may swap the request via the returned event instance.
4. **Resolves** the terminal handler from the container under `Psr\Http\Server\RequestHandlerInterface` (type-checked) and runs the middleware stack against it — there is no hard-coded `new ControllerDispatcher(...)` on the hot path, so an app can pre-register its own terminal handler (Beta-1 Phase 1 decoupling).
5. Dispatches `ResponseGeneratedEvent`. Listeners may swap the response.
6. Returns the response.

### `terminate(ServerRequestInterface, ResponseInterface): void`

Called by `WaffleRuntime` after the response has been emitted. Dispatches `TerminateEvent` for post-response async work (audit logging, fire-and-forget tasks).

### `reset(): void`

Called between FrankenPHP worker requests. Currently calls `$container->reset()`.

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
| `Waffle\Handler\ControllerArgumentResolver` | Hydrates the controller method's arguments. Detects `#[Dto]` on a parameter's type and instantiates it from the parsed body — validation happens inside the DTO's Property Hooks (RFC-011). Beta-1 hardening: each body value is **pre-validated** against the constructor parameter's declared type (`assertAssignable()` — scalars, unions, and nullability) and a mismatch (e.g. a string for `int $age`) becomes a field-level `422` carrying the offending `field` and **no** `previous` chain, so a native `\TypeError` can never reach the catch. Property Hook failures during construction are then unified: typed `ValidationExceptionInterface` bubbles verbatim (preserving `field`); a plain `\InvalidArgumentException` is rewrapped as a `422` with `previous` chained. The framework never catches `\Error` subclasses. |
| `Waffle\Handler\ControllerResponseConverter` | Converts a controller's scalar / array return into a PSR-7 `ResponseInterface`. String returns (auto-`text/html`) carry `Content-Security-Policy: default-src 'self'` + `X-Content-Type-Options: nosniff` headers as an XSS safety floor (Beta-1 Phase 3 Task 3.3). The CSP is configurable via the `$stringResponseCsp` constructor parameter. |
| `Waffle\Core\BaseController` | Default `BaseControllerInterface` implementation; provides `jsonResponse()` and similar helpers. |
| `Waffle\Abstract\AbstractController` | Abstract base that user controllers may extend. |

## Exceptions

All inherit from `Waffle\Exception\WaffleException`:

- `RouteNotFoundException` — implements `RouteNotFoundExceptionInterface`. Rendered as RFC 7807 `404`.
- `ValidationException` — implements `ValidationExceptionInterface`. Rendered as RFC 7807 `422`, with the optional `getField()` surfaced into the payload.
- `RenderingException` — generic rendering failures.
- `InvalidConfigurationException` — kernel raised this when required setters are missing.

The `ErrorHandlerMiddleware` translates each via interface-matching, so application exceptions opt into the right HTTP status by implementing the corresponding contract interface.
