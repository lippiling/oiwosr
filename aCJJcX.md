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

fwr.0wp4aoe.cn/55159.Doc
fwr.0wp4aoe.cn/77735.Doc
fwr.0wp4aoe.cn/55579.Doc
fwr.0wp4aoe.cn/93917.Doc
fwr.0wp4aoe.cn/11559.Doc
fwr.0wp4aoe.cn/13915.Doc
fwr.0wp4aoe.cn/15795.Doc
fwr.0wp4aoe.cn/53395.Doc
fwr.0wp4aoe.cn/51319.Doc
fwr.0wp4aoe.cn/91973.Doc
fwe.0wp4aoe.cn/57939.Doc
fwe.0wp4aoe.cn/53511.Doc
fwe.0wp4aoe.cn/79331.Doc
fwe.0wp4aoe.cn/91733.Doc
fwe.0wp4aoe.cn/11131.Doc
fwe.0wp4aoe.cn/97957.Doc
fwe.0wp4aoe.cn/53511.Doc
fwe.0wp4aoe.cn/53933.Doc
fwe.0wp4aoe.cn/57999.Doc
fwe.0wp4aoe.cn/57975.Doc
fww.0wp4aoe.cn/73511.Doc
fww.0wp4aoe.cn/93513.Doc
fww.0wp4aoe.cn/73791.Doc
fww.0wp4aoe.cn/19797.Doc
fww.0wp4aoe.cn/15337.Doc
fww.0wp4aoe.cn/35577.Doc
fww.0wp4aoe.cn/51953.Doc
fww.0wp4aoe.cn/97357.Doc
fww.0wp4aoe.cn/97997.Doc
fww.0wp4aoe.cn/57553.Doc
fwq.0wp4aoe.cn/35555.Doc
fwq.0wp4aoe.cn/53937.Doc
fwq.0wp4aoe.cn/53539.Doc
fwq.0wp4aoe.cn/19793.Doc
fwq.0wp4aoe.cn/33191.Doc
fwq.0wp4aoe.cn/97591.Doc
fwq.0wp4aoe.cn/13979.Doc
fwq.0wp4aoe.cn/55957.Doc
fwq.0wp4aoe.cn/75573.Doc
fwq.0wp4aoe.cn/97791.Doc
fqm.0wp4aoe.cn/99731.Doc
fqm.0wp4aoe.cn/35353.Doc
fqm.0wp4aoe.cn/51511.Doc
fqm.0wp4aoe.cn/57339.Doc
fqm.0wp4aoe.cn/33573.Doc
fqm.0wp4aoe.cn/33793.Doc
fqm.0wp4aoe.cn/99591.Doc
fqm.0wp4aoe.cn/15377.Doc
fqm.0wp4aoe.cn/00644.Doc
fqm.0wp4aoe.cn/19919.Doc
fqn.0wp4aoe.cn/71397.Doc
fqn.0wp4aoe.cn/11979.Doc
fqn.0wp4aoe.cn/11719.Doc
fqn.0wp4aoe.cn/93113.Doc
fqn.0wp4aoe.cn/71197.Doc
fqn.0wp4aoe.cn/59157.Doc
fqn.0wp4aoe.cn/75711.Doc
fqn.0wp4aoe.cn/46464.Doc
fqn.0wp4aoe.cn/57339.Doc
fqn.0wp4aoe.cn/42666.Doc
fqb.0wp4aoe.cn/31179.Doc
fqb.0wp4aoe.cn/31757.Doc
fqb.0wp4aoe.cn/57311.Doc
fqb.0wp4aoe.cn/53957.Doc
fqb.0wp4aoe.cn/79351.Doc
fqb.0wp4aoe.cn/53175.Doc
fqb.0wp4aoe.cn/31737.Doc
fqb.0wp4aoe.cn/24622.Doc
fqb.0wp4aoe.cn/73955.Doc
fqb.0wp4aoe.cn/33153.Doc
fqv.0wp4aoe.cn/35375.Doc
fqv.0wp4aoe.cn/19139.Doc
fqv.0wp4aoe.cn/31153.Doc
fqv.0wp4aoe.cn/17717.Doc
fqv.0wp4aoe.cn/31591.Doc
fqv.0wp4aoe.cn/59793.Doc
fqv.0wp4aoe.cn/15159.Doc
fqv.0wp4aoe.cn/40646.Doc
fqv.0wp4aoe.cn/99331.Doc
fqv.0wp4aoe.cn/15975.Doc
fqc.0wp4aoe.cn/51355.Doc
fqc.0wp4aoe.cn/55199.Doc
fqc.0wp4aoe.cn/59937.Doc
fqc.0wp4aoe.cn/53139.Doc
fqc.0wp4aoe.cn/59593.Doc
fqc.0wp4aoe.cn/75715.Doc
fqc.0wp4aoe.cn/55733.Doc
fqc.0wp4aoe.cn/19775.Doc
fqc.0wp4aoe.cn/97373.Doc
fqc.0wp4aoe.cn/35997.Doc
fqx.0wp4aoe.cn/71313.Doc
fqx.0wp4aoe.cn/75191.Doc
fqx.0wp4aoe.cn/31177.Doc
fqx.0wp4aoe.cn/33317.Doc
fqx.0wp4aoe.cn/55135.Doc
fqx.0wp4aoe.cn/79915.Doc
fqx.0wp4aoe.cn/73191.Doc
fqx.0wp4aoe.cn/73951.Doc
fqx.0wp4aoe.cn/28066.Doc
fqx.0wp4aoe.cn/79535.Doc
fqz.0wp4aoe.cn/91951.Doc
fqz.0wp4aoe.cn/55995.Doc
fqz.0wp4aoe.cn/66408.Doc
fqz.0wp4aoe.cn/15717.Doc
fqz.0wp4aoe.cn/35955.Doc
fqz.0wp4aoe.cn/17935.Doc
fqz.0wp4aoe.cn/51739.Doc
fqz.0wp4aoe.cn/57199.Doc
fqz.0wp4aoe.cn/77715.Doc
fqz.0wp4aoe.cn/37917.Doc
fql.0wp4aoe.cn/95375.Doc
fql.0wp4aoe.cn/55999.Doc
fql.0wp4aoe.cn/59731.Doc
fql.0wp4aoe.cn/71731.Doc
fql.0wp4aoe.cn/91971.Doc
fql.0wp4aoe.cn/17519.Doc
fql.0wp4aoe.cn/35113.Doc
fql.0wp4aoe.cn/37119.Doc
fql.0wp4aoe.cn/55713.Doc
fql.0wp4aoe.cn/57533.Doc
fqk.0wp4aoe.cn/31577.Doc
fqk.0wp4aoe.cn/99335.Doc
fqk.0wp4aoe.cn/31797.Doc
fqk.0wp4aoe.cn/75793.Doc
fqk.0wp4aoe.cn/77737.Doc
fqk.0wp4aoe.cn/93511.Doc
fqk.0wp4aoe.cn/17539.Doc
fqk.0wp4aoe.cn/44628.Doc
fqk.0wp4aoe.cn/93935.Doc
fqk.0wp4aoe.cn/51199.Doc
fqj.0wp4aoe.cn/17357.Doc
fqj.0wp4aoe.cn/51395.Doc
fqj.0wp4aoe.cn/97797.Doc
fqj.0wp4aoe.cn/33115.Doc
fqj.0wp4aoe.cn/33395.Doc
fqj.0wp4aoe.cn/99511.Doc
fqj.0wp4aoe.cn/73973.Doc
fqj.0wp4aoe.cn/51799.Doc
fqj.0wp4aoe.cn/53719.Doc
fqj.0wp4aoe.cn/31153.Doc
fqh.0wp4aoe.cn/73973.Doc
fqh.0wp4aoe.cn/39353.Doc
fqh.0wp4aoe.cn/04204.Doc
fqh.0wp4aoe.cn/93737.Doc
fqh.0wp4aoe.cn/17537.Doc
fqh.0wp4aoe.cn/33935.Doc
fqh.0wp4aoe.cn/51579.Doc
fqh.0wp4aoe.cn/17959.Doc
fqh.0wp4aoe.cn/37757.Doc
fqh.0wp4aoe.cn/91595.Doc
fqg.0wp4aoe.cn/97597.Doc
fqg.0wp4aoe.cn/13555.Doc
fqg.0wp4aoe.cn/57519.Doc
fqg.0wp4aoe.cn/37517.Doc
fqg.0wp4aoe.cn/00204.Doc
fqg.0wp4aoe.cn/11979.Doc
fqg.0wp4aoe.cn/37791.Doc
fqg.0wp4aoe.cn/11339.Doc
fqg.0wp4aoe.cn/93551.Doc
fqg.0wp4aoe.cn/57973.Doc
fqf.0wp4aoe.cn/19991.Doc
fqf.0wp4aoe.cn/55175.Doc
fqf.0wp4aoe.cn/71117.Doc
fqf.0wp4aoe.cn/59557.Doc
fqf.0wp4aoe.cn/99599.Doc
fqf.0wp4aoe.cn/17755.Doc
fqf.0wp4aoe.cn/75737.Doc
fqf.0wp4aoe.cn/35971.Doc
fqf.0wp4aoe.cn/31519.Doc
fqf.0wp4aoe.cn/55953.Doc
fqd.0wp4aoe.cn/08440.Doc
fqd.0wp4aoe.cn/33739.Doc
fqd.0wp4aoe.cn/39115.Doc
fqd.0wp4aoe.cn/53135.Doc
fqd.0wp4aoe.cn/11593.Doc
fqd.0wp4aoe.cn/35333.Doc
fqd.0wp4aoe.cn/11175.Doc
fqd.0wp4aoe.cn/95739.Doc
fqd.0wp4aoe.cn/37195.Doc
fqd.0wp4aoe.cn/37773.Doc
fqs.0wp4aoe.cn/55377.Doc
fqs.0wp4aoe.cn/39991.Doc
fqs.0wp4aoe.cn/19199.Doc
fqs.0wp4aoe.cn/93353.Doc
fqs.0wp4aoe.cn/79599.Doc
fqs.0wp4aoe.cn/39991.Doc
fqs.0wp4aoe.cn/80662.Doc
fqs.0wp4aoe.cn/73171.Doc
fqs.0wp4aoe.cn/73717.Doc
fqs.0wp4aoe.cn/35355.Doc
fqa.0wp4aoe.cn/79975.Doc
fqa.0wp4aoe.cn/79775.Doc
fqa.0wp4aoe.cn/71157.Doc
fqa.0wp4aoe.cn/39515.Doc
fqa.0wp4aoe.cn/99535.Doc
fqa.0wp4aoe.cn/53533.Doc
fqa.0wp4aoe.cn/15573.Doc
fqa.0wp4aoe.cn/17537.Doc
fqa.0wp4aoe.cn/37115.Doc
fqa.0wp4aoe.cn/93395.Doc
fqp.0wp4aoe.cn/53999.Doc
fqp.0wp4aoe.cn/79171.Doc
fqp.0wp4aoe.cn/99379.Doc
fqp.0wp4aoe.cn/19195.Doc
fqp.0wp4aoe.cn/19197.Doc
fqp.0wp4aoe.cn/91139.Doc
fqp.0wp4aoe.cn/99791.Doc
fqp.0wp4aoe.cn/39757.Doc
fqp.0wp4aoe.cn/79757.Doc
fqp.0wp4aoe.cn/91335.Doc
fqo.0wp4aoe.cn/95117.Doc
fqo.0wp4aoe.cn/95197.Doc
fqo.0wp4aoe.cn/37155.Doc
fqo.0wp4aoe.cn/57973.Doc
fqo.0wp4aoe.cn/59599.Doc
fqo.0wp4aoe.cn/13117.Doc
fqo.0wp4aoe.cn/15517.Doc
fqo.0wp4aoe.cn/93539.Doc
fqo.0wp4aoe.cn/93999.Doc
fqo.0wp4aoe.cn/77917.Doc
fqi.0wp4aoe.cn/77371.Doc
fqi.0wp4aoe.cn/20408.Doc
fqi.0wp4aoe.cn/11511.Doc
fqi.0wp4aoe.cn/95933.Doc
fqi.0wp4aoe.cn/35931.Doc
fqi.0wp4aoe.cn/79397.Doc
fqi.0wp4aoe.cn/71733.Doc
fqi.0wp4aoe.cn/53577.Doc
fqi.0wp4aoe.cn/39117.Doc
fqi.0wp4aoe.cn/17997.Doc
fqu.0wp4aoe.cn/55731.Doc
fqu.0wp4aoe.cn/13397.Doc
fqu.0wp4aoe.cn/11357.Doc
fqu.0wp4aoe.cn/53315.Doc
fqu.0wp4aoe.cn/13397.Doc
fqu.0wp4aoe.cn/77951.Doc
fqu.0wp4aoe.cn/55373.Doc
fqu.0wp4aoe.cn/53379.Doc
fqu.0wp4aoe.cn/59797.Doc
fqu.0wp4aoe.cn/19193.Doc
fqy.0wp4aoe.cn/15917.Doc
fqy.0wp4aoe.cn/95337.Doc
fqy.0wp4aoe.cn/51595.Doc
fqy.0wp4aoe.cn/33975.Doc
fqy.0wp4aoe.cn/35557.Doc
fqy.0wp4aoe.cn/11159.Doc
fqy.0wp4aoe.cn/33755.Doc
fqy.0wp4aoe.cn/11771.Doc
fqy.0wp4aoe.cn/37775.Doc
fqy.0wp4aoe.cn/99733.Doc
fqt.0wp4aoe.cn/71777.Doc
fqt.0wp4aoe.cn/13333.Doc
fqt.0wp4aoe.cn/51799.Doc
fqt.0wp4aoe.cn/75911.Doc
fqt.0wp4aoe.cn/71953.Doc
fqt.0wp4aoe.cn/57351.Doc
fqt.0wp4aoe.cn/11355.Doc
fqt.0wp4aoe.cn/73353.Doc
fqt.0wp4aoe.cn/95575.Doc
fqt.0wp4aoe.cn/97597.Doc
fqr.0wp4aoe.cn/15915.Doc
fqr.0wp4aoe.cn/95357.Doc
fqr.0wp4aoe.cn/77391.Doc
fqr.0wp4aoe.cn/55551.Doc
fqr.0wp4aoe.cn/86002.Doc
fqr.0wp4aoe.cn/15717.Doc
fqr.0wp4aoe.cn/17915.Doc
fqr.0wp4aoe.cn/37977.Doc
fqr.0wp4aoe.cn/35173.Doc
fqr.0wp4aoe.cn/11399.Doc
fqe.0wp4aoe.cn/53533.Doc
fqe.0wp4aoe.cn/15759.Doc
fqe.0wp4aoe.cn/15759.Doc
fqe.0wp4aoe.cn/11593.Doc
fqe.0wp4aoe.cn/53131.Doc
fqe.0wp4aoe.cn/11731.Doc
fqe.0wp4aoe.cn/37939.Doc
fqe.0wp4aoe.cn/59711.Doc
fqe.0wp4aoe.cn/31795.Doc
fqe.0wp4aoe.cn/97371.Doc
fqw.0wp4aoe.cn/37779.Doc
fqw.0wp4aoe.cn/91171.Doc
fqw.0wp4aoe.cn/39759.Doc
fqw.0wp4aoe.cn/77937.Doc
fqw.0wp4aoe.cn/15973.Doc
fqw.0wp4aoe.cn/93731.Doc
fqw.0wp4aoe.cn/11979.Doc
fqw.0wp4aoe.cn/53531.Doc
fqw.0wp4aoe.cn/97757.Doc
fqw.0wp4aoe.cn/53331.Doc
fqq.0wp4aoe.cn/77397.Doc
fqq.0wp4aoe.cn/13599.Doc
fqq.0wp4aoe.cn/91519.Doc
fqq.0wp4aoe.cn/17315.Doc
fqq.0wp4aoe.cn/11331.Doc
fqq.0wp4aoe.cn/73139.Doc
fqq.0wp4aoe.cn/99373.Doc
fqq.0wp4aoe.cn/33539.Doc
fqq.0wp4aoe.cn/33191.Doc
fqq.0wp4aoe.cn/51517.Doc
dmm.0wp4aoe.cn/99539.Doc
dmm.0wp4aoe.cn/37531.Doc
dmm.0wp4aoe.cn/59397.Doc
dmm.0wp4aoe.cn/53313.Doc
dmm.0wp4aoe.cn/97199.Doc
dmm.0wp4aoe.cn/79337.Doc
dmm.0wp4aoe.cn/37799.Doc
dmm.0wp4aoe.cn/17317.Doc
dmm.0wp4aoe.cn/11931.Doc
dmm.0wp4aoe.cn/59555.Doc
dmn.0wp4aoe.cn/13559.Doc
dmn.0wp4aoe.cn/31959.Doc
dmn.0wp4aoe.cn/11991.Doc
dmn.0wp4aoe.cn/73951.Doc
dmn.0wp4aoe.cn/91117.Doc
dmn.0wp4aoe.cn/13513.Doc
dmn.0wp4aoe.cn/39115.Doc
dmn.0wp4aoe.cn/55117.Doc
dmn.0wp4aoe.cn/57135.Doc
dmn.0wp4aoe.cn/53317.Doc
dmb.0wp4aoe.cn/11537.Doc
dmb.0wp4aoe.cn/91175.Doc
dmb.0wp4aoe.cn/53153.Doc
dmb.0wp4aoe.cn/13371.Doc
dmb.0wp4aoe.cn/95917.Doc
dmb.0wp4aoe.cn/39735.Doc
dmb.0wp4aoe.cn/51915.Doc
dmb.0wp4aoe.cn/95713.Doc
dmb.0wp4aoe.cn/48466.Doc
dmb.0wp4aoe.cn/19151.Doc
dmv.0wp4aoe.cn/11391.Doc
dmv.0wp4aoe.cn/91955.Doc
dmv.0wp4aoe.cn/37337.Doc
dmv.0wp4aoe.cn/33577.Doc
dmv.0wp4aoe.cn/17731.Doc
dmv.0wp4aoe.cn/95735.Doc
dmv.0wp4aoe.cn/33975.Doc
dmv.0wp4aoe.cn/71933.Doc
dmv.0wp4aoe.cn/13771.Doc
dmv.0wp4aoe.cn/62408.Doc
dmc.0wp4aoe.cn/55115.Doc
dmc.0wp4aoe.cn/51753.Doc
dmc.0wp4aoe.cn/77519.Doc
dmc.0wp4aoe.cn/97353.Doc
dmc.0wp4aoe.cn/93715.Doc
dmc.0wp4aoe.cn/48064.Doc
dmc.0wp4aoe.cn/37515.Doc
dmc.0wp4aoe.cn/33193.Doc
dmc.0wp4aoe.cn/19513.Doc
dmc.0wp4aoe.cn/13359.Doc
dmx.0wp4aoe.cn/91917.Doc
dmx.0wp4aoe.cn/39979.Doc
dmx.0wp4aoe.cn/73999.Doc
dmx.0wp4aoe.cn/15519.Doc
dmx.0wp4aoe.cn/19913.Doc
dmx.0wp4aoe.cn/19597.Doc
dmx.0wp4aoe.cn/37795.Doc
dmx.0wp4aoe.cn/19793.Doc
dmx.0wp4aoe.cn/97977.Doc
dmx.0wp4aoe.cn/95377.Doc
dmz.0wp4aoe.cn/57913.Doc
dmz.0wp4aoe.cn/77979.Doc
dmz.0wp4aoe.cn/73513.Doc
dmz.0wp4aoe.cn/33951.Doc
dmz.0wp4aoe.cn/17593.Doc
dmz.0wp4aoe.cn/75399.Doc
dmz.0wp4aoe.cn/48060.Doc
dmz.0wp4aoe.cn/11317.Doc
dmz.0wp4aoe.cn/48086.Doc
dmz.0wp4aoe.cn/11199.Doc
dml.0wp4aoe.cn/28024.Doc
dml.0wp4aoe.cn/35715.Doc
dml.0wp4aoe.cn/59371.Doc
dml.0wp4aoe.cn/73971.Doc
dml.0wp4aoe.cn/71333.Doc
dml.0wp4aoe.cn/77371.Doc
dml.0wp4aoe.cn/37377.Doc
dml.0wp4aoe.cn/73531.Doc
dml.0wp4aoe.cn/57993.Doc
dml.0wp4aoe.cn/06046.Doc
dmk.0wp4aoe.cn/19175.Doc
dmk.0wp4aoe.cn/53151.Doc
dmk.0wp4aoe.cn/11177.Doc
dmk.0wp4aoe.cn/37135.Doc
dmk.0wp4aoe.cn/31331.Doc
dmk.0wp4aoe.cn/39775.Doc
dmk.0wp4aoe.cn/99375.Doc
dmk.0wp4aoe.cn/84626.Doc
dmk.0wp4aoe.cn/95153.Doc
dmk.0wp4aoe.cn/39711.Doc
dmj.0wp4aoe.cn/19379.Doc
dmj.0wp4aoe.cn/55179.Doc
dmj.0wp4aoe.cn/55591.Doc
dmj.0wp4aoe.cn/51717.Doc
dmj.0wp4aoe.cn/93335.Doc
dmj.0wp4aoe.cn/79357.Doc
dmj.0wp4aoe.cn/35157.Doc
dmj.0wp4aoe.cn/71151.Doc
dmj.0wp4aoe.cn/02288.Doc
dmj.0wp4aoe.cn/33391.Doc
dmh.0wp4aoe.cn/31117.Doc
dmh.0wp4aoe.cn/40428.Doc
dmh.0wp4aoe.cn/59797.Doc
dmh.0wp4aoe.cn/99919.Doc
dmh.0wp4aoe.cn/77533.Doc
dmh.0wp4aoe.cn/35113.Doc
dmh.0wp4aoe.cn/55713.Doc
dmh.0wp4aoe.cn/15713.Doc
dmh.0wp4aoe.cn/31779.Doc
dmh.0wp4aoe.cn/35337.Doc
dmg.0wp4aoe.cn/99377.Doc
dmg.0wp4aoe.cn/93931.Doc
dmg.0wp4aoe.cn/13573.Doc
dmg.0wp4aoe.cn/35551.Doc
dmg.0wp4aoe.cn/99737.Doc
dmg.0wp4aoe.cn/39533.Doc
dmg.0wp4aoe.cn/35393.Doc
dmg.0wp4aoe.cn/53199.Doc
dmg.0wp4aoe.cn/99537.Doc
dmg.0wp4aoe.cn/55751.Doc
dmf.0wp4aoe.cn/37713.Doc
dmf.0wp4aoe.cn/93337.Doc
dmf.0wp4aoe.cn/15579.Doc
dmf.0wp4aoe.cn/77759.Doc
dmf.0wp4aoe.cn/06082.Doc
dmf.0wp4aoe.cn/79935.Doc
dmf.0wp4aoe.cn/11973.Doc
dmf.0wp4aoe.cn/31999.Doc
dmf.0wp4aoe.cn/95199.Doc
dmf.0wp4aoe.cn/19573.Doc
dmd.0wp4aoe.cn/88468.Doc
dmd.0wp4aoe.cn/39939.Doc
dmd.0wp4aoe.cn/91317.Doc
dmd.0wp4aoe.cn/71795.Doc
dmd.0wp4aoe.cn/28066.Doc
dmd.0wp4aoe.cn/59333.Doc
dmd.0wp4aoe.cn/73779.Doc
dmd.0wp4aoe.cn/59159.Doc
dmd.0wp4aoe.cn/91595.Doc
dmd.0wp4aoe.cn/95353.Doc
dms.0wp4aoe.cn/11351.Doc
dms.0wp4aoe.cn/17557.Doc
dms.0wp4aoe.cn/71531.Doc
dms.0wp4aoe.cn/59153.Doc
dms.0wp4aoe.cn/31731.Doc
dms.0wp4aoe.cn/95793.Doc
dms.0wp4aoe.cn/91357.Doc
dms.0wp4aoe.cn/53991.Doc
dms.0wp4aoe.cn/06808.Doc
dms.0wp4aoe.cn/73931.Doc
dma.0wp4aoe.cn/22662.Doc
dma.0wp4aoe.cn/71155.Doc
dma.0wp4aoe.cn/39755.Doc
dma.0wp4aoe.cn/13335.Doc
dma.0wp4aoe.cn/80806.Doc
dma.0wp4aoe.cn/02242.Doc
dma.0wp4aoe.cn/57599.Doc
dma.0wp4aoe.cn/95975.Doc
dma.0wp4aoe.cn/95975.Doc
dma.0wp4aoe.cn/31155.Doc
dmp.0wp4aoe.cn/28622.Doc
dmp.0wp4aoe.cn/79397.Doc
dmp.0wp4aoe.cn/79775.Doc
dmp.0wp4aoe.cn/75959.Doc
dmp.0wp4aoe.cn/93137.Doc
dmp.0wp4aoe.cn/35951.Doc
dmp.0wp4aoe.cn/95937.Doc
dmp.0wp4aoe.cn/5.Doc
dmp.0wp4aoe.cn/73735.Doc
dmp.0wp4aoe.cn/79359.Doc
dmo.0wp4aoe.cn/37951.Doc
dmo.0wp4aoe.cn/55375.Doc
dmo.0wp4aoe.cn/59117.Doc
dmo.0wp4aoe.cn/13915.Doc
dmo.0wp4aoe.cn/95179.Doc
dmo.0wp4aoe.cn/53311.Doc
dmo.0wp4aoe.cn/33999.Doc
dmo.0wp4aoe.cn/53311.Doc
dmo.0wp4aoe.cn/57195.Doc
dmo.0wp4aoe.cn/71559.Doc
dmi.0wp4aoe.cn/37739.Doc
dmi.0wp4aoe.cn/55559.Doc
dmi.0wp4aoe.cn/33957.Doc
dmi.0wp4aoe.cn/31399.Doc
dmi.0wp4aoe.cn/59935.Doc
dmi.0wp4aoe.cn/17395.Doc
dmi.0wp4aoe.cn/37173.Doc
dmi.0wp4aoe.cn/77715.Doc
dmi.0wp4aoe.cn/53319.Doc
dmi.0wp4aoe.cn/73137.Doc
