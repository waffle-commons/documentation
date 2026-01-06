# Runtime Reference (`waffle-commons/runtime`)

Responsible for the Application entry point and Server loop.

## `Waffle\Runtime\WaffleRuntime`

### `run(): void`
The main execution method used in `public/index.php`.
1.  **Boot**: Instantiates the Kernel.
2.  **FrankenPHP Support**: Detects `frankenphp_handle_request` function.
    - If present, enters the **Worker Mode** loop (handling multiple requests in one process).
    - If absent, handles a single request (Standard PHP-FPM mode).
3.  **Emit**: Sends the response to the client.

## Preloading
The Runtime and Core are optimized for Opcache Preloading. Generating a preload script (via `waffle-commons/utils` or composer) significantly improves boot time in production.
