# Ahead-of-Time Compilation (Beta-5)

> **Status:** shipped in Beta-5 (AOT-01 / AOT-02, RFC-019, roadmap AXE1).
> **Companion pages:** [AOT reference](../reference/aot.md) · [How to: Enable AOT](../how-to/enable-aot.md).

This page explains *why* Waffle grows an Ahead-of-Time build step and *how* the compiled artifacts stay honest. For exact class names and signatures, use the [reference](../reference/aot.md).

## The cold-start problem

FrankenPHP keeps the application resident, so the bootstrap cost — building the container, discovering routes — is normally amortised across thousands of requests on one warm worker. But the *first* request a fresh worker serves still pays that cost in full, and in a high-density deployment (many small containers, aggressive autoscaling, frequent rollouts) workers are minted constantly. Two pieces of that first-request bootstrap are pure reflection:

1. **Container wiring.** The reflection-based `Waffle\Commons\Container\Container` resolves every service by inspecting constructor signatures with `ReflectionClass` the first time each service is requested.
2. **Route discovery.** The `RouteDiscoverer` scans the controller directory and parses every `#[Route]` attribute to assemble the routing table.

Both produce a result that is *identical on every boot of a given build* — the code did not change between deploys. AOT moves that deterministic work out of the request path and into an explicit build step, so a fresh worker loads a finished artifact instead of recomputing it. The goal is **sub-millisecond cold starts** in dense containers.

## A build step, never magic at runtime

The defining rule (RFC-019) is that AOT is **opt-in and transparent**. Nothing is compiled, discovered, or cached during an HTTP request: the artifacts are produced only by explicit CLI commands (`waffle container:compile`, `waffle route:compile`), and the fast path is consulted only when the operator sets `WAFFLE_AOT=1`. With the flag unset — the default, and the entire dev experience — the kernel behaves exactly as it did before: reflection container, live route discovery. There is no hidden warm-up, no auto-regeneration, no staleness daemon.

This makes the feature drop-in and reversible. A wrong, corrupt, or absent artifact never breaks the application; it degrades to the reflection path with a logged warning. The fast path can only ever be *faster*, never *different*.

## The compiled container

At build time the `Waffle\Commons\Console\Compiler\ContainerCompiler` takes the **fully-booted, locked** runtime container and reads its private `definitions` map (via reflection on the contract, so the compiler never depends on the concrete container package). For every definition it decides whether the service is *inlinable*:

- A **class-string concrete** whose constructor it can fully resolve by reflection is inlined — the compiler emits a literal `new \Fully\Qualified\Class(...)` expression, wiring each dependency through `$this->get(...)` exactly as `Autowire` would have.
- Everything else — **closures** (lazy factories) and **pre-registered objects** — is a *passthrough*: the generated `get()` delegates to the composed runtime container, because that construction logic cannot be expressed as static source.

The emitted class implements `Waffle\Commons\Contracts\Container\CompiledContainerInterface` and **composes** the runtime container (holds it as a `readonly` property). `has()` and `set()` delegate verbatim; only the *resolution mechanism* for inlinable services changes — static constructor calls replace per-request reflection. The graph is therefore the same concrete classes wired the same way, just resolved faster.

The compiler does not invent its own wiring rules. The reflection logic in `inlinableClass()` / `emitArgument()` mirrors `Waffle\Commons\Container\Autowire::resolveDependencies()` branch for branch — including the subtle cases: a nullable dependency that is not registered resolves to `null` (not a thrown exception), and a union-typed parameter selects the first *registered* member left-to-right via a `has()`-guarded ternary chain, falling back to the declared default. Matching those branches is the whole point: a compiled container that diverged from `Autowire` would be a different application.

### Proven graph-identical, not assumed

The promise "the compiled graph equals the runtime graph" is not a comment — it is a test. `ContainerCompilerTest::testCompiledGraphIsIdenticalToRuntimeGraph()` seeds a runtime container, captures an FQCN-normalised *signature* of a set of services (leaf, mid, root, an interface-bound concrete, an optional-dependency service, two union cases), then compiles that same container, instantiates the artifact, and asserts the two signatures are byte-identical. Optional-dependency and union edge cases each get their own assertion. If the compiler ever drifts from `Autowire`, this snapshot turns red.

### Statelessness is preserved

A compiled container introduces **no new cross-request state**. The only mutable field on the generated class is a per-request memo map (`$instances`) — the exact analogue of the runtime container's own per-request instance cache. Its `reset()` does two things: it cascades `reset()` to every memoised service implementing `Waffle\Commons\Contracts\Service\ResettableInterface`, then resets the composed runtime container. So an inlined resettable service still participates in the FrankenPHP per-worker reset cascade, and the `igor-php` audit stays clean.

## The route trie

Route matching has its own AOT track (AOT-02). The sequential `Waffle\Commons\Routing\Router::resolve()` walks a priority-sorted list and runs a per-route PCRE until one matches — O(n) in the number of routes. The `Waffle\Commons\Routing\Trie\RouteTrie` replaces that scan with a **segment-keyed lookup tree**: a static segment (`users`) is a literal key, a dynamic segment (`{id}` / `{id:\d+}`) is a single dynamic child carrying its parameter name and optional constraint, and a catch-all (`{path:.*}`) is a wildcard leaf that swallows the path remainder. Resolution walks the tree in **O(depth)** — the number of path segments — independent of how many routes the application declares.

The hard requirement is **behavioural parity**, not merely "fast". The trie must return the *same* route the sequential matcher would, including priority ordering, 405 Method-Not-Allowed handling, and HEAD/OPTIONS semantics:

- Every route is tagged at build time with the index it held in the already-priority-sorted list. Lookup collects *all* path-matching candidates from every branch (static, dynamic, catch-all) and sorts them by that build-time order, so a higher-priority dynamic route still beats a lower-priority static one — the trie never hardcodes a naive "static-beats-dynamic" preference.
- The 405 `Allow`-set construction and the HEAD-matches-GET / auto-OPTIONS rules are ported verbatim from the `Router`.
- A root-mounted catch-all (`/{path:.*}`) is evaluated even when the path is exhausted, so it matches the root path `/` exactly as the sequential matcher's full-path PCRE did (AOT-03).

`route:compile` serialises the discovered, priority-sorted route list (or, when the application wires the optional trie builder, the flattened trie array from `RouteTrie::toArray()`) into a PHP artifact via `serialize()` + base64 — chosen over `var_export()` because the immutable `MatchedRoute` DTOs round-trip exactly without a `__set_state()` hook. At boot the `Router` prefers the prebuilt trie from its cache; if only the route list is cached (an older build), it rebuilds the trie from that list. Either way matching stays O(depth), and a `Router` seeded with routes directly (no boot pass, e.g. in a test) transparently falls back to the sequential matcher.

## Staleness is the operator's job, loudly

The deliberate non-feature here is **fingerprinting** (AOT-04). The kernel's `CompiledContainerLoader` performs no cross-component hash comparison — it cannot tell whether an artifact matches the current code. A stale artifact would silently serve an outdated service graph, which is a worse failure than a slow boot. Rather than building a brittle invalidation system, Waffle makes the contract explicit: every successful AOT load emits a prominent `LoggerInterface::warning()` reminding the operator that the artifact is **not** validated against the current code and **must** be regenerated (`bin/waffle container:compile`) after *any* code change. The recommended discipline is to regenerate both artifacts as a build/deploy step, never by hand. See the [how-to](../how-to/enable-aot.md) for the exact pipeline placement.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
