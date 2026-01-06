# How to Handle Errors

Waffle automates error rendering using the RFC 7807 standard. However, you can control what users see by throwing specific exceptions.

## Throwing Exceptions

To trigger an error response, simply throw an exception from your Controller or Service.

```php
use Waffle\Commons\Routing\Exception\RouteNotFoundException;

public function show(string $id): ResponseInterface
{
    $user = $this->repository->find($id);

    if (!$user) {
        // Returns a 404 response
        throw new RouteNotFoundException("User $id not found."); 
    }
    
    // ...
}
```

## Customizing the Status Code (Planned)

Currently, the `JsonErrorRenderer` maps exceptions to status codes internally:
- `RouteNotFoundExceptionInterface` -> **404**
- `InvalidArgumentException` -> **400**
- All others -> **500**

*Support for mapping custom exception classes to specific status codes (e.g., `MyCustomException` -> 402) is planned for the next release.*
