# Security Reference (`waffle-commons/security`)

> **Release:** `v0.1.0-beta0`

Hierarchical Attribute-Based Access Control (ABAC) for the Waffle Framework, plus a stateless CSRF protection layer and a container decorator (`SecureContainer`) that hardens service retrieval. Enforcement is wired through PSR-15 middleware that sits after routing in the pipeline.

## `Security` — exact signature

```php
namespace Waffle\Commons\Security;

use Waffle\Commons\Contracts\Config\ConfigInterface;
use Waffle\Commons\Security\Abstract\AbstractSecurity;

class Security extends AbstractSecurity
{
    public function __construct(ConfigInterface $cfg)
    {
        $this->level = $cfg->getInt(key: 'waffle.security.level', default: 1) ?? 1;
    }
}
```

`AbstractSecurity` implements `SecurityInterface::analyze($object, $expectations)`. The single public action is to walk the rule ladder up to `$this->level` and throw a `SecurityExceptionInterface` if any rule fails.

## The security ladder

Concrete rule classes live in `src/Rule/`:

| Class | Level | Scope |
| :--- | :--- | :--- |
| `Level1Rule` | 1 | Public — minimum sanity / type-consistency check. |
| `Level2Rule` … `Level9Rule` | 2-9 | Intermediate validators (typed properties, attribute-level constraints, CSRF cooperation). |
| `Level10Rule` | 10 | Paranoid — maximum strictness. |

The kernel reads `waffle.security.level` from `app.yaml` and constructs `Security` with that level:

```yaml
# config/app.yaml
waffle:
  security:
    level: 5
```

## Middleware

### `Waffle\Commons\Security\Middleware\SecurityMiddleware`

PSR-15 middleware that:

1. Reads `_controller` and `_method` from the request attributes (set by `CoreRoutingMiddleware`).
2. Calls `Security::analyze()` against the resolved controller class.
3. Lets the request pass on success; raises `SecurityExceptionInterface` on failure (rendered as `403` by the error handler).
4. Logs access denials via the injected PSR-3 logger.

### `Waffle\Commons\Security\Middleware\CsrfMiddleware`

Stateless CSRF protection using **double-submit cookies** with HMAC-signed tokens. **No PHP sessions are ever touched** — the implementation is FrankenPHP-safe. Tokens are issued / validated through `Waffle\Commons\Contracts\Security\Csrf\CsrfTokenManagerInterface`, with backing storage delegated to a `CacheInterface` (typically `RedisCache` in production for cross-worker token sharing).

## `SecureContainer`

`Waffle\Commons\Security\Container\SecureContainer` wraps any `Waffle\Commons\Contracts\Container\ContainerInterface` and runs the security check before `get($id)` returns the service — preventing low-privilege code paths from pulling sensitive services out of the container.

## Attributes

All security attributes live in `Waffle\Commons\Contracts\Security\*` (the contracts package — implementations don't redeclare them):

### `Waffle\Commons\Contracts\Security\Attribute\Rule`

Declares the security level required by a controller method or class. The kernel's effective level must be `>=` the declared level for execution to proceed:

```php
use Waffle\Commons\Contracts\Security\Attribute\Rule;
use Waffle\Commons\Contracts\Constant\Constant;

final class AdminController
{
    #[Rule(level: Constant::SECURITY_LEVEL10)]
    public function dangerous(): Response { /* … */ }
}
```

### `Waffle\Commons\Contracts\Security\Attribute\Voter`

Marks a class as an ABAC voter. The class must implement `VoterInterface`.

```php
#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD | Attribute::IS_REPEATABLE)]
final readonly class Voter
{
    public function __construct(public string $name) {}
}
```

`$name` is the FQCN of the voter class. Multiple `#[Voter]` attributes can be stacked on a single method (`IS_REPEATABLE`).

### `Waffle\Commons\Contracts\Security\Csrf\Attribute\RequiresCsrfToken`

Marks a controller method as requiring CSRF token validation. The `CsrfMiddleware` reads this attribute to decide whether to validate the request's token.

## Security exceptions

- `SecurityExceptionInterface` (in contracts) — root interface. The error handler maps this to HTTP `403`.
- `MissingCsrfTokenException` / `InvalidCsrfTokenException` — implementations live in `src/Csrf/Exception/`.

## PHP 8.5 features used

- **Typed integer security levels** declared as typed constants (`Constant::SECURITY_LEVEL1 = 1` … `SECURITY_LEVEL10 = 10`).
- **`final readonly`** value-object attributes (`#[Voter]`, `#[RequiresCsrfToken]`).
- **Constructor property promotion** with explicit access on every rule class.
- **`#[Override]`** on every `analyze()` override.
