# Gaussian Elimination over GF(2) and over a Field

## Origin & Motivation

Gaussian Elimination is the classical algorithm for solving linear systems, attributed to Carl Friedrich Gauss (19th century) though known to Chinese mathematicians much earlier. It reduces a matrix to row echelon form through a sequence of elementary row operations, then solves via back-substitution.

Two contexts dominate in algorithm design:

- **Over a field (ℝ, ℚ, or Z/pZ):** Standard Gaussian elimination for solving `Ax = b`, computing rank, determinant, and matrix inverse. Over a prime field Z/pZ, all divisions are replaced by modular inverses — making the algorithm exact with no floating-point error.

- **Over GF(2):** All arithmetic is modulo 2 (addition = XOR, multiplication = AND). Row operations become XOR operations on bitmasks. This specialization appears in linear algebra over bits: XOR basis construction, solving XOR systems, counting solutions to XOR equations, and finding the XOR span of a set of numbers.

Complexity: **O(n² * m/w)** for GF(2) with bitset optimization (w = 64), **O(n² * m)** over a field, both for an n×m matrix.

---

## Where It Is Used

**Over a field (Z/pZ):**
- Solving systems of linear equations modulo a prime
- Computing matrix rank, determinant, inverse
- Polynomial interpolation (Vandermonde system)
- Linear programming (simplex basis finding)
- Competitive programming: counting solutions, finding any solution

**Over GF(2):**
- XOR basis / linear basis construction
- Solving XOR constraint systems
- Maximum XOR of a subset (XOR basis + greedy)
- Counting subsets with XOR = target
- Linear codes (Hamming, Reed-Muller): encoding / decoding
- LFSR analysis (Berlekamp-Massey connection)
- Competitive programming: XOR reachability, XOR spanning

---

## Part 1 — Gaussian Elimination over Z/pZ

### Algorithm

Given n×m augmented matrix [A|b], reduce to row echelon form:

```
For each column c from 0 to min(n,m)-1:
    Find pivot row r >= current_row where A[r][c] != 0
    If no pivot: column is free (variable is free)
    Swap rows current_row and r
    Divide pivot row by A[current_row][c] (normalize to 1)
    For each other row r2:
        A[r2] -= A[r2][c] * A[current_row]   (eliminate)
    current_row++
```

### Back-substitution

After reduced row echelon form (RREF):
- Variables corresponding to pivot columns are **basic** (uniquely determined)
- Variables corresponding to non-pivot columns are **free** (can be any value)
- System is inconsistent if a row `[0 0 ... 0 | b]` with `b ≠ 0` appears

---

## Part 2 — Gaussian Elimination over GF(2)

### Arithmetic

- Addition = XOR (`a + b = a ^ b`)
- Multiplication = AND (`a * b = a & b`)
- The only non-zero element is 1; 1^{-1} = 1
- No need to normalize pivot rows (pivot is always 1)
- Row elimination = XOR the pivot row into other rows where the pivot column is 1

### XOR Basis (Linear Basis)

The XOR basis of a set of numbers {x_1, ..., x_n} is a minimal set of numbers spanning the same XOR space. Used to:
- Check if a number is in the XOR span of a set
- Find the maximum XOR achievable by XOR-ing any subset

```
Insert x into basis:
    For each basis element b (from highest bit):
        x = min(x, x XOR b)
    If x != 0: add x to basis
```

### Solving XOR System

Given matrix A over GF(2) and vector b, solve Ax = b (XOR of selected rows = b).

---

## Complexity Analysis

| Operation | Time (naive) | Time (GF(2) bitset) | Space |
|---|---|---|---|
| Row echelon form | O(n² * m) | O(n² * m/64) | O(n * m) |
| Full RREF | O(n² * m) | O(n² * m/64) | O(n * m) |
| Determinant | O(n³) | O(n³/64) | O(n²) |
| Rank | O(n * min(n,m)) | O(n * m/64) | O(n * m) |
| XOR basis insert | O(k) | O(k/64) | O(k) |
| XOR basis max | O(k) | O(k) | O(k) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

const ll MOD = 998244353;

ll pw(ll a, ll b, ll m = MOD) {
    ll r = 1; a %= m;
    for (; b; b >>= 1, a = a*a%m) if (b&1) r = r*a%m;
    return r;
}
ll inv(ll a) { return pw(a, MOD - 2); }

// ================================================================
// GAUSSIAN ELIMINATION OVER Z/pZ
// ================================================================

struct GaussField {
    int n, m;           // n rows, m columns (augmented: m = vars + 1)
    vector<vector<ll>> A;
    vector<int> pivot_col; // pivot_col[row] = column of pivot in that row
    int rank_;

    GaussField(int n, int m) : n(n), m(m), A(n, vector<ll>(m, 0)),
        rank_(0) {}

    void set(int i, int j, ll val) { A[i][j] = (val % MOD + MOD) % MOD; }

    // Reduce to RREF. Returns rank.
    int solve() {
        int row = 0;
        pivot_col.assign(n, -1);
        vector<bool> is_pivot(m, false);

        for (int col = 0; col < m - 1 && row < n; col++) {
            // Find pivot
            int pivot = -1;
            for (int r = row; r < n; r++) {
                if (A[r][col]) { pivot = r; break; }
            }
            if (pivot == -1) continue; // free variable

            swap(A[row], A[pivot]);
            pivot_col[row] = col;
            is_pivot[col] = true;

            // Normalize pivot row
            ll inv_p = inv(A[row][col]);
            for (int j = col; j < m; j++)
                A[row][j] = A[row][j] * inv_p % MOD;

            // Eliminate all other rows
            for (int r = 0; r < n; r++) {
                if (r == row || !A[r][col]) continue;
                ll factor = A[r][col];
                for (int j = col; j < m; j++)
                    A[r][j] = (A[r][j] - factor * A[row][j] % MOD + MOD) % MOD;
            }
            row++;
        }
        rank_ = row;
        return rank_;
    }

    // After solve(), extract solution.
    // Returns:
    //  -1 = no solution (inconsistent)
    //   0 = unique solution
    //   k = infinitely many (k free variables)
    // x[i] = particular solution for basic vars, 0 for free vars.
    int extract(vector<ll>& x) {
        x.assign(m - 1, 0);

        // Check consistency
        for (int r = rank_; r < n; r++)
            if (A[r][m - 1]) return -1; // [0 ... 0 | b] with b != 0

        // Determine free variables
        set<int> pivot_set;
        for (int r = 0; r < rank_; r++)
            if (pivot_col[r] != -1) pivot_set.insert(pivot_col[r]);

        int free_vars = (m - 1) - (int)pivot_set.size();

        // Read basic variables from RREF (free vars = 0)
        for (int r = 0; r < rank_; r++) {
            int c = pivot_col[r];
            if (c == -1) continue;
            x[c] = A[r][m - 1]; // RHS value (free vars contribute 0)
        }

        return free_vars;
    }

    // Determinant of the (n x n) submatrix A[0..n-1][0..n-1]
    // Assumes square matrix; modifies A (call before solve or on copy)
    ll determinant() {
        assert(n == m);
        ll det = 1;
        auto B = A; // work on copy
        int sgn = 1;
        for (int col = 0; col < n; col++) {
            int pivot = -1;
            for (int r = col; r < n; r++)
                if (B[r][col]) { pivot = r; break; }
            if (pivot == -1) return 0;
            if (pivot != col) { swap(B[col], B[pivot]); sgn = -sgn; }
            det = det * B[col][col] % MOD;
            ll inv_p = inv(B[col][col]);
            for (int r = col + 1; r < n; r++) {
                if (!B[r][col]) continue;
                ll factor = B[r][col] * inv_p % MOD;
                for (int j = col; j < n; j++)
                    B[r][j] = (B[r][j] - factor * B[col][j] % MOD + MOD) % MOD;
            }
        }
        return (det * sgn % MOD + MOD) % MOD;
    }
};

// ================================================================
// GAUSSIAN ELIMINATION OVER GF(2) — bitset version
// Each row stored as a bitset (uint64 array for large systems)
// ================================================================

struct GaussGF2 {
    int n, m; // n rows, m columns
    // Store each row as vector of uint64 (packed bits)
    int words;
    vector<vector<uint64_t>> A;
    vector<int> pivot_row; // pivot_row[col] = row index of pivot, -1 if none
    int rank_;

    GaussGF2(int n, int m)
        : n(n), m(m), words((m + 63) / 64),
          A(n, vector<uint64_t>(((m + 63) / 64), 0)),
          pivot_row(m, -1), rank_(0) {}

    void set(int r, int c, int val) {
        if (val & 1) A[r][c / 64] |=  (1ULL << (c % 64));
        else         A[r][c / 64] &= ~(1ULL << (c % 64));
    }

    int get(int r, int c) const {
        return (A[r][c / 64] >> (c % 64)) & 1;
    }

    void xor_row(int dst, int src) {
        for (int w = 0; w < words; w++)
            A[dst][w] ^= A[src][w];
    }

    bool row_zero(int r) const {
        for (int w = 0; w < words; w++) if (A[r][w]) return false;
        return true;
    }

    // Returns rank
    int solve() {
        int row = 0;
        for (int col = 0; col < m && row < n; col++) {
            // Find pivot
            int pivot = -1;
            for (int r = row; r < n; r++) {
                if (get(r, col)) { pivot = r; break; }
            }
            if (pivot == -1) continue;

            swap(A[row], A[pivot]);
            pivot_row[col] = row;

            // Eliminate: XOR pivot row into every other row that has 1 in col
            for (int r = 0; r < n; r++) {
                if (r != row && get(r, col))
                    xor_row(r, row);
            }
            row++;
        }
        rank_ = row;
        return rank_;
    }

    // Check if vector b (length m) is in the row space (can be expressed
    // as XOR of subset of rows of original A).
    // Augmented system: append b as extra column, check consistency.
    // Or: run solve on augmented matrix.

    // Extract solution x (length m, bits) from RREF augmented matrix
    // Column m is the RHS.
    // Returns: -1=inconsistent, 0=unique, k=k free vars
    int extract(vector<int>& x) {
        x.assign(m - 1, 0);
        for (int r = rank_; r < n; r++)
            if (get(r, m - 1)) return -1;

        set<int> pivots;
        for (int c = 0; c < m - 1; c++)
            if (pivot_row[c] != -1) pivots.insert(c);

        int free_count = (m - 1) - (int)pivots.size();
        for (int c = 0; c < m - 1; c++) {
            int r = pivot_row[c];
            if (r == -1) continue;
            x[c] = get(r, m - 1);
        }
        return free_count;
    }
};

// ================================================================
// XOR BASIS (Linear Basis over GF(2))
// Supports: insert, max XOR, check membership, count XOR-able values
// ================================================================
struct XorBasis {
    static const int BITS = 60; // max bit length
    ll basis[BITS + 1];
    int sz;

    XorBasis() : sz(0) { memset(basis, 0, sizeof(basis)); }

    // Insert x into basis. Returns true if x is linearly independent
    // (i.e., not already in the span of current basis).
    bool insert(ll x) {
        for (int i = BITS; i >= 0; i--) {
            if (!((x >> i) & 1)) continue;
            if (!basis[i]) {
                basis[i] = x;
                sz++;
                return true;
            }
            x ^= basis[i];
        }
        return false; // x is in span of existing basis
    }

    // Maximum XOR value achievable by XOR-ing any subset
    ll max_xor() const {
        ll res = 0;
        for (int i = BITS; i >= 0; i--)
            res = max(res, res ^ basis[i]);
        return res;
    }

    // Minimum XOR value achievable (> 0 if basis non-empty)
    ll min_xor() const {
        ll res = 0;
        for (int i = 0; i <= BITS; i++)
            if (basis[i]) { res = basis[i]; break; }
        return res;
    }

    // Check if x is in the XOR span
    bool contains(ll x) const {
        for (int i = BITS; i >= 0; i--) {
            if (!((x >> i) & 1)) continue;
            if (!basis[i]) return false;
            x ^= basis[i];
        }
        return true; // x reduced to 0
    }

    // Count of distinct XOR values achievable (= 2^sz)
    ll count() const { return 1LL << sz; }

    // k-th smallest XOR value (0-indexed, over all subset XORs)
    // Requires reduced basis (each basis element has unique highest bit).
    ll kth(ll k) const {
        // Collect non-zero basis elements in sorted order
        vector<ll> b;
        for (int i = 0; i <= BITS; i++)
            if (basis[i]) b.push_back(basis[i]);
        ll res = 0;
        for (int i = 0; i < (int)b.size(); i++)
            if ((k >> i) & 1) res ^= b[i];
        return res;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // --- Gaussian over Z/pZ ---
    {
        printf("=== Gaussian Elimination over Z/pZ ===\n");

        // Solve: x + 2y = 5
        //        3x + 4y = 6
        // Augmented matrix [A|b]:
        GaussField G(2, 3);
        G.set(0, 0, 1); G.set(0, 1, 2); G.set(0, 2, 5);
        G.set(1, 0, 3); G.set(1, 1, 4); G.set(1, 2, 6);
        int rank = G.solve();

        vector<ll> x;
        int status = G.extract(x);
        printf("Rank=%d, status=%d (0=unique,-1=inconsistent,k=free)\n",
               rank, status);
        printf("Solution: x=%lld, y=%lld\n", x[0], x[1]);
        // x + 2y = 5, 3x + 4y = 6 => 3x+6y=15, 3x+4y=6 => 2y=9, y=9/2 mod p
        // y = 9 * inv(2) mod p = 9 * (p+1)/2 mod p
        ll expected_y = 9 * inv(2) % MOD;
        ll expected_x = (5 - 2 * expected_y % MOD + MOD) % MOD;
        printf("Expected: x=%lld, y=%lld\n", expected_x, expected_y);
    }

    {
        printf("\n=== Inconsistent System ===\n");
        // x + y = 1
        // x + y = 2   (inconsistent)
        GaussField G(2, 3);
        G.set(0,0,1); G.set(0,1,1); G.set(0,2,1);
        G.set(1,0,1); G.set(1,1,1); G.set(1,2,2);
        G.solve();
        vector<ll> x;
        int status = G.extract(x);
        printf("Status=%d (expected -1 = inconsistent)\n", status);
    }

    {
        printf("\n=== Free Variables ===\n");
        // x + y + z = 3 (one equation, 3 unknowns -> 2 free vars)
        GaussField G(1, 4);
        G.set(0,0,1); G.set(0,1,1); G.set(0,2,1); G.set(0,3,3);
        G.solve();
        vector<ll> x;
        int status = G.extract(x);
        printf("Status=%d free vars, particular: x=%lld y=%lld z=%lld\n",
               status, x[0], x[1], x[2]);
    }

    // --- Determinant ---
    {
        printf("\n=== Determinant mod p ===\n");
        GaussField G(3, 3);
        // [[1,2,3],[4,5,6],[7,8,10]] det = -3
        ll mat[3][3] = {{1,2,3},{4,5,6},{7,8,10}};
        for (int i=0;i<3;i++) for(int j=0;j<3;j++) G.A[i][j]=mat[i][j];
        G.n=G.m=3;
        printf("det = %lld\n", G.determinant()); // -3 mod p = MOD-3
    }

    // --- GF(2) Gaussian ---
    {
        printf("\n=== Gaussian over GF(2) ===\n");
        // Solve XOR system:
        // x1 XOR x2 = 1
        // x2 XOR x3 = 0
        // x1 XOR x3 = 1
        // Augmented [A|b] with 3 vars + 1 RHS = 4 cols
        GaussGF2 G(3, 4);
        G.set(0,0,1); G.set(0,1,1); G.set(0,2,0); G.set(0,3,1);
        G.set(1,0,0); G.set(1,1,1); G.set(1,2,1); G.set(1,3,0);
        G.set(2,0,1); G.set(2,1,0); G.set(2,2,1); G.set(2,3,1);
        int rank = G.solve();
        printf("Rank=%d\n", rank);
        vector<int> x;
        int status = G.extract(x);
        printf("Status=%d, x=[%d,%d,%d]\n", status, x[0], x[1], x[2]);
        // Verify: x1^x2=1? x2^x3=0? x1^x3=1?
        printf("Check: %d^%d=%d(want 1), %d^%d=%d(want 0), %d^%d=%d(want 1)\n",
            x[0],x[1],x[0]^x[1], x[1],x[2],x[1]^x[2], x[0],x[2],x[0]^x[2]);
    }

    // --- XOR Basis ---
    {
        printf("\n=== XOR Basis ===\n");
        XorBasis xb;
        for (ll v : {3LL, 5LL, 6LL, 7LL, 9LL}) {
            bool added = xb.insert(v);
            printf("Insert %lld: %s\n", v, added ? "added" : "redundant");
        }
        printf("Basis size: %d\n", xb.sz);
        printf("Max XOR: %lld\n", xb.max_xor());
        printf("Count of distinct XORs: %lld\n", xb.count());
        printf("Contains 4: %s\n", xb.contains(4) ? "yes" : "no");
        printf("Contains 15: %s\n", xb.contains(15) ? "yes" : "no");
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

// ================================================================
// GAUSSIAN ELIMINATION OVER Z/pZ
// ================================================================
public class GaussField {
    private const long MOD = 998244353;
    public int N, M;
    public long[,] A;
    private int[] pivotCol;
    public int Rank;

    private long Inv(long a) {
        long r = 1; a %= MOD;
        for (long b = MOD - 2; b > 0; b >>= 1, a = a * a % MOD)
            if ((b & 1) == 1) r = r * a % MOD;
        return r;
    }

    public GaussField(int n, int m) {
        N = n; M = m;
        A = new long[n, m];
        pivotCol = new int[n];
        Array.Fill(pivotCol, -1);
    }

    public void Set(int i, int j, long v) =>
        A[i, j] = ((v % MOD) + MOD) % MOD;

    public int Solve() {
        int row = 0;
        for (int col = 0; col < M - 1 && row < N; col++) {
            int pivot = -1;
            for (int r = row; r < N; r++)
                if (A[r, col] != 0) { pivot = r; break; }
            if (pivot < 0) continue;

            // Swap
            for (int j = 0; j < M; j++)
                (A[row, j], A[pivot, j]) = (A[pivot, j], A[row, j]);

            pivotCol[row] = col;
            long invP = Inv(A[row, col]);
            for (int j = col; j < M; j++) A[row, j] = A[row, j] * invP % MOD;

            for (int r = 0; r < N; r++) {
                if (r == row || A[r, col] == 0) continue;
                long f = A[r, col];
                for (int j = col; j < M; j++)
                    A[r, j] = (A[r, j] - f * A[row, j] % MOD + MOD) % MOD;
            }
            row++;
        }
        return Rank = row;
    }

    // Returns -1=inconsistent, 0=unique, k=free vars count
    public int Extract(out long[] x) {
        x = new long[M - 1];
        for (int r = Rank; r < N; r++)
            if (A[r, M - 1] != 0) return -1;

        var pivotSet = new HashSet<int>();
        for (int r = 0; r < Rank; r++)
            if (pivotCol[r] >= 0) pivotSet.Add(pivotCol[r]);

        for (int r = 0; r < Rank; r++) {
            int c = pivotCol[r];
            if (c >= 0) x[c] = A[r, M - 1];
        }
        return (M - 1) - pivotSet.Count;
    }

    public long Determinant() {
        if (N != M) throw new Exception("Must be square");
        long det = 1, sgn = 1;
        var B = (long[,])A.Clone();
        for (int col = 0; col < N; col++) {
            int pivot = -1;
            for (int r = col; r < N; r++)
                if (B[r, col] != 0) { pivot = r; break; }
            if (pivot < 0) return 0;
            if (pivot != col) {
                for (int j = 0; j < N; j++)
                    (B[col, j], B[pivot, j]) = (B[pivot, j], B[col, j]);
                sgn = -sgn;
            }
            det = det * B[col, col] % MOD;
            long inv = Inv(B[col, col]);
            for (int r = col + 1; r < N; r++) {
                if (B[r, col] == 0) continue;
                long f = B[r, col] * inv % MOD;
                for (int j = col; j < N; j++)
                    B[r, j] = (B[r, j] - f * B[col, j] % MOD + MOD) % MOD;
            }
        }
        return (det * sgn % MOD + MOD) % MOD;
    }
}

// ================================================================
// XOR BASIS over GF(2)
// ================================================================
public class XorBasis {
    private const int BITS = 60;
    private readonly long[] basis = new long[BITS + 1];
    public int Size { get; private set; }

    public bool Insert(long x) {
        for (int i = BITS; i >= 0; i--) {
            if (((x >> i) & 1) == 0) continue;
            if (basis[i] == 0) { basis[i] = x; Size++; return true; }
            x ^= basis[i];
        }
        return false;
    }

    public long MaxXor() {
        long res = 0;
        for (int i = BITS; i >= 0; i--)
            res = Math.Max(res, res ^ basis[i]);
        return res;
    }

    public bool Contains(long x) {
        for (int i = BITS; i >= 0; i--) {
            if (((x >> i) & 1) == 0) continue;
            if (basis[i] == 0) return false;
            x ^= basis[i];
        }
        return true;
    }

    public long Count => 1L << Size;
}

// ================================================================
// GAUSSIAN ELIMINATION OVER GF(2)
// ================================================================
public class GaussGF2 {
    private readonly int n, m, words;
    private readonly ulong[,] A;
    private readonly int[] pivotRow;
    public int Rank { get; private set; }

    public GaussGF2(int n, int m) {
        this.n = n; this.m = m;
        words = (m + 63) / 64;
        A = new ulong[n, words];
        pivotRow = new int[m];
        Array.Fill(pivotRow, -1);
    }

    public void Set(int r, int c, int v) {
        if ((v & 1) == 1) A[r, c/64] |=  (1UL << (c%64));
        else               A[r, c/64] &= ~(1UL << (c%64));
    }

    public int Get(int r, int c) =>
        (int)((A[r, c/64] >> (c%64)) & 1);

    private void XorRow(int dst, int src) {
        for (int w = 0; w < words; w++) A[dst, w] ^= A[src, w];
    }

    public int Solve() {
        int row = 0;
        for (int col = 0; col < m && row < n; col++) {
            int pivot = -1;
            for (int r = row; r < n; r++)
                if (Get(r, col) == 1) { pivot = r; break; }
            if (pivot < 0) continue;

            // Swap rows
            for (int w = 0; w < words; w++)
                (A[row, w], A[pivot, w]) = (A[pivot, w], A[row, w]);

            pivotRow[col] = row;

            for (int r = 0; r < n; r++)
                if (r != row && Get(r, col) == 1) XorRow(r, row);
            row++;
        }
        return Rank = row;
    }

    public int Extract(out int[] x) {
        x = new int[m - 1];
        for (int r = Rank; r < n; r++)
            if (Get(r, m - 1) == 1) return -1;
        var pivots = new HashSet<int>();
        for (int c = 0; c < m - 1; c++)
            if (pivotRow[c] >= 0) { pivots.Add(c); x[c] = Get(pivotRow[c], m-1); }
        return (m - 1) - pivots.Count;
    }
}

public class Program {
    public static void Main() {
        // Gaussian over Z/pZ
        var G = new GaussField(2, 3);
        G.Set(0,0,1); G.Set(0,1,2); G.Set(0,2,5);
        G.Set(1,0,3); G.Set(1,1,4); G.Set(1,2,6);
        G.Solve();
        int st = G.Extract(out var x);
        Console.WriteLine($"Status={st}, x={x[0]}, y={x[1]}");

        // Determinant
        var D = new GaussField(3, 3);
        long[,] mat = {{1,2,3},{4,5,6},{7,8,10}};
        for (int i=0;i<3;i++) for(int j=0;j<3;j++) D.A[i,j]=mat[i,j];
        Console.WriteLine($"det = {D.Determinant()}"); // MOD-3

        // GF(2)
        var G2 = new GaussGF2(3, 4);
        G2.Set(0,0,1);G2.Set(0,1,1);G2.Set(0,3,1);
        G2.Set(1,1,1);G2.Set(1,2,1);G2.Set(1,3,0);
        G2.Set(2,0,1);G2.Set(2,2,1);G2.Set(2,3,1);
        G2.Solve();
        G2.Extract(out var x2);
        Console.WriteLine($"GF2 solution: [{string.Join(",",x2)}]");

        // XOR Basis
        var xb = new XorBasis();
        foreach (long v in new long[]{3,5,6,7,9})
            Console.WriteLine($"Insert {v}: {xb.Insert(v)}");
        Console.WriteLine($"Max XOR={xb.MaxXor()}, Count={xb.Count}");
    }
}
```

---

## GF(2) Reduction vs Field Reduction

| Property | Over Z/pZ | Over GF(2) |
|---|---|---|
| Pivot normalization | Divide row by pivot | Not needed (pivot = 1) |
| Row elimination | Subtract multiple | XOR |
| Division | Modular inverse | Not needed |
| Free variables | Detected by missing pivot | Same |
| Row operations | 3 types | XOR only |
| Storage | 64-bit integers | 1 bit per entry → bitset |
| Speedup via bitset | Not applicable | 64x via `uint64` |

---

## Pitfalls

- **GF(2) bitset word boundary** — when accessing bit `c` in a row stored as `uint64[]`, use `row[c/64]` and bit `c%64`. Off-by-one in the word index silently reads/writes the wrong 64-bit block, corrupting the entire row.
- **Augmented matrix column count** — for a system with `k` variables, the augmented matrix has `k+1` columns. The last column is the RHS. Iterating columns `0..k-1` for pivots but forgetting to stop before column `k` can make the algorithm pivot on the RHS column, producing wrong results.
- **Inconsistency check location** — after RREF, check rows `rank..n-1` for any row of the form `[0 0 ... 0 | b]` with `b ≠ 0`. Checking only during elimination misses inconsistencies discovered late, and checking the wrong column (k vs k-1) produces wrong consistency verdicts.
- **XOR basis insert must reduce greedily** — when inserting `x` into the XOR basis, reduce `x` by XOR-ing with basis elements from the highest bit downward. Trying basis elements in arbitrary order fails to maintain the invariant that each basis element has a unique highest bit, corrupting future insertions.
- **Max XOR greedy** — after building the basis, compute max XOR by iterating from highest bit to lowest and greedily XOR-ing if it increases the result (`res = max(res, res ^ basis[i])`). Iterating from lowest to highest bit gives wrong results because high-bit choices dominate.
- **Determinant sign** — each row swap multiplies the determinant by -1. Track the sign as ±1 and apply at the end: `det = product_of_pivots * sgn`. In modular arithmetic, `-1 ≡ MOD - 1`. Forgetting the sign gives wrong determinants for matrices requiring an odd number of swaps.
- **GF(2) system: pivot elimination must clear ALL rows** — for RREF (not just echelon form), eliminate in ALL other rows (both above and below the current pivot row). Eliminating only downward produces row echelon form, not RREF. Back-substitution still works, but the extract step assumes RREF.

---

## Conclusion

Gaussian Elimination over GF(2) and over Z/pZ are **fundamental linear algebra primitives**:

- **Over Z/pZ:** exact linear system solving, rank and determinant computation with no floating-point error — the standard tool for linear algebra problems in competitive programming.
- **Over GF(2):** the XOR basis is the canonical structure for XOR-reachability, maximum XOR subset, and XOR equation systems — appearing in dozens of competitive programming problem types.
- **Bitset GF(2):** packing 64 GF(2) matrix entries per word gives a 64x speedup, making Gaussian elimination practical for n×m matrices with n, m up to a few thousand.

**Key takeaway:**  
For GF(2), the XOR basis is sufficient for most competitive programming problems (maximum XOR, membership, count). Full GF(2) Gaussian elimination is needed only for solving XOR systems or computing the rank/null space of a GF(2) matrix. Over Z/pZ, always work modulo a prime and replace division with modular inverse — the algorithm is otherwise identical to real Gaussian elimination.
