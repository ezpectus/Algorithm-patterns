# Van Emde Boas Tree

## Origin & Motivation

The Van Emde Boas (vEB) tree was introduced by Peter van Emde Boas in 1975. It is a data structure for maintaining a dynamic set of integers from a bounded universe `{0, 1, ..., U-1}` and supports all standard priority queue operations — insert, delete, successor, predecessor, min, max — in **O(log log U)** time.

This beats the O(log n) bound of balanced BSTs when U is polynomial in n, and even beats O(log n / log log n) for many practical universe sizes. The key insight is recursive decomposition of the universe: split the u-bit universe into √u "clusters" of size √u each. A top-level summary tracks which clusters are non-empty. This halves the problem size at each recursion level, giving O(log log U) depth.

Complexity: **O(log log U)** per operation, **O(U)** space (or O(n log log U) with hashing).

---

## Where It Is Used

- Integer priority queues over bounded universes
- Network routing (IP address lookups, next-hop computation)
- Computational geometry (x-coordinate successor queries)
- Integer sorting in O(n log log U) via repeated extraction of min
- Competitive programming: fast successor/predecessor on integers
- Operating systems: memory page management, process scheduling

---

## Key Operations

| Operation | Time | Description |
|---|---|---|
| `insert(x)` | O(log log U) | Add integer x to the set |
| `delete(x)` | O(log log U) | Remove integer x |
| `successor(x)` | O(log log U) | Smallest element > x |
| `predecessor(x)` | O(log log U) | Largest element < x |
| `min()` | O(1) | Smallest element |
| `max()` | O(1) | Largest element |
| `member(x)` | O(log log U) | Is x in the set? |

---

## Structure

A vEB tree over universe `{0, ..., U-1}` where `U = 2^k` stores:

- `min`, `max` — minimum and maximum elements (or -∞/+∞ if empty). **The min is NOT stored in any cluster recursively** — this is the key trick enabling O(1) min/max.
- `summary` — a vEB tree over `{0, ..., √U - 1}` tracking which clusters are non-empty
- `cluster[0..√U-1]` — √U sub-vEB trees each over universe `{0, ..., √U - 1}`

**Universe splitting:** For x in `{0..U-1}`:
```
high(x) = x / √U = x >> (k/2)   // cluster index
low(x)  = x % √U = x & (√U-1)   // position within cluster
index(i, j) = i * √U + j         // reconstruct from (cluster, position)
```

**Base cases:** For U = 2 (1-bit universe), store just min and max (a 2-element set). No clusters needed.

---

## Algorithm: Insert

```
insert(V, x):
    if V is empty:
        V.min = V.max = x
        return

    if x < V.min: swap(x, V.min)   // x becomes new min, old min recurses down
    if x > V.max: V.max = x

    if U > 2:                        // non-base case
        h = high(x), l = low(x)
        if V.cluster[h] is empty:
            insert(V.summary, h)     // mark cluster h as non-empty
        insert(V.cluster[h], l)      // insert l into cluster h
```

**Why O(log log U):** At most one recursive call to `insert` (the other, into summary, only happens when the cluster was empty, and then the cluster insert hits the base case immediately). So T(U) = T(√U) + O(1), giving T(U) = O(log log U).

---

## Algorithm: Successor

```
successor(V, x):
    if U == 2:
        if x == 0 and V.max == 1: return 1
        return ∞

    if x < V.min: return V.min     // min is always the answer if x < min

    h = high(x), l = low(x)
    max_low = V.cluster[h].max     // max in x's cluster

    if max_low > l:
        // successor is in same cluster
        return index(h, successor(V.cluster[h], l))
    else:
        // successor is in a later cluster
        next_h = successor(V.summary, h)
        if next_h == ∞: return ∞
        return index(next_h, V.cluster[next_h].min)
```

---

## Algorithm: Delete

```
delete(V, x):
    if V.min == V.max:
        V.min = V.max = -∞     // set becomes empty
        return

    if U == 2:
        V.min = V.max = (x == 0) ? 1 : 0
        return

    if x == V.min:
        // Find new min: it's in the first non-empty cluster
        first_cluster = V.summary.min
        x = index(first_cluster, V.cluster[first_cluster].min)
        V.min = x    // update min

    h = high(x), l = low(x)
    delete(V.cluster[h], l)
    if V.cluster[h].min == -∞:   // cluster became empty
        delete(V.summary, h)
        if x == V.max:
            // Update max
            max_h = V.summary.max
            if max_h == -∞: V.max = V.min
            else:           V.max = index(max_h, V.cluster[max_h].max)
    elif x == V.max:
        V.max = index(h, V.cluster[h].max)
```

---

## Complexity Analysis

| Operation | Recurrence | Complexity |
|---|---|---|
| Insert | T(U) = T(√U) + O(1) | O(log log U) |
| Delete | T(U) = T(√U) + O(1) | O(log log U) |
| Successor | T(U) = T(√U) + O(1) | O(log log U) |
| Predecessor | T(U) = T(√U) + O(1) | O(log log U) |
| Min / Max | O(1) | O(1) |
| Space (array) | — | O(U) |
| Space (hash map) | — | O(n log log U) |

The recurrence T(U) = T(√U) + O(1): substituting U = 2^{2^k}, T(2^{2^k}) = T(2^{2^{k-1}}) + O(1) = ... = O(k) = O(log log U).

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// VAN EMDE BOAS TREE — Recursive implementation
// Universe: {0, ..., U-1} where U is a power of 2
// ================================================================
struct VEB {
    int U;          // universe size
    int lo_size;    // sqrt(U) = lower half size
    int hi_size;    // sqrt(U) = upper half size (same for power-of-2 U)
    int min_val, max_val; // -1 = empty
    VEB* summary;
    vector<VEB*> cluster;

    static const int EMPTY = -1;

    VEB(int U) : U(U), min_val(EMPTY), max_val(EMPTY),
        summary(nullptr)
    {
        if (U <= 2) return; // base case: no clusters needed
        lo_size = 1;
        while (lo_size * lo_size < U) lo_size <<= 1;
        hi_size = U / lo_size;

        summary = new VEB(hi_size);
        cluster.assign(hi_size, nullptr);
    }

    ~VEB() {
        delete summary;
        for (VEB* c : cluster) delete c;
    }

    bool empty() const { return min_val == EMPTY; }

    int high(int x) const { return x / lo_size; }
    int low(int x)  const { return x % lo_size; }
    int idx(int h, int l) const { return h * lo_size + l; }

    VEB* get_cluster(int h) {
        if (!cluster[h]) cluster[h] = new VEB(lo_size);
        return cluster[h];
    }

    void insert(int x) {
        if (empty()) { min_val = max_val = x; return; }

        if (x < min_val) swap(x, min_val);
        if (x > max_val) max_val = x;

        if (U <= 2) return;

        int h = high(x), l = low(x);
        if (get_cluster(h)->empty())
            summary->insert(h);
        get_cluster(h)->insert(l);
    }

    void del(int x) {
        if (min_val == max_val) { min_val = max_val = EMPTY; return; }

        if (U == 2) {
            min_val = max_val = (x == 0) ? 1 : 0;
            return;
        }

        if (x == min_val) {
            int fc = summary->min_val;
            x = idx(fc, cluster[fc]->min_val);
            min_val = x;
        }

        int h = high(x), l = low(x);
        get_cluster(h)->del(l);

        if (get_cluster(h)->empty()) {
            summary->del(h);
            if (x == max_val) {
                int mh = summary->max_val;
                if (mh == EMPTY) max_val = min_val;
                else             max_val = idx(mh, cluster[mh]->max_val);
            }
        } else if (x == max_val) {
            max_val = idx(h, cluster[h]->max_val);
        }
    }

    int successor(int x) const {
        if (U == 2) {
            if (x == 0 && max_val == 1) return 1;
            return -1; // no successor
        }

        if (!empty() && x < min_val) return min_val;

        int h = high(x), l = low(x);
        int max_low = (cluster[h] && !cluster[h]->empty())
                      ? cluster[h]->max_val : EMPTY;

        if (max_low != EMPTY && l < max_low) {
            int nl = cluster[h]->successor(l);
            return idx(h, nl);
        } else {
            int nh = summary->successor(h);
            if (nh == EMPTY) return EMPTY;
            return idx(nh, cluster[nh]->min_val);
        }
    }

    int predecessor(int x) const {
        if (U == 2) {
            if (x == 1 && min_val == 0) return 0;
            return EMPTY;
        }

        if (!empty() && x > max_val) return max_val;

        int h = high(x), l = low(x);
        int min_low = (cluster[h] && !cluster[h]->empty())
                      ? cluster[h]->min_val : EMPTY;

        if (min_low != EMPTY && l > min_low) {
            int pl = cluster[h]->predecessor(l);
            return idx(h, pl);
        } else {
            int ph = summary->predecessor(h);
            if (ph == EMPTY) {
                if (!empty() && x > min_val) return min_val;
                return EMPTY;
            }
            return idx(ph, cluster[ph]->max_val);
        }
    }

    bool member(int x) const {
        if (x == min_val || x == max_val) return true;
        if (U == 2 || empty()) return false;
        int h = high(x);
        if (!cluster[h]) return false;
        return cluster[h]->member(low(x));
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // Universe size U = 16 (4-bit integers: 0..15)
    VEB veb(16);

    // Insert elements
    for (int x : {2, 3, 7, 11, 14, 5}) {
        veb.insert(x);
    }

    printf("=== Van Emde Boas Tree (U=16) ===\n");
    printf("Inserted: {2, 3, 5, 7, 11, 14}\n");
    printf("Min: %d\n", veb.min_val); // 2
    printf("Max: %d\n", veb.max_val); // 14

    // Successor queries
    printf("\nSuccessor queries:\n");
    for (int x : {1, 2, 3, 6, 10, 13, 14}) {
        int s = veb.successor(x);
        printf("  succ(%d) = %d\n", x, s);
    }

    // Predecessor queries
    printf("\nPredecessor queries:\n");
    for (int x : {3, 5, 8, 12, 14, 15}) {
        int p = veb.predecessor(x);
        printf("  pred(%d) = %d\n", x, p);
    }

    // Member queries
    printf("\nMember queries:\n");
    for (int x : {2, 4, 7, 9, 14}) {
        printf("  %d in set: %s\n", x, veb.member(x) ? "yes" : "no");
    }

    // Delete
    printf("\nDelete 7 and 11:\n");
    veb.del(7);
    veb.del(11);
    printf("Min: %d, Max: %d\n", veb.min_val, veb.max_val);
    printf("succ(5) = %d (expect 14)\n", veb.successor(5));

    // Integer sorting via repeated min-extraction: O(n log log U)
    {
        printf("\n=== Integer sort via vEB ===\n");
        VEB v(64);
        for (int x : {17, 3, 45, 8, 63, 21, 0, 9}) v.insert(x);
        printf("Sorted: ");
        while (!v.empty()) {
            int m = v.min_val;
            printf("%d ", m);
            v.del(m);
        }
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

public class VEB {
    private const int Empty = -1;

    private readonly int U, loSize, hiSize;
    public int Min { get; private set; } = Empty;
    public int Max { get; private set; } = Empty;

    private VEB? summary;
    private readonly VEB?[] cluster;

    public VEB(int U) {
        this.U = U;
        if (U <= 2) { loSize = hiSize = 1; cluster = Array.Empty<VEB?>(); return; }

        loSize = 1;
        while (loSize * loSize < U) loSize <<= 1;
        hiSize = U / loSize;

        summary = new VEB(hiSize);
        cluster = new VEB?[hiSize];
    }

    public bool IsEmpty => Min == Empty;

    private int High(int x) => x / loSize;
    private int Low(int x)  => x % loSize;
    private int Idx(int h, int l) => h * loSize + l;

    private VEB GetCluster(int h) =>
        cluster[h] ??= new VEB(loSize);

    public void Insert(int x) {
        if (IsEmpty) { Min = Max = x; return; }
        if (x < Min) (x, Min) = (Min, x);
        if (x > Max) Max = x;
        if (U <= 2) return;

        int h = High(x), l = Low(x);
        if (GetCluster(h).IsEmpty) summary!.Insert(h);
        GetCluster(h).Insert(l);
    }

    public void Delete(int x) {
        if (Min == Max) { Min = Max = Empty; return; }
        if (U == 2) { Min = Max = (x == 0) ? 1 : 0; return; }

        if (x == Min) {
            int fc = summary!.Min;
            x = Idx(fc, cluster[fc]!.Min);
            Min = x;
        }

        int h2 = High(x), l2 = Low(x);
        GetCluster(h2).Delete(l2);

        if (GetCluster(h2).IsEmpty) {
            summary!.Delete(h2);
            if (x == Max) {
                int mh = summary.Max;
                Max = mh == Empty ? Min : Idx(mh, cluster[mh]!.Max);
            }
        } else if (x == Max) {
            Max = Idx(h2, cluster[h2]!.Max);
        }
    }

    public int Successor(int x) {
        if (U == 2) return (x == 0 && Max == 1) ? 1 : Empty;
        if (!IsEmpty && x < Min) return Min;

        int h = High(x), l = Low(x);
        var cl = cluster[h];
        if (cl != null && !cl.IsEmpty && l < cl.Max) {
            return Idx(h, cl.Successor(l));
        }
        int nh = summary!.Successor(h);
        return nh == Empty ? Empty : Idx(nh, cluster[nh]!.Min);
    }

    public int Predecessor(int x) {
        if (U == 2) return (x == 1 && Min == 0) ? 0 : Empty;
        if (!IsEmpty && x > Max) return Max;

        int h = High(x), l = Low(x);
        var cl = cluster[h];
        if (cl != null && !cl.IsEmpty && l > cl.Min) {
            return Idx(h, cl.Predecessor(l));
        }
        int ph = summary!.Predecessor(h);
        if (ph != Empty) return Idx(ph, cluster[ph]!.Max);
        return (!IsEmpty && x > Min) ? Min : Empty;
    }

    public bool Member(int x) {
        if (x == Min || x == Max) return true;
        if (U == 2 || IsEmpty) return false;
        var cl = cluster[High(x)];
        return cl != null && cl.Member(Low(x));
    }

    public static void Main() {
        var veb = new VEB(16);
        foreach (int x in new[]{2,3,7,11,14,5}) veb.Insert(x);

        Console.WriteLine($"Min={veb.Min}, Max={veb.Max}");
        Console.WriteLine($"succ(6)={veb.Successor(6)}, succ(3)={veb.Successor(3)}");
        Console.WriteLine($"pred(8)={veb.Predecessor(8)}, pred(14)={veb.Predecessor(14)}");
        Console.WriteLine($"member(7)={veb.Member(7)}, member(6)={veb.Member(6)}");

        veb.Delete(7);
        Console.WriteLine($"After delete(7): succ(5)={veb.Successor(5)}");

        // Sort
        var v2 = new VEB(64);
        foreach (int x in new[]{17,3,45,8,63,21,0,9}) v2.Insert(x);
        Console.Write("Sorted: ");
        while (!v2.IsEmpty) { Console.Write($"{v2.Min} "); v2.Delete(v2.Min); }
        Console.WriteLine();
    }
}
```

---

## Recurrence Derivation

```
T(U) = T(√U) + O(1)

Let U = 2^{2^k}:
  T(2^{2^k}) = T(2^{2^{k-1}}) + O(1)
             = T(2^{2^{k-2}}) + O(1) + O(1)
             = ...
             = T(2) + k * O(1)
             = O(k)
             = O(log log U)   since k = log log U

For insert: at most ONE recursive call reaches a non-empty cluster.
If cluster was empty → insert into summary (cluster hits base case immediately).
If cluster was non-empty → no summary insert (cluster has space to absorb).
→ Only one recursive call of depth at the next level → T(U) = T(√U) + O(1). ✓
```

---

## Comparison with Other Integer Data Structures

| Structure | Insert | Delete | Succ/Pred | Min/Max | Space |
|---|---|---|---|---|---|
| Balanced BST | O(log n) | O(log n) | O(log n) | O(log n) | O(n) |
| Van Emde Boas | O(log log U) | O(log log U) | O(log log U) | O(1) | O(U) |
| vEB with hash | O(log log U) | O(log log U) | O(log log U) | O(1) | O(n log log U) |
| Y-Fast Trie | O(log log U) | O(log log U) | O(log log U) | O(1) | O(n) |
| Fusion Tree | O(log n / log log n) | O(log n / log log n) | O(log n / log log n) | — | O(n) |

vEB wins when U is small (e.g., 32-bit integers: U=2^{32}, log log U = 5).

---

## Pitfalls

- **Min is NOT stored recursively in any cluster** — the minimum of a vEB tree is stored only in `V.min` and not propagated down to `V.cluster[high(min)]`. This is what makes insert O(1) at base cases and ensures the single-recursive-call property. Inserting min into a cluster breaks the invariant and makes operations incorrect.
- **Lazy cluster creation** — allocating all √U clusters upfront uses O(U) space even for a few elements. Use lazy initialization (`cluster[h] = new VEB(lo_size)` only when first needed) to keep space proportional to inserted elements. With a hash map for clusters, space becomes O(n log log U).
- **Delete must handle the case x == min specially** — when deleting the minimum, find the new minimum from the first non-empty cluster (`summary.min` → `cluster[first].min`), promote it to `V.min`, then delete it from the cluster. Deleting min directly from a cluster without this promotion step corrupts the invariant.
- **U must be a power of 2** — the universe splitting assumes U = 2^k for exact integer sqrt computation. For non-power-of-2 universes, round U up to the next power of 2. Values outside [0, U-1] must be rejected; inserting them causes out-of-bounds cluster access.
- **Base case U=2 has no clusters** — when U=2, the structure stores only min and max (values 0 and 1). Any access to `cluster` for U=2 must be guarded. Recursive calls that reach U=2 must be handled directly without further recursion.
- **Successor after max returns EMPTY, not U** — successor of the maximum element (or any element >= max) returns EMPTY (represented as -1). Callers must check for EMPTY before using the result. Using the result as an index without checking can cause out-of-bounds access.

---

## Conclusion

The Van Emde Boas tree achieves **O(log log U) per operation** — theoretically optimal for integer data structures on a universe of size U:

- All operations reduce to at most one recursive call on a universe of size √U, giving the T(U) = T(√U) + O(1) recurrence.
- Min and Max are stored directly in the node, enabling O(1) queries without any recursion.
- The space cost O(U) is its main practical limitation — feasible for U ≤ 2^{20} (≈ 1 million), but impractical for 32-bit or 64-bit universes without hashing.
- With hash maps for clusters, space drops to O(n log log U) while maintaining O(log log U) expected time.

**Key takeaway:**  
vEB trees are optimal for integer sets when the universe is small (U ≤ 10^6 in practice). For larger universes, use Y-Fast Tries (same asymptotic complexity, O(n) space) or accept the O(log n) bound of balanced BSTs. The implementation complexity is moderate — the critical invariant is that min is never stored recursively in a cluster, which is the source of the single-recursion property that makes the O(log log U) analysis work.
