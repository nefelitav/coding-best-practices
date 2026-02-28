# Coding Guidelines & Best Practices

A comprehensive guide to coding standards and best practices organized by topic.

---

## Table of Contents

1. [Architecture & Design](#architecture--design)
2. [Code Organization & Structure](#code-organization--structure)
3. [Functions & Methods](#functions--methods)
4. [Error Handling & Exceptions](#error-handling--exceptions)
5. [Data Management & Persistence](#data-management--persistence)
6. [Testing](#testing)
7. [Security](#security)
8. [Documentation & Comments](#documentation--comments)
9. [PHP-Specific Guidelines](#php-specific-guidelines)
10. [Code Quality & Performance](#code-quality--performance)

---

## Architecture & Design

### Hexagonal Architecture
Organize code so controllers manage input/output, services handle business logic, and repositories deal with database interactions.

### Dependency Injection (DI)
Leverage DI for automatic object creation and easier testing.

### Domain Objects & Immutability
Domain objects should be immutable by default. State changes must happen through explicit methods that return new instances. Mutability is very dangerous!

### Immutability
Mark class properties as `readonly` to ensure immutability.

### Data Transfer Objects (DTOs)
Use DTOs when crossing boundaries (API, service, or persistence layers) to control exactly what data is exposed, decouple your domain model, and prevent leaking internal implementation details.

### Repository Pattern
Define interfaces for repositories to decouple usage from implementation and simplify transitions.

### Repository Implementation
Repositories must hide persistence implementation details and return domain objects, not persistence models. Hydrate models.

### Status Machines
If business rules depend on status and transitions are restricted, model them explicitly, not as strings or flags.

---

## Code Organization & Structure

### Function Responsibility
Each function should have one clear responsibility for better maintainability.

### Function Order
Arrange functions and calls in the order they run to improve readability.

### Private Functions
Make functions private if they shouldn't be exposed outside the class. Place private functions at the bottom of the class for better readability.

### Class Extension Prevention
Use `final` to prevent a class from being extended or subclassed.

### Import Statements
Always import packages at the top of your files.

### Trailing Commas
Add trailing commas for cleaner diffs and easier code edits.

### Line Breaking
Break lines when they get too big and you need to scroll to read the code, for readability.

### Edge Case Centralization
Keep edge cases and hacks in one place to reduce confusion.

### Code Ownership
CODEOWNERS ensures the right people automatically review changes, improving code quality and accountability. Define CODEOWNERS by mapping clear, minimal path patterns to the smallest responsible team (not individuals), keep the file ordered from most-specific to least-specific rules, and review it regularly to reflect ownership changes.

---

## Functions & Methods

### Named Parameters
Use named parameters to clarify what each argument represents.

### Conditional Extraction
Extract complex if conditions into variables for readability.

### Boolean Naming
Start bool var with 'is' or 'has' to show it is bool.

### Enumerations
Use enums to manage different options. Cleaner than using strings.

### Nullable Fields
Decide carefully if a field should be nullable or required.

### Custom Functions
Prefer company-built custom functions over reinventing the wheel.

### Custom Exceptions
Use custom exceptions to handle distinct error scenarios clearly.

---

## Error Handling & Exceptions

### Exception Handling
Use exception handling to manage errors gracefully.

### Assertions for Preconditions
Use assertions to check internal preconditions.

### Logging Errors
Log errors to help with debugging and issue tracking.

### Database Transactions
Use database transactions when multiple operations must succeed or fail as a single unit. Keep the db transaction as light as possible. Add assertion for code that should only be inside a db transaction.

---

## Data Management & Persistence

### Database Queries
Always run EXPLAIN (or EXPLAIN ANALYZE) before optimizing a query, and don't trust it's fast unless you confirm the plan uses the right index and scans as few rows as possible.

### Query Optimization
Join tables on indexed columns to speed up queries.

### Batch Queries
Design queries to retrieve all needed data at once, minimizing multiple round-trips.

### Column Placement
When adding a new column in a migration, place it after a specific column and update related indexes.

### Data Normalization
Avoid storing the same information in multiple places to prevent errors during insert, update, or delete.

### Data Seeding
Use data seeds to populate the database with initial or test data.

### Boolean Column Type
Although BOOLEAN is an alias, TINYINT(1) is clearer about storing 0 or 1.

### Date Formatting
Always use shared date format constants (such as DateFormat::*) instead of hard-coded date format strings to ensure consistency, correctness, and maintainability across the codebase.

---

## Testing

### Test Structure
Each test must follow a clear Arrange – Act – Assert structure and verify one observable behavior.

### Test Coverage
Always assert both positive and negative outcomes.

### Comprehensive Assertions
Make assertions as generic and global as possible: assert against the entire array instead of individual items to catch unexpected additions.

### Exception Testing
To check that an exception is thrown in a test and then assert more things, add a try catch and add assertions inside catch. At the end of try, add self::fail.

### Reusable Assertions
Create reusable assertion helpers for models and aggregates e.g. ProductTestAssertions::assertProductCreated($product, $expectedData).

### Test Fixtures
Fixtures must create only the data required for the test and must make intent explicit e.g. createInvoiceableOrganization().

### Test Process Classes
Use a Test Process class when tests involve repeated multi-step workflows, complex setup, or when you want scenario-style readability in integration tests. Prefer Process classes over duplicating setup logic.

### Test Interfaces
Do not create interfaces solely for testing. Use in-memory implementations or concrete test doubles instead.

### Test Doubles
Inject test doubles via the constructor, never via method parameters. Initialize all test doubles in setUp(). Override implementations using $this->getDi()->overrideImplementation() so wiring mirrors production behavior.

### Mocking Best Practices
Minimize mocks to keep tests closer to real behavior.

### Idempotency Testing
Explicitly test idempotency.

### Enum Testing
Test all enum values explicitly.

### Test Method Naming
Prefer domain-language method names in tests, processes, and assertions.

### ORM Model Isolation
Do not pass ORM models between test helpers. Fixtures, Processes, and Assertions should query repositories themselves. This keeps tests closer to real production behavior and avoids leaking implementation details.

### Time Freezing
Freeze time in tests to ensure consistent, predictable results.

### Data Providers
Use data providers only to test meaningful variations of the same behavior, keep the data minimal and clearly named, and ensure failures can be traced to a specific data set.

---

## Security

### Input Validation
Always validate user input to ensure data integrity and security.

### Output Sanitization
Sanitize output using h() to prevent XSS attacks.

### SQL Injection Prevention
Use parameters in SQL queries to prevent SQL injection.

---

## Documentation & Comments

### Comments Usage
Add comments only to explain complex or non-obvious code. If the code needs comments, it's likely not good enough.

### API Documentation
Add clear documentation to Swagger for easy API understanding.

### Development Console
Always check the browser console for errors during development.

### Feature Flags
Use feature flags to safely deploy and control new features.

### Dry-Run Functions
Add a dry-run function to scripts to preview changes without executing them.

### URL Stability
When adding links in the frontend, use stable URLs that won't change over time (e.g., Confluence share links).

---

## PHP-Specific Guidelines

### Type Hints
Use type hints to improve code clarity and catch errors early.

### Type Checking in Tests
Prefer assertSame over assertEquals for stricter type and value checks.

### Boolean Assertions
Prefer assertTrue/assertFalse for boolean assertions over assertSame.

### Array Transformations
Use array_map to simplify array transformations and write cleaner code.

### Nullsafe Operator
Use the nullsafe operator (?->) to simplify conditional returns when accessing properties or methods on nullable objects.

### Ternary Operators
Apply ternary operators for concise conditional expressions.

### Tuple Usage
Avoid using tuples, because they are not well typed.

---

## Code Quality & Performance

### Hard-coding
Avoid hardcoding to keep code flexible and easier to maintain.

### Edge Cases
Always consider edge cases, like when data is null.

---

## Summary

These guidelines provide a comprehensive framework for writing maintainable, secure, and testable code. They emphasize:

- **Clean Architecture**: Clear separation of concerns
- **Robust Testing**: Thorough and well-organized test strategies
- **Security First**: Input validation and output sanitization
- **Code Clarity**: Readable code with clear intent
- **Performance**: Optimized queries and batch operations
- **Immutability**: Safer domain objects through immutability

Follow these principles consistently to build high-quality software that is easy to maintain, test, and extend.

