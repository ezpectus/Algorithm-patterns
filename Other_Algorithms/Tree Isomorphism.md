# Tree Isomorphism

## Origin & Motivation

Two graphs G and H are **isomorphic** if there exists a bijection between their vertex sets that preserves all edges. For general graphs, isomorphism is a notoriously hard problem (GI-complete — not known to be in P or NP-complete). However, for **trees**, isomorphism can be decided in **O(n log n)** or even **O(n)** time.

The canonical approach: assign each tree a **canonical hash** (or canonical label) that is equal for two trees if and only if they are isomorphic. The hash is computed bottom-up: leaf hashes are identical, and an internal node's hash is determined by the sorted multiset of its children's hashes. Sorting children before hashing ensures the canonical form is invariant under reordering.

For **rooted trees**: straightforward bottom-up hashing. For **unrooted trees**: find the **centroid(s)** first (every tree has 1 or 2 centroids), root the tree at the centroid, then hash. Two unrooted trees are isomorphic iff their centroid-rooted canonical hashes match.

Complexity: **O(n log n)** using string hashing or AHU algorithm.

---

## Where It Is Used

- Checking if two parse trees / ASTs are structurally identical
- Detecting duplicate subtrees in compiler optimization
- Graph database queries (subgraph isomorphism on trees)
- Competitive programming: comparing tree structures, counting isomorphic subtrees
- Chemical informatics: comparing molecular structures (trees of bonds)
- Phylogenetics: comparing evolutionary trees

---

## Key Concepts

**Rooted tree isomorphism:** Two rooted trees (T1, r1) and (T2, r2) are isomorphic iff there exists a bijection mapping r1 to r2 and preserving all parent-child relationships.

**Canonical form:** A unique string/integer representation of a tree such that two trees have the same canonical form iff they are isomorphic. For rooted trees:
```
canonical(leaf)    = "0"  or  hash(sorted({}) + id)
canonical(v)       = hash(sorted([canonical(child) for child in children(v)]))
```

**Centroid:** A vertex whose removal leaves no component with more than n/2 vertices. Every tree has 1 or 2 centroids. Used to root unrooted trees canonically.

**AHU Algorithm (Aho, Hopcroft, Ullman, 1974):** O(n) tree isomorphism via integer labeling — assigns integer labels bottom-up where same label ↔ same canonical subtree structure.

---

## Algorithm Variants

### Rooted Tree Isomorphism (O(n log n))

1. Compute canonical hash for each vertex bottom-up
2. Two rooted trees are isomorphic iff their root hashes are equal

### Unrooted Tree Isomorphism (O(n log n))

1. Find centroid(s) of both trees
2. Root each tree at its centroid
3. Compute canonical hash
4. If a tree has 2 centroids, its canonical hash = combination of both centroid-rooted hashes
5. Compare hashes

### AHU Algorithm — O(n) with integer labels

1. Process leaves first (all get label 0)
2. Sort children's labels (multiset)
3. Map each new multiset to a new integer label (using a map)
4. Two trees are isomorphic iff their root labels are equal after processing

---

## Centroid Finding

The centroid is the vertex minimizing the maximum component size after its removal. Equivalently, it is the vertex where `max(size[child], n - size[v]) <= n/2` for all children.

```
Find centroid:
    Root tree at any vertex, compute subtree sizes
    For each vertex v:
        max_comp = max(n - size[v], max over children of size[child])
        if max_comp <= n/2: v is a centroid
```

Every tree has exactly 1 or 2 centroids. If 2 centroids: they are adjacent.

---

## Complexity Analysis

| Algorithm | Time | Space | Notes |
|---|---|---|---|
| Naive (string canonical form) | O(n log² n) | O(n log n) | Sort strings |
| Hashing (polynomial) | O(n log n) | O(n) | Sort hash arrays |
| AHU (integer labels) | O(n log n) | O(n) | Due to sorting |
| AHU with radix sort | O(n) | O(n) | Optimal |
| Subtree count (all n subtrees) | O(n log n) | O(n) | Hash all subtrees |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// ROOTED TREE ISOMORPHISM — canonical hashing
// Assigns each subtree a unique integer label bottom-up.
// Two subtrees have the same label iff they are isomorphic.
// ================================================================
struct RootedTreeIso {
    // Global label map: sorted vector of child labels -> new label
    map<vector<int>, int> label_map;
    int next_label = 0;

    int get_label(vector<int> children_labels) {
        sort(children_labels.begin(), children_labels.end());
        auto it = label_map.find(children_labels);
        if (it != label_map.end()) return it->second;
        label_map[children_labels] = next_label;
        return next_label++;
    }

    // Compute canonical label for each vertex in rooted tree
    // adj[v] = list of children (parent already excluded by caller)
    // Returns label of root
    int label(int v, int parent, const vector<vector<int>>& adj,
              vector<int>& labels) {
        vector<int> child_labels;
        for (int u : adj[v]) {
            if (u == parent) continue;
            child_labels.push_back(label(u, v, adj, labels));
        }
        labels[v] = get_label(child_labels);
        return labels[v];
    }

    // Check if two rooted trees are isomorphic
    bool isomorphic_rooted(
        int root1, const vector<vector<int>>& adj1,
        int root2, const vector<vector<int>>& adj2)
    {
        int n1 = adj1.size(), n2 = adj2.size();
        if (n1 != n2) return false;

        vector<int> lab1(n1), lab2(n2);
        int l1 = label(root1, -1, adj1, lab1);
        int l2 = label(root2, -1, adj2, lab2);
        return l1 == l2;
    }
};

// ================================================================
// CENTROID FINDING
// ================================================================
int find_centroid(int v, int parent, int n,
                  vector<vector<int>>& adj,
                  vector<int>& sz) {
    sz[v] = 1;
    int heavy = 0;
    for (int u : adj[v]) {
        if (u == parent) continue;
        sz[v] += sz[find_centroid(u, v, n, adj, sz)];
        heavy = max(heavy, sz[u]);
    }
    heavy = max(heavy, n - sz[v]); // component above v
    if (heavy <= n / 2) return v;
    for (int u : adj[v]) {
        if (u == parent) continue;
        if (sz[u] > n / 2) return find_centroid(u, v, n, adj, sz);
    }
    return v;
}

// Returns centroid(s): 1 or 2 vertices
vector<int> centroids(int n, vector<vector<int>>& adj) {
    vector<int> sz(n, 0);
    vector<int> result;
    // DFS to compute sizes rooted at 0
    function<void(int,int)> dfs_sz = [&](int v, int p) {
        sz[v] = 1;
        for (int u : adj[v]) if (u != p) { dfs_sz(u,v); sz[v] += sz[u]; }
    };
    dfs_sz(0, -1);

    // Find all centroids
    function<void(int,int)> find = [&](int v, int p) {
        int heavy = n - sz[v]; // component above
        for (int u : adj[v]) if (u != p) heavy = max(heavy, sz[u]);
        if (heavy <= n / 2) result.push_back(v);
        for (int u : adj[v]) if (u != p) find(u, v);
    };
    find(0, -1);
    return result;
}

// ================================================================
// UNROOTED TREE ISOMORPHISM
// Root at centroid(s), compare canonical hashes.
// ================================================================
bool trees_isomorphic(int n1, vector<vector<int>> adj1,
                      int n2, vector<vector<int>> adj2)
{
    if (n1 != n2) return false;
    if (n1 == 1) return true; // both are single vertices

    RootedTreeIso iso;

    auto get_hash = [&](int n, vector<vector<int>>& adj,
                         vector<int> roots) -> vector<int> {
        vector<int> hashes;
        vector<int> lab(n);
        for (int r : roots) {
            iso.label(r, -1, adj, lab);
            hashes.push_back(lab[r]);
        }
        sort(hashes.begin(), hashes.end());
        return hashes;
    };

    auto c1 = centroids(n1, adj1);
    auto c2 = centroids(n2, adj2);

    // Number of centroids must match
    if (c1.size() != c2.size()) return false;

    auto h1 = get_hash(n1, adj1, c1);
    auto h2 = get_hash(n2, adj2, c2);
    return h1 == h2;
}

// ================================================================
// COUNT DISTINCT SUBTREE TYPES in a single tree
// Assigns labels to all subtrees, returns frequency map
// ================================================================
map<int,int> count_subtree_types(int root, int n,
                                  vector<vector<int>>& adj)
{
    RootedTreeIso iso;
    vector<int> labels(n);
    iso.label(root, -1, adj, labels);
    map<int,int> freq;
    for (int v = 0; v < n; v++) freq[labels[v]]++;
    return freq;
}

// ================================================================
// CHECK IF TREE T2 APPEARS AS A SUBTREE IN T1
// ================================================================
bool subtree_exists(int n1, vector<vector<int>>& adj1, int root1,
                    int n2, vector<vector<int>>& adj2, int root2)
{
    RootedTreeIso iso;

    // Get canonical label of T2's root
    vector<int> lab2(n2);
    int target = iso.label(root2, -1, adj2, lab2);

    // Label all subtrees of T1
    vector<int> lab1(n1);
    iso.label(root1, -1, adj1, lab1);

    // Check if any subtree of T1 has the same label
    for (int v = 0; v < n1; v++)
        if (lab1[v] == target) return true;
    return false;
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Test 1: Two isomorphic trees
    //   T1:  0-1, 0-2, 1-3, 1-4  (rooted at 0)
    //   T2:  0-1, 0-2, 2-3, 2-4  (rooted at 0)
    // Both are: root with two children, one child has 2 leaves
    {
        printf("=== Rooted Tree Isomorphism ===\n");
        int n = 5;
        vector<vector<int>> T1(n), T2(n);
        T1[0]={1,2}; T1[1]={0,3,4}; T1[2]={0}; T1[3]={1}; T1[4]={1};
        T2[0]={1,2}; T2[1]={0};     T2[2]={0,3,4}; T2[3]={2}; T2[4]={2};

        RootedTreeIso iso;
        printf("T1 rooted at 0 ≅ T2 rooted at 0: %s\n",
               iso.isomorphic_rooted(0, T1, 0, T2) ? "YES" : "NO"); // YES
    }

    // Test 2: Non-isomorphic trees
    //   T1: path of 4 nodes  0-1-2-3 (rooted at 0)
    //   T2: star of 4 nodes  0-{1,2,3} (rooted at 0)
    {
        printf("\n=== Non-isomorphic trees ===\n");
        int n = 4;
        vector<vector<int>> T1(n), T2(n);
        T1[0]={1}; T1[1]={0,2}; T1[2]={1,3}; T1[3]={2};
        T2[0]={1,2,3}; T2[1]={0}; T2[2]={0}; T2[3]={0};

        RootedTreeIso iso;
        printf("Path(4) ≅ Star(4): %s\n",
               iso.isomorphic_rooted(0, T1, 0, T2) ? "YES" : "NO"); // NO
    }

    // Test 3: Unrooted tree isomorphism
    //   T1 and T2 both = path of 5 nodes, presented with different labeling
    {
        printf("\n=== Unrooted Tree Isomorphism ===\n");
        int n = 5;
        // T1: 0-1-2-3-4 (path)
        vector<vector<int>> T1(n);
        T1[0]={1}; T1[1]={0,2}; T1[2]={1,3}; T1[3]={2,4}; T1[4]={3};

        // T2: 4-2-0-3-1 (same path, relabeled)
        vector<vector<int>> T2(n);
        T2[4]={2}; T2[2]={4,0}; T2[0]={2,3}; T2[3]={0,1}; T2[1]={3};

        printf("Path(5) isomorphic to relabeled Path(5): %s\n",
               trees_isomorphic(n, T1, n, T2) ? "YES" : "NO"); // YES

        // T3: star K_{1,4}
        vector<vector<int>> T3(n);
        T3[0]={1,2,3,4}; T3[1]={0}; T3[2]={0}; T3[3]={0}; T3[4]={0};
        printf("Path(5) isomorphic to Star(5): %s\n",
               trees_isomorphic(n, T1, n, T3) ? "YES" : "NO"); // NO
    }

    // Test 4: Count distinct subtree types
    {
        printf("\n=== Count Subtree Types ===\n");
        //     0
        //    /|\
        //   1 2 3
        //  /|
        // 4 5
        int n = 6;
        vector<vector<int>> T(n);
        T[0]={1,2,3}; T[1]={0,4,5}; T[2]={0}; T[3]={0};
        T[4]={1}; T[5]={1};

        auto freq = count_subtree_types(0, n, T);
        printf("Subtree type frequencies:\n");
        for (auto [label, cnt] : freq)
            printf("  type=%d count=%d\n", label, cnt);
        // Leaves (4,5,2,3) should all have the same type (cnt=4)
        // Vertex 1 (root of subtree with 2 leaves) gets its own type
        // Vertex 0 gets its own type
    }

    // Test 5: Find if T2 appears as subtree in T1
    {
        printf("\n=== Subtree containment ===\n");
        int n1=6, n2=3;
        vector<vector<int>> T1(n1), T2(n2);
        T1[0]={1,2,3}; T1[1]={0,4,5}; T1[2]={0}; T1[3]={0}; T1[4]={1}; T1[5]={1};
        // T2: root with 2 leaf children
        T2[0]={1,2}; T2[1]={0}; T2[2]={0};

        printf("T2 (root+2leaves) is subtree of T1: %s\n",
               subtree_exists(n1, T1, 0, n2, T2, 0) ? "YES" : "NO"); // YES
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class TreeIsomorphism {
    private readonly Dictionary<List<int>, int> labelMap =
        new(new ListComparer());
    private int nextLabel = 0;

    private int GetLabel(List<int> childLabels) {
        childLabels.Sort();
        if (labelMap.TryGetValue(childLabels, out int lbl)) return lbl;
        labelMap[childLabels] = nextLabel;
        return nextLabel++;
    }

    public int Label(int v, int parent, List<int>[] adj, int[] labels) {
        var childLabels = new List<int>();
        foreach (int u in adj[v]) {
            if (u == parent) continue;
            childLabels.Add(Label(u, v, adj, labels));
        }
        return labels[v] = GetLabel(childLabels);
    }

    public bool IsomorphicRooted(
        int root1, List<int>[] adj1,
        int root2, List<int>[] adj2)
    {
        if (adj1.Length != adj2.Length) return false;
        int n = adj1.Length;
        var l1 = new int[n]; var l2 = new int[n];
        return Label(root1,-1,adj1,l1) == Label(root2,-1,adj2,l2);
    }

    // Find centroids of undirected tree
    public static List<int> Centroids(int n, List<int>[] adj) {
        var sz  = new int[n];
        var res = new List<int>();

        void DfsSz(int v, int p) {
            sz[v] = 1;
            foreach (int u in adj[v]) if (u!=p) { DfsSz(u,v); sz[v]+=sz[u]; }
        }
        DfsSz(0, -1);

        void Find(int v, int p) {
            int heavy = n - sz[v];
            foreach (int u in adj[v]) if (u!=p) heavy = Math.Max(heavy, sz[u]);
            if (heavy <= n/2) res.Add(v);
            foreach (int u in adj[v]) if (u!=p) Find(u, v);
        }
        Find(0, -1);
        return res;
    }

    public bool IsomorphicUnrooted(
        int n1, List<int>[] adj1,
        int n2, List<int>[] adj2)
    {
        if (n1 != n2) return false;
        if (n1 == 1) return true;

        var c1 = Centroids(n1, adj1);
        var c2 = Centroids(n2, adj2);
        if (c1.Count != c2.Count) return false;

        List<int> GetHashes(int n, List<int>[] adj, List<int> roots) {
            var hashes = new List<int>();
            var lab = new int[n];
            foreach (int r in roots) { Label(r,-1,adj,lab); hashes.Add(lab[r]); }
            hashes.Sort();
            return hashes;
        }

        var h1 = GetHashes(n1, adj1, c1);
        var h2 = GetHashes(n2, adj2, c2);
        if (h1.Count != h2.Count) return false;
        for (int i = 0; i < h1.Count; i++) if (h1[i] != h2[i]) return false;
        return true;
    }

    public static void Main() {
        var iso = new TreeIsomorphism();

        // Rooted: both trees = root + one child-with-two-leaves + one leaf
        int n = 5;
        var T1 = new List<int>[n];
        var T2 = new List<int>[n];
        for (int i=0;i<n;i++) { T1[i]=new(); T2[i]=new(); }
        T1[0].AddRange(new[]{1,2}); T1[1].AddRange(new[]{0,3,4}); T1[2].Add(0); T1[3].Add(1); T1[4].Add(1);
        T2[0].AddRange(new[]{1,2}); T2[1].Add(0);     T2[2].AddRange(new[]{0,3,4}); T2[3].Add(2); T2[4].Add(2);
        Console.WriteLine($"Rooted iso: {iso.IsomorphicRooted(0,T1,0,T2)}"); // True

        // Unrooted: Path(5) vs relabeled Path(5)
        iso = new TreeIsomorphism();
        var P1 = new List<int>[5]; for(int i=0;i<5;i++) P1[i]=new();
        P1[0].Add(1); P1[1].AddRange(new[]{0,2}); P1[2].AddRange(new[]{1,3}); P1[3].AddRange(new[]{2,4}); P1[4].Add(3);
        var P2 = new List<int>[5]; for(int i=0;i<5;i++) P2[i]=new();
        P2[4].Add(2); P2[2].AddRange(new[]{4,0}); P2[0].AddRange(new[]{2,3}); P2[3].AddRange(new[]{0,1}); P2[1].Add(3);
        Console.WriteLine($"Path(5) unrooted iso: {iso.IsomorphicUnrooted(5,P1,5,P2)}"); // True

        var Star = new List<int>[5]; for(int i=0;i<5;i++) Star[i]=new();
        Star[0].AddRange(new[]{1,2,3,4}); Star[1].Add(0); Star[2].Add(0); Star[3].Add(0); Star[4].Add(0);
        Console.WriteLine($"Path(5) vs Star(5): {iso.IsomorphicUnrooted(5,P1,5,Star)}"); // False
    }

    // Equality comparer for List<int> keys in Dictionary
    private class ListComparer : IEqualityComparer<List<int>> {
        public bool Equals(List<int>? a, List<int>? b) {
            if (a == null || b == null) return a == b;
            if (a.Count != b.Count) return false;
            for (int i = 0; i < a.Count; i++) if (a[i] != b[i]) return false;
            return true;
        }
        public int GetHashCode(List<int> a) {
            int h = 17;
            foreach (int x in a) h = h * 31 + x;
            return h;
        }
    }
}
```

---

## AHU Algorithm — Integer Labeling Detail

The AHU algorithm assigns integer labels in a single bottom-up pass using a global map from sorted child-label tuples to new integers:

```
Level 0 (leaves): all leaves get label 0
Level 1: vertices whose all children are leaves
    For a vertex with 3 leaf children: label = map[{0,0,0}]
    For a vertex with 2 leaf children: label = map[{0,0}]
    etc.
Level 2: process next level up
    ...

Final: root label encodes entire tree structure
```

Two trees are isomorphic iff their roots get the same final label. The map ensures that the same multiset of child labels always produces the same parent label — making labels canonical.

**Key property:** The map is shared across both trees being compared. This means labels from T1 and T2 are directly comparable — no need for separate canonicalization.

---

## Centroid — Why It Works for Unrooted Isomorphism

Every tree has 1 or 2 centroids:
- **1 centroid:** There is a unique center of balance. Root both trees at their centroids and compare.
- **2 centroids:** The two centroids are adjacent. Each centroid roots "half" the tree. Canonical form = sorted pair of the two centroid-rooted hashes. Two trees with 2 centroids are isomorphic iff their sorted centroid-hash pairs match.

```
If tree has 1 centroid c:
  canonical(T) = label(T rooted at c)

If tree has 2 centroids c1, c2:
  canonical(T) = sorted({label(T rooted at c1), label(T rooted at c2)})
```

This is valid because: if T1 ≅ T2, the isomorphism must map centroid(s) to centroid(s) (centroids are structural invariants), so the centroid-rooted canonical forms will match.

---

## Pitfalls

- **Sort child labels before hashing** — the canonical form requires children to be processed in a canonical order. Without sorting, two trees that differ only in the order their children are listed would get different labels despite being isomorphic. Always sort the child label list before looking it up in the map.
- **Unrooted: handle 1 vs 2 centroids** — trees with 1 centroid and trees with 2 centroids must be handled differently. Two trees where one has 1 centroid and the other has 2 centroids are never isomorphic — check centroid counts first. For 2-centroid trees, the canonical hash must be the sorted pair of both centroid-rooted hashes, not just one of them.
- **Label map must be shared between both trees** — when comparing T1 and T2, use the same `label_map` instance for both. If separate maps are used, label integers assigned to T1 and T2 are independent and not comparable even for identical structures.
- **Size check before centroid/hash** — if `n1 != n2`, the trees are trivially non-isomorphic. Check this before any computation. Also check that both have the same number of edges (= n-1 for trees, so this is equivalent to n equality).
- **Subtree containment uses rooted comparison only** — when checking if T2 appears as a rooted subtree of T1, root T2 at its root and compare against all subtree labels in T1. This only checks rooted subtree containment. For unrooted subtree containment, the problem is more complex and requires checking all possible rootings of T2.
- **Hash collisions in polynomial hashing** — if using polynomial hash instead of the map-based AHU label, collisions can cause false positives. Use double hashing (two independent polynomials with different primes) or the map-based approach for correctness. Map-based AHU is always collision-free.

---

## Complexity Summary

| Approach | Rooted | Unrooted | Space |
|---|---|---|---|
| AHU (map-based) | O(n log n) | O(n log n) | O(n) |
| AHU (radix sort) | O(n) | O(n) | O(n) |
| Polynomial hash | O(n log n) | O(n log n) | O(n) |
| String canonical | O(n log² n) | O(n log² n) | O(n log n) |
| Naive brute force | O(n!) | O(n!) | O(n) |

---

## Conclusion

Tree isomorphism is **one of the rare graph problems with an efficient exact algorithm** — O(n log n) vs NP-hardness for general graph isomorphism:

- **Rooted tree isomorphism:** Canonical bottom-up labeling. Two rooted trees are isomorphic iff they share the same root label. Straightforward AHU implementation.
- **Unrooted tree isomorphism:** Find centroid(s) first, then apply rooted isomorphism. Two unrooted trees are isomorphic iff their centroid-rooted canonical forms match.
- **Subtree counting and containment:** The same labeling assigns equal labels to all isomorphic subtrees, enabling O(n log n) preprocessing for O(1) subtree type queries.

**Key takeaway:**  
Tree isomorphism reduces to sorting. The entire algorithm is: compute subtree labels bottom-up, sort child labels at each node, look up in a shared map. The centroid is the only non-trivial step — but centroid finding is itself O(n). With a shared label map between the two trees being compared, the final answer is a single integer comparison at the root(s).
