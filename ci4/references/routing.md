# CI4 Routing — Complete Reference

All routes defined in `app/Config/Routes.php`.

## HTTP Verb Routes

```php
$routes->get('users', 'UserController::index');
$routes->post('users', 'UserController::create');
$routes->put('users/(:num)', 'UserController::update/$1');
$routes->patch('users/(:num)', 'UserController::update/$1');
$routes->delete('users/(:num)', 'UserController::delete/$1');
$routes->match(['get', 'post'], 'form', 'FormController::index');
```

## Route Placeholders

| Placeholder | Matches | Regex |
|---|---|---|
| `(:num)` | Digits only | `[0-9]+` |
| `(:alpha)` | Alphabetic only | `[a-zA-Z]+` |
| `(:alphanum)` | Alphanumeric | `[a-zA-Z0-9]+` |
| `(:segment)` | URL segment (no slashes) | `[^/]+` |
| `(:any)` | Anything including slashes | `.+` |

### Custom Regex Placeholders
```php
$routes->get('products/([a-z]{2})/(:num)', 'Products::show/$1/$2');
```

## Route Groups

```php
$routes->group('admin', ['namespace' => 'App\Controllers\Admin'], function ($routes) {
    $routes->get('dashboard', 'DashboardController::index');
    $routes->get('users', 'UsersController::index');
    $routes->resource('events');
});

// Nested groups
$routes->group('api', function ($routes) {
    $routes->group('v1', ['namespace' => 'App\Controllers\Api\V1'], function ($routes) {
        $routes->resource('users');
    });
});
```

## Filters on Routes

```php
// Single filter
$routes->get('dashboard', 'DashboardController::index', ['filter' => 'session']);

// Multiple filters
$routes->get('admin', 'AdminController::index', ['filter' => ['session', 'group:admin']]);

// Filter with arguments
$routes->get('admin', 'AdminController::index', ['filter' => 'group:admin,superadmin']);

// Filter on a group
$routes->group('admin', ['filter' => 'session'], function ($routes) {
    $routes->get('/', 'Admin\DashboardController::index');
});
```

**GOTCHA**: Filter options on parent route groups are **not** merged with child groups. Each group must declare its own filters.

## Named Routes

```php
$routes->get('profile', 'ProfileController::index', ['as' => 'profile']);
$routes->get('users/(:num)', 'UserController::show/$1', ['as' => 'user.show']);

// Generate URL from named route
$url = url_to('profile');
$url = url_to('user.show', 42);    // /users/42
```

## Redirects

```php
$routes->addRedirect('old-path', 'new-path');
$routes->addRedirect('old/(:any)', 'new/$1');
```

## Resource Routes

```php
// Generates full CRUD routes
$routes->resource('photos');
// GET    /photos             → Photos::index()
// GET    /photos/new         → Photos::new()
// POST   /photos             → Photos::create()
// GET    /photos/(:segment)  → Photos::show($id)
// GET    /photos/(:segment)/edit → Photos::edit($id)
// PUT    /photos/(:segment)  → Photos::update($id)
// DELETE /photos/(:segment)  → Photos::delete($id)

// Limit methods
$routes->resource('photos', ['only' => ['index', 'show', 'create']]);
$routes->resource('photos', ['except' => ['new', 'edit']]);

// Custom controller
$routes->resource('photos', ['controller' => 'App\Controllers\Admin\PhotosController']);

// API-only resource (no new/edit views)
$routes->presenter('photos');
```

## CLI Routes

```php
$routes->cli('maintenance/on', 'MaintenanceController::enable');
$routes->cli('maintenance/off', 'MaintenanceController::disable');
```

## Route Configuration

```php
// Default namespace
$routes->setDefaultNamespace('App\Controllers');

// Default controller
$routes->setDefaultController('Home');

// Default method
$routes->setDefaultMethod('index');

// 404 override
$routes->set404Override(function () {
    return view('errors/custom_404');
});

// Auto-routing (disabled by default in CI4 — keep it off for security)
$routes->setAutoRoute(false);
```

## Route Priority

Routes are matched in the order they are defined. First match wins. Place more specific routes before generic ones:

```php
// CORRECT order
$routes->get('users/new', 'UserController::new');       // specific first
$routes->get('users/(:num)', 'UserController::show/$1'); // generic second

// WRONG order — 'new' would match (:num) pattern... wait, 'new' is not numeric
// But with (:segment):
$routes->get('users/(:segment)', 'UserController::show/$1'); // catches 'new' too!
$routes->get('users/new', 'UserController::new');             // never reached
```
