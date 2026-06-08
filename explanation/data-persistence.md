# The Universal Data & Persistence Layer (RFC-022)

> **Status:** shipped across the `0.1.0-beta3` cycle in `waffle-commons/data` (+ contracts).
> **Companion pages:** [data reference](../reference/data.md) · [How to: Database Migrations](../how-to/database-migrations.md).

This page explains *why* the persistence layer looks the way it does. For exact signatures, use the reference.

## Why not an ORM

Doctrine- and Eloquent-style ORMs were architected for short-lived, single-request processes. Under FrankenPHP's resident worker they fail in three characteristic ways (RFC-022 §2):

1. **Memory accumulation** — a Unit-of-Work/Identity-Map caches every loaded entity; over thousands of worker loops the curve only goes up, until the OOM killer ends the process.
2. **Connection loss** — persistent workers hold sockets open; when the server drops an idle connection, a naive layer throws fatal errors instead of reconnecting.
3. **Context pollution** — stateful entities shared between loops can leak Request A's data into Request B.

The UDPL inverts all three: **no entities, no tracking, no shared mutable state**. A row is fetched, validated, frozen into a `readonly` DTO, and forgotten by the layer that produced it. The only stateful object in the whole component is the connection pool, and its state is explicitly recyclable (`ResettableInterface::reset()` rolls back stragglers and re-pools handles between iterations). The `igor-php` audit gates this property in CI fashion: zero state mutations, zero findings.

## One query language, many engines: the SQR

Frameworks usually "unify" backends by forcing SQL semantics onto NoSQL engines, producing leaky abstractions. The UDPL instead unifies the **representation**, not the execution: repositories build a *Semantic Query Representation* — an immutable AST of projection, source, AND-combined predicates, ordering, and bounded pagination — and each backend owns a **compiler** that translates the SQR into its native form:

| Family | Compiler output | Execution character |
| :--- | :--- | :--- |
| Relational (MySQL, MariaDB, SQLite, MSSQL, PostgreSQL, Oracle) | parameterised SQL, `?` placeholders | full push-down |
| MongoDB | native filter document (`$eq`, `$gt`, `$in`, anchored `$regex`) | full push-down |
| Firestore | structured query payload on an isolated path | equality-only push-down; the rest is flagged for in-memory evaluation |
| Key-value (Redis, DynamoDB) | `GET` / `MGET` key commands | key lookup only — everything richer is **rejected** |
| Cassandra (CQL) | parameterised CQL | CQL subset; `<>`, `NOT IN`, `LIKE`, `OFFSET` are **rejected**; `ALLOW FILTERING` is advisory |
| GraphQL | query document + declared variables | the external service acts as a virtual engine |

Two design rules keep this honest:

- **Injection safety is structural, not disciplinary.** No compiler ever concatenates an operand into query text — SQL gets bound `?` parameters (typed `PARAM_BOOL`/`PARAM_INT`/`PARAM_NULL` at the driver), GraphQL gets declared variables, Mongo gets a data-bound filter document. Identifiers are quoted-and-escaped per dialect; GraphQL names are validated against the grammar.
- **A backend that cannot honour a clause rejects it loudly.** The key-value compiler refuses projections, orderings and pagination; the Cassandra compiler refuses non-CQL operators and `OFFSET`. A caller can never believe a backend is filtering when it is not — that is the difference between an abstraction and a lie.

The *fetch-simple-then-filter* strategy (Firestore's non-equality clauses, the flat-file store) is implemented once, in the `InMemoryEvaluator`, with SQL-faithful semantics (`NULL` never satisfies a range; strict equality; nulls sort first deterministically).

## The repository layer: typed, stateless, streaming

`RepositoryInterface` (in **contracts**, per RFC-022 §3) is the application-facing surface: `find()`, `findOne()`, `stream()`, generic over the target DTO. Hydration goes through constructor-driven mapping with PHP 8.5 Property-Hook validation — a poisoned record throws a `ValidationExceptionInterface` exactly like poisoned request input, it never becomes a silently corrupt object.

`stream()` exists because of the §4.1 buffer-streaming mandate: result sets must not materialise in memory at once. `SQLRepository` honours it with a true driver cursor (one row buffered at a time, the connection guaranteed back to the pool even when the consumer abandons the generator); page-oriented backends honour it by yielding from an already-bounded page — which is the honest version of the same contract.

## Live drivers are ports + thin adapters

Each non-PDO family separates **transport** (a tiny interface) from **logic** (compiler + repository):

- it keeps repository logic fully testable without servers;
- it absorbs extension quirks — `MongoDB\Driver\Manager` is `final` and undoublable, so `MongoSessionInterface` fronts it; the adapter also excludes the BSON `_id` by default because an `ObjectId` is not a flat scalar row value;
- it lets us tell the truth about Cassandra: **no maintained native CQL client exists for PHP 8.5** (the DataStax extension is abandoned), so `CqlSessionInterface` is an injectable port — a Stargate gateway or future extension plugs in without touching the repository.

Redis (`ext-redis`) and MongoDB (`ext-mongodb`) ship live adapters; GraphQL executes over **PSR-18/PSR-17**, which also keeps the component inside its dependency perimeter — like every component, `data` depends only on `waffle-commons/contracts` (plus PSR interfaces and the PHP extensions themselves).

All raw backend output funnels through one class, the `RowNormaliser` — the single place where wire-level `mixed` is allowed to exist, guard-narrowed and rejected on any shape violation.

## The sandbox

The workspace compose file provides the proving ground: **`waffle-postgres`** (the primary relational sandbox — `db:migrate` runs against it end-to-end) and **`waffle-mongo`** (internal-only, used by the data component's live integration tests, which skip cleanly when the service is absent). `waffle-redis` doubles as the key-value integration target.
