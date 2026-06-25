===== Python事件溯源模式实战 =====
——事件溯源将应用程序状态的所有变更存储为不可变的事件序列，而非直接存储当前状态

[1] 事件溯源核心概念
传统CRUD只保存当前状态，丢失了变更历史。事件溯源保存每次变更事件：
  · 事件(Event)：已发生的不可变事实，是状态的唯一来源
  · 聚合根(Aggregate Root)：通过重放事件重建业务对象
  · 事件存储(Event Store)：追加式日志，只插入不修改
  · 投影(Projection)：从事件流构建的读模型

[2] 事件定义与基础架构
事件是溯源模式的核心数据结构。

    from dataclasses import dataclass, field
    from typing import Dict, Any, List, Optional
    from datetime import datetime
    import uuid

    @dataclass
    class Event:
        """领域事件：表示已发生的业务事实，一旦创建不可修改"""
        event_id: str = field(default_factory=lambda: uuid.uuid4().hex)
        aggregate_id: str = ""        # 所属聚合ID
        event_type: str = ""          # 事件类型
        data: Dict[str, Any] = field(default_factory=dict)  # 事件数据
        version: int = 1              # 版本号(乐观锁)
        timestamp: datetime = field(default_factory=datetime.utcnow)

[3] 聚合根：事件重放恢复状态
聚合根通过apply方法逐步应用事件重建当前状态。

    class OrderAggregate:
        """订单聚合根：从事件流重建订单状态"""

        def __init__(self):
            self.id: Optional[str] = None
            self.status: str = "new"
            self.items: List[dict] = []
            self.total_amount: float = 0.0
            self._version: int = 0  # 当前版本号
            self._changes: List[Event] = []  # 未提交的新事件

        def apply(self, event: Event) -> None:
            """应用事件到聚合状态（事件重放的核心方法）"""
            if event.event_type == "OrderCreated":
                self.id = event.aggregate_id
                self.items = list(event.data.get("items", []))
                self.total_amount = event.data.get("total", 0.0)
                self.status = "pending"
            elif event.event_type == "OrderShipped":
                self.status = "shipped"
            elif event.event_type == "OrderCancelled":
                self.status = "cancelled"
            self._version += 1

        def create_order(self, items: List[dict], total: float) -> None:
            """创建订单的领域行为：验证并产生事件"""
            if not items:
                raise ValueError("订单必须包含商品")
            event = Event(
                aggregate_id=self.id or uuid.uuid4().hex,
                event_type="OrderCreated",
                data={"items": items, "total": total},
            )
            self._changes.append(event)  # 先暂存到未提交列表
            self.apply(event)            # 立即应用到当前状态

        def ship(self) -> None:
            """发货：状态必须为pending才能发货"""
            if self.status != "pending":
                raise ValueError(f"无法发货，当前状态: {self.status}")
            event = Event(
                aggregate_id=self.id,
                event_type="OrderShipped",
                data={},
            )
            self._changes.append(event)
            self.apply(event)

        def get_uncommitted_changes(self) -> List[Event]:
            """获取未持久化的新事件"""
            return list(self._changes)

[4] 事件存储：追加式日志
事件存储只追加不修改，支持事件溯源和审计。

    from collections import defaultdict

    class EventStore:
        """事件存储：追加式日志，事件的唯一真实来源"""

        def __init__(self):
            # 按聚合ID存储事件流
            self._streams: Dict[str, List[Event]] = defaultdict(list)
            # 全局事件日志（用于投影重建）
            self._global: List[Event] = []

        def append(self, aggregate_id: str, events: List[Event], expected_version: int):
            """追加事件到聚合流（乐观锁检查）"""
            current_version = len(self._streams[aggregate_id])
            if current_version != expected_version:
                raise ValueError(
                    f"并发冲突: 期望版本 {expected_version}，当前版本 {current_version}"
                )
            self._streams[aggregate_id].extend(events)
            self._global.extend(events)

        def get_events(self, aggregate_id: str) -> List[Event]:
            """获取指定聚合的事件流"""
            return list(self._streams.get(aggregate_id, []))

        def get_all_events(self) -> List[Event]:
            """获取所有事件（按时间排序）"""
            return sorted(self._global, key=lambda e: e.timestamp)

[5] 聚合重建函数
从事件存储中读取事件流重建聚合根到最新状态。

    def rebuild_aggregate(event_store: EventStore, aggregate_id: str) -> OrderAggregate:
        """从事件存储重建聚合根"""
        aggregate = OrderAggregate()
        events = event_store.get_events(aggregate_id)
        for event in events:
            aggregate.apply(event)  # 重放每个事件
        return aggregate  # 返回最新状态

[6] 快照机制：性能优化
当事件流过长时，每次重建都重放所有事件会导致性能下降。

    class SnapshotStore:
        """快照存储：保存聚合的某个时间点的状态快照"""

        def __init__(self):
            self._snapshots: Dict[str, tuple] = {}  # aggregate_id -> (version, state)

        def save_snapshot(self, aggregate_id: str, version: int, state: dict):
            """保存快照：记录某版本时的完整状态"""
            self._snapshots[aggregate_id] = (version, state)

        def get_snapshot(self, aggregate_id: str) -> Optional[tuple]:
            """获取快照：(版本号, 状态)"""
            return self._snapshots.get(aggregate_id)

    def rebuild_with_snapshot(event_store: EventStore, snap_store: SnapshotStore,
                              aggregate_id: str) -> OrderAggregate:
        """带快照的聚合重建：从快照版本开始，只重放快照之后的事件"""
        aggregate = OrderAggregate()
        snapshot = snap_store.get_snapshot(aggregate_id)
        start_version = 0
        if snapshot:
            # 从快照恢复状态
            version, state = snapshot
            aggregate.__dict__.update(state)
            start_version = version
        # 只重放快照之后的新事件
        events = event_store.get_events(aggregate_id)
        for event in events[start_version:]:
            aggregate.apply(event)
        return aggregate

[7] 事件升级(Upcasting)
随着业务演进，事件结构可能变化。升级机制兼容旧格式事件。

    def upcast_old_event(raw: dict) -> Event:
        """将旧格式事件升级到新版本"""
        if raw.get("event_type") == "OrderCreated" and raw.get("version", 1) == 1:
            # V1的data是items列表，V2增加了total字段
            data = raw["data"]
            if "total" not in data:
                data["total"] = sum(
                    item["price"] * item["qty"] for item in data.get("items", [])
                )
            raw["version"] = 2
        return Event(**raw)

[8] 投影/读模型构建
投影从事件流构建查询专用的读模型。

    class OrderProjection:
        """订单投影：从事件流构建读模型"""

        def __init__(self):
            self._orders: Dict[str, dict] = {}

        def project(self, event: Event):
            """处理事件，更新读模型"""
            if event.event_type == "OrderCreated":
                self._orders[event.aggregate_id] = {
                    "id": event.aggregate_id,
                    "items": event.data["items"],
                    "total": event.data["total"],
                    "status": "pending",
                }
            elif event.event_type == "OrderShipped":
                if event.aggregate_id in self._orders:
                    self._orders[event.aggregate_id]["status"] = "shipped"

        def rebuild_from_history(self, event_store: EventStore):
            """从全部历史事件重建读模型"""
            for event in event_store.get_all_events():
                self.project(event)

    if __name__ == "__main__":
        store = EventStore()
        snap_store = SnapshotStore()
        # 创建订单
        order = OrderAggregate()
        order.id = "ORD-001"
        order.create_order([{"sku": "A", "qty": 1, "price": 100}], 100.0)
        order.ship()
        # 持久化事件
        store.append(order.id, order.get_uncommitted_changes(), 0)
        # 快照
        snap_store.save_snapshot(order.id, 2, {"id": order.id, "status": order.status})
        # 重建验证
        restored = rebuild_with_snapshot(store, snap_store, "ORD-001")
        print(f"重建状态: {restored.status}")

etq.yghdsxcnbza.cn/42006.Doc
etq.yghdsxcnbza.cn/59975.Doc
etq.yghdsxcnbza.cn/68222.Doc
etq.yghdsxcnbza.cn/88820.Doc
etq.yghdsxcnbza.cn/88606.Doc
etq.yghdsxcnbza.cn/71319.Doc
etq.yghdsxcnbza.cn/86006.Doc
etq.yghdsxcnbza.cn/48604.Doc
etq.yghdsxcnbza.cn/46666.Doc
etq.yghdsxcnbza.cn/84606.Doc
erm.yghdsxcnbza.cn/80806.Doc
erm.yghdsxcnbza.cn/40088.Doc
erm.yghdsxcnbza.cn/71195.Doc
erm.yghdsxcnbza.cn/00640.Doc
erm.yghdsxcnbza.cn/22800.Doc
erm.yghdsxcnbza.cn/46404.Doc
erm.yghdsxcnbza.cn/80048.Doc
erm.yghdsxcnbza.cn/68446.Doc
erm.yghdsxcnbza.cn/68040.Doc
erm.yghdsxcnbza.cn/28826.Doc
ern.yghdsxcnbza.cn/22448.Doc
ern.yghdsxcnbza.cn/66446.Doc
ern.yghdsxcnbza.cn/84464.Doc
ern.yghdsxcnbza.cn/02004.Doc
ern.yghdsxcnbza.cn/82688.Doc
ern.yghdsxcnbza.cn/06046.Doc
ern.yghdsxcnbza.cn/08446.Doc
ern.yghdsxcnbza.cn/39715.Doc
ern.yghdsxcnbza.cn/86220.Doc
ern.yghdsxcnbza.cn/60046.Doc
erb.yghdsxcnbza.cn/08204.Doc
erb.yghdsxcnbza.cn/88642.Doc
erb.yghdsxcnbza.cn/28464.Doc
erb.yghdsxcnbza.cn/08282.Doc
erb.yghdsxcnbza.cn/82882.Doc
erb.yghdsxcnbza.cn/42442.Doc
erb.yghdsxcnbza.cn/84444.Doc
erb.yghdsxcnbza.cn/80660.Doc
erb.yghdsxcnbza.cn/04008.Doc
erb.yghdsxcnbza.cn/24288.Doc
erv.yghdsxcnbza.cn/68466.Doc
erv.yghdsxcnbza.cn/82846.Doc
erv.yghdsxcnbza.cn/40866.Doc
erv.yghdsxcnbza.cn/02884.Doc
erv.yghdsxcnbza.cn/06226.Doc
erv.yghdsxcnbza.cn/62480.Doc
erv.yghdsxcnbza.cn/40066.Doc
erv.yghdsxcnbza.cn/28848.Doc
erv.yghdsxcnbza.cn/66664.Doc
erv.yghdsxcnbza.cn/91957.Doc
erc.yghdsxcnbza.cn/46648.Doc
erc.yghdsxcnbza.cn/88640.Doc
erc.yghdsxcnbza.cn/26822.Doc
erc.yghdsxcnbza.cn/68682.Doc
erc.yghdsxcnbza.cn/40264.Doc
erc.yghdsxcnbza.cn/39513.Doc
erc.yghdsxcnbza.cn/24428.Doc
erc.yghdsxcnbza.cn/68800.Doc
erc.yghdsxcnbza.cn/46686.Doc
erc.yghdsxcnbza.cn/19119.Doc
erx.yghdsxcnbza.cn/46482.Doc
erx.yghdsxcnbza.cn/64664.Doc
erx.yghdsxcnbza.cn/62648.Doc
erx.yghdsxcnbza.cn/80680.Doc
erx.yghdsxcnbza.cn/42200.Doc
erx.yghdsxcnbza.cn/62406.Doc
erx.yghdsxcnbza.cn/26226.Doc
erx.yghdsxcnbza.cn/08622.Doc
erx.yghdsxcnbza.cn/86880.Doc
erx.yghdsxcnbza.cn/60048.Doc
erz.yghdsxcnbza.cn/06624.Doc
erz.yghdsxcnbza.cn/40262.Doc
erz.yghdsxcnbza.cn/62284.Doc
erz.yghdsxcnbza.cn/64462.Doc
erz.yghdsxcnbza.cn/86686.Doc
erz.yghdsxcnbza.cn/68462.Doc
erz.yghdsxcnbza.cn/80608.Doc
erz.yghdsxcnbza.cn/08460.Doc
erz.yghdsxcnbza.cn/04022.Doc
erz.yghdsxcnbza.cn/39131.Doc
erl.yghdsxcnbza.cn/20028.Doc
erl.yghdsxcnbza.cn/20268.Doc
erl.yghdsxcnbza.cn/64848.Doc
erl.yghdsxcnbza.cn/68864.Doc
erl.yghdsxcnbza.cn/40020.Doc
erl.yghdsxcnbza.cn/62828.Doc
erl.yghdsxcnbza.cn/02084.Doc
erl.yghdsxcnbza.cn/40602.Doc
erl.yghdsxcnbza.cn/02222.Doc
erl.yghdsxcnbza.cn/11577.Doc
erk.yghdsxcnbza.cn/48842.Doc
erk.yghdsxcnbza.cn/46622.Doc
erk.yghdsxcnbza.cn/48600.Doc
erk.yghdsxcnbza.cn/68804.Doc
erk.yghdsxcnbza.cn/20628.Doc
erk.yghdsxcnbza.cn/22288.Doc
erk.yghdsxcnbza.cn/48222.Doc
erk.yghdsxcnbza.cn/86480.Doc
erk.yghdsxcnbza.cn/68026.Doc
erk.yghdsxcnbza.cn/68826.Doc
erj.yghdsxcnbza.cn/40846.Doc
erj.yghdsxcnbza.cn/20448.Doc
erj.yghdsxcnbza.cn/86064.Doc
erj.yghdsxcnbza.cn/77373.Doc
erj.yghdsxcnbza.cn/48848.Doc
erj.yghdsxcnbza.cn/04268.Doc
erj.yghdsxcnbza.cn/22208.Doc
erj.yghdsxcnbza.cn/46600.Doc
erj.yghdsxcnbza.cn/08226.Doc
erj.yghdsxcnbza.cn/88846.Doc
erh.yghdsxcnbza.cn/00284.Doc
erh.yghdsxcnbza.cn/40626.Doc
erh.yghdsxcnbza.cn/40066.Doc
erh.yghdsxcnbza.cn/66624.Doc
erh.yghdsxcnbza.cn/00882.Doc
erh.yghdsxcnbza.cn/62020.Doc
erh.yghdsxcnbza.cn/62002.Doc
erh.yghdsxcnbza.cn/64666.Doc
erh.yghdsxcnbza.cn/88044.Doc
erh.yghdsxcnbza.cn/71759.Doc
erg.yghdsxcnbza.cn/40226.Doc
erg.yghdsxcnbza.cn/04844.Doc
erg.yghdsxcnbza.cn/64806.Doc
erg.yghdsxcnbza.cn/80608.Doc
erg.yghdsxcnbza.cn/02808.Doc
erg.yghdsxcnbza.cn/40682.Doc
erg.yghdsxcnbza.cn/79717.Doc
erg.yghdsxcnbza.cn/11595.Doc
erg.yghdsxcnbza.cn/28040.Doc
erg.yghdsxcnbza.cn/28440.Doc
erf.yghdsxcnbza.cn/28046.Doc
erf.yghdsxcnbza.cn/60844.Doc
erf.yghdsxcnbza.cn/00604.Doc
erf.yghdsxcnbza.cn/15577.Doc
erf.yghdsxcnbza.cn/80004.Doc
erf.yghdsxcnbza.cn/80822.Doc
erf.yghdsxcnbza.cn/08628.Doc
erf.yghdsxcnbza.cn/42222.Doc
erf.yghdsxcnbza.cn/04028.Doc
erf.yghdsxcnbza.cn/44680.Doc
erd.yghdsxcnbza.cn/88862.Doc
erd.yghdsxcnbza.cn/08864.Doc
erd.yghdsxcnbza.cn/59553.Doc
erd.yghdsxcnbza.cn/28840.Doc
erd.yghdsxcnbza.cn/28088.Doc
erd.yghdsxcnbza.cn/15137.Doc
erd.yghdsxcnbza.cn/44886.Doc
erd.yghdsxcnbza.cn/02448.Doc
erd.yghdsxcnbza.cn/88208.Doc
erd.yghdsxcnbza.cn/20888.Doc
ers.yghdsxcnbza.cn/84820.Doc
ers.yghdsxcnbza.cn/11139.Doc
ers.yghdsxcnbza.cn/28446.Doc
ers.yghdsxcnbza.cn/60006.Doc
ers.yghdsxcnbza.cn/26284.Doc
ers.yghdsxcnbza.cn/11779.Doc
ers.yghdsxcnbza.cn/82004.Doc
ers.yghdsxcnbza.cn/46686.Doc
ers.yghdsxcnbza.cn/40406.Doc
ers.yghdsxcnbza.cn/84886.Doc
era.yghdsxcnbza.cn/59359.Doc
era.yghdsxcnbza.cn/22080.Doc
era.yghdsxcnbza.cn/24800.Doc
era.yghdsxcnbza.cn/79113.Doc
era.yghdsxcnbza.cn/28462.Doc
era.yghdsxcnbza.cn/20048.Doc
era.yghdsxcnbza.cn/35397.Doc
era.yghdsxcnbza.cn/86806.Doc
era.yghdsxcnbza.cn/42600.Doc
era.yghdsxcnbza.cn/04480.Doc
erp.yghdsxcnbza.cn/40048.Doc
erp.yghdsxcnbza.cn/62482.Doc
erp.yghdsxcnbza.cn/02068.Doc
erp.yghdsxcnbza.cn/48068.Doc
erp.yghdsxcnbza.cn/62682.Doc
erp.yghdsxcnbza.cn/60224.Doc
erp.yghdsxcnbza.cn/68880.Doc
erp.yghdsxcnbza.cn/46420.Doc
erp.yghdsxcnbza.cn/86486.Doc
erp.yghdsxcnbza.cn/60228.Doc
ero.yghdsxcnbza.cn/19331.Doc
ero.yghdsxcnbza.cn/68422.Doc
ero.yghdsxcnbza.cn/48260.Doc
ero.yghdsxcnbza.cn/08820.Doc
ero.yghdsxcnbza.cn/86402.Doc
ero.yghdsxcnbza.cn/08202.Doc
ero.yghdsxcnbza.cn/64240.Doc
ero.yghdsxcnbza.cn/84008.Doc
ero.yghdsxcnbza.cn/40402.Doc
ero.yghdsxcnbza.cn/26060.Doc
eri.yghdsxcnbza.cn/86828.Doc
eri.yghdsxcnbza.cn/24686.Doc
eri.yghdsxcnbza.cn/04260.Doc
eri.yghdsxcnbza.cn/22062.Doc
eri.yghdsxcnbza.cn/46860.Doc
eri.yghdsxcnbza.cn/82804.Doc
eri.yghdsxcnbza.cn/08488.Doc
eri.yghdsxcnbza.cn/22826.Doc
eri.yghdsxcnbza.cn/80020.Doc
eri.yghdsxcnbza.cn/19335.Doc
eru.yghdsxcnbza.cn/88822.Doc
eru.yghdsxcnbza.cn/88828.Doc
eru.yghdsxcnbza.cn/46026.Doc
eru.yghdsxcnbza.cn/44222.Doc
eru.yghdsxcnbza.cn/66088.Doc
eru.yghdsxcnbza.cn/66668.Doc
eru.yghdsxcnbza.cn/08062.Doc
eru.yghdsxcnbza.cn/06246.Doc
eru.yghdsxcnbza.cn/64086.Doc
eru.yghdsxcnbza.cn/66860.Doc
ery.yghdsxcnbza.cn/26448.Doc
ery.yghdsxcnbza.cn/68608.Doc
ery.yghdsxcnbza.cn/28642.Doc
ery.yghdsxcnbza.cn/48844.Doc
ery.yghdsxcnbza.cn/20604.Doc
ery.yghdsxcnbza.cn/60840.Doc
ery.yghdsxcnbza.cn/44484.Doc
ery.yghdsxcnbza.cn/88644.Doc
ery.yghdsxcnbza.cn/82084.Doc
ery.yghdsxcnbza.cn/42222.Doc
ert.yghdsxcnbza.cn/24868.Doc
ert.yghdsxcnbza.cn/62440.Doc
ert.yghdsxcnbza.cn/40044.Doc
ert.yghdsxcnbza.cn/24084.Doc
ert.yghdsxcnbza.cn/60268.Doc
ert.yghdsxcnbza.cn/02820.Doc
ert.yghdsxcnbza.cn/06668.Doc
ert.yghdsxcnbza.cn/84804.Doc
ert.yghdsxcnbza.cn/20000.Doc
ert.yghdsxcnbza.cn/24444.Doc
err.yghdsxcnbza.cn/46666.Doc
err.yghdsxcnbza.cn/26864.Doc
err.yghdsxcnbza.cn/02602.Doc
err.yghdsxcnbza.cn/40004.Doc
err.yghdsxcnbza.cn/08628.Doc
err.yghdsxcnbza.cn/40260.Doc
err.yghdsxcnbza.cn/80660.Doc
err.yghdsxcnbza.cn/22648.Doc
err.yghdsxcnbza.cn/04826.Doc
err.yghdsxcnbza.cn/40482.Doc
ere.yghdsxcnbza.cn/04688.Doc
ere.yghdsxcnbza.cn/60866.Doc
ere.yghdsxcnbza.cn/84268.Doc
ere.yghdsxcnbza.cn/46448.Doc
ere.yghdsxcnbza.cn/28426.Doc
ere.yghdsxcnbza.cn/42202.Doc
ere.yghdsxcnbza.cn/57579.Doc
ere.yghdsxcnbza.cn/79195.Doc
ere.yghdsxcnbza.cn/02620.Doc
ere.yghdsxcnbza.cn/80086.Doc
erw.yghdsxcnbza.cn/44280.Doc
erw.yghdsxcnbza.cn/64400.Doc
erw.yghdsxcnbza.cn/80862.Doc
erw.yghdsxcnbza.cn/20260.Doc
erw.yghdsxcnbza.cn/60804.Doc
erw.yghdsxcnbza.cn/84422.Doc
erw.yghdsxcnbza.cn/86608.Doc
erw.yghdsxcnbza.cn/73731.Doc
erw.yghdsxcnbza.cn/88068.Doc
erw.yghdsxcnbza.cn/19959.Doc
erq.yghdsxcnbza.cn/66664.Doc
erq.yghdsxcnbza.cn/08640.Doc
erq.yghdsxcnbza.cn/22062.Doc
erq.yghdsxcnbza.cn/64028.Doc
erq.yghdsxcnbza.cn/46408.Doc
erq.yghdsxcnbza.cn/22664.Doc
erq.yghdsxcnbza.cn/68428.Doc
erq.yghdsxcnbza.cn/55735.Doc
erq.yghdsxcnbza.cn/64202.Doc
erq.yghdsxcnbza.cn/60068.Doc
eem.yghdsxcnbza.cn/04024.Doc
eem.yghdsxcnbza.cn/02268.Doc
eem.yghdsxcnbza.cn/80802.Doc
eem.yghdsxcnbza.cn/84008.Doc
eem.yghdsxcnbza.cn/20064.Doc
eem.yghdsxcnbza.cn/88264.Doc
eem.yghdsxcnbza.cn/48828.Doc
eem.yghdsxcnbza.cn/20226.Doc
eem.yghdsxcnbza.cn/08868.Doc
eem.yghdsxcnbza.cn/17111.Doc
een.yghdsxcnbza.cn/71973.Doc
een.yghdsxcnbza.cn/04668.Doc
een.yghdsxcnbza.cn/80264.Doc
een.yghdsxcnbza.cn/82200.Doc
een.yghdsxcnbza.cn/44488.Doc
een.yghdsxcnbza.cn/64244.Doc
een.yghdsxcnbza.cn/44228.Doc
een.yghdsxcnbza.cn/95773.Doc
een.yghdsxcnbza.cn/04460.Doc
een.yghdsxcnbza.cn/88862.Doc
eeb.yghdsxcnbza.cn/48600.Doc
eeb.yghdsxcnbza.cn/40440.Doc
eeb.yghdsxcnbza.cn/80600.Doc
eeb.yghdsxcnbza.cn/46244.Doc
eeb.yghdsxcnbza.cn/19555.Doc
eeb.yghdsxcnbza.cn/06486.Doc
eeb.yghdsxcnbza.cn/60888.Doc
eeb.yghdsxcnbza.cn/02844.Doc
eeb.yghdsxcnbza.cn/46800.Doc
eeb.yghdsxcnbza.cn/24464.Doc
eev.yghdsxcnbza.cn/60680.Doc
eev.yghdsxcnbza.cn/15979.Doc
eev.yghdsxcnbza.cn/04442.Doc
eev.yghdsxcnbza.cn/24882.Doc
eev.yghdsxcnbza.cn/22408.Doc
eev.yghdsxcnbza.cn/96868.Doc
eev.yghdsxcnbza.cn/60202.Doc
eev.yghdsxcnbza.cn/88462.Doc
eev.yghdsxcnbza.cn/28662.Doc
eev.yghdsxcnbza.cn/02422.Doc
eec.yghdsxcnbza.cn/46660.Doc
eec.yghdsxcnbza.cn/80448.Doc
eec.yghdsxcnbza.cn/68440.Doc
eec.yghdsxcnbza.cn/71973.Doc
eec.yghdsxcnbza.cn/68668.Doc
eec.yghdsxcnbza.cn/84082.Doc
eec.yghdsxcnbza.cn/84024.Doc
eec.yghdsxcnbza.cn/64004.Doc
eec.yghdsxcnbza.cn/02220.Doc
eec.yghdsxcnbza.cn/02406.Doc
eex.yghdsxcnbza.cn/71133.Doc
eex.yghdsxcnbza.cn/20208.Doc
eex.yghdsxcnbza.cn/62080.Doc
eex.yghdsxcnbza.cn/22680.Doc
eex.yghdsxcnbza.cn/02464.Doc
eex.yghdsxcnbza.cn/60062.Doc
eex.yghdsxcnbza.cn/40004.Doc
eex.yghdsxcnbza.cn/80426.Doc
eex.yghdsxcnbza.cn/02666.Doc
eex.yghdsxcnbza.cn/80486.Doc
eez.yghdsxcnbza.cn/20028.Doc
eez.yghdsxcnbza.cn/62668.Doc
eez.yghdsxcnbza.cn/55755.Doc
eez.yghdsxcnbza.cn/06862.Doc
eez.yghdsxcnbza.cn/44208.Doc
eez.yghdsxcnbza.cn/84826.Doc
eez.yghdsxcnbza.cn/80868.Doc
eez.yghdsxcnbza.cn/26260.Doc
eez.yghdsxcnbza.cn/46464.Doc
eez.yghdsxcnbza.cn/86866.Doc
eel.yghdsxcnbza.cn/00240.Doc
eel.yghdsxcnbza.cn/20004.Doc
eel.yghdsxcnbza.cn/17713.Doc
eel.yghdsxcnbza.cn/86808.Doc
eel.yghdsxcnbza.cn/22242.Doc
eel.yghdsxcnbza.cn/20060.Doc
eel.yghdsxcnbza.cn/26840.Doc
eel.yghdsxcnbza.cn/40628.Doc
eel.yghdsxcnbza.cn/42868.Doc
eel.yghdsxcnbza.cn/80462.Doc
eek.yghdsxcnbza.cn/24442.Doc
eek.yghdsxcnbza.cn/46802.Doc
eek.yghdsxcnbza.cn/82666.Doc
eek.yghdsxcnbza.cn/64000.Doc
eek.yghdsxcnbza.cn/68882.Doc
eek.yghdsxcnbza.cn/66660.Doc
eek.yghdsxcnbza.cn/15135.Doc
eek.yghdsxcnbza.cn/28682.Doc
eek.yghdsxcnbza.cn/20406.Doc
eek.yghdsxcnbza.cn/80240.Doc
eej.yghdsxcnbza.cn/35731.Doc
eej.yghdsxcnbza.cn/68824.Doc
eej.yghdsxcnbza.cn/22626.Doc
eej.yghdsxcnbza.cn/88088.Doc
eej.yghdsxcnbza.cn/04008.Doc
eej.yghdsxcnbza.cn/04640.Doc
eej.yghdsxcnbza.cn/26082.Doc
eej.yghdsxcnbza.cn/20244.Doc
eej.yghdsxcnbza.cn/44244.Doc
eej.yghdsxcnbza.cn/66066.Doc
eeh.yghdsxcnbza.cn/80866.Doc
eeh.yghdsxcnbza.cn/84202.Doc
eeh.yghdsxcnbza.cn/82880.Doc
eeh.yghdsxcnbza.cn/24848.Doc
eeh.yghdsxcnbza.cn/02228.Doc
eeh.yghdsxcnbza.cn/22840.Doc
eeh.yghdsxcnbza.cn/40800.Doc
eeh.yghdsxcnbza.cn/82268.Doc
eeh.yghdsxcnbza.cn/40480.Doc
eeh.yghdsxcnbza.cn/00264.Doc
eeg.yghdsxcnbza.cn/42442.Doc
eeg.yghdsxcnbza.cn/22482.Doc
eeg.yghdsxcnbza.cn/80080.Doc
eeg.yghdsxcnbza.cn/42888.Doc
eeg.yghdsxcnbza.cn/28480.Doc
eeg.yghdsxcnbza.cn/60288.Doc
eeg.yghdsxcnbza.cn/53759.Doc
eeg.yghdsxcnbza.cn/62084.Doc
eeg.yghdsxcnbza.cn/26400.Doc
eeg.yghdsxcnbza.cn/84246.Doc
eef.yghdsxcnbza.cn/64688.Doc
eef.yghdsxcnbza.cn/04662.Doc
eef.yghdsxcnbza.cn/06660.Doc
eef.yghdsxcnbza.cn/66480.Doc
eef.yghdsxcnbza.cn/04804.Doc
eef.yghdsxcnbza.cn/75335.Doc
eef.yghdsxcnbza.cn/66088.Doc
eef.yghdsxcnbza.cn/26282.Doc
eef.yghdsxcnbza.cn/28662.Doc
eef.yghdsxcnbza.cn/24402.Doc
eed.yghdsxcnbza.cn/22628.Doc
eed.yghdsxcnbza.cn/04884.Doc
eed.yghdsxcnbza.cn/82608.Doc
eed.yghdsxcnbza.cn/26066.Doc
eed.yghdsxcnbza.cn/00044.Doc
eed.yghdsxcnbza.cn/44888.Doc
eed.yghdsxcnbza.cn/00620.Doc
eed.yghdsxcnbza.cn/20068.Doc
eed.yghdsxcnbza.cn/00666.Doc
eed.yghdsxcnbza.cn/22240.Doc
ees.yghdsxcnbza.cn/26866.Doc
ees.yghdsxcnbza.cn/62282.Doc
ees.yghdsxcnbza.cn/68406.Doc
ees.yghdsxcnbza.cn/04084.Doc
ees.yghdsxcnbza.cn/60220.Doc
ees.yghdsxcnbza.cn/02400.Doc
ees.yghdsxcnbza.cn/00804.Doc
ees.yghdsxcnbza.cn/66862.Doc
ees.yghdsxcnbza.cn/71755.Doc
ees.yghdsxcnbza.cn/40604.Doc
eea.yghdsxcnbza.cn/64224.Doc
eea.yghdsxcnbza.cn/04826.Doc
eea.yghdsxcnbza.cn/48080.Doc
eea.yghdsxcnbza.cn/66084.Doc
eea.yghdsxcnbza.cn/06804.Doc
eea.yghdsxcnbza.cn/66026.Doc
eea.yghdsxcnbza.cn/37779.Doc
eea.yghdsxcnbza.cn/24006.Doc
eea.yghdsxcnbza.cn/44246.Doc
eea.yghdsxcnbza.cn/73371.Doc
eep.yghdsxcnbza.cn/60680.Doc
eep.yghdsxcnbza.cn/28642.Doc
eep.yghdsxcnbza.cn/06606.Doc
eep.yghdsxcnbza.cn/82442.Doc
eep.yghdsxcnbza.cn/62844.Doc
eep.yghdsxcnbza.cn/42466.Doc
eep.yghdsxcnbza.cn/06080.Doc
eep.yghdsxcnbza.cn/04828.Doc
eep.yghdsxcnbza.cn/28484.Doc
eep.yghdsxcnbza.cn/22042.Doc
eeo.yghdsxcnbza.cn/00222.Doc
eeo.yghdsxcnbza.cn/48668.Doc
eeo.yghdsxcnbza.cn/51377.Doc
eeo.yghdsxcnbza.cn/80266.Doc
eeo.yghdsxcnbza.cn/73555.Doc
eeo.yghdsxcnbza.cn/31715.Doc
eeo.yghdsxcnbza.cn/17959.Doc
eeo.yghdsxcnbza.cn/82622.Doc
eeo.yghdsxcnbza.cn/84624.Doc
eeo.yghdsxcnbza.cn/24888.Doc
eei.yghdsxcnbza.cn/22204.Doc
eei.yghdsxcnbza.cn/26262.Doc
eei.yghdsxcnbza.cn/02066.Doc
eei.yghdsxcnbza.cn/06420.Doc
eei.yghdsxcnbza.cn/42260.Doc
eei.yghdsxcnbza.cn/60004.Doc
eei.yghdsxcnbza.cn/48624.Doc
eei.yghdsxcnbza.cn/66628.Doc
eei.yghdsxcnbza.cn/80640.Doc
eei.yghdsxcnbza.cn/08264.Doc
eeu.yghdsxcnbza.cn/40206.Doc
eeu.yghdsxcnbza.cn/48662.Doc
eeu.yghdsxcnbza.cn/46224.Doc
eeu.yghdsxcnbza.cn/75171.Doc
eeu.yghdsxcnbza.cn/08842.Doc
eeu.yghdsxcnbza.cn/42626.Doc
eeu.yghdsxcnbza.cn/46264.Doc
eeu.yghdsxcnbza.cn/26006.Doc
eeu.yghdsxcnbza.cn/17395.Doc
eeu.yghdsxcnbza.cn/02200.Doc
eey.yghdsxcnbza.cn/68086.Doc
eey.yghdsxcnbza.cn/64806.Doc
eey.yghdsxcnbza.cn/82864.Doc
eey.yghdsxcnbza.cn/24286.Doc
eey.yghdsxcnbza.cn/44648.Doc
eey.yghdsxcnbza.cn/33733.Doc
eey.yghdsxcnbza.cn/02686.Doc
eey.yghdsxcnbza.cn/66260.Doc
eey.yghdsxcnbza.cn/24062.Doc
eey.yghdsxcnbza.cn/62682.Doc
eet.yghdsxcnbza.cn/44040.Doc
eet.yghdsxcnbza.cn/08662.Doc
eet.yghdsxcnbza.cn/33717.Doc
eet.yghdsxcnbza.cn/26486.Doc
eet.yghdsxcnbza.cn/80486.Doc
eet.yghdsxcnbza.cn/08068.Doc
eet.yghdsxcnbza.cn/24282.Doc
eet.yghdsxcnbza.cn/86022.Doc
eet.yghdsxcnbza.cn/99977.Doc
eet.yghdsxcnbza.cn/24846.Doc
