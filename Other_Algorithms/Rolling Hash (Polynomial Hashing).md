# 🧠 Rolling Hash (Polynomial Hashing) — Accelerating String Problems

## 📜 Origin & Motivation
String algorithms often need to compare substrings quickly.  
Naively, comparing two substrings of length `k` takes O(k) time.  
When repeated many times (e.g., in substring search, palindrome checks, or duplicate detection), this becomes too slow.

**Rolling Hash** (a.k.a. Polynomial Hashing) solves this by representing substrings as numeric values that can be updated in O(1) time when sliding a window.  
This enables fast substring comparison, duplicate detection, and efficient string algorithms.

---

## 🧩 Where It’s Used
- **Substring search** — Rabin–Karp algorithm  
- **Duplicate substring detection** — longest duplicate substring, plagiarism detection  
- **Palindrome checking** — compare forward and backward hashes  
- **String matching in competitive programming** — fast equality checks  
- **Hashing in suffix arrays / suffix automata** — quick substring identity checks  

---

## 🔁 When to Use Rolling Hash vs Other Algorithms

| Task / Scenario                  | Use Rolling Hash | Use KMP | Use Trie / Automaton |
|----------------------------------|------------------|---------|-----------------------|
| Single pattern search            | ❌               | ✅       | ✅                    |
| Multiple patterns search         | ❌               | ❌       | ✅ (Aho–Corasick)     |
| Substring equality checks        | ✅               | ❌       | ❌                    |
| Palindrome / reverse checks      | ✅               | ❌       | ❌                    |
| Duplicate substring detection    | ✅               | ❌       | ❌                    |

---

## 🧱 Core Idea: Polynomial Hash
We map a string `S = s₀s₁...sₙ₋₁` to a number using a polynomial:
```
H(S) = (s₀ * p⁰ + s₁ * p¹ + s₂ * p² + ... + sₙ₋₁ * pⁿ⁻¹) mod M
```

- `p` = base (commonly a prime near alphabet size, e.g., 31 or 131)  
- `M` = large prime modulus (e.g., 1e9+7, 1e9+9, or 2^64 with overflow)  

**Rolling property:**  
If we know the hash of prefix `S[0..i]`, we can compute substring hashes in O(1) using precomputed powers of `p`.

---

## 🚀 Implementation (C#)

```csharp
class RollingHash {
    private readonly long[] prefix;
    private readonly long[] power;
    private readonly long mod;
    private readonly long baseP;

    public RollingHash(string s, long baseP = 131, long mod = 1000000007) {
        int n = s.Length;
        this.mod = mod;
        this.baseP = baseP;
        prefix = new long[n + 1];
        power = new long[n + 1];
        power[0] = 1;

        for (int i = 0; i < n; i++) {
            prefix[i + 1] = (prefix[i] * baseP + s[i]) % mod;
            power[i + 1] = (power[i] * baseP) % mod;
        }
    }

    // Get hash of substring [l, r) (0-indexed, exclusive of r)
    public long GetHash(int l, int r) {
        long hash = (prefix[r] - (prefix[l] * power[r - l]) % mod + mod) % mod;
        return hash;
    }
}
```

## 🚀 Implementation (C++)
```cpp
struct RollingHash {
    vector<long long> prefix, power;
    long long baseP, mod;

    RollingHash(const string& s, long long baseP = 131, long long mod = 1000000007) {
        int n = s.size();
        this->baseP = baseP;
        this->mod = mod;
        prefix.assign(n + 1, 0);
        power.assign(n + 1, 1);

        for (int i = 0; i < n; i++) {
            prefix[i + 1] = (prefix[i] * baseP + s[i]) % mod;
            power[i + 1] = (power[i] * baseP) % mod;
        }
    }

    long long getHash(int l, int r) {
        long long hash = (prefix[r] - (prefix[l] * power[r - l]) % mod + mod) % mod;
        return hash;
    }
};
```

## ⏱️ Complexity Analysis

- **Preprocessing:** O(n)  
  - Compute prefix hashes for all prefixes of the string.  
  - Precompute powers of the base `p` modulo `M`.  
  - Done once, reused for all queries.

- **Substring hash query:** O(1)  
  - Any substring `[l, r)` can be extracted in constant time using:  
    ```
    hash(l, r) = (prefix[r] - prefix[l] * power[r - l]) mod M
    ```
  - This makes repeated substring comparisons extremely fast.

- **Space:** O(n)  
  - Store two arrays of length `n+1`:  
    - `prefix[i]` = hash of prefix `s[0..i-1]`  
    - `power[i]` = p^i mod M  

---

## ⚠️ Pitfalls

- **Collisions:**  
  - Different substrings may hash to the same value.  
  - **Mitigation:** use *double hashing* (two different bases and/or moduli).  

- **Choice of base and modulus:**  
  - Base `p` should be larger than the alphabet size (e.g., 31 for lowercase letters, 131 for general ASCII).  
  - Modulus `M` should be a large prime (e.g., 1e9+7, 1e9+9) or rely on 64‑bit unsigned overflow.  

- **Leading zeros:**  
  - Strings of different lengths can sometimes hash to the same value if not careful.  
  - Always compare substring lengths alongside hashes.  

- **Overflow issues:**  
  - In languages without automatic big integer handling, ensure multiplication is done in 64‑bit space.  

- **Hash normalization:**  
  - Always apply `(value + M) % M` to avoid negative results after subtraction.  

---

## ✅ Conclusion

Rolling Hash (Polynomial Hashing) is a **powerful tool** for string problems where fast substring comparison is needed.

It transforms strings into numeric fingerprints that:

- ⚡ Allow O(1) substring equality checks  
- 📚 Enable efficient algorithms like Rabin–Karp  
- 🛡️ Can be made robust with double hashing  
- 🌐 Are widely used in competitive programming and real systems  

👉 **Key takeaway:** Precompute once in O(n), compare substrings in O(1).  
This pattern is the backbone of many advanced string algorithms (e.g., suffix array LCP checks, palindrome detection, duplicate substring search).


---
