# 🧠 Rabin-Karp — Rolling Hash Pattern Matching

---

## 📜 Origin & Motivation

Rabin-Karp was introduced in 1987 by Michael Rabin and Richard Karp to solve **multiple pattern matching** efficiently.  
Instead of comparing strings character-by-character, it uses **rolling hash values** to compare substrings in constant time.

Its power lies in **hashing**: if two substrings have the same hash, they might match — and if not, they definitely don’t.  
This allows fast filtering before doing actual character comparisons.

---

## 🧩 Where It’s Used

- **Pattern matching** — find all occurrences of a pattern in a string
- **Plagiarism detection** — compare large documents via hashes
- **Substring search** — fast filtering with hash collisions
- **Bioinformatics** — genome sequence scanning
- **Data deduplication** — detect repeated blocks

---

## 🔁 When to Use Rabin-Karp vs Other Algorithms

| Task | Use Rabin-Karp | Use Z-Algorithm | Use Suffix Array |
|------|------------------|------------------|------------------|
| Find one pattern | ✅ | ✅ | ✅ |
| Find multiple patterns | ✅ | ❌ | ✅ |
| Real-time streaming | ✅ | ❌ | ❌ |
| Avoid character comparisons | ✅ | ❌ | ❌ |
| Count unique substrings | ❌ | ✅ | ✅ |
| Detect repeated blocks | ✅ | ✅ | ✅ |

---

## 🧱 Rabin-Karp Code (C#)

```csharp
public static List<int> RabinKarp(string text, string pattern) {
    const int baseVal = 256;
    const int mod = 1000000007;
    int n = text.Length, m = pattern.Length;
    long patternHash = 0, windowHash = 0, power = 1;
    List<int> result = new();

    for (int i = 0; i < m; i++) {
        patternHash = (patternHash * baseVal + pattern[i]) % mod;
        windowHash = (windowHash * baseVal + text[i]) % mod;
        if (i > 0) power = (power * baseVal) % mod;
    }

    if (windowHash == patternHash && text.Substring(0, m) == pattern)
        result.Add(0);

    for (int i = m; i < n; i++) {
        windowHash = (windowHash - text[i - m] * power % mod + mod) % mod;
        windowHash = (windowHash * baseVal + text[i]) % mod;

        if (windowHash == patternHash && text.Substring(i - m + 1, m) == pattern)
            result.Add(i - m + 1);
    }

    return result;
}
```

## 🧱 Rabin-Karp Code (C++)
```cpp
vector<int> rabinKarp(const string& text, const string& pattern) {
    const int base = 256;
    const int mod = 1e9 + 7;
    int n = text.size(), m = pattern.size();
    long long patternHash = 0, windowHash = 0, power = 1;
    vector<int> result;

    for (int i = 0; i < m; ++i) {
        patternHash = (patternHash * base + pattern[i]) % mod;
        windowHash = (windowHash * base + text[i]) % mod;
        if (i > 0) power = (power * base) % mod;
    }

    if (windowHash == patternHash && text.substr(0, m) == pattern)
        result.push_back(0);

    for (int i = m; i < n; ++i) {
        windowHash = (windowHash - text[i - m] * power % mod + mod) % mod;
        windowHash = (windowHash * base + text[i]) % mod;

        if (windowHash == patternHash && text.substr(i - m + 1, m) == pattern)
            result.push_back(i - m + 1);
    }

    return result;
}
```

## 🧩 Problem: Find All Occurrences of a Pattern


## 🔍 Description
Given a string text and a pattern p, return all starting indices where p occurs in text.

## 💡 Idea

- Precompute hash of p
- Slide a window of length p.Length over text
- Compare hash of window with hash of p
- If hashes match, verify with actual substring comparison

## 🧱 Implementation (C#)
```csharp
public IList<int> FindPattern(string text, string pattern) {
    return RabinKarp(text, pattern);
}
```

## 🧱 Implementation (C++)
```cpp
vector<int> findPattern(const string& text, const string& pattern) {
    return rabinKarp(text, pattern);
}
```

## ⏱️ Complexity
- Time: O(n + m) average case — thanks to constant-time hash comparisons
- Worst-case: O(n·m) — only if hash collisions force full substring comparisons
- Space: O(1) — no extra memory beyond a few integers

## 🧘Conclusion
Rabin-Karp isn’t just a hash — it’s a rolling fingerprint engine. 

You lead through:

- fast filtering — skip character-by-character scans unless hashes match
- minimal comparisons — verify only when hashes collide
- modular arithmetic — keep hashes bounded and efficient
- sliding window reuse — update hashes in constant time
- And you do it with one clean scan, no backtracking, no preprocessing.

This module doesn’t just match patterns — it compresses search into hash space, turning brute-force into probabilistic precision.

It’s ready to be:

- reused in substring search, deduplication, and streaming filters
- extended to multiple patterns, 2D grids, or rolling hashes over trees
- benchmarked against Z, KMP, and suffix arrays — each with its own tradeoffs

Rabin-Karp lives as a hash-based primitive in your algorithmic arsenal. 
It’s not just fast — it’s architecturally clean, and ready to scale.


---




