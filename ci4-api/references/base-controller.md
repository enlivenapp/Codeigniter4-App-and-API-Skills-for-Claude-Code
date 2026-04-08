# BaseApiController — Complete Reference

All API controllers extend this. It provides response helpers, auth helpers, and body parsing.

## Full Implementation

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
     * Returns the authenticated user (token auth), or returns a 401 response.
     * Callers MUST check: if ($user instanceof ResponseInterface) return $user;
     */
    protected function requireAuth(): \CodeIgniter\Shield\Entities\User|ResponseInterface
    {
        $user = auth('tokens')->user();
        if (!$user) {
            return $this->unauthorized('Unauthorized.');
        }
        return $user;
    }

    /**
     * Returns the authenticated user or null (no error response).
     */
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

    // ─── Success Responses ───────────────────────────────────────────

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

    // ─── Error Responses ─────────────────────────────────────────────

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

    // ─── Pagination ──────────────────────────────────────────────────

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

## Usage Patterns

### requireAuth() — Union Type Pattern

```php
public function show($id = null): ResponseInterface
{
    $user = $this->requireAuth();
    if ($user instanceof ResponseInterface) return $user;
    
    // $user is now guaranteed to be a User entity
    $item = $this->model->find($id);
    return $this->success($item);
}
```

### Paginated Response Shape

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

### Extending the Base Controller

If your API needs additional helpers:

```php
class BaseApiController extends Controller
{
    // ... existing methods ...

    /**
     * Require user to be in at least one of the specified groups.
     */
    protected function requireGroup(string ...$groups): \CodeIgniter\Shield\Entities\User|ResponseInterface
    {
        $user = $this->requireAuth();
        if ($user instanceof ResponseInterface) return $user;

        if (!$user->inGroup(...$groups)) {
            return $this->forbidden('Insufficient permissions.');
        }

        return $user;
    }

    /**
     * Require user to have a specific permission.
     */
    protected function requirePermission(string $permission): \CodeIgniter\Shield\Entities\User|ResponseInterface
    {
        $user = $this->requireAuth();
        if ($user instanceof ResponseInterface) return $user;

        if (!$user->can($permission)) {
            return $this->forbidden("Missing permission: {$permission}");
        }

        return $user;
    }
}
```
