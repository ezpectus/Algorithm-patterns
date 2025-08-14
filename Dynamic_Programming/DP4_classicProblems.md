# Dynamic Programming Pattern – C# Guide
---
This guide covers four classic problems that can be solved using the dynamic programming technique. 
I will explain each problem, demonstrate the C# solution and analyse the time and space complexity.


---
## 1. Climbing Stairs (Fibonacci-style DP)
**Problem:**
You are climbing a staircase with n steps. 
You can climb 1 or 2 steps at a time. How many distinct ways can you reach the top?

### Approach (DP, Bottom-Up):
- This is a classic Fibonacci-style recurrence:
- Ways to reach step i = ways to reach i-1 + ways to reach i-2

### We build up from base cases:

dp[0] = 1 (1 way to stay at ground)
dp[1] = 1 (1 way to reach first step)

C# Code:
```csharp
public int ClimbStairs(int n) {
    if (n <= 1) return 1;
    int[] dp = new int[n + 1];
    dp[0] = dp[1] = 1;

    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }

    return dp[n];
}
```
## Complexity:
- Time: O(n)
- Space: O(n) (can be optimized to O(1) using two variables)

## 2. House Robber (DP with Decision State)
**Problem:**
Given an array of non-negative integers representing the amount of money in each house, you cannot rob two adjacent houses. 
Find the maximum amount you can rob.

### Approach (DP, Bottom-Up):

- At each house i, you decide:
- Rob it → dp[i] = dp[i-2] + nums[i]
- Skip it → dp[i] = dp[i-1]
- We take the max of both options.

C# Code:
```csharp
public int Rob(int[] nums) {
    if (nums.Length == 0) return 0;
    if (nums.Length == 1) return nums[0];

    int[] dp = new int[nums.Length];
    dp[0] = nums[0];
    dp[1] = Math.Max(nums[0], nums[1]);

    for (int i = 2; i < nums.Length; i++) {
        dp[i] = Math.Max(dp[i - 1], dp[i - 2] + nums[i]);
    }

    return dp[nums.Length - 1];
}
```
 ## Complexity:
- Time: O(n)
- Space: O(n) (can be optimized to O(1))

## 3. Longest Common Subsequence (2D DP Table)

**Problem:**
Given two strings text1 and text2, return the length of their longest common subsequence.

### Approach (DP Table):

#### We build a 2D table dp[i][j]:

- If text1[i-1] == text2[j-1] → dp[i][j] = dp[i-1][j-1] + 1
- Else → dp[i][j] = max(dp[i-1][j], dp[i][j-1])

C# Code:
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
## Complexity:
- Time: O(m·n)
- Space: O(m·n)


## 4. 0/1 Knapsack (Classic DP Optimization)
**Problem:**
Given weights and values of n items and a knapsack capacity W, find the maximum value that can be put in the knapsack.

### Approach (DP Table):

#### We build a 2D table dp[i][w]:

- If we include item i: dp[i][w] = dp[i-1][w - weight[i]] + value[i]
- If we exclude item i: dp[i][w] = dp[i-1][w]
- Take the max of both.

C# Code:
```csharp
public int Knapsack(int[] weights, int[] values, int W) {
    int n = weights.Length;
    int[,] dp = new int[n + 1, W + 1];

    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= W; w++) {
            if (weights[i - 1] <= w) {
                dp[i, w] = Math.Max(dp[i - 1, w], dp[i - 1, w - weights[i - 1]] + values[i - 1]);
            } else {
                dp[i, w] = dp[i - 1, w];
            }
        }
    }

    return dp[n, W];
}
```
## Complexity:

- Time: O(n·W)
- Space: O(n·W)

 ## Conclusion
Dynamic Programming is a powerful technique for solving problems with overlapping subproblems and optimal substructure. 
It replaces brute-force recursion with memoization or tabulation, leading to massive performance gains.

 ## Advantages
 
- Time Complexity Reduction: From exponential to polynomial (e.g., Fibonacci from O(2ⁿ) → O(n))
- Memoization or Tabulation: Store intermediate results to avoid recomputation
- Clear State Transition Logic: Each DP solution is built on a recurrence relation
- Scalable and Predictable: Once the recurrence is known, implementation is straightforward

## Limitations

- Requires Problem Decomposition: You must identify the correct subproblem structure
- Space Usage Can Be High: Especially in 2D DP (e.g., LCS, Knapsack)
- Not Always Intuitive: Some problems require clever state definitions

## When to Use DP
**Use it when:**

- The problem has overlapping subproblems
- You can define a recurrence relation
- You want to avoid recomputation
- You need to maximize/minimize something (e.g., value, length, cost)

## Example Use Cases:

- Problem Type	DP Benefit
- Fibonacci / Climbing Stairs	Avoids exponential recursion
- House Robber	Tracks optimal decision per index
- Longest Common Subsequence	Builds solution via 2D table
- 0/1 Knapsack	Maximizes value under constraints

---

## Final Thought
Dynamic Programming is not just a technique — it's a design philosophy. It teaches you to think recursively, cache intelligently, and build solutions layer by layer.
Mastering it means you'll unlock solutions to some of the hardest algorithmic problems — and write code that’s both elegant and efficient.


---
