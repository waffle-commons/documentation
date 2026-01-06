# How to Add Middleware

Middleware intercepts requests and responses. Follow these steps to create and register one.

## 1. Create the Middleware Class

Implement `Psr\Http\Server\MiddlewareInterface`.

```php
<?php

declare(strict_types=1);

namespace App\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class LoggerMiddleware implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // Pre-processing
        error_log('Request: ' . $request->getUri()->getPath());

        // Call next handler
        $response = $handler->handle($request);

        // Post-processing
        return $response;
    }
}
```

## 2. Register it

You can register middleware in your bootstrapping logic (e.g., `public/index.php`).

```php
// ... $kernel initialization

$kernel->getMiddlewareStack()->add(new \App\Middleware\LoggerMiddleware());
```

- `add()`: Runs **last** (closest to Controller).
- `prepend()`: Runs **first** (closest to Request entry).
