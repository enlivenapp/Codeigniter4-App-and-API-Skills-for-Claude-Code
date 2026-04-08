# API Error Handling — Complete Reference

## HTTP Status Code Guide

| Status | BaseApiController Method | When to use |
|---|---|---|
| `200` | `success()` | Successful GET, PUT, PATCH |
| `201` | `created()` | Successful POST that created a resource |
| `204` | `noContent()` | Successful DELETE (no body returned) |
| `400` | `error(..., 400)` | Bad request — malformed JSON, missing required params |
| `401` | `unauthorized()` | No token / invalid token / expired token |
| `403` | `forbidden()` | Valid token, but insufficient permissions |
| `404` | `notFound()` | Resource does not exist |
| `409` | `conflict()` | State conflict — duplicate, already exists, can't transition |
| `422` | `validationError()` | Validation failed — field-level errors in `errors` object |
| `429` | `error(..., 429)` | Rate limit exceeded |
| `500` | `serverError()` | Unexpected internal error |

**Rule**: Never return `200` with an error message in the body. Clients check status codes first.

## Error Response Shape

```json
{
    "status": "error",
    "message": "Validation failed.",
    "data": null,
    "errors": {
        "email": "The email field is required.",
        "name": "The name field must be at least 2 characters."
    }
}
```

- `message` is always a single human-readable string
- `errors` is `null` unless it's a `422` validation error, in which case it's an object keyed by field name

## Global Exception Handler for APIs

Override CI4's exception handler to return JSON for API routes:

```php
<?php
// app/Config/Exceptions.php

namespace Config;

use CodeIgniter\Config\BaseConfig;
use CodeIgniter\Debug\ExceptionHandlerInterface;
use Throwable;

class Exceptions extends BaseConfig
{
    public function handler(int $statusCode, Throwable $exception): ExceptionHandlerInterface
    {
        // Use JSON handler for API routes
        $uri = service('request')->getUri()->getPath();
        if (str_starts_with($uri, 'api/')) {
            return new \App\Libraries\ApiExceptionHandler($this);
        }

        return new \CodeIgniter\Debug\ExceptionHandler($this);
    }
}
```

```php
<?php
// app/Libraries/ApiExceptionHandler.php

namespace App\Libraries;

use CodeIgniter\Debug\BaseExceptionHandler;
use CodeIgniter\Debug\ExceptionHandlerInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;
use Throwable;

class ApiExceptionHandler extends BaseExceptionHandler implements ExceptionHandlerInterface
{
    public function handle(
        Throwable $exception,
        RequestInterface $request,
        ResponseInterface $response,
        int $statusCode,
        int $exitCode,
    ): void {
        $message = ENVIRONMENT === 'production'
            ? 'An unexpected error occurred.'
            : $exception->getMessage();

        $body = [
            'status'  => 'error',
            'message' => $message,
            'data'    => null,
            'errors'  => null,
        ];

        // In development, include debug info
        if (ENVIRONMENT !== 'production') {
            $body['debug'] = [
                'exception' => get_class($exception),
                'file'      => $exception->getFile(),
                'line'      => $exception->getLine(),
            ];
        }

        $response->setStatusCode($statusCode)
            ->setJSON($body)
            ->send();

        exit($exitCode);
    }
}
```

## Common Error Patterns

### Not Found

```php
public function show($id = null): ResponseInterface
{
    $item = $this->model->find($id);
    if (!$item) {
        return $this->notFound('User not found.');
    }
    return $this->success($item);
}
```

### Validation Error

```php
if (!$this->validate($rules)) {
    return $this->validationError($this->validator->getErrors());
}
```

### Authorization Check

```php
$user = $this->requireAuth();
if ($user instanceof ResponseInterface) return $user;

if (!$user->can('posts.edit')) {
    return $this->forbidden('You do not have permission to edit posts.');
}
```

### Conflict (Duplicate)

```php
$existing = $this->model->where('slug', $body['slug'])->first();
if ($existing) {
    return $this->conflict('A resource with this slug already exists.');
}
```

### Try/Catch for External Services

```php
try {
    $result = $externalService->process($data);
    return $this->success($result);
} catch (\Exception $e) {
    log_message('error', 'External service failed: ' . $e->getMessage());
    return $this->serverError('External service unavailable. Please try again.');
}
```

---

## Common Pitfalls

1. **`requireAuth()` returns a union type** — it returns either a `User` or a `ResponseInterface` (on auth failure). Always check:
   ```php
   $user = $this->requireAuth();
   if ($user instanceof ResponseInterface) return $user;
   ```
   Calling `$user->id` without this check causes a fatal error when unauthenticated.

2. **`getBody()` returns `array`** — use `$body['key']`, not `$body->key`.

3. **Never type-hint override params** — `show($id = null)` not `show(int $id = null)`. Breaks ResourceController parent signature.

4. **`format()` name conflict** — `ResourceController` has a `protected format()` method. Never define `private function format()` in a subclass — PHP throws an access level conflict fatal error.

5. **Webhook routes must NOT have `api_auth` filter** — external services can't send your Bearer token.

6. **Always `return` responses** — `$this->success(...)` without `return` sends nothing.

7. **Stripe webhook: always return `200`** — even for event types you don't handle. Non-200 causes Stripe to retry indefinitely.

8. **Token `raw_token` is only available at generation time** — store it immediately or it's gone. The DB stores the hash, not the plain text.

9. **CORS preflight is an `OPTIONS` request** — handle it explicitly or browsers will block all cross-origin API calls before they reach your controllers.

10. **JSON body can be `null`** — `$this->request->getJSON(true)` returns `null` if the body isn't valid JSON. Always guard: `$body = is_array($body) ? $body : []`.

11. **Don't mix session and token auth on the same endpoint** — use the `chain` filter if an endpoint serves both web (session) and mobile (token) clients.

12. **CI4 validation uses request data by default** — when validating JSON body data, the body must be accessible via `$this->request->getPost()` or you need to use `$this->validateData($body, $rules)` for raw arrays.
