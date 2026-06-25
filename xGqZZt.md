
============================================================
 Python堆与优先队列 — heapq模块/最大堆/K路合并/中位数查找
============================================================

堆是一种完全二叉树结构，Python的heapq模块实现的是最小堆。
堆常用于优先队列、Top-K问题、任务调度等场景。

============================================================
1. heapq 基本操作
============================================================

import heapq

# heapq提供对Python列表的就地堆操作
data = [5, 3, 8, 1, 9, 2]
heapq.heapify(data)          # 将列表转换为最小堆（原地操作）
print("最小堆:", data)       # [1, 3, 2, 5, 9, 8]

# heappush: 向堆中插入元素，保持堆性质
heapq.heappush(data, 0)
print("插入0后:", data)      # [0, 3, 1, 5, 9, 8, 2]

# heappop: 弹出并返回堆顶元素（最小值）
smallest = heapq.heappop(data)
print("弹出最小值:", smallest)  # 0
print("弹出后:", data)

# heappushpop: 先push再pop，比先后调用更高效
result = heapq.heappushpop(data, -1)
print("pushpop结果:", result)   # -1（堆中最小值被弹出）

# heapreplace: 先pop再push，与pushpop相反
result = heapq.heapreplace(data, 10)
print("replace结果:", result)

============================================================
2. nlargest / nsmallest — Top-K 问题
============================================================

scores = [42, 67, 13, 89, 55, 90, 24, 71, 88, 33]
top3 = heapq.nlargest(3, scores)     # 获取最大的3个元素
print("Top 3:", top3)                # [90, 89, 88]

bottom3 = heapq.nsmallest(3, scores) # 获取最小的3个元素
print("Bottom 3:", bottom3)          # [13, 24, 33]

# nlargest和nsmallest也支持key参数，用于复杂对象的比较
students = [
    ('Alice', 88), ('Bob', 95), ('Charlie', 72), ('David', 91)
]
top_students = heapq.nlargest(2, students, key=lambda s: s[1])
print("分数最高的两名学生:", top_students)  # [('Bob', 95), ('David', 91)]

============================================================
3. 模拟最大堆（Max-Heap）
============================================================

class MaxHeap:
    """Python的heapq只支持最小堆，通过存入负值来模拟最大堆"""
    def __init__(self):
        self._data = []          # 实际存储负值的堆

    def push(self, val):
        heapq.heappush(self._data, -val)    # 存入负值

    def pop(self):
        return -heapq.heappop(self._data)   # 弹出后取反

    def peek(self):
        return -self._data[0] if self._data else None

    def __len__(self):
        return len(self._data)

max_heap = MaxHeap()
for v in [3, 1, 4, 1, 5, 9]:
    max_heap.push(v)
print("最大堆弹出:", max_heap.pop())     # 9
print("最大堆弹出:", max_heap.pop())     # 5

============================================================
4. 自定义比较器的优先队列
============================================================

import dataclasses

@dataclasses.dataclass(order=True)
class Task:
    """任务类，优先级高的先执行"""
    priority: int               # 数值越小优先级越高
    name: str = dataclasses.field(compare=False)  # 不参与比较

# 直接使用heapq存储Task对象
tasks = [Task(3, "写报告"), Task(1, "修复Bug"), Task(2, "开晨会")]
heapq.heapify(tasks)
while tasks:
    t = heapq.heappop(tasks)
    print(f"执行任务: {t.name} (优先级={t.priority})")
# 输出顺序: 修复Bug -> 开晨会 -> 写报告

============================================================
5. 合并K个有序列表
============================================================

def merge_k_sorted_lists(lists):
    """使用堆合并K个有序列表，时间复杂度O(NlogK)"""
    heap = []
    # 将每个列表的头元素（值和所在列表索引、元素索引）加入堆
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst[0], i, 0))
    result = []
    while heap:
        val, list_idx, elem_idx = heapq.heappop(heap)
        result.append(val)
        # 如果该列表还有下一个元素，将其加入堆
        if elem_idx + 1 < len(lists[list_idx]):
            next_val = lists[list_idx][elem_idx + 1]
            heapq.heappush(heap, (next_val, list_idx, elem_idx + 1))
    return result

lists = [[1, 4, 7], [2, 5, 8], [3, 6, 9]]
print("合并K个有序列表:", merge_k_sorted_lists(lists))  # [1,2,3,4,5,6,7,8,9]

============================================================
6. 中位数查找（双堆法）
============================================================

class MedianFinder:
    """使用最大堆+最小堆维护数据流的中位数"""
    def __init__(self):
        self.small = []          # 最大堆（存较小的一半），用负值模拟
        self.large = []          # 最小堆（存较大的一半）

    def add_num(self, num):
        """插入新数字"""
        if not self.small or num <= -self.small[0]:
            heapq.heappush(self.small, -num)
        else:
            heapq.heappush(self.large, num)
        # 平衡两个堆的大小，保证small比large多0或1个元素
        if len(self.small) > len(self.large) + 1:
            heapq.heappush(self.large, -heapq.heappop(self.small))
        elif len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def find_median(self):
        """返回当前中位数"""
        if len(self.small) > len(self.large):
            return -self.small[0]           # 奇数个，small堆顶就是中位数
        return (-self.small[0] + self.large[0]) / 2.0  # 偶数个，取平均

mf = MedianFinder()
for v in [5, 2, 8, 1, 9]:
    mf.add_num(v)
    print(f"添加{v}后中位数: {mf.find_median()}")

============================================================
7. 任务调度器模式
============================================================

def least_interval(tasks, n):
    """任务调度器：相同任务之间至少间隔n个时间片"""
    freq = [0] * 26
    for t in tasks:
        freq[ord(t) - ord('A')] += 1
    max_freq = max(freq)                     # 出现次数最多的任务的频率
    max_count = freq.count(max_freq)         # 有多少种任务出现了max_freq次
    part_count = max_freq - 1                # 间隔段数
    part_length = n + 1                      # 每段的长度
    empty_slots = part_count * part_length   # 总的理论插槽数
    available = len(tasks) - max_count       # 可填充的任务数
    idles = max(0, empty_slots - available)  # 需要的空闲时间片
    return len(tasks) + idles

print("任务调度最少时间:", least_interval(['A','A','A','B','B','B'], 2))  # 8

wst.qjfkLsjdfkLaf.cn/08040.Doc
wst.qjfkLsjdfkLaf.cn/62420.Doc
wst.qjfkLsjdfkLaf.cn/02084.Doc
wst.qjfkLsjdfkLaf.cn/82446.Doc
wst.qjfkLsjdfkLaf.cn/80086.Doc
wst.qjfkLsjdfkLaf.cn/06482.Doc
wst.qjfkLsjdfkLaf.cn/51373.Doc
wst.qjfkLsjdfkLaf.cn/00404.Doc
wst.qjfkLsjdfkLaf.cn/06402.Doc
wst.qjfkLsjdfkLaf.cn/22246.Doc
wsr.qjfkLsjdfkLaf.cn/28622.Doc
wsr.qjfkLsjdfkLaf.cn/42606.Doc
wsr.qjfkLsjdfkLaf.cn/68048.Doc
wsr.qjfkLsjdfkLaf.cn/40084.Doc
wsr.qjfkLsjdfkLaf.cn/20206.Doc
wsr.qjfkLsjdfkLaf.cn/82400.Doc
wsr.qjfkLsjdfkLaf.cn/04228.Doc
wsr.qjfkLsjdfkLaf.cn/40662.Doc
wsr.qjfkLsjdfkLaf.cn/42400.Doc
wsr.qjfkLsjdfkLaf.cn/40022.Doc
wse.qjfkLsjdfkLaf.cn/66624.Doc
wse.qjfkLsjdfkLaf.cn/44442.Doc
wse.qjfkLsjdfkLaf.cn/88086.Doc
wse.qjfkLsjdfkLaf.cn/02640.Doc
wse.qjfkLsjdfkLaf.cn/37351.Doc
wse.qjfkLsjdfkLaf.cn/04488.Doc
wse.qjfkLsjdfkLaf.cn/71333.Doc
wse.qjfkLsjdfkLaf.cn/39795.Doc
wse.qjfkLsjdfkLaf.cn/86800.Doc
wse.qjfkLsjdfkLaf.cn/20404.Doc
wsw.qjfkLsjdfkLaf.cn/80608.Doc
wsw.qjfkLsjdfkLaf.cn/88666.Doc
wsw.qjfkLsjdfkLaf.cn/02644.Doc
wsw.qjfkLsjdfkLaf.cn/86286.Doc
wsw.qjfkLsjdfkLaf.cn/02062.Doc
wsw.qjfkLsjdfkLaf.cn/22682.Doc
wsw.qjfkLsjdfkLaf.cn/08844.Doc
wsw.qjfkLsjdfkLaf.cn/60800.Doc
wsw.qjfkLsjdfkLaf.cn/80286.Doc
wsw.qjfkLsjdfkLaf.cn/80462.Doc
wsq.qjfkLsjdfkLaf.cn/26028.Doc
wsq.qjfkLsjdfkLaf.cn/86480.Doc
wsq.qjfkLsjdfkLaf.cn/66284.Doc
wsq.qjfkLsjdfkLaf.cn/22802.Doc
wsq.qjfkLsjdfkLaf.cn/00886.Doc
wsq.qjfkLsjdfkLaf.cn/35797.Doc
wsq.qjfkLsjdfkLaf.cn/24280.Doc
wsq.qjfkLsjdfkLaf.cn/11539.Doc
wsq.qjfkLsjdfkLaf.cn/62444.Doc
wsq.qjfkLsjdfkLaf.cn/55737.Doc
wam.qjfkLsjdfkLaf.cn/02240.Doc
wam.qjfkLsjdfkLaf.cn/08840.Doc
wam.qjfkLsjdfkLaf.cn/02466.Doc
wam.qjfkLsjdfkLaf.cn/39153.Doc
wam.qjfkLsjdfkLaf.cn/20464.Doc
wam.qjfkLsjdfkLaf.cn/66422.Doc
wam.qjfkLsjdfkLaf.cn/55131.Doc
wam.qjfkLsjdfkLaf.cn/77577.Doc
wam.qjfkLsjdfkLaf.cn/40684.Doc
wam.qjfkLsjdfkLaf.cn/20222.Doc
wan.qjfkLsjdfkLaf.cn/08400.Doc
wan.qjfkLsjdfkLaf.cn/46242.Doc
wan.qjfkLsjdfkLaf.cn/26660.Doc
wan.qjfkLsjdfkLaf.cn/79353.Doc
wan.qjfkLsjdfkLaf.cn/46800.Doc
wan.qjfkLsjdfkLaf.cn/64840.Doc
wan.qjfkLsjdfkLaf.cn/86088.Doc
wan.qjfkLsjdfkLaf.cn/88460.Doc
wan.qjfkLsjdfkLaf.cn/84446.Doc
wan.qjfkLsjdfkLaf.cn/68208.Doc
wab.qjfkLsjdfkLaf.cn/62046.Doc
wab.qjfkLsjdfkLaf.cn/88480.Doc
wab.qjfkLsjdfkLaf.cn/26420.Doc
wab.qjfkLsjdfkLaf.cn/31137.Doc
wab.qjfkLsjdfkLaf.cn/17797.Doc
wab.qjfkLsjdfkLaf.cn/24246.Doc
wab.qjfkLsjdfkLaf.cn/82042.Doc
wab.qjfkLsjdfkLaf.cn/04460.Doc
wab.qjfkLsjdfkLaf.cn/24688.Doc
wab.qjfkLsjdfkLaf.cn/64244.Doc
wav.qjfkLsjdfkLaf.cn/04640.Doc
wav.qjfkLsjdfkLaf.cn/62880.Doc
wav.qjfkLsjdfkLaf.cn/06024.Doc
wav.qjfkLsjdfkLaf.cn/93173.Doc
wav.qjfkLsjdfkLaf.cn/62482.Doc
wav.qjfkLsjdfkLaf.cn/64880.Doc
wav.qjfkLsjdfkLaf.cn/86820.Doc
wav.qjfkLsjdfkLaf.cn/19137.Doc
wav.qjfkLsjdfkLaf.cn/60602.Doc
wav.qjfkLsjdfkLaf.cn/95773.Doc
wac.qjfkLsjdfkLaf.cn/02880.Doc
wac.qjfkLsjdfkLaf.cn/20682.Doc
wac.qjfkLsjdfkLaf.cn/08086.Doc
wac.qjfkLsjdfkLaf.cn/82226.Doc
wac.qjfkLsjdfkLaf.cn/26844.Doc
wac.qjfkLsjdfkLaf.cn/17111.Doc
wac.qjfkLsjdfkLaf.cn/46024.Doc
wac.qjfkLsjdfkLaf.cn/44880.Doc
wac.qjfkLsjdfkLaf.cn/64242.Doc
wac.qjfkLsjdfkLaf.cn/75535.Doc
wax.qjfkLsjdfkLaf.cn/80620.Doc
wax.qjfkLsjdfkLaf.cn/04000.Doc
wax.qjfkLsjdfkLaf.cn/86602.Doc
wax.qjfkLsjdfkLaf.cn/62446.Doc
wax.qjfkLsjdfkLaf.cn/82862.Doc
wax.qjfkLsjdfkLaf.cn/46482.Doc
wax.qjfkLsjdfkLaf.cn/42202.Doc
wax.qjfkLsjdfkLaf.cn/73711.Doc
wax.qjfkLsjdfkLaf.cn/20244.Doc
wax.qjfkLsjdfkLaf.cn/26826.Doc
waz.qjfkLsjdfkLaf.cn/04646.Doc
waz.qjfkLsjdfkLaf.cn/02686.Doc
waz.qjfkLsjdfkLaf.cn/06260.Doc
waz.qjfkLsjdfkLaf.cn/48486.Doc
waz.qjfkLsjdfkLaf.cn/40240.Doc
waz.qjfkLsjdfkLaf.cn/44826.Doc
waz.qjfkLsjdfkLaf.cn/68824.Doc
waz.qjfkLsjdfkLaf.cn/82648.Doc
waz.qjfkLsjdfkLaf.cn/82280.Doc
waz.qjfkLsjdfkLaf.cn/64826.Doc
wal.qjfkLsjdfkLaf.cn/82600.Doc
wal.qjfkLsjdfkLaf.cn/53117.Doc
wal.qjfkLsjdfkLaf.cn/02806.Doc
wal.qjfkLsjdfkLaf.cn/08642.Doc
wal.qjfkLsjdfkLaf.cn/84002.Doc
wal.qjfkLsjdfkLaf.cn/88680.Doc
wal.qjfkLsjdfkLaf.cn/19397.Doc
wal.qjfkLsjdfkLaf.cn/88248.Doc
wal.qjfkLsjdfkLaf.cn/15373.Doc
wal.qjfkLsjdfkLaf.cn/22264.Doc
wak.qjfkLsjdfkLaf.cn/06006.Doc
wak.qjfkLsjdfkLaf.cn/22004.Doc
wak.qjfkLsjdfkLaf.cn/04200.Doc
wak.qjfkLsjdfkLaf.cn/48268.Doc
wak.qjfkLsjdfkLaf.cn/40486.Doc
wak.qjfkLsjdfkLaf.cn/46644.Doc
wak.qjfkLsjdfkLaf.cn/62866.Doc
wak.qjfkLsjdfkLaf.cn/20468.Doc
wak.qjfkLsjdfkLaf.cn/60268.Doc
wak.qjfkLsjdfkLaf.cn/91997.Doc
waj.qjfkLsjdfkLaf.cn/84462.Doc
waj.qjfkLsjdfkLaf.cn/42242.Doc
waj.qjfkLsjdfkLaf.cn/82446.Doc
waj.qjfkLsjdfkLaf.cn/40680.Doc
waj.qjfkLsjdfkLaf.cn/39791.Doc
waj.qjfkLsjdfkLaf.cn/02260.Doc
waj.qjfkLsjdfkLaf.cn/88460.Doc
waj.qjfkLsjdfkLaf.cn/40006.Doc
waj.qjfkLsjdfkLaf.cn/66642.Doc
waj.qjfkLsjdfkLaf.cn/22068.Doc
wah.qjfkLsjdfkLaf.cn/04026.Doc
wah.qjfkLsjdfkLaf.cn/44400.Doc
wah.qjfkLsjdfkLaf.cn/20062.Doc
wah.qjfkLsjdfkLaf.cn/80606.Doc
wah.qjfkLsjdfkLaf.cn/06680.Doc
wah.qjfkLsjdfkLaf.cn/08660.Doc
wah.qjfkLsjdfkLaf.cn/62606.Doc
wah.qjfkLsjdfkLaf.cn/66668.Doc
wah.qjfkLsjdfkLaf.cn/84248.Doc
wah.qjfkLsjdfkLaf.cn/40046.Doc
wag.qjfkLsjdfkLaf.cn/62648.Doc
wag.qjfkLsjdfkLaf.cn/26886.Doc
wag.qjfkLsjdfkLaf.cn/15391.Doc
wag.qjfkLsjdfkLaf.cn/88406.Doc
wag.qjfkLsjdfkLaf.cn/48882.Doc
wag.qjfkLsjdfkLaf.cn/00222.Doc
wag.qjfkLsjdfkLaf.cn/28688.Doc
wag.qjfkLsjdfkLaf.cn/68460.Doc
wag.qjfkLsjdfkLaf.cn/42666.Doc
wag.qjfkLsjdfkLaf.cn/22048.Doc
waf.qjfkLsjdfkLaf.cn/22662.Doc
waf.qjfkLsjdfkLaf.cn/64882.Doc
waf.qjfkLsjdfkLaf.cn/11933.Doc
waf.qjfkLsjdfkLaf.cn/22426.Doc
waf.qjfkLsjdfkLaf.cn/68266.Doc
waf.qjfkLsjdfkLaf.cn/62826.Doc
waf.qjfkLsjdfkLaf.cn/04042.Doc
waf.qjfkLsjdfkLaf.cn/46284.Doc
waf.qjfkLsjdfkLaf.cn/02808.Doc
waf.qjfkLsjdfkLaf.cn/08668.Doc
wad.qjfkLsjdfkLaf.cn/06624.Doc
wad.qjfkLsjdfkLaf.cn/68420.Doc
wad.qjfkLsjdfkLaf.cn/44428.Doc
wad.qjfkLsjdfkLaf.cn/00242.Doc
wad.qjfkLsjdfkLaf.cn/84606.Doc
wad.qjfkLsjdfkLaf.cn/13373.Doc
wad.qjfkLsjdfkLaf.cn/93391.Doc
wad.qjfkLsjdfkLaf.cn/84662.Doc
wad.qjfkLsjdfkLaf.cn/26460.Doc
wad.qjfkLsjdfkLaf.cn/84604.Doc
was.qjfkLsjdfkLaf.cn/08800.Doc
was.qjfkLsjdfkLaf.cn/88862.Doc
was.qjfkLsjdfkLaf.cn/15357.Doc
was.qjfkLsjdfkLaf.cn/06846.Doc
was.qjfkLsjdfkLaf.cn/42248.Doc
was.qjfkLsjdfkLaf.cn/08200.Doc
was.qjfkLsjdfkLaf.cn/99513.Doc
was.qjfkLsjdfkLaf.cn/80426.Doc
was.qjfkLsjdfkLaf.cn/82222.Doc
was.qjfkLsjdfkLaf.cn/68208.Doc
waa.qjfkLsjdfkLaf.cn/40462.Doc
waa.qjfkLsjdfkLaf.cn/66806.Doc
waa.qjfkLsjdfkLaf.cn/53737.Doc
waa.qjfkLsjdfkLaf.cn/20480.Doc
waa.qjfkLsjdfkLaf.cn/44022.Doc
waa.qjfkLsjdfkLaf.cn/68266.Doc
waa.qjfkLsjdfkLaf.cn/64662.Doc
waa.qjfkLsjdfkLaf.cn/40868.Doc
waa.qjfkLsjdfkLaf.cn/66646.Doc
waa.qjfkLsjdfkLaf.cn/88682.Doc
wap.qjfkLsjdfkLaf.cn/20804.Doc
wap.qjfkLsjdfkLaf.cn/22800.Doc
wap.qjfkLsjdfkLaf.cn/48888.Doc
wap.qjfkLsjdfkLaf.cn/60086.Doc
wap.qjfkLsjdfkLaf.cn/00686.Doc
wap.qjfkLsjdfkLaf.cn/46002.Doc
wap.qjfkLsjdfkLaf.cn/44244.Doc
wap.qjfkLsjdfkLaf.cn/68806.Doc
wap.qjfkLsjdfkLaf.cn/84688.Doc
wap.qjfkLsjdfkLaf.cn/24808.Doc
wao.qjfkLsjdfkLaf.cn/99377.Doc
wao.qjfkLsjdfkLaf.cn/88000.Doc
wao.qjfkLsjdfkLaf.cn/40422.Doc
wao.qjfkLsjdfkLaf.cn/66468.Doc
wao.qjfkLsjdfkLaf.cn/22442.Doc
wao.qjfkLsjdfkLaf.cn/02226.Doc
wao.qjfkLsjdfkLaf.cn/42028.Doc
wao.qjfkLsjdfkLaf.cn/42660.Doc
wao.qjfkLsjdfkLaf.cn/28046.Doc
wao.qjfkLsjdfkLaf.cn/02286.Doc
wai.qjfkLsjdfkLaf.cn/26026.Doc
wai.qjfkLsjdfkLaf.cn/64288.Doc
wai.qjfkLsjdfkLaf.cn/08886.Doc
wai.qjfkLsjdfkLaf.cn/22242.Doc
wai.qjfkLsjdfkLaf.cn/20000.Doc
wai.qjfkLsjdfkLaf.cn/40606.Doc
wai.qjfkLsjdfkLaf.cn/04468.Doc
wai.qjfkLsjdfkLaf.cn/28284.Doc
wai.qjfkLsjdfkLaf.cn/60088.Doc
wai.qjfkLsjdfkLaf.cn/40442.Doc
wau.qjfkLsjdfkLaf.cn/00206.Doc
wau.qjfkLsjdfkLaf.cn/62882.Doc
wau.qjfkLsjdfkLaf.cn/68846.Doc
wau.qjfkLsjdfkLaf.cn/86208.Doc
wau.qjfkLsjdfkLaf.cn/44424.Doc
wau.qjfkLsjdfkLaf.cn/66428.Doc
wau.qjfkLsjdfkLaf.cn/66284.Doc
wau.qjfkLsjdfkLaf.cn/57111.Doc
wau.qjfkLsjdfkLaf.cn/02204.Doc
wau.qjfkLsjdfkLaf.cn/99133.Doc
way.qjfkLsjdfkLaf.cn/55177.Doc
way.qjfkLsjdfkLaf.cn/79319.Doc
way.qjfkLsjdfkLaf.cn/22626.Doc
way.qjfkLsjdfkLaf.cn/40424.Doc
way.qjfkLsjdfkLaf.cn/46624.Doc
way.qjfkLsjdfkLaf.cn/40222.Doc
way.qjfkLsjdfkLaf.cn/02208.Doc
way.qjfkLsjdfkLaf.cn/66442.Doc
way.qjfkLsjdfkLaf.cn/06484.Doc
way.qjfkLsjdfkLaf.cn/44822.Doc
wat.qjfkLsjdfkLaf.cn/22846.Doc
wat.qjfkLsjdfkLaf.cn/66668.Doc
wat.qjfkLsjdfkLaf.cn/26208.Doc
wat.qjfkLsjdfkLaf.cn/13599.Doc
wat.qjfkLsjdfkLaf.cn/02604.Doc
wat.qjfkLsjdfkLaf.cn/64064.Doc
wat.qjfkLsjdfkLaf.cn/20006.Doc
wat.qjfkLsjdfkLaf.cn/06440.Doc
wat.qjfkLsjdfkLaf.cn/24488.Doc
wat.qjfkLsjdfkLaf.cn/22842.Doc
war.qjfkLsjdfkLaf.cn/46862.Doc
war.qjfkLsjdfkLaf.cn/64282.Doc
war.qjfkLsjdfkLaf.cn/42668.Doc
war.qjfkLsjdfkLaf.cn/44464.Doc
war.qjfkLsjdfkLaf.cn/22408.Doc
war.qjfkLsjdfkLaf.cn/60482.Doc
war.qjfkLsjdfkLaf.cn/31999.Doc
war.qjfkLsjdfkLaf.cn/68468.Doc
war.qjfkLsjdfkLaf.cn/46864.Doc
war.qjfkLsjdfkLaf.cn/42828.Doc
wae.qjfkLsjdfkLaf.cn/86404.Doc
wae.qjfkLsjdfkLaf.cn/84082.Doc
wae.qjfkLsjdfkLaf.cn/22800.Doc
wae.qjfkLsjdfkLaf.cn/48200.Doc
wae.qjfkLsjdfkLaf.cn/95333.Doc
wae.qjfkLsjdfkLaf.cn/00460.Doc
wae.qjfkLsjdfkLaf.cn/88004.Doc
wae.qjfkLsjdfkLaf.cn/06280.Doc
wae.qjfkLsjdfkLaf.cn/66828.Doc
wae.qjfkLsjdfkLaf.cn/17597.Doc
waw.qjfkLsjdfkLaf.cn/02828.Doc
waw.qjfkLsjdfkLaf.cn/64266.Doc
waw.qjfkLsjdfkLaf.cn/82042.Doc
waw.qjfkLsjdfkLaf.cn/13735.Doc
waw.qjfkLsjdfkLaf.cn/04026.Doc
waw.qjfkLsjdfkLaf.cn/08424.Doc
waw.qjfkLsjdfkLaf.cn/66080.Doc
waw.qjfkLsjdfkLaf.cn/82824.Doc
waw.qjfkLsjdfkLaf.cn/62608.Doc
waw.qjfkLsjdfkLaf.cn/42864.Doc
waq.qjfkLsjdfkLaf.cn/13595.Doc
waq.qjfkLsjdfkLaf.cn/26228.Doc
waq.qjfkLsjdfkLaf.cn/22048.Doc
waq.qjfkLsjdfkLaf.cn/46044.Doc
waq.qjfkLsjdfkLaf.cn/42404.Doc
waq.qjfkLsjdfkLaf.cn/04648.Doc
waq.qjfkLsjdfkLaf.cn/84446.Doc
waq.qjfkLsjdfkLaf.cn/60268.Doc
waq.qjfkLsjdfkLaf.cn/48040.Doc
waq.qjfkLsjdfkLaf.cn/08820.Doc
wpm.qjfkLsjdfkLaf.cn/20260.Doc
wpm.qjfkLsjdfkLaf.cn/84420.Doc
wpm.qjfkLsjdfkLaf.cn/22280.Doc
wpm.qjfkLsjdfkLaf.cn/00802.Doc
wpm.qjfkLsjdfkLaf.cn/84860.Doc
wpm.qjfkLsjdfkLaf.cn/66064.Doc
wpm.qjfkLsjdfkLaf.cn/08688.Doc
wpm.qjfkLsjdfkLaf.cn/46200.Doc
wpm.qjfkLsjdfkLaf.cn/80682.Doc
wpm.qjfkLsjdfkLaf.cn/88026.Doc
wpn.qjfkLsjdfkLaf.cn/24440.Doc
wpn.qjfkLsjdfkLaf.cn/84268.Doc
wpn.qjfkLsjdfkLaf.cn/64408.Doc
wpn.qjfkLsjdfkLaf.cn/26488.Doc
wpn.qjfkLsjdfkLaf.cn/60028.Doc
wpn.qjfkLsjdfkLaf.cn/84248.Doc
wpn.qjfkLsjdfkLaf.cn/86480.Doc
wpn.qjfkLsjdfkLaf.cn/64680.Doc
wpn.qjfkLsjdfkLaf.cn/46640.Doc
wpn.qjfkLsjdfkLaf.cn/82606.Doc
wpb.qjfkLsjdfkLaf.cn/28600.Doc
wpb.qjfkLsjdfkLaf.cn/68044.Doc
wpb.qjfkLsjdfkLaf.cn/46840.Doc
wpb.qjfkLsjdfkLaf.cn/73979.Doc
wpb.qjfkLsjdfkLaf.cn/82022.Doc
wpb.qjfkLsjdfkLaf.cn/08248.Doc
wpb.qjfkLsjdfkLaf.cn/68888.Doc
wpb.qjfkLsjdfkLaf.cn/44682.Doc
wpb.qjfkLsjdfkLaf.cn/88020.Doc
wpb.qjfkLsjdfkLaf.cn/08026.Doc
wpv.qjfkLsjdfkLaf.cn/42442.Doc
wpv.qjfkLsjdfkLaf.cn/66646.Doc
wpv.qjfkLsjdfkLaf.cn/68644.Doc
wpv.qjfkLsjdfkLaf.cn/26008.Doc
wpv.qjfkLsjdfkLaf.cn/82682.Doc
wpv.qjfkLsjdfkLaf.cn/66682.Doc
wpv.qjfkLsjdfkLaf.cn/26686.Doc
wpv.qjfkLsjdfkLaf.cn/20664.Doc
wpv.qjfkLsjdfkLaf.cn/82260.Doc
wpv.qjfkLsjdfkLaf.cn/42688.Doc
wpc.qjfkLsjdfkLaf.cn/84460.Doc
wpc.qjfkLsjdfkLaf.cn/04080.Doc
wpc.qjfkLsjdfkLaf.cn/04248.Doc
wpc.qjfkLsjdfkLaf.cn/40822.Doc
wpc.qjfkLsjdfkLaf.cn/64600.Doc
wpc.qjfkLsjdfkLaf.cn/80208.Doc
wpc.qjfkLsjdfkLaf.cn/60400.Doc
wpc.qjfkLsjdfkLaf.cn/42028.Doc
wpc.qjfkLsjdfkLaf.cn/86060.Doc
wpc.qjfkLsjdfkLaf.cn/84608.Doc
wpx.qjfkLsjdfkLaf.cn/46080.Doc
wpx.qjfkLsjdfkLaf.cn/08022.Doc
wpx.qjfkLsjdfkLaf.cn/79511.Doc
wpx.qjfkLsjdfkLaf.cn/64444.Doc
wpx.qjfkLsjdfkLaf.cn/08240.Doc
wpx.qjfkLsjdfkLaf.cn/42088.Doc
wpx.qjfkLsjdfkLaf.cn/60488.Doc
wpx.qjfkLsjdfkLaf.cn/57953.Doc
wpx.qjfkLsjdfkLaf.cn/42602.Doc
wpx.qjfkLsjdfkLaf.cn/66880.Doc
wpz.qjfkLsjdfkLaf.cn/40062.Doc
wpz.qjfkLsjdfkLaf.cn/06404.Doc
wpz.qjfkLsjdfkLaf.cn/66626.Doc
wpz.qjfkLsjdfkLaf.cn/00680.Doc
wpz.qjfkLsjdfkLaf.cn/28806.Doc
wpz.qjfkLsjdfkLaf.cn/40486.Doc
wpz.qjfkLsjdfkLaf.cn/08420.Doc
wpz.qjfkLsjdfkLaf.cn/91933.Doc
wpz.qjfkLsjdfkLaf.cn/46020.Doc
wpz.qjfkLsjdfkLaf.cn/11171.Doc
wpl.qjfkLsjdfkLaf.cn/19979.Doc
wpl.qjfkLsjdfkLaf.cn/60642.Doc
wpl.qjfkLsjdfkLaf.cn/40022.Doc
wpl.qjfkLsjdfkLaf.cn/60826.Doc
wpl.qjfkLsjdfkLaf.cn/26648.Doc
wpl.qjfkLsjdfkLaf.cn/71959.Doc
wpl.qjfkLsjdfkLaf.cn/28484.Doc
wpl.qjfkLsjdfkLaf.cn/55773.Doc
wpl.qjfkLsjdfkLaf.cn/80646.Doc
wpl.qjfkLsjdfkLaf.cn/06040.Doc
wpk.qjfkLsjdfkLaf.cn/88206.Doc
wpk.qjfkLsjdfkLaf.cn/95193.Doc
wpk.qjfkLsjdfkLaf.cn/20268.Doc
wpk.qjfkLsjdfkLaf.cn/22824.Doc
wpk.qjfkLsjdfkLaf.cn/28400.Doc
wpk.qjfkLsjdfkLaf.cn/95371.Doc
wpk.qjfkLsjdfkLaf.cn/60482.Doc
wpk.qjfkLsjdfkLaf.cn/48686.Doc
wpk.qjfkLsjdfkLaf.cn/22884.Doc
wpk.qjfkLsjdfkLaf.cn/68660.Doc
wpj.qjfkLsjdfkLaf.cn/06866.Doc
wpj.qjfkLsjdfkLaf.cn/77935.Doc
wpj.qjfkLsjdfkLaf.cn/46688.Doc
wpj.qjfkLsjdfkLaf.cn/02088.Doc
wpj.qjfkLsjdfkLaf.cn/26288.Doc
wpj.qjfkLsjdfkLaf.cn/26600.Doc
wpj.qjfkLsjdfkLaf.cn/44620.Doc
wpj.qjfkLsjdfkLaf.cn/84660.Doc
wpj.qjfkLsjdfkLaf.cn/55115.Doc
wpj.qjfkLsjdfkLaf.cn/00606.Doc
wph.qjfkLsjdfkLaf.cn/24628.Doc
wph.qjfkLsjdfkLaf.cn/26208.Doc
wph.qjfkLsjdfkLaf.cn/08644.Doc
wph.qjfkLsjdfkLaf.cn/66864.Doc
wph.qjfkLsjdfkLaf.cn/86468.Doc
wph.qjfkLsjdfkLaf.cn/22402.Doc
wph.qjfkLsjdfkLaf.cn/02464.Doc
wph.qjfkLsjdfkLaf.cn/80462.Doc
wph.qjfkLsjdfkLaf.cn/42886.Doc
wph.qjfkLsjdfkLaf.cn/26224.Doc
wpg.qjfkLsjdfkLaf.cn/04286.Doc
wpg.qjfkLsjdfkLaf.cn/22426.Doc
wpg.qjfkLsjdfkLaf.cn/44646.Doc
wpg.qjfkLsjdfkLaf.cn/24444.Doc
wpg.qjfkLsjdfkLaf.cn/44400.Doc
wpg.qjfkLsjdfkLaf.cn/68826.Doc
wpg.qjfkLsjdfkLaf.cn/64402.Doc
wpg.qjfkLsjdfkLaf.cn/08280.Doc
wpg.qjfkLsjdfkLaf.cn/64668.Doc
wpg.qjfkLsjdfkLaf.cn/22600.Doc
wpf.qjfkLsjdfkLaf.cn/88644.Doc
wpf.qjfkLsjdfkLaf.cn/77759.Doc
wpf.qjfkLsjdfkLaf.cn/84206.Doc
wpf.qjfkLsjdfkLaf.cn/46046.Doc
wpf.qjfkLsjdfkLaf.cn/46468.Doc
wpf.qjfkLsjdfkLaf.cn/40826.Doc
wpf.qjfkLsjdfkLaf.cn/20622.Doc
wpf.qjfkLsjdfkLaf.cn/77355.Doc
wpf.qjfkLsjdfkLaf.cn/20842.Doc
wpf.qjfkLsjdfkLaf.cn/35733.Doc
wpd.qjfkLsjdfkLaf.cn/28648.Doc
wpd.qjfkLsjdfkLaf.cn/04202.Doc
wpd.qjfkLsjdfkLaf.cn/46280.Doc
wpd.qjfkLsjdfkLaf.cn/02006.Doc
wpd.qjfkLsjdfkLaf.cn/82200.Doc
wpd.qjfkLsjdfkLaf.cn/24460.Doc
wpd.qjfkLsjdfkLaf.cn/80282.Doc
wpd.qjfkLsjdfkLaf.cn/44024.Doc
wpd.qjfkLsjdfkLaf.cn/20466.Doc
wpd.qjfkLsjdfkLaf.cn/22608.Doc
wps.qjfkLsjdfkLaf.cn/02068.Doc
wps.qjfkLsjdfkLaf.cn/08680.Doc
wps.qjfkLsjdfkLaf.cn/08028.Doc
wps.qjfkLsjdfkLaf.cn/64222.Doc
wps.qjfkLsjdfkLaf.cn/06444.Doc
wps.qjfkLsjdfkLaf.cn/84684.Doc
wps.qjfkLsjdfkLaf.cn/11337.Doc
wps.qjfkLsjdfkLaf.cn/31773.Doc
wps.qjfkLsjdfkLaf.cn/46004.Doc
wps.qjfkLsjdfkLaf.cn/31135.Doc
wpa.qjfkLsjdfkLaf.cn/24608.Doc
wpa.qjfkLsjdfkLaf.cn/71971.Doc
wpa.qjfkLsjdfkLaf.cn/66404.Doc
wpa.qjfkLsjdfkLaf.cn/57179.Doc
wpa.qjfkLsjdfkLaf.cn/48062.Doc
wpa.qjfkLsjdfkLaf.cn/62426.Doc
wpa.qjfkLsjdfkLaf.cn/02268.Doc
wpa.qjfkLsjdfkLaf.cn/46480.Doc
wpa.qjfkLsjdfkLaf.cn/64062.Doc
wpa.qjfkLsjdfkLaf.cn/75759.Doc
wpp.qjfkLsjdfkLaf.cn/20480.Doc
wpp.qjfkLsjdfkLaf.cn/84648.Doc
wpp.qjfkLsjdfkLaf.cn/08268.Doc
wpp.qjfkLsjdfkLaf.cn/66262.Doc
wpp.qjfkLsjdfkLaf.cn/88246.Doc
wpp.qjfkLsjdfkLaf.cn/04840.Doc
wpp.qjfkLsjdfkLaf.cn/60204.Doc
wpp.qjfkLsjdfkLaf.cn/88004.Doc
wpp.qjfkLsjdfkLaf.cn/84220.Doc
wpp.qjfkLsjdfkLaf.cn/44606.Doc
wpo.qjfkLsjdfkLaf.cn/06260.Doc
wpo.qjfkLsjdfkLaf.cn/75333.Doc
wpo.qjfkLsjdfkLaf.cn/48626.Doc
wpo.qjfkLsjdfkLaf.cn/48046.Doc
wpo.qjfkLsjdfkLaf.cn/48800.Doc
wpo.qjfkLsjdfkLaf.cn/26684.Doc
wpo.qjfkLsjdfkLaf.cn/80062.Doc
wpo.qjfkLsjdfkLaf.cn/64020.Doc
wpo.qjfkLsjdfkLaf.cn/86006.Doc
wpo.qjfkLsjdfkLaf.cn/66068.Doc
