# Shield Actions — Email Activation, 2FA, Magic Links & Password Handling

## Email Activation (Registration)

Require new users to verify their email before they can log in.

### Enable

```php
// app/Config/Auth.php
public array $actions = [
    'register' => \CodeIgniter\Shield\Authentication\Actions\EmailActivator::class,
    'login'    => null,
];
```

### Flow

1. User registers via the registration form
2. Shield sends an activation email with a 6-digit code
3. User is redirected to the activation page to enter the code
4. Once verified, the user can log in normally

### Manual Activation

```php
// Skip email verification — activate directly
$user->activate();

// Deactivate
$user->deactivate();

// Check status
$user->isActivated();      // bool
$user->isNotActivated();   // bool
```

### Requirements

- `app/Config/Email.php` must be properly configured
- Email sending must work (SMTP, sendmail, etc.)

---

## Two-Factor Authentication (Email 2FA)

Adds a second verification step after password login — Shield sends a code via email.

### Enable

```php
// app/Config/Auth.php
public array $actions = [
    'register' => null,
    'login'    => \CodeIgniter\Shield\Authentication\Actions\Email2FA::class,
];
```

### Flow

1. User enters email + password (standard login)
2. If credentials are valid, Shield sends a 6-digit code via email
3. User is redirected to a code entry page
4. User enters the code
5. If valid, login completes and session is established

### Both Actions Enabled

```php
// Email activation on register + 2FA on login
public array $actions = [
    'register' => \CodeIgniter\Shield\Authentication\Actions\EmailActivator::class,
    'login'    => \CodeIgniter\Shield\Authentication\Actions\Email2FA::class,
];
```

### Custom 2FA Views

Override in `Auth.php`'s `$views` array:

```php
public array $views = [
    'action_email_2fa'        => 'auth/2fa_show',        // code entry form
    'action_email_2fa_verify' => 'auth/2fa_verify',      // verification result
    'action_email_2fa_email'  => 'auth/emails/2fa_code', // email template
];
```

---

## Magic Link Login

Passwordless login — the user enters their email, receives a link, clicks it, and is logged in.

### Flow

1. User visits `/login/magic-link`
2. Enters their email address
3. Shield sends an email with a one-time login link
4. User clicks the link
5. Shield authenticates the user and redirects to the login redirect URL

### Custom Views

```php
public array $views = [
    'magic-link-login'   => 'auth/magic_link_form',    // email entry form
    'magic-link-message' => 'auth/magic_link_sent',    // "check your email" page
    'magic-link-email'   => 'auth/emails/magic_link',  // email template
];
```

### Default Routes

| Route | Purpose |
|---|---|
| `GET /login/magic-link` | Show email entry form |
| `POST /login/magic-link` | Send magic link email |
| `GET /login/verify-magic-link` | Verify link and log in |

---

## Password Handling

### Password Validators

Configured in `Auth.php`. Shield validates passwords through a validator chain:

| Validator | What it checks |
|---|---|
| `CompositionValidator` | Min length (default 8), can require mixed case, numbers, special chars |
| `NothingPersonalValidator` | Rejects passwords containing user's email, username, or personal info |
| `DictionaryValidator` | Rejects common passwords from a bundled dictionary |
| `PwnedValidator` | Checks Have I Been Pwned API (disabled by default — external HTTP call) |

```php
public array $passwordValidators = [
    \CodeIgniter\Shield\Authentication\Passwords\CompositionValidator::class,
    \CodeIgniter\Shield\Authentication\Passwords\NothingPersonalValidator::class,
    \CodeIgniter\Shield\Authentication\Passwords\DictionaryValidator::class,
    // \CodeIgniter\Shield\Authentication\Passwords\PwnedValidator::class,
];

public int $minimumPasswordLength = 8;
```

### Password Hashing

Passwords are **automatically hashed** when set via the User entity:

```php
$user->password = 'plain-text';  // auto-hashed via entity setter
// Stored as Argon2id (preferred) or bcrypt hash
```

**GOTCHA**: Never manually hash passwords before setting them on the entity — you'll double-hash, making the password unverifiable.

### Force Password Reset

```php
// Set the flag
$user->forcePasswordReset();

// The next time the user accesses a route protected by the 'force-reset' filter,
// they'll be redirected to change their password.

// Route configuration
$routes->group('', ['filter' => ['session', 'force-reset']], static function ($routes) {
    // all protected routes
});
```

### Password Validation Outside of Shield

```php
$passwords = service('passwords');

// Check password strength
$result = $passwords->check($password, $user);
if (!$result->isOK()) {
    $errors = $result->reason();
}

// Hash a password manually (rare — entity does this automatically)
$hash = $passwords->hash($password);

// Verify a password against a hash
$passwords->verify($password, $hash);  // bool
```

---

## Banning Users

```php
$user = auth()->getProvider()->findById($id);

// Ban with a reason
$user->ban('Violation of terms of service');

// Check ban status
if ($user->isBanned()) {
    $reason = $user->getBanMessage();
    // "Violation of terms of service"
}

// Remove ban
$user->unBan();
```

### Ban Behavior

- Banned users **cannot log in** — Shield checks ban status during `attempt()` and returns a failure result
- Already-logged-in banned users are NOT automatically logged out — you need to check `isBanned()` in a filter if you want real-time enforcement
- The ban message is stored and can be displayed to the user

### Real-Time Ban Enforcement (Custom Filter)

```php
class BanCheckFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $user = auth()->user();
        if ($user && $user->isBanned()) {
            auth()->logout();
            return redirect()->to('/login')->with('error', 'Your account has been suspended.');
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```
