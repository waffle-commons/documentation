# Security Reference (`waffle-commons/security`)

The Security component enforces Attribute-Based Access Control (ABAC) via the Container and HTTP Middleware.

## `SecurityMiddleware`

The `SecurityMiddleware` is a PSR-15 middleware that intercepts incoming requests to perform security audits before the controller is executed.

- It extracts the controller and method from the request attributes.
- It triggers the `SecureContainer::analyze()` method.
- It logs access denials via PSR-3 for auditing.

## `SecureContainer`

The `SecureContainer` is a decorator that wraps the inner PSR-11 container.
- It intercepts every `get()` call to perform security analysis on resolved objects.
- It provides an `analyze(string $controller, string $method)` method for on-demand auditing (used by the Middleware).

## Security Attributes

### `Waffle\Commons\Contracts\Security\Attribute\Voter`

Used to declare security requirements on classes or methods.

```php
#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD | Attribute::IS_REPEATABLE)]
final readonly class Voter
{
    public function __construct(public string $name) {}
}
```

The `name` must be the FQCN of a class implementing `Waffle\Commons\Contracts\Security\VoterInterface`.

## Security Rules (Hierarchy Levels)

Waffle enforces a base security level configured in `app.yaml`.

- **Level 1 (`Level1Rule`)**: Consistency Check.
- **Level 2 - 9**: Intermediate validation rules (Code Integrity, Property Type safety).
- **Level 10 (`Level10Rule`)**: Paranoid Check. Maximum strictness.
