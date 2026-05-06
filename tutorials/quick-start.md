# Quick Start Guide

Welcome to the Waffle Framework! This guide will help you set up a new project and create your first Controller.

## Prerequisites

- PHP 8.5 or higher *(optional)*
- Composer
- Docker

## 1. Installation

Create a new project using Composer:

```bash
composer create-project waffle-commons/skeleton my-app
cd my-app
```

Start the development server using the built-in PHP server or Docker:

```bash
# Using Docker Compose
docker compose up -d
```

Your application is now running at `https://localhost/`.

## 2. Create a Controller

Let's create a simple "Hello World" controller. Create a file named `src/Controller/HelloController.php`:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Psr\Http\Message\ResponseInterface;
use Waffle\Commons\Routing\Attribute\Route;
use Waffle\Core\BaseController;

final class HelloController extends BaseController
{
    #[Route(path: '/hello/{name}', name: 'hello')]
    public function index(string $name): ResponseInterface
    {
        return $this->jsonResponse(data: ['message' => "Hello $name"]);
    }
}
```

> [!NOTE]
> We extend `Waffle\Core\BaseController` to access helper methods like `jsonResponse()`.
> Routes are defined using the `#[Route]` attribute from `Waffle\Commons\Routing\Attribute`.

## 3. Test Your Endpoint

Open your browser or use `curl` to visit the new endpoint:

```bash
curl https://localhost/hello/Waffle
```

**Expected Output:**

```json
{
  "message": "Hello Waffle!"
}
```

## 4. Bootstrapping the Kernel

The Kernel is the heart of your application. In Alpha 5, it requires an **Event Dispatcher** and a **Logger** to be fully functional.

These are typically configured in your Kernel Factory (`src/Factory/AppKernelFactory.php`):

```php
use Waffle\Commons\Log\StreamLogger;
use Waffle\Commons\EventDispatcher\Dispatcher\EventDispatcher;
use Waffle\Commons\EventDispatcher\Provider\ListenerProvider;

$logger = new StreamLogger(streamPath: 'php://stderr');
$dispatcher = new EventDispatcher(new ListenerProvider());

$kernel = new AppKernel();
$kernel->setLogger($logger);
$kernel->setEventDispatcher($dispatcher);
$kernel->setConfiguration($config);
$kernel->setSecurity($security);

$kernel->boot()->configure();
```

## 5. Understanding the Entry Point

The application entry point `public/index.php` leverages `WaffleRuntime` to orchestrate the lifecycle.

```php
// public/index.php
use Waffle\Commons\Runtime\WaffleRuntime;
use App\Factory\AppKernelFactory;

$kernel = AppKernelFactory::create($env, $debug);
$request = AppKernelFactory::createRequest();

$runtime = new WaffleRuntime();
$runtime->run($kernel, $request);
```

Congratulations! You have successfully built and configured your first Waffle application with full logging and event support.
