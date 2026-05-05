# Contracts Reference (`waffle-commons/contracts`)

This package contains the interfaces that define the Waffle Architecture. It ensures decoupling between components and provides a strict contract for core services.

## Core Interfaces

### `Waffle\Commons\Contracts\Core\KernelInterface`

The heart of the application, responsible for the request/response lifecycle.

```php
interface KernelInterface
{
    public function boot(): static;
    public function configure(): void;
    public function handle(ServerRequestInterface $request): ResponseInterface;
    public function reset(): void;
}
```

## Event Dispatcher (PSR-14)

Waffle implements a strict PSR-14 compliant event system.

### `Waffle\Commons\Contracts\EventDispatcher\EventDispatcherInterface`

```php
interface EventDispatcherInterface extends \Psr\EventDispatcher\EventDispatcherInterface
{
    public function dispatch(object $event): object;
}
```

### `Waffle\Commons\Contracts\EventDispatcher\ListenerProviderInterface`

```php
interface ListenerProviderInterface
{
    /** @return iterable<callable> */
    public function getListenersForEvent(object $event): iterable;
}
```

## Validation

A contract-first validation system used for DTOs and entity integrity.

### `Waffle\Commons\Contracts\Validation\ValidatorInterface`

```php
interface ValidatorInterface
{
    public function validate(object $value, array $groups = []): ValidationResultInterface;
}
```

### `Waffle\Commons\Contracts\Validation\ValidationResultInterface`

```php
interface ValidationResultInterface
{
    public function isValid(): bool;

    /** @return ViolationInterface[] */
    public function getViolations(): array;
}
```

### `Waffle\Commons\Contracts\Validation\ViolationInterface`

```php
interface ViolationInterface
{
    public function getMessage(): string;
    public function getPropertyPath(): string;
    public function getInvalidValue(): mixed;
}
```

## Logging (PSR-3)

Waffle relies on the standard `Psr\Log\LoggerInterface`. The default implementation `StreamLogger` leverages PHP 8.5 features for configuration:

```php
// implementation detail in StreamLogger.php
public function __construct(
    private(set) readonly string $streamPath = 'php://stderr',
    private(set) readonly string $channel = LogChannel::APP,
    private(set) readonly int $permissions = 0644,
)
```

## Security

### `Waffle\Commons\Contracts\Security\SecurityInterface`

Used by the `SecureContainer` to analyze rules before execution.

```php
interface SecurityInterface
{
    public function analyze(object $object, array $expectations = []): void;
}
```
