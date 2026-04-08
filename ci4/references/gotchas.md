# CI4 Common Pitfalls & Gotchas

## Query Builder

1. **`whereNull()` does not exist in CI4.** Use `->where('col IS NULL')`. This is a Laravel method — it will not error, but it won't work correctly.

2. **`->select('DISTINCT col')` returns wrong results silently.** Use `->select('col')->distinct()` instead.

3. **`orLike()` after a `where()` produces wrong SQL** without grouping. Wrap with `groupStart()`/`groupEnd()`:
   ```php
   $builder->where('active', 1)
       ->groupStart()
           ->like('name', 'rob')
           ->orLike('email', 'rob')
       ->groupEnd();
   ```

4. **`->select()` is cumulative** — calling it multiple times appends columns, it does not replace them.

5. **`insertBatch()` bypasses model `$allowedFields`** — it inserts everything you give it. Be deliberate about what columns you pass.

## Models

6. **Always use `protected $returnType = 'object'`** — `'array'` means `$row['key']` instead of `$row->key`, which is inconsistent and error-prone across the codebase.

7. **`getJSON(true)` returns `array`** — use `$body['key']`, not `$body->key`. This is a common source of "Trying to get property of non-object" errors.

8. **Model `save()` decides insert vs update** based on whether the primary key is present in data — there is no magic `upsert()`.

9. **Soft delete `delete()` only sets `deleted_at`** — queries automatically exclude soft-deleted rows. Use `withDeleted()` to include them.

10. **`{id}` in validation rules** (e.g., `is_unique[users.email,id,{id}]`) only works when the model's primary key is in the data being validated. It is NOT replaced automatically from URL segments.

## Controllers

11. **Never type-hint overridden ResourceController params**: `show($id = null)` not `show(int $id = null)`. The parent signature uses `$id = null` — a type hint breaks the override.

12. **ResourceController has a `protected format()` method** — never define a `private function format()` in a subclass. Access level conflict causes a fatal error.

13. **`redirect()` must be `return`ed** — `redirect()->to('/foo')` without `return` does nothing. The redirect creates a response object; without `return`, it's discarded.

14. **`$this->request->getJSON(true)` can return `null`** if the request body isn't valid JSON. Always handle that:
    ```php
    $body = $this->request->getJSON(true);
    $body = is_array($body) ? $body : [];
    ```

## Views

15. **Never use `return` or early exit inside a view** that uses `$this->extend()`. Layout rendering requires all sections to complete. Use `if/else` for conditional content.

16. **View sections cannot be nested.** Always call `$this->endSection()` before opening another section. This fails silently — no error, just missing output.

17. **`$this->include()` passes parent variables.** But `view()` called inside a view does NOT pass parent variables — you must pass data explicitly: `view('partial', ['var' => $var])`.

## Database / Migrations

18. **Backtick-quote reserved words in raw SQL**: `` `key` ``, `` `order` ``, `` `index` ``, `` `group` ``. Without quotes, MySQL treats them as keywords.

19. **`app_settings` tables often use `key` as a column** — always quote it in raw SQL.

20. **Migration timestamps must be unique** — if two migrations share a timestamp, one may be skipped silently.

21. **Foreign keys must be dropped in `down()`** before dropping the parent table.

## Configuration

22. **`.env` values need no quotes** for simple strings, but URLs with special chars should be quoted: `app.baseURL = 'http://example.com/'`

23. **`CI_ENVIRONMENT`** controls error display. In `production`, errors are hidden and logged to `writable/logs/`. In `development`, they show on screen.

## Sessions

24. **Session regeneration** — call `session()->regenerate()` after login to prevent session fixation attacks.

25. **Flash data is only available on the NEXT request** — you can't set flash data and read it in the same request. Use `keepFlashdata()` to persist it one more request.

---

## CI4 vs Laravel — Do Not Confuse

| What | Laravel | CI4 |
|---|---|---|
| Find by ID | `User::find(1)` | `$model->find(1)` |
| Find with where | `User::where()->get()` | `$model->where()->findAll()` |
| Where NULL | `whereNull('col')` | `->where('col IS NULL')` |
| Or Where | `orWhere()` | `->orWhere()` (same) |
| Request input | `$request->input('key')` | `$this->request->getPost('key')` or `getVar()` |
| Validate | `$request->validate([])` | `$this->validate([])` |
| Debug dump | `dd()` | `d()` or `var_dump(); exit;` |
| CLI | `artisan` | `spark` |
| Writable dir | `storage/` | `writable/` |
| Query scopes | Eloquent scopes | Method chaining returning `static` |
| Middleware | Middleware | Filters |
| Service providers | Service Providers | `app/Config/Services.php` |
| Facades | Facades | `\Config\Services::serviceName()` |
| Auth check | `Auth::check()` | `auth()->loggedIn()` |
| Current user | `Auth::user()` | `auth()->user()` |
| Auth attempt | `Auth::attempt()` (returns bool) | `auth()->attempt()` (returns Result object) |
| Roles | `$user->hasRole('admin')` | `$user->inGroup('admin')` |
| Permissions | `$user->can('edit posts')` | `$user->can('posts.edit')` |
| Blade `@auth` | `@auth ... @endauth` | `<?php if (auth()->loggedIn()): ?>` |
| Migrations up | `Schema::create()` | `$this->forge->createTable()` |
| Auth routes | `Auth::routes()` | Shield auto-registers routes |
| Token auth | Sanctum | `$user->generateAccessToken()` |
| Guards | Guards | Authenticators |
| Middleware groups | Middleware groups | Filter aliases + route groups |
| `Route::middleware()` | `Route::middleware('auth')` | `['filter' => 'session']` |
| Eager loading | `with('relation')` | CI4 has no built-in eager loading |
| Blade directives | `@foreach`, `@if` | `<?php foreach (): ?>`, `<?php if (): ?>` |
| `Request::has()` | `$request->has('key')` | `$this->request->getVar('key') !== null` |
