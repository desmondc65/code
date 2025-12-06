Here is the complete guide and explanation for Dijkstra's Algorithm in Markdown.

-----

# Dijkstra's Algorithm: A Complete Guide (C++)

### 1\. What is Dijkstra’s Algorithm?

Dijkstra's (pronounced *dike-stra*) algorithm is a **Single-Source Shortest Path (SSSP)** algorithm.

  * **Goal:** Find the shortest path from a **starting node** to **all other nodes** in a graph.
  * **Condition:** The graph must have **non-negative edge weights**. (If edges are negative, you must use Bellman-Ford).
  * **Mechanism:** It is a **Greedy Algorithm**. It always processes the closest "unvisited" node next.

Think of it as **Breadth-First Search (BFS)** adapted for weighted graphs. While BFS spreads out layer-by-layer (1 hop, 2 hops), Dijkstra spreads out based on *cost* (cost 1, cost 2, etc.).

-----

### 2\. The Logic (Step-by-Step)

We rely on a principle called **Relaxation**.

  * Imagine we are at node $u$ and we know the shortest distance to $u$ is $d[u]$.
  * If there is an edge from $u \to v$ with weight $w$, we check:
    $$if (d[u] + w < d[v])$$
  * If true, we have found a better (shorter) path to $v$. We update $d[v]$ and add $v$ to our queue to explore later.

#### Data Structures Needed

1.  **`dist[]`**: An array where `dist[i]` stores the shortest distance from the source to node `i`. Initialize all to **Infinity** ($\infty$), except the source, which is **0**.
2.  **`priority_queue` (Min-Heap)**: Stores pairs of `{distance, node}`. This allows us to efficiently retrieve the node with the *smallest* current distance.
3.  **`adj`**: An Adjacency List `vector<vector<pair<int, int>>>` to store the graph.

-----

### 3\. Complexity Analysis

  * **Time Complexity:** $O(E \log V)$
      * $V$ is the number of vertices, $E$ is the number of edges.
      * We iterate through edges, and every update pushes to the priority queue (a $\log V$ operation).
  * **Space Complexity:** $O(V + E)$
      * To store the graph and the distance array.

-----

### 4\. C++ Template (Standard)

This is a production-ready template. It handles `long long` to prevent overflow and uses the standard library Min-Heap.

```cpp
#include <iostream>
#include <vector>
#include <queue>
#include <limits>

using namespace std;

// Define Infinity as a large enough number
const long long INF = numeric_limits<long long>::max();

// Pair shorthand: {weight, target_node}
using pii = pair<long long, int>; 

// Dijkstra Function
// n: number of nodes
// start: starting node
// adj: adjacency list where adj[u] contains pairs {weight, v}
vector<long long> dijkstra(int n, int start, const vector<vector<pii>>& adj) {
    
    // 1. Initialize distances to Infinity
    vector<long long> dist(n + 1, INF); 
    dist[start] = 0;

    // 2. Min-Heap Priority Queue: Stores {current_dist, node}
    // We use 'greater' to make it a Min-Heap (default is Max-Heap)
    priority_queue<pii, vector<pii>, greater<pii>> pq;
    
    // Push the start node
    pq.push({0, start});

    while (!pq.empty()) {
        // Get the node with the smallest distance
        long long d = pq.top().first;
        int u = pq.top().second;
        pq.pop();

        // OPTIMIZATION: Stale/Redundant Entry Check
        // If we found a shorter path to 'u' before popping this specific 
        // entry from the PQ, ignore this one.
        if (d > dist[u]) continue;

        // Explore neighbors
        for (auto& edge : adj[u]) {
            long long weight = edge.first;
            int v = edge.second;

            // Relaxation Step
            if (dist[u] + weight < dist[v]) {
                dist[v] = dist[u] + weight;
                pq.push({dist[v], v});
            }
        }
    }

    return dist;
}
```

-----

### 5\. Example Walkthrough

Let's say we have this graph:

1.  **A -\> B** (Weight 4)
2.  **A -\> C** (Weight 2)
3.  **C -\> B** (Weight 1)
4.  **B -\> D** (Weight 5)

**Goal:** Shortest path from A to D.

1.  **Init:** `dist[A]=0`, others $\infty$. PQ: `{(0, A)}`
2.  **Pop A (0):**
      * Update B: `dist[B] = 0 + 4 = 4`. Push `{(4, B)}`
      * Update C: `dist[C] = 0 + 2 = 2`. Push `{(2, C)}`
      * PQ contains: `{(2, C), (4, B)}` (Ordered by weight)
3.  **Pop C (2):** (Smallest in PQ)
      * Neighbor B: `dist[C] + 1 = 2 + 1 = 3`.
      * Is $3 < dist[B]$ (which is 4)? **Yes.**
      * Update `dist[B] = 3`. Push `{(3, B)}`.
      * PQ contains: `{(3, B), (4, B)}`
4.  **Pop B (3):**
      * Neighbor D: `dist[B] + 5 = 3 + 5 = 8`.
      * Update `dist[D] = 8`. Push `{(8, D)}`.
      * PQ contains: `{(4, B), (8, D)}`
5.  **Pop B (4):**
      * **Stale check:** `d (4) > dist[B] (3)`. Ignore.
6.  **Pop D (8):** Done.

**Result:** Shortest path A -\> C -\> B -\> D is 8.

-----

### 6\. Practice Problem Example

**Problem:** "Network Delay Time"
*There are `N` network nodes, labeled 1 to N. You are given a list of travel times as directed edges `times[i] = (u, v, w)`, where `u` is the source node, `v` is the target node, and `w` is the time it takes for a signal to travel from source to target.*

*We send a signal from a given node `K`. Return the time it takes for all `N` nodes to receive the signal. If it is impossible for all nodes to receive the signal, return -1.*

**Solution Implementation:**

```cpp
#include <iostream>
#include <vector>
#include <queue>
#include <algorithm>

using namespace std;

class Solution {
public:
    int networkDelayTime(vector<vector<int>>& times, int n, int k) {
        // 1. Build Adjacency List
        // Note: The problem gives vector<int> {u, v, w}
        // We convert to adj[u] -> {weight, v}
        vector<vector<pair<int, int>>> adj(n + 1);
        for (auto& t : times) {
            adj[t[0]].push_back({t[2], t[1]}); // {weight, target}
        }

        // 2. Run Dijkstra
        priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> pq;
        vector<int> dist(n + 1, 1e9); // Using 1e9 as INF

        dist[k] = 0;
        pq.push({0, k});

        while (!pq.empty()) {
            int d = pq.top().first;
            int u = pq.top().second;
            pq.pop();

            if (d > dist[u]) continue;

            for (auto& edge : adj[u]) {
                int weight = edge.first;
                int v = edge.second;
                if (dist[u] + weight < dist[v]) {
                    dist[v] = dist[u] + weight;
                    pq.push({dist[v], v});
                }
            }
        }

        // 3. Find the max distance (since signal must reach ALL nodes)
        int maxDist = 0;
        for (int i = 1; i <= n; ++i) {
            if (dist[i] == 1e9) return -1; // Unreachable node
            maxDist = max(maxDist, dist[i]);
        }

        return maxDist;
    }
};

int main() {
    Solution sol;
    // Edges: 2->1 (1), 2->3 (1), 3->4 (1)
    vector<vector<int>> times = {{2,1,1},{2,3,1},{3,4,1}};
    int n = 4;
    int k = 2; // Start at node 2

    cout << "Max time: " << sol.networkDelayTime(times, n, k) << endl; 
    // Output should be 2 (Path 2->3->4 is cost 2)
    return 0;
}
```

### 7\. Common Pitfalls

1.  **Directed vs Undirected:** Make sure you push edges to the adjacency list correctly. If undirected, `adj[u].push({w, v})` AND `adj[v].push({w, u})`.
2.  **Integer Overflow:** If path weights can exceed `2^31 - 1`, use `long long`.
3.  **Negative Edges:** Dijkstra loops infinitely or gives wrong answers with negative edges. Use **Bellman-Ford** or **SPFA** algorithm instead.