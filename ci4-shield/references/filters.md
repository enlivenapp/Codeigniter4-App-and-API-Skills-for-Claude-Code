# Shield Filters — Complete Reference

Shield provides pre-built filters that are **auto-registered** via the Registrar class. You do NOT need to add them to `app/Config/Filters.php` manually.

## Available Filters

| Filter | Purpose | Arguments |
|---|---|---|
| `session` | Requires session authentication. Redirects to login if not authed. | None |
| `tokens` | Requires Bearer token authentication. Returns 401 JSON if not authed. | None |
| `hmac` | Requires HMAC token authentication. | None |
| `jwt` | Requires JWT authentication. | None |
| `chain` | Tries authenticators in sequence (session first, then tokens). For SPAs + mobile. | None |
| `group` | Checks if user belongs to specified group(s). | Group names: `group:admin,superadmin` |
| `permission` | Checks if user has specified permission(s). | Permission names: `permission:users.edit` |
| `force-reset` | Checks if user must reset their password. Redirects to reset form. | None |
| `auth-rates` | Rate limiting for authentication routes. | None |

## Applying to Individual Routes

```php
// Session auth (web pages)
$routes->get('dashboard', 'DashboardController::index', ['filter' => 'session']);

// Token auth (API)
$routes->get('api/me', 'Api\UserController::me', ['filter' => 'tokens']);

// Chain filter — works for both session (SPA) and token (mobile) users
$routes->get('api/profile', 'Api\ProfileController::index', ['filter' => 'chain']);

// Group restriction (single group)
$routes->get('admin', 'AdminController::index', ['filter' => 'group:admin']);

// Group restriction (multiple groups — user must be in at least one)
$routes->get('admin', 'AdminController::index', ['filter' => 'group:admin,superadmin']);

// Permission restriction
$routes->get('admin/users', 'Admin\UsersController::index', ['filter' => 'permission:users.edit']);

// Combined: must be authed AND in admin group
$routes->get('admin', 'AdminController::index', ['filter' => ['session', 'group:admin']]);

// Combined: auth + group + permission
$routes->get('admin/settings', 'Admin\SettingsController::index', [
    'filter' => ['session', 'group:admin', 'permission:admin.settings']
]);
```

## Applying to Route Groups

```php
$routes->group('admin', ['filter' => ['session', 'group:admin,superadmin']], static function ($routes) {
    $routes->get('/', 'Admin\DashboardController::index');
    $routes->get('posts', 'Admin\PostsController::index');
    
    // Nested group with additional filter
    $routes->group('', ['filter' => 'permission:users.manage'], static function ($routes) {
        $routes->resource('users');
    });
});
```

**GOTCHA**: Filter options on parent route groups are **not** merged with child groups. Each group must specify its own filters explicitly.

```php
// WRONG — child does NOT inherit 'session' from parent
$routes->group('admin', ['filter' => 'session'], function ($routes) {
    $routes->group('users', function ($routes) {  // NO session filter!
        $routes->get('/', 'Admin\UsersController::index');
    });
});

// CORRECT — child explicitly declares its filters
$routes->group('admin', ['filter' => 'session'], function ($routes) {
    $routes->group('users', ['filter' => ['session', 'group:admin']], function ($routes) {
        $routes->get('/', 'Admin\UsersController::index');
    });
});
```

## Global Filter Application

```php
// app/Config/Filters.php — apply to URI patterns
public array $filters = [
    'session' => [
        'before' => ['admin/*', 'dashboard'],
    ],
    'group:admin' => [
        'before' => ['admin/*'],
    ],
    'auth-rates' => [
        'before' => ['login', 'register'],
    ],
];
```

## Filter Order Matters

When combining multiple filters, the order determines execution sequence:

```php
// CORRECT: auth first, then authorization
['filter' => ['session', 'group:admin']]

// WRONG: group check runs before auth — user is null, fails unexpectedly
['filter' => ['group:admin', 'session']]
```

1. Authentication filter runs first (`session`, `tokens`, `chain`) — establishes who the user is
2. Authorization filter runs second (`group`, `permission`) — checks what the user can do
3. Other filters (`force-reset`, `auth-rates`) — additional checks

## Force Password Reset

When `$user->forcePasswordReset()` has been called:

```php
// Apply force-reset filter to all protected routes
$routes->group('', ['filter' => ['session', 'force-reset']], static function ($routes) {
    $routes->get('dashboard', 'DashboardController::index');
    $routes->get('profile', 'ProfileController::index');
});
```

The filter redirects the user to the password reset form until they change their password.

## Rate Limiting Auth Routes

```php
// Apply to login/register to prevent brute force
$routes->post('login', '\CodeIgniter\Shield\Controllers\LoginController::loginAction', [
    'filter' => 'auth-rates'
]);
$routes->post('register', '\CodeIgniter\Shield\Controllers\RegisterController::registerAction', [
    'filter' => 'auth-rates'
]);
```

## Chain Filter Details

The `chain` filter tries authenticators in order and uses the first that succeeds:

1. First tries `session` — is the user logged in via session cookie?
2. If not, tries `tokens` — is there a valid Bearer token?
3. If neither succeeds, returns 401

Use `chain` when a single endpoint serves both:
- Web SPA clients (session auth via cookie)
- Mobile/API clients (Bearer token auth)

Do NOT use `chain` when you know the auth type — use the specific filter instead.
