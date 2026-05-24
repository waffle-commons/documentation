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

## Catch-all & priority routing (Beta-1 Phase 3)

The `#[Route]` attribute takes an optional `int $priority = 0`. The router sorts the compiled table by descending priority at boot time and **caches the sorted collection**, so a hot boot pays no sort cost. Higher numbers match first; negative numbers run last.

This unlocks the EcoShield-Gateway pattern: a catch-all controller that forwards any URI no other controller claimed to the legacy monolith — without throwing `404`.

```php
use Waffle\Commons\Routing\Attribute\Route;

#[Route(path: '/', name: 'gateway', priority: -1000)]
final class GatewayController
{
    /**
     * Matches whatever path slipped past every priority-0 route. `{forwarded:.*}`
     * spans every remaining segment, and the negative class-level priority is
     * inherited by every method that doesn't declare its own — so `fallback()`
     * runs last automatically.
     */
    #[Route(path: '{forwarded:.*}', name: 'fallback')]
    public function fallback(string $forwarded): ResponseInterface
    {
        // forward to the legacy backend …
    }
}
```

**Resolution rules**

- **Method wins on conflict.** If a method declares a non-zero `priority`, it overrides the class-level value. Otherwise the class value is inherited (mirroring how `path` already composes).
- **Default is `0`.** Unannotated routes keep their declaration order within the `priority = 0` bucket (PHP 8.5's `usort` is stable).
- **Three placeholder forms.** `{name}` matches a single segment, `{name:regex}` constrains it (e.g. `{id:\d+}`), and `{name:.*}` spans multiple segments — the form a real gateway catch-all uses. Static text between placeholders is matched literally.
