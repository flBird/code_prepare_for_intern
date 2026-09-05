# 图的最短路径：BFS / DFS / Dijkstra / Floyd 整理

> 整理时间：2026-09-05
> 适用：给定无权/带权无向图，求两点间最短距离
> 本题背景：邻接矩阵存边，边带权 `d` → 属于"有权图"，正确解法是 Dijkstra 或 Floyd

---

## 0. 计算思想（总览）

- **无权图**：BFS 按层扩展，第一次到达目标即最短路（边数最少）。
- **有权图（边带权 d）**：按"层"走 ≠ 按"权"走，BFS/DFS 都不能保证最短路，必须用
  - **Dijkstra**：单源最短路（从一个起点到所有点），O(n²) 朴素 / O((V+E)logV) 堆优化
  - **Floyd-Warshall**：多源最短路（任意两点），O(n³)，特别适合"多次询问任意两点距离"
- **DFS**：本质是穷举路径，带"永久 visited"只能走出一条生成树路径，取 `min` 也只是树内挑，
  **得不到全局最短路**（详见第 3 节）。
- 本题边带权 `d`，是**有权图** → 正确解法是下面的 Dijkstra 或 Floyd，不是 BFS/DFS。

一句话：无权用 BFS；有权单源用 Dijkstra；有权全源/多次询问用 Floyd。

---

## 1. BFS 基本写法（无权图最短路）

核心：队列 + 分层，首次到达即最短（仅适用于无权图）。

邻接表版（最常用）：
```python
from collections import deque

def bfs_shortest(graph, s, t):
    # graph: 邻接表，graph[u] = [v1, v2, ...]（无权，或忽略权重）
    n = len(graph)
    dist = [-1] * n          # -1 表示未访问
    dist[s] = 0
    q = deque([s])
    while q:
        u = q.popleft()
        for v in graph[u]:
            if dist[v] == -1:
                dist[v] = dist[u] + 1   # 无权：层数 +1
                if v == t:
                    return dist[v]
                q.append(v)
    return dist[t]           # 仍为 -1 → 不可达
```

邻接矩阵版（对应本题数据结构）：
```python
from collections import deque

def bfs_matrix(matrix, s, t):
    n = len(matrix)
    dist = [-1] * n
    dist[s] = 0
    q = deque([s])
    while q:
        u = q.popleft()
        for v in range(n):
            if matrix[u][v] != inf and dist[v] == -1:
                dist[v] = dist[u] + 1
                if v == t:
                    return dist[v]
                q.append(v)
    return dist[t]
```

复杂度：O(V+E)（邻接表）/ O(V²)（邻接矩阵）。若边带权则**不能用 BFS 求最短路**。

---

## 2. DFS 基本写法（遍历 / 判连通 / 搜路径）

核心：递归或栈，visited 防止走回头路。适合"是否存在路径""遍历全图"，**不适合求最短距离**。

```python
def dfs(u, target, graph, visited):
    if u == target:
        return True              # 找到了一条路径（未必最短）
    visited[u] = True
    for v in graph[u]:
        if not visited[v]:
            if dfs(v, target, graph, visited):
                return True
    return False
```

要点：
- DFS 能判断"可达 / 不可达"，但第一次到达目标的路径长度不一定是最小的（除非所有边权都等于 1，退化为 BFS）。
- 若想用 DFS 搜"所有路径取最短"，必须**不永久标记 visited**（回溯时要 `visited[v]=False`），否则会漏路径且极慢。

---

## 3. 本题的错误解法（递归 DFS + 永久 visited）

原错误代码（带权图却用 DFS 求最短路）：
```python
from math import inf

def distance(matrix, x, y):
    visited = [False for _ in range(len(matrix))]
    n = len(matrix)
    def bfs(start, y):                 # 名字叫 bfs，其实是 DFS
        if start == y:
            return 0
        visited[start] = True
        res = inf
        for j in range(n):
            if visited[j] == False and matrix[start][j] != inf:
                visited[j] = True
                res = min(res, matrix[start][j] + bfs(j, y))
        return res
    return bfs(x, y)
```

### 为什么错（逻辑层面）

`visited` 在整个 `distance` 调用里只有一份，且 `visited[j]=True` 后**永久不再撤销**（无回溯）。
DFS 从 start 只会走出一棵"生成树"，`min` 只是在这棵树内部挑一条路径——但真正的最短路往往要
"绕过"已被别的支路占用的结点，永久 visited 把这些结点锁死，导致**漏掉更短路径**。

反例（带权图，求 0→3）：
```
边: 0-1 (10), 0-2 (1), 2-3 (1), 1-3 (1)
```
真正最短路：`0→2→3` = 2。
但这段 DFS：从 0 先试邻居 1 → `bfs(1)` 走到 3 并把 3 标记 visited，返回 10；
再试邻居 2 → `bfs(2)` 时 3、0 都已 visited，无路可走返回 inf；
最终 `min(10, inf) = 10`，错成 10。

### 历史踩坑（导致"没输出"的运行时错误）

1. `inf` 未定义 → `NameError`，被外层裸 `except: break` 吞掉 → 静默退出、无输出。
   （修法：`from math import inf`）
2. 输入结点 1-based 但矩阵 0-based，出现编号 `n` 时 `matrix[n][...]` 越界 → `IndexError`，
   同样被裸 `except` 吞掉 → 无输出。
   （修法：输入 `i-1, j-1, x-1, y-1` 转 0-based，或矩阵开 `(n+1)×(n+1)`）
3. 裸 `except:` 会吞掉**任何**异常，调试时应改成 `except Exception as e: print('ERR:', e)` 或只捕获 `EOFError`。

**结论**：这段既不是 BFS 也不是正确最短路算法；带权图求最短路请用第 4 / 第 5 节。

---

## 4. 正确解法一：Dijkstra（单源最短路，O(n²) 朴素版）

思想：维护 `dist[j]` = 从起点 x 到 j 的当前已知最短距离。每轮从"尚未确定最短路的结点"中，
挑 `dist` 最小的那一个 `u`，**确定**它的最短路，并用 `u` 去"松弛"它的邻居
（`dist[j] = min(dist[j], dist[u] + matrix[u][j])`）。

```python
from math import inf

def distance(matrix, x, y):
    n = len(matrix)
    dist = [inf for _ in range(n)]
    visited = [False for _ in range(n)]
    for j in range(n):                 # 初始化：起点直达的边
        dist[j] = matrix[x][j] if j != x else 0
    visited[x] = True
    # dist[j] 表示从 x 到 j 的最短距离
    while 1:
        u, best = -1, inf
        for j in range(n):             # 选未确定中最短的那个
            if visited[j] == False and best > dist[j]:
                u = j
                best = dist[j]
        if u == -1 or u == y:          # 剩下的都不可达，或已到目标
            return dist[y]
        visited[u] = True              # 确定 u 的最短路
        for j in range(n):             # 用 u 松弛邻居
            dist[j] = min(dist[j], dist[u] + matrix[u][j])
    return dist[y]
```

与错误解法的关键区别：
- 这里的 `visited[u]=True` 表示"**u 的最短路已确定**"，只在确定后才标记，且不会阻碍别人被松弛
  （松弛是更新 `dist`，不依赖 visited）。
- `dist[j]` 借助"已确定结点"持续被刷新，最终收敛到全局最短路；而错误解法的 `visited` 直接封死路径。

复杂度：O(n²)（朴素，每轮扫一遍选最小）。用堆可优化到 O((V+E)logV)。

---

## 5. 正确解法二：Floyd-Warshall（全源最短路，O(n³)）

思想：**插点法**。枚举"中间点" `k`，若经过 `k` 能让 `i→j` 更短，就更新 `matrix[i][j]`。
一次性算出**任意两点**的最短路，适合多次询问。

```python
from math import inf

def distance(matrix, x, y):
    n = len(matrix)
    for k in range(n):                  # 中间点 k 必须在最外层！
        for i in range(n):
            for j in range(n):
                if matrix[i][k] != inf and matrix[j][k] != inf:
                    matrix[i][j] = min(matrix[i][j], matrix[i][k] + matrix[k][j])
    return matrix[x][y]
```

要点：
- **`k` 的循环必须在最外层**，否则语义变成"只用部分中间点"，结果错误。
- 原地修改 `matrix`，调用后 `matrix[i][j]` 就是 i→j 最短路；`== inf` 即不可达。
- `if matrix[i][k]!=inf and matrix[j][k]!=inf` 防止 `inf+inf`（虽 Python 中 inf+inf 仍是 inf 不报错，
  但加判断更清晰、避免潜在问题）。
- 复杂度 O(n³)。n 较小时非常简洁好写。

---

## 6. 三种正确解法对比

| 方法 | 适用 | 复杂度 | 特点 |
|---|---|---|---|
| BFS | 无权图单源最短路 | O(V+E) | 首次到达即最短；带权不可用 |
| Dijkstra | 有权图单源最短路 | O(n²) / O((V+E)logV) | 边权非负；每轮选最小 dist 松弛 |
| Floyd | 有权图全源最短路 | O(n³) | 一次算任意两点；k 循环在最外层 |

---

## 7. 易错点汇总

1. `inf` 必须定义：`from math import inf`（或 `inf = float('inf')`）。
2. 输入 1-based 时，结点要 `-1` 转 0-based，或矩阵开 `(n+1)×(n+1)`。
3. 裸 `except:` 会吞掉所有异常导致"无输出"，调试时打印异常或只捕获 `EOFError`。
4. 带权图不能用 BFS/DFS 求最短路；选 Dijkstra（单源）或 Floyd（全源）。
5. Floyd 的 `k` 必须在最外层循环。
6. Dijkstra 的 `visited` 语义是"已确定最短路"，与 DFS 的"防回头"语义不同，不要混用。
7. 矩阵建无向边要双向赋值：`matrix[i][j]=matrix[j][i]=d`。
