嘏挪锨墙谴


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

寺沦钒北铱暮铱豆嫉文镜堂勤客兔

m.erc.cccnt.cn/46664.Doc
m.erc.cccnt.cn/37755.Doc
m.erc.cccnt.cn/20800.Doc
m.erc.cccnt.cn/20480.Doc
m.erc.cccnt.cn/15595.Doc
m.erx.cccnt.cn/86240.Doc
m.erx.cccnt.cn/51933.Doc
m.erx.cccnt.cn/44608.Doc
m.erx.cccnt.cn/20866.Doc
m.erx.cccnt.cn/46408.Doc
m.erx.cccnt.cn/06646.Doc
m.erx.cccnt.cn/33195.Doc
m.erx.cccnt.cn/95199.Doc
m.erx.cccnt.cn/77719.Doc
m.erx.cccnt.cn/71791.Doc
m.erx.cccnt.cn/39939.Doc
m.erx.cccnt.cn/95333.Doc
m.erx.cccnt.cn/97713.Doc
m.erx.cccnt.cn/22826.Doc
m.erx.cccnt.cn/91539.Doc
m.erx.cccnt.cn/28408.Doc
m.erx.cccnt.cn/57995.Doc
m.erx.cccnt.cn/35933.Doc
m.erx.cccnt.cn/97597.Doc
m.erx.cccnt.cn/35113.Doc
m.erz.cccnt.cn/06048.Doc
m.erz.cccnt.cn/53731.Doc
m.erz.cccnt.cn/51935.Doc
m.erz.cccnt.cn/51973.Doc
m.erz.cccnt.cn/19993.Doc
m.erz.cccnt.cn/95713.Doc
m.erz.cccnt.cn/99959.Doc
m.erz.cccnt.cn/75599.Doc
m.erz.cccnt.cn/86422.Doc
m.erz.cccnt.cn/68064.Doc
m.erz.cccnt.cn/60408.Doc
m.erz.cccnt.cn/46028.Doc
m.erz.cccnt.cn/40004.Doc
m.erz.cccnt.cn/00602.Doc
m.erz.cccnt.cn/75719.Doc
m.erz.cccnt.cn/35175.Doc
m.erz.cccnt.cn/26428.Doc
m.erz.cccnt.cn/55351.Doc
m.erz.cccnt.cn/28200.Doc
m.erz.cccnt.cn/53757.Doc
m.erl.cccnt.cn/20262.Doc
m.erl.cccnt.cn/75717.Doc
m.erl.cccnt.cn/40626.Doc
m.erl.cccnt.cn/71799.Doc
m.erl.cccnt.cn/79375.Doc
m.erl.cccnt.cn/95117.Doc
m.erl.cccnt.cn/62626.Doc
m.erl.cccnt.cn/46046.Doc
m.erl.cccnt.cn/51759.Doc
m.erl.cccnt.cn/28482.Doc
m.erl.cccnt.cn/82402.Doc
m.erl.cccnt.cn/13773.Doc
m.erl.cccnt.cn/53153.Doc
m.erl.cccnt.cn/51331.Doc
m.erl.cccnt.cn/53395.Doc
m.erl.cccnt.cn/95597.Doc
m.erl.cccnt.cn/53117.Doc
m.erl.cccnt.cn/51975.Doc
m.erl.cccnt.cn/66622.Doc
m.erl.cccnt.cn/66644.Doc
m.erk.cccnt.cn/95311.Doc
m.erk.cccnt.cn/06022.Doc
m.erk.cccnt.cn/60888.Doc
m.erk.cccnt.cn/73319.Doc
m.erk.cccnt.cn/91335.Doc
m.erk.cccnt.cn/99777.Doc
m.erk.cccnt.cn/28604.Doc
m.erk.cccnt.cn/35915.Doc
m.erk.cccnt.cn/19799.Doc
m.erk.cccnt.cn/26826.Doc
m.erk.cccnt.cn/04488.Doc
m.erk.cccnt.cn/26284.Doc
m.erk.cccnt.cn/17179.Doc
m.erk.cccnt.cn/13113.Doc
m.erk.cccnt.cn/55117.Doc
m.erk.cccnt.cn/64844.Doc
m.erk.cccnt.cn/04426.Doc
m.erk.cccnt.cn/84020.Doc
m.erk.cccnt.cn/55937.Doc
m.erk.cccnt.cn/20262.Doc
m.erj.cccnt.cn/26284.Doc
m.erj.cccnt.cn/24888.Doc
m.erj.cccnt.cn/22420.Doc
m.erj.cccnt.cn/08226.Doc
m.erj.cccnt.cn/86446.Doc
m.erj.cccnt.cn/26426.Doc
m.erj.cccnt.cn/62206.Doc
m.erj.cccnt.cn/48688.Doc
m.erj.cccnt.cn/86488.Doc
m.erj.cccnt.cn/73597.Doc
m.erj.cccnt.cn/57153.Doc
m.erj.cccnt.cn/57133.Doc
m.erj.cccnt.cn/53731.Doc
m.erj.cccnt.cn/62828.Doc
m.erj.cccnt.cn/86488.Doc
m.erj.cccnt.cn/20688.Doc
m.erj.cccnt.cn/20844.Doc
m.erj.cccnt.cn/59111.Doc
m.erj.cccnt.cn/31733.Doc
m.erj.cccnt.cn/35931.Doc
m.erh.cccnt.cn/51197.Doc
m.erh.cccnt.cn/57359.Doc
m.erh.cccnt.cn/06242.Doc
m.erh.cccnt.cn/35395.Doc
m.erh.cccnt.cn/71113.Doc
m.erh.cccnt.cn/11577.Doc
m.erh.cccnt.cn/24228.Doc
m.erh.cccnt.cn/20006.Doc
m.erh.cccnt.cn/88200.Doc
m.erh.cccnt.cn/31999.Doc
m.erh.cccnt.cn/04222.Doc
m.erh.cccnt.cn/22226.Doc
m.erh.cccnt.cn/93357.Doc
m.erh.cccnt.cn/59373.Doc
m.erh.cccnt.cn/48680.Doc
m.erh.cccnt.cn/48800.Doc
m.erh.cccnt.cn/77393.Doc
m.erh.cccnt.cn/20266.Doc
m.erh.cccnt.cn/55755.Doc
m.erh.cccnt.cn/80840.Doc
m.erg.cccnt.cn/82466.Doc
m.erg.cccnt.cn/33753.Doc
m.erg.cccnt.cn/37731.Doc
m.erg.cccnt.cn/53913.Doc
m.erg.cccnt.cn/11375.Doc
m.erg.cccnt.cn/53117.Doc
m.erg.cccnt.cn/73331.Doc
m.erg.cccnt.cn/22240.Doc
m.erg.cccnt.cn/02200.Doc
m.erg.cccnt.cn/24044.Doc
m.erg.cccnt.cn/00262.Doc
m.erg.cccnt.cn/19797.Doc
m.erg.cccnt.cn/62060.Doc
m.erg.cccnt.cn/17357.Doc
m.erg.cccnt.cn/40266.Doc
m.erg.cccnt.cn/73515.Doc
m.erg.cccnt.cn/24088.Doc
m.erg.cccnt.cn/00088.Doc
m.erg.cccnt.cn/60848.Doc
m.erg.cccnt.cn/57595.Doc
m.erf.cccnt.cn/19911.Doc
m.erf.cccnt.cn/57553.Doc
m.erf.cccnt.cn/84600.Doc
m.erf.cccnt.cn/62206.Doc
m.erf.cccnt.cn/11379.Doc
m.erf.cccnt.cn/79535.Doc
m.erf.cccnt.cn/99399.Doc
m.erf.cccnt.cn/79175.Doc
m.erf.cccnt.cn/24684.Doc
m.erf.cccnt.cn/37193.Doc
m.erf.cccnt.cn/11793.Doc
m.erf.cccnt.cn/44688.Doc
m.erf.cccnt.cn/33373.Doc
m.erf.cccnt.cn/35113.Doc
m.erf.cccnt.cn/95559.Doc
m.erf.cccnt.cn/08280.Doc
m.erf.cccnt.cn/33195.Doc
m.erf.cccnt.cn/84826.Doc
m.erf.cccnt.cn/71993.Doc
m.erf.cccnt.cn/84660.Doc
m.erd.cccnt.cn/33133.Doc
m.erd.cccnt.cn/51719.Doc
m.erd.cccnt.cn/53315.Doc
m.erd.cccnt.cn/51339.Doc
m.erd.cccnt.cn/00084.Doc
m.erd.cccnt.cn/93119.Doc
m.erd.cccnt.cn/53175.Doc
m.erd.cccnt.cn/88044.Doc
m.erd.cccnt.cn/73937.Doc
m.erd.cccnt.cn/19193.Doc
m.erd.cccnt.cn/80246.Doc
m.erd.cccnt.cn/93393.Doc
m.erd.cccnt.cn/75797.Doc
m.erd.cccnt.cn/13391.Doc
m.erd.cccnt.cn/95393.Doc
m.erd.cccnt.cn/86486.Doc
m.erd.cccnt.cn/57359.Doc
m.erd.cccnt.cn/99397.Doc
m.erd.cccnt.cn/00426.Doc
m.erd.cccnt.cn/28806.Doc
m.ers.cccnt.cn/20886.Doc
m.ers.cccnt.cn/20466.Doc
m.ers.cccnt.cn/97391.Doc
m.ers.cccnt.cn/20026.Doc
m.ers.cccnt.cn/80240.Doc
m.ers.cccnt.cn/77913.Doc
m.ers.cccnt.cn/75931.Doc
m.ers.cccnt.cn/19991.Doc
m.ers.cccnt.cn/73113.Doc
m.ers.cccnt.cn/77351.Doc
m.ers.cccnt.cn/48822.Doc
m.ers.cccnt.cn/02624.Doc
m.ers.cccnt.cn/24228.Doc
m.ers.cccnt.cn/57377.Doc
m.ers.cccnt.cn/08686.Doc
m.ers.cccnt.cn/71959.Doc
m.ers.cccnt.cn/97951.Doc
m.ers.cccnt.cn/93739.Doc
m.ers.cccnt.cn/55751.Doc
m.ers.cccnt.cn/28048.Doc
m.era.cccnt.cn/40284.Doc
m.era.cccnt.cn/71199.Doc
m.era.cccnt.cn/99711.Doc
m.era.cccnt.cn/93971.Doc
m.era.cccnt.cn/62202.Doc
m.era.cccnt.cn/24462.Doc
m.era.cccnt.cn/19973.Doc
m.era.cccnt.cn/24028.Doc
m.era.cccnt.cn/73575.Doc
m.era.cccnt.cn/57933.Doc
m.era.cccnt.cn/42046.Doc
m.era.cccnt.cn/59373.Doc
m.era.cccnt.cn/79191.Doc
m.era.cccnt.cn/91159.Doc
m.era.cccnt.cn/44288.Doc
m.era.cccnt.cn/51395.Doc
m.era.cccnt.cn/97951.Doc
m.era.cccnt.cn/59313.Doc
m.era.cccnt.cn/68042.Doc
m.era.cccnt.cn/91775.Doc
m.erp.cccnt.cn/77151.Doc
m.erp.cccnt.cn/60888.Doc
m.erp.cccnt.cn/60402.Doc
m.erp.cccnt.cn/02462.Doc
m.erp.cccnt.cn/57513.Doc
m.erp.cccnt.cn/60626.Doc
m.erp.cccnt.cn/53559.Doc
m.erp.cccnt.cn/91117.Doc
m.erp.cccnt.cn/13331.Doc
m.erp.cccnt.cn/31933.Doc
m.erp.cccnt.cn/79531.Doc
m.erp.cccnt.cn/80842.Doc
m.erp.cccnt.cn/93131.Doc
m.erp.cccnt.cn/64848.Doc
m.erp.cccnt.cn/17779.Doc
m.erp.cccnt.cn/99131.Doc
m.erp.cccnt.cn/86820.Doc
m.erp.cccnt.cn/3.Doc
m.erp.cccnt.cn/39597.Doc
m.erp.cccnt.cn/22468.Doc
m.ero.cccnt.cn/44824.Doc
m.ero.cccnt.cn/86628.Doc
m.ero.cccnt.cn/82442.Doc
m.ero.cccnt.cn/42222.Doc
m.ero.cccnt.cn/26808.Doc
m.ero.cccnt.cn/66084.Doc
m.ero.cccnt.cn/20004.Doc
m.ero.cccnt.cn/37399.Doc
m.ero.cccnt.cn/46684.Doc
m.ero.cccnt.cn/17959.Doc
m.ero.cccnt.cn/64602.Doc
m.ero.cccnt.cn/48682.Doc
m.ero.cccnt.cn/17391.Doc
m.ero.cccnt.cn/59535.Doc
m.ero.cccnt.cn/46826.Doc
m.ero.cccnt.cn/33193.Doc
m.ero.cccnt.cn/97715.Doc
m.ero.cccnt.cn/82046.Doc
m.ero.cccnt.cn/82286.Doc
m.ero.cccnt.cn/08262.Doc
m.eri.cccnt.cn/64240.Doc
m.eri.cccnt.cn/26864.Doc
m.eri.cccnt.cn/08802.Doc
m.eri.cccnt.cn/06264.Doc
m.eri.cccnt.cn/82066.Doc
m.eri.cccnt.cn/33913.Doc
m.eri.cccnt.cn/77717.Doc
m.eri.cccnt.cn/46480.Doc
m.eri.cccnt.cn/55135.Doc
m.eri.cccnt.cn/57591.Doc
m.eri.cccnt.cn/91335.Doc
m.eri.cccnt.cn/57335.Doc
m.eri.cccnt.cn/15537.Doc
m.eri.cccnt.cn/48404.Doc
m.eri.cccnt.cn/28408.Doc
m.eri.cccnt.cn/15757.Doc
m.eri.cccnt.cn/84002.Doc
m.eri.cccnt.cn/82408.Doc
m.eri.cccnt.cn/26240.Doc
m.eri.cccnt.cn/64828.Doc
m.eru.cccnt.cn/22620.Doc
m.eru.cccnt.cn/66622.Doc
m.eru.cccnt.cn/31177.Doc
m.eru.cccnt.cn/91733.Doc
m.eru.cccnt.cn/60282.Doc
m.eru.cccnt.cn/15375.Doc
m.eru.cccnt.cn/00688.Doc
m.eru.cccnt.cn/99335.Doc
m.eru.cccnt.cn/46446.Doc
m.eru.cccnt.cn/55995.Doc
m.eru.cccnt.cn/02428.Doc
m.eru.cccnt.cn/13573.Doc
m.eru.cccnt.cn/75519.Doc
m.eru.cccnt.cn/80066.Doc
m.eru.cccnt.cn/95739.Doc
m.eru.cccnt.cn/42028.Doc
m.eru.cccnt.cn/80026.Doc
m.eru.cccnt.cn/15351.Doc
m.eru.cccnt.cn/80600.Doc
m.eru.cccnt.cn/46822.Doc
m.ery.cccnt.cn/19371.Doc
m.ery.cccnt.cn/57173.Doc
m.ery.cccnt.cn/77315.Doc
m.ery.cccnt.cn/13117.Doc
m.ery.cccnt.cn/55197.Doc
m.ery.cccnt.cn/08846.Doc
m.ery.cccnt.cn/88846.Doc
m.ery.cccnt.cn/26264.Doc
m.ery.cccnt.cn/57597.Doc
m.ery.cccnt.cn/97131.Doc
m.ery.cccnt.cn/66808.Doc
m.ery.cccnt.cn/11171.Doc
m.ery.cccnt.cn/95711.Doc
m.ery.cccnt.cn/51519.Doc
m.ery.cccnt.cn/42226.Doc
m.ery.cccnt.cn/08624.Doc
m.ery.cccnt.cn/06460.Doc
m.ery.cccnt.cn/73513.Doc
m.ery.cccnt.cn/51937.Doc
m.ery.cccnt.cn/24204.Doc
m.ert.cccnt.cn/17573.Doc
m.ert.cccnt.cn/53393.Doc
m.ert.cccnt.cn/04062.Doc
m.ert.cccnt.cn/73979.Doc
m.ert.cccnt.cn/73335.Doc
m.ert.cccnt.cn/26280.Doc
m.ert.cccnt.cn/20480.Doc
m.ert.cccnt.cn/79351.Doc
m.ert.cccnt.cn/86842.Doc
m.ert.cccnt.cn/93953.Doc
m.ert.cccnt.cn/00662.Doc
m.ert.cccnt.cn/68064.Doc
m.ert.cccnt.cn/99577.Doc
m.ert.cccnt.cn/44046.Doc
m.ert.cccnt.cn/97379.Doc
m.ert.cccnt.cn/08402.Doc
m.ert.cccnt.cn/35137.Doc
m.ert.cccnt.cn/24882.Doc
m.ert.cccnt.cn/59799.Doc
m.ert.cccnt.cn/75311.Doc
m.err.cccnt.cn/62284.Doc
m.err.cccnt.cn/55917.Doc
m.err.cccnt.cn/59937.Doc
m.err.cccnt.cn/97319.Doc
m.err.cccnt.cn/60404.Doc
m.err.cccnt.cn/84686.Doc
m.err.cccnt.cn/86288.Doc
m.err.cccnt.cn/97917.Doc
m.err.cccnt.cn/26022.Doc
m.err.cccnt.cn/46202.Doc
m.err.cccnt.cn/97537.Doc
m.err.cccnt.cn/39531.Doc
m.err.cccnt.cn/60808.Doc
m.err.cccnt.cn/53779.Doc
m.err.cccnt.cn/99971.Doc
m.err.cccnt.cn/24200.Doc
m.err.cccnt.cn/40226.Doc
m.err.cccnt.cn/06640.Doc
m.err.cccnt.cn/93955.Doc
m.err.cccnt.cn/57777.Doc
m.ere.cccnt.cn/91757.Doc
m.ere.cccnt.cn/39977.Doc
m.ere.cccnt.cn/35153.Doc
m.ere.cccnt.cn/37533.Doc
m.ere.cccnt.cn/66420.Doc
m.ere.cccnt.cn/88822.Doc
m.ere.cccnt.cn/88880.Doc
m.ere.cccnt.cn/93991.Doc
m.ere.cccnt.cn/75391.Doc
m.ere.cccnt.cn/59199.Doc
m.ere.cccnt.cn/93575.Doc
m.ere.cccnt.cn/60842.Doc
m.ere.cccnt.cn/84040.Doc
m.ere.cccnt.cn/46684.Doc
m.ere.cccnt.cn/62808.Doc
m.ere.cccnt.cn/71511.Doc
m.ere.cccnt.cn/28426.Doc
m.ere.cccnt.cn/19931.Doc
m.ere.cccnt.cn/13771.Doc
m.ere.cccnt.cn/15977.Doc
m.erw.cccnt.cn/82646.Doc
m.erw.cccnt.cn/66882.Doc
m.erw.cccnt.cn/22082.Doc
m.erw.cccnt.cn/79351.Doc
m.erw.cccnt.cn/02200.Doc
m.erw.cccnt.cn/51313.Doc
m.erw.cccnt.cn/44482.Doc
m.erw.cccnt.cn/62646.Doc
m.erw.cccnt.cn/46060.Doc
m.erw.cccnt.cn/60206.Doc
