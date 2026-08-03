# Ash Record - Write-up

**Category:** Coding  
**Difficulty:** Medium

---

## Challenge Description

The challenge provides a list of recovered residues, where each residue consists of:

- A timestamp
- A material type

It also provides a suspected extraction sequence of material types.

The objective is to determine the **longest prefix** of the suspected sequence that can be matched as a subsequence of the recovered residues while satisfying the following conditions:

- Residues must appear in chronological order.
- Consecutive matched residues must be at least `min_gap` units apart in time.

The answer is the maximum number of consecutive steps from the beginning of the sequence that can be confirmed.

---

## Solution

Since the extraction sequence length is very small (`P ≤ 12`), dynamic programming is an ideal approach.

First, sort all recovered residues by timestamp.

Define:

```
dp[i]
```

where `dp[i]` stores the earliest possible timestamp at which the first `i + 1` elements of the sequence can be matched.

Initially, every entry is marked as unreachable.

For every residue in chronological order:

- If it matches the first sequence element, update `dp[0]`.
- Otherwise, process the DP array backwards.
- If the residue matches the current sequence element and the minimum time gap is satisfied, update the earliest timestamp for that state.

Processing backwards prevents a residue from being reused multiple times during the same iteration.

Finally, count how many entries in the DP array are reachable.

---

## Algorithm

1. Read the extraction sequence.
2. Read all residues.
3. Sort residues by timestamp.
4. Initialize the DP array.
5. Iterate through every residue.
6. Update the DP states in reverse order.
7. Count the longest reachable prefix.

---

## Complexity Analysis

### Time Complexity

```
O(N log N + N × P)
```

where:

- `N ≤ 5000`
- `P ≤ 12`

Since `P` is extremely small, the algorithm is effectively linear after sorting.

### Space Complexity

```
O(P)
```

---

## C++ Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int N, P, min_gap;
    cin >> N >> P >> min_gap;

    vector<string> pattern(P);

    for (int i = 0; i < P; i++)
        cin >> pattern[i];

    vector<pair<int, string>> residues(N);

    for (int i = 0; i < N; i++)
        cin >> residues[i].first >> residues[i].second;

    sort(residues.begin(), residues.end());

    vector<int> dp(P, INF);

    for (auto &r : residues) {
        int t = r.first;
        string type = r.second;

        for (int i = P - 1; i >= 0; i--) {

            if (type != pattern[i])
                continue;

            if (i == 0) {
                dp[0] = min(dp[0], t);
            }
            else if (dp[i - 1] != INF &&
                     t - dp[i - 1] >= min_gap) {
                dp[i] = min(dp[i], t);
            }
        }
    }

    int ans = 0;

    while (ans < P && dp[ans] != INF)
        ans++;

    cout << ans << '\n';

    return 0;
}
```

---

## Result

The solution successfully determines the longest prefix of the extraction sequence that can be confirmed while respecting the required minimum time gap. By storing only the earliest valid timestamp for each prefix length, the dynamic programming approach remains both efficient and easy to implement.

### Flag

```text
HTB{th3_h4ml3t_w4s_k3pt_n0t_burn3d}
```

---

## Conclusion

This challenge combines chronological ordering with subsequence matching under additional constraints. Sorting the residues and using a compact dynamic programming approach allows each residue to be processed efficiently, resulting in an optimal solution that comfortably satisfies the problem limits.

## Author: [Nitesh8766](https://github.com/Nitesh8766)
