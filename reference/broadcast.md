# Broadcast Reference (`waffle-commons/waffle` + contracts)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *Reactive state broadcasting via PHP 8.5 property write-hooks (REACTIVE-01, RFC-018, AXE3)*
> **Requires:** PHP 8.5+. The contracts vocabulary lives in `waffle-commons/contracts`; the concretes ship in `waffle-commons/waffle`. No external transport SDK is pulled into core.

Reactive broadcasting turns a flagged property mutation into a real-time push without performing I/O at mutation time. A `#[Broadcast]` `set` write-hook enqueues an immutable `MutationRecord` into a request-scoped buffer; a finish-request listener drains the buffer and publishes the batch over a transport (SSE by default), after the response cycle. Hooked properties cannot be `readonly` in PHP 8.5, so broadcasting applies to mutable DTOs (`final class` + `public private(set)`) only — never to `final readonly` value objects.

For the architectural reasoning (why no I/O in the hook, why a finish-request flush), see [Explanation: Reactive State Broadcasting](../explanation/reactive-broadcast.md).

## Attribute — `Waffle\Commons\Contracts\Reactive\Attribute\Broadcast`

`final readonly` attribute, target **`TARGET_PROPERTY`**. Marks a mutable property whose changes are broadcast in real time.

```php
#[Attribute(Attribute::TARGET_PROPERTY)]
final readonly class Broadcast
{
    public function __construct(
        public string $channel,   // logical channel, e.g. 'orders' or 'user.42'
    ) {}
}
```

- The attribute is **metadata**: it documents the channel a property's mutations publish on. The actual enqueue is performed by the `set` hook body (the attribute does not intercept the write itself).
- Because a hooked property cannot be `readonly`, the attribute is only valid on a mutable-state DTO property — the idiomatic shape is `public private(set)` on a `final class`.

## Mutation record — `Waffle\Commons\Contracts\Reactive\MutationRecord`

`final readonly` value object: one broadcast-flagged property mutation, captured without any I/O.

```php
final readonly class MutationRecord
{
    public function __construct(
        public string $channel,       // logical channel the mutation publishes on
        public string $entityClass,   // FQCN of the DTO/Entity that changed
        public string $property,      // name of the mutated property
        public mixed  $value,         // the new value (serialized at flush time)
    ) {}
}
```

Constructed inside the write-hook and appended to the buffer; serialized later by the transport. `$value` is `mixed` because a broadcast property may carry any scalar/array; encoding happens at flush time, not at record time.

## Buffer contract — `Waffle\Commons\Contracts\Reactive\BroadcastBufferInterface`

Request-scoped accumulator of `MutationRecord`s. Extends `Waffle\Commons\Contracts\Service\ResettableInterface`, so the kernel empties it between worker iterations.

```php
interface BroadcastBufferInterface extends ResettableInterface
{
    public function record(MutationRecord $record): void;  // append (called from a write-hook; no I/O)

    /** @return list<MutationRecord> */
    public function drain(): array;                        // return every buffered record in order, and clear
}
```

- `record()` is an in-memory append — it must never perform I/O.
- `drain()` is read-and-clear: it returns the records in record order and empties the buffer in the same call, so a second drain returns `[]`.
- `reset()` (inherited from `ResettableInterface`) is the worker hook that discards any un-drained records between requests.

### Concrete — `Waffle\Reactive\RequestBroadcastBuffer`

`final` implementation of `BroadcastBufferInterface` **and** `ResettableInterface` (declared directly).

```php
final class RequestBroadcastBuffer implements BroadcastBufferInterface, ResettableInterface
{
    public function record(MutationRecord $record): void;  // appends to an internal list<MutationRecord>
    public function drain(): array;                        // returns the list, resets it to []
    public function reset(): void;                          // resets the list to []
}
```

- Holds exactly one piece of request-scoped state: a `list<MutationRecord>`.
- It re-declares `implements ResettableInterface` **directly** (not only transitively via `BroadcastBufferInterface`) because the shallow worker-safety audit (`wfl igor`) requires the explicit clause on the concrete class.
- Registered in the container under `BroadcastBufferInterface::class` so controllers and mutable DTOs receive the same per-request instance; `Container::reset()` empties it each worker loop.

## Transport contract — `Waffle\Commons\Contracts\Reactive\BroadcastTransportInterface`

Sink that publishes drained records to real-time subscribers. Invoked by the finish-request flush, **never** from inside a write-hook. It names no delivery technology, so the contracts perimeter stays SDK-free.

```php
interface BroadcastTransportInterface
{
    public function push(MutationRecord $record): void;    // publish one record to its channel

    /** @param list<MutationRecord> $records */
    public function pushBatch(array $records): void;        // publish a batch, in record order
}
```

### Concrete — `Waffle\Reactive\Sse\SseBroadcastTransport`

`final readonly` Server-Sent Events transport.

```php
final readonly class SseBroadcastTransport implements BroadcastTransportInterface
{
    /** @param Closure(string): void $sink  Receives each serialized SSE frame. */
    public function __construct(private Closure $sink);

    public function push(MutationRecord $record): void;       // frames the record, writes it to the sink
    public function pushBatch(array $records): void;          // push() each record, in order
}
```

- **Wire format.** Each record is framed as `event: <channel>\ndata: <json>\n\n`. The `data:` payload is `json_encode([...])` of `{channel, entity, property, value}` (note the key is `entity`, mapped from `MutationRecord::$entityClass`).
- **Injected sink.** The framing is written to a `Closure(string): void`, so it is unit-testable (capture the frame) and pluggable (point it at FrankenPHP's native SSE output, a hub fan-out, or — in the demo apps — a logger) without an external SDK.
- **Channel sanitization.** The channel is interpolated raw into the `event:` line, so every CR/LF and C0 control byte (including NUL) is stripped first (`preg_replace('/[\x00-\x1F\x7F]/', '', $channel)`) to prevent SSE field/event injection. The JSON payload cannot carry a bare newline, so only the channel is guarded.
- **Fail-soft encoding.** A value that cannot be JSON-encoded (e.g. a resource) degrades to `{}` rather than throwing — the flush must never break request teardown.
- **Stateless:** holds only its sink and writes synchronously when the flush invokes it.

## Finish-request listener — `Waffle\Event\Listener\BroadcastFlushListener`

`final readonly` listener that flushes the buffer after the response.

```php
final readonly class BroadcastFlushListener
{
    public function __construct(
        private BroadcastBufferInterface $buffer,
        private BroadcastTransportInterface $transport,
    ) {}

    public function __invoke(TerminateEvent $event): void;
}
```

- Subscribed to `Waffle\Event\TerminateEvent`, which fires **after** the response has been emitted to the client but **before** the kernel resets request-scoped services.
- On invocation it `drain()`s the buffer; if the result is `[]` it returns immediately (no transport call), otherwise it calls `transport->pushBatch($records)`.
- Register it **before** latency-tolerant terminate listeners (diagnostics, deferred tasks) so the real-time push runs first. The buffer also resets itself defensively, so a skipped terminate (non-terminable kernel, early exit) never leaks mutations into the next request.

## Wiring

There is no auto-discovery — the buffer, transport, and listener are wired explicitly in the application's kernel factory:

```php
use Waffle\Commons\Contracts\Reactive\BroadcastBufferInterface;
use Waffle\Event\Listener\BroadcastFlushListener;
use Waffle\Event\TerminateEvent;
use Waffle\Reactive\RequestBroadcastBuffer;
use Waffle\Reactive\Sse\SseBroadcastTransport;

// 1. Request-scoped buffer, registered under its contract (resettable ⇒ emptied each worker loop).
$broadcastBuffer = new RequestBroadcastBuffer();
$container->set(BroadcastBufferInterface::class, $broadcastBuffer);

// 2. Transport: SSE frames written to an injected sink.
$broadcastTransport = new SseBroadcastTransport(sink: static function (string $frame): void {
    // Production: FrankenPHP SSE output / hub fan-out. Demo: write the frame to a log.
});

// 3. Flush on TerminateEvent — registered BEFORE other terminate listeners.
$listenerProvider->addListener(
    TerminateEvent::class,
    new BroadcastFlushListener($broadcastBuffer, $broadcastTransport),
);
```

## Configuration & environment

Reactive broadcasting ships **no dedicated config keys or environment variables**. The behaviour is fully expressed in code:

- the **channel** is the `Broadcast` attribute / `MutationRecord` argument on the DTO (a string literal such as `'orders'`);
- the **transport target** is the injected `Closure` sink on `SseBroadcastTransport`;
- the **listener ordering** is the registration order against `TerminateEvent`.

The demo apps point the SSE sink at the container log, so the broadcast frames surface in `docker logs` (acting as a stand-in collector). Swap the sink for a real SSE/hub writer to deliver to browsers.

## Worker-safety contract

The attribute, the mutation record, both transports' logic, and the flush listener are immutable. `RequestBroadcastBuffer` is the only stateful object and is explicitly recyclable via `reset()` / `drain()`; the kernel resets it between iterations. The component passes the `igor-php` worker-mode audit with zero findings — no per-request mutation bleeds across the FrankenPHP worker boundary.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
