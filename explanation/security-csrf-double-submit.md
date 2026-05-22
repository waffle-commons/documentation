# CSRF: Stateless Signed Double-Submit with Per-Browser Binding

> **Release:** `v0.1.0-beta1`
> **Diátaxis quadrant:** Explanation
> **Tracks:** SEC-01 (option C — stateless HMAC + anonymous SID binding)

## Why we moved off the cache

Beta-0 stored CSRF tokens in the shared PSR-16 cache under a key derived only from the logical id (`csrf.<sha256(form-id)>`). Because the key carried no user-identifying material, **any token written for `form:login` overwrote every other request's token for the same id** — including those of other users on a shared Redis backend. An attacker holding any valid token for `form:login` could submit it against another user's session and bypass CSRF protection entirely (OWASP A01/A02).

The fix in Beta-1 ([SEC-01]) is to abandon server-side storage altogether and switch to a **stateless signed double-submit cookie** scheme. There is no cache, no shared key, and no cross-user contamination by construction — because there is nothing to contaminate.

## How the new tokens work

A token is now a self-validating opaque value. The wire format, before base64url encoding, is:

```
[ nonce: 16 bytes ] [ expiresAt: 8 bytes big-endian uint64 ] [ hmac: 32 bytes ]
```

where

```
hmac = HMAC-SHA256(nonce || expiresAt || id || sessionId, secret)
```

Two pieces of context are folded into the HMAC payload:

- the logical `id` (e.g. `form:login`) — prevents **cross-form** replay; a token issued for `form:login` cannot validate against `form:account-deletion`;
- the `sessionId` — the value of the per-browser `WAFFLE_SID` cookie published as the `_anon_sid` request attribute by `AnonymousSessionMiddleware` — prevents **cross-browser** replay; a token minted under one browser cannot validate from another.

Validation, in `Waffle\Commons\Security\Csrf\CsrfTokenManager::validate()`:

1. base64url-decode and length-check (`56 bytes` exactly);
2. reject if the embedded `expiresAt` is in the past (zero means non-expiring);
3. recompute the HMAC over `(nonce || expiresAt || id || sessionId)` with the server secret;
4. compare with the candidate HMAC via `hash_equals()` — constant time.

No database, no Redis, no PSR-16 lookup.

## The anonymous SID (`WAFFLE_SID`)

The framework deliberately has no session abstraction — `$_SESSION` is forbidden by the FrankenPHP rules. To still get per-browser binding without inventing a full session backend, Beta-1 ships `AnonymousSessionMiddleware`:

- on first request: mints a 32-byte random identifier, base64url-encodes it, and sets `WAFFLE_SID=<value>; Path=/; Max-Age=2592000; HttpOnly; SameSite=Lax; Secure` (the `Secure` flag is dropped on plain-HTTP requests so dev still works);
- on subsequent requests: reuses the cookie value, validating its shape (43-char base64url alphabet);
- always publishes the value as the `_anon_sid` request attribute so downstream middleware (chiefly `CsrfMiddleware`) can fold it into HMAC operations.

The SID is **opaque and anonymous**. It identifies a browser, not a user. It does not survive cookie clearing (intentionally). It carries no PII and no claims.

## FrankenPHP alignment

Resident-worker mode does not tolerate hidden per-request state. The Beta-0 manager held no instance state itself, but the `CacheInterface` it depended on did — and any divergence between worker memory (`ArrayCache`) and shared storage (`RedisCache`) could surface as intermittent CSRF failures. The Beta-1 manager has **no I/O**: every `issue()` and `validate()` call is a pure function of its arguments and the injected secret.

## What `revoke()` and `hasValid()` do now

With no inventory, neither operation can offer meaningful semantics:

- `revoke($id)` is a no-op. Individual tokens cannot be invalidated without rotating the signing secret (which would invalidate every active token).
- `hasValid($id)` returns `false`. Callers that previously used it to avoid double-issuance must just call `issue()`.

These methods stay on `CsrfTokenManagerInterface` for API symmetry with future cache-backed managers, but the stateless variant treats them as documentation.

## Operational requirements

1. **Provide a secret of at least 32 bytes.** `MIN_SECRET_BYTES` is enforced at construction; shorter secrets raise `InvalidArgumentException`. Config key `waffle.security.csrf.secret` (falling back to env `WAFFLE_CSRF_SECRET`). `AppKernelFactory::resolveCsrfSecret()` refuses to boot in `prod` when the secret is missing or short; in non-prod environments it falls back to an ephemeral per-process value so dev/test still boot.
2. **Place `AnonymousSessionMiddleware` before `CsrfMiddleware` in the pipeline.** Without the `_anon_sid` attribute, every CSRF validation fails-closed — by design. The kernel factories wire this for you in the canonical order; custom pipelines must respect it.
3. **Rotate the secret on suspected compromise.** Rotation invalidates every issued token simultaneously — there is no graceful per-token revocation.
4. **Pair issuance with cookie + header on the client.** The middleware contract (`Waffle\Commons\Security\Middleware\CsrfMiddleware`) extracts the candidate from header → form field → cookie. Real defense requires the cookie *and* a non-cookie source (header or form field) to both carry the same signed token — application code is responsible for setting both.

## Trade-offs we accepted

- **No per-token revocation.** This is intrinsic to stateless signing; rotation is the only lever. We accept this because the attack surface that revocation guards against (single-token theft) is already mitigated by the short default TTL (`Constant::DEFAULT_TTL` = 3600s).
- **The token carries its expiry on the wire.** This is fine: tampering with the timestamp invalidates the HMAC.
- **Cross-worker sharing is free.** Every worker derives the same HMAC from the same secret, so there is no cache-warming or replication concern.
- **The SID is recoverable by clearing cookies.** A user who clears `WAFFLE_SID` gets a fresh one — existing tokens for the previous SID become invalid for the new browser session. This is the intended behaviour: it costs the user one extra round-trip to re-issue tokens; it costs an attacker the ability to lift a token from a cleared browser.

## Related documents

- [Reference: `#[PublicAccess]` attribute](../reference/attributes-public-access.md)
- [Explanation: Fail-Closed ABAC](./security-fail-closed-abac.md)
