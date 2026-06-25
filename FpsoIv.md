
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

qyr.guohua888.cn/00062.Doc
qyr.guohua888.cn/28264.Doc
qyr.guohua888.cn/82804.Doc
qyr.guohua888.cn/71317.Doc
qyr.guohua888.cn/15577.Doc
qyr.guohua888.cn/66248.Doc
qyr.guohua888.cn/51199.Doc
qyr.guohua888.cn/82246.Doc
qyr.guohua888.cn/22062.Doc
qyr.guohua888.cn/44284.Doc
qye.guohua888.cn/20066.Doc
qye.guohua888.cn/48808.Doc
qye.guohua888.cn/28486.Doc
qye.guohua888.cn/04008.Doc
qye.guohua888.cn/04824.Doc
qye.guohua888.cn/64882.Doc
qye.guohua888.cn/80480.Doc
qye.guohua888.cn/00688.Doc
qye.guohua888.cn/64224.Doc
qye.guohua888.cn/00648.Doc
qyw.guohua888.cn/22826.Doc
qyw.guohua888.cn/64200.Doc
qyw.guohua888.cn/84626.Doc
qyw.guohua888.cn/86262.Doc
qyw.guohua888.cn/28086.Doc
qyw.guohua888.cn/48464.Doc
qyw.guohua888.cn/88680.Doc
qyw.guohua888.cn/08022.Doc
qyw.guohua888.cn/48006.Doc
qyw.guohua888.cn/42400.Doc
qyq.guohua888.cn/64408.Doc
qyq.guohua888.cn/00468.Doc
qyq.guohua888.cn/82026.Doc
qyq.guohua888.cn/82244.Doc
qyq.guohua888.cn/68220.Doc
qyq.guohua888.cn/84020.Doc
qyq.guohua888.cn/20028.Doc
qyq.guohua888.cn/48404.Doc
qyq.guohua888.cn/64442.Doc
qyq.guohua888.cn/42040.Doc
qtm.guohua888.cn/46644.Doc
qtm.guohua888.cn/22022.Doc
qtm.guohua888.cn/04220.Doc
qtm.guohua888.cn/06448.Doc
qtm.guohua888.cn/82808.Doc
qtm.guohua888.cn/42266.Doc
qtm.guohua888.cn/00466.Doc
qtm.guohua888.cn/64000.Doc
qtm.guohua888.cn/57791.Doc
qtm.guohua888.cn/40248.Doc
qtn.guohua888.cn/40802.Doc
qtn.guohua888.cn/11577.Doc
qtn.guohua888.cn/40406.Doc
qtn.guohua888.cn/53559.Doc
qtn.guohua888.cn/55991.Doc
qtn.guohua888.cn/48488.Doc
qtn.guohua888.cn/60424.Doc
qtn.guohua888.cn/06082.Doc
qtn.guohua888.cn/51313.Doc
qtn.guohua888.cn/73955.Doc
qtb.guohua888.cn/48086.Doc
qtb.guohua888.cn/77117.Doc
qtb.guohua888.cn/17919.Doc
qtb.guohua888.cn/26044.Doc
qtb.guohua888.cn/11135.Doc
qtb.guohua888.cn/08848.Doc
qtb.guohua888.cn/57511.Doc
qtb.guohua888.cn/44006.Doc
qtb.guohua888.cn/55551.Doc
qtb.guohua888.cn/60082.Doc
qtv.guohua888.cn/22840.Doc
qtv.guohua888.cn/26248.Doc
qtv.guohua888.cn/62820.Doc
qtv.guohua888.cn/62686.Doc
qtv.guohua888.cn/04242.Doc
qtv.guohua888.cn/24000.Doc
qtv.guohua888.cn/24064.Doc
qtv.guohua888.cn/20022.Doc
qtv.guohua888.cn/42862.Doc
qtv.guohua888.cn/22040.Doc
qtc.guohua888.cn/02008.Doc
qtc.guohua888.cn/71915.Doc
qtc.guohua888.cn/20844.Doc
qtc.guohua888.cn/44006.Doc
qtc.guohua888.cn/59911.Doc
qtc.guohua888.cn/39755.Doc
qtc.guohua888.cn/48826.Doc
qtc.guohua888.cn/55313.Doc
qtc.guohua888.cn/77955.Doc
qtc.guohua888.cn/95995.Doc
qtx.guohua888.cn/73311.Doc
qtx.guohua888.cn/79739.Doc
qtx.guohua888.cn/59591.Doc
qtx.guohua888.cn/19759.Doc
qtx.guohua888.cn/59135.Doc
qtx.guohua888.cn/39731.Doc
qtx.guohua888.cn/33515.Doc
qtx.guohua888.cn/00080.Doc
qtx.guohua888.cn/95991.Doc
qtx.guohua888.cn/75999.Doc
qtz.guohua888.cn/68888.Doc
qtz.guohua888.cn/17139.Doc
qtz.guohua888.cn/97995.Doc
qtz.guohua888.cn/91557.Doc
qtz.guohua888.cn/33397.Doc
qtz.guohua888.cn/75113.Doc
qtz.guohua888.cn/15733.Doc
qtz.guohua888.cn/91735.Doc
qtz.guohua888.cn/24888.Doc
qtz.guohua888.cn/62460.Doc
qtl.guohua888.cn/53319.Doc
qtl.guohua888.cn/97313.Doc
qtl.guohua888.cn/11573.Doc
qtl.guohua888.cn/93379.Doc
qtl.guohua888.cn/53531.Doc
qtl.guohua888.cn/33751.Doc
qtl.guohua888.cn/53317.Doc
qtl.guohua888.cn/93971.Doc
qtl.guohua888.cn/19591.Doc
qtl.guohua888.cn/94480.Doc
qtk.guohua888.cn/55157.Doc
qtk.guohua888.cn/73331.Doc
qtk.guohua888.cn/53797.Doc
qtk.guohua888.cn/02020.Doc
qtk.guohua888.cn/55917.Doc
qtk.guohua888.cn/26406.Doc
qtk.guohua888.cn/13157.Doc
qtk.guohua888.cn/82662.Doc
qtk.guohua888.cn/46624.Doc
qtk.guohua888.cn/02844.Doc
qtj.guohua888.cn/64800.Doc
qtj.guohua888.cn/91117.Doc
qtj.guohua888.cn/64280.Doc
qtj.guohua888.cn/53791.Doc
qtj.guohua888.cn/73357.Doc
qtj.guohua888.cn/97159.Doc
qtj.guohua888.cn/84424.Doc
qtj.guohua888.cn/02062.Doc
qtj.guohua888.cn/51751.Doc
qtj.guohua888.cn/84226.Doc
qth.guohua888.cn/60840.Doc
qth.guohua888.cn/48824.Doc
qth.guohua888.cn/00604.Doc
qth.guohua888.cn/00084.Doc
qth.guohua888.cn/24480.Doc
qth.guohua888.cn/26484.Doc
qth.guohua888.cn/84866.Doc
qth.guohua888.cn/82080.Doc
qth.guohua888.cn/68640.Doc
qth.guohua888.cn/17797.Doc
qtg.guohua888.cn/42080.Doc
qtg.guohua888.cn/08640.Doc
qtg.guohua888.cn/22246.Doc
qtg.guohua888.cn/20464.Doc
qtg.guohua888.cn/00626.Doc
qtg.guohua888.cn/20464.Doc
qtg.guohua888.cn/04422.Doc
qtg.guohua888.cn/66428.Doc
qtg.guohua888.cn/64864.Doc
qtg.guohua888.cn/68664.Doc
qtf.guohua888.cn/60840.Doc
qtf.guohua888.cn/44444.Doc
qtf.guohua888.cn/02420.Doc
qtf.guohua888.cn/62428.Doc
qtf.guohua888.cn/64088.Doc
qtf.guohua888.cn/28806.Doc
qtf.guohua888.cn/84662.Doc
qtf.guohua888.cn/42428.Doc
qtf.guohua888.cn/00008.Doc
qtf.guohua888.cn/46446.Doc
qtd.guohua888.cn/28428.Doc
qtd.guohua888.cn/82822.Doc
qtd.guohua888.cn/46020.Doc
qtd.guohua888.cn/20426.Doc
qtd.guohua888.cn/28460.Doc
qtd.guohua888.cn/44466.Doc
qtd.guohua888.cn/46868.Doc
qtd.guohua888.cn/80006.Doc
qtd.guohua888.cn/42460.Doc
qtd.guohua888.cn/40620.Doc
qts.guohua888.cn/40422.Doc
qts.guohua888.cn/46426.Doc
qts.guohua888.cn/24004.Doc
qts.guohua888.cn/00008.Doc
qts.guohua888.cn/68264.Doc
qts.guohua888.cn/60606.Doc
qts.guohua888.cn/42228.Doc
qts.guohua888.cn/77660.Doc
qts.guohua888.cn/00286.Doc
qts.guohua888.cn/28202.Doc
qta.guohua888.cn/62046.Doc
qta.guohua888.cn/46428.Doc
qta.guohua888.cn/02206.Doc
qta.guohua888.cn/46044.Doc
qta.guohua888.cn/48042.Doc
qta.guohua888.cn/40062.Doc
qta.guohua888.cn/22622.Doc
qta.guohua888.cn/62240.Doc
qta.guohua888.cn/06260.Doc
qta.guohua888.cn/84606.Doc
qtp.guohua888.cn/06448.Doc
qtp.guohua888.cn/08028.Doc
qtp.guohua888.cn/40004.Doc
qtp.guohua888.cn/00844.Doc
qtp.guohua888.cn/20280.Doc
qtp.guohua888.cn/64420.Doc
qtp.guohua888.cn/04060.Doc
qtp.guohua888.cn/66684.Doc
qtp.guohua888.cn/46040.Doc
qtp.guohua888.cn/06868.Doc
qto.guohua888.cn/26406.Doc
qto.guohua888.cn/48680.Doc
qto.guohua888.cn/88602.Doc
qto.guohua888.cn/99339.Doc
qto.guohua888.cn/08286.Doc
qto.guohua888.cn/44866.Doc
qto.guohua888.cn/82648.Doc
qto.guohua888.cn/64606.Doc
qto.guohua888.cn/28282.Doc
qto.guohua888.cn/95719.Doc
qti.guohua888.cn/22880.Doc
qti.guohua888.cn/60462.Doc
qti.guohua888.cn/20084.Doc
qti.guohua888.cn/28886.Doc
qti.guohua888.cn/48604.Doc
qti.guohua888.cn/80008.Doc
qti.guohua888.cn/60440.Doc
qti.guohua888.cn/42846.Doc
qti.guohua888.cn/60008.Doc
qti.guohua888.cn/62402.Doc
qtu.guohua888.cn/88006.Doc
qtu.guohua888.cn/22022.Doc
qtu.guohua888.cn/26862.Doc
qtu.guohua888.cn/88220.Doc
qtu.guohua888.cn/88486.Doc
qtu.guohua888.cn/60226.Doc
qtu.guohua888.cn/46898.Doc
qtu.guohua888.cn/15258.Doc
qtu.guohua888.cn/71430.Doc
qtu.guohua888.cn/18638.Doc
qty.guohua888.cn/37513.Doc
qty.guohua888.cn/91377.Doc
qty.guohua888.cn/60809.Doc
qty.guohua888.cn/23962.Doc
qty.guohua888.cn/50838.Doc
qty.guohua888.cn/30246.Doc
qty.guohua888.cn/48935.Doc
qty.guohua888.cn/20115.Doc
qty.guohua888.cn/76865.Doc
qty.guohua888.cn/77718.Doc
qtt.guohua888.cn/54731.Doc
qtt.guohua888.cn/32820.Doc
qtt.guohua888.cn/33368.Doc
qtt.guohua888.cn/16505.Doc
qtt.guohua888.cn/16062.Doc
qtt.guohua888.cn/03602.Doc
qtt.guohua888.cn/29816.Doc
qtt.guohua888.cn/14113.Doc
qtt.guohua888.cn/44778.Doc
qtt.guohua888.cn/85468.Doc
qtr.guohua888.cn/95850.Doc
qtr.guohua888.cn/77543.Doc
qtr.guohua888.cn/35979.Doc
qtr.guohua888.cn/73166.Doc
qtr.guohua888.cn/56564.Doc
qtr.guohua888.cn/26288.Doc
qtr.guohua888.cn/60862.Doc
qtr.guohua888.cn/82682.Doc
qtr.guohua888.cn/04684.Doc
qtr.guohua888.cn/64420.Doc
qte.guohua888.cn/60200.Doc
qte.guohua888.cn/84404.Doc
qte.guohua888.cn/42628.Doc
qte.guohua888.cn/68828.Doc
qte.guohua888.cn/86202.Doc
qte.guohua888.cn/44082.Doc
qte.guohua888.cn/80264.Doc
qte.guohua888.cn/99779.Doc
qte.guohua888.cn/84402.Doc
qte.guohua888.cn/04260.Doc
qtw.guohua888.cn/28480.Doc
qtw.guohua888.cn/88046.Doc
qtw.guohua888.cn/04444.Doc
qtw.guohua888.cn/20288.Doc
qtw.guohua888.cn/64040.Doc
qtw.guohua888.cn/04666.Doc
qtw.guohua888.cn/06842.Doc
qtw.guohua888.cn/44042.Doc
qtw.guohua888.cn/26608.Doc
qtw.guohua888.cn/06280.Doc
qtq.guohua888.cn/46208.Doc
qtq.guohua888.cn/37333.Doc
qtq.guohua888.cn/53553.Doc
qtq.guohua888.cn/62200.Doc
qtq.guohua888.cn/84606.Doc
qtq.guohua888.cn/26282.Doc
qtq.guohua888.cn/20208.Doc
qtq.guohua888.cn/64864.Doc
qtq.guohua888.cn/80862.Doc
qtq.guohua888.cn/48000.Doc
qrm.guohua888.cn/46484.Doc
qrm.guohua888.cn/99959.Doc
qrm.guohua888.cn/04222.Doc
qrm.guohua888.cn/62006.Doc
qrm.guohua888.cn/80086.Doc
qrm.guohua888.cn/13331.Doc
qrm.guohua888.cn/44460.Doc
qrm.guohua888.cn/06488.Doc
qrm.guohua888.cn/60424.Doc
qrm.guohua888.cn/44084.Doc
qrn.guohua888.cn/73757.Doc
qrn.guohua888.cn/22480.Doc
qrn.guohua888.cn/22802.Doc
qrn.guohua888.cn/40244.Doc
qrn.guohua888.cn/22600.Doc
qrn.guohua888.cn/33773.Doc
qrn.guohua888.cn/44082.Doc
qrn.guohua888.cn/22864.Doc
qrn.guohua888.cn/86424.Doc
qrn.guohua888.cn/24248.Doc
qrb.guohua888.cn/42606.Doc
qrb.guohua888.cn/20464.Doc
qrb.guohua888.cn/26022.Doc
qrb.guohua888.cn/20840.Doc
qrb.guohua888.cn/80262.Doc
qrb.guohua888.cn/42228.Doc
qrb.guohua888.cn/02088.Doc
qrb.guohua888.cn/04840.Doc
qrb.guohua888.cn/84046.Doc
qrb.guohua888.cn/46006.Doc
qrv.guohua888.cn/08620.Doc
qrv.guohua888.cn/26086.Doc
qrv.guohua888.cn/88064.Doc
qrv.guohua888.cn/24066.Doc
qrv.guohua888.cn/04660.Doc
qrv.guohua888.cn/86008.Doc
qrv.guohua888.cn/66282.Doc
qrv.guohua888.cn/04282.Doc
qrv.guohua888.cn/28062.Doc
qrv.guohua888.cn/59333.Doc
qrc.guohua888.cn/66600.Doc
qrc.guohua888.cn/13397.Doc
qrc.guohua888.cn/08800.Doc
qrc.guohua888.cn/06660.Doc
qrc.guohua888.cn/04044.Doc
qrc.guohua888.cn/88808.Doc
qrc.guohua888.cn/57939.Doc
qrc.guohua888.cn/88408.Doc
qrc.guohua888.cn/64646.Doc
qrc.guohua888.cn/44228.Doc
qrx.guohua888.cn/42206.Doc
qrx.guohua888.cn/84888.Doc
qrx.guohua888.cn/62600.Doc
qrx.guohua888.cn/84286.Doc
qrx.guohua888.cn/22202.Doc
qrx.guohua888.cn/40648.Doc
qrx.guohua888.cn/40286.Doc
qrx.guohua888.cn/42406.Doc
qrx.guohua888.cn/28620.Doc
qrx.guohua888.cn/62220.Doc
qrz.guohua888.cn/26682.Doc
qrz.guohua888.cn/88662.Doc
qrz.guohua888.cn/08222.Doc
qrz.guohua888.cn/06642.Doc
qrz.guohua888.cn/20686.Doc
qrz.guohua888.cn/66868.Doc
qrz.guohua888.cn/04466.Doc
qrz.guohua888.cn/99973.Doc
qrz.guohua888.cn/40482.Doc
qrz.guohua888.cn/26664.Doc
qrl.guohua888.cn/44844.Doc
qrl.guohua888.cn/80824.Doc
qrl.guohua888.cn/26242.Doc
qrl.guohua888.cn/22822.Doc
qrl.guohua888.cn/68026.Doc
qrl.guohua888.cn/24826.Doc
qrl.guohua888.cn/88846.Doc
qrl.guohua888.cn/84440.Doc
qrl.guohua888.cn/80068.Doc
qrl.guohua888.cn/22026.Doc
qrk.guohua888.cn/86828.Doc
qrk.guohua888.cn/82288.Doc
qrk.guohua888.cn/22460.Doc
qrk.guohua888.cn/80084.Doc
qrk.guohua888.cn/26244.Doc
qrk.guohua888.cn/28604.Doc
qrk.guohua888.cn/08802.Doc
qrk.guohua888.cn/20266.Doc
qrk.guohua888.cn/80440.Doc
qrk.guohua888.cn/80246.Doc
qrj.guohua888.cn/26864.Doc
qrj.guohua888.cn/06606.Doc
qrj.guohua888.cn/44042.Doc
qrj.guohua888.cn/60680.Doc
qrj.guohua888.cn/60442.Doc
qrj.guohua888.cn/39131.Doc
qrj.guohua888.cn/40000.Doc
qrj.guohua888.cn/08220.Doc
qrj.guohua888.cn/66844.Doc
qrj.guohua888.cn/80480.Doc
qrh.guohua888.cn/26486.Doc
qrh.guohua888.cn/42884.Doc
qrh.guohua888.cn/80824.Doc
qrh.guohua888.cn/88604.Doc
qrh.guohua888.cn/86804.Doc
qrh.guohua888.cn/20820.Doc
qrh.guohua888.cn/40824.Doc
qrh.guohua888.cn/22424.Doc
qrh.guohua888.cn/82842.Doc
qrh.guohua888.cn/06244.Doc
qrg.guohua888.cn/04002.Doc
qrg.guohua888.cn/20602.Doc
qrg.guohua888.cn/68424.Doc
qrg.guohua888.cn/24240.Doc
qrg.guohua888.cn/86402.Doc
qrg.guohua888.cn/60004.Doc
qrg.guohua888.cn/22408.Doc
qrg.guohua888.cn/20200.Doc
qrg.guohua888.cn/15755.Doc
qrg.guohua888.cn/64608.Doc
qrf.guohua888.cn/64242.Doc
qrf.guohua888.cn/84440.Doc
qrf.guohua888.cn/84222.Doc
qrf.guohua888.cn/04262.Doc
qrf.guohua888.cn/86062.Doc
qrf.guohua888.cn/88484.Doc
qrf.guohua888.cn/86486.Doc
qrf.guohua888.cn/86028.Doc
qrf.guohua888.cn/24600.Doc
qrf.guohua888.cn/75597.Doc
qrd.guohua888.cn/40604.Doc
qrd.guohua888.cn/22402.Doc
qrd.guohua888.cn/82220.Doc
qrd.guohua888.cn/44004.Doc
qrd.guohua888.cn/66248.Doc
qrd.guohua888.cn/44462.Doc
qrd.guohua888.cn/88042.Doc
qrd.guohua888.cn/60648.Doc
qrd.guohua888.cn/82604.Doc
qrd.guohua888.cn/40026.Doc
qrs.guohua888.cn/04862.Doc
qrs.guohua888.cn/40682.Doc
qrs.guohua888.cn/82488.Doc
qrs.guohua888.cn/82846.Doc
qrs.guohua888.cn/86686.Doc
qrs.guohua888.cn/84888.Doc
qrs.guohua888.cn/08024.Doc
qrs.guohua888.cn/48464.Doc
qrs.guohua888.cn/53151.Doc
qrs.guohua888.cn/86806.Doc
qra.guohua888.cn/44880.Doc
qra.guohua888.cn/48844.Doc
qra.guohua888.cn/62602.Doc
qra.guohua888.cn/60444.Doc
qra.guohua888.cn/26446.Doc
qra.guohua888.cn/62442.Doc
qra.guohua888.cn/42402.Doc
qra.guohua888.cn/84062.Doc
qra.guohua888.cn/86844.Doc
qra.guohua888.cn/24880.Doc
qrp.guohua888.cn/44424.Doc
qrp.guohua888.cn/28286.Doc
qrp.guohua888.cn/28604.Doc
qrp.guohua888.cn/62886.Doc
qrp.guohua888.cn/62628.Doc
qrp.guohua888.cn/66606.Doc
qrp.guohua888.cn/20064.Doc
qrp.guohua888.cn/28840.Doc
qrp.guohua888.cn/24440.Doc
qrp.guohua888.cn/00662.Doc
qro.guohua888.cn/86682.Doc
qro.guohua888.cn/04620.Doc
qro.guohua888.cn/44886.Doc
qro.guohua888.cn/48428.Doc
qro.guohua888.cn/44680.Doc
qro.guohua888.cn/88206.Doc
qro.guohua888.cn/28820.Doc
qro.guohua888.cn/02228.Doc
qro.guohua888.cn/02644.Doc
qro.guohua888.cn/44008.Doc
qri.guohua888.cn/82026.Doc
qri.guohua888.cn/48008.Doc
qri.guohua888.cn/24444.Doc
qri.guohua888.cn/28028.Doc
qri.guohua888.cn/95957.Doc
qri.guohua888.cn/66620.Doc
qri.guohua888.cn/42666.Doc
qri.guohua888.cn/20882.Doc
qri.guohua888.cn/35991.Doc
qri.guohua888.cn/80864.Doc
