# 🧠Hybrid Constraint Engines — System-Level Algorithms for Structured Problems

This document presents three hybrid algorithms designed to solve entire classes of constraint-driven, state-compressed, and offline problems. 
Each engine combines multiple architectural techniques to maximize clarity, scalability, and reuse.

---

## 🔷 Engine 1: MatchLayer Engine  
_Constraint Matching via Bitmask Compression, Layered Augmentation, and Offline Batching_

### 📜 Purpose  
Solves assignment and matching problems with constraints, such as:
- Task-to-agent allocation with exclusions  
- Bipartite matching under dependency rules  
- Multi-agent scheduling with overlap restrictions

### 🧱 Architecture  
| Component       | Role                                      |
|----------------|-------------------------------------------|
| **Bitmask DP** | Encodes assignment states and constraints  
| **Layered BFS + DFS** | Finds disjoint augmenting paths (Hopcroft–Karp style)  
| **Mo-style batching** | Groups queries by mask similarity for offline resolution  

### 🔧 Execution Phases  
1. Preprocess constraints into bitmask-compatible form  
2. Batch queries by shared structure  
3. Build layered graph of possible assignments  
4. Apply augmenting paths to maximize valid matches  
5. Return results in original query order

### 🚀 С++ Code Skeleton

```cpp
struct Query { int id, mask; };
sort(queries.begin(), queries.end(), [](auto& a, auto& b) {
    return a.mask < b.mask;
});

for (auto& batch : batched_queries) {
    HopcroftKarp hk(U, V);
    for (auto& edge : batch.edges) {
        if (valid_under_mask(edge, batch.mask)) {
            hk.addEdge(edge.u, edge.v);
        }
    }
    answers[batch.id] = hk.maxMatching();
}

```


---

## ⏱️ Complexity — MatchLayer Engine

**Time:**  
- O(Q · √n · m) — Q batches, each using Hopcroft–Karp

**Space:**  
- O(n + m + Q) — graph + masks + answers

---

## ⚠️ Pitfalls — MatchLayer Engine

- Requires all constraints to be encoded as bitmasks  
- Matching graph must be bipartite and static per batch  
- Batching must preserve query semantics

---

## 🔷 Engine 2: BatchUnion Engine  
_Offline Constraint Query Processor via Union–Find, Segment Tree, and Mo’s Algorithm_

### 📜 Purpose  
Processes offline queries on constrained states, such as:
- Dynamic connectivity under range updates  
- Interval-based coverage queries  
- Offline graph queries with rollback

### 🧱 Architecture

| Component       | Role                                         |
|----------------|----------------------------------------------|
| Mo’s Algorithm | Sorts and batches queries for minimal recomputation |
| Union–Find     | Tracks dynamic connectivity or grouping      |
| Segment Tree   | Propagates range-based updates efficiently   |

### 🔧 Execution Phases

1. Sort queries by block and secondary key  
2. Apply union-find to track evolving structure  
3. Use segment tree to apply and rollback updates  
4. Resolve each query in batch context  
5. Restore answers to original order

### 🚀 С++ Code Skeleton

```cpp
struct Query { int l, r, id; };
sort(queries.begin(), queries.end(), mo_cmp);

int currL = 0, currR = -1;
UnionFind uf(n);
SegmentTree seg(n);

for (auto& q : queries) {
    while (currR < q.r) apply_add(++currR);
    while (currL > q.l) apply_add(--currL);
    while (currR > q.r) apply_remove(currR--);
    while (currL < q.l) apply_remove(currL++);

    answers[q.id] = resolve_query(uf, seg);
}
```


---

## ⏱️ Complexity — BatchUnion Engine

**Time:**  
- O(Q · √n · log n) — Mo’s traversal + segment tree operations

**Space:**  
- O(n + Q) — union-find + segment tree + answers

---

## ⚠️ Pitfalls — BatchUnion Engine

- Only works for offline queries  
- Rollback logic must be carefully managed  
- Union–Find must support undo or persistent mode

---

## 🔷 Engine 3: PropagateMask Engine  
_Subset Assignment with Layered Propagation and Lazy State Updates_

### 📜 Purpose  
Solves subset-based assignment problems with constraints, such as:
- Partitioning under global rules  
- Graph traversal with memory  
- Scheduling with forbidden overlaps

### 🧱 Architecture

| Component     | Role                                      |
|--------------|-------------------------------------------|
| Bitmask       | Encodes subset states and transitions     |
| BFS layering  | Builds propagation levels across state space |
| Lazy updates  | Applies transitions only when triggered   |

### 🔧 Execution Phases

1. Encode initial state as bitmask  
2. Build BFS layers of reachable configurations  
3. Apply lazy updates to propagate valid transitions  
4. Track visited states and resolve final configuration  
5. Return result or count of valid assignments

### 🚀 С++ Code Skeleton

```cpp
int fullMask = (1 << n) - 1;
queue<int> q;
vector<int> dist(1 << n, -1);
q.push(0); dist[0] = 0;

while (!q.empty()) {
    int mask = q.front(); q.pop();
    for (int i = 0; i < n; ++i) {
        if (!(mask & (1 << i)) && valid(mask, i)) {
            int next = mask | (1 << i);
            if (dist[next] == -1) {
                dist[next] = dist[mask] + 1;
                q.push(next);
            }
        }
    }
}
```

---

## ⏱️ Complexity — PropagateMask Engine

**Time:**  
- O(2ⁿ · n) — full mask traversal

**Space:**  
- O(2ⁿ) — visited states

---

## ⚠️ Pitfalls — PropagateMask Engine

- Only feasible for small n (typically n ≤ 20)  
- Requires `valid()` function to be fast and stateless  
- BFS layering must avoid redundant transitions

---

## 🧩 Summary Table

| Engine Name     | Problem Class                         | Core Techniques                             |
|----------------|----------------------------------------|---------------------------------------------|
| **MatchLayer**   | Matching with constraints              | Bitmask + Layered BFS/DFS + Mo batching     |
| **BatchUnion**   | Offline queries on constrained states  | Mo’s + Union–Find + Segment Tree            |
| **PropagateMask**| Subset assignment with propagation     | Bitmask + BFS layering + Lazy updates       |

---

## ✅ Final Notes

Each engine in this document is designed to be:

- **Modular and extensible** — ready to adapt to new constraints and problem variants  
- **Architecturally clear** — built around triggers, phases, and minimal recall  
- **System-level ready** — suitable for integration into educational wikis, algorithm libraries, or competitive templates

---


### 🧠Architecture

These engines reflect a design philosophy focused on:

- **Trigger-based execution**  
- **Phase separation**  
- **State compression**  
- **Offline optimization**  
- **Minimal recall documentation**

They are not just solutions — they are **thinking systems**.






---
