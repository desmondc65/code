This is a comprehensive guide to Breadth-First Search (BFS), tailored for high-performance C++ implementation and algorithmic interviews (like Google).

### What is BFS?

Breadth-First Search is a graph traversal algorithm that explores nodes **layer by layer**. It starts at a source node and explores all its immediate neighbors before moving to the next level of neighbors.

**The Golden Rule:** BFS guarantees the **shortest path** in an unweighted graph. If you need the shortest path in a graph where edges have no weights (or uniform weights), BFS is the optimal choice.

### Core Mechanism: The Queue

Unlike Depth-First Search (DFS) which uses a **Stack** (LIFO) or recursion, BFS uses a **Queue** (FIFO).

1.  **Push** the starting node into the queue and mark it as visited.
2.  **Loop** while the queue is not empty.
3.  **Pop** the front node (current node).
4.  **Explore** all unvisited neighbors of the current node.
5.  **Push** those neighbors into the queue and mark them as visited **immediately**.

### Complexity Analysis

  * **Time Complexity:** $O(V + E)$
      * $V$ is the number of vertices, $E$ is the number of edges. We visit every node once and inspect every edge once.
  * **Space Complexity:** $O(V)$
      * In the worst case (a star graph), the queue might store the majority of the vertices. We also need $O(V)$ for the `visited` array.

-----

### Critical Implementation Details (Interview Standards)

1.  **When to mark visited:** You must mark a node as visited **the moment you push it into the queue**, not when you pop it. If you wait until you pop it, you will add the same node to the queue multiple times, leading to massive inefficiency or infinite loops.
2.  **Level Tracking:** If the problem asks for the "number of steps" or to print the tree level-by-level, you need a nested loop inside the while loop to process the entire size of the queue at that snapshot.

-----

### Template 1: General Graph (Adjacency List)

Use this for standard graph problems (e.g., social networks, connected components).

```cpp
#include <iostream>
#include <vector>
#include <queue>

using namespace std;

// Adjacency list representation
void bfs(int startNode, int numNodes, const vector<vector<int>>& adj) {
    // 1. Setup Data Structures
    vector<bool> visited(numNodes, false);
    queue<int> q;

    // 2. Initialize
    visited[startNode] = true;
    q.push(startNode);

    // 3. Process
    while (!q.empty()) {
        int curr = q.front();
        q.pop();

        cout << "Visiting: " << curr << endl;

        // Iterate over neighbors
        for (int neighbor : adj[curr]) {
            if (!visited[neighbor]) {
                visited[neighbor] = true; // Mark visited IMMEDIATELY
                q.push(neighbor);
            }
        }
    }
}
```

### Template 2: Matrix / Grid (2D Array)

This is highly relevant for Google interviews (e.g., "Rotting Oranges", "Shortest Path in Binary Matrix").

**Key differences:**

  * Nodes are `(r, c)` coordinates.
  * Neighbors are calculated using a `directions` array (Up, Down, Left, Right).
  * Boundary checks are required.

<!-- end list -->

```cpp
#include <vector>
#include <queue>
#include <tuple>

using namespace std;

int shortestPathBinaryMatrix(vector<vector<int>>& grid) {
    int rows = grid.size();
    int cols = grid[0].size();
    
    // Example: 1 is blocked, 0 is open. Find shortest path from (0,0) to (R-1, C-1)
    if (grid[0][0] == 1 || grid[rows-1][cols-1] == 1) return -1;

    // Direction arrays for moving Up, Down, Left, Right
    // For 8 directions, add diagonals to this list
    int dir[4][2] = {{0, 1}, {0, -1}, {1, 0}, {-1, 0}};

    queue<pair<int, int>> q;
    q.push({0, 0});
    grid[0][0] = 1; // Reuse grid as visited array (1 means visited/blocked)
                    // Alternatively, use a separate vector<vector<bool>> visited

    int pathLength = 1; // Start counting

    while (!q.empty()) {
        int levelSize = q.size(); // Snapshot for level-by-level processing

        for (int i = 0; i < levelSize; ++i) {
            auto [r, c] = q.front();
            q.pop();

            // Target check
            if (r == rows - 1 && c == cols - 1) return pathLength;

            for (auto& d : dir) {
                int nr = r + d[0];
                int nc = c + d[1];

                // Boundary Checks and Visited Check
                if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] == 0) {
                    q.push({nr, nc});
                    grid[nr][nc] = 1; // Mark visited immediately
                }
            }
        }
        pathLength++;
    }

    return -1; // Target unreachable
}
```

### Template 3: Multi-Source BFS

Sometimes you have multiple starting points (e.g., "Distance to nearest Police Station" from every house).

**Strategy:** Push **all** starting nodes into the queue at initialization with distance 0. Then run standard BFS.

```cpp
vector<int> multiSourceBFS(int nodes, const vector<vector<int>>& adj, const vector<int>& sources) {
    queue<int> q;
    vector<int> dist(nodes, -1); // -1 indicates unvisited

    // Push ALL sources first
    for (int src : sources) {
        q.push(src);
        dist[src] = 0;
    }

    while (!q.empty()) {
        int curr = q.front();
        q.pop();

        for (int neighbor : adj[curr]) {
            if (dist[neighbor] == -1) { // If not visited
                dist[neighbor] = dist[curr] + 1;
                q.push(neighbor);
            }
        }
    }
    return dist;
}
```

-----

### Junior vs. Senior Mistakes

| Feature | Junior Implementation | Senior Implementation |
| :--- | :--- | :--- |
| **Marking Visited** | Marks node when popping from queue. | Marks node immediately upon pushing to queue. |
| **Space** | Uses `std::map<int, bool>` for visited (Slow, $O(\log N)$ overhead). | Uses `std::vector<bool>` or `std::vector<int>` (Fast, $O(1)$ access). |
| **Grid Boundary** | Writes 4 separate `if` statements for neighbors. | Uses `directions` array `{{0,1}...}` loop. |
| **Graph Type** | Tries to use BFS for weighted graphs. | Knows BFS is only for unweighted; uses Dijkstra for weighted. |

### When NOT to use BFS

1.  **Weighted Graphs:** If edges have different costs (weights), BFS assumes all edges equal 1. It will find the path with the fewest edges, not the lowest weight. **Use Dijkstra.**
2.  **Memory Constraints:** If the graph is massive (infinite) or the branching factor is huge, the queue grows exponentially. **DFS (Iterative Deepening)** might be better for memory.

### Next Step

Since you are prepping for Google, would you like a practice problem that specifically tests **Multi-Source BFS** or **BFS with Bitmasking** (harder variation)?