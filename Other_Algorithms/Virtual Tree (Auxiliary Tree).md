# Virtual Tree (Auxiliary Tree)

## Origin & Motivation

The **Virtual Tree** (also called **Auxiliary Tree** or **Key Vertex Tree**) is a technique for compressing a rooted tree to contain only a small subset of **key vertices** (query vertices) plus their pairwise LCAs, while preserving all ancestor-descendant relationships between them. The resulting compressed tree has at most `2k - 1` nodes for `k` key vertices.

The motivation: many tree DP problems involve queries over a subset of `k` vertices out of `n` total. Running the DP on the full tree costs O(n) per query. Building a virtual tree on the key vertices and running the DP on it costs O(k log n) per query — a massive speedup when k << n and there are many queries.

The technique was systematically formalized in Chinese competitive programming communities and is widely used in IOI/ICPC-style problems.

Complexity: **O(k log n)** to build the virtual tree (k = key vertices, log n for LCA), **O(k)** to run DP on it.

---

## Where It Is Used

- Tree DP on a subset of vertices with many queries
- Counting paths between marked vertices
- Minimum Steiner tree approximation on trees
- Distance queries between selected nodes
- Competitive programming: "given k marked vertices, compute some aggregate over all paths between them"
-染色 (coloring) problems on trees where only a few nodes change per query

---

## Key Concepts

**Key vertices:** The subset S of vertices of interest. |S| = k.

**Virtual tree of S:** The minimal subtree of the original tree that connects all vertices in S, compressed to contain only:
- All vertices in S
- All pairwise LCAs of consecutive vertices in S (when sorted by DFS order/Euler tour order)

**DFS order (in-order):** The Euler tour timestamp `tin[v]`. Sorting key vertices by `tin[v]` gives the order in which they appear in the DFS traversal.

**Property:** The virtual tree of k vertices has at most `2k - 1` nodes and `2k - 2` edges. In practice it is much smaller — only distinct LCAs are added.

---

## Construction Algorithm

### Preprocessing

Build an LCA structure (binary lifting or Euler tour + sparse table) on the original tree. Compute `tin[v]` (DFS entry time) for all vertices.

### Building the Virtual Tree

**Input:** Set S of k key vertices.

**Step 1:** Sort S by DFS order: `tin[u] < tin[v]`.

**Step 2:** Add pairwise LCAs of consecutive elements in sorted S. The set of all necessary nodes is:
```
S ∪ { lca(S[i], S[i+1]) | i = 0..k-2 }
```
(Plus the LCA of all key vertices = LCA of S[0] and S[k-1] = the root of the virtual tree.)

**Step 3:** Sort the combined set by DFS order, deduplicate.

**Step 4:** Build the virtual tree using a stack:
```
stack = [root_of_virtual_tree]
for each node v in sorted order (skip root if already in stack):
    l = lca(v, stack.top())
    if l != stack.top():
        // l is an ancestor of stack.top() but not equal:
        // pop nodes deeper than l, connecting them
        while stack has ≥ 2 elements and tin[stack[second]] >= tin[l]:
            add edge (stack.top(), stack.second) to virtual tree
            stack.pop()
        if stack.top() != l:
            add edge (stack.top(), l) to virtual tree
            stack.pop()
            stack.push(l)
    stack.push(v)

// Pop remaining stack
while stack has ≥ 2 elements:
    add edge (stack.top(), stack.second) to virtual tree
    stack.pop()
```

The stack maintains the current path from the virtual root down to the deepest node processed so far, in DFS order.

---

## Complexity Analysis

| Step | Time | Notes |
|---|---|---|
| Preprocessing (LCA + tin) | O(n log n) | Once per tree |
| Sort key vertices | O(k log k) | Per query |
| Compute LCAs of consecutive | O(k log n) | Per query |
| Build virtual tree (stack) | O(k) | Per query |
| DP on virtual tree | O(k) | Per query |
| Total per query | O(k log n) | Dominates |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

const int MAXN  = 200005;
const int LOG   = 18;
const ll  INF   = 1e18;

// ================================================================
// LCA PREPROCESSING
// ================================================================
int n;
vector<pair<int,ll>> adj[MAXN]; // {child, edge_weight}
int par[MAXN][LOG];
ll  dist[MAXN];   // distance from root
int dep[MAXN];    // depth
int tin[MAXN], tout[MAXN];
int timer_ = 0;

void dfs(int v, int p, ll d, int depth) {
    par[v][0] = p;
    dist[v]   = d;
    dep[v]    = depth;
    tin[v]    = timer_++;

    for (int i = 1; i < LOG; i++)
        par[v][i] = par[par[v][i-1]][i-1];

    for (auto [u, w] : adj[v])
        if (u != p) dfs(u, v, d + w, depth + 1);

    tout[v] = timer_++;
}

bool is_ancestor(int u, int v) {
    return tin[u] <= tin[v] && tout[v] <= tout[u];
}

int lca(int u, int v) {
    if (is_ancestor(u, v)) return u;
    if (is_ancestor(v, u)) return v;
    for (int i = LOG-1; i >= 0; i--)
        if (!is_ancestor(par[u][i], v)) u = par[u][i];
    return par[u][0];
}

// ================================================================
// VIRTUAL TREE CONSTRUCTION
// ================================================================
struct VirtualTree {
    // Virtual tree nodes and adjacency
    // adj_vt[v] = list of {child, weight_in_virtual_tree}
    // Weight = dist[child] - dist[parent] (compressed edge weight)
    map<int, vector<pair<int,ll>>> adj_vt;
    int root;

    void clear() { adj_vt.clear(); }

    void add_edge(int u, int v) {
        // u is parent of v in virtual tree
        ll w = dist[v] - dist[u]; // compressed edge weight
        adj_vt[u].push_back({v, w});
    }

    // Build virtual tree from key vertices (original vertex indices)
    // Returns root of virtual tree
    int build(vector<int>& keys) {
        if (keys.empty()) return -1;

        // Sort by DFS entry time
        sort(keys.begin(), keys.end(), [](int a, int b){ return tin[a] < tin[b]; });

        // Collect all necessary nodes: keys + pairwise LCAs of consecutive keys
        vector<int> nodes = keys;
        for (int i = 0; i + 1 < (int)keys.size(); i++)
            nodes.push_back(lca(keys[i], keys[i+1]));

        // Also add LCA of first and last (= virtual root)
        // Actually: lca of all keys = lca(keys[0], keys.back())
        // This is already included if we process consecutive LCAs correctly
        // Add it explicitly for safety:
        nodes.push_back(lca(keys[0], keys.back()));

        // Sort and deduplicate
        sort(nodes.begin(), nodes.end(), [](int a, int b){ return tin[a] < tin[b]; });
        nodes.erase(unique(nodes.begin(), nodes.end()), nodes.end());

        root = nodes[0]; // smallest tin = virtual root (= LCA of all)
        clear();

        // Stack-based virtual tree construction
        vector<int> stk;
        stk.push_back(nodes[0]);

        for (int i = 1; i < (int)nodes.size(); i++) {
            int v = nodes[i];
            int l = lca(v, stk.back());

            if (l != stk.back()) {
                // l is a proper ancestor of stk.back()
                // Pop nodes until we reach l or go above l
                while (stk.size() > 1 && dep[stk[stk.size()-2]] >= dep[l]) {
                    add_edge(stk[stk.size()-2], stk.back());
                    stk.pop_back();
                }
                if (stk.back() != l) {
                    add_edge(l, stk.back());
                    stk.pop_back();
                    stk.push_back(l);
                }
            }
            stk.push_back(v);
        }

        // Pop remaining stack
        while (stk.size() > 1) {
            add_edge(stk[stk.size()-2], stk.back());
            stk.pop_back();
        }

        return root;
    }
};

// ================================================================
// EXAMPLE DP ON VIRTUAL TREE:
// Problem: k key vertices marked. For each vertex in the virtual tree,
// compute: minimum distance to any key vertex in its subtree.
// ================================================================
VirtualTree vt;
set<int> key_set;

// dp[v] = minimum distance from v to any key vertex in v's subtree
// Returns dp[v]
ll dp_min_dist(int v, int parent) {
    ll result = key_set.count(v) ? 0 : INF; // v itself is a key?

    for (auto [child, w] : vt.adj_vt[v]) {
        if (child == parent) continue;
        ll child_dp = dp_min_dist(child, v);
        if (child_dp < INF) result = min(result, child_dp + w);
    }
    return result;
}

// ================================================================
// EXAMPLE DP: Count paths between key vertices passing through root
// This is a common pattern: compute answer as sum over virtual tree
// ================================================================

// dp2[v] = number of key vertices in v's virtual subtree
map<int,ll> dp2_memo;

ll dp_count_keys(int v) {
    ll cnt = key_set.count(v) ? 1 : 0;
    dp2_memo[v] = 0; // will accumulate

    for (auto [child, w] : vt.adj_vt[v]) {
        ll child_cnt = dp_count_keys(child);
        // Paths from v's subtree (excluding current child) to current child's subtree
        // Cross paths through v = dp2_memo[v] * child_cnt
        // Add to global answer externally
        dp2_memo[v] += child_cnt;
        cnt += child_cnt;
    }
    return cnt;
}

// ================================================================
// FULL EXAMPLE: Minimum spanning tree of k marked vertices on tree
// (Minimum virtual tree weight = sum of compressed edge weights)
// ================================================================
ll min_virtual_tree_weight(vector<int>& keys) {
    if (keys.size() <= 1) return 0;
    vt.build(keys);

    // Sum all edge weights in virtual tree
    ll total = 0;
    for (auto& [v, children] : vt.adj_vt)
        for (auto [child, w] : children)
            total += w;
    return total;
    // This equals the minimum Steiner tree weight on the original tree
    // connecting all key vertices (= sum of unique edges on all paths)
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Build a tree: 1-2-3-4-5 (path), with some extra edges
    //    1
    //   / \
    //  2   3
    // / \   \
    //4   5   6
    n = 6;
    // 0-indexed, root = 0
    auto add = [](int u, int v, ll w=1) {
        adj[u].push_back({v,w}); adj[v].push_back({u,w});
    };
    add(0,1,1); add(0,2,2);
    add(1,3,1); add(1,4,3);
    add(2,5,1);

    dfs(0, 0, 0, 0);

    // Query 1: virtual tree for keys {3, 4, 5}
    {
        printf("=== Virtual Tree for keys {3,4,5} ===\n");
        vector<int> keys = {3, 4, 5};
        key_set = {3, 4, 5};
        int root = vt.build(keys);
        printf("Virtual tree root: %d\n", root);
        printf("Virtual tree edges (parent -> child, weight):\n");
        for (auto& [v, children] : vt.adj_vt)
            for (auto [child, w] : children)
                printf("  %d -> %d  (w=%lld)\n", v, child, w);

        // Minimum virtual tree weight = min Steiner tree connecting {3,4,5}
        ll w = min_virtual_tree_weight(keys);
        printf("Steiner tree weight (sum of compressed edges): %lld\n", w);
        // Path 3-1-4 (weight 1+3=4) and 1-0-2-5 (weight 2+1=3 additional)
        // But Steiner = min subtree connecting {3,4,5}:
        // 3-1-4 (w=4) and 1-0-2-5 (w=3) -> total unique path weight = 4+3=7?
        // Actually: Steiner tree = smallest subtree spanning {3,4,5}
        // Path: 3-1-4 and 1-0-2-5: unique edges {3-1, 1-4, 1-0, 0-2, 2-5}
        // weights = 1+3+1+2+1 = 8
    }

    // Query 2: virtual tree for keys {3, 5}
    {
        printf("\n=== Virtual Tree for keys {3,5} ===\n");
        vector<int> keys = {3, 5};
        vt.build(keys);
        printf("Edges:\n");
        for (auto& [v, children] : vt.adj_vt)
            for (auto [child, w] : children)
                printf("  %d -> %d  (w=%lld)\n", v, child, w);
        printf("Steiner weight: %lld\n", min_virtual_tree_weight(keys));
        // Path 3->1->0->2->5, weights 1+1+2+1 = 5
    }

    // Query 3: minimum distance from root to closest key
    {
        printf("\n=== Min dist from each virtual node to nearest key {3,4} ===\n");
        vector<int> keys = {3, 4};
        key_set = {3, 4};
        vt.build(keys);
        ll d = dp_min_dist(vt.root, -1);
        printf("Min dist from virtual root %d to nearest key: %lld\n", vt.root, d);
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class VirtualTreeSolver {
    private const int LOG  = 18;
    private const long INF = long.MaxValue / 2;

    private readonly int n;
    private readonly List<(int v, long w)>[] adj;
    private readonly int[,] par;
    private readonly long[] dist;
    private readonly int[]  dep, tin, tout;
    private int timer_;

    // Virtual tree
    private readonly Dictionary<int, List<(int child, long w)>> vtAdj = new();
    public int VtRoot { get; private set; }

    public VirtualTreeSolver(int n) {
        this.n = n;
        adj  = new List<(int,long)>[n];
        par  = new int[n, LOG];
        dist = new long[n];
        dep  = new int[n];
        tin  = new int[n];
        tout = new int[n];
        for (int i = 0; i < n; i++) adj[i] = new();
    }

    public void AddEdge(int u, int v, long w = 1) {
        adj[u].Add((v, w)); adj[v].Add((u, w));
    }

    public void Build(int root = 0) {
        Dfs(root, root, 0, 0);
    }

    private void Dfs(int v, int p, long d, int depth) {
        par[v, 0] = p; dist[v] = d; dep[v] = depth;
        tin[v] = timer_++;
        for (int i = 1; i < LOG; i++)
            par[v, i] = par[par[v, i-1], i-1];
        foreach (var (u, w) in adj[v])
            if (u != p) Dfs(u, v, d + w, depth + 1);
        tout[v] = timer_++;
    }

    private bool IsAnc(int u, int v) =>
        tin[u] <= tin[v] && tout[v] <= tout[u];

    public int LCA(int u, int v) {
        if (IsAnc(u, v)) return u;
        if (IsAnc(v, u)) return v;
        for (int i = LOG-1; i >= 0; i--)
            if (!IsAnc(par[u, i], v)) u = par[u, i];
        return par[u, 0];
    }

    // Build virtual tree from key vertices
    public void BuildVT(List<int> keys) {
        vtAdj.Clear();
        if (keys.Count == 0) return;

        keys.Sort((a, b) => tin[a].CompareTo(tin[b]));

        var nodes = new List<int>(keys);
        for (int i = 0; i + 1 < keys.Count; i++)
            nodes.Add(LCA(keys[i], keys[i+1]));
        nodes.Add(LCA(keys[0], keys[^1]));

        nodes.Sort((a, b) => tin[a].CompareTo(tin[b]));
        // Deduplicate
        var dedup = new List<int> { nodes[0] };
        for (int i = 1; i < nodes.Count; i++)
            if (nodes[i] != nodes[i-1]) dedup.Add(nodes[i]);
        nodes = dedup;

        VtRoot = nodes[0];
        var stk = new List<int> { nodes[0] };

        void AddVtEdge(int u, int v) {
            long w = dist[v] - dist[u];
            if (!vtAdj.ContainsKey(u)) vtAdj[u] = new();
            vtAdj[u].Add((v, w));
        }

        for (int i = 1; i < nodes.Count; i++) {
            int v = nodes[i];
            int l = LCA(v, stk[^1]);

            if (l != stk[^1]) {
                while (stk.Count > 1 && dep[stk[^2]] >= dep[l]) {
                    AddVtEdge(stk[^2], stk[^1]);
                    stk.RemoveAt(stk.Count - 1);
                }
                if (stk[^1] != l) {
                    AddVtEdge(l, stk[^1]);
                    stk.RemoveAt(stk.Count - 1);
                    stk.Add(l);
                }
            }
            stk.Add(v);
        }

        while (stk.Count > 1) {
            AddVtEdge(stk[^2], stk[^1]);
            stk.RemoveAt(stk.Count - 1);
        }
    }

    // Sum of all virtual tree edge weights = Steiner tree weight
    public long SteinerWeight() {
        long total = 0;
        foreach (var (_, children) in vtAdj)
            foreach (var (_, w) in children)
                total += w;
        return total;
    }

    // DP: min distance from each node to nearest key vertex
    public long DpMinDist(int v, int parent, HashSet<int> keys) {
        long result = keys.Contains(v) ? 0 : INF;
        if (!vtAdj.ContainsKey(v)) return result;
        foreach (var (child, w) in vtAdj[v]) {
            if (child == parent) continue;
            long cd = DpMinDist(child, v, keys);
            if (cd < INF) result = Math.Min(result, cd + w);
        }
        return result;
    }

    public static void Main() {
        //    0
        //   / \
        //  1   2
        // / \   \
        //3   4   5
        var solver = new VirtualTreeSolver(6);
        solver.AddEdge(0,1,1); solver.AddEdge(0,2,2);
        solver.AddEdge(1,3,1); solver.AddEdge(1,4,3);
        solver.AddEdge(2,5,1);
        solver.Build(0);

        // Virtual tree for keys {3,4,5}
        var keys = new List<int>{3,4,5};
        solver.BuildVT(keys);
        Console.WriteLine($"VT root: {solver.VtRoot}");
        Console.WriteLine($"Steiner weight for {{3,4,5}}: {solver.SteinerWeight()}");

        // Virtual tree for keys {3,5}
        solver.BuildVT(new List<int>{3,5});
        Console.WriteLine($"Steiner weight for {{3,5}}: {solver.SteinerWeight()}");

        // Min dist from virtual root to nearest key in {3,4}
        var ks = new HashSet<int>{3,4};
        solver.BuildVT(new List<int>(ks));
        long d = solver.DpMinDist(solver.VtRoot, -1, ks);
        Console.WriteLine($"Min dist from VT root {solver.VtRoot} to nearest key: {d}");
    }
}
```

---

## Virtual Tree — Step by Step Example

```
Original tree (0-indexed, root=0):
      0
    /   \
   1     2
  / \     \
 3   4     5

Key vertices: {3, 4, 5}
DFS order (tin): 0<1<3<4<2<5

Step 1: Sort keys by tin: [3, 4, 5]

Step 2: Add consecutive LCAs:
  lca(3,4) = 1
  lca(4,5) = 0

Step 3: Combined nodes: {0,1,3,4,5}
Sorted by tin: [0, 1, 3, 4, 5]

Step 4: Stack construction:
  push 0:        stack=[0]
  process 1:     lca(1,0)=0=stack.top(), push 1.  stack=[0,1]
  process 3:     lca(3,1)=1=stack.top(), push 3.  stack=[0,1,3]
  process 4:     lca(4,3)=1 ≠ stack.top()=3:
                   pop 3: add edge 1->3.  stack=[0,1]
                   1==lca, push 4.        stack=[0,1,4]
  process 5:     lca(5,4)=0 ≠ stack.top()=4:
                   pop 4: add edge 1->4.  stack=[0,1]
                   dep[0]<dep[lca=0], pop 1: add edge 0->1. stack=[0]
                   stack.top()==0==lca, push 5. stack=[0,5]
  
  Remaining: pop 5: add edge 0->5.

Virtual tree edges: 0->1, 0->5, 1->3, 1->4
Virtual tree:
      0
    /   \
   1     5
  / \
 3   4
```

---

## Pitfalls

- **LCA of all keys = virtual root, must be in node set** — the root of the virtual tree is `lca(keys[0], keys.back())` after sorting by DFS order. If this LCA is not explicitly added to the node set, the stack construction may miss edges connecting to it, producing a disconnected virtual tree. Always add the global LCA explicitly (or rely on consecutive-pair LCAs to cover it — but add it for safety).
- **Stack comparison uses depth, not tin** — when deciding whether to pop the second-to-top element, compare `dep[stk[second]] >= dep[l]` (depth of l). Using `tin` for this comparison is wrong — a node with smaller `tin` may be deeper in the tree due to DFS ordering. Depth is the correct criterion for determining whether the LCA sits above or below a stack element.
- **Deduplication after merge** — after combining key vertices and their pairwise LCAs, sort and deduplicate the combined list. Without deduplication, a key vertex that is also an LCA appears twice, causing duplicate edges and inflated DP results.
- **Virtual tree edges go parent → child, not original edge direction** — the edge weight in the virtual tree is `dist[child] - dist[parent]` (sum of original edge weights on the compressed path), not the original edge weight. Using the original edge weight ignores compressed intermediate edges and gives wrong distances.
- **DP on virtual tree: iterate virtual tree adjacency, not original** — a common bug is accidentally traversing `adj[v]` (original tree neighbors) instead of `vtAdj[v]` (virtual tree children) during the DP phase. Since the virtual tree contains only a subset of nodes, this produces wrong DP values for non-key intermediate vertices.
- **Multiple queries: reset virtual tree state** — the virtual tree is rebuilt per query. If `vtAdj` or intermediate state from the previous query is not cleared, edges from previous queries pollute the current virtual tree. Call `vtAdj.clear()` at the start of each `BuildVT` call.

---

## Complexity Summary

| Phase | Time | Notes |
|---|---|---|
| Preprocessing (LCA + tin) | O(n log n) | Once |
| Sort keys per query | O(k log k) | Per query |
| Compute k-1 LCAs | O(k log n) | Per query |
| Stack construction | O(k) | Per query |
| DP on virtual tree | O(k) | Per query |
| Total per query | O(k log n) | vs O(n) on full tree |

---

## Conclusion

The Virtual Tree is the **standard technique for reducing tree DP from O(n) to O(k log n) per query** when only k out of n vertices participate:

- It compresses the original tree to a minimal subtree with at most 2k-1 nodes — all key vertices and the LCAs needed to connect them.
- Construction requires only sorting by DFS order and a single linear-time stack pass after LCA queries.
- Any tree DP that was O(n) on the full tree becomes O(k) on the virtual tree, with O(k log n) dominated by the LCA queries.
- Common applications: minimum Steiner tree on trees, counting/summing paths between marked vertices, distance queries in competitive programming.

**Key takeaway:**  
The virtual tree is built in three steps: sort keys by DFS order, compute k-1 consecutive LCAs, then run a stack-based edge assignment. The critical invariant is that the stack always holds the current DFS-order path from the virtual root to the deepest processed node. Every push adds one node; every pop adds one virtual tree edge. The total cost is O(k log n) per query regardless of n.
