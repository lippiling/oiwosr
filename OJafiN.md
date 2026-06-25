Python operator 模块
==========================

operator 提供函数形式的运算符, 可作为 key 函数传递给 map/sorted/groupby 等。

1. itemgetter — 按索引或键取值
-----------------------------------

from operator import itemgetter

# 单个键: 返回元素本身
data = [("Alice", 25), ("Bob", 30), ("Charlie", 20)]
get_age = itemgetter(1)              # 取元组第二个元素
sorted_by_age = sorted(data, key=get_age)
print("按年龄排序:", sorted_by_age)

# 多个键: 返回元组
get_name_and_age = itemgetter(0, 1)
for person in data:
    print("姓名与年龄:", get_name_and_age(person))

# 字典取值
students = [{"name": "张三", "score": 90}, {"name": "李四", "score": 85}]
get_score = itemgetter("score")
print("最高分:", max(students, key=get_score))

# 多维索引: itemgetter 支持嵌套取值 ? 不支持, 需自行组合
# 但可结合 list comprehension 使用
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
get_first_col = itemgetter(0)
print("第一列:", [get_first_col(row) for row in matrix])  # [1, 4, 7]

2. attrgetter — 按属性取值
-----------------------------

from operator import attrgetter

class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    def __repr__(self):
        return f"{self.name}({self.age})"

people = [Person("小明", 18), Person("小红", 22), Person("小刚", 16)]
get_age = attrgetter("age")
print("最年轻的人:", min(people, key=get_age))        # 小刚(16)

# 多属性排序: 先按 age 再按 name
sorted_people = sorted(people, key=attrgetter("age", "name"))
print("多属性排序:", sorted_people)

# 链式属性访问: attrgetter("a.b.c")
class Address:
    def __init__(self, city):
        self.city = city

class Employee:
    def __init__(self, name, address):
        self.name = name
        self.address = address

emps = [Employee("张三", Address("北京")), Employee("李四", Address("上海"))]
get_city = attrgetter("address.city")
print("所有城市:", [get_city(e) for e in emps])       # ['北京', '上海']

3. methodcaller — 调用方法
-------------------------------

from operator import methodcaller

words = ["hello", "world", "python"]
to_upper = methodcaller("upper")                    # 调用 str.upper()
print("大写转换:", [to_upper(w) for w in words])     # ['HELLO', 'WORLD', 'PYTHON']

# 带参数的方法调用: str.startswith(prefix)
start_with_p = methodcaller("startswith", "p")
filtered = [w for w in words if start_with_p(w)]
print("以p开头:", filtered)                           # ['python']

# 带关键字参数: str.split(sep=None, maxsplit=-1)
csv_data = ["a,b,c", "d,e,f"]
splitter = methodcaller("split", ",")
print("CSV 解析:", [splitter(line) for line in csv_data])
# [['a','b','c'], ['d','e','f']]

4. 算术和逻辑操作: add / sub / mul / truth / not_
------------------------------------------------------

from operator import add, sub, mul, truediv, truth, not_

# 配合 reduce 使用
from functools import reduce
numbers = [1, 2, 3, 4, 5]
print("累加:", reduce(add, numbers))                 # 15
print("累乘:", reduce(mul, numbers))                 # 120

# 计算差值序列 (相邻元素差)
diffs = list(map(sub, numbers[1:], numbers[:-1]))
print("相邻差值:", diffs)                             # [1, 1, 1, 1, 1]

# truth / not_ — 布尔判断
items = [0, 1, "", "hello", [], [1]]
print("truth 测试:", [truth(x) for x in items])      # [False, True, False, True, False, True]
print("not_ 测试:", [not_(x) for x in items])         # [True, False, True, False, True, False]

# 实战: 统计真值数量
# 等价于 sum(1 for x in items if x)
true_count = sum(truth(x) for x in items)
print("真值数量:", true_count)                        # 3

5. inplace 运算符
--------------------

from operator import iadd, iadd as iadd_a  # 导入别名示例

# inplace 运算符会尝试修改第一个参数, 否则回退到普通运算
a_list = [1, 2, 3]
b_list = [4, 5]
result = iadd(a_list, b_list)               # 相当于 a_list += b_list
print("inplace 相加:", result)               # [1, 2, 3, 4, 5]
print("原列表是否被修改:", a_list is result)  # True (列表可变)

# 不可变对象会回退
x = 10
y = 5
print("不可变对象:", iadd(x, y))              # 15 (x 不变)

from operator import isub, imul, itruediv
a = 20
print("inplace 减法:", isub(a, 5))           # 15
print("inplace 乘法:", imul(a, 2))           # 40
print("inplace 除法:", itruediv(a, 3))       # 6.666...

# 使用场景: 在函数式编程中累加状态
from functools import reduce
nums = [1, 2, 3, 4, 5]
# 用 iadd 模拟 +=, 注意不可变对象回退
sum_via_iadd = reduce(iadd, nums, 0)
print("reduce + iadd 求和:", sum_via_iadd)   # 15

6. 运算符重载辅助
--------------------

import operator

# Python 运算符与 operator 函数的映射表:
# +  → add
# -  → sub
# *  → mul
# /  → truediv
# // → floordiv
# %  → mod
# ** → pow
# << → lshift
# >> → rshift
# &  → and_ (注意: 避免与 and 关键字冲突)
# |  → or_
# ^  → xor
# ~  → invert
# <  → lt
# <= → le
# == → eq
# != → ne
# >  → gt
# >= → ge

# 动态运算符选择: 根据配置选择比较方式
def flexible_sort(data, op_name, value):
    """根据操作符名称筛选数据"""
    ops = {
        "gt": operator.gt,       # 大于
        "ge": operator.ge,       # 大于等于
        "lt": operator.lt,       # 小于
        "le": operator.le,       # 小于等于
        "eq": operator.eq,       # 等于
        "ne": operator.ne,       # 不等于
    }
    compare = ops[op_name]
    return [x for x in data if compare(x, value)]

nums = [1, 5, 8, 3, 9, 2]
print("大于5:", flexible_sort(nums, "gt", 5))     # [8, 9]
print("小于等于3:", flexible_sort(nums, "le", 3))  # [1, 3, 2]

7. 与 itertools.groupby 和 sorted 结合
-------------------------------------------

from itertools import groupby
from operator import itemgetter, attrgetter

# 场景: 对交易记录按日期和类型分组统计
transactions = [
    ("2024-01-01", "收入", 100),
    ("2024-01-01", "支出", 50),
    ("2024-01-01", "收入", 200),
    ("2024-01-02", "支出", 30),
    ("2024-01-02", "收入", 150),
]

# 第一步: 按日期排序
sorted_by_date = sorted(transactions, key=itemgetter(0))
# 第二步: 按日期分组
for date, group in groupby(sorted_by_date, key=itemgetter(0)):
    group_list = list(group)
    total = sum(item[2] for item in group_list)
    print(f"日期 {date}: 交易 {len(group_list)} 笔, 金额合计 {total}")

# 更复杂的场景: 先按日期再按类型排序
sorted_multi = sorted(transactions, key=itemgetter(0, 1))
for (date, type_), group in groupby(sorted_multi, key=itemgetter(0, 1)):
    amounts = [item[2] for item in group]
    print(f"{date} {type_}: {sum(amounts)} 元")

# operator 搭配 max/min 的 key 参数
max_income = max(transactions, key=itemgetter(2))
print("最大金额交易:", max_income)                 # ("2024-01-01", "收入", 200)

# 数据去重: 保留首次出现的记录
from collections import OrderedDict
unique_by_date = list(OrderedDict((itemgetter(0)(t), t) for t in transactions).values())
print("去重后(按日期):", unique_by_date)

总结: operator 让排序/分组/筛选代码更简洁, 避免编写大量 lambda 表达式, 同时性能略优.

equ.yghdsxcnbza.cn/44246.Doc
equ.yghdsxcnbza.cn/66862.Doc
equ.yghdsxcnbza.cn/08642.Doc
equ.yghdsxcnbza.cn/02668.Doc
equ.yghdsxcnbza.cn/64606.Doc
equ.yghdsxcnbza.cn/62440.Doc
equ.yghdsxcnbza.cn/82802.Doc
equ.yghdsxcnbza.cn/20802.Doc
equ.yghdsxcnbza.cn/57195.Doc
equ.yghdsxcnbza.cn/60602.Doc
eqy.yghdsxcnbza.cn/13177.Doc
eqy.yghdsxcnbza.cn/44268.Doc
eqy.yghdsxcnbza.cn/84062.Doc
eqy.yghdsxcnbza.cn/93171.Doc
eqy.yghdsxcnbza.cn/62064.Doc
eqy.yghdsxcnbza.cn/79911.Doc
eqy.yghdsxcnbza.cn/80084.Doc
eqy.yghdsxcnbza.cn/84828.Doc
eqy.yghdsxcnbza.cn/64222.Doc
eqy.yghdsxcnbza.cn/99391.Doc
eqt.yghdsxcnbza.cn/08446.Doc
eqt.yghdsxcnbza.cn/99199.Doc
eqt.yghdsxcnbza.cn/04846.Doc
eqt.yghdsxcnbza.cn/66086.Doc
eqt.yghdsxcnbza.cn/06688.Doc
eqt.yghdsxcnbza.cn/66004.Doc
eqt.yghdsxcnbza.cn/39395.Doc
eqt.yghdsxcnbza.cn/46240.Doc
eqt.yghdsxcnbza.cn/44264.Doc
eqt.yghdsxcnbza.cn/31735.Doc
eqr.yghdsxcnbza.cn/24248.Doc
eqr.yghdsxcnbza.cn/17975.Doc
eqr.yghdsxcnbza.cn/00066.Doc
eqr.yghdsxcnbza.cn/40684.Doc
eqr.yghdsxcnbza.cn/06402.Doc
eqr.yghdsxcnbza.cn/60804.Doc
eqr.yghdsxcnbza.cn/82242.Doc
eqr.yghdsxcnbza.cn/84480.Doc
eqr.yghdsxcnbza.cn/88644.Doc
eqr.yghdsxcnbza.cn/64488.Doc
eqe.yghdsxcnbza.cn/82222.Doc
eqe.yghdsxcnbza.cn/40428.Doc
eqe.yghdsxcnbza.cn/44422.Doc
eqe.yghdsxcnbza.cn/42086.Doc
eqe.yghdsxcnbza.cn/20246.Doc
eqe.yghdsxcnbza.cn/44668.Doc
eqe.yghdsxcnbza.cn/22248.Doc
eqe.yghdsxcnbza.cn/02246.Doc
eqe.yghdsxcnbza.cn/66048.Doc
eqe.yghdsxcnbza.cn/06042.Doc
eqw.yghdsxcnbza.cn/08862.Doc
eqw.yghdsxcnbza.cn/62882.Doc
eqw.yghdsxcnbza.cn/80026.Doc
eqw.yghdsxcnbza.cn/62224.Doc
eqw.yghdsxcnbza.cn/46642.Doc
eqw.yghdsxcnbza.cn/46846.Doc
eqw.yghdsxcnbza.cn/02228.Doc
eqw.yghdsxcnbza.cn/64246.Doc
eqw.yghdsxcnbza.cn/08888.Doc
eqw.yghdsxcnbza.cn/55397.Doc
eqq.yghdsxcnbza.cn/97539.Doc
eqq.yghdsxcnbza.cn/62884.Doc
eqq.yghdsxcnbza.cn/86428.Doc
eqq.yghdsxcnbza.cn/17557.Doc
eqq.yghdsxcnbza.cn/24646.Doc
eqq.yghdsxcnbza.cn/22888.Doc
eqq.yghdsxcnbza.cn/42226.Doc
eqq.yghdsxcnbza.cn/04080.Doc
eqq.yghdsxcnbza.cn/40066.Doc
eqq.yghdsxcnbza.cn/44888.Doc
wmm.yghdsxcnbza.cn/42662.Doc
wmm.yghdsxcnbza.cn/02846.Doc
wmm.yghdsxcnbza.cn/82042.Doc
wmm.yghdsxcnbza.cn/20680.Doc
wmm.yghdsxcnbza.cn/44224.Doc
wmm.yghdsxcnbza.cn/95533.Doc
wmm.yghdsxcnbza.cn/02066.Doc
wmm.yghdsxcnbza.cn/46262.Doc
wmm.yghdsxcnbza.cn/62882.Doc
wmm.yghdsxcnbza.cn/40042.Doc
wmn.yghdsxcnbza.cn/04046.Doc
wmn.yghdsxcnbza.cn/40222.Doc
wmn.yghdsxcnbza.cn/22488.Doc
wmn.yghdsxcnbza.cn/68286.Doc
wmn.yghdsxcnbza.cn/82082.Doc
wmn.yghdsxcnbza.cn/08806.Doc
wmn.yghdsxcnbza.cn/22620.Doc
wmn.yghdsxcnbza.cn/22244.Doc
wmn.yghdsxcnbza.cn/26864.Doc
wmn.yghdsxcnbza.cn/66208.Doc
wmb.yghdsxcnbza.cn/66622.Doc
wmb.yghdsxcnbza.cn/84066.Doc
wmb.yghdsxcnbza.cn/04886.Doc
wmb.yghdsxcnbza.cn/22486.Doc
wmb.yghdsxcnbza.cn/60064.Doc
wmb.yghdsxcnbza.cn/06282.Doc
wmb.yghdsxcnbza.cn/46406.Doc
wmb.yghdsxcnbza.cn/42088.Doc
wmb.yghdsxcnbza.cn/44428.Doc
wmb.yghdsxcnbza.cn/13533.Doc
wmv.yghdsxcnbza.cn/82484.Doc
wmv.yghdsxcnbza.cn/88824.Doc
wmv.yghdsxcnbza.cn/22006.Doc
wmv.yghdsxcnbza.cn/20862.Doc
wmv.yghdsxcnbza.cn/80800.Doc
wmv.yghdsxcnbza.cn/44422.Doc
wmv.yghdsxcnbza.cn/86828.Doc
wmv.yghdsxcnbza.cn/11397.Doc
wmv.yghdsxcnbza.cn/00482.Doc
wmv.yghdsxcnbza.cn/42008.Doc
wmc.yghdsxcnbza.cn/08660.Doc
wmc.yghdsxcnbza.cn/86202.Doc
wmc.yghdsxcnbza.cn/62802.Doc
wmc.yghdsxcnbza.cn/80806.Doc
wmc.yghdsxcnbza.cn/84404.Doc
wmc.yghdsxcnbza.cn/82404.Doc
wmc.yghdsxcnbza.cn/40424.Doc
wmc.yghdsxcnbza.cn/66248.Doc
wmc.yghdsxcnbza.cn/82848.Doc
wmc.yghdsxcnbza.cn/66660.Doc
wmx.yghdsxcnbza.cn/88268.Doc
wmx.yghdsxcnbza.cn/24220.Doc
wmx.yghdsxcnbza.cn/04004.Doc
wmx.yghdsxcnbza.cn/44444.Doc
wmx.yghdsxcnbza.cn/68848.Doc
wmx.yghdsxcnbza.cn/44446.Doc
wmx.yghdsxcnbza.cn/40424.Doc
wmx.yghdsxcnbza.cn/82880.Doc
wmx.yghdsxcnbza.cn/86600.Doc
wmx.yghdsxcnbza.cn/04248.Doc
wmz.yghdsxcnbza.cn/40486.Doc
wmz.yghdsxcnbza.cn/02284.Doc
wmz.yghdsxcnbza.cn/62226.Doc
wmz.yghdsxcnbza.cn/68000.Doc
wmz.yghdsxcnbza.cn/80262.Doc
wmz.yghdsxcnbza.cn/68688.Doc
wmz.yghdsxcnbza.cn/31319.Doc
wmz.yghdsxcnbza.cn/06668.Doc
wmz.yghdsxcnbza.cn/02882.Doc
wmz.yghdsxcnbza.cn/53579.Doc
wml.yghdsxcnbza.cn/84648.Doc
wml.yghdsxcnbza.cn/82484.Doc
wml.yghdsxcnbza.cn/46440.Doc
wml.yghdsxcnbza.cn/66866.Doc
wml.yghdsxcnbza.cn/26066.Doc
wml.yghdsxcnbza.cn/60680.Doc
wml.yghdsxcnbza.cn/66466.Doc
wml.yghdsxcnbza.cn/22284.Doc
wml.yghdsxcnbza.cn/64282.Doc
wml.yghdsxcnbza.cn/82604.Doc
wmk.yghdsxcnbza.cn/02246.Doc
wmk.yghdsxcnbza.cn/17517.Doc
wmk.yghdsxcnbza.cn/24666.Doc
wmk.yghdsxcnbza.cn/46800.Doc
wmk.yghdsxcnbza.cn/06268.Doc
wmk.yghdsxcnbza.cn/88644.Doc
wmk.yghdsxcnbza.cn/48280.Doc
wmk.yghdsxcnbza.cn/84002.Doc
wmk.yghdsxcnbza.cn/42402.Doc
wmk.yghdsxcnbza.cn/48802.Doc
wmj.yghdsxcnbza.cn/00448.Doc
wmj.yghdsxcnbza.cn/53759.Doc
wmj.yghdsxcnbza.cn/82488.Doc
wmj.yghdsxcnbza.cn/64288.Doc
wmj.yghdsxcnbza.cn/19551.Doc
wmj.yghdsxcnbza.cn/26246.Doc
wmj.yghdsxcnbza.cn/06828.Doc
wmj.yghdsxcnbza.cn/44660.Doc
wmj.yghdsxcnbza.cn/82600.Doc
wmj.yghdsxcnbza.cn/06000.Doc
wmh.yghdsxcnbza.cn/62046.Doc
wmh.yghdsxcnbza.cn/17155.Doc
wmh.yghdsxcnbza.cn/62088.Doc
wmh.yghdsxcnbza.cn/86820.Doc
wmh.yghdsxcnbza.cn/40420.Doc
wmh.yghdsxcnbza.cn/08268.Doc
wmh.yghdsxcnbza.cn/08620.Doc
wmh.yghdsxcnbza.cn/08262.Doc
wmh.yghdsxcnbza.cn/02040.Doc
wmh.yghdsxcnbza.cn/64280.Doc
wmg.yghdsxcnbza.cn/84228.Doc
wmg.yghdsxcnbza.cn/22626.Doc
wmg.yghdsxcnbza.cn/28822.Doc
wmg.yghdsxcnbza.cn/66446.Doc
wmg.yghdsxcnbza.cn/66286.Doc
wmg.yghdsxcnbza.cn/04428.Doc
wmg.yghdsxcnbza.cn/20284.Doc
wmg.yghdsxcnbza.cn/26604.Doc
wmg.yghdsxcnbza.cn/28484.Doc
wmg.yghdsxcnbza.cn/60846.Doc
wmf.yghdsxcnbza.cn/42806.Doc
wmf.yghdsxcnbza.cn/86224.Doc
wmf.yghdsxcnbza.cn/66642.Doc
wmf.yghdsxcnbza.cn/66448.Doc
wmf.yghdsxcnbza.cn/04800.Doc
wmf.yghdsxcnbza.cn/42006.Doc
wmf.yghdsxcnbza.cn/86886.Doc
wmf.yghdsxcnbza.cn/06424.Doc
wmf.yghdsxcnbza.cn/64642.Doc
wmf.yghdsxcnbza.cn/48604.Doc
wmd.yghdsxcnbza.cn/64288.Doc
wmd.yghdsxcnbza.cn/02446.Doc
wmd.yghdsxcnbza.cn/80826.Doc
wmd.yghdsxcnbza.cn/40866.Doc
wmd.yghdsxcnbza.cn/60266.Doc
wmd.yghdsxcnbza.cn/42024.Doc
wmd.yghdsxcnbza.cn/11193.Doc
wmd.yghdsxcnbza.cn/13551.Doc
wmd.yghdsxcnbza.cn/08000.Doc
wmd.yghdsxcnbza.cn/06008.Doc
wms.yghdsxcnbza.cn/06846.Doc
wms.yghdsxcnbza.cn/66624.Doc
wms.yghdsxcnbza.cn/28044.Doc
wms.yghdsxcnbza.cn/22866.Doc
wms.yghdsxcnbza.cn/04004.Doc
wms.yghdsxcnbza.cn/04200.Doc
wms.yghdsxcnbza.cn/48200.Doc
wms.yghdsxcnbza.cn/48048.Doc
wms.yghdsxcnbza.cn/53377.Doc
wms.yghdsxcnbza.cn/40202.Doc
wma.yghdsxcnbza.cn/46266.Doc
wma.yghdsxcnbza.cn/66020.Doc
wma.yghdsxcnbza.cn/26488.Doc
wma.yghdsxcnbza.cn/66280.Doc
wma.yghdsxcnbza.cn/28428.Doc
wma.yghdsxcnbza.cn/20228.Doc
wma.yghdsxcnbza.cn/64404.Doc
wma.yghdsxcnbza.cn/40402.Doc
wma.yghdsxcnbza.cn/68460.Doc
wma.yghdsxcnbza.cn/84006.Doc
wmp.yghdsxcnbza.cn/53351.Doc
wmp.yghdsxcnbza.cn/00220.Doc
wmp.yghdsxcnbza.cn/28884.Doc
wmp.yghdsxcnbza.cn/64620.Doc
wmp.yghdsxcnbza.cn/80442.Doc
wmp.yghdsxcnbza.cn/42460.Doc
wmp.yghdsxcnbza.cn/79151.Doc
wmp.yghdsxcnbza.cn/80004.Doc
wmp.yghdsxcnbza.cn/48404.Doc
wmp.yghdsxcnbza.cn/08240.Doc
wmo.yghdsxcnbza.cn/86826.Doc
wmo.yghdsxcnbza.cn/88840.Doc
wmo.yghdsxcnbza.cn/66888.Doc
wmo.yghdsxcnbza.cn/04066.Doc
wmo.yghdsxcnbza.cn/20048.Doc
wmo.yghdsxcnbza.cn/26482.Doc
wmo.yghdsxcnbza.cn/84804.Doc
wmo.yghdsxcnbza.cn/48882.Doc
wmo.yghdsxcnbza.cn/22862.Doc
wmo.yghdsxcnbza.cn/37395.Doc
wmi.yghdsxcnbza.cn/08006.Doc
wmi.yghdsxcnbza.cn/20828.Doc
wmi.yghdsxcnbza.cn/60886.Doc
wmi.yghdsxcnbza.cn/28642.Doc
wmi.yghdsxcnbza.cn/22484.Doc
wmi.yghdsxcnbza.cn/88664.Doc
wmi.yghdsxcnbza.cn/88408.Doc
wmi.yghdsxcnbza.cn/66080.Doc
wmi.yghdsxcnbza.cn/62242.Doc
wmi.yghdsxcnbza.cn/26008.Doc
wmu.yghdsxcnbza.cn/28842.Doc
wmu.yghdsxcnbza.cn/00086.Doc
wmu.yghdsxcnbza.cn/24488.Doc
wmu.yghdsxcnbza.cn/88462.Doc
wmu.yghdsxcnbza.cn/44886.Doc
wmu.yghdsxcnbza.cn/02040.Doc
wmu.yghdsxcnbza.cn/02862.Doc
wmu.yghdsxcnbza.cn/28682.Doc
wmu.yghdsxcnbza.cn/59593.Doc
wmu.yghdsxcnbza.cn/20224.Doc
wmy.yghdsxcnbza.cn/80620.Doc
wmy.yghdsxcnbza.cn/33379.Doc
wmy.yghdsxcnbza.cn/48428.Doc
wmy.yghdsxcnbza.cn/02022.Doc
wmy.yghdsxcnbza.cn/66422.Doc
wmy.yghdsxcnbza.cn/20482.Doc
wmy.yghdsxcnbza.cn/84224.Doc
wmy.yghdsxcnbza.cn/60468.Doc
wmy.yghdsxcnbza.cn/46442.Doc
wmy.yghdsxcnbza.cn/88008.Doc
wmt.yghdsxcnbza.cn/86644.Doc
wmt.yghdsxcnbza.cn/04640.Doc
wmt.yghdsxcnbza.cn/55115.Doc
wmt.yghdsxcnbza.cn/80044.Doc
wmt.yghdsxcnbza.cn/62880.Doc
wmt.yghdsxcnbza.cn/66228.Doc
wmt.yghdsxcnbza.cn/00040.Doc
wmt.yghdsxcnbza.cn/86646.Doc
wmt.yghdsxcnbza.cn/22408.Doc
wmt.yghdsxcnbza.cn/04086.Doc
wmr.yghdsxcnbza.cn/66260.Doc
wmr.yghdsxcnbza.cn/19113.Doc
wmr.yghdsxcnbza.cn/66282.Doc
wmr.yghdsxcnbza.cn/93791.Doc
wmr.yghdsxcnbza.cn/97537.Doc
wmr.yghdsxcnbza.cn/20242.Doc
wmr.yghdsxcnbza.cn/00422.Doc
wmr.yghdsxcnbza.cn/60086.Doc
wmr.yghdsxcnbza.cn/62022.Doc
wmr.yghdsxcnbza.cn/08444.Doc
wme.yghdsxcnbza.cn/28668.Doc
wme.yghdsxcnbza.cn/06080.Doc
wme.yghdsxcnbza.cn/20644.Doc
wme.yghdsxcnbza.cn/39735.Doc
wme.yghdsxcnbza.cn/48426.Doc
wme.yghdsxcnbza.cn/68462.Doc
wme.yghdsxcnbza.cn/88426.Doc
wme.yghdsxcnbza.cn/00220.Doc
wme.yghdsxcnbza.cn/08402.Doc
wme.yghdsxcnbza.cn/40468.Doc
wmw.yghdsxcnbza.cn/24220.Doc
wmw.yghdsxcnbza.cn/80624.Doc
wmw.yghdsxcnbza.cn/79399.Doc
wmw.yghdsxcnbza.cn/26200.Doc
wmw.yghdsxcnbza.cn/82486.Doc
wmw.yghdsxcnbza.cn/20422.Doc
wmw.yghdsxcnbza.cn/26466.Doc
wmw.yghdsxcnbza.cn/42206.Doc
wmw.yghdsxcnbza.cn/82842.Doc
wmw.yghdsxcnbza.cn/42868.Doc
wmq.yghdsxcnbza.cn/97191.Doc
wmq.yghdsxcnbza.cn/44460.Doc
wmq.yghdsxcnbza.cn/48444.Doc
wmq.yghdsxcnbza.cn/66000.Doc
wmq.yghdsxcnbza.cn/26226.Doc
wmq.yghdsxcnbza.cn/68400.Doc
wmq.yghdsxcnbza.cn/00620.Doc
wmq.yghdsxcnbza.cn/04624.Doc
wmq.yghdsxcnbza.cn/44822.Doc
wmq.yghdsxcnbza.cn/62686.Doc
wnm.yghdsxcnbza.cn/62848.Doc
wnm.yghdsxcnbza.cn/24640.Doc
wnm.yghdsxcnbza.cn/80620.Doc
wnm.yghdsxcnbza.cn/44624.Doc
wnm.yghdsxcnbza.cn/62080.Doc
wnm.yghdsxcnbza.cn/64262.Doc
wnm.yghdsxcnbza.cn/97915.Doc
wnm.yghdsxcnbza.cn/79597.Doc
wnm.yghdsxcnbza.cn/64404.Doc
wnm.yghdsxcnbza.cn/80064.Doc
wnn.yghdsxcnbza.cn/22228.Doc
wnn.yghdsxcnbza.cn/22844.Doc
wnn.yghdsxcnbza.cn/84086.Doc
wnn.yghdsxcnbza.cn/40484.Doc
wnn.yghdsxcnbza.cn/80224.Doc
wnn.yghdsxcnbza.cn/82228.Doc
wnn.yghdsxcnbza.cn/06660.Doc
wnn.yghdsxcnbza.cn/02448.Doc
wnn.yghdsxcnbza.cn/73917.Doc
wnn.yghdsxcnbza.cn/24028.Doc
wnb.yghdsxcnbza.cn/48882.Doc
wnb.yghdsxcnbza.cn/24428.Doc
wnb.yghdsxcnbza.cn/13359.Doc
wnb.yghdsxcnbza.cn/04484.Doc
wnb.yghdsxcnbza.cn/66460.Doc
wnb.yghdsxcnbza.cn/00604.Doc
wnb.yghdsxcnbza.cn/60800.Doc
wnb.yghdsxcnbza.cn/62268.Doc
wnb.yghdsxcnbza.cn/40262.Doc
wnb.yghdsxcnbza.cn/88600.Doc
wnv.yghdsxcnbza.cn/84602.Doc
wnv.yghdsxcnbza.cn/60600.Doc
wnv.yghdsxcnbza.cn/80244.Doc
wnv.yghdsxcnbza.cn/42604.Doc
wnv.yghdsxcnbza.cn/84086.Doc
wnv.yghdsxcnbza.cn/19379.Doc
wnv.yghdsxcnbza.cn/48866.Doc
wnv.yghdsxcnbza.cn/44866.Doc
wnv.yghdsxcnbza.cn/42660.Doc
wnv.yghdsxcnbza.cn/55559.Doc
wnc.yghdsxcnbza.cn/28444.Doc
wnc.yghdsxcnbza.cn/08642.Doc
wnc.yghdsxcnbza.cn/62860.Doc
wnc.yghdsxcnbza.cn/68460.Doc
wnc.yghdsxcnbza.cn/00080.Doc
wnc.yghdsxcnbza.cn/26420.Doc
wnc.yghdsxcnbza.cn/22480.Doc
wnc.yghdsxcnbza.cn/48048.Doc
wnc.yghdsxcnbza.cn/26006.Doc
wnc.yghdsxcnbza.cn/37311.Doc
wnx.yghdsxcnbza.cn/24284.Doc
wnx.yghdsxcnbza.cn/64820.Doc
wnx.yghdsxcnbza.cn/84084.Doc
wnx.yghdsxcnbza.cn/66202.Doc
wnx.yghdsxcnbza.cn/04866.Doc
wnx.yghdsxcnbza.cn/00286.Doc
wnx.yghdsxcnbza.cn/48220.Doc
wnx.yghdsxcnbza.cn/48646.Doc
wnx.yghdsxcnbza.cn/42886.Doc
wnx.yghdsxcnbza.cn/24208.Doc
wnz.yghdsxcnbza.cn/66888.Doc
wnz.yghdsxcnbza.cn/97955.Doc
wnz.yghdsxcnbza.cn/26060.Doc
wnz.yghdsxcnbza.cn/04866.Doc
wnz.yghdsxcnbza.cn/20086.Doc
wnz.yghdsxcnbza.cn/44686.Doc
wnz.yghdsxcnbza.cn/64804.Doc
wnz.yghdsxcnbza.cn/68822.Doc
wnz.yghdsxcnbza.cn/88620.Doc
wnz.yghdsxcnbza.cn/55117.Doc
wnl.yghdsxcnbza.cn/66246.Doc
wnl.yghdsxcnbza.cn/22048.Doc
wnl.yghdsxcnbza.cn/62604.Doc
wnl.yghdsxcnbza.cn/88624.Doc
wnl.yghdsxcnbza.cn/04400.Doc
wnl.yghdsxcnbza.cn/20226.Doc
wnl.yghdsxcnbza.cn/39793.Doc
wnl.yghdsxcnbza.cn/84088.Doc
wnl.yghdsxcnbza.cn/42206.Doc
wnl.yghdsxcnbza.cn/44448.Doc
wnk.yghdsxcnbza.cn/62260.Doc
wnk.yghdsxcnbza.cn/20642.Doc
wnk.yghdsxcnbza.cn/48482.Doc
wnk.yghdsxcnbza.cn/86822.Doc
wnk.yghdsxcnbza.cn/44080.Doc
wnk.yghdsxcnbza.cn/48446.Doc
wnk.yghdsxcnbza.cn/04688.Doc
wnk.yghdsxcnbza.cn/46424.Doc
wnk.yghdsxcnbza.cn/22468.Doc
wnk.yghdsxcnbza.cn/51977.Doc
wnj.yghdsxcnbza.cn/48048.Doc
wnj.yghdsxcnbza.cn/06844.Doc
wnj.yghdsxcnbza.cn/62664.Doc
wnj.yghdsxcnbza.cn/15311.Doc
wnj.yghdsxcnbza.cn/20680.Doc
wnj.yghdsxcnbza.cn/28248.Doc
wnj.yghdsxcnbza.cn/80846.Doc
wnj.yghdsxcnbza.cn/64462.Doc
wnj.yghdsxcnbza.cn/02662.Doc
wnj.yghdsxcnbza.cn/60246.Doc
wnh.yghdsxcnbza.cn/28808.Doc
wnh.yghdsxcnbza.cn/00244.Doc
wnh.yghdsxcnbza.cn/40640.Doc
wnh.yghdsxcnbza.cn/15759.Doc
wnh.yghdsxcnbza.cn/82864.Doc
wnh.yghdsxcnbza.cn/02684.Doc
wnh.yghdsxcnbza.cn/26026.Doc
wnh.yghdsxcnbza.cn/88860.Doc
wnh.yghdsxcnbza.cn/08204.Doc
wnh.yghdsxcnbza.cn/28242.Doc
wng.yghdsxcnbza.cn/68686.Doc
wng.yghdsxcnbza.cn/00086.Doc
wng.yghdsxcnbza.cn/93537.Doc
wng.yghdsxcnbza.cn/08088.Doc
wng.yghdsxcnbza.cn/22260.Doc
wng.yghdsxcnbza.cn/42462.Doc
wng.yghdsxcnbza.cn/99335.Doc
wng.yghdsxcnbza.cn/68006.Doc
wng.yghdsxcnbza.cn/68226.Doc
wng.yghdsxcnbza.cn/00020.Doc
wnf.yghdsxcnbza.cn/06064.Doc
wnf.yghdsxcnbza.cn/24204.Doc
wnf.yghdsxcnbza.cn/44886.Doc
wnf.yghdsxcnbza.cn/46040.Doc
wnf.yghdsxcnbza.cn/02048.Doc
wnf.yghdsxcnbza.cn/31733.Doc
wnf.yghdsxcnbza.cn/24882.Doc
wnf.yghdsxcnbza.cn/26242.Doc
wnf.yghdsxcnbza.cn/00860.Doc
wnf.yghdsxcnbza.cn/31199.Doc
wnd.yghdsxcnbza.cn/84864.Doc
wnd.yghdsxcnbza.cn/60848.Doc
wnd.yghdsxcnbza.cn/62020.Doc
wnd.yghdsxcnbza.cn/00840.Doc
wnd.yghdsxcnbza.cn/00484.Doc
wnd.yghdsxcnbza.cn/44862.Doc
wnd.yghdsxcnbza.cn/97373.Doc
wnd.yghdsxcnbza.cn/46644.Doc
wnd.yghdsxcnbza.cn/60420.Doc
wnd.yghdsxcnbza.cn/02042.Doc
wns.yghdsxcnbza.cn/02626.Doc
wns.yghdsxcnbza.cn/04866.Doc
wns.yghdsxcnbza.cn/00682.Doc
wns.yghdsxcnbza.cn/80042.Doc
wns.yghdsxcnbza.cn/68202.Doc
wns.yghdsxcnbza.cn/60648.Doc
wns.yghdsxcnbza.cn/20022.Doc
wns.yghdsxcnbza.cn/68086.Doc
wns.yghdsxcnbza.cn/86820.Doc
wns.yghdsxcnbza.cn/84264.Doc
wna.yghdsxcnbza.cn/68060.Doc
wna.yghdsxcnbza.cn/28026.Doc
wna.yghdsxcnbza.cn/44800.Doc
wna.yghdsxcnbza.cn/64068.Doc
wna.yghdsxcnbza.cn/68602.Doc
wna.yghdsxcnbza.cn/44206.Doc
wna.yghdsxcnbza.cn/64244.Doc
wna.yghdsxcnbza.cn/66264.Doc
wna.yghdsxcnbza.cn/42222.Doc
wna.yghdsxcnbza.cn/44060.Doc
