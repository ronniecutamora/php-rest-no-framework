# PHP REST API — Subagent Index

Subagents for this framework-free PHP 8.3.8+ REST API. Each agent is a domain specialist that can read, write, and edit files autonomously.

```
index.php → Controller → Service → Repository → Entity → Database (PDO/SQLite)
```

---

## Agent Map

| Agent | File | Trigger phrases |
|---|---|---|
| **module-builder** | `module-builder.md` | "add a new resource", "create a [X] module", "add [X] endpoint", "new CRUD resource", "add Comment module", "create Category feature" |
| **test-writer** | `test-writer.md` | "write tests for", "add test coverage", "create a test file", "how do I test this endpoint", "add [Resource]ControllerTest", "api() helper" |
| **router-expert** | `router-expert.md` | "how does routing work", "add a named route", "custom endpoint", "/users/login resolution", "unknown resource error", "how is the controller resolved" |
| **contracts-expert** | `contracts-expert.md` | "explain the contract chain", "what interfaces exist", "how do layers connect", "EntityInterface", "RepositoryInterface", "ServiceInterface", "DIP", "Template Method" |
| **setup-expert** | `setup-expert.md` | "set up the project", "initialize the database", "start a new PHP REST API", "Database class", ".htaccess", "CORS headers", "composer.json", "database file not found" |

---

## Agent Responsibilities

| Agent | Owns | Does NOT touch |
|---|---|---|
| `module-builder` | `lib/[Resource]/` (all 4 files) + `lib/Config/ConfigController.php` install block | `index.php`, `lib/Utils/` |
| `test-writer` | `tests/Feature/[Resource]ControllerTest.php` | Production code |
| `router-expert` | `index.php` routing logic | Service/Repository/Entity layers |
| `contracts-expert` | `lib/Utils/` (all 7 contract files) | Module-specific files |
| `setup-expert` | `composer.json`, `index.php` skeleton, `.htaccess`, `lib/Utils/Database.php` | Business logic layers |

---

## Dependency Order

When setting up a new project, use agents in this order:

```
setup-expert → contracts-expert → module-builder → test-writer
```

When adding a named route to an existing module:

```
router-expert → (module-builder if new service method needed)
```

---

## Critical Rules (All Agents Must Follow)

1. `__DIR__` for DB path — never `./`
2. `mkdir` inside `Database::get()` — never in a constructor
3. `password` excluded from `columns()` — never returned in responses
4. `user_id` set from authenticated user — never from raw client input
5. SQL only inside Repository methods — never in Service or Controller
6. Always `composer dump-autoload` after adding new class files
7. Always `api('POST', '/config/install')` in `beforeAll()` in tests
8. URL resources are plural (`/users`, `/posts`) — router strips trailing `s` to build class name
9. CORS headers at the very top of `index.php` — before any output
10. Throw `\InvalidArgumentException` for business errors — caught by `index.php` and returned as HTTP 400 JSON
