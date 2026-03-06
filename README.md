# A Practical Starter for Junior Software Engineers

> Short, focused guidance for junior software engineers who want to learn core coding practices.

---

## Why This Repo?

This repository is a **starter guide for junior engineers**. It focuses on the basics that matter most:

- Write clear, maintainable code
- Avoid common bugs
- Write simple, reliable tests
- Keep code secure

---

## Main Learning Guides

| Guide | Best For |
|-------|----------|
| [Code Organization](./docs/CODE_ORGANIZATION.md) | Learning clean code basics | 
| [Architecture & Design](./docs/ARCHITECTURE_DESIGN.md) | Understanding system design | 
| [Testing](./docs/TESTING.md) | Writing tests that work | 
| [Security](./docs/SECURITY.md) | Protecting your app |
| [Performance](./docs/PERFORMANCE.md) | Making queries fast |

---

## Interactive Challenges

Test your knowledge:

### Challenge #1: Messy Function
```php
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

How many issues can you find? [See the solution](./docs/challenges/CHALLENGE_01.md)

