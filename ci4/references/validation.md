# CI4 Validation — Complete Reference

## Using Validation

### In Controllers

```php
$rules = [
    'email' => 'required|valid_email',
    'name'  => 'required|min_length[2]|max_length[100]',
];

if (!$this->validate($rules)) {
    return redirect()->back()->withInput()->with('errors', $this->validator->getErrors());
}

// With custom error messages
$messages = [
    'email' => [
        'required'    => 'Email is required.',
        'valid_email' => 'Please enter a valid email.',
    ],
];
if (!$this->validate($rules, $messages)) { ... }
```

### validateData() — Validate Arbitrary Data

```php
// Validate data not from the request (useful for file upload rules)
if (!$this->validateData($data, $rules)) { ... }
if (!$this->validateData([], $fileRules)) { ... }  // file validation
```

### In Models

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

// {id} placeholder — auto-replaced with the current record's
// primary key during updates, so uniqueness check skips the current row
```

### Standalone Validation Service

```php
$validation = \Config\Services::validation();

$validation->setRules([
    'email' => 'required|valid_email',
    'name'  => 'required',
]);

if (!$validation->run($data)) {
    $errors = $validation->getErrors();
}
```

## All Built-in Rules

### General Rules

| Rule | Description | Example |
|---|---|---|
| `required` | Field must be present and not empty | `required` |
| `permit_empty` | Allow empty value, skip other rules if empty | `permit_empty\|valid_email` |
| `if_exist` | Only validate if field exists in data | `if_exist\|min_length[2]` |
| `in_list` | Must be one of the listed values | `in_list[admin,user,staff]` |
| `not_in_list` | Must NOT be one of the listed values | `not_in_list[banned,deleted]` |
| `matches` | Must match another field | `matches[password_confirm]` |
| `differs` | Must differ from another field | `differs[username]` |
| `is_unique` | Must be unique in DB table | `is_unique[users.email]` |
| `is_not_unique` | Must already exist in DB table | `is_not_unique[roles.name]` |

### String Rules

| Rule | Description | Example |
|---|---|---|
| `min_length` | Minimum string length | `min_length[3]` |
| `max_length` | Maximum string length | `max_length[255]` |
| `exact_length` | Exact string length | `exact_length[10]` |
| `alpha` | Only alphabetic characters | `alpha` |
| `alpha_numeric` | Only alphanumeric | `alpha_numeric` |
| `alpha_numeric_space` | Alphanumeric + spaces | `alpha_numeric_space` |
| `alpha_dash` | Alpha + dashes + underscores | `alpha_dash` |
| `alpha_numeric_punct` | Alphanumeric + common punctuation | `alpha_numeric_punct` |
| `regex_match` | Must match regex pattern | `regex_match[/^[A-Z]/]` |
| `valid_email` | Valid email format | `valid_email` |
| `valid_emails` | Comma-separated valid emails | `valid_emails` |
| `valid_url` | Valid URL | `valid_url` |
| `valid_url_strict` | Valid URL (stricter) | `valid_url_strict[https]` |
| `valid_ip` | Valid IP address | `valid_ip` |
| `valid_base64` | Valid base64 string | `valid_base64` |
| `valid_json` | Valid JSON string | `valid_json` |
| `valid_date` | Valid date string | `valid_date[Y-m-d]` |

### Numeric Rules

| Rule | Description | Example |
|---|---|---|
| `numeric` | Numeric (including decimals) | `numeric` |
| `integer` | Integer only | `integer` |
| `decimal` | Decimal number | `decimal` |
| `is_natural` | Natural number (0+) | `is_natural` |
| `is_natural_no_zero` | Natural number (1+) | `is_natural_no_zero` |
| `greater_than` | Greater than value | `greater_than[0]` |
| `greater_than_equal_to` | Greater than or equal | `greater_than_equal_to[1]` |
| `less_than` | Less than value | `less_than[100]` |
| `less_than_equal_to` | Less than or equal | `less_than_equal_to[99]` |

### Database Rules

| Rule | Description | Example |
|---|---|---|
| `is_unique` | Unique in table.column | `is_unique[users.email]` |
| `is_unique` | Unique, ignoring a row | `is_unique[users.email,id,{id}]` |
| `is_not_unique` | Must exist in table | `is_not_unique[roles.name]` |

The `is_unique` ignore syntax: `is_unique[table.column,ignore_field,ignore_value]`
- During updates, use `{id}` as ignore_value — auto-replaced with the current record's primary key
- Example: `is_unique[users.email,id,{id}]` — unique email, but skip the current user

### File Upload Rules

**Important**: File rules must use `validateData([], $rules)` not `validate($rules)`.

| Rule | Description | Example |
|---|---|---|
| `uploaded` | File was actually uploaded | `uploaded[avatar]` |
| `max_size` | Max file size in KB | `max_size[avatar,2048]` |
| `max_dims` | Max image dimensions | `max_dims[avatar,1024,768]` |
| `min_dims` | Min image dimensions (4.6+) | `min_dims[avatar,100,100]` |
| `mime_in` | Allowed MIME types | `mime_in[avatar,image/png,image/jpeg]` |
| `ext_in` | Allowed extensions | `ext_in[avatar,png,jpg,gif]` |
| `is_image` | Must be an image | `is_image[avatar]` |

```php
$rules = [
    'avatar' => [
        'label' => 'Avatar',
        'rules' => [
            'uploaded[avatar]',
            'is_image[avatar]',
            'mime_in[avatar,image/jpg,image/jpeg,image/png,image/webp]',
            'max_size[avatar,2048]',
            'max_dims[avatar,1024,768]',
        ],
    ],
];

if (!$this->validateData([], $rules)) { ... }
```

## Custom Validation Rules

### Inline Closure

```php
$rules = [
    'username' => [
        'required',
        static function (string $value): bool {
            return !str_contains($value, 'admin');
        },
    ],
];
```

### Rule Class

```php
// app/Validation/CustomRules.php
<?php
namespace App\Validation;

class CustomRules
{
    public function not_reserved(string $str): bool
    {
        $reserved = ['admin', 'root', 'system'];
        return !in_array(strtolower($str), $reserved);
    }

    public function valid_slug(string $str): bool
    {
        return preg_match('/^[a-z0-9-]+$/', $str) === 1;
    }
}

// Register in app/Config/Validation.php
public array $ruleSets = [
    \CodeIgniter\Validation\StrictRules\CreditCardRules::class,
    \CodeIgniter\Validation\StrictRules\FileRules::class,
    \CodeIgniter\Validation\StrictRules\FormatRules::class,
    \CodeIgniter\Validation\StrictRules\Rules::class,
    \App\Validation\CustomRules::class,  // add your class
];

// Usage
$rules = ['slug' => 'required|valid_slug'];
```

## Validation Rule Groups

Define reusable rule sets in `app/Config/Validation.php`:

```php
// app/Config/Validation.php
public array $userCreate = [
    'name'  => 'required|min_length[2]|max_length[150]',
    'email' => 'required|valid_email|is_unique[users.email]',
    'role'  => 'required|in_list[admin,user]',
];

public array $userUpdate = [
    'name'  => 'if_exist|min_length[2]|max_length[150]',
    'email' => 'if_exist|valid_email',
    'role'  => 'if_exist|in_list[admin,user]',
];

// Usage in controller
if (!$this->validate('userCreate')) { ... }
```

## Error Display

```php
// Get all errors
$errors = $this->validator->getErrors();
// ['email' => 'The email field is required.', 'name' => 'The name field...']

// Get single field error
$error = $this->validator->getError('email');

// Check if field has error
$this->validator->hasError('email');  // bool

// In view — display all errors
<?php if (session()->getFlashdata('errors')): ?>
<ul>
    <?php foreach (session()->getFlashdata('errors') as $error): ?>
        <li><?= esc($error) ?></li>
    <?php endforeach; ?>
</ul>
<?php endif; ?>

// In view — display per-field error
<?php if (isset($errors['email'])): ?>
    <span class="text-danger"><?= esc($errors['email']) ?></span>
<?php endif; ?>
```
