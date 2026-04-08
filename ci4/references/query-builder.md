# CI4 Query Builder — Complete Reference

The Query Builder is available via `$this->db` in models, or `\Config\Database::connect()` anywhere.

```php
$db = \Config\Database::connect();
$builder = $db->table('users');
```

## SELECT

```php
$builder->select('id, name, email');
$builder->select('COUNT(*) AS total');
$builder->selectMin('age');
$builder->selectMax('age');
$builder->selectAvg('score');
$builder->selectSum('amount');
$builder->selectCount('id');
$builder->selectCount('id', 'user_count');  // aliased
```

## DISTINCT

```php
// CORRECT
$builder->select('role')->distinct();

// WRONG — silently returns wrong results in CI4
$builder->select('DISTINCT role');
```

## WHERE

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
$builder->orWhereIn('role', ['admin', 'superadmin']);
$builder->orWhereNotIn('status', ['inactive']);

// WHERE NULL — CRITICAL GOTCHA
$builder->where('deleted_at IS NULL');     // CORRECT
// $builder->whereNull('deleted_at');      // DOES NOT EXIST in CI4

// WHERE NOT NULL
$builder->where('deleted_at IS NOT NULL'); // CORRECT

// LIKE
$builder->like('name', 'rob');            // LIKE '%rob%'
$builder->like('name', 'rob', 'after');   // LIKE 'rob%'
$builder->like('name', 'rob', 'before');  // LIKE '%rob'
$builder->notLike('name', 'test');

// OR LIKE — GOTCHA: must use groupStart/groupEnd after a where()
$builder->where('active', 1)
    ->groupStart()
        ->like('name', 'rob')
        ->orLike('email', 'rob')
    ->groupEnd();

// BETWEEN (no native method — use where)
$builder->where('age >=', 18)->where('age <=', 65);

// Raw WHERE
$builder->where("YEAR(created_at) = ", 2024);
```

## GROUP START / GROUP END

For complex WHERE clauses with OR logic:

```php
// WHERE active = 1 AND (role = 'admin' OR role = 'superadmin')
$builder->where('active', 1)
    ->groupStart()
        ->where('role', 'admin')
        ->orWhere('role', 'superadmin')
    ->groupEnd();

// Nested groups
$builder->where('active', 1)
    ->groupStart()
        ->groupStart()
            ->where('role', 'admin')
            ->where('verified', 1)
        ->groupEnd()
        ->orGroupStart()
            ->where('role', 'superadmin')
        ->groupEnd()
    ->groupEnd();

// OR group start
$builder->where('status', 'active')
    ->orGroupStart()
        ->where('role', 'admin')
        ->where('override', 1)
    ->groupEnd();

// NOT group start
$builder->notGroupStart()
    ->where('role', 'banned')
    ->orWhere('role', 'suspended')
->groupEnd();
```

## GROUP BY / HAVING / ORDER BY / LIMIT

```php
$builder->groupBy('role');
$builder->groupBy(['role', 'department']);  // multiple
$builder->having('count >', 5);
$builder->having('total >=', 100);
$builder->orderBy('created_at', 'DESC');
$builder->orderBy('name', 'ASC');
$builder->orderBy('RANDOM()');            // random order
$builder->limit(10);
$builder->limit(10, 20);  // limit 10, offset 20
$builder->offset(20);
```

## JOINS

```php
$builder->join('orders', 'orders.user_id = users.id');
$builder->join('roles', 'roles.id = users.role_id', 'left');
$builder->join('profiles', 'profiles.user_id = users.id', 'left outer');

// Join types: 'inner' (default), 'left', 'right', 'outer', 'left outer', 'right outer'

// Complex join conditions
$builder->join('orders', 'orders.user_id = users.id AND orders.status = "active"', 'left');
```

## GET (Execute)

```php
$query  = $builder->get();              // run query
$rows   = $query->getResult();          // array of objects
$rows   = $query->getResultArray();     // array of arrays
$row    = $query->getRow();             // first row as object
$row    = $query->getRowArray();        // first row as array
$count  = $query->getNumRows();         // number of rows
$fields = $query->getFieldData();       // column metadata
$names  = $query->getFieldNames();      // column names

// Get with limit
$query = $builder->get(10);             // LIMIT 10
$query = $builder->get(10, 20);         // LIMIT 10 OFFSET 20
```

## INSERT

```php
$builder->insert(['name' => 'Rob', 'email' => 'r@b.com']);
$id = $db->insertID();  // last inserted ID

// Ignore duplicates
$builder->ignore(true)->insert($data);  // INSERT IGNORE
```

## UPDATE

```php
$builder->where('id', 1)->update(['name' => 'Robert']);

// SET
$builder->set('name', 'Robert');
$builder->set('views', 'views+1', false);  // false = no escaping (raw SQL)
$builder->where('id', 1)->update();

// Increment / Decrement (raw SQL via set)
$builder->set('count', 'count+1', false)->where('id', 1)->update();
$builder->set('stock', 'stock-1', false)->where('id', 1)->update();
```

## DELETE

```php
$builder->where('id', 1)->delete();
$builder->emptyTable();    // DELETE all rows (no WHERE)
$builder->truncate();      // TRUNCATE table (faster, resets auto-increment)
```

## Batch Operations

```php
// Insert multiple rows at once
$data = [
    ['name' => 'Alice', 'email' => 'alice@example.com'],
    ['name' => 'Bob',   'email' => 'bob@example.com'],
];
$builder->insertBatch($data);   // returns number of rows inserted

// Update multiple rows at once
$data = [
    ['id' => 1, 'name' => 'Alice Updated'],
    ['id' => 2, 'name' => 'Bob Updated'],
];
$builder->updateBatch($data, 'id');  // second arg = match column

// Upsert (insert or update on conflict)
$builder->upsertBatch($data);       // CI4 4.3+
```

**GOTCHA**: `insertBatch()` bypasses model `$allowedFields` — pass only the columns you intend to insert.

## Subqueries

```php
$subquery = $db->table('orders')->select('user_id')->where('total >', 100);
$builder->whereIn('id', $subquery);

// Subquery in SELECT
$builder->selectSubquery($db->table('orders')->selectCount('id')->where('orders.user_id = users.id'), 'order_count');
```

## Raw Queries

```php
// Positional binds
$query = $db->query("SELECT * FROM users WHERE id = ?", [$id]);
$rows  = $query->getResult();

// Named binds
$query = $db->query("SELECT * FROM users WHERE email = :email:", ['email' => $email]);

// Multiple results
$rows = $query->getResult();       // objects
$rows = $query->getResultArray();  // arrays
$row  = $query->getRow();          // single object
```

## Debugging

```php
// Get compiled SQL without executing
$sql = $builder->where('active', 1)->getCompiledSelect();
// → "SELECT * FROM `users` WHERE `active` = 1"

// Reset = false preserves the query for continued chaining
$sql = $builder->where('active', 1)->getCompiledSelect(false);
$rows = $builder->limit(10)->get()->getResult();

// Other compiled methods
$sql = $builder->getCompiledInsert();
$sql = $builder->getCompiledUpdate();
$sql = $builder->getCompiledDelete();

// Last query (after execution)
$lastQuery = $db->getLastQuery();
echo (string) $lastQuery;  // full SQL with values interpolated
```

---

## Transactions

Transactions wrap multiple queries so they either all succeed or all roll back.

### High-Level (Recommended)

```php
$db = \Config\Database::connect();

$db->transStart();

$db->table('orders')->insert(['user_id' => 1, 'total' => 50.00]);
$db->table('ticket_reservations')->insert(['order_id' => $db->insertID(), 'qty' => 2]);

if ($db->transStatus() === false) {
    // Something failed — transComplete() will roll back
}

$db->transComplete(); // commits on success, rolls back on failure
```

### Exception Mode

```php
$db->transException(true)->transStart();

try {
    $db->table('orders')->insert([...]);
    $db->table('tickets')->insert([...]);
    $db->transComplete();
} catch (\Throwable $e) {
    $db->transRollback();
    // handle error
}
```

### Manual Control

```php
$db->transBegin();
// ... queries ...
$db->transCommit();   // explicit commit
$db->transRollback(); // explicit rollback
```

### Transaction Notes
- `transStart()` is the recommended high-level wrapper; it calls `transBegin()` internally.
- Nested transactions use reference counting (`transDepth`) — the outermost `transComplete()` commits.
- `transStatus()` returns `false` if any query in the transaction failed.
