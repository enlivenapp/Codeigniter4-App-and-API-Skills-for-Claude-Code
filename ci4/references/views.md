# CI4 Views — Complete Reference

Views live in `app/Views/` as `.php` files.

## Basic View

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

## Layout + Section Pattern

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
    <?= $this->section('extra_scripts') ?>   // opened inside content
    <script>console.log('hi')</script>
    <?= $this->endSection() ?>
<?= $this->endSection() ?>

// CORRECT — close content first, then open extra_scripts
<?= $this->section('content') ?>
<p>Hello</p>
<?= $this->endSection() ?>                   // content closed

<?= $this->section('extra_scripts') ?>
<script>console.log('hi')</script>
<?= $this->endSection() ?>
```

## Partials

```php
// Include a partial (inherits parent view variables)
<?= $this->include('partials/_navbar') ?>

// Include with explicit data
<?= view('partials/_card', ['item' => $item]) ?>
```

**GOTCHA**: `$this->include()` does pass parent variables. `view()` inside a view does NOT pass the parent view's variables automatically — pass data explicitly.

## View Data Escaping

```php
esc($value);             // HTML (default)
esc($value, 'html');     // HTML entities
esc($value, 'js');       // JavaScript context
esc($value, 'attr');     // HTML attribute context
esc($value, 'url');      // URL encoding
esc($value, 'raw');      // No escaping (use carefully)
```

## View Cells

View Cells are mini-controllers that generate HTML fragments. Useful for reusable components.

```php
// Simple cell (function-based)
// app/Cells/RecentPostsCell.php
<?php
namespace App\Cells;

class RecentPostsCell
{
    public function render(array $params = []): string
    {
        $limit = $params['limit'] ?? 5;
        $posts = model('PostModel')->orderBy('created_at', 'DESC')->findAll($limit);
        return view('cells/recent_posts', ['posts' => $posts]);
    }
}

// Usage in any view
<?= view_cell('App\Cells\RecentPostsCell', ['limit' => 10]) ?>
<?= view_cell('\App\Cells\RecentPostsCell::render', 'limit=10') ?>
```

### Controlled Cells (Class-based)

```php
// app/Cells/AlertMessage.php
<?php
namespace App\Cells;

use CodeIgniter\View\Cells\Cell;

class AlertMessage extends Cell
{
    public string $type    = 'info';
    public string $message = '';

    // Computed property
    public function getClassAttribute(): string
    {
        return match($this->type) {
            'error'   => 'alert alert-danger',
            'success' => 'alert alert-success',
            default   => 'alert alert-info',
        };
    }
}

// app/Cells/alert_message.php (view — auto-discovered by naming convention)
<div class="<?= $classAttribute ?>">
    <?= esc($message) ?>
</div>

// Usage
<?= view_cell('AlertMessage', ['type' => 'success', 'message' => 'Saved!']) ?>
```

## Caching Views

```php
// Cache a view for 60 seconds
return view('expensive_page', $data, ['cache' => 60]);

// Cache with a custom name
return view('expensive_page', $data, ['cache' => 60, 'cache_name' => 'my_page_cache']);
```

## Conditional Display Patterns

```php
// Conditional content based on data
<?php if (empty($users)): ?>
    <p>No users found.</p>
<?php else: ?>
    <?php foreach ($users as $user): ?>
        <p><?= esc($user->name) ?></p>
    <?php endforeach; ?>
<?php endif; ?>

// Flash data display
<?php if (session()->getFlashdata('message')): ?>
    <div class="alert alert-success"><?= session()->getFlashdata('message') ?></div>
<?php endif; ?>

<?php if (session()->getFlashdata('error')): ?>
    <div class="alert alert-danger"><?= session()->getFlashdata('error') ?></div>
<?php endif; ?>

// Validation errors
<?php if (session()->getFlashdata('errors')): ?>
    <ul>
    <?php foreach (session()->getFlashdata('errors') as $error): ?>
        <li><?= esc($error) ?></li>
    <?php endforeach; ?>
    </ul>
<?php endif; ?>
```

## Form Helper in Views

```php
<?php helper('form'); ?>

<?= form_open('users/create') ?>
<?= form_open_multipart('users/create') ?>  <!-- for file uploads -->

<?= csrf_field() ?>  <!-- CSRF token hidden field -->

<?= form_input('name', old('name'), ['class' => 'form-control']) ?>
<?= form_password('password', '', ['class' => 'form-control']) ?>
<?= form_textarea('bio', old('bio'), ['rows' => 5]) ?>
<?= form_dropdown('role', ['admin' => 'Admin', 'user' => 'User'], old('role')) ?>
<?= form_checkbox('active', '1', old('active') == '1') ?>
<?= form_submit('submit', 'Save') ?>

<?= form_close() ?>

// Old input (repopulate after validation failure)
<input type="text" name="email" value="<?= old('email') ?>">
```
