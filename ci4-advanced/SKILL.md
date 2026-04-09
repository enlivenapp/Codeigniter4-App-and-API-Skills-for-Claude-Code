---
name: ci4-advanced
description: Use when building plugin systems, addon architectures, dynamic namespace registration, runtime route injection, or any CI4 pattern that goes beyond standard app/ conventions. Activates on mentions of "plugin", "addon", "dynamic namespace", "runtime routes", "addNamespace", "Registrar", "module discovery", "external namespace", or extending CI4 with code outside app/.
version: 1.0.0
---

# CodeIgniter 4 -- Advanced Patterns Reference

How to make CI4's subsystems (autoloader, migrations, seeds, commands, routes, filters, config) work from code that lives outside the standard `app/` directory. Covers both documented-but-underused features and runtime techniques the docs don't explain.

> **Related skills**: `ci4` for framework fundamentals, `ci4-api` for REST APIs, `ci4-shield` for authentication.

## Reference Documents

For deep dives, read the relevant reference from `references/`:

| Reference | When to read |
|---|---|
| `references/runtime-namespaces.md` | addNamespace() at runtime, two-phase boot, timing constraints |
| `references/external-subsystems.md` | Running migrations, seeds, commands, View Cells, filters from outside app/ |
| `references/registrars-and-discovery.md` | Registrar pattern, Config/Modules.php auto-discovery |
| `references/runtime-config.md` | Mutating shared config instances, event bootstrap, dynamic routes |

---

## Runtime Namespace Registration

CI4 docs show static PSR-4 mapping in `Config/Autoload.php`. But `service('autoloader')->addNamespace()` lets you register namespaces at runtime -- essential when you don't know what code exists until the app boots.

```php
// Static (docs show this) -- known at deploy time
// app/Config/Autoload.php
public $psr4 = [
    APP_NAMESPACE => APPPATH,
    'Acme\Blog'   => ROOTPATH . 'acme/Blog',  // fixed path
];

// Runtime -- discovered at boot time
service('autoloader')->addNamespace('Plugins\\Store', ROOTPATH . 'plugins/Store/');
```

After runtime registration, all CI4 subsystems find classes in that namespace: controllers, models, migrations, commands, View Cells, filters.

See `references/runtime-namespaces.md` for two-phase boot (web vs CLI) and timing constraints.

---

## Running Subsystems from External Namespaces

Once a namespace is registered (statically or at runtime), CI4's subsystems can target it:

### Migrations

```php
// Run migrations for a specific namespace only
$migrate = \Config\Services::migrations();
$migrate->setNamespace('Plugins\\Store')->latest();
```

CI4 auto-scans `Database/Migrations/` in all registered namespaces. `setNamespace()` narrows it to one.

### Seeders (No setNamespace -- Must Glob)

CI4 has no `setNamespace()` for seeders. Glob the directory and call each class:

```php
$seedsPath = ROOTPATH . 'plugins/Store/Database/Seeds/';
$namespace = 'Plugins\\Store';
$seeder    = new \CodeIgniter\Database\Seeder(config('Database'));

foreach (glob($seedsPath . '*.php') as $file) {
    $class = $namespace . '\\Database\\Seeds\\' . pathinfo($file, PATHINFO_FILENAME);
    $seeder->call($class);
}
```

### Commands

Spark discovers commands from any registered namespace's `Commands/` directory. The catch: **the namespace must be registered before the `Commands` service is constructed**, and Spark builds that service early -- before `pre_system` fires. If you register namespaces in `pre_system` (which is correct for web requests), your commands will work on the web but `php spark list` won't show them and `php spark your:command` will fail with "command not found."

The fix is overriding `Services::commands()` to register namespaces in the CLI path specifically:

```php
// app/Config/Services.php
public static function commands(bool $getShared = true): Commands
{
    if ($getShared) {
        return static::getSharedInstance('commands');
    }

    if (is_cli()) {
        // Register external namespaces BEFORE Commands is built
        foreach ($addons as $addon) {
            $basePath = ROOTPATH . 'plugins/' . $addon->folder . '/';
            if (is_dir($basePath . 'Commands')) {
                service('autoloader')->addNamespace($addon->namespace, $basePath);
            }
        }
    }

    return new Commands();
}
```

This is the only way to get Spark to discover commands from runtime-registered namespaces. The docs don't mention it.

### View Cells

CI4 auto-finds View Cells in any namespace's `Cells/` subdirectory. You can omit the full namespace in `view_cell()`:

```php
// Full namespace always works
<?= view_cell('Plugins\\Store\\Cells\\FeaturedProducts::display') ?>

// Short form works if class is in a Cells/ subdirectory of a registered namespace
<?= view_cell('FeaturedProducts::display') ?>
```

### Filters

Register filter aliases from external namespaces via Registrar (see below) or manually in `Config/Filters.php`:

```php
public array $aliases = [
    'store_auth' => \Plugins\Store\Filters\StoreAuth::class,
];
```

See `references/external-subsystems.md` for complete patterns.

---

## Registrars -- Config Extension from External Code

CI4's Registrar pattern lets any namespace merge values into config classes at instantiation. Documented but widely underused.

Create `Config/Registrar.php` in any registered namespace:

```php
<?php
namespace Plugins\Store\Config;

class Registrar
{
    // Method name = config class name
    public static function Filters(): array
    {
        return [
            'aliases' => [
                'store_auth' => \Plugins\Store\Filters\StoreAuth::class,
            ],
        ];
    }

    public static function Pager(): array
    {
        return [
            'templates' => [
                'store_pager' => 'Plugins\Store\Views\pager',
            ],
        ];
    }
}
```

Values are merged when `config('Filters')` is first called. Registrar values overwrite defaults; `.env` values overwrite Registrars.

Requires auto-discovery enabled in `Config/Modules.php` with `'registrars'` in `$aliases`.

See `references/registrars-and-discovery.md` for what you can register and gotchas.

---

## Module Auto-Discovery

`Config/Modules.php` controls what CI4 discovers from external namespaces:

```php
public $aliases = [
    'events',      // Config/Events.php in any namespace
    'filters',     // Config/Filters.php
    'registrars',  // Config/Registrar.php
    'routes',      // Config/Routes.php
    'services',    // Config/Services.php extending BaseService
];
```

With discovery enabled, a module at `plugins/Store/` with a registered namespace can ship its own `Config/Services.php`, `Config/Routes.php`, etc. and they'll be found automatically.

**Service discovery from modules:**

```php
<?php
namespace Plugins\Store\Config;

use CodeIgniter\Config\BaseService;

class Services extends BaseService
{
    public static function cart(bool $getShared = true)
    {
        if ($getShared) {
            return static::getSharedInstance('cart');
        }
        return new \Plugins\Store\Services\CartService();
    }
}
```

Now `service('cart')` works app-wide.

---

## Service Overrides

Override framework services in `app/Config/Services.php` to inject behavior at construction time:

```php
<?php
namespace Config;

use CodeIgniter\CLI\Commands;
use CodeIgniter\Config\BaseService;

class Services extends BaseService
{
    public static function commands(bool $getShared = true): Commands
    {
        if ($getShared) {
            return static::getSharedInstance('commands');
        }

        // Register external namespaces BEFORE Commands service is built
        // so Spark discovers their Commands/ directories
        if (is_cli()) {
            // your namespace registration logic here
        }

        return new Commands();
    }
}
```

This is how you make `php spark` find commands from code outside `app/` without relying on auto-discovery.

---

## Runtime Config Mutation

`config()` returns shared instances. Modifying them affects the current request:

```php
// Add CSRF exemptions at runtime
$filters = config('Filters');
$filters->globals['before']['csrf']['except'][] = 'webhooks/*';

// Change supported locales before route matching
config('App')->supportedLocales = ['en', 'fr', 'de'];
```

This is powerful but invisible -- there's no trace in config files that these values changed. Use it when config must depend on runtime state (DB values, active plugins, etc). Prefer Registrars when the values are known at namespace registration time.

---

## Event-Driven Bootstrap

The `pre_system` event fires before route matching -- use it to set up state that routing and controllers depend on:

```php
// app/Config/Events.php
Events::on('pre_system', static function (): void {
    // Modify config before routing uses it
    $locales = /* fetch from DB or cache */;
    config('App')->supportedLocales = $locales;

    // IncomingRequest copied supportedLocales at construction --
    // sync it so {locale} route segments validate correctly
    service('request')->setValidLocales($locales);

    // Register namespaces before route matching
    // so external controllers are resolvable
    foreach ($activeAddons as $addon) {
        service('autoloader')->addNamespace($addon['namespace'], $addon['path']);
    }
});
```

Wrap in try/catch -- during fresh installs or migrations, DB tables may not exist yet.

---

## Dynamic Route Inclusion

Include route files from external paths inside `Routes.php`:

```php
// app/Config/Routes.php
$externalRouteFiles = [/* paths to route files */];

foreach ($externalRouteFiles as $routeFile) {
    if (is_file($routeFile)) {
        require $routeFile;
    }
}

// External route files have $routes in scope:
// plugins/Store/Config/Routes.php
$routes->group('store', ['namespace' => 'Plugins\Store\Controllers'], static function ($routes) {
    $routes->get('/', 'Catalog::index');
    $routes->post('cart/add', 'Cart::add');
});
```

If using `{locale}` groups, include external routes inside the group too so they work with locale prefixes.

**Alternative**: If auto-discovery is enabled and the namespace has `Config/Routes.php`, CI4 discovers it automatically. Manual inclusion gives you control over ordering and context (e.g., inside a locale group).

---

## Key Gotchas

1. **Namespace before class reference** -- `addNamespace()` must happen before any class in that namespace is referenced. PHP caches autoload failures per request.
2. **CLI vs web boot** -- web uses `pre_system` event; CLI needs namespaces registered before `Commands` service construction. These are different code paths.
3. **No setNamespace for seeders** -- migrations have it, seeders don't. Glob and call manually.
4. **Registrars need auto-discovery** -- `Config/Modules.php` must have `'registrars'` in `$aliases` and `$enabled = true`.
5. **Registrar methods are static** -- they return arrays, not modify objects. The framework merges the returned values.
6. **Don't instantiate Config classes in Registrars** -- causes duplicate instantiation loops. Return plain arrays only.
7. **config() shares instances** -- runtime mutations persist for the entire request. Good for dynamic config, but invisible to anyone reading config files.
8. **Route inclusion order** -- routes are matched first-come-first-served. External routes included after app routes won't shadow them.
9. **IncomingRequest caches supportedLocales** -- if you modify `config('App')->supportedLocales` in `pre_system`, also call `service('request')->setValidLocales()` to sync.
10. **exclude-from-classmap for migrations** -- add `"exclude-from-classmap": ["**/Database/Migrations/**"]` to `composer.json` so Composer's optimized autoloader doesn't cache stale migration classes from external namespaces.
