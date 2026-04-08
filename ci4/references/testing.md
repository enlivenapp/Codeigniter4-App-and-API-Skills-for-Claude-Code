# CI4 Testing — Complete Reference

CI4 uses PHPUnit. Test files go in `tests/`.

## Running Tests

```bash
composer test
# or
php vendor/bin/phpunit
php vendor/bin/phpunit tests/unit/UserModelTest.php
php vendor/bin/phpunit --filter testFindUser
php vendor/bin/phpunit --group database
```

## Unit Test

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

    public function testInsertUser(): void
    {
        $model = new UserModel();
        $id = $model->insert([
            'name'  => 'Test User',
            'email' => 'test@example.com',
        ]);
        $this->assertIsInt($id);
        $this->assertGreaterThan(0, $id);
    }
}
```

## Feature / HTTP Test

```php
<?php
namespace Tests\Feature;

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

    public function testShow(): void
    {
        $result = $this->get('users/1');
        $result->assertStatus(200);
        $result->assertSee('Rob');
    }

    public function testCreate(): void
    {
        $result = $this->post('users', [
            'name'  => 'New User',
            'email' => 'new@example.com',
        ]);
        $result->assertRedirectTo(base_url('users'));
    }

    public function testCreateValidationFails(): void
    {
        $result = $this->post('users', [
            'name' => '',  // required field empty
        ]);
        $result->assertStatus(200);  // re-displays form
        $result->assertSee('name field is required');
    }

    public function testApiEndpoint(): void
    {
        $result = $this->withHeaders([
            'Authorization' => 'Bearer ' . $this->token,
            'Content-Type'  => 'application/json',
        ])->call('post', 'api/v1/users', json_encode([
            'name'  => 'API User',
            'email' => 'api@example.com',
        ]));

        $result->assertStatus(201);
        $result->assertJSONFragment(['name' => 'API User']);
    }
}
```

### FeatureTestTrait Methods

```php
// HTTP methods
$result = $this->get($uri);
$result = $this->post($uri, $data);
$result = $this->put($uri, $data);
$result = $this->patch($uri, $data);
$result = $this->delete($uri);
$result = $this->options($uri);

// Generic call
$result = $this->call('get', $uri);
$result = $this->call('post', $uri, $data);

// With headers
$result = $this->withHeaders(['X-Custom' => 'value'])->get($uri);

// With session data
$result = $this->withSession(['user_id' => 1])->get('dashboard');

// With body (for JSON APIs)
$result = $this->withBody(json_encode($data))->call('post', $uri);
```

### Test Result Assertions

```php
// Status
$result->assertStatus(200);
$result->assertOK();                    // 200
$result->assertRedirect();              // 3xx
$result->assertRedirectTo($url);

// Content
$result->assertSee('text');             // text appears in body
$result->assertDontSee('text');
$result->assertSeeElement('h1');        // CSS selector
$result->assertDontSeeElement('.error');
$result->assertSeeLink('Click Me');     // <a> with text
$result->assertSeeInField('name', 'Rob');

// JSON
$result->assertJSONFragment(['key' => 'value']);
$result->assertJSONExact($expected);

// Headers
$result->assertHeader('Content-Type', 'application/json');
$result->assertHeaderMissing('X-Custom');

// Cookies
$result->assertCookie('session');
$result->assertCookieMissing('old_cookie');
$result->assertCookieExpired('temp');

// Session
$result->assertSessionHas('key');
$result->assertSessionHas('key', 'value');
$result->assertSessionMissing('key');
```

## Database Test Trait

```php
use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\DatabaseTestTrait;

class UserModelTest extends CIUnitTestCase
{
    use DatabaseTestTrait;

    // Rolls back DB changes after each test
    protected $refresh = true;

    // Run this seeder before each test
    protected $seed = 'UserSeeder';

    // Use a specific DB group
    protected $DBGroup = 'tests';

    public function testUserCount(): void
    {
        $this->assertCount(5, model('UserModel')->findAll());
    }
}
```

### Database Assertions

```php
$this->seeInDatabase('users', ['email' => 'rob@example.com']);
$this->dontSeeInDatabase('users', ['email' => 'deleted@example.com']);
$this->seeNumRecords(5, 'users', ['active' => 1]);
$this->grabFromDatabase('users', 'name', ['id' => 1]);  // returns column value
```

## Testing with Shield Authentication

```php
use CodeIgniter\Test\FeatureTestTrait;
use CodeIgniter\Test\DatabaseTestTrait;

class AdminTest extends CIUnitTestCase
{
    use FeatureTestTrait;
    use DatabaseTestTrait;

    protected $refresh = true;

    public function testAdminPageRequiresAuth(): void
    {
        $result = $this->get('admin');
        $result->assertRedirectTo(base_url('login'));
    }

    public function testAdminPageAccessible(): void
    {
        $user = $this->createUser();
        $user->addGroup('admin');

        $result = $this->actingAs($user)->get('admin');
        $result->assertStatus(200);
    }

    private function createUser(): \CodeIgniter\Shield\Entities\User
    {
        $users = auth()->getProvider();
        $user  = new \CodeIgniter\Shield\Entities\User([
            'username' => 'testuser',
            'email'    => 'test@example.com',
            'password' => 'TestPass123!',
        ]);
        $users->save($user);
        return $users->findById($users->getInsertID());
    }
}
```

### actingAs() — Authenticate for a test

```php
$this->actingAs($user)->get('admin/dashboard');
$this->actingAs($user)->post('api/posts', $data);
```

## Mocking Services

```php
// Mock a service
$mock = $this->createMock(\App\Services\PaymentService::class);
$mock->method('charge')->willReturn(true);

// Inject into Services
\Config\Services::injectMock('payment', $mock);

// Mock the cache
$mockCache = $this->createMock(\CodeIgniter\Cache\CacheInterface::class);
$mockCache->method('get')->willReturn(null);
\Config\Services::injectMock('cache', $mockCache);

// Reset mocks after test
\Config\Services::reset();
```

## Test Configuration

### phpunit.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit bootstrap="vendor/codeigniter4/framework/system/Test/bootstrap.php"
         colors="true">
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory>tests/feature</directory>
        </testsuite>
    </testsuites>
    <php>
        <server name="app.baseURL" value="http://localhost:8080/"/>
        <const name="HOMEPATH" value="./"/>
        <const name="CONFIGPATH" value="./app/Config/"/>
        <const name="PUBLICPATH" value="./public/"/>
    </php>
</phpunit>
```

### Test .env

Create `phpunit.xml` or `.env.testing` with test database config:
```
database.tests.hostname = localhost
database.tests.database = test_db
database.tests.username = root
database.tests.password =
database.tests.DBDriver = MySQLi
```
