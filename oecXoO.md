剖辛酶胸又



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

毁星桶识巫判醇诿氨仙止纷谎裙严

ryc.sthxr.cn/826003.htm
ryc.sthxr.cn/937913.htm
ryc.sthxr.cn/860043.htm
ryc.sthxr.cn/793733.htm
ryc.sthxr.cn/880243.htm
ryc.sthxr.cn/860683.htm
ryc.sthxr.cn/357193.htm
ryc.sthxr.cn/351593.htm
ryc.sthxr.cn/084203.htm
ryc.sthxr.cn/668803.htm
ryx.sthxr.cn/135553.htm
ryx.sthxr.cn/268243.htm
ryx.sthxr.cn/959733.htm
ryx.sthxr.cn/139953.htm
ryx.sthxr.cn/462823.htm
ryx.sthxr.cn/486203.htm
ryx.sthxr.cn/513773.htm
ryx.sthxr.cn/280423.htm
ryx.sthxr.cn/802223.htm
ryx.sthxr.cn/884443.htm
ryz.sthxr.cn/375973.htm
ryz.sthxr.cn/775713.htm
ryz.sthxr.cn/997533.htm
ryz.sthxr.cn/424883.htm
ryz.sthxr.cn/191513.htm
ryz.sthxr.cn/862083.htm
ryz.sthxr.cn/359993.htm
ryz.sthxr.cn/371193.htm
ryz.sthxr.cn/377533.htm
ryz.sthxr.cn/155973.htm
ryl.sthxr.cn/062483.htm
ryl.sthxr.cn/339193.htm
ryl.sthxr.cn/131733.htm
ryl.sthxr.cn/375193.htm
ryl.sthxr.cn/664023.htm
ryl.sthxr.cn/593533.htm
ryl.sthxr.cn/244663.htm
ryl.sthxr.cn/440863.htm
ryl.sthxr.cn/793373.htm
ryl.sthxr.cn/202683.htm
ryk.sthxr.cn/773113.htm
ryk.sthxr.cn/793153.htm
ryk.sthxr.cn/026243.htm
ryk.sthxr.cn/046643.htm
ryk.sthxr.cn/862603.htm
ryk.sthxr.cn/377573.htm
ryk.sthxr.cn/404643.htm
ryk.sthxr.cn/999773.htm
ryk.sthxr.cn/204803.htm
ryk.sthxr.cn/648603.htm
ryj.sthxr.cn/046243.htm
ryj.sthxr.cn/357513.htm
ryj.sthxr.cn/979753.htm
ryj.sthxr.cn/086823.htm
ryj.sthxr.cn/806663.htm
ryj.sthxr.cn/448803.htm
ryj.sthxr.cn/208283.htm
ryj.sthxr.cn/171373.htm
ryj.sthxr.cn/313373.htm
ryj.sthxr.cn/997193.htm
ryh.sthxr.cn/080203.htm
ryh.sthxr.cn/355153.htm
ryh.sthxr.cn/539773.htm
ryh.sthxr.cn/008283.htm
ryh.sthxr.cn/991113.htm
ryh.sthxr.cn/359393.htm
ryh.sthxr.cn/884003.htm
ryh.sthxr.cn/060263.htm
ryh.sthxr.cn/228043.htm
ryh.sthxr.cn/339313.htm
ryg.sthxr.cn/628483.htm
ryg.sthxr.cn/664623.htm
ryg.sthxr.cn/713573.htm
ryg.sthxr.cn/719393.htm
ryg.sthxr.cn/539393.htm
ryg.sthxr.cn/111373.htm
ryg.sthxr.cn/824423.htm
ryg.sthxr.cn/240283.htm
ryg.sthxr.cn/228023.htm
ryg.sthxr.cn/199913.htm
ryf.sthxr.cn/266403.htm
ryf.sthxr.cn/713593.htm
ryf.sthxr.cn/040423.htm
ryf.sthxr.cn/648243.htm
ryf.sthxr.cn/864663.htm
ryf.sthxr.cn/335953.htm
ryf.sthxr.cn/420443.htm
ryf.sthxr.cn/999713.htm
ryf.sthxr.cn/484223.htm
ryf.sthxr.cn/759313.htm
ryd.sthxr.cn/806643.htm
ryd.sthxr.cn/268843.htm
ryd.sthxr.cn/351993.htm
ryd.sthxr.cn/975333.htm
ryd.sthxr.cn/244243.htm
ryd.sthxr.cn/428603.htm
ryd.sthxr.cn/604863.htm
ryd.sthxr.cn/713533.htm
ryd.sthxr.cn/806683.htm
ryd.sthxr.cn/442643.htm
rys.sthxr.cn/668203.htm
rys.sthxr.cn/719373.htm
rys.sthxr.cn/886623.htm
rys.sthxr.cn/739133.htm
rys.sthxr.cn/959333.htm
rys.sthxr.cn/197373.htm
rys.sthxr.cn/513733.htm
rys.sthxr.cn/606483.htm
rys.sthxr.cn/046023.htm
rys.sthxr.cn/939113.htm
rya.sthxr.cn/664063.htm
rya.sthxr.cn/068623.htm
rya.sthxr.cn/557593.htm
rya.sthxr.cn/355713.htm
rya.sthxr.cn/684043.htm
rya.sthxr.cn/044683.htm
rya.sthxr.cn/977573.htm
rya.sthxr.cn/511333.htm
rya.sthxr.cn/004083.htm
rya.sthxr.cn/197353.htm
ryp.sthxr.cn/395933.htm
ryp.sthxr.cn/068463.htm
ryp.sthxr.cn/228063.htm
ryp.sthxr.cn/739593.htm
ryp.sthxr.cn/913713.htm
ryp.sthxr.cn/719913.htm
ryp.sthxr.cn/260803.htm
ryp.sthxr.cn/991713.htm
ryp.sthxr.cn/597533.htm
ryp.sthxr.cn/793553.htm
ryo.sthxr.cn/428603.htm
ryo.sthxr.cn/804043.htm
ryo.sthxr.cn/248463.htm
ryo.sthxr.cn/373713.htm
ryo.sthxr.cn/517333.htm
ryo.sthxr.cn/428683.htm
ryo.sthxr.cn/080803.htm
ryo.sthxr.cn/175973.htm
ryo.sthxr.cn/357313.htm
ryo.sthxr.cn/882883.htm
ryi.sthxr.cn/193973.htm
ryi.sthxr.cn/268083.htm
ryi.sthxr.cn/139533.htm
ryi.sthxr.cn/973173.htm
ryi.sthxr.cn/820443.htm
ryi.sthxr.cn/113393.htm
ryi.sthxr.cn/222483.htm
ryi.sthxr.cn/642023.htm
ryi.sthxr.cn/428603.htm
ryi.sthxr.cn/717993.htm
ryu.sthxr.cn/353333.htm
ryu.sthxr.cn/597733.htm
ryu.sthxr.cn/028803.htm
ryu.sthxr.cn/806063.htm
ryu.sthxr.cn/204643.htm
ryu.sthxr.cn/739773.htm
ryu.sthxr.cn/733173.htm
ryu.sthxr.cn/379773.htm
ryu.sthxr.cn/975973.htm
ryu.sthxr.cn/020283.htm
ryy.sthxr.cn/959173.htm
ryy.sthxr.cn/662443.htm
ryy.sthxr.cn/404063.htm
ryy.sthxr.cn/046023.htm
ryy.sthxr.cn/517353.htm
ryy.sthxr.cn/022023.htm
ryy.sthxr.cn/575373.htm
ryy.sthxr.cn/002023.htm
ryy.sthxr.cn/682843.htm
ryy.sthxr.cn/913333.htm
ryt.sthxr.cn/886283.htm
ryt.sthxr.cn/599393.htm
ryt.sthxr.cn/824203.htm
ryt.sthxr.cn/111513.htm
ryt.sthxr.cn/391573.htm
ryt.sthxr.cn/911773.htm
ryt.sthxr.cn/939133.htm
ryt.sthxr.cn/779113.htm
ryt.sthxr.cn/773153.htm
ryt.sthxr.cn/733313.htm
ryr.sthxr.cn/171133.htm
ryr.sthxr.cn/351753.htm
ryr.sthxr.cn/791733.htm
ryr.sthxr.cn/240083.htm
ryr.sthxr.cn/868243.htm
ryr.sthxr.cn/042883.htm
ryr.sthxr.cn/426863.htm
ryr.sthxr.cn/862863.htm
ryr.sthxr.cn/973353.htm
ryr.sthxr.cn/173313.htm
rye.sthxr.cn/880803.htm
rye.sthxr.cn/664683.htm
rye.sthxr.cn/795533.htm
rye.sthxr.cn/735313.htm
rye.sthxr.cn/375793.htm
rye.sthxr.cn/955113.htm
rye.sthxr.cn/448803.htm
rye.sthxr.cn/197713.htm
rye.sthxr.cn/620843.htm
rye.sthxr.cn/773133.htm
ryw.sthxr.cn/228423.htm
ryw.sthxr.cn/311593.htm
ryw.sthxr.cn/800223.htm
ryw.sthxr.cn/886403.htm
ryw.sthxr.cn/599593.htm
ryw.sthxr.cn/646063.htm
ryw.sthxr.cn/624463.htm
ryw.sthxr.cn/157353.htm
ryw.sthxr.cn/068803.htm
ryw.sthxr.cn/371953.htm
ryq.sthxr.cn/682083.htm
ryq.sthxr.cn/600243.htm
ryq.sthxr.cn/468803.htm
ryq.sthxr.cn/602043.htm
ryq.sthxr.cn/937313.htm
ryq.sthxr.cn/884003.htm
ryq.sthxr.cn/428003.htm
ryq.sthxr.cn/262243.htm
ryq.sthxr.cn/026643.htm
ryq.sthxr.cn/840423.htm
rtm.sthxr.cn/939953.htm
rtm.sthxr.cn/773373.htm
rtm.sthxr.cn/339973.htm
rtm.sthxr.cn/680443.htm
rtm.sthxr.cn/660483.htm
rtm.sthxr.cn/139353.htm
rtm.sthxr.cn/515933.htm
rtm.sthxr.cn/280643.htm
rtm.sthxr.cn/200223.htm
rtm.sthxr.cn/151933.htm
rtn.sthxr.cn/135153.htm
rtn.sthxr.cn/020263.htm
rtn.sthxr.cn/159953.htm
rtn.sthxr.cn/399773.htm
rtn.sthxr.cn/379793.htm
rtn.sthxr.cn/080603.htm
rtn.sthxr.cn/151113.htm
rtn.sthxr.cn/428043.htm
rtn.sthxr.cn/919113.htm
rtn.sthxr.cn/844623.htm
rtb.sthxr.cn/399513.htm
rtb.sthxr.cn/759513.htm
rtb.sthxr.cn/799593.htm
rtb.sthxr.cn/511173.htm
rtb.sthxr.cn/466223.htm
rtb.sthxr.cn/862643.htm
rtb.sthxr.cn/226803.htm
rtb.sthxr.cn/751953.htm
rtb.sthxr.cn/206863.htm
rtb.sthxr.cn/975533.htm
rtv.sthxr.cn/686063.htm
rtv.sthxr.cn/579773.htm
rtv.sthxr.cn/937773.htm
rtv.sthxr.cn/719393.htm
rtv.sthxr.cn/000663.htm
rtv.sthxr.cn/624083.htm
rtv.sthxr.cn/246643.htm
rtv.sthxr.cn/008863.htm
rtv.sthxr.cn/044803.htm
rtv.sthxr.cn/913373.htm
rtc.sthxr.cn/173393.htm
rtc.sthxr.cn/171913.htm
rtc.sthxr.cn/375933.htm
rtc.sthxr.cn/866403.htm
rtc.sthxr.cn/284403.htm
rtc.sthxr.cn/135913.htm
rtc.sthxr.cn/868803.htm
rtc.sthxr.cn/046663.htm
rtc.sthxr.cn/573793.htm
rtc.sthxr.cn/159753.htm
rtx.sthxr.cn/666083.htm
rtx.sthxr.cn/426243.htm
rtx.sthxr.cn/468023.htm
rtx.sthxr.cn/682863.htm
rtx.sthxr.cn/884083.htm
rtx.sthxr.cn/351333.htm
rtx.sthxr.cn/284243.htm
rtx.sthxr.cn/060003.htm
rtx.sthxr.cn/317373.htm
rtx.sthxr.cn/880203.htm
rtz.sthxr.cn/913953.htm
rtz.sthxr.cn/660003.htm
rtz.sthxr.cn/335973.htm
rtz.sthxr.cn/531553.htm
rtz.sthxr.cn/773173.htm
rtz.sthxr.cn/133933.htm
rtz.sthxr.cn/846203.htm
rtz.sthxr.cn/864623.htm
rtz.sthxr.cn/791793.htm
rtz.sthxr.cn/537593.htm
rtl.sthxr.cn/571733.htm
rtl.sthxr.cn/953553.htm
rtl.sthxr.cn/488643.htm
rtl.sthxr.cn/371333.htm
rtl.sthxr.cn/513773.htm
rtl.sthxr.cn/331373.htm
rtl.sthxr.cn/040603.htm
rtl.sthxr.cn/117593.htm
rtl.sthxr.cn/206083.htm
rtl.sthxr.cn/919933.htm
rtk.sthxr.cn/046883.htm
rtk.sthxr.cn/204883.htm
rtk.sthxr.cn/826463.htm
rtk.sthxr.cn/717553.htm
rtk.sthxr.cn/197153.htm
rtk.sthxr.cn/684803.htm
rtk.sthxr.cn/248003.htm
rtk.sthxr.cn/597133.htm
rtk.sthxr.cn/646843.htm
rtk.sthxr.cn/600663.htm
rtj.sthxr.cn/777913.htm
rtj.sthxr.cn/000003.htm
rtj.sthxr.cn/248403.htm
rtj.sthxr.cn/840263.htm
rtj.sthxr.cn/759333.htm
rtj.sthxr.cn/422823.htm
rtj.sthxr.cn/913593.htm
rtj.sthxr.cn/557513.htm
rtj.sthxr.cn/997173.htm
rtj.sthxr.cn/599973.htm
rth.sthxr.cn/315333.htm
rth.sthxr.cn/333953.htm
rth.sthxr.cn/775193.htm
rth.sthxr.cn/575133.htm
rth.sthxr.cn/593953.htm
rth.sthxr.cn/684203.htm
rth.sthxr.cn/977553.htm
rth.sthxr.cn/204283.htm
rth.sthxr.cn/311553.htm
rth.sthxr.cn/197353.htm
rtg.sthxr.cn/840883.htm
rtg.sthxr.cn/442023.htm
rtg.sthxr.cn/375933.htm
rtg.sthxr.cn/462403.htm
rtg.sthxr.cn/066483.htm
rtg.sthxr.cn/224463.htm
rtg.sthxr.cn/884863.htm
rtg.sthxr.cn/288403.htm
rtg.sthxr.cn/684463.htm
rtg.sthxr.cn/428883.htm
rtf.sthxr.cn/353773.htm
rtf.sthxr.cn/537593.htm
rtf.sthxr.cn/824463.htm
rtf.sthxr.cn/602263.htm
rtf.sthxr.cn/317153.htm
rtf.sthxr.cn/133533.htm
rtf.sthxr.cn/713953.htm
rtf.sthxr.cn/408843.htm
rtf.sthxr.cn/715133.htm
rtf.sthxr.cn/880023.htm
rtd.sthxr.cn/393173.htm
rtd.sthxr.cn/319793.htm
rtd.sthxr.cn/082663.htm
rtd.sthxr.cn/935773.htm
rtd.sthxr.cn/848623.htm
rtd.sthxr.cn/426863.htm
rtd.sthxr.cn/759793.htm
rtd.sthxr.cn/448023.htm
rtd.sthxr.cn/482003.htm
rtd.sthxr.cn/973113.htm
rts.sthxr.cn/533193.htm
rts.sthxr.cn/862023.htm
rts.sthxr.cn/242263.htm
rts.sthxr.cn/844643.htm
rts.sthxr.cn/995173.htm
rts.sthxr.cn/840623.htm
rts.sthxr.cn/882663.htm
rts.sthxr.cn/777713.htm
rts.sthxr.cn/951933.htm
rts.sthxr.cn/933933.htm
rta.sthxr.cn/975933.htm
rta.sthxr.cn/848023.htm
rta.sthxr.cn/804603.htm
rta.sthxr.cn/484263.htm
rta.sthxr.cn/048003.htm
rta.sthxr.cn/597733.htm
rta.sthxr.cn/482843.htm
rta.sthxr.cn/644463.htm
rta.sthxr.cn/042023.htm
rta.sthxr.cn/575153.htm
rtp.sthxr.cn/246243.htm
rtp.sthxr.cn/820863.htm
rtp.sthxr.cn/866083.htm
rtp.sthxr.cn/260043.htm
rtp.sthxr.cn/282283.htm
rtp.sthxr.cn/999373.htm
rtp.sthxr.cn/662463.htm
rtp.sthxr.cn/864643.htm
rtp.sthxr.cn/802083.htm
rtp.sthxr.cn/240003.htm
rto.sthxr.cn/866463.htm
rto.sthxr.cn/062843.htm
rto.sthxr.cn/448223.htm
rto.sthxr.cn/684823.htm
rto.sthxr.cn/642403.htm
