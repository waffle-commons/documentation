# How to Broadcast State Changes in Real Time

Waffle ships reactive state broadcasting (REACTIVE-01, RFC-018) in the `waffle-commons/waffle` component. You flag a mutable DTO property with `#[Broadcast(channel: …)]`; its PHP 8.5 `set` write-hook enqueues a `MutationRecord` into a request-scoped buffer with **no I/O**, and a finish-request listener drains the buffer and pushes the mutations over a transport (Server-Sent Events by default) **after** the response has been emitted. This recipe wires the pieces and broadcasts your first mutation.

## 1. Wire the buffer, transport, and flush listener

The reactive path is wired explicitly in your kernel factory — there is no auto-discovery. Register the request-scoped buffer under its contract, build an SSE transport with a sink, and subscribe the flush listener to `TerminateEvent`:

```php
use Waffle\Commons\Contracts\Reactive\BroadcastBufferInterface;
use Waffle\Event\Listener\BroadcastFlushListener;
use Waffle\Event\TerminateEvent;
use Waffle\Reactive\RequestBroadcastBuffer;
use Waffle\Reactive\Sse\SseBroadcastTransport;

// Request-scoped buffer — resettable, so the kernel empties it each worker loop.
$broadcastBuffer = new RequestBroadcastBuffer();
$container->set(BroadcastBufferInterface::class, $broadcastBuffer);

// Transport: each SSE frame is handed to the injected sink. Point it at your real
// SSE output / hub in production; for a first run, log it so you can see the frames.
$broadcastTransport = new SseBroadcastTransport(sink: static function (string $frame) use ($logger): void {
    $logger->info('SSE frame: {frame}', ['frame' => $frame]);
});

// Flush after the response is emitted. Register it BEFORE other terminate listeners
// (diagnostics, deferred tasks) so the latency-sensitive push runs first.
$listenerProvider->addListener(
    TerminateEvent::class,
    new BroadcastFlushListener($broadcastBuffer, $broadcastTransport),
);
```

> Registering the buffer under `BroadcastBufferInterface::class` is what lets your controllers and DTOs receive the same per-request instance through container resolution.

## 2. Write a mutable DTO with a `#[Broadcast]` property

A broadcast property uses a `set` write-hook, and **a hooked property cannot be `readonly` in PHP 8.5**. So the DTO is a `final class` (never `final readonly`) and external immutability comes from `public private(set)` — the world reads the value, only the class writes it. Inject the buffer (nullable) so the DTO still works in tests and during hydration, where no buffer exists:

```php
use Waffle\Commons\Contracts\Reactive\Attribute\Broadcast;
use Waffle\Commons\Contracts\Reactive\BroadcastBufferInterface;
use Waffle\Commons\Contracts\Reactive\MutationRecord;

final class OrderStatus
{
    #[Broadcast(channel: 'orders')]
    public private(set) string $status {
        set {
            $this->status = $value;
            // No I/O here — just record the mutation into the request-scoped buffer.
            $this->buffer?->record(new MutationRecord(
                channel: 'orders',
                entityClass: self::class,
                property: 'status',
                value: $value,
            ));
        }
    }

    public function __construct(
        public readonly string $orderId,
        string $status,
        private readonly ?BroadcastBufferInterface $buffer = null,
    ) {
        $this->status = $status;
    }

    public function transitionTo(string $status): void
    {
        $this->status = $status;  // goes through the write-hook → records a mutation
    }
}
```

> **Skip the seed if you only want transitions.** Because the constructor assignment also runs the hook, you can suppress broadcasting the initial value by seeding `$status` *before* the buffer is attached (assign `$this->status` first, set `$this->buffer = $buffer` afterwards). Then only later transitions broadcast, not the starting state.

## 3. Mutate the DTO from a controller

Resolve the buffer in the action, attach it to the DTO, and mutate. Every assignment records a mutation; the controller returns its response **without touching the transport**:

```php
use Psr\Http\Message\ResponseInterface;
use Waffle\Commons\Contracts\Reactive\BroadcastBufferInterface;

public function order(BroadcastBufferInterface $buffer): ResponseInterface
{
    $order = new OrderStatus(orderId: 'ORD-4242', status: 'pending', buffer: $buffer);

    $order->transitionTo('paid');
    $order->transitionTo('shipped');
    $order->transitionTo('delivered');

    // Three mutations are now buffered. The SSE flush happens AFTER this response,
    // on TerminateEvent — nothing in this method performs network I/O.
    return $this->jsonResponse(data: [
        'reference'    => $order->orderId,
        'final_status' => $order->status,
    ]);
}
```

## 4. Observe the broadcast

When the response has been emitted, `TerminateEvent` fires, `BroadcastFlushListener` drains the buffer, and `SseBroadcastTransport` frames each record and writes it to your sink. With the logging sink from step 1, each mutation appears as one SSE frame:

```
event: orders
data: {"channel":"orders","entity":"App\\Dto\\OrderStatus","property":"status","value":"paid"}

```

The frame is the SSE wire format (`event: <channel>\ndata: <json>\n\n`). In a real deployment the sink writes these frames to FrankenPHP's SSE output or your hub, and subscribed browsers receive them live.

## How it works

- The `#[Broadcast]` `set` hook only **records** — an in-memory append to `RequestBroadcastBuffer`. It performs no I/O, cannot block, and cannot leak a connection.
- `BroadcastFlushListener` is subscribed to `TerminateEvent`, which fires *after* the response is emitted but *before* request-scoped services reset. It `drain()`s the buffer (read-and-clear) and, only if non-empty, calls `transport->pushBatch(...)`.
- `SseBroadcastTransport` sanitizes the channel (strips CR/LF/NUL to prevent SSE field injection) and JSON-encodes the payload; an unencodable value degrades to `{}` so teardown never breaks.
- `RequestBroadcastBuffer` is the only request-scoped state and implements `ResettableInterface`; the kernel empties it each worker loop, so mutations never bleed across requests (`wfl igor` reports 0 KO).

## Swapping the transport

`SseBroadcastTransport` is just one implementation of `BroadcastTransportInterface`. To deliver over Mercure, a WebSocket gateway, or any other channel, implement that two-method interface (`push` / `pushBatch`) in your own wrapper component — keeping the external SDK out of the contracts perimeter — and inject it into `BroadcastFlushListener` instead. The DTO, the attribute, and the buffer are unchanged.

See the [broadcast reference](../reference/broadcast.md) for the full API (attribute, record, buffer, transport, listener) and [Explanation: Reactive State Broadcasting](../explanation/reactive-broadcast.md) for the design rationale. For the `TerminateEvent` lifecycle hook this builds on, see [How to Use Events](events.md).

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
