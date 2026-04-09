# External Subsystems -- Complete Reference

How to run CI4's built-in subsystems (migrations, seeds, commands, View Cells, filters, validation) from code outside `app/`.

Prerequisite: the external namespace must be registered (statically in `Config/Autoload.php` or at runtime via `addNamespace()`).

## Migrations

CI4's migration runner auto-scans `Database/Migrations/` in all registered namespaces.

**Run all namespaces:**
```php
$migrate = \Config\Services::migrations();
$migrate->setNamespace(null)->latest();  // null = all namespaces
```

**Run one namespace only:**
```php
$migrate->setNamespace('Plugins\\Store')->latest();
```

**Rollback one namespace:**
```php
$migrate->setNamespace('Plugins\\Store')->regress(-1);  // last batch
```

**Via Spark:**
```bash
php spark migrate -n Plugins\\Store
php spark migrate:rollback -n Plugins\\Store
```

Migration files follow standard CI4 naming -- just use the external namespace:

```php
<?php
namespace Plugins\Store\Database\Migrations;

use CodeIgniter\Database\Migration;

class CreateProductsTable extends Migration
{
    public function up(): void
    {
        $this->forge->addField([
            'id'    => ['type' => 'INT', 'unsigned' => true, 'auto_increment' => true],
            'name'  => ['type' => 'VARCHAR', 'constraint' => 255],
            'price' => ['type' => 'DECIMAL', 'constraint' => '10,2'],
        ]);
        $this->forge->addPrimaryKey('id');
        $this->forge->createTable('store_products');
    }

    public function down(): void
    {
        $this->forge->dropTable('store_products');
    }
}
```

## Seeders

CI4's Seeder class has **no `setNamespace()` method**. You must glob the directory and call each class:

```php
$namespace = 'Plugins\\Store';
$seedsPath = ROOTPATH . 'plugins/Store/Database/Seeds/';

if (is_dir($seedsPath)) {
    $seeder = new \CodeIgniter\Database\Seeder(config('Database'));

    foreach (glob($seedsPath . '*.php') as $file) {
        $class = $namespace . '\\Database\\Seeds\\' . pathinfo($file, PATHINFO_FILENAME);
        try {
            $seeder->call($class);
        } catch (\Throwable $e) {
            log_message('error', "Seeder failed: {$class} -- {$e->getMessage()}");
        }
    }
}
```

Seeder files use the external namespace:

```php
<?php
namespace Plugins\Store\Database\Seeds;

use CodeIgniter\Database\Seeder;

class DefaultSettings extends Seeder
{
    public function run(): void
    {
        $this->db->table('store_settings')->insertBatch([
            ['key' => 'currency', 'value' => 'USD'],
            ['key' => 'tax_rate', 'value' => '0.00'],
        ]);
    }
}
```

**Idempotency**: seeders run every time they're called. Use `INSERT IGNORE`, `upsert()`, or check-before-insert to avoid duplicates on re-activation.

## Commands

Spark discovers commands from `Commands/` in any registered namespace. No extra config needed.

```php
<?php
namespace Plugins\Store\Commands;

use CodeIgniter\CLI\BaseCommand;
use CodeIgniter\CLI\CLI;

class SyncInventory extends BaseCommand
{
    protected $group       = 'Store';
    protected $name        = 'store:sync';
    protected $description = 'Sync product inventory.';

    public function run(array $params): void
    {
        CLI::write('Syncing...', 'green');
    }
}
```

**Requirement**: the namespace must be registered before the Commands service is constructed. For CLI, this means registering in `Services::commands()` (see runtime-namespaces.md).

## View Cells

CI4 auto-finds View Cells in any registered namespace's `Cells/` subdirectory:

```php
<?php
namespace Plugins\Store\Cells;

class FeaturedProducts
{
    public function display(): string
    {
        $products = model('Plugins\Store\Models\ProductModel')->findAll(4);
        return view('Plugins\Store\Cells\featured_products', ['products' => $products]);
    }
}
```

Call with or without full namespace:

```php
// Full namespace (always works)
<?= view_cell('Plugins\\Store\\Cells\\FeaturedProducts::display') ?>

// Short form (works when in a Cells/ subdirectory)
<?= view_cell('FeaturedProducts::display') ?>
```

## Filters

Two approaches to register filters from external code:

**Via Registrar (preferred -- auto-discovered):**

```php
<?php
namespace Plugins\Store\Config;

class Registrar
{
    public static function Filters(): array
    {
        return [
            'aliases' => [
                'store_auth' => \Plugins\Store\Filters\StoreAuth::class,
            ],
        ];
    }
}
```

**Via manual registration in app/Config/Filters.php:**

```php
public array $aliases = [
    'store_auth' => \Plugins\Store\Filters\StoreAuth::class,
];
```

**Via runtime config mutation:**

```php
$filters = config('Filters');
$filters->aliases['store_auth'] = \Plugins\Store\Filters\StoreAuth::class;
```

Apply to routes as normal:

```php
$routes->group('store/admin', ['filter' => 'store_auth'], static function ($routes) {
    // ...
});
```

## Validation Rules

Custom validation rules from external namespaces work with callable syntax:

```php
$rules = [
    'license_key' => ['required', [\Plugins\Store\Validation\LicenseRules::class, 'validFormat']],
];
```

Or register a rule group via Registrar:

```php
<?php
namespace Plugins\Store\Config;

class Registrar
{
    public static function Validation(): array
    {
        return [
            'ruleSets' => [
                \Plugins\Store\Validation\StoreRules::class,
            ],
        ];
    }
}
```

## Language Files

CI4 auto-discovers language files in `Language/{locale}/` under any registered namespace:

```
plugins/Store/Language/en/Store.php
plugins/Store/Language/fr/Store.php
```

```php
// Works automatically once namespace is registered
echo lang('Store.productName');
```

## Helpers

Helpers from external namespaces are found automatically by `helper()`:

```
plugins/Store/Helpers/store_helper.php
```

```php
helper('store');  // Finds it in the registered namespace
```

Or auto-load in `Config/Autoload.php`:

```php
public $helpers = ['store'];
```
