# How-To: Routing

Waffle uses PHP 8 Attributes to define routes directly in your controller classes. This keeps your routing logic close to your application code.

## Defining a Route

Use the `#[Route]` attribute from `Waffle\Commons\Contracts\Routing\Attribute` to map a URL path to a controller method. (The attribute was relocated to `contracts` in Beta-2; the old `Waffle\Commons\Routing\Attribute\Route` no longer exists.)

```php
namespace App\Controller;

use Waffle\Commons\Contracts\Routing\Attribute\Route;
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

## HTTP Method Filtering & Overloading

Waffle supports defining allowed HTTP methods for each route, enabling **route overloading** where multiple controller methods can handle the same URL path, so long as they specify different HTTP methods.

### Specifying HTTP Methods

By default, routes handle `GET` requests if no methods are specified. You can specify a list of allowed HTTP methods via the `methods` property:

```php
use Waffle\Commons\Contracts\Routing\Attribute\Route;
use Waffle\Commons\Contracts\Routing\Constant as Routing;

// Explicitly allow both GET and POST
#[Route(path: '/items', methods: ['GET', 'POST'], name: 'items')]
public function handleItems(): ResponseInterface
{
    // ...
}
```

Using Waffle's routing constants is highly recommended to prevent typos:

```php
#[Route(path: '/blog', methods: [Routing::METHOD_POST], name: 'blog_create')]
public function create(): ResponseInterface
{
    // ...
}
```

### Route Overloading Example

```php
use Waffle\Commons\Contracts\Routing\Attribute\Route;
use Waffle\Commons\Contracts\Routing\Constant as Routing;

final class ArticleController extends BaseController
{
    // Matches GET /articles (Fetch list)
    #[Route(path: '/articles', methods: [Routing::METHOD_GET], name: 'articles_list')]
    public function list(): ResponseInterface
    {
        // ...
    }

    // Matches POST /articles (Create new article)
    #[Route(path: '/articles', methods: [Routing::METHOD_POST], name: 'articles_create')]
    public function create(): ResponseInterface
    {
        // ...
    }
}
```

### Method matching, HEAD, OPTIONS & RFC 7231 / RFC 7807 Rejections
When a request is made:
1. **Match:** Waffle looks for a route matching both the path and the requested HTTP method (case-insensitively). A `HEAD` request matches a `GET` route (RFC 7231 §4.3.2). If found, it dispatches immediately.
2. **`OPTIONS` preflight:** an `OPTIONS` request to a known path with no explicit `OPTIONS` handler is auto-answered with `204 No Content` + the `Allow` header (when `CoreRoutingMiddleware` is wired with a PSR-17 response factory — the default).
3. **`405` rejection:** if the path matches but no route accepts the (non-OPTIONS) method, the system automatically:
   - Merges the allowed methods across every path-matching route, then augments them — `HEAD` is added when `GET` is allowed, `OPTIONS` is always added — de-duplicates, and **sorts alphabetically**.
   - Throws a `MethodNotAllowedException` (HTTP `405`).
   - Injects the RFC 7231-compliant `Allow` header (e.g. `Allow: GET, HEAD, OPTIONS, POST`) into the response.
   - Renders a clean, RFC 7807-compliant `405 Method Not Allowed` JSON response.


## Catch-all & priority routing (Beta-1 Phase 3)

The `#[Route]` attribute takes an optional `int $priority = 0`. The router sorts the compiled table by descending priority at boot time and **caches the sorted collection**, so a hot boot pays no sort cost. Higher numbers match first; negative numbers run last.

This unlocks the EcoShield-Gateway pattern: a catch-all controller that forwards any URI no other controller claimed to the legacy monolith — without throwing `404`.

```php
use Waffle\Commons\Contracts\Routing\Attribute\Route;

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
