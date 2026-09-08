# How-To: Secure a Controller

> Waffle ships two complementary authorization layers: a **global security level** (the object-integrity rule ladder applied by the `SecureContainer`) and **per-method attributes** (`#[Voter]`, `#[RequiresCsrfToken]`, `#[PublicAccess]`) for request-level control — with a **fail-closed ABAC default** that denies any action without an explicit policy.

> **Authentication vs. authorization.** This page covers **authorization** (*may you do
> this?* — `waffle-commons/security`). Establishing *who* the caller is — OAuth2/OIDC,
> JWT Bearer, gateway assertions, API keys, passkeys — is the job of the **Universal
> Authentication Bridge** (`waffle-commons/auth`, RFC-021): see
> [How to Authenticate Requests](authentication.md).

## 1. Configure the global security level

The security level defines the base rigorousness of the **object-integrity** checks performed on every object the `SecureContainer` resolves. The kernel reads `waffle.security.level` from `config/app.yaml` and constructs `Waffle\Commons\Security\Security` with it. Levels are integers from `1` (permissive) to `10` (strictest).

```yaml
# config/app.yaml
waffle:
  security:
    level: 10     # 1 = permissive … 10 = strictest object-integrity checks
```

When a service is pulled from the container, `SecureContainer::get()` runs it through `Security::analyze()`: every `LevelNRule` from 1 up to your configured level is evaluated, and any rule returning `false` raises a `SecurityExceptionInterface`, which the `ErrorHandlerMiddleware` renders as HTTP `403`.

## 2. Know the two authorization layers

There is **no per-route `#[Rule]` attribute**. The level ladder above is an application-wide *structural* gate — it asks whether an object is well-formed and safe to wire, independent of who is calling. Per-route access control is carried entirely by context-aware **`#[Voter]` attributes** (next section), which see the authenticated identity and the resource under decision. The two layers are complementary, not redundant — see [The Two Authorization Layers](../explanation/security-two-layer-authorization.md) for why both `analyze()` entry points exist.

## 3. ABAC voters via `#[Voter]`

For decisions that depend on per-request state (identity, roles, ownership), declare a `#[Voter]`. Its `decide()` method runs at dispatch time and receives two arguments:

- **`SecurityContextInterface $ctx`** — the request-scoped security context: the authenticated identity (`getIdentity()`, `null` when anonymous) and the client IP. Roles live on the identity as a `list<string>` (`$ctx->getIdentity()?->roles`).
- **`mixed $subject`** — the resource under decision. Until a richer resource resolver lands, this is the PSR-7 `ServerRequestInterface`, so a voter can read route attributes, the body, or load the target entity itself.

```php
use Waffle\Commons\Contracts\Auth\SecurityContextInterface;
use Waffle\Commons\Contracts\Security\Attribute\Voter;
use Waffle\Commons\Contracts\Security\VoterInterface;

#[Voter(name: IsAdminVoter::class)]
final class AdminController
{
    public function deleteAction(): ResponseInterface { /* … */ }
}

final class IsAdminVoter implements VoterInterface
{
    public function decide(SecurityContextInterface $ctx, mixed $subject = null): bool
    {
        // Roles are a list<string> on the verified identity; anonymous ⇒ deny.
        return in_array('ROLE_ADMIN', $ctx->getIdentity()?->roles ?? [], strict: true);
    }
}
```

Voters are resolved through the **PSR-11 container**, so they may declare constructor dependencies (repositories, clocks, policy services). Keep them **stateless** — the container memoizes each voter for the worker's lifetime.

### Ownership / IDOR decisions

Because `decide()` receives both the caller (`$ctx`) and the resource context (`$subject`), object-level ownership — the classic [IDOR](https://owasp.org/Top10/A01_2021-Broken_Access_Control/) defence — is expressible directly:

```php
use Psr\Http\Message\ServerRequestInterface;
use Waffle\Commons\Contracts\Auth\SecurityContextInterface;
use Waffle\Commons\Contracts\Security\VoterInterface;

final class OwnerVoter implements VoterInterface
{
    public function __construct(private DocumentRepository $documents) {}

    public function decide(SecurityContextInterface $ctx, mixed $subject = null): bool
    {
        $identity = $ctx->getIdentity();
        if ($identity === null || !$subject instanceof ServerRequestInterface) {
            return false; // anonymous, or no request context → deny
        }

        $document = $this->documents->find((string) $subject->getAttribute('id'));

        // Deny unless the authenticated subject owns the requested document.
        return $document !== null && $document->ownerId === $identity->subject;
    }
}
```

`#[Voter]` is repeatable (`Attribute::IS_REPEATABLE`); the request is denied if **any** voter returns `false`.

> **What `$subject` is (and isn't).** By default the current PSR-7
> `ServerRequestInterface` is threaded in as `$subject`, so an ownership voter reads
> the identifier off the request (route attribute, query, or body) and loads the
> resource itself — exactly as `OwnerVoter` does above. Alternatively, the application
> may wire a `Waffle\Commons\Contracts\Security\SubjectResolverInterface` into
> `SecurityMiddleware` (SEC-05): it runs post-routing / pre-dispatch, turns a route
> parameter into the hydrated domain object, and that object then reaches every voter
> as `$subject` — with any resolution failure denied fail-closed (`403`), never
> silently downgraded. Either way, treat the gate as deny-by-default **policy
> enforcement**: never assume the framework has already fetched or authorized the
> target object.

## 4. Fail-closed default — explicit `#[PublicAccess]` for public endpoints

Beta-1 enables **fail-closed ABAC**: an action that carries no `#[Voter]` at all (neither on the method nor its declaring class) is denied with HTTP `403`. The previous "no rules ⇒ allow" semantics are gone — forgetting to attach a voter is no longer silently permissive.

For endpoints that genuinely require no authorization (health probes, login forms, public APIs) attach `#[PublicAccess]` explicitly:

```php
use Waffle\Commons\Contracts\Security\Attribute\PublicAccess;
use Waffle\Commons\Contracts\Routing\Attribute\Route;

final class HealthController
{
    #[Route(path: '/health', name: 'health')]
    #[PublicAccess]
    public function ping(): ResponseInterface { /* … */ }
}
```

`#[PublicAccess]` is **method-only** (`Attribute::TARGET_METHOD`, SEC-05): every public action must carry its own opt-out, so adding a method to a controller never silently inherits an exposure decision. A class-level placement is inert — the `SecureContainer` never reads it, and an action without its own `#[Voter]` or method-level `#[PublicAccess]` still gets a `403`. See the [`#[PublicAccess]` attribute reference](../reference/attributes-public-access.md) and the [Fail-Closed ABAC explanation](../explanation/security-fail-closed-abac.md).

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

Waffle ships a fully **stateless signed double-submit** CSRF subsystem (`Waffle\Commons\Security\Csrf\CsrfTokenManager`): tokens are self-validating `HMAC-SHA256(nonce ‖ expiresAt ‖ id ‖ principal)` payloads. **No PHP sessions, no cache, no Redis** — every CSRF check is a pure function of the request, the configured signing secret, and the binding principal resolved by `CsrfBindingResolver`. Two pieces of context are folded into the HMAC:

- the **logical id** (e.g. `form:login`) — prevents cross-form replay;
- the **binding principal** — `auth:<subject>` when authenticated, else `anon:<WAFFLE_SID>` — prevents cross-session replay. A token minted while anonymous is mathematically invalid once the session authenticates (SEC-01 session-tossing defence).

Operational requirements:

1. Set `waffle.security.csrf.secret` (or env `WAFFLE_CSRF_SECRET`) to a 32+ byte value. Production refuses to boot without one; non-prod falls back to an ephemeral per-process secret.
2. Place `AnonymousSessionMiddleware` **before** `CsrfMiddleware` in the pipeline — the kernel factories do this for you in the canonical order.

See the [CSRF explanation page](../explanation/security-csrf-double-submit.md) for the full design rationale.

## 6. How the layers compose

Canonical middleware order (already wired by the skeleton's `AppKernelFactory` — the security spine is unchanged since Beta-1; Beta-5 adds the telemetry and transaction slots):

```
ErrorHandler → Metrics → Tracing → TrustedHost → Cors → AnonymousSession → Authentication → Routing → Csrf → Security → TransactionIsolation → SecureHeaders → Dispatcher
```

`SecurityMiddleware` reads `_classname` + `_method` from the request attributes (set by `CoreRoutingMiddleware`), runs the optional `SubjectResolverInterface` (SEC-05 — a resolution failure denies fail-closed), then delegates to `SecureContainer::analyze($controller, $method, $request, $resolvedSubject)`:

1. Collects `#[Voter]` attributes from the method **and** its declaring class.
2. **Fail-closed:** if the voter list is empty AND no method-level `#[PublicAccess]` is attached → `SecurityException(403)`.
3. Otherwise resolves each voter from the PSR-11 container and runs `VoterInterface::decide($ctx, $subject)` — the resolved subject when a resolver is wired, else the PSR-7 request; any `false` → `SecurityException(403)`.
4. Any failure → `SecurityExceptionInterface` → RFC 7807 `403` via the error handler.

`CsrfMiddleware` runs the CSRF check on actions tagged `#[RequiresCsrfToken]`, using the SID published by `AnonymousSessionMiddleware`.

`SecureContainer` also wraps the PSR-11 container and applies `Security::analyze()` before every `get()` — preventing low-privilege code from pulling sensitive services. Even a service retrieved manually from the container has passed the audit by the time it reaches your constructor:

```php
public function __construct(private UserService $service)
{
    // If we are here, $service has been analyzed and approved.
}
```

## 7. Handling file uploads safely (SEC-05)

**Never** pass attacker-influenced metadata — most importantly `UploadedFileInterface::getClientFilename()` — straight into a transfer command. The client controls that value and can embed traversal sequences (`../../etc/cron.d/evil`) to escape your storage directory.

`Waffle\Commons\Http\UploadedFile::moveTo()` already rejects a destination containing `../`, `..\`, or a null byte (it throws a `ValidationException`, which is an `\InvalidArgumentException`). Build the destination from a value **you** control, and confine any user-supplied fragment with `Assert`:

```php
use Waffle\Commons\Utils\Assert;

// Generate the stored name yourself; never trust the client filename.
$storedName = bin2hex(random_bytes(16)) . '.bin';

// Or, if you must keep a user-supplied sub-path, confine it to a base dir:
$target = Assert::within('/var/app/uploads', $userSuppliedSubPath); // throws if it escapes

$uploadedFile->moveTo($target);
```

`Assert::safePath()` rejects any traversal segment; `Assert::within($base, $path)` additionally guarantees the resolved target stays inside `$base`. See the [Utils reference](../reference/utils.md).

Since the Beta6 audit, `UploadedFile` also takes an optional `baseDir` sixth constructor parameter: when set, `moveTo()` itself enforces `Assert::within($baseDir, $targetPath)` on every destination — configure it wherever you construct upload objects with a known storage root, so containment holds even if a call site forgets the manual check.
