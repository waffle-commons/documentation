# Security Reference (`waffle-commons/security`)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; SEC-01 CSRF subject-binding + `WAFFLE_SID` rotation · SEC-04 fail-closed CORS

Hierarchical Attribute-Based Access Control (ABAC) for the Waffle Framework, plus a fully stateless CSRF protection layer (signed double-submit with per-browser binding) and a container decorator (`SecureContainer`) that hardens service retrieval. Enforcement is wired through PSR-15 middleware that sits after routing in the pipeline.

## Canonical Beta-1 middleware order

```
ErrorHandler → TrustedHost → Cors → AnonymousSession → Authentication → Routing → Csrf → Security → SecureHeaders → Dispatcher
```

`CorsMiddleware` runs early (before routing) so a pre-flight is answered before the router's OPTIONS short-circuit. `AnonymousSessionMiddleware` issues a per-browser `WAFFLE_SID` cookie (32 random bytes, base64url, 30-day Max-Age, HttpOnly, SameSite=Lax, Secure on HTTPS) and **rotates** it on authentication (SEC-01 session fixation). The SID is the per-browser handle the CSRF HMAC binds to while anonymous; once authenticated the HMAC binds to the user's subject instead.

## Required configuration

```yaml
# config/app.yaml
waffle:
  security:
    level: 5
    csrf:
      # 32+ byte signing secret. Resolved from env WAFFLE_CSRF_SECRET when set
      # to '%env(...)%'. In prod a missing/short value aborts boot.
      secret: '%env(WAFFLE_CSRF_SECRET)%'
```

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

1. Reads `_classname` and `_method` from the request attributes (set by `CoreRoutingMiddleware`).
2. Calls `SecureContainer::analyze($controller, $method, $request)` — the fail-closed, context-aware `#[Voter]` (Layer-2) authorization path. Each voter is resolved from the PSR-11 container and asked `decide(SecurityContextInterface $ctx, mixed $subject = null): bool`, so it sees the authenticated identity and the request; with no voter **and** no `#[PublicAccess]`, the request is denied. (Object-integrity — the `Level1Rule`…`Level10Rule` ladder via `Security::analyze()` — is the separate Layer-1 check that runs at container resolution; see [The Two Authorization Layers](../explanation/security-two-layer-authorization.md).)
3. Lets the request pass on success; raises `SecurityExceptionInterface` on failure (rendered as `403` by the error handler).
4. Logs access denials via the injected PSR-3 logger.

### `Waffle\Commons\Security\Middleware\AnonymousSessionMiddleware`

PSR-15 middleware that ensures every request carries an anonymous-session identifier (`WAFFLE_SID` cookie). On first hit it mints a 32-byte random value, publishes it as the `_anon_sid` request attribute for downstream middleware, and emits a `Set-Cookie` header on the response. Subsequent requests reuse the existing cookie. **No `$_SESSION`, no server-side store** — the cookie carries the identifier. Required for `CsrfMiddleware` to bind tokens to a single browser.

```php
public function __construct(
    private ?SecurityContextInterface $securityContext = null,
    private bool $forceSecureCookie = false,
) {}
```

- **SID rotation (SEC-01 session fixation):** when the injected `SecurityContextInterface` reports the request authenticated **and** a pre-existing `WAFFLE_SID` was presented, the cookie is re-issued with a fresh value — a SID an attacker fixed before login can never ride the now-privileged session. (The middleware sits above `AuthenticationMiddleware`, so the context is populated as the response unwinds.)
- **`$forceSecureCookie`:** emit `Secure` unconditionally (for deployments behind a TLS-terminating proxy forwarding plain HTTP); otherwise `Secure` is set only on HTTPS.

### `Waffle\Commons\Security\Middleware\CsrfMiddleware`

Stateless CSRF protection using **signed double-submit cookies with principal binding** (SEC-01). Tokens are self-validating signed payloads:

```
nonce (16 bytes) || expiresAt (8 bytes BE uint64) || HMAC-SHA256(nonce || expiresAt || id || principal, secret)
```

The logical `id` prevents cross-form replay; the **binding principal** prevents cross-session replay. `CsrfBindingResolver::resolve()` derives that principal from the request, highest precedence first:

1. `auth:<subject>` — the authenticated identity's subject (from the `_auth_identity` attribute set by `AuthenticationMiddleware`);
2. `anon:<sid>` — the per-browser anonymous SID (from `_anon_sid`).

Folding the **subject** into the HMAC means a token minted while anonymous (`anon:…`) is mathematically invalid the instant the session authenticates (`auth:…`) — defeating **session tossing**. A null binding (no SID published) fails closed. **No PHP sessions, no cache, no server-side storage.** Validation uses `hash_equals()` for constant-time comparison (SEC-03).

### `Waffle\Commons\Security\Middleware\CorsMiddleware`

Fail-closed Cross-Origin Resource Sharing (SEC-04). A cross-origin request is permitted **only** when its `Origin` exactly matches the configured allow-list; an empty allow-list rejects every cross-origin request. Same-origin and `Origin`-less (server-to-server, curl) requests pass through untouched. Wire it before routing (see the middleware order above), constructed with a `CorsPolicy` and a PSR-17 `ResponseFactoryInterface`:

```php
use Waffle\Commons\Security\Cors\CorsPolicy;
use Waffle\Commons\Security\Middleware\CorsMiddleware;

new CorsMiddleware(
    new CorsPolicy(
        allowedOrigins: ['https://app.example.com'], // exact scheme://host[:port]; '*' only without credentials
        allowedMethods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
        allowedHeaders: ['Content-Type', 'Authorization', 'X-CSRF-Token'],
        allowCredentials: false,
        maxAge: 600,
    ),
    $responseFactory,
);
```

- **Pre-flight** (`OPTIONS` + `Access-Control-Request-Method`) → `204` with the negotiated `Access-Control-*` headers when allowed, `403` when not — the handler never runs.
- **Disallowed cross-origin actual request** → `403` *before* the handler runs (a forged cross-site write never executes).
- **Allowed response** → decorated with `Access-Control-Allow-Origin` (the exact origin, never a blanket `*` for credentialed policies) + `Vary: Origin`.
- `CorsPolicy` throws if a wildcard `*` origin is combined with `allowCredentials` (Fetch-standard rule).

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
