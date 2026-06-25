Python itertools 高级模式
===============================

itertools 提供高效的迭代器工具, 所有函数均返回迭代器——惰性求值, 内存友好.

1. 无限迭代器: cycle / islice / tee
-------------------------------------

from itertools import cycle, islice, tee

# cycle 无限循环可迭代对象
counter = 0
for item in cycle(["红", "黄", "绿"]):
    print("循环信号灯:", item)
    counter += 1
    if counter >= 5:
        break

# islice 对迭代器做切片 (惰性)
# islice(iterable, start, stop, step)
data = range(100)
subset = islice(data, 10, 30, 2)  # 索引10到30, 步长2
print("islice 切片结果:", list(subset))

# tee 将一个迭代器克隆为 N 个独立副本
src = range(5)
a, b, c = tee(src, 3)            # 克隆出3个独立迭代器
print("tee 迭代器a:", list(a))    # [0,1,2,3,4]
print("tee 迭代器b:", list(b))    # [0,1,2,3,4]
print("tee 迭代器c:", list(c))    # [0,1,2,3,4]
# 注意: tee 会缓存数据, 如果原始迭代器很大且多个副本进度差异大, 会消耗内存

2. 组合数学: combinations / permutations / product
-----------------------------------------------------

from itertools import combinations, permutations, product

# combinations: 无放回组合, 元素不重复, 顺序无关
items = ["A", "B", "C"]
print("组合(2个):", list(combinations(items, 2)))
# [('A','B'), ('A','C'), ('B','C')]

# permutations: 排列, 顺序有关, 元素不重复
print("排列(2个):", list(permutations(items, 2)))
# [('A','B'), ('A','C'), ('B','A'), ('B','C'), ('C','A'), ('C','B')]

# product: 笛卡尔积, 相当于嵌套循环, repeat 参数允许重复
print("笛卡尔积:", list(product([0, 1], repeat=2)))
# [(0,0), (0,1), (1,0), (1,1)]

# 实战: 枚举所有可能的密码组合
digits = "0123456789"
# product(digits, repeat=4) 生成所有4位数字密码
for pin in islice(product(digits, repeat=4), 3):
    print("密码组合:", "".join(pin))

3. groupby — 分组聚合
-----------------------
groupby 要求输入已排序, 只对连续相同键分组.

from itertools import groupby

data = [("北京", 100), ("北京", 200), ("上海", 150), ("上海", 300), ("广州", 80)]
# 按城市分组前必须先排序
sorted_data = sorted(data, key=lambda x: x[0])

result = {}
for city, group in groupby(sorted_data, key=lambda x: x[0]):
    values = [item[1] for item in group]
    result[city] = {"总量": sum(values), "次数": len(values), "平均": sum(values)/len(values)}
print("groupby 分组统计:", result)

# groupby 返回的 group 是迭代器, 需要遍历前消费

4. accumulate — 累计运算
--------------------------

from itertools import accumulate
import operator

nums = [1, 2, 3, 4, 5]

# 默认累加
print("累计求和:", list(accumulate(nums)))          # [1, 3, 6, 10, 15]

# 累计乘积
print("累计乘积:", list(accumulate(nums, operator.mul)))  # [1, 2, 6, 24, 120]

# 自定义函数: 累计最大值
print("累计最大值:", list(accumulate(nums, max)))   # [1, 2, 3, 4, 5]

# 斐波那契数列
def fib():
    yield 0
    yield 1
    yield from accumulate(range(2, 20), lambda a, b: a + b)
# 注意: accumulate 的 func 接收 (上一次结果, 当前元素)

5. chain / chain.from_iterable — 展平
-----------------------------------------

from itertools import chain

# chain 连接多个可迭代对象
lists = [[1, 2], [3, 4], [5]]
flattened = chain.from_iterable(lists)  # 等价于 chain(*lists)
print("展平列表:", list(flattened))                # [1, 2, 3, 4, 5]

# chain 可混合不同类型
mixed = chain("ABC", [1, 2], (3,))
print("混合类型连接:", list(mixed))                # ['A','B','C',1,2,3]

6. zip_longest — 最长拉链
--------------------------

from itertools import zip_longest

names = ["Alice", "Bob", "Charlie"]
scores = [85, 92]
# 默认 zip 会在最短处截断, zip_longest 用 fillvalue 填充
zipped = zip_longest(names, scores, fillvalue="缺考")
print("zip_longest 结果:", list(zipped))
# [('Alice', 85), ('Bob', 92), ('Charlie', '缺考')]

7. takewhile / dropwhile — 条件筛选
--------------------------------------

from itertools import takewhile, dropwhile

nums = [2, 4, 6, 7, 8, 9, 10]

# takewhile: 拿取满足条件的元素, 遇到第一个不满足就停止
print("takewhile 偶数:", list(takewhile(lambda x: x % 2 == 0, nums)))
# [2, 4, 6]

# dropwhile: 丢弃满足条件的元素, 遇到第一个不满足就保留后续全部
print("dropwhile 偶数:", list(dropwhile(lambda x: x % 2 == 0, nums)))
# [7, 8, 9, 10]

8. pairwise (Python 3.10+) / batched (Python 3.12+)
------------------------------------------------------

from itertools import pairwise, batched

# pairwise: 返回连续重叠对 (s -> (s0,s1), (s1,s2), ...)
items = [1, 2, 3, 4, 5]
print("pairwise 相邻对:", list(pairwise(items)))
# [(1,2), (2,3), (3,4), (4,5)]
# 应用: 检查序列是否单调递增
print("是否单调递增:", all(a < b for a, b in pairwise(items)))  # True

# batched: 将迭代器分成指定大小的块
data_batches = list(batched(range(10), 3))
print("batched 分块:", data_batches)
# [(0,1,2), (3,4,5), (6,7,8), (9,)]

9. 自定义迭代器链式模式
-------------------------

def custom_iterator_chain(data):
    """链式组合多个 itertools 操作"""
    from itertools import filterfalse, compress, starmap
    # filterfalse: 保留不满足条件的元素
    return (
        data
        # 这里用函数组合展示模式, 实际需顺序调用
    )

# 典型流水线模式
def pipeline_example(nums):
    """数据处理流水线: 过滤 -> 变换 -> 分组 -> 聚合"""
    # 过滤正数
    positives = filter(lambda x: x > 0, nums)
    # 映射平方
    squares = map(lambda x: x ** 2, positives)
    # 按奇偶分组
    sorted_squares = sorted(squares, key=lambda x: x % 2)
    grouped = groupby(sorted_squares, key=lambda x: x % 2)
    result = {}
    for is_even, group in grouped:
        result["偶数" if is_even == 0 else "奇数"] = list(group)
    return result

print("流水线结果:", pipeline_example([-3, -2, 1, 2, 3, 4]))

# compress: 按选择器筛选
data = ["a", "b", "c", "d"]
selectors = [1, 0, 1, 0]
print("compress 筛选:", list(compress(data, selectors)))  # ['a', 'c']

# starmap: 用 * 解包参数
pairs = [(2, 3), (4, 5), (6, 7)]
print("starmap 求和:", list(starmap(lambda a, b: a + b, pairs)))
# [5, 9, 13]

总结: itertools 函数组合可构建声明式数据管道, 避免中间列表, 代码更简洁高效.

qgg.zhuzhoujiudazxL1517.cn/86888.Doc
qgg.zhuzhoujiudazxL1517.cn/84682.Doc
qgg.zhuzhoujiudazxL1517.cn/55595.Doc
qgg.zhuzhoujiudazxL1517.cn/04844.Doc
qgg.zhuzhoujiudazxL1517.cn/80448.Doc
qgg.zhuzhoujiudazxL1517.cn/60044.Doc
qgg.zhuzhoujiudazxL1517.cn/88080.Doc
qgg.zhuzhoujiudazxL1517.cn/28228.Doc
qgg.zhuzhoujiudazxL1517.cn/88062.Doc
qgg.zhuzhoujiudazxL1517.cn/00428.Doc
qgf.zhuzhoujiudazxL1517.cn/80228.Doc
qgf.zhuzhoujiudazxL1517.cn/88426.Doc
qgf.zhuzhoujiudazxL1517.cn/00242.Doc
qgf.zhuzhoujiudazxL1517.cn/00224.Doc
qgf.zhuzhoujiudazxL1517.cn/64882.Doc
qgf.zhuzhoujiudazxL1517.cn/44648.Doc
qgf.zhuzhoujiudazxL1517.cn/06804.Doc
qgf.zhuzhoujiudazxL1517.cn/93511.Doc
qgf.zhuzhoujiudazxL1517.cn/62282.Doc
qgf.zhuzhoujiudazxL1517.cn/48842.Doc
qgd.zhuzhoujiudazxL1517.cn/53313.Doc
qgd.zhuzhoujiudazxL1517.cn/68244.Doc
qgd.zhuzhoujiudazxL1517.cn/02460.Doc
qgd.zhuzhoujiudazxL1517.cn/44400.Doc
qgd.zhuzhoujiudazxL1517.cn/46260.Doc
qgd.zhuzhoujiudazxL1517.cn/28862.Doc
qgd.zhuzhoujiudazxL1517.cn/88844.Doc
qgd.zhuzhoujiudazxL1517.cn/42660.Doc
qgd.zhuzhoujiudazxL1517.cn/00442.Doc
qgd.zhuzhoujiudazxL1517.cn/88602.Doc
qgs.zhuzhoujiudazxL1517.cn/82608.Doc
qgs.zhuzhoujiudazxL1517.cn/59971.Doc
qgs.zhuzhoujiudazxL1517.cn/24480.Doc
qgs.zhuzhoujiudazxL1517.cn/60686.Doc
qgs.zhuzhoujiudazxL1517.cn/22008.Doc
qgs.zhuzhoujiudazxL1517.cn/84448.Doc
qgs.zhuzhoujiudazxL1517.cn/06688.Doc
qgs.zhuzhoujiudazxL1517.cn/42642.Doc
qgs.zhuzhoujiudazxL1517.cn/24486.Doc
qgs.zhuzhoujiudazxL1517.cn/80844.Doc
qga.zhuzhoujiudazxL1517.cn/28462.Doc
qga.zhuzhoujiudazxL1517.cn/48042.Doc
qga.zhuzhoujiudazxL1517.cn/88048.Doc
qga.zhuzhoujiudazxL1517.cn/44084.Doc
qga.zhuzhoujiudazxL1517.cn/46442.Doc
qga.zhuzhoujiudazxL1517.cn/42644.Doc
qga.zhuzhoujiudazxL1517.cn/08628.Doc
qga.zhuzhoujiudazxL1517.cn/20880.Doc
qga.zhuzhoujiudazxL1517.cn/42684.Doc
qga.zhuzhoujiudazxL1517.cn/00064.Doc
qgp.zhuzhoujiudazxL1517.cn/26802.Doc
qgp.zhuzhoujiudazxL1517.cn/04068.Doc
qgp.zhuzhoujiudazxL1517.cn/80468.Doc
qgp.zhuzhoujiudazxL1517.cn/42288.Doc
qgp.zhuzhoujiudazxL1517.cn/20086.Doc
qgp.zhuzhoujiudazxL1517.cn/22604.Doc
qgp.zhuzhoujiudazxL1517.cn/04802.Doc
qgp.zhuzhoujiudazxL1517.cn/42224.Doc
qgp.zhuzhoujiudazxL1517.cn/26868.Doc
qgp.zhuzhoujiudazxL1517.cn/64642.Doc
qgo.zhuzhoujiudazxL1517.cn/46466.Doc
qgo.zhuzhoujiudazxL1517.cn/28686.Doc
qgo.zhuzhoujiudazxL1517.cn/08222.Doc
qgo.zhuzhoujiudazxL1517.cn/26686.Doc
qgo.zhuzhoujiudazxL1517.cn/08804.Doc
qgo.zhuzhoujiudazxL1517.cn/84202.Doc
qgo.zhuzhoujiudazxL1517.cn/26404.Doc
qgo.zhuzhoujiudazxL1517.cn/86088.Doc
qgo.zhuzhoujiudazxL1517.cn/48608.Doc
qgo.zhuzhoujiudazxL1517.cn/04224.Doc
qgi.zhuzhoujiudazxL1517.cn/60822.Doc
qgi.zhuzhoujiudazxL1517.cn/02404.Doc
qgi.zhuzhoujiudazxL1517.cn/86244.Doc
qgi.zhuzhoujiudazxL1517.cn/48864.Doc
qgi.zhuzhoujiudazxL1517.cn/86220.Doc
qgi.zhuzhoujiudazxL1517.cn/51759.Doc
qgi.zhuzhoujiudazxL1517.cn/40804.Doc
qgi.zhuzhoujiudazxL1517.cn/22686.Doc
qgi.zhuzhoujiudazxL1517.cn/28868.Doc
qgi.zhuzhoujiudazxL1517.cn/88802.Doc
qgu.zhuzhoujiudazxL1517.cn/04624.Doc
qgu.zhuzhoujiudazxL1517.cn/02682.Doc
qgu.zhuzhoujiudazxL1517.cn/68204.Doc
qgu.zhuzhoujiudazxL1517.cn/86224.Doc
qgu.zhuzhoujiudazxL1517.cn/97791.Doc
qgu.zhuzhoujiudazxL1517.cn/53379.Doc
qgu.zhuzhoujiudazxL1517.cn/62264.Doc
qgu.zhuzhoujiudazxL1517.cn/64040.Doc
qgu.zhuzhoujiudazxL1517.cn/62840.Doc
qgu.zhuzhoujiudazxL1517.cn/40020.Doc
qgy.zhuzhoujiudazxL1517.cn/64686.Doc
qgy.zhuzhoujiudazxL1517.cn/46824.Doc
qgy.zhuzhoujiudazxL1517.cn/46068.Doc
qgy.zhuzhoujiudazxL1517.cn/62200.Doc
qgy.zhuzhoujiudazxL1517.cn/00208.Doc
qgy.zhuzhoujiudazxL1517.cn/64460.Doc
qgy.zhuzhoujiudazxL1517.cn/06442.Doc
qgy.zhuzhoujiudazxL1517.cn/42044.Doc
qgy.zhuzhoujiudazxL1517.cn/64688.Doc
qgy.zhuzhoujiudazxL1517.cn/86620.Doc
qgt.zhuzhoujiudazxL1517.cn/44242.Doc
qgt.zhuzhoujiudazxL1517.cn/26260.Doc
qgt.zhuzhoujiudazxL1517.cn/42040.Doc
qgt.zhuzhoujiudazxL1517.cn/22688.Doc
qgt.zhuzhoujiudazxL1517.cn/28460.Doc
qgt.zhuzhoujiudazxL1517.cn/88864.Doc
qgt.zhuzhoujiudazxL1517.cn/66206.Doc
qgt.zhuzhoujiudazxL1517.cn/40628.Doc
qgt.zhuzhoujiudazxL1517.cn/28480.Doc
qgt.zhuzhoujiudazxL1517.cn/48886.Doc
qgr.zhuzhoujiudazxL1517.cn/22420.Doc
qgr.zhuzhoujiudazxL1517.cn/04268.Doc
qgr.zhuzhoujiudazxL1517.cn/08000.Doc
qgr.zhuzhoujiudazxL1517.cn/20844.Doc
qgr.zhuzhoujiudazxL1517.cn/22268.Doc
qgr.zhuzhoujiudazxL1517.cn/86626.Doc
qgr.zhuzhoujiudazxL1517.cn/44462.Doc
qgr.zhuzhoujiudazxL1517.cn/20686.Doc
qgr.zhuzhoujiudazxL1517.cn/40622.Doc
qgr.zhuzhoujiudazxL1517.cn/08664.Doc
qge.zhuzhoujiudazxL1517.cn/48824.Doc
qge.zhuzhoujiudazxL1517.cn/39135.Doc
qge.zhuzhoujiudazxL1517.cn/24206.Doc
qge.zhuzhoujiudazxL1517.cn/06666.Doc
qge.zhuzhoujiudazxL1517.cn/44246.Doc
qge.zhuzhoujiudazxL1517.cn/82486.Doc
qge.zhuzhoujiudazxL1517.cn/64680.Doc
qge.zhuzhoujiudazxL1517.cn/84066.Doc
qge.zhuzhoujiudazxL1517.cn/22608.Doc
qge.zhuzhoujiudazxL1517.cn/86848.Doc
qgw.zhuzhoujiudazxL1517.cn/20684.Doc
qgw.zhuzhoujiudazxL1517.cn/06626.Doc
qgw.zhuzhoujiudazxL1517.cn/84880.Doc
qgw.zhuzhoujiudazxL1517.cn/02204.Doc
qgw.zhuzhoujiudazxL1517.cn/04400.Doc
qgw.zhuzhoujiudazxL1517.cn/08440.Doc
qgw.zhuzhoujiudazxL1517.cn/62644.Doc
qgw.zhuzhoujiudazxL1517.cn/20682.Doc
qgw.zhuzhoujiudazxL1517.cn/28262.Doc
qgw.zhuzhoujiudazxL1517.cn/37113.Doc
qgq.zhuzhoujiudazxL1517.cn/62664.Doc
qgq.zhuzhoujiudazxL1517.cn/28280.Doc
qgq.zhuzhoujiudazxL1517.cn/28622.Doc
qgq.zhuzhoujiudazxL1517.cn/88624.Doc
qgq.zhuzhoujiudazxL1517.cn/62400.Doc
qgq.zhuzhoujiudazxL1517.cn/84000.Doc
qgq.zhuzhoujiudazxL1517.cn/33715.Doc
qgq.zhuzhoujiudazxL1517.cn/60288.Doc
qgq.zhuzhoujiudazxL1517.cn/62682.Doc
qgq.zhuzhoujiudazxL1517.cn/04868.Doc
qfm.zhuzhoujiudazxL1517.cn/00822.Doc
qfm.zhuzhoujiudazxL1517.cn/28660.Doc
qfm.zhuzhoujiudazxL1517.cn/62222.Doc
qfm.zhuzhoujiudazxL1517.cn/02222.Doc
qfm.zhuzhoujiudazxL1517.cn/04268.Doc
qfm.zhuzhoujiudazxL1517.cn/82228.Doc
qfm.zhuzhoujiudazxL1517.cn/66862.Doc
qfm.zhuzhoujiudazxL1517.cn/48864.Doc
qfm.zhuzhoujiudazxL1517.cn/48266.Doc
qfm.zhuzhoujiudazxL1517.cn/24808.Doc
qfn.zhuzhoujiudazxL1517.cn/68864.Doc
qfn.zhuzhoujiudazxL1517.cn/02624.Doc
qfn.zhuzhoujiudazxL1517.cn/82886.Doc
qfn.zhuzhoujiudazxL1517.cn/86402.Doc
qfn.zhuzhoujiudazxL1517.cn/02668.Doc
qfn.zhuzhoujiudazxL1517.cn/66484.Doc
qfn.zhuzhoujiudazxL1517.cn/37991.Doc
qfn.zhuzhoujiudazxL1517.cn/20668.Doc
qfn.zhuzhoujiudazxL1517.cn/22068.Doc
qfn.zhuzhoujiudazxL1517.cn/64222.Doc
qfb.zhuzhoujiudazxL1517.cn/24206.Doc
qfb.zhuzhoujiudazxL1517.cn/86688.Doc
qfb.zhuzhoujiudazxL1517.cn/19553.Doc
qfb.zhuzhoujiudazxL1517.cn/82002.Doc
qfb.zhuzhoujiudazxL1517.cn/24060.Doc
qfb.zhuzhoujiudazxL1517.cn/26628.Doc
qfb.zhuzhoujiudazxL1517.cn/08284.Doc
qfb.zhuzhoujiudazxL1517.cn/62882.Doc
qfb.zhuzhoujiudazxL1517.cn/66488.Doc
qfb.zhuzhoujiudazxL1517.cn/68286.Doc
qfv.zhuzhoujiudazxL1517.cn/68444.Doc
qfv.zhuzhoujiudazxL1517.cn/82824.Doc
qfv.zhuzhoujiudazxL1517.cn/86620.Doc
qfv.zhuzhoujiudazxL1517.cn/75579.Doc
qfv.zhuzhoujiudazxL1517.cn/22844.Doc
qfv.zhuzhoujiudazxL1517.cn/24800.Doc
qfv.zhuzhoujiudazxL1517.cn/84086.Doc
qfv.zhuzhoujiudazxL1517.cn/80400.Doc
qfv.zhuzhoujiudazxL1517.cn/44640.Doc
qfv.zhuzhoujiudazxL1517.cn/40064.Doc
qfc.zhuzhoujiudazxL1517.cn/84222.Doc
qfc.zhuzhoujiudazxL1517.cn/20066.Doc
qfc.zhuzhoujiudazxL1517.cn/57311.Doc
qfc.zhuzhoujiudazxL1517.cn/86880.Doc
qfc.zhuzhoujiudazxL1517.cn/64668.Doc
qfc.zhuzhoujiudazxL1517.cn/88602.Doc
qfc.zhuzhoujiudazxL1517.cn/66648.Doc
qfc.zhuzhoujiudazxL1517.cn/46846.Doc
qfc.zhuzhoujiudazxL1517.cn/48002.Doc
qfc.zhuzhoujiudazxL1517.cn/22228.Doc
qfx.zhuzhoujiudazxL1517.cn/02444.Doc
qfx.zhuzhoujiudazxL1517.cn/06604.Doc
qfx.zhuzhoujiudazxL1517.cn/02260.Doc
qfx.zhuzhoujiudazxL1517.cn/46004.Doc
qfx.zhuzhoujiudazxL1517.cn/04844.Doc
qfx.zhuzhoujiudazxL1517.cn/64262.Doc
qfx.zhuzhoujiudazxL1517.cn/08022.Doc
qfx.zhuzhoujiudazxL1517.cn/02640.Doc
qfx.zhuzhoujiudazxL1517.cn/24686.Doc
qfx.zhuzhoujiudazxL1517.cn/28620.Doc
qfz.zhuzhoujiudazxL1517.cn/28020.Doc
qfz.zhuzhoujiudazxL1517.cn/11991.Doc
qfz.zhuzhoujiudazxL1517.cn/84884.Doc
qfz.zhuzhoujiudazxL1517.cn/60226.Doc
qfz.zhuzhoujiudazxL1517.cn/20228.Doc
qfz.zhuzhoujiudazxL1517.cn/48080.Doc
qfz.zhuzhoujiudazxL1517.cn/26446.Doc
qfz.zhuzhoujiudazxL1517.cn/60606.Doc
qfz.zhuzhoujiudazxL1517.cn/82042.Doc
qfz.zhuzhoujiudazxL1517.cn/42880.Doc
qfl.zhuzhoujiudazxL1517.cn/80442.Doc
qfl.zhuzhoujiudazxL1517.cn/40608.Doc
qfl.zhuzhoujiudazxL1517.cn/48402.Doc
qfl.zhuzhoujiudazxL1517.cn/00802.Doc
qfl.zhuzhoujiudazxL1517.cn/22446.Doc
qfl.zhuzhoujiudazxL1517.cn/17337.Doc
qfl.zhuzhoujiudazxL1517.cn/00202.Doc
qfl.zhuzhoujiudazxL1517.cn/46608.Doc
qfl.zhuzhoujiudazxL1517.cn/42080.Doc
qfl.zhuzhoujiudazxL1517.cn/44226.Doc
qfk.zhuzhoujiudazxL1517.cn/80402.Doc
qfk.zhuzhoujiudazxL1517.cn/06820.Doc
qfk.zhuzhoujiudazxL1517.cn/93971.Doc
qfk.zhuzhoujiudazxL1517.cn/68480.Doc
qfk.zhuzhoujiudazxL1517.cn/00266.Doc
qfk.zhuzhoujiudazxL1517.cn/82880.Doc
qfk.zhuzhoujiudazxL1517.cn/22668.Doc
qfk.zhuzhoujiudazxL1517.cn/24448.Doc
qfk.zhuzhoujiudazxL1517.cn/26662.Doc
qfk.zhuzhoujiudazxL1517.cn/86266.Doc
qfj.zhuzhoujiudazxL1517.cn/88840.Doc
qfj.zhuzhoujiudazxL1517.cn/24442.Doc
qfj.zhuzhoujiudazxL1517.cn/40064.Doc
qfj.zhuzhoujiudazxL1517.cn/64628.Doc
qfj.zhuzhoujiudazxL1517.cn/82662.Doc
qfj.zhuzhoujiudazxL1517.cn/26426.Doc
qfj.zhuzhoujiudazxL1517.cn/42826.Doc
qfj.zhuzhoujiudazxL1517.cn/24224.Doc
qfj.zhuzhoujiudazxL1517.cn/31573.Doc
qfj.zhuzhoujiudazxL1517.cn/79591.Doc
qfh.zhuzhoujiudazxL1517.cn/68088.Doc
qfh.zhuzhoujiudazxL1517.cn/60626.Doc
qfh.zhuzhoujiudazxL1517.cn/40862.Doc
qfh.zhuzhoujiudazxL1517.cn/06284.Doc
qfh.zhuzhoujiudazxL1517.cn/84428.Doc
qfh.zhuzhoujiudazxL1517.cn/48008.Doc
qfh.zhuzhoujiudazxL1517.cn/86044.Doc
qfh.zhuzhoujiudazxL1517.cn/62262.Doc
qfh.zhuzhoujiudazxL1517.cn/33159.Doc
qfh.zhuzhoujiudazxL1517.cn/71575.Doc
qfg.zhuzhoujiudazxL1517.cn/53517.Doc
qfg.zhuzhoujiudazxL1517.cn/62422.Doc
qfg.zhuzhoujiudazxL1517.cn/44666.Doc
qfg.zhuzhoujiudazxL1517.cn/24626.Doc
qfg.zhuzhoujiudazxL1517.cn/08206.Doc
qfg.zhuzhoujiudazxL1517.cn/08840.Doc
qfg.zhuzhoujiudazxL1517.cn/22482.Doc
qfg.zhuzhoujiudazxL1517.cn/00260.Doc
qfg.zhuzhoujiudazxL1517.cn/88624.Doc
qfg.zhuzhoujiudazxL1517.cn/31973.Doc
qff.zhuzhoujiudazxL1517.cn/40804.Doc
qff.zhuzhoujiudazxL1517.cn/44664.Doc
qff.zhuzhoujiudazxL1517.cn/46202.Doc
qff.zhuzhoujiudazxL1517.cn/20608.Doc
qff.zhuzhoujiudazxL1517.cn/08002.Doc
qff.zhuzhoujiudazxL1517.cn/82840.Doc
qff.zhuzhoujiudazxL1517.cn/62004.Doc
qff.zhuzhoujiudazxL1517.cn/68288.Doc
qff.zhuzhoujiudazxL1517.cn/80088.Doc
qff.zhuzhoujiudazxL1517.cn/86466.Doc
qfd.zhuzhoujiudazxL1517.cn/26600.Doc
qfd.zhuzhoujiudazxL1517.cn/22444.Doc
qfd.zhuzhoujiudazxL1517.cn/06402.Doc
qfd.zhuzhoujiudazxL1517.cn/66468.Doc
qfd.zhuzhoujiudazxL1517.cn/24808.Doc
qfd.zhuzhoujiudazxL1517.cn/13915.Doc
qfd.zhuzhoujiudazxL1517.cn/26080.Doc
qfd.zhuzhoujiudazxL1517.cn/40068.Doc
qfd.zhuzhoujiudazxL1517.cn/37117.Doc
qfd.zhuzhoujiudazxL1517.cn/04628.Doc
qfs.zhuzhoujiudazxL1517.cn/28086.Doc
qfs.zhuzhoujiudazxL1517.cn/46888.Doc
qfs.zhuzhoujiudazxL1517.cn/60424.Doc
qfs.zhuzhoujiudazxL1517.cn/64044.Doc
qfs.zhuzhoujiudazxL1517.cn/08488.Doc
qfs.zhuzhoujiudazxL1517.cn/31973.Doc
qfs.zhuzhoujiudazxL1517.cn/97533.Doc
qfs.zhuzhoujiudazxL1517.cn/86648.Doc
qfs.zhuzhoujiudazxL1517.cn/84260.Doc
qfs.zhuzhoujiudazxL1517.cn/62882.Doc
qfa.zhuzhoujiudazxL1517.cn/66228.Doc
qfa.zhuzhoujiudazxL1517.cn/00286.Doc
qfa.zhuzhoujiudazxL1517.cn/86000.Doc
qfa.zhuzhoujiudazxL1517.cn/20204.Doc
qfa.zhuzhoujiudazxL1517.cn/04604.Doc
qfa.zhuzhoujiudazxL1517.cn/22680.Doc
qfa.zhuzhoujiudazxL1517.cn/06622.Doc
qfa.zhuzhoujiudazxL1517.cn/68264.Doc
qfa.zhuzhoujiudazxL1517.cn/62082.Doc
qfa.zhuzhoujiudazxL1517.cn/42260.Doc
qfp.zhuzhoujiudazxL1517.cn/22282.Doc
qfp.zhuzhoujiudazxL1517.cn/86848.Doc
qfp.zhuzhoujiudazxL1517.cn/95751.Doc
qfp.zhuzhoujiudazxL1517.cn/28020.Doc
qfp.zhuzhoujiudazxL1517.cn/06046.Doc
qfp.zhuzhoujiudazxL1517.cn/15199.Doc
qfp.zhuzhoujiudazxL1517.cn/62624.Doc
qfp.zhuzhoujiudazxL1517.cn/20286.Doc
qfp.zhuzhoujiudazxL1517.cn/40800.Doc
qfp.zhuzhoujiudazxL1517.cn/26648.Doc
qfo.zhuzhoujiudazxL1517.cn/68480.Doc
qfo.zhuzhoujiudazxL1517.cn/84442.Doc
qfo.zhuzhoujiudazxL1517.cn/40080.Doc
qfo.zhuzhoujiudazxL1517.cn/40802.Doc
qfo.zhuzhoujiudazxL1517.cn/59995.Doc
qfo.zhuzhoujiudazxL1517.cn/04648.Doc
qfo.zhuzhoujiudazxL1517.cn/88404.Doc
qfo.zhuzhoujiudazxL1517.cn/86826.Doc
qfo.zhuzhoujiudazxL1517.cn/08648.Doc
qfo.zhuzhoujiudazxL1517.cn/04486.Doc
qfi.zhuzhoujiudazxL1517.cn/28620.Doc
qfi.zhuzhoujiudazxL1517.cn/35337.Doc
qfi.zhuzhoujiudazxL1517.cn/42420.Doc
qfi.zhuzhoujiudazxL1517.cn/06842.Doc
qfi.zhuzhoujiudazxL1517.cn/88602.Doc
qfi.zhuzhoujiudazxL1517.cn/46062.Doc
qfi.zhuzhoujiudazxL1517.cn/06442.Doc
qfi.zhuzhoujiudazxL1517.cn/64464.Doc
qfi.zhuzhoujiudazxL1517.cn/60240.Doc
qfi.zhuzhoujiudazxL1517.cn/84246.Doc
qfu.zhuzhoujiudazxL1517.cn/22268.Doc
qfu.zhuzhoujiudazxL1517.cn/22484.Doc
qfu.zhuzhoujiudazxL1517.cn/44064.Doc
qfu.zhuzhoujiudazxL1517.cn/00644.Doc
qfu.zhuzhoujiudazxL1517.cn/20622.Doc
qfu.zhuzhoujiudazxL1517.cn/46684.Doc
qfu.zhuzhoujiudazxL1517.cn/06008.Doc
qfu.zhuzhoujiudazxL1517.cn/06846.Doc
qfu.zhuzhoujiudazxL1517.cn/22048.Doc
qfu.zhuzhoujiudazxL1517.cn/37753.Doc
qfy.zhuzhoujiudazxL1517.cn/64680.Doc
qfy.zhuzhoujiudazxL1517.cn/68440.Doc
qfy.zhuzhoujiudazxL1517.cn/04866.Doc
qfy.zhuzhoujiudazxL1517.cn/88686.Doc
qfy.zhuzhoujiudazxL1517.cn/62282.Doc
qfy.zhuzhoujiudazxL1517.cn/02406.Doc
qfy.zhuzhoujiudazxL1517.cn/02222.Doc
qfy.zhuzhoujiudazxL1517.cn/42046.Doc
qfy.zhuzhoujiudazxL1517.cn/57135.Doc
qfy.zhuzhoujiudazxL1517.cn/24262.Doc
qft.zhuzhoujiudazxL1517.cn/64886.Doc
qft.zhuzhoujiudazxL1517.cn/20480.Doc
qft.zhuzhoujiudazxL1517.cn/02408.Doc
qft.zhuzhoujiudazxL1517.cn/48004.Doc
qft.zhuzhoujiudazxL1517.cn/68640.Doc
qft.zhuzhoujiudazxL1517.cn/00062.Doc
qft.zhuzhoujiudazxL1517.cn/68442.Doc
qft.zhuzhoujiudazxL1517.cn/48246.Doc
qft.zhuzhoujiudazxL1517.cn/82286.Doc
qft.zhuzhoujiudazxL1517.cn/60404.Doc
qfr.zhuzhoujiudazxL1517.cn/44268.Doc
qfr.zhuzhoujiudazxL1517.cn/46806.Doc
qfr.zhuzhoujiudazxL1517.cn/88202.Doc
qfr.zhuzhoujiudazxL1517.cn/84688.Doc
qfr.zhuzhoujiudazxL1517.cn/26640.Doc
qfr.zhuzhoujiudazxL1517.cn/44048.Doc
qfr.zhuzhoujiudazxL1517.cn/00824.Doc
qfr.zhuzhoujiudazxL1517.cn/00422.Doc
qfr.zhuzhoujiudazxL1517.cn/24820.Doc
qfr.zhuzhoujiudazxL1517.cn/80062.Doc
qfe.zhuzhoujiudazxL1517.cn/48084.Doc
qfe.zhuzhoujiudazxL1517.cn/64842.Doc
qfe.zhuzhoujiudazxL1517.cn/28422.Doc
qfe.zhuzhoujiudazxL1517.cn/06460.Doc
qfe.zhuzhoujiudazxL1517.cn/57153.Doc
qfe.zhuzhoujiudazxL1517.cn/22820.Doc
qfe.zhuzhoujiudazxL1517.cn/84826.Doc
qfe.zhuzhoujiudazxL1517.cn/48604.Doc
qfe.zhuzhoujiudazxL1517.cn/68462.Doc
qfe.zhuzhoujiudazxL1517.cn/40242.Doc
qfw.zhuzhoujiudazxL1517.cn/64202.Doc
qfw.zhuzhoujiudazxL1517.cn/42060.Doc
qfw.zhuzhoujiudazxL1517.cn/66808.Doc
qfw.zhuzhoujiudazxL1517.cn/08464.Doc
qfw.zhuzhoujiudazxL1517.cn/80022.Doc
qfw.zhuzhoujiudazxL1517.cn/28242.Doc
qfw.zhuzhoujiudazxL1517.cn/06242.Doc
qfw.zhuzhoujiudazxL1517.cn/24228.Doc
qfw.zhuzhoujiudazxL1517.cn/28800.Doc
qfw.zhuzhoujiudazxL1517.cn/88806.Doc
qfq.zhuzhoujiudazxL1517.cn/44046.Doc
qfq.zhuzhoujiudazxL1517.cn/04282.Doc
qfq.zhuzhoujiudazxL1517.cn/00646.Doc
qfq.zhuzhoujiudazxL1517.cn/06406.Doc
qfq.zhuzhoujiudazxL1517.cn/84826.Doc
qfq.zhuzhoujiudazxL1517.cn/28040.Doc
qfq.zhuzhoujiudazxL1517.cn/60224.Doc
qfq.zhuzhoujiudazxL1517.cn/60464.Doc
qfq.zhuzhoujiudazxL1517.cn/80680.Doc
qfq.zhuzhoujiudazxL1517.cn/26282.Doc
qdm.zhuzhoujiudazxL1517.cn/04244.Doc
qdm.zhuzhoujiudazxL1517.cn/97115.Doc
qdm.zhuzhoujiudazxL1517.cn/40022.Doc
qdm.zhuzhoujiudazxL1517.cn/00840.Doc
qdm.zhuzhoujiudazxL1517.cn/08680.Doc
qdm.zhuzhoujiudazxL1517.cn/26288.Doc
qdm.zhuzhoujiudazxL1517.cn/04028.Doc
qdm.zhuzhoujiudazxL1517.cn/51179.Doc
qdm.zhuzhoujiudazxL1517.cn/06842.Doc
qdm.zhuzhoujiudazxL1517.cn/22048.Doc
qdn.zhuzhoujiudazxL1517.cn/48882.Doc
qdn.zhuzhoujiudazxL1517.cn/20208.Doc
qdn.zhuzhoujiudazxL1517.cn/88064.Doc
qdn.zhuzhoujiudazxL1517.cn/84624.Doc
qdn.zhuzhoujiudazxL1517.cn/44462.Doc
qdn.zhuzhoujiudazxL1517.cn/06688.Doc
qdn.zhuzhoujiudazxL1517.cn/40402.Doc
qdn.zhuzhoujiudazxL1517.cn/08084.Doc
qdn.zhuzhoujiudazxL1517.cn/75731.Doc
qdn.zhuzhoujiudazxL1517.cn/77997.Doc
qdb.zhuzhoujiudazxL1517.cn/40848.Doc
qdb.zhuzhoujiudazxL1517.cn/08086.Doc
qdb.zhuzhoujiudazxL1517.cn/68884.Doc
qdb.zhuzhoujiudazxL1517.cn/04424.Doc
qdb.zhuzhoujiudazxL1517.cn/20440.Doc
qdb.zhuzhoujiudazxL1517.cn/04486.Doc
qdb.zhuzhoujiudazxL1517.cn/64244.Doc
qdb.zhuzhoujiudazxL1517.cn/86428.Doc
qdb.zhuzhoujiudazxL1517.cn/62860.Doc
qdb.zhuzhoujiudazxL1517.cn/86666.Doc
qdv.zhuzhoujiudazxL1517.cn/60840.Doc
qdv.zhuzhoujiudazxL1517.cn/28888.Doc
qdv.zhuzhoujiudazxL1517.cn/08828.Doc
qdv.zhuzhoujiudazxL1517.cn/68282.Doc
qdv.zhuzhoujiudazxL1517.cn/88428.Doc
qdv.zhuzhoujiudazxL1517.cn/02260.Doc
qdv.zhuzhoujiudazxL1517.cn/60408.Doc
qdv.zhuzhoujiudazxL1517.cn/64244.Doc
qdv.zhuzhoujiudazxL1517.cn/84824.Doc
qdv.zhuzhoujiudazxL1517.cn/46844.Doc
qdc.zhuzhoujiudazxL1517.cn/06880.Doc
qdc.zhuzhoujiudazxL1517.cn/02868.Doc
qdc.zhuzhoujiudazxL1517.cn/04626.Doc
qdc.zhuzhoujiudazxL1517.cn/02648.Doc
qdc.zhuzhoujiudazxL1517.cn/86240.Doc
qdc.zhuzhoujiudazxL1517.cn/02624.Doc
qdc.zhuzhoujiudazxL1517.cn/06202.Doc
qdc.zhuzhoujiudazxL1517.cn/3.Doc
qdc.zhuzhoujiudazxL1517.cn/28088.Doc
qdc.zhuzhoujiudazxL1517.cn/22648.Doc
qdx.zhuzhoujiudazxL1517.cn/84446.Doc
qdx.zhuzhoujiudazxL1517.cn/00602.Doc
qdx.zhuzhoujiudazxL1517.cn/80642.Doc
qdx.zhuzhoujiudazxL1517.cn/46288.Doc
qdx.zhuzhoujiudazxL1517.cn/79977.Doc
qdx.zhuzhoujiudazxL1517.cn/40262.Doc
qdx.zhuzhoujiudazxL1517.cn/46284.Doc
qdx.zhuzhoujiudazxL1517.cn/84428.Doc
qdx.zhuzhoujiudazxL1517.cn/66428.Doc
qdx.zhuzhoujiudazxL1517.cn/26482.Doc
qdz.zhuzhoujiudazxL1517.cn/64402.Doc
qdz.zhuzhoujiudazxL1517.cn/28684.Doc
qdz.zhuzhoujiudazxL1517.cn/80668.Doc
qdz.zhuzhoujiudazxL1517.cn/62262.Doc
qdz.zhuzhoujiudazxL1517.cn/68624.Doc
qdz.zhuzhoujiudazxL1517.cn/00880.Doc
qdz.zhuzhoujiudazxL1517.cn/28026.Doc
qdz.zhuzhoujiudazxL1517.cn/68060.Doc
qdz.zhuzhoujiudazxL1517.cn/64606.Doc
qdz.zhuzhoujiudazxL1517.cn/26808.Doc
qdl.zhuzhoujiudazxL1517.cn/00280.Doc
qdl.zhuzhoujiudazxL1517.cn/44080.Doc
qdl.zhuzhoujiudazxL1517.cn/00488.Doc
qdl.zhuzhoujiudazxL1517.cn/00626.Doc
qdl.zhuzhoujiudazxL1517.cn/28000.Doc
qdl.zhuzhoujiudazxL1517.cn/62826.Doc
qdl.zhuzhoujiudazxL1517.cn/48244.Doc
qdl.zhuzhoujiudazxL1517.cn/66686.Doc
qdl.zhuzhoujiudazxL1517.cn/11151.Doc
qdl.zhuzhoujiudazxL1517.cn/44042.Doc
