# 🧠 Tarjan’s Algorithm — Bridges & Articulation Points Detection

---

## 📜 Origin & Motivation

Tarjan’s Algorithm was introduced in 1972 by **Robert Tarjan**, a pioneer in graph theory and data structures.  
It was designed to efficiently identify **critical components** in undirected graphs — specifically:

- **Bridges**: edges whose removal increases the number of connected components  
- **Articulation points**: vertices whose removal disconnects the graph

These concepts are vital in:

- 🛠️ Network reliability  
- 🔌 Circuit design  
- 🔗 Connectivity analysis  
- 🧠 Structural graph insights

---

## 💡 Core Idea

Tarjan’s algorithm performs a **DFS traversal**, assigning each node:

- `disc[u]`: discovery time — when the node is first visited  
- `low[u]`: low-link value — the earliest reachable ancestor via DFS or back edge

By comparing these values, we detect:

- **Bridge**: if `low[v] > disc[u]`, then edge `(u, v)` is a bridge  
- **Articulation point**: if `low[v] ≥ disc[u]` and `u` is not the DFS root, then `u` is an articulation point

The algorithm runs in **O(V + E)** time, making it ideal for large sparse graphs.

---

## 🧩 Where It’s Used

- 🧱 Network design — identifying weak links in communication or power grids  
- 👥 Social graphs — finding influencers or critical connectors  
- 🧮 Compiler design — control flow analysis  
- 🗺️ Game maps — detecting chokepoints or strategic zones  
- 🔐 Security — vulnerability analysis in system graphs

---

## 🔁 When to Use Tarjan vs Other Graph Algorithms

| Task                      | Use Tarjan | Use DFS | Use Union-Find |
|---------------------------|------------|---------|----------------|
| Find bridges              | ✅         | ❌      | ❌             |
| Find articulation points  | ✅         | ❌      | ❌             |
| Check connectivity        | ❌         | ✅      | ✅             |
| Detect cycles             | ❌         | ✅      | ✅             |
| Dynamic connectivity      | ❌         | ❌      | ✅             |

---

## 🧱 Tarjan’s Algorithm (C++)

```cpp
void tarjan(int u, int parent, vector<vector<int>>& adj,
            vector<int>& disc, vector<int>& low,
            vector<bool>& visited, vector<bool>& isArticulation,
            vector<pair<int,int>>& bridges, int& time) {
    visited[u] = true;
    disc[u] = low[u] = ++time;
    int children = 0;

    for (int v : adj[u]) {
        if (!visited[v]) {
            children++;
            tarjan(v, u, adj, disc, low, visited, isArticulation, bridges, time);
            low[u] = min(low[u], low[v]);

            if (low[v] > disc[u])
                bridges.push_back({u, v});

            if (parent != -1 && low[v] >= disc[u])
                isArticulation[u] = true;
        }
        else if (v != parent) {
            low[u] = min(low[u], disc[v]);
        }
    }

    if (parent == -1 && children > 1)
        isArticulation[u] = true;
}
```

---

## 🧱 Tarjan’s Algorithm (C#)
- Here’s the equivalent C# version using List<List<int>> and recursion. 
- It mirrors the same logic and structure as the C++ version, with idiomatic C# syntax.

```csharp
void Tarjan(int u, int parent, List<List<int>> adj,
            int[] disc, int[] low, bool[] visited,
            bool[] isArticulation, List<(int, int)> bridges, ref int time) {
    visited[u] = true;
    disc[u] = low[u] = ++time;
    int children = 0;

    foreach (int v in adj[u]) {
        if (!visited[v]) {
            children++;
            Tarjan(v, u, adj, disc, low, visited, isArticulation, bridges, ref time);
            low[u] = Math.Min(low[u], low[v]);

            if (low[v] > disc[u])
                bridges.Add((u, v));

            if (parent != -1 && low[v] >= disc[u])
                isArticulation[u] = true;
        }
        else if (v != parent) {
            low[u] = Math.Min(low[u], disc[v]);
        }
    }

    if (parent == -1 && children > 1)
        isArticulation[u] = true;
}
```

## ⏱️ Complexity Analysis

---

### 🔧 Time Complexity

Tarjan’s algorithm runs in **linear time**, proportional to the size of the graph:

- **DFS traversal**: `O(V + E)`  
  - Each vertex is visited once  
  - Each edge is explored once  
  - No redundant computations or revisits

This makes it highly efficient for **sparse graphs**, where `E ≈ V`.

---

### 💾 Space Complexity

Memory usage is also linear, with the following components:

- `O(V)` for:
  - `disc[]` — discovery times for each vertex  
  - `low[]` — low-link values tracking earliest reachable ancestor  
  - `visited[]` — boolean flags for DFS traversal  
  - `isArticulation[]` — flags marking articulation points

- `O(E)` for:
  - Adjacency list representation of the graph  
  - Each edge stored once (undirected)

No auxiliary stacks or recursion depth tracking are required beyond standard DFS.

---

## 🧠 Key Concepts Recap

Let’s reinforce the core mechanics:

- **Discovery time (`disc[u]`)**  
  Timestamp when node `u` is first visited during DFS

- **Low-link value (`low[u]`)**  
  The smallest discovery time reachable from `u`, including back edges

- **Bridge condition**  
  If `low[v] > disc[u]`, then edge `(u, v)` is a **bridge** — its removal disconnects the graph

- **Articulation point condition**  
  If `low[v] ≥ disc[u]` and `u ≠ root`, then `u` is an **articulation point** — its removal increases the number of connected components

- **Root special case**  
  If DFS root has **2 or more children**, it is an articulation point

These conditions are checked **during DFS**, not after — making the algorithm **online and efficient**.

---

## ✅ Conclusion

Tarjan’s Algorithm is more than a traversal — it’s a **connectivity scanner** that exposes the **structural integrity** of a graph.

Whether you're working on:

- 🧱 **Network design** — finding weak links  
- 🧠 **Control flow analysis** — detecting critical blocks  
- 🗺️ **Game maps** — identifying chokepoints  
- 🔐 **Security graphs** — locating vulnerable nodes

Tarjan gives you:

- ⚡ **Linear-time performance**  
- 🧠 **Deep structural insight**  
- 🛠️ **Precise detection of bridges and articulation points**

It transforms raw graph data into a **resilience blueprint** — showing what holds the system together, and what could break it apart.

---


