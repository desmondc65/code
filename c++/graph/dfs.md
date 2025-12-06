Here is the comprehensive guide to Depth-First Search (DFS), structured for high-performance C++ implementation and interview preparation.

### What is DFS?

Depth-First Search is a graph traversal algorithm that explores as far as possible along each branch before backtracking. It dives deep into the graph structure immediately.

**The Analogy:** Think of solving a maze. You pick a path and keep walking until you hit a dead end. Once you hit a wall, you retrace your steps (backtrack) to the last intersection and try a different path.

[Image of DFS traversal]

### Core Mechanism: The Stack

While BFS uses a **Queue** (FIFO), DFS relies on a **Stack** (LIFO).

  * **Implicit Stack:** Uses the system call stack via **Recursion**. This is the standard implementation.
  * **Explicit Stack:** Uses `std::stack` for iterative implementation (prevents Stack Overflow on extremely deep graphs).

### Complexity Analysis

  * **Time Complexity:** $O(V + E)$
      * We visit every vertex $V$ and inspect every edge $E$ exactly once (twice in undirected graphs, once from each side).
  * **Space Complexity:** $O(V)$
      * In the worst case (a long line or skewed tree), the recursion stack grows to the size of $V$.
      * In a balanced tree, space is $O(\log V)$ (the height of the tree).

-----

### Critical Implementation Details

1.  **Recursion Limits:** Python has a low recursion limit (1000). C++ usually handles deep recursion well (approx $10^5$ frames depending on OS/Compiler stack size), but for massive graphs, an **Iterative DFS** is safer.
2.  **Backtracking vs. DFS:** Pure DFS marks a node as visited and **never** visits it again. **Backtracking** (e.g., generating all permutations) unmarks the node (resets state) after returning from the recursion.
3.  **Disconnected Components:** A single DFS call only visits nodes connected to the start node. You usually need a wrapper loop to iterate through all nodes to ensure the whole graph is covered.

-----

### Template 1: Recursive DFS (The Standard)

This is the most common form used in interviews due to code simplicity.

```cpp
#include <iostream>
#include <vector>

using namespace std;

// Pass adjacency list by const reference to avoid copying
void dfsRecursive(int u, const vector<vector<int>>& adj, vector<bool>& visited) {
    // 1. Mark as visited immediately
    visited[u] = true;
    cout << "Visiting: " << u << endl;

    // 2. Iterate neighbors
    for (int v : adj[u]) {
        if (!visited[v]) {
            dfsRecursive(v, adj, visited);
        }
    }
}

// Wrapper to handle disconnected components
void solve(int numNodes, const vector<vector<int>>& adj) {
    vector<bool> visited(numNodes, false);
    
    for(int i = 0; i < numNodes; ++i) {
        if(!visited[i]) {
            dfsRecursive(i, adj, visited);
        }
    }
}
```

### Template 2: Iterative DFS (Explicit Stack)

Use this if you fear Stack Overflow or need to strictly control the traversal order without function overhead.

**Note on Order:** To mimic recursive order exactly (left-to-right), you must push neighbors onto the stack in **reverse order**.

```cpp
#include <vector>
#include <stack>
#include <iostream>

using namespace std;

void dfsIterative(int startNode, int numNodes, const vector<vector<int>>& adj) {
    vector<bool> visited(numNodes, false);
    stack<int> s;

    s.push(startNode);
    // Note: We don't mark visited here for Iterative DFS usually, 
    // we mark it when we POP to handle multiple paths adding same node to stack.
    // However, a stricter optimization is to mark on PUSH if you don't need to re-evaluate.
    
    while (!s.empty()) {
        int u = s.top();
        s.pop();

        if (!visited[u]) {
            visited[u] = true;
            cout << "Visiting: " << u << endl;

            // Push neighbors
            // Reverse iterator used so the first neighbor is processed first (LIFO)
            for (auto it = adj[u].rbegin(); it != adj[u].rend(); ++it) {
                if (!visited[*it]) {
                    s.push(*it);
                }
            }
        }
    }
}
```

### Template 3: Grid DFS (Islands / Flood Fill)

Standard for "Number of Islands" or "Max Area of Island".

```cpp
#include <vector>
using namespace std;

// Direction arrays (Up, Down, Left, Right)
const int dr[] = {-1, 1, 0, 0};
const int dc[] = {0, 0, -1, 1};

void dfsGrid(int r, int c, vector<vector<char>>& grid) {
    int rows = grid.size();
    int cols = grid[0].size();

    // 1. Mark visited (Mutation strategy: change '1' to '0' to save space)
    // If you cannot mutate input, pass a vector<vector<bool>>& visited
    grid[r][c] = '0'; 

    // 2. Explore 4 directions
    for (int i = 0; i < 4; ++i) {
        int nr = r + dr[i];
        int nc = c + dc[i];

        // Bounds check AND "is land" check
        if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] == '1') {
            dfsGrid(nr, nc, grid);
        }
    }
}

int numIslands(vector<vector<char>>& grid) {
    if (grid.empty()) return 0;
    int count = 0;
    int rows = grid.size();
    int cols = grid[0].size();

    for (int i = 0; i < rows; ++i) {
        for (int j = 0; j < cols; ++j) {
            if (grid[i][j] == '1') { // Found an unvisited land mass
                count++;
                dfsGrid(i, j, grid); // Sink the whole island
            }
        }
    }
    return count;
}
```

-----

### Junior vs. Senior Implementation

| Feature | Junior Implementation | Senior Implementation |
| :--- | :--- | :--- |
| **Passing Data** | Passes `adj` or huge vectors by value `(vector<int> adj)`. | Always passes by `const reference` `(const vector<int>& adj)` to avoid $O(V+E)$ copy overhead per call. |
| **Globals** | Declares `visited` and `adj` globally to avoid passing arguments. | Encapsulates logic in a `class` or passes references to keep code re-entrant and thread-safe. |
| **Cycle Detection** | Only uses `bool visited`. Fails to detect cycles in **Directed** graphs. | Uses 3-state coloring: `0` (Unvisited), `1` (Visiting/Recursion Stack), `2` (Visited/Done). |
| **Grid Boundary** | Writes messy `if (r+1 < rows)` logic repeated 4 times. | Uses `dr`/`dc` arrays for clean loops and centralized boundary checking. |

### When to use DFS over BFS?

1.  **Memory is tight:** DFS uses $O(depth)$ space. In a wide tree, this is much less than BFS's $O(width)$.
2.  **Path Existence vs. Shortest Path:** If you just need to know if a path *exists*, or you need to visit *every* node (Connected Components), DFS is often easier to code recursively.
3.  **Topological Sort / Cycle Detection:** DFS is the algorithm of choice for these. You cannot do a standard Topological Sort easily with BFS (though Kahn's algorithm exists, DFS is more intuitive for cycle checks).
4.  **Backtracking Puzzles:** Sudoku, N-Queens, Maze Generation. These are inherently DFS.

### Next Step

Since you are targeting Google, **Topological Sort** and **Cycle Detection in Directed Graphs** are high-frequency DFS applications. Would you like a deep dive into **3-Color DFS (White-Gray-Black)** for cycle detection?