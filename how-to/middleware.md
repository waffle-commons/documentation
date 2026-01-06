# How to Create Custom Middleware

Middleware allows you to interpret or modify requests before they reach your controller, and responses before they are sent to the client.

## 1. Implement `MiddlewareInterface`

Create a new class implementing the standard PSR-15 interface.

```php
namespace App\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class LoggerMiddleware implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // Logic BEFORE Controller
        $path = $request->getUri()->getPath();
        error_log("Incoming Request: $path");

        // Delegate to the next middleware/controller
        $response = $handler->handle($request);

        // Logic AFTER Controller
        error_log("Response Status: " . $response->getStatusCode());

        return $response;
    }
}
```

## 2. Register Global Middleware

To make your middleware run on every request, you need to add it to the stack. Open `public/index.php` (or your bootstrapping logic).

```php
use App\Middleware\LoggerMiddleware;

// ... Kernel Initialization

/** @var \Waffle\Kernel $kernel */
$kernel->getMiddlewareStack()->add(new LoggerMiddleware());
```

Using `add()` puts it at the **end** of the stack (executed last). Use `prepend()` to run it **first**.
