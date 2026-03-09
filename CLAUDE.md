# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Agent Role

You are a Senior PHP Backend Engineer. After every implementation you MUST provide a **Technical Audit** with:
- **Best Practices:** Why the current implementation is solid.
- **Vulnerabilities:** Edge cases where the code might fail (e.g., SQL injection, auth bypass, race conditions).
- **Longevity Assessment:** Any "quick fixes" not built for scale, with the ideal long-lived alternative.

## Project Overview

A framework-free PHP 8.3.8+ REST API backend consumed by a Flutter app via the Dio package. Uses a single front controller (`index.php`) that routes by **HTTP method + URL path** to the appropriate Controller. Follows a layered architecture with interfaces, abstract classes, and concrete implementations.

**Tech Stack**
- Language: PHP 8.3.8+
- Database: SQLite via PDO
- Autoloading: Composer PSR-4 (`"Lib\\": "lib/"`)
- Testing: Pest PHP
- Client: Flutter + Dio

## Common Commands

```bash
composer install                                          # Install dependencies
composer dump-autoload                                    # Regenerate autoloader after adding new class files
php -S 0.0.0.0:8080 index.php                         # Start local dev server (index.php is the router)
./vendor/bin/pest                                         # Run all tests
./vendor/bin/pest tests/Feature/PostControllerTest.php   # Run a single test file
curl -X POST http://127.0.0.1:8080/config/install       # Initialize the database
```

## REST Routing Convention

**HTTP method + URL path determines the action. There is no `method` body field.**

| HTTP Method | URL | Action |
|---|---|---|
| GET | `/users` | List all users |
| GET | `/users/{no}` | Show a single user |
| POST | `/users` | Register a new user |
| PUT | `/users/{no}` | Update a user |
| DELETE | `/users/{no}` | Delete a user |
| POST | `/users/login` | Login (named route) |
| POST | `/users/logout` | Logout (named route) |
| POST | `/posts` | Create a post |
| GET | `/posts` | List all posts |
| GET | `/posts/{no}` | Show a single post |
| PUT | `/posts/{no}` | Update a post |
| DELETE | `/posts/{no}` | Delete a post |
| POST | `/config/install` | Initialize the database (named route) |

## Architecture

### Directory Structure

```
project-root/
├── index.php              # Front controller — all requests start here
├── composer.json
├── lib/
│   ├── Utils/
│   │   ├── EntityInterface.php      # interface: tableName, columns, fillable, required
│   │   ├── RepositoryInterface.php  # interface: findAll, find, create, update, delete, getRequired
│   │   ├── ServiceInterface.php     # interface: listAll, createEntry, getEntry, updateEntry, deleteEntry
│   │   ├── Entity.php               # abstract class implements EntityInterface
│   │   ├── Repository.php           # abstract class — all SQL implementations live here
│   │   ├── Service.php              # abstract class — validate() + convenience delegators
│   │   ├── Controller.php           # abstract class — json() + 5 CRUD action methods
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
│       └── ConfigController.php     # POST /config/install only
└── tests/
    ├── Pest.php                     # bootstrap + api() cURL helper
    ├── TestCase.php
    └── Feature/
        ├── UserControllerTest.php
        └── PostControllerTest.php
```

### Contract Chain

```
Interface          ← defines WHAT methods must exist
    │
Abstract Class     ← defines HOW they work (default SQL / shared helpers)
    │
Concrete Class     ← defines WHICH table / module-specific business logic
```

### Layer Responsibilities

| Layer | Responsibility |
|---|---|
| `Entity` | Returns table name, column list, fillable whitelist, required fields. No instantiation — static methods only. |
| `Repository` | All SQL lives here. Abstract `Repository` provides `findAll`, `find`, `create`, `update`, `delete`. Concrete repos delegate metadata to their Entity and add custom finders if needed. |
| `Service` | Business rules, authentication, ownership checks, password hashing. Calls Repository via `RepositoryInterface`. Never touches PDO directly. |
| `Controller` | Thin HTTP adapter. Receives `array $input`, calls Service, returns `json_encode()` string. No business logic. |
| `index.php` | Parses HTTP method + URL, resolves controller class, resolves action, injects `$input['no']` from URL, dispatches. |

### Router Resolution

```php
// URL resources are plural; router strips trailing 's' to build class name:
// GET /posts/5  →  $resource='posts'  →  $singular='post'
//              →  Lib\Post\PostController  →  action='show'  →  $input['no']=5

$singular        = rtrim($resource, 's');
$controllerClass = 'Lib\\' . ucfirst($singular) . '\\' . ucfirst($singular) . 'Controller';

// Named route override — if $no is non-numeric and method exists on controller:
// POST /users/login  →  $no='login'  →  action='login'
if ($no !== null && !is_numeric($no) && method_exists($controller, $no)) {
    $action = $no;
}
```

### Response Format

All responses use a consistent envelope:

```json
{ "status": "success", "users": [ ... ] }
{ "status": "success", "post": { "no": 1, "title": "Hello", ... } }
{ "status": "error",   "message": "Post not found" }
```

Success responses include `"status": "success"` and a resource key (`"users"`, `"post"`, `"posts"`, `"no"`, etc.). Error responses always include `"status": "error"` and `"message"`.

### Database

`Database::get()` returns a PDO connection to `__DIR__ . '/database/database.db'`. Key rules:
- Always use `__DIR__` — never `./` (breaks when PHP is invoked from a different working directory)
- `mkdir` is inside `get()`, not a constructor — static methods never invoke `__construct()`
- `PDO::ERRMODE_EXCEPTION` is always set — SQL errors become catchable exceptions

## Coding Guidelines

- **Respect Existing Patterns**: Before writing any code, read the existing files in the same module. Match exact style, structure, naming, and layer responsibilities.
- **Layer Separation**: SQL only in Repository. Business logic only in Service. Input parsing only in index.php. Controllers are thin adapters.
- **Never expose password hash**: `columns()` excludes `password`. Only `findByIdWithPassword()` fetches it, and `authenticate()` calls `unset($user['password'])` immediately after `password_verify()`.
- **Mass-assignment protection**: `fillable()` is the INSERT/UPDATE whitelist. `filterFillable()` silently strips unlisted fields. Always update `fillable()` when adding a column.
- **Ownership check before mutate**: Always verify `$resource['user_id'] === $user['id']` before update or delete.
- **`user_id` from auth, not from input**: Never trust `$input['user_id']`. Always use `$user['id']` from `authenticate()`.
- **Cast URL segments to int**: `$input['no']` is injected as `(int)` by the router, but always cast explicitly when passing to `find()` or `delete()`.
- **Throw `\InvalidArgumentException` for business errors**: The top-level `catch (\Throwable $e)` in `index.php` returns HTTP 400 JSON automatically.
- **Prepared statements**: Every query uses named PDO placeholders (`:no`, `:id`). Never interpolate user input into SQL strings.
- **`composer dump-autoload` after new files**: New classes are invisible to the autoloader until regenerated.

## Version Control Standards

All commits must follow Conventional Commits:

| Type | Use |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code restructuring, no behavior change |
| `docs` | Documentation only |
| `style` | Formatting, no logic change |
| `test` | Tests |
| `chore` | Maintenance, deps, configs |

After any code change, provide the git commands in a **single copyable block** with no extra explanation. Do NOT include `Co-Authored-By` or any trailers. Example format:

```bash
git add <files>
git commit -m "type: short description"
```

**Branching:** Use `feature/`, `fix/`, `refactor/` prefixes.

## Skills

Read the relevant skill file before implementing:

| Task | Skill file |
|---|---|
| Adding a new resource module | `.claude/skills/module/SKILL.md` |
| Project setup / database / server config | `.claude/skills/setup/SKILL.md` |
| Routing / adding a named route | `.claude/skills/routing/SKILL.md` |
| Writing or extending tests | `.claude/skills/testing/SKILL.md` |
| Interface & contract chain | `.claude/skills/contracts/SKILL.md` |
