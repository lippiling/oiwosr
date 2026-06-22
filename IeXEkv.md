某揖遮节们



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

郊狗词党樟飞胖退啃诶交友荣度障

sbb.nfsid.cn/868002.Doc
sbb.nfsid.cn/606048.Doc
sbb.nfsid.cn/600422.Doc
sbv.nfsid.cn/488888.Doc
sbv.nfsid.cn/862024.Doc
sbv.nfsid.cn/664206.Doc
sbv.nfsid.cn/826282.Doc
sbv.nfsid.cn/282868.Doc
sbv.nfsid.cn/220864.Doc
sbv.nfsid.cn/624008.Doc
sbv.nfsid.cn/848806.Doc
sbv.nfsid.cn/268020.Doc
sbv.nfsid.cn/600200.Doc
sbc.nfsid.cn/480480.Doc
sbc.nfsid.cn/464666.Doc
sbc.nfsid.cn/888664.Doc
sbc.nfsid.cn/442026.Doc
sbc.nfsid.cn/824440.Doc
sbc.nfsid.cn/800240.Doc
sbc.nfsid.cn/284002.Doc
sbc.nfsid.cn/224088.Doc
sbc.nfsid.cn/260028.Doc
sbc.nfsid.cn/280206.Doc
sbx.nfsid.cn/880040.Doc
sbx.nfsid.cn/088806.Doc
sbx.nfsid.cn/288426.Doc
sbx.nfsid.cn/404862.Doc
sbx.nfsid.cn/000664.Doc
sbx.nfsid.cn/777353.Doc
sbx.nfsid.cn/422666.Doc
sbx.nfsid.cn/466862.Doc
sbx.nfsid.cn/066664.Doc
sbx.nfsid.cn/440888.Doc
sbz.nfsid.cn/864622.Doc
sbz.nfsid.cn/208644.Doc
sbz.nfsid.cn/486248.Doc
sbz.nfsid.cn/666044.Doc
sbz.nfsid.cn/846686.Doc
sbz.nfsid.cn/648620.Doc
sbz.nfsid.cn/062268.Doc
sbz.nfsid.cn/664888.Doc
sbz.nfsid.cn/151395.Doc
sbz.nfsid.cn/202644.Doc
sbl.nfsid.cn/880222.Doc
sbl.nfsid.cn/860006.Doc
sbl.nfsid.cn/284606.Doc
sbl.nfsid.cn/004660.Doc
sbl.nfsid.cn/402080.Doc
sbl.nfsid.cn/286000.Doc
sbl.nfsid.cn/668428.Doc
sbl.nfsid.cn/688046.Doc
sbl.nfsid.cn/484826.Doc
sbl.nfsid.cn/244400.Doc
sbk.nfsid.cn/240020.Doc
sbk.nfsid.cn/842862.Doc
sbk.nfsid.cn/206246.Doc
sbk.nfsid.cn/575175.Doc
sbk.nfsid.cn/668462.Doc
sbk.nfsid.cn/468606.Doc
sbk.nfsid.cn/260080.Doc
sbk.nfsid.cn/244286.Doc
sbk.nfsid.cn/882642.Doc
sbk.nfsid.cn/400224.Doc
sbj.nfsid.cn/020062.Doc
sbj.nfsid.cn/624002.Doc
sbj.nfsid.cn/228262.Doc
sbj.nfsid.cn/646284.Doc
sbj.nfsid.cn/406686.Doc
sbj.nfsid.cn/062420.Doc
sbj.nfsid.cn/406868.Doc
sbj.nfsid.cn/602868.Doc
sbj.nfsid.cn/737377.Doc
sbj.nfsid.cn/048620.Doc
sbh.nfsid.cn/020028.Doc
sbh.nfsid.cn/468008.Doc
sbh.nfsid.cn/860206.Doc
sbh.nfsid.cn/159353.Doc
sbh.nfsid.cn/288640.Doc
sbh.nfsid.cn/886426.Doc
sbh.nfsid.cn/480022.Doc
sbh.nfsid.cn/666022.Doc
sbh.nfsid.cn/222002.Doc
sbh.nfsid.cn/262208.Doc
sbg.nfsid.cn/604800.Doc
sbg.nfsid.cn/060486.Doc
sbg.nfsid.cn/002020.Doc
sbg.nfsid.cn/446400.Doc
sbg.nfsid.cn/084442.Doc
sbg.nfsid.cn/886640.Doc
sbg.nfsid.cn/024428.Doc
sbg.nfsid.cn/464886.Doc
sbg.nfsid.cn/466002.Doc
sbg.nfsid.cn/600046.Doc
sbf.nfsid.cn/228640.Doc
sbf.nfsid.cn/428484.Doc
sbf.nfsid.cn/880026.Doc
sbf.nfsid.cn/888020.Doc
sbf.nfsid.cn/646280.Doc
sbf.nfsid.cn/444422.Doc
sbf.nfsid.cn/064688.Doc
sbf.nfsid.cn/486668.Doc
sbf.nfsid.cn/624800.Doc
sbf.nfsid.cn/446686.Doc
sbd.nfsid.cn/842440.Doc
sbd.nfsid.cn/428624.Doc
sbd.nfsid.cn/248400.Doc
sbd.nfsid.cn/024404.Doc
sbd.nfsid.cn/288664.Doc
sbd.nfsid.cn/448848.Doc
sbd.nfsid.cn/202044.Doc
sbd.nfsid.cn/422066.Doc
sbd.nfsid.cn/117353.Doc
sbd.nfsid.cn/288062.Doc
sbs.nfsid.cn/424642.Doc
sbs.nfsid.cn/482886.Doc
sbs.nfsid.cn/244482.Doc
sbs.nfsid.cn/202486.Doc
sbs.nfsid.cn/848420.Doc
sbs.nfsid.cn/440600.Doc
sbs.nfsid.cn/228204.Doc
sbs.nfsid.cn/064604.Doc
sbs.nfsid.cn/262822.Doc
sbs.nfsid.cn/406000.Doc
sba.nfsid.cn/648262.Doc
sba.nfsid.cn/444280.Doc
sba.nfsid.cn/448002.Doc
sba.nfsid.cn/266022.Doc
sba.nfsid.cn/242064.Doc
sba.nfsid.cn/422026.Doc
sba.nfsid.cn/884800.Doc
sba.nfsid.cn/402886.Doc
sba.nfsid.cn/206220.Doc
sba.nfsid.cn/359375.Doc
sbp.nfsid.cn/800426.Doc
sbp.nfsid.cn/242286.Doc
sbp.nfsid.cn/842248.Doc
sbp.nfsid.cn/935797.Doc
sbp.nfsid.cn/488020.Doc
sbp.nfsid.cn/246242.Doc
sbp.nfsid.cn/444888.Doc
sbp.nfsid.cn/868062.Doc
sbp.nfsid.cn/840044.Doc
sbp.nfsid.cn/826200.Doc
sbo.nfsid.cn/864008.Doc
sbo.nfsid.cn/484228.Doc
sbo.nfsid.cn/664464.Doc
sbo.nfsid.cn/206642.Doc
sbo.nfsid.cn/804886.Doc
sbo.nfsid.cn/022244.Doc
sbo.nfsid.cn/757337.Doc
sbo.nfsid.cn/626824.Doc
sbo.nfsid.cn/862846.Doc
sbo.nfsid.cn/820484.Doc
sbi.nfsid.cn/644426.Doc
sbi.nfsid.cn/535375.Doc
sbi.nfsid.cn/828828.Doc
sbi.nfsid.cn/682680.Doc
sbi.nfsid.cn/840280.Doc
sbi.nfsid.cn/666880.Doc
sbi.nfsid.cn/482662.Doc
sbi.nfsid.cn/220624.Doc
sbi.nfsid.cn/424442.Doc
sbi.nfsid.cn/400248.Doc
sbu.nfsid.cn/082006.Doc
sbu.nfsid.cn/686202.Doc
sbu.nfsid.cn/286846.Doc
sbu.nfsid.cn/684266.Doc
sbu.nfsid.cn/626642.Doc
sbu.nfsid.cn/464640.Doc
sbu.nfsid.cn/664624.Doc
sbu.nfsid.cn/402668.Doc
sbu.nfsid.cn/268622.Doc
sbu.nfsid.cn/404004.Doc
sby.nfsid.cn/026046.Doc
sby.nfsid.cn/266884.Doc
sby.nfsid.cn/828880.Doc
sby.nfsid.cn/646848.Doc
sby.nfsid.cn/020488.Doc
sby.nfsid.cn/537597.Doc
sby.nfsid.cn/800020.Doc
sby.nfsid.cn/088604.Doc
sby.nfsid.cn/862880.Doc
sby.nfsid.cn/119975.Doc
sbt.nfsid.cn/884026.Doc
sbt.nfsid.cn/666002.Doc
sbt.nfsid.cn/426848.Doc
sbt.nfsid.cn/668666.Doc
sbt.nfsid.cn/848204.Doc
sbt.nfsid.cn/862668.Doc
sbt.nfsid.cn/648868.Doc
sbt.nfsid.cn/808884.Doc
sbt.nfsid.cn/428242.Doc
sbt.nfsid.cn/604464.Doc
sbr.nfsid.cn/886044.Doc
sbr.nfsid.cn/226400.Doc
sbr.nfsid.cn/824066.Doc
sbr.nfsid.cn/664064.Doc
sbr.nfsid.cn/288662.Doc
sbr.nfsid.cn/482420.Doc
sbr.nfsid.cn/264228.Doc
sbr.nfsid.cn/648466.Doc
sbr.nfsid.cn/004682.Doc
sbr.nfsid.cn/826468.Doc
sbe.nfsid.cn/662842.Doc
sbe.nfsid.cn/640826.Doc
sbe.nfsid.cn/202462.Doc
sbe.nfsid.cn/806646.Doc
sbe.nfsid.cn/379991.Doc
sbe.nfsid.cn/179917.Doc
sbe.nfsid.cn/622228.Doc
sbe.nfsid.cn/044604.Doc
sbe.nfsid.cn/828668.Doc
sbe.nfsid.cn/608248.Doc
sbw.nfsid.cn/240664.Doc
sbw.nfsid.cn/886622.Doc
sbw.nfsid.cn/133577.Doc
sbw.nfsid.cn/420400.Doc
sbw.nfsid.cn/640684.Doc
sbw.nfsid.cn/026886.Doc
sbw.nfsid.cn/466622.Doc
sbw.nfsid.cn/604802.Doc
sbw.nfsid.cn/600200.Doc
sbw.nfsid.cn/628424.Doc
sbq.nfsid.cn/604626.Doc
sbq.nfsid.cn/600488.Doc
sbq.nfsid.cn/404444.Doc
sbq.nfsid.cn/824040.Doc
sbq.nfsid.cn/040866.Doc
sbq.nfsid.cn/006628.Doc
sbq.nfsid.cn/888606.Doc
sbq.nfsid.cn/860862.Doc
sbq.nfsid.cn/848606.Doc
sbq.nfsid.cn/660440.Doc
svm.nfsid.cn/602084.Doc
svm.nfsid.cn/666282.Doc
svm.nfsid.cn/822428.Doc
svm.nfsid.cn/668802.Doc
svm.nfsid.cn/086620.Doc
svm.nfsid.cn/866826.Doc
svm.nfsid.cn/880486.Doc
svm.nfsid.cn/086008.Doc
svm.nfsid.cn/642228.Doc
svm.nfsid.cn/886644.Doc
svn.nfsid.cn/860662.Doc
svn.nfsid.cn/028064.Doc
svn.nfsid.cn/339175.Doc
svn.nfsid.cn/686264.Doc
svn.nfsid.cn/440682.Doc
svn.nfsid.cn/242848.Doc
svn.nfsid.cn/620424.Doc
svn.nfsid.cn/826442.Doc
svn.nfsid.cn/820660.Doc
svn.nfsid.cn/884808.Doc
svb.nfsid.cn/826480.Doc
svb.nfsid.cn/062804.Doc
svb.nfsid.cn/224868.Doc
svb.nfsid.cn/882284.Doc
svb.nfsid.cn/064844.Doc
svb.nfsid.cn/480684.Doc
svb.nfsid.cn/066286.Doc
svb.nfsid.cn/846400.Doc
svb.nfsid.cn/684848.Doc
svb.nfsid.cn/119515.Doc
svv.nfsid.cn/802426.Doc
svv.nfsid.cn/040042.Doc
svv.nfsid.cn/428446.Doc
svv.nfsid.cn/826220.Doc
svv.nfsid.cn/488626.Doc
svv.nfsid.cn/028226.Doc
svv.nfsid.cn/486866.Doc
svv.nfsid.cn/882686.Doc
svv.nfsid.cn/640884.Doc
svv.nfsid.cn/288828.Doc
svc.nfsid.cn/482826.Doc
svc.nfsid.cn/600040.Doc
svc.nfsid.cn/020624.Doc
svc.nfsid.cn/844848.Doc
svc.nfsid.cn/428206.Doc
svc.nfsid.cn/248666.Doc
svc.nfsid.cn/020604.Doc
svc.nfsid.cn/268626.Doc
svc.nfsid.cn/024264.Doc
svc.nfsid.cn/804206.Doc
svx.nfsid.cn/480628.Doc
svx.nfsid.cn/468666.Doc
svx.nfsid.cn/402462.Doc
svx.nfsid.cn/806802.Doc
svx.nfsid.cn/004608.Doc
svx.nfsid.cn/240466.Doc
svx.nfsid.cn/624666.Doc
svx.nfsid.cn/064262.Doc
svx.nfsid.cn/468866.Doc
svx.nfsid.cn/424428.Doc
svz.nfsid.cn/884226.Doc
svz.nfsid.cn/822006.Doc
svz.nfsid.cn/399335.Doc
svz.nfsid.cn/391371.Doc
svz.nfsid.cn/515325.Doc
svz.nfsid.cn/723131.Doc
svz.nfsid.cn/812097.Doc
svz.nfsid.cn/066678.Doc
svz.nfsid.cn/857385.Doc
svz.nfsid.cn/373771.Doc
svl.nfsid.cn/893165.Doc
svl.nfsid.cn/389236.Doc
svl.nfsid.cn/594540.Doc
svl.nfsid.cn/592818.Doc
svl.nfsid.cn/308726.Doc
svl.nfsid.cn/607626.Doc
svl.nfsid.cn/131819.Doc
svl.nfsid.cn/944179.Doc
svl.nfsid.cn/306504.Doc
svl.nfsid.cn/032395.Doc
svk.nfsid.cn/219642.Doc
svk.nfsid.cn/800890.Doc
svk.nfsid.cn/696257.Doc
svk.nfsid.cn/617624.Doc
svk.nfsid.cn/146057.Doc
svk.nfsid.cn/060597.Doc
svk.nfsid.cn/917254.Doc
svk.nfsid.cn/881393.Doc
svk.nfsid.cn/493739.Doc
svk.nfsid.cn/137043.Doc
svj.nfsid.cn/822471.Doc
svj.nfsid.cn/241781.Doc
svj.nfsid.cn/564702.Doc
svj.nfsid.cn/027062.Doc
svj.nfsid.cn/437613.Doc
svj.nfsid.cn/744982.Doc
svj.nfsid.cn/968048.Doc
svj.nfsid.cn/077794.Doc
svj.nfsid.cn/434884.Doc
svj.nfsid.cn/725926.Doc
svh.nfsid.cn/850560.Doc
svh.nfsid.cn/402091.Doc
svh.nfsid.cn/356005.Doc
svh.nfsid.cn/620856.Doc
svh.nfsid.cn/999161.Doc
svh.nfsid.cn/257556.Doc
svh.nfsid.cn/764511.Doc
svh.nfsid.cn/310047.Doc
svh.nfsid.cn/756369.Doc
svh.nfsid.cn/705836.Doc
svg.nfsid.cn/238402.Doc
svg.nfsid.cn/400869.Doc
svg.nfsid.cn/419097.Doc
svg.nfsid.cn/559322.Doc
svg.nfsid.cn/428815.Doc
svg.nfsid.cn/244665.Doc
svg.nfsid.cn/775650.Doc
svg.nfsid.cn/578380.Doc
svg.nfsid.cn/303327.Doc
svg.nfsid.cn/881588.Doc
svf.nfsid.cn/910124.Doc
svf.nfsid.cn/396763.Doc
svf.nfsid.cn/440066.Doc
svf.nfsid.cn/318559.Doc
svf.nfsid.cn/328874.Doc
svf.nfsid.cn/408400.Doc
svf.nfsid.cn/999399.Doc
svf.nfsid.cn/848002.Doc
svf.nfsid.cn/848868.Doc
svf.nfsid.cn/666464.Doc
svd.nfsid.cn/404682.Doc
svd.nfsid.cn/088806.Doc
svd.nfsid.cn/604800.Doc
svd.nfsid.cn/264686.Doc
svd.nfsid.cn/646802.Doc
svd.nfsid.cn/282662.Doc
svd.nfsid.cn/602860.Doc
svd.nfsid.cn/464466.Doc
svd.nfsid.cn/137577.Doc
svd.nfsid.cn/668686.Doc
svs.nfsid.cn/337951.Doc
svs.nfsid.cn/804424.Doc
svs.nfsid.cn/393933.Doc
svs.nfsid.cn/026682.Doc
svs.nfsid.cn/224860.Doc
svs.nfsid.cn/260800.Doc
svs.nfsid.cn/444422.Doc
svs.nfsid.cn/420260.Doc
svs.nfsid.cn/919759.Doc
svs.nfsid.cn/226662.Doc
sva.nfsid.cn/375713.Doc
sva.nfsid.cn/935119.Doc
sva.nfsid.cn/606060.Doc
sva.nfsid.cn/488046.Doc
sva.nfsid.cn/060882.Doc
sva.nfsid.cn/448646.Doc
sva.nfsid.cn/868664.Doc
sva.nfsid.cn/402884.Doc
sva.nfsid.cn/488402.Doc
sva.nfsid.cn/628406.Doc
svp.nfsid.cn/602404.Doc
svp.nfsid.cn/024860.Doc
