# How to Register and Verify Passkeys (WebAuthn)

The `waffle-commons/auth` component ships a WebAuthn / passkey scheme (AUTH-01, RFC-021): your users enrol a FaceID / TouchID / Windows Hello passkey or a YubiKey, then log in with it. The cryptography lives behind one audited adapter (`WebAuthnLibAdapter`, the sole importer of `web-auth/webauthn-lib`); your application supplies just **two stores** — one for pending challenges, one for enrolled credentials — and the framework stays stateless across FrankenPHP worker requests.

This recipe walks the two ceremonies end to end. For the full API, see the [WebAuthn reference](../reference/webauthn.md); for the design rationale, [WebAuthn & Passkeys](../explanation/webauthn-passkeys.md).

## 1. Build the verifier (the relying-party config)

The adapter is configured once with your relying party and reused read-only across requests. Wire it in your container / `bin` bootstrap:

```php
use Waffle\Commons\Auth\WebAuthn\WebAuthnLibAdapter;
use Waffle\Commons\Contracts\Auth\WebAuthn\WebAuthnVerifierInterface;

$verifier = new WebAuthnLibAdapter(
    relyingPartyId:   'example.com',              // the effective domain (no scheme, no port)
    relyingPartyName: 'Example App',              // shown by the authenticator UI
    allowedOrigins:   ['https://example.com'],    // exact client origins you accept
    userVerification: 'preferred',                // 'preferred' (default) | 'required'
);
$container->set(WebAuthnVerifierInterface::class, $verifier);
```

Set `userVerification: 'required'` for **passwordless** logins — the library then rejects any response whose authenticator did not verify the user (biometric / PIN). The value is validated at construction: an unsupported string throws `WebAuthnException` immediately (fail-closed), so a misconfigured relying party never boots into a weakened state. `'discouraged'` is intentionally not accepted.

> The relying-party id and origins are security-critical: they bind every passkey to your domain. A mismatch between `relyingPartyId`, `allowedOrigins`, and the URL the browser actually loads makes every ceremony fail.

## 2. Implement the two app-provided stores

The framework owns the crypto; **you** own the state. Implement two interfaces against your storage (database, Redis, PSR-16 cache, …):

```php
use Waffle\Commons\Contracts\Auth\WebAuthn\CredentialRepositoryInterface;
use Waffle\Commons\Contracts\Auth\WebAuthn\RegisteredCredentialInterface;
use Waffle\Commons\Auth\WebAuthn\WebAuthnChallengeStoreInterface;
use Waffle\Commons\Contracts\Auth\WebAuthn\AssertionOptionsInterface;

// Enrolled passkeys — the ONLY persistent state of the surface.
final class PdoCredentialRepository implements CredentialRepositoryInterface
{
    public function findByCredentialId(string $credentialId): ?RegisteredCredentialInterface { /* … */ }
    public function findByUserHandle(string $userHandle): array { /* list<RegisteredCredentialInterface> */ }
    public function save(RegisteredCredentialInterface $credential): void { /* … */ }
    public function updateSignCount(string $credentialId, int $signCount): void { /* … */ }
}

// Pending login options — single-use, never in worker memory.
final class CacheChallengeStore implements WebAuthnChallengeStoreInterface
{
    public function take(string $ceremonyId): ?AssertionOptionsInterface { /* fetch + delete (single-use) */ }
}
```

Persist credential fields exactly as received — `credentialId()`, `publicKey()`, `userHandle()`, `signCount()`, and `transports()` are all base64url-safe strings / scalars, so a `RegisteredCredential` round-trips cleanly through a database or cache.

## 3. Register a passkey (attestation ceremony)

Use `WebAuthnCeremony` to bookend enrolment. It excludes the user's already-enrolled passkeys automatically.

```php
use Waffle\Commons\Auth\WebAuthn\WebAuthnCeremony;
use Waffle\Commons\Auth\WebAuthn\WebAuthnUser;

$ceremony = new WebAuthnCeremony($verifier, $credentials);

// --- Route A: GET /webauthn/register/options ---
$user = new WebAuthnUser(
    id:          $opaqueUserHandle,   // stable, opaque, NON-PII surrogate (never an email)
    name:        'ada@example.com',   // machine-facing account name
    displayName: 'Ada Lovelace',      // shown by the authenticator UI
);
$options = $ceremony->startRegistration($user);

// Persist the challenge against a ceremony id, then hand the JSON to the browser.
$myChallengeStore->put($ceremonyId, $options);     // your storage
return $this->jsonResponse(data: ['ceremonyId' => $ceremonyId, 'options' => $options->toArray()]);
// Browser: navigator.credentials.create({ publicKey: options }) → returns an attestation response
```

```php
// --- Route B: POST /webauthn/register/verify ---
$options = $myChallengeStore->take($ceremonyId);   // the SAME options object you issued
$clientResponseJson = (string) $request->getBody(); // the browser's attestation JSON

$credential = $ceremony->finishRegistration($options, $clientResponseJson);
// finishRegistration() verifies the attestation AND save()s the credential for you.
// On failure it throws InvalidWebAuthnRegistrationException (HTTP 400) and persists nothing.
```

You must replay the **exact options object** you issued — it carries the challenge the verifier binds the attestation to. The adapter advertises ES256 and RS256 and requests `none` attestation, so platform authenticators (FaceID/TouchID/Windows Hello) and roaming keys (YubiKeys) all enrol without an attestation-metadata service.

## 4. Log in with a passkey (assertion ceremony)

Issue login options, then let the inbound authenticator verify the assertion.

```php
// --- Route C: GET /webauthn/login/options ---
$options = $ceremony->startAuthentication($userHandle); // scoped to the user's passkeys; empty handle ⇒ discoverable login
$myChallengeStore->put($ceremonyId, $options);
return $this->jsonResponse(data: ['ceremonyId' => $ceremonyId, 'options' => $options->toArray()]);
// Browser: navigator.credentials.get({ publicKey: options }) → returns an assertion response
```

The browser then POSTs the assertion response **with the `X-Wfl-Webauthn-Ceremony` header** naming the pending ceremony. Register `WebAuthnAuthenticator` in the bridge and the verification is automatic:

```php
use Waffle\Commons\Auth\AuthenticationBridge;
use Waffle\Commons\Auth\Middleware\AuthenticationMiddleware;
use Waffle\Commons\Auth\WebAuthn\WebAuthnAuthenticator;

$bridge = new AuthenticationBridge($context, [
    new WebAuthnAuthenticator($verifier, $challengeStore, $credentials), // X-Wfl-Webauthn-Ceremony
    // … your other schemes (JWT, assertions, API key) — first match wins
]);
$stack->add(middleware: new AuthenticationMiddleware($bridge));
```

`WebAuthnAuthenticator::supports()` matches on the `X-Wfl-Webauthn-Ceremony` header (`WebAuthnAuthenticator::CEREMONY_HEADER`). On a valid assertion it advances the stored signature counter (clone detection) and produces a `UserIdentity` keyed on the credential's user handle — readable downstream exactly like any other authenticated identity (see [How to Authenticate Requests](authentication.md) §3).

## 5. What fail-closed looks like

| Situation | Result |
|---|---|
| Unsupported `userVerification` value | `WebAuthnException` at **construction** (the adapter never builds) |
| Bad challenge / origin / signature, or unsupported attestation format on registration | `InvalidWebAuthnRegistrationException` → **400**; credential never enrolled |
| Bad challenge / origin / signature, unknown credential, or non-monotonic counter on login | `InvalidWebAuthnAssertionException` → **401** (also an `AuthenticationExceptionInterface`) |
| Malformed assertion body / missing credential id | `InvalidWebAuthnAssertionException` → **401** |
| Corrupt stored base64url credential field | `WebAuthnException` |

Every one of these is an `AuthExceptionInterface`, so a single `catch (AuthExceptionInterface)` covers passkey failures alongside JWT, HMAC, and OAuth — and the error-handler renders the status carried in the exception code. There is no fallback to the next scheme and never a silent anonymous downgrade.

## See also

- [Explanation: WebAuthn & Passkeys](../explanation/webauthn-passkeys.md)
- [Reference: WebAuthn / passkeys](../reference/webauthn.md)
- [How to Authenticate Requests with the Universal Authentication Bridge](authentication.md) — the inbound bridge WebAuthn plugs into
- [How to Secure Your Application](security.md) — authorization (ABAC voters) after authentication

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
