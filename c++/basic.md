# Phase 1: The "Google C++" Standard

> **Goal**: Stop writing "Competitive Programming C++" and start writing "Production C++".

> **Crucial Reality**: If you write `using namespace std;` or manually manage memory with `new` and `delete`, you are signalling that you are a junior engineer or stuck in 1998.

---

## 1. The Google Style Guide (Non-Negotiable)

Google has a specific [Open Source Style Guide](https://google.github.io/styleguide/cppguide.html). You don't need to memorize it all, but you **must** adhere to these during the interview:

### No Exceptions

Google code is generally **exception-free**. Do not use `try`, `catch`, or `throw`.

**Why?**
- Exceptions introduce hidden control flow that's hard to reason about
- They complicate resource management and can cause memory leaks if not handled properly
- Performance overhead: exception handling adds runtime cost even when no exception is thrown
- Google's massive codebase predates modern exception safety, and retrofitting is impractical

**Instead**: Return status codes (`bool`, `std::optional`, or pointers).

```cpp
// ❌ BAD: Using exceptions
int ParseInt(const std::string& s) {
    if (s.empty()) throw std::invalid_argument("empty string");
    return std::stoi(s);
}

// ✅ GOOD: Return std::optional
std::optional<int> ParseInt(const std::string& s) {
    if (s.empty()) return std::nullopt;
    try {
        return std::stoi(s);  // stoi can still throw internally
    } catch (...) {
        return std::nullopt;
    }
}

// ✅ GOOD: Return bool with output parameter
bool ParseInt(const std::string& s, int* result) {
    if (s.empty() || result == nullptr) return false;
    // parsing logic...
    *result = parsed_value;
    return true;
}
```

### Variable Naming

| Element | Convention | Example |
|---------|------------|---------|
| Local variables | `snake_case` | `int user_count = 0;` |
| Function arguments | `snake_case` | `void Process(int input_value)` |
| Functions | `PascalCase` | `void CalculateTotal()` |
| Classes | `PascalCase` | `class UserManager {}` |
| Private members | `snake_case_` | `int user_count_;` |
| Constants | `kPascalCase` | `const int kMaxSize = 100;` |

```cpp
class UserManager {
 public:
    // PascalCase for methods
    void AddUser(const std::string& user_name) {  // snake_case for params
        int local_count = 0;  // snake_case for locals
        user_count_++;        // trailing underscore for private members
    }
    
    int GetUserCount() const { return user_count_; }

 private:
    int user_count_ = 0;      // trailing underscore mandatory
    std::string database_path_;
};
```

### Headers

Always use `#pragma once` or standard include guards.

```cpp
// Option 1: #pragma once (preferred, simpler)
#pragma once

class MyClass { /* ... */ };

// Option 2: Traditional include guards
#ifndef PROJECT_PATH_FILE_H_
#define PROJECT_PATH_FILE_H_

class MyClass { /* ... */ };

#endif  // PROJECT_PATH_FILE_H_
```

---

## 2. Modern C++ Mandates

### Smart Pointers: NEVER use `malloc`/`free`. Rarely use `new`/`delete`.

**Why?**
- Manual memory management is error-prone (leaks, double-free, use-after-free)
- Smart pointers provide automatic cleanup via RAII (Resource Acquisition Is Initialization)
- They make ownership semantics explicit and self-documenting

#### `std::unique_ptr` — Default Choice (Exclusive Ownership)

```cpp
#include <memory>

// ❌ BAD: Raw pointer, manual management
Node* node = new Node(42);
// ... code that might return early or throw ...
delete node;  // Easy to forget, leak if exception thrown

// ✅ GOOD: unique_ptr, automatic cleanup
auto node = std::make_unique<Node>(42);
// No delete needed! Automatically cleaned up when out of scope

// Ownership transfer
std::unique_ptr<Node> other = std::move(node);  // node is now nullptr
```

**Key Properties of `unique_ptr`:**
- Zero overhead compared to raw pointer (no reference counting)
- Cannot be copied, only moved
- Use when there's exactly one owner

#### `std::shared_ptr` — Only When Ownership is Truly Shared

```cpp
#include <memory>

// Use when multiple objects need to keep something alive
auto shared_data = std::make_shared<Data>();
auto another_ref = shared_data;  // Reference count = 2

// Both can use the data
shared_data->Process();
another_ref->Process();

// Data is deleted only when BOTH go out of scope
```

**Key Properties of `shared_ptr`:**
- Reference counting overhead (not free!)
- Thread-safe reference count (but not thread-safe access to the object)
- Use sparingly: usually indicates unclear ownership design

#### `std::weak_ptr` — Breaking Cycles

```cpp
// Problem: Circular references with shared_ptr = memory leak
struct Node {
    std::shared_ptr<Node> next;  // Creates cycle if A->B->A
};

// Solution: Use weak_ptr for back-references
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;    // Doesn't contribute to reference count
    
    void AccessPrev() {
        if (auto locked = prev.lock()) {  // Returns shared_ptr or nullptr
            locked->DoSomething();
        }
    }
};
```

### Casts: No C-style Casts

**Why?**
- C-style casts are dangerous: `(int)x` can do multiple things silently
- Modern casts are explicit about intent and safer

```cpp
double d = 3.14;

// ❌ BAD: C-style cast (ambiguous, dangerous)
int i = (int)d;

// ✅ GOOD: static_cast (compile-time, safe conversions)
int i = static_cast<int>(d);

// Other casts for specific purposes:
// const_cast     - Remove/add const (use rarely!)
// dynamic_cast   - Safe downcast with runtime check (requires RTTI)
// reinterpret_cast - Bit-level reinterpretation (dangerous, low-level)
```

| Cast | Purpose | Example |
|------|---------|---------|
| `static_cast` | Normal conversions (numeric, up/down class hierarchy) | `static_cast<int>(3.14)` |
| `const_cast` | Add/remove `const` | `const_cast<char*>(str)` |
| `dynamic_cast` | Safe polymorphic downcast | `dynamic_cast<Derived*>(base_ptr)` |
| `reinterpret_cast` | Reinterpret bits | `reinterpret_cast<char*>(&int_val)` |

### Const Correctness

**Rule**: If something shouldn't change, make it `const`. This is not optional.

```cpp
class Rectangle {
 public:
    Rectangle(int width, int height) 
        : width_(width), height_(height) {}
    
    // ✅ Method doesn't modify state → mark const
    int GetArea() const { return width_ * height_; }
    
    // ✅ Returns const reference to prevent modification
    const std::string& GetName() const { return name_; }
    
    // This method modifies state, so NO const
    void SetWidth(int width) { width_ = width; }

 private:
    int width_;
    int height_;
    std::string name_;
};

void ProcessRectangle(const Rectangle& rect) {  // const ref = won't modify
    int area = rect.GetArea();  // OK: GetArea is const
    // rect.SetWidth(10);       // ERROR: can't call non-const method
}
```

**Const with Pointers:**
```cpp
int value = 42;

const int* ptr1 = &value;       // Pointer to const int (can't modify *ptr1)
int* const ptr2 = &value;       // Const pointer to int (can't modify ptr2)
const int* const ptr3 = &value; // Const pointer to const int (can't modify either)

// Read right-to-left: "ptr1 is a pointer to a const int"
```

---

## 3. Google C++ Checklist — Detailed Explanations

### ✅ Why `std::vector` is Preferred Over `std::list` (Cache Locality)

**TL;DR**: `vector` is almost always faster due to **cache locality**, even for operations where `list` has better Big-O complexity.

#### Memory Layout Comparison

```
std::vector<int>:
┌────┬────┬────┬────┬────┬────┐
│ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │  ← Contiguous memory
└────┴────┴────┴────┴────┴────┘
  All elements in same cache line(s)

std::list<int>:
┌────┐     ┌────┐     ┌────┐
│ 1  │────▶│ 2  │────▶│ 3  │    ← Scattered in memory
└────┘     └────┘     └────┘
  Each node potentially in different cache line
```

#### Why Cache Locality Matters

Modern CPUs don't fetch individual bytes; they fetch **cache lines** (typically 64 bytes). When you access element 0 of a vector, elements 1-15 are already in cache (free!).

```cpp
// Iterating through 1 million integers:

std::vector<int> vec(1000000);
for (int x : vec) { /* process */ }
// Fast: Sequential memory access, CPU prefetcher works perfectly

std::list<int> lst(1000000);
for (int x : lst) { /* process */ }
// Slow: Each node requires pointer chase, cache miss on every access
```

**Benchmark Reality**: Even middle insertion (where `list` is O(1) vs `vector`'s O(n)) is often faster with `vector` for sizes under ~1000 elements due to cache effects.

#### When to Use `std::list`

Almost never. Consider `list` only when:
1. You need **stable iterators** (iterators remain valid after insert/erase)
2. You're doing **constant insertions/deletions at known positions** with iterator already available
3. Elements are **very large** and copying is expensive (but even then, consider `vector<unique_ptr<T>>`)

```cpp
// ✅ Default choice: vector
std::vector<int> data;

// Rare case: Need stable iterators
std::list<int> data;
auto it = data.insert(data.begin(), 42);
data.push_back(100);  // 'it' still valid, points to 42
```

---

### ✅ How to Use `std::move` to Avoid Expensive Copies

**What is `std::move`?**
- It's a **cast** to an rvalue reference, signaling "I'm done with this, you can steal its resources"
- It enables **move semantics**: transferring ownership instead of copying

```cpp
#include <string>
#include <vector>
#include <utility>  // for std::move

void ProcessString(std::string s) { /* ... */ }

std::string CreateLargeString() {
    std::string result(1000000, 'x');  // 1MB string
    return result;  // Automatically moved (Return Value Optimization)
}

int main() {
    std::string large = CreateLargeString();
    
    // ❌ Expensive copy (1MB copied)
    ProcessString(large);  // 'large' is still valid
    
    // ✅ Cheap move (just pointer swap, ~constant time)
    ProcessString(std::move(large));  // 'large' is now in "valid but unspecified state"
    // Don't use 'large' after this! (except to reassign or destroy)
}
```

#### Move Semantics with Containers

```cpp
std::vector<std::string> strings;
std::string s = "hello world";

// ❌ Copy: allocates new memory, copies characters
strings.push_back(s);

// ✅ Move: transfers ownership, s becomes empty
strings.push_back(std::move(s));

// Even better: construct in-place
strings.emplace_back("hello world");  // No temporary string created
```

#### Writing Move-Aware Classes

```cpp
class Buffer {
 public:
    // Constructor
    explicit Buffer(size_t size) : size_(size), data_(new int[size]) {}
    
    // Destructor
    ~Buffer() { delete[] data_; }
    
    // Copy constructor (expensive)
    Buffer(const Buffer& other) : size_(other.size_), data_(new int[other.size_]) {
        std::copy(other.data_, other.data_ + size_, data_);
    }
    
    // Move constructor (cheap!)
    Buffer(Buffer&& other) noexcept 
        : size_(other.size_), data_(other.data_) {
        other.size_ = 0;
        other.data_ = nullptr;  // Prevent double-delete
    }
    
    // Move assignment operator
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }

 private:
    size_t size_;
    int* data_;
};
```

**Rule of Five**: If you define any of destructor, copy constructor, copy assignment, move constructor, or move assignment, you should define all five.

---

### ✅ Custom Comparator for `std::priority_queue` (Min-Heap)

**Default behavior**: `std::priority_queue` is a **max-heap** (largest element at top).

```cpp
#include <queue>
#include <vector>
#include <functional>

// Default: Max-heap
std::priority_queue<int> max_heap;
max_heap.push(3);
max_heap.push(1);
max_heap.push(4);
max_heap.top();  // Returns 4

// Min-heap using std::greater
std::priority_queue<int, std::vector<int>, std::greater<int>> min_heap;
min_heap.push(3);
min_heap.push(1);
min_heap.push(4);
min_heap.top();  // Returns 1
```

#### Understanding the Comparator

The comparator answers: "Should `a` have **lower priority** than `b`?"
- `std::less<T>` (default): `a < b` means `a` has lower priority → **max-heap**
- `std::greater<T>`: `a > b` means `a` has lower priority → **min-heap**

#### Custom Comparator with Structs

```cpp
struct Task {
    int priority;
    std::string name;
};

// Method 1: Functor (most common)
struct TaskComparator {
    bool operator()(const Task& a, const Task& b) const {
        // Return true if a should come AFTER b (lower priority)
        return a.priority > b.priority;  // Min-heap by priority
    }
};

std::priority_queue<Task, std::vector<Task>, TaskComparator> task_queue;

// Method 2: Lambda (C++20 or with decltype trick)
auto cmp = [](const Task& a, const Task& b) {
    return a.priority > b.priority;
};
std::priority_queue<Task, std::vector<Task>, decltype(cmp)> task_queue2(cmp);

// Method 3: Define operator< on the struct (for max-heap)
struct Task2 {
    int priority;
    std::string name;
    
    bool operator<(const Task2& other) const {
        return priority < other.priority;  // Max-heap by priority
    }
};
std::priority_queue<Task2> task_queue3;  // Uses operator< by default
```

#### Common Interview Pattern: Top K Elements

```cpp
// Find K largest elements using min-heap of size K
std::vector<int> TopK(const std::vector<int>& nums, int k) {
    // Min-heap: smallest of the "top K" is at the top
    std::priority_queue<int, std::vector<int>, std::greater<int>> min_heap;
    
    for (int num : nums) {
        min_heap.push(num);
        if (min_heap.size() > k) {
            min_heap.pop();  // Remove smallest
        }
    }
    
    std::vector<int> result;
    while (!min_heap.empty()) {
        result.push_back(min_heap.top());
        min_heap.pop();
    }
    return result;  // K largest elements
}
```

---

### ✅ `std::map` vs `std::unordered_map`

| Feature | `std::map` | `std::unordered_map` |
|---------|-----------|---------------------|
| Implementation | Red-Black Tree | Hash Table |
| Lookup | O(log n) | O(1) average, O(n) worst |
| Insert | O(log n) | O(1) average, O(n) worst |
| Delete | O(log n) | O(1) average, O(n) worst |
| Ordered? | ✅ Yes (sorted by key) | ❌ No |
| Memory | Less overhead | More overhead (buckets) |
| Iterator invalidation | Stable | Can invalidate on rehash |

#### When to Use Each

```cpp
#include <map>
#include <unordered_map>

// ✅ Use unordered_map for most cases (faster lookups)
std::unordered_map<std::string, int> word_count;
word_count["hello"]++;
if (word_count.count("hello")) { /* exists */ }

// ✅ Use map when you need:
// 1. Ordered iteration
std::map<std::string, int> sorted_scores;
sorted_scores["alice"] = 100;
sorted_scores["bob"] = 95;
for (const auto& [name, score] : sorted_scores) {
    // Iterates in alphabetical order: alice, bob
}

// 2. Range queries
auto it = sorted_scores.lower_bound("b");  // First key >= "b"
auto end = sorted_scores.upper_bound("c"); // First key > "c"
// Iterate from "b" to "c"

// 3. Custom comparison
std::map<std::string, int, std::greater<std::string>> reverse_sorted;
```

#### Hash Function for Custom Types

```cpp
struct Point {
    int x, y;
    
    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
};

// Custom hash function
struct PointHash {
    size_t operator()(const Point& p) const {
        // Combine hashes (simple approach)
        return std::hash<int>()(p.x) ^ (std::hash<int>()(p.y) << 1);
    }
};

std::unordered_map<Point, std::string, PointHash> point_names;
point_names[{1, 2}] = "A";
```

#### Performance Gotcha: Hash Collisions

```cpp
// Worst case: All keys hash to same bucket → O(n) per operation
// This can happen with adversarial input!

// Good practice: Use std::unordered_map for internal data,
// but be cautious with user-provided keys (potential DoS vector)
```

---

## Quick Reference Card

```cpp
// ✅ GOOD: Modern Google-style C++
#pragma once
#include <memory>
#include <optional>
#include <string>
#include <vector>

class DataProcessor {
 public:
    explicit DataProcessor(std::string name) : name_(std::move(name)) {}
    
    std::optional<int> ProcessValue(int input_value) const {
        if (input_value < 0) return std::nullopt;
        return input_value * 2;
    }
    
    void AddData(std::unique_ptr<Data> data) {
        data_items_.push_back(std::move(data));
    }

 private:
    std::string name_;
    std::vector<std::unique_ptr<Data>> data_items_;
};

// ❌ BAD: Old-style C++
#ifndef DATA_PROCESSOR_H
#define DATA_PROCESSOR_H
using namespace std;  // NEVER DO THIS

class data_processor {  // Wrong naming
public:
    data_processor(string name) {  // Missing explicit, copy instead of move
        name_ = name;
    }
    
    int process_value(int inputValue) {  // Wrong naming, not const, no error handling
        if (inputValue < 0) throw runtime_error("negative");  // No exceptions!
        return inputValue * 2;
    }
    
    void add_data(Data* data) {  // Raw pointer, unclear ownership
        data_items.push_back(data);
    }
    
private:
    string name_;  // OK
    vector<Data*> data_items;  // Memory leak waiting to happen
};
#endif
```
