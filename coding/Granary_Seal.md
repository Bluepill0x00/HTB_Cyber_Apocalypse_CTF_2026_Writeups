# Granary Seal - Write-up

**Category:** Coding  
**Difficulty:** Easy

---

## Challenge Description

The challenge provides three custody rolls:

- Clerks
- Countersigners
- Couriers

Each order consists of one name from each role. An order is considered **valid** only if:

- The clerk exists in the clerk custody roll.
- The countersigner exists in the countersigner custody roll.
- The courier exists in the courier custody roll.

The objective is to count how many orders satisfy all three conditions.

---

## Solution

Since the only operation required is checking whether a name exists in a specific custody roll, the ideal data structure is an `unordered_set`.

Three hash sets are created:

- `clerks`
- `countersigners`
- `couriers`

Each name from the corresponding custody roll is inserted into its set.

For every order, we simply check whether all three names exist in their respective sets.

If they do, we increment the answer.

---

## Algorithm

1. Read the clerk list into an `unordered_set`.
2. Read the countersigner list into another `unordered_set`.
3. Read the courier list into a third `unordered_set`.
4. For every order:
   - Check whether the clerk exists.
   - Check whether the countersigner exists.
   - Check whether the courier exists.
5. Count every order where all three checks succeed.

---

## Complexity Analysis

### Time Complexity

```
O(C + CS + R + N)
```

where:

- `C` = number of clerks
- `CS` = number of countersigners
- `R` = number of couriers
- `N` = number of orders

Each lookup in an `unordered_set` is **O(1)** on average.

### Space Complexity

```
O(C + CS + R)
```

---

## C++ Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int C;
    cin >> C;

    unordered_set<string> clerks;

    for (int i = 0; i < C; i++) {
        string s;
        cin >> s;
        clerks.insert(s);
    }

    int CS;
    cin >> CS;

    unordered_set<string> countersigners;

    for (int i = 0; i < CS; i++) {
        string s;
        cin >> s;
        countersigners.insert(s);
    }

    int R;
    cin >> R;

    unordered_set<string> couriers;

    for (int i = 0; i < R; i++) {
        string s;
        cin >> s;
        couriers.insert(s);
    }

    int N;
    cin >> N;

    int answer = 0;

    while (N--) {
        string clerk, counter, courier;
        cin >> clerk >> counter >> courier;

        if (clerks.count(clerk) &&
            countersigners.count(counter) &&
            couriers.count(courier))
            answer++;
    }

    cout << answer << '\n';

    return 0;
}
```

---

## Result

The solution correctly validates every order by verifying that each participant belongs to the appropriate custody roll. Using hash sets allows every lookup to be performed in constant average time, making the solution efficient even for the maximum input size.

### Flag

```text
HTB{th3_0ld_s3qu3nc3_n3v3r_h3s1t4t3s}
```

---

## Conclusion

This challenge is a straightforward application of hash-based lookups. By storing each custody roll in an `unordered_set`, every order can be validated in constant time, resulting in an optimal solution with linear overall complexity.

## Author: [Nitesh8766](https://github.com/Nitesh8766)
