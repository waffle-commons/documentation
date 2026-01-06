# HTTP Reference (`waffle-commons/http`)

Waffle fully embraces PSR-7 (HTTP Message) and PSR-17 (HTTP Factories) standards. It does not reinvent the wheel but provides robust implementations and helpers where necessary.

## Request & Response

The framework utilizes standard interfaces:

-   **Request**: `Psr\Http\Message\ServerRequestInterface`
-   **Response**: `Psr\Http\Message\ResponseInterface`

## Creating Responses

In your Controllers, it is recommended to use the `ResponseFactoryInterface` via Dependency Injection or helper methods provided by `AbstractController`.

### Using `AbstractController` Helper

The `Waffle\Abstract\AbstractController` provides a `jsonResponse` method for convenience.

```php
// In a Controller extending BaseController
return $this->jsonResponse(data: ['status' => 'ok']);
```

### Manual Creation

You can also use a PSR-17 Factory directly if you need more control (e.g., for non-JSON responses).

```php
use Psr\Http\Message\ResponseFactoryInterface;

public function index(ResponseFactoryInterface $factory): ResponseInterface
{
    $response = $factory->createResponse(200);
    $response->getBody()->write('Hello World');
    
    return $response;
}
```

## Runtime & Emitter

The `WaffleRuntime` and `AppKernelFactory` orchestrate the HTTP lifecycle.

-   **GlobalsFactory**: Creates a PSR-7 Request from PHP superglobals (`$_SERVER`, etc.).
-   **ResponseEmitter**: Emits the PSR-7 Response to the client (sending headers and outputting the body).
-   **WaffleRuntime**: Ties everything together (`$runtime->run(...)`).
