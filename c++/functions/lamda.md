Here is a breakdown of C++ Lambda functions, structured for clarity and technical precision.

### What is a Lambda Expression?

Introduced in C++11, a lambda is an **anonymous function object** (functor). Under the hood, the compiler generates a unique, unnamed struct with an overloaded `operator()` method. They are primarily used to encapsulate short snippets of code that are passed to algorithms or asynchronous methods.

### 1\. Anatomy of a Lambda

The full syntax is:
`[captures] (parameters) specifiers -> return_type { body }`

  * **`[ ]` Capture Clause:** Defines how variables from the surrounding scope are accessible inside the lambda (by value or by reference).
  * **`( )` Parameters:** Just like a standard function (optional if empty).
  * **`mutable` (Specifier):** Optional. Allows modification of variables captured by value.
  * **`->` Return Type:** Optional. The compiler usually deduces this automatically.
  * **`{ }` Body:** The code to be executed.

-----

### 2\. The Capture Clause (`[ ]`)

This is the most critical part to understand for memory safety.

| Syntax | Description |
| :--- | :--- |
| `[]` | Captures nothing. |
| `[=]` | Captures all local variables by **value** (copy). Read-only by default. |
| `[&]` | Captures all local variables by **reference**. |
| `[x]` | Captures `x` by value. |
| `[&y]` | Captures `y` by reference. |
| `[=, &y]` | Captures all by value, except `y` by reference. |
| `[this]` | Captures the current object pointer (needed to access class members). |

> **Critical Warning:** Be extremely careful with `[&]` when the lambda might outlive the current scope (e.g., returning a lambda or passing it to a thread). This creates **dangling references**, a common source of crashes.

-----

### 3\. Usage Examples

#### Basic Syntax

```cpp
#include <iostream>

int main() {
    // A simple lambda stored in a variable
    auto greet = []() {
        std::cout << "Hello from Lambda!" << std::endl;
    };
    
    greet(); // Call it
    
    // Direct execution (Immediately Invoked Lambda Expression)
    [](int x) { std::cout << x * x; }(5); // Prints 25
}
```

#### Capturing Variables (State)

```cpp
#include <iostream>

int main() {
    int multiplier = 3;
    int counter = 0;

    // 'multiplier' captured by value (copy), 'counter' by reference
    auto logic = [multiplier, &counter](int val) {
        counter++; 
        return val * multiplier;
    };

    std::cout << logic(10) << "\n"; // Output: 30
    std::cout << "Counter: " << counter << "\n"; // Output: 1
}
```

-----

### 4\. Sorting with Lambdas (Your Request)

The most common use case is passing a custom comparator to `std::sort`.

#### Example A: Sorting Integers (Custom Order)

```cpp
#include <iostream>
#include <vector>
#include <algorithm> // Required for std::sort

int main() {
    std::vector<int> nums = {1, 5, 2, 8, 3};

    // Sort Descending (Greater than)
    std::sort(nums.begin(), nums.end(), [](int a, int b) {
        return a > b; 
    });

    for(int n : nums) std::cout << n << " "; // Output: 8 5 3 2 1
}
```

#### Example B: Sorting Custom Objects

This is standard in technical interviews (e.g., sorting intervals or logs).

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

struct Employee {
    std::string name;
    int salary;
    int age;
};

int main() {
    std::vector<Employee> staff = {
        {"Alice", 70000, 30},
        {"Bob", 50000, 45},
        {"Charlie", 70000, 25}
    };

    // Sort by Salary (Descending), then by Age (Ascending) if salaries match
    std::sort(staff.begin(), staff.end(), [](const Employee& a, const Employee& b) {
        if (a.salary != b.salary) {
            return a.salary > b.salary; // Higher salary first
        }
        return a.age < b.age; // Younger age first (tie-breaker)
    });

    for(const auto& e : staff) {
        std::cout << e.name << ": " << e.salary << " (" << e.age << ")\n";
    }
    // Output:
    // Alice: 70000 (30)
    // Charlie: 70000 (25) -> Wait, logic check: I sorted age Ascending, so Charlie (25) should be before Alice (30).
    // Correct Output logic applied by std::sort:
    // Charlie: 70000 (25)
    // Alice: 70000 (30)
    // Bob: 50000 (45)
}
```

-----

### 5\. Advanced: Generic Lambdas (C++14)

You can use `auto` in the parameter list. This creates a template-like lambda that works with any type supporting the operations used in the body.

```cpp
auto printDouble = [](auto value) {
    std::cout << value * 2 << std::endl;
};

printDouble(5);     // Int: Prints 10
printDouble(3.5);   // Double: Prints 7.0
printDouble(std::string("Ha")); // Error unless string supports operator* (it doesn't by default)
```

-----

### Summary Checklist

1.  Use `[]` to access nothing.
2.  Use `[&]` sparingly and carefully.
3.  Use lambdas for one-off logic in `std::sort`, `std::for_each`, or `std::transform`.
4.  Remember `mutable` if you need to modify a value-captured variable inside the lambda.

**Would you like me to provide a "Google-style" interview question that specifically tests edge cases in lambda captures?**