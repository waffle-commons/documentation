# WebAuthn & Passkeys (AUTH-01)

> **Status:** shipped in `waffle-commons/auth` (+ contracts) in the `0.1.0-beta5` cycle, as the AUTH-01 / AXE6 deliverable of RFC-021.
> **Companion pages:** [auth/webauthn reference](../reference/webauthn.md) · [How to: Register and Verify Passkeys](../how-to/register-and-verify-passkeys.md) · [The Universal Authentication Bridge](authentication-universal-bridge.md).

This page explains *why* the passkey surface looks the way it does. For exact signatures, use the reference.

## Why passkeys, and why one wrapper

A passkey is a public-key credential bound to an origin and held by an authenticator — a platform one (FaceID, TouchID, Windows Hello) or a roaming one (a YubiKey). The browser does the key handling through the WebAuthn API (`navigator.credentials.create()` / `.get()`); the server's job is to issue a one-time challenge, then cryptographically verify the authenticator's response. That verification is not something to hand-roll: it means parsing CBOR, decoding COSE public keys, walking attestation formats, and checking authenticator-data flags. A single mistake there is a silent authentication bypass.

So the framework does not implement any of it. The entire passkey surface sits behind **one contract** — `Waffle\Commons\Contracts\Auth\WebAuthn\WebAuthnVerifierInterface` — and a **single concrete adapter**, `Waffle\Commons\Auth\WebAuthn\WebAuthnLibAdapter`, is the *only* class in the monorepo that imports `web-auth/webauthn-lib` (`Webauthn\**`). Every other WebAuthn type — the ceremony service, the inbound authenticator, the option DTOs, the credential value object — speaks the contract, never the library. The audited library owns the cryptography; the framework owns the boundary, the typing, and the worker-safety.

This is the same posture the [JWT and OAuth subsystems](authentication-universal-bridge.md) take: *implement the rules that matter strictly, delegate the heavy crypto to one audited place, and never let a vendor type leak into the rest of the codebase.*

## The two ceremonies, and the stateless split

WebAuthn has exactly two ceremonies, and the component maps each onto a method pair on the verifier:

- **Registration (attestation)** — enrol a new passkey. `createRegistrationOptions()` mints a `PublicKeyCredentialCreationOptions` (ES256 + RS256 parameters, the relying party, the user entity, the configured user-verification requirement, and an `excludeCredentials` list of already-enrolled passkeys so a user cannot double-register on one authenticator). `verifyRegistration()` checks the browser's attestation response against those options and returns the credential to persist.
- **Authentication (assertion)** — log in with an enrolled passkey. `createAssertionOptions()` mints a `PublicKeyCredentialRequestOptions` scoped to the user's `allowCredentials` (or empty for a usernameless/discoverable login). `verifyAssertion()` checks the response against the issued options *and* the stored credential, returning the new signature counter.

The defining constraint is FrankenPHP's resident worker: **nothing about a pending ceremony may live in worker memory between requests.** The challenge minted on request N must survive to request N+2 (the browser round-trip) without the worker remembering it. The component enforces this by making the adapter and both ceremony services **stateless**, and pushing the only two pieces of state *out of the contract entirely*:

1. **The challenge store.** The challenge a ceremony mints travels back inside the issued options (`RegistrationOptionsInterface::challenge()` / `AssertionOptionsInterface::challenge()`, base64url). The application persists it against an opaque ceremony id — in a cache, a session row, Redis, wherever — and replays the *same options object* to the verifier. The inbound `WebAuthnAuthenticator` reaches this through the app-provided `WebAuthnChallengeStoreInterface`, whose `take()` is single-use by contract.
2. **The credential store.** Enrolled passkeys live in the app's `CredentialRepositoryInterface` implementation — a database, Redis, anything. The contract's own docblock calls this out as *"the ONLY stateful part of the WebAuthn surface,"* and it lives in application storage, never in the worker.

The result is that the adapter builds its serializer and both ceremony validators **once at construction** from the relying-party configuration and reuses them read-only across every request. The component passes the `igor-php` worker-mode audit because there is no per-request state for a worker loop to leak.

## Configurable user verification, advertised on both options

User verification (UV) is the flag that says *the authenticator confirmed the human* (a biometric or a PIN), as opposed to merely *the device was present*. WebAuthn defines three levels; the framework deliberately exposes only two:

- `'preferred'` (the default) — the authenticator decides; a UV-capable device will perform it, a bare security key may not.
- `'required'` — used for passwordless logins. The library's `CheckUserVerification` ceremony step rejects any response whose authenticator data lacks the UV flag.

`'discouraged'` is **intentionally not offered**: the framework never weakens UV below the protocol default. The chosen requirement is carried on *both* ceremonies — it rides the registration `authenticatorSelection` as well as the assertion options — so a `'required'` deployment rejects a UV-less response at *either* enrolment or login, not just one of them. Advertising it on both is what lets the library do the rejection for you instead of you re-checking a flag by hand.

## Fail-closed, by construction

The passkey surface inherits the bridge's fail-closed posture and tightens it in three places:

- **Bad UV value is rejected at construction.** Untyped configuration can reach the constructor, so an unsupported user-verification string throws `WebAuthnException` *when the adapter is built* — never a silent widening of UV at request time. A misconfigured relying party cannot boot into a weakened state.
- **Every verification failure throws, the credential is never enrolled / the login never succeeds.** `verifyRegistration()` raises `InvalidWebAuthnRegistrationException` (HTTP 400) on a bad challenge, origin, signature, or an unsupported attestation format; the credential is simply never saved. `verifyAssertion()` raises `InvalidWebAuthnAssertionException` (HTTP 401) on a bad challenge, origin, signature, an unknown credential, or — critically — a **non-monotonic signature counter**, the standard signal that an authenticator has been cloned.
- **Corrupted stored fields fail closed.** Every binary field is carried as base64url; a stored value that does not strict-decode raises `WebAuthnException` rather than feeding garbage into the verifier.

Because `InvalidWebAuthnAssertionException` also implements `AuthenticationExceptionInterface`, a failed passkey login is treated by the [Universal Authentication Bridge](authentication-universal-bridge.md) exactly like any other rejected inbound credential: a fail-closed 401, no fallback to the next scheme, no silent anonymous downgrade.

## How it plugs into the Universal Authentication Bridge

WebAuthn is not a side door — it is one more **inbound scheme** under RFC-021. `WebAuthnAuthenticator` implements the same `AuthenticatorInterface` as the JWT, assertion, and API-key schemes, so it slots into the bridge's *first-match-wins* chain. Its `supports()` is a pure presence check: does the request carry the `X-Wfl-Webauthn-Ceremony` header naming a pending login? If so, `authenticate()` replays the issued options from the challenge store, looks up the asserted credential, delegates the crypto to the verifier, advances the stored signature counter, and maps the verified user handle into a `UserIdentity` — the same identity type every other scheme produces. Every WebAuthn exception sits under `AuthExceptionInterface`, so a single `catch (AuthExceptionInterface)` covers passkey failures alongside JWT, HMAC, and OAuth ones.

## What the library does, and what it deliberately does not

The adapter wires the library for **`none` attestation statements** (`NoneAttestationStatementSupport`) and the two passkey algorithms that matter: **ES256** (COSE `-7`, ECDSA P-256/SHA-256 — the mandatory-to-implement passkey algorithm) and **RS256** (COSE `-257`). Attestation conveyance is requested as `none` — the framework does not collect device-identifying attestation for a self-hosted relying party, which keeps registration private and avoids a metadata-service dependency. The user handle is treated as the app's opaque, stable surrogate (matching `WebAuthnUserInterface::id()`), carried verbatim so it stays equal to the value a repository keys on — never raw authenticator bytes, never PII (WebAuthn §4 caps it at 64 bytes, and the library rejects an over-long handle when the options are built).

In short: the framework supplies a typed, stateless, fail-closed shell; the audited library supplies the cryptography; and the application supplies the two stores. Each owns exactly the thing it is best placed to own.

## See also

- [How to: Register and Verify Passkeys](../how-to/register-and-verify-passkeys.md)
- [Reference: WebAuthn / passkeys](../reference/webauthn.md)
- [The Universal Authentication Bridge](authentication-universal-bridge.md) — the inbound-scheme model WebAuthn plugs into
- RFC-021 (monorepo `project_system/RFCs/`) — normative specification

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
