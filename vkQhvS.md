仆惫肪豆沃


Python SOLID 原则实践
============================

SOLID 是面向对象设计的五大原则，能让代码更易维护、扩展和测试。
本文用 Python 示例逐一讲解。

1. 单一职责原则 (Single Responsibility Principle)
---------------------------------------------------
一个类应该只有一个引起变化的原因，即只负责一项职责。

class ReportGenerator:
    """只负责生成报告——单一职责"""
    def generate(self, data: dict) -> str:
        # 将数据格式化为报告字符串
        report_lines = [f"报告标题: {data.get('title', '无标题')}"]
        report_lines.append(f"作者: {data.get('author', '未知')}")
        for key, value in data.get('items', {}).items():
            report_lines.append(f"  {key}: {value}")
        return "\n".join(report_lines)


class ReportSaver:
    """只负责保存报告——单一职责，与 ReportGenerator 分离"""
    def save(self, report: str, path: str) -> None:
        with open(path, 'w', encoding='utf-8') as f:
            f.write(report)


# 错误示范：一个类既生成又保存又发送，三个变化原因
class BadReport:
    def generate(self, data): ...
    def save(self, path): ...
    def send_email(self, recipient): ...


2. 开闭原则 (Open/Closed Principle)
-------------------------------------
对扩展开放，对修改关闭。通过继承或组合增加新行为。

from abc import ABC, abstractmethod

class DiscountStrategy(ABC):
    """折扣策略抽象基类——对扩展开放"""
    @abstractmethod
    def apply(self, price: float) -> float: ...


class NoDiscount(DiscountStrategy):
    def apply(self, price: float) -> float:
        return price


class PercentageDiscount(DiscountStrategy):
    def __init__(self, percent: float):
        self.percent = percent

    def apply(self, price: float) -> float:
        return price * (1 - self.percent / 100)


class Order:
    """订单类——对修改关闭，不需要因为新折扣类型而修改"""
    def __init__(self, price: float, strategy: DiscountStrategy):
        self.price = price
        self.strategy = strategy

    def final_price(self) -> float:
        return self.strategy.apply(self.price)


# 新增会员折扣无需修改 Order——只需新增策略类
class MemberDiscount(DiscountStrategy):
    def __init__(self, level: str):
        self.level = level

    def apply(self, price: float) -> float:
        ratios = {'gold': 0.7, 'silver': 0.85, 'bronze': 0.95}
        return price * ratios.get(self.level, 1.0)


3. 里氏替换原则 (Liskov Substitution Principle)
-------------------------------------------------
子类必须能替换父类而不破坏程序正确性。

class Rectangle:
    """矩形基类"""
    def __init__(self, width: int, height: int):
        self.width = width
        self.height = height

    def area(self) -> int:
        return self.width * self.height


class Square(Rectangle):
    """正方形——违反 LSP！设 width 会改变 height，反之亦然"""
    def __init__(self, side: int):
        super().__init__(side, side)

    # 问题：Square 重设 width 也会改变 height，违反父类契约
    @property
    def width(self):
        return self._width

    @width.setter
    def width(self, value):
        self._width = value
        self._height = value  # 副作用！违反 LSP

    @property
    def height(self):
        return self._height

    @height.setter
    def height(self, value):
        self._height = value
        self._width = value  # 副作用！违反 LSP


# 更好的做法：使用独立接口，不强制继承关系
class Shape(ABC):
    @abstractmethod
    def area(self) -> int: ...


class ProperSquare(Shape):
    """不继承 Rectangle，直接实现 Shape，避免 LSP 违规"""
    def __init__(self, side: int):
        self.side = side

    def area(self) -> int:
        return self.side * self.side


4. 接口隔离原则 (Interface Segregation Principle)
---------------------------------------------------
客户端不应被迫依赖它们不使用的接口。用小型专注的协议替代庞大接口。

from typing import Protocol

class Printer(Protocol):
    """小型专注协议：只负责打印"""
    def print_document(self, doc: str) -> None: ...


class Scanner(Protocol):
    """小型专注协议：只负责扫描"""
    def scan_document(self) -> str: ...


class SimplePrinter:
    """只实现了 Printer，不需要扫描功能"""
    def print_document(self, doc: str) -> None:
        print(f"打印: {doc}")


class MultiFunctionDevice:
    """需要多个功能时才组合多个协议"""
    def print_document(self, doc: str) -> None:
        print(f"打印: {doc}")

    def scan_document(self) -> str:
        return "扫描结果"


5. 依赖反转原则 (Dependency Inversion Principle)
---------------------------------------------------
高层模块不应依赖低层模块，二者都应依赖抽象。

class EmailSender(ABC):
    """抽象：发送器接口"""
    @abstractmethod
    def send(self, message: str, to: str) -> None: ...


class SmtpEmailSender(EmailSender):
    """低层模块：具体实现"""
    def send(self, message: str, to: str) -> None:
        print(f"通过 SMTP 发送 '{message}' 到 {to}")


class NotificationService:
    """高层模块：依赖抽象而非具体实现"""
    def __init__(self, sender: EmailSender):  # 注入抽象
        self.sender = sender

    def notify(self, user_email: str, msg: str) -> None:
        self.sender.send(msg, user_email)


# 使用依赖注入，轻松切换实现
smtp = SmtpEmailSender()
service = NotificationService(smtp)
service.notify("alice@example.com", "订单已确认")


总结：SOLID 原则指导我们写出高内聚低耦合的代码。单一职责让类更聚焦，
开闭原则让扩展更容易，里氏替换保证继承安全，接口隔离避免臃肿接口，
依赖反转降低模块间耦合。实践中不必教条，合理运用即可。

衬谱艘反鞠现聪酚碧壁干椒屹督褪

m.eei.cccnt.cn/97315.Doc
m.eei.cccnt.cn/48402.Doc
m.eei.cccnt.cn/91139.Doc
m.eei.cccnt.cn/37715.Doc
m.eei.cccnt.cn/46000.Doc
m.eei.cccnt.cn/59755.Doc
m.eei.cccnt.cn/44286.Doc
m.eei.cccnt.cn/19953.Doc
m.eei.cccnt.cn/11395.Doc
m.eei.cccnt.cn/15795.Doc
m.eei.cccnt.cn/95531.Doc
m.eei.cccnt.cn/53993.Doc
m.eei.cccnt.cn/44824.Doc
m.eei.cccnt.cn/31753.Doc
m.eei.cccnt.cn/28424.Doc
m.eeu.cccnt.cn/31757.Doc
m.eeu.cccnt.cn/33597.Doc
m.eeu.cccnt.cn/11371.Doc
m.eeu.cccnt.cn/31137.Doc
m.eeu.cccnt.cn/84240.Doc
m.eeu.cccnt.cn/95717.Doc
m.eeu.cccnt.cn/42446.Doc
m.eeu.cccnt.cn/15135.Doc
m.eeu.cccnt.cn/40024.Doc
m.eeu.cccnt.cn/79571.Doc
m.eeu.cccnt.cn/77775.Doc
m.eeu.cccnt.cn/15137.Doc
m.eeu.cccnt.cn/91935.Doc
m.eeu.cccnt.cn/88226.Doc
m.eeu.cccnt.cn/84266.Doc
m.eeu.cccnt.cn/28200.Doc
m.eeu.cccnt.cn/04864.Doc
m.eeu.cccnt.cn/19317.Doc
m.eeu.cccnt.cn/31195.Doc
m.eeu.cccnt.cn/42262.Doc
m.eey.cccnt.cn/53999.Doc
m.eey.cccnt.cn/68044.Doc
m.eey.cccnt.cn/64882.Doc
m.eey.cccnt.cn/11917.Doc
m.eey.cccnt.cn/99933.Doc
m.eey.cccnt.cn/19971.Doc
m.eey.cccnt.cn/19737.Doc
m.eey.cccnt.cn/37315.Doc
m.eey.cccnt.cn/02860.Doc
m.eey.cccnt.cn/71313.Doc
m.eey.cccnt.cn/57191.Doc
m.eey.cccnt.cn/75951.Doc
m.eey.cccnt.cn/77357.Doc
m.eey.cccnt.cn/75535.Doc
m.eey.cccnt.cn/79151.Doc
m.eey.cccnt.cn/79771.Doc
m.eey.cccnt.cn/13799.Doc
m.eey.cccnt.cn/35717.Doc
m.eey.cccnt.cn/31915.Doc
m.eey.cccnt.cn/04084.Doc
m.eet.cccnt.cn/44080.Doc
m.eet.cccnt.cn/62824.Doc
m.eet.cccnt.cn/55955.Doc
m.eet.cccnt.cn/73799.Doc
m.eet.cccnt.cn/13159.Doc
m.eet.cccnt.cn/66608.Doc
m.eet.cccnt.cn/08044.Doc
m.eet.cccnt.cn/15571.Doc
m.eet.cccnt.cn/73715.Doc
m.eet.cccnt.cn/86866.Doc
m.eet.cccnt.cn/91351.Doc
m.eet.cccnt.cn/93771.Doc
m.eet.cccnt.cn/46022.Doc
m.eet.cccnt.cn/31919.Doc
m.eet.cccnt.cn/44660.Doc
m.eet.cccnt.cn/60202.Doc
m.eet.cccnt.cn/24864.Doc
m.eet.cccnt.cn/71911.Doc
m.eet.cccnt.cn/11791.Doc
m.eet.cccnt.cn/39115.Doc
m.eer.cccnt.cn/39557.Doc
m.eer.cccnt.cn/68442.Doc
m.eer.cccnt.cn/39391.Doc
m.eer.cccnt.cn/80428.Doc
m.eer.cccnt.cn/77351.Doc
m.eer.cccnt.cn/22420.Doc
m.eer.cccnt.cn/19973.Doc
m.eer.cccnt.cn/93353.Doc
m.eer.cccnt.cn/17717.Doc
m.eer.cccnt.cn/13517.Doc
m.eer.cccnt.cn/95597.Doc
m.eer.cccnt.cn/37535.Doc
m.eer.cccnt.cn/31135.Doc
m.eer.cccnt.cn/48460.Doc
m.eer.cccnt.cn/77933.Doc
m.eer.cccnt.cn/79993.Doc
m.eer.cccnt.cn/99117.Doc
m.eer.cccnt.cn/84600.Doc
m.eer.cccnt.cn/46280.Doc
m.eer.cccnt.cn/95131.Doc
m.eee.cccnt.cn/20462.Doc
m.eee.cccnt.cn/46264.Doc
m.eee.cccnt.cn/60464.Doc
m.eee.cccnt.cn/19955.Doc
m.eee.cccnt.cn/33973.Doc
m.eee.cccnt.cn/73311.Doc
m.eee.cccnt.cn/73779.Doc
m.eee.cccnt.cn/77717.Doc
m.eee.cccnt.cn/99737.Doc
m.eee.cccnt.cn/53173.Doc
m.eee.cccnt.cn/75959.Doc
m.eee.cccnt.cn/15951.Doc
m.eee.cccnt.cn/75193.Doc
m.eee.cccnt.cn/19995.Doc
m.eee.cccnt.cn/44288.Doc
m.eee.cccnt.cn/40842.Doc
m.eee.cccnt.cn/51113.Doc
m.eee.cccnt.cn/26086.Doc
m.eee.cccnt.cn/84882.Doc
m.eee.cccnt.cn/75913.Doc
m.eew.cccnt.cn/86002.Doc
m.eew.cccnt.cn/19797.Doc
m.eew.cccnt.cn/46400.Doc
m.eew.cccnt.cn/77173.Doc
m.eew.cccnt.cn/51793.Doc
m.eew.cccnt.cn/15191.Doc
m.eew.cccnt.cn/66046.Doc
m.eew.cccnt.cn/26084.Doc
m.eew.cccnt.cn/82882.Doc
m.eew.cccnt.cn/53535.Doc
m.eew.cccnt.cn/77797.Doc
m.eew.cccnt.cn/08862.Doc
m.eew.cccnt.cn/97711.Doc
m.eew.cccnt.cn/13175.Doc
m.eew.cccnt.cn/77797.Doc
m.eew.cccnt.cn/79795.Doc
m.eew.cccnt.cn/11339.Doc
m.eew.cccnt.cn/33511.Doc
m.eew.cccnt.cn/57731.Doc
m.eew.cccnt.cn/15395.Doc
m.eeq.cccnt.cn/62026.Doc
m.eeq.cccnt.cn/97553.Doc
m.eeq.cccnt.cn/84024.Doc
m.eeq.cccnt.cn/55117.Doc
m.eeq.cccnt.cn/02284.Doc
m.eeq.cccnt.cn/82284.Doc
m.eeq.cccnt.cn/13517.Doc
m.eeq.cccnt.cn/62646.Doc
m.eeq.cccnt.cn/53911.Doc
m.eeq.cccnt.cn/37555.Doc
m.eeq.cccnt.cn/17933.Doc
m.eeq.cccnt.cn/77751.Doc
m.eeq.cccnt.cn/15917.Doc
m.eeq.cccnt.cn/71313.Doc
m.eeq.cccnt.cn/88062.Doc
m.eeq.cccnt.cn/59353.Doc
m.eeq.cccnt.cn/40006.Doc
m.eeq.cccnt.cn/22022.Doc
m.eeq.cccnt.cn/06224.Doc
m.eeq.cccnt.cn/64464.Doc
m.ewm.cccnt.cn/06428.Doc
m.ewm.cccnt.cn/20002.Doc
m.ewm.cccnt.cn/04806.Doc
m.ewm.cccnt.cn/77517.Doc
m.ewm.cccnt.cn/80004.Doc
m.ewm.cccnt.cn/91539.Doc
m.ewm.cccnt.cn/57591.Doc
m.ewm.cccnt.cn/11931.Doc
m.ewm.cccnt.cn/17135.Doc
m.ewm.cccnt.cn/40824.Doc
m.ewm.cccnt.cn/95513.Doc
m.ewm.cccnt.cn/11195.Doc
m.ewm.cccnt.cn/59379.Doc
m.ewm.cccnt.cn/15377.Doc
m.ewm.cccnt.cn/11713.Doc
m.ewm.cccnt.cn/68668.Doc
m.ewm.cccnt.cn/51319.Doc
m.ewm.cccnt.cn/88028.Doc
m.ewm.cccnt.cn/57373.Doc
m.ewm.cccnt.cn/02864.Doc
m.ewn.cccnt.cn/59155.Doc
m.ewn.cccnt.cn/59791.Doc
m.ewn.cccnt.cn/51771.Doc
m.ewn.cccnt.cn/19933.Doc
m.ewn.cccnt.cn/59331.Doc
m.ewn.cccnt.cn/75997.Doc
m.ewn.cccnt.cn/39779.Doc
m.ewn.cccnt.cn/11539.Doc
m.ewn.cccnt.cn/15531.Doc
m.ewn.cccnt.cn/73753.Doc
m.ewn.cccnt.cn/00224.Doc
m.ewn.cccnt.cn/57915.Doc
m.ewn.cccnt.cn/00226.Doc
m.ewn.cccnt.cn/91157.Doc
m.ewn.cccnt.cn/35591.Doc
m.ewn.cccnt.cn/31777.Doc
m.ewn.cccnt.cn/11591.Doc
m.ewn.cccnt.cn/04288.Doc
m.ewn.cccnt.cn/13555.Doc
m.ewn.cccnt.cn/40800.Doc
m.ewb.cccnt.cn/19791.Doc
m.ewb.cccnt.cn/13535.Doc
m.ewb.cccnt.cn/06664.Doc
m.ewb.cccnt.cn/88880.Doc
m.ewb.cccnt.cn/60660.Doc
m.ewb.cccnt.cn/08244.Doc
m.ewb.cccnt.cn/97171.Doc
m.ewb.cccnt.cn/88682.Doc
m.ewb.cccnt.cn/91777.Doc
m.ewb.cccnt.cn/15395.Doc
m.ewb.cccnt.cn/33173.Doc
m.ewb.cccnt.cn/55939.Doc
m.ewb.cccnt.cn/99535.Doc
m.ewb.cccnt.cn/04842.Doc
m.ewb.cccnt.cn/97355.Doc
m.ewb.cccnt.cn/59973.Doc
m.ewb.cccnt.cn/20002.Doc
m.ewb.cccnt.cn/24680.Doc
m.ewb.cccnt.cn/33555.Doc
m.ewb.cccnt.cn/64086.Doc
m.ewv.cccnt.cn/37539.Doc
m.ewv.cccnt.cn/73193.Doc
m.ewv.cccnt.cn/59913.Doc
m.ewv.cccnt.cn/40220.Doc
m.ewv.cccnt.cn/64048.Doc
m.ewv.cccnt.cn/71131.Doc
m.ewv.cccnt.cn/04646.Doc
m.ewv.cccnt.cn/33339.Doc
m.ewv.cccnt.cn/28880.Doc
m.ewv.cccnt.cn/51757.Doc
m.ewv.cccnt.cn/11539.Doc
m.ewv.cccnt.cn/53335.Doc
m.ewv.cccnt.cn/59575.Doc
m.ewv.cccnt.cn/35193.Doc
m.ewv.cccnt.cn/73777.Doc
m.ewv.cccnt.cn/26266.Doc
m.ewv.cccnt.cn/53773.Doc
m.ewv.cccnt.cn/31399.Doc
m.ewv.cccnt.cn/73397.Doc
m.ewv.cccnt.cn/26686.Doc
m.ewc.cccnt.cn/99951.Doc
m.ewc.cccnt.cn/02402.Doc
m.ewc.cccnt.cn/42080.Doc
m.ewc.cccnt.cn/00628.Doc
m.ewc.cccnt.cn/33797.Doc
m.ewc.cccnt.cn/08820.Doc
m.ewc.cccnt.cn/68242.Doc
m.ewc.cccnt.cn/13575.Doc
m.ewc.cccnt.cn/77779.Doc
m.ewc.cccnt.cn/79573.Doc
m.ewc.cccnt.cn/53393.Doc
m.ewc.cccnt.cn/91353.Doc
m.ewc.cccnt.cn/77137.Doc
m.ewc.cccnt.cn/53171.Doc
m.ewc.cccnt.cn/11335.Doc
m.ewc.cccnt.cn/95533.Doc
m.ewc.cccnt.cn/59593.Doc
m.ewc.cccnt.cn/53739.Doc
m.ewc.cccnt.cn/20804.Doc
m.ewc.cccnt.cn/31151.Doc
m.ewx.cccnt.cn/39755.Doc
m.ewx.cccnt.cn/15951.Doc
m.ewx.cccnt.cn/99773.Doc
m.ewx.cccnt.cn/39331.Doc
m.ewx.cccnt.cn/00424.Doc
m.ewx.cccnt.cn/31977.Doc
m.ewx.cccnt.cn/48800.Doc
m.ewx.cccnt.cn/73977.Doc
m.ewx.cccnt.cn/71395.Doc
m.ewx.cccnt.cn/88064.Doc
m.ewx.cccnt.cn/84666.Doc
m.ewx.cccnt.cn/80086.Doc
m.ewx.cccnt.cn/08442.Doc
m.ewx.cccnt.cn/88486.Doc
m.ewx.cccnt.cn/39339.Doc
m.ewx.cccnt.cn/95393.Doc
m.ewx.cccnt.cn/75353.Doc
m.ewx.cccnt.cn/19537.Doc
m.ewx.cccnt.cn/53355.Doc
m.ewx.cccnt.cn/71971.Doc
m.ewz.cccnt.cn/46288.Doc
m.ewz.cccnt.cn/13773.Doc
m.ewz.cccnt.cn/28026.Doc
m.ewz.cccnt.cn/51375.Doc
m.ewz.cccnt.cn/64624.Doc
m.ewz.cccnt.cn/75113.Doc
m.ewz.cccnt.cn/79339.Doc
m.ewz.cccnt.cn/55797.Doc
m.ewz.cccnt.cn/95719.Doc
m.ewz.cccnt.cn/26444.Doc
m.ewz.cccnt.cn/33117.Doc
m.ewz.cccnt.cn/44260.Doc
m.ewz.cccnt.cn/33135.Doc
m.ewz.cccnt.cn/35559.Doc
m.ewz.cccnt.cn/11751.Doc
m.ewz.cccnt.cn/62440.Doc
m.ewz.cccnt.cn/57957.Doc
m.ewz.cccnt.cn/64200.Doc
m.ewz.cccnt.cn/95995.Doc
m.ewz.cccnt.cn/33939.Doc
m.ewl.cccnt.cn/11331.Doc
m.ewl.cccnt.cn/73197.Doc
m.ewl.cccnt.cn/77795.Doc
m.ewl.cccnt.cn/33951.Doc
m.ewl.cccnt.cn/35575.Doc
m.ewl.cccnt.cn/97173.Doc
m.ewl.cccnt.cn/17531.Doc
m.ewl.cccnt.cn/40002.Doc
m.ewl.cccnt.cn/59739.Doc
m.ewl.cccnt.cn/19333.Doc
m.ewl.cccnt.cn/42082.Doc
m.ewl.cccnt.cn/37757.Doc
m.ewl.cccnt.cn/15995.Doc
m.ewl.cccnt.cn/31793.Doc
m.ewl.cccnt.cn/37111.Doc
m.ewl.cccnt.cn/28248.Doc
m.ewl.cccnt.cn/40442.Doc
m.ewl.cccnt.cn/35711.Doc
m.ewl.cccnt.cn/55913.Doc
m.ewl.cccnt.cn/40804.Doc
m.ewk.cccnt.cn/48404.Doc
m.ewk.cccnt.cn/39517.Doc
m.ewk.cccnt.cn/19179.Doc
m.ewk.cccnt.cn/53797.Doc
m.ewk.cccnt.cn/04268.Doc
m.ewk.cccnt.cn/95391.Doc
m.ewk.cccnt.cn/97571.Doc
m.ewk.cccnt.cn/24402.Doc
m.ewk.cccnt.cn/77559.Doc
m.ewk.cccnt.cn/77993.Doc
m.ewk.cccnt.cn/22668.Doc
m.ewk.cccnt.cn/59333.Doc
m.ewk.cccnt.cn/28246.Doc
m.ewk.cccnt.cn/97393.Doc
m.ewk.cccnt.cn/53173.Doc
m.ewk.cccnt.cn/55315.Doc
m.ewk.cccnt.cn/95593.Doc
m.ewk.cccnt.cn/99955.Doc
m.ewk.cccnt.cn/77591.Doc
m.ewk.cccnt.cn/11197.Doc
m.ewj.cccnt.cn/19351.Doc
m.ewj.cccnt.cn/93333.Doc
m.ewj.cccnt.cn/84624.Doc
m.ewj.cccnt.cn/22062.Doc
m.ewj.cccnt.cn/84040.Doc
m.ewj.cccnt.cn/11157.Doc
m.ewj.cccnt.cn/20428.Doc
m.ewj.cccnt.cn/82820.Doc
m.ewj.cccnt.cn/02260.Doc
m.ewj.cccnt.cn/26402.Doc
m.ewj.cccnt.cn/59377.Doc
m.ewj.cccnt.cn/46660.Doc
m.ewj.cccnt.cn/44026.Doc
m.ewj.cccnt.cn/15315.Doc
m.ewj.cccnt.cn/31773.Doc
m.ewj.cccnt.cn/80446.Doc
m.ewj.cccnt.cn/53793.Doc
m.ewj.cccnt.cn/15753.Doc
m.ewj.cccnt.cn/35979.Doc
m.ewj.cccnt.cn/22066.Doc
m.ewh.cccnt.cn/91135.Doc
m.ewh.cccnt.cn/33377.Doc
m.ewh.cccnt.cn/68266.Doc
m.ewh.cccnt.cn/02406.Doc
m.ewh.cccnt.cn/75591.Doc
m.ewh.cccnt.cn/37773.Doc
m.ewh.cccnt.cn/93793.Doc
m.ewh.cccnt.cn/22668.Doc
m.ewh.cccnt.cn/39199.Doc
m.ewh.cccnt.cn/55379.Doc
m.ewh.cccnt.cn/99351.Doc
m.ewh.cccnt.cn/55339.Doc
m.ewh.cccnt.cn/33917.Doc
m.ewh.cccnt.cn/93995.Doc
m.ewh.cccnt.cn/26288.Doc
m.ewh.cccnt.cn/95935.Doc
m.ewh.cccnt.cn/97515.Doc
m.ewh.cccnt.cn/59135.Doc
m.ewh.cccnt.cn/19795.Doc
m.ewh.cccnt.cn/88646.Doc
m.ewg.cccnt.cn/44048.Doc
m.ewg.cccnt.cn/06242.Doc
m.ewg.cccnt.cn/39953.Doc
m.ewg.cccnt.cn/15739.Doc
m.ewg.cccnt.cn/88444.Doc
m.ewg.cccnt.cn/24888.Doc
m.ewg.cccnt.cn/28608.Doc
m.ewg.cccnt.cn/51511.Doc
m.ewg.cccnt.cn/44860.Doc
m.ewg.cccnt.cn/28686.Doc
m.ewg.cccnt.cn/26640.Doc
m.ewg.cccnt.cn/53979.Doc
m.ewg.cccnt.cn/64464.Doc
m.ewg.cccnt.cn/93137.Doc
m.ewg.cccnt.cn/57511.Doc
m.ewg.cccnt.cn/53773.Doc
m.ewg.cccnt.cn/73191.Doc
m.ewg.cccnt.cn/59911.Doc
m.ewg.cccnt.cn/91515.Doc
m.ewg.cccnt.cn/95917.Doc
