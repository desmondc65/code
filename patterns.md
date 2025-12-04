# Phase 2: Implementation Patterns

> **Goal**: Speed and Standard Library Mastery.

> **Harsh Truth**: If you implement a Linked List from scratch when you could have used `std::list` (unless explicitly asked to), you failed the "Knowledge of Tools" competency.

---

## 1. Essential STL Mastery

### Vectors: `reserve()` vs `resize()`

Understanding these is crucial for performance-critical code.

| Method | Effect | Size After | Capacity After | Elements |
|--------|--------|------------|----------------|----------|
| `reserve(n)` | Allocates memory only | Unchanged | ≥ n | Unchanged |
| `resize(n)` | Changes actual size | n | ≥ n | Added/removed |

```cpp
#include <vector>

std::vector<int> vec;

// reserve(): Pre-allocate memory WITHOUT creating elements
vec.reserve(1000);
// vec.size() == 0, vec.capacity() >= 1000
// vec[0] = 5;  // ❌ UNDEFINED BEHAVIOR! No elements exist

// resize(): Actually create elements
vec.resize(1000);
// vec.size() == 1000, vec.capacity() >= 1000
// vec[0] = 5;  // ✅ OK, element exists (default initialized to 0)

vec.resize(500);   // Shrinks size, destroys last 500 elements
vec.resize(800);   // Grows size, new elements are default-initialized (0)
vec.resize(900, -1); // New elements initialized to -1
```

#### How Capacity Doubling Works

When you `push_back` beyond capacity, the vector:
1. Allocates new memory (typically 2x current capacity)
2. Moves/copies all elements to new memory
3. Deallocates old memory

```cpp
std::vector<int> vec;
// Typical growth pattern:
// capacity: 0 → 1 → 2 → 4 → 8 → 16 → 32 → ...

for (int i = 0; i < 1000; ++i) {
    vec.push_back(i);
    // Approximately log₂(1000) ≈ 10 reallocations
}

// ✅ BETTER: Pre-allocate if you know the size
std::vector<int> vec2;
vec2.reserve(1000);  // ONE allocation
for (int i = 0; i < 1000; ++i) {
    vec2.push_back(i);  // No reallocations!
}
```

**Interview Tip**: Always mention `reserve()` when building a vector of known size:
```cpp
std::vector<int> BuildResult(const std::vector<int>& input) {
    std::vector<int> result;
    result.reserve(input.size());  // Mention this! Shows awareness
    for (int x : input) {
        result.push_back(x * 2);
    }
    return result;
}
```

---

### Maps: `emplace()` vs `insert()` and Efficient Lookups

#### `emplace()` vs `insert()`

```cpp
#include <map>
#include <unordered_map>

std::map<std::string, std::string> dict;

// insert(): Requires constructing pair first
dict.insert(std::make_pair("key", "value"));  // Creates temporary pair
dict.insert({"key2", "value2"});              // Brace initialization

// emplace(): Constructs in-place (more efficient for complex types)
dict.emplace("key3", "value3");  // No temporary pair created

// For simple types (int, etc.), difference is negligible
// For complex types with expensive constructors, emplace() can be faster
```

#### Handling Missing Keys: Avoid Double-Lookup!

```cpp
std::unordered_map<std::string, int> word_count;

// ❌ BAD: Double lookup pattern
if (word_count.count("hello") == 0) {  // First lookup
    word_count["hello"] = 0;           // Second lookup (insert)
}
word_count["hello"]++;                 // Third lookup!

// ❌ ALSO BAD: operator[] creates entry with default value
int count = word_count["world"];  // If "world" doesn't exist, inserts {"world", 0}
// This might not be what you want!

// ✅ GOOD: Single lookup with find()
auto it = word_count.find("hello");
if (it != word_count.end()) {
    it->second++;  // Use the iterator
} else {
    word_count.emplace("hello", 1);
}

// ✅ BEST: Use try_emplace() (C++17) - single operation
auto [it, inserted] = word_count.try_emplace("hello", 1);
if (!inserted) {
    it->second++;  // Already existed, increment
}

// ✅ For simple counting, operator[] is actually fine:
word_count["hello"]++;  // OK: default-constructs to 0, then increments
```

#### `try_emplace()` vs `emplace()` vs `insert_or_assign()`

```cpp
std::map<std::string, std::string> m;

// emplace(): Constructs the value even if key exists (wasteful)
m.emplace("key", "expensive_to_construct_value");
m.emplace("key", "another_value");  // Value constructed but discarded

// try_emplace(): Only constructs value if key doesn't exist
m.try_emplace("key", "value");      // Inserted
m.try_emplace("key", "new_value");  // NOT inserted, "new_value" never constructed

// insert_or_assign(): Insert or update (like operator[] but returns info)
auto [it, inserted] = m.insert_or_assign("key", "updated_value");
// inserted == false (key existed), value is now "updated_value"
```

---

### Strings: Modern Methods and `std::string_view`

#### Essential `std::string` Methods

```cpp
#include <string>

std::string s = "Hello, World!";

// Finding substrings
size_t pos = s.find("World");      // Returns 7, or std::string::npos if not found
size_t pos2 = s.find('o');         // First 'o' at position 4
size_t pos3 = s.rfind('o');        // Last 'o' at position 8
size_t pos4 = s.find("xyz");       // Returns std::string::npos

// Always check for npos!
if (pos != std::string::npos) {
    // Found it at position 'pos'
}

// Substring extraction
std::string sub1 = s.substr(7);      // "World!" (from pos 7 to end)
std::string sub2 = s.substr(0, 5);   // "Hello" (from pos 0, length 5)

// Comparison
if (s.starts_with("Hello")) { /* C++20 */ }
if (s.ends_with("!")) { /* C++20 */ }

// Before C++20:
if (s.compare(0, 5, "Hello") == 0) { /* starts with "Hello" */ }
if (s.size() >= 1 && s.compare(s.size() - 1, 1, "!") == 0) { /* ends with "!" */ }

// Modification
s.replace(7, 5, "Universe");  // "Hello, Universe!"
s.erase(5, 2);                // "HelloUniverse!"
s.insert(5, ", ");            // "Hello, Universe!"

// Useful for parsing
std::string line = "apple,banana,cherry";
size_t start = 0;
size_t end = line.find(',');
while (end != std::string::npos) {
    std::string token = line.substr(start, end - start);
    start = end + 1;
    end = line.find(',', start);
}
std::string last_token = line.substr(start);  // Don't forget the last one!
```

#### `std::string_view` (C++17) — Read-Only, Zero-Copy

**What is it?** A non-owning view into a string. No memory allocation, no copying.

```cpp
#include <string_view>

// ❌ BAD: Copies the string for a read-only operation
void PrintLength(const std::string& s) {
    std::cout << s.length();
}

PrintLength("Hello");  // Creates temporary std::string from literal!

// ✅ GOOD: No allocation, works with string literals, std::string, or char*
void PrintLength(std::string_view s) {
    std::cout << s.length();
}

PrintLength("Hello");                  // No copy, views the literal directly
PrintLength(std::string("Hello"));     // No copy, views the string's data
PrintLength(char_ptr);                 // No copy

// string_view has most read-only string operations
std::string_view sv = "Hello, World!";
sv.substr(0, 5);     // Returns string_view "Hello" (no allocation!)
sv.find("World");    // Works just like string
sv.remove_prefix(7); // Now sv == "World!" (mutates the view, not the data)
```

**Warning: Dangling Views**
```cpp
// ❌ DANGER: string_view doesn't own the data!
std::string_view GetView() {
    std::string s = "temporary";
    return s;  // DANGLING! s is destroyed, view points to garbage
}

// ✅ SAFE: Return string if you need ownership
std::string GetString() {
    std::string s = "temporary";
    return s;  // OK, moved out
}
```

**Interview Pattern: Use `string_view` for function parameters**
```cpp
// ✅ Google-style: string_view for read-only string parameters
bool ContainsDigit(std::string_view s) {
    for (char c : s) {
        if (std::isdigit(c)) return true;
    }
    return false;
}
```

---

## 2. Algorithms Library (`<algorithm>`)

> **Rule**: Don't reinvent the wheel. If you're writing a loop, ask yourself: "Is there an algorithm for this?"

### `std::sort` — IntroSort (Hybrid Algorithm)

```cpp
#include <algorithm>
#include <vector>

std::vector<int> nums = {5, 2, 8, 1, 9};

// Basic sort (ascending)
std::sort(nums.begin(), nums.end());

// Descending order
std::sort(nums.begin(), nums.end(), std::greater<int>());

// Custom comparator
std::sort(nums.begin(), nums.end(), [](int a, int b) {
    return a > b;  // Descending
});

// Sort by custom criteria (e.g., by absolute value)
std::sort(nums.begin(), nums.end(), [](int a, int b) {
    return std::abs(a) < std::abs(b);
});
```

**What is IntroSort?**
- Starts with QuickSort (fast average case)
- Switches to HeapSort if recursion gets too deep (prevents O(n²) worst case)
- Uses InsertionSort for small partitions (cache-friendly)
- **Complexity**: O(n log n) guaranteed, not stable

```cpp
// For stable sort (preserves relative order of equal elements):
std::stable_sort(nums.begin(), nums.end());

// Partial sort (only sort first k elements)
std::partial_sort(nums.begin(), nums.begin() + k, nums.end());

// nth_element: Find the nth smallest, partition around it (O(n) average)
std::nth_element(nums.begin(), nums.begin() + k, nums.end());
// nums[k] is now the (k+1)th smallest element
// Elements before k are smaller, elements after are larger (not sorted!)
```

---

### `std::lower_bound` / `std::upper_bound` — Binary Search

**These work on SORTED ranges only!**

```cpp
#include <algorithm>
#include <vector>

std::vector<int> sorted = {1, 2, 4, 4, 4, 6, 8};
//                         0  1  2  3  4  5  6

// lower_bound: First element >= target
auto lb = std::lower_bound(sorted.begin(), sorted.end(), 4);
// Points to index 2 (first 4)

// upper_bound: First element > target
auto ub = std::upper_bound(sorted.begin(), sorted.end(), 4);
// Points to index 5 (the 6)

// Count occurrences of 4:
int count = ub - lb;  // 3 occurrences

// Check if element exists:
if (lb != sorted.end() && *lb == 4) {
    // 4 exists in the vector
}

// Binary search (just checks existence):
bool found = std::binary_search(sorted.begin(), sorted.end(), 4);

// equal_range: Returns both bounds at once
auto [lo, hi] = std::equal_range(sorted.begin(), sorted.end(), 4);
```

**Common Interview Pattern: Insert Position**
```cpp
// Where would 5 go to maintain sorted order?
auto it = std::lower_bound(sorted.begin(), sorted.end(), 5);
// it points to index 5 (the 6), so 5 would be inserted at index 5

// Insert while maintaining order:
sorted.insert(std::lower_bound(sorted.begin(), sorted.end(), 5), 5);
```

**With Custom Comparator:**
```cpp
struct Person {
    std::string name;
    int age;
};

std::vector<Person> people = {{"Alice", 25}, {"Bob", 30}, {"Charlie", 35}};

// Find first person with age >= 30
auto it = std::lower_bound(people.begin(), people.end(), 30,
    [](const Person& p, int age) { return p.age < age; });
// Note: comparator takes (element, value), not (element, element)!
```

---

### `std::next_permutation` — Generate Permutations

Generates the next lexicographically greater permutation. Returns `false` when we've wrapped around.

```cpp
#include <algorithm>
#include <vector>
#include <string>

// Generate all permutations of a vector
std::vector<int> nums = {1, 2, 3};
std::sort(nums.begin(), nums.end());  // MUST start sorted for all permutations!

do {
    // Process permutation: {1,2,3}, {1,3,2}, {2,1,3}, {2,3,1}, {3,1,2}, {3,2,1}
    for (int n : nums) std::cout << n << " ";
    std::cout << "\n";
} while (std::next_permutation(nums.begin(), nums.end()));

// Works with strings too!
std::string s = "abc";
std::sort(s.begin(), s.end());
do {
    std::cout << s << "\n";  // abc, acb, bac, bca, cab, cba
} while (std::next_permutation(s.begin(), s.end()));

// Previous permutation also exists:
std::prev_permutation(nums.begin(), nums.end());
```

**Interview Use Case: Find K-th Permutation**
```cpp
std::string GetKthPermutation(int n, int k) {
    std::string s;
    for (int i = 1; i <= n; ++i) s += ('0' + i);
    
    for (int i = 1; i < k; ++i) {  // k-1 advances
        std::next_permutation(s.begin(), s.end());
    }
    return s;
}
// Note: For large k, there's a mathematical O(n) solution
```

---

### `std::unique` — Remove Consecutive Duplicates

**Important**: `unique` doesn't actually remove elements. It moves unique elements to the front and returns an iterator to the new logical end.

```cpp
#include <algorithm>
#include <vector>

std::vector<int> nums = {1, 1, 2, 2, 2, 3, 1};

// unique only removes CONSECUTIVE duplicates
auto new_end = std::unique(nums.begin(), nums.end());
// nums is now: {1, 2, 3, 1, ?, ?, ?}  (last 3 elements are garbage)
// new_end points to the first garbage element

// MUST erase to actually remove:
nums.erase(new_end, nums.end());
// nums is now: {1, 2, 3, 1}

// To remove ALL duplicates, sort first:
std::vector<int> nums2 = {1, 1, 2, 2, 2, 3, 1};
std::sort(nums2.begin(), nums2.end());      // {1, 1, 1, 2, 2, 2, 3}
nums2.erase(std::unique(nums2.begin(), nums2.end()), nums2.end());
// nums2 is now: {1, 2, 3}
```

**Idiomatic "Sort and Unique" Pattern:**
```cpp
// One-liner to sort and remove duplicates:
auto& v = nums;
std::sort(v.begin(), v.end());
v.erase(std::unique(v.begin(), v.end()), v.end());
```

---

### Other Essential Algorithms

```cpp
#include <algorithm>
#include <numeric>  // for accumulate, iota

std::vector<int> v = {1, 2, 3, 4, 5};

// std::accumulate - Sum (or fold with any operation)
int sum = std::accumulate(v.begin(), v.end(), 0);  // 15
int product = std::accumulate(v.begin(), v.end(), 1, std::multiplies<int>());  // 120

// std::count / std::count_if
int twos = std::count(v.begin(), v.end(), 2);  // 1
int evens = std::count_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; });  // 2

// std::find / std::find_if
auto it = std::find(v.begin(), v.end(), 3);
auto it2 = std::find_if(v.begin(), v.end(), [](int x) { return x > 3; });

// std::all_of / std::any_of / std::none_of
bool all_positive = std::all_of(v.begin(), v.end(), [](int x) { return x > 0; });
bool has_even = std::any_of(v.begin(), v.end(), [](int x) { return x % 2 == 0; });

// std::transform - Map operation
std::vector<int> squared(v.size());
std::transform(v.begin(), v.end(), squared.begin(), [](int x) { return x * x; });

// std::reverse
std::reverse(v.begin(), v.end());  // In-place

// std::rotate - Rotate left by n positions
std::rotate(v.begin(), v.begin() + 2, v.end());  // {3,4,5,1,2}

// std::iota - Fill with incrementing values
std::vector<int> indices(10);
std::iota(indices.begin(), indices.end(), 0);  // {0,1,2,3,4,5,6,7,8,9}

// std::min_element / std::max_element
auto min_it = std::min_element(v.begin(), v.end());
auto max_it = std::max_element(v.begin(), v.end());

// std::minmax_element - Both at once
auto [min_iter, max_iter] = std::minmax_element(v.begin(), v.end());

// std::copy / std::copy_if
std::vector<int> evens_only;
std::copy_if(v.begin(), v.end(), std::back_inserter(evens_only),
             [](int x) { return x % 2 == 0; });

// std::fill
std::fill(v.begin(), v.end(), 0);  // All zeros

// std::swap
std::swap(v[0], v[1]);  // Swap two elements
```

---

## 3. Code Snippet: The "Google Style" Setup

### Competitive Programming vs Google Interview Style

```cpp
// ❌ BAD: Competitive Programming Style
#include <bits/stdc++.h>    // Non-standard, includes everything
using namespace std;        // Pollutes global namespace

int main() {
    int n; cin >> n;
    vector<int> a(n);
    for(int i=0;i<n;i++) cin>>a[i];  // Cramped, no spaces
    sort(a.begin(),a.end());
    cout << a[0];
    return 0;
}
```

**Problems:**
1. `bits/stdc++.h` is non-portable (GCC only)
2. `using namespace std` can cause name collisions
3. Poor readability, no error handling
4. Using `int` for size (should be `size_t`)

```cpp
// ✅ GOOD: Google Interview Style
#include <algorithm>
#include <iostream>
#include <optional>
#include <vector>

// PascalCase for function names
std::optional<int> FindMinimum(const std::vector<int>& data) {
    // snake_case for local variables
    if (data.empty()) {
        return std::nullopt;  // Explicit error handling
    }
    
    // Use algorithms instead of raw loops when appropriate
    auto min_element = std::min_element(data.begin(), data.end());
    return *min_element;
}

int main() {
    std::vector<int> numbers = {5, 2, 8, 1, 9};
    
    auto result = FindMinimum(numbers);
    if (result.has_value()) {
        std::cout << "Minimum: " << result.value() << "\n";
    } else {
        std::cout << "Error: empty input\n";
    }
    
    return 0;
}
```

### Complete Example: Google-Style Solution

```cpp
#pragma once

#include <algorithm>
#include <optional>
#include <string>
#include <string_view>
#include <unordered_map>
#include <vector>

// Function to find target in sorted array using binary search
// Returns index if found, std::nullopt otherwise
std::optional<size_t> BinarySearch(const std::vector<int>& sorted_data, 
                                    int target) {
    auto it = std::lower_bound(sorted_data.begin(), sorted_data.end(), target);
    
    if (it != sorted_data.end() && *it == target) {
        return std::distance(sorted_data.begin(), it);
    }
    return std::nullopt;
}

// Function to count word frequencies in a text
// Uses string_view for read-only input, unordered_map for O(1) lookups
std::unordered_map<std::string, int> CountWordFrequencies(
    std::string_view text) {
    
    std::unordered_map<std::string, int> frequency_map;
    
    size_t start = 0;
    size_t end = text.find(' ');
    
    while (end != std::string_view::npos) {
        auto word = text.substr(start, end - start);
        if (!word.empty()) {
            frequency_map[std::string(word)]++;  // string_view → string for key
        }
        start = end + 1;
        end = text.find(' ', start);
    }
    
    // Don't forget the last word
    auto last_word = text.substr(start);
    if (!last_word.empty()) {
        frequency_map[std::string(last_word)]++;
    }
    
    return frequency_map;
}

// Function to find top K frequent elements
// Uses partial_sort for efficiency when k << n
std::vector<std::pair<std::string, int>> TopKFrequent(
    const std::unordered_map<std::string, int>& frequency_map,
    size_t k) {
    
    std::vector<std::pair<std::string, int>> pairs(
        frequency_map.begin(), frequency_map.end());
    
    // Clamp k to valid range
    k = std::min(k, pairs.size());
    
    // partial_sort is more efficient than full sort when k << n
    std::partial_sort(pairs.begin(), pairs.begin() + k, pairs.end(),
        [](const auto& a, const auto& b) {
            return a.second > b.second;  // Descending by frequency
        });
    
    pairs.resize(k);
    return pairs;
}
```

### Interview Template

Use this mental template when starting any problem:

```cpp
#include <vector>
#include <string>
#include <algorithm>
#include <optional>
// Add other headers as needed

class Solution {
 public:
    // PascalCase method name
    // const& for read-only inputs
    // Return optional or bool+output param for error cases
    std::optional<ReturnType> SolveProblem(const InputType& input) {
        // 1. Handle edge cases first
        if (input.empty()) {
            return std::nullopt;
        }
        
        // 2. Use descriptive snake_case variable names
        size_t input_size = input.size();
        
        // 3. Prefer STL algorithms over raw loops
        // 4. Add brief comments for non-obvious logic
        
        return result;
    }

 private:
    // Private helpers use same naming conventions
    bool IsValid(const Item& item) const {
        return item.value_ > 0;
    }
    
    // Private members with trailing underscore
    int cache_size_ = 0;
};
```

---

## Quick Reference: STL Algorithm Complexity

| Algorithm | Average | Worst | Notes |
|-----------|---------|-------|-------|
| `sort` | O(n log n) | O(n log n) | IntroSort, not stable |
| `stable_sort` | O(n log n) | O(n log² n) | Merge sort, stable |
| `partial_sort` | O(n log k) | O(n log k) | HeapSort variant |
| `nth_element` | O(n) | O(n²) | QuickSelect, usually O(n) |
| `lower_bound` | O(log n) | O(log n) | Binary search |
| `find` | O(n) | O(n) | Linear search |
| `unique` | O(n) | O(n) | Consecutive only |
| `next_permutation` | O(n) | O(n) | Per call |
| `reverse` | O(n) | O(n) | In-place |
| `accumulate` | O(n) | O(n) | Linear fold |

---

## Common Mistakes to Avoid

```cpp
// ❌ Mistake 1: Forgetting to sort before unique
std::vector<int> v = {1, 2, 1, 2};
v.erase(std::unique(v.begin(), v.end()), v.end());
// v is {1, 2, 1, 2} - NOT what you wanted!

// ❌ Mistake 2: Using lower_bound on unsorted data
std::vector<int> unsorted = {5, 2, 8, 1};
auto it = std::lower_bound(unsorted.begin(), unsorted.end(), 3);
// Undefined behavior! Must be sorted.

// ❌ Mistake 3: Not checking iterator validity
std::vector<int> v = {1, 2, 3};
auto it = std::find(v.begin(), v.end(), 99);
int val = *it;  // Crash! it == v.end()

// ❌ Mistake 4: Iterator invalidation after insert/erase
std::vector<int> v = {1, 2, 3};
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it == 2) {
        v.erase(it);  // 'it' is now invalid!
    }
}

// ✅ Correct way:
for (auto it = v.begin(); it != v.end(); ) {
    if (*it == 2) {
        it = v.erase(it);  // erase returns next valid iterator
    } else {
        ++it;
    }
}

// ✅ Even better with remove-erase idiom:
v.erase(std::remove(v.begin(), v.end(), 2), v.end());

// ❌ Mistake 5: Double lookup in maps
if (map.count(key)) {
    map[key]++;  // Two lookups!
}

// ✅ Correct:
auto it = map.find(key);
if (it != map.end()) {
    it->second++;  // Single lookup
}
```
