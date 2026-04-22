# Fast Walsh-Hadamard Transform (FWHT) — XOR / AND / OR Convolution

## Origin & Motivation

The Fast Walsh-Hadamard Transform (FWHT) is the analogue of the Fast Fourier Transform (FFT) for boolean/bitwise convolutions. Where FFT computes polynomial multiplication (convolution over addition `(i+j)`), FWHT computes convolution over bitwise operations:

- **XOR convolution:** `c[k] = Σ a[i] * b[j]` where `i XOR j = k`
- **AND convolution:** `c[k] = Σ a[i] * b[j]` where `i AND j = k`
- **OR convolution:** `c[k] = Σ a[i] * b[j]` where `i OR j = k`

The naive computation of each costs O(4^n) for arrays of size 2^n. FWHT reduces this to **O(n * 2^n)** using a divide-and-conquer structure identical in form to FFT but over GF(2) or the boolean lattice.

The Hadamard matrix of order 2^n is:
```
H_1 = [1  1]    H_n = H_1 ⊗ H_{n-1}  (Kronecker product)
      [1 -1]
```

Applying H_n to a vector corresponds to evaluating the function at all 2^n "frequency points" simultaneously in O(n * 2^n).

Complexity: **O(n * 2^n)** = **O(N log N)** where N = 2^n.

---

## Where It Is Used

- Competitive programming: XOR/AND/OR convolution on arrays indexed by subsets
- SOS DP (Sum over Subsets) — equivalent to AND/OR convolution
- Counting pairs with XOR = k (XOR convolution of frequency arrays)
- Subset sum convolution (combining XOR + popcount constraint)
- AI/ML: Walsh-Hadamard transform for feature hashing, locality-sensitive hashing
- Error-correcting codes: Reed-Muller codes use Hadamard matrices
- Quantum computing: Hadamard gate on multiple qubits = FWHT

---

## Three Convolution Types

### XOR Convolution

```
c[k] = Σ_{i XOR j = k} a[i] * b[j]

Transform: WHT(a)[k] = Σ_i (-1)^{popcount(i AND k)} * a[i]
Inverse: IWHT = (1/2^n) * WHT  (self-inverse up to scaling)

Algorithm:
  WHT(a), WHT(b)
  c_hat[i] = a_hat[i] * b_hat[i]   (pointwise multiply)
  c = IWHT(c_hat)
```

### AND Convolution (Subset Sum / SOS)

```
c[k] = Σ_{i AND j = k} a[i] * b[j]

Transform: a_hat[k] = Σ_{k SUBSET i} a[i]   (sum over supersets)
           (OR: a_hat[k] = Σ_{i SUBSET k} a[i] — depends on convention)

Convention used here: AND conv uses "sum over subsets" (zeta transform over AND)
a_hat[mask] = Σ_{sub SUBSET mask} a[sub]

Inverse: Möbius inversion (inclusion-exclusion)
```

### OR Convolution (Superset Sum)

```
c[k] = Σ_{i OR j = k} a[i] * b[j]

Transform: a_hat[k] = Σ_{k SUBSET i} a[i]   (sum over supersets)
           OR: a_hat[k] = Σ_{i SUPERSET k} a[i]

Inverse: Möbius inversion over OR lattice
```

---

## Butterfly Operations

Each transform reduces to O(log N) layers of "butterfly" operations:

### XOR butterfly (Hadamard butterfly):
```
for step = 1, 2, 4, ..., N/2:
    for i = 0, 2*step, 4*step, ...:
        for j = i, i+1, ..., i+step-1:
            u = a[j]
            v = a[j + step]
            a[j]        = u + v
            a[j + step] = u - v
```

Inverse XOR: same butterfly but divide by 2 at each level (or divide by N at the end).

### AND butterfly (Zeta transform over AND):
```
for i = 0 to n-1:        // for each bit
    for mask = 0 to N-1:
        if NOT (mask & (1<<i)):
            a[mask | (1<<i)] += a[mask]
            // equivalently: a[supermask] += a[submask]
```

### OR butterfly (Zeta transform over OR):
```
for i = 0 to n-1:
    for mask = 0 to N-1:
        if mask & (1<<i):
            a[mask ^ (1<<i)] += a[mask]   // a[submask] += a[supermask]
            // equivalently: a[mask] += a[mask ^ (1<<i)]
```

---

## Complexity Analysis

| Transform | Time | Space | In-place |
|---|---|---|---|
| WHT (XOR) | O(N log N) | O(N) | Yes |
| Zeta/Möbius AND | O(N log N) | O(N) | Yes |
| Zeta/Möbius OR | O(N log N) | O(N) | Yes |
| XOR convolution (full) | O(N log N) | O(N) | — |
| AND/OR convolution (full) | O(N log N) | O(N) | — |
| Subset sum convolution | O(N log^2 N) | O(N log N) | — |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

const int MOD = 998244353;

// ================================================================
// XOR CONVOLUTION — Walsh-Hadamard Transform
// ================================================================

// In-place WHT (Walsh-Hadamard Transform)
// invert=false: forward transform
// invert=true:  inverse transform (divides by N at end)
void wht_xor(vector<ll>& a, bool invert) {
    int n = a.size();
    for (int step = 1; step < n; step <<= 1) {
        for (int i = 0; i < n; i += step << 1) {
            for (int j = i; j < i + step; j++) {
                ll u = a[j], v = a[j + step];
                a[j]        = u + v;
                a[j + step] = u - v;
            }
        }
    }
    if (invert) {
        for (ll& x : a) x /= n;
    }
}

// XOR convolution: c[k] = sum_{i XOR j = k} a[i]*b[j]
vector<ll> conv_xor(vector<ll> a, vector<ll> b) {
    int n = a.size();
    wht_xor(a, false);
    wht_xor(b, false);
    for (int i = 0; i < n; i++) a[i] *= b[i];
    wht_xor(a, true);
    return a;
}

// XOR convolution modulo MOD (avoids large intermediate values)
void wht_xor_mod(vector<ll>& a, bool invert, ll mod) {
    int n = a.size();
    ll inv2 = (mod + 1) / 2; // works when mod is odd prime
    for (int step = 1; step < n; step <<= 1) {
        for (int i = 0; i < n; i += step << 1) {
            for (int j = i; j < i + step; j++) {
                ll u = a[j], v = a[j + step];
                a[j]        = (u + v) % mod;
                a[j + step] = (u - v + mod) % mod;
            }
        }
    }
    if (invert) {
        // Multiply by inverse of n = 2^log2(n)
        ll inv_n = 1;
        for (int i = 0; i < __lg(n); i++) inv_n = inv_n * inv2 % mod;
        for (ll& x : a) x = x * inv_n % mod;
    }
}

vector<ll> conv_xor_mod(vector<ll> a, vector<ll> b, ll mod) {
    int n = a.size();
    wht_xor_mod(a, false, mod);
    wht_xor_mod(b, false, mod);
    for (int i = 0; i < n; i++) a[i] = a[i] * b[i] % mod;
    wht_xor_mod(a, true, mod);
    return a;
}

// ================================================================
// AND CONVOLUTION — Zeta/Möbius transform over AND lattice
// a_hat[mask] = sum_{submask of mask} a[submask]
// ================================================================
void zeta_and(vector<ll>& a) {
    int n = __lg(a.size()); // log2
    for (int i = 0; i < n; i++)
        for (int mask = 0; mask < (int)a.size(); mask++)
            if (mask & (1 << i))
                a[mask] += a[mask ^ (1 << i)];
}

// Inverse: Möbius inversion (inclusion-exclusion)
void mobius_and(vector<ll>& a) {
    int n = __lg(a.size());
    for (int i = 0; i < n; i++)
        for (int mask = 0; mask < (int)a.size(); mask++)
            if (mask & (1 << i))
                a[mask] -= a[mask ^ (1 << i)];
}

// AND convolution: c[k] = sum_{i AND j = k} a[i]*b[j]
vector<ll> conv_and(vector<ll> a, vector<ll> b) {
    zeta_and(a);
    zeta_and(b);
    for (int i = 0; i < (int)a.size(); i++) a[i] *= b[i];
    mobius_and(a);
    return a;
}

// ================================================================
// OR CONVOLUTION — Zeta/Möbius transform over OR lattice
// a_hat[mask] = sum_{superset of mask} a[superset]
// (Alternatively: a_hat[mask] = sum_{mask subset of s} a[s])
// ================================================================
void zeta_or(vector<ll>& a) {
    int n = __lg(a.size());
    for (int i = 0; i < n; i++)
        for (int mask = 0; mask < (int)a.size(); mask++)
            if (!(mask & (1 << i)))
                a[mask | (1 << i)] += a[mask];
}

void mobius_or(vector<ll>& a) {
    int n = __lg(a.size());
    for (int i = 0; i < n; i++)
        for (int mask = 0; mask < (int)a.size(); mask++)
            if (!(mask & (1 << i)))
                a[mask | (1 << i)] -= a[mask];
}

// OR convolution: c[k] = sum_{i OR j = k} a[i]*b[j]
vector<ll> conv_or(vector<ll> a, vector<ll> b) {
    zeta_or(a);
    zeta_or(b);
    for (int i = 0; i < (int)a.size(); i++) a[i] *= b[i];
    mobius_or(a);
    return a;
}

// ================================================================
// SUBSET SUM CONVOLUTION
// c[k] = sum_{i OR j = k, popcount(i)+popcount(j)=popcount(k)} a[i]*b[j]
//       = sum_{i AND j = 0, i OR j = k} a[i]*b[j]
// (Disjoint subset convolution)
// Uses rank (popcount) decomposition + OR convolution per rank.
// Time: O(N * n^2) where N = 2^n
// ================================================================
vector<ll> subset_sum_conv(vector<ll> a, vector<ll> b) {
    int N = a.size();
    int n = __lg(N); // N = 2^n

    // a_hat[k][mask] = a[mask] if popcount(mask) == k else 0
    vector<vector<ll>> fa(n + 1, vector<ll>(N, 0));
    vector<vector<ll>> fb(n + 1, vector<ll>(N, 0));
    for (int mask = 0; mask < N; mask++) {
        fa[__builtin_popcount(mask)][mask] = a[mask];
        fb[__builtin_popcount(mask)][mask] = b[mask];
    }

    // OR-transform each rank layer
    for (int k = 0; k <= n; k++) {
        zeta_or(fa[k]);
        zeta_or(fb[k]);
    }

    // Convolve over ranks (polynomial multiplication in rank)
    vector<vector<ll>> fc(n + 1, vector<ll>(N, 0));
    for (int k = 0; k <= n; k++)
        for (int j = 0; j <= k; j++)
            for (int mask = 0; mask < N; mask++)
                fc[k][mask] += fa[j][mask] * fb[k - j][mask];

    // Inverse OR-transform each rank layer
    for (int k = 0; k <= n; k++) mobius_or(fc[k]);

    // Extract result: c[mask] = fc[popcount(mask)][mask]
    vector<ll> c(N, 0);
    for (int mask = 0; mask < N; mask++)
        c[mask] = fc[__builtin_popcount(mask)][mask];

    return c;
}

// ================================================================
// SOS DP — Sum over Subsets, equivalent to AND zeta transform
// dp[mask] = sum_{sub SUBSET mask} a[sub]
// O(n * 2^n)
// ================================================================
vector<ll> sos_dp(vector<ll> a) {
    // Same as zeta_and
    zeta_and(a);
    return a;
}

// ================================================================
// XOR BASIS (linear basis over GF(2))
// Related to FWHT: finds a basis for the XOR span of a set of numbers.
// ================================================================
struct XorBasis {
    vector<ll> basis;

    void insert(ll x) {
        for (ll b : basis) x = min(x, x ^ b);
        if (x) basis.push_back(x);
    }

    bool contains(ll x) {
        for (ll b : basis) x = min(x, x ^ b);
        return x == 0;
    }

    ll max_xor() {
        ll res = 0;
        for (ll b : basis) res = max(res, res ^ b);
        return res;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // XOR convolution
    {
        vector<ll> a = {1, 2, 3, 4};   // a[0..3]
        vector<ll> b = {1, 1, 1, 1};
        auto c = conv_xor(a, b);
        printf("XOR convolution:\n");
        printf("  a = [1,2,3,4], b = [1,1,1,1]\n");
        printf("  c = [");
        for (int i = 0; i < (int)c.size(); i++)
            printf("%lld%s", c[i], i+1<(int)c.size()?",":"]");
        // c[k] = sum_{i^j=k} a[i]*b[j]
        // c[0] = a[0]*b[0]+a[1]*b[1]+a[2]*b[2]+a[3]*b[3] = 1+2+3+4=10
        printf("\n");
    }

    // AND convolution
    {
        vector<ll> a = {0, 1, 2, 3};
        vector<ll> b = {1, 1, 1, 1};
        auto c = conv_and(a, b);
        printf("\nAND convolution:\n");
        printf("  a=[0,1,2,3], b=[1,1,1,1]\n  c=[");
        for (int i = 0; i < (int)c.size(); i++)
            printf("%lld%s", c[i], i+1<(int)c.size()?",":"]");
        printf("\n");
    }

    // OR convolution
    {
        vector<ll> a = {3, 2, 1, 0};
        vector<ll> b = {1, 1, 1, 1};
        auto c = conv_or(a, b);
        printf("\nOR convolution:\n");
        printf("  a=[3,2,1,0], b=[1,1,1,1]\n  c=[");
        for (int i = 0; i < (int)c.size(); i++)
            printf("%lld%s", c[i], i+1<(int)c.size()?",":"]");
        printf("\n");
    }

    // Subset sum convolution
    {
        vector<ll> a = {0, 1, 2, 3};
        vector<ll> b = {1, 1, 1, 1};
        auto c = subset_sum_conv(a, b);
        printf("\nSubset sum convolution (disjoint):\n");
        printf("  a=[0,1,2,3], b=[1,1,1,1]\n  c=[");
        for (int i = 0; i < (int)c.size(); i++)
            printf("%lld%s", c[i], i+1<(int)c.size()?",":"]");
        printf("\n");
    }

    // Count pairs with XOR = target
    {
        printf("\nCount pairs with XOR=3 in {1,2,3,4,5}:\n");
        vector<ll> freq(8, 0);
        for (int x : {1,2,3,4,5}) freq[x]++;
        auto c = conv_xor(freq, freq);
        // c[3] = number of ordered pairs (i,j) with i^j=3
        // Subtract diagonal (i=j contributes i^i=0, so only c[0] affected)
        printf("  c[3] = %lld ordered pairs\n", c[3]); // each unordered pair counted twice

        // Note: c[0] includes pairs where i=j, so c[0] -= 5 for unordered same-element
    }

    // SOS DP
    {
        vector<ll> a = {1, 2, 3, 4};
        auto sos = sos_dp(a);
        printf("\nSOS DP (sum over subsets):\n");
        for (int mask = 0; mask < 4; mask++)
            printf("  sos[%d] = %lld  (subsets: ", mask, sos[mask]);
        printf(")\n");
        // sos[3] = a[0]+a[1]+a[2]+a[3] = 10 (all subsets of {0,1})
    }

    // XOR basis
    {
        XorBasis xb;
        for (int x : {1,2,3,4,5,6,7}) xb.insert(x);
        printf("\nXOR basis of {1..7}: size=%d, max_xor=%lld\n",
               (int)xb.basis.size(), xb.max_xor());
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public static class FWHT {

    // ================================================================
    // XOR CONVOLUTION — Walsh-Hadamard Transform
    // ================================================================
    public static void WhtXor(long[] a, bool invert) {
        int n = a.Length;
        for (int step = 1; step < n; step <<= 1) {
            for (int i = 0; i < n; i += step << 1) {
                for (int j = i; j < i + step; j++) {
                    long u = a[j], v = a[j + step];
                    a[j]        = u + v;
                    a[j + step] = u - v;
                }
            }
        }
        if (invert)
            for (int i = 0; i < n; i++) a[i] /= n;
    }

    public static long[] ConvXor(long[] a, long[] b) {
        a = (long[])a.Clone(); b = (long[])b.Clone();
        WhtXor(a, false); WhtXor(b, false);
        for (int i = 0; i < a.Length; i++) a[i] *= b[i];
        WhtXor(a, true);
        return a;
    }

    // Modular XOR convolution
    public static void WhtXorMod(long[] a, bool invert, long mod) {
        int n = a.Length;
        long inv2 = (mod + 1) / 2;
        for (int step = 1; step < n; step <<= 1) {
            for (int i = 0; i < n; i += step << 1) {
                for (int j = i; j < i + step; j++) {
                    long u = a[j], v = a[j + step];
                    a[j]        = (u + v) % mod;
                    a[j + step] = (u - v + mod) % mod;
                }
            }
        }
        if (invert) {
            long invN = 1;
            for (int i = 0; i < BitLength(n) - 1; i++)
                invN = invN * inv2 % mod;
            for (int i = 0; i < n; i++) a[i] = a[i] * invN % mod;
        }
    }

    private static int BitLength(int x) {
        int c = 0; while (x > 0) { c++; x >>= 1; } return c;
    }

    // ================================================================
    // AND CONVOLUTION — Zeta/Möbius over AND lattice
    // ================================================================
    public static void ZetaAnd(long[] a) {
        int n = BitLength(a.Length) - 1;
        for (int i = 0; i < n; i++)
            for (int mask = 0; mask < a.Length; mask++)
                if ((mask & (1 << i)) != 0)
                    a[mask] += a[mask ^ (1 << i)];
    }

    public static void MobiusAnd(long[] a) {
        int n = BitLength(a.Length) - 1;
        for (int i = 0; i < n; i++)
            for (int mask = 0; mask < a.Length; mask++)
                if ((mask & (1 << i)) != 0)
                    a[mask] -= a[mask ^ (1 << i)];
    }

    public static long[] ConvAnd(long[] a, long[] b) {
        a = (long[])a.Clone(); b = (long[])b.Clone();
        ZetaAnd(a); ZetaAnd(b);
        for (int i = 0; i < a.Length; i++) a[i] *= b[i];
        MobiusAnd(a);
        return a;
    }

    // ================================================================
    // OR CONVOLUTION — Zeta/Möbius over OR lattice
    // ================================================================
    public static void ZetaOr(long[] a) {
        int n = BitLength(a.Length) - 1;
        for (int i = 0; i < n; i++)
            for (int mask = 0; mask < a.Length; mask++)
                if ((mask & (1 << i)) == 0)
                    a[mask | (1 << i)] += a[mask];
    }

    public static void MobiusOr(long[] a) {
        int n = BitLength(a.Length) - 1;
        for (int i = 0; i < n; i++)
            for (int mask = 0; mask < a.Length; mask++)
                if ((mask & (1 << i)) == 0)
                    a[mask | (1 << i)] -= a[mask];
    }

    public static long[] ConvOr(long[] a, long[] b) {
        a = (long[])a.Clone(); b = (long[])b.Clone();
        ZetaOr(a); ZetaOr(b);
        for (int i = 0; i < a.Length; i++) a[i] *= b[i];
        MobiusOr(a);
        return a;
    }

    // ================================================================
    // SUBSET SUM CONVOLUTION — disjoint union
    // ================================================================
    public static long[] SubsetSumConv(long[] a, long[] b) {
        int N = a.Length;
        int n = BitLength(N) - 1;

        var fa = new long[n + 1][];
        var fb = new long[n + 1][];
        for (int k = 0; k <= n; k++) {
            fa[k] = new long[N];
            fb[k] = new long[N];
        }
        for (int mask = 0; mask < N; mask++) {
            int pc = PopCount(mask);
            fa[pc][mask] = a[mask];
            fb[pc][mask] = b[mask];
        }

        for (int k = 0; k <= n; k++) {
            ZetaOr(fa[k]);
            ZetaOr(fb[k]);
        }

        var fc = new long[n + 1][];
        for (int k = 0; k <= n; k++) fc[k] = new long[N];
        for (int k = 0; k <= n; k++)
            for (int j = 0; j <= k; j++)
                for (int mask = 0; mask < N; mask++)
                    fc[k][mask] += fa[j][mask] * fb[k - j][mask];

        for (int k = 0; k <= n; k++) MobiusOr(fc[k]);

        var c = new long[N];
        for (int mask = 0; mask < N; mask++)
            c[mask] = fc[PopCount(mask)][mask];
        return c;
    }

    private static int PopCount(int x) {
        int c = 0; while (x > 0) { c += x & 1; x >>= 1; } return c;
    }

    // ================================================================
    // SOS DP — Sum over subsets (same as ZetaAnd)
    // ================================================================
    public static long[] SosDp(long[] a) {
        a = (long[])a.Clone();
        ZetaAnd(a);
        return a;
    }

    // ================================================================
    // Usage
    // ================================================================
    public static void Main() {
        // XOR convolution
        long[] a = {1,2,3,4}, b = {1,1,1,1};
        var cXor = ConvXor(a, b);
        Console.WriteLine($"XOR conv: [{string.Join(",",cXor)}]");

        // AND convolution
        long[] a2 = {0,1,2,3};
        var cAnd = ConvAnd(a2, b);
        Console.WriteLine($"AND conv: [{string.Join(",",cAnd)}]");

        // OR convolution
        long[] a3 = {3,2,1,0};
        var cOr = ConvOr(a3, b);
        Console.WriteLine($"OR conv:  [{string.Join(",",cOr)}]");

        // Subset sum convolution
        var cSub = SubsetSumConv(a2, b);
        Console.WriteLine($"Subset:   [{string.Join(",",cSub)}]");

        // SOS DP
        long[] f = {1,2,3,4};
        var sos = SosDp(f);
        Console.WriteLine($"SOS DP:   [{string.Join(",",sos)}]");
        // sos[3] = f[0]+f[1]+f[2]+f[3] = 10 (mask 11 = subsets 00,01,10,11)
    }
}
```

---

## Transform Comparison Table

| Transform | Operation | Butterfly | Inverse |
|---|---|---|---|
| FFT | Polynomial (add index) | `(u+v, u+wv)` | Conjugate + /N |
| XOR (WHT) | XOR index | `(u+v, u-v)` | Same + /N |
| AND (Zeta) | AND index | `a[m] += a[m^bit]` | `a[m] -= a[m^bit]` |
| OR (Zeta) | OR index | `a[m\|bit] += a[m]` | `a[m\|bit] -= a[m]` |

---

## Relationship Between Transforms

```
AND ↔ OR: complementing indices maps one to the other
          a[mask] → a[complement(mask)] converts AND conv to OR conv

XOR ↔ AND/OR: via subset sum convolution (XOR with popcount constraint)

SOS DP = AND zeta transform = "prefix sum over subset lattice"
```

---

## Pitfalls

- **Array size must be a power of 2** — all FWHT variants require N = 2^n. If the input arrays have non-power-of-2 size, pad with zeros to the next power of 2. Using a non-power-of-2 size causes the butterfly to step out of bounds or produce wrong results at boundary indices.
- **XOR inverse divides by N, not by 2** — the WHT is self-inverse up to a factor of N (the array size). Always divide by N (or equivalently, by 2 for each of the log₂N levels). Dividing by 2 at each level avoids overflow in the intermediate computation but requires the forward pass to not already divide. Keep the convention consistent.
- **AND vs OR direction** — the AND zeta transform accumulates `a[mask] += a[mask ^ bit]` (add from submask to mask = sum over subsets), while OR accumulates `a[mask | bit] += a[mask]` (add from mask to supermask = propagate upward). Confusing the two produces the wrong convolution type silently.
- **Subset sum convolution requires popcount layering** — regular OR convolution computes `c[k] = Σ_{i OR j = k} a[i]*b[j]`, which includes pairs where i AND j ≠ 0. The subset sum (disjoint union) convolution requires i AND j = 0. Forgetting the rank decomposition and using plain OR convolution gives wrong answers for problems requiring disjoint subsets.
- **Overflow in XOR WHT** — the WHT uses additions and subtractions. For input values up to V and array size N = 2^n, intermediate values can reach V * N. For V = 10^9 and N = 2^20, intermediates reach 10^{15} — fits in `int64`. For larger values, use modular WHT throughout.
- **Modular inverse of N requires N to be coprime to mod** — the inverse WHT divides by N. For mod = 2^32, N = 2^k, and N has no inverse mod 2^32. Use a prime modulus (e.g., 998244353) for modular XOR convolution. AND/OR convolutions don't need division so they work with any modulus.
- **Subset sum convolution time is O(N n²)** — for N = 2^20 and n = 20, this is 4 × 10^8 operations — borderline. For competitive programming with N ≤ 2^20, it may TLE. Optimize by combining the rank-polynomial multiplication with the zeta transform in a single pass, or reduce to N ≤ 2^18.

---

## Complexity Summary

| Operation | N = 2^n | Time | Notes |
|---|---|---|---|
| XOR/AND/OR transform | 2^20 | O(20 * 2^20) ≈ 2×10^7 | Very fast |
| XOR convolution | 2^20 | O(20 * 2^20) | Two transforms + pointwise |
| AND/OR convolution | 2^20 | O(20 * 2^20) | Same |
| Subset sum convolution | 2^20 | O(20² * 2^20) ≈ 4×10^8 | Slower |
| Naive XOR convolution | 2^20 | O(4^20) ≈ 10^{12} | Infeasible |

---

## Conclusion

FWHT is the **Fast Fourier Transform for boolean/bitwise convolutions**:

- XOR convolution uses the Walsh-Hadamard matrix and runs in O(N log N) with simple +/- butterflies.
- AND/OR convolutions use the Möbius/zeta transforms over the subset lattice — equivalent to SOS DP.
- Subset sum convolution handles the stricter disjoint-union condition via popcount layering at O(N log² N).
- All three are standard competitive programming primitives for problems involving XOR sums, subset sums, or counting pairs by bitwise operation.

**Key takeaway:**  
The butterfly structure is identical for all three transforms — only the element-wise operation changes: `(u+v, u-v)` for XOR, `(a+b, b)` for AND, and `(a, a+b)` for OR (in one direction). The inverse replaces addition with subtraction and adds a division by N for XOR. For AND and OR, the same butterfly applied again undoes the transform — making them their own inverses up to sign.
