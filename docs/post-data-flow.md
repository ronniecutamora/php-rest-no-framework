# Post Feature — Data Flow

End-to-end walkthrough of how a `POST /posts` request travels through every layer.

---

## Request Entry

```
POST /posts
Body: { "title": "Hello", "content": "World", "id": "john", "password": "secret" }
```

---

## Round-Trip Data Flow

```
  CLIENT
    │  POST /posts  {title, content, id, password}
    ▼
  index.php                         parse method+URL → build $input → dispatch
    │  $input array
    ▼
  PostController::create()          [inherited from Controller]
    │  $input array
    ▼
  PostService::createEntry()        authenticate → validate → build $data
    │  {id, password}                                  │  {title, content, user_id}
    ▼                                                  ▼
  UserService::authenticate()       Service::create()  [inherited from Service]
    │  $user or throws                  │  validate required fields
    │  ◄── $user['id'] used here        ▼
    │                              PostRepository::create()  [inherited from Repository]
    │                                  │  filterFillable() via PostEntity::fillable()
    │                                  │  tableName()       via PostEntity::tableName()
    │                                  ▼
    │                              Database::get()  → PDO → SQLite
    │                                  │
    │                                  │  INSERT INTO posts (title, content, user_id)
    │                                  │  VALUES (:title, :content, :user_id)
    │                                  │
    │                                  ▼
    │                              lastInsertId()  →  '7'
    │                                  │
    │                              ◄───┘  $no = '7'
    │
    ◄───────────────────────────────────┘  ['status'=>'success', 'no'=>7, 'user_id'=>'john']
    │
    ▼
  PostController::json()            json_encode()
    │
    ▼
  index.php                         echo response string
    │
    ▼
  CLIENT
    │  HTTP 200
    ▼
  { "status": "success", "no": 7, "user_id": "john" }
```

---

## Layer-by-Layer Breakdown

### 1. index.php — Router

Parses the HTTP method and URL, builds `$input`, resolves the controller class, and dispatches.

```php
// POST /posts  →  resource='posts', no=null, action='create'
$input = ['title' => 'Hello', 'content' => 'World', 'id' => 'john', 'password' => 'secret'];

$controller = new PostController();
echo $controller->create($input);
```

**Owns:** HTTP parsing, controller resolution, action dispatch.
**Does not:** contain any business logic or SQL.

---

### 2. PostController — HTTP Adapter

Wires the service. Inherits all 5 CRUD action methods from abstract `Controller`.

```php
class PostController extends Controller
{
    private PostService $svc;
    public function __construct()                  { $this->svc = new PostService(); }
    protected function service(): ServiceInterface { return $this->svc; }
}
```

`create()` is inherited — `PostController` wrote zero lines for it:

```php
// From abstract Controller:
public function create(array $input): string {
    return $this->json($this->service()->createEntry($input));
}
```

**Owns:** service wiring, JSON encoding.
**Does not:** contain business logic, SQL, or auth.

---

### 3. PostService — Business Logic

Authenticates the caller, validates input, sets `user_id` from auth (never from client), and delegates persistence to the repository.

```php
class PostService extends Service
{
    public function createEntry(array $input): array
    {
        // 1. Authenticate
        $user = $this->userService->authenticate($input['id'] ?? '', $input['password'] ?? '');
        if (!$user) throw new \InvalidArgumentException('Login required');

        // 2. Validate
        if (empty($input['title']) || empty($input['content'])) {
            throw new \InvalidArgumentException('Title and content are required');
        }

        // 3. Persist — user_id from auth, NEVER from $input
        $no = $this->create([
            'title'   => trim($input['title']),
            'content' => trim($input['content']),
            'user_id' => $user['id'],
        ]);

        return ['status' => 'success', 'no' => $no, 'user_id' => $user['id']];
    }
}
```

`$this->create()` is inherited from abstract `Service`, which calls `validate()` then `PostRepository::create()`.

**Owns:** authentication, authorization, ownership rules, business validation.
**Does not:** write SQL or touch PDO directly.

---

### 4. PostRepository — SQL

Delegates table metadata to `PostEntity`. Inherits all SQL (`findAll`, `find`, `create`, `update`, `delete`) from abstract `Repository`.

```php
class PostRepository extends Repository
{
    protected function tableName(): string { return PostEntity::tableName(); }
    protected function columns(): array    { return PostEntity::columns(); }
    protected function fillable(): array   { return PostEntity::fillable(); }
    protected function required(): array   { return PostEntity::required(); }
}
```

`create()` is inherited — `PostRepository` wrote zero SQL for it:

```php
// From abstract Repository:
public function create(array $data): string
{
    $data = $this->filterFillable($data); // strips fields not in fillable()
    // Builds: INSERT INTO posts (title, content, user_id) VALUES (:title, :content, :user_id)
    $stmt->execute($params);
    return $db->lastInsertId();           // e.g. '7'
}
```

**Owns:** SQL queries, mass-assignment filtering.
**Does not:** contain business logic or auth.

---

### 5. PostEntity — Metadata

Pure static metadata. Never instantiated. Answers questions about the `posts` table.

```php
class PostEntity extends Entity
{
    public static function tableName(): string { return 'posts'; }
    public static function columns(): array    { return ['no', 'title', 'content', 'user_id', 'created_at', 'updated_at']; }
    public static function fillable(): array   { return ['title', 'content', 'user_id', 'updated_at']; }
    public static function required(): array   { return ['title', 'content', 'user_id']; }
}
```

**Owns:** table name, column list, INSERT/UPDATE whitelist, required field list.
**Does not:** execute any code, hold state, or get instantiated.

---

## What Each Class Wrote vs Inherited

| Class | Wrote | Got for free |
|---|---|---|
| `PostEntity` | 4 static metadata methods | — |
| `PostRepository` | 4 metadata delegation methods | `findAll`, `find`, `create`, `update`, `delete` |
| `PostService` | `listAll`, `createEntry`, `getEntry`, `updateEntry`, `deleteEntry` | `validate`, `findAll`, `find`, `create`, `update`, `delete` |
| `PostController` | constructor + `service()` | `list`, `show`, `create`, `update`, `delete` |

---

## Interface Role

Neither `PostService` nor `PostController` import a concrete class directly — they depend on interfaces:

```php
// PostService depends on the interface, not the concrete PostRepository:
abstract class Service {
    abstract protected function repository(): RepositoryInterface;
}

// PostController depends on the interface, not the concrete PostService:
abstract class Controller {
    abstract protected function service(): ServiceInterface;
}
```

This means the SQL implementation (`PostRepository`) can be swapped or mocked without touching `PostService` or `PostController`.

---

## Response

```json
{ "status": "success", "no": 7, "user_id": "john" }
```
