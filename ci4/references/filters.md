# CI4 Filters — Complete Reference

Filters run before and/or after a controller method. Defined in `app/Filters/` and registered in `app/Config/Filters.php`.

## Creating a Filter

```php
<?php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class AuthFilter implements FilterInterface
{
    /**
     * Runs before the controller.
     * Return nothing to continue; return a Response to stop the chain.
     */
    public function before(RequestInterface $request, $arguments = null)
    {
        if (!auth()->loggedIn()) {
            return redirect()->to('/login');
        }
    }

    /**
     * Runs after the controller.
     * Can modify $response or return nothing.
     */
    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
        // Optional post-processing
    }
}
```

## Registering Filters

```php
// app/Config/Filters.php

public array $aliases = [
    'session'    => \App\Filters\SessionAuthFilter::class,
    'api_auth'   => \App\Filters\ApiAuthFilter::class,
    'api_group'  => \App\Filters\ApiGroupFilter::class,
    'admin_auth' => \App\Filters\AdminFilter::class,
    'throttle'   => \App\Filters\ThrottleFilter::class,
];
```

## Applying Filters

### On Individual Routes

```php
// Single filter
$routes->get('dashboard', 'DashboardController::index', ['filter' => 'session']);

// Multiple filters (run in order)
$routes->get('admin', 'AdminController::index', ['filter' => ['session', 'group:admin']]);

// Filter with arguments (accessible as $arguments in the filter)
$routes->get('admin', 'AdminController::index', ['filter' => 'group:admin,superadmin']);
// $arguments = ['admin', 'superadmin'] in the filter's before() method
```

### On Route Groups

```php
$routes->group('admin', ['filter' => 'session'], function ($routes) {
    $routes->get('dashboard', 'Admin\DashboardController::index');
    $routes->get('users', 'Admin\UsersController::index');
});

// Nested groups — filters are NOT inherited/merged from parent
$routes->group('admin', ['filter' => 'session'], function ($routes) {
    $routes->get('/', 'Admin\DashboardController::index');
    
    // This group does NOT inherit 'session' from parent — must declare it
    $routes->group('users', ['filter' => ['session', 'group:admin']], function ($routes) {
        $routes->get('/', 'Admin\UsersController::index');
    });
});
```

**GOTCHA**: Filter options on parent route groups are **not** merged into child groups. Each group must specify its own filters explicitly.

### Global Filters

```php
// app/Config/Filters.php

// Run on every request
public array $globals = [
    'before' => [
        'csrf',
        'honeypot',
    ],
    'after' => [
        'toolbar',  // debug toolbar (only in development)
    ],
];

// Except certain routes
public array $globals = [
    'before' => [
        'csrf' => ['except' => ['api/*']],  // skip CSRF for API routes
    ],
];
```

### URI-Pattern Filters

```php
// app/Config/Filters.php

public array $filters = [
    'session' => [
        'before' => ['admin/*', 'dashboard'],
    ],
    'group:admin' => [
        'before' => ['admin/*'],
    ],
    'throttle' => [
        'before' => ['api/*'],
    ],
];
```

## Filter Arguments

```php
// Route definition
$routes->get('admin', 'AdminController::index', ['filter' => 'group:admin,superadmin']);

// In the filter
public function before(RequestInterface $request, $arguments = null)
{
    // $arguments = ['admin', 'superadmin']
    $user = auth()->user();
    
    foreach ((array) $arguments as $group) {
        if ($user->inGroup($group)) {
            return;  // allowed — continue to controller
        }
    }
    
    return redirect()->to('/forbidden');
}
```

## Filter Execution Order

1. Global `before` filters run first
2. Route-specific `before` filters run next (in the order specified)
3. Controller method executes
4. Route-specific `after` filters run
5. Global `after` filters run last

## Common Filter Patterns

### Maintenance Mode Filter

```php
class MaintenanceFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        if (setting('App.maintenanceMode') && !str_starts_with(current_url(), site_url('admin'))) {
            return service('response')->setStatusCode(503)->setBody(view('maintenance'));
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

### CORS Filter

```php
class CorsFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        if ($request->getMethod() === 'options') {
            return service('response')
                ->setStatusCode(204)
                ->setHeader('Access-Control-Allow-Origin', '*')
                ->setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type')
                ->setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
        return $response
            ->setHeader('Access-Control-Allow-Origin', '*')
            ->setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type')
            ->setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    }
}
```

### Rate Limiting Filter

```php
class ThrottleFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $throttler = service('throttler');
        $maxRequests = (int) ($arguments[0] ?? 60);
        $perSeconds  = (int) ($arguments[1] ?? 60);
        $key = 'throttle-' . $request->getIPAddress();

        if (!$throttler->check($key, $maxRequests, $perSeconds)) {
            return service('response')
                ->setStatusCode(429)
                ->setHeader('Retry-After', $throttler->getTokenTime())
                ->setJSON(['error' => 'Too many requests.']);
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

## Shield's Auto-Registered Filters

Shield provides these filters automatically (no manual registration needed):

| Filter | Purpose |
|---|---|
| `session` | Requires session authentication |
| `tokens` | Requires Bearer token authentication |
| `hmac` | Requires HMAC token authentication |
| `jwt` | Requires JWT authentication |
| `chain` | Tries authenticators in sequence |
| `group` | Checks group membership (e.g., `group:admin`) |
| `permission` | Checks permission (e.g., `permission:users.edit`) |
| `force-reset` | Checks if password reset is required |
| `auth-rates` | Rate limiting for auth routes |

See the `ci4-shield` skill for complete Shield filter documentation.
