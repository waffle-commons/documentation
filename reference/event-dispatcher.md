# Event Dispatcher Reference (`waffle-commons/event-dispatcher`)

> **Release:** `v0.1.0-beta2` &nbsp;|&nbsp; *No behavioural changes since Beta-1*
> **PSR Compliance:** PSR-14 (`Psr\EventDispatcher\EventDispatcherInterface`, `ListenerProviderInterface`, `StoppableEventInterface`)

Minimal, strict PSR-14 dispatcher and listener provider. No magic, no auto-wiring — listeners are registered explicitly or discovered via the `#[AsEventListener]` attribute by the kernel factory.

## Surface

| Class | Role |
| :--- | :--- |
| `Waffle\Commons\EventDispatcher\Dispatcher\EventDispatcher` | `final readonly` PSR-14 dispatcher. Iterates `getListenersForEvent()`, invokes each listener, honours `StoppableEventInterface`. |
| `Waffle\Commons\EventDispatcher\Provider\ListenerProvider` | PSR-14 listener provider. Holds the registered listeners and yields them for a given event. |
| `Waffle\Commons\EventDispatcher\Event\AbstractStoppableEvent` | Base class for events that may halt propagation mid-iteration. |
| `Waffle\Commons\EventDispatcher\Attribute\AsEventListener` | `final readonly` attribute marking a class/method as a listener. Discovered by `AppKernelFactory`. |

## `EventDispatcher`

```php
namespace Waffle\Commons\EventDispatcher\Dispatcher;

final readonly class EventDispatcher implements EventDispatcherInterface
{
    public function __construct(
        private ListenerProviderInterface $listenerProvider,
    );

    public function dispatch(object $event): object;
}
```

Behaviour:

1. Asks the provider for listeners interested in `$event` (PSR-14: class-based dispatch).
2. Invokes each listener in priority order.
3. If the event implements `StoppableEventInterface` and `isPropagationStopped()` returns `true`, iteration halts.
4. Returns the (possibly mutated) event.

## `#[AsEventListener]` attribute

```php
namespace Waffle\Commons\EventDispatcher\Attribute;

#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
final readonly class AsEventListener
{
    public function __construct(
        public ?string $event = null,
        public int     $priority = 0,
    );
}
```

| Parameter | Meaning |
| :--- | :--- |
| `event` | FQCN of the event the listener responds to. `null` lets the listener declare via its method signature instead (`__invoke(MyEvent $event)`). |
| `priority` | Higher runs first (descending sort). Default `0`. |

## Discovery (auto-registration)

`AppKernelFactory::discoverAndRegisterListeners()` (in the skeleton) scans `waffle.paths.event_listeners` for PHP files mentioning `AsEventListener`, resolves their FQCNs via `Waffle\Commons\Utils\Service\ClassParser`, and registers them with the `ListenerProvider`. The scan is **once at boot**; the resident worker reuses the registered list across requests.

## `AbstractStoppableEvent`

```php
namespace Waffle\Commons\EventDispatcher\Event;

abstract class AbstractStoppableEvent implements StoppableEventInterface
{
    private bool $propagationStopped = false;

    public function isPropagationStopped(): bool;
    public function stopPropagation(): void;
}
```

Inherit when your event needs an "early veto" pattern (e.g. a `RequestReceivedEvent` whose first listener can deny the entire request).

## Kernel-emitted events (Waffle conventions)

The `waffle-commons/waffle` kernel emits a small set of lifecycle events that listeners commonly subscribe to. The exact set is documented in [core.md](core.md); typical members include:

- `RequestReceivedEvent` — fired before middleware dispatch.
- `ResponseGeneratedEvent` — fired after middleware dispatch, before emission.
- `TerminateEvent` — fired after the response has been emitted to the client (heavy post-response work).

Application listeners typically tag their class with `#[AsEventListener]` and implement `__invoke(SpecificEvent $event)`.

## Worker-mode safety

- `EventDispatcher` is `final readonly`. No per-request state.
- `ListenerProvider` is registered once at boot and reused. Listeners themselves are expected to be stateless (or to reset between requests if they're registered as services on the container with `ResettableInterface`).
- Events are short-lived per request — they're created, dispatched, mutated, and discarded.

## Related

- [core.md](core.md) — kernel lifecycle events.
- [how-to/events.md](../how-to/events.md) — task-oriented event guide.
- [contracts.md](contracts.md) — `EventDispatcherInterface`, `EventListenerInterface`.
