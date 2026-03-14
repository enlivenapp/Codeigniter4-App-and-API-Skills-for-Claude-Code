# Claude Code Skills: CodeIgniter 4

A pair of Claude Code skills for building with CodeIgniter 4. Designed to give Claude a solid, accurate understanding of the CI4 framework.

- **`/ci4`** - Core CI4 framework: MVC conventions, routing, controllers, models, query builder, views, migrations, filters, Spark CLI, and Shield auth.
- **`/ci4-api`** - Building robust REST APIs with CI4: response envelopes, versioning, token auth, validation, rate limiting, CORS, and webhooks.

---

## Installation

Clone this repo directly into your Claude Code skills directory:

```bash
git clone https://github.com/enlivenapp/Codeigniter4-App-and-API-Skills-for-Claude-Code.git ~/.claude/skills/codeigniter4
```

Or if you already have other skills and want to add just these:

```bash
cd ~/.claude/skills
git clone https://github.com/enlivenapp/Codeigniter4-App-and-API-Skills-for-Claude-Code.git
cp -r Codeigniter4-App-and-API-Skills-for-Claude-Code/ci4 .
cp -r Codeigniter4-App-and-API-Skills-for-Claude-Code/ci4-api .
```

Restart Claude Code. The skills are now available as `/ci4` and `/ci4-api`. In your Claude Code session, simply type `/ci4` or `/ci4-api` to invoke the relevant skill. Claude will load the full framework reference and apply it to your project automatically.

---

## Setting Up a CI4 Project from Scratch

Start here every time. This gets you to a known, working base before writing any application code.

### Requirements

- PHP 8.1+
- Composer
- MySQL (or another supported DB)
- A web server (Apache/Nginx) or use `php spark serve` for local dev

### 1. Create the Project

```bash
composer create-project codeigniter4/appstarter my-app
cd my-app
```

> Official docs: https://codeigniter.com/user_guide/installation/installing_composer.html

### 2. Configure the Environment

```bash
cp env .env
```

Open `.env` and set:

```ini
CI_ENVIRONMENT = development

# Must have trailing slash
app.baseURL = 'http://localhost:8080/'

# Database
database.default.hostname = localhost
database.default.database = your_database
database.default.username = your_user
database.default.password = your_password
database.default.DBDriver = MySQLi
```

### 3. Generate Encryption Key

```bash
php spark key:generate
```

Writes the key directly into your `.env`. Run this once per project.

### 4. Clean URLs (Optional but Recommended)

In `app/Config/App.php`, set:

```php
public string $indexPage = '';
```

Make sure your web server has URL rewriting enabled (Apache `.htaccess` is included by default in the appstarter).

### 5. Start the Dev Server

```bash
php spark serve
```

Visit `http://localhost:8080` and you should see the CI4 welcome page.

> **Never use `spark serve` in production.** It is single-threaded and not suitable for real traffic. Use Apache or Nginx.

---

## Adding Shield (Authentication)

Shield is the official CI4 authentication and authorization library.

> Official docs: https://shield.codeigniter.com/getting_started/install/

### 1. Install via Composer

```bash
composer require codeigniter4/shield
```

### 2. Run Shield Setup

```bash
php spark shield:setup
```

This command:
- Publishes `Auth.php`, `AuthGroups.php`, and `AuthToken.php` to `app/Config/`
- Wires up auth helpers in `app/Config/Autoload.php`
- Adds Shield routes to `app/Config/Routes.php`
- Prompts to run migrations. Say **yes**.

### 3. Run Migrations (if not run during setup)

```bash
php spark migrate --all
```

Shield creates these tables:
- `auth_identities` - emails, usernames, passwords
- `auth_tokens` - personal access tokens (for API auth)
- `auth_token_logins` - token login history
- `auth_groups_users` - user group assignments
- `auth_permissions_users` - per-user permission overrides

Verify with:
```bash
php spark migrate:status
```

### 4. Configure Groups

Edit `app/Config/AuthGroups.php` to define your groups. Shield ships with these defaults:

```php
public array $groups = [
    'superadmin' => ['title' => 'Super Admin', 'description' => 'Complete control of the site.'],
    'admin'      => ['title' => 'Admin',       'description' => 'Day to day administrators of the site.'],
    'developer'  => ['title' => 'Developer',   'description' => 'Site programmers.'],
    'user'       => ['title' => 'User',        'description' => 'General users of the site. Often customers.'],
    'beta'       => ['title' => 'Beta User',   'description' => 'Has access to beta-level features.'],
];
```

Add, remove, or rename groups to match your application's needs.

### 5. Create Your First Admin User

**Interactive (recommended for getting started):**

```bash
php spark shield:user create
php spark shield:user addgroup <username> superadmin
```

**Programmatic (seeder - good for repeatable dev environments):**

```bash
php spark make:seeder AdminSeeder
```

```php
<?php
namespace App\Database\Seeds;

use CodeIgniter\Database\Seeder;
use CodeIgniter\Shield\Entities\User;

class AdminSeeder extends Seeder
{
    public function run(): void
    {
        $users = auth()->getProvider();

        $user = new User([
            'username' => 'admin',
            'email'    => 'admin@example.com',
            'password' => 'change-me-immediately',
        ]);

        $users->save($user);
        $user = $users->findById($users->getInsertID());
        $users->addToGroup($user, 'superadmin');
    }
}
```

```bash
php spark db:seed AdminSeeder
```

---

## Full Setup Checklist

```bash
composer create-project codeigniter4/appstarter my-app
cd my-app
cp env .env
# Edit .env: CI_ENVIRONMENT, app.baseURL, database credentials
php spark key:generate
composer require codeigniter4/shield
php spark shield:setup          # say yes to migrations
php spark shield:user create
php spark shield:user addgroup <username> superadmin
php spark serve
```

You now have a working CI4 + Shield installation. Type `/ci4` or `/ci4-api` in your Claude Code session to get started.

---

## Resources

- [CodeIgniter 4 User Guide](https://codeigniter.com/user_guide/)
- [CodeIgniter 4 Installation](https://codeigniter.com/user_guide/installation/installing_composer.html)
- [CodeIgniter 4 on GitHub](https://github.com/codeigniter4/CodeIgniter4)
- [CodeIgniter Shield Docs](https://shield.codeigniter.com/)
- [CodeIgniter Shield on GitHub](https://github.com/codeigniter4/shield)
- [CodeIgniter Forum](https://forum.codeigniter.com/)
- [CodeIgniter on Slack](https://codeigniterchat.slack.com/)
