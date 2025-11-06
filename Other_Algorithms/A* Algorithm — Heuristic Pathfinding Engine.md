# A* Algorithm — Heuristic Pathfinding Engine

---

## Origin & Motivation

**A*** (A-star) was introduced in **1968** by **Hart, Nilsson, and Raphael**.  
It **extends Dijkstra’s algorithm** by adding a **heuristic function** to guide the search **toward the goal**.

Instead of expanding the **closest node by distance**, A* picks the node with the **lowest estimated total cost**:

\[
f(n) = g(n) + h(n)
\]

- \( g(n) \): **actual cost** from start to node \( n \)  
- \( h(n) \): **estimated cost** from \( n \) to goal (**heuristic**)

This makes A* **optimal** and **complete** **if** the heuristic is **admissible** (never overestimates).

---

## Where It’s Used

| Domain | Use Case |
|-------|----------|
| **Game AI** | Grid pathfinding, unit movement |
| **Robotics** | Navigation, obstacle avoidance |
| **GPS** | Shortest route with traffic heuristics |
| **Puzzles** | 8-puzzle, sliding tiles |
| **AI Planning** | State-space search with goal bias |

---

## When to Use A* vs Alternatives

| Scenario | A* | Dijkstra | BFS | DFS |
|--------|----|----------|-----|-----|
| Weighted graph + goal | Yes | Yes | No | No |
| Heuristic available | Yes | No | No | No |
| Goal-directed search | Yes | No | No | No |
| All-pairs shortest paths | No | No | No | No |
| Unweighted graph | No | No | Yes | No |
| Exploration without goal | No | Yes | Yes | Yes |

---

## Core Idea

1. **Priority queue** ordered by \( f(n) = g(n) + h(n) \)
2. Start: \( g(start) = 0 \), push start node
3. While queue not empty:
   - Pop node with **lowest \( f \)**
   - If it's **goal** → done
   - For each neighbor:
     - Compute **tentative \( g \)**
     - If better → **update** and **push**
4. **Reconstruct path** via `came_from`

---

## Implementation (C++)

```cpp
struct Node {
    int id;
    double g, f;
    bool operator>(const Node& other) const { return f > other.f; }
};

vector<int> a_star(int n, int start, int goal,
                   vector<vector<pair<int, double>>>& adj,
                   function<double(int)> heuristic) {
    vector<double> g(n, 1e9);
    vector<int> came_from(n, -1);
    priority_queue<Node, vector<Node>, greater<Node>> pq;

    g[start] = 0;
    pq.push({start, 0, heuristic(start)});

    while (!pq.empty()) {
        Node curr = pq.top(); pq.pop();
        if (curr.id == goal) break;

        for (auto& [to, cost] : adj[curr.id]) {
            double new_g = g[curr.id] + cost;
            if (new_g < g[to]) {
                g[to] = new_g;
                came_from[to] = curr.id;
                pq.push({to, new_g, new_g + heuristic(to)});
            }
        }
    }

    // Reconstruct
    vector<int> path;
    for (int v = goal; v != -1; v = came_from[v])
        path.push_back(v);
    reverse(path.begin(), path.end());
    return path;
}
```
## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class Node : IComparable<Node> {
    public int Id;
    public double G, F;
    public Node(int id, double g, double f) { Id = id; G = g; F = f; }
    public int CompareTo(Node other) => F.CompareTo(other.F);
}

public class AStar {
    public static List<int> Run(int n, int start, int goal, List<(int, double)>[] adj, Func<int, double> heuristic) {
        var g = new double[n];
        Array.Fill(g, 1e9);
        var cameFrom = new int[n];
        Array.Fill(cameFrom, -1);

        var pq = new SortedSet<Node>();
        g[start] = 0;
        pq.Add(new Node(start, 0, heuristic(start)));

        while (pq.Count > 0) {
            var current = pq.Min;
            pq.Remove(current);
            if (current.Id == goal) break;

            foreach (var (neighbor, cost) in adj[current.Id]) {
                double tentativeG = g[current.Id] + cost;
                if (tentativeG < g[neighbor]) {
                    g[neighbor] = tentativeG;
                    cameFrom[neighbor] = current.Id;
                    pq.Add(new Node(neighbor, tentativeG, tentativeG + heuristic(neighbor)));
                }
            }
        }

        var path = new List<int>();
        for (int v = goal; v != -1; v = cameFrom[v])
            path.Add(v);
        path.Reverse();
        return path;
    }
}
```
## Complexity Analysis

| Phase             | Complexity   |
|-------------------|--------------|
| **Initialization** | O(V)         |
| **Main loop**      | O(E log V)   |
| **Space**          | O(V + E)     |

* Each node is processed **once**  
* Each edge may trigger a **priority queue update**  
* Heuristic function should be **O(1)** for efficiency

---

## Why It’s Efficient

* A* **avoids unnecessary exploration** by guiding search **toward the goal**  
* With a good heuristic (e.g., **Manhattan** or **Euclidean distance**), it expands **far fewer nodes** than Dijkstra  
* Still **guarantees optimality** if heuristic is **admissible** and **consistent**

---

## Impact of Heuristic

| Heuristic Type     | Behavior                              |
|--------------------|---------------------------------------|
| **Admissible**     | Guarantees optimal path               |
| **Consistent**     | Ensures no reprocessing               |
| **Overestimating** | May miss optimal path (unsafe)        |
| **Zero heuristic** | Reduces to Dijkstra                   |

---

## Pitfalls

* Heuristic **must be admissible** for correctness  
* Priority queue **may contain duplicates** → always check `g[n]` before processing  
* Graph must be **connected**; unreachable goal returns partial path or empty  
* Inconsistent heuristic may cause **reprocessing** and inefficiency

---

## Conclusion

A* is a **goal-directed shortest path engine**:

* Combines **Dijkstra’s optimality** with **heuristic speed**  
* Ideal for **spatial graphs and game maps**  
* Adapts to **domain knowledge** via heuristic  
* Guarantees **optimality** with admissible heuristics  

> **Key takeaway**:  
> **A* shines when you have a clear goal and a good heuristic.**  
> It’s the **go-to algorithm** for **intelligent pathfinding** in weighted graphs with spatial structure.



---
