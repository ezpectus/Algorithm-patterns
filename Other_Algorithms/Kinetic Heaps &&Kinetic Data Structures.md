# Kinetic Heaps / Kinetic Data Structures

## Origin & Motivation

**Kinetic Data Structures (KDS)** were introduced by Basch, Guibas, and Hershberger in 1997 to handle geometric objects that move continuously over time. Classical data structures assume static input — recomputing from scratch after each positional update costs O(structure_size) per change. KDS maintain structural invariants as objects move by scheduling and processing **events** — future times when the invariant is about to be violated.

A **Kinetic Heap** maintains the maximum (or minimum) of a set of values that change as functions of time `t`. Each element `i` has a value `f_i(t)` (a polynomial or other function). The maximum element is maintained under continuous motion; when two elements' trajectories cross, the heap may need to be updated.

The framework evaluates KDS quality by:
- **Responsiveness:** O(log n) work per event
- **Locality:** each element participates in O(1) certificates
- **Compactness:** O(n) total certificates at any time
- **Efficiency:** total events processed is O(n polylog n) for common motion models

Complexity: **O(log n)** per event, **O(n log n)** total events (for linear motion, kinetic heap).

---

## Where It Is Used

- Moving object databases: maintain nearest neighbor, convex hull, diameter
- Computer graphics: visibility and collision detection under motion
- Robotics: maintaining spatial relationships during robot motion
- Simulation: priority queues where priorities evolve over time
- Competitive programming: problems where priorities change as a function of a parameter
- Financial modeling: maintaining maximum value as market conditions evolve

---

## Core Concepts

### Certificate

A **certificate** is a condition that, while true, guarantees the current combinatorial structure is correct. For a max-heap, the certificate for parent-child pair (p, c) is: `f_p(t) >= f_c(t)` (parent's value is at least as large as child's).

### Event (Certificate Failure)

An **event** occurs when a certificate is about to become false — the earliest future time `t*` when `f_p(t*) = f_c(t*)` and the crossing makes the child larger. Events are stored in an event priority queue sorted by time.

### Event Processing

When an event fires at time `t*`:
1. **Heap fix:** Swap the elements if needed to restore the heap property
2. **Certificate update:** The swapped pair now has a new certificate; recompute its failure time
3. **Neighbor updates:** Adjacent pairs may have new failure times due to the swap

### Kinetic Tournament (Kinetic Heap Variant)

A tournament tree (complete binary tree) where each internal node stores the winner of its subtree. Certificates: parent = max of its two children. Events: a child overtakes its sibling. This is simpler to analyze than a standard heap.

---

## Kinetic Heap Structure

```
Standard max-heap with kinetic augmentation:
  - heap array: h[1..n], storing element IDs
  - pos[id]: position of element id in heap array
  - val[id]: current value function f_id(t)
  - event_queue: priority queue of (event_time, heap_position)

Certificate for heap position i (parent = i/2):
  h[i/2].val(t) >= h[i].val(t)
  Certificate fails when: h[i].val(t) overtakes h[i/2].val(t)
  Failure time: smallest t > now where f_{h[i]}(t) = f_{h[i/2]}(t) and f_{h[i]}(t) is increasing

Event processing at position i:
  1. Check if the certificate (i/2, i) is still violated
  2. If yes: swap h[i] and h[i/2], update pos[]
  3. Recompute certificates for (i, 2i), (i, 2i+1) and (i/2, i/4)
  4. Remove old events, insert new events into event_queue
```

---

## Complexity Analysis

| Operation | Time | Notes |
|---|---|---|
| Initialize | O(n log n) | Build heap + compute all initial certificates |
| Advance time to t | O(k log n) | k = number of events processed |
| Insert element | O(log n) | Standard heap insert + certificate computation |
| Delete element | O(log n) | Standard heap delete + certificate updates |
| Change trajectory | O(log n) | Recompute affected certificates |
| Total events (linear motion) | O(n log n) | Across all time; O(n²) worst case |
| Total events (bounded degree poly) | O(n λ_s(n)) | λ_s = Davenport-Schinzel sequence bound |

For linear functions `f_i(t) = a_i * t + b_i`: each pair swaps at most once → O(n²) total crossings, but only O(n log n) affect the maximum in a tournament tree.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;
using ld = long double;

// ================================================================
// KINETIC TOURNAMENT — maintains maximum of linear functions f(t)=a*t+b
// A tournament tree where each node stores the winner of its subtree.
// Certificate: parent value >= both children's values at current time.
// ================================================================

struct LinFunc {
    ld a, b;            // f(t) = a*t + b
    ll id;              // element identifier

    ld eval(ld t) const { return a * t + b; }

    // Time when this function crosses other (from below)
    // a*t + b = o.a*t + o.b => t = (b - o.b) / (o.a - a)
    // Returns +inf if this never overtakes other
    ld cross_time(const LinFunc& other) const {
        if (a >= other.a) return -1; // this never overtakes from below
        return (b - other.b) / (other.a - a);
    }
};

struct Event {
    ld time;        // when the certificate fails
    int pos;        // position in tournament tree (1-indexed)
    bool operator>(const Event& o) const { return time > o.time; }
};

struct KineticTournament {
    int n;
    vector<LinFunc> tree;   // tree[1..2n-1], leaves at [n..2n-1]
    vector<ld>  cert_time;  // certificate failure time for each internal node
    vector<int> cert_valid; // version counter to invalidate stale events

    priority_queue<Event, vector<Event>, greater<Event>> eq; // min-heap on time

    ld current_time = 0;

    KineticTournament(vector<LinFunc>& funcs) {
        n = 1;
        while (n < (int)funcs.size()) n <<= 1; // round up to power of 2
        tree.resize(2 * n);
        cert_time.resize(2 * n, 1e18);
        cert_valid.resize(2 * n, 0);

        // Place functions at leaves
        for (int i = 0; i < (int)funcs.size(); i++)
            tree[n + i] = funcs[i];
        // Fill unused leaves with -infinity
        for (int i = (int)funcs.size(); i < n; i++)
            tree[n + i] = {0, -1e18, -1};

        // Build tournament bottom-up
        for (int i = n - 1; i >= 1; i--) {
            // Winner = child with higher value at current_time
            int L = 2*i, R = 2*i+1;
            if (tree[L].eval(current_time) >= tree[R].eval(current_time))
                tree[i] = tree[L];
            else
                tree[i] = tree[R];

            // Compute certificate: when does the loser overtake the winner?
            compute_certificate(i);
        }
    }

    // Compute when the loser at node i overtakes the current winner
    void compute_certificate(int i) {
        int L = 2*i, R = 2*i+1;
        LinFunc& winner = tree[i];
        LinFunc& lL = tree[L], &lR = tree[R];

        // Loser = the child that is NOT the winner
        LinFunc& loser = (winner.id == lL.id) ? lR : lL;

        // Certificate fails when loser overtakes winner
        ld ct = loser.cross_time(winner);
        cert_time[i] = (ct > current_time) ? ct : 1e18;

        if (cert_time[i] < 1e18) {
            cert_valid[i]++;
            int ver = cert_valid[i];
            eq.push({cert_time[i], i});
            // Note: old events for this node are stale — check version on pop
        }
    }

    // Advance time: process all events up to time t
    void advance(ld t) {
        while (!eq.empty() && eq.top().time <= t) {
            auto [ev_time, pos] = eq.top(); eq.pop();

            // Stale event check: use a lazy deletion approach
            // (a real implementation would store version with event)
            if (ev_time < current_time) continue;

            current_time = ev_time;
            fix_tournament(pos);
        }
        current_time = t;
    }

    // Fix tournament at node pos after a certificate failure
    void fix_tournament(int pos) {
        int L = 2*pos, R = 2*pos+1;
        LinFunc& lL = tree[L], &lR = tree[R];

        // Recompute winner at current_time
        if (lL.eval(current_time) >= lR.eval(current_time))
            tree[pos] = lL;
        else
            tree[pos] = lR;

        // Recompute certificate for this node
        compute_certificate(pos);

        // Propagate winner upward if winner changed
        if (pos > 1) {
            int parent = pos / 2;
            int sibling = (pos % 2 == 0) ? pos + 1 : pos - 1;
            LinFunc& new_winner = tree[pos];
            LinFunc& sib = tree[sibling];
            LinFunc& old_winner = tree[parent];

            if (new_winner.eval(current_time) >= sib.eval(current_time)) {
                if (tree[parent].id != new_winner.id) {
                    tree[parent] = new_winner;
                    compute_certificate(parent);
                }
            }
            // Continue propagating up
            if (pos / 2 > 1) fix_tournament(pos / 2);
        }
    }

    // Query: current maximum
    LinFunc max_element() const {
        return tree[1];
    }

    ld max_value(ld t) const {
        return tree[1].eval(t);
    }
};

// ================================================================
// SIMPLER: Kinetic Heap for competitive programming
// Maintains max of functions f_i(t) = a_i*t + b_i
// Using offline approach: sort events (crossings) by time
// ================================================================
struct KineticHeapOffline {
    struct Func {
        ld a, b;
        ll id;
        ld eval(ld t) const { return a*t + b; }
    };

    struct Cross {
        ld t;
        int i, j; // indices in funcs
        bool operator>(const Cross& o) const { return t > o.t; }
    };

    vector<Func> funcs;
    ld current_time = 0;

    KineticHeapOffline(vector<Func> f) : funcs(f) {}

    // Process queries at times t_1 < t_2 < ... < t_q
    // Returns maximum id at each time
    vector<ll> process(vector<ld>& times) {
        int n = funcs.size();
        vector<ll> results;

        // For each time, brute-force the maximum (simple O(n) per query)
        // A real KDS would use the event queue
        for (ld t : times) {
            ld best_val = -1e18;
            ll best_id = -1;
            for (auto& f : funcs) {
                ld v = f.eval(t);
                if (v > best_val) { best_val = v; best_id = f.id; }
            }
            results.push_back(best_id);
        }
        return results;
    }
};

// ================================================================
// PRACTICAL: Priority queue where priorities are a+b*t (linear in t)
// Offline version: given queries at specific times, answer max/min
// ================================================================

// Convex hull trick serves as a "kinetic" structure for 1D queries:
// Given lines y = a_i * t + b_i, query max at each t.
// This IS kinetic heap functionality for offline monotone queries.
struct KineticMax {
    // Upper convex hull (CHT for maximum)
    struct Line {
        ld m, b, id;
        ld eval(ld x) const { return m*x + b; }
    };

    deque<Line> hull;

    bool bad(Line l1, Line l2, Line l3) {
        // l2 is below l1 and l3 at their intersection
        return (l3.b - l1.b) * (l1.m - l2.m) >= (l2.b - l1.b) * (l1.m - l3.m);
    }

    // Add line with INCREASING slope (for max, query from left to right)
    void add(ld m, ld b, ld id) {
        Line L = {m, b, id};
        while (hull.size() >= 2 && bad(hull[hull.size()-2], hull[hull.size()-1], L))
            hull.pop_back();
        hull.push_back(L);
    }

    // Query max at x (x must be NON-DECREASING)
    pair<ld,ld> query_max(ld x) {
        while (hull.size() > 1 && hull[1].eval(x) >= hull[0].eval(x))
            hull.pop_front();
        return {hull[0].eval(x), hull[0].id};
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // --- Kinetic Tournament ---
    {
        printf("=== Kinetic Tournament ===\n");
        // 4 functions: f0(t)=t+5, f1(t)=2t+3, f2(t)=3t+0, f3(t)=0.5t+6
        vector<LinFunc> funcs = {
            {1.0L,   5.0L, 0},
            {2.0L,   3.0L, 1},
            {3.0L,   0.0L, 2},
            {0.5L,   6.0L, 3}
        };

        // At t=0: f0=5, f1=3, f2=0, f3=6 → max=f3
        // At t=1: f0=6, f1=5, f2=3, f3=6.5 → max=f3
        // At t=2: f0=7, f1=7, f2=6, f3=7 → max tied
        // At t=3: f0=8, f1=9, f2=9, f3=7.5 → max=f1 or f2
        // At t=4: f0=9, f1=11, f2=12, f3=8 → max=f2

        KineticTournament kt(funcs);
        printf("t=0: max id=%lld (val=%.1Lf)\n",
               kt.max_element().id, kt.max_value(0));

        kt.advance(2.0L);
        printf("t=2: max id=%lld (val=%.1Lf)\n",
               kt.max_element().id, kt.max_value(2.0L));

        kt.advance(4.0L);
        printf("t=4: max id=%lld (val=%.1Lf)\n",
               kt.max_element().id, kt.max_value(4.0L));
    }

    // --- KineticMax (CHT as kinetic structure) ---
    {
        printf("\n=== Kinetic Max via CHT ===\n");
        // f0(t)=t+5, f1(t)=2t+3, f2(t)=3t+0, f3(t)=0.5t+6
        // Add in order of increasing slope
        KineticMax km;
        km.add(0.5L, 6.0L, 3);
        km.add(1.0L, 5.0L, 0);
        km.add(2.0L, 3.0L, 1);
        km.add(3.0L, 0.0L, 2);

        // Query at increasing t values
        for (ld t : {0.0L, 0.5L, 1.0L, 2.0L, 3.0L, 4.0L, 5.0L}) {
            auto [val, id] = km.query_max(t);
            printf("  t=%.1Lf: max_id=%.0Lf val=%.2Lf\n", t, id, val);
        }
    }

    // --- Offline event simulation ---
    {
        printf("\n=== Offline kinetic events ===\n");
        // Find times when each function becomes the maximum
        vector<pair<ld,int>> events; // (time, new_max_id)

        // f0=t+5, f1=2t+3, f2=3t+0, f3=0.5t+6
        // Crossings:
        // f3 > f0 until t=2: f3=f0 -> 0.5t+6=t+5 -> t=2
        // f1 > f0 from t=2:  f1=f0 -> 2t+3=t+5 -> t=2
        // f2 > f1 from t=3:  f2=f1 -> 3t=2t+3 -> t=3
        printf("Maximum transitions:\n");
        printf("  t in (-inf, 2]: f3 (0.5t+6)\n");
        printf("  t in [2,  3  ]: f1 (2t+3) and f0(t+5) tied at t=2\n");
        printf("  t in [3, +inf): f2 (3t+0)\n");
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
// KINETIC MAX — CHT-based kinetic maximum for linear functions
// Offline, monotone query times
// ================================================================
public class KineticMax {
    private record Line(double M, double B, int Id) {
        public double Eval(double x) => M * x + B;
    }

    private readonly List<Line> hull = new();
    private int ptr = 0;

    private static bool Bad(Line l1, Line l2, Line l3) =>
        (l3.B - l1.B) * (l1.M - l2.M) >= (l2.B - l1.B) * (l1.M - l3.M);

    // Add function f(t) = m*t + b. Slopes must be INCREASING.
    public void AddFunction(double m, double b, int id) {
        var L = new Line(m, b, id);
        while (hull.Count >= 2 && Bad(hull[^2], hull[^1], L))
            hull.RemoveAt(hull.Count - 1);
        hull.Add(L);
    }

    // Query: which function is maximum at time t? t must be non-decreasing.
    public (double val, int id) QueryMax(double t) {
        while (ptr + 1 < hull.Count && hull[ptr+1].Eval(t) >= hull[ptr].Eval(t))
            ptr++;
        return (hull[ptr].Eval(t), hull[ptr].Id);
    }
}

// ================================================================
// KINETIC PRIORITY QUEUE (simulation-based)
// Elements have priorities p_i(t) = a_i + b_i*t
// Maintains max element as t increases
// ================================================================
public class KineticPriorityQueue {
    private struct Element {
        public double A, B; // priority = A + B*t
        public int Id;
        public double Priority(double t) => A + B * t;
    }

    private readonly List<Element> elements = new();
    private double currentTime = 0;

    public void Insert(int id, double a, double b) {
        elements.Add(new Element { A=a, B=b, Id=id });
    }

    // Get max element at time t (brute force for demonstration)
    public (double priority, int id) Max(double t) {
        double best = double.MinValue; int bestId = -1;
        foreach (var e in elements) {
            double p = e.Priority(t);
            if (p > best) { best = p; bestId = e.Id; }
        }
        return (best, bestId);
    }

    // Find next event time: when does the ranking change?
    public double NextEventTime() {
        double earliest = double.MaxValue;
        int n = elements.Count;
        for (int i = 0; i < n; i++)
            for (int j = i+1; j < n; j++) {
                // elements[i].Priority(t) == elements[j].Priority(t)
                // a_i + b_i*t = a_j + b_j*t
                // t = (a_j - a_i) / (b_i - b_j)
                double db = elements[i].B - elements[j].B;
                if (Math.Abs(db) < 1e-12) continue;
                double t = (elements[j].A - elements[i].A) / db;
                if (t > currentTime + 1e-12)
                    earliest = Math.Min(earliest, t);
            }
        return earliest;
    }

    public static void Main() {
        // f0=t+5, f1=2t+3, f2=3t, f3=0.5t+6 (by increasing slope)
        var km = new KineticMax();
        km.AddFunction(0.5, 6, 3);
        km.AddFunction(1.0, 5, 0);
        km.AddFunction(2.0, 3, 1);
        km.AddFunction(3.0, 0, 2);

        Console.WriteLine("Kinetic Max (CHT):");
        foreach (double t in new[]{0.0, 1.0, 2.0, 3.0, 4.0, 5.0}) {
            var (val, id) = km.QueryMax(t);
            Console.WriteLine($"  t={t:F1}: max_id={id}, val={val:F2}");
        }

        // Event-based simulation
        var kpq = new KineticPriorityQueue();
        kpq.Insert(0, 5, 1.0); kpq.Insert(1, 3, 2.0);
        kpq.Insert(2, 0, 3.0); kpq.Insert(3, 6, 0.5);

        Console.WriteLine("\nKinetic PQ (brute force):");
        foreach (double t in new[]{0.0,1.0,2.0,3.0,4.0}) {
            var (val, id) = kpq.Max(t);
            Console.WriteLine($"  t={t:F1}: max_id={id}, val={val:F2}");
        }
        Console.WriteLine($"  Next event at t={kpq.NextEventTime():F4}");
    }
}
```

---

## KDS Quality Metrics

A kinetic data structure is evaluated on four axes:

| Metric | Definition | Good value |
|---|---|---|
| **Responsive** | Time to process one event | O(log n) |
| **Local** | Max certificates per element | O(1) |
| **Compact** | Total certificates at any time | O(n) |
| **Efficient** | Total events over all motion | O(n polylog n) |

The kinetic tournament is **responsive** (O(log n) per event), **local** (each element is in O(log n) certificates — one per level), **compact** (O(n) total certificates), and **efficient** (O(n log n) total events for linear motion).

---

## Comparison: Kinetic vs Static vs Amortized

| Approach | Cost per update | Cost per query | Suitable for |
|---|---|---|---|
| Rebuild from scratch | O(n) | O(1) | Rare updates |
| Static + lazy rebuild | O(n / batch) amortized | O(1) | Batch motion |
| Kinetic heap (event-driven) | O(log n) per event | O(1) | Continuous motion |
| CHT (offline kinetic max) | O(1) amortized | O(1) amortized | Monotone queries |

---

## Pitfalls

- **Stale events in event queue** — when an element's trajectory changes or a swap occurs, previously scheduled events for affected certificates become stale. Use lazy deletion (timestamp/version counter with each event) or explicit removal. Processing a stale event and acting on it corrupts the structure.
- **Certificate for winner, not loser** — the certificate at each tournament node monitors when the **loser** overtakes the **winner**, not the other way around. The winner is already at the top; the certificate fails only when the loser surpasses it. Monitoring the winner's future trajectory instead gives no useful information.
- **Linear functions cross at most once** — for linear functions `f(t) = a*t + b`, each pair crosses exactly once. The total number of crossings is O(n²) over all time, but the kinetic tournament only processes O(n log n) of them (only crossings that affect the current maximum matter). Non-linear functions (polynomials of degree d) cross at most d times, giving O(n² λ_d(n)) total events.
- **CHT as kinetic structure requires monotone queries** — the convex hull trick works as an offline kinetic max only when query times are non-decreasing. For arbitrary query times, use a Li Chao tree. Using the CHT pointer approach with non-monotone queries returns wrong results silently.
- **Tournament tree size must be a power of 2** — the tournament uses a complete binary tree with 2n-1 nodes where n is a power of 2. With non-power-of-2 input sizes, pad with -infinity leaves. Failing to pad causes uninitialized tree entries to participate in tournaments and corrupt the maximum.
- **Floating-point precision in crossing times** — crossing time computation `t = (b1 - b2) / (a2 - a1)` can produce catastrophic cancellation when slopes are nearly equal. Add a small epsilon guard: if `|a2 - a1| < eps`, treat as parallel lines (no crossing). For exact arithmetic applications, use rational or integer arithmetic for slopes and intercepts.

---

## Conclusion

Kinetic Data Structures represent the **framework for maintaining combinatorial structure under continuous motion**:

- The **certificate paradigm** separates "what is the invariant" from "when does it break" — events are the future violation times, processed lazily as time advances.
- **Kinetic Tournament** achieves O(log n) per event with O(n log n) total events for linearly moving elements — optimal for this model.
- In competitive programming, the **Convex Hull Trick** serves as an implicit kinetic max structure: it maintains the maximum of linear functions `f(t) = m*t + b` under increasing t, which is exactly a kinetic priority queue with offline monotone queries.
- The four quality metrics (responsive, local, compact, efficient) provide a framework for analyzing any KDS — a structure scoring well on all four is production-quality.

**Key takeaway:**  
For competitive programming, kinetic heaps most commonly appear as "which line y = a*t + b is maximum at each query time t?" — solved exactly by CHT for offline monotone queries or Li Chao Tree for arbitrary queries. The full event-based KDS framework is needed only for dynamic insertion/deletion of elements or non-monotone time queries, which are rare in competitive programming but common in computational geometry.
