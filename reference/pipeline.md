# Pipeline Reference (`waffle-commons/pipeline`)

Provides a PSR-15 compliant middleware stack.

## `Waffle\Commons\Pipeline\MiddlewareStack`
The standard implementation of `MiddlewareStackInterface`.
- **Storage**: Stores middleware in a generic array.
- **Execution Order**: FIFO (First Added, First Executed if added via `prepend`; Last Executed if added via `add` in the stack flow).

*Correction on Execution Flow:*
- `add($m)`: Appends to end. Runs LAST before controller.
- `prepend($m)`: Puts at start. Runs FIRST.

## `Waffle\Commons\Pipeline\RequestHandler`
An internal class that iterates through the middleware stack. It maintains a pointer to current middleware and the next handler.
