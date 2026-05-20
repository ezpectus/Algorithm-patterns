# Functional Graph — Cycle Detection, k-th Ancestor

## Origin & Motivation

A **functional graph** is a directed graph where every vertex has **exactly one outgoing edge**. It is defined by a function `f: V → V` where `f(v)` is the unique successor of `v`. The structure of a functional graph is completely determined: it consists of several **rho-shaped (ρ) components** — each component has a **tail** (path leading into a cycle) and a **cycle** (the cycle itself). This matches the shape of Floyd's cycle detection algorithm, which gave rise to the "rho" name.

Because every vertex has exactly one successor, functional graphs admit extremely efficient algorithms:
- **Cycle detection:** Floyd's or Brent's algorithm in O(n) time and O(1) space
- **k-th ancestor (iterate f k times):** Binary lifting in O(log k) per query after O(n log n) preprocessing
- **Cycle membership:** O(n) preprocessing, O(1) query

Complexity: **O(n)** cycle detection, **O(n log n)** preprocessing for k-th ancestor, **O(log k)** per ancestor query.

---

## Where It Is Used

- Permutation analysis (every permutation is a functional graph)
- Pseudorandom number generators (cycle detection in PRNG sequences)
- Pollard's Rho factorization (functional graph on Z/nZ)
- Competitive programming: "apply f k times to x"
- Hash table analysis (chaining and probe sequences)
- Cellular automata, game state analysis
- Finding periodic points of iterated functions

---

## Structure of Functional Graphs

Every connected component of a functional graph has exactly one cycle. Vertices not on the cycle form trees rooted at cycle nodes, with edges directed toward the root (toward the cycle).

```
Component structure:
  tail vertices → → → [cycle node] ↔ ↔ ↔ [cycle node]
                              ↑↑              ↑↑
                          tree nodes      tree nodes
                         (tails leading    (tails)
                          into cycle)

Each component = one cycle + trees hanging off cycle nodes
Global structure = union of such components (forest of rho shapes)
```

---

## Part 1 — Cycle Detection

### Floyd's Algorithm (Tortoise and Hare)

Two pointers advance at different speeds. The hare moves twice as fast as the tortoise. When they meet, they are inside a cycle.

```
tortoise = f(x0)
hare     = f(f(x0))

Phase 1: find meeting point
while tortoise != hare:
    tortoise = f(tortoise)
    hare     = f(f(hare))
// They meet somewhere inside the cycle

Phase 2: find cycle start (entry point)
tortoise = x0
while tortoise != hare:
    tortoise = f(tortoise)
    hare     = f(hare)
// Both now at cycle entry point

Phase 3: find cycle length
length = 1
hare = f(tortoise)
while tortoise != hare:
    hare = f(hare)
    length++
```

**Phase 2 proof:** Let μ = tail length, λ = cycle length. At meeting point, hare traveled 2t steps, tortoise t steps, so hare is λ ahead of tortoise inside the cycle. Moving tortoise to start and advancing both at speed 1: after μ steps, tortoise reaches cycle entry, hare has traveled μ more steps from the meeting point = cycle entry (since μ + (meeting_point_offset) ≡ 0 mod λ).

### Brent's Algorithm

Faster constant factor: save a checkpoint every power of 2 steps, compare current hare with saved checkpoint.

```
power = 1, length = 1
tortoise = x0, hare = f(x0)

while tortoise != hare:
    if power == length:
        tortoise = hare   // save checkpoint
        power *= 2
        length = 0
    hare = f(hare)
    length++

// length = cycle length
// Find cycle start:
hare = x0
for i in range(length): hare = f(hare)  // advance hare by cycle_length
tortoise = x0
while tortoise != hare:
    tortoise = f(tortoise)
    hare     = f(hare)
// tortoise = hare = cycle entry point
```

---

## Part 2 — k-th Ancestor via Binary Lifting

For functional graphs, `f^k(v)` (applying f k times) = the k-th ancestor of v following the successor edges. Binary lifting precomputes `anc[v][j]` = vertex reached after `2^j` steps from v.

```
Preprocessing:
  anc[v][0] = f(v)      // one step
  anc[v][j] = anc[anc[v][j-1]][j-1]   // 2^j steps = two 2^{j-1} steps

Query f^k(v):
  for j = LOG-1 downto 0:
      if (k >> j) & 1:
          v = anc[v][j]
          k -= (1 << j)
  return v
```

This decomposes k in binary: `k = 2^{j1} + 2^{j2} + ...` and applies each power-of-2 jump in order.

---

## Part 3 — Full Component Analysis

For each vertex, determine:
- Whether it is on a cycle (`on_cycle[v]`)
- Its cycle ID and position within the cycle
- Its distance to the cycle (`dist_to_cycle[v]`)

Algorithm:

```
visited[v] = 0 (unvisited), 1 (in progress), 2 (done)

For each unvisited vertex, follow f(v) until:
  - We hit a vertex visited in this run (found a cycle)
  - We hit a vertex already fully processed (reached a previous component)

When cycle found: mark all cycle vertices, assign cycle positions
Mark tail vertices with dist_to_cycle
```

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Floyd's cycle detection | O(μ + λ) = O(n) | O(1) |
| Brent's cycle detection | O(μ + λ) = O(n) | O(1) |
| Full graph analysis | O(n) | O(n) |
| Binary lifting preprocess | O(n log n) | O(n log n) |
| k-th ancestor query | O(log k) | O(1) |
| Cycle membership query | O(1) | O(n) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// FLOYD'S CYCLE DETECTION — single sequence x, f(x), f(f(x)), ...
// Returns {cycle_start, cycle_length}
// ================================================================
pair<int,int> floyd(int x0, function<int(int)> f) {
    // Phase 1: find meeting point
    int slow = f(x0), fast = f(f(x0));
    while (slow != fast) {
        slow = f(slow);
        fast = f(f(fast));
    }

    // Phase 2: find cycle start (μ)
    slow = x0;
    while (slow != fast) {
        slow = f(slow);
        fast = f(fast);
    }
    int cycle_start = slow;

    // Phase 3: find cycle length (λ)
    int len = 1;
    fast = f(slow);
    while (fast != slow) {
        fast = f(fast);
        len++;
    }

    return {cycle_start, len};
}

// ================================================================
// BRENT'S CYCLE DETECTION
// Returns {cycle_start, cycle_length}
// ================================================================
pair<int,int> brent(int x0, function<int(int)> f) {
    int power = 1, len = 1;
    int tortoise = x0, hare = f(x0);

    while (tortoise != hare) {
        if (power == len) {
            tortoise = hare;
            power <<= 1;
            len = 0;
        }
        hare = f(hare);
        len++;
    }

    // Find cycle start
    hare = x0;
    for (int i = 0; i < len; i++) hare = f(hare);
    tortoise = x0;
    while (tortoise != hare) {
        tortoise = f(tortoise);
        hare     = f(hare);
    }

    return {tortoise, len};
}

// ================================================================
// FUNCTIONAL GRAPH — full analysis
// For n vertices with successor nxt[v] in [0,n)
// ================================================================
struct FunctionalGraph {
    int n;
    vector<int> nxt;            // successor function f
    vector<int> cycle_id;       // which cycle does v belong to (or lead to)
    vector<int> pos_in_cycle;   // position of v within its cycle (-1 if tail)
    vector<int> dist_to_cycle;  // steps from v to first cycle vertex (0 if on cycle)
    vector<bool> on_cycle;
    int num_cycles;

    // Binary lifting for k-th ancestor
    static const int LOG = 40; // supports k up to 2^40
    vector<vector<int>> anc;

    FunctionalGraph(int n, vector<int> nxt)
        : n(n), nxt(nxt),
          cycle_id(n,-1), pos_in_cycle(n,-1), dist_to_cycle(n,0),
          on_cycle(n,false), num_cycles(0),
          anc(LOG, vector<int>(n)) {}

    void build() {
        // Step 1: Detect all cycles using coloring DFS
        // color: 0=unvisited, 1=in_stack, 2=done
        vector<int> color(n, 0);

        for (int start = 0; start < n; start++) {
            if (color[start] != 0) continue;

            // Follow chain from start until revisit or already-done vertex
            vector<int> path;
            unordered_map<int,int> path_pos;
            int v = start;

            while (color[v] == 0) {
                color[v] = 1;
                path_pos[v] = (int)path.size();
                path.push_back(v);
                v = nxt[v];
            }

            if (color[v] == 1) {
                // v is on a new cycle: extract it
                int cid = num_cycles++;
                int start_idx = path_pos[v];
                int pos = 0;
                for (int i = start_idx; i < (int)path.size(); i++) {
                    on_cycle[path[i]] = true;
                    cycle_id[path[i]] = cid;
                    pos_in_cycle[path[i]] = pos++;
                }
            }

            // Mark all vertices in path as done
            for (int u : path) color[u] = 2;
        }

        // Step 2: For tail vertices, compute dist_to_cycle and cycle_id
        // Process: follow nxt until reaching a cycle vertex
        for (int v = 0; v < n; v++) {
            if (on_cycle[v]) continue;
            if (cycle_id[v] != -1) continue; // already processed

            // Collect the tail path
            vector<int> tail;
            int u = v;
            while (!on_cycle[u] && cycle_id[u] == -1) {
                tail.push_back(u);
                u = nxt[u];
            }
            // u is now either on cycle or already processed
            int cid = on_cycle[u] ? cycle_id[u] : cycle_id[u];
            int base_dist = on_cycle[u] ? 0 : dist_to_cycle[u];

            for (int i = (int)tail.size()-1; i >= 0; i--) {
                dist_to_cycle[tail[i]] = base_dist + (int)(tail.size() - i);
                cycle_id[tail[i]]       = cid;
            }
        }

        // Step 3: Binary lifting
        for (int v = 0; v < n; v++) anc[0][v] = nxt[v];
        for (int j = 1; j < LOG; j++)
            for (int v = 0; v < n; v++)
                anc[j][v] = anc[j-1][anc[j-1][v]];
    }

    // k-th successor of v: apply f k times
    int kth(int v, ll k) const {
        for (int j = 0; j < LOG; j++)
            if ((k >> j) & 1) v = anc[j][v];
        return v;
    }

    // Period of v: minimum k>0 s.t. f^k(v) = v
    // = cycle_length for cycle vertices,
    //   lcm or period analysis for tails (period = cycle length of the cycle)
    int period(int v) const {
        if (!on_cycle[v]) return -1; // tails don't have a period in this sense
        // Find cycle length: follow until return
        int len = 1, u = nxt[v];
        while (u != v) { u = nxt[u]; len++; }
        return len;
    }

    // Are u and v in the same component (same cycle)?
    bool same_component(int u, int v) const {
        return cycle_id[u] == cycle_id[v];
    }
};

// ================================================================
// APPLICATION: Permutation cycle decomposition
// A permutation is a bijective functional graph on [0,n)
// All vertices are on cycles (no tails)
// ================================================================
struct Permutation {
    int n;
    vector<int> perm;
    vector<int> cycle_id, pos_in_cycle, cycle_len;
    int num_cycles;

    Permutation(vector<int> p) : n(p.size()), perm(p),
        cycle_id(p.size(),-1), pos_in_cycle(p.size(),-1),
        cycle_len(0), num_cycles(0) {}

    void decompose() {
        for (int start = 0; start < n; start++) {
            if (cycle_id[start] != -1) continue;
            int cid = num_cycles++;
            int v = start, pos = 0, len = 0;
            while (cycle_id[v] == -1) {
                cycle_id[v] = cid;
                pos_in_cycle[v] = pos++;
                len++;
                v = perm[v];
            }
            cycle_len.push_back(len);
        }
    }

    // k-th power of permutation applied to v
    int power(int v, ll k) const {
        int cid = cycle_id[v];
        int len = cycle_len[cid];
        int pos = pos_in_cycle[v];
        int target_pos = (int)((pos + k % len + len) % len);
        // Find vertex with cycle_id=cid and pos_in_cycle=target_pos
        // Need reverse map: cycle + pos -> vertex
        // Build on demand or precompute
        // Simple: just follow the permutation k%len times
        int u = v;
        ll steps = ((k % len) + len) % len;
        for (ll i = 0; i < steps; i++) u = perm[u];
        return u;
    }

    // Cycle lengths
    vector<int> get_cycle_lengths() const { return cycle_len; }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // --- Floyd's detection on a simple sequence ---
    {
        printf("=== Floyd's Cycle Detection ===\n");
        // f(x) = (x*x + 1) % 7
        auto f = [](int x) -> int { return (x*x + 1) % 7; };
        auto [start, len] = floyd(0, f);
        printf("x=0, f(x)=(x^2+1) mod 7:\n");
        printf("  Cycle start = %d, cycle length = %d\n", start, len);

        // Print sequence
        printf("  Sequence: ");
        int x = 0;
        for (int i = 0; i < 15; i++) { printf("%d ", x); x = f(x); }
        printf("\n");
    }

    // --- Brent's detection ---
    {
        printf("\n=== Brent's Cycle Detection ===\n");
        auto f = [](int x) -> int { return (x*x + 1) % 13; };
        auto [start, len] = brent(3, f);
        printf("x=3, f(x)=(x^2+1) mod 13:\n");
        printf("  Cycle start = %d, cycle length = %d\n", start, len);
    }

    // --- Functional graph analysis ---
    {
        printf("\n=== Functional Graph Analysis ===\n");
        // 0->1->2->3->1 (cycle: 1->2->3->1, tail: 0->1)
        // 4->5->6->4 (cycle: 4->5->6->4)
        // 7->4 (tail: 7->4)
        int n = 8;
        vector<int> nxt = {1, 2, 3, 1, 5, 6, 4, 4};
        FunctionalGraph fg(n, nxt);
        fg.build();

        printf("Vertex analysis:\n");
        for (int v = 0; v < n; v++) {
            printf("  v=%d: on_cycle=%d dist_to_cycle=%d cycle_id=%d",
                v, fg.on_cycle[v], fg.dist_to_cycle[v], fg.cycle_id[v]);
            if (fg.on_cycle[v])
                printf(" pos_in_cycle=%d", fg.pos_in_cycle[v]);
            printf("\n");
        }

        // k-th successor queries
        printf("\nk-th successor:\n");
        printf("  f^5(0) = %d\n",  fg.kth(0, 5));   // 0->1->2->3->1->2 = 2
        printf("  f^10(0) = %d\n", fg.kth(0, 10));
        printf("  f^3(7) = %d\n",  fg.kth(7, 3));   // 7->4->5->6 = 6
        printf("  f^100(4) = %d\n", fg.kth(4, 100)); // cycle 4-5-6 len 3: 100%3=1, 4->5
    }

    // --- Permutation decomposition ---
    {
        printf("\n=== Permutation Cycle Decomposition ===\n");
        // perm = [2, 0, 3, 1, 5, 4]  (two cycles: (0 2 3 1) and (4 5))
        Permutation perm({2, 0, 3, 1, 5, 4});
        perm.decompose();
        printf("Cycles: %d\n", perm.num_cycles);
        printf("Cycle lengths: ");
        for (int l : perm.get_cycle_lengths()) printf("%d ", l);
        printf("\n");

        // k-th power applied to vertex 0
        for (ll k : {1LL, 2LL, 4LL, 100LL}) {
            printf("  perm^%lld(0) = %d\n", k, perm.power(0, k));
        }
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class FunctionalGraph {
    private readonly int n;
    private readonly int[] nxt;
    public  readonly int[] CycleId, PosInCycle, DistToCycle;
    public  readonly bool[] OnCycle;
    public int NumCycles { get; private set; }

    private const int LOG = 40;
    private readonly int[,] anc;

    public FunctionalGraph(int n, int[] nxt) {
        this.n = n; this.nxt = nxt;
        CycleId      = new int[n]; Array.Fill(CycleId, -1);
        PosInCycle   = new int[n]; Array.Fill(PosInCycle, -1);
        DistToCycle  = new int[n];
        OnCycle      = new bool[n];
        anc          = new int[LOG, n];
    }

    public void Build() {
        var color = new int[n]; // 0=unvisited, 1=in_stack, 2=done

        for (int start = 0; start < n; start++) {
            if (color[start] != 0) continue;

            var path = new List<int>();
            var pathPos = new Dictionary<int,int>();
            int v = start;

            while (color[v] == 0) {
                color[v] = 1;
                pathPos[v] = path.Count;
                path.Add(v);
                v = nxt[v];
            }

            if (color[v] == 1) {
                int cid = NumCycles++;
                int si = pathPos[v], pos = 0;
                for (int i = si; i < path.Count; i++) {
                    OnCycle[path[i]] = true;
                    CycleId[path[i]] = cid;
                    PosInCycle[path[i]] = pos++;
                }
            }
            foreach (int u in path) color[u] = 2;
        }

        // Compute dist_to_cycle for tail vertices
        for (int v = 0; v < n; v++) {
            if (OnCycle[v] || CycleId[v] != -1) continue;
            var tail = new List<int>();
            int u = v;
            while (!OnCycle[u] && CycleId[u] == -1) { tail.Add(u); u = nxt[u]; }
            int cid = CycleId[u];
            int baseDist = OnCycle[u] ? 0 : DistToCycle[u];
            for (int i = tail.Count - 1; i >= 0; i--) {
                DistToCycle[tail[i]] = baseDist + (tail.Count - i);
                CycleId[tail[i]]     = cid;
            }
        }

        // Binary lifting
        for (int v = 0; v < n; v++) anc[0, v] = nxt[v];
        for (int j = 1; j < LOG; j++)
            for (int v = 0; v < n; v++)
                anc[j, v] = anc[j-1, anc[j-1, v]];
    }

    public int Kth(int v, long k) {
        for (int j = 0; j < LOG; j++)
            if (((k >> j) & 1) == 1) v = anc[j, v];
        return v;
    }

    // Floyd's cycle detection on a function
    public static (int start, int len) Floyd(int x0, Func<int,int> f) {
        int slow = f(x0), fast = f(f(x0));
        while (slow != fast) { slow = f(slow); fast = f(f(fast)); }

        slow = x0;
        while (slow != fast) { slow = f(slow); fast = f(fast); }
        int cycleStart = slow;

        int len = 1; fast = f(slow);
        while (fast != slow) { fast = f(fast); len++; }
        return (cycleStart, len);
    }

    public static (int start, int len) Brent(int x0, Func<int,int> f) {
        int power = 1, len = 1, tortoise = x0, hare = f(x0);
        while (tortoise != hare) {
            if (power == len) { tortoise = hare; power <<= 1; len = 0; }
            hare = f(hare); len++;
        }
        hare = x0;
        for (int i = 0; i < len; i++) hare = f(hare);
        tortoise = x0;
        while (tortoise != hare) { tortoise = f(tortoise); hare = f(hare); }
        return (tortoise, len);
    }

    public static void Main() {
        // Floyd
        int x = 0;
        Func<int,int> fx = v => (v*v + 1) % 7;
        var (cs, cl) = Floyd(0, fx);
        Console.WriteLine($"Floyd: cycle_start={cs}, cycle_len={cl}");

        // Functional graph
        int[] nxt = {1,2,3,1,5,6,4,4};
        var fg = new FunctionalGraph(8, nxt);
        fg.Build();

        for (int v = 0; v < 8; v++)
            Console.WriteLine($"v={v}: on_cycle={fg.OnCycle[v]} dist={fg.DistToCycle[v]} cid={fg.CycleId[v]}");

        Console.WriteLine($"\nf^5(0)   = {fg.Kth(0,5)}");
        Console.WriteLine($"f^3(7)   = {fg.Kth(7,3)}");
        Console.WriteLine($"f^100(4) = {fg.Kth(4,100)}");
    }
}
```

---

## Floyd's Algorithm — Proof of Phase 2

```
Let:
  μ = tail length (steps from x0 to cycle entry)
  λ = cycle length
  ν = position of meeting point within cycle (from cycle entry)

At meeting point (Phase 1):
  Tortoise has taken t steps:   position = μ + ν  (mod λ inside cycle)
  Hare has taken 2t steps:      position = μ + ν  (same position)
  → 2t - t = kλ for some k ≥ 1
  → t = kλ

Phase 2: reset tortoise to x0, both advance 1 step per iteration.
After μ more steps:
  Tortoise: at position μ (= cycle entry)
  Hare: at position (ν + μ) mod λ from cycle entry
       = (t + μ) mod λ
       = (kλ + μ) mod λ
       = μ mod λ

If μ < λ: hare is at position μ in the cycle = same as cycle entry position μ
If μ ≥ λ: μ mod λ still gives the entry point by definition

→ Both arrive at cycle entry simultaneously after μ steps. ✓
```

---

## Pitfalls

- **LOG must be large enough for k** — binary lifting uses `LOG` bits. For k up to 10^18, `LOG = 60` is required. Setting `LOG = 20` (correct for n ≤ 10^6) fails silently for large k values — the high bits are not stored, and the query returns the wrong vertex without any error.
- **Cycle detection initialization** — Floyd's phase 1 must start with `slow = f(x0)` and `fast = f(f(x0))` — NOT with both at `x0`. If both start at `x0`, they are always equal and the while loop exits immediately with wrong results. The tortoise and hare must start one step apart.
- **Tail vertices have no period** — calling `period(v)` on a tail vertex is undefined; tail vertices never return to themselves. Always check `on_cycle[v]` before computing periods. The period of a tail vertex "through the cycle" is the cycle length of the cycle it leads into, which is different from the vertex's own period.
- **Functional graph coloring: two-pass for tails** — during cycle detection, tail vertices are colored "done" after a cycle is found, but their `cycle_id` and `dist_to_cycle` are not set in the first pass. A second pass is needed to propagate this information from cycle vertices backward through tail chains. Omitting the second pass leaves tail vertices with incorrect or uninitialized data.
- **Permutation k-th power: use k mod cycle_length** — for a permutation cycle of length L, applying the permutation k times is equivalent to applying it `k mod L` times. For large k (up to 10^{18}), always reduce modulo the cycle length before iterating. Without reduction, even binary lifting correctly handles large k — but naive iteration in a loop does not.
- **Brent's algorithm: length counts from reset, not from start** — the `len` counter in Brent's tracks the length of the current "power interval", not the total path length. When `power == len`, the checkpoint is reset to the current hare position and `len` resets to 0. The final `len` value when the loop exits is the cycle length — not the tail length or total path length. Using `len` as the tail length gives wrong cycle start detection.

---

## Complexity Summary

| Operation | Time | Space |
|---|---|---|
| Floyd's / Brent's (single sequence) | O(μ + λ) | O(1) |
| Full graph cycle analysis | O(n) | O(n) |
| Binary lifting preprocessing | O(n log k_max) | O(n log k_max) |
| k-th successor query | O(log k) | O(1) |
| Permutation cycle decomposition | O(n) | O(n) |
| Permutation k-th power | O(cycle_len) or O(log k) | O(1) or O(n log k) |

---

## Conclusion

Functional graphs are **the simplest nontrivial directed graph structure** — exactly one outgoing edge per vertex — yet they support a rich set of efficient algorithms:

- Floyd's and Brent's algorithms detect cycles in O(μ+λ) time with O(1) space — the most space-efficient cycle detection possible.
- Binary lifting turns repeated function application into O(log k) queries after O(n log n) preprocessing — handling k up to 10^{18} trivially.
- Every connected component decomposes cleanly into a cycle plus incoming trees, enabling O(n) preprocessing that answers all structural queries (cycle membership, distance to cycle, component identity) in O(1).

**Key takeaway:**  
The two most common functional graph operations in competitive programming are "find the cycle" (Floyd/Brent, O(1) space) and "apply f exactly k times" (binary lifting, O(log k) per query). The binary lifting table is identical to the one used for LCA on trees — the same 2D array `anc[j][v] = f^{2^j}(v)`, just applied to the successor function instead of tree parents.
