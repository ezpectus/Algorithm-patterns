# Biconnected Components

## Origin & Motivation

A **biconnected component (BCC)** of an undirected graph is a maximal biconnected subgraph — a subgraph where any two vertices are connected by at least two vertex-disjoint paths. Equivalently, no single vertex removal (articulation point) disconnects the component.

Every edge belongs to exactly one BCC. Articulation points are shared between multiple BCCs — they are the "join points" between components. The **block-cut tree** (or BC-tree) represents this decomposition: alternating nodes for BCCs and articulation points, forming a tree.

The algorithm is a single DFS in **O(V + E)** using a stack of edges and low-link values — a direct extension of the bridge/AP detection algorithm.

Complexity: **O(V + E)** time, **O(V + E)** space.

---

## Where It Is Used

- Graph reliability: finding all 2-connected subgraphs
- Block-cut tree construction for tree-based queries on general graphs
- Planarity testing (each BCC tested independently)
- Competitive programming: queries that reduce to tree problems via block-cut tree
- Network design: identifying redundant connectivity regions
- Ear decomposition of biconnected graphs

---

## Key Definitions

**Biconnected graph:** A connected graph with no articulation point (equivalently, any two edges lie on a common simple cycle).

**Biconnected component:** A maximal biconnected subgraph. Note:
- A single edge (bridge) forms its own BCC of size 2 (just that edge)
- All edges in a simple cycle form one BCC
- Articulation points belong to multiple BCCs; all other vertices belong to exactly one BCC

**Block-cut tree:** A tree where:
- Each BCC is a "block" node
- Each articulation point is a "cut" node
- Edges connect each cut node to the blocks it belongs to

---

## Algorithm

Single DFS with edge stack. When we detect that vertex `v` is an articulation point root of a BCC (either root with 2+ children, or `low[child] >= disc[v]`), pop all edges from the stack down to and including the edge `(v, child)` — these form one BCC.

```
dfs(v, parent_edge):
    disc[v] = low[v] = timer++
    children = 0

    for each (u, eid) in adj[v]:
        if eid == parent_edge: continue

        if disc[u] == -1:
            children++
            stack.push((v, u))
            dfs(u, eid)
            low[v] = min(low[v], low[u])

            // BCC root condition
            if (parent_edge == -1 and children > 1) or
               (parent_edge != -1 and low[u] >= disc[v]):
                pop edges from stack until (v,u) into new BCC

        elif disc[u] < disc[v]:   // back edge (push once, going upward)
            stack.push((v, u))
            low[v] = min(low[v], disc[u])

// After DFS: pop remaining stack as one more BCC
```

---

## Block-Cut Tree

After finding all BCCs, build the block-cut tree:

```
For each BCC b:
    For each articulation point v in BCC b:
        add edge (block_node[b], cut_node[v]) to tree

Non-articulation vertices belong to exactly one BCC:
    add them as leaves of their BCC's block node
```

The block-cut tree has `|BCCs| + |APs|` nodes and is always a tree (or forest for disconnected graphs).

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| DFS + low-link | O(V + E) | O(V) |
| BCC extraction (edge stack) | O(E) total | O(E) |
| Block-cut tree construction | O(V + E) | O(V + E) |
| Total | O(V + E) | O(V + E) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct BiconnectedComponents {
    int n;
    vector<vector<pair<int,int>>> g; // {neighbor, edge_id}
    int num_edges;

    vector<int>  disc, low;
    int timer_;

    // Each BCC = set of vertex indices in that component
    vector<vector<int>> bccs;       // vertex sets
    vector<vector<pair<int,int>>> bcc_edges; // edge sets per BCC
    vector<int> comp;               // comp[v] = BCC id for non-AP vertices
    vector<bool> is_ap;

    stack<pair<int,int>> stk; // edge stack

    BiconnectedComponents(int n)
        : n(n), g(n), num_edges(0),
          disc(n,-1), low(n,0), timer_(0),
          comp(n,-1), is_ap(n,false) {}

    void add_edge(int u, int v) {
        int id = num_edges++;
        g[u].push_back({v, id});
        g[v].push_back({u, id});
    }

    void pop_bcc(int u, int v) {
        // Pop edges from stack until edge (u,v) inclusive
        vector<pair<int,int>> edges;
        while (stk.top() != make_pair(u,v) &&
               stk.top() != make_pair(v,u)) {
            edges.push_back(stk.top());
            stk.pop();
        }
        edges.push_back(stk.top());
        stk.pop();
        bcc_edges.push_back(edges);

        // Collect vertices
        set<int> verts;
        for (auto [a,b] : edges) { verts.insert(a); verts.insert(b); }
        bccs.push_back(vector<int>(verts.begin(), verts.end()));
    }

    void dfs(int v, int parent_edge) {
        disc[v] = low[v] = timer_++;
        int children = 0;

        for (auto [u, eid] : g[v]) {
            if (eid == parent_edge) continue;

            if (disc[u] == -1) {
                children++;
                stk.push({v, u});
                dfs(u, eid);
                low[v] = min(low[v], low[u]);

                bool is_root    = (parent_edge == -1);
                bool ap_nonroot = (!is_root && low[u] >= disc[v]);
                bool ap_root    = (is_root && children > 1);

                if (ap_nonroot || ap_root) {
                    is_ap[v] = true;
                    pop_bcc(v, u);
                }
            } else if (disc[u] < disc[v]) {
                // Back edge: push only in one direction (upward)
                stk.push({v, u});
                low[v] = min(low[v], disc[u]);
            }
        }
    }

    void solve() {
        for (int v = 0; v < n; v++) {
            if (disc[v] != -1) continue;
            dfs(v, -1);
            // Remaining edges on stack = one final BCC
            if (!stk.empty()) {
                vector<pair<int,int>> edges;
                while (!stk.empty()) {
                    edges.push_back(stk.top());
                    stk.pop();
                }
                bcc_edges.push_back(edges);
                set<int> verts;
                for (auto [a,b] : edges) { verts.insert(a); verts.insert(b); }
                bccs.push_back(vector<int>(verts.begin(), verts.end()));
            }
        }
    }

    // ================================================================
    // BLOCK-CUT TREE
    // Node layout:
    //   nodes 0..n-1          = original vertices (cut nodes if AP)
    //   nodes n..n+bccs-1     = BCC block nodes
    // Returns adjacency list of block-cut tree
    // ================================================================
    vector<vector<int>> block_cut_tree() {
        int total = n + (int)bccs.size();
        vector<vector<int>> tree(total);

        for (int b = 0; b < (int)bccs.size(); b++) {
            int block_node = n + b;
            for (int v : bccs[b]) {
                // Connect block to vertex if v is AP or has degree in this BCC
                // Every vertex in a BCC connects to that BCC's block node
                tree[block_node].push_back(v);
                tree[v].push_back(block_node);
            }
        }

        // Deduplicate edges (a non-AP vertex appears in exactly one BCC,
        // but APs appear in multiple — duplicates occur)
        for (int i = 0; i < total; i++) {
            sort(tree[i].begin(), tree[i].end());
            tree[i].erase(unique(tree[i].begin(), tree[i].end()), tree[i].end());
        }
        return tree;
    }
};

// ================================================================
// QUERY: Given block-cut tree, answer:
// "Is the path from u to v in the original graph forced to pass
//  through articulation point w?"
// Equivalent to: is w on the path from u to v in the block-cut tree?
// ================================================================

// LCA on block-cut tree for path queries
struct LCA {
    int n, LOG;
    vector<int> depth;
    vector<vector<int>> up;

    LCA(const vector<vector<int>>& tree, int root = 0)
        : n(tree.size()), LOG(20),
          depth(tree.size(), 0), up(tree.size(), vector<int>(20, -1))
    {
        // BFS to set depth and up[v][0]
        vector<bool> visited(n, false);
        queue<int> q;
        q.push(root); visited[root] = true;
        while (!q.empty()) {
            int v = q.front(); q.pop();
            for (int u : tree[v]) {
                if (!visited[u]) {
                    visited[u] = true;
                    depth[u] = depth[v] + 1;
                    up[u][0] = v;
                    q.push(u);
                }
            }
        }
        up[root][0] = root;
        for (int j = 1; j < LOG; j++)
            for (int v = 0; v < n; v++)
                up[v][j] = up[up[v][j-1]][j-1];
    }

    int lca(int u, int v) {
        if (depth[u] < depth[v]) swap(u, v);
        int diff = depth[u] - depth[v];
        for (int j = 0; j < LOG; j++)
            if ((diff >> j) & 1) u = up[u][j];
        if (u == v) return u;
        for (int j = LOG-1; j >= 0; j--)
            if (up[u][j] != up[v][j]) { u = up[u][j]; v = up[v][j]; }
        return up[u][0];
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // Graph: 0-1-2-0 (triangle) + bridge 2-3 + 3-4-5-3 (triangle)
    {
        BiconnectedComponents bcc(6);
        bcc.add_edge(0,1); bcc.add_edge(1,2); bcc.add_edge(2,0);
        bcc.add_edge(2,3);
        bcc.add_edge(3,4); bcc.add_edge(4,5); bcc.add_edge(5,3);
        bcc.solve();

        printf("=== Biconnected Components ===\n");
        printf("Total BCCs: %d\n", (int)bcc.bccs.size());
        for (int i = 0; i < (int)bcc.bccs.size(); i++) {
            printf("  BCC %d vertices: ", i);
            for (int v : bcc.bccs[i]) printf("%d ", v);
            printf("| edges: ");
            for (auto [u,v] : bcc.bcc_edges[i]) printf("(%d,%d) ", u,v);
            printf("\n");
        }

        printf("Articulation points: ");
        for (int v = 0; v < 6; v++) if (bcc.is_ap[v]) printf("%d ", v);
        printf("\n");

        // Block-cut tree
        auto tree = bcc.block_cut_tree();
        printf("\nBlock-cut tree (nodes 0-5 = vertices, 6+ = BCCs):\n");
        for (int i = 0; i < (int)tree.size(); i++) {
            if (tree[i].empty()) continue;
            printf("  node %d -> ", i);
            for (int u : tree[i]) printf("%d ", u);
            printf("\n");
        }
    }

    // More complex example: two 4-cycles sharing an AP
    {
        printf("\n=== Two cycles sharing AP ===\n");
        // 0-1-2-3-0 (cycle) + 3-4-5-6-3 (cycle), vertex 3 is AP
        BiconnectedComponents bcc(7);
        bcc.add_edge(0,1); bcc.add_edge(1,2); bcc.add_edge(2,3); bcc.add_edge(3,0);
        bcc.add_edge(3,4); bcc.add_edge(4,5); bcc.add_edge(5,6); bcc.add_edge(6,3);
        bcc.solve();

        printf("BCCs: %d\n", (int)bcc.bccs.size());
        for (int i = 0; i < (int)bcc.bccs.size(); i++) {
            printf("  BCC %d: vertices {", i);
            for (int v : bcc.bccs[i]) printf("%d,", v);
            printf("}\n");
        }
        printf("APs: ");
        for (int v = 0; v < 7; v++) if (bcc.is_ap[v]) printf("%d ", v);
        printf("\n"); // Expected: AP = {3}

        // Is vertex 3 on every path from {0,1,2} to {4,5,6}?
        // Yes — it's the only AP connecting the two BCCs.
        // Verified via block-cut tree: tree path from BCC0 to BCC1 goes through node 3.
    }

    // Graph with no AP (fully biconnected)
    {
        printf("\n=== Fully biconnected graph ===\n");
        BiconnectedComponents bcc(4);
        bcc.add_edge(0,1); bcc.add_edge(1,2); bcc.add_edge(2,3); bcc.add_edge(3,0);
        bcc.add_edge(0,2); // extra diagonal
        bcc.solve();
        printf("BCCs: %d (expect 1)\n", (int)bcc.bccs.size());
        printf("APs: ");
        bool any = false;
        for (int v = 0; v < 4; v++) if (bcc.is_ap[v]) { printf("%d ", v); any=true; }
        if (!any) printf("none");
        printf("\n");
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class BiconnectedComponents {
    private readonly int n;
    private readonly List<(int v, int id)>[] g;
    private int numEdges;

    private readonly int[]  disc, low;
    public  readonly bool[] IsAp;
    private int timer_;

    public List<List<int>>           BCCVertices { get; } = new();
    public List<List<(int,int)>>     BCCEdges    { get; } = new();

    private readonly Stack<(int,int)> stk = new();

    public BiconnectedComponents(int n) {
        this.n = n;
        g   = new List<(int,int)>[n];
        for (int i = 0; i < n; i++) g[i] = new();
        disc = new int[n]; Array.Fill(disc, -1);
        low  = new int[n];
        IsAp = new bool[n];
    }

    public void AddEdge(int u, int v) {
        int id = numEdges++;
        g[u].Add((v, id)); g[v].Add((u, id));
    }

    private void PopBCC(int u, int v) {
        var edges = new List<(int,int)>();
        while (true) {
            var top = stk.Peek();
            stk.Pop();
            edges.Add(top);
            if ((top.Item1==u && top.Item2==v) ||
                (top.Item1==v && top.Item2==u)) break;
        }
        BCCEdges.Add(edges);
        var verts = new HashSet<int>();
        foreach (var (a,b) in edges) { verts.Add(a); verts.Add(b); }
        BCCVertices.Add(new List<int>(verts));
    }

    private void Dfs(int v, int pe) {
        disc[v] = low[v] = timer_++;
        int children = 0;

        foreach (var (u, eid) in g[v]) {
            if (eid == pe) continue;

            if (disc[u] == -1) {
                children++;
                stk.Push((v, u));
                Dfs(u, eid);
                low[v] = Math.Min(low[v], low[u]);

                bool isRoot    = pe == -1;
                bool apNonRoot = !isRoot && low[u] >= disc[v];
                bool apRoot    = isRoot  && children > 1;

                if (apNonRoot || apRoot) {
                    IsAp[v] = true;
                    PopBCC(v, u);
                }
            } else if (disc[u] < disc[v]) {
                stk.Push((v, u));
                low[v] = Math.Min(low[v], disc[u]);
            }
        }
    }

    public void Solve() {
        for (int v = 0; v < n; v++) {
            if (disc[v] != -1) continue;
            Dfs(v, -1);
            if (stk.Count > 0) {
                var edges = new List<(int,int)>();
                while (stk.Count > 0) edges.Add(stk.Pop());
                BCCEdges.Add(edges);
                var verts = new HashSet<int>();
                foreach (var (a,b) in edges) { verts.Add(a); verts.Add(b); }
                BCCVertices.Add(new List<int>(verts));
            }
        }
    }

    // Block-cut tree: nodes 0..n-1 = vertices, n..n+BCCs-1 = block nodes
    public List<int>[] BlockCutTree() {
        int total = n + BCCVertices.Count;
        var tree  = new List<int>[total];
        for (int i = 0; i < total; i++) tree[i] = new();

        for (int b = 0; b < BCCVertices.Count; b++) {
            int bn = n + b;
            foreach (int v in BCCVertices[b]) {
                tree[bn].Add(v);
                tree[v].Add(bn);
            }
        }

        // Deduplicate
        for (int i = 0; i < total; i++) {
            tree[i].Sort();
            var dedup = new List<int>();
            for (int j = 0; j < tree[i].Count; j++)
                if (j == 0 || tree[i][j] != tree[i][j-1])
                    dedup.Add(tree[i][j]);
            tree[i] = dedup;
        }
        return tree;
    }

    public static void Main() {
        // Triangle + bridge + triangle
        var bcc = new BiconnectedComponents(6);
        bcc.AddEdge(0,1); bcc.AddEdge(1,2); bcc.AddEdge(2,0);
        bcc.AddEdge(2,3);
        bcc.AddEdge(3,4); bcc.AddEdge(4,5); bcc.AddEdge(5,3);
        bcc.Solve();

        Console.WriteLine($"BCCs: {bcc.BCCVertices.Count}");
        for (int i = 0; i < bcc.BCCVertices.Count; i++) {
            Console.Write($"  BCC {i} vertices: ");
            Console.WriteLine(string.Join(" ", bcc.BCCVertices[i]));
        }
        Console.Write("APs: ");
        for (int v = 0; v < 6; v++) if (bcc.IsAp[v]) Console.Write($"{v} ");
        Console.WriteLine();

        var tree = bcc.BlockCutTree();
        Console.WriteLine("Block-cut tree:");
        for (int i = 0; i < tree.Length; i++) {
            if (tree[i].Count == 0) continue;
            string label = i < 6 ? $"v{i}" : $"BCC{i-6}";
            Console.WriteLine($"  {label} -> {string.Join(" ", tree[i].ConvertAll(x => x < 6 ? $"v{x}" : $"BCC{x-6}"))}");
        }
    }
}
```

---

## Block-Cut Tree — Structure

```
Graph: 0-1-2-0 (triangle) -- bridge(2-3) -- 3-4-5-3 (triangle)

BCCs:
  BCC0 = {0,1,2}    (triangle, edges: 0-1, 1-2, 2-0)
  BCC1 = {2,3}      (bridge edge 2-3)
  BCC2 = {3,4,5}    (triangle, edges: 3-4, 4-5, 5-3)

Articulation points: {2, 3}

Block-cut tree:
       [BCC0]
         |
        v2          ← articulation point
         |
       [BCC1]
         |
        v3          ← articulation point
         |
       [BCC2]

Non-AP vertices 0,1 → leaf children of BCC0 block node
Non-AP vertices 4,5 → leaf children of BCC2 block node
```

---

## Properties of BCCs

- Every edge belongs to **exactly one** BCC
- Every non-AP vertex belongs to **exactly one** BCC
- Every articulation point belongs to **two or more** BCCs
- A BCC with only one edge is a bridge (and that edge is its own BCC)
- A BCC with 2+ edges has all edges on a common simple cycle (2-edge-connected internally)
- The number of BCCs = number of edges in the block-cut tree = |APs| + |BCCs| - 1 (tree property)

---

## Pitfalls

- **Push back edge only once** — in undirected DFS, edge `(v, u)` is seen from both directions. Push to the edge stack only when `disc[u] < disc[v]` (going upward / back edge direction), not when `disc[u] > disc[v]` (the forward direction of the same edge). Pushing twice inflates the BCC edge sets.
- **Pop condition uses `>=` not `>`** — the BCC root condition for non-root vertices is `low[u] >= disc[v]` (non-strict). Using strict `>` (the bridge condition) only pops BCCs at bridges, missing BCCs rooted at articulation points where `low[u] == disc[v]`.
- **Root vertex: pop after each extra child** — for the DFS root with k children, there are k BCCs. Pop after each child beyond the first (when `children > 1` the condition fires). A common mistake is popping only once at the root, producing one large incorrect BCC instead of k separate ones.
- **Final stack pop after DFS** — after `dfs(root)` returns, the stack may contain edges of one remaining BCC (the one containing the root and its first child). Always check and pop remaining stack contents as the last BCC. Forgetting this loses the root's primary BCC.
- **Block-cut tree has duplicate edges to AP vertices** — an AP vertex `v` belongs to `k` BCCs. The block-cut tree correctly has `k` edges incident to `v`. The deduplication step in the implementation above removes duplicate edges between a vertex and a block node — but these should not exist if the vertex is added to each BCC's vertex set exactly once. Verify vertex sets per BCC are deduplicated before building the tree.
- **Single-vertex components** — isolated vertices (degree 0) have no edges and form no BCC. They must be handled separately if the problem requires tracking all vertices. In the standard edge-based BCC algorithm, isolated vertices never appear in any BCC.

---

## Conclusion

Biconnected components decompose an undirected graph into its **maximally 2-vertex-connected pieces**:

- Every edge belongs to exactly one BCC, and the block-cut tree captures the full structure of how BCCs connect through articulation points.
- The single DFS algorithm finds all BCCs in O(V+E) using an edge stack — direct extension of the articulation point detection algorithm.
- The block-cut tree reduces many graph problems (path queries, connectivity under vertex deletion) to tree problems, enabling O(log n) or O(1) queries with LCA preprocessing.
- Key distinction from bridges/2-edge-connected components: BCCs are **vertex** based (removing any single non-AP vertex keeps the BCC connected), while 2-edge-connected components are **edge** based (no bridge inside them).

**Key takeaway:**  
The implementation has three critical details: push edges (not vertices) onto the stack, use `low[u] >= disc[v]` (non-strict) for the pop condition, and always pop the remaining stack after each connected component's root DFS completes. Everything else — low-link computation, parent-edge tracking, root special case — is identical to bridge/AP detection.
