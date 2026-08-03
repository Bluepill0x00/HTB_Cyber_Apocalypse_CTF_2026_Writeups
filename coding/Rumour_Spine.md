# Rumour Spine - Write-up

**Category:** Coding  
**Difficulty:** Hard

---

## Challenge Description

The challenge models a directed graph where:

- Each node represents a quiet hand.
- Each directed edge represents a possible route through which the priming signal can travel.
- The coordinator (`S`) and the target district (`T`) cannot be removed.

The objective is to determine the **minimum number of intermediate nodes** that must be removed so that **every path** from `S` to `T` is disconnected.

---

## Solution

This is a classic **Minimum Vertex Cut** problem on a directed graph.

Unlike the standard minimum edge cut problem, removing **vertices** requires transforming the graph before applying a maximum flow algorithm.

### Node Splitting

Every vertex `v` is divided into two nodes:

```
v_in
v_out
```

These two nodes are connected by an edge.

For every removable vertex:

```
v_in → v_out (capacity = 1)
```

Removing this edge corresponds to removing the original vertex.

Since the coordinator (`S`) and target (`T`) cannot be removed, their connecting edges are assigned infinite capacity:

```
S_in → S_out (INF)
T_in → T_out (INF)
```

For every original directed edge:

```
u → v
```

we create:

```
u_out → v_in (INF)
```

After constructing this transformed graph, the answer becomes the **minimum cut**, which is equal to the **maximum flow** by the Max-Flow Min-Cut Theorem.

Dinic's algorithm efficiently computes the maximum flow for the given constraints.

---

## Algorithm

1. Read the graph.
2. Split every node into an input and output node.
3. Connect every split node with capacity:
   - `1` for removable nodes.
   - `INF` for `S` and `T`.
4. Convert every original edge into an infinite-capacity edge between split nodes.
5. Run Dinic's Maximum Flow algorithm.
6. Output the resulting maximum flow.

---

## Complexity Analysis

Let:

- `V = 2 × N`
- `E = original edges + split edges`

### Time Complexity

```
O(E × V²)
```

using Dinic's algorithm, which is more than sufficient for:

- `N ≤ 120`
- `E ≤ 360`

### Space Complexity

```
O(V + E)
```

---

## C++ Solution

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;

struct Edge {
    int to, rev, cap;
};

struct Dinic {
    int N;
    vector<vector<Edge>> g;
    vector<int> level, ptr;

    Dinic(int n) : N(n), g(n), level(n), ptr(n) {}

    void addEdge(int v, int to, int cap) {
        g[v].push_back({to, (int)g[to].size(), cap});
        g[to].push_back({v, (int)g[v].size() - 1, 0});
    }

    bool bfs(int s, int t) {
        fill(level.begin(), level.end(), -1);

        queue<int> q;
        q.push(s);
        level[s] = 0;

        while (!q.empty()) {
            int v = q.front();
            q.pop();

            for (auto &e : g[v]) {
                if (e.cap > 0 && level[e.to] == -1) {
                    level[e.to] = level[v] + 1;
                    q.push(e.to);
                }
            }
        }

        return level[t] != -1;
    }

    int dfs(int v, int t, int pushed) {
        if (!pushed) return 0;
        if (v == t) return pushed;

        for (int &cid = ptr[v]; cid < (int)g[v].size(); cid++) {
            Edge &e = g[v][cid];

            if (e.cap > 0 && level[e.to] == level[v] + 1) {
                int tr = dfs(e.to, t, min(pushed, e.cap));

                if (!tr) continue;

                e.cap -= tr;
                g[e.to][e.rev].cap += tr;

                return tr;
            }
        }

        return 0;
    }

    int maxflow(int s, int t) {
        int flow = 0;

        while (bfs(s, t)) {
            fill(ptr.begin(), ptr.end(), 0);

            while (int pushed = dfs(s, t, INF))
                flow += pushed;
        }

        return flow;
    }
};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int N, E, S, T;
    cin >> N >> E >> S >> T;

    Dinic dinic(2 * N);

    auto in = [](int x) { return 2 * x; };
    auto out = [](int x) { return 2 * x + 1; };

    for (int i = 0; i < N; i++) {
        if (i == S || i == T)
            dinic.addEdge(in(i), out(i), INF);
        else
            dinic.addEdge(in(i), out(i), 1);
    }

    for (int i = 0; i < E; i++) {
        int u, v;
        cin >> u >> v;
        dinic.addEdge(out(u), in(v), INF);
    }

    cout << dinic.maxflow(out(S), in(T)) << '\n';

    return 0;
}
```

---

## Result

By transforming each vertex into an input/output pair and assigning capacities to represent removable nodes, the problem becomes a standard maximum flow computation. Dinic's algorithm efficiently computes the minimum number of intermediate nodes that must be removed to disconnect every path from the coordinator to the target.

### Flag

```text
HTB{qu13t_h4nds_c4n_st1ll_b3_cut}
```

---

## Conclusion

This challenge demonstrates a classic graph transformation technique. Converting a **minimum vertex cut** problem into a **maximum flow** problem using **node splitting** is a well-known algorithmic approach that allows powerful network flow algorithms such as Dinic's to solve what initially appears to be a difficult graph problem. The resulting solution is both optimal and efficient for the given constraints.

## Author: [Nitesh8766](https://github.com/Nitesh8766)
