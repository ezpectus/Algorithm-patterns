# Union-Find (DSU with Rollback)  
*Persistent Connectivity — Undo in O(1), O(log n) amortized*

---

## Origin & Motivation  
**Disjoint Set Union (DSU)** — the **gold standard** for dynamic **set merging** and **connectivity queries**.

**DSU with Rollback** takes it to the next level:  
- **Undo** the last `union` operation  
- **Offline query processing**  
- **Divide & Conquer over time**  
- **Dynamic graphs with reversibility**

> **The magic**:  
> **No path compression** + **union by size/rank** + **change log stack**

---

## Where It’s Used  

| Domain | Use Case |
|-------|----------|
| **Offline Dynamic Connectivity** | Yes |
| **Divide & Conquer on Queries** | Yes |
| **Competitive Programming** | Codeforces, AtCoder (rollback DSU tasks) |
| **Persistent Data Structures** | Partial |
| **Online Real-Time Queries** | No (rollback inefficient) |

---

## When to Use DSU with Rollback  

| Requirement | **Rollback DSU** | **Classic DSU** |
|------------|------------------|-----------------|
| **Undo operations** | Yes | No |
| **Offline queries** | Yes | Warning (awkward) |
| **Online queries** | No | Yes |
| **Path compression** | No | Yes |
| **Simplicity** | Warning | Yes |
| **Time D&C** | Yes | No |

> **Use Rollback DSU when**:  
> - You need to **undo unions**  
> - Queries are **offline**  
> - You're doing **time-based divide & conquer**

---

## Core Idea — Step by Step  

### **1. Store core state**  
```cpp
vector<int> parent, size;
```
### 2. Union by size (NO path compression!)
```
if (size[a] < size[b]) swap(a, b);
parent[b] = a;
size[a] += size[b];
```

### 3. Log every change to a stack
```
history.push({b, old_parent_of_b, old_size_of_a});
```

### 4. Rollback — restore from stack
```
auto [v, old_p, old_s] = history.top();
parent[v] = old_p;
size[parent[v]] = old_s;
```

## Full Implementation (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

class RollbackDSU {
private:
    struct Change {
        int v;           // child node
        int old_parent;  // previous parent
        int old_size;    // previous size of new parent
        Change(int v, int p, int s) : v(v), old_parent(p), old_size(s) {}
    };

    int n;
    vector<int> parent, sz;
    stack<Change> history;

public:
    // Constructor: n components
    RollbackDSU(int _n) : n(_n), parent(_n), sz(_n, 1) {
        iota(parent.begin(), parent.end(), 0);  // parent[i] = i
    }

    // Find root (no path compression!)
    int find(int v) {
        while (v != parent[v])
            v = parent[v];
        return v;
    }

    // Union two sets by size + log change
    bool unite(int a, int b) {
        a = find(a);
        b = find(b);
        if (a == b) return false;  // already connected

        // Ensure a is larger
        if (sz[a] < sz[b]) swap(a, b);

        // === LOG CHANGE ===
        history.push(Change(b, parent[b], sz[a]));

        // === PERFORM UNION ===
        parent[b] = a;
        sz[a] += sz[b];
        return true;
    }

    // Undo last union
    void rollback() {
        if (history.empty()) return;

        auto c = history.top();
        history.pop();

        // Restore size of the new parent
        sz[parent[c.v]] = c.old_size;

        // Restore old parent of the child
        parent[c.v] = c.old_parent;
    }

    // --- Utility Functions ---
    int get_size(int v) { return sz[find(v)]; }
    bool same(int a, int b) { return find(a) == find(b); }
    int history_size() const { return history.size(); }

    // Clear all history (reset to initial state)
    void clear_history() {
        while (!history.empty()) history.pop();
    }
};

// === USAGE EXAMPLE ===
int main() {
    RollbackDSU dsu(6);

    cout << dsu.unite(0, 1) << endl;  // 1 (merged)
    cout << dsu.unite(2, 3) << endl;  // 1
    cout << dsu.same(0, 1) << endl;   // 1
    cout << dsu.same(0, 2) << endl;   // 0

    dsu.rollback();  // undo 2-3
    cout << dsu.same(2, 3) << endl;   // 0

    dsu.unite(4, 5);
    dsu.rollback();  // undo 4-5
    cout << dsu.same(4, 5) << endl;   // 0

    return 0;
}
```

## Full Implementation (C#)
```cpp
using System;
using System.Collections.Generic;

public class RollbackDSU
{
    private struct Change
    {
        public int v, old_parent, old_size;
        public Change(int v, int p, int s) { this.v = v; old_parent = p; old_size = s; }
    }

    private int[] parent, sz;
    private Stack<Change> history;

    public RollbackDSU(int n)
    {
        parent = new int[n];
        sz = new int[n];
        history = new Stack<Change>();
        for (int i = 0; i < n; i++)
        {
            parent[i] = i;
            sz[i] = 1;
        }
    }

    public int Find(int v)
    {
        while (v != parent[v])
            v = parent[v];
        return v;
    }

    public bool Unite(int a, int b)
    {
        a = Find(a);
        b = Find(b);
        if (a == b) return false;

        if (sz[a] < sz[b])
        {
            int tmp = a; a = b; b = tmp;
        }

        // Log change
        history.Push(new Change(b, parent[b], sz[a]));

        // Perform union
        parent[b] = a;
        sz[a] += sz[b];
        return true;
    }

    public void Rollback()
    {
        if (history.Count == 0) return;
        var c = history.Pop();
        sz[parent[c.v]] = c.old_size;
        parent[c.v] = c.old_parent;
    }

    // Utilities
    public int GetSize(int v) => sz[Find(v)];
    public bool Same(int a, int b) => Find(a) == Find(b);
    public int HistorySize => history.Count;
    public void ClearHistory() => history.Clear();
}

// === USAGE EXAMPLE ===
class Program
{
    static void Main()
    {
        var dsu = new RollbackDSU(6);

        Console.WriteLine(dsu.Unite(0, 1));  // True
        Console.WriteLine(dsu.Unite(2, 3));  // True
        Console.WriteLine(dsu.Same(0, 1));   // True
        Console.WriteLine(dsu.Same(0, 2));   // False

        dsu.Rollback();  // undo 2-3
        Console.WriteLine(dsu.Same(2, 3));   // False
    }
}
```

## Complexity Analysis  

| Operation | **Time Complexity** | **Notes** |
|----------|----------------------|---------|
| **Find** | **O(log n)** worst-case | **No path compression** → linear chain possible |
| **Union** | **O(log n)** amortized | **Union by size** → tree height ≤ log n |
| **Rollback** | **O(1)** | **Instant restore** from stack |
| **Space** | **O(n + q)** | `n` = nodes, `q` = number of `union` calls |

> **Amortized O(log n)** guaranteed by **union by size**  
> **Rollback is O(1)** — just pop & restore

---

## Pitfalls & Fixes  

| **Issue** | **Fix / Mitigation** |
|----------|-----------------------|
| **Path compression breaks rollback** | **Never use** `find` with path halving/compression |
| **Online queries** | **Not suitable** — only **offline** algorithms |
| **Stack overflow / memory blowup** | Only log **successful** `union` operations |
| **Wrong rollback order** | Always rollback **in LIFO** (last in, first out) |
| **Forgot to rollback** | Use **RAII**, **scope guards**, or **checkpoint system** |

---

## Conclusion  
**DSU with Rollback — the secret weapon for offline dynamic problems**:

- **O(log n) amortized** operations  
- **O(1) undo** — time travel in constant time  
- **Perfect for Divide & Conquer over time**  
- **Clean, reliable, fast** — no hidden slowdowns  

**Key takeaway:**  
> **When you need to go back in time — use Rollback DSU.**  
> **No path compression. Stack of changes. Zero regrets.**



---
