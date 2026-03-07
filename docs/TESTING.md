# Testing

Write tests that are clear, maintainable, and verify real behavior.

---

## 1. Arrange-Act-Assert Structure

Each test must follow a clear Arrange–Act–Assert structure and verify one observable behavior.

**Pattern:**
```
Arrange: Set up test data and dependencies
Act: Perform the action being tested
Assert: Verify the result
```

**Example:**
```php
// ❌ BAD: Multiple behaviors in one test
public function testUserFunctionality(): void {
    $user = new User();
    $user->name = 'John';
    $this->assertEquals('John', $user->name);
    
    $user->suspend();
    $this->assertEquals('suspended', $user->status);
    
    $user->resume();
    $this->assertEquals('active', $user->status);
}
    
// ✅ GOOD: Clear structure
public function testUserCanBeSuspended(): void {
    // Arrange
    $user = User::create(['name' => 'John', 'status' => 'active']);
    $service = new UserSuspensionService();
    
    // Act
    $suspendedUser = $service->suspend($user, 'Spam');
    
    // Assert
    $this->assertEquals('suspended', $suspendedUser->status);
    $this->assertEquals('Spam', $suspendedUser->suspension_reason);
}
```

---

## 2. Test Both Positive and Negative Outcomes

Always assert both positive outcomes (happy path) and negative outcomes (edge cases).

**Example:**
```php
class InvoiceServiceTest extends TestCase {
    // ✅ GOOD: Test success case
    public function testCreateValidInvoice(): void {
        $invoice = InvoiceService::create([
            'amount' => 100,
            'customer_id' => 1,
        ]);
        
        $this->assertNotNull($invoice->id);
        $this->assertEquals(100, $invoice->amount);
    }
    
    // ✅ GOOD: Test failure cases
    public function testCannotCreateInvoiceWithoutAmount(): void {
        $this->expectException(ValidationException::class);
        
        InvoiceService::create(['customer_id' => 1]);
    }
    
    public function testCannotCreateInvoiceWithNegativeAmount(): void {
        $this->expectException(ValidationException::class);
        
        InvoiceService::create(['amount' => -100, 'customer_id' => 1]);
    }
}
```

---

## 3. Assertion Specificity

Make assertions as specific as possible - assert against the entire collection instead of individual items to catch unexpected additions.

**Example:**
```php
// ❌ BAD: Assertions too loose
$users = $userRepository->getAllAdmins();
$this->assertCount(1, $users);
$this->assertEquals('John', $users[0]->name); // What if there's a second user?

// ✅ GOOD: Assert entire structure
$admins = $userRepository->getAllAdmins();
$this->assertEquals(['John'], array_map(fn($u) => $u->name, $admins));

// ✅ GOOD: Use exact assertions
$this->assertSame($expected, $actual); // Type and value
$this->assertEquals($expected, $actual); // Just value
```

---

## 4. Exception Testing with Try-Catch

To check that an exception is thrown in a test and then assert more things, use try-catch pattern with `fail()`.

**Example:**
```php
// ✅ GOOD: Assert exception and additional behavior
public function testSuspensionThrowsExceptionAndLogsError(): void {
    try {
        $this->service->suspendWithInvalidReason($user, '');
    } catch (InvalidArgumentException $e) {
        $this->assertEquals('Reason cannot be empty', $e->getMessage());
        $this->assertTrue(Log::hasError('Suspension failed'));
        return;
    }
    
    $this->fail('Expected InvalidArgumentException was not thrown');
}

// Alternative: Using PHPUnit's built-in methods
// But then you don't get to assert e.g. that nothing was persisted
public function testSuspensionThrows(): void {
    $this->expectException(InvalidArgumentException::class);
    $this->expectExceptionMessage('Reason cannot be empty');
    
    $this->service->suspendWithInvalidReason($user, '');
}
```

---

## 5. Reusable Assertion Helpers

Create reusable assertion helpers for models and aggregates.

**Example:**
```php
// ✅ GOOD: Centralized assertions
class ProductTestAssertions {
    public static function assertProductCreated(Product $product, array $expected): void {
        self::assertNotNull($product->id);
        self::assertEquals($expected['name'], $product->name);
        self::assertEquals($expected['sku'], $product->sku);
        self::assertEquals($expected['price'], $product->price);
        self::assertTrue($product->isActive);
    }
    
    public static function assertProductSuspended(Product $product): void {
        self::assertFalse($product->isActive);
        self::assertNotNull($product->suspended_at);
    }
}

// Usage
public function testCreateProduct(): void {
    $product = ProductService::create(['name' => 'Widget', 'sku' => 'WGT-001', 'price' => 29.99]);
    
    ProductTestAssertions::assertProductCreated($product, [
        'name' => 'Widget',
        'sku' => 'WGT-001',
        'price' => 29.99,
    ]);
}
```

---

## 6. Test Fixtures

Fixtures must create only the data required for the test and must make intent explicit through descriptive method names.

**Example:**
```php
// ❌ BAD: Unclear what data is created
protected function setUp(): void {
    $this->user = User::create(['name' => 'John']);
    $this->org = Organization::create(['name' => 'Acme']);
    $this->invoice = Invoice::create(['organization_id' => $this->org->id]);
}

// ✅ GOOD: Intent is clear and minimal
protected function createInvoiceableOrganization(): Organization {
    return Organization::create([
        'name' => 'Acme',
        'status' => 'active',
        'has_billing_setup' => true,
    ]);
}

protected function createAdminUser(): User {
    return User::create([
        'name' => 'Admin',
        'role' => UserRole::ADMIN,
        'is_active' => true,
    ]);
}

// Usage - intent is obvious
public function testOnlyAdminsCanApproveInvoices(): void {
    $admin = $this->createAdminUser();
    $org = $this->createInvoiceableOrganization();
    
    $this->assertTrue($admin->canApprove($org));
}
```

---

## 7. Test Process Classes

Use a Test Process class when tests involve repeated multi-step workflows, complex setup, or when you want scenario-style readability in integration tests.

**Example:**
```php
// ✅ GOOD: Clear workflow in process class
class OrderCheckoutProcess {
    private Order $order;
    
    public function withCustomer(Customer $customer): self {
        $this->order = Order::create(['customer_id' => $customer->id]);
        return $this;
    }
    
    public function withItems(array $items): self {
        foreach ($items as $item) {
            $this->order->addItem($item);
        }
        return $this;
    }
    
    public function checkout(): Order {
        return OrderService::checkout($this->order);
    }
    
    public function getOrder(): Order {
        return $this->order;
    }
}

// Usage - reads like a story
public function testOrderCheckoutFlow(): void {
    $customer = Customer::create(['name' => 'John']);
    
    $order = (new OrderCheckoutProcess())
        ->withCustomer($customer)
        ->withItems(['item1', 'item2'])
        ->checkout()
        ->getOrder();
    
    $this->assertEquals('completed', $order->status);
    $this->assertEquals(2, count($order->items));
}
```

---

## 8. Avoid Test Interfaces

Do not create interfaces solely for testing. Use in-memory implementations or concrete test doubles instead.

**Example:**
```php
// ❌ BAD: Interface exists only for testing
interface UserRepositoryInterface { /* Only used in tests */ }

class UserRepository implements UserRepositoryInterface { /* Production */ }
class InMemoryUserRepository implements UserRepositoryInterface { /* Tests only */ }

// ✅ GOOD: Use concrete implementations
final class InMemoryUserRepository {
    private array $users = [];
    
    public function save(User $user): void {
        $this->users[$user->id] = $user;
    }
    
    public function find(int $id): ?User {
        return $this->users[$id] ?? null;
    }
}

// In test setup
$this->repository = new InMemoryUserRepository();
$this->service = new UserService($this->repository);
```

---

## 9. Test Double Injection

Inject test doubles via the constructor, never via method parameters. Initialize all test doubles in setUp(). Override implementations using DI container.

**Example:**
```php
// ✅ GOOD: Constructor injection
class OrderServiceTest extends TestCase {
    private MockRepository $repository;
    private OrderService $service;
    
    protected function setUp(): void {
        $this->repository = new MockRepository();
        $this->service = new OrderService($this->repository);
    }
    
    public function testOrderIsPersistedAfterCreation(): void {
        $order = $this->service->create(['amount' => 100]);
        
        $this->repository->assertSaved($order);
    }
}

// Using DI container override
protected function setUp(): void {
    parent::setUp();
    
    $this->getDi()->overrideImplementation(
        PaymentGateway::class,
        new MockPaymentGateway()
    );
}
```

---

## 10. Idempotency Testing

Explicitly test that operations are idempotent (running them twice doesn't change the result).

**Example:**
```php
// ✅ GOOD: Test idempotency
public function testActivatingUserIsIdempotent(): void {
    $user = User::create(['status' => 'inactive']);
    
    // Activate once
    UserService::activate($user);
    $this->assertEquals('active', $user->status);
    
    // Activate again
    UserService::activate($user);
    $this->assertEquals('active', $user->status);
    
    // Should still be one user in database
    $this->assertCount(1, User::all());
}

public function testProcessingOrderTwiceDoesNotDuplicateCharges(): void {
    $order = Order::create(['amount' => 100, 'status' => 'pending']);
    
    // Process once
    $result1 = OrderProcessor::process($order);
    $this->assertTrue($result1);
    
    // Process again
    $result2 = OrderProcessor::process($order);
    $this->assertFalse($result2); // Should not process again
    
    // Verify only one charge
    $this->assertCount(1, PaymentLog::forOrder($order->id));
}
```

---

## 11. Enum Testing

Test all enum values explicitly to ensure they're handled correctly.

**Example:**
```php
// ✅ GOOD: Test all enum values
public function testAllOrderStatusesHaveTransitions(): void {
    foreach (OrderStatus::cases() as $status) {
        $this->assertNotEmpty($status->validTransitions());
    }
}

public function testAllUserRolesHavePermissions(): void {
    foreach (UserRole::cases() as $role) {
        $permissions = $role->getPermissions();
        $this->assertIsArray($permissions);
        $this->assertNotEmpty($permissions);
    }
}

class OrderStatusTest extends TestCase {
    public function testCanTransitionFromPendingToConfirmed(): void {
        $this->assertTrue(OrderStatus::PENDING->canTransitionTo(OrderStatus::CONFIRMED));
    }
    
    public function testCannotTransitionFromConfirmedToPending(): void {
        $this->assertFalse(OrderStatus::CONFIRMED->canTransitionTo(OrderStatus::PENDING));
    }
}
```

---

## 12. Domain Language in Tests

Prefer domain-language method names in tests, processes, and assertions.

**Example:**
```php
// ❌ BAD: Test language
public function testUserA(): void {
    $u = User::create(['name' => 'John']);
    $s = new Service();
    $r = $s->method1($u);
    $this->assertTrue($r);
}

// ✅ GOOD: Domain language
public function testCustomerCanCheckoutOrder(): void {
    $customer = $this->createPremiumCustomer();
    $order = $this->createOrderWithItems($customer);
    
    $checkout = OrderCheckout::performFor($customer, $order);
    
    $this->assertTrue($checkout->isSuccessful());
    $this->assertEquals('completed', $order->status);
}
```

---

## 13. ORM Models in Tests

Do not pass ORM models between test helpers. Fixtures, Processes, and Assertions should query repositories themselves.

**Example:**
```php
// ❌ BAD: Passing ORM model
class OrderTestHelper {
    public function markAsShipped(Order $order): void {
        $order->status = 'shipped'; // Works in test, not production-like
        $order->save();
    }
}

// ✅ GOOD: Use repository pattern
class OrderTestHelper {
    public function __construct(private OrderRepository $repository) {}
    
    public function markAsShipped(int $orderId): void {
        $order = $this->repository->find($orderId);
        $this->repository->save($order->ship());
    }
}

// Usage
public function testOrderShipment(): void {
    $order = OrderFactory::create();
    
    (new OrderTestHelper($this->repository))
        ->markAsShipped($order->id);
    
    $refreshed = $this->repository->find($order->id);
    $this->assertEquals('shipped', $refreshed->status);
}
```

---

## 14. Freeze Time

Freeze time in tests to ensure consistent, predictable results.

**Example:**
```php
use Carbon\Carbon;

class EmailReminderTest extends TestCase {
    public function testReminderSentOneDayBeforeExpiry(): void {
        // Freeze time at 2024-01-01
        Carbon::setTestNow(Carbon::parse('2024-01-01'));
        
        $invoice = Invoice::create([
            'due_date' => Carbon::parse('2024-01-02'),
        ]);
        
        $reminders = EmailReminder::generate();
        
        $this->assertContains($invoice->id, $reminders->pluck('invoice_id'));
        
        // Restore time
        Carbon::setTestNow();
    }
}
```

---

## 15. Data Providers

Use data providers only to test meaningful variations of the same behavior. Keep the data minimal and clearly named. Ensure failures can be traced to a specific data set.

**Example:**
```php
// ✅ GOOD: Meaningful variations with clear data
#[DataProvider('validEmailProvider')]
public function testValidEmailsAreAccepted(string $email): void {
    $this->assertTrue(EmailValidator::isValid($email));
}

public static function validEmailProvider(): array {
    return [
        'simple email' => ['user@example.com'],
        'email with plus' => ['user+tag@example.com'],
        'email with subdomain' => ['user@mail.example.com'],
    ];
}

#[DataProvider('invalidEmailProvider')]
public function testInvalidEmailsAreRejected(string $email): void {
    $this->assertFalse(EmailValidator::isValid($email));
}

public static function invalidEmailProvider(): array {
    return [
        'missing domain' => ['user@'],
        'missing user' => ['@example.com'],
        'no at symbol' => ['userexample.com'],
    ];
}

// ❌ BAD: Excessive data variations
#[DataProvider('randomTestData')]
public function testSomething($a, $b, $c, $d, $e, $f): void {
    // Hard to understand what each variation tests
}
```

---

## 16. Edge Cases

Always consider edge cases, like when data is null, empty, or at boundaries.

**Example:**
```php
// ✅ GOOD: Test edge cases
public function testCalculateTotalWithNoItems(): void {
    $order = Order::create();
    $this->assertEquals(0, $order->calculateTotal());
}

public function testCalculateTotalWithZeroPrice(): void {
    $order = Order::create();
    $order->addItem(['price' => 0, 'quantity' => 5]);
    $this->assertEquals(0, $order->calculateTotal());
}

public function testCalculateTotalWithNegativeQuantity(): void {
    $this->expectException(InvalidQuantityException::class);
    
    $order = Order::create();
    $order->addItem(['price' => 100, 'quantity' => -1]);
}

public function testFindWithNullId(): void {
    $result = $this->repository->find(null);
    $this->assertNull($result);
}

public function testStringWithSpecialCharacters(): void {
    $user = User::create(['name' => '<script>alert("xss")</script>']);
    $this->assertContains('&lt;script&gt;', htmlspecialchars($user->name));
}
```

---

## 17. Test Double Strategies

Minimize mocks to keep tests closer to real behavior. Use strategic doubles for slow operations.

**Example:**
```php
// ✅ GOOD: Minimal mocking
class OrderServiceTest extends TestCase {
    public function testOrderProcessing(): void {
        // Use real repository
        $repository = new InMemoryOrderRepository();
        
        // Use real calculation logic
        $calculator = new TaxCalculator();
        
        // Mock only slow/external services
        $paymentGateway = new MockPaymentGateway();
        
        $service = new OrderService($repository, $calculator, $paymentGateway);
        
        $order = $service->process($this->createTestOrder());
        
        $this->assertEquals('completed', $order->status);
    }
}

// ❌ BAD: Over-mocking
class OrderServiceTest extends TestCase {
    public function testOrderProcessing(): void {
        $repository = Mock::of(OrderRepository::class);
        $calculator = Mock::of(TaxCalculator::class);
        $gateway = Mock::of(PaymentGateway::class);
        
        // Tests mock behavior, not actual logic
        $this->assertMockCalled($repository, 'save');
    }
}
```

---

## 18. Dry-Run Functions

Add a dry-run function to scripts to preview changes without executing them.

**Example:**
```php
// ✅ GOOD: Dry-run support
class DataMigrationScript {
    public function execute(bool $dryRun = true): void {
        $changes = $this->calculateChanges();
        
        if ($dryRun) {
            echo "DRY RUN - Would make " . count($changes) . " changes:\n";
            foreach ($changes as $change) {
                echo "  - " . $change['description'] . "\n";
            }
            return;
        }
        
        foreach ($changes as $change) {
            $this->applyChange($change);
            echo "Applied: " . $change['description'] . "\n";
        }
    }
}

// Usage
$script = new DataMigrationScript();
$script->execute(dryRun: true); // Preview
$script->execute(dryRun: false); // Execute

// Test
public function testDryRunDoesNotModifyData(): void {
    $script = new DataMigrationScript();
    
    $before = User::all()->count();
    $script->execute(dryRun: true);
    $after = User::all()->count();
    
    $this->assertEquals($before, $after);
}
```

---

## Quick Reference

| Practice | Problem | Solution |
|----------|---------|----------|
| **AAA** | Unclear test structure | Arrange → Act → Assert |
| **Both Outcomes** | Incomplete testing | Test success and failure |
| **Specific Assertions** | Loose verification | Assert exact values/structure |
| **Exception Testing** | Limited exception checks | Use try-catch with fail() |
| **Assertion Helpers** | Duplicate verification | Create reusable assertion classes |
| **Clear Fixtures** | Unclear test setup | Use descriptive fixture methods |
| **Process Classes** | Complex workflow tests | Use process classes for scenarios |
| **No Test Interfaces** | Over-engineering | Use concrete implementations |
| **Constructor Injection** | Unclear test setup | Inject via constructor |
| **Idempotency** | Side effects missed | Test running twice |
| **All Enums** | Missing enum cases | Test every enum value |
| **Domain Language** | Unclear test intent | Use business terminology |
| **No ORM Passing** | Production unlike tests | Query repositories in helpers |
| **Freeze Time** | Flaky time-based tests | Use Clock::freeze() |
| **Data Providers** | Excessive test bloat | Use for meaningful variations only |
| **Edge Cases** | Missing boundary tests | Test null, empty, limits |
| **Minimal Mocks** | Brittle tests | Mock only slow operations |
| **Dry-Run** | Destructive scripts | Add dry-run mode |

