# 🧠 Z-Algorithm — Compressed Prefix Matching

---

## 📜 Origin & Motivation

The Z-Algorithm was introduced to solve **pattern matching** problems efficiently.  
Instead of brute-force comparisons, it builds a **Z-array**, where `Z[i]` is the length of the longest prefix of `s` that matches the substring starting at `i`.

It was popularized in competitive programming and string processing libraries due to its **linear time complexity** and **reuse of previous comparisons**.

---

## 🧩 Where It’s Used

- **Pattern matching** — find all occurrences of a pattern in a string
- **Longest common prefix** — between suffixes and the full string
- **String compression** — detect repeated blocks
- **Substring uniqueness** — count distinct substrings
- **Bioinformatics** — DNA sequence alignment

---

## 🔁 When to Use Z vs Other Algorithms

| Task | Use Z-Algorithm | Use Suffix Array | Use KMP |
|------|------------------|------------------|---------|
| Find all pattern matches | ✅ | ✅ (with binary search) | ✅ |
| LCP between suffixes | ✅ | ✅ (with LCP array) | ❌ |
| Count unique substrings | ✅ (with Z) | ✅ | ❌ |
| Preprocessing for fast queries | ✅ | ✅ | ❌ |
| Real-time streaming | ❌ | ❌ | ✅ |

---

## 🧱 Z-Algorithm Code (C#)

```csharp
public static int[] BuildZ(string s) {
    int n = s.Length;
    int[] z = new int[n];
    int l = 0, r = 0;

    for (int i = 1; i < n; i++) {
        if (i <= r)
            z[i] = Math.Min(r - i + 1, z[i - l]);

        while (i + z[i] < n && s[z[i]] == s[i + z[i]])
            z[i]++;

        if (i + z[i] - 1 > r) {
            l = i;
            r = i + z[i] - 1;
        }
    }

    z[0] = n;
    return z;
}
```

## 🧱 Z-Algorithm Code (C++)

```cpp
vector<int> buildZ(const string& s) {
    int n = s.size();
    vector<int> z(n);
    int l = 0, r = 0;

    for (int i = 1; i < n; ++i) {
        if (i <= r)
            z[i] = min(r - i + 1, z[i - l]);

        while (i + z[i] < n && s[z[i]] == s[i + z[i]])
            ++z[i];

        if (i + z[i] - 1 > r) {
            l = i;
            r = i + z[i] - 1;
        }
    }

    z[0] = n;
    return z;
}
```

---
## Example of use:

## 🧩 Problem: 2223. Sum of Scores of Built Strings

## 🔍 Description
- You build a string s by prepending characters one by one. 
- Each intermediate string si is compared to the final string s. Score of si = length of longest common prefix with s.
- Return the sum of all scores.

### 💡 Idea
- Each si is a suffix of s. We need to compute LCP(si, s) for all i. 
 This is exactly what the Z-array gives:

- Z[i] = LCP(s[i:], s)

- Total score = sum(Z)

## 🧱 Implementation (C#)
```csharp
public long SumScores(string s) {
    int[] z = BuildZ(s);
    long total = 0;
    foreach (int val in z)
        total += val;
    return total;
}
```

## 🧱 Implementation (C++)
```cpp
long long sumScores(const string& s) {
    vector<int> z = buildZ(s);
    long long total = 0;
    for (int val : z)
        total += val;
    return total;
}
```


## ⏱️ Complexity

- Time: O(n)
- Space: O(n)

## 🧘Conclusion
The Z-Algorithm isn’t just a tool — it’s a compressed symmetry engine. 
It transforms brute-force prefix matching into a linear-time reusable scan, tracking how each suffix aligns with the full string.

In this problem, you don’t search — you compress and sum. You lead through:

- structural reuse

- minimal mutation

- one clean pass

This module lives not as a trick, but as a core primitive in your algorithmic library. 
It’s ready to be reused, extended, and benchmarked against suffix arrays, rolling hashes, and more.



---
