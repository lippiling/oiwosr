Python 描述符协议深入
===========================

描述符是 Python 最核心的底层机制之一，@property、@classmethod、
@staticmethod 乃至函数绑定都是基于描述符实现的。

1. 描述符协议基础
---------------------
实现 __get__、__set__、__delete__ 中任一方法的类即为描述符。

class VerboseDescriptor:
    """打印日志的描述符——演示协议方法的调用时机"""
    def __init__(self, name: str):
        self.name = name

    def __get__(self, obj, objtype=None):
        """访问属性时调用"""
        if obj is None:
            return self  # 通过类访问时返回描述符本身
        print(f"__get__: 访问 {self.name}")
        return getattr(obj, f"_{self.name}", None)

    def __set__(self, obj, value):
        """赋值时调用"""
        print(f"__set__: 设置 {self.name} = {value}")
        setattr(obj, f"_{self.name}", value)

    def __delete__(self, obj):
        """删除属性时调用"""
        print(f"__delete__: 删除 {self.name}")
        delattr(obj, f"_{self.name}")


class MyClass:
    """使用描述符的类"""
    attr = VerboseDescriptor("attr")  # 类属性——描述符实例

    def __init__(self, value):
        self.attr = value  # 触发 __set__


obj = MyClass(42)         # __set__: 设置 attr = 42
print(obj.attr)           # __get__: 访问 attr → 42
del obj.attr              # __delete__: 删除 attr
print(MyClass.attr)       # 返回描述符本身（obj 为 None）


2. 数据描述符 vs 非数据描述符
---------------------------------
数据描述符（有 __set__）优先级高于实例属性（__dict__）。
非数据描述符（只有 __get__）优先级低于实例属性。

class DataDescriptor:
    """数据描述符——有 __set__，优先级高于实例属性"""
    def __get__(self, obj, objtype=None):
        return "数据描述符的值"

    def __set__(self, obj, value):
        print(f"数据描述符 __set__: {value}")


class NonDataDescriptor:
    """非数据描述符——只有 __get__，优先级低于实例属性"""
    def __get__(self, obj, objtype=None):
        return "非数据描述符的值"


class Demo:
    data_attr = DataDescriptor()       # 数据描述符
    non_data_attr = NonDataDescriptor()  # 非数据描述符

    def __init__(self):
        self.__dict__["non_data_attr"] = "实例属性覆盖"  # 直接操作 __dict__


d = Demo()
print(d.data_attr)        # "数据描述符的值"——数据描述符优先
d.data_attr = "新值"      # 调用 __set__
print(d.data_attr)        # "数据描述符的值"——__get__ 返回固定值

print(d.non_data_attr)    # "实例属性覆盖"——实例属性优先级高于非数据描述符
del d.__dict__["non_data_attr"]
print(d.non_data_attr)    # "非数据描述符的值"——删除实例属性后回退到描述符


3. 方法绑定——__get__ 的典型应用
----------------------------------------
函数也是描述符！函数的 __get__ 方法实现了 self 绑定。

class MethodDemo:
    def method(self, x: int) -> str:
        """普通实例方法——底层也是描述符"""
        return f"方法被调用: self={self}, x={x}"


md = MethodDemo()

# 通过实例访问——__get__ 绑定 self
bound_method = md.method
print(bound_method)       # <bound method MethodDemo.method of ...>
print(bound_method(5))    # "方法被调用: self=<...>, x=5"

# 通过类访问——__get__ 返回原始函数（不绑定）
unbound = MethodDemo.method
print(unbound)            # <function MethodDemo.method at ...>
print(unbound(md, 5))     # 手动传 self

# 等价手动调用 __get__
bound_manual = MethodDemo.method.__get__(md, MethodDemo)
print(bound_manual(5))    # 结果相同


4. @property 本质是数据描述符
-------------------------------------
@property 通过描述符协议实现 getter/setter/deleter。

class PropertyDemo:
    """手动重写 @property 的逻辑来理解描述符"""
    def __init__(self):
        self._x = 0

    def get_x(self):
        return self._x

    def set_x(self, value):
        if value < 0:
            raise ValueError("不能为负")
        self._x = value

    def del_x(self):
        del self._x

    # 手动创建 property 实例
    x = property(get_x, set_x, del_x, "x 属性的文档")


# 等价于使用 @property 装饰器
class PropertyDemo2:
    @property
    def x(self):
        """x 属性的文档"""
        return self._x

    @x.setter
    def x(self, value):
        if value < 0:
            raise ValueError("不能为负")
        self._x = value

    @x.deleter
    def x(self):
        del self._x


pd = PropertyDemo2()
pd.x = 42
print(pd.x)  # 42


5. @classmethod 和 @staticmethod 也是描述符
-------------------------------------------------
classmethod 绑定到类，staticmethod 返回原始函数。

class MethodTypes:
    """演示不同方法的描述符行为"""
    def regular(self):
        """实例方法——绑定到实例"""
        return f"实例方法: {type(self).__name__}"

    @classmethod
    def class_meth(cls):
        """类方法——绑定到类"""
        return f"类方法: {cls.__name__}"

    @staticmethod
    def static_meth():
        """静态方法——没有绑定"""
        return "静态方法"


mt = MethodTypes()
print(mt.regular())      # 实例方法: MethodTypes
print(mt.class_meth())   # 类方法: MethodTypes
print(mt.static_meth())  # 静态方法

# 实现原理
# MethodTypes.__dict__["regular"].__get__(mt, MethodTypes)  → bound method
# MethodTypes.__dict__["class_meth"].__get__(None, MethodTypes) → bound classmethod
# MethodTypes.__dict__["static_meth"].__get__(mt, MethodTypes) → original function


6. 验证描述符——类型检查属性
----------------------------------
自定义描述符实现属性校验。

class ValidatedAttribute:
    """验证描述符——对属性赋值进行类型和范围校验"""
    def __init__(self, name: str, expected_type: type, min_val=None, max_val=None):
        self.name = f"_{name}"  # 存储用的私有属性名
        self.expected_type = expected_type
        self.min_val = min_val
        self.max_val = max_val

    def __set_name__(self, owner, name):
        """Python 3.6+，自动获取属性名"""
        self.name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.name)

    def __set__(self, obj, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(
                f"期望类型 {self.expected_type.__name__}，"
                f"收到 {type(value).__name__}"
            )
        if self.min_val is not None and value < self.min_val:
            raise ValueError(f"值 {value} 小于最小值 {self.min_val}")
        if self.max_val is not None and value > self.max_val:
            raise ValueError(f"值 {value} 大于最大值 {self.max_val}")
        setattr(obj, self.name, value)


class Product:
    """商品——使用验证描述符定义字段"""
    name = ValidatedAttribute("name", str)        # 必须为字符串
    price = ValidatedAttribute("price", (int, float), min_val=0)  # 非负数
    quantity = ValidatedAttribute("quantity", int, min_val=0, max_val=10000)

    def __init__(self, name: str, price: float, quantity: int):
        self.name = name
        self.price = price
        self.quantity = quantity

    @property
    def total_value(self) -> float:
        return self.price * self.quantity


# 使用
p = Product("笔记本", 29.9, 100)
print(p.total_value)  # 2990.0

# p.name = 123       # TypeError!
# p.price = -10      # ValueError!
# p.quantity = 99999 # ValueError!


7. __set_name__——获取属性名
-------------------------------
Python 3.6+ 的描述符扩展，自动知道在类中的属性名。

class AutoNamedDescriptor:
    """使用 __set_name__ 自动获取属性名"""
    def __set_name__(self, owner, name):
        """在类创建时自动调用，owner 是所属类，name 是属性名"""
        self.public_name = name
        self.private_name = f"_{name}"
        print(f"描述符绑定: {owner.__name__}.{name}")

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name)

    def __set__(self, obj, value):
        print(f"设置 {self.public_name} = {value}")
        setattr(obj, self.private_name, value)


class Model:
    """使用自动命名描述符"""
    field1 = AutoNamedDescriptor()
    field2 = AutoNamedDescriptor()


m = Model()
m.field1 = "hello"  # 设置 field1 = hello
print(m.field1)     # hello


总结：描述符是 Python 属性访问的核心机制，理解描述符协议能深入理解
Python 对象模型。数据描述符（定义 __set__）优先级高于实例属性，
非数据描述符（仅 __get__）优先级低于实例属性。@property、
@classmethod、@staticmethod 底层都是描述符，自定义描述符可实现
强大的属性校验和转换逻辑。

dhc.aa5uzL7.cn/51171.Doc
dhc.aa5uzL7.cn/55775.Doc
dhc.aa5uzL7.cn/86620.Doc
dhc.aa5uzL7.cn/48824.Doc
dhc.aa5uzL7.cn/77715.Doc
dhc.aa5uzL7.cn/59119.Doc
dhc.aa5uzL7.cn/31193.Doc
dhc.aa5uzL7.cn/59333.Doc
dhc.aa5uzL7.cn/11533.Doc
dhc.aa5uzL7.cn/97795.Doc
dhx.aa5uzL7.cn/51551.Doc
dhx.aa5uzL7.cn/91135.Doc
dhx.aa5uzL7.cn/55113.Doc
dhx.aa5uzL7.cn/33179.Doc
dhx.aa5uzL7.cn/57979.Doc
dhx.aa5uzL7.cn/95775.Doc
dhx.aa5uzL7.cn/33359.Doc
dhx.aa5uzL7.cn/73731.Doc
dhx.aa5uzL7.cn/13913.Doc
dhx.aa5uzL7.cn/57735.Doc
dhz.aa5uzL7.cn/59953.Doc
dhz.aa5uzL7.cn/35517.Doc
dhz.aa5uzL7.cn/75335.Doc
dhz.aa5uzL7.cn/55931.Doc
dhz.aa5uzL7.cn/91195.Doc
dhz.aa5uzL7.cn/39199.Doc
dhz.aa5uzL7.cn/11179.Doc
dhz.aa5uzL7.cn/11399.Doc
dhz.aa5uzL7.cn/75597.Doc
dhz.aa5uzL7.cn/71711.Doc
dhl.aa5uzL7.cn/71160.Doc
dhl.aa5uzL7.cn/75379.Doc
dhl.aa5uzL7.cn/73951.Doc
dhl.aa5uzL7.cn/11391.Doc
dhl.aa5uzL7.cn/17315.Doc
dhl.aa5uzL7.cn/31711.Doc
dhl.aa5uzL7.cn/75573.Doc
dhl.aa5uzL7.cn/57913.Doc
dhl.aa5uzL7.cn/19351.Doc
dhl.aa5uzL7.cn/91739.Doc
dhk.aa5uzL7.cn/73115.Doc
dhk.aa5uzL7.cn/57517.Doc
dhk.aa5uzL7.cn/39371.Doc
dhk.aa5uzL7.cn/73919.Doc
dhk.aa5uzL7.cn/31753.Doc
dhk.aa5uzL7.cn/1.Doc
dhk.aa5uzL7.cn/99113.Doc
dhk.aa5uzL7.cn/93535.Doc
dhk.aa5uzL7.cn/95137.Doc
dhk.aa5uzL7.cn/13131.Doc
dhj.aa5uzL7.cn/31751.Doc
dhj.aa5uzL7.cn/39395.Doc
dhj.aa5uzL7.cn/19957.Doc
dhj.aa5uzL7.cn/52284.Doc
dhj.aa5uzL7.cn/73755.Doc
dhj.aa5uzL7.cn/19599.Doc
dhj.aa5uzL7.cn/46226.Doc
dhj.aa5uzL7.cn/71537.Doc
dhj.aa5uzL7.cn/79795.Doc
dhj.aa5uzL7.cn/95573.Doc
dhh.aa5uzL7.cn/11915.Doc
dhh.aa5uzL7.cn/82460.Doc
dhh.aa5uzL7.cn/64846.Doc
dhh.aa5uzL7.cn/26800.Doc
dhh.aa5uzL7.cn/06286.Doc
dhh.aa5uzL7.cn/79317.Doc
dhh.aa5uzL7.cn/24646.Doc
dhh.aa5uzL7.cn/48008.Doc
dhh.aa5uzL7.cn/15133.Doc
dhh.aa5uzL7.cn/57337.Doc
dhg.aa5uzL7.cn/99331.Doc
dhg.aa5uzL7.cn/35775.Doc
dhg.aa5uzL7.cn/93571.Doc
dhg.aa5uzL7.cn/73133.Doc
dhg.aa5uzL7.cn/99193.Doc
dhg.aa5uzL7.cn/33911.Doc
dhg.aa5uzL7.cn/91559.Doc
dhg.aa5uzL7.cn/51577.Doc
dhg.aa5uzL7.cn/00246.Doc
dhg.aa5uzL7.cn/82482.Doc
dhf.aa5uzL7.cn/79351.Doc
dhf.aa5uzL7.cn/93391.Doc
dhf.aa5uzL7.cn/99399.Doc
dhf.aa5uzL7.cn/84424.Doc
dhf.aa5uzL7.cn/35975.Doc
dhf.aa5uzL7.cn/57937.Doc
dhf.aa5uzL7.cn/71731.Doc
dhf.aa5uzL7.cn/91371.Doc
dhf.aa5uzL7.cn/95153.Doc
dhf.aa5uzL7.cn/64484.Doc
dhd.aa5uzL7.cn/08264.Doc
dhd.aa5uzL7.cn/33551.Doc
dhd.aa5uzL7.cn/77979.Doc
dhd.aa5uzL7.cn/77135.Doc
dhd.aa5uzL7.cn/59577.Doc
dhd.aa5uzL7.cn/79593.Doc
dhd.aa5uzL7.cn/55119.Doc
dhd.aa5uzL7.cn/57737.Doc
dhd.aa5uzL7.cn/53797.Doc
dhd.aa5uzL7.cn/73955.Doc
dhs.aa5uzL7.cn/35315.Doc
dhs.aa5uzL7.cn/55775.Doc
dhs.aa5uzL7.cn/73993.Doc
dhs.aa5uzL7.cn/37599.Doc
dhs.aa5uzL7.cn/39717.Doc
dhs.aa5uzL7.cn/91795.Doc
dhs.aa5uzL7.cn/93931.Doc
dhs.aa5uzL7.cn/73133.Doc
dhs.aa5uzL7.cn/17357.Doc
dhs.aa5uzL7.cn/97559.Doc
dha.aa5uzL7.cn/79537.Doc
dha.aa5uzL7.cn/75591.Doc
dha.aa5uzL7.cn/39993.Doc
dha.aa5uzL7.cn/99935.Doc
dha.aa5uzL7.cn/11571.Doc
dha.aa5uzL7.cn/15579.Doc
dha.aa5uzL7.cn/51131.Doc
dha.aa5uzL7.cn/51595.Doc
dha.aa5uzL7.cn/51153.Doc
dha.aa5uzL7.cn/55775.Doc
dhp.aa5uzL7.cn/42424.Doc
dhp.aa5uzL7.cn/33119.Doc
dhp.aa5uzL7.cn/19359.Doc
dhp.aa5uzL7.cn/75331.Doc
dhp.aa5uzL7.cn/95953.Doc
dhp.aa5uzL7.cn/08082.Doc
dhp.aa5uzL7.cn/55173.Doc
dhp.aa5uzL7.cn/93771.Doc
dhp.aa5uzL7.cn/95599.Doc
dhp.aa5uzL7.cn/99111.Doc
dho.aa5uzL7.cn/55539.Doc
dho.aa5uzL7.cn/57733.Doc
dho.aa5uzL7.cn/93937.Doc
dho.aa5uzL7.cn/11951.Doc
dho.aa5uzL7.cn/93711.Doc
dho.aa5uzL7.cn/71795.Doc
dho.aa5uzL7.cn/37391.Doc
dho.aa5uzL7.cn/73115.Doc
dho.aa5uzL7.cn/59711.Doc
dho.aa5uzL7.cn/97357.Doc
dhi.aa5uzL7.cn/15975.Doc
dhi.aa5uzL7.cn/53917.Doc
dhi.aa5uzL7.cn/99537.Doc
dhi.aa5uzL7.cn/71957.Doc
dhi.aa5uzL7.cn/51359.Doc
dhi.aa5uzL7.cn/59537.Doc
dhi.aa5uzL7.cn/77517.Doc
dhi.aa5uzL7.cn/73175.Doc
dhi.aa5uzL7.cn/91311.Doc
dhi.aa5uzL7.cn/31771.Doc
dhu.aa5uzL7.cn/35759.Doc
dhu.aa5uzL7.cn/37533.Doc
dhu.aa5uzL7.cn/51799.Doc
dhu.aa5uzL7.cn/97919.Doc
dhu.aa5uzL7.cn/39915.Doc
dhu.aa5uzL7.cn/79519.Doc
dhu.aa5uzL7.cn/91597.Doc
dhu.aa5uzL7.cn/33395.Doc
dhu.aa5uzL7.cn/35199.Doc
dhu.aa5uzL7.cn/59577.Doc
dhy.aa5uzL7.cn/97111.Doc
dhy.aa5uzL7.cn/31513.Doc
dhy.aa5uzL7.cn/91771.Doc
dhy.aa5uzL7.cn/35715.Doc
dhy.aa5uzL7.cn/79731.Doc
dhy.aa5uzL7.cn/59791.Doc
dhy.aa5uzL7.cn/95731.Doc
dhy.aa5uzL7.cn/91517.Doc
dhy.aa5uzL7.cn/13157.Doc
dhy.aa5uzL7.cn/35711.Doc
dht.aa5uzL7.cn/73135.Doc
dht.aa5uzL7.cn/31753.Doc
dht.aa5uzL7.cn/17991.Doc
dht.aa5uzL7.cn/51335.Doc
dht.aa5uzL7.cn/71957.Doc
dht.aa5uzL7.cn/77975.Doc
dht.aa5uzL7.cn/93939.Doc
dht.aa5uzL7.cn/39395.Doc
dht.aa5uzL7.cn/95179.Doc
dht.aa5uzL7.cn/57771.Doc
dhr.aa5uzL7.cn/82688.Doc
dhr.aa5uzL7.cn/68002.Doc
dhr.aa5uzL7.cn/51573.Doc
dhr.aa5uzL7.cn/51131.Doc
dhr.aa5uzL7.cn/59777.Doc
dhr.aa5uzL7.cn/71355.Doc
dhr.aa5uzL7.cn/35313.Doc
dhr.aa5uzL7.cn/71593.Doc
dhr.aa5uzL7.cn/71539.Doc
dhr.aa5uzL7.cn/35757.Doc
dhe.aa5uzL7.cn/99991.Doc
dhe.aa5uzL7.cn/73777.Doc
dhe.aa5uzL7.cn/39779.Doc
dhe.aa5uzL7.cn/93133.Doc
dhe.aa5uzL7.cn/33979.Doc
dhe.aa5uzL7.cn/13955.Doc
dhe.aa5uzL7.cn/15751.Doc
dhe.aa5uzL7.cn/39337.Doc
dhe.aa5uzL7.cn/71917.Doc
dhe.aa5uzL7.cn/84866.Doc
dhw.aa5uzL7.cn/93171.Doc
dhw.aa5uzL7.cn/75535.Doc
dhw.aa5uzL7.cn/59997.Doc
dhw.aa5uzL7.cn/51559.Doc
dhw.aa5uzL7.cn/91973.Doc
dhw.aa5uzL7.cn/55337.Doc
dhw.aa5uzL7.cn/35733.Doc
dhw.aa5uzL7.cn/26462.Doc
dhw.aa5uzL7.cn/71313.Doc
dhw.aa5uzL7.cn/33139.Doc
dhq.aa5uzL7.cn/79111.Doc
dhq.aa5uzL7.cn/95935.Doc
dhq.aa5uzL7.cn/55595.Doc
dhq.aa5uzL7.cn/57571.Doc
dhq.aa5uzL7.cn/35531.Doc
dhq.aa5uzL7.cn/46066.Doc
dhq.aa5uzL7.cn/99753.Doc
dhq.aa5uzL7.cn/59517.Doc
dhq.aa5uzL7.cn/53711.Doc
dhq.aa5uzL7.cn/93579.Doc
dgm.aa5uzL7.cn/71579.Doc
dgm.aa5uzL7.cn/33513.Doc
dgm.aa5uzL7.cn/13395.Doc
dgm.aa5uzL7.cn/73351.Doc
dgm.aa5uzL7.cn/91199.Doc
dgm.aa5uzL7.cn/19399.Doc
dgm.aa5uzL7.cn/13935.Doc
dgm.aa5uzL7.cn/13131.Doc
dgm.aa5uzL7.cn/53579.Doc
dgm.aa5uzL7.cn/17975.Doc
dgn.aa5uzL7.cn/33191.Doc
dgn.aa5uzL7.cn/99537.Doc
dgn.aa5uzL7.cn/79719.Doc
dgn.aa5uzL7.cn/98304.Doc
dgn.aa5uzL7.cn/33599.Doc
dgn.aa5uzL7.cn/15511.Doc
dgn.aa5uzL7.cn/13517.Doc
dgn.aa5uzL7.cn/55951.Doc
dgn.aa5uzL7.cn/39733.Doc
dgn.aa5uzL7.cn/55357.Doc
dgb.aa5uzL7.cn/97157.Doc
dgb.aa5uzL7.cn/35579.Doc
dgb.aa5uzL7.cn/51119.Doc
dgb.aa5uzL7.cn/73973.Doc
dgb.aa5uzL7.cn/77753.Doc
dgb.aa5uzL7.cn/53553.Doc
dgb.aa5uzL7.cn/77593.Doc
dgb.aa5uzL7.cn/13513.Doc
dgb.aa5uzL7.cn/13573.Doc
dgb.aa5uzL7.cn/95915.Doc
dgv.aa5uzL7.cn/75157.Doc
dgv.aa5uzL7.cn/31711.Doc
dgv.aa5uzL7.cn/71957.Doc
dgv.aa5uzL7.cn/55979.Doc
dgv.aa5uzL7.cn/31359.Doc
dgv.aa5uzL7.cn/75353.Doc
dgv.aa5uzL7.cn/73353.Doc
dgv.aa5uzL7.cn/15759.Doc
dgv.aa5uzL7.cn/75979.Doc
dgv.aa5uzL7.cn/91193.Doc
dgc.aa5uzL7.cn/19735.Doc
dgc.aa5uzL7.cn/51339.Doc
dgc.aa5uzL7.cn/13191.Doc
dgc.aa5uzL7.cn/15975.Doc
dgc.aa5uzL7.cn/15315.Doc
dgc.aa5uzL7.cn/51195.Doc
dgc.aa5uzL7.cn/71955.Doc
dgc.aa5uzL7.cn/39593.Doc
dgc.aa5uzL7.cn/19595.Doc
dgc.aa5uzL7.cn/11773.Doc
dgx.aa5uzL7.cn/17311.Doc
dgx.aa5uzL7.cn/17597.Doc
dgx.aa5uzL7.cn/22224.Doc
dgx.aa5uzL7.cn/11713.Doc
dgx.aa5uzL7.cn/91533.Doc
dgx.aa5uzL7.cn/19957.Doc
dgx.aa5uzL7.cn/53753.Doc
dgx.aa5uzL7.cn/11171.Doc
dgx.aa5uzL7.cn/06420.Doc
dgx.aa5uzL7.cn/55931.Doc
dgz.aa5uzL7.cn/31375.Doc
dgz.aa5uzL7.cn/37915.Doc
dgz.aa5uzL7.cn/17715.Doc
dgz.aa5uzL7.cn/82880.Doc
dgz.aa5uzL7.cn/95753.Doc
dgz.aa5uzL7.cn/53319.Doc
dgz.aa5uzL7.cn/71557.Doc
dgz.aa5uzL7.cn/39731.Doc
dgz.aa5uzL7.cn/31315.Doc
dgz.aa5uzL7.cn/88226.Doc
dgl.aa5uzL7.cn/59397.Doc
dgl.aa5uzL7.cn/19137.Doc
dgl.aa5uzL7.cn/37771.Doc
dgl.aa5uzL7.cn/73195.Doc
dgl.aa5uzL7.cn/75773.Doc
dgl.aa5uzL7.cn/59595.Doc
dgl.aa5uzL7.cn/97319.Doc
dgl.aa5uzL7.cn/71537.Doc
dgl.aa5uzL7.cn/37195.Doc
dgl.aa5uzL7.cn/17777.Doc
dgk.aa5uzL7.cn/91377.Doc
dgk.aa5uzL7.cn/79155.Doc
dgk.aa5uzL7.cn/97977.Doc
dgk.aa5uzL7.cn/99993.Doc
dgk.aa5uzL7.cn/73973.Doc
dgk.aa5uzL7.cn/7.Doc
dgk.aa5uzL7.cn/53715.Doc
dgk.aa5uzL7.cn/31545.Doc
dgk.aa5uzL7.cn/39357.Doc
dgk.aa5uzL7.cn/51555.Doc
dgj.aa5uzL7.cn/17337.Doc
dgj.aa5uzL7.cn/75711.Doc
dgj.aa5uzL7.cn/99357.Doc
dgj.aa5uzL7.cn/57575.Doc
dgj.aa5uzL7.cn/17519.Doc
dgj.aa5uzL7.cn/39535.Doc
dgj.aa5uzL7.cn/55939.Doc
dgj.aa5uzL7.cn/95151.Doc
dgj.aa5uzL7.cn/59979.Doc
dgj.aa5uzL7.cn/99739.Doc
dgh.aa5uzL7.cn/15915.Doc
dgh.aa5uzL7.cn/13111.Doc
dgh.aa5uzL7.cn/35355.Doc
dgh.aa5uzL7.cn/35399.Doc
dgh.aa5uzL7.cn/99951.Doc
dgh.aa5uzL7.cn/17713.Doc
dgh.aa5uzL7.cn/77519.Doc
dgh.aa5uzL7.cn/77579.Doc
dgh.aa5uzL7.cn/77575.Doc
dgh.aa5uzL7.cn/33715.Doc
dgg.aa5uzL7.cn/33599.Doc
dgg.aa5uzL7.cn/55513.Doc
dgg.aa5uzL7.cn/82868.Doc
dgg.aa5uzL7.cn/73193.Doc
dgg.aa5uzL7.cn/37595.Doc
dgg.aa5uzL7.cn/75799.Doc
dgg.aa5uzL7.cn/71793.Doc
dgg.aa5uzL7.cn/99173.Doc
dgg.aa5uzL7.cn/08480.Doc
dgg.aa5uzL7.cn/75755.Doc
dgf.aa5uzL7.cn/39315.Doc
dgf.aa5uzL7.cn/57931.Doc
dgf.aa5uzL7.cn/33793.Doc
dgf.aa5uzL7.cn/75777.Doc
dgf.aa5uzL7.cn/77319.Doc
dgf.aa5uzL7.cn/95317.Doc
dgf.aa5uzL7.cn/11599.Doc
dgf.aa5uzL7.cn/91515.Doc
dgf.aa5uzL7.cn/53537.Doc
dgf.aa5uzL7.cn/55933.Doc
dgd.aa5uzL7.cn/57331.Doc
dgd.aa5uzL7.cn/39773.Doc
dgd.aa5uzL7.cn/99395.Doc
dgd.aa5uzL7.cn/77751.Doc
dgd.aa5uzL7.cn/77913.Doc
dgd.aa5uzL7.cn/00826.Doc
dgd.aa5uzL7.cn/60284.Doc
dgd.aa5uzL7.cn/77597.Doc
dgd.aa5uzL7.cn/53553.Doc
dgd.aa5uzL7.cn/71519.Doc
dgs.aa5uzL7.cn/17799.Doc
dgs.aa5uzL7.cn/73939.Doc
dgs.aa5uzL7.cn/68642.Doc
dgs.aa5uzL7.cn/39151.Doc
dgs.aa5uzL7.cn/19119.Doc
dgs.aa5uzL7.cn/37153.Doc
dgs.aa5uzL7.cn/57171.Doc
dgs.aa5uzL7.cn/93151.Doc
dgs.aa5uzL7.cn/91113.Doc
dgs.aa5uzL7.cn/75393.Doc
dga.aa5uzL7.cn/59937.Doc
dga.aa5uzL7.cn/93577.Doc
dga.aa5uzL7.cn/55917.Doc
dga.aa5uzL7.cn/91179.Doc
dga.aa5uzL7.cn/91157.Doc
dga.aa5uzL7.cn/97599.Doc
dga.aa5uzL7.cn/37313.Doc
dga.aa5uzL7.cn/15959.Doc
dga.aa5uzL7.cn/15197.Doc
dga.aa5uzL7.cn/51737.Doc
dgp.aa5uzL7.cn/53735.Doc
dgp.aa5uzL7.cn/73519.Doc
dgp.aa5uzL7.cn/33373.Doc
dgp.aa5uzL7.cn/53319.Doc
dgp.aa5uzL7.cn/06688.Doc
dgp.aa5uzL7.cn/26684.Doc
dgp.aa5uzL7.cn/51939.Doc
dgp.aa5uzL7.cn/19575.Doc
dgp.aa5uzL7.cn/91195.Doc
dgp.aa5uzL7.cn/51511.Doc
dgo.aa5uzL7.cn/59573.Doc
dgo.aa5uzL7.cn/59557.Doc
dgo.aa5uzL7.cn/19117.Doc
dgo.aa5uzL7.cn/77317.Doc
dgo.aa5uzL7.cn/17917.Doc
dgo.aa5uzL7.cn/51731.Doc
dgo.aa5uzL7.cn/75515.Doc
dgo.aa5uzL7.cn/95597.Doc
dgo.aa5uzL7.cn/68660.Doc
dgo.aa5uzL7.cn/95791.Doc
dgi.aa5uzL7.cn/24840.Doc
dgi.aa5uzL7.cn/08264.Doc
dgi.aa5uzL7.cn/77313.Doc
dgi.aa5uzL7.cn/59179.Doc
dgi.aa5uzL7.cn/55599.Doc
dgi.aa5uzL7.cn/51173.Doc
dgi.aa5uzL7.cn/19535.Doc
dgi.aa5uzL7.cn/93537.Doc
dgi.aa5uzL7.cn/51917.Doc
dgi.aa5uzL7.cn/51319.Doc
dgu.aa5uzL7.cn/33799.Doc
dgu.aa5uzL7.cn/06262.Doc
dgu.aa5uzL7.cn/97791.Doc
dgu.aa5uzL7.cn/13371.Doc
dgu.aa5uzL7.cn/35771.Doc
dgu.aa5uzL7.cn/99757.Doc
dgu.aa5uzL7.cn/91935.Doc
dgu.aa5uzL7.cn/31777.Doc
dgu.aa5uzL7.cn/31191.Doc
dgu.aa5uzL7.cn/79595.Doc
dgy.aa5uzL7.cn/73579.Doc
dgy.aa5uzL7.cn/91371.Doc
dgy.aa5uzL7.cn/44222.Doc
dgy.aa5uzL7.cn/13753.Doc
dgy.aa5uzL7.cn/37531.Doc
dgy.aa5uzL7.cn/39717.Doc
dgy.aa5uzL7.cn/39719.Doc
dgy.aa5uzL7.cn/37153.Doc
dgy.aa5uzL7.cn/35317.Doc
dgy.aa5uzL7.cn/26840.Doc
dgt.aa5uzL7.cn/71779.Doc
dgt.aa5uzL7.cn/95935.Doc
dgt.aa5uzL7.cn/37717.Doc
dgt.aa5uzL7.cn/35911.Doc
dgt.aa5uzL7.cn/11393.Doc
dgt.aa5uzL7.cn/11911.Doc
dgt.aa5uzL7.cn/44484.Doc
dgt.aa5uzL7.cn/55755.Doc
dgt.aa5uzL7.cn/75937.Doc
dgt.aa5uzL7.cn/31953.Doc
dgr.aa5uzL7.cn/15993.Doc
dgr.aa5uzL7.cn/33157.Doc
dgr.aa5uzL7.cn/08242.Doc
dgr.aa5uzL7.cn/33395.Doc
dgr.aa5uzL7.cn/77357.Doc
dgr.aa5uzL7.cn/11399.Doc
dgr.aa5uzL7.cn/99197.Doc
dgr.aa5uzL7.cn/95951.Doc
dgr.aa5uzL7.cn/57331.Doc
dgr.aa5uzL7.cn/35975.Doc
dge.aa5uzL7.cn/40022.Doc
dge.aa5uzL7.cn/17735.Doc
dge.aa5uzL7.cn/75711.Doc
dge.aa5uzL7.cn/57377.Doc
dge.aa5uzL7.cn/71919.Doc
dge.aa5uzL7.cn/37595.Doc
dge.aa5uzL7.cn/17551.Doc
dge.aa5uzL7.cn/20824.Doc
dge.aa5uzL7.cn/42008.Doc
dge.aa5uzL7.cn/86466.Doc
dgw.aa5uzL7.cn/31553.Doc
dgw.aa5uzL7.cn/15199.Doc
dgw.aa5uzL7.cn/57573.Doc
dgw.aa5uzL7.cn/31791.Doc
dgw.aa5uzL7.cn/77959.Doc
dgw.aa5uzL7.cn/31751.Doc
dgw.aa5uzL7.cn/15515.Doc
dgw.aa5uzL7.cn/53377.Doc
dgw.aa5uzL7.cn/11373.Doc
dgw.aa5uzL7.cn/71117.Doc
dgq.aa5uzL7.cn/53793.Doc
dgq.aa5uzL7.cn/71975.Doc
dgq.aa5uzL7.cn/24606.Doc
dgq.aa5uzL7.cn/15335.Doc
dgq.aa5uzL7.cn/93537.Doc
dgq.aa5uzL7.cn/13173.Doc
dgq.aa5uzL7.cn/51197.Doc
dgq.aa5uzL7.cn/51171.Doc
dgq.aa5uzL7.cn/55371.Doc
dgq.aa5uzL7.cn/37913.Doc
dfm.aa5uzL7.cn/35719.Doc
dfm.aa5uzL7.cn/79353.Doc
dfm.aa5uzL7.cn/31331.Doc
dfm.aa5uzL7.cn/75535.Doc
dfm.aa5uzL7.cn/17315.Doc
dfm.aa5uzL7.cn/17975.Doc
dfm.aa5uzL7.cn/39997.Doc
dfm.aa5uzL7.cn/39579.Doc
dfm.aa5uzL7.cn/19973.Doc
dfm.aa5uzL7.cn/75391.Doc
