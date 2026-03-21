# CDCL — Conflict-Driven Clause Learning

## Origin & Motivation

CDCL was developed through a series of solver implementations in the 1990s — most notably GRASP (Marques-Silva and Sakallah, 1996) and Chaff (Moskewicz et al., 2001). It extends DPLL with four interacting mechanisms: **implication graph construction**, **conflict analysis**, **non-chronological backjumping**, and **clause learning**. Together these transform the worst-case exponential DPLL into a solver that routinely handles industrial SAT instances with millions of variables and clauses in seconds.

The fundamental insight: when a conflict occurs, it is caused by a small subset of past decisions. By analyzing the implication graph, the solver derives a new clause — a **learned clause** — that is logically implied by the original formula and that rules out the exact combination of decisions that led to the conflict. The solver then jumps back to the earliest decision level mentioned in the learned clause, rather than just one level.

Complexity: **O(2^n)** worst case. In practice near-polynomial on structured industrial instances.

---

## Where It Is Used

- Industrial hardware and software verification
- Automated theorem proving over propositional logic
- AI planning and scheduling (SAT encoding)
- Cryptanalysis and equivalence checking
- Package dependency solving (APT, Cargo, pip-compile)
- Compiler and synthesis tools (Yosys, ABC)
- Competitive programming: encoding NP-hard problems into CNF

---

## Key Concepts

| Concept | Description |
|---|---|
| Decision level | Depth of the current branching stack; increments on each decision |
| Implication graph | DAG recording which assignments forced which others via BCP |
| Conflict clause | A clause all of whose literals are falsified under current assignment |
| UIP (Unique Implication Point) | A node in the implication graph dominating all paths from the last decision to the conflict |
| Learned clause | Negation of the assignments at the cut separating UIP from conflict |
| Backjump level | Second-highest decision level in the learned clause |
| LBD (Literal Block Distance) | Number of distinct decision levels in a clause; quality metric for learned clauses |
| VSIDS | Variable State Independent Decaying Sum — activity-based branching heuristic |

---

## Algorithm Structure

```
CDCL(F):
    dl = 0                          // decision level
    if BCP(F, dl) == CONFLICT:
        return UNSAT                // conflict at level 0 is unconditional

    loop:
        v = pick_variable()         // VSIDS: highest activity unassigned var
        if v == NONE: return SAT    // all variables assigned

        dl++
        assign(v, polarity(v))      // decision assignment at level dl
        trail.push(v, dl, reason=NULL)

        while True:
            status = BCP(F, dl)     // watched-literal unit propagation
            if status != CONFLICT:
                break               // no conflict, make next decision

            if dl == 0:
                return UNSAT        // conflict at root level

            learned, backjump_dl = analyze_conflict()
            add_clause(F, learned)  // add learned clause permanently
            bump_activity(learned)  // VSIDS: bump vars in learned clause
            backtrack(backjump_dl)  // undo assignments above backjump_dl
            dl = backjump_dl

            if should_restart():    // Luby / geometric policy
                backtrack(0)
                dl = 0

            if should_clean():      // remove low-LBD learned clauses
                clean_learned_clauses()
```

---

## Watched Literals (Two-Pointer BCP)

Each clause watches exactly **two** of its literals. A clause is re-examined only when one of its watched literals becomes false. When that happens, the solver searches for another unfalsfied literal to watch. If none exists, the clause is unit (force the remaining unset literal) or conflicting (all literals false).

Cost: **O(1) amortized per assignment** — the key performance difference over naive BCP.

```
on assign(l = false):
    for each clause C watching literal l:
        find another literal l' in C that is not false
        if found:    move watch from l to l'
        elif one unset literal remains: force it (unit)
        else:        return CONFLICT
```

---

## Conflict Analysis and First-UIP Cut

After BCP returns CONFLICT, CDCL traverses the implication graph backward from the conflict node to find the **First UIP** — the closest dominator to the conflict on the path from the last decision.

The **learned clause** is constructed by taking the negation of all assignments on the **reason side** of the cut separating First UIP from the conflict:

```
analyze_conflict():
    clause = {conflict literals}
    seen = {}
    at_current_level = count of literals at dl in clause

    while at_current_level > 1:        // resolve until only First UIP remains at dl
        l = last assigned literal in clause at current level
        reason = reason_clause(l)       // clause that forced l
        resolve(clause, reason, l)      // remove l, add reason's other literals
        at_current_level = recount

    learned = clause
    backjump_dl = max decision level among non-UIP literals in learned
    return learned, backjump_dl
```

The resulting learned clause has the First UIP as its only literal at the current decision level. After backjumping, BCP immediately forces the UIP literal (the learned clause becomes unit).

---

## Complexity Analysis

| Component | Cost per conflict |
|---|---|
| BCP (watched literals) | O(propagations) amortized O(1) each |
| Conflict analysis | O(clause size * resolution steps) |
| Backjump | O(trail length undone) |
| VSIDS bump + decay | O(vars in learned clause) |
| Clause deletion (LBD) | O(learned clause count), periodic |

| Metric | Bound |
|---|---|
| Worst case | O(2^n) |
| Learned clauses space | O(conflicts * avg_clause_size) |
| Number of conflicts before solve | Empirically O(n^1..2) on structured instances |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// Literal encoding: variable v (1-indexed)
// positive literal = 2*v,  negative literal = 2*v+1
// negation of literal l = l ^ 1

struct CDCL {
    // -------------------------------------------------------
    // Data
    // -------------------------------------------------------
    int n;                          // number of variables
    vector<vector<int>> clauses;    // all clauses (original + learned)
    int orig_size;                  // number of original clauses

    vector<int>  val;               // val[v]: 0=unset, 1=true, -1=false
    vector<int>  level;             // decision level of assignment
    vector<int>  reason;            // index of clause that forced this var (-1 = decision)
    vector<int>  trail;             // assignment order
    vector<int>  trail_lim;        // trail.size() at each decision level

    vector<vector<int>> watch;      // watch[lit] = clause indices watching this lit
    int dl;                         // current decision level

    // VSIDS activity
    vector<double> activity;
    double var_inc = 1.0;
    static constexpr double var_decay = 0.95;

    // Phase saving (polarity cache)
    vector<int> phase;

    int lit_pos(int v) { return 2 * v; }
    int lit_neg(int v) { return 2 * v + 1; }
    int var_of(int l)  { return l >> 1; }
    bool sign(int l)   { return l & 1; }          // true = negative literal
    bool lit_true(int l)  { int v=var_of(l); return v&&val[v]==(sign(l)?-1:1); }
    bool lit_false(int l) { int v=var_of(l); return v&&val[v]==(sign(l)? 1:-1); }

    // -------------------------------------------------------
    // Construction
    // -------------------------------------------------------
    CDCL(int n, const vector<vector<int>>& raw_clauses) :
        n(n), val(n+1,0), level(n+1,0), reason(n+1,-1),
        watch(2*(n+1)), activity(n+1,0.0), phase(n+1,1), dl(0)
    {
        // Encode raw clauses (raw literal l: positive=l, negative=-l)
        for (auto& rc : raw_clauses) {
            vector<int> c;
            for (int l : rc)
                c.push_back(l > 0 ? lit_pos(l) : lit_neg(-l));
            clauses.push_back(c);
        }
        orig_size = clauses.size();
        attach_all();
    }

    void attach(int ci) {
        auto& c = clauses[ci];
        if (c.size() >= 1) watch[c[0]^1].push_back(ci); // watch negation
        if (c.size() >= 2) watch[c[1]^1].push_back(ci);
    }

    void attach_all() {
        for (int i = 0; i < (int)clauses.size(); i++) attach(i);
    }

    // -------------------------------------------------------
    // Assignment
    // -------------------------------------------------------
    void assign(int l, int d, int r) {
        int v = var_of(l);
        val[v]    = sign(l) ? -1 : 1;
        level[v]  = d;
        reason[v] = r;
        phase[v]  = sign(l) ? -1 : 1;
        trail.push_back(l);
    }

    void unassign(int l) {
        val[var_of(l)] = 0;
        reason[var_of(l)] = -1;
    }

    // -------------------------------------------------------
    // Watched-literal BCP
    // Returns conflict clause index, or -1 if no conflict
    // -------------------------------------------------------
    int propagate() {
        static int qhead = 0;
        qhead = (int)trail.size(); // process from current position backward
        // Actually iterate all newly assigned literals
        int prop_ptr = trail_lim.empty() ? 0 : (int)trail.size();
        // Use a simple queue over trail
        int qi = trail_lim.empty() ? 0 : trail_lim.back();
        while (qi < (int)trail.size()) {
            int l = trail[qi++];
            int false_lit = l ^ 1;        // this literal just became false
            auto& wl = watch[false_lit];
            for (int i = 0; i < (int)wl.size(); ) {
                int ci = wl[i];
                auto& c = clauses[ci];
                // Make sure false_lit is at position 1
                if (c[0] == false_lit) swap(c[0], c[1]);
                // c[1] == false_lit now
                if (lit_true(c[0])) { i++; continue; }  // clause satisfied by c[0]
                // Search for new watch
                bool found = false;
                for (int j = 2; j < (int)c.size(); j++) {
                    if (!lit_false(c[j])) {
                        swap(c[1], c[j]);
                        wl.erase(wl.begin() + i);
                        watch[c[1]^1].push_back(ci);
                        found = true;
                        break;
                    }
                }
                if (!found) {
                    if (lit_false(c[0])) return ci;  // conflict
                    // c[0] is unit
                    assign(c[0], dl, ci);
                    i++;
                }
            }
        }
        return -1;
    }

    // -------------------------------------------------------
    // Conflict analysis — First UIP scheme
    // Returns learned clause; sets backjump_dl
    // -------------------------------------------------------
    vector<int> analyze(int conflict_ci, int& backjump_dl) {
        vector<int> learned;
        vector<bool> seen(n+1, false);
        int counter = 0;
        int l = -1;
        int idx = (int)trail.size() - 1;

        vector<int>* reason_clause = &clauses[conflict_ci];
        do {
            for (int rl : *reason_clause) {
                int v = var_of(rl);
                if (!seen[v]) {
                    seen[v] = true;
                    activity[v] += var_inc;   // VSIDS bump
                    if (level[v] == dl) counter++;
                    else if (level[v] > 0) learned.push_back(rl);
                }
            }
            // Find last assigned variable at current level in seen
            while (!seen[var_of(trail[idx])]) idx--;
            l = trail[idx--];
            int v = var_of(l);
            seen[v] = false;
            counter--;
            if (counter > 0)
                reason_clause = &clauses[reason[v]];
        } while (counter > 0);

        learned.insert(learned.begin(), l ^ 1); // UIP literal (negated)

        // Compute backjump level
        backjump_dl = 0;
        for (int i = 1; i < (int)learned.size(); i++)
            backjump_dl = max(backjump_dl, level[var_of(learned[i])]);

        // VSIDS decay
        var_inc /= var_decay;
        for (int v = 1; v <= n; v++) activity[v] *= var_decay;

        return learned;
    }

    // -------------------------------------------------------
    // Backtrack to decision level d
    // -------------------------------------------------------
    void backtrack(int d) {
        while ((int)trail.size() > (d < (int)trail_lim.size() ? trail_lim[d] : 0)) {
            unassign(trail.back());
            trail.pop_back();
        }
        while ((int)trail_lim.size() > d) trail_lim.pop_back();
        dl = d;
    }

    // -------------------------------------------------------
    // VSIDS variable selection
    // -------------------------------------------------------
    int pick_var() {
        int best = -1;
        double best_act = -1;
        for (int v = 1; v <= n; v++) {
            if (val[v] == 0 && activity[v] > best_act) {
                best = v; best_act = activity[v];
            }
        }
        return best;
    }

    // -------------------------------------------------------
    // Main CDCL loop
    // -------------------------------------------------------
    bool solve() {
        // Check trivially forced units at level 0
        if (propagate() != -1) return false;

        while (true) {
            int v = pick_var();
            if (v == -1) return true;  // all assigned

            dl++;
            trail_lim.push_back(trail.size());
            // Phase saving: use last known polarity
            int decision_lit = phase[v] == 1 ? lit_pos(v) : lit_neg(v);
            assign(decision_lit, dl, -1);

            while (true) {
                int conflict = propagate();
                if (conflict == -1) break;  // no conflict

                if (dl == 0) return false;  // conflict at root = UNSAT

                int backjump_dl;
                vector<int> learned = analyze(conflict, backjump_dl);

                // Add learned clause
                int ci = clauses.size();
                clauses.push_back(learned);
                attach(ci);

                backtrack(backjump_dl);
                dl = backjump_dl;

                // The learned clause is now unit: force its UIP literal
                assign(learned[0], dl, ci);
            }
        }
    }

    // -------------------------------------------------------
    // Learned clause deletion (LBD-based, periodic)
    // Keep only clauses with LBD <= threshold or original clauses
    // -------------------------------------------------------
    int lbd(const vector<int>& c) {
        set<int> levels;
        for (int l : c)
            if (val[var_of(l)] != 0) levels.insert(level[var_of(l)]);
        return levels.size();
    }

    void clean_learned(int lbd_limit = 4) {
        // Rebuild watch lists; remove learned clauses with lbd > limit
        for (int i = 0; i < 2*(n+1); i++) watch[i].clear();
        vector<vector<int>> kept;
        for (int i = 0; i < (int)clauses.size(); i++) {
            if (i < orig_size || lbd(clauses[i]) <= lbd_limit)
                kept.push_back(clauses[i]);
        }
        clauses = move(kept);
        orig_size = min(orig_size, (int)clauses.size());
        attach_all();
    }

    void print() {
        for (int v = 1; v <= n; v++)
            printf("x%d = %s\n", v, val[v]==1?"true":"false");
    }
};

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
int main() {
    // (x1 ∨ ¬x2) ∧ (¬x1 ∨ x3) ∧ (x2 ∨ ¬x3) ∧ (¬x1 ∨ ¬x2 ∨ ¬x3)
    int n = 3;
    vector<vector<int>> clauses = {
        {1,-2},{-1,3},{2,-3},{-1,-2,-3}
    };

    CDCL solver(n, clauses);
    if (solver.solve()) { puts("SAT"); solver.print(); }
    else puts("UNSAT");

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class CDCL {
    private readonly int n;
    private readonly List<List<int>> clauses;
    private readonly int origSize;

    private readonly int[]    val;       // 0=unset, 1=true, -1=false
    private readonly int[]    lvl;       // decision level of assignment
    private readonly int[]    reason;    // clause index that forced var, -1=decision
    private readonly List<int> trail = new();
    private readonly List<int> trailLim = new();
    private readonly List<int>[] watch;
    private readonly double[] activity;
    private readonly int[]    phase;
    private double varInc = 1.0;
    private const double VarDecay = 0.95;
    private int dl = 0;

    private int LitPos(int v) => 2 * v;
    private int LitNeg(int v) => 2 * v + 1;
    private int VarOf(int l)  => l >> 1;
    private bool IsNeg(int l) => (l & 1) == 1;
    private bool LitTrue(int l)  { int v=VarOf(l); return v>0&&val[v]==(IsNeg(l)?-1:1); }
    private bool LitFalse(int l) { int v=VarOf(l); return v>0&&val[v]==(IsNeg(l)? 1:-1); }

    public CDCL(int n, List<int[]> raw) {
        this.n  = n;
        val      = new int[n+1];
        lvl      = new int[n+1];
        reason   = new int[n+1]; Array.Fill(reason, -1);
        activity = new double[n+1];
        phase    = new int[n+1]; Array.Fill(phase, 1);
        watch    = new List<int>[2*(n+1)];
        for (int i = 0; i < watch.Length; i++) watch[i] = new();

        clauses = new();
        foreach (var rc in raw) {
            var c = rc.Select(l => l > 0 ? LitPos(l) : LitNeg(-l)).ToList();
            clauses.Add(c);
        }
        origSize = clauses.Count;
        AttachAll();
    }

    private void Attach(int ci) {
        var c = clauses[ci];
        if (c.Count >= 1) watch[c[0]^1].Add(ci);
        if (c.Count >= 2) watch[c[1]^1].Add(ci);
    }

    private void AttachAll() { for (int i=0;i<clauses.Count;i++) Attach(i); }

    private void Assign(int l, int d, int r) {
        int v = VarOf(l);
        val[v]    = IsNeg(l) ? -1 : 1;
        lvl[v]    = d;
        reason[v] = r;
        phase[v]  = IsNeg(l) ? -1 : 1;
        trail.Add(l);
    }

    private void Unassign(int l) { val[VarOf(l)] = 0; reason[VarOf(l)] = -1; }

    private int Propagate() {
        int qi = trailLim.Count > 0 ? trailLim[^1] : 0;
        while (qi < trail.Count) {
            int l = trail[qi++];
            int fl = l ^ 1;
            var wl = watch[fl];
            for (int i = 0; i < wl.Count; ) {
                int ci = wl[i];
                var c  = clauses[ci];
                if (c[0] == fl) { (c[0], c[1]) = (c[1], c[0]); }
                if (LitTrue(c[0])) { i++; continue; }
                bool found = false;
                for (int j = 2; j < c.Count; j++) {
                    if (!LitFalse(c[j])) {
                        (c[1], c[j]) = (c[j], c[1]);
                        wl.RemoveAt(i);
                        watch[c[1]^1].Add(ci);
                        found = true; break;
                    }
                }
                if (!found) {
                    if (LitFalse(c[0])) return ci;
                    Assign(c[0], dl, ci); i++;
                }
            }
        }
        return -1;
    }

    private List<int> Analyze(int confCi, out int backjumpDl) {
        var learned = new List<int>();
        var seen    = new bool[n+1];
        int counter = 0, idx = trail.Count - 1;
        List<int> rc = clauses[confCi];

        do {
            foreach (int rl in rc) {
                int v = VarOf(rl);
                if (!seen[v]) {
                    seen[v] = true;
                    activity[v] += varInc;
                    if (lvl[v] == dl) counter++;
                    else if (lvl[v] > 0) learned.Add(rl);
                }
            }
            while (!seen[VarOf(trail[idx])]) idx--;
            int uip = trail[idx--];
            int uv  = VarOf(uip);
            seen[uv] = false; counter--;
            if (counter > 0) rc = clauses[reason[uv]];
            else { learned.Insert(0, uip ^ 1); break; }
        } while (true);

        backjumpDl = learned.Skip(1).Select(l => lvl[VarOf(l)]).DefaultIfEmpty(0).Max();
        varInc /= VarDecay;
        for (int v = 1; v <= n; v++) activity[v] *= VarDecay;
        return learned;
    }

    private void Backtrack(int d) {
        while (trail.Count > (d < trailLim.Count ? trailLim[d] : 0)) {
            Unassign(trail[^1]); trail.RemoveAt(trail.Count-1);
        }
        while (trailLim.Count > d) trailLim.RemoveAt(trailLim.Count-1);
        dl = d;
    }

    private int PickVar() {
        int best = -1; double bestAct = -1;
        for (int v = 1; v <= n; v++)
            if (val[v] == 0 && activity[v] > bestAct) { best=v; bestAct=activity[v]; }
        return best;
    }

    public bool Solve() {
        if (Propagate() != -1) return false;
        while (true) {
            int v = PickVar();
            if (v == -1) return true;
            dl++;
            trailLim.Add(trail.Count);
            Assign(phase[v]==1 ? LitPos(v) : LitNeg(v), dl, -1);
            while (true) {
                int conf = Propagate();
                if (conf == -1) break;
                if (dl == 0) return false;
                var learned = Analyze(conf, out int bjDl);
                int ci = clauses.Count;
                clauses.Add(learned);
                Attach(ci);
                Backtrack(bjDl);
                Assign(learned[0], dl, ci);
            }
        }
    }

    public void Print() {
        for (int v = 1; v <= n; v++)
            Console.WriteLine($"x{v} = {(val[v]==1?"true":"false")}");
    }

    public static void Main() {
        var raw = new List<int[]> {
            new[]{1,-2}, new[]{-1,3}, new[]{2,-3}, new[]{-1,-2,-3}
        };
        var s = new CDCL(3, raw);
        if (s.Solve()) { Console.WriteLine("SAT"); s.Print(); }
        else Console.WriteLine("UNSAT");
    }
}
```

---

## CDCL Components and Their Impact

| Component | Without | With | Gain |
|---|---|---|---|
| Watched literals | O(m) per propagation | O(1) amortized | 10–100x BCP speed |
| Clause learning | Exponential restarts | Cut repeated subproblems | Orders of magnitude |
| Non-chron. backjump | Redo all work | Jump to root of conflict | Eliminates redundant search |
| VSIDS | Static order | Conflict-adaptive | 2–10x fewer decisions |
| Phase saving | Restart from scratch | Warm polarity cache | Faster post-restart convergence |
| Restarts | None | Luby/geometric | Escape heavy-tailed distributions |
| LBD deletion | Memory blowup | Bounded learned DB | Sustained cache efficiency |

---

## Pitfalls

- **Watched literal invariant** — after BCP, each clause must have two non-false watched literals (unless the clause is unit or satisfied). Any operation that assigns variables outside BCP (e.g., direct assignment during backtracking) must not break this invariant. Rebuild watch lists after bulk operations.
- **Trail pointer in BCP** — BCP must process only literals assigned since the last BCP call. Restarting from index 0 each time is correct but O(trail) per call. Use a persistent `qhead` pointer advanced incrementally; reset it only on backtrack.
- **Reason clause invalidation** — when learned clauses are deleted, any variable whose `reason` points to a deleted clause must have its reason set to -1 or the clause must be kept. Deleting a reason clause while its variable is still assigned corrupts conflict analysis.
- **First UIP requires exactly one literal at current level** — if the counter in `analyze` reaches 0 before resolving, the UIP is the last variable touched, and the learned clause has exactly one literal at `dl`. Terminating one step early or late produces a clause at the wrong cut, causing incorrect backjump levels.
- **Backjump level computation** — the backjump level is the **second-highest** decision level in the learned clause (highest is `dl`, which belongs to the UIP literal). If all non-UIP literals are at level 0, jump to level 0. Forgetting this case causes jumping to negative levels or level 0 being skipped.
- **VSIDS decay overflow** — `varInc` grows as `1/VarDecay^conflicts`. After millions of conflicts it overflows `double`. Renormalize: when any activity exceeds `1e100`, divide all activities and `varInc` by `1e100`.
- **Phase saving and UNSAT** — on UNSAT instances, phase saving has no effect (every variable is eventually forced). On SAT instances it dramatically reduces post-restart decisions. Do not disable it as a "simplification" — it is responsible for a significant fraction of CDCL's practical performance on satisfiable instances.
- **Clause minimization** — the First-UIP learned clause can often be further shrunk by self-subsuming resolution. Industrial solvers apply recursive minimization, removing literals whose reason clauses are fully contained in `seen`. Omitting this is correct but produces larger learned clauses with higher LBD.

---

## Conclusion

CDCL is the **state of the art in propositional satisfiability** and the engine behind every industrial SAT solver:

- Watched-literal BCP reduces per-propagation cost to O(1) amortized — the single biggest constant-factor win.
- Clause learning with First-UIP cuts entire subtrees of the search space permanently, turning exponential DPLL into near-polynomial behavior on structured instances.
- Non-chronological backjumping eliminates redundant rediscovery of the same conflict from different paths.
- VSIDS with phase saving and periodic restarts ensures the solver escapes heavy-tailed search distributions and focuses branching on conflict-relevant variables.

**Key takeaway:**  
CDCL = DPLL + watched literals + First-UIP clause learning + non-chronological backjumping + VSIDS + restarts + LBD deletion. Each component is independently justified but the combination is superadditive — removing any one of them measurably degrades performance on benchmark suites. For any problem requiring more than a few hundred variables, use an existing CDCL solver (MiniSat, Glucose, CaDiCaL, Kissat) rather than a hand-rolled DPLL.
