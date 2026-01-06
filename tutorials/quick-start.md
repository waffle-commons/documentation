# Your First Waffle Application

Welcome to Waffle! This tutorial will guide you through creating your first API using the Waffle Framework.

## Prerequisites

- **PHP 8.5+**
- **Composer**
- **Docker** & Docker Compose

## 1. Installation

The easiest way to start is using the official skeleton.

```bash
composer create-project waffle-commons/skeleton my-waffle-app
```

## 2. Start the Environment

Waffle provides a Docker configuration optimized for development.

1.  Navigate to your project:
    ```bash
    cd my-waffle-app
    ```
2.  Start the containers:
    ```bash
    docker compose up -d
    ```

Your API is now running at `http://localhost`.

## 3. Create a Controller

Let's create a simple "Hello World" endpoint. Create a new file `src/Controller/HelloController.php`:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Psr\Http\Message\ResponseInterface;
use Waffle\Commons\Routing\Attribute\Argument;
use Waffle\Commons\Routing\Attribute\Route;
use Waffle\Core\BaseController;

final class HelloController extends BaseController
{
    use JsonTrait;

    #[Route(
        path: 'hello/{name}',
        name: 'hello',
        arguments: [
            new Argument(classType: 'string', paramName: 'name', required: false),
        ],
    )]
    public function index(string $name): ResponseInterface
    {
        return $this->jsonResponse(data: [
            'message' => "Hello, $name!",
            'framework' => 'Waffle v0.1.0-alpha4'
        ]);
    }
}
```

## 4. Test It

Open your terminal and request the endpoint:

```bash
curl http://localhost/hello/Waffles
```

**Output:**
```json
{
  "message": "Hello, Waffles!",
  "framework": "Waffle v0.1.0-alpha4"
}
```
