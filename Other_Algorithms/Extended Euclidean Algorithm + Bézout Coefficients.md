# Extended Euclidean Algorithm + Bézout Coefficients

## Origin & Motivation

The Euclidean algorithm for computing GCD was known to Euclid around 300 BCE. The **Extended Euclidean Algorithm** computes not only `gcd(a, b)` but also integers `x, y` such that:

```
a * x + b * y = gcd(a, b)
```

These are the **Bézout coefficients**, guaranteed to exist by Bézout's identity (a consequence of the fact that ℤ is a principal ideal domain). The algorithm runs in **O(log min(a, b))** time — the same as the basic Euclidean algorithm — by backtracking through the recursive calls to recover `x` and `y`.

The extended algorithm is the computational backbone of modular inverse, Chinese Remainder Theorem, solving linear Diophantine equations, and lattice basis reduction. It is one of the most frequently used primitives in number theory algorithms.

Complexity: **O(log min(a, b))** time, **O(log min(a, b))** space (recursive) or **O(1)** space (iterative).

---

## Where It Is Used

- Modular multiplicative inverse: `a^{-1} mod m` when `gcd(a, m) = 1`
- Solving linear Diophantine equations: `ax + by = c`
- Chinese Remainder Theorem reconstruction
- RSA key generation (modular inverse of e mod φ(n))
- Fraction reduction and rational arithmetic
- Lattice algorithms (LLL basis reduction)
- Polynomial GCD over fields

---

## Bézout's Identity

For any integers `a, b` (not both zero):

```
∃ x, y ∈ ℤ : a*x + b*y = gcd(a, b)
```

**Corollary:** `ax + by = c` has integer solutions iff `gcd(a, b) | c`.

The Bézout coefficients `x, y` are not unique — the general solution is:

```
x = x_0 + (b/g) * t
y = y_0 - (a/g) * t
for any t ∈ ℤ, where g = gcd(a, b)
```

The algorithm finds the particular solution `(x_0, y_0)` with `|x_0| ≤ b/(2g)` and `|y_0| ≤ a/(2g)`.

---

## Algorithm

### Recursive Formulation

```
extgcd(a, b):
    if b == 0:
        return (a, 1, 0)   // gcd=a, x=1, y=0: a*1 + 0*0 = a
    (g, x1, y1) = extgcd(b, a % b)
    x = y1
    y = x1 - (a / b) * y1
    return (g, x, y)

// Invariant: at each step, g = a*x + b*y
// Base: g = a = a*1 + 0*0
// Step: g = b*x1 + (a%b)*y1
//          = b*x1 + (a - (a/b)*b)*y1
//          = a*y1 + b*(x1 - (a/b)*y1)
//       so x = y1, y = x1 - (a/b)*y1
```

### Iterative Formulation (O(1) space)

```
extgcd_iter(a, b):
    old_r, r = a, b
    old_s, s = 1, 0   // x coefficients
    old_t, t = 0, 1   // y coefficients

    while r != 0:
        q = old_r / r
        old_r, r = r, old_r - q*r
        old_s, s = s, old_s - q*s
        old_t, t = t, old_t - q*t

    return (old_r, old_s, old_t)  // (gcd, x, y)
```

---

## Modular Inverse

`a^{-1} mod m` exists iff `gcd(a, m) = 1`. Compute via extended GCD:

```
(g, x, y) = extgcd(a, m)
if g != 1: no inverse exists
return ((x % m) + m) % m   // ensure positive result
```

**Fermat's little theorem alternative** (only when m is prime):
`a^{-1} ≡ a^{m-2} (mod m)` — O(log m) via fast exponentiation.
Extended GCD is preferred when m is not prime or when Bézout coefficients are needed directly.

---

## Linear Diophantine Equations

Solve `ax + by = c` over integers:

```
1. g = gcd(a, b); if c % g != 0: no solution
2. Find x0, y0 such that a*x0 + b*y0 = g  (via extgcd)
3. Multiply: a*(x0 * c/g) + b*(y0 * c/g) = c
4. General solution:
   x = x0*(c/g) + (b/g)*t
   y = y0*(c/g) - (a/g)*t
   for any t ∈ ℤ
```

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| GCD computation | O(log min(a,b)) | O(1) iterative |
| Bézout coefficients | O(log min(a,b)) | O(log) recursive / O(1) iterative |
| Modular inverse | O(log m) | O(1) |
| Number of recursion steps | ≤ 2 * log_φ(min(a,b)) | — |

The number of steps is bounded by the Fibonacci sequence: consecutive Fibonacci numbers are the worst case for Euclidean GCD (require the most steps). For 64-bit integers: at most ~93 steps.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// EXTENDED GCD — Recursive
// Returns gcd(a,b). Sets x,y such that a*x + b*y = gcd(a,b).
// ================================================================
ll extgcd(ll a, ll b, ll& x, ll& y) {
    if (b == 0) {
        x = 1; y = 0;
        return a;
    }
    ll x1, y1;
    ll g = extgcd(b, a % b, x1, y1);
    x = y1;
    y = x1 - (a / b) * y1;
    return g;
}

// ================================================================
// EXTENDED GCD — Iterative (O(1) space)
// ================================================================
ll extgcd_iter(ll a, ll b, ll& x, ll& y) {
    ll old_r = a, r = b;
    ll old_s = 1, s = 0; // x-coefficients
    ll old_t = 0, t = 1; // y-coefficients

    while (r != 0) {
        ll q = old_r / r;
        ll tmp;

        tmp = r;    r    = old_r - q * r;    old_r = tmp;
        tmp = s;    s    = old_s - q * s;    old_s = tmp;
        tmp = t;    t    = old_t - q * t;    old_t = tmp;
    }

    x = old_s;
    y = old_t;
    return old_r; // gcd
}

// ================================================================
// MODULAR INVERSE — using Extended GCD
// Returns a^{-1} mod m, or -1 if gcd(a,m) != 1
// ================================================================
ll mod_inv(ll a, ll m) {
    ll x, y;
    ll g = extgcd(a, m, x, y);
    if (g != 1) return -1; // inverse does not exist
    return (x % m + m) % m;
}

// ================================================================
// MODULAR INVERSE — Fermat's little theorem (m must be prime)
// ================================================================
ll powmod(ll base, ll exp, ll mod) {
    ll result = 1;
    base %= mod;
    while (exp > 0) {
        if (exp & 1) result = result * base % mod;
        base = base * base % mod;
        exp >>= 1;
    }
    return result;
}

ll mod_inv_fermat(ll a, ll p) {
    return powmod(a, p - 2, p); // p must be prime
}

// ================================================================
// LINEAR DIOPHANTINE EQUATION: ax + by = c
// Returns false if no solution exists.
// Sets one particular solution (x0, y0).
// General: x = x0 + (b/g)*t, y = y0 - (a/g)*t, t ∈ ℤ
// ================================================================
bool linear_diophantine(ll a, ll b, ll c,
                         ll& x0, ll& y0, ll& g) {
    g = extgcd(a, b, x0, y0);
    if (c % g != 0) return false;  // no solution
    ll scale = c / g;
    x0 *= scale;
    y0 *= scale;
    return true;
}

// ================================================================
// MINIMUM POSITIVE x SOLUTION of ax + by = c
// Among all integer solutions, find the one with smallest x > 0
// ================================================================
ll min_positive_x(ll a, ll b, ll c) {
    ll x0, y0, g;
    if (!linear_diophantine(a, b, c, x0, y0, g)) return -1;
    ll step = b / g; // step size for x in general solution
    // x = x0 + step * t; find smallest positive x
    // t = ceil(-x0 / step)
    ll t = (-x0 + step - 1) / step; // ceiling division
    if (step < 0) { step = -step; t = (x0 + step - 1) / step; }
    ll x = x0 + (b / g) * t;
    // Adjust to smallest positive
    x = ((x0 % (b/g)) + (b/g)) % (b/g);
    if (x == 0) x = b / g;
    return x;
}

// ================================================================
// CHINESE REMAINDER THEOREM (CRT) — two congruences
// Solve: x ≡ r1 (mod m1), x ≡ r2 (mod m2)
// Returns (solution, lcm) or (-1, -1) if inconsistent
// ================================================================
pair<ll,ll> crt2(ll r1, ll m1, ll r2, ll m2) {
    ll x, y;
    ll g = extgcd(m1, m2, x, y);
    if ((r2 - r1) % g != 0) return {-1, -1}; // inconsistent

    ll lcm = m1 / g * m2; // avoid overflow
    ll diff = (r2 - r1) / g;
    // x ≡ r1 + m1 * (diff * x_bezout mod (m2/g)) (mod lcm)
    ll t = ((__int128)diff * x % (m2/g) + m2/g) % (m2/g);
    ll sol = (r1 + (__int128)m1 * t) % lcm;
    return {sol, lcm};
}

// ================================================================
// CRT — general: solve system of congruences x ≡ r_i (mod m_i)
// ================================================================
pair<ll,ll> crt(vector<ll>& r, vector<ll>& m) {
    ll cur_r = r[0], cur_m = m[0];
    for (int i = 1; i < (int)r.size(); i++) {
        auto [sol, lcm] = crt2(cur_r, cur_m, r[i], m[i]);
        if (sol == -1) return {-1, -1};
        cur_r = sol; cur_m = lcm;
    }
    return {cur_r, cur_m};
}

// ================================================================
// VERIFY Bézout: checks a*x + b*y == gcd(a,b)
// ================================================================
bool verify(ll a, ll b, ll x, ll y, ll g) {
    return a * x + b * y == g;
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Basic extgcd
    {
        ll a = 35, b = 15, x, y;
        ll g = extgcd(a, b, x, y);
        printf("extgcd(%lld, %lld) = %lld\n", a, b, g);
        printf("  Bezout: %lld*%lld + %lld*%lld = %lld  [%s]\n",
            a, x, b, y, a*x+b*y,
            verify(a,b,x,y,g) ? "OK" : "FAIL");
    }

    // Iterative version
    {
        ll a = 99, b = 78, x, y;
        ll g = extgcd_iter(a, b, x, y);
        printf("\nextgcd_iter(%lld, %lld) = %lld\n", a, b, g);
        printf("  Bezout: %lld*%lld + %lld*%lld = %lld  [%s]\n",
            a, x, b, y, a*x+b*y,
            verify(a,b,x,y,g) ? "OK" : "FAIL");
    }

    // Modular inverse
    {
        printf("\nModular inverses:\n");
        for (auto [a, m] : vector<pair<ll,ll>>{{3,7},{7,13},{4,9},{6,9}}) {
            ll inv = mod_inv(a, m);
            if (inv == -1)
                printf("  %lld^-1 mod %lld = does not exist\n", a, m);
            else
                printf("  %lld^-1 mod %lld = %lld  (check: %lld)\n",
                       a, m, inv, a*inv%m);
        }
    }

    // Linear Diophantine
    {
        printf("\nLinear Diophantine equations:\n");
        ll a=6, b=10, c=14, x0, y0, g;
        if (linear_diophantine(a, b, c, x0, y0, g))
            printf("  %lldu + %lldv = %lld: u=%lld, v=%lld (g=%lld)\n",
                   a, b, c, x0, y0, g);
        else
            printf("  %lldu + %lldv = %lld: no solution\n", a, b, c);

        a=3; b=5; c=7;
        if (linear_diophantine(a, b, c, x0, y0, g))
            printf("  %lldu + %lldv = %lld: u=%lld, v=%lld\n",
                   a, b, c, x0, y0);
        else
            printf("  %lldu + %lldv = %lld: no solution\n", a, b, c);
    }

    // CRT
    {
        printf("\nCRT:\n");
        // x ≡ 2 (mod 3), x ≡ 3 (mod 5), x ≡ 2 (mod 7) => x ≡ 23 (mod 105)
        vector<ll> r = {2, 3, 2}, m = {3, 5, 7};
        auto [sol, lcm] = crt(r, m);
        printf("  x ≡ %lld (mod %lld)  [expected: x≡23 mod 105]\n", sol, lcm);
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Numerics;

public static class ExtGCD {

    // ================================================================
    // EXTENDED GCD — Iterative, O(log min(a,b)) time, O(1) space
    // Returns gcd(a,b), sets x,y s.t. a*x + b*y = gcd(a,b)
    // ================================================================
    public static long Compute(long a, long b, out long x, out long y) {
        long oldR = a, r = b;
        long oldS = 1, s = 0;
        long oldT = 0, t = 1;

        while (r != 0) {
            long q = oldR / r;
            (oldR, r) = (r, oldR - q * r);
            (oldS, s) = (s, oldS - q * s);
            (oldT, t) = (t, oldT - q * t);
        }

        x = oldS;
        y = oldT;
        return oldR; // gcd
    }

    // ================================================================
    // EXTENDED GCD — Recursive
    // ================================================================
    public static long Recursive(long a, long b, out long x, out long y) {
        if (b == 0) { x = 1; y = 0; return a; }
        long g = Recursive(b, a % b, out long x1, out long y1);
        x = y1;
        y = x1 - (a / b) * y1;
        return g;
    }

    // ================================================================
    // MODULAR INVERSE — a^{-1} mod m, returns -1 if doesn't exist
    // ================================================================
    public static long ModInverse(long a, long m) {
        long g = Compute(a, m, out long x, out _);
        if (g != 1) return -1;
        return (x % m + m) % m;
    }

    // ================================================================
    // MODULAR INVERSE — Fermat's little theorem (m prime only)
    // ================================================================
    public static long ModInverseFermat(long a, long p) {
        return ModPow(a, p - 2, p);
    }

    private static long ModPow(long b, long e, long m) {
        long r = 1; b %= m;
        while (e > 0) {
            if ((e & 1) == 1) r = (long)((BigInteger)r * b % m);
            b = (long)((BigInteger)b * b % m);
            e >>= 1;
        }
        return r;
    }

    // ================================================================
    // LINEAR DIOPHANTINE: ax + by = c
    // Returns false if no solution. Sets particular solution (x0, y0).
    // General: x = x0 + (b/g)*t, y = y0 - (a/g)*t
    // ================================================================
    public static bool LinearDiophantine(
        long a, long b, long c,
        out long x0, out long y0, out long g)
    {
        g = Compute(a, b, out x0, out y0);
        if (c % g != 0) return false;
        long scale = c / g;
        x0 *= scale;
        y0 *= scale;
        return true;
    }

    // ================================================================
    // CRT — two congruences: x ≡ r1 (mod m1), x ≡ r2 (mod m2)
    // Returns (solution mod lcm, lcm), or (-1,-1) if inconsistent
    // ================================================================
    public static (long sol, long lcm) CRT2(
        long r1, long m1, long r2, long m2)
    {
        long g = Compute(m1, m2, out long x, out _);
        if ((r2 - r1) % g != 0) return (-1, -1);

        long lcm = m1 / g * m2;
        long diff = (r2 - r1) / g;
        long t = (long)((BigInteger)diff * x % (m2 / g));
        t = (t % (m2 / g) + m2 / g) % (m2 / g);
        long sol = (long)((r1 + (BigInteger)m1 * t) % lcm);
        return (sol, lcm);
    }

    // ================================================================
    // CRT — general system
    // ================================================================
    public static (long sol, long lcm) CRT(
        List<long> r, List<long> m)
    {
        long curR = r[0], curM = m[0];
        for (int i = 1; i < r.Count; i++) {
            var (sol, lcm) = CRT2(curR, curM, r[i], m[i]);
            if (sol == -1) return (-1, -1);
            curR = sol; curM = lcm;
        }
        return (curR, curM);
    }

    // ================================================================
    // Usage
    // ================================================================
    public static void Main() {
        // Basic
        long g = Compute(35, 15, out long x, out long y);
        Console.WriteLine($"gcd(35,15)={g}, x={x}, y={y}, verify={35*x+15*y}");

        g = Recursive(99, 78, out x, out y);
        Console.WriteLine($"gcd(99,78)={g}, x={x}, y={y}, verify={99*x+78*y}");

        // Modular inverse
        Console.WriteLine("\nModular inverses:");
        foreach (var (a, m) in new[]{(3L,7L),(7L,13L),(4L,9L),(6L,9L)}) {
            long inv = ModInverse(a, m);
            if (inv == -1) Console.WriteLine($"  {a}^-1 mod {m} = N/A");
            else Console.WriteLine($"  {a}^-1 mod {m} = {inv}  (check: {a*inv%m})");
        }

        // Linear Diophantine
        Console.WriteLine("\nLinear Diophantine:");
        if (LinearDiophantine(6, 10, 14, out long x0, out long y0, out g))
            Console.WriteLine($"  6u+10v=14: u={x0}, v={y0}, g={g}");

        // CRT
        Console.WriteLine("\nCRT:");
        var (sol, lcm) = CRT(new List<long>{2,3,2}, new List<long>{3,5,7});
        Console.WriteLine($"  x={sol} mod {lcm}  [expected: 23 mod 105]");
    }
}
```

---

## Correctness Proof Sketch

**Invariant:** At each step of the iterative algorithm, the following hold simultaneously:

```
old_r = a * old_s + b * old_t
r     = a * s     + b * t
```

**Base:** `old_r = a = a*1 + b*0`, `r = b = a*0 + b*1`. ✓

**Step:** After the update with quotient `q`:
```
new_old_r = r               = a*s + b*t             ✓
new_r     = old_r - q*r     = (a*old_s+b*old_t) - q*(a*s+b*t)
                             = a*(old_s - q*s) + b*(old_t - q*t)
                             = a*new_s + b*new_t     ✓
```

**Termination:** When `r = 0`, `old_r = gcd(a,b)` and `a*old_s + b*old_t = gcd(a,b)`. ✓

---

## Coefficient Bounds

The Bézout coefficients satisfy:

```
|x| ≤ b / (2 * gcd(a,b))
|y| ≤ a / (2 * gcd(a,b))
```

For `a, b ≤ 10^18`, the coefficients fit in 64-bit integers. However, intermediate products like `a * x` can overflow `int64`. Use `__int128` in C++ or `BigInteger` in C# for intermediate computations.

---

## Pitfalls

- **Negative inputs** — the algorithm works correctly for negative inputs. `gcd(-6, 10) = gcd(6, 10) = 2`. However, the sign convention for GCD may vary: some implementations return a negative GCD for negative inputs. Always normalize: `if (g < 0) { g = -g; x = -x; y = -y; }`.
- **Modular inverse normalization** — the Bézout coefficient `x` satisfies `a*x ≡ gcd(a,m) (mod m)`. If `gcd(a,m) = 1`, then `x` is the inverse, but `x` may be negative. Always normalize: `(x % m + m) % m`. Returning a negative modular inverse causes subtle bugs in modular arithmetic.
- **Linear Diophantine divisibility check** — `ax + by = c` has solutions iff `gcd(a, b) | c`. Forgetting this check leads to a particular solution `(x0, y0)` that satisfies `ax0 + by0 = gcd(a,b) * (c/gcd)` but with `c/gcd` being a non-integer, producing wrong scaled solutions.
- **CRT overflow** — computing `r1 + m1 * t` can overflow 64-bit integers when `m1` and `t` are both large. Use `__int128` in C++ or `BigInteger`/`checked` arithmetic in C# for intermediate products.
- **CRT inconsistency check** — two congruences `x ≡ r1 (mod m1)` and `x ≡ r2 (mod m2)` are compatible iff `gcd(m1, m2) | (r2 - r1)`. Without this check, the algorithm computes a nonsensical solution. Always verify `(r2 - r1) % gcd(m1, m2) == 0` before proceeding.
- **Recursive stack depth** — the recursive extgcd has depth O(log min(a, b)) ≈ 93 for 64-bit inputs. This is safe on all practical systems (stack frame is tiny). However, for very deep recursion in embedded systems, use the iterative version.
- **extgcd(0, b)** — `gcd(0, b) = b`. The base case `if b == 0: return (a, 1, 0)` handles `extgcd(a, 0)` correctly. But `extgcd(0, b)` requires one step: `extgcd(b, 0 % b) = extgcd(b, 0) = (b, 1, 0)`, then `x = 0, y = 1`. Verify: `0*0 + b*1 = b`. ✓ Some implementations special-case `a = 0` unnecessarily.

---

## Conclusion

The Extended Euclidean Algorithm is the **fundamental number-theoretic primitive**:

- Computes `gcd(a, b)` and Bézout coefficients `(x, y)` in O(log min(a,b)) — optimal time and O(1) space in iterative form.
- Directly yields modular inverses: the most frequently needed operation in modular arithmetic (hashing, NTT, RSA, CRT).
- Solves all linear Diophantine equations `ax + by = c` in closed form: existence check, particular solution, and general solution family.
- Drives the Chinese Remainder Theorem reconstruction for systems of congruences.

**Key takeaway:**  
Always implement the iterative variant for production code — it avoids stack overhead and is simpler to reason about. The three parallel update lines `(old_r, r) <- (r, old_r - q*r)` and similarly for `s` and `t` are the complete algorithm. Verify correctness with `assert(a * x + b * y == g)` after every call during development.
