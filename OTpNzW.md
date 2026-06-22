似瞪慈富战


Python weakref 弱引用
============================

weakref 允许引用对象但不增加引用计数, 在 GC 时对象可被回收, 防止内存泄漏。

1. ref() — 基础弱引用
------------------------

import weakref
import gc

class ExpensiveObject:
    """模拟一个占用资源的大对象"""
    def __init__(self, name):
        self.name = name
        print(f"  创建: {self.name}")
    def __del__(self):
        print(f"  销毁: {self.name}")

# 创建弱引用
obj = ExpensiveObject("数据缓存")
ref = weakref.ref(obj)                         # 创建弱引用, 不增加引用计数
print("弱引用对象:", ref)                       # <weakref at ...; to 'ExpensiveObject'>
print("通过弱引用访问:", ref().name)             # 数据缓存 (调用 ref() 获取原对象)

# 删除强引用后, 弱引用自动失效
del obj                                         # 触发 __del__
print("对象被回收后:", ref())                    # None (弱引用已死)

# 检查弱引用是否存活: ref() 返回 None 或对象
def check_alive(ref):
    """检查弱引用指向的对象是否仍然存活"""
    obj = ref()
    if obj is not None:
        print(f"对象存活: {obj}")
    else:
        print("对象已被回收")

obj2 = ExpensiveObject("临时对象")
ref2 = weakref.ref(obj2)
check_alive(ref2)                               # 存活
del obj2
gc.collect()                                    # 强制垃圾回收
check_alive(ref2)                               # 已回收

# ref 的可调用特性: ref() 返回对象或 None
# 可作为缓存失效检测机制

2. WeakValueDictionary — 值弱引用字典
----------------------------------------

import weakref
import gc

# WeakValueDictionary: 键强引用, 值弱引用
# 值对象被其他强引用释放后, 键自动移除
# 最适合: 对象缓存 (缓存对象但允许被回收)

class Image:
    """模拟图片对象, 占用大量内存"""
    def __init__(self, filename):
        self.filename = filename
        print(f"  加载图片: {filename}")
    def __del__(self):
        print(f"  释放图片: {self.filename}")
    def display(self):
        print(f"显示: {self.filename}")

# 图片缓存: 避免重复加载, 但不阻止图片被回收
image_cache = weakref.WeakValueDictionary()

def get_image(filename):
    """从缓存获取图片, 不存在则加载"""
    img = image_cache.get(filename)
    if img is None:
        img = Image(filename)
        image_cache[filename] = img
    return img

# 第一次加载
img1 = get_image("photo.jpg")                    # 加载图片
print("缓存中的图片:", image_cache)               # 有内容

# 第二次获取, 从缓存返回
img2 = get_image("photo.jpg")
print("img1 和 img2 是同一对象:", img1 is img2)   # True

# 删除所有强引用
del img1
del img2
gc.collect()                                     # 触发 GC
print("回收后缓存:", image_cache)                 # 缓存自动清除

# 应用: 对象池, 监听器, 缓存系统
# 注意: WeakValueDictionary 的值必须是可弱引用的 (list/int 不行, 需包装)

3. WeakKeyDictionary — 键弱引用字典
---------------------------------------

import weakref

# WeakKeyDictionary: 键弱引用, 值强引用
# 键对象被回收后, 条目自动删除
# 最适合: 为对象附加元数据 (不修改原对象)

class Widget:
    """UI 控件基类"""
    def __init__(self, name):
        self.name = name
    def __repr__(self):
        return f"Widget({self.name})"

# 为控件存储额外元数据, 但不修改 Widget 类
metadata = weakref.WeakKeyDictionary()

btn1 = Widget("button_ok")
btn2 = Widget("button_cancel")

metadata[btn1] = {"x": 10, "y": 20, "visible": True}
metadata[btn2] = {"x": 30, "y": 40, "visible": False}

print("元数据缓存:", len(metadata))               # 2

# 删除控件, 元数据自动清理
del btn1
print("btn1 删除后元数据条数:", len(metadata))    # 1

# 应用场景: 事件监听器, 属性注入, 缓存计算结果
# 避免在业务对象上添加不必要的属性

# 对比普通字典: 如果忘记删除元数据, 会导致内存泄漏
# 实际项目中常用于框架级别的装饰器

class EventEmitter:
    """事件发射器: 使用 WeakKeyDictionary 存储监听器"""
    def __init__(self):
        self._listeners = weakref.WeakKeyDictionary()

    def on(self, obj, callback):
        """为对象注册监听器"""
        if obj not in self._listeners:
            self._listeners[obj] = []
        self._listeners[obj].append(callback)

    def emit(self, obj, *args, **kwargs):
        """触发对象的监听器"""
        if obj in self._listeners:
            for cb in self._listeners[obj]:
                cb(obj, *args, **kwargs)

emitter = EventEmitter()
player = Widget("player")
emitter.on(player, lambda o, e: print(f"{o} 触发事件: {e}"))
emitter.emit(player, "跳跃")
# 删除 player 后监听器自动清理, 无需手动 deregister

4. WeakSet — 弱引用集合
--------------------------

import weakref

# WeakSet: 集合元素为弱引用, 对象回收后自动移除
# 场景: 跟踪存活的对象实例

class Connection:
    """网络连接, 用 WeakSet 跟踪所有活跃连接"""
    _active_connections = weakref.WeakSet()

    def __init__(self, conn_id):
        self.conn_id = conn_id
        self._active_connections.add(self)
        print(f"连接建立: {conn_id}")

    def __del__(self):
        print(f"连接关闭: {self.conn_id}")

    @classmethod
    def active_count(cls):
        """当前活跃连接数"""
        return len(cls._active_connections)

    @classmethod
    def list_active(cls):
        """列出所有活跃连接 ID"""
        return [c.conn_id for c in cls._active_connections]

# 模拟连接链路
c1 = Connection("conn-001")
c2 = Connection("conn-002")
c3 = Connection("conn-003")
print("活跃连接数:", Connection.active_count())   # 3

del c1
gc.collect()
print("关闭一个后活跃连接:", Connection.active_count())  # 2
print("活跃连接列表:", Connection.list_active())

# 其他 WeakSet 妙用: 去重事件监听, 跟踪工厂产生的对象
class Factory:
    _instances = weakref.WeakSet()

    def create(self, name):
        obj = ExpensiveObject(name)
        self._instances.add(obj)
        return obj

    def count(self):
        return len(self._instances)

factory = Factory()
a = factory.create("A")
b = factory.create("B")
print("工厂实例计数:", factory.count())           # 2
del a
print("回收后计数:", factory.count())             # 1

5. finalize — 清理回调
--------------------------

import weakref

# finalize 在对象被垃圾回收时自动执行清理回调
# 比 __del__ 更可靠, 不参与循环引用检测

class TempFile:
    """模拟临时文件, 在回收时自动删除"""
    def __init__(self, name):
        self.name = name
        # 注册清理回调: 当此对象被回收时自动删除文件
        weakref.finalize(self, self._cleanup, name)
        print(f"创建临时文件: {name}")

    @staticmethod
    def _cleanup(filename):
        """实际的清理操作 (静态方法, 不持有 self 引用)"""
        print(f"  [finalize] 清理临时文件: {filename}")
        # 实际代码: os.remove(filename)

def test_finalize():
    tf = TempFile("tmp_data.bin")
    # 函数返回时 tf 被回收, finalize 自动触发

test_finalize()                                  # 输出清理信息
print("函数已返回, 临时文件已自动清理")

# finalize 的优势:
# 1. 不参与循环引用, 不会阻止 GC
# 2. 可指定参数和关键字参数
# 3. 支持多个 finalize 注册在同一对象上
# 4. 可调用 detach() 取消清理

class Resource:
    def __init__(self, rid):
        self.rid = rid
        self._finalizer = weakref.finalize(self, self._release, rid)

    @staticmethod
    def _release(rid):
        print(f"释放资源: {rid}")

    def close(self):
        """手动关闭, 取消自动清理"""
        self._release(self.rid)
        self._finalizer.detach()                 # 取消 finalize

res = Resource("R-001")
res.close()                                      # 手动关闭, finalize 不再触发
print("手动关闭后, 回收不再触发清理")
del res
gc.collect()                                     # 无额外输出

6. proxy — 代理对象
-----------------------

import weakref

# proxy 创建弱引用代理, 使用代理就像使用原对象一样
# 区别: ref() 需要调用 ref() 来获取对象, proxy 直接像对象一样操作

class Data:
    def __init__(self, value):
        self.value = value

    def display(self):
        print(f"数据: {self.value}")

    def __add__(self, other):
        return Data(self.value + other.value)

data = Data(42)
proxy = weakref.proxy(data)                     # 创建代理

# 代理可直接访问属性和方法, 无需调用
print("代理访问属性:", proxy.value)              # 42
proxy.display()                                  # 数据: 42

# 代理支持运算符
data2 = Data(10)
proxy2 = weakref.proxy(data2)
result = proxy + proxy2                         # 通过代理调用 __add__
print("代理加法结果:", result.value)

# 原对象回收后, 访问代理会抛出 ReferenceError
del data
gc.collect()
try:
    print(proxy.value)                          # 抛出 ReferenceError
except weakref.ReferenceError as e:
    print("代理引用失效:", e)

# proxy vs ref 选择:
# proxy: 无需调用 () 语法更自然, 但每个操作都有引用检测开销
# ref:  需要显式调用 ref(), 但效率略高, 可用于条件检查

# 何时用 proxy:
class Tree:
    """树结构: 父节点用弱引用代理避免循环引用"""
    def __init__(self, name, parent=None):
        self.name = name
        self.children = []
        self._parent_ref = weakref.proxy(parent) if parent else None

    @property
    def parent(self):
        return self._parent_ref

    def add_child(self, child):
        child._parent_ref = weakref.proxy(self)  # 父节点用弱引用
        self.children.append(child)

root = Tree("root")
child = Tree("child", root)
print("子节点访问父节点:", child.parent.name)     # root (通过代理)

7. 引用循环与垃圾回收
--------------------------

import weakref
import gc

# 场景: 没有弱引用时的循环引用问题
class Node:
    def __init__(self, name):
        self.name = name
        self.parent = None
        self.children = []

    def __del__(self):
        print(f"  Node {self.name} 被回收")

# 创建循环引用图
a = Node("A")
b = Node("B")
a.children.append(b)
b.parent = a                                    # 循环引用: a -> b -> a

# 删除外部引用
del a
del b

# Python GC 可以处理循环引用, 但不是立即回收
gc.collect()                                     # 强制回收会触发两个 __del__
print("循环引用已回收")

# 使用弱引用避免循环引用导致的内存泄漏
class WeakNode:
    def __init__(self, name, parent=None):
        self.name = name
        self.children = []
        # 父引用使用弱引用, 避免循环引用
        self._parent = weakref.ref(parent) if parent else None

    @property
    def parent(self):
        return self._parent() if self._parent else None

    def add_child(self, child):
        self.children.append(child)

# 验证: 弱引用不会产生循环引用
gc.set_debug(gc.DEBUG_SAVEALL)                   # 收集所有不可达对象

wa = WeakNode("WA")
wb = WeakNode("WB", wa)
wa.add_child(wb)

# 清理
del wa
del wb
unreachable = gc.collect()
print("弱引用版本不可达对象数:", unreachable)      # 通常为 0

# 内存管理最佳实践:
# 1. 缓存用 WeakValueDictionary
# 2. 元数据用 WeakKeyDictionary
# 3. 父→子用强引用, 子→父用弱引用
# 4. 回调清理用 finalize
# 5. 对象追踪用 WeakSet

总结: weakref 是防止内存泄漏的关键工具, 尤其适合缓存、事件监听、对象池和树形结构.

扯懈仙谛把方志渡弥猎目坟猩抢蒂

say.jouwir.cn/020248.Doc
say.jouwir.cn/846000.Doc
say.jouwir.cn/604808.Doc
say.jouwir.cn/620644.Doc
say.jouwir.cn/266286.Doc
say.jouwir.cn/224848.Doc
say.jouwir.cn/335713.Doc
say.jouwir.cn/133119.Doc
sat.jouwir.cn/286406.Doc
sat.jouwir.cn/082822.Doc
sat.jouwir.cn/808200.Doc
sat.jouwir.cn/533391.Doc
sat.jouwir.cn/604068.Doc
sat.jouwir.cn/600620.Doc
sat.jouwir.cn/284400.Doc
sat.jouwir.cn/088620.Doc
sat.jouwir.cn/086080.Doc
sat.jouwir.cn/664400.Doc
sar.jouwir.cn/224680.Doc
sar.jouwir.cn/682442.Doc
sar.jouwir.cn/882626.Doc
sar.jouwir.cn/888048.Doc
sar.jouwir.cn/020200.Doc
sar.jouwir.cn/268204.Doc
sar.jouwir.cn/804860.Doc
sar.jouwir.cn/284020.Doc
sar.jouwir.cn/042484.Doc
sar.jouwir.cn/274610.Doc
sae.jouwir.cn/916878.Doc
sae.jouwir.cn/864602.Doc
sae.jouwir.cn/662428.Doc
sae.jouwir.cn/268220.Doc
sae.jouwir.cn/040242.Doc
sae.jouwir.cn/666282.Doc
sae.jouwir.cn/202648.Doc
sae.jouwir.cn/444040.Doc
sae.jouwir.cn/428026.Doc
sae.jouwir.cn/680820.Doc
saw.jouwir.cn/846048.Doc
saw.jouwir.cn/428048.Doc
saw.jouwir.cn/628808.Doc
saw.jouwir.cn/880062.Doc
saw.jouwir.cn/088486.Doc
saw.jouwir.cn/379977.Doc
saw.jouwir.cn/646066.Doc
saw.jouwir.cn/884040.Doc
saw.jouwir.cn/244626.Doc
saw.jouwir.cn/408202.Doc
saq.jouwir.cn/600628.Doc
saq.jouwir.cn/684600.Doc
saq.jouwir.cn/006226.Doc
saq.jouwir.cn/682408.Doc
saq.jouwir.cn/260488.Doc
saq.jouwir.cn/824802.Doc
saq.jouwir.cn/240022.Doc
saq.jouwir.cn/284846.Doc
saq.jouwir.cn/266062.Doc
saq.jouwir.cn/482664.Doc
spm.jouwir.cn/628840.Doc
spm.jouwir.cn/286400.Doc
spm.jouwir.cn/626800.Doc
spm.jouwir.cn/484400.Doc
spm.jouwir.cn/866668.Doc
spm.jouwir.cn/260028.Doc
spm.jouwir.cn/060620.Doc
spm.jouwir.cn/042626.Doc
spm.jouwir.cn/840066.Doc
spm.jouwir.cn/002648.Doc
spn.jouwir.cn/044666.Doc
spn.jouwir.cn/888688.Doc
spn.jouwir.cn/220260.Doc
spn.jouwir.cn/446624.Doc
spn.jouwir.cn/466648.Doc
spn.jouwir.cn/040088.Doc
spn.jouwir.cn/462442.Doc
spn.jouwir.cn/428624.Doc
spn.jouwir.cn/802264.Doc
spn.jouwir.cn/842688.Doc
spb.jouwir.cn/280424.Doc
spb.jouwir.cn/228626.Doc
spb.jouwir.cn/240202.Doc
spb.jouwir.cn/640664.Doc
spb.jouwir.cn/088822.Doc
spb.jouwir.cn/260884.Doc
spb.jouwir.cn/246080.Doc
spb.jouwir.cn/004042.Doc
spb.jouwir.cn/228062.Doc
spb.jouwir.cn/006482.Doc
spv.jouwir.cn/022824.Doc
spv.jouwir.cn/622004.Doc
spv.jouwir.cn/622268.Doc
spv.jouwir.cn/868244.Doc
spv.jouwir.cn/404480.Doc
spv.jouwir.cn/880680.Doc
spv.jouwir.cn/480226.Doc
spv.jouwir.cn/619498.Doc
spv.jouwir.cn/888886.Doc
spv.jouwir.cn/026240.Doc
spc.jouwir.cn/840880.Doc
spc.jouwir.cn/206600.Doc
spc.jouwir.cn/086868.Doc
spc.jouwir.cn/444408.Doc
spc.jouwir.cn/062804.Doc
spc.jouwir.cn/026844.Doc
spc.jouwir.cn/808640.Doc
spc.jouwir.cn/222428.Doc
spc.jouwir.cn/408022.Doc
spc.jouwir.cn/068462.Doc
spx.jouwir.cn/666684.Doc
spx.jouwir.cn/262682.Doc
spx.jouwir.cn/248208.Doc
spx.jouwir.cn/608646.Doc
spx.jouwir.cn/800686.Doc
spx.jouwir.cn/266240.Doc
spx.jouwir.cn/462286.Doc
spx.jouwir.cn/246400.Doc
spx.jouwir.cn/642082.Doc
spx.jouwir.cn/400808.Doc
spz.jouwir.cn/066868.Doc
spz.jouwir.cn/624248.Doc
spz.jouwir.cn/400460.Doc
spz.jouwir.cn/668242.Doc
spz.jouwir.cn/046880.Doc
spz.jouwir.cn/084446.Doc
spz.jouwir.cn/022044.Doc
spz.jouwir.cn/600082.Doc
spz.jouwir.cn/820686.Doc
spz.jouwir.cn/846462.Doc
spl.jouwir.cn/028000.Doc
spl.jouwir.cn/806208.Doc
spl.jouwir.cn/268004.Doc
spl.jouwir.cn/220080.Doc
spl.jouwir.cn/660600.Doc
spl.jouwir.cn/862224.Doc
spl.jouwir.cn/060006.Doc
spl.jouwir.cn/042828.Doc
spl.jouwir.cn/862466.Doc
spl.jouwir.cn/246064.Doc
spk.jouwir.cn/480880.Doc
spk.jouwir.cn/420464.Doc
spk.jouwir.cn/886404.Doc
spk.jouwir.cn/884286.Doc
spk.jouwir.cn/428426.Doc
spk.jouwir.cn/862882.Doc
spk.jouwir.cn/046262.Doc
spk.jouwir.cn/202800.Doc
spk.jouwir.cn/848246.Doc
spk.jouwir.cn/406404.Doc
spj.jouwir.cn/288000.Doc
spj.jouwir.cn/668422.Doc
spj.jouwir.cn/688446.Doc
spj.jouwir.cn/571999.Doc
spj.jouwir.cn/088826.Doc
spj.jouwir.cn/660022.Doc
spj.jouwir.cn/400644.Doc
spj.jouwir.cn/882686.Doc
spj.jouwir.cn/824446.Doc
spj.jouwir.cn/848624.Doc
sph.jouwir.cn/660242.Doc
sph.jouwir.cn/866482.Doc
sph.jouwir.cn/866026.Doc
sph.jouwir.cn/846626.Doc
sph.jouwir.cn/244068.Doc
sph.jouwir.cn/060024.Doc
sph.jouwir.cn/608248.Doc
sph.jouwir.cn/246686.Doc
sph.jouwir.cn/848626.Doc
sph.jouwir.cn/642282.Doc
spg.jouwir.cn/933557.Doc
spg.jouwir.cn/280864.Doc
spg.jouwir.cn/062204.Doc
spg.jouwir.cn/424662.Doc
spg.jouwir.cn/422660.Doc
spg.jouwir.cn/400064.Doc
spg.jouwir.cn/826644.Doc
spg.jouwir.cn/004040.Doc
spg.jouwir.cn/648088.Doc
spg.jouwir.cn/404846.Doc
spf.jouwir.cn/242442.Doc
spf.jouwir.cn/800026.Doc
spf.jouwir.cn/444939.Doc
spf.jouwir.cn/000862.Doc
spf.jouwir.cn/888844.Doc
spf.jouwir.cn/866400.Doc
spf.jouwir.cn/206888.Doc
spf.jouwir.cn/286428.Doc
spf.jouwir.cn/888604.Doc
spf.jouwir.cn/624022.Doc
spd.jouwir.cn/286844.Doc
spd.jouwir.cn/088026.Doc
spd.jouwir.cn/042404.Doc
spd.jouwir.cn/646288.Doc
spd.jouwir.cn/624668.Doc
spd.jouwir.cn/826222.Doc
spd.jouwir.cn/408046.Doc
spd.jouwir.cn/828860.Doc
spd.jouwir.cn/084202.Doc
spd.jouwir.cn/886264.Doc
sps.jouwir.cn/626486.Doc
sps.jouwir.cn/260644.Doc
sps.jouwir.cn/824824.Doc
sps.jouwir.cn/048844.Doc
sps.jouwir.cn/682460.Doc
sps.jouwir.cn/484242.Doc
sps.jouwir.cn/224044.Doc
sps.jouwir.cn/820624.Doc
sps.jouwir.cn/642622.Doc
sps.jouwir.cn/446468.Doc
spa.jouwir.cn/626422.Doc
spa.jouwir.cn/844042.Doc
spa.jouwir.cn/624888.Doc
spa.jouwir.cn/064464.Doc
spa.jouwir.cn/206286.Doc
spa.jouwir.cn/886840.Doc
spa.jouwir.cn/822020.Doc
spa.jouwir.cn/480620.Doc
spa.jouwir.cn/048008.Doc
spa.jouwir.cn/008460.Doc
spp.jouwir.cn/224022.Doc
spp.jouwir.cn/024002.Doc
spp.jouwir.cn/442482.Doc
spp.jouwir.cn/068428.Doc
spp.jouwir.cn/004480.Doc
spp.jouwir.cn/806260.Doc
spp.jouwir.cn/602688.Doc
spp.jouwir.cn/862444.Doc
spp.jouwir.cn/860666.Doc
spp.jouwir.cn/604646.Doc
spo.jouwir.cn/884606.Doc
spo.jouwir.cn/442408.Doc
spo.jouwir.cn/486400.Doc
spo.jouwir.cn/448882.Doc
spo.jouwir.cn/204680.Doc
spo.jouwir.cn/662222.Doc
spo.jouwir.cn/264828.Doc
spo.jouwir.cn/288082.Doc
spo.jouwir.cn/482480.Doc
spo.jouwir.cn/604066.Doc
spi.jouwir.cn/226442.Doc
spi.jouwir.cn/480202.Doc
spi.jouwir.cn/466282.Doc
spi.jouwir.cn/248826.Doc
spi.jouwir.cn/464424.Doc
spi.jouwir.cn/286664.Doc
spi.jouwir.cn/886846.Doc
spi.jouwir.cn/228000.Doc
spi.jouwir.cn/864226.Doc
spi.jouwir.cn/400484.Doc
spu.jouwir.cn/200266.Doc
spu.jouwir.cn/026206.Doc
spu.jouwir.cn/084248.Doc
spu.jouwir.cn/480020.Doc
spu.jouwir.cn/000006.Doc
spu.jouwir.cn/802842.Doc
spu.jouwir.cn/886848.Doc
spu.jouwir.cn/915733.Doc
spu.jouwir.cn/842604.Doc
spu.jouwir.cn/828268.Doc
spy.jouwir.cn/444204.Doc
spy.jouwir.cn/420086.Doc
spy.jouwir.cn/171573.Doc
spy.jouwir.cn/911379.Doc
spy.jouwir.cn/426062.Doc
spy.jouwir.cn/862468.Doc
spy.jouwir.cn/062686.Doc
spy.jouwir.cn/264844.Doc
spy.jouwir.cn/224800.Doc
spy.jouwir.cn/064848.Doc
spt.jouwir.cn/802028.Doc
spt.jouwir.cn/604842.Doc
spt.jouwir.cn/466282.Doc
spt.jouwir.cn/886060.Doc
spt.jouwir.cn/802260.Doc
spt.jouwir.cn/682200.Doc
spt.jouwir.cn/048868.Doc
spt.jouwir.cn/828444.Doc
spt.jouwir.cn/606608.Doc
spt.jouwir.cn/939171.Doc
spr.jouwir.cn/488486.Doc
spr.jouwir.cn/862840.Doc
spr.jouwir.cn/602240.Doc
spr.jouwir.cn/088240.Doc
spr.jouwir.cn/662482.Doc
spr.jouwir.cn/828226.Doc
spr.jouwir.cn/408444.Doc
spr.jouwir.cn/040688.Doc
spr.jouwir.cn/660406.Doc
spr.jouwir.cn/402844.Doc
spe.jouwir.cn/468224.Doc
spe.jouwir.cn/464442.Doc
spe.jouwir.cn/608608.Doc
spe.jouwir.cn/700977.Doc
spe.jouwir.cn/282884.Doc
spe.jouwir.cn/353010.Doc
spe.jouwir.cn/484666.Doc
spe.jouwir.cn/620260.Doc
spe.jouwir.cn/604440.Doc
spe.jouwir.cn/660428.Doc
spw.jouwir.cn/466688.Doc
spw.jouwir.cn/608442.Doc
spw.jouwir.cn/608242.Doc
spw.jouwir.cn/086664.Doc
spw.jouwir.cn/422864.Doc
spw.jouwir.cn/082442.Doc
spw.jouwir.cn/468602.Doc
spw.jouwir.cn/020068.Doc
spw.jouwir.cn/000880.Doc
spw.jouwir.cn/026044.Doc
spq.jouwir.cn/662268.Doc
spq.jouwir.cn/042886.Doc
spq.jouwir.cn/206020.Doc
spq.jouwir.cn/462264.Doc
spq.jouwir.cn/082062.Doc
spq.jouwir.cn/404848.Doc
spq.jouwir.cn/040468.Doc
spq.jouwir.cn/422662.Doc
spq.jouwir.cn/513131.Doc
spq.jouwir.cn/284020.Doc
som.cggkm.cn/242628.Doc
som.cggkm.cn/268486.Doc
som.cggkm.cn/868640.Doc
som.cggkm.cn/224820.Doc
som.cggkm.cn/288422.Doc
som.cggkm.cn/066024.Doc
som.cggkm.cn/800648.Doc
som.cggkm.cn/842624.Doc
som.cggkm.cn/068028.Doc
som.cggkm.cn/060264.Doc
son.cggkm.cn/064284.Doc
son.cggkm.cn/822400.Doc
son.cggkm.cn/155175.Doc
son.cggkm.cn/440840.Doc
son.cggkm.cn/820408.Doc
son.cggkm.cn/268600.Doc
son.cggkm.cn/751551.Doc
son.cggkm.cn/822844.Doc
son.cggkm.cn/620884.Doc
son.cggkm.cn/482404.Doc
sob.cggkm.cn/006428.Doc
sob.cggkm.cn/735735.Doc
sob.cggkm.cn/020228.Doc
sob.cggkm.cn/084028.Doc
sob.cggkm.cn/864282.Doc
sob.cggkm.cn/426648.Doc
sob.cggkm.cn/468800.Doc
sob.cggkm.cn/644062.Doc
sob.cggkm.cn/880248.Doc
sob.cggkm.cn/200806.Doc
sov.cggkm.cn/868028.Doc
sov.cggkm.cn/208266.Doc
sov.cggkm.cn/804804.Doc
sov.cggkm.cn/886600.Doc
sov.cggkm.cn/048008.Doc
sov.cggkm.cn/282822.Doc
sov.cggkm.cn/800448.Doc
sov.cggkm.cn/020464.Doc
sov.cggkm.cn/020886.Doc
sov.cggkm.cn/975377.Doc
soc.cggkm.cn/840402.Doc
soc.cggkm.cn/828684.Doc
soc.cggkm.cn/915355.Doc
soc.cggkm.cn/262260.Doc
soc.cggkm.cn/806680.Doc
soc.cggkm.cn/640222.Doc
soc.cggkm.cn/064462.Doc
soc.cggkm.cn/040022.Doc
soc.cggkm.cn/002242.Doc
soc.cggkm.cn/466446.Doc
sox.cggkm.cn/846462.Doc
sox.cggkm.cn/862666.Doc
sox.cggkm.cn/884842.Doc
sox.cggkm.cn/826008.Doc
sox.cggkm.cn/282604.Doc
sox.cggkm.cn/822680.Doc
sox.cggkm.cn/224088.Doc
sox.cggkm.cn/686020.Doc
sox.cggkm.cn/644420.Doc
sox.cggkm.cn/000662.Doc
soz.cggkm.cn/664622.Doc
soz.cggkm.cn/242686.Doc
soz.cggkm.cn/060086.Doc
soz.cggkm.cn/000682.Doc
soz.cggkm.cn/846684.Doc
soz.cggkm.cn/246268.Doc
soz.cggkm.cn/426280.Doc
soz.cggkm.cn/022820.Doc
soz.cggkm.cn/402840.Doc
soz.cggkm.cn/422462.Doc
sol.cggkm.cn/228884.Doc
sol.cggkm.cn/868486.Doc
sol.cggkm.cn/202608.Doc
sol.cggkm.cn/824484.Doc
sol.cggkm.cn/600626.Doc
sol.cggkm.cn/222222.Doc
sol.cggkm.cn/628842.Doc
