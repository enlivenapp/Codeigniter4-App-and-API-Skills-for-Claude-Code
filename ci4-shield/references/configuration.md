# Shield Configuration — Complete Reference

## Auth.php

```php
<?php
// app/Config/Auth.php

namespace Config;

use CodeIgniter\Shield\Config\Auth as ShieldAuth;

class Auth extends ShieldAuth
{
    // ─── Redirects ───────────────────────────────────────────────────
    public array $redirects = [
        'register' => '/',
        'login'    => '/',
        'logout'   => 'login',
    ];

    // ─── Session Config ──────────────────────────────────────────────
    public array $sessionConfig = [
        'field'              => 'user',
        'allowRemembering'   => true,
        'rememberCookieName' => 'remember',
        'rememberLength'     => 30 * DAY,
    ];

    // ─── Actions (Email Activation / 2FA) ────────────────────────────
    // null = disabled, class = enabled
    public array $actions = [
        'register' => null,
        // \CodeIgniter\Shield\Authentication\Actions\EmailActivator::class,
        'login'    => null,
        // \CodeIgniter\Shield\Authentication\Actions\Email2FA::class,
    ];

    // ─── Authenticators ──────────────────────────────────────────────
    public array $authenticators = [
        'session' => \CodeIgniter\Shield\Authentication\Authenticators\Session::class,
        'tokens'  => \CodeIgniter\Shield\Authentication\Authenticators\AccessTokens::class,
        'hmac'    => \CodeIgniter\Shield\Authentication\Authenticators\HmacSha256::class,
        // 'jwt'  => \CodeIgniter\Shield\Authentication\Authenticators\JWT::class,
    ];

    public string $defaultAuthenticator = 'session';

    // ─── Login Fields ────────────────────────────────────────────────
    // Which fields can be used to log in
    public array $validFields = ['email'];
    // For username + email login:
    // public array $validFields = ['email', 'username'];

    // ─── Password Validators ─────────────────────────────────────────
    public array $passwordValidators = [
        \CodeIgniter\Shield\Authentication\Passwords\CompositionValidator::class,
        \CodeIgniter\Shield\Authentication\Passwords\NothingPersonalValidator::class,
        \CodeIgniter\Shield\Authentication\Passwords\DictionaryValidator::class,
        // \CodeIgniter\Shield\Authentication\Passwords\PwnedValidator::class,
    ];

    public int $minimumPasswordLength = 8;

    // ─── User Provider ───────────────────────────────────────────────
    // Point to your custom UserModel if you extended it
    public array $userProvider = [
        'class' => \App\Models\UserModel::class,
    ];

    // ─── Views ───────────────────────────────────────────────────────
    public array $views = [
        'login'                       => '\CodeIgniter\Shield\Views\login',
        'register'                    => '\CodeIgniter\Shield\Views\register',
        'layout'                      => '\CodeIgniter\Shield\Views\layout',
        'action_email_2fa'            => '\CodeIgniter\Shield\Views\email_2fa_show',
        'action_email_2fa_verify'     => '\CodeIgniter\Shield\Views\email_2fa_verify',
        'action_email_2fa_email'      => '\CodeIgniter\Shield\Views\Email\email_2fa_email',
        'action_email_activate_show'  => '\CodeIgniter\Shield\Views\email_activate_show',
        'action_email_activate_email' => '\CodeIgniter\Shield\Views\Email\email_activate_email',
        'magic-link-login'            => '\CodeIgniter\Shield\Views\magic_link_form',
        'magic-link-message'          => '\CodeIgniter\Shield\Views\magic_link_message',
        'magic-link-email'            => '\CodeIgniter\Shield\Views\Email\magic_link_email',
    ];

    // ─── Email Validation Rules ──────────────────────────────────────
    public array $emailValidationRules = [
        'label' => 'Auth.email',
        'rules' => 'required|max_length[254]|valid_email',
    ];

    // ─── Username Validation Rules ───────────────────────────────────
    public array $usernameValidationRules = [
        'label' => 'Auth.username',
        'rules' => 'required|max_length[30]|min_length[3]|regex_match[/\A[a-zA-Z0-9\.]+\z/]',
    ];
}
```

## AuthGroups.php

```php
<?php
// app/Config/AuthGroups.php

namespace Config;

use CodeIgniter\Shield\Config\AuthGroups as ShieldAuthGroups;

class AuthGroups extends ShieldAuthGroups
{
    // ─── Groups ──────────────────────────────────────────────────────
    public array $groups = [
        'superadmin' => [
            'title'       => 'Super Admin',
            'description' => 'Complete control of the site.',
        ],
        'admin' => [
            'title'       => 'Admin',
            'description' => 'Day to day administrators.',
        ],
        'developer' => [
            'title'       => 'Developer',
            'description' => 'Site developers.',
        ],
        'user' => [
            'title'       => 'User',
            'description' => 'General users.',
        ],
    ];

    // New users get this group automatically
    public string $defaultGroup = 'user';

    // ─── Permissions ─────────────────────────────────────────────────
    public array $permissions = [
        'admin.access'  => 'Can access the admin area',
        'admin.settings' => 'Can access site settings',
        'users.create'  => 'Can create new users',
        'users.edit'    => 'Can edit existing users',
        'users.delete'  => 'Can delete users',
        'posts.create'  => 'Can create posts',
        'posts.edit'    => 'Can edit posts',
        'posts.delete'  => 'Can delete posts',
    ];

    // ─── Permission Matrix ───────────────────────────────────────────
    // Wildcard '*' grants all permissions in a namespace
    public array $matrix = [
        'superadmin' => ['admin.*', 'users.*', 'posts.*'],
        'admin'      => ['admin.access', 'users.create', 'users.edit', 'posts.*'],
        'developer'  => ['admin.access', 'posts.*'],
        'user'       => ['posts.create'],
    ];
}
```

## Password Validators

| Validator | What it checks |
|---|---|
| `CompositionValidator` | Minimum length (default 8), can require mixed case, numbers, special chars |
| `NothingPersonalValidator` | Rejects passwords containing user's email, username, or other personal info |
| `DictionaryValidator` | Rejects common passwords from a bundled dictionary |
| `PwnedValidator` | Checks against the Have I Been Pwned API (disabled by default — external HTTP call) |

## JWT Configuration

If using JWT auth, uncomment `'jwt' => JWT::class` in `$authenticators` and configure:

```php
// app/Config/AuthToken.php
public array $jwtConfig = [
    'algorithm'    => 'HS256',      // HS256, HS384, HS512, RS256, RS384, RS512
    'timeToLive'   => HOUR,         // Token expiration
    'key'          => '',           // Secret key for HMAC algorithms
    'privateKey'   => '',           // Private key path for RSA algorithms
    'publicKey'    => '',           // Public key path for RSA algorithms
];
```

## Required .env Config

Email config is required for email activation, 2FA, and magic links:

```env
email.fromEmail = noreply@example.com
email.fromName = My App
email.protocol = smtp
email.SMTPHost = smtp.example.com
email.SMTPUser = user
email.SMTPPass = pass
email.SMTPPort = 587
email.SMTPCrypto = tls
```
