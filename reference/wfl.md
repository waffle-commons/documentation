# Reference: `wfl` — Waffle Local Developer CLI

`wfl` is the host-side developer CLI for the `waffle-commons` monorepo. It is a single Bash script (`bin/wfl`) that wraps `docker` / `docker compose`, `composer`, `mago`, and PHPUnit so every command runs **inside the `waffle-dev` container** — you never run PHP on the host (per `CLAUDE.md`).

> **Scope:** This sheet documents the commands that physically exist in `bin/wfl`. Run `wfl help` for the same surface in your terminal.

## Installation

`wfl init` symlinks the script into `~/.local/bin/wfl`. If that directory is on your `$PATH`, `wfl` is then available globally:

```bash
./bin/wfl init      # init submodules, boot Docker, install git hooks, symlink wfl
```

If `~/.local/bin` is not on your `$PATH`, `wfl init` prints a warning telling you to add it to your shell rc.

## Environment

| Item | Value |
| :--- | :--- |
| Primary container | `waffle-dev` |
| Legacy container | `legacy-backend` |
| Redis container | `waffle-redis` |
| Compose file | `workspace/docker-compose.yml` |
| Active PHP profile | `workspace/docker/php/mode-active.ini` (symlink → `mode-debug.ini` or `mode-bench.ini`) |

## Command reference

### Lifecycle

| Command | Description |
| :--- | :--- |
| `wfl init` | Initialise submodules (`git submodule update --init --recursive`), boot the Docker stack, install git hooks (`scripts/install-git-hooks.sh`), and symlink `wfl` into `~/.local/bin`. |
| `wfl up [args…]` | `docker compose up -d` (extra args passed through). |
| `wfl down [args…]` | `docker compose down` (extra args passed through). |
| `wfl status` | Print the active PHP profile and the state of the `waffle-dev`, `legacy-backend`, and `waffle-redis` containers. |

### Execution (inside the container)

| Command | Description |
| :--- | :--- |
| `wfl shell [component]` | Open a Bash shell in the container; with a component name, `cd`s into `/waffle-commons/<component>` first. |
| `wfl run [component] <cmd…>` | Run an arbitrary command in a component (if the first arg is an existing component dir) or at the monorepo root otherwise. |
| `wfl mago [component]` | Run `composer mago` (fmt + lint + analyze + guard) for a component. Defaults to the component inferred from your current directory. |
| `wfl test [component]` | Run `composer tests` for a component. Defaults to the component inferred from your current directory. |

When no component is given, `wfl` infers it from your working directory (you must be inside `<repo>/<component>/…`); otherwise it asks you to pass one explicitly.

### PHP profile switching

The active PHP profile is a symlink that `wfl` flips, then it restarts the `waffle` service.

| Command | Effect |
| :--- | :--- |
| `wfl debug` | Activate the 🐛 **debug** profile (`mode-debug.ini`: Xdebug on, JIT off) and restart the container. |
| `wfl bench` | Activate the 🚀 **bench** profile (`mode-bench.ini`: Xdebug off, JIT on) and restart the container. |

### Maintenance

| Command | Description |
| :--- | :--- |
| `wfl logs clear` | Truncate `/tmp/xdebug.log` and `/var/log/caddy/*.log` inside the container. |
| `wfl cache clear [component]` | Remove the PSR-16 framework cache (`var/cache/psr16`) for a component. |
| `wfl components` | List the known submodule components (read from `.gitmodules`). |
| `wfl help` | Print the built-in help. |

### Development

| Command | Description |
| :--- | :--- |
| `wfl link <consumer> <provider>` | Add a Composer `path` repository (`../<provider>`, non-symlink) to `<consumer>/composer.json` and `composer update waffle-commons/<provider>` so the consumer builds against your **local** provider checkout. |
| `wfl unlink <consumer> <provider>` | Remove that `path` repository from `<consumer>/composer.json` and `composer update` back to the registry version. |
| `wfl csrf-init [token-id]` | Generate a matching `WAFFLE_SID` cookie value and a signed `X-CSRF-Token` (HMAC over `nonce ‖ expiresAt ‖ id ‖ sid`) for manual API testing. Token id defaults to `_default`. Reads `WAFFLE_CSRF_SECRET` from the environment or `.env`; generates an ephemeral secret if none is found. |
| `wfl secret-gen` | Print a fresh 32-byte (256-bit) cryptographically secure application secret, suitable for `APP_SECRET` / `WAFFLE_CSRF_SECRET`. |

> **Note on `link` / `unlink`:** both take **two** arguments — the *consumer* component and the *provider* component, in that order. For example, `wfl link routing utils` makes `routing` build against your local `utils` checkout.

## See also

- [How-To: Local multi-component workflow](../how-to/local-development-workflow.md) — `link` / `unlink` / `debug` / `bench` in practice.
- [How-To: Secure a Controller](../how-to/secure-a-controller.md) — `wfl csrf-init` for testing CSRF-protected routes.
