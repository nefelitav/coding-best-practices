# Code Organization

Write clear, readable, and maintainable code through consistent organization and naming conventions.

---

## 1. Single Responsibility Principle (SRP)

Each function should have one clear responsibility for better maintainability.

**Example:**
```php
// ❌ BAD: Function does everything
function processOrder($orderId) {
    $order = DB::find($orderId);
    $order->status = 'processing';
    DB::save($order);
    
    Mail::send($order->customer->email, 'Order Processing');
    
    Inventory::reduce($order->items);
    
    Log::info('Order processed: ' . $orderId);
}

// ✅ GOOD: Each function has one job
function processOrder(int $orderId): void {
    $order = $this->updateOrderStatus($orderId, 'processing');
    $this->notifyCustomer($order);
    $this->updateInventory($order);
    $this->logOrderProcessed($orderId);
}

private function updateOrderStatus(int $orderId, string $status): Order {
    $order = DB::find($orderId);
    $order->status = $status;
    DB::save($order);
    return $order;
}

private function notifyCustomer(Order $order): void {
    Mail::send($order->customer->email, 'Order Processing');
}

private function updateInventory(Order $order): void {
    Inventory::reduce($order->items);
}

private function logOrderProcessed(int $orderId): void {
    Log::info('Order processed: ' . $orderId);
}
```

---

## 2. Function/Method Ordering

Arrange functions and calls in the order they run to improve readability. It's not a very strict rule, but it can help readers follow the flow of execution.

**Pattern:**
```php
class OrderService {
    // Public interface first
    public function process(Order $order): void {
        $this->validate($order);
        $this->calculate($order);
        $this->persist($order);
        $this->notify($order);
    }
    
    // Private helpers in order of use
    private function validate(Order $order): void { /* ... */ }
    private function calculate(Order $order): void { /* ... */ }
    private function persist(Order $order): void { /* ... */ }
    private function notify(Order $order): void { /* ... */ }
}
```

---

## 3. Extract Complex Conditions into Variables

Extract complex if conditions into variables for readability.

**Example:**
```php
// ❌ BAD: Hard to understand
if ($user->isActive() && !$user->isDeleted() && $user->hasPermission('edit') && $user->getSubscription()->isValid() && $user->getSubscription()->isExpired() === false) {
    // Update user
}

// ✅ GOOD: Clear intent
$isActive = $user->isActive();
$isNotDeleted = !$user->isDeleted();
$canEdit = $user->hasPermission('edit');
$hasValidSubscription = $user->getSubscription()->isValid() && !$user->getSubscription()->isExpired();

if ($isActive && $isNotDeleted && $canEdit && $hasValidSubscription) {
    // Update user
}

// ✅ EVEN BETTER: Extract to helper method
if ($this->userCanUpdate($user)) {
    // Update user
}

private function userCanUpdate(User $user): bool {
    return $user->isActive()
        && !$user->isDeleted()
        && $user->hasPermission('edit')
        && $user->hasValidSubscription();
}
```

---

## 4. Boolean Variable Naming

Start boolean variables with `is`, `has`, `can`, or `should` to clearly indicate they're booleans.

**Example:**
```php
// ❌ BAD: Unclear what these are
$active = true;
$admin = false;

// ✅ GOOD: Clear boolean indicators
$isActive = true;
$isAdmin = false;

$hasPermission = true;
$canDelete = false;
$shouldNotify = true;
```

---

## 5. Named Parameters

Use named parameters to clarify what each argument represents. Makes code self-documenting.

**Example:**
```php
// ❌ BAD: What does each argument do?
updateInvoice($id, 100, 2, 3, true, 'pending');

// ✅ GOOD: Named parameters
updateInvoice(
    invoiceId: $id,
    amount: 100,
    taxPercent: 2,
    discountPercent: 3,
    isUrgent: true,
    status: 'pending',
);
```

---

## 7. Import Organization

Always import packages at the top of your files for clarity and to follow conventions.

**Example:**
```php
// ❌ BAD: No organization
class UserService {
    public function create() {
        $mail = new Illuminate\Mail\Mailer();
        $db = new PDO();
    }
}

// ✅ GOOD: Clear imports
use App\Models\User;
use App\Repositories\UserRepository;
use Illuminate\Mail\Mailer;
use Illuminate\Support\Facades\DB;

class UserService {
    public function __construct(
        private UserRepository $repository,
        private Mailer $mailer,
    ) {}
}
```

---

## 8. Comments

Write code that's clear enough that you don't need comments. Comments should explain **why**, not **what**.

**Example:**
```php
// ❌ BAD: Obvious comments
$total = $price + $tax; // Add tax to price
if ($age >= 18) { // Check if age is 18 or more
    $isAdult = true;
}

// ✅ GOOD: Comment explains intent/business logic
// We cache expensive calculation for 24 hours to avoid DB overload on high-traffic days
$total = $cache->remember('order_total', now()->addDay(), function() {
    return $this->calculateExpensiveTotal();
});

// Retry on failure because payment gateway can be temporarily unavailable
if ($this->processPayment($order)) {
    // ...
} else {
    // Retry once after 2 seconds
    sleep(2);
    $this->processPayment($order);
}
```

---

## 9. Private Functions

Make functions private if they shouldn't be exposed outside the class. Place private functions at the bottom of the class for better readability.

**Example:**
```php
// ✅ GOOD: Public first, private at bottom
class OrderService {
    public function process(Order $order): void {
        $this->validate($order);
        $this->save($order);
    }
    
    public function cancel(Order $order): void {
        // ...
    }
    
    // Private helpers at the bottom
    private function validate(Order $order): void { /* ... */ }
    
    private function save(Order $order): void { /* ... */ }
}
```

---

## 10. Readonly Properties

Mark class properties as `readonly` to ensure immutability and prevent accidental mutations.

**Example:**
```php
// ❌ BAD: Can be modified
class User {
    public $name;
    public $email;
    
    public function __construct($name, $email) {
        $this->name = $name;
        $this->email = $email;
    }
}
$user->name = 'Hacker'; // Oops, changed

// ✅ GOOD: Immutable
class User {
    public function __construct(
        public readonly string $name,
        public readonly string $email,
    ) {}
}
$user->name = 'Hacker'; // Error: Can't modify readonly
```

---

## 11. Final Classes

Use `final` to prevent a class from being extended or subclassed unless there's a specific need to allow inheritance.

**Example:**
```php
// ❌ BAD: Can be extended unexpectedly
class Repository {
    public function find($id) {
        return DB::query("SELECT * FROM users WHERE id = ?", [$id]);
    }
}

// Someone extends it and breaks things
class BrokenRepository extends Repository {
    public function find($id) {
        return cache('user_' . $id); // Inconsistent behavior!
    }
}

// ✅ GOOD: Explicitly prevent extension
final class UserRepository {
    public function find($id) {
        return DB::query("SELECT * FROM users WHERE id = ?", [$id]);
    }
}
// Cannot extend - must use interface or composition
```

---

## 12. Constants

Use named constants instead of magic numbers or strings.

**Example:**
```php
// ❌ BAD: Magic numbers everywhere
if ($user->age >= 18) { /* ... */ }
if ($order->total > 1000) { /* ... */ }
if ($attempts > 3) { /* ... */ }

// ✅ GOOD: Named constants
const ADULT_AGE = 18;
const HIGH_VALUE_ORDER = 1000;
const MAX_LOGIN_ATTEMPTS = 3;

if ($user->age >= self::ADULT_AGE) { /* ... */ }
if ($order->total > self::HIGH_VALUE_ORDER) { /* ... */ }
if ($attempts > self::MAX_LOGIN_ATTEMPTS) { /* ... */ }
```

---

## 13. Break Long Lines

Break long lines at logical points for readability.

**Example:**
```php
// ❌ BAD: Hard to read
$message = 'Dear ' . $user->firstName . ' ' . $user->lastName . ', your order #' . $order->id . ' has been shipped. Expected delivery: ' . $order->expectedDelivery . '. Thank you!';

// ✅ GOOD: Logical breaks
$message = sprintf(
    'Dear %s %s, your order #%d has been shipped. Expected delivery: %s. Thank you!',
    $user->firstName,
    $user->lastName,
    $order->id,
    $order->expectedDelivery,
);

// ✅ GOOD: For method chains
$users = DB::table('users')
    ->where('is_active', true)
    ->where('created_at', '>', now()->subDays(30))
    ->orderBy('created_at', 'desc')
    ->limit(10)
    ->get();
```

---

## 14. Trailing Commas

Use trailing commas in arrays and parameter lists.

**Benefits:**
- Cleaner git diffs
- Easier to add new items
- Fewer merge conflicts

**Example:**
```php
// ❌ BAD: No trailing comma
$config = [
    'host' => 'localhost',
    'port' => 3306,
    'database' => 'mydb' // have to add comma here to add new line
];

// ✅ GOOD: Trailing comma: adding new item is just adding a new line, no edits to existing lines
$config = [
    'host' => 'localhost',
    'port' => 3306,
    'database' => 'mydb',
    'charset' => 'utf8',
];
```

---

## 15. Ternary Operators

Use ternary operators for simple assignments, but extract complex logic to methods or if statements.

**Example:**
```php
// ✅ GOOD: Simple ternary
$status = $isActive ? 'active' : 'inactive';
$label = $isPremium ? 'Premium User' : 'Standard User';

// ❌ BAD: Complex nested ternary (hard to read)
$label = $isPremium ? ($isVip ? 'VIP Premium' : 'Regular Premium') : ($isTrialUser ? 'Trial' : 'Inactive');

// ✅ GOOD: Extract to method
$label = $this->determinUserLabel($isPremium, $isVip, $isTrialUser);
```

---

## 16. Nullsafe Operator

Use the nullsafe operator (`?->`) to simplify conditional returns when accessing properties on nullable objects.

**Example:**
```php
// ❌ BAD: Nested null checks
$email = null;
if ($user !== null && $user->profile !== null) {
    $email = $user->profile->email;
}

// ✅ GOOD: Nullsafe operator
$email = $user?->profile?->email;

// Useful in operations
$hasEmail = strlen($user?->profile?->email) > 0;
sendEmail($user?->email ?? 'default@example.com');
```

---

## 17. Nullable vs Required Fields

Decide carefully if a field should be nullable or required. Make it explicit.

**Example:**
```php
// ❌ BAD: Unclear intent
class User {
    public ?string $phone;
    public ?string $address;
}

// ✅ GOOD: Explicit decision
class User {
    // Middle name is optional - some users don't have it
    public ?string $middleName;
    
    // Email is always required
    public string $email;
    
    // Phone can be optionally provided
    public ?string $phone;
    
    // Address is required
    public string $address;
}
```

---

## 18. Hardcoding

Avoid hardcoding to keep code flexible and easier to maintain.

**Example:**
```php
// ❌ BAD: Hardcoded everywhere
class EmailService {
    public function sendWelcome($email) {
        Mail::send(
            to: $email,
            from: 'noreply@example.com', // HARDCODED
            subject: 'Welcome!', // HARDCODED
        );
    }
}

// ✅ GOOD: Use configuration
class EmailService {
    public function __construct(private Config $config) {}
    
    public function sendWelcome($email) {
        Mail::send(
            to: $email,
            from: $this->config->get('mail.from'),
            subject: $this->config->get('mail.welcome_subject'),
        );
    }
}

// config/mail.php
return [
    'from' => env('MAIL_FROM', 'noreply@example.com'),
    'welcome_subject' => 'Welcome to our platform!',
];
```

---

## 19. Custom Exceptions

Create specific exception types for different error conditions instead of generic ones.

**Example:**
```php
// ❌ BAD: Generic exceptions
class UserService {
    public function find($id) {
        $user = DB::query("SELECT * FROM users WHERE id = ?", [$id]);
        if (!$user) {
            throw new Exception('Not found'); // What's not found? User? Order?
        }
        return $user;
    }
}

// ✅ GOOD: Specific exceptions
class UserNotFound extends Exception {}
class InvalidEmailFormat extends Exception {}
class UserAlreadyExists extends Exception {}

class UserService {
    public function find($id): User {
        $user = DB::query("SELECT * FROM users WHERE id = ?", [$id]);
        if (!$user) {
            throw new UserNotFound("User $id does not exist");
        }
        return $user;
    }
}

// Now callers can catch specific exceptions
try {
    $user = $userService->find($id);
} catch (UserNotFound $e) {
    Log::warning("User not found: " . $e->getMessage());
} catch (Exception $e) {
    Log::error("Unexpected error: " . $e->getMessage());
}
```

---

## 20. Exception Handling

Use exception handling to manage errors gracefully, not for control flow.

**Example:**
```php
// ❌ BAD: Silent failures and generic exceptions
class UserService {
    public function find($id) {
        try {
            return DB::query("SELECT * FROM users WHERE id = ?", [$id])[0];
        } catch (Exception $e) {
            return null; // Silent failure - hard to debug
        }
    }
}

// ✅ GOOD: Graceful error handling with specific exceptions
class UserNotFoundException extends Exception {}

public function getUser(int $id): User {
    $user = $this->repository->findById($id);
    if (!$user) {
        throw new UserNotFoundException("User with ID $id not found");
    }
    return $user;
}
```

**Benefits:**
- ✅ Specific exceptions allow callers to handle different errors differently
- ✅ Explicit error handling is easier to debug
- ✅ Don't silence errors with generic catch blocks

---

## Quick Reference

| Pattern | Problem | Solution |
|---------|---------|----------|
| **SRP** | Functions do too much | One responsibility per function |
| **Ordering** | Hard to follow | Order matches execution flow |
| **Complex Conditions** | Unreadable if statements | Extract to variables/methods |
| **Boolean Names** | Unclear types | Use `is`, `has`, `can`, `should` |
| **Named Parameters** | Unclear arguments | Use named parameters |
| **Imports** | Scattered dependencies | Group at top of file |
| **Comments** | Over-commenting | Explain intent, not obvious code |
| **Private Functions** | Unclear API surface | Mark private, place at bottom |
| **Readonly** | Accidental mutations | Use readonly on properties |
| **Final** | Unexpected subclasses | Prevent inheritance by default |
| **Constants** | Magic numbers | Use named constants |
| **Long Lines** | Hard to read | Break at logical points |
| **Trailing Commas** | Messy diffs | Add trailing comma to last item |
| **Ternary** | Complex logic | Keep simple, extract complex |
| **Nullsafe** | Nested null checks | Use ?-> operator |
| **Nullable** | Unclear intent | Be explicit about nullability |
| **Hardcoding** | Inflexible code | Use configuration |
| **Custom Exceptions** | Generic errors | Create specific exception types |
| **Exception Handling** | Control flow abuse | Reserve for exceptional cases |

