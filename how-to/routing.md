# How-To: Routing

Waffle uses PHP 8 Attributes to define routes directly in your controller classes. This keeps your routing logic close to your application code.

## Defining a Route

Use the `#[Route]` attribute from `Waffle\Commons\Routing\Attribute` to map a URL path to a controller method.

```php
namespace App\Controller;

use Waffle\Commons\Routing\Attribute\Route;
use Waffle\Core\BaseController;

final class BlogController extends BaseController
{
    #[Route(path: '/blog', name: 'blog_list')]
    public function list(): ResponseInterface
    {
        // ...
    }
}
```

## Route Parameters

You can define dynamic route parameters using curly braces `{param}`. These parameters are passed to your controller method arguments.

### Basic Usage

```php
#[Route(path: '/blog/{id}', name: 'blog_show')]
public function show(string $id): ResponseInterface
{
    // $id contains the value from the URL
}
```

### Strict Typing & Validation

To enforce strict typing and validation, you can use the `Argument` attribute within the `Route` definition. This ensures the router validates the parameter before invoking your controller.

```php
use Waffle\Commons\Routing\Attribute\Argument;

#[Route(
    path: '/blog/{id}', 
    name: 'blog_show',
    arguments: [
        new Argument(classType: 'int', paramName: 'id', required: true)
    ]
)]
public function show(int $id): ResponseInterface
{
    // $id is guaranteed to be an integer
    return $this->jsonResponse(data: ['id' => $id]);
}
```

## Route Prefixes

You can also apply the `#[Route]` attribute to the **Class** itself to define a prefix for all methods within that controller.

```php
#[Route(path: '/api/v1', name: 'api_')]
final class ApiController extends BaseController
{
    // Matches /api/v1/users
    // Route name: api_users
    #[Route(path: '/users', name: 'users')]
    public function users(): ResponseInterface
    {
        // ...
    }
}
```
