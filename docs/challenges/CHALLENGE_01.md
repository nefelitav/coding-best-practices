# 🎮 Challenge #1: The Messy Function

## The Problem

You've been asked to review this function. It works, but it has MANY issues. Can you spot them all?

```php
<?php

function processUserData($data) {
    $user = DB::query("SELECT * FROM users WHERE id = " . $data['id']);
    
    if ($user) {
        $user['name'] = $data['name'];
        DB::query("UPDATE users SET name = '" . $data['name'] . "' WHERE id = " . $data['id']);
        Mail::send($user['email'], "Profile Updated");
        Log::info("Updated user " . $data['id']);
        return true;
    }
    return false;
}
```

## Your Task

Find at least 7 issues with this code. Write them down before checking the solution!

---

## 🤔 Think About:

- **Security**: Are there any vulnerabilities?
- **Structure**: Does the function do too much?
- **Error Handling**: What could go wrong?
- **Testing**: Could you easily test this?
- **Type Safety**: Are types being used properly?

---

## Hints (Read these if you're stuck)

<details>
<summary>💡 Hint 1: Security</summary>

Look at how the SQL queries are constructed. Are user inputs being used safely?

</details>

<details>
<summary>💡 Hint 2: Responsibility</summary>

How many different things is this function doing? Remember: one function, one responsibility!

</details>

<details>
<summary>💡 Hint 3: Types</summary>

What type is `$data`? What type is returned? Could this be clearer?

</details>

---

## 🎯 The Solution

### Issues Found

1. **🚨 SQL Injection Vulnerability (CRITICAL)**
   - Line 4: `"SELECT * FROM users WHERE id = " . $data['id']`
   - Line 8: `"UPDATE users SET name = '" . $data['name'] . "'"`
   - **Fix**: Use parameterized queries

2. **🎯 Too Many Responsibilities**
   - Function does 5 things: fetch, update, email, log, return
   - **Fix**: Split into separate functions

3. **⚠️ No Input Validation**
   - Doesn't check if `$data` has required keys
   - Doesn't validate email format
   - **Fix**: Validate all inputs

4. **⚠️ No Error Handling**
   - What if DB query fails?
   - What if email fails?
   - **Fix**: Add proper exception handling

5. **📝 No Type Hints**
   - `$data` could be anything
   - Return type is boolean but unclear
   - **Fix**: Add type declarations

6. **🔄 Not Using Transactions**
   - Update and other operations should be atomic
   - **Fix**: Wrap in DB transaction

7. **🧪 Hard to Test**
   - Direct DB and Mail calls
   - Can't test in isolation
   - **Fix**: Use dependency injection

8. **📦 Data Not Validated**
   - Name could be empty or malicious
   - Email format not checked
   - **Fix**: Use DTOs or validation

---

## ✨ The Fixed Version

```php
<?php

final class UserProfileUpdater
{
    public function __construct(
        private readonly UserRepository $userRepository,
        private readonly EmailService $emailService,
        private readonly Logger $logger,
    ) {}

    public function updateProfile(UpdateUserProfileCommand $command): void
    {
        // Validate input
        $this->validateCommand($command);
        
        // Fetch user
        $user = $this->userRepository->findById($command->userId);
        if ($user === null) {
            throw new UserNotFoundException($command->userId);
        }
        
        // Update user in transaction
        DB::transaction(function() use ($user, $command) {
            $user->updateName($command->name);
            $this->userRepository->save($user);
        });
        
        // Send notification (outside transaction)
        $this->notifyUserOfUpdate($user);
        
        // Log success
        $this->logUpdate($user);
    }
    
    private function validateCommand(UpdateUserProfileCommand $command): void
    {
        if (empty($command->name)) {
            throw new InvalidNameException('Name cannot be empty');
        }
        
        if (strlen($command->name) > 255) {
            throw new InvalidNameException('Name too long');
        }
    }
    
    private function notifyUserOfUpdate(User $user): void
    {
        try {
            $this->emailService->send(
                to: $user->getEmail(),
                subject: 'Profile Updated',
                template: 'profile-updated',
            );
        } catch (EmailException $e) {
            // Log but don't fail the update
            $this->logger->warning('Failed to send profile update email', [
                'userId' => $user->getId(),
                'error' => $e->getMessage(),
            ]);
        }
    }
    
    private function logUpdate(User $user): void
    {
        $this->logger->info('User profile updated', [
            'userId' => $user->getId(),
        ]);
    }
}

// Command DTO for type safety
final readonly class UpdateUserProfileCommand
{
    public function __construct(
        public int $userId,
        public string $name,
    ) {}
}

// Usage
$command = new UpdateUserProfileCommand(
    userId: 123,
    name: 'John Doe',
);

$updater = new UserProfileUpdater($userRepository, $emailService, $logger);
$updater->updateProfile($command);
```

---

## 🎓 What We Learned

### Security
- ✅ **No SQL Injection**: Repository handles queries safely
- ✅ **Input Validation**: Validates before processing

### Structure
- ✅ **Single Responsibility**: Each method does ONE thing
- ✅ **Dependency Injection**: Dependencies injected, not hardcoded
- ✅ **Type Safety**: DTOs ensure correct data types

### Error Handling
- ✅ **Explicit Exceptions**: Clear error cases
- ✅ **Graceful Degradation**: Email failure doesn't break update
- ✅ **Transactions**: Data consistency guaranteed

### Testing
- ✅ **Testable**: Can mock all dependencies
- ✅ **Clear Boundaries**: Each method tests independently
