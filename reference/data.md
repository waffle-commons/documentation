# Data Reference (`waffle-commons/data`)

> **Release:** `v0.1.0-beta3` *(in progress)* &nbsp;|&nbsp; New component (RFC-022)
> **Requires:** PHP 8.5+, `ext-pdo`. Depends only on `waffle-commons/contracts`.

The data & persistence layer, designed for FrankenPHP resident-worker mode. There is **no ORM, no identity map, no change tracking**: a row becomes an immutable value object and nothing more. The component splits cleanly into a warm connection pool, a backend-agnostic query AST, parameterized compilers, a property-hook hydrator, and a stateless migration runner.

## Connection pool — `Waffle\Commons\Data\Connection\PDOConnectionPool`

`final` implementation of `Waffle\Commons\Contracts\Data\Connection\ConnectionPoolInterface` and `Waffle\Commons\Contracts\Service\ResettableInterface`.

```php
public function __construct(
    Closure $factory,                 // () : PDO — produces a freshly connected handle
    int $maxConnections = 8,          // hard ceiling on simultaneously borrowed connections
    string $pingQuery = 'SELECT 1',   // liveness probe run before dispensing
);

public function acquire(): PDO;                              // ping-before-dispense; reconnects transparently
public function release(PDO $connection): void;             // return to the idle set (idempotent)
public function prepare(PDO $connection, string $sql): PDOStatement; // cached prepared statement
public function reset(): void;                              // worker reset: rollback stragglers, clear cache
public function idleCount(): int;
public function activeCount(): int;
```

- **Ping-before-dispense** — every connection is probed with `pingQuery` before it leaves the pool; a dropped socket is discarded and a fresh handle is created via `$factory`, so a stale connection never reaches the caller. The pool forces `PDO::ATTR_ERRMODE => ERRMODE_EXCEPTION` on every connection it creates.
- **`reset()`** is the FrankenPHP worker hook: borrowed handles return to the idle set, any open transaction is rolled back (a request that crashed mid-transaction must not leak a lock into the next), and the per-connection statement cache is cleared. Sockets stay open so the next request reuses warm connections.
- Internal maps are keyed by `spl_object_id()`, so a connection is tracked by identity and can never be double-pooled.

A pool exhaustion (`activeCount() == maxConnections`) or a failed connection raises a `DatabaseException`.

## Query AST (SQR)

The query model is pure representation — it holds no backend knowledge and is consumed by a compiler.

### `Waffle\Commons\Data\Query\Query`

Immutable, copy-on-write builder. Every method returns a new instance.

```php
public static function select(string ...$fields): self;
public function from(string $source): self;
public function where(Comparison ...$criteria): self;       // ANDed together
public function orderBy(string $field, Direction $direction = Direction::Ascending): self;
public function limit(int $limit): self;
public function offset(int $offset): self;
```

### `Waffle\Commons\Data\Query\Criteria`

Static factory producing `Comparison` predicates (centralises field validation; scalar values are typed precisely, no `mixed`):

| Factory | Operator |
| :--- | :--- |
| `Criteria::eq($field, $value)` | `=` |
| `Criteria::neq($field, $value)` | `<>` |
| `Criteria::gt` / `gte` / `lt` / `lte` | `>` / `>=` / `<` / `<=` |
| `Criteria::like($field, $value)` | `LIKE` |
| `Criteria::in($field, $values)` / `notIn(...)` | `IN` / `NOT IN` (non-empty value list) |

A blank field name, or an empty `in`/`notIn` value set, throws `\InvalidArgumentException`.

### Enums

- `Waffle\Commons\Data\Query\Operator: string` — `Equal '='`, `NotEqual '<>'`, `GreaterThan '>'`, `GreaterThanOrEqual '>='`, `LessThan '<'`, `LessThanOrEqual '<='`, `In 'IN'`, `NotIn 'NOT IN'`, `Like 'LIKE'`; `isSetOperator(): bool` distinguishes `IN`/`NOT IN`.
- `Waffle\Commons\Data\Query\Direction: string` — `Ascending 'ASC'`, `Descending 'DESC'`.

## SQL compiler — `Waffle\Commons\Data\Compiler\SQLCompiler`

```php
public function __construct(private SQLDialect $dialect = SQLDialect::MySQL);
public function compile(Query $query): CompiledQuery; // throws \InvalidArgumentException
```

Produces a `Waffle\Commons\Data\Compiler\CompiledQuery` (`final readonly`):

```php
public string $sql;        // parameterized: '?' placeholders only
public array $parameters;  // positional bound values, in order
```

All values are bound as positional `?` placeholders — **no value is ever interpolated into the SQL string** (OWASP A03). Identifiers are quoted per dialect. `compile()` throws `\InvalidArgumentException` when the query has no source table or a set predicate carries no values.

`Waffle\Commons\Data\Compiler\SQLDialect` (`enum`) selects identifier quoting and pagination grammar:

| Case | Identifier quoting | Pagination |
| :--- | :--- | :--- |
| `SQLDialect::MySQL` | `` `ident` `` | `LIMIT … OFFSET …` |
| `SQLDialect::SQLite` | `` `ident` `` | `LIMIT … OFFSET …` |
| `SQLDialect::MSSQL` | `[ident]` | `OFFSET … ROWS FETCH NEXT … ROWS ONLY` |

`SQLDialect::quoteIdentifier(string): string` quotes dotted identifiers segment-by-segment (and escapes embedded quote characters); `SQLDialect::paginate(?int $limit, ?int $offset): string`.

## Firestore compiler — `Waffle\Commons\Data\Compiler\FirestoreCompiler`

```php
public function compile(Query $query, FirestoreScope $scope): CompiledFirestoreQuery;
```

Targets Cloud Firestore's structured query shape with **mandatory path isolation**:

```php
FirestoreScope::public(string $appId, string $collection): self;            // /artifacts/{appId}/public/data/{collection}
FirestoreScope::private(string $appId, string $userId, string $collection): self; // /artifacts/{appId}/users/{userId}/{collection}
```

Path segments are validated and URL-encoded; illegal segments are rejected. Only equality predicates are pushed server-side; range comparisons and ordering set `requiresInMemoryFilter = true` so the caller knows to post-filter client-side. The result `Waffle\Commons\Data\Compiler\CompiledFirestoreQuery` (`final readonly`) exposes:

```php
public string $path;
public array  $filters;
public array  $orderings;
public ?int   $limit;
public bool   $requiresInMemoryFilter;
public function toJson(): string;
```

## Hydrator — `Waffle\Commons\Data\Hydrator\PropertyHookHydrator`

```php
/** @template T of object  @param class-string<T> $target */
public function __construct(string $target);

/** @return T  @throws ValidationExceptionInterface */
public function hydrate(array $row): object;
```

Maps a raw backend row onto an immutable DTO by invoking its constructor with named arguments. Integrity is enforced in two layers:

1. **Pre-construction type check** — each column is validated against the matching constructor parameter's declared scalar type (with the one lossless widening PHP performs, `int → float`) *before* construction, so a native `\TypeError` can never escape.
2. **Property Hooks** — the DTO's own PHP 8.5 `set` hooks then reject semantically invalid values.

Either layer failing yields a `Waffle\Commons\Contracts\Exception\Validation\ValidationExceptionInterface` — a poisoned persisted record is rejected exactly like poisoned request input (RFC 7807 `422`). No identity map, no proxying.

## Migration runner — `Waffle\Commons\Data\Migration\MigrationRunner`

`final` implementation of `Waffle\Commons\Contracts\Data\Migration\MigrationRunnerInterface`.

```php
public function __construct(ConnectionPoolInterface $pool, ConfigInterface $config);

/**
 * @param (Closure(string): void)|null $onApplied  per-version progress callback
 * @return list<string>                             versions applied this run ([] when up to date)
 * @throws DatabaseExceptionInterface
 */
public function run(?Closure $onApplied = null): array;
```

- Borrows one connection from the pool, provisions `waffle_migrations (version VARCHAR(255) PRIMARY KEY, applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)` if absent, then applies every `*.sql` file under the configured directory (`waffle.database.migrations_path`) **in lexicographic version order** whose version (the filename without `.sql`) is not yet recorded.
- Each migration runs in **its own transaction**: on failure the transaction is rolled back, the run aborts, and a `DatabaseException` carrying the offending version is thrown. Already-applied migrations are skipped, so a re-run is a no-op.
- Discovery and execution happen only on the explicit CLI path — never during an HTTP request (zero-magic registry).

**Engine caveat:** per-migration transactions are fully atomic on engines with transactional DDL (SQLite, PostgreSQL). On MySQL, DDL (`CREATE`/`ALTER TABLE`) triggers an implicit commit, so a failed DDL step cannot be rolled back — keep one schema change per migration file on MySQL.

Surfaced to operators as `bin/waffle db:migrate` (see [console.md](console.md)). For the end-to-end workflow, see [How to: Database Migrations](../how-to/database-migrations.md).

## Exceptions

Both implement their respective contracts interface so callers catch persistence/validation failures without coupling to a driver:

- `Waffle\Commons\Data\Exception\DatabaseException` → `Waffle\Commons\Contracts\Data\Exception\DatabaseExceptionInterface`. `getSqlState(): ?string` lifts the ANSI `SQLSTATE` from a `PDOException` (`null` for non-relational backends). `DatabaseException::fromThrowable(\Throwable, ?string)` wraps any backend error while preserving the original as `previous`.
- `Waffle\Commons\Data\Exception\ValidationException` → `Waffle\Commons\Contracts\Exception\Validation\ValidationExceptionInterface`. `getField(): ?string` names the offending field.

## Worker-safety contract

`PDOConnectionPool` is the only stateful object, and its state is explicitly recyclable via `reset()`. Compilers, the hydrator, and the query AST are stateless / immutable. The kernel calls `reset()` between worker iterations (and the `db:migrate` command resets the pool on the way out), so no per-request state leaks across the FrankenPHP worker boundary.

## Quick example

```php
use Waffle\Commons\Data\Query\Query;
use Waffle\Commons\Data\Query\Criteria;
use Waffle\Commons\Data\Compiler\SQLCompiler;
use Waffle\Commons\Data\Compiler\SQLDialect;

$query = Query::select('id', 'email')
    ->from('users')
    ->where(Criteria::eq('status', 'active'))
    ->limit(10);

$compiled = new SQLCompiler(SQLDialect::SQLite)->compile($query);

$connection = $pool->acquire();
$statement = $pool->prepare($connection, $compiled->sql);
$statement->execute($compiled->parameters);
// ... map rows via PropertyHookHydrator ...
$pool->release($connection);
```
