# How to Configure Connection Pooling (Beta-5)

Waffle keeps database connections warm across FrankenPHP worker iterations with a memory-resident **connection pool** (`waffle-commons/data`, DBAL-01/DBAL-02). The pool health-checks each connection before handing it out ("ping-before-dispense"), rolls back stragglers between requests, and — via the `TransactionIsolationMiddleware` — wraps every write request in a single transaction that commits on success and rolls back on any error. This recipe wires it end-to-end.

For the design reasoning see [Explanation: Memory-Resident Connection Pooling](../explanation/connection-pooling.md); for the full API see the [Connection Pool reference](../reference/connection-pool.md).

> The skeleton and workspace templates already include everything below. Follow this guide when assembling a kernel by hand, or to tune the pool.

## 1. Configure database access

The pool is built from the same `waffle.database.*` block used by migrations (see [How to Run Database Migrations](database-migrations.md)). Add it to `config/app.yaml`, with credentials from the environment:

```yaml
waffle:
  database:
    driver: 'pgsql'                 # pgsql (default) | mysql/mariadb | sqlite | sqlsrv | oci
    host: '%env(DB_HOST)%'
    port: '%env(DB_PORT)%'
    database: '%env(DB_NAME)%'
    username: '%env(DB_USER)%'
    password: '%env(DB_PASSWORD)%'
    charset: 'utf8'                 # ignored by pgsql; used by mysql/oci
```

`AppKernelFactory::buildConnectionPool()` reads this block, builds the engine-specific PDO DSN, and adapts the liveness probe per engine (`SELECT 1`, or `SELECT 1 FROM DUAL` on Oracle).

## 2. Build the pool and register it

`buildConnectionPool()` returns a `PDOConnectionPool` whose factory opens a fresh `PDO` (in `ERRMODE_EXCEPTION`) only when the pool needs one. Register the single instance under all three relational pool ids so every consumer resolves the same warm pool:

```php
use Waffle\Commons\Contracts\Data\Connection\ConnectionPoolInterface;
use Waffle\Commons\Contracts\Data\Connection\RelationalConnectionPoolInterface;
use Waffle\Commons\Data\Connection\PDOConnectionPool;

$pool = AppKernelFactory::buildConnectionPool($config); // ping-before-dispense baked in

$container->set(ConnectionPoolInterface::class, $pool);
$container->set(RelationalConnectionPoolInterface::class, $pool);
$container->set(PDOConnectionPool::class, $pool);
```

Because the pool implements `ResettableInterface`, the kernel resets it between worker iterations automatically — rolling back any open transaction and recycling handles. You do not call `reset()` yourself.

## 3. Add the failsafe transaction middleware

Register `TransactionIsolationMiddleware` so every write verb runs inside one transaction. **Placement matters:** add it *inside* the error handler and *before* the dispatcher, so a thrown exception unwinds through it (rollback + rethrow) before the error handler renders the response.

```php
use Waffle\Commons\Contracts\Data\Connection\RelationalConnectionPoolInterface;
use Waffle\Commons\Data\Middleware\TransactionIsolationMiddleware;

/** @var RelationalConnectionPoolInterface $pool */
$pool = $container->get(RelationalConnectionPoolInterface::class);
$stack->add(new TransactionIsolationMiddleware($pool));
```

By default it wraps `POST`, `PUT`, `PATCH`, and `DELETE`. To change which verbs are transactional, pass a second argument:

```php
// e.g. also wrap a custom verb, or restrict to POST only:
$stack->add(new TransactionIsolationMiddleware($pool, ['POST']));
```

## 4. Borrow a connection in a controller

Inside a transactional write request, every `acquire()` returns the **same pinned connection** the middleware opened the transaction on (connection affinity), so your writes are part of that transaction. Always release the lease in a `finally`:

```php
use Waffle\Commons\Contracts\Data\Connection\RelationalConnectionPoolInterface;

public function createUser(RelationalConnectionPoolInterface $pool): ResponseInterface
{
    $lease = $pool->acquire();   // same pinned lease during a write request
    $pdo   = $lease->pdo();      // typed PDO access via the relational lease

    try {
        $statement = $pool->prepare($lease, 'INSERT INTO users (id, email) VALUES (?, ?)');
        $statement->execute(['u-1', 'a@example.com']);
        // no manual commit: the middleware commits when this handler returns 2xx,
        // and rolls back if you throw.
    } finally {
        $pool->release($lease);  // no-op on the pinned lease until endRequestScope()
    }

    // ... build and return the response
}
```

Repositories follow the same pattern internally — `SQLRepository` borrows, uses `pdo()`, and releases in a `finally`, so a streamed result always returns its connection even if the consumer abandons the generator.

> If a write throws, the middleware rolls back and **rethrows the original error** — your error handler still renders the real failure (RFC 7807), not a transaction artifact. A controller that catches its own exception and returns a `2xx` will COMMIT, by design.

## 5. (Optional) Tune the pool size

`maxConnections` is the hard ceiling on simultaneously borrowed connections (default `8`). It is a constructor parameter — not currently a `waffle.database.*` key — so to change it, construct the pool yourself instead of calling `buildConnectionPool()`:

```php
use Waffle\Commons\Data\Connection\PDOConnectionPool;
use PDO;

$pool = new PDOConnectionPool(
    factory: static fn(): PDO => new PDO($dsn, $user, $pass, [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    ]),
    maxConnections: 16,
    pingQuery: 'SELECT 1',           // 'SELECT 1 FROM DUAL' on Oracle
);
```

When all `maxConnections` are in use and no idle handle is available, the next `acquire()` raises a `DatabaseException` ("Connection pool exhausted") rather than opening an unbounded number of sockets — keep `maxConnections` at or below your database's per-worker connection budget.

## 6. (Optional) Pool Redis the same way

The Redis pool (`RedisConnectionPool`) is the key-value counterpart. Because the data component never hard-depends on `ext-redis`, you inject the client factory, the `PING` health-check, and an optional reset hook (`DISCARD`/`UNWATCH`) as closures:

```php
use Waffle\Commons\Data\Connection\RedisConnectionPool;
use Redis;

$redisPool = new RedisConnectionPool(
    factory: static function (): Redis {
        $client = new Redis();
        $client->connect($host, $port);

        return $client;
    },
    healthCheck: static fn(Redis $c): bool => $c->ping() !== false,
    maxConnections: 8,
    onReset: static function (Redis $c): void {            // scrub any open MULTI/WATCH between requests
        $c->discard();
    },
);
```

`acquire()` returns a `RedisConnectionInterface`; call `client()` for the concrete handle. Register it under `RedisConnectionPoolInterface` and the kernel will reset it between iterations.

## How it works

- The pool keeps a bounded *idle set* of established handles and probes each one (`SELECT 1` / `PING`) before dispensing — a dead socket is discarded and replaced, so a stale connection never reaches your code.
- Between worker iterations the kernel calls `reset()`: borrowed handles return to idle, any open transaction is rolled back, the prepared-statement cache is cleared, and request-scoped affinity is dropped — so nothing leaks into the next request and `wfl igor` stays 0 KO.
- The `TransactionIsolationMiddleware` adds the safety net for writes: pin one connection, one transaction, commit on success, rollback + rethrow on failure.

See the [Connection Pool reference](../reference/connection-pool.md) for the full API and the [data reference](../reference/data.md) for how repositories consume the pool.

***

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
