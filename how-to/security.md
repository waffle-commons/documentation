# How to Secure Your Application

Waffle implements a global **Attribute-Based Access Control (ABAC)** system integrated into both the Container and the HTTP Pipeline.

> **Authentication vs. authorization.** This page covers **authorization** (*may you do
> this?* — `waffle-commons/security`). Establishing *who* the caller is — OAuth2/OIDC,
> JWT Bearer, gateway assertions, API keys — is the job of the **Universal Authentication
> Bridge** (`waffle-commons/auth`, RFC-021): see
> [How to Authenticate Requests](authentication.md).

## Configuring the Global Security Level

The security level defines the base rigorousness of checks performed on every object instantiated by the application. Configure this in `config/app.yaml`:

```yaml
waffle:
  security:
    # Level 1 (Basic) to Level 10 (Paranoid)
    level: 5
```

## Securing Controllers with Middleware

The `SecurityMiddleware` automatically protects your routes by analyzing the target controller and method before execution. 

To enable it, ensure it's added to your Kernel's middleware stack. It will:
1. Identify the target Controller and Method.
2. Read any `#[Voter]` attributes.
3. Deny access (403) if any voter rejects the request.

## Fail-closed default (Beta-1)

> An action without a `#[Voter]` is denied with HTTP `403` unless it explicitly carries `#[PublicAccess]`. Missing policy is treated as deny, not allow.

This change is intentional — see [`#[PublicAccess]` reference](../reference/attributes-public-access.md) and the [Fail-Closed ABAC explanation](../explanation/security-fail-closed-abac.md).

## Using Granular Security Attributes

You can apply specific security requirements using the `#[Voter]` attribute on a controller class or a specific method.

```php
use Waffle\Commons\Contracts\Security\Attribute\Voter;
use App\Security\Voter\IsAdminVoter;

#[Voter(name: IsAdminVoter::class)]
class AdminController
{
    public function deleteAction() { ... }
}
```

### Creating a Custom Voter

A Voter must implement `Waffle\Commons\Contracts\Security\VoterInterface`:

```php
namespace App\Security\Voter;

use Waffle\Commons\Contracts\Auth\SecurityContextInterface;
use Waffle\Commons\Contracts\Security\VoterInterface;

class IsAdminVoter implements VoterInterface
{
    public function decide(SecurityContextInterface $ctx, mixed $subject = null): bool
    {
        // $ctx carries the authenticated identity (roles, subject id, client IP);
        // $subject is the resource under decision (currently the PSR-7 request).
        return in_array('ROLE_ADMIN', $ctx->getIdentity()?->roles ?? [], strict: true);
    }
}
```

## Automated Security Analysis

The `SecureContainer` ensures that even if a service is retrieved manually from the container, it passes the security audit:

```php
public function __construct(private UserService $service)
{
    // If we are here, $service has been analyzed and approved.
}
```
