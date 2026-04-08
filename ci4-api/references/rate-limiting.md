# API Rate Limiting — Complete Reference

CI4 has a built-in `Throttler` via the services container.

## Throttle Filter

```php
<?php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class ApiThrottleFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $throttler = service('throttler');

        // arguments from route: 'api_throttle:60,60'
        $maxRequests = (int) ($arguments[0] ?? 60);
        $perSeconds  = (int) ($arguments[1] ?? 60);

        // Key by IP — or by user ID for authenticated routes
        $user = auth('tokens')->user();
        $key  = $user
            ? 'api-user-' . $user->id
            : 'api-ip-' . $request->getIPAddress();

        if (!$throttler->check($key, $maxRequests, $perSeconds)) {
            return service('response')
                ->setStatusCode(429)
                ->setHeader('Retry-After', (string) $throttler->getTokenTime())
                ->setHeader('X-RateLimit-Limit', (string) $maxRequests)
                ->setHeader('X-RateLimit-Remaining', '0')
                ->setJSON([
                    'status'  => 'error',
                    'message' => 'Too many requests. Please slow down.',
                    'data'    => null,
                    'errors'  => null,
                ]);
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

## Register Filter

```php
// app/Config/Filters.php
public array $aliases = [
    'api_throttle' => \App\Filters\ApiThrottleFilter::class,
];
```

## Apply to Routes

```php
// 60 requests per 60 seconds on all API routes
$routes->group('api/v1', ['filter' => 'api_throttle:60,60'], function ($routes) {
    // ...
});

// Stricter on auth endpoints (prevent brute force)
$routes->post('api/v1/auth/login', 'AuthController::login', [
    'filter' => 'api_throttle:5,60'   // 5 attempts per minute
]);

// Different limits for different route groups
$routes->group('api/v1', function ($routes) {
    // Read operations — generous
    $routes->group('', ['filter' => 'api_throttle:120,60'], function ($routes) {
        $routes->get('posts', 'PostsController::index');
        $routes->get('posts/(:num)', 'PostsController::show/$1');
    });

    // Write operations — stricter
    $routes->group('', ['filter' => 'api_throttle:30,60'], function ($routes) {
        $routes->post('posts', 'PostsController::create');
        $routes->put('posts/(:num)', 'PostsController::update/$1');
    });
});
```

## Inline Throttling (No Filter)

For fine-grained control within a controller method:

```php
public function search(): ResponseInterface
{
    $throttler = service('throttler');
    $key = 'search-' . $this->request->getIPAddress();

    if (!$throttler->check($key, 10, MINUTE)) {
        return $this->error('Too many search requests. Try again shortly.', 429);
    }

    // ... perform search
}
```

## Rate Limit Headers

Include rate limit info in responses so clients can self-regulate:

```php
// In BaseApiController or a response filter
protected function withRateLimitHeaders(ResponseInterface $response, string $key, int $limit): ResponseInterface
{
    $throttler = service('throttler');
    $remaining = max(0, $limit - $throttler->getTokenTime());

    return $response
        ->setHeader('X-RateLimit-Limit', (string) $limit)
        ->setHeader('X-RateLimit-Remaining', (string) $remaining);
}
```

## Throttler API

```php
$throttler = service('throttler');

// Check (returns true if allowed, false if rate-limited)
$throttler->check($key, $capacity, $seconds);

// Get time until next token is available (seconds)
$throttler->getTokenTime();

// Remove a throttle key
$throttler->remove($key);
```

## Constants for Time

CI4 provides time constants for readability:

```php
SECOND  // 1
MINUTE  // 60
HOUR    // 3600
DAY     // 86400
WEEK    // 604800
MONTH   // 2592000
YEAR    // 31536000
```
