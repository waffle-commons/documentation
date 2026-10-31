# How to Run Background Jobs

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *A Redis Streams queue and the `queue:work` worker (QUEUE-01, QUEUE-02, QUEUE-03)*

This guide moves work out of the request: a controller dispatches a message, a separate worker process handles it,
retries it when it fails, and keeps it in a dead-letter store when it keeps failing. Every class and option named
here is in the [queue reference](../reference/queue.md).

**Use a queue** when the work can fail and must not be lost: sending mail, calling a slow third party, generating a
document. Short best-effort work that may run right after the response belongs to `async`'s finish-request deferral
instead ([How to Defer Post-Response Work](defer-post-response-work.md)); it runs inside the same worker and is not
retried.

**You need:** `waffle-commons/queue` (`composer require waffle-commons/queue`), Redis ≥ 6.2 reachable from both the
application and the worker, `ext-redis`, and a `DtoSerializerInterface` in your container
([How to Serialize DTOs](serialize-dtos.md)). Install `pcntl` in the worker's image so that it drains on SIGTERM.

---

## 1. Define the message and its handler

A message is a `#[Dto]` that implements `MessageInterface` and holds data only:

```php
use Waffle\Commons\Contracts\Attribute\Dto;
use Waffle\Commons\Contracts\Queue\MessageInterface;

#[Dto]
final readonly class SendInvoice implements MessageInterface
{
    public function __construct(
        public int $invoiceId,
    ) {}
}
```

The worker rebuilds the message through this constructor, so put any validation in it. A validating DTO uses PHP 8.5
property hooks, which cannot be `readonly`: declare it `final class` with `public private(set)` properties, as
`Envelope` and `Email` do.

The handler declares the class it handles. Delivery is at least once, so make it idempotent — here by passing the
envelope id on as an idempotency key:

```php
use InvalidArgumentException;
use Waffle\Commons\Contracts\Queue\Envelope;
use Waffle\Commons\Contracts\Queue\MessageHandlerInterface;
use Waffle\Commons\Contracts\Queue\MessageInterface;

final readonly class SendInvoiceHandler implements MessageHandlerInterface
{
    public function __construct(
        private InvoiceSender $invoices,
    ) {}

    #[\Override]
    public function handles(): string
    {
        return SendInvoice::class;
    }

    #[\Override]
    public function handle(MessageInterface $message, Envelope $envelope): void
    {
        if (!$message instanceof SendInvoice) {
            throw new InvalidArgumentException(sprintf('This handler sends invoices, not %s.', $message::class));
        }

        $this->invoices->send($message->invoiceId, idempotencyKey: $envelope->id);
    }
}
```

Throwing fails the delivery: the worker retries it later. Keep a handler well under 60 seconds (the claim idle time,
`StreamOptions::DEFAULT_CLAIM_IDLE_MS`), or another worker reclaims the message while it still runs.

## 2. Wire the worker

Register `queue:work` where your application registers its console commands, in `bin/waffle`. This is the workspace's
wiring, with a message of your own:

```php
use Waffle\Commons\Console\Command\QueueWorkCommand;
use Waffle\Commons\Contracts\Queue\MessageHandlerInterface;
use Waffle\Commons\Queue\Envelope\EnvelopeCodec;
use Waffle\Commons\Queue\RedisStreams\RedisFailedMessageStore;
use Waffle\Commons\Queue\RedisStreams\RedisStreamsQueue;
use Waffle\Commons\Queue\RedisStreams\StreamOptions;
use Waffle\Commons\Utils\Clock\SystemClock;

$clock = new SystemClock();
$handlers = [new SendInvoiceHandler($invoiceSender)];
$codec = new EnvelopeCodec(
    serializer: $serializer, // the application's DtoSerializerInterface
    messages: array_map(static fn(MessageHandlerInterface $handler): string => $handler->handles(), $handlers),
);

// No read timeout on this pool: the worker blocks on the stream for up to --wait milliseconds,
// which the 1 s read timeout of the HTTP pools would cut.
$queuePool = RedisPoolFactory::create($redisDsn, null);

$app->add(command: new QueueWorkCommand(
    consumer: new RedisStreamsQueue(
        pool: $queuePool,
        codec: $codec,
        clock: $clock,
        options: new StreamOptions(consumer: gethostname() ?: 'worker'),
    ),
    failed: new RedisFailedMessageStore(pool: $queuePool, codec: $codec, clock: $clock),
    handlers: $handlers,
    resettable: $container, // the application container, reset after every message
    clock: $clock,
    logger: $logger,
));
```

- **The handled classes are the allow-list.** The codec rebuilds only the classes the handlers declare; an entry of
  any other class is set aside in `{prefix}{queue}:unreadable`, never instantiated.
- **One consumer name per host.** `gethostname()` gives one per container. A name per process would grow the Redis
  consumer group without bound, since the worker recycles often.
- `RedisPoolFactory::create(string $dsn, ?float $readTimeout = self::READ_TIMEOUT)` is the template apps' pool
  factory (`App\Factory\RedisPoolFactory` in the skeleton); `null` disables the read timeout.

## 3. Dispatch from the application

Register the same driver as the dispatcher in your kernel factory, where the other services are set on the container:

```php
use Waffle\Commons\Contracts\Queue\QueueDispatcherInterface;

$container->set(QueueDispatcherInterface::class, new RedisStreamsQueue(
    pool: $redisPool,
    codec: new EnvelopeCodec(serializer: $serializer, messages: [SendInvoice::class]),
    clock: new SystemClock(),
    options: new StreamOptions(consumer: gethostname() ?: 'web'),
));
```

Code that starts background work then asks for the interface only:

```php
final readonly class InvoiceController
{
    public function __construct(
        private QueueDispatcherInterface $dispatcher,
    ) {}

    #[Route(path: '/invoices/{id}/send', name: 'invoice_send', methods: ['POST'])]
    public function send(int $id): void
    {
        $this->dispatcher->dispatch(new SendInvoice($id));
    }
}
```

The route needs its access rule like any other — a `#[Voter]`, or `#[PublicAccess]` for a public action — or the
fail-closed security layer answers 403 ([How to Secure a Controller](secure-a-controller.md)). A `void` action answers
204.

Pass a queue name to separate workloads (`dispatch($message, 'mail')`, then `queue:work --queue=mail`), and an
instant to delay one (`dispatch($message, availableAt: $clock->now()->modify('+10 minutes'))`). The envelope
records the current trace context, so the worker's spans continue the request's trace.

Use the same `StreamOptions` prefix on both sides: it names the Redis keys.

## 4. Run the worker

```bash
php bin/waffle queue:work --queue=default --max-messages=1000 --memory-limit=128
```

The worker exits 0 after `--max-messages` messages, at `--memory-limit`, or on SIGTERM / SIGINT once the message in
flight is settled; it exits 1 when Redis fails. Run it under something that restarts it:

- **Kubernetes:** a Deployment (its pods restart on exit) running the command above, with
  `terminationGracePeriodSeconds` longer than your slowest handler plus `--wait`.
- **Docker Compose:** a service with `restart: unless-stopped` and a `stop_grace_period` sized the same way.

Several workers can share a queue: each delivery goes to one consumer of the group.

## 5. Handle what keeps failing

A failed delivery comes back after `--backoff` seconds (5 by default), doubled at each further failure and capped at
an hour. After `--max-attempts` failed deliveries (3 by default) the message goes to the dead-letter store with its
envelope and the reason. To look at it and replay it:

```php
foreach ($failedStore->list(limit: 50) as $failed) {
    echo $failed->envelope->id, ' ', $failed->reason, PHP_EOL;

    $dispatcher->dispatch($failed->envelope->message, $failed->envelope->queue);
    $failedStore->remove($failed->envelope->id);
}
```

The store's Redis keys are `waffle:queue:failed` (a hash by id) and `waffle:queue:failed:order` (oldest first), so
`redis-cli HGETALL waffle:queue:failed` shows the raw records.

## 6. Send mail in the background

`Email` is already a queue message, and `MailerInterface` is the contract a transport implements; Waffle ships no
transport before v1. Bind your mailer, register a handler like the workspace's `EmailHandler` (it checks the message is
an `Email` and calls `$this->mailer->send($message)`), and dispatch from the request:

```php
$this->dispatcher->dispatch(new Email(
    from: 'billing@example.com',
    to: ['ada@example.com'],
    subject: 'Your invoice',
    text: 'Please find your invoice attached.',
));
```

`Email` refuses an invalid address, a subject with a line break, no recipient, or neither a text nor an HTML body.

## 7. Test it

In a kernel test, swap the dispatcher for `waffle-commons/testing`'s `InMemoryQueue` and assert what was dispatched
([How to Test Through the Kernel](test-through-the-kernel.md)):

```php
$queue = new InMemoryQueue(new FrozenClock('2026-11-05T09:00:00+00:00'));
$client = $this->createClient(static function (ContainerInterface $container) use ($queue): void {
    $container->set(QueueDispatcherInterface::class, $queue);
});

self::assertStatus(204, $client->request('POST', '/invoices/42/send')); // once the route's access rule passes
self::assertCount(1, $queue->dispatched());
self::assertInstanceOf(SendInvoice::class, $queue->dispatched()[0]->message);
```

To test a handler, call `handle()` directly with a message and an `Envelope`; `InMemoryQueue` can also serve a worker
(`receive()`, `acknowledge()`, `release()`, and `reclaim()` to play a crash).
