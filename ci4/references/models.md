# CI4 Models — Complete Reference

## Full Model Template

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

## CRUD Operations

```php
$model = new UserModel();

// ─── Find ─────────────────────────────────────────────
$user  = $model->find(1);              // by primary key, returns object
$users = $model->find([1, 2, 3]);      // by multiple IDs
$users = $model->findAll();            // all rows
$user  = $model->first();             // first result
$users = $model->findAll(10, 20);     // limit 10, offset 20

// ─── Where ────────────────────────────────────────────
$user  = $model->where('email', 'a@b.com')->first();
$users = $model->where('active', 1)->findAll();
$users = $model->where('role', 'admin')->orderBy('name')->findAll();

// ─── Insert ───────────────────────────────────────────
$id = $model->insert(['name' => 'Rob', 'email' => 'r@b.com']);
// Returns inserted ID or false on failure

// ─── Update ───────────────────────────────────────────
$model->update(1, ['name' => 'Robert']);
// Update multiple
$model->whereIn('id', [1, 2])->set(['active' => 0])->update();

// ─── Delete ───────────────────────────────────────────
$model->delete(1);
$model->delete([1, 2, 3]);  // delete multiple

// ─── Count ────────────────────────────────────────────
$count = $model->countAll();
$count = $model->where('active', 1)->countAllResults();

// ─── Check existence ──────────────────────────────────
$exists = $model->where('email', 'r@b.com')->countAllResults() > 0;
```

## Save (Insert or Update)

`save()` decides insert vs update based on whether the primary key is present in data:

```php
// Insert (no primary key in data)
$model->save(['name' => 'Rob', 'email' => 'r@b.com']);

// Update (primary key present in data)
$model->save(['id' => 1, 'name' => 'Robert']);
```

## Soft Deletes

```php
protected $useSoftDeletes = true;
protected $deletedField   = 'deleted_at';

$model->delete(1);                    // sets deleted_at, does NOT remove row
$model->delete(1, true);             // hard delete (permanently removes row)
$model->withDeleted()->findAll();    // includes soft-deleted rows
$model->onlyDeleted()->findAll();    // only soft-deleted rows
$model->purgeDeleted();             // permanently remove all soft-deleted rows
```

## Timestamps

```php
protected $useTimestamps = true;
protected $createdField  = 'created_at';
protected $updatedField  = 'updated_at';
// Automatically sets created_at on insert, updated_at on update.
// Columns must exist in the table.

// Disable for a single operation
$model->skipTimestamps(true)->update(1, $data);
```

## Validation in Models

```php
protected $validationRules = [
    'email' => 'required|valid_email|is_unique[users.email,id,{id}]',
    'name'  => 'required|min_length[2]',
];

protected $validationMessages = [
    'email' => [
        'is_unique' => 'That email is already taken.',
    ],
];

$model->save($data);          // validates before saving
$model->errors();             // returns validation errors after failed save
$model->skipValidation(true); // bypass validation for this call

// {id} placeholder in is_unique — automatically replaced with the current
// record's primary key value during updates, so it won't fail on itself.
```

## Pagination

```php
// Controller
$users = $model->paginate(20);      // 20 per page, reads ?page= from URL
$pager = $model->pager;             // pager instance for view

// With where clause
$users = $model->where('active', 1)->paginate(20);
$pager = $model->pager;

// Named pager group (for multiple pagers on one page)
$users  = $model->paginate(20, 'users');
$events = $eventModel->paginate(10, 'events');

// View
echo $pager->links();               // full pagination links
echo $pager->simpleLinks();         // Previous / Next only
echo $pager->links('users', 'bootstrap_full');  // named group + template
```

## Query Scopes (Method Chaining)

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

public function recent(int $days = 30): static
{
    return $this->where('created_at >', date('Y-m-d', strtotime("-{$days} days")));
}

// Usage
$users = $model->active()->byRole('admin')->recent()->findAll();
```

## Callbacks (Model Events)

```php
protected $beforeInsert = ['hashPassword'];
protected $afterInsert  = ['logCreation'];
protected $beforeUpdate = ['hashPassword'];
protected $afterUpdate  = [];
protected $beforeFind   = [];
protected $afterFind    = [];
protected $beforeDelete = [];
protected $afterDelete  = [];

protected function hashPassword(array $data): array
{
    if (isset($data['data']['password'])) {
        $data['data']['password'] = password_hash($data['data']['password'], PASSWORD_DEFAULT);
    }
    return $data;
}

protected function logCreation(array $data): array
{
    log_message('info', 'User created with ID: ' . $data['id']);
    return $data;
}
```

### Callback Data Structure

- **beforeInsert**: `$data['data']` contains the row data being inserted
- **afterInsert**: `$data['id']` contains the new record's ID, `$data['data']` contains the row data
- **beforeUpdate**: `$data['data']` contains the update data, `$data['id']` contains the primary key(s)
- **afterUpdate**: same as beforeUpdate plus `$data['result']` (bool)
- **beforeFind**: `$data['method']` contains the find method name
- **afterFind**: `$data['data']` contains the found row(s)
- **beforeDelete**: `$data['id']` contains the primary key(s)
- **afterDelete**: `$data['id']` contains the primary key(s), `$data['result']` (bool)

## Entities

Entities are typed object representations of a row. Optional but useful for transformation logic.

```php
<?php
namespace App\Entities;

use CodeIgniter\Entity\Entity;

class User extends Entity
{
    // Cast columns to PHP types
    protected $casts = [
        'active'     => 'boolean',
        'metadata'   => 'json',
        'created_at' => 'datetime',
    ];

    // Custom setter — auto-transforms on assignment
    public function setPassword(string $pass): static
    {
        $this->attributes['password'] = password_hash($pass, PASSWORD_DEFAULT);
        return $this;
    }

    // Custom getter
    public function getDisplayName(): string
    {
        return $this->attributes['first_name'] . ' ' . $this->attributes['last_name'];
    }
}
```

### Using Entities with Models

```php
// In model
protected $returnType = 'App\Entities\User';

// Usage
$user = $model->find(1);        // returns User entity, not stdClass
$user->name = 'New Name';       // uses setter if defined
$user->password = 'secret';     // calls setPassword()
$model->save($user);            // entity implements toArray() for save
```

### Available Casts

| Cast | PHP Type |
|---|---|
| `'integer'` | int |
| `'float'` | float |
| `'double'` | float |
| `'string'` | string |
| `'boolean'` | bool |
| `'array'` | array (from JSON/serialized) |
| `'object'` | object (from JSON/serialized) |
| `'json'` | array (from JSON column) |
| `'json-array'` | array (from JSON column) |
| `'datetime'` | Time instance |
| `'timestamp'` | int (Unix timestamp) |
| `'uri'` | URI instance |
| `'int-bool'` | bool (from 0/1 column) |
| `'csv'` | array (from comma-separated string) |
