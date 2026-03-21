# Weighted Union — DSU with Edge Weights

## Origin & Motivation

Weighted Union (also called Weighted DSU or DSU with potential) generalizes the parity DSU from GF(2) to arbitrary **abelian groups** — most commonly integers under addition. Each node `v` stores a **weight** (or potential) `w[v]` representing its value relative to the root of its component. This allows the structure to answer "what is the difference (or ratio, or XOR) between the values of u and v?" in O(α(n)) time under arbitrary online union operations.

The structure is essentially a DSU where edges carry group-valued labels, and path compression propagates these labels correctly along the compressed path. It is the most general single-component label-propagation structure achievable within the DSU framework.

Complexity: **O(α(n))** amortized per operation, **O(n)** space.

---

## Where It Is Used

- Difference constraint systems (offline, single connected component queries)
- Relative weight / potential queries in dynamic graphs
- Kruskal's algorithm generalization (weighted spanning structures)
- Competitive programming: "what is A[i] - A[j]?" with online merge operations
- Physics simulation: potential difference in electrical networks
- Database: relational constraint propagation

---

## Comparison with DSU Variants

| Variant | Weight domain | Operation | Query |
|---|---|---|---|
| Standard DSU | None | Union | Same component? |
| DSU with Parity | GF(2) | XOR | Parity of path |
| Weighted DSU | (Z, +) | Addition | Difference of potentials |
| Weighted DSU | (Q\*, ×) | Multiplication | Ratio of potentials |
| Weighted DSU | (GF(2)^k, XOR) | XOR per bit | Multi-bit parity |

All share the same augmented find / union structure — only the group operation changes.

---

## Core Idea

Each node `v` stores:
- `parent[v]` — parent in the DSU tree
- `rank[v]` — union-by-rank value (for roots)
- `weight[v]` — cumulative weight from `v` to its root: `val[v] - val[root]`

**Invariant:** `weight[v]` = `val[v] - val[root(v)]` where `val[v]` is the conceptual absolute value of node `v` (never stored explicitly).

### Find with path compression

```
find(v):
    if parent[v] == v:
        return (v, weight=0)
    (root, root_weight) = find(parent[v])
    weight[v] += weight[parent[v]]   // chain: weight[v] now = val[v] - val[root]
    parent[v] = root
    return (root, weight[v])
```

### Union with weight

To assert `val[v] - val[u] = w` (i.e., add edge u→v with difference w):

```
(ru, wu) = find(u)   // wu = val[u] - val[ru]
(rv, wv) = find(v)   // wv = val[v] - val[rv]

if ru == rv:
    // Check consistency: val[v] - val[u] should equal w
    // val[v] - val[u] = (val[v] - val[rv]) - (val[u] - val[ru])
    //                 = wv - wu   (since ru == rv)
    consistent = (wv - wu == w)
    return

// Merge rv under ru:
// We need: weight[rv] = val[rv] - val[ru]
// val[v] - val[u] = w
// wv = val[v] - val[rv]  =>  val[rv] = val[v] - wv
// wu = val[u] - val[ru]  =>  val[ru] = val[u] - wu
// val[rv] - val[ru] = (val[v] - wv) - (val[u] - wu)
//                   = w - wv + wu   (since val[v] - val[u] = w ... NO, reverse)
// val[v] - val[u] = w  =>  val[u] = val[v] - w
// weight[rv] = val[rv] - val[ru]
//            = (val[v] - wv) - (val[u] - wu)
//            = val[v] - wv - val[u] + wu
//            = w - wv + wu    (substituting val[v] - val[u] = w... careful with signs)
// Deriving cleanly:
//   val[v] = val[u] + w
//   val[rv] = val[v] - wv = val[u] + w - wv
//   val[ru] = val[u] - wu
//   weight[rv] = val[rv] - val[ru] = wu + w - wv
weight[rv] = wu + w - wv
parent[rv] = ru
```

**Query `val[v] - val[u]`:**

```
(ru, wu) = find(u)
(rv, wv) = find(v)
if ru != rv: return UNDEFINED
return wv - wu       // (val[v] - val[root]) - (val[u] - val[root])
```

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Initialize | O(n) | O(n) |
| Find (path compression) | O(α(n)) amortized | O(1) |
| Union with weight | O(α(n)) amortized | O(1) |
| Difference query | O(α(n)) amortized | O(1) |
| Total for m operations | O(m α(n)) | O(n) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// Weighted DSU over (Z, +):
// weight[v] = val[v] - val[root(v)]
// Supports:
//   unite(u, v, w)  — assert val[v] - val[u] = w
//   query(u, v)     — return val[v] - val[u] (or LLONG_MIN if different components)
struct WeightedDSU {
    vector<int>       parent, rnk;
    vector<long long> weight;  // weight[v] = val[v] - val[root]

    WeightedDSU(int n) : parent(n), rnk(n, 0), weight(n, 0) {
        iota(parent.begin(), parent.end(), 0);
    }

    // Returns {root, weight[v] relative to root}
    // Path compression updates weight[v] to be relative to the new root directly.
    pair<int, long long> find(int v) {
        if (parent[v] == v) return {v, 0};
        auto [root, _] = find(parent[v]);
        weight[v] += weight[parent[v]]; // chain: val[v] - val[root]
        parent[v] = root;
        return {root, weight[v]};
    }

    // Assert val[v] - val[u] = w.
    // Returns true if merged (different components).
    // Sets `consistent` to whether the constraint agrees with existing ones.
    bool unite(int u, int v, long long w, bool& consistent) {
        auto [ru, wu] = find(u); // wu = val[u] - val[ru]
        auto [rv, wv] = find(v); // wv = val[v] - val[rv]

        if (ru == rv) {
            consistent = (wv - wu == w);
            return false;
        }

        consistent = true;
        // weight[rv] = val[rv] - val[ru] = wu + w - wv
        long long w_rv = wu + w - wv;

        if (rnk[ru] < rnk[rv]) {
            // Attach ru under rv instead
            // weight[ru] = val[ru] - val[rv] = -w_rv
            swap(ru, rv);
            w_rv = -w_rv;
        }

        parent[rv] = ru;
        weight[rv] = w_rv;
        if (rnk[ru] == rnk[rv]) rnk[ru]++;

        return true;
    }

    // Returns val[v] - val[u], or LLONG_MIN if u and v are in different components.
    long long query(int u, int v) {
        auto [ru, wu] = find(u);
        auto [rv, wv] = find(v);
        if (ru != rv) return LLONG_MIN;
        return wv - wu;
    }

    bool same(int u, int v) {
        return find(u).first == find(v).first;
    }
};

// -------------------------------------------------------
// Weighted DSU over (Q*, x):  ratio queries
// weight[v] = val[v] / val[root]
// unite(u, v, r): assert val[v] / val[u] = r
// query(u, v):    return val[v] / val[u]
// -------------------------------------------------------
struct RatioDSU {
    vector<int>    parent, rnk;
    vector<double> weight; // weight[v] = val[v] / val[root]

    RatioDSU(int n) : parent(n), rnk(n, 0), weight(n, 1.0) {
        iota(parent.begin(), parent.end(), 0);
    }

    pair<int, double> find(int v) {
        if (parent[v] == v) return {v, 1.0};
        auto [root, _] = find(parent[v]);
        weight[v] *= weight[parent[v]]; // chain: val[v] / val[root]
        parent[v] = root;
        return {root, weight[v]};
    }

    // Assert val[v] / val[u] = r
    bool unite(int u, int v, double r, bool& consistent) {
        auto [ru, wu] = find(u); // wu = val[u] / val[ru]
        auto [rv, wv] = find(v); // wv = val[v] / val[rv]

        if (ru == rv) {
            consistent = (fabs(wv / wu - r) < 1e-9);
            return false;
        }

        consistent = true;
        // weight[rv] = val[rv] / val[ru]
        // val[v] / val[u] = r  =>  val[v] = r * val[u]
        // wv = val[v] / val[rv]  =>  val[rv] = val[v] / wv = r * val[u] / wv
        // wu = val[u] / val[ru]  =>  val[ru] = val[u] / wu
        // weight[rv] = val[rv] / val[ru] = (r * val[u] / wv) / (val[u] / wu)
        //            = r * wu / wv
        double w_rv = r * wu / wv;

        if (rnk[ru] < rnk[rv]) {
            swap(ru, rv);
            w_rv = 1.0 / w_rv;
        }

        parent[rv] = ru;
        weight[rv] = w_rv;
        if (rnk[ru] == rnk[rv]) rnk[ru]++;
        return true;
    }

    // Returns val[v] / val[u], or -1.0 if different components
    double query(int u, int v) {
        auto [ru, wu] = find(u);
        auto [rv, wv] = find(v);
        if (ru != rv) return -1.0;
        return wv / wu;
    }
};

// -------------------------------------------------------
// Application: Online difference constraint checker
// Given constraints A[i] - A[j] = d, detect contradictions.
// -------------------------------------------------------
struct DifferenceChecker {
    WeightedDSU dsu;
    bool consistent;

    DifferenceChecker(int n) : dsu(n), consistent(true) {}

    // Assert A[v] - A[u] = d. Returns false if contradiction.
    bool add(int u, int v, long long d) {
        bool ok;
        dsu.unite(u, v, d, ok);
        if (!ok) consistent = false;
        return ok;
    }

    // Returns A[v] - A[u] if determined, else LLONG_MIN
    long long diff(int u, int v) { return dsu.query(u, v); }

    bool is_consistent() const { return consistent; }
};

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
int main() {
    // Integer difference DSU
    {
        WeightedDSU dsu(5);
        bool ok;

        // A[1] - A[0] = 3
        dsu.unite(0, 1, 3, ok);
        printf("unite(0,1,3): merged=%s\n", ok?"yes":"no");

        // A[2] - A[1] = -1  =>  A[2] - A[0] = 2
        dsu.unite(1, 2, -1, ok);

        // A[3] - A[2] = 5   =>  A[3] - A[0] = 7
        dsu.unite(2, 3, 5, ok);

        printf("A[3] - A[0] = %lld\n", dsu.query(0, 3)); // 7
        printf("A[2] - A[1] = %lld\n", dsu.query(1, 2)); // -1

        // Consistent constraint: A[3] - A[0] = 7
        dsu.unite(0, 3, 7, ok);
        printf("Constraint A[3]-A[0]=7: consistent=%s\n", ok?"yes":"no"); // yes

        // Contradicting constraint: A[3] - A[0] = 99
        dsu.unite(0, 3, 99, ok);
        printf("Constraint A[3]-A[0]=99: consistent=%s\n", ok?"yes":"no"); // no

        // Disconnected nodes
        long long q = dsu.query(0, 4);
        printf("A[4] - A[0] = %s\n",
            q == LLONG_MIN ? "undefined (different components)" : to_string(q).c_str());
    }

    printf("\n");

    // Ratio DSU (LeetCode 399 — Evaluate Division style)
    {
        RatioDSU rdsu(4);
        bool ok;

        // A[1] / A[0] = 2.0
        rdsu.unite(0, 1, 2.0, ok);
        // A[2] / A[1] = 3.0  =>  A[2] / A[0] = 6.0
        rdsu.unite(1, 2, 3.0, ok);

        printf("A[2] / A[0] = %.2f\n", rdsu.query(0, 2)); // 6.00
        printf("A[0] / A[2] = %.2f\n", rdsu.query(2, 0)); // 0.17

        // Consistent ratio
        rdsu.unite(0, 2, 6.0, ok);
        printf("Constraint A[2]/A[0]=6: consistent=%s\n", ok?"yes":"no"); // yes

        // Contradicting ratio
        rdsu.unite(0, 2, 5.0, ok);
        printf("Constraint A[2]/A[0]=5: consistent=%s\n", ok?"yes":"no"); // no
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;

// -------------------------------------------------------
// Weighted DSU over (long, +)
// weight[v] = val[v] - val[root(v)]
// -------------------------------------------------------
public class WeightedDSU {
    private readonly int[]  parent, rank;
    private readonly long[] weight;

    public WeightedDSU(int n) {
        parent = new int[n];
        rank   = new int[n];
        weight = new long[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    public (int root, long w) Find(int v) {
        if (parent[v] == v) return (v, 0);
        var (root, _) = Find(parent[v]);
        weight[v] += weight[parent[v]];
        parent[v] = root;
        return (root, weight[v]);
    }

    // Assert val[v] - val[u] = w
    public bool Unite(int u, int v, long w, out bool consistent) {
        var (ru, wu) = Find(u);
        var (rv, wv) = Find(v);

        if (ru == rv) {
            consistent = (wv - wu == w);
            return false;
        }

        consistent = true;
        long wRv = wu + w - wv;

        if (rank[ru] < rank[rv]) {
            (ru, rv) = (rv, ru);
            wRv = -wRv;
        }

        parent[rv] = ru;
        weight[rv] = wRv;
        if (rank[ru] == rank[rv]) rank[ru]++;
        return true;
    }

    // val[v] - val[u]; long.MinValue if different components
    public long Query(int u, int v) {
        var (ru, wu) = Find(u);
        var (rv, wv) = Find(v);
        return ru == rv ? wv - wu : long.MinValue;
    }

    public bool Same(int u, int v) => Find(u).root == Find(v).root;
}

// -------------------------------------------------------
// Ratio DSU over (double, *)
// weight[v] = val[v] / val[root(v)]
// -------------------------------------------------------
public class RatioDSU {
    private readonly int[]    parent, rank;
    private readonly double[] weight;

    public RatioDSU(int n) {
        parent = new int[n];
        rank   = new int[n];
        weight = new double[n];
        for (int i = 0; i < n; i++) { parent[i] = i; weight[i] = 1.0; }
    }

    public (int root, double w) Find(int v) {
        if (parent[v] == v) return (v, 1.0);
        var (root, _) = Find(parent[v]);
        weight[v] *= weight[parent[v]];
        parent[v] = root;
        return (root, weight[v]);
    }

    // Assert val[v] / val[u] = r
    public bool Unite(int u, int v, double r, out bool consistent) {
        var (ru, wu) = Find(u);
        var (rv, wv) = Find(v);

        if (ru == rv) {
            consistent = Math.Abs(wv / wu - r) < 1e-9;
            return false;
        }

        consistent = true;
        double wRv = r * wu / wv;

        if (rank[ru] < rank[rv]) {
            (ru, rv) = (rv, ru);
            wRv = 1.0 / wRv;
        }

        parent[rv] = ru;
        weight[rv] = wRv;
        if (rank[ru] == rank[rv]) rank[ru]++;
        return true;
    }

    // val[v] / val[u]; -1 if different components
    public double Query(int u, int v) {
        var (ru, wu) = Find(u);
        var (rv, wv) = Find(v);
        return ru == rv ? wv / wu : -1.0;
    }
}

// -------------------------------------------------------
// Online difference constraint checker
// -------------------------------------------------------
public class DifferenceChecker {
    private readonly WeightedDSU dsu;
    public bool Consistent { get; private set; } = true;

    public DifferenceChecker(int n) { dsu = new WeightedDSU(n); }

    public bool Add(int u, int v, long d) {
        bool ok;
        dsu.Unite(u, v, d, out ok);
        if (!ok) Consistent = false;
        return ok;
    }

    public long Diff(int u, int v) => dsu.Query(u, v);
}

public class Program {
    public static void Main() {
        // Integer weighted DSU
        var dsu = new WeightedDSU(5);

        dsu.Unite(0, 1,  3, out _);  // A[1]-A[0] = 3
        dsu.Unite(1, 2, -1, out _);  // A[2]-A[1] = -1
        dsu.Unite(2, 3,  5, out _);  // A[3]-A[2] = 5

        Console.WriteLine($"A[3]-A[0] = {dsu.Query(0, 3)}"); // 7
        Console.WriteLine($"A[2]-A[1] = {dsu.Query(1, 2)}"); // -1

        dsu.Unite(0, 3, 7, out bool ok1);
        Console.WriteLine($"Constraint A[3]-A[0]=7: consistent={ok1}");  // True
        dsu.Unite(0, 3, 99, out bool ok2);
        Console.WriteLine($"Constraint A[3]-A[0]=99: consistent={ok2}"); // False

        long q = dsu.Query(0, 4);
        Console.WriteLine($"A[4]-A[0]: {(q == long.MinValue ? "undefined" : q.ToString())}");

        Console.WriteLine();

        // Ratio DSU
        var rdsu = new RatioDSU(3);
        rdsu.Unite(0, 1, 2.0, out _);  // A[1]/A[0] = 2
        rdsu.Unite(1, 2, 3.0, out _);  // A[2]/A[1] = 3

        Console.WriteLine($"A[2]/A[0] = {rdsu.Query(0, 2):F2}"); // 6.00
        Console.WriteLine($"A[0]/A[2] = {rdsu.Query(2, 0):F6}"); // 0.166667

        rdsu.Unite(0, 2, 6.0, out bool rok1);
        Console.WriteLine($"Constraint A[2]/A[0]=6: consistent={rok1}"); // True
        rdsu.Unite(0, 2, 5.0, out bool rok2);
        Console.WriteLine($"Constraint A[2]/A[0]=5: consistent={rok2}"); // False
    }
}
```

---

## Weight Derivation — Step by Step

The union weight formula is the most error-prone part. Deriving it formally for `unite(u, v, w)` asserting `val[v] - val[u] = w`:

```
Let:
  wu = weight[u after find] = val[u] - val[ru]
  wv = weight[v after find] = val[v] - val[rv]

We attach rv under ru. Need: weight[rv] = val[rv] - val[ru]

val[ru] = val[u] - wu         (from definition of wu)
val[rv] = val[v] - wv         (from definition of wv)
val[v]  = val[u] + w          (the constraint being added)

weight[rv] = val[rv] - val[ru]
           = (val[v] - wv) - (val[u] - wu)
           = val[v] - wv - val[u] + wu
           = (val[u] + w) - wv - val[u] + wu
           = w + wu - wv
```

For ratio DSU, replace subtraction with division and addition with multiplication:

```
weight[rv] = val[rv] / val[ru]
           = (val[v] / wv) / (val[u] / wu)
           = (val[v] / val[u]) * (wu / wv)
           = r * wu / wv
```

---

## Pitfalls

- **Reading parent's weight before updating parent pointer** — identical to parity DSU: `weight[v] += weight[parent[v]]` must execute before `parent[v] = root`. In an iterative implementation, collect the full path first, then propagate weights backward from root to leaf.
- **Sign flip on union rank swap** — when `rank[ru] < rank[rv]` and we attach ru under rv instead, the weight stored is `val[ru] - val[rv] = -weight[rv_original]`. Forgetting the sign flip produces a structurally valid tree with incorrect weights. For ratio DSU, the flip is `1.0 / w` not `-w`.
- **LLONG_MIN as sentinel** — `query` returns `LLONG_MIN` for disconnected nodes. Arithmetic on this sentinel (e.g., `query(u,v) + 1`) silently overflows. Check for `LLONG_MIN` or use `optional<long long>` before using the result.
- **Floating-point ratio DSU is approximate** — each find accumulates multiplicative errors. On long chains before compression, error compounds. After compression, it stabilizes. Use exact rational arithmetic (`__int128` numerator/denominator pairs) for problems requiring exact ratio answers.
- **Difference constraint system is not complete** — Weighted DSU handles single-component difference queries in O(α(n)). For a **full** difference constraint system (detect infeasibility across all components with all constraints), use Bellman-Ford. Weighted DSU only detects contradictions within a merged component, not globally until the relevant nodes are united.
- **Consistency flag is per-constraint, not global** — `consistent` returned by `unite` reflects only whether the new constraint contradicts existing ones. If the caller ignores a `consistent=false` return and continues merging, the DSU structure remains valid (it does not update on contradiction), but subsequent queries on that component may return wrong values if a different merge later connects the same nodes.
- **Non-abelian groups break path compression** — the weight propagation `weight[v] += weight[parent[v]]` assumes the operation is commutative and associative. For non-abelian groups (matrix multiplication, permutations), path compression cannot be applied without storing the full sequence — use a different structure.

---

## Conclusion

Weighted DSU is the **canonical structure for online potential / difference / ratio queries under dynamic merging**:

- Extends standard DSU with zero asymptotic overhead — O(α(n)) per operation.
- Supports any abelian group as the weight domain: differences (Z, +), ratios (Q\*, ×), XOR (GF(2)^k, ⊕).
- Handles online constraint addition and contradiction detection in a unified framework.
- The parity DSU is exactly Weighted DSU instantiated over GF(2).

**Key takeaway:**  
When a problem asks "given a dynamic graph where edges carry values and we need to query the cumulative value between two nodes", Weighted DSU is the answer if the value domain is an abelian group. Derive the union weight formula from first principles using the invariant `weight[v] = val[v] OP_inverse val[root]` — the formula for integers and ratios follows mechanically from this definition.
