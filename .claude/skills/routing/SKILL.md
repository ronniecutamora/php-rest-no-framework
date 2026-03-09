---
name: routing
description: How index.php parses HTTP requests and resolves controller + action in this PHP REST API. Use this skill when the user asks "how does routing work", "add a named route", "add a custom endpoint like login", "how does /users/login get resolved", "add a non-standard action", "why does my route return unknown resource", or anything related to the front controller logic in index.php.
---

## When to use this skill

- "How does routing work?"
- "Add a named route (e.g. /users/login, /posts/publish)"
- "Add a custom endpoint"
- "Why does my route return 'Unknown resource'?"
- "How does /config/install get resolved?"
- "Where does $input['no'] come from?"
- "How do I add a non-CRUD action to a controller?"

## Dependencies / Prerequisites

- setup skill completed (index.php exists)
- contracts skill completed (Controller abstract class exists)

---

## How the Router Works

```
GET /posts/5
 │
 ├─ $httpMethod = 'GET'
 ├─ $resource   = 'posts'         ← segments[0]
 ├─ $no         = '5'             ← segments[1] (string from URL)
 │
 ├─ $singular        = rtrim('posts', 's')  = 'post'
 ├─ $controllerClass = 'Lib\Post\PostController'
 │
 ├─ $action = 'show'              ← GET + $no is not null
 │
 ├─ $input['no'] = (int)'5' = 5  ← injected by router
 │
 └─ PostController::show($input)
```

---

## Steps — Complete index.php Routing Logic

```php
<?php
require_once __DIR__ . '/vendor/autoload.php';

header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
header('Access-Control-Allow-Headers: Content-Type');

if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
    exit(0);
}

header('Content-Type: application/json');

try {
    $httpMethod = $_SERVER['REQUEST_METHOD'];

    // 1. Parse URL into segments
    $uri      = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
    $uri      = trim($uri, '/');
    $segments = explode('/', $uri);

    $resource = $segments[0] ?? '';   // 'posts', 'users', 'config'
    $no       = $segments[1] ?? null; // '5', 'login', 'install', or null

    // 2. Read JSON body; fall back to form data
    $body  = file_get_contents('php://input');
    $input = json_decode($body, true) ?? $_REQUEST ?? [];

    // 3. Inject numeric {no} from URL into $input
    if ($no !== null && is_numeric($no)) {
        $input['no'] = (int)$no;
    }

    // 4. Resolve controller — strip plural 's', then build class name
    $singular        = rtrim($resource, 's');
    $controllerClass = 'Lib\\' . ucfirst($singular) . '\\' . ucfirst($singular) . 'Controller';

    if (!class_exists($controllerClass)) {
        throw new \InvalidArgumentException("Unknown resource: {$resource}");
    }

    // 5. Resolve default action from HTTP method + presence of $no
    $action = match(true) {
        $httpMethod === 'GET'                  && $no === null => 'list',
        $httpMethod === 'GET'                  && $no !== null => 'show',
        $httpMethod === 'POST'                                 => 'create',
        in_array($httpMethod, ['PUT', 'PATCH'])                => 'update',
        $httpMethod === 'DELETE'                               => 'delete',
        default => throw new \InvalidArgumentException("Unsupported method: {$httpMethod}"),
    };

    $controller = new $controllerClass();

    // 6. Named route override — if $no is non-numeric and is a real method, use it as action
    //    POST /users/login  → $no = 'login' → $action = 'login'
    //    POST /config/install → $no = 'install' → $action = 'install'
    if ($no !== null && !is_numeric($no) && method_exists($controller, $no)) {
        $action = $no;
    }

    if (!method_exists($controller, $action)) {
        throw new \InvalidArgumentException("Action '{$action}' not found on {$resource}");
    }

    echo $controller->$action($input);

} catch (\Throwable $e) {
    http_response_code(400);
    echo json_encode(['status' => 'error', 'message' => $e->getMessage()]);
}
```

---

## Action Resolution Table

| Request | `$resource` | `$no` | `$action` | Notes |
|---|---|---|---|---|
| `GET /posts` | `posts` | null | `list` | |
| `GET /posts/5` | `posts` | `5` | `show` | `$input['no'] = 5` injected |
| `POST /posts` | `posts` | null | `create` | |
| `PUT /posts/5` | `posts` | `5` | `update` | |
| `DELETE /posts/5` | `posts` | `5` | `delete` | |
| `POST /users/login` | `users` | `login` | `login` | named override |
| `POST /users/logout` | `users` | `logout` | `logout` | named override |
| `POST /config/install` | `config` | `install` | `install` | named override |

---

## Steps — Adding a Named Route

To add `POST /posts/publish`:

### 1. Add a `publish()` method to PostController

```php
public function publish(array $input): string
{
    return $this->json($this->svc->publishEntry($input));
}
```

### 2. Add `publishEntry()` to PostService

```php
public function publishEntry(array $input): array
{
    $user = $this->userService->authenticate($input['id'] ?? '', $input['password'] ?? '');
    if (!$user) throw new \InvalidArgumentException('Login required');
    // ... your logic
    return ['status' => 'success', 'message' => 'Post published'];
}
```

### 3. No changes to index.php needed

The named route override (`$no = 'publish'` + `method_exists($controller, 'publish')`) handles it automatically.

---

## Gotchas

- **URL resources must be plural** (`/users`, `/posts`, `/config`) — the router strips the trailing `s` to build the class name. Singular URLs will result in double-stripping (e.g. `user` → `use` → `Lib\Use\UseController` — not found).

- **`$no` from the URL is always a string** — even `'5'` is a string. The router casts it to `(int)` only when `is_numeric($no)` is true. Services always receive `$input['no']` as an integer.

- **Named routes require the method to exist on the Controller** — the override only fires if `method_exists($controller, $no)` returns true. If the method doesn't exist, `$action` stays as `create`/`delete`/etc. and you'll get an error.

- **Controllers and Services must never read `$_SERVER` or `php://input`** — all data comes through `array $input`. The router is the only place that reads the raw request.

- **`parse_url()` is required before trimming** — raw `REQUEST_URI` can include query strings (`/posts?page=2`). `parse_url(..., PHP_URL_PATH)` strips them cleanly before splitting into segments.

- **`rtrim($resource, 's')` is not a true inflector** — it only strips a trailing literal `s`. It works for `users→user`, `posts→post`, `configs→config`. It does NOT handle irregular plurals (e.g. `categories→categor`). For irregular plurals, add an explicit mapping in index.php.
