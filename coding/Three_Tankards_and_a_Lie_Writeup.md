# Three Tankards and a Lie - Write-up

**Category:** Coding\
**Difficulty:** Easy

------------------------------------------------------------------------

## Description

The challenge presents **N** tankards, each initially holding the item
with the same index. A sequence of **M** swaps is performed, where each
swap exchanges the contents of two tankards.

After all swaps have been applied, we are given **Q** queries. For each
queried starting position, we must determine the final position of the
item that originally started there.

------------------------------------------------------------------------

## Solution

Since the constraints are relatively small (`N ≤ 2000`, `M ≤ 5000`), the
most straightforward approach is to directly simulate every swap.

Initially, each position contains its own item:

``` text
Position : 1 2 3 4 5
Item     : 1 2 3 4 5
```

Maintain an array where:

``` cpp
position[i] = item currently at position i;
```

Initially,

``` cpp
position[i] = i;
```

For every swap `(a, b)`:

``` cpp
swap(position[a], position[b]);
```

Once every swap has been processed, construct the reverse mapping:

``` cpp
answer[item] = final_position;
```

This allows every query to be answered in constant time.

## Algorithm

1.  Read `N`, `M`, and `Q`.
2.  Initialize `position[i] = i`.
3.  Simulate every swap.
4.  Build the inverse mapping: `answer[position[i]] = i;`
5.  Output `answer[p]` for each query.

## Complexity

-   **Time:** `O(N + M + Q)`
-   **Space:** `O(N)`

## C++ Solution

``` cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int N, M, Q;
    cin >> N >> M >> Q;

    vector<int> position(N + 1);

    for (int i = 1; i <= N; i++)
        position[i] = i;

    for (int i = 0; i < M; i++) {
        int a, b;
        cin >> a >> b;
        swap(position[a], position[b]);
    }

    vector<int> answer(N + 1);

    for (int i = 1; i <= N; i++)
        answer[position[i]] = i;

    while (Q--) {
        int p;
        cin >> p;
        cout << answer[p] << '\n';
    }

    return 0;
}
```

## Flag

``` text
HTB{n3v3r_pl4ys_4lw4ys_w4tch3s}
```
## Author: [Nitesh8766](https://github.com/Nitesh8766)
