# Shield Token Authentication — Complete Reference

Covers Access Tokens, HMAC Tokens, and JWT.

---

## Access Token Authentication (API)

Stateless auth using Bearer tokens in the `Authorization` header. Ideal for APIs and mobile apps.

### Generating Tokens

```php
$user  = auth()->user();
$token = $user->generateAccessToken('Work Laptop');

// raw_token — plain-text token (ONLY available at creation time)
// Hashed before storage — capture and display immediately
$rawToken = $token->raw_token;

// Generate with specific scopes (permissions)
$token = $user->generateAccessToken('CI Server', ['posts.create', 'posts.edit']);
```

### Client Sends

```
Authorization: Bearer <raw_token>
Content-Type: application/json
```

### Authenticating Requests

```php
// In a controller protected by the 'tokens' filter
$user = auth('tokens')->user();
```

### Managing Tokens

```php
// All tokens for a user
$tokens = $user->accessTokens();

// Specific token by ID
$token = $user->getAccessToken($tokenId);

// Revoke (delete) one token
$user->revokeAccessToken($tokenId);

// Revoke all tokens
$user->revokeAllAccessTokens();
```

### Token Scopes (Permissions)

```php
// Generate with scopes
$token = $user->generateAccessToken('CI Server', ['posts.create', 'posts.edit']);

// Check scopes on the current token
$user->tokenCan('posts.create');    // true
$user->tokenCant('users.delete');   // true
```

### Mobile Login Example

```php
<?php
namespace App\Controllers\Auth;

use App\Controllers\BaseController;
use CodeIgniter\HTTP\ResponseInterface;

class LoginController extends BaseController
{
    public function mobileLogin(): ResponseInterface
    {
        $rules = [
            'email'       => 'required|valid_email',
            'password'    => 'required',
            'device_name' => 'required|string',
        ];

        if (! $this->validate($rules)) {
            return $this->response
                ->setJSON(['errors' => $this->validator->getErrors()])
                ->setStatusCode(422);
        }

        $result = auth()->attempt([
            'email'    => $this->request->getPost('email'),
            'password' => $this->request->getPost('password'),
        ]);

        if (! $result->isOK()) {
            return $this->response
                ->setJSON(['error' => $result->reason()])
                ->setStatusCode(401);
        }

        $token = auth()->user()->generateAccessToken(
            $this->request->getPost('device_name')
        );

        return $this->response
            ->setJSON(['token' => $token->raw_token]);
    }
}
```

### Token Lifecycle

1. **Generation**: `generateAccessToken('name')` creates token, returns with `raw_token`
2. **Storage**: Shield stores the **hashed** token in `auth_identities`. Raw token is NOT stored.
3. **Usage**: Client sends `Authorization: Bearer <raw_token>` on every request
4. **Validation**: `auth('tokens')->attempt()` hashes incoming token, matches against stored hashes
5. **Revocation**: `revokeAccessToken($id)` deletes the identity record

**CRITICAL**: `raw_token` is only available at creation time. If lost, generate a new token.

---

## HMAC SHA256 Authentication

HMAC auth uses a shared secret key to sign requests. More secure than Bearer tokens because the secret never travels over the wire — only a computed hash does.

### Generating HMAC Tokens

```php
$token = $user->generateHmacToken('Work Laptop');

// Both the key and secret are needed by the client
$key    = $token->raw_token;      // public key
$secret = $token->rawSecretKey;   // secret key — display once, stored hashed
```

### Client-Side Request Signing

The client computes the HMAC signature and sends it in the Authorization header:

```
Authorization: HMAC-SHA256 <token>:<hmac_hash>
```

Where `<hmac_hash>` is:
```php
$hash = hash_hmac('sha256', $requestBody, $secretKey);
```

### Managing HMAC Tokens

```php
$tokens = $user->hmacTokens();
$user->revokeHmacToken($tokenId);
$user->revokeAllHmacTokens();
```

---

## JWT Authentication

JWT (JSON Web Token) for stateless, standards-based authentication.

### Setup

1. Uncomment `'jwt' => JWT::class` in `Auth.php`'s `$authenticators` array
2. Configure in `app/Config/AuthToken.php`:

```php
public array $jwtConfig = [
    'algorithm'    => 'HS256',      // HS256, HS384, HS512, RS256, RS384, RS512
    'timeToLive'   => HOUR,         // Token expiration time
    'key'          => '',           // Secret key for HMAC algorithms
    'privateKey'   => '',           // Private key path for RSA
    'publicKey'    => '',           // Public key path for RSA
];
```

3. Use the `jwt` filter on routes:

```php
$routes->group('api', ['filter' => 'jwt'], static function ($routes) {
    $routes->get('profile', 'Api\ProfileController::index');
});
```

### JWT vs Access Tokens

| Feature | Access Tokens | JWT |
|---|---|---|
| Storage | Server-side (DB) | Client-side (stateless) |
| Revocation | Immediate (delete from DB) | Must wait for expiry (or maintain blacklist) |
| Payload | None (just an opaque token) | Can contain claims (user ID, roles, etc.) |
| DB lookup | Required per request | Not required (self-contained) |
| Best for | Simple APIs, revocable access | Microservices, third-party integrations |
