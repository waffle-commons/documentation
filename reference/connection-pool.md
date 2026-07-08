# Connection Pool Reference (`waffle-commons/data`)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *Memory-resident DB pooling — DBAL-01/DBAL-02 (AXE4), an RFC-022 extension*
> **Requires:** PHP 8.5+, `ext-pdo`. The Redis pool additionally suggests `ext-redis` (injected, never hard-required). Depends only on `waffle-commons/contracts`.

The authoritative API listing for memory-resident connection pooling: the backend-neutral pool contracts, the two shipped pools (`PDOConnectionPool`, `RedisConnectionPool`), the lease handles they dispense, and the failsafe `TransactionIsolationMiddleware`. For the design reasoning (heal-on-lease, reset-rolls-back, connection affinity) see [Explanation: Memory-Resident Connection Pooling](../explanation/connection-pooling.md); the pool also appears in the broader [data reference](data.md).

## Contracts (`Waffle\Commons\Contracts\Data\Connection`)

### `ConnectionPoolInterface`

Backend-neutral pool. Leases a `ConnectionInterface` and reclaims it on release.

```php
public function acquire(): ConnectionInterface;            // ping-before-dispense; reconnects transparently
public function release(ConnectionInterface $connection): void; // return to idle (fail-soft on a foreign lease)
```

Implementations are expected to also implement `Waffle\Commons\Contracts\Service\ResettableInterface` so the kernel can recycle handles between worker requests. `acquire()` throws a `Waffle\Commons\Contracts\Data\Exception\DatabaseExceptionInterface` when no healthy connection can be established.

### `RelationalConnectionPoolInterface extends ConnectionPoolInterface`

Narrows the lease to a typed `PdoConnectionInterface` (covariant return, legal in PHP 8.5) and adds request-scoped affinity for the transaction middleware:

```php
#[\Override]
public function acquire(): PdoConnectionInterface;         // SAME pinned lease while a scope is open
public function beginRequestScope(): PdoConnectionInterface; // pin one connection; idempotent within a scope
public function endRequestScope(): void;                   // unpin + return to idle (no-op when no scope open)
```

### `RedisConnectionPoolInterface extends ConnectionPoolInterface`

Narrows the lease to a typed `RedisConnectionInterface`:

```php
#[\Override]
public function acquire(): RedisConnectionInterface;
```

### Lease handles

A lease is a thin, immutable view over an underlying driver handle. `ConnectionInterface`:

```php
public function kind(): ConnectionKind;  // Pdo | Redis | Stream
public function isAlive(): bool;         // liveness probe; false ⇒ pool culls + re-establishes
public function id(): int;               // stable identity (the underlying handle's spl_object_id)
```

- `PdoConnectionInterface extends ConnectionInterface` adds `public function pdo(): PDO;` — the single typed escape hatch to the live, exception-error-mode handle. The `PDO` import is confined to this one file so the contracts perimeter is preserved.
- `RedisConnectionInterface extends ConnectionInterface` adds `public function client(): object;` — typed `object` so `ext-redis` (`\Redis`/`\Relay`) stays out of the contracts perimeter.

### Enums & diagnostics

- `ConnectionKind: string` — `Pdo 'pdo'`, `Redis 'redis'`, `Stream 'stream'`. Labels a handle for the orphaned-connection tracer.
- `ConnectionTrackerInterface extends ResettableInterface` — the optional DIAG-03 dev-only ledger. `trackOpen(string $id, ConnectionKind $kind): void`, `trackClose(string $id): void`, `openConnections(): array` (`list<array{id: string, kind: ConnectionKind}>`). Production omits the tracker entirely (zero overhead).

## Relational pool — `Waffle\Commons\Data\Connection\PDOConnectionPool`

`final` implementation of `RelationalConnectionPoolInterface` and `ResettableInterface`.

```php
public function __construct(
    Closure $factory,                              // (): PDO — produces a freshly connected handle
    int $maxConnections = 8,                        // hard ceiling on simultaneously borrowed connections
    string $pingQuery = 'SELECT 1',                 // non-empty liveness probe run before dispensing
    ?ConnectionTrackerInterface $tracker = null,    // DIAG-03 tracer; null ⇒ no tracing, zero overhead
);

public function acquire(): PdoConnectionInterface;             // ping-before-dispense; reconnects transparently
public function beginRequestScope(): PdoConnectionInterface;   // pin one lease for the request (DBAL-01)
public function endRequestScope(): void;                       // unpin + release the pinned lease
public function release(ConnectionInterface $connection): void; // return to idle (idempotent, fail-soft)
public function prepare(PdoConnectionInterface $connection, string $sql): PDOStatement; // cached prepared statement
public function reset(): void;                                  // worker reset: rollback stragglers, recycle, clear cache
public function idleCount(): int;
public function activeCount(): int;
```

- **Ping-before-dispense** — every idle handle is probed with `pingQuery` before it leaves the pool (`PDO::query()` then `closeCursor()`); a dropped socket is discarded and a fresh handle is created via `$factory`. The pool forces `PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION` on every connection it creates, so the probe and every query throw `PDOException` rather than failing silently.
- **`beginRequestScope()` / `endRequestScope()`** — pin one connection so every `acquire()` during the request returns the *same* lease (and `release()` of it is a no-op until `endRequestScope()`); idempotent within a scope. This is what makes `TransactionIsolationMiddleware`'s single transaction enclose all downstream repository writes.
- **`reset()`** — the FrankenPHP worker hook: drops the pinned affinity, returns borrowed handles to the idle set, **rolls back any open transaction** (`inTransaction()` → `rollBack()`), and clears the per-connection prepared-statement cache. Sockets stay open so the next request reuses warm connections.
- **`release()`** is fail-soft and idempotent: a lease minted by another pool/kind, or whose handle this pool never issued (DBAL-03), is silently ignored; re-keying on `spl_object_id()` means releasing the same handle twice is a no-op rather than a duplicate idle entry.
- **`prepare()`** caches the compiled `PDOStatement` per `(connection, SQL)`; repeated calls return the same handle, and the cache is wiped on `reset()` so it never crosses a request boundary.
- **Engine note:** Oracle has no table-less `SELECT`, so pass `pingQuery: 'SELECT 1 FROM DUAL'` when pooling `oci` connections (the skeleton/workspace `AppKernelFactory::buildConnectionPool()` does this automatically).

A pool exhaustion (`activeCount() == maxConnections` with no idle handle), a `maxConnections` below `1`, or a failed connection/prepare raises a `Waffle\Commons\Data\Exception\DatabaseException` (`DatabaseExceptionInterface`).

## Redis pool — `Waffle\Commons\Data\Connection\RedisConnectionPool`

`final` implementation of `RedisConnectionPoolInterface` and `ResettableInterface`. The key-value counterpart to `PDOConnectionPool`; the client is typed `object` throughout so the component never assumes `ext-redis` — the concrete client, its `PING`, and its reset are injected as closures.

```php
public function __construct(
    Closure $factory,                              // (): object — produces a freshly connected client
    Closure $healthCheck,                           // (object): bool — liveness probe (PING); true ⇒ live
    int $maxConnections = 8,                        // hard ceiling on borrowed clients
    ?Closure $onReset = null,                        // (object): void — per-handle reset on reset() (e.g. DISCARD/UNWATCH)
    ?ConnectionTrackerInterface $tracker = null,    // DIAG-03 tracer
);

public function acquire(): RedisConnectionInterface; // heal-on-lease
public function release(ConnectionInterface $connection): void;
public function reset(): void;                        // recycle handles + run onReset hook per idle client
public function idleCount(): int;
public function activeCount(): int;
```

- **Heal-on-lease** — every idle client is probed via `$healthCheck` (a `PING`) before dispensing; a dropped client is discarded and replaced transparently.
- **`reset()`** returns borrowed handles to the idle set and, when `onReset` is wired, runs it on each idle client (a `DISCARD`/`UNWATCH`) so an open `MULTI`/`WATCH` never bleeds into the next request. A throwing `onReset` is swallowed (it runs inside `Container::reset()` and must never escape); the broken client is reaped by the next heal-on-lease probe.
- `release()`, `idleCount()`, `activeCount()`, exhaustion, and the `maxConnections < 1` guard behave exactly as on `PDOConnectionPool`.

## Lease implementations (`Waffle\Commons\Data\Connection`)

`final readonly` views minted by the pools; identity is the underlying handle's `spl_object_id()`.

- **`PdoConnection`** — `__construct(PDO $pdo, string $pingQuery = 'SELECT 1')`. `pdo(): PDO`, `kind(): ConnectionKind` (`Pdo`), `id(): int`, `isAlive(): bool` (runs `pingQuery`, `closeCursor()`, catches `PDOException`).
- **`RedisConnection`** — `__construct(object $client, Closure $healthCheck)`. `client(): object`, `kind(): ConnectionKind` (`Redis`), `id(): int`, `isAlive(): bool` (delegates to `$healthCheck`).

## Failsafe transaction middleware — `Waffle\Commons\Data\Middleware\TransactionIsolationMiddleware`

`final readonly` PSR-15 `MiddlewareInterface` (DBAL-02). Wraps every write request in one transaction borrowed from the relational pool.

```php
public function __construct(
    RelationalConnectionPoolInterface $pool,
    ?array $writeMethods = null,   // list<string>; defaults to ['POST', 'PUT', 'PATCH', 'DELETE']
);

#[\Override]
public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface;
```

- Non-write verbs pass straight through to `$handler->handle()`.
- For a write verb: `beginRequestScope()` pins one connection (DBAL-01), `beginTransaction()`, run the handler, then `commit()` on a normal return.
- On **any** uncaught `Throwable`: roll back quietly (`inTransaction()` → `rollBack()`, swallowing a `PDOException` from an already-severed connection) and **rethrow the original error** unchanged.
- A `finally` calls `endRequestScope()`, unpinning and returning the handle every time.
- **Pipeline ordering:** place this INSIDE the error handler (after Security, before the Dispatcher) so a thrown exception unwinds through the middleware — which rolls back and rethrows — before the error handler renders it. A write controller that catches its own exception and returns a `2xx` will COMMIT, by design.
- **Default write methods:** the `DEFAULT_WRITE_METHODS` constant is `['POST', 'PUT', 'PATCH', 'DELETE']`; pass a custom `$writeMethods` list to widen or narrow it. Methods are compared case-insensitively (`strtoupper`).

## Worker-safety contract

`PDOConnectionPool` and `RedisConnectionPool` are the only stateful objects in the relational/key-value path, and their state is explicitly recyclable via `reset()`. The lease handles and the middleware are stateless/immutable (`final readonly`) — the only transactional state lives on the pooled connection, which `reset()` rolls back if the middleware's `finally` was ever skipped. The kernel calls `reset()` between worker iterations, so no per-request state (pinned lease, open transaction, prepared-statement cache, `MULTI`/`WATCH`) leaks across the FrankenPHP worker boundary, and the component passes the `igor-php` worker-mode audit with zero findings.

## Wiring (skeleton / workspace)

The template `AppKernelFactory` builds the pool from `waffle.database.*` and registers it under all three relational pool ids, then adds the middleware:

```php
$pool = AppKernelFactory::buildConnectionPool($config); // PDOConnectionPool, pingQuery adapted per engine
$container->set(ConnectionPoolInterface::class, $pool);
$container->set(RelationalConnectionPoolInterface::class, $pool);
$container->set(PDOConnectionPool::class, $pool);

// inside the error handler, before the dispatcher:
$stack->add(new TransactionIsolationMiddleware($pool));
```

`maxConnections` is a constructor parameter (default `8`) — the templates do not currently expose it as a `waffle.database.*` config key; override it by constructing the pool yourself. See [How to: Configure Connection Pooling](../how-to/configure-connection-pooling.md) for the end-to-end recipe.

***

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
