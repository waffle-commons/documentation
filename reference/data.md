# Data Reference (`waffle-commons/data`)

> **Release:** `0.1.0-beta3` *(in progress)* &nbsp;|&nbsp; New component (RFC-022)
> **Requires:** PHP 8.5+, `ext-pdo`, `psr/http-client`, `psr/http-factory`, `psr/http-message`. Depends only on `waffle-commons/contracts`.
> **Suggests:** `ext-redis` (live key-value driver), `ext-mongodb` (live document driver).

The Universal Data & Persistence Layer, designed for FrankenPHP resident-worker mode. There is **no ORM, no identity map, no change tracking**: a row becomes an immutable value object and nothing more. The component splits cleanly into a warm connection pool, a backend-agnostic query AST (the SQR), one parameterized compiler per backend family, a property-hook hydrator, a typed stateless repository layer, live network drivers, and a stateless migration runner.

For the architectural reasoning (why compile-layer + ports, why no Active Record), see [Explanation: The Universal Data & Persistence Layer](../explanation/data-persistence.md).

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
- **Engine note:** Oracle has no table-less `SELECT`, so pass `pingQuery: 'SELECT 1 FROM DUAL'` when pooling `oci` connections (the workspace `AppKernelFactory` does this automatically).

A pool exhaustion (`activeCount() == maxConnections`) or a failed connection raises a `DatabaseException`.

## Query AST (SQR)

The query model is pure representation — it holds no backend knowledge and is consumed by a compiler. As of Beta-3 the SQR **vocabulary lives in `waffle-commons/contracts`** so the repository contract can speak it: the value classes below implement `Waffle\Commons\Contracts\Data\Query\{QueryInterface, ComparisonInterface, OrderInterface}` (PHP 8.4+ *interface properties* — `public string $field { get; }` — no legacy getters).

### `Waffle\Commons\Data\Query\Query`

Immutable, copy-on-write builder implementing `QueryInterface`. Every method returns a new instance.

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

### Enums (relocated to contracts in Beta-3 — ⚠️ BC)

- `Waffle\Commons\Contracts\Data\Enum\Operator: string` — `Equal '='`, `NotEqual '<>'`, `GreaterThan '>'`, `GreaterThanOrEqual '>='`, `LessThan '<'`, `LessThanOrEqual '<='`, `In 'IN'`, `NotIn 'NOT IN'`, `Like 'LIKE'`; `isSetOperator(): bool` distinguishes `IN`/`NOT IN`.
- `Waffle\Commons\Contracts\Data\Enum\Direction: string` — `Ascending 'ASC'`, `Descending 'DESC'`.

> **Breaking change:** these enums previously lived at `Waffle\Commons\Data\Query\Operator` / `Direction`. Update imports to the `Contracts\Data\Enum` namespace.

## Repositories (RFC-022 §3)

Every repository implements `Waffle\Commons\Contracts\Data\Repository\RepositoryInterface` (`@template T of object`) for reads, and `Waffle\Commons\Contracts\Data\Repository\WritableRepositoryInterface` (which extends it) for CRUD writes:

```php
// RepositoryInterface (read)
public function find(QueryInterface $query): array;        // list<T>
public function findOne(QueryInterface $query): ?object;   // T|null — bounds the call server-side where possible
public function stream(QueryInterface $query): Generator;  // Generator<int, T> — §4.1 buffer streaming

// WritableRepositoryInterface (write) extends RepositoryInterface
public function save(object $entity): void;                // INSERT (null identity) or UPDATE/upsert
public function delete(object $entity): void;
public function findById(int|string $id): ?object;         // T|null
```

Writes go through a pure **Data Mapper** — no Active Record. `Waffle\Commons\Contracts\Data\Mapper\DataMapperInterface` (`@template T of object`):

```php
public function target(): string;                          // table / collection / key-prefix / endpoint
public function identityField(): string;                   // default 'id'
public function fields(): array;                            // list<string> — projection (read counterpart of toRow keys)
public function identify(object $entity): int|string|null; // null ⇒ save() performs an INSERT
public function toRow(object $entity): array;              // array<string, scalar|null>
```

All repositories are **stateless** (safe to share across worker requests), hydrate through the [`PropertyHookHydrator`](#hydrator--wafflecommonsdatahydratorpropertyhookhydrator), wrap backend failures as `DatabaseExceptionInterface`, reject malformed SQR with `\InvalidArgumentException`, and surface poisoned rows as `ValidationExceptionInterface`. The relational and flat-file repositories accept the mapper as an **optional trailing argument** — omit it for a read-only repository (the write methods then throw `\InvalidArgumentException`).

| Repository | Constructor | Backend / notes |
| :--- | :--- | :--- |
| `Repository\SQLRepository` | `(ConnectionPoolInterface $pool, string $target, SQLCompiler $compiler = new SQLCompiler(), ?DataMapperInterface $mapper = null, ?SQLWriteCompiler $writeCompiler = null)` | Any PDO engine. `stream()` is a **true driver cursor**; writes run in a transaction (rollback on failure). `$writeCompiler` SHOULD share the read compiler's dialect. |
| `Repository\FirestoreRepository` | `forPublic(FirestoreClientInterface $client, string $target, SecurityContextInterface $security, DataMapperInterface $mapper, string $appId)` / `forPrivate(...)` | Document store with the three guardrails (§4.2) — see the [Firestore compiler](#firestore-compiler--wafflecommonsdatacompilerfirestorecompiler) + [driver](#document-firestore--driverfirestorefirestoreclientinterface). |
| `Repository\JsonFileRepository` | `(string $path, string $target, JsonFileStore $store = new JsonFileStore(), ?DataMapperInterface $mapper = null)` | Atomic flat-file JSON (§4.3); SQR + writes evaluated in memory then written atomically. |
| `Repository\KeyValueRepository` | `(KeyValueClientInterface $client, string $target, KeyValueCompiler $compiler = new KeyValueCompiler(), ?DataMapperInterface $mapper = null)` | Redis/DynamoDB; one JSON document per key. Writes require an explicit identity (no auto-id). |
| `Repository\MongoRepository` | `(MongoSessionInterface $session, string $target, MongoCompiler $compiler = new MongoCompiler(), ?DataMapperInterface $mapper = null)` | Server-side push-down; writes are insert / replace-upsert / deleteOne. |
| `Repository\CassandraRepository` | `(CqlSessionInterface $session, string $target, CassandraCompiler $compiler = new CassandraCompiler(), ?DataMapperInterface $mapper = null)` | Parameterised CQL; `save()` is a CQL `INSERT` (upsert). |
| `Repository\GraphQLRepository` | `(GraphQLExecutor $executor, string $target, GraphQLCompiler $compiler = new GraphQLCompiler(), ?DataMapperInterface $mapper = null)` | GraphQL service as a virtual engine; writes are Hasura-style mutations. |

`$target` is the `class-string<T>` of the `readonly` DTO each row hydrates into. `findOne()` rebuilds the SQR with a server-side bound (`LIMIT 1` / `limit: 1`) whenever the concrete `Query` is passed and the backend supports it; backends without a cursor implement `stream()` by yielding from the bounded result page — only `SQLRepository` streams from a live cursor. `findById()` reuses the read path with an `identityField = id` equality predicate.

## SQL compiler — `Waffle\Commons\Data\Compiler\SQLCompiler`

```php
public function __construct(private SQLDialect $dialect = SQLDialect::MySQL);
public function compile(QueryInterface $query): CompiledQuery; // throws \InvalidArgumentException
```

Produces a `Waffle\Commons\Data\Compiler\CompiledQuery` (`final readonly`):

```php
public string $sql;        // parameterized: '?' placeholders only
public array $parameters;  // positional bound values, in order
```

All values are bound as positional `?` placeholders — **no value is ever interpolated into the SQL string** (OWASP A03). Identifiers are quoted per dialect. `compile()` throws `\InvalidArgumentException` when the query has no source table or a set predicate carries no values.

`Waffle\Commons\Data\Compiler\SQLDialect` (`enum`) selects identifier quoting and pagination grammar — **all six major relational engines** are covered:

| Case | Identifier quoting | Pagination |
| :--- | :--- | :--- |
| `SQLDialect::MySQL` | `` `ident` `` | `LIMIT … OFFSET …` (64-bit sentinel when only an offset is set) |
| `SQLDialect::MariaDB` | `` `ident` `` | identical to MySQL (wire- and syntax-compatible) |
| `SQLDialect::SQLite` | `"ident"` | `LIMIT … OFFSET …` (`-1` sentinel) |
| `SQLDialect::MSSQL` | `[ident]` | `OFFSET … ROWS FETCH NEXT … ROWS ONLY` (requires an `ORDER BY`) |
| `SQLDialect::PostgreSQL` | `"ident"` | independent `LIMIT …` / `OFFSET …` (an offset may stand alone) |
| `SQLDialect::Oracle` | `"ident"` | `OFFSET … ROWS FETCH NEXT … ROWS ONLY` (12c+; supply an `ORDER BY` for a deterministic page) |

`SQLDialect::quoteIdentifier(string): string` quotes dotted identifiers segment-by-segment (and escapes embedded quote characters); `SQLDialect::paginate(?int $limit, ?int $offset): string`.

## Firestore compiler — `Waffle\Commons\Data\Compiler\FirestoreCompiler`

```php
public function compile(QueryInterface $query, FirestoreScope $scope): CompiledFirestoreQuery;
```

Targets Cloud Firestore's structured query shape with **mandatory path isolation**:

```php
FirestoreScope::public(string $appId, string $collection): self;            // /artifacts/{appId}/public/data/{collection}
FirestoreScope::private(string $appId, string $userId, string $collection): self; // /artifacts/{appId}/users/{userId}/{collection}
```

Path segments are validated and URL-encoded; illegal segments are rejected. Only equality predicates are pushed server-side; range comparisons and ordering set `requiresInMemoryFilter = true` so the caller knows to post-filter client-side (the [`InMemoryEvaluator`](#in-memory-evaluator--wafflecommonsdataevaluationinmemoryevaluator) is that engine). The result `Waffle\Commons\Data\Compiler\CompiledFirestoreQuery` (`final readonly`) exposes:

```php
public string $path;
public array  $filters;
public array  $orderings;
public ?int   $limit;
public bool   $requiresInMemoryFilter;
public function toJson(): string;
```

## MongoDB compiler — `Waffle\Commons\Data\Compiler\MongoCompiler`

```php
public function compile(QueryInterface $query): CompiledMongoQuery; // throws \InvalidArgumentException (no collection)
```

Unlike Firestore, MongoDB evaluates rich predicates server-side, so **every** predicate is pushed down into a native filter document — nothing is deferred to memory. Operators map onto `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`; a SQL `LIKE` becomes an **anchored, metacharacter-escaped** `$regex` (`%` → `.*`, `_` → `.`). The explicit `$eq` form is used even for equality so multiple predicates on one field merge losslessly into a single operator document.

`CompiledMongoQuery` (`final readonly`) mirrors the driver's two-argument `find($filter, $options)` shape:

```php
public string $collection;
public array  $filter;              // field → operator → operand
public MongoFindOptions $options;   // projection, sort, limit, skip
```

`MongoFindOptions` (`final readonly`): `array $projection` (field → 1), `array $sort` (field → 1|-1), `?int $limit`, `?int $skip`.

## Key-value compiler — `Waffle\Commons\Data\Compiler\KeyValueCompiler`

```php
public function compile(QueryInterface $query): CompiledKeyValueCommand; // throws \InvalidArgumentException
```

A key-value store addresses opaque values by key alone, so the compiler accepts **only** the degenerate SQR that maps onto a key lookup and rejects everything richer with a precise message — never a silent mistranslation:

- accepted: a single key-equality predicate (→ `GET`) or key-membership predicate (→ `MGET`);
- rejected: a missing namespace (`from()`), any projection, ordering, `limit()`/`offset()`, more than one predicate, any other operator, or a `null` key.

`CompiledKeyValueCommand` (`final readonly`): `KeyValueOperation $operation` (`Get = 'GET'` | `MGet = 'MGET'`), `string $namespace`, `array $keys` (fully-qualified, `{namespace}:{value}`-prefixed).

## Cassandra compiler — `Waffle\Commons\Data\Compiler\CassandraCompiler`

```php
public function compile(QueryInterface $query): CompiledCassandraQuery; // throws \InvalidArgumentException
```

CQL is SQL-like but deliberately narrower: equality, the four range operators, and `IN` compile to parameterised CQL with `?` markers; **`<>`, `NOT IN`, `LIKE`, and `OFFSET` pagination are rejected** (CQL has none of them — Cassandra pages with token state). `CompiledCassandraQuery` (`final readonly`):

```php
public string $cql;
public array  $parameters;
public bool   $requiresAllowFiltering; // advisory: true whenever a WHERE is present —
                                       // only the deployment knows its primary keys
```

## GraphQL compiler — `Waffle\Commons\Data\Compiler\GraphQLCompiler`

```php
public function compile(QueryInterface $query): CompiledGraphQLQuery; // throws \InvalidArgumentException
```

Elevates an external GraphQL service to a virtual database engine (§4.3): the source becomes the root field, the projection the selection set, predicates a `where` argument in the conventional nested operator form (`field: {_gt: $v0}`), and pagination the `limit`/`offset` arguments. Every operand is emitted as a **declared variable, never inlined** — the injection guarantee of bound `?` parameters, in GraphQL form. Root, projection, and predicate names are validated against the GraphQL name grammar (`/^[_A-Za-z][_0-9A-Za-z]*$/`). GraphQL has no `SELECT *`, so an empty projection is rejected.

`CompiledGraphQLQuery` (`final readonly`):

```php
public string $query;       // executable GraphQL document
public array  $variables;   // operands keyed by variable name
public string $root;        // root field the executor reads rows from (data.{root})
public function toJson(): string; // standard {query, variables} POST body ({} when no variables)
```

## Live network drivers (`Waffle\Commons\Data\Driver\…`)

Driver transports are **ports**: tiny interfaces the repositories depend on, with thin live adapters over the actual extensions. Repository logic stays fully testable without a server; every driver failure is rethrown as a `DatabaseException` (§7.3).

### Key-value — `Driver\KeyValue\KeyValueClientInterface`

```php
public function get(string $key): ?string;            // null on a miss
public function getMany(array $keys): array;          // array<string, string|null>, keyed by requested key
```

Live adapter: **`Driver\KeyValue\RedisKeyValueClient`** (`final`, requires `ext-redis`) — wraps an injected, connected `\Redis` handle (held `readonly`; only stateless `GET`/`MGET` commands, no transactions or subscriptions).

### Document — `Driver\Mongo\MongoSessionInterface`

```php
public function find(CompiledMongoQuery $query): array; // list<array<string, int|float|string|bool|null>>
```

Live adapter: **`Driver\Mongo\MongoDriverSession`** (`final`, requires `ext-mongodb`) — `__construct(Manager $manager, string $database, RowNormaliser $normaliser = new RowNormaliser())`. The port exists because the extension's `MongoDB\Driver\Manager` is `final` and cannot be doubled. Documents are read with an array type map and validated into flat scalar rows; **the BSON `_id` is excluded by default** (a BSON `ObjectId` is not a flat scalar and would rightly be rejected) — store an explicit scalar id field instead.

### Wide-column — `Driver\Cql\CqlSessionInterface`

```php
public function execute(CompiledCassandraQuery $query): array;
```

The transport is deliberately injectable: **as of PHP 8.5 no maintained native CQL binary-protocol client exists** (the DataStax `ext-cassandra` is abandoned). Live access goes through an environment-provided adapter — typically a Stargate gateway (whose GraphQL face is served by the `GraphQLExecutor` family) or a future extension. Implementations decide whether to honour `requiresAllowFiltering` by appending `ALLOW FILTERING`.

### GraphQL — `Driver\Graph\GraphQLExecutor`

```php
public function __construct(
    ClientInterface $client,                 // PSR-18 — the framework's async http-client in production
    RequestFactoryInterface $requestFactory, // PSR-17
    StreamFactoryInterface $streamFactory,   // PSR-17
    string $endpoint,
    RowNormaliser $normaliser = new RowNormaliser(),
);

public function execute(CompiledGraphQLQuery $compiled): array; // list of flat rows from data.{root}
public function mutate(CompiledGraphQLMutation $mutation): void; // write; asserts 200 + no errors, ignores the return shape
```

POSTs the standard `{query, variables}` body (`Content-Type: application/json`), then unwraps `data.{root}`. A non-200 status, a transport failure, a malformed envelope, or a GraphQL `errors` array all fail the call as a `DatabaseException` — errors are never silently ignored. `mutate()` runs a `GraphQLMutationCompiler`-produced mutation and validates only transport status + the `errors` array, so any provider's mutation return shape is accepted.

### Document — `Driver\Firestore\FirestoreClientInterface`

```php
public function getDocument(string $path, string $id): ?array;                     // ?array<string, scalar|null>
public function queryCollection(string $path, array $filters, ?int $limit): array; // equality filters ONLY (Rule 2)
public function setDocument(string $path, ?string $id, array $row): string;         // returns the (possibly auto-assigned) id
public function deleteDocument(string $path, string $id): void;
```

Live adapter: **`Driver\Firestore\FirestoreRestClient`** — `__construct(ClientInterface $client, RequestFactoryInterface $requestFactory, StreamFactoryInterface $streamFactory, string $endpoint, RowNormaliser $normaliser = new RowNormaliser())`. A thin REST boundary that POSTs each operation as a JSON envelope to a Firestore client / raw NoSQL proxy over PSR-18. The port is deliberately dead-simple: `queryCollection` accepts only equality filters + an optional limit, so Rule 2 is structural — compound/range/sort resolve in `FirestoreRepository` via the `InMemoryEvaluator`.

> The write methods on the other ports mirror this: `KeyValueClientInterface` adds `set`/`delete`, `MongoSessionInterface` adds `insert`/`upsert`/`deleteOne`, and `CqlSessionInterface` adds `executeWrite(string $cql, array $parameters)`.

## In-memory evaluator — `Waffle\Commons\Data\Evaluation\InMemoryEvaluator`

```php
public function evaluate(QueryInterface $query, array $rows): array; // filter → sort → paginate
```

The engine behind the *fetch-simple-then-filter* strategy (flat-file JSON, and Firestore when `requiresInMemoryFilter` is raised). Semantics are SQL-faithful and strict: equality is `===` / set membership is strict `in_array`; **a `NULL` on either side of a range predicate never matches** (`NULL > x` is unknown); `LIKE` is anchored with `%`/`_` wildcards and never matches a non-string; multi-key sorting ranks **nulls first, deterministically**. Holds no state — every call is pure over its arguments.

## Flat-file JSON store — `Waffle\Commons\Data\Storage\JsonFileStore`

```php
public function __construct(
    InMemoryEvaluator $evaluator = new InMemoryEvaluator(),
    RowNormaliser $normaliser = new RowNormaliser(),
);

public function read(string $path): array;                       // [] when the file does not exist
public function query(string $path, QueryInterface $query): array;
public function write(string $path, array $rows): void;          // atomic: temp file + rename
```

RFC-022 §4.3 storage: each collection is one JSON file holding a list of flat rows. Writes are crash- and concurrency-safe by construction — the payload is written to a uniquely-named temp file **in the same directory** under `LOCK_EX`, then `rename()`d over the target (atomic on POSIX), so a reader sees either the whole previous file or the whole new one. The temp file never touches `sys_get_temp_dir()` (statelessness mandate; same-filesystem rename). Decoded data is validated through the `RowNormaliser` — a corrupt store throws a `DatabaseException`.

**Type note:** JSON carries a single number type, so a whole-number float (`7.0`) is encoded as `7` and reads back as an `int`.

## Row normaliser — `Waffle\Commons\Data\Hydrator\RowNormaliser`

```php
public function fromJsonRows(string $json): array;               // JSON payload → list of flat rows
public function fromJsonRow(string $json): array;                // one JSON document → one flat row
public function normaliseAll(array $rows): array;
public function normalise(array $row): array;
public function fromFetch(PDOStatement $statement): ?array;      // next cursor row, null when exhausted
```

The **single funnel** for untyped backend output (`json_decode()`, `PDOStatement::fetch()`, wire payloads are `mixed` by signature): every value is narrowed by an `is_array()` / `is_scalar()` guard before use, and any shape violation throws a `DatabaseException` — a poisoned row never reaches application code with a widened type.

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

Surfaced to operators as `bin/waffle db:migrate` (see [console.md](console.md)). For the end-to-end workflow against the workspace PostgreSQL sandbox, see [How to: Database Migrations](../how-to/database-migrations.md).

## Warmup — `Waffle\Commons\Data\Warmup\QueryWarmer`

*Added in Beta-3.* Implements `Waffle\Commons\Contracts\Data\Warmup\DataWarmerInterface` — the engine behind `bin/waffle data:warmup` (see [console.md](console.md)):

```php
public function __construct(array $queries, SQLCompiler $compiler, string $cacheFile);
public function warmUp(): array; // list<string> — human-readable artifact descriptors
```

- Compiles a registry of **named SQR trees** (`array<non-empty-string, QueryInterface>`) through the dialect-bound `SQLCompiler` into a single `<?php return […]` artifact (`['name' => ['sql' => …, 'parameters' => […]]]`).
- The artifact is replaced **atomically** (write-to-temporary + `rename`), so a resident worker can never observe a half-written file; when OPcache is active on the CLI it is immediately primed via `opcache_compile_file()`, removing compilation and disk I/O from the first live request.
- Stateless and idempotent by construction (`final readonly`): the query map is fixed at boot and every run recompiles from scratch — safe to re-run after every deploy. Applications wire their warmers in `bin/waffle` (see the skeleton/workspace reference entry points).
- Failures (unwritable directory, failed atomic publish) raise `WarmupException`.

## Exceptions

Both implement their respective contracts interface so callers catch persistence/validation failures without coupling to a driver:

- `Waffle\Commons\Data\Exception\DatabaseException` → `Waffle\Commons\Contracts\Data\Exception\DatabaseExceptionInterface`. `getSqlState(): ?string` lifts the ANSI `SQLSTATE` from a `PDOException` (`null` for non-relational backends). `DatabaseException::fromThrowable(\Throwable, ?string)` wraps any backend error while preserving the original as `previous`.
- `Waffle\Commons\Data\Exception\ValidationException` → `Waffle\Commons\Contracts\Exception\Validation\ValidationExceptionInterface`. `getField(): ?string` names the offending field.
- `Waffle\Commons\Data\Exception\SecurityPathViolationException` → `…\Data\Exception\SecurityPathViolationExceptionInterface` (extends `DatabaseExceptionInterface`). Raised when a Firestore operation would target a non-isolated path (Rule 1).
- `Waffle\Commons\Data\Exception\UnauthenticatedAccessException` → `…\Data\Exception\UnauthenticatedAccessExceptionInterface` (extends `DatabaseExceptionInterface`). Raised when a guarded Firestore read/write is attempted without an authenticated identity (Rule 3).
- `Waffle\Commons\Data\Exception\WarmupException` (extends `DatabaseException`). Raised when a `data:warmup` artifact cannot be compiled or published (unwritable cache directory, failed atomic rename); carries no SQLSTATE.

**Leak containment:** every driver call-site wraps its backend failure with an *explicit sanitized message* — credentials, DSNs and driver internals never reach a rendered RFC 7807 body; the raw driver error survives only as `previous`, for logging.

## Worker-safety contract

`PDOConnectionPool` is the only stateful object, and its state is explicitly recyclable via `reset()`. Compilers, repositories, drivers, the evaluator, the normaliser, the hydrator, the warmer, and the query AST are stateless / immutable (the component passes the `igor-php` worker-mode audit with zero findings). The kernel calls `reset()` between worker iterations (and the `db:migrate` command resets the pool on the way out), so no per-request state leaks across the FrankenPHP worker boundary.

## Quick example

```php
use Waffle\Commons\Data\Compiler\SQLCompiler;
use Waffle\Commons\Data\Compiler\SQLDialect;
use Waffle\Commons\Data\Query\Criteria;
use Waffle\Commons\Data\Query\Query;
use Waffle\Commons\Data\Repository\SQLRepository;

// One readonly DTO per row; Property Hooks validate at construction.
final readonly class UserRow
{
    public function __construct(
        public string $id,
        public string $email,
    ) {}
}

/** @var SQLRepository<UserRow> $users */
$users = new SQLRepository($pool, UserRow::class, new SQLCompiler(SQLDialect::PostgreSQL));

$query = Query::select('id', 'email')
    ->from('users')
    ->where(Criteria::eq('status', 'active'))
    ->orderBy('id')
    ->limit(10);

$page = $users->find($query);          // list<UserRow>
$first = $users->findOne($query);      // UserRow|null — server-side LIMIT 1

foreach ($users->stream($query) as $user) {
    // true driver cursor: one row in memory at a time (§4.1)
}
```

The same `$query` compiles unchanged for any other backend — swap the repository (`MongoRepository`, `KeyValueRepository`, …) and the SQR is translated into that engine's native form.
