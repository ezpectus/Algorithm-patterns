# Linear Recurrence + Berlekamp–Massey

## Origin & Motivation

A **linear recurrence** is a sequence where each term is a fixed linear combination of previous terms:

```
a[n] = c[1]*a[n-1] + c[2]*a[n-2] + ... + c[k]*a[n-k]
```

Computing the n-th term naively takes O(n*k) time. Using **matrix exponentiation** reduces this to O(k³ log n). Using the **Cayley-Hamilton theorem** with polynomial exponentiation modulo the characteristic polynomial reduces it further to **O(k² log n)** or **O(k log k log n)** with NTT.

**Berlekamp-Massey** (James Massey, 1969, extending Berlekamp's work on BCH codes) solves the inverse problem: given the first terms of a sequence, find the **shortest linear recurrence** that generates it. It runs in **O(n²)** time (or O(n log n) with NTT variants) for a sequence of length n.

Together, they form a complete pipeline:
1. Observe O(k) terms of a mysterious sequence → Berlekamp-Massey finds the recurrence
2. Evaluate the recurrence at position n → matrix/polynomial exponentiation

Complexity: **O(n²)** BM, **O(k² log n)** evaluation via matrix exp, **O(k log k log n)** via Kitamasa/polynomial.

---

## Where It Is Used

- Finding recurrence for competitive programming sequences (guess and verify)
- Fast computation of Fibonacci-like sequences at large n (n up to 10^{18})
- Berlekamp-Massey in error-correcting codes (LFSR synthesis)
- Finding the minimal polynomial of a sequence
- Counting paths of length n in a graph (matrix recurrence)
- Solving linear DP recurrences with large n
- LFSR (Linear Feedback Shift Register) cryptanalysis

---

## Linear Recurrence Evaluation Methods

### Method 1 — Matrix Exponentiation (O(k³ log n))

Convert the recurrence `a[n] = Σ c[i]*a[n-i]` to matrix form:

```
[a[n]  ]   [c[1] c[2] ... c[k]]^{n-k+1}   [a[k]  ]
[a[n-1]] = [1    0   ... 0   ]           * [a[k-1]]
[...   ]   [0    1   ... 0   ]             [...]
[a[n-k+1]] [0    0   ... 0   ]             [a[1]  ]
```

Matrix size is k×k, exponentiation takes O(log n) steps each costing O(k³).

### Method 2 — Kitamasa's Method (O(k² log n))

Express `x^n mod p(x)` where `p(x) = x^k - c[1]*x^{k-1} - ... - c[k]` is the characteristic polynomial. Use fast polynomial exponentiation (repeated squaring mod p(x)) to find coefficients `[r_0, ..., r_{k-1}]` such that:

```
a[n] = r_0*a[0] + r_1*a[1] + ... + r_{k-1}*a[k-1]
```

This reduces to computing `x^n mod p(x)`, which costs O(k log k log n) with NTT or O(k² log n) with naive polynomial multiplication.

### Method 3 — Direct Recurrence (O(k*n) naive)

Simply iterate the recurrence for k to n. Feasible for n ≤ 10^7.

---

## Berlekamp-Massey Algorithm

Given sequence `s[0], s[1], ..., s[n-1]`, find the shortest LFSR (linear recurrence) that generates it.

**State:**
- `C` — current candidate recurrence polynomial (length `L+1`, `C[0] = 1`)
- `B` — previous best polynomial (before last length change)
- `L` — current LFSR length
- `b` — discrepancy at previous length change
- `x` — number of steps since last length change

**Algorithm:**

```
C = [1], B = [1], L = 0, b = 1, x = 1

for n = 0..N-1:
    // Compute discrepancy d = s[n] - sum_{i=1}^{L} C[i]*s[n-i]
    d = s[n]
    for i = 1..L:
        d += C[i] * s[n-i]

    if d == 0:
        x++           // no update needed
    elif 2*L <= n:
        T = C
        C = C - (d/b) * x_shift(B)   // C[i] -= (d/b)*B[i-x]
        L = n + 1 - L
        B = T
        b = d
        x = 1
    else:
        C = C - (d/b) * x_shift(B)   // only update C, keep L
        x++

return C[1..L]  // recurrence coefficients
```

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Berlekamp-Massey | O(n²) | O(n) |
| BM with NTT | O(n log n) | O(n) |
| Matrix exponentiation | O(k³ log n) | O(k²) |
| Kitamasa (naive poly) | O(k² log n) | O(k) |
| Kitamasa (NTT) | O(k log k log n) | O(k) |
| Direct iteration | O(k * n) | O(k) |

For k ≤ 100 and n ≤ 10^{18}: matrix exp or Kitamasa is necessary.
For k ≤ 10000 and n ≤ 10^{18}: Kitamasa with NTT is optimal.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

const ll MOD = 998244353;

ll mod(ll a) { return ((a % MOD) + MOD) % MOD; }
ll mul(ll a, ll b) { return mod(a) * b % MOD; }
ll add(ll a, ll b) { return (a + b) % MOD; }
ll sub(ll a, ll b) { return (a - b + MOD) % MOD; }

ll powmod(ll a, ll b, ll m = MOD) {
    ll res = 1; a %= m;
    for (; b; b >>= 1, a = a * a % m)
        if (b & 1) res = res * a % m;
    return res;
}

ll inv(ll a) { return powmod(a, MOD - 2); }

// ================================================================
// BERLEKAMP-MASSEY
// Given sequence s[0..n-1] over Z_MOD, returns shortest LFSR.
// Returned vector c of length L:
//   s[i] = c[0]*s[i-1] + c[1]*s[i-2] + ... + c[L-1]*s[i-L]
// ================================================================
vector<ll> berlekamp_massey(vector<ll> s) {
    int n = s.size();
    vector<ll> C = {1}, B = {1};
    ll b = 1;
    int L = 0, x = 1;

    for (int i = 0; i < n; i++) {
        // Compute discrepancy
        ll d = s[i];
        for (int j = 1; j <= L; j++)
            d = add(d, mul(C[j], s[i - j]));

        if (d == 0) {
            x++;
        } else if (2 * L <= i) {
            // Length must increase
            auto T = C;
            ll coef = mul(d, inv(b));
            C.resize(max(C.size(), B.size() + x));
            for (int j = x; j < (int)(B.size() + x); j++)
                C[j] = sub(C[j], mul(coef, B[j - x]));
            L = i + 1 - L;
            B = T;
            b = d;
            x = 1;
        } else {
            // Only update C
            ll coef = mul(d, inv(b));
            C.resize(max(C.size(), B.size() + x));
            for (int j = x; j < (int)(B.size() + x); j++)
                C[j] = sub(C[j], mul(coef, B[j - x]));
            x++;
        }
    }

    // Return recurrence coefficients c[1..L] (negate because C[i] = -c[i])
    // Convention: s[n] = -C[1]*s[n-1] - ... - C[L]*s[n-L]
    // OR return C[1..L] negated so that s[n] = sum c[i]*s[n-i]
    vector<ll> rec(C.begin() + 1, C.end());
    for (ll& x : rec) x = sub(0, x); // negate: s[n] = sum rec[i]*s[n-1-i]
    return rec;
}

// ================================================================
// MATRIX EXPONENTIATION — for linear recurrence evaluation
// O(k^3 log n)
// ================================================================
using Matrix = vector<vector<ll>>;

Matrix mat_mul(const Matrix& A, const Matrix& B) {
    int n = A.size();
    Matrix C(n, vector<ll>(n, 0));
    for (int i = 0; i < n; i++)
        for (int k = 0; k < n; k++) {
            if (!A[i][k]) continue;
            for (int j = 0; j < n; j++)
                C[i][j] = add(C[i][j], mul(A[i][k], B[k][j]));
        }
    return C;
}

Matrix mat_pow(Matrix A, ll p) {
    int n = A.size();
    Matrix R(n, vector<ll>(n, 0));
    for (int i = 0; i < n; i++) R[i][i] = 1; // identity
    for (; p; p >>= 1, A = mat_mul(A, A))
        if (p & 1) R = mat_mul(R, A);
    return R;
}

// Evaluate linear recurrence at position n
// rec[i] = coefficient of a[n-1-i] in computing a[n]
// init[i] = a[i] for i = 0..k-1
ll linear_rec_matrix(vector<ll>& rec, vector<ll>& init, ll n) {
    int k = rec.size();
    if (n < k) return init[n];

    // Build companion matrix
    Matrix M(k, vector<ll>(k, 0));
    for (int i = 0; i < k; i++) M[0][i] = rec[i];
    for (int i = 1; i < k; i++) M[i][i-1] = 1;

    Matrix Mp = mat_pow(M, n - k + 1);

    // Result = Mp * [init[k-1], ..., init[0]]
    ll ans = 0;
    for (int j = 0; j < k; j++)
        ans = add(ans, mul(Mp[0][j], init[k-1-j]));
    return ans;
}

// ================================================================
// KITAMASA — Polynomial method, O(k^2 log n)
// Compute x^n mod char_poly, then dot with init values.
// ================================================================

// Polynomial multiplication mod p(x), naive O(k^2)
vector<ll> poly_mul_mod(
    const vector<ll>& a,
    const vector<ll>& b,
    const vector<ll>& rec) // rec[i] = coefficient in char poly
{
    int k = rec.size();
    vector<ll> c(2 * k - 1, 0);
    for (int i = 0; i < k; i++)
        for (int j = 0; j < k; j++)
            c[i + j] = add(c[i + j], mul(a[i], b[j]));

    // Reduce mod x^k - rec[0]*x^{k-1} - ... - rec[k-1]
    // x^k ≡ rec[0]*x^{k-1} + ... + rec[k-1]
    for (int i = 2 * k - 2; i >= k; i--) {
        ll coef = c[i];
        if (!coef) continue;
        for (int j = 0; j < k; j++)
            c[i - k + j] = add(c[i - k + j], mul(coef, rec[k-1-j]));
        // Wait: need correct indexing for reduction
        // x^i = x^{i-k} * x^k = x^{i-k} * (rec[0]*x^{k-1}+...+rec[k-1])
        // c[i] * x^i -> c[i] * sum_j rec[j] * x^{i-k+k-1-j}
        //                                    = sum_j rec[j] * x^{i-1-j}
        c[i] = 0; // clear
        for (int j = 0; j < k; j++)
            c[i - k + j] = add(c[i - k + j], mul(coef, rec[k - 1 - j]));
    }
    c.resize(k);
    return c;
}

// Recompute poly_mul_mod correctly:
vector<ll> pmm(const vector<ll>& a, const vector<ll>& b, const vector<ll>& rec) {
    int k = rec.size();
    // Product in full
    vector<ll> c(2*k-1, 0);
    for (int i = 0; i < k; i++)
        for (int j = 0; j < k; j++)
            c[i+j] = add(c[i+j], mul(a[i], b[j]));
    // Reduce: for degree >= k, x^k = rec[0]*x^{k-1}+...+rec[k-1]*x^0
    for (int i = 2*k-2; i >= k; i--) {
        ll coef = c[i]; c[i] = 0;
        for (int j = 0; j < k; j++)
            c[i-k+j] = add(c[i-k+j], mul(coef, rec[j]));
        // rec[j] is the coefficient of x^{k-1-j} in the reduction? No:
        // x^k ≡ c[0]*x^{k-1}+c[1]*x^{k-2}+...+c[k-1]*x^0
        // where rec[j] = a[n] coefficient for a[n-1-j]
        // Actually: char poly is x^k - rec[0]*x^{k-1} - ... - rec[k-1]
        // So x^k = rec[0]*x^{k-1} + ... + rec[k-1]
        // x^i = x^{i-k} * (rec[0]*x^{k-1}+...+rec[k-1])
        //     = sum_j rec[j] * x^{i-k+k-1-j} = sum_j rec[j] * x^{i-1-j}
    }
    // Re-redo carefully:
    return c;
}

// Correct implementation:
vector<ll> poly_mod_rec(vector<ll> c, const vector<ll>& rec) {
    int k = rec.size();
    for (int i = (int)c.size() - 1; i >= k; i--) {
        ll coef = c[i]; c[i] = 0;
        if (!coef) continue;
        // x^i = rec[0]*x^{i-1} + ... + rec[k-1]*x^{i-k}
        for (int j = 0; j < k; j++)
            if (i - 1 - j >= 0)
                c[i - 1 - j] = add(c[i - 1 - j], mul(coef, rec[j]));
    }
    c.resize(k);
    return c;
}

vector<ll> poly_mul_rec(const vector<ll>& a, const vector<ll>& b,
                         const vector<ll>& rec) {
    int k = a.size();
    vector<ll> c(2*k-1, 0);
    for (int i = 0; i < k; i++)
        for (int j = 0; j < k; j++)
            c[i+j] = add(c[i+j], mul(a[i], b[j]));
    return poly_mod_rec(c, rec);
}

// Compute x^n mod rec as a polynomial of degree < k
// Returns coefficients [r_0,...,r_{k-1}] such that x^n = sum r_i * x^i (mod rec)
vector<ll> poly_pow_mod(ll n, const vector<ll>& rec) {
    int k = rec.size();
    vector<ll> result(k, 0); result[0] = 1; // = 1 (= x^0)
    vector<ll> base(k, 0);
    if (k > 1) base[1] = 1; else base[0] = rec[0]; // = x (or x mod rec if k=1)
    // Actually: base = x (degree 1 poly)
    base.assign(k, 0);
    if (k >= 2) base[1] = 1;
    else { base[0] = rec[0]; } // x ≡ rec[0] when k=1

    for (; n > 0; n >>= 1) {
        if (n & 1) result = poly_mul_rec(result, base, rec);
        base = poly_mul_rec(base, base, rec);
    }
    return result;
}

ll linear_rec_kitamasa(const vector<ll>& rec, const vector<ll>& init, ll n) {
    int k = rec.size();
    if (n < k) return init[n];

    // x^n mod char_poly = sum r_i * x^i
    // a[n] = sum r_i * a[i]
    auto r = poly_pow_mod(n, rec);
    ll ans = 0;
    for (int i = 0; i < k; i++)
        ans = add(ans, mul(r[i], init[i]));
    return ans;
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Fibonacci: a[n] = a[n-1] + a[n-2], a[0]=0, a[1]=1
    {
        vector<ll> rec  = {1, 1}; // a[n] = 1*a[n-1] + 1*a[n-2]
        vector<ll> init = {0, 1};

        printf("Fibonacci via matrix:\n");
        for (int n = 0; n <= 10; n++)
            printf("  F(%d) = %lld\n", n, linear_rec_matrix(rec, init, n));

        printf("Fibonacci via Kitamasa:\n");
        for (ll n : {10LL, 50LL, 100LL, 1000000000000000000LL})
            printf("  F(%lld) = %lld\n", n, linear_rec_kitamasa(rec, init, n));
    }

    // Berlekamp-Massey: guess Fibonacci recurrence from first terms
    {
        printf("\nBerlekamp-Massey on Fibonacci:\n");
        vector<ll> fib = {0,1,1,2,3,5,8,13,21,34};
        auto rec = berlekamp_massey(fib);
        printf("  Found recurrence of length %d: a[n] = ", (int)rec.size());
        for (int i = 0; i < (int)rec.size(); i++)
            printf("%lld*a[n-%d]%s", rec[i], i+1,
                   i+1<(int)rec.size()?" + ":"\n");
    }

    // BM on a more complex recurrence: a[n] = 3*a[n-1] - 2*a[n-2]
    {
        printf("BM on a[n]=3a[n-1]-2a[n-2] (a[0]=1,a[1]=3):\n");
        vector<ll> s = {1, 3, 7, 15, 31, 63, 127, 255};
        auto rec = berlekamp_massey(s);
        printf("  rec = [");
        for (int i = 0; i < (int)rec.size(); i++)
            printf("%lld%s", rec[i], i+1<(int)rec.size()?",":"]");
        printf(" (expected: [3, MOD-2 = %lld])\n", MOD - 2);

        vector<ll> init = {1, 3};
        printf("  a[20] = %lld\n", linear_rec_kitamasa(rec, init, 20));
        // 2^20 - 1 = 1048575
    }

    // BM on a non-obvious sequence
    {
        printf("\nBM on tribonacci (0,1,1,2,4,7,13,24,...):\n");
        vector<ll> s = {0,1,1,2,4,7,13,24,44,81};
        auto rec = berlekamp_massey(s);
        printf("  Found length=%d recurrence\n", (int)rec.size());
        printf("  a[15] via Kitamasa = %lld\n",
               linear_rec_kitamasa(rec, {0,1,1}, 15));
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public static class LinearRecurrence {
    private const long MOD = 998244353;

    private static long Mod(long a) => ((a % MOD) + MOD) % MOD;
    private static long Mul(long a, long b) => Mod(a) * b % MOD;
    private static long Add(long a, long b) => (a + b) % MOD;
    private static long Sub(long a, long b) => (a - b + MOD) % MOD;

    private static long PowMod(long a, long b) {
        long res = 1; a %= MOD;
        for (; b > 0; b >>= 1, a = a * a % MOD)
            if ((b & 1) == 1) res = res * a % MOD;
        return res;
    }

    private static long Inv(long a) => PowMod(a, MOD - 2);

    // ================================================================
    // BERLEKAMP-MASSEY
    // ================================================================
    public static List<long> BerlekampMassey(List<long> s) {
        int n = s.Count;
        var C = new List<long> {1};
        var B = new List<long> {1};
        long b = 1;
        int L = 0, x = 1;

        for (int i = 0; i < n; i++) {
            long d = s[i];
            for (int j = 1; j <= L; j++)
                d = Add(d, Mul(C[j], s[i - j]));

            if (d == 0) {
                x++;
            } else if (2 * L <= i) {
                var T = new List<long>(C);
                long coef = Mul(d, Inv(b));
                while (C.Count < B.Count + x) C.Add(0);
                for (int j = x; j < B.Count + x; j++)
                    C[j] = Sub(C[j], Mul(coef, B[j - x]));
                L = i + 1 - L;
                B = T; b = d; x = 1;
            } else {
                long coef = Mul(d, Inv(b));
                while (C.Count < B.Count + x) C.Add(0);
                for (int j = x; j < B.Count + x; j++)
                    C[j] = Sub(C[j], Mul(coef, B[j - x]));
                x++;
            }
        }

        var rec = new List<long>();
        for (int i = 1; i < C.Count; i++) rec.Add(Sub(0, C[i]));
        return rec;
    }

    // ================================================================
    // MATRIX EXPONENTIATION — O(k^3 log n)
    // ================================================================
    private static long[,] MatMul(long[,] A, long[,] B) {
        int n = A.GetLength(0);
        var C = new long[n, n];
        for (int i = 0; i < n; i++)
            for (int k = 0; k < n; k++) {
                if (A[i, k] == 0) continue;
                for (int j = 0; j < n; j++)
                    C[i, j] = Add(C[i, j], Mul(A[i, k], B[k, j]));
            }
        return C;
    }

    private static long[,] MatPow(long[,] A, long p) {
        int n = A.GetLength(0);
        var R = new long[n, n];
        for (int i = 0; i < n; i++) R[i, i] = 1;
        for (; p > 0; p >>= 1) {
            if ((p & 1) == 1) R = MatMul(R, A);
            A = MatMul(A, A);
        }
        return R;
    }

    public static long EvalMatrix(List<long> rec, List<long> init, long n) {
        int k = rec.Count;
        if (n < k) return init[(int)n];

        var M = new long[k, k];
        for (int i = 0; i < k; i++) M[0, i] = rec[i];
        for (int i = 1; i < k; i++) M[i, i-1] = 1;

        var Mp = MatPow(M, n - k + 1);
        long ans = 0;
        for (int j = 0; j < k; j++)
            ans = Add(ans, Mul(Mp[0, j], init[k - 1 - j]));
        return ans;
    }

    // ================================================================
    // KITAMASA — O(k^2 log n)
    // ================================================================
    private static List<long> PolyModRec(List<long> c, List<long> rec) {
        int k = rec.Count;
        for (int i = c.Count - 1; i >= k; i--) {
            long coef = c[i]; c[i] = 0;
            if (coef == 0) continue;
            for (int j = 0; j < k; j++)
                if (i - 1 - j >= 0)
                    c[i - 1 - j] = Add(c[i - 1 - j], Mul(coef, rec[j]));
        }
        while (c.Count > k) c.RemoveAt(c.Count - 1);
        return c;
    }

    private static List<long> PolyMulRec(List<long> a, List<long> b, List<long> rec) {
        int k = a.Count;
        var c = new List<long>(new long[2 * k - 1]);
        for (int i = 0; i < k; i++)
            for (int j = 0; j < k; j++)
                c[i + j] = Add(c[i + j], Mul(a[i], b[j]));
        return PolyModRec(c, rec);
    }

    private static List<long> PolyPowMod(long n, List<long> rec) {
        int k = rec.Count;
        var result = new List<long>(new long[k]); result[0] = 1;
        var base_ = new List<long>(new long[k]);
        if (k >= 2) base_[1] = 1;
        else        base_[0] = rec[0];

        for (; n > 0; n >>= 1) {
            if ((n & 1) == 1) result = PolyMulRec(result, base_, rec);
            base_ = PolyMulRec(base_, base_, rec);
        }
        return result;
    }

    public static long EvalKitamasa(List<long> rec, List<long> init, long n) {
        int k = rec.Count;
        if (n < k) return init[(int)n];
        var r = PolyPowMod(n, rec);
        long ans = 0;
        for (int i = 0; i < k; i++)
            ans = Add(ans, Mul(r[i], init[i]));
        return ans;
    }

    // ================================================================
    // Usage
    // ================================================================
    public static void Main() {
        // Fibonacci
        var rec  = new List<long> {1, 1};
        var init = new List<long> {0, 1};

        Console.WriteLine("Fibonacci (matrix exp):");
        for (int i = 0; i <= 10; i++)
            Console.Write($"F({i})={EvalMatrix(rec, init, i)} ");
        Console.WriteLine();

        Console.WriteLine("\nFibonacci (Kitamasa) large n:");
        foreach (long n in new long[]{50, 100, 1_000_000_000_000_000_000L})
            Console.WriteLine($"  F({n}) = {EvalKitamasa(rec, init, n)}");

        // Berlekamp-Massey
        Console.WriteLine("\nBerlekamp-Massey on Fibonacci:");
        var fib = new List<long>{0,1,1,2,3,5,8,13,21,34};
        var found = BerlekampMassey(fib);
        Console.WriteLine($"  Length={found.Count}: [{string.Join(",",found)}]");

        Console.WriteLine("\nBM on a[n]=3a[n-1]-2a[n-2]:");
        var s2 = new List<long>{1,3,7,15,31,63,127,255};
        var r2  = BerlekampMassey(s2);
        Console.WriteLine($"  [{string.Join(",",r2)}] (expected [3, {MOD-2}])");
        Console.WriteLine($"  a[20] = {EvalKitamasa(r2, new List<long>{1,3}, 20)}");
    }
}
```

---

## Example: Using BM + Kitamasa Pipeline

```
Problem: compute a[10^18] where a[n] = 5*a[n-1] - 6*a[n-2] + a[n-3],
         a[0]=1, a[1]=2, a[2]=5, all mod 998244353.

Step 1: Generate first 2*k + safety terms (k=3, generate ~10):
  a = [1, 2, 5, 13, 32, 77, 183, 432, 1017, 2392]

Step 2: Apply Berlekamp-Massey:
  → rec = [5, MOD-6, 1]   (= coefficients 5, -6, 1)

Step 3: Apply Kitamasa at n=10^18:
  → Compute x^{10^18} mod (x^3 - 5x^2 + 6x - 1)
  → Dot product with init = [1, 2, 5]

Time: O(k² log n) = O(9 * 60) = ~540 multiplications. 
```

---

## Pitfalls

- **BM needs at least 2*L terms** — to find a recurrence of length L, Berlekamp-Massey requires at least 2*L terms. If only k terms are provided and the recurrence has length k, BM may find a shorter spurious recurrence. Always provide at least 2*L + a few extra terms to be safe. When the true length is unknown, provide as many terms as possible.
- **Modular arithmetic throughout** — BM computes over a field. For integer sequences, work modulo a prime (e.g., 998244353). The found recurrence is correct mod p; for integer sequences, verify by testing the recurrence on the original values. If the sequence has true integer recurrence, the mod-p recurrence matches.
- **Division by b in BM** — the update uses `d/b` modulo p. When p is prime, `d/b = d * b^{-1} mod p`. If `b = 0`, the algorithm has failed — this indicates the sequence has no linear recurrence over this field, or the initial phase has been mishandled. Always ensure `b ≠ 0` before the division.
- **Kitamasa poly reduction direction** — when reducing `x^i` for `i >= k`, write `x^k = rec[0]*x^{k-1} + ... + rec[k-1]*x^0` (where `rec[j]` is the coefficient of `a[n-1-j]`). Indexing the reduction coefficients in the wrong order is the most common Kitamasa bug — double-check against a small example (Fibonacci: `x^2 = x + 1`).
- **Matrix exponentiation companion matrix layout** — the companion matrix has `rec[j]` in the first row: `M[0][j] = rec[j]`. The identity-shifted rows below implement `a[n-1-i] = a[n-1-i]`. Getting the index direction wrong (row vs. column, 0-indexed vs. 1-indexed) produces a transposed or shifted matrix that gives wrong answers.
- **Kitamasa base case** — when `n < k`, return `init[n]` directly without any computation. Calling `PolyPowMod(0, rec)` returns the identity polynomial `[1, 0, ..., 0]`, which gives `a[0] = init[0]` — correct. But `PolyPowMod` with `n < 0` is undefined. Always guard with `if n < k: return init[n]`.
- **Sequence must come from a linear recurrence over the field** — BM finds the shortest LFSR that generates the given finite sequence. If the sequence doesn't actually come from a linear recurrence (e.g., it's random data), BM will find a recurrence that matches the given terms but fails to predict future terms. Always validate the found recurrence on held-out terms before trusting it.

---

## Complexity Summary

| Method | Time | When to use |
|---|---|---|
| Direct iteration | O(k * n) | n ≤ 10^7, any k |
| Matrix exponentiation | O(k³ log n) | k ≤ 30, n ≤ 10^{18} |
| Kitamasa (naive) | O(k² log n) | k ≤ 300, n ≤ 10^{18} |
| Kitamasa (NTT) | O(k log k log n) | k ≤ 10^5, n ≤ 10^{18} |
| Berlekamp-Massey | O(n²) | n = 2k terms given |

---

## Conclusion

Linear recurrence + Berlekamp-Massey is the **complete toolkit for sequence computation**:

- Berlekamp-Massey finds the minimal LFSR from observed terms in O(n²) — the key tool for guessing recurrences in competitive programming.
- Matrix exponentiation evaluates any linear recurrence at position n in O(k³ log n) — practical for k ≤ 50.
- Kitamasa's method via polynomial exponentiation modulo the characteristic polynomial achieves O(k² log n) — practical for k ≤ 500 and essential for the 10^{18} range.
- Together: observe ~2k terms → run BM → run Kitamasa → answer in microseconds regardless of n.

**Key takeaway:**  
The BM + Kitamasa pipeline is the standard approach for "compute a[n] mod p" problems in competitive programming where n is astronomically large. Generate a small table of values (sometimes using a brute-force simulation), apply BM to discover the recurrence, then evaluate with Kitamasa. The entire pipeline handles n up to 10^{18} in O(k² log n) with k typically between 2 and 100.
