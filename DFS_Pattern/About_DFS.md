# 🧠 Depth-First Search (DFS) — Algorithm Template

## 🔍 Introduction

Depth-First Search (DFS) is a graph traversal technique that explores as far as possible along each branch before backtracking.

**Core idea**:  
> "Go deep first, then backtrack when stuck."

DFS is ideal for exploring **all paths**, **connected components**, **recursive structures**, and **state spaces** where decisions depend on previous steps.

---

## 🧠 When to Apply DFS

Use DFS when:

- ✅ You need to explore all possible paths or configurations
- ✅ The graph/tree has depth-based structure
- ✅ You're solving backtracking, recursion, or component detection
- ✅ You need to track visited states or paths
- ❌ You don’t need shortest path (use BFS instead)
- ❌ The graph isn’t too deep (to avoid stack overflow)

**Common applications**:

- Tree traversal (preorder, inorder, postorder)
- Connected components (islands, regions)
- Backtracking (sudoku, permutations, combinations)
- Cycle detection
- Pathfinding with constraints

---

## 🧱 DFS Template — Recursive

```csharp
void DFS(Node node, HashSet<Node> visited) {
    if (visited.Contains(node)) return;

    visited.Add(node);
    // process node

    foreach (var neighbor in GetNeighbors(node)) {
        DFS(neighbor, visited);
    }
}
```

## 🧱 DFS Template — Iterative (Stack)
```csharp
void DFSIterative(Node start) {
    var stack = new Stack<Node>();
    var visited = new HashSet<Node>();

    stack.Push(start);

    while (stack.Count > 0) {
        var node = stack.Pop();
        if (visited.Contains(node)) continue;

        visited.Add(node);
        // process node

        foreach (var neighbor in GetNeighbors(node)) {
            stack.Push(neighbor);
        }
    }
}
```


## 🧩 DFS Patterns

| Pattern              | Example Problems                          | Key Idea                     |
|----------------------|-------------------------------------------|------------------------------|
| Tree Traversal       | Binary Tree Paths, Subtree Sum            | Recursive depth-first logic |
| Grid DFS             | Number of Islands, Surrounded Regions     | Explore 2D matrix recursively |
| Backtracking         | Permutations, N-Queens, Sudoku            | Try → Recurse → Undo         |
| State Space Search   | Word Ladder II, Sliding Puzzle            | Encode visited states        |
| Cycle Detection      | Directed Graph Cycle Detection            | Track recursion stack        |



## ✅ How to Validate DFS

Before applying DFS, ask:

- Is depth-first exploration meaningful?
- Do I need to explore all paths or just one?
- Can I encode visited states to avoid repetition?
- Is recursion safe (depth not too large)?

## ❌ Common Mistakes:

- ❌ Forgetting to mark nodes as visited → infinite recursion
- ❌ Not handling base cases → stack overflow
- ❌ Re-visiting nodes → exponential time
- ❌ Using DFS when BFS is needed for shortest path

## 🧠 Memorization Tips

DFS = Recursion or Stack + Visited Set:

- Think in terms of: "Go deeper, then backtrack"
- For grids, define directions array
- For backtracking, always undo after recursion
- For cycle detection, track recursion stack

## 🏗️ DFS Training Progression:

- ✅ Tree traversal (binary trees)
- ✅ Grid DFS (islands, regions)
- ✅ Backtracking (combinatorics, puzzles)
- ✅ DFS with state encoding (graph problems)
- ✅ DFS with cycle detection (directed graphs)

## 💡 Full Code Examples (C#)

## 1. Number of Islands (Grid DFS)

```csharp
public int NumIslands(char[][] grid) {
    int m = grid.Length, n = grid[0].Length;
    int count = 0;

    void DFS(int i, int j) {
        if (i < 0 || j < 0 || i >= m || j >= n || grid[i][j] != '1') return;
        grid[i][j] = '0';
        DFS(i+1, j); DFS(i-1, j); DFS(i, j+1); DFS(i, j-1);
    }

    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            if (grid[i][j] == '1') {
                DFS(i, j);
                count++;
            }

    return count;
}
```


## 2. Permutations (Backtracking)

```csharp
public IList<IList<int>> Permute(int[] nums) {
    var res = new List<IList<int>>();
    var used = new bool[nums.Length];

    void DFS(List<int> path) {
        if (path.Count == nums.Length) {
            res.Add(new List<int>(path));
            return;
        }

        for (int i = 0; i < nums.Length; i++) {
            if (used[i]) continue;
            used[i] = true;
            path.Add(nums[i]);
            DFS(path);
            path.RemoveAt(path.Count - 1);
            used[i] = false;
        }
    }

    DFS(new List<int>());
    return res;
}

```

## 3. Detect Cycle in Directed Graph

```csharp
public bool HasCycle(int n, List<int>[] graph) {
    var visited = new bool[n];
    var recStack = new bool[n];

    bool DFS(int node) {
        if (recStack[node]) return true;
        if (visited[node]) return false;

        visited[node] = true;
        recStack[node] = true;

        foreach (var neighbor in graph[node])
            if (DFS(neighbor)) return true;

        recStack[node] = false;
        return false;
    }

    for (int i = 0; i < n; i++)
        if (DFS(i)) return true;

    return false;
}
```

## 4. Word Search (Grid DFS with Backtracking)

```csharp
public bool Exist(char[][] board, string word) {
    int m = board.Length, n = board[0].Length;

    bool DFS(int i, int j, int k) {
        if (k == word.Length) return true;
        if (i < 0 || j < 0 || i >= m || j >= n || board[i][j] != word[k]) return false;

        char temp = board[i][j];
        board[i][j] = '#';

        bool found = DFS(i+1, j, k+1) || DFS(i-1, j, k+1) || DFS(i, j+1, k+1) || DFS(i, j-1, k+1);
        board[i][j] = temp;

        return found;
    }

    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            if (DFS(i, j, 0)) return true;

    return false;
}
```

## 🎯 Conclusion
DFS is a powerful tool for exploring depth, building recursive logic, and solving combinatorial problems. 
It shines when:

- You need to explore all configurations
- You’re working with trees, grids, or graphs
- You want to build elegant recursive solutions

Mastering DFS = mastering recursion, backtracking, and state tracking. 
Once internalized, DFS becomes your go-to strategy for depth-first reasoning and clean problem decomposition.

---


