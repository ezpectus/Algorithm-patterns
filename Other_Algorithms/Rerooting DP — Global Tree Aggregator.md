# Rerooting DP — Global Tree Aggregator

---

## Origin & Motivation

**Rerooting DP** is a powerful technique for solving **tree problems** where the **answer depends on the root**.  
Instead of **recomputing from scratch** for every root, rerooting **propagates local DP values** across the tree, allowing **efficient global aggregation**.

It’s especially useful when:

* You want to **compute answers for every node as root**  
* The problem is **root-sensitive** (subtree sizes, distances, contributions)  
* You need **linear time** across all reroots  

---

## Where It’s Used

| Domain | Use Case |
|--------|----------|
| Tree DP | Subtree metrics, distances, contributions |
| Graph Analytics | Centrality, influence propagation |
| Competitive Programming | Reroot-sensitive scoring |
| Bioinformatics | Phylogenetic tree rerooting |
| Game Trees | Rerooted evaluations |

---

## When to Use Rerooting vs Alternatives

| Scenario | Rerooting | DFS DP | BFS | Heavy-Light |
|--------|-----------|--------|-----|-------------|
| Need answer for every root | Yes | No | No | No |
| Subtree aggregation | Yes | Yes | No | No |
| Path queries | No | No | Yes | Yes |
| Root-sensitive scoring | Yes | No | No | No |
| Dynamic rerooting | Yes | No | No | No |

---

## Core Idea

1. **Post-order DP**: Compute DP values for each **subtree rooted at node `u`**  
2. **Rerooting pass**: Propagate **parent contributions** to children, adjusting DP values as if each child were the **new root**  
3. Use **edge-based transfer functions** to move DP values across the tree  

This allows computing the **answer for every node as root** in **O(n)** time.

---

## Implementation (C++)

```cpp
struct Edge {
    int to;
};

int n;
vector<vector<Edge>> adj;
vector<int> dp, res;

int merge(int a, int b) {
    return a + b; // example: sum of subtree sizes
}

int add_root(int val) {
    return val + 1; // include current node
}

void dfs1(int u, int p) {
    dp[u] = 0;
    for (auto& e : adj[u]) {
        int v = e.to;
        if (v == p) continue;
        dfs1(v, u);
        dp[u] = merge(dp[u], add_root(dp[v]));
    }
}

void dfs2(int u, int p, int up) {
    vector<int> prefix, suffix;
    for (auto& e : adj[u]) {
        int v = e.to;
        if (v == p) continue;
        prefix.push_back(add_root(dp[v]));
        suffix.push_back(add_root(dp[v]));
    }

    for (int i = 1; i < prefix.size(); ++i)
        prefix[i] = merge(prefix[i], prefix[i - 1]);
    for (int i = suffix.size() - 2; i >= 0; --i)
        suffix[i] = merge(suffix[i], suffix[i + 1]);

    int child = 0;
    for (auto& e : adj[u]) {
        int v = e.to;
        if (v == p) continue;
        int left = (child == 0) ? 0 : prefix[child - 1];
        int right = (child == suffix.size() - 1) ? 0 : suffix[child + 1];
        int combined = merge(left, right);
        dfs2(v, u, add_root(merge(up, combined)));
        ++child;
    }

    res[u] = add_root(merge(dp[u], up));
}
```

## Implementation (C#)

```csharp
public class Rerooting {
    public class Edge { public int To; }

    int n;
    List<Edge>[] adj;
    int[] dp, res;

    int Merge(int a, int b) => a + b;
    int AddRoot(int val) => val + 1;

    void Dfs1(int u, int p) {
        dp[u] = 0;
        foreach (var e in adj[u]) {
            int v = e.To;
            if (v == p) continue;
            Dfs1(v, u);
            dp[u] = Merge(dp[u], AddRoot(dp[v]));
        }
    }

    void Dfs2(int u, int p, int up) {
        var prefix = new List<int>();
        var suffix = new List<int>();

        foreach (var e in adj[u]) {
            int v = e.To;
            if (v == p) continue;
            int val = AddRoot(dp[v]);
            prefix.Add(val);
            suffix.Add(val);
        }

        for (int i = 1; i < prefix.Count; i++)
            prefix[i] = Merge(prefix[i], prefix[i - 1]);
        for (int i = suffix.Count - 2; i >= 0; i--)
            suffix[i] = Merge(suffix[i], suffix[i + 1]);

        int child = 0;
        foreach (var e in adj[u]) {
            int v = e.To;
            if (v == p) continue;
            int left = (child == 0) ? 0 : prefix[child - 1];
            int right = (child == suffix.Count - 1) ? 0 : suffix[child + 1];
            int combined = Merge(left, right);
            Dfs2(v, u, AddRoot(Merge(up, combined)));
            child++;
        }

        res[u] = AddRoot(Merge(dp[u], up));
    }
}
```

## Complexity Analysis

| Phase         | Complexity |
|---------------|------------|
| **First DFS** | O(n)       |
| **Second DFS**| O(n)       |
| **Space**     | O(n)       |

> **Linear time across all reroots**  
> Efficient for trees with up to **10⁵ nodes**

---

## Why It’s Powerful

* **Computes global answers** from **local DP**  
* **Avoids recomputation** for each root  
* **Modular**: works with any **associative merge + root function**  
* **Elegant propagation** via **prefix/suffix trick**

---

## Impact of Design Choices

| Choice               | Effect |
|----------------------|--------|
| **Merge function**   | Defines how DP values combine |
| **AddRoot function** | Adds root contribution |
| **Prefix/suffix trick** | Enables fast reroot propagation |
| **Edge-based reroot** | Generalizes to weighted trees |

---

## Pitfalls

* **Merge must be associative**  
* **AddRoot must be consistent**  
* **Edge direction matters** — avoid double counting  
* **Debugging reroot logic** can be tricky  

---

## Conclusion

Rerooting DP is a **global tree aggregator**:

* Computes **answers for every root** in **linear time**  
* Turns **local subtree DP** into **full-tree insights**  
* **Modular and reusable** across tree problems  
* Ideal for **scoring, distances, and reroot-sensitive metrics**  

> **Key takeaway**:  
> **Rerooting DP transforms subtree logic into global tree answers — elegantly, efficiently, and scalably.**

---


