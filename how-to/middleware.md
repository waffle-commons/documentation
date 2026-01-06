# How-To: Middleware

Middleware allows you to inspect and filter HTTP requests entering your application and responses leaving it. Waffle uses PSR-15 standard middleware.

## Creating a Middleware

To create a middleware, implement the `Psr\Http\Server\MiddlewareInterface`.

### Example: Logging Middleware

Create `src/Middleware/LogMiddleware.php`:

```php
<?php

declare(strict_types=1);

namespace App\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Psr\Log\LoggerInterface;

class LogMiddleware implements MiddlewareInterface
{
    public function __construct(
        private LoggerInterface $logger
    ) {}

    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // 1. Logic before the controller
        $this->logger->info('Request received: ' . $request->getUri()->getPath());

        // 2. Delegate to the next middleware/controller
        $response = $handler->handle($request);

        // 3. Logic after the controller
        $this->logger->info('Response generated: ' . $response->getStatusCode());

        return $response;
    }
}
```

## Registering Middleware

Middleware is registered in your `App\Factory\AppKernelFactory`. This is where the application pipeline is assembled.

Open `src/Factory/AppKernelFactory.php` and locate the `create` method:

```php
// src/Factory/AppKernelFactory.php

use App\Middleware\LogMiddleware;
use Psr\Log\NullLogger; // Or your concrete Logger

// ... inside create() method ...

// 5. Instantiate the Pipeline Middleware
$stack = new MiddlewareStack();

// ... existing error handler ...

// Register your custom middleware
$stack->add(new LogMiddleware(new NullLogger()));
```

> [!TIP]
> Use `$stack->prepend($middleware)` if you need your middleware to run *before* others (e.g., for early security checks or error handling).
