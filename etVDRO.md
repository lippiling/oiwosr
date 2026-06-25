
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

wjc.irampL.cn/62228.Doc
wjc.irampL.cn/28224.Doc
wjc.irampL.cn/84248.Doc
wjc.irampL.cn/88002.Doc
wjc.irampL.cn/40444.Doc
wjc.irampL.cn/02086.Doc
wjc.irampL.cn/06602.Doc
wjc.irampL.cn/02240.Doc
wjc.irampL.cn/08608.Doc
wjc.irampL.cn/26440.Doc
wjx.irampL.cn/46044.Doc
wjx.irampL.cn/48666.Doc
wjx.irampL.cn/39531.Doc
wjx.irampL.cn/88442.Doc
wjx.irampL.cn/68462.Doc
wjx.irampL.cn/46448.Doc
wjx.irampL.cn/22860.Doc
wjx.irampL.cn/08600.Doc
wjx.irampL.cn/00040.Doc
wjx.irampL.cn/44424.Doc
wjz.irampL.cn/40088.Doc
wjz.irampL.cn/22206.Doc
wjz.irampL.cn/08626.Doc
wjz.irampL.cn/62404.Doc
wjz.irampL.cn/26464.Doc
wjz.irampL.cn/26000.Doc
wjz.irampL.cn/00280.Doc
wjz.irampL.cn/35151.Doc
wjz.irampL.cn/64808.Doc
wjz.irampL.cn/24044.Doc
wjl.irampL.cn/42020.Doc
wjl.irampL.cn/13319.Doc
wjl.irampL.cn/20880.Doc
wjl.irampL.cn/46484.Doc
wjl.irampL.cn/60028.Doc
wjl.irampL.cn/80424.Doc
wjl.irampL.cn/86466.Doc
wjl.irampL.cn/06004.Doc
wjl.irampL.cn/80800.Doc
wjl.irampL.cn/06280.Doc
wjk.irampL.cn/71159.Doc
wjk.irampL.cn/46688.Doc
wjk.irampL.cn/66262.Doc
wjk.irampL.cn/48226.Doc
wjk.irampL.cn/42864.Doc
wjk.irampL.cn/28620.Doc
wjk.irampL.cn/00006.Doc
wjk.irampL.cn/84286.Doc
wjk.irampL.cn/26822.Doc
wjk.irampL.cn/57771.Doc
wjj.irampL.cn/97953.Doc
wjj.irampL.cn/20808.Doc
wjj.irampL.cn/99935.Doc
wjj.irampL.cn/88000.Doc
wjj.irampL.cn/44444.Doc
wjj.irampL.cn/22880.Doc
wjj.irampL.cn/22664.Doc
wjj.irampL.cn/22600.Doc
wjj.irampL.cn/06444.Doc
wjj.irampL.cn/40266.Doc
wjh.irampL.cn/60240.Doc
wjh.irampL.cn/84884.Doc
wjh.irampL.cn/64428.Doc
wjh.irampL.cn/00842.Doc
wjh.irampL.cn/44482.Doc
wjh.irampL.cn/44000.Doc
wjh.irampL.cn/08682.Doc
wjh.irampL.cn/04600.Doc
wjh.irampL.cn/33937.Doc
wjh.irampL.cn/80806.Doc
wjg.irampL.cn/04284.Doc
wjg.irampL.cn/68224.Doc
wjg.irampL.cn/80462.Doc
wjg.irampL.cn/48682.Doc
wjg.irampL.cn/20406.Doc
wjg.irampL.cn/51795.Doc
wjg.irampL.cn/44008.Doc
wjg.irampL.cn/82444.Doc
wjg.irampL.cn/88002.Doc
wjg.irampL.cn/06280.Doc
wjf.irampL.cn/00688.Doc
wjf.irampL.cn/93915.Doc
wjf.irampL.cn/64204.Doc
wjf.irampL.cn/68826.Doc
wjf.irampL.cn/64862.Doc
wjf.irampL.cn/64226.Doc
wjf.irampL.cn/82424.Doc
wjf.irampL.cn/00444.Doc
wjf.irampL.cn/44406.Doc
wjf.irampL.cn/22466.Doc
wjd.irampL.cn/08442.Doc
wjd.irampL.cn/53717.Doc
wjd.irampL.cn/40886.Doc
wjd.irampL.cn/99157.Doc
wjd.irampL.cn/02068.Doc
wjd.irampL.cn/68468.Doc
wjd.irampL.cn/26666.Doc
wjd.irampL.cn/46464.Doc
wjd.irampL.cn/44464.Doc
wjd.irampL.cn/40026.Doc
wjs.irampL.cn/84820.Doc
wjs.irampL.cn/62204.Doc
wjs.irampL.cn/20468.Doc
wjs.irampL.cn/46200.Doc
wjs.irampL.cn/66008.Doc
wjs.irampL.cn/06024.Doc
wjs.irampL.cn/20400.Doc
wjs.irampL.cn/00460.Doc
wjs.irampL.cn/42224.Doc
wjs.irampL.cn/06046.Doc
wja.irampL.cn/62046.Doc
wja.irampL.cn/46042.Doc
wja.irampL.cn/82024.Doc
wja.irampL.cn/00688.Doc
wja.irampL.cn/28040.Doc
wja.irampL.cn/28802.Doc
wja.irampL.cn/28426.Doc
wja.irampL.cn/95571.Doc
wja.irampL.cn/66682.Doc
wja.irampL.cn/88664.Doc
wjp.irampL.cn/46420.Doc
wjp.irampL.cn/42002.Doc
wjp.irampL.cn/84888.Doc
wjp.irampL.cn/82844.Doc
wjp.irampL.cn/20800.Doc
wjp.irampL.cn/19339.Doc
wjp.irampL.cn/42488.Doc
wjp.irampL.cn/64406.Doc
wjp.irampL.cn/06488.Doc
wjp.irampL.cn/84026.Doc
wjo.irampL.cn/80082.Doc
wjo.irampL.cn/82448.Doc
wjo.irampL.cn/91535.Doc
wjo.irampL.cn/48206.Doc
wjo.irampL.cn/62640.Doc
wjo.irampL.cn/00046.Doc
wjo.irampL.cn/71553.Doc
wjo.irampL.cn/13339.Doc
wjo.irampL.cn/88082.Doc
wjo.irampL.cn/53537.Doc
wji.irampL.cn/20820.Doc
wji.irampL.cn/28242.Doc
wji.irampL.cn/64008.Doc
wji.irampL.cn/73131.Doc
wji.irampL.cn/77157.Doc
wji.irampL.cn/40064.Doc
wji.irampL.cn/64666.Doc
wji.irampL.cn/79539.Doc
wji.irampL.cn/04206.Doc
wji.irampL.cn/97515.Doc
wju.irampL.cn/06862.Doc
wju.irampL.cn/02668.Doc
wju.irampL.cn/62626.Doc
wju.irampL.cn/24004.Doc
wju.irampL.cn/57171.Doc
wju.irampL.cn/62226.Doc
wju.irampL.cn/71757.Doc
wju.irampL.cn/60268.Doc
wju.irampL.cn/06428.Doc
wju.irampL.cn/20208.Doc
wjy.irampL.cn/22420.Doc
wjy.irampL.cn/93773.Doc
wjy.irampL.cn/24008.Doc
wjy.irampL.cn/33375.Doc
wjy.irampL.cn/40624.Doc
wjy.irampL.cn/66460.Doc
wjy.irampL.cn/04288.Doc
wjy.irampL.cn/24604.Doc
wjy.irampL.cn/86244.Doc
wjy.irampL.cn/60204.Doc
wjt.irampL.cn/60842.Doc
wjt.irampL.cn/64262.Doc
wjt.irampL.cn/82284.Doc
wjt.irampL.cn/82008.Doc
wjt.irampL.cn/44408.Doc
wjt.irampL.cn/84000.Doc
wjt.irampL.cn/28242.Doc
wjt.irampL.cn/42402.Doc
wjt.irampL.cn/88640.Doc
wjt.irampL.cn/46248.Doc
wjr.irampL.cn/06604.Doc
wjr.irampL.cn/91333.Doc
wjr.irampL.cn/28428.Doc
wjr.irampL.cn/40842.Doc
wjr.irampL.cn/80684.Doc
wjr.irampL.cn/40488.Doc
wjr.irampL.cn/08006.Doc
wjr.irampL.cn/24622.Doc
wjr.irampL.cn/04866.Doc
wjr.irampL.cn/66464.Doc
wje.irampL.cn/40426.Doc
wje.irampL.cn/44402.Doc
wje.irampL.cn/86422.Doc
wje.irampL.cn/04608.Doc
wje.irampL.cn/82002.Doc
wje.irampL.cn/68420.Doc
wje.irampL.cn/88842.Doc
wje.irampL.cn/84406.Doc
wje.irampL.cn/24024.Doc
wje.irampL.cn/13557.Doc
wjw.irampL.cn/48048.Doc
wjw.irampL.cn/62802.Doc
wjw.irampL.cn/80848.Doc
wjw.irampL.cn/64424.Doc
wjw.irampL.cn/80224.Doc
wjw.irampL.cn/00646.Doc
wjw.irampL.cn/88062.Doc
wjw.irampL.cn/46448.Doc
wjw.irampL.cn/48202.Doc
wjw.irampL.cn/08844.Doc
wjq.irampL.cn/00266.Doc
wjq.irampL.cn/40048.Doc
wjq.irampL.cn/86840.Doc
wjq.irampL.cn/40020.Doc
wjq.irampL.cn/24222.Doc
wjq.irampL.cn/62622.Doc
wjq.irampL.cn/86608.Doc
wjq.irampL.cn/48088.Doc
wjq.irampL.cn/00624.Doc
wjq.irampL.cn/84064.Doc
whm.irampL.cn/55551.Doc
whm.irampL.cn/24828.Doc
whm.irampL.cn/48664.Doc
whm.irampL.cn/60066.Doc
whm.irampL.cn/68228.Doc
whm.irampL.cn/84462.Doc
whm.irampL.cn/39319.Doc
whm.irampL.cn/68008.Doc
whm.irampL.cn/06046.Doc
whm.irampL.cn/86820.Doc
whn.irampL.cn/80604.Doc
whn.irampL.cn/13159.Doc
whn.irampL.cn/40842.Doc
whn.irampL.cn/80884.Doc
whn.irampL.cn/68824.Doc
whn.irampL.cn/66640.Doc
whn.irampL.cn/28444.Doc
whn.irampL.cn/42042.Doc
whn.irampL.cn/28082.Doc
whn.irampL.cn/82864.Doc
whb.irampL.cn/66282.Doc
whb.irampL.cn/13131.Doc
whb.irampL.cn/42224.Doc
whb.irampL.cn/64086.Doc
whb.irampL.cn/00266.Doc
whb.irampL.cn/28664.Doc
whb.irampL.cn/53375.Doc
whb.irampL.cn/91353.Doc
whb.irampL.cn/99797.Doc
whb.irampL.cn/86664.Doc
whv.irampL.cn/51535.Doc
whv.irampL.cn/24864.Doc
whv.irampL.cn/88804.Doc
whv.irampL.cn/62022.Doc
whv.irampL.cn/06426.Doc
whv.irampL.cn/88228.Doc
whv.irampL.cn/66064.Doc
whv.irampL.cn/46824.Doc
whv.irampL.cn/88080.Doc
whv.irampL.cn/42862.Doc
whc.irampL.cn/68864.Doc
whc.irampL.cn/48864.Doc
whc.irampL.cn/86248.Doc
whc.irampL.cn/88226.Doc
whc.irampL.cn/95775.Doc
whc.irampL.cn/46820.Doc
whc.irampL.cn/28806.Doc
whc.irampL.cn/66486.Doc
whc.irampL.cn/68804.Doc
whc.irampL.cn/02804.Doc
whx.irampL.cn/24042.Doc
whx.irampL.cn/19175.Doc
whx.irampL.cn/37519.Doc
whx.irampL.cn/35184.Doc
whx.irampL.cn/04242.Doc
whx.irampL.cn/24486.Doc
whx.irampL.cn/06880.Doc
whx.irampL.cn/80604.Doc
whx.irampL.cn/66240.Doc
whx.irampL.cn/24446.Doc
whz.irampL.cn/00862.Doc
whz.irampL.cn/68462.Doc
whz.irampL.cn/00604.Doc
whz.irampL.cn/17159.Doc
whz.irampL.cn/80466.Doc
whz.irampL.cn/04008.Doc
whz.irampL.cn/20084.Doc
whz.irampL.cn/48642.Doc
whz.irampL.cn/48402.Doc
whz.irampL.cn/68244.Doc
whl.irampL.cn/26280.Doc
whl.irampL.cn/20426.Doc
whl.irampL.cn/06648.Doc
whl.irampL.cn/86268.Doc
whl.irampL.cn/48028.Doc
whl.irampL.cn/44440.Doc
whl.irampL.cn/91735.Doc
whl.irampL.cn/40288.Doc
whl.irampL.cn/06262.Doc
whl.irampL.cn/97551.Doc
whk.irampL.cn/04840.Doc
whk.irampL.cn/44648.Doc
whk.irampL.cn/42244.Doc
whk.irampL.cn/88208.Doc
whk.irampL.cn/62806.Doc
whk.irampL.cn/82686.Doc
whk.irampL.cn/04220.Doc
whk.irampL.cn/88646.Doc
whk.irampL.cn/26628.Doc
whk.irampL.cn/48046.Doc
whj.irampL.cn/86288.Doc
whj.irampL.cn/86006.Doc
whj.irampL.cn/55173.Doc
whj.irampL.cn/97979.Doc
whj.irampL.cn/60420.Doc
whj.irampL.cn/71755.Doc
whj.irampL.cn/06028.Doc
whj.irampL.cn/64662.Doc
whj.irampL.cn/24040.Doc
whj.irampL.cn/68444.Doc
whh.irampL.cn/06280.Doc
whh.irampL.cn/80608.Doc
whh.irampL.cn/00804.Doc
whh.irampL.cn/04824.Doc
whh.irampL.cn/35551.Doc
whh.irampL.cn/15917.Doc
whh.irampL.cn/08862.Doc
whh.irampL.cn/64882.Doc
whh.irampL.cn/91917.Doc
whh.irampL.cn/15717.Doc
whg.irampL.cn/80042.Doc
whg.irampL.cn/68886.Doc
whg.irampL.cn/66266.Doc
whg.irampL.cn/64246.Doc
whg.irampL.cn/48800.Doc
whg.irampL.cn/44646.Doc
whg.irampL.cn/46246.Doc
whg.irampL.cn/48668.Doc
whg.irampL.cn/62024.Doc
whg.irampL.cn/59995.Doc
whf.irampL.cn/57953.Doc
whf.irampL.cn/40620.Doc
whf.irampL.cn/40666.Doc
whf.irampL.cn/28284.Doc
whf.irampL.cn/24440.Doc
whf.irampL.cn/08224.Doc
whf.irampL.cn/68266.Doc
whf.irampL.cn/68462.Doc
whf.irampL.cn/42042.Doc
whf.irampL.cn/26680.Doc
whd.irampL.cn/40808.Doc
whd.irampL.cn/80286.Doc
whd.irampL.cn/60822.Doc
whd.irampL.cn/00008.Doc
whd.irampL.cn/88482.Doc
whd.irampL.cn/06046.Doc
whd.irampL.cn/00406.Doc
whd.irampL.cn/22468.Doc
whd.irampL.cn/44044.Doc
whd.irampL.cn/42446.Doc
whs.irampL.cn/22262.Doc
whs.irampL.cn/68008.Doc
whs.irampL.cn/22488.Doc
whs.irampL.cn/00228.Doc
whs.irampL.cn/66242.Doc
whs.irampL.cn/60280.Doc
whs.irampL.cn/95571.Doc
whs.irampL.cn/64640.Doc
whs.irampL.cn/62844.Doc
whs.irampL.cn/28880.Doc
wha.irampL.cn/06464.Doc
wha.irampL.cn/44228.Doc
wha.irampL.cn/77391.Doc
wha.irampL.cn/40886.Doc
wha.irampL.cn/48402.Doc
wha.irampL.cn/88066.Doc
wha.irampL.cn/26088.Doc
wha.irampL.cn/00404.Doc
wha.irampL.cn/80808.Doc
wha.irampL.cn/00488.Doc
whp.irampL.cn/20420.Doc
whp.irampL.cn/17753.Doc
whp.irampL.cn/88846.Doc
whp.irampL.cn/84044.Doc
whp.irampL.cn/24826.Doc
whp.irampL.cn/06222.Doc
whp.irampL.cn/28660.Doc
whp.irampL.cn/08042.Doc
whp.irampL.cn/08426.Doc
whp.irampL.cn/82600.Doc
who.irampL.cn/26246.Doc
who.irampL.cn/24686.Doc
who.irampL.cn/60466.Doc
who.irampL.cn/44022.Doc
who.irampL.cn/20884.Doc
who.irampL.cn/33599.Doc
who.irampL.cn/66286.Doc
who.irampL.cn/66400.Doc
who.irampL.cn/13911.Doc
who.irampL.cn/44622.Doc
whi.irampL.cn/06622.Doc
whi.irampL.cn/48444.Doc
whi.irampL.cn/88024.Doc
whi.irampL.cn/06840.Doc
whi.irampL.cn/68468.Doc
whi.irampL.cn/02206.Doc
whi.irampL.cn/28088.Doc
whi.irampL.cn/24204.Doc
whi.irampL.cn/00242.Doc
whi.irampL.cn/80824.Doc
whu.irampL.cn/20660.Doc
whu.irampL.cn/46826.Doc
whu.irampL.cn/62068.Doc
whu.irampL.cn/86428.Doc
whu.irampL.cn/00800.Doc
whu.irampL.cn/62262.Doc
whu.irampL.cn/82448.Doc
whu.irampL.cn/62280.Doc
whu.irampL.cn/44022.Doc
whu.irampL.cn/68808.Doc
why.irampL.cn/64646.Doc
why.irampL.cn/42282.Doc
why.irampL.cn/19133.Doc
why.irampL.cn/04846.Doc
why.irampL.cn/64268.Doc
why.irampL.cn/37795.Doc
why.irampL.cn/55771.Doc
why.irampL.cn/48688.Doc
why.irampL.cn/11993.Doc
why.irampL.cn/02028.Doc
wht.irampL.cn/06028.Doc
wht.irampL.cn/46248.Doc
wht.irampL.cn/84220.Doc
wht.irampL.cn/64040.Doc
wht.irampL.cn/02808.Doc
wht.irampL.cn/08480.Doc
wht.irampL.cn/64606.Doc
wht.irampL.cn/48426.Doc
wht.irampL.cn/26886.Doc
wht.irampL.cn/06620.Doc
whr.irampL.cn/86204.Doc
whr.irampL.cn/86626.Doc
whr.irampL.cn/26060.Doc
whr.irampL.cn/06288.Doc
whr.irampL.cn/68404.Doc
whr.irampL.cn/46082.Doc
whr.irampL.cn/40860.Doc
whr.irampL.cn/51373.Doc
whr.irampL.cn/80868.Doc
whr.irampL.cn/66600.Doc
whe.irampL.cn/02444.Doc
whe.irampL.cn/40808.Doc
whe.irampL.cn/73179.Doc
whe.irampL.cn/08464.Doc
whe.irampL.cn/42804.Doc
whe.irampL.cn/60404.Doc
whe.irampL.cn/82086.Doc
whe.irampL.cn/80886.Doc
whe.irampL.cn/48042.Doc
whe.irampL.cn/35571.Doc
whw.irampL.cn/80824.Doc
whw.irampL.cn/88080.Doc
whw.irampL.cn/55531.Doc
whw.irampL.cn/22844.Doc
whw.irampL.cn/22442.Doc
whw.irampL.cn/88840.Doc
whw.irampL.cn/66444.Doc
whw.irampL.cn/20020.Doc
whw.irampL.cn/02204.Doc
whw.irampL.cn/77773.Doc
whq.irampL.cn/04426.Doc
whq.irampL.cn/24060.Doc
whq.irampL.cn/44802.Doc
whq.irampL.cn/60846.Doc
whq.irampL.cn/33717.Doc
whq.irampL.cn/08222.Doc
whq.irampL.cn/82602.Doc
whq.irampL.cn/84400.Doc
whq.irampL.cn/40664.Doc
whq.irampL.cn/46680.Doc
wgm.irampL.cn/33139.Doc
wgm.irampL.cn/40462.Doc
wgm.irampL.cn/24246.Doc
wgm.irampL.cn/82844.Doc
wgm.irampL.cn/39319.Doc
wgm.irampL.cn/68602.Doc
wgm.irampL.cn/48840.Doc
wgm.irampL.cn/46646.Doc
wgm.irampL.cn/06688.Doc
wgm.irampL.cn/20220.Doc
