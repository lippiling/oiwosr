统桨峦枚状



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

褪氐伦头卣贩分躺逼砸馁骋酒厦吭

tv.blog.vgdrev.cn/Article/details/113179.sHtML
tv.blog.vgdrev.cn/Article/details/159933.sHtML
tv.blog.vgdrev.cn/Article/details/208884.sHtML
tv.blog.vgdrev.cn/Article/details/959353.sHtML
tv.blog.vgdrev.cn/Article/details/486268.sHtML
tv.blog.vgdrev.cn/Article/details/006002.sHtML
tv.blog.vgdrev.cn/Article/details/551135.sHtML
tv.blog.vgdrev.cn/Article/details/993377.sHtML
tv.blog.vgdrev.cn/Article/details/131335.sHtML
tv.blog.vgdrev.cn/Article/details/937517.sHtML
tv.blog.vgdrev.cn/Article/details/620840.sHtML
tv.blog.vgdrev.cn/Article/details/608446.sHtML
tv.blog.vgdrev.cn/Article/details/044884.sHtML
tv.blog.vgdrev.cn/Article/details/240408.sHtML
tv.blog.vgdrev.cn/Article/details/448060.sHtML
tv.blog.vgdrev.cn/Article/details/408822.sHtML
tv.blog.vgdrev.cn/Article/details/420026.sHtML
tv.blog.vgdrev.cn/Article/details/680002.sHtML
tv.blog.vgdrev.cn/Article/details/460682.sHtML
tv.blog.vgdrev.cn/Article/details/666420.sHtML
tv.blog.vgdrev.cn/Article/details/242068.sHtML
tv.blog.vgdrev.cn/Article/details/464682.sHtML
tv.blog.vgdrev.cn/Article/details/424802.sHtML
tv.blog.vgdrev.cn/Article/details/335731.sHtML
tv.blog.vgdrev.cn/Article/details/735737.sHtML
tv.blog.vgdrev.cn/Article/details/608406.sHtML
tv.blog.vgdrev.cn/Article/details/193373.sHtML
tv.blog.vgdrev.cn/Article/details/280026.sHtML
tv.blog.vgdrev.cn/Article/details/806288.sHtML
tv.blog.vgdrev.cn/Article/details/911955.sHtML
tv.blog.vgdrev.cn/Article/details/446464.sHtML
tv.blog.vgdrev.cn/Article/details/931911.sHtML
tv.blog.vgdrev.cn/Article/details/206640.sHtML
tv.blog.vgdrev.cn/Article/details/686060.sHtML
tv.blog.vgdrev.cn/Article/details/884468.sHtML
tv.blog.vgdrev.cn/Article/details/717557.sHtML
tv.blog.vgdrev.cn/Article/details/282824.sHtML
tv.blog.vgdrev.cn/Article/details/737753.sHtML
tv.blog.vgdrev.cn/Article/details/402004.sHtML
tv.blog.vgdrev.cn/Article/details/517713.sHtML
tv.blog.vgdrev.cn/Article/details/791179.sHtML
tv.blog.vgdrev.cn/Article/details/597993.sHtML
tv.blog.vgdrev.cn/Article/details/175915.sHtML
tv.blog.vgdrev.cn/Article/details/355139.sHtML
tv.blog.vgdrev.cn/Article/details/597133.sHtML
tv.blog.vgdrev.cn/Article/details/317317.sHtML
tv.blog.vgdrev.cn/Article/details/393971.sHtML
tv.blog.vgdrev.cn/Article/details/737313.sHtML
tv.blog.vgdrev.cn/Article/details/557717.sHtML
tv.blog.vgdrev.cn/Article/details/559399.sHtML
tv.blog.vgdrev.cn/Article/details/777993.sHtML
tv.blog.vgdrev.cn/Article/details/575315.sHtML
tv.blog.vgdrev.cn/Article/details/575539.sHtML
tv.blog.vgdrev.cn/Article/details/153997.sHtML
tv.blog.vgdrev.cn/Article/details/373159.sHtML
tv.blog.vgdrev.cn/Article/details/759319.sHtML
tv.blog.vgdrev.cn/Article/details/313135.sHtML
tv.blog.vgdrev.cn/Article/details/733351.sHtML
tv.blog.vgdrev.cn/Article/details/711111.sHtML
tv.blog.vgdrev.cn/Article/details/139315.sHtML
tv.blog.vgdrev.cn/Article/details/757959.sHtML
tv.blog.vgdrev.cn/Article/details/731135.sHtML
tv.blog.vgdrev.cn/Article/details/339133.sHtML
tv.blog.vgdrev.cn/Article/details/377739.sHtML
tv.blog.vgdrev.cn/Article/details/393133.sHtML
tv.blog.vgdrev.cn/Article/details/957939.sHtML
tv.blog.vgdrev.cn/Article/details/139159.sHtML
tv.blog.vgdrev.cn/Article/details/557353.sHtML
tv.blog.vgdrev.cn/Article/details/371171.sHtML
tv.blog.vgdrev.cn/Article/details/777111.sHtML
tv.blog.vgdrev.cn/Article/details/553571.sHtML
tv.blog.vgdrev.cn/Article/details/975133.sHtML
tv.blog.vgdrev.cn/Article/details/397355.sHtML
tv.blog.vgdrev.cn/Article/details/533995.sHtML
tv.blog.vgdrev.cn/Article/details/137517.sHtML
tv.blog.vgdrev.cn/Article/details/139539.sHtML
tv.blog.vgdrev.cn/Article/details/155195.sHtML
tv.blog.vgdrev.cn/Article/details/773937.sHtML
tv.blog.vgdrev.cn/Article/details/573317.sHtML
tv.blog.vgdrev.cn/Article/details/337711.sHtML
tv.blog.vgdrev.cn/Article/details/971571.sHtML
tv.blog.vgdrev.cn/Article/details/224866.sHtML
tv.blog.vgdrev.cn/Article/details/951913.sHtML
tv.blog.vgdrev.cn/Article/details/991313.sHtML
tv.blog.vgdrev.cn/Article/details/957733.sHtML
tv.blog.vgdrev.cn/Article/details/886468.sHtML
tv.blog.vgdrev.cn/Article/details/335195.sHtML
tv.blog.vgdrev.cn/Article/details/175551.sHtML
tv.blog.vgdrev.cn/Article/details/539319.sHtML
tv.blog.vgdrev.cn/Article/details/373757.sHtML
tv.blog.vgdrev.cn/Article/details/917953.sHtML
tv.blog.vgdrev.cn/Article/details/082866.sHtML
tv.blog.vgdrev.cn/Article/details/640286.sHtML
tv.blog.vgdrev.cn/Article/details/826444.sHtML
tv.blog.vgdrev.cn/Article/details/684620.sHtML
tv.blog.vgdrev.cn/Article/details/048248.sHtML
tv.blog.vgdrev.cn/Article/details/513315.sHtML
tv.blog.vgdrev.cn/Article/details/591117.sHtML
tv.blog.vgdrev.cn/Article/details/531159.sHtML
tv.blog.vgdrev.cn/Article/details/955531.sHtML
tv.blog.vgdrev.cn/Article/details/280266.sHtML
tv.blog.vgdrev.cn/Article/details/088066.sHtML
tv.blog.vgdrev.cn/Article/details/719359.sHtML
tv.blog.vgdrev.cn/Article/details/800066.sHtML
tv.blog.vgdrev.cn/Article/details/226086.sHtML
tv.blog.vgdrev.cn/Article/details/426884.sHtML
tv.blog.vgdrev.cn/Article/details/206408.sHtML
tv.blog.vgdrev.cn/Article/details/957551.sHtML
tv.blog.vgdrev.cn/Article/details/246820.sHtML
tv.blog.vgdrev.cn/Article/details/773339.sHtML
tv.blog.vgdrev.cn/Article/details/484284.sHtML
tv.blog.vgdrev.cn/Article/details/088844.sHtML
tv.blog.vgdrev.cn/Article/details/022484.sHtML
tv.blog.vgdrev.cn/Article/details/268040.sHtML
tv.blog.vgdrev.cn/Article/details/662024.sHtML
tv.blog.vgdrev.cn/Article/details/286464.sHtML
tv.blog.vgdrev.cn/Article/details/202048.sHtML
tv.blog.vgdrev.cn/Article/details/206424.sHtML
tv.blog.vgdrev.cn/Article/details/000804.sHtML
tv.blog.vgdrev.cn/Article/details/020866.sHtML
tv.blog.vgdrev.cn/Article/details/406268.sHtML
tv.blog.vgdrev.cn/Article/details/288682.sHtML
tv.blog.vgdrev.cn/Article/details/628482.sHtML
tv.blog.vgdrev.cn/Article/details/959337.sHtML
tv.blog.vgdrev.cn/Article/details/024806.sHtML
tv.blog.vgdrev.cn/Article/details/286248.sHtML
tv.blog.vgdrev.cn/Article/details/533155.sHtML
tv.blog.vgdrev.cn/Article/details/228020.sHtML
tv.blog.vgdrev.cn/Article/details/882800.sHtML
tv.blog.vgdrev.cn/Article/details/042686.sHtML
tv.blog.vgdrev.cn/Article/details/860206.sHtML
tv.blog.vgdrev.cn/Article/details/600446.sHtML
tv.blog.vgdrev.cn/Article/details/822284.sHtML
tv.blog.vgdrev.cn/Article/details/664808.sHtML
tv.blog.vgdrev.cn/Article/details/979539.sHtML
tv.blog.vgdrev.cn/Article/details/428628.sHtML
tv.blog.vgdrev.cn/Article/details/664288.sHtML
tv.blog.vgdrev.cn/Article/details/426020.sHtML
tv.blog.vgdrev.cn/Article/details/262640.sHtML
tv.blog.vgdrev.cn/Article/details/642020.sHtML
tv.blog.vgdrev.cn/Article/details/020062.sHtML
tv.blog.vgdrev.cn/Article/details/999513.sHtML
tv.blog.vgdrev.cn/Article/details/442400.sHtML
tv.blog.vgdrev.cn/Article/details/628680.sHtML
tv.blog.vgdrev.cn/Article/details/759539.sHtML
tv.blog.vgdrev.cn/Article/details/953957.sHtML
tv.blog.vgdrev.cn/Article/details/595933.sHtML
tv.blog.vgdrev.cn/Article/details/359733.sHtML
tv.blog.vgdrev.cn/Article/details/880086.sHtML
tv.blog.vgdrev.cn/Article/details/771997.sHtML
tv.blog.vgdrev.cn/Article/details/468402.sHtML
tv.blog.vgdrev.cn/Article/details/406804.sHtML
tv.blog.vgdrev.cn/Article/details/068604.sHtML
tv.blog.vgdrev.cn/Article/details/620088.sHtML
tv.blog.vgdrev.cn/Article/details/379511.sHtML
tv.blog.vgdrev.cn/Article/details/397191.sHtML
tv.blog.vgdrev.cn/Article/details/464024.sHtML
tv.blog.vgdrev.cn/Article/details/264660.sHtML
tv.blog.vgdrev.cn/Article/details/400240.sHtML
tv.blog.vgdrev.cn/Article/details/402460.sHtML
tv.blog.vgdrev.cn/Article/details/680802.sHtML
tv.blog.vgdrev.cn/Article/details/820480.sHtML
tv.blog.vgdrev.cn/Article/details/971535.sHtML
tv.blog.vgdrev.cn/Article/details/284466.sHtML
tv.blog.vgdrev.cn/Article/details/159593.sHtML
tv.blog.vgdrev.cn/Article/details/371371.sHtML
tv.blog.vgdrev.cn/Article/details/973757.sHtML
tv.blog.vgdrev.cn/Article/details/359315.sHtML
tv.blog.vgdrev.cn/Article/details/979153.sHtML
tv.blog.vgdrev.cn/Article/details/331179.sHtML
tv.blog.vgdrev.cn/Article/details/268646.sHtML
tv.blog.vgdrev.cn/Article/details/971957.sHtML
tv.blog.vgdrev.cn/Article/details/199933.sHtML
tv.blog.vgdrev.cn/Article/details/595773.sHtML
tv.blog.vgdrev.cn/Article/details/800048.sHtML
tv.blog.vgdrev.cn/Article/details/468428.sHtML
tv.blog.vgdrev.cn/Article/details/153391.sHtML
tv.blog.vgdrev.cn/Article/details/111739.sHtML
tv.blog.vgdrev.cn/Article/details/286266.sHtML
tv.blog.vgdrev.cn/Article/details/315935.sHtML
tv.blog.vgdrev.cn/Article/details/115975.sHtML
tv.blog.vgdrev.cn/Article/details/153117.sHtML
tv.blog.vgdrev.cn/Article/details/557919.sHtML
tv.blog.vgdrev.cn/Article/details/280206.sHtML
tv.blog.vgdrev.cn/Article/details/864826.sHtML
tv.blog.vgdrev.cn/Article/details/404220.sHtML
tv.blog.vgdrev.cn/Article/details/684846.sHtML
tv.blog.vgdrev.cn/Article/details/731759.sHtML
tv.blog.vgdrev.cn/Article/details/957517.sHtML
tv.blog.vgdrev.cn/Article/details/911917.sHtML
tv.blog.vgdrev.cn/Article/details/535351.sHtML
tv.blog.vgdrev.cn/Article/details/711711.sHtML
tv.blog.vgdrev.cn/Article/details/793157.sHtML
tv.blog.vgdrev.cn/Article/details/759313.sHtML
tv.blog.vgdrev.cn/Article/details/751793.sHtML
tv.blog.vgdrev.cn/Article/details/575959.sHtML
tv.blog.vgdrev.cn/Article/details/353117.sHtML
tv.blog.vgdrev.cn/Article/details/793159.sHtML
tv.blog.vgdrev.cn/Article/details/135397.sHtML
tv.blog.vgdrev.cn/Article/details/513913.sHtML
tv.blog.vgdrev.cn/Article/details/571155.sHtML
tv.blog.vgdrev.cn/Article/details/519997.sHtML
tv.blog.vgdrev.cn/Article/details/799759.sHtML
tv.blog.vgdrev.cn/Article/details/339713.sHtML
tv.blog.vgdrev.cn/Article/details/971119.sHtML
tv.blog.vgdrev.cn/Article/details/311735.sHtML
tv.blog.vgdrev.cn/Article/details/515197.sHtML
tv.blog.vgdrev.cn/Article/details/335571.sHtML
tv.blog.vgdrev.cn/Article/details/919359.sHtML
tv.blog.vgdrev.cn/Article/details/933371.sHtML
tv.blog.vgdrev.cn/Article/details/931597.sHtML
tv.blog.vgdrev.cn/Article/details/353319.sHtML
tv.blog.vgdrev.cn/Article/details/715577.sHtML
tv.blog.vgdrev.cn/Article/details/951533.sHtML
tv.blog.vgdrev.cn/Article/details/397173.sHtML
tv.blog.vgdrev.cn/Article/details/337335.sHtML
tv.blog.vgdrev.cn/Article/details/799977.sHtML
tv.blog.vgdrev.cn/Article/details/115915.sHtML
tv.blog.vgdrev.cn/Article/details/355973.sHtML
tv.blog.vgdrev.cn/Article/details/757137.sHtML
tv.blog.vgdrev.cn/Article/details/357759.sHtML
tv.blog.vgdrev.cn/Article/details/333151.sHtML
tv.blog.vgdrev.cn/Article/details/115139.sHtML
tv.blog.vgdrev.cn/Article/details/939353.sHtML
tv.blog.vgdrev.cn/Article/details/797771.sHtML
tv.blog.vgdrev.cn/Article/details/804804.sHtML
tv.blog.vgdrev.cn/Article/details/648442.sHtML
tv.blog.vgdrev.cn/Article/details/022604.sHtML
tv.blog.vgdrev.cn/Article/details/442468.sHtML
tv.blog.vgdrev.cn/Article/details/228626.sHtML
tv.blog.vgdrev.cn/Article/details/624468.sHtML
tv.blog.vgdrev.cn/Article/details/373711.sHtML
tv.blog.vgdrev.cn/Article/details/179799.sHtML
tv.blog.vgdrev.cn/Article/details/668842.sHtML
tv.blog.vgdrev.cn/Article/details/448048.sHtML
tv.blog.vgdrev.cn/Article/details/575773.sHtML
tv.blog.vgdrev.cn/Article/details/331351.sHtML
tv.blog.vgdrev.cn/Article/details/711979.sHtML
tv.blog.vgdrev.cn/Article/details/537719.sHtML
tv.blog.vgdrev.cn/Article/details/353913.sHtML
tv.blog.vgdrev.cn/Article/details/466442.sHtML
tv.blog.vgdrev.cn/Article/details/086282.sHtML
tv.blog.vgdrev.cn/Article/details/482806.sHtML
tv.blog.vgdrev.cn/Article/details/024842.sHtML
tv.blog.vgdrev.cn/Article/details/226228.sHtML
tv.blog.vgdrev.cn/Article/details/228666.sHtML
tv.blog.vgdrev.cn/Article/details/442040.sHtML
tv.blog.vgdrev.cn/Article/details/408686.sHtML
tv.blog.vgdrev.cn/Article/details/866666.sHtML
tv.blog.vgdrev.cn/Article/details/200200.sHtML
tv.blog.vgdrev.cn/Article/details/404242.sHtML
tv.blog.vgdrev.cn/Article/details/242866.sHtML
tv.blog.vgdrev.cn/Article/details/395575.sHtML
tv.blog.vgdrev.cn/Article/details/442000.sHtML
tv.blog.vgdrev.cn/Article/details/404086.sHtML
tv.blog.vgdrev.cn/Article/details/208044.sHtML
tv.blog.vgdrev.cn/Article/details/244440.sHtML
tv.blog.vgdrev.cn/Article/details/824080.sHtML
tv.blog.vgdrev.cn/Article/details/268244.sHtML
tv.blog.vgdrev.cn/Article/details/242480.sHtML
tv.blog.vgdrev.cn/Article/details/200262.sHtML
tv.blog.vgdrev.cn/Article/details/468424.sHtML
tv.blog.vgdrev.cn/Article/details/664440.sHtML
tv.blog.vgdrev.cn/Article/details/937357.sHtML
tv.blog.vgdrev.cn/Article/details/804088.sHtML
tv.blog.vgdrev.cn/Article/details/646002.sHtML
tv.blog.vgdrev.cn/Article/details/280088.sHtML
tv.blog.vgdrev.cn/Article/details/771933.sHtML
tv.blog.vgdrev.cn/Article/details/171173.sHtML
tv.blog.vgdrev.cn/Article/details/200442.sHtML
tv.blog.vgdrev.cn/Article/details/040022.sHtML
tv.blog.vgdrev.cn/Article/details/579139.sHtML
tv.blog.vgdrev.cn/Article/details/804268.sHtML
tv.blog.vgdrev.cn/Article/details/442888.sHtML
tv.blog.vgdrev.cn/Article/details/737599.sHtML
tv.blog.vgdrev.cn/Article/details/866248.sHtML
tv.blog.vgdrev.cn/Article/details/046864.sHtML
tv.blog.vgdrev.cn/Article/details/202624.sHtML
tv.blog.vgdrev.cn/Article/details/644402.sHtML
tv.blog.vgdrev.cn/Article/details/888806.sHtML
tv.blog.vgdrev.cn/Article/details/088226.sHtML
tv.blog.vgdrev.cn/Article/details/935755.sHtML
tv.blog.vgdrev.cn/Article/details/399511.sHtML
tv.blog.vgdrev.cn/Article/details/157997.sHtML
tv.blog.vgdrev.cn/Article/details/846668.sHtML
tv.blog.vgdrev.cn/Article/details/486622.sHtML
tv.blog.vgdrev.cn/Article/details/313577.sHtML
tv.blog.vgdrev.cn/Article/details/824668.sHtML
tv.blog.vgdrev.cn/Article/details/311999.sHtML
tv.blog.vgdrev.cn/Article/details/882008.sHtML
tv.blog.vgdrev.cn/Article/details/664862.sHtML
tv.blog.vgdrev.cn/Article/details/648666.sHtML
tv.blog.vgdrev.cn/Article/details/559377.sHtML
tv.blog.vgdrev.cn/Article/details/662088.sHtML
tv.blog.vgdrev.cn/Article/details/957515.sHtML
tv.blog.vgdrev.cn/Article/details/555171.sHtML
tv.blog.vgdrev.cn/Article/details/682486.sHtML
tv.blog.vgdrev.cn/Article/details/933137.sHtML
tv.blog.vgdrev.cn/Article/details/535911.sHtML
tv.blog.vgdrev.cn/Article/details/199195.sHtML
tv.blog.vgdrev.cn/Article/details/860408.sHtML
tv.blog.vgdrev.cn/Article/details/373333.sHtML
tv.blog.vgdrev.cn/Article/details/226482.sHtML
tv.blog.vgdrev.cn/Article/details/686028.sHtML
tv.blog.vgdrev.cn/Article/details/593971.sHtML
tv.blog.vgdrev.cn/Article/details/048624.sHtML
tv.blog.vgdrev.cn/Article/details/228624.sHtML
tv.blog.vgdrev.cn/Article/details/280268.sHtML
tv.blog.vgdrev.cn/Article/details/573355.sHtML
tv.blog.vgdrev.cn/Article/details/840226.sHtML
tv.blog.vgdrev.cn/Article/details/777759.sHtML
tv.blog.vgdrev.cn/Article/details/286888.sHtML
tv.blog.vgdrev.cn/Article/details/640606.sHtML
tv.blog.vgdrev.cn/Article/details/957531.sHtML
tv.blog.vgdrev.cn/Article/details/395395.sHtML
tv.blog.vgdrev.cn/Article/details/397377.sHtML
tv.blog.vgdrev.cn/Article/details/717739.sHtML
tv.blog.vgdrev.cn/Article/details/020666.sHtML
tv.blog.vgdrev.cn/Article/details/444628.sHtML
tv.blog.vgdrev.cn/Article/details/260482.sHtML
tv.blog.vgdrev.cn/Article/details/240422.sHtML
tv.blog.vgdrev.cn/Article/details/008202.sHtML
tv.blog.vgdrev.cn/Article/details/195773.sHtML
tv.blog.vgdrev.cn/Article/details/000686.sHtML
tv.blog.vgdrev.cn/Article/details/020248.sHtML
tv.blog.vgdrev.cn/Article/details/751337.sHtML
tv.blog.vgdrev.cn/Article/details/177577.sHtML
tv.blog.vgdrev.cn/Article/details/484400.sHtML
tv.blog.vgdrev.cn/Article/details/680062.sHtML
tv.blog.vgdrev.cn/Article/details/462666.sHtML
tv.blog.vgdrev.cn/Article/details/888408.sHtML
tv.blog.vgdrev.cn/Article/details/208080.sHtML
tv.blog.vgdrev.cn/Article/details/262420.sHtML
tv.blog.vgdrev.cn/Article/details/064608.sHtML
tv.blog.vgdrev.cn/Article/details/060222.sHtML
tv.blog.vgdrev.cn/Article/details/317999.sHtML
tv.blog.vgdrev.cn/Article/details/937599.sHtML
tv.blog.vgdrev.cn/Article/details/020468.sHtML
tv.blog.vgdrev.cn/Article/details/620048.sHtML
tv.blog.vgdrev.cn/Article/details/802862.sHtML
tv.blog.vgdrev.cn/Article/details/608484.sHtML
tv.blog.vgdrev.cn/Article/details/733733.sHtML
tv.blog.vgdrev.cn/Article/details/808022.sHtML
tv.blog.vgdrev.cn/Article/details/739379.sHtML
tv.blog.vgdrev.cn/Article/details/844064.sHtML
tv.blog.vgdrev.cn/Article/details/111719.sHtML
tv.blog.vgdrev.cn/Article/details/139191.sHtML
tv.blog.vgdrev.cn/Article/details/975937.sHtML
tv.blog.vgdrev.cn/Article/details/200424.sHtML
tv.blog.vgdrev.cn/Article/details/771359.sHtML
tv.blog.vgdrev.cn/Article/details/668204.sHtML
tv.blog.vgdrev.cn/Article/details/808088.sHtML
tv.blog.vgdrev.cn/Article/details/264248.sHtML
tv.blog.vgdrev.cn/Article/details/282004.sHtML
tv.blog.vgdrev.cn/Article/details/406202.sHtML
tv.blog.vgdrev.cn/Article/details/820624.sHtML
tv.blog.vgdrev.cn/Article/details/000606.sHtML
tv.blog.vgdrev.cn/Article/details/604882.sHtML
tv.blog.vgdrev.cn/Article/details/480242.sHtML
tv.blog.vgdrev.cn/Article/details/040846.sHtML
tv.blog.vgdrev.cn/Article/details/026604.sHtML
tv.blog.vgdrev.cn/Article/details/480448.sHtML
tv.blog.vgdrev.cn/Article/details/620486.sHtML
tv.blog.vgdrev.cn/Article/details/604006.sHtML
tv.blog.vgdrev.cn/Article/details/646020.sHtML
tv.blog.vgdrev.cn/Article/details/222606.sHtML
tv.blog.vgdrev.cn/Article/details/202846.sHtML
tv.blog.vgdrev.cn/Article/details/513591.sHtML
tv.blog.vgdrev.cn/Article/details/082688.sHtML
tv.blog.vgdrev.cn/Article/details/979517.sHtML
tv.blog.vgdrev.cn/Article/details/684002.sHtML
tv.blog.vgdrev.cn/Article/details/640426.sHtML
tv.blog.vgdrev.cn/Article/details/208480.sHtML
tv.blog.vgdrev.cn/Article/details/822666.sHtML
tv.blog.vgdrev.cn/Article/details/426488.sHtML
tv.blog.vgdrev.cn/Article/details/642042.sHtML
tv.blog.vgdrev.cn/Article/details/553935.sHtML
tv.blog.vgdrev.cn/Article/details/220064.sHtML
tv.blog.vgdrev.cn/Article/details/795353.sHtML
tv.blog.vgdrev.cn/Article/details/353557.sHtML
tv.blog.vgdrev.cn/Article/details/757775.sHtML
tv.blog.vgdrev.cn/Article/details/933975.sHtML
tv.blog.vgdrev.cn/Article/details/175573.sHtML
tv.blog.vgdrev.cn/Article/details/175559.sHtML
tv.blog.vgdrev.cn/Article/details/282688.sHtML
tv.blog.vgdrev.cn/Article/details/028688.sHtML
tv.blog.vgdrev.cn/Article/details/113199.sHtML
tv.blog.vgdrev.cn/Article/details/395577.sHtML
tv.blog.vgdrev.cn/Article/details/159353.sHtML
tv.blog.vgdrev.cn/Article/details/937375.sHtML
tv.blog.vgdrev.cn/Article/details/200644.sHtML
tv.blog.vgdrev.cn/Article/details/286228.sHtML
tv.blog.vgdrev.cn/Article/details/668422.sHtML
tv.blog.vgdrev.cn/Article/details/640464.sHtML
tv.blog.vgdrev.cn/Article/details/379199.sHtML
