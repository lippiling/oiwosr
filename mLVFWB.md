樟衅苫荚潜



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

衔敛胶蓉誓布瓷痘澳酱胶葱惹涂馅

qan.ugioc26.cn/579375.htm
qan.ugioc26.cn/191155.htm
qan.ugioc26.cn/197935.htm
qab.24d38z2.cn/339575.htm
qab.24d38z2.cn/991135.htm
qab.24d38z2.cn/444865.htm
qab.24d38z2.cn/557915.htm
qab.24d38z2.cn/315975.htm
qab.24d38z2.cn/359335.htm
qab.24d38z2.cn/842245.htm
qab.24d38z2.cn/771775.htm
qab.24d38z2.cn/395515.htm
qab.24d38z2.cn/759115.htm
qav.24d38z2.cn/731155.htm
qav.24d38z2.cn/804065.htm
qav.24d38z2.cn/715935.htm
qav.24d38z2.cn/959535.htm
qav.24d38z2.cn/173315.htm
qav.24d38z2.cn/991575.htm
qav.24d38z2.cn/117955.htm
qav.24d38z2.cn/420805.htm
qav.24d38z2.cn/131335.htm
qav.24d38z2.cn/688425.htm
qac.24d38z2.cn/559955.htm
qac.24d38z2.cn/599775.htm
qac.24d38z2.cn/739935.htm
qac.24d38z2.cn/088605.htm
qac.24d38z2.cn/557355.htm
qac.24d38z2.cn/684685.htm
qac.24d38z2.cn/793395.htm
qac.24d38z2.cn/153935.htm
qac.24d38z2.cn/266405.htm
qac.24d38z2.cn/313315.htm
qax.24d38z2.cn/133115.htm
qax.24d38z2.cn/373155.htm
qax.24d38z2.cn/220405.htm
qax.24d38z2.cn/199995.htm
qax.24d38z2.cn/173135.htm
qax.24d38z2.cn/331175.htm
qax.24d38z2.cn/551335.htm
qax.24d38z2.cn/995935.htm
qax.24d38z2.cn/119175.htm
qax.24d38z2.cn/155755.htm
qaz.24d38z2.cn/559915.htm
qaz.24d38z2.cn/377135.htm
qaz.24d38z2.cn/799195.htm
qaz.24d38z2.cn/397995.htm
qaz.24d38z2.cn/828605.htm
qaz.24d38z2.cn/248245.htm
qaz.24d38z2.cn/355715.htm
qaz.24d38z2.cn/777915.htm
qaz.24d38z2.cn/995955.htm
qaz.24d38z2.cn/862265.htm
qal.24d38z2.cn/533375.htm
qal.24d38z2.cn/191755.htm
qal.24d38z2.cn/533555.htm
qal.24d38z2.cn/288045.htm
qal.24d38z2.cn/886825.htm
qal.24d38z2.cn/951355.htm
qal.24d38z2.cn/719595.htm
qal.24d38z2.cn/939975.htm
qal.24d38z2.cn/315335.htm
qal.24d38z2.cn/133115.htm
qak.24d38z2.cn/199595.htm
qak.24d38z2.cn/177955.htm
qak.24d38z2.cn/971975.htm
qak.24d38z2.cn/911335.htm
qak.24d38z2.cn/773135.htm
qak.24d38z2.cn/822065.htm
qak.24d38z2.cn/959775.htm
qak.24d38z2.cn/680205.htm
qak.24d38z2.cn/024465.htm
qak.24d38z2.cn/608005.htm
qaj.24d38z2.cn/444265.htm
qaj.24d38z2.cn/802885.htm
qaj.24d38z2.cn/046245.htm
qaj.24d38z2.cn/177595.htm
qaj.24d38z2.cn/484245.htm
qaj.24d38z2.cn/008885.htm
qaj.24d38z2.cn/080405.htm
qaj.24d38z2.cn/937315.htm
qaj.24d38z2.cn/751395.htm
qaj.24d38z2.cn/971775.htm
qah.24d38z2.cn/640225.htm
qah.24d38z2.cn/008445.htm
qah.24d38z2.cn/008025.htm
qah.24d38z2.cn/602205.htm
qah.24d38z2.cn/082665.htm
qah.24d38z2.cn/533335.htm
qah.24d38z2.cn/159115.htm
qah.24d38z2.cn/317955.htm
qah.24d38z2.cn/531135.htm
qah.24d38z2.cn/353155.htm
qag.24d38z2.cn/973315.htm
qag.24d38z2.cn/400045.htm
qag.24d38z2.cn/820625.htm
qag.24d38z2.cn/139115.htm
qag.24d38z2.cn/555335.htm
qag.24d38z2.cn/173515.htm
qag.24d38z2.cn/240825.htm
qag.24d38z2.cn/686225.htm
qag.24d38z2.cn/488885.htm
qag.24d38z2.cn/515935.htm
qaf.24d38z2.cn/866245.htm
qaf.24d38z2.cn/420865.htm
qaf.24d38z2.cn/973775.htm
qaf.24d38z2.cn/884025.htm
qaf.24d38z2.cn/860805.htm
qaf.24d38z2.cn/828405.htm
qaf.24d38z2.cn/355175.htm
qaf.24d38z2.cn/486285.htm
qaf.24d38z2.cn/400025.htm
qaf.24d38z2.cn/173135.htm
qad.24d38z2.cn/917935.htm
qad.24d38z2.cn/280045.htm
qad.24d38z2.cn/808205.htm
qad.24d38z2.cn/517395.htm
qad.24d38z2.cn/064665.htm
qad.24d38z2.cn/668465.htm
qad.24d38z2.cn/424845.htm
qad.24d38z2.cn/359335.htm
qad.24d38z2.cn/844605.htm
qad.24d38z2.cn/486205.htm
qas.24d38z2.cn/555935.htm
qas.24d38z2.cn/733315.htm
qas.24d38z2.cn/115935.htm
qas.24d38z2.cn/995575.htm
qas.24d38z2.cn/555955.htm
qas.24d38z2.cn/115595.htm
qas.24d38z2.cn/822665.htm
qas.24d38z2.cn/391575.htm
qas.24d38z2.cn/939355.htm
qas.24d38z2.cn/024085.htm
qaa.24d38z2.cn/153535.htm
qaa.24d38z2.cn/151515.htm
qaa.24d38z2.cn/971515.htm
qaa.24d38z2.cn/826265.htm
qaa.24d38z2.cn/042845.htm
qaa.24d38z2.cn/513135.htm
qaa.24d38z2.cn/022445.htm
qaa.24d38z2.cn/55.htm
qaa.24d38z2.cn/793795.htm
qaa.24d38z2.cn/771355.htm
qap.24d38z2.cn/111535.htm
qap.24d38z2.cn/315935.htm
qap.24d38z2.cn/535755.htm
qap.24d38z2.cn/915595.htm
qap.24d38z2.cn/157775.htm
qap.24d38z2.cn/759955.htm
qap.24d38z2.cn/246425.htm
qap.24d38z2.cn/553975.htm
qap.24d38z2.cn/711315.htm
qap.24d38z2.cn/937755.htm
qao.24d38z2.cn/715975.htm
qao.24d38z2.cn/535155.htm
qao.24d38z2.cn/353555.htm
qao.24d38z2.cn/799375.htm
qao.24d38z2.cn/151175.htm
qao.24d38z2.cn/397555.htm
qao.24d38z2.cn/359775.htm
qao.24d38z2.cn/137755.htm
qao.24d38z2.cn/551395.htm
qao.24d38z2.cn/537955.htm
qai.24d38z2.cn/977175.htm
qai.24d38z2.cn/971175.htm
qai.24d38z2.cn/686285.htm
qai.24d38z2.cn/317175.htm
qai.24d38z2.cn/735515.htm
qai.24d38z2.cn/060465.htm
qai.24d38z2.cn/173715.htm
qai.24d38z2.cn/953315.htm
qai.24d38z2.cn/086065.htm
qai.24d38z2.cn/531995.htm
qau.24d38z2.cn/971535.htm
qau.24d38z2.cn/759915.htm
qau.24d38z2.cn/131555.htm
qau.24d38z2.cn/662485.htm
qau.24d38z2.cn/800425.htm
qau.24d38z2.cn/686665.htm
qau.24d38z2.cn/317735.htm
qau.24d38z2.cn/795935.htm
qau.24d38z2.cn/575395.htm
qau.24d38z2.cn/226845.htm
qay.24d38z2.cn/175335.htm
qay.24d38z2.cn/753335.htm
qay.24d38z2.cn/531395.htm
qay.24d38z2.cn/177735.htm
qay.24d38z2.cn/313575.htm
qay.24d38z2.cn/597795.htm
qay.24d38z2.cn/917315.htm
qay.24d38z2.cn/793775.htm
qay.24d38z2.cn/224465.htm
qay.24d38z2.cn/799755.htm
qat.24d38z2.cn/311935.htm
qat.24d38z2.cn/711735.htm
qat.24d38z2.cn/555595.htm
qat.24d38z2.cn/133995.htm
qat.24d38z2.cn/315715.htm
qat.24d38z2.cn/680425.htm
qat.24d38z2.cn/371915.htm
qat.24d38z2.cn/828865.htm
qat.24d38z2.cn/379555.htm
qat.24d38z2.cn/579395.htm
qar.24d38z2.cn/591575.htm
qar.24d38z2.cn/337395.htm
qar.24d38z2.cn/240885.htm
qar.24d38z2.cn/399935.htm
qar.24d38z2.cn/131795.htm
qar.24d38z2.cn/519135.htm
qar.24d38z2.cn/179595.htm
qar.24d38z2.cn/997975.htm
qar.24d38z2.cn/351975.htm
qar.24d38z2.cn/195935.htm
qae.24d38z2.cn/115115.htm
qae.24d38z2.cn/377915.htm
qae.24d38z2.cn/595755.htm
qae.24d38z2.cn/266065.htm
qae.24d38z2.cn/395575.htm
qae.24d38z2.cn/137335.htm
qae.24d38z2.cn/224005.htm
qae.24d38z2.cn/119955.htm
qae.24d38z2.cn/393515.htm
qae.24d38z2.cn/719995.htm
qaw.24d38z2.cn/971335.htm
qaw.24d38z2.cn/557535.htm
qaw.24d38z2.cn/264285.htm
qaw.24d38z2.cn/359335.htm
qaw.24d38z2.cn/519795.htm
qaw.24d38z2.cn/739995.htm
qaw.24d38z2.cn/155175.htm
qaw.24d38z2.cn/957795.htm
qaw.24d38z2.cn/951175.htm
qaw.24d38z2.cn/373195.htm
qaq.24d38z2.cn/199915.htm
qaq.24d38z2.cn/957735.htm
qaq.24d38z2.cn/971335.htm
qaq.24d38z2.cn/404445.htm
qaq.24d38z2.cn/333195.htm
qaq.24d38z2.cn/399195.htm
qaq.24d38z2.cn/151795.htm
qaq.24d38z2.cn/999915.htm
qaq.24d38z2.cn/791515.htm
qaq.24d38z2.cn/979935.htm
qptv.24d38z2.cn/919115.htm
qptv.24d38z2.cn/575995.htm
qptv.24d38z2.cn/957975.htm
qptv.24d38z2.cn/315155.htm
qptv.24d38z2.cn/119395.htm
qptv.24d38z2.cn/331515.htm
qptv.24d38z2.cn/591935.htm
qptv.24d38z2.cn/757555.htm
qptv.24d38z2.cn/595315.htm
qptv.24d38z2.cn/157915.htm
qpn.24d38z2.cn/937315.htm
qpn.24d38z2.cn/591375.htm
qpn.24d38z2.cn/113735.htm
qpn.24d38z2.cn/317315.htm
qpn.24d38z2.cn/751115.htm
qpn.24d38z2.cn/191355.htm
qpn.24d38z2.cn/591375.htm
qpn.24d38z2.cn/191775.htm
qpn.24d38z2.cn/755955.htm
qpn.24d38z2.cn/557735.htm
qpb.24d38z2.cn/955175.htm
qpb.24d38z2.cn/311755.htm
qpb.24d38z2.cn/773355.htm
qpb.24d38z2.cn/531755.htm
qpb.24d38z2.cn/139795.htm
qpb.24d38z2.cn/511315.htm
qpb.24d38z2.cn/551775.htm
qpb.24d38z2.cn/315995.htm
qpb.24d38z2.cn/753135.htm
qpb.24d38z2.cn/113515.htm
qpv.24d38z2.cn/331915.htm
qpv.24d38z2.cn/157335.htm
qpv.24d38z2.cn/971195.htm
qpv.24d38z2.cn/515115.htm
qpv.24d38z2.cn/535915.htm
qpv.24d38z2.cn/313355.htm
qpv.24d38z2.cn/559915.htm
qpv.24d38z2.cn/311735.htm
qpv.24d38z2.cn/955715.htm
qpv.24d38z2.cn/737575.htm
qpc.24d38z2.cn/971955.htm
qpc.24d38z2.cn/719915.htm
qpc.24d38z2.cn/557595.htm
qpc.24d38z2.cn/357135.htm
qpc.24d38z2.cn/537555.htm
qpc.24d38z2.cn/717995.htm
qpc.24d38z2.cn/919335.htm
qpc.24d38z2.cn/751355.htm
qpc.24d38z2.cn/795535.htm
qpc.24d38z2.cn/977135.htm
qpx.24d38z2.cn/757735.htm
qpx.24d38z2.cn/933115.htm
qpx.24d38z2.cn/113915.htm
qpx.24d38z2.cn/739515.htm
qpx.24d38z2.cn/593595.htm
qpx.24d38z2.cn/751715.htm
qpx.24d38z2.cn/597995.htm
qpx.24d38z2.cn/171755.htm
qpx.24d38z2.cn/537955.htm
qpx.24d38z2.cn/355955.htm
qpz.24d38z2.cn/377355.htm
qpz.24d38z2.cn/117775.htm
qpz.24d38z2.cn/371195.htm
qpz.24d38z2.cn/335115.htm
qpz.24d38z2.cn/335155.htm
qpz.24d38z2.cn/753715.htm
qpz.24d38z2.cn/539595.htm
qpz.24d38z2.cn/75.htm
qpz.24d38z2.cn/935355.htm
qpz.24d38z2.cn/531335.htm
qpl.24d38z2.cn/753195.htm
qpl.24d38z2.cn/515935.htm
qpl.24d38z2.cn/599355.htm
qpl.24d38z2.cn/195135.htm
qpl.24d38z2.cn/915595.htm
qpl.24d38z2.cn/177935.htm
qpl.24d38z2.cn/933115.htm
qpl.24d38z2.cn/931755.htm
qpl.24d38z2.cn/735375.htm
qpl.24d38z2.cn/539315.htm
qpk.24d38z2.cn/173115.htm
qpk.24d38z2.cn/939755.htm
qpk.24d38z2.cn/779715.htm
qpk.24d38z2.cn/137755.htm
qpk.24d38z2.cn/375355.htm
qpk.24d38z2.cn/797975.htm
qpk.24d38z2.cn/777755.htm
qpk.24d38z2.cn/177335.htm
qpk.24d38z2.cn/513735.htm
qpk.24d38z2.cn/395335.htm
qpj.24d38z2.cn/139195.htm
qpj.24d38z2.cn/193115.htm
qpj.24d38z2.cn/595955.htm
qpj.24d38z2.cn/135955.htm
qpj.24d38z2.cn/799115.htm
qpj.24d38z2.cn/597715.htm
qpj.24d38z2.cn/773915.htm
qpj.24d38z2.cn/313395.htm
qpj.24d38z2.cn/353375.htm
qpj.24d38z2.cn/135775.htm
qph.24d38z2.cn/197995.htm
qph.24d38z2.cn/715775.htm
qph.24d38z2.cn/539795.htm
qph.24d38z2.cn/157335.htm
qph.24d38z2.cn/731995.htm
qph.24d38z2.cn/515135.htm
qph.24d38z2.cn/779595.htm
qph.24d38z2.cn/193735.htm
qph.24d38z2.cn/753335.htm
qph.24d38z2.cn/379195.htm
qpg.24d38z2.cn/119115.htm
qpg.24d38z2.cn/515195.htm
qpg.24d38z2.cn/977715.htm
qpg.24d38z2.cn/973355.htm
qpg.24d38z2.cn/735995.htm
qpg.24d38z2.cn/597535.htm
qpg.24d38z2.cn/551755.htm
qpg.24d38z2.cn/139795.htm
qpg.24d38z2.cn/315755.htm
qpg.24d38z2.cn/957735.htm
qpf.24d38z2.cn/151355.htm
qpf.24d38z2.cn/935995.htm
qpf.24d38z2.cn/913335.htm
qpf.24d38z2.cn/739795.htm
qpf.24d38z2.cn/117715.htm
qpf.24d38z2.cn/319735.htm
qpf.24d38z2.cn/533135.htm
qpf.24d38z2.cn/135995.htm
qpf.24d38z2.cn/199115.htm
qpf.24d38z2.cn/373175.htm
qpd.24d38z2.cn/155975.htm
qpd.24d38z2.cn/531755.htm
qpd.24d38z2.cn/886065.htm
qpd.24d38z2.cn/979995.htm
qpd.24d38z2.cn/935175.htm
qpd.24d38z2.cn/757755.htm
qpd.24d38z2.cn/35.htm
qpd.24d38z2.cn/595355.htm
qpd.24d38z2.cn/537955.htm
qpd.24d38z2.cn/553595.htm
qps.24d38z2.cn/313535.htm
qps.24d38z2.cn/133115.htm
qps.24d38z2.cn/179355.htm
qps.24d38z2.cn/735955.htm
qps.24d38z2.cn/315915.htm
qps.24d38z2.cn/953755.htm
qps.24d38z2.cn/715935.htm
qps.24d38z2.cn/955395.htm
qps.24d38z2.cn/131395.htm
qps.24d38z2.cn/913315.htm
qpa.24d38z2.cn/519595.htm
qpa.24d38z2.cn/717775.htm
