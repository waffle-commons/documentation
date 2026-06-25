# How-To: Configure CORS

> **Beta 4 (SEC-04)** — Waffle ships a dedicated, **fail-closed** CORS middleware. With no configured origins, *every* cross-origin request is refused. You open the door explicitly, one exact origin at a time — there is no permissive default to forget to lock down.

## 1. The fail-closed default

`Waffle\Commons\Security\Middleware\CorsMiddleware` is wired by the skeleton's `AppKernelFactory` with an **empty allow-list**. That means:

- A request with **no `Origin`** header (server-to-server, `curl`, same-origin navigations) passes straight through, untouched.
- A request whose `Origin` is **same-origin** passes through, untouched.
- A **cross-origin** request whose `Origin` is not allow-listed is rejected with HTTP `403` **before the controller runs** — a forged cross-site write never executes.

So out of the box your API is closed to browsers on other origins. You make it usable from a front-end by listing that front-end's origin.

## 2. Allow your front-end origins

Add exact origins (`scheme://host[:port]`) under `waffle.security.cors.allowed_origins` in `config/app.yaml`:

```yaml
# config/app.yaml
waffle:
  security:
    # CORS fail-closed (SEC-04): exact-origin allow-list.
    # Empty ⇒ every cross-origin request is refused.
    cors:
      allowed_origins:
        - 'https://app.example.com'
        - 'http://localhost:5173'   # local Vite/React dev server
```

Origins are matched **exactly** and case-sensitively. `https://app.example.com` does **not** allow `https://app.example.com:8443` or `http://app.example.com` — list each one you actually serve.

## 3. The `*` wildcard — and why it is banned with credentials

`CorsPolicy` accepts a `*` wildcard **only for non-credentialed policies**. Combining `*` with `allowCredentials: true` is rejected at construction time (per the Fetch standard), because it would expose authenticated responses to every site on the web:

```php
use Waffle\Commons\Security\Cors\CorsPolicy;

// ✅ public, read-only API with no cookies/Authorization → wildcard is fine
new CorsPolicy(allowedOrigins: ['*']);

// ❌ throws InvalidArgumentException — wildcard + credentials is forbidden
new CorsPolicy(allowedOrigins: ['*'], allowCredentials: true);
```

For any endpoint that carries cookies, `Authorization`, or session state, always list **exact** origins. The middleware then echoes the *exact* request origin back in `Access-Control-Allow-Origin` (never a blanket `*`) and adds `Vary: Origin`.

## 4. Tuning methods, headers, credentials, and cache

The skeleton wires only the origin allow-list and takes `CorsPolicy`'s secure defaults for the rest. To customise, construct the policy yourself in your kernel factory:

```php
use Waffle\Commons\Security\Cors\CorsPolicy;
use Waffle\Commons\Security\Middleware\CorsMiddleware;

$policy = new CorsPolicy(
    allowedOrigins:   ['https://app.example.com'],
    allowedMethods:   ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'], // default
    allowedHeaders:   ['Content-Type', 'Authorization', 'X-CSRF-Token'],    // default
    allowCredentials: true,   // send cookies / Authorization cross-origin
    maxAge:           600,    // pre-flight cache, seconds (default)
);

$stack->add(new CorsMiddleware($policy, $responseFactory));
```

When `allowCredentials` is `true`, the middleware adds `Access-Control-Allow-Credentials: true` to both pre-flight and actual responses.

## 5. Pre-flight (`OPTIONS`) handling

A browser pre-flight — an `OPTIONS` request carrying `Access-Control-Request-Method` — is answered entirely inside the middleware:

- **Allowed origin** → `204 No Content` with the negotiated `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, and `Access-Control-Max-Age`.
- **Disallowed origin** → `403`. The application handler never runs.

This is why `CorsMiddleware` must sit **before routing**: it answers the pre-flight ahead of the router's own `OPTIONS` short-circuit.

## 6. Pipeline placement

The skeleton's canonical middleware order already places CORS correctly — early, right after host validation and before anything that touches application state:

```
ErrorHandler → TrustedHost → Cors → AnonymousSession → Authentication → Routing → Csrf → Security → SecureHeaders → Dispatcher
```

`CorsMiddleware` is stateless across requests (FrankenPHP worker rule) and holds no mutable state. See the [Security reference](../reference/security.md) and the [Secure a Controller](secure-a-controller.md) guide for how CORS composes with the ABAC and CSRF layers.
