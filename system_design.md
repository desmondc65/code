# Phase 3: System Design & Concurrency

> **Goal**: Handle L4+ scale and complexity questions.

---

## 1. Concurrency in C++

You may be asked to make your code **thread-safe**. Understanding C++ concurrency primitives is essential.

### Core Concepts: Race Conditions and Data Races

```cpp
// ❌ DANGEROUS: Data race - undefined behavior!
int counter = 0;

void IncrementCounter() {
    for (int i = 0; i < 1000000; ++i) {
        counter++;  // NOT atomic: read-modify-write
    }
}

int main() {
    std::thread t1(IncrementCounter);
    std::thread t2(IncrementCounter);
    t1.join();
    t2.join();
    // counter is NOT 2000000! Could be anything.
}
```

**What happens:**
```
Thread 1: READ counter (0)
Thread 2: READ counter (0)
Thread 1: INCREMENT (0 → 1)
Thread 2: INCREMENT (0 → 1)  // Lost update!
Thread 1: WRITE counter (1)
Thread 2: WRITE counter (1)
```

---

### `std::mutex` — Mutual Exclusion

A mutex ensures only one thread can access a critical section at a time.

```cpp
#include <mutex>
#include <thread>

class Counter {
 public:
    void Increment() {
        mutex_.lock();      // Acquire lock
        ++value_;           // Critical section
        mutex_.unlock();    // Release lock
    }
    
    int GetValue() const {
        mutex_.lock();
        int v = value_;
        mutex_.unlock();
        return v;
    }

 private:
    mutable std::mutex mutex_;  // mutable: can lock in const methods
    int value_ = 0;
};
```

**Problem**: If an exception is thrown between `lock()` and `unlock()`, the mutex is never released → **Deadlock!**

---

### `std::lock_guard` — RAII Locking (Preferred)

`lock_guard` automatically releases the mutex when it goes out of scope (RAII pattern).

```cpp
#include <mutex>

class Counter {
 public:
    void Increment() {
        std::lock_guard<std::mutex> lock(mutex_);  // Locks mutex
        ++value_;
        // lock_guard destructor automatically unlocks when scope exits
        // Even if exception is thrown!
    }
    
    int GetValue() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return value_;
    }

 private:
    mutable std::mutex mutex_;
    int value_ = 0;
};
```

**C++17 CTAD (Class Template Argument Deduction):**
```cpp
std::lock_guard lock(mutex_);  // No need for <std::mutex>
```

---

### `std::unique_lock` — Flexible Locking

`unique_lock` offers more control than `lock_guard`:
- Can be locked/unlocked multiple times
- Can defer locking
- Can transfer ownership (movable)
- Required for condition variables

```cpp
#include <mutex>

class FlexibleCounter {
 public:
    void ConditionalIncrement(bool should_increment) {
        std::unique_lock<std::mutex> lock(mutex_);
        
        if (!should_increment) {
            lock.unlock();  // Can manually unlock
            DoSomethingWithoutLock();
            lock.lock();    // Can re-lock
        }
        
        ++value_;
    }
    
    // Deferred locking
    void DeferredOperation() {
        std::unique_lock<std::mutex> lock(mutex_, std::defer_lock);
        // Mutex NOT locked yet
        
        DoPreparation();
        
        lock.lock();  // Now lock
        ++value_;
    }
    
    // Try to lock (non-blocking)
    bool TryIncrement() {
        std::unique_lock<std::mutex> lock(mutex_, std::try_to_lock);
        if (lock.owns_lock()) {
            ++value_;
            return true;
        }
        return false;  // Couldn't acquire lock
    }

 private:
    std::mutex mutex_;
    int value_ = 0;
    
    void DoSomethingWithoutLock() { /* ... */ }
    void DoPreparation() { /* ... */ }
};
```

---

### `std::atomic<T>` — Lock-Free Operations

For simple operations (counters, flags), atomics are faster than mutexes.

```cpp
#include <atomic>
#include <thread>

class AtomicCounter {
 public:
    void Increment() {
        ++value_;  // Atomic increment, no lock needed!
    }
    
    void Add(int n) {
        value_ += n;  // Also atomic
    }
    
    int GetValue() const {
        return value_.load();  // Atomic read
    }
    
    // Compare-and-swap (CAS) - fundamental lock-free operation
    bool CompareAndSet(int expected, int desired) {
        return value_.compare_exchange_strong(expected, desired);
        // If value_ == expected: set to desired, return true
        // If value_ != expected: set expected to current value, return false
    }

 private:
    std::atomic<int> value_{0};
};

// Usage
AtomicCounter counter;

void Worker() {
    for (int i = 0; i < 1000000; ++i) {
        counter.Increment();
    }
}

int main() {
    std::thread t1(Worker);
    std::thread t2(Worker);
    t1.join();
    t2.join();
    // counter.GetValue() == 2000000 (guaranteed!)
}
```

**Common Atomic Types:**
```cpp
std::atomic<bool> flag{false};        // Atomic flag
std::atomic<int> counter{0};          // Atomic counter
std::atomic<int*> ptr{nullptr};       // Atomic pointer

// Atomic operations
flag.store(true);                     // Write
bool b = flag.load();                 // Read
int old = counter.fetch_add(1);       // Add and return old value
int old2 = counter.exchange(100);     // Swap and return old value
```

**When to use atomics vs mutexes:**
| Use Atomics | Use Mutexes |
|-------------|-------------|
| Single variable operations | Multiple variable operations |
| Simple counters/flags | Complex invariants |
| Performance-critical | Need to protect larger critical sections |

---

### `std::condition_variable` — Thread Signaling

Condition variables allow threads to wait for a condition to become true. Essential for producer-consumer patterns.

```cpp
#include <condition_variable>
#include <mutex>
#include <queue>
#include <thread>

template <typename T>
class ThreadSafeQueue {
 public:
    void Push(T value) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            queue_.push(std::move(value));
        }
        // Notify AFTER releasing lock for better performance
        cv_.notify_one();  // Wake up one waiting thread
    }
    
    T Pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        
        // Wait until queue is not empty
        // Spurious wakeups can occur, so use predicate version
        cv_.wait(lock, [this] { return !queue_.empty(); });
        
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }
    
    // Non-blocking try_pop
    bool TryPop(T& value) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (queue_.empty()) {
            return false;
        }
        value = std::move(queue_.front());
        queue_.pop();
        return true;
    }
    
    // Pop with timeout
    template <typename Duration>
    bool PopWithTimeout(T& value, Duration timeout) {
        std::unique_lock<std::mutex> lock(mutex_);
        
        if (!cv_.wait_for(lock, timeout, [this] { return !queue_.empty(); })) {
            return false;  // Timeout
        }
        
        value = std::move(queue_.front());
        queue_.pop();
        return true;
    }

 private:
    std::queue<T> queue_;
    std::mutex mutex_;
    std::condition_variable cv_;
};
```

#### Producer-Consumer Example

```cpp
#include <iostream>
#include <thread>
#include <vector>

ThreadSafeQueue<int> task_queue;
std::atomic<bool> done{false};

void Producer(int id) {
    for (int i = 0; i < 10; ++i) {
        task_queue.Push(id * 100 + i);
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}

void Consumer(int id) {
    while (!done || /* check if queue has items */) {
        int task;
        if (task_queue.TryPop(task)) {
            std::cout << "Consumer " << id << " processed task " << task << "\n";
        } else if (done) {
            break;
        } else {
            std::this_thread::yield();  // Let other threads run
        }
    }
}

int main() {
    std::vector<std::thread> producers;
    std::vector<std::thread> consumers;
    
    // Start consumers
    for (int i = 0; i < 2; ++i) {
        consumers.emplace_back(Consumer, i);
    }
    
    // Start producers
    for (int i = 0; i < 3; ++i) {
        producers.emplace_back(Producer, i);
    }
    
    // Wait for producers
    for (auto& t : producers) t.join();
    
    done = true;  // Signal consumers to stop
    
    // Wait for consumers
    for (auto& t : consumers) t.join();
}
```

---

### Deadlock: What It Is and How to Prevent It

**Deadlock** occurs when two or more threads are waiting for each other to release resources, creating a circular wait.

```cpp
// ❌ DEADLOCK EXAMPLE
std::mutex mutex_a, mutex_b;

void Thread1() {
    std::lock_guard<std::mutex> lock_a(mutex_a);  // Holds A
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard<std::mutex> lock_b(mutex_b);  // Waits for B
    // ...
}

void Thread2() {
    std::lock_guard<std::mutex> lock_b(mutex_b);  // Holds B
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard<std::mutex> lock_a(mutex_a);  // Waits for A
    // ...
}

// Thread 1: Has A, wants B
// Thread 2: Has B, wants A
// → DEADLOCK!
```

#### Prevention Strategies

**1. Always acquire locks in the same order:**
```cpp
// ✅ GOOD: Both threads lock A first, then B
void Thread1() {
    std::lock_guard<std::mutex> lock_a(mutex_a);
    std::lock_guard<std::mutex> lock_b(mutex_b);
    // ...
}

void Thread2() {
    std::lock_guard<std::mutex> lock_a(mutex_a);  // Same order!
    std::lock_guard<std::mutex> lock_b(mutex_b);
    // ...
}
```

**2. Use `std::lock()` to lock multiple mutexes atomically:**
```cpp
// ✅ GOOD: Lock both mutexes atomically (deadlock-free)
void SafeOperation() {
    std::unique_lock<std::mutex> lock_a(mutex_a, std::defer_lock);
    std::unique_lock<std::mutex> lock_b(mutex_b, std::defer_lock);
    
    std::lock(lock_a, lock_b);  // Locks both without deadlock
    // ...
}
```

**3. Use `std::scoped_lock` (C++17) — Best approach:**
```cpp
// ✅ BEST: scoped_lock handles multiple mutexes automatically
void SafeOperation() {
    std::scoped_lock lock(mutex_a, mutex_b);  // Locks both, deadlock-free
    // Both are unlocked when scope exits
}
```

**4. Use try_lock with backoff:**
```cpp
void TryLockWithBackoff() {
    while (true) {
        if (mutex_a.try_lock()) {
            if (mutex_b.try_lock()) {
                // Got both locks!
                // ... do work ...
                mutex_b.unlock();
                mutex_a.unlock();
                return;
            }
            mutex_a.unlock();  // Release A if couldn't get B
        }
        std::this_thread::yield();  // Backoff
    }
}
```

#### Four Conditions for Deadlock (Coffman Conditions)

All four must be true for deadlock to occur:

1. **Mutual Exclusion**: Resources are non-shareable
2. **Hold and Wait**: Thread holds resource while waiting for another
3. **No Preemption**: Resources can't be forcibly taken
4. **Circular Wait**: Circular chain of threads waiting

**Prevention**: Break any one condition. Typically, we break "Circular Wait" by enforcing a lock ordering.

---

### Thread-Safe Singleton (Double-Checked Locking)

```cpp
#include <mutex>

class Singleton {
 public:
    static Singleton& GetInstance() {
        // Double-checked locking pattern
        if (instance_ == nullptr) {
            std::lock_guard<std::mutex> lock(mutex_);
            if (instance_ == nullptr) {
                instance_ = new Singleton();
            }
        }
        return *instance_;
    }
    
    // Delete copy/move
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;

 private:
    Singleton() = default;
    
    static Singleton* instance_;
    static std::mutex mutex_;
};

// Better: Use static local variable (Meyers' Singleton)
// Thread-safe in C++11 and later
class BetterSingleton {
 public:
    static BetterSingleton& GetInstance() {
        static BetterSingleton instance;  // Thread-safe initialization
        return instance;
    }
    
    BetterSingleton(const BetterSingleton&) = delete;
    BetterSingleton& operator=(const BetterSingleton&) = delete;

 private:
    BetterSingleton() = default;
};
```

---

### Read-Write Lock (`std::shared_mutex`)

When reads are much more frequent than writes:

```cpp
#include <shared_mutex>
#include <map>

class ThreadSafeCache {
 public:
    std::optional<int> Get(const std::string& key) const {
        std::shared_lock<std::shared_mutex> lock(mutex_);  // Multiple readers OK
        auto it = cache_.find(key);
        if (it != cache_.end()) {
            return it->second;
        }
        return std::nullopt;
    }
    
    void Put(const std::string& key, int value) {
        std::unique_lock<std::shared_mutex> lock(mutex_);  // Exclusive write
        cache_[key] = value;
    }

 private:
    mutable std::shared_mutex mutex_;
    std::map<std::string, int> cache_;
};
```

---

## 2. Large Scale Concepts

### Memory Constraints: External Merge Sort

**Problem**: Sort 1TB of data with only 16GB RAM.

**Solution**: External Merge Sort

```
Step 1: DIVIDE
┌─────────────────────────────────────────────────────────┐
│                    1TB unsorted file                     │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐       ┌─────────┐
    │ Chunk 1 │  │ Chunk 2 │  │ Chunk 3 │  ...  │Chunk 64 │
    │  16GB   │  │  16GB   │  │  16GB   │       │  16GB   │
    └─────────┘  └─────────┘  └─────────┘       └─────────┘
         │            │            │                 │
         ▼            ▼            ▼                 ▼
    Sort in RAM  Sort in RAM  Sort in RAM      Sort in RAM
         │            │            │                 │
         ▼            ▼            ▼                 ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐       ┌─────────┐
    │ Sorted  │  │ Sorted  │  │ Sorted  │  ...  │ Sorted  │
    │ Chunk 1 │  │ Chunk 2 │  │ Chunk 3 │       │Chunk 64 │
    └─────────┘  └─────────┘  └─────────┘       └─────────┘
    
Step 2: K-WAY MERGE
    Each chunk contributes a small buffer (e.g., 256MB)
    Use min-heap to merge all chunks simultaneously
    
    ┌───────────────────────────────────────┐
    │           Min-Heap (64 elements)       │
    │  One element from each sorted chunk    │
    └───────────────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────────────┐
    │                  1TB sorted output file                  │
    └─────────────────────────────────────────────────────────┘
```

#### Implementation Sketch

```cpp
#include <queue>
#include <fstream>
#include <vector>
#include <algorithm>

struct ChunkElement {
    int value;
    int chunk_id;
    
    bool operator>(const ChunkElement& other) const {
        return value > other.value;  // Min-heap
    }
};

void ExternalMergeSort(const std::string& input_file,
                       const std::string& output_file,
                       size_t memory_limit) {
    // Step 1: Create sorted chunks
    std::vector<std::string> chunk_files;
    {
        std::ifstream input(input_file, std::ios::binary);
        std::vector<int> buffer;
        buffer.reserve(memory_limit / sizeof(int));
        
        int value;
        while (input.read(reinterpret_cast<char*>(&value), sizeof(int))) {
            buffer.push_back(value);
            
            if (buffer.size() * sizeof(int) >= memory_limit) {
                // Sort chunk in memory
                std::sort(buffer.begin(), buffer.end());
                
                // Write to temporary file
                std::string chunk_file = "chunk_" + 
                    std::to_string(chunk_files.size()) + ".tmp";
                std::ofstream out(chunk_file, std::ios::binary);
                for (int v : buffer) {
                    out.write(reinterpret_cast<char*>(&v), sizeof(int));
                }
                chunk_files.push_back(chunk_file);
                buffer.clear();
            }
        }
        
        // Don't forget the last partial chunk
        if (!buffer.empty()) {
            std::sort(buffer.begin(), buffer.end());
            std::string chunk_file = "chunk_" + 
                std::to_string(chunk_files.size()) + ".tmp";
            std::ofstream out(chunk_file, std::ios::binary);
            for (int v : buffer) {
                out.write(reinterpret_cast<char*>(&v), sizeof(int));
            }
            chunk_files.push_back(chunk_file);
        }
    }
    
    // Step 2: K-way merge using min-heap
    std::vector<std::ifstream> chunk_streams;
    std::priority_queue<ChunkElement, 
                        std::vector<ChunkElement>,
                        std::greater<ChunkElement>> min_heap;
    
    // Open all chunk files and seed the heap
    for (size_t i = 0; i < chunk_files.size(); ++i) {
        chunk_streams.emplace_back(chunk_files[i], std::ios::binary);
        int value;
        if (chunk_streams[i].read(reinterpret_cast<char*>(&value), sizeof(int))) {
            min_heap.push({value, static_cast<int>(i)});
        }
    }
    
    // Merge
    std::ofstream output(output_file, std::ios::binary);
    while (!min_heap.empty()) {
        ChunkElement smallest = min_heap.top();
        min_heap.pop();
        
        output.write(reinterpret_cast<const char*>(&smallest.value), sizeof(int));
        
        // Read next element from the same chunk
        int next_value;
        if (chunk_streams[smallest.chunk_id].read(
                reinterpret_cast<char*>(&next_value), sizeof(int))) {
            min_heap.push({next_value, smallest.chunk_id});
        }
    }
    
    // Cleanup: Delete temporary files
    for (const auto& file : chunk_files) {
        std::remove(file.c_str());
    }
}
```

#### Complexity Analysis

| Metric | Value |
|--------|-------|
| **Disk I/O** | O(n) reads + O(n) writes per pass |
| **Passes** | 2 (create chunks + merge) for reasonable k |
| **Memory** | O(memory_limit) |
| **Time** | O(n log n) for sorting + O(n log k) for k-way merge |

**Key Optimizations:**
1. **Buffered I/O**: Read/write in large blocks (e.g., 4MB) to minimize disk seeks
2. **Replacement Selection**: Can create chunks ~2x memory size on average
3. **Multi-threaded sorting**: Sort chunks in parallel
4. **SSD vs HDD**: SSDs handle random access better; HDDs benefit from sequential access

---

### MapReduce: Distributed Data Processing

**What is MapReduce?**

A programming model for processing large datasets in parallel across a cluster.

```
                    INPUT DATA (Distributed)
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
       ┌─────────┐    ┌─────────┐    ┌─────────┐
       │  Map 1  │    │  Map 2  │    │  Map 3  │
       └────┬────┘    └────┬────┘    └────┬────┘
            │              │              │
            ▼              ▼              ▼
     (key, value)    (key, value)    (key, value)
            │              │              │
            └──────────────┼──────────────┘
                           │
                      SHUFFLE & SORT
                    (Group by key)
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
       ┌─────────┐    ┌─────────┐    ┌─────────┐
       │Reduce 1 │    │Reduce 2 │    │Reduce 3 │
       └────┬────┘    └────┬────┘    └────┬────┘
            │              │              │
            ▼              ▼              ▼
                      OUTPUT DATA
```

#### Classic Example: Word Count

```cpp
// MAP FUNCTION
// Input: (document_id, document_text)
// Output: list of (word, 1) pairs
std::vector<std::pair<std::string, int>> Map(
    const std::string& doc_id,
    const std::string& text) {
    
    std::vector<std::pair<std::string, int>> output;
    std::istringstream stream(text);
    std::string word;
    
    while (stream >> word) {
        // Emit (word, 1) for each word
        output.emplace_back(word, 1);
    }
    return output;
}

// REDUCE FUNCTION
// Input: (word, list of counts)
// Output: (word, total_count)
std::pair<std::string, int> Reduce(
    const std::string& word,
    const std::vector<int>& counts) {
    
    int total = 0;
    for (int count : counts) {
        total += count;
    }
    return {word, total};
}
```

**Execution Flow for Word Count:**
```
Documents:
  Doc1: "hello world"
  Doc2: "hello mapreduce"
  Doc3: "world of mapreduce"

MAP Phase (parallel):
  Mapper1(Doc1) → [("hello", 1), ("world", 1)]
  Mapper2(Doc2) → [("hello", 1), ("mapreduce", 1)]
  Mapper3(Doc3) → [("world", 1), ("of", 1), ("mapreduce", 1)]

SHUFFLE & SORT (framework handles this):
  "hello"     → [1, 1]
  "mapreduce" → [1, 1]
  "of"        → [1]
  "world"     → [1, 1]

REDUCE Phase (parallel):
  Reducer("hello", [1, 1])     → ("hello", 2)
  Reducer("mapreduce", [1, 1]) → ("mapreduce", 2)
  Reducer("of", [1])           → ("of", 1)
  Reducer("world", [1, 1])     → ("world", 2)
```

#### More MapReduce Examples

**1. Inverted Index (Search Engine)**
```cpp
// Build an index: word → list of documents containing it

// MAP: (doc_id, text) → [(word, doc_id), ...]
std::vector<std::pair<std::string, std::string>> MapInvertedIndex(
    const std::string& doc_id,
    const std::string& text) {
    
    std::set<std::string> unique_words;
    std::istringstream stream(text);
    std::string word;
    while (stream >> word) {
        unique_words.insert(word);
    }
    
    std::vector<std::pair<std::string, std::string>> output;
    for (const auto& w : unique_words) {
        output.emplace_back(w, doc_id);
    }
    return output;
}

// REDUCE: (word, [doc_ids]) → (word, sorted_doc_id_list)
std::pair<std::string, std::vector<std::string>> ReduceInvertedIndex(
    const std::string& word,
    const std::vector<std::string>& doc_ids) {
    
    std::vector<std::string> sorted_docs = doc_ids;
    std::sort(sorted_docs.begin(), sorted_docs.end());
    return {word, sorted_docs};
}
```

**2. Distributed Sort**
```cpp
// Sort 1TB of data across 100 machines

// MAP: Partition data by key range
// Each mapper outputs (partition_id, record)
// where partition_id = hash(key) % num_reducers

// REDUCE: Each reducer receives one partition
// Locally sorts its partition
// Output is globally sorted (partitions are ordered)
```

**3. PageRank (Simplified)**
```cpp
// MAP: For each page, distribute its rank to outgoing links
std::vector<std::pair<std::string, double>> MapPageRank(
    const std::string& page,
    double rank,
    const std::vector<std::string>& outlinks) {
    
    std::vector<std::pair<std::string, double>> output;
    double contribution = rank / outlinks.size();
    
    for (const auto& link : outlinks) {
        output.emplace_back(link, contribution);
    }
    return output;
}

// REDUCE: Sum contributions, apply damping factor
std::pair<std::string, double> ReducePageRank(
    const std::string& page,
    const std::vector<double>& contributions) {
    
    const double damping = 0.85;
    double sum = 0;
    for (double c : contributions) {
        sum += c;
    }
    double new_rank = (1 - damping) + damping * sum;
    return {page, new_rank};
}
// Run multiple iterations until convergence
```

#### MapReduce Design Patterns

| Pattern | Description | Example |
|---------|-------------|---------|
| **Summarization** | Aggregate values by key | Word count, avg/min/max |
| **Filtering** | Select subset of records | Log analysis, data cleaning |
| **Data Organization** | Restructure data | Inverted index, sort |
| **Join** | Combine multiple datasets | SQL-like joins |
| **Graph** | Iterative graph algorithms | PageRank, shortest path |

#### When to Mention MapReduce in Interviews

- **"How would you process 1PB of log files?"** → MapReduce
- **"Design a search engine index builder"** → MapReduce for inverted index
- **"Count unique users across 1B events"** → MapReduce with HyperLogLog
- **"Find top K most frequent items in a huge dataset"** → MapReduce with heap

---

### Other Large-Scale Concepts

#### Consistent Hashing

Used for distributed caching (e.g., Memcached, Redis cluster).

```
Problem: Simple hash (key % num_servers) fails when servers added/removed
         → All keys get remapped!

Solution: Consistent hashing
         → Only K/N keys remapped on average

┌─────────────────────────────────────────┐
│           Hash Ring (0 to 2^32-1)        │
│                                         │
│               Server A                   │
│                  ●                       │
│            ╱           ╲                │
│          ╱               ╲              │
│        ╱                   ╲            │
│   Server D ●               ● Server B    │
│        ╲                   ╱            │
│          ╲               ╱              │
│            ╲           ╱                │
│                  ●                       │
│               Server C                   │
│                                         │
│   Keys hash to ring, assigned to next   │
│   server clockwise                       │
└─────────────────────────────────────────┘
```

#### Bloom Filters

Space-efficient probabilistic data structure for set membership.

```cpp
// Properties:
// - Can say "definitely not in set" (no false negatives)
// - Can say "probably in set" (possible false positives)
// - Very space efficient: ~10 bits per element for 1% FP rate

// Use cases:
// - Pre-filter before expensive lookup
// - Spell checkers
// - Network packet filtering
// - Avoiding disk reads for non-existent keys
```

#### Reservoir Sampling

Select k random items from a stream of unknown size.

```cpp
#include <random>
#include <vector>

template <typename T>
std::vector<T> ReservoirSample(std::istream& stream, size_t k) {
    std::vector<T> reservoir(k);
    std::random_device rd;
    std::mt19937 gen(rd());
    
    T item;
    size_t count = 0;
    
    while (stream >> item) {
        if (count < k) {
            reservoir[count] = item;
        } else {
            // Replace with probability k/(count+1)
            std::uniform_int_distribution<size_t> dist(0, count);
            size_t j = dist(gen);
            if (j < k) {
                reservoir[j] = item;
            }
        }
        ++count;
    }
    
    return reservoir;
}
```

---

## Quick Reference: Concurrency Primitives

| Primitive | Use Case | Notes |
|-----------|----------|-------|
| `std::mutex` | Basic mutual exclusion | Lock/unlock manually |
| `std::lock_guard` | RAII mutex wrapper | Simple, non-transferable |
| `std::unique_lock` | Flexible RAII wrapper | Deferable, transferable |
| `std::scoped_lock` | Multiple mutex locking | Deadlock-free (C++17) |
| `std::shared_mutex` | Read-write lock | Multiple readers OR one writer |
| `std::atomic<T>` | Lock-free operations | Simple types only |
| `std::condition_variable` | Thread signaling | Producer-consumer |

---

## Interview Tips

1. **Always mention thread safety** when designing classes that might be used concurrently

2. **Prefer `scoped_lock` or `lock_guard`** over manual lock/unlock

3. **Use atomics for counters**, mutexes for complex operations

4. **Know the deadlock conditions** and prevention strategies

5. **For large-scale problems**, always ask about:
   - Data size vs memory constraints
   - Single machine vs distributed
   - Batch vs streaming
   - Consistency requirements

6. **MapReduce mental model**:
   - Map: Transform each record independently
   - Shuffle: Group by key (framework handles)
   - Reduce: Aggregate grouped records
