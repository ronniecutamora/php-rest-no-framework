---
name: setup
description: Project bootstrap for a framework-free PHP 8.3.8+ REST API. Use this skill when the user says "set up the project", "initialize the database", "start a new PHP REST API", "bootstrap the project", or asks about the Database class, CORS headers, or composer.json setup.
---

## When to use this skill

- "Set up the project"
- "Initialize the database"
- "Start a new PHP REST API"
- "Create the Database class"
- "Configure composer.json / PSR-4"
- "Add CORS headers"
- "Why is the database file not found?"

## Dependencies / Prerequisites

- PHP 8.3.8+
- Composer installed globally

---

## Steps

### 1. Initialize Composer

```bash
composer init
```

When prompted, skip `name`, `type`, `authors` — they are optional. The only required output is `composer.json`.

### 2. Set up composer.json

Replace the generated file with:

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

### 3. Install dependencies and generate autoloader

```bash
composer install
```

### 4. Create the Database class

```php
<?php
// lib/Utils/Database.php
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

### 5. Create index.php — add built-in server router guard at the very top

```php
<?php
// Built-in server: serve real static files directly, route everything else through index.php
if (PHP_SAPI === 'cli-server' && is_file(__DIR__ . $_SERVER['REQUEST_URI'])) {
    return false;
}

require_once __DIR__ . '/vendor/autoload.php';

// CORS — must appear before any other output
header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
header('Access-Control-Allow-Headers: Content-Type');

if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
    exit(0);
}

header('Content-Type: application/json');

// ... routing logic follows (see routing skill)
```

### 7. Initialize the database schema

After deploying `index.php` and `ConfigController`, call:

```bash
curl -X POST http://localhost/config/install
```

Or during development:

```bash
php -S 0.0.0.0:8080 index.php
curl -X POST http://127.0.0.1:8080/config/install
```

### 8. Regenerate autoloader after adding any new class file

```bash
composer dump-autoload
```

---

## Gotchas

- **Never use `./database/database.db`** — `./` resolves to the current working directory, which changes depending on how PHP is invoked (web server, CLI, test runner). Always use `__DIR__ . '/database/database.db'`.

- **`mkdir` must be inside `Database::get()`, not a constructor** — `get()` is a static method. PHP never calls `__construct()` when you call `Database::get()`. Any setup logic must live inside the static method itself.

- **`PDO::ERRMODE_EXCEPTION` must always be set** — without it, PDO silently swallows SQL errors and returns `false`, making bugs invisible.

- **CORS headers must appear before any output** — headers cannot be sent after output has started. Put them at the very top of `index.php`, before `require_once` or any echo.

- **Run `composer dump-autoload` after adding new class files** — Composer's autoloader will not find new classes until the classmap is regenerated.
