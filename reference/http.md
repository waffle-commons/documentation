# HTTP Reference (`waffle-commons/http`)

Provides PSR-7 (Message) and PSR-17 (Factory) implementations.

## PSR-7 Objects
- `Waffle\Commons\Http\ServerRequest`: Represents an incoming server request.
- `Waffle\Commons\Http\Response`: Represents an outgoing response.
- `Waffle\Commons\Http\Stream`: Wraps PHP streams for body handling.
- `Waffle\Commons\Http\UploadedFile`: Manages `$_FILES`.

## PSR-17 Factories
- `Waffle\Commons\Http\Factory\ResponseFactory`: Creates `Response` instances.
- `Waffle\Commons\Http\Factory\StreamFactory`: Creates `Stream` instances.

The component is designed to be lightweight and strictly compliant with the PSRs.
