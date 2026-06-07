# How-To: Validate & Cleanse Input with `Assert`

> **Goal:** validate *and* normalise request data declaratively, inside PHP 8.5 property hooks, with no validation library — using the stateless `Assert` helper from `waffle-commons/utils`.

Waffle keeps input validation where it belongs: in the value object, expressed with a **Property Hook** (see [Quick Start §4](../tutorials/quick-start.md)). `Assert` makes that ergonomic for the common cases — each method **validates and returns the cleansed value**, so a whole field fits on one line.

## When to reach for `Assert`

- You want the *fail-fast* behaviour a hook needs — throw on the first bad value.
- You want the value **normalised** as it is stored (trim, lower-case…), not just checked.
- The check is a common primitive: e-mail, UUID, length, numeric range, IP/CIDR, port, filesystem path.

To collect *every* violation at once (e.g. a large form), use `Contracts\Validation\ValidatorInterface` instead. The two are complementary.

## 1. Write a `#[Dto]` with short-set hooks

`src/Dto/RegistrationInput.php`:

```php
<?php

declare(strict_types=1);

namespace App\Dto;

use Waffle\Commons\Contracts\Attribute\Dto;
use Waffle\Commons\Utils\Assert;

#[Dto]
final class RegistrationInput
{
    public private(set) string $email {
        set => Assert::email($value);                            // trimmed + lower-cased
    }

    public private(set) string $username {
        set => Assert::length(Assert::notEmpty($value), 3, 32);  // trimmed, then 3–32 chars
    }

    public private(set) int $age {
        set => Assert::range($value, 18, 130);
    }

    public private(set) string $signupIp {
        set => Assert::ip($value);                               // trimmed + lower-cased
    }

    public function __construct(string $email, string $username, int $age, string $signupIp)
    {
        $this->email = $email;
        $this->username = $username;
        $this->age = $age;
        $this->signupIp = $signupIp;
    }
}
```

Each assertion returns the cleansed value, which is what the hook stores. `Assert::length(Assert::notEmpty($value), 3, 32)` shows the methods composing: `notEmpty` trims first, then `length` bounds the result.

## 2. Type-hint it in a controller

```php
#[Route(path: 'register', name: 'register')]
public function register(RegistrationInput $input): ResponseInterface
{
    // $input is already validated and cleansed.
    return $this->jsonResponse(data: [
        'email'    => $input->email,
        'username' => $input->username,
        'age'      => $input->age,
    ]);
}
```

The `ControllerArgumentResolver` decodes the JSON body, maps keys to constructor parameters by name, and instantiates the DTO — the hooks validate before your action runs.

## 3. See it work

A valid body comes back cleansed (note the trimmed, lower-cased e-mail and IPv6):

```bash
curl -k -X POST https://localhost/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"  ADA@Example.COM ","username":"  ada  ","age":36,"signupIp":"2001:DB8::1"}'
# → 200 {"email":"ada@example.com","username":"ada","age":36,"signup_ip":"2001:db8::1"}
```

An invalid one never reaches the action — the hook throws, and the `ErrorHandlerMiddleware` renders an RFC 7807 `422`:

```bash
curl -k -X POST https://localhost/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"not-an-email","username":"ada","age":36,"signupIp":"10.0.0.1"}'
```

```json
{
  "type":   "about:blank",
  "title":  "Unprocessable Entity",
  "status": 422,
  "detail": "\"not-an-email\" is not a valid email address."
}
```

## Notes

- **Custom messages.** Every assertion takes an optional trailing message: `Assert::email($value, 'Use your work e-mail.')`.
- **The `field` key.** `Assert` checks are *value-level*, so the 422 payload has no `field` (the assertion doesn't know the property name). When you need it, throw `Waffle\Exception\ValidationException` with `field:` from a full hook — see [Quick Start §4](../tutorials/quick-start.md) and [How-To: Error Handling](error-handling.md).
- **Worker-safe.** `Assert` holds no state; it is safe to call under FrankenPHP worker mode (verified by `composer igor`).

This recipe ships as a runnable demo in the skeleton (`src/Dto/RegistrationInput.php` + `POST /register`). For the full method list, see [Utils Reference → `Assert`](../reference/utils.md#assert--input-validation--cleansing).
