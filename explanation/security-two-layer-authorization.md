# The Two Authorization Layers

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *Formalized in Beta-5 (ARCH-01)*
> **Diátaxis quadrant:** Explanation
> **Tracks:** AUTHZ-01, ARCH-01

## Why there are two `analyze()` methods

Waffle enforces authorization at two distinct layers, and both entry points happen to be named `analyze()` — historically a source of confusion. They are **complementary, not redundant**; Beta-5 keeps both with clearly separated responsibilities.

| Layer | Method | Question it answers | Driven by |
|-------|--------|---------------------|-----------|
| **1 — Object integrity** | `SecurityInterface::analyze(object $object, array $expectations = [])` | "Is this *service/object* structurally allowed to be wired and used at the configured strictness?" | The **Level 1–10 rule ladder**, set in app YAML (`waffle.security.level`). |
| **2 — Request authorization** | `SecureContainer::analyze(string $controller, string $method, ?ServerRequestInterface $request = null)` | "Is *this caller* allowed to perform *this route/method* on *this resource*?" | Context-aware **`#[Voter]`** attributes (fail-closed). |

## Layer 1 — object integrity (the Level ladder)

When a service is pulled from the container, `SecureContainer::get($id)` runs `SecurityInterface::analyze()` on the resolved object. `AbstractSecurity` walks the rule ladder (`Level1Rule`…`Level10Rule`) **up to the level configured for the application** and throws a `SecurityExceptionInterface` if any rule fails. The level is an explicit, documented application setting:

```yaml
# config/app.yaml
waffle:
  security:
    level: 10   # 1 = permissive … 10 = strictest object-integrity checks
```

This layer is about *structural* trust — the object is well-formed, correctly typed, and safe to wire. It is independent of who is calling.

## Layer 2 — request authorization (context-aware voters)

`SecurityMiddleware` calls `SecureContainer::analyze($controller, $method, $request)` once routing has resolved the target. This layer collects `#[Voter]` attributes from the method and its declaring class, resolves each voter from the **PSR-11 container** (so voters may have their own dependencies), and calls:

```php
$voter->decide($securityContext, $request);
```

The voter sees **who** is calling — the request-scoped `SecurityContextInterface` (the authenticated identity — with its `roles` — and the client IP) — and **what** is under decision — the subject, currently the PSR-7 request. That is what makes ownership / IDOR rules expressible (see [Secure a Controller](../how-to/secure-a-controller.md)). With no voter *and* no `#[PublicAccess]`, the request is denied `403` — see [Fail-Closed ABAC](security-fail-closed-abac.md).

## How they compose

A request can be rejected by **either** layer:

1. A malformed or untrusted service fails **Layer 1** at container-resolution time, before any controller runs.
2. A well-formed service whose action the caller is not authorized for fails **Layer 2** at dispatch time.

Neither layer subsumes the other: object integrity is not about identity, and voters do not re-validate object structure. Keeping them separate is deliberate — each has exactly one job.
