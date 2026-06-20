胶忻衬幢拷


# Python 领域驱动设计 (Domain-Driven Design)
# Entity, ValueObject, Aggregate, DomainService, Repository
# DDD 以领域为核心，通过通用语言在代码中精确表达业务概念。

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Optional

# 值对象 (Value Object) - 不可变
@dataclass(frozen=True)
class Money:
    amount: float; currency: str = "CNY"
    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("货币不匹配")
        return Money(self.amount + other.amount, self.currency)

@dataclass(frozen=True)
class Address:
    province: str; city: str; detail: str

# 实体 (Entity) - 有唯一标识
@dataclass
class Customer:
    id: str; name: str; email: str
    address: Address | None = None
    def change_email(self, new_email: str) -> None:
        if "@" not in new_email:
            raise ValueError("邮箱格式无效")
        self.email = new_email

# 聚合 (Aggregate)
@dataclass
class OrderLine:
    product_id: str; quantity: int; unit_price: Money

@dataclass
class Order:
    """聚合根：保证聚合内部业务不变量"""
    id: str; customer_id: str
    lines: list[OrderLine] = field(default_factory=list)
    total: Money = field(default_factory=lambda: Money(0))
    status: str = "pending"

    def add_line(self, pid: str, qty: int, price: Money) -> None:
        self.lines.append(OrderLine(pid, qty, price))
        self.total = self.total.add(Money(price.amount * qty))

    def submit(self) -> None:
        if not self.lines:
            raise ValueError("空订单不能提交")
        self.status = "submitted"

# 领域服务 (Domain Service)
class PricingService:
    def apply_discount(self, order: Order, rate: float) -> Money:
        if not 0 < rate <= 1:
            raise ValueError("折扣率需在 0~1 之间")
        return Money(round(order.total.amount * (1 - rate), 2))

# 仓储 (Repository)
class OrderRepository(ABC):
    @abstractmethod
    def save(self, o: Order) -> None: ...
    @abstractmethod
    def by_id(self, oid: str) -> Optional[Order]: ...

class MemoryOrderRepo(OrderRepository):
    def __init__(self):
        self._store: dict[str, Order] = {}
    def save(self, o: Order) -> None:
        self._store[o.id] = o
    def by_id(self, oid: str) -> Optional[Order]:
        return self._store.get(oid)

if __name__ == "__main__":
    c = Customer("C001", "张三", "z@test.com",
                 Address("广东", "深圳", "科技园"))
    o = Order("O001", c.id)
    o.add_line("P001", 2, Money(99.00))
    o.submit()
    final = PricingService().apply_discount(o, 0.1)
    print(f"折后价: {final.amount} {final.currency}")

刂酒擞诽静恍胸核司亓贤关姿头炔

qup.ag5qr6tv.cn/759315.htm
qup.ag5qr6tv.cn/397775.htm
qup.ag5qr6tv.cn/935355.htm
qup.ag5qr6tv.cn/599315.htm
qup.ag5qr6tv.cn/551575.htm
qup.ag5qr6tv.cn/795515.htm
qup.ag5qr6tv.cn/133795.htm
qup.ag5qr6tv.cn/739175.htm
quo.ag5qr6tv.cn/117595.htm
quo.ag5qr6tv.cn/119315.htm
quo.ag5qr6tv.cn/557795.htm
quo.ag5qr6tv.cn/353375.htm
quo.ag5qr6tv.cn/771135.htm
quo.ag5qr6tv.cn/739315.htm
quo.ag5qr6tv.cn/713575.htm
quo.ag5qr6tv.cn/973935.htm
quo.ag5qr6tv.cn/137355.htm
quo.ag5qr6tv.cn/595335.htm
qui.ag5qr6tv.cn/595195.htm
qui.ag5qr6tv.cn/753335.htm
qui.ag5qr6tv.cn/171515.htm
qui.ag5qr6tv.cn/335915.htm
qui.ag5qr6tv.cn/359195.htm
qui.ag5qr6tv.cn/195535.htm
qui.ag5qr6tv.cn/539315.htm
qui.ag5qr6tv.cn/753375.htm
qui.ag5qr6tv.cn/599535.htm
qui.ag5qr6tv.cn/317775.htm
quu.ag5qr6tv.cn/793795.htm
quu.ag5qr6tv.cn/155955.htm
quu.ag5qr6tv.cn/191395.htm
quu.ag5qr6tv.cn/577975.htm
quu.ag5qr6tv.cn/599155.htm
quu.ag5qr6tv.cn/311315.htm
quu.ag5qr6tv.cn/133575.htm
quu.ag5qr6tv.cn/399795.htm
quu.ag5qr6tv.cn/739175.htm
quu.ag5qr6tv.cn/739955.htm
quy.ag5qr6tv.cn/777135.htm
quy.ag5qr6tv.cn/533515.htm
quy.ag5qr6tv.cn/977535.htm
quy.ag5qr6tv.cn/115155.htm
quy.ag5qr6tv.cn/555555.htm
quy.ag5qr6tv.cn/357735.htm
quy.ag5qr6tv.cn/933355.htm
quy.ag5qr6tv.cn/977355.htm
quy.ag5qr6tv.cn/159155.htm
quy.ag5qr6tv.cn/531195.htm
qut.ag5qr6tv.cn/939395.htm
qut.ag5qr6tv.cn/971375.htm
qut.ag5qr6tv.cn/315995.htm
qut.ag5qr6tv.cn/335315.htm
qut.ag5qr6tv.cn/975175.htm
qut.ag5qr6tv.cn/391555.htm
qut.ag5qr6tv.cn/577395.htm
qut.ag5qr6tv.cn/517935.htm
qut.ag5qr6tv.cn/533955.htm
qut.ag5qr6tv.cn/535775.htm
qur.ag5qr6tv.cn/797515.htm
qur.ag5qr6tv.cn/771135.htm
qur.ag5qr6tv.cn/975375.htm
qur.ag5qr6tv.cn/935975.htm
qur.ag5qr6tv.cn/133135.htm
qur.ag5qr6tv.cn/135755.htm
qur.ag5qr6tv.cn/153935.htm
qur.ag5qr6tv.cn/911135.htm
qur.ag5qr6tv.cn/393975.htm
qur.ag5qr6tv.cn/391155.htm
que.ag5qr6tv.cn/137735.htm
que.ag5qr6tv.cn/757995.htm
que.ag5qr6tv.cn/339735.htm
que.ag5qr6tv.cn/513535.htm
que.ag5qr6tv.cn/937355.htm
que.ag5qr6tv.cn/977915.htm
que.ag5qr6tv.cn/535595.htm
que.ag5qr6tv.cn/171535.htm
que.ag5qr6tv.cn/757375.htm
que.ag5qr6tv.cn/133335.htm
quw.ag5qr6tv.cn/771195.htm
quw.ag5qr6tv.cn/317155.htm
quw.ag5qr6tv.cn/953995.htm
quw.ag5qr6tv.cn/737515.htm
quw.ag5qr6tv.cn/175575.htm
quw.ag5qr6tv.cn/759995.htm
quw.ag5qr6tv.cn/913575.htm
quw.ag5qr6tv.cn/371335.htm
quw.ag5qr6tv.cn/397975.htm
quw.ag5qr6tv.cn/115775.htm
quq.ag5qr6tv.cn/391935.htm
quq.ag5qr6tv.cn/335315.htm
quq.ag5qr6tv.cn/935115.htm
quq.ag5qr6tv.cn/115335.htm
quq.ag5qr6tv.cn/397735.htm
quq.ag5qr6tv.cn/353715.htm
quq.ag5qr6tv.cn/733935.htm
quq.ag5qr6tv.cn/555795.htm
quq.ag5qr6tv.cn/939735.htm
quq.ag5qr6tv.cn/133315.htm
qytv.ag5qr6tv.cn/991315.htm
qytv.ag5qr6tv.cn/739395.htm
qytv.ag5qr6tv.cn/593975.htm
qytv.ag5qr6tv.cn/333355.htm
qytv.ag5qr6tv.cn/999175.htm
qytv.ag5qr6tv.cn/733735.htm
qytv.ag5qr6tv.cn/793355.htm
qytv.ag5qr6tv.cn/997955.htm
qytv.ag5qr6tv.cn/375135.htm
qytv.ag5qr6tv.cn/779735.htm
qyn.ag5qr6tv.cn/117395.htm
qyn.ag5qr6tv.cn/155315.htm
qyn.ag5qr6tv.cn/795995.htm
qyn.ag5qr6tv.cn/313975.htm
qyn.ag5qr6tv.cn/757155.htm
qyn.ag5qr6tv.cn/537955.htm
qyn.ag5qr6tv.cn/313115.htm
qyn.ag5qr6tv.cn/139115.htm
qyn.ag5qr6tv.cn/395115.htm
qyn.ag5qr6tv.cn/159535.htm
qyb.ag5qr6tv.cn/537175.htm
qyb.ag5qr6tv.cn/333115.htm
qyb.ag5qr6tv.cn/331395.htm
qyb.ag5qr6tv.cn/119735.htm
qyb.ag5qr6tv.cn/955155.htm
qyb.ag5qr6tv.cn/759135.htm
qyb.ag5qr6tv.cn/917715.htm
qyb.ag5qr6tv.cn/337115.htm
qyb.ag5qr6tv.cn/757955.htm
qyb.ag5qr6tv.cn/153155.htm
qyv.ag5qr6tv.cn/559135.htm
qyv.ag5qr6tv.cn/319955.htm
qyv.ag5qr6tv.cn/557755.htm
qyv.ag5qr6tv.cn/711195.htm
qyv.ag5qr6tv.cn/951175.htm
qyv.ag5qr6tv.cn/155775.htm
qyv.ag5qr6tv.cn/111735.htm
qyv.ag5qr6tv.cn/377595.htm
qyv.ag5qr6tv.cn/759155.htm
qyv.ag5qr6tv.cn/933135.htm
qyc.ag5qr6tv.cn/339795.htm
qyc.ag5qr6tv.cn/137315.htm
qyc.ag5qr6tv.cn/333995.htm
qyc.ag5qr6tv.cn/933395.htm
qyc.ag5qr6tv.cn/171515.htm
qyc.ag5qr6tv.cn/193795.htm
qyc.ag5qr6tv.cn/917375.htm
qyc.ag5qr6tv.cn/733515.htm
qyc.ag5qr6tv.cn/995515.htm
qyc.ag5qr6tv.cn/513515.htm
qyx.ag5qr6tv.cn/157195.htm
qyx.ag5qr6tv.cn/775595.htm
qyx.ag5qr6tv.cn/191715.htm
qyx.ag5qr6tv.cn/599575.htm
qyx.ag5qr6tv.cn/575375.htm
qyx.ag5qr6tv.cn/337195.htm
qyx.ag5qr6tv.cn/799755.htm
qyx.ag5qr6tv.cn/551995.htm
qyx.ag5qr6tv.cn/755355.htm
qyx.ag5qr6tv.cn/975935.htm
qyz.ag5qr6tv.cn/137515.htm
qyz.ag5qr6tv.cn/553775.htm
qyz.ag5qr6tv.cn/753935.htm
qyz.ag5qr6tv.cn/395195.htm
qyz.ag5qr6tv.cn/577775.htm
qyz.ag5qr6tv.cn/771535.htm
qyz.ag5qr6tv.cn/115155.htm
qyz.ag5qr6tv.cn/155375.htm
qyz.ag5qr6tv.cn/379775.htm
qyz.ag5qr6tv.cn/913595.htm
qyl.ag5qr6tv.cn/957315.htm
qyl.ag5qr6tv.cn/399935.htm
qyl.ag5qr6tv.cn/177755.htm
qyl.ag5qr6tv.cn/373195.htm
qyl.ag5qr6tv.cn/955175.htm
qyl.ag5qr6tv.cn/577935.htm
qyl.ag5qr6tv.cn/371515.htm
qyl.ag5qr6tv.cn/513175.htm
qyl.ag5qr6tv.cn/397155.htm
qyl.ag5qr6tv.cn/197795.htm
qyk.ag5qr6tv.cn/171195.htm
qyk.ag5qr6tv.cn/791555.htm
qyk.ag5qr6tv.cn/795175.htm
qyk.ag5qr6tv.cn/775535.htm
qyk.ag5qr6tv.cn/595595.htm
qyk.ag5qr6tv.cn/359135.htm
qyk.ag5qr6tv.cn/551715.htm
qyk.ag5qr6tv.cn/975175.htm
qyk.ag5qr6tv.cn/197755.htm
qyk.ag5qr6tv.cn/755715.htm
qyj.ag5qr6tv.cn/513915.htm
qyj.ag5qr6tv.cn/337595.htm
qyj.ag5qr6tv.cn/919375.htm
qyj.ag5qr6tv.cn/977735.htm
qyj.ag5qr6tv.cn/739755.htm
qyj.ag5qr6tv.cn/353935.htm
qyj.ag5qr6tv.cn/795555.htm
qyj.ag5qr6tv.cn/371175.htm
qyj.ag5qr6tv.cn/537595.htm
qyj.ag5qr6tv.cn/397715.htm
qyh.ag5qr6tv.cn/339375.htm
qyh.ag5qr6tv.cn/917315.htm
qyh.ag5qr6tv.cn/175375.htm
qyh.ag5qr6tv.cn/797955.htm
qyh.ag5qr6tv.cn/973575.htm
qyh.ag5qr6tv.cn/591135.htm
qyh.ag5qr6tv.cn/919535.htm
qyh.ag5qr6tv.cn/953175.htm
qyh.ag5qr6tv.cn/339975.htm
qyh.ag5qr6tv.cn/757755.htm
qyg.ag5qr6tv.cn/375795.htm
qyg.ag5qr6tv.cn/153915.htm
qyg.ag5qr6tv.cn/535955.htm
qyg.ag5qr6tv.cn/193955.htm
qyg.ag5qr6tv.cn/131915.htm
qyg.ag5qr6tv.cn/975775.htm
qyg.ag5qr6tv.cn/171515.htm
qyg.ag5qr6tv.cn/911715.htm
qyg.ag5qr6tv.cn/537355.htm
qyg.ag5qr6tv.cn/793155.htm
qyf.ag5qr6tv.cn/933115.htm
qyf.ag5qr6tv.cn/111995.htm
qyf.ag5qr6tv.cn/999955.htm
qyf.ag5qr6tv.cn/919315.htm
qyf.ag5qr6tv.cn/375115.htm
qyf.ag5qr6tv.cn/531535.htm
qyf.ag5qr6tv.cn/557395.htm
qyf.ag5qr6tv.cn/935975.htm
qyf.ag5qr6tv.cn/177355.htm
qyf.ag5qr6tv.cn/777195.htm
qyd.ag5qr6tv.cn/573735.htm
qyd.ag5qr6tv.cn/719735.htm
qyd.ag5qr6tv.cn/913735.htm
qyd.ag5qr6tv.cn/151575.htm
qyd.ag5qr6tv.cn/571375.htm
qyd.ag5qr6tv.cn/313135.htm
qyd.ag5qr6tv.cn/555555.htm
qyd.ag5qr6tv.cn/355715.htm
qyd.ag5qr6tv.cn/135395.htm
qyd.ag5qr6tv.cn/575355.htm
qys.ag5qr6tv.cn/597915.htm
qys.ag5qr6tv.cn/533515.htm
qys.ag5qr6tv.cn/731175.htm
qys.ag5qr6tv.cn/377135.htm
qys.ag5qr6tv.cn/339515.htm
qys.ag5qr6tv.cn/593515.htm
qys.ag5qr6tv.cn/773735.htm
qys.ag5qr6tv.cn/713375.htm
qys.ag5qr6tv.cn/339735.htm
qys.ag5qr6tv.cn/777595.htm
qya.ag5qr6tv.cn/735955.htm
qya.ag5qr6tv.cn/351395.htm
qya.ag5qr6tv.cn/359715.htm
qya.ag5qr6tv.cn/939555.htm
qya.ag5qr6tv.cn/551115.htm
qya.ag5qr6tv.cn/117555.htm
qya.ag5qr6tv.cn/373335.htm
qya.ag5qr6tv.cn/137795.htm
qya.ag5qr6tv.cn/913375.htm
qya.ag5qr6tv.cn/979315.htm
qyp.ag5qr6tv.cn/199515.htm
qyp.ag5qr6tv.cn/119315.htm
qyp.ag5qr6tv.cn/359755.htm
qyp.ag5qr6tv.cn/715375.htm
qyp.ag5qr6tv.cn/735715.htm
qyp.ag5qr6tv.cn/115375.htm
qyp.ag5qr6tv.cn/999595.htm
qyp.ag5qr6tv.cn/139395.htm
qyp.ag5qr6tv.cn/179515.htm
qyp.ag5qr6tv.cn/511755.htm
qyo.ag5qr6tv.cn/595575.htm
qyo.ag5qr6tv.cn/131195.htm
qyo.ag5qr6tv.cn/333555.htm
qyo.ag5qr6tv.cn/753115.htm
qyo.ag5qr6tv.cn/337395.htm
qyo.ag5qr6tv.cn/935395.htm
qyo.ag5qr6tv.cn/373315.htm
qyo.ag5qr6tv.cn/575755.htm
qyo.ag5qr6tv.cn/377975.htm
qyo.ag5qr6tv.cn/597375.htm
qyi.ag5qr6tv.cn/373715.htm
qyi.ag5qr6tv.cn/751915.htm
qyi.ag5qr6tv.cn/913555.htm
qyi.ag5qr6tv.cn/175995.htm
qyi.ag5qr6tv.cn/913955.htm
qyi.ag5qr6tv.cn/139395.htm
qyi.ag5qr6tv.cn/131555.htm
qyi.ag5qr6tv.cn/137995.htm
qyi.ag5qr6tv.cn/739515.htm
qyi.ag5qr6tv.cn/153135.htm
qyu.ag5qr6tv.cn/137195.htm
qyu.ag5qr6tv.cn/517515.htm
qyu.ag5qr6tv.cn/351135.htm
qyu.ag5qr6tv.cn/775515.htm
qyu.ag5qr6tv.cn/513335.htm
qyu.ag5qr6tv.cn/371195.htm
qyu.ag5qr6tv.cn/519315.htm
qyu.ag5qr6tv.cn/977375.htm
qyu.ag5qr6tv.cn/517795.htm
qyu.ag5qr6tv.cn/191175.htm
qyy.ag5qr6tv.cn/713555.htm
qyy.ag5qr6tv.cn/117175.htm
qyy.ag5qr6tv.cn/755375.htm
qyy.ag5qr6tv.cn/973995.htm
qyy.ag5qr6tv.cn/979375.htm
qyy.ag5qr6tv.cn/557175.htm
qyy.ag5qr6tv.cn/795315.htm
qyy.ag5qr6tv.cn/359775.htm
qyy.ag5qr6tv.cn/575155.htm
qyy.ag5qr6tv.cn/555555.htm
qyt.ag5qr6tv.cn/571755.htm
qyt.ag5qr6tv.cn/759175.htm
qyt.ag5qr6tv.cn/733155.htm
qyt.ag5qr6tv.cn/133775.htm
qyt.ag5qr6tv.cn/957375.htm
qyt.ag5qr6tv.cn/173535.htm
qyt.ag5qr6tv.cn/377535.htm
qyt.ag5qr6tv.cn/977775.htm
qyt.ag5qr6tv.cn/991115.htm
qyt.ag5qr6tv.cn/517375.htm
qyr.ag5qr6tv.cn/915795.htm
qyr.ag5qr6tv.cn/399595.htm
qyr.ag5qr6tv.cn/795135.htm
qyr.ag5qr6tv.cn/931755.htm
qyr.ag5qr6tv.cn/771935.htm
qyr.ag5qr6tv.cn/357315.htm
qyr.ag5qr6tv.cn/959555.htm
qyr.ag5qr6tv.cn/913595.htm
qyr.ag5qr6tv.cn/915795.htm
qyr.ag5qr6tv.cn/571775.htm
qye.ag5qr6tv.cn/793795.htm
qye.ag5qr6tv.cn/593395.htm
qye.ag5qr6tv.cn/951555.htm
qye.ag5qr6tv.cn/791515.htm
qye.ag5qr6tv.cn/375795.htm
qye.ag5qr6tv.cn/573135.htm
qye.ag5qr6tv.cn/911155.htm
qye.ag5qr6tv.cn/531735.htm
qye.ag5qr6tv.cn/573335.htm
qye.ag5qr6tv.cn/975155.htm
qyw.ag5qr6tv.cn/533715.htm
qyw.ag5qr6tv.cn/393515.htm
qyw.ag5qr6tv.cn/379135.htm
qyw.ag5qr6tv.cn/779395.htm
qyw.ag5qr6tv.cn/737795.htm
qyw.ag5qr6tv.cn/371995.htm
qyw.ag5qr6tv.cn/933515.htm
qyw.ag5qr6tv.cn/311755.htm
qyw.ag5qr6tv.cn/575795.htm
qyw.ag5qr6tv.cn/997115.htm
qyq.ag5qr6tv.cn/591395.htm
qyq.ag5qr6tv.cn/573155.htm
qyq.ag5qr6tv.cn/733515.htm
qyq.ag5qr6tv.cn/557995.htm
qyq.ag5qr6tv.cn/377195.htm
qyq.ag5qr6tv.cn/919775.htm
qyq.ag5qr6tv.cn/551755.htm
qyq.ag5qr6tv.cn/519155.htm
qyq.ag5qr6tv.cn/593315.htm
qyq.ag5qr6tv.cn/199335.htm
qttv.ag5qr6tv.cn/935795.htm
qttv.ag5qr6tv.cn/757135.htm
qttv.ag5qr6tv.cn/173975.htm
qttv.ag5qr6tv.cn/157935.htm
qttv.ag5qr6tv.cn/111995.htm
qttv.ag5qr6tv.cn/535775.htm
qttv.ag5qr6tv.cn/917735.htm
qttv.ag5qr6tv.cn/777795.htm
qttv.ag5qr6tv.cn/113115.htm
qttv.ag5qr6tv.cn/157375.htm
qtn.ag5qr6tv.cn/991395.htm
qtn.ag5qr6tv.cn/575935.htm
qtn.ag5qr6tv.cn/957335.htm
qtn.ag5qr6tv.cn/957735.htm
qtn.ag5qr6tv.cn/393335.htm
qtn.ag5qr6tv.cn/591315.htm
qtn.ag5qr6tv.cn/713915.htm
qtn.ag5qr6tv.cn/919375.htm
qtn.ag5qr6tv.cn/313575.htm
qtn.ag5qr6tv.cn/577375.htm
qtb.ag5qr6tv.cn/333515.htm
qtb.ag5qr6tv.cn/151595.htm
qtb.ag5qr6tv.cn/179595.htm
qtb.ag5qr6tv.cn/717175.htm
qtb.ag5qr6tv.cn/333575.htm
qtb.ag5qr6tv.cn/919335.htm
qtb.ag5qr6tv.cn/515995.htm
qtb.ag5qr6tv.cn/771575.htm
qtb.ag5qr6tv.cn/311795.htm
qtb.ag5qr6tv.cn/315555.htm
qtv.ag5qr6tv.cn/955915.htm
qtv.ag5qr6tv.cn/991755.htm
qtv.ag5qr6tv.cn/771535.htm
qtv.ag5qr6tv.cn/173375.htm
qtv.ag5qr6tv.cn/157135.htm
qtv.ag5qr6tv.cn/153595.htm
qtv.ag5qr6tv.cn/517395.htm
