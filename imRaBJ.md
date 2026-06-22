孪浦谢素耙


# Python 事件风暴模式 (Event Storming)
# 领域事件，事件记录，事件调度器，事件处理器，事件版本控制
# -----------------------------------------------------------
# 事件风暴通过领域事件还原业务全貌，让领域专家与开发团队
# 在同一张事件图上达成共识。事件是"已经发生的事实"。
# -----------------------------------------------------------

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime
from typing import Callable

# ===================== 领域事件基类 =====================
@dataclass
class DomainEvent:
    event_id: str
    occurred_at: datetime = field(default_factory=datetime.now)
    version: int = 1  # 事件版本号，用于向后兼容升级

# ===================== 具体事件 =====================
class OrderSubmitted(DomainEvent):
    def __init__(self, order_id: str, customer_id: str, total: float):
        super().__init__(event_id=f"evt-{order_id}")
        self.order_id = order_id
        self.customer_id = customer_id
        self.total = total

class OrderApproved(DomainEvent):
    def __init__(self, order_id: str, approver: str):
        super().__init__(event_id=f"evt-{order_id}-approved")
        self.order_id = order_id
        self.approver = approver

# ===================== 事件记录器 =====================
class EventStore:
    """事件存储：记录所有已发生的领域事件"""
    def __init__(self):
        self._events: list[DomainEvent] = []

    def append(self, event: DomainEvent) -> None:
        self._events.append(event)

    def get_events(self, since: datetime | None = None) -> list[DomainEvent]:
        if since is None:
            return list(self._events)
        return [e for e in self._events if e.occurred_at > since]

# ===================== 事件处理器 =====================
EventHandler = Callable[[DomainEvent], None]

class EventDispatcher:
    """事件调度器：将事件分发给已注册的处理器"""
    def __init__(self, event_store: EventStore):
        self._handlers: dict[type, list[EventHandler]] = {}
        self._store = event_store

    def register(self, evt_type: type, handler: EventHandler) -> None:
        self._handlers.setdefault(evt_type, []).append(handler)

    def dispatch(self, event: DomainEvent) -> None:
        self._store.append(event)  # 先持久化再分发
        for handler in self._handlers.get(type(event), []):
            handler(event)

# ===================== 具体处理器 =====================
def send_email(event: OrderSubmitted) -> None:
    print(f"[邮件] 订单 {event.order_id} 已提交，金额 {event.total}")

def update_inventory(event: OrderApproved) -> None:
    print(f"[库存] 订单 {event.order_id} 已审核，更新库存")

# ===================== 事件版本迁移 =====================
class EventUpcaster:
    @staticmethod
    def upcast_v1_to_v2(old: dict) -> dict:
        old["version"] = 2
        old["occurred_at"] = old.pop("created_at", old.get("occurred_at"))
        return old

if __name__ == "__main__":
    store = EventStore()
    dispatcher = EventDispatcher(store)
    dispatcher.register(OrderSubmitted, send_email)
    dispatcher.register(OrderApproved, update_inventory)
    dispatcher.dispatch(OrderSubmitted("O001", "C001", 299.0))
    dispatcher.dispatch(OrderApproved("O001", "admin"))
    print(f"已记录 {len(store.get_events())} 个事件")

鼗傩衫钦匚绞萍忠坠由讶纺膛衅吓

dgh.fffbf.cn/424200.Doc
dgh.fffbf.cn/484086.Doc
dgh.fffbf.cn/268028.Doc
dgh.fffbf.cn/684284.Doc
dgh.fffbf.cn/464208.Doc
dgh.fffbf.cn/680884.Doc
dgh.fffbf.cn/597333.Doc
dgh.fffbf.cn/882042.Doc
dgh.fffbf.cn/064082.Doc
dgh.fffbf.cn/480080.Doc
dgg.fffbf.cn/088866.Doc
dgg.fffbf.cn/882842.Doc
dgg.fffbf.cn/424826.Doc
dgg.fffbf.cn/868864.Doc
dgg.fffbf.cn/046064.Doc
dgg.fffbf.cn/086420.Doc
dgg.fffbf.cn/648802.Doc
dgg.fffbf.cn/424042.Doc
dgg.fffbf.cn/804884.Doc
dgg.fffbf.cn/800000.Doc
dgf.fffbf.cn/422808.Doc
dgf.fffbf.cn/202666.Doc
dgf.fffbf.cn/379753.Doc
dgf.fffbf.cn/062446.Doc
dgf.fffbf.cn/646848.Doc
dgf.fffbf.cn/971353.Doc
dgf.fffbf.cn/139719.Doc
dgf.fffbf.cn/408242.Doc
dgf.fffbf.cn/206804.Doc
dgf.fffbf.cn/557759.Doc
dgd.fffbf.cn/486826.Doc
dgd.fffbf.cn/022446.Doc
dgd.fffbf.cn/420068.Doc
dgd.fffbf.cn/662466.Doc
dgd.fffbf.cn/680400.Doc
dgd.fffbf.cn/006608.Doc
dgd.fffbf.cn/060204.Doc
dgd.fffbf.cn/802826.Doc
dgd.fffbf.cn/446084.Doc
dgd.fffbf.cn/286428.Doc
dgs.fffbf.cn/880626.Doc
dgs.fffbf.cn/264402.Doc
dgs.fffbf.cn/626068.Doc
dgs.fffbf.cn/208000.Doc
dgs.fffbf.cn/379937.Doc
dgs.fffbf.cn/024466.Doc
dgs.fffbf.cn/684086.Doc
dgs.fffbf.cn/319775.Doc
dgs.fffbf.cn/824624.Doc
dgs.fffbf.cn/597753.Doc
dga.fffbf.cn/446000.Doc
dga.fffbf.cn/428604.Doc
dga.fffbf.cn/935915.Doc
dga.fffbf.cn/006064.Doc
dga.fffbf.cn/444088.Doc
dga.fffbf.cn/486024.Doc
dga.fffbf.cn/888208.Doc
dga.fffbf.cn/646862.Doc
dga.fffbf.cn/606648.Doc
dga.fffbf.cn/373315.Doc
dgp.fffbf.cn/028202.Doc
dgp.fffbf.cn/393393.Doc
dgp.fffbf.cn/000000.Doc
dgp.fffbf.cn/666602.Doc
dgp.fffbf.cn/175555.Doc
dgp.fffbf.cn/880040.Doc
dgp.fffbf.cn/260660.Doc
dgp.fffbf.cn/262406.Doc
dgp.fffbf.cn/244002.Doc
dgp.fffbf.cn/288402.Doc
dgo.fffbf.cn/068628.Doc
dgo.fffbf.cn/824028.Doc
dgo.fffbf.cn/866600.Doc
dgo.fffbf.cn/264684.Doc
dgo.fffbf.cn/204608.Doc
dgo.fffbf.cn/082844.Doc
dgo.fffbf.cn/288644.Doc
dgo.fffbf.cn/993559.Doc
dgo.fffbf.cn/602846.Doc
dgo.fffbf.cn/513771.Doc
dgi.fffbf.cn/442266.Doc
dgi.fffbf.cn/206464.Doc
dgi.fffbf.cn/086406.Doc
dgi.fffbf.cn/646060.Doc
dgi.fffbf.cn/486428.Doc
dgi.fffbf.cn/462206.Doc
dgi.fffbf.cn/931597.Doc
dgi.fffbf.cn/084220.Doc
dgi.fffbf.cn/266248.Doc
dgi.fffbf.cn/800424.Doc
dgu.fffbf.cn/260024.Doc
dgu.fffbf.cn/068042.Doc
dgu.fffbf.cn/006464.Doc
dgu.fffbf.cn/806602.Doc
dgu.fffbf.cn/448682.Doc
dgu.fffbf.cn/660004.Doc
dgu.fffbf.cn/480468.Doc
dgu.fffbf.cn/135111.Doc
dgu.fffbf.cn/206080.Doc
dgu.fffbf.cn/882022.Doc
dgy.fffbf.cn/242480.Doc
dgy.fffbf.cn/260828.Doc
dgy.fffbf.cn/359999.Doc
dgy.fffbf.cn/224620.Doc
dgy.fffbf.cn/404008.Doc
dgy.fffbf.cn/222266.Doc
dgy.fffbf.cn/622464.Doc
dgy.fffbf.cn/846464.Doc
dgy.fffbf.cn/264044.Doc
dgy.fffbf.cn/262222.Doc
dgt.fffbf.cn/460822.Doc
dgt.fffbf.cn/064426.Doc
dgt.fffbf.cn/622260.Doc
dgt.fffbf.cn/620460.Doc
dgt.fffbf.cn/220228.Doc
dgt.fffbf.cn/646888.Doc
dgt.fffbf.cn/484020.Doc
dgt.fffbf.cn/644402.Doc
dgt.fffbf.cn/202864.Doc
dgt.fffbf.cn/765823.Doc
dgr.fffbf.cn/424088.Doc
dgr.fffbf.cn/067326.Doc
dgr.fffbf.cn/367661.Doc
dgr.fffbf.cn/166324.Doc
dgr.fffbf.cn/183699.Doc
dgr.fffbf.cn/224299.Doc
dgr.fffbf.cn/866998.Doc
dgr.fffbf.cn/389662.Doc
dgr.fffbf.cn/165455.Doc
dgr.fffbf.cn/848717.Doc
dge.fffbf.cn/998023.Doc
dge.fffbf.cn/772127.Doc
dge.fffbf.cn/491802.Doc
dge.fffbf.cn/394254.Doc
dge.fffbf.cn/979581.Doc
dge.fffbf.cn/911435.Doc
dge.fffbf.cn/048428.Doc
dge.fffbf.cn/915803.Doc
dge.fffbf.cn/462919.Doc
dge.fffbf.cn/080220.Doc
dgw.fffbf.cn/802028.Doc
dgw.fffbf.cn/260046.Doc
dgw.fffbf.cn/208804.Doc
dgw.fffbf.cn/460282.Doc
dgw.fffbf.cn/604622.Doc
dgw.fffbf.cn/466228.Doc
dgw.fffbf.cn/684846.Doc
dgw.fffbf.cn/408888.Doc
dgw.fffbf.cn/088088.Doc
dgw.fffbf.cn/860624.Doc
dgq.fffbf.cn/284848.Doc
dgq.fffbf.cn/886224.Doc
dgq.fffbf.cn/242880.Doc
dgq.fffbf.cn/044684.Doc
dgq.fffbf.cn/844640.Doc
dgq.fffbf.cn/242826.Doc
dgq.fffbf.cn/020802.Doc
dgq.fffbf.cn/644660.Doc
dgq.fffbf.cn/808040.Doc
dgq.fffbf.cn/246064.Doc
dfm.fffbf.cn/626828.Doc
dfm.fffbf.cn/028680.Doc
dfm.fffbf.cn/648446.Doc
dfm.fffbf.cn/242422.Doc
dfm.fffbf.cn/460028.Doc
dfm.fffbf.cn/688826.Doc
dfm.fffbf.cn/202048.Doc
dfm.fffbf.cn/426420.Doc
dfm.fffbf.cn/608888.Doc
dfm.fffbf.cn/822024.Doc
dfn.fffbf.cn/171577.Doc
dfn.fffbf.cn/226266.Doc
dfn.fffbf.cn/915731.Doc
dfn.fffbf.cn/882064.Doc
dfn.fffbf.cn/628060.Doc
dfn.fffbf.cn/400680.Doc
dfn.fffbf.cn/759911.Doc
dfn.fffbf.cn/262002.Doc
dfn.fffbf.cn/444466.Doc
dfn.fffbf.cn/777911.Doc
dfb.fffbf.cn/620680.Doc
dfb.fffbf.cn/046424.Doc
dfb.fffbf.cn/088408.Doc
dfb.fffbf.cn/602662.Doc
dfb.fffbf.cn/040802.Doc
dfb.fffbf.cn/808662.Doc
dfb.fffbf.cn/084846.Doc
dfb.fffbf.cn/800820.Doc
dfb.fffbf.cn/044804.Doc
dfb.fffbf.cn/820840.Doc
dfv.fffbf.cn/462228.Doc
dfv.fffbf.cn/066048.Doc
dfv.fffbf.cn/442644.Doc
dfv.fffbf.cn/668088.Doc
dfv.fffbf.cn/842464.Doc
dfv.fffbf.cn/848600.Doc
dfv.fffbf.cn/248646.Doc
dfv.fffbf.cn/664448.Doc
dfv.fffbf.cn/202644.Doc
dfv.fffbf.cn/626000.Doc
dfc.fffbf.cn/646640.Doc
dfc.fffbf.cn/462280.Doc
dfc.fffbf.cn/640862.Doc
dfc.fffbf.cn/702020.Doc
dfc.fffbf.cn/848800.Doc
dfc.fffbf.cn/660868.Doc
dfc.fffbf.cn/060020.Doc
dfc.fffbf.cn/622400.Doc
dfc.fffbf.cn/640842.Doc
dfc.fffbf.cn/664068.Doc
dfx.fffbf.cn/882480.Doc
dfx.fffbf.cn/086620.Doc
dfx.fffbf.cn/602686.Doc
dfx.fffbf.cn/004084.Doc
dfx.fffbf.cn/408844.Doc
dfx.fffbf.cn/084280.Doc
dfx.fffbf.cn/604440.Doc
dfx.fffbf.cn/202286.Doc
dfx.fffbf.cn/246244.Doc
dfx.fffbf.cn/204840.Doc
dfz.fffbf.cn/446680.Doc
dfz.fffbf.cn/486844.Doc
dfz.fffbf.cn/997733.Doc
dfz.fffbf.cn/604626.Doc
dfz.fffbf.cn/022246.Doc
dfz.fffbf.cn/353993.Doc
dfz.fffbf.cn/066462.Doc
dfz.fffbf.cn/424688.Doc
dfz.fffbf.cn/662286.Doc
dfz.fffbf.cn/642008.Doc
dfl.fffbf.cn/551575.Doc
dfl.fffbf.cn/864682.Doc
dfl.fffbf.cn/044646.Doc
dfl.fffbf.cn/046464.Doc
dfl.fffbf.cn/804600.Doc
dfl.fffbf.cn/060866.Doc
dfl.fffbf.cn/404004.Doc
dfl.fffbf.cn/660442.Doc
dfl.fffbf.cn/004224.Doc
dfl.fffbf.cn/446060.Doc
dfk.fffbf.cn/826224.Doc
dfk.fffbf.cn/286828.Doc
dfk.fffbf.cn/644468.Doc
dfk.fffbf.cn/046448.Doc
dfk.fffbf.cn/208860.Doc
dfk.fffbf.cn/131511.Doc
dfk.fffbf.cn/135393.Doc
dfk.fffbf.cn/606662.Doc
dfk.fffbf.cn/644868.Doc
dfk.fffbf.cn/644688.Doc
dfj.fffbf.cn/668222.Doc
dfj.fffbf.cn/600068.Doc
dfj.fffbf.cn/800068.Doc
dfj.fffbf.cn/022048.Doc
dfj.fffbf.cn/200082.Doc
dfj.fffbf.cn/660082.Doc
dfj.fffbf.cn/402068.Doc
dfj.fffbf.cn/226484.Doc
dfj.fffbf.cn/244806.Doc
dfj.fffbf.cn/206860.Doc
dfh.fffbf.cn/862002.Doc
dfh.fffbf.cn/602606.Doc
dfh.fffbf.cn/228464.Doc
dfh.fffbf.cn/408264.Doc
dfh.fffbf.cn/260280.Doc
dfh.fffbf.cn/044866.Doc
dfh.fffbf.cn/739579.Doc
dfh.fffbf.cn/084248.Doc
dfh.fffbf.cn/466280.Doc
dfh.fffbf.cn/888824.Doc
dfg.fffbf.cn/115951.Doc
dfg.fffbf.cn/600206.Doc
dfg.fffbf.cn/226224.Doc
dfg.fffbf.cn/466608.Doc
dfg.fffbf.cn/402808.Doc
dfg.fffbf.cn/448606.Doc
dfg.fffbf.cn/200864.Doc
dfg.fffbf.cn/860600.Doc
dfg.fffbf.cn/979933.Doc
dfg.fffbf.cn/333151.Doc
dff.fffbf.cn/480088.Doc
dff.fffbf.cn/482266.Doc
dff.fffbf.cn/680222.Doc
dff.fffbf.cn/886442.Doc
dff.fffbf.cn/448228.Doc
dff.fffbf.cn/046000.Doc
dff.fffbf.cn/422426.Doc
dff.fffbf.cn/460226.Doc
dff.fffbf.cn/771999.Doc
dff.fffbf.cn/066066.Doc
dfd.fffbf.cn/608448.Doc
dfd.fffbf.cn/440084.Doc
dfd.fffbf.cn/804088.Doc
dfd.fffbf.cn/880680.Doc
dfd.fffbf.cn/004420.Doc
dfd.fffbf.cn/224846.Doc
dfd.fffbf.cn/955997.Doc
dfd.fffbf.cn/064880.Doc
dfd.fffbf.cn/084028.Doc
dfd.fffbf.cn/406002.Doc
dfs.jouwir.cn/648260.Doc
dfs.jouwir.cn/242488.Doc
dfs.jouwir.cn/286066.Doc
dfs.jouwir.cn/866686.Doc
dfs.jouwir.cn/571931.Doc
dfs.jouwir.cn/808888.Doc
dfs.jouwir.cn/844620.Doc
dfs.jouwir.cn/088444.Doc
dfs.jouwir.cn/486804.Doc
dfs.jouwir.cn/040864.Doc
dfa.jouwir.cn/444026.Doc
dfa.jouwir.cn/040044.Doc
dfa.jouwir.cn/880660.Doc
dfa.jouwir.cn/688840.Doc
dfa.jouwir.cn/263590.Doc
dfa.jouwir.cn/438803.Doc
dfa.jouwir.cn/252338.Doc
dfa.jouwir.cn/819795.Doc
dfa.jouwir.cn/483309.Doc
dfa.jouwir.cn/915898.Doc
dfp.jouwir.cn/429347.Doc
dfp.jouwir.cn/840864.Doc
dfp.jouwir.cn/456812.Doc
dfp.jouwir.cn/422420.Doc
dfp.jouwir.cn/099940.Doc
dfp.jouwir.cn/557428.Doc
dfp.jouwir.cn/195432.Doc
dfp.jouwir.cn/825337.Doc
dfp.jouwir.cn/602659.Doc
dfp.jouwir.cn/392542.Doc
dfo.jouwir.cn/549728.Doc
dfo.jouwir.cn/464047.Doc
dfo.jouwir.cn/856660.Doc
dfo.jouwir.cn/020400.Doc
dfo.jouwir.cn/206240.Doc
dfo.jouwir.cn/020602.Doc
dfo.jouwir.cn/288020.Doc
dfo.jouwir.cn/408644.Doc
dfo.jouwir.cn/468404.Doc
dfo.jouwir.cn/220228.Doc
dfi.jouwir.cn/886664.Doc
dfi.jouwir.cn/600646.Doc
dfi.jouwir.cn/004286.Doc
dfi.jouwir.cn/488660.Doc
dfi.jouwir.cn/844044.Doc
dfi.jouwir.cn/806664.Doc
dfi.jouwir.cn/806244.Doc
dfi.jouwir.cn/648846.Doc
dfi.jouwir.cn/442240.Doc
dfi.jouwir.cn/684600.Doc
dfu.jouwir.cn/626226.Doc
dfu.jouwir.cn/280484.Doc
dfu.jouwir.cn/426204.Doc
dfu.jouwir.cn/408662.Doc
dfu.jouwir.cn/662860.Doc
dfu.jouwir.cn/226202.Doc
dfu.jouwir.cn/284406.Doc
dfu.jouwir.cn/608480.Doc
dfu.jouwir.cn/864884.Doc
dfu.jouwir.cn/266684.Doc
dfy.jouwir.cn/482802.Doc
dfy.jouwir.cn/460060.Doc
dfy.jouwir.cn/444460.Doc
dfy.jouwir.cn/402462.Doc
dfy.jouwir.cn/696349.Doc
dfy.jouwir.cn/266802.Doc
dfy.jouwir.cn/002864.Doc
dfy.jouwir.cn/060282.Doc
dfy.jouwir.cn/008260.Doc
dfy.jouwir.cn/828688.Doc
dft.jouwir.cn/866444.Doc
dft.jouwir.cn/200624.Doc
dft.jouwir.cn/062066.Doc
dft.jouwir.cn/264826.Doc
dft.jouwir.cn/600468.Doc
dft.jouwir.cn/888260.Doc
dft.jouwir.cn/848428.Doc
dft.jouwir.cn/262822.Doc
dft.jouwir.cn/882246.Doc
dft.jouwir.cn/826446.Doc
dfr.jouwir.cn/202846.Doc
dfr.jouwir.cn/828284.Doc
dfr.jouwir.cn/440840.Doc
dfr.jouwir.cn/064886.Doc
dfr.jouwir.cn/442668.Doc
dfr.jouwir.cn/088884.Doc
dfr.jouwir.cn/848200.Doc
dfr.jouwir.cn/848840.Doc
dfr.jouwir.cn/006822.Doc
dfr.jouwir.cn/442002.Doc
dfe.jouwir.cn/446600.Doc
dfe.jouwir.cn/042260.Doc
dfe.jouwir.cn/848466.Doc
dfe.jouwir.cn/808684.Doc
dfe.jouwir.cn/848688.Doc
