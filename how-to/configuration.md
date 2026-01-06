# How to Manage Configuration

Waffle allows you to manage settings using YAML files and Environment variables.

## Adding a Configuration Key

Open `config/app.yaml` and add your custom configuration.

```yaml
# config/app.yaml
myapp:
  api_key: '%env(MY_API_KEY)%'
  timeout: 30
```

*Note: Use `%env(VAR)%` to inject environment variables securely.*

## Injecting Configuration into a Service

You can inject the core `ConfigInterface` into any service to access these values.

```php
namespace App\Service;

use Waffle\Commons\Contracts\Config\ConfigInterface;

class ApiClient
{
    private string $apiKey;

    public function __construct(ConfigInterface $config)
    {
        // Enforce type safety
        $this->apiKey = $config->getString('myapp.api_key');
        
        // Optional with default value
        $timeout = $config->getInt('myapp.timeout', 5);
    }
}
```

The Container will automatically inject the active configuration instance when `ApiClient` is requested.
