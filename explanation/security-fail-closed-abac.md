# Fail-Closed ABAC

> **Release:** `0.1.0-beta3` &nbsp;|&nbsp; *Architectural choice retained from Beta-1*
> **Diátaxis quadrant:** Explanation
> **Tracks:** SEC-02

## The change in one sentence

Starting with Beta-1, a controller action that carries no `#[Voter]` attributes returns **HTTP 403** by default. The previous behavior — silently allowing the call — was a fail-open anti-pattern (OWASP A01) and is gone.

## What used to happen

`Waffle\Commons\Security\Container\SecureContainer::analyze()` walked all `#[Voter]` attributes on the target class and method and called each voter's `decide()`. If the attribute list was empty, the loop body never ran, and the request flowed through to the dispatcher unchallenged. A developer who forgot to attach a voter to a sensitive action would never see a failure: missing policy meant missing enforcement.

## What happens now

`analyze()` rejects the call as soon as it observes an empty voter list, **unless** the target carries a `#[PublicAccess]` attribute. The denial is a `SecurityException` with HTTP code `403`, rendered by `ErrorHandlerMiddleware` the same way any other access denial is rendered.

This makes "I forgot to wire a Voter" loud rather than invisible. Forgetting `#[Voter]` produces an outage in development; the developer adds either a real voter or `#[PublicAccess]` and ships. The class of bug where production silently exposes an admin endpoint because a Voter never landed is, by construction, impossible.

## Why `#[PublicAccess]` exists

Some endpoints genuinely require no authorization: a `/health` probe, a `/login` form, a publicly cacheable read endpoint. Forcing those to attach a dummy `AlwaysAllowVoter` would dilute the signal Voters carry. Instead, the developer makes the policy decision *explicit*:

```php
use Waffle\Commons\Contracts\Security\Attribute\PublicAccess;

final class HealthController
{
    #[PublicAccess]
    public function ping(): Response { /* ... */ }
}
```

The attribute is documentation as much as it is mechanism. A `git grep PublicAccess` enumerates every intentional public endpoint in the codebase — exactly the inventory a security reviewer wants.

## Class-level vs method-level placement

- **On the class:** every action in the controller is treated as public *unless* the action itself carries a `#[Voter]`. Method-level voters always override the class-level default.
- **On a method:** only that specific action is public; sibling actions still require their own voters or `#[PublicAccess]`.

The asymmetry favors explicitness: a method that needs protection cannot be accidentally exempted by something written at the class level.

## Order of checks inside `analyze()`

1. Reflect on the controller class and target method.
2. Collect all `#[Voter]` attributes from both.
3. If the list is empty:
   - if the method or class carries `#[PublicAccess]`, return successfully;
   - otherwise throw `SecurityException(403, …)`.
4. Otherwise, run every voter; the first that returns `false` throws `SecurityException(403, …)` (consensus pattern — every voter must approve).

The new logic only adds step 3; the voting semantics for non-empty lists are unchanged.

## What this means for existing controllers

A pre-Beta-1 controller that had no `#[Voter]` will start returning 403 on first request after upgrade. The migration is straightforward:

- If the action *should* be public: add `#[PublicAccess]`.
- If the action *should* be authenticated: attach the appropriate `#[Voter(…)]`.

We deliberately did not provide a "legacy fail-open" config flag. The change is the point.

## Related documents

- [Reference: `#[PublicAccess]` attribute](../reference/attributes-public-access.md)
- [Explanation: Stateless CSRF](./security-csrf-double-submit.md)
- [Reference: `waffle-commons/security`](../reference/security.md)
