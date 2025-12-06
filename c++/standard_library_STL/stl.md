This detailed guide maps the specific **function names** (API) to the operations and complexities discussed previously.

In C++, precise knowledge of function signatures is critical because:

1.  **Return types matter:** Unlike Python, `pop()` returns `void`, not the element.
2.  **Safety varies:** `[]` is unchecked, `at()` is checked.
3.  **Side effects:** `map[]` creates elements if they don't exist.

-----

### 1\. Sequence Containers (`vector`, `deque`, `list`)

These containers share a standard interface, but availability of specific functions depends on the underlying structure.

#### **Modification & Capacity**

| Function Name | Description | Complexity | Supported By |
| :--- | :--- | :--- | :--- |
| **`push_back(val)`** | Adds element to the end. | $O(1)$ | Vector, Deque, List |
| **`pop_back()`** | Removes last element (returns `void`). | $O(1)$ | Vector, Deque, List |
| **`push_front(val)`** | Adds element to the start. | $O(1)$ | Deque, List (**Not Vector**) |
| **`pop_front()`** | Removes first element. | $O(1)$ | Deque, List (**Not Vector**) |
| **`insert(it, val)`** | Inserts `val` *before* iterator `it`. | $O(N)$ (Vector/Deque)<br>$O(1)$ (List) | All |
| **`erase(it)`** | Removes element at `it`. | $O(N)$ (Vector/Deque)<br>$O(1)$ (List) | All |
| **`emplace_back(...)`**| Constructs element in-place at end. | $O(1)$ | Vector, Deque, List |
| **`clear()`** | Removes all elements (size becomes 0). | $O(N)$ | All |

#### **Element Access**

| Function Name | Description | Safety | Complexity |
| :--- | :--- | :--- | :--- |
| **`operator[i]`** | Returns reference to element at index `i`. | **Unsafe** (Undefined behavior if OOB). | $O(1)$ |
| **`at(i)`** | Returns reference to element at index `i`. | **Safe** (Throws `out_of_range`). | $O(1)$ |
| **`front()`** | Reference to first element. | Undefined on empty. | $O(1)$ |
| **`back()`** | Reference to last element. | Undefined on empty. | $O(1)$ |
| **`data()`** | Returns raw pointer `T*` to memory array. | Vector only. | $O(1)$ |

#### **Vector Specific: Memory Management**

Knowing the difference between these two is a common Google interview check.

1.  **`reserve(n)`**: Changes **capacity**. Allocates memory but creates no objects. Use this before a loop to prevent reallocations.
2.  **`resize(n)`**: Changes **size**. Allocates memory **AND** default constructs `n` objects.

-----

### 2\. Associative Containers (`set`, `map`)

These are sorted. Operations rely on keys.

#### **Lookup Operations**

| Function Name | Description | Complexity (Tree) | Complexity (Hash) |
| :--- | :--- | :--- | :--- |
| **`find(key)`** | Returns iterator to element (or `end()` if not found). | $O(\log N)$ | $O(1)$ |
| **`count(key)`** | Returns 1 if present, 0 if not (for unique containers). | $O(\log N)$ | $O(1)$ |
| **`contains(key)`**| Returns bool (C++20 feature). | $O(\log N)$ | $O(1)$ |

#### **Sorted Specific (`set`/`map` only)**

These functions leverage the Red-Black Tree structure.

  * **`lower_bound(key)`**: Returns iterator to first element **$\ge$** `key`.
  * **`upper_bound(key)`**: Returns iterator to first element **$>$** `key`.
  * **`equal_range(key)`**: Returns pair of iterators `[lower, upper]`.

#### **Map Specific Access**

  * **`operator[key]`**:
      * Returns reference to value mapped to `key`.
      * **Danger:** If `key` does not exist, it **inserts** a default-constructed value. Do not use this just to check existence.
  * **`at(key)`**:
      * Returns reference to value.
      * Throws `std::out_of_range` if key is missing.

-----

### 3\. Container Adapters (`stack`, `queue`, `priority_queue`)

These restrict interfaces. They **do not** support iterators (no `begin`/`end`).

#### **Stack (LIFO)**

  * **`push(val)`** / **`emplace(...)`**: Add to top.
  * **`pop()`**: Remove top (returns void).
  * **`top()`**: Returns reference to top element.

#### **Queue (FIFO)**

  * **`push(val)`**: Add to back.
  * **`pop()`**: Remove from front.
  * **`front()`**: Access first element.
  * **`back()`**: Access last element.

#### **Priority Queue (Heap)**

  * **`push(val)`**: Insert element ($O(\log N)$).
  * **`pop()`**: Remove highest priority element ($O(\log N)$).
  * **`top()`**: Access highest priority element ($O(1)$).
      * *Note:* It uses `top()`, not `front()`.

-----

### 4\. String (`std::string`)

A string is almost identical to `std::vector<char>`, but with added text manipulation functions.

| Function Name | Description | Note |
| :--- | :--- | :--- |
| **`substr(pos, len)`**| Returns a new string substring. | $O(N)$ (Allocates memory). |
| **`find(str)`** | Returns index (size\_t) of first occurrence or `string::npos`. | $O(N \cdot M)$. |
| **`c_str()`** | Returns `const char*` (C-style string). | Essential for legacy APIs. |
| **`append(str)`** | Appends to end (like `+=`). | |
| **`compare(str)`** | Returns 0, \<0, or \>0 (like `strcmp`). | |

-----

### 5\. `std::list` Specific Operations (Splice)

Because `std::list` is a linked list, it has member functions that are optimized to manipulate pointers rather than copy data. Using generic `std::sort` on a list fails; you must use `list::sort`.

  * **`list.sort()`**: Sorts the list ($O(N \log N)$).
  * **`list.merge(other_list)`**: Merges two sorted lists.
  * **`list.splice(pos, other_list)`**: Moves elements from `other_list` to `pos` in the current list.
      * **Complexity:** $O(1)$
      * **Interview Value:** This is the key to implementing LRU Cache in $O(1)$. You move a node from the middle to the front by just changing pointers, no memory allocation involved.

### Next Step

Would you like to proceed with the **LRU Cache implementation** task? I will ask you to write the `get` and `put` functions using `std::list::splice` and `std::unordered_map` to verify you can combine these APIs effectively.