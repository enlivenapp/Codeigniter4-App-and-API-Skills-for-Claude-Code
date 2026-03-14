---
name: ci4-api
description: Building robust, portable REST APIs with CodeIgniter 4. Use when creating or working with CI4 API controllers, JSON responses, API versioning, token authentication, request validation, error handling, rate limiting, or webhooks. Activates on mentions of "CI4 API", "REST API", "API controller", "api/v1", "Bearer token", "webhook", or "JSON response" in a CI4 context.
version: 1.0.0
---

# CodeIgniter 4 — REST API Reference

A well-built CI4 API is stateless, versioned, consistently enveloped, and callable from any client — web, mobile, third-party — without breaking. This skill covers everything needed to build one properly.

> For core CI4 framework patterns (models, query builder, views, migrations, Spark CLI) see the `/ci4` skill.

---

## Guiding Principles

1. **Stateless** — every request must be self-contained. No session state. Auth via Bearer token on every request.
2. **Consistent envelope** — every response, success or error, uses the same JSON shape. Clients should never have to guess the structure.
3. **Versioned** — breaking changes get a new version. Existing clients never break.
4. **Explicit HTTP status codes** — `200`, `201`, `400`, `401`, `403`, `404`, `409`, `422`, `500`. Never return `200` with an error body.
5. **One source of truth** — the API owns the data. Web controllers, mobile apps, and third parties all call the API. No client bypasses it to hit the DB directly.

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
    ApiThrottleFilter.php       # Rate limiting (optional)
  Config/
    Routes.php                  # All routes including api/v1 group
```

---

## Versioning

### Recommended: URL Versioning
```
/api/v1/users
/api/v2/users    ← breaking change gets new version
```

Simple, visible, cacheable, works with every HTTP client without custom headers.

```php
// app/Config/Routes.php
$routes->group('api/v1', [
    'namespace' => 'App\Controllers\Api\V1',
    'filter'    => 'api_auth',
], function ($routes) {
    $routes->resource('users');
    $routes->resource('events');
    $routes->post('auth/login',  'AuthController::login');
    $routes->post('auth/logout', 'AuthController::logout');
});
```

### Alternative Approaches (no detail — choose one and be consistent)
- **Header versioning** — `Accept: application/vnd.myapp.v1+json`
- **Query string** — `/api/users?version=1` (least recommended — breaks caching)

---

## Response Envelope

Every API response — success or failure — uses the same shape. Clients can always rely on this structure.

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
| `message` | string | Human-readable summary. Always present. |
| `data` | object \| array \| null | Payload on success. `null` on error. |
| `errors` | object \| null | Validation errors keyed by field. `null` on success. |

---

## BaseApiController

All API controllers extend this. It provides response helpers, auth helpers, and body parsing. Define it once, use it everywhere.

```php
<?php
namespace App\Controllers\Api\V1;

use CodeIgniter\Controller;
use CodeIgniter\HTTP\ResponseInterface;

class BaseApiController extends Controller
{
    protected $format = 'json';

    // ─── Auth Helpers ────────────────────────────────────────────────

    /**
     * Returns the authenticated user (token auth) or aborts with 401.
     */
    protected function requireAuth(): \CodeIgniter\Shield\Entities\User
    {
        $user = auth('tokens')->user();
        if (!$user) {
            $this->forbidden();  // exits
        }
        return $user;
    }

    protected function apiUser(): ?\CodeIgniter\Shield\Entities\User
    {
        return auth('tokens')->user();
    }

    // ─── Request Helpers ─────────────────────────────────────────────

    /**
     * Parse JSON body. Always returns array.
     */
    protected function getBody(): array
    {
        $body = $this->request->getJSON(true);
        return is_array($body) ? $body : [];
    }

    // ─── Response Helpers ────────────────────────────────────────────

    protected function success(mixed $data = null, string $message = 'OK', int $status = 200): ResponseInterface
    {
        return $this->response->setStatusCode($status)->setJSON([
            'status'  => 'success',
            'message' => $message,
            'data'    => $data,
            'errors'  => null,
        ]);
    }

    protected function created(mixed $data = null, string $message = 'Created.'): ResponseInterface
    {
        return $this->success($data, $message, 201);
    }

    protected function noContent(): ResponseInterface
    {
        return $this->response->setStatusCode(204);
    }

    protected function error(string $message, int $status = 400, mixed $errors = null): ResponseInterface
    {
        return $this->response->setStatusCode($status)->setJSON([
            'status'  => 'error',
            'message' => $message,
            'data'    => null,
            'errors'  => $errors,
        ]);
    }

    protected function validationError(array $errors, string $message = 'Validation failed.'): ResponseInterface
    {
        return $this->error($message, 422, $errors);
    }

    protected function notFound(string $message = 'Not found.'): ResponseInterface
    {
        return $this->error($message, 404);
    }

    protected function forbidden(string $message = 'Forbidden.'): ResponseInterface
    {
        return $this->error($message, 403);
    }

    protected function unauthorized(string $message = 'Unauthorized.'): ResponseInterface
    {
        return $this->error($message, 401);
    }

    protected function conflict(string $message = 'Conflict.'): ResponseInterface
    {
        return $this->error($message, 409);
    }

    protected function serverError(string $message = 'Server error.'): ResponseInterface
    {
        return $this->error($message, 500);
    }

    /**
     * Paginated list response — includes pagination metadata.
     */
    protected function paginated(array $data, object $pager, string $message = 'OK'): ResponseInterface
    {
        return $this->response->setStatusCode(200)->setJSON([
            'status'  => 'success',
            'message' => $message,
            'data'    => $data,
            'errors'  => null,
            'meta'    => [
                'current_page' => $pager->getCurrentPage(),
                'per_page'     => $pager->getPerPage(),
                'total'        => $pager->getTotal(),
                'page_count'   => $pager->getPageCount(),
            ],
        ]);
    }
}
```

---

## API Controller Pattern

```php
<?php
namespace App\Controllers\Api\V1;

use App\Models\UserModel;

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
        $users = $this->model->findAll();
        return $this->success($users, 'Users retrieved.');
    }

    // GET /api/v1/users/:id
    public function show($id = null): ResponseInterface  // never type-hint override params
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

## Routing

```php
// app/Config/Routes.php

$routes->group('api/v1', [
    'namespace' => 'App\Controllers\Api\V1',
], function ($routes) {

    // Public routes (no auth)
    $routes->post('auth/login',   'AuthController::login');
    $routes->post('auth/register','AuthController::register');

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

    // Webhook routes — NO auth filter (Stripe signs its own payloads)
    $routes->post('webhooks/stripe', 'StripeController::webhook');
});
```

---

## Authentication (Shield Token Auth)

### Login — Issue a Token
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

### Logout — Revoke Token
```php
// POST /api/v1/auth/logout
public function logout(): ResponseInterface
{
    $user = $this->requireAuth();
    // Revoke the current token
    $user->revokeAccessToken(auth('tokens')->getPayload());
    return $this->success(null, 'Logged out.');
}
```

### Me — Current User
```php
// GET /api/v1/auth/me
public function me(): ResponseInterface
{
    $user = $this->requireAuth();
    return $this->success([
        'id'     => $user->id,
        'name'   => $user->name,
        'email'  => $user->email,
        'groups' => $user->getGroups(),
    ], 'Authenticated user.');
}
```

### Client Usage
```
Authorization: Bearer <raw_token>
Content-Type: application/json
```

---

## Filters

### ApiAuthFilter — Token Validation
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

### ApiGroupFilter — Role Restriction
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
            if ($user->inGroup($group)) return; // allowed
        }

        return service('response')->setStatusCode(403)->setJSON([
            'status' => 'error', 'message' => 'Forbidden.',
            'data' => null, 'errors' => null,
        ]);
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

### Register Filters
```php
// app/Config/Filters.php
public array $aliases = [
    'api_auth'     => \App\Filters\ApiAuthFilter::class,
    'api_group'    => \App\Filters\ApiGroupFilter::class,
    'api_throttle' => \App\Filters\ApiThrottleFilter::class,
];
```

---

## Request Validation

### Rules Reference
```php
$rules = [
    'name'     => 'required|min_length[2]|max_length[150]',
    'email'    => 'required|valid_email|is_unique[users.email]',
    'age'      => 'required|integer|greater_than[0]',
    'role'     => 'required|in_list[admin,user,staff]',
    'avatar'   => 'if_exist|is_image[avatar]|max_size[avatar,2048]',
    // if_exist — only validate if the field is present (good for PATCH/PUT)
    // is_unique[table.column,ignore_field,ignore_value] — for update uniqueness
];

if (!$this->validate($rules)) {
    return $this->validationError($this->validator->getErrors());
}
```

### Custom Error Messages
```php
$rules = ['email' => 'required|valid_email'];
$messages = [
    'email' => [
        'required'    => 'Email address is required.',
        'valid_email' => 'Please provide a valid email address.',
    ],
];
if (!$this->validate($rules, $messages)) {
    return $this->validationError($this->validator->getErrors());
}
```

---

## Rate Limiting (Throttle)

CI4 has a built-in `Throttler` via the `IncomingRequest` service. Apply per-route or globally.

### Simple — Apply in Filter
```php
<?php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;
use CodeIgniter\Throttle\Throttler;

class ApiThrottleFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $throttler = service('throttler');

        // arguments[0] = max requests, arguments[1] = per seconds
        $maxRequests = (int) ($arguments[0] ?? 60);
        $perSeconds  = (int) ($arguments[1] ?? 60);

        // Key by IP — or by user ID for authenticated routes
        $key = 'api-' . $request->getIPAddress();

        if (!$throttler->check($key, $maxRequests, $perSeconds)) {
            return service('response')
                ->setStatusCode(429)
                ->setHeader('Retry-After', $throttler->getTokenTime())
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

### Apply to Routes
```php
// 60 requests per 60 seconds on all API routes
$routes->group('api/v1', ['filter' => 'api_throttle:60,60'], function ($routes) { ... });

// Stricter on auth endpoints (prevent brute force)
$routes->post('api/v1/auth/login', 'AuthController::login', ['filter' => 'api_throttle:5,60']);
```

### Without a Filter — Inline in Controller
```php
$throttler = service('throttler');
if (!$throttler->check(md5($this->request->getIPAddress()), 10, MINUTE)) {
    return $this->error('Too many requests.', 429);
}
```

---

## CORS

For APIs consumed by browsers from a different origin, set CORS headers. Add a filter that runs on all API routes.

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
        if ($request->getMethod() === 'options') {
            return service('response')
                ->setStatusCode(204)
                ->setHeader('Access-Control-Allow-Origin', '*')
                ->setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type, X-Requested-With')
                ->setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
        return $response
            ->setHeader('Access-Control-Allow-Origin', '*')
            ->setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type, X-Requested-With')
            ->setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
    }
}
```

> In production, replace `'*'` with a specific allowed origin: `'https://yourapp.com'`

---

## Webhooks (Receiving)

Webhooks are inbound HTTP POST requests from external services (Stripe, GitHub, etc.) triggered by events on their end. They are **not authenticated with your Bearer token** — they use their own signature verification.

### General Pattern
```php
// Routes — NO api_auth filter on webhook endpoints
$routes->post('api/v1/webhooks/stripe', 'StripeController::webhook');
$routes->post('api/v1/webhooks/github', 'GithubController::webhook');
```

### Stripe Webhook Example
```php
<?php
namespace App\Controllers\Api\V1;

class StripeController extends BaseApiController
{
    public function webhook(): ResponseInterface
    {
        $payload   = $this->request->getBody();
        $sigHeader = $this->request->getHeaderLine('Stripe-Signature');
        $secret    = env('STRIPE_WEBHOOK_SECRET');

        // Verify the signature — NEVER skip this
        try {
            $event = \Stripe\Webhook::constructEvent($payload, $sigHeader, $secret);
        } catch (\Stripe\Exception\SignatureVerificationException $e) {
            return $this->error('Invalid signature.', 400);
        } catch (\UnexpectedValueException $e) {
            return $this->error('Invalid payload.', 400);
        }

        // Route by event type
        switch ($event->type) {
            case 'payment_intent.succeeded':
                $this->handlePaymentSuccess($event->data->object);
                break;

            case 'payment_intent.payment_failed':
                $this->handlePaymentFailed($event->data->object);
                break;

            case 'customer.subscription.deleted':
                $this->handleSubscriptionCancelled($event->data->object);
                break;

            default:
                // Log unknown events but always return 200
                log_message('info', 'Unhandled Stripe event: ' . $event->type);
        }

        // Always return 200 — anything else causes Stripe to retry
        return $this->success(null, 'Webhook received.');
    }

    private function handlePaymentSuccess(object $paymentIntent): void
    {
        // Update order status, generate tickets, send confirmation email, etc.
    }

    private function handlePaymentFailed(object $paymentIntent): void
    {
        // Release reserved stock, notify user, etc.
    }
}
```

### Webhook Rules
- **Always return `200`** even for unhandled event types. Non-200 responses cause retries.
- **Always verify the signature** before processing anything.
- **Process async when possible** — queue heavy work, return `200` immediately.
- **Log everything** — webhook failures are hard to debug without a trail.
- **Idempotency** — webhooks can be delivered more than once. Check if you've already processed an event ID before acting.

```php
// Idempotency check example
$eventId = $event->id;
if ($this->processedWebhookModel->find($eventId)) {
    return $this->success(null, 'Already processed.');
}
$this->processedWebhookModel->insert(['id' => $eventId, 'processed_at' => date('Y-m-d H:i:s')]);
```

---

## Sending Webhooks (Outbound)

If your app needs to notify third parties of events:

```php
<?php
namespace App\Services;

class WebhookService
{
    public function dispatch(string $url, string $secret, string $event, array $payload): bool
    {
        $body      = json_encode(['event' => $event, 'data' => $payload]);
        $signature = hash_hmac('sha256', $body, $secret);

        $client   = \Config\Services::curlrequest();
        $response = $client->post($url, [
            'headers' => [
                'Content-Type'     => 'application/json',
                'X-Signature'      => $signature,
                'X-Event'          => $event,
            ],
            'body'    => $body,
            'timeout' => 10,
        ]);

        return $response->getStatusCode() === 200;
    }
}
```

---

## Pagination

```php
// Controller
public function index(): ResponseInterface
{
    $perPage = (int) ($this->request->getGet('per_page') ?? 20);
    $users   = $this->model->paginate($perPage);
    $pager   = $this->model->pager;

    return $this->paginated($users, $pager, 'Users retrieved.');
}
```

Response shape:
```json
{
    "status": "success",
    "message": "Users retrieved.",
    "data": [ ... ],
    "errors": null,
    "meta": {
        "current_page": 1,
        "per_page": 20,
        "total": 143,
        "page_count": 8
    }
}
```

Client navigates with `?page=2&per_page=20`.

---

## Error Handling — HTTP Status Code Guide

| Status | Method | When to use |
|---|---|---|
| `200` | `success()` | Successful GET, PUT, PATCH |
| `201` | `created()` | Successful POST that created a resource |
| `204` | `noContent()` | Successful DELETE (no body) |
| `400` | `error(..., 400)` | Bad request — malformed JSON, missing required params |
| `401` | `unauthorized()` | No token / invalid token |
| `403` | `forbidden()` | Valid token, insufficient permissions |
| `404` | `notFound()` | Resource does not exist |
| `409` | `conflict()` | State conflict — duplicate, already exists, can't transition |
| `422` | `validationError()` | Validation failed — field-level errors |
| `429` | `error(..., 429)` | Rate limit exceeded |
| `500` | `serverError()` | Unexpected internal error |

**Never return `200` with an error message in the body.** Clients check status codes first.

---

## Common Pitfalls

- **`getBody()` returns `array`** — use `$body['key']`, not `$body->key`.
- **Never type-hint override params** — `show($id = null)` not `show(int $id = null)`. Breaks ResourceController parent signature.
- **`format()` name conflict** — `ResourceController` has a `protected format()` method. Never define `private function format()` in a subclass — PHP throws an access level conflict fatal error.
- **Webhook routes must NOT have `api_auth` filter** — external services can't send your Bearer token.
- **Always `return` responses** — `$this->success(...)` without `return` sends nothing.
- **Stripe webhook: always return `200`** — even for event types you don't handle. Non-200 causes Stripe to retry indefinitely.
- **Token `raw_token` is only available at generation time** — store it immediately or it's gone. The DB stores the hash, not the plain text.
- **CORS preflight is an `OPTIONS` request** — handle it explicitly or browsers will block all cross-origin API calls before they reach your controllers.
