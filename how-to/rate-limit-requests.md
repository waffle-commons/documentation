# How to Rate-Limit Requests

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *Token-bucket rate limiting at the edge (NET-01, RFC-017 §3.1)*

This guide turns on the token-bucket limiter, picks what a bucket is counted against, and decides what
happens when its storage is down. The skeleton ships it wired and enabled; the steps below show what that
wiring does, so you can tune it or reproduce it in your own kernel factory.

## 1. Give it a shared Redis

The limiter keeps its buckets in Redis, never in worker memory: a FrankenPHP pod runs many workers, and a
per-worker bucket would admit *capacity × workers* requests. Each check is one Lua script on the Redis
server, so concurrent workers never admit more than the bucket holds.

The skeleton's `docker-compose.yml` runs a `waffle-redis` service and `.env.example` points `REDIS_DSN` at
it. In production, give `REDIS_DSN` your own Redis:

```dotenv
REDIS_DSN=redis://:a-password@redis.internal:6379
```

The limiter talks to Redis through the phpredis pool of `waffle-commons/data`, so the image needs the
`redis` extension (the skeleton's `docker/Dockerfile` installs it, and its `composer.json` requires
`ext-redis`).

## 2. Configure the bucket

```yaml
# config/app.yaml
waffle:
  security:
    rate_limit:
      enabled: true
      capacity: 120                  # requests per period, bursts included
      period_seconds: 60             # 1 to 86400
      on_storage_failure: 'open'     # open | closed
      trusted_proxies: []            # proxy addresses or CIDR ranges
      redis_dsn: '%env(REDIS_DSN)%'
```

`capacity: 120` with `period_seconds: 60` is a `TokenBucket(120, 60)`: a client may send 120 requests at
once, then one more every half second as the bucket refills. Every key is listed in
[Reference: Configuration Keys](../reference/configuration-keys.md).

## 3. Mount it after authentication

The middleware counts against the **authenticated subject** when there is one, otherwise against the
**client IP**. Mount it after `AuthenticationMiddleware`, so the subject is known, and after the probes
and `/waffle-metrics`, so orchestrator and Prometheus traffic is never throttled. The skeleton's
`App\Factory\RateLimitFactory` does both:

```php
// src/Factory/AppKernelFactory.php — after AuthenticationMiddleware
RateLimitFactory::register($stack, $config, $securityContext, $responseFactory, $secureLogger);
```

Its core builds the limiter from the configuration:

```php
$limiter = new RateLimiter(
    new RedisLimiterStateStorage(RedisPoolFactory::create($dsn)),
    new TokenBucket($capacity, $periodSeconds),
    StorageFailureMode::tryFrom($onStorageFailure) ?? StorageFailureMode::Open,
    $logger,
);

$stack->add(middleware: new RateLimitMiddleware(
    $limiter,
    new TrustedProxyClientIpResolver($trustedProxies),
    $securityContext,
    $responseFactory,
));
```

The bucket keys are `sub:<subject>` for an authenticated request, `ip:<address>` otherwise, and
`anonymous` for a request that carries no address at all. In Redis they live under `waffle:ratelimit:`.

## 4. Behind a proxy or an ingress, list the proxies

Behind a load balancer `REMOTE_ADDR` is the proxy's address, and every client would share its bucket.
List the proxies:

```yaml
      trusted_proxies: ['10.0.0.0/8', 'fd00::/8']
```

`TrustedProxyClientIpResolver` then reads `X-Forwarded-For` **only** when `REMOTE_ADDR` is one of them,
and walks it from the right: each hop a trusted proxy appended is skipped, and the first address that is
not a trusted proxy is the client. A client can prepend whatever it likes to the header; it cannot make
your proxy append a false hop. Never trust `X-Forwarded-For` without this list: the client would choose
its own bucket.

## 5. Decide what a Redis outage does

`on_storage_failure` is the limiter's answer when Redis is unreachable:

- **`open`** (default): the request goes through, a warning is logged on the `SECURITY` channel, and no
  `RateLimit-*` header is sent — the counts are unknown. The limiter protects capacity, and a Redis outage
  must not take the whole API down.
- **`closed`**: the request is refused with `503`. Use it where the limiter is the brute-force defence.

Authentication and authorization stay fail-closed whatever this setting says.

## 6. Protect a login endpoint with its own limiter

A global per-IP bucket does little against credential stuffing spread over many addresses. Give the
login route a stricter limiter, keyed by the account under attack and closed on outage:

```php
$loginLimiter = new RateLimiter(
    new RedisLimiterStateStorage(RedisPoolFactory::create($dsn)),
    new TokenBucket(5, 300),            // 5 attempts per account every 5 minutes
    StorageFailureMode::Closed,
    $logger,
);

$decision = $loginLimiter->consume('login:' . $username);
if (!$decision->allowed) {
    // 429 when the bucket is empty; 503 when the storage was unreachable ($decision->enforced === false)
}
```

## 7. Check it

```bash
curl -sk -D - -o /dev/null https://localhost/ | grep -i -e '^ratelimit' -e '^retry-after'
# RateLimit-Limit: 120
# RateLimit-Remaining: 119
# RateLimit-Reset: <seconds until the bucket is full again>
```

Every enforced response carries `RateLimit-Limit`, `RateLimit-Remaining` and `RateLimit-Reset`. An empty
bucket answers `429` with `Retry-After` and an RFC 7807 body:

```json
{"title":"Too Many Requests","status":429}
```

On the skeleton's own dev stack, the 122nd request of a minute got that `429` with `Retry-After: 1` (the
bucket refills while the requests arrive), and `/healthz` still answered `200`.

## Related

- [Reference: Security — rate limiting](../reference/security.md#rate-limiting-net-01)
- [Reference: HTTP — client IP behind trusted proxies](../reference/http.md#client-ip-behind-trusted-proxies-net-01)
- [Reference: Data — shared state for limiters and breakers](../reference/data.md#shared-state-for-limiters-and-breakers-net-0103)
- [How to: Make Outbound Calls Resilient](make-outbound-calls-resilient.md) — the breaker shares this storage family.
