# The Ultimate DP Archetypes Cheat Sheet
A comprehensive guide to recognizing, defining, and optimizing the 6 most common Dynamic Programming patterns found in technical interviews and competitive programming.

---

## 1. Knapsack DP (Subset Selection)

### Core Concept
You are given a set of items, each with a weight and a value. You must select a subset of items to maximize value or minimize cost without exceeding a resource capacity (weight bound $W$). 



### State Definition & Transitions
* **0/1 Knapsack (Each item can be used at most once):**
  * **State:** `dp[w]` = Maximum value achievable with exactly/at most capacity `w`.
  * **Transition:** Loop through items. For each item with `weight` and `value`, loop **backwards** from capacity $W$ down to `weight`:
    $$dp[w] = \max(dp[w], dp[w - weight] + value)$$
* **Unbounded Knapsack (Each item can be used infinitely many times):**
  * **State:** Same as above.
  * **Transition:** Loop through items. Loop **forwards** from `weight` up to capacity $W$:
    $$dp[w] = \max(dp[w], dp[w - weight] + value)$$

### Space Optimization Trick
Always drop the 2D item dimension (`dp[i][w] -> dp[w]`). Loops moving **backwards** preserve the state of the *previous* item iteration (0/1), while loops moving **forwards** allow a single item to overwrite and compound on its *current* iteration state (Unbounded).

### Classic Patterns & LeetCode Markers
* "Find a subset of elements that sum up to exactly $K$..."
* **Examples:** LeetCode 416 (Partition Equal Subset Sum), LeetCode 322 (Coin Change), LeetCode 2742 (Painting the Walls).

---

## 2. Linear & Sequence DP (1D / 2D Alignment)

### Core Concept
Decisions are made sequentially from left to right along a single string, array, or a pair of sequences. Subproblems look at prefixes or suffixes of the collection.



### State Definition & Transitions
* **Single Sequence (e.g., LIS):**
  * **State:** `dp[i]` = Optimal solution ending exactly at index `i`.
  * **Transition:** Check all preceding indices $j < i$:
    $$dp[i] = \max_{0 \le j < i}(dp[j] + 1) \quad \text{if } nums[i] > nums[j]$$
* **Two Sequences (e.g., LCS / Edit Distance):**
  * **State:** `dp[i][j]` = Optimal solution matching prefix `A[0...i]` with prefix `B[0...j]`.
  * **Transition:** Compare characters `A[i]` and `B[j]`:
    $$dp[i][j] = \begin{cases} dp[i-1][j-1] + 1 & \text{if } A[i] == B[j] \\ \max(dp[i-1][j], dp[i][j-1]) & \text{if } A[i] \neq B[j] \end{cases}$$

### Space Optimization Trick
For two-sequence problems, `dp[i][j]` only ever depends on the current row `i` and the previous row `i-1`. You can optimize space from $O(M \times N)$ to $O(\min(M, N))$ by utilizing a rolling array or two rows (`prev` and `curr`).

### Classic Patterns & LeetCode Markers
* "Find the longest common/increasing subsequence...", "Minimum operations to convert string A to B..."
* **Examples:** LeetCode 300 (Longest Increasing Subsequence), LeetCode 1143 (Longest Common Subsequence), LeetCode 72 (Edit Distance).

---

## 3. Grid & Matrix DP (Coordinate Navigation)

### Core Concept
You are placed on a 2D grid/matrix and must find an optimal path from a starting coordinate (e.g., top-left corner) to a destination coordinate (e.g., bottom-right corner) moving only in restricted directions (e.g., Right and Down).



### State Definition & Transitions
* **State:** `dp[r][c]` = Minimum cost, maximum profit, or total paths to reach cell `(r, c)`.
* **Transition:** Accumulate values from all valid adjacent cell inputs that can transition directly into `(r, c)`. For standard Right/Down movement:
  $$dp[r][c] = \min(dp[r-1][c], dp[r][c-1]) + cost[r][c]$$

### Space Optimization Trick
Instead of keeping the entire 2D grid matrix in memory, you only need to look at the current row and the row immediately above it. You can track this using a single 1D array of size equal to the grid's column count, updating it in place: `dp[c] = min(dp[c], dp[c-1]) + cost[r][c]`.

### Classic Patterns & LeetCode Markers
* "Given an $M \times N$ matrix, find a path from top-left to bottom-right...", "You can only move down or right..."
* **Examples:** LeetCode 62 (Unique Paths), LeetCode 64 (Minimum Path Sum), LeetCode 120 (Triangle).

---

## 4. Interval DP (Subsegment Merging)

### Core Concept
The problem asks you to find the optimal solution for a full range `[i, j]`. The solution for the outer range depends directly on solving all smaller, interior sub-ranges `[i, k]` and `[k+1, j]` first, then merging them.



### State Definition & Transitions
* **State:** `dp[i][j]` = The optimal value for the subarray/substring spanning from index `i` to index `j`.
* **Transition:** Iterate through all possible split points $k$ between $i$ and $j$:
  $$dp[i][j] = \min_{i \le k < j} \left( dp[i][k] + dp[k+1][j] + \text{cost of merging } [i,k] \text{ and } [k+1,j] \right)$$

### Loop Execution Pattern
You cannot compute this with trivial top-down nested index loops because larger intervals depend on smaller intervals. You must loop by **interval length** first:
```csharp
for (int len = 1; len <= n; len++) {
    for (int i = 0; i <= n - len; i++) {
        int j = i + len - 1;
        // Calculate dp[i][j] by looping over split points k
    }
}
```

### Classic Patterns & LeetCode Markers
* "Find the longest palindromic subsequence...", "Game theory where players pick items from either end of an array...", "Merging adjacent blocks together..."
* **Examples:** LeetCode 516 (Longest Palindromic Subsequence), LeetCode 312 (Burst Balloons), LeetCode 1039 (Minimum Score Triangulation of Polygon).

---

## 5. Tree DP (Subtree Aggregation)

### Core Concept
Dynamic Programming executed over a hierarchical tree structure. The solution for a parent node depends on aggregating the optimal solutions calculated independently for all of its children subtrees.

### State Definition & Transitions
* **State:** Typically defined as `dp[node][state]`. Frequently, `state` is binary: `0` (exclude current node) or `1` (include current node).
* **Transition:** Executed via a **Post-Order Depth-First Search (DFS)**. Compute children states first, then pull them up to the parent:
  $$dp[node][1] = node.val + \sum_{child} dp[child][0]$$
  $$dp[node][0] = \sum_{child} \max(dp[child][0], dp[child][1])$$

### Structural Pattern
No explicit tabular array matrix iteration is required. The call stack of the recursive DFS inherently schedules the correct dependency pipeline (bottom-up from leaf nodes up to the root).

### Classic Patterns & LeetCode Markers
* "Given a tree/graph with no cycles...", "You cannot choose adjacent nodes connected by an edge..."
* **Examples:** LeetCode 337 (House Robber III), LeetCode 124 (Binary Tree Maximum Path Sum), LeetCode 834 (Sum of Distances in Tree).

---

## 6. Bitmask DP (Subsets Tracking)

### Core Concept
Used when problem constraints are tiny (typically $N \le 20$). You need to track the exact subset of items or elements that have already been visited, chosen, or processed. An integer's binary representation represents this set (where the $i$-th bit is `1` if visited, `0` if not).

### State Definition & Transitions
* **State:** `dp[mask][i]` = Optimal solution given that the subset of elements represented by the bits in `mask` have been processed, and the current element is `i`.
* **Transition:** To compute states for a `mask`, search for an element $j$ whose bit is currently set to `0` in the mask, and transition to a new state where that bit is flipped to `1`:
  $$\text{if } ((mask \ \& \ (1 \ll j)) == 0):$$
  $$nextMask = mask \ | \ (1 \ll j)$$
  $$dp[nextMask][j] = \min(dp[nextMask][j], dp[mask][i] + cost[i][j])$$

### Bitwise Operations Cheat Sheet
* Check if item $i$ is in mask: `(mask & (1 << i)) != 0`
* Add item $i$ to mask: `nextMask = mask | (1 << i)`
* Total number of states: $2^N$ (hence why $N$ must be small).

### Classic Patterns & LeetCode Markers
* "Find the shortest path visiting every city exactly once (TSP)...", "Match $N$ workers with $N$ tasks optimally...", Constraints explicitly bound by $N \le 18$ or $N \le 20$.
* **Examples:** LeetCode 698 (Partition to K Equal Sum Subsets), LeetCode 847 (Shortest Path Visiting All Nodes), LeetCode 526 (Beautiful Arrangement).
