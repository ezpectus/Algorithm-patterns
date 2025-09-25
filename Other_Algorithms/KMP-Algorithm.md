# 🧠 KMP Algorithm — Prefix Failure Engine

---

## 📜 Origin & Motivation

The Knuth-Morris-Pratt (KMP) algorithm was introduced in 1977 to solve **pattern matching** efficiently without backtracking.  
Instead of restarting the search when a mismatch occurs, KMP uses a **failure function** (also called prefix table) to skip redundant comparisons.

It was one of the first algorithms to show that **preprocessing the pattern** can lead to linear-time matching, even in worst-case scenarios.

---

## 🧩 Where It’s Used

- **Pattern matching** — find all occurrences of a pattern in a string
- **Text editors** — search and replace
- **DNA sequence alignment**
- **Firewall filtering** — match against known malicious patterns
- **Compiler design** — lexical analysis

---

## 🔁 When to Use KMP vs Other Algorithms

| Task | Use KMP | Use Z-Algorithm | Use Rabin-Karp |
|------|---------|------------------|----------------|
| Find one pattern | ✅ | ✅ | ✅ |
| Avoid backtracking | ✅ | ❌ | ✅ |
| Worst-case guarantee | ✅ | ❌ | ❌ |
| Multiple patterns | ❌ | ❌ | ✅ |
| Real-time streaming | ✅ | ❌ | ✅ |

---

## 🧱 KMP Prefix Table Code (C#)

```csharp
public static int[] BuildPrefixTable(string pattern) {
    int m = pattern.Length;
    int[] pi = new int[m];
    int j = 0;

    for (int i = 1; i < m; i++) {
        while (j > 0 && pattern[i] != pattern[j])
            j = pi[j - 1];

        if (pattern[i] == pattern[j])
            j++;

        pi[i] = j;
    }

    return pi;
}
```


## 🧱 KMP Prefix Table Code (C++)

```cpp
vector<int> buildPrefixTable(const string& pattern) {
    int m = pattern.size();
    vector<int> pi(m);
    int j = 0;

    for (int i = 1; i < m; ++i) {
        while (j > 0 && pattern[i] != pattern[j])
            j = pi[j - 1];

        if (pattern[i] == pattern[j])
            ++j;

        pi[i] = j;
    }

    return pi;
}
```


## 🧩 Problem: Find All Occurrences of a Pattern

### 🔍 Description

Given a string `text` and a pattern `p`, return all starting indices where `p` occurs in `text`.

---

### 💡 Idea

- Precompute prefix table for `p`  
- Scan `text` with two pointers  
- On mismatch, jump using prefix table instead of restarting

---

## 🧱 Implementation (C#)

```csharp
public static List<int> KMPSearch(string text, string pattern) {
    int[] pi = BuildPrefixTable(pattern);
    List<int> result = new();
    int j = 0;

    for (int i = 0; i < text.Length; i++) {
        while (j > 0 && text[i] != pattern[j])
            j = pi[j - 1];

        if (text[i] == pattern[j])
            j++;

        if (j == pattern.Length) {
            result.Add(i - j + 1);
            j = pi[j - 1];
        }
    }

    return result;
}
```

## 🧱 Implementation (C++)
```cpp
vector<int> kmpSearch(const string& text, const string& pattern) {
    vector<int> pi = buildPrefixTable(pattern);
    vector<int> result;
    int j = 0;

    for (int i = 0; i < text.size(); ++i) {
        while (j > 0 && text[i] != pattern[j])
            j = pi[j - 1];

        if (text[i] == pattern[j])
            ++j;

        if (j == pattern.size()) {
            result.push_back(i - j + 1);
            j = pi[j - 1];
        }
    }

    return result;
}
```

## ⏱️ Complexity

- **Time:** O(n + m)  
  - `O(m)` to build the prefix table  
  - `O(n)` to scan the text using the table  
  - No backtracking — every character is processed at most once  
- **Space:** O(m)  
  - Stores prefix table of size `m`  
  - No auxiliary structures or hash maps

KMP guarantees linear time even in worst-case scenarios, unlike naive matching or hash-based methods that may degrade due to collisions or repeated mismatches.

---

## Conclusion

KMP isn’t just a matcher — it’s a **failure-driven prefix engine**.  
You lead through:

- **preprocessing the pattern** — compressing its internal structure into a reusable table  
- **skipping redundant comparisons** — no backtracking, no wasted effort  
- **guaranteed linear time**, even in adversarial inputs

This module doesn’t just find patterns — it **compresses mismatch recovery into prefix space**,  
turning brute-force into **deterministic flow control**.

It’s ready to be reused in:

- **compilers** — for lexical scanning  
- **network filters** — for malicious pattern detection  
- **real-time search** — where latency matters

KMP lives as a **clean, deterministic primitive** in your algorithmic library —  
no hash collisions, no randomness, no brute force.  
Just structure, reuse, and architectural clarity.


---
