# Performance & Scalability

Write fast code that scales as your business grows.

---

## 1. Database Indexes

### Rule: Join tables on indexed columns

Speed up queries dramatically by joining on indexed columns.

**Example:**
```php
// ❌ BAD: No indexes
SELECT * FROM orders 
WHERE user_id = ? AND created_at > ?; // Scans all rows

// ✅ GOOD: Add indexes
ALTER TABLE orders ADD INDEX idx_user_created (user_id, created_at);

// Query becomes fast (uses index)
SELECT * FROM orders 
WHERE user_id = ? AND created_at > ?;
```

---

## 2. Query Optimization

### Rule: Always run EXPLAIN before optimizing

Don't trust it's fast unless you confirm the plan uses the right index and scans as few rows as possible.

**Pattern:**
```php
// Step 1: Write query
SELECT * FROM orders 
JOIN customers ON orders.customer_id = customers.id
WHERE customers.country = ? AND orders.created_at > ?;

// Step 2: Run EXPLAIN
EXPLAIN SELECT * FROM orders 
JOIN customers ON orders.customer_id = customers.id
WHERE customers.country = ? AND orders.created_at > ?;

// Step 3: Check results
// type: Should be 'ref' or 'range', not 'ALL'
// rows: Should be small (not millions)
// key: Should show which index is being used
// key_len: Should be reasonable size

// Step 4: Add indexes if needed
ALTER TABLE customers ADD INDEX idx_country (country);
ALTER TABLE orders ADD INDEX idx_customer_created (customer_id, created_at);

// Step 5: EXPLAIN again to confirm improvement
EXPLAIN SELECT * FROM orders ...;
```

---

## 3. Batch Queries

### Rule: Fetch multiple items at once instead of looping

Avoid N+1 query problem by batching queries.

**Example:**
```php
// ❌ BAD: N+1 problem (1 + 100 queries = 101 queries!)
$orders = Order::all();
foreach ($orders as $order) {
    $customer = Customer::find($order->customer_id); // Query inside loop!
    echo $customer->name;
}

// ✅ GOOD: Eager load (1 query)
$orders = Order::with('customer')->get(); // Loads all customers at once
foreach ($orders as $order) {
    echo $order->customer->name; // No query, already loaded
}

// ✅ GOOD: Batch fetch
$orderIds = [1, 2, 3, 4, 5];
$orders = Order::whereIn('id', $orderIds)->get(); // Fetches all 5 at once

// ✅ GOOD: Use JOIN
$orders = Order::join('customers', 'orders.customer_id', '=', 'customers.id')
    ->select('orders.*', 'customers.name')
    ->get();
```

---

## 4. Database Transactions

### Rule: Keep transactions as light as possible

Lock tables briefly. Add assertions for code that should only be inside a transaction.

**Example:**
```php
// ❌ BAD: Long transaction
DB::transaction(function() {
    $customer = Customer::create($data); // Fast
    sleep(5); // ❌ Why is this here?
    $order = Order::create(['customer_id' => $customer->id]); // Tables locked for 5+ seconds!
});

// ✅ GOOD: Light transaction
$customer = Customer::create($data); // Outside transaction
$order = DB::transaction(function() use ($customer) {
    return Order::create(['customer_id' => $customer->id]); // Fast
});

// ✅ GOOD: With assertion
$order = DB::transaction(function() use ($customer) {
    // This code should ONLY run inside transaction
    assert(DB::transactionLevel() > 0, 'Must be in transaction');
    
    return Order::create(['customer_id' => $customer->id]);
});
```

---

## 5. Database Migrations

### Rule: Place new columns after specific columns and update indexes

Think carefully about column placement and indexes when migrating.

**Example:**
```php
// ✅ GOOD: Strategic column placement
Schema::table('orders', function (Blueprint $table) {
    // Add new column after 'customer_id'
    $table->string('tracking_number')->after('customer_id')->index();
    
    // Update composite index
    $table->index(['customer_id', 'tracking_number']);
});

// ❌ BAD: Adds column at end, breaks performance
Schema::table('orders', function (Blueprint $table) {
    $table->string('tracking_number'); // Added at end, index may be inefficient
});

// ✅ GOOD: Add covering index
Schema::table('orders', function (Blueprint $table) {
    // Index includes both lookup and data columns
    $table->index(['customer_id', 'created_at', 'total']);
});
```

---

## 6. Data Normalization

### Rule: Avoid storing the same information in multiple places

Prevent errors during insert, update, or delete by normalizing data.

**Example:**
```php
// ❌ BAD: Denormalized (duplicate data)
CREATE TABLE orders (
    id INT,
    customer_name VARCHAR(255), -- ❌ Duplicated from customers table
    customer_email VARCHAR(255), -- ❌ Duplicated from customers table
    total DECIMAL(10, 2),
    customer_calculated_total DECIMAL(10, 2) -- ❌ Duplicated (can drift!)
);

// If customer name changes, must update everywhere!
// If calculation changes, must recalculate everywhere!
// Easy to have stale/inconsistent data

// ✅ GOOD: Normalized
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255),
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    total DECIMAL(10, 2),
    created_at TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

// Query joins to get data, always up-to-date
SELECT o.id, c.name, c.email, o.total 
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

---

## Performance Checklist

Before shipping code that touches the database:

- [ ] Queries use parameterized syntax
- [ ] Queries have been EXPLAIN'd to verify index usage
- [ ] No N+1 query patterns (use eager loading)
- [ ] Batch operations use WHERE IN instead of looping
- [ ] Transaction scope is minimal and focused
- [ ] Indexes are created for foreign keys and WHERE clauses
- [ ] No redundant data storage
- [ ] Long-running operations happen outside transactions

---

## Quick Reference

| Pattern | Problem | Solution |
|---------|---------|----------|
| **No Indexes** | Slow queries | Add `INDEX` on join columns |
| **Unknown Query Plan** | Assume it's fast | Run `EXPLAIN` first |
| **N+1 Queries** | Load each item separately | Eager load or batch fetch |
| **Long Transactions** | Tables locked too long | Keep transactions minimal |
| **No Index Strategy** | Random performance | Plan indexes during migration |
| **Duplicate Data** | Inconsistency | Normalize relationships |
| **No Analysis** | Slow without knowing why | Use `EXPLAIN ANALYZE` |

