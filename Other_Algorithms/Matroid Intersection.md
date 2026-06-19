# Matroid Intersection

## Origin & Motivation

**Matroid Intersection** is one of the deepest results in combinatorial optimisation. Given two matroids `M₁ = (E, I₁)` and `M₂ = (E, I₂)` over the same ground set `E`, the problem is to find the largest common independent set `S ∈ I₁ ∩ I₂`.

The algorithm was discovered by Lawler in 1975, building on the matroid theory of Whitney (1935) and the exchange axioms formalised by Edmonds (1970).

**The problem it solves:** Many combinatorial problems that seem unrelated — bipartite matching, arborescences, colourful spanning trees, common bases of two matroids — all reduce to matroid intersection. The remarkable fact is that while finding the maximum independent set in a single matroid is trivially solvable by greedy algorithms, the intersection of two matroids requires a non-trivial augmenting-path algorithm. The intersection of three or more matroids is NP-hard in general.

**Why matroids?** A matroid is an abstraction of linear independence that captures exactly the structure where greedy algorithms work. The exchange axiom (if `A, B ∈ I` and `|A| < |B|`, then there exists `x ∈ B \ A` such that `A ∪ {x} ∈ I`) is what makes greedy provably optimal for single-matroid problems.

Complexity: **O(r² · |E| · T)** where `r` is the rank of the intersection and `T` is the time to check independence in one matroid. For partition matroids `T = O(|E|)`; for graphic matroids `T = O(|E| · α(|E|))`.

---

## Where It Is Used

- **Bipartite matching** — the intersection of two partition matroids (at most 1 edge per left node AND at most 1 edge per right node) is exactly a matching
- **Arborescences** — directed spanning trees rooted at a fixed vertex
- **Colourful spanning forests** — find a spanning forest using at most `k` edges of each colour (intersection of graphic matroid with partition matroid)
- **Common bases** — find a base that is simultaneously in two matroids (e.g. a spanning tree that is also an independent set of another matroid)
- **Scheduling** — scheduling jobs on machines where each matroid encodes a different resource constraint
- **Competitive programming** — problems asking for a maximum set satisfying two independent "exchange-type" constraints simultaneously

---

## Core Concepts

### Matroid Definition

A **matroid** `M = (E, I)` consists of:
- A finite ground set `E`
- A family of **independent sets** `I ⊆ 2^E`

satisfying three axioms:
```
(I1) ∅ ∈ I                                        (empty set is independent)
(I2) If A ∈ I and B ⊆ A, then B ∈ I              (downward closed / hereditary)
(I3) If A, B ∈ I and |A| < |B|, then              (exchange axiom)
     ∃ x ∈ B \ A such that A ∪ {x} ∈ I
```

The **rank** of a matroid is the size of its largest independent set (any two maximal independent sets have the same size, by the exchange axiom).

### Common Matroid Types

```
Uniform matroid U(k, n): any ≤ k elements from an n-element ground set.
  I = { S ⊆ E : |S| ≤ k }

Partition matroid: ground set partitioned into groups G₁,...,Gₜ;
  at most kᵢ elements from each group Gᵢ.
  I = { S : |S ∩ Gᵢ| ≤ kᵢ for all i }

Graphic matroid M(G): edges of graph G; forests are independent.
  I = { F ⊆ E(G) : F is a forest (acyclic) }

Linear matroid: columns of a matrix over a field; linearly independent subsets.
  I = { S : columns in S are linearly independent }

Transversal matroid: system of sets A₁,...,Aₙ; a set S is independent
  if it can be matched to distinct sets (partial system of distinct representatives).
```

### The Exchange Graph

For the current common independent set `S ∈ I₁ ∩ I₂`, define the **exchange graph** `D_S = (E, A)`:

```
For each x ∈ E \ S and y ∈ S:
  arc (x → y) if (S - y + x) ∈ I₁   [x can replace y in M₁]
  arc (y → x) if (S - y + x) ∈ I₂   [x can replace y in M₂]

Special sets:
  X₁ = { x ∈ E \ S : S + x ∈ I₁ }   [x can be freely added to M₁]
  X₂ = { x ∈ E \ S : S + x ∈ I₂ }   [x can be freely added to M₂]
```

**An augmenting path** is a path from some node in `X₁` to some node in `X₂` in `D_S`. Applying the symmetric difference of `S` with the nodes on this path gives a new common independent set of size `|S| + 1`.

---

## Algorithm

```
MatroidIntersection(E, M₁, M₂):
    S = ∅

    loop:
        Compute X₁ = { x ∈ E\S : S ∪ {x} ∈ I₁ }
        Compute X₂ = { x ∈ E\S : S ∪ {x} ∈ I₂ }

        Build exchange graph D_S:
            for x ∈ E\S, y ∈ S:
                if (S - y + x) ∈ I₁: add arc x → y
                if (S - y + x) ∈ I₂: add arc y → x

        Find shortest path P from X₁ to X₂ in D_S (BFS)

        if no path exists:
            return S        // S is maximum common independent set

        // Augment: symmetric difference
        S = S △ V(P)        // elements on P: toggle in/out of S
```

**Correctness:** The algorithm is correct by the matroid intersection theorem (Edmonds 1970): `S` is maximum if and only if no augmenting path exists in `D_S`.

**Termination:** Each augmentation increases `|S|` by 1. Since `|S| ≤ rank(M₁)` and `|S| ≤ rank(M₂)`, the algorithm terminates in at most `min(rank(M₁), rank(M₂))` iterations.

---

## Weighted Matroid Intersection

The weighted version finds the maximum-weight common independent set `S ∈ I₁ ∩ I₂` with maximum `Σ w(e)` for `e ∈ S`.

```
Weighted algorithm:
    Same exchange graph D_S, but instead of BFS (shortest path by hops),
    use Bellman-Ford (or Dijkstra with potentials) to find the
    minimum-weight augmenting path (where arc weights encode the
    weight changes from swapping elements in/out).

    Arc weights in D_S:
        arc x → y (x replaces y in M₁): weight = -w(x) + w(y)  [remove y, add x]
        arc y → x (x replaces y in M₂): weight = -w(x) + w(y)  [same]
        entry arc (source → x for x ∈ X₁): weight = -w(x)
        exit arc  (x → sink for x ∈ X₂):  weight = 0

    Find minimum-weight path (augmenting path that maximises added weight).
    O(r · |E| · (|E| + r log r)) with Dijkstra + Johnson potentials.
```

---

## Complexity Summary

| Algorithm | Complexity | Notes |
|---|---|---|
| Unweighted matroid intersection | O(r² · |E| · T) | T = independence oracle time, r = rank |
| Weighted matroid intersection | O(r · |E| · (r + |E|) · T) | Bellman-Ford based augmentation |
| Graphic matroid independence check | O(|E| · α(n)) | Union-Find |
| Partition matroid independence check | O(|E|) | Count per group |
| Linear matroid independence check | O(r²) | Gaussian elimination |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// MATROID BASE CLASS
// ================================================================
struct Matroid {
    virtual bool independent(const vector<int>& S) = 0;
    virtual ~Matroid() = default;
};

// ================================================================
// GRAPHIC MATROID — forests in graph G are independent sets
// ================================================================
struct GraphicMatroid : Matroid {
    int n;
    vector<pair<int,int>> edges;

    GraphicMatroid(int n, vector<pair<int,int>> edges) : n(n), edges(edges) {}

    bool independent(const vector<int>& S) override {
        vector<int> par(n); iota(par.begin(), par.end(), 0);
        function<int(int)> find = [&](int x)->int {
            return par[x]==x ? x : par[x]=find(par[x]);
        };
        for (int eid : S) {
            auto [u,v] = edges[eid];
            int pu=find(u), pv=find(v);
            if (pu==pv) return false; // cycle detected
            par[pu] = pv;
        }
        return true;
    }
};

// ================================================================
// PARTITION MATROID — at most k[g] elements from group g
// ================================================================
struct PartitionMatroid : Matroid {
    int m;
    vector<int> group, k;

    PartitionMatroid(int m, vector<int> group, vector<int> k)
        : m(m), group(group), k(k) {}

    bool independent(const vector<int>& S) override {
        vector<int> cnt(k.size(), 0);
        for (int e : S) {
            if (++cnt[group[e]] > k[group[e]]) return false;
        }
        return true;
    }
};

// ================================================================
// UNIFORM MATROID — any ≤ k elements are independent
// ================================================================
struct UniformMatroid : Matroid {
    int k;
    UniformMatroid(int k) : k(k) {}
    bool independent(const vector<int>& S) override { return (int)S.size() <= k; }
};

// ================================================================
// LINEAR MATROID over GF(2) — linearly independent columns
// ================================================================
struct LinearMatroidGF2 : Matroid {
    int rows;
    vector<vector<int>> cols; // cols[i] = column i as binary vector of length `rows`

    LinearMatroidGF2(int rows, vector<vector<int>> cols) : rows(rows), cols(cols) {}

    bool independent(const vector<int>& S) override {
        // Gaussian elimination over GF(2)
        vector<int> basis(rows, -1); // basis[r] = index of pivot for row r
        for (int eid : S) {
            vector<int> col = cols[eid];
            for (int r = 0; r < rows; r++) {
                if (!col[r]) continue;
                if (basis[r] == -1) { basis[r] = eid; break; }
                for (int rr = 0; rr < rows; rr++) col[rr] ^= cols[basis[r]][rr];
            }
            bool zero = true;
            for (int r = 0; r < rows; r++) if (col[r]) { zero=false; break; }
            if (zero) return false; // linearly dependent
        }
        return true;
    }
};

// ================================================================
// MATROID INTERSECTION — augmenting path algorithm (Lawler 1975)
// Returns the maximum common independent set S ∈ I₁ ∩ I₂.
// ================================================================
vector<int> matroid_intersection(int m, Matroid& M1, Matroid& M2) {
    vector<bool> inS(m, false);
    vector<int> S;

    while (true) {
        // BFS to find shortest augmenting path from X₁ to X₂
        // prev[e] = the element we came from (-1 = came from virtual source)
        vector<int> prev(m, -2); // -2 = unvisited
        queue<int> q;

        // Seed BFS: elements x ∉ S where S+x ∈ I₁ (these are in X₁)
        for (int x = 0; x < m; x++) {
            if (inS[x]) continue;
            vector<int> Spx = S; Spx.push_back(x);
            if (M1.independent(Spx)) {
                prev[x] = -1; // came from virtual source
                q.push(x);
            }
        }

        bool found = false;
        int end_node = -1;

        while (!q.empty() && !found) {
            int u = q.front(); q.pop();

            if (!inS[u]) {
                // u ∈ E \ S: check if u ∈ X₂ (S + u ∈ I₂) → augmenting path found
                vector<int> Spu = S; Spu.push_back(u);
                if (M2.independent(Spu)) {
                    found = true; end_node = u;
                    break;
                }
                // Arcs x → y in D_S: for each y ∈ S, arc (u→y) if (S-y+u) ∈ I₂
                for (int y : S) {
                    if (prev[y] != -2) continue;
                    vector<int> Smy_pu;
                    for (int e : S) if (e != y) Smy_pu.push_back(e);
                    Smy_pu.push_back(u);
                    if (M2.independent(Smy_pu)) {
                        prev[y] = u;
                        q.push(y);
                    }
                }
            } else {
                // u ∈ S: arcs y → x in D_S: for each x ∉ S, arc (u→x) if (S-u+x) ∈ I₁
                int y = u;
                for (int x = 0; x < m; x++) {
                    if (inS[x] || prev[x] != -2) continue;
                    vector<int> Smy_px;
                    for (int e : S) if (e != y) Smy_px.push_back(e);
                    Smy_px.push_back(x);
                    if (M1.independent(Smy_px)) {
                        prev[x] = y;
                        q.push(x);
                    }
                }
            }
        }

        if (!found) break; // S is maximum: no augmenting path exists

        // Augment: trace back path from end_node and toggle elements in/out of S
        int cur = end_node;
        while (cur != -1) {
            if (inS[cur]) {
                inS[cur] = false;
                S.erase(find(S.begin(), S.end(), cur));
            } else {
                inS[cur] = true;
                S.push_back(cur);
            }
            cur = prev[cur];
        }
    }
    return S;
}

// ================================================================
// Usage + tests
// ================================================================
int main() {
    // ---- Bipartite matching via partition × partition ----
    {
        printf("=== Bipartite Matching (Partition × Partition) ===\n");
        // 3 left nodes, 3 right nodes; edges: e0=(0,3), e1=(0,4), e2=(1,3), e3=(1,5), e4=(2,4), e5=(2,5)
        int m = 6;
        vector<pair<int,int>> edges = {{0,3},{0,4},{1,3},{1,5},{2,4},{2,5}};
        vector<int> lg = {0,0,1,1,2,2}; // left node of each edge
        vector<int> rg = {0,1,0,2,1,2}; // right node - 3 (0-indexed)
        vector<int> kl(3,1), kr(3,1);
        PartitionMatroid M1(m,lg,kl); // at most 1 edge per left node
        PartitionMatroid M2(m,rg,kr); // at most 1 edge per right node

        auto S = matroid_intersection(m, M1, M2);
        printf("Max matching = %d (expect 3)\n", (int)S.size());
        printf("Edges: ");
        for (int e:S) printf("(%d-%d) ", edges[e].first, edges[e].second);
        printf("\n");
    }

    // ---- Colourful spanning forest (graphic × partition) ----
    {
        printf("\n=== Colourful Spanning Forest (Graphic × Partition) ===\n");
        // Graph on 5 nodes, edges have colours; at most 2 edges of each colour
        int n_nodes = 5;
        vector<pair<int,int>> edges = {{0,1},{0,2},{1,2},{1,3},{2,3},{3,4},{2,4}};
        vector<int> colour =          {  0,    0,    1,    1,    2,    2,    0  };
        int m = (int)edges.size();
        vector<int> k_col(3, 2); // at most 2 of each colour

        GraphicMatroid    M1(n_nodes, edges);
        PartitionMatroid  M2(m, colour, k_col);

        auto S = matroid_intersection(m, M1, M2);
        printf("Max colourful forest = %d edges\n", (int)S.size());
        printf("Edges: ");
        for (int e:S) printf("(%d-%d,col%d) ", edges[e].first, edges[e].second, colour[e]);
        printf("\n");
        // Verify
        bool ok = M1.independent(S) && M2.independent(S);
        printf("Valid: %s\n", ok?"YES":"NO");
    }

    // ---- Graphic matroid ∩ Uniform matroid ----
    {
        printf("\n=== Graphic × Uniform Matroid ===\n");
        int n_nodes = 5;
        vector<pair<int,int>> edges = {{0,1},{0,2},{1,2},{1,3},{2,3},{3,4}};
        int m = (int)edges.size();
        GraphicMatroid M1(n_nodes, edges);
        UniformMatroid M2(3); // at most 3 edges

        auto S = matroid_intersection(m, M1, M2);
        printf("Max forest with ≤3 edges: %d (expect 3)\n", (int)S.size());
        bool ok = M1.independent(S) && M2.independent(S);
        printf("Valid: %s\n", ok?"YES":"NO");
    }

    // ---- Stress: partition × partition vs brute force ----
    {
        printf("\n=== Stress: Partition × Partition, 500 trials ===\n");
        srand(42); int errors=0;
        for (int trial=0; trial<500; trial++) {
            int nl=2+rand()%4, nr=2+rand()%4;
            int m=0; vector<int> lg, rg;
            for (int i=0;i<nl;i++) for (int j=0;j<nr;j++) if (rand()%2) {
                lg.push_back(i); rg.push_back(j); m++;
            }
            if (m==0) continue;
            vector<int> kl(nl,1), kr(nr,1);
            PartitionMatroid M1(m,lg,kl);
            PartitionMatroid M2(m,rg,kr);

            int got=(int)matroid_intersection(m,M1,M2).size();

            // Brute force
            int exp=0;
            for (int mask=0;mask<(1<<m);mask++){
                vector<int>S; for(int i=0;i<m;i++) if(mask>>i&1)S.push_back(i);
                if (M1.independent(S)&&M2.independent(S)) exp=max(exp,(int)S.size());
            }
            if (got!=exp) errors++;
        }
        printf("Result: %s\n",errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- Stress: graphic × uniform ----
    {
        printf("\n=== Stress: Graphic × Uniform, 300 trials ===\n");
        srand(99); int errors=0;
        for (int trial=0; trial<300; trial++) {
            int n_nodes=3+rand()%4;
            vector<pair<int,int>> edges;
            for (int i=0;i<n_nodes;i++) for (int j=i+1;j<n_nodes;j++)
                if (rand()%2) edges.push_back({i,j});
            int m=(int)edges.size(); if (m<2||m>15) continue;
            int k=1+rand()%min(m,n_nodes-1);
            GraphicMatroid M1(n_nodes,edges);
            UniformMatroid M2(k);

            int got=(int)matroid_intersection(m,M1,M2).size();
            int exp=0;
            for (int mask=0;mask<(1<<m);mask++){
                vector<int>S; for(int i=0;i<m;i++) if(mask>>i&1)S.push_back(i);
                if (M1.independent(S)&&M2.independent(S)) exp=max(exp,(int)S.size());
            }
            if (got!=exp) errors++;
        }
        printf("Result: %s\n",errors==0?"OK":("FAIL "+to_string(errors)).c_str());
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
// MATROID INTERFACE
// ================================================================
public interface IMatroid
{
    bool Independent(List<int> S);
}

// ================================================================
// GRAPHIC MATROID — forests in graph G
// ================================================================
public class GraphicMatroid : IMatroid
{
    private readonly int _n;
    private readonly (int u, int v)[] _edges;

    public GraphicMatroid(int n, (int u, int v)[] edges) { _n=n; _edges=edges; }

    public bool Independent(List<int> S)
    {
        var par = Enumerable.Range(0, _n).ToArray();
        int Find(int x) { while (par[x]!=x) { par[x]=par[par[x]]; x=par[x]; } return x; }
        foreach (int eid in S)
        {
            int pu=Find(_edges[eid].u), pv=Find(_edges[eid].v);
            if (pu==pv) return false;
            par[pu]=pv;
        }
        return true;
    }
}

// ================================================================
// PARTITION MATROID — at most k[g] elements from group g
// ================================================================
public class PartitionMatroid : IMatroid
{
    private readonly int[] _group, _k;

    public PartitionMatroid(int[] group, int[] k) { _group=group; _k=k; }

    public bool Independent(List<int> S)
    {
        var cnt = new int[_k.Length];
        foreach (int e in S) if (++cnt[_group[e]] > _k[_group[e]]) return false;
        return true;
    }
}

// ================================================================
// UNIFORM MATROID — any ≤ k elements
// ================================================================
public class UniformMatroid : IMatroid
{
    private readonly int _k;
    public UniformMatroid(int k) { _k=k; }
    public bool Independent(List<int> S) => S.Count <= _k;
}

// ================================================================
// MATROID INTERSECTION
// ================================================================
public static class MatroidIntersection
{
    public static List<int> Solve(int m, IMatroid M1, IMatroid M2)
    {
        var inS = new bool[m];
        var S = new List<int>();

        while (true)
        {
            var prev = Enumerable.Repeat(-2, m).ToArray();
            var q = new Queue<int>();

            // Seed BFS with X₁ = { x ∉ S : S+x ∈ I₁ }
            for (int x=0; x<m; x++)
            {
                if (inS[x]) continue;
                var Spx = new List<int>(S) { x };
                if (M1.Independent(Spx)) { prev[x]=-1; q.Enqueue(x); }
            }

            bool found=false; int endNode=-1;

            while (q.Count>0 && !found)
            {
                int u=q.Dequeue();
                if (!inS[u])
                {
                    var Spu = new List<int>(S) { u };
                    if (M2.Independent(Spu)) { found=true; endNode=u; break; }

                    foreach (int y in S)
                    {
                        if (prev[y]!=-2) continue;
                        var tmp = S.Where(e=>e!=y).Append(u).ToList();
                        if (M2.Independent(tmp)) { prev[y]=u; q.Enqueue(y); }
                    }
                }
                else
                {
                    int y=u;
                    for (int x=0; x<m; x++)
                    {
                        if (inS[x]||prev[x]!=-2) continue;
                        var tmp = S.Where(e=>e!=y).Append(x).ToList();
                        if (M1.Independent(tmp)) { prev[x]=y; q.Enqueue(x); }
                    }
                }
            }

            if (!found) break;

            int cur=endNode;
            while (cur!=-1)
            {
                if (inS[cur]) { inS[cur]=false; S.Remove(cur); }
                else           { inS[cur]=true;  S.Add(cur);    }
                cur=prev[cur];
            }
        }
        return S;
    }

    public static void Main()
    {
        // Bipartite matching
        var lg = new[]{0,0,1,1,2,2};
        var rg = new[]{0,1,0,2,1,2};
        var M1=new PartitionMatroid(lg,new[]{1,1,1});
        var M2=new PartitionMatroid(rg,new[]{1,1,1});
        var S=Solve(6,M1,M2);
        Console.WriteLine($"Bipartite matching size = {S.Count} (expect 3)");

        // Graphic × Uniform
        var edges = new (int,int)[]{(0,1),(0,2),(1,2),(1,3),(2,3),(3,4)};
        var Mg=new GraphicMatroid(5,edges);
        var Mu=new UniformMatroid(3);
        var S2=Solve(edges.Length,Mg,Mu);
        Console.WriteLine($"Graphic ∩ Uniform(3) = {S2.Count} (expect 3)");
    }
}
```

---

## Bipartite Matching as Matroid Intersection

Bipartite matching is the most celebrated special case of matroid intersection. Given a bipartite graph `G = (L ∪ R, E)`:

```
Ground set E = edge set of G

M₁ = Partition Matroid with:
    Group(e) = left endpoint of e
    k[v] = 1 for every left node v
    (At most 1 edge incident to each left node)

M₂ = Partition Matroid with:
    Group(e) = right endpoint of e
    k[v] = 1 for every right node v
    (At most 1 edge incident to each right node)

S ∈ I₁ ∩ I₂  ⟺  S is a matching in G

Maximum matching = maximum common independent set of M₁ and M₂
```

The matroid intersection algorithm on this instance reduces to the standard **augmenting-path algorithm for bipartite matching** (König-Egerváry theorem), running in O(m√n) with Hopcroft-Karp. The matroid intersection algorithm runs in O(r² · m) which is slower for pure bipartite matching, but handles the general case where no special structure can be exploited.

---

## Pitfalls

- **Intersection of ≥ 3 matroids is NP-hard** — the algorithm only works for exactly two matroids. If a problem requires membership in three independent set families simultaneously, matroid intersection does not apply unless one of the three families can be embedded into one of the two matroid constraints.
- **The independence oracle must be called with the ACTUAL proposed set** — a common mistake is to check `M.independent(S ∪ {x})` by incrementally testing if x can be added rather than rebuilding from scratch. For matroids that support incremental updates this is fine, but the naive `independent(vector)` function must test the full proposed set, not just the new element.
- **Augmenting path must be traced carefully in both directions** — elements on the augmenting path alternate between `E\S` and `S`. When toggling: elements in `S` are removed, elements in `E\S` are added. The path starts and ends at elements in `E\S` (from `X₁` to `X₂`), so the path length is always odd and `|S|` increases by exactly 1.
- **Exchange graph arc directions** — `arc(x→y)` exists when `(S - y + x) ∈ I₁` and `arc(y→x)` when `(S - y + x) ∈ I₂`. Swapping which matroid governs which arc direction produces an incorrect algorithm that may return a non-independent set.
- **BFS gives the SHORTEST augmenting path** — using DFS instead of BFS still finds an augmenting path and gives a correct algorithm, but may produce unnecessarily long paths. BFS minimises the number of independence oracle calls per augmentation and is standard.
- **Weighted version needs Bellman-Ford, not BFS** — the unweighted algorithm uses BFS (shortest path by hops). The weighted version uses Bellman-Ford/SPFA over the exchange graph with arc weights derived from element weights. Using BFS for the weighted version finds augmenting paths but does not guarantee maximum weight.

---

## Conclusion

Matroid Intersection is one of the most powerful tools in polyhedral combinatorics and polynomial-time optimisation:

- The **exchange graph** captures exactly which element swaps preserve independence in both matroids simultaneously. An augmenting path in this graph corresponds to a sequence of safe swaps that increases the common independent set size by 1.
- **Every bipartite matching problem**, colourful spanning tree problem, and common base problem reduces to matroid intersection — making it a universal reduction target for "find a maximum set satisfying two hereditary independence constraints".
- The algorithm's correctness rests on the matroid intersection theorem: the maximum size of a common independent set equals the minimum, over all partitions `(A, B)` of `E`, of `r₁(A) + r₂(B)` (a minimax theorem analogous to König's theorem for bipartite matching).
- Intersecting three or more matroids is NP-hard in general, making two-matroid intersection a remarkable boundary between polynomial and NP-hard optimisation.

**Key takeaway:** implement a `bool independent(vector<int> S)` oracle for each matroid, then call `matroid_intersection(m, M1, M2)`. The algorithm makes `O(r²·|E|)` oracle calls total, where `r` is the size of the optimal common independent set — the entire complexity reduction to the oracle means you only need to implement independence checking correctly, and the rest is generic.
