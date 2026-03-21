# Alpha–Beta Pruning — Game Search Optimization

## Origin & Motivation

Alpha–Beta pruning was developed independently by several researchers in the late 1950s and formalized by Donald Knuth and Ronald Moore in 1975. It is not a separate algorithm but an **optimization of Minimax** that eliminates subtrees that cannot possibly affect the final decision. The result is identical to Minimax — same best move, same value — but with dramatically fewer nodes evaluated.

The key insight: if MAX has already found a move with value `v`, any MIN node that has already found a response with value `<= v` can be immediately cut — MAX would never allow play to reach that MIN node because a better option already exists. Symmetrically for MIN. This yields **alpha** (the best value MAX is guaranteed) and **beta** (the best value MIN is guaranteed), and pruning fires when `alpha >= beta`.

With optimal move ordering, the number of evaluated nodes drops from **O(b^d)** to **O(b^(d/2))** — effectively doubling the achievable search depth for the same computational budget.

---

## Where It Is Used

- Chess, checkers, reversi engines (core search algorithm)
- Connect Four, tic-tac-toe complete solvers
- Any two-player zero-sum perfect-information game
- Combinatorial game theory search
- Basis for further enhancements: PVS, MTD(f), aspiration windows

---

## Alpha and Beta — Definition

| Variable | Owner | Meaning |
|---|---|---|
| alpha (α) | MAX | Best value MAX is guaranteed from current path upward. Initial: -INF |
| beta (β) | MIN | Best value MIN is guaranteed from current path upward. Initial: +INF |

At any node:
- **Alpha cutoff (β-cutoff):** At a MIN node, if the current value `<= alpha`, MAX will never choose this branch — prune remaining children.
- **Beta cutoff (α-cutoff):** At a MAX node, if the current value `>= beta`, MIN will never allow this branch — prune remaining children.

The pruning condition: `alpha >= beta` — the search window has collapsed, no further exploration is useful.

---

## Algorithm (Pseudocode)

```
alphabeta(state, depth, alpha, beta, maximizing):
    if depth == 0 or is_terminal(state):
        return evaluate(state)

    if maximizing:
        value = -INF
        for each move in legal_moves(state):
            child = apply_move(state, move)
            value = max(value, alphabeta(child, depth-1, alpha, beta, false))
            alpha = max(alpha, value)
            if alpha >= beta:
                break              // beta cutoff — MIN won't allow this
        return value
    else:
        value = +INF
        for each move in legal_moves(state):
            child = apply_move(state, move)
            value = min(value, alphabeta(child, depth-1, alpha, beta, true))
            beta = min(beta, value)
            if alpha >= beta:
                break              // alpha cutoff — MAX won't allow this
        return value

// Root call:
alphabeta(root, depth, -INF, +INF, true)
```

---

## Move Ordering Impact

The number of pruned nodes depends entirely on move ordering quality:

| Ordering Quality | Nodes Evaluated | Effective Branching | Depth Gain |
|---|---|---|---|
| Worst (reverse-optimal) | O(b^d) | b | 0 |
| Random | O(b^(3d/4)) | b^(3/4) | ~33% |
| Good (heuristic) | O(b^(d/2+ε)) | ~sqrt(b) | ~2x |
| Perfect (always best first) | O(b^(d/2)) | sqrt(b) | 2x |

**Move ordering heuristics:**
- **Killer moves** — moves that caused beta cutoffs at the same depth in sibling subtrees
- **History heuristic** — moves that historically caused cutoffs, accumulated across the search
- **MVV-LVA** (chess) — order captures by Most Valuable Victim / Least Valuable Attacker
- **Transposition table** — use previously computed best moves from identical positions
- **Static evaluation** — order moves by a fast heuristic before recursing

---

## Complexity Analysis

| Scenario | Nodes | Notes |
|---|---|---|
| No pruning (Minimax) | O(b^d) | Baseline |
| Worst-case ordering | O(b^d) | Same as Minimax |
| Random ordering | O(b^(3d/4)) | Average case |
| Perfect ordering | O(b^(d/2)) | Best case — sqrt(b) effective branching |
| Space | O(b * d) | DFS stack, same as Minimax |

For chess (b≈35, d=10): Minimax evaluates ~35^10 ≈ 2.76×10^15 nodes. Alpha-Beta with good ordering evaluates ~35^5 ≈ 5.25×10^7 — a factor of ~5×10^7 reduction.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// -------------------------------------------------------
// Tic-Tac-Toe state (same as Minimax file, abbreviated)
// -------------------------------------------------------
struct TTT {
    array<int,9> board{};
    int turn; // 1=X (MAX), -1=O (MIN)

    TTT() : turn(1) { board.fill(0); }

    int winner() const {
        static const int L[8][3]={{0,1,2},{3,4,5},{6,7,8},
                                   {0,3,6},{1,4,7},{2,5,8},
                                   {0,4,8},{2,4,6}};
        for (auto& l : L) {
            int s = board[l[0]]+board[l[1]]+board[l[2]];
            if (s== 3) return  1;
            if (s==-3) return -1;
        }
        return 0;
    }

    int moves_left() const { return (int)count(board.begin(),board.end(),0); }
    bool is_terminal() const { return winner()!=0 || moves_left()==0; }

    int evaluate() const {
        int w = winner();
        if (w== 1) return 10 + moves_left();
        if (w==-1) return -10 - moves_left();
        return 0;
    }

    void apply(int pos) { board[pos] = turn; turn = -turn; }
    void undo (int pos) { turn = -turn; board[pos] = 0; }

    vector<int> legal_moves() const {
        vector<int> m;
        for (int i=0;i<9;i++) if (board[i]==0) m.push_back(i);
        return m;
    }

    void print() const {
        auto ch=[](int v){return v==1?'X':v==-1?'O':'.';};
        for (int r=0;r<3;r++){
            printf(" %c | %c | %c\n",ch(board[r*3]),ch(board[r*3+1]),ch(board[r*3+2]));
            if(r<2) printf("---|---|---\n");
        }
        printf("\n");
    }
};

// -------------------------------------------------------
// Alpha-Beta (fail-soft variant)
// Returns the true minimax value; prunes subtrees that
// cannot improve upon the current alpha/beta window.
// -------------------------------------------------------
int alphabeta(TTT& state, int depth, int alpha, int beta, bool maximizing) {
    if (depth == 0 || state.is_terminal())
        return state.evaluate();

    auto moves = state.legal_moves();

    if (maximizing) {
        int value = INT_MIN;
        for (int m : moves) {
            state.apply(m);
            value = max(value, alphabeta(state, depth-1, alpha, beta, false));
            state.undo(m);

            alpha = max(alpha, value);
            if (alpha >= beta) break;   // beta cutoff
        }
        return value;
    } else {
        int value = INT_MAX;
        for (int m : moves) {
            state.apply(m);
            value = min(value, alphabeta(state, depth-1, alpha, beta, true));
            state.undo(m);

            beta = min(beta, value);
            if (alpha >= beta) break;   // alpha cutoff
        }
        return value;
    }
}

// -------------------------------------------------------
// Find best move using Alpha-Beta
// -------------------------------------------------------
int best_move(TTT& state) {
    bool maximizing = (state.turn == 1);
    int best_val = maximizing ? INT_MIN : INT_MAX;
    int best_pos = -1;

    for (int m : state.legal_moves()) {
        state.apply(m);
        int val = alphabeta(state, 9, INT_MIN, INT_MAX, !maximizing);
        state.undo(m);

        bool better = maximizing ? val > best_val : val < best_val;
        if (better) { best_val = val; best_pos = m; }
    }
    return best_pos;
}

// -------------------------------------------------------
// Negamax formulation (cleaner, equivalent to Alpha-Beta)
// Both players try to MAXIMIZE their own score.
// evaluate() must return score from the CURRENT player's view.
// -------------------------------------------------------
int negamax(TTT& state, int depth, int alpha, int beta) {
    if (depth == 0 || state.is_terminal())
        return state.turn * state.evaluate(); // flip sign for current player

    int value = INT_MIN;
    for (int m : state.legal_moves()) {
        state.apply(m);
        value = max(value, -negamax(state, depth-1, -beta, -alpha));
        state.undo(m);

        alpha = max(alpha, value);
        if (alpha >= beta) break;
    }
    return value;
}

// -------------------------------------------------------
// Alpha-Beta with move ordering (killer + history heuristic)
// -------------------------------------------------------
struct ABWithOrdering {
    // killer[depth] = move that caused beta cutoff at this depth
    array<int, 16> killer{};
    array<int, 9>  history{};

    ABWithOrdering() { killer.fill(-1); history.fill(0); }

    // Order moves: killer first, then by history score
    vector<int> order_moves(const TTT& state, int depth) {
        auto moves = state.legal_moves();
        sort(moves.begin(), moves.end(), [&](int a, int b){
            if (a == killer[depth]) return true;
            if (b == killer[depth]) return false;
            return history[a] > history[b];
        });
        return moves;
    }

    int search(TTT& state, int depth, int alpha, int beta, bool maximizing) {
        if (depth == 0 || state.is_terminal())
            return state.evaluate();

        auto moves = order_moves(state, depth);

        if (maximizing) {
            int value = INT_MIN;
            for (int m : moves) {
                state.apply(m);
                int v = search(state, depth-1, alpha, beta, false);
                state.undo(m);

                if (v > value) value = v;
                if (v > alpha) alpha = v;
                if (alpha >= beta) {
                    killer[depth] = m;      // record killer
                    history[m]++;           // bump history
                    break;
                }
            }
            return value;
        } else {
            int value = INT_MAX;
            for (int m : moves) {
                state.apply(m);
                int v = search(state, depth-1, alpha, beta, true);
                state.undo(m);

                if (v < value) value = v;
                if (v < beta) beta = v;
                if (alpha >= beta) {
                    killer[depth] = m;
                    history[m]++;
                    break;
                }
            }
            return value;
        }
    }
};

// -------------------------------------------------------
// Iterative Deepening Alpha-Beta (IDDFS + AB)
// Runs AB at depth 1, 2, ..., max_depth.
// Uses results from shallower searches to order moves.
// -------------------------------------------------------
int iterative_deepening(TTT& state, int max_depth) {
    int best_pos = -1;
    bool maximizing = (state.turn == 1);

    for (int depth = 1; depth <= max_depth; depth++) {
        int best_val = maximizing ? INT_MIN : INT_MAX;
        for (int m : state.legal_moves()) {
            state.apply(m);
            int val = alphabeta(state, depth-1, INT_MIN, INT_MAX, !maximizing);
            state.undo(m);

            bool better = maximizing ? val > best_val : val < best_val;
            if (better) { best_val = val; best_pos = m; }
        }
    }
    return best_pos;
}

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
int main() {
    TTT game;
    printf("Initial board:\n");
    game.print();

    while (!game.is_terminal()) {
        printf("%s to move:\n", game.turn==1?"X (MAX)":"O (MIN)");
        int pos = best_move(game);
        game.apply(pos);
        game.print();
    }

    int w = game.winner();
    if      (w== 1) printf("X wins!\n");
    else if (w==-1) printf("O wins!\n");
    else            printf("Draw!\n");

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class TTT {
    public int[] Board;
    public int   Turn;  // 1=X (MAX), -1=O (MIN)

    public TTT() { Board = new int[9]; Turn = 1; }
    public TTT(TTT o) { Board = (int[])o.Board.Clone(); Turn = o.Turn; }

    public int Winner() {
        int[,] L = { {0,1,2},{3,4,5},{6,7,8},{0,3,6},{1,4,7},{2,5,8},{0,4,8},{2,4,6} };
        for (int i = 0; i < 8; i++) {
            int s = Board[L[i,0]] + Board[L[i,1]] + Board[L[i,2]];
            if (s ==  3) return  1;
            if (s == -3) return -1;
        }
        return 0;
    }

    public int MovesLeft() => Board.Count(v => v == 0);
    public bool IsTerminal() => Winner() != 0 || MovesLeft() == 0;

    public int Evaluate() {
        int w = Winner();
        if (w ==  1) return 10 + MovesLeft();
        if (w == -1) return -10 - MovesLeft();
        return 0;
    }

    public void Apply(int pos) { Board[pos] = Turn; Turn = -Turn; }
    public void Undo (int pos) { Turn = -Turn; Board[pos] = 0; }

    public List<int> LegalMoves() {
        var m = new List<int>();
        for (int i = 0; i < 9; i++) if (Board[i] == 0) m.Add(i);
        return m;
    }

    public void Print() {
        char Ch(int v) => v == 1 ? 'X' : v == -1 ? 'O' : '.';
        for (int r = 0; r < 3; r++) {
            Console.WriteLine($" {Ch(Board[r*3])} | {Ch(Board[r*3+1])} | {Ch(Board[r*3+2])}");
            if (r < 2) Console.WriteLine("---|---|---");
        }
        Console.WriteLine();
    }
}

public class AlphaBeta {
    // -------------------------------------------------------
    // Fail-soft Alpha-Beta
    // -------------------------------------------------------
    public static int Search(TTT state, int depth, int alpha, int beta, bool maximizing) {
        if (depth == 0 || state.IsTerminal())
            return state.Evaluate();

        if (maximizing) {
            int value = int.MinValue;
            foreach (int m in state.LegalMoves()) {
                state.Apply(m);
                value = Math.Max(value, Search(state, depth-1, alpha, beta, false));
                state.Undo(m);
                alpha = Math.Max(alpha, value);
                if (alpha >= beta) break;   // beta cutoff
            }
            return value;
        } else {
            int value = int.MaxValue;
            foreach (int m in state.LegalMoves()) {
                state.Apply(m);
                value = Math.Min(value, Search(state, depth-1, alpha, beta, true));
                state.Undo(m);
                beta = Math.Min(beta, value);
                if (alpha >= beta) break;   // alpha cutoff
            }
            return value;
        }
    }

    // -------------------------------------------------------
    // Negamax Alpha-Beta (cleaner formulation)
    // evaluate() must return score from CURRENT player's view.
    // -------------------------------------------------------
    public static int Negamax(TTT state, int depth, int alpha, int beta) {
        if (depth == 0 || state.IsTerminal())
            return state.Turn * state.Evaluate();

        int value = int.MinValue;
        foreach (int m in state.LegalMoves()) {
            state.Apply(m);
            value = Math.Max(value, -Negamax(state, depth-1, -beta, -alpha));
            state.Undo(m);
            alpha = Math.Max(alpha, value);
            if (alpha >= beta) break;
        }
        return value;
    }

    // Best move selection
    public static int BestMove(TTT state) {
        bool maximizing = state.Turn == 1;
        int bestVal = maximizing ? int.MinValue : int.MaxValue;
        int bestPos = -1;

        foreach (int m in state.LegalMoves()) {
            state.Apply(m);
            int val = Search(state, 9, int.MinValue, int.MaxValue, !maximizing);
            state.Undo(m);

            bool better = maximizing ? val > bestVal : val < bestVal;
            if (better) { bestVal = val; bestPos = m; }
        }
        return bestPos;
    }

    // -------------------------------------------------------
    // Alpha-Beta with killer + history heuristic move ordering
    // -------------------------------------------------------
    private readonly int[]  killer  = new int[16];
    private readonly int[]  history = new int[9];

    public AlphaBeta() { Array.Fill(killer, -1); }

    private List<int> OrderMoves(TTT state, int depth) {
        var moves = state.LegalMoves();
        moves.Sort((a, b) => {
            if (a == killer[depth]) return -1;
            if (b == killer[depth]) return  1;
            return history[b].CompareTo(history[a]);
        });
        return moves;
    }

    public int SearchOrdered(TTT state, int depth, int alpha, int beta, bool maximizing) {
        if (depth == 0 || state.IsTerminal())
            return state.Evaluate();

        var moves = OrderMoves(state, depth);

        if (maximizing) {
            int value = int.MinValue;
            foreach (int m in moves) {
                state.Apply(m);
                int v = SearchOrdered(state, depth-1, alpha, beta, false);
                state.Undo(m);
                if (v > value) value = v;
                if (v > alpha) alpha = v;
                if (alpha >= beta) { killer[depth] = m; history[m]++; break; }
            }
            return value;
        } else {
            int value = int.MaxValue;
            foreach (int m in moves) {
                state.Apply(m);
                int v = SearchOrdered(state, depth-1, alpha, beta, true);
                state.Undo(m);
                if (v < value) value = v;
                if (v < beta)  beta  = v;
                if (alpha >= beta) { killer[depth] = m; history[m]++; break; }
            }
            return value;
        }
    }

    public static void Main() {
        var game = new TTT();
        Console.WriteLine("Initial board:");
        game.Print();

        while (!game.IsTerminal()) {
            Console.WriteLine(game.Turn == 1 ? "X (MAX) to move:" : "O (MIN) to move:");
            int pos = BestMove(game);
            game.Apply(pos);
            game.Print();
        }

        int w = game.Winner();
        Console.WriteLine(w == 1 ? "X wins!" : w == -1 ? "O wins!" : "Draw!");
    }
}
```

---

## Fail-Hard vs Fail-Soft

Two standard Alpha-Beta variants differ in what they return when pruning occurs:

| Variant | Returns when pruned | Use case |
|---|---|---|
| Fail-hard | Exactly alpha or beta (the bound) | Simpler; used with aspiration windows |
| Fail-soft | The best value found (may exceed window) | More info; enables PVS and MTD(f) |

```
// Fail-hard MAX node (returns alpha on cutoff):
if (value >= beta) return beta;    // hard bound
alpha = max(alpha, value);

// Fail-soft MAX node (returns actual best value):
if (value >= beta) return value;   // may exceed beta
alpha = max(alpha, value);
```

---

## Advanced Variants Built on Alpha-Beta

| Variant | Key idea | Gain over plain Alpha-Beta |
|---|---|---|
| PVS (Principal Variation Search) | Search first move with full window, rest with null window | ~10-15% fewer nodes |
| MTD(f) | Binary search over the value using zero-width windows | Optimal node count for given value |
| Aspiration windows | Start with narrow [alpha, beta] around expected value; re-search on failure | Fewer nodes on average |
| Quiescence search | Continue search at leaf until position is "quiet" | Eliminates horizon effect |
| Late Move Reductions (LMR) | Reduce depth for moves ordered late (likely bad) | 2-4x speedup in practice |
| Null-move pruning | Let the opponent play twice; if they can't improve, prune | Large practical speedup |

---

## Pitfalls

- **Passing alpha/beta by value is required** — each recursive call must receive the current window but must not retroactively modify the caller's alpha/beta through a reference. In languages with pass-by-reference, accidentally sharing alpha/beta across siblings corrupts the window.
- **INT_MIN negation overflows** — in Negamax, `-negamax(...)` is applied at every call. If the function returns `INT_MIN`, negating it overflows in C++ and C#. Use `INT_MIN/2` and `INT_MAX/2` as sentinel values, or a dedicated `NEG_INF = -1e9`.
- **Make/undo asymmetry** — using in-place make/undo instead of state copies is faster but requires an exact inverse. Any piece of state not restored (e.g., a zobrist hash, a move counter, castling rights in chess) silently corrupts all subsequent evaluations.
- **Alpha-Beta does not prune the first child** — the first child is always fully searched with the full window. Pruning only fires from the second child onward. If the first child is the worst move, no pruning occurs in the entire subtree. Move ordering quality for the first move is therefore critical.
- **Killer moves are depth-specific** — a killer at depth `d` is a move index, not a full move description. In games where the same index can represent different moves at different board positions (e.g., piece-type moves in chess), use (from, to) pairs and validate that the killer is legal before trying it.
- **Transposition table interaction** — storing Alpha-Beta results in a transposition table requires saving the **bound type** (exact, lower bound, upper bound) alongside the value, because the stored value may have been produced under a different window. Applying an exact value from a different window as if it were exact corrupts the search.
- **Asymmetric evaluation** — the evaluation function must satisfy `eval(state, MAX) = -eval(state, MIN)` for Negamax to be correct. If the function uses absolute piece counts without sign-flipping, Negamax computes wrong values at MIN nodes.

---

## Conclusion

Alpha–Beta Pruning is the **single most important optimization in adversarial game search**:

- It is a provably correct optimization of Minimax — produces identical results with far fewer nodes.
- With good move ordering it reduces effective branching factor from `b` to `sqrt(b)`, doubling the search depth for the same time budget.
- Negamax is the cleanest formulation — both players maximize their own negated-opponent score, eliminating the maximizing/minimizing flag and making extensions like PVS trivial to implement.
- Every competitive game engine in existence (Stockfish, Leela Chess Zero's classical backend, Komodo, AlphaZero's MCTS policy backup) uses Alpha-Beta or a direct descendant of it as its core tree search.

**Key takeaway:**  
Implement Alpha-Beta in Negamax form with in-place make/undo, iterative deepening, and a transposition table. Add move ordering via killer moves and history heuristic. This alone produces a competitive engine for most combinatorial games. PVS, LMR, and null-move pruning are the next layer for serious engines — all of them are Alpha-Beta with a narrowed window or reduced depth, not fundamentally different algorithms.
