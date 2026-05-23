# Pipeline Reference (`waffle-commons/pipeline`)

> **Release:** `v0.1.0-beta1`
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

Routes the request by calling `RouterInterface::matchRequest()` (which returns the immutable `MatchedRoute` DTO since Beta-1 — see [`reference/routing`](routing.md#matchedroute-dto)), then exposes the result on the request attributes:

- `_classname` — FQCN of the resolved controller class.
- `_method` — method name to invoke.
- `_arguments` — `array<string, mixed>` of per-argument metadata from `#[Argument]`.
- `_path` — original route path pattern.
- `_name` — route name from `#[Route(name: ...)]`.
- `_params` — `array<string, mixed>` of `{placeholder}` values extracted from the URI.

When no route matches, raises `Waffle\Commons\Contracts\Routing\Exception\RouteNotFoundException` (the concrete `final` class added in Beta-1, which implements `RouteNotFoundExceptionInterface` and is rendered as RFC 7807 `404` by `JsonErrorRenderer`). Beta-0 raised a generic `RuntimeException` here and produced a `500`; that mapping has been fixed.

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

Adds the baseline security response headers (`X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, etc.) to every outgoing response.

## Conventional ordering (Beta-1)

```php
$stack = (new MiddlewareStack())
    ->add(new ErrorHandlerMiddleware($renderer, $logger))               // 1. outermost (catches everything)
    ->add(new TrustedHostMiddleware($config->getArray('waffle.trusted_hosts', []) ?? []))
    ->add(new AnonymousSessionMiddleware())                             // 3. issues WAFFLE_SID cookie + _anon_sid attr
    ->add(new CoreRoutingMiddleware($router))                           // 4. resolves _classname / _method
    ->add(new CsrfMiddleware($csrfTokenManager))                        // 5. validates #[RequiresCsrfToken] using _anon_sid
    ->add(new SecurityMiddleware($secureContainer, $logger))            // 6. fail-closed ABAC analysis
    ->add(new SecureHeadersMiddleware())                                // 7. innermost — adds defensive response headers
;

$handler = $stack->createHandler($controllerDispatcher);
$response = $handler->handle($request);
```

`AnonymousSessionMiddleware` must run before `CsrfMiddleware` — the CSRF HMAC binds to the SID it publishes. `CoreRoutingMiddleware` must run before `CsrfMiddleware` and `SecurityMiddleware` — both read the `_classname`/`_method` attributes it sets. After `createHandler()`, the stack is locked. The terminal `$controllerDispatcher` is invoked only if no middleware short-circuits with its own response.
