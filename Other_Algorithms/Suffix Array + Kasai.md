# 🧠 Suffix Array + Kasai — Substring Search & LCP Engine

## 📜 Origin & Motivation
The **Suffix Array** is a sorted array of all suffixes of a string.  
It provides a compact alternative to suffix trees with O(n log n) construction and O(n) space.  

The **Kasai algorithm** builds the **LCP (Longest Common Prefix) array** in O(n), enabling efficient substring queries, repeated substring detection, and string matching.  

Together, they form a must‑have toolkit for advanced string processing.

---

## 🧩 Where It’s Used
- 🔍 **Substring search** in O(m log n) (binary search on suffix array).  
- 📊 **LCP queries** for repeated substrings, longest duplicate substring.  
- 🧬 **Bioinformatics** (DNA/RNA motif search).  
- 📚 **Text indexing** (search engines, compression).  
- 🎮 **Pattern matching** in competitive programming.  

---

## 🔁 When to Use vs Alternatives

| Task / Scenario                  | Suffix Array + Kasai | Suffix Tree | Rolling Hash |
|----------------------------------|----------------------|-------------|--------------|
| Substring search                 | ✅                   | ✅          | ✅ (probabilistic) |
| LCP queries                      | ✅                   | ✅          | ❌            |
| Memory efficiency                | ✅                   | ❌          | ✅            |
| Online queries (dynamic updates) | ❌                   | ❌          | ✅            |
| Simplicity of implementation     | ✅                   | ❌          | ✅            |

---

## 🧱 Core Idea
1. Build suffix array: sort all suffixes of string `s`.  
2. Build LCP array with Kasai:  
   - LCP[i] = length of longest common prefix between suffixes SA[i] and SA[i-1].  
   - Computed in O(n) using rank array + sliding window.  
3. Use cases:  
   - Substring existence → binary search in SA.  
   - Longest repeated substring → max(LCP).  
   - Distinct substrings count → n(n+1)/2 − ΣLCP.  

---

## 🚀 Implementation (C++)

```cpp
vector<int> buildSA(const string& s) {
    int n = s.size(), N = max(256, n) + 1;
    vector<int> sa(n), rnk(n), tmp(n);
    for (int i = 0; i < n; i++) { sa[i] = i; rnk[i] = s[i]; }
    for (int k = 1; k < n; k <<= 1) {
        auto cmp = [&](int i, int j) {
            if (rnk[i] != rnk[j]) return rnk[i] < rnk[j];
            int ri = i + k < n ? rnk[i+k] : -1;
            int rj = j + k < n ? rnk[j+k] : -1;
            return ri < rj;
        };
        sort(sa.begin(), sa.end(), cmp);
        tmp[sa[0]] = 0;
        for (int i = 1; i < n; i++)
            tmp[sa[i]] = tmp[sa[i-1]] + cmp(sa[i-1], sa[i]);
        rnk = tmp;
    }
    return sa;
}

vector<int> buildLCP(const string& s, const vector<int>& sa) {
    int n = s.size();
    vector<int> rnk(n), lcp(n);
    for (int i = 0; i < n; i++) rnk[sa[i]] = i;
    int h = 0;
    for (int i = 0; i < n; i++) {
        if (rnk[i] > 0) {
            int j = sa[rnk[i]-1];
            while (i+h<n && j+h<n && s[i+h]==s[j+h]) h++;
            lcp[rnk[i]] = h;
            if (h > 0) h--;
        }
    }
    return lcp;
}
```

## 🚀 Implementation (C#)
```cpp
public static class SuffixArray {
    public static int[] BuildSA(string s) {
        int n = s.Length;
        int[] sa = new int[n], rnk = new int[n], tmp = new int[n];
        for (int i = 0; i < n; i++) { sa[i] = i; rnk[i] = s[i]; }

        for (int k = 1; k < n; k <<= 1) {
            Array.Sort(sa, (i, j) => {
                if (rnk[i] != rnk[j]) return rnk[i].CompareTo(rnk[j]);
                int ri = i + k < n ? rnk[i+k] : -1;
                int rj = j + k < n ? rnk[j+k] : -1;
                return ri.CompareTo(rj);
            });
            tmp[sa[0]] = 0;
            for (int i = 1; i < n; i++)
                tmp[sa[i]] = tmp[sa[i-1]] + 
                             ((rnk[sa[i-1]] != rnk[sa[i]] || 
                               (sa[i-1]+k<n? rnk[sa[i-1]+k]:-1) != (sa[i]+k<n? rnk[sa[i]+k]:-1)) ? 1 : 0);
            Array.Copy(tmp, rnk, n);
        }
        return sa;
    }

    public static int[] BuildLCP(string s, int[] sa) {
        int n = s.Length;
        int[] rnk = new int[n], lcp = new int[n];
        for (int i = 0; i < n; i++) rnk[sa[i]] = i;
        int h = 0;
        for (int i = 0; i < n; i++) {
            if (rnk[i] > 0) {
                int j = sa[rnk[i]-1];
                while (i+h<n && j+h<n && s[i+h]==s[j+h]) h++;
                lcp[rnk[i]] = h;
                if (h > 0) h--;
            }
        }
        return lcp;
    }
}
```

## ⏱️ Complexity Analysis

- **Suffix Array construction:** O(n log n)  
  - Using the doubling + sort method.  
  - Each iteration doubles the compared prefix length.  
  - Requires O(log n) iterations, each with O(n log n) sorting.  
  - More advanced algorithms (e.g., SA‑IS, DC3) can achieve O(n), but are harder to implement.  

- **Kasai LCP construction:** O(n)  
  - Builds the LCP array in linear time using the rank array.  
  - Maintains a sliding window of the current LCP length, reducing redundant comparisons.  

- **Substring search:** O(m log n)  
  - Binary search over suffix array positions.  
  - Each comparison may take up to O(m) in the worst case.  
  - Optimizations: use LCP information to reduce comparisons.  

- **Space:** O(n)  
  - Arrays: suffix array (SA), rank, LCP, and temporary buffers.  
  - Memory footprint is smaller than suffix trees (which require O(n log n) or more).  

---

## ⚠️ Pitfalls

- **Suffix array building:**  
  - Naive suffix sorting is O(n² log n).  
  - Always use doubling + sort or linear‑time algorithms.  

- **LCP indexing:**  
  - LCP[i] = LCP between suffixes SA[i] and SA[i‑1].  
  - Common mistake: assuming it’s between SA[i] and SA[i+1].  

- **Substring search:**  
  - Binary search requires comparing substrings character by character.  
  - Worst‑case O(m log n).  
  - Use LCP intervals or enhanced structures (e.g., RMQ on LCP) to speed up.  

- **Memory usage:**  
  - O(n) arrays are fine for moderate strings.  
  - For very large strings (e.g., >10⁷ characters), suffix automaton or compressed suffix structures may be more practical.  

- **Alphabet assumptions:**  
  - Ensure stable sorting of suffixes works for the given alphabet (ASCII, Unicode).  
  - For large alphabets, coordinate compression may be needed.  

---

## ✅ Conclusion

Suffix Array + Kasai is a **must‑have engine** for string problems:

- ⚡ Enables O(m log n) substring search.  
- 📊 Provides LCP array in O(n) for repeated substring queries.  
- 🔗 Foundation for suffix‑based algorithms (distinct substrings, longest duplicate substring, suffix comparisons).  
- 🛡️ Deterministic and memory‑efficient compared to suffix trees.  

👉 **Key takeaway:**  
Suffix Array + Kasai is the **go‑to combo** for substring search and LCP queries — elegant, efficient, and indispensable in advanced string processing.  
It balances **simplicity, performance, and memory efficiency**, making it a cornerstone of modern string algorithms.

---
