# Runtime Config Mutation -- Complete Reference

## Shared Config Instances

`config()` returns a shared instance. Modifications affect the entire request:

```php
$appConfig = config('App');
$appConfig->baseURL = 'https://example.com/';
// Every subsequent config('App') call returns this modified instance
```

This is how you make config depend on runtime state -- DB values, active addons, request properties.

## Common Runtime Config Patterns

### Dynamic CSRF Exemptions

Add route patterns that should skip CSRF at boot time:

```php
$filters = config('Filters');

// CSRF might be configured as string, array, or array with 'except'
if (isset($filters->globals['before']['csrf']['except'])) {
    $filters->globals['before']['csrf']['except'] = array_merge(
        $filters->globals['before']['csrf']['except'],
        ['webhooks/*', 'api/*']
    );
} elseif (isset($filters->globals['before']['csrf'])) {
    $filters->globals['before']['csrf'] = ['except' => ['webhooks/*', 'api/*']];
}
```

### Dynamic Locale Support

Populate supported locales from DB before route matching:

```php
Events::on('pre_system', static function (): void {
    try {
        $cache = \Config\Services::cache();
        $codes = $cache->get('active_languages');

        if ($codes === null) {
            $codes = /* fetch from DB */;
            $cache->save('active_languages', $codes, 3600);
        }

        if (!empty($codes)) {
            config('App')->supportedLocales = $codes;

            // IncomingRequest copied supportedLocales at construction.
            // Sync it so {locale} route segments validate correctly.
            service('request')->setValidLocales($codes);
        }
    } catch (\Throwable $e) {
        // DB not available (fresh install, CLI). Use defaults.
        log_message('debug', 'Locale bootstrap skipped: ' . $e->getMessage());
    }
});
```

### Dynamic Filter Aliases

Register filter aliases based on what addons are active:

```php
$filters = config('Filters');
$filters->aliases['addon_auth'] = \Plugins\Store\Filters\StoreAuth::class;
```

## Event-Driven Bootstrap

The `pre_system` event fires before routing. It's the right place for runtime config changes that affect how the request is processed.

```php
// app/Config/Events.php
Events::on('pre_system', static function (): void {

    // 1. Config modifications (locale, filters, etc.)
    // 2. Namespace registration for external code
    // 3. Any setup that must happen before routing

});
```

### Available Events for Bootstrap

| Event | When | Use for |
|---|---|---|
| `pre_system` | Before routing | Config changes, namespace registration |
| `post_controller_constructor` | After controller created | Controller-dependent setup |
| `DBQuery` | After each query | Query logging, debugging |

`pre_system` is the most useful for advanced patterns because it runs before CI4 makes routing and filter decisions.

### Error Handling in pre_system

Always wrap DB-dependent code in try/catch:

```php
Events::on('pre_system', static function (): void {
    try {
        // DB-dependent logic
    } catch (\Throwable $e) {
        log_message('debug', 'Bootstrap skipped: ' . $e->getMessage());
    }
});
```

During fresh installs, CLI migrations, or test environments, the database may not be available. Failing silently with a log message keeps the app bootable.

## Dynamic Route Inclusion

Include route files from external paths inside `app/Config/Routes.php`:

```php
// app/Config/Routes.php

// App routes first
$routes->get('/', 'Home::index');

// External routes
$externalRouteFiles = /* collect paths from active addons */;
foreach ($externalRouteFiles as $file) {
    if (is_file($file)) {
        require $file;
    }
}

// Locale group with external routes too
$routes->group('{locale}', static function ($routes) use ($externalRouteFiles): void {
    $routes->get('/', 'Home::index');

    foreach ($externalRouteFiles as $file) {
        if (is_file($file)) {
            require $file;
        }
    }
});
```

External route files write to the `$routes` variable already in scope:

```php
<?php
// plugins/Store/Config/Routes.php
$routes->group('store', ['namespace' => 'Plugins\Store\Controllers'], static function ($routes) {
    $routes->get('/', 'Catalog::index');
    $routes->get('product/(:segment)', 'Catalog::product/$1');
});
```

### Manual vs Auto-Discovered Routes

**Auto-discovery** (`Config/Modules.php` with `'routes'` alias): CI4 finds and includes `Config/Routes.php` from all registered namespaces. Simple but no control over inclusion order or context.

**Manual inclusion**: you choose where in `Routes.php` to include them. Required when:
- External routes need to be inside a `{locale}` group
- Route ordering matters (app routes should match before addon routes)
- You need conditional inclusion (only include if addon is active)

## Activation Chain Pattern

When activating an addon at runtime (not just booting known addons), run subsystems in order:

```php
function activateAddon(string $namespace, string $path): string
{
    // 1. Register namespace (must be first)
    service('autoloader')->addNamespace($namespace, $path);

    // 2. Run migrations
    try {
        \Config\Services::migrations()->setNamespace($namespace)->latest();
    } catch (\Throwable $e) {
        return 'migration_failed';
    }

    // 3. Run seeders (glob -- no setNamespace)
    $seedsPath = $path . 'Database/Seeds/';
    if (is_dir($seedsPath)) {
        $seeder = new \CodeIgniter\Database\Seeder(config('Database'));
        foreach (glob($seedsPath . '*.php') as $file) {
            $class = $namespace . '\\Database\\Seeds\\' . pathinfo($file, PATHINFO_FILENAME);
            try {
                $seeder->call($class);
            } catch (\Throwable $e) {
                return 'seed_failed';
            }
        }
    }

    // 4. Run installer if present (filesystem setup, etc.)
    $installerClass = $namespace . '\\Installer';
    if (class_exists($installerClass)) {
        $installer = new $installerClass();
        try {
            $installer->up();
        } catch (\Throwable $e) {
            try { $installer->down(); } catch (\Throwable $r) {}
            return 'install_failed';
        }
    }

    return 'activated';
}
```

The Installer pattern (up/down with rollback) is not a CI4 feature -- it's a convention for addon lifecycle management. Keep `up()` idempotent and `down()` safe to call on partial state.

## Gotchas

1. **Config mutation is request-scoped** -- changes don't persist between requests. They must be reapplied on every boot.

2. **Order of operations in pre_system** -- register namespaces before modifying config objects that depend on classes from those namespaces.

3. **setValidLocales() is easy to forget** -- `IncomingRequest` copies `supportedLocales` at construction (before `pre_system`). If you modify locales dynamically, you must sync the request object too.

4. **Route files are `require`d, not `require_once`d** -- if you include external routes in both the top-level and a `{locale}` group, the route file executes twice. This is intentional (routes are additive), but be aware.
