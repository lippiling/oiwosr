婪诎矫费亓


Python functools 深度应用
===============================

functools 提供高阶函数工具, 用于函数式编程、缓存、分发和装饰器。

1. lru_cache — 最近最少使用缓存
-----------------------------------

from functools import lru_cache
import time

# maxsize 设置缓存容量, typed=True 区分不同类型参数
@lru_cache(maxsize=128, typed=True)
def fibonacci(n):
    """递归斐波那契, 使用缓存避免重复计算"""
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

start = time.perf_counter()
print("fibonacci(35):", fibonacci(35))
print("计算耗时:", time.perf_counter() - start)  # 极快, 因为有缓存
print("缓存信息:", fibonacci.cache_info())       # CacheInfo(hits=..., misses=..., ...)
# cache_info() 返回命中/未命中/最大/当前大小

# 查看和清除缓存
print("缓存参数:", fibonacci.cache_parameters()) # {'maxsize': 128, 'typed': True}
fibonacci.cache_clear()                          # 清空缓存

# typed=True 时, 1 和 1.0 被视为不同参数
@lru_cache(typed=True)
def multiply(a, b):
    return a * b

multiply(1, 2)
multiply(1.0, 2)
print("typed 区分 int/float:", multiply.cache_info().misses)  # 2 次未命中

2. cache (Python 3.9+) — 无大小限制的缓存
--------------------------------------------

from functools import cache

@cache
def expensive_compute(n):
    """等价于 @lru_cache(maxsize=None), 内存允许时无限缓存"""
    print(f"  计算 {n}...")
    return sum(range(n))

print("首次调用:", expensive_compute(10_000_000))  # 实际计算
print("缓存调用:", expensive_compute(10_000_000))  # 直接从缓存返回

3. cached_property — 惰性计算属性
------------------------------------

from functools import cached_property
import math

class Circle:
    def __init__(self, radius):
        self.radius = radius

    @cached_property
    def area(self):
        """首次访问时计算并缓存, 后续直接返回缓存值"""
        print("  计算面积...")
        return math.pi * self.radius ** 2

    @cached_property
    def circumference(self):
        print("  计算周长...")
        return 2 * math.pi * self.radius

c = Circle(5)
print("面积:", c.area)         # 触发计算
print("面积:", c.area)         # 直接从缓存读取, 不重复计算
print("周长:", c.circumference) # 触发计算

# cached_property 在对象生命周期内有效, 修改 radius 不会自动更新
c.radius = 10
print("修改后面积(缓存旧值):", c.area)  # 仍是之前的结果

# 删除属性可清除缓存, 下次访问重新计算
del c.area
print("清除后面积:", c.area)   # 重新计算

4. singledispatch — 单参数类型分发
---------------------------------------

from functools import singledispatch

@singledispatch
def serialize(obj):
    """默认序列化函数, 抛出类型错误"""
    raise TypeError(f"不支持的类型: {type(obj)}")

# 为特定类型注册实现
@serialize.register(int)
def _(obj):
    return f"int:{obj}"

@serialize.register(str)
def _(obj):
    return f"str:'{obj}'"

@serialize.register(list)
@serialize.register(tuple)      # 多个类型注册同一个实现
def _(obj):
    items = ", ".join(serialize(item) for item in obj)
    return f"list:[{items}]"

@serialize.register(dict)
def _(obj):
    pairs = ", ".join(f"{k}: {serialize(v)}" for k, v in obj.items())
    return f"dict:{{{pairs}}}"

# 注册 None 类型的处理
@serialize.register(type(None))
def _(obj):
    return "null"

print(serialize(42))           # int:42
print(serialize("hello"))      # str:'hello'
print(serialize([1, "a", None]))  # list:[int:1, str:'a', null]
print(serialize({"x": 10}))    # dict:{x: int:10}

# 查看所有注册类型
print("已注册的类型:", serialize.registry)

5. singledispatchmethod — 方法级别类型分发
---------------------------------------------

from functools import singledispatchmethod

class Processor:
    @singledispatchmethod
    def process(self, arg):
        """默认处理方法"""
        raise NotImplementedError(f"未支持类型: {type(arg)}")

    @process.register(int)
    def _(self, arg):
        return f"处理整数: {arg * 2}"

    @process.register(str)
    def _(self, arg):
        return f"处理字符串: {arg.upper()}"

    @process.register(list)
    def _(self, arg):
        return [self.process(item) for item in arg]

p = Processor()
print(p.process(5))            # 处理整数: 10
print(p.process("hello"))      # 处理字符串: HELLO
print(p.process([1, "a"]))     # 递归处理列表

6. partial / partialmethod — 偏函数
---------------------------------------

from functools import partial, partialmethod
import os

# partial: 固定函数的部分参数
def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)   # 固定 exponent=2
cube = partial(power, exponent=3)     # 固定 exponent=3
print("平方:", square(5))              # 25
print("立方:", cube(5))                # 125

# 实际应用: 读取文件时指定编码
read_utf8 = partial(open, encoding="utf-8", mode="r")
# 用 read_utf8("file.txt") 代替 open("file.txt", encoding="utf-8", mode="r")

# partialmethod: 用于类方法
class Cell:
    def __init__(self):
        self._alive = False

    def set_state(self, state):
        self._alive = state

    set_alive = partialmethod(set_state, True)
    set_dead = partialmethod(set_state, False)

cell = Cell()
cell.set_alive()            # 等价于 cell.set_state(True)
print("细胞存活:", cell._alive)

7. reduce — 累积归约
------------------------

from functools import reduce

# reduce 将二元函数累积应用到序列上
numbers = [1, 2, 3, 4, 5]
product = reduce(lambda a, b: a * b, numbers)
print("累乘结果:", product)               # 120

# reduce 支持初始值, 为空序列时返回初始值
sum_with_init = reduce(lambda a, b: a + b, [], 0)
print("空序列归约:", sum_with_init)       # 0

# 实战: 扁平化嵌套列表
nested = [[1, 2], [3, 4], [5]]
flat = reduce(lambda acc, x: acc + x, nested, [])
print("展平嵌套列表:", flat)              # [1, 2, 3, 4, 5]

8. wraps — 装饰器保留元数据
-------------------------------

from functools import wraps

def log_calls(func):
    @wraps(func)   # 保留原函数的 __name__, __doc__, __module__ 等
    def wrapper(*args, **kwargs):
        print(f"调用函数: {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log_calls
def greet(name):
    """向指定用户问好"""
    return f"你好, {name}!"

print(greet("小明"))         # 输出调用日志及问候
print("函数名:", greet.__name__)   # greet (没有 wraps 会变成 wrapper)
print("文档:", greet.__doc__.strip())  # 向指定用户问好

9. total_ordering — 自动补全比较方法
----------------------------------------

from functools import total_ordering

@total_ordering
class Version:
    """只需定义 __eq__ 和 __lt__, 其余比较操作自动生成"""
    def __init__(self, major, minor):
        self.major = major
        self.minor = minor

    def __eq__(self, other):
        return (self.major, self.minor) == (other.major, other.minor)

    def __lt__(self, other):
        return (self.major, self.minor) < (other.major, other.minor)

v1 = Version(2, 0)
v2 = Version(3, 1)
print("v1 < v2:", v1 < v2)   # True
print("v1 <= v2:", v1 <= v2)  # True (由 __lt__ 和 __eq__ 推导)
print("v1 > v2:", v1 > v2)    # False
print("v1 >= v2:", v1 >= v2)  # False

10. cmp_to_key — 旧式比较转键函数
------------------------------------

from functools import cmp_to_key

def custom_cmp(a, b):
    """自定义比较逻辑: 按长度优先, 同长按字母序"""
    if len(a) != len(b):
        return len(b) - len(a)   # 长度降序
    return -1 if a < b else (1 if a > b else 0)  # 字母升序

strings = ["banana", "apple", "cherry", "date", "elderberry"]
sorted_strings = sorted(strings, key=cmp_to_key(custom_cmp))
print("自定义排序:", sorted_strings)
# ['elderberry', 'banana', 'cherry', 'apple', 'date']

总结: functools 可优化性能(缓存)、提高复用(偏函数)、增强可读性(类型分发), 是高级 Python 编程的基石.

佬鞍窗字醋侥徒醋杉衔憾有贩百坷

rfs.hjiocz.cn/264603.htm
rfs.hjiocz.cn/551353.htm
rfs.hjiocz.cn/997313.htm
rfs.hjiocz.cn/620883.htm
rfs.hjiocz.cn/406043.htm
rfa.hjiocz.cn/177353.htm
rfa.hjiocz.cn/979333.htm
rfa.hjiocz.cn/642003.htm
rfa.hjiocz.cn/666083.htm
rfa.hjiocz.cn/199773.htm
rfa.hjiocz.cn/579153.htm
rfa.hjiocz.cn/222803.htm
rfa.hjiocz.cn/088083.htm
rfa.hjiocz.cn/997773.htm
rfa.hjiocz.cn/842423.htm
rfp.hjiocz.cn/408643.htm
rfp.hjiocz.cn/246663.htm
rfp.hjiocz.cn/446843.htm
rfp.hjiocz.cn/535913.htm
rfp.hjiocz.cn/319133.htm
rfp.hjiocz.cn/062643.htm
rfp.hjiocz.cn/391133.htm
rfp.hjiocz.cn/488663.htm
rfp.hjiocz.cn/379713.htm
rfp.hjiocz.cn/533173.htm
rfo.hjiocz.cn/486203.htm
rfo.hjiocz.cn/680403.htm
rfo.hjiocz.cn/919773.htm
rfo.hjiocz.cn/995393.htm
rfo.hjiocz.cn/648263.htm
rfo.hjiocz.cn/060043.htm
rfo.hjiocz.cn/260043.htm
rfo.hjiocz.cn/775173.htm
rfo.hjiocz.cn/848663.htm
rfo.hjiocz.cn/046483.htm
rfi.hjiocz.cn/573513.htm
rfi.hjiocz.cn/353173.htm
rfi.hjiocz.cn/064043.htm
rfi.hjiocz.cn/222283.htm
rfi.hjiocz.cn/711173.htm
rfi.hjiocz.cn/799533.htm
rfi.hjiocz.cn/044483.htm
rfi.hjiocz.cn/280863.htm
rfi.hjiocz.cn/939733.htm
rfi.hjiocz.cn/157973.htm
rfu.hjiocz.cn/244883.htm
rfu.hjiocz.cn/686843.htm
rfu.hjiocz.cn/315353.htm
rfu.hjiocz.cn/353373.htm
rfu.hjiocz.cn/080403.htm
rfu.hjiocz.cn/862843.htm
rfu.hjiocz.cn/537913.htm
rfu.hjiocz.cn/957793.htm
rfu.hjiocz.cn/331333.htm
rfu.hjiocz.cn/468843.htm
rfy.hjiocz.cn/444603.htm
rfy.hjiocz.cn/991753.htm
rfy.hjiocz.cn/951953.htm
rfy.hjiocz.cn/086663.htm
rfy.hjiocz.cn/228843.htm
rfy.hjiocz.cn/133533.htm
rfy.hjiocz.cn/579373.htm
rfy.hjiocz.cn/066223.htm
rfy.hjiocz.cn/000243.htm
rfy.hjiocz.cn/995373.htm
rft.hjiocz.cn/597593.htm
rft.hjiocz.cn/531953.htm
rft.hjiocz.cn/662603.htm
rft.hjiocz.cn/557113.htm
rft.hjiocz.cn/555133.htm
rft.hjiocz.cn/060023.htm
rft.hjiocz.cn/622043.htm
rft.hjiocz.cn/171513.htm
rft.hjiocz.cn/317733.htm
rft.hjiocz.cn/484003.htm
rfr.hjiocz.cn/822223.htm
rfr.hjiocz.cn/773973.htm
rfr.hjiocz.cn/420643.htm
rfr.hjiocz.cn/020443.htm
rfr.hjiocz.cn/602443.htm
rfr.hjiocz.cn/220063.htm
rfr.hjiocz.cn/753533.htm
rfr.hjiocz.cn/644843.htm
rfr.hjiocz.cn/888643.htm
rfr.hjiocz.cn/797113.htm
rfe.hjiocz.cn/426603.htm
rfe.hjiocz.cn/862023.htm
rfe.hjiocz.cn/862883.htm
rfe.hjiocz.cn/379993.htm
rfe.hjiocz.cn/573793.htm
rfe.hjiocz.cn/719353.htm
rfe.hjiocz.cn/284663.htm
rfe.hjiocz.cn/860263.htm
rfe.hjiocz.cn/957513.htm
rfe.hjiocz.cn/155133.htm
rfw.hjiocz.cn/068283.htm
rfw.hjiocz.cn/008663.htm
rfw.hjiocz.cn/646403.htm
rfw.hjiocz.cn/371753.htm
rfw.hjiocz.cn/406863.htm
rfw.hjiocz.cn/808803.htm
rfw.hjiocz.cn/339953.htm
rfw.hjiocz.cn/199513.htm
rfw.hjiocz.cn/408423.htm
rfw.hjiocz.cn/240463.htm
rfq.hjiocz.cn/175153.htm
rfq.hjiocz.cn/137953.htm
rfq.hjiocz.cn/442823.htm
rfq.hjiocz.cn/939173.htm
rfq.hjiocz.cn/333393.htm
rfq.hjiocz.cn/977733.htm
rfq.hjiocz.cn/206263.htm
rfq.hjiocz.cn/468463.htm
rfq.hjiocz.cn/622823.htm
rfq.hjiocz.cn/711753.htm
rdm.hjiocz.cn/600643.htm
rdm.hjiocz.cn/422843.htm
rdm.hjiocz.cn/557373.htm
rdm.hjiocz.cn/315593.htm
rdm.hjiocz.cn/622483.htm
rdm.hjiocz.cn/404443.htm
rdm.hjiocz.cn/159733.htm
rdm.hjiocz.cn/397333.htm
rdm.hjiocz.cn/864063.htm
rdm.hjiocz.cn/460043.htm
rdn.hjiocz.cn/713353.htm
rdn.hjiocz.cn/860483.htm
rdn.hjiocz.cn/359333.htm
rdn.hjiocz.cn/771393.htm
rdn.hjiocz.cn/864403.htm
rdn.hjiocz.cn/846243.htm
rdn.hjiocz.cn/840463.htm
rdn.hjiocz.cn/628483.htm
rdn.hjiocz.cn/444283.htm
rdn.hjiocz.cn/793373.htm
rdb.hjiocz.cn/022223.htm
rdb.hjiocz.cn/666823.htm
rdb.hjiocz.cn/080883.htm
rdb.hjiocz.cn/262263.htm
rdb.hjiocz.cn/288403.htm
rdb.hjiocz.cn/828623.htm
rdb.hjiocz.cn/397133.htm
rdb.hjiocz.cn/044643.htm
rdb.hjiocz.cn/088003.htm
rdb.hjiocz.cn/282203.htm
rdv.hjiocz.cn/135993.htm
rdv.hjiocz.cn/806223.htm
rdv.hjiocz.cn/082823.htm
rdv.hjiocz.cn/604043.htm
rdv.hjiocz.cn/886863.htm
rdv.hjiocz.cn/553573.htm
rdv.hjiocz.cn/468203.htm
rdv.hjiocz.cn/468643.htm
rdv.hjiocz.cn/973373.htm
rdv.hjiocz.cn/244083.htm
rdc.hjiocz.cn/464023.htm
rdc.hjiocz.cn/646403.htm
rdc.hjiocz.cn/757333.htm
rdc.hjiocz.cn/402443.htm
rdc.hjiocz.cn/119353.htm
rdc.hjiocz.cn/088823.htm
rdc.hjiocz.cn/826883.htm
rdc.hjiocz.cn/882243.htm
rdc.hjiocz.cn/195713.htm
rdc.hjiocz.cn/088823.htm
rdx.hjiocz.cn/971393.htm
rdx.hjiocz.cn/828283.htm
rdx.hjiocz.cn/662283.htm
rdx.hjiocz.cn/533513.htm
rdx.hjiocz.cn/846803.htm
rdx.hjiocz.cn/575553.htm
rdx.hjiocz.cn/488423.htm
rdx.hjiocz.cn/208423.htm
rdx.hjiocz.cn/082423.htm
rdx.hjiocz.cn/666883.htm
rdz.hjiocz.cn/844283.htm
rdz.hjiocz.cn/311593.htm
rdz.hjiocz.cn/846823.htm
rdz.hjiocz.cn/860063.htm
rdz.hjiocz.cn/442843.htm
rdz.hjiocz.cn/220883.htm
rdz.hjiocz.cn/200043.htm
rdz.hjiocz.cn/793333.htm
rdz.hjiocz.cn/242683.htm
rdz.hjiocz.cn/535593.htm
rdl.hjiocz.cn/591973.htm
rdl.hjiocz.cn/373333.htm
rdl.hjiocz.cn/208623.htm
rdl.hjiocz.cn/222403.htm
rdl.hjiocz.cn/084683.htm
rdl.hjiocz.cn/379953.htm
rdl.hjiocz.cn/313533.htm
rdl.hjiocz.cn/620463.htm
rdl.hjiocz.cn/664243.htm
rdl.hjiocz.cn/800663.htm
rdk.hjiocz.cn/062863.htm
rdk.hjiocz.cn/620643.htm
rdk.hjiocz.cn/797713.htm
rdk.hjiocz.cn/624483.htm
rdk.hjiocz.cn/204063.htm
rdk.hjiocz.cn/460003.htm
rdk.hjiocz.cn/371193.htm
rdk.hjiocz.cn/826483.htm
rdk.hjiocz.cn/711753.htm
rdk.hjiocz.cn/519733.htm
rdj.hjiocz.cn/402843.htm
rdj.hjiocz.cn/955193.htm
rdj.hjiocz.cn/480403.htm
rdj.hjiocz.cn/997773.htm
rdj.hjiocz.cn/262483.htm
rdj.hjiocz.cn/600043.htm
rdj.hjiocz.cn/519193.htm
rdj.hjiocz.cn/264203.htm
rdj.hjiocz.cn/713153.htm
rdj.hjiocz.cn/824223.htm
rdh.hjiocz.cn/482023.htm
rdh.hjiocz.cn/604843.htm
rdh.hjiocz.cn/486623.htm
rdh.hjiocz.cn/246203.htm
rdh.hjiocz.cn/006063.htm
rdh.hjiocz.cn/191333.htm
rdh.hjiocz.cn/446023.htm
rdh.hjiocz.cn/626003.htm
rdh.hjiocz.cn/000603.htm
rdh.hjiocz.cn/824203.htm
rdg.hjiocz.cn/997933.htm
rdg.hjiocz.cn/426203.htm
rdg.hjiocz.cn/282883.htm
rdg.hjiocz.cn/717713.htm
rdg.hjiocz.cn/020483.htm
rdg.hjiocz.cn/826863.htm
rdg.hjiocz.cn/793573.htm
rdg.hjiocz.cn/915513.htm
rdg.hjiocz.cn/280643.htm
rdg.hjiocz.cn/131993.htm
rdf.hjiocz.cn/026823.htm
rdf.hjiocz.cn/537513.htm
rdf.hjiocz.cn/737513.htm
rdf.hjiocz.cn/153113.htm
rdf.hjiocz.cn/935353.htm
rdf.hjiocz.cn/757933.htm
rdf.hjiocz.cn/080403.htm
rdf.hjiocz.cn/337733.htm
rdf.hjiocz.cn/179553.htm
rdf.hjiocz.cn/202683.htm
rdd.hjiocz.cn/848683.htm
rdd.hjiocz.cn/377973.htm
rdd.hjiocz.cn/579713.htm
rdd.hjiocz.cn/397153.htm
rdd.hjiocz.cn/155933.htm
rdd.hjiocz.cn/399953.htm
rdd.hjiocz.cn/979913.htm
rdd.hjiocz.cn/779393.htm
rdd.hjiocz.cn/155333.htm
rdd.hjiocz.cn/993993.htm
rds.hjiocz.cn/713713.htm
rds.hjiocz.cn/599753.htm
rds.hjiocz.cn/333113.htm
rds.hjiocz.cn/735193.htm
rds.hjiocz.cn/660023.htm
rds.hjiocz.cn/139773.htm
rds.hjiocz.cn/779713.htm
rds.hjiocz.cn/115153.htm
rds.hjiocz.cn/537953.htm
rds.hjiocz.cn/775593.htm
rda.hjiocz.cn/355133.htm
rda.hjiocz.cn/773993.htm
rda.hjiocz.cn/913153.htm
rda.hjiocz.cn/175173.htm
rda.hjiocz.cn/313173.htm
rda.hjiocz.cn/737313.htm
rda.hjiocz.cn/593513.htm
rda.hjiocz.cn/317173.htm
rda.hjiocz.cn/973333.htm
rda.hjiocz.cn/379333.htm
rdp.hjiocz.cn/991173.htm
rdp.hjiocz.cn/551353.htm
rdp.hjiocz.cn/559533.htm
rdp.hjiocz.cn/335193.htm
rdp.hjiocz.cn/511373.htm
rdp.hjiocz.cn/513953.htm
rdp.hjiocz.cn/799313.htm
rdp.hjiocz.cn/553933.htm
rdp.hjiocz.cn/028883.htm
rdp.hjiocz.cn/739733.htm
rdo.hjiocz.cn/468683.htm
rdo.hjiocz.cn/913313.htm
rdo.hjiocz.cn/373933.htm
rdo.hjiocz.cn/600043.htm
rdo.hjiocz.cn/519193.htm
rdo.hjiocz.cn/553353.htm
rdo.hjiocz.cn/262883.htm
rdo.hjiocz.cn/397973.htm
rdo.hjiocz.cn/044463.htm
rdo.hjiocz.cn/680443.htm
rdi.hjiocz.cn/260403.htm
rdi.hjiocz.cn/284003.htm
rdi.hjiocz.cn/488463.htm
rdi.hjiocz.cn/840683.htm
rdi.hjiocz.cn/024283.htm
rdi.hjiocz.cn/537973.htm
rdi.hjiocz.cn/939713.htm
rdi.hjiocz.cn/264403.htm
rdi.hjiocz.cn/024083.htm
rdi.hjiocz.cn/535993.htm
rdu.hjiocz.cn/620223.htm
rdu.hjiocz.cn/119393.htm
rdu.hjiocz.cn/060223.htm
rdu.hjiocz.cn/197553.htm
rdu.hjiocz.cn/264843.htm
rdu.hjiocz.cn/119353.htm
rdu.hjiocz.cn/733393.htm
rdu.hjiocz.cn/082863.htm
rdu.hjiocz.cn/486443.htm
rdu.hjiocz.cn/959173.htm
rdy.hjiocz.cn/753973.htm
rdy.hjiocz.cn/519133.htm
rdy.hjiocz.cn/868063.htm
rdy.hjiocz.cn/682063.htm
rdy.hjiocz.cn/579553.htm
rdy.hjiocz.cn/840403.htm
rdy.hjiocz.cn/159173.htm
rdy.hjiocz.cn/646683.htm
rdy.hjiocz.cn/248063.htm
rdy.hjiocz.cn/799533.htm
rdt.hjiocz.cn/755773.htm
rdt.hjiocz.cn/486063.htm
rdt.hjiocz.cn/779133.htm
rdt.hjiocz.cn/888683.htm
rdt.hjiocz.cn/793133.htm
rdt.hjiocz.cn/917333.htm
rdt.hjiocz.cn/826063.htm
rdt.hjiocz.cn/468803.htm
rdt.hjiocz.cn/044223.htm
rdt.hjiocz.cn/319333.htm
rdr.hjiocz.cn/684263.htm
rdr.hjiocz.cn/444003.htm
rdr.hjiocz.cn/537333.htm
rdr.hjiocz.cn/608403.htm
rdr.hjiocz.cn/620063.htm
rdr.hjiocz.cn/151113.htm
rdr.hjiocz.cn/004403.htm
rdr.hjiocz.cn/428063.htm
rdr.hjiocz.cn/084623.htm
rdr.hjiocz.cn/559993.htm
rde.hjiocz.cn/882423.htm
rde.hjiocz.cn/466643.htm
rde.hjiocz.cn/848243.htm
rde.hjiocz.cn/511973.htm
rde.hjiocz.cn/888483.htm
rde.hjiocz.cn/797513.htm
rde.hjiocz.cn/886023.htm
rde.hjiocz.cn/595993.htm
rde.hjiocz.cn/488803.htm
rde.hjiocz.cn/446063.htm
rdw.hjiocz.cn/828043.htm
rdw.hjiocz.cn/240603.htm
rdw.hjiocz.cn/286223.htm
rdw.hjiocz.cn/022803.htm
rdw.hjiocz.cn/240803.htm
rdw.hjiocz.cn/157333.htm
rdw.hjiocz.cn/066863.htm
rdw.hjiocz.cn/664283.htm
rdw.hjiocz.cn/488063.htm
rdw.hjiocz.cn/462803.htm
rdq.hjiocz.cn/040683.htm
rdq.hjiocz.cn/042823.htm
rdq.hjiocz.cn/333733.htm
rdq.hjiocz.cn/688063.htm
rdq.hjiocz.cn/208223.htm
rdq.hjiocz.cn/319993.htm
rdq.hjiocz.cn/888843.htm
rdq.hjiocz.cn/319793.htm
rdq.hjiocz.cn/208863.htm
rdq.hjiocz.cn/119773.htm
rsm.hjiocz.cn/464663.htm
rsm.hjiocz.cn/204683.htm
rsm.hjiocz.cn/333993.htm
rsm.hjiocz.cn/024603.htm
rsm.hjiocz.cn/119993.htm
rsm.hjiocz.cn/662263.htm
rsm.hjiocz.cn/822423.htm
rsm.hjiocz.cn/593933.htm
rsm.hjiocz.cn/440023.htm
rsm.hjiocz.cn/480243.htm
rsn.hjiocz.cn/993733.htm
rsn.hjiocz.cn/595973.htm
rsn.hjiocz.cn/242283.htm
rsn.hjiocz.cn/640843.htm
rsn.hjiocz.cn/159593.htm
rsn.hjiocz.cn/199713.htm
rsn.hjiocz.cn/379973.htm
rsn.hjiocz.cn/846403.htm
rsn.hjiocz.cn/866863.htm
rsn.hjiocz.cn/842803.htm
