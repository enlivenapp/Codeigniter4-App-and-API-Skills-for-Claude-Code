# CI4 Spark CLI — Complete Reference

## Development

```bash
php spark serve                          # dev server on port 8080
php spark serve --port 3000              # custom port
php spark serve --host 0.0.0.0           # listen on all interfaces
```

## Routes

```bash
php spark routes                         # list all registered routes
```

## Migrations

```bash
php spark migrate                        # run pending migrations
php spark migrate:rollback               # roll back last batch
php spark migrate:rollback -b 2         # roll back 2 batches
php spark migrate:status                # show migration status
php spark migrate:refresh               # rollback all + re-migrate (destructive!)
php spark migrate:reset                 # rollback all (destructive!)
```

## Seeds

```bash
php spark db:seed DatabaseSeeder         # run a seeder
php spark db:seed UserSeeder             # run specific seeder
```

## Generators (Scaffolding)

```bash
# Controllers
php spark make:controller UserController
php spark make:controller Api/UserController --restful   # ResourceController
php spark make:controller Admin/UsersController --suffix  # keeps "Controller" suffix

# Models
php spark make:model UserModel
php spark make:model UserModel --entity                  # also creates Entity class

# Migrations
php spark make:migration CreateUsersTable
php spark make:migration AddPhoneToUsers

# Filters
php spark make:filter AuthFilter

# Seeds
php spark make:seeder UserSeeder

# Entities
php spark make:entity UserEntity

# Libraries
php spark make:library PaymentGateway

# Commands (custom spark commands)
php spark make:command ClearExpiredTokens

# Validation rules
php spark make:validation CustomRules

# Config
php spark make:config MyConfig

# Cells (view cells)
php spark make:cell RecentPosts
```

### Generator Options

```bash
--namespace App        # target namespace (default: App)
--suffix              # append type suffix to class name
--force               # overwrite existing file
```

## Cache

```bash
php spark cache:clear                    # clear all cache
php spark cache:info                     # cache driver info
```

## Sessions

```bash
php spark session:migration              # generate session DB table migration
```

## Encryption

```bash
php spark key:generate                   # generate encryption key for .env
```

## Database

```bash
php spark db:table                       # list all tables
php spark db:table users                 # show table structure
php spark db:create my_database          # create database
```

## Shield (Auth)

```bash
php spark shield:setup                   # publish config + migrations
php spark shield:user create             # create user via CLI
php spark shield:user activate           # activate a user
php spark shield:user deactivate         # deactivate a user
php spark shield:user changename         # change username
php spark shield:user changepassword     # change password
```

## Maintenance

```bash
php spark env                            # show current environment
php spark phpini:check                   # check php.ini settings
```

## Namespaces

```bash
php spark namespaces                     # list all registered namespaces
```

## Custom Commands

Create a custom spark command:

```php
<?php
// app/Commands/ClearExpiredTokens.php

namespace App\Commands;

use CodeIgniter\CLI\BaseCommand;
use CodeIgniter\CLI\CLI;

class ClearExpiredTokens extends BaseCommand
{
    protected $group       = 'App';
    protected $name        = 'tokens:clear';
    protected $description = 'Clear expired access tokens';
    protected $usage       = 'tokens:clear [days]';
    protected $arguments   = [
        'days' => 'Number of days to keep (default: 30)',
    ];
    protected $options     = [
        '--dry-run' => 'Show what would be deleted without deleting',
    ];

    public function run(array $params)
    {
        $days   = (int) ($params[0] ?? 30);
        $dryRun = CLI::getOption('dry-run');

        $cutoff = date('Y-m-d H:i:s', strtotime("-{$days} days"));

        $db = \Config\Database::connect();
        $count = $db->table('auth_identities')
            ->where('type', 'access_token')
            ->where('last_used_at <', $cutoff)
            ->countAllResults(false);

        if ($dryRun) {
            CLI::write("Would delete {$count} expired tokens.", 'yellow');
            return;
        }

        $db->table('auth_identities')
            ->where('type', 'access_token')
            ->where('last_used_at <', $cutoff)
            ->delete();

        CLI::write("Deleted {$count} expired tokens.", 'green');
    }
}
```

Run: `php spark tokens:clear 60 --dry-run`

### CLI Helper Methods

```php
// Output
CLI::write('Normal text');
CLI::write('Green text', 'green');
CLI::write('Red bold', 'red', 'bold');
CLI::error('Error message');                // stderr, red
CLI::newLine();

// Input
$name = CLI::prompt('What is your name?');
$role = CLI::prompt('Role?', ['admin', 'user']);  // with choices
$confirm = CLI::prompt('Are you sure?', ['y', 'n']);

// Progress
CLI::showProgress(false);                  // start
for ($i = 1; $i <= 100; $i++) {
    CLI::showProgress($i, 100);           // update
}
CLI::showProgress(false);                  // end

// Table
CLI::table([
    ['Name', 'Email', 'Role'],
    ['Rob', 'rob@example.com', 'admin'],
    ['Alice', 'alice@example.com', 'user'],
]);
```
