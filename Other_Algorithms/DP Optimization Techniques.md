# DP Optimization Techniques — Divide & Conquer, Knuth, Convex Hull Trick

## Origin & Motivation

Standard DP recurrences of the form `dp[i][j] = min over k < j of (dp[i-1][k] + cost(k, j))` run in O(n^3) or O(n^2) naively. Three classical optimizations reduce these to O(n log n) or O(n^2) by exploiting structural properties of the cost function and the optimal split point:

- **Divide & Conquer optimization** — when the optimal split `opt(i, j)` is monotone in `j`, divide the j-range in half and recurse. Requires: `opt(i, j) <= opt(i, j+1)`.
- **Knuth's optimization** — when the cost function satisfies the **quadrangle inequality** and **monotonicity**, the optimal split is monotone in both `i` and `j`. Reduces O(n^3) to O(n^2).
- **Convex Hull Trick (CHT)** — when the recurrence decomposes into a linear function of a previous DP state: `dp[j] = min over i of (m[i] * x[j] + b[i])`. Maintains a convex hull of lines; query each `x[j]` for the minimum. O(n log n) or O(n) depending on query order.

Each optimization applies to a specific structural class of DP — identifying which class applies to a given problem is the main skill.

---

## Where It Is Used

| Optimization | Typical Problems |
|---|---|
| Divide & Conquer | Optimal BST layers, SMAWK-adjacent problems, 1D-1D DP with monotone opt |
| Knuth | Optimal BST, matrix chain (specific forms), optimal polygon triangulation |
| Convex Hull Trick | Slope trick, line container problems, 1D DP with linear cost decomposition |
| Li Chao Tree (CHT variant) | Online CHT with arbitrary query order, non-monotone slopes |

---

## 1. Divide & Conquer Optimization

### Condition

Recurrence: `dp[i][j] = min_{k in [lo, j-1]} (dp[i-1][k] + cost(k, j))`

Optimization applies when the optimal `k` (call it `opt[i][j]`) satisfies:

```
opt[i][j] <= opt[i][j+1]    for all valid i, j
```

This is the **monotonicity of the optimal split point** in j.

### Algorithm

```
solve(layer i, j_lo, j_hi, k_lo, k_hi):
    j_mid = (j_lo + j_hi) / 2
    find opt[i][j_mid] by scanning k in [k_lo, min(k_hi, j_mid-1)]
    dp[i][j_mid] = min over that range

    solve(i, j_lo,   j_mid-1, k_lo,       opt[i][j_mid])
    solve(i, j_mid+1, j_hi,   opt[i][j_mid], k_hi)
```

Each level of recursion touches O(n) (k, j) pairs total → O(n log n) per DP layer → O(kn log n) for k layers.

### Complexity

| Naive | O(kn^2) |
|---|---|
| D&C optimized | O(kn log n) |

---

## 2. Knuth's Optimization

### Conditions

Recurrence: `dp[i][j] = min_{i < k < j} (dp[i][k] + dp[k][j]) + cost(i, j)`

Two conditions must hold:

**Quadrangle inequality (QI):** For `a <= b <= c <= d`:
```
cost(a, c) + cost(b, d) <= cost(a, d) + cost(b, c)
```

**Monotonicity:** cost is monotone: `cost(b, c) <= cost(a, d)` when `a <= b <= c <= d`.

If both hold, the optimal split `opt[i][j]` satisfies:
```
opt[i][j-1] <= opt[i][j] <= opt[i+1][j]
```

### Algorithm

Fill dp table diagonally (by interval length). For each cell `(i, j)`, scan k only in `[opt[i][j-1], opt[i+1][j]]`.

```
for len = 2 to n:
    for i = 0 to n-len:
        j = i + len
        dp[i][j] = INF
        for k = opt[i][j-1] to opt[i+1][j]:
            if dp[i][k] + dp[k][j] + cost(i,j) < dp[i][j]:
                dp[i][j] = dp[i][k] + dp[k][j] + cost(i,j)
                opt[i][j] = k
```

### Complexity

| Naive | O(n^3) |
|---|---|
| Knuth optimized | O(n^2) |

---

## 3. Convex Hull Trick (CHT)

### Condition

Recurrence decomposes as:

```
dp[j] = min over i < j of { dp[i] + cost(i, j) }
```

where `cost(i, j) = m[i] * x[j] + b[i]` — a **linear function** in `x[j]` with slope `m[i]` and intercept `b[i]`.

Each "previous state" i defines a line `y = m[i] * x + b[i]`. The recurrence asks: for query point `x[j]`, find the line with minimum y-value.

This is exactly the problem of maintaining a lower envelope of lines — the **convex hull** of lines.

### Variants

| Variant | Slopes | Queries | Complexity |
|---|---|---|---|
| Offline monotone slopes + monotone queries | Decreasing | Increasing | O(n) — two pointers |
| Offline monotone slopes + arbitrary queries | Decreasing | Arbitrary | O(n log n) — binary search on hull |
| Online arbitrary slopes + arbitrary queries | Arbitrary | Arbitrary | O(n log n) — Li Chao Tree |

### CHT — Offline, Monotone Slopes and Queries

Maintain a deque of lines forming the lower convex hull. Lines are added in order of decreasing slope. Queries are processed in increasing order of x — a pointer advances monotonically along the hull.

```
Intersection of lines L1: y = m1*x + b1  and  L2: y = m2*x + b2:
x_intersect = (b2 - b1) / (m1 - m2)

Line L2 is made redundant by L1 and L3 if:
intersect(L1, L3) <= intersect(L1, L2)
```

### Li Chao Tree — Online, Arbitrary Slopes and Queries

A segment tree over the x-domain. Each node stores a "dominant" line — the one that wins the query at the midpoint. Line insertion takes O(log n) by pushing the non-dominant line to children. Query takes O(log n) by traversing the path and taking the minimum across all dominant lines on the path.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;
const ll INF = 1e18;

// ================================================================
// 1. DIVIDE & CONQUER OPTIMIZATION
// ================================================================
// Example problem: dp[t][j] = min_{k < j} (dp[t-1][k] + cost(k, j))
// cost(k, j) must have monotone optimal split.

struct DnCDP {
    int n, layers;
    vector<vector<ll>> dp;
    // cost function — problem specific; here: (j-k)^2 as placeholder
    function<ll(int,int)> cost;

    DnCDP(int n, int layers, function<ll(int,int)> cost)
        : n(n), layers(layers), dp(layers+1, vector<ll>(n+1, INF)), cost(cost)
    {}

    void solve_layer(int t, int j_lo, int j_hi, int k_lo, int k_hi) {
        if (j_lo > j_hi) return;
        int j_mid = (j_lo + j_hi) / 2;
        int opt = k_lo;
        ll best = INF;

        int k_hi_clamp = min(k_hi, j_mid - 1);
        for (int k = k_lo; k <= k_hi_clamp; k++) {
            if (dp[t-1][k] == INF) continue;
            ll val = dp[t-1][k] + cost(k, j_mid);
            if (val < best) { best = val; opt = k; }
        }
        dp[t][j_mid] = best;

        solve_layer(t, j_lo,     j_mid-1, k_lo, opt);
        solve_layer(t, j_mid+1, j_hi,    opt,   k_hi);
    }

    // Run all layers. dp[0][0] = 0 must be set before calling.
    vector<ll> run() {
        dp[0][0] = 0;
        for (int t = 1; t <= layers; t++) {
            fill(dp[t].begin(), dp[t].end(), INF);
            solve_layer(t, 1, n, 0, n-1);
        }
        return dp[layers];
    }
};

// ================================================================
// 2. KNUTH'S OPTIMIZATION
// ================================================================
// dp[i][j] = min_{i < k < j} (dp[i][k] + dp[k][j]) + cost(i, j)
// Requires quadrangle inequality on cost.

struct KnuthDP {
    int n;
    vector<vector<ll>> dp, opt;
    function<ll(int,int)> cost;

    KnuthDP(int n, function<ll(int,int)> cost)
        : n(n), dp(n, vector<ll>(n, 0)),
          opt(n, vector<ll>(n, 0)), cost(cost)
    {}

    ll solve() {
        // Base: intervals of length 1 are 0
        for (int i = 0; i < n; i++) {
            dp[i][i] = 0;
            opt[i][i] = i;
        }
        // Base: intervals of length 2
        for (int i = 0; i < n-1; i++) {
            dp[i][i+1] = cost(i, i+1);
            opt[i][i+1] = i;
        }

        for (int len = 2; len < n; len++) {
            for (int i = 0; i + len < n; i++) {
                int j = i + len;
                dp[i][j] = INF;
                // Knuth: scan k in [opt[i][j-1], opt[i+1][j]]
                int k_lo = (int)opt[i][j-1];
                int k_hi = (j+1 < n) ? (int)opt[i+1][j] : j-1;
                k_hi = min(k_hi, j-1);

                for (int k = k_lo; k <= k_hi; k++) {
                    ll val = dp[i][k] + dp[k+1][j] + cost(i, j);
                    if (val < dp[i][j]) {
                        dp[i][j] = val;
                        opt[i][j] = k;
                    }
                }
            }
        }
        return dp[0][n-1];
    }
};

// ================================================================
// 3. CONVEX HULL TRICK — Offline, monotone slopes + monotone queries
// ================================================================
// Maintains lower convex hull of lines y = m*x + b.
// Add lines in DECREASING slope order.
// Query points in INCREASING order.

struct CHT {
    struct Line { ll m, b; };
    deque<Line> hull;
    int ptr = 0; // pointer for monotone queries

    // x-coordinate of intersection of L1 and L2
    // Returns intersection * (m1 - m2) to avoid floating point
    // We compare intersections: bad(L1, L2, L3) if L2 is above hull
    bool bad(Line l1, Line l2, Line l3) {
        // L2 is redundant if intersection(l1,l3) <= intersection(l1,l2)
        // (b3-b1)/(m1-m3) <= (b2-b1)/(m1-m2)
        // Cross multiply (careful with sign since m1>m2>m3, denom positive):
        return (__int128)(l3.b - l1.b) * (l1.m - l2.m)
            <= (__int128)(l2.b - l1.b) * (l1.m - l3.m);
    }

    // Add line y = m*x + b. Slopes must be added in DECREASING order.
    void add(ll m, ll b) {
        Line l = {m, b};
        while (hull.size() >= 2 && bad(hull[hull.size()-2], hull[hull.size()-1], l))
            hull.pop_back();
        hull.push_back(l);
    }

    // Query minimum y at x. x must be INCREASING across calls.
    ll query(ll x) {
        while (ptr + 1 < (int)hull.size() &&
               hull[ptr+1].m * x + hull[ptr+1].b <= hull[ptr].m * x + hull[ptr].b)
            ptr++;
        return hull[ptr].m * x + hull[ptr].b;
    }

    // Query minimum y at arbitrary x (binary search on hull)
    ll query_bs(ll x) {
        int lo = 0, hi = (int)hull.size() - 1;
        while (lo < hi) {
            int mid = (lo + hi) / 2;
            if (hull[mid].m * x + hull[mid].b <= hull[mid+1].m * x + hull[mid+1].b)
                hi = mid;
            else
                lo = mid + 1;
        }
        return hull[lo].m * x + hull[lo].b;
    }
};

// ================================================================
// 4. LI CHAO TREE — Online CHT, arbitrary slopes and queries
// ================================================================
struct LiChaoTree {
    struct Line {
        ll m, b;
        ll eval(ll x) const { return m * x + b; }
    };

    static const ll NEG_INF = -1e18;
    static const ll POS_INF =  1e18;

    struct Node {
        Line line;
        bool has_line = false;
        int  left = 0, right = 0;
    };

    vector<Node> nodes;
    ll x_lo, x_hi;
    int root;

    LiChaoTree(ll x_lo, ll x_hi) : x_lo(x_lo), x_hi(x_hi) {
        nodes.push_back({}); // dummy node 0
        root = new_node();
    }

    int new_node() {
        nodes.push_back({});
        return (int)nodes.size() - 1;
    }

    void add_line(int v, ll lo, ll hi, Line line) {
        if (!nodes[v].has_line) {
            nodes[v].line = line;
            nodes[v].has_line = true;
            return;
        }

        ll mid = lo + (hi - lo) / 2;
        bool left_better  = line.eval(lo)  < nodes[v].line.eval(lo);
        bool mid_better   = line.eval(mid) < nodes[v].line.eval(mid);

        if (mid_better) swap(nodes[v].line, line);

        if (lo == hi) return;

        if (left_better != mid_better) {
            if (!nodes[v].left) nodes[v].left = new_node();
            add_line(nodes[v].left, lo, mid, line);
        } else {
            if (!nodes[v].right) nodes[v].right = new_node();
            add_line(nodes[v].right, mid+1, hi, line);
        }
    }

    ll query(int v, ll lo, ll hi, ll x) {
        if (!v) return POS_INF;
        ll res = nodes[v].has_line ? nodes[v].line.eval(x) : POS_INF;
        if (lo == hi) return res;
        ll mid = lo + (hi - lo) / 2;
        if (x <= mid) return min(res, query(nodes[v].left,  lo,  mid, x));
        else          return min(res, query(nodes[v].right, mid+1, hi, x));
    }

    void add(ll m, ll b) { add_line(root, x_lo, x_hi, {m, b}); }
    ll query(ll x)       { return query(root, x_lo, x_hi, x);  }
};

// ================================================================
// Usage examples
// ================================================================
int main() {
    // --- D&C DP: split array into k groups minimizing sum of (group_sum)^2 ---
    // Simplified: cost(i,j) = (j-i)*(j-i)
    {
        int n = 8, k = 3;
        auto cost = [](int i, int j) -> ll { return (ll)(j-i)*(j-i); };
        DnCDP dnc(n, k, cost);
        auto result = dnc.run();
        printf("D&C DP dp[%d][%d] = %lld\n", k, n, result[n]);
    }

    // --- Knuth: optimal interval merging with cost = interval length ---
    {
        int n = 5;
        // cost(i,j) = j - i (sum of interval lengths for merge-sort-style cost)
        auto cost = [](int i, int j) -> ll { return j - i; };
        KnuthDP knuth(n, cost);
        printf("Knuth DP result = %lld\n", knuth.solve());
    }

    // --- CHT: classic "minimum cost to hire workers" style ---
    // dp[j] = min over i < j of (dp[i] + a[i] * b[j])
    // slope = a[i] (decreasing), query = b[j] (increasing)
    {
        int n = 6;
        vector<ll> a = {10, 8, 6, 4, 2, 1};
        vector<ll> b = {1,  2, 3, 4, 5, 6};
        vector<ll> dp(n, INF);
        dp[0] = 0;
        CHT cht;
        cht.add(a[0], dp[0]);
        for (int j = 1; j < n; j++) {
            dp[j] = cht.query(b[j]);
            cht.add(a[j], dp[j]);
        }
        printf("CHT DP final = %lld\n", dp[n-1]);
    }

    // --- Li Chao Tree: arbitrary query order ---
    {
        LiChaoTree lct(-1000, 1000);
        lct.add(3, 1);    // y = 3x + 1
        lct.add(-2, 10);  // y = -2x + 10
        lct.add(1, -5);   // y = x - 5

        printf("Li Chao min at x=2: %lld\n", lct.query(2));  // min(7, 6, -3) = -3
        printf("Li Chao min at x=5: %lld\n", lct.query(5));  // min(16,-0,0)  = 0
        printf("Li Chao min at x=-3: %lld\n", lct.query(-3));// min(-8,16,-8) = -8
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
// 1. DIVIDE & CONQUER DP
// ================================================================
public class DnCDP {
    private readonly int n, layers;
    private readonly long[,] dp;
    private readonly Func<int,int,long> cost;

    public DnCDP(int n, int layers, Func<int,int,long> cost) {
        this.n = n; this.layers = layers; this.cost = cost;
        dp = new long[layers+1, n+1];
        for (int i = 0; i <= layers; i++)
            for (int j = 0; j <= n; j++) dp[i,j] = long.MaxValue/2;
    }

    private void SolveLayer(int t, int jLo, int jHi, int kLo, int kHi) {
        if (jLo > jHi) return;
        int jMid = (jLo + jHi) / 2;
        int opt = kLo;
        long best = long.MaxValue/2;

        for (int k = kLo; k <= Math.Min(kHi, jMid-1); k++) {
            if (dp[t-1,k] == long.MaxValue/2) continue;
            long val = dp[t-1,k] + cost(k, jMid);
            if (val < best) { best = val; opt = k; }
        }
        dp[t, jMid] = best;

        SolveLayer(t, jLo,     jMid-1, kLo, opt);
        SolveLayer(t, jMid+1, jHi,    opt,  kHi);
    }

    public long[] Run() {
        dp[0,0] = 0;
        for (int t = 1; t <= layers; t++)
            SolveLayer(t, 1, n, 0, n-1);
        var result = new long[n+1];
        for (int j = 0; j <= n; j++) result[j] = dp[layers,j];
        return result;
    }
}

// ================================================================
// 2. KNUTH'S OPTIMIZATION
// ================================================================
public class KnuthDP {
    private readonly int n;
    private readonly long[,] dp, opt;
    private readonly Func<int,int,long> cost;

    public KnuthDP(int n, Func<int,int,long> cost) {
        this.n = n; this.cost = cost;
        dp  = new long[n,n];
        opt = new long[n,n];
    }

    public long Solve() {
        for (int i = 0; i < n; i++) { dp[i,i] = 0; opt[i,i] = i; }
        for (int i = 0; i < n-1; i++) { dp[i,i+1] = cost(i,i+1); opt[i,i+1] = i; }

        for (int len = 2; len < n; len++) {
            for (int i = 0; i+len < n; i++) {
                int j = i + len;
                dp[i,j] = long.MaxValue/2;
                int kLo = (int)opt[i, j-1];
                int kHi = (j+1 < n) ? (int)opt[i+1, j] : j-1;
                kHi = Math.Min(kHi, j-1);

                for (int k = kLo; k <= kHi; k++) {
                    long val = dp[i,k] + dp[k+1,j] + cost(i,j);
                    if (val < dp[i,j]) { dp[i,j] = val; opt[i,j] = k; }
                }
            }
        }
        return dp[0, n-1];
    }
}

// ================================================================
// 3. CONVEX HULL TRICK — monotone slopes + monotone queries
// ================================================================
public class CHT {
    private readonly record struct Line(long M, long B) {
        public long Eval(long x) => M * x + B;
    }

    private readonly List<Line> hull = new();
    private int ptr = 0;

    private bool Bad(Line l1, Line l2, Line l3) =>
        (decimal)(l3.B - l1.B) * (l1.M - l2.M) <=
        (decimal)(l2.B - l1.B) * (l1.M - l3.M);

    public void Add(long m, long b) {
        var l = new Line(m, b);
        while (hull.Count >= 2 && Bad(hull[^2], hull[^1], l))
            hull.RemoveAt(hull.Count - 1);
        hull.Add(l);
    }

    // Monotone increasing x
    public long Query(long x) {
        while (ptr + 1 < hull.Count && hull[ptr+1].Eval(x) <= hull[ptr].Eval(x))
            ptr++;
        return hull[ptr].Eval(x);
    }

    // Arbitrary x (binary search)
    public long QueryBS(long x) {
        int lo = 0, hi = hull.Count - 1;
        while (lo < hi) {
            int mid = (lo + hi) / 2;
            if (hull[mid].Eval(x) <= hull[mid+1].Eval(x)) hi = mid;
            else lo = mid + 1;
        }
        return hull[lo].Eval(x);
    }
}

// ================================================================
// 4. LI CHAO TREE
// ================================================================
public class LiChaoTree {
    private record struct Line(long M, long B) {
        public long Eval(long x) => M * x + B;
    }

    private class Node {
        public Line?  Line;
        public Node?  Left, Right;
    }

    private readonly Node root = new();
    private readonly long xLo, xHi;
    private const long Inf = long.MaxValue / 2;

    public LiChaoTree(long xLo, long xHi) { this.xLo = xLo; this.xHi = xHi; }

    private void AddLine(Node node, long lo, long hi, Line line) {
        if (node.Line is null) { node.Line = line; return; }

        long mid = lo + (hi - lo) / 2;
        bool leftBetter = line.Eval(lo)  < node.Line.Value.Eval(lo);
        bool midBetter  = line.Eval(mid) < node.Line.Value.Eval(mid);

        if (midBetter) {
            var tmp = node.Line.Value;
            node.Line = line;
            line = tmp;
        }

        if (lo == hi) return;

        if (leftBetter != midBetter) {
            node.Left ??= new Node();
            AddLine(node.Left, lo, mid, line);
        } else {
            node.Right ??= new Node();
            AddLine(node.Right, mid+1, hi, line);
        }
    }

    private long Query(Node? node, long lo, long hi, long x) {
        if (node is null) return Inf;
        long res = node.Line?.Eval(x) ?? Inf;
        if (lo == hi) return res;
        long mid = lo + (hi - lo) / 2;
        return Math.Min(res, x <= mid
            ? Query(node.Left,  lo,    mid, x)
            : Query(node.Right, mid+1, hi,  x));
    }

    public void Add(long m, long b)  => AddLine(root, xLo, xHi, new Line(m, b));
    public long Query(long x)        => Query(root, xLo, xHi, x);
}

public class Program {
    public static void Main() {
        // D&C DP
        var dnc = new DnCDP(8, 3, (i, j) => (long)(j-i)*(j-i));
        dnc.Run();
        Console.WriteLine("D&C DP initialized");

        // Knuth DP
        var knuth = new KnuthDP(5, (i, j) => j - i);
        Console.WriteLine($"Knuth DP result = {knuth.Solve()}");

        // CHT
        var cht = new CHT();
        long[] a = {10,8,6,4,2,1}, b = {1,2,3,4,5,6};
        long[] dp = new long[6]; dp[0] = 0;
        cht.Add(a[0], dp[0]);
        for (int j = 1; j < 6; j++) {
            dp[j] = cht.Query(b[j]);
            cht.Add(a[j], dp[j]);
        }
        Console.WriteLine($"CHT DP final = {dp[5]}");

        // Li Chao Tree
        var lct = new LiChaoTree(-1000, 1000);
        lct.Add(3, 1); lct.Add(-2, 10); lct.Add(1, -5);
        Console.WriteLine($"Li Chao min at x=2:  {lct.Query(2)}");   // -3
        Console.WriteLine($"Li Chao min at x=5:  {lct.Query(5)}");   // 0
        Console.WriteLine($"Li Chao min at x=-3: {lct.Query(-3)}");  // -8
    }
}
```

---

## Choosing the Right Optimization

```
Does cost decompose as m[i]*x[j] + b[i]?
    YES → Convex Hull Trick / Li Chao Tree
          Slopes monotone + queries monotone? → CHT with deque, O(n)
          Slopes monotone + queries arbitrary? → CHT with binary search, O(n log n)
          Arbitrary slopes and queries?        → Li Chao Tree, O(n log n)

    NO → Does recurrence have form dp[i][j] = min_k(dp[i][k] + dp[k][j] + cost(i,j))?
              YES + quadrangle inequality? → Knuth O(n^2)

         Does opt[i][j] satisfy opt[i][j] <= opt[i][j+1]?
              YES → Divide & Conquer, O(n log n) per layer
```

---

## Quadrangle Inequality Verification

For Knuth to apply, verify QI on the cost function:

```
cost(a,c) + cost(b,d) <= cost(a,d) + cost(b,c)   for a <= b <= c <= d
```

Common cost functions satisfying QI:

- `cost(i,j) = w[j] - w[i]` (prefix sums of positive weights)
- `cost(i,j) = (j - i)^2` — NOT QI in general; depends on context
- `cost(i,j) = sum of elements in [i+1, j]` — satisfies QI if elements non-negative
- `cost(i,j) = frequency * log(frequency)` — entropy cost for optimal BST

---

## Pitfalls

- **D&C: monotonicity of opt must be verified** — if `opt[i][j] <= opt[i][j+1]` does not hold for the specific cost function, D&C produces wrong answers silently. Test on small inputs against a brute-force O(n^2) baseline before submitting.
- **Knuth: opt table boundary** — `opt[i][j-1]` requires `j > i+1` and `opt[i+1][j]` requires `i+1 < j`. When `j = i+1` (base case), the range collapses to a single candidate. Initialize `opt[i][i] = i` and handle length-2 intervals before the main loop.
- **CHT: slope ordering** — the deque-based CHT requires lines added in strictly decreasing slope order (for minimum hull). If slopes are increasing, reverse the convention or use `max` instead of `min`. Mixing orders without validation produces a malformed hull that passes small tests but fails on larger inputs.
- **CHT: integer overflow in bad() check** — the cross-multiplication `(b3-b1)*(m1-m2)` overflows 64-bit integers when values are large. Use `__int128` in C++ or `decimal` / `BigInteger` in C# for the comparison. Alternatively, use floating-point with a small epsilon, accepting rare errors.
- **CHT: monotone pointer never resets** — the `ptr` in the monotone query CHT must start at 0 and only advance. If the DP iterates over multiple independent queries that are not globally increasing, either use binary search or a fresh CHT per query group.
- **Li Chao Tree: x-domain must be fixed at construction** — the tree is built over `[x_lo, x_hi]`. Querying outside this range gives wrong results. When x-coordinates are unknown at construction time, use coordinate compression or a dynamic segment tree.
- **Knuth O(n^2) space** — storing `dp[n][n]` and `opt[n][n]` uses O(n^2) memory. For `n = 5000`, this is 200 MB of `long` arrays. Use `int` for opt if indices fit, and verify memory limits before applying Knuth.
- **CHT for maximum** — to find maximum instead of minimum, negate all slopes and intercepts, query, then negate the result. Alternatively, maintain an upper convex hull by reversing the `bad()` inequality sign.

---

## Complexity Summary

| Technique | Naive | Optimized | Condition |
|---|---|---|---|
| Divide & Conquer | O(kn^2) | O(kn log n) | opt[i][j] monotone in j |
| Knuth | O(n^3) | O(n^2) | Quadrangle inequality + monotonicity |
| CHT (monotone) | O(n^2) | O(n) | Linear cost, monotone slopes & queries |
| CHT (binary search) | O(n^2) | O(n log n) | Linear cost, monotone slopes |
| Li Chao Tree | O(n^2) | O(n log n) | Linear cost, arbitrary |

---

## Conclusion

These three DP optimization techniques cover the most important structural classes of DP recurrences encountered in competitive programming and algorithm design:

- **Divide & Conquer** applies when the optimal transition point is monotone — a single condition, easy to verify, and the implementation is a clean recursive wrapper around the standard transition.
- **Knuth's optimization** applies to interval DP with a quadrangle-inequality cost — reduces O(n^3) to O(n^2) with a two-line modification to the loop bounds.
- **Convex Hull Trick** applies when the transition decomposes into a linear function — reduces O(n^2) to O(n) or O(n log n) by maintaining a geometric envelope of lines rather than scanning all previous states.

**Key takeaway:**  
Recognize the recurrence structure first. If the cost is linear in a previous-state variable, try CHT. If it satisfies QI, try Knuth. If neither but opt is monotone, try D&C. Applying the wrong optimization to a recurrence that does not satisfy its precondition produces wrong answers with no compile-time or runtime error — always verify the structural condition on small cases against brute force.
