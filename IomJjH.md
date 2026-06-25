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

dzh.baoquan026.cn/64864.Doc
dzh.baoquan026.cn/24242.Doc
dzh.baoquan026.cn/88406.Doc
dzh.baoquan026.cn/26408.Doc
dzh.baoquan026.cn/62886.Doc
dzh.baoquan026.cn/84688.Doc
dzh.baoquan026.cn/28064.Doc
dzh.baoquan026.cn/64024.Doc
dzh.baoquan026.cn/08248.Doc
dzh.baoquan026.cn/02608.Doc
dzg.baoquan026.cn/22466.Doc
dzg.baoquan026.cn/02264.Doc
dzg.baoquan026.cn/08682.Doc
dzg.baoquan026.cn/40246.Doc
dzg.baoquan026.cn/20006.Doc
dzg.baoquan026.cn/84660.Doc
dzg.baoquan026.cn/00402.Doc
dzg.baoquan026.cn/04228.Doc
dzg.baoquan026.cn/84006.Doc
dzg.baoquan026.cn/40062.Doc
dzf.baoquan026.cn/46886.Doc
dzf.baoquan026.cn/46686.Doc
dzf.baoquan026.cn/46804.Doc
dzf.baoquan026.cn/46864.Doc
dzf.baoquan026.cn/20620.Doc
dzf.baoquan026.cn/20442.Doc
dzf.baoquan026.cn/06206.Doc
dzf.baoquan026.cn/20480.Doc
dzf.baoquan026.cn/08488.Doc
dzf.baoquan026.cn/62820.Doc
dzd.baoquan026.cn/40440.Doc
dzd.baoquan026.cn/00044.Doc
dzd.baoquan026.cn/86626.Doc
dzd.baoquan026.cn/44002.Doc
dzd.baoquan026.cn/86882.Doc
dzd.baoquan026.cn/62600.Doc
dzd.baoquan026.cn/60206.Doc
dzd.baoquan026.cn/02204.Doc
dzd.baoquan026.cn/20662.Doc
dzd.baoquan026.cn/88208.Doc
dzs.baoquan026.cn/66808.Doc
dzs.baoquan026.cn/66626.Doc
dzs.baoquan026.cn/66242.Doc
dzs.baoquan026.cn/06648.Doc
dzs.baoquan026.cn/24860.Doc
dzs.baoquan026.cn/60480.Doc
dzs.baoquan026.cn/04266.Doc
dzs.baoquan026.cn/40406.Doc
dzs.baoquan026.cn/28606.Doc
dzs.baoquan026.cn/60002.Doc
dza.baoquan026.cn/04220.Doc
dza.baoquan026.cn/02606.Doc
dza.baoquan026.cn/80844.Doc
dza.baoquan026.cn/84800.Doc
dza.baoquan026.cn/48468.Doc
dza.baoquan026.cn/64022.Doc
dza.baoquan026.cn/44008.Doc
dza.baoquan026.cn/42622.Doc
dza.baoquan026.cn/66002.Doc
dza.baoquan026.cn/04802.Doc
dzp.baoquan026.cn/46806.Doc
dzp.baoquan026.cn/46428.Doc
dzp.baoquan026.cn/42484.Doc
dzp.baoquan026.cn/66480.Doc
dzp.baoquan026.cn/24082.Doc
dzp.baoquan026.cn/60680.Doc
dzp.baoquan026.cn/60008.Doc
dzp.baoquan026.cn/44246.Doc
dzp.baoquan026.cn/28040.Doc
dzp.baoquan026.cn/82624.Doc
dzo.baoquan026.cn/82624.Doc
dzo.baoquan026.cn/66484.Doc
dzo.baoquan026.cn/84286.Doc
dzo.baoquan026.cn/88024.Doc
dzo.baoquan026.cn/66426.Doc
dzo.baoquan026.cn/60844.Doc
dzo.baoquan026.cn/04808.Doc
dzo.baoquan026.cn/60066.Doc
dzo.baoquan026.cn/20282.Doc
dzo.baoquan026.cn/48444.Doc
dzi.baoquan026.cn/84008.Doc
dzi.baoquan026.cn/68606.Doc
dzi.baoquan026.cn/24486.Doc
dzi.baoquan026.cn/08684.Doc
dzi.baoquan026.cn/86284.Doc
dzi.baoquan026.cn/86840.Doc
dzi.baoquan026.cn/08028.Doc
dzi.baoquan026.cn/80420.Doc
dzi.baoquan026.cn/60888.Doc
dzi.baoquan026.cn/68848.Doc
dzu.baoquan026.cn/08606.Doc
dzu.baoquan026.cn/08288.Doc
dzu.baoquan026.cn/08080.Doc
dzu.baoquan026.cn/00402.Doc
dzu.baoquan026.cn/40222.Doc
dzu.baoquan026.cn/82862.Doc
dzu.baoquan026.cn/62002.Doc
dzu.baoquan026.cn/42882.Doc
dzu.baoquan026.cn/82400.Doc
dzu.baoquan026.cn/44042.Doc
dzy.baoquan026.cn/24822.Doc
dzy.baoquan026.cn/80826.Doc
dzy.baoquan026.cn/02662.Doc
dzy.baoquan026.cn/82842.Doc
dzy.baoquan026.cn/24204.Doc
dzy.baoquan026.cn/40404.Doc
dzy.baoquan026.cn/02244.Doc
dzy.baoquan026.cn/24222.Doc
dzy.baoquan026.cn/22288.Doc
dzy.baoquan026.cn/66244.Doc
dzt.baoquan026.cn/86266.Doc
dzt.baoquan026.cn/48862.Doc
dzt.baoquan026.cn/22424.Doc
dzt.baoquan026.cn/64884.Doc
dzt.baoquan026.cn/44066.Doc
dzt.baoquan026.cn/88264.Doc
dzt.baoquan026.cn/88860.Doc
dzt.baoquan026.cn/48066.Doc
dzt.baoquan026.cn/48422.Doc
dzt.baoquan026.cn/06282.Doc
dzr.baoquan026.cn/04240.Doc
dzr.baoquan026.cn/62842.Doc
dzr.baoquan026.cn/62620.Doc
dzr.baoquan026.cn/42060.Doc
dzr.baoquan026.cn/66002.Doc
dzr.baoquan026.cn/66286.Doc
dzr.baoquan026.cn/62602.Doc
dzr.baoquan026.cn/44486.Doc
dzr.baoquan026.cn/86060.Doc
dzr.baoquan026.cn/00842.Doc
dze.baoquan026.cn/82422.Doc
dze.baoquan026.cn/02086.Doc
dze.baoquan026.cn/20846.Doc
dze.baoquan026.cn/24202.Doc
dze.baoquan026.cn/64020.Doc
dze.baoquan026.cn/64680.Doc
dze.baoquan026.cn/68008.Doc
dze.baoquan026.cn/82228.Doc
dze.baoquan026.cn/26460.Doc
dze.baoquan026.cn/68882.Doc
dzw.baoquan026.cn/88046.Doc
dzw.baoquan026.cn/86204.Doc
dzw.baoquan026.cn/40068.Doc
dzw.baoquan026.cn/64848.Doc
dzw.baoquan026.cn/68802.Doc
dzw.baoquan026.cn/42688.Doc
dzw.baoquan026.cn/04880.Doc
dzw.baoquan026.cn/80284.Doc
dzw.baoquan026.cn/06084.Doc
dzw.baoquan026.cn/26666.Doc
dzq.baoquan026.cn/26080.Doc
dzq.baoquan026.cn/68420.Doc
dzq.baoquan026.cn/46060.Doc
dzq.baoquan026.cn/20666.Doc
dzq.baoquan026.cn/62022.Doc
dzq.baoquan026.cn/42020.Doc
dzq.baoquan026.cn/82086.Doc
dzq.baoquan026.cn/48444.Doc
dzq.baoquan026.cn/68444.Doc
dzq.baoquan026.cn/86662.Doc
dlm.baoquan026.cn/60448.Doc
dlm.baoquan026.cn/82662.Doc
dlm.baoquan026.cn/40644.Doc
dlm.baoquan026.cn/84200.Doc
dlm.baoquan026.cn/60080.Doc
dlm.baoquan026.cn/66486.Doc
dlm.baoquan026.cn/80688.Doc
dlm.baoquan026.cn/86640.Doc
dlm.baoquan026.cn/66806.Doc
dlm.baoquan026.cn/80282.Doc
dln.baoquan026.cn/04026.Doc
dln.baoquan026.cn/20426.Doc
dln.baoquan026.cn/48404.Doc
dln.baoquan026.cn/06266.Doc
dln.baoquan026.cn/28486.Doc
dln.baoquan026.cn/66020.Doc
dln.baoquan026.cn/26208.Doc
dln.baoquan026.cn/00260.Doc
dln.baoquan026.cn/02262.Doc
dln.baoquan026.cn/24868.Doc
dlb.baoquan026.cn/62624.Doc
dlb.baoquan026.cn/64464.Doc
dlb.baoquan026.cn/68040.Doc
dlb.baoquan026.cn/60026.Doc
dlb.baoquan026.cn/04008.Doc
dlb.baoquan026.cn/40064.Doc
dlb.baoquan026.cn/84048.Doc
dlb.baoquan026.cn/28660.Doc
dlb.baoquan026.cn/60606.Doc
dlb.baoquan026.cn/80668.Doc
dlv.baoquan026.cn/73133.Doc
dlv.baoquan026.cn/66060.Doc
dlv.baoquan026.cn/84626.Doc
dlv.baoquan026.cn/04080.Doc
dlv.baoquan026.cn/26048.Doc
dlv.baoquan026.cn/04604.Doc
dlv.baoquan026.cn/57759.Doc
dlv.baoquan026.cn/62028.Doc
dlv.baoquan026.cn/48806.Doc
dlv.baoquan026.cn/00228.Doc
dlc.baoquan026.cn/42486.Doc
dlc.baoquan026.cn/75757.Doc
dlc.baoquan026.cn/48642.Doc
dlc.baoquan026.cn/84400.Doc
dlc.baoquan026.cn/80682.Doc
dlc.baoquan026.cn/20886.Doc
dlc.baoquan026.cn/04202.Doc
dlc.baoquan026.cn/66628.Doc
dlc.baoquan026.cn/08646.Doc
dlc.baoquan026.cn/42284.Doc
dlx.baoquan026.cn/86464.Doc
dlx.baoquan026.cn/95991.Doc
dlx.baoquan026.cn/40462.Doc
dlx.baoquan026.cn/26200.Doc
dlx.baoquan026.cn/08000.Doc
dlx.baoquan026.cn/46044.Doc
dlx.baoquan026.cn/60624.Doc
dlx.baoquan026.cn/28288.Doc
dlx.baoquan026.cn/06222.Doc
dlx.baoquan026.cn/02060.Doc
dlz.baoquan026.cn/02448.Doc
dlz.baoquan026.cn/64048.Doc
dlz.baoquan026.cn/00648.Doc
dlz.baoquan026.cn/71371.Doc
dlz.baoquan026.cn/60826.Doc
dlz.baoquan026.cn/26268.Doc
dlz.baoquan026.cn/64800.Doc
dlz.baoquan026.cn/62604.Doc
dlz.baoquan026.cn/22204.Doc
dlz.baoquan026.cn/64040.Doc
dll.baoquan026.cn/60646.Doc
dll.baoquan026.cn/68444.Doc
dll.baoquan026.cn/66448.Doc
dll.baoquan026.cn/31917.Doc
dll.baoquan026.cn/48486.Doc
dll.baoquan026.cn/44402.Doc
dll.baoquan026.cn/40424.Doc
dll.baoquan026.cn/06204.Doc
dll.baoquan026.cn/06862.Doc
dll.baoquan026.cn/62020.Doc
dlk.baoquan026.cn/68682.Doc
dlk.baoquan026.cn/22648.Doc
dlk.baoquan026.cn/08286.Doc
dlk.baoquan026.cn/46620.Doc
dlk.baoquan026.cn/40220.Doc
dlk.baoquan026.cn/20882.Doc
dlk.baoquan026.cn/77959.Doc
dlk.baoquan026.cn/84868.Doc
dlk.baoquan026.cn/26622.Doc
dlk.baoquan026.cn/02206.Doc
dlj.baoquan026.cn/99977.Doc
dlj.baoquan026.cn/84680.Doc
dlj.baoquan026.cn/20862.Doc
dlj.baoquan026.cn/66088.Doc
dlj.baoquan026.cn/60080.Doc
dlj.baoquan026.cn/80682.Doc
dlj.baoquan026.cn/44682.Doc
dlj.baoquan026.cn/24066.Doc
dlj.baoquan026.cn/46024.Doc
dlj.baoquan026.cn/24684.Doc
dlh.baoquan026.cn/06884.Doc
dlh.baoquan026.cn/28686.Doc
dlh.baoquan026.cn/11311.Doc
dlh.baoquan026.cn/02226.Doc
dlh.baoquan026.cn/42046.Doc
dlh.baoquan026.cn/42044.Doc
dlh.baoquan026.cn/48228.Doc
dlh.baoquan026.cn/22480.Doc
dlh.baoquan026.cn/48480.Doc
dlh.baoquan026.cn/44224.Doc
dlg.baoquan026.cn/00482.Doc
dlg.baoquan026.cn/22664.Doc
dlg.baoquan026.cn/66460.Doc
dlg.baoquan026.cn/22842.Doc
dlg.baoquan026.cn/68886.Doc
dlg.baoquan026.cn/42028.Doc
dlg.baoquan026.cn/08686.Doc
dlg.baoquan026.cn/53133.Doc
dlg.baoquan026.cn/86002.Doc
dlg.baoquan026.cn/68606.Doc
dlf.baoquan026.cn/68088.Doc
dlf.baoquan026.cn/99359.Doc
dlf.baoquan026.cn/06862.Doc
dlf.baoquan026.cn/22402.Doc
dlf.baoquan026.cn/68642.Doc
dlf.baoquan026.cn/04042.Doc
dlf.baoquan026.cn/82688.Doc
dlf.baoquan026.cn/04460.Doc
dlf.baoquan026.cn/28262.Doc
dlf.baoquan026.cn/08062.Doc
dld.baoquan026.cn/37975.Doc
dld.baoquan026.cn/46268.Doc
dld.baoquan026.cn/48622.Doc
dld.baoquan026.cn/46004.Doc
dld.baoquan026.cn/62626.Doc
dld.baoquan026.cn/39537.Doc
dld.baoquan026.cn/42802.Doc
dld.baoquan026.cn/44062.Doc
dld.baoquan026.cn/24002.Doc
dld.baoquan026.cn/40888.Doc
dls.baoquan026.cn/20620.Doc
dls.baoquan026.cn/11319.Doc
dls.baoquan026.cn/20402.Doc
dls.baoquan026.cn/08468.Doc
dls.baoquan026.cn/02682.Doc
dls.baoquan026.cn/62440.Doc
dls.baoquan026.cn/42866.Doc
dls.baoquan026.cn/40620.Doc
dls.baoquan026.cn/42826.Doc
dls.baoquan026.cn/00466.Doc
dla.baoquan026.cn/06488.Doc
dla.baoquan026.cn/80648.Doc
dla.baoquan026.cn/26404.Doc
dla.baoquan026.cn/08002.Doc
dla.baoquan026.cn/62008.Doc
dla.baoquan026.cn/80886.Doc
dla.baoquan026.cn/66804.Doc
dla.baoquan026.cn/64648.Doc
dla.baoquan026.cn/22628.Doc
dla.baoquan026.cn/64448.Doc
dlp.baoquan026.cn/42086.Doc
dlp.baoquan026.cn/31377.Doc
dlp.baoquan026.cn/19933.Doc
dlp.baoquan026.cn/20648.Doc
dlp.baoquan026.cn/20204.Doc
dlp.baoquan026.cn/04426.Doc
dlp.baoquan026.cn/84244.Doc
dlp.baoquan026.cn/44680.Doc
dlp.baoquan026.cn/22048.Doc
dlp.baoquan026.cn/40466.Doc
dlo.baoquan026.cn/06668.Doc
dlo.baoquan026.cn/66800.Doc
dlo.baoquan026.cn/00044.Doc
dlo.baoquan026.cn/08064.Doc
dlo.baoquan026.cn/82644.Doc
dlo.baoquan026.cn/46400.Doc
dlo.baoquan026.cn/40626.Doc
dlo.baoquan026.cn/48660.Doc
dlo.baoquan026.cn/62202.Doc
dlo.baoquan026.cn/46840.Doc
dli.baoquan026.cn/35755.Doc
dli.baoquan026.cn/84066.Doc
dli.baoquan026.cn/44206.Doc
dli.baoquan026.cn/08680.Doc
dli.baoquan026.cn/77137.Doc
dli.baoquan026.cn/04666.Doc
dli.baoquan026.cn/26884.Doc
dli.baoquan026.cn/26286.Doc
dli.baoquan026.cn/62482.Doc
dli.baoquan026.cn/88000.Doc
dlu.baoquan026.cn/64608.Doc
dlu.baoquan026.cn/44426.Doc
dlu.baoquan026.cn/22046.Doc
dlu.baoquan026.cn/59597.Doc
dlu.baoquan026.cn/86422.Doc
dlu.baoquan026.cn/44464.Doc
dlu.baoquan026.cn/24806.Doc
dlu.baoquan026.cn/02846.Doc
dlu.baoquan026.cn/24002.Doc
dlu.baoquan026.cn/28860.Doc
dly.baoquan026.cn/04244.Doc
dly.baoquan026.cn/46444.Doc
dly.baoquan026.cn/59373.Doc
dly.baoquan026.cn/77335.Doc
dly.baoquan026.cn/02626.Doc
dly.baoquan026.cn/91995.Doc
dly.baoquan026.cn/55511.Doc
dly.baoquan026.cn/02480.Doc
dly.baoquan026.cn/20866.Doc
dly.baoquan026.cn/48022.Doc
dlt.baoquan026.cn/24206.Doc
dlt.baoquan026.cn/02088.Doc
dlt.baoquan026.cn/04006.Doc
dlt.baoquan026.cn/64088.Doc
dlt.baoquan026.cn/40442.Doc
dlt.baoquan026.cn/80648.Doc
dlt.baoquan026.cn/60840.Doc
dlt.baoquan026.cn/42006.Doc
dlt.baoquan026.cn/60460.Doc
dlt.baoquan026.cn/84468.Doc
dlr.baoquan026.cn/68002.Doc
dlr.baoquan026.cn/15951.Doc
dlr.baoquan026.cn/82824.Doc
dlr.baoquan026.cn/44846.Doc
dlr.baoquan026.cn/00288.Doc
dlr.baoquan026.cn/35511.Doc
dlr.baoquan026.cn/68420.Doc
dlr.baoquan026.cn/62408.Doc
dlr.baoquan026.cn/24684.Doc
dlr.baoquan026.cn/06284.Doc
dle.baoquan026.cn/26286.Doc
dle.baoquan026.cn/02080.Doc
dle.baoquan026.cn/00062.Doc
dle.baoquan026.cn/60266.Doc
dle.baoquan026.cn/66802.Doc
dle.baoquan026.cn/64804.Doc
dle.baoquan026.cn/48288.Doc
dle.baoquan026.cn/80044.Doc
dle.baoquan026.cn/08248.Doc
dle.baoquan026.cn/24406.Doc
dlw.baoquan026.cn/40802.Doc
dlw.baoquan026.cn/46840.Doc
dlw.baoquan026.cn/40244.Doc
dlw.baoquan026.cn/48822.Doc
dlw.baoquan026.cn/20882.Doc
dlw.baoquan026.cn/06004.Doc
dlw.baoquan026.cn/42220.Doc
dlw.baoquan026.cn/28262.Doc
dlw.baoquan026.cn/82668.Doc
dlw.baoquan026.cn/64486.Doc
dlq.baoquan026.cn/73351.Doc
dlq.baoquan026.cn/62004.Doc
dlq.baoquan026.cn/00208.Doc
dlq.baoquan026.cn/88400.Doc
dlq.baoquan026.cn/68682.Doc
dlq.baoquan026.cn/62888.Doc
dlq.baoquan026.cn/46280.Doc
dlq.baoquan026.cn/28202.Doc
dlq.baoquan026.cn/68628.Doc
dlq.baoquan026.cn/08622.Doc
dkm.baoquan026.cn/80882.Doc
dkm.baoquan026.cn/95777.Doc
dkm.baoquan026.cn/04866.Doc
dkm.baoquan026.cn/55357.Doc
dkm.baoquan026.cn/64824.Doc
dkm.baoquan026.cn/08220.Doc
dkm.baoquan026.cn/04004.Doc
dkm.baoquan026.cn/26868.Doc
dkm.baoquan026.cn/66608.Doc
dkm.baoquan026.cn/00420.Doc
dkn.baoquan026.cn/02600.Doc
dkn.baoquan026.cn/13315.Doc
dkn.baoquan026.cn/00264.Doc
dkn.baoquan026.cn/46606.Doc
dkn.baoquan026.cn/84082.Doc
dkn.baoquan026.cn/95111.Doc
dkn.baoquan026.cn/06686.Doc
dkn.baoquan026.cn/88000.Doc
dkn.baoquan026.cn/15977.Doc
dkn.baoquan026.cn/04602.Doc
dkb.baoquan026.cn/48440.Doc
dkb.baoquan026.cn/77117.Doc
dkb.baoquan026.cn/00480.Doc
dkb.baoquan026.cn/86066.Doc
dkb.baoquan026.cn/82648.Doc
dkb.baoquan026.cn/28200.Doc
dkb.baoquan026.cn/68848.Doc
dkb.baoquan026.cn/40824.Doc
dkb.baoquan026.cn/88422.Doc
dkb.baoquan026.cn/44620.Doc
dkv.baoquan026.cn/84660.Doc
dkv.baoquan026.cn/64028.Doc
dkv.baoquan026.cn/84800.Doc
dkv.baoquan026.cn/64000.Doc
dkv.baoquan026.cn/46686.Doc
dkv.baoquan026.cn/04866.Doc
dkv.baoquan026.cn/84006.Doc
dkv.baoquan026.cn/60660.Doc
dkv.baoquan026.cn/84282.Doc
dkv.baoquan026.cn/84288.Doc
dkc.baoquan026.cn/80800.Doc
dkc.baoquan026.cn/28040.Doc
dkc.baoquan026.cn/44482.Doc
dkc.baoquan026.cn/86428.Doc
dkc.baoquan026.cn/28086.Doc
dkc.baoquan026.cn/46406.Doc
dkc.baoquan026.cn/46042.Doc
dkc.baoquan026.cn/08282.Doc
dkc.baoquan026.cn/68222.Doc
dkc.baoquan026.cn/31335.Doc
dkx.baoquan026.cn/48428.Doc
dkx.baoquan026.cn/28246.Doc
dkx.baoquan026.cn/48468.Doc
dkx.baoquan026.cn/84428.Doc
dkx.baoquan026.cn/82866.Doc
dkx.baoquan026.cn/75595.Doc
dkx.baoquan026.cn/53959.Doc
dkx.baoquan026.cn/55137.Doc
dkx.baoquan026.cn/57937.Doc
dkx.baoquan026.cn/15597.Doc
dkz.baoquan026.cn/80240.Doc
dkz.baoquan026.cn/37999.Doc
dkz.baoquan026.cn/00808.Doc
dkz.baoquan026.cn/79995.Doc
dkz.baoquan026.cn/79355.Doc
dkz.baoquan026.cn/59777.Doc
dkz.baoquan026.cn/53313.Doc
dkz.baoquan026.cn/91917.Doc
dkz.baoquan026.cn/71559.Doc
dkz.baoquan026.cn/59773.Doc
