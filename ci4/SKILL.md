---
name: ci4
description: Comprehensive CodeIgniter 4 framework skill. Use when working with any CodeIgniter 4 project — routing, controllers, models, views, query builder, migrations, filters, services, Spark CLI, and Shield auth. Activates on mentions of "CodeIgniter", "CI4", "spark", "CI4 model", "CI4 controller", "CI4 route", or any CI4-specific pattern.
version: 1.0.0
---

# CodeIgniter 4 — Complete Framework Reference

CodeIgniter 4 is a full-stack PHP 8+ MVC framework. It is **not Laravel**. Do not apply Laravel APIs, method names, or conventions here. When in doubt, check this document — not memory of another framework.

---

## Directory Structure

```
app/
  Config/         # All configuration classes (Routes, Database, Filters, Auth, etc.)
  Controllers/    # HTTP controllers
  Database/
    Migrations/   # Migration files (timestamped)
    Seeds/        # Seeder classes
  Filters/        # Before/after request filters
  Libraries/      # Custom libraries
  Models/         # Model classes
  Services/       # Custom service classes
  Views/          # View templates (.php)
    layouts/      # Layout templates
    partials/     # Reusable partials
public/           # Web root — index.php entry point lives here
writable/         # Cache, logs, sessions (must be writable)
tests/
vendor/
.env              # Environment config (copy from `env`)
spark             # CLI entry point
```

---

## MVC Conventions — Hard Rules

1. **Controllers handle HTTP** — receive request, call model/service, return response. No business logic.
2. **Models handle data** — all database interaction lives here. No HTTP concerns.
3. **Views handle display** — no DB calls, no business logic. Only presentation.
4. **Never call models from views.** Ever.
5. Web controllers call models directly — that is standard CI4. If the project has a separate API layer, check for a CI4 API skill for those conventions instead.
6. One controller = one resource. Name them descriptively (`UserController`, `EventController`).

---

## Configuration

All config lives in `app/Config/` as PHP classes (not arrays, not .ini files).

```php
// app/Config/Database.php — DB connection
// app/Config/Routes.php   — route definitions
// app/Config/Filters.php  — filter aliases + global filters
// app/Config/App.php      — base URL, timezone, etc.
```

`.env` overrides any config value using dot-notation:
```
database.default.hostname = localhost
database.default.database = mydb
app.baseURL = 'http://example.com/'
CI_ENVIRONMENT = development
```

---

## Routing

Defined in `app/Config/Routes.php`.

```php
$routes->get('users', 'UserController::index');
$routes->post('users', 'UserController::create');
$routes->get('users/(:num)', 'UserController::show/$1');
$routes->put('users/(:num)', 'UserController::update/$1');
$routes->delete('users/(:num)', 'UserController::delete/$1');

// Route groups
$routes->group('admin', ['namespace' => 'App\Controllers\Admin'], function ($routes) {
    $routes->get('users', 'UsersController::index');
    $routes->resource('events');
});

// Apply a filter to a route
$routes->get('dashboard', 'DashboardController::index', ['filter' => 'session']);

// Multiple filters
$routes->get('admin', 'AdminController::index', ['filter' => ['session', 'group:admin']]);

// Named routes
$routes->get('profile', 'ProfileController::index', ['as' => 'profile']);

// Redirect
$routes->addRedirect('old-path', 'new-path');
```

### Route Placeholders
| Placeholder | Matches |
|---|---|
| `(:num)` | Digits only |
| `(:alpha)` | Alphabetic only |
| `(:alphanum)` | Alphanumeric |
| `(:segment)` | URL segment (no slashes) |
| `(:any)` | Anything (use sparingly) |

### Resource Routes
```php
// Generates full CRUD routes
$routes->resource('photos');
// GET    /photos           → index()
// GET    /photos/new       → new()
// POST   /photos           → create()
// GET    /photos/(:segment) → show($id)
// GET    /photos/(:segment)/edit → edit($id)
// PUT    /photos/(:segment) → update($id)
// DELETE /photos/(:segment) → delete($id)

// Limit which methods are generated
$routes->resource('photos', ['only' => ['index', 'show', 'create']]);
$routes->resource('photos', ['except' => ['new', 'edit']]);
```

---

## Controllers

### Base Structure
```php
<?php
namespace App\Controllers;

use App\Controllers\BaseController;

class UserController extends BaseController
{
    public function index()
    {
        return view('users/index', ['users' => []]);
    }
}
```

### BaseController
`app/Controllers/BaseController.php` — extend this for all controllers. It provides:
- `$this->request` — IncomingRequest
- `$this->response` — ResponseInterface
- `$this->logger` — Logger

### ResourceController
Used for RESTful resources. Implements `index`, `show`, `new`, `edit`, `create`, `update`, `delete`.

```php
use CodeIgniter\RESTful\ResourceController;

class PhotoController extends ResourceController
{
    protected $modelName = 'App\Models\PhotoModel';
    protected $format    = 'json'; // response format

    public function index()
    {
        return $this->respond($this->model->findAll());
    }

    public function show($id = null)  // NEVER add type hints to override params
    {
        $photo = $this->model->find($id);
        if (!$photo) return $this->failNotFound('Photo not found');
        return $this->respond($photo);
    }
}
```

**GOTCHA**: Never add PHP type hints (`int $id`) to overridden ResourceController methods like `show($id = null)`. The parent signature uses `$id = null` — a type hint breaks the override.

### Request Data
```php
// GET params
$this->request->getGet('name');
$this->request->getGet(); // all GET

// POST params
$this->request->getPost('email');
$this->request->getPost(); // all POST

// JSON body (API endpoints)
$body = $this->request->getJSON(true); // true = associative array
// OR
$body = json_decode($this->request->getBody(), true);

// All input (GET + POST)
$this->request->getVar('key');

// File upload
$file = $this->request->getFile('avatar');
```

### Validation in Controllers
```php
$rules = [
    'email' => 'required|valid_email',
    'name'  => 'required|min_length[2]|max_length[100]',
];

if (!$this->validate($rules)) {
    // Returns validation errors
    return redirect()->back()->withInput()->with('errors', $this->validator->getErrors());
    // or for API:
    return $this->failValidationErrors($this->validator->getErrors());
}
```

### Redirects
```php
return redirect()->to('/dashboard');
return redirect()->back();
return redirect()->route('profile');
return redirect()->back()->withInput();
return redirect()->back()->with('message', 'Saved!');
return redirect()->back()->with('error', 'Something went wrong.');
```

### Session Flash Data
```php
session()->setFlashdata('message', 'User created.');
session()->setFlashdata('error', 'Something went wrong.');
// In view:
// session()->getFlashdata('message')
```

---

## Models

### Full Model Template
```php
<?php
namespace App\Models;

use CodeIgniter\Model;

class UserModel extends Model
{
    protected $table            = 'users';
    protected $primaryKey       = 'id';
    protected $useAutoIncrement = true;
    protected $returnType       = 'object';   // Always use 'object' (not 'array')
    protected $useSoftDeletes   = false;
    protected $protectFields    = true;
    protected $allowedFields    = ['name', 'email', 'role'];

    // Timestamps
    protected $useTimestamps = true;
    protected $createdField  = 'created_at';
    protected $updatedField  = 'updated_at';
    protected $deletedField  = 'deleted_at'; // only needed with soft deletes

    // Validation
    protected $validationRules      = [];
    protected $validationMessages   = [];
    protected $skipValidation       = false;

    // Callbacks
    protected $allowCallbacks = true;
    protected $beforeInsert   = [];
    protected $afterInsert    = [];
    protected $beforeUpdate   = [];
    protected $afterUpdate    = [];
    protected $beforeFind     = [];
    protected $afterFind      = [];
    protected $beforeDelete   = [];
    protected $afterDelete    = [];
}
```

### CRUD Operations
```php
$model = new UserModel();

// Find
$user  = $model->find(1);              // by primary key, returns object
$users = $model->findAll();            // all rows
$user  = $model->first();             // first result
$users = $model->findAll(10, 20);     // limit 10, offset 20

// Where
$user  = $model->where('email', 'a@b.com')->first();
$users = $model->where('active', 1)->findAll();

// Insert
$id = $model->insert(['name' => 'Rob', 'email' => 'r@b.com']);
// Returns inserted ID or false on failure

// Update
$model->update(1, ['name' => 'Robert']);

// Delete
$model->delete(1);

// Soft delete restore
$model->withDeleted()->find(1);   // includes soft-deleted rows
$model->onlyDeleted()->findAll(); // only soft-deleted rows

// Count
$count = $model->countAll();
$count = $model->where('active', 1)->countAllResults();

// Check existence
$exists = $model->where('email', 'r@b.com')->countAllResults() > 0;
```

### Soft Deletes
```php
protected $useSoftDeletes = true;
protected $deletedField   = 'deleted_at';

$model->delete(1);                    // sets deleted_at, does NOT remove row
$model->delete(1, true);             // hard delete (removes row)
$model->withDeleted()->findAll();    // includes soft-deleted
$model->onlyDeleted()->findAll();    // only soft-deleted
```

### Timestamps
```php
protected $useTimestamps = true;
// Automatically sets created_at on insert, updated_at on update.
// Columns must exist in the table.
```

### Validation in Models
```php
protected $validationRules = [
    'email' => 'required|valid_email|is_unique[users.email,id,{id}]',
    'name'  => 'required|min_length[2]',
];

$model->save($data);          // validates before saving
$model->errors();             // returns validation errors after failed save
$model->skipValidation(true); // bypass validation for this call
```

### Saving (insert or update)
```php
// insert (no primary key in data)
$model->save(['name' => 'Rob', 'email' => 'r@b.com']);

// update (primary key present in data)
$model->save(['id' => 1, 'name' => 'Robert']);
```

### Pagination
```php
$users = $model->paginate(20);          // 20 per page, reads ?page= from URL
$pager = $model->pager;                 // pager instance for view

// In view:
echo $pager->links();
echo $pager->simpleLinks();
```

### Query Scopes (method chaining)
```php
// Define chainable scope methods that return static
public function active(): static
{
    return $this->where('active', 1);
}

public function byRole(string $role): static
{
    return $this->where('role', $role);
}

// Usage
$users = $model->active()->byRole('admin')->findAll();
```

### Observers / Callbacks
```php
protected $beforeInsert = ['hashPassword'];

protected function hashPassword(array $data): array
{
    if (isset($data['data']['password'])) {
        $data['data']['password'] = password_hash($data['data']['password'], PASSWORD_DEFAULT);
    }
    return $data;
}
```

---

## Query Builder

The Query Builder is available via `$this->db` in models, or `\Config\Database::connect()` anywhere.

```php
$db = \Config\Database::connect();
$builder = $db->table('users');
```

### SELECT
```php
$builder->select('id, name, email');
$builder->select('COUNT(*) AS total');
$builder->selectMin('age');
$builder->selectMax('age');
$builder->selectAvg('score');
$builder->selectSum('amount');
```

### DISTINCT
```php
// CORRECT
$builder->select('role')->distinct();

// WRONG — silently returns wrong results in CI4
$builder->select('DISTINCT role');
```

### WHERE
```php
$builder->where('active', 1);
$builder->where('age >', 18);
$builder->where('name !=', 'Rob');
$builder->where('created_at >', '2024-01-01');

// Multiple WHERE (AND)
$builder->where('active', 1)->where('role', 'admin');

// OR WHERE
$builder->orWhere('role', 'superadmin');

// WHERE IN
$builder->whereIn('id', [1, 2, 3]);
$builder->whereNotIn('status', ['banned', 'suspended']);

// WHERE NULL — CRITICAL GOTCHA
$builder->where('deleted_at IS NULL');     // CORRECT
// $builder->whereNull('deleted_at');      // DOES NOT EXIST in CI4 (Laravel only)

// WHERE NOT NULL
$builder->where('deleted_at IS NOT NULL'); // CORRECT

// LIKE
$builder->like('name', 'rob');         // LIKE '%rob%'
$builder->like('name', 'rob', 'after'); // LIKE 'rob%'
$builder->like('name', 'rob', 'before'); // LIKE '%rob'

// OR LIKE — GOTCHA: must use groupStart/groupEnd after a where()
$builder->where('active', 1)
        ->groupStart()
            ->like('name', 'rob')
            ->orLike('email', 'rob')
        ->groupEnd();
```

### GROUP BY / ORDER BY / LIMIT
```php
$builder->groupBy('role');
$builder->having('count > ', 5);
$builder->orderBy('created_at', 'DESC');
$builder->orderBy('name', 'ASC');
$builder->limit(10);
$builder->limit(10, 20); // limit 10, offset 20
$builder->offset(20);
```

### JOINS
```php
$builder->join('orders', 'orders.user_id = users.id');
$builder->join('roles', 'roles.id = users.role_id', 'left');
// Join types: 'left', 'right', 'outer', 'inner', 'left outer', 'right outer'
```

### GET (execute)
```php
$query  = $builder->get();              // run query
$rows   = $query->getResult();          // array of objects
$rows   = $query->getResultArray();     // array of arrays
$row    = $query->getRow();             // first row as object
$row    = $query->getRowArray();        // first row as array
$value  = $query->getFieldData();       // column metadata
```

### INSERT
```php
$builder->insert(['name' => 'Rob', 'email' => 'r@b.com']);
$db->insertID(); // last inserted ID
```

### UPDATE
```php
$builder->where('id', 1)->update(['name' => 'Robert']);
```

### DELETE
```php
$builder->where('id', 1)->delete();
$builder->emptyTable(); // DELETE all rows
$builder->truncate();   // TRUNCATE table
```

### Raw Queries
```php
$query = $db->query("SELECT * FROM users WHERE id = ?", [$id]);
$rows  = $query->getResult();

// Named binds
$query = $db->query("SELECT * FROM users WHERE email = :email:", ['email' => $email]);
```

### Subqueries
```php
$subquery = $db->table('orders')->select('user_id')->where('total >', 100);
$builder->whereIn('id', $subquery);
```

---

## Entities

Entities are typed object representations of a row. Optional but useful for transformation logic.

```php
<?php
namespace App\Entities;

use CodeIgniter\Entity\Entity;

class User extends Entity
{
    protected $casts = [
        'active'     => 'boolean',
        'metadata'   => 'json',
        'created_at' => 'datetime',
    ];

    public function setPassword(string $pass): static
    {
        $this->attributes['password'] = password_hash($pass, PASSWORD_DEFAULT);
        return $this;
    }
}
```

In model:
```php
protected $returnType = 'App\Entities\User';
```

---

## Views

### Basic View
```php
// In controller
return view('users/index', ['users' => $users, 'title' => 'Users']);

// In view (app/Views/users/index.php)
<h1><?= esc($title) ?></h1>
<?php foreach ($users as $user): ?>
    <p><?= esc($user->name) ?></p>
<?php endforeach; ?>
```

**Always use `esc()` for output** — it XSS-escapes by default.

### Layout + Section Pattern
```php
// Layout (app/Views/layouts/main.php)
<!DOCTYPE html>
<html>
<head><title><?= $this->renderSection('title') ?></title></head>
<body>
    <?= $this->renderSection('content') ?>
    <?= $this->renderSection('extra_scripts') ?>
</body>
</html>

// Child view
<?= $this->extend('layouts/main') ?>

<?= $this->section('title') ?>My Page<?= $this->endSection() ?>

<?= $this->section('content') ?>
<p>Hello</p>
<?= $this->endSection() ?>

<?= $this->section('extra_scripts') ?>
<script>console.log('hi')</script>
<?= $this->endSection() ?>
```

**GOTCHA: Never use `return` or early exit inside a CI4 view that uses `$this->extend()`.** Layout rendering requires all sections to complete. Use `if/else` to conditionally show content, never `return`.

**GOTCHA: CI4 view sections cannot be nested.** Always call `$this->endSection()` for the content section BEFORE starting `$this->section('extra_scripts')`. This fails silently — no error, just missing output.

```php
// WRONG — nested sections, scripts will be swallowed
<?= $this->section('content') ?>
<p>Hello</p>
    <?= $this->section('extra_scripts') ?>   // ← opened inside content
    <script>console.log('hi')</script>
    <?= $this->endSection() ?>
<?= $this->endSection() ?>

// CORRECT — close content first, then open extra_scripts
<?= $this->section('content') ?>
<p>Hello</p>
<?= $this->endSection() ?>                   // ← content closed

<?= $this->section('extra_scripts') ?>
<script>console.log('hi')</script>
<?= $this->endSection() ?>
```

### Partials
```php
// Include a partial
<?= $this->include('partials/_navbar') ?>

// Include with data
<?= view('partials/_card', ['item' => $item]) ?>
```

### View Data Escaping
```php
esc($value);             // HTML (default)
esc($value, 'html');     // HTML
esc($value, 'js');       // JavaScript context
esc($value, 'attr');     // HTML attribute
esc($value, 'url');      // URL
esc($value, 'raw');      // No escaping
```

---

## Filters

> **Coming from older CI versions?** Auth checks and request guards were commonly handled in `function __construct()` (no visibility keyword) inside controllers. CI4's Filter system is the proper place for that logic — move it out of constructors and into a Filter class.

Filters run before and/or after a controller method. Defined in `app/Filters/` and registered in `app/Config/Filters.php`.

### Creating a Filter
```php
<?php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class AuthFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        if (!auth()->loggedIn()) {
            return redirect()->to('/login');
        }
        // Return nothing to continue; return a Response to stop the chain
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
        // Runs after the controller. Modify $response or return nothing.
    }
}
```

### Registering Filters
```php
// app/Config/Filters.php
public array $aliases = [
    'session'    => \App\Filters\SessionAuthFilter::class,
    'api_auth'   => \App\Filters\ApiAuthFilter::class,
    'api_group'  => \App\Filters\ApiGroupFilter::class,
];

// Apply globally
public array $globals = [
    'before' => [],
    'after'  => [],
];
```

### Applying Filters to Routes
```php
// Single filter
$routes->get('admin', 'AdminController::index', ['filter' => 'session']);

// Multiple filters
$routes->get('admin', 'AdminController::index', ['filter' => ['session', 'api_group:admin']]);

// Filter with arguments (accessible as $arguments in filter)
['filter' => 'api_group:admin,superadmin']
```

---

## Database Migrations

### Create a Migration
```bash
php spark make:migration CreateUsersTable
```

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
            'email'      => ['type' => 'VARCHAR', 'constraint' => 255, 'unique' => true],
            'active'     => ['type' => 'TINYINT', 'constraint' => 1, 'default' => 1],
            'created_at' => ['type' => 'DATETIME', 'null' => true],
            'updated_at' => ['type' => 'DATETIME', 'null' => true],
            'deleted_at' => ['type' => 'DATETIME', 'null' => true],
        ]);

        $this->forge->addPrimaryKey('id');
        $this->forge->addKey('email', true); // unique index
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

### Common Field Types
```php
'id'         => ['type' => 'INT', 'unsigned' => true, 'auto_increment' => true]
'name'       => ['type' => 'VARCHAR', 'constraint' => 150]
'body'       => ['type' => 'TEXT']
'price'      => ['type' => 'DECIMAL', 'constraint' => '10,2']
'active'     => ['type' => 'TINYINT', 'constraint' => 1, 'default' => 1]
'status'     => ['type' => 'ENUM', 'constraint' => ['pending','active','closed'], 'default' => 'pending']
'metadata'   => ['type' => 'JSON', 'null' => true]
'created_at' => ['type' => 'DATETIME', 'null' => true]
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
            ['name' => 'Admin', 'email' => 'admin@example.com', 'role' => 'admin'],
        ];
        $this->db->table('users')->insertBatch($data);
    }
}

// Master seeder calling others
class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call('UserSeeder');
        $this->call('SettingsSeeder');
    }
}
```

---

## Services

Services are singleton-like helpers, resolved via `\Config\Services`.

```php
// Built-in services
$session    = \Config\Services::session();
$validation = \Config\Services::validation();
$uri        = \Config\Services::uri();
$request    = \Config\Services::request();
$response   = \Config\Services::response();
$cache      = \Config\Services::cache();
$logger     = \Config\Services::logger();
$email      = \Config\Services::email();

// Custom service — register in app/Config/Services.php
public static function stripe(bool $getShared = true): \App\Services\StripeService
{
    if ($getShared) return static::getSharedInstance('stripe');
    return new \App\Services\StripeService();
}

// Usage
$stripe = \Config\Services::stripe();
```

---

## Spark CLI

```bash
# Development server (public/ as root)
php spark serve
php spark serve --port 8080

# List all registered routes
php spark routes

# Migrations
php spark migrate                          # run pending migrations
php spark migrate:rollback                 # roll back last batch
php spark migrate:rollback -b 2           # roll back 2 batches
php spark migrate:status                  # show migration status
php spark migrate:refresh                 # rollback all + re-migrate
php spark migrate:reset                   # rollback all

# Seeds
php spark db:seed DatabaseSeeder          # run a seeder

# Generators (scaffold)
php spark make:controller UserController
php spark make:controller Api/UserController --restful   # REST resource
php spark make:model UserModel
php spark make:model UserModel --entity   # with Entity class
php spark make:migration CreateUsersTable
php spark make:filter AuthFilter
php spark make:seeder UserSeeder
php spark make:entity UserEntity
php spark make:library MyLibrary
php spark make:command MyCommand

# Cache
php spark cache:clear

# Session
php spark session:migration               # generate session DB table migration

# Key
php spark key:generate                    # generate encryption key for .env
```

---

## Helpers

```php
// Load a helper
helper('url');         // url_to(), base_url(), site_url(), redirect()
helper('form');        // form_open(), form_input(), etc.
helper('text');        // word_limiter(), character_limiter()
helper('filesystem');  // write_file(), read_file(), delete_files()
helper('date');        // now(), timezone_select()

// Load multiple
helper(['url', 'form', 'text']);

// Auto-loaded globally in BaseController or Config/Autoload.php
```

### Common Helper Functions
```php
base_url('path/to/resource');   // full URL from web root
site_url('users/1');            // full URL with index.php (if not hidden)
url_to('ControllerName::method', $arg); // URL from route name or controller
current_url();                  // current full URL
previous_url();                 // previous URL
redirect()->to(url_to('home')); // redirect
esc($value);                    // XSS escape
log_message('debug', 'msg');    // write to writable/logs
```

---

## Caching

```php
$cache = \Config\Services::cache();

$cache->save('key', $data, 300);        // save for 300 seconds
$data = $cache->get('key');             // retrieve (null if missing/expired)
$cache->delete('key');                  // delete
$cache->clean();                        // clear all

// Remember pattern
$data = cache()->remember('key', 300, function () {
    return $this->model->expensive();
});
```

---

## Email

```php
$email = \Config\Services::email();

$email->setFrom('noreply@example.com', 'My App');
$email->setTo('user@example.com');
$email->setSubject('Welcome!');
$email->setMessage(view('emails/welcome', ['name' => $name]));
$email->send();
```

Config in `app/Config/Email.php` or `.env`:
```
email.fromEmail = noreply@example.com
email.fromName = My App
email.SMTPHost = smtp.example.com
email.SMTPUser = user
email.SMTPPass = pass
email.SMTPPort = 587
email.SMTPCrypto = tls
email.protocol = smtp
```

---

## Shield (Authentication & Authorization)

Shield is the official CI4 auth library. Install via Composer and run `php spark shield:setup`.

### Setup
```bash
composer require codeigniter4/shield
php spark shield:setup   # publishes config + migration
php spark migrate
```

### Checking Auth State
```php
auth()->loggedIn()          // bool
auth()->user()              // current User entity
auth()->id()                // current user ID
auth()->check(['token' => $token])  // validate a token
```

### Session Auth (web)
```php
// Login
$result = auth()->attempt(['email' => $email, 'password' => $pass]);
if ($result->isOK()) {
    return redirect()->to('/dashboard');
}

// Logout
auth()->logout();
return redirect()->to('/login');
```

### Token Auth (API)
```php
// Generate token for a user
$user->generateAccessToken('token-name');  // returns AccessToken entity
$token->raw_token;  // the plain-text token (only available on first generation)

// Authenticate a request (Bearer token in Authorization header)
$result = auth('tokens')->attempt();
// or via filter: ['filter' => 'auth:tokens']

// Get the current authed user in API controller
$user = auth('tokens')->user();
```

### Groups (Authorization)
```php
// Add user to a group
$user->addGroup('admin');

// Remove from group
$user->removeGroup('admin');

// Check group membership
$user->inGroup('admin');             // bool
$user->inGroup('admin', 'superadmin'); // in any of these groups

// Get all groups
$user->getGroups();
```

Groups are defined in `app/Config/AuthGroups.php`:
```php
public array $groups = [
    'superadmin'       => ['title' => 'Super Admin',       'description' => '...'],
    'admin'            => ['title' => 'Administrator',     'description' => '...'],
    'customer_service' => ['title' => 'Customer Service',  'description' => '...'],
    'user'             => ['title' => 'User',               'description' => '...'],
];
```

### Shield Filters
```php
// Protect routes with session auth
$routes->get('dashboard', 'DashboardController::index', ['filter' => 'session']);

// Protect API routes with token auth
$routes->get('api/v1/me', 'Api\UserController::me', ['filter' => 'auth:tokens']);

// Group-restricted route (custom filter reading Shield groups)
$routes->get('admin', 'AdminController::index', ['filter' => ['session', 'group:admin']]);
```

### Shield Permissions
Permissions are defined in `app/Config/AuthGroups.php` under `$permissions` and `$matrix`. Use `$user->can('permission.action')` to check.

---

## Common Pitfalls & Gotchas

### Query Builder
- `whereNull()` does **not exist** in CI4. Use `->where('col IS NULL')`.
- `->select('DISTINCT col')` returns wrong results silently. Use `->select('col')->distinct()`.
- `orLike()` after a `where()` needs `groupStart()` / `groupEnd()` to produce correct SQL.
- `->select()` is cumulative — calling it multiple times appends, not replaces.

### Models
- `protected $returnType = 'object'` — use this universally. `'array'` means `$row['key']` instead of `$row->key`, which is inconsistent and error-prone.
- `getBody()` (or `getJSON(true)`) returns `array` — use `$body['key']`, not `$body->key`.
- Model `save()` decides insert vs update based on whether the primary key is present in data — no magic `upsert()`.
- Soft delete `delete()` only sets `deleted_at`; queries automatically exclude soft-deleted rows. Use `withDeleted()` to include them.

### Controllers
- Never type-hint overridden ResourceController params: `show($id = null)` not `show(int $id = null)`.
- If your API controller extends `ResourceController`, it already has a `protected format()` method — never define a private method named `format()` in the subclass (access level conflict / fatal error).
- `redirect()` must be `return`ed — `redirect()->to('/foo')` without `return` does nothing.

### Views
- Never use `return` or early exit inside a view using `$this->extend()`. It prevents sections from completing, causing blank output.
- View sections cannot be nested. Close the content section with `endSection()` before opening `extra_scripts`.
- `$this->include()` does not pass the parent view's variables automatically — pass data explicitly: `view('partial', ['var' => $var])`.

### Database / Migrations
- Backtick-quote reserved words in raw SQL: `` `key` ``, `` `order` ``, `` `index` ``.
- `app_settings` tables often use `key` as a column name — always quote it in raw SQL.
- `insertBatch()` ignores `$allowedFields` — it inserts everything you give it. Be deliberate about what you pass.

### CI4 vs Laravel — Do Not Confuse
| Laravel | CI4 Equivalent |
|---|---|
| `User::find(1)` | `$model->find(1)` |
| `User::where()->get()` | `$model->where()->findAll()` |
| `whereNull('col')` | `->where('col IS NULL')` |
| `orWhere()` | `->orWhere()` ✓ same |
| `Request::input('key')` | `$this->request->getPost('key')` or `getVar()` |
| `$request->validate([])` | `$this->validate([])` |
| `dd()` | `d()` (CI4) or `var_dump(); exit;` |
| `artisan` | `spark` |
| `storage/` | `writable/` |
| Eloquent scopes | Method chaining returning `static` |
| Middleware | Filters |
| Service Providers | `app/Config/Services.php` |
| Facades | `\Config\Services::serviceName()` |

---

## Testing

CI4 uses PHPUnit. Test files go in `tests/`.

```bash
composer test
# or
php vendor/bin/phpunit
php vendor/bin/phpunit tests/unit/UserModelTest.php
```

### Unit Test
```php
<?php
namespace Tests\Unit;

use CodeIgniter\Test\CIUnitTestCase;
use App\Models\UserModel;

class UserModelTest extends CIUnitTestCase
{
    public function testFindUser(): void
    {
        $model = new UserModel();
        $user  = $model->find(1);
        $this->assertNotNull($user);
        $this->assertEquals('Rob', $user->name);
    }
}
```

### Feature / HTTP Test
```php
use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\FeatureTestTrait;

class UserControllerTest extends CIUnitTestCase
{
    use FeatureTestTrait;

    public function testIndex(): void
    {
        $result = $this->get('users');
        $result->assertStatus(200);
        $result->assertSee('Users');
    }

    public function testCreate(): void
    {
        $result = $this->post('users', ['name' => 'Rob', 'email' => 'r@b.com']);
        $result->assertRedirectTo(base_url('users'));
    }
}
```

### Database Test Traits
```php
use CodeIgniter\Test\DatabaseTestTrait;

// Rolls back DB changes after each test
protected $refresh = true;
protected $seed    = 'UserSeeder';
```
