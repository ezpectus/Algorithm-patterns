# Dynamic Programming — Algorithm Template

## Introduction
Dynamic Programming (DP) is a problem-solving technique that breaks a problem into **overlapping subproblems** and solves each subproblem only once, storing the results for reuse.  

The core idea:
> **"Solve small problems → store the answers → use them to solve bigger problems."**

DP often turns an **exponential-time brute force** into a **polynomial-time solution** by avoiding repeated calculations.

---

## When to Apply DP
You should consider DP when:
1. **Optimal substructure** — The solution to the problem can be composed from solutions to its subproblems.
2. **Overlapping subproblems** — The same subproblem appears multiple times.
3. You have to find:
   - Minimum / maximum value (shortest path, max profit, min cost).
   - Count of possible ways (number of paths, combinations).
   - Boolean decision (possible/impossible).

---

## Types of DP

## 1. **1D DP** — Linear state
- The DP array `dp[i]` stores the answer for state `i`.
- State depends on **previous states** (e.g., `dp[i-1]`, `dp[i-2]`).

**Example:** Climbing Stairs — Number of distinct ways to climb `n` steps, 1 or 2 steps at a time.

**Pseudocode:**
dp[0] = 1
dp[1] = 1
for i = 2 to n:
dp[i] = dp[i-1] + dp[i-2]


**C# Implementation:**
```csharp
public int ClimbStairs(int n) {
    if (n <= 2) return n;
    int[] dp = new int[n + 1];
    dp[0] = 1; dp[1] = 1;
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    return dp[n];
}

```


## 2. 2D DP — Matrix state
dp[i][j] describes the solution for a subproblem defined by two parameters.

Common in string problems, grids, and knapsack.

Example: Longest Common Subsequence (LCS)

Pseudocode:

```java

for i = 0..m:
    dp[i][0] = 0
for j = 0..n:
    dp[0][j] = 0

for i = 1..m:
    for j = 1..n:
        if s1[i-1] == s2[j-1]:
            dp[i][j] = dp[i-1][j-1] + 1
        else:
            dp[i][j] = max(dp[i-1][j], dp[i][j-1])

```
C# Implementation:

```csharp

public int LongestCommonSubsequence(string text1, string text2) {
    int m = text1.Length, n = text2.Length;
    int[,] dp = new int[m + 1, n + 1];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1[i - 1] == text2[j - 1]) {
                dp[i, j] = dp[i - 1, j - 1] + 1;
            } else {
                dp[i, j] = Math.Max(dp[i - 1, j], dp[i, j - 1]);
            }
        }
    }
    return dp[m, n];
}
```

## Advanced DP

## 1. Space Optimization (Rolling Array)
When dp[i] depends only on dp[i-1], we can store just two rows instead of the full table.

Example: Optimized Fibonacci

```csharp

public int Fib(int n) {
    if (n < 2) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        int temp = a + b;
        a = b;
        b = temp;
    }
    return b;
}
```
## 2. Knapsack Problem
0/1 Knapsack: Maximize value within weight limit.

Pseudocode:

```
dp[0..W] = 0
for each item:
    for w = W down to weight[item]:
        dp[w] = max(dp[w], dp[w - weight[item]] + value[item])
```

## C# Implementation:

```csharp

public int Knapsack(int[] weights, int[] values, int W) {
    int[] dp = new int[W + 1];
    for (int i = 0; i < weights.Length; i++) {
        for (int w = W; w >= weights[i]; w--) {
            dp[w] = Math.Max(dp[w], dp[w - weights[i]] + values[i]);
        }
    }
    return dp[W];
}

```

## 3. Bitmask DP
Useful for subset problems and TSP.
Used for problems involving subsets of elements.

Example: Traveling Salesman Problem (TSP)

State:
dp[mask][i] = min cost to visit all cities in mask ending at city i.
```csharp

public int TSP(int[,] dist) {
    int n = dist.GetLength(0);
    int maxMask = 1 << n;
    int[,] dp = new int[maxMask, n];
    const int INF = 1_000_000_000;

    for (int mask = 0; mask < maxMask; mask++)
        for (int i = 0; i < n; i++)
            dp[mask, i] = INF;

    dp[1, 0] = 0; // start from city 0

    for (int mask = 1; mask < maxMask; mask++) {
        for (int u = 0; u < n; u++) {
            if ((mask & (1 << u)) == 0) continue;
            for (int v = 0; v < n; v++) {
                if ((mask & (1 << v)) != 0) continue;
                int nextMask = mask | (1 << v);
                dp[nextMask, v] = Math.Min(dp[nextMask, v], dp[mask, u] + dist[u, v]);
            }
        }
    }

    int ans = INF;
    for (int i = 1; i < n; i++) {
        ans = Math.Min(ans, dp[maxMask - 1, i] + dist[i, 0]);
    }
    return ans;
}
```
## Complexity:
- Time — O(2^n * n^2)
- Space — O(2^n * n)

## 4. DP on Trees
- Solve problems on trees using DFS + memoization.

Example: Maximum independent set in a tree.

```csharp
public class TreeDP {
    List<int>[] adj;
    int[,] dp;
    bool[] visited;

    public int MaxIndependentSet(List<int>[] graph) {
        adj = graph;
        int n = graph.Length;
        dp = new int[n, 2];
        visited = new bool[n];
        DFS(0, -1);
        return Math.Max(dp[0,0], dp[0,1]);
    }

    void DFS(int u, int parent) {
        dp[u,0] = 0;
        dp[u,1] = 1;
        foreach (var v in adj[u]) {
            if (v == parent) continue;
            DFS(v, u);
            dp[u,0] += Math.Max(dp[v,0], dp[v,1]);
            dp[u,1] += dp[v,0];
        }
    }
}
```
## Complexity:
- Time — O(n)
- Space — O(n)


## 5. Digit DP
- Used for counting numbers with constraints on digits.

Example: Count numbers ≤ N with no consecutive equal digits.

```csharp
public class DigitDP {
    string num;
    int[,,] memo;

    public int Count(int N) {
        num = N.ToString();
        memo = new int[num.Length + 1, 11, 2];
        for (int i = 0; i < memo.GetLength(0); i++)
            for (int j = 0; j < memo.GetLength(1); j++)
                for (int k = 0; k < memo.GetLength(2); k++)
                    memo[i,j,k] = -1;
        return DFS(0, 10, true);
    }

    int DFS(int pos, int prevDigit, bool tight) {
        if (pos == num.Length) return 1;
        if (memo[pos, prevDigit, tight ? 1 : 0] != -1)
            return memo[pos, prevDigit, tight ? 1 : 0];

        int limit = tight ? num[pos] - '0' : 9;
        int res = 0;
        for (int d = 0; d <= limit; d++) {
            if (d != prevDigit) {
                res += DFS(pos + 1, d, tight && (d == limit));
            }
        }
        return memo[pos, prevDigit, tight ? 1 : 0] = res;
    }
}
```
## Complexity:
- Time — O(len * 10 * 2)
- Space — O(len * 11 * 2)


### Complexity

- 1D DP: O(n) time, O(n) or O(1) space.
- 2D DP: O(n*m) time, O(n*m) space.
- Bitmask DP: O(2^n * n) time, O(2^n * n) space.

### Common Mistakes

- Wrong base case initialization — DP will give wrong answers if base states are not correct.
- Incorrect loop order — In 1D knapsack, reverse loop is crucial to avoid reusing an item multiple times.
- Off-by-one errors — Pay attention to array bounds.
- State misunderstanding — Clearly define what dp[i] or dp[i][j] means.
- Forgetting to mod results when working with large counts.

### Memorization Tips

- Always define the state in words before coding.
- Draw the recursion tree to find overlapping subproblems.
- Fill a small example table manually before implementing.
- If memory is too high, check if rolling array optimization is possible.
- Many problems are “current answer = best of previous answers + current choice”.



## Conclusion:
Dynamic Programming is not about memorizing patterns — it’s about learning to think in states and transitions.
Always start by defining exactly what your DP array represents, set correct base cases, and write the recurrence relation clearly. 
Work through small examples by hand to verify your logic before coding. 
Train by solving problems of increasing difficulty, moving from simple 1D cases to 2D tables, knapsack variants, and advanced techniques like bitmask, 
digit DP, and tree DP. With enough practice, you’ll start seeing DP patterns everywhere, 
and turning seemingly impossible exponential problems into elegant, efficient solutions will become second nature.







---
