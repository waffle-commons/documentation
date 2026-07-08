# Reactive State Broadcasting (REACTIVE-01)

> **Status:** shipped in `waffle-commons/waffle` (+ contracts) since the beta5 cycle (RFC-018, AXE3).
> **Companion pages:** [broadcast reference](../reference/broadcast.md) · [How to: Broadcast State Changes](../how-to/broadcast-state-changes.md).

This page explains *why* reactive broadcasting looks the way it does. For exact signatures, use the reference.

## The problem: pushing changes without coupling the mutation to I/O

Real-time UIs want a server-side state change — an order moving from `pending` to `shipped` — to reach subscribers the instant it happens. The naive implementation pushes from inside the setter: the property assignment opens a socket, writes to a hub, and blocks until the network acknowledges. Under FrankenPHP's resident worker that is wrong on three counts:

1. **Latency in the hot path** — every assignment now pays a network round-trip; a loop that advances a status three times pays it three times, inside the request the client is waiting on.
2. **Statefulness** — a setter that talks to a transport hides a connection somewhere, and connections are exactly the mutable state the worker-mode audit forbids.
3. **Partial-failure semantics** — if the push fails mid-request, has the state changed or not? Coupling the mutation to the I/O makes the answer ambiguous.

REACTIVE-01 separates the two concerns completely: **the mutation records its intent; a later phase performs the I/O.** The property write is pure and synchronous; the broadcast happens after the response has already been emitted.

## The mechanism: a PHP 8.5 write-hook that records, never sends

The trigger is the `#[Broadcast(channel: …)]` attribute placed on a **`set` write-hook** of a mutable DTO property. The hook does exactly two things — assign the backing value, then append a `MutationRecord` to a request-scoped buffer:

```php
#[Broadcast(channel: 'orders')]
public private(set) string $status {
    set {
        $this->status = $value;
        $this->buffer?->record(new MutationRecord(
            channel: 'orders',
            entityClass: self::class,
            property: 'status',
            value: $value,
        ));
    }
}
```

There is **no I/O in the hook**. `record()` is an in-memory `array` append; it cannot block, cannot fail on a network, and cannot leak a connection. A `MutationRecord` is an immutable value capturing *what changed* (channel, entity FQCN, property name, new value) — never *how to send it*. The attribute itself carries only the channel name; it is metadata that documents intent and drives discovery, not behaviour.

## Why the property cannot be `readonly` — and what that constrains

A PHP 8.5 property hook and `readonly` are mutually exclusive: a hooked property cannot be `readonly`. Broadcasting therefore applies **only to mutable-state DTOs** and never to the framework's `final readonly` value objects (the ones data hydration produces, for instance).

The idiomatic shape is a `final class` whose broadcast property is `public private(set)`: the world can read `$order->status` but only the class itself can write it, so external immutability is preserved through asymmetric visibility rather than `readonly`. The buffer is injected (nullable) at the constructor, which keeps the DTO usable outside a request — in tests, or during hydration — where no buffer exists and the `?->` simply records nothing.

> This is the same constraint documented for hooked DTOs everywhere in Waffle: a hooked DTO is `final class` + `public private(set)`; only unhooked DTOs may be `final readonly`.

## The flush: after the response, off the hot path

The buffer is a single, request-scoped accumulator (`RequestBroadcastBuffer`). Mutations pile up in record order during the request and the controller returns its response *without ever touching the transport*.

The I/O happens in `BroadcastFlushListener`, subscribed to `TerminateEvent` — the lifecycle event that fires **after the response has been emitted to the client** but before the kernel resets request-scoped services. The listener `drain()`s the buffer (returning every record and emptying it in one call) and, if anything was recorded, hands the batch to a `BroadcastTransportInterface`:

```
write-hook → buffer.record()        (during request, no I/O)
        … response is emitted …
TerminateEvent → BroadcastFlushListener
        → buffer.drain()            (read-and-clear)
        → transport.pushBatch(records)   (the only I/O)
```

This ordering is deliberate. The push is the latency-sensitive side effect, so the flush listener is registered **before** the diagnostics/deferred-task listeners — the real-time fan-out runs first. And because the drain happens through the event dispatcher rather than inside the setter, the application can subscribe additional listeners to the same `TerminateEvent` without the DTO knowing anything about them.

## The transport is a port, not an SDK

`BroadcastTransportInterface` is a two-method sink (`push` one record, `pushBatch` a list). It lives in **contracts** and names no delivery technology, so the contracts perimeter stays dependency-free — no Mercure client, no WebSocket library leaks into the core.

The shipped implementation is `SseBroadcastTransport`, which serializes each record into the Server-Sent Events wire format (`event: <channel>\ndata: <json>\n\n`) and writes it to an **injected `Closure(string): void` sink**. That indirection matters twice over:

- **Testability** — the SSE framing is a pure function of the record, so a unit test passes a closure that captures the frame and asserts on the bytes, with no stream involved.
- **Pluggability** — an integrator points the sink at FrankenPHP's native SSE output, a hub fan-out, or (in the demo apps) the container log, without an external SDK. A Mercure or WebSocket bridge, if wanted, ships as its own wrapper component implementing the same interface.

The SSE transport also **sanitizes the channel** before interpolating it into the `event:` line: SSE delimits fields with `\n` and events with `\n\n`, so a channel carrying a CR/LF/NUL could otherwise smuggle a second `data:`/`event:` field — an SSE field-injection. The payload is JSON-encoded (so it cannot carry a bare newline) and a value that cannot be encoded degrades to `{}` rather than throwing, because the flush must never break request teardown.

## Worker safety: the buffer is the only state, and it is recyclable

The whole reactive path is stateless except for the buffer. `RequestBroadcastBuffer` holds one `list<MutationRecord>` and **directly declares `implements ResettableInterface`** — not merely transitively through `BroadcastBufferInterface` — because the shallow worker-safety audit requires the explicit clause on the concrete class. The kernel empties the buffer between worker iterations (and `drain()`/`reset()` both clear it), so a mutation recorded in request A can never bleed into request B. `igor-php` confirms this with zero findings: the attribute, the record, the transport, and the listener are all immutable, and the one stateful object recycles cleanly.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
