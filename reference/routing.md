# Routing Reference

The Routing component (`waffle-commons/routing`) maps HTTP requests to Controllers using PHP Attributes.

## `#[Route]` Attribute

Use this attribute on controller methods.

```php
#[Route(path: '/path/{param}', name: 'route_name')]
```
- **path**: The URL pattern. Supports dynamic parameters `{name}`.
- **name**: A unique name for the route (used for generation/debugging).

## `RouteDiscoverer`

The `RouteDiscoverer` scans the directory defined in `waffle.paths.controllers` (in `app.yaml`).
- It uses Reflection to find methods tagged with `#[Route]`.
- It builds a simple route collection array.
