# Performance Strategy

Waffle is designed to be one of the fastest PHP frameworks available. The strategy rests on four pillars that compound: a resident **worker process** (no per-request bootstrap), **ahead-of-time compilation** (no first-request reflection), **memory-resident pooling** (no per-request connection handshakes), and **finish-request deferral** (no non-essential work on the latency path). Each pillar is explained below or on its companion page — and each one is only safe because of the [statelessness mandate](architecture.md#the-statelessness-mandate) (`ΔM = 0`, audited by `wfl igor`).

## FrankenPHP Integration
The `WaffleRuntime` native integration with [FrankenPHP](https://frankenphp.dev) allows the application to stay in memory between requests. This eliminates the bootstrapping overhead (Container creation, Config loading) for subsequent requests, leading to sub-millisecond response times.

Because workers are long-lived, cumulative metric counters must live in APCu shared memory rather than the per-worker heap; the `GcCollector`/`MemoryCollector` then surface the memory-pressure signals described below without growing the worker. See [Contract-First Observability](observability-telemetry.md).

## Preloading
The framework structure is "Preloading-Friendly".
- The `waffle-commons/*` libraries are designed to be preloaded into Opcache shared memory.
- Using `opcache.preload` in your `php.ini` ensures that all core classes are available instantly, reducing I/O and CPU usage.

## Ahead-of-Time Compilation
Complementing OPcache preloading, [Ahead-of-Time Compilation](aot-compilation.md) (opt-in via `WAFFLE_AOT=1`) moves deterministic container-wiring and route-discovery work out of the first-request path — a reflection-free compiled container plus a serialized route trie — for sub-millisecond cold starts.

## Memory-bounded HTTP proxying (FinOps)

An edge gateway proxies traffic to slower upstreams (e.g. a legacy monolith). Doing that inside a resident worker *without* leaking memory or pinning threads is a FinOps concern: wasted RAM and workers blocked on a slow backend both cost money at scale. The `waffle-commons/http-client` (PSR-18) is engineered for exactly this workload.

- **Persistent connections.** The client holds a single `\CurlHandle` plus a `\CurlMultiHandle` for the worker's lifetime, reused via `curl_reset()` on every `sendRequest()`. libcurl's DNS cache and keep-alive pool stay warm across requests, so repeated calls to the same upstream skip the TCP/TLS handshake entirely. The database equivalent — keeping handles warm across worker iterations with a bounded, health-checked, reset-on-iteration pool — is [Memory-Resident Connection Pooling](connection-pooling.md).
- **Non-blocking transfer.** Requests are driven through the multi interface: the worker parks on `curl_multi_select()` (a socket-level wait) between `curl_multi_exec()` ticks instead of busy-spinning a CPU or blocking inside `curl_exec()`. A slow upstream can no longer pin a worker beyond the hard `10s` total-timeout ceiling (`1s` to connect).
- **Bounded memory, both directions.** Response bodies stream into a PSR-7 stream in **8 KiB** chunks (`CURLOPT_WRITEFUNCTION`); request bodies are *pulled* from the PSR-7 request stream in 8 KiB chunks (`CURLOPT_READFUNCTION` + `CURLOPT_UPLOAD`). Neither side is ever materialised whole in worker RAM — proxying a multi-gigabyte upload or download costs roughly one 8 KiB buffer, not gigabytes.

The net effect is a **fixed, predictable per-worker memory ceiling regardless of payload size**, and workers that are never held hostage by a slow backend. See the [HTTP Client reference](../reference/http-client.md) for the exact cURL options, SSRF protocol allowlist, and timeout constants.

The same libcurl multi-interface also powers Beta-5 [concurrent fan-out](async-finish-request-deferral.md#async-02-real-concurrency-where-it-actually-exists): N outbound requests complete in roughly the wall-clock of the slowest via one multi-handle loop. And [finish-request deferral](async-finish-request-deferral.md) keeps short post-response work (mail, audit, webhooks) off the user-perceived latency path entirely, draining it on `TerminateEvent` after the response is flushed.

## Memory-resident connection pooling

The database analogue of the persistent cURL handles above is [Memory-Resident Connection Pooling](connection-pooling.md) (Beta-5, DBAL-01/DBAL-02): PDO and Redis handles stay warm across worker iterations in a bounded, health-checked pool — a dead socket is healed on lease, never surfaced to the caller — and the pool's `reset()` rolls back any straggler transaction at the iteration boundary. Write requests are additionally wrapped in a single failsafe transaction on a pinned connection by `TransactionIsolationMiddleware`. The result: no per-request TCP/TLS/auth handshake against the database, with the worker-safety invariants intact.

## Measured numbers

*Placeholder — the Beta-6 benchmark campaign lands its results here.*

The Beta-6 stabilization cycle runs a **K6 tri-engine benchmark**, and this section will carry the published figures rather than claims. Method, for reproducibility:

- **Latency:** constant-arrival-rate scenarios (open-model load, so latency degradation cannot hide behind a slowing load generator), reported as percentile distributions (p50/p95/p99) per engine.
- **Memory stability:** a long soak run asserting the `ΔM = 0` invariant — flat worker RSS across the full soak, the runtime counterpart of the static `wfl igor` audit.

Until those runs are published, this page intentionally quotes **no absolute numbers**: every performance claim above is architectural (what work is avoided and why), not a benchmark result.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
