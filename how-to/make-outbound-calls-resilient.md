# How to Make Outbound Calls Resilient

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *Timeouts, bounded retries and a shared circuit breaker (NET-02/03, RFC-024)*

An upstream that slows down or fails should cost your API a bounded amount of time, not a worker per
request. This guide sets the transfer timeouts, adds bounded retries, and puts a circuit breaker in front
of each upstream origin. The skeleton ships all three wired from `config/app.yaml`; the steps show what
that wiring does.

## 1. Bound every transfer

`Client` takes its connect and total timeouts from a `TimeoutPolicy`. The defaults are 1 s to connect and
10 s in total, and the policy cannot express an infinite timeout: cURL reads `0` as "no timeout", so its
property hooks refuse anything below 1 ms and above 30 000 ms (connect) or 120 000 ms (total).

```yaml
# config/app.yaml
waffle:
  http_client:
    resilience:
      connect_timeout_ms: 1000
      total_timeout_ms: 10000
```

```php
$client = new Client($responseFactory, $streamFactory, new SsrfGuard())
    ->withTimeouts(new TimeoutPolicy(connectMs: 500, totalMs: 4000));
```

`withTimeouts()` returns a copy: the client is `final readonly`, so a per-call override never changes the
instance other requests use.

## 2. Retry what is safe to retry

`RetryingClient` wraps any PSR-18 client:

```yaml
    resilience:
      max_attempts: 3            # 1 to 10, the first attempt included; 1 turns retries off
      base_delay_ms: 100
      max_delay_ms: 2000
      budget_ms: 15000
      retry_non_idempotent: false
```

- **What is retried:** a transport failure (`NetworkExceptionInterface`) or a `429`, `502`, `503` or
  `504` response (`RetryPolicy::RETRYABLE_STATUSES`).
- **Which requests:** idempotent methods only — `GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE` — unless
  `retry_non_idempotent` is set. A `POST` that timed out may have been processed; retrying it is your
  decision, not the client's.
- **How long it waits:** exponential backoff with full jitter, capped at `max_delay_ms`. A numeric
  `Retry-After` from the upstream may exceed the cap when it fits the remaining budget.
- **The hard bound:** no wait starts once `budget_ms` would be exceeded; the last response or failure is
  returned instead. Retries cannot multiply the worst case.
- **Every attempt is a new transfer** through the inner client, so the SSRF guard resolves, validates and
  pins the host again — DNS can change between attempts.

## 3. Put a breaker in front of each upstream

`CircuitBreakerClient` keeps one circuit per target origin (`https://api.example.com:443`). Its state lives
in Redis, so every worker sees the same circuit; a per-worker breaker would open once per worker and
protect nothing.

```yaml
    breaker:
      enabled: true
      failure_rate_percent: 50     # open when half the calls in the window failed…
      minimum_calls: 10            # …out of at least 10 calls
      window_seconds: 60
      cool_down_seconds: 30        # then let a trial call through
      half_open_trials: 1
      redis_dsn: '%env(REDIS_DSN)%'
```

- A transport failure or a `5xx` response counts as a failure; any other response, a `4xx` included,
  counts as a success: the upstream answered.
- An open circuit fails fast with `CircuitOpenException`, code `503`, before any connection is opened. The
  error handler renders that status.
- After the cool-down the circuit is half-open: `half_open_trials` calls go through; a success closes the
  circuit, a failure opens it again.
- **When Redis itself is down**, the breaker behaves as closed: calls go through, with a warning and the
  `waffle_circuit_storage_errors_total` metric. Failing fast would turn a Redis outage into an outage of
  every upstream integration; the timeouts still bound each attempt.

## 4. Compose them in this order

```php
// RetryingClient(CircuitBreakerClient(client)): every attempt asks the breaker first,
// and the breaker's refusal is never retried. RedisPoolFactory is the skeleton's
// App\Factory\RedisPoolFactory (a phpredis pool from a redis:// DSN).
$resilient = new RetryingClient(
    new CircuitBreakerClient(
        $client,
        new RedisCircuitStateStorage(RedisPoolFactory::create($dsn)),
        new CircuitBreakerPolicy(failureRatePercent: 50, minimumCalls: 10),
        new CircuitObservers($eventDispatcher, $metricsRegistry, $logger),
    ),
    new RetryPolicy(maxAttempts: 3),
);
```

The skeleton's `App\Factory\HttpClientFactory::register()` builds exactly this for both outbound clients
it puts in the container: `ConcurrentClientInterface` (the fan-out demo) and `ClientInterface`, whose inner
client is `AuthenticatedSettlingClient` — the outbound credentials stay **under** the retries, so the
signed assertion is applied again on every attempt.

## 5. Keep concurrent batches concurrent

`ConcurrentClientInterface::sendRequests()` throws the first failure and drops the other outcomes, which a
breaker and a retry policy both need. The decorators therefore settle a batch through
`SettlingClientInterface::settleRequests()`: every request ends with its response or its exception, the
breaker checks every request's circuit before any is sent, and retries go out in waves — each request
keeps its own attempts and budget. A decorator that does not implement `SettlingClientInterface` below
them is sent one request at a time, which is why the skeleton's credentials decorator is the *settling*
one.

## 6. Watch the circuits

- **Events:** each transition dispatches `Waffle\Commons\Contracts\CircuitBreaker\Event\CircuitStateChanged`
  (`$circuit`, `$from`, `$to`) through PSR-14 — a listener in any component can react without importing
  `http-client`.
- **Metrics** on `/waffle-metrics`: `waffle_circuit_transitions_total{circuit,to}`,
  `waffle_circuit_state{circuit}` (0 closed, 1 half-open, 2 open) and
  `waffle_circuit_storage_errors_total{circuit}`.
- **Redis:** each circuit is a hash under `waffle:circuit:<origin>`.

Under fault injection on the workspace stack (beta7, 2026-10-21), stopping the upstream opened its circuit
on the first failing request; while open, requests failed in 11–23 ms without a call; after the cool-down
a trial with the upstream still down re-opened it, and with the upstream back the trial closed it.

## Related

- [Reference: HTTP Client — resilience](../reference/http-client.md#resilience-net-0203)
- [Reference: Contracts — outbound resilience](../reference/contracts.md#outbound-resilience-net-0203)
- [Reference: Configuration Keys](../reference/configuration-keys.md)
- [How to: Rate-Limit Requests](rate-limit-requests.md)
