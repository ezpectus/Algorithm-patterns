# Baby-step Giant-step (BSGS) — Discrete Logarithm

## Origin & Motivation

Baby-step Giant-step (BSGS) was introduced by Daniel Shanks in 1971 as an algorithm for computing **discrete logarithms** in cyclic groups. Given a generator `g`, a target `y`, and a modulus `p`, it finds `x` such that:

```
g^x ≡ y (mod p)
```

The naive approach tries all `x` from `0` to `p-1` in O(p) time. BSGS uses a **meet-in-the-middle** decomposition: write `x = i*m - j` where `m = ⌈√p⌉`. Then `g^x = y` becomes `g^{im} = y * g^j`, separating the equation into a "giant step" part (computing `(g^m)^i` for i = 0..m) and a "baby step" part (computing `y * g^j` for j = 0..m), then finding a collision.

This achieves **O(√p)** time and space — a quadratic speedup over brute force.

Complexity: **O(√p * log p)** time (with hash map: O(√p) expected), **O(√p)** space.

---

## Where It Is Used

- Cryptanalysis of Diffie-Hellman key exchange (small subgroup attacks)
- Breaking discrete logarithm in groups of small order
- Number theory: index-calculus preprocessing
- Pohlig-Hellman algorithm (reduces DLP to prime-order subgroups)
- Computing square roots in finite fields
- Elliptic curve discrete logarithm (ECDLP) for small groups
- Competitive programming: discrete log modulo p queries

---

## Problem Variants

| Variant | Problem | Algorithm |
|---|---|---|
| Standard | g^x ≡ y (mod p), p prime | BSGS O(√p) |
| Generalized | g^x ≡ y (mod m), gcd(g,m)≠1 possible | Extended BSGS |
| Any group | g^x = y in cyclic group of order n | BSGS O(√n) |
| Multiple targets | g^x ≡ y_i (mod p) for many y_i | Precompute baby steps once |

---

## Core Algorithm (Standard BSGS)

### Setup

`p` is prime (or at least `gcd(g, p) = 1`). Find `x ∈ [0, p-1]` s.t. `g^x ≡ y (mod p)`.

Let `m = ⌈√p⌉`. Write `x = i*m - j` for `i ∈ [1, m]`, `j ∈ [0, m-1]`.

Then:
```
g^{im - j} ≡ y (mod p)
g^{im}     ≡ y * g^j (mod p)
(g^m)^i    ≡ y * g^j (mod p)
```

### Baby Steps (precompute)

For `j = 0` to `m-1`:
- Compute `baby[j] = y * g^j mod p`
- Store in hash map: `table[y * g^j mod p] = j`

### Giant Steps (search)

Let `G = g^m mod p`. For `i = 1` to `m`:
- Compute `giant_i = G^i mod p = (g^m)^i mod p`
- If `giant_i` is in `table`: `x = i*m - table[giant_i]`, verify and return

### Answer

`x = i*m - j` where the collision `G^i = y * g^j` was found.

---

## Generalized BSGS (arbitrary modulus)

When `m` is not prime or `gcd(g, m) ≠ 1`, the standard approach fails because `g^j` may not cover all residues. The generalized BSGS handles this by:

1. Factor out powers of `gcd(g, m)` from both sides iteratively.
2. Once `gcd(g^k, m) | y * gcd(g^{k-1}, m)` for some `k`, reduce to a standard BSGS on the remaining problem.
3. Apply standard BSGS to `(g/gcd)^x ≡ (y/gcd) (mod m/gcd)`.

```
Solve g^x ≡ y (mod m):

k = 0, factor = 1, cur_m = m, cur_y = y
while gcd(g, cur_m) != 1:
    if cur_y % gcd(g, cur_m) != 0: no solution
    k++
    cur_y /= gcd(g, cur_m)
    cur_m /= gcd(g, cur_m)
    factor = factor * g / gcd(g, m)  // track removed powers

Now solve (g^k_inv) * g^x ≡ cur_y (mod cur_m) where gcd(g, cur_m) = 1
Apply standard BSGS, then recover actual x = solution + k
```

---

## Complexity Analysis

| Phase | Time | Space |
|---|---|---|
| Baby steps (precompute) | O(m) = O(√p) | O(m) = O(√p) hash map |
| Giant steps (search) | O(m) = O(√p) | O(1) |
| Hash map operations | O(1) expected | — |
| Total time | O(√p) expected | O(√p) |
| With sorted array + binary search | O(√p log √p) | O(√p) |

For p ≈ 10^18: m ≈ 10^9 — too large. BSGS is practical for p ≤ 10^12 (m ≤ 10^6).

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;
using lll = __int128;

ll powmod(ll base, ll exp, ll mod) {
    ll result = 1; base %= mod;
    while (exp > 0) {
        if (exp & 1) result = (lll)result * base % mod;
        base = (lll)base * base % mod;
        exp >>= 1;
    }
    return result;
}

// ================================================================
// STANDARD BSGS
// Solve g^x ≡ y (mod p), p prime, gcd(g, p) = 1.
// Returns x in [0, p-1], or -1 if no solution.
// ================================================================
ll bsgs(ll g, ll y, ll p) {
    g %= p; y %= p;
    if (y == 1) return 0; // g^0 = 1
    if (g == 0) return y == 0 ? 1 : -1; // degenerate

    ll m = (ll)ceil(sqrt((double)p)) + 1;

    // Baby steps: table[y * g^j mod p] = j
    unordered_map<ll, ll> table;
    ll gj = y % p; // g^0 * y
    for (ll j = 0; j < m; j++) {
        table[gj] = j;
        gj = (lll)gj * g % p;
    }

    // Giant steps: compute G = g^m, then G^i for i = 1..m
    ll G = powmod(g, m, p);
    ll Gi = G;
    for (ll i = 1; i <= m; i++) {
        auto it = table.find(Gi);
        if (it != table.end()) {
            ll x = i * m - it->second;
            if (x >= 0) return x % (p - 1); // x mod ord(g), ord(g) | p-1
        }
        Gi = (lll)Gi * G % p;
    }
    return -1; // no solution
}

// ================================================================
// BSGS — Search in range [lo, hi]
// Find x in [lo, hi] such that g^x ≡ y (mod p)
// ================================================================
ll bsgs_range(ll g, ll y, ll p, ll lo, ll hi) {
    g %= p; y %= p;
    ll m = (ll)ceil(sqrt((double)(hi - lo))) + 1;

    // Adjust: g^x = y => g^{x-lo} = y * g^{-lo} mod p
    ll inv_g_lo = powmod(powmod(g, lo, p), p - 2, p); // g^{-lo} mod p
    ll y_adj = (lll)y * inv_g_lo % p;

    // Baby steps: table[y_adj * g^j] = j
    unordered_map<ll, ll> table;
    ll gj = y_adj;
    for (ll j = 0; j <= m; j++) {
        table[gj] = j;
        gj = (lll)gj * g % p;
    }

    // Giant steps
    ll G = powmod(g, m, p);
    ll Gi = G;
    for (ll i = 1; i <= m; i++) {
        auto it = table.find(Gi);
        if (it != table.end()) {
            ll x = lo + i * m - it->second;
            if (x >= lo && x <= hi) return x;
        }
        Gi = (lll)Gi * G % p;
    }
    return -1;
}

// ================================================================
// GENERALIZED BSGS
// Solve g^x ≡ y (mod m) for arbitrary m (not necessarily prime).
// gcd(g, m) may be != 1.
// Returns smallest non-negative x, or -1 if no solution.
// ================================================================
ll extgcd(ll a, ll b, ll& x, ll& y) {
    ll r=b, old_r=a, s=0, old_s=1, t=1, old_t=0;
    while (r) {
        ll q=old_r/r;
        swap(r, old_r -= q*r);
        swap(s, old_s -= q*s);
        swap(t, old_t -= q*t);
    }
    x=old_s; y=old_t; return old_r;
}

ll mod_inv(ll a, ll m) {
    ll x,y; ll g=extgcd(a,m,x,y);
    return g==1 ? (x%m+m)%m : -1;
}

ll bsgs_general(ll g, ll y, ll mod) {
    g %= mod; y %= mod;
    if (mod == 1) return 0;

    // Handle leading powers of gcd(g, mod)
    ll factor = 1, cur_mod = mod, cur_y = y;
    ll k = 0;
    ll d;
    while ((d = __gcd(g, cur_mod)) != 1) {
        if (cur_y % d != 0) return -1; // no solution
        cur_y /= d;
        cur_mod /= d;
        factor = (lll)factor * (g / d) % cur_mod;
        k++;
        if (factor == cur_y % cur_mod) return k; // early exit
    }

    // Now gcd(g, cur_mod) = 1; solve g^{x-k} ≡ cur_y * factor^{-1} (mod cur_mod)
    ll inv_factor = mod_inv(factor % cur_mod, cur_mod);
    if (inv_factor == -1) return -1;
    ll new_y = (lll)cur_y * inv_factor % cur_mod;

    // Standard BSGS on g^z ≡ new_y (mod cur_mod)
    ll m = (ll)ceil(sqrt((double)cur_mod)) + 1;

    unordered_map<ll, ll> table;
    ll gj = new_y;
    for (ll j = 0; j <= m; j++) {
        if (!table.count(gj)) table[gj] = j;
        gj = (lll)gj * g % cur_mod;
    }

    ll G = powmod(g, m, cur_mod);
    ll Gi = G;
    for (ll i = 1; i <= m; i++) {
        auto it = table.find(Gi % cur_mod);
        if (it != table.end()) {
            ll z = i * m - it->second;
            if (z >= 0) return z + k;
        }
        Gi = (lll)Gi * G % cur_mod;
    }
    return -1;
}

// ================================================================
// BSGS FOR MULTIPLE TARGETS
// Given g, p, and many targets y_i, find x_i s.t. g^{x_i} ≡ y_i (mod p)
// Precompute baby steps once, reuse for all targets.
// ================================================================
struct BSGSMulti {
    ll g, p, m;
    ll G; // g^m mod p
    unordered_map<ll, ll> table; // baby[val] = exponent j such that g^j = val

    // Precompute only the giant-step base; baby steps vary per target
    // Actually: precompute the full power table of g
    // For multiple queries: precompute g^0, g^1, ..., g^{m-1}
    // Then for each y: compute y * g^j and look it up

    // Alternative: precompute all g^i for i in [0, p-1] in O(p) space
    // For truly multiple queries on same g,p: build full DL table
    unordered_map<ll, ll> full_table;

    void build(ll _g, ll _p) {
        g = _g; p = _p;
        ll cur = 1;
        for (ll i = 0; i < p - 1; i++) {
            if (!full_table.count(cur)) full_table[cur] = i;
            cur = (__int128)cur * g % p;
        }
    }

    ll query(ll y) {
        y %= p;
        auto it = full_table.find(y);
        return it != full_table.end() ? it->second : -1;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // Standard BSGS: find x such that 3^x ≡ 7 (mod 11)
    // 3^0=1, 3^1=3, 3^2=9, 3^3=5, 3^4=4, 3^5=1... wait 11 is prime
    // 3^1=3, 3^2=9, 3^3=27%11=5, 3^4=15%11=4, 3^5=12%11=1, so ord=5
    // 3^6=3, 3^7=9, 3^8=5, 3^9=4, 3^10=1
    // 7 is not a power of 3 mod 11 if ord(3)=5... 
    // Let's use g=2, p=13: 2^1=2,2^2=4,2^3=8,2^4=3,2^5=6,2^6=12,
    //   2^7=11,2^8=9,2^9=5,2^10=10,2^11=7,2^12=1
    // So 2^11 ≡ 7 (mod 13)
    {
        ll g=2, y=7, p=13;
        ll x = bsgs(g, y, p);
        printf("2^x ≡ 7 (mod 13): x = %lld\n", x); // 11
        printf("Verify: 2^%lld mod 13 = %lld\n", x, powmod(g,x,p));
    }

    // No solution case
    {
        // g=3, y=7, p=11: ord(3)=5, powers are {1,3,9,5,4}; 7 not in set
        ll x = bsgs(3, 7, 11);
        printf("\n3^x ≡ 7 (mod 11): x = %lld %s\n", x,
               x==-1 ? "(no solution)" : "");
    }

    // Larger example
    {
        ll g=5, p=1000000007LL;
        ll y=powmod(g, 123456789LL, p); // construct known answer
        ll x = bsgs(g, y, p);
        printf("\n5^x ≡ %lld (mod 10^9+7): x = %lld\n", y, x);
        printf("Verify: %s\n", powmod(g,x,p)==y ? "OK" : "FAIL");
    }

    // Range query
    {
        ll g=2, y=powmod(2, 50, 101), p=101;
        ll x = bsgs_range(g, y, p, 30, 70);
        printf("\n2^x ≡ %lld (mod 101) in [30,70]: x = %lld\n", y, x);
    }

    // Generalized BSGS (non-prime modulus)
    {
        // 2^x ≡ 8 (mod 16): 2^3=8 but gcd(2,16)=2≠1
        ll x = bsgs_general(2, 8, 16);
        printf("\n2^x ≡ 8 (mod 16): x = %lld\n", x); // 3
        if (x != -1)
            printf("Verify: 2^%lld mod 16 = %lld\n", x, powmod(2,x,16));
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

public static class BSGS {

    private static long PowMod(long b, long e, long m) {
        long r = 1; b %= m;
        while (e > 0) {
            if ((e & 1) == 1) r = (long)((BigInteger)r * b % m);
            b = (long)((BigInteger)b * b % m);
            e >>= 1;
        }
        return r;
    }

    private static long ModInv(long a, long m) {
        long oldR=a, r=m, oldS=1, s=0;
        while (r != 0) {
            long q = oldR/r;
            (oldR,r) = (r, oldR-q*r);
            (oldS,s) = (s, oldS-q*s);
        }
        return oldR == 1 ? (oldS % m + m) % m : -1;
    }

    // ================================================================
    // STANDARD BSGS — g^x ≡ y (mod p), p prime
    // Returns x in [0, p-1] or -1 if no solution.
    // ================================================================
    public static long Solve(long g, long y, long p) {
        g %= p; y %= p;
        if (y == 1) return 0;
        if (g == 0) return y == 0 ? 1 : -1;

        long m = (long)Math.Ceiling(Math.Sqrt(p)) + 1;

        // Baby steps: table[y * g^j mod p] = j
        var table = new Dictionary<long, long>();
        long gj = y;
        for (long j = 0; j <= m; j++) {
            if (!table.ContainsKey(gj)) table[gj] = j;
            gj = (long)((BigInteger)gj * g % p);
        }

        // Giant steps
        long G = PowMod(g, m, p);
        long Gi = G;
        for (long i = 1; i <= m; i++) {
            if (table.TryGetValue(Gi, out long j)) {
                long x = i * m - j;
                if (x >= 0) return x % (p - 1);
            }
            Gi = (long)((BigInteger)Gi * G % p);
        }
        return -1;
    }

    // ================================================================
    // BSGS — Search in range [lo, hi]
    // ================================================================
    public static long SolveRange(long g, long y, long p, long lo, long hi) {
        g %= p; y %= p;
        long m = (long)Math.Ceiling(Math.Sqrt(hi - lo)) + 1;

        long invGLo = PowMod(PowMod(g, lo, p), p - 2, p);
        long yAdj   = (long)((BigInteger)y * invGLo % p);

        var table = new Dictionary<long, long>();
        long gj = yAdj;
        for (long j = 0; j <= m; j++) {
            if (!table.ContainsKey(gj)) table[gj] = j;
            gj = (long)((BigInteger)gj * g % p);
        }

        long G = PowMod(g, m, p);
        long Gi = G;
        for (long i = 1; i <= m; i++) {
            if (table.TryGetValue(Gi, out long j)) {
                long x = lo + i * m - j;
                if (x >= lo && x <= hi) return x;
            }
            Gi = (long)((BigInteger)Gi * G % p);
        }
        return -1;
    }

    // ================================================================
    // GENERALIZED BSGS — g^x ≡ y (mod m), arbitrary m
    // ================================================================
    public static long SolveGeneral(long g, long y, long mod) {
        g %= mod; y %= mod;
        if (mod == 1) return 0;

        long factor = 1, curMod = mod, curY = y, k = 0;
        long d;
        while ((d = GCD(g, curMod)) != 1) {
            if (curY % d != 0) return -1;
            curY  /= d;
            curMod /= d;
            factor = (long)((BigInteger)factor * (g / d) % curMod);
            k++;
            if (factor == curY % curMod) return k;
        }

        long invFactor = ModInv(factor % curMod, curMod);
        if (invFactor == -1) return -1;
        long newY = (long)((BigInteger)curY * invFactor % curMod);

        long m2 = (long)Math.Ceiling(Math.Sqrt(curMod)) + 1;
        var table = new Dictionary<long, long>();
        long gj2 = newY;
        for (long j = 0; j <= m2; j++) {
            if (!table.ContainsKey(gj2)) table[gj2] = j;
            gj2 = (long)((BigInteger)gj2 * g % curMod);
        }

        long G2 = PowMod(g, m2, curMod);
        long Gi2 = G2;
        for (long i = 1; i <= m2; i++) {
            long key = Gi2 % curMod;
            if (table.TryGetValue(key, out long j2)) {
                long z = i * m2 - j2;
                if (z >= 0) return z + k;
            }
            Gi2 = (long)((BigInteger)Gi2 * G2 % curMod);
        }
        return -1;
    }

    private static long GCD(long a, long b) =>
        b == 0 ? a : GCD(b, a % b);

    public static void Main() {
        // 2^x ≡ 7 (mod 13) = 11
        long x = Solve(2, 7, 13);
        Console.WriteLine($"2^x ≡ 7 (mod 13): x={x}");
        Console.WriteLine($"Verify: 2^{x} mod 13 = {PowMod(2,x,13)}");

        // No solution
        Console.WriteLine($"\n3^x ≡ 7 (mod 11): {Solve(3,7,11)}");

        // Large example
        long p = 1_000_000_007L;
        long y = PowMod(5, 123456789L, p);
        x = Solve(5, y, p);
        Console.WriteLine($"\n5^x ≡ {y} (mod 10^9+7): x={x}");
        Console.WriteLine($"Verify: {(PowMod(5,x,p)==y?"OK":"FAIL")}");

        // Range
        x = SolveRange(2, PowMod(2,50,101), 101, 30, 70);
        Console.WriteLine($"\n2^x in [30,70]: x={x}");

        // Generalized
        x = SolveGeneral(2, 8, 16);
        Console.WriteLine($"\n2^x ≡ 8 (mod 16): x={x}");
        if (x != -1)
            Console.WriteLine($"Verify: 2^{x} mod 16 = {PowMod(2,x,16)}");
    }
}
```

---

## Meet-in-the-Middle Visualization

```
Goal: find x such that g^x ≡ y (mod p)
Write: x = i*m - j,  m = ⌈√p⌉

Baby steps (j = 0..m):           Giant steps (i = 1..m):
  y * g^0 = y                      G^1 = g^m
  y * g^1                          G^2 = g^{2m}
  y * g^2                          G^3 = g^{3m}
  ...                              ...
  y * g^{m-1}                      G^m = g^{m^2} ≈ g^p

Store in hash map ──────► Look up giant step values
                               in baby step table
           Collision found: G^i = y * g^j
           ⟹  g^{im} = y * g^j
           ⟹  g^{im-j} = y
           ⟹  x = im - j
```

---

## Pohlig-Hellman + BSGS

When the group order `n = p-1` is smooth (has only small prime factors), Pohlig-Hellman reduces the DLP to multiple smaller DLPs, each solved by BSGS:

```
Factor n = q1^{e1} * q2^{e2} * ... * qk^{ek}

For each prime power qi^{ei}:
    Find x mod qi^{ei} using BSGS in subgroup of order qi^{ei}
    Cost: O(ei * sqrt(qi))

Combine results via CRT to get x mod n
Total cost: O(sum_i ei * sqrt(qi))
```

For n smooth up to B: O(k * √B) vs O(√n) for plain BSGS — exponential improvement when n has small factors.

---

## Pitfalls

- **m must be strictly greater than √p, not equal** — use `m = ceil(sqrt(p)) + 1` to ensure the search covers the full range. With `m = floor(sqrt(p))`, values near `p` are missed when `i*m - j` slightly exceeds the covered range. The +1 provides a safety margin.
- **Baby step stores y * g^j, NOT g^j alone** — the baby step table stores `y * g^j mod p`, not just `g^j`. The collision is between `(g^m)^i` and `y * g^j`. Storing `g^j` and checking `G^i / y` instead requires a modular division per giant step — correct but slower without the precomputed adjustment.
- **Hash map collision vs mathematical collision** — hash maps have collision probability O(m/|hash_range|). Use a good hash function or `unordered_map` with a custom hash. A poor hash increases expected query time from O(1) to O(m), degrading overall to O(m^2).
- **Result normalization: x mod (p-1), not x mod p** — when `g` is a primitive root mod `p`, its order is `p-1`. The solution `x = i*m - j` may be larger than `p-1`. Always reduce: `x % (p-1)`. When `g` is not a primitive root, reduce modulo the actual order `ord_p(g)`.
- **Generalized BSGS: factor loop may not terminate** — if `y = 0` and `g ≠ 0`, the factoring loop divides `cur_y` by `gcd(g, cur_mod)` repeatedly. Since `0 % d = 0` always, this loops until `cur_mod = 1`, at which point any `x` is a solution. Handle `y = 0` separately: the only solution is `x` s.t. `p | g^x`, which requires specific analysis.
- **No solution vs solution = 0** — returning -1 for "no solution" conflicts if the actual answer could be 0 (i.e., `g^0 = 1 ≡ y`). Always check `y ≡ 1 (mod p)` separately before running the main algorithm and return 0 immediately. Without this, the main loop may miss the x=0 case since the baby step table stores y*g^0 = y, which equals G^0 = 1 only when y = 1.
- **__int128 / BigInteger for intermediate products** — `g * gj mod p` with `g, gj < p ≈ 10^9` gives an intermediate value up to `10^18` which fits in `int64`. But for `p ≈ 10^12`, the product `g * gj < 10^24` — must use `__int128` in C++ or `BigInteger` in C#. Forgetting this silently overflows, producing wrong baby step values.

---

## Complexity Summary

| Algorithm | Time | Space | Use case |
|---|---|---|---|
| Brute force | O(p) | O(1) | p ≤ 10^6 |
| BSGS | O(√p) | O(√p) | p ≤ 10^12 |
| Pohlig-Hellman + BSGS | O(√B * log p) | O(√B) | p-1 is B-smooth |
| Index Calculus | O(e^{√(log p log log p)}) | large | cryptographic p |
| GNFS | Sub-exponential | large | general DLP |

---

## Conclusion

Baby-step Giant-step is the **canonical meet-in-the-middle algorithm for the discrete logarithm problem**:

- Achieves O(√p) time and space by writing the exponent as `x = i*m - j` and finding collisions between precomputed baby steps and online giant steps.
- Practical for group orders up to ~10^12 on modern hardware.
- Forms the inner subroutine for Pohlig-Hellman, which reduces large group DLP to many small DLPs each solved by BSGS.
- The generalized variant handles non-prime moduli and non-coprime bases via iterative GCD reduction.

**Key takeaway:**  
BSGS is the first tool to reach for when asked to compute a discrete logarithm for p ≤ 10^12. The implementation is straightforward: precompute m ≈ √p baby steps in a hash map, then test m giant steps for collisions. The only subtlety is the table stores `y * g^j` (not `g^j`) so the collision condition `G^i = y * g^j` directly gives `x = im - j` without division.
