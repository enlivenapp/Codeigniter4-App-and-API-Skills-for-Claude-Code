# CI4 Services, Helpers, Caching, Email & Sessions Reference

## Services

Services are singleton-like helpers, resolved via `\Config\Services`.

### Built-in Services

```php
$session    = \Config\Services::session();
$validation = \Config\Services::validation();
$uri        = \Config\Services::uri();
$request    = \Config\Services::request();
$response   = \Config\Services::response();
$cache      = \Config\Services::cache();
$logger     = \Config\Services::logger();
$email      = \Config\Services::email();
$throttler  = \Config\Services::throttler();
$encrypter  = \Config\Services::encrypter();
$curlrequest = \Config\Services::curlrequest();

// Shorthand
service('session');
service('validation');
service('cache');
```

### Custom Service

```php
// Register in app/Config/Services.php
public static function stripe(bool $getShared = true): \App\Services\StripeService
{
    if ($getShared) return static::getSharedInstance('stripe');
    return new \App\Services\StripeService();
}

// Usage
$stripe = \Config\Services::stripe();
$stripe = service('stripe');
```

### Shared vs Non-Shared

```php
// Shared (singleton — same instance every time, default)
$db = \Config\Services::database();

// Non-shared (new instance)
$db = \Config\Services::database(false);
```

---

## Helpers

### Loading Helpers

```php
// Load single
helper('url');

// Load multiple
helper(['url', 'form', 'text']);

// Auto-load in BaseController
class BaseController extends Controller
{
    protected $helpers = ['url', 'form'];
}

// Auto-load globally in app/Config/Autoload.php
public $helpers = ['url', 'form'];
```

### URL Helper

```php
helper('url');

base_url('path/to/resource');           // full URL from web root
site_url('users/1');                    // full URL with index.php (if configured)
url_to('ControllerName::method', $arg); // URL from controller method
url_to('named-route', $arg);           // URL from named route
current_url();                          // current full URL
previous_url();                         // previous URL (from session)
uri_string();                           // current URI path only (no domain)
anchor('users', 'View Users');          // <a href="...">View Users</a>
```

### Form Helper

```php
helper('form');

form_open('users/create');                    // <form action="..." method="post">
form_open('users/create', ['class' => 'form-inline']);
form_open_multipart('users/create');          // with enctype for file uploads
form_close();                                 // </form>
csrf_field();                                 // CSRF hidden input
form_input('name', old('name'), ['class' => 'form-control']);
form_password('password');
form_textarea('bio', old('bio'), ['rows' => 5]);
form_dropdown('role', ['admin' => 'Admin', 'user' => 'User'], old('role'));
form_checkbox('active', '1', old('active') == '1');
form_radio('gender', 'male', old('gender') === 'male');
form_submit('submit', 'Save');
form_hidden('user_id', $id);
set_value('name');                            // same as old('name')
```

### Text Helper

```php
helper('text');

word_limiter($text, 25);               // limit by words
character_limiter($text, 100);         // limit by characters
ellipsize($text, 30);                  // truncate with ellipsis in middle
ascii_to_entities($text);              // convert to HTML entities
```

### Filesystem Helper

```php
helper('filesystem');

write_file('./path/to/file.txt', $data);
$contents = read_file('./path/to/file.txt');   // deprecated — use file_get_contents
delete_files('./path/to/dir/', true);          // true = delete dir too
$files = get_filenames('./path/');
$info  = get_file_info('./path/to/file.txt');
$size  = get_dir_file_info('./path/');
```

### Date Helper

```php
helper('date');

now();                                  // current timestamp
timezone_select();                      // timezone dropdown HTML
```

### Number Helper

```php
helper('number');

number_to_size(1024);                   // "1 KB"
number_to_amount(1234567);             // "1.23 million"
number_to_currency(1234.56, 'USD');    // "$1,234.56"
number_to_roman(14);                   // "XIV"
```

### Custom Helpers

```php
// app/Helpers/my_helper.php
<?php

if (!function_exists('format_phone')) {
    function format_phone(string $phone): string
    {
        return preg_replace('/(\d{3})(\d{3})(\d{4})/', '($1) $2-$3', $phone);
    }
}

// Usage
helper('my');  // loads my_helper.php
echo format_phone('5551234567');
```

---

## Caching

### Basic Usage

```php
$cache = \Config\Services::cache();

$cache->save('key', $data, 300);        // save for 300 seconds (5 min)
$data = $cache->get('key');             // retrieve (null if missing/expired)
$cache->delete('key');                  // delete single
$cache->clean();                        // clear all cache

// Check if key exists
if ($cache->get('key') !== null) { ... }
```

### Remember Pattern

```php
$data = cache()->remember('expensive_query', 300, function () {
    return model('UserModel')->where('active', 1)->findAll();
});
```

### Cache Drivers

Configured in `app/Config/Cache.php`:

```php
public string $handler = 'file';  // file, redis, memcached, predis, wincache

// Redis config
public array $redis = [
    'host'     => '127.0.0.1',
    'password' => null,
    'port'     => 6379,
    'timeout'  => 0,
    'database' => 0,
];
```

### Tagging (Redis/Memcached only)

```php
$cache->save('user_1', $data, 300, ['users']);
$cache->save('user_2', $data, 300, ['users']);
$cache->deleteMatching('users');  // delete all tagged 'users'
```

---

## Email

### Configuration

`app/Config/Email.php` or `.env`:
```
email.fromEmail = noreply@example.com
email.fromName = My App
email.protocol = smtp
email.SMTPHost = smtp.example.com
email.SMTPUser = user
email.SMTPPass = pass
email.SMTPPort = 587
email.SMTPCrypto = tls
email.mailType = html
```

### Sending Email

```php
$email = \Config\Services::email();

$email->setFrom('noreply@example.com', 'My App');
$email->setTo('user@example.com');
$email->setCC('cc@example.com');
$email->setBCC('bcc@example.com');
$email->setSubject('Welcome!');
$email->setMessage(view('emails/welcome', ['name' => $name]));

if (!$email->send()) {
    log_message('error', $email->printDebugger(['headers']));
}

// Reset for next send
$email->clear();
```

### Attachments

```php
$email->attach('/path/to/file.pdf');
$email->attach('/path/to/image.png', 'inline');  // inline image
```

---

## Sessions

### Configuration

`app/Config/Session.php` or `.env`:
```
session.driver = CodeIgniter\Session\Handlers\DatabaseHandler
session.cookieName = ci_session
session.expiration = 7200
session.savePath = ci_sessions   # table name for DB driver
session.matchIP = false
session.timeToUpdate = 300
session.regenerateDestroy = false
```

### Database Session Table

```bash
php spark session:migration    # generates migration for session table
php spark migrate
```

### Usage

```php
$session = session();  // or \Config\Services::session()

// Set
$session->set('key', 'value');
$session->set(['key1' => 'val1', 'key2' => 'val2']);

// Get
$value = $session->get('key');
$value = session('key');           // shorthand
$all   = $session->get();          // all session data

// Check
$session->has('key');              // bool

// Remove
$session->remove('key');
$session->remove(['key1', 'key2']);

// Flash data (available only on next request)
$session->setFlashdata('message', 'Success!');
$session->getFlashdata('message');
$session->keepFlashdata('message');  // keep for one more request

// Temp data (auto-expires)
$session->setTempdata('token', $value, 300);  // expires in 300 seconds
$session->getTempdata('token');

// Destroy
$session->destroy();

// Regenerate ID (do this after login)
$session->regenerate();
```

---

## Encryption

```php
// Generate key (run once, add to .env)
// php spark key:generate

$encrypter = \Config\Services::encrypter();

// Encrypt
$encrypted = $encrypter->encrypt('sensitive data');

// Decrypt
$plaintext = $encrypter->decrypt($encrypted);

// Encrypt for URL/cookie (base64-safe)
$encrypted = base64_encode($encrypter->encrypt('data'));
$plaintext = $encrypter->decrypt(base64_decode($encrypted));
```

---

## HTTP Client (CURLRequest)

```php
$client = \Config\Services::curlrequest();

// GET
$response = $client->get('https://api.example.com/users');
$body     = $response->getBody();
$status   = $response->getStatusCode();
$json     = json_decode($body, true);

// POST with JSON
$response = $client->post('https://api.example.com/users', [
    'headers' => ['Content-Type' => 'application/json'],
    'json'    => ['name' => 'Rob', 'email' => 'r@b.com'],
]);

// POST with form data
$response = $client->post('https://api.example.com/login', [
    'form_params' => ['email' => 'r@b.com', 'password' => 'secret'],
]);

// With auth header
$response = $client->get('https://api.example.com/me', [
    'headers' => ['Authorization' => 'Bearer ' . $token],
]);

// Options
$response = $client->get($url, [
    'timeout'     => 10,        // seconds
    'verify'      => false,     // skip SSL verification (dev only)
    'allow_redirects' => true,
]);
```

---

## Events

```php
// app/Config/Events.php
use CodeIgniter\Events\Events;

// Register a listener
Events::on('user_created', static function ($user) {
    log_message('info', "User created: {$user->email}");
});

// Trigger an event (from anywhere)
Events::trigger('user_created', $user);

// Multiple listeners — they run in registration order
Events::on('order_placed', static function ($order) { /* send email */ });
Events::on('order_placed', static function ($order) { /* update inventory */ });
```

---

## Logging

```php
log_message('debug', 'Debug message');
log_message('info', 'Informational message');
log_message('notice', 'Notice');
log_message('warning', 'Warning message');
log_message('error', 'Error message');
log_message('critical', 'Critical error');
log_message('alert', 'Alert');
log_message('emergency', 'System is unusable');

// With context
log_message('error', 'User {id} failed login', ['id' => $userId]);
```

Logs written to `writable/logs/`. Configure threshold in `app/Config/Logger.php`.
