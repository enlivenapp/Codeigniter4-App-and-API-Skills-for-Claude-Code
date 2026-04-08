# API CORS — Complete Reference

For APIs consumed by browsers from a different origin, CORS headers are required.

## CI4 Built-in CORS (4.5+)

CI4 4.5+ includes a native CORS library. Configure in `app/Config/Cors.php`:

```php
<?php
namespace Config;

use CodeIgniter\Config\BaseConfig;

class Cors extends BaseConfig
{
    // Default CORS config
    public array $default = [
        'allowedOrigins'         => ['https://yourapp.com'],
        'allowedOriginsPatterns' => [],
        'supportsCredentials'    => false,
        'allowedHeaders'         => ['Authorization', 'Content-Type', 'X-Requested-With'],
        'exposedHeaders'         => [],
        'allowedMethods'         => ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
        'maxAge'                 => 7200,
    ];

    // Separate config for API routes (more permissive or restrictive)
    public array $api = [
        'allowedOrigins'         => ['https://app.example.com', 'https://admin.example.com'],
        'allowedOriginsPatterns' => [],
        'supportsCredentials'    => true,
        'allowedHeaders'         => ['Authorization', 'Content-Type'],
        'exposedHeaders'         => ['X-RateLimit-Limit', 'X-RateLimit-Remaining'],
        'allowedMethods'         => ['GET', 'POST', 'PUT', 'DELETE'],
        'maxAge'                 => 7200,
    ];
}
```

Apply via filter in `app/Config/Filters.php`:

```php
public array $filters = [
    'cors:api' => [
        'before' => ['api/*'],
        'after'  => ['api/*'],
    ],
];
```

## Custom CORS Filter (Manual)

For CI4 < 4.5 or full control:

```php
<?php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class CorsFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        // Handle preflight OPTIONS request
        if (strtolower($request->getMethod()) === 'options') {
            return service('response')
                ->setStatusCode(204)
                ->setHeader('Access-Control-Allow-Origin', $this->getAllowedOrigin($request))
                ->setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type, X-Requested-With')
                ->setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS')
                ->setHeader('Access-Control-Max-Age', '7200');
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
        return $response
            ->setHeader('Access-Control-Allow-Origin', $this->getAllowedOrigin($request))
            ->setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type, X-Requested-With')
            ->setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
    }

    /**
     * Check request origin against whitelist.
     */
    private function getAllowedOrigin(RequestInterface $request): string
    {
        $origin = $request->getHeaderLine('Origin');

        $allowed = [
            'https://yourapp.com',
            'https://admin.yourapp.com',
        ];

        // Development — allow localhost
        if (ENVIRONMENT === 'development') {
            $allowed[] = 'http://localhost:3000';
            $allowed[] = 'http://localhost:5173';
        }

        if (in_array($origin, $allowed)) {
            return $origin;
        }

        return $allowed[0];  // default
    }
}
```

## Register & Apply

```php
// app/Config/Filters.php
public array $aliases = [
    'cors' => \App\Filters\CorsFilter::class,
];

// Apply to API routes
public array $filters = [
    'cors' => [
        'before' => ['api/*'],
        'after'  => ['api/*'],
    ],
];
```

## Key Points

1. **Preflight** is an `OPTIONS` request the browser sends before the actual request. It must return `204` with the correct headers.
2. **`Access-Control-Allow-Origin`** — use specific origins in production, never `'*'` with credentials.
3. **`Access-Control-Allow-Credentials: true`** — required if the client sends cookies or auth headers. Cannot be used with `Origin: *`.
4. **`Access-Control-Max-Age`** — how long (seconds) the browser caches the preflight response. `7200` = 2 hours.
5. **Both `before` and `after`** — the CORS filter must run as both. `before` handles preflight, `after` adds headers to actual responses.
6. **Route for OPTIONS** — if you get 404 on preflight, ensure your route configuration doesn't block OPTIONS requests. The `before` filter returning a response prevents the 404.
