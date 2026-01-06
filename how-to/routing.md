# How to Configure Routing

Waffle uses PHP Attributes to define routes directly next to your code.

## Defining a Route

Add the `#[Route]` attribute to any public method in your Controller.

```php
use Waffle\Commons\Routing\Attribute\Route;

#[Route(path: '/user', name: 'user_list')]
public function list(): ResponseInterface
{
    // ...
}
```

## Using Parameters

Dynamic parts of the URL are defined with curly braces `{}`.

### Basic Parameter
```php
#[Route(path: '/user/{id}', name: 'user_show')]
public function show(string $id): ResponseInterface
{
    // $id matches the URL segment
}
```

### Multiple Parameters
The order of arguments in the method does not matter; names must match.

```php
#[Route(path: '/blog/{category}/{slug}', name: 'blog_post')]
public function post(string $slug, string $category): ResponseInterface
{
    // ...
}
```

## Restricting HTTP Methods (Planned)

*Note: In v0.1.0-alpha4, the `#[Route]` attribute does not yet strictly enforce HTTP methods at the routing level. You should check the request method inside your controller if necessary.*

```php
public function update(ServerRequestInterface $request): ResponseInterface
{
    if ($request->getMethod() !== 'POST') {
        // Return 405 Method Not Allowed
    }
}
```
