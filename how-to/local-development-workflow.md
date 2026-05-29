# How-To: Work on multiple components locally

The `waffle-commons` monorepo is a set of **independent Git submodules**, each published as its own Composer package. By default a component builds against the *released* version of its dependencies (e.g. `routing` pulls `waffle-commons/utils` from Packagist). When you need to change a provider and a consumer **at the same time**, the `wfl` CLI lets you point the consumer at your local provider checkout — and then put it back.

All of these commands run against the `waffle-dev` container. See the [`wfl` reference](../reference/wfl.md) for the full command surface.

## Link a local provider into a consumer

`wfl link` takes **two** arguments: the *consumer* component, then the *provider* you want it to build against locally.

```bash
# Make the routing component build against your LOCAL utils checkout
wfl link routing utils
```

This:

1. Adds a Composer `path` repository (`url: ../utils`, non-symlink copy) to the **top** of `routing/composer.json`'s `repositories` list.
2. Runs `composer update waffle-commons/utils` inside the container so `routing`'s `vendor/` now contains your local `utils` source.

You can now edit `utils/src/**` and re-run `routing`'s tests against the change without publishing anything:

```bash
wfl test routing      # composer tests for routing, using local utils
```

> **Tip:** the link is a non-symlinked `path` repository, so Composer *copies* the provider in. After editing the provider, re-run `wfl link routing utils` (or `composer update` inside `routing`) to refresh the copy.

## Unlink when you're done

`wfl unlink` reverses the change with the same two-argument shape:

```bash
wfl unlink routing utils
```

This removes the `../utils` `path` repository from `routing/composer.json` (dropping the whole `repositories` key if it becomes empty) and runs `composer update waffle-commons/utils` to restore the registry version. Commit `routing/composer.json` only in its unlinked state — local `path` repositories must never be committed.

## Switch PHP profiles: debug vs. bench

The dev image ships two PHP profiles. `wfl` flips the active one (a symlink) and restarts the container:

```bash
wfl debug    # 🐛 Xdebug ON, JIT OFF  — step-debugging and coverage
wfl bench    # 🚀 Xdebug OFF, JIT ON  — realistic performance numbers
wfl status   # show which profile is currently active
```

Use `wfl debug` while writing and stepping through code; switch to `wfl bench` before benchmarking or profiling so Xdebug's overhead doesn't skew the numbers.

## A typical cross-component session

```bash
wfl up                        # boot the stack
wfl debug                     # Xdebug on while we work

wfl link security utils       # security will build against local utils
# … edit utils/src and security/src …
wfl test utils                # green here first
wfl test security             # then verify the consumer
wfl mago security             # fmt + lint + analyze + guard, zero baselines

wfl unlink security utils     # restore the registry dependency
wfl bench                     # back to JIT for a perf sanity check
```

## Notes & guardrails

- `wfl link` / `unlink` validate that both components exist and have a `composer.json`; an unknown component aborts with an error.
- The dependency **perimeter** still applies: `mago guard` only permits the dependencies declared in each component's `mago.toml`. Linking `utils` into `routing` works because `routing`'s perimeter already permits `Waffle\Commons\Utils\**`; linking a provider a component is *not* permitted to depend on will still fail `wfl mago`.
- Never commit a `path` repository entry. The link is a local-only convenience — always `wfl unlink` before committing the consumer's `composer.json`.

## See also

- [`wfl` reference](../reference/wfl.md)
- [Quick Start](../tutorials/quick-start.md)
