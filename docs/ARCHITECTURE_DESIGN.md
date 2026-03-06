# Architecture & Design

Solid architectural patterns make code scalable, testable, and maintainable.

---

## 1. Hexagonal Architecture (Ports & Adapters)

Organize code into layers: controllers manage input/output, services handle business logic, repositories deal with database interactions.

**Benefits:**
- Clear separation of concerns
- Easy to test (swap implementations)
- Easy to change databases
- Business logic doesn't depend on frameworks

**Pattern:**
```
Controller (input/output)
    ↓
Service (business logic)
    ↓
Repository (data persistence)
```

**Example:**
```php
// ❌ BAD: Controller mixed with logic
class UserController {
    public function create() {
        $user = new User($_POST['name'], $_POST['email']);
        DB::insert('users', $user->toArray());
        Mail::send($user->email, 'Welcome!');
        return response('User created');
    }
}

// ✅ GOOD: Clear separation
class UserController {
    public function __construct(private CreateUserService $service) {}
    
    public function create(CreateUserRequest $request) {
        $user = $this->service->execute($request->toCommand());
        return response('User created', 201);
    }
}

class CreateUserService {
    public function execute(CreateUserCommand $command): User {
        $user = User::create($command);
        $this->userRepository->save($user);
        $this->emailService->sendWelcome($user);
        return $user;
    }
}

class UserRepository {
    public function save(User $user): void {
        DB::insert('users', $this->hydrate($user));
    }
    
    private function hydrate(User $user): array {
        return ['name' => $user->name, 'email' => $user->email];
    }
}
```

---

## 2. DTOs (Data Transfer Objects)

Use DTOs when crossing boundaries (API, service, or persistence layers) to control what data is exposed and prevent leaking internal implementation details.

**Example:**
```php
// ❌ BAD: Exposing internal domain object directly
class UserRepository {
    public function findById(int $id): UserModel {}
}

// ✅ GOOD: Using DTOs to control boundaries
class UserDTO {
    public function __construct(
        public readonly int $id,
        public readonly string $name,
        public readonly string $email,
        public readonly string $role,
    ) {}
}

// Repository returns DTOs for queries
class UserRepository {
    public function findById(int $id): ?UserDTO {
        $row = DB::query("SELECT id, name, email, role FROM users WHERE id = ?", [$id]);
        
        if (!$row) {
            return null;
        }
        
        return new UserDTO(
            name: $row['name'],
            email: $row['email'],
            role: $row['role'],
        );
    }
}

// Usage - clean and simple
$userDTO = $this->userRepository->findById(1);
echo $userDTO->name; // Just read data, can't modify
```

**Example - Mixed Approach:**
```php
class UserRepository {
    // Return DTO for read-only queries
    public function findForDisplay(int $id): ?UserDTO {
        $row = DB::query("SELECT id, name, email, role FROM users WHERE id = ?", [$id]);
        return $row ? new UserDTO(...) : null;
    }
    
    // Return Model for business operations
    public function findForUpdate(int $id): ?User {
        $row = DB::query("SELECT * FROM users WHERE id = ?", [$id]);
        return $row ? $this->hydrateModel($row) : null;
    }
    
    private function hydrateModel(array $row): User {
        return new User(
            id: $row['id'],
            name: $row['name'],
            email: $row['email'],
            password: $row['password'],
            status: $row['status'],
            // ... all domain properties
        );
    }
}

// Usage
// For display - use DTO
$userDTO = $this->userRepository->findForDisplay($id);
return view('user.profile', ['user' => $userDTO]);

// For business logic - use Model
$user = $this->userRepository->findForUpdate($id);
$user->suspend(); // Domain method
$this->userRepository->save($user);
```

**Rule of thumb:** Return DTOs from repositories when the caller only needs to READ data. Return Models when the caller needs to MODIFY or perform domain operations.

---

## 3. Dependency Injection (DI)

Leverage DI for automatic object creation and easier testing.

**Example:**
```php
// ❌ BAD: Hard-coded dependencies
class UserService {
    private $db;
    
    public function __construct() {
        $this->db = new Database(); // Hard-coded - can't test or change
    }
    
    public function getUser($id) {
        return $this->db->query("SELECT * FROM users WHERE id = $id");
    }
}

// Usage
$service = new UserService();

// ✅ GOOD: Injected dependencies
class UserService {
    private UserRepository $repository;
    
    public function __construct(UserRepository $repository) {
        $this->repository = $repository;
    }
    
    public function getUser(int $id): User {
        return $this->repository->findById($id);
    }
}

// Usage - container wires it
$service = $container->get(UserService::class);

// Or in tests, inject a fake
$fakeRepo = new InMemoryUserRepository();
$service = new UserService($fakeRepo);
$user = $service->getUser(1);  // Uses fake, not real database
```

**Benefits:**
- ✅ Injected dependencies are easier to test
- ✅ Easy to swap implementations
- ✅ No hidden coupling
- ✅ Clear dependencies at a glance

---

## 4. Repositories & Hydration

Repositories must hide persistence implementation details and return domain objects, not persistence models. Hydrate models carefully.

**Rules:**
- Never return raw database records
- Hydrate models with only domain data
- Repository methods describe business operations, not queries

**Example:**
```php
// ❌ BAD: Returning database array
class UserRepository {
    public function find($id): array {
        return DB::query("SELECT * FROM users WHERE id = ?", [$id]);
    }
}

// ✅ GOOD: Return domain objects, hide implementation
class UserRepository {
    public function find($id): ?User {
        $row = DB::query("SELECT * FROM users WHERE id = ?", [$id]);
        return $row ? $this->hydrate($row) : null;
    }
    
    private function hydrate(array $row): User {
        return new User(
            id: $row['id'],
            name: $row['name'],
            email: $row['email'],
        );
    }
}

// Business-focused method names
class OrderRepository {
    public function findByCustomer(int $customerId): array {
        return array_map(
            fn($row) => $this->hydrate($row),
            DB::query("SELECT * FROM orders WHERE customer_id = ?", [$customerId])
        );
    }
}
```

---

## 5. Interfaces for Repositories

Define interfaces for repositories to decouple usage from implementation and simplify transitions.

**Example:**
```php
// ✅ GOOD: Program against interface
interface UserRepositoryInterface {
    public function find(int $id): ?User;
    public function save(User $user): void;
}

class UserRepository implements UserRepositoryInterface {
    public function find(int $id): ?User { /* ... */ }
    public function save(User $user): void { /* ... */ }
}

class CreateUserService {
    public function __construct(private UserRepositoryInterface $repository) {}
}

// Easy to swap implementation
class InMemoryUserRepository implements UserRepositoryInterface {
    private array $users = [];
    
    public function find(int $id): ?User {
        return $this->users[$id] ?? null;
    }
    
    public function save(User $user): void {
        $this->users[$user->id] = $user;
    }
}
```

---

## 6. Immutability

Domain objects should be immutable by default. State changes must happen through explicit methods that return new instances.

**Example:**
```php
// ❌ BAD: Mutable state is dangerous
class Product {
    public $name;
    public $price;
    public $quantity;
    
    public function __construct($name, $price, $quantity) {
        $this->name = $name;
        $this->price = $price;
        $this->quantity = $quantity;
    }
}

// Easy to corrupt state
$product->price = -100;
$product->quantity = 0;

// ✅ GOOD: Immutable objects with explicit transitions
class Product {
    private readonly string $id;
    private readonly string $name;
    private readonly float $price;
    private readonly int $quantity;
    
    public function __construct(string $id, string $name, float $price, int $quantity) {
        $this->id = $id;
        $this->name = $name;
        $this->price = $price;
        $this->quantity = $quantity;
    }
    
    public function getId(): string { return $this->id; }
    public function getName(): string { return $this->name; }
    public function getPrice(): float { return $this->price; }
    public function getQuantity(): int { return $this->quantity; }
    
    public function applyDiscount(float $percent): self {
        return new self(
            $this->id,
            $this->name,
            $this->price * (1 - $percent / 100),
            $this->quantity
        );
    }
    
    public function reduceStock(int $amount): self {
        if ($amount > $this->quantity) {
            throw new InvalidArgumentException('Not enough stock');
        }
        return new self(
            $this->id,
            $this->name,
            $this->price,
            $this->quantity - $amount
        );
    }
}

// Safe - must use methods
$discounted = $product->applyDiscount(10);
$updated = $product->reduceStock(5);
```

**Benefits:**
- ✅ Immutable objects are predictable
- ✅ Can't accidentally corrupt state
- ✅ Changes are explicit and intentional
- ✅ Thread-safe
- ✅ Easier to reason about
---

## 7. Status Machines (State Pattern)

If business rules depend on status and transitions are restricted, model them explicitly, not as strings or flags.

**When to use:**
- Limited set of states
- Restricted transitions
- Status-dependent behavior

**Example:**
```php
// ❌ BAD: Strings and magic numbers
class Order {
    public $status = 'pending'; // What states are valid?
    
    public function complete() {
        if ($this->status === 'pending') {
            $this->status = 'completed'; // Easy to get wrong
        }
    }
}

// ✅ GOOD: Explicit state machine
enum OrderStatus {
    case PENDING;
    case CONFIRMED;
    case SHIPPED;
    case DELIVERED;
    case CANCELLED;
    
    public function canTransitionTo(OrderStatus $next): bool {
        return match($this) {
            self::PENDING => $next === self::CONFIRMED || $next === self::CANCELLED,
            self::CONFIRMED => $next === self::SHIPPED,
            self::SHIPPED => $next === self::DELIVERED,
            default => false,
        };
    }
}

class Order {
    public function __construct(
        public readonly OrderStatus $status = OrderStatus::PENDING
    ) {}
    
    public function confirm(): self {
        if (!$this->status->canTransitionTo(OrderStatus::CONFIRMED)) {
            throw new InvalidStateTransition("Cannot confirm from {$this->status->name}");
        }
        return new self(OrderStatus::CONFIRMED);
    }
}
```

---

## 8. Enums

Use enums to manage different options instead of strings to get type safety and IDE autocomplete.

**Example:**
```php
// ❌ BAD: Magic strings are error-prone
if ($user->role === 'admin') {
    // What if someone types 'Admin' or 'ADMIN'?
}

if ($order->status === 'pending') {
    // String comparison - easy to typo
}

// Easy to create invalid states
$user->role = 'superuser'; // Typo - but code runs
$order->status = 'invalid_state'; // Typo - but code runs

// ✅ GOOD: Use enums for type safety
enum UserRole: string {
    case ADMIN = 'admin';
    case USER = 'user';
    case GUEST = 'guest';
}

enum OrderStatus: string {
    case PENDING = 'pending';
    case PROCESSING = 'processing';
    case COMPLETED = 'completed';
}

if ($user->role === UserRole::ADMIN) {
    // Type-safe, autocomplete works, IDE helps
}

if ($order->status === OrderStatus::PENDING) {
    // Can't typo enum values
}

class User {
    public function __construct(
        public readonly UserRole $role = UserRole::GUEST
    ) {}
    
    public function delete(): void {
        if ($this->role !== UserRole::ADMIN) {
            throw new PermissionDenied('Only admins can delete');
        }
    }
}

// Safe - must use enum cases
$user = new User(role: UserRole::ADMIN);  // Works
// $user->role = 'superuser';  // Error - can't assign string
```

**Benefits:**
- ✅ No more typos in status values
- ✅ Autocomplete works perfectly
- ✅ Refactoring is safe
- ✅ Type-checked at IDE level

---

## Quick Reference

| Pattern | Problem | Solution |
|---------|---------|----------|
| **Layered Architecture** | Mixed concerns | Separate controllers, services, repositories |
| **DTOs** | Leaking internals | Transfer objects at boundaries |
| **DI** | Hard-coded dependencies | Inject via constructor |
| **Repositories** | Leaking DB details | Hide implementation, return domain objects |
| **Interfaces** | Tight coupling | Program against interfaces |
| **Immutability** | Accidental mutations | Use readonly + factory methods |
| **State Machine** | Invalid state transitions | Model states explicitly |
| **Enums** | String magic | Type-safe enum values |

