---
name: module
description: Step-by-step runbook for adding any new resource module (Entity → Repository → Service → Controller) to this PHP REST API. Use this skill when the user says "add a new resource", "create a [x] module", "add [x] endpoint", "new CRUD resource", "add a Comment module", "create a Category feature", or anything involving creating a new set of CRUD endpoints.
---

## When to use this skill

- "Add a new resource"
- "Create a [X] module" (e.g. Comment, Category, Order)
- "Add [X] endpoints"
- "New CRUD resource"
- "How do I add a feature?"

## Dependencies / Prerequisites

- setup skill: `lib/` exists, autoloader works
- contracts skill: all 7 contract files exist in `lib/Utils/`

---

## Steps

Replace `[ResourceName]` with the PascalCase name (e.g. `Comment`) and `[resource_name]` with snake_case (e.g. `comment`). The table name is typically the plural snake_case (e.g. `comments`).

### 1. Create lib/[ResourceName]/[ResourceName]Entity.php

```php
<?php
namespace Lib\[ResourceName];
use Lib\Utils\Entity;

class [ResourceName]Entity extends Entity
{
    public static function tableName(): string { return '[resource_name]s'; }

    // NEVER include password in columns()
    public static function columns(): array { return ['no', 'title', 'content', 'user_id', 'created_at', 'updated_at']; }

    // fillable = INSERT/UPDATE whitelist; update_at and server-set fields included
    public static function fillable(): array { return ['title', 'content', 'user_id', 'updated_at']; }

    public static function required(): array { return ['title', 'content', 'user_id']; }
}
```

**Rules:**
- `columns()` = what SELECT returns. Exclude `password` always.
- `fillable()` = what can be written. Include `updated_at` if the table has it.
- `required()` = validated on create via `Service::validate()`.

---

### 2. Create lib/[ResourceName]/[ResourceName]Repository.php

```php
<?php
namespace Lib\[ResourceName];
use Lib\Utils\Repository;

class [ResourceName]Repository extends Repository
{
    protected function tableName(): string { return [ResourceName]Entity::tableName(); }
    protected function columns(): array    { return [ResourceName]Entity::columns(); }
    protected function fillable(): array   { return [ResourceName]Entity::fillable(); }
    protected function required(): array   { return [ResourceName]Entity::required(); }

    // Add custom finders below only if needed.
    // Example: find by a foreign key
    // public function findByUserId(string $userId): array
    // {
    //     $db   = \Lib\Utils\Database::get();
    //     $cols = implode(', ', $this->columns());
    //     $stmt = $db->prepare("SELECT {$cols} FROM {$this->tableName()} WHERE user_id = :user_id ORDER BY no DESC");
    //     $stmt->execute([':user_id' => $userId]);
    //     return $stmt->fetchAll(\PDO::FETCH_ASSOC);
    // }
}
```

**Rules:**
- All 4 delegation methods must call the Entity statics — never hardcode strings here.
- Inherited `findAll`, `find`, `create`, `update`, `delete` are available automatically.
- Add custom finders here, but **never write SQL inside Service or Controller**.

---

### 3. Create lib/[ResourceName]/[ResourceName]Service.php

```php
<?php
namespace Lib\[ResourceName];
use Lib\User\UserService;
use Lib\Utils\RepositoryInterface;
use Lib\Utils\Service;

class [ResourceName]Service extends Service
{
    private [ResourceName]Repository $repo;
    private UserService $userService;

    public function __construct()
    {
        $this->repo        = new [ResourceName]Repository();
        $this->userService = new UserService();
    }

    protected function repository(): RepositoryInterface { return $this->repo; }

    public function listAll(array $input): array
    {
        return ['status' => 'success', '[resource_name]s' => $this->findAll()];
    }

    public function createEntry(array $input): array
    {
        // 1. Authenticate
        $user = $this->userService->authenticate($input['id'] ?? '', $input['password'] ?? '');
        if (!$user) throw new \InvalidArgumentException('Login required');

        // 2. Validate business fields
        if (empty($input['title']) || empty($input['content'])) {
            throw new \InvalidArgumentException('Title and content are required');
        }

        // 3. Create — user_id set from auth, NOT from $input
        $no = $this->create([
            'title'   => trim($input['title']),
            'content' => trim($input['content']),
            'user_id' => $user['id'],
        ]);

        return ['status' => 'success', 'no' => $no, 'user_id' => $user['id']];
    }

    public function getEntry(array $input): array
    {
        if (empty($input['no'])) throw new \InvalidArgumentException('no is required');

        $item = $this->find((int)$input['no']);
        if (!$item) throw new \InvalidArgumentException('[ResourceName] not found');

        return ['status' => 'success', '[resource_name]' => $item];
    }

    public function updateEntry(array $input): array
    {
        // 1. Authenticate
        $user = $this->userService->authenticate($input['id'] ?? '', $input['password'] ?? '');
        if (!$user) throw new \InvalidArgumentException('Login required');

        // 2. Find resource
        if (empty($input['no'])) throw new \InvalidArgumentException('no is required');
        $item = $this->find((int)$input['no']);
        if (!$item) throw new \InvalidArgumentException('[ResourceName] not found');

        // 3. Ownership check
        if ($item['user_id'] !== $user['id']) {
            throw new \InvalidArgumentException('Permission denied');
        }

        // 4. Build update data — updated_at set by server, not client
        $data = [];
        if (isset($input['title']))   $data['title']   = trim($input['title']);
        if (isset($input['content'])) $data['content']  = trim($input['content']);
        $data['updated_at'] = date('Y-m-d H:i:s');

        $this->update((int)$input['no'], $data);
        return ['status' => 'success', 'message' => '[ResourceName] updated'];
    }

    public function deleteEntry(array $input): array
    {
        // 1. Authenticate
        $user = $this->userService->authenticate($input['id'] ?? '', $input['password'] ?? '');
        if (!$user) throw new \InvalidArgumentException('Login required');

        // 2. Find resource
        if (empty($input['no'])) throw new \InvalidArgumentException('no is required');
        $item = $this->find((int)$input['no']);
        if (!$item) throw new \InvalidArgumentException('[ResourceName] not found');

        // 3. Ownership check
        if ($item['user_id'] !== $user['id']) {
            throw new \InvalidArgumentException('Permission denied');
        }

        $this->delete((int)$input['no']);
        return ['status' => 'success', 'message' => '[ResourceName] deleted'];
    }
}
```

---

### 4. Create lib/[ResourceName]/[ResourceName]Controller.php

```php
<?php
namespace Lib\[ResourceName];
use Lib\Utils\Controller;
use Lib\Utils\ServiceInterface;

class [ResourceName]Controller extends Controller
{
    private [ResourceName]Service $svc;

    public function __construct()    { $this->svc = new [ResourceName]Service(); }
    protected function service(): ServiceInterface { return $this->svc; }

    // Inherits: list, show, create, update, delete
    // Add extra methods here only for non-standard routes:
    // public function myAction(array $input): string { return $this->json(...); }
}
```

---

### 5. Add the table to ConfigController::install()

In `lib/Config/ConfigController.php`, add inside `install()`:

```php
$db->exec("DROP TABLE IF EXISTS [resource_name]s");
$db->exec("CREATE TABLE [resource_name]s (
    no         INTEGER PRIMARY KEY AUTOINCREMENT,
    title      TEXT NOT NULL,
    content    TEXT NOT NULL,
    user_id    TEXT NOT NULL,
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now'))
)");
```

### 6. Regenerate the autoloader

```bash
composer dump-autoload
```

### 7. Initialize and test

```bash
# Terminal 1
php -S 0.0.0.0:8080 index.php

# Terminal 2 — reset DB with new table
curl -X POST http://127.0.0.1:8080/config/install

# Smoke test
curl -X GET http://127.0.0.1:8080/[resource_name]s
```

---

## Gotchas

- **`user_id` must come from the authenticated user, never from `$input`** — e.g. `$user['id']` after `authenticate()`, not `$input['user_id']`. If you allow the client to set `user_id`, anyone can impersonate another user.

- **Always cast `$input['no']` to `(int)` before `find()` or `delete()`** — URL segments are strings. `find('5')` may work in SQLite but will fail strict type checks.

- **Ownership check is mandatory for update and delete** — always verify `$item['user_id'] === $user['id']` before mutating. Skipping this lets any authenticated user modify anyone's data.

- **`updated_at` must be set by the server** — use `date('Y-m-d H:i:s')`. Never trust `$input['updated_at']` from the client.

- **SQL only inside Repository** — if you need a custom query, add a method to `[ResourceName]Repository`. Services and Controllers must never call `Database::get()` or `$db->prepare()`.

- **Always update `fillable()` in the Entity when adding a new column** — `filterFillable()` silently drops fields not listed. Forgetting this is a common cause of "why is my new field not saving?"

- **Run `composer dump-autoload` after creating new files** — new classes are invisible to the autoloader until the classmap is regenerated.

- **Add the new table to `ConfigController::install()`** — otherwise `POST /config/install` won't create it and your INSERT will fail with "no such table".
