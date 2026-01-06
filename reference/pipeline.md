# Pipeline Reference (`waffle-commons/pipeline`)

The Pipeline component manages the HTTP middleware stack. It allows you to intercept and process requests and responses before they reach the controller.

## MiddlewareStack

The `Waffle\Commons\Pipeline\MiddlewareStack` class implements `MiddlewareStackInterface`. It manages a list of PSR-15 `MiddlewareInterface` instances.

### Usage

```php
use Waffle\Commons\Pipeline\MiddlewareStack;

$stack = new MiddlewareStack();
```

### methods

#### `add(MiddlewareInterface $middleware): self`

Appends a middleware to the **end** of the stack. It will be executed *after* previously added middleware.

```php
$stack->add($myMiddleware);
```

#### `prepend(MiddlewareInterface $middleware): self`

Prepends a middleware to the **beginning** of the stack. It will be executed *before* existing middleware. This is useful for error handlers or critical security checks that must run first.

```php
// ErrorHandler should usually be first
$stack->prepend($errorHandler);
```

#### `createHandler(RequestHandlerInterface $fallbackHandler): RequestHandlerInterface`

Compiles the middleware stack into a single `RequestHandlerInterface`. If the stack is exhausted (no middleware generates a response), the `$fallbackHandler` (usually the Controller Dispatcher) is executed.

```php
$handler = $stack->createHandler($dispatcher);
$response = $handler->handle($request);
```
