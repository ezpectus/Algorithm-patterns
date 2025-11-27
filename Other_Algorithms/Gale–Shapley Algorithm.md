# Gale–Shapley Algorithm — Stable Matching Problem

---

## Origin & Motivation
The **Gale–Shapley algorithm** (1962) was introduced by **David Gale** and **Lloyd Shapley**.  
It solves the **Stable Marriage Problem**: given two sets of equal size (e.g., men and women), each with preference lists, find a **stable matching**.  

**Stability** means: no pair (m, w) exists such that both prefer each other over their current partners.  
This algorithm is also known as the **Deferred Acceptance Algorithm**.  

Applications include:
- **Medical residency programs** (National Resident Matching Program in the US).  
- **School choice systems**.  
- **Resource allocation** in economics and game theory.  

---

## Core Idea
1. Each man proposes to the most-preferred woman not yet rejected.  
2. Each woman considers all proposals and tentatively accepts the best one, rejecting the rest.  
3. Rejected men propose to their next choice.  
4. Repeat until all are matched.  

**Guarantee:**  
- Always terminates with a stable matching.  
- Optimal for the proposing side (men in the standard version).  

---

## Complexity Analysis

### Time Complexity
- Each man can propose to at most n women.  
- Each proposal is processed in O(1).  
- Total: **O(n²)**.  

### Space Complexity
- O(n²) for preference lists.  
- O(n) for current matches.  

---

## Comparison with Other Algorithms

| Algorithm              | Complexity | Stability Guarantee | Notes                                      |
|------------------------|------------|---------------------|--------------------------------------------|
| **Gale–Shapley**       | O(n²)      | Always              | Simple, efficient, guarantees stability     |
| Random Matching        | O(n)       | None                | Fast but unstable                          |
| Optimization (ILP/LP)  | Polynomial | Optional            | Can optimize fairness but more complex      |

---

## Key Concepts Recap
- **Stable matching:** no blocking pairs.  
- **Deferred acceptance:** proposals are tentative until final.  
- **Optimality:** result is optimal for proposers, pessimal for receivers.  
- **Fairness trade-off:** switching proposer side changes who benefits.  

---

## Gale–Shapley Algorithm (C++ Implementation)
```cpp
#include <bits/stdc++.h>
using namespace std;

class GaleShapley {
private:
    int n;
    vector<vector<int>> menPref, womenPref;
    vector<vector<int>> womenRank;  // woman → rank of man (lower = better)
    vector<int> wife, husband;
    vector<int> proposalIdx;
    queue<int> freeMen;

public:
    GaleShapley(int size,
                const vector<vector<int>>& mPref,
                const vector<vector<int>>& wPref)
        : n(size), menPref(mPref), womenPref(wPref) {
        wife.assign(n, -1);
        husband.assign(n, -1);
        proposalIdx.assign(n, 0);
        womenRank.assign(n, vector<int>(n));

        // Precompute women's ranking of men
        for (int w = 0; w < n; w++) {
            for (int r = 0; r < n; r++) {
                womenRank[w][womenPref[w][r]] = r;
            }
        }

        // All men start free
        for (int m = 0; m < n; m++) freeMen.push(m);
    }

    vector<int> solve() {
        while (!freeMen.empty()) {
            int m = freeMen.front(); freeMen.pop();
            int w = menPref[m][proposalIdx[m]++];

            if (husband[w] == -1) {
                // Woman is free
                husband[w] = m;
                wife[m] = w;
            }
            else if (womenRank[w][m] < womenRank[w][husband[w]]) {
                // Woman prefers new man
                int old = husband[w];
                wife[old] = -1;
                freeMen.push(old);

                husband[w] = m;
                wife[m] = w;
            }
            else {
                // Rejected → try next woman
                freeMen.push(m);
            }
        }
        return wife;  // wife[m] = woman for man m
    }
};

// USAGE 
int main() {
    int n = 4;
    vector<vector<int>> men = {
        {3,1,2,0}, {1,0,2,3}, {0,3,2,1}, {2,1,3,0}
    };
    vector<vector<int>> women = {
        {0,1,2,3}, {3,2,1,0}, {1,0,3,2}, {2,3,0,1}
    };

    GaleShapley gs(n, men, women);
    vector<int> matching = gs.solve();

    cout << "Stable matching (men-optimal):\n";
    for (int m = 0; m < n; m++) {
        cout << "Man " << m << " → Woman " << matching[m] << endl;
    }

    return 0;
}
```

## Gale–Shapley Algorithm (C# Implementation)
```cpp
using System;
using System.Collections.Generic;

public class GaleShapley
{
    private int n;
    private List<int>[] menPref, womenPref;
    private int[,] womenRank;
    private int[] wife, husband;
    private int[] proposalIdx;
    private Queue<int> freeMen;

    public GaleShapley(int size, List<int>[] mPref, List<int>[] wPref)
    {
        n = size;
        menPref = mPref;
        womenPref = wPref;
        wife = new int[n]; Array.Fill(wife, -1);
        husband = new int[n]; Array.Fill(husband, -1);
        proposalIdx = new int[n];
        womenRank = new int[n, n];
        freeMen = new Queue<int>();

        // Precompute women's ranking
        for (int w = 0; w < n; w++)
            for (int r = 0; r < n; r++)
                womenRank[w, womenPref[w][r]] = r;

        for (int i = 0; i < n; i++) freeMen.Enqueue(i);
    }

    public int[] Solve()
    {
        while (freeMen.Count > 0)
        {
            int m = freeMen.Dequeue();
            int w = menPref[m][proposalIdx[m]++];

            if (husband[w] == -1)
            {
                husband[w] = m;
                wife[m] = w;
            }
            else if (womenRank[w, m] < womenRank[w, husband[w]])
            {
                int old = husband[w];
                wife[old] = -1;
                freeMen.Enqueue(old);

                husband[w] = m;
                wife[m] = w;
            }
            else
            {
                freeMen.Enqueue(m);
            }
        }
        return wife;
    }
}

// USAGE 
class Program
{
    static void Main()
    {
        int n = 4;
        var men = new List<int>[] {
            new() {3,1,2,0}, new() {1,0,2,3},
            new() {0,3,2,1}, new() {2,1,3,0}
        };
        var women = new List<int>[] {
            new() {0,1,2,3}, new() {3,2,1,0},
            new() {1,0,3,2}, new() {2,3,0,1}
        };

        var gs = new GaleShapley(n, men, women);
        int[] matching = gs.Solve();

        Console.WriteLine("Stable matching (men-optimal):");
        for (int m = 0; m < n; m++)
            Console.WriteLine($"Man {m} → Woman {matching[m]}");
    }
}
```


### Step-by-Step Execution
1. **Initialization**
   - All men are free.
   - Each man has a pointer to the next woman on his preference list.
   - Women have no partners initially.

2. **Proposal Phase**
   - A free man proposes to the next woman on his list.
   - If the woman is free, she tentatively accepts.
   - If she already has a partner, she compares the new proposal with her current partner:
     - If she prefers the new man, she accepts him and rejects the old partner.
     - Otherwise, she rejects the new man.

3. **Iteration**
   - Rejected men continue proposing to the next woman on their list.
   - This continues until all men are matched.

4. **Termination**
   - The algorithm stops when no free men remain.
   - Each man is matched to exactly one woman, and the matching is stable.

---

### Why Stability is Guaranteed
- Suppose there exists a blocking pair (m, w) where both prefer each other over their current partners.
- This cannot happen because:
  - Men propose in order of preference.
  - Women always keep the best proposal they have received so far.
- Therefore, if m prefers w, he must have proposed to her earlier.
- If w rejected m, it means she had a better partner (from her perspective).
- Hence, no blocking pair can exist.

---

### Optimality Properties
- **Man-optimal:** Each man gets the best partner possible among all stable matchings.
- **Woman-pessimal:** Each woman gets the worst partner possible among all stable matchings.
- If women propose instead, the roles reverse.

---

### Complexity Analysis
- **Time Complexity:** O(n²)  
  - Each man can propose to at most n women.  
  - Each proposal is processed in constant time.  
- **Space Complexity:** O(n²)  
  - Preference lists require quadratic storage.  
  - Matches require linear storage.

---

### Practical Applications
- **Medical Residency Matching:** Used in the US National Resident Matching Program.  
- **School Choice:** Assigning students to schools based on preferences.  
- **Resource Allocation:** Matching donors to recipients, or workers to jobs.  
- **Game Theory & Economics:** Studying equilibria in matching markets.

---

### Comparison with Other Approaches

| Algorithm              | Complexity | Stability Guarantee | Notes                                      |
|------------------------|------------|---------------------|--------------------------------------------|
| **Gale–Shapley**       | O(n²)      | Always              | Efficient, guarantees stability             |
| Random Matching        | O(n)       | None                | Fast but unstable                          |
| Optimization (ILP/LP)  | Polynomial | Optional            | Can optimize fairness but more complex      |

---

##  Conclusion
The **Gale–Shapley algorithm** is a cornerstone of matching theory:
- Guarantees **stable matchings** in O(n²) time.  
- Simple to implement, yet powerful in practice.  
- Optimal for the proposing side, highlighting fairness trade-offs.  
- Widely applied in medicine, education, and economics.  

It exemplifies how a **clean algorithmic idea** can have both **deep theoretical impact** and **real-world utility**.


---





