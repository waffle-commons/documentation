# Testing Reference (`waffle-commons/testing`)

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *Kernel testing bridge and test doubles (TEST-01)*
> **Requires:** PHP 8.5+, PHPUnit ^12.5. A development package: `composer require --dev waffle-commons/testing`.
> Depends only on `waffle-commons/contracts` and the PSR interfaces; PHPUnit is permitted in
> `Waffle\Commons\Testing\PHPUnit` only.
> Task guide: [How to Test Through the Kernel](../how-to/test-through-the-kernel.md).

The testing bridge sends PSR-7 requests through an application's **real** PSR-15 pipeline, in-process — no web
server, no network — and gives the test typed responses to assert on. Its doubles implement the same contracts as
production code, so a test swaps them in where the application resolves the real ones.

Only `WaffleTestCase` touches PHPUnit (the component's `mago.toml` guard holds it to that sub-namespace): the kernel
client and the doubles work with any test runner.

---

## 1. `PHPUnit\WaffleTestCase`

```php
abstract class WaffleTestCase extends TestCase
{
    abstract protected function createKernel(): KernelInterface;

    protected function createClient(?Closure $configure = null): KernelClient;

    public static function assertStatus(int $expected, TestResponse $response, string $message = ''): void;
    public static function assertHeader(string $expected, string $name, TestResponse $response, string $message = ''): void;
    public static function assertJsonBody(array $expected, TestResponse $response, string $message = ''): void;
}
```

- **`createKernel()`** builds the application's kernel, usually in one line through its kernel factory (for example
  `AppKernelFactory::create(Constant::ENV_TEST)`).
- **`createClient()`** boots a **fresh** kernel each time. The optional closure — `static function
  (ContainerInterface $container): void` — runs between the kernel's `boot()` and `configure()`, before the container
  locks: that is where a test sets its doubles. A double reaches what the container resolves (controllers and their
  dependencies), not a collaborator the kernel factory built up front.
- **`assertStatus()`** quotes the response body when the status differs; **`assertJsonBody()`** compares the decoded
  body with `$expected`.

## 2. The client and the response

### `Kernel\KernelClient`

```php
final readonly class KernelClient
{
    public const string DEFAULT_HOST = 'localhost';
    public const string DEFAULT_REMOTE_ADDR = '127.0.0.1';

    public static function boot(KernelInterface $kernel, ?Closure $configure = null): self;
    public function container(): ContainerInterface;

    public function request(
        string $method,
        string $uri,
        array $headers = [],
        string $body = '',
        array $server = [],
    ): TestResponse;

    public function jsonRequest(string $method, string $uri, array $data, array $headers = []): TestResponse;

    public function createRequest(
        string $method,
        string $uri,
        array $headers = [],
        string $body = '',
        array $server = [],
    ): ServerRequestInterface;

    public function send(ServerRequestInterface $request): TestResponse;
}
```

- **The worker's sequence.** Every request runs `handle()`, then `terminate()` when the kernel is terminable, then
  `reset()`, even when the kernel throws. Sending the same request twice therefore shows whether any state leaks from
  one request into the next.
- **Requests as production builds them.** The client builds requests with the PSR-17 `ServerRequestFactoryInterface`
  and `StreamFactoryInterface` from the kernel's container (the application's kernel factory registers them; the
  skeleton's does), and fills what the production `GlobalsFactory` derives: CGI server parameters, query parameters,
  cookies and the parsed body. Like production, a body is parsed only for `POST`: a urlencoded form or a JSON object.
  A relative URI is sent to `http://localhost`.
- **`jsonRequest()`** sends `$data` as a JSON body. **`createRequest()` + `send()`** give full control — pass
  `REMOTE_ADDR` and other server parameters in `$server`. Multipart bodies are not built.

### `Kernel\TestResponse`

```php
final readonly class TestResponse
{
    public function __construct(
        public ResponseInterface $response,
        public string $body,
    ) {}

    public function status(): int;
    public function header(string $name): string;
    public function hasHeader(string $name): bool;
    public function json(): array;
}
```

`$body` is the response body read once; `json()` decodes it and throws `Exception\InvalidJsonException` when it is not
a JSON object or list.

## 3. Test doubles

| Double | Stands in for | Behaviour |
| :--- | :--- | :--- |
| `Clock\FrozenClock` | `Psr\Clock\ClockInterface` (PSR-20) | Shows one instant until the test calls `set()` or `advance('+90 seconds')`. Production code uses `Waffle\Commons\Utils\Clock\SystemClock`. |
| `Http\ReplayClient` | PSR-18 `ClientInterface` and `ConcurrentClientInterface` | Replays the answers the test queued per method and URI, each once and in order, and records every request. A request with no answer left throws `UnexpectedRequestException`. |
| `Queue\InMemoryQueue` | `QueueDispatcherInterface` and `QueueConsumerInterface` | Records every dispatched envelope, with deterministic ids, and serves a worker: `receive()` follows the injected clock and never blocks. |
| `Queue\InMemoryFailedMessageStore` | `FailedMessageStoreInterface` | Keeps what a worker gave up on — full envelope, reason, instant from the clock — oldest first. |
| `Mailer\MailerSpy` | `MailerInterface` | Sends nothing; records every `Email`. |
| `RateLimit\InMemoryLimiterStateStorage` | `LimiterStateStorageInterface` | Token buckets in memory, with the Redis store's arithmetic on the injected clock: advance a `FrozenClock` to refill. One process only — production keeps limiter state in Redis, never in worker memory. |

```php
final class FrozenClock implements ClockInterface
{
    public function __construct(DateTimeImmutable|string $now);
    public function now(): DateTimeImmutable;
    public function set(DateTimeImmutable|string $now): void;
    public function advance(DateInterval|string $by): void;
}

final class ReplayClient implements ClientInterface, ConcurrentClientInterface
{
    public function addResponse(string $method, string $uri, ResponseInterface $response): void;
    public function addNetworkError(string $method, string $uri): void;
    /** @return list<RequestInterface> */
    public function requests(): array;
    public function pending(): int;
    public function sendRequest(RequestInterface $request): ResponseInterface;
    public function sendRequests(array $requests): array;
    public function promise(RequestInterface $request): PromiseInterface;
}

final class InMemoryQueue implements QueueDispatcherInterface, QueueConsumerInterface
{
    public function __construct(
        private readonly ClockInterface $clock,
    ) {}

    // dispatch(), receive(), acknowledge(), release(): the queue contracts
    public function reclaim(): void;
    /** @return list<Envelope> */
    public function dispatched(?string $queue = null): array;
    /** @return list<Envelope> */
    public function acknowledged(): array;
}

final class InMemoryFailedMessageStore implements FailedMessageStoreInterface
{
    public function __construct(
        private readonly ClockInterface $clock,
    ) {}
}

final class MailerSpy implements MailerInterface
{
    public function send(Email $email): void;
    /** @return list<Email> */
    public function sent(): array;
}

final class InMemoryLimiterStateStorage implements LimiterStateStorageInterface
{
    public function __construct(
        private readonly ClockInterface $clock,
    ) {}

    public function consume(string $key, TokenBucket $bucket, int $cost): RateLimitDecision;
}
```

- `ReplayClient::addNetworkError()` queues a `SimulatedNetworkException`, a PSR-18 `NetworkExceptionInterface`, so a
  test can exercise retries, fallbacks and circuit breakers without a real outage. `pending()` counts the answers not
  used yet.
- `InMemoryQueue::reclaim()` plays the recovery after a worker crash: the deliveries handed out and never settled come
  back. Settling a delivery it never handed out, or settling one twice, throws `UnknownDeliveryException`.

## 4. Exceptions (`Waffle\Commons\Testing\Exception`)

| Exception | Kind | When |
| :--- | :--- | :--- |
| `MissingServiceException` | `LogicException` | The kernel's container lacks a service the bridge needs, such as the PSR-17 factories. |
| `InvalidJsonException` | `UnexpectedValueException` | `TestResponse::json()` on a body that is not a JSON object or list. |
| `UnexpectedRequestException` | `LogicException` | The code under test sent a request `ReplayClient` has no answer for. Deliberately **not** a PSR-18 exception, so code that handles upstream failures cannot swallow it. |
| `SimulatedNetworkException` | `RuntimeException`, PSR-18 `NetworkExceptionInterface` | The network failure `addNetworkError()` queued. |
| `UnknownDeliveryException` | `LogicException`, `QueueExceptionInterface` | A delivery `InMemoryQueue` never handed out was settled, or one was settled twice. |

## 5. Worker safety

The doubles are mutable on purpose: a test sets the time, queues answers and reads what was recorded. They live for
one test and are never resident in a worker, so each mutable property carries `#[WorkerSafe]` with that reason, and
they do not implement `ResettableInterface` — the kernel resets after every request, which would erase what the test
asserts. `wfl igor` reports 0 KO.
