# Performance Strategy

Waffle is designed to be one of the fastest PHP frameworks available.

## FrankenPHP Integration
The `WaffleRuntime` native integration with [FrankenPHP](https://frankenphp.dev) allows the application to stay in memory between requests. This eliminates the bootstrapping overhead (Container creation, Config loading) for subsequent requests, leading to sub-millisecond response times.

## Preloading
The framework structure is "Preloading-Friendly".
- The `waffle-commons/*` libraries are designed to be preloaded into Opcache shared memory.
- Using `opcache.preload` in your `php.ini` ensures that all core classes are available instantly, reducing I/O and CPU usage.

## Memory-bounded HTTP proxying (FinOps)

An edge gateway proxies traffic to slower upstreams (e.g. a legacy monolith). Doing that inside a resident worker *without* leaking memory or pinning threads is a FinOps concern: wasted RAM and workers blocked on a slow backend both cost money at scale. The `waffle-commons/http-client` (PSR-18) is engineered for exactly this workload.

- **Persistent connections.** The client holds a single `\CurlHandle` plus a `\CurlMultiHandle` for the worker's lifetime, reused via `curl_reset()` on every `sendRequest()`. libcurl's DNS cache and keep-alive pool stay warm across requests, so repeated calls to the same upstream skip the TCP/TLS handshake entirely.
- **Non-blocking transfer.** Requests are driven through the multi interface: the worker parks on `curl_multi_select()` (a socket-level wait) between `curl_multi_exec()` ticks instead of busy-spinning a CPU or blocking inside `curl_exec()`. A slow upstream can no longer pin a worker beyond the hard `10s` total-timeout ceiling (`1s` to connect).
- **Bounded memory, both directions.** Response bodies stream into a PSR-7 stream in **8 KiB** chunks (`CURLOPT_WRITEFUNCTION`); request bodies are *pulled* from the PSR-7 request stream in 8 KiB chunks (`CURLOPT_READFUNCTION` + `CURLOPT_UPLOAD`). Neither side is ever materialised whole in worker RAM — proxying a multi-gigabyte upload or download costs roughly one 8 KiB buffer, not gigabytes.

The net effect is a **fixed, predictable per-worker memory ceiling regardless of payload size**, and workers that are never held hostage by a slow backend. See the [HTTP Client reference](../reference/http-client.md) for the exact cURL options, SSRF protocol allowlist, and timeout constants.
