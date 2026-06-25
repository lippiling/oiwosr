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

qbm.www669hq.cn/24260.Doc
qbm.www669hq.cn/57975.Doc
qbm.www669hq.cn/66448.Doc
qbm.www669hq.cn/24088.Doc
qbm.www669hq.cn/48864.Doc
qbm.www669hq.cn/82488.Doc
qbm.www669hq.cn/42066.Doc
qbm.www669hq.cn/44860.Doc
qbm.www669hq.cn/64484.Doc
qbm.www669hq.cn/86484.Doc
qbn.www669hq.cn/48668.Doc
qbn.www669hq.cn/26848.Doc
qbn.www669hq.cn/64642.Doc
qbn.www669hq.cn/68844.Doc
qbn.www669hq.cn/02420.Doc
qbn.www669hq.cn/48462.Doc
qbn.www669hq.cn/80860.Doc
qbn.www669hq.cn/64242.Doc
qbn.www669hq.cn/60044.Doc
qbn.www669hq.cn/88422.Doc
qbb.www669hq.cn/48688.Doc
qbb.www669hq.cn/20482.Doc
qbb.www669hq.cn/26660.Doc
qbb.www669hq.cn/06288.Doc
qbb.www669hq.cn/64428.Doc
qbb.www669hq.cn/71515.Doc
qbb.www669hq.cn/68686.Doc
qbb.www669hq.cn/62424.Doc
qbb.www669hq.cn/48628.Doc
qbb.www669hq.cn/22844.Doc
qbv.www669hq.cn/53553.Doc
qbv.www669hq.cn/48022.Doc
qbv.www669hq.cn/82066.Doc
qbv.www669hq.cn/62808.Doc
qbv.www669hq.cn/64866.Doc
qbv.www669hq.cn/22204.Doc
qbv.www669hq.cn/84442.Doc
qbv.www669hq.cn/28088.Doc
qbv.www669hq.cn/88804.Doc
qbv.www669hq.cn/00840.Doc
qbc.www669hq.cn/40646.Doc
qbc.www669hq.cn/24644.Doc
qbc.www669hq.cn/13973.Doc
qbc.www669hq.cn/08282.Doc
qbc.www669hq.cn/42482.Doc
qbc.www669hq.cn/04682.Doc
qbc.www669hq.cn/20604.Doc
qbc.www669hq.cn/08628.Doc
qbc.www669hq.cn/28208.Doc
qbc.www669hq.cn/82660.Doc
qbx.www669hq.cn/40288.Doc
qbx.www669hq.cn/28864.Doc
qbx.www669hq.cn/06448.Doc
qbx.www669hq.cn/48240.Doc
qbx.www669hq.cn/17171.Doc
qbx.www669hq.cn/33133.Doc
qbx.www669hq.cn/46660.Doc
qbx.www669hq.cn/04626.Doc
qbx.www669hq.cn/20228.Doc
qbx.www669hq.cn/28000.Doc
qbz.www669hq.cn/46648.Doc
qbz.www669hq.cn/64084.Doc
qbz.www669hq.cn/13733.Doc
qbz.www669hq.cn/20242.Doc
qbz.www669hq.cn/46226.Doc
qbz.www669hq.cn/44666.Doc
qbz.www669hq.cn/22282.Doc
qbz.www669hq.cn/26066.Doc
qbz.www669hq.cn/84620.Doc
qbz.www669hq.cn/00802.Doc
qbl.www669hq.cn/42846.Doc
qbl.www669hq.cn/22200.Doc
qbl.www669hq.cn/48062.Doc
qbl.www669hq.cn/66000.Doc
qbl.www669hq.cn/40048.Doc
qbl.www669hq.cn/06422.Doc
qbl.www669hq.cn/00026.Doc
qbl.www669hq.cn/08460.Doc
qbl.www669hq.cn/26862.Doc
qbl.www669hq.cn/60860.Doc
qbk.www669hq.cn/40088.Doc
qbk.www669hq.cn/40408.Doc
qbk.www669hq.cn/44626.Doc
qbk.www669hq.cn/00486.Doc
qbk.www669hq.cn/80662.Doc
qbk.www669hq.cn/00248.Doc
qbk.www669hq.cn/42044.Doc
qbk.www669hq.cn/35775.Doc
qbk.www669hq.cn/73971.Doc
qbk.www669hq.cn/86640.Doc
qbj.www669hq.cn/68640.Doc
qbj.www669hq.cn/62828.Doc
qbj.www669hq.cn/22644.Doc
qbj.www669hq.cn/08448.Doc
qbj.www669hq.cn/40482.Doc
qbj.www669hq.cn/24608.Doc
qbj.www669hq.cn/93991.Doc
qbj.www669hq.cn/80446.Doc
qbj.www669hq.cn/08206.Doc
qbj.www669hq.cn/64824.Doc
qbh.www669hq.cn/08286.Doc
qbh.www669hq.cn/68042.Doc
qbh.www669hq.cn/86088.Doc
qbh.www669hq.cn/06800.Doc
qbh.www669hq.cn/02026.Doc
qbh.www669hq.cn/02022.Doc
qbh.www669hq.cn/40220.Doc
qbh.www669hq.cn/60480.Doc
qbh.www669hq.cn/02066.Doc
qbh.www669hq.cn/40628.Doc
qbg.www669hq.cn/99733.Doc
qbg.www669hq.cn/79133.Doc
qbg.www669hq.cn/86060.Doc
qbg.www669hq.cn/08666.Doc
qbg.www669hq.cn/84084.Doc
qbg.www669hq.cn/71771.Doc
qbg.www669hq.cn/00828.Doc
qbg.www669hq.cn/68868.Doc
qbg.www669hq.cn/26666.Doc
qbg.www669hq.cn/42600.Doc
qbf.www669hq.cn/42260.Doc
qbf.www669hq.cn/24002.Doc
qbf.www669hq.cn/82664.Doc
qbf.www669hq.cn/26828.Doc
qbf.www669hq.cn/00206.Doc
qbf.www669hq.cn/42282.Doc
qbf.www669hq.cn/00664.Doc
qbf.www669hq.cn/84880.Doc
qbf.www669hq.cn/86840.Doc
qbf.www669hq.cn/82840.Doc
qbd.www669hq.cn/88020.Doc
qbd.www669hq.cn/42064.Doc
qbd.www669hq.cn/40462.Doc
qbd.www669hq.cn/26868.Doc
qbd.www669hq.cn/08048.Doc
qbd.www669hq.cn/99131.Doc
qbd.www669hq.cn/44600.Doc
qbd.www669hq.cn/04486.Doc
qbd.www669hq.cn/48448.Doc
qbd.www669hq.cn/08284.Doc
qbs.www669hq.cn/88404.Doc
qbs.www669hq.cn/28882.Doc
qbs.www669hq.cn/06202.Doc
qbs.www669hq.cn/22084.Doc
qbs.www669hq.cn/00828.Doc
qbs.www669hq.cn/37179.Doc
qbs.www669hq.cn/66206.Doc
qbs.www669hq.cn/88420.Doc
qbs.www669hq.cn/06602.Doc
qbs.www669hq.cn/26660.Doc
qba.www669hq.cn/82086.Doc
qba.www669hq.cn/24608.Doc
qba.www669hq.cn/28266.Doc
qba.www669hq.cn/75175.Doc
qba.www669hq.cn/64244.Doc
qba.www669hq.cn/80406.Doc
qba.www669hq.cn/44040.Doc
qba.www669hq.cn/24408.Doc
qba.www669hq.cn/60062.Doc
qba.www669hq.cn/20026.Doc
qbp.www669hq.cn/53199.Doc
qbp.www669hq.cn/26820.Doc
qbp.www669hq.cn/06046.Doc
qbp.www669hq.cn/48606.Doc
qbp.www669hq.cn/62600.Doc
qbp.www669hq.cn/02086.Doc
qbp.www669hq.cn/02882.Doc
qbp.www669hq.cn/66842.Doc
qbp.www669hq.cn/08220.Doc
qbp.www669hq.cn/20080.Doc
qbo.www669hq.cn/86882.Doc
qbo.www669hq.cn/44802.Doc
qbo.www669hq.cn/28608.Doc
qbo.www669hq.cn/66880.Doc
qbo.www669hq.cn/60002.Doc
qbo.www669hq.cn/02802.Doc
qbo.www669hq.cn/44848.Doc
qbo.www669hq.cn/62826.Doc
qbo.www669hq.cn/68226.Doc
qbo.www669hq.cn/66820.Doc
qbi.www669hq.cn/20826.Doc
qbi.www669hq.cn/82800.Doc
qbi.www669hq.cn/84880.Doc
qbi.www669hq.cn/44488.Doc
qbi.www669hq.cn/02646.Doc
qbi.www669hq.cn/68006.Doc
qbi.www669hq.cn/64022.Doc
qbi.www669hq.cn/86448.Doc
qbi.www669hq.cn/46260.Doc
qbi.www669hq.cn/82088.Doc
qbu.www669hq.cn/88024.Doc
qbu.www669hq.cn/02040.Doc
qbu.www669hq.cn/68420.Doc
qbu.www669hq.cn/24240.Doc
qbu.www669hq.cn/04020.Doc
qbu.www669hq.cn/73559.Doc
qbu.www669hq.cn/80648.Doc
qbu.www669hq.cn/66402.Doc
qbu.www669hq.cn/60802.Doc
qbu.www669hq.cn/08806.Doc
qby.www669hq.cn/48424.Doc
qby.www669hq.cn/84240.Doc
qby.www669hq.cn/84426.Doc
qby.www669hq.cn/48684.Doc
qby.www669hq.cn/08866.Doc
qby.www669hq.cn/28406.Doc
qby.www669hq.cn/66884.Doc
qby.www669hq.cn/40680.Doc
qby.www669hq.cn/68444.Doc
qby.www669hq.cn/00246.Doc
qbt.www669hq.cn/22460.Doc
qbt.www669hq.cn/40840.Doc
qbt.www669hq.cn/60280.Doc
qbt.www669hq.cn/40046.Doc
qbt.www669hq.cn/62882.Doc
qbt.www669hq.cn/68206.Doc
qbt.www669hq.cn/66204.Doc
qbt.www669hq.cn/46282.Doc
qbt.www669hq.cn/02440.Doc
qbt.www669hq.cn/62402.Doc
qbr.www669hq.cn/02644.Doc
qbr.www669hq.cn/24806.Doc
qbr.www669hq.cn/20206.Doc
qbr.www669hq.cn/04608.Doc
qbr.www669hq.cn/84822.Doc
qbr.www669hq.cn/66842.Doc
qbr.www669hq.cn/84482.Doc
qbr.www669hq.cn/62642.Doc
qbr.www669hq.cn/22682.Doc
qbr.www669hq.cn/00642.Doc
qbe.www669hq.cn/28466.Doc
qbe.www669hq.cn/04248.Doc
qbe.www669hq.cn/55599.Doc
qbe.www669hq.cn/24466.Doc
qbe.www669hq.cn/06466.Doc
qbe.www669hq.cn/80060.Doc
qbe.www669hq.cn/91775.Doc
qbe.www669hq.cn/24224.Doc
qbe.www669hq.cn/06288.Doc
qbe.www669hq.cn/46264.Doc
qbw.www669hq.cn/80662.Doc
qbw.www669hq.cn/00484.Doc
qbw.www669hq.cn/00840.Doc
qbw.www669hq.cn/04828.Doc
qbw.www669hq.cn/26062.Doc
qbw.www669hq.cn/28846.Doc
qbw.www669hq.cn/86460.Doc
qbw.www669hq.cn/06284.Doc
qbw.www669hq.cn/86642.Doc
qbw.www669hq.cn/64248.Doc
qbq.www669hq.cn/24280.Doc
qbq.www669hq.cn/22826.Doc
qbq.www669hq.cn/40666.Doc
qbq.www669hq.cn/42284.Doc
qbq.www669hq.cn/44460.Doc
qbq.www669hq.cn/80020.Doc
qbq.www669hq.cn/46222.Doc
qbq.www669hq.cn/24046.Doc
qbq.www669hq.cn/04668.Doc
qbq.www669hq.cn/46800.Doc
qvm.www669hq.cn/60404.Doc
qvm.www669hq.cn/00460.Doc
qvm.www669hq.cn/60608.Doc
qvm.www669hq.cn/08002.Doc
qvm.www669hq.cn/04222.Doc
qvm.www669hq.cn/40424.Doc
qvm.www669hq.cn/00008.Doc
qvm.www669hq.cn/62240.Doc
qvm.www669hq.cn/26866.Doc
qvm.www669hq.cn/06244.Doc
qvn.www669hq.cn/40244.Doc
qvn.www669hq.cn/08400.Doc
qvn.www669hq.cn/66804.Doc
qvn.www669hq.cn/42626.Doc
qvn.www669hq.cn/26804.Doc
qvn.www669hq.cn/31599.Doc
qvn.www669hq.cn/39575.Doc
qvn.www669hq.cn/28060.Doc
qvn.www669hq.cn/86448.Doc
qvn.www669hq.cn/88664.Doc
qvb.www669hq.cn/66442.Doc
qvb.www669hq.cn/02628.Doc
qvb.www669hq.cn/04422.Doc
qvb.www669hq.cn/28200.Doc
qvb.www669hq.cn/44880.Doc
qvb.www669hq.cn/42626.Doc
qvb.www669hq.cn/84222.Doc
qvb.www669hq.cn/60686.Doc
qvb.www669hq.cn/04008.Doc
qvb.www669hq.cn/44688.Doc
qvv.www669hq.cn/80886.Doc
qvv.www669hq.cn/46440.Doc
qvv.www669hq.cn/82248.Doc
qvv.www669hq.cn/86060.Doc
qvv.www669hq.cn/86688.Doc
qvv.www669hq.cn/84020.Doc
qvv.www669hq.cn/64828.Doc
qvv.www669hq.cn/24246.Doc
qvv.www669hq.cn/46686.Doc
qvv.www669hq.cn/84824.Doc
qvc.www669hq.cn/48684.Doc
qvc.www669hq.cn/68020.Doc
qvc.www669hq.cn/62802.Doc
qvc.www669hq.cn/86484.Doc
qvc.www669hq.cn/06204.Doc
qvc.www669hq.cn/68646.Doc
qvc.www669hq.cn/80428.Doc
qvc.www669hq.cn/84044.Doc
qvc.www669hq.cn/84642.Doc
qvc.www669hq.cn/08004.Doc
qvx.www669hq.cn/04860.Doc
qvx.www669hq.cn/17355.Doc
qvx.www669hq.cn/80042.Doc
qvx.www669hq.cn/68484.Doc
qvx.www669hq.cn/57535.Doc
qvx.www669hq.cn/28246.Doc
qvx.www669hq.cn/02248.Doc
qvx.www669hq.cn/66204.Doc
qvx.www669hq.cn/28042.Doc
qvx.www669hq.cn/42680.Doc
qvz.www669hq.cn/86806.Doc
qvz.www669hq.cn/88026.Doc
qvz.www669hq.cn/84682.Doc
qvz.www669hq.cn/66220.Doc
qvz.www669hq.cn/40068.Doc
qvz.www669hq.cn/08442.Doc
qvz.www669hq.cn/02482.Doc
qvz.www669hq.cn/40684.Doc
qvz.www669hq.cn/40828.Doc
qvz.www669hq.cn/04402.Doc
qvl.www669hq.cn/48862.Doc
qvl.www669hq.cn/66468.Doc
qvl.www669hq.cn/84008.Doc
qvl.www669hq.cn/02208.Doc
qvl.www669hq.cn/24024.Doc
qvl.www669hq.cn/22228.Doc
qvl.www669hq.cn/68424.Doc
qvl.www669hq.cn/08462.Doc
qvl.www669hq.cn/64486.Doc
qvl.www669hq.cn/82866.Doc
qvk.www669hq.cn/20288.Doc
qvk.www669hq.cn/57771.Doc
qvk.www669hq.cn/22480.Doc
qvk.www669hq.cn/04824.Doc
qvk.www669hq.cn/04064.Doc
qvk.www669hq.cn/20860.Doc
qvk.www669hq.cn/08466.Doc
qvk.www669hq.cn/48400.Doc
qvk.www669hq.cn/28464.Doc
qvk.www669hq.cn/86480.Doc
qvj.www669hq.cn/22466.Doc
qvj.www669hq.cn/20448.Doc
qvj.www669hq.cn/08444.Doc
qvj.www669hq.cn/44068.Doc
qvj.www669hq.cn/82886.Doc
qvj.www669hq.cn/82248.Doc
qvj.www669hq.cn/24024.Doc
qvj.www669hq.cn/82866.Doc
qvj.www669hq.cn/62248.Doc
qvj.www669hq.cn/84484.Doc
qvh.www669hq.cn/42246.Doc
qvh.www669hq.cn/86286.Doc
qvh.www669hq.cn/08620.Doc
qvh.www669hq.cn/20828.Doc
qvh.www669hq.cn/42826.Doc
qvh.www669hq.cn/48288.Doc
qvh.www669hq.cn/33335.Doc
qvh.www669hq.cn/19319.Doc
qvh.www669hq.cn/55377.Doc
qvh.www669hq.cn/17151.Doc
qvg.www669hq.cn/59333.Doc
qvg.www669hq.cn/79157.Doc
qvg.www669hq.cn/57337.Doc
qvg.www669hq.cn/46264.Doc
qvg.www669hq.cn/48020.Doc
qvg.www669hq.cn/73531.Doc
qvg.www669hq.cn/55739.Doc
qvg.www669hq.cn/39151.Doc
qvg.www669hq.cn/31993.Doc
qvg.www669hq.cn/31519.Doc
qvf.www669hq.cn/55733.Doc
qvf.www669hq.cn/55539.Doc
qvf.www669hq.cn/15531.Doc
qvf.www669hq.cn/59771.Doc
qvf.www669hq.cn/99159.Doc
qvf.www669hq.cn/73139.Doc
qvf.www669hq.cn/59339.Doc
qvf.www669hq.cn/37959.Doc
qvf.www669hq.cn/13791.Doc
qvf.www669hq.cn/93191.Doc
qvd.www669hq.cn/35371.Doc
qvd.www669hq.cn/31515.Doc
qvd.www669hq.cn/15731.Doc
qvd.www669hq.cn/59153.Doc
qvd.www669hq.cn/99131.Doc
qvd.www669hq.cn/57595.Doc
qvd.www669hq.cn/71575.Doc
qvd.www669hq.cn/19915.Doc
qvd.www669hq.cn/11391.Doc
qvd.www669hq.cn/73377.Doc
qvs.www669hq.cn/13993.Doc
qvs.www669hq.cn/79953.Doc
qvs.www669hq.cn/73319.Doc
qvs.www669hq.cn/28208.Doc
qvs.www669hq.cn/99335.Doc
qvs.www669hq.cn/99713.Doc
qvs.www669hq.cn/68240.Doc
qvs.www669hq.cn/44480.Doc
qvs.www669hq.cn/51717.Doc
qvs.www669hq.cn/51793.Doc
qva.www669hq.cn/99733.Doc
qva.www669hq.cn/97397.Doc
qva.www669hq.cn/53317.Doc
qva.www669hq.cn/26460.Doc
qva.www669hq.cn/82066.Doc
qva.www669hq.cn/17537.Doc
qva.www669hq.cn/99579.Doc
qva.www669hq.cn/93359.Doc
qva.www669hq.cn/64040.Doc
qva.www669hq.cn/51351.Doc
qvp.www669hq.cn/31577.Doc
qvp.www669hq.cn/53771.Doc
qvp.www669hq.cn/37393.Doc
qvp.www669hq.cn/13533.Doc
qvp.www669hq.cn/95559.Doc
qvp.www669hq.cn/39555.Doc
qvp.www669hq.cn/42048.Doc
qvp.www669hq.cn/28404.Doc
qvp.www669hq.cn/97977.Doc
qvp.www669hq.cn/39957.Doc
qvo.www669hq.cn/82882.Doc
qvo.www669hq.cn/91131.Doc
qvo.www669hq.cn/48028.Doc
qvo.www669hq.cn/13591.Doc
qvo.www669hq.cn/22882.Doc
qvo.www669hq.cn/13339.Doc
qvo.www669hq.cn/53371.Doc
qvo.www669hq.cn/73131.Doc
qvo.www669hq.cn/75331.Doc
qvo.www669hq.cn/55559.Doc
qvi.www669hq.cn/31919.Doc
qvi.www669hq.cn/75351.Doc
qvi.www669hq.cn/93599.Doc
qvi.www669hq.cn/31533.Doc
qvi.www669hq.cn/99537.Doc
qvi.www669hq.cn/15311.Doc
qvi.www669hq.cn/57799.Doc
qvi.www669hq.cn/57577.Doc
qvi.www669hq.cn/99191.Doc
qvi.www669hq.cn/37979.Doc
qvu.www669hq.cn/95935.Doc
qvu.www669hq.cn/71799.Doc
qvu.www669hq.cn/15995.Doc
qvu.www669hq.cn/19197.Doc
qvu.www669hq.cn/11537.Doc
qvu.www669hq.cn/99957.Doc
qvu.www669hq.cn/99971.Doc
qvu.www669hq.cn/91515.Doc
qvu.www669hq.cn/39193.Doc
qvu.www669hq.cn/77739.Doc
qvy.www669hq.cn/19919.Doc
qvy.www669hq.cn/31599.Doc
qvy.www669hq.cn/71519.Doc
qvy.www669hq.cn/93357.Doc
qvy.www669hq.cn/59753.Doc
qvy.www669hq.cn/91599.Doc
qvy.www669hq.cn/24002.Doc
qvy.www669hq.cn/97939.Doc
qvy.www669hq.cn/55713.Doc
qvy.www669hq.cn/17951.Doc
qvt.www669hq.cn/37511.Doc
qvt.www669hq.cn/57517.Doc
qvt.www669hq.cn/75317.Doc
qvt.www669hq.cn/91597.Doc
qvt.www669hq.cn/55571.Doc
qvt.www669hq.cn/77591.Doc
qvt.www669hq.cn/97399.Doc
qvt.www669hq.cn/91317.Doc
qvt.www669hq.cn/55931.Doc
qvt.www669hq.cn/31971.Doc
qvr.www669hq.cn/11953.Doc
qvr.www669hq.cn/93599.Doc
qvr.www669hq.cn/73931.Doc
qvr.www669hq.cn/95511.Doc
qvr.www669hq.cn/17915.Doc
qvr.www669hq.cn/04424.Doc
qvr.www669hq.cn/79337.Doc
qvr.www669hq.cn/39137.Doc
qvr.www669hq.cn/02824.Doc
qvr.www669hq.cn/97339.Doc
