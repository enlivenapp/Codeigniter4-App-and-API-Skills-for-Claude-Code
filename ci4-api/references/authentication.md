# API Authentication — Complete Reference

## ApiAuthFilter — Token Validation

```php
<?php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class ApiAuthFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $result = auth('tokens')->attempt();

        if (!$result->isOK()) {
            return service('response')
                ->setStatusCode(401)
                ->setJSON([
                    'status'  => 'error',
                    'message' => 'Unauthorized.',
                    'data'    => null,
                    'errors'  => null,
                ]);
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

## ApiGroupFilter — Role Restriction

```php
<?php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class ApiGroupFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $user = auth('tokens')->user();
        if (!$user) {
            return service('response')->setStatusCode(401)->setJSON([
                'status' => 'error', 'message' => 'Unauthorized.',
                'data' => null, 'errors' => null,
            ]);
        }

        // $arguments = ['admin', 'superadmin'] from filter: 'api_group:admin,superadmin'
        foreach ((array) $arguments as $group) {
            if ($user->inGroup($group)) return;  // allowed
        }

        return service('response')->setStatusCode(403)->setJSON([
            'status' => 'error', 'message' => 'Forbidden.',
            'data' => null, 'errors' => null,
        ]);
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

## Register Filters

```php
// app/Config/Filters.php
public array $aliases = [
    'api_auth'     => \App\Filters\ApiAuthFilter::class,
    'api_group'    => \App\Filters\ApiGroupFilter::class,
    'api_throttle' => \App\Filters\ApiThrottleFilter::class,
];
```

## Login — Issue a Token

```php
// POST /api/v1/auth/login
public function login(): ResponseInterface
{
    $body = $this->getBody();

    $rules = [
        'email'    => 'required|valid_email',
        'password' => 'required',
    ];

    if (!$this->validate($rules)) {
        return $this->validationError($this->validator->getErrors());
    }

    $result = auth('tokens')->attempt([
        'email'    => $body['email'],
        'password' => $body['password'],
    ]);

    if (!$result->isOK()) {
        return $this->unauthorized('Invalid credentials.');
    }

    $user  = auth('tokens')->user();
    $token = $user->generateAccessToken('api');

    return $this->success([
        'token' => $token->raw_token,   // plain-text — only available on creation
        'user'  => [
            'id'    => $user->id,
            'name'  => $user->name,
            'email' => $user->email,
        ],
    ], 'Login successful.');
}
```

## Register — Create Account + Token

```php
// POST /api/v1/auth/register
public function register(): ResponseInterface
{
    $body = $this->getBody();

    $rules = [
        'username' => 'required|min_length[3]|max_length[30]|is_unique[users.username]',
        'email'    => 'required|valid_email|is_unique[auth_identities.secret]',
        'password' => 'required|min_length[8]',
    ];

    if (!$this->validate($rules)) {
        return $this->validationError($this->validator->getErrors());
    }

    $users = auth()->getProvider();
    $user  = new \CodeIgniter\Shield\Entities\User([
        'username' => $body['username'],
        'email'    => $body['email'],
        'password' => $body['password'],
    ]);

    $users->save($user);
    $user = $users->findById($users->getInsertID());
    $users->addToDefaultGroup($user);

    $token = $user->generateAccessToken('api');

    return $this->created([
        'token' => $token->raw_token,
        'user'  => [
            'id'       => $user->id,
            'username' => $user->username,
            'email'    => $user->email,
        ],
    ], 'Account created.');
}
```

## Logout — Revoke Token

```php
// POST /api/v1/auth/logout
public function logout(): ResponseInterface
{
    $user = $this->requireAuth();
    if ($user instanceof ResponseInterface) return $user;

    // Revoke the current token
    $user->revokeAccessToken(auth('tokens')->getPayload());

    return $this->success(null, 'Logged out.');
}
```

## Me — Current User

```php
// GET /api/v1/auth/me
public function me(): ResponseInterface
{
    $user = $this->requireAuth();
    if ($user instanceof ResponseInterface) return $user;

    return $this->success([
        'id'     => $user->id,
        'name'   => $user->name,
        'email'  => $user->email,
        'groups' => $user->getGroups(),
    ], 'Authenticated user.');
}
```

## Client Usage

```
Authorization: Bearer <raw_token>
Content-Type: application/json
```

## Token Lifecycle

1. **Generation**: `$user->generateAccessToken('name')` creates a token and returns it with `raw_token`
2. **Storage**: Shield stores the **hashed** token in `auth_identities`. The raw token is NOT stored.
3. **Usage**: Client sends `Authorization: Bearer <raw_token>` on every request
4. **Validation**: `auth('tokens')->attempt()` hashes the incoming token and matches against stored hashes
5. **Revocation**: `$user->revokeAccessToken($tokenId)` deletes the token record

**CRITICAL**: `raw_token` is only available at creation time. If lost, generate a new token.

## Token Scopes (Permissions)

```php
// Generate with specific scopes
$token = $user->generateAccessToken('CI Server', ['posts.create', 'posts.edit']);

// Check scopes
$user->tokenCan('posts.create');    // true
$user->tokenCant('users.delete');   // true
```

## Route Configuration

```php
// app/Config/Routes.php
$routes->group('api/v1', [
    'namespace' => 'App\Controllers\Api\V1',
], function ($routes) {

    // Public routes (no auth)
    $routes->post('auth/login',    'AuthController::login');
    $routes->post('auth/register', 'AuthController::register');

    // Authenticated routes
    $routes->group('', ['filter' => 'api_auth'], function ($routes) {
        $routes->get('auth/me',      'AuthController::me');
        $routes->post('auth/logout', 'AuthController::logout');
        $routes->resource('users');
        $routes->resource('events');
    });

    // Admin-only routes
    $routes->group('', ['filter' => ['api_auth', 'api_group:admin']], function ($routes) {
        $routes->get('admin/stats', 'AdminController::stats');
    });

    // Webhook routes — NO auth filter
    $routes->post('webhooks/stripe', 'StripeController::webhook');
});
```
