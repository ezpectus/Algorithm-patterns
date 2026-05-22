# SOS DP — Sum over Subsets

## Origin & Motivation

**SOS DP** (Sum over Subsets Dynamic Programming) efficiently computes, for every bitmask `mask`, the sum of `f[sub]` over all subsets `sub` of `mask`:

```
dp[mask] = Σ f[sub]   for all sub ⊆ mask
```

The naive approach iterates over all subsets of each mask using the submask enumeration trick, costing **O(3^n)** total (since each bit is in one of three states relative to mask: in sub, in mask but not sub, not in mask). SOS DP reduces this to **O(n * 2^n)** by processing one bit at a time — exactly the same gain that FFT gives over naive convolution.

The name "SOS" and the n * 2^n algorithm were popularized in competitive programming by Codeforces user [x-squared] and others around 2014–2016, though the underlying technique (zeta transform over the subset lattice) appears in combinatorics literature much earlier.

Complexity: **O(n * 2^n)** time, **O(2^n)** space.

---

## Where It Is Used

- AND convolution (FWHT over AND lattice) — already covered as FWHT
- Counting submasks with a property for all masks simultaneously
- Competitive programming: "for each mask, count elements of array A whose bitmask is a subset of mask"
- Maximum XOR or AND over subsets
- DP over subsets where each state depends on sums over all its subsets
- Covering problems: "is every element of the universe covered by mask?"
- Team assignment problems with bitmask constraints

---

## Core Idea

Process the n bit positions one at a time. For bit `i`:

```
If bit i is set in mask:
    dp[mask] += dp[mask without bit i]
```

After processing all n bits, `dp[mask]` = sum of `f[sub]` for all `sub ⊆ mask`.

**Why this works:** Think of it as n rounds of merging. After round `i`, `dp[mask]` contains the sum of `f[sub]` over all subsets `sub` of `mask` that agree with `mask` on bits `i+1, ..., n-1` (the "high" bits already processed). After all n rounds, the constraint is removed and every subset is summed.

---

## Algorithm

### Forward (subset sum): dp[mask] = Σ f[sub], sub ⊆ mask

```
// Initialize: dp[mask] = f[mask] for all masks
for i = 0 to n-1:
    for mask = 0 to 2^n - 1:
        if mask has bit i set:
            dp[mask] += dp[mask ^ (1 << i)]
```

### Reverse (superset sum): dp[mask] = Σ f[sup], sup ⊇ mask

```
for i = 0 to n-1:
    for mask = 0 to 2^n - 1:
        if NOT (mask has bit i set):
            dp[mask] += dp[mask | (1 << i)]
```

### Möbius Inversion (recover f from dp)

If `dp[mask] = Σ f[sub], sub ⊆ mask`, then:

```
// Invert: recover f from dp
for i = 0 to n-1:
    for mask = 0 to 2^n - 1:
        if mask has bit i set:
            dp[mask] -= dp[mask ^ (1 << i)]
// After inversion: dp[mask] = f[mask] (original values restored)
```

This is inclusion-exclusion applied bit by bit.

---

## Variants

| Variant | dp[mask] result | Inner update |
|---|---|---|
| Subset sum (zeta AND) | Σ f[sub], sub ⊆ mask | `dp[mask] += dp[mask ^ (1<<i)]` when bit i set |
| Superset sum (zeta OR) | Σ f[sup], sup ⊇ mask | `dp[mask] += dp[mask\|(1<<i)]` when bit i NOT set |
| Subset max | max f[sub], sub ⊆ mask | Replace `+=` with `= max(...)` |
| Subset min | min f[sub], sub ⊆ mask | Replace `+=` with `= min(...)` |
| Subset AND | AND of f[sub], sub ⊆ mask | Replace `+=` with `&=` |
| Subset OR | OR of f[sub], sub ⊆ mask | Replace `+=` with `\|=` |

---

## Complexity Analysis

| Approach | Time | Space |
|---|---|---|
| Naive submask enumeration | O(3^n) | O(2^n) |
| SOS DP | O(n * 2^n) | O(2^n) |
| Savings | 3^n / (n * 2^n) = (3/2)^n / n | — |

For n=20: naive = 3^20 ≈ 3.5×10^9, SOS = 20 * 10^6 = 2×10^7. ~175× speedup.
For n=25: naive ≈ 8.5×10^11, SOS = 25 * 33.5×10^6 ≈ 8.4×10^8. ~1000× speedup.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// SOS DP — Subset Sum (Zeta transform over AND)
// dp[mask] = sum of f[sub] for all sub ⊆ mask
// In-place: modifies f[]
// ================================================================
void sos_subset_sum(vector<ll>& f, int n) {
    // n = number of bits, array size = 1<<n
    for (int i = 0; i < n; i++)
        for (int mask = 0; mask < (1 << n); mask++)
            if ((mask >> i) & 1)
                f[mask] += f[mask ^ (1 << i)];
}

// ================================================================
// SOS DP — Superset Sum (Zeta transform over OR)
// dp[mask] = sum of f[sup] for all sup ⊇ mask
// ================================================================
void sos_superset_sum(vector<ll>& f, int n) {
    for (int i = 0; i < n; i++)
        for (int mask = 0; mask < (1 << n); mask++)
            if (!((mask >> i) & 1))
                f[mask] += f[mask | (1 << i)];
}

// ================================================================
// MÖBIUS INVERSION — inverse of subset sum
// Recovers original f from the summed array
// ================================================================
void mobius_inversion(vector<ll>& f, int n) {
    for (int i = 0; i < n; i++)
        for (int mask = 0; mask < (1 << n); mask++)
            if ((mask >> i) & 1)
                f[mask] -= f[mask ^ (1 << i)];
}

// ================================================================
// SUBSET MAX — dp[mask] = max f[sub] for sub ⊆ mask
// ================================================================
void sos_subset_max(vector<ll>& f, int n) {
    for (int i = 0; i < n; i++)
        for (int mask = 0; mask < (1 << n); mask++)
            if ((mask >> i) & 1)
                f[mask] = max(f[mask], f[mask ^ (1 << i)]);
}

// ================================================================
// SUPERSET MIN — dp[mask] = min f[sup] for sup ⊇ mask
// ================================================================
void sos_superset_min(vector<ll>& f, int n) {
    for (int i = 0; i < n; i++)
        for (int mask = 0; mask < (1 << n); mask++)
            if (!((mask >> i) & 1))
                f[mask] = min(f[mask], f[mask | (1 << i)]);
}

// ================================================================
// APPLICATION 1: Count arrays elements whose mask is subset of query
// Given array A[] (values in [0, 2^n)), and queries mask_i:
// answer[mask_i] = count of j such that A[j] is a subset of mask_i
// ================================================================
vector<ll> count_subsets(const vector<int>& A, int n, const vector<int>& queries) {
    vector<ll> freq(1 << n, 0);
    for (int x : A) freq[x]++;

    // SOS: freq[mask] = sum of freq[sub] for sub ⊆ mask
    sos_subset_sum(freq, n);

    vector<ll> ans(queries.size());
    for (int i = 0; i < (int)queries.size(); i++)
        ans[i] = freq[queries[i]];
    return ans;
}

// ================================================================
// APPLICATION 2: AND Convolution
// c[k] = Σ a[i]*b[j] where i AND j = k
// ================================================================
vector<ll> and_convolution(vector<ll> a, vector<ll> b, int n) {
    // SOS = zeta transform for AND
    sos_subset_sum(a, n);
    sos_subset_sum(b, n);
    vector<ll> c(1 << n);
    for (int mask = 0; mask < (1 << n); mask++) c[mask] = a[mask] * b[mask];
    // Inverse (Möbius)
    mobius_inversion(c, n);
    return c;
}

// ================================================================
// APPLICATION 3: Maximum OR over subsets
// For each mask, find max value achievable by OR-ing
// some subset of elements that is a superset of mask.
// ================================================================
vector<ll> max_superset(vector<ll> f, int n) {
    sos_superset_min(f, n); // or max depending on direction
    return f;
}

// ================================================================
// APPLICATION 4: Profile DP optimization with SOS
// Given array val[mask] and a condition, compute:
// dp[mask] = max over all sub ⊆ mask of val[sub]
//            where sub satisfies some property
// ================================================================
vector<ll> dp_max_subset(vector<ll>& val, int n) {
    // val[mask] = value of mask if valid, -INF otherwise
    sos_subset_max(val, n);
    return val; // val[mask] = max valid sub ⊆ mask
}

// ================================================================
// NAIVE VERIFICATION — O(3^n), used to verify SOS
// ================================================================
vector<ll> naive_subset_sum(const vector<ll>& f, int n) {
    int N = 1 << n;
    vector<ll> dp(N, 0);
    for (int mask = 0; mask < N; mask++)
        for (int sub = mask; sub > 0; sub = (sub-1) & mask) {
            dp[mask] += f[sub];
        }
    // Handle sub=0 (empty subset)
    for (int mask = 0; mask < N; mask++) dp[mask] += f[0];
    return dp;
}

// ================================================================
// Usage
// ================================================================
int main() {
    int n = 3; // 3 bits, masks 0..7

    // Basic SOS
    {
        printf("=== Basic SOS DP ===\n");
        // f[mask] = mask itself as value
        vector<ll> f = {0, 1, 2, 3, 4, 5, 6, 7};
        printf("Original f: ");
        for (ll x : f) printf("%lld ", x);
        printf("\n");

        auto naive = naive_subset_sum(f, n);
        sos_subset_sum(f, n);

        printf("SOS  dp[mask]: ");
        for (ll x : f) printf("%lld ", x);
        printf("\n");

        printf("Naive dp[mask]: ");
        for (ll x : naive) printf("%lld ", x);
        printf("\n");

        printf("Match: %s\n", f == naive ? "YES" : "NO");
    }

    // Möbius inversion
    {
        printf("\n=== Möbius Inversion ===\n");
        vector<ll> original = {1, 2, 0, 3, 1, 0, 2, 1};
        vector<ll> f = original;
        sos_subset_sum(f, n);
        printf("After SOS: ");
        for (ll x : f) printf("%lld ", x);
        printf("\n");
        mobius_inversion(f, n);
        printf("After Möbius: ");
        for (ll x : f) printf("%lld ", x);
        printf("\n");
        printf("Matches original: %s\n", f == original ? "YES" : "NO");
    }

    // Count subsets query
    {
        printf("\n=== Count Subsets ===\n");
        // Elements: 5(=101), 3(=011), 6(=110), 1(=001)
        vector<int> A = {5, 3, 6, 1};
        vector<int> queries = {7, 5, 3, 0};
        // mask=7(=111): all elements are subsets of 111 → 4
        // mask=5(=101): subsets are 5(101), 1(001) → 2
        // mask=3(=011): subsets are 3(011), 1(001) → 2
        // mask=0(=000): only 0 itself → 0 (none of A[] is 0)

        auto ans = count_subsets(A, n, queries);
        for (int i = 0; i < (int)queries.size(); i++)
            printf("  query mask=%d(%s): count=%lld\n",
                   queries[i],
                   [&]() -> string {
                       string s;
                       for (int b = n-1; b >= 0; b--)
                           s += ((queries[i]>>b)&1)?'1':'0';
                       return s;
                   }().c_str(),
                   ans[i]);
    }

    // AND convolution
    {
        printf("\n=== AND Convolution ===\n");
        // a = [1,1,1,1,0,0,0,0], b = [1,0,1,0,1,0,1,0]
        vector<ll> a = {1,1,1,1,0,0,0,0};
        vector<ll> b = {1,0,1,0,1,0,1,0};
        auto c = and_convolution(a, b, n);
        printf("c[k] = sum a[i]*b[j] where i AND j = k:\n");
        for (int k = 0; k < 8; k++)
            printf("  c[%d] = %lld\n", k, c[k]);
        // c[0]: pairs where i AND j = 0 = {(0,0),(0,2),(0,4),(0,6),(1,0),(2,0),(3,0)}...
    }

    // Subset max
    {
        printf("\n=== Subset Max ===\n");
        vector<ll> f = {0, 5, 3, 0, 8, 0, 0, 0};
        sos_subset_max(f, n);
        printf("max over subsets:\n");
        for (int mask = 0; mask < 8; mask++)
            printf("  mask=%d: max=%lld\n", mask, f[mask]);
        // mask=7(111): max(f[0..7])=8
        // mask=5(101): max(f[0],f[1],f[4],f[5])=max(0,5,8,0)=8
        // mask=3(011): max(f[0],f[1],f[2],f[3])=max(0,5,3,0)=5
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public static class SosDP {

    // dp[mask] = sum of f[sub] for all sub ⊆ mask
    public static void SubsetSum(long[] f, int n) {
        for (int i = 0; i < n; i++)
            for (int mask = 0; mask < (1 << n); mask++)
                if (((mask >> i) & 1) == 1)
                    f[mask] += f[mask ^ (1 << i)];
    }

    // dp[mask] = sum of f[sup] for all sup ⊇ mask
    public static void SupersetSum(long[] f, int n) {
        for (int i = 0; i < n; i++)
            for (int mask = 0; mask < (1 << n); mask++)
                if (((mask >> i) & 1) == 0)
                    f[mask] += f[mask | (1 << i)];
    }

    // Inverse of SubsetSum: recover original f
    public static void MobiusInversion(long[] f, int n) {
        for (int i = 0; i < n; i++)
            for (int mask = 0; mask < (1 << n); mask++)
                if (((mask >> i) & 1) == 1)
                    f[mask] -= f[mask ^ (1 << i)];
    }

    // dp[mask] = max of f[sub] for all sub ⊆ mask
    public static void SubsetMax(long[] f, int n) {
        for (int i = 0; i < n; i++)
            for (int mask = 0; mask < (1 << n); mask++)
                if (((mask >> i) & 1) == 1)
                    f[mask] = Math.Max(f[mask], f[mask ^ (1 << i)]);
    }

    // dp[mask] = min of f[sup] for all sup ⊇ mask
    public static void SupersetMin(long[] f, int n) {
        for (int i = 0; i < n; i++)
            for (int mask = 0; mask < (1 << n); mask++)
                if (((mask >> i) & 1) == 0)
                    f[mask] = Math.Min(f[mask], f[mask | (1 << i)]);
    }

    // AND convolution: c[k] = Σ a[i]*b[j] where i AND j = k
    public static long[] AndConvolution(long[] a, long[] b, int n) {
        var ca = (long[])a.Clone();
        var cb = (long[])b.Clone();
        SubsetSum(ca, n);
        SubsetSum(cb, n);
        var c = new long[1 << n];
        for (int mask = 0; mask < (1<<n); mask++) c[mask] = ca[mask] * cb[mask];
        MobiusInversion(c, n);
        return c;
    }

    // Count elements of A[] whose value is a subset of each query mask
    public static long[] CountSubsets(int[] A, int n, int[] queries) {
        var freq = new long[1 << n];
        foreach (int x in A) freq[x]++;
        SubsetSum(freq, n);
        var ans = new long[queries.Length];
        for (int i = 0; i < queries.Length; i++) ans[i] = freq[queries[i]];
        return ans;
    }

    public static void Main() {
        int n = 3;

        // Basic SOS
        long[] f = {0,1,2,3,4,5,6,7};
        var orig = (long[])f.Clone();
        SubsetSum(f, n);
        Console.Write("SOS dp: "); foreach(var x in f) Console.Write($"{x} "); Console.WriteLine();

        // Inversion
        MobiusInversion(f, n);
        bool match = true;
        for(int i=0;i<f.Length;i++) if(f[i]!=orig[i]) match=false;
        Console.WriteLine($"Möbius inversion restores original: {match}");

        // Count subsets
        var ans = CountSubsets(new[]{5,3,6,1}, n, new[]{7,5,3,0});
        Console.Write("Count subsets: "); foreach(var x in ans) Console.Write($"{x} "); Console.WriteLine();

        // AND convolution
        var c = AndConvolution(new long[]{1,1,1,1,0,0,0,0}, new long[]{1,0,1,0,1,0,1,0}, n);
        Console.Write("AND conv: "); foreach(var x in c) Console.Write($"{x} "); Console.WriteLine();

        // Subset max
        long[] g = {0,5,3,0,8,0,0,0};
        SubsetMax(g, n);
        Console.Write("Subset max: "); foreach(var x in g) Console.Write($"{x} "); Console.WriteLine();
    }
}
```

---

## Why O(n * 2^n) and not O(3^n)

The key insight: instead of iterating all subsets of each mask, process **one bit at a time**.

```
After processing bit 0:
  dp[mask] = Σ f[sub] for sub ⊆ mask, sub agrees with mask on bits 1..n-1

After processing bits 0 and 1:
  dp[mask] = Σ f[sub] for sub ⊆ mask, sub agrees with mask on bits 2..n-1

After processing all n bits:
  dp[mask] = Σ f[sub] for all sub ⊆ mask  (no constraint remaining)
```

Each "processing bit i" step takes O(2^n) — just one pass over all masks. Total: O(n * 2^n).

**Contrast with naive:** For each mask, iterate all its submasks. The total number of (mask, submask) pairs = Σ_{mask} 2^{popcount(mask)} = Σ_{k=0}^{n} C(n,k) * 2^k = (1+2)^n = 3^n.

---

## Submask Enumeration (O(3^n) — when needed)

When SOS is not applicable (non-additive aggregates that can't be built incrementally), enumerate submasks directly:

```cpp
// For each mask, iterate all its submasks in O(2^popcount(mask))
for (int mask = 0; mask < (1<<n); mask++) {
    for (int sub = mask; sub > 0; sub = (sub-1) & mask) {
        // process (mask, sub)
    }
    // Don't forget sub=0 (empty subset)
}
// Total iterations: 3^n
```

The trick `sub = (sub-1) & mask` works because:
- `sub-1` flips the lowest set bit and sets all lower bits
- `& mask` keeps only bits that are in `mask`
- This efficiently enumerates all submasks in decreasing order

---

## Pitfalls

- **Loop order: bit index in outer, mask in inner** — SOS requires the outer loop to be over bit positions (0..n-1) and the inner loop over all masks. Reversing the loops (mask outer, bit inner) computes something different — not the subset sum. This is the single most common implementation bug.
- **Bit condition: update only when bit is set** — for subset sum, update `dp[mask] += dp[mask ^ (1<<i)]` only when bit `i` is set in `mask`. Without the condition check, every mask is updated unconditionally and the result is wrong. For superset sum, the condition is negated (bit i NOT set).
- **In-place vs separate array** — SOS works in-place: `f[mask] += f[mask ^ (1<<i)]` reads and writes the same array. This is correct because the lower mask (`mask ^ (1<<i)`) is always processed before the current mask in the inner loop (since we go from 0 to 2^n-1, and `mask ^ (1<<i) < mask` when bit i is set). No separate "previous layer" array is needed.
- **Möbius inversion exact reversal** — the Möbius inversion is the exact inverse of the subset sum transform: apply the same loop structure but subtract instead of add. After inversion, `f[mask]` returns to its original value. Using the wrong sign (adding instead of subtracting) gives double-counting artifacts.
- **Subset max/min is NOT invertible** — unlike the sum transform, the max/min variants cannot be inverted with a simple undo pass. Once `dp[mask]` is updated to the max of its subsets, the original value is lost. Store the original array separately if both the original and the SOS-max are needed.
- **Array size must be exactly 2^n** — SOS accesses indices from 0 to `2^n - 1`. If the array is declared as `vector<ll> f(n)` (size n) instead of `vector<ll> f(1<<n)` (size 2^n), all accesses beyond index n-1 are out of bounds. Always allocate `1<<n` elements.

---

## Complexity Summary

| n | 3^n (naive) | n * 2^n (SOS) | Ratio |
|---|---|---|---|
| 10 | 59,049 | 10,240 | 5.8× |
| 15 | 14,348,907 | 491,520 | 29× |
| 20 | 3,486,784,401 | 20,971,520 | 166× |
| 25 | ~847 billion | 838,860,800 | ~1010× |

---

## Conclusion

SOS DP is the **canonical O(n * 2^n) technique for subset aggregation problems**:

- Replaces O(3^n) submask enumeration for additive (sum, count), extremal (max, min), and bitwise (AND, OR) aggregates.
- The outer-bit, inner-mask loop structure processes exactly n * 2^n cells — one pass per bit, one operation per mask per bit.
- Möbius inversion exactly undoes the subset sum, enabling AND convolution and related Dirichlet-series-style computations on subsets.
- The same structure underlies FWHT (AND/OR convolution) and the SOS DP optimization used in competitive programming subset DP problems.

**Key takeaway:**  
SOS DP is three lines: outer loop `for i in 0..n`, inner loop `for mask in 0..2^n`, update `if bit i set in mask: dp[mask] += dp[mask ^ (1<<i)]`. The outer-inner order and the bit-set condition are the only two things that can go wrong. Everything else — initialization, query, inversion — follows mechanically from this core.
