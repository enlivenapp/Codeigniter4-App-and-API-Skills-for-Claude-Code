# Registrars and Auto-Discovery -- Complete Reference

## Registrars

A Registrar is a class that merges configuration values into CI4 config objects at instantiation time. Any registered namespace can ship one.

### How It Works

1. Code calls `config('Filters')` (or any config class)
2. CI4 instantiates `Config\Filters`
3. CI4 scans all registered namespaces for `Config\Registrar` classes
4. For each Registrar with a `Filters()` method, the returned array is merged
5. `.env` values are applied last (highest priority)

Priority: **defaults < Registrar < .env**

### Creating a Registrar

Place at `Config/Registrar.php` in any registered namespace:

```php
<?php
namespace Plugins\Store\Config;

class Registrar
{
    /**
     * Method name matches the Config class name.
     * Must be static. Must return an array.
     */
    public static function Filters(): array
    {
        return [
            'aliases' => [
                'store_auth'  => \Plugins\Store\Filters\StoreAuth::class,
                'store_cors'  => \Plugins\Store\Filters\StoreCors::class,
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

    public static function Validation(): array
    {
        return [
            'ruleSets' => [
                \Plugins\Store\Validation\StoreRules::class,
            ],
        ];
    }

    public static function ContentSecurityPolicy(): array
    {
        return [
            'scriptSrc' => ['self', 'https://js.stripe.com'],
        ];
    }
}
```

### What You Can Register

Any config class that extends `CodeIgniter\Config\BaseConfig` supports Registrars. Common targets:

| Config Class | What to merge |
|---|---|
| `Filters` | Filter aliases, globals, route-specific filters |
| `Pager` | Pagination templates |
| `Validation` | Rule sets, custom rule classes |
| `ContentSecurityPolicy` | CSP directives |
| `Toolbar` | Debug toolbar collectors |
| `Format` | Response format handlers |

### Registrar Gotchas

1. **Methods must be static and return arrays** -- no constructors, no side effects.

2. **Do NOT instantiate Config classes inside Registrars:**
   ```php
   // BAD -- causes infinite loop
   public static function Filters(): array
   {
       $app = config('App');  // This triggers Registrar scanning again
       return [];
   }
   ```

3. **Array merge behavior** -- Registrar values are merged into existing config. For nested arrays, Registrar values overwrite keys that exist. New keys are added.

4. **Multiple Registrars** -- if two namespaces both have `Registrar::Filters()`, both are merged. Last one wins for duplicate keys.

5. **Requires auto-discovery** -- Registrars only work if `Config/Modules.php` has `$enabled = true` and `'registrars'` in `$aliases`.

## Module Auto-Discovery

`Config/Modules.php` controls what CI4 discovers from external namespaces:

```php
<?php
namespace Config;

use CodeIgniter\Config\BaseConfig;

class Modules extends BaseConfig
{
    public $enabled = true;           // Master switch
    public $discoverInComposer = true; // Scan Composer packages too

    public $aliases = [
        'events',      // Config/Events.php
        'filters',     // Config/Filters.php
        'registrars',  // Config/Registrar.php
        'routes',      // Config/Routes.php
        'services',    // Config/Services.php
    ];
}
```

### What Each Alias Discovers

**events** -- `Config/Events.php` in external namespaces. Event listeners registered there are loaded automatically.

**filters** -- `Config/Filters.php` in external namespaces. Filter aliases and globals are merged. (Note: usually better to use a Registrar for this.)

**registrars** -- `Config/Registrar.php` as described above.

**routes** -- `Config/Routes.php` in external namespaces. Route definitions are loaded automatically. `$routes` is in scope.

**services** -- `Config/Services.php` extending `BaseService` in external namespaces. Service methods become available via `service()`.

### Service Discovery from Modules

An external namespace can define services that become globally available:

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

With discovery enabled, `service('cart')` works from anywhere in the app.

### Route Discovery from Modules

An external namespace's `Config/Routes.php` is auto-included:

```php
<?php
// plugins/Store/Config/Routes.php
// $routes is in scope via discovery

$routes->group('store', ['namespace' => 'Plugins\Store\Controllers'], static function ($routes) {
    $routes->get('/', 'Catalog::index');
});
```

**Trade-off**: auto-discovery is convenient but you lose control over route ordering and context (e.g., including routes inside a `{locale}` group). Manual `require` in `app/Config/Routes.php` gives full control.

### Composer Package Discovery

With `$discoverInComposer = true`, CI4 scans Composer packages too. This is how packages like `codeigniter4/shield` auto-register their filters, services, and routes.

### Discovery Performance

Auto-discovery scans namespaces on every request (results are not cached by default). For production, ensure only necessary aliases are enabled. Remove aliases you don't use.

## Registrars vs Runtime Config Mutation

| | Registrars | Runtime Mutation |
|---|---|---|
| **When** | Config instantiation | Any time after boot |
| **How** | Static method returns array | Modify `config()` shared instance |
| **Visible** | In the Registrar file | Invisible -- no config file trace |
| **Use when** | Values known at namespace registration | Values depend on runtime state (DB, request) |
| **Example** | Filter aliases, pager templates | CSRF exemptions from DB-driven addons |
