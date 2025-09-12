# Breadth-First Search — Algorithm Template

## Introduction
Breadth-First Search (BFS) is a graph traversal technique that explores nodes level by level, 
starting from a source node and visiting all its neighbors before moving deeper.

## The core idea:

- "Explore all nodes at distance d before moving to distance d+1."

- BFS is ideal for finding the shortest path in an unweighted graph or grid, and for problems involving step-by-step expansion.

## When to Apply BFS

**Use BFS when:**

- You need the shortest path or minimum steps.
- The graph is unweighted (or all edges have equal cost).
- You need to explore all reachable nodes from a source.

## You’re working with:

- Grids (2D traversal)
- Word ladders / transformations
- Multi-source expansion (e.g., rotten oranges)
- Level-order traversal in trees


## BFS Template — Single Source

```csharp
Queue<Node> queue = new Queue<Node>();
HashSet<Node> visited = new HashSet<Node>();

queue.Enqueue(start);
visited.Add(start);

while (queue.Count > 0) {
    var node = queue.Dequeue();
    // process node

    foreach (var neighbor in GetNeighbors(node)) {
        if (!visited.Contains(neighbor)) {
            queue.Enqueue(neighbor);
            visited.Add(neighbor);
        }
    }
}
```

## BFS with Level Tracking
Useful when you need to count steps or levels (e.g., shortest path).

```csharp
int steps = 0;
Queue<Node> queue = new Queue<Node>();
HashSet<Node> visited = new HashSet<Node>();

queue.Enqueue(start);
visited.Add(start);

while (queue.Count > 0) {
    int size = queue.Count;
    for (int i = 0; i < size; i++) {
        var node = queue.Dequeue();
        // process node

        foreach (var neighbor in GetNeighbors(node)) {
            if (!visited.Contains(neighbor)) {
                queue.Enqueue(neighbor);
                visited.Add(neighbor);
            }
        }
    }
    steps++;
}
```

## BFS on Grids (2D Matrix)

```csharp
int[][] directions = new int[][] {
    new int[]{0,1}, new int[]{1,0}, new int[]{0,-1}, new int[]{-1,0}
};

Queue<(int x, int y)> queue = new Queue<(int, int)>();
bool[,] visited = new bool[m, n];

queue.Enqueue((startX, startY));
visited[startX, startY] = true;

while (queue.Count > 0) {
    var (x, y) = queue.Dequeue();
    foreach (var dir in directions) {
        int nx = x + dir[0], ny = y + dir[1];
        if (IsValid(nx, ny) && !visited[nx, ny]) {
            queue.Enqueue((nx, ny));
            visited[nx, ny] = true;
        }
    }
}
```

## Multi-Source BFS

Start BFS from multiple sources simultaneously.

```csharp
Queue<Node> queue = new Queue<Node>();
HashSet<Node> visited = new HashSet<Node>();

foreach (var source in sources) {
    queue.Enqueue(source);
    visited.Add(source);
}

while (queue.Count > 0) {
    var node = queue.Dequeue();
    foreach (var neighbor in GetNeighbors(node)) {
        if (!visited.Contains(neighbor)) {
            queue.Enqueue(neighbor);
            visited.Add(neighbor);
        }
    }
}
```

## Common BFS Problems

| **Problem Type**         | **Example Problems**                          |
|--------------------------|-----------------------------------------------|
| Shortest path            | Word Ladder, Maze                             |
| Grid traversal           | Rotting Oranges, Number of Islands            |
| Level-order traversal    | Binary Tree Level Order Traversal             |
| Multi-source expansion   | Walls and Gates, Fire Spread                  |
| State transitions        | Lock Combinations, Sliding Puzzle             |


## Complexity

- Time: $$O(V + E)$$ for graphs $$O(m \cdot n)$$ for grids
- Space: $$O(V)$$ for visited set and queue

## Common Mistakes in BFS

- ❌ Forgetting to mark nodes as visited → can lead to infinite loops
- ❌ Enqueuing nodes multiple times → causes redundant work
- ❌ Not handling grid boundaries correctly → index out of bounds
- ❌ Using DFS when BFS is needed for shortest path → incorrect results
- ❌ Not tracking levels when required → loss of depth information

---

## Memorization Tips

- **BFS = Queue + Visited Set**
- Think in **layers** — each loop iteration = one level deeper
- For **grids**, always define a `directions` array
- For **shortest path**, track `steps` or `distance`
- For **multi-source BFS**, enqueue **all sources** at the start

---

## Conclusion

BFS is a powerful tool for exploring graphs and grids in a structured, level-by-level manner.  
It shines in:
- **Shortest path problems**
- **Multi-source expansion**
- **State-space traversal**

Mastering BFS means mastering the art of **controlled exploration** — knowing:
- When to expand
- When to prune
- How to track progress

---

## BFS Training Progression

1. ✅ Simple grid traversal  
2. ✅ Word ladder transformations  
3. ✅ Multi-source BFS  
4. ✅ BFS with state encoding (e.g., lock puzzles, visited masks)

Once internalized, BFS becomes your go-to strategy for **clarity**, **correctness**, and **optimality** in graph-based problems.




---
