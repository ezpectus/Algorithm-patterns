# Amortized Analysis — Aggregate, Accounting, and Potential Methods

## Origin & Motivation

**Amortized analysis** bounds the *average* cost per operation over a worst-case sequence of operations, even when individual operations can be expensive. It was formalized as a systematic technique by Tarjan in 1985, building on earlier ad-hoc arguments (e.g. dynamic array doubling).

**The problem it solves:** Some data structures have operations whose *worst-case single-operation cost* is high (e.g. resizing a dynamic array costs O(n)), but whose cost *averaged over any sequence of operations* is low (O(1)). Worst-case-per-operation analysis would conclude the structure has O(n) per operation — true but overly pessimistic, since expensive operations are rare and "pay for themselves" via the cheap operations that preceded them. Amortized analysis captures the tighter, true bound.

**Three techniques, same goal:** prove `Σ actual_cost(op_i) ≤ Σ amortized_cost(op_i)` for any sequence, where the amortized costs are individually simple bounds (often constants).

1. **Aggregate method** — directly bound the total cost of `n` operations, then divide by `n`.
2. **Accounting (banker's) method** — assign each operation a fixed "charge"; cheap operations bank surplus credit, expensive operations withdraw banked credit. Requires the credit balance to never go negative.
3. **Potential method** — define a potential function `Φ` over the data structure's state; amortized cost = actual cost + `ΔΦ`. The most general and mathematically clean technique.

---

## Where It Is Used

- Dynamic arrays (`std::vector`, `ArrayList`): doubling strategy gives O(1) amortized push_back
- Hash tables: amortized O(1) insert despite occasional full-table rehashing
- Splay trees: O(log n) amortized per operation despite no balance invariant
- Union-Find with path compression: O(log* n) or O(1) amortized per operation
- Fibonacci heaps: O(1) amortized insert/decrease-key, O(log n) amortized extract-min
- Two-stack (or two-pointer) queue implementations: O(1) amortized enqueue/dequeue
- Binary counters and binary representations of numbers: O(1) amortized increment

---

## The Aggregate Method

### Idea

Compute the total cost `T(n)` of any sequence of `n` operations (worst case), then the amortized cost per operation is `T(n)/n`.

### Example: Dynamic Array Doubling

```
push_back(): if array is full, double capacity (copy all elements), then append.

Total cost of n pushes (starting from capacity 1):
  Resizes happen at sizes 1, 2, 4, 8, ..., up to n.
  Resize at size 2^k costs O(2^k) (copying 2^k elements).
  
  Total copying cost = 1 + 2 + 4 + ... + n ≤ 2n   (geometric series)
  Total append cost  = n   (one O(1) append per push)
  
  T(n) = 2n + n = 3n
  Amortized cost per push = T(n)/n = 3 = O(1)
```

### Example: Binary Counter Increment

```
increment(): flip trailing 1s to 0, flip the first 0 to 1.

Bit i flips on increment k if and only if k is a multiple of 2^i
(after the flip-to-0 cascade). Over n increments:
  bit 0 flips n times
  bit 1 flips n/2 times
  bit 2 flips n/4 times
  ...
  
  T(n) = n + n/2 + n/4 + ... < 2n
  Amortized cost per increment = T(n)/n < 2 = O(1)
```

**Limitation of the aggregate method:** it gives a single number for ALL operations averaged together — it cannot assign *different* amortized costs to different operation types (e.g. "insert costs O(1) amortized, delete costs O(log n) amortized"). The accounting and potential methods can.

---

## The Accounting (Banker's) Method

### Idea

Assign each operation an **amortized cost** (a fixed charge, possibly different from its actual cost). When `amortized > actual`, the surplus is **banked as credit** on the data structure (often attached to specific elements). When `amortized < actual`, the deficit is **paid for using previously banked credit**. The key invariant: **the credit balance must never go negative** at any point in any sequence.

```
Total actual cost = Total amortized cost - final credit balance
                   <= Total amortized cost    (since credit >= 0 always)
```

### Example: Stack with MultiPop

```
push(x): actual cost 1.  Charge 2 (amortized).
         1 pays for the push itself; 1 is BANKED on element x as its
         "future pop" credit.

pop():   actual cost 1.  Charge 0 (amortized) — paid using x's banked credit.

multipop(k): actual cost = number of elements actually popped (≤ k).
             Each popped element WITHDRAWS its own banked credit (1 each).
             Charge 0 (amortized) for the whole multipop call.

Invariant: every element on the stack has exactly 1 unit of banked credit
(from its push). It can only be popped once, so the credit exactly covers
the cost of that eventual pop. Credit balance = (number of elements
currently on stack) >= 0 always.
```

**Result:** `push` is O(1) amortized (charge 2), `pop`/`multipop` are O(1) amortized (charge 0) — even though a single `multipop(1000000)` call can have actual cost 1,000,000.

---

## The Potential Method

### Idea

Define a **potential function** `Φ: states → ℝ≥0` mapping the data structure's state to a non-negative real number (think of it as "stored energy"). For a sequence of operations transforming state `D_0 → D_1 → ... → D_n`:

```
amortized_cost(op_i) = actual_cost(op_i) + Φ(D_i) - Φ(D_{i-1})

Summing telescopes:
  Σ amortized_cost = Σ actual_cost + Φ(D_n) - Φ(D_0)

If Φ(D_0) = 0 and Φ(D_i) >= 0 for all i:
  Σ actual_cost = Σ amortized_cost - Φ(D_n) <= Σ amortized_cost
```

This is the **most general** of the three methods — the accounting method is the special case where `Φ(D) = ` total banked credit in state `D`.

### Example: Dynamic Array via Potential

```
Φ(D) = 2*size - capacity

When the array is exactly full (size = capacity): Φ = size (≥0).
When the array was just doubled (size = capacity/2): Φ = 0.
Φ is always in [0, size] — non-negative, as required.

push_back() WITHOUT resize:
  actual = 1
  ΔΦ = (2*(size+1) - capacity) - (2*size - capacity) = 2
  amortized = 1 + 2 = 3

push_back() WITH resize (size == capacity before):
  actual = 1 + size   (1 for append, size for copying old elements)
  capacity doubles: capacity_new = 2*capacity_old = 2*size
  Φ_before = 2*size - size = size
  Φ_after  = 2*(size+1) - 2*size = 2
  ΔΦ = 2 - size
  amortized = (1+size) + (2-size) = 3

Both cases give amortized cost EXACTLY 3 — constant, regardless of resize.
```

### Example: Binary Counter via Potential

```
Φ(D) = number of 1-bits in the counter

increment() that flips t trailing 1s to 0, then flips one 0 to 1:
  actual = t + 1
  Φ_before = (number of 1s before) = ... includes the t ones being flipped
  Φ_after  = Φ_before - t + 1   (t ones removed, 1 one added)
  ΔΦ = 1 - t
  amortized = (t+1) + (1-t) = 2

Amortized cost is EXACTLY 2 — constant, regardless of how many bits cascade.
```

### Example: Two-Stack Queue via Potential

```
Φ(D) = |in_stack|   (number of elements not yet transferred to out_stack)

enqueue(x): push x onto in_stack.
  actual = 1
  ΔΦ = +1
  amortized = 1 + 1 = 2

dequeue() when out_stack is non-empty: pop from out_stack.
  actual = 1
  ΔΦ = 0
  amortized = 1

dequeue() when out_stack is empty (triggers full transfer of k elements):
  actual = k (transfer) + 1 (final pop) = k+1
  ΔΦ = (0) - (k) = -k        [in_stack had k elements, now has 0]
  amortized = (k+1) + (-k) = 1

Both dequeue cases give amortized cost EXACTLY 1 — the potential drop
during a transfer exactly cancels the transfer's actual cost.
```

---

## Complexity Summary

| Structure | Operation | Amortized Cost | Technique |
|---|---|---|---|
| Dynamic array (doubling) | push_back | O(1) [exactly 3] | Aggregate / Potential |
| Binary counter | increment | O(1) [exactly 2] | Aggregate / Potential |
| Stack with multipop | push | O(1) [charge 2] | Accounting |
| Stack with multipop | pop / multipop(k) | O(1) [charge 0] | Accounting |
| Two-stack queue | enqueue | O(1) [exactly 2] | Potential |
| Two-stack queue | dequeue | O(1) [exactly 1] | Potential |
| Union-Find (path compression + union by rank) | find / union | O(log* n) or O(α(n)) | Potential (rank-based) |
| Splay tree | any operation | O(log n) | Potential (access lemma) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// DYNAMIC ARRAY (doubling) — POTENTIAL METHOD
// Potential: Phi = 2*size - capacity
// Amortized cost of push_back = actual_cost + Delta(Phi) = 3 (constant)
// ================================================================
struct DynamicArray {
    int size_ = 0, capacity_ = 1;
    long long total_actual = 0;

    int potential() const { return 2*size_ - capacity_; }

    // Returns {actual_cost, amortized_cost}
    pair<int,int> push_back() {
        int phi_before = potential();
        int actual = 1; // base cost: write the new element
        if (size_ == capacity_) {
            actual += size_; // copy all existing elements during resize
            capacity_ *= 2;
        }
        size_++;
        int phi_after = potential();
        int amortized = actual + (phi_after - phi_before);
        total_actual += actual;
        return {actual, amortized};
    }
};

// ================================================================
// BINARY COUNTER INCREMENT — POTENTIAL METHOD
// Potential: Phi = number of 1-bits in the counter
// Amortized cost of increment = actual_cost + Delta(Phi) = 2 (constant)
// ================================================================
struct BinaryCounter {
    vector<int> bits;
    long long total_actual = 0;

    int potential() const { return (int)count(bits.begin(), bits.end(), 1); }

    // Returns {actual_flips, amortized_cost}
    pair<int,int> increment() {
        int phi_before = potential();
        int flips = 0, i = 0;
        while (i < (int)bits.size() && bits[i] == 1) { bits[i] = 0; flips++; i++; }
        if (i == (int)bits.size()) bits.push_back(1);
        else bits[i] = 1;
        flips++;
        int phi_after = potential();
        int amortized = flips + (phi_after - phi_before);
        total_actual += flips;
        return {flips, amortized};
    }
};

// ================================================================
// STACK WITH MULTIPOP — ACCOUNTING (BANKER'S) METHOD
// push charges 2: 1 pays for the push itself, 1 is BANKED as credit
// on that element for its eventual single pop.
// pop/multipop withdraws 1 credit per element actually popped.
// Credit balance must never go negative.
// ================================================================
struct StackMultiPop {
    vector<int> data;
    long long credit_balance = 0;
    long long total_actual = 0;

    // push: actual cost 1, charges 2 (1 used now, 1 banked)
    int push(int val) {
        data.push_back(val);
        total_actual += 1;
        credit_balance += 1; // bank the prepaid credit
        return 1;
    }
    // multipop(k): actual cost = number of elements actually popped
    // each popped element withdraws its banked credit
    int multipop(int k) {
        int cnt = 0;
        while (!data.empty() && cnt < k) { data.pop_back(); cnt++; }
        credit_balance -= cnt;
        total_actual += cnt;
        return cnt;
    }
};

// ================================================================
// TWO-STACK QUEUE — POTENTIAL METHOD
// Potential: Phi = |in_stack| (elements not yet transferred)
// enqueue: amortized = 2 (constant)
// dequeue: amortized = 1 (constant), even when it triggers a full transfer
// ================================================================
struct TwoStackQueue {
    vector<int> in_stack, out_stack;
    long long total_actual = 0;

    int potential() const { return (int)in_stack.size(); }

    // Returns {actual_cost, amortized_cost}
    pair<int,int> enqueue(int val) {
        int phi_before = potential();
        in_stack.push_back(val);
        int actual = 1;
        int phi_after = potential();
        int amortized = actual + (phi_after - phi_before);
        total_actual += actual;
        return {actual, amortized};
    }
    // Returns {actual_cost, amortized_cost, value} ; value=-1 if empty
    tuple<int,int,int> dequeue() {
        int phi_before = potential();
        int actual = 0;
        if (out_stack.empty()) {
            while (!in_stack.empty()) {
                out_stack.push_back(in_stack.back());
                in_stack.pop_back();
                actual++;
            }
        }
        int val = -1;
        if (!out_stack.empty()) { val = out_stack.back(); out_stack.pop_back(); actual++; }
        int phi_after = potential();
        int amortized = actual + (phi_after - phi_before);
        total_actual += actual;
        return {actual, amortized, val};
    }
};

// ================================================================
// Usage + stress tests
// ================================================================
int main() {
    // ---- Dynamic Array: aggregate + potential ----
    {
        printf("=== Dynamic Array (doubling) ===\n");
        DynamicArray arr;
        printf("First 10 pushes (actual, amortized):\n  ");
        for (int i = 0; i < 10; i++) {
            auto [actual, amort] = arr.push_back();
            printf("(%d,%d) ", actual, amort);
        }
        printf("\n");

        int errors = 0;
        for (int n : {1,2,3,7,10,100,1000,100000}) {
            DynamicArray a;
            int max_amort = 0;
            for (int i = 0; i < n; i++) {
                auto [act,am] = a.push_back();
                max_amort = max(max_amort, am);
            }
            bool ok = (max_amort <= 3 && a.total_actual <= 3LL*n);
            if (!ok) errors++;
            printf("n=%7d  total_actual=%8lld  <=3n=%8lld  max_amortized=%d\n",
                   n, a.total_actual, 3LL*n, max_amort);
        }
        printf("All amortized costs <= 3, total <= 3n: %s\n", errors==0?"OK":"FAIL");
    }

    // ---- Binary Counter: aggregate + potential ----
    {
        printf("\n=== Binary Counter Increment ===\n");
        BinaryCounter bc;
        printf("First 8 increments (flips, amortized):\n  ");
        for (int i = 0; i < 8; i++) {
            auto [flips,amort] = bc.increment();
            printf("(%d,%d) ", flips, amort);
        }
        printf("\n");

        int errors = 0;
        for (int n : {1,7,8,15,16,1000,1000000}) {
            BinaryCounter c;
            int max_amort = 0;
            for (int i = 0; i < n; i++) {
                auto [fl,am] = c.increment();
                max_amort = max(max_amort, am);
            }
            bool ok = (max_amort <= 2 && c.total_actual <= 2LL*n);
            if (!ok) errors++;
            printf("n=%8d  total_flips=%9lld  <=2n=%9lld  max_amortized=%d\n",
                   n, c.total_actual, 2LL*n, max_amort);
        }
        printf("All amortized costs <= 2, total <= 2n: %s\n", errors==0?"OK":"FAIL");
    }

    // ---- Stack with MultiPop: accounting method ----
    {
        printf("\n=== Stack with MultiPop (accounting method) ===\n");
        srand(99); int errors = 0;
        for (int trial = 0; trial < 500; trial++) {
            StackMultiPop st;
            int n_pushes = 0;
            int ops = 10 + rand()%200;
            for (int i = 0; i < ops; i++) {
                if (rand()%2 == 0) { st.push(rand()%100); n_pushes++; }
                else st.multipop(1+rand()%10);
                if (st.credit_balance < 0) errors++; // credit must never go negative
            }
            if (st.total_actual > 2LL*n_pushes) errors++; // aggregate bound
        }
        printf("Result: %s  (credit_balance>=0 always, total_actual<=2*pushes)\n",
               errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- Two-Stack Queue: potential method ----
    {
        printf("\n=== Two-Stack Queue (potential method) ===\n");
        TwoStackQueue q;
        printf("enqueue 1,2,3; dequeue x3; enqueue 4,5; dequeue x2:\n");
        for (int v : {1,2,3}) { auto [a,am]=q.enqueue(v); printf("  enq(%d): actual=%d amortized=%d\n",v,a,am); }
        for (int i=0;i<3;i++) { auto [a,am,v]=q.dequeue(); printf("  deq()=%d: actual=%d amortized=%d\n",v,a,am); }
        for (int v : {4,5}) { auto [a,am]=q.enqueue(v); printf("  enq(%d): actual=%d amortized=%d\n",v,a,am); }
        for (int i=0;i<2;i++) { auto [a,am,v]=q.dequeue(); printf("  deq()=%d: actual=%d amortized=%d\n",v,a,am); }

        srand(7); int errors = 0;
        for (int trial = 0; trial < 500; trial++) {
            TwoStackQueue tq;
            queue<int> ref;
            int n_enq=0, n_deq=0;
            int ops = 10 + rand()%300;
            for (int i = 0; i < ops; i++) {
                if (rand()%2==0 || ref.empty()) {
                    int v = rand()%1000;
                    auto [a,am] = tq.enqueue(v);
                    ref.push(v); n_enq++;
                    if (am != 2) errors++; // enqueue amortized must be exactly 2
                } else {
                    auto [a,am,v] = tq.dequeue();
                    int expected = ref.front(); ref.pop();
                    n_deq++;
                    if (v != expected) errors++; // correctness
                    if (am != 1) errors++;       // dequeue amortized must be exactly 1
                }
            }
            long long bound = 2LL*n_enq + 1LL*n_deq;
            if (tq.total_actual > bound) errors++; // aggregate bound
        }
        printf("Result: %s  (enq amortized==2, deq amortized==1, correctness, aggregate bound)\n",
               errors==0?"OK":("FAIL "+to_string(errors)).c_str());
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
// DYNAMIC ARRAY (doubling) — POTENTIAL METHOD — C#
// ================================================================
public class DynamicArray
{
    public int Size = 0, Capacity = 1;
    public long TotalActual = 0;

    public int Potential() => 2*Size - Capacity;

    public (int actual, int amortized) PushBack()
    {
        int phiBefore = Potential();
        int actual = 1;
        if (Size == Capacity) { actual += Size; Capacity *= 2; }
        Size++;
        int phiAfter = Potential();
        int amortized = actual + (phiAfter - phiBefore);
        TotalActual += actual;
        return (actual, amortized);
    }
}

// ================================================================
// BINARY COUNTER — POTENTIAL METHOD — C#
// ================================================================
public class BinaryCounter
{
    public List<int> Bits = new();
    public long TotalActual = 0;

    public int Potential() => Bits.Count(b => b==1);

    public (int flips, int amortized) Increment()
    {
        int phiBefore = Potential();
        int flips = 0, i = 0;
        while (i < Bits.Count && Bits[i]==1) { Bits[i]=0; flips++; i++; }
        if (i == Bits.Count) Bits.Add(1);
        else Bits[i] = 1;
        flips++;
        int phiAfter = Potential();
        int amortized = flips + (phiAfter - phiBefore);
        TotalActual += flips;
        return (flips, amortized);
    }
}

// ================================================================
// STACK WITH MULTIPOP — ACCOUNTING METHOD — C#
// ================================================================
public class StackMultiPop
{
    private readonly List<int> _data = new();
    public long CreditBalance = 0;
    public long TotalActual = 0;

    public int Push(int val)
    {
        _data.Add(val);
        TotalActual += 1;
        CreditBalance += 1;
        return 1;
    }

    public int MultiPop(int k)
    {
        int cnt = 0;
        while (_data.Count > 0 && cnt < k) { _data.RemoveAt(_data.Count-1); cnt++; }
        CreditBalance -= cnt;
        TotalActual += cnt;
        return cnt;
    }
}

// ================================================================
// TWO-STACK QUEUE — POTENTIAL METHOD — C#
// ================================================================
public class TwoStackQueue
{
    private readonly Stack<int> _in = new(), _out = new();
    public long TotalActual = 0;

    public int Potential() => _in.Count;

    public (int actual, int amortized) Enqueue(int val)
    {
        int phiBefore = Potential();
        _in.Push(val);
        int actual = 1;
        int phiAfter = Potential();
        int amortized = actual + (phiAfter - phiBefore);
        TotalActual += actual;
        return (actual, amortized);
    }

    public (int actual, int amortized, int value) Dequeue()
    {
        int phiBefore = Potential();
        int actual = 0;
        if (_out.Count == 0)
            while (_in.Count > 0) { _out.Push(_in.Pop()); actual++; }
        int val = -1;
        if (_out.Count > 0) { val = _out.Pop(); actual++; }
        int phiAfter = Potential();
        int amortized = actual + (phiAfter - phiBefore);
        TotalActual += actual;
        return (actual, amortized, val);
    }
}

public static class Program
{
    public static void Main()
    {
        // Dynamic array
        var arr = new DynamicArray();
        Console.Write("Dynamic array first 10 pushes: ");
        for (int i=0;i<10;i++) { var (a,am)=arr.PushBack(); Console.Write($"({a},{am}) "); }
        Console.WriteLine();

        // Binary counter
        var bc = new BinaryCounter();
        Console.Write("Binary counter first 8 increments: ");
        for (int i=0;i<8;i++) { var (f,am)=bc.Increment(); Console.Write($"({f},{am}) "); }
        Console.WriteLine();

        // Two-stack queue
        var q = new TwoStackQueue();
        foreach (int v in new[]{1,2,3}) { var (a,am)=q.Enqueue(v); Console.WriteLine($"enq({v}): actual={a} amortized={am}"); }
        for (int i=0;i<3;i++) { var (a,am,v)=q.Dequeue(); Console.WriteLine($"deq()={v}: actual={a} amortized={am}"); }
    }
}
```

---

## Comparing the Three Methods on One Example

For the **dynamic array doubling** problem, all three methods reach the same conclusion (O(1) amortized) via different reasoning:

```
AGGREGATE:
  Sum the total cost of n pushes directly: 1+2+4+...+n + n ≤ 3n.
  Divide by n: amortized = 3.
  (Simple, but only gives ONE number for the whole sequence — can't
   distinguish "this push is cheap, that one is expensive".)

ACCOUNTING:
  Charge each push 3 credits: 1 pays for the append, 2 are banked
  on the newly-added element.
  When a resize happens at size s (doubling to 2s), exactly s elements
  must be copied — and the most recent s/2 elements each have 2 banked
  credits, totaling s credits, exactly covering the resize cost.
  (Requires finding the right "credit per element" — here 2.)

POTENTIAL:
  Phi = 2*size - capacity.
  Compute amortized = actual + Delta(Phi) directly for both cases
  (resize / no resize) — both give exactly 3.
  (Most systematic: no need to reason about "where credit comes from",
   just compute Delta(Phi) mechanically.)
```

The potential method is generally preferred for complex structures (splay trees, Fibonacci heaps) because the potential function can be defined directly from structural invariants (subtree sizes, rank, marked nodes) without needing to invent a credit-flow story.

---

## Pitfalls

- **Potential function must be non-negative for ALL reachable states** — if `Φ(D) < 0` is possible, the bound `Σ actual ≤ Σ amortized` can fail (the telescoping sum subtracts a negative final potential, which *increases* the right side, but if `Φ` goes negative mid-sequence the per-step amortized costs may not even be valid). Always verify `Φ(D) ≥ 0` for every reachable state, not just the typical ones.
- **Φ(D_0) = 0 is required for the cleanest bound** — if the initial potential is nonzero, the bound becomes `Σ actual ≤ Σ amortized - Φ(D_n) + Φ(D_0)`, which is still valid but the "total actual ≤ total amortized" simplification requires `Φ(D_0) = 0`. For an empty data structure this is usually natural (e.g. empty array has size=0).
- **Accounting method: credit must be tied to a specific responsibility** — banking credit "in general" without specifying which future operation it pays for makes it easy to accidentally double-spend the same credit. The stack multipop example bank exactly 1 credit per pushed element, used by exactly 1 future pop of that element — this one-to-one correspondence is what guarantees the balance never goes negative.
- **Aggregate method hides per-operation costs** — the aggregate method proves `T(n)/n = O(1)` for the *total* sequence, but cannot conclude "every individual push is O(1) amortized" in a way that composes with other amortized bounds. If your algorithm interleaves operations from two different amortized structures, use the potential method, which gives a true per-operation amortized cost.
- **Binary counter potential must count CURRENT 1-bits, not lifetime flips** — `Φ(D)` is a function of the *current state* of the counter, not a running total. Confusing "number of 1-bits right now" with "total flips so far" breaks the telescoping argument entirely, since `Φ` would then be monotonically non-decreasing and never able to "give back" stored potential.
- **Resize factor matters for the constant** — doubling capacity gives amortized O(1) with constant 3; using a smaller growth factor (e.g. capacity += 1, linear growth) gives amortized **O(n)** per push, not O(1) — the geometric series argument requires *exponential* growth. Always verify the growth factor is a fixed multiplicative constant > 1.
- **Real-world deletion can break amortized array guarantees** — if a dynamic array also *shrinks* (halves capacity when size drops below capacity/4, say), the potential function must account for both growth and shrinkage. A naive combination of "double on overflow, halve on underflow" can cause "thrashing" (alternating resize at the boundary) unless the shrink threshold is set with enough hysteresis (e.g. halve only when size < capacity/4, not capacity/2).

---

## Conclusion

Amortized analysis provides **tight average-case bounds with worst-case guarantees** — the key insight that expensive operations are necessarily rare, paid for by the cheap operations that built up the structure:

- The **aggregate method** is simplest: sum the total cost directly. Best for quick sanity checks but gives only one aggregate number.
- The **accounting method** assigns intuitive "credit" charges per operation type, requiring the credit balance to stay non-negative — useful when there's a natural notion of "this operation pays for that future operation".
- The **potential method** is the most general and systematic: define `Φ` from structural invariants, then amortized cost = actual + `ΔΦ` falls out mechanically. It is the standard tool for analyzing self-adjusting structures (splay trees, Fibonacci heaps) where no simple credit story exists.

**Key takeaway:** all three methods prove the same underlying fact — `Σ actual_cost ≤ Σ amortized_cost` — via different bookkeeping. When in doubt, default to the potential method: find a function of the data structure's state that increases right before expensive operations and decreases right after, then `actual + ΔΦ` will be constant (or otherwise simple) by construction.
