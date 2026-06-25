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

exm.wandk5.cn/64202.Doc
exm.wandk5.cn/62000.Doc
exm.wandk5.cn/28262.Doc
exm.wandk5.cn/66208.Doc
exm.wandk5.cn/02648.Doc
exm.wandk5.cn/82486.Doc
exm.wandk5.cn/28468.Doc
exm.wandk5.cn/22640.Doc
exm.wandk5.cn/22422.Doc
exm.wandk5.cn/26428.Doc
exn.wandk5.cn/26820.Doc
exn.wandk5.cn/84600.Doc
exn.wandk5.cn/15406.Doc
exn.wandk5.cn/46660.Doc
exn.wandk5.cn/68200.Doc
exn.wandk5.cn/26246.Doc
exn.wandk5.cn/44266.Doc
exn.wandk5.cn/44680.Doc
exn.wandk5.cn/02202.Doc
exn.wandk5.cn/04624.Doc
exb.wandk5.cn/24820.Doc
exb.wandk5.cn/08606.Doc
exb.wandk5.cn/02280.Doc
exb.wandk5.cn/46082.Doc
exb.wandk5.cn/13289.Doc
exb.wandk5.cn/26622.Doc
exb.wandk5.cn/26028.Doc
exb.wandk5.cn/82482.Doc
exb.wandk5.cn/88244.Doc
exb.wandk5.cn/26086.Doc
exv.wandk5.cn/84680.Doc
exv.wandk5.cn/60220.Doc
exv.wandk5.cn/86046.Doc
exv.wandk5.cn/64026.Doc
exv.wandk5.cn/28684.Doc
exv.wandk5.cn/28882.Doc
exv.wandk5.cn/68606.Doc
exv.wandk5.cn/80882.Doc
exv.wandk5.cn/42206.Doc
exv.wandk5.cn/88224.Doc
exc.wandk5.cn/64606.Doc
exc.wandk5.cn/66446.Doc
exc.wandk5.cn/46512.Doc
exc.wandk5.cn/60688.Doc
exc.wandk5.cn/46264.Doc
exc.wandk5.cn/68002.Doc
exc.wandk5.cn/04006.Doc
exc.wandk5.cn/60244.Doc
exc.wandk5.cn/49895.Doc
exc.wandk5.cn/40526.Doc
exx.wandk5.cn/80668.Doc
exx.wandk5.cn/44848.Doc
exx.wandk5.cn/84024.Doc
exx.wandk5.cn/22226.Doc
exx.wandk5.cn/64840.Doc
exx.wandk5.cn/39373.Doc
exx.wandk5.cn/82226.Doc
exx.wandk5.cn/88448.Doc
exx.wandk5.cn/46484.Doc
exx.wandk5.cn/80602.Doc
exz.wandk5.cn/06786.Doc
exz.wandk5.cn/40600.Doc
exz.wandk5.cn/26488.Doc
exz.wandk5.cn/00226.Doc
exz.wandk5.cn/22286.Doc
exz.wandk5.cn/04662.Doc
exz.wandk5.cn/86826.Doc
exz.wandk5.cn/08680.Doc
exz.wandk5.cn/40644.Doc
exz.wandk5.cn/26860.Doc
exl.wandk5.cn/04006.Doc
exl.wandk5.cn/44804.Doc
exl.wandk5.cn/60204.Doc
exl.wandk5.cn/40068.Doc
exl.wandk5.cn/88226.Doc
exl.wandk5.cn/62268.Doc
exl.wandk5.cn/44684.Doc
exl.wandk5.cn/60204.Doc
exl.wandk5.cn/66286.Doc
exl.wandk5.cn/24486.Doc
exk.wandk5.cn/40406.Doc
exk.wandk5.cn/20860.Doc
exk.wandk5.cn/02068.Doc
exk.wandk5.cn/40640.Doc
exk.wandk5.cn/00686.Doc
exk.wandk5.cn/88860.Doc
exk.wandk5.cn/20088.Doc
exk.wandk5.cn/88460.Doc
exk.wandk5.cn/06002.Doc
exk.wandk5.cn/84088.Doc
exj.wandk5.cn/08824.Doc
exj.wandk5.cn/24640.Doc
exj.wandk5.cn/64042.Doc
exj.wandk5.cn/02280.Doc
exj.wandk5.cn/66442.Doc
exj.wandk5.cn/46842.Doc
exj.wandk5.cn/06352.Doc
exj.wandk5.cn/48880.Doc
exj.wandk5.cn/86488.Doc
exj.wandk5.cn/02082.Doc
exh.wandk5.cn/26820.Doc
exh.wandk5.cn/64802.Doc
exh.wandk5.cn/80422.Doc
exh.wandk5.cn/20208.Doc
exh.wandk5.cn/64821.Doc
exh.wandk5.cn/46682.Doc
exh.wandk5.cn/28888.Doc
exh.wandk5.cn/28282.Doc
exh.wandk5.cn/42642.Doc
exh.wandk5.cn/80008.Doc
exg.wandk5.cn/80668.Doc
exg.wandk5.cn/00420.Doc
exg.wandk5.cn/06060.Doc
exg.wandk5.cn/46428.Doc
exg.wandk5.cn/40642.Doc
exg.wandk5.cn/04824.Doc
exg.wandk5.cn/82656.Doc
exg.wandk5.cn/68824.Doc
exg.wandk5.cn/24028.Doc
exg.wandk5.cn/80026.Doc
exf.wandk5.cn/02888.Doc
exf.wandk5.cn/60086.Doc
exf.wandk5.cn/06486.Doc
exf.wandk5.cn/20264.Doc
exf.wandk5.cn/24008.Doc
exf.wandk5.cn/00282.Doc
exf.wandk5.cn/33107.Doc
exf.wandk5.cn/86844.Doc
exf.wandk5.cn/82686.Doc
exf.wandk5.cn/35930.Doc
exd.wandk5.cn/42826.Doc
exd.wandk5.cn/26862.Doc
exd.wandk5.cn/64060.Doc
exd.wandk5.cn/86244.Doc
exd.wandk5.cn/66204.Doc
exd.wandk5.cn/88662.Doc
exd.wandk5.cn/80640.Doc
exd.wandk5.cn/22664.Doc
exd.wandk5.cn/66828.Doc
exd.wandk5.cn/66882.Doc
exs.wandk5.cn/64804.Doc
exs.wandk5.cn/60864.Doc
exs.wandk5.cn/22028.Doc
exs.wandk5.cn/82484.Doc
exs.wandk5.cn/86488.Doc
exs.wandk5.cn/22080.Doc
exs.wandk5.cn/06268.Doc
exs.wandk5.cn/22664.Doc
exs.wandk5.cn/68204.Doc
exs.wandk5.cn/04084.Doc
exa.wandk5.cn/22420.Doc
exa.wandk5.cn/88886.Doc
exa.wandk5.cn/46466.Doc
exa.wandk5.cn/02868.Doc
exa.wandk5.cn/08442.Doc
exa.wandk5.cn/24808.Doc
exa.wandk5.cn/82282.Doc
exa.wandk5.cn/06226.Doc
exa.wandk5.cn/84224.Doc
exa.wandk5.cn/68408.Doc
exp.wandk5.cn/00260.Doc
exp.wandk5.cn/68208.Doc
exp.wandk5.cn/48006.Doc
exp.wandk5.cn/88282.Doc
exp.wandk5.cn/84426.Doc
exp.wandk5.cn/64868.Doc
exp.wandk5.cn/40042.Doc
exp.wandk5.cn/40668.Doc
exp.wandk5.cn/80400.Doc
exp.wandk5.cn/46286.Doc
exo.wandk5.cn/40042.Doc
exo.wandk5.cn/88448.Doc
exo.wandk5.cn/86842.Doc
exo.wandk5.cn/62202.Doc
exo.wandk5.cn/24220.Doc
exo.wandk5.cn/64028.Doc
exo.wandk5.cn/68664.Doc
exo.wandk5.cn/80842.Doc
exo.wandk5.cn/24286.Doc
exo.wandk5.cn/80048.Doc
exi.wandk5.cn/20844.Doc
exi.wandk5.cn/44620.Doc
exi.wandk5.cn/66646.Doc
exi.wandk5.cn/04066.Doc
exi.wandk5.cn/42886.Doc
exi.wandk5.cn/82888.Doc
exi.wandk5.cn/68042.Doc
exi.wandk5.cn/80848.Doc
exi.wandk5.cn/06680.Doc
exi.wandk5.cn/80426.Doc
exu.wandk5.cn/69130.Doc
exu.wandk5.cn/88462.Doc
exu.wandk5.cn/42215.Doc
exu.wandk5.cn/86248.Doc
exu.wandk5.cn/28028.Doc
exu.wandk5.cn/40426.Doc
exu.wandk5.cn/28686.Doc
exu.wandk5.cn/20660.Doc
exu.wandk5.cn/66622.Doc
exu.wandk5.cn/44842.Doc
exy.wandk5.cn/44262.Doc
exy.wandk5.cn/20402.Doc
exy.wandk5.cn/08222.Doc
exy.wandk5.cn/28464.Doc
exy.wandk5.cn/48464.Doc
exy.wandk5.cn/48668.Doc
exy.wandk5.cn/00060.Doc
exy.wandk5.cn/26282.Doc
exy.wandk5.cn/25934.Doc
exy.wandk5.cn/48226.Doc
ext.wandk5.cn/06642.Doc
ext.wandk5.cn/88248.Doc
ext.wandk5.cn/60600.Doc
ext.wandk5.cn/84428.Doc
ext.wandk5.cn/71542.Doc
ext.wandk5.cn/80200.Doc
ext.wandk5.cn/28224.Doc
ext.wandk5.cn/86806.Doc
ext.wandk5.cn/28208.Doc
ext.wandk5.cn/82848.Doc
exr.wandk5.cn/64420.Doc
exr.wandk5.cn/60004.Doc
exr.wandk5.cn/88800.Doc
exr.wandk5.cn/88046.Doc
exr.wandk5.cn/80000.Doc
exr.wandk5.cn/48260.Doc
exr.wandk5.cn/68286.Doc
exr.wandk5.cn/24662.Doc
exr.wandk5.cn/86428.Doc
exr.wandk5.cn/48660.Doc
exe.wandk5.cn/40828.Doc
exe.wandk5.cn/48488.Doc
exe.wandk5.cn/06684.Doc
exe.wandk5.cn/04866.Doc
exe.wandk5.cn/44486.Doc
exe.wandk5.cn/28682.Doc
exe.wandk5.cn/08802.Doc
exe.wandk5.cn/88842.Doc
exe.wandk5.cn/48444.Doc
exe.wandk5.cn/82262.Doc
exw.wandk5.cn/04880.Doc
exw.wandk5.cn/66004.Doc
exw.wandk5.cn/24028.Doc
exw.wandk5.cn/60686.Doc
exw.wandk5.cn/02602.Doc
exw.wandk5.cn/06684.Doc
exw.wandk5.cn/82088.Doc
exw.wandk5.cn/82002.Doc
exw.wandk5.cn/22464.Doc
exw.wandk5.cn/06648.Doc
exq.wandk5.cn/60688.Doc
exq.wandk5.cn/20246.Doc
exq.wandk5.cn/00424.Doc
exq.wandk5.cn/04808.Doc
exq.wandk5.cn/04840.Doc
exq.wandk5.cn/62480.Doc
exq.wandk5.cn/46040.Doc
exq.wandk5.cn/44260.Doc
exq.wandk5.cn/40628.Doc
exq.wandk5.cn/20028.Doc
ezm.wandk5.cn/82684.Doc
ezm.wandk5.cn/88260.Doc
ezm.wandk5.cn/08020.Doc
ezm.wandk5.cn/60460.Doc
ezm.wandk5.cn/80822.Doc
ezm.wandk5.cn/82668.Doc
ezm.wandk5.cn/40060.Doc
ezm.wandk5.cn/62826.Doc
ezm.wandk5.cn/66484.Doc
ezm.wandk5.cn/86024.Doc
ezn.wandk5.cn/28888.Doc
ezn.wandk5.cn/60862.Doc
ezn.wandk5.cn/04042.Doc
ezn.wandk5.cn/68886.Doc
ezn.wandk5.cn/88846.Doc
ezn.wandk5.cn/42688.Doc
ezn.wandk5.cn/64084.Doc
ezn.wandk5.cn/42802.Doc
ezn.wandk5.cn/01017.Doc
ezn.wandk5.cn/48624.Doc
ezb.wandk5.cn/60408.Doc
ezb.wandk5.cn/64282.Doc
ezb.wandk5.cn/63558.Doc
ezb.wandk5.cn/24680.Doc
ezb.wandk5.cn/82420.Doc
ezb.wandk5.cn/80002.Doc
ezb.wandk5.cn/40086.Doc
ezb.wandk5.cn/20484.Doc
ezb.wandk5.cn/20422.Doc
ezb.wandk5.cn/40886.Doc
ezv.wandk5.cn/69840.Doc
ezv.wandk5.cn/02248.Doc
ezv.wandk5.cn/24464.Doc
ezv.wandk5.cn/22206.Doc
ezv.wandk5.cn/20444.Doc
ezv.wandk5.cn/22024.Doc
ezv.wandk5.cn/44206.Doc
ezv.wandk5.cn/11390.Doc
ezv.wandk5.cn/22646.Doc
ezv.wandk5.cn/02604.Doc
ezc.wandk5.cn/84266.Doc
ezc.wandk5.cn/66844.Doc
ezc.wandk5.cn/06202.Doc
ezc.wandk5.cn/84066.Doc
ezc.wandk5.cn/86484.Doc
ezc.wandk5.cn/44844.Doc
ezc.wandk5.cn/62668.Doc
ezc.wandk5.cn/88046.Doc
ezc.wandk5.cn/86066.Doc
ezc.wandk5.cn/02060.Doc
ezx.wandk5.cn/06624.Doc
ezx.wandk5.cn/26206.Doc
ezx.wandk5.cn/02428.Doc
ezx.wandk5.cn/11129.Doc
ezx.wandk5.cn/06420.Doc
ezx.wandk5.cn/24826.Doc
ezx.wandk5.cn/40466.Doc
ezx.wandk5.cn/65289.Doc
ezx.wandk5.cn/66082.Doc
ezx.wandk5.cn/82228.Doc
ezz.wandk5.cn/02220.Doc
ezz.wandk5.cn/46442.Doc
ezz.wandk5.cn/08820.Doc
ezz.wandk5.cn/20088.Doc
ezz.wandk5.cn/04464.Doc
ezz.wandk5.cn/60006.Doc
ezz.wandk5.cn/02260.Doc
ezz.wandk5.cn/00040.Doc
ezz.wandk5.cn/28406.Doc
ezz.wandk5.cn/48602.Doc
ezl.wandk5.cn/46808.Doc
ezl.wandk5.cn/80262.Doc
ezl.wandk5.cn/85406.Doc
ezl.wandk5.cn/66046.Doc
ezl.wandk5.cn/82260.Doc
ezl.wandk5.cn/28040.Doc
ezl.wandk5.cn/40008.Doc
ezl.wandk5.cn/04082.Doc
ezl.wandk5.cn/20040.Doc
ezl.wandk5.cn/24246.Doc
ezk.wandk5.cn/88448.Doc
ezk.wandk5.cn/06860.Doc
ezk.wandk5.cn/62422.Doc
ezk.wandk5.cn/08804.Doc
ezk.wandk5.cn/46868.Doc
ezk.wandk5.cn/00288.Doc
ezk.wandk5.cn/84286.Doc
ezk.wandk5.cn/02400.Doc
ezk.wandk5.cn/40666.Doc
ezk.wandk5.cn/80004.Doc
ezj.wandk5.cn/20408.Doc
ezj.wandk5.cn/62626.Doc
ezj.wandk5.cn/88220.Doc
ezj.wandk5.cn/02224.Doc
ezj.wandk5.cn/62888.Doc
ezj.wandk5.cn/66644.Doc
ezj.wandk5.cn/68842.Doc
ezj.wandk5.cn/64602.Doc
ezj.wandk5.cn/48062.Doc
ezj.wandk5.cn/02086.Doc
ezh.wandk5.cn/82648.Doc
ezh.wandk5.cn/82800.Doc
ezh.wandk5.cn/40620.Doc
ezh.wandk5.cn/46244.Doc
ezh.wandk5.cn/82800.Doc
ezh.wandk5.cn/42246.Doc
ezh.wandk5.cn/04868.Doc
ezh.wandk5.cn/22042.Doc
ezh.wandk5.cn/48060.Doc
ezh.wandk5.cn/26682.Doc
ezg.wandk5.cn/42822.Doc
ezg.wandk5.cn/42646.Doc
ezg.wandk5.cn/26422.Doc
ezg.wandk5.cn/48466.Doc
ezg.wandk5.cn/86802.Doc
ezg.wandk5.cn/68886.Doc
ezg.wandk5.cn/84800.Doc
ezg.wandk5.cn/85732.Doc
ezg.wandk5.cn/42420.Doc
ezg.wandk5.cn/66648.Doc
ezf.wandk5.cn/44246.Doc
ezf.wandk5.cn/40046.Doc
ezf.wandk5.cn/62802.Doc
ezf.wandk5.cn/60240.Doc
ezf.wandk5.cn/40244.Doc
ezf.wandk5.cn/28860.Doc
ezf.wandk5.cn/40462.Doc
ezf.wandk5.cn/84866.Doc
ezf.wandk5.cn/88222.Doc
ezf.wandk5.cn/28642.Doc
ezd.wandk5.cn/46808.Doc
ezd.wandk5.cn/82248.Doc
ezd.wandk5.cn/46266.Doc
ezd.wandk5.cn/88086.Doc
ezd.wandk5.cn/26886.Doc
ezd.wandk5.cn/68660.Doc
ezd.wandk5.cn/68426.Doc
ezd.wandk5.cn/22226.Doc
ezd.wandk5.cn/48660.Doc
ezd.wandk5.cn/60040.Doc
ezs.wandk5.cn/06226.Doc
ezs.wandk5.cn/84802.Doc
ezs.wandk5.cn/06220.Doc
ezs.wandk5.cn/20482.Doc
ezs.wandk5.cn/44448.Doc
ezs.wandk5.cn/40428.Doc
ezs.wandk5.cn/22842.Doc
ezs.wandk5.cn/86880.Doc
ezs.wandk5.cn/84042.Doc
ezs.wandk5.cn/44020.Doc
eza.wandk5.cn/68640.Doc
eza.wandk5.cn/40222.Doc
eza.wandk5.cn/64022.Doc
eza.wandk5.cn/40402.Doc
eza.wandk5.cn/08022.Doc
eza.wandk5.cn/06624.Doc
eza.wandk5.cn/82406.Doc
eza.wandk5.cn/88440.Doc
eza.wandk5.cn/22462.Doc
eza.wandk5.cn/08224.Doc
ezp.wandk5.cn/00882.Doc
ezp.wandk5.cn/66082.Doc
ezp.wandk5.cn/02248.Doc
ezp.wandk5.cn/24084.Doc
ezp.wandk5.cn/04046.Doc
ezp.wandk5.cn/24242.Doc
ezp.wandk5.cn/46426.Doc
ezp.wandk5.cn/22480.Doc
ezp.wandk5.cn/80222.Doc
ezp.wandk5.cn/06260.Doc
ezo.wandk5.cn/06226.Doc
ezo.wandk5.cn/84666.Doc
ezo.wandk5.cn/02651.Doc
ezo.wandk5.cn/48442.Doc
ezo.wandk5.cn/62684.Doc
ezo.wandk5.cn/22288.Doc
ezo.wandk5.cn/24860.Doc
ezo.wandk5.cn/40002.Doc
ezo.wandk5.cn/04242.Doc
ezo.wandk5.cn/64446.Doc
ezi.wandk5.cn/46646.Doc
ezi.wandk5.cn/42480.Doc
ezi.wandk5.cn/24086.Doc
ezi.wandk5.cn/62442.Doc
ezi.wandk5.cn/22662.Doc
ezi.wandk5.cn/00088.Doc
ezi.wandk5.cn/62684.Doc
ezi.wandk5.cn/66682.Doc
ezi.wandk5.cn/88448.Doc
ezi.wandk5.cn/00408.Doc
ezu.wandk5.cn/44808.Doc
ezu.wandk5.cn/48826.Doc
ezu.wandk5.cn/84468.Doc
ezu.wandk5.cn/64028.Doc
ezu.wandk5.cn/22220.Doc
ezu.wandk5.cn/26048.Doc
ezu.wandk5.cn/72173.Doc
ezu.wandk5.cn/00802.Doc
ezu.wandk5.cn/88884.Doc
ezu.wandk5.cn/02022.Doc
ezy.wandk5.cn/66840.Doc
ezy.wandk5.cn/48482.Doc
ezy.wandk5.cn/26620.Doc
ezy.wandk5.cn/00828.Doc
ezy.wandk5.cn/68666.Doc
ezy.wandk5.cn/68420.Doc
ezy.wandk5.cn/08002.Doc
ezy.wandk5.cn/06048.Doc
ezy.wandk5.cn/60288.Doc
ezy.wandk5.cn/42802.Doc
ezt.wandk5.cn/88262.Doc
ezt.wandk5.cn/82604.Doc
ezt.wandk5.cn/02426.Doc
ezt.wandk5.cn/02446.Doc
ezt.wandk5.cn/64220.Doc
ezt.wandk5.cn/42842.Doc
ezt.wandk5.cn/28260.Doc
ezt.wandk5.cn/48828.Doc
ezt.wandk5.cn/84824.Doc
ezt.wandk5.cn/06682.Doc
ezr.wandk5.cn/22420.Doc
ezr.wandk5.cn/40046.Doc
ezr.wandk5.cn/68808.Doc
ezr.wandk5.cn/48008.Doc
ezr.wandk5.cn/00886.Doc
ezr.wandk5.cn/60644.Doc
ezr.wandk5.cn/42868.Doc
ezr.wandk5.cn/64048.Doc
ezr.wandk5.cn/04242.Doc
ezr.wandk5.cn/80884.Doc
