# PHP REST API — Skills & Agents Index

This project is a framework-free PHP 8.3.8+ REST API with SQLite, PSR-4 autoloading, and a 4-layer architecture:

```
index.php → Controller → Service → Repository → Database (PDO/SQLite)
```

---

## Subagent Map

Subagents in `.claude/agents/` autonomously read, write, and edit files for their domain.

| If the user says… | Use agent |
|---|---|
| "Set up the project", "initialize the database", "start a new PHP REST API", "Database class", ".htaccess", "CORS", "composer.json" | **setup-expert** |
| "Explain the contract chain", "what interfaces exist", "how do layers connect", "EntityInterface", "RepositoryInterface", "ServiceInterface", "DIP", "Template Method" | **contracts-expert** |
| "Add a new resource", "create a [X] module", "add [X] endpoint", "new CRUD", "add Comment", "add Category" | **module-builder** |
| "How does routing work", "add a named route", "custom endpoint", "/users/login resolution", "unknown resource error", "how is the controller resolved" | **router-expert** |
| "Write tests for", "add test coverage", "create a test file", "how do I test this endpoint", "api() helper", "beforeAll", "Pest PHP setup" | **test-writer** |

---

## Skill Map

Skills in `.claude/skills/` provide reference documentation and step-by-step runbooks.

| If the user says… | Use skill |
|---|---|
| "Set up the project", "initialize the database", "start a new PHP REST API", "Database class", ".htaccess", "CORS", "composer.json" | **setup** |
| "Explain the contract chain", "what interfaces exist", "how do layers connect", "EntityInterface", "RepositoryInterface", "ServiceInterface", "DIP", "Template Method" | **contracts** |
| "Add a new resource", "create a [X] module", "add [X] endpoint", "new CRUD", "add Comment", "add Category" | **module** |
| "How does routing work", "add a named route", "custom endpoint", "/users/login resolution", "unknown resource error", "how is the controller resolved" | **routing** |
| "Write tests for", "add test coverage", "create a test file", "how do I test this endpoint", "api() helper", "beforeAll", "Pest PHP setup" | **testing** |

---

## Project Architecture

```
project-root/
├── index.php              # Front controller — all requests enter here
├── .htaccess              # Apache rewrite: all URLs → index.php
├── composer.json          # PSR-4: "Lib\\": "lib/"
├── lib/
│   ├── Utils/
│   │   ├── EntityInterface.php
│   │   ├── RepositoryInterface.php
│   │   ├── ServiceInterface.php
│   │   ├── Entity.php               # abstract
│   │   ├── Repository.php           # abstract — all SQL lives here
│   │   ├── Service.php              # abstract — validate() + delegators
│   │   ├── Controller.php           # abstract — json() + 5 CRUD actions
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
│       └── ConfigController.php     # POST /config/install
└── tests/
    ├── Pest.php                     # bootstrap + api() helper
    ├── TestCase.php
    └── Feature/
        ├── UserControllerTest.php
        └── PostControllerTest.php
```

---

## Critical Rules (Always Apply)

1. `__DIR__` for DB path — never `./`
2. `mkdir` inside `Database::get()` — never in a constructor
3. `password` excluded from `columns()` — never returned in responses
4. `user_id` set from authenticated user — never from raw client input
5. SQL only inside Repository methods — never in Service or Controller
6. Always `composer dump-autoload` after adding new class files
7. Always `api('POST', '/config/install')` in `beforeAll()` in tests
8. URL resources are plural (`/users`, `/posts`) — router strips trailing `s` to build class name
9. CORS headers at the very top of `index.php` — before any output

---

## REST Routing Convention

| HTTP Method | URL | Action | Controller method |
|---|---|---|---|
| GET | `/resource` | list | `list` |
| GET | `/resource/{no}` | show | `show` |
| POST | `/resource` | create | `create` |
| PUT / PATCH | `/resource/{no}` | update | `update` |
| DELETE | `/resource/{no}` | delete | `delete` |
| POST | `/resource/action` | named route | `action` (method override) |

**There is no `method` field in the request body.** HTTP verb + URL path determines everything.
