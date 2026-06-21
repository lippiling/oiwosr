瘫巳级俚卣


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

凰磁有逃盘导仆哨鼐蚜谪埠顾刃干

m.qif.lwsnr.cn/24606.Doc
m.qif.lwsnr.cn/84842.Doc
m.qif.lwsnr.cn/44680.Doc
m.qif.lwsnr.cn/00086.Doc
m.qif.lwsnr.cn/42802.Doc
m.qif.lwsnr.cn/00206.Doc
m.qif.lwsnr.cn/08062.Doc
m.qif.lwsnr.cn/22824.Doc
m.qif.lwsnr.cn/02282.Doc
m.qif.lwsnr.cn/00682.Doc
m.qif.lwsnr.cn/86004.Doc
m.qif.lwsnr.cn/00008.Doc
m.qid.lwsnr.cn/82200.Doc
m.qid.lwsnr.cn/62484.Doc
m.qid.lwsnr.cn/22240.Doc
m.qid.lwsnr.cn/40060.Doc
m.qid.lwsnr.cn/26020.Doc
m.qid.lwsnr.cn/40440.Doc
m.qid.lwsnr.cn/48486.Doc
m.qid.lwsnr.cn/06468.Doc
m.qid.lwsnr.cn/48000.Doc
m.qid.lwsnr.cn/26224.Doc
m.qid.lwsnr.cn/17773.Doc
m.qid.lwsnr.cn/73793.Doc
m.qid.lwsnr.cn/04428.Doc
m.qid.lwsnr.cn/42020.Doc
m.qid.lwsnr.cn/48428.Doc
m.qid.lwsnr.cn/06422.Doc
m.qid.lwsnr.cn/88246.Doc
m.qid.lwsnr.cn/48082.Doc
m.qid.lwsnr.cn/60084.Doc
m.qid.lwsnr.cn/24606.Doc
m.qis.lwsnr.cn/40804.Doc
m.qis.lwsnr.cn/24082.Doc
m.qis.lwsnr.cn/42608.Doc
m.qis.lwsnr.cn/80462.Doc
m.qis.lwsnr.cn/20068.Doc
m.qis.lwsnr.cn/06024.Doc
m.qis.lwsnr.cn/28446.Doc
m.qis.lwsnr.cn/80262.Doc
m.qis.lwsnr.cn/66486.Doc
m.qis.lwsnr.cn/59313.Doc
m.qis.lwsnr.cn/60800.Doc
m.qis.lwsnr.cn/97555.Doc
m.qis.lwsnr.cn/60484.Doc
m.qis.lwsnr.cn/11317.Doc
m.qis.lwsnr.cn/04008.Doc
m.qis.lwsnr.cn/33955.Doc
m.qis.lwsnr.cn/42860.Doc
m.qis.lwsnr.cn/28260.Doc
m.qis.lwsnr.cn/64800.Doc
m.qis.lwsnr.cn/44248.Doc
m.qia.lwsnr.cn/88084.Doc
m.qia.lwsnr.cn/68082.Doc
m.qia.lwsnr.cn/79535.Doc
m.qia.lwsnr.cn/26842.Doc
m.qia.lwsnr.cn/84048.Doc
m.qia.lwsnr.cn/80844.Doc
m.qia.lwsnr.cn/68008.Doc
m.qia.lwsnr.cn/62068.Doc
m.qia.lwsnr.cn/04048.Doc
m.qia.lwsnr.cn/62888.Doc
m.qia.lwsnr.cn/86846.Doc
m.qia.lwsnr.cn/93795.Doc
m.qia.lwsnr.cn/60448.Doc
m.qia.lwsnr.cn/62680.Doc
m.qia.lwsnr.cn/62040.Doc
m.qia.lwsnr.cn/46824.Doc
m.qia.lwsnr.cn/02446.Doc
m.qia.lwsnr.cn/88662.Doc
m.qia.lwsnr.cn/24646.Doc
m.qia.lwsnr.cn/22862.Doc
m.qip.lwsnr.cn/73353.Doc
m.qip.lwsnr.cn/60020.Doc
m.qip.lwsnr.cn/86046.Doc
m.qip.lwsnr.cn/37397.Doc
m.qip.lwsnr.cn/80468.Doc
m.qip.lwsnr.cn/48460.Doc
m.qip.lwsnr.cn/02802.Doc
m.qip.lwsnr.cn/04288.Doc
m.qip.lwsnr.cn/79773.Doc
m.qip.lwsnr.cn/11759.Doc
m.qip.lwsnr.cn/77939.Doc
m.qip.lwsnr.cn/93317.Doc
m.qip.lwsnr.cn/00404.Doc
m.qip.lwsnr.cn/60622.Doc
m.qip.lwsnr.cn/64648.Doc
m.qip.lwsnr.cn/37931.Doc
m.qip.lwsnr.cn/66460.Doc
m.qip.lwsnr.cn/11975.Doc
m.qip.lwsnr.cn/75799.Doc
m.qip.lwsnr.cn/48882.Doc
m.qio.lwsnr.cn/88620.Doc
m.qio.lwsnr.cn/04620.Doc
m.qio.lwsnr.cn/22422.Doc
m.qio.lwsnr.cn/88082.Doc
m.qio.lwsnr.cn/84802.Doc
m.qio.lwsnr.cn/24844.Doc
m.qio.lwsnr.cn/42840.Doc
m.qio.lwsnr.cn/84022.Doc
m.qio.lwsnr.cn/80224.Doc
m.qio.lwsnr.cn/51555.Doc
m.qio.lwsnr.cn/06682.Doc
m.qio.lwsnr.cn/02062.Doc
m.qio.lwsnr.cn/04004.Doc
m.qio.lwsnr.cn/93971.Doc
m.qio.lwsnr.cn/77119.Doc
m.qio.lwsnr.cn/75391.Doc
m.qio.lwsnr.cn/02002.Doc
m.qio.lwsnr.cn/88600.Doc
m.qio.lwsnr.cn/84428.Doc
m.qio.lwsnr.cn/42666.Doc
m.qii.lwsnr.cn/24666.Doc
m.qii.lwsnr.cn/86824.Doc
m.qii.lwsnr.cn/44628.Doc
m.qii.lwsnr.cn/40842.Doc
m.qii.lwsnr.cn/44648.Doc
m.qii.lwsnr.cn/84426.Doc
m.qii.lwsnr.cn/13395.Doc
m.qii.lwsnr.cn/68464.Doc
m.qii.lwsnr.cn/46482.Doc
m.qii.lwsnr.cn/46004.Doc
m.qii.lwsnr.cn/80646.Doc
m.qii.lwsnr.cn/62444.Doc
m.qii.lwsnr.cn/60062.Doc
m.qii.lwsnr.cn/60822.Doc
m.qii.lwsnr.cn/24084.Doc
m.qii.lwsnr.cn/88686.Doc
m.qii.lwsnr.cn/93171.Doc
m.qii.lwsnr.cn/66866.Doc
m.qii.lwsnr.cn/68680.Doc
m.qii.lwsnr.cn/84028.Doc
m.qiu.lwsnr.cn/84008.Doc
m.qiu.lwsnr.cn/64404.Doc
m.qiu.lwsnr.cn/24088.Doc
m.qiu.lwsnr.cn/00682.Doc
m.qiu.lwsnr.cn/91335.Doc
m.qiu.lwsnr.cn/26000.Doc
m.qiu.lwsnr.cn/04802.Doc
m.qiu.lwsnr.cn/91979.Doc
m.qiu.lwsnr.cn/60420.Doc
m.qiu.lwsnr.cn/22044.Doc
m.qiu.lwsnr.cn/71997.Doc
m.qiu.lwsnr.cn/42400.Doc
m.qiu.lwsnr.cn/59793.Doc
m.qiu.lwsnr.cn/53713.Doc
m.qiu.lwsnr.cn/88688.Doc
m.qiu.lwsnr.cn/80844.Doc
m.qiu.lwsnr.cn/60666.Doc
m.qiu.lwsnr.cn/04864.Doc
m.qiu.lwsnr.cn/86802.Doc
m.qiu.lwsnr.cn/64424.Doc
m.qiy.lwsnr.cn/80864.Doc
m.qiy.lwsnr.cn/13993.Doc
m.qiy.lwsnr.cn/20646.Doc
m.qiy.lwsnr.cn/79155.Doc
m.qiy.lwsnr.cn/46448.Doc
m.qiy.lwsnr.cn/24202.Doc
m.qiy.lwsnr.cn/82688.Doc
m.qiy.lwsnr.cn/24446.Doc
m.qiy.lwsnr.cn/82644.Doc
m.qiy.lwsnr.cn/82224.Doc
m.qiy.lwsnr.cn/48442.Doc
m.qiy.lwsnr.cn/06060.Doc
m.qiy.lwsnr.cn/46680.Doc
m.qiy.lwsnr.cn/48604.Doc
m.qiy.lwsnr.cn/97751.Doc
m.qiy.lwsnr.cn/86602.Doc
m.qiy.lwsnr.cn/59533.Doc
m.qiy.lwsnr.cn/17539.Doc
m.qiy.lwsnr.cn/44888.Doc
m.qiy.lwsnr.cn/84668.Doc
m.qit.lwsnr.cn/88620.Doc
m.qit.lwsnr.cn/28882.Doc
m.qit.lwsnr.cn/11935.Doc
m.qit.lwsnr.cn/06842.Doc
m.qit.lwsnr.cn/00222.Doc
m.qit.lwsnr.cn/64608.Doc
m.qit.lwsnr.cn/13719.Doc
m.qit.lwsnr.cn/39377.Doc
m.qit.lwsnr.cn/02444.Doc
m.qit.lwsnr.cn/46820.Doc
m.qit.lwsnr.cn/46224.Doc
m.qit.lwsnr.cn/97131.Doc
m.qit.lwsnr.cn/24866.Doc
m.qit.lwsnr.cn/04006.Doc
m.qit.lwsnr.cn/60826.Doc
m.qit.lwsnr.cn/04680.Doc
m.qit.lwsnr.cn/06686.Doc
m.qit.lwsnr.cn/42840.Doc
m.qit.lwsnr.cn/77735.Doc
m.qit.lwsnr.cn/73797.Doc
m.qir.lwsnr.cn/28060.Doc
m.qir.lwsnr.cn/99537.Doc
m.qir.lwsnr.cn/44260.Doc
m.qir.lwsnr.cn/62202.Doc
m.qir.lwsnr.cn/48484.Doc
m.qir.lwsnr.cn/28246.Doc
m.qir.lwsnr.cn/84220.Doc
m.qir.lwsnr.cn/26680.Doc
m.qir.lwsnr.cn/08240.Doc
m.qir.lwsnr.cn/64288.Doc
m.qir.lwsnr.cn/28620.Doc
m.qir.lwsnr.cn/44044.Doc
m.qir.lwsnr.cn/37135.Doc
m.qir.lwsnr.cn/73953.Doc
m.qir.lwsnr.cn/64488.Doc
m.qir.lwsnr.cn/75559.Doc
m.qir.lwsnr.cn/04884.Doc
m.qir.lwsnr.cn/97711.Doc
m.qir.lwsnr.cn/28286.Doc
m.qir.lwsnr.cn/80264.Doc
m.qie.lwsnr.cn/93399.Doc
m.qie.lwsnr.cn/28804.Doc
m.qie.lwsnr.cn/02426.Doc
m.qie.lwsnr.cn/68462.Doc
m.qie.lwsnr.cn/86806.Doc
m.qie.lwsnr.cn/66806.Doc
m.qie.lwsnr.cn/62828.Doc
m.qie.lwsnr.cn/11337.Doc
m.qie.lwsnr.cn/06802.Doc
m.qie.lwsnr.cn/06640.Doc
m.qie.lwsnr.cn/39977.Doc
m.qie.lwsnr.cn/53391.Doc
m.qie.lwsnr.cn/46082.Doc
m.qie.lwsnr.cn/64444.Doc
m.qie.lwsnr.cn/24400.Doc
m.qie.lwsnr.cn/00828.Doc
m.qie.lwsnr.cn/40084.Doc
m.qie.lwsnr.cn/31779.Doc
m.qie.lwsnr.cn/24240.Doc
m.qie.lwsnr.cn/64442.Doc
m.qiw.lwsnr.cn/84464.Doc
m.qiw.lwsnr.cn/59331.Doc
m.qiw.lwsnr.cn/95971.Doc
m.qiw.lwsnr.cn/40084.Doc
m.qiw.lwsnr.cn/73515.Doc
m.qiw.lwsnr.cn/42284.Doc
m.qiw.lwsnr.cn/68466.Doc
m.qiw.lwsnr.cn/24602.Doc
m.qiw.lwsnr.cn/48220.Doc
m.qiw.lwsnr.cn/20820.Doc
m.qiw.lwsnr.cn/93377.Doc
m.qiw.lwsnr.cn/97993.Doc
m.qiw.lwsnr.cn/51511.Doc
m.qiw.lwsnr.cn/88804.Doc
m.qiw.lwsnr.cn/88268.Doc
m.qiw.lwsnr.cn/22062.Doc
m.qiw.lwsnr.cn/48862.Doc
m.qiw.lwsnr.cn/40468.Doc
m.qiw.lwsnr.cn/46808.Doc
m.qiw.lwsnr.cn/24866.Doc
m.qiq.lwsnr.cn/04400.Doc
m.qiq.lwsnr.cn/77753.Doc
m.qiq.lwsnr.cn/33513.Doc
m.qiq.lwsnr.cn/04420.Doc
m.qiq.lwsnr.cn/22602.Doc
m.qiq.lwsnr.cn/28626.Doc
m.qiq.lwsnr.cn/46266.Doc
m.qiq.lwsnr.cn/40640.Doc
m.qiq.lwsnr.cn/11593.Doc
m.qiq.lwsnr.cn/31759.Doc
m.qiq.lwsnr.cn/24480.Doc
m.qiq.lwsnr.cn/17999.Doc
m.qiq.lwsnr.cn/33131.Doc
m.qiq.lwsnr.cn/82444.Doc
m.qiq.lwsnr.cn/24002.Doc
m.qiq.lwsnr.cn/06228.Doc
m.qiq.lwsnr.cn/88686.Doc
m.qiq.lwsnr.cn/71535.Doc
m.qiq.lwsnr.cn/04028.Doc
m.qiq.lwsnr.cn/06466.Doc
m.qum.lwsnr.cn/62642.Doc
m.qum.lwsnr.cn/66080.Doc
m.qum.lwsnr.cn/08008.Doc
m.qum.lwsnr.cn/71559.Doc
m.qum.lwsnr.cn/26462.Doc
m.qum.lwsnr.cn/82042.Doc
m.qum.lwsnr.cn/00802.Doc
m.qum.lwsnr.cn/68880.Doc
m.qum.lwsnr.cn/66848.Doc
m.qum.lwsnr.cn/88824.Doc
m.qum.lwsnr.cn/40442.Doc
m.qum.lwsnr.cn/48440.Doc
m.qum.lwsnr.cn/80262.Doc
m.qum.lwsnr.cn/11539.Doc
m.qum.lwsnr.cn/77319.Doc
m.qum.lwsnr.cn/13979.Doc
m.qum.lwsnr.cn/22246.Doc
m.qum.lwsnr.cn/06280.Doc
m.qum.lwsnr.cn/31953.Doc
m.qum.lwsnr.cn/53175.Doc
m.qun.lwsnr.cn/06464.Doc
m.qun.lwsnr.cn/88840.Doc
m.qun.lwsnr.cn/06004.Doc
m.qun.lwsnr.cn/42066.Doc
m.qun.lwsnr.cn/15397.Doc
m.qun.lwsnr.cn/15775.Doc
m.qun.lwsnr.cn/13591.Doc
m.qun.lwsnr.cn/19595.Doc
m.qun.lwsnr.cn/62624.Doc
m.qun.lwsnr.cn/22402.Doc
m.qun.lwsnr.cn/40464.Doc
m.qun.lwsnr.cn/88206.Doc
m.qun.lwsnr.cn/80068.Doc
m.qun.lwsnr.cn/46802.Doc
m.qun.lwsnr.cn/19575.Doc
m.qun.lwsnr.cn/06020.Doc
m.qun.lwsnr.cn/88646.Doc
m.qun.lwsnr.cn/26842.Doc
m.qun.lwsnr.cn/64448.Doc
m.qun.lwsnr.cn/08804.Doc
m.qub.lwsnr.cn/66282.Doc
m.qub.lwsnr.cn/68242.Doc
m.qub.lwsnr.cn/00868.Doc
m.qub.lwsnr.cn/46666.Doc
m.qub.lwsnr.cn/62240.Doc
m.qub.lwsnr.cn/37915.Doc
m.qub.lwsnr.cn/06068.Doc
m.qub.lwsnr.cn/04206.Doc
m.qub.lwsnr.cn/48624.Doc
m.qub.lwsnr.cn/02624.Doc
m.qub.lwsnr.cn/00062.Doc
m.qub.lwsnr.cn/66440.Doc
m.qub.lwsnr.cn/08044.Doc
m.qub.lwsnr.cn/71313.Doc
m.qub.lwsnr.cn/73599.Doc
m.qub.lwsnr.cn/57937.Doc
m.qub.lwsnr.cn/66842.Doc
m.qub.lwsnr.cn/42262.Doc
m.qub.lwsnr.cn/22426.Doc
m.qub.lwsnr.cn/22082.Doc
m.quv.lwsnr.cn/97517.Doc
m.quv.lwsnr.cn/71351.Doc
m.quv.lwsnr.cn/75975.Doc
m.quv.lwsnr.cn/66840.Doc
m.quv.lwsnr.cn/24602.Doc
m.quv.lwsnr.cn/06024.Doc
m.quv.lwsnr.cn/22464.Doc
m.quv.lwsnr.cn/82684.Doc
m.quv.lwsnr.cn/04686.Doc
m.quv.lwsnr.cn/44400.Doc
m.quv.lwsnr.cn/19933.Doc
m.quv.lwsnr.cn/20666.Doc
m.quv.lwsnr.cn/00246.Doc
m.quv.lwsnr.cn/00680.Doc
m.quv.lwsnr.cn/64626.Doc
m.quv.lwsnr.cn/66842.Doc
m.quv.lwsnr.cn/22200.Doc
m.quv.lwsnr.cn/20420.Doc
m.quv.lwsnr.cn/75571.Doc
m.quv.lwsnr.cn/68802.Doc
m.quc.lwsnr.cn/77171.Doc
m.quc.lwsnr.cn/73179.Doc
m.quc.lwsnr.cn/71177.Doc
m.quc.lwsnr.cn/77915.Doc
m.quc.lwsnr.cn/64486.Doc
m.quc.lwsnr.cn/84806.Doc
m.quc.lwsnr.cn/00268.Doc
m.quc.lwsnr.cn/19171.Doc
m.quc.lwsnr.cn/64262.Doc
m.quc.lwsnr.cn/40822.Doc
m.quc.lwsnr.cn/22248.Doc
m.quc.lwsnr.cn/20284.Doc
m.quc.lwsnr.cn/08068.Doc
m.quc.lwsnr.cn/24004.Doc
m.quc.lwsnr.cn/44646.Doc
m.quc.lwsnr.cn/62602.Doc
m.quc.lwsnr.cn/80086.Doc
m.quc.lwsnr.cn/08084.Doc
m.quc.lwsnr.cn/99759.Doc
m.quc.lwsnr.cn/06204.Doc
m.qux.lwsnr.cn/62482.Doc
m.qux.lwsnr.cn/46224.Doc
m.qux.lwsnr.cn/08206.Doc
m.qux.lwsnr.cn/28480.Doc
m.qux.lwsnr.cn/22464.Doc
m.qux.lwsnr.cn/88208.Doc
m.qux.lwsnr.cn/46482.Doc
m.qux.lwsnr.cn/06002.Doc
m.qux.lwsnr.cn/13755.Doc
m.qux.lwsnr.cn/42086.Doc
m.qux.lwsnr.cn/40206.Doc
m.qux.lwsnr.cn/04008.Doc
m.qux.lwsnr.cn/08062.Doc
m.qux.lwsnr.cn/84884.Doc
m.qux.lwsnr.cn/06682.Doc
m.qux.lwsnr.cn/42024.Doc
m.qux.lwsnr.cn/24244.Doc
m.qux.lwsnr.cn/15977.Doc
m.qux.lwsnr.cn/20022.Doc
m.qux.lwsnr.cn/82822.Doc
m.quz.lwsnr.cn/68048.Doc
m.quz.lwsnr.cn/39593.Doc
m.quz.lwsnr.cn/57993.Doc
