# How to Authenticate Requests with the Universal Authentication Bridge

The `waffle-commons/auth` component (RFC-021) is Waffle's entire **authentication** layer:
it establishes *who* the caller is. Authorization (*may you do this?*) stays in
`waffle-commons/security` ([Fail-Closed ABAC](../explanation/security-fail-closed-abac.md)).

The bridge is **protocol-agnostic** and works in both directions:

| Direction | You get | Schemes |
|---|---|---|
| Inbound | `AuthenticationMiddleware` + interchangeable authenticators | OAuth2/OIDC, JWT Bearer, gateway assertions, API key, HTTP Basic |
| Outbound | `AuthenticatedClient` (PSR-18 decorator) + credential providers | signed assertions, Bearer tokens, client-credentials, API key, Basic |

## 1. Provide the shared secret (fail-closed)

Every secret-requiring scheme refuses to boot without a ≥ 32-byte secret
(`MissingAuthSecretException` — RFC-021 §4.2):

```yaml
# config/app.yaml
waffle:
  auth:
    secret: '%env(WAFFLE_AUTH_SECRET)%'
```

## 2. Wire the inbound bridge

```php
use Waffle\Commons\Auth\AuthenticationBridge;
use Waffle\Commons\Auth\Authenticator\{ApiKeyAuthenticator, AssertionAuthenticator, JwtAuthenticator};
use Waffle\Commons\Auth\Jwt\{JwtConfig, JwtValidator};
use Waffle\Commons\Auth\Jwt\Key\StaticKeyResolver;
use Waffle\Commons\Auth\Middleware\AuthenticationMiddleware;
use Waffle\Commons\Auth\SecurityContext;
use Waffle\Commons\Auth\Uab\{AuthBridgeSigner, AuthBridgeVerifier};

// Request-scoped identity holder — register it in the container so the
// kernel wipes it between FrankenPHP worker loops (ResettableInterface).
$context = new SecurityContext();
$container->set(SecurityContextInterface::class, $context);

$bridge = new AuthenticationBridge($context, [
    new AssertionAuthenticator(new AuthBridgeVerifier($authSecret)),   // X-Wfl-Assert-User
    new JwtAuthenticator(new JwtValidator(                              // Authorization: Bearer
        config: new JwtConfig(algorithms: ['HS256'], issuer: $iss, audience: $aud),
        keys: new StaticKeyResolver(['HS256' => $authSecret]),
    )),
    new ApiKeyAuthenticator([$apiKey => new UserIdentity(subject: 'svc-demo')]), // X-Api-Key
]);

$stack->add(middleware: new AuthenticationMiddleware($bridge));
// Canonical order: ErrorHandler → TrustedHost → AnonymousSession →
// Authentication → Routing → Csrf → Security → SecureHeaders → Dispatcher.
```

Rules (RFC-021 §3.2): the **first** authenticator whose `supports()` matches wins; invalid
credentials **throw** (401/403, rendered by the error-handler — no fallback, no silent
anonymous downgrade); no credentials at all ⇒ anonymous pass-through (ABAC still decides
access).

## 3. Read the verified identity

```php
use Waffle\Commons\Contracts\Auth\Constant as AuthConstant;
use Waffle\Commons\Contracts\Auth\UserIdentityInterface;

public function me(ServerRequestInterface $request): ResponseInterface
{
    $identity = $request->getAttribute(AuthConstant::REQUEST_ATTRIBUTE); // '_auth_identity'
    if (!$identity instanceof UserIdentityInterface) {
        throw new AuthenticationException('Authentication required.');   // 401
    }

    return $this->jsonResponse(data: ['subject' => $identity->subject, 'roles' => $identity->roles]);
}
```

## 4. Connect to a popular OIDC provider (Google, Microsoft, Keycloak, Auth0…)

```php
use Waffle\Commons\Auth\Oauth\{OauthClient, OauthConfig, OidcDiscovery, Pkce};
use Waffle\Commons\Auth\Oauth\Preset\ProviderPreset;
use Waffle\Commons\Auth\Oauth\Transaction\{OauthTransaction, OauthTransactionCodec};

// Endpoints: from a shipped preset, or from OIDC discovery for any provider.
$metadata = ProviderPreset::Google->metadata();
// $metadata = new OidcDiscovery($http, $requestFactory, $cache)->discover('https://my-keycloak/realms/app');

$oauth = new OauthClient(
    config: new OauthConfig($metadata, $clientId, $clientSecret, 'https://app.example/callback'),
    http: $psr18Client, requests: $requestFactory, streams: $streamFactory,
);

// Login route: mint the stateless transaction (state/nonce/PKCE in a signed cookie).
$tx = OauthTransaction::start(returnTo: '/dashboard');
$codec = new OauthTransactionCodec($authSecret);                 // → WAFFLE_OAUTH_TX cookie
$url = $oauth->createAuthorizationUrl($tx->state, $tx->nonce, Pkce::challenge($tx->codeVerifier));

// Callback route: reopen the cookie, verify state, exchange the code.
$tx = $codec->decode($cookieValue);
hash_equals($tx->state, $queryState) || throw new OauthException('state mismatch', 403);
$tokens = $oauth->exchangeCode($queryCode, $tx->codeVerifier);
$identity = $jwtValidator->validate($tokens->idToken ?? '', expectedNonce: $tx->nonce);
```

PKCE is **always S256**; `state`/`nonce`/verifier never touch a session — they ride a
short-TTL HMAC-signed cookie. GitHub (OAuth2-only) resolves identities through its
userinfo endpoint instead of an ID token (`ProviderPreset::GitHub`).

For provider-published RS256 keys, swap the key resolver:
`new JwksKeyResolver($metadata->jwksUri, $http, $requestFactory, $cache)` — fetched once,
selected by `kid`, cached (PSR-16).

## 5. Authenticate outbound calls (RFC-021 §4.7)

```php
use Waffle\Commons\Auth\Client\AuthenticatedClient;
use Waffle\Commons\Auth\Credential\{AssertionCredentialsProvider, ClientCredentialsProvider};

$client = new AuthenticatedClient($psr18Client, [
    // Propagate the active identity to a downstream service (Strangler-Fig):
    new AssertionCredentialsProvider($signer, $context, allowedHosts: ['legacy-backend']),
    // Or acquire + cache a client-credentials token for a partner API:
    new ClientCredentialsProvider($oauth, $cache, allowedHosts: ['api.partner.example']),
]);
```

Providers are **host-gated** (credentials never leak to unrelated hosts) and **never
overwrite** a header the caller already set.

## 6. Receive gateway assertions downstream

A Waffle app behind a gateway verifies `X-Wfl-Assert-User` with one middleware:

```php
$stack->add(middleware: new GatewayAssertionMiddleware(new AuthBridgeVerifier($authSecret), $context));
```

Tampered signature, expired window (> 5 s), or client-IP mismatch ⇒ immediate **403**.
A non-Waffle monolith needs ~40 lines of plain PHP — see the reference implementation in
`workspace/docker/legacy/public/router.php`.

## See also

- [Explanation: The Universal Authentication Bridge](../explanation/authentication-universal-bridge.md)
- [Reference: Auth component](../reference/auth.md)
- [How to Secure Your Application](security.md) (authorization — ABAC voters)
