# Minimax — Game Tree Search Algorithm

## Origin & Motivation

Minimax was formalized by John von Neumann in 1928 as part of game theory, and adapted into a recursive tree search algorithm for combinatorial games by Claude Shannon (1950) and Alan Turing. It is the foundational algorithm for **two-player zero-sum games** with perfect information — games where one player's gain is exactly the other's loss, and both players have full knowledge of the game state.

The algorithm models both players optimally: the **maximizing player** (MAX) tries to maximize the evaluation score, the **minimizing player** (MIN) tries to minimize it. By recursively alternating between these perspectives down to terminal states (or a depth limit), Minimax computes the best achievable outcome under the assumption that both players play perfectly.

Complexity: **O(b^d)** time, **O(b * d)** space, where `b` is the branching factor and `d` is the search depth.

---

## Where It Is Used

- Chess, checkers, Go engines (base algorithm before Alpha-Beta / MCTS)
- Tic-tac-toe, Connect Four, Reversi solvers
- Game AI in strategy games
- Decision theory and adversarial planning
- Combinatorial game theory proofs

---

## Core Idea

A game tree is a tree where:
- **Nodes** represent game states
- **Edges** represent legal moves
- **Leaves** are terminal states (win/loss/draw) or states at maximum depth, evaluated by a **heuristic evaluation function**
- **MAX nodes** choose the child with the highest value
- **MIN nodes** choose the child with the lowest value

```
        MAX
       / | \
      3  5   2
    MIN MIN MIN
   /|\ /|\  /|\
  3 1 2 5 4 2 8 6
```

MAX picks 5 (best among 3, 5, 2 after MIN minimizes each subtree).

---

## Algorithm (Pseudocode)

```
minimax(state, depth, is_maximizing):
    if depth == 0 or is_terminal(state):
        return evaluate(state)

    if is_maximizing:
        best = -INF
        for each move in legal_moves(state):
            child = apply_move(state, move)
            val = minimax(child, depth - 1, false)
            best = max(best, val)
        return best
    else:
        best = +INF
        for each move in legal_moves(state):
            child = apply_move(state, move)
            val = minimax(child, depth - 1, true)
            best = min(best, val)
        return best

// To find the best move from root:
best_move = argmax over moves of minimax(apply(state, move), depth-1, false)
```

---

## Complexity Analysis

| Metric | Bound | Notes |
|---|---|---|
| Time | O(b^d) | Full tree expansion |
| Space | O(b * d) | DFS stack: at most d levels, b siblings each |
| Nodes evaluated | b^d | Reduced to ~b^(d/2) with Alpha-Beta |
| Optimal play guarantee | Yes | Assuming exact evaluation at leaves |

- `b` — branching factor (chess ≈ 35, tic-tac-toe ≈ 9 at root)
- `d` — search depth
- With Alpha-Beta pruning, effective branching factor drops to `sqrt(b)` in the best case, doubling the achievable depth for the same budget

---

## Evaluation Function

The quality of Minimax depends entirely on the **evaluation function** applied at leaf nodes (non-terminal states at max depth):

| Game | Typical Heuristic |
|---|---|
| Chess | Material count + piece-square tables + mobility |
| Tic-tac-toe | +10 win, -10 loss, 0 draw (exact for full search) |
| Connect Four | Count of open three-in-a-rows |
| Reversi | Disk count + corner ownership + mobility |

Terminal states return exact values: `+INF` (MAX wins), `-INF` (MIN wins), `0` (draw).

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// -------------------------------------------------------
// Generic Minimax framework
// -------------------------------------------------------
// State must support:
//   bool   is_terminal()
//   int    evaluate()          // from MAX's perspective, + = good for MAX
//   vector<State> children(bool maximizing)
//   void   print()

// -------------------------------------------------------
// Tic-Tac-Toe implementation
// -------------------------------------------------------
struct TTT {
    // board[i]: 0=empty, 1=X (MAX), -1=O (MIN)
    array<int, 9> board{};
    int turn; // 1=X to move, -1=O to move

    TTT() : turn(1) { board.fill(0); }

    bool is_terminal() const {
        return winner() != 0 || moves_left() == 0;
    }

    int winner() const {
        static const int lines[8][3] = {
            {0,1,2},{3,4,5},{6,7,8},
            {0,3,6},{1,4,7},{2,5,8},
            {0,4,8},{2,4,6}
        };
        for (auto& l : lines) {
            int s = board[l[0]] + board[l[1]] + board[l[2]];
            if (s ==  3) return  1;
            if (s == -3) return -1;
        }
        return 0;
    }

    int moves_left() const {
        return (int)count(board.begin(), board.end(), 0);
    }

    // Evaluate from MAX's perspective
    // Scale by moves_left to prefer faster wins
    int evaluate() const {
        int w = winner();
        if (w ==  1) return 10 + moves_left();   // X wins fast
        if (w == -1) return -10 - moves_left();  // O wins fast
        return 0;                                 // draw
    }

    vector<TTT> children() const {
        vector<TTT> result;
        for (int i = 0; i < 9; i++) {
            if (board[i] == 0) {
                TTT next = *this;
                next.board[i] = turn;
                next.turn = -turn;
                result.push_back(next);
            }
        }
        return result;
    }

    void print() const {
        const char sym[] = {' ', 'X', '.', 'O'};
        // sym[board[i]+1]: -1->O=sym[0], 0->sym[1]=' ', 1->X=sym[2]
        // simpler mapping:
        auto ch = [](int v) { return v==1?'X': v==-1?'O':'.'; };
        for (int r = 0; r < 3; r++) {
            printf(" %c | %c | %c\n",
                ch(board[r*3]), ch(board[r*3+1]), ch(board[r*3+2]));
            if (r < 2) printf("---|---|---\n");
        }
        printf("\n");
    }
};

// -------------------------------------------------------
// Pure Minimax (no pruning)
// Returns the minimax value of the state.
// -------------------------------------------------------
int minimax(const TTT& state, int depth, bool maximizing) {
    if (depth == 0 || state.is_terminal())
        return state.evaluate();

    if (maximizing) {
        int best = INT_MIN;
        for (const auto& child : state.children())
            best = max(best, minimax(child, depth - 1, false));
        return best;
    } else {
        int best = INT_MAX;
        for (const auto& child : state.children())
            best = min(best, minimax(child, depth - 1, true));
        return best;
    }
}

// -------------------------------------------------------
// Find best move for the current player
// -------------------------------------------------------
TTT best_move(const TTT& state) {
    bool maximizing = (state.turn == 1);
    int best_val = maximizing ? INT_MIN : INT_MAX;
    TTT best_child = state;

    for (const auto& child : state.children()) {
        int val = minimax(child, 9, !maximizing);
        if (maximizing ? val > best_val : val < best_val) {
            best_val = val;
            best_child = child;
        }
    }
    return best_child;
}

// -------------------------------------------------------
// Minimax with move tracking (returns value + best move index)
// -------------------------------------------------------
pair<int,int> minimax_with_move(
    const TTT& state, int depth, bool maximizing)
{
    if (depth == 0 || state.is_terminal())
        return {state.evaluate(), -1};

    auto children = state.children();
    int best_val  = maximizing ? INT_MIN : INT_MAX;
    int best_idx  = 0;

    for (int i = 0; i < (int)children.size(); i++) {
        auto [val, _] = minimax_with_move(children[i], depth-1, !maximizing);
        if (maximizing ? val > best_val : val < best_val) {
            best_val = val;
            best_idx = i;
        }
    }
    return {best_val, best_idx};
}

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
int main() {
    TTT game;
    printf("Initial board:\n");
    game.print();

    // Play full game: X (MAX) vs O (MIN), both playing optimally
    while (!game.is_terminal()) {
        bool max_turn = (game.turn == 1);
        printf("%s to move:\n", max_turn ? "X (MAX)" : "O (MIN)");
        game = best_move(game);
        game.print();
    }

    int w = game.winner();
    if      (w ==  1) printf("X wins!\n");
    else if (w == -1) printf("O wins!\n");
    else              printf("Draw!\n");

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

// -------------------------------------------------------
// Tic-Tac-Toe Minimax
// -------------------------------------------------------
public class TTT {
    public readonly int[] Board;   // 0=empty, 1=X (MAX), -1=O (MIN)
    public readonly int Turn;      // 1=X, -1=O

    public TTT() { Board = new int[9]; Turn = 1; }

    private TTT(int[] board, int turn) {
        Board = (int[])board.Clone();
        Turn  = turn;
    }

    public bool IsTerminal() => Winner() != 0 || MovesLeft() == 0;

    public int Winner() {
        int[,] lines = {
            {0,1,2},{3,4,5},{6,7,8},
            {0,3,6},{1,4,7},{2,5,8},
            {0,4,8},{2,4,6}
        };
        for (int l = 0; l < 8; l++) {
            int s = Board[lines[l,0]] + Board[lines[l,1]] + Board[lines[l,2]];
            if (s ==  3) return  1;
            if (s == -3) return -1;
        }
        return 0;
    }

    public int MovesLeft() {
        int c = 0;
        foreach (int v in Board) if (v == 0) c++;
        return c;
    }

    public int Evaluate() {
        int w = Winner();
        if (w ==  1) return 10 + MovesLeft();
        if (w == -1) return -10 - MovesLeft();
        return 0;
    }

    public List<TTT> Children() {
        var result = new List<TTT>();
        for (int i = 0; i < 9; i++) {
            if (Board[i] == 0) {
                var b = (int[])Board.Clone();
                b[i] = Turn;
                result.Add(new TTT(b, -Turn));
            }
        }
        return result;
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

public class Minimax {
    // Pure Minimax — no pruning
    public static int Run(TTT state, int depth, bool maximizing) {
        if (depth == 0 || state.IsTerminal())
            return state.Evaluate();

        if (maximizing) {
            int best = int.MinValue;
            foreach (var child in state.Children())
                best = Math.Max(best, Run(child, depth - 1, false));
            return best;
        } else {
            int best = int.MaxValue;
            foreach (var child in state.Children())
                best = Math.Min(best, Run(child, depth - 1, true));
            return best;
        }
    }

    // Returns best child state for current player
    public static TTT BestMove(TTT state) {
        bool maximizing = state.Turn == 1;
        int  bestVal    = maximizing ? int.MinValue : int.MaxValue;
        TTT  bestChild  = state;

        foreach (var child in state.Children()) {
            int val = Run(child, 9, !maximizing);
            bool better = maximizing ? val > bestVal : val < bestVal;
            if (better) { bestVal = val; bestChild = child; }
        }
        return bestChild;
    }

    public static void Main() {
        var game = new TTT();
        Console.WriteLine("Initial board:");
        game.Print();

        while (!game.IsTerminal()) {
            Console.WriteLine(game.Turn == 1 ? "X (MAX) to move:" : "O (MIN) to move:");
            game = BestMove(game);
            game.Print();
        }

        int w = game.Winner();
        Console.WriteLine(w == 1 ? "X wins!" : w == -1 ? "O wins!" : "Draw!");
    }
}
```

---

## Minimax Extensions

| Extension | Idea | Gain |
|---|---|---|
| Alpha-Beta pruning | Skip branches that cannot affect result | Reduces nodes from O(b^d) to O(b^(d/2)) |
| Iterative deepening | Run depth 1, 2, ... d; use shallow result for move ordering | Better move ordering, anytime behavior |
| Quiescence search | At leaf, keep searching captures/checks to avoid horizon effect | More accurate evaluation |
| Transposition table | Cache evaluated positions by Zobrist hash | Avoid re-evaluating identical states reached by different move orders |
| Aspiration windows | Search with narrow alpha-beta window around expected value | Fewer nodes on average |
| MCTS | Replace tree eval with random rollouts + UCB selection | Handles huge branching factors (Go, general games) |

---

## Pitfalls

- **Horizon effect** — Minimax cuts off at depth `d` and evaluates with a heuristic. If a bad exchange happens just beyond the horizon, the position looks better than it is. Mitigate with quiescence search: continue searching until the position is "quiet" (no captures, checks, or major threats).
- **Evaluation function symmetry** — the evaluation must be consistent: `eval(state)` from MAX's view should equal `-eval(state)` from MIN's view. If the function is not zero-sum consistent, the algorithm will not play optimally even on a correct tree.
- **Integer overflow on INF** — using `INT_MAX` as the initial worst-case value and then doing `INT_MIN - 1` or `-INT_MAX` overflows. Use sentinel values like `1e9` or `INT_MAX / 2`.
- **Mutating state instead of copying** — applying and undoing moves (make/unmake) is more efficient than copying the full state at each node, but errors in unmake logic corrupt the search. Always verify unmake restores the state exactly; use a copy-based approach during development.
- **Turn ambiguity** — the `is_maximizing` flag must correspond to the actual player to move in the game state. If the state encodes the turn internally, ensure the flag and the state's turn field agree at every recursive call.
- **Depth 0 with non-terminal state** — when `depth == 0` and the state is not terminal, the heuristic is called. If the heuristic is not implemented or returns 0 for all non-terminal states, the algorithm degenerates to random play beyond depth 0.
- **No moves available (stalemate)** — if `children()` returns an empty list and the state is not technically terminal (e.g., stalemate in chess), the loop returns `INT_MIN` or `INT_MAX` unchanged. Add an explicit check: if no moves, evaluate as draw or loss depending on game rules.

---

## Conclusion

Minimax is the **theoretical backbone of adversarial game search**:

- It guarantees optimal play for both players in finite two-player zero-sum games with perfect information.
- The algorithm is exact on small games (tic-tac-toe, connect four) and heuristic on large ones (chess, checkers) via depth-limited search with an evaluation function.
- Every practical game engine — from 1950s chess programs to modern engines — is Minimax with Alpha-Beta pruning, move ordering, and a domain-specific evaluation function layered on top.
- For games with very high branching factors (Go, b ≈ 250), Minimax is replaced by Monte Carlo Tree Search, but the maximizing/minimizing logic is preserved in the UCT selection policy.

**Key takeaway:**  
Minimax alone is correct but impractical beyond toy games. The path to a strong engine is: Minimax → Alpha-Beta pruning (free 2x depth) → iterative deepening with move ordering → transposition table → quiescence search → tuned evaluation function. Each step multiplies effective search depth with no algorithmic complexity change.
