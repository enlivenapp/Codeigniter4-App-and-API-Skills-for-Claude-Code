# Shield User Entity & User Provider — Complete Reference

## User Entity

Shield's `User` entity (`CodeIgniter\Shield\Entities\User`) provides all auth-related methods.

### Key Methods

```php
$user = auth()->user();

// ─── Identity & Credentials ─────────────────────────────────────
$user->getEmail();                    // email address
$user->getEmailIdentity();           // the email Identity entity
$user->username;                     // username

// ─── Password ────────────────────────────────────────────────────
$user->password = 'new-password';    // auto-hashed via entity setter
$user->forcePasswordReset();         // require change on next login

// ─── Status ──────────────────────────────────────────────────────
$user->isBanned();                   // bool
$user->ban('Reason');                // ban with optional message
$user->unBan();                      // remove ban
$user->getBanMessage();              // get the ban reason

$user->isActivated();                // bool — has completed activation
$user->activate();                   // manually activate
$user->deactivate();                 // deactivate

$user->isNotActivated();             // bool
$user->status();                     // status string
$user->statusMessage();              // human-readable status

// ─── Timestamps ──────────────────────────────────────────────────
$user->last_active;                  // last activity timestamp

// ─── Groups ──────────────────────────────────────────────────────
$user->addGroup('admin');
$user->removeGroup('admin');
$user->inGroup('admin');             // bool
$user->getGroups();                  // ['admin', 'developer']

// ─── Permissions ─────────────────────────────────────────────────
$user->can('posts.edit');            // bool (group matrix + direct)
$user->cannot('users.delete');       // bool
$user->addPermission('admin.access');
$user->removePermission('admin.access');
$user->getPermissions();             // direct permissions only
$user->hasPermission('admin.access'); // direct only, not group matrix

// ─── Access Tokens ───────────────────────────────────────────────
$token = $user->generateAccessToken('name');
$token = $user->generateAccessToken('name', ['scope1', 'scope2']);
$user->accessTokens();               // all tokens
$user->getAccessToken($tokenId);
$user->revokeAccessToken($tokenId);
$user->revokeAllAccessTokens();
$user->tokenCan('posts.create');     // current token scope check
$user->tokenCant('users.delete');

// ─── HMAC Tokens ─────────────────────────────────────────────────
$token = $user->generateHmacToken('name');
$user->hmacTokens();
$user->revokeHmacToken($tokenId);
$user->revokeAllHmacTokens();
```

### Extending the User Entity

```php
<?php
namespace App\Entities;

use CodeIgniter\Shield\Entities\User as ShieldUser;

class User extends ShieldUser
{
    // Custom properties/methods
    public function getDisplayName(): string
    {
        return $this->first_name
            ? "{$this->first_name} {$this->last_name}"
            : $this->username;
    }

    public function getAvatarUrl(): string
    {
        return $this->avatar_url ?? '/images/default-avatar.png';
    }

    public function isAdmin(): bool
    {
        return $this->inGroup('admin', 'superadmin');
    }
}
```

---

## User Provider (UserModel)

The User Provider is Shield's `UserModel`. Extend it to add custom fields or behavior.

### Extending the UserModel

```php
<?php
namespace App\Models;

use CodeIgniter\Shield\Models\UserModel as ShieldUserModel;

class UserModel extends ShieldUserModel
{
    protected function initialize(): void
    {
        parent::initialize();

        // Add custom allowed fields
        $this->allowedFields = [
            ...$this->allowedFields,
            'first_name',
            'last_name',
            'phone',
            'avatar_url',
            'bio',
        ];
    }
}
```

**IMPORTANT**: When you extend the UserModel, update `app/Config/Auth.php`:

```php
// app/Config/Auth.php
public array $userProvider = [
    'class' => \App\Models\UserModel::class,
];
```

Without this, Shield uses its own model and ignores your customizations.

### Getting the User Provider

```php
$users = auth()->getProvider();  // returns the configured UserModel
```

---

## Creating Users Programmatically

```php
use CodeIgniter\Shield\Entities\User;

$users = auth()->getProvider();

$user = new User([
    'username' => 'johndoe',
    'email'    => 'john@example.com',
    'password' => 'secretpassword',  // auto-hashed
]);

$users->save($user);

// Get the inserted user with ID
$user = $users->findById($users->getInsertID());

// Add to default group
$users->addToDefaultGroup($user);

// Or add to specific group
$user->addGroup('admin');
```

### Creating with Custom Fields

```php
$user = new User([
    'username'   => 'johndoe',
    'email'      => 'john@example.com',
    'password'   => 'secretpassword',
    'first_name' => 'John',
    'last_name'  => 'Doe',
    'phone'      => '555-0100',
]);

$users->save($user);
```

---

## Finding Users

```php
$users = auth()->getProvider();

// By ID
$user = $users->findById(1);

// By credentials (email or username)
$user = $users->findByCredentials(['email' => 'john@example.com']);
$user = $users->findByCredentials(['username' => 'johndoe']);

// By multiple IDs
$results = $users->findById([1, 2, 3]);

// All users
$all = $users->findAll();

// With query conditions
$admins = $users->where('active', 1)->findAll();
```

### Efficient Listing (Avoid N+1 Queries)

```php
// Preload identities, groups, and permissions
$allUsers = $users
    ->withIdentities()
    ->withGroups()
    ->withPermissions()
    ->findAll(20);
```

Without these preloads, each call to `$user->getEmail()`, `$user->getGroups()`, etc. triggers a separate DB query.

---

## Updating Users

```php
$users = auth()->getProvider();
$user  = $users->findById(1);

// Update standard fields
$user->fill([
    'username'   => 'new_username',
    'first_name' => 'John',
]);
$users->save($user);

// Update password (auto-hashed by entity setter)
$user->password = 'new_password';
$users->save($user);

// Update email (updates auth_identities, not users table)
$user->email = 'new@example.com';
$users->save($user);
```

**GOTCHA**: Password auto-hashed by the entity setter. Never manually hash before setting — you'll double-hash.

---

## Deleting Users

```php
$users = auth()->getProvider();

$users->delete($id);              // soft delete (if enabled)
$users->delete($id, true);        // permanent delete
```

---

## Banning Users

```php
$user = auth()->getProvider()->findById($id);

// Ban
$user->ban('Violation of terms of service');

// Check
if ($user->isBanned()) {
    $reason = $user->getBanMessage();
}

// Unban
$user->unBan();
```

Banned users cannot log in — Shield checks ban status during `attempt()`.

---

## Force Password Reset

```php
$user->forcePasswordReset();
```

On next login, the user is redirected to a password reset form. Use the `force-reset` filter on routes to enforce this redirect on already-logged-in users.

```php
$routes->group('', ['filter' => ['session', 'force-reset']], static function ($routes) {
    // ...
});
```

---

## Spark User Management

```bash
php spark shield:user create           # create user via CLI
php spark shield:user activate         # activate a user
php spark shield:user deactivate       # deactivate a user
php spark shield:user changename       # change username
php spark shield:user changepassword   # change password
```
