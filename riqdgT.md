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

qxy.LreLnc.cn/57175.Doc
qxy.LreLnc.cn/55337.Doc
qxy.LreLnc.cn/35537.Doc
qxy.LreLnc.cn/24260.Doc
qxy.LreLnc.cn/02088.Doc
qxy.LreLnc.cn/71575.Doc
qxy.LreLnc.cn/19739.Doc
qxy.LreLnc.cn/91193.Doc
qxy.LreLnc.cn/19339.Doc
qxy.LreLnc.cn/11919.Doc
qxt.LreLnc.cn/71957.Doc
qxt.LreLnc.cn/33751.Doc
qxt.LreLnc.cn/15197.Doc
qxt.LreLnc.cn/77175.Doc
qxt.LreLnc.cn/84006.Doc
qxt.LreLnc.cn/73379.Doc
qxt.LreLnc.cn/59519.Doc
qxt.LreLnc.cn/39997.Doc
qxt.LreLnc.cn/95959.Doc
qxt.LreLnc.cn/77355.Doc
qxr.LreLnc.cn/51915.Doc
qxr.LreLnc.cn/51311.Doc
qxr.LreLnc.cn/97195.Doc
qxr.LreLnc.cn/48242.Doc
qxr.LreLnc.cn/11739.Doc
qxr.LreLnc.cn/71793.Doc
qxr.LreLnc.cn/11597.Doc
qxr.LreLnc.cn/22826.Doc
qxr.LreLnc.cn/00286.Doc
qxr.LreLnc.cn/57935.Doc
qxe.LreLnc.cn/37977.Doc
qxe.LreLnc.cn/17935.Doc
qxe.LreLnc.cn/79371.Doc
qxe.LreLnc.cn/62026.Doc
qxe.LreLnc.cn/53391.Doc
qxe.LreLnc.cn/19737.Doc
qxe.LreLnc.cn/51759.Doc
qxe.LreLnc.cn/71339.Doc
qxe.LreLnc.cn/15973.Doc
qxe.LreLnc.cn/93737.Doc
qxw.LreLnc.cn/11537.Doc
qxw.LreLnc.cn/97173.Doc
qxw.LreLnc.cn/19515.Doc
qxw.LreLnc.cn/79511.Doc
qxw.LreLnc.cn/11359.Doc
qxw.LreLnc.cn/89713.Doc
qxw.LreLnc.cn/31713.Doc
qxw.LreLnc.cn/13175.Doc
qxw.LreLnc.cn/31359.Doc
qxw.LreLnc.cn/39517.Doc
qxq.LreLnc.cn/39519.Doc
qxq.LreLnc.cn/97175.Doc
qxq.LreLnc.cn/73917.Doc
qxq.LreLnc.cn/57131.Doc
qxq.LreLnc.cn/20040.Doc
qxq.LreLnc.cn/31391.Doc
qxq.LreLnc.cn/35793.Doc
qxq.LreLnc.cn/37391.Doc
qxq.LreLnc.cn/57737.Doc
qxq.LreLnc.cn/45548.Doc
qzm.LreLnc.cn/53531.Doc
qzm.LreLnc.cn/33595.Doc
qzm.LreLnc.cn/37933.Doc
qzm.LreLnc.cn/19157.Doc
qzm.LreLnc.cn/35319.Doc
qzm.LreLnc.cn/31755.Doc
qzm.LreLnc.cn/55753.Doc
qzm.LreLnc.cn/51957.Doc
qzm.LreLnc.cn/91915.Doc
qzm.LreLnc.cn/19137.Doc
qzn.LreLnc.cn/93597.Doc
qzn.LreLnc.cn/33313.Doc
qzn.LreLnc.cn/77597.Doc
qzn.LreLnc.cn/57757.Doc
qzn.LreLnc.cn/51157.Doc
qzn.LreLnc.cn/55971.Doc
qzn.LreLnc.cn/37911.Doc
qzn.LreLnc.cn/55193.Doc
qzn.LreLnc.cn/13599.Doc
qzn.LreLnc.cn/59131.Doc
qzb.LreLnc.cn/33577.Doc
qzb.LreLnc.cn/19177.Doc
qzb.LreLnc.cn/95751.Doc
qzb.LreLnc.cn/37593.Doc
qzb.LreLnc.cn/75315.Doc
qzb.LreLnc.cn/31131.Doc
qzb.LreLnc.cn/15395.Doc
qzb.LreLnc.cn/55113.Doc
qzb.LreLnc.cn/91953.Doc
qzb.LreLnc.cn/73911.Doc
qzv.LreLnc.cn/39355.Doc
qzv.LreLnc.cn/35131.Doc
qzv.LreLnc.cn/15571.Doc
qzv.LreLnc.cn/39913.Doc
qzv.LreLnc.cn/33957.Doc
qzv.LreLnc.cn/11159.Doc
qzv.LreLnc.cn/57537.Doc
qzv.LreLnc.cn/31595.Doc
qzv.LreLnc.cn/64282.Doc
qzv.LreLnc.cn/79193.Doc
qzc.LreLnc.cn/79337.Doc
qzc.LreLnc.cn/55777.Doc
qzc.LreLnc.cn/75753.Doc
qzc.LreLnc.cn/75535.Doc
qzc.LreLnc.cn/37795.Doc
qzc.LreLnc.cn/20244.Doc
qzc.LreLnc.cn/15533.Doc
qzc.LreLnc.cn/46842.Doc
qzc.LreLnc.cn/35313.Doc
qzc.LreLnc.cn/93933.Doc
qzx.LreLnc.cn/77999.Doc
qzx.LreLnc.cn/75777.Doc
qzx.LreLnc.cn/7.Doc
qzx.LreLnc.cn/69502.Doc
qzx.LreLnc.cn/95739.Doc
qzx.LreLnc.cn/31351.Doc
qzx.LreLnc.cn/91115.Doc
qzx.LreLnc.cn/55939.Doc
qzx.LreLnc.cn/91357.Doc
qzx.LreLnc.cn/59535.Doc
qzz.LreLnc.cn/19555.Doc
qzz.LreLnc.cn/15533.Doc
qzz.LreLnc.cn/57397.Doc
qzz.LreLnc.cn/57115.Doc
qzz.LreLnc.cn/95515.Doc
qzz.LreLnc.cn/51735.Doc
qzz.LreLnc.cn/17133.Doc
qzz.LreLnc.cn/26848.Doc
qzz.LreLnc.cn/64664.Doc
qzz.LreLnc.cn/77957.Doc
qzl.LreLnc.cn/35775.Doc
qzl.LreLnc.cn/77731.Doc
qzl.LreLnc.cn/11799.Doc
qzl.LreLnc.cn/17373.Doc
qzl.LreLnc.cn/77773.Doc
qzl.LreLnc.cn/31591.Doc
qzl.LreLnc.cn/57733.Doc
qzl.LreLnc.cn/15779.Doc
qzl.LreLnc.cn/19599.Doc
qzl.LreLnc.cn/51719.Doc
qzk.LreLnc.cn/13537.Doc
qzk.LreLnc.cn/15393.Doc
qzk.LreLnc.cn/11739.Doc
qzk.LreLnc.cn/75139.Doc
qzk.LreLnc.cn/71359.Doc
qzk.LreLnc.cn/66624.Doc
qzk.LreLnc.cn/11311.Doc
qzk.LreLnc.cn/97779.Doc
qzk.LreLnc.cn/75351.Doc
qzk.LreLnc.cn/55113.Doc
qzj.LreLnc.cn/77999.Doc
qzj.LreLnc.cn/79755.Doc
qzj.LreLnc.cn/75915.Doc
qzj.LreLnc.cn/13739.Doc
qzj.LreLnc.cn/91317.Doc
qzj.LreLnc.cn/57179.Doc
qzj.LreLnc.cn/55951.Doc
qzj.LreLnc.cn/33151.Doc
qzj.LreLnc.cn/55359.Doc
qzj.LreLnc.cn/57399.Doc
qzh.LreLnc.cn/73357.Doc
qzh.LreLnc.cn/11511.Doc
qzh.LreLnc.cn/55779.Doc
qzh.LreLnc.cn/17113.Doc
qzh.LreLnc.cn/35777.Doc
qzh.LreLnc.cn/13731.Doc
qzh.LreLnc.cn/33539.Doc
qzh.LreLnc.cn/10385.Doc
qzh.LreLnc.cn/75577.Doc
qzh.LreLnc.cn/51317.Doc
qzg.LreLnc.cn/97913.Doc
qzg.LreLnc.cn/17915.Doc
qzg.LreLnc.cn/97993.Doc
qzg.LreLnc.cn/51113.Doc
qzg.LreLnc.cn/53937.Doc
qzg.LreLnc.cn/37197.Doc
qzg.LreLnc.cn/35333.Doc
qzg.LreLnc.cn/19595.Doc
qzg.LreLnc.cn/79733.Doc
qzg.LreLnc.cn/99993.Doc
qzf.LreLnc.cn/17911.Doc
qzf.LreLnc.cn/15555.Doc
qzf.LreLnc.cn/19591.Doc
qzf.LreLnc.cn/15933.Doc
qzf.LreLnc.cn/73773.Doc
qzf.LreLnc.cn/77197.Doc
qzf.LreLnc.cn/59393.Doc
qzf.LreLnc.cn/99337.Doc
qzf.LreLnc.cn/57157.Doc
qzf.LreLnc.cn/48602.Doc
qzd.LreLnc.cn/99377.Doc
qzd.LreLnc.cn/75735.Doc
qzd.LreLnc.cn/33315.Doc
qzd.LreLnc.cn/19777.Doc
qzd.LreLnc.cn/93151.Doc
qzd.LreLnc.cn/46400.Doc
qzd.LreLnc.cn/79735.Doc
qzd.LreLnc.cn/11551.Doc
qzd.LreLnc.cn/73535.Doc
qzd.LreLnc.cn/55119.Doc
qzs.LreLnc.cn/97395.Doc
qzs.LreLnc.cn/77755.Doc
qzs.LreLnc.cn/82242.Doc
qzs.LreLnc.cn/51191.Doc
qzs.LreLnc.cn/79719.Doc
qzs.LreLnc.cn/13973.Doc
qzs.LreLnc.cn/33399.Doc
qzs.LreLnc.cn/97597.Doc
qzs.LreLnc.cn/88682.Doc
qzs.LreLnc.cn/97359.Doc
qza.LreLnc.cn/55373.Doc
qza.LreLnc.cn/75919.Doc
qza.LreLnc.cn/51133.Doc
qza.LreLnc.cn/75137.Doc
qza.LreLnc.cn/79737.Doc
qza.LreLnc.cn/71937.Doc
qza.LreLnc.cn/82628.Doc
qza.LreLnc.cn/35559.Doc
qza.LreLnc.cn/15575.Doc
qza.LreLnc.cn/13993.Doc
qzp.LreLnc.cn/19733.Doc
qzp.LreLnc.cn/71331.Doc
qzp.LreLnc.cn/55397.Doc
qzp.LreLnc.cn/37195.Doc
qzp.LreLnc.cn/55195.Doc
qzp.LreLnc.cn/55793.Doc
qzp.LreLnc.cn/95551.Doc
qzp.LreLnc.cn/97593.Doc
qzp.LreLnc.cn/91319.Doc
qzp.LreLnc.cn/35171.Doc
qzo.LreLnc.cn/11557.Doc
qzo.LreLnc.cn/57915.Doc
qzo.LreLnc.cn/57953.Doc
qzo.LreLnc.cn/11313.Doc
qzo.LreLnc.cn/71971.Doc
qzo.LreLnc.cn/35775.Doc
qzo.LreLnc.cn/84248.Doc
qzo.LreLnc.cn/84668.Doc
qzo.LreLnc.cn/3.Doc
qzo.LreLnc.cn/79317.Doc
qzi.LreLnc.cn/93579.Doc
qzi.LreLnc.cn/59117.Doc
qzi.LreLnc.cn/11715.Doc
qzi.LreLnc.cn/26486.Doc
qzi.LreLnc.cn/15759.Doc
qzi.LreLnc.cn/97957.Doc
qzi.LreLnc.cn/91935.Doc
qzi.LreLnc.cn/06826.Doc
qzi.LreLnc.cn/11371.Doc
qzi.LreLnc.cn/93353.Doc
qzu.LreLnc.cn/15977.Doc
qzu.LreLnc.cn/39935.Doc
qzu.LreLnc.cn/33951.Doc
qzu.LreLnc.cn/17993.Doc
qzu.LreLnc.cn/73777.Doc
qzu.LreLnc.cn/51797.Doc
qzu.LreLnc.cn/95717.Doc
qzu.LreLnc.cn/13377.Doc
qzu.LreLnc.cn/97977.Doc
qzu.LreLnc.cn/73751.Doc
qzy.LreLnc.cn/93559.Doc
qzy.LreLnc.cn/59597.Doc
qzy.LreLnc.cn/77539.Doc
qzy.LreLnc.cn/71759.Doc
qzy.LreLnc.cn/95753.Doc
qzy.LreLnc.cn/11911.Doc
qzy.LreLnc.cn/97393.Doc
qzy.LreLnc.cn/79751.Doc
qzy.LreLnc.cn/13399.Doc
qzy.LreLnc.cn/59559.Doc
qzt.LreLnc.cn/11991.Doc
qzt.LreLnc.cn/93557.Doc
qzt.LreLnc.cn/79591.Doc
qzt.LreLnc.cn/11333.Doc
qzt.LreLnc.cn/71155.Doc
qzt.LreLnc.cn/15575.Doc
qzt.LreLnc.cn/99519.Doc
qzt.LreLnc.cn/55933.Doc
qzt.LreLnc.cn/55991.Doc
qzt.LreLnc.cn/59717.Doc
qzr.LreLnc.cn/19711.Doc
qzr.LreLnc.cn/33395.Doc
qzr.LreLnc.cn/20064.Doc
qzr.LreLnc.cn/35793.Doc
qzr.LreLnc.cn/93795.Doc
qzr.LreLnc.cn/33317.Doc
qzr.LreLnc.cn/77599.Doc
qzr.LreLnc.cn/97533.Doc
qzr.LreLnc.cn/15517.Doc
qzr.LreLnc.cn/9.Doc
qze.LreLnc.cn/95395.Doc
qze.LreLnc.cn/71375.Doc
qze.LreLnc.cn/97157.Doc
qze.LreLnc.cn/79979.Doc
qze.LreLnc.cn/91733.Doc
qze.LreLnc.cn/95799.Doc
qze.LreLnc.cn/95399.Doc
qze.LreLnc.cn/71737.Doc
qze.LreLnc.cn/35591.Doc
qze.LreLnc.cn/37579.Doc
qzw.LreLnc.cn/55511.Doc
qzw.LreLnc.cn/37173.Doc
qzw.LreLnc.cn/35715.Doc
qzw.LreLnc.cn/79535.Doc
qzw.LreLnc.cn/99173.Doc
qzw.LreLnc.cn/35397.Doc
qzw.LreLnc.cn/17531.Doc
qzw.LreLnc.cn/99951.Doc
qzw.LreLnc.cn/71193.Doc
qzw.LreLnc.cn/77339.Doc
qzq.LreLnc.cn/55313.Doc
qzq.LreLnc.cn/13535.Doc
qzq.LreLnc.cn/91771.Doc
qzq.LreLnc.cn/17359.Doc
qzq.LreLnc.cn/73535.Doc
qzq.LreLnc.cn/53913.Doc
qzq.LreLnc.cn/91559.Doc
qzq.LreLnc.cn/59995.Doc
qzq.LreLnc.cn/97395.Doc
qzq.LreLnc.cn/51937.Doc
qlm.LreLnc.cn/51335.Doc
qlm.LreLnc.cn/11379.Doc
qlm.LreLnc.cn/37757.Doc
qlm.LreLnc.cn/19335.Doc
qlm.LreLnc.cn/37177.Doc
qlm.LreLnc.cn/91519.Doc
qlm.LreLnc.cn/79373.Doc
qlm.LreLnc.cn/91335.Doc
qlm.LreLnc.cn/97535.Doc
qlm.LreLnc.cn/33719.Doc
qln.LreLnc.cn/15733.Doc
qln.LreLnc.cn/17519.Doc
qln.LreLnc.cn/75315.Doc
qln.LreLnc.cn/39755.Doc
qln.LreLnc.cn/73173.Doc
qln.LreLnc.cn/17555.Doc
qln.LreLnc.cn/99577.Doc
qln.LreLnc.cn/75193.Doc
qln.LreLnc.cn/53955.Doc
qln.LreLnc.cn/97997.Doc
qlb.LreLnc.cn/77595.Doc
qlb.LreLnc.cn/17579.Doc
qlb.LreLnc.cn/64022.Doc
qlb.LreLnc.cn/59139.Doc
qlb.LreLnc.cn/73355.Doc
qlb.LreLnc.cn/73557.Doc
qlb.LreLnc.cn/39375.Doc
qlb.LreLnc.cn/84060.Doc
qlb.LreLnc.cn/97373.Doc
qlb.LreLnc.cn/99117.Doc
qlv.LreLnc.cn/53333.Doc
qlv.LreLnc.cn/33739.Doc
qlv.LreLnc.cn/99593.Doc
qlv.LreLnc.cn/22820.Doc
qlv.LreLnc.cn/00608.Doc
qlv.LreLnc.cn/79339.Doc
qlv.LreLnc.cn/39917.Doc
qlv.LreLnc.cn/83329.Doc
qlv.LreLnc.cn/95959.Doc
qlv.LreLnc.cn/79535.Doc
qlc.LreLnc.cn/37573.Doc
qlc.LreLnc.cn/46226.Doc
qlc.LreLnc.cn/19331.Doc
qlc.LreLnc.cn/99995.Doc
qlc.LreLnc.cn/48222.Doc
qlc.LreLnc.cn/35731.Doc
qlc.LreLnc.cn/73519.Doc
qlc.LreLnc.cn/19117.Doc
qlc.LreLnc.cn/95335.Doc
qlc.LreLnc.cn/99737.Doc
qlx.LreLnc.cn/99395.Doc
qlx.LreLnc.cn/97171.Doc
qlx.LreLnc.cn/15337.Doc
qlx.LreLnc.cn/31199.Doc
qlx.LreLnc.cn/51559.Doc
qlx.LreLnc.cn/35137.Doc
qlx.LreLnc.cn/15531.Doc
qlx.LreLnc.cn/71173.Doc
qlx.LreLnc.cn/48248.Doc
qlx.LreLnc.cn/51799.Doc
qlz.LreLnc.cn/73171.Doc
qlz.LreLnc.cn/35151.Doc
qlz.LreLnc.cn/11195.Doc
qlz.LreLnc.cn/39573.Doc
qlz.LreLnc.cn/01309.Doc
qlz.LreLnc.cn/31739.Doc
qlz.LreLnc.cn/73915.Doc
qlz.LreLnc.cn/75535.Doc
qlz.LreLnc.cn/42240.Doc
qlz.LreLnc.cn/97171.Doc
qll.LreLnc.cn/02424.Doc
qll.LreLnc.cn/51553.Doc
qll.LreLnc.cn/11999.Doc
qll.LreLnc.cn/13595.Doc
qll.LreLnc.cn/53315.Doc
qll.LreLnc.cn/59911.Doc
qll.LreLnc.cn/13353.Doc
qll.LreLnc.cn/71391.Doc
qll.LreLnc.cn/57171.Doc
qll.LreLnc.cn/82464.Doc
qlk.LreLnc.cn/86820.Doc
qlk.LreLnc.cn/11975.Doc
qlk.LreLnc.cn/73115.Doc
qlk.LreLnc.cn/64642.Doc
qlk.LreLnc.cn/75191.Doc
qlk.LreLnc.cn/37357.Doc
qlk.LreLnc.cn/75591.Doc
qlk.LreLnc.cn/15753.Doc
qlk.LreLnc.cn/19959.Doc
qlk.LreLnc.cn/11399.Doc
qlj.LreLnc.cn/55715.Doc
qlj.LreLnc.cn/19153.Doc
qlj.LreLnc.cn/91773.Doc
qlj.LreLnc.cn/93771.Doc
qlj.LreLnc.cn/95599.Doc
qlj.LreLnc.cn/48464.Doc
qlj.LreLnc.cn/91537.Doc
qlj.LreLnc.cn/51157.Doc
qlj.LreLnc.cn/51917.Doc
qlj.LreLnc.cn/95771.Doc
qlh.LreLnc.cn/91537.Doc
qlh.LreLnc.cn/13331.Doc
qlh.LreLnc.cn/35715.Doc
qlh.LreLnc.cn/99917.Doc
qlh.LreLnc.cn/55749.Doc
qlh.LreLnc.cn/13157.Doc
qlh.LreLnc.cn/93131.Doc
qlh.LreLnc.cn/97519.Doc
qlh.LreLnc.cn/71533.Doc
qlh.LreLnc.cn/55937.Doc
qlg.LreLnc.cn/42240.Doc
qlg.LreLnc.cn/75775.Doc
qlg.LreLnc.cn/95755.Doc
qlg.LreLnc.cn/53779.Doc
qlg.LreLnc.cn/28224.Doc
qlg.LreLnc.cn/93739.Doc
qlg.LreLnc.cn/15399.Doc
qlg.LreLnc.cn/19131.Doc
qlg.LreLnc.cn/59595.Doc
qlg.LreLnc.cn/17791.Doc
qlf.LreLnc.cn/53977.Doc
qlf.LreLnc.cn/15311.Doc
qlf.LreLnc.cn/11935.Doc
qlf.LreLnc.cn/39553.Doc
qlf.LreLnc.cn/97517.Doc
qlf.LreLnc.cn/57557.Doc
qlf.LreLnc.cn/31995.Doc
qlf.LreLnc.cn/00226.Doc
qlf.LreLnc.cn/55979.Doc
qlf.LreLnc.cn/37175.Doc
qld.LreLnc.cn/39517.Doc
qld.LreLnc.cn/51739.Doc
qld.LreLnc.cn/06280.Doc
qld.LreLnc.cn/13939.Doc
qld.LreLnc.cn/33737.Doc
qld.LreLnc.cn/15339.Doc
qld.LreLnc.cn/53131.Doc
qld.LreLnc.cn/15311.Doc
qld.LreLnc.cn/19571.Doc
qld.LreLnc.cn/77719.Doc
qls.LreLnc.cn/15973.Doc
qls.LreLnc.cn/26880.Doc
qls.LreLnc.cn/75935.Doc
qls.LreLnc.cn/73173.Doc
qls.LreLnc.cn/55997.Doc
qls.LreLnc.cn/93737.Doc
qls.LreLnc.cn/48406.Doc
qls.LreLnc.cn/57137.Doc
qls.LreLnc.cn/17373.Doc
qls.LreLnc.cn/08824.Doc
qla.LreLnc.cn/57513.Doc
qla.LreLnc.cn/19931.Doc
qla.LreLnc.cn/97331.Doc
qla.LreLnc.cn/11197.Doc
qla.LreLnc.cn/53939.Doc
qla.LreLnc.cn/53753.Doc
qla.LreLnc.cn/15731.Doc
qla.LreLnc.cn/73591.Doc
qla.LreLnc.cn/77931.Doc
qla.LreLnc.cn/79593.Doc
qlp.LreLnc.cn/15755.Doc
qlp.LreLnc.cn/95591.Doc
qlp.LreLnc.cn/93135.Doc
qlp.LreLnc.cn/31773.Doc
qlp.LreLnc.cn/53339.Doc
qlp.LreLnc.cn/51511.Doc
qlp.LreLnc.cn/5.Doc
qlp.LreLnc.cn/71177.Doc
qlp.LreLnc.cn/91535.Doc
qlp.LreLnc.cn/71799.Doc
