# Console Reference (`waffle-commons/console`)

> **Release:** `0.1.0-beta3` *(in progress)* &nbsp;|&nbsp; Adds the `db:migrate` command (RFC-022)

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
| `Waffle\Commons\Console\Command\MigrateCommand` | `db:migrate` | Applies pending SQL migrations through `waffle-commons/data`'s `MigrationRunnerInterface`, prints applied versions, then resets the connection pool. See [data.md](data.md) and [How to: Database Migrations](../how-to/database-migrations.md). |
| `Waffle\Commons\Console\Command\MemoryAuditCommand` | `igor:audit` | Streams the monorepo-wide Igor memory-leak & state-mutation audit (`igor.sh`). Thin command depending only on `Waffle\Commons\Contracts\Runtime\AuditRunnerInterface`; the `proc_open` engine lives in `waffle-commons/runtime`. Distinct from `security:audit` (which audits ABAC/CSRF). See [§ `igor:audit`](#igoraudit) below. |
| `Waffle\Commons\Console\Command\DataWarmupCommand` | `data:warmup` | *Added in Beta-3.* Invokes every registered `Waffle\Commons\Contracts\Data\Warmup\DataWarmerInterface`: compiled artifacts (parameterised SQL from SQR trees, routing tables, …) are serialised into PHP cache files and primed into OPcache shared memory, removing compilation and disk I/O from the first live request. Idempotent and strictly CLI-side; applications wire their warmers (e.g. `data`'s `QueryWarmer`) in `bin/waffle`. See [data.md → Warmup](data.md#warmup--wafflecommonsdatawarmupquerywarmer). |

All commands extend `Waffle\Commons\Console\Command\AbstractCommand` which provides shared helpers. `MigrateCommand` and `DataWarmupCommand` depend only on contracts interfaces (`MigrationRunnerInterface` + `ResettableInterface`, `DataWarmerInterface`); the concrete services from `waffle-commons/data` are injected by the application's `bin/waffle`.

## Waffle Maker commands (RFC-020)

Scaffolding generators under `Waffle\Commons\Console\Maker\Command\…`. Every maker extends `AbstractMakerCommand` (PSR-4 namespace discovery from the nearest `composer.json`, atomic file writes, refuses to overwrite without `--force`/`-f`, `--target=DIR` to override the destination — otherwise `src/<Subfolder>` of the current package):

| Command | Generates |
| :--- | :--- |
| `make:controller` | An immutable controller extending `BaseController` (`--route=`, `--priority=`). |
| `make:dto` | A `final` DTO with promoted properties + PHP 8.5 validation `set` hooks (`field:type` positionals). |
| `make:entity` | *Added in Beta-3.* An immutable RFC-022 persistence entity with property-hook validation — the shape `PropertyHookHydrator` hydrates into (`field:type` positionals). |
| `make:repository` | *Added in Beta-3.* A stateless repository composing the worker-safe `SQLRepository` **plus** its `DataMapperInterface` mapper pair (`--table=`, `--identity=`, `field:type` positionals; entity expected in the sibling `Entity` namespace — scaffold it first with `make:entity`). |
| `make:middleware` | A PSR-15 middleware. |
| `make:voter` | An ABAC security voter (fail-closed: `decide()` returns `false` by default). |
| `make:command` | A console command class (`--command-name=`). |
| `make:http-client` | A secure PSR-18 HTTP client wrapper (`--base-uri=`). |
| `make:event-pair` | A coordinated PSR-14 event (`…Event extends AbstractStoppableEvent`) + `#[AsEventListener]` listener pair. |

## `igor:audit`

`MemoryAuditCommand` streams the monorepo-wide Igor audit (`igor.sh`). It is **thin by design**: it depends only on `Waffle\Commons\Contracts\Runtime\AuditRunnerInterface` (plus Input/Output), so `console` gains no dependency edge — the `proc_open` engine (`ProcessAuditRunner`) and its hooked-DTO config (`IgorAuditConfig`) live in `waffle-commons/runtime`, injected by the application's `bin/waffle`. This mirrors how `db:migrate` consumes `MigrationRunnerInterface`.

```php
final readonly class MemoryAuditCommand extends AbstractCommand
{
    public const string NAME = 'igor:audit'; // typed class constant

    public function __construct(
        private AuditRunnerInterface $runner,   // waffle-commons/contracts
        private string $projectRoot,            // directory holding igor.sh
        private bool $localByDefault = false,   // set by bin/waffle when inside the container
    ) {}

    // execute(): SUCCESS (0) audit passed · FAILURE (1) dangerous state · NO_INPUT (66) script missing
}
```

The concrete adapter, its typed constants, and the PHP 8.5 hooked `IgorAuditConfig` are documented with the engine in [runtime.md → `igor:audit` execution engine](runtime.md#igoraudit-execution-engine).

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

From `Waffle\Commons\Contracts\Console\Enum\ExitCode` (`enum ExitCode: int`, mirroring BSD `sysexits(3)`):

- `SUCCESS = 0`
- `FAILURE = 1`
- `USAGE = 64`
- `DATA_ERR = 65`
- `NO_INPUT = 66`
- `NO_PERM = 77`
- `CONFIG = 78`

The enum also exposes `isSuccess(): bool` / `isFailure(): bool`. `run()` returns `USAGE` when invoked with no command name (and prints the help listing), `SUCCESS` on a successful command, and `FAILURE` on any thrown `ConsoleExceptionInterface` or other `Throwable` (the message is written to stderr). Individual commands may return the more specific codes — e.g. `igor:audit` returns `NO_INPUT` when `igor.sh` is missing.

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
