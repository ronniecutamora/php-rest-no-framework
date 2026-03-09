---
name: contracts
description: The full Interface + Abstract Class contract chain for Entity, Repository, Service, and Controller layers in this PHP REST API. Use this skill when the user asks "explain the contract chain", "what interfaces exist", "how do layers connect", "why use an interface AND an abstract class", "what is EntityInterface", "what is ServiceInterface", or asks about DIP, ISP, or the Template Method pattern in this codebase.
---

## When to use this skill

- "Explain the contract chain"
- "What interfaces exist?"
- "How do the layers connect?"
- "Why use an interface AND an abstract class?"
- "What is EntityInterface / RepositoryInterface / ServiceInterface?"
- "How do I implement DIP here?"
- "What does the abstract Repository provide?"

## Dependencies / Prerequisites

- setup skill completed (lib/ directory and autoloader exist)

---

## The Two-Layer Contract Pattern

```
Interface          ← defines WHAT methods must exist (no implementation)
    │
Abstract Class     ← defines HOW they work (default implementation)
    │
Concrete Class     ← defines WHICH table/rules/logic (module-specific)
```

---

## Steps — Create all 7 contract files

### 1. lib/Utils/EntityInterface.php

```php
<?php
namespace Lib\Utils;

interface EntityInterface
{
    public static function tableName(): string;
    public static function columns(): array;   // columns to SELECT — NEVER include password
    public static function fillable(): array;  // INSERT/UPDATE whitelist (mass-assignment protection)
    public static function required(): array;  // must be non-empty on create
}
```

### 2. lib/Utils/Entity.php

```php
<?php
namespace Lib\Utils;

abstract class Entity implements EntityInterface
{
    abstract public static function tableName(): string;
    abstract public static function columns(): array;
    abstract public static function fillable(): array;
    abstract public static function required(): array;
}
```

### 3. lib/Utils/RepositoryInterface.php

```php
<?php
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

### 4. lib/Utils/Repository.php

```php
<?php
namespace Lib\Utils;
use PDO;

abstract class Repository implements RepositoryInterface
{
    abstract protected function tableName(): string;
    abstract protected function columns(): array;
    abstract protected function fillable(): array;
    abstract protected function required(): array;

    public function getRequired(): array { return $this->required(); }

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
        $data         = $this->filterFillable($data);
        $db           = Database::get();
        $keys         = array_keys($data);
        $cols         = implode(', ', $keys);
        $placeholders = implode(', ', array_map(fn($k) => ':' . $k, $keys));
        $params       = [];
        foreach ($data as $k => $v) { $params[':' . $k] = $v; }

        $stmt = $db->prepare("INSERT INTO {$this->tableName()} ({$cols}) VALUES ({$placeholders})");
        $stmt->execute($params);
        return $db->lastInsertId();
    }

    public function update(int $no, array $data): int
    {
        $data   = $this->filterFillable($data);
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

### 5. lib/Utils/ServiceInterface.php

```php
<?php
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

### 6. lib/Utils/Service.php

```php
<?php
namespace Lib\Utils;

abstract class Service implements ServiceInterface
{
    abstract protected function repository(): RepositoryInterface;

    abstract public function listAll(array $input): array;
    abstract public function createEntry(array $input): array;
    abstract public function getEntry(array $input): array;
    abstract public function updateEntry(array $input): array;
    abstract public function deleteEntry(array $input): array;

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

### 7. lib/Utils/Controller.php

```php
<?php
namespace Lib\Utils;

abstract class Controller
{
    abstract protected function service(): ServiceInterface;

    protected function json(array $data): string
    {
        header('Content-Type: application/json');
        return json_encode($data);
    }

    public function list(array $input): string   { return $this->json($this->service()->listAll($input)); }
    public function show(array $input): string   { return $this->json($this->service()->getEntry($input)); }
    public function create(array $input): string { return $this->json($this->service()->createEntry($input)); }
    public function update(array $input): string { return $this->json($this->service()->updateEntry($input)); }
    public function delete(array $input): string { return $this->json($this->service()->deleteEntry($input)); }
}
```

---

## Contract Chain Reference

```
EntityInterface          ← 4 static schema methods
    ↑ implements
  Entity (abstract)      ← all 4 abstract
    ↑ extends
  XxxEntity              ← provides tableName, columns, fillable, required

RepositoryInterface      ← findAll, find, create, update, delete, getRequired
    ↑ implements
  Repository (abstract)  ← full PDO SQL + filterFillable
    ↑ extends
  XxxRepository          ← delegates 4 metadata methods to XxxEntity; custom finders if needed

ServiceInterface         ← listAll, createEntry, getEntry, updateEntry, deleteEntry
    ↑ implements
  Service (abstract)     ← validate() + convenience delegators
    ↑ extends
  XxxService             ← full business logic (auth, ownership, hashing)

Controller (abstract)    ← json() + 5 CRUD actions → ServiceInterface
    ↑ extends
  XxxController          ← service() → XxxService; extra named-route methods if needed
```

---

## Gotchas

- **`columns()` must NEVER include `password`** — it is excluded from default SELECT to prevent accidental exposure. Only `findByIdWithPassword()` fetches it.

- **`fillable()` is the mass-assignment whitelist** — any field not listed is silently stripped by `filterFillable()` before INSERT/UPDATE. Always keep it up to date when adding columns.

- **Service depends on `RepositoryInterface`, not on a concrete repo class** — this is DIP. It allows the Service to be tested with a mock repository.

- **Abstract `Repository` provides all SQL** — concrete repos only implement the 4 metadata methods. Never duplicate SQL logic in a concrete repo; add a custom finder method instead.

- **`validate()` in Service checks `required()` fields** — it is called inside `Service::create()`. Do not call it again manually in the concrete Service unless validating additional constraints.
