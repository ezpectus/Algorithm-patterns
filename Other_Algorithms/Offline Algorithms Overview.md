# Offline Algorithms Overview — DSU on Tree, Mo's Algorithm, Sweep Line, Persistent DS

## Origin & Motivation

Offline algorithms receive **all queries in advance** before answering any of them. This freedom — knowing future queries — enables dramatic complexity improvements over online approaches by reordering queries, batching them, or preprocessing the input in ways that make each query cheap.

The four techniques here cover the canonical offline patterns:

- **DSU on Tree (Small-to-Large / Euler Tour merging)** — subtree aggregate queries answered by merging small sets into large ones in DFS order.
- **Mo's Algorithm** — range queries on arrays answered by sorting queries to minimize the total movement of a sliding window.
- **Sweep Line** — geometric or interval queries answered by processing events in sorted order, maintaining a 1D data structure.
- **Persistent Data Structures** — functional versioning of a data structure; each update produces a new version, enabling point-in-time queries without storing full copies.

Each technique converts a naively O(n^2) or O(n^2 log n) problem into O(n log n) or O(n sqrt(n)).

---

## 1. DSU on Tree (Small-to-Large Merging / Euler Tour)

### Idea

For subtree queries (e.g., "count distinct colors in subtree of v"), the naive approach processes each subtree from scratch: O(n^2). DSU on Tree (also called "small-to-large merging" or "Sack" — Small-to-large Aggregation by Combining Kids) exploits the observation:

**When merging two sets, always iterate over the smaller set and insert into the larger.**

Each element is merged at most O(log n) times (each merge at least doubles the set it moves to), giving O(n log n) total insertions.

**Euler Tour variant:** Assign DFS timestamps. A subtree `[in[v], out[v]]` becomes a contiguous range. Apply Mo's algorithm or offline BFS on these ranges.

### Algorithm (Small-to-Large)

```
dfs(v):
    data[v] = {color[v]}
    for each child c of v:
        dfs(c)
        if |data[c]| > |data[v]|:
            swap(data[v], data[c])
        // Merge smaller (data[c]) into larger (data[v])
        for each element e in data[c]:
            data[v].insert(e)
    answer[v] = |data[v]|   // e.g., distinct element count
```

### Complexity

| Operation | Time |
|---|---|
| Total insertions across all merges | O(n log n) |
| Space | O(n) if using hash sets; O(n log n) for ordered sets |

---

## 2. Mo's Algorithm

### Idea

Given an array of length `n` and `q` range queries `[l, r]`, answer all queries offline by sorting them into an order that minimizes total movement of the range endpoints. Uses a **block decomposition**: sort queries by block of `l`, then by `r` within each block (alternating direction for cache efficiency).

The current range `[curL, curR]` slides to each query's range by adding/removing elements one at a time. If the add/remove operations cost O(1), total time is O((n + q) sqrt(n)).

### Query Ordering

```
Block size B = ceil(sqrt(n))
Sort queries by (l / B, r)  — or alternating r direction per block
```

Total movement analysis:
- `r` moves at most O(n) per block, O(n * (n/B)) = O(n^2/B) total
- `l` moves at most O(B) per query, O(qB) total
- Setting B = sqrt(n): both terms are O(n sqrt(n))

### Algorithm

```
Sort queries by Mo order
curL = 0, curR = -1
for each query (l, r, idx) in sorted order:
    while curR < r: add(++curR)
    while curL > l: add(--curL)
    while curR > r: remove(curR--)
    while curL < l: remove(curL++)
    answer[idx] = current_aggregate()
```

### Mo's on Trees

For path queries on trees, flatten the tree using Euler tour into an array, then apply Mo on the resulting sequence. Requires careful handling of the LCA.

### Complexity

| Version | Time | Space |
|---|---|---|
| Standard Mo | O((n+q) sqrt(n)) | O(n + q) |
| Mo with rollback (no delete) | O((n+q) sqrt(n)) | O(n + q) |
| Mo on trees | O((n+q) sqrt(n)) | O(n + q) |

---

## 3. Sweep Line

### Idea

Reduce a 2D problem to a sequence of 1D problems by sweeping a vertical (or horizontal) line across the x-axis. Events (left endpoint, right endpoint, query) are sorted by x-coordinate. As the sweep line moves, a 1D data structure (BIT, segment tree, sorted set) maintains the current state.

### Common Applications

| Problem | Events | 1D Structure |
|---|---|---|
| Rectangle area union | Interval open/close | Segment tree with lazy count |
| Point-in-rectangle queries | Rectangle open/close + point | BIT (y-coordinate) |
| Interval intersection count | Interval start/end | BIT |
| Closest pair of points | Point insert/delete | Sorted set (y-coordinate) |
| Segment intersections | Segment start/end | Balanced BST (y-order) |

### Algorithm (Rectangle area union)

```
Create events: for each rectangle [x1,x2] x [y1,y2]:
    add event (x1, +1, y1, y2)   // interval [y1,y2] becomes active
    add event (x2, -1, y1, y2)   // interval [y1,y2] becomes inactive
Sort events by x

Sweep:
    for each event at x (sorted):
        area += seg_tree.total_covered() * (x - prev_x)
        seg_tree.update(y1, y2, +1 or -1)   // add/remove interval
        prev_x = x
```

### Complexity

| Step | Time |
|---|---|
| Sort events | O(E log E) |
| Process each event | O(log n) per event |
| Total | O(E log n) |

---

## 4. Persistent Data Structures

### Idea

Instead of mutating a data structure in place, each update creates a **new version** by copying only the O(log n) nodes on the path from root to the modified leaf (path copying / structural sharing). All previous versions remain accessible.

**Persistent Segment Tree** is the most widely used: version `k` shares all unchanged nodes with version `k-1`. Space per update: O(log n) new nodes. Query on version `k`: O(log n).

### Key Application: Offline Range k-th Smallest

Build a persistent segment tree over sorted values. Version `i` = segment tree after inserting `A[0..i-1]`. Query `kth(l, r, k)`:

```
Subtract version (l-1) from version r node-by-node.
Left child count = version_r.left.count - version_(l-1).left.count
If k <= left_count: recurse left
Else: k -= left_count, recurse right
```

This answers k-th smallest in range [l, r] in O(log n) per query, O(n log n) build, O(n log n) space.

### Complexity

| Operation | Time | Space per version |
|---|---|---|
| Build (n insertions) | O(n log n) | O(log n) new nodes each |
| Point query on version k | O(log n) | O(1) |
| Range query (two versions) | O(log n) | O(1) |
| Total space | — | O(n log n) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// 1. DSU ON TREE — Small-to-Large merging
//    Query: count of distinct values in subtree
// ================================================================
struct DSUOnTree {
    int n;
    vector<vector<int>> adj;
    vector<int> color, answer;
    vector<set<int>*> data;

    DSUOnTree(int n, vector<int>& col)
        : n(n), adj(n), color(col), answer(n), data(n, nullptr) {}

    void add_edge(int u, int v) { adj[u].push_back(v); adj[v].push_back(u); }

    void dfs(int v, int par) {
        data[v] = new set<int>();
        data[v]->insert(color[v]);

        for (int c : adj[v]) {
            if (c == par) continue;
            dfs(c, v);

            // Small-to-large: merge smaller into larger
            if (data[c]->size() > data[v]->size())
                swap(data[c], data[v]);

            for (int x : *data[c])
                data[v]->insert(x);

            delete data[c];
            data[c] = nullptr;
        }
        answer[v] = (int)data[v]->size();
    }

    void solve(int root = 0) { dfs(root, -1); }
};

// ================================================================
// 2. DSU ON TREE — Euler Tour + offline subtree queries
//    (alternative: turn subtree into range query)
// ================================================================
struct EulerTourSubtree {
    int n, timer = 0;
    vector<vector<int>> adj;
    vector<int> in, out, order;

    EulerTourSubtree(int n) : n(n), adj(n), in(n), out(n) {}

    void add_edge(int u, int v) { adj[u].push_back(v); adj[v].push_back(u); }

    void dfs(int v, int p) {
        in[v] = timer++;
        order.push_back(v);
        for (int c : adj[v]) if (c != p) dfs(c, v);
        out[v] = timer - 1;
    }

    // Subtree of v = range [in[v], out[v]] in `order`
    // Apply any range query technique on order[]
};

// ================================================================
// 3. MO'S ALGORITHM
//    Query: count of distinct values in A[l..r]
// ================================================================
struct Mo {
    int n, q, block;
    vector<int>& A;
    vector<int> cnt; // cnt[x] = frequency of x in current window
    int distinct = 0;
    int curL = 0, curR = -1;

    Mo(int n, vector<int>& A) : n(n), q(0), block((int)ceil(sqrt(n))),
        A(A), cnt(*max_element(A.begin(), A.end()) + 2, 0) {}

    void add(int pos) {
        if (++cnt[A[pos]] == 1) distinct++;
    }
    void remove(int pos) {
        if (--cnt[A[pos]] == 0) distinct--;
    }

    vector<int> solve(vector<pair<int,int>>& queries) {
        q = queries.size();
        vector<int> idx(q);
        iota(idx.begin(), idx.end(), 0);

        sort(idx.begin(), idx.end(), [&](int a, int b) {
            int ba = queries[a].first / block;
            int bb = queries[b].first / block;
            if (ba != bb) return ba < bb;
            // Alternate sort direction for cache efficiency
            return (ba & 1) ? queries[a].second > queries[b].second
                            : queries[a].second < queries[b].second;
        });

        vector<int> ans(q);
        for (int i : idx) {
            int l = queries[i].first, r = queries[i].second;
            while (curR < r) add(++curR);
            while (curL > l) add(--curL);
            while (curR > r) remove(curR--);
            while (curL < l) remove(curL++);
            ans[i] = distinct;
        }
        return ans;
    }
};

// ================================================================
// 4. SWEEP LINE — count points inside rectangles
//    Offline: for each rectangle query [x1,x2] x [y1,y2],
//    count points (px, py) with x1<=px<=x2, y1<=py<=y2
// ================================================================
struct BIT {
    int n; vector<int> tree;
    BIT(int n) : n(n), tree(n+1, 0) {}
    void update(int i, int v) { for (i++; i<=n; i+=i&-i) tree[i]+=v; }
    int query(int i) { int s=0; for(i++; i>0; i-=i&-i) s+=tree[i]; return s; }
    int query(int l, int r) { return query(r) - (l>0?query(l-1):0); }
};

struct SweepLine {
    // Event: (x, type, y1, y2, query_idx)
    // type: 0=point, 1=rect_open(+), 2=rect_close(-)
    struct Event {
        int x, type, y, y2, idx, sign;
    };

    int max_y;
    vector<Event> events;
    int n_queries = 0;

    SweepLine(int max_y) : max_y(max_y) {}

    void add_point(int x, int y) {
        events.push_back({x, 0, y, y, -1, +1});
    }

    // query: count points in [x1,x2] x [y1,y2]
    void add_query(int x1, int y1, int x2, int y2, int idx) {
        // Decompose into prefix queries:
        // answer = f(x2,y2) - f(x1-1,y2) - f(x2,y1-1) + f(x1-1,y1-1)
        events.push_back({x2,   1, y1, y2, idx, +1});
        events.push_back({x1-1, 1, y1, y2, idx, -1});
        n_queries++;
    }

    vector<int> solve(int q) {
        vector<int> ans(q, 0);
        sort(events.begin(), events.end(), [](const Event& a, const Event& b){
            if (a.x != b.x) return a.x < b.x;
            return a.type < b.type; // points before queries at same x
        });

        BIT bit(max_y + 1);
        for (auto& e : events) {
            if (e.type == 0) {
                bit.update(e.y, +1);
            } else {
                int val = bit.query(e.y, e.y2) * e.sign;
                ans[e.idx] += val;
            }
        }
        return ans;
    }
};

// ================================================================
// 5. PERSISTENT SEGMENT TREE
//    Build: one version per prefix A[0..i]
//    Query: k-th smallest in A[l..r]
// ================================================================
struct PersistentSegTree {
    struct Node {
        int left, right, cnt;
    };

    int sz = 0;
    vector<Node> nodes;
    vector<int> roots;
    int n; // value range [0, n)

    PersistentSegTree(int n, int max_nodes)
        : n(n), nodes(max_nodes), roots(1, 0)
    {
        nodes[0] = {0, 0, 0};
        sz = 1;
    }

    int new_node() { nodes[sz] = {0, 0, 0}; return sz++; }

    // Insert value v into version `prev`, return new root
    int update(int prev, int lo, int hi, int v) {
        int cur = new_node();
        nodes[cur] = nodes[prev];
        nodes[cur].cnt++;
        if (lo == hi) return cur;
        int mid = (lo + hi) / 2;
        if (v <= mid)
            nodes[cur].left = update(nodes[prev].left, lo, mid, v);
        else
            nodes[cur].right = update(nodes[prev].right, mid+1, hi, v);
        return cur;
    }

    void insert(int v) {
        roots.push_back(update(roots.back(), 0, n-1, v));
    }

    // k-th smallest (1-indexed) in prefix [0..r] minus prefix [0..l-1]
    int kth(int l_root, int r_root, int lo, int hi, int k) {
        if (lo == hi) return lo;
        int mid = (lo + hi) / 2;
        int left_cnt = nodes[nodes[r_root].left].cnt
                     - nodes[nodes[l_root].left].cnt;
        if (k <= left_cnt)
            return kth(nodes[l_root].left, nodes[r_root].left, lo, mid, k);
        else
            return kth(nodes[l_root].right, nodes[r_root].right,
                       mid+1, hi, k - left_cnt);
    }

    // k-th smallest in A[l..r] (0-indexed, 1-indexed k)
    // roots[i] = version after inserting A[0..i-1]
    int query_kth(int l, int r, int k) {
        return kth(roots[l], roots[r+1], 0, n-1, k);
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // --- DSU on Tree: distinct colors in subtrees ---
    {
        int n = 7;
        vector<int> color = {1, 2, 1, 3, 2, 1, 3};
        DSUOnTree sack(n, color);
        sack.add_edge(0,1); sack.add_edge(0,2);
        sack.add_edge(1,3); sack.add_edge(1,4);
        sack.add_edge(2,5); sack.add_edge(2,6);
        sack.solve(0);
        printf("Distinct colors in subtree[0] = %d\n", sack.answer[0]); // 3
        printf("Distinct colors in subtree[1] = %d\n", sack.answer[1]); // 3
    }

    // --- Mo's Algorithm ---
    {
        vector<int> A = {1, 2, 1, 3, 2, 3, 1};
        Mo mo(A.size(), A);
        vector<pair<int,int>> queries = {{0,3},{2,5},{1,6}};
        auto ans = mo.solve(queries);
        printf("Mo distinct [0,3]=%d [2,5]=%d [1,6]=%d\n",
            ans[0], ans[1], ans[2]); // 3, 2, 3
    }

    // --- Persistent Segment Tree: k-th smallest ---
    {
        vector<int> A = {3, 1, 4, 1, 5, 9, 2, 6};
        int n = A.size();
        // Coordinate compress
        vector<int> sorted_A = A;
        sort(sorted_A.begin(), sorted_A.end());
        sorted_A.erase(unique(sorted_A.begin(), sorted_A.end()), sorted_A.end());
        int m = sorted_A.size();
        auto compress = [&](int v) {
            return (int)(lower_bound(sorted_A.begin(), sorted_A.end(), v) - sorted_A.begin());
        };

        PersistentSegTree pst(m, n * 40);
        for (int x : A) pst.insert(compress(x));

        // 2nd smallest in A[2..5] = {4,1,5,9} → sorted {1,4,5,9} → 2nd = 4
        int ans = pst.query_kth(2, 5, 2);
        printf("2nd smallest in A[2..5] = %d\n", sorted_A[ans]); // 4
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// ================================================================
// 1. SMALL-TO-LARGE MERGING (DSU on Tree)
// ================================================================
public class SmallToLarge {
    private readonly int n;
    private readonly List<int>[] adj;
    private readonly int[] color;
    public readonly int[] Answer;
    private readonly HashSet<int>[] data;

    public SmallToLarge(int n, int[] color) {
        this.n = n; this.color = color;
        adj    = Enumerable.Range(0, n).Select(_ => new List<int>()).ToArray();
        Answer = new int[n];
        data   = new HashSet<int>[n];
    }

    public void AddEdge(int u, int v) { adj[u].Add(v); adj[v].Add(u); }

    private void Dfs(int v, int par) {
        data[v] = new HashSet<int> { color[v] };
        foreach (int c in adj[v]) {
            if (c == par) continue;
            Dfs(c, v);
            if (data[c].Count > data[v].Count)
                (data[v], data[c]) = (data[c], data[v]);
            foreach (int x in data[c]) data[v].Add(x);
            data[c] = null!;
        }
        Answer[v] = data[v].Count;
    }

    public void Solve(int root = 0) => Dfs(root, -1);
}

// ================================================================
// 2. MO'S ALGORITHM
// ================================================================
public class Mo {
    private readonly int[] A;
    private readonly int block;
    private readonly int[] cnt;
    private int distinct = 0, curL = 0, curR = -1;

    public Mo(int[] A) {
        this.A = A;
        block  = (int)Math.Ceiling(Math.Sqrt(A.Length));
        cnt    = new int[A.Max() + 2];
    }

    private void Add(int pos) { if (++cnt[A[pos]] == 1) distinct++; }
    private void Remove(int pos) { if (--cnt[A[pos]] == 0) distinct--; }

    public int[] Solve(List<(int l, int r)> queries) {
        int q = queries.Count;
        var idx = Enumerable.Range(0, q).ToArray();
        Array.Sort(idx, (a, b) => {
            int ba = queries[a].l / block, bb = queries[b].l / block;
            if (ba != bb) return ba.CompareTo(bb);
            return (ba & 1) == 1
                ? queries[b].r.CompareTo(queries[a].r)
                : queries[a].r.CompareTo(queries[b].r);
        });

        var ans = new int[q];
        foreach (int i in idx) {
            var (l, r) = queries[i];
            while (curR < r) Add(++curR);
            while (curL > l) Add(--curL);
            while (curR > r) Remove(curR--);
            while (curL < l) Remove(curL++);
            ans[i] = distinct;
        }
        return ans;
    }
}

// ================================================================
// 3. SWEEP LINE — count points in rectangles
// ================================================================
public class SweepLine {
    private record Event(int X, int Type, int Y1, int Y2, int Idx, int Sign);
    private readonly List<Event> events = new();
    private readonly int maxY;

    public SweepLine(int maxY) { this.maxY = maxY; }

    public void AddPoint(int x, int y) =>
        events.Add(new Event(x, 0, y, y, -1, 1));

    public void AddQuery(int x1, int y1, int x2, int y2, int idx) {
        events.Add(new Event(x2,   1, y1, y2, idx, +1));
        events.Add(new Event(x1-1, 1, y1, y2, idx, -1));
    }

    public int[] Solve(int q) {
        var ans = new int[q];
        var sorted = events.OrderBy(e => e.X).ThenBy(e => e.Type).ToList();
        var bit = new int[maxY + 2];

        void Update(int i, int v) { for (i++; i <= maxY+1; i += i&-i) bit[i] += v; }
        int Query(int i) { int s=0; for (i++; i>0; i-=i&-i) s+=bit[i]; return s; }
        int QueryRange(int l, int r) => Query(r) - (l > 0 ? Query(l-1) : 0);

        foreach (var e in sorted) {
            if (e.Type == 0) Update(e.Y1, 1);
            else ans[e.Idx] += QueryRange(e.Y1, e.Y2) * e.Sign;
        }
        return ans;
    }
}

// ================================================================
// 4. PERSISTENT SEGMENT TREE
// ================================================================
public class PersistentSegTree {
    private struct Node { public int Left, Right, Cnt; }

    private readonly Node[] nodes;
    private readonly List<int> roots = new() { 0 };
    private int sz = 1;
    private readonly int n;

    public PersistentSegTree(int n, int maxNodes) {
        this.n = n;
        nodes  = new Node[maxNodes];
    }

    private int NewNode() { nodes[sz] = new Node(); return sz++; }

    private int Update(int prev, int lo, int hi, int v) {
        int cur = NewNode();
        nodes[cur] = nodes[prev];
        nodes[cur].Cnt++;
        if (lo == hi) return cur;
        int mid = (lo + hi) / 2;
        if (v <= mid)
            nodes[cur].Left = Update(nodes[prev].Left, lo, mid, v);
        else
            nodes[cur].Right = Update(nodes[prev].Right, mid+1, hi, v);
        return cur;
    }

    public void Insert(int v) => roots.Add(Update(roots[^1], 0, n-1, v));

    private int Kth(int lRoot, int rRoot, int lo, int hi, int k) {
        if (lo == hi) return lo;
        int mid = (lo + hi) / 2;
        int leftCnt = nodes[nodes[rRoot].Left].Cnt - nodes[nodes[lRoot].Left].Cnt;
        if (k <= leftCnt)
            return Kth(nodes[lRoot].Left, nodes[rRoot].Left, lo, mid, k);
        return Kth(nodes[lRoot].Right, nodes[rRoot].Right, mid+1, hi, k - leftCnt);
    }

    public int QueryKth(int l, int r, int k) =>
        Kth(roots[l], roots[r+1], 0, n-1, k);
}

public class Program {
    public static void Main() {
        // Small-to-large
        var stl = new SmallToLarge(7, new[]{1,2,1,3,2,1,3});
        stl.AddEdge(0,1); stl.AddEdge(0,2);
        stl.AddEdge(1,3); stl.AddEdge(1,4);
        stl.AddEdge(2,5); stl.AddEdge(2,6);
        stl.Solve();
        Console.WriteLine($"Distinct subtree[0]={stl.Answer[0]}, subtree[1]={stl.Answer[1]}");

        // Mo's
        var mo   = new Mo(new[]{1,2,1,3,2,3,1});
        var qMo  = new List<(int,int)>{ (0,3),(2,5),(1,6) };
        var ansMo = mo.Solve(qMo);
        Console.WriteLine($"Mo [0,3]={ansMo[0]} [2,5]={ansMo[1]} [1,6]={ansMo[2]}");

        // Sweep line
        var sweep = new SweepLine(10);
        sweep.AddPoint(2, 3); sweep.AddPoint(5, 7); sweep.AddPoint(1, 5);
        sweep.AddQuery(1, 2, 6, 8, 0);
        var ansSweep = sweep.Solve(1);
        Console.WriteLine($"Points in rect [1,2]x[6,8]: {ansSweep[0]}");

        // Persistent segment tree
        int[] A = {3,1,4,1,5,9,2,6};
        var sortedA = A.Distinct().OrderBy(x=>x).ToArray();
        int Compress(int v) => Array.BinarySearch(sortedA, v);
        var pst = new PersistentSegTree(sortedA.Length, A.Length * 40);
        foreach (int x in A) pst.Insert(Compress(x));
        int res = pst.QueryKth(2, 5, 2);
        Console.WriteLine($"2nd smallest in A[2..5] = {sortedA[res]}");
    }
}
```

---

## Technique Selection Guide

```
Query type?
│
├── Subtree queries on tree (static tree, offline)
│       → DSU on Tree / Small-to-Large merging   O(n log n)
│       → Euler Tour + range query technique
│
├── Range queries on array (offline, add/remove O(1))
│       → Mo's Algorithm                          O((n+q) sqrt(n))
│
├── Path queries on tree (offline)
│       → Mo on Trees (Euler tour flattening)     O((n+q) sqrt(n))
│
├── Geometric / interval problems (sorted by one dimension)
│       → Sweep Line + 1D data structure          O(E log n)
│
├── Historical / versioned queries ("state at time k")
│       → Persistent Data Structure               O(log n) per query
│
└── Range k-th order statistics (offline or online)
        → Persistent Segment Tree                 O(n log n) build, O(log n) query
```

---

## Pitfalls

**DSU on Tree / Small-to-Large:**
- **Swapping data pointers, not contents** — the key operation is swapping which set is "large" before merging. If the swap is omitted or done incorrectly, the merge degenerates to O(n^2) in the worst case (always merging into the small set). Always verify that after the swap, `|data[v]| >= |data[c]|`.
- **Memory management** — in C++, using `set` or `unordered_set` via pointers and deleting merged sets avoids O(n log n) memory. Forgetting to delete the smaller set after merging leaks memory on large inputs.

**Mo's Algorithm:**
- **Block size tuning** — the optimal block size is `sqrt(n)` for equal add/remove costs. If add/remove is O(log n), the optimal block size changes to `sqrt(n log n)`. Using the wrong block size for the operation cost increases total time by a log factor.
- **Expansion before contraction** — when moving from `[curL, curR]` to `[l, r]`, always expand first (increase range), then contract. Contracting first may temporarily make the range invalid (curL > curR), corrupting the aggregate. The order: grow R, grow L, shrink R, shrink L.
- **Mo ordering with ties** — breaking ties by `r` direction (alternating per block) is the "hilbert curve" optimization and reduces cache misses. Without it, Mo is still correct but ~30% slower in practice.

**Sweep Line:**
- **Event ordering at same x** — when a point and a query rectangle boundary share the same x-coordinate, process points before open-rectangle events and after close-rectangle events (or vice versa, depending on whether endpoints are inclusive/exclusive). Getting this wrong produces off-by-one errors on boundary cases.
- **Coordinate compression on y** — the BIT/segment tree operates on y-coordinates. If y-values are large (up to 10^9), compress them to indices in `[0, m)` first. Forgetting compression causes MLE or index out-of-bounds.

**Persistent Segment Tree:**
- **Node count budget** — each of n insertions creates O(log n) new nodes. Allocate `n * (log2(n) + 5)` nodes minimum. Under-allocating causes out-of-bounds writes that corrupt unrelated nodes silently.
- **Root indexing off by one** — `roots[i]` = version after inserting A[0..i-1]. Query `kth(l, r, k)` uses `roots[l]` and `roots[r+1]`. Using `roots[l-1]` or `roots[r]` shifts the prefix count by one element, producing wrong k-th values.
- **Coordinate compression before insert** — the persistent segment tree indexes by value, not by position. Values must be compressed to `[0, m)` before insertion, and the result decompressed after the query. Inserting raw large values causes index out-of-bounds.
- **No in-place modification** — every update must create new nodes; never modify an existing node's fields after it has been assigned to a version root. In-place modification destroys previous versions, making the structure non-persistent.

---

## Complexity Summary

| Technique | Build | Per Query | Total |
|---|---|---|---|
| Small-to-Large DSU on Tree | — | amortized O(log^2 n) | O(n log n) |
| Mo's Algorithm | O(q log q) sort | O(sqrt(n)) amortized | O((n+q) sqrt(n)) |
| Sweep Line | O(E log E) sort | O(log n) | O(E log n) |
| Persistent Segment Tree | O(n log n) | O(log n) | O((n+q) log n) |

---

## Conclusion

These four offline techniques collectively cover the most important classes of offline algorithmic problems:

- **DSU on Tree / Small-to-Large** converts tree subtree problems into O(n log n) via the fundamental "always merge smaller into larger" invariant.
- **Mo's Algorithm** turns arbitrary range queries into O(n sqrt(n)) via geometric query ordering — the single most broadly applicable offline array technique.
- **Sweep Line** reduces 2D problems to 1D by processing events in sorted order — the backbone of computational geometry algorithms.
- **Persistent Data Structures** enable time-travel queries at O(log n) cost per version by sharing structure between versions.

**Key takeaway:**  
The decision to use an offline technique is made at the problem level: if all queries are known in advance and there is no mandatory online constraint, always consider offline. The gain ranges from a factor of sqrt(n) (Mo) to a factor of n/log(n) (persistent vs naive). Choose based on what structure the problem exposes: tree subtrees → DSU on tree; array ranges → Mo; geometric ordering → sweep line; versioned queries → persistence.
