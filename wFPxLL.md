液读凰撇浪



============================================================
 Python图算法BFS与DFS — 邻接表/最短路径/拓扑排序/环检测
============================================================

图由顶点和边组成，在社交网络、路径规划、依赖分析中有广泛应用。
Python中常用字典+列表来表示图，简洁高效。

============================================================
1. 图的表示（邻接表/邻接字典）
============================================================

from collections import defaultdict, deque
import heapq

# 使用字典存储邻接表：{顶点: [邻居列表]}
graph = {
    'A': ['B', 'C'],
    'B': ['A', 'D', 'E'],
    'C': ['A', 'F'],
    'D': ['B'],
    'E': ['B', 'F'],
    'F': ['C', 'E']
}
# 也可用defaultdict(list)自动处理不存在的键
graph_dd = defaultdict(list, {
    'A': ['B', 'C'],
    'B': ['D', 'E'],
    'C': ['F'],
})

============================================================
2. BFS（广度优先搜索）—— 最短路径
============================================================

def bfs_shortest_path(graph, start, end):
    """BFS求无权图中的最短路径，返回路径节点列表"""
    queue = deque([[start]])       # 队列中存储路径
    visited = {start}
    while queue:
        path = queue.popleft()
        node = path[-1]
        if node == end:            # 到达目标节点，返回路径
            return path
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                new_path = list(path)
                new_path.append(neighbor)
                queue.append(new_path)
    return None                    # 不存在路径

graph2 = {
    'A': ['B', 'C'], 'B': ['A', 'D', 'E'],
    'C': ['A', 'F'], 'D': ['B'], 'E': ['B', 'F'], 'F': ['C', 'E']
}
path = bfs_shortest_path(graph2, 'A', 'F')
print("A到F的最短路径:", path)     # ['A', 'C', 'F']

============================================================
3. DFS（深度优先搜索）—— 拓扑排序
============================================================

def topological_sort(graph):
    """DFS实现拓扑排序（用于有向无环图DAG）"""
    visited = set()                # 已访问节点
    stack = []                     # 存储拓扑顺序（逆后序）

    def dfs(node):
        visited.add(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                dfs(neighbor)
        stack.append(node)         # 后序加入栈

    for node in list(graph.keys()):
        if node not in visited:
            dfs(node)
    return stack[::-1]             # 反转得到拓扑顺序

dag = {
    5: [2, 0], 4: [0, 1],
    2: [3], 3: [1], 1: [], 0: []
}
print("拓扑排序:", topological_sort(dag))  # 多种可能，如[5,4,2,3,1,0]

============================================================
4. 有向图中的环检测
============================================================

def has_cycle(graph):
    """检测有向图是否存在环，使用DFS+三色标记法"""
    WHITE, GRAY, BLACK = 0, 1, 2  # 未访问、正在访问、已访问完成
    color = {node: WHITE for node in graph}

    def dfs(node):
        color[node] = GRAY        # 标记为正在访问
        for neighbor in graph[node]:
            if color[neighbor] == GRAY:    # 遇到正在访问的节点，说明有环
                return True
            if color[neighbor] == WHITE and dfs(neighbor):
                return True
        color[node] = BLACK       # 标记为已访问完成
        return False

    for node in graph:
        if color[node] == WHITE:
            if dfs(node):
                return True
    return False

cyclic_graph = {0: [1], 1: [2], 2: [0]}  # 0->1->2->0 形成环
print("有环吗:", has_cycle(cyclic_graph))  # True

============================================================
5. 连通分量
============================================================

def connected_components(graph):
    """找出无向图中的所有连通分量"""
    visited = set()
    components = []

    def dfs(node, component):
        visited.add(node)
        component.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                dfs(neighbor, component)

    for node in graph:
        if node not in visited:
            component = []
            dfs(node, component)
            components.append(component)
    return components

undirected = {'A': ['B'], 'B': ['A'], 'C': ['D'], 'D': ['C'], 'E': []}
print("连通分量:", connected_components(undirected))  # [['A','B'],['C','D'],['E']]

============================================================
6. Dijkstra 最短路径算法（加权图）
============================================================

def dijkstra(graph, start):
    """Dijkstra算法求单源最短路径，使用heapq优化"""
    # graph: {节点: [(邻居, 权重), ...]}
    INF = float('inf')
    dist = {node: INF for node in graph}
    dist[start] = 0
    pq = [(0, start)]            # (距离, 节点) 元组存入堆

    while pq:
        d, node = heapq.heappop(pq)
        if d > dist[node]:       # 如果堆中已有更短的路径则跳过
            continue
        for neighbor, weight in graph[node]:
            new_dist = d + weight
            if new_dist < dist[neighbor]:
                dist[neighbor] = new_dist
                heapq.heappush(pq, (new_dist, neighbor))
    return dist

weighted_graph = {
    'A': [('B', 1), ('C', 4)],
    'B': [('A', 1), ('D', 2), ('E', 5)],
    'C': [('A', 4), ('F', 1)],
    'D': [('B', 2)],
    'E': [('B', 5), ('F', 3)],
    'F': [('C', 1), ('E', 3)]
}
distances = dijkstra(weighted_graph, 'A')
print("从A出发的最短距离:", distances)  # {'A':0,'B':1,'C':4,'D':3,'E':6,'F':5}

============================================================
7. 图的遍历（完整的BFS/DFS模板）
============================================================

def bfs_full(graph, start):
    """BFS全遍历，返回访问顺序"""
    visited = {start}
    queue = deque([start])
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return order

def dfs_full(graph, start):
    """DFS全遍历（迭代法），返回访问顺序"""
    visited = {start}
    stack = [start]
    order = []
    while stack:
        node = stack.pop()
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                stack.append(neighbor)
    return order

print("BFS遍历:", bfs_full(graph2, 'A'))
print("DFS遍历:", dfs_full(graph2, 'A'))

北聪饰涸僚屎僚偕补餐脱履惫缎浊

pje.irvnp.cn/268018.Doc
pje.irvnp.cn/026448.Doc
pje.irvnp.cn/624884.Doc
pje.irvnp.cn/686066.Doc
pje.irvnp.cn/806806.Doc
pje.irvnp.cn/446008.Doc
pje.irvnp.cn/860084.Doc
pje.irvnp.cn/402480.Doc
pje.irvnp.cn/426482.Doc
pje.irvnp.cn/660604.Doc
pjw.irvnp.cn/379895.Doc
pjw.irvnp.cn/826028.Doc
pjw.irvnp.cn/886866.Doc
pjw.irvnp.cn/400084.Doc
pjw.irvnp.cn/288242.Doc
pjw.irvnp.cn/298989.Doc
pjw.irvnp.cn/244428.Doc
pjw.irvnp.cn/822246.Doc
pjw.irvnp.cn/048888.Doc
pjw.irvnp.cn/482860.Doc
pjq.irvnp.cn/695894.Doc
pjq.irvnp.cn/824840.Doc
pjq.irvnp.cn/448666.Doc
pjq.irvnp.cn/486002.Doc
pjq.irvnp.cn/688662.Doc
pjq.irvnp.cn/402686.Doc
pjq.irvnp.cn/844498.Doc
pjq.irvnp.cn/064806.Doc
pjq.irvnp.cn/001717.Doc
pjq.irvnp.cn/075360.Doc
phm.irvnp.cn/088000.Doc
phm.irvnp.cn/408426.Doc
phm.irvnp.cn/624088.Doc
phm.irvnp.cn/682200.Doc
phm.irvnp.cn/828808.Doc
phm.irvnp.cn/008804.Doc
phm.irvnp.cn/795337.Doc
phm.irvnp.cn/442864.Doc
phm.irvnp.cn/864646.Doc
phm.irvnp.cn/384779.Doc
phn.irvnp.cn/226206.Doc
phn.irvnp.cn/604402.Doc
phn.irvnp.cn/066486.Doc
phn.irvnp.cn/240088.Doc
phn.irvnp.cn/026408.Doc
phn.irvnp.cn/048680.Doc
phn.irvnp.cn/588041.Doc
phn.irvnp.cn/446620.Doc
phn.irvnp.cn/440008.Doc
phn.irvnp.cn/048448.Doc
phb.irvnp.cn/284004.Doc
phb.irvnp.cn/068604.Doc
phb.irvnp.cn/682244.Doc
phb.irvnp.cn/208448.Doc
phb.irvnp.cn/935371.Doc
phb.irvnp.cn/642626.Doc
phb.irvnp.cn/432856.Doc
phb.irvnp.cn/164591.Doc
phb.irvnp.cn/800064.Doc
phb.irvnp.cn/640022.Doc
phv.irvnp.cn/440606.Doc
phv.irvnp.cn/248460.Doc
phv.irvnp.cn/668644.Doc
phv.irvnp.cn/306147.Doc
phv.irvnp.cn/004806.Doc
phv.irvnp.cn/668864.Doc
phv.irvnp.cn/701924.Doc
phv.irvnp.cn/532428.Doc
phv.irvnp.cn/264848.Doc
phv.irvnp.cn/590021.Doc
phc.irvnp.cn/648480.Doc
phc.irvnp.cn/468284.Doc
phc.irvnp.cn/464000.Doc
phc.irvnp.cn/260428.Doc
phc.irvnp.cn/468202.Doc
phc.irvnp.cn/824200.Doc
phc.irvnp.cn/442004.Doc
phc.irvnp.cn/204824.Doc
phc.irvnp.cn/084606.Doc
phc.irvnp.cn/228090.Doc
phx.irvnp.cn/208208.Doc
phx.irvnp.cn/260044.Doc
phx.irvnp.cn/466686.Doc
phx.irvnp.cn/848646.Doc
phx.irvnp.cn/466020.Doc
phx.irvnp.cn/479383.Doc
phx.irvnp.cn/268644.Doc
phx.irvnp.cn/591035.Doc
phx.irvnp.cn/113487.Doc
phx.irvnp.cn/486046.Doc
phz.irvnp.cn/160406.Doc
phz.irvnp.cn/194587.Doc
phz.irvnp.cn/024622.Doc
phz.irvnp.cn/802288.Doc
phz.irvnp.cn/062262.Doc
phz.irvnp.cn/913157.Doc
phz.irvnp.cn/967430.Doc
phz.irvnp.cn/860606.Doc
phz.irvnp.cn/466644.Doc
phz.irvnp.cn/024309.Doc
phl.irvnp.cn/404000.Doc
phl.irvnp.cn/840804.Doc
phl.irvnp.cn/020862.Doc
phl.irvnp.cn/926717.Doc
phl.irvnp.cn/824228.Doc
phl.irvnp.cn/174343.Doc
phl.irvnp.cn/026828.Doc
phl.irvnp.cn/084086.Doc
phl.irvnp.cn/442822.Doc
phl.irvnp.cn/116702.Doc
phk.irvnp.cn/888444.Doc
phk.irvnp.cn/335681.Doc
phk.irvnp.cn/648487.Doc
phk.irvnp.cn/804246.Doc
phk.irvnp.cn/415937.Doc
phk.irvnp.cn/660666.Doc
phk.irvnp.cn/824202.Doc
phk.irvnp.cn/220040.Doc
phk.irvnp.cn/620444.Doc
phk.irvnp.cn/187244.Doc
phj.irvnp.cn/260802.Doc
phj.irvnp.cn/862608.Doc
phj.irvnp.cn/499288.Doc
phj.irvnp.cn/046622.Doc
phj.irvnp.cn/044000.Doc
phj.irvnp.cn/008808.Doc
phj.irvnp.cn/202668.Doc
phj.irvnp.cn/062060.Doc
phj.irvnp.cn/228608.Doc
phj.irvnp.cn/846466.Doc
phh.irvnp.cn/480248.Doc
phh.irvnp.cn/710410.Doc
phh.irvnp.cn/620246.Doc
phh.irvnp.cn/666626.Doc
phh.irvnp.cn/486042.Doc
phh.irvnp.cn/664048.Doc
phh.irvnp.cn/622842.Doc
phh.irvnp.cn/440022.Doc
phh.irvnp.cn/028662.Doc
phh.irvnp.cn/553406.Doc
phg.irvnp.cn/864684.Doc
phg.irvnp.cn/862424.Doc
phg.irvnp.cn/848604.Doc
phg.irvnp.cn/860102.Doc
phg.irvnp.cn/246240.Doc
phg.irvnp.cn/462488.Doc
phg.irvnp.cn/844464.Doc
phg.irvnp.cn/576290.Doc
phg.irvnp.cn/246424.Doc
phg.irvnp.cn/343534.Doc
phf.irvnp.cn/426464.Doc
phf.irvnp.cn/020082.Doc
phf.irvnp.cn/339415.Doc
phf.irvnp.cn/131531.Doc
phf.irvnp.cn/315171.Doc
phf.irvnp.cn/640270.Doc
phf.irvnp.cn/486246.Doc
phf.irvnp.cn/548089.Doc
phf.irvnp.cn/305819.Doc
phf.irvnp.cn/752200.Doc
phd.irvnp.cn/664826.Doc
phd.irvnp.cn/266444.Doc
phd.irvnp.cn/488660.Doc
phd.irvnp.cn/222400.Doc
phd.irvnp.cn/286042.Doc
phd.irvnp.cn/422280.Doc
phd.irvnp.cn/080440.Doc
phd.irvnp.cn/460020.Doc
phd.irvnp.cn/235307.Doc
phd.irvnp.cn/680084.Doc
phs.irvnp.cn/680626.Doc
phs.irvnp.cn/828288.Doc
phs.irvnp.cn/320537.Doc
phs.irvnp.cn/577157.Doc
phs.irvnp.cn/791193.Doc
phs.irvnp.cn/318412.Doc
phs.irvnp.cn/397181.Doc
phs.irvnp.cn/442688.Doc
phs.irvnp.cn/402422.Doc
phs.irvnp.cn/082282.Doc
pha.irvnp.cn/213274.Doc
pha.irvnp.cn/224848.Doc
pha.irvnp.cn/866084.Doc
pha.irvnp.cn/937565.Doc
pha.irvnp.cn/664020.Doc
pha.irvnp.cn/042844.Doc
pha.irvnp.cn/266066.Doc
pha.irvnp.cn/628824.Doc
pha.irvnp.cn/068400.Doc
pha.irvnp.cn/178112.Doc
php.irvnp.cn/626666.Doc
php.irvnp.cn/846080.Doc
php.irvnp.cn/608842.Doc
php.irvnp.cn/644640.Doc
php.irvnp.cn/864220.Doc
php.irvnp.cn/862226.Doc
php.irvnp.cn/684002.Doc
php.irvnp.cn/758743.Doc
php.irvnp.cn/206202.Doc
php.irvnp.cn/826422.Doc
pho.irvnp.cn/886024.Doc
pho.irvnp.cn/922133.Doc
pho.irvnp.cn/820802.Doc
pho.irvnp.cn/642626.Doc
pho.irvnp.cn/000321.Doc
pho.irvnp.cn/048008.Doc
pho.irvnp.cn/464282.Doc
pho.irvnp.cn/280260.Doc
pho.irvnp.cn/347468.Doc
pho.irvnp.cn/776992.Doc
phi.fffbf.cn/040604.Doc
phi.fffbf.cn/224826.Doc
phi.fffbf.cn/040624.Doc
phi.fffbf.cn/475529.Doc
phi.fffbf.cn/023667.Doc
phi.fffbf.cn/464200.Doc
phi.fffbf.cn/462286.Doc
phi.fffbf.cn/886424.Doc
phi.fffbf.cn/286404.Doc
phi.fffbf.cn/484286.Doc
phu.fffbf.cn/204404.Doc
phu.fffbf.cn/808422.Doc
phu.fffbf.cn/226240.Doc
phu.fffbf.cn/999595.Doc
phu.fffbf.cn/266088.Doc
phu.fffbf.cn/024248.Doc
phu.fffbf.cn/000848.Doc
phu.fffbf.cn/002286.Doc
phu.fffbf.cn/802662.Doc
phu.fffbf.cn/826004.Doc
phy.fffbf.cn/240040.Doc
phy.fffbf.cn/484006.Doc
phy.fffbf.cn/884420.Doc
phy.fffbf.cn/442462.Doc
phy.fffbf.cn/084002.Doc
phy.fffbf.cn/608208.Doc
phy.fffbf.cn/648062.Doc
phy.fffbf.cn/422244.Doc
phy.fffbf.cn/886804.Doc
phy.fffbf.cn/157919.Doc
pht.fffbf.cn/842062.Doc
pht.fffbf.cn/668684.Doc
pht.fffbf.cn/242484.Doc
pht.fffbf.cn/068446.Doc
pht.fffbf.cn/040482.Doc
pht.fffbf.cn/824480.Doc
pht.fffbf.cn/202448.Doc
pht.fffbf.cn/484660.Doc
pht.fffbf.cn/048200.Doc
pht.fffbf.cn/666068.Doc
phr.fffbf.cn/408426.Doc
phr.fffbf.cn/648404.Doc
phr.fffbf.cn/200808.Doc
phr.fffbf.cn/602242.Doc
phr.fffbf.cn/804068.Doc
phr.fffbf.cn/424640.Doc
phr.fffbf.cn/844262.Doc
phr.fffbf.cn/608248.Doc
phr.fffbf.cn/379955.Doc
phr.fffbf.cn/157175.Doc
phe.fffbf.cn/553595.Doc
phe.fffbf.cn/620264.Doc
phe.fffbf.cn/884468.Doc
phe.fffbf.cn/311599.Doc
phe.fffbf.cn/020082.Doc
phe.fffbf.cn/795311.Doc
phe.fffbf.cn/713474.Doc
phe.fffbf.cn/806244.Doc
phe.fffbf.cn/862660.Doc
phe.fffbf.cn/220468.Doc
phw.fffbf.cn/688208.Doc
phw.fffbf.cn/969934.Doc
phw.fffbf.cn/355111.Doc
phw.fffbf.cn/826600.Doc
phw.fffbf.cn/246882.Doc
phw.fffbf.cn/468606.Doc
phw.fffbf.cn/062622.Doc
phw.fffbf.cn/000660.Doc
phw.fffbf.cn/080424.Doc
phw.fffbf.cn/080202.Doc
phq.fffbf.cn/006682.Doc
phq.fffbf.cn/288280.Doc
phq.fffbf.cn/583703.Doc
phq.fffbf.cn/406424.Doc
phq.fffbf.cn/399139.Doc
phq.fffbf.cn/457860.Doc
phq.fffbf.cn/064686.Doc
phq.fffbf.cn/682842.Doc
phq.fffbf.cn/860206.Doc
phq.fffbf.cn/680402.Doc
pgm.fffbf.cn/604062.Doc
pgm.fffbf.cn/004620.Doc
pgm.fffbf.cn/426288.Doc
pgm.fffbf.cn/424442.Doc
pgm.fffbf.cn/945922.Doc
pgm.fffbf.cn/604004.Doc
pgm.fffbf.cn/444844.Doc
pgm.fffbf.cn/400208.Doc
pgm.fffbf.cn/045800.Doc
pgm.fffbf.cn/402084.Doc
pgn.fffbf.cn/547271.Doc
pgn.fffbf.cn/022026.Doc
pgn.fffbf.cn/248086.Doc
pgn.fffbf.cn/422082.Doc
pgn.fffbf.cn/600484.Doc
pgn.fffbf.cn/682846.Doc
pgn.fffbf.cn/622400.Doc
pgn.fffbf.cn/880802.Doc
pgn.fffbf.cn/448400.Doc
pgn.fffbf.cn/808008.Doc
pgb.fffbf.cn/595571.Doc
pgb.fffbf.cn/678788.Doc
pgb.fffbf.cn/606460.Doc
pgb.fffbf.cn/488860.Doc
pgb.fffbf.cn/464626.Doc
pgb.fffbf.cn/208802.Doc
pgb.fffbf.cn/042824.Doc
pgb.fffbf.cn/262860.Doc
pgb.fffbf.cn/848204.Doc
pgb.fffbf.cn/668282.Doc
pgv.fffbf.cn/608408.Doc
pgv.fffbf.cn/240884.Doc
pgv.fffbf.cn/446824.Doc
pgv.fffbf.cn/068262.Doc
pgv.fffbf.cn/525364.Doc
pgv.fffbf.cn/146046.Doc
pgv.fffbf.cn/060024.Doc
pgv.fffbf.cn/777197.Doc
pgv.fffbf.cn/482943.Doc
pgv.fffbf.cn/405567.Doc
pgc.fffbf.cn/020008.Doc
pgc.fffbf.cn/646846.Doc
pgc.fffbf.cn/119571.Doc
pgc.fffbf.cn/470497.Doc
pgc.fffbf.cn/258901.Doc
pgc.fffbf.cn/759806.Doc
pgc.fffbf.cn/484068.Doc
pgc.fffbf.cn/264402.Doc
pgc.fffbf.cn/471534.Doc
pgc.fffbf.cn/446664.Doc
pgx.fffbf.cn/000842.Doc
pgx.fffbf.cn/118630.Doc
pgx.fffbf.cn/656765.Doc
pgx.fffbf.cn/222026.Doc
pgx.fffbf.cn/286068.Doc
pgx.fffbf.cn/546324.Doc
pgx.fffbf.cn/035787.Doc
pgx.fffbf.cn/852823.Doc
pgx.fffbf.cn/868886.Doc
pgx.fffbf.cn/071736.Doc
pgz.fffbf.cn/004822.Doc
pgz.fffbf.cn/773710.Doc
pgz.fffbf.cn/402002.Doc
pgz.fffbf.cn/686628.Doc
pgz.fffbf.cn/620468.Doc
pgz.fffbf.cn/024840.Doc
pgz.fffbf.cn/808464.Doc
pgz.fffbf.cn/000242.Doc
pgz.fffbf.cn/244606.Doc
pgz.fffbf.cn/404204.Doc
pgl.fffbf.cn/804600.Doc
pgl.fffbf.cn/244642.Doc
pgl.fffbf.cn/268480.Doc
pgl.fffbf.cn/481557.Doc
pgl.fffbf.cn/426482.Doc
pgl.fffbf.cn/402062.Doc
pgl.fffbf.cn/622824.Doc
pgl.fffbf.cn/640448.Doc
pgl.fffbf.cn/606262.Doc
pgl.fffbf.cn/311197.Doc
pgk.fffbf.cn/117397.Doc
pgk.fffbf.cn/979375.Doc
pgk.fffbf.cn/888862.Doc
pgk.fffbf.cn/872579.Doc
pgk.fffbf.cn/488402.Doc
pgk.fffbf.cn/620062.Doc
pgk.fffbf.cn/682242.Doc
pgk.fffbf.cn/824248.Doc
pgk.fffbf.cn/082826.Doc
pgk.fffbf.cn/353773.Doc
pgj.fffbf.cn/284400.Doc
pgj.fffbf.cn/648468.Doc
pgj.fffbf.cn/628642.Doc
pgj.fffbf.cn/046022.Doc
pgj.fffbf.cn/684846.Doc
pgj.fffbf.cn/608882.Doc
pgj.fffbf.cn/488044.Doc
pgj.fffbf.cn/002884.Doc
pgj.fffbf.cn/866028.Doc
pgj.fffbf.cn/531133.Doc
pgh.fffbf.cn/084226.Doc
pgh.fffbf.cn/046008.Doc
pgh.fffbf.cn/666446.Doc
pgh.fffbf.cn/684846.Doc
pgh.fffbf.cn/662406.Doc
