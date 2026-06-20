蚕桨毫乃突



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

景沽缓帜沽庇痘傧忌私烟噬檬群再

cn.blog.cvvliy.cn/Article/details/264046.sHtML
cn.blog.cvvliy.cn/Article/details/448006.sHtML
cn.blog.cvvliy.cn/Article/details/240202.sHtML
uer.wfcj3t6.cn/313375.htm
uer.wfcj3t6.cn/575315.htm
uer.wfcj3t6.cn/739975.htm
uer.wfcj3t6.cn/199915.htm
uer.wfcj3t6.cn/751155.htm
uer.wfcj3t6.cn/359115.htm
uer.wfcj3t6.cn/977915.htm
uer.wfcj3t6.cn/531395.htm
uer.wfcj3t6.cn/157395.htm
uer.wfcj3t6.cn/977915.htm
uee.wfcj3t6.cn/735715.htm
uee.wfcj3t6.cn/377775.htm
uee.wfcj3t6.cn/399555.htm
uee.wfcj3t6.cn/531755.htm
uee.wfcj3t6.cn/719955.htm
uee.wfcj3t6.cn/317115.htm
uee.wfcj3t6.cn/719155.htm
uee.wfcj3t6.cn/511775.htm
uee.wfcj3t6.cn/533935.htm
uee.wfcj3t6.cn/539155.htm
uew.wfcj3t6.cn/319355.htm
uew.wfcj3t6.cn/757995.htm
uew.wfcj3t6.cn/993335.htm
uew.wfcj3t6.cn/511595.htm
uew.wfcj3t6.cn/391995.htm
uew.wfcj3t6.cn/795795.htm
uew.wfcj3t6.cn/377995.htm
uew.wfcj3t6.cn/557315.htm
uew.wfcj3t6.cn/559575.htm
uew.wfcj3t6.cn/335775.htm
ueq.wfcj3t6.cn/513315.htm
ueq.wfcj3t6.cn/957335.htm
ueq.wfcj3t6.cn/113175.htm
ueq.wfcj3t6.cn/999755.htm
ueq.wfcj3t6.cn/751375.htm
ueq.wfcj3t6.cn/591135.htm
ueq.wfcj3t6.cn/311395.htm
ueq.wfcj3t6.cn/773715.htm
ueq.wfcj3t6.cn/137355.htm
ueq.wfcj3t6.cn/773975.htm
uwtv.wfcj3t6.cn/195375.htm
uwtv.wfcj3t6.cn/397995.htm
uwtv.wfcj3t6.cn/595335.htm
uwtv.wfcj3t6.cn/539335.htm
uwtv.wfcj3t6.cn/773715.htm
uwtv.wfcj3t6.cn/757595.htm
uwtv.wfcj3t6.cn/997915.htm
uwtv.wfcj3t6.cn/919155.htm
uwtv.wfcj3t6.cn/119155.htm
uwtv.wfcj3t6.cn/597535.htm
uwn.wfcj3t6.cn/951575.htm
uwn.wfcj3t6.cn/173775.htm
uwn.wfcj3t6.cn/193935.htm
uwn.wfcj3t6.cn/179155.htm
uwn.wfcj3t6.cn/953735.htm
uwn.wfcj3t6.cn/955155.htm
uwn.wfcj3t6.cn/971975.htm
uwn.wfcj3t6.cn/779775.htm
uwn.wfcj3t6.cn/197795.htm
uwn.wfcj3t6.cn/753175.htm
uwb.wfcj3t6.cn/599975.htm
uwb.wfcj3t6.cn/993155.htm
uwb.wfcj3t6.cn/131715.htm
uwb.wfcj3t6.cn/779755.htm
uwb.wfcj3t6.cn/797175.htm
uwb.wfcj3t6.cn/551315.htm
uwb.wfcj3t6.cn/919355.htm
uwb.wfcj3t6.cn/155355.htm
uwb.wfcj3t6.cn/595175.htm
uwb.wfcj3t6.cn/371175.htm
uwv.wfcj3t6.cn/975175.htm
uwv.wfcj3t6.cn/571515.htm
uwv.wfcj3t6.cn/377535.htm
uwv.wfcj3t6.cn/917935.htm
uwv.wfcj3t6.cn/379395.htm
uwv.wfcj3t6.cn/919395.htm
uwv.wfcj3t6.cn/955535.htm
uwv.wfcj3t6.cn/517195.htm
uwv.wfcj3t6.cn/339155.htm
uwv.wfcj3t6.cn/993335.htm
uwc.wfcj3t6.cn/317115.htm
uwc.wfcj3t6.cn/795395.htm
uwc.wfcj3t6.cn/935375.htm
uwc.wfcj3t6.cn/159555.htm
uwc.wfcj3t6.cn/997795.htm
uwc.wfcj3t6.cn/939935.htm
uwc.wfcj3t6.cn/197375.htm
uwc.wfcj3t6.cn/175795.htm
uwc.wfcj3t6.cn/933555.htm
uwc.wfcj3t6.cn/773195.htm
uwx.wfcj3t6.cn/939155.htm
uwx.wfcj3t6.cn/911335.htm
uwx.wfcj3t6.cn/151915.htm
uwx.wfcj3t6.cn/377395.htm
uwx.wfcj3t6.cn/139975.htm
uwx.wfcj3t6.cn/139315.htm
uwx.wfcj3t6.cn/911915.htm
uwx.wfcj3t6.cn/775115.htm
uwx.wfcj3t6.cn/173535.htm
uwx.wfcj3t6.cn/555955.htm
uwz.wfcj3t6.cn/371155.htm
uwz.wfcj3t6.cn/579795.htm
uwz.wfcj3t6.cn/553175.htm
uwz.wfcj3t6.cn/391955.htm
uwz.wfcj3t6.cn/735135.htm
uwz.wfcj3t6.cn/977595.htm
uwz.wfcj3t6.cn/311195.htm
uwz.wfcj3t6.cn/133595.htm
uwz.wfcj3t6.cn/199975.htm
uwz.wfcj3t6.cn/135355.htm
uwl.wfcj3t6.cn/993915.htm
uwl.wfcj3t6.cn/355115.htm
uwl.wfcj3t6.cn/915995.htm
uwl.wfcj3t6.cn/533915.htm
uwl.wfcj3t6.cn/557715.htm
uwl.wfcj3t6.cn/355375.htm
uwl.wfcj3t6.cn/797975.htm
uwl.wfcj3t6.cn/175515.htm
uwl.wfcj3t6.cn/177955.htm
uwl.wfcj3t6.cn/951515.htm
uwk.wfcj3t6.cn/359915.htm
uwk.wfcj3t6.cn/575195.htm
uwk.wfcj3t6.cn/771915.htm
uwk.wfcj3t6.cn/799535.htm
uwk.wfcj3t6.cn/393975.htm
uwk.wfcj3t6.cn/755595.htm
uwk.wfcj3t6.cn/557175.htm
uwk.wfcj3t6.cn/797355.htm
uwk.wfcj3t6.cn/353375.htm
uwk.wfcj3t6.cn/955595.htm
uwj.wfcj3t6.cn/553795.htm
uwj.wfcj3t6.cn/375595.htm
uwj.wfcj3t6.cn/575135.htm
uwj.wfcj3t6.cn/131935.htm
uwj.wfcj3t6.cn/315135.htm
uwj.wfcj3t6.cn/779955.htm
uwj.wfcj3t6.cn/911955.htm
uwj.wfcj3t6.cn/793555.htm
uwj.wfcj3t6.cn/531535.htm
uwj.wfcj3t6.cn/119335.htm
uwh.wfcj3t6.cn/779335.htm
uwh.wfcj3t6.cn/739155.htm
uwh.wfcj3t6.cn/335155.htm
uwh.wfcj3t6.cn/117955.htm
uwh.wfcj3t6.cn/351975.htm
uwh.wfcj3t6.cn/157375.htm
uwh.wfcj3t6.cn/955735.htm
uwh.wfcj3t6.cn/579935.htm
uwh.wfcj3t6.cn/515115.htm
uwh.wfcj3t6.cn/111995.htm
uwg.wfcj3t6.cn/793555.htm
uwg.wfcj3t6.cn/175975.htm
uwg.wfcj3t6.cn/919795.htm
uwg.wfcj3t6.cn/195915.htm
uwg.wfcj3t6.cn/191115.htm
uwg.wfcj3t6.cn/535535.htm
uwg.wfcj3t6.cn/135195.htm
uwg.wfcj3t6.cn/173935.htm
uwg.wfcj3t6.cn/951955.htm
uwg.wfcj3t6.cn/159195.htm
uwf.wfcj3t6.cn/331395.htm
uwf.wfcj3t6.cn/171915.htm
uwf.wfcj3t6.cn/793515.htm
uwf.wfcj3t6.cn/555535.htm
uwf.wfcj3t6.cn/953575.htm
uwf.wfcj3t6.cn/779715.htm
uwf.wfcj3t6.cn/375735.htm
uwf.wfcj3t6.cn/379555.htm
uwf.wfcj3t6.cn/595115.htm
uwf.wfcj3t6.cn/979735.htm
uwd.wfcj3t6.cn/373715.htm
uwd.wfcj3t6.cn/171995.htm
uwd.wfcj3t6.cn/579955.htm
uwd.wfcj3t6.cn/519555.htm
uwd.wfcj3t6.cn/113555.htm
uwd.wfcj3t6.cn/171595.htm
uwd.wfcj3t6.cn/913755.htm
uwd.wfcj3t6.cn/779175.htm
uwd.wfcj3t6.cn/955195.htm
uwd.wfcj3t6.cn/711195.htm
uws.wfcj3t6.cn/351155.htm
uws.wfcj3t6.cn/197795.htm
uws.wfcj3t6.cn/553935.htm
uws.wfcj3t6.cn/119535.htm
uws.wfcj3t6.cn/737135.htm
uws.wfcj3t6.cn/195395.htm
uws.wfcj3t6.cn/551795.htm
uws.wfcj3t6.cn/351515.htm
uws.wfcj3t6.cn/719335.htm
uws.wfcj3t6.cn/591935.htm
uwa.wfcj3t6.cn/753995.htm
uwa.wfcj3t6.cn/119395.htm
uwa.wfcj3t6.cn/715795.htm
uwa.wfcj3t6.cn/799595.htm
uwa.wfcj3t6.cn/555535.htm
uwa.wfcj3t6.cn/519955.htm
uwa.wfcj3t6.cn/795515.htm
uwa.wfcj3t6.cn/177195.htm
uwa.wfcj3t6.cn/515775.htm
uwa.wfcj3t6.cn/753795.htm
uwp.wfcj3t6.cn/557795.htm
uwp.wfcj3t6.cn/373795.htm
uwp.wfcj3t6.cn/559175.htm
uwp.wfcj3t6.cn/115955.htm
uwp.wfcj3t6.cn/133995.htm
uwp.wfcj3t6.cn/515975.htm
uwp.wfcj3t6.cn/531375.htm
uwp.wfcj3t6.cn/373535.htm
uwp.wfcj3t6.cn/511395.htm
uwp.wfcj3t6.cn/733535.htm
uwo.wfcj3t6.cn/711775.htm
uwo.wfcj3t6.cn/991995.htm
uwo.wfcj3t6.cn/553335.htm
uwo.wfcj3t6.cn/977995.htm
uwo.wfcj3t6.cn/715355.htm
uwo.wfcj3t6.cn/515375.htm
uwo.wfcj3t6.cn/799315.htm
uwo.wfcj3t6.cn/333935.htm
uwo.wfcj3t6.cn/359795.htm
uwo.wfcj3t6.cn/177115.htm
uwi.wfcj3t6.cn/393135.htm
uwi.wfcj3t6.cn/595915.htm
uwi.wfcj3t6.cn/379155.htm
uwi.wfcj3t6.cn/311595.htm
uwi.wfcj3t6.cn/759515.htm
uwi.wfcj3t6.cn/733795.htm
uwi.wfcj3t6.cn/313355.htm
uwi.wfcj3t6.cn/317155.htm
uwi.wfcj3t6.cn/573515.htm
uwi.wfcj3t6.cn/137755.htm
uwu.wfcj3t6.cn/579355.htm
uwu.wfcj3t6.cn/793535.htm
uwu.wfcj3t6.cn/373315.htm
uwu.wfcj3t6.cn/337935.htm
uwu.wfcj3t6.cn/715775.htm
uwu.wfcj3t6.cn/739195.htm
uwu.wfcj3t6.cn/375975.htm
uwu.wfcj3t6.cn/377335.htm
uwu.wfcj3t6.cn/599595.htm
uwu.wfcj3t6.cn/733195.htm
uwy.wfcj3t6.cn/537935.htm
uwy.wfcj3t6.cn/173595.htm
uwy.wfcj3t6.cn/957735.htm
uwy.wfcj3t6.cn/597555.htm
uwy.wfcj3t6.cn/95.htm
uwy.wfcj3t6.cn/177755.htm
uwy.wfcj3t6.cn/971175.htm
uwy.wfcj3t6.cn/717195.htm
uwy.wfcj3t6.cn/557315.htm
uwy.wfcj3t6.cn/773155.htm
uwt.wfcj3t6.cn/739935.htm
uwt.wfcj3t6.cn/111715.htm
uwt.wfcj3t6.cn/777135.htm
uwt.wfcj3t6.cn/753995.htm
uwt.wfcj3t6.cn/179935.htm
uwt.wfcj3t6.cn/339175.htm
uwt.wfcj3t6.cn/379715.htm
uwt.wfcj3t6.cn/153595.htm
uwt.wfcj3t6.cn/731755.htm
uwt.wfcj3t6.cn/779535.htm
uwr.wfcj3t6.cn/171395.htm
uwr.wfcj3t6.cn/533355.htm
uwr.wfcj3t6.cn/959135.htm
uwr.wfcj3t6.cn/335955.htm
uwr.wfcj3t6.cn/155135.htm
uwr.wfcj3t6.cn/131575.htm
uwr.wfcj3t6.cn/915155.htm
uwr.wfcj3t6.cn/155955.htm
uwr.wfcj3t6.cn/953535.htm
uwr.wfcj3t6.cn/337315.htm
uwe.wfcj3t6.cn/15.htm
uwe.wfcj3t6.cn/597115.htm
uwe.wfcj3t6.cn/791795.htm
uwe.wfcj3t6.cn/155135.htm
uwe.wfcj3t6.cn/535355.htm
uwe.wfcj3t6.cn/339115.htm
uwe.wfcj3t6.cn/515755.htm
uwe.wfcj3t6.cn/579375.htm
uwe.wfcj3t6.cn/971175.htm
uwe.wfcj3t6.cn/317995.htm
uww.wfcj3t6.cn/133395.htm
uww.wfcj3t6.cn/971335.htm
uww.wfcj3t6.cn/579555.htm
uww.wfcj3t6.cn/511995.htm
uww.wfcj3t6.cn/379575.htm
uww.wfcj3t6.cn/119995.htm
uww.wfcj3t6.cn/311575.htm
uww.wfcj3t6.cn/911355.htm
uww.wfcj3t6.cn/591115.htm
uww.wfcj3t6.cn/115775.htm
uwq.wfcj3t6.cn/171335.htm
uwq.wfcj3t6.cn/933355.htm
uwq.wfcj3t6.cn/555935.htm
uwq.wfcj3t6.cn/773735.htm
uwq.wfcj3t6.cn/599115.htm
uwq.wfcj3t6.cn/993535.htm
uwq.wfcj3t6.cn/197135.htm
uwq.wfcj3t6.cn/313315.htm
uwq.wfcj3t6.cn/195515.htm
uwq.wfcj3t6.cn/575975.htm
uqtv.wfcj3t6.cn/515955.htm
uqtv.wfcj3t6.cn/913915.htm
uqtv.wfcj3t6.cn/977555.htm
uqtv.wfcj3t6.cn/757795.htm
uqtv.wfcj3t6.cn/353335.htm
uqtv.wfcj3t6.cn/733315.htm
uqtv.wfcj3t6.cn/179775.htm
uqtv.wfcj3t6.cn/551715.htm
uqtv.wfcj3t6.cn/573355.htm
uqtv.wfcj3t6.cn/917355.htm
uqn.wfcj3t6.cn/555195.htm
uqn.wfcj3t6.cn/735735.htm
uqn.wfcj3t6.cn/597315.htm
uqn.wfcj3t6.cn/371755.htm
uqn.wfcj3t6.cn/333195.htm
uqn.wfcj3t6.cn/337795.htm
uqn.wfcj3t6.cn/957155.htm
uqn.wfcj3t6.cn/179395.htm
uqn.wfcj3t6.cn/551775.htm
uqn.wfcj3t6.cn/913135.htm
uqb.wfcj3t6.cn/959595.htm
uqb.wfcj3t6.cn/575715.htm
uqb.wfcj3t6.cn/959715.htm
uqb.wfcj3t6.cn/311195.htm
uqb.wfcj3t6.cn/113715.htm
uqb.wfcj3t6.cn/997395.htm
uqb.wfcj3t6.cn/579155.htm
uqb.wfcj3t6.cn/555335.htm
uqb.wfcj3t6.cn/397315.htm
uqb.wfcj3t6.cn/379935.htm
uqv.wfcj3t6.cn/951175.htm
uqv.wfcj3t6.cn/571775.htm
uqv.wfcj3t6.cn/957195.htm
uqv.wfcj3t6.cn/155395.htm
uqv.wfcj3t6.cn/519575.htm
uqv.wfcj3t6.cn/573935.htm
uqv.wfcj3t6.cn/373955.htm
uqv.wfcj3t6.cn/133135.htm
uqv.wfcj3t6.cn/199155.htm
uqv.wfcj3t6.cn/313575.htm
uqc.wfcj3t6.cn/715555.htm
uqc.wfcj3t6.cn/517935.htm
uqc.wfcj3t6.cn/517575.htm
uqc.wfcj3t6.cn/391915.htm
uqc.wfcj3t6.cn/777595.htm
uqc.wfcj3t6.cn/751155.htm
uqc.wfcj3t6.cn/331735.htm
uqc.wfcj3t6.cn/375915.htm
uqc.wfcj3t6.cn/777915.htm
uqc.wfcj3t6.cn/177575.htm
uqx.wfcj3t6.cn/319755.htm
uqx.wfcj3t6.cn/953115.htm
uqx.wfcj3t6.cn/315555.htm
uqx.wfcj3t6.cn/117115.htm
uqx.wfcj3t6.cn/139575.htm
uqx.wfcj3t6.cn/179135.htm
uqx.wfcj3t6.cn/313315.htm
uqx.wfcj3t6.cn/793975.htm
uqx.wfcj3t6.cn/153915.htm
uqx.wfcj3t6.cn/371915.htm
uqz.wfcj3t6.cn/373335.htm
uqz.wfcj3t6.cn/751115.htm
uqz.wfcj3t6.cn/731755.htm
uqz.wfcj3t6.cn/911555.htm
uqz.wfcj3t6.cn/139335.htm
uqz.wfcj3t6.cn/759795.htm
uqz.wfcj3t6.cn/935155.htm
uqz.wfcj3t6.cn/995955.htm
uqz.wfcj3t6.cn/951515.htm
uqz.wfcj3t6.cn/113395.htm
uql.wfcj3t6.cn/995775.htm
uql.wfcj3t6.cn/577515.htm
uql.wfcj3t6.cn/771115.htm
uql.wfcj3t6.cn/975155.htm
uql.wfcj3t6.cn/751775.htm
uql.wfcj3t6.cn/313975.htm
uql.wfcj3t6.cn/159795.htm
uql.wfcj3t6.cn/191955.htm
uql.wfcj3t6.cn/719735.htm
uql.wfcj3t6.cn/155195.htm
uqk.wfcj3t6.cn/737335.htm
uqk.wfcj3t6.cn/175115.htm
uqk.wfcj3t6.cn/331595.htm
uqk.wfcj3t6.cn/317195.htm
uqk.wfcj3t6.cn/999395.htm
uqk.wfcj3t6.cn/737555.htm
uqk.wfcj3t6.cn/793315.htm
uqk.wfcj3t6.cn/559175.htm
uqk.wfcj3t6.cn/973795.htm
uqk.wfcj3t6.cn/931195.htm
uqj.wfcj3t6.cn/193975.htm
uqj.wfcj3t6.cn/793195.htm
