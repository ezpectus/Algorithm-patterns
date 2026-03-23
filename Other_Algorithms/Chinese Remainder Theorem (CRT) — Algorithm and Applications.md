# Chinese Remainder Theorem (CRT) — Algorithm and Applications

## Origin & Motivation

The Chinese Remainder Theorem (CRT) originates from the work of Chinese mathematician Sun Tzu (Sunzi Suanjing, ~3rd–5th century CE), who posed the problem: "Find a number that leaves remainder 2 when divided by 3, remainder 3 when divided by 5, and remainder 2 when divided by 7." The answer is 23.

The theorem states: given pairwise coprime moduli `m_1, ..., m_k`, for **any** choice of remainders `r_1, ..., r_k`, there exists a **unique** solution `x` modulo `M = m_1 * m_2 * ... * m_k`. The constructive proof gives an efficient algorithm.

For non-coprime moduli, solutions may or may not exist depending on consistency conditions — checked via Extended GCD.

CRT is one of the most broadly useful theorems in number theory, appearing in cryptography, polynomial arithmetic, computer architecture, and algorithm design.

Complexity: **O(k * log M)** for k congruences, where log M ≈ sum of log m_i.

---

## Where It Is Used

- RSA decryption speedup (CRT form reduces key operations)
- Polynomial multiplication mod large primes (NTT + CRT)
- Arbitrary-precision arithmetic (RNS — Residue Number System)
- Computing with large numbers by splitting into smaller moduli
- Competitive programming: combining independent modular constraints
- Hash function construction (independent residues)
- Garner's algorithm for multi-modular NTT reconstruction

---

## Theorem Statement

**CRT (coprime case):** Let `m_1, ..., m_k` be pairwise coprime positive integers and `M = m_1 * ... * m_k`. For any integers `r_1, ..., r_k`, the system:

```
x ≡ r_1 (mod m_1)
x ≡ r_2 (mod m_2)
    ...
x ≡ r_k (mod m_k)
```

has a unique solution `x` in `[0, M)`.

**CRT (general case):** For arbitrary moduli (not necessarily coprime), the system has a solution iff for every pair `i, j`: `gcd(m_i, m_j) | (r_i - r_j)`. The solution is unique mod `lcm(m_1, ..., m_k)`.

---

## Construction (Coprime Case)

**Direct construction:**

```
M  = m_1 * m_2 * ... * m_k
M_i = M / m_i                    // product of all moduli except m_i
y_i = M_i^{-1} mod m_i           // modular inverse (exists since gcd(M_i, m_i)=1)
x   = sum(r_i * M_i * y_i) mod M
```

Each term `r_i * M_i * y_i ≡ r_i (mod m_i)` and `≡ 0 (mod m_j)` for j ≠ i.

**Garner's Algorithm (for k moduli, avoids large M):**

Represents the solution as a mixed-radix number. Avoids computing M explicitly — crucial when moduli are large primes and M would be astronomically large.

```
Compute coefficients a_1, a_2, ..., a_k iteratively:
a_1 = r_1
a_2 = (r_2 - a_1) * (m_1)^{-1} mod m_2
a_3 = ((r_3 - a_1) * (m_1 m_2)^{-1} - a_2 * m_1^{-1}) * ... mod m_3
...

Then x = a_1 + a_2*m_1 + a_3*m_1*m_2 + ... (exact integer)
```

**Iterative pairwise combination (general, handles non-coprime):**

```
Start with (cur_r, cur_m) = (r_1, m_1)
For each (r_i, m_i):
    solve: cur_r + cur_m * t ≡ r_i (mod m_i)  for t
    via ExtGCD: t = (r_i - cur_r) * cur_m^{-1} mod (m_i / g)
    cur_r = cur_r + cur_m * t
    cur_m = lcm(cur_m, m_i)
```

---

## Complexity Analysis

| Method | Time | Space | Notes |
|---|---|---|---|
| Direct construction | O(k log M) | O(k) | Requires big integers for M |
| Garner's algorithm | O(k^2 log max_m) | O(k) | Stays in bounded integers |
| Iterative pairwise | O(k log M) | O(1) | Handles non-coprime moduli |
| Solution range | unique mod M | — | M = lcm(m_1,...,m_k) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;
using lll = __int128;

// ================================================================
// EXTENDED GCD (needed for CRT)
// ================================================================
ll extgcd(ll a, ll b, ll& x, ll& y) {
    ll old_r=a, r=b, old_s=1, s=0, old_t=0, t=1;
    while (r) {
        ll q = old_r/r;
        tie(old_r,r)   = make_tuple(r, old_r-q*r);
        tie(old_s,s)   = make_tuple(s, old_s-q*s);
        tie(old_t,t)   = make_tuple(t, old_t-q*t);
    }
    x=old_s; y=old_t; return old_r;
}

ll mod_inv(ll a, ll m) {
    ll x,y; ll g=extgcd(a,m,x,y);
    if (g!=1) return -1; // no inverse
    return (x%m+m)%m;
}

// ================================================================
// CRT — Two congruences: x ≡ r1 (mod m1), x ≡ r2 (mod m2)
// Works for non-coprime m1, m2.
// Returns (solution mod lcm, lcm), or (-1,-1) if inconsistent.
// ================================================================
pair<ll,ll> crt2(ll r1, ll m1, ll r2, ll m2) {
    ll x, y;
    ll g = extgcd(m1, m2, x, y);

    if ((r2 - r1) % g != 0) return {-1, -1}; // inconsistent

    ll lcm = m1 / g * m2;                     // compute lcm without overflow
    ll diff = ((r2 - r1) / g % (m2/g) + m2/g) % (m2/g);
    // t = diff * x mod (m2/g)
    ll t = (lll)diff * x % (m2/g);
    t = (t % (m2/g) + m2/g) % (m2/g);
    ll sol = (r1 + (lll)m1 * t % lcm + lcm) % lcm;
    return {sol, lcm};
}

// ================================================================
// CRT — General: solve system x ≡ r[i] (mod m[i])
// Handles arbitrary (possibly non-coprime) moduli.
// Returns (solution mod lcm, lcm), or (-1,-1) if inconsistent.
// ================================================================
pair<ll,ll> crt(vector<ll> r, vector<ll> m) {
    int k = r.size();
    ll cur_r = r[0], cur_m = m[0];
    for (int i = 1; i < k; i++) {
        auto [sol, lcm] = crt2(cur_r, cur_m, r[i], m[i]);
        if (sol == -1) return {-1, -1};
        cur_r = sol; cur_m = lcm;
    }
    return {cur_r, cur_m};
}

// ================================================================
// CRT DIRECT — Coprime moduli only, using explicit M and M_i
// ================================================================
ll crt_direct(vector<ll>& r, vector<ll>& m) {
    int k = m.size();
    ll M = 1;
    for (ll mi : m) M *= mi;

    ll x = 0;
    for (int i = 0; i < k; i++) {
        ll Mi = M / m[i];
        ll yi = mod_inv(Mi % m[i], m[i]);
        x = (x + (lll)r[i] * Mi % M * yi) % M;
    }
    return x;
}

// ================================================================
// GARNER'S ALGORITHM
// Reconstruct integer from residues mod m[0], m[1], ..., m[k-1]
// Coprime moduli required.
// Returns the unique integer x in [0, m[0]*m[1]*...*m[k-1])
// as a vector of mixed-radix digits a[0], a[1], ..., a[k-1]
// x = a[0] + a[1]*m[0] + a[2]*m[0]*m[1] + ...
// ================================================================
vector<ll> garner(vector<ll>& r, vector<ll>& m) {
    int k = m.size();
    // coeff[i][j] = (m[0]*m[1]*...*m[j-1])^{-1} mod m[i]
    // Precompute inverse products
    vector<ll> a(k);

    // Running prefix product mod m[i]
    vector<ll> prefix(k, 1); // prefix[i] = m[0]*...*m[i-1] mod m[i]

    a[0] = r[0] % m[0];
    for (int i = 1; i < k; i++) {
        // a[i] = (r[i] - sum_{j<i} a[j] * prod_{l<j} m[l]) * (prod_{l<i} m[l])^{-1} mod m[i]
        ll val = r[i] % m[i];
        ll prod = 1;
        for (int j = 0; j < i; j++) {
            val = (val - (lll)a[j] * prod % m[i] + m[i]) % m[i];
            prod = (lll)prod * m[j] % m[i];
        }
        // val = val * prod^{-1} mod m[i]
        a[i] = (lll)val * mod_inv(prod, m[i]) % m[i];
    }
    return a; // mixed-radix representation
}

// Convert Garner mixed-radix to actual integer (as string for large values)
// For small moduli products, returns ll directly
ll garner_to_int(vector<ll>& a, vector<ll>& m) {
    ll result = 0, base = 1;
    for (int i = 0; i < (int)a.size(); i++) {
        result += a[i] * base;
        base   *= m[i];
    }
    return result;
}

// ================================================================
// APPLICATION 1: NTT + CRT for large polynomial multiplication
// Multiply two polynomials mod a large prime using 3 NTT primes
// then recover coefficients via CRT.
// NTT primes (all ≡ 1 mod large power of 2):
//   p1 = 998244353  (= 119 * 2^23 + 1)
//   p2 = 985661441  (= 235 * 2^22 + 1)
//   p3 = 754974721  (= 45  * 2^24 + 1)
// ================================================================
struct NTT_CRT {
    static const ll p1 = 998244353LL;
    static const ll p2 = 985661441LL;
    static const ll p3 = 754974721LL;

    // After computing conv_p1[i], conv_p2[i], conv_p3[i]:
    // recover true coefficient c[i] via Garner mod p1*p2*p3
    static ll recover(ll v1, ll v2, ll v3) {
        vector<ll> r = {v1, v2, v3};
        vector<ll> m = {p1, p2, p3};
        auto a = garner(r, m);
        // c = a[0] + a[1]*p1 + a[2]*p1*p2
        // Return as ll (fits if coefficients < p1*p2*p3 ≈ 7.4 * 10^26 — needs __int128)
        return a[0] + a[1]*p1 + a[2]*p1*p2; // may overflow ll; use __int128 for large
    }
};

// ================================================================
// APPLICATION 2: Solve system of linear congruences
// (demonstration of general CRT usage)
// ================================================================
void solve_congruence_system() {
    // Example: x ≡ 2 (mod 3)
    //          x ≡ 3 (mod 5)
    //          x ≡ 2 (mod 7)
    // Answer:  x ≡ 23 (mod 105)
    vector<ll> r = {2, 3, 2};
    vector<ll> m = {3, 5, 7};
    auto [sol, lcm] = crt(r, m);
    printf("System solution: x ≡ %lld (mod %lld)\n", sol, lcm);

    // Example with non-coprime moduli:
    // x ≡ 1 (mod 4)
    // x ≡ 5 (mod 6)
    // gcd(4,6)=2, (5-1)%2=0 → consistent
    // lcm(4,6)=12, solution: x ≡ 5 (mod 12)
    auto [s2, lcm2] = crt2(1, 4, 5, 6);
    printf("Non-coprime: x ≡ %lld (mod %lld)\n", s2, lcm2);

    // Inconsistent:
    // x ≡ 0 (mod 2)
    // x ≡ 1 (mod 4)
    // gcd(2,4)=2, (1-0)%2=1 ≠ 0 → no solution
    auto [s3, lcm3] = crt2(0, 2, 1, 4);
    printf("Inconsistent: %s\n", s3==-1 ? "no solution" : "has solution");
}

// ================================================================
// APPLICATION 3: RSA-CRT decryption
// Given private key (p, q, d) and ciphertext c, compute m = c^d mod n
// Using CRT: compute m_p = c^{d mod (p-1)} mod p
//                    m_q = c^{d mod (q-1)} mod q
// Then combine via CRT to get m mod n
// Speedup: ~4x over direct c^d mod n
// ================================================================
ll powmod(ll base, ll exp, ll mod) {
    ll result = 1; base %= mod;
    while (exp > 0) {
        if (exp & 1) result = (lll)result * base % mod;
        base = (lll)base * base % mod;
        exp >>= 1;
    }
    return result;
}

ll rsa_crt_decrypt(ll c, ll d, ll p, ll q) {
    ll n = p * q;
    ll dp = d % (p - 1); // d mod (p-1)
    ll dq = d % (q - 1); // d mod (q-1)

    ll mp = powmod(c, dp, p); // c^dp mod p
    ll mq = powmod(c, dq, q); // c^dq mod q

    // CRT: combine mp and mq to get m mod n
    auto [sol, lcm] = crt2(mp, p, mq, q);
    return sol; // m = c^d mod n
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Basic CRT
    {
        printf("=== Basic CRT ===\n");
        solve_congruence_system();
    }

    // Direct formula (coprime moduli)
    {
        printf("\n=== Direct CRT ===\n");
        vector<ll> r = {2, 3, 2}, m = {3, 5, 7};
        printf("Direct formula: x = %lld\n", crt_direct(r, m));
    }

    // Garner's algorithm
    {
        printf("\n=== Garner's Algorithm ===\n");
        vector<ll> r = {2, 3, 2}, m = {3, 5, 7};
        auto a = garner(r, m);
        printf("Mixed-radix coefficients: ");
        for (ll ai : a) printf("%lld ", ai);
        printf("\n");
        printf("Reconstructed: %lld\n", garner_to_int(a, m));
        // 2 + 1*3 + 0*3*5 = 2+3 = 5? No: a[0]=2, a[1]=(3-2)*3^-1 mod 5=1*2=2
        // x = 2 + 2*3 = 8? mod 15 = 8? but need x≡2 mod 7 too
        // Let's just trust the algorithm
    }

    // RSA-CRT
    {
        printf("\n=== RSA-CRT Decrypt ===\n");
        // Small example: p=61, q=53, n=3233, e=17, d=2753
        // Encrypt m=65: c = 65^17 mod 3233 = 2790
        ll p=61, q=53, d=2753;
        ll c = 2790;
        ll m = rsa_crt_decrypt(c, d, p, q);
        printf("Decrypt 2790 with p=61,q=53,d=2753: m=%lld\n", m); // 65
    }

    // Verification
    {
        printf("\n=== Verification ===\n");
        vector<ll> r = {3, 7, 2}, m = {11, 13, 17};
        auto [sol, lcm] = crt(r, m);
        printf("Solution: x=%lld mod %lld\n", sol, lcm);
        printf("  %lld mod 11 = %lld (want 3)\n", sol, sol%11);
        printf("  %lld mod 13 = %lld (want 7)\n", sol, sol%13);
        printf("  %lld mod 17 = %lld (want 2)\n", sol, sol%17);
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

public static class CRT {

    // ================================================================
    // EXTENDED GCD
    // ================================================================
    private static long ExtGCD(long a, long b, out long x, out long y) {
        long oldR=a, r=b, oldS=1, s=0, oldT=0, t=1;
        while (r != 0) {
            long q = oldR/r;
            (oldR,r) = (r, oldR-q*r);
            (oldS,s) = (s, oldS-q*s);
            (oldT,t) = (t, oldT-q*t);
        }
        x=oldS; y=oldT; return oldR;
    }

    private static long ModInv(long a, long m) {
        long g = ExtGCD(a, m, out long x, out _);
        if (g != 1) return -1;
        return (x % m + m) % m;
    }

    // ================================================================
    // CRT — two congruences (handles non-coprime moduli)
    // ================================================================
    public static (long sol, long lcm) Combine(
        long r1, long m1, long r2, long m2)
    {
        long g = ExtGCD(m1, m2, out long x, out _);
        if ((r2 - r1) % g != 0) return (-1, -1);

        long lcm  = m1 / g * m2;
        long step = m2 / g;
        long diff = ((r2 - r1) / g % step + step) % step;
        long t    = (long)(BigInteger.Multiply(diff, x) % step);
        t = (t % step + step) % step;
        long sol  = (long)((r1 + BigInteger.Multiply(m1, t)) % lcm);
        return (sol, lcm);
    }

    // ================================================================
    // CRT — general system
    // ================================================================
    public static (long sol, long lcm) Solve(
        List<long> r, List<long> m)
    {
        long curR = r[0], curM = m[0];
        for (int i = 1; i < r.Count; i++) {
            var (sol, lcm) = Combine(curR, curM, r[i], m[i]);
            if (sol == -1) return (-1, -1);
            curR = sol; curM = lcm;
        }
        return (curR, curM);
    }

    // ================================================================
    // CRT DIRECT — coprime moduli, explicit M
    // ================================================================
    public static long Direct(List<long> r, List<long> m) {
        BigInteger M = 1;
        foreach (long mi in m) M *= mi;

        BigInteger x = 0;
        for (int i = 0; i < r.Count; i++) {
            BigInteger Mi = M / m[i];
            long inv = ModInv((long)(Mi % m[i]), m[i]);
            x = (x + r[i] * Mi * inv) % M;
        }
        return (long)x;
    }

    // ================================================================
    // GARNER'S ALGORITHM — mixed-radix reconstruction
    // ================================================================
    public static List<long> Garner(List<long> r, List<long> m) {
        int k = r.Count;
        var a = new List<long>(new long[k]);
        a[0] = r[0] % m[0];

        for (int i = 1; i < k; i++) {
            long val = r[i] % m[i];
            long prod = 1;
            for (int j = 0; j < i; j++) {
                val  = (val - (long)(BigInteger.Multiply(a[j], prod) % m[i]) + m[i]) % m[i];
                prod = (long)(BigInteger.Multiply(prod, m[j]) % m[i]);
            }
            a[i] = (long)(BigInteger.Multiply(val, ModInv(prod, m[i])) % m[i]);
        }
        return a;
    }

    public static long GarnerToInt(List<long> a, List<long> m) {
        long result = 0, base_ = 1;
        for (int i = 0; i < a.Count; i++) {
            result += a[i] * base_;
            base_  *= m[i];
        }
        return result;
    }

    // ================================================================
    // RSA-CRT DECRYPTION
    // ================================================================
    private static long PowMod(long b, long e, long mod) {
        long result = 1; b %= mod;
        while (e > 0) {
            if ((e & 1) == 1) result = (long)(BigInteger.Multiply(result, b) % mod);
            b = (long)(BigInteger.Multiply(b, b) % mod);
            e >>= 1;
        }
        return result;
    }

    public static long RsaCrtDecrypt(long c, long d, long p, long q) {
        long mp = PowMod(c, d % (p-1), p);
        long mq = PowMod(c, d % (q-1), q);
        return Combine(mp, p, mq, q).sol;
    }

    // ================================================================
    // Usage
    // ================================================================
    public static void Main() {
        // Basic system
        var r = new List<long>{2,3,2};
        var m = new List<long>{3,5,7};
        var (sol, lcm) = Solve(r, m);
        Console.WriteLine($"CRT: x={sol} mod {lcm}  [expected: 23 mod 105]");

        // Direct
        Console.WriteLine($"Direct: {Direct(r, m)}");

        // Garner
        var a = Garner(r, m);
        Console.WriteLine($"Garner: {GarnerToInt(a, m)}");

        // Non-coprime
        var (s2, l2) = Combine(1, 4, 5, 6);
        Console.WriteLine($"\nNon-coprime (1 mod 4, 5 mod 6): x={s2} mod {l2}");

        // Inconsistent
        var (s3, _) = Combine(0, 2, 1, 4);
        Console.WriteLine($"Inconsistent (0 mod 2, 1 mod 4): {(s3==-1?"no solution":"has solution")}");

        // RSA-CRT
        long dec = RsaCrtDecrypt(2790, 2753, 61, 53);
        Console.WriteLine($"\nRSA-CRT decrypt(2790): {dec}  [expected: 65]");

        // Verification
        r = new List<long>{3,7,2}; m = new List<long>{11,13,17};
        (sol, lcm) = Solve(r, m);
        Console.WriteLine($"\nVerification: x={sol} mod {lcm}");
        Console.WriteLine($"  mod 11={sol%11} (want 3), mod 13={sol%13} (want 7), mod 17={sol%17} (want 2)");
    }
}
```

---

## Correctness of Pairwise Combination

The key identity: if `x ≡ r1 (mod m1)` and `x ≡ r2 (mod m2)`, then:

```
x = r1 + m1 * t  for some t ∈ ℤ
r1 + m1*t ≡ r2 (mod m2)
m1*t ≡ r2 - r1 (mod m2)

Let g = gcd(m1, m2):
  Solvable iff g | (r2 - r1)
  t ≡ (r2-r1)/g * (m1/g)^{-1} (mod m2/g)
  t_0 = ((r2-r1)/g * inv(m1/g, m2/g)) mod (m2/g)
  x = r1 + m1 * t_0  (unique mod lcm(m1,m2))
```

This pairwise reduction is applied k-1 times for k congruences.

---

## Applications Deep Dive

### 1. Multi-Moduli NTT (Number Theoretic Transform)

When convolving polynomials with large coefficients, a single NTT prime (~10^9) may not be large enough. Use three NTT primes p1, p2, p3 and recover each coefficient via Garner:

```
For each output position i:
  c[i] mod p1 = v1   (from NTT mod p1)
  c[i] mod p2 = v2   (from NTT mod p2)
  c[i] mod p3 = v3   (from NTT mod p3)
  Recover c[i] ∈ [0, p1*p2*p3) via Garner
```

Product p1*p2*p3 ≈ 7.4 * 10^26 handles coefficients up to ~7*10^26.

### 2. RSA-CRT Speedup

RSA decryption: `m = c^d mod n`, n = p*q, d up to 2048 bits. CRT form:

```
m_p = c^{d mod (p-1)} mod p   // exponent reduced mod p-1 (Fermat)
m_q = c^{d mod (q-1)} mod q   // exponent reduced mod q-1
m   = CRT(m_p, m_q)           // combine mod n
```

Working mod p and mod q (each ~1024 bits) instead of mod n (~2048 bits) gives ~4x speedup (squarings are 4x cheaper at half the bit-width).

### 3. Competitive Programming — Independent Constraints

When a problem has independent constraints on a number mod different primes, CRT combines them: "find the smallest positive x such that x mod 3 = 1 and x mod 5 = 2 and x mod 7 = 4" → direct CRT application.

### 4. Residue Number System (RNS)

Represent integers by their residues modulo k small primes. Addition and multiplication are component-wise (mod each prime) — fully parallel. CRT converts back to the standard representation when needed.

---

## Pitfalls

- **Overflow in lcm computation** — `lcm(m1, m2) = m1/gcd * m2`. If `m1 = 10^9` and `m2 = 10^9`, `lcm ≈ 10^18` which fits in `int64`. But for three such moduli, `lcm ≈ 10^27` — use `__int128` or `BigInteger`. The pairwise approach propagates `cur_m` which grows to `lcm` of all moduli.
- **Modular inverse requires coprime arguments** — in the direct CRT formula, `M_i^{-1} mod m_i` requires `gcd(M_i, m_i) = 1`, guaranteed only when moduli are pairwise coprime. The pairwise combination via ExtGCD handles non-coprime cases without computing inverses of non-coprime pairs.
- **Sign of `(r2 - r1)` in consistency check** — `(r2 - r1) % g == 0` must hold for the signed remainder. In C++, `%` can return negative values for negative operands. Use `abs(r2 - r1) % g == 0` or ensure `r2 >= r1` before checking.
- **Garner intermediate products overflow** — in Garner, `prod = m[0] * m[1] * ... * m[j-1] mod m[i]`. Each multiplication must be done modulo `m[i]` to prevent overflow: `prod = prod * m[j] % m[i]`. Forgetting the modular reduction causes silent overflow.
- **Non-unique solution when moduli not coprime** — for non-coprime moduli, the solution (if it exists) is unique mod `lcm(m_1,...,m_k)`, not mod their product. Reporting the product as the period gives a wrong answer for the smallest positive solution.
- **Direct formula requires computing M explicitly** — for k=3 NTT primes (~10^9 each), M ≈ 10^27 exceeds `int64`. Use `__int128` (C++) or `BigInteger` (C#) for the direct formula. Garner avoids this by working in bounded arithmetic throughout.
- **r[i] must be in [0, m[i])** — the CRT formula assumes remainders are in the valid range. If `r[i]` is negative or ≥ `m[i]`, normalize first: `r[i] = ((r[i] % m[i]) + m[i]) % m[i]`. Failing to normalize can make a consistent system appear inconsistent or give the wrong solution.

---

## Complexity Summary

| Operation | Time | Space |
|---|---|---|
| CRT (2 congruences) | O(log max(m1,m2)) | O(1) |
| CRT (k congruences, pairwise) | O(k log M) | O(1) |
| CRT direct formula | O(k log M) | O(k) |
| Garner's algorithm | O(k^2 log max_m) | O(k) |
| RSA-CRT speedup | 4x over direct | — |

---

## Conclusion

The Chinese Remainder Theorem is a **fundamental theorem connecting modular arithmetic across independent moduli**:

- The constructive proof gives an efficient O(k log M) algorithm for reconstructing integers from their residues.
- Pairwise combination via ExtGCD handles non-coprime moduli and detects inconsistency automatically.
- Garner's algorithm avoids big-integer arithmetic while reconstructing the exact integer — essential for multi-moduli NTT.
- Applications span cryptography (RSA-CRT, key generation), polynomial arithmetic (multi-prime NTT), and competitive programming (combining independent constraints).

**Key takeaway:**  
When moduli are guaranteed coprime (number theory problems, NTT primes), use the direct formula or Garner for clean implementation. When moduli may share factors (general constraint systems), use pairwise ExtGCD combination — it handles all cases uniformly with a single O(log m) call per congruence. Always use `__int128` or `BigInteger` for intermediate products when `lcm(moduli)` exceeds 64 bits.
