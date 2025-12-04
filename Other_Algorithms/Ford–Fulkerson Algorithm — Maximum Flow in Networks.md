# Ford–Fulkerson Algorithm — Maximum Flow in Networks

---

## Origin & Motivation
The Ford–Fulkerson method was introduced by L. R. Ford Jr. and D. R. Fulkerson in 1956.  
It addresses the **maximum flow problem** in a directed graph with capacities on edges:

- **Objective:** maximize the flow from a source node `s` to a sink node `t`.  
- **Constraints:** flow on each edge ≤ capacity, and flow conservation at intermediate nodes.  
- **Key insight:** augmenting paths can iteratively increase flow until no more exist.

---

## Core Idea

### Residual Graph
- Construct a residual graph with capacities representing how much more flow can be pushed.  
- If an edge `(u,v)` has capacity `c` and current flow `f`, residual capacity = `c - f`.  
- Add reverse edges to allow flow cancellation.

### Augmenting Path
- Find a path from `s` to `t` in the residual graph.  
- Push as much flow as possible along this path (minimum residual capacity).  

### Iteration
- Update residual graph after each augmentation.  
- Repeat until no augmenting path exists.  

### Termination
- When no path from `s` to `t` exists in the residual graph, the current flow is maximum.  

---

## Where It’s Used
- **Network routing** — bandwidth allocation, traffic engineering.  
- **Bipartite matching** — job assignment, scheduling.  
- **Image segmentation** — min-cut/max-flow formulations.  
- **Operations research** — resource distribution, logistics.  

---

## Ford–Fulkerson Algorithm (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

class FordFulkerson {
private:
    int n;
    vector<vector<int>> capacity;
    vector<vector<int>> residual;

    bool bfs(int s, int t, vector<int>& parent) {
        vector<bool> visited(n, false);
        queue<int> q;
        q.push(s);
        visited[s] = true;
        parent[s] = -1;

        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (int v = 0; v < n; v++) {
                if (!visited[v] && residual[u][v] > 0) {
                    q.push(v);
                    parent[v] = u;
                    visited[v] = true;
                    if (v == t) return true;
                }
            }
        }
        return false;
    }

public:
    FordFulkerson(int nodes) : n(nodes) {
        capacity.assign(n, vector<int>(n, 0));
        residual = capacity;
    }

    void addEdge(int u, int v, int cap) {
        capacity[u][v] += cap;
        residual[u][v] = capacity[u][v];
        residual[v][u] = 0;  // reverse edge
    }

    int maxFlow(int source, int sink) {
        vector<int> parent(n);
        int max_flow = 0;

        while (bfs(source, sink, parent)) {
            int path_flow = INT_MAX;
            for (int v = sink; v != source; v = parent[v]) {
                int u = parent[v];
                path_flow = min(path_flow, residual[u][v]);
            }

            for (int v = sink; v != source; v = parent[v]) {
                int u = parent[v];
                residual[u][v] -= path_flow;
                residual[v][u] += path_flow;
            }

            max_flow += path_flow;
        }

        return max_flow;
    }

    // === TEST FUNCTION ===
    static void runTests() {
        cout << "Ford-Fulkerson — Max Flow Tests\n\n";

        // Test 1: Simple graph
        {
            FordFulkerson ff(4);
            ff.addEdge(0, 1, 10);
            ff.addEdge(0, 2, 5);
            ff.addEdge(1, 3, 15);
            ff.addEdge(2, 3, 10);
            int flow = ff.maxFlow(0, 3);
            cout << "Test 1: Expected 15, Got " << flow << " → " << (flow == 15 ? "PASS" : "FAIL") << "\n\n";
        }

        // Test 2: Classic example
        {
            FordFulkerson ff(6);
            ff.addEdge(0, 1, 16);
            ff.addEdge(0, 2, 13);
            ff.addEdge(1, 2, 10);
            ff.addEdge(1, 3, 12);
            ff.addEdge(2, 1, 4);
            ff.addEdge(2, 4, 14);
            ff.addEdge(3, 2, 9);
            ff.addEdge(3, 5, 20);
            ff.addEdge(4, 3, 7);
            ff.addEdge(4, 5, 4);
            int flow = ff.maxFlow(0, 5);
            cout << "Test 2: Expected 23, Got " << flow << " → " << (flow == 23 ? "PASS" : "FAIL") << "\n\n";
        }

        cout << "All tests completed!\n";
    }
};

int main() {
    FordFulkerson::runTests();
    return 0;
}
```

## Ford–Fulkerson Algorithm (C#)
```cpp
using System;
using System.Collections.Generic;

public class FordFulkerson
{
    private int n;
    private int[,] capacity;
    private int[,] residual;

    public FordFulkerson(int nodes)
    {
        n = nodes;
        capacity = new int[n, n];
        residual = new int[n, n];
    }

    public void AddEdge(int u, int v, int cap)
    {
        capacity[u, v] += cap;
        residual[u, v] = capacity[u, v];
    }

    private bool Bfs(int s, int t, int[] parent)
    {
        bool[] visited = new bool[n];
        Queue<int> q = new Queue<int>();
        q.Enqueue(s);
        visited[s] = true;
        parent[s] = -1;

        while (q.Count > 0)
        {
            int u = q.Dequeue();
            for (int v = 0; v < n; v++)
            {
                if (!visited[v] && residual[u, v] > 0)
                {
                    q.Enqueue(v);
                    parent[v] = u;
                    visited[v] = true;
                    if (v == t) return true;
                }
            }
        }
        return false;
    }

    public int MaxFlow(int source, int sink)
    {
        int[] parent = new int[n];
        int max_flow = 0;

        while (Bfs(source, sink, parent))
        {
            int path_flow = int.MaxValue;
            for (int v = sink; v != source; v = parent[v])
            {
                int u = parent[v];
                path_flow = Math.Min(path_flow, residual[u, v]);
            }

            for (int v = sink; v != source; v = parent[v])
            {
                int u = parent[v];
                residual[u, v] -= path_flow;
                residual[v, u] += path_flow;
            }

            max_flow += path_flow;
        }

        return max_flow;
    }

    // === TEST METHOD ===
    public static void RunTests()
    {
        Console.WriteLine("Ford-Fulkerson — Max Flow Tests\n");

        // Test 1
        {
            var ff = new FordFulkerson(4);
            ff.AddEdge(0, 1, 10);
            ff.AddEdge(0, 2, 5);
            ff.AddEdge(1, 3, 15);
            ff.AddEdge(2, 3, 10);
            int flow = ff.MaxFlow(0, 3);
            Console.WriteLine($"Test 1: Expected 15, Got {flow} → {(flow == 15 ? "PASS" : "FAIL")}\n");
        }

        // Test 2 — Classic
        {
            var ff = new FordFulkerson(6);
            ff.AddEdge(0, 1, 16);
            ff.AddEdge(0, 2, 13);
            ff.AddEdge(1, 2, 10);
            ff.AddEdge(1, 3, 12);
            ff.AddEdge(2, 1, 4);
            ff.AddEdge(2, 4, 14);
            ff.AddEdge(3, 2, 9);
            ff.AddEdge(3, 5, 20);
            ff.AddEdge(4, 3, 7);
            ff.AddEdge(4, 5, 4);
            int flow = ff.MaxFlow(0, 5);
            Console.WriteLine($"Test 2: Expected 23, Got {flow} → {(flow == 23 ? "PASS" : "FAIL")}\n");
        }

        Console.WriteLine("All tests completed!");
    }
}

class Program
{
    static void Main()
    {
        FordFulkerson.RunTests();
    }
}
```



## Time Complexity
- **Naive DFS:**  
  - O(E × max_flow)  
  - Can be very slow if the maximum flow value is large, since each augmentation may only push 1 unit of flow.
- **Edmonds–Karp (BFS):**  
  - O(V × E²)  
  - Guarantees polynomial time by always finding the shortest augmenting path in terms of edge count.
- **Practical performance:**  
  - Ford–Fulkerson with BFS is efficient for moderate graphs and widely used in practice.

---

## Space Complexity
- **Residual graph storage:**  
  - O(V²) if implemented with adjacency matrix.  
  - O(E) if implemented with adjacency list.
- **Auxiliary arrays:**  
  - O(V) for parent tracking and visited flags during BFS/DFS.
- **Total:**  
  - Linear in graph size, dominated by residual graph representation.

---

## Key Concepts Recap
- **Residual graph:** Tracks remaining capacity and reverse flows for cancellation.  
- **Augmenting path:** Path along which additional flow can be pushed.  
- **Max-flow min-cut theorem:** Maximum flow equals minimum cut capacity.  
- **Variants:**  
  - Edmonds–Karp (BFS) → polynomial guarantee.  
  - Dinic’s algorithm → faster with blocking flows and layered graphs.  

---

## ✅ Conclusion
The **Ford–Fulkerson algorithm** is a cornerstone of network flow theory:
- Simple yet powerful.  
- Forms the basis for advanced algorithms like **Edmonds–Karp** and **Dinic**.  
- Widely applied in computer science, operations research, and engineering.  

It exemplifies how iterative augmentation and residual capacity analysis can solve complex optimization problems efficiently, balancing theoretical complexity with practical effectiveness.

---
