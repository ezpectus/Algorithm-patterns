# 🧠 Gabow’s Algorithm — Stack-Based SCC Engine

## 📜 Origin & Motivation

**Gabow’s Algorithm** was introduced by **Harold N. Gabow** in **2000**, in his paper  
*"Path-based depth-first search for strong and biconnected components"*.

It was designed as a **non-recursive alternative** to Tarjan’s algorithm for finding **strongly connected components (SCCs)** in directed graphs.

### 🔑 Key Innovations:
- Uses **two stacks** to track DFS path and SCC boundaries  
- Avoids recursion depth issues  
- Maintains **linear time complexity**:  
  ```
  O(V + E)
  ```

### 🔄 Compared to:
| Algorithm         | Year | Method         | Notes                          |
|------------------|------|----------------|--------------------------------|
| Kosaraju         | 1978 | Two DFS passes | Simple but requires graph reversal |
| Tarjan           | 1972 | Recursive DFS  | Uses low-link values, recursion-heavy |
| **Gabow**         | 2000 | Stack-based DFS| Iterative, recursion-safe, tight control |

---

## 🧩 Where It’s Used

- 🔁 Detecting **strongly connected components** in directed graphs  
- 🧱 Building **condensation graphs** (DAG of SCCs)  
- 🧠 Analyzing **cycles** and **reachability**  
- 🛡️ Systems with **limited recursion depth** (embedded, constrained platforms)  
- 🏆 Competitive programming: fast, stack-safe alternative to Tarjan

---

## 🔁 When to Use Gabow vs Alternatives

| Scenario / Task                      | Gabow | Tarjan | Kosaraju |
|--------------------------------------|--------|--------|----------|
| Recursion-safe SCC detection         | ✅     | ❌     | ✅       |
| Stack-based control                  | ✅     | ❌     | ❌       |
| Simple implementation                | ❌     | ✅     | ✅       |
| Condensation graph construction      | ✅     | ✅     | ✅       |
| Memory-sensitive environments        | ✅     | ❌     | ❌       |
| Educational clarity                  | ❌     | ✅     | ✅       |

---

## 🧱 Core Idea

Gabow’s algorithm performs a **DFS traversal** while maintaining two stacks:

- `S` — tracks the current DFS path  
- `P` — tracks potential SCC boundaries

Each node is assigned a **discovery index**.  
When a node finishes, if it’s at the top of `P`, we pop nodes from `S` to form an SCC.

This avoids recursion and low-link calculations, relying purely on **stack state**.

---

## 🧩 Phase Breakdown

### 🔹 Phase 1: Initialization

Prepare:
- `graph[u]` — adjacency list  
- `index[] = -1` — discovery index  
- `visited[] = false` — active status  
- `stackS`, `stackP` — empty  
- `components[]` — list of SCCs  
- `idx = 0` — discovery counter

---

### 🔹 Phase 2: DFS Traversal

For each unvisited node `u`:
1. Assign `index[u] = idx++`  
2. Push `u` to both `stackS` and `stackP`  
3. Recurse on neighbors `v`:
   - If `index[v] == -1`: DFS on `v`  
   - If `visited[v] == true`:  
     - While `index[stackP.top()] > index[v]`, pop `stackP`

---

### 🔹 Phase 3: SCC Extraction

If `stackP.top() == u`:
1. Pop `stackP`  
2. Pop nodes from `stackS` until `u`  
3. Mark them as inactive  
4. Store as one SCC in `components[]`

---

## 🚀 Implementation (C++)

```cpp
void gabowSCC(int u, vector<vector<int>>& graph, vector<int>& index,
              vector<int>& stackS, vector<int>& stackP, vector<bool>& visited,
              vector<vector<int>>& components, int& idx) {
    index[u] = idx++;
    stackS.push_back(u);
    stackP.push_back(u);
    visited[u] = true;

    for (int v : graph[u]) {
        if (index[v] == -1)
            gabowSCC(v, graph, index, stackS, stackP, visited, components, idx);
        else if (visited[v]) {
            while (!stackP.empty() && index[stackP.back()] > index[v])
                stackP.pop_back();
        }
    }

    if (!stackP.empty() && stackP.back() == u) {
        stackP.pop_back();
        vector<int> component;
        int v;
        do {
            v = stackS.back();
            stackS.pop_back();
            visited[v] = false;
            component.push_back(v);
        } while (v != u);
        components.push_back(component);
    }
}

```


## 🚀 Implementation (C#)
```csharp
void GabowSCC(int u, List<List<int>> graph, int[] index, Stack<int> stackS,
              Stack<int> stackP, bool[] visited, List<List<int>> components, ref int idx) {
    index[u] = idx++;
    stackS.Push(u);
    stackP.Push(u);
    visited[u] = true;

    foreach (int v in graph[u]) {
        if (index[v] == -1)
            GabowSCC(v, graph, index, stackS, stackP, visited, components, ref idx);
        else if (visited[v]) {
            while (stackP.Count > 0 && index[stackP.Peek()] > index[v])
                stackP.Pop();
        }
    }

    if (stackP.Count > 0 && stackP.Peek() == u) {
        stackP.Pop();
        var component = new List<int>();
        int v;
        do {
            v = stackS.Pop();
            visited[v] = false;
            component.Add(v);
        } while (v != u);
        components.Add(component);
    }
}
```
## ⏱️ Complexity Analysis

| Metric               | Value                        |
|----------------------|------------------------------|
| Time Complexity      | \( O(V + E) \) — linear in graph size |
| Space Complexity     | \( O(V) \) — stacks and arrays |
| Recursion Depth      | Avoidable (can be iterative) |
| SCC Count            | ≤ V                          |
| Preprocessing        | None — works directly on adjacency list |
| SCC Granularity      | Efficient on dense graphs with small cycles |
| Parallelization      | Post-SCC processing is parallelizable |

---

## ⚠️ Pitfalls

- **Indexing:** `index[]` must start at −1  
- **StackP logic:** Only pop when `index[top] > index[v]`  
- **Visited flags:** Must reset when node exits SCC  
- **Graph cycles:** Gabow handles them cleanly via stack boundaries  
- **Disconnected components:** Must run DFS from every unvisited node  
- **Graph representation:** Use adjacency list for efficiency  
- **Component ordering:** SCCs appear in reverse topological order

---

## ✅ Conclusion

Gabow’s Algorithm is a **stack-based SCC engine** that excels in environments where **recursion is dangerous** and **stack control is critical**.

- 🧠 Avoids recursion via dual-stack strategy  
- ⚡ Linear time, tight control over DFS  
- 🛡️ Ideal for constrained environments and embedded systems  
- 🔗 Produces clean SCC partitions for DAG condensation  
- 🧩 Handles cycles, nested components, and deep graphs with precision

👉 **Key takeaway:** Gabow’s method is the **low-level, stack-safe alternative to Tarjan** — perfect when recursion is risky, control is everything, and performance must remain linear.



---
