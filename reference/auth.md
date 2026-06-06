# Reference: Auth Component (`waffle-commons/auth`)

Universal Authentication Bridge — RFC-021. Contracts live in
`Waffle\Commons\Contracts\Auth`; implementations in `Waffle\Commons\Auth`.

## Contracts (`Waffle\Commons\Contracts\Auth`)

| Contract | Surface |
|---|---|
| `UserIdentityInterface` | `string $subject { get; }` · `?string $email { get; }` · `list<string> $roles { get; }` · `array<string, mixed> $claims { get; }` |
| `SecurityContextInterface` (extends `Service\ResettableInterface`) | `authenticate(UserIdentityInterface $identity, ?string $clientIp = null): void` · `isAuthenticated(): bool` · `getIdentity(): ?UserIdentityInterface` · `getClientIp(): ?string` · `reset(): void` |
| `AuthenticatorInterface` | `supports(ServerRequestInterface $request): bool` · `authenticate(ServerRequestInterface $request): UserIdentityInterface` |
| `AuthenticationBridgeInterface` | `authenticate(ServerRequestInterface $request): ?UserIdentityInterface` |
| `CredentialsProviderInterface` | `supports(RequestInterface $request): bool` · `apply(RequestInterface $request): RequestInterface` |
| `Assertion\UserAssertionInterface` | `$subject`, `$email`, `$roles`, `$tenant`, `$issuedAt`, `$expiresAt`, `$ipHash`, `$payload` (all `{ get; }`) |
| `Assertion\AssertionSignerInterface` | `sign(UserAssertionInterface $assertion): string` · `hashClientIp(string $clientIp): string` |
| `Assertion\AssertionVerifierInterface` | `verify(#[\SensitiveParameter] string $headerValue, string $expectedClientIp): UserAssertionInterface` |
| `Token\TokenValidatorInterface` | `validate(#[\SensitiveParameter] string $token, ?string $expectedNonce = null): UserIdentityInterface` |
| `Token\TokenSetInterface` | `$accessToken`, `$tokenType`, `$idToken`, `$refreshToken`, `$expiresIn`, `$scope`, `$issuedAt` · `isExpired(?int $now = null): bool` |
| `Token\KeyResolverInterface` | `resolve(string $algorithm, ?string $keyId = null): string` |
| `Oauth\ProviderMetadataInterface` | `$issuer`, `$authorizationEndpoint`, `$tokenEndpoint`, `$jwksUri`, `$userinfoEndpoint` |
| `Oauth\DiscoveryInterface` | `discover(string $issuer): ProviderMetadataInterface` |
| `Oauth\OauthClientInterface` | `createAuthorizationUrl(string $state, string $nonce, string $codeChallenge): string` · `exchangeCode(string $code, string $codeVerifier): TokenSetInterface` · `clientCredentials(?string $scope = null): TokenSetInterface` |

### `Constant` (`Waffle\Commons\Contracts\Auth\Constant`)

| Constant | Value |
|---|---|
| `ASSERTION_HEADER` | `X-Wfl-Assert-User` |
| `SECRET_ENV_KEY` | `WAFFLE_AUTH_SECRET` |
| `ASSERTION_TTL` | `5` (seconds) |
| `MIN_SECRET_BYTES` | `32` |
| `REQUEST_ATTRIBUTE` | `_auth_identity` |
| `API_KEY_HEADER` | `X-Api-Key` |
| `AUTHORIZATION_HEADER` / `BEARER_PREFIX` / `BASIC_PREFIX` | `Authorization` / `Bearer ` / `Basic ` |
| `CLAIM_SUBJECT` … `CLAIM_IP_HASH` | `usr`, `eml`, `rol`, `ten`, `iat`, `exp`, `iph` |
| `OAUTH_TRANSACTION_COOKIE` / `OAUTH_TRANSACTION_TTL` | `WAFFLE_OAUTH_TX` / `600` |

### Exception hierarchy (HTTP code as exception code)

`AuthExceptionInterface` (500) → `AuthenticationExceptionInterface` (401) →
`InvalidTokenExceptionInterface` (401) · `InvalidAssertionExceptionInterface` (403) →
`SignatureVerificationExceptionInterface` / `ExpiredAssertionExceptionInterface` /
`ClientIpHijackingExceptionInterface` (403). Plus `MissingAuthSecretExceptionInterface`
(500, fail-closed boot) and `OauthExceptionInterface` (502).

## Implementations (`Waffle\Commons\Auth`)

| Class | Notes |
|---|---|
| `SecurityContext` | Request-scoped, `ResettableInterface`; the component's only mutable service. Register it in the container so the kernel reset chain wipes it per worker loop. |
| `AuthenticationBridge` | `__construct(SecurityContextInterface $context, list<AuthenticatorInterface> $authenticators)` — first `supports()` wins, fail-closed. |
| `Middleware\AuthenticationMiddleware` | PSR-15 — `__construct(AuthenticationBridgeInterface $bridge, ?LoggerInterface $logger = null)`; publishes `_auth_identity`. |
| `Middleware\GatewayAssertionMiddleware` | PSR-15 downstream firewall — `__construct(AssertionVerifierInterface $verifier, SecurityContextInterface $context)`. |
| `Identity\UserIdentity` | Hooked + `private(set)` immutable identity; `UserIdentity::fromAssertion(UserAssertionInterface): self`. |
| `Identity\ClaimMapping` | `__construct(string $subject = 'sub', ?string $email = 'email', ?string $roles = 'roles', ?string $tenant = null)` · `identityFrom(array $claims): UserIdentity`. |
| `Uab\UserAssertion` | Hooked + `private(set)` VO of the seven claims; virtual `$payload` serializes canonical JSON; enforces `exp − iat ≤ 5`. |
| `Uab\AuthBridgeSigner` | `__construct(#[\SensitiveParameter] string $secret)` (≥ 32 bytes or `MissingAuthSecretException`) · `sign()` · `hashClientIp()`. |
| `Uab\AuthBridgeVerifier` | Same fail-closed constructor · `verify()` — `hash_equals` MAC, temporal window, keyed IP binding. |
| `Authenticator\JwtAuthenticator` | `Authorization: Bearer` → `TokenValidatorInterface`. |
| `Authenticator\AssertionAuthenticator` | `X-Wfl-Assert-User` → verifier, IP from `REMOTE_ADDR`. |
| `Authenticator\ApiKeyAuthenticator` | `__construct(array<string, UserIdentityInterface> $identitiesByKey, string $headerName = 'X-Api-Key')` — `hash_equals` per candidate. |
| `Authenticator\BasicAuthenticator` | `__construct(array<string, string> $users, list<string> $roles = [])` — `password_verify()` for hashes, `hash_equals()` for opaque tokens. |
| `Jwt\JwtValidator` | `__construct(JwtConfig $config, KeyResolverInterface $keys, JwtParser $parser = new JwtParser())` — allow-list, no `none`, no HS/RS confusion, `exp`/`nbf`/`iat` + leeway, `iss`/`aud`, OIDC `nonce`. |
| `Jwt\JwtConfig` | `__construct(list<string> $algorithms, string $issuer, string $audience, int $leeway = 0, ClaimMapping $mapping = new ClaimMapping())` — supported: `HS256`, `RS256`. |
| `Jwt\Key\StaticKeyResolver` | `__construct(array<string, string> $keysByAlgorithm)`. |
| `Jwt\Key\JwksKeyResolver` | `__construct(string $jwksUri, ClientInterface $http, RequestFactoryInterface $requests, CacheInterface $cache, int $cacheTtl = 3600)` — RS256 only, `kid` selection, JWK→PEM via `JwkConverter`. |
| `Oauth\OauthClient` | `__construct(OauthConfig $config, ClientInterface $http, RequestFactoryInterface $requests, StreamFactoryInterface $streams)` — auth-code + PKCE S256, client-credentials. |
| `Oauth\OidcDiscovery` | `.well-known/openid-configuration` → `ProviderMetadata`, PSR-16 cached, issuer pinning (RFC 8414 §3.3). |
| `Oauth\Preset\ProviderPreset` | `Google` · `Microsoft` · `GitHub` — `metadata()`, `claimMapping()`, `defaultScopes()`. |
| `Oauth\Pkce` | `generateVerifier(): string` · `challenge(string $verifier): string` (S256 only). |
| `Oauth\Transaction\OauthTransaction` | `start(string $returnTo = '/'): self` — high-entropy `state`/`nonce`/verifier. |
| `Oauth\Transaction\OauthTransactionCodec` | Signed short-TTL cookie codec — `encode()`, `decode()` (403 on tamper/expiry). |
| `Client\AuthenticatedClient` | PSR-18 decorator — `__construct(ClientInterface $inner, list<CredentialsProviderInterface> $providers)`. |
| `Credential\AssertionCredentialsProvider` | `__construct(AssertionSignerInterface $signer, SecurityContextInterface $context, list<string> $allowedHosts, ?string $tenant = null)` — no identity/IP ⇒ no header. |
| `Credential\BearerCredentialsProvider` / `ApiKeyCredentialsProvider` / `BasicCredentialsProvider` | Static credentials, host-gated, never overwrite. |
| `Credential\ClientCredentialsProvider` | `__construct(OauthClientInterface $oauth, CacheInterface $cache, list<string> $allowedHosts, ?string $scope = null, string $cacheKey = 'waffle.auth.client_credentials')` — token cached until 30 s before expiry. |

## Quality gates

`composer mago` (zero baselines) · `composer tests` (190 tests, 100% statement coverage) ·
`composer igor` (Worker-Mode compatible — zero state errors).
