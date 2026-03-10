# PHP REST API from Scratch — Study Guide

A focused guide to building a framework-free PHP REST API with clean layered architecture, covering only **User** and **Post** resources. Consumed by a Flutter app via the Dio package.

Work through each phase in order. When you see a **🧪 QUIZ** marker, stop and ask Claude to generate a quiz for that phase before continuing.

---

## Table of Contents

- [Phase 1 — Technology & Project Setup](#phase-1--technology--project-setup)
- [Phase 2 — Architecture & Design Principles](#phase-2--architecture--design-principles)
- [Phase 3 — Interfaces & Abstract Classes](#phase-3--interfaces--abstract-classes)
- [Phase 4 — User Module](#phase-4--user-module)
- [Phase 5 — Post Module](#phase-5--post-module)
- [Phase 6 — Entry Point, Routing & Error Handling](#phase-6--entry-point-routing--error-handling)
- [Phase 7 — Testing with Pest PHP](#phase-7--testing-with-pest-php)
- [Phase 8 — Flutter Client, Reference & Pitfalls](#phase-8--flutter-client-reference--pitfalls)

---

## Phase 1 — Technology & Project Setup

### 1.1 Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | PHP 8.3.8+ | Arrow functions, named arguments, `match` expressions, enums |
| Database | SQLite via PDO | Zero-config, single-file, no server required |
| Autoloading | Composer PSR-4 | Industry standard, zero runtime overhead |
| Testing | Pest PHP | Expressive syntax, built on PHPUnit |
| Framework | **None** | Full control, no magic, every line is explicit and learnable |
| Client | Flutter + Dio | Cross-platform, clean HTTP verb support |

### 1.2 Standard REST Routing Convention

This project follows the REST convention where **HTTP method + URL path** determine the action. There is no `method` body field.

| HTTP Method | URL | Action |
|---|---|---|
| GET | `/products` | **list** — return all records |
| GET | `/products/123` | **show** — return one record |
| POST | `/products` | **create** — insert a new record |
| PUT or PATCH | `/products/123` | **update** — modify a record |
| DELETE | `/products/123` | **delete** — remove a record |

Applied to this project:

| HTTP Method | URL | Action |
|---|---|---|
| POST | `/users` | Register a new user |
| GET | `/users` | List all users |
| GET | `/users/{no}` | Show a single user |
| PUT | `/users/{no}` | Update a user |
| DELETE | `/users/{no}` | Delete a user |
| POST | `/users/login` | Login |
| POST | `/users/logout` | Logout |
| POST | `/posts` | Create a post |
| GET | `/posts` | List all posts |
| GET | `/posts/{no}` | Show a single post |
| PUT | `/posts/{no}` | Update a post |
| DELETE | `/posts/{no}` | Delete a post |
| POST | `/config/install` | Initialize the database |

### 1.3 Project Setup & Autoloading

#### Step 1 — Initialize Composer

```bash
composer init
```

#### Step 2 — composer.json

```json
{
    "autoload": {
        "psr-4": {
            "Lib\\": "lib/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    },
    "require-dev": {
        "pestphp/pest": "*"
    },
    "config": {
        "allow-plugins": {
            "pestphp/pest-plugin": true
        }
    }
}
```

`"Lib\\": "lib/"` tells Composer that any class whose fully-qualified name starts with `Lib\` lives under the `lib/` directory:

```
Lib\User\UserService    →  lib/User/UserService.php
Lib\Utils\Repository   →  lib/Utils/Repository.php
```

#### Step 3 — Install and generate the autoloader

```bash
composer install

# After adding new class files:
composer dump-autoload
```

#### Step 4 — Require the autoloader once

Only `index.php` (the front controller) needs this:

```php
require_once __DIR__ . '/vendor/autoload.php';
```

### 1.4 PSR-4 Autoloading

PSR-4 is a standard from PHP-FIG *(PHP Framework Interop Group, "PSR-4: Autoloader", 2013)* that defines a deterministic mapping between a fully-qualified class name and a file path. Composer implements it via its `ClassLoader`. Files load on first use — no manual `require` calls are ever needed.

### 1.5 Database Layer

`lib/Utils/Database.php` is a **static factory** that returns a PDO connection on demand.

```php
<?php
namespace Lib\Utils;

use PDO;

class Database
{
    static public $path = __DIR__ . '/database/database.db';

    static public function get(): PDO
    {
        $dir = dirname(self::$path);
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }

        $db = new PDO('sqlite:' . self::$path);
        $db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        return $db;
    }
}
```

Key points:

- `__DIR__` is always the directory containing this file — it does not change based on where PHP is invoked from. Using `./` instead would break when running tests or CLI commands from different directories.
- `PDO::ERRMODE_EXCEPTION` — SQL errors become catchable PHP exceptions, not silent failures.
- The `mkdir` call is inside `get()`, **not in a constructor**. Because `get()` is `static`, PHP never calls the constructor via `Database::get()`. Any setup logic must live inside the static method itself.
- To switch to MySQL, change only this file: `new PDO("mysql:host=...;dbname=...", $user, $pass)`.

### 1.6 Database Schema

Tables are created by calling `POST /config/install` (Phase 7). The SQL is:

```sql
CREATE TABLE users (
    no       INTEGER PRIMARY KEY AUTOINCREMENT,
    id       TEXT NOT NULL UNIQUE,
    password TEXT NOT NULL,
    name     TEXT DEFAULT '',
    created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE posts (
    no         INTEGER PRIMARY KEY AUTOINCREMENT,
    title      TEXT NOT NULL,
    content    TEXT NOT NULL,
    user_id    TEXT NOT NULL,
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now'))
);
```

---

<!-- QUIZ: Phase 1 — Technology & Project Setup -->
> 🧪 **QUIZ CHECKPOINT** — Ask Claude to generate a Phase 1 quiz before continuing.

---

## Phase 2 — Architecture & Design Principles

### 2.1 Architecture Overview

```
HTTP Request  (GET /posts/5, POST /posts, DELETE /users/2, ...)
      │
      ▼
  index.php  ◄── single front controller
      │           reads HTTP method + URL path
      │           resolves Controller class + action method
      ▼
  Controller  ◄── thin HTTP adapter, returns JSON
      │
      ▼
   Service   ◄── business rules, validation, auth checks
      │
      ▼
 Repository  ◄── SQL queries only — the sole DB access layer
      │
      ▼
  Database   ◄── PDO / SQLite static factory
```

Supporting structures (no layer of their own):

```
  Interface   ◄── defines the contract (what methods must exist)
      │
  Abstract    ◄── provides default implementation (the how)
      │
  Concrete    ◄── module-specific logic (UserService, PostRepository, ...)
```

Every module (User, Post, Config) follows the same 4-file pattern:
`XxxEntity` → `XxxRepository` → `XxxService` → `XxxController`

### 2.2 Design Principles

#### Separation of Concerns (SoC)

> "Gather together the things that change for the same reasons. Separate those things that change for different reasons."
> — Robert C. Martin, *Clean Architecture* (2017), Chapter 7

| Layer | Only reason to change |
|---|---|
| Entity | The table schema changes |
| Repository | The SQL dialect or DB engine changes |
| Service | Business rules change |
| Controller | The HTTP protocol or response format changes |

#### Repository Pattern

> "A Repository mediates between the domain and data mapping layers, acting like an in-memory domain object collection."
> — Martin Fowler, *Patterns of Enterprise Application Architecture* (2002), p. 322

Isolating all SQL in the Repository means you can swap SQLite for MySQL or PostgreSQL by changing only that layer.

#### Dependency Inversion Principle (DIP)

> "High-level modules should not depend on low-level modules. Both should depend on abstractions."
> — Robert C. Martin, *Agile Software Development, Principles, Patterns, and Practices* (2002), Chapter 11

In this project, `Service` depends on `RepositoryInterface`, not on `UserRepository` directly. This means a Service can be tested with a mock repository.

#### Interface Segregation Principle (ISP)

> "No client should be forced to depend on methods it does not use."
> — Robert C. Martin, *Agile Software Development* (2002), Chapter 12

Each interface in this project is narrow and focused:
- `RepositoryInterface` — only CRUD
- `ServiceInterface` — only the 5 entry methods
- `EntityInterface` — only schema metadata

#### Template Method Pattern

> "Define the skeleton of an algorithm in a base class, deferring some steps to subclasses."
> — Gang of Four, *Design Patterns: Elements of Reusable Object-Oriented Software* (1994), p. 325

`Repository` (abstract class) provides the full SQL implementation for `findAll`, `create`, etc. Concrete repos like `UserRepository` only supply the table name and column list — the algorithm is inherited.

### 2.3 Interfaces vs Abstract Classes — The Two-Layer Contract

This project uses **both** — for different purposes:

| | Interface | Abstract Class |
|---|---|---|
| **Purpose** | Define *what* methods must exist | Define *how* methods work (default impl.) |
| **Can have method bodies?** | No (PHP < 8.0) | Yes |
| **Can be instantiated?** | No | No |
| **Supports multiple inheritance?** | Yes | No |
| **When to use** | Public contract between layers | Shared implementation across subclasses |

The pattern used here:

```
Interface          ← defines the contract (what)
    │
Abstract Class     ← implements the interface + provides defaults (how)
    │
Concrete Class     ← provides module-specific details (which)
```

Example:

```php
// What a Repository must be able to do:
interface RepositoryInterface {
    public function findAll(): array;
    public function find(int $no): ?array;
    // ...
}

// How it does it (default SQL implementation):
abstract class Repository implements RepositoryInterface {
    public function findAll(): array { /* PDO SELECT * ... */ }
    // ...
}

// Which table and columns apply:
class UserRepository extends Repository {
    protected function tableName(): string { return 'users'; }
    // ...
}
```

### 2.4 Directory Structure

```
project-root/
├── index.php              # Front controller + built-in server router (no .htaccess needed)
├── composer.json
├── lib/
│   ├── Utils/
│   │   ├── EntityInterface.php      # interface
│   │   ├── RepositoryInterface.php  # interface
│   │   ├── ServiceInterface.php     # interface
│   │   ├── Entity.php               # abstract class implements EntityInterface
│   │   ├── Repository.php           # abstract class implements RepositoryInterface
│   │   ├── Service.php              # abstract class implements ServiceInterface
│   │   ├── Controller.php           # abstract class
│   │   └── Database.php             # static PDO factory
│   ├── User/
│   │   ├── UserEntity.php
│   │   ├── UserRepository.php
│   │   ├── UserService.php
│   │   └── UserController.php
│   ├── Post/
│   │   ├── PostEntity.php
│   │   ├── PostRepository.php
│   │   ├── PostService.php
│   │   └── PostController.php
│   └── Config/
│       └── ConfigController.php
└── tests/
    ├── Pest.php
    ├── TestCase.php
    └── Feature/
        ├── UserControllerTest.php
        └── PostControllerTest.php
```

---

<!-- QUIZ: Phase 2 — Architecture & Design Principles -->
> 🧪 **QUIZ CHECKPOINT** — Ask Claude to generate a Phase 2 quiz before continuing.

---

## Phase 3 — Interfaces & Abstract Classes

### 3.1 EntityInterface

Defines the schema metadata contract. Any class that describes a database table must implement these four static methods.

```php
<?php
// lib/Utils/EntityInterface.php
namespace Lib\Utils;

interface EntityInterface
{
    public static function tableName(): string;
    public static function columns(): array;   // columns to SELECT
    public static function fillable(): array;  // INSERT/UPDATE whitelist
    public static function required(): array;  // must be present on create
}
```

| Method | Purpose |
|---|---|
| `tableName()` | The SQL table name string |
| `columns()` | Which columns to SELECT. Sensitive fields like `password` are excluded. |
| `fillable()` | Whitelist for INSERT/UPDATE — fields not listed are silently stripped (mass-assignment protection). |
| `required()` | Fields that must be present and non-empty on create. |

### 3.2 Entity — Abstract Class

`Entity` implements `EntityInterface` and declares all four methods as `abstract`, forcing every concrete entity to supply them. It provides no default behaviour — it is purely a base class for type checking and contract enforcement.

```php
<?php
// lib/Utils/Entity.php
namespace Lib\Utils;

abstract class Entity implements EntityInterface
{
    abstract public static function tableName(): string;
    abstract public static function columns(): array;
    abstract public static function fillable(): array;
    abstract public static function required(): array;
}
```

**Why have an abstract class if it adds no logic?**
It provides a concrete PHP type (`Entity`) that can be used for type-hints and `instanceof` checks elsewhere, without coupling code to a specific module. It also allows you to add shared static helpers later without modifying the interface.

### 3.3 RepositoryInterface

Defines what every Repository can do. This is the contract the Service layer depends on.

```php
<?php
// lib/Utils/RepositoryInterface.php
namespace Lib\Utils;

interface RepositoryInterface
{
    public function findAll(): array;
    public function find(int $no): ?array;
    public function create(array $data): string; // returns lastInsertId
    public function update(int $no, array $data): int; // returns rowCount
    public function delete(int $no): int;             // returns rowCount
    public function getRequired(): array;
}
```

### 3.4 Repository — Abstract Class

`Repository` implements `RepositoryInterface` and provides the full, reusable SQL implementations. Concrete repositories only implement the four abstract metadata methods (`tableName`, `columns`, `fillable`, `required`).

```php
<?php
// lib/Utils/Repository.php
namespace Lib\Utils;
use PDO;

abstract class Repository implements RepositoryInterface
{
    abstract protected function tableName(): string;
    abstract protected function columns(): array;
    abstract protected function fillable(): array;
    abstract protected function required(): array;

    public function getRequired(): array
    {
        return $this->required();
    }

    // Mass-assignment guard: silently strips any key not in fillable()
    protected function filterFillable(array $data): array
    {
        return array_intersect_key($data, array_flip($this->fillable()));
    }

    public function findAll(): array
    {
        $db   = Database::get();
        $cols = implode(', ', $this->columns());
        $stmt = $db->query("SELECT {$cols} FROM {$this->tableName()} ORDER BY no DESC");
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    public function find(int $no): ?array
    {
        $db   = Database::get();
        $cols = implode(', ', $this->columns());
        $stmt = $db->prepare("SELECT {$cols} FROM {$this->tableName()} WHERE no = :no");
        $stmt->execute([':no' => $no]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        return $row ?: null;
    }

    public function create(array $data): string
    {
        $data = $this->filterFillable($data);
        if (empty($data)) {
            throw new \InvalidArgumentException('No fillable fields provided for create');
        }

        $db           = Database::get();
        $keys         = array_keys($data);
        $cols         = implode(', ', $keys);
        $placeholders = implode(', ', array_map(fn($k) => ':' . $k, $keys));
        $params       = [];
        foreach ($data as $k => $v) {
            $params[':' . $k] = $v;
        }

        $stmt = $db->prepare("INSERT INTO {$this->tableName()} ({$cols}) VALUES ({$placeholders})");
        $stmt->execute($params);
        return $db->lastInsertId();
    }

    public function update(int $no, array $data): int
    {
        $data = $this->filterFillable($data);
        if (empty($data)) {
            throw new \InvalidArgumentException('No fillable fields provided for update');
        }

        $db     = Database::get();
        $sets   = [];
        $params = [':no' => $no];
        foreach ($data as $k => $v) {
            $sets[]           = "{$k} = :{$k}";
            $params[':' . $k] = $v;
        }

        $sql  = "UPDATE {$this->tableName()} SET " . implode(', ', $sets) . " WHERE no = :no";
        $stmt = $db->prepare($sql);
        $stmt->execute($params);
        return $stmt->rowCount();
    }

    public function delete(int $no): int
    {
        $db   = Database::get();
        $stmt = $db->prepare("DELETE FROM {$this->tableName()} WHERE no = :no");
        $stmt->execute([':no' => $no]);
        return $stmt->rowCount();
    }
}
```

> **Security note:** Every query uses PDO prepared statements with named placeholders (`:no`, `:id`). User-supplied values are never interpolated directly into SQL strings, making SQL injection impossible.

### 3.5 ServiceInterface

Defines the five public actions every service must expose — one per REST action.

```php
<?php
// lib/Utils/ServiceInterface.php
namespace Lib\Utils;

interface ServiceInterface
{
    public function listAll(array $input): array;
    public function createEntry(array $input): array;
    public function getEntry(array $input): array;
    public function updateEntry(array $input): array;
    public function deleteEntry(array $input): array;
}
```

### 3.6 Service — Abstract Class

`Service` implements `ServiceInterface`, declares all five entry methods as `abstract`, and provides shared helpers: `validate()` and convenience delegators to the Repository.

```php
<?php
// lib/Utils/Service.php
namespace Lib\Utils;

abstract class Service implements ServiceInterface
{
    abstract protected function repository(): RepositoryInterface;

    abstract public function listAll(array $input): array;
    abstract public function createEntry(array $input): array;
    abstract public function getEntry(array $input): array;
    abstract public function updateEntry(array $input): array;
    abstract public function deleteEntry(array $input): array;

    // Shared validation helper — throws on missing or blank required fields
    public function validate(array $data, array $requiredFields): void
    {
        $missing = [];
        foreach ($requiredFields as $field) {
            if (!isset($data[$field]) || (is_string($data[$field]) && trim($data[$field]) === '')) {
                $missing[] = $field;
            }
        }
        if (!empty($missing)) {
            throw new \InvalidArgumentException('Required fields missing: ' . implode(', ', $missing));
        }
    }

    // Convenience delegators — child Services call these rather than the Repository directly
    public function findAll(): array      { return $this->repository()->findAll(); }
    public function find(int $no): ?array { return $this->repository()->find($no); }

    public function create(array $data): string
    {
        $this->validate($data, $this->repository()->getRequired());
        return $this->repository()->create($data);
    }

    public function update(int $no, array $data): int { return $this->repository()->update($no, $data); }
    public function delete(int $no): int              { return $this->repository()->delete($no); }
}
```

### 3.7 Controller — Abstract Class

The Controller is the HTTP boundary. It converts a raw `array $input` into a JSON response string. No business logic belongs here.

```php
<?php
// lib/Utils/Controller.php
namespace Lib\Utils;

abstract class Controller
{
    abstract protected function service(): ServiceInterface;

    protected function json(array $data): string
    {
        header('Content-Type: application/json');
        return json_encode($data);
    }

    // Standard REST actions — map directly to ServiceInterface methods:
    public function list(array $input): string   { return $this->json($this->service()->listAll($input)); }
    public function show(array $input): string   { return $this->json($this->service()->getEntry($input)); }
    public function create(array $input): string { return $this->json($this->service()->createEntry($input)); }
    public function update(array $input): string { return $this->json($this->service()->updateEntry($input)); }
    public function delete(array $input): string { return $this->json($this->service()->deleteEntry($input)); }
}
```

The HTTP-to-method mapping:

| HTTP Verb | URL shape | Controller method | ServiceInterface method |
|---|---|---|---|
| GET | `/resource` | `list` | `listAll` |
| GET | `/resource/{no}` | `show` | `getEntry` |
| POST | `/resource` | `create` | `createEntry` |
| PUT / PATCH | `/resource/{no}` | `update` | `updateEntry` |
| DELETE | `/resource/{no}` | `delete` | `deleteEntry` |

### 3.8 The Full Contract Chain

```
EntityInterface          ← 4 static schema methods
    ↑ implements
  Entity (abstract)      ← all 4 abstract — forces concrete impl.
    ↑ extends
  UserEntity             ← returns 'users', ['no','id','name',...], etc.


RepositoryInterface      ← findAll, find, create, update, delete, getRequired
    ↑ implements
  Repository (abstract)  ← PDO SQL implementations + filterFillable
    ↑ extends
  UserRepository         ← tableName/columns/fillable/required from UserEntity
                            + custom: findById, findByIdWithPassword


ServiceInterface         ← listAll, createEntry, getEntry, updateEntry, deleteEntry
    ↑ implements
  Service (abstract)     ← validate() + convenience delegators
    ↑ extends
  UserService            ← full business logic: auth, password hash, ownership


Controller (abstract)    ← json() + 5 CRUD methods delegating to ServiceInterface
    ↑ extends
  UserController         ← service() → UserService
                            + extra: login(), logout()
```

---

<!-- QUIZ: Phase 3 — Interfaces & Abstract Classes -->
> 🧪 **QUIZ CHECKPOINT** — Ask Claude to generate a Phase 3 quiz before continuing.

---

## Phase 4 — User Module

### 4.1 UserEntity

```php
<?php
namespace Lib\User;
use Lib\Utils\Entity;

class UserEntity extends Entity
{
    public static function tableName(): string { return 'users'; }

    // password excluded from default SELECT — never returned accidentally
    public static function columns(): array { return ['no', 'id', 'name', 'created_at']; }

    // password IS fillable — needs to be stored (hashed)
    public static function fillable(): array { return ['id', 'password', 'name']; }

    public static function required(): array { return ['id', 'password']; }
}
```

### 4.2 UserRepository

```php
<?php
namespace Lib\User;
use Lib\Utils\Database;
use Lib\Utils\Repository;
use PDO;

class UserRepository extends Repository
{
    protected function tableName(): string { return UserEntity::tableName(); }
    protected function columns(): array    { return UserEntity::columns(); }
    protected function fillable(): array   { return UserEntity::fillable(); }
    protected function required(): array   { return UserEntity::required(); }

    // Lookup without password — for profile display
    public function findById(string $id): ?array
    {
        $db   = Database::get();
        $cols = implode(', ', $this->columns());
        $stmt = $db->prepare("SELECT {$cols} FROM {$this->tableName()} WHERE id = :id");
        $stmt->execute([':id' => $id]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        return $row ?: null;
    }

    // Lookup with password — for authentication only
    public function findByIdWithPassword(string $id): ?array
    {
        $db   = Database::get();
        $stmt = $db->prepare("SELECT no, id, password, name FROM {$this->tableName()} WHERE id = :id");
        $stmt->execute([':id' => $id]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        return $row ?: null;
    }
}
```

**Why two separate find methods?**
`findById` uses `columns()`, which excludes `password` by design. `findByIdWithPassword` hard-codes the column list to include the hash — it is used only during authentication and the hash is immediately discarded after `password_verify()`.

### 4.3 UserService

```php
<?php
namespace Lib\User;
use Lib\Utils\RepositoryInterface;
use Lib\Utils\Service;

class UserService extends Service
{
    private UserRepository $repo;

    public function __construct()
    {
        $this->repo = new UserRepository();
    }

    protected function repository(): RepositoryInterface { return $this->repo; }

    // ── Authentication helper (used by PostService too) ──────────────────

    public function authenticate(string $id, string $password): ?array
    {
        if (empty($id) || empty($password)) return null;

        $user = $this->repo->findByIdWithPassword($id);
        if (!$user || !password_verify($password, $user['password'])) return null;

        unset($user['password']); // never expose the hash
        return $user;
    }

    // ── ServiceInterface implementation ──────────────────────────────────

    public function listAll(array $input): array
    {
        return ['status' => 'success', 'users' => $this->findAll()];
    }

    public function createEntry(array $input): array
    {
        if (empty($input['id']) || empty($input['password'])) {
            throw new \InvalidArgumentException('ID and password are required');
        }

        $id = trim($input['id']);

        if (strlen($id) < 3) {
            throw new \InvalidArgumentException('ID must be at least 3 characters');
        }
        if (strlen($input['password']) < 6) {
            throw new \InvalidArgumentException('Password must be at least 6 characters');
        }
        if ($this->repo->findById($id)) {
            throw new \InvalidArgumentException('ID already exists');
        }

        $name = trim($input['name'] ?? '');
        $no   = $this->create([
            'id'       => $id,
            'password' => password_hash($input['password'], PASSWORD_DEFAULT),
            'name'     => $name,
        ]);

        return ['status' => 'success', 'no' => $no, 'id' => $id, 'name' => $name];
    }

    public function getEntry(array $input): array
    {
        if (empty($input['no'])) throw new \InvalidArgumentException('no is required');

        $user = $this->find((int)$input['no']);
        if (!$user) throw new \InvalidArgumentException('User not found');

        return ['status' => 'success', 'user' => $user];
    }

    public function updateEntry(array $input): array
    {
        $user = $this->authenticate($input['id'] ?? '', $input['password'] ?? '');
        if (!$user) throw new \InvalidArgumentException('Authentication failed');

        $data = [];
        if (isset($input['name']))          $data['name']     = trim($input['name']);
        if (!empty($input['new_password'])) $data['password'] = password_hash($input['new_password'], PASSWORD_DEFAULT);

        if (empty($data)) throw new \InvalidArgumentException('No fields to update');

        $this->update((int)$user['no'], $data);
        return ['status' => 'success', 'message' => 'User updated'];
    }

    public function deleteEntry(array $input): array
    {
        // Must be authenticated — no unauthenticated deletes
        $user = $this->authenticate($input['id'] ?? '', $input['password'] ?? '');
        if (!$user) throw new \InvalidArgumentException('Authentication failed');

        if (empty($input['no'])) throw new \InvalidArgumentException('no is required');

        // Users may only delete their own account
        if ((int)$user['no'] !== (int)$input['no']) {
            throw new \InvalidArgumentException('Permission denied');
        }

        $affected = $this->delete((int)$input['no']);
        if ($affected === 0) throw new \InvalidArgumentException('User not found');

        return ['status' => 'success', 'message' => 'User deleted'];
    }

    // ── Extra entries for custom routes ──────────────────────────────────

    public function loginEntry(array $input): array
    {
        $user = $this->authenticate($input['id'] ?? '', $input['password'] ?? '');
        if (!$user) throw new \InvalidArgumentException('Login failed');
        return ['status' => 'success', 'user' => $user];
    }

    public function logoutEntry(array $input): array
    {
        // Stateless API — no server-side session to invalidate
        return ['status' => 'success', 'message' => 'Logged out'];
    }
}
```

### 4.4 UserController

```php
<?php
namespace Lib\User;
use Lib\Utils\Controller;
use Lib\Utils\ServiceInterface;

class UserController extends Controller
{
    private UserService $svc;

    public function __construct()
    {
        $this->svc = new UserService();
    }

    protected function service(): ServiceInterface { return $this->svc; }

    // Inherits: list, show, create, update, delete

    // Custom routes beyond standard CRUD:
    // POST /users/login  → login()
    // POST /users/logout → logout()
    public function login(array $input): string  { return $this->json($this->svc->loginEntry($input)); }
    public function logout(array $input): string { return $this->json($this->svc->logoutEntry($input)); }
}
```

---

<!-- QUIZ: Phase 4 — User Module -->
> 🧪 **QUIZ CHECKPOINT** — Ask Claude to generate a Phase 4 quiz before continuing.

---

## Phase 5 — Post Module

### 5.1 PostEntity

```php
<?php
namespace Lib\Post;
use Lib\Utils\Entity;

class PostEntity extends Entity
{
    public static function tableName(): string { return 'posts'; }
    public static function columns(): array    { return ['no', 'title', 'content', 'user_id', 'created_at', 'updated_at']; }
    public static function fillable(): array   { return ['title', 'content', 'user_id', 'updated_at']; }
    public static function required(): array   { return ['title', 'content', 'user_id']; }
}
```

Note: `user_id` is in `fillable` because the Service inserts it — but it always comes from the authenticated user, never from raw client input.

### 5.2 PostRepository

```php
<?php
namespace Lib\Post;
use Lib\Utils\Repository;

class PostRepository extends Repository
{
    protected function tableName(): string { return PostEntity::tableName(); }
    protected function columns(): array    { return PostEntity::columns(); }
    protected function fillable(): array   { return PostEntity::fillable(); }
    protected function required(): array   { return PostEntity::required(); }
    // Inherits findAll, find, create, update, delete from Repository
}
```

### 5.3 PostService

```php
<?php
namespace Lib\Post;
use Lib\User\UserService;
use Lib\Utils\RepositoryInterface;
use Lib\Utils\Service;

class PostService extends Service
{
    private PostRepository $repo;
    private UserService    $userService;

    public function __construct()
    {
        $this->repo        = new PostRepository();
        $this->userService = new UserService();
    }

    protected function repository(): RepositoryInterface { return $this->repo; }

    public function listAll(array $input): array
    {
        return ['status' => 'success', 'posts' => $this->findAll()];
    }

    public function createEntry(array $input): array
    {
        // Authenticate first — credentials sent in the request body
        $user = $this->userService->authenticate($input['id'] ?? '', $input['password'] ?? '');
        if (!$user) throw new \InvalidArgumentException('Login required');

        if (empty($input['title']) || empty($input['content'])) {
            throw new \InvalidArgumentException('Title and content are required');
        }

        $no = $this->create([
            'title'   => trim($input['title']),
            'content' => trim($input['content']),
            'user_id' => $user['id'],  // set from auth, not from client
        ]);

        return ['status' => 'success', 'no' => $no, 'user_id' => $user['id']];
    }

    public function getEntry(array $input): array
    {
        if (empty($input['no'])) throw new \InvalidArgumentException('no is required');

        $post = $this->find((int)$input['no']);
        if (!$post) throw new \InvalidArgumentException('Post not found');

        return ['status' => 'success', 'post' => $post];
    }

    public function updateEntry(array $input): array
    {
        $user = $this->userService->authenticate($input['id'] ?? '', $input['password'] ?? '');
        if (!$user) throw new \InvalidArgumentException('Login required');

        if (empty($input['no'])) throw new \InvalidArgumentException('no is required');

        $post = $this->find((int)$input['no']);
        if (!$post) throw new \InvalidArgumentException('Post not found');

        // Ownership check — only the author may edit
        if ($post['user_id'] !== $user['id']) {
            throw new \InvalidArgumentException('Permission denied');
        }

        $data = [];
        if (isset($input['title']))   $data['title']   = trim($input['title']);
        if (isset($input['content'])) $data['content']  = trim($input['content']);
        $data['updated_at'] = date('Y-m-d H:i:s'); // set by server, not client

        $this->update((int)$input['no'], $data);
        return ['status' => 'success', 'message' => 'Post updated'];
    }

    public function deleteEntry(array $input): array
    {
        $user = $this->userService->authenticate($input['id'] ?? '', $input['password'] ?? '');
        if (!$user) throw new \InvalidArgumentException('Login required');

        if (empty($input['no'])) throw new \InvalidArgumentException('no is required');

        $post = $this->find((int)$input['no']);
        if (!$post) throw new \InvalidArgumentException('Post not found');

        if ($post['user_id'] !== $user['id']) {
            throw new \InvalidArgumentException('Permission denied');
        }

        $this->delete((int)$input['no']);
        return ['status' => 'success', 'message' => 'Post deleted'];
    }
}
```

### 5.4 PostController

```php
<?php
namespace Lib\Post;
use Lib\Utils\Controller;
use Lib\Utils\ServiceInterface;

class PostController extends Controller
{
    private PostService $svc;

    public function __construct()    { $this->svc = new PostService(); }
    protected function service(): ServiceInterface { return $this->svc; }
    // Inherits: list, show, create, update, delete — nothing else needed
}
```

---

<!-- QUIZ: Phase 5 — Post Module -->
> 🧪 **QUIZ CHECKPOINT** — Ask Claude to generate a Phase 5 quiz before continuing.

---

## Phase 6 — Entry Point, Routing & Error Handling

### 6.1 Built-in Server Router

This project uses PHP's built-in server — no Apache or `.htaccess` needed. Pass `index.php` as the router script so every request is handled through it:

```bash
php -S 0.0.0.0:8080 index.php
```

When a router script is specified, PHP's built-in server calls it for every request. If the router returns `false`, the server serves the file directly (useful for static assets). Otherwise `index.php` handles the request.

Add this guard at the very top of `index.php` (before `require_once`):

```php
// Built-in server: serve real static files (images, css, js) directly
if (PHP_SAPI === 'cli-server' && is_file(__DIR__ . $_SERVER['REQUEST_URI'])) {
    return false;
}
```

`$_SERVER['REQUEST_URI']` retains the original path (`/posts/5`), so the routing logic works unchanged.

### 6.2 Front Controller — index.php

```php
<?php
require_once __DIR__ . '/vendor/autoload.php';

header('Content-Type: application/json');

try {
    $httpMethod = $_SERVER['REQUEST_METHOD'];

    // Parse: GET /posts/5  →  segments = ['posts', '5']
    $uri      = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
    $uri      = trim($uri, '/');
    $segments = explode('/', $uri);

    $resource = $segments[0] ?? '';   // 'posts'
    $no       = $segments[1] ?? null; // '5', 'login', or null

    // Read JSON body; fall back to form data
    $body  = file_get_contents('php://input');
    $input = json_decode($body, true) ?? $_REQUEST ?? [];

    // Inject numeric {no} from URL so Services receive it in $input
    if ($no !== null && is_numeric($no)) {
        $input['no'] = (int)$no;
    }

    // Resolve controller class from resource name
    // Strip plural 's': 'posts' → 'post' → Lib\Post\PostController
    //                   'users' → 'user' → Lib\User\UserController
    //                   'config' stays 'config' → Lib\Config\ConfigController
    $singular        = rtrim($resource, 's');
    $controllerClass = 'Lib\\' . ucfirst($singular) . '\\' . ucfirst($singular) . 'Controller';

    if (!class_exists($controllerClass)) {
        throw new \InvalidArgumentException("Unknown resource: {$resource}");
    }

    // Resolve action from HTTP method + whether {no} is present
    $action = match(true) {
        $httpMethod === 'GET'                  && $no === null => 'list',
        $httpMethod === 'GET'                  && $no !== null => 'show',
        $httpMethod === 'POST'                                 => 'create',
        in_array($httpMethod, ['PUT', 'PATCH'])                => 'update',
        $httpMethod === 'DELETE'                               => 'delete',
        default => throw new \InvalidArgumentException("Unsupported method: {$httpMethod}"),
    };

    $controller = new $controllerClass();

    // Named route override: POST /users/login → $no = 'login' → action = 'login'
    //                       POST /config/install → action = 'install'
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

### 6.3 How the Router Resolves Each Request

| Request | `$resource` | `$no` | Controller | Action |
|---|---|---|---|---|
| `GET /posts` | `posts` | null | PostController | `list` |
| `GET /posts/5` | `posts` | `5` | PostController | `show` |
| `POST /posts` | `posts` | null | PostController | `create` |
| `PUT /posts/5` | `posts` | `5` | PostController | `update` |
| `DELETE /posts/5` | `posts` | `5` | PostController | `delete` |
| `POST /users/login` | `users` | `login` | UserController | `login` *(named override)* |
| `POST /users/logout` | `users` | `logout` | UserController | `logout` *(named override)* |
| `POST /config/install` | `config` | `install` | ConfigController | `install` *(named override)* |

### 6.4 Input Parsing

All data flows through a single `array $input`. Controllers and Services never inspect `$_SERVER` or call `file_get_contents` directly.

- **JSON body** — `php://input` decoded. Dio sends `Content-Type: application/json` by default.
- **`{no}` from URL** — cast to `int` and injected into `$input['no']` by the router.

### 6.5 Error Handling

All business errors are thrown as `\InvalidArgumentException` inside Service methods. The top-level `catch (\Throwable $e)` returns HTTP 400 with a JSON envelope:

```json
{ "status": "error", "message": "Post not found" }
```

All success responses include `"status": "success"`:

```json
{ "status": "success", "post": { "no": 1, "title": "Hello", ... } }
```

---

<!-- QUIZ: Phase 6 — Entry Point, Routing & Error Handling -->
> 🧪 **QUIZ CHECKPOINT** — Ask Claude to generate a Phase 6 quiz before continuing.

---

## Phase 7 — Testing with Pest PHP

### 7.1 Setup

```bash
composer require --dev pestphp/pest
./vendor/bin/pest --init
```

### 7.2 Database Initialization Endpoint

`ConfigController` exposes `POST /config/install` which drops and recreates all tables:

```php
<?php
namespace Lib\Config;
use Lib\Utils\Controller;
use Lib\Utils\Database;
use Lib\Utils\ServiceInterface;

class ConfigController extends Controller
{
    protected function service(): ServiceInterface
    {
        throw new \LogicException('Config has no standard CRUD service');
    }

    public function install(array $input = []): string
    {
        $db = Database::get();

        $db->exec("DROP TABLE IF EXISTS users");
        $db->exec("CREATE TABLE users (
            no INTEGER PRIMARY KEY AUTOINCREMENT,
            id TEXT UNIQUE NOT NULL,
            password TEXT NOT NULL,
            name TEXT DEFAULT '',
            created_at TEXT DEFAULT (datetime('now'))
        )");

        $db->exec("DROP TABLE IF EXISTS posts");
        $db->exec("CREATE TABLE posts (
            no INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            content TEXT NOT NULL,
            user_id TEXT NOT NULL,
            created_at TEXT DEFAULT (datetime('now')),
            updated_at TEXT DEFAULT (datetime('now'))
        )");

        return $this->json(['status' => 'success', 'message' => 'Database installed']);
    }
}
```

Call it once after deployment:

```bash
curl -X POST http://localhost/config/install
```

### 7.3 tests/Pest.php — Bootstrap and api() Helper

```php
<?php
pest()->extend(Tests\TestCase::class)->in('Feature');

define('API_BASE_URL', 'http://127.0.0.1:8080');

/**
 * @param string $method  HTTP verb: GET, POST, PUT, DELETE
 * @param string $path    e.g. '/posts'  or  '/posts/5'
 * @param array  $body    JSON body for POST / PUT
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

### 7.4 UserControllerTest.php

```php
<?php

beforeAll(function () {
    api('POST', '/config/install');
});

test('POST /users — register success', function () {
    $res = api('POST', '/users', ['id' => 'alice', 'password' => 'secret123', 'name' => 'Alice']);

    expect($res['status'])->toBe('success')
        ->and($res['no'])->toBeGreaterThan(0)
        ->and($res['id'])->toBe('alice');
});

test('POST /users — duplicate id returns error', function () {
    api('POST', '/users', ['id' => 'dupuser', 'password' => 'pass123']);
    $res = api('POST', '/users', ['id' => 'dupuser', 'password' => 'pass123']);

    expect($res['status'])->toBe('error')
        ->and($res['message'])->toContain('already exists');
});

test('POST /users/login — success', function () {
    api('POST', '/users', ['id' => 'logintest', 'password' => 'pass123']);
    $res = api('POST', '/users/login', ['id' => 'logintest', 'password' => 'pass123']);

    expect($res['status'])->toBe('success')
        ->and($res['user']['id'])->toBe('logintest')
        ->and($res['user'])->not->toHaveKey('password');
});

test('POST /users/login — wrong password returns error', function () {
    $res = api('POST', '/users/login', ['id' => 'logintest', 'password' => 'wrong']);

    expect($res['status'])->toBe('error')
        ->and($res['message'])->toContain('Login failed');
});

test('GET /users/{no} — show user', function () {
    $created = api('POST', '/users', ['id' => 'showtest', 'password' => 'pass123']);
    $res     = api('GET', '/users/' . $created['no']);

    expect($res['status'])->toBe('success')
        ->and($res['user']['id'])->toBe('showtest');
});

test('GET /users/{no} — not found returns error', function () {
    $res = api('GET', '/users/99999');

    expect($res['status'])->toBe('error')
        ->and($res['message'])->toContain('not found');
});
```

### 7.5 PostControllerTest.php

```php
<?php

define('POST_USER',  'postuser');
define('POST_PASS',  'password123');
define('OTHER_USER', 'otheruser');
define('OTHER_PASS', 'password456');

beforeAll(function () {
    api('POST', '/config/install');
    api('POST', '/users', ['id' => POST_USER,  'password' => POST_PASS]);
    api('POST', '/users', ['id' => OTHER_USER, 'password' => OTHER_PASS]);
});

test('POST /posts — create success', function () {
    $res = api('POST', '/posts', [
        'title' => 'Hello', 'content' => 'World',
        'id' => POST_USER, 'password' => POST_PASS,
    ]);

    expect($res['status'])->toBe('success')
        ->and($res['no'])->toBeGreaterThan(0);
});

test('POST /posts — unauthenticated returns error', function () {
    $res = api('POST', '/posts', ['title' => 'T', 'content' => 'C']);

    expect($res['status'])->toBe('error')
        ->and($res['message'])->toContain('Login required');
});

test('GET /posts — list returns array', function () {
    $res = api('GET', '/posts');

    expect($res['status'])->toBe('success')
        ->and($res['posts'])->toBeArray();
});

test('GET /posts/{no} — show post', function () {
    $created = api('POST', '/posts', [
        'title' => 'Show Me', 'content' => 'Body',
        'id' => POST_USER, 'password' => POST_PASS,
    ]);
    $res = api('GET', '/posts/' . $created['no']);

    expect($res['status'])->toBe('success')
        ->and($res['post']['title'])->toBe('Show Me');
});

test('PUT /posts/{no} — update success', function () {
    $created = api('POST', '/posts', [
        'title' => 'Old', 'content' => 'Old body',
        'id' => POST_USER, 'password' => POST_PASS,
    ]);
    $res = api('PUT', '/posts/' . $created['no'], [
        'title' => 'New', 'id' => POST_USER, 'password' => POST_PASS,
    ]);

    expect($res['status'])->toBe('success');

    $post = api('GET', '/posts/' . $created['no']);
    expect($post['post']['title'])->toBe('New');
});

test('PUT /posts/{no} — permission denied for other user', function () {
    $created = api('POST', '/posts', [
        'title' => 'Mine', 'content' => 'Body',
        'id' => POST_USER, 'password' => POST_PASS,
    ]);
    $res = api('PUT', '/posts/' . $created['no'], [
        'title' => 'Hacked', 'id' => OTHER_USER, 'password' => OTHER_PASS,
    ]);

    expect($res['status'])->toBe('error')
        ->and($res['message'])->toContain('Permission denied');
});

test('DELETE /posts/{no} — delete success', function () {
    $created = api('POST', '/posts', [
        'title' => 'Delete Me', 'content' => 'Body',
        'id' => POST_USER, 'password' => POST_PASS,
    ]);
    api('DELETE', '/posts/' . $created['no'], [
        'id' => POST_USER, 'password' => POST_PASS,
    ]);

    $check = api('GET', '/posts/' . $created['no']);
    expect($check['status'])->toBe('error');
});
```

### 7.6 Test Coverage Checklist

For every endpoint write tests for:
1. **Happy path** — valid input, `status: success`
2. **Unauthenticated** — missing `id` / `password`
3. **Missing required fields** — empty title, etc.
4. **Not found** — `no = 99999`
5. **Permission denied** — another user tries to mutate

### 7.7 Running Tests

```bash
# Terminal 1 — start the dev server
php -S 0.0.0.0:8080 index.php

# Terminal 2 — run all tests
./vendor/bin/pest

# Single file
./vendor/bin/pest tests/Feature/PostControllerTest.php
```

---

<!-- QUIZ: Phase 7 — Testing with Pest PHP -->
> 🧪 **QUIZ CHECKPOINT** — Ask Claude to generate a Phase 7 quiz before continuing.

---

## Phase 8 — Flutter Client, Reference & Pitfalls

### 8.1 pubspec.yaml

```yaml
dependencies:
  dio: ^5.0.0
```

### 8.2 ApiService — Base Class

```dart
import 'package:dio/dio.dart';

class ApiService {
  static const String baseUrl = 'http://your-server.com';

  final Dio _dio = Dio(BaseOptions(
    baseUrl: baseUrl,
    contentType: 'application/json',
    responseType: ResponseType.json,
  ));

  Future<Map<String, dynamic>> get(String path) =>
      _call(() => _dio.get(path));

  Future<Map<String, dynamic>> post(String path, [Map<String, dynamic>? body]) =>
      _call(() => _dio.post(path, data: body));

  Future<Map<String, dynamic>> put(String path, [Map<String, dynamic>? body]) =>
      _call(() => _dio.put(path, data: body));

  Future<Map<String, dynamic>> delete(String path, [Map<String, dynamic>? body]) =>
      _call(() => _dio.delete(path, data: body));

  Future<Map<String, dynamic>> _call(Future<Response> Function() fn) async {
    try {
      final response = await fn();
      return response.data as Map<String, dynamic>;
    } on DioException catch (e) {
      return {
        'status': 'error',
        'message': e.response?.data?['message'] ?? e.message,
      };
    }
  }
}
```

### 8.3 UserApiService

```dart
class UserApiService extends ApiService {
  Future<Map<String, dynamic>> register({
    required String id,
    required String password,
    String name = '',
  }) => post('/users', {'id': id, 'password': password, 'name': name});

  Future<Map<String, dynamic>> login({
    required String id,
    required String password,
  }) => post('/users/login', {'id': id, 'password': password});

  Future<Map<String, dynamic>> logout() => post('/users/logout');

  Future<Map<String, dynamic>> listUsers() => get('/users');

  Future<Map<String, dynamic>> getUser(int no) => get('/users/$no');

  Future<Map<String, dynamic>> updateUser({
    required int no,
    required String id,
    required String password,
    String? name,
    String? newPassword,
  }) => put('/users/$no', {
    'id': id, 'password': password,
    if (name != null) 'name': name,
    if (newPassword != null) 'new_password': newPassword,
  });

  Future<Map<String, dynamic>> deleteUser(int no) => delete('/users/$no');
}
```

### 8.4 PostApiService

```dart
class PostApiService extends ApiService {
  Future<Map<String, dynamic>> listPosts() => get('/posts');

  Future<Map<String, dynamic>> getPost(int no) => get('/posts/$no');

  Future<Map<String, dynamic>> createPost({
    required String title,
    required String content,
    required String id,
    required String password,
  }) => post('/posts', {
    'title': title, 'content': content,
    'id': id, 'password': password,
  });

  Future<Map<String, dynamic>> updatePost({
    required int no,
    required String id,
    required String password,
    String? title,
    String? content,
  }) => put('/posts/$no', {
    'id': id, 'password': password,
    if (title != null) 'title': title,
    if (content != null) 'content': content,
  });

  Future<Map<String, dynamic>> deletePost({
    required int no,
    required String id,
    required String password,
  }) => delete('/posts/$no', {'id': id, 'password': password});
}
```

### 8.5 Usage in a Flutter Widget

```dart
final userApi = UserApiService();
final postApi = PostApiService();

// Register and login
final reg = await userApi.register(id: 'alice', password: 'secret123', name: 'Alice');
final login = await userApi.login(id: 'alice', password: 'secret123');

if (login['status'] == 'success') {
  final user = login['user'] as Map<String, dynamic>;

  // Create a post
  final created = await postApi.createPost(
    title: 'Hello World',
    content: 'My first post',
    id: user['id'],
    password: 'secret123',
  );

  // List posts
  final list = await postApi.listPosts();
  final posts = list['posts'] as List;
}
```

### 8.6 Complete Request / Response Reference

All requests use `Content-Type: application/json`.

#### User

```
POST /users
Body: { "id": "alice", "password": "secret123", "name": "Alice" }
→    { "status": "success", "no": 1, "id": "alice", "name": "Alice" }

GET /users
→    { "status": "success", "users": [ ... ] }

GET /users/1
→    { "status": "success", "user": { "no": 1, "id": "alice", "name": "Alice", "created_at": "..." } }

PUT /users/1
Body: { "id": "alice", "password": "secret123", "name": "Alice B", "new_password": "newpass" }
→    { "status": "success", "message": "User updated" }

DELETE /users/1
Body: { "id": "alice", "password": "secret123" }
→    { "status": "success", "message": "User deleted" }

POST /users/login
Body: { "id": "alice", "password": "secret123" }
→    { "status": "success", "user": { "no": 1, "id": "alice", "name": "Alice", "created_at": "..." } }

POST /users/logout
→    { "status": "success", "message": "Logged out" }
```

#### Post

```
POST /posts
Body: { "title": "Hello", "content": "World", "id": "alice", "password": "secret123" }
→    { "status": "success", "no": 1, "user_id": "alice" }

GET /posts
→    { "status": "success", "posts": [ { "no": 1, "title": "Hello", ... } ] }

GET /posts/1
→    { "status": "success", "post": { "no": 1, "title": "Hello", "content": "World", ... } }

PUT /posts/1
Body: { "title": "Updated", "id": "alice", "password": "secret123" }
→    { "status": "success", "message": "Post updated" }

DELETE /posts/1
Body: { "id": "alice", "password": "secret123" }
→    { "status": "success", "message": "Post deleted" }
```

#### Config

```
POST /config/install
→    { "status": "success", "message": "Database installed" }
```

#### Error (any endpoint)

```json
{ "status": "error", "message": "Post not found" }
```

### 8.7 Common Pitfalls

#### 1. Forgetting to update fillable() when adding a column

Repository silently strips unknown fields. Always update `Entity::fillable()` alongside schema changes.

#### 2. Writing SQL outside the Repository

All `$db->prepare()` calls belong in a Repository method. Service and Controller must never touch PDO directly.

#### 3. Not using prepared statements

```php
// WRONG — SQL injection
$stmt = $db->query("SELECT * FROM posts WHERE no = {$input['no']}");

// CORRECT
$stmt = $db->prepare("SELECT * FROM posts WHERE no = :no");
$stmt->execute([':no' => (int)$input['no']]);
```

#### 4. Exposing the password hash

`columns()` excludes `password`. Only `findByIdWithPassword()` fetches it, and `UserService::authenticate()` calls `unset($user['password'])` immediately. Never return the hash in a response.

#### 5. Not casting URL segments to int

URL segments are strings. Always cast: `(int)$input['no']` before passing to `find()` or `delete()`.

#### 6. mkdir in a constructor that is never called

`Database::get()` is static — the constructor is never invoked. Any setup (mkdir, config loading) must be inside `get()`.

#### 7. Relative database path

`./database/database.db` resolves differently in a web server vs CLI vs test runner. Always use `__DIR__`:

```php
static public $path = __DIR__ . '/database/database.db';
```

#### 8. Forgetting `index.php` in the server command

```bash
# WRONG — requests to /posts return 404
php -S 0.0.0.0:8080

# CORRECT — index.php is the router for every request
php -S 0.0.0.0:8080 index.php
```

#### 9. CORS headers missing for Flutter on device

```php
// Top of index.php — before any output
header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
header('Access-Control-Allow-Headers: Content-Type');

if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
    exit(0);
}
```

#### 10. Composer autoload not regenerated after adding files

```bash
composer dump-autoload
```

Without this, `index.php` cannot resolve the new class and throws a fatal error.

---

<!-- QUIZ: Phase 8 — Flutter Client, Reference & Pitfalls -->
> 🧪 **QUIZ CHECKPOINT** — Ask Claude to generate a Phase 8 (final) quiz when you are ready.

---

## Further Reading

- PHP-FIG, *PSR-4: Autoloader Standard* (2013) — https://www.php-fig.org/psr/psr-4/
- Martin Fowler, *Patterns of Enterprise Application Architecture* (2002) — Repository Pattern, p. 322
- Gang of Four, *Design Patterns: Elements of Reusable Object-Oriented Software* (1994) — Template Method, p. 325
- Robert C. Martin, *Agile Software Development, Principles, Patterns, and Practices* (2002) — SOLID Principles
- Robert C. Martin, *Clean Architecture* (2017) — Separation of Concerns, Dependency Rule
