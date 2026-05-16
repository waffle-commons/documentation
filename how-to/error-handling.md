# How-To: Error Handling

> **Beta 0** — Waffle ships a PSR-15 `ErrorHandlerMiddleware` paired with an RFC 7807 ("Problem Details for HTTP APIs") JSON renderer.

## 1. Automatic handling

`Waffle\Commons\ErrorHandler\Middleware\ErrorHandlerMiddleware` is the **outermost** middleware in the kernel's stack — it wraps `$handler->handle()` in `try { … } catch (Throwable)` and delegates to `Waffle\Commons\ErrorHandler\Renderer\JsonErrorRenderer` for response construction.

```php
$stack
    ->add(new ErrorHandlerMiddleware($renderer, $logger))   // outermost — catches everything
    ->add(/* … */);
```

Every `Throwable` raised deeper in the stack is logged via the injected `Psr\Log\LoggerInterface` and rendered as a JSON response.

## 2. Throw an exception, get an RFC 7807 response

```php
public function show(ServerRequestInterface $request): ResponseInterface
{
    throw new \RuntimeException('Something went wrong.');
}
```

Production response (`debug = false`):

```http
HTTP/1.1 500 Internal Server Error
Content-Type: application/problem+json

{
  "type":     "about:blank",
  "title":    "Internal Server Error",
  "status":   500,
  "detail":   "An internal server error occurred.",
  "instance": "/show"
}
```

Debug response (`debug = true`):

```json
{
  "type":     "about:blank",
  "title":    "Internal Server Error",
  "status":   500,
  "detail":   "Something went wrong.",
  "instance": "/show",
  "trace":    ["#0 …", "#1 …"],
  "file":     "/app/src/Controller/UserController.php",
  "line":     42
}
```

In production mode any 5xx `detail` is masked to `"An internal server error occurred."`; in debug mode the original message plus `trace` / `file` / `line` are exposed.

## 3. Use contract interfaces to pick the HTTP status

`JsonErrorRenderer::determineStatusCode()` walks **interfaces**, not exception classes — your application exceptions opt into a specific status by implementing the right contract interface. The built-in mappings:

| Interface | HTTP status |
| :--- | :--- |
| `Waffle\Commons\Contracts\Routing\Exception\RouteNotFoundExceptionInterface` | `404` |
| `Waffle\Commons\Contracts\Exception\Validation\ValidationExceptionInterface` | `422` |
| `\InvalidArgumentException` (or implementors of `Waffle\Commons\Contracts\Console\Exception\InvalidArgumentExceptionInterface`) | `400` |
| Anything else | `500` |

```php
use Waffle\Commons\Contracts\Exception\Validation\ValidationExceptionInterface;

final class InvalidEmailException
    extends \DomainException
    implements ValidationExceptionInterface
{
    public function __construct(private readonly string $field, string $message)
    {
        parent::__construct($message);
    }

    public function getField(): ?string { return $this->field; }
}

// …

throw new InvalidEmailException(field: 'email', message: 'Invalid email format.');
```

The renderer produces:

```json
{
  "type":     "about:blank",
  "title":    "Unprocessable Entity",
  "status":   422,
  "detail":   "Invalid email format.",
  "instance": "/register",
  "field":    "email"
}
```

The `field` key is added automatically when the exception implements `ValidationExceptionInterface` and `getField()` returns non-null (RFC-011).

## 4. Customizing the renderer

`JsonErrorRenderer`'s constructor takes a PSR-17 `ResponseFactoryInterface` and a `bool $debug` flag:

```php
$renderer = new JsonErrorRenderer(
    responseFactory: new \Waffle\Commons\Http\Factory\ResponseFactory(),
    debug: $config->getBool('app.debug', default: false) ?? false,
);
```

`debug` should be **false** in production. Application errors then surface as their masked `500` form; only the log message receives the full detail.
