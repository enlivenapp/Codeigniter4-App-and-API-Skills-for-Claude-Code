---
name: ci4-api
description: Building robust, portable REST APIs with CodeIgniter 4. Use when creating or working with CI4 API controllers, JSON responses, API versioning, token authentication, request validation, error handling, rate limiting, or webhooks. Activates on mentions of "CI4 API", "REST API", "API controller", "api/v1", "Bearer token", "webhook", or "JSON response" in a CI4 context.
version: 2.0.0
---

# CodeIgniter 4 — REST API Reference

A well-built CI4 API is stateless, versioned, consistently enveloped, and callable from any client — web, mobile, third-party — without breaking.

> **Related skills**: `ci4` for core framework patterns, `ci4-shield` for auth deep dives.

## Reference Documents

For deep dives, read the relevant reference from `references/`:

| Reference | When to read |
|---|---|
| `references/base-controller.md` | BaseApiController, response helpers, auth helpers, body parsing |
| `references/authentication.md` | Login/logout/me endpoints, token lifecycle, filters |
| `references/webhooks.md` | Receiving webhooks (Stripe, GitHub), sending webhooks, signature verification |
| `references/rate-limiting.md` | Throttle filter, per-route limits, inline throttling |
| `references/cors.md` | CORS filter, preflight handling, production config |
| `references/error-handling.md` | Status code guide, error envelope, common pitfalls |

---

## Guiding Principles

1. **Stateless** — every request is self-contained. No session state. Auth via Bearer token.
2. **Consistent envelope** — every response uses the same JSON shape.
3. **Versioned** — breaking changes get a new version. Existing clients never break.
4. **Explicit HTTP status codes** — `200`, `201`, `400`, `401`, `403`, `404`, `409`, `422`, `500`. Never return `200` with an error body.
5. **One source of truth** — the API owns the data. Web, mobile, and third parties all call the API.

---

## Directory Structure

```
app/
  Controllers/
    Api/
      V1/
        BaseApiController.php   # Base for all API controllers
        UsersController.php
        EventsController.php
        AuthController.php
        StripeController.php    # Webhook receiver
  Filters/
    ApiAuthFilter.php           # Bearer token validation
    ApiGroupFilter.php          # Group/role restriction
    ApiThrottleFilter.php       # Rate limiting
```

---

## Response Envelope

Every API response — success or failure — uses the same shape:

```json
{
    "status":  "success",
    "message": "User retrieved.",
    "data":    { ... },
    "errors":  null
}
```

| Field | Type | Notes |
|---|---|---|
| `status` | `"success"` \| `"error"` | Always present |
| `message` | string | Human-readable summary |
| `data` | object \| array \| null | Payload on success, `null` on error |
| `errors` | object \| null | Validation errors keyed by field, `null` on success |

For paginated responses, add a `meta` object (see references/base-controller.md).

---

## Versioning

### URL Versioning (Recommended)

```
/api/v1/users
/api/v2/users    ← breaking change gets new version
```

```php
// app/Config/Routes.php
$routes->group('api/v1', [
    'namespace' => 'App\Controllers\Api\V1',
], function ($routes) {
    // Public routes
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

    // Webhooks — NO auth filter
    $routes->post('webhooks/stripe', 'StripeController::webhook');
});
```

---

## API Controller Pattern

All API controllers extend `BaseApiController` (see `references/base-controller.md` for full implementation).

```php
<?php
namespace App\Controllers\Api\V1;

use App\Models\UserModel;
use CodeIgniter\HTTP\ResponseInterface;

class UsersController extends BaseApiController
{
    protected UserModel $model;

    public function __construct()
    {
        $this->model = new UserModel();
    }

    // GET /api/v1/users
    public function index(): ResponseInterface
    {
        $perPage = (int) ($this->request->getGet('per_page') ?? 20);
        $users   = $this->model->paginate($perPage);
        return $this->paginated($users, $this->model->pager, 'Users retrieved.');
    }

    // GET /api/v1/users/:id
    public function show($id = null): ResponseInterface
    {
        $user = $this->model->find($id);
        if (!$user) return $this->notFound('User not found.');
        return $this->success($user, 'User retrieved.');
    }

    // POST /api/v1/users
    public function create(): ResponseInterface
    {
        $body = $this->getBody();

        $rules = [
            'name'  => 'required|min_length[2]|max_length[150]',
            'email' => 'required|valid_email|is_unique[users.email]',
        ];

        if (!$this->validate($rules)) {
            return $this->validationError($this->validator->getErrors());
        }

        $id = $this->model->insert([
            'name'  => $body['name'],
            'email' => $body['email'],
        ]);

        if (!$id) return $this->serverError('Could not create user.');
        return $this->created($this->model->find($id), 'User created.');
    }

    // PUT /api/v1/users/:id
    public function update($id = null): ResponseInterface
    {
        $user = $this->model->find($id);
        if (!$user) return $this->notFound('User not found.');

        $body = $this->getBody();

        $rules = [
            'name'  => 'if_exist|min_length[2]|max_length[150]',
            'email' => "if_exist|valid_email|is_unique[users.email,id,{$id}]",
        ];

        if (!$this->validate($rules)) {
            return $this->validationError($this->validator->getErrors());
        }

        $this->model->update($id, $body);
        return $this->success($this->model->find($id), 'User updated.');
    }

    // DELETE /api/v1/users/:id
    public function delete($id = null): ResponseInterface
    {
        $user = $this->model->find($id);
        if (!$user) return $this->notFound('User not found.');

        $this->model->delete($id);
        return $this->noContent();
    }
}
```

---

## Validation

```php
$rules = [
    'name'     => 'required|min_length[2]|max_length[150]',
    'email'    => 'required|valid_email|is_unique[users.email]',
    'age'      => 'required|integer|greater_than[0]',
    'role'     => 'required|in_list[admin,user,staff]',
    // if_exist — only validate if the field is present (good for PUT/PATCH)
    'name'     => 'if_exist|min_length[2]|max_length[150]',
];

if (!$this->validate($rules)) {
    return $this->validationError($this->validator->getErrors());
}

// Custom error messages
$messages = [
    'email' => [
        'required'    => 'Email address is required.',
        'valid_email' => 'Please provide a valid email address.',
    ],
];
if (!$this->validate($rules, $messages)) { ... }
```

---

## Filters

### Register

```php
// app/Config/Filters.php
public array $aliases = [
    'api_auth'     => \App\Filters\ApiAuthFilter::class,
    'api_group'    => \App\Filters\ApiGroupFilter::class,
    'api_throttle' => \App\Filters\ApiThrottleFilter::class,
];
```

See `references/authentication.md` for filter implementations.

---

## Key Gotchas

See `references/error-handling.md` for the complete list.

1. **`requireAuth()` returns a union type** — always check: `if ($user instanceof ResponseInterface) return $user;`
2. **`getBody()` returns `array`** — use `$body['key']`, not `$body->key`
3. **Never type-hint override params** — `show($id = null)` not `show(int $id = null)`
4. **`format()` name conflict** — `ResourceController` has `protected format()`, never define your own
5. **Webhook routes must NOT have `api_auth` filter**
6. **Always `return` responses** — `$this->success(...)` without `return` sends nothing
7. **Token `raw_token` is only available once** — capture it at generation time
8. **CORS preflight is OPTIONS** — handle it or browsers block cross-origin calls
9. **Always return `200` from webhooks** — non-200 causes retries from services like Stripe
