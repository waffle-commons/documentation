# Console Reference (`waffle-commons/console`)

> **Release:** `v0.1.0-beta1`

A minimalist, zero-magic CLI runtime (RFC-012). Commands are registered **explicitly** at boot — no auto-discovery — and resolve their dependencies through constructor injection.

## `ConsoleApplication` — exact signature

```php
namespace Waffle\Commons\Console;

final class ConsoleApplication implements ConsoleApplicationInterface
{
    public function __construct(
        private readonly string $name = Constant::DEFAULT_APP_NAME,
        private readonly string $version = '0.0.0',
        private readonly OutputInterface $output = new StreamOutput(),
        ?array $argv = null, // null → reads $_SERVER['argv']
    ) { /* … */ }

    public function getName(): string;
    public function getVersion(): string;
    public function add(CommandInterface $command): void;
    public function has(string $name): bool;
    public function find(string $name): CommandInterface; // throws CommandNotFoundException
    public function all(): array;
    public function run(): int; // returns ExitCode::*->value
}
```

## Built-in commands

| Class | Command name | Purpose |
| :--- | :--- | :--- |
| `Waffle\Commons\Console\Command\CacheClearCommand` | `cache:clear` | Flushes the configured `CacheInterface` backend (route cache, security tokens). |
| `Waffle\Commons\Console\Command\RouteListCommand` | `route:list` | Renders the compiled route table. |
| `Waffle\Commons\Console\Command\SecurityAuditCommand` | `security:audit` | Walks controllers and prints the resolved access ladder (`#[Rule]` / `#[Voter]`). |

All commands extend `Waffle\Commons\Console\Command\AbstractCommand` which provides shared helpers.

## I/O

| Class | Role |
| :--- | :--- |
| `Waffle\Commons\Console\Input\ArgvInput` | `InputInterface` implementation parsing `argv` (positional + named options). |
| `Waffle\Commons\Console\Output\StreamOutput` | Default `OutputInterface` writing to `STDOUT` / `STDERR`. |
| `Waffle\Commons\Console\Output\NullOutput` | Silent `OutputInterface` for tests and quiet runs. |

## Verbosity flags

`run()` interprets the following before dispatching to a command, via `OutputInterface::setVerbosity(Verbosity)`:

- `--quiet` → `Verbosity::QUIET`
- (default) → `Verbosity::NORMAL`
- `-v` / `--verbose` → `Verbosity::VERBOSE`
- `-vv` / `--very-verbose` → `Verbosity::VERY_VERBOSE`
- `-vvv` / `--debug` → `Verbosity::DEBUG`

## Exit codes

From `Waffle\Commons\Contracts\Console\Enum\ExitCode` (`enum ExitCode: int`):

- `SUCCESS = 0`
- `FAILURE = 1`
- `USAGE = 2`

`run()` returns `USAGE` when invoked with no command name (and prints the help listing), `SUCCESS` on a successful command, `FAILURE` on any thrown `ConsoleExceptionInterface` or other `Throwable` (the message is written to stderr).

## Exceptions

- `Waffle\Commons\Console\Exception\ConsoleException` (base) — implements `Waffle\Commons\Contracts\Console\Exception\ConsoleExceptionInterface`.
- `Waffle\Commons\Console\Exception\CommandNotFoundException`.
- `Waffle\Commons\Console\Exception\InvalidArgumentException`.

## Quick example

```php
use Waffle\Commons\Console\ConsoleApplication;
use Waffle\Commons\Console\Command\CacheClearCommand;
use Waffle\Commons\Console\Command\RouteListCommand;

$app = new ConsoleApplication(name: 'Waffle', version: '0.1.0-beta1');

$app->add(new CacheClearCommand($cache));
$app->add(new RouteListCommand($router));

exit($app->run());
```

The built-in `list` command name (no registration required) prints the same usage banner that `run()` shows when no command is provided.
