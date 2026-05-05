# How to Use Events (PSR-14)

Waffle provides a fully decoupled event system based on the PSR-14 standard.

## Creating an Event

An event in Waffle is a simple PHP object. It can optionally implement `Psr\EventDispatcher\StoppableEventInterface` if you want to allow listeners to stop propagation.

```php
namespace App\Event;

class UserRegisteredEvent
{
    public function __construct(
        public readonly string $email,
        public readonly \DateTimeImmutable $registeredAt,
    ) {}
}
```

## Creating a Listener

You can create a listener by using the `#[AsEventListener]` attribute on a class or a specific method.

### Using Attribute on a Method

This is the recommended way to create a listener. The event type is automatically resolved from the method's first parameter type hint.

```php
use Waffle\Commons\EventDispatcher\Attribute\AsEventListener;
use App\Event\UserRegisteredEvent;

class WelcomeEmailListener
{
    #[AsEventListener(priority: 10)]
    public function onUserRegistered(UserRegisteredEvent $event): void
    {
        // Send welcome email...
    }
}
```

### Using Attribute on a Class

If you put the attribute on the class, Waffle will look for the first public method (excluding `__construct`) to use as the handler. You must specify the event class in the attribute.

```php
#[AsEventListener(event: UserRegisteredEvent::class)]
class LoggerListener
{
    public function handle(UserRegisteredEvent $event): void
    {
        // Log user registration...
    }
}
```

## Dispatching an Event

To dispatch an event, inject the `EventDispatcherInterface` into your service or controller:

```php
use Waffle\Commons\Contracts\EventDispatcher\EventDispatcherInterface;
use App\Event\UserRegisteredEvent;

class RegistrationService
{
    public function __construct(
        private EventDispatcherInterface $dispatcher
    ) {}

    public function register(string $email): void
    {
        // ... registration logic ...

        $this->dispatcher->dispatch(new UserRegisteredEvent($email, new \DateTimeImmutable()));
    }
}
```

## Core Kernel Events

Waffle dispatches several events during the request lifecycle:

| Event Class | Dispatch Time | Purpose |
|-------------|---------------|---------|
| `Waffle\Event\RequestReceivedEvent` | Before Middleware | Modify the incoming request. |
| `Waffle\Event\ResponseGeneratedEvent` | After Middleware | Modify the outgoing response. |
| `Waffle\Event\TerminateEvent` | After Response Emission | Perform heavy background tasks. |
