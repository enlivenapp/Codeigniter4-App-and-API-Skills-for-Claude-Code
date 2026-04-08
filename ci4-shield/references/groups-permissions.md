# Shield Groups & Permissions — Complete Reference

## Groups

Groups are Shield's role system — more flexible than traditional roles because users can belong to multiple groups simultaneously.

### Working with Groups

```php
$user = auth()->user();

// Add to group(s)
$user->addGroup('admin');
$user->addGroup('admin', 'developer');  // multiple at once

// Remove from group(s)
$user->removeGroup('admin');
$user->removeGroup('admin', 'developer');

// Check membership
$user->inGroup('admin');                // bool
$user->inGroup('admin', 'superadmin');  // true if in ANY of these

// Get all groups
$groups = $user->getGroups();  // ['admin', 'developer']
```

### Defining Groups

Groups are defined in `app/Config/AuthGroups.php`:

```php
public array $groups = [
    'superadmin' => [
        'title'       => 'Super Admin',
        'description' => 'Complete control of the site.',
    ],
    'admin' => [
        'title'       => 'Admin',
        'description' => 'Day to day administrators.',
    ],
    'developer' => [
        'title'       => 'Developer',
        'description' => 'Site developers.',
    ],
    'user' => [
        'title'       => 'User',
        'description' => 'General users.',
    ],
    'beta' => [
        'title'       => 'Beta Tester',
        'description' => 'Access to beta features.',
    ],
];

// New users automatically get this group
public string $defaultGroup = 'user';
```

### Checking in Controllers

```php
public function adminDashboard()
{
    if (! auth()->user()->inGroup('admin', 'superadmin')) {
        return redirect()->to('/')->with('error', 'Access denied.');
    }
    // ...
}
```

### Checking in Views

```php
<?php if (auth()->user()->inGroup('admin')): ?>
    <a href="/admin">Admin Panel</a>
<?php endif; ?>

<?php if (auth()->user()->inGroup('admin', 'developer')): ?>
    <a href="/admin/tools">Developer Tools</a>
<?php endif; ?>
```

### Route-Level Group Checks

```php
// Using Shield's group filter (preferred over controller checks)
$routes->get('admin', 'AdminController::index', ['filter' => 'group:admin,superadmin']);

$routes->group('admin', ['filter' => 'group:admin,superadmin'], static function ($routes) {
    $routes->get('/', 'Admin\DashboardController::index');
    $routes->resource('users');
});
```

---

## Permissions

Permissions provide fine-grained access control. They can be granted via the group matrix or directly to individual users.

### Defining Permissions

```php
// app/Config/AuthGroups.php
public array $permissions = [
    'admin.access'  => 'Can access the admin area',
    'admin.settings' => 'Can access site settings',
    'users.create'  => 'Can create new users',
    'users.edit'    => 'Can edit existing users',
    'users.delete'  => 'Can delete users',
    'posts.create'  => 'Can create posts',
    'posts.edit'    => 'Can edit posts',
    'posts.delete'  => 'Can delete posts',
    'posts.publish' => 'Can publish posts',
];
```

### Permission Matrix

The matrix assigns permissions to groups. This is the primary way permissions are granted:

```php
public array $matrix = [
    'superadmin' => ['admin.*', 'users.*', 'posts.*'],
    'admin'      => ['admin.access', 'users.create', 'users.edit', 'posts.*'],
    'developer'  => ['admin.access', 'posts.*'],
    'user'       => ['posts.create'],
    'beta'       => [],  // no default permissions
];
```

### Wildcard Permissions

`*` grants all permissions in a namespace:

```php
'superadmin' => ['admin.*']
// Grants: admin.access, admin.settings, admin.anything_else_defined
```

### Checking Permissions

```php
$user = auth()->user();

// Check (considers group matrix AND direct permissions)
$user->can('posts.create');       // bool
$user->can('admin.access');       // bool

// Inverse
$user->cannot('users.delete');    // bool
```

### Direct User Permissions

Beyond the group matrix, you can assign permissions directly to specific users:

```php
// Add direct permission
$user->addPermission('admin.access', 'users.create');

// Remove direct permission
$user->removePermission('admin.access');

// Get all direct permissions (not from groups)
$perms = $user->getPermissions();  // ['admin.access', 'users.create']

// Check if user has a specific direct permission
$user->hasPermission('admin.access');  // only checks direct, not group matrix
```

### Use Cases for Direct Permissions

- Granting a specific user access to something their group doesn't normally have
- Temporary elevated permissions (e.g., during an incident)
- One-off overrides without creating a new group

### Checking in Controllers

```php
public function editUser($id = null)
{
    if (! auth()->user()->can('users.edit')) {
        return redirect()->to('/')->with('error', 'Permission denied.');
    }
    // ...
}
```

### Checking in Views

```php
<?php if (auth()->user()->can('posts.create')): ?>
    <a href="/posts/new">New Post</a>
<?php endif; ?>

<?php if (auth()->user()->can('admin.settings')): ?>
    <a href="/admin/settings">Settings</a>
<?php endif; ?>
```

### Route-Level Permission Checks

```php
$routes->get('admin/settings', 'Admin\SettingsController::index', [
    'filter' => 'permission:admin.settings'
]);

$routes->group('admin', ['filter' => ['session', 'group:admin']], static function ($routes) {
    $routes->get('/', 'Admin\DashboardController::index');

    // Additional permission check within the group
    $routes->group('', ['filter' => 'permission:users.manage'], static function ($routes) {
        $routes->resource('users');
    });
});
```

---

## How Permissions Resolve

When `$user->can('posts.edit')` is called, Shield checks:

1. **Direct user permissions** — does this user have `posts.edit` directly?
2. **Group matrix** — is `posts.edit` (or `posts.*`) assigned to any of the user's groups?

If either is true, `can()` returns `true`.

### Permission Naming Convention

Use dot-notation namespaces: `resource.action`

```
admin.access
admin.settings
users.create
users.edit
users.delete
posts.create
posts.edit
posts.delete
posts.publish
media.upload
media.delete
```

This allows wildcard grants like `posts.*` or `admin.*`.
