# 🧠 Manacher’s Algorithm — Palindromic Radius Engine

---

## 📜 Origin & Motivation

Manacher’s Algorithm was introduced by Glenn Manacher in 1975 to solve the problem of finding the **longest palindromic substring** in linear time.  
Traditional approaches (like expanding around centers) take O(n²) time. Manacher compresses this by **reusing symmetry** and tracking palindromic radii.

It’s one of the few algorithms that handles **odd and even length palindromes uniformly**, using a clever transformation of the input string.

---

## 🧩 Where It’s Used

- **Longest palindromic substring** — in O(n) time
- **Count all palindromic substrings**
- **Palindrome detection in real-time**
- **DNA symmetry analysis**
- **Text compression and pattern folding**

---

## 🔁 When to Use Manacher vs Other Algorithms

| Task | Use Manacher | Use Expand Around Center | Use DP |
|------|--------------|---------------------------|--------|
| Longest palindromic substring | ✅ | ❌ (O(n²)) | ❌ |
| Count all palindromes | ✅ | ✅ | ✅ |
| Real-time palindrome detection | ✅ | ❌ | ❌ |
| Works with even/odd lengths | ✅ | ❌ | ✅ |
| Space-efficient | ✅ | ❌ | ❌ |

---

## 🧱 Manacher’s Algorithm Code (C#)

```csharp
public static int[] Manacher(string s) {
    string t = "#" + string.Join("#", s.ToCharArray()) + "#";
    int n = t.Length;
    int[] p = new int[n];
    int c = 0, r = 0;

    for (int i = 0; i < n; i++) {
        int mirror = 2 * c - i;
        if (i < r)
            p[i] = Math.Min(r - i, p[mirror]);

        while (i - p[i] - 1 >= 0 && i + p[i] + 1 < n && t[i - p[i] - 1] == t[i + p[i] + 1])
            p[i]++;

        if (i + p[i] > r) {
            c = i;
            r = i + p[i];
        }
    }

    return p;
}
```

## 🧱 Manacher’s Algorithm Code (C++)
```cpp
vector<int> manacher(const string& s) {
    string t = "#";
    for (char c : s) {
        t += c;
        t += "#";
    }

    int n = t.size();
    vector<int> p(n);
    int c = 0, r = 0;

    for (int i = 0; i < n; ++i) {
        int mirror = 2 * c - i;
        if (i < r)
            p[i] = min(r - i, p[mirror]);

        while (i - p[i] - 1 >= 0 && i + p[i] + 1 < n && t[i - p[i] - 1] == t[i + p[i] + 1])
            p[i]++;

        if (i + p[i] > r) {
            c = i;
            r = i + p[i];
        }
    }

    return p;
}
```

---
## 🧩 Problem: Longest Palindromic Substring
## 🔍 Description
Given a string s, return the longest palindromic substring.

## 💡 Idea

- Transform s into t by inserting # between characters to unify odd/even lengths
- Run Manacher’s algorithm to compute p[i] — radius of palindrome centered at i
- Find the maximum p[i] and reconstruct the original substring

## 🧱 Implementation (C#)
```csharp
public string LongestPalindrome(string s) {
    int[] p = Manacher(s);
    int maxLen = 0, center = 0;

    for (int i = 0; i < p.Length; i++) {
        if (p[i] > maxLen) {
            maxLen = p[i];
            center = i;
        }
    }

    int start = (center - maxLen) / 2;
    return s.Substring(start, maxLen);
}
```

## 🧱 Implementation (C++)
```cpp
string longestPalindrome(const string& s) {
    vector<int> p = manacher(s);
    int maxLen = 0, center = 0;

    for (int i = 0; i < p.size(); ++i) {
        if (p[i] > maxLen) {
            maxLen = p[i];
            center = i;
        }
    }

    int start = (center - maxLen) / 2;
    return s.substr(start, maxLen);
}
```

## ⏱️ Complexity
- Time: O(n) — linear scan with symmetry reuse
- Space: O(n) — for transformed string and radius array

## 🧘Conclusion
Manacher isn’t just a palindrome finder — it’s a radius-based symmetry engine. 

You lead through:

- center expansion reuse
- odd/even unification
- mirror-based optimization
- one-pass scan with no backtracking

This module doesn’t just detect palindromes — it compresses symmetry into radius space, ready to be reused in substring analysis, DNA folding, and real-time pattern detection.




---
