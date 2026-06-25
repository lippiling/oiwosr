
============================================================
 Python哈希表与字典实现 — dict底层/hash与eq/集与冻结集
============================================================

Python的dict和set底层使用哈希表实现，是Python中最常用的数据结构之一。
理解哈希表的工作机制对写出高效、正确的Python代码至关重要。

============================================================
1. Python dict 底层原理
============================================================
# Python 3.6+ 的 dict 使用"紧凑哈希表"实现，比旧版节省约50%内存
# Python 3.7+ 保证字典保持插入顺序
#
# 内部结构：
#   1. 哈希数组（indices）：存储稀疏的索引信息
#   2. 条目数组（entries）：按插入顺序紧凑存储键值对
#
# 冲突解决：开放地址法（Open Addressing），使用伪随机探测序列
# 负载因子（Load Factor）：dict约为2/3，当哈希表密度达到此值时触发扩容

============================================================
2. 自定义 __hash__ 和 __eq__
============================================================

class Person:
    """自定义类，实现hash和eq以便用作字典键或集合元素"""
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __hash__(self):
        """哈希函数：使用name和age的组合哈希值"""
        return hash((self.name, self.age))  # 元组是hashable的

    def __eq__(self, other):
        """相等判断：两个Person的name和age都相等时才相等"""
        if not isinstance(other, Person):
            return False
        return self.name == other.name and self.age == other.age

    def __repr__(self):
        return f"Person({self.name}, {self.age})"

# 使用自定义类作为字典键
p1 = Person("Alice", 25)
p2 = Person("Bob", 30)
p3 = Person("Alice", 25)    # 与p1相等

d = {p1: "工程师", p2: "设计师"}
print(d[p1])                # "工程师"
print(d[p3])                # "工程师"（因为p3 == p1，哈希值相同）
print(d[Person("Alice", 25)])  # "工程师"（相同hash和eq）

============================================================
3. hashable vs unhashable 类型
============================================================
# hashable（可哈希）类型：可作为字典键或集合元素
# - int, float, str, tuple(仅当所有元素hashable), frozenset
# - 自定义类默认hashable（id-based）
#
# unhashable（不可哈希）类型：尝试作为字典键会抛出TypeError
# - list, dict, set, bytearray
# - 包含不可哈希元素的tuple

# 合法示例：使用tuple作为键
location_map = {
    (40.7128, -74.0060): "纽约",
    (51.5074, -0.1278): "伦敦"
}
print("坐标对应城市:", location_map[(40.7128, -74.0060)])

# 非法示例（会报错）：
# bad_dict = {[1, 2, 3]: "list"}  # TypeError: unhashable type: 'list'

============================================================
4. set 的内部实现
============================================================
# Python的set也是基于哈希表实现的，与dict类似但没有value部分
# 特性：
# - 元素必须hashable
# - 无序（但Python 3.7+的迭代顺序实际与插入顺序相关）
# - O(1)平均查找、插入、删除
# - 不支持索引和切片

s = set()
s.add(3)            # 添加元素
s.add(1)
s.add(2)
s.add(3)            # 重复元素不会添加
print("set:", s)            # {1, 2, 3}
s.discard(2)        # 安全删除（元素不存在不会报错）
s.remove(1)         # 删除元素（不存在会抛出KeyError）
print("after remove:", s)

# 集合运算：并集、交集、差集、对称差
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}
print("并集:", a | b)                 # {1, 2, 3, 4, 5, 6}
print("交集:", a & b)                 # {3, 4}
print("差集:", a - b)                 # {1, 2}
print("对称差:", a ^ b)               # {1, 2, 5, 6}
print("子集检查:", {1, 2}.issubset(a))  # True
print("超集检查:", a.issuperset({1, 2})) # True

============================================================
5. frozenset 不可变集合
============================================================
# frozenset是不可变的set，可以用作字典键或set的元素
fs = frozenset([1, 2, 3])
print("frozenset:", fs)         # frozenset({1, 2, 3})

# frozenset作为字典键
inventory = {
    frozenset(["苹果", "香蕉"]): "水果区",
    frozenset(["牛奶", "酸奶"]): "乳制品区"
}
print("库存分类:", inventory)

# frozenset作为set的元素
set_of_sets = {frozenset([1, 2]), frozenset([3, 4])}
print("集合的集合:", set_of_sets)

============================================================
6. 负载因子和扩容（Resizing）
============================================================
# 当dict/set中的元素数量超过哈希表容量的2/3时触发扩容
# 扩容时哈希表大小通常翻倍（或更大）
# 扩容后所有元素需要重新哈希（rehash），这是O(n)操作
# 因此提前预分配大小可以避免多次扩容

# 预估元素数量时预分配大小（仅作示例，实际不直接暴露容量接口）
def build_large_dict(n):
    """预分配容量避免多次扩容"""
    # Python 3.6+ 没有公开的预分配API
    # 但可以通过创建后批量添加来减少扩容次数
    d = {}
    for i in range(n):
        d[i] = str(i)
    return d

============================================================
7. Python 3.7+ 字典有序性保证
============================================================
# Python 3.7 正式将字典保持插入顺序列为语言规范
# 之前的Python版本中dict是无序的或顺序只是实现细节
#
# 如果需要"有序字典"，有几种选择：
# 1. dict (Python 3.7+) — 保持插入顺序
# 2. collections.OrderedDict — 额外的功能如move_to_end()
# 3. 按key排序：dict(sorted(d.items()))

# 演示dict保持插入顺序
ordered = {}
ordered["z"] = 1
ordered["a"] = 2
ordered["m"] = 3
ordered["b"] = 4
print("有序字典:", list(ordered.keys()))  # ['z', 'a', 'm', 'b']

# 按key排序
sorted_dict = dict(sorted(ordered.items()))
print("排序后:", list(sorted_dict.keys()))  # ['a', 'b', 'm', 'z']

============================================================
使用示例
============================================================
if __name__ == "__main__":
    # 计数器：使用dict统计频率
    text = "hello world"
    freq = {}
    for ch in text:
        freq[ch] = freq.get(ch, 0) + 1
    print("字符频率:", freq)

    # 使用set去重
    nums = [1, 2, 2, 3, 3, 3, 4, 5, 5]
    unique = list(set(nums))
    print("去重后:", unique)

eer.yghdsxcnbza.cn/46266.Doc
eer.yghdsxcnbza.cn/06888.Doc
eer.yghdsxcnbza.cn/80400.Doc
eer.yghdsxcnbza.cn/66408.Doc
eer.yghdsxcnbza.cn/08820.Doc
eer.yghdsxcnbza.cn/20088.Doc
eer.yghdsxcnbza.cn/82260.Doc
eer.yghdsxcnbza.cn/26688.Doc
eer.yghdsxcnbza.cn/82680.Doc
eer.yghdsxcnbza.cn/40008.Doc
eee.yghdsxcnbza.cn/20226.Doc
eee.yghdsxcnbza.cn/22242.Doc
eee.yghdsxcnbza.cn/26022.Doc
eee.yghdsxcnbza.cn/24686.Doc
eee.yghdsxcnbza.cn/08604.Doc
eee.yghdsxcnbza.cn/26622.Doc
eee.yghdsxcnbza.cn/00480.Doc
eee.yghdsxcnbza.cn/22420.Doc
eee.yghdsxcnbza.cn/24042.Doc
eee.yghdsxcnbza.cn/24462.Doc
eew.yghdsxcnbza.cn/26804.Doc
eew.yghdsxcnbza.cn/44844.Doc
eew.yghdsxcnbza.cn/62468.Doc
eew.yghdsxcnbza.cn/66064.Doc
eew.yghdsxcnbza.cn/62828.Doc
eew.yghdsxcnbza.cn/75935.Doc
eew.yghdsxcnbza.cn/80840.Doc
eew.yghdsxcnbza.cn/26880.Doc
eew.yghdsxcnbza.cn/40204.Doc
eew.yghdsxcnbza.cn/62466.Doc
eeq.yghdsxcnbza.cn/82660.Doc
eeq.yghdsxcnbza.cn/31317.Doc
eeq.yghdsxcnbza.cn/42840.Doc
eeq.yghdsxcnbza.cn/48280.Doc
eeq.yghdsxcnbza.cn/66400.Doc
eeq.yghdsxcnbza.cn/66280.Doc
eeq.yghdsxcnbza.cn/08842.Doc
eeq.yghdsxcnbza.cn/48284.Doc
eeq.yghdsxcnbza.cn/42628.Doc
eeq.yghdsxcnbza.cn/08060.Doc
ewm.yghdsxcnbza.cn/82804.Doc
ewm.yghdsxcnbza.cn/68226.Doc
ewm.yghdsxcnbza.cn/86068.Doc
ewm.yghdsxcnbza.cn/66000.Doc
ewm.yghdsxcnbza.cn/84000.Doc
ewm.yghdsxcnbza.cn/15913.Doc
ewm.yghdsxcnbza.cn/66202.Doc
ewm.yghdsxcnbza.cn/22486.Doc
ewm.yghdsxcnbza.cn/42628.Doc
ewm.yghdsxcnbza.cn/71515.Doc
ewn.yghdsxcnbza.cn/68640.Doc
ewn.yghdsxcnbza.cn/40642.Doc
ewn.yghdsxcnbza.cn/28224.Doc
ewn.yghdsxcnbza.cn/53337.Doc
ewn.yghdsxcnbza.cn/68004.Doc
ewn.yghdsxcnbza.cn/62642.Doc
ewn.yghdsxcnbza.cn/26080.Doc
ewn.yghdsxcnbza.cn/28880.Doc
ewn.yghdsxcnbza.cn/40680.Doc
ewn.yghdsxcnbza.cn/60228.Doc
ewb.yghdsxcnbza.cn/40480.Doc
ewb.yghdsxcnbza.cn/40488.Doc
ewb.yghdsxcnbza.cn/86460.Doc
ewb.yghdsxcnbza.cn/06086.Doc
ewb.yghdsxcnbza.cn/60422.Doc
ewb.yghdsxcnbza.cn/40004.Doc
ewb.yghdsxcnbza.cn/26246.Doc
ewb.yghdsxcnbza.cn/86284.Doc
ewb.yghdsxcnbza.cn/04666.Doc
ewb.yghdsxcnbza.cn/68822.Doc
ewv.yghdsxcnbza.cn/46606.Doc
ewv.yghdsxcnbza.cn/08662.Doc
ewv.yghdsxcnbza.cn/48404.Doc
ewv.yghdsxcnbza.cn/73577.Doc
ewv.yghdsxcnbza.cn/46846.Doc
ewv.yghdsxcnbza.cn/40804.Doc
ewv.yghdsxcnbza.cn/84426.Doc
ewv.yghdsxcnbza.cn/59559.Doc
ewv.yghdsxcnbza.cn/62088.Doc
ewv.yghdsxcnbza.cn/24004.Doc
ewc.yghdsxcnbza.cn/06046.Doc
ewc.yghdsxcnbza.cn/02404.Doc
ewc.yghdsxcnbza.cn/71551.Doc
ewc.yghdsxcnbza.cn/04206.Doc
ewc.yghdsxcnbza.cn/80688.Doc
ewc.yghdsxcnbza.cn/73155.Doc
ewc.yghdsxcnbza.cn/68400.Doc
ewc.yghdsxcnbza.cn/02426.Doc
ewc.yghdsxcnbza.cn/80446.Doc
ewc.yghdsxcnbza.cn/60642.Doc
ewx.yghdsxcnbza.cn/24260.Doc
ewx.yghdsxcnbza.cn/86066.Doc
ewx.yghdsxcnbza.cn/42228.Doc
ewx.yghdsxcnbza.cn/24664.Doc
ewx.yghdsxcnbza.cn/66608.Doc
ewx.yghdsxcnbza.cn/86806.Doc
ewx.yghdsxcnbza.cn/31995.Doc
ewx.yghdsxcnbza.cn/80882.Doc
ewx.yghdsxcnbza.cn/64068.Doc
ewx.yghdsxcnbza.cn/40884.Doc
ewz.yghdsxcnbza.cn/22266.Doc
ewz.yghdsxcnbza.cn/48286.Doc
ewz.yghdsxcnbza.cn/22082.Doc
ewz.yghdsxcnbza.cn/82622.Doc
ewz.yghdsxcnbza.cn/33353.Doc
ewz.yghdsxcnbza.cn/82888.Doc
ewz.yghdsxcnbza.cn/88602.Doc
ewz.yghdsxcnbza.cn/28240.Doc
ewz.yghdsxcnbza.cn/48804.Doc
ewz.yghdsxcnbza.cn/24608.Doc
ewl.yghdsxcnbza.cn/82060.Doc
ewl.yghdsxcnbza.cn/24624.Doc
ewl.yghdsxcnbza.cn/99117.Doc
ewl.yghdsxcnbza.cn/04668.Doc
ewl.yghdsxcnbza.cn/80842.Doc
ewl.yghdsxcnbza.cn/06048.Doc
ewl.yghdsxcnbza.cn/66402.Doc
ewl.yghdsxcnbza.cn/00484.Doc
ewl.yghdsxcnbza.cn/84284.Doc
ewl.yghdsxcnbza.cn/48806.Doc
ewk.yghdsxcnbza.cn/86684.Doc
ewk.yghdsxcnbza.cn/24066.Doc
ewk.yghdsxcnbza.cn/82086.Doc
ewk.yghdsxcnbza.cn/44244.Doc
ewk.yghdsxcnbza.cn/28822.Doc
ewk.yghdsxcnbza.cn/86664.Doc
ewk.yghdsxcnbza.cn/48848.Doc
ewk.yghdsxcnbza.cn/60840.Doc
ewk.yghdsxcnbza.cn/55979.Doc
ewk.yghdsxcnbza.cn/08088.Doc
ewj.yghdsxcnbza.cn/06868.Doc
ewj.yghdsxcnbza.cn/13751.Doc
ewj.yghdsxcnbza.cn/62662.Doc
ewj.yghdsxcnbza.cn/46208.Doc
ewj.yghdsxcnbza.cn/48808.Doc
ewj.yghdsxcnbza.cn/22060.Doc
ewj.yghdsxcnbza.cn/20628.Doc
ewj.yghdsxcnbza.cn/39515.Doc
ewj.yghdsxcnbza.cn/46240.Doc
ewj.yghdsxcnbza.cn/08486.Doc
ewh.yghdsxcnbza.cn/44284.Doc
ewh.yghdsxcnbza.cn/04882.Doc
ewh.yghdsxcnbza.cn/59999.Doc
ewh.yghdsxcnbza.cn/46246.Doc
ewh.yghdsxcnbza.cn/48224.Doc
ewh.yghdsxcnbza.cn/20002.Doc
ewh.yghdsxcnbza.cn/64684.Doc
ewh.yghdsxcnbza.cn/46886.Doc
ewh.yghdsxcnbza.cn/00440.Doc
ewh.yghdsxcnbza.cn/44662.Doc
ewg.yghdsxcnbza.cn/22820.Doc
ewg.yghdsxcnbza.cn/42044.Doc
ewg.yghdsxcnbza.cn/64646.Doc
ewg.yghdsxcnbza.cn/02248.Doc
ewg.yghdsxcnbza.cn/28402.Doc
ewg.yghdsxcnbza.cn/86824.Doc
ewg.yghdsxcnbza.cn/13911.Doc
ewg.yghdsxcnbza.cn/46086.Doc
ewg.yghdsxcnbza.cn/66644.Doc
ewg.yghdsxcnbza.cn/20444.Doc
ewf.yghdsxcnbza.cn/28062.Doc
ewf.yghdsxcnbza.cn/02228.Doc
ewf.yghdsxcnbza.cn/62086.Doc
ewf.yghdsxcnbza.cn/68044.Doc
ewf.yghdsxcnbza.cn/04660.Doc
ewf.yghdsxcnbza.cn/88020.Doc
ewf.yghdsxcnbza.cn/42880.Doc
ewf.yghdsxcnbza.cn/44204.Doc
ewf.yghdsxcnbza.cn/91331.Doc
ewf.yghdsxcnbza.cn/04800.Doc
ewd.yghdsxcnbza.cn/48406.Doc
ewd.yghdsxcnbza.cn/60844.Doc
ewd.yghdsxcnbza.cn/86640.Doc
ewd.yghdsxcnbza.cn/22486.Doc
ewd.yghdsxcnbza.cn/66824.Doc
ewd.yghdsxcnbza.cn/08060.Doc
ewd.yghdsxcnbza.cn/22024.Doc
ewd.yghdsxcnbza.cn/60684.Doc
ewd.yghdsxcnbza.cn/02884.Doc
ewd.yghdsxcnbza.cn/86680.Doc
ews.yghdsxcnbza.cn/24026.Doc
ews.yghdsxcnbza.cn/35739.Doc
ews.yghdsxcnbza.cn/66624.Doc
ews.yghdsxcnbza.cn/60246.Doc
ews.yghdsxcnbza.cn/40420.Doc
ews.yghdsxcnbza.cn/24244.Doc
ews.yghdsxcnbza.cn/82604.Doc
ews.yghdsxcnbza.cn/26284.Doc
ews.yghdsxcnbza.cn/68646.Doc
ews.yghdsxcnbza.cn/02004.Doc
ewa.yghdsxcnbza.cn/55995.Doc
ewa.yghdsxcnbza.cn/39557.Doc
ewa.yghdsxcnbza.cn/26088.Doc
ewa.yghdsxcnbza.cn/84002.Doc
ewa.yghdsxcnbza.cn/88684.Doc
ewa.yghdsxcnbza.cn/48224.Doc
ewa.yghdsxcnbza.cn/37957.Doc
ewa.yghdsxcnbza.cn/40686.Doc
ewa.yghdsxcnbza.cn/46264.Doc
ewa.yghdsxcnbza.cn/46200.Doc
ewp.yghdsxcnbza.cn/44228.Doc
ewp.yghdsxcnbza.cn/24664.Doc
ewp.yghdsxcnbza.cn/20202.Doc
ewp.yghdsxcnbza.cn/39173.Doc
ewp.yghdsxcnbza.cn/44224.Doc
ewp.yghdsxcnbza.cn/26244.Doc
ewp.yghdsxcnbza.cn/82600.Doc
ewp.yghdsxcnbza.cn/20806.Doc
ewp.yghdsxcnbza.cn/17555.Doc
ewp.yghdsxcnbza.cn/66444.Doc
ewo.yghdsxcnbza.cn/24422.Doc
ewo.yghdsxcnbza.cn/64242.Doc
ewo.yghdsxcnbza.cn/40864.Doc
ewo.yghdsxcnbza.cn/39737.Doc
ewo.yghdsxcnbza.cn/46866.Doc
ewo.yghdsxcnbza.cn/88464.Doc
ewo.yghdsxcnbza.cn/86626.Doc
ewo.yghdsxcnbza.cn/86080.Doc
ewo.yghdsxcnbza.cn/86886.Doc
ewo.yghdsxcnbza.cn/00264.Doc
ewi.yghdsxcnbza.cn/86826.Doc
ewi.yghdsxcnbza.cn/42662.Doc
ewi.yghdsxcnbza.cn/20204.Doc
ewi.yghdsxcnbza.cn/86248.Doc
ewi.yghdsxcnbza.cn/08024.Doc
ewi.yghdsxcnbza.cn/77977.Doc
ewi.yghdsxcnbza.cn/64806.Doc
ewi.yghdsxcnbza.cn/62048.Doc
ewi.yghdsxcnbza.cn/11935.Doc
ewi.yghdsxcnbza.cn/62020.Doc
ewu.yghdsxcnbza.cn/68066.Doc
ewu.yghdsxcnbza.cn/20048.Doc
ewu.yghdsxcnbza.cn/40240.Doc
ewu.yghdsxcnbza.cn/46280.Doc
ewu.yghdsxcnbza.cn/02844.Doc
ewu.yghdsxcnbza.cn/26228.Doc
ewu.yghdsxcnbza.cn/48866.Doc
ewu.yghdsxcnbza.cn/88642.Doc
ewu.yghdsxcnbza.cn/24422.Doc
ewu.yghdsxcnbza.cn/42882.Doc
ewy.yghdsxcnbza.cn/48446.Doc
ewy.yghdsxcnbza.cn/80424.Doc
ewy.yghdsxcnbza.cn/64002.Doc
ewy.yghdsxcnbza.cn/64644.Doc
ewy.yghdsxcnbza.cn/82822.Doc
ewy.yghdsxcnbza.cn/66280.Doc
ewy.yghdsxcnbza.cn/26284.Doc
ewy.yghdsxcnbza.cn/64668.Doc
ewy.yghdsxcnbza.cn/88628.Doc
ewy.yghdsxcnbza.cn/97717.Doc
ewt.yghdsxcnbza.cn/68642.Doc
ewt.yghdsxcnbza.cn/00400.Doc
ewt.yghdsxcnbza.cn/39933.Doc
ewt.yghdsxcnbza.cn/48646.Doc
ewt.yghdsxcnbza.cn/84062.Doc
ewt.yghdsxcnbza.cn/00280.Doc
ewt.yghdsxcnbza.cn/35173.Doc
ewt.yghdsxcnbza.cn/44448.Doc
ewt.yghdsxcnbza.cn/02024.Doc
ewt.yghdsxcnbza.cn/73175.Doc
ewr.yghdsxcnbza.cn/88048.Doc
ewr.yghdsxcnbza.cn/79111.Doc
ewr.yghdsxcnbza.cn/66046.Doc
ewr.yghdsxcnbza.cn/46268.Doc
ewr.yghdsxcnbza.cn/86040.Doc
ewr.yghdsxcnbza.cn/82240.Doc
ewr.yghdsxcnbza.cn/28080.Doc
ewr.yghdsxcnbza.cn/08282.Doc
ewr.yghdsxcnbza.cn/26020.Doc
ewr.yghdsxcnbza.cn/59373.Doc
ewe.yghdsxcnbza.cn/39195.Doc
ewe.yghdsxcnbza.cn/80288.Doc
ewe.yghdsxcnbza.cn/00242.Doc
ewe.yghdsxcnbza.cn/46002.Doc
ewe.yghdsxcnbza.cn/00208.Doc
ewe.yghdsxcnbza.cn/62846.Doc
ewe.yghdsxcnbza.cn/44006.Doc
ewe.yghdsxcnbza.cn/00066.Doc
ewe.yghdsxcnbza.cn/00686.Doc
ewe.yghdsxcnbza.cn/57555.Doc
eww.yghdsxcnbza.cn/60688.Doc
eww.yghdsxcnbza.cn/60606.Doc
eww.yghdsxcnbza.cn/40224.Doc
eww.yghdsxcnbza.cn/06400.Doc
eww.yghdsxcnbza.cn/60026.Doc
eww.yghdsxcnbza.cn/64242.Doc
eww.yghdsxcnbza.cn/40280.Doc
eww.yghdsxcnbza.cn/40624.Doc
eww.yghdsxcnbza.cn/46468.Doc
eww.yghdsxcnbza.cn/00880.Doc
ewq.yghdsxcnbza.cn/19937.Doc
ewq.yghdsxcnbza.cn/20228.Doc
ewq.yghdsxcnbza.cn/88064.Doc
ewq.yghdsxcnbza.cn/24000.Doc
ewq.yghdsxcnbza.cn/55179.Doc
ewq.yghdsxcnbza.cn/88000.Doc
ewq.yghdsxcnbza.cn/06286.Doc
ewq.yghdsxcnbza.cn/86482.Doc
ewq.yghdsxcnbza.cn/42824.Doc
ewq.yghdsxcnbza.cn/28628.Doc
eqm.yghdsxcnbza.cn/68608.Doc
eqm.yghdsxcnbza.cn/22222.Doc
eqm.yghdsxcnbza.cn/60480.Doc
eqm.yghdsxcnbza.cn/19973.Doc
eqm.yghdsxcnbza.cn/48244.Doc
eqm.yghdsxcnbza.cn/84468.Doc
eqm.yghdsxcnbza.cn/68048.Doc
eqm.yghdsxcnbza.cn/42428.Doc
eqm.yghdsxcnbza.cn/40004.Doc
eqm.yghdsxcnbza.cn/64228.Doc
eqn.yghdsxcnbza.cn/22862.Doc
eqn.yghdsxcnbza.cn/82464.Doc
eqn.yghdsxcnbza.cn/42268.Doc
eqn.yghdsxcnbza.cn/04046.Doc
eqn.yghdsxcnbza.cn/82020.Doc
eqn.yghdsxcnbza.cn/22666.Doc
eqn.yghdsxcnbza.cn/40646.Doc
eqn.yghdsxcnbza.cn/80488.Doc
eqn.yghdsxcnbza.cn/60006.Doc
eqn.yghdsxcnbza.cn/00602.Doc
eqb.yghdsxcnbza.cn/24068.Doc
eqb.yghdsxcnbza.cn/71599.Doc
eqb.yghdsxcnbza.cn/82822.Doc
eqb.yghdsxcnbza.cn/66040.Doc
eqb.yghdsxcnbza.cn/40242.Doc
eqb.yghdsxcnbza.cn/95759.Doc
eqb.yghdsxcnbza.cn/84846.Doc
eqb.yghdsxcnbza.cn/40464.Doc
eqb.yghdsxcnbza.cn/40028.Doc
eqb.yghdsxcnbza.cn/02262.Doc
eqv.yghdsxcnbza.cn/22824.Doc
eqv.yghdsxcnbza.cn/82488.Doc
eqv.yghdsxcnbza.cn/84806.Doc
eqv.yghdsxcnbza.cn/91391.Doc
eqv.yghdsxcnbza.cn/59359.Doc
eqv.yghdsxcnbza.cn/44224.Doc
eqv.yghdsxcnbza.cn/55573.Doc
eqv.yghdsxcnbza.cn/08606.Doc
eqv.yghdsxcnbza.cn/42462.Doc
eqv.yghdsxcnbza.cn/24868.Doc
eqc.yghdsxcnbza.cn/22802.Doc
eqc.yghdsxcnbza.cn/04462.Doc
eqc.yghdsxcnbza.cn/26620.Doc
eqc.yghdsxcnbza.cn/64468.Doc
eqc.yghdsxcnbza.cn/62086.Doc
eqc.yghdsxcnbza.cn/80088.Doc
eqc.yghdsxcnbza.cn/48462.Doc
eqc.yghdsxcnbza.cn/79555.Doc
eqc.yghdsxcnbza.cn/40400.Doc
eqc.yghdsxcnbza.cn/11179.Doc
eqx.yghdsxcnbza.cn/40688.Doc
eqx.yghdsxcnbza.cn/46848.Doc
eqx.yghdsxcnbza.cn/40040.Doc
eqx.yghdsxcnbza.cn/86062.Doc
eqx.yghdsxcnbza.cn/60228.Doc
eqx.yghdsxcnbza.cn/60844.Doc
eqx.yghdsxcnbza.cn/08662.Doc
eqx.yghdsxcnbza.cn/08642.Doc
eqx.yghdsxcnbza.cn/80268.Doc
eqx.yghdsxcnbza.cn/48888.Doc
eqz.yghdsxcnbza.cn/31757.Doc
eqz.yghdsxcnbza.cn/08200.Doc
eqz.yghdsxcnbza.cn/33579.Doc
eqz.yghdsxcnbza.cn/46886.Doc
eqz.yghdsxcnbza.cn/66002.Doc
eqz.yghdsxcnbza.cn/73717.Doc
eqz.yghdsxcnbza.cn/82062.Doc
eqz.yghdsxcnbza.cn/64240.Doc
eqz.yghdsxcnbza.cn/11197.Doc
eqz.yghdsxcnbza.cn/68846.Doc
eql.yghdsxcnbza.cn/73591.Doc
eql.yghdsxcnbza.cn/44422.Doc
eql.yghdsxcnbza.cn/66428.Doc
eql.yghdsxcnbza.cn/26426.Doc
eql.yghdsxcnbza.cn/99515.Doc
eql.yghdsxcnbza.cn/86680.Doc
eql.yghdsxcnbza.cn/08066.Doc
eql.yghdsxcnbza.cn/00088.Doc
eql.yghdsxcnbza.cn/22086.Doc
eql.yghdsxcnbza.cn/08488.Doc
eqk.yghdsxcnbza.cn/44848.Doc
eqk.yghdsxcnbza.cn/17575.Doc
eqk.yghdsxcnbza.cn/04624.Doc
eqk.yghdsxcnbza.cn/22604.Doc
eqk.yghdsxcnbza.cn/31973.Doc
eqk.yghdsxcnbza.cn/44842.Doc
eqk.yghdsxcnbza.cn/46040.Doc
eqk.yghdsxcnbza.cn/37939.Doc
eqk.yghdsxcnbza.cn/44064.Doc
eqk.yghdsxcnbza.cn/44026.Doc
eqj.yghdsxcnbza.cn/51593.Doc
eqj.yghdsxcnbza.cn/64880.Doc
eqj.yghdsxcnbza.cn/84428.Doc
eqj.yghdsxcnbza.cn/28202.Doc
eqj.yghdsxcnbza.cn/80882.Doc
eqj.yghdsxcnbza.cn/64208.Doc
eqj.yghdsxcnbza.cn/04844.Doc
eqj.yghdsxcnbza.cn/55379.Doc
eqj.yghdsxcnbza.cn/86244.Doc
eqj.yghdsxcnbza.cn/26644.Doc
eqh.yghdsxcnbza.cn/82640.Doc
eqh.yghdsxcnbza.cn/40840.Doc
eqh.yghdsxcnbza.cn/28826.Doc
eqh.yghdsxcnbza.cn/06864.Doc
eqh.yghdsxcnbza.cn/79151.Doc
eqh.yghdsxcnbza.cn/64624.Doc
eqh.yghdsxcnbza.cn/40222.Doc
eqh.yghdsxcnbza.cn/51959.Doc
eqh.yghdsxcnbza.cn/15115.Doc
eqh.yghdsxcnbza.cn/66022.Doc
eqg.yghdsxcnbza.cn/71591.Doc
eqg.yghdsxcnbza.cn/46486.Doc
eqg.yghdsxcnbza.cn/24086.Doc
eqg.yghdsxcnbza.cn/60202.Doc
eqg.yghdsxcnbza.cn/40200.Doc
eqg.yghdsxcnbza.cn/46220.Doc
eqg.yghdsxcnbza.cn/46466.Doc
eqg.yghdsxcnbza.cn/26866.Doc
eqg.yghdsxcnbza.cn/60664.Doc
eqg.yghdsxcnbza.cn/08606.Doc
eqf.yghdsxcnbza.cn/42884.Doc
eqf.yghdsxcnbza.cn/46864.Doc
eqf.yghdsxcnbza.cn/84688.Doc
eqf.yghdsxcnbza.cn/22064.Doc
eqf.yghdsxcnbza.cn/02608.Doc
eqf.yghdsxcnbza.cn/22202.Doc
eqf.yghdsxcnbza.cn/42466.Doc
eqf.yghdsxcnbza.cn/22862.Doc
eqf.yghdsxcnbza.cn/22266.Doc
eqf.yghdsxcnbza.cn/80844.Doc
eqd.yghdsxcnbza.cn/20066.Doc
eqd.yghdsxcnbza.cn/06242.Doc
eqd.yghdsxcnbza.cn/80886.Doc
eqd.yghdsxcnbza.cn/60428.Doc
eqd.yghdsxcnbza.cn/06866.Doc
eqd.yghdsxcnbza.cn/20804.Doc
eqd.yghdsxcnbza.cn/66820.Doc
eqd.yghdsxcnbza.cn/02242.Doc
eqd.yghdsxcnbza.cn/46068.Doc
eqd.yghdsxcnbza.cn/84886.Doc
eqs.yghdsxcnbza.cn/17751.Doc
eqs.yghdsxcnbza.cn/06028.Doc
eqs.yghdsxcnbza.cn/64288.Doc
eqs.yghdsxcnbza.cn/66806.Doc
eqs.yghdsxcnbza.cn/00460.Doc
eqs.yghdsxcnbza.cn/82684.Doc
eqs.yghdsxcnbza.cn/64644.Doc
eqs.yghdsxcnbza.cn/26822.Doc
eqs.yghdsxcnbza.cn/60284.Doc
eqs.yghdsxcnbza.cn/04864.Doc
eqa.yghdsxcnbza.cn/46480.Doc
eqa.yghdsxcnbza.cn/66646.Doc
eqa.yghdsxcnbza.cn/82002.Doc
eqa.yghdsxcnbza.cn/06406.Doc
eqa.yghdsxcnbza.cn/60680.Doc
eqa.yghdsxcnbza.cn/66260.Doc
eqa.yghdsxcnbza.cn/39173.Doc
eqa.yghdsxcnbza.cn/35553.Doc
eqa.yghdsxcnbza.cn/26662.Doc
eqa.yghdsxcnbza.cn/68242.Doc
eqp.yghdsxcnbza.cn/22286.Doc
eqp.yghdsxcnbza.cn/40480.Doc
eqp.yghdsxcnbza.cn/44260.Doc
eqp.yghdsxcnbza.cn/88028.Doc
eqp.yghdsxcnbza.cn/68808.Doc
eqp.yghdsxcnbza.cn/60868.Doc
eqp.yghdsxcnbza.cn/64280.Doc
eqp.yghdsxcnbza.cn/20284.Doc
eqp.yghdsxcnbza.cn/08222.Doc
eqp.yghdsxcnbza.cn/91731.Doc
eqo.yghdsxcnbza.cn/48426.Doc
eqo.yghdsxcnbza.cn/40880.Doc
eqo.yghdsxcnbza.cn/42468.Doc
eqo.yghdsxcnbza.cn/44004.Doc
eqo.yghdsxcnbza.cn/75133.Doc
eqo.yghdsxcnbza.cn/00064.Doc
eqo.yghdsxcnbza.cn/86464.Doc
eqo.yghdsxcnbza.cn/82066.Doc
eqo.yghdsxcnbza.cn/26466.Doc
eqo.yghdsxcnbza.cn/42068.Doc
eqi.yghdsxcnbza.cn/00280.Doc
eqi.yghdsxcnbza.cn/26066.Doc
eqi.yghdsxcnbza.cn/82608.Doc
eqi.yghdsxcnbza.cn/42620.Doc
eqi.yghdsxcnbza.cn/02020.Doc
eqi.yghdsxcnbza.cn/20884.Doc
eqi.yghdsxcnbza.cn/62668.Doc
eqi.yghdsxcnbza.cn/88444.Doc
eqi.yghdsxcnbza.cn/84440.Doc
eqi.yghdsxcnbza.cn/86042.Doc
