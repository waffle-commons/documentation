# How-To: Error Handling

Waffle provides a robust error handling system out of the box using the `ErrorHandlerMiddleware`.

## Automatic Handling

The `Waffle\Commons\ErrorHandler\Middleware\ErrorHandlerMiddleware` is automatically prepended to the middleware stack in `AppKernelFactory`. It catches **all** exceptions thrown during the request lifecycle and converts them into a formatted HTTP response using the `JsonErrorRenderer`.

## usage

To trigger an error response, simply throw an exception from your Controller or Middleware.

### Standard Exceptions

Throwing a generic `\RuntimeException` or `\Exception` will result in a **500 Internal Server Error**.

```php
public function index(): ResponseInterface
{
    throw new \RuntimeException("Something went wrong!");
}
```

**Response:**
```json
{
    "error": "Internal Server Error",
    "message": "Something went wrong!",
    "code": 500
}
```

### Waffle Exceptions

You can throw specific Waffle exceptions to trigger different HTTP status codes.

#### `Waffle\Exception\WaffleException`

This is the base class for framework exceptions.

#### `Waffle\Commons\Security\Exception\SecurityException`

Throwing this exception will result in a **500 Error** (or potentially 403/401 depending on future renderer logic, currently defaults to 500 in the codebase).

```php
use Waffle\Commons\Security\Exception\SecurityException;

public function secret(): ResponseInterface
{
    throw new SecurityException("Access Denied");
}
```

## Customizing the Renderer

The default `JsonErrorRenderer` respects the `app.debug` configuration.
- **Debug=true**: Shows full stack traces.
- **Debug=false**: Shows generic error messages for security.
