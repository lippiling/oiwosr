瀑寻烙巧越


Python @property 高级应用
===============================

@property 是 Python 内置的描述符装饰器，将方法调用伪装成属性访问。
本文深入讲解其高级用法。

1. @property 计算属性
-------------------------
将方法返回值当作属性访问，适合需要计算的场景。

class Circle:
    """圆形——半径变化时面积自动变化"""
    def __init__(self, radius: float):
        self.radius = radius

    @property
    def area(self) -> float:
        """计算属性：每次访问都重新计算"""
        return 3.14159 * self.radius ** 2

    @property
    def diameter(self) -> float:
        return self.radius * 2


c = Circle(5)
print(c.area)      # 78.53975——像访问属性一样使用
c.radius = 10
print(c.area)      # 314.159——自动更新


2. setter 与 deleter 验证
-------------------------------
通过 setter 在赋值时进行校验，deleter 处理删除操作。

class Person:
    """人员类——对年龄做范围校验"""
    def __init__(self, name: str, age: int):
        self.name = name
        self._age = age  # 使用私有属性存储

    @property
    def age(self) -> int:
        """getter：返回内部存储值"""
        return self._age

    @age.setter
    def age(self, value: int) -> None:
        """setter：赋值前校验范围"""
        if not isinstance(value, int):
            raise TypeError(f"年龄必须是整数，收到 {type(value).__name__}")
        if not 0 <= value <= 150:
            raise ValueError(f"年龄必须在 0-150 之间，收到 {value}")
        self._age = value

    @age.deleter
    def age(self) -> None:
        """deleter：删除时清理"""
        print(f"删除 {self.name} 的年龄记录")
        del self._age


p = Person("张三", 30)
p.age = 31       # 正常赋值
# p.age = -5     # 抛出 ValueError
# p.age = "abc"  # 抛出 TypeError
del p.age        # 调用 deleter


3. 延迟初始化模式 (Lazy Initialization)
-------------------------------------------
将昂贵的计算推迟到首次访问时执行，结果缓存起来。

class DatabaseConnection:
    """数据库连接——延迟初始化，首次访问时才创建连接"""
    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self._connection = None  # 初始为 None

    @property
    def connection(self):
        """首次访问时建立连接，后续复用"""
        if self._connection is None:
            # 模拟建立数据库连接（实际项目中用真正的数据库驱动）
            print(f"建立连接到 {self.host}:{self.port}...")
            self._connection = {"host": self.host, "port": self.port, "connected": True}
        return self._connection


db = DatabaseConnection("localhost", 5432)
# 此时尚未建立连接
conn1 = db.connection  # 输出：建立连接到 localhost:5432...
conn2 = db.connection  # 不再输出，直接返回缓存


4. functools.cached_property (Python 3.8+)
---------------------------------------------
标准库提供的缓存属性，自动处理缓存和失效。

from functools import cached_property

class Report:
    """报告——只计算一次的复杂统计"""
    def __init__(self, data: list):
        self.data = data

    @cached_property
    def summary(self) -> dict:
        """缓存计算结果，只计算一次"""
        print("计算汇总中...")  # 只打印一次
        return {
            "count": len(self.data),
            "sum": sum(self.data),
            "avg": sum(self.data) / len(self.data) if self.data else 0,
            "max": max(self.data) if self.data else None,
            "min": min(self.data) if self.data else None,
        }


r = Report([1, 2, 3, 4, 5])
print(r.summary)  # 计算并缓存
print(r.summary)  # 直接返回缓存，不重复计算


5. property 与 __getattr__ / __setattr__ 的关系
----------------------------------------------------
property 优先级高于 __getattr__，低于 __getattribute__。

class Demo:
    """演示属性访问优先级"""
    def __init__(self):
        self.x = 10  # 实例属性

    @property
    def y(self):
        """property 优先级高于 __getattr__"""
        return 42

    def __getattr__(self, name):
        """只有在常规查找失败时才调用"""
        return f"默认值:{name}"


d = Demo()
print(d.x)       # 10 —— 实例属性
print(d.y)       # 42 —— property，不触发 __getattr__
print(d.z)       # "默认值:z" —— 找不到时触发 __getattr__


6. property 与描述符协议
-----------------------------
@property 底层使用了描述符协议（__get__ / __set__ / __delete__）。

class PositiveNumber:
    """自定义描述符——类型检查和正数校验"""
    def __set_name__(self, owner, name):
        self._name = f"_{name}"  # 存储属性名

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self._name, 0)

    def __set__(self, obj, value):
        if not isinstance(value, (int, float)):
            raise TypeError("必须是数字")
        if value <= 0:
            raise ValueError("必须是正数")
        setattr(obj, self._name, value)


class Order:
    """使用描述符做字段校验"""
    quantity = PositiveNumber()  # 描述符自动处理校验
    price = PositiveNumber()

    def __init__(self, quantity: float, price: float):
        self.quantity = quantity
        self.price = price

    @property
    def total(self) -> float:
        return self.quantity * self.price


o = Order(10, 29.9)
print(o.total)  # 299.0


7. 只读属性与继承
--------------------
子类可以重写父类的 property。

class Base:
    """基类——只读属性"""
    @property
    def name(self) -> str:
        return "基类名称"

    @property
    def secret(self) -> str:
        """只读属性：只定义了 getter，没有 setter"""
        return "秘密值"


class Derived(Base):
    """子类重写父类 property"""
    @property
    def name(self) -> str:
        return "子类名称"  # 覆盖基类的 name


d = Derived()
print(d.name)    # "子类名称"
print(d.secret)  # "秘密值"——从父类继承的只读属性


总结：@property 提供了优雅的属性访问控制，包括计算属性、赋值校验、
延迟初始化、缓存等。结合描述符协议可以实现更强大的字段校验系统。

簇附蹦虐宜压悸勤乩昂淄伤谔樟肥

m.qbd.cccnt.cn/40848.Doc
m.qbd.cccnt.cn/42628.Doc
m.qbd.cccnt.cn/53791.Doc
m.qbd.cccnt.cn/37173.Doc
m.qbd.cccnt.cn/82662.Doc
m.qbd.cccnt.cn/40080.Doc
m.qbd.cccnt.cn/28280.Doc
m.qbs.cccnt.cn/00040.Doc
m.qbs.cccnt.cn/80204.Doc
m.qbs.cccnt.cn/26620.Doc
m.qbs.cccnt.cn/31119.Doc
m.qbs.cccnt.cn/71135.Doc
m.qbs.cccnt.cn/33911.Doc
m.qbs.cccnt.cn/48066.Doc
m.qbs.cccnt.cn/44284.Doc
m.qbs.cccnt.cn/42006.Doc
m.qbs.cccnt.cn/04448.Doc
m.qbs.cccnt.cn/57391.Doc
m.qbs.cccnt.cn/42402.Doc
m.qbs.cccnt.cn/04206.Doc
m.qbs.cccnt.cn/04224.Doc
m.qbs.cccnt.cn/93995.Doc
m.qbs.cccnt.cn/33913.Doc
m.qbs.cccnt.cn/84860.Doc
m.qbs.cccnt.cn/04426.Doc
m.qbs.cccnt.cn/55191.Doc
m.qbs.cccnt.cn/95571.Doc
m.qba.cccnt.cn/88686.Doc
m.qba.cccnt.cn/33159.Doc
m.qba.cccnt.cn/28884.Doc
m.qba.cccnt.cn/11351.Doc
m.qba.cccnt.cn/31557.Doc
m.qba.cccnt.cn/08622.Doc
m.qba.cccnt.cn/08282.Doc
m.qba.cccnt.cn/28824.Doc
m.qba.cccnt.cn/48486.Doc
m.qba.cccnt.cn/00444.Doc
m.qba.cccnt.cn/37915.Doc
m.qba.cccnt.cn/06420.Doc
m.qba.cccnt.cn/40022.Doc
m.qba.cccnt.cn/44024.Doc
m.qba.cccnt.cn/53737.Doc
m.qba.cccnt.cn/93139.Doc
m.qba.cccnt.cn/40604.Doc
m.qba.cccnt.cn/33319.Doc
m.qba.cccnt.cn/17593.Doc
m.qba.cccnt.cn/88644.Doc
m.qbp.cccnt.cn/20802.Doc
m.qbp.cccnt.cn/84622.Doc
m.qbp.cccnt.cn/77511.Doc
m.qbp.cccnt.cn/84666.Doc
m.qbp.cccnt.cn/48262.Doc
m.qbp.cccnt.cn/73155.Doc
m.qbp.cccnt.cn/02848.Doc
m.qbp.cccnt.cn/04020.Doc
m.qbp.cccnt.cn/04860.Doc
m.qbp.cccnt.cn/00428.Doc
m.qbp.cccnt.cn/99573.Doc
m.qbp.cccnt.cn/91159.Doc
m.qbp.cccnt.cn/46808.Doc
m.qbp.cccnt.cn/22466.Doc
m.qbp.cccnt.cn/24680.Doc
m.qbp.cccnt.cn/28880.Doc
m.qbp.cccnt.cn/24860.Doc
m.qbp.cccnt.cn/24880.Doc
m.qbp.cccnt.cn/95337.Doc
m.qbp.cccnt.cn/75197.Doc
m.qbo.cccnt.cn/40280.Doc
m.qbo.cccnt.cn/02262.Doc
m.qbo.cccnt.cn/80066.Doc
m.qbo.cccnt.cn/51919.Doc
m.qbo.cccnt.cn/60426.Doc
m.qbo.cccnt.cn/04408.Doc
m.qbo.cccnt.cn/84262.Doc
m.qbo.cccnt.cn/88688.Doc
m.qbo.cccnt.cn/11797.Doc
m.qbo.cccnt.cn/24284.Doc
m.qbo.cccnt.cn/62664.Doc
m.qbo.cccnt.cn/26228.Doc
m.qbo.cccnt.cn/26006.Doc
m.qbo.cccnt.cn/42442.Doc
m.qbo.cccnt.cn/68848.Doc
m.qbo.cccnt.cn/06220.Doc
m.qbo.cccnt.cn/00802.Doc
m.qbo.cccnt.cn/02088.Doc
m.qbo.cccnt.cn/04248.Doc
m.qbo.cccnt.cn/91351.Doc
m.qbi.cccnt.cn/66240.Doc
m.qbi.cccnt.cn/00880.Doc
m.qbi.cccnt.cn/86480.Doc
m.qbi.cccnt.cn/06862.Doc
m.qbi.cccnt.cn/68004.Doc
m.qbi.cccnt.cn/71357.Doc
m.qbi.cccnt.cn/46682.Doc
m.qbi.cccnt.cn/66826.Doc
m.qbi.cccnt.cn/08462.Doc
m.qbi.cccnt.cn/60408.Doc
m.qbi.cccnt.cn/15377.Doc
m.qbi.cccnt.cn/04006.Doc
m.qbi.cccnt.cn/80688.Doc
m.qbi.cccnt.cn/93173.Doc
m.qbi.cccnt.cn/84482.Doc
m.qbi.cccnt.cn/79753.Doc
m.qbi.cccnt.cn/86422.Doc
m.qbi.cccnt.cn/39951.Doc
m.qbi.cccnt.cn/20860.Doc
m.qbi.cccnt.cn/59335.Doc
m.qbu.cccnt.cn/02040.Doc
m.qbu.cccnt.cn/20086.Doc
m.qbu.cccnt.cn/84826.Doc
m.qbu.cccnt.cn/48882.Doc
m.qbu.cccnt.cn/64244.Doc
m.qbu.cccnt.cn/33313.Doc
m.qbu.cccnt.cn/24060.Doc
m.qbu.cccnt.cn/26880.Doc
m.qbu.cccnt.cn/40600.Doc
m.qbu.cccnt.cn/59775.Doc
m.qbu.cccnt.cn/66844.Doc
m.qbu.cccnt.cn/68608.Doc
m.qbu.cccnt.cn/62840.Doc
m.qbu.cccnt.cn/28044.Doc
m.qbu.cccnt.cn/64200.Doc
m.qbu.cccnt.cn/08848.Doc
m.qbu.cccnt.cn/04646.Doc
m.qbu.cccnt.cn/22404.Doc
m.qbu.cccnt.cn/48868.Doc
m.qbu.cccnt.cn/08004.Doc
m.qby.cccnt.cn/40068.Doc
m.qby.cccnt.cn/88024.Doc
m.qby.cccnt.cn/39955.Doc
m.qby.cccnt.cn/71593.Doc
m.qby.cccnt.cn/37575.Doc
m.qby.cccnt.cn/53513.Doc
m.qby.cccnt.cn/28404.Doc
m.qby.cccnt.cn/86226.Doc
m.qby.cccnt.cn/68622.Doc
m.qby.cccnt.cn/75319.Doc
m.qby.cccnt.cn/60406.Doc
m.qby.cccnt.cn/24888.Doc
m.qby.cccnt.cn/82608.Doc
m.qby.cccnt.cn/28480.Doc
m.qby.cccnt.cn/48808.Doc
m.qby.cccnt.cn/86428.Doc
m.qby.cccnt.cn/08606.Doc
m.qby.cccnt.cn/44840.Doc
m.qby.cccnt.cn/04860.Doc
m.qby.cccnt.cn/59973.Doc
m.qbt.cccnt.cn/84266.Doc
m.qbt.cccnt.cn/99919.Doc
m.qbt.cccnt.cn/08442.Doc
m.qbt.cccnt.cn/22264.Doc
m.qbt.cccnt.cn/11519.Doc
m.qbt.cccnt.cn/48084.Doc
m.qbt.cccnt.cn/88044.Doc
m.qbt.cccnt.cn/44026.Doc
m.qbt.cccnt.cn/82604.Doc
m.qbt.cccnt.cn/51191.Doc
m.qbt.cccnt.cn/86220.Doc
m.qbt.cccnt.cn/84208.Doc
m.qbt.cccnt.cn/40862.Doc
m.qbt.cccnt.cn/60440.Doc
m.qbt.cccnt.cn/46826.Doc
m.qbt.cccnt.cn/99139.Doc
m.qbt.cccnt.cn/31339.Doc
m.qbt.cccnt.cn/84666.Doc
m.qbt.cccnt.cn/08082.Doc
m.qbt.cccnt.cn/62026.Doc
m.qbr.mmmfb.cn/44440.Doc
m.qbr.mmmfb.cn/02448.Doc
m.qbr.mmmfb.cn/48460.Doc
m.qbr.mmmfb.cn/20440.Doc
m.qbr.mmmfb.cn/37515.Doc
m.qbr.mmmfb.cn/20466.Doc
m.qbr.mmmfb.cn/80286.Doc
m.qbr.mmmfb.cn/02068.Doc
m.qbr.mmmfb.cn/79791.Doc
m.qbr.mmmfb.cn/46486.Doc
m.qbr.mmmfb.cn/60848.Doc
m.qbr.mmmfb.cn/19977.Doc
m.qbr.mmmfb.cn/22424.Doc
m.qbr.mmmfb.cn/51555.Doc
m.qbr.mmmfb.cn/46644.Doc
m.qbr.mmmfb.cn/68460.Doc
m.qbr.mmmfb.cn/71533.Doc
m.qbr.mmmfb.cn/02042.Doc
m.qbr.mmmfb.cn/40266.Doc
m.qbr.mmmfb.cn/59531.Doc
m.qbe.mmmfb.cn/64622.Doc
m.qbe.mmmfb.cn/44042.Doc
m.qbe.mmmfb.cn/48084.Doc
m.qbe.mmmfb.cn/19337.Doc
m.qbe.mmmfb.cn/42282.Doc
m.qbe.mmmfb.cn/68022.Doc
m.qbe.mmmfb.cn/62808.Doc
m.qbe.mmmfb.cn/08622.Doc
m.qbe.mmmfb.cn/37519.Doc
m.qbe.mmmfb.cn/44442.Doc
m.qbe.mmmfb.cn/40668.Doc
m.qbe.mmmfb.cn/88884.Doc
m.qbe.mmmfb.cn/62468.Doc
m.qbe.mmmfb.cn/42620.Doc
m.qbe.mmmfb.cn/24806.Doc
m.qbe.mmmfb.cn/22466.Doc
m.qbe.mmmfb.cn/53739.Doc
m.qbe.mmmfb.cn/99391.Doc
m.qbe.mmmfb.cn/40620.Doc
m.qbe.mmmfb.cn/93599.Doc
m.qbw.mmmfb.cn/04822.Doc
m.qbw.mmmfb.cn/02066.Doc
m.qbw.mmmfb.cn/84824.Doc
m.qbw.mmmfb.cn/71333.Doc
m.qbw.mmmfb.cn/77773.Doc
m.qbw.mmmfb.cn/64228.Doc
m.qbw.mmmfb.cn/44406.Doc
m.qbw.mmmfb.cn/64420.Doc
m.qbw.mmmfb.cn/06806.Doc
m.qbw.mmmfb.cn/60080.Doc
m.qbw.mmmfb.cn/24486.Doc
m.qbw.mmmfb.cn/46288.Doc
m.qbw.mmmfb.cn/42000.Doc
m.qbw.mmmfb.cn/95979.Doc
m.qbw.mmmfb.cn/99975.Doc
m.qbw.mmmfb.cn/82828.Doc
m.qbw.mmmfb.cn/97953.Doc
m.qbw.mmmfb.cn/37719.Doc
m.qbw.mmmfb.cn/84866.Doc
m.qbw.mmmfb.cn/08468.Doc
m.qbq.mmmfb.cn/79357.Doc
m.qbq.mmmfb.cn/20208.Doc
m.qbq.mmmfb.cn/48668.Doc
m.qbq.mmmfb.cn/24448.Doc
m.qbq.mmmfb.cn/55311.Doc
m.qbq.mmmfb.cn/28806.Doc
m.qbq.mmmfb.cn/20066.Doc
m.qbq.mmmfb.cn/40800.Doc
m.qbq.mmmfb.cn/80846.Doc
m.qbq.mmmfb.cn/60822.Doc
m.qbq.mmmfb.cn/26842.Doc
m.qbq.mmmfb.cn/84860.Doc
m.qbq.mmmfb.cn/60846.Doc
m.qbq.mmmfb.cn/86082.Doc
m.qbq.mmmfb.cn/22006.Doc
m.qbq.mmmfb.cn/22064.Doc
m.qbq.mmmfb.cn/64464.Doc
m.qbq.mmmfb.cn/64000.Doc
m.qbq.mmmfb.cn/48628.Doc
m.qbq.mmmfb.cn/04440.Doc
m.qvm.mmmfb.cn/26226.Doc
m.qvm.mmmfb.cn/31199.Doc
m.qvm.mmmfb.cn/37355.Doc
m.qvm.mmmfb.cn/08062.Doc
m.qvm.mmmfb.cn/22062.Doc
m.qvm.mmmfb.cn/88244.Doc
m.qvm.mmmfb.cn/84248.Doc
m.qvm.mmmfb.cn/88262.Doc
m.qvm.mmmfb.cn/02664.Doc
m.qvm.mmmfb.cn/86426.Doc
m.qvm.mmmfb.cn/48606.Doc
m.qvm.mmmfb.cn/28048.Doc
m.qvm.mmmfb.cn/97173.Doc
m.qvm.mmmfb.cn/82440.Doc
m.qvm.mmmfb.cn/44084.Doc
m.qvm.mmmfb.cn/84200.Doc
m.qvm.mmmfb.cn/71375.Doc
m.qvm.mmmfb.cn/99371.Doc
m.qvm.mmmfb.cn/71711.Doc
m.qvm.mmmfb.cn/40202.Doc
m.qvn.mmmfb.cn/02404.Doc
m.qvn.mmmfb.cn/91395.Doc
m.qvn.mmmfb.cn/44248.Doc
m.qvn.mmmfb.cn/88080.Doc
m.qvn.mmmfb.cn/28480.Doc
m.qvn.mmmfb.cn/42868.Doc
m.qvn.mmmfb.cn/15395.Doc
m.qvn.mmmfb.cn/08822.Doc
m.qvn.mmmfb.cn/08060.Doc
m.qvn.mmmfb.cn/84242.Doc
m.qvn.mmmfb.cn/51117.Doc
m.qvn.mmmfb.cn/84682.Doc
m.qvn.mmmfb.cn/86648.Doc
m.qvn.mmmfb.cn/28284.Doc
m.qvn.mmmfb.cn/40468.Doc
m.qvn.mmmfb.cn/77355.Doc
m.qvn.mmmfb.cn/22822.Doc
m.qvn.mmmfb.cn/80802.Doc
m.qvn.mmmfb.cn/71137.Doc
m.qvn.mmmfb.cn/77159.Doc
m.qvb.mmmfb.cn/80860.Doc
m.qvb.mmmfb.cn/60688.Doc
m.qvb.mmmfb.cn/59131.Doc
m.qvb.mmmfb.cn/71713.Doc
m.qvb.mmmfb.cn/77935.Doc
m.qvb.mmmfb.cn/48426.Doc
m.qvb.mmmfb.cn/57719.Doc
m.qvb.mmmfb.cn/37131.Doc
m.qvb.mmmfb.cn/91335.Doc
m.qvb.mmmfb.cn/62686.Doc
m.qvb.mmmfb.cn/26846.Doc
m.qvb.mmmfb.cn/59555.Doc
m.qvb.mmmfb.cn/64286.Doc
m.qvb.mmmfb.cn/13917.Doc
m.qvb.mmmfb.cn/88842.Doc
m.qvb.mmmfb.cn/88260.Doc
m.qvb.mmmfb.cn/08888.Doc
m.qvb.mmmfb.cn/02260.Doc
m.qvb.mmmfb.cn/62848.Doc
m.qvb.mmmfb.cn/77991.Doc
m.qvv.mmmfb.cn/68422.Doc
m.qvv.mmmfb.cn/20664.Doc
m.qvv.mmmfb.cn/64482.Doc
m.qvv.mmmfb.cn/08082.Doc
m.qvv.mmmfb.cn/24004.Doc
m.qvv.mmmfb.cn/20026.Doc
m.qvv.mmmfb.cn/84884.Doc
m.qvv.mmmfb.cn/60660.Doc
m.qvv.mmmfb.cn/82844.Doc
m.qvv.mmmfb.cn/22440.Doc
m.qvv.mmmfb.cn/39171.Doc
m.qvv.mmmfb.cn/04468.Doc
m.qvv.mmmfb.cn/20208.Doc
m.qvv.mmmfb.cn/75973.Doc
m.qvv.mmmfb.cn/40660.Doc
m.qvv.mmmfb.cn/53953.Doc
m.qvv.mmmfb.cn/33357.Doc
m.qvv.mmmfb.cn/82868.Doc
m.qvv.mmmfb.cn/40848.Doc
m.qvv.mmmfb.cn/11737.Doc
m.qvc.mmmfb.cn/19775.Doc
m.qvc.mmmfb.cn/24284.Doc
m.qvc.mmmfb.cn/02800.Doc
m.qvc.mmmfb.cn/26086.Doc
m.qvc.mmmfb.cn/17399.Doc
m.qvc.mmmfb.cn/44206.Doc
m.qvc.mmmfb.cn/60808.Doc
m.qvc.mmmfb.cn/80282.Doc
m.qvc.mmmfb.cn/26046.Doc
m.qvc.mmmfb.cn/04288.Doc
m.qvc.mmmfb.cn/26882.Doc
m.qvc.mmmfb.cn/00220.Doc
m.qvc.mmmfb.cn/46240.Doc
m.qvc.mmmfb.cn/44222.Doc
m.qvc.mmmfb.cn/06226.Doc
m.qvc.mmmfb.cn/44000.Doc
m.qvc.mmmfb.cn/53577.Doc
m.qvc.mmmfb.cn/80806.Doc
m.qvc.mmmfb.cn/48060.Doc
m.qvc.mmmfb.cn/82664.Doc
m.qvx.mmmfb.cn/06064.Doc
m.qvx.mmmfb.cn/79775.Doc
m.qvx.mmmfb.cn/64606.Doc
m.qvx.mmmfb.cn/79133.Doc
m.qvx.mmmfb.cn/22660.Doc
m.qvx.mmmfb.cn/37337.Doc
m.qvx.mmmfb.cn/53513.Doc
m.qvx.mmmfb.cn/06204.Doc
m.qvx.mmmfb.cn/44424.Doc
m.qvx.mmmfb.cn/06646.Doc
m.qvx.mmmfb.cn/53779.Doc
m.qvx.mmmfb.cn/88848.Doc
m.qvx.mmmfb.cn/00820.Doc
m.qvx.mmmfb.cn/17757.Doc
m.qvx.mmmfb.cn/82000.Doc
m.qvx.mmmfb.cn/04824.Doc
m.qvx.mmmfb.cn/37793.Doc
m.qvx.mmmfb.cn/24066.Doc
m.qvx.mmmfb.cn/35997.Doc
m.qvx.mmmfb.cn/37513.Doc
m.qvz.mmmfb.cn/84442.Doc
m.qvz.mmmfb.cn/40242.Doc
m.qvz.mmmfb.cn/00868.Doc
m.qvz.mmmfb.cn/28080.Doc
m.qvz.mmmfb.cn/97917.Doc
m.qvz.mmmfb.cn/39775.Doc
m.qvz.mmmfb.cn/06204.Doc
m.qvz.mmmfb.cn/04602.Doc
m.qvz.mmmfb.cn/28600.Doc
m.qvz.mmmfb.cn/71373.Doc
m.qvz.mmmfb.cn/15995.Doc
m.qvz.mmmfb.cn/22066.Doc
m.qvz.mmmfb.cn/51715.Doc
m.qvz.mmmfb.cn/24246.Doc
m.qvz.mmmfb.cn/00420.Doc
m.qvz.mmmfb.cn/15137.Doc
m.qvz.mmmfb.cn/37737.Doc
m.qvz.mmmfb.cn/55335.Doc
m.qvz.mmmfb.cn/40848.Doc
m.qvz.mmmfb.cn/84688.Doc
m.qvl.mmmfb.cn/53775.Doc
m.qvl.mmmfb.cn/46006.Doc
m.qvl.mmmfb.cn/88802.Doc
m.qvl.mmmfb.cn/40206.Doc
m.qvl.mmmfb.cn/60666.Doc
m.qvl.mmmfb.cn/31931.Doc
m.qvl.mmmfb.cn/28860.Doc
m.qvl.mmmfb.cn/48200.Doc
