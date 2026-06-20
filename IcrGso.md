喂闭蚊杭谢


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

诓祷踊灿献附尤杀幸踩炎烈梁可路

tv.blog.yqvyet.cn/Article/details/606864.sHtML
tv.blog.yqvyet.cn/Article/details/082844.sHtML
tv.blog.yqvyet.cn/Article/details/393131.sHtML
tv.blog.yqvyet.cn/Article/details/288668.sHtML
tv.blog.yqvyet.cn/Article/details/228806.sHtML
tv.blog.yqvyet.cn/Article/details/353999.sHtML
tv.blog.yqvyet.cn/Article/details/117355.sHtML
tv.blog.yqvyet.cn/Article/details/573939.sHtML
tv.blog.yqvyet.cn/Article/details/593393.sHtML
tv.blog.yqvyet.cn/Article/details/319977.sHtML
tv.blog.yqvyet.cn/Article/details/046244.sHtML
tv.blog.yqvyet.cn/Article/details/008486.sHtML
tv.blog.yqvyet.cn/Article/details/953771.sHtML
tv.blog.yqvyet.cn/Article/details/559977.sHtML
tv.blog.yqvyet.cn/Article/details/280406.sHtML
tv.blog.yqvyet.cn/Article/details/955717.sHtML
tv.blog.yqvyet.cn/Article/details/999373.sHtML
tv.blog.yqvyet.cn/Article/details/117537.sHtML
tv.blog.yqvyet.cn/Article/details/460800.sHtML
tv.blog.yqvyet.cn/Article/details/282646.sHtML
tv.blog.yqvyet.cn/Article/details/406880.sHtML
tv.blog.yqvyet.cn/Article/details/828064.sHtML
tv.blog.yqvyet.cn/Article/details/086628.sHtML
tv.blog.yqvyet.cn/Article/details/820806.sHtML
tv.blog.yqvyet.cn/Article/details/311317.sHtML
tv.blog.yqvyet.cn/Article/details/999133.sHtML
tv.blog.yqvyet.cn/Article/details/197191.sHtML
tv.blog.yqvyet.cn/Article/details/264824.sHtML
tv.blog.yqvyet.cn/Article/details/995591.sHtML
tv.blog.yqvyet.cn/Article/details/046206.sHtML
tv.blog.yqvyet.cn/Article/details/862284.sHtML
tv.blog.yqvyet.cn/Article/details/397797.sHtML
tv.blog.yqvyet.cn/Article/details/951519.sHtML
tv.blog.yqvyet.cn/Article/details/406628.sHtML
tv.blog.yqvyet.cn/Article/details/082088.sHtML
tv.blog.yqvyet.cn/Article/details/351317.sHtML
tv.blog.yqvyet.cn/Article/details/135135.sHtML
tv.blog.yqvyet.cn/Article/details/799797.sHtML
tv.blog.yqvyet.cn/Article/details/751133.sHtML
tv.blog.yqvyet.cn/Article/details/466442.sHtML
tv.blog.yqvyet.cn/Article/details/646800.sHtML
tv.blog.yqvyet.cn/Article/details/602844.sHtML
tv.blog.yqvyet.cn/Article/details/248460.sHtML
tv.blog.yqvyet.cn/Article/details/226804.sHtML
tv.blog.yqvyet.cn/Article/details/006822.sHtML
tv.blog.yqvyet.cn/Article/details/800840.sHtML
tv.blog.yqvyet.cn/Article/details/599337.sHtML
tv.blog.yqvyet.cn/Article/details/753537.sHtML
tv.blog.yqvyet.cn/Article/details/628646.sHtML
tv.blog.yqvyet.cn/Article/details/422442.sHtML
tv.blog.yqvyet.cn/Article/details/573715.sHtML
tv.blog.yqvyet.cn/Article/details/935513.sHtML
tv.blog.yqvyet.cn/Article/details/224664.sHtML
tv.blog.yqvyet.cn/Article/details/446264.sHtML
tv.blog.yqvyet.cn/Article/details/224448.sHtML
tv.blog.yqvyet.cn/Article/details/595133.sHtML
tv.blog.yqvyet.cn/Article/details/426228.sHtML
tv.blog.yqvyet.cn/Article/details/682208.sHtML
tv.blog.yqvyet.cn/Article/details/440888.sHtML
tv.blog.yqvyet.cn/Article/details/660406.sHtML
tv.blog.yqvyet.cn/Article/details/226004.sHtML
tv.blog.yqvyet.cn/Article/details/842260.sHtML
tv.blog.yqvyet.cn/Article/details/311779.sHtML
tv.blog.yqvyet.cn/Article/details/919333.sHtML
tv.blog.yqvyet.cn/Article/details/840648.sHtML
tv.blog.yqvyet.cn/Article/details/006468.sHtML
tv.blog.yqvyet.cn/Article/details/040826.sHtML
tv.blog.yqvyet.cn/Article/details/626860.sHtML
tv.blog.yqvyet.cn/Article/details/622200.sHtML
tv.blog.yqvyet.cn/Article/details/462246.sHtML
tv.blog.yqvyet.cn/Article/details/480046.sHtML
tv.blog.yqvyet.cn/Article/details/640826.sHtML
tv.blog.yqvyet.cn/Article/details/680828.sHtML
tv.blog.yqvyet.cn/Article/details/484886.sHtML
tv.blog.yqvyet.cn/Article/details/482688.sHtML
tv.blog.yqvyet.cn/Article/details/733179.sHtML
tv.blog.yqvyet.cn/Article/details/880224.sHtML
tv.blog.yqvyet.cn/Article/details/919719.sHtML
tv.blog.yqvyet.cn/Article/details/448444.sHtML
tv.blog.yqvyet.cn/Article/details/993977.sHtML
tv.blog.yqvyet.cn/Article/details/171159.sHtML
tv.blog.yqvyet.cn/Article/details/046440.sHtML
tv.blog.yqvyet.cn/Article/details/480206.sHtML
tv.blog.yqvyet.cn/Article/details/848602.sHtML
tv.blog.yqvyet.cn/Article/details/462608.sHtML
tv.blog.yqvyet.cn/Article/details/711153.sHtML
tv.blog.yqvyet.cn/Article/details/428822.sHtML
tv.blog.yqvyet.cn/Article/details/797953.sHtML
tv.blog.yqvyet.cn/Article/details/355597.sHtML
tv.blog.yqvyet.cn/Article/details/957115.sHtML
tv.blog.yqvyet.cn/Article/details/868428.sHtML
tv.blog.yqvyet.cn/Article/details/202084.sHtML
tv.blog.yqvyet.cn/Article/details/797791.sHtML
tv.blog.yqvyet.cn/Article/details/488460.sHtML
tv.blog.yqvyet.cn/Article/details/884644.sHtML
tv.blog.yqvyet.cn/Article/details/137997.sHtML
tv.blog.yqvyet.cn/Article/details/422840.sHtML
tv.blog.yqvyet.cn/Article/details/153575.sHtML
tv.blog.yqvyet.cn/Article/details/791555.sHtML
tv.blog.yqvyet.cn/Article/details/119191.sHtML
tv.blog.yqvyet.cn/Article/details/931939.sHtML
tv.blog.yqvyet.cn/Article/details/193937.sHtML
tv.blog.yqvyet.cn/Article/details/040682.sHtML
tv.blog.yqvyet.cn/Article/details/717937.sHtML
tv.blog.yqvyet.cn/Article/details/684840.sHtML
tv.blog.yqvyet.cn/Article/details/424882.sHtML
tv.blog.yqvyet.cn/Article/details/931557.sHtML
tv.blog.yqvyet.cn/Article/details/284846.sHtML
tv.blog.yqvyet.cn/Article/details/066848.sHtML
tv.blog.yqvyet.cn/Article/details/755199.sHtML
tv.blog.yqvyet.cn/Article/details/975957.sHtML
tv.blog.yqvyet.cn/Article/details/979191.sHtML
tv.blog.yqvyet.cn/Article/details/664002.sHtML
tv.blog.yqvyet.cn/Article/details/046048.sHtML
tv.blog.yqvyet.cn/Article/details/111171.sHtML
tv.blog.yqvyet.cn/Article/details/044480.sHtML
tv.blog.yqvyet.cn/Article/details/733951.sHtML
tv.blog.yqvyet.cn/Article/details/391957.sHtML
tv.blog.yqvyet.cn/Article/details/622640.sHtML
tv.blog.yqvyet.cn/Article/details/440848.sHtML
tv.blog.yqvyet.cn/Article/details/955979.sHtML
tv.blog.yqvyet.cn/Article/details/555911.sHtML
tv.blog.yqvyet.cn/Article/details/628868.sHtML
tv.blog.yqvyet.cn/Article/details/466462.sHtML
tv.blog.yqvyet.cn/Article/details/282202.sHtML
tv.blog.yqvyet.cn/Article/details/402606.sHtML
tv.blog.yqvyet.cn/Article/details/539597.sHtML
tv.blog.yqvyet.cn/Article/details/246200.sHtML
tv.blog.yqvyet.cn/Article/details/642206.sHtML
tv.blog.yqvyet.cn/Article/details/086488.sHtML
tv.blog.yqvyet.cn/Article/details/886002.sHtML
tv.blog.yqvyet.cn/Article/details/626006.sHtML
tv.blog.yqvyet.cn/Article/details/222644.sHtML
tv.blog.yqvyet.cn/Article/details/260484.sHtML
tv.blog.yqvyet.cn/Article/details/624686.sHtML
tv.blog.yqvyet.cn/Article/details/715115.sHtML
tv.blog.yqvyet.cn/Article/details/626606.sHtML
tv.blog.yqvyet.cn/Article/details/220486.sHtML
tv.blog.yqvyet.cn/Article/details/846424.sHtML
tv.blog.yqvyet.cn/Article/details/355915.sHtML
tv.blog.yqvyet.cn/Article/details/113775.sHtML
tv.blog.yqvyet.cn/Article/details/268448.sHtML
tv.blog.yqvyet.cn/Article/details/080660.sHtML
tv.blog.yqvyet.cn/Article/details/084040.sHtML
tv.blog.yqvyet.cn/Article/details/622888.sHtML
tv.blog.yqvyet.cn/Article/details/682804.sHtML
tv.blog.yqvyet.cn/Article/details/622646.sHtML
tv.blog.yqvyet.cn/Article/details/262284.sHtML
tv.blog.yqvyet.cn/Article/details/648200.sHtML
tv.blog.yqvyet.cn/Article/details/399135.sHtML
tv.blog.yqvyet.cn/Article/details/686020.sHtML
tv.blog.yqvyet.cn/Article/details/759757.sHtML
tv.blog.yqvyet.cn/Article/details/242828.sHtML
tv.blog.yqvyet.cn/Article/details/022826.sHtML
tv.blog.yqvyet.cn/Article/details/117373.sHtML
tv.blog.yqvyet.cn/Article/details/804084.sHtML
tv.blog.yqvyet.cn/Article/details/579711.sHtML
tv.blog.yqvyet.cn/Article/details/664468.sHtML
tv.blog.yqvyet.cn/Article/details/800844.sHtML
tv.blog.yqvyet.cn/Article/details/422462.sHtML
tv.blog.yqvyet.cn/Article/details/939173.sHtML
tv.blog.yqvyet.cn/Article/details/404826.sHtML
tv.blog.yqvyet.cn/Article/details/355319.sHtML
tv.blog.yqvyet.cn/Article/details/426202.sHtML
tv.blog.yqvyet.cn/Article/details/319951.sHtML
tv.blog.yqvyet.cn/Article/details/133717.sHtML
tv.blog.yqvyet.cn/Article/details/682666.sHtML
tv.blog.yqvyet.cn/Article/details/008424.sHtML
tv.blog.yqvyet.cn/Article/details/777197.sHtML
tv.blog.yqvyet.cn/Article/details/424000.sHtML
tv.blog.yqvyet.cn/Article/details/173515.sHtML
tv.blog.yqvyet.cn/Article/details/664068.sHtML
tv.blog.yqvyet.cn/Article/details/866444.sHtML
tv.blog.yqvyet.cn/Article/details/020804.sHtML
tv.blog.yqvyet.cn/Article/details/137337.sHtML
tv.blog.yqvyet.cn/Article/details/715333.sHtML
tv.blog.yqvyet.cn/Article/details/955355.sHtML
tv.blog.yqvyet.cn/Article/details/206226.sHtML
tv.blog.yqvyet.cn/Article/details/846882.sHtML
tv.blog.yqvyet.cn/Article/details/846688.sHtML
tv.blog.yqvyet.cn/Article/details/333353.sHtML
tv.blog.yqvyet.cn/Article/details/246402.sHtML
tv.blog.yqvyet.cn/Article/details/046202.sHtML
tv.blog.yqvyet.cn/Article/details/262244.sHtML
tv.blog.yqvyet.cn/Article/details/204688.sHtML
tv.blog.yqvyet.cn/Article/details/620262.sHtML
tv.blog.yqvyet.cn/Article/details/820680.sHtML
tv.blog.yqvyet.cn/Article/details/680006.sHtML
tv.blog.yqvyet.cn/Article/details/975133.sHtML
tv.blog.yqvyet.cn/Article/details/068062.sHtML
tv.blog.yqvyet.cn/Article/details/882840.sHtML
tv.blog.yqvyet.cn/Article/details/462202.sHtML
tv.blog.yqvyet.cn/Article/details/913355.sHtML
tv.blog.yqvyet.cn/Article/details/880082.sHtML
tv.blog.yqvyet.cn/Article/details/684448.sHtML
tv.blog.yqvyet.cn/Article/details/028042.sHtML
tv.blog.yqvyet.cn/Article/details/428644.sHtML
tv.blog.yqvyet.cn/Article/details/400262.sHtML
tv.blog.yqvyet.cn/Article/details/808006.sHtML
tv.blog.yqvyet.cn/Article/details/977131.sHtML
tv.blog.yqvyet.cn/Article/details/422862.sHtML
tv.blog.yqvyet.cn/Article/details/737115.sHtML
tv.blog.yqvyet.cn/Article/details/086482.sHtML
tv.blog.yqvyet.cn/Article/details/046084.sHtML
tv.blog.yqvyet.cn/Article/details/280028.sHtML
tv.blog.yqvyet.cn/Article/details/640006.sHtML
tv.blog.yqvyet.cn/Article/details/022426.sHtML
tv.blog.yqvyet.cn/Article/details/662660.sHtML
tv.blog.yqvyet.cn/Article/details/462804.sHtML
tv.blog.yqvyet.cn/Article/details/177931.sHtML
tv.blog.yqvyet.cn/Article/details/666862.sHtML
tv.blog.yqvyet.cn/Article/details/731739.sHtML
tv.blog.yqvyet.cn/Article/details/280084.sHtML
tv.blog.yqvyet.cn/Article/details/022428.sHtML
tv.blog.yqvyet.cn/Article/details/171979.sHtML
tv.blog.yqvyet.cn/Article/details/068660.sHtML
tv.blog.yqvyet.cn/Article/details/791773.sHtML
tv.blog.yqvyet.cn/Article/details/022002.sHtML
tv.blog.yqvyet.cn/Article/details/886448.sHtML
tv.blog.yqvyet.cn/Article/details/911171.sHtML
tv.blog.yqvyet.cn/Article/details/155593.sHtML
tv.blog.yqvyet.cn/Article/details/731757.sHtML
tv.blog.yqvyet.cn/Article/details/822282.sHtML
tv.blog.yqvyet.cn/Article/details/428420.sHtML
tv.blog.yqvyet.cn/Article/details/484626.sHtML
tv.blog.yqvyet.cn/Article/details/008448.sHtML
tv.blog.yqvyet.cn/Article/details/288048.sHtML
tv.blog.yqvyet.cn/Article/details/440642.sHtML
tv.blog.yqvyet.cn/Article/details/402482.sHtML
tv.blog.yqvyet.cn/Article/details/846262.sHtML
tv.blog.yqvyet.cn/Article/details/642820.sHtML
tv.blog.yqvyet.cn/Article/details/115955.sHtML
tv.blog.yqvyet.cn/Article/details/535353.sHtML
tv.blog.yqvyet.cn/Article/details/682402.sHtML
tv.blog.yqvyet.cn/Article/details/208062.sHtML
tv.blog.yqvyet.cn/Article/details/264462.sHtML
tv.blog.yqvyet.cn/Article/details/991797.sHtML
tv.blog.yqvyet.cn/Article/details/684684.sHtML
tv.blog.yqvyet.cn/Article/details/602224.sHtML
tv.blog.yqvyet.cn/Article/details/559713.sHtML
tv.blog.yqvyet.cn/Article/details/660242.sHtML
tv.blog.yqvyet.cn/Article/details/155137.sHtML
tv.blog.yqvyet.cn/Article/details/844444.sHtML
tv.blog.yqvyet.cn/Article/details/268082.sHtML
tv.blog.yqvyet.cn/Article/details/139397.sHtML
tv.blog.yqvyet.cn/Article/details/317571.sHtML
tv.blog.yqvyet.cn/Article/details/602228.sHtML
tv.blog.yqvyet.cn/Article/details/997919.sHtML
tv.blog.yqvyet.cn/Article/details/315199.sHtML
tv.blog.yqvyet.cn/Article/details/044222.sHtML
tv.blog.yqvyet.cn/Article/details/313531.sHtML
tv.blog.yqvyet.cn/Article/details/048024.sHtML
tv.blog.yqvyet.cn/Article/details/428044.sHtML
tv.blog.yqvyet.cn/Article/details/860264.sHtML
tv.blog.yqvyet.cn/Article/details/357795.sHtML
tv.blog.yqvyet.cn/Article/details/682266.sHtML
tv.blog.yqvyet.cn/Article/details/791537.sHtML
tv.blog.yqvyet.cn/Article/details/577335.sHtML
tv.blog.yqvyet.cn/Article/details/597911.sHtML
tv.blog.yqvyet.cn/Article/details/202002.sHtML
tv.blog.yqvyet.cn/Article/details/622448.sHtML
tv.blog.yqvyet.cn/Article/details/717333.sHtML
tv.blog.yqvyet.cn/Article/details/208002.sHtML
tv.blog.yqvyet.cn/Article/details/466808.sHtML
tv.blog.yqvyet.cn/Article/details/791935.sHtML
tv.blog.yqvyet.cn/Article/details/242002.sHtML
tv.blog.yqvyet.cn/Article/details/464860.sHtML
tv.blog.yqvyet.cn/Article/details/868660.sHtML
tv.blog.yqvyet.cn/Article/details/484040.sHtML
tv.blog.yqvyet.cn/Article/details/040802.sHtML
tv.blog.yqvyet.cn/Article/details/791331.sHtML
tv.blog.yqvyet.cn/Article/details/840600.sHtML
tv.blog.yqvyet.cn/Article/details/222264.sHtML
tv.blog.yqvyet.cn/Article/details/517157.sHtML
tv.blog.yqvyet.cn/Article/details/028866.sHtML
tv.blog.yqvyet.cn/Article/details/860244.sHtML
tv.blog.yqvyet.cn/Article/details/915797.sHtML
tv.blog.yqvyet.cn/Article/details/755593.sHtML
tv.blog.yqvyet.cn/Article/details/680420.sHtML
tv.blog.yqvyet.cn/Article/details/264684.sHtML
tv.blog.yqvyet.cn/Article/details/664844.sHtML
tv.blog.yqvyet.cn/Article/details/739157.sHtML
tv.blog.yqvyet.cn/Article/details/242246.sHtML
tv.blog.yqvyet.cn/Article/details/626662.sHtML
tv.blog.yqvyet.cn/Article/details/199175.sHtML
tv.blog.yqvyet.cn/Article/details/842868.sHtML
tv.blog.yqvyet.cn/Article/details/868422.sHtML
tv.blog.yqvyet.cn/Article/details/468440.sHtML
tv.blog.yqvyet.cn/Article/details/806406.sHtML
tv.blog.yqvyet.cn/Article/details/935755.sHtML
tv.blog.yqvyet.cn/Article/details/028008.sHtML
tv.blog.yqvyet.cn/Article/details/460222.sHtML
tv.blog.yqvyet.cn/Article/details/046686.sHtML
tv.blog.yqvyet.cn/Article/details/408244.sHtML
tv.blog.yqvyet.cn/Article/details/068200.sHtML
tv.blog.yqvyet.cn/Article/details/648860.sHtML
tv.blog.yqvyet.cn/Article/details/024688.sHtML
tv.blog.yqvyet.cn/Article/details/957517.sHtML
tv.blog.yqvyet.cn/Article/details/688846.sHtML
tv.blog.yqvyet.cn/Article/details/642244.sHtML
tv.blog.yqvyet.cn/Article/details/488668.sHtML
tv.blog.yqvyet.cn/Article/details/191755.sHtML
tv.blog.yqvyet.cn/Article/details/173917.sHtML
tv.blog.yqvyet.cn/Article/details/424082.sHtML
tv.blog.yqvyet.cn/Article/details/119777.sHtML
tv.blog.yqvyet.cn/Article/details/551573.sHtML
tv.blog.yqvyet.cn/Article/details/533151.sHtML
tv.blog.yqvyet.cn/Article/details/884266.sHtML
tv.blog.yqvyet.cn/Article/details/200426.sHtML
tv.blog.yqvyet.cn/Article/details/066020.sHtML
tv.blog.yqvyet.cn/Article/details/915119.sHtML
tv.blog.yqvyet.cn/Article/details/626068.sHtML
tv.blog.yqvyet.cn/Article/details/000228.sHtML
tv.blog.yqvyet.cn/Article/details/040868.sHtML
tv.blog.yqvyet.cn/Article/details/084082.sHtML
tv.blog.yqvyet.cn/Article/details/557593.sHtML
tv.blog.yqvyet.cn/Article/details/040266.sHtML
tv.blog.yqvyet.cn/Article/details/286288.sHtML
tv.blog.yqvyet.cn/Article/details/733175.sHtML
tv.blog.yqvyet.cn/Article/details/799753.sHtML
tv.blog.yqvyet.cn/Article/details/919371.sHtML
tv.blog.yqvyet.cn/Article/details/804648.sHtML
tv.blog.yqvyet.cn/Article/details/844400.sHtML
tv.blog.yqvyet.cn/Article/details/082048.sHtML
tv.blog.yqvyet.cn/Article/details/193935.sHtML
tv.blog.yqvyet.cn/Article/details/484464.sHtML
tv.blog.yqvyet.cn/Article/details/806240.sHtML
tv.blog.yqvyet.cn/Article/details/260640.sHtML
tv.blog.yqvyet.cn/Article/details/086088.sHtML
tv.blog.yqvyet.cn/Article/details/820466.sHtML
tv.blog.yqvyet.cn/Article/details/191359.sHtML
tv.blog.yqvyet.cn/Article/details/311731.sHtML
tv.blog.yqvyet.cn/Article/details/004264.sHtML
tv.blog.yqvyet.cn/Article/details/620644.sHtML
tv.blog.yqvyet.cn/Article/details/468428.sHtML
tv.blog.yqvyet.cn/Article/details/355739.sHtML
tv.blog.yqvyet.cn/Article/details/642682.sHtML
tv.blog.yqvyet.cn/Article/details/240624.sHtML
tv.blog.yqvyet.cn/Article/details/113359.sHtML
tv.blog.yqvyet.cn/Article/details/282424.sHtML
tv.blog.yqvyet.cn/Article/details/579997.sHtML
tv.blog.yqvyet.cn/Article/details/268604.sHtML
tv.blog.yqvyet.cn/Article/details/915557.sHtML
tv.blog.yqvyet.cn/Article/details/937939.sHtML
tv.blog.yqvyet.cn/Article/details/242206.sHtML
tv.blog.yqvyet.cn/Article/details/200400.sHtML
tv.blog.yqvyet.cn/Article/details/004824.sHtML
tv.blog.yqvyet.cn/Article/details/979997.sHtML
tv.blog.yqvyet.cn/Article/details/931355.sHtML
tv.blog.yqvyet.cn/Article/details/719175.sHtML
tv.blog.yqvyet.cn/Article/details/628404.sHtML
tv.blog.yqvyet.cn/Article/details/280008.sHtML
tv.blog.yqvyet.cn/Article/details/062844.sHtML
tv.blog.yqvyet.cn/Article/details/000406.sHtML
tv.blog.yqvyet.cn/Article/details/422008.sHtML
tv.blog.yqvyet.cn/Article/details/357351.sHtML
tv.blog.yqvyet.cn/Article/details/426088.sHtML
tv.blog.yqvyet.cn/Article/details/466846.sHtML
tv.blog.yqvyet.cn/Article/details/933193.sHtML
tv.blog.yqvyet.cn/Article/details/399351.sHtML
tv.blog.yqvyet.cn/Article/details/399171.sHtML
tv.blog.yqvyet.cn/Article/details/484060.sHtML
tv.blog.yqvyet.cn/Article/details/802422.sHtML
tv.blog.yqvyet.cn/Article/details/680466.sHtML
tv.blog.yqvyet.cn/Article/details/088822.sHtML
tv.blog.yqvyet.cn/Article/details/244046.sHtML
tv.blog.yqvyet.cn/Article/details/064204.sHtML
tv.blog.yqvyet.cn/Article/details/733993.sHtML
tv.blog.yqvyet.cn/Article/details/759571.sHtML
tv.blog.yqvyet.cn/Article/details/648008.sHtML
tv.blog.yqvyet.cn/Article/details/862688.sHtML
tv.blog.yqvyet.cn/Article/details/571139.sHtML
tv.blog.yqvyet.cn/Article/details/999175.sHtML
tv.blog.yqvyet.cn/Article/details/753953.sHtML
tv.blog.yqvyet.cn/Article/details/939331.sHtML
tv.blog.yqvyet.cn/Article/details/248448.sHtML
tv.blog.yqvyet.cn/Article/details/757593.sHtML
tv.blog.yqvyet.cn/Article/details/315155.sHtML
tv.blog.yqvyet.cn/Article/details/660620.sHtML
tv.blog.yqvyet.cn/Article/details/222866.sHtML
tv.blog.yqvyet.cn/Article/details/111397.sHtML
tv.blog.yqvyet.cn/Article/details/159199.sHtML
tv.blog.yqvyet.cn/Article/details/337337.sHtML
tv.blog.yqvyet.cn/Article/details/428682.sHtML
tv.blog.yqvyet.cn/Article/details/064828.sHtML
tv.blog.yqvyet.cn/Article/details/606826.sHtML
tv.blog.yqvyet.cn/Article/details/624848.sHtML
tv.blog.yqvyet.cn/Article/details/204088.sHtML
tv.blog.yqvyet.cn/Article/details/240608.sHtML
tv.blog.yqvyet.cn/Article/details/204602.sHtML
tv.blog.yqvyet.cn/Article/details/004222.sHtML
tv.blog.yqvyet.cn/Article/details/868264.sHtML
tv.blog.yqvyet.cn/Article/details/953553.sHtML
tv.blog.yqvyet.cn/Article/details/666800.sHtML
tv.blog.yqvyet.cn/Article/details/422600.sHtML
