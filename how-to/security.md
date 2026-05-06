# How to Secure Your Application

Waffle implements a global **Attribute-Based Access Control (ABAC)** system integrated into both the Container and the HTTP Pipeline.

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

use Waffle\Commons\Contracts\Security\VoterInterface;

class IsAdminVoter implements VoterInterface
{
    public function decide(): bool
    {
        // Your logic here (e.g. check session, roles, etc.)
        return true; 
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
