# Persistent Segment Tree — Versioned Range Query Engine

---

## Origin & Motivation

The Persistent Segment Tree is a powerful data structure designed to support range queries across multiple versions of an array.  
Unlike traditional segment trees that overwrite data during updates, the persistent variant preserves historical states by creating new versions with each update.  
This is achieved through partial copying: only the nodes affected by the update are duplicated, while the rest are shared across versions.

This structure is ideal for problems involving:
- Time-travel queries  
- Rollbacks and undo operations  
- Historical data analysis  
- K-th order statistics across versions

It is widely used in competitive programming and systems requiring immutable data snapshots.

---

## Architecture

| Component       | Description                                                  |
|----------------|--------------------------------------------------------------|
| `Node`          | Represents a segment tree node with value and child pointers |
| `Version[]`     | Stores root nodes of all versions                            |
| `Build()`       | Constructs the initial segment tree from input array         |
| `Update()`      | Creates a new version with a point update                    |
| `Query()`       | Performs range queries on a specific version                 |

Each update creates a new root node and duplicates only O(log n) nodes along the update path.  
All other nodes are shared, ensuring efficient memory usage.

---

## Execution Phases

### 1. Build Phase
- Recursively construct the initial segment tree from the input array.
- Each leaf node stores a single element; internal nodes store aggregates (e.g., sum, min, max).

### 2. Update Phase
- To update a value at index `i`, create a new version by copying nodes along the path from root to leaf.
- Only O(log n) nodes are duplicated; unchanged nodes are reused.

### 3. Query Phase
- Perform standard segment tree queries (e.g., sum over range `[l, r]`) on any version by traversing its root.

---

## C++ Implementation

```cpp
struct Node {
    int val;
    Node *left, *right;
    Node(int v = 0) : val(v), left(nullptr), right(nullptr) {}
};

Node* build(int l, int r, const vector<int>& arr) {
    if (l == r) return new Node(arr[l]);
    int m = (l + r) / 2;
    Node* node = new Node();
    node->left = build(l, m, arr);
    node->right = build(m + 1, r, arr);
    node->val = node->left->val + node->right->val;
    return node;
}

Node* update(Node* prev, int l, int r, int idx, int val) {
    if (l == r) return new Node(val);
    int m = (l + r) / 2;
    Node* node = new Node();
    if (idx <= m) {
        node->left = update(prev->left, l, m, idx, val);
        node->right = prev->right;
    } else {
        node->left = prev->left;
        node->right = update(prev->right, m + 1, r, idx, val);
    }
    node->val = node->left->val + node->right->val;
    return node;
}

int query(Node* node, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) return 0;
    if (ql <= l && r <= qr) return node->val;
    int m = (l + r) / 2;
    return query(node->left, l, m, ql, qr) + query(node->right, m + 1, r, ql, qr);
}
```

## C# Implementation
```cpp
class Node {
    public int Val;
    public Node Left, Right;
    public Node(int val = 0) {
        Val = val;
        Left = Right = null;
    }
}

class PersistentSegmentTree {
    public Node Build(int l, int r, int[] arr) {
        if (l == r) return new Node(arr[l]);
        int m = (l + r) / 2;
        var node = new Node();
        node.Left = Build(l, m, arr);
        node.Right = Build(m + 1, r, arr);
        node.Val = node.Left.Val + node.Right.Val;
        return node;
    }

    public Node Update(Node prev, int l, int r, int idx, int val) {
        if (l == r) return new Node(val);
        int m = (l + r) / 2;
        var node = new Node();
        if (idx <= m) {
            node.Left = Update(prev.Left, l, m, idx, val);
            node.Right = prev.Right;
        } else {
            node.Left = prev.Left;
            node.Right = Update(prev.Right, m + 1, r, idx, val);
        }
        node.Val = node.Left.Val + node.Right.Val;
        return node;
    }

    public int Query(Node node, int l, int r, int ql, int qr) {
        if (qr < l || r < ql) return 0;
        if (ql <= l && r <= qr) return node.Val;
        int m = (l + r) / 2;
        return Query(node.Left, l, m, ql, qr) + Query(node.Right, m + 1, r, ql, qr);
    }
}
```

## Complexity Analysis

| Operation | Time Complexity | Description |
|-----------|------------------|-------------|
| Build     | O(n)             | Constructs initial tree from array |
| Update    | O(log n)         | Creates new version with point update |
| Query     | O(log n)         | Performs range query on any version |
| Space     | O(n log n)       | Total space across all versions |

Each update creates a new root and O(log n) new nodes.  
All other nodes are shared, making it memory-efficient for many versions.

---

## Pitfalls

- Only supports point updates by default.  
  Range updates require lazy propagation, which adds complexity and must be handled carefully.

- Each version must be tracked explicitly.  
  Losing a root pointer means losing access to that version permanently.

- Debugging across versions can be difficult.  
  Use visual tools, version indexing, or structured logging to trace updates and queries.

- Memory usage grows with the number of versions.  
  Avoid excessive updates without pruning or compression strategies.

---

## Use Cases

- Historical range queries (e.g., sum in version k)  
- K-th order statistics using persistent trees with coordinate compression  
- Rollback systems in editors, games, or simulations  
- Time-travel queries in competitive programming  
- Immutable data snapshots for audit, verification, or forensic analysis

---

## Summary

The Persistent Segment Tree is a versioned data structure that enables efficient range queries across multiple historical states.  
It combines the power of segment trees with partial persistence, making it ideal for problems involving history, rollback, and immutable queries.  
Its logarithmic update and query time, along with shared memory across versions, make it both fast and space-efficient.




---
