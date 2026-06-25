# `#[PublicAccess]` — Attribute Reference

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *No behavioural changes since Beta-1*
> **Diátaxis quadrant:** Reference
> **Component:** `waffle-commons/contracts`
> **Namespace:** `Waffle\Commons\Contracts\Security\Attribute`

Explicit opt-out from the Beta-1 fail-closed ABAC default. A controller action that carries no `#[Voter]` will be denied with HTTP 403 unless the action — or its declaring class — also carries `#[PublicAccess]`.

## Signature

```php
namespace Waffle\Commons\Contracts\Security\Attribute;

#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
final readonly class PublicAccess {}
```

- **Targets:** `TARGET_CLASS | TARGET_METHOD`.
- **Repeatable:** No. Adding the attribute twice on the same target is a syntactically valid but redundant declaration.
- **Constructor parameters:** None. The attribute is a tag, not a configuration carrier.

## Resolution semantics

`Waffle\Commons\Security\Container\SecureContainer::analyze($request, $controller, $method)` consults `#[PublicAccess]` only when the target carries no `#[Voter]`. The check order is:

1. Has the **method** any `#[Voter]`? → Run the voters; `#[PublicAccess]` is ignored.
2. Has the **class** any `#[Voter]`? → Run the voters; `#[PublicAccess]` is ignored.
3. Has the **method** `#[PublicAccess]`? → Permit.
4. Has the **class** `#[PublicAccess]`? → Permit.
5. Otherwise → Deny with `SecurityException(403, …)`.

The implication: **adding any `#[Voter]` overrides `#[PublicAccess]` on the same scope.** A controller annotated `#[PublicAccess]` at the class level whose method carries `#[Voter(AdminVoter::class)]` still requires the voter to approve. This matches the principle that explicit policy beats default policy.

## Examples

### Public health probe (method-level)

```php
use Waffle\Commons\Contracts\Security\Attribute\PublicAccess;
use Waffle\Commons\Contracts\Routing\Attribute\Route;

final class HealthController
{
    #[Route(path: '/health', name: 'health')]
    #[PublicAccess]
    public function ping(): Response { /* ... */ }
}
```

### Public-facing controller with one protected action (class-level + override)

```php
use Waffle\Commons\Contracts\Security\Attribute\PublicAccess;
use Waffle\Commons\Contracts\Security\Attribute\Voter;
use App\Security\Voter\AdminOnly;

#[PublicAccess]
final class StatusController
{
    #[Route(path: '/status', name: 'status.public')]
    public function summary(): Response { /* ... */ }

    #[Route(path: '/status/internal', name: 'status.internal')]
    #[Voter(AdminOnly::class)]
    public function detailed(): Response { /* ... */ }
}
```

The class-level attribute makes `summary()` public; `detailed()` is protected because its own `#[Voter]` overrides the default.

## Common mistakes

| Pattern | Result | Why |
| :--- | :--- | :--- |
| Mixed `#[PublicAccess]` *and* `#[Voter]` on the same method. | Voter runs; `#[PublicAccess]` ignored. | A voter declares intent to enforce; `#[PublicAccess]` only applies when there is no policy. |
| `#[PublicAccess]` on the class, no method-level attributes anywhere. | All actions are public. | Class default applies. Add `#[Voter]` per method to opt back in. |
| Omitting both `#[Voter]` and `#[PublicAccess]`. | HTTP 403 at runtime. | Fail-closed default — see [Fail-Closed ABAC](../explanation/security-fail-closed-abac.md). |

## Related documents

- [Explanation: Fail-Closed ABAC](../explanation/security-fail-closed-abac.md)
- [Reference: `waffle-commons/security`](./security.md)
