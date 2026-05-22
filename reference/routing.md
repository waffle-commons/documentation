# Routing Reference (`waffle-commons/routing`)

> **Release:** `v0.1.0-beta1`

Attribute-driven router. Routes live next to the controller code via the `#[Route]` attribute; the `RouteDiscoverer` scans the configured controller directory at boot time and the compiled table is cached.

## `#[Route]` attribute — exact signature

From `src/Attribute/Route.php`:

```php
namespace Waffle\Commons\Routing\Attribute;

use Attribute;

#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
final class Route
{
    /**
     * @param array<Argument>|null $arguments
     */
    public function __construct(
        public string $path,
        public ?string $name = null,
        public ?array $arguments = null,
    ) {}
}
```

| Parameter | Type | Default | Purpose |
| :--- | :--- | :--- | :--- |
| `path` | `string` | (required) | URL pattern. Use `{name}` for placeholders — values land in `$request->getAttribute('_route_params')`. |
| `name` | `?string` | `null` | Unique route name. Used for debugging and the `route:list` console command. |
| `arguments` | `?array<Argument>` | `null` | Container-resolved argument injection (see below). |

Targets both `TARGET_CLASS` and `TARGET_METHOD`. Class-level `#[Route]` defines a path prefix for every method-level `#[Route]` on that class.

## `#[Argument]` attribute — exact signature

From `src/Attribute/Argument.php`:

```php
namespace Waffle\Commons\Routing\Attribute;

use Attribute;

#[Attribute]
final class Argument
{
    public function __construct(
        public string $classType,
        public string $paramName,
        public bool $required = true,
    ) {}
}
```

Tells the router that the controller's `$paramName` argument should be resolved from the container by FQCN `$classType`. Use this when you want a controller method to receive a service that isn't naturally derivable from the route path:

```php
use Waffle\Commons\Routing\Attribute\Route;
use Waffle\Commons\Routing\Attribute\Argument;

#[Route(
    path: '/users',
    name: 'users.create',
    arguments: [
        new Argument(classType: UserRepository::class, paramName: 'repo'),
    ],
)]
public function create(ServerRequestInterface $request, UserRepository $repo): ResponseInterface
{
    // $repo is fetched from the container at dispatch time
}
```

## `RouterInterface`

From `Waffle\Commons\Contracts\Routing\RouterInterface`:

```php
public function boot(ContainerInterface $container): static;

/**
 * @return array{
 *   classname: class-string,
 *   method:    string,
 *   arguments: array<string, mixed>,
 *   path:      string,
 *   name:      non-falsy-string,
 *   params?:   array<string, mixed>,
 * }|null
 */
public function matchRequest(ServerRequestInterface $request): ?array;

/**
 * @return array<array-key, array{
 *   classname: class-string,
 *   method:    string,
 *   arguments: array<string, mixed>,
 *   path:      string,
 *   name:      non-falsy-string,
 * }>
 */
public function getRoutes(): array;
```

`matchRequest()` returns `null` when no route matches — `CoreRoutingMiddleware` converts that to the concrete `Waffle\Commons\Contracts\Routing\Exception\RouteNotFoundException` (rendered as RFC 7807 `404` by the error handler).

## Components

| Class | Responsibility |
| :--- | :--- |
| `Waffle\Commons\Routing\Router` | The router entry point. Boots from the container, compiles the route table. |
| `Waffle\Commons\Routing\RouteDiscoverer` | Walks the controller directory (`waffle.paths.controllers` in `app.yaml`), opens each PHP file with `ReflectionTrait`, reads `#[Route]` attributes. |
| `Waffle\Commons\Routing\ControllerFinder` | Filesystem traversal of `*.php` files. |
| `Waffle\Commons\Routing\RouteParser` | Converts a `Route` attribute + reflection metadata into the canonical route-match array. |
| `Waffle\Commons\Routing\Trait\RequestTrait` | Helpers for extracting routing metadata (`_controller`, `_route_params`) from a PSR-7 request. |

## Configuration

```yaml
# config/app.yaml
waffle:
  paths:
    controllers: src/Controller
```

`RouteDiscoverer` is rooted at `APP_ROOT . '/' . waffle.paths.controllers` and recursively scans for `*.php` files containing controllers with `#[Route]` attributes.

## Request attributes after routing

When `CoreRoutingMiddleware` runs successfully, the following attributes are present on the request when the controller is invoked:

- `_controller` — the FQCN of the resolved controller class.
- `_method` — the controller method name to invoke.
- `_route_params` — `array<string, mixed>` of `{placeholder}` values extracted from the path.

The `Waffle\Commons\Routing\Trait\RequestTrait` provides typed accessors for these.
