# 🧠 Link/Cut Trees — Splay-Based Dynamic Tree Engine (Path Queries, Updates, Cuts)

---

## 📜 Origin & Motivation

Link/Cut Trees were introduced by **Daniel Sleator** and **Robert Tarjan** in 1983 to support **dynamic trees** — forests of rooted trees that change over time.  
They allow efficient operations like:

- `link(u, v)`: connect node `u` as a child of `v`  
- `cut(u)`: remove the edge between `u` and its parent  
- `pathQuery(u, v)`: query or update values along the path from `u` to `v`

The core idea is to represent each tree as a collection of **splay trees**, enabling fast access and restructuring via rotations.

---

## 🧩 Use Cases

- 🌲 Dynamic connectivity in forests  
- 📊 Path queries and updates (e.g., sum, max, min)  
- 🧩 Euler tour tree alternatives  
- 🎮 Competitive programming: dynamic trees with fast link/cut

---

## 🔁 When to Use Link/Cut Trees vs Alternatives

| Scenario                          | Link/Cut Tree | Euler Tour Tree | Heavy-Light | Segment Tree |
|----------------------------------|---------------|------------------|--------------|---------------|
| Dynamic edge insert/delete       | ✅            | ✅               | ❌           | ❌            |
| Path queries between any nodes   | ✅            | ❌               | ✅           | ❌            |
| Subtree queries                  | ❌            | ✅               | ✅           | ✅            |
| Online updates                   | ✅            | ✅               | ✅           | ✅            |
| Tree topology changes            | ✅            | ✅               | ❌           | ❌            |

---

## 🧱 Core Architecture

### 🎯 Triggers

| Condition                        | Action in Code           |
|----------------------------------|--------------------------|
| Need to expose path u–v         | Splay and access nodes   |
| Need to link u to v             | Make u root, attach to v |
| Need to cut u from parent       | Splay u, remove left child |
| Need to query path u–v          | Expose path, aggregate   |

---

### 🔧 Algorithm Steps

1. Represent each node as a splay tree node  
2. Use `access(u)` to expose the path from root to `u`  
3. Use `makeRoot(u)` to reroot the tree at `u`  
4. Use `link(u, v)` to connect `u` under `v`  
5. Use `cut(u)` to disconnect `u` from its parent  
6. Use `query(u, v)` to aggregate along the path

---

## 🚀 C++ Skeleton

```cpp
struct Node {
    Node *left, *right, *parent;
    bool revert;
    int value, aggregate;

    Node(int val) : left(nullptr), right(nullptr), parent(nullptr), revert(false), value(val), aggregate(val) {}
};

void push(Node* x);
void update(Node* x);
bool isRoot(Node* x);
void rotate(Node* x);
void splay(Node* x);
void access(Node* x);
void makeRoot(Node* x);
void link(Node* u, Node* v);
void cut(Node* u);
int query(Node* u, Node* v);

```



## 🚀 C# Implementation (Skeleton)

```csharp
public class LinkCutNode {
    public LinkCutNode Left, Right, Parent;
    public bool Revert;
    public int Value, Aggregate;

    public LinkCutNode(int value) {
        Value = value;
        Aggregate = value;
    }

    public void Push() {
        if (Revert) {
            (Left, Right) = (Right, Left);
            if (Left != null) Left.Revert ^= true;
            if (Right != null) Right.Revert ^= true;
            Revert = false;
        }
    }

    public void Update() {
        Aggregate = Value;
        if (Left != null) Aggregate += Left.Aggregate;
        if (Right != null) Aggregate += Right.Aggregate;
    }

    public bool IsRoot() => Parent == null || (Parent.Left != this && Parent.Right != this);
}

public class LinkCutTree {
    public void Rotate(LinkCutNode x) { /* Splay rotation logic */ }
    public void Splay(LinkCutNode x) { /* Splay logic with push/update */ }
    public void Access(LinkCutNode x) { /* Expose path from root to x */ }
    public void MakeRoot(LinkCutNode x) { /* Reroot tree at x */ }
    public void Link(LinkCutNode u, LinkCutNode v) { /* Connect u under v */ }
    public void Cut(LinkCutNode u) { /* Remove edge between u and its parent */ }
    public int Query(LinkCutNode u, LinkCutNode v) { /* Aggregate value along path u–v */ }
}
```

# ⏱️ Complexity Analysis  
- Splay depth: O(log n) — each splay operation restructures tree in amortized logarithmic time  
- Access path: O(log n) — exposes path from root to node  
- Total time complexity: O(log n) per operation — link, cut, query, update  
- Space complexity: O(n) — one splay node per tree node

# ⚠️ Pitfalls  
- 🔁 Splay tree must maintain correct parent pointers and rotations  
- 🧹 Revert flags must be pushed down before rotations to preserve tree orientation  
- ⚠️ Forgetting to update aggregates after splay leads to incorrect query results  
- 🧩 Cycles must be prevented — link only if nodes are in different trees  
- 🧠 Path exposure must be done before any query or update — otherwise stale structure

# ✅ Conclusion  
Link/Cut Trees are a splay-based engine for dynamic tree manipulation:

- 🔁 Support fast link, cut, and path queries  
- 📊 Maintain tree topology under edge insertions and deletions  
- 🧠 Ideal for problems where tree structure changes over time  
- 🏆 A powerful tool for dynamic connectivity and path-based aggregation

👉 **Key takeaway**: Link/Cut Trees transform mutable forests into queryable, updatable structures — elegant, efficient, and indispensable for dynamic tree problems.


---
