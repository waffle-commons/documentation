# WebAuthn / Passkeys Reference (`waffle-commons/auth`)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *Adds the WebAuthn / passkey surface (AUTH-01 / AXE6) under RFC-021.*
> **Requires:** PHP 8.5+, `ext-openssl`, `web-auth/webauthn-lib ^5.3`, `symfony/serializer ^8.1`, `symfony/uid ^8.1`. Depends only on `waffle-commons/contracts`.

The WebAuthn surface is the passkey scheme of the Universal Authentication Bridge. The cryptographic core sits behind a single contract (`Waffle\Commons\Contracts\Auth\WebAuthn\WebAuthnVerifierInterface`) and one concrete adapter (`Waffle\Commons\Auth\WebAuthn\WebAuthnLibAdapter`) — the **only** class that imports `web-auth/webauthn-lib`. Everything else (the ceremony service, the inbound authenticator, the option/credential DTOs) speaks the contract. The component is stateless across FrankenPHP worker requests; the only stateful pieces — the challenge store and the credential repository — are interfaces the integrating app provides.

For the architectural reasoning (why one wrapper, the stateless split, configurable UV, fail-closed), see [Explanation: WebAuthn & Passkeys](../explanation/webauthn-passkeys.md).

## Contracts (`Waffle\Commons\Contracts\Auth\WebAuthn`)

| Contract | Surface |
|---|---|
| `WebAuthnVerifierInterface` | `createRegistrationOptions(WebAuthnUserInterface $user, list<RegisteredCredentialInterface> $existing = []): RegistrationOptionsInterface` · `verifyRegistration(RegistrationOptionsInterface $options, string $clientResponseJson): RegisteredCredentialInterface` · `createAssertionOptions(list<RegisteredCredentialInterface> $allowed = []): AssertionOptionsInterface` · `verifyAssertion(AssertionOptionsInterface $options, string $clientResponseJson, RegisteredCredentialInterface $credential): int` |
| `RegistrationOptionsInterface` | `challenge(): string` (base64url) · `toArray(): array<string, mixed>` · `toJson(): string` |
| `AssertionOptionsInterface` | `challenge(): string` (base64url) · `toArray(): array<string, mixed>` · `toJson(): string` |
| `RegisteredCredentialInterface` | `credentialId(): string` · `publicKey(): string` (COSE, base64url) · `userHandle(): string` · `signCount(): int` · `transports(): list<string>` |
| `WebAuthnUserInterface` | `id(): string` (opaque, stable, non-PII handle) · `name(): string` · `displayName(): string` |
| `CredentialRepositoryInterface` | `findByCredentialId(string $credentialId): ?RegisteredCredentialInterface` · `findByUserHandle(string $userHandle): list<RegisteredCredentialInterface>` · `save(RegisteredCredentialInterface $credential): void` · `updateSignCount(string $credentialId, int $signCount): void` |

`RegistrationOptionsInterface` and `AssertionOptionsInterface` map to the WebAuthn `PublicKeyCredentialCreationOptions` / `PublicKeyCredentialRequestOptions`; `RegisteredCredentialInterface` maps to a `PublicKeyCredentialSource`; `WebAuthnUserInterface` maps to a `PublicKeyCredentialUserEntity`. `CredentialRepositoryInterface` is the persistence boundary — **the only stateful part of the contract surface**, implemented by the app and injected into the ceremony and authenticator; it lives in application storage (database, Redis, …), never in worker memory.

## Cryptographic core — `Waffle\Commons\Auth\WebAuthn\WebAuthnLibAdapter`

`final` implementation of `WebAuthnVerifierInterface`, and the sole importer of `web-auth/webauthn-lib`.

```php
public function __construct(
    string $relyingPartyId,                 // effective domain, e.g. 'example.com'
    string $relyingPartyName,               // human-facing RP name (authenticator UI)
    array $allowedOrigins,                  // list<string>, e.g. ['https://example.com']
    string $userVerification = 'preferred', // 'preferred' (default) | 'required'
);

public function createRegistrationOptions(WebAuthnUserInterface $user, array $existing = []): RegistrationOptionsInterface;
public function verifyRegistration(RegistrationOptionsInterface $options, string $clientResponseJson): RegisteredCredentialInterface;
public function createAssertionOptions(array $allowed = []): AssertionOptionsInterface;
public function verifyAssertion(AssertionOptionsInterface $options, string $clientResponseJson, RegisteredCredentialInterface $credential): int;
```

- **Stateless construction.** The library serializer and both ceremony validators (`AuthenticatorAttestationResponseValidator`, `AuthenticatorAssertionResponseValidator`) are built once in the constructor from the relying-party config and reused read-only across requests (FrankenPHP worker rule). The component passes the `igor-php` audit with zero state findings.
- **Algorithms.** Registration advertises **ES256** (COSE `-7`, ECDSA P-256/SHA-256 — mandatory-to-implement) and **RS256** (COSE `-257`).
- **Attestation.** Conveyance is requested as `none`; the support manager registers `NoneAttestationStatementSupport`. The relying party does not collect device-identifying attestation.
- **User verification.** Accepts only `'preferred'` or `'required'` (the `AuthenticatorSelectionCriteria` constants). The value is **validated at construction** — an unsupported value throws `WebAuthnException` (fail-closed, never a silent widening of UV). The requirement is carried on **both** ceremonies: the registration `authenticatorSelection` and the assertion options, so the library's `CheckUserVerification` step can reject a UV-less response at enrolment *or* login.
- **Exclude / allow lists.** `createRegistrationOptions()` builds `excludeCredentials` from `$existing` (prevents double-registration on one authenticator); `createAssertionOptions()` builds `allowCredentials` from `$allowed` (an empty list enables a usernameless/discoverable login).
- **Counter return.** `verifyAssertion()` returns the new signature counter (`int`) the caller persists via `CredentialRepositoryInterface::updateSignCount()`; a non-monotonic counter is rejected by the library as a cloned authenticator.
- **base64url discipline.** Every binary field is carried as base64url (via [`Base64Url`](#wafflecommonsauthcodecbase64url)); a stored field that does not strict-decode raises `WebAuthnException`.

`createRegistrationOptions()` / `createAssertionOptions()` wrap any server-side build/serialize failure as `WebAuthnException`. `verifyRegistration()` wraps any verification failure as `InvalidWebAuthnRegistrationException` (the credential is never enrolled); `verifyAssertion()` wraps any failure as `InvalidWebAuthnAssertionException`.

## Registration ceremony — `Waffle\Commons\Auth\WebAuthn\WebAuthnCeremony`

`final readonly` service that bookends the **enrolment** ceremony, separate from the inbound authenticator.

```php
public function __construct(
    WebAuthnVerifierInterface $verifier,
    CredentialRepositoryInterface $credentials,
);

public function startRegistration(WebAuthnUserInterface $user): RegistrationOptionsInterface;          // excludes the user's already-enrolled credentials
public function finishRegistration(RegistrationOptionsInterface $options, string $clientResponseJson): RegisteredCredentialInterface; // verifies, then save()s
public function startAuthentication(string $userHandle): AssertionOptionsInterface;                    // login options scoped to the user's passkeys (empty handle ⇒ discoverable login)
```

`startRegistration()` looks up the user's enrolled passkeys (`findByUserHandle()`) and passes them as the exclude list. `finishRegistration()` verifies the attestation response and, on success, persists the credential through the injected repository — the verifier never stores anything. `finishRegistration()` throws `InvalidWebAuthnRegistrationExceptionInterface` on a failed attestation (nothing is persisted).

## Inbound authenticator — `Waffle\Commons\Auth\WebAuthn\WebAuthnAuthenticator`

`final readonly` implementation of `Waffle\Commons\Contracts\Auth\AuthenticatorInterface` — the WebAuthn **login** scheme of the bridge, mirroring `JwtAuthenticator`.

```php
public const string CEREMONY_HEADER = 'X-Wfl-Webauthn-Ceremony';

public function __construct(
    WebAuthnVerifierInterface $verifier,
    WebAuthnChallengeStoreInterface $challenges,
    CredentialRepositoryInterface $credentials,
);

public function supports(ServerRequestInterface $request): bool;                  // presence of CEREMONY_HEADER
public function authenticate(ServerRequestInterface $request): UserIdentityInterface;
```

- `supports()` is a pure presence check on `X-Wfl-Webauthn-Ceremony` (the header naming the pending login ceremony). This is what lets the bridge's *first-match-wins* chain route a passkey login to this scheme.
- `authenticate()` reads the ceremony id from the header, takes the issued options from the challenge store, narrows the PSR-7 body's `id` field with `is_string()` (the untyped wire boundary), looks up the stored credential, delegates the crypto to `verifyAssertion()`, advances the stored counter via `updateSignCount()`, and returns a `Waffle\Commons\Auth\Identity\UserIdentity` keyed on the credential's `userHandle()`.
- **Fail-closed:** a missing/unknown ceremony id, a missing/unknown credential, malformed JSON, a missing credential id, or any verification failure throws `InvalidWebAuthnAssertionException` (HTTP 401) — which is an `AuthenticationExceptionInterface`, so the bridge renders it as a fail-closed 401 with no scheme fallback.

## Challenge store — `Waffle\Commons\Auth\WebAuthn\WebAuthnChallengeStoreInterface`

The stateful, **app-provided** half of the login surface — single-use, never in worker memory.

```php
public function take(string $ceremonyId): ?AssertionOptionsInterface; // single-use; null when unknown/consumed/expired
```

The challenge a login mints is persisted here against an opaque ceremony id (a cache, session, database, …) and replayed when the browser returns its assertion. `take()` consumes it — a second call for the same id MUST return `null`.

## Option DTOs

Both implement their contract and validate the challenge with PHP 8.5 property hooks; `public private(set)` provides write-once immutability (hooked properties cannot be `readonly`).

### `Waffle\Commons\Auth\WebAuthn\RegistrationOptions`

```php
public private(set) string $challenge; // base64url, non-empty (hook-validated)
public private(set) string $json;      // library-serialized creation options, non-empty (hook-validated)

public function __construct(string $challenge, string $json);
public function challenge(): string;
public function toArray(): array;       // array<string, mixed> via OptionsCodec; throws WebAuthnException on corrupt JSON
public function toJson(): string;       // the JSON for navigator.credentials.create()
```

### `Waffle\Commons\Auth\WebAuthn\AssertionOptions`

Identical shape; `toJson()` returns the JSON for `navigator.credentials.get()` and `verifyAssertion()` replays it. An empty challenge or empty JSON throws `\InvalidArgumentException` at construction.

## Value objects

### `Waffle\Commons\Auth\WebAuthn\RegisteredCredential`

`final readonly` implementation of `RegisteredCredentialInterface`. Maps to the library's `CredentialRecord`; every binary field is a base64url string so it is JSON/storage friendly.

```php
/** @param list<string> $transports */
public function __construct(
    string $credentialId,   // base64url
    string $publicKey,      // COSE public key, base64url
    string $userHandle,     // the app's opaque, stable handle (== WebAuthnUserInterface::id())
    int $signCount,
    array $transports = [], // e.g. ['internal', 'usb', 'nfc']
);
```

### `Waffle\Commons\Auth\WebAuthn\WebAuthnUser`

`final readonly` implementation of `WebAuthnUserInterface`.

```php
public function __construct(string $id, string $name, string $displayName);
```

`$id` MUST be a stable, opaque, non-PII surrogate (WebAuthn §4) — never an email. The library caps the handle at 64 bytes; an over-long handle is rejected by the verifier when the options are built.

## Codec & internals

### `Waffle\Commons\Auth\WebAuthn\OptionsCodec`

Typed boundary around the library-serialized options JSON: the single place that narrows the untyped `json_decode()` result back to an `array<string, mixed>` for the option DTOs' `toArray()`.

```php
public static function decode(string $json): array; // array<string, mixed>; throws WebAuthnException when not a decodable object
```

### `Waffle\Commons\Auth\Codec\Base64Url`

Shared with the bridge (assertion, JWT, OAuth wire formats) — `final readonly`, pure functions, unpadded base64url (RFC 4648 §5).

```php
public static function encode(string $binary): string;
public static function decode(string $value): ?string; // null on malformed input (strict mode)
```

## Exceptions

All sit under the bridge's exception tree, so a `catch (AuthExceptionInterface)` covers passkey failures alongside JWT, HMAC, and OAuth. The HTTP status travels as the exception code.

| Concrete (`Waffle\Commons\Auth\WebAuthn\Exception`) | Contract (`Waffle\Commons\Contracts\Auth\WebAuthn\Exception`) | When |
|---|---|---|
| `WebAuthnException` (`class`, extends `AuthException`) | `WebAuthnExceptionInterface` (extends `AuthExceptionInterface`) | Base passkey failure — unsupported UV value at construction, a server-side build/serialize failure, corrupt stored base64url, or a non-decodable options JSON. |
| `InvalidWebAuthnRegistrationException` (`final`, HTTP **400**) | `InvalidWebAuthnRegistrationExceptionInterface` (extends `WebAuthnExceptionInterface`) | Attestation (registration) verification failed: wrong challenge/origin, bad signature, or unsupported attestation format. The credential is never enrolled. |
| `InvalidWebAuthnAssertionException` (`final`, HTTP **401**) | `InvalidWebAuthnAssertionExceptionInterface` (extends `WebAuthnExceptionInterface` **and** `AuthenticationExceptionInterface`) | Assertion (login) verification failed: wrong challenge/origin, bad signature, unknown credential, or a non-monotonic signature counter (cloned authenticator). Rendered as a fail-closed 401 by the bridge. |

## Worker-safety contract

The adapter, ceremony service, inbound authenticator, option DTOs, value objects, and codecs are stateless / immutable — the WebAuthn surface adds zero per-request worker state. The only stateful collaborators are the app-provided `CredentialRepositoryInterface` and `WebAuthnChallengeStoreInterface`, which live in application storage, never in the worker. The component passes the `igor-php` worker-mode audit with zero findings.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
