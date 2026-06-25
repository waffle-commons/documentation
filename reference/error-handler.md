# Error Handler Reference (`waffle-commons/error-handler`)

> **Release:** `0.1.0-beta5`
> **PSR Compliance:** PSR-15 (`MiddlewareInterface`), produces RFC 7807 ("Problem Details for HTTP APIs") responses.

RFC 7807 JSON error rendering plus the PSR-15 middleware that ties exceptions to it. The outermost layer of the canonical pipeline — it catches every `Throwable` thrown downstream and converts it into a structured, security-aware response.

## Surface

| Class | Role |
| :--- | :--- |
| `Waffle\Commons\ErrorHandler\Middleware\ErrorHandlerMiddleware` | PSR-15 middleware. Wraps `$handler->handle()` in `try { … } catch (Throwable)` and delegates to the renderer. |
| `Waffle\Commons\ErrorHandler\Renderer\JsonErrorRenderer` | `final readonly` PSR-7 RFC 7807 renderer. Maps exceptions to HTTP status codes; redacts internal details in production. |

## `JsonErrorRenderer` — constructor

```php
new JsonErrorRenderer(
    responseFactory: $psr17ResponseFactory,
    debug:           false,  // true → include trace/file/line; false → masked 5xx
);
```

`$debug = true` adds the stack trace, source file, and line number to the response body. **Never enable in production** — it leaks paths and code structure.

## Status-code resolution

`JsonErrorRenderer::determineStatusCode()` resolves in this order:

1. `ValidationExceptionInterface` → **422** (precedence override — the field-level error always wins regardless of the exception's `getCode()`).
2. Exception `code` in range `[400, 599]` → use it verbatim. (`SecurityException(403)`, `InvalidCsrfTokenException(403)`, `MethodNotAllowedException(405)`, …)
3. `RouteNotFoundExceptionInterface` → **404**.
4. `MethodNotAllowedExceptionInterface` → **405**.
5. `InvalidArgumentException` → **400**.
6. Anything else → **500**.

## Response body shape (RFC 7807)

```json
{
  "type":     "about:blank",
  "title":    "Forbidden",
  "status":   403,
  "detail":   "Security Policy Violation: AdminController::deleteAction declares no #[Voter]...",
  "instance": "/admin/users/42",
  "field":    "<optional, ValidationException only>"
}
```

- `Content-Type: application/problem+json` per RFC 7807.
- For a `MethodNotAllowedExceptionInterface`, an RFC 7231 `Allow` header is added listing the allowed methods (e.g. `Allow: GET, HEAD, OPTIONS, POST`); it is omitted only when the list is empty.
- `field` is added only when the exception implements `ValidationExceptionInterface` and returns a non-null `getField()`.
- In production (`$debug === false`), responses with status ≥ 500 have their `detail` replaced by `"An internal server error occurred."` to avoid leaking internals.
- In debug, the body additionally carries `trace` (array of frames), `file`, `line`.

## `ErrorHandlerMiddleware`

```php
namespace Waffle\Commons\ErrorHandler\Middleware;

final readonly class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private ErrorRendererInterface $renderer,
        private LoggerInterface        $logger,
    );
}
```

Behaviour:

1. Calls `$handler->handle($request)`.
2. On `Throwable`, logs (at level matching the resolved status — 5xx as `error`, 4xx as `warning`) and returns `$renderer->render($e, $request)`.
3. Returns the downstream response untouched on success.

The middleware is **prepended** to the stack so it wraps everything. See the [pipeline reference](pipeline.md) for the canonical Beta-1 ordering.

## Status titles

`getTitleForStatus()` follows RFC 9110:

| Status | Title |
| :--- | :--- |
| `400` | Bad Request |
| `401` | Unauthorized |
| `403` | Forbidden |
| `404` | Not Found |
| `405` | Method Not Allowed |
| `409` | Conflict |
| `415` | Unsupported Media Type |
| `422` | Unprocessable Entity |
| `429` | Too Many Requests |
| `500` | Internal Server Error |
| `502` | Bad Gateway |
| `503` | Service Unavailable |

Any unmapped status falls through to a generic `'Error'`.

## Worker-mode safety

The middleware and renderer are both `final readonly` with no instance state across requests. Logger and response factory dependencies are injected; both are themselves stateless contracts under Waffle's PSR-3 / PSR-17 implementations.

## Related

- [pipeline.md](pipeline.md) — `ErrorHandlerMiddleware` placement in the canonical stack.
- [contracts.md](contracts.md) — the exception interfaces resolved against (`ValidationExceptionInterface`, `RouteNotFoundExceptionInterface`).
- [how-to/error-handling.md](../how-to/error-handling.md) — task-oriented guide.
