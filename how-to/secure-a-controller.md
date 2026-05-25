# How-To: Secure a Controller

> **Beta 1** — Waffle ships three complementary security layers: a **global security level** (the rule ladder), **per-method attributes** (`#[Rule]`, `#[Voter]`, `#[RequiresCsrfToken]`, `#[PublicAccess]`) for finer-grained control, and a **fail-closed ABAC default** that denies any action without an explicit policy.

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

## 4. Fail-closed default — explicit `#[PublicAccess]` for public endpoints

Beta-1 enables **fail-closed ABAC**: an action that carries no `#[Voter]` at all (neither on the method nor its declaring class) is denied with HTTP `403`. The previous "no rules ⇒ allow" semantics are gone — forgetting to attach a voter is no longer silently permissive.

For endpoints that genuinely require no authorization (health probes, login forms, public APIs) attach `#[PublicAccess]` explicitly:

```php
use Waffle\Commons\Contracts\Security\Attribute\PublicAccess;
use Waffle\Commons\Routing\Attribute\Route;

final class HealthController
{
    #[Route(path: '/health', name: 'health')]
    #[PublicAccess]
    public function ping(): ResponseInterface { /* … */ }
}
```

A method-level `#[Voter]` always wins over a class-level `#[PublicAccess]`, so mixed-policy controllers stay safe. See the [`#[PublicAccess]` attribute reference](../reference/attributes-public-access.md) and the [Fail-Closed ABAC explanation](../explanation/security-fail-closed-abac.md).

## 5. CSRF on mutating routes

For `POST` / `PUT` / `PATCH` / `DELETE` endpoints, add `#[RequiresCsrfToken]`:

```php
use Waffle\Commons\Contracts\Security\Csrf\Attribute\RequiresCsrfToken;

#[RequiresCsrfToken]
public function transferFunds(ServerRequestInterface $request): ResponseInterface
{
    // CsrfMiddleware validated the signed token before reaching here.
}
```

Beta-1 ships a fully **stateless signed double-submit** CSRF subsystem (`Waffle\Commons\Security\Csrf\CsrfTokenManager`): tokens are self-validating `HMAC-SHA256(nonce ‖ expiresAt ‖ id ‖ sessionId)` payloads. **No PHP sessions, no cache, no Redis** — every CSRF check is a pure function of the request, the configured signing secret, and the per-browser `WAFFLE_SID` cookie issued by `AnonymousSessionMiddleware`. Two pieces of context are folded into the HMAC:

- the **logical id** (e.g. `form:login`) — prevents cross-form replay;
- the **anonymous session id** — prevents cross-browser replay.

Operational requirements:

1. Set `waffle.security.csrf.secret` (or env `WAFFLE_CSRF_SECRET`) to a 32+ byte value. Production refuses to boot without one; non-prod falls back to an ephemeral per-process secret.
2. Place `AnonymousSessionMiddleware` **before** `CsrfMiddleware` in the pipeline — the kernel factories do this for you in the canonical order.

See the [CSRF explanation page](../explanation/security-csrf-double-submit.md) for the full design rationale.

## 6. How the layers compose

Canonical Beta-1 middleware order (already wired by the skeleton's `AppKernelFactory`):

```
ErrorHandler → TrustedHost → AnonymousSession → Routing → Csrf → Security → SecureHeaders → Dispatcher
```

`SecurityMiddleware` then delegates to `SecureContainer::analyze($controller, $method)`:

1. Reads `_classname` + `_method` from the request attributes (set by `CoreRoutingMiddleware`).
2. Collects `#[Voter]` attributes from the method **and** its declaring class.
3. **Fail-closed:** if the voter list is empty AND no `#[PublicAccess]` is attached → `SecurityException(403)`.
4. Otherwise runs every `VoterInterface::decide()` — any `false` → `SecurityException(403)`.
5. Any failure → `SecurityExceptionInterface` → RFC 7807 `403` via the error handler.

`CsrfMiddleware` runs the CSRF check on actions tagged `#[RequiresCsrfToken]`, using the SID published by `AnonymousSessionMiddleware`.

`SecureContainer` also wraps the PSR-11 container and applies `Security::analyze()` before every `get()` — preventing low-privilege code from pulling sensitive services.
