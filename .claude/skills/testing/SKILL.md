---
name: testing
description: Pest PHP test setup and patterns for this PHP REST API project. Use this skill when the user says "write tests for", "add test coverage", "create a test file", "how do I test this endpoint", "add a UserControllerTest", "how does the api() helper work", "set up Pest PHP", or needs help with the beforeAll, test server, or 5-case coverage checklist.
---

## When to use this skill

- "Write tests for [resource]"
- "Add test coverage"
- "Create a test file"
- "How do I test this endpoint?"
- "Set up Pest PHP"
- "How does the api() helper work?"
- "What test cases should I write?"

## Dependencies / Prerequisites

- setup skill: project bootstrapped, `composer.json` has `pestphp/pest` in `require-dev`
- PHP dev server must be running before tests execute
- `POST /config/install` must work (ConfigController exists)

---

## Steps

### 1. Install Pest and generate bootstrap files

```bash
composer require --dev pestphp/pest
./vendor/bin/pest --init
```

This creates `tests/Pest.php` and `tests/TestCase.php`.

### 2. Replace tests/Pest.php with the project bootstrap

```php
<?php
pest()->extend(Tests\TestCase::class)->in('Feature');

define('API_BASE_URL', 'http://127.0.0.1:8080');

/**
 * @param string $method  HTTP verb: GET, POST, PUT, DELETE
 * @param string $path    e.g. '/posts'  or  '/posts/5'
 * @param array  $body    JSON body for POST / PUT / DELETE
 */
function api(string $method, string $path, array $body = []): array
{
    $ch   = curl_init(API_BASE_URL . $path);
    $opts = [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
        CURLOPT_TIMEOUT        => 10,
        CURLOPT_CUSTOMREQUEST  => $method,
    ];
    if (!empty($body)) {
        $opts[CURLOPT_POSTFIELDS] = json_encode($body);
    }
    curl_setopt_array($ch, $opts);
    return json_decode(curl_exec($ch), true);
}
```

### 3. Start the dev server (Terminal 1)

```bash
php -S 0.0.0.0:8080 index.php
```

Leave this running. It must be active before tests run.

### 4. Create tests/Feature/[ResourceName]ControllerTest.php

Use the template below and fill in `[ResourceName]`, `[resource_name]`, `TEST_USER`, `TEST_PASS`.

### 5. Run tests

```bash
# All tests
./vendor/bin/pest

# Single file
./vendor/bin/pest tests/Feature/[ResourceName]ControllerTest.php

# With output (verbose)
./vendor/bin/pest --verbose
```

---

## Boilerplate — [ResourceName]ControllerTest.php

```php
<?php

define('[RESOURCE_NAME]_USER',  'testuser');
define('[RESOURCE_NAME]_PASS',  'password123');
define('[RESOURCE_NAME]_OTHER', 'otheruser');
define('[RESOURCE_NAME]_OPASS', 'password456');

beforeAll(function () {
    // Always reset the DB before each test file runs
    api('POST', '/config/install');
    api('POST', '/users', ['id' => [RESOURCE_NAME]_USER,  'password' => [RESOURCE_NAME]_PASS]);
    api('POST', '/users', ['id' => [RESOURCE_NAME]_OTHER, 'password' => [RESOURCE_NAME]_OPASS]);
});

// ── Happy path ────────────────────────────────────────────────────────────

test('POST /[resource_name]s — create success', function () {
    $res = api('POST', '/[resource_name]s', [
        'title'    => 'Test Title',
        'content'  => 'Test Content',
        'id'       => [RESOURCE_NAME]_USER,
        'password' => [RESOURCE_NAME]_PASS,
    ]);

    expect($res['status'])->toBe('success')
        ->and($res['no'])->toBeGreaterThan(0);
});

test('GET /[resource_name]s — list returns array', function () {
    $res = api('GET', '/[resource_name]s');

    expect($res['status'])->toBe('success')
        ->and($res['[resource_name]s'])->toBeArray();
});

test('GET /[resource_name]s/{no} — show', function () {
    $created = api('POST', '/[resource_name]s', [
        'title' => 'Show Test', 'content' => 'Body',
        'id' => [RESOURCE_NAME]_USER, 'password' => [RESOURCE_NAME]_PASS,
    ]);
    $res = api('GET', '/[resource_name]s/' . $created['no']);

    expect($res['status'])->toBe('success')
        ->and($res['[resource_name]']['title'])->toBe('Show Test');
});

test('PUT /[resource_name]s/{no} — update success', function () {
    $created = api('POST', '/[resource_name]s', [
        'title' => 'Old', 'content' => 'Old',
        'id' => [RESOURCE_NAME]_USER, 'password' => [RESOURCE_NAME]_PASS,
    ]);
    $res = api('PUT', '/[resource_name]s/' . $created['no'], [
        'title' => 'New',
        'id' => [RESOURCE_NAME]_USER, 'password' => [RESOURCE_NAME]_PASS,
    ]);

    expect($res['status'])->toBe('success');

    $check = api('GET', '/[resource_name]s/' . $created['no']);
    expect($check['[resource_name]']['title'])->toBe('New');
});

test('DELETE /[resource_name]s/{no} — delete success', function () {
    $created = api('POST', '/[resource_name]s', [
        'title' => 'Delete Me', 'content' => 'Body',
        'id' => [RESOURCE_NAME]_USER, 'password' => [RESOURCE_NAME]_PASS,
    ]);
    api('DELETE', '/[resource_name]s/' . $created['no'], [
        'id' => [RESOURCE_NAME]_USER, 'password' => [RESOURCE_NAME]_PASS,
    ]);

    $check = api('GET', '/[resource_name]s/' . $created['no']);
    expect($check['status'])->toBe('error');
});

// ── Unauthenticated ───────────────────────────────────────────────────────

test('POST /[resource_name]s — unauthenticated returns error', function () {
    $res = api('POST', '/[resource_name]s', ['title' => 'T', 'content' => 'C']);

    expect($res['status'])->toBe('error')
        ->and($res['message'])->toContain('Login required');
});

// ── Missing required fields ───────────────────────────────────────────────

test('POST /[resource_name]s — missing title returns error', function () {
    $res = api('POST', '/[resource_name]s', [
        'content' => 'No title here',
        'id' => [RESOURCE_NAME]_USER, 'password' => [RESOURCE_NAME]_PASS,
    ]);

    expect($res['status'])->toBe('error');
});

// ── Not found ─────────────────────────────────────────────────────────────

test('GET /[resource_name]s/{no} — not found returns error', function () {
    $res = api('GET', '/[resource_name]s/99999');

    expect($res['status'])->toBe('error')
        ->and($res['message'])->toContain('not found');
});

// ── Permission denied ─────────────────────────────────────────────────────

test('PUT /[resource_name]s/{no} — permission denied for other user', function () {
    $created = api('POST', '/[resource_name]s', [
        'title' => 'Mine', 'content' => 'Body',
        'id' => [RESOURCE_NAME]_USER, 'password' => [RESOURCE_NAME]_PASS,
    ]);
    $res = api('PUT', '/[resource_name]s/' . $created['no'], [
        'title' => 'Hacked',
        'id' => [RESOURCE_NAME]_OTHER, 'password' => [RESOURCE_NAME]_OPASS,
    ]);

    expect($res['status'])->toBe('error')
        ->and($res['message'])->toContain('Permission denied');
});
```

---

## 5-Case Coverage Checklist

For every endpoint write tests covering:

| # | Case | What to assert |
|---|---|---|
| 1 | Happy path | `status: success`, correct data returned |
| 2 | Unauthenticated | `status: error`, message contains `Login required` |
| 3 | Missing required fields | `status: error` |
| 4 | Not found (`no = 99999`) | `status: error`, message contains `not found` |
| 5 | Permission denied (other user's resource) | `status: error`, message contains `Permission denied` |

---

## Gotchas

- **Always call `api('POST', '/config/install')` in `beforeAll()`** — without this, data from a previous test run accumulates, causing `'ID already exists'` failures and flaky tests.

- **The dev server must be running before `./vendor/bin/pest`** — if the server is down, all cURL calls silently return `null` and `json_decode` returns `null`, causing confusing failures.

- **Pass credentials in the body for DELETE requests too** — `DELETE /posts/5` requires `id` and `password` in the JSON body. The cURL `api()` helper supports a third `$body` argument for this.

- **Assert password hash is never exposed** — for any endpoint that returns a user object:
  ```php
  expect($res['user'])->not->toHaveKey('password');
  ```

- **Define test user constants at the top of each file** — e.g. `define('POST_USER', 'postuser')`. Avoids magic strings scattered across tests.

- **Do not share state between test files via the database** — each test file must call `api('POST', '/config/install')` in its own `beforeAll()` to start fresh.

- **Run a single file for faster iteration during development**:
  ```bash
  ./vendor/bin/pest tests/Feature/PostControllerTest.php
  ```
