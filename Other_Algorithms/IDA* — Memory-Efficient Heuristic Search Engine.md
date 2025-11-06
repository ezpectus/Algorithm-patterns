# IDA* — Memory-Efficient Heuristic Search Engine

---

## Origin & Motivation

**IDA*** (Iterative Deepening A*) was introduced in **1985** by **Richard Korf** as a solution to **A*’s memory limitations**.  
While A* uses a **priority queue** and stores **all visited nodes**, IDA* performs **depth-first search with a cost threshold**, iteratively increasing the bound.

It retains **A*’s optimality** and **heuristic guidance**, but with **linear space complexity**, making it ideal for **large or implicit state spaces**.

---

## Where It’s Used

| Domain | Use Case |
|--------|----------|
| Puzzle Solvers | 15-puzzle, Rubik’s Cube |
| AI Planning | Goal-directed search with large state space |
| Game Solvers | Turn-based logic, combinatorial states |
| Constraint Satisfaction | Bounded search with heuristics |
| Robotics | Motion planning with limited memory |

---

## When to Use IDA* vs Alternatives

| Scenario | IDA* | A* | DFS | BFS |
|--------|------|----|-----|-----|
| Large state space | Yes | No | Yes | No |
| Memory-constrained environment | Yes | No | Yes | No |
| Heuristic available | Yes | Yes | No | No |
| Optimal path required | Yes | Yes | No | No |
| Fast approximate solution | No | Yes | Yes | Yes |

---

## Core Idea

* Perform **depth-first search** with a cost threshold `f(n) = g(n) + h(n)`  
* If a node’s cost **exceeds** the current threshold, return that cost as a **candidate** for the next iteration  
* **Increase** the threshold to the **minimum cost** that exceeded it  
* **Repeat** until the goal is found  

This mimics **A*’s expansion order** but **avoids storing the entire frontier**.

---

## Implementation (C++)

```cpp
struct State {
    int g;         // cost so far
    int h;         // heuristic
    int f() const { return g + h; }
    // Add domain-specific fields and operators
};

int ida_star(State start, function<int(State)> heuristic, function<bool(State)> is_goal, function<vector<State>(State)> expand) {
    int threshold = heuristic(start);

    while (true) {
        int temp = dfs(start, 0, threshold, heuristic, is_goal, expand);
        if (temp == -1) return threshold; // goal found
        if (temp == INT_MAX) return -1;   // no solution
        threshold = temp;
    }
}

int dfs(State node, int g, int threshold, function<int(State)> h, function<bool(State)> goal, function<vector<State>(State)> expand) {
    int f = g + h(node);
    if (f > threshold) return f;
    if (goal(node)) return -1;

    int minThreshold = INT_MAX;
    for (State succ : expand(node)) {
        int temp = dfs(succ, g + 1, threshold, h, goal, expand);
        if (temp == -1) return -1;
        minThreshold = min(minThreshold, temp);
    }
    return minThreshold;
}
```



## Implementation (C#)

```csharp
public class State {
    public int G, H;
    public int F => G + H;
    // Add domain-specific fields and methods
}

public class IDAStar {
    public static int Run(State start, Func<State, int> heuristic, Func<State, bool> isGoal, Func<State, List<State>> expand) {
        int threshold = heuristic(start);

        while (true) {
            int temp = DFS(start, 0, threshold, heuristic, isGoal, expand);
            if (temp == -1) return threshold;
            if (temp == int.MaxValue) return -1;
            threshold = temp;
        }
    }

    private static int DFS(State node, int g, int threshold, Func<State, int> h, Func<State, bool> goal, Func<State, List<State>> expand) {
        int f = g + h(node);
        if (f > threshold) return f;
        if (goal(node)) return -1;

        int minThreshold = int.MaxValue;
        foreach (var succ in expand(node)) {
            int temp = DFS(succ, g + 1, threshold, h, goal, expand);
            if (temp == -1) return -1;
            minThreshold = Math.Min(minThreshold, temp);
        }
        return minThreshold;
    }
}
```

## Complexity Analysis

| Metric | Value |
|--------|-------|
| **Time** | O(b^d) worst-case |
| **Space** | O(d) — linear stack |
| **Optimality** | Yes if heuristic is admissible |
| **Completeness** | Yes if finite branching |

> *b*: branching factor  
> *d*: depth of optimal solution  
> Each iteration may **re-expand nodes**, but memory stays **linear**

---

## Why It’s Elegant

* **Memory-efficient**: no frontier or visited set needed  
* **Heuristic-guided**: expands promising paths first  
* **Optimal**: finds shortest path if heuristic is admissible  
* **Modular**: works with any domain that supports DFS and heuristics  

---

## Impact of Heuristic

| Heuristic Quality | Behavior |
|-------------------|----------|
| **Admissible** | Guarantees optimality |
| **Consistent** | Reduces re-expansion |
| **Weak** | More iterations, slower convergence |
| **Zero** | Reduces to iterative DFS |

---

## Pitfalls

* **Re-expansion overhead**: no memoization → repeated work  
* **No visited set**: cycles must be handled manually  
* **Threshold tuning**: poor heuristics lead to many iterations  
* **Not suitable for dense graphs**: DFS may explore too deeply  

---

## Conclusion

IDA* is a **memory-efficient heuristic search engine**:

* Combines **A*’s optimality** with **DFS’s space efficiency**  
* Ideal for **large or implicit state spaces**  
* **Modular and domain-agnostic**  
* Guarantees **optimality** with admissible heuristics  

> **Key takeaway**:  
> **IDA* shines when memory is tight and the state space is deep.**  
> It’s the **go-to algorithm** for **puzzle solvers and AI planners** where A* would run out of space.

---
