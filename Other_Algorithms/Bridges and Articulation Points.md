# Bridges and Articulation Points

## Origin & Motivation

Bridges and articulation points are the **critical connectivity elements** of an undirected graph. Removing them disconnects the graph or increases the number of connected components.

- **Bridge** (cut edge): an edge `(u, v)` whose removal increases the number of connected components.
- **Articulation point** (cut vertex): a vertex `v` whose removal (along with all its edges) increases the number of connected components.

Both are found by a single DFS in **O(V + E)** time using **low-link values** — the same technique underlying Tarjan's SCC algorithm adapted for undirected graphs. The key insight: edge `(u, v)` is a bridge iff there is no back edge from the subtree of `v` that reaches `u` or any ancestor of `u`. Equivalently, `low[v] > disc[u]` means no alternative path exists to `u`'s side.

Complexity: **O(V + E)** time, **O(V)** extra space.

---

## Where It Is Used

- Network reliability: finding single points of failure in communication networks
- Biconnected components: decomposing a graph into 2-edge-connected or 2-vertex-connected parts
- Road network analysis: critical roads/intersections
- Competitive programming: graph connectivity, edge/vertex criticality queries
- Database design: finding functional dependencies
- Bioinformatics: critical proteins in interaction networks

---

## Key Definitions

**Discovery time `disc[v]`:** The DFS timestamp when vertex `v` is first visited.

**Low-link `low[v]`:** The minimum discovery time reachable from the subtree rooted at `v` using at most one back edge to an ancestor.

```
low[v] = min(
    disc[v],
    min(disc[u])     for back edges (v, u) going to ancestors,
    min(low[child])  for tree edges (v, child)
)
```

**Bridge condition:** Edge `(u, v)` (where v is a child of u in DFS tree) is a bridge iff:
```
low[v] > disc[u]
```
No path from v's subtree reaches u or higher → removing (u,v) disconnects.

**Articulation point conditions:** Vertex `u` is an articulation point iff:
1. `u` is the **DFS root** and has **≥ 2 children** in the DFS tree, OR
2. `u` is **not the root** and has a child `v` with `low[v] >= disc[u]` (v's subtree cannot bypass u)

---

## Difference Between Bridge and Articulation Point Conditions

```
Bridge:              low[v] >  disc[u]   (strict — no path back at all)
Articulation point:  low[v] >= disc[u]   (non-strict — path back only reaches u itself)

If low[v] == disc[u]: removing edge (u,v) doesn't disconnect (v can still reach u via back edge),
                       but removing vertex u does disconnect (v loses its only connection to ancestors).
```

---

## Handling Multiple Edges (Multigraphs)

In standard DFS on undirected graphs, when processing neighbor `u` of `v`, skip `u` if it's the parent. But with **multiple edges** between the same pair of vertices, two edges from `v` to `u` mean one is a tree edge and one is a back edge — removing one does not disconnect.

**Solution:** Track parent edge index, not just parent vertex. Only skip the exact edge used to arrive at `v`, not all edges to the parent.

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| DFS traversal | O(V + E) | O(V) stack |
| Low-link computation | O(V + E) | O(V) |
| Bridge detection | O(V + E) | O(V) |
| Articulation point detection | O(V + E) | O(V) |
| Biconnected components | O(V + E) | O(V + E) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// BRIDGES AND ARTICULATION POINTS — Single DFS, O(V+E)
// ================================================================
struct BridgeAP {
    int n;
    vector<vector<pair<int,int>>> g; // g[u] = {(v, edge_id)}
    int num_edges;

    vector<int>  disc, low;
    vector<bool> is_ap;
    vector<bool> is_bridge;
    int timer_;

    BridgeAP(int n) : n(n), g(n), num_edges(0),
        disc(n,-1), low(n,0), is_ap(n,false),
        is_bridge(0), timer_(0) {}

    void add_edge(int u, int v) {
        int id = num_edges++;
        g[u].push_back({v, id});
        g[v].push_back({u, id});
        is_bridge.push_back(false);
    }

    // DFS from vertex v; parent_edge = edge id used to arrive at v (-1 for root)
    void dfs(int v, int parent_edge) {
        disc[v] = low[v] = timer_++;
        int children = 0; // children in DFS tree (for root AP check)

        for (auto [u, eid] : g[v]) {
            if (eid == parent_edge) continue; // skip the exact edge we came from

            if (disc[u] == -1) {
                // Tree edge
                children++;
                dfs(u, eid);
                low[v] = min(low[v], low[u]);

                // Bridge check: u's subtree has no back edge to v or above
                if (low[u] > disc[v])
                    is_bridge[eid] = true;

                // Articulation point check (non-root)
                if (parent_edge != -1 && low[u] >= disc[v])
                    is_ap[v] = true;
            } else {
                // Back edge: u is an ancestor already visited
                low[v] = min(low[v], disc[u]);
            }
        }

        // Root articulation point: 2+ children in DFS tree
        if (parent_edge == -1 && children >= 2)
            is_ap[v] = true;
    }

    void solve() {
        for (int v = 0; v < n; v++)
            if (disc[v] == -1) dfs(v, -1);
    }

    vector<pair<int,int>> get_bridges(const vector<pair<int,int>>& edges) {
        vector<pair<int,int>> result;
        for (int i = 0; i < (int)edges.size(); i++)
            if (is_bridge[i]) result.push_back(edges[i]);
        return result;
    }

    vector<int> get_aps() {
        vector<int> result;
        for (int v = 0; v < n; v++)
            if (is_ap[v]) result.push_back(v);
        return result;
    }
};

// ================================================================
// BICONNECTED COMPONENTS
// A biconnected component is a maximal 2-connected subgraph.
// Every biconnected component is an edge or a set of edges
// that form a cycle (no articulation point inside them).
// Decomposition: split at articulation points.
// ================================================================
struct BiconnectedComponents {
    int n;
    vector<vector<pair<int,int>>> g; // {neighbor, edge_id}
    int num_edges;

    vector<int>  disc, low;
    vector<bool> visited;
    stack<pair<int,int>> stk; // stack of edges
    vector<vector<pair<int,int>>> bccs; // each BCC = list of edges
    int timer_;

    BiconnectedComponents(int n) : n(n), g(n), num_edges(0),
        disc(n,-1), low(n,0), visited(n,false), timer_(0) {}

    void add_edge(int u, int v) {
        int id = num_edges++;
        g[u].push_back({v, id});
        g[v].push_back({u, id});
    }

    void dfs(int v, int parent_edge) {
        disc[v] = low[v] = timer_++;
        int children = 0;

        for (auto [u, eid] : g[v]) {
            if (eid == parent_edge) continue;

            if (disc[u] == -1) {
                children++;
                stk.push({v, u}); // push edge onto stack
                dfs(u, eid);
                low[v] = min(low[v], low[u]);

                // v is articulation point: pop BCC
                if ((parent_edge == -1 && children > 1) ||
                    (parent_edge != -1 && low[u] >= disc[v]))
                {
                    vector<pair<int,int>> bcc;
                    while (stk.top() != make_pair(v, u)) {
                        bcc.push_back(stk.top()); stk.pop();
                    }
                    bcc.push_back(stk.top()); stk.pop();
                    bccs.push_back(bcc);
                }
            } else if (disc[u] < disc[v]) { // back edge (avoid pushing twice)
                stk.push({v, u});
                low[v] = min(low[v], disc[u]);
            }
        }
    }

    void solve() {
        for (int v = 0; v < n; v++) {
            if (disc[v] == -1) {
                dfs(v, -1);
                // Pop remaining edges as one BCC
                if (!stk.empty()) {
                    vector<pair<int,int>> bcc;
                    while (!stk.empty()) { bcc.push_back(stk.top()); stk.pop(); }
                    bccs.push_back(bcc);
                }
            }
        }
    }
};

// ================================================================
// 2-EDGE-CONNECTED COMPONENTS
// Contract all bridges; each remaining connected component
// is 2-edge-connected (no bridge inside).
// ================================================================
struct TwoEdgeConnected {
    int n;
    vector<vector<pair<int,int>>> g;
    int num_edges;
    vector<int>  disc, low, comp;
    vector<bool> is_bridge;
    int timer_, num_comps;

    TwoEdgeConnected(int n) : n(n), g(n), num_edges(0),
        disc(n,-1), low(n,0), comp(n,-1),
        timer_(0), num_comps(0) {}

    void add_edge(int u, int v) {
        int id = num_edges++;
        g[u].push_back({v, id});
        g[v].push_back({u, id});
        is_bridge.push_back(false);
    }

    void dfs_bridge(int v, int pe) {
        disc[v] = low[v] = timer_++;
        for (auto [u, eid] : g[v]) {
            if (eid == pe) continue;
            if (disc[u] == -1) {
                dfs_bridge(u, eid);
                low[v] = min(low[v], low[u]);
                if (low[u] > disc[v]) is_bridge[eid] = true;
            } else {
                low[v] = min(low[v], disc[u]);
            }
        }
    }

    void dfs_comp(int v, int c) {
        comp[v] = c;
        for (auto [u, eid] : g[v]) {
            if (comp[u] == -1 && !is_bridge[eid])
                dfs_comp(u, c);
        }
    }

    // Returns number of 2-edge-connected components
    int solve() {
        for (int v = 0; v < n; v++)
            if (disc[v] == -1) dfs_bridge(v, -1);
        for (int v = 0; v < n; v++)
            if (comp[v] == -1) { dfs_comp(v, num_comps); num_comps++; }
        return num_comps;
    }

    // Build bridge tree: a tree on the 2ECC components
    vector<vector<int>> bridge_tree() {
        vector<vector<int>> tree(num_comps);
        for (int u = 0; u < n; u++)
            for (auto [v, eid] : g[u])
                if (is_bridge[eid] && comp[u] != comp[v])
                    tree[comp[u]].push_back(comp[v]);
        return tree;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // Graph: 0-1-2-0 (triangle, no bridges) + 2-3 (bridge) + 3-4-5-3 (cycle)
    {
        printf("=== Bridges and Articulation Points ===\n");
        BridgeAP bap(6);
        vector<pair<int,int>> edges = {
            {0,1},{1,2},{2,0},   // triangle: no bridges
            {2,3},               // bridge
            {3,4},{4,5},{5,3}    // cycle: no bridges
        };
        for (auto [u,v] : edges) bap.add_edge(u, v);
        bap.solve();

        printf("Bridges:\n");
        for (int i = 0; i < (int)edges.size(); i++)
            if (bap.is_bridge[i])
                printf("  (%d, %d)\n", edges[i].first, edges[i].second);

        printf("Articulation points: ");
        for (int v = 0; v < 6; v++)
            if (bap.is_ap[v]) printf("%d ", v);
        printf("\n");
        // Bridge: (2,3). AP: vertex 2 (connects triangle to rest) and 3
        // Actually: 2 is AP (remove it: {0,1} and {3,4,5} disconnect)
        // 3 is AP (remove it: {4,5} and the rest disconnect)? No: 4-5-3 is a cycle
        // but removing 3: {4,5} and {0,1,2} are disconnected.
        // So APs = {2, 3}. Verify with output.
    }

    // Biconnected components
    {
        printf("\n=== Biconnected Components ===\n");
        BiconnectedComponents bcc(6);
        bcc.add_edge(0,1); bcc.add_edge(1,2); bcc.add_edge(2,0);
        bcc.add_edge(2,3);
        bcc.add_edge(3,4); bcc.add_edge(4,5); bcc.add_edge(5,3);
        bcc.solve();

        printf("BCCs found: %d\n", (int)bcc.bccs.size());
        for (int i = 0; i < (int)bcc.bccs.size(); i++) {
            printf("  BCC %d: ", i);
            for (auto [u,v] : bcc.bccs[i]) printf("(%d,%d) ", u, v);
            printf("\n");
        }
    }

    // 2-edge-connected components + bridge tree
    {
        printf("\n=== 2-Edge-Connected Components ===\n");
        TwoEdgeConnected tec(6);
        tec.add_edge(0,1); tec.add_edge(1,2); tec.add_edge(2,0);
        tec.add_edge(2,3);
        tec.add_edge(3,4); tec.add_edge(4,5); tec.add_edge(5,3);
        int cnt = tec.solve();
        printf("2ECCs: %d\n", cnt);
        printf("comp: ");
        for (int v = 0; v < 6; v++) printf("%d ", tec.comp[v]);
        printf("\n");

        auto tree = tec.bridge_tree();
        printf("Bridge tree edges:\n");
        for (int u = 0; u < cnt; u++)
            for (int v : tree[u]) if (u < v)
                printf("  comp%d -- comp%d\n", u, v);
    }

    // Multigraph (two edges between same vertices)
    {
        printf("\n=== Multigraph (parallel edges) ===\n");
        BridgeAP bap(3);
        // Two edges between 0 and 1: not a bridge
        // One edge between 1 and 2: bridge
        bap.add_edge(0, 1);
        bap.add_edge(0, 1); // parallel edge
        bap.add_edge(1, 2);
        bap.solve();
        printf("Bridge (0,1) [parallel]: %s\n", bap.is_bridge[0] ? "yes":"no"); // no
        printf("Bridge (0,1) [parallel]: %s\n", bap.is_bridge[1] ? "yes":"no"); // no
        printf("Bridge (1,2): %s\n",            bap.is_bridge[2] ? "yes":"no"); // yes
        printf("AP vertex 1: %s\n",             bap.is_ap[1]     ? "yes":"no"); // yes
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

// ================================================================
// BRIDGES AND ARTICULATION POINTS
// ================================================================
public class BridgeAP {
    private readonly int n;
    private readonly List<(int v, int id)>[] g;
    private int numEdges;

    private readonly int[]  disc, low;
    public  readonly bool[] IsAp;
    public  readonly bool[] IsBridge;
    private int timer_;

    public BridgeAP(int n) {
        this.n = n;
        g = new List<(int,int)>[n];
        for (int i = 0; i < n; i++) g[i] = new();
        disc    = new int[n]; Array.Fill(disc, -1);
        low     = new int[n];
        IsAp    = new bool[n];
        IsBridge = Array.Empty<bool>();
    }

    public void AddEdge(int u, int v) {
        int id = numEdges++;
        g[u].Add((v, id)); g[v].Add((u, id));
        Array.Resize(ref IsBridge, numEdges);
    }

    private void Dfs(int v, int parentEdge) {
        disc[v] = low[v] = timer_++;
        int children = 0;

        foreach (var (u, eid) in g[v]) {
            if (eid == parentEdge) continue;

            if (disc[u] == -1) {
                children++;
                Dfs(u, eid);
                low[v] = Math.Min(low[v], low[u]);

                if (low[u] > disc[v])
                    IsBridge[eid] = true;

                if (parentEdge != -1 && low[u] >= disc[v])
                    IsAp[v] = true;
            } else {
                low[v] = Math.Min(low[v], disc[u]);
            }
        }

        if (parentEdge == -1 && children >= 2)
            IsAp[v] = true;
    }

    public void Solve() {
        for (int v = 0; v < n; v++)
            if (disc[v] == -1) Dfs(v, -1);
    }

    public List<int> GetAPs() {
        var res = new List<int>();
        for (int v = 0; v < n; v++) if (IsAp[v]) res.Add(v);
        return res;
    }

    public List<int> GetBridges() {
        var res = new List<int>();
        for (int i = 0; i < numEdges; i++) if (IsBridge[i]) res.Add(i);
        return res;
    }
}

// ================================================================
// 2-EDGE-CONNECTED COMPONENTS
// ================================================================
public class TwoEdgeConnected {
    private readonly int n;
    private readonly List<(int v, int id)>[] g;
    private int numEdges;
    private readonly int[]  disc, low;
    public  readonly int[]  Comp;
    private bool[] isBridge;
    private int timer_, numComps;
    public int NumComps => numComps;

    public TwoEdgeConnected(int n) {
        this.n = n;
        g = new List<(int,int)>[n];
        for (int i = 0; i < n; i++) g[i] = new();
        disc = new int[n]; Array.Fill(disc, -1);
        low  = new int[n];
        Comp = new int[n]; Array.Fill(Comp, -1);
        isBridge = Array.Empty<bool>();
    }

    public void AddEdge(int u, int v) {
        int id = numEdges++;
        g[u].Add((v, id)); g[v].Add((u, id));
        Array.Resize(ref isBridge, numEdges);
    }

    private void DfsBridge(int v, int pe) {
        disc[v] = low[v] = timer_++;
        foreach (var (u, eid) in g[v]) {
            if (eid == pe) continue;
            if (disc[u] == -1) {
                DfsBridge(u, eid);
                low[v] = Math.Min(low[v], low[u]);
                if (low[u] > disc[v]) isBridge[eid] = true;
            } else {
                low[v] = Math.Min(low[v], disc[u]);
            }
        }
    }

    private void DfsComp(int v, int c) {
        Comp[v] = c;
        foreach (var (u, eid) in g[v])
            if (Comp[u] == -1 && !isBridge[eid])
                DfsComp(u, c);
    }

    public int Solve() {
        for (int v = 0; v < n; v++) if (disc[v] == -1) DfsBridge(v, -1);
        for (int v = 0; v < n; v++) if (Comp[v] == -1) { DfsComp(v, numComps); numComps++; }
        return numComps;
    }

    public List<int>[] BridgeTree() {
        var tree = new List<int>[numComps];
        for (int i = 0; i < numComps; i++) tree[i] = new();
        for (int u = 0; u < n; u++)
            foreach (var (v, eid) in g[u])
                if (isBridge[eid] && Comp[u] != Comp[v])
                    tree[Comp[u]].Add(Comp[v]);
        return tree;
    }
}

public class Program {
    public static void Main() {
        // Graph: triangle 0-1-2 + bridge 2-3 + cycle 3-4-5
        var bap = new BridgeAP(6);
        bap.AddEdge(0,1); bap.AddEdge(1,2); bap.AddEdge(2,0);
        bap.AddEdge(2,3);
        bap.AddEdge(3,4); bap.AddEdge(4,5); bap.AddEdge(5,3);
        bap.Solve();

        Console.Write("Articulation points: ");
        Console.WriteLine(string.Join(" ", bap.GetAPs()));

        Console.Write("Bridge edge IDs: ");
        Console.WriteLine(string.Join(" ", bap.GetBridges()));

        // 2-edge-connected components
        var tec = new TwoEdgeConnected(6);
        tec.AddEdge(0,1); tec.AddEdge(1,2); tec.AddEdge(2,0);
        tec.AddEdge(2,3);
        tec.AddEdge(3,4); tec.AddEdge(4,5); tec.AddEdge(5,3);
        int cnt = tec.Solve();
        Console.WriteLine($"2ECCs: {cnt}");
        Console.Write("comp: ");
        for (int v = 0; v < 6; v++) Console.Write($"{tec.Comp[v]} ");
        Console.WriteLine();

        var tree = tec.BridgeTree();
        Console.WriteLine("Bridge tree:");
        for (int u = 0; u < cnt; u++)
            foreach (int v in tree[u]) if (u < v)
                Console.WriteLine($"  comp{u} -- comp{v}");
    }
}
```

---

## Visualization of Key Cases

```
Graph: 0-1-2-0 (triangle) -- bridge -- 3-4-5-3 (triangle)
              2 ---- 3
             / \    / \
            0---1  4---5

disc/low after DFS from 0:
  disc: [0, 1, 2, 3, 4, 5]
  low:  [0, 0, 0, 3, 3, 3]   (0,1,2 can reach 0; 3,4,5 can only reach 3)

Bridge (2,3): low[3]=3 > disc[2]=2  ✓
AP vertex 2:  low[3]=3 >= disc[2]=2 ✓  (non-root case)
AP vertex 3:  low[4]=3 >= disc[3]=3 ✓  (non-root case... but 3 is root of its subtree call)

No bridge in triangle: low[1]=0 <= disc[0]=0, low[2]=0 <= disc[0]=0
No bridge in 3-4-5 cycle: all low values = 3 = disc[3]
```

---

## Bridge Tree

After finding all bridges and 2-edge-connected components, the **bridge tree** is a tree where:
- Each node represents one 2-edge-connected component
- Each edge represents a bridge connecting two components

The bridge tree is useful for:
- Finding the minimum number of edges to add to make the graph 2-edge-connected (= `(leaves + 1) / 2` where leaves = leaf nodes of the bridge tree)
- Path queries between components (which bridges does a path cross?)
- Competitive programming: queries on paths in the bridge tree via LCA + HLD

---

## Pitfalls

- **Parent tracking must use edge ID, not vertex ID** — in multigraphs (multiple edges between same pair), tracking the parent by vertex skips all edges to the parent, incorrectly treating a second parallel edge as a back edge with `low[v] = min(low[v], disc[parent])`. This falsely eliminates the bridge between them. Always track the specific edge used to arrive (`parent_edge` = edge ID) and skip only that edge.
- **Root articulation point uses `>= 2 children`, others use `low[v] >= disc[u]`** — the root condition is different because the root has no ancestor that could provide an alternative path. For non-root vertices: `low[child] >= disc[v]` means the child's subtree cannot bypass `v`. For root: any single child is fine (root removal doesn't disconnect that subtree from others if there's only one child).
- **Bridge uses strict `>`, articulation point uses `>=`** — `low[v] > disc[u]` for bridge (no path back to u or above), `low[v] >= disc[u]` for AP (path back reaches at most u itself). Using `>=` for bridges incorrectly marks edges in simple cycles as bridges; using `>` for APs misses vertices that are cut by their own back-edge connections.
- **Back edge `low` update: use `disc[u]`, not `low[u]`** — when processing back edge `(v, u)` where u is an ancestor, update `low[v] = min(low[v], disc[u])`. Using `low[u]` instead can propagate lower values than valid (since `low[u]` may reflect paths not reachable from `v`'s subtree in undirected graphs). In directed graphs (Tarjan SCC), `low[u]` is correct for cross edges, but in undirected bridge/AP finding, always use `disc[u]` for the back edge update.
- **Recursion depth for large graphs** — DFS recurses up to depth V. For V = 10^5 in a linear chain, the call stack overflows. Use an iterative DFS variant for production code. The iterative version requires an explicit stack storing `(vertex, edge_iterator, parent_edge)` and a separate phase to process the low-link update after the child returns.
- **Biconnected components: edge-based, not vertex-based** — BCCs are defined by edges (each edge belongs to exactly one BCC), not by vertices (a vertex can belong to multiple BCCs — articulation points are shared). When reporting BCCs as vertex sets, include the articulation point in each adjacent BCC. Reporting only the non-AP vertices misses the connection point.

---

## Complexity Summary

| Problem | Time | Space |
|---|---|---|
| Bridges | O(V + E) | O(V) |
| Articulation points | O(V + E) | O(V) |
| Biconnected components | O(V + E) | O(V + E) |
| 2-edge-connected components | O(V + E) | O(V) |
| Bridge tree construction | O(V + E) | O(V) |

---

## Conclusion

Bridges and articulation points are the **fundamental vulnerability analysis of undirected graphs**:

- Both are found in a single O(V+E) DFS using low-link values — the same underlying technique as Tarjan's SCC for directed graphs, adapted for undirected connectivity.
- The distinction between bridge condition (`low[v] > disc[u]`, strict) and AP condition (`low[v] >= disc[u]`, non-strict) is critical and flows directly from the definitions.
- The **bridge tree** (built from 2-edge-connected components) is a key structure for further analysis: path queries between components, minimum edges to add for 2-edge-connectivity, and LCA-based bridge-crossing queries.
- Always track parent by **edge ID** rather than vertex ID to handle multigraphs correctly.

**Key takeaway:**  
The entire algorithm fits in 30 lines: DFS timestamp, low-link propagation (min of child's low for tree edges, disc[u] for back edges), bridge check on tree edge return (strict inequality), AP check on tree edge return (non-strict, plus root special case). The only implementation subtlety beyond Tarjan SCC is the parent-edge tracking instead of parent-vertex tracking.
