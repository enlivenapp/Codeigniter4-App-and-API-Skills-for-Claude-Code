# Runtime Namespaces -- Complete Reference

## Static vs Runtime Registration

**Static** (`Config/Autoload.php`) -- use when paths are known at deploy time:

```php
public $psr4 = [
    APP_NAMESPACE => APPPATH,
    'Acme\Blog'   => ROOTPATH . 'acme/Blog',
];
```

**Runtime** (`service('autoloader')->addNamespace()`) -- use when paths depend on DB state, user config, or filesystem discovery:

```php
service('autoloader')->addNamespace('Plugins\\Store', ROOTPATH . 'plugins/Store/');
```

After registration, all CI4 subsystems resolve classes in that namespace:

| Class | Resolved Path |
|---|---|
| `Plugins\Store\Models\ProductModel` | `plugins/Store/Models/ProductModel.php` |
| `Plugins\Store\Controllers\Catalog` | `plugins/Store/Controllers/Catalog.php` |
| `Plugins\Store\Database\Migrations\CreateProducts` | `plugins/Store/Database/Migrations/CreateProducts.php` |
| `Plugins\Store\Commands\SyncInventory` | `plugins/Store/Commands/SyncInventory.php` |

## Two-Phase Boot Pattern

Web requests and CLI commands need namespaces registered at different points:

### Web: pre_system Event

```php
// app/Config/Events.php
Events::on('pre_system', static function (): void {
    try {
        // Discover active addons from DB, register each namespace
        $addons = /* fetch from DB */;
        foreach ($addons as $addon) {
            service('autoloader')->addNamespace(
                $addon->namespace,
                ROOTPATH . 'plugins/' . $addon->folder . '/'
            );
        }
    } catch (\Throwable $e) {
        // DB may not exist yet (fresh install). Log and continue.
        log_message('debug', 'Namespace registration skipped: ' . $e->getMessage());
    }
});
```

### CLI: Services Override

Spark builds the Commands service before `pre_system` fires. Override `Services::commands()`:

```php
// app/Config/Services.php
public static function commands(bool $getShared = true): Commands
{
    if ($getShared) {
        return static::getSharedInstance('commands');
    }

    if (is_cli()) {
        // Only register namespaces for addons that have Commands/
        $addons = /* fetch from DB */;
        foreach ($addons as $addon) {
            $basePath = ROOTPATH . 'plugins/' . $addon->folder . '/';
            if (is_dir($basePath . 'Commands')) {
                service('autoloader')->addNamespace(
                    $addon->namespace,
                    $basePath
                );
            }
        }
    }

    return new Commands();
}
```

### Why Two Phases?

- Web requests need full namespace registration (controllers, models, views) before route matching
- CLI only needs namespaces with `Commands/` directories for Spark discovery
- Keeping CLI boot minimal avoids unnecessary DB queries and class loading

## Timing Constraints

```
Web request lifecycle:
  1. index.php / CodeIgniter bootstrap
  2. pre_system event fires          <-- register namespaces HERE
  3. Route matching                  <-- needs controller classes resolvable
  4. Filter execution
  5. Controller dispatch

CLI lifecycle:
  1. spark bootstrap
  2. Services::commands() called     <-- register namespaces HERE
  3. Command discovery (scans Commands/ in all namespaces)
  4. Command execution
```

**Critical**: if PHP tries to autoload a class before its namespace is registered, the failure is cached for the rest of the request. Later registration won't fix it.

## Multiple Namespaces for the Same Path

You can map multiple namespaces to directories under a single addon:

```php
$base = ROOTPATH . 'plugins/Store/';
service('autoloader')->addNamespace('Plugins\\Store', $base);
service('autoloader')->addNamespace('Plugins\\Store\\Api', $base . 'Api/');
```

Usually unnecessary -- a single top-level registration is enough since PSR-4 maps subdirectories automatically.

## service('autoloader') is Shared

All `addNamespace()` calls accumulate on the same autoloader instance. No need to pass it around or worry about separate instances.

## Gotchas

1. **Composer optimized autoloader** -- `composer dump-autoload -o` creates a classmap. Runtime `addNamespace()` still works because CI4's autoloader runs alongside Composer's, but Composer's optimized lookups won't find dynamically registered classes.

2. **Namespace collision** -- if two addons register overlapping namespaces, the last one wins. Use unique prefixes.

3. **Don't register inside constructors** -- config classes and services may be instantiated before you expect. Register in `pre_system` or `Services::commands()`.
