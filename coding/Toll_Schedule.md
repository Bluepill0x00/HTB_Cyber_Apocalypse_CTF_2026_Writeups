# Toll Schedule - Write-up

**Category:** Coding  
**Difficulty:** Medium

---

## Challenge Description

The challenge provides:

- `N` convoy arrival times.
- `G` checkpoint clearance opening times (`G ≥ N`).

Each convoy must be assigned to exactly one clearance that opens **at or after** its arrival time. Every clearance can only be used once.

The goal is to minimize the **total waiting time**, where:

```
waiting time = clearance opening time - arrival time
```

---

## Solution

Since the assignment must preserve the ordering of arrivals and clearances to achieve the minimum waiting time, the first step is to sort both arrays.

A greedy approach is not sufficient because choosing an early clearance for one convoy may negatively affect later assignments.

Instead, we use **Dynamic Programming**.

Let:

```
dp[i][j]
```

represent the minimum total waiting time for assigning the first `i` convoys using the first `j` clearances.

For every clearance, we have two choices:

1. Skip the current clearance.
2. Assign it to the current convoy (only if its opening time is not earlier than the convoy's arrival).

The transitions are:

```
dp[i][j] = dp[i][j-1]
```

or

```
dp[i][j] = dp[i-1][j-1] + (clearance[j] - arrival[i])
```

whenever the assignment is valid.

The answer is stored in:

```
dp[N][G]
```

---

## Algorithm

1. Read the arrival times.
2. Read the clearance opening times.
3. Sort both arrays.
4. Initialize the DP table.
5. Process every convoy and every clearance.
6. Output the minimum waiting time.

---

## Complexity Analysis

### Time Complexity

```
O(N × G)
```

Sorting requires

```
O(N log N + G log G)
```

which is dominated by the DP.

### Space Complexity

```
O(N × G)
```

---

## C++ Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long INF = 1e18;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int N, G;
    cin >> N >> G;

    vector<int> arrival(N + 1), gate(G + 1);

    for (int i = 1; i <= N; i++)
        cin >> arrival[i];

    for (int i = 1; i <= G; i++)
        cin >> gate[i];

    sort(arrival.begin() + 1, arrival.end());
    sort(gate.begin() + 1, gate.end());

    vector<vector<long long>> dp(N + 1, vector<long long>(G + 1, INF));

    for (int j = 0; j <= G; j++)
        dp[0][j] = 0;

    for (int i = 1; i <= N; i++) {
        for (int j = 1; j <= G; j++) {

            dp[i][j] = dp[i][j - 1];

            if (gate[j] >= arrival[i] && dp[i - 1][j - 1] != INF) {
                dp[i][j] = min(dp[i][j],
                               dp[i - 1][j - 1] + (gate[j] - arrival[i]));
            }
        }
    }

    cout << dp[N][G] << '\n';

    return 0;
}
```

---

## Result

The dynamic programming approach evaluates every valid assignment while ensuring the minimum possible total waiting time. Sorting the arrivals and clearances simplifies the state transitions and guarantees an optimal solution.

### Flag

```text
HTB{th3_p4ss_pr3f3rs_0n3_b4nn3r}
```

---

## Conclusion

This challenge demonstrates a classic dynamic programming optimization problem. By considering whether to use or skip each clearance, the algorithm efficiently computes the minimum total waiting time while satisfying all assignment constraints. The solution runs comfortably within the given limits and produces the optimal answer.

## Author: [Nitesh8766](https://github.com/Nitesh8766)
