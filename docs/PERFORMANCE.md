# Performance & Scalability

Write fast code that scales as your business grows.

---

## 1. Database Indexes

### Rule: Use indexes on columns that are frequently filtered or joined

Indexes dramatically speed up queries by avoiding full table scans.

**Example:**
```php
// ❌ BAD: No index on frequently queried column
SELECT * FROM orders WHERE customer_id = ?; // Scans all rows

// ✅ GOOD: Add index
ALTER TABLE orders ADD INDEX idx_customer_id (customer_id);
SELECT * FROM orders WHERE customer_id = ?; // Uses index, fast!

// ✅ GOOD: Composite index for common WHERE + ORDER BY patterns
ALTER TABLE orders ADD INDEX idx_customer_created (customer_id, created_at);
SELECT * FROM orders 
WHERE customer_id = ? AND created_at > ? 
ORDER BY created_at DESC; // Uses composite index
```

**When to index:**
- Foreign key columns (for JOINs)
- Columns used in WHERE clauses
- Columns used in ORDER BY
- Columns used in GROUP BY

**When NOT to index:**
- Low-cardinality columns (too many duplicate values)
- Columns that are rarely queried
- Small tables (index overhead not worth it)

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

## Performance Checklist

Before shipping code that touches the database:

- [ ] Queries use parameterized syntax
- [ ] Queries have been EXPLAIN'd to verify index usage
- [ ] No N+1 query patterns (use eager loading)
- [ ] Batch operations use WHERE IN instead of looping
- [ ] Transaction scope is minimal and focused
- [ ] Indexes are created for foreign keys and WHERE clauses

---

## Quick Reference

| Pattern | Problem | Solution |
|---------|---------|----------|
| **No Indexes** | Slow queries | Add `INDEX` on join columns |
| **Unknown Query Plan** | Assume it's fast | Run `EXPLAIN` first |
| **N+1 Queries** | Load each item separately | Eager load or batch fetch |
| **Long Transactions** | Tables locked too long | Keep transactions minimal |


