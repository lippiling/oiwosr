斜昭痪月智


Python copy 深拷贝与浅拷贝
===============================

copy 模块提供浅拷贝和深拷贝操作, 理解两者的区别对避免副作用至关重要。

1. 浅拷贝 vs 深拷贝: 核心概念
--------------------------------

import copy

# 浅拷贝: 创建新对象, 但只复制一层引用, 嵌套对象仍共享
# 深拷贝: 递归复制所有嵌套对象, 完全独立

# 基本类型 (int, str, float, bool) 是不可变的,
# 浅拷贝和深拷贝对它们没有实际区别
a = 42
b = copy.copy(a)
print("基本类型浅拷贝:", a is b)                  # True (小整数被缓存, 但不是所有情况)

# 主要区别体现在复合对象: 列表, 字典, 自定义对象
nested = [[1, 2], [3, 4]]
shallow = copy.copy(nested)
deep = copy.deepcopy(nested)

# 浅拷贝: 外层是新列表, 但内层列表是同一对象
print("浅拷贝外层独立:", nested is shallow)       # False
print("浅拷贝内层共享:", nested[0] is shallow[0]) # True

# 深拷贝: 内外层都是独立对象
print("深拷贝外层独立:", nested is deep)          # False
print("深拷贝内层独立:", nested[0] is deep[0])    # False

2. copy.copy vs copy.deepcopy — 实战演示
------------------------------------------

import copy

original = {
    "name": "Alice",
    "scores": [85, 92, 78],
    "address": {"city": "北京", "zip": "100000"}
}

# 浅拷贝
shallow_copy = copy.copy(original)

# 深拷贝
deep_copy = copy.deepcopy(original)

# 修改浅拷贝中的嵌套对象
shallow_copy["scores"].append(99)               # 同时影响原始对象!
shallow_copy["address"]["city"] = "上海"         # 同时影响原始对象!
print("修改浅拷贝后:")
print("  原始对象:", original)                   # scores 多了 99, city 变成了上海
print("  浅拷贝:", shallow_copy)

# 修改深拷贝中的嵌套对象
deep_copy["scores"].append(100)
deep_copy["address"]["city"] = "广州"
print("\n修改深拷贝后:")
print("  原始对象:", original)                   # 不受影响
print("  深拷贝:", deep_copy)                    # 独立变化

# 浅拷贝顶层独立: 替换顶层键不影响原始
shallow_copy["name"] = "Bob"                    # 只影响浅拷贝
print("顶层替换不影响原始:", original["name"])    # 仍是 Alice

3. __copy__ 和 __deepcopy__ 自定义协议
------------------------------------------

import copy

class DatabaseConnection:
    """模拟数据库连接, 浅拷贝共享连接, 深拷贝创建新连接"""

    def __init__(self, conn_str: str):
        self.conn_str = conn_str
        self.connected = False
        self.transactions = []                   # 事务列表, 应独立

    def connect(self):
        print(f"连接数据库: {self.conn_str}")
        self.connected = True

    def __copy__(self):
        """浅拷贝: 复用相同的连接字符串但不共享事务"""
        new = type(self)(self.conn_str)
        new.connected = self.connected           # 共享连接状态
        return new                               # transactions 是空列表

    def __deepcopy__(self, memo):
        """深拷贝: 完全独立的事务列表"""
        new = type(self)(self.conn_str)
        new.connected = self.connected
        new.transactions = copy.deepcopy(self.transactions, memo)
        return new

    def add_transaction(self, txn):
        self.transactions.append(txn)

conn = DatabaseConnection("postgres://localhost/db")
conn.connect()
conn.add_transaction("TXN001")

# 浅拷贝: transactions 独立 (因为 __copy__ 返回空列表)
shallow_conn = copy.copy(conn)
shallow_conn.add_transaction("TXN002")
print("原始事务 (浅拷贝不影响):", conn.transactions)    # ['TXN001']
print("浅拷贝事务:", shallow_conn.transactions)         # ['TXN002']

# 深拷贝: 完全独立
deep_conn = copy.deepcopy(conn)
deep_conn.add_transaction("TXN003")
print("原始事务 (深拷贝不影响):", conn.transactions)    # ['TXN001']

# 另一个例子: 单例模式对象应覆盖 __copy__ 和 __deepcopy__
class Singleton:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    def __copy__(self):
        return self                               # 返回单例自身
    def __deepcopy__(self, memo):
        return self                               # 返回单例自身

s1 = Singleton()
s2 = copy.copy(s1)
s3 = copy.deepcopy(s1)
print("单例浅拷贝:", s1 is s2)                     # True
print("单例深拷贝:", s1 is s3)                     # True

4. 浅拷贝的常见方式
---------------------

# 方式1: copy.copy
import copy
list_a = [1, [2, 3]]
list_b = copy.copy(list_a)

# 方式2: 列表切片 ([:] 就是浅拷贝)
list_c = list_a[:]
print("切片浅拷贝:", list_a[1] is list_c[1])      # True

# 方式3: list() 构造函数
list_d = list(list_a)
print("list() 浅拷贝:", list_a[1] is list_d[1])   # True

# 方式4: dict.copy() 方法
dict_a = {"key": [1, 2]}
dict_b = dict_a.copy()
print("dict.copy() 浅拷贝:", dict_a["key"] is dict_b["key"])  # True

# 方式5: 列表的 copy() 方法 (Python 3.3+)
list_e = list_a.copy()
print("list.copy() 浅拷贝:", list_a[1] is list_e[1])  # True

# 方式6: 集合同样浅拷贝
set_a = {1, 2, 3}
set_b = set_a.copy()
print("set.copy() 浅拷贝:", set_a is set_b)       # False

5. deepcopy 的 memo 字典机制
-------------------------------

import copy

# deepcopy 使用 memo 字典跟踪已复制的对象, 避免:
# 1. 重复拷贝同一对象
# 2. 对象间的循环引用导致无限递归

class Node:
    def __init__(self, name):
        self.name = name
        self.children = []
        self.parent = None

# 构建循环引用图
root = Node("root")
child = Node("child")
root.children.append(child)
child.parent = root                              # 循环引用: root -> child -> root

# deepcopy 能正确处理循环引用
copied_root = copy.deepcopy(root)
copied_child = copied_root.children[0]
print("深拷贝循环引用成功:", copied_child.parent.name)  # root
print("循环引用结构保持一致:", copied_child.parent is copied_root)  # True

# 理解 memo 字典的工作方式
original_list = [1, 2, 3]
memo = {}
# 手动模拟 deepcopy 的 memo 跟踪
# memo 的键是原始对象的 id, 值是已创建的副本
copied = copy.deepcopy(original_list, memo)      # 内部使用 memo
print("Memo 跟踪:", id(original_list) in memo)   # True

# 在 __deepcopy__ 中手动管理 memo
class ManagedNode:
    def __init__(self, name):
        self.name = name
        self.ref = None

    def __deepcopy__(self, memo):
        # 先检查 memo 是否已有该对象的副本
        if id(self) in memo:
            return memo[id(self)]
        new = ManagedNode(copy.deepcopy(self.name, memo))
        memo[id(self)] = new
        # ref 字段稍后由外部设置
        return new

a = ManagedNode("A")
b = ManagedNode("B")
a.ref = b
b.ref = a                                       # 双向循环引用

copied_a = copy.deepcopy(a)
print("ManagedNode 深拷贝:", copied_a.ref.ref.name)  # A

6. 深拷贝的限制
------------------

import copy

# 1. 单例对象: deepcopy 默认返回单例自身
singleton = None                                 # None 是单例
print("None 深拷贝:", copy.deepcopy(singleton) is singleton)  # True

# 其他单例: True, False, Ellipsis, NotImplemented
print("True 深拷贝:", copy.deepcopy(True) is True)           # True

# 2. 模块: 不可深拷贝
import math
try:
    copy.deepcopy(math)
except TypeError as e:
    print("模块不可深拷贝:", e)

# 3. 文件句柄等系统资源: 不可拷贝
try:
    copy.deepcopy(open)
except TypeError as e:
    print("打开文件不可拷贝:", e)  # 但函数本身可拷贝引用, 实际打开的文件不可

# 4. 包含自定义 __slots__ 且没有 __dict__ 的对象
class SlotsOnly:
    __slots__ = ("x",)
    def __init__(self, x):
        self.x = x

s = SlotsOnly(10)
try:
    s_copy = copy.deepcopy(s)
    print("__slots__ 对象深拷贝:", s_copy.x)     # 可正常工作
except:
    print("__slots__ 深拷贝失败")

# 但如果没有 __dict__ 且自定义 __getstate__/__setstate__ 可能有问题

# 5. 递归限制
import sys
print("默认递归深度:", sys.getrecursionlimit())  # 默认 1000
# 极端深嵌套可用递归深度限制保护:
class DeepNode:
    def __init__(self, depth=0):
        self.depth = depth
        self.next = DeepNode(depth + 1) if depth < 500 else None
# 超过递归深度的 deepcopy 会 RecursionError

7. 性能对比
--------------

import copy
import time

# 构建一个大对象
data = {
    "level1": [
        {"name": f"item_{i}", "values": list(range(100))}
        for i in range(100)
    ],
    "metadata": {"created": "2024-01-01", "version": "1.0"},
}

# 浅拷贝性能
start = time.perf_counter()
for _ in range(1000):
    s = copy.copy(data)
shallow_time = time.perf_counter() - start

# 深拷贝性能
start = time.perf_counter()
for _ in range(1000):
    d = copy.deepcopy(data)
deep_time = time.perf_counter() - start

print(f"浅拷贝 1000 次: {shallow_time*1000:.1f} ms")
print(f"深拷贝 1000 次: {deep_time*1000:.1f} ms")
print(f"深拷贝/浅拷贝 速度比: {deep_time/shallow_time:.1f}x")

# 手动实现浅拷贝 vs 深拷贝的时间差异根源
# 浅拷贝: 只复制顶层结构 (字典本身的哈希表)
# 深拷贝: 递归访问每个嵌套对象, 对每个元素调用 copy

# 性能优化: 配合 __deepcopy__ 自定义加速
class OptimizedData:
    def __init__(self, data):
        self.data = data
        self.cache = None

    def __deepcopy__(self, memo):
        # 只深拷贝重要的 data 字段, 跳过 cache
        new = OptimizedData(copy.deepcopy(self.data, memo))
        # cache 不拷贝, 由新对象按需生成
        return new

print("\n深拷贝优化建议:")
print("  - 实现 __copy__/__deepcopy__ 跳过不需要复制的字段")
print("  - 小对象用浅拷贝即可, 避免不必要的深拷贝")
print("  - 对不可变嵌套结构考虑使用共享引用而非拷贝")

总结: 浅拷贝经济但共享嵌套对象; 深拷贝完整独立但成本高; 通过自定义协议可平衡二者.

侄凹侄猩纯妥锥恋猛汹下偻丈记疑

fvw.nfsid.cn/500892.Doc
fvw.nfsid.cn/446000.Doc
fvw.nfsid.cn/462222.Doc
fvw.nfsid.cn/244828.Doc
fvw.nfsid.cn/609234.Doc
fvq.nfsid.cn/123778.Doc
fvq.nfsid.cn/486484.Doc
fvq.nfsid.cn/844808.Doc
fvq.nfsid.cn/929734.Doc
fvq.nfsid.cn/242600.Doc
fvq.nfsid.cn/422420.Doc
fvq.nfsid.cn/464004.Doc
fvq.nfsid.cn/886688.Doc
fvq.nfsid.cn/846284.Doc
fvq.nfsid.cn/932598.Doc
fcm.nfsid.cn/082404.Doc
fcm.nfsid.cn/484266.Doc
fcm.nfsid.cn/462082.Doc
fcm.nfsid.cn/604022.Doc
fcm.nfsid.cn/242406.Doc
fcm.nfsid.cn/220282.Doc
fcm.nfsid.cn/824668.Doc
fcm.nfsid.cn/202484.Doc
fcm.nfsid.cn/006846.Doc
fcm.nfsid.cn/424284.Doc
fcn.nfsid.cn/409650.Doc
fcn.nfsid.cn/844240.Doc
fcn.nfsid.cn/042000.Doc
fcn.nfsid.cn/088266.Doc
fcn.nfsid.cn/280206.Doc
fcn.nfsid.cn/442616.Doc
fcn.nfsid.cn/408084.Doc
fcn.nfsid.cn/460028.Doc
fcn.nfsid.cn/600802.Doc
fcn.nfsid.cn/624040.Doc
fcb.nfsid.cn/093062.Doc
fcb.nfsid.cn/008468.Doc
fcb.nfsid.cn/804208.Doc
fcb.nfsid.cn/844200.Doc
fcb.nfsid.cn/068228.Doc
fcb.nfsid.cn/806884.Doc
fcb.nfsid.cn/448202.Doc
fcb.nfsid.cn/888082.Doc
fcb.nfsid.cn/323616.Doc
fcb.nfsid.cn/000802.Doc
fcv.nfsid.cn/682862.Doc
fcv.nfsid.cn/280608.Doc
fcv.nfsid.cn/840246.Doc
fcv.nfsid.cn/488043.Doc
fcv.nfsid.cn/400080.Doc
fcv.nfsid.cn/366579.Doc
fcv.nfsid.cn/688066.Doc
fcv.nfsid.cn/867820.Doc
fcv.nfsid.cn/761302.Doc
fcv.nfsid.cn/842662.Doc
fcc.nfsid.cn/082442.Doc
fcc.nfsid.cn/139807.Doc
fcc.nfsid.cn/600482.Doc
fcc.nfsid.cn/884084.Doc
fcc.nfsid.cn/288406.Doc
fcc.nfsid.cn/266068.Doc
fcc.nfsid.cn/048488.Doc
fcc.nfsid.cn/022204.Doc
fcc.nfsid.cn/179373.Doc
fcc.nfsid.cn/737155.Doc
fcx.nfsid.cn/448284.Doc
fcx.nfsid.cn/226600.Doc
fcx.nfsid.cn/420464.Doc
fcx.nfsid.cn/826242.Doc
fcx.nfsid.cn/044222.Doc
fcx.nfsid.cn/282600.Doc
fcx.nfsid.cn/082460.Doc
fcx.nfsid.cn/446642.Doc
fcx.nfsid.cn/068848.Doc
fcx.nfsid.cn/260666.Doc
fcz.nfsid.cn/824622.Doc
fcz.nfsid.cn/200420.Doc
fcz.nfsid.cn/607676.Doc
fcz.nfsid.cn/202064.Doc
fcz.nfsid.cn/402620.Doc
fcz.nfsid.cn/620688.Doc
fcz.nfsid.cn/806248.Doc
fcz.nfsid.cn/575519.Doc
fcz.nfsid.cn/428846.Doc
fcz.nfsid.cn/440408.Doc
fcl.irvnp.cn/648424.Doc
fcl.irvnp.cn/044606.Doc
fcl.irvnp.cn/934296.Doc
fcl.irvnp.cn/335991.Doc
fcl.irvnp.cn/026220.Doc
fcl.irvnp.cn/460231.Doc
fcl.irvnp.cn/555953.Doc
fcl.irvnp.cn/804086.Doc
fcl.irvnp.cn/026488.Doc
fcl.irvnp.cn/620680.Doc
fck.irvnp.cn/264082.Doc
fck.irvnp.cn/670123.Doc
fck.irvnp.cn/262482.Doc
fck.irvnp.cn/286426.Doc
fck.irvnp.cn/600604.Doc
fck.irvnp.cn/406068.Doc
fck.irvnp.cn/086862.Doc
fck.irvnp.cn/226466.Doc
fck.irvnp.cn/282440.Doc
fck.irvnp.cn/222664.Doc
fcj.irvnp.cn/393605.Doc
fcj.irvnp.cn/244648.Doc
fcj.irvnp.cn/646424.Doc
fcj.irvnp.cn/113315.Doc
fcj.irvnp.cn/064244.Doc
fcj.irvnp.cn/593331.Doc
fcj.irvnp.cn/061727.Doc
fcj.irvnp.cn/601127.Doc
fcj.irvnp.cn/204404.Doc
fcj.irvnp.cn/206064.Doc
fch.irvnp.cn/622860.Doc
fch.irvnp.cn/462246.Doc
fch.irvnp.cn/040048.Doc
fch.irvnp.cn/800248.Doc
fch.irvnp.cn/404219.Doc
fch.irvnp.cn/204646.Doc
fch.irvnp.cn/608020.Doc
fch.irvnp.cn/686088.Doc
fch.irvnp.cn/686468.Doc
fch.irvnp.cn/402264.Doc
fcg.irvnp.cn/282284.Doc
fcg.irvnp.cn/513595.Doc
fcg.irvnp.cn/379339.Doc
fcg.irvnp.cn/066282.Doc
fcg.irvnp.cn/204264.Doc
fcg.irvnp.cn/371957.Doc
fcg.irvnp.cn/484282.Doc
fcg.irvnp.cn/044866.Doc
fcg.irvnp.cn/278579.Doc
fcg.irvnp.cn/224004.Doc
fcf.irvnp.cn/006424.Doc
fcf.irvnp.cn/680604.Doc
fcf.irvnp.cn/640088.Doc
fcf.irvnp.cn/004062.Doc
fcf.irvnp.cn/800268.Doc
fcf.irvnp.cn/286828.Doc
fcf.irvnp.cn/331511.Doc
fcf.irvnp.cn/323720.Doc
fcf.irvnp.cn/604286.Doc
fcf.irvnp.cn/626244.Doc
fcd.irvnp.cn/446606.Doc
fcd.irvnp.cn/133159.Doc
fcd.irvnp.cn/222804.Doc
fcd.irvnp.cn/591195.Doc
fcd.irvnp.cn/086426.Doc
fcd.irvnp.cn/886420.Doc
fcd.irvnp.cn/006404.Doc
fcd.irvnp.cn/602686.Doc
fcd.irvnp.cn/826626.Doc
fcd.irvnp.cn/284820.Doc
fcs.irvnp.cn/206228.Doc
fcs.irvnp.cn/715137.Doc
fcs.irvnp.cn/046880.Doc
fcs.irvnp.cn/446442.Doc
fcs.irvnp.cn/066646.Doc
fcs.irvnp.cn/620446.Doc
fcs.irvnp.cn/804668.Doc
fcs.irvnp.cn/937957.Doc
fcs.irvnp.cn/444686.Doc
fcs.irvnp.cn/544021.Doc
fca.irvnp.cn/822404.Doc
fca.irvnp.cn/860444.Doc
fca.irvnp.cn/680882.Doc
fca.irvnp.cn/513195.Doc
fca.irvnp.cn/888260.Doc
fca.irvnp.cn/439743.Doc
fca.irvnp.cn/266648.Doc
fca.irvnp.cn/640268.Doc
fca.irvnp.cn/535810.Doc
fca.irvnp.cn/680640.Doc
fcp.irvnp.cn/608280.Doc
fcp.irvnp.cn/822046.Doc
fcp.irvnp.cn/502346.Doc
fcp.irvnp.cn/608284.Doc
fcp.irvnp.cn/536569.Doc
fcp.irvnp.cn/064428.Doc
fcp.irvnp.cn/860804.Doc
fcp.irvnp.cn/486866.Doc
fcp.irvnp.cn/862628.Doc
fcp.irvnp.cn/024282.Doc
fco.irvnp.cn/086688.Doc
fco.irvnp.cn/543213.Doc
fco.irvnp.cn/068262.Doc
fco.irvnp.cn/400840.Doc
fco.irvnp.cn/886013.Doc
fco.irvnp.cn/680880.Doc
fco.irvnp.cn/828486.Doc
fco.irvnp.cn/462464.Doc
fco.irvnp.cn/804022.Doc
fco.irvnp.cn/666440.Doc
fci.irvnp.cn/008264.Doc
fci.irvnp.cn/248444.Doc
fci.irvnp.cn/488286.Doc
fci.irvnp.cn/604820.Doc
fci.irvnp.cn/240060.Doc
fci.irvnp.cn/200864.Doc
fci.irvnp.cn/662000.Doc
fci.irvnp.cn/026066.Doc
fci.irvnp.cn/020951.Doc
fci.irvnp.cn/288042.Doc
fcu.irvnp.cn/288060.Doc
fcu.irvnp.cn/602002.Doc
fcu.irvnp.cn/068004.Doc
fcu.irvnp.cn/624262.Doc
fcu.irvnp.cn/880888.Doc
fcu.irvnp.cn/380171.Doc
fcu.irvnp.cn/264980.Doc
fcu.irvnp.cn/086448.Doc
fcu.irvnp.cn/422828.Doc
fcu.irvnp.cn/682862.Doc
fcy.irvnp.cn/682524.Doc
fcy.irvnp.cn/286628.Doc
fcy.irvnp.cn/442288.Doc
fcy.irvnp.cn/080082.Doc
fcy.irvnp.cn/541809.Doc
fcy.irvnp.cn/208408.Doc
fcy.irvnp.cn/082482.Doc
fcy.irvnp.cn/181666.Doc
fcy.irvnp.cn/862024.Doc
fcy.irvnp.cn/864668.Doc
fct.irvnp.cn/711915.Doc
fct.irvnp.cn/626084.Doc
fct.irvnp.cn/193531.Doc
fct.irvnp.cn/200626.Doc
fct.irvnp.cn/866828.Doc
fct.irvnp.cn/863489.Doc
fct.irvnp.cn/068246.Doc
fct.irvnp.cn/064088.Doc
fct.irvnp.cn/798886.Doc
fct.irvnp.cn/826428.Doc
fcr.irvnp.cn/606884.Doc
fcr.irvnp.cn/337735.Doc
fcr.irvnp.cn/666420.Doc
fcr.irvnp.cn/080000.Doc
fcr.irvnp.cn/331348.Doc
fcr.irvnp.cn/084444.Doc
fcr.irvnp.cn/802422.Doc
fcr.irvnp.cn/680222.Doc
fcr.irvnp.cn/925496.Doc
fcr.irvnp.cn/600646.Doc
fce.irvnp.cn/300626.Doc
fce.irvnp.cn/908237.Doc
fce.irvnp.cn/187989.Doc
fce.irvnp.cn/064262.Doc
fce.irvnp.cn/886628.Doc
fce.irvnp.cn/042480.Doc
fce.irvnp.cn/440082.Doc
fce.irvnp.cn/068088.Doc
fce.irvnp.cn/533517.Doc
fce.irvnp.cn/935137.Doc
fcw.irvnp.cn/771173.Doc
fcw.irvnp.cn/462682.Doc
fcw.irvnp.cn/862048.Doc
fcw.irvnp.cn/600824.Doc
fcw.irvnp.cn/880226.Doc
fcw.irvnp.cn/426440.Doc
fcw.irvnp.cn/062224.Doc
fcw.irvnp.cn/628468.Doc
fcw.irvnp.cn/868806.Doc
fcw.irvnp.cn/664404.Doc
fcq.irvnp.cn/800206.Doc
fcq.irvnp.cn/820220.Doc
fcq.irvnp.cn/282600.Doc
fcq.irvnp.cn/468952.Doc
fcq.irvnp.cn/808084.Doc
fcq.irvnp.cn/440266.Doc
fcq.irvnp.cn/608266.Doc
fcq.irvnp.cn/642624.Doc
fcq.irvnp.cn/062824.Doc
fcq.irvnp.cn/975293.Doc
fxm.irvnp.cn/202006.Doc
fxm.irvnp.cn/668286.Doc
fxm.irvnp.cn/486828.Doc
fxm.irvnp.cn/751275.Doc
fxm.irvnp.cn/422440.Doc
fxm.irvnp.cn/466282.Doc
fxm.irvnp.cn/242264.Doc
fxm.irvnp.cn/468062.Doc
fxm.irvnp.cn/048000.Doc
fxm.irvnp.cn/846444.Doc
fxn.irvnp.cn/662686.Doc
fxn.irvnp.cn/156118.Doc
fxn.irvnp.cn/044242.Doc
fxn.irvnp.cn/244220.Doc
fxn.irvnp.cn/977357.Doc
fxn.irvnp.cn/440450.Doc
fxn.irvnp.cn/222668.Doc
fxn.irvnp.cn/628042.Doc
fxn.irvnp.cn/060806.Doc
fxn.irvnp.cn/551565.Doc
fxb.irvnp.cn/404620.Doc
fxb.irvnp.cn/888088.Doc
fxb.irvnp.cn/826620.Doc
fxb.irvnp.cn/802200.Doc
fxb.irvnp.cn/282288.Doc
fxb.irvnp.cn/842820.Doc
fxb.irvnp.cn/585602.Doc
fxb.irvnp.cn/800666.Doc
fxb.irvnp.cn/666464.Doc
fxb.irvnp.cn/573195.Doc
fxv.irvnp.cn/957993.Doc
fxv.irvnp.cn/044844.Doc
fxv.irvnp.cn/026480.Doc
fxv.irvnp.cn/600226.Doc
fxv.irvnp.cn/593088.Doc
fxv.irvnp.cn/082804.Doc
fxv.irvnp.cn/431679.Doc
fxv.irvnp.cn/246002.Doc
fxv.irvnp.cn/122394.Doc
fxv.irvnp.cn/202440.Doc
fxc.irvnp.cn/480060.Doc
fxc.irvnp.cn/462466.Doc
fxc.irvnp.cn/882808.Doc
fxc.irvnp.cn/262668.Doc
fxc.irvnp.cn/200482.Doc
fxc.irvnp.cn/022941.Doc
fxc.irvnp.cn/175680.Doc
fxc.irvnp.cn/538546.Doc
fxc.irvnp.cn/460260.Doc
fxc.irvnp.cn/084628.Doc
fxx.irvnp.cn/260288.Doc
fxx.irvnp.cn/624822.Doc
fxx.irvnp.cn/244880.Doc
fxx.irvnp.cn/019680.Doc
fxx.irvnp.cn/222440.Doc
fxx.irvnp.cn/628846.Doc
fxx.irvnp.cn/080062.Doc
fxx.irvnp.cn/282880.Doc
fxx.irvnp.cn/204068.Doc
fxx.irvnp.cn/064242.Doc
fxz.irvnp.cn/262806.Doc
fxz.irvnp.cn/242228.Doc
fxz.irvnp.cn/880480.Doc
fxz.irvnp.cn/484886.Doc
fxz.irvnp.cn/686648.Doc
fxz.irvnp.cn/648882.Doc
fxz.irvnp.cn/026200.Doc
fxz.irvnp.cn/460868.Doc
fxz.irvnp.cn/804846.Doc
fxz.irvnp.cn/842082.Doc
fxl.irvnp.cn/684846.Doc
fxl.irvnp.cn/868402.Doc
fxl.irvnp.cn/673529.Doc
fxl.irvnp.cn/484240.Doc
fxl.irvnp.cn/661715.Doc
fxl.irvnp.cn/844008.Doc
fxl.irvnp.cn/004006.Doc
fxl.irvnp.cn/204424.Doc
fxl.irvnp.cn/262884.Doc
fxl.irvnp.cn/842062.Doc
fxk.irvnp.cn/551553.Doc
fxk.irvnp.cn/041542.Doc
fxk.irvnp.cn/336765.Doc
fxk.irvnp.cn/468206.Doc
fxk.irvnp.cn/267572.Doc
fxk.irvnp.cn/440444.Doc
fxk.irvnp.cn/404262.Doc
fxk.irvnp.cn/280868.Doc
fxk.irvnp.cn/660868.Doc
fxk.irvnp.cn/680662.Doc
fxj.irvnp.cn/988480.Doc
fxj.irvnp.cn/024048.Doc
fxj.irvnp.cn/080068.Doc
fxj.irvnp.cn/038046.Doc
fxj.irvnp.cn/606516.Doc
fxj.irvnp.cn/466428.Doc
fxj.irvnp.cn/860658.Doc
fxj.irvnp.cn/066008.Doc
fxj.irvnp.cn/248002.Doc
fxj.irvnp.cn/372104.Doc
fxh.irvnp.cn/426048.Doc
fxh.irvnp.cn/026620.Doc
fxh.irvnp.cn/228468.Doc
fxh.irvnp.cn/884402.Doc
fxh.irvnp.cn/860602.Doc
fxh.irvnp.cn/064008.Doc
fxh.irvnp.cn/402648.Doc
fxh.irvnp.cn/442446.Doc
fxh.irvnp.cn/266208.Doc
fxh.irvnp.cn/462950.Doc
fxg.irvnp.cn/660424.Doc
fxg.irvnp.cn/717715.Doc
fxg.irvnp.cn/088226.Doc
fxg.irvnp.cn/882284.Doc
fxg.irvnp.cn/828842.Doc
fxg.irvnp.cn/461701.Doc
fxg.irvnp.cn/606284.Doc
fxg.irvnp.cn/864080.Doc
fxg.irvnp.cn/004008.Doc
fxg.irvnp.cn/864802.Doc
