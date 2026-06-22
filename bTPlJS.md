拓迫诮逞诤



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

伪谕诖愿媚热峦乒踩壮亚粱拥陆母

fei.vwbnt.cn/226266.Doc
fei.vwbnt.cn/864648.Doc
fei.vwbnt.cn/082280.Doc
fei.vwbnt.cn/008200.Doc
fei.vwbnt.cn/958263.Doc
fei.vwbnt.cn/864200.Doc
feu.vwbnt.cn/604406.Doc
feu.vwbnt.cn/828402.Doc
feu.vwbnt.cn/242200.Doc
feu.vwbnt.cn/264826.Doc
feu.vwbnt.cn/717955.Doc
feu.vwbnt.cn/959328.Doc
feu.vwbnt.cn/006408.Doc
feu.vwbnt.cn/860048.Doc
feu.vwbnt.cn/484088.Doc
feu.vwbnt.cn/262448.Doc
fey.vwbnt.cn/656238.Doc
fey.vwbnt.cn/684864.Doc
fey.vwbnt.cn/064066.Doc
fey.vwbnt.cn/042682.Doc
fey.vwbnt.cn/226002.Doc
fey.vwbnt.cn/488066.Doc
fey.vwbnt.cn/404262.Doc
fey.vwbnt.cn/426082.Doc
fey.vwbnt.cn/202084.Doc
fey.vwbnt.cn/248608.Doc
fet.vwbnt.cn/868082.Doc
fet.vwbnt.cn/648042.Doc
fet.vwbnt.cn/921340.Doc
fet.vwbnt.cn/640282.Doc
fet.vwbnt.cn/115553.Doc
fet.vwbnt.cn/646200.Doc
fet.vwbnt.cn/826220.Doc
fet.vwbnt.cn/608666.Doc
fet.vwbnt.cn/600862.Doc
fet.vwbnt.cn/662844.Doc
fer.vwbnt.cn/793593.Doc
fer.vwbnt.cn/020200.Doc
fer.vwbnt.cn/426860.Doc
fer.vwbnt.cn/228668.Doc
fer.vwbnt.cn/884244.Doc
fer.vwbnt.cn/620860.Doc
fer.vwbnt.cn/222606.Doc
fer.vwbnt.cn/626286.Doc
fer.vwbnt.cn/448008.Doc
fer.vwbnt.cn/826642.Doc
fee.vwbnt.cn/933551.Doc
fee.vwbnt.cn/008806.Doc
fee.vwbnt.cn/513373.Doc
fee.vwbnt.cn/848864.Doc
fee.vwbnt.cn/795337.Doc
fee.vwbnt.cn/252951.Doc
fee.vwbnt.cn/120746.Doc
fee.vwbnt.cn/484268.Doc
fee.vwbnt.cn/599911.Doc
fee.vwbnt.cn/046848.Doc
few.vwbnt.cn/022828.Doc
few.vwbnt.cn/680220.Doc
few.vwbnt.cn/220422.Doc
few.vwbnt.cn/840882.Doc
few.vwbnt.cn/206026.Doc
few.vwbnt.cn/624282.Doc
few.vwbnt.cn/228642.Doc
few.vwbnt.cn/028202.Doc
few.vwbnt.cn/424020.Doc
few.vwbnt.cn/860244.Doc
feq.vwbnt.cn/082444.Doc
feq.vwbnt.cn/000848.Doc
feq.vwbnt.cn/464880.Doc
feq.vwbnt.cn/484026.Doc
feq.vwbnt.cn/624684.Doc
feq.vwbnt.cn/862844.Doc
feq.vwbnt.cn/319937.Doc
feq.vwbnt.cn/862202.Doc
feq.vwbnt.cn/628550.Doc
feq.vwbnt.cn/244464.Doc
fwm.vwbnt.cn/440802.Doc
fwm.vwbnt.cn/860800.Doc
fwm.vwbnt.cn/353553.Doc
fwm.vwbnt.cn/624880.Doc
fwm.vwbnt.cn/442082.Doc
fwm.vwbnt.cn/622826.Doc
fwm.vwbnt.cn/468406.Doc
fwm.vwbnt.cn/868448.Doc
fwm.vwbnt.cn/046444.Doc
fwm.vwbnt.cn/488880.Doc
fwn.vwbnt.cn/731577.Doc
fwn.vwbnt.cn/024840.Doc
fwn.vwbnt.cn/284066.Doc
fwn.vwbnt.cn/840224.Doc
fwn.vwbnt.cn/868462.Doc
fwn.vwbnt.cn/246486.Doc
fwn.vwbnt.cn/026646.Doc
fwn.vwbnt.cn/462662.Doc
fwn.vwbnt.cn/939931.Doc
fwn.vwbnt.cn/424884.Doc
fwb.vwbnt.cn/919779.Doc
fwb.vwbnt.cn/640080.Doc
fwb.vwbnt.cn/319937.Doc
fwb.vwbnt.cn/208044.Doc
fwb.vwbnt.cn/440040.Doc
fwb.vwbnt.cn/204440.Doc
fwb.vwbnt.cn/684886.Doc
fwb.vwbnt.cn/757197.Doc
fwb.vwbnt.cn/484440.Doc
fwb.vwbnt.cn/460404.Doc
fwv.vwbnt.cn/406244.Doc
fwv.vwbnt.cn/666606.Doc
fwv.vwbnt.cn/600642.Doc
fwv.vwbnt.cn/379593.Doc
fwv.vwbnt.cn/800682.Doc
fwv.vwbnt.cn/959995.Doc
fwv.vwbnt.cn/777191.Doc
fwv.vwbnt.cn/799119.Doc
fwv.vwbnt.cn/402204.Doc
fwv.vwbnt.cn/024642.Doc
fwc.vwbnt.cn/464060.Doc
fwc.vwbnt.cn/422268.Doc
fwc.vwbnt.cn/426482.Doc
fwc.vwbnt.cn/008404.Doc
fwc.vwbnt.cn/084808.Doc
fwc.vwbnt.cn/608040.Doc
fwc.vwbnt.cn/088440.Doc
fwc.vwbnt.cn/420282.Doc
fwc.vwbnt.cn/884864.Doc
fwc.vwbnt.cn/084020.Doc
fwx.vwbnt.cn/628064.Doc
fwx.vwbnt.cn/422420.Doc
fwx.vwbnt.cn/466444.Doc
fwx.vwbnt.cn/426204.Doc
fwx.vwbnt.cn/002026.Doc
fwx.vwbnt.cn/006066.Doc
fwx.vwbnt.cn/060684.Doc
fwx.vwbnt.cn/648464.Doc
fwx.vwbnt.cn/626440.Doc
fwx.vwbnt.cn/848268.Doc
fwz.vwbnt.cn/828446.Doc
fwz.vwbnt.cn/602408.Doc
fwz.vwbnt.cn/422688.Doc
fwz.vwbnt.cn/800008.Doc
fwz.vwbnt.cn/200828.Doc
fwz.vwbnt.cn/661663.Doc
fwz.vwbnt.cn/911193.Doc
fwz.vwbnt.cn/680648.Doc
fwz.vwbnt.cn/573935.Doc
fwz.vwbnt.cn/175533.Doc
fwl.vwbnt.cn/288680.Doc
fwl.vwbnt.cn/863904.Doc
fwl.vwbnt.cn/993755.Doc
fwl.vwbnt.cn/847664.Doc
fwl.vwbnt.cn/938877.Doc
fwl.vwbnt.cn/808208.Doc
fwl.vwbnt.cn/082406.Doc
fwl.vwbnt.cn/956466.Doc
fwl.vwbnt.cn/393397.Doc
fwl.vwbnt.cn/600682.Doc
fwk.vwbnt.cn/464882.Doc
fwk.vwbnt.cn/326006.Doc
fwk.vwbnt.cn/282286.Doc
fwk.vwbnt.cn/184460.Doc
fwk.vwbnt.cn/133997.Doc
fwk.vwbnt.cn/715775.Doc
fwk.vwbnt.cn/246020.Doc
fwk.vwbnt.cn/466600.Doc
fwk.vwbnt.cn/143735.Doc
fwk.vwbnt.cn/862884.Doc
fwj.vwbnt.cn/646600.Doc
fwj.vwbnt.cn/822666.Doc
fwj.vwbnt.cn/038132.Doc
fwj.vwbnt.cn/044084.Doc
fwj.vwbnt.cn/194150.Doc
fwj.vwbnt.cn/615844.Doc
fwj.vwbnt.cn/735971.Doc
fwj.vwbnt.cn/000026.Doc
fwj.vwbnt.cn/842024.Doc
fwj.vwbnt.cn/048886.Doc
fwh.vwbnt.cn/062422.Doc
fwh.vwbnt.cn/484628.Doc
fwh.vwbnt.cn/244062.Doc
fwh.vwbnt.cn/410623.Doc
fwh.vwbnt.cn/424840.Doc
fwh.vwbnt.cn/834180.Doc
fwh.vwbnt.cn/404008.Doc
fwh.vwbnt.cn/029595.Doc
fwh.vwbnt.cn/391337.Doc
fwh.vwbnt.cn/868482.Doc
fwg.vwbnt.cn/604222.Doc
fwg.vwbnt.cn/208868.Doc
fwg.vwbnt.cn/004244.Doc
fwg.vwbnt.cn/688002.Doc
fwg.vwbnt.cn/608222.Doc
fwg.vwbnt.cn/162268.Doc
fwg.vwbnt.cn/886606.Doc
fwg.vwbnt.cn/446004.Doc
fwg.vwbnt.cn/371799.Doc
fwg.vwbnt.cn/448082.Doc
fwf.vwbnt.cn/026866.Doc
fwf.vwbnt.cn/220446.Doc
fwf.vwbnt.cn/042808.Doc
fwf.vwbnt.cn/462224.Doc
fwf.vwbnt.cn/042048.Doc
fwf.vwbnt.cn/882824.Doc
fwf.vwbnt.cn/244288.Doc
fwf.vwbnt.cn/608644.Doc
fwf.vwbnt.cn/206468.Doc
fwf.vwbnt.cn/242664.Doc
fwd.vwbnt.cn/248402.Doc
fwd.vwbnt.cn/139733.Doc
fwd.vwbnt.cn/062204.Doc
fwd.vwbnt.cn/408606.Doc
fwd.vwbnt.cn/468644.Doc
fwd.vwbnt.cn/068862.Doc
fwd.vwbnt.cn/822226.Doc
fwd.vwbnt.cn/426084.Doc
fwd.vwbnt.cn/024882.Doc
fwd.vwbnt.cn/422608.Doc
fws.vwbnt.cn/735317.Doc
fws.vwbnt.cn/848042.Doc
fws.vwbnt.cn/828820.Doc
fws.vwbnt.cn/068408.Doc
fws.vwbnt.cn/200028.Doc
fws.vwbnt.cn/400240.Doc
fws.vwbnt.cn/486248.Doc
fws.vwbnt.cn/206428.Doc
fws.vwbnt.cn/666402.Doc
fws.vwbnt.cn/775973.Doc
fwa.vwbnt.cn/084204.Doc
fwa.vwbnt.cn/842880.Doc
fwa.vwbnt.cn/822026.Doc
fwa.vwbnt.cn/202864.Doc
fwa.vwbnt.cn/600282.Doc
fwa.vwbnt.cn/068264.Doc
fwa.vwbnt.cn/468622.Doc
fwa.vwbnt.cn/468664.Doc
fwa.vwbnt.cn/464882.Doc
fwa.vwbnt.cn/886060.Doc
fwp.vwbnt.cn/064224.Doc
fwp.vwbnt.cn/600402.Doc
fwp.vwbnt.cn/466044.Doc
fwp.vwbnt.cn/848882.Doc
fwp.vwbnt.cn/733357.Doc
fwp.vwbnt.cn/828864.Doc
fwp.vwbnt.cn/688648.Doc
fwp.vwbnt.cn/062284.Doc
fwp.vwbnt.cn/840828.Doc
fwp.vwbnt.cn/822022.Doc
fwo.vwbnt.cn/442044.Doc
fwo.vwbnt.cn/826240.Doc
fwo.vwbnt.cn/686204.Doc
fwo.vwbnt.cn/466400.Doc
fwo.vwbnt.cn/626026.Doc
fwo.vwbnt.cn/442646.Doc
fwo.vwbnt.cn/228422.Doc
fwo.vwbnt.cn/228680.Doc
fwo.vwbnt.cn/644442.Doc
fwo.vwbnt.cn/648844.Doc
fwi.vwbnt.cn/662646.Doc
fwi.vwbnt.cn/426044.Doc
fwi.vwbnt.cn/008286.Doc
fwi.vwbnt.cn/886602.Doc
fwi.vwbnt.cn/681486.Doc
fwi.vwbnt.cn/992058.Doc
fwi.vwbnt.cn/038864.Doc
fwi.vwbnt.cn/221616.Doc
fwi.vwbnt.cn/682432.Doc
fwi.vwbnt.cn/094588.Doc
fwu.vwbnt.cn/030085.Doc
fwu.vwbnt.cn/884069.Doc
fwu.vwbnt.cn/345327.Doc
fwu.vwbnt.cn/091263.Doc
fwu.vwbnt.cn/446188.Doc
fwu.vwbnt.cn/544393.Doc
fwu.vwbnt.cn/642638.Doc
fwu.vwbnt.cn/870565.Doc
fwu.vwbnt.cn/843552.Doc
fwu.vwbnt.cn/609328.Doc
fwy.vwbnt.cn/006635.Doc
fwy.vwbnt.cn/815512.Doc
fwy.vwbnt.cn/826719.Doc
fwy.vwbnt.cn/192022.Doc
fwy.vwbnt.cn/733828.Doc
fwy.vwbnt.cn/522676.Doc
fwy.vwbnt.cn/111928.Doc
fwy.vwbnt.cn/760680.Doc
fwy.vwbnt.cn/825391.Doc
fwy.vwbnt.cn/859795.Doc
fwt.vwbnt.cn/738884.Doc
fwt.vwbnt.cn/762126.Doc
fwt.vwbnt.cn/083437.Doc
fwt.vwbnt.cn/449574.Doc
fwt.vwbnt.cn/178851.Doc
fwt.vwbnt.cn/062196.Doc
fwt.vwbnt.cn/418983.Doc
fwt.vwbnt.cn/713554.Doc
fwt.vwbnt.cn/442375.Doc
fwt.vwbnt.cn/117058.Doc
fwr.vwbnt.cn/113280.Doc
fwr.vwbnt.cn/748194.Doc
fwr.vwbnt.cn/129132.Doc
fwr.vwbnt.cn/988754.Doc
fwr.vwbnt.cn/624396.Doc
fwr.vwbnt.cn/852880.Doc
fwr.vwbnt.cn/663255.Doc
fwr.vwbnt.cn/258498.Doc
fwr.vwbnt.cn/082838.Doc
fwr.vwbnt.cn/020284.Doc
fwe.vwbnt.cn/040204.Doc
fwe.vwbnt.cn/406848.Doc
fwe.vwbnt.cn/684882.Doc
fwe.vwbnt.cn/642808.Doc
fwe.vwbnt.cn/268086.Doc
fwe.vwbnt.cn/662246.Doc
fwe.vwbnt.cn/422820.Doc
fwe.vwbnt.cn/222864.Doc
fwe.vwbnt.cn/044800.Doc
fwe.vwbnt.cn/084226.Doc
fww.vwbnt.cn/660624.Doc
fww.vwbnt.cn/460862.Doc
fww.vwbnt.cn/608648.Doc
fww.vwbnt.cn/406426.Doc
fww.vwbnt.cn/842666.Doc
fww.vwbnt.cn/624026.Doc
fww.vwbnt.cn/060200.Doc
fww.vwbnt.cn/274665.Doc
fww.vwbnt.cn/220242.Doc
fww.vwbnt.cn/280024.Doc
fwq.vwbnt.cn/102665.Doc
fwq.vwbnt.cn/873551.Doc
fwq.vwbnt.cn/224800.Doc
fwq.vwbnt.cn/686606.Doc
fwq.vwbnt.cn/601284.Doc
fwq.vwbnt.cn/828066.Doc
fwq.vwbnt.cn/599577.Doc
fwq.vwbnt.cn/125524.Doc
fwq.vwbnt.cn/402022.Doc
fwq.vwbnt.cn/660026.Doc
fqm.vwbnt.cn/771339.Doc
fqm.vwbnt.cn/202628.Doc
fqm.vwbnt.cn/608682.Doc
fqm.vwbnt.cn/506199.Doc
fqm.vwbnt.cn/600282.Doc
fqm.vwbnt.cn/806228.Doc
fqm.vwbnt.cn/666220.Doc
fqm.vwbnt.cn/660064.Doc
fqm.vwbnt.cn/880088.Doc
fqm.vwbnt.cn/444640.Doc
fqn.vwbnt.cn/262662.Doc
fqn.vwbnt.cn/866802.Doc
fqn.vwbnt.cn/046842.Doc
fqn.vwbnt.cn/824802.Doc
fqn.vwbnt.cn/026024.Doc
fqn.vwbnt.cn/979973.Doc
fqn.vwbnt.cn/426082.Doc
fqn.vwbnt.cn/840082.Doc
fqn.vwbnt.cn/262688.Doc
fqn.vwbnt.cn/371377.Doc
fqb.vwbnt.cn/266204.Doc
fqb.vwbnt.cn/268444.Doc
fqb.vwbnt.cn/884228.Doc
fqb.vwbnt.cn/755713.Doc
fqb.vwbnt.cn/242064.Doc
fqb.vwbnt.cn/846680.Doc
fqb.vwbnt.cn/440682.Doc
fqb.vwbnt.cn/006406.Doc
fqb.vwbnt.cn/280628.Doc
fqb.vwbnt.cn/646488.Doc
fqv.vwbnt.cn/624428.Doc
fqv.vwbnt.cn/377799.Doc
fqv.vwbnt.cn/064222.Doc
fqv.vwbnt.cn/264600.Doc
fqv.vwbnt.cn/320500.Doc
fqv.vwbnt.cn/485971.Doc
fqv.vwbnt.cn/133979.Doc
fqv.vwbnt.cn/020868.Doc
fqv.vwbnt.cn/164084.Doc
fqv.vwbnt.cn/860666.Doc
fqc.vwbnt.cn/802644.Doc
fqc.vwbnt.cn/616591.Doc
fqc.vwbnt.cn/406688.Doc
fqc.vwbnt.cn/426486.Doc
fqc.vwbnt.cn/040248.Doc
fqc.vwbnt.cn/677491.Doc
fqc.vwbnt.cn/244424.Doc
fqc.vwbnt.cn/488400.Doc
fqc.vwbnt.cn/008842.Doc
fqc.vwbnt.cn/248626.Doc
fqx.vwbnt.cn/648684.Doc
fqx.vwbnt.cn/177717.Doc
fqx.vwbnt.cn/064006.Doc
fqx.vwbnt.cn/660282.Doc
fqx.vwbnt.cn/804806.Doc
fqx.vwbnt.cn/193977.Doc
fqx.vwbnt.cn/064866.Doc
fqx.vwbnt.cn/286666.Doc
fqx.vwbnt.cn/391775.Doc
