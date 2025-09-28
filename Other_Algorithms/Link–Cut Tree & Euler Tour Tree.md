# 🧠 Link–Cut Tree & Euler Tour Tree — Dynamic Tree Data Structures

---

## 📜 Origin & Motivation

Dynamic trees are data structures that maintain a forest of rooted trees under operations like:

- **Link(u, v)** — connect two trees by adding edge `(u, v)`
- **Cut(u, v)** — remove edge `(u, v)`
- **Query(u, v)** — compute aggregate information along the path or subtree

These operations are essential in:

- 🧮 Dynamic connectivity  
- 🧠 Heavy-light decomposition  
- 🗺️ Network updates  
- 🏁 Online tree algorithms

### 🔹 Link–Cut Tree

Introduced by **Daniel Sleator and Robert Tarjan** in 1983, Link–Cut Trees use **splay trees** to represent paths and support efficient path queries and updates.

### 🔹 Euler Tour Tree

Euler Tour Trees represent each tree as a **sequence of visits** in an Euler tour and store it in a **balanced BST** (e.g., Treap, Splay, Segment Tree).  
They support subtree queries and dynamic connectivity with simpler implementation.

---

## 💡 Core Idea

### 🔹 Link–Cut Tree

- Each node maintains a **preferred path** to its parent
- Paths are stored in **auxiliary trees** (splay trees)
- Operations like `access`, `link`, `cut`, and `findRoot` are supported in `O(log N)` time

### 🔹 Euler Tour Tree

- Perform an **Euler tour traversal** of the tree
- Store the tour in a balanced BST
- Subtree queries become **range queries**
- Updates are done by splitting and merging tour segments

Both structures allow **online updates** and **fast queries** on dynamic forests.

---

## 🧩 Where It’s Used

- 🔗 Dynamic connectivity in forests  
- 🧠 Path and subtree queries  
- 🏁 Online algorithms for trees  
- 📈 Network design and updates  
- 🧮 Heavy-light decomposition (as a backend)  
- 🧊 Dynamic LCA and rerooting

---

## 🔁 When to Use Link–Cut vs Euler Tour Tree

| Task                        | Link–Cut Tree | Euler Tour Tree |
|-----------------------------|---------------|------------------|
| Path queries (u to v)       | ✅            | ❌               |
| Subtree queries             | ❌            | ✅               |
| Link/Cut operations         | ✅            | ✅               |
| Rerooting                   | ✅            | ✅               |
| Simpler implementation      | ❌            | ✅               |
| Supports LCA                | ✅            | ❌               |

---

## 🧱 Link–Cut Tree (C++ Skeleton)

Link–Cut Tree uses **splay trees** to represent paths in a forest.  
Each node maintains its own subtree value and a `revert` flag to support rerooting.

### 🔧 Core Mechanics

- `push()` propagates the `revert` flag to children  
- `update()` recalculates subtree aggregates  
- Operations like `access`, `link`, `cut`, and `findRoot` are built on top of splay logic

### 🧩 C++ Skeleton

```cpp
struct Node {
    Node *left, *right, *parent;
    bool revert;
    int value, subtreeValue;

    void push() {
        if (revert) {
            swap(left, right);
            if (left) left->revert ^= true;
            if (right) right->revert ^= true;
            revert = false;
        }
    }

    void update() {
        subtreeValue = value;
        if (left) subtreeValue += left->subtreeValue;
        if (right) subtreeValue += right->subtreeValue;
    }
};

// Core operations:
// - access(Node* u)
// - link(Node* u, Node* v)
// - cut(Node* u, Node* v)
// - findRoot(Node* u)
```


## 🧱 Link–Cut Tree (C# Skeleton)
This version mirrors the C++ logic using object-oriented C#. 
It’s ideal for dynamic forests where path queries and rerooting are required.

🧩 C# Skeleton
```csharp
class Node {
    public Node Left, Right, Parent;
    public bool Revert;
    public int Value, SubtreeValue;

    public void Push() {
        if (Revert) {
            (Left, Right) = (Right, Left);
            if (Left != null) Left.Revert ^= true;
            if (Right != null) Right.Revert ^= true;
            Revert = false;
        }
    }

    public void Update() {
        SubtreeValue = Value;
        if (Left != null) SubtreeValue += Left.SubtreeValue;
        if (Right != null) SubtreeValue += Right.SubtreeValue;
    }
}
// Core operations:
// Access(Node u), Link(Node u, Node v), Cut(Node u, Node v), FindRoot(Node u)
```

## 🧱 Euler Tour Tree (C++ Skeleton)

Euler Tour Tree represents each tree as a sequence of visits in an Euler tour. 
Subtree queries become range queries over the tour array.

## 🔧 Core Mechanics

- dfs() builds the Euler tour
- build() initializes the tour and segment tree
- Subtree queries = range queries
- Link/Cut = split/merge tour segments

🧩 C++ Skeleton
```cpp
struct EulerTourTree {
    vector<int> tour;
    SegmentTree seg;

    void dfs(int u, int p) {
        tour.push_back(u);
        for (int v : adj[u]) {
            if (v == p) continue;
            dfs(v, u);
            tour.push_back(u);
        }
    }

    void build(int root) {
        tour.clear();
        dfs(root, -1);
        seg.build(tour);
    }

    // Subtree query = range query over tour
    // Link/Cut = split/merge tour segments
};
```

## 🧱 Euler Tour Tree (C# Skeleton)
This version uses a list to store the Euler tour and a segment tree for range queries. 
It’s ideal for subtree-centric logic and dynamic connectivity.

## 🧩 C# Skeleton
```csharp
class EulerTourTree {
    public List<int> Tour = new();
    public SegmentTree Seg;

    public void DFS(int u, int p, List<List<int>> adj) {
        Tour.Add(u);
        foreach (int v in adj[u]) {
            if (v == p) continue;
            DFS(v, u, adj);
            Tour.Add(u);
        }
    }

    public void Build(int root, List<List<int>> adj) {
        Tour.Clear();
        DFS(root, -1, adj);
        Seg = new SegmentTree(Tour);
    }

    // Subtree query = Seg.Query(l, r)
    // Link/Cut = split/merge Tour segments
}
```

## ⏱️ Complexity Analysis

---

### 🔧 Time Complexity

| Operation        | Link–Cut Tree | Euler Tour Tree |
|------------------|---------------|------------------|
| Link / Cut       | `O(log N)`    | `O(log N)`       |
| Path query       | `O(log N)`    | ❌               |
| Subtree query    | ❌            | `O(log N)`       |
| Reroot           | `O(log N)`    | `O(log N)`       |

- **Link–Cut Tree** uses **splay trees** to represent preferred paths.  
  Each operation like `access`, `link`, `cut`, and `findRoot` is performed in `O(log N)` amortized time.

- **Euler Tour Tree** represents the tree as a **sequence of visits** in an Euler tour.  
  Subtree queries become **range queries** over the tour, supported by segment trees or balanced BSTs.

- Both structures support **dynamic updates** and **logarithmic-time operations**, but differ in query capabilities:
  - Link–Cut Tree excels at **path queries**
  - Euler Tour Tree excels at **subtree queries**

---

### 💾 Space Complexity

- **Link–Cut Tree**:  
  - Uses `O(N)` space for nodes and auxiliary splay trees  
  - Each node stores parent, children, and metadata

- **Euler Tour Tree**:  
  - Uses `O(N)` for storing Euler tour and node metadata  
  - May require `O(N log N)` if built on segment trees or treaps  
  - Simpler structure, but heavier if range queries are complex

---

## 🧠 Key Concepts Recap

Let’s reinforce the architectural essence:

- **Link–Cut Tree**  
  - Path-centric  
  - Splay-based  
  - Powerful but complex  
  - Ideal for path queries and LCA

- **Euler Tour Tree**  
  - Subtree-centric  
  - Tour-based  
  - Simpler and modular  
  - Ideal for subtree queries and rerooting

- Both support:
  - 🔗 Dynamic connectivity  
  - 🔄 Online updates  
  - 🧠 Real-time forest manipulation

- **Design choice**:  
  Choose based on query type:
  - Use **Link–Cut Tree** for path-based logic  
  - Use **Euler Tour Tree** for subtree-based logic

---

## ✅ Conclusion

**Link–Cut Tree** and **Euler Tour Tree** are **dynamic tree engines** — they allow real-time updates and queries on mutable forests.

Whether you're building:

- 🧠 Online tree algorithms  
- 🧮 Dynamic connectivity systems  
- 🏁 Competitive programming solutions  
- 🗺️ Network rerouting logic

These tools give you:

- ⚡ **Logarithmic-time operations**  
- 🧠 **Structural flexibility**  
- 🛠️ **Architectural control over trees**

They’re not just data structures — they’re **dynamic routers** for tree-based systems.  
Once mastered, they become part of your **graph toolkit**, ready to deploy when static trees aren’t enough.

---

