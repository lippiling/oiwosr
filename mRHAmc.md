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

eho.xkfrnc.cn/04662.Doc
eho.xkfrnc.cn/07606.Doc
eho.xkfrnc.cn/82660.Doc
eho.xkfrnc.cn/80666.Doc
eho.xkfrnc.cn/20620.Doc
eho.xkfrnc.cn/68062.Doc
eho.xkfrnc.cn/82886.Doc
eho.xkfrnc.cn/00282.Doc
eho.xkfrnc.cn/46606.Doc
eho.xkfrnc.cn/04860.Doc
ehi.xkfrnc.cn/08640.Doc
ehi.xkfrnc.cn/84244.Doc
ehi.xkfrnc.cn/48808.Doc
ehi.xkfrnc.cn/40660.Doc
ehi.xkfrnc.cn/24846.Doc
ehi.xkfrnc.cn/66422.Doc
ehi.xkfrnc.cn/64844.Doc
ehi.xkfrnc.cn/26042.Doc
ehi.xkfrnc.cn/40620.Doc
ehi.xkfrnc.cn/06048.Doc
ehu.xkfrnc.cn/80866.Doc
ehu.xkfrnc.cn/64044.Doc
ehu.xkfrnc.cn/48862.Doc
ehu.xkfrnc.cn/88280.Doc
ehu.xkfrnc.cn/22020.Doc
ehu.xkfrnc.cn/60802.Doc
ehu.xkfrnc.cn/82826.Doc
ehu.xkfrnc.cn/24626.Doc
ehu.xkfrnc.cn/20088.Doc
ehu.xkfrnc.cn/00286.Doc
ehy.xkfrnc.cn/04048.Doc
ehy.xkfrnc.cn/66284.Doc
ehy.xkfrnc.cn/82228.Doc
ehy.xkfrnc.cn/28084.Doc
ehy.xkfrnc.cn/04240.Doc
ehy.xkfrnc.cn/42206.Doc
ehy.xkfrnc.cn/42440.Doc
ehy.xkfrnc.cn/24044.Doc
ehy.xkfrnc.cn/44622.Doc
ehy.xkfrnc.cn/00842.Doc
eht.xkfrnc.cn/28824.Doc
eht.xkfrnc.cn/86644.Doc
eht.xkfrnc.cn/22064.Doc
eht.xkfrnc.cn/44260.Doc
eht.xkfrnc.cn/40660.Doc
eht.xkfrnc.cn/22080.Doc
eht.xkfrnc.cn/40884.Doc
eht.xkfrnc.cn/46004.Doc
eht.xkfrnc.cn/44048.Doc
eht.xkfrnc.cn/46442.Doc
ehr.xkfrnc.cn/42682.Doc
ehr.xkfrnc.cn/26262.Doc
ehr.xkfrnc.cn/06486.Doc
ehr.xkfrnc.cn/04082.Doc
ehr.xkfrnc.cn/24802.Doc
ehr.xkfrnc.cn/04444.Doc
ehr.xkfrnc.cn/64220.Doc
ehr.xkfrnc.cn/68245.Doc
ehr.xkfrnc.cn/46022.Doc
ehr.xkfrnc.cn/86884.Doc
ehe.xkfrnc.cn/24640.Doc
ehe.xkfrnc.cn/44888.Doc
ehe.xkfrnc.cn/28662.Doc
ehe.xkfrnc.cn/28806.Doc
ehe.xkfrnc.cn/28228.Doc
ehe.xkfrnc.cn/62648.Doc
ehe.xkfrnc.cn/22864.Doc
ehe.xkfrnc.cn/04486.Doc
ehe.xkfrnc.cn/24288.Doc
ehe.xkfrnc.cn/46802.Doc
ehw.xkfrnc.cn/00264.Doc
ehw.xkfrnc.cn/08004.Doc
ehw.xkfrnc.cn/04422.Doc
ehw.xkfrnc.cn/66640.Doc
ehw.xkfrnc.cn/46020.Doc
ehw.xkfrnc.cn/04448.Doc
ehw.xkfrnc.cn/86048.Doc
ehw.xkfrnc.cn/28264.Doc
ehw.xkfrnc.cn/60062.Doc
ehw.xkfrnc.cn/22464.Doc
ehq.xkfrnc.cn/08844.Doc
ehq.xkfrnc.cn/24602.Doc
ehq.xkfrnc.cn/68084.Doc
ehq.xkfrnc.cn/82026.Doc
ehq.xkfrnc.cn/68064.Doc
ehq.xkfrnc.cn/22840.Doc
ehq.xkfrnc.cn/82842.Doc
ehq.xkfrnc.cn/40026.Doc
ehq.xkfrnc.cn/28804.Doc
ehq.xkfrnc.cn/53595.Doc
egm.xkfrnc.cn/62868.Doc
egm.xkfrnc.cn/24802.Doc
egm.xkfrnc.cn/42028.Doc
egm.xkfrnc.cn/86068.Doc
egm.xkfrnc.cn/22640.Doc
egm.xkfrnc.cn/20080.Doc
egm.xkfrnc.cn/22822.Doc
egm.xkfrnc.cn/02824.Doc
egm.xkfrnc.cn/42606.Doc
egm.xkfrnc.cn/40240.Doc
egn.xkfrnc.cn/02026.Doc
egn.xkfrnc.cn/20008.Doc
egn.xkfrnc.cn/48606.Doc
egn.xkfrnc.cn/66800.Doc
egn.xkfrnc.cn/04662.Doc
egn.xkfrnc.cn/06288.Doc
egn.xkfrnc.cn/08484.Doc
egn.xkfrnc.cn/06646.Doc
egn.xkfrnc.cn/24880.Doc
egn.xkfrnc.cn/66466.Doc
egb.xkfrnc.cn/48266.Doc
egb.xkfrnc.cn/60060.Doc
egb.xkfrnc.cn/86246.Doc
egb.xkfrnc.cn/86660.Doc
egb.xkfrnc.cn/08440.Doc
egb.xkfrnc.cn/04846.Doc
egb.xkfrnc.cn/80022.Doc
egb.xkfrnc.cn/80228.Doc
egb.xkfrnc.cn/66042.Doc
egb.xkfrnc.cn/81711.Doc
egv.xkfrnc.cn/88204.Doc
egv.xkfrnc.cn/86440.Doc
egv.xkfrnc.cn/89906.Doc
egv.xkfrnc.cn/20666.Doc
egv.xkfrnc.cn/42642.Doc
egv.xkfrnc.cn/48202.Doc
egv.xkfrnc.cn/86882.Doc
egv.xkfrnc.cn/88444.Doc
egv.xkfrnc.cn/62802.Doc
egv.xkfrnc.cn/06282.Doc
egc.xkfrnc.cn/66820.Doc
egc.xkfrnc.cn/48444.Doc
egc.xkfrnc.cn/42808.Doc
egc.xkfrnc.cn/86222.Doc
egc.xkfrnc.cn/24680.Doc
egc.xkfrnc.cn/28486.Doc
egc.xkfrnc.cn/60842.Doc
egc.xkfrnc.cn/26842.Doc
egc.xkfrnc.cn/60088.Doc
egc.xkfrnc.cn/48668.Doc
egx.xkfrnc.cn/06240.Doc
egx.xkfrnc.cn/64620.Doc
egx.xkfrnc.cn/02806.Doc
egx.xkfrnc.cn/20282.Doc
egx.xkfrnc.cn/28608.Doc
egx.xkfrnc.cn/82402.Doc
egx.xkfrnc.cn/48204.Doc
egx.xkfrnc.cn/28266.Doc
egx.xkfrnc.cn/44662.Doc
egx.xkfrnc.cn/88662.Doc
egz.xkfrnc.cn/24022.Doc
egz.xkfrnc.cn/04006.Doc
egz.xkfrnc.cn/64062.Doc
egz.xkfrnc.cn/02204.Doc
egz.xkfrnc.cn/24240.Doc
egz.xkfrnc.cn/64466.Doc
egz.xkfrnc.cn/66866.Doc
egz.xkfrnc.cn/08002.Doc
egz.xkfrnc.cn/02646.Doc
egz.xkfrnc.cn/37439.Doc
egl.xkfrnc.cn/04640.Doc
egl.xkfrnc.cn/62086.Doc
egl.xkfrnc.cn/80426.Doc
egl.xkfrnc.cn/84282.Doc
egl.xkfrnc.cn/28282.Doc
egl.xkfrnc.cn/80408.Doc
egl.xkfrnc.cn/68648.Doc
egl.xkfrnc.cn/26420.Doc
egl.xkfrnc.cn/60646.Doc
egl.xkfrnc.cn/44868.Doc
egk.xkfrnc.cn/64040.Doc
egk.xkfrnc.cn/88422.Doc
egk.xkfrnc.cn/48208.Doc
egk.xkfrnc.cn/82260.Doc
egk.xkfrnc.cn/04804.Doc
egk.xkfrnc.cn/80882.Doc
egk.xkfrnc.cn/42862.Doc
egk.xkfrnc.cn/42648.Doc
egk.xkfrnc.cn/80062.Doc
egk.xkfrnc.cn/46080.Doc
egj.xkfrnc.cn/62626.Doc
egj.xkfrnc.cn/82806.Doc
egj.xkfrnc.cn/60266.Doc
egj.xkfrnc.cn/62080.Doc
egj.xkfrnc.cn/48204.Doc
egj.xkfrnc.cn/04000.Doc
egj.xkfrnc.cn/24604.Doc
egj.xkfrnc.cn/44886.Doc
egj.xkfrnc.cn/68648.Doc
egj.xkfrnc.cn/42884.Doc
egh.xkfrnc.cn/48460.Doc
egh.xkfrnc.cn/04222.Doc
egh.xkfrnc.cn/28020.Doc
egh.xkfrnc.cn/64608.Doc
egh.xkfrnc.cn/64044.Doc
egh.xkfrnc.cn/44848.Doc
egh.xkfrnc.cn/02668.Doc
egh.xkfrnc.cn/24440.Doc
egh.xkfrnc.cn/80886.Doc
egh.xkfrnc.cn/22664.Doc
egg.xkfrnc.cn/60640.Doc
egg.xkfrnc.cn/84420.Doc
egg.xkfrnc.cn/26244.Doc
egg.xkfrnc.cn/00282.Doc
egg.xkfrnc.cn/82824.Doc
egg.xkfrnc.cn/80860.Doc
egg.xkfrnc.cn/48482.Doc
egg.xkfrnc.cn/86808.Doc
egg.xkfrnc.cn/08026.Doc
egg.xkfrnc.cn/28620.Doc
egf.xkfrnc.cn/88286.Doc
egf.xkfrnc.cn/20626.Doc
egf.xkfrnc.cn/64486.Doc
egf.xkfrnc.cn/66460.Doc
egf.xkfrnc.cn/82046.Doc
egf.xkfrnc.cn/06842.Doc
egf.xkfrnc.cn/62682.Doc
egf.xkfrnc.cn/48686.Doc
egf.xkfrnc.cn/86062.Doc
egf.xkfrnc.cn/40084.Doc
egd.xkfrnc.cn/22264.Doc
egd.xkfrnc.cn/48426.Doc
egd.xkfrnc.cn/40806.Doc
egd.xkfrnc.cn/20248.Doc
egd.xkfrnc.cn/28266.Doc
egd.xkfrnc.cn/26086.Doc
egd.xkfrnc.cn/00864.Doc
egd.xkfrnc.cn/48886.Doc
egd.xkfrnc.cn/22260.Doc
egd.xkfrnc.cn/02422.Doc
egs.xkfrnc.cn/20282.Doc
egs.xkfrnc.cn/80864.Doc
egs.xkfrnc.cn/48628.Doc
egs.xkfrnc.cn/80624.Doc
egs.xkfrnc.cn/60606.Doc
egs.xkfrnc.cn/46202.Doc
egs.xkfrnc.cn/08484.Doc
egs.xkfrnc.cn/24084.Doc
egs.xkfrnc.cn/82220.Doc
egs.xkfrnc.cn/86248.Doc
ega.xkfrnc.cn/24806.Doc
ega.xkfrnc.cn/46666.Doc
ega.xkfrnc.cn/60828.Doc
ega.xkfrnc.cn/40882.Doc
ega.xkfrnc.cn/80666.Doc
ega.xkfrnc.cn/46464.Doc
ega.xkfrnc.cn/40604.Doc
ega.xkfrnc.cn/68088.Doc
ega.xkfrnc.cn/82080.Doc
ega.xkfrnc.cn/88208.Doc
egp.xkfrnc.cn/24428.Doc
egp.xkfrnc.cn/84048.Doc
egp.xkfrnc.cn/40880.Doc
egp.xkfrnc.cn/60204.Doc
egp.xkfrnc.cn/64800.Doc
egp.xkfrnc.cn/44044.Doc
egp.xkfrnc.cn/02424.Doc
egp.xkfrnc.cn/86842.Doc
egp.xkfrnc.cn/06060.Doc
egp.xkfrnc.cn/04826.Doc
ego.xkfrnc.cn/08808.Doc
ego.xkfrnc.cn/86660.Doc
ego.xkfrnc.cn/84220.Doc
ego.xkfrnc.cn/20464.Doc
ego.xkfrnc.cn/62402.Doc
ego.xkfrnc.cn/40860.Doc
ego.xkfrnc.cn/26468.Doc
ego.xkfrnc.cn/02842.Doc
ego.xkfrnc.cn/26248.Doc
ego.xkfrnc.cn/86404.Doc
egi.xkfrnc.cn/44826.Doc
egi.xkfrnc.cn/04604.Doc
egi.xkfrnc.cn/00646.Doc
egi.xkfrnc.cn/64046.Doc
egi.xkfrnc.cn/26202.Doc
egi.xkfrnc.cn/42240.Doc
egi.xkfrnc.cn/26260.Doc
egi.xkfrnc.cn/20608.Doc
egi.xkfrnc.cn/44868.Doc
egi.xkfrnc.cn/42622.Doc
egu.xkfrnc.cn/88422.Doc
egu.xkfrnc.cn/24660.Doc
egu.xkfrnc.cn/22820.Doc
egu.xkfrnc.cn/00426.Doc
egu.xkfrnc.cn/04820.Doc
egu.xkfrnc.cn/24886.Doc
egu.xkfrnc.cn/86024.Doc
egu.xkfrnc.cn/88080.Doc
egu.xkfrnc.cn/86688.Doc
egu.xkfrnc.cn/06022.Doc
egy.xkfrnc.cn/24866.Doc
egy.xkfrnc.cn/26444.Doc
egy.xkfrnc.cn/62646.Doc
egy.xkfrnc.cn/46288.Doc
egy.xkfrnc.cn/88420.Doc
egy.xkfrnc.cn/86464.Doc
egy.xkfrnc.cn/88022.Doc
egy.xkfrnc.cn/64062.Doc
egy.xkfrnc.cn/86806.Doc
egy.xkfrnc.cn/44024.Doc
egt.xkfrnc.cn/02844.Doc
egt.xkfrnc.cn/28446.Doc
egt.xkfrnc.cn/28444.Doc
egt.xkfrnc.cn/08684.Doc
egt.xkfrnc.cn/46840.Doc
egt.xkfrnc.cn/26666.Doc
egt.xkfrnc.cn/88480.Doc
egt.xkfrnc.cn/44660.Doc
egt.xkfrnc.cn/08288.Doc
egt.xkfrnc.cn/68402.Doc
egr.xkfrnc.cn/46864.Doc
egr.xkfrnc.cn/44000.Doc
egr.xkfrnc.cn/26246.Doc
egr.xkfrnc.cn/86482.Doc
egr.xkfrnc.cn/28800.Doc
egr.xkfrnc.cn/82640.Doc
egr.xkfrnc.cn/88426.Doc
egr.xkfrnc.cn/26808.Doc
egr.xkfrnc.cn/40040.Doc
egr.xkfrnc.cn/62482.Doc
ege.xkfrnc.cn/26046.Doc
ege.xkfrnc.cn/84842.Doc
ege.xkfrnc.cn/22448.Doc
ege.xkfrnc.cn/46048.Doc
ege.xkfrnc.cn/66640.Doc
ege.xkfrnc.cn/46842.Doc
ege.xkfrnc.cn/42800.Doc
ege.xkfrnc.cn/64024.Doc
ege.xkfrnc.cn/82044.Doc
ege.xkfrnc.cn/62024.Doc
egw.xkfrnc.cn/20086.Doc
egw.xkfrnc.cn/84204.Doc
egw.xkfrnc.cn/04862.Doc
egw.xkfrnc.cn/02266.Doc
egw.xkfrnc.cn/28664.Doc
egw.xkfrnc.cn/48026.Doc
egw.xkfrnc.cn/00008.Doc
egw.xkfrnc.cn/20608.Doc
egw.xkfrnc.cn/60426.Doc
egw.xkfrnc.cn/88202.Doc
egq.xkfrnc.cn/82448.Doc
egq.xkfrnc.cn/04626.Doc
egq.xkfrnc.cn/06202.Doc
egq.xkfrnc.cn/26680.Doc
egq.xkfrnc.cn/46028.Doc
egq.xkfrnc.cn/28622.Doc
egq.xkfrnc.cn/42824.Doc
egq.xkfrnc.cn/22688.Doc
egq.xkfrnc.cn/06020.Doc
egq.xkfrnc.cn/26424.Doc
efm.xkfrnc.cn/40628.Doc
efm.xkfrnc.cn/44008.Doc
efm.xkfrnc.cn/28006.Doc
efm.xkfrnc.cn/22804.Doc
efm.xkfrnc.cn/28826.Doc
efm.xkfrnc.cn/48440.Doc
efm.xkfrnc.cn/24606.Doc
efm.xkfrnc.cn/88468.Doc
efm.xkfrnc.cn/64280.Doc
efm.xkfrnc.cn/22448.Doc
efn.xkfrnc.cn/62286.Doc
efn.xkfrnc.cn/40682.Doc
efn.xkfrnc.cn/80822.Doc
efn.xkfrnc.cn/02222.Doc
efn.xkfrnc.cn/84824.Doc
efn.xkfrnc.cn/26484.Doc
efn.xkfrnc.cn/48686.Doc
efn.xkfrnc.cn/80602.Doc
efn.xkfrnc.cn/60446.Doc
efn.xkfrnc.cn/80624.Doc
efb.xkfrnc.cn/80008.Doc
efb.xkfrnc.cn/40486.Doc
efb.xkfrnc.cn/88648.Doc
efb.xkfrnc.cn/68884.Doc
efb.xkfrnc.cn/20408.Doc
efb.xkfrnc.cn/04086.Doc
efb.xkfrnc.cn/00462.Doc
efb.xkfrnc.cn/28202.Doc
efb.xkfrnc.cn/28244.Doc
efb.xkfrnc.cn/62604.Doc
efv.xkfrnc.cn/28066.Doc
efv.xkfrnc.cn/68822.Doc
efv.xkfrnc.cn/68028.Doc
efv.xkfrnc.cn/42068.Doc
efv.xkfrnc.cn/00484.Doc
efv.xkfrnc.cn/26240.Doc
efv.xkfrnc.cn/60020.Doc
efv.xkfrnc.cn/22046.Doc
efv.xkfrnc.cn/84826.Doc
efv.xkfrnc.cn/26866.Doc
efc.xkfrnc.cn/64846.Doc
efc.xkfrnc.cn/86268.Doc
efc.xkfrnc.cn/86040.Doc
efc.xkfrnc.cn/64820.Doc
efc.xkfrnc.cn/60262.Doc
efc.xkfrnc.cn/84222.Doc
efc.xkfrnc.cn/04886.Doc
efc.xkfrnc.cn/60680.Doc
efc.xkfrnc.cn/46026.Doc
efc.xkfrnc.cn/26208.Doc
efx.xkfrnc.cn/06826.Doc
efx.xkfrnc.cn/68468.Doc
efx.xkfrnc.cn/48084.Doc
efx.xkfrnc.cn/48048.Doc
efx.xkfrnc.cn/26484.Doc
efx.xkfrnc.cn/66626.Doc
efx.xkfrnc.cn/04228.Doc
efx.xkfrnc.cn/08600.Doc
efx.xkfrnc.cn/28684.Doc
efx.xkfrnc.cn/82664.Doc
efz.xkfrnc.cn/06482.Doc
efz.xkfrnc.cn/60545.Doc
efz.xkfrnc.cn/48806.Doc
efz.xkfrnc.cn/88246.Doc
efz.xkfrnc.cn/68086.Doc
efz.xkfrnc.cn/46820.Doc
efz.xkfrnc.cn/64262.Doc
efz.xkfrnc.cn/82640.Doc
efz.xkfrnc.cn/84820.Doc
efz.xkfrnc.cn/08606.Doc
efl.xkfrnc.cn/60488.Doc
efl.xkfrnc.cn/04866.Doc
efl.xkfrnc.cn/08682.Doc
efl.xkfrnc.cn/20420.Doc
efl.xkfrnc.cn/40868.Doc
efl.xkfrnc.cn/46026.Doc
efl.xkfrnc.cn/48400.Doc
efl.xkfrnc.cn/66626.Doc
efl.xkfrnc.cn/06862.Doc
efl.xkfrnc.cn/02880.Doc
efk.xkfrnc.cn/44680.Doc
efk.xkfrnc.cn/46886.Doc
efk.xkfrnc.cn/20648.Doc
efk.xkfrnc.cn/08888.Doc
efk.xkfrnc.cn/28684.Doc
efk.xkfrnc.cn/00440.Doc
efk.xkfrnc.cn/02086.Doc
efk.xkfrnc.cn/84622.Doc
efk.xkfrnc.cn/20024.Doc
efk.xkfrnc.cn/24848.Doc
efj.xkfrnc.cn/66820.Doc
efj.xkfrnc.cn/68060.Doc
efj.xkfrnc.cn/24228.Doc
efj.xkfrnc.cn/80208.Doc
efj.xkfrnc.cn/02824.Doc
efj.xkfrnc.cn/42464.Doc
efj.xkfrnc.cn/48486.Doc
efj.xkfrnc.cn/24080.Doc
efj.xkfrnc.cn/04046.Doc
efj.xkfrnc.cn/48420.Doc
efh.xkfrnc.cn/28682.Doc
efh.xkfrnc.cn/46822.Doc
efh.xkfrnc.cn/64624.Doc
efh.xkfrnc.cn/04086.Doc
efh.xkfrnc.cn/46084.Doc
efh.xkfrnc.cn/62286.Doc
efh.xkfrnc.cn/84428.Doc
efh.xkfrnc.cn/64464.Doc
efh.xkfrnc.cn/28086.Doc
efh.xkfrnc.cn/00688.Doc
efg.xkfrnc.cn/86040.Doc
efg.xkfrnc.cn/88442.Doc
efg.xkfrnc.cn/86822.Doc
efg.xkfrnc.cn/64628.Doc
efg.xkfrnc.cn/20480.Doc
efg.xkfrnc.cn/42824.Doc
efg.xkfrnc.cn/20628.Doc
efg.xkfrnc.cn/00200.Doc
efg.xkfrnc.cn/80026.Doc
efg.xkfrnc.cn/66440.Doc
eff.xkfrnc.cn/40028.Doc
eff.xkfrnc.cn/48662.Doc
eff.xkfrnc.cn/42268.Doc
eff.xkfrnc.cn/08284.Doc
eff.xkfrnc.cn/44008.Doc
eff.xkfrnc.cn/48080.Doc
eff.xkfrnc.cn/48402.Doc
eff.xkfrnc.cn/60884.Doc
eff.xkfrnc.cn/46202.Doc
eff.xkfrnc.cn/60288.Doc
efd.xkfrnc.cn/04682.Doc
efd.xkfrnc.cn/64444.Doc
efd.xkfrnc.cn/26488.Doc
efd.xkfrnc.cn/02206.Doc
efd.xkfrnc.cn/04608.Doc
efd.xkfrnc.cn/04484.Doc
efd.xkfrnc.cn/60806.Doc
efd.xkfrnc.cn/80082.Doc
efd.xkfrnc.cn/20806.Doc
efd.xkfrnc.cn/84426.Doc
