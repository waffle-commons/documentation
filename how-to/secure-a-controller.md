# How-To: Secure a Controller

> **Beta 0** — Waffle ships two complementary security layers: a **global security level** that gates every controller via the rule ladder, and **per-method attributes** (`#[Rule]`, `#[Voter]`, `#[RequiresCsrfToken]`) for finer-grained control.

## 1. Configure the global security level

The kernel reads `waffle.security.level` from `config/app.yaml` and constructs `Waffle\Commons\Security\Security` with it. Levels are integers from `1` (public) to `10` (paranoid).

```yaml
# config/app.yaml
waffle:
  security:
    level: 5      # see Constant::SECURITY_LEVEL1 … SECURITY_LEVEL10
```

When `SecurityMiddleware` runs the resolved controller through `Security::analyze()`, every `LevelNRule` from 1 up to your configured level is evaluated. Any `LevelNRule::isValid()` returning `false` raises a `SecurityExceptionInterface`, which the `ErrorHandlerMiddleware` renders as HTTP `403`.

## 2. Tighten with `#[Rule]` on a method or class

Use `Waffle\Commons\Contracts\Security\Attribute\Rule` to declare the **minimum** security level a route requires. If the kernel's effective level is below that, execution is denied.

```php
use Waffle\Commons\Contracts\Security\Attribute\Rule;
use Waffle\Commons\Contracts\Constant\Constant;

final class AdminController
{
    #[Rule(level: Constant::SECURITY_LEVEL10)]
    public function dropDatabase(): ResponseInterface
    {
        // Only reachable when the kernel's level >= 10.
    }
}
```

Class-level `#[Rule]` applies to every method on the class; method-level `#[Rule]` overrides the class default for that method.

## 3. ABAC voters via `#[Voter]`

For decisions that depend on per-request state (user roles, ownership, etc.), declare a `#[Voter]` whose `decide()` method runs at dispatch time:

```php
use Waffle\Commons\Contracts\Security\Attribute\Voter;
use Waffle\Commons\Contracts\Security\VoterInterface;

#[Voter(name: IsAdminVoter::class)]
final class AdminController
{
    public function deleteAction(): ResponseInterface { /* … */ }
}

final class IsAdminVoter implements VoterInterface
{
    public function decide(): bool
    {
        // Check user roles / session / signed claims here.
        return false;
    }
}
```

`#[Voter]` is repeatable (`Attribute::IS_REPEATABLE`); the request is denied if **any** voter returns `false`.

## 4. CSRF on mutating routes

For `POST` / `PUT` / `PATCH` / `DELETE` endpoints, add `#[RequiresCsrfToken]`:

```php
use Waffle\Commons\Contracts\Security\Csrf\Attribute\RequiresCsrfToken;

#[RequiresCsrfToken]
public function transferFunds(ServerRequestInterface $request): ResponseInterface
{
    // CsrfMiddleware validated the double-submit token before reaching here.
}
```

The middleware is stateless — it uses **HMAC-signed double-submit cookies**, with token storage delegated to a `Waffle\Commons\Contracts\Cache\CacheInterface` (typically `RedisCache` in production so multiple workers share token state).

## 5. How the layers compose

`SecurityMiddleware`:

1. Reads `_controller` + `_method` from the request attributes (set by `CoreRoutingMiddleware`).
2. Calls `Security::analyze($controller)` — checks the level ladder.
3. Reads `#[Rule]` attributes — checks declared minimums.
4. Reads `#[Voter]` attributes — calls each `VoterInterface::decide()`.
5. Any failure → `SecurityExceptionInterface` → RFC 7807 `403`.

`CsrfMiddleware` (when configured) runs the CSRF check on attributes tagged `#[RequiresCsrfToken]`.

`SecureContainer` (optional layer) wraps the PSR-11 container and applies `Security::analyze()` before every `get()` — preventing low-privilege code from pulling sensitive services.
