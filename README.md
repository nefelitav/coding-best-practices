# The Junior Engineer’s Guide to Coding Best Practices

> Short, focused guidance for junior software engineers who want to learn core coding practices.

---

## About This Guide

This repository is a **starter guide for junior engineers**. It's about all the small things that make a big difference.

I created this repository to document the lessons I learned during my first few years as a software engineer, presented in a way that is easy to understand and immediately applicable.

Clean coding practices can also help AI coding assistants like Copilot give better, more reliable suggestions, training both you and the AI to write accurate, maintainable, and secure code.

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

