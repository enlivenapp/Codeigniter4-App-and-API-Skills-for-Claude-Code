---
name: ci4-shield
description: Comprehensive CodeIgniter 4 Shield authentication and authorization skill. Use when working with Shield auth — session login, access tokens, HMAC tokens, JWT, groups, permissions, user model/entity, filters, email activation, two-factor auth, magic links, banning, force password reset, or customizing auth views/controllers. Activates on mentions of "Shield", "auth()", "loggedIn", "groups", "permissions", "access token", "HMAC", "JWT", "Email2FA", "EmailActivator", "magic link", "Shield filter", or any Shield-specific pattern in a CI4 context.
version: 2.0.0
---

# CodeIgniter 4 Shield — Auth Reference

Shield is the **official** authentication and authorization library for CodeIgniter 4. It is **not Laravel Sanctum, Passport, or Breeze**. Do not apply Laravel auth patterns here.

> **Related skills**: `ci4` for core framework patterns, `ci4-api` for REST API patterns.

## Reference Documents

For deep dives, read the relevant reference from `references/`:

| Reference | When to read |
|---|---|
| `references/configuration.md` | Auth.php, AuthGroups.php, password validators, views, authenticators |
| `references/session-auth.md` | Login/logout flow, remember me, web authentication |
| `references/token-auth.md` | Access tokens, HMAC tokens, JWT — generation, revocation, scopes |
| `references/groups-permissions.md` | Groups, permissions, matrix, direct user permissions, wildcards |
| `references/user-model.md` | User entity, UserModel, extending, creating/finding/updating users |
| `references/filters.md` | All Shield filters, route protection, filter arguments |
| `references/actions.md` | Email activation, Email 2FA, magic links, password handling, banning |
| `references/events-customization.md` | Events, custom views, extending controllers, routes, testing |

---

## Installation & Setup

```bash
composer require codeigniter4/shield
php spark shield:setup   # publishes Config files + migrations
php spark migrate        # creates Shield's database tables
```

### Database Tables

| Table | Purpose |
|---|---|
| `users` | Core user data (username, active, last_active) |
| `auth_identities` | All credential types (email/password, access tokens, HMAC keys) |
| `auth_logins` | Login attempt log (success + failure) |
| `auth_remember_tokens` | Remember-me tokens |
| `auth_groups_users` | User-to-group pivot |
| `auth_permissions_users` | User-to-permission pivot |

---

## Auth Helper — Core Functions

The `auth()` helper is globally available. No manual loading needed.

```php
auth()->loggedIn();         // bool — is someone logged in?
auth()->user();             // User entity or null
auth()->id();               // int user ID or null

// Specify authenticator
auth('session')->loggedIn();
auth('tokens')->user();

// Attempt login
$result = auth()->attempt([
    'email'    => $email,
    'password' => $password,
]);
if ($result->isOK()) { /* success */ }
// $result->reason() — error message on failure

// Logout
auth()->logout();

// Check credentials without logging in
$result = auth()->check(['email' => $email, 'password' => $password]);
```

**GOTCHA**: `attempt()` returns a `Result` object, not a boolean. Always check `$result->isOK()`.

---

## Groups & Permissions (Quick Reference)

See `references/groups-permissions.md` for complete reference.

```php
$user = auth()->user();

// Groups
$user->addGroup('admin');
$user->removeGroup('admin');
$user->inGroup('admin');                // bool
$user->inGroup('admin', 'superadmin');  // true if in ANY
$user->getGroups();                     // ['admin', 'developer']

// Permissions
$user->can('posts.create');             // bool (via group matrix OR direct)
$user->cannot('users.delete');          // bool
$user->addPermission('admin.access');   // direct permission
$user->removePermission('admin.access');
```

Groups and permissions are defined in `app/Config/AuthGroups.php`.

---

## Filters (Quick Reference)

See `references/filters.md` for complete reference. Shield auto-registers these — no manual registration needed.

| Filter | Purpose |
|---|---|
| `session` | Requires session auth |
| `tokens` | Requires Bearer token auth |
| `hmac` | Requires HMAC token auth |
| `jwt` | Requires JWT auth |
| `chain` | Tries session, then tokens (SPA + mobile) |
| `group` | Checks group membership |
| `permission` | Checks permission |
| `force-reset` | Checks if password reset required |
| `auth-rates` | Rate limiting for auth routes |

```php
// Route protection
$routes->get('dashboard', 'DashboardController::index', ['filter' => 'session']);
$routes->get('admin', 'AdminController::index', ['filter' => ['session', 'group:admin,superadmin']]);
$routes->get('api/me', 'Api\UserController::me', ['filter' => 'tokens']);
```

---

## User Entity (Quick Reference)

See `references/user-model.md` for complete reference.

```php
$user = auth()->user();

$user->getEmail();                // email address
$user->username;                  // username
$user->password = 'new-pass';    // auto-hashed via setter

$user->isBanned();                // bool
$user->ban('Reason');             // ban user
$user->unBan();                   // remove ban

$user->isActivated();             // bool
$user->activate();                // manual activation

$user->forcePasswordReset();      // require change on next login

// Access tokens
$token = $user->generateAccessToken('name');
$user->revokeAccessToken($tokenId);
```

---

## Configuration (Quick Reference)

See `references/configuration.md` for complete reference.

```php
// app/Config/Auth.php — key settings
public array $redirects = ['register' => '/', 'login' => '/', 'logout' => 'login'];
public array $actions = [
    'register' => null,   // EmailActivator::class for email verification
    'login'    => null,   // Email2FA::class for two-factor auth
];
public array $validFields = ['email'];  // add 'username' for username login
public string $defaultAuthenticator = 'session';
```

```php
// app/Config/AuthGroups.php — groups + permissions
public array $groups = ['superadmin' => [...], 'admin' => [...], 'user' => [...]];
public string $defaultGroup = 'user';
public array $permissions = ['admin.access' => '...', 'users.create' => '...'];
public array $matrix = ['superadmin' => ['admin.*', 'users.*'], 'admin' => ['admin.access']];
```

---

## Key Gotchas

See `references/events-customization.md` for the complete list.

1. **`attempt()` returns a `Result`, not bool** — always use `$result->isOK()`
2. **`raw_token` only available once** — capture at generation, hashed before storage
3. **Filter order matters** — `session` must run before `group` (auth before authz)
4. **Parent route group filters don't merge into children** — declare on each group
5. **Custom UserModel must be registered** in `Auth.php`'s `$userProvider`
6. **Password auto-hashed by entity setter** — never manually hash before setting
7. **Credentials live in `auth_identities`** — not the `users` table
8. **Email config required** for activation, 2FA, and magic links
9. **`$validFields` controls login fields** — add `'username'` to allow username login
10. **`chain` filter is for dual-client endpoints** — don't use when auth type is known
