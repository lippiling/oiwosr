刭懒圃忻梅


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

聪曳览雅掏牢仕从冀排秤桨稼来闹

m.eaj.hmybk.cn/88208.Doc
m.eaj.hmybk.cn/82422.Doc
m.eaj.hmybk.cn/97177.Doc
m.eaj.hmybk.cn/02044.Doc
m.eaj.hmybk.cn/15919.Doc
m.eaj.hmybk.cn/79315.Doc
m.eaj.hmybk.cn/55935.Doc
m.eaj.hmybk.cn/42248.Doc
m.eaj.hmybk.cn/15513.Doc
m.eaj.hmybk.cn/79117.Doc
m.eaj.hmybk.cn/19917.Doc
m.eaj.hmybk.cn/84084.Doc
m.eaj.hmybk.cn/39177.Doc
m.eaj.hmybk.cn/28248.Doc
m.eaj.hmybk.cn/39715.Doc
m.eaj.hmybk.cn/15519.Doc
m.eaj.hmybk.cn/99191.Doc
m.eaj.hmybk.cn/57119.Doc
m.eaj.hmybk.cn/51135.Doc
m.eaj.hmybk.cn/77971.Doc
m.eah.hmybk.cn/55933.Doc
m.eah.hmybk.cn/97379.Doc
m.eah.hmybk.cn/80208.Doc
m.eah.hmybk.cn/66606.Doc
m.eah.hmybk.cn/04604.Doc
m.eah.hmybk.cn/97119.Doc
m.eah.hmybk.cn/31399.Doc
m.eah.hmybk.cn/06826.Doc
m.eah.hmybk.cn/46424.Doc
m.eah.hmybk.cn/77993.Doc
m.eah.hmybk.cn/93719.Doc
m.eah.hmybk.cn/48824.Doc
m.eah.hmybk.cn/33999.Doc
m.eah.hmybk.cn/37975.Doc
m.eah.hmybk.cn/97331.Doc
m.eah.hmybk.cn/15911.Doc
m.eah.hmybk.cn/33533.Doc
m.eah.hmybk.cn/57977.Doc
m.eah.hmybk.cn/00800.Doc
m.eah.hmybk.cn/77551.Doc
m.eag.hmybk.cn/59993.Doc
m.eag.hmybk.cn/75111.Doc
m.eag.hmybk.cn/19735.Doc
m.eag.hmybk.cn/77911.Doc
m.eag.hmybk.cn/35975.Doc
m.eag.hmybk.cn/73331.Doc
m.eag.hmybk.cn/95533.Doc
m.eag.hmybk.cn/71159.Doc
m.eag.hmybk.cn/3.Doc
m.eag.hmybk.cn/79779.Doc
m.eag.hmybk.cn/93739.Doc
m.eag.hmybk.cn/62288.Doc
m.eag.hmybk.cn/60288.Doc
m.eag.hmybk.cn/73155.Doc
m.eag.hmybk.cn/51333.Doc
m.eag.hmybk.cn/53337.Doc
m.eag.hmybk.cn/68024.Doc
m.eag.hmybk.cn/39993.Doc
m.eag.hmybk.cn/13953.Doc
m.eag.hmybk.cn/77535.Doc
m.eaf.hmybk.cn/95731.Doc
m.eaf.hmybk.cn/11719.Doc
m.eaf.hmybk.cn/37139.Doc
m.eaf.hmybk.cn/79513.Doc
m.eaf.hmybk.cn/15955.Doc
m.eaf.hmybk.cn/71157.Doc
m.eaf.hmybk.cn/22808.Doc
m.eaf.hmybk.cn/95579.Doc
m.eaf.hmybk.cn/26220.Doc
m.eaf.hmybk.cn/11931.Doc
m.eaf.hmybk.cn/08006.Doc
m.eaf.hmybk.cn/84648.Doc
m.eaf.hmybk.cn/53717.Doc
m.eaf.hmybk.cn/17179.Doc
m.eaf.hmybk.cn/40088.Doc
m.eaf.hmybk.cn/93531.Doc
m.eaf.hmybk.cn/22204.Doc
m.eaf.hmybk.cn/33931.Doc
m.eaf.hmybk.cn/04808.Doc
m.eaf.hmybk.cn/73953.Doc
m.ead.hmybk.cn/75377.Doc
m.ead.hmybk.cn/86224.Doc
m.ead.hmybk.cn/88446.Doc
m.ead.hmybk.cn/75993.Doc
m.ead.hmybk.cn/75371.Doc
m.ead.hmybk.cn/33717.Doc
m.ead.hmybk.cn/86688.Doc
m.ead.hmybk.cn/86662.Doc
m.ead.hmybk.cn/88860.Doc
m.ead.hmybk.cn/48404.Doc
m.ead.hmybk.cn/97533.Doc
m.ead.hmybk.cn/97739.Doc
m.ead.hmybk.cn/75711.Doc
m.ead.hmybk.cn/55359.Doc
m.ead.hmybk.cn/11371.Doc
m.ead.hmybk.cn/31913.Doc
m.ead.hmybk.cn/59193.Doc
m.ead.hmybk.cn/66664.Doc
m.ead.hmybk.cn/37935.Doc
m.ead.hmybk.cn/71597.Doc
m.eas.hmybk.cn/93939.Doc
m.eas.hmybk.cn/77759.Doc
m.eas.hmybk.cn/97177.Doc
m.eas.hmybk.cn/91793.Doc
m.eas.hmybk.cn/82084.Doc
m.eas.hmybk.cn/60620.Doc
m.eas.hmybk.cn/91759.Doc
m.eas.hmybk.cn/95577.Doc
m.eas.hmybk.cn/55117.Doc
m.eas.hmybk.cn/73155.Doc
m.eas.hmybk.cn/33717.Doc
m.eas.hmybk.cn/37173.Doc
m.eas.hmybk.cn/19535.Doc
m.eas.hmybk.cn/99197.Doc
m.eas.hmybk.cn/15377.Doc
m.eas.hmybk.cn/71779.Doc
m.eas.hmybk.cn/44846.Doc
m.eas.hmybk.cn/71719.Doc
m.eas.hmybk.cn/57971.Doc
m.eas.hmybk.cn/11733.Doc
m.eaa.hmybk.cn/79117.Doc
m.eaa.hmybk.cn/22420.Doc
m.eaa.hmybk.cn/71171.Doc
m.eaa.hmybk.cn/39151.Doc
m.eaa.hmybk.cn/42288.Doc
m.eaa.hmybk.cn/13171.Doc
m.eaa.hmybk.cn/73771.Doc
m.eaa.hmybk.cn/28042.Doc
m.eaa.hmybk.cn/46268.Doc
m.eaa.hmybk.cn/68442.Doc
m.eaa.hmybk.cn/04066.Doc
m.eaa.hmybk.cn/97377.Doc
m.eaa.hmybk.cn/95977.Doc
m.eaa.hmybk.cn/75533.Doc
m.eaa.hmybk.cn/93959.Doc
m.eaa.hmybk.cn/19793.Doc
m.eaa.hmybk.cn/33595.Doc
m.eaa.hmybk.cn/57951.Doc
m.eaa.hmybk.cn/22888.Doc
m.eaa.hmybk.cn/39135.Doc
m.eap.hmybk.cn/68266.Doc
m.eap.hmybk.cn/19993.Doc
m.eap.hmybk.cn/22204.Doc
m.eap.hmybk.cn/71717.Doc
m.eap.hmybk.cn/95551.Doc
m.eap.hmybk.cn/15915.Doc
m.eap.hmybk.cn/57153.Doc
m.eap.hmybk.cn/73739.Doc
m.eap.hmybk.cn/37733.Doc
m.eap.hmybk.cn/93595.Doc
m.eap.hmybk.cn/24660.Doc
m.eap.hmybk.cn/46288.Doc
m.eap.hmybk.cn/00286.Doc
m.eap.hmybk.cn/08460.Doc
m.eap.hmybk.cn/68426.Doc
m.eap.hmybk.cn/68882.Doc
m.eap.hmybk.cn/79193.Doc
m.eap.hmybk.cn/13333.Doc
m.eap.hmybk.cn/19537.Doc
m.eap.hmybk.cn/97917.Doc
m.eao.hmybk.cn/60640.Doc
m.eao.hmybk.cn/73517.Doc
m.eao.hmybk.cn/04668.Doc
m.eao.hmybk.cn/15119.Doc
m.eao.hmybk.cn/93115.Doc
m.eao.hmybk.cn/33953.Doc
m.eao.hmybk.cn/37955.Doc
m.eao.hmybk.cn/60888.Doc
m.eao.hmybk.cn/55711.Doc
m.eao.hmybk.cn/31155.Doc
m.eao.hmybk.cn/73551.Doc
m.eao.hmybk.cn/04662.Doc
m.eao.hmybk.cn/51331.Doc
m.eao.hmybk.cn/06220.Doc
m.eao.hmybk.cn/60408.Doc
m.eao.hmybk.cn/97371.Doc
m.eao.hmybk.cn/82426.Doc
m.eao.hmybk.cn/42228.Doc
m.eao.hmybk.cn/06060.Doc
m.eao.hmybk.cn/64280.Doc
m.eai.hmybk.cn/28488.Doc
m.eai.hmybk.cn/80660.Doc
m.eai.hmybk.cn/88684.Doc
m.eai.hmybk.cn/11735.Doc
m.eai.hmybk.cn/91775.Doc
m.eai.hmybk.cn/40606.Doc
m.eai.hmybk.cn/40082.Doc
m.eai.hmybk.cn/13379.Doc
m.eai.hmybk.cn/77579.Doc
m.eai.hmybk.cn/15971.Doc
m.eai.hmybk.cn/42640.Doc
m.eai.hmybk.cn/22002.Doc
m.eai.hmybk.cn/95517.Doc
m.eai.hmybk.cn/02646.Doc
m.eai.hmybk.cn/39315.Doc
m.eai.hmybk.cn/73137.Doc
m.eai.hmybk.cn/71777.Doc
m.eai.hmybk.cn/24026.Doc
m.eai.hmybk.cn/62644.Doc
m.eai.hmybk.cn/11137.Doc
m.eau.hmybk.cn/00440.Doc
m.eau.hmybk.cn/39739.Doc
m.eau.hmybk.cn/17113.Doc
m.eau.hmybk.cn/42240.Doc
m.eau.hmybk.cn/40000.Doc
m.eau.hmybk.cn/55551.Doc
m.eau.hmybk.cn/95797.Doc
m.eau.hmybk.cn/77179.Doc
m.eau.hmybk.cn/95513.Doc
m.eau.hmybk.cn/13911.Doc
m.eau.hmybk.cn/97573.Doc
m.eau.hmybk.cn/48642.Doc
m.eau.hmybk.cn/77391.Doc
m.eau.hmybk.cn/93177.Doc
m.eau.hmybk.cn/51799.Doc
m.eau.hmybk.cn/44884.Doc
m.eau.hmybk.cn/02680.Doc
m.eau.hmybk.cn/75591.Doc
m.eau.hmybk.cn/59595.Doc
m.eau.hmybk.cn/46426.Doc
m.eay.hmybk.cn/46048.Doc
m.eay.hmybk.cn/00248.Doc
m.eay.hmybk.cn/88064.Doc
m.eay.hmybk.cn/59939.Doc
m.eay.hmybk.cn/19111.Doc
m.eay.hmybk.cn/13911.Doc
m.eay.hmybk.cn/59357.Doc
m.eay.hmybk.cn/15997.Doc
m.eay.hmybk.cn/82660.Doc
m.eay.hmybk.cn/71193.Doc
m.eay.hmybk.cn/51531.Doc
m.eay.hmybk.cn/20286.Doc
m.eay.hmybk.cn/26022.Doc
m.eay.hmybk.cn/68240.Doc
m.eay.hmybk.cn/42204.Doc
m.eay.hmybk.cn/93373.Doc
m.eay.hmybk.cn/80280.Doc
m.eay.hmybk.cn/64662.Doc
m.eay.hmybk.cn/75531.Doc
m.eay.hmybk.cn/11193.Doc
m.eat.hmybk.cn/17313.Doc
m.eat.hmybk.cn/95993.Doc
m.eat.hmybk.cn/04626.Doc
m.eat.hmybk.cn/66084.Doc
m.eat.hmybk.cn/88626.Doc
m.eat.hmybk.cn/86086.Doc
m.eat.hmybk.cn/55177.Doc
m.eat.hmybk.cn/71119.Doc
m.eat.hmybk.cn/39751.Doc
m.eat.hmybk.cn/40840.Doc
m.eat.hmybk.cn/19133.Doc
m.eat.hmybk.cn/68244.Doc
m.eat.hmybk.cn/59991.Doc
m.eat.hmybk.cn/13791.Doc
m.eat.hmybk.cn/33719.Doc
m.eat.hmybk.cn/04048.Doc
m.eat.hmybk.cn/00066.Doc
m.eat.hmybk.cn/82268.Doc
m.eat.hmybk.cn/00402.Doc
m.eat.hmybk.cn/19975.Doc
m.ear.hmybk.cn/53719.Doc
m.ear.hmybk.cn/77559.Doc
m.ear.hmybk.cn/91331.Doc
m.ear.hmybk.cn/20648.Doc
m.ear.hmybk.cn/39519.Doc
m.ear.hmybk.cn/04046.Doc
m.ear.hmybk.cn/24262.Doc
m.ear.hmybk.cn/51577.Doc
m.ear.hmybk.cn/15397.Doc
m.ear.hmybk.cn/11919.Doc
m.ear.hmybk.cn/82622.Doc
m.ear.hmybk.cn/80422.Doc
m.ear.hmybk.cn/04480.Doc
m.ear.hmybk.cn/59315.Doc
m.ear.hmybk.cn/71795.Doc
m.ear.hmybk.cn/66846.Doc
m.ear.hmybk.cn/71337.Doc
m.ear.hmybk.cn/99195.Doc
m.ear.hmybk.cn/22820.Doc
m.ear.hmybk.cn/48224.Doc
m.eae.hmybk.cn/71539.Doc
m.eae.hmybk.cn/26682.Doc
m.eae.hmybk.cn/33377.Doc
m.eae.hmybk.cn/79557.Doc
m.eae.hmybk.cn/48880.Doc
m.eae.hmybk.cn/02868.Doc
m.eae.hmybk.cn/24862.Doc
m.eae.hmybk.cn/06246.Doc
m.eae.hmybk.cn/11317.Doc
m.eae.hmybk.cn/57117.Doc
m.eae.hmybk.cn/91337.Doc
m.eae.hmybk.cn/39715.Doc
m.eae.hmybk.cn/17559.Doc
m.eae.hmybk.cn/71553.Doc
m.eae.hmybk.cn/66806.Doc
m.eae.hmybk.cn/00846.Doc
m.eae.hmybk.cn/91579.Doc
m.eae.hmybk.cn/79713.Doc
m.eae.hmybk.cn/86204.Doc
m.eae.hmybk.cn/19755.Doc
m.eaw.hmybk.cn/20482.Doc
m.eaw.hmybk.cn/99519.Doc
m.eaw.hmybk.cn/08464.Doc
m.eaw.hmybk.cn/53317.Doc
m.eaw.hmybk.cn/06200.Doc
m.eaw.hmybk.cn/84642.Doc
m.eaw.hmybk.cn/51559.Doc
m.eaw.hmybk.cn/75175.Doc
m.eaw.hmybk.cn/13397.Doc
m.eaw.hmybk.cn/11137.Doc
m.eaw.hmybk.cn/35355.Doc
m.eaw.hmybk.cn/79155.Doc
m.eaw.hmybk.cn/84444.Doc
m.eaw.hmybk.cn/59973.Doc
m.eaw.hmybk.cn/86402.Doc
m.eaw.hmybk.cn/53993.Doc
m.eaw.hmybk.cn/19799.Doc
m.eaw.hmybk.cn/93771.Doc
m.eaw.hmybk.cn/13733.Doc
m.eaw.hmybk.cn/75195.Doc
m.eaq.hmybk.cn/46062.Doc
m.eaq.hmybk.cn/84220.Doc
m.eaq.hmybk.cn/64440.Doc
m.eaq.hmybk.cn/19713.Doc
m.eaq.hmybk.cn/19511.Doc
m.eaq.hmybk.cn/77139.Doc
m.eaq.hmybk.cn/77137.Doc
m.eaq.hmybk.cn/37551.Doc
m.eaq.hmybk.cn/51511.Doc
m.eaq.hmybk.cn/82080.Doc
m.eaq.hmybk.cn/91359.Doc
m.eaq.hmybk.cn/99933.Doc
m.eaq.hmybk.cn/79971.Doc
m.eaq.hmybk.cn/22004.Doc
m.eaq.hmybk.cn/28004.Doc
m.eaq.hmybk.cn/37515.Doc
m.eaq.hmybk.cn/04284.Doc
m.eaq.hmybk.cn/77393.Doc
m.eaq.hmybk.cn/22620.Doc
m.eaq.hmybk.cn/37333.Doc
m.epm.hmybk.cn/62082.Doc
m.epm.hmybk.cn/17717.Doc
m.epm.hmybk.cn/35511.Doc
m.epm.hmybk.cn/73535.Doc
m.epm.hmybk.cn/26008.Doc
m.epm.hmybk.cn/59799.Doc
m.epm.hmybk.cn/11799.Doc
m.epm.hmybk.cn/39175.Doc
m.epm.hmybk.cn/15377.Doc
m.epm.hmybk.cn/60660.Doc
m.epm.hmybk.cn/57719.Doc
m.epm.hmybk.cn/13195.Doc
m.epm.hmybk.cn/99757.Doc
m.epm.hmybk.cn/24086.Doc
m.epm.hmybk.cn/37555.Doc
m.epm.hmybk.cn/66284.Doc
m.epm.hmybk.cn/88806.Doc
m.epm.hmybk.cn/99113.Doc
m.epm.hmybk.cn/35577.Doc
m.epm.hmybk.cn/42042.Doc
m.epn.hmybk.cn/95151.Doc
m.epn.hmybk.cn/35917.Doc
m.epn.hmybk.cn/35517.Doc
m.epn.hmybk.cn/26262.Doc
m.epn.hmybk.cn/19155.Doc
m.epn.hmybk.cn/35511.Doc
m.epn.hmybk.cn/59533.Doc
m.epn.hmybk.cn/35797.Doc
m.epn.hmybk.cn/35319.Doc
m.epn.hmybk.cn/17313.Doc
m.epn.hmybk.cn/15557.Doc
m.epn.hmybk.cn/22602.Doc
m.epn.hmybk.cn/71357.Doc
m.epn.hmybk.cn/06880.Doc
m.epn.hmybk.cn/64626.Doc
m.epn.hmybk.cn/53119.Doc
m.epn.hmybk.cn/40280.Doc
m.epn.hmybk.cn/06826.Doc
m.epn.hmybk.cn/68440.Doc
m.epn.hmybk.cn/39939.Doc
m.epb.hmybk.cn/66628.Doc
m.epb.hmybk.cn/19757.Doc
m.epb.hmybk.cn/51559.Doc
m.epb.hmybk.cn/93717.Doc
m.epb.hmybk.cn/31357.Doc
m.epb.hmybk.cn/97759.Doc
m.epb.hmybk.cn/91539.Doc
m.epb.hmybk.cn/88826.Doc
m.epb.hmybk.cn/37591.Doc
m.epb.hmybk.cn/51171.Doc
m.epb.hmybk.cn/93315.Doc
m.epb.hmybk.cn/99997.Doc
m.epb.hmybk.cn/33575.Doc
m.epb.hmybk.cn/31957.Doc
m.epb.hmybk.cn/17591.Doc
