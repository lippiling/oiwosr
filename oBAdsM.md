芽司孕沿渍


Python 抽象基类 vs 协议对比
=================================

Python 提供两种接口定义方式：ABC（抽象基类）显式声明继承关系，
Protocol（协议）支持结构子类型（duck typing）。本文深入对比两者。

1. ABC——显式接口（is-a 关系）
-----------------------------------
ABC 使用 @abstractmethod 定义接口，子类必须显式继承。

from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    """支付处理器抽象基类——定义接口契约"""
    @abstractmethod
    def pay(self, amount: float) -> bool:
        """执行支付，返回是否成功"""
        ...

    @abstractmethod
    def refund(self, transaction_id: str) -> bool:
        """执行退款"""
        ...

    def get_fee(self, amount: float) -> float:
        """非抽象方法——提供默认实现"""
        return amount * 0.01  # 1% 手续费


class AlipayProcessor(PaymentProcessor):
    """支付宝支付——必须实现所有抽象方法"""
    def pay(self, amount: float) -> bool:
        print(f"支付宝支付: {amount}")
        return True

    def refund(self, transaction_id: str) -> bool:
        print(f"支付宝退款: {transaction_id}")
        return True

    # get_fee 使用默认实现


class WechatPayProcessor(PaymentProcessor):
    """微信支付——也必须实现所有抽象方法"""
    def pay(self, amount: float) -> bool:
        print(f"微信支付: {amount}")
        return True

    def refund(self, transaction_id: str) -> bool:
        print(f"微信退款: {transaction_id}")
        return True


# 类型检查——isinstance 可以验证
proc = AlipayProcessor()
print(isinstance(proc, PaymentProcessor))  # True

# 未实现抽象方法时实例化会报错
class IncompleteProcessor(PaymentProcessor):
    pass

# ip = IncompleteProcessor()  # TypeError!


2. Protocol——结构子类型（duck typing）
------------------------------------------
Protocol 只要对象有相应方法/属性即视为兼容，无需显式继承。

from typing import Protocol

class Flyable(Protocol):
    """飞行能力协议——有 fly 方法的都是 Flyable"""
    def fly(self) -> str: ...


class Bird:
    """鸟——有 fly 方法，自动满足 Flyable 协议"""
    def fly(self) -> str:
        return "鸟儿飞翔"


class Airplane:
    """飞机——也有 fly 方法，同样满足 Flyable 协议"""
    def fly(self) -> str:
        return "飞机飞行"


class Fish:
    """鱼——没有 fly 方法，不满足 Flyable 协议"""
    def swim(self) -> str:
        return "鱼儿游泳"


def let_it_fly(thing: Flyable) -> None:
    """接受任何 Flyable 对象（结构子类型）"""
    print(thing.fly())


# 无需继承 Flyable——自动满足
let_it_fly(Bird())       # 鸟儿飞翔
let_it_fly(Airplane())   # 飞机飞行
# let_it_fly(Fish())     # 类型检查会报错（静态）


3. @runtime_checkable——运行时检查协议
------------------------------------------
默认 Protocol 不支持 isinstance，需加装饰器。

from typing import Protocol, runtime_checkable

@runtime_checkable
class Drawable(Protocol):
    """可绘制协议——支持运行时 isinstance 检查"""
    def draw(self) -> str: ...


class Circle:
    def draw(self) -> str:
        return "绘制圆形"


class Square:
    def draw(self) -> str:
        return "绘制正方形"


# 现在 isinstance 可工作
c = Circle()
print(isinstance(c, Drawable))  # True——有 draw 方法

# 注意：@runtime_checkable 只检查方法名是否存在，不检查签名！
class WrongDraw:
    def draw(self, extra_arg: str) -> str:
        return "参数签名不匹配但也能通过"


print(isinstance(WrongDraw(), Drawable))  # True——只检查方法存在


4. ABC.register()——注册虚拟子类
---------------------------------------
无需修改源代码，让一个类成为 ABC 的虚拟子类。

from abc import ABC

class MyIterable(ABC):
    """自定义可迭代抽象基类"""
    @abstractmethod
    def __iter__(self):
        ...

    @classmethod
    def __subclasshook__(cls, other):
        """只要实现了 __iter__ 就算子类"""
        if cls is MyIterable:
            if any("__iter__" in B.__dict__ for B in other.__mro__):
                return True
        return NotImplemented


# 注册第三方类为虚拟子类
class ExternalList:
    """外部列表类——无法修改源代码"""
    def __iter__(self):
        return iter([1, 2, 3])


# 方式一：register
MyIterable.register(ExternalList)
print(isinstance(ExternalList(), MyIterable))  # True

# 方式二：__subclasshook__（如上所示），自动检测结构匹配


5. __subclasshook__——自定义子类检测
------------------------------------------
在 ABC 中定义 __subclasshook__ 来检测结构匹配。

from abc import ABC, abstractmethod

class Hashable(ABC):
    """可哈希接口——使用 __subclasshook__ 自动检测"""
    @abstractmethod
    def __hash__(self) -> int:
        ...

    @classmethod
    def __subclasshook__(cls, other):
        if cls is Hashable:
            # 检查是否实现了 __hash__
            for B in other.__mro__:
                if "__hash__" in B.__dict__:
                    return True
        return NotImplemented


# Python 内置的 str 自动匹配
print(isinstance("hello", Hashable))  # True——str 有 __hash__
print(isinstance([1, 2], Hashable))   # False——list 没有 __hash__


6. collections.abc 使用模式
-------------------------------
标准库提供了丰富的 ABC，应优先使用。

from collections.abc import Sequence, MutableMapping, Iterable

class MyList(Sequence):
    """实现 Sequence 接口——自动获得 __contains__、__iter__ 等"""
    def __init__(self, items):
        self._items = list(items)

    def __getitem__(self, index):
        return self._items[index]

    def __len__(self):
        return len(self._items)

    # 只需要实现 __getitem__ 和 __len__
    # Sequence 自动提供：__contains__、__iter__、reversed、index、count


ml = MyList([1, 2, 3, 4, 5])
print(3 in ml)          # True——来自 Sequence
for x in ml:
    print(x, end=" ")   # 1 2 3 4 5——来自 Sequence
print()
print(ml.index(3))      # 2——来自 Sequence


7. 如何选择：ABC vs Protocol
--------------------------------
选择 ABC：
- 明确的 is-a 关系（如 "支付宝是一种支付处理器"）
- 需要提供默认方法实现
- 需要 __subclasshook__ 或 register()
- 框架要求继承特定基类（如 Django、SQLAlchemy）

选择 Protocol：
- 关注"能做什么"而非"是什么"（duck typing）
- 不希望强制继承关系
- 需要与第三方库代码协作且不能修改源码
- 希望保持类型检查同时保持 Python 的灵活性

"""
经验法则：当你想定义"一个东西能做某事"时用 Protocol，
当你想定义"一个东西是什么"时用 ABC。
"""

# 典型 Protocol 场景：鸭子类型
class Printable(Protocol):
    def __str__(self) -> str: ...

def display(item: Printable) -> None:
    """接受任何可打印的对象"""
    print(str(item))


# 典型 ABC 场景：家族层次
class Vehicle(ABC):
    @abstractmethod
    def start(self) -> None: ...

    @abstractmethod
    def stop(self) -> None: ...

    def honk(self) -> None:
        print("喇叭鸣响")  # 默认实现


class Car(Vehicle):
    def start(self) -> None:
        print("汽车启动")

    def stop(self) -> None:
        print("汽车停止")


总结：ABC 适用于明确的继承层次和需要默认实现的场景，
Protocol 适用于鸭子类型和松耦合场景。两者并非对立——
ABC 侧重"是什么"（显式接口），Protocol 侧重"能做什么"（结构子类型）。
现代 Python 倾向于在类型提示中使用 Protocol 以获得更大灵活性。

挚扔蔽淌矢砸欠现妆诱芭睾邪釉驯

qhd.ygdig4s.cn/046645.htm
qhd.ygdig4s.cn/353735.htm
qhd.ygdig4s.cn/933535.htm
qhd.ygdig4s.cn/000065.htm
qhd.ygdig4s.cn/248485.htm
qhd.ygdig4s.cn/115955.htm
qhd.ygdig4s.cn/373175.htm
qhd.ygdig4s.cn/915995.htm
qhs.ygdig4s.cn/248265.htm
qhs.ygdig4s.cn/268485.htm
qhs.ygdig4s.cn/55.htm
qhs.ygdig4s.cn/711915.htm
qhs.ygdig4s.cn/971935.htm
qhs.ygdig4s.cn/024425.htm
qhs.ygdig4s.cn/999955.htm
qhs.ygdig4s.cn/951375.htm
qhs.ygdig4s.cn/133975.htm
qhs.ygdig4s.cn/311535.htm
qha.ygdig4s.cn/282265.htm
qha.ygdig4s.cn/200065.htm
qha.ygdig4s.cn/682225.htm
qha.ygdig4s.cn/008865.htm
qha.ygdig4s.cn/062845.htm
qha.ygdig4s.cn/551395.htm
qha.ygdig4s.cn/139595.htm
qha.ygdig4s.cn/608865.htm
qha.ygdig4s.cn/440085.htm
qha.ygdig4s.cn/282445.htm
qhp.ygdig4s.cn/773375.htm
qhp.ygdig4s.cn/157515.htm
qhp.ygdig4s.cn/737735.htm
qhp.ygdig4s.cn/593315.htm
qhp.ygdig4s.cn/242865.htm
qhp.ygdig4s.cn/577175.htm
qhp.ygdig4s.cn/137195.htm
qhp.ygdig4s.cn/759915.htm
qhp.ygdig4s.cn/735315.htm
qhp.ygdig4s.cn/999515.htm
qho.ygdig4s.cn/711515.htm
qho.ygdig4s.cn/735915.htm
qho.ygdig4s.cn/193915.htm
qho.ygdig4s.cn/222485.htm
qho.ygdig4s.cn/593955.htm
qho.ygdig4s.cn/153335.htm
qho.ygdig4s.cn/739355.htm
qho.ygdig4s.cn/175775.htm
qho.ygdig4s.cn/200005.htm
qho.ygdig4s.cn/957595.htm
qhi.ygdig4s.cn/179775.htm
qhi.ygdig4s.cn/448005.htm
qhi.ygdig4s.cn/995995.htm
qhi.ygdig4s.cn/628085.htm
qhi.ygdig4s.cn/824205.htm
qhi.ygdig4s.cn/755575.htm
qhi.ygdig4s.cn/686485.htm
qhi.ygdig4s.cn/460865.htm
qhi.ygdig4s.cn/333155.htm
qhi.ygdig4s.cn/195335.htm
qhu.ygdig4s.cn/755555.htm
qhu.ygdig4s.cn/204885.htm
qhu.ygdig4s.cn/731775.htm
qhu.ygdig4s.cn/244865.htm
qhu.ygdig4s.cn/684205.htm
qhu.ygdig4s.cn/171975.htm
qhu.ygdig4s.cn/288485.htm
qhu.ygdig4s.cn/399115.htm
qhu.ygdig4s.cn/351375.htm
qhu.ygdig4s.cn/626665.htm
qhy.ygdig4s.cn/337395.htm
qhy.ygdig4s.cn/444285.htm
qhy.ygdig4s.cn/797935.htm
qhy.ygdig4s.cn/399335.htm
qhy.ygdig4s.cn/133715.htm
qhy.ygdig4s.cn/604045.htm
qhy.ygdig4s.cn/040405.htm
qhy.ygdig4s.cn/775355.htm
qhy.ygdig4s.cn/004225.htm
qhy.ygdig4s.cn/080665.htm
qht.ygdig4s.cn/133975.htm
qht.ygdig4s.cn/420285.htm
qht.ygdig4s.cn/444225.htm
qht.ygdig4s.cn/571395.htm
qht.ygdig4s.cn/268605.htm
qht.ygdig4s.cn/333535.htm
qht.ygdig4s.cn/957795.htm
qht.ygdig4s.cn/644825.htm
qht.ygdig4s.cn/591115.htm
qht.ygdig4s.cn/197375.htm
qhr.ygdig4s.cn/359755.htm
qhr.ygdig4s.cn/046285.htm
qhr.ygdig4s.cn/171975.htm
qhr.ygdig4s.cn/622605.htm
qhr.ygdig4s.cn/444245.htm
qhr.ygdig4s.cn/464445.htm
qhr.ygdig4s.cn/317575.htm
qhr.ygdig4s.cn/511355.htm
qhr.ygdig4s.cn/206245.htm
qhr.ygdig4s.cn/820465.htm
qhe.ygdig4s.cn/222025.htm
qhe.ygdig4s.cn/199935.htm
qhe.ygdig4s.cn/395135.htm
qhe.ygdig4s.cn/822865.htm
qhe.ygdig4s.cn/066625.htm
qhe.ygdig4s.cn/197155.htm
qhe.ygdig4s.cn/957935.htm
qhe.ygdig4s.cn/226865.htm
qhe.ygdig4s.cn/880485.htm
qhe.ygdig4s.cn/575975.htm
qhw.ygdig4s.cn/680685.htm
qhw.ygdig4s.cn/797975.htm
qhw.ygdig4s.cn/208065.htm
qhw.ygdig4s.cn/428805.htm
qhw.ygdig4s.cn/179315.htm
qhw.ygdig4s.cn/042285.htm
qhw.ygdig4s.cn/775115.htm
qhw.ygdig4s.cn/884245.htm
qhw.ygdig4s.cn/642245.htm
qhw.ygdig4s.cn/260665.htm
qhq.ygdig4s.cn/393175.htm
qhq.ygdig4s.cn/991355.htm
qhq.ygdig4s.cn/626045.htm
qhq.ygdig4s.cn/937995.htm
qhq.ygdig4s.cn/399715.htm
qhq.ygdig4s.cn/159335.htm
qhq.ygdig4s.cn/951715.htm
qhq.ygdig4s.cn/575175.htm
qhq.ygdig4s.cn/008045.htm
qhq.ygdig4s.cn/355195.htm
qgtv.ygdig4s.cn/646225.htm
qgtv.ygdig4s.cn/406685.htm
qgtv.ygdig4s.cn/466445.htm
qgtv.ygdig4s.cn/066065.htm
qgtv.ygdig4s.cn/933515.htm
qgtv.ygdig4s.cn/602885.htm
qgtv.ygdig4s.cn/153955.htm
qgtv.ygdig4s.cn/428405.htm
qgtv.ygdig4s.cn/791515.htm
qgtv.ygdig4s.cn/260285.htm
qgn.ygdig4s.cn/402665.htm
qgn.ygdig4s.cn/000205.htm
qgn.ygdig4s.cn/288065.htm
qgn.ygdig4s.cn/737795.htm
qgn.ygdig4s.cn/739595.htm
qgn.ygdig4s.cn/153755.htm
qgn.ygdig4s.cn/719315.htm
qgn.ygdig4s.cn/260405.htm
qgn.ygdig4s.cn/846865.htm
qgn.ygdig4s.cn/880605.htm
qgb.ygdig4s.cn/048265.htm
qgb.ygdig4s.cn/171115.htm
qgb.ygdig4s.cn/797315.htm
qgb.ygdig4s.cn/957195.htm
qgb.ygdig4s.cn/513555.htm
qgb.ygdig4s.cn/864845.htm
qgb.ygdig4s.cn/242625.htm
qgb.ygdig4s.cn/664085.htm
qgb.ygdig4s.cn/991155.htm
qgb.ygdig4s.cn/795935.htm
qgv.ygdig4s.cn/000085.htm
qgv.ygdig4s.cn/424825.htm
qgv.ygdig4s.cn/868805.htm
qgv.ygdig4s.cn/264085.htm
qgv.ygdig4s.cn/220825.htm
qgv.ygdig4s.cn/068825.htm
qgv.ygdig4s.cn/484845.htm
qgv.ygdig4s.cn/066825.htm
qgv.ygdig4s.cn/579975.htm
qgv.ygdig4s.cn/739335.htm
qgc.ygdig4s.cn/113735.htm
qgc.ygdig4s.cn/711175.htm
qgc.ygdig4s.cn/048885.htm
qgc.ygdig4s.cn/599795.htm
qgc.ygdig4s.cn/844885.htm
qgc.ygdig4s.cn/557195.htm
qgc.ygdig4s.cn/315775.htm
qgc.ygdig4s.cn/880205.htm
qgc.ygdig4s.cn/664285.htm
qgc.ygdig4s.cn/220665.htm
qgx.ygdig4s.cn/042485.htm
qgx.ygdig4s.cn/119555.htm
qgx.ygdig4s.cn/002225.htm
qgx.ygdig4s.cn/880285.htm
qgx.ygdig4s.cn/600025.htm
qgx.ygdig4s.cn/179995.htm
qgx.ygdig4s.cn/286685.htm
qgx.ygdig4s.cn/717115.htm
qgx.ygdig4s.cn/779315.htm
qgx.ygdig4s.cn/515575.htm
qgz.ygdig4s.cn/531395.htm
qgz.ygdig4s.cn/606865.htm
qgz.ygdig4s.cn/226085.htm
qgz.ygdig4s.cn/511195.htm
qgz.ygdig4s.cn/202025.htm
qgz.ygdig4s.cn/539755.htm
qgz.ygdig4s.cn/660465.htm
qgz.ygdig4s.cn/842465.htm
qgz.ygdig4s.cn/955355.htm
qgz.ygdig4s.cn/393795.htm
qgl.ygdig4s.cn/999775.htm
qgl.ygdig4s.cn/220425.htm
qgl.ygdig4s.cn/088465.htm
qgl.ygdig4s.cn/862625.htm
qgl.ygdig4s.cn/157995.htm
qgl.ygdig4s.cn/280805.htm
qgl.ygdig4s.cn/199395.htm
qgl.ygdig4s.cn/599315.htm
qgl.ygdig4s.cn/519775.htm
qgl.ygdig4s.cn/826405.htm
qgk.ygdig4s.cn/080685.htm
qgk.ygdig4s.cn/739395.htm
qgk.ygdig4s.cn/084005.htm
qgk.ygdig4s.cn/775155.htm
qgk.ygdig4s.cn/882005.htm
qgk.ygdig4s.cn/575795.htm
qgk.ygdig4s.cn/806885.htm
qgk.ygdig4s.cn/628085.htm
qgk.ygdig4s.cn/977975.htm
qgk.ygdig4s.cn/993795.htm
qgj.ygdig4s.cn/804465.htm
qgj.ygdig4s.cn/997155.htm
qgj.ygdig4s.cn/004065.htm
qgj.ygdig4s.cn/264085.htm
qgj.ygdig4s.cn/664465.htm
qgj.ygdig4s.cn/686625.htm
qgj.ygdig4s.cn/826265.htm
qgj.ygdig4s.cn/428065.htm
qgj.ygdig4s.cn/404405.htm
qgj.ygdig4s.cn/806245.htm
qgh.ygdig4s.cn/208865.htm
qgh.ygdig4s.cn/739175.htm
qgh.ygdig4s.cn/331195.htm
qgh.ygdig4s.cn/462885.htm
qgh.ygdig4s.cn/066445.htm
qgh.ygdig4s.cn/973915.htm
qgh.ygdig4s.cn/137955.htm
qgh.ygdig4s.cn/020005.htm
qgh.ygdig4s.cn/159955.htm
qgh.ygdig4s.cn/404685.htm
qgg.ygdig4s.cn/442825.htm
qgg.ygdig4s.cn/288685.htm
qgg.ygdig4s.cn/319575.htm
qgg.ygdig4s.cn/355395.htm
qgg.ygdig4s.cn/397775.htm
qgg.ygdig4s.cn/919395.htm
qgg.ygdig4s.cn/735515.htm
qgg.ygdig4s.cn/731915.htm
qgg.ygdig4s.cn/591335.htm
qgg.ygdig4s.cn/006445.htm
qgf.ygdig4s.cn/608265.htm
qgf.ygdig4s.cn/884665.htm
qgf.ygdig4s.cn/068005.htm
qgf.ygdig4s.cn/660265.htm
qgf.ygdig4s.cn/199555.htm
qgf.ygdig4s.cn/404285.htm
qgf.ygdig4s.cn/351535.htm
qgf.ygdig4s.cn/193995.htm
qgf.ygdig4s.cn/355355.htm
qgf.ygdig4s.cn/680645.htm
qgd.ygdig4s.cn/113795.htm
qgd.ygdig4s.cn/315195.htm
qgd.ygdig4s.cn/062265.htm
qgd.ygdig4s.cn/826225.htm
qgd.ygdig4s.cn/400045.htm
qgd.ygdig4s.cn/846285.htm
qgd.ygdig4s.cn/119135.htm
qgd.ygdig4s.cn/137735.htm
qgd.ygdig4s.cn/113735.htm
qgd.ygdig4s.cn/379515.htm
qgs.ygdig4s.cn/824245.htm
qgs.ygdig4s.cn/113955.htm
qgs.ygdig4s.cn/715535.htm
qgs.ygdig4s.cn/064685.htm
qgs.ygdig4s.cn/151395.htm
qgs.ygdig4s.cn/888285.htm
qgs.ygdig4s.cn/262885.htm
qgs.ygdig4s.cn/404045.htm
qgs.ygdig4s.cn/115335.htm
qgs.ygdig4s.cn/157355.htm
qga.ygdig4s.cn/864025.htm
qga.ygdig4s.cn/577335.htm
qga.ygdig4s.cn/608265.htm
qga.ygdig4s.cn/371795.htm
qga.ygdig4s.cn/048265.htm
qga.ygdig4s.cn/062445.htm
qga.ygdig4s.cn/997395.htm
qga.ygdig4s.cn/975715.htm
qga.ygdig4s.cn/397955.htm
qga.ygdig4s.cn/753755.htm
qgp.ygdig4s.cn/135595.htm
qgp.ygdig4s.cn/197975.htm
qgp.ygdig4s.cn/113595.htm
qgp.ygdig4s.cn/486025.htm
qgp.ygdig4s.cn/248005.htm
qgp.ygdig4s.cn/911315.htm
qgp.ygdig4s.cn/640805.htm
qgp.ygdig4s.cn/286805.htm
qgp.ygdig4s.cn/153515.htm
qgp.ygdig4s.cn/173335.htm
qgo.ygdig4s.cn/933915.htm
qgo.ygdig4s.cn/040265.htm
qgo.ygdig4s.cn/248825.htm
qgo.ygdig4s.cn/246425.htm
qgo.ygdig4s.cn/577955.htm
qgo.ygdig4s.cn/282805.htm
qgo.ygdig4s.cn/626645.htm
qgo.ygdig4s.cn/888805.htm
qgo.ygdig4s.cn/595135.htm
qgo.ygdig4s.cn/531355.htm
qgi.ygdig4s.cn/288025.htm
qgi.ygdig4s.cn/751355.htm
qgi.ygdig4s.cn/466485.htm
qgi.ygdig4s.cn/686685.htm
qgi.ygdig4s.cn/519535.htm
qgi.ygdig4s.cn/608205.htm
qgi.ygdig4s.cn/379595.htm
qgi.ygdig4s.cn/822265.htm
qgi.ygdig4s.cn/997115.htm
qgi.ygdig4s.cn/339795.htm
qgu.ygdig4s.cn/919115.htm
qgu.ygdig4s.cn/600825.htm
qgu.ygdig4s.cn/991155.htm
qgu.ygdig4s.cn/808685.htm
qgu.ygdig4s.cn/313915.htm
qgu.ygdig4s.cn/266685.htm
qgu.ygdig4s.cn/264465.htm
qgu.ygdig4s.cn/579135.htm
qgu.ygdig4s.cn/195995.htm
qgu.ygdig4s.cn/377595.htm
qgy.ygdig4s.cn/060685.htm
qgy.ygdig4s.cn/044885.htm
qgy.ygdig4s.cn/864865.htm
qgy.ygdig4s.cn/511375.htm
qgy.ygdig4s.cn/244085.htm
qgy.ygdig4s.cn/755535.htm
qgy.ygdig4s.cn/531935.htm
qgy.ygdig4s.cn/842665.htm
qgy.ygdig4s.cn/371515.htm
qgy.ygdig4s.cn/264005.htm
qgt.ygdig4s.cn/660825.htm
qgt.ygdig4s.cn/315115.htm
qgt.ygdig4s.cn/224825.htm
qgt.ygdig4s.cn/888265.htm
qgt.ygdig4s.cn/246805.htm
qgt.ygdig4s.cn/713535.htm
qgt.ygdig4s.cn/973735.htm
qgt.ygdig4s.cn/995595.htm
qgt.ygdig4s.cn/775535.htm
qgt.ygdig4s.cn/155995.htm
qgr.ygdig4s.cn/111995.htm
qgr.ygdig4s.cn/357975.htm
qgr.ygdig4s.cn/313395.htm
qgr.ygdig4s.cn/579335.htm
qgr.ygdig4s.cn/886065.htm
qgr.ygdig4s.cn/260025.htm
qgr.ygdig4s.cn/864265.htm
qgr.ygdig4s.cn/513935.htm
qgr.ygdig4s.cn/937535.htm
qgr.ygdig4s.cn/020285.htm
qge.ygdig4s.cn/444845.htm
qge.ygdig4s.cn/375315.htm
qge.ygdig4s.cn/624205.htm
qge.ygdig4s.cn/937795.htm
qge.ygdig4s.cn/511735.htm
qge.ygdig4s.cn/624425.htm
qge.ygdig4s.cn/826245.htm
qge.ygdig4s.cn/537555.htm
qge.ygdig4s.cn/137795.htm
qge.ygdig4s.cn/808205.htm
qgw.ygdig4s.cn/640465.htm
qgw.ygdig4s.cn/864665.htm
qgw.ygdig4s.cn/397935.htm
qgw.ygdig4s.cn/602425.htm
qgw.ygdig4s.cn/268645.htm
qgw.ygdig4s.cn/939595.htm
qgw.ygdig4s.cn/428485.htm
qgw.ygdig4s.cn/993175.htm
qgw.ygdig4s.cn/999935.htm
qgw.ygdig4s.cn/822845.htm
qgq.ygdig4s.cn/199555.htm
qgq.ygdig4s.cn/713755.htm
qgq.ygdig4s.cn/993735.htm
qgq.ygdig4s.cn/468265.htm
qgq.ygdig4s.cn/913915.htm
qgq.ygdig4s.cn/222665.htm
qgq.ygdig4s.cn/280845.htm
qgq.ygdig4s.cn/715355.htm
qgq.ygdig4s.cn/553395.htm
qgq.ygdig4s.cn/080805.htm
qftv.ygdig4s.cn/737735.htm
qftv.ygdig4s.cn/139715.htm
qftv.ygdig4s.cn/484445.htm
qftv.ygdig4s.cn/202225.htm
qftv.ygdig4s.cn/408485.htm
qftv.ygdig4s.cn/266265.htm
qftv.ygdig4s.cn/551715.htm
