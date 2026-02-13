# Burrows–Wheeler Transform — Permutation for Compression

## Origin & Motivation
The Burrows–Wheeler Transform (BWT) was introduced by Michael Burrows and David Wheeler in 1994.
It is a reversible permutation (rearrangement) of the characters in a string that does not compress the data itself — but dramatically improves compressibility for subsequent stages (especially run-length encoding + entropy coding).
Key insight: many real-world texts contain long runs of identical or similar characters when viewed in certain contexts. BWT tends to group identical letters together into long runs,
making them easy targets for simple compressors like Move-To-Front + Huffman or arithmetic coding.
BWT is the core transformation behind bzip2 (one of the best general-purpose compressors of the late 1990s–2000s) and is widely used in bioinformatics (FM-index, Bowtie, BWA aligners).

## Core Idea
The Transformation Steps

- Append a unique sentinel (usually $, lexicographically smallest character)
- Generate all cyclic rotations of the string
- Sort these rotations lexicographically
- Take the last column of the sorted matrix → this is the BWT

The resulting string often contains long runs of identical characters (clustering effect).

## Why it clusters similar characters
- When two suffixes are very similar (long common prefix), their rotations will be close in sorted order.
- The character immediately preceding a long matching suffix often becomes grouped with others preceding similar suffixes → runs appear.

## Inverse BWT (reconstruction)
Given only the BWT string (and knowing the position of $ or using an index), we can recover the original string in O(n) time using:

- Sort the BWT to get the first column (F)
- Build the LF-mapping (Last-to-First)
- Start from the row ending with $ and follow LF links backwards

## Where It’s Used

- bzip2 compression (.bz2 files)
- Bioinformatics read alignment — FM-index (full-text minute index), used in Bowtie, BWA, HISAT2
- Data deduplication and backup systems
- Suffix array construction (BWT → suffix array in O(n) space/time)
- Pattern matching in large texts (counting and locating occurrences efficiently)
- Any place where reversible clustering of symbols improves entropy coding


## Burrows–Wheeler Transform (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

string bwt(const string& s) {
    int n = s.size();
    vector<string> rotations(n);
    string t = s + '$';  // sentinel must be smallest

    for (int i = 0; i < n; ++i) {
        rotations[i] = t.substr(i) + t.substr(0, i);
    }

    sort(rotations.begin(), rotations.end());

    string result;
    for (const auto& rot : rotations) {
        result += rot.back();
    }
    return result;
}

// Naive inverse BWT (O(n²) — for illustration only)
string inverse_bwt_naive(const string& b) {
    int n = b.size();
    vector<string> table(n);

    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < n; ++j) {
            table[j] = b[j] + table[j];
        }
        sort(table.begin(), table.end());
    }

    for (const auto& row : table) {
        if (row.back() == '$') {
            return row.substr(0, n - 1);  // remove sentinel
        }
    }
    return "";
}

// Efficient inverse BWT — O(n)
string inverse_bwt(const string& b) {
    int n = b.size();
    vector<int> next(n);
    vector<pair<char, int>> sorted;

    for (int i = 0; i < n; ++i) {
        sorted.emplace_back(b[i], i);
    }
    sort(sorted.begin(), sorted.end());

    // Build LF mapping: position in sorted list → original position
    for (int i = 0; i < n; ++i) {
        next[sorted[i].second] = i;
    }

    // Find row that ends with $
    int p = -1;
    for (int i = 0; i < n; ++i) {
        if (b[i] == '$') {
            p = i;
            break;
        }
    }

    string result;
    for (int i = 0; i < n - 1; ++i) {  // skip sentinel
        p = next[p];
        result = b[p] + result;
    }
    return result;
}

// === TEST FUNCTION ===
static void runTests() {
    cout << "Burrows-Wheeler Transform — Tests\n\n";

    // Classic example
    {
        string s = "banana";
        string expected_bwt = "annb$aa";
        string got = bwt(s);
        cout << "banana → BWT: " << got << " → "
             << (got == expected_bwt ? "PASS" : "FAIL") << "\n";

        string recovered = inverse_bwt(got);
        cout << "Inverse: " << recovered << " → "
             << (recovered == s ? "PASS" : "FAIL") << "\n\n";
    }

    // Another example
    {
        string s = "abracadabra";
        string got = bwt(s);
        string recovered = inverse_bwt(got);
        cout << "abracadabra → recovered: " << recovered << " → "
             << (recovered == s ? "PASS" : "FAIL") << "\n\n";
    }

    cout << "All tests completed!\n";
}

int main() {
    runTests();
    return 0;
}
```

## Burrows–Wheeler Transform (C#)
```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class BurrowsWheeler
{
    public static string BWT(string s)
    {
        int n = s.Length;
        var rotations = new List<string>();
        string t = s + "$";

        for (int i = 0; i < n; i++)
        {
            rotations.Add(t.Substring(i) + t.Substring(0, i));
        }

        rotations.Sort();
        return string.Concat(rotations.Select(r => r[^1]));
    }

    // Efficient inverse BWT
    public static string InverseBWT(string b)
    {
        int n = b.Length;
        var sorted = b.Select((c, i) => (c, i)).OrderBy(x => x.c).ToArray();
        var next = new int[n];

        for (int i = 0; i < n; i++)
        {
            next[sorted[i].i] = i;
        }

        int p = Array.FindIndex(b.ToCharArray(), c => c == '$');
        var result = new char[n - 1];

        for (int i = n - 2; i >= 0; i--)
        {
            p = next[p];
            result[i] = b[p];
        }

        return new string(result);
    }

    public static void RunTests()
    {
        Console.WriteLine("Burrows-Wheeler Transform — Tests\n");

        string s1 = "banana";
        string b1 = BWT(s1);
        Console.WriteLine($"banana → BWT: {b1} → {(b1 == "annb$aa" ? "PASS" : "FAIL")}");
        Console.WriteLine($"Inverse: {InverseBWT(b1)} → {(InverseBWT(b1) == s1 ? "PASS" : "FAIL")}\n");

        string s2 = "abracadabra";
        string recovered = InverseBWT(BWT(s2));
        Console.WriteLine($"abracadabra → recovered: {recovered} → {(recovered == s2 ? "PASS" : "FAIL")}\n");

        Console.WriteLine("All tests completed!");
    }
}

class Program
{
    static void Main() => BurrowsWheeler.RunTests();
}
```


## Time & Space Complexity
Forward BWT (naive implementation with explicit rotations):

- Time: O(n² log n) — due to sorting n strings of length n
- Space: O(n²) — storing all rotations

## Practical / suffix-array based BWT:

- Time: O(n) (using DC3 or similar suffix array construction)
- Space: O(n)

## Inverse BWT (efficient version):

- Time: O(n log |Σ|) (sorting step) or O(n) with counting sort for small alphabet
- Space: O(n)

-- 
## Key Concepts Recap

- Cyclic rotations → all possible ways to "rotate" the string
- Lexicographic sort → groups similar contexts together
- Last column = BWT → often contains long runs of equal symbols
- First column ≈ sorted version of original characters
- LF-mapping (Last-to-First) → links last column position to first column position of the same character
- Sentinel ($) → ensures uniqueness and marks string end
- Reversible → original string can always be recovered exactly


## ✅ Conclusion
The Burrows–Wheeler Transform is a brilliant, almost magical permutation:

- It does not compress — but turns mediocre data into something extremely compressible by grouping equal symbols into long runs.
- Achieves this with a conceptually simple idea (rotations + sort + last column) yet profound clustering effect.
- Powers bzip2, many modern bioinformatics aligners, and efficient full-text indices (FM-index).
- Shows how a clever, reversible reordering can unlock dramatic gains for subsequent compression or search stages.

BWT is a perfect example of how reversible transformations can sometimes outperform direct compression — a timeless idea in data compression and string processing.



---
