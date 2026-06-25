
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

dbp.baoquan026.cn/44422.Doc
dbp.baoquan026.cn/79359.Doc
dbp.baoquan026.cn/57773.Doc
dbp.baoquan026.cn/55137.Doc
dbp.baoquan026.cn/97315.Doc
dbp.baoquan026.cn/53599.Doc
dbp.baoquan026.cn/51197.Doc
dbp.baoquan026.cn/15175.Doc
dbp.baoquan026.cn/68226.Doc
dbp.baoquan026.cn/51917.Doc
dbo.baoquan026.cn/33591.Doc
dbo.baoquan026.cn/39995.Doc
dbo.baoquan026.cn/93373.Doc
dbo.baoquan026.cn/91517.Doc
dbo.baoquan026.cn/06280.Doc
dbo.baoquan026.cn/95519.Doc
dbo.baoquan026.cn/22284.Doc
dbo.baoquan026.cn/22420.Doc
dbo.baoquan026.cn/95931.Doc
dbo.baoquan026.cn/91575.Doc
dbi.baoquan026.cn/59733.Doc
dbi.baoquan026.cn/75759.Doc
dbi.baoquan026.cn/79311.Doc
dbi.baoquan026.cn/71137.Doc
dbi.baoquan026.cn/53913.Doc
dbi.baoquan026.cn/97551.Doc
dbi.baoquan026.cn/55371.Doc
dbi.baoquan026.cn/99535.Doc
dbi.baoquan026.cn/95953.Doc
dbi.baoquan026.cn/31193.Doc
dbu.baoquan026.cn/57939.Doc
dbu.baoquan026.cn/88206.Doc
dbu.baoquan026.cn/31511.Doc
dbu.baoquan026.cn/59717.Doc
dbu.baoquan026.cn/51739.Doc
dbu.baoquan026.cn/95531.Doc
dbu.baoquan026.cn/97915.Doc
dbu.baoquan026.cn/33575.Doc
dbu.baoquan026.cn/57179.Doc
dbu.baoquan026.cn/33775.Doc
dby.baoquan026.cn/11751.Doc
dby.baoquan026.cn/93755.Doc
dby.baoquan026.cn/99191.Doc
dby.baoquan026.cn/39331.Doc
dby.baoquan026.cn/13579.Doc
dby.baoquan026.cn/71937.Doc
dby.baoquan026.cn/75957.Doc
dby.baoquan026.cn/79135.Doc
dby.baoquan026.cn/37757.Doc
dby.baoquan026.cn/53513.Doc
dbt.baoquan026.cn/33371.Doc
dbt.baoquan026.cn/77577.Doc
dbt.baoquan026.cn/13337.Doc
dbt.baoquan026.cn/95315.Doc
dbt.baoquan026.cn/37379.Doc
dbt.baoquan026.cn/77979.Doc
dbt.baoquan026.cn/77155.Doc
dbt.baoquan026.cn/35795.Doc
dbt.baoquan026.cn/79377.Doc
dbt.baoquan026.cn/51191.Doc
dbr.baoquan026.cn/77331.Doc
dbr.baoquan026.cn/53199.Doc
dbr.baoquan026.cn/91911.Doc
dbr.baoquan026.cn/33135.Doc
dbr.baoquan026.cn/71135.Doc
dbr.baoquan026.cn/39935.Doc
dbr.baoquan026.cn/93339.Doc
dbr.baoquan026.cn/51975.Doc
dbr.baoquan026.cn/37711.Doc
dbr.baoquan026.cn/59111.Doc
dbe.baoquan026.cn/13957.Doc
dbe.baoquan026.cn/35331.Doc
dbe.baoquan026.cn/75133.Doc
dbe.baoquan026.cn/15337.Doc
dbe.baoquan026.cn/51373.Doc
dbe.baoquan026.cn/15933.Doc
dbe.baoquan026.cn/77933.Doc
dbe.baoquan026.cn/95357.Doc
dbe.baoquan026.cn/59359.Doc
dbe.baoquan026.cn/91973.Doc
dbw.baoquan026.cn/55717.Doc
dbw.baoquan026.cn/77733.Doc
dbw.baoquan026.cn/99513.Doc
dbw.baoquan026.cn/97717.Doc
dbw.baoquan026.cn/93957.Doc
dbw.baoquan026.cn/93753.Doc
dbw.baoquan026.cn/35977.Doc
dbw.baoquan026.cn/33135.Doc
dbw.baoquan026.cn/19955.Doc
dbw.baoquan026.cn/77159.Doc
dbq.baoquan026.cn/99593.Doc
dbq.baoquan026.cn/39331.Doc
dbq.baoquan026.cn/51357.Doc
dbq.baoquan026.cn/33793.Doc
dbq.baoquan026.cn/91199.Doc
dbq.baoquan026.cn/31737.Doc
dbq.baoquan026.cn/19551.Doc
dbq.baoquan026.cn/11959.Doc
dbq.baoquan026.cn/39951.Doc
dbq.baoquan026.cn/75995.Doc
dvm.baoquan026.cn/37715.Doc
dvm.baoquan026.cn/71955.Doc
dvm.baoquan026.cn/99319.Doc
dvm.baoquan026.cn/39753.Doc
dvm.baoquan026.cn/99531.Doc
dvm.baoquan026.cn/91135.Doc
dvm.baoquan026.cn/17951.Doc
dvm.baoquan026.cn/08062.Doc
dvm.baoquan026.cn/55997.Doc
dvm.baoquan026.cn/15797.Doc
dvn.baoquan026.cn/11393.Doc
dvn.baoquan026.cn/17777.Doc
dvn.baoquan026.cn/57115.Doc
dvn.baoquan026.cn/91795.Doc
dvn.baoquan026.cn/79313.Doc
dvn.baoquan026.cn/31715.Doc
dvn.baoquan026.cn/39517.Doc
dvn.baoquan026.cn/33173.Doc
dvn.baoquan026.cn/53953.Doc
dvn.baoquan026.cn/15593.Doc
dvb.baoquan026.cn/40660.Doc
dvb.baoquan026.cn/93393.Doc
dvb.baoquan026.cn/11317.Doc
dvb.baoquan026.cn/19597.Doc
dvb.baoquan026.cn/15135.Doc
dvb.baoquan026.cn/97599.Doc
dvb.baoquan026.cn/11931.Doc
dvb.baoquan026.cn/93733.Doc
dvb.baoquan026.cn/13159.Doc
dvb.baoquan026.cn/19377.Doc
dvv.baoquan026.cn/33319.Doc
dvv.baoquan026.cn/13551.Doc
dvv.baoquan026.cn/73351.Doc
dvv.baoquan026.cn/91911.Doc
dvv.baoquan026.cn/99711.Doc
dvv.baoquan026.cn/13379.Doc
dvv.baoquan026.cn/57531.Doc
dvv.baoquan026.cn/99153.Doc
dvv.baoquan026.cn/71577.Doc
dvv.baoquan026.cn/51791.Doc
dvc.baoquan026.cn/59379.Doc
dvc.baoquan026.cn/39173.Doc
dvc.baoquan026.cn/53991.Doc
dvc.baoquan026.cn/97371.Doc
dvc.baoquan026.cn/97377.Doc
dvc.baoquan026.cn/95111.Doc
dvc.baoquan026.cn/91793.Doc
dvc.baoquan026.cn/59737.Doc
dvc.baoquan026.cn/59195.Doc
dvc.baoquan026.cn/33119.Doc
dvx.baoquan026.cn/31553.Doc
dvx.baoquan026.cn/35319.Doc
dvx.baoquan026.cn/93571.Doc
dvx.baoquan026.cn/37739.Doc
dvx.baoquan026.cn/15595.Doc
dvx.baoquan026.cn/17173.Doc
dvx.baoquan026.cn/13795.Doc
dvx.baoquan026.cn/57317.Doc
dvx.baoquan026.cn/53733.Doc
dvx.baoquan026.cn/37959.Doc
dvz.baoquan026.cn/75939.Doc
dvz.baoquan026.cn/19119.Doc
dvz.baoquan026.cn/71535.Doc
dvz.baoquan026.cn/91739.Doc
dvz.baoquan026.cn/11317.Doc
dvz.baoquan026.cn/22460.Doc
dvz.baoquan026.cn/08804.Doc
dvz.baoquan026.cn/71779.Doc
dvz.baoquan026.cn/75999.Doc
dvz.baoquan026.cn/53593.Doc
dvl.baoquan026.cn/51197.Doc
dvl.baoquan026.cn/13197.Doc
dvl.baoquan026.cn/11797.Doc
dvl.baoquan026.cn/31993.Doc
dvl.baoquan026.cn/35137.Doc
dvl.baoquan026.cn/51337.Doc
dvl.baoquan026.cn/51391.Doc
dvl.baoquan026.cn/71551.Doc
dvl.baoquan026.cn/13755.Doc
dvl.baoquan026.cn/19995.Doc
dvk.baoquan026.cn/57339.Doc
dvk.baoquan026.cn/97397.Doc
dvk.baoquan026.cn/93179.Doc
dvk.baoquan026.cn/57711.Doc
dvk.baoquan026.cn/53399.Doc
dvk.baoquan026.cn/99577.Doc
dvk.baoquan026.cn/75395.Doc
dvk.baoquan026.cn/57959.Doc
dvk.baoquan026.cn/93591.Doc
dvk.baoquan026.cn/79357.Doc
dvj.baoquan026.cn/06046.Doc
dvj.baoquan026.cn/19973.Doc
dvj.baoquan026.cn/91313.Doc
dvj.baoquan026.cn/73597.Doc
dvj.baoquan026.cn/13115.Doc
dvj.baoquan026.cn/31357.Doc
dvj.baoquan026.cn/75915.Doc
dvj.baoquan026.cn/99975.Doc
dvj.baoquan026.cn/99935.Doc
dvj.baoquan026.cn/55379.Doc
dvh.baoquan026.cn/15373.Doc
dvh.baoquan026.cn/71137.Doc
dvh.baoquan026.cn/57753.Doc
dvh.baoquan026.cn/75159.Doc
dvh.baoquan026.cn/13955.Doc
dvh.baoquan026.cn/84084.Doc
dvh.baoquan026.cn/73715.Doc
dvh.baoquan026.cn/97335.Doc
dvh.baoquan026.cn/95119.Doc
dvh.baoquan026.cn/79353.Doc
dvg.baoquan026.cn/37373.Doc
dvg.baoquan026.cn/17377.Doc
dvg.baoquan026.cn/15335.Doc
dvg.baoquan026.cn/57393.Doc
dvg.baoquan026.cn/19911.Doc
dvg.baoquan026.cn/73933.Doc
dvg.baoquan026.cn/55793.Doc
dvg.baoquan026.cn/91179.Doc
dvg.baoquan026.cn/79753.Doc
dvg.baoquan026.cn/37359.Doc
dvf.baoquan026.cn/71375.Doc
dvf.baoquan026.cn/35319.Doc
dvf.baoquan026.cn/71755.Doc
dvf.baoquan026.cn/57977.Doc
dvf.baoquan026.cn/93373.Doc
dvf.baoquan026.cn/35115.Doc
dvf.baoquan026.cn/15397.Doc
dvf.baoquan026.cn/93395.Doc
dvf.baoquan026.cn/28866.Doc
dvf.baoquan026.cn/91773.Doc
dvd.baoquan026.cn/55553.Doc
dvd.baoquan026.cn/75599.Doc
dvd.baoquan026.cn/04088.Doc
dvd.baoquan026.cn/91519.Doc
dvd.baoquan026.cn/99353.Doc
dvd.baoquan026.cn/53171.Doc
dvd.baoquan026.cn/11779.Doc
dvd.baoquan026.cn/37995.Doc
dvd.baoquan026.cn/97919.Doc
dvd.baoquan026.cn/53593.Doc
dvs.baoquan026.cn/35113.Doc
dvs.baoquan026.cn/55317.Doc
dvs.baoquan026.cn/17395.Doc
dvs.baoquan026.cn/95337.Doc
dvs.baoquan026.cn/99133.Doc
dvs.baoquan026.cn/77373.Doc
dvs.baoquan026.cn/55333.Doc
dvs.baoquan026.cn/39313.Doc
dvs.baoquan026.cn/39153.Doc
dvs.baoquan026.cn/73757.Doc
dva.baoquan026.cn/33395.Doc
dva.baoquan026.cn/35317.Doc
dva.baoquan026.cn/97537.Doc
dva.baoquan026.cn/39973.Doc
dva.baoquan026.cn/35931.Doc
dva.baoquan026.cn/35393.Doc
dva.baoquan026.cn/79937.Doc
dva.baoquan026.cn/11951.Doc
dva.baoquan026.cn/73375.Doc
dva.baoquan026.cn/51391.Doc
dvp.baoquan026.cn/11935.Doc
dvp.baoquan026.cn/93155.Doc
dvp.baoquan026.cn/93371.Doc
dvp.baoquan026.cn/91957.Doc
dvp.baoquan026.cn/15799.Doc
dvp.baoquan026.cn/73157.Doc
dvp.baoquan026.cn/59357.Doc
dvp.baoquan026.cn/31917.Doc
dvp.baoquan026.cn/19195.Doc
dvp.baoquan026.cn/93519.Doc
dvo.baoquan026.cn/91195.Doc
dvo.baoquan026.cn/15751.Doc
dvo.baoquan026.cn/39179.Doc
dvo.baoquan026.cn/97911.Doc
dvo.baoquan026.cn/75511.Doc
dvo.baoquan026.cn/13593.Doc
dvo.baoquan026.cn/97195.Doc
dvo.baoquan026.cn/39795.Doc
dvo.baoquan026.cn/35795.Doc
dvo.baoquan026.cn/91757.Doc
dvi.baoquan026.cn/79951.Doc
dvi.baoquan026.cn/39937.Doc
dvi.baoquan026.cn/97113.Doc
dvi.baoquan026.cn/93333.Doc
dvi.baoquan026.cn/55517.Doc
dvi.baoquan026.cn/15977.Doc
dvi.baoquan026.cn/75375.Doc
dvi.baoquan026.cn/19511.Doc
dvi.baoquan026.cn/79751.Doc
dvi.baoquan026.cn/93359.Doc
dvu.baoquan026.cn/33191.Doc
dvu.baoquan026.cn/99915.Doc
dvu.baoquan026.cn/11357.Doc
dvu.baoquan026.cn/37577.Doc
dvu.baoquan026.cn/71797.Doc
dvu.baoquan026.cn/95395.Doc
dvu.baoquan026.cn/79533.Doc
dvu.baoquan026.cn/71359.Doc
dvu.baoquan026.cn/75357.Doc
dvu.baoquan026.cn/11779.Doc
dvy.baoquan026.cn/71197.Doc
dvy.baoquan026.cn/99979.Doc
dvy.baoquan026.cn/59393.Doc
dvy.baoquan026.cn/19177.Doc
dvy.baoquan026.cn/53971.Doc
dvy.baoquan026.cn/13931.Doc
dvy.baoquan026.cn/17357.Doc
dvy.baoquan026.cn/79755.Doc
dvy.baoquan026.cn/75599.Doc
dvy.baoquan026.cn/75931.Doc
dvt.baoquan026.cn/97399.Doc
dvt.baoquan026.cn/51371.Doc
dvt.baoquan026.cn/17933.Doc
dvt.baoquan026.cn/39553.Doc
dvt.baoquan026.cn/42848.Doc
dvt.baoquan026.cn/79151.Doc
dvt.baoquan026.cn/37117.Doc
dvt.baoquan026.cn/59355.Doc
dvt.baoquan026.cn/51175.Doc
dvt.baoquan026.cn/33991.Doc
dvr.baoquan026.cn/39931.Doc
dvr.baoquan026.cn/11399.Doc
dvr.baoquan026.cn/15575.Doc
dvr.baoquan026.cn/97193.Doc
dvr.baoquan026.cn/51171.Doc
dvr.baoquan026.cn/39359.Doc
dvr.baoquan026.cn/77113.Doc
dvr.baoquan026.cn/73519.Doc
dvr.baoquan026.cn/77379.Doc
dvr.baoquan026.cn/99175.Doc
dve.baoquan026.cn/37313.Doc
dve.baoquan026.cn/55599.Doc
dve.baoquan026.cn/53171.Doc
dve.baoquan026.cn/22628.Doc
dve.baoquan026.cn/17359.Doc
dve.baoquan026.cn/15359.Doc
dve.baoquan026.cn/75993.Doc
dve.baoquan026.cn/15757.Doc
dve.baoquan026.cn/73193.Doc
dve.baoquan026.cn/91919.Doc
dvw.baoquan026.cn/79513.Doc
dvw.baoquan026.cn/77757.Doc
dvw.baoquan026.cn/35157.Doc
dvw.baoquan026.cn/91391.Doc
dvw.baoquan026.cn/19157.Doc
dvw.baoquan026.cn/95977.Doc
dvw.baoquan026.cn/79979.Doc
dvw.baoquan026.cn/79191.Doc
dvw.baoquan026.cn/93533.Doc
dvw.baoquan026.cn/97793.Doc
dvq.baoquan026.cn/93173.Doc
dvq.baoquan026.cn/15519.Doc
dvq.baoquan026.cn/44448.Doc
dvq.baoquan026.cn/75335.Doc
dvq.baoquan026.cn/97337.Doc
dvq.baoquan026.cn/57999.Doc
dvq.baoquan026.cn/51777.Doc
dvq.baoquan026.cn/15359.Doc
dvq.baoquan026.cn/77375.Doc
dvq.baoquan026.cn/31173.Doc
dcm.baoquan026.cn/97191.Doc
dcm.baoquan026.cn/15777.Doc
dcm.baoquan026.cn/53139.Doc
dcm.baoquan026.cn/93513.Doc
dcm.baoquan026.cn/31799.Doc
dcm.baoquan026.cn/35331.Doc
dcm.baoquan026.cn/33115.Doc
dcm.baoquan026.cn/57397.Doc
dcm.baoquan026.cn/35571.Doc
dcm.baoquan026.cn/39513.Doc
dcn.baoquan026.cn/66684.Doc
dcn.baoquan026.cn/51337.Doc
dcn.baoquan026.cn/57777.Doc
dcn.baoquan026.cn/99357.Doc
dcn.baoquan026.cn/75537.Doc
dcn.baoquan026.cn/91515.Doc
dcn.baoquan026.cn/35317.Doc
dcn.baoquan026.cn/93755.Doc
dcn.baoquan026.cn/08444.Doc
dcn.baoquan026.cn/55155.Doc
dcb.baoquan026.cn/53999.Doc
dcb.baoquan026.cn/15371.Doc
dcb.baoquan026.cn/19953.Doc
dcb.baoquan026.cn/15771.Doc
dcb.baoquan026.cn/95331.Doc
dcb.baoquan026.cn/99355.Doc
dcb.baoquan026.cn/11191.Doc
dcb.baoquan026.cn/77399.Doc
dcb.baoquan026.cn/35751.Doc
dcb.baoquan026.cn/57513.Doc
dcv.baoquan026.cn/19313.Doc
dcv.baoquan026.cn/51119.Doc
dcv.baoquan026.cn/57713.Doc
dcv.baoquan026.cn/59931.Doc
dcv.baoquan026.cn/11133.Doc
dcv.baoquan026.cn/99157.Doc
dcv.baoquan026.cn/51753.Doc
dcv.baoquan026.cn/99977.Doc
dcv.baoquan026.cn/57733.Doc
dcv.baoquan026.cn/15731.Doc
dcc.baoquan026.cn/37137.Doc
dcc.baoquan026.cn/13375.Doc
dcc.baoquan026.cn/37535.Doc
dcc.baoquan026.cn/33191.Doc
dcc.baoquan026.cn/57911.Doc
dcc.baoquan026.cn/91799.Doc
dcc.baoquan026.cn/97777.Doc
dcc.baoquan026.cn/19335.Doc
dcc.baoquan026.cn/00642.Doc
dcc.baoquan026.cn/75333.Doc
dcx.baoquan026.cn/55511.Doc
dcx.baoquan026.cn/93111.Doc
dcx.baoquan026.cn/73595.Doc
dcx.baoquan026.cn/93933.Doc
dcx.baoquan026.cn/13791.Doc
dcx.baoquan026.cn/13375.Doc
dcx.baoquan026.cn/79971.Doc
dcx.baoquan026.cn/95135.Doc
dcx.baoquan026.cn/73537.Doc
dcx.baoquan026.cn/71759.Doc
dcz.baoquan026.cn/53731.Doc
dcz.baoquan026.cn/95131.Doc
dcz.baoquan026.cn/19597.Doc
dcz.baoquan026.cn/39771.Doc
dcz.baoquan026.cn/55973.Doc
dcz.baoquan026.cn/51719.Doc
dcz.baoquan026.cn/77953.Doc
dcz.baoquan026.cn/97353.Doc
dcz.baoquan026.cn/93777.Doc
dcz.baoquan026.cn/82080.Doc
dcl.baoquan026.cn/62080.Doc
dcl.baoquan026.cn/68484.Doc
dcl.baoquan026.cn/64246.Doc
dcl.baoquan026.cn/04626.Doc
dcl.baoquan026.cn/20200.Doc
dcl.baoquan026.cn/42868.Doc
dcl.baoquan026.cn/48202.Doc
dcl.baoquan026.cn/20266.Doc
dcl.baoquan026.cn/24202.Doc
dcl.baoquan026.cn/84628.Doc
dck.baoquan026.cn/33759.Doc
dck.baoquan026.cn/62624.Doc
dck.baoquan026.cn/00460.Doc
dck.baoquan026.cn/04844.Doc
dck.baoquan026.cn/46260.Doc
dck.baoquan026.cn/26244.Doc
dck.baoquan026.cn/08046.Doc
dck.baoquan026.cn/62682.Doc
dck.baoquan026.cn/42202.Doc
dck.baoquan026.cn/06846.Doc
dcj.baoquan026.cn/40428.Doc
dcj.baoquan026.cn/20284.Doc
dcj.baoquan026.cn/48668.Doc
dcj.baoquan026.cn/64066.Doc
dcj.baoquan026.cn/80626.Doc
dcj.baoquan026.cn/04608.Doc
dcj.baoquan026.cn/08802.Doc
dcj.baoquan026.cn/60608.Doc
dcj.baoquan026.cn/28288.Doc
dcj.baoquan026.cn/20062.Doc
dch.baoquan026.cn/64406.Doc
dch.baoquan026.cn/42088.Doc
dch.baoquan026.cn/68806.Doc
dch.baoquan026.cn/82884.Doc
dch.baoquan026.cn/04462.Doc
dch.baoquan026.cn/46284.Doc
dch.baoquan026.cn/22086.Doc
dch.baoquan026.cn/64088.Doc
dch.baoquan026.cn/06604.Doc
dch.baoquan026.cn/82204.Doc
dcg.baoquan026.cn/60888.Doc
dcg.baoquan026.cn/84240.Doc
dcg.baoquan026.cn/42248.Doc
dcg.baoquan026.cn/82800.Doc
dcg.baoquan026.cn/84840.Doc
dcg.baoquan026.cn/28282.Doc
dcg.baoquan026.cn/64666.Doc
dcg.baoquan026.cn/08864.Doc
dcg.baoquan026.cn/44280.Doc
dcg.baoquan026.cn/04080.Doc
dcf.baoquan026.cn/68602.Doc
dcf.baoquan026.cn/82866.Doc
dcf.baoquan026.cn/04046.Doc
dcf.baoquan026.cn/22264.Doc
dcf.baoquan026.cn/66266.Doc
dcf.baoquan026.cn/46820.Doc
dcf.baoquan026.cn/20822.Doc
dcf.baoquan026.cn/26064.Doc
dcf.baoquan026.cn/88226.Doc
dcf.baoquan026.cn/40042.Doc
