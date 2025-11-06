# DSU on Tree — Subtree Query Engine

---

## Origin & Motivation

**DSU on Tree** (also known as **Small-to-Large**, or **Sack Technique**) is a powerful optimization for **answering queries on subtrees** of a rooted tree.  
Popularized in **competitive programming** (2013–2015) by Russian coders, it uses **DSU-like merging**: always merge **smaller sets into larger ones** to keep operations efficient.

It enables **offline subtree queries** in **linear or near-linear time**, where naive DFS would be too slow.

---

## Where It’s Used

| Domain | Use Case |
|--------|----------|
| Tree Queries | Frequency, color count, label statistics |
| Competitive Programming | Codeforces, AtCoder, CSES |
| Subtree Analytics | Histograms, counters, majority elements |
| Game Trees | Local state aggregation |
| Offline Query Processing | Preprocessing-heavy problems |

---

## When to Use DSU on Tree vs Alternatives

| Scenario | DSU on Tree | Segment Tree | HLD | Euler Tour |
|--------|-------------|--------------|-----|------------|
| Subtree queries | Yes | No | No | Yes |
| Path queries | No | Yes | Yes | Yes |
| Online updates | No | Yes | Yes | Yes |
| Offline frequency/count queries | Yes | No | No | Yes |
| Heavy subtree merging | Yes | No | No | No |

---

## Core Idea

1. **DFS** to compute **subtree sizes**  
2. For each node:  
   * Process **light children** first (small subtrees)  
   * Process **heavy child** last (largest subtree)  
3. Use a **global counter/map** to track frequencies  
4. **Merge small subtree data into large one**  
5. **Answer queries** after processing each node  

This ensures each element is **added/removed at most O(log n) times** → total **O(n log n)**

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

const int N = 100005;
vector<int> adj[N], color(N), freq(N), answer(N);
int sz[N], cnt[N];
bool big[N];

void dfsSize(int u, int p) {
    sz[u] = 1;
    for (int v : adj[u])
        if (v != p) {
            dfsSize(v, u);
            sz[u] += sz[v];
        }
}

void add(int u, int p, int delta) {
    cnt[color[u]] += delta;
    for (int v : adj[u])
        if (v != p && !big[v])
            add(v, u, delta);
}

void dfs(int u, int p, bool keep) {
    int mx = -1, bigChild = -1;
    for (int v : adj[u])
        if (v != p && sz[v] > mx) {
            mx = sz[v];
            bigChild = v;
        }

    for (int v : adj[u])
        if (v != p && v != bigChild)
            dfs(v, u, false);

    if (bigChild != -1) {
        dfs(bigChild, u, true);
        big[bigChild] = true;
    }

    add(u, p, +1);
    answer[u] = cnt[color[u]]; // example: count of own color in subtree

    if (bigChild != -1)
        big[bigChild] = false;
    if (!keep)
        add(u, p, -1);
}

// Usage:
// dfsSize(1, -1);
// dfs(1, -1, true);
```

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class DSUonTree {
    const int N = 100005;
    List<int>[] adj = new List<int>[N];
    int[] color = new int[N], answer = new int[N], sz = new int[N], cnt = new int[N];
    bool[] big = new bool[N];

    public DSUonTree() {
        for (int i = 0; i < N; i++) adj[i] = new List<int>();
    }

    public void AddEdge(int u, int v) {
        adj[u].Add(v);
        adj[v].Add(u);
    }

    void DfsSize(int u, int p) {
        sz[u] = 1;
        foreach (int v in adj[u])
            if (v != p) {
                DfsSize(v, u);
                sz[u] += sz[v];
            }
    }

    void Add(int u, int p, int delta) {
        cnt[color[u]] += delta;
        foreach (int v in adj[u])
            if (v != p && !big[v])
                Add(v, u, delta);
    }

    void Dfs(int u, int p, bool keep) {
        int mx = -1, bigChild = -1;
        foreach (int v in adj[u])
            if (v != p && sz[v] > mx) {
                mx = sz[v];
                bigChild = v;
            }

        foreach (int v in adj[u])
            if (v != p && v != bigChild)
                Dfs(v, u, false);

        if (bigChild != -1) {
            Dfs(bigChild, u, true);
            big[bigChild] = true;
        }

        Add(u, p, +1);
        answer[u] = cnt[color[u]];

        if (bigChild != -1)
            big[bigChild] = false;
        if (!keep)
            Add(u, p, -1);
    }

    public int[] Solve(int root, int[] colors) {
        Array.Copy(colors, color, colors.Length);
        DfsSize(root, -1);
        Dfs(root, -1, true);
        return answer;
    }
}
```

## Complexity Analysis

| Phase                   | Complexity                     |
|-------------------------|--------------------------------|
| **Subtree size DFS**    | O(n)                           |
| **Main DFS**            | O(n log n)                     |
| **Space**               | O(n + C) — where C is color domain size |

---

## Pitfalls

* Requires **offline query model**  
* Must **carefully manage add/remove logic**  
* **Heavy child must be processed last**  
* **Global counters must be reset** unless `keep == true`  
* **Not suitable** for path queries or dynamic updates  

---

## Conclusion

**DSU on Tree** is the **subtree query engine**:

* Efficient for **subtree frequency/count queries**  
* Merges **small-to-large** for **amortized efficiency**  
* Ideal for **offline color/histogram problems**  
* **Modular and reusable** across tree DP problems  

> **Key takeaway**:  
> **DSU on Tree transforms naive subtree queries into scalable, elegant solutions** — perfect for **frequency-based analytics on rooted trees**.

---



