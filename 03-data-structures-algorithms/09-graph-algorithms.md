# 09 — 图算法：遍历、最短路径、最小生成树、拓扑排序

> 关联篇目：[04 图与哈希表](./04-graph-hash-bloom.md) | [05 并查集](./05-compound-structures.md) | [08 算法范式](./08-algorithm-paradigms.md)

---

## 1. 图的遍历

### 1.1 DFS（深度优先）

```cpp
void dfs(vector<vector<int>>& graph, int u, vector<bool>& visited) {
    visited[u] = true;
    for (int v : graph[u])
        if (!visited[v])
            dfs(graph, v, visited);
}

// 迭代版
void dfsIterative(vector<vector<int>>& graph, int start) {
    vector<bool> visited(graph.size(), false);
    stack<int> stk;
    stk.push(start);
    while (!stk.empty()) {
        int u = stk.top(); stk.pop();
        if (visited[u]) continue;
        visited[u] = true;
        for (int v : graph[u])
            if (!visited[v]) stk.push(v);
    }
}
```

| 操作 | 复杂度 |
|------|:---:|
| 时间 | O(V + E) |
| 空间 | O(V)（递归栈 / 显式栈） |

### 1.2 BFS（广度优先）

```cpp
void bfs(vector<vector<int>>& graph, int start) {
    vector<bool> visited(graph.size(), false);
    queue<int> q;
    q.push(start);
    visited[start] = true;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : graph[u]) {
            if (!visited[v]) {
                visited[v] = true;
                q.push(v);
            }
        }
    }
}
```

**DFS vs BFS**：

| | DFS | BFS |
|------|------|------|
| 结构 | 栈（递归） | 队列 |
| 适用 | 连通分量、环检测、拓扑排序 | **最短路径（无权）**、层序遍历 |
| 空间 | O(深度) — 可能 O(V) | O(宽度) — 最坏 O(V) |

---

## 2. 最短路径

### 2.1 Dijkstra——非负权图单源最短路径

```cpp
vector<int> dijkstra(vector<vector<pair<int,int>>>& graph, int src) {
    int n = graph.size();
    vector<int> dist(n, INT_MAX);
    dist[src] = 0;
    // 小顶堆: {距离, 节点}
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    pq.push({0, src});

    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;  // 懒惰删除
        for (auto [v, w] : graph[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist;
}
```

| 实现 | 时间复杂度 |
|------|:---:|
| 朴素（邻接矩阵） | O(V²) |
| 堆优化（邻接表） | O((V+E) log V) |

### 2.2 Bellman-Ford——可处理负权边

```cpp
vector<int> bellmanFord(vector<vector<int>>& edges, int V, int src) {
    vector<int> dist(V, INT_MAX);
    dist[src] = 0;
    for (int i = 0; i < V - 1; ++i)  // 松弛 V-1 轮
        for (auto& e : edges)
            if (dist[e[0]] != INT_MAX && dist[e[0]] + e[2] < dist[e[1]])
                dist[e[1]] = dist[e[0]] + e[2];
    // 第 V 轮还能松弛 → 存在负环
    for (auto& e : edges)
        if (dist[e[0]] != INT_MAX && dist[e[0]] + e[2] < dist[e[1]])
            return {};  // 负环！
    return dist;
}
```

| 算法 | 时间 | 负权边 | 负环 |
|------|:---:|:---:|:---:|
| Dijkstra | O(E log V) | ❌ | ❌ |
| Bellman-Ford | O(VE) | ✅ | ✅（可检测） |
| SPFA（Bellman-Ford 优化） | O(E) 平均，O(VE) 最坏 | ✅ | ✅ |

### 2.3 Floyd-Warshall——全源最短路径

```cpp
// dp[k][i][j] = 只经过前 k 个节点的 i→j 最短距离
// dp[k][i][j] = min(dp[k-1][i][j], dp[k-1][i][k] + dp[k-1][k][j])
void floydWarshall(vector<vector<int>>& dist, int V) {
    for (int k = 0; k < V; ++k)
        for (int i = 0; i < V; ++i)
            for (int j = 0; j < V; ++j)
                if (dist[i][k] != INT_MAX && dist[k][j] != INT_MAX)
                    dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j]);
}
```

O(V³)，适合稠密图。三层循环中 k 必须在最外层。

---

## 3. 最小生成树（MST）

### 3.1 Prim——从点出发

```cpp
int prim(vector<vector<pair<int,int>>>& graph, int V) {
    vector<bool> inMST(V, false);
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    pq.push({0, 0});  // {权重, 节点}
    int total = 0, edges = 0;

    while (!pq.empty() && edges < V) {
        auto [w, u] = pq.top(); pq.pop();
        if (inMST[u]) continue;
        inMST[u] = true;
        total += w;
        edges++;
        for (auto [v, weight] : graph[u])
            if (!inMST[v]) pq.push({weight, v});
    }
    return total;
}
```

O((V+E) log V)，类似 Dijkstra。

### 3.2 Kruskal——从边出发

```cpp
int kruskal(vector<vector<int>>& edges, int V) {
    sort(edges.begin(), edges.end(),
         [](auto& a, auto& b) { return a[2] < b[2]; });  // 按权重升序
    UnionFind uf(V);
    int total = 0, count = 0;
    for (auto& e : edges) {
        int u = e[0], v = e[1], w = e[2];
        if (uf.find(u) != uf.find(v)) {
            uf.unite(u, v);
            total += w;
            if (++count == V - 1) break;
        }
    }
    return total;
}
```

O(E log E)，边稀疏时更好。

---

## 4. 拓扑排序——有向无环图（DAG）的线性排序

```cpp
vector<int> topologicalSort(vector<vector<int>>& graph, int V) {
    vector<int> indegree(V, 0);
    for (int u = 0; u < V; ++u)
        for (int v : graph[u]) indegree[v]++;

    queue<int> q;
    for (int i = 0; i < V; ++i)
        if (indegree[i] == 0) q.push(i);

    vector<int> order;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        order.push_back(u);
        for (int v : graph[u])
            if (--indegree[v] == 0) q.push(v);
    }
    // 如果 order.size() < V → 有环！
    return order;
}
```

**Kahn 算法**（BFS 版拓扑排序）：O(V+E)。另有 DFS 版本（后序遍历入栈再反转）。

### 应用场景

- 编译依赖（A 依赖 B → B 先编译）
- 课程表安排
- 任务调度（DAG 工作流）

---

## 5. 关键路径（CPM）

在 AOE 网（边表示活动，顶点表示事件）中，关键路径是**从源点到汇点的最长时间路径**，决定了工程的最短完成时间。

```
步骤：
1. 拓扑排序 → 计算每个事件的最早开始时间 ve[]
2. 逆拓扑排序 → 计算每个事件的最晚开始时间 vl[]
3. 对每条边 (u,v)：松弛时间 = vl[v] - ve[u] - w(u,v)
   松弛时间 = 0 的边在关键路径上
```

---

## 6. 最短路径算法选择

```
单源最短路径
├─ 无负权 → Dijkstra
├─ 有负权 → Bellman-Ford（或 SPFA）
└─ DAG → 拓扑排序 + DP（O(V+E)，最快）

全源最短路径
├─ 稠密图 + V 不大 → Floyd-Warshall O(V³)
└─ 稀疏图 → V 次 Dijkstra O(V × E log V)

最小生成树
├─ 稠密图 → Prim O(V²)
└─ 稀疏图 → Kruskal O(E log E)
```

---

## 小结

| 算法 | 类型 | 时间 | 关键 |
|------|------|:---:|------|
| DFS/BFS | 遍历 | O(V+E) | BFS = 无权最短路径 |
| Dijkstra | 单源最短 | O(E log V) | 非负权 |
| Bellman-Ford | 单源最短 | O(VE) | 可负权，可测负环 |
| Floyd | 全源最短 | O(V³) | 三层循环，k 在外 |
| Prim | MST | O(E log V) | 从点扩展 |
| Kruskal | MST | O(E log E) | 边排序 + 并查集 |
| Kahn | 拓扑排序 | O(V+E) | BFS + 入度表 |

下一篇 [10 字符串算法](./10-string-algorithms.md)。
