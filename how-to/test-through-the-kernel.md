# How to Test Through the Kernel

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *In-process requests through the real pipeline, with test doubles (TEST-01)*

A controller unit test bypasses the pipeline: it never sees the trusted-host check, the authentication bridge, the
fail-closed security layer or the DTO hydration — the places where a request is most often refused. This guide tests
through all of them, in-process, with `waffle-commons/testing`. Every class named here is in the
[testing reference](../reference/testing.md).

**You need:** `composer require --dev waffle-commons/testing` and PHPUnit 12.5.

---

## 1. Write the test case

Extend `WaffleTestCase` and build your kernel in `createKernel()`. This is the workspace's own bridge test:

```php
use Waffle\Commons\Contracts\Core\KernelInterface;
use Waffle\Commons\Testing\PHPUnit\WaffleTestCase;
use Workspace\Factory\AppKernelFactory;

final class KernelBridgeTest extends WaffleTestCase
{
    public static function setUpBeforeClass(): void
    {
        // Boot constants that public/index.php normally defines.
        if (!defined('APP_ROOT')) {
            define('APP_ROOT', dirname(__DIR__, 3));
        }
        if (!defined('APP_CONFIG')) {
            define('APP_CONFIG', 'config');
        }
    }

    #[\Override]
    protected function createKernel(): KernelInterface
    {
        return AppKernelFactory::create(env: 'dev', debug: false);
    }

    public function testTheHomePageAnswersThroughTheRealPipeline(): void
    {
        $client = $this->createClient();
        $first = $client->request('GET', '/');
        $second = $client->request('GET', '/');

        self::assertStatus(200, $first);
        // Same request, same response: no state leaks from one request into the next.
        self::assertJsonBody($first->json(), $second);
    }
}
```

Define the boot constants only if your kernel factory reads them, as the template apps' factories do.

## 2. Send requests and assert

```php
$client = $this->createClient();

$response = $client->request('GET', '/orders/42', headers: ['Authorization' => 'Bearer ' . $token]);
self::assertStatus(200, $response);
self::assertJsonBody(['id' => 42, 'status' => 'shipped'], $response);

$created = $client->jsonRequest('POST', '/orders', ['email' => 'ada@example.com', 'lines' => []]);
self::assertStatus(200, $created);                 // an action returning a #[Dto] answers 200 with its JSON
self::assertHeader('application/json', 'Content-Type', $created);
self::assertArrayHasKey('id', $created->json());
```

- Each request runs as the worker runs it: `handle()`, `terminate()`, then `reset()` — even when the kernel throws.
  Sending the same request twice catches state that leaks between requests.
- The request is built as production builds it, so the whole pipeline applies: an untrusted `Host` answers 400, a
  route without its access rule answers 403, a route marked `#[RequiresCsrfToken]` needs its token. Use these tests to
  prove those rules, not only to get around them.
- Use `createRequest()` and `send()` for full control, for example a client address in `$server['REMOTE_ADDR']`.

## 3. Swap in test doubles

Give `createClient()` a closure: it runs before the container locks, so whatever it sets is what the controllers
receive.

```php
use Psr\Clock\ClockInterface;
use Waffle\Commons\Contracts\Container\ContainerInterface;
use Waffle\Commons\Contracts\Mailer\MailerInterface;
use Waffle\Commons\Contracts\Queue\QueueDispatcherInterface;
use Waffle\Commons\Testing\Clock\FrozenClock;
use Waffle\Commons\Testing\Mailer\MailerSpy;
use Waffle\Commons\Testing\Queue\InMemoryQueue;

$clock = new FrozenClock('2026-11-05T09:00:00+00:00');
$queue = new InMemoryQueue($clock);
$mailer = new MailerSpy();

$client = $this->createClient(static function (ContainerInterface $container) use ($clock, $queue, $mailer): void {
    $container->set(ClockInterface::class, $clock);
    $container->set(QueueDispatcherInterface::class, $queue);
    $container->set(MailerInterface::class, $mailer);
});

$client->jsonRequest('POST', '/invoices/42/send', []);

self::assertCount(1, $queue->dispatched());
$clock->advance('+1 hour'); // time moves only when the test says so
```

A double reaches what the container resolves — controllers and their dependencies. A collaborator your kernel factory
builds up front, outside the container, does not see it: move it into the container, or test it on its own.

## 4. Replay an upstream

`ReplayClient` answers outbound calls from a script and records them, so no test reaches the network:

```php
$upstream = new ReplayClient(); // $responseFactory: any PSR-17 ResponseFactoryInterface
$upstream->addResponse('GET', 'https://rates.example.com/eur', $responseFactory->createResponse(200));
$upstream->addNetworkError('GET', 'https://rates.example.com/usd'); // a PSR-18 NetworkExceptionInterface

// set it in the closure, under the id your code depends on, then:
self::assertSame(0, $upstream->pending()); // every scripted answer was used
```

A request it has no answer for throws `UnexpectedRequestException`, which is deliberately not a PSR-18 exception: code
that handles upstream failures cannot swallow it, so an unplanned call fails the test.

## 5. Test a queue worker without Redis

`InMemoryQueue` is also a consumer: `receive()` hands out what was dispatched (following the clock, never blocking),
`acknowledge()` and `release()` settle a delivery, and `reclaim()` plays a worker crash. Pair it with
`InMemoryFailedMessageStore` to check what lands in the dead-letter store
([How to Run Background Jobs](run-background-jobs.md#7-test-it)).
