# The Universal Authentication Bridge (RFC-021)

## Why a dedicated component

Waffle's `security` component answers *"may you do this?"* (fail-closed ABAC, voters,
CSRF). Nothing in the framework answered *"who are you?"*. RFC-021 introduces that layer
as its own component — `waffle-commons/auth` — with a strict boundary:

| Concern | Component | RFC |
|---|---|---|
| Authentication — *who are you?* | `waffle-commons/auth` | RFC-021 |
| Authorization — *may you do this?* | `waffle-commons/security` | RFC-002 |

The design goal is captured by the RFC's title: **universal**. Connecting a PHP
application to Google, Microsoft, Keycloak, Auth0, an API-key partner, or a legacy
monolith traditionally means one vendor SDK per provider — each with its own HTTP stack,
cache, and global state. That is technical debt by accretion. The bridge replaces all of
it with **one contract surface** (`Waffle\Commons\Contracts\Auth`) and native, PSR-only
implementations: every scheme is just an `AuthenticatorInterface` (inbound) and/or a
`CredentialsProviderInterface` (outbound).

## The three inbound rules

1. **First match wins.** Authenticators are consulted in registration order; the first
   whose `supports()` recognizes the request's credential shape performs the validation.
2. **Fail-closed.** A supporting scheme that rejects credentials throws (401/403). There
   is no fallback to the next scheme and never a silent anonymous downgrade — a tampered
   token must never become an anonymous request that happens to hit a public route.
3. **Anonymous is explicit.** No recognizable credentials ⇒ the bridge returns `null` and
   the request proceeds anonymously. Whether anonymous may access the route is the
   authorization layer's decision (`#[PublicAccess]` / voters), not the bridge's.

## The SecurityContext and the zero-leak mandate

The `SecurityContext` is the **only mutable service** in the component: a request-scoped
holder of the verified identity and the original client IP. In FrankenPHP resident-worker
mode, mutable state is where identities leak from request N into request N+1 — the exact
class of bug the statelessness mandate exists to prevent.

The context therefore implements `ResettableInterface`, and the kernel's reset chain wipes
it after **every** request: `WaffleRuntime` (finally block) → `AbstractKernel::reset()` →
`SecureContainer::reset()` (the decorator forwards) → `Container::reset()` (every
registered resettable instance). Implementing RFC-021 hardened two links of that chain:
the `SecureContainer` decorator now forwards `reset()` to the container it wraps, and
`Container::set()` memoizes already-built objects so boot-injected services participate in
the reset loop even when nothing ever `get()`s them.

## The Gateway Assertion Protocol

The bridge's service-to-service scheme — the original heart of RFC-021 — lets a Waffle
edge service propagate a verified identity to a downstream application (Symfony, Laravel,
plain PHP, or another Waffle app) without re-authentication. This is the Strangler-Fig
pattern's missing piece: the monolith keeps working, but stops owning login.

The wire format is deliberately boring: `base64url(JSON) . hex(HMAC-SHA256)` in the
`X-Wfl-Assert-User` header, with seven compact claims (`usr`, `eml`, `rol`, `ten`, `iat`,
`exp`, `iph`). Three properties make it safe:

- **Constant-time verification** — the MAC comparison uses `hash_equals()`; a single
  mutated character yields HTTP 403 with no timing side channel.
- **A 5-second window** — `exp` must be in the future AND `exp − iat ≤ 5 s`. The second
  clause stops a compromised or buggy sender from minting long-lived assertions; replays
  die within seconds either way.
- **Keyed IP binding** — the payload carries `iph = HMAC-SHA256(client IP, secret)`, not
  the raw address. The receiver recomputes the hash over the client IP *it* observes; a
  hijacked assertion presented from another address fails the comparison. Privacy and
  anti-hijacking in one claim — and because `iph` is signed, a forwarded `X-Forwarded-For`
  cannot be forged without breaking the signature.

A receiver needs nothing from Waffle: the workspace's fake legacy monolith verifies
assertions in ~40 lines of plain PHP (`workspace/docker/legacy/public/router.php`).

## JWT and OAuth without debt

The JWT validator implements the few rules that matter, strictly: an explicit algorithm
allow-list (`alg: none` is rejected unconditionally and cannot be allow-listed), key/type
consistency (an RSA public key can never feed the HMAC path — the classic HS/RS confusion
attack), signature verification *before* any claim is read, and mandatory `iss`/`aud`
pinning. RS256 keys come from a static PEM or from the provider's JWKS document — fetched
over PSR-18, converted from JWK members to PEM by a ~40-line native DER builder, and
cached in any PSR-16 cache. No JOSE mega-library.

The OAuth2/OIDC engine is stateless by construction: `state`, `nonce`, and the PKCE
verifier (S256 only) travel in a short-TTL HMAC-signed cookie — the same codec discipline
as the assertion protocol — never in `$_SESSION`. ID tokens are validated by the JWT
subsystem, including the transaction `nonce`. Presets ship for Google, Microsoft, and
GitHub (OAuth2-only, identity via userinfo); every other OIDC provider works through
discovery.

## Fail-closed bootstrapping

Every secret-requiring scheme refuses to construct when `WAFFLE_AUTH_SECRET` is missing,
empty, or shorter than 32 bytes, aborting the kernel boot with
`MissingAuthSecretException`. A misconfigured bridge can never degrade into an
unauthenticated bypass — the same posture as the CSRF subsystem and the ABAC default.

## See also

- [How to Authenticate Requests](../how-to/authentication.md)
- [Reference: Auth component](../reference/auth.md)
- [Fail-Closed ABAC](security-fail-closed-abac.md) — the authorization side
- RFC-021 (monorepo `project_system/RFCs/`) — normative specification
