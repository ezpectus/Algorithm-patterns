# 🧠 Greedy — Algorithm Template

## 🔍 Introduction

Greedy is a strategy where you make the **locally optimal choice** at each step, hoping it leads to a **globally optimal solution**.

**Core idea**:  
> "Choose the best option now — if you can prove it won’t break the overall goal."

Greedy works when you can define an **invariant** that guarantees correctness after each decision.

---

## 🧠 When to Apply Greedy

Use greedy when:

- ✅ There's a clear optimization metric (time, cost, length, count…)
- ✅ Local decisions can be formalized
- ✅ You can prove greedy won't break the global goal
- ❌ You don’t need to track multiple states (like in DP)
- ❌ Future decisions don’t depend on current ones

---

## 💡 Common Applications

- **Interval scheduling** (meeting rooms, coverage)
- **Sort + select** (activity selection, task assignment)
- **Monotonic structures** (stack/queue)
- **Packing & partitioning** (coins, containers)
- **Removal/replacement** (lex smallest string, cost minimization)


## 🧱 Greedy Template — Core Structure
```csharp
// 1. Sort input if needed
Array.Sort(data, customComparer);

// 2. Iterate and make greedy choices
foreach (var item in data) {
    if (IsValidGreedyChoice(item)) {
        ApplyChoice(item);
    }
}
```

## 🧩 Greedy Patterns

| Pattern              | Example Problems                          | Key Idea                     |
|----------------------|-------------------------------------------|------------------------------|
| Sort + Select        | Activity Selection, Merge Intervals       | Sort by start/end time       |
| Stack-based Greedy   | Next Greater Element, Histogram           | Monotonic stack              |
| Greedy Removal       | Remove K Digits, Lex Smallest             | Remove by priority           |
| Greedy Packing       | Coin Change (greedy version), Tasks       | Pick largest/smallest first |
| Greedy Expansion     | Jump Game, Gas Station                    | Expand reachable boundary    |

## ✅ How to Validate Greedy

Before applying greedy, ask:

- Invariant: What must remain true after each step?
- Counterexample: Can you build a case where greedy fails?
- Proof: Why does greedy work? Via sorting, monotonicity, or impossibility of improvement?

## ❌ Common Mistakes: 

- ❌ Using greedy without proof → incorrect results
- ❌ Wrong sorting order → breaks invariant
- ❌ Ignoring edge cases → incomplete coverage
- ❌ Using greedy instead of DP → when state dependencies exist

## 🧠 Memorization Tips

- Greedy = Local choice + Invariant
- Always ask: “Can I prove greedy is optimal?”
- Sorting is often the key
- Stack/queue help enforce monotonicity
- If unsure — build a counterexample

## 🏗️ Greedy Training Progression

- ✅ Sort + Select (Activity Selection, Intervals) 
- ✅ Stack-based greedy (Next Greater Element, Histogram) 
- ✅ Greedy removal (Remove K Digits, Lex Smallest) 
- ✅ Greedy expansion (Jump Game, Gas Station) 
- ✅ Greedy with proof (Scheduling, Huffman Coding)

## 💡 Full Code Examples (C#)

## 1. Activity Selection (Sort + Select)

```csharp
public int MaxMeetings(int[][] intervals) {
    Array.Sort(intervals, (a, b) => a[1].CompareTo(b[1]));
    int count = 0, end = -1;

    foreach (var meeting in intervals) {
        if (meeting[0] > end) {
            count++;
            end = meeting[1];
        }
    }
    return count;
}
```

## 2. Remove K Digits (Greedy Removal with Stack)

```csharp
public string RemoveKDigits(string num, int k) {
    var stack = new Stack<char>();

    foreach (char digit in num) {
        while (stack.Count > 0 && k > 0 && stack.Peek() > digit) {
            stack.Pop();
            k--;
        }
        stack.Push(digit);
    }

    while (k-- > 0) stack.Pop();

    var result = new string(stack.Reverse().ToArray()).TrimStart('0');
    return result == "" ? "0" : result;
}
```

## 3. Jump Game (Greedy Expansion)

```csharp
public bool CanJump(int[] nums) {
    int maxReach = 0;

    for (int i = 0; i < nums.Length; i++) {
        if (i > maxReach) return false;
        maxReach = Math.Max(maxReach, i + nums[i]);
    }

    return true;
}
```

## 4. Gas Station (Greedy with Circular Invariant)

```csharp
public int CanCompleteCircuit(int[] gas, int[] cost) {
    int total = 0, curr = 0, start = 0;

    for (int i = 0; i < gas.Length; i++) {
        int diff = gas[i] - cost[i];
        total += diff;
        curr += diff;

        if (curr < 0) {
            start = i + 1;
            curr = 0;
        }
    }

    return total < 0 ? -1 : start;
}
```


## 🎯 Conclusion
Greedy is not just "grab the best" — it's a strategic architecture built on invariants and proof.
It gives fast, elegant solutions when you:

- Identify local optima
- Validate global correctness
- Build reusable patterns

Mastering greedy = mastering invariants. 
Once internalized, it becomes your go-to tool for clarity, speed, and elegance in algorithmic design.
Global correctness will be checked
Constructs proofs or counterexamples
The greedy mastery is the mastery of invariants. When you master it, you don't just solve problems, you build algorithms.




----
