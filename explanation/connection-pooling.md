# Memory-Resident Connection Pooling (Beta-5)

> **Status:** shipped in `waffle-commons/data` (+ contracts) for `0.1.0-beta5` (DBAL-01/DBAL-02, AXE4 — an RFC-022 extension).
> **Companion pages:** [Connection Pool reference](../reference/connection-pool.md) · [How to: Configure Connection Pooling](../how-to/configure-connection-pooling.md) · [The Universal Data & Persistence Layer](data-persistence.md).

This page explains *why* the pool exists and *how* it stays safe under a resident worker. For exact signatures, use the [reference](../reference/connection-pool.md).

## The problem: a connection is per-process, but the process never dies

Under FrankenPHP the PHP process loops — one boot, then thousands of request iterations against the same in-memory kernel (see [The Request Lifecycle](lifecycle.md)). A naive `new PDO(...)` per request would throw away the TCP/TLS handshake and the server-side session on every loop, paying connection cost on the hot path; opening one socket at boot and reusing it forever is the opposite failure — the database eventually drops an idle connection and the next query dies with a fatal "server has gone away".

A resident worker needs the middle ground: a small set of connections kept **warm** across iterations, **health-checked** before each use, and **scrubbed** of request-scoped state at the end of every loop so nothing leaks from request A into request B. That is the connection pool.

## Heal-on-lease: a dead socket never reaches the caller

The pool keeps an *idle set* of established handles. When a caller borrows one (`acquire()`), the pool runs a lightweight liveness probe **before** dispensing it — a `SELECT 1` for relational connections, a `PING` health-check closure for Redis. A handle that fails the probe is silently discarded and a fresh one is created via the injected factory; the caller only ever receives a connection that answered a moment ago. This is "ping-before-dispense" / "heal-on-lease": severed sockets self-heal transparently, so the classic "MySQL server has gone away after idle" failure simply cannot surface.

The probe cost is a trivial query per borrow — one round trip for Redis, two under PostgreSQL, where pdo_pgsql runs `PDO::query()` as a prepared statement — which is still far cheaper than the alternative (reconnect-on-every-request), which pays a full handshake.

### The freshness window: probe only an old proof

*Added in `0.1.0-beta7` (CARRY-01).* Under load the probe is not trivial any more: on a pooled read it is half of the database work, and the beta6 benchmark traced Waffle's early saturation knee to it. `PDOConnectionPool`'s `pingFreshnessMs` skips the probe for a connection proven alive less than that many milliseconds ago, so a busy worker probes a connection about once per window instead of once per request.

What counts as a proof matters more than the window. A connection that just connected, or just answered the probe, is proven alive. A connection coming back from a lease is not: the request may have failed on it because the database had dropped it, and the pool never sees the caller's errors. Pools that skip validation for recently *used* connections (PgBouncer's `server_check_delay`, HikariCP's `aliveBypassWindowMs`) can do so because they see those errors and evict broken connections; this pool cannot, so a dead connection would keep itself "fresh" forever. Stamping only proofs bounds the damage instead: when the database drops its connections, a connection proven alive just before fails leases until its proof is older than the window, then the probe catches it and the pool reconnects.

That bound is the trade-off you choose with the window, which is why the library default stays `0` (probe every dispense) and the skeleton opts in with 500 ms, HikariCP's default bypass window. The readiness probe does not rely on the lease's probe any more: `ConnectionPoolHealthCheck` asks the leased connection `isAlive()` itself, so `/readyz` never reports a database it cannot reach. On the beta7 bench, 500 ms halved Engine A's database transactions per read and moved its `dbread` knee from 1200 to at least 1500 rps (one laptop, a relative measurement: `data/benchmarks/CARRY-01-GATE-RESULT.md` in the data repository).

## Bounded by construction

The pool is bounded: `maxConnections` (default `8`) is a hard ceiling on how many connections can be *simultaneously borrowed*. When every connection is in use and a new one is requested, the pool raises a `DatabaseException` ("Connection pool exhausted") rather than opening an unbounded number of sockets — a runaway request pattern can never exhaust the database's connection slots or balloon worker memory. Internally every handle is keyed by its `spl_object_id()`, so the same physical connection can never be double-counted or double-pooled.

## Reset-rolls-back: the worker-iteration boundary

The pool is the **only** stateful object in the data component, and its state is explicitly recyclable. It implements `ResettableInterface`, and the kernel calls `reset()` between worker iterations (the same mechanism that closes PSR-7 streams — see [lifecycle](lifecycle.md)). On `reset()` the pool:

1. returns every borrowed handle to the idle set;
2. **rolls back any still-open transaction** — a request that crashed mid-transaction must never leak a held lock or a half-applied write into the next iteration;
3. clears the per-connection prepared-statement cache;
4. drops any request-scoped connection affinity (see below).

The underlying sockets stay open, so the next request reuses warm connections — but no per-request state crosses the boundary. This is what keeps the component **Igor-clean**: `wfl igor` reports the pool as resettable with zero residual state, and the memory curve stays flat (`ΔM = 0`).

The Redis pool mirrors this: its `reset()` returns handles and runs an optional `onReset` hook (a `DISCARD`/`UNWATCH`) so an open `MULTI`/`WATCH` never bleeds into the next request. A throwing reset hook is swallowed — a client too broken to scrub is reaped by the next heal-on-lease probe — because `reset()` runs inside the container's reset and must never throw.

## Failsafe transactions: `TransactionIsolationMiddleware`

Heal-on-lease and reset keep a single connection clean, but a *write* request is the dangerous case: it can open a transaction, take locks, and then throw before committing. Leaving that transaction dangling would poison the connection for the next worker iteration. The `TransactionIsolationMiddleware` (DBAL-02) closes that gap automatically.

For every write verb (`POST`/`PUT`/`PATCH`/`DELETE` by default) the middleware:

- pins one connection for the whole request via `beginRequestScope()` (connection affinity, DBAL-01), so **every** repository `acquire()` during that request returns the *same* pinned lease and all their writes run inside one transaction;
- `beginTransaction()` before the handler runs;
- `commit()` when the handler returns normally;
- on **any** uncaught throwable, rolls back and rethrows the original error;
- unpins and returns the handle in a `finally` (`endRequestScope()`).

The result is that lock/transaction leakage across worker iterations is structurally impossible: either the write commits cleanly or it is rolled back before the error handler ever renders the failure. (A controller that catches its own exception and returns a `2xx` will commit — by design.) And even if the `finally` were somehow skipped, the pool's `reset()` rolls back the straggler on the way out — defence in depth.

### Why affinity matters

Without `beginRequestScope()`, each repository would `acquire()` an *independent* connection from the pool, and the middleware's transaction — opened on yet another connection — would contain none of their writes. Pinning one connection for the request is what makes the single wrapping transaction actually enclose the downstream repository writes. That is why the middleware belongs **inside** the error handler but **before** the dispatcher in the pipeline.

## Backend-neutral contract, two concretes

Pooling is defined contract-first. `ConnectionPoolInterface` (in **contracts**) leases a backend-neutral `ConnectionInterface`; relational and Redis consumers depend on the narrowed `RelationalConnectionPoolInterface` / `RedisConnectionPoolInterface`, whose `acquire()` is covariantly narrowed to a typed `PdoConnectionInterface` / `RedisConnectionInterface`. The two shipped implementations — `PDOConnectionPool` and `RedisConnectionPool` — never leak a concrete `PDO` or `ext-redis` type through the generic pool surface: the `PDO` import is confined to the one relational lease interface, and the Redis client is typed as `object` with its `PING` and reset injected as closures, so the data component does not hard-depend on `ext-redis`.

This is the same discipline as the rest of the persistence layer (see [The Universal Data & Persistence Layer](data-persistence.md)): the pool is a port, the extension is an injected detail.

***

> *Verified for Waffle Framework 0.1.0-beta6 running on PHP 8.5.6+.*
