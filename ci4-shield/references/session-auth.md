# Shield Session Authentication — Complete Reference

Standard login/logout flow for browser-based (web) applications.

## Login Flow

```php
// In a controller
public function loginAction()
{
    $rules = [
        'email'    => 'required|valid_email',
        'password' => 'required',
    ];

    if (! $this->validate($rules)) {
        return redirect()->back()->withInput()->with('errors', $this->validator->getErrors());
    }

    $result = auth()->attempt([
        'email'    => $this->request->getPost('email'),
        'password' => $this->request->getPost('password'),
    ]);

    if (! $result->isOK()) {
        return redirect()->back()->with('error', $result->reason());
    }

    return redirect()->to(config('Auth')->redirects['login']);
}
```

## Logout

```php
public function logout()
{
    auth()->logout();
    return redirect()->to(config('Auth')->redirects['logout']);
}
```

## Remember Me

Configured in `Auth.php`:

```php
public array $sessionConfig = [
    'field'              => 'user',
    'allowRemembering'   => true,
    'rememberCookieName' => 'remember',
    'rememberLength'     => 30 * DAY,
];
```

When a user checks "Remember me" on the login form, Shield:
1. Stores a token in `auth_remember_tokens`
2. Sets a cookie with the configured name
3. On return visits, the cookie automatically re-authenticates the session

The "Remember me" checkbox should be named `'remember'` — Shield handles it automatically during `attempt()`.

## Checking Auth State

```php
// Is anyone logged in?
auth()->loggedIn();        // bool

// Get current user
$user = auth()->user();    // User entity or null

// Get current user ID
$id = auth()->id();        // int or null

// Specify the session authenticator explicitly
auth('session')->loggedIn();
auth('session')->user();
```

## Auth Attempt Result

`auth()->attempt()` returns a `Result` object, **not** a boolean:

```php
$result = auth()->attempt($credentials);

$result->isOK();       // bool — did auth succeed?
$result->reason();     // string — error message if failed
$result->extraInfo();  // mixed — additional info (rare)
```

**GOTCHA**: Never use `if ($result)` — `Result` is always truthy. Always use `$result->isOK()`.

## Check Credentials Without Login

```php
$result = auth()->check([
    'email'    => $email,
    'password' => $password,
]);

if ($result->isOK()) {
    // Credentials are valid, but user is NOT logged in
}
```

## Login View Example

```php
<!-- app/Views/auth/login.php -->
<?= $this->extend('layouts/main') ?>

<?= $this->section('content') ?>
<form action="<?= url_to('login') ?>" method="post">
    <?= csrf_field() ?>

    <?php if (session('error')): ?>
        <div class="alert alert-danger"><?= session('error') ?></div>
    <?php endif; ?>

    <div class="mb-3">
        <label for="email">Email</label>
        <input type="email" name="email" id="email"
               value="<?= old('email') ?>" required>
    </div>

    <div class="mb-3">
        <label for="password">Password</label>
        <input type="password" name="password" id="password" required>
    </div>

    <?php if (config('Auth')->sessionConfig['allowRemembering']): ?>
    <div class="mb-3">
        <label>
            <input type="checkbox" name="remember" value="1"> Remember me
        </label>
    </div>
    <?php endif; ?>

    <button type="submit">Login</button>

    <p><a href="<?= url_to('magic-link') ?>">Forgot password?</a></p>
</form>
<?= $this->endSection() ?>
```

## Checking Auth in Views

```php
<?php if (auth()->loggedIn()): ?>
    <p>Welcome, <?= esc(auth()->user()->username) ?></p>
    <a href="<?= url_to('logout') ?>">Logout</a>
<?php else: ?>
    <a href="<?= url_to('login') ?>">Login</a>
<?php endif; ?>

<?php if (auth()->user()?->inGroup('admin')): ?>
    <a href="/admin">Admin Panel</a>
<?php endif; ?>
```
