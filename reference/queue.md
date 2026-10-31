# Queue Reference (`waffle-commons/queue`)

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *Background jobs on Redis Streams (QUEUE-01, QUEUE-02, QUEUE-03)*
> **Requires:** PHP 8.5+, `ext-redis`, Redis ≥ 6.2 — one server or a Sentinel-managed primary, not Redis Cluster.
> Depends only on `waffle-commons/contracts` and `psr/clock`.
> **Companions:** the worker command `queue:work` ships in [`waffle-commons/console`](console.md); the envelope codec
> writes messages through the [serializer](serializer.md); the test doubles are in [`waffle-commons/testing`](testing.md).
> Task guide: [How to Run Background Jobs](../how-to/run-background-jobs.md).

A queue carries background jobs from the process that dispatches them to a worker process that handles them. It is
deliberately small — one driver (Redis Streams), one worker, at-least-once delivery, bounded retries and a
dead-letter store — and not a message bus. It is distinct from `async`'s finish-request deferral, which runs after
the response inside the same request ([async.md](async.md)).

Like every component, `queue` depends only on `waffle-commons/contracts` (plus the PSR clock interface): the
serializer, the Redis pool and the tracer all come in through contracts.

---

## 1. Contracts (`Waffle\Commons\Contracts\Queue`)

### `MessageInterface`

```php
interface MessageInterface {}
```

A background job is a strictly typed `#[Dto]` that implements this marker interface. It holds data only — scalars,
backed enums, nested DTOs and lists of them — never a closure, a resource or a service. A consumer rebuilds it, in
another process, **through its constructor**, so its validation runs again; it rebuilds only the message classes it
allows, and never with `unserialize()`: a queue is an input channel. Delivery is at least once, so a handler is
idempotent or de-duplicates on the envelope id.

### `Envelope`

A message on its way through a queue, with what delivery needs to know. The constructor uses PHP 8.5 **property
hooks** on promoted parameters (each `set` block validates before the value is stored) and **asymmetric
visibility** (`public private(set)`: readable everywhere, writable only inside the class):

```php
final class Envelope
{
    public function __construct(
        public private(set) string $id {
            set(string $value) {
                if ($value === '') {
                    throw new InvalidArgumentException('An envelope id must not be empty.');
                }
                $this->id = $value;
            }
        },
        public private(set) MessageInterface $message,
        public private(set) string $queue {
            set(string $value) {
                if ($value === '') {
                    throw new InvalidArgumentException('An envelope queue must not be empty.');
                }
                $this->queue = $value;
            }
        },
        public private(set) DateTimeImmutable $availableAt,
        public private(set) int $attempts = 0 {
            set(int $value) {
                if ($value < 0) {
                    throw new InvalidArgumentException('An envelope cannot have a negative number of attempts.');
                }
                $this->attempts = $value;
            }
        },
        public private(set) array $traceContext = [],
        public private(set) ?string $receipt = null,
    ) {}

    public function withReceipt(string $receipt): self;
    public function withNextAttempt(DateTimeImmutable $availableAt): self;
}
```

| Property | Meaning |
| :--- | :--- |
| `$id` | Unique and stable across deliveries: handlers de-duplicate on it. |
| `$queue` | The queue the message was dispatched to. |
| `$availableAt` | No consumer delivers the message before this instant. |
| `$attempts` | The deliveries that already failed: 0 on the first delivery. |
| `$traceContext` | `array<string, string>`: the W3C trace context (`traceparent`, `tracestate`) of the dispatching code, so the worker continues the trace. |
| `$receipt` | The transport's handle for the current delivery (a Redis stream entry id); `null` until a consumer receives it. |

The `with` copies are the only way to change an envelope. They use PHP 8.5's **`clone` with properties**
(`clone($this, ['receipt' => $receipt])`), which runs the same hooks as the constructor. `withNextAttempt()` adds
one attempt, sets the new availability and clears the receipt.

### `QueueDispatcherInterface`

```php
interface QueueDispatcherInterface
{
    public const string DEFAULT_QUEUE = 'default';

    public function dispatch(
        MessageInterface $message,
        string $queue = self::DEFAULT_QUEUE,
        ?DateTimeImmutable $availableAt = null,
    ): Envelope;
}
```

Code that starts background work depends on this interface only. The dispatcher assigns the id and records the
current trace context. The message is available at once, or from `$availableAt` for delayed work. Throws
`QueueExceptionInterface` when the queue cannot take the message.

### `QueueConsumerInterface`

```php
interface QueueConsumerInterface
{
    public function receive(string $queue, int $timeoutMs = 0): ?Envelope;
    public function acknowledge(Envelope $envelope): void;
    public function release(Envelope $envelope, DateTimeImmutable $availableAt): void;
}
```

A received message is in flight until the worker settles it: `acknowledge()` when it was handled (it is never
delivered again), `release()` to deliver it again from `$availableAt`, as `$envelope->withNextAttempt($availableAt)`.
`receive()` waits up to `$timeoutMs` milliseconds (0: do not wait) and returns `null` when nothing became available;
the envelope it returns carries the receipt of this delivery. A delivery that is never settled (the worker crashed)
is delivered again once the transport reclaims it: no message is lost.

### `MessageHandlerInterface`

```php
interface MessageHandlerInterface
{
    /** @return class-string<MessageInterface> */
    public function handles(): string;

    public function handle(MessageInterface $message, Envelope $envelope): void;
}
```

The worker hands each message to the handler of its class. Returning means the message was handled; throwing means
the delivery failed. The classes the registered handlers declare are also the only ones a consumer rebuilds: a
message nobody handles is never instantiated.

### `FailedMessageStoreInterface` and `FailedMessage`

```php
interface FailedMessageStoreInterface
{
    public function store(Envelope $envelope, string $reason): void;
    /** @return list<FailedMessage> */
    public function list(int $limit = 100): array;
    public function remove(string $id): void;
}

final readonly class FailedMessage
{
    public function __construct(
        public Envelope $envelope,
        public string $reason,
        public DateTimeImmutable $failedAt,
    ) {}
}
```

The dead-letter store keeps a message once its attempts are exhausted, so that nothing is dropped silently.
`list()` returns the oldest first. `FailedMessage::$envelope` is the envelope as it would be delivered again: its
attempts count every failed delivery, the last one included. `$reason` is the exception class and message.

### `Exception\QueueExceptionInterface`

```php
interface QueueExceptionInterface extends Throwable {}
```

A transport failure — the broker is unreachable, refused a command or returned an unreadable message — catchable
without coupling to a driver.

### Mail (`Waffle\Commons\Contracts\Mailer`, QUEUE-03)

```php
interface MailerInterface
{
    public function send(Email $email): void;
}
```

Sending transactional mail is a **contract only**: SMTP and API transports are post-v1, and an application binds any
mailer it chooses. Sending blocks on the network, so a request does not send: it dispatches the `Email`, which is a
queue message, and a worker hands it to the mailer.

`Email` is a `#[Dto] final class Email implements MessageInterface`. Its hooks check every address and keep CR and LF
out of the subject, so a value from a request cannot inject a header:

```php
public function __construct(
    public private(set) string $from { set(string $value) { $this->from = self::address($value, 'from'); } },
    #[ListOf('string')]
    public private(set) array $to { set(array $value) { /* at least one recipient, each a valid address */ } },
    public private(set) string $subject {
        set(string $value) {
            if ($value === '' || strpbrk($value, "\r\n") !== false) {
                throw new InvalidArgumentException('An email subject is one line of text.');
            }
            $this->subject = $value;
        }
    },
    public private(set) ?string $text = null,
    public private(set) ?string $html = null,
    #[ListOf('string')]
    public private(set) array $cc = [] { /* each a valid address */ },
    #[ListOf('string')]
    public private(set) array $bcc = [] { /* each a valid address */ },
    public private(set) ?string $replyTo = null { /* a valid address, or null */ },
)
```

The constructor also refuses an email with neither a text nor an HTML body. Addresses are plain
(`ada@example.com`), without a display name. Elided hook bodies (`/* … */`) are summarised; the source is
`contracts/src/Mailer/Email.php`.

---

## 2. The Redis Streams driver (`Waffle\Commons\Queue`)

### `RedisStreams\RedisStreamsQueue`

```php
final readonly class RedisStreamsQueue implements QueueDispatcherInterface, QueueConsumerInterface
{
    public function __construct(
        private RedisConnectionPoolInterface $pool,
        private EnvelopeCodec $codec,
        private ClockInterface $clock,
        private StreamOptions $options,
        private TracerInterface $tracer = new NullTracer(),
        private TextMapPropagatorInterface $propagator = new NullTextMapPropagator(),
    ) {}
}
```

Connections come from the contracts' `RedisConnectionPoolInterface` (phpredis clients), leased per operation and
always released; the class keeps no state between calls.

- **`dispatch()`** creates the envelope (a 32-character hexadecimal id from `random_bytes(16)`, the current trace
  context through the tracer and propagator), then adds it to the queue's stream, or to the queue's delayed set
  when `$availableAt` is later than now.
- **`receive()`** first moves the delayed messages that are due into the stream (at most `promoteBatch` per call).
  It then reclaims a delivery that a worker left unsettled for longer than `claimIdleMs` (`XAUTOCLAIM`): the crash
  counts as a failed attempt. Only then does it read a new entry (`XREADGROUP`, waiting up to the timeout, 1 ms at
  least). The consumer group is created on first use, from the start of the stream. An entry the codec cannot read
  is moved to the queue's `:unreadable` stream and settled, so it never blocks the queue.
- **`acknowledge()`** removes the entry (`XACK`, then `XDEL`).
- **`release()`** settles the failed delivery and queues the next attempt — at once or delayed — in **one Lua
  script**: a crash between the two steps cannot drop the message.

`acknowledge()` and `release()` throw `QueueException` when the envelope carries no receipt.

### `RedisStreams\StreamOptions`

```php
final class StreamOptions
{
    public const string DEFAULT_PREFIX = 'waffle:queue:';
    public const string DEFAULT_GROUP = 'waffle';
    public const int DEFAULT_CLAIM_IDLE_MS = 60_000;
    public const int DEFAULT_PROMOTE_BATCH = 100;

    public function __construct(
        public private(set) string $consumer { set(string $value) { /* not empty */ } },
        public private(set) string $prefix = self::DEFAULT_PREFIX { set(string $value) { /* not empty */ } },
        public private(set) string $group = self::DEFAULT_GROUP { set(string $value) { /* not empty */ } },
        public private(set) int $claimIdleMs = self::DEFAULT_CLAIM_IDLE_MS { set(int $value) { /* at least 1 */ } },
        public private(set) int $promoteBatch = self::DEFAULT_PROMOTE_BATCH { set(int $value) { /* at least 1 */ } },
    ) {}

    public function stream(string $queue): string;     // {prefix}{queue}
    public function delayed(string $queue): string;    // {prefix}{queue}:delayed
    public function unreadable(string $queue): string; // {prefix}{queue}:unreadable
}
```

Each hook throws `InvalidArgumentException` naming the option. Give `$consumer` one name per host (per container),
not per process: Redis keeps every consumer name in the group until it is deleted, and the worker recycles often.

**Redis keys:**

| Key | Type | Holds |
| :--- | :--- | :--- |
| `{prefix}{queue}` | stream, consumer group `{group}` | the messages ready for delivery, and the deliveries in flight |
| `{prefix}{queue}:delayed` | sorted set, scored by due time (ms) | delayed messages and retries waiting for their back-off |
| `{prefix}{queue}:unreadable` | stream | the entries the codec refused, for inspection |
| `{prefix}failed`, `{prefix}failed:order` | hash, sorted set | the dead-letter store |

### `RedisStreams\RedisFailedMessageStore`

```php
final readonly class RedisFailedMessageStore implements FailedMessageStoreInterface
{
    public function __construct(
        private RedisConnectionPoolInterface $pool,
        private EnvelopeCodec $codec,
        private ClockInterface $clock,
        private string $prefix = StreamOptions::DEFAULT_PREFIX,
    ) {}
}
```

Each failed message is kept with its full envelope (id, attempts, trace context), the reason and the failure time
from the clock, in `{prefix}failed` (by id) and `{prefix}failed:order` (by failure time, oldest first). A record it
cannot read back is skipped by `list()`.

### `Envelope\EnvelopeCodec`

```php
final readonly class EnvelopeCodec
{
    /** @param list<class-string<MessageInterface>> $messages The message classes it rebuilds (those a handler handles). */
    public function __construct(
        private DtoSerializerInterface $serializer,
        private array $messages,
    ) {}

    public function encode(Envelope $envelope): string;
    public function decode(string $json): ?Envelope;
    public static function milliseconds(DateTimeImmutable $instant): int;
    public static function instant(int $milliseconds): ?DateTimeImmutable;
}
```

`encode()` writes the envelope as one JSON object — `id`, `type` (the message class), `queue`, `payload` (the message
through `DtoSerializerInterface::serialize()`), `attempts`, `available_at` (milliseconds since the epoch) and `trace`.
`decode()` reads one back **only into a class listed in `$messages`**, through the serializer and so through the
class constructor and its hooks. It never throws: anything it cannot read — malformed JSON, a missing field, a class
nobody handles, a payload the message refuses — returns `null`, which the driver sets aside as unreadable.

### `Health\StreamsHealthCheck`

```php
final readonly class StreamsHealthCheck implements HealthCheckInterface
{
    public function __construct(
        private RedisConnectionPoolInterface $pool,
        private StreamOptions $options,
        private string $queue = QueueDispatcherInterface::DEFAULT_QUEUE,
        private string $name = 'queue',
        private bool $critical = true,
    ) {}
}
```

A `/readyz` probe ([pipeline.md](pipeline.md#orchestrator-probes-ops-0102)): it leases a connection and runs `XLEN` on
the queue's stream. Up reports `N message(s) waiting`; any failure is reported down with its reason, never thrown, as
the health contract requires.

### `Exception\QueueException`

`final class QueueException extends RuntimeException implements QueueExceptionInterface`, built through named
constructors: `transport()` (Redis or the pool failed), `refused()` (Redis refused a command), `unexpectedClient()`
(the pool did not hand out an ext-redis client), `notReceived()` (an envelope without a receipt was settled) and
`unencodable()` (the serializer or JSON encoding refused the envelope).

---

## 3. The worker: `queue:work` (`waffle-commons/console`)

```php
final readonly class QueueWorkCommand extends AbstractCommand
{
    /**
     * @param list<MessageHandlerInterface> $handlers   One handler per message class.
     * @param ResettableInterface           $resettable What is reset after every message: the application container.
     */
    public function __construct(
        private QueueConsumerInterface $consumer,
        private FailedMessageStoreInterface $failed,
        private array $handlers,
        private ResettableInterface $resettable,
        private ClockInterface $clock,
        private LoggerInterface $logger = new NullLogger(),
        private TracerInterface $tracer = new NullTracer(),
        private TextMapPropagatorInterface $propagator = new NullTextMapPropagator(),
    ) {}
}
```

The command depends on contracts only; the application builds the driver and registers the command in its
`bin/waffle` ([How to Run Background Jobs](../how-to/run-background-jobs.md#2-wire-the-worker)).

```text
queue:work [--queue=default] [--max-messages=1000] [--memory-limit=128] [--max-attempts=3] [--backoff=5] [--wait=1000]
```

| Option | Default | Accepted | Meaning |
| :--- | :--- | :--- | :--- |
| `--queue` | `default` | not empty | The queue to work on. |
| `--max-messages` | `1000` | ≥ 1 | How many messages to settle before the worker exits to be restarted. |
| `--memory-limit` | `128` (MB) | 1 – 1 048 576 | The memory at which the worker exits to be restarted. |
| `--max-attempts` | `3` | ≥ 1 | How many deliveries of a message may fail before it is dead-lettered. |
| `--backoff` | `5` (s) | 0 – 3600 | The wait before the second delivery; it doubles at each further failure. |
| `--wait` | `1000` (ms) | ≥ 1 | How long each read waits for a message: an idle worker notices a stop signal within this time. |

The defaults and bounds are the typed constants of `Waffle\Commons\Console\Queue\WorkerOptions`
(`DEFAULT_MAX_MESSAGES`, `DEFAULT_MEMORY_LIMIT_MB`, `DEFAULT_MAX_ATTEMPTS`, `DEFAULT_BACKOFF_SECONDS`,
`DEFAULT_WAIT_MS`, `MAX_BACKOFF_SECONDS = 3600`, `MAX_MEMORY_LIMIT_MB = 1_048_576`).

**What it does with each message:**

- It hands the message to the handler of its class. Each message continues the trace of the code that dispatched it.
- **Handled:** the message is acknowledged.
- **Failed:** the delivery is released for another attempt after `min(3600, backoff × 2^attempts)` seconds (the
  exponent stops growing after 12 doublings). Once `--max-attempts` deliveries have failed, the message goes to the
  dead-letter store with its envelope and the reason, and is acknowledged.
- **After every message**, in all cases, the worker resets `$resettable` — the application container — as the HTTP
  worker resets it after every request.

**When it stops:**

| Exit code | When |
| :--- | :--- |
| `0` | On SIGTERM or SIGINT, once the message in flight is settled (with `pcntl`); after `--max-messages` messages; at `--memory-limit`. A supervisor then starts a fresh process. |
| `1` | The queue or the dead-letter store failed. The message in flight is delivered again once the queue reclaims it. |

Without `pcntl`, a signal stops the worker at once, and the message in flight is delivered again after the claim idle
time; the worker says so when it starts.

---

## 4. Delivery guarantees

- **At least once.** A handler is idempotent, or de-duplicates on `$envelope->id`.
- **Keep handlers shorter than `claimIdleMs`** (60 s by default). A delivery still running after that is reclaimed
  by another worker and delivered a second time.
- **Deploy the handlers before the code that dispatches a new message class.** A worker that does not know the class
  sets the entry aside as unreadable instead of handling it.
- **One Redis server, or a Sentinel-managed primary — not Redis Cluster.** The release script writes the stream and
  the delayed set in one call; in a cluster the two keys would have to share a hash slot.
- **The worker's Redis pool has no read timeout.** Its reads block for up to `--wait` milliseconds by design, so the
  1 s read timeout the HTTP pools use (RFC-023 §3.2) must not apply to it.

## 5. Worker safety

`RedisStreamsQueue`, `RedisFailedMessageStore`, `EnvelopeCodec`, `StreamsHealthCheck` and `QueueWorkCommand` are
`final readonly` and keep nothing between calls; `wfl igor` reports 0 KO. The worker is itself a long-running process
and resets the container after every message.

## 6. Testing

`waffle-commons/testing` ships `Queue\InMemoryQueue` (a dispatcher and consumer that records every envelope),
`Queue\InMemoryFailedMessageStore` and `Mailer\MailerSpy` — see the [testing reference](testing.md#3-test-doubles).
The driver's own tests run against a real Redis (`WAFFLE_TEST_REDIS_HOST`).
