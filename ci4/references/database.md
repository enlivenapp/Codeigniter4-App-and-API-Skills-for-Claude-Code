# CI4 Database — Migrations, Seeds & Forge Reference

## Migrations

### Create a Migration

```bash
php spark make:migration CreateUsersTable
php spark make:migration AddPhoneToUsers
```

Creates timestamped file in `app/Database/Migrations/`.

### Migration File Structure

```php
<?php
namespace App\Database\Migrations;

use CodeIgniter\Database\Migration;

class CreateUsersTable extends Migration
{
    public function up(): void
    {
        $this->forge->addField([
            'id'         => ['type' => 'INT', 'constraint' => 11, 'unsigned' => true, 'auto_increment' => true],
            'name'       => ['type' => 'VARCHAR', 'constraint' => 150],
            'email'      => ['type' => 'VARCHAR', 'constraint' => 255],
            'active'     => ['type' => 'TINYINT', 'constraint' => 1, 'default' => 1],
            'created_at' => ['type' => 'DATETIME', 'null' => true],
            'updated_at' => ['type' => 'DATETIME', 'null' => true],
            'deleted_at' => ['type' => 'DATETIME', 'null' => true],
        ]);

        $this->forge->addPrimaryKey('id');
        $this->forge->addKey('email', true);  // true = unique index
        $this->forge->createTable('users');
    }

    public function down(): void
    {
        $this->forge->dropTable('users');
    }
}
```

### Adding Columns

```php
public function up(): void
{
    $fields = [
        'phone' => ['type' => 'VARCHAR', 'constraint' => 20, 'null' => true, 'after' => 'email'],
    ];
    $this->forge->addColumn('users', $fields);
}

public function down(): void
{
    $this->forge->dropColumn('users', 'phone');
}
```

### Modifying Columns

```php
public function up(): void
{
    $fields = [
        'name' => [
            'name'       => 'name',           // same name = modify, not rename
            'type'       => 'VARCHAR',
            'constraint' => 255,              // was 150
        ],
    ];
    $this->forge->modifyColumn('users', $fields);
}
```

### Renaming Columns

```php
public function up(): void
{
    $fields = [
        'name' => [
            'name'       => 'full_name',      // different name = rename
            'type'       => 'VARCHAR',
            'constraint' => 150,
        ],
    ];
    $this->forge->modifyColumn('users', $fields);
}
```

### Adding Indexes

```php
// In createTable context
$this->forge->addPrimaryKey('id');
$this->forge->addKey('email', true);                  // unique
$this->forge->addKey('created_at');                    // regular index
$this->forge->addKey(['role', 'active']);              // composite index
$this->forge->addUniqueKey('slug');                    // unique index
$this->forge->addForeignKey('user_id', 'users', 'id', 'CASCADE', 'CASCADE');
// CASCADE = on update / on delete

// After table exists
$this->forge->addKey('phone');
$this->forge->processIndexes('users');
```

### Dropping

```php
$this->forge->dropTable('users');                     // drop table
$this->forge->dropTable('users', true);               // IF EXISTS
$this->forge->dropColumn('users', 'phone');           // drop column
$this->forge->dropColumn('users', ['phone', 'fax']);  // drop multiple
$this->forge->dropKey('users', 'email');              // drop index
$this->forge->dropForeignKey('orders', 'orders_user_id_foreign'); // drop FK
```

### Common Field Types

```php
'id'         => ['type' => 'INT', 'constraint' => 11, 'unsigned' => true, 'auto_increment' => true]
'name'       => ['type' => 'VARCHAR', 'constraint' => 150]
'slug'       => ['type' => 'VARCHAR', 'constraint' => 255, 'unique' => true]
'body'       => ['type' => 'TEXT']
'long_text'  => ['type' => 'LONGTEXT']
'price'      => ['type' => 'DECIMAL', 'constraint' => '10,2']
'active'     => ['type' => 'TINYINT', 'constraint' => 1, 'default' => 1]
'count'      => ['type' => 'INT', 'constraint' => 11, 'default' => 0, 'unsigned' => true]
'status'     => ['type' => 'ENUM', 'constraint' => ['pending', 'active', 'closed'], 'default' => 'pending']
'metadata'   => ['type' => 'JSON', 'null' => true]
'created_at' => ['type' => 'DATETIME', 'null' => true]
'sort_order' => ['type' => 'INT', 'constraint' => 11, 'default' => 0]

// Nullable field
'phone'      => ['type' => 'VARCHAR', 'constraint' => 20, 'null' => true]

// Field positioning
'phone'      => ['type' => 'VARCHAR', 'constraint' => 20, 'after' => 'email']
'priority'   => ['type' => 'INT', 'constraint' => 3, 'first' => true]
```

### Migration Spark Commands

```bash
php spark migrate                      # run pending migrations
php spark migrate:rollback             # roll back last batch
php spark migrate:rollback -b 2       # roll back 2 batches
php spark migrate:status              # show migration status
php spark migrate:refresh             # rollback all + re-migrate
php spark migrate:reset               # rollback all
```

---

## Seeds

```php
<?php
namespace App\Database\Seeds;

use CodeIgniter\Database\Seeder;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        $data = [
            [
                'name'       => 'Admin',
                'email'      => 'admin@example.com',
                'role'       => 'admin',
                'created_at' => date('Y-m-d H:i:s'),
            ],
            [
                'name'       => 'User',
                'email'      => 'user@example.com',
                'role'       => 'user',
                'created_at' => date('Y-m-d H:i:s'),
            ],
        ];
        $this->db->table('users')->insertBatch($data);
    }
}
```

### Master Seeder

```php
class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call('UserSeeder');
        $this->call('SettingsSeeder');
        $this->call('CategorySeeder');
    }
}
```

### Run Seeds

```bash
php spark db:seed DatabaseSeeder
php spark db:seed UserSeeder
```

---

## Database Configuration

```php
// app/Config/Database.php or .env
database.default.hostname = localhost
database.default.database = mydb
database.default.username = root
database.default.password = secret
database.default.DBDriver = MySQLi
database.default.port     = 3306
database.default.charset  = utf8mb4
database.default.DBCollat = utf8mb4_general_ci

// Multiple connections
database.tests.hostname = localhost
database.tests.database = test_db
```

### Using Multiple Connections

```php
// Default connection
$db = \Config\Database::connect();

// Named connection
$db = \Config\Database::connect('tests');

// In a model
protected $DBGroup = 'tests';
```

---

## Gotchas

- Backtick-quote reserved words in raw SQL: `` `key` ``, `` `order` ``, `` `index` ``
- `app_settings` tables often use `key` as a column name — always quote it in raw SQL
- `insertBatch()` in seeds ignores model `$allowedFields` — it inserts everything you give it
- Migration timestamps must be unique — if two migrations have the same timestamp, one may be skipped
- Always provide `down()` methods for rollback support
- Foreign keys should be dropped in `down()` before dropping the parent table
