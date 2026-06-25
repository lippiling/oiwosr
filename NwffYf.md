
============================================================
 Python并查集Union-Find — 路径压缩/按秩合并/连通分量/Kruskal
============================================================

并查集（Disjoint Set Union, DSU）是一种处理不相交集合的合并与查询问题的
数据结构。常用于图论中的连通分量检测、环检测、最小生成树等场景。

============================================================
1. Union-Find 基础实现
============================================================

class UnionFind:
    """并查集类，初始时每个元素独立的集合"""
    def __init__(self, n):
        self.parent = list(range(n))    # parent[i]表示i的父节点
        self.rank = [1] * n             # rank[i]表示以i为根的树的高度（近似）
        self.count = n                  # 当前连通分量（集合）的数量

    def find(self, x):
        """查找x所属集合的根节点（带路径压缩）"""
        if self.parent[x] != x:
            # 递归查找时将路径上的所有节点直接指向根节点
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, x, y):
        """合并x和y所在的集合（按秩合并）"""
        root_x = self.find(x)
        root_y = self.find(y)
        if root_x == root_y:
            return False                # 已经在同一个集合中
        # 将高度较低的树接到高度较高的树下
        if self.rank[root_x] < self.rank[root_y]:
            self.parent[root_x] = root_y
        elif self.rank[root_x] > self.rank[root_y]:
            self.parent[root_y] = root_x
        else:
            self.parent[root_y] = root_x
            self.rank[root_x] += 1      # 高度相同时合并后高度+1
        self.count -= 1                 # 连通分量数减1
        return True

    def connected(self, x, y):
        """判断x和y是否在同一个连通分量中"""
        return self.find(x) == self.find(y)

    def get_count(self):
        """返回当前连通分量的数量"""
        return self.count

============================================================
2. 路径压缩与按秩合并详解
============================================================
# 路径压缩（Path Compression）：
#   在find操作中将沿途所有节点直接指向根节点
#   时间复杂度：近乎O(1)（阿克曼函数的反函数）
#
# 按秩合并（Union by Rank）：
#   将高度较小的树合并到高度较大的树中，避免树过深
#
# 同时使用两种优化后，单次操作的平摊时间复杂度为O(α(n))
# 其中α(n)是阿克曼函数的反函数，在实际问题中α(n) <= 5

============================================================
3. 检测图中的连通分量
============================================================

def count_components(n, edges):
    """计算n个节点（0~n-1）在边列表edges中的连通分量数"""
    uf = UnionFind(n)
    for u, v in edges:
        uf.union(u, v)
    return uf.get_count()

# 示例：0-1-2 3-4 五个节点，两个连通分量
n = 5
edges = [(0, 1), (1, 2), (3, 4)]
print("连通分量数:", count_components(n, edges))  # 2

============================================================
4. 检测图中的环
============================================================

def has_cycle(n, edges):
    """检测无向图是否有环：如果两个节点已连通，再添加边就形成环"""
    uf = UnionFind(n)
    for u, v in edges:
        if uf.connected(u, v):
            return True         # 发现环
        uf.union(u, v)
    return False

# 示例：0-1-2-0 三角形图，有环
edges_with_cycle = [(0, 1), (1, 2), (2, 0)]
print("有环吗:", has_cycle(3, edges_with_cycle))  # True

============================================================
5. 岛屿数量问题（Number of Islands）
============================================================

def num_islands(grid):
    """计算二维网格中岛屿（连通的'1'）的数量"""
    if not grid or not grid[0]:
        return 0
    rows, cols = len(grid), len(grid[0])
    uf = UnionFind(rows * cols)
    # 将所有的'0'（水域）连接到一个虚拟节点，但更简单的方法：
    # 统计'1'的数量，对相邻的'1'进行合并
    ones = 0
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                ones += 1
                idx = r * cols + c
                # 检查右边和下边的相邻陆地
                if r + 1 < rows and grid[r + 1][c] == '1':
                    uf.union(idx, (r + 1) * cols + c)
                if c + 1 < cols and grid[r][c + 1] == '1':
                    uf.union(idx, r * cols + c + 1)

    # 最终岛屿数量 = 所有'1'的数量 - 成功的合并次数
    # 更方便：用一个单独的count跟踪
    return ones - (n * cols * rows - uf.get_count())

# 简化版：用DFS更直观
def num_islands_dfs(grid):
    """DFS解法，标记已访问的陆地"""
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])
    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return
        grid[r][c] = '0'          # 标记为已访问
        dfs(r + 1, c); dfs(r - 1, c)
        dfs(r, c + 1); dfs(r, c - 1)
    count = 0
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                count += 1
                dfs(r, c)
    return count

grid = [
    ['1','1','0','0','0'],
    ['1','1','0','0','0'],
    ['0','0','1','0','0'],
    ['0','0','0','1','1']
]
print("岛屿数量:", num_islands_dfs(grid))  # 3

============================================================
6. 朋友圈问题（Friend Circles）
============================================================

def find_circle_num(M):
    """给定好友关系矩阵M，计算朋友圈数量"""
    n = len(M)
    uf = UnionFind(n)
    for i in range(n):
        for j in range(i + 1, n):
            if M[i][j] == 1:      # i和j是直接好友
                uf.union(i, j)
    return uf.get_count()

M = [
    [1, 1, 0],
    [1, 1, 0],
    [0, 0, 1]
]
print("朋友圈数量:", find_circle_num(M))  # 2

============================================================
7. Kruskal 最小生成树算法
============================================================

def kruskal_mst(n, edges):
    """Kruskal算法求最小生成树
    n: 节点数量
    edges: [(权重, u, v), ...]
    返回: (最小总权重, 选择的边列表)
    """
    edges = sorted(edges)           # 按权重从小到大排序
    uf = UnionFind(n)
    total_weight = 0
    mst_edges = []

    for weight, u, v in edges:
        if uf.union(u, v):          # 如果u和v不在同一集合中
            total_weight += weight
            mst_edges.append((u, v, weight))
            if len(mst_edges) == n - 1:  # MST有n-1条边
                break

    return total_weight, mst_edges

# 示例：6个节点的加权图
weighted_edges = [
    (4, 0, 1), (2, 0, 2), (3, 1, 2),
    (1, 1, 3), (5, 2, 3), (6, 2, 4),
    (7, 3, 4), (8, 3, 5), (9, 4, 5)
]
total, mst = kruskal_mst(6, weighted_edges)
print("MST总权重:", total)        # 21
print("MST边:", mst)

============================================================
使用示例
============================================================
if __name__ == "__main__":
    uf = UnionFind(10)
    uf.union(0, 1); uf.union(1, 2); uf.union(3, 4)
    uf.union(5, 6); uf.union(6, 7); uf.union(7, 8)
    print("0和2连通:", uf.connected(0, 2))   # True
    print("0和3连通:", uf.connected(0, 3))   # False
    print("连通分量数:", uf.get_count())      # 4

wfw.qjfkLsjdfkLaf.cn/46882.Doc
wfw.qjfkLsjdfkLaf.cn/26840.Doc
wfw.qjfkLsjdfkLaf.cn/86802.Doc
wfw.qjfkLsjdfkLaf.cn/20626.Doc
wfw.qjfkLsjdfkLaf.cn/88202.Doc
wfw.qjfkLsjdfkLaf.cn/02862.Doc
wfw.qjfkLsjdfkLaf.cn/68840.Doc
wfw.qjfkLsjdfkLaf.cn/60004.Doc
wfw.qjfkLsjdfkLaf.cn/88624.Doc
wfw.qjfkLsjdfkLaf.cn/88604.Doc
wfq.qjfkLsjdfkLaf.cn/68600.Doc
wfq.qjfkLsjdfkLaf.cn/06482.Doc
wfq.qjfkLsjdfkLaf.cn/02080.Doc
wfq.qjfkLsjdfkLaf.cn/44828.Doc
wfq.qjfkLsjdfkLaf.cn/42022.Doc
wfq.qjfkLsjdfkLaf.cn/00080.Doc
wfq.qjfkLsjdfkLaf.cn/24262.Doc
wfq.qjfkLsjdfkLaf.cn/04408.Doc
wfq.qjfkLsjdfkLaf.cn/86420.Doc
wfq.qjfkLsjdfkLaf.cn/46884.Doc
wdm.qjfkLsjdfkLaf.cn/51317.Doc
wdm.qjfkLsjdfkLaf.cn/31153.Doc
wdm.qjfkLsjdfkLaf.cn/66484.Doc
wdm.qjfkLsjdfkLaf.cn/48688.Doc
wdm.qjfkLsjdfkLaf.cn/20008.Doc
wdm.qjfkLsjdfkLaf.cn/24686.Doc
wdm.qjfkLsjdfkLaf.cn/46806.Doc
wdm.qjfkLsjdfkLaf.cn/11397.Doc
wdm.qjfkLsjdfkLaf.cn/04246.Doc
wdm.qjfkLsjdfkLaf.cn/66224.Doc
wdn.qjfkLsjdfkLaf.cn/42482.Doc
wdn.qjfkLsjdfkLaf.cn/82404.Doc
wdn.qjfkLsjdfkLaf.cn/08480.Doc
wdn.qjfkLsjdfkLaf.cn/06880.Doc
wdn.qjfkLsjdfkLaf.cn/48642.Doc
wdn.qjfkLsjdfkLaf.cn/84268.Doc
wdn.qjfkLsjdfkLaf.cn/40800.Doc
wdn.qjfkLsjdfkLaf.cn/06684.Doc
wdn.qjfkLsjdfkLaf.cn/00680.Doc
wdn.qjfkLsjdfkLaf.cn/04604.Doc
wdb.qjfkLsjdfkLaf.cn/22406.Doc
wdb.qjfkLsjdfkLaf.cn/33359.Doc
wdb.qjfkLsjdfkLaf.cn/44044.Doc
wdb.qjfkLsjdfkLaf.cn/57399.Doc
wdb.qjfkLsjdfkLaf.cn/48822.Doc
wdb.qjfkLsjdfkLaf.cn/00466.Doc
wdb.qjfkLsjdfkLaf.cn/22422.Doc
wdb.qjfkLsjdfkLaf.cn/66846.Doc
wdb.qjfkLsjdfkLaf.cn/19999.Doc
wdb.qjfkLsjdfkLaf.cn/86846.Doc
wdv.qjfkLsjdfkLaf.cn/00028.Doc
wdv.qjfkLsjdfkLaf.cn/22602.Doc
wdv.qjfkLsjdfkLaf.cn/15379.Doc
wdv.qjfkLsjdfkLaf.cn/80880.Doc
wdv.qjfkLsjdfkLaf.cn/04286.Doc
wdv.qjfkLsjdfkLaf.cn/00844.Doc
wdv.qjfkLsjdfkLaf.cn/02404.Doc
wdv.qjfkLsjdfkLaf.cn/60086.Doc
wdv.qjfkLsjdfkLaf.cn/82026.Doc
wdv.qjfkLsjdfkLaf.cn/04208.Doc
wdc.qjfkLsjdfkLaf.cn/66086.Doc
wdc.qjfkLsjdfkLaf.cn/24082.Doc
wdc.qjfkLsjdfkLaf.cn/88826.Doc
wdc.qjfkLsjdfkLaf.cn/60480.Doc
wdc.qjfkLsjdfkLaf.cn/82482.Doc
wdc.qjfkLsjdfkLaf.cn/42882.Doc
wdc.qjfkLsjdfkLaf.cn/82248.Doc
wdc.qjfkLsjdfkLaf.cn/22060.Doc
wdc.qjfkLsjdfkLaf.cn/40828.Doc
wdc.qjfkLsjdfkLaf.cn/62400.Doc
wdx.qjfkLsjdfkLaf.cn/24826.Doc
wdx.qjfkLsjdfkLaf.cn/06680.Doc
wdx.qjfkLsjdfkLaf.cn/64804.Doc
wdx.qjfkLsjdfkLaf.cn/02620.Doc
wdx.qjfkLsjdfkLaf.cn/24482.Doc
wdx.qjfkLsjdfkLaf.cn/93937.Doc
wdx.qjfkLsjdfkLaf.cn/86842.Doc
wdx.qjfkLsjdfkLaf.cn/64640.Doc
wdx.qjfkLsjdfkLaf.cn/68242.Doc
wdx.qjfkLsjdfkLaf.cn/64208.Doc
wdz.qjfkLsjdfkLaf.cn/00402.Doc
wdz.qjfkLsjdfkLaf.cn/60284.Doc
wdz.qjfkLsjdfkLaf.cn/60802.Doc
wdz.qjfkLsjdfkLaf.cn/04024.Doc
wdz.qjfkLsjdfkLaf.cn/06882.Doc
wdz.qjfkLsjdfkLaf.cn/24802.Doc
wdz.qjfkLsjdfkLaf.cn/68440.Doc
wdz.qjfkLsjdfkLaf.cn/80240.Doc
wdz.qjfkLsjdfkLaf.cn/75537.Doc
wdz.qjfkLsjdfkLaf.cn/24020.Doc
wdl.qjfkLsjdfkLaf.cn/22288.Doc
wdl.qjfkLsjdfkLaf.cn/44884.Doc
wdl.qjfkLsjdfkLaf.cn/82684.Doc
wdl.qjfkLsjdfkLaf.cn/62042.Doc
wdl.qjfkLsjdfkLaf.cn/82008.Doc
wdl.qjfkLsjdfkLaf.cn/40202.Doc
wdl.qjfkLsjdfkLaf.cn/20444.Doc
wdl.qjfkLsjdfkLaf.cn/84464.Doc
wdl.qjfkLsjdfkLaf.cn/00848.Doc
wdl.qjfkLsjdfkLaf.cn/62408.Doc
wdk.qjfkLsjdfkLaf.cn/84864.Doc
wdk.qjfkLsjdfkLaf.cn/46420.Doc
wdk.qjfkLsjdfkLaf.cn/93331.Doc
wdk.qjfkLsjdfkLaf.cn/68464.Doc
wdk.qjfkLsjdfkLaf.cn/28626.Doc
wdk.qjfkLsjdfkLaf.cn/00688.Doc
wdk.qjfkLsjdfkLaf.cn/68640.Doc
wdk.qjfkLsjdfkLaf.cn/62444.Doc
wdk.qjfkLsjdfkLaf.cn/42040.Doc
wdk.qjfkLsjdfkLaf.cn/48886.Doc
wdj.qjfkLsjdfkLaf.cn/40280.Doc
wdj.qjfkLsjdfkLaf.cn/15913.Doc
wdj.qjfkLsjdfkLaf.cn/88426.Doc
wdj.qjfkLsjdfkLaf.cn/68222.Doc
wdj.qjfkLsjdfkLaf.cn/86806.Doc
wdj.qjfkLsjdfkLaf.cn/84226.Doc
wdj.qjfkLsjdfkLaf.cn/68844.Doc
wdj.qjfkLsjdfkLaf.cn/55311.Doc
wdj.qjfkLsjdfkLaf.cn/88064.Doc
wdj.qjfkLsjdfkLaf.cn/66066.Doc
wdh.qjfkLsjdfkLaf.cn/80280.Doc
wdh.qjfkLsjdfkLaf.cn/57531.Doc
wdh.qjfkLsjdfkLaf.cn/48280.Doc
wdh.qjfkLsjdfkLaf.cn/60420.Doc
wdh.qjfkLsjdfkLaf.cn/88448.Doc
wdh.qjfkLsjdfkLaf.cn/66860.Doc
wdh.qjfkLsjdfkLaf.cn/24822.Doc
wdh.qjfkLsjdfkLaf.cn/40680.Doc
wdh.qjfkLsjdfkLaf.cn/48622.Doc
wdh.qjfkLsjdfkLaf.cn/84682.Doc
wdg.qjfkLsjdfkLaf.cn/88482.Doc
wdg.qjfkLsjdfkLaf.cn/73959.Doc
wdg.qjfkLsjdfkLaf.cn/20280.Doc
wdg.qjfkLsjdfkLaf.cn/06428.Doc
wdg.qjfkLsjdfkLaf.cn/28802.Doc
wdg.qjfkLsjdfkLaf.cn/44460.Doc
wdg.qjfkLsjdfkLaf.cn/46060.Doc
wdg.qjfkLsjdfkLaf.cn/00668.Doc
wdg.qjfkLsjdfkLaf.cn/42886.Doc
wdg.qjfkLsjdfkLaf.cn/42866.Doc
wdf.qjfkLsjdfkLaf.cn/26426.Doc
wdf.qjfkLsjdfkLaf.cn/80402.Doc
wdf.qjfkLsjdfkLaf.cn/84084.Doc
wdf.qjfkLsjdfkLaf.cn/44060.Doc
wdf.qjfkLsjdfkLaf.cn/82048.Doc
wdf.qjfkLsjdfkLaf.cn/53117.Doc
wdf.qjfkLsjdfkLaf.cn/06026.Doc
wdf.qjfkLsjdfkLaf.cn/80244.Doc
wdf.qjfkLsjdfkLaf.cn/68246.Doc
wdf.qjfkLsjdfkLaf.cn/60208.Doc
wdd.qjfkLsjdfkLaf.cn/88860.Doc
wdd.qjfkLsjdfkLaf.cn/88080.Doc
wdd.qjfkLsjdfkLaf.cn/62448.Doc
wdd.qjfkLsjdfkLaf.cn/62024.Doc
wdd.qjfkLsjdfkLaf.cn/28028.Doc
wdd.qjfkLsjdfkLaf.cn/71591.Doc
wdd.qjfkLsjdfkLaf.cn/46884.Doc
wdd.qjfkLsjdfkLaf.cn/20684.Doc
wdd.qjfkLsjdfkLaf.cn/06402.Doc
wdd.qjfkLsjdfkLaf.cn/82644.Doc
wds.qjfkLsjdfkLaf.cn/28604.Doc
wds.qjfkLsjdfkLaf.cn/02062.Doc
wds.qjfkLsjdfkLaf.cn/22688.Doc
wds.qjfkLsjdfkLaf.cn/42844.Doc
wds.qjfkLsjdfkLaf.cn/28080.Doc
wds.qjfkLsjdfkLaf.cn/00420.Doc
wds.qjfkLsjdfkLaf.cn/66822.Doc
wds.qjfkLsjdfkLaf.cn/95175.Doc
wds.qjfkLsjdfkLaf.cn/88228.Doc
wds.qjfkLsjdfkLaf.cn/93717.Doc
wda.qjfkLsjdfkLaf.cn/28064.Doc
wda.qjfkLsjdfkLaf.cn/44248.Doc
wda.qjfkLsjdfkLaf.cn/86240.Doc
wda.qjfkLsjdfkLaf.cn/48842.Doc
wda.qjfkLsjdfkLaf.cn/20880.Doc
wda.qjfkLsjdfkLaf.cn/40844.Doc
wda.qjfkLsjdfkLaf.cn/82088.Doc
wda.qjfkLsjdfkLaf.cn/66800.Doc
wda.qjfkLsjdfkLaf.cn/31593.Doc
wda.qjfkLsjdfkLaf.cn/84048.Doc
wdp.qjfkLsjdfkLaf.cn/40004.Doc
wdp.qjfkLsjdfkLaf.cn/60040.Doc
wdp.qjfkLsjdfkLaf.cn/26444.Doc
wdp.qjfkLsjdfkLaf.cn/64020.Doc
wdp.qjfkLsjdfkLaf.cn/86242.Doc
wdp.qjfkLsjdfkLaf.cn/82022.Doc
wdp.qjfkLsjdfkLaf.cn/64624.Doc
wdp.qjfkLsjdfkLaf.cn/37931.Doc
wdp.qjfkLsjdfkLaf.cn/68066.Doc
wdp.qjfkLsjdfkLaf.cn/39111.Doc
wdo.qjfkLsjdfkLaf.cn/04688.Doc
wdo.qjfkLsjdfkLaf.cn/80288.Doc
wdo.qjfkLsjdfkLaf.cn/80646.Doc
wdo.qjfkLsjdfkLaf.cn/26266.Doc
wdo.qjfkLsjdfkLaf.cn/22024.Doc
wdo.qjfkLsjdfkLaf.cn/08400.Doc
wdo.qjfkLsjdfkLaf.cn/35353.Doc
wdo.qjfkLsjdfkLaf.cn/02420.Doc
wdo.qjfkLsjdfkLaf.cn/88406.Doc
wdo.qjfkLsjdfkLaf.cn/48846.Doc
wdi.qjfkLsjdfkLaf.cn/22228.Doc
wdi.qjfkLsjdfkLaf.cn/64266.Doc
wdi.qjfkLsjdfkLaf.cn/04820.Doc
wdi.qjfkLsjdfkLaf.cn/46064.Doc
wdi.qjfkLsjdfkLaf.cn/82820.Doc
wdi.qjfkLsjdfkLaf.cn/88022.Doc
wdi.qjfkLsjdfkLaf.cn/04226.Doc
wdi.qjfkLsjdfkLaf.cn/20208.Doc
wdi.qjfkLsjdfkLaf.cn/26288.Doc
wdi.qjfkLsjdfkLaf.cn/71375.Doc
wdu.qjfkLsjdfkLaf.cn/55793.Doc
wdu.qjfkLsjdfkLaf.cn/26442.Doc
wdu.qjfkLsjdfkLaf.cn/51335.Doc
wdu.qjfkLsjdfkLaf.cn/62042.Doc
wdu.qjfkLsjdfkLaf.cn/86882.Doc
wdu.qjfkLsjdfkLaf.cn/68202.Doc
wdu.qjfkLsjdfkLaf.cn/68048.Doc
wdu.qjfkLsjdfkLaf.cn/82262.Doc
wdu.qjfkLsjdfkLaf.cn/08620.Doc
wdu.qjfkLsjdfkLaf.cn/08860.Doc
wdy.qjfkLsjdfkLaf.cn/88480.Doc
wdy.qjfkLsjdfkLaf.cn/20008.Doc
wdy.qjfkLsjdfkLaf.cn/02040.Doc
wdy.qjfkLsjdfkLaf.cn/48808.Doc
wdy.qjfkLsjdfkLaf.cn/02040.Doc
wdy.qjfkLsjdfkLaf.cn/22880.Doc
wdy.qjfkLsjdfkLaf.cn/24642.Doc
wdy.qjfkLsjdfkLaf.cn/08008.Doc
wdy.qjfkLsjdfkLaf.cn/86428.Doc
wdy.qjfkLsjdfkLaf.cn/00260.Doc
wdt.qjfkLsjdfkLaf.cn/64222.Doc
wdt.qjfkLsjdfkLaf.cn/44666.Doc
wdt.qjfkLsjdfkLaf.cn/60464.Doc
wdt.qjfkLsjdfkLaf.cn/62024.Doc
wdt.qjfkLsjdfkLaf.cn/22862.Doc
wdt.qjfkLsjdfkLaf.cn/95111.Doc
wdt.qjfkLsjdfkLaf.cn/24282.Doc
wdt.qjfkLsjdfkLaf.cn/24288.Doc
wdt.qjfkLsjdfkLaf.cn/26422.Doc
wdt.qjfkLsjdfkLaf.cn/06864.Doc
wdr.qjfkLsjdfkLaf.cn/22466.Doc
wdr.qjfkLsjdfkLaf.cn/02460.Doc
wdr.qjfkLsjdfkLaf.cn/46424.Doc
wdr.qjfkLsjdfkLaf.cn/62466.Doc
wdr.qjfkLsjdfkLaf.cn/79955.Doc
wdr.qjfkLsjdfkLaf.cn/42420.Doc
wdr.qjfkLsjdfkLaf.cn/02224.Doc
wdr.qjfkLsjdfkLaf.cn/00860.Doc
wdr.qjfkLsjdfkLaf.cn/22480.Doc
wdr.qjfkLsjdfkLaf.cn/24206.Doc
wde.qjfkLsjdfkLaf.cn/44888.Doc
wde.qjfkLsjdfkLaf.cn/62062.Doc
wde.qjfkLsjdfkLaf.cn/42804.Doc
wde.qjfkLsjdfkLaf.cn/82026.Doc
wde.qjfkLsjdfkLaf.cn/02820.Doc
wde.qjfkLsjdfkLaf.cn/77971.Doc
wde.qjfkLsjdfkLaf.cn/68266.Doc
wde.qjfkLsjdfkLaf.cn/59577.Doc
wde.qjfkLsjdfkLaf.cn/08220.Doc
wde.qjfkLsjdfkLaf.cn/28448.Doc
wdw.qjfkLsjdfkLaf.cn/86044.Doc
wdw.qjfkLsjdfkLaf.cn/20664.Doc
wdw.qjfkLsjdfkLaf.cn/80066.Doc
wdw.qjfkLsjdfkLaf.cn/80466.Doc
wdw.qjfkLsjdfkLaf.cn/28604.Doc
wdw.qjfkLsjdfkLaf.cn/60602.Doc
wdw.qjfkLsjdfkLaf.cn/59535.Doc
wdw.qjfkLsjdfkLaf.cn/64048.Doc
wdw.qjfkLsjdfkLaf.cn/80820.Doc
wdw.qjfkLsjdfkLaf.cn/24286.Doc
wdq.qjfkLsjdfkLaf.cn/20620.Doc
wdq.qjfkLsjdfkLaf.cn/48886.Doc
wdq.qjfkLsjdfkLaf.cn/08428.Doc
wdq.qjfkLsjdfkLaf.cn/08280.Doc
wdq.qjfkLsjdfkLaf.cn/82208.Doc
wdq.qjfkLsjdfkLaf.cn/53159.Doc
wdq.qjfkLsjdfkLaf.cn/44068.Doc
wdq.qjfkLsjdfkLaf.cn/24866.Doc
wdq.qjfkLsjdfkLaf.cn/28488.Doc
wdq.qjfkLsjdfkLaf.cn/46284.Doc
wsm.qjfkLsjdfkLaf.cn/66688.Doc
wsm.qjfkLsjdfkLaf.cn/48066.Doc
wsm.qjfkLsjdfkLaf.cn/04860.Doc
wsm.qjfkLsjdfkLaf.cn/80646.Doc
wsm.qjfkLsjdfkLaf.cn/02868.Doc
wsm.qjfkLsjdfkLaf.cn/26826.Doc
wsm.qjfkLsjdfkLaf.cn/64886.Doc
wsm.qjfkLsjdfkLaf.cn/39591.Doc
wsm.qjfkLsjdfkLaf.cn/04280.Doc
wsm.qjfkLsjdfkLaf.cn/66882.Doc
wsn.qjfkLsjdfkLaf.cn/20024.Doc
wsn.qjfkLsjdfkLaf.cn/64602.Doc
wsn.qjfkLsjdfkLaf.cn/24846.Doc
wsn.qjfkLsjdfkLaf.cn/04868.Doc
wsn.qjfkLsjdfkLaf.cn/00662.Doc
wsn.qjfkLsjdfkLaf.cn/24824.Doc
wsn.qjfkLsjdfkLaf.cn/51551.Doc
wsn.qjfkLsjdfkLaf.cn/24822.Doc
wsn.qjfkLsjdfkLaf.cn/68820.Doc
wsn.qjfkLsjdfkLaf.cn/40640.Doc
wsb.qjfkLsjdfkLaf.cn/66208.Doc
wsb.qjfkLsjdfkLaf.cn/62060.Doc
wsb.qjfkLsjdfkLaf.cn/26824.Doc
wsb.qjfkLsjdfkLaf.cn/24488.Doc
wsb.qjfkLsjdfkLaf.cn/97555.Doc
wsb.qjfkLsjdfkLaf.cn/88002.Doc
wsb.qjfkLsjdfkLaf.cn/73513.Doc
wsb.qjfkLsjdfkLaf.cn/46460.Doc
wsb.qjfkLsjdfkLaf.cn/62244.Doc
wsb.qjfkLsjdfkLaf.cn/88408.Doc
wsv.qjfkLsjdfkLaf.cn/86442.Doc
wsv.qjfkLsjdfkLaf.cn/06288.Doc
wsv.qjfkLsjdfkLaf.cn/02248.Doc
wsv.qjfkLsjdfkLaf.cn/48222.Doc
wsv.qjfkLsjdfkLaf.cn/00626.Doc
wsv.qjfkLsjdfkLaf.cn/77177.Doc
wsv.qjfkLsjdfkLaf.cn/86644.Doc
wsv.qjfkLsjdfkLaf.cn/48084.Doc
wsv.qjfkLsjdfkLaf.cn/08666.Doc
wsv.qjfkLsjdfkLaf.cn/73951.Doc
wsc.qjfkLsjdfkLaf.cn/84804.Doc
wsc.qjfkLsjdfkLaf.cn/62626.Doc
wsc.qjfkLsjdfkLaf.cn/24204.Doc
wsc.qjfkLsjdfkLaf.cn/80206.Doc
wsc.qjfkLsjdfkLaf.cn/88024.Doc
wsc.qjfkLsjdfkLaf.cn/40624.Doc
wsc.qjfkLsjdfkLaf.cn/06000.Doc
wsc.qjfkLsjdfkLaf.cn/26228.Doc
wsc.qjfkLsjdfkLaf.cn/26620.Doc
wsc.qjfkLsjdfkLaf.cn/06226.Doc
wsx.qjfkLsjdfkLaf.cn/84448.Doc
wsx.qjfkLsjdfkLaf.cn/66242.Doc
wsx.qjfkLsjdfkLaf.cn/66026.Doc
wsx.qjfkLsjdfkLaf.cn/64286.Doc
wsx.qjfkLsjdfkLaf.cn/20062.Doc
wsx.qjfkLsjdfkLaf.cn/08404.Doc
wsx.qjfkLsjdfkLaf.cn/20482.Doc
wsx.qjfkLsjdfkLaf.cn/22648.Doc
wsx.qjfkLsjdfkLaf.cn/82288.Doc
wsx.qjfkLsjdfkLaf.cn/40428.Doc
wsz.qjfkLsjdfkLaf.cn/04628.Doc
wsz.qjfkLsjdfkLaf.cn/88060.Doc
wsz.qjfkLsjdfkLaf.cn/51353.Doc
wsz.qjfkLsjdfkLaf.cn/00006.Doc
wsz.qjfkLsjdfkLaf.cn/62440.Doc
wsz.qjfkLsjdfkLaf.cn/22086.Doc
wsz.qjfkLsjdfkLaf.cn/60424.Doc
wsz.qjfkLsjdfkLaf.cn/86220.Doc
wsz.qjfkLsjdfkLaf.cn/86202.Doc
wsz.qjfkLsjdfkLaf.cn/84622.Doc
wsl.qjfkLsjdfkLaf.cn/64022.Doc
wsl.qjfkLsjdfkLaf.cn/82040.Doc
wsl.qjfkLsjdfkLaf.cn/66064.Doc
wsl.qjfkLsjdfkLaf.cn/42840.Doc
wsl.qjfkLsjdfkLaf.cn/04682.Doc
wsl.qjfkLsjdfkLaf.cn/86004.Doc
wsl.qjfkLsjdfkLaf.cn/40086.Doc
wsl.qjfkLsjdfkLaf.cn/44622.Doc
wsl.qjfkLsjdfkLaf.cn/66208.Doc
wsl.qjfkLsjdfkLaf.cn/28284.Doc
wsk.qjfkLsjdfkLaf.cn/40800.Doc
wsk.qjfkLsjdfkLaf.cn/00884.Doc
wsk.qjfkLsjdfkLaf.cn/79599.Doc
wsk.qjfkLsjdfkLaf.cn/00028.Doc
wsk.qjfkLsjdfkLaf.cn/28426.Doc
wsk.qjfkLsjdfkLaf.cn/40066.Doc
wsk.qjfkLsjdfkLaf.cn/86420.Doc
wsk.qjfkLsjdfkLaf.cn/60482.Doc
wsk.qjfkLsjdfkLaf.cn/06682.Doc
wsk.qjfkLsjdfkLaf.cn/55355.Doc
wsj.qjfkLsjdfkLaf.cn/95151.Doc
wsj.qjfkLsjdfkLaf.cn/60442.Doc
wsj.qjfkLsjdfkLaf.cn/40086.Doc
wsj.qjfkLsjdfkLaf.cn/26828.Doc
wsj.qjfkLsjdfkLaf.cn/04848.Doc
wsj.qjfkLsjdfkLaf.cn/33751.Doc
wsj.qjfkLsjdfkLaf.cn/04008.Doc
wsj.qjfkLsjdfkLaf.cn/62244.Doc
wsj.qjfkLsjdfkLaf.cn/84444.Doc
wsj.qjfkLsjdfkLaf.cn/33113.Doc
wsh.qjfkLsjdfkLaf.cn/71599.Doc
wsh.qjfkLsjdfkLaf.cn/88602.Doc
wsh.qjfkLsjdfkLaf.cn/06802.Doc
wsh.qjfkLsjdfkLaf.cn/48888.Doc
wsh.qjfkLsjdfkLaf.cn/04088.Doc
wsh.qjfkLsjdfkLaf.cn/04466.Doc
wsh.qjfkLsjdfkLaf.cn/24842.Doc
wsh.qjfkLsjdfkLaf.cn/08288.Doc
wsh.qjfkLsjdfkLaf.cn/62468.Doc
wsh.qjfkLsjdfkLaf.cn/08248.Doc
wsg.qjfkLsjdfkLaf.cn/24082.Doc
wsg.qjfkLsjdfkLaf.cn/88480.Doc
wsg.qjfkLsjdfkLaf.cn/42808.Doc
wsg.qjfkLsjdfkLaf.cn/44420.Doc
wsg.qjfkLsjdfkLaf.cn/62204.Doc
wsg.qjfkLsjdfkLaf.cn/60044.Doc
wsg.qjfkLsjdfkLaf.cn/02800.Doc
wsg.qjfkLsjdfkLaf.cn/02482.Doc
wsg.qjfkLsjdfkLaf.cn/95117.Doc
wsg.qjfkLsjdfkLaf.cn/06244.Doc
wsf.qjfkLsjdfkLaf.cn/11179.Doc
wsf.qjfkLsjdfkLaf.cn/44684.Doc
wsf.qjfkLsjdfkLaf.cn/80224.Doc
wsf.qjfkLsjdfkLaf.cn/15339.Doc
wsf.qjfkLsjdfkLaf.cn/44844.Doc
wsf.qjfkLsjdfkLaf.cn/24246.Doc
wsf.qjfkLsjdfkLaf.cn/26608.Doc
wsf.qjfkLsjdfkLaf.cn/22646.Doc
wsf.qjfkLsjdfkLaf.cn/80284.Doc
wsf.qjfkLsjdfkLaf.cn/60428.Doc
wsd.qjfkLsjdfkLaf.cn/13117.Doc
wsd.qjfkLsjdfkLaf.cn/86826.Doc
wsd.qjfkLsjdfkLaf.cn/68824.Doc
wsd.qjfkLsjdfkLaf.cn/26826.Doc
wsd.qjfkLsjdfkLaf.cn/42848.Doc
wsd.qjfkLsjdfkLaf.cn/26246.Doc
wsd.qjfkLsjdfkLaf.cn/00424.Doc
wsd.qjfkLsjdfkLaf.cn/86284.Doc
wsd.qjfkLsjdfkLaf.cn/84486.Doc
wsd.qjfkLsjdfkLaf.cn/82266.Doc
wss.qjfkLsjdfkLaf.cn/31755.Doc
wss.qjfkLsjdfkLaf.cn/02026.Doc
wss.qjfkLsjdfkLaf.cn/51515.Doc
wss.qjfkLsjdfkLaf.cn/48626.Doc
wss.qjfkLsjdfkLaf.cn/68668.Doc
wss.qjfkLsjdfkLaf.cn/33173.Doc
wss.qjfkLsjdfkLaf.cn/88446.Doc
wss.qjfkLsjdfkLaf.cn/44808.Doc
wss.qjfkLsjdfkLaf.cn/44002.Doc
wss.qjfkLsjdfkLaf.cn/20648.Doc
wsa.qjfkLsjdfkLaf.cn/68024.Doc
wsa.qjfkLsjdfkLaf.cn/24284.Doc
wsa.qjfkLsjdfkLaf.cn/80248.Doc
wsa.qjfkLsjdfkLaf.cn/06002.Doc
wsa.qjfkLsjdfkLaf.cn/86840.Doc
wsa.qjfkLsjdfkLaf.cn/02280.Doc
wsa.qjfkLsjdfkLaf.cn/86666.Doc
wsa.qjfkLsjdfkLaf.cn/02084.Doc
wsa.qjfkLsjdfkLaf.cn/28220.Doc
wsa.qjfkLsjdfkLaf.cn/86622.Doc
wsp.qjfkLsjdfkLaf.cn/08002.Doc
wsp.qjfkLsjdfkLaf.cn/08842.Doc
wsp.qjfkLsjdfkLaf.cn/48804.Doc
wsp.qjfkLsjdfkLaf.cn/04284.Doc
wsp.qjfkLsjdfkLaf.cn/82804.Doc
wsp.qjfkLsjdfkLaf.cn/40248.Doc
wsp.qjfkLsjdfkLaf.cn/20082.Doc
wsp.qjfkLsjdfkLaf.cn/00808.Doc
wsp.qjfkLsjdfkLaf.cn/24268.Doc
wsp.qjfkLsjdfkLaf.cn/68046.Doc
wso.qjfkLsjdfkLaf.cn/64000.Doc
wso.qjfkLsjdfkLaf.cn/00246.Doc
wso.qjfkLsjdfkLaf.cn/82824.Doc
wso.qjfkLsjdfkLaf.cn/26446.Doc
wso.qjfkLsjdfkLaf.cn/22408.Doc
wso.qjfkLsjdfkLaf.cn/62660.Doc
wso.qjfkLsjdfkLaf.cn/59575.Doc
wso.qjfkLsjdfkLaf.cn/59711.Doc
wso.qjfkLsjdfkLaf.cn/66424.Doc
wso.qjfkLsjdfkLaf.cn/88680.Doc
wsi.qjfkLsjdfkLaf.cn/08422.Doc
wsi.qjfkLsjdfkLaf.cn/71937.Doc
wsi.qjfkLsjdfkLaf.cn/77337.Doc
wsi.qjfkLsjdfkLaf.cn/06680.Doc
wsi.qjfkLsjdfkLaf.cn/33731.Doc
wsi.qjfkLsjdfkLaf.cn/82008.Doc
wsi.qjfkLsjdfkLaf.cn/20288.Doc
wsi.qjfkLsjdfkLaf.cn/86840.Doc
wsi.qjfkLsjdfkLaf.cn/42486.Doc
wsi.qjfkLsjdfkLaf.cn/82644.Doc
wsu.qjfkLsjdfkLaf.cn/99559.Doc
wsu.qjfkLsjdfkLaf.cn/48284.Doc
wsu.qjfkLsjdfkLaf.cn/86862.Doc
wsu.qjfkLsjdfkLaf.cn/24602.Doc
wsu.qjfkLsjdfkLaf.cn/66668.Doc
wsu.qjfkLsjdfkLaf.cn/44006.Doc
wsu.qjfkLsjdfkLaf.cn/28422.Doc
wsu.qjfkLsjdfkLaf.cn/91513.Doc
wsu.qjfkLsjdfkLaf.cn/68426.Doc
wsu.qjfkLsjdfkLaf.cn/68848.Doc
wsy.qjfkLsjdfkLaf.cn/66820.Doc
wsy.qjfkLsjdfkLaf.cn/97733.Doc
wsy.qjfkLsjdfkLaf.cn/40266.Doc
wsy.qjfkLsjdfkLaf.cn/86666.Doc
wsy.qjfkLsjdfkLaf.cn/60488.Doc
wsy.qjfkLsjdfkLaf.cn/44424.Doc
wsy.qjfkLsjdfkLaf.cn/60666.Doc
wsy.qjfkLsjdfkLaf.cn/64026.Doc
wsy.qjfkLsjdfkLaf.cn/46084.Doc
wsy.qjfkLsjdfkLaf.cn/86224.Doc
