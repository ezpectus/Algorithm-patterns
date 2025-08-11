# Problem Guide — Longest Substring Without Repeating Characters
Problem
Given a string s, find the length of the longest substring without repeating characters.

Example:

```
Input: s = "abcabcbb"  → Output: 3   (substring "abc")
Input: s = "bbbbb"     → Output: 1
Input: s = "pwwkew"    → Output: 3   (substring "wke")
```

## Constraints:
- 0 <= s.length <= 10^5
- Characters may be ASCII or Unicode.

### Approach
- We use a sliding window with two pointers (left and right) and a hash map to store the last seen index of each character.
- When we encounter a duplicate within the current window, we move the left pointer to the position right after the last occurrence of that character.
- Why This Is Better Than Alternatives
- Compared to brute force (O(n^2)) — we only scan each character once → O(n).
- No extra substring copies — avoids memory overhead.
- Simple and safe — the last-index map prevents moving left backwards incorrectly.
- Scales easily for strings with length up to 10^5.

--- 

## Code Implementations

```csharp

using System;
using System.Collections.Generic;

public class Solution {
    public static int LengthOfLongestSubstring(string s) {
        if (string.IsNullOrEmpty(s)) return 0;
        var lastIndex = new Dictionary<char, int>();
        int left = 0, maxLen = 0;
        for (int right = 0; right < s.Length; right++) {
            char c = s[right];
            if (lastIndex.TryGetValue(c, out int prev) && prev >= left) {
                left = prev + 1;
            }
            lastIndex[c] = right;
            maxLen = Math.Max(maxLen, right - left + 1);
        }
        return maxLen;
    }

    public static void Main() {
        Console.WriteLine(LengthOfLongestSubstring("abcabcbb")); // 3
        Console.WriteLine(LengthOfLongestSubstring("bbbbb"));    // 1
        Console.WriteLine(LengthOfLongestSubstring("pwwkew"));   // 3
    }
}
```



### C++
````cpp

#include <bits/stdc++.h>
using namespace std;

int lengthOfLongestSubstring(const string &s) {
    unordered_map<char, int> lastIndex;
    int left = 0, maxLen = 0;
    for (int right = 0; right < (int)s.size(); ++right) {
        char c = s[right];
        if (lastIndex.count(c) && lastIndex[c] >= left) {
            left = lastIndex[c] + 1;
        }
        lastIndex[c] = right;
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}

int main() {
    cout << lengthOfLongestSubstring("abcabcbb") << "\n"; // 3
    cout << lengthOfLongestSubstring("bbbbb") << "\n";    // 1
    cout << lengthOfLongestSubstring("pwwkew") << "\n";   // 3
}
````


### Java
````java

import java.util.HashMap;
import java.util.Map;

public class Solution {
    public static int lengthOfLongestSubstring(String s) {
        if (s == null || s.length() == 0) return 0;
        Map<Character, Integer> lastIndex = new HashMap<>();
        int left = 0, maxLen = 0;
        for (int right = 0; right < s.length(); right++) {
            char c = s.charAt(right);
            if (lastIndex.containsKey(c) && lastIndex.get(c) >= left) {
                left = lastIndex.get(c) + 1;
            }
            lastIndex.put(c, right);
            maxLen = Math.max(maxLen, right - left + 1);
        }
        return maxLen;
    }

    public static void main(String[] args) {
        System.out.println(lengthOfLongestSubstring("abcabcbb")); // 3
        System.out.println(lengthOfLongestSubstring("bbbbb"));    // 1
        System.out.println(lengthOfLongestSubstring("pwwkew"));   // 3
    }
}
````

### Python
````python

def length_of_longest_substring(s: str) -> int:
    last_index = {}
    left = 0
    max_len = 0
    for right, c in enumerate(s):
        if c in last_index and last_index[c] >= left:
            left = last_index[c] + 1
        last_index[c] = right
        max_len = max(max_len, right - left + 1)
    return max_len

print(length_of_longest_substring("abcabcbb"))  # 3
print(length_of_longest_substring("bbbbb"))     # 1
print(length_of_longest_substring("pwwkew"))    # 3

````

---

## Time & Space Complexity
- Time Complexity: O(n) — each character is processed at most twice.
- Space Complexity: O(min(n, m)) — where m is the character set size.
  
---

## Conclusion
- This sliding-window solution is optimal for the problem "Longest Substring Without Repeating Characters."
- It runs in O(n) time and uses memory proportional to the number of distinct characters. 
- Unlike brute-force solutions, it scales well for large inputs and avoids unnecessary string copying. 
- The two-pointer + hash map pattern is also applicable to many other substring problems, making it a reusable tool for interviews and production code.



---
