# CI4 Controllers — Complete Reference

## Base Structure

All controllers extend `BaseController` (`app/Controllers/BaseController.php`).

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

### BaseController Provides
- `$this->request` — IncomingRequest
- `$this->response` — ResponseInterface
- `$this->logger` — Logger
- `$this->helpers` — array of helpers to auto-load

```php
class BaseController extends Controller
{
    protected $helpers = ['url', 'form'];  // auto-loaded helpers

    public function initController(...)
    {
        parent::initController(...);
        // Custom initialization here
    }
}
```

## ResourceController

For RESTful resources. Implements `index`, `show`, `new`, `edit`, `create`, `update`, `delete`.

```php
<?php
namespace App\Controllers;

use CodeIgniter\RESTful\ResourceController;

class PhotoController extends ResourceController
{
    protected $modelName = 'App\Models\PhotoModel';
    protected $format    = 'json';  // 'json' or 'xml'

    public function index()
    {
        return $this->respond($this->model->findAll());
    }

    public function show($id = null)  // NEVER type-hint override params
    {
        $photo = $this->model->find($id);
        if (!$photo) return $this->failNotFound('Photo not found');
        return $this->respond($photo);
    }

    public function create()
    {
        $data = $this->request->getJSON(true);
        $id   = $this->model->insert($data);
        return $this->respondCreated($this->model->find($id));
    }

    public function update($id = null)
    {
        $data = $this->request->getJSON(true);
        $this->model->update($id, $data);
        return $this->respond($this->model->find($id));
    }

    public function delete($id = null)
    {
        $this->model->delete($id);
        return $this->respondDeleted(['id' => $id]);
    }
}
```

**GOTCHA**: Never add PHP type hints (`int $id`) to overridden ResourceController methods. The parent signature uses `$id = null` — a type hint breaks the override.

**GOTCHA**: `ResourceController` has a `protected format()` method. Never define a private method named `format()` in a subclass — access level conflict / fatal error.

### ResourceController Response Methods

```php
$this->respond($data, 200);          // Generic response
$this->respondCreated($data);        // 201
$this->respondDeleted($data);        // 200 with deleted confirmation
$this->respondNoContent();           // 204
$this->fail($message, 400);          // Generic failure
$this->failNotFound($message);       // 404
$this->failValidationErrors($errors); // 422
$this->failForbidden($message);      // 403
$this->failUnauthorized($message);   // 401
$this->failServerError($message);    // 500
$this->failTooManyRequests($message); // 429
```

## Request Data

```php
// GET parameters
$this->request->getGet('name');       // single
$this->request->getGet();            // all GET params

// POST parameters
$this->request->getPost('email');     // single
$this->request->getPost();           // all POST params

// JSON body (API endpoints)
$body = $this->request->getJSON(true);  // true = associative array
// OR
$body = json_decode($this->request->getBody(), true);

// All input (GET + POST)
$this->request->getVar('key');

// Specific HTTP method data
$this->request->getRawInput();        // PUT/PATCH/DELETE body

// Request method
$this->request->getMethod();          // 'get', 'post', 'put', etc.

// Check if AJAX
$this->request->isAJAX();

// IP address
$this->request->getIPAddress();

// Headers
$this->request->getHeaderLine('Authorization');
$this->request->header('Content-Type');
```

## File Uploads

```php
$file = $this->request->getFile('avatar');

// Check
$file->isValid();                     // was it uploaded without errors?
$file->hasMoved();                    // has it already been moved?

// Info
$file->getName();                     // original filename
$file->getClientExtension();          // extension from client
$file->getClientMimeType();           // MIME from client
$file->getTempName();                 // tmp path
$file->getSize();                     // size in bytes (string)
$file->getSizeByUnit('mb');           // size in specified unit

// Move
$file->move(WRITEPATH . 'uploads');                    // move to directory
$file->move(WRITEPATH . 'uploads', 'custom_name.jpg'); // with custom name

// Store (convenience — moves with random name)
$path = $file->store();               // returns relative path
$path = $file->store('avatars');      // store in subdirectory

// Multiple files
$files = $this->request->getFiles();
$files = $this->request->getFileMultiple('images');
```

### File Upload Validation

```php
$rules = [
    'avatar' => [
        'label' => 'Avatar',
        'rules' => [
            'uploaded[avatar]',
            'is_image[avatar]',
            'mime_in[avatar,image/jpg,image/jpeg,image/png,image/webp]',
            'max_size[avatar,2048]',      // KB
            'max_dims[avatar,1024,768]',  // width, height
        ],
    ],
];

if (!$this->validateData([], $rules)) {
    return redirect()->back()->with('errors', $this->validator->getErrors());
}
```

## Validation in Controllers

```php
$rules = [
    'email' => 'required|valid_email',
    'name'  => 'required|min_length[2]|max_length[100]',
];

if (!$this->validate($rules)) {
    return redirect()->back()->withInput()->with('errors', $this->validator->getErrors());
}

// With custom messages
$messages = [
    'email' => [
        'required'    => 'Email address is required.',
        'valid_email' => 'Please provide a valid email.',
    ],
];
if (!$this->validate($rules, $messages)) { ... }

// validateData() — validate arbitrary data (not just request data)
if (!$this->validateData($data, $rules)) { ... }
```

## Redirects

```php
return redirect()->to('/dashboard');
return redirect()->back();
return redirect()->route('profile');          // named route
return redirect()->back()->withInput();       // preserve old input
return redirect()->back()->with('message', 'Saved!');  // flash data
return redirect()->back()->with('error', 'Something went wrong.');
return redirect()->to(url_to('UserController::show', $id));
```

**GOTCHA**: `redirect()` must be `return`ed — `redirect()->to('/foo')` without `return` does nothing.

## Session Flash Data

```php
// Set
session()->setFlashdata('message', 'User created.');
session()->setFlashdata('error', 'Something went wrong.');
session()->setFlashdata('errors', $this->validator->getErrors());

// Get (in controller or view)
session()->getFlashdata('message');
session()->getFlashdata('error');

// Shorthand via redirect
return redirect()->back()->with('message', 'Saved!');
// In view: session('message') or session()->getFlashdata('message')
```

## Returning Responses

```php
// View
return view('users/index', $data);

// JSON
return $this->response->setJSON($data);

// With status code
return $this->response->setStatusCode(201)->setJSON($data);

// With headers
return $this->response->setHeader('X-Custom', 'value')->setJSON($data);

// Download
return $this->response->download('filename.pdf', $fileData);

// No content
return $this->response->setStatusCode(204);
```
