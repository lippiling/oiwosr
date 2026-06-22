日夷资媒敝


===== Python CQRS模式基础实战 =====
——命令查询职责分离(CQRS)将读操作和写操作分离为不同的模型，优化各自性能

[1] CQRS核心理念
传统CRUD使用同一模型读写数据，CQRS则拆分为：
  · 命令(Command)：改变状态的操作，不返回数据，强调验证和一致性
  · 查询(Query)：读取状态的操作，不修改数据，强调性能和灵活性

分离带来的优势：读模型可针对查询优化(如反规范化)，写模型可聚焦业务规则。

[2] 命令(Command)定义
命令是不可变的数据传输对象，表达用户的意图。

    from dataclasses import dataclass, field
    from typing import Any, Dict, Optional
    from datetime import datetime
    import uuid

    @dataclass(frozen=True)  # frozen=True 使命令不可变
    class Command:
        """命令基类：表达"做什么"，不可变且必须通过验证"""
        command_id: str = field(default_factory=lambda: uuid.uuid4().hex)
        timestamp: datetime = field(default_factory=datetime.utcnow)

    @dataclass(frozen=True)
    class CreateOrderCommand(Command):
        """创建订单命令：包含创建订单所需的所有数据"""
        customer_id: str
        items: list  # 商品列表 [{sku, qty, price}]
        shipping_address: str

        def validate(self) -> Optional[str]:
            """业务验证：返回错误信息或None表示验证通过"""
            if not self.customer_id:
                return "客户ID不能为空"
            if not self.items:
                return "订单至少包含一个商品"
            if any(item["qty"] <= 0 for item in self.items):
                return "商品数量必须大于0"
            return None  # 验证通过

    @dataclass(frozen=True)
    class CancelOrderCommand(Command):
        """取消订单命令"""
        order_id: str
        reason: str

[3] 查询(Query)定义
查询不改变状态，可以按需设计数据结构。

    @dataclass(frozen=True)
    class Query:
        """查询基类：表达"问什么"，无副作用"""
        query_id: str = field(default_factory=lambda: uuid.uuid4().hex)

    @dataclass(frozen=True)
    class GetOrderQuery(Query):
        """获取订单查询：按ID查找"""
        order_id: str

    @dataclass(frozen=True)
    class ListCustomerOrdersQuery(Query):
        """列出客户订单查询：带分页"""
        customer_id: str
        page: int = 1
        page_size: int = 20

[4] 命令处理器协议
命令处理器负责执行命令中的业务逻辑。

    from typing import Protocol, List

    class CommandHandler(Protocol):
        """命令处理器协议：所有命令处理器必须实现execute方法"""
        def execute(self, command: Command) -> None: ...

    class CreateOrderHandler:
        """创建订单命令处理器：执行业务逻辑"""

        def __init__(self, write_repo):
            self._repo = write_repo  # 写模型仓库

        def execute(self, command: CreateOrderCommand) -> None:
            """执行创建订单：验证→构建→保存"""
            # 1. 验证命令
            error = command.validate()
            if error:
                raise ValueError(f"命令验证失败: {error}")
            # 2. 构建订单聚合
            order = {
                "order_id": command.command_id,
                "customer_id": command.customer_id,
                "items": command.items,
                "status": "pending",
                "created_at": command.timestamp,
            }
            # 3. 保存到写模型(GET / POST)
            self._repo.save(order)

    class CancelOrderHandler:
        """取消订单命令处理器"""

        def __init__(self, write_repo):
            self._repo = write_repo

        def execute(self, command: CancelOrderCommand) -> None:
            order = self._repo.find_by_id(command.order_id)
            if not order:
                raise ValueError(f"订单不存在: {command.order_id}")
            if order["status"] == "shipped":
                raise ValueError("已发货订单不能取消")
            order["status"] = "cancelled"
            order["cancel_reason"] = command.reason
            self._repo.save(order)

[5] 查询处理器协议
查询处理器从读模型获取数据，不涉及业务逻辑。

    class QueryHandler(Protocol):
        """查询处理器协议"""
        def execute(self, query: Query) -> Any: ...

    class GetOrderHandler:
        """订单查询处理器：从读模型获取数据"""

        def __init__(self, read_repo):
            self._read_repo = read_repo  # 读模型仓库

        def execute(self, query: GetOrderQuery) -> Optional[Dict[str, Any]]:
            """从读模型直接获取结果"""
            return self._read_repo.find_by_id(query.order_id)

[6] 命令总线与查询总线分离
CQRS的核心在于命令总线和查询总线是独立的两条路径。

    class CommandBus:
        """命令总线：路由命令到对应的处理器"""

        def __init__(self):
            self._handlers: Dict[str, CommandHandler] = {}

        def register(self, command_type: str, handler: CommandHandler):
            """注册命令类型及其处理器"""
            self._handlers[command_type] = handler

        def dispatch(self, command: Command) -> None:
            """分发命令到注册的处理器"""
            handler = self._handlers.get(type(command).__name__)
            if not handler:
                raise ValueError(f"未注册的命令: {type(command).__name__}")
            handler.execute(command)

    class QueryBus:
        """查询总线：路由查询到对应的查询处理器"""

        def __init__(self):
            self._handlers: Dict[str, QueryHandler] = {}

        def register(self, query_type: str, handler: QueryHandler):
            self._handlers[query_type] = handler

        def dispatch(self, query: Query) -> Any:
            """分发查询并返回结果"""
            handler = self._handlers.get(type(query).__name__)
            if not handler:
                raise ValueError(f"未注册的查询: {type(query).__name__}")
            return handler.execute(query)

[7] 读模型反规范化
读模型为查询效率对数据进行预处理和扁平化。

    class OrderReadModel:
        """反规范化的订单读模型：将关联数据合并，减少关联查询"""

        def __init__(self):
            self._orders: Dict[str, dict] = {}

        def denormalize_and_save(self, order_data: dict, customer_name: str):
            """将订单数据和客户名合并为扁平结构存储"""
            read_model = {
                "id": order_data["order_id"],
                "customer_name": customer_name,  # 直接冗余存储，避免JOIN
                "total_items": len(order_data["items"]),
                "total_amount": sum(
                    i["qty"] * i["price"] for i in order_data["items"]
                ),
                "status": order_data["status"],
                "created_at": order_data["created_at"].isoformat(),
            }
            self._orders[read_model["id"]] = read_model

        def find_by_id(self, order_id: str) -> Optional[dict]:
            return self._orders.get(order_id)

[8] 何时使用CQRS
CQRS适合以下场景：读写负载极不对称、读模型需要多维度查询、团队可独立演进读写两端。
不适合简单CRUD应用——过度设计会引入不必要的复杂度。

    if __name__ == "__main__":
        # 初始化写模型和读模型
        write_repo = {}  # 模拟写存储
        read_model = OrderReadModel()

        # 组建命令总线
        cmd_bus = CommandBus()
        cmd_bus.register("CreateOrderCommand", CreateOrderHandler(write_repo))
        cmd_bus.register("CancelOrderCommand", CancelOrderHandler(write_repo))

        # 组建查询总线
        query_bus = QueryBus()
        query_bus.register("GetOrderQuery", GetOrderHandler(read_model))

        # 使用命令创建订单(写)
        cmd = CreateOrderCommand(
            customer_id="CUST-001",
            items=[{"sku": "BOOK-01", "qty": 2, "price": 49.0}],
            shipping_address="北京市朝阳区",
        )
        cmd_bus.dispatch(cmd)

        # 更新读模型(通常在事件处理器中完成)
        read_model.denormalize_and_save(
            write_repo[cmd.command_id], "张三"
        )

        # 使用查询获取数据(读)
        result = query_bus.dispatch(GetOrderQuery(order_id=cmd.command_id))
        print(f"读模型查询结果: {result}")

裙诱撤潭栈纫傲赡伟们拘僮惺谟套

dxl.irvnp.cn/604222.Doc
dxk.irvnp.cn/391519.Doc
dxk.irvnp.cn/600226.Doc
dxk.irvnp.cn/402460.Doc
dxk.irvnp.cn/066688.Doc
dxk.irvnp.cn/226260.Doc
dxk.irvnp.cn/793111.Doc
dxk.irvnp.cn/002482.Doc
dxk.irvnp.cn/911739.Doc
dxk.irvnp.cn/260484.Doc
dxk.irvnp.cn/628608.Doc
dxj.irvnp.cn/044488.Doc
dxj.irvnp.cn/804686.Doc
dxj.irvnp.cn/339739.Doc
dxj.irvnp.cn/044226.Doc
dxj.irvnp.cn/359355.Doc
dxj.irvnp.cn/682026.Doc
dxj.irvnp.cn/157755.Doc
dxj.irvnp.cn/593179.Doc
dxj.irvnp.cn/951119.Doc
dxj.irvnp.cn/662220.Doc
dxh.irvnp.cn/828082.Doc
dxh.irvnp.cn/642446.Doc
dxh.irvnp.cn/206844.Doc
dxh.irvnp.cn/668648.Doc
dxh.irvnp.cn/335199.Doc
dxh.irvnp.cn/440468.Doc
dxh.irvnp.cn/157791.Doc
dxh.irvnp.cn/280862.Doc
dxh.irvnp.cn/622466.Doc
dxh.irvnp.cn/226206.Doc
dxg.irvnp.cn/620226.Doc
dxg.irvnp.cn/606020.Doc
dxg.irvnp.cn/260844.Doc
dxg.irvnp.cn/842004.Doc
dxg.irvnp.cn/044846.Doc
dxg.irvnp.cn/028000.Doc
dxg.irvnp.cn/840044.Doc
dxg.irvnp.cn/266064.Doc
dxg.irvnp.cn/840642.Doc
dxf.irvnp.cn/444820.Doc
dxf.irvnp.cn/442202.Doc
dxf.irvnp.cn/224800.Doc
dxf.irvnp.cn/066866.Doc
dxf.irvnp.cn/842446.Doc
dxf.irvnp.cn/266022.Doc
dxf.irvnp.cn/422286.Doc
dxf.irvnp.cn/193997.Doc
dxf.irvnp.cn/064884.Doc
dxf.irvnp.cn/686640.Doc
dxd.irvnp.cn/826802.Doc
dxd.irvnp.cn/191315.Doc
dxd.irvnp.cn/171953.Doc
dxd.irvnp.cn/624204.Doc
dxd.irvnp.cn/286480.Doc
dxd.irvnp.cn/244428.Doc
dxd.irvnp.cn/571357.Doc
dxd.irvnp.cn/882066.Doc
dxd.irvnp.cn/797951.Doc
dxd.irvnp.cn/442040.Doc
dxs.irvnp.cn/826204.Doc
dxs.irvnp.cn/733171.Doc
dxs.irvnp.cn/933595.Doc
dxs.irvnp.cn/820684.Doc
dxs.irvnp.cn/446028.Doc
dxs.irvnp.cn/480266.Doc
dxs.irvnp.cn/486622.Doc
dxs.irvnp.cn/428868.Doc
dxs.irvnp.cn/513359.Doc
dxs.irvnp.cn/802488.Doc
dxa.irvnp.cn/777357.Doc
dxa.irvnp.cn/131711.Doc
dxa.irvnp.cn/408060.Doc
dxa.irvnp.cn/028844.Doc
dxa.irvnp.cn/440282.Doc
dxa.irvnp.cn/602840.Doc
dxa.irvnp.cn/848248.Doc
dxa.irvnp.cn/462204.Doc
dxa.irvnp.cn/575333.Doc
dxa.irvnp.cn/664248.Doc
dxp.irvnp.cn/379359.Doc
dxp.irvnp.cn/400860.Doc
dxp.irvnp.cn/866020.Doc
dxp.irvnp.cn/422246.Doc
dxp.irvnp.cn/882426.Doc
dxp.irvnp.cn/606000.Doc
dxp.irvnp.cn/604880.Doc
dxp.irvnp.cn/206288.Doc
dxp.irvnp.cn/373935.Doc
dxp.irvnp.cn/262620.Doc
dxo.irvnp.cn/608640.Doc
dxo.irvnp.cn/082402.Doc
dxo.irvnp.cn/175373.Doc
dxo.irvnp.cn/662866.Doc
dxo.irvnp.cn/408464.Doc
dxo.irvnp.cn/662804.Doc
dxo.irvnp.cn/802042.Doc
dxo.irvnp.cn/951153.Doc
dxo.irvnp.cn/482804.Doc
dxo.irvnp.cn/220802.Doc
dxi.irvnp.cn/606446.Doc
dxi.irvnp.cn/802602.Doc
dxi.irvnp.cn/806642.Doc
dxi.irvnp.cn/200848.Doc
dxi.irvnp.cn/755755.Doc
dxi.irvnp.cn/660602.Doc
dxi.irvnp.cn/422804.Doc
dxi.irvnp.cn/597573.Doc
dxi.irvnp.cn/404640.Doc
dxi.irvnp.cn/484822.Doc
dxu.irvnp.cn/002406.Doc
dxu.irvnp.cn/448028.Doc
dxu.irvnp.cn/440206.Doc
dxu.irvnp.cn/880204.Doc
dxu.irvnp.cn/426444.Doc
dxu.irvnp.cn/486668.Doc
dxu.irvnp.cn/460228.Doc
dxu.irvnp.cn/486824.Doc
dxu.irvnp.cn/284404.Doc
dxu.irvnp.cn/808028.Doc
dxy.irvnp.cn/448424.Doc
dxy.irvnp.cn/606648.Doc
dxy.irvnp.cn/353119.Doc
dxy.irvnp.cn/660082.Doc
dxy.irvnp.cn/331151.Doc
dxy.irvnp.cn/044640.Doc
dxy.irvnp.cn/842511.Doc
dxy.irvnp.cn/044200.Doc
dxy.irvnp.cn/422882.Doc
dxy.irvnp.cn/848262.Doc
dxt.irvnp.cn/115553.Doc
dxt.irvnp.cn/939117.Doc
dxt.irvnp.cn/468624.Doc
dxt.irvnp.cn/868206.Doc
dxt.irvnp.cn/006266.Doc
dxt.irvnp.cn/202444.Doc
dxt.irvnp.cn/088000.Doc
dxt.irvnp.cn/028084.Doc
dxt.irvnp.cn/068026.Doc
dxt.irvnp.cn/773115.Doc
dxr.irvnp.cn/224840.Doc
dxr.irvnp.cn/846024.Doc
dxr.irvnp.cn/820084.Doc
dxr.irvnp.cn/424406.Doc
dxr.irvnp.cn/284666.Doc
dxr.irvnp.cn/402444.Doc
dxr.irvnp.cn/260282.Doc
dxr.irvnp.cn/028644.Doc
dxr.irvnp.cn/020660.Doc
dxr.irvnp.cn/442808.Doc
dxe.irvnp.cn/460448.Doc
dxe.irvnp.cn/086044.Doc
dxe.irvnp.cn/715531.Doc
dxe.irvnp.cn/800802.Doc
dxe.irvnp.cn/688440.Doc
dxe.irvnp.cn/373759.Doc
dxe.irvnp.cn/840228.Doc
dxe.irvnp.cn/868042.Doc
dxe.irvnp.cn/606082.Doc
dxe.irvnp.cn/400406.Doc
dxw.irvnp.cn/068024.Doc
dxw.irvnp.cn/628268.Doc
dxw.irvnp.cn/224844.Doc
dxw.irvnp.cn/220404.Doc
dxw.irvnp.cn/466422.Doc
dxw.irvnp.cn/888646.Doc
dxw.irvnp.cn/220086.Doc
dxw.irvnp.cn/595579.Doc
dxw.irvnp.cn/600262.Doc
dxw.irvnp.cn/842484.Doc
dxq.irvnp.cn/684640.Doc
dxq.irvnp.cn/111975.Doc
dxq.irvnp.cn/204666.Doc
dxq.irvnp.cn/886824.Doc
dxq.irvnp.cn/860020.Doc
dxq.irvnp.cn/208004.Doc
dxq.irvnp.cn/886868.Doc
dxq.irvnp.cn/262400.Doc
dxq.irvnp.cn/688204.Doc
dxq.irvnp.cn/068486.Doc
dzm.irvnp.cn/608426.Doc
dzm.irvnp.cn/751195.Doc
dzm.irvnp.cn/842408.Doc
dzm.irvnp.cn/468406.Doc
dzm.irvnp.cn/482680.Doc
dzm.irvnp.cn/975793.Doc
dzm.irvnp.cn/886064.Doc
dzm.irvnp.cn/242682.Doc
dzm.irvnp.cn/888248.Doc
dzm.irvnp.cn/822868.Doc
dzn.irvnp.cn/640040.Doc
dzn.irvnp.cn/600686.Doc
dzn.irvnp.cn/660620.Doc
dzn.irvnp.cn/020406.Doc
dzn.irvnp.cn/997355.Doc
dzn.irvnp.cn/246028.Doc
dzn.irvnp.cn/442880.Doc
dzn.irvnp.cn/260822.Doc
dzn.irvnp.cn/088060.Doc
dzn.irvnp.cn/888840.Doc
dzb.irvnp.cn/040824.Doc
dzb.irvnp.cn/115991.Doc
dzb.irvnp.cn/204866.Doc
dzb.irvnp.cn/668066.Doc
dzb.irvnp.cn/242062.Doc
dzb.irvnp.cn/066624.Doc
dzb.irvnp.cn/448806.Doc
dzb.irvnp.cn/717719.Doc
dzb.irvnp.cn/628642.Doc
dzb.irvnp.cn/242482.Doc
dzv.irvnp.cn/337377.Doc
dzv.irvnp.cn/884482.Doc
dzv.irvnp.cn/266808.Doc
dzv.irvnp.cn/406844.Doc
dzv.irvnp.cn/448408.Doc
dzv.irvnp.cn/939577.Doc
dzv.irvnp.cn/044664.Doc
dzv.irvnp.cn/088244.Doc
dzv.irvnp.cn/840204.Doc
dzv.irvnp.cn/840888.Doc
dzc.irvnp.cn/048000.Doc
dzc.irvnp.cn/220220.Doc
dzc.irvnp.cn/406262.Doc
dzc.irvnp.cn/686444.Doc
dzc.irvnp.cn/280488.Doc
dzc.irvnp.cn/971711.Doc
dzc.irvnp.cn/664264.Doc
dzc.irvnp.cn/288466.Doc
dzc.irvnp.cn/204860.Doc
dzc.irvnp.cn/448482.Doc
dzx.irvnp.cn/046264.Doc
dzx.irvnp.cn/640228.Doc
dzx.irvnp.cn/482242.Doc
dzx.irvnp.cn/064088.Doc
dzx.irvnp.cn/642840.Doc
dzx.irvnp.cn/064262.Doc
dzx.irvnp.cn/248006.Doc
dzx.irvnp.cn/480042.Doc
dzx.irvnp.cn/040868.Doc
dzx.irvnp.cn/886688.Doc
dzz.irvnp.cn/468620.Doc
dzz.irvnp.cn/048402.Doc
dzz.irvnp.cn/791791.Doc
dzz.irvnp.cn/422260.Doc
dzz.irvnp.cn/466224.Doc
dzz.irvnp.cn/911991.Doc
dzz.irvnp.cn/804620.Doc
dzz.irvnp.cn/884046.Doc
dzz.irvnp.cn/440660.Doc
dzz.irvnp.cn/006022.Doc
dzl.irvnp.cn/719777.Doc
dzl.irvnp.cn/026684.Doc
dzl.irvnp.cn/840062.Doc
dzl.irvnp.cn/391977.Doc
dzl.irvnp.cn/931791.Doc
dzl.irvnp.cn/464466.Doc
dzl.irvnp.cn/440488.Doc
dzl.irvnp.cn/171975.Doc
dzl.irvnp.cn/200040.Doc
dzl.irvnp.cn/048464.Doc
dzk.irvnp.cn/882404.Doc
dzk.irvnp.cn/284862.Doc
dzk.irvnp.cn/682682.Doc
dzk.irvnp.cn/044400.Doc
dzk.irvnp.cn/664404.Doc
dzk.irvnp.cn/682020.Doc
dzk.irvnp.cn/939153.Doc
dzk.irvnp.cn/268088.Doc
dzk.irvnp.cn/002820.Doc
dzk.irvnp.cn/400460.Doc
dzj.irvnp.cn/860628.Doc
dzj.irvnp.cn/204402.Doc
dzj.irvnp.cn/406844.Doc
dzj.irvnp.cn/626448.Doc
dzj.irvnp.cn/951755.Doc
dzj.irvnp.cn/284044.Doc
dzj.irvnp.cn/539577.Doc
dzj.irvnp.cn/606886.Doc
dzj.irvnp.cn/446226.Doc
dzj.irvnp.cn/666060.Doc
dzh.irvnp.cn/004664.Doc
dzh.irvnp.cn/115157.Doc
dzh.irvnp.cn/773191.Doc
dzh.irvnp.cn/597937.Doc
dzh.irvnp.cn/959799.Doc
dzh.irvnp.cn/880244.Doc
dzh.irvnp.cn/468688.Doc
dzh.irvnp.cn/624840.Doc
dzh.irvnp.cn/002040.Doc
dzh.irvnp.cn/842206.Doc
dzg.irvnp.cn/622062.Doc
dzg.irvnp.cn/028260.Doc
dzg.irvnp.cn/284684.Doc
dzg.irvnp.cn/448246.Doc
dzg.irvnp.cn/600806.Doc
dzg.irvnp.cn/115155.Doc
dzg.irvnp.cn/355131.Doc
dzg.irvnp.cn/200004.Doc
dzg.irvnp.cn/622402.Doc
dzg.irvnp.cn/628048.Doc
dzf.irvnp.cn/842642.Doc
dzf.irvnp.cn/048006.Doc
dzf.irvnp.cn/602060.Doc
dzf.irvnp.cn/179353.Doc
dzf.irvnp.cn/406864.Doc
dzf.irvnp.cn/084802.Doc
dzf.irvnp.cn/640060.Doc
dzf.irvnp.cn/824088.Doc
dzf.irvnp.cn/862262.Doc
dzf.irvnp.cn/228888.Doc
dzd.irvnp.cn/195117.Doc
dzd.irvnp.cn/804466.Doc
dzd.irvnp.cn/280248.Doc
dzd.irvnp.cn/820884.Doc
dzd.irvnp.cn/866440.Doc
dzd.irvnp.cn/044486.Doc
dzd.irvnp.cn/266442.Doc
dzd.irvnp.cn/028840.Doc
dzd.irvnp.cn/840640.Doc
dzd.irvnp.cn/882228.Doc
dzs.irvnp.cn/220602.Doc
dzs.irvnp.cn/420004.Doc
dzs.irvnp.cn/242066.Doc
dzs.irvnp.cn/260428.Doc
dzs.irvnp.cn/244280.Doc
dzs.irvnp.cn/860004.Doc
dzs.irvnp.cn/759793.Doc
dzs.irvnp.cn/024480.Doc
dzs.irvnp.cn/557719.Doc
dzs.irvnp.cn/428408.Doc
dza.irvnp.cn/939733.Doc
dza.irvnp.cn/624664.Doc
dza.irvnp.cn/468428.Doc
dza.irvnp.cn/686468.Doc
dza.irvnp.cn/240468.Doc
dza.irvnp.cn/862864.Doc
dza.irvnp.cn/399195.Doc
dza.irvnp.cn/664202.Doc
dza.irvnp.cn/448620.Doc
dza.irvnp.cn/866280.Doc
dzp.irvnp.cn/808668.Doc
dzp.irvnp.cn/424846.Doc
dzp.irvnp.cn/937595.Doc
dzp.irvnp.cn/660666.Doc
dzp.irvnp.cn/260202.Doc
dzp.irvnp.cn/048822.Doc
dzp.irvnp.cn/066024.Doc
dzp.irvnp.cn/248860.Doc
dzp.irvnp.cn/484608.Doc
dzp.irvnp.cn/604882.Doc
dzo.irvnp.cn/644604.Doc
dzo.irvnp.cn/393779.Doc
dzo.irvnp.cn/846604.Doc
dzo.irvnp.cn/602682.Doc
dzo.irvnp.cn/795993.Doc
dzo.irvnp.cn/262426.Doc
dzo.irvnp.cn/088206.Doc
dzo.irvnp.cn/660668.Doc
dzo.irvnp.cn/735159.Doc
dzo.irvnp.cn/608224.Doc
dzi.irvnp.cn/088208.Doc
dzi.irvnp.cn/248006.Doc
dzi.irvnp.cn/602404.Doc
dzi.irvnp.cn/640848.Doc
dzi.irvnp.cn/264820.Doc
dzi.irvnp.cn/117753.Doc
dzi.irvnp.cn/517537.Doc
dzi.irvnp.cn/048426.Doc
dzi.irvnp.cn/824486.Doc
dzi.irvnp.cn/335137.Doc
dzu.irvnp.cn/028488.Doc
dzu.irvnp.cn/286420.Doc
dzu.irvnp.cn/337315.Doc
dzu.irvnp.cn/046624.Doc
dzu.irvnp.cn/042444.Doc
dzu.irvnp.cn/800000.Doc
dzu.irvnp.cn/262088.Doc
dzu.irvnp.cn/224024.Doc
dzu.irvnp.cn/662200.Doc
dzu.irvnp.cn/606808.Doc
dzy.irvnp.cn/537379.Doc
dzy.irvnp.cn/068884.Doc
dzy.irvnp.cn/559999.Doc
dzy.irvnp.cn/622866.Doc
dzy.irvnp.cn/860664.Doc
dzy.irvnp.cn/242822.Doc
dzy.irvnp.cn/199919.Doc
dzy.irvnp.cn/448660.Doc
dzy.irvnp.cn/791795.Doc
dzy.irvnp.cn/680200.Doc
dzt.irvnp.cn/002002.Doc
dzt.irvnp.cn/157917.Doc
dzt.irvnp.cn/662644.Doc
dzt.irvnp.cn/448402.Doc
dzt.irvnp.cn/133175.Doc
