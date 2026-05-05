## The Nature of DP: Recursive & Recurrence Relations

All problems that can be solved with Dynamic Programming (DP) are **inherently recursive**.

This is because a problem must have two properties to be a DP problem:
1.  **Optimal Substructure**: The optimal solution to a big problem can be found from the optimal solutions of its smaller subproblems.
2.  **Overlapping Subproblems**: The same small subproblems are solved over and over again.

The relationship between the big problem and its smaller subproblems is defined by a **recurrence relation**. This relation is the mathematical expression of the problem's recursive structure.

So, if a problem is a DP problem, it must have a recurrence relation.

---

## Top-Down vs. Bottom-Up

### Can Bottom-Up Always Be Used?

Theoretically, **almost always**. In practice, **not always**.

The Bottom-Up (Tabulation) approach is generally faster because it has no recursion overhead. However, it requires a **clear and predictable order** for solving subproblems. You must know the path from the smallest problem to the final goal.

### When Bottom-Up is Difficult

Top-Down (Memoization) is a better choice in these cases:

1.  **Sparse State Space**: The total number of subproblems is huge, but you only need to solve a few of them to get the final answer. Bottom-up would solve everything, including many unnecessary subproblems. Top-down only solves what's needed.
2.  **Complex State Transitions**: The dependencies between subproblems are not simple and linear (e.g., `dp[i]` depends on `dp[i-1]`). If `dp[i]` depends on something like `dp[i/2]` or `dp[log(i)]`, figuring out the `for` loop order for a bottom-up approach can be very difficult. Top-down handles this naturally with recursion.

---

## Terminology: Memoization vs. Tabulation

Both methods store and reuse calculated results. The different names are used to describe two distinct **approaches**.

### Memoization (Used in Top-Down)

* **What it is**: "Remembering" the results of function calls. The name comes from "memo".
* **How it works**: It's a **lazy, on-demand** approach. You start with the main problem and use recursion to call subproblems. If a subproblem has already been solved, you return the stored (memoized) value. If not, you compute it, store it, and then return it.
* **Core idea**: `Recursive calls + Caching`



### Tabulation (Used in Bottom-Up)

* **What it is**: "Filling a table". The name comes from "table".
* **How it works**: It's an **eager, systematic** approach. You start with the smallest possible subproblems (the base cases). Then you use loops to solve bigger and bigger problems in order, filling a table (`dp` array) until you reach the final answer.
* **Core idea**: `Iterative loops + Table filling`