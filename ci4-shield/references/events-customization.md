# Shield Events, Custom Views, Extending Controllers, Routes & Testing

## Events

Shield fires events you can hook into via `app/Config/Events.php`:

```php
<?php
// app/Config/Events.php

use CodeIgniter\Events\Events;

// New user registered
Events::on('register', static function ($user) {
    log_message('info', "New user registered: {$user->email}");

    // Send welcome email
    $email = \Config\Services::email();
    $email->setTo($user->email);
    $email->setSubject('Welcome to Our App');
    $email->setMessage("Hello {$user->username}, welcome!");
    $email->send();
});

// Successful login
Events::on('login', static function ($user) {
    log_message('info', "User logged in: {$user->email}");

    // Update last login timestamp
    model('UserModel')->update($user->id, [
        'last_login' => date('Y-m-d H:i:s'),
    ]);
});

// Failed login attempt
Events::on('failedLogin', static function ($credentials) {
    // $credentials contains email/username but NOT the password
    $identifier = $credentials['email'] ?? $credentials['username'] ?? 'unknown';
    log_message('warning', "Failed login attempt for: {$identifier}");
});

// Logout
Events::on('logout', static function ($user) {
    log_message('info', "User logged out: {$user->email}");
});

// Magic link login
Events::on('magicLogin', static function () {
    $user = auth()->user();
    log_message('info', "Magic link login: {$user->email}");
});
```

### Available Events

| Event | Arguments | When |
|---|---|---|
| `register` | `$user` | After successful registration |
| `login` | `$user` | After successful login |
| `failedLogin` | `$credentials` (no password) | After failed login attempt |
| `logout` | `$user` | After logout |
| `magicLogin` | none | After magic link login |

---

## Custom Views

Override Shield's default views by changing paths in `Auth.php`'s `$views` array:

```php
// app/Config/Auth.php
public array $views = [
    'login'    => 'auth/login',      // app/Views/auth/login.php
    'register' => 'auth/register',   // app/Views/auth/register.php
    'layout'   => 'layouts/auth',    // layout for auth pages
];
```

### Registration View Example

```php
<!-- app/Views/auth/register.php -->
<?= $this->extend('layouts/main') ?>

<?= $this->section('content') ?>
<form action="<?= url_to('register') ?>" method="post">
    <?= csrf_field() ?>

    <?php if (session('error')): ?>
        <div class="alert alert-danger"><?= session('error') ?></div>
    <?php endif; ?>

    <div class="mb-3">
        <label for="username">Username</label>
        <input type="text" name="username" id="username"
               value="<?= old('username') ?>" required>
    </div>

    <div class="mb-3">
        <label for="email">Email</label>
        <input type="email" name="email" id="email"
               value="<?= old('email') ?>" required>
    </div>

    <div class="mb-3">
        <label for="password">Password</label>
        <input type="password" name="password" id="password" required>
    </div>

    <div class="mb-3">
        <label for="password_confirm">Confirm Password</label>
        <input type="password" name="password_confirm" id="password_confirm" required>
    </div>

    <button type="submit">Register</button>
</form>
<?= $this->endSection() ?>
```

---

## Extending Shield Controllers

You can extend Shield's built-in controllers to add custom behavior:

```php
<?php
namespace App\Controllers\Auth;

use CodeIgniter\Shield\Controllers\LoginController as ShieldLoginController;

class LoginController extends ShieldLoginController
{
    // Override the login view
    public function loginView()
    {
        // Custom logic before showing login
        return parent::loginView();
    }

    // Override login action
    public function loginAction()
    {
        // Custom pre-login logic (e.g., check maintenance mode)
        if (setting('App.maintenanceMode')) {
            return redirect()->to('/maintenance');
        }

        $result = parent::loginAction();

        // Custom post-login logic (e.g., audit trail)

        return $result;
    }
}

// Update routes to use your custom controller
$routes->get('login', 'Auth\LoginController::loginView');
$routes->post('login', 'Auth\LoginController::loginAction');
```

---

## Shield Routes

Shield registers its own routes automatically. View them with:

```bash
php spark routes
```

### Default Routes

| Route | Method | Purpose |
|---|---|---|
| `register` | GET/POST | Registration form + action |
| `login` | GET/POST | Login form + action |
| `logout` | GET | Logout |
| `login/magic-link` | GET/POST | Magic link form + action |
| `login/verify-magic-link` | GET | Magic link verification |
| `auth/a/show` | GET | Action (2FA/activation) code entry |
| `auth/a/handle` | POST | Action code verification |
| `auth/a/verify` | GET | Action email link verification |

### Customizing Routes

Define your own routes pointing to Shield's (or your extended) controllers:

```php
// app/Config/Routes.php
$routes->get('login', '\CodeIgniter\Shield\Controllers\LoginController::loginView');
$routes->post('login', '\CodeIgniter\Shield\Controllers\LoginController::loginAction');
$routes->get('register', '\CodeIgniter\Shield\Controllers\RegisterController::registerView');
$routes->post('register', '\CodeIgniter\Shield\Controllers\RegisterController::registerAction');
$routes->get('logout', '\CodeIgniter\Shield\Controllers\LoginController::logoutAction');
```

---

## Testing with Shield

### actingAs() — Authenticate for Tests

```php
use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\FeatureTestTrait;
use CodeIgniter\Test\DatabaseTestTrait;

class AdminTest extends CIUnitTestCase
{
    use FeatureTestTrait;
    use DatabaseTestTrait;

    protected $refresh = true;

    public function testAdminPageRequiresAuth(): void
    {
        $result = $this->get('admin');
        $result->assertRedirectTo(base_url('login'));
    }

    public function testAdminPageAccessible(): void
    {
        $user = $this->createTestUser();
        $user->addGroup('admin');

        $result = $this->actingAs($user)->get('admin');
        $result->assertStatus(200);
    }

    public function testRegularUserCannotAccessAdmin(): void
    {
        $user = $this->createTestUser();
        // user is in 'user' group by default

        $result = $this->actingAs($user)->get('admin');
        $result->assertStatus(403);  // or redirect, depending on filter
    }

    public function testApiWithToken(): void
    {
        $user  = $this->createTestUser();
        $token = $user->generateAccessToken('test');

        $result = $this->withHeaders([
            'Authorization' => 'Bearer ' . $token->raw_token,
        ])->get('api/v1/me');

        $result->assertStatus(200);
        $result->assertJSONFragment(['email' => 'test@example.com']);
    }

    private function createTestUser(): \CodeIgniter\Shield\Entities\User
    {
        $users = auth()->getProvider();
        $user  = new \CodeIgniter\Shield\Entities\User([
            'username' => 'testuser',
            'email'    => 'test@example.com',
            'password' => 'TestPass123!',
        ]);
        $users->save($user);
        $user = $users->findById($users->getInsertID());
        $users->addToDefaultGroup($user);
        return $user;
    }
}
```

---

## Common Gotchas

1. **`attempt()` returns a `Result`, not bool** — always use `$result->isOK()`, never `if ($result)`.

2. **`raw_token` is only available once** — after `generateAccessToken()`, the raw token is hashed. Capture immediately.

3. **Filter order matters** — `['session', 'group:admin']` is correct. Session must authenticate first, then group checks authorization.

4. **Parent route group filters don't merge into children** — each group must explicitly declare its filters.

5. **Custom UserModel must be registered** in `Auth.php`'s `$userProvider`. Without this, Shield uses its own model.

6. **Password auto-hashed by entity setter** — never manually hash before setting. You'll double-hash.

7. **Credentials live in `auth_identities`** — email/password are stored as identity records, not columns on the `users` table. Don't add email/password columns to your users migration.

8. **`$validFields` controls login fields** — if you want username login, add `'username'` to the array.

9. **Email config is required** for email activation, 2FA, and magic links. These features fail silently without it.

10. **`chain` filter is for dual-client endpoints** — tries session first, then tokens. Don't use when you know the auth type.

### Shield vs Laravel Auth

| Laravel | Shield |
|---|---|
| `Auth::check()` | `auth()->loggedIn()` |
| `Auth::user()` | `auth()->user()` |
| `Auth::id()` | `auth()->id()` |
| `Auth::attempt()` (returns bool) | `auth()->attempt()` (returns Result) |
| `$user->hasRole('admin')` | `$user->inGroup('admin')` |
| `$user->hasPermissionTo('x')` | `$user->can('x')` |
| `$request->user()` | `auth()->user()` |
| Middleware | Filters |
| Sanctum tokens | `$user->generateAccessToken()` |
| Guards | Authenticators |
| `@auth` Blade | `<?php if (auth()->loggedIn()): ?>` |
| `Auth::routes()` | Shield auto-registers routes |
