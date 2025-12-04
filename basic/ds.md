# Core Data Structures for Google Interviews

> This is the baseline knowledge required for a Google interview. Do not mistake "knowing what a Stack is" with "knowing how to implement it efficiently in C++ under pressure."

Google interviewers probe for **deep understanding** of memory, STL implementation details, and complexity analysis.

---

## 1. Array / Dynamic Array (`std::vector`)

In C++, you almost never use raw arrays (`int arr[10]`) in an interview context unless specifically asked for low-level memory manipulation. You use `std::vector`.

### Concept

Contiguous memory blocks. Elements stored sequentially in memory.

```
Memory Layout:
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│  1  │  2  │  3  │  4  │  5  │     │     │     │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  ▲                       ▲                   ▲
  │                       │                   │
begin()                 end()            capacity()
         size = 5              capacity = 8
```

### C++ STL

```cpp
#include <vector>

std::vector<int> nums;                    // Empty vector
std::vector<int> nums(10);                // 10 elements, default initialized (0)
std::vector<int> nums(10, -1);            // 10 elements, all -1
std::vector<int> nums = {1, 2, 3, 4, 5};  // Initializer list

// Common operations
nums.push_back(6);          // Add to end: O(1) amortized
nums.pop_back();            // Remove from end: O(1)
nums[0] = 10;               // Random access: O(1)
nums.size();                // Current size
nums.empty();               // Check if empty
nums.front();               // First element
nums.back();                // Last element
nums.clear();               // Remove all elements

// Capacity management
nums.reserve(1000);         // Pre-allocate (doesn't change size)
nums.resize(100);           // Change size (adds/removes elements)
nums.shrink_to_fit();       // Release unused capacity
```

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Access `v[i]` | $O(1)$ | Direct memory offset |
| `push_back` | Amortized $O(1)$ | $O(n)$ when reallocation needed |
| `pop_back` | $O(1)$ | No reallocation |
| Insert middle | $O(n)$ | Requires shifting elements |
| Erase middle | $O(n)$ | Requires shifting elements |
| `find` (linear) | $O(n)$ | Must scan |

### When to Use

- When you need **random access** (indexing)
- The **default choice** for Dynamic Programming (DP) tables
- When **memory locality** (cache friendliness) matters
- Almost always prefer over raw arrays

### The Google Standard

#### ❌ Do NOT use `insert`/`erase` in a loop without acknowledging $O(n^2)$

```cpp
// ❌ BAD: O(n²) - each erase is O(n), done n times
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) {
        it = v.erase(it);  // Shifts all elements after it
    } else {
        ++it;
    }
}

// ✅ GOOD: O(n) - remove-erase idiom
v.erase(std::remove_if(v.begin(), v.end(), 
    [](int x) { return x % 2 == 0; }), v.end());

// ✅ C++20: Even cleaner
std::erase_if(v, [](int x) { return x % 2 == 0; });
```

#### ✅ DO use `reserve()` when you know the size

```cpp
// ❌ BAD: Multiple reallocations as vector grows
std::vector<int> result;
for (int i = 0; i < 100000; ++i) {
    result.push_back(i);  // Reallocates ~17 times (capacity doubles)
}

// ✅ GOOD: Single allocation
std::vector<int> result;
result.reserve(100000);  // Pre-allocate
for (int i = 0; i < 100000; ++i) {
    result.push_back(i);  // No reallocations
}

// Interview tip: Always mention this!
// "I'll reserve space since we know the size, to avoid reallocations"
```

#### How Vectors Grow

```cpp
std::vector<int> v;
// Typical growth pattern:
// capacity: 0 → 1 → 2 → 4 → 8 → 16 → 32 → ...
// Each reallocation:
//   1. Allocate new buffer (2x size)
//   2. Move/copy all elements
//   3. Deallocate old buffer
// This is why push_back is "amortized O(1)" - expensive occasionally
```

### Common Patterns

```cpp
// 2D vector (matrix)
std::vector<std::vector<int>> matrix(rows, std::vector<int>(cols, 0));
matrix[i][j] = value;

// Use as stack
std::vector<int> stack;
stack.push_back(x);      // push
stack.back();            // top
stack.pop_back();        // pop

// Iterate with index
for (size_t i = 0; i < v.size(); ++i) {
    // v[i]
}

// Range-based (preferred when index not needed)
for (const auto& elem : v) {
    // elem
}

// Iterate with iterators (for algorithms)
for (auto it = v.begin(); it != v.end(); ++it) {
    // *it
}
```

---

## 2. Linked List

For LeetCode, you are rarely using `std::list` (which is a doubly linked list). You are usually manipulating a **custom struct** provided by the problem.

### Concept

Nodes containing data and a pointer to the next node. **Non-contiguous memory**.

```
Singly Linked List:
┌───┬───┐    ┌───┬───┐    ┌───┬───┐    ┌───┬───┐
│ 1 │ ●─┼───▶│ 2 │ ●─┼───▶│ 3 │ ●─┼───▶│ 4 │ ╳ │
└───┴───┘    └───┴───┘    └───┴───┘    └───┴───┘
  head                                   tail

Doubly Linked List (std::list):
     ┌───┬───┬───┐    ┌───┬───┬───┐    ┌───┬───┬───┐
╳◀───┤ ● │ 1 │ ●─┼───▶│ ● │ 2 │ ●─┼───▶│ ● │ 3 │ ╳ │
     └───┴───┴───┘◀───┴───┴───┴───┘◀───┴───┴───┴───┘
```

### C++ Implementation

```cpp
// Standard LeetCode definition
struct ListNode {
    int val;
    ListNode* next;
    ListNode() : val(0), next(nullptr) {}
    ListNode(int x) : val(x), next(nullptr) {}
    ListNode(int x, ListNode* next) : val(x), next(next) {}
};

// Create nodes
ListNode* head = new ListNode(1);
head->next = new ListNode(2);
head->next->next = new ListNode(3);

// Traverse
ListNode* curr = head;
while (curr != nullptr) {
    std::cout << curr->val << " ";
    curr = curr->next;
}
```

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Access by index | $O(n)$ | Must traverse from head |
| Insert at head | $O(1)$ | Just pointer manipulation |
| Insert at tail (no tail ptr) | $O(n)$ | Must traverse |
| Insert at position (with ptr) | $O(1)$ | Just pointer manipulation |
| Delete (with ptr to prev) | $O(1)$ | Just pointer manipulation |
| Search | $O(n)$ | Must traverse |

### When to Use

- When **constant-time insertion/deletion** is required at head or known position
- **Specific Patterns:**
  - Fast & Slow Pointers (Floyd's Cycle Detection)
  - Merging sorted lists
  - Reversing nodes in groups
  - LRU Cache (doubly linked list + hash map)

### The Google Standard

#### ✅ Master the "Dummy Head" Technique

Always use a dummy head node when the head might change.

```cpp
// ❌ BAD: Edge case spaghetti
ListNode* DeleteValue(ListNode* head, int val) {
    // Handle deleting head
    while (head != nullptr && head->val == val) {
        ListNode* temp = head;
        head = head->next;
        delete temp;
    }
    
    // Handle rest
    ListNode* curr = head;
    while (curr != nullptr && curr->next != nullptr) {
        if (curr->next->val == val) {
            ListNode* temp = curr->next;
            curr->next = curr->next->next;
            delete temp;
        } else {
            curr = curr->next;
        }
    }
    return head;
}

// ✅ GOOD: Dummy head eliminates edge cases
ListNode* DeleteValue(ListNode* head, int val) {
    ListNode dummy(0, head);  // Dummy on STACK (no memory leak)
    ListNode* curr = &dummy;
    
    while (curr->next != nullptr) {
        if (curr->next->val == val) {
            ListNode* to_delete = curr->next;
            curr->next = curr->next->next;
            delete to_delete;
        } else {
            curr = curr->next;
        }
    }
    return dummy.next;  // New head
}
```

#### Memory Management

```cpp
// In LeetCode: Memory leaks often ignored for speed
// In Google interview: ASK the interviewer!

// "Should I handle memory cleanup here, or focus on the algorithm?"

// If yes, use this pattern:
void DeleteList(ListNode* head) {
    while (head != nullptr) {
        ListNode* temp = head;
        head = head->next;
        delete temp;
    }
}
```

### Essential Patterns

#### Reverse a Linked List

```cpp
ListNode* ReverseList(ListNode* head) {
    ListNode* prev = nullptr;
    ListNode* curr = head;
    
    while (curr != nullptr) {
        ListNode* next = curr->next;  // Save next
        curr->next = prev;            // Reverse pointer
        prev = curr;                  // Move prev forward
        curr = next;                  // Move curr forward
    }
    return prev;  // New head
}
```

#### Detect Cycle (Floyd's Algorithm)

```cpp
bool HasCycle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    
    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;
    }
    return false;
}

// Find cycle start
ListNode* DetectCycleStart(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    
    // Find meeting point
    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) break;
    }
    
    if (fast == nullptr || fast->next == nullptr) {
        return nullptr;  // No cycle
    }
    
    // Move slow to head, advance both by 1
    slow = head;
    while (slow != fast) {
        slow = slow->next;
        fast = fast->next;
    }
    return slow;  // Cycle start
}
```

#### Find Middle Node

```cpp
ListNode* FindMiddle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    
    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
    }
    return slow;  // Middle (or second middle if even length)
}
```

---

## 3. Hash Table (`std::unordered_map` / `std::unordered_set`)

This is the **most critical data structure** for optimization.

> **Warning:** Do not confuse `unordered_map` (hash table) with `map` (Red-Black Tree)!

### Concept

Key-value pairs mapped via a hash function.

```
Hash Table Internals:
┌─────────────────────────────────────────────┐
│              Bucket Array                    │
├──────┬──────┬──────┬──────┬──────┬──────────┤
│  0   │  1   │  2   │  3   │  4   │   ...    │
├──────┼──────┼──────┼──────┼──────┼──────────┤
│      │  ●   │      │  ●   │  ●   │          │
│      │  │   │      │  │   │  │   │          │
│      │  ▼   │      │  ▼   │  ▼   │          │
│      │[K,V] │      │[K,V] │[K,V] │          │
│      │  │   │      │      │  │   │          │
│      │  ▼   │      │      │  ▼   │          │
│      │[K,V] │      │      │[K,V] │ Collision│
│      │      │      │      │      │  Chain   │
└──────┴──────┴──────┴──────┴──────┴──────────┘

hash(key) % bucket_count → bucket index
```

### C++ STL

```cpp
#include <unordered_map>
#include <unordered_set>

// Hash Map: Key → Value
std::unordered_map<std::string, int> word_count;
word_count["hello"] = 1;
word_count["world"]++;               // Default constructs to 0, then increments
word_count.insert({"foo", 42});
word_count.emplace("bar", 100);      // More efficient

// Access
int count = word_count["hello"];     // ⚠️ Inserts if not exists!
int count = word_count.at("hello");  // Throws if not exists

// Check existence
if (word_count.count("hello")) { /* exists */ }
if (word_count.contains("hello")) { /* C++20 */ }

// Find (single lookup)
auto it = word_count.find("hello");
if (it != word_count.end()) {
    std::cout << it->first << ": " << it->second;
}

// Erase
word_count.erase("hello");

// Hash Set: Just keys, no values
std::unordered_set<int> seen;
seen.insert(5);
seen.count(5);  // 1 if exists, 0 otherwise
seen.erase(5);
```

### Time Complexity

| Operation | Average | Worst Case |
|-----------|---------|------------|
| Insert | $O(1)$ | $O(n)$ |
| Search | $O(1)$ | $O(n)$ |
| Delete | $O(1)$ | $O(n)$ |

**Worst case** occurs with many hash collisions (pathological input).

### When to Use

- **Counting frequencies** (character → count)
- **Checking existence** (Two Sum problem)
- **Memoization** in DP (caching results)
- **De-duplication**
- **Graph adjacency list** (node → neighbors)

### The Google Standard

#### ✅ Know the Difference: `unordered_map` vs `map`

```cpp
// unordered_map: Hash Table - O(1) average
std::unordered_map<int, int> hash_map;  // ✅ For most cases

// map: Red-Black Tree - O(log n) always
std::map<int, int> tree_map;  // Use when you need sorted keys

// ❌ PERFORMANCE BUG: Using map when order doesn't matter
// This makes all operations O(log n) instead of O(1)!
```

| Feature | `unordered_map` | `map` |
|---------|-----------------|-------|
| Implementation | Hash Table | Red-Black Tree |
| Lookup | $O(1)$ avg | $O(\log n)$ |
| Insert | $O(1)$ avg | $O(\log n)$ |
| Ordered? | ❌ No | ✅ Yes (sorted) |
| Range queries | ❌ No | ✅ `lower_bound`, `upper_bound` |

#### Avoid Double Lookups

```cpp
std::unordered_map<std::string, int> m;

// ❌ BAD: Two lookups
if (m.count(key)) {      // First lookup
    m[key]++;            // Second lookup
}

// ❌ ALSO BAD: Three lookups!
if (m.count(key) == 0) { // First
    m[key] = 0;          // Second
}
m[key]++;                // Third

// ✅ GOOD: Single lookup
auto it = m.find(key);
if (it != m.end()) {
    it->second++;
} else {
    m.emplace(key, 1);
}

// ✅ EVEN BETTER for counting: Just use operator[]
m[key]++;  // Default-constructs to 0 if missing, then increments
           // This is actually fine for simple counting!

// ✅ BEST: try_emplace (C++17)
auto [it, inserted] = m.try_emplace(key, 1);
if (!inserted) {
    it->second++;
}
```

#### Custom Keys (Pairs, Tuples, Structs)

`unordered_map` doesn't natively support `std::pair` or `std::vector` as keys.

```cpp
// ❌ COMPILE ERROR: No hash function for pair
std::unordered_map<std::pair<int, int>, int> grid;  // Error!

// ✅ Solution 1: Use map (O(log n) but works)
std::map<std::pair<int, int>, int> grid;

// ✅ Solution 2: Custom hash function
struct PairHash {
    size_t operator()(const std::pair<int, int>& p) const {
        return std::hash<long long>()(
            (static_cast<long long>(p.first) << 32) | p.second
        );
    }
};
std::unordered_map<std::pair<int, int>, int, PairHash> grid;

// ✅ Solution 3: Encode as string (simple but slower)
std::unordered_map<std::string, int> grid;
grid[std::to_string(x) + "," + std::to_string(y)] = value;

// ✅ Solution 4: Encode as single integer (if bounds known)
// For grid coordinates 0 <= x, y < 10000:
std::unordered_map<int, int> grid;
auto encode = [](int x, int y) { return x * 10000 + y; };
grid[encode(x, y)] = value;
```

### Common Patterns

#### Two Sum (Classic)

```cpp
std::vector<int> TwoSum(const std::vector<int>& nums, int target) {
    std::unordered_map<int, int> num_to_index;
    
    for (int i = 0; i < nums.size(); ++i) {
        int complement = target - nums[i];
        auto it = num_to_index.find(complement);
        if (it != num_to_index.end()) {
            return {it->second, i};
        }
        num_to_index[nums[i]] = i;
    }
    return {};  // No solution
}
```

#### Frequency Count

```cpp
std::unordered_map<char, int> CountFrequency(const std::string& s) {
    std::unordered_map<char, int> freq;
    for (char c : s) {
        freq[c]++;
    }
    return freq;
}
```

#### Group Anagrams

```cpp
std::vector<std::vector<std::string>> GroupAnagrams(
    std::vector<std::string>& strs) {
    
    std::unordered_map<std::string, std::vector<std::string>> groups;
    
    for (const auto& s : strs) {
        std::string key = s;
        std::sort(key.begin(), key.end());  // Sorted string as key
        groups[key].push_back(s);
    }
    
    std::vector<std::vector<std::string>> result;
    for (auto& [key, group] : groups) {
        result.push_back(std::move(group));
    }
    return result;
}
```

---

## 4. Queue (`std::queue` / `std::deque`)

### Concept

**FIFO** (First In, First Out).

```
Queue Operations:
                    front                    back
                      │                        │
                      ▼                        ▼
                ┌─────┬─────┬─────┬─────┬─────┐
  pop() ◀────── │  1  │  2  │  3  │  4  │  5  │ ◀────── push()
                └─────┴─────┴─────┴─────┴─────┘
```

### C++ STL

```cpp
#include <queue>

std::queue<int> q;

// Operations
q.push(1);       // Add to back
q.front();       // View front element (don't use on empty queue!)
q.back();        // View back element
q.pop();         // Remove front (doesn't return it!)
q.empty();       // Check if empty
q.size();        // Number of elements

// Common pattern: BFS
while (!q.empty()) {
    int curr = q.front();
    q.pop();
    // Process curr, push neighbors
}
```

### Time Complexity

| Operation | Complexity |
|-----------|------------|
| `push` | $O(1)$ |
| `pop` | $O(1)$ |
| `front`/`back` | $O(1)$ |

### When to Use

- **BFS (Breadth-First Search)**: Finding shortest path in unweighted graph/grid
- **Level-order traversal** of trees
- **Processing in order of arrival**

### The Google Standard

#### Use `std::deque` for Monotonic Queue

If you need a **Monotonic Queue** (e.g., Sliding Window Maximum), use `std::deque` directly.

```cpp
#include <deque>

// Sliding Window Maximum - O(n) solution
std::vector<int> MaxSlidingWindow(std::vector<int>& nums, int k) {
    std::deque<int> dq;  // Stores indices, decreasing order of values
    std::vector<int> result;
    
    for (int i = 0; i < nums.size(); ++i) {
        // Remove elements outside window
        while (!dq.empty() && dq.front() <= i - k) {
            dq.pop_front();
        }
        
        // Remove smaller elements (they'll never be the max)
        while (!dq.empty() && nums[dq.back()] < nums[i]) {
            dq.pop_back();
        }
        
        dq.push_back(i);
        
        // Window is full, record result
        if (i >= k - 1) {
            result.push_back(nums[dq.front()]);
        }
    }
    return result;
}
```

### BFS Template

```cpp
#include <queue>
#include <vector>
#include <unordered_set>

int BfsShortestPath(int start, int target, 
                    const std::vector<std::vector<int>>& graph) {
    std::queue<int> q;
    std::unordered_set<int> visited;
    
    q.push(start);
    visited.insert(start);
    int distance = 0;
    
    while (!q.empty()) {
        int level_size = q.size();  // Process level by level
        
        for (int i = 0; i < level_size; ++i) {
            int curr = q.front();
            q.pop();
            
            if (curr == target) {
                return distance;
            }
            
            for (int neighbor : graph[curr]) {
                if (visited.find(neighbor) == visited.end()) {
                    visited.insert(neighbor);
                    q.push(neighbor);
                }
            }
        }
        ++distance;
    }
    return -1;  // Not found
}
```

---

## 5. Stack (`std::stack`)

### Concept

**LIFO** (Last In, First Out).

```
Stack Operations:
                  top
                   │
                   ▼
              ┌─────┐
  pop() ◀──── │  5  │ ◀──── push()
              ├─────┤
              │  4  │
              ├─────┤
              │  3  │
              ├─────┤
              │  2  │
              ├─────┤
              │  1  │
              └─────┘
```

### C++ STL

```cpp
#include <stack>

std::stack<int> s;

// Operations
s.push(1);       // Add to top
s.top();         // View top element (don't use on empty stack!)
s.pop();         // Remove top (doesn't return it!)
s.empty();       // Check if empty
s.size();        // Number of elements

// Common pattern
while (!s.empty()) {
    int curr = s.top();
    s.pop();
    // Process curr
}
```

### Time Complexity

| Operation | Complexity |
|-----------|------------|
| `push` | $O(1)$ |
| `pop` | $O(1)$ |
| `top` | $O(1)$ |

### When to Use

- **DFS (Iterative)**: Converting recursion to iteration
- **Parsing**: Valid Parentheses, evaluating expressions
- **Monotonic Stack**: "Next greater element", "Largest rectangle in histogram"
- **Backtracking**: Undo operations

### The Google Standard

#### Using `vector` as Stack

Often you can use `std::vector` directly as a stack. Slightly faster due to lower overhead.

```cpp
// Using std::stack
std::stack<int> s;
s.push(1);
s.top();
s.pop();

// Using std::vector (often preferred)
std::vector<int> stack;
stack.push_back(1);  // push
stack.back();        // top
stack.pop_back();    // pop
```

### Essential Patterns

#### Valid Parentheses

```cpp
bool IsValid(const std::string& s) {
    std::stack<char> stack;
    std::unordered_map<char, char> pairs = {
        {')', '('}, {']', '['}, {'}', '{'}
    };
    
    for (char c : s) {
        if (pairs.count(c)) {  // Closing bracket
            if (stack.empty() || stack.top() != pairs[c]) {
                return false;
            }
            stack.pop();
        } else {  // Opening bracket
            stack.push(c);
        }
    }
    return stack.empty();
}
```

#### Monotonic Stack: Next Greater Element

```cpp
// For each element, find the next greater element to its right
std::vector<int> NextGreaterElement(const std::vector<int>& nums) {
    int n = nums.size();
    std::vector<int> result(n, -1);
    std::stack<int> stack;  // Stores indices
    
    for (int i = 0; i < n; ++i) {
        // Pop all elements smaller than current
        while (!stack.empty() && nums[stack.top()] < nums[i]) {
            result[stack.top()] = nums[i];
            stack.pop();
        }
        stack.push(i);
    }
    return result;
}
```

#### Largest Rectangle in Histogram

```cpp
int LargestRectangleArea(std::vector<int>& heights) {
    std::stack<int> stack;  // Indices of increasing heights
    int max_area = 0;
    heights.push_back(0);  // Sentinel to flush stack
    
    for (int i = 0; i < heights.size(); ++i) {
        while (!stack.empty() && heights[stack.top()] > heights[i]) {
            int h = heights[stack.top()];
            stack.pop();
            int width = stack.empty() ? i : (i - stack.top() - 1);
            max_area = std::max(max_area, h * width);
        }
        stack.push(i);
    }
    
    heights.pop_back();  // Restore original
    return max_area;
}
```

#### Iterative DFS (Tree Traversal)

```cpp
// Preorder: Node → Left → Right
std::vector<int> PreorderIterative(TreeNode* root) {
    std::vector<int> result;
    if (root == nullptr) return result;
    
    std::stack<TreeNode*> stack;
    stack.push(root);
    
    while (!stack.empty()) {
        TreeNode* node = stack.top();
        stack.pop();
        result.push_back(node->val);
        
        // Push right first so left is processed first
        if (node->right) stack.push(node->right);
        if (node->left) stack.push(node->left);
    }
    return result;
}
```

---

## Comparison Cheat Sheet

| Data Structure | C++ STL | Access | Search | Insert/Delete | Ideal For |
|:---------------|:--------|:-------|:-------|:--------------|:----------|
| **Array** | `vector` | $O(1)$ | $O(n)$ | $O(n)$* | Random access, DP tables |
| **Linked List** | Custom / `list` | $O(n)$ | $O(n)$ | $O(1)$ | Splicing, LRU cache |
| **Queue** | `queue` | N/A | N/A | $O(1)$ | BFS, Level-order |
| **Stack** | `stack` | N/A | N/A | $O(1)$ | DFS, Parsing, Monotonic |
| **Hash Table** | `unordered_map` | N/A | $O(1)$ | $O(1)$ | Lookups, Frequency |
| **BST** | `map` / `set` | N/A | $O(\log n)$ | $O(\log n)$ | Sorted data, Ranges |

*\* $O(1)$ if inserting at the end of a vector.*

---

## Quick Decision Guide

```
Need random access by index?
    └── Yes → std::vector

Need O(1) lookup by key?
    └── Yes → std::unordered_map / unordered_set

Need sorted keys or range queries?
    └── Yes → std::map / set

Need FIFO processing?
    └── Yes → std::queue (BFS)

Need LIFO processing?
    └── Yes → std::stack (DFS, parsing)

Need frequent insert/delete in middle?
    └── Yes → std::list (but consider if vector is still faster)
```

---

## Interview Tips

1. **Always state complexity** when choosing a data structure
   - "I'll use unordered_map for O(1) lookups"

2. **Mention trade-offs**
   - "Vector has better cache locality, but list has O(1) insertion"

3. **Know the defaults**
   - `vector` is almost always your first choice
   - `unordered_map` over `map` unless you need ordering

4. **Watch for edge cases**
   - Empty containers
   - Single element
   - Duplicate keys

5. **Memory considerations**
   - `reserve()` for vectors
   - Dummy head for linked lists
   - Ask about memory cleanup
