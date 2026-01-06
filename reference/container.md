# Container Reference (`waffle-commons/container`)

A strict PSR-11 implementation supporting Autowiring.

## `Waffle\Commons\Container\Container`

### `get(string $id): mixed`
Retrieves a service.
1.  **Cache Hit**: Returns if already resolved.
2.  **Circular Dependency Check**: Throws exception if detected.
3.  **Build**:
    - If Closure: Executes it.
    - If Class Name: **Autowires** it via Reflection.

### Autowiring Logic
The container inspects the constructor:
- **Interfaces**: Resolves the class bound to the interface.
- **Concrete Classes**: Instantiates them recursively.
- **Primitives**:
    - If a default value exists, it uses it.
    - If it is nullable, it passes `null`.
    - Otherwise, throws `ContainerException`.

## `ServiceWithDefaultParam`
This is an internal tested concept ensuring that:
`public function __construct(int $port = 80)`
resolves successfully without configuration (injects `80`).
