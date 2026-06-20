谕渍捌伦松


===============================================================================
  Python 类型推导与协议 (Protocol & TypeGuard)
===============================================================================
  深入 Python 的类型系统：Protocol 实现结构子类型（鸭子类型），
  TypeGuard 自定义类型收窄，@runtime_checkable 运行时检查，
  以及协变/逆变、类型桩文件等高级主题。
===============================================================================

from typing import (
    Protocol, TypeVar, runtime_checkable,
    TypeGuard, assert_type, reveal_type,
    Iterator, Iterable, overload
)
from typing import TYPE_CHECKING

T = TypeVar("T")

# ===================================================================
# 第一部分：Protocol 结构子类型
# ===================================================================

# -------------------------------------------------------------------
# 1. 基本 Protocol：定义行为接口而非类层次
# -------------------------------------------------------------------
class 可打印(Protocol):
    """任何具有 .to_str() 方法的对象都自动满足此协议"""
    def to_str(self) -> str: ...

class 用户:
    def __init__(self, name: str):
        self.name = name
    def to_str(self) -> str:
        return f"用户: {self.name}"

class 商品:
    def __init__(self, title: str, price: float):
        self.title = title
        self.price = price
    def to_str(self) -> str:
        return f"{self.title} (¥{self.price})"

def 格式化输出(对象: 可打印) -> None:
    """接受任何满足 可打印 协议的对象"""
    print(f"格式化: {对象.to_str()}")

格式化输出(用户("张三"))          # → 格式化: 用户: 张三
格式化输出(商品("Python教程", 39.9))  # → 格式化: Python教程 (¥39.9)

# -------------------------------------------------------------------
# 2. 多方法协议：定义完整接口
# -------------------------------------------------------------------
class 可迭代集合(Iterable[T], Protocol[T]):
    """协议可以继承其他协议或类型"""
    def __len__(self) -> int: ...
    def __iter__(self) -> Iterator[T]: ...

def 获取大小(集合: 可迭代集合) -> int:
    return len(集合)

print(f"列表大小: {获取大小([1, 2, 3])}")       # → 3
print(f"集合大小: {获取大小({1, 2, 3, 4})}")    # → 4

# -------------------------------------------------------------------
# 3. @runtime_checkable：运行时协议检查
# -------------------------------------------------------------------
@runtime_checkable
class 可序列化(Protocol):
    def 序列化(self) -> bytes: ...
    def 反序列化(self, 数据: bytes) -> None: ...

class JSON文档:
    def 序列化(self) -> bytes:
        return b'{"key": "value"}'
    def 反序列化(self, 数据: bytes) -> None:
        print(f"反序列化: {数据}")

class XML文档:
    def 序列化(self) -> bytes:
        return b"<root><key>value</key></root>"
    def 反序列化(self, 数据: bytes) -> None:
        print(f"解析 XML: {数据}")

def 保存文档(文档: 可序列化) -> None:
    if isinstance(文档, 可序列化):
        数据 = 文档.序列化()
        print(f"保存 {len(数据)} 字节")
    else:
        raise TypeError("对象不可序列化")

保存文档(JSON文档())    # → 保存 16 字节

# ===================================================================
# 第二部分：TypeGuard 类型守卫
# ===================================================================

# -------------------------------------------------------------------
# 4. TypeGuard：自定义类型收窄
# -------------------------------------------------------------------
def 是字符串列表(值: list[object]) -> TypeGuard[list[str]]:
    """TypeGuard 让类型检查器知道：如果返回 True，参数就是 list[str]"""
    return all(isinstance(x, str) for x in 值)

def 是整数列表(值: list[object]) -> TypeGuard[list[int]]:
    return all(isinstance(x, int) for x in 值)

def 处理混合列表(列表: list[object]) -> None:
    if 是字符串列表(列表):
        print(f"字符串列表: {', '.join(列表)}")
    elif 是整数列表(列表):
        总和 = sum(列表)
        print(f"整数列表, 总和: {总和}")
    else:
        print(f"混合类型列表: {列表}")

处理混合列表(["a", "b", "c"])       # → 字符串列表: a, b, c
处理混合列表([1, 2, 3, 4, 5])       # → 整数列表, 总和: 15
处理混合列表([1, "two", 3])         # → 混合类型列表

# -------------------------------------------------------------------
# 5. TypeGuard 在类方法中的使用
# -------------------------------------------------------------------
class 数据处理器:
    @staticmethod
    def 是有效记录(数据: dict[str, object]) -> TypeGuard[dict[str, str | int]]:
        return (
            "id" in 数据 and isinstance(数据["id"], int)
            and "name" in 数据 and isinstance(数据["name"], str)
        )
    @classmethod
    def 处理(cls, 数据: dict[str, object]) -> None:
        if cls.是有效记录(数据):
            print(f"处理记录: id={数据['id']}, name={数据['name']}")
        else:
            print("无效记录")

数据处理器.处理({"id": 1, "name": "张三"})

# ===================================================================
# 第三部分：类型推断辅助工具
# ===================================================================

# -------------------------------------------------------------------
# 6. assert_type 和 reveal_type
# -------------------------------------------------------------------
def 类型检查辅助():
    值1: int = 42
    # reveal_type(值1)    # mypy 显示: Revealed type is 'builtins.str'
    print(f"类型检查辅助工具: assert_type / reveal_type")

类型检查辅助()

# -------------------------------------------------------------------
# 7. 类型桩文件 (.pyi)
# -------------------------------------------------------------------
def 桩文件示例():
    """
    类型桩文件 (.pyi) 为纯 Python 项目提供类型信息：
    - 放在与 .py 文件同目录下，文件名相同，扩展名为 .pyi
    - 只包含类型签名，不含实现
    """
    print("类型桩文件 (.pyi) 为已有模块添加类型信息")
    print("优点：不改动源码即可添加类型")

桩文件示例()

# ===================================================================
# 第四部分：协变与逆变在 Protocol 中的应用
# ===================================================================

# -------------------------------------------------------------------
# 8. Protocol 中的协变与逆变
# -------------------------------------------------------------------
T_co = TypeVar("T_co", covariant=True)
T_con = TypeVar("T_con", contravariant=True)

class 生产者(Protocol[T_co]):
    """协变协议：只产生（返回）值"""
    def 获取(self) -> T_co: ...

class 消费者(Protocol[T_con]):
    """逆变协议：只消费（接收）值"""
    def 放入(self, 值: T_con) -> None: ...

class Int生产者:
    def 获取(self) -> int:
        return 42

def 使用生产者(p: 生产者[float]) -> None:
    print(f"生产值: {p.获取()}")

使用生产者(Int生产者())     # 协变允许

class Float消费者:
    def 放入(self, 值: float) -> None:
        print(f"消费浮点: {值}")

def 需要int消费者(c: 消费者[int]) -> None:
    c.放入(100)

需要int消费者(Float消费者())  # 逆变允许

# ===================================================================
# 第五部分：综合应用示例
# ===================================================================

# -------------------------------------------------------------------
# 9. 完整的 Protocol 实战：仓库模式
# -------------------------------------------------------------------
T实体 = TypeVar("T实体")

class 仓库(Protocol[T实体]):
    def 按ID获取(self, id: int) -> T实体 | None: ...
    def 保存(self, 实体: T实体) -> None: ...
    def 删除(self, id: int) -> bool: ...

@runtime_checkable
class 可验证(Protocol):
    def 验证(self) -> bool: ...

class 用户仓库:
    def __init__(self):
        self._数据: dict[int, str] = {}
    def 按ID获取(self, id: int) -> str | None:
        return self._数据.get(id)
    def 保存(self, 实体: str) -> None:
        id = hash(实体) % 10000
        self._数据[id] = 实体
    def 删除(self, id: int) -> bool:
        return self._数据.pop(id, None) is not None
    def 验证(self) -> bool:
        return len(self._数据) < 1000

def 执行仓库操作(仓库对象: 仓库[str]) -> None:
    仓库对象.保存("测试数据")
    if isinstance(仓库对象, 可验证):
        print(f"验证通过: {仓库对象.验证()}")

u = 用户仓库()
执行仓库操作(u)

# ===================================================================
# 总结
# ===================================================================
# Python 类型推导与协议核心概念：
#
# Protocol:
#   - 结构子类型：根据对象的方法签名判断类型（鸭子类型）
#   - @runtime_checkable: 支持 isinstance() 运行时检查
#   - 可继承其他协议/类型组合接口
#
# TypeGuard:
#   - 自定义类型收窄函数
#   - 类型检查器根据返回值收窄类型
#
# 辅助工具:
#   - assert_type: 运行时断言类型（开发调试）
#   - reveal_type: 让类型检查器显示推断类型
#   - .pyi 桩文件: 为已有模块添加类型信息
#
# 协变/逆变:
#   - 协变 (covariant): 生产者/只读容器
#   - 逆变 (contravariant): 消费者/只写容器
#   - 不变 (invariant): 读写兼备的容器
员匝佳苏号氐掷瞎潮挡资分罕恍寄

wwj.wfcj3t6.cn/480265.htm
wwj.wfcj3t6.cn/660065.htm
wwj.wfcj3t6.cn/153375.htm
wwj.wfcj3t6.cn/333935.htm
wwj.wfcj3t6.cn/280865.htm
wwj.wfcj3t6.cn/991315.htm
wwj.wfcj3t6.cn/337195.htm
wwj.wfcj3t6.cn/202845.htm
wwh.wfcj3t6.cn/662845.htm
wwh.wfcj3t6.cn/864665.htm
wwh.wfcj3t6.cn/220825.htm
wwh.wfcj3t6.cn/068645.htm
wwh.wfcj3t6.cn/686605.htm
wwh.wfcj3t6.cn/064245.htm
wwh.wfcj3t6.cn/006045.htm
wwh.wfcj3t6.cn/179935.htm
wwh.wfcj3t6.cn/042625.htm
wwh.wfcj3t6.cn/206465.htm
wwg.wfcj3t6.cn/951515.htm
wwg.wfcj3t6.cn/606885.htm
wwg.wfcj3t6.cn/595375.htm
wwg.wfcj3t6.cn/911715.htm
wwg.wfcj3t6.cn/886625.htm
wwg.wfcj3t6.cn/642885.htm
wwg.wfcj3t6.cn/139555.htm
wwg.wfcj3t6.cn/939355.htm
wwg.wfcj3t6.cn/686085.htm
wwg.wfcj3t6.cn/402065.htm
wwf.wfcj3t6.cn/400845.htm
wwf.wfcj3t6.cn/422025.htm
wwf.wfcj3t6.cn/684805.htm
wwf.wfcj3t6.cn/206225.htm
wwf.wfcj3t6.cn/602885.htm
wwf.wfcj3t6.cn/062485.htm
wwf.wfcj3t6.cn/244645.htm
wwf.wfcj3t6.cn/864025.htm
wwf.wfcj3t6.cn/357915.htm
wwf.wfcj3t6.cn/862405.htm
wwd.wfcj3t6.cn/577755.htm
wwd.wfcj3t6.cn/404865.htm
wwd.wfcj3t6.cn/046445.htm
wwd.wfcj3t6.cn/991995.htm
wwd.wfcj3t6.cn/068025.htm
wwd.wfcj3t6.cn/999935.htm
wwd.wfcj3t6.cn/999115.htm
wwd.wfcj3t6.cn/064885.htm
wwd.wfcj3t6.cn/000245.htm
wwd.wfcj3t6.cn/351715.htm
wws.wfcj3t6.cn/284225.htm
wws.wfcj3t6.cn/133735.htm
wws.wfcj3t6.cn/753975.htm
wws.wfcj3t6.cn/280205.htm
wws.wfcj3t6.cn/242045.htm
wws.wfcj3t6.cn/135995.htm
wws.wfcj3t6.cn/915135.htm
wws.wfcj3t6.cn/606605.htm
wws.wfcj3t6.cn/086485.htm
wws.wfcj3t6.cn/315915.htm
wwa.wfcj3t6.cn/888825.htm
wwa.wfcj3t6.cn/191335.htm
wwa.wfcj3t6.cn/046445.htm
wwa.wfcj3t6.cn/604865.htm
wwa.wfcj3t6.cn/240225.htm
wwa.wfcj3t6.cn/022025.htm
wwa.wfcj3t6.cn/335355.htm
wwa.wfcj3t6.cn/446645.htm
wwa.wfcj3t6.cn/404425.htm
wwa.wfcj3t6.cn/622605.htm
wwp.wfcj3t6.cn/191995.htm
wwp.wfcj3t6.cn/468405.htm
wwp.wfcj3t6.cn/022225.htm
wwp.wfcj3t6.cn/664085.htm
wwp.wfcj3t6.cn/226805.htm
wwp.wfcj3t6.cn/660285.htm
wwp.wfcj3t6.cn/840025.htm
wwp.wfcj3t6.cn/824425.htm
wwp.wfcj3t6.cn/406265.htm
wwp.wfcj3t6.cn/248245.htm
wwo.wfcj3t6.cn/951315.htm
wwo.wfcj3t6.cn/820425.htm
wwo.wfcj3t6.cn/660885.htm
wwo.wfcj3t6.cn/660485.htm
wwo.wfcj3t6.cn/448865.htm
wwo.wfcj3t6.cn/000865.htm
wwo.wfcj3t6.cn/353775.htm
wwo.wfcj3t6.cn/573175.htm
wwo.wfcj3t6.cn/333135.htm
wwo.wfcj3t6.cn/420605.htm
wwi.wfcj3t6.cn/591715.htm
wwi.wfcj3t6.cn/955195.htm
wwi.wfcj3t6.cn/268405.htm
wwi.wfcj3t6.cn/177395.htm
wwi.wfcj3t6.cn/686265.htm
wwi.wfcj3t6.cn/040025.htm
wwi.wfcj3t6.cn/222685.htm
wwi.wfcj3t6.cn/448085.htm
wwi.wfcj3t6.cn/535555.htm
wwi.wfcj3t6.cn/868245.htm
wwu.wfcj3t6.cn/646445.htm
wwu.wfcj3t6.cn/680245.htm
wwu.wfcj3t6.cn/642665.htm
wwu.wfcj3t6.cn/917395.htm
wwu.wfcj3t6.cn/600405.htm
wwu.wfcj3t6.cn/422445.htm
wwu.wfcj3t6.cn/646605.htm
wwu.wfcj3t6.cn/664045.htm
wwu.wfcj3t6.cn/040085.htm
wwu.wfcj3t6.cn/088645.htm
wwy.wfcj3t6.cn/373315.htm
wwy.wfcj3t6.cn/004045.htm
wwy.wfcj3t6.cn/953555.htm
wwy.wfcj3t6.cn/608045.htm
wwy.wfcj3t6.cn/606025.htm
wwy.wfcj3t6.cn/400825.htm
wwy.wfcj3t6.cn/351315.htm
wwy.wfcj3t6.cn/282065.htm
wwy.wfcj3t6.cn/573795.htm
wwy.wfcj3t6.cn/971535.htm
wwt.wfcj3t6.cn/884085.htm
wwt.wfcj3t6.cn/911355.htm
wwt.wfcj3t6.cn/044645.htm
wwt.wfcj3t6.cn/339335.htm
wwt.wfcj3t6.cn/820205.htm
wwt.wfcj3t6.cn/757975.htm
wwt.wfcj3t6.cn/931135.htm
wwt.wfcj3t6.cn/260845.htm
wwt.wfcj3t6.cn/668405.htm
wwt.wfcj3t6.cn/242285.htm
wwr.wfcj3t6.cn/688445.htm
wwr.wfcj3t6.cn/468405.htm
wwr.wfcj3t6.cn/822805.htm
wwr.wfcj3t6.cn/751375.htm
wwr.wfcj3t6.cn/404205.htm
wwr.wfcj3t6.cn/622265.htm
wwr.wfcj3t6.cn/642445.htm
wwr.wfcj3t6.cn/040845.htm
wwr.wfcj3t6.cn/640045.htm
wwr.wfcj3t6.cn/886485.htm
wwe.wfcj3t6.cn/260845.htm
wwe.wfcj3t6.cn/042205.htm
wwe.wfcj3t6.cn/846825.htm
wwe.wfcj3t6.cn/462445.htm
wwe.wfcj3t6.cn/240065.htm
wwe.wfcj3t6.cn/139935.htm
wwe.wfcj3t6.cn/537775.htm
wwe.wfcj3t6.cn/604405.htm
wwe.wfcj3t6.cn/860885.htm
wwe.wfcj3t6.cn/248805.htm
www.wfcj3t6.cn/264425.htm
www.wfcj3t6.cn/111595.htm
www.wfcj3t6.cn/646685.htm
www.wfcj3t6.cn/464625.htm
www.wfcj3t6.cn/040425.htm
www.wfcj3t6.cn/553115.htm
www.wfcj3t6.cn/664885.htm
www.wfcj3t6.cn/848825.htm
www.wfcj3t6.cn/755355.htm
www.wfcj3t6.cn/260625.htm
wwq.wfcj3t6.cn/442085.htm
wwq.wfcj3t6.cn/442025.htm
wwq.wfcj3t6.cn/913715.htm
wwq.wfcj3t6.cn/842805.htm
wwq.wfcj3t6.cn/242625.htm
wwq.wfcj3t6.cn/628645.htm
wwq.wfcj3t6.cn/311995.htm
wwq.wfcj3t6.cn/222285.htm
wwq.wfcj3t6.cn/864485.htm
wwq.wfcj3t6.cn/860465.htm
wqtv.wfcj3t6.cn/266845.htm
wqtv.wfcj3t6.cn/644045.htm
wqtv.wfcj3t6.cn/884445.htm
wqtv.wfcj3t6.cn/824425.htm
wqtv.wfcj3t6.cn/420825.htm
wqtv.wfcj3t6.cn/335395.htm
wqtv.wfcj3t6.cn/228685.htm
wqtv.wfcj3t6.cn/084085.htm
wqtv.wfcj3t6.cn/006605.htm
wqtv.wfcj3t6.cn/113735.htm
wqn.wfcj3t6.cn/240405.htm
wqn.wfcj3t6.cn/826445.htm
wqn.wfcj3t6.cn/428845.htm
wqn.wfcj3t6.cn/622045.htm
wqn.wfcj3t6.cn/517735.htm
wqn.wfcj3t6.cn/828625.htm
wqn.wfcj3t6.cn/711955.htm
wqn.wfcj3t6.cn/688065.htm
wqn.wfcj3t6.cn/644685.htm
wqn.wfcj3t6.cn/339935.htm
wqb.wfcj3t6.cn/373155.htm
wqb.wfcj3t6.cn/517915.htm
wqb.wfcj3t6.cn/775775.htm
wqb.wfcj3t6.cn/157575.htm
wqb.wfcj3t6.cn/206845.htm
wqb.wfcj3t6.cn/044485.htm
wqb.wfcj3t6.cn/579975.htm
wqb.wfcj3t6.cn/848285.htm
wqb.wfcj3t6.cn/917335.htm
wqb.wfcj3t6.cn/600425.htm
wqv.wfcj3t6.cn/686805.htm
wqv.wfcj3t6.cn/244865.htm
wqv.wfcj3t6.cn/462285.htm
wqv.wfcj3t6.cn/593795.htm
wqv.wfcj3t6.cn/519915.htm
wqv.wfcj3t6.cn/000025.htm
wqv.wfcj3t6.cn/197995.htm
wqv.wfcj3t6.cn/626205.htm
wqv.wfcj3t6.cn/468225.htm
wqv.wfcj3t6.cn/822065.htm
wqc.wfcj3t6.cn/024225.htm
wqc.wfcj3t6.cn/864885.htm
wqc.wfcj3t6.cn/662485.htm
wqc.wfcj3t6.cn/535515.htm
wqc.wfcj3t6.cn/519555.htm
wqc.wfcj3t6.cn/046225.htm
wqc.wfcj3t6.cn/004445.htm
wqc.wfcj3t6.cn/733755.htm
wqc.wfcj3t6.cn/086045.htm
wqc.wfcj3t6.cn/113395.htm
wqx.wfcj3t6.cn/246285.htm
wqx.wfcj3t6.cn/082245.htm
wqx.wfcj3t6.cn/624685.htm
wqx.wfcj3t6.cn/046645.htm
wqx.wfcj3t6.cn/620825.htm
wqx.wfcj3t6.cn/804085.htm
wqx.wfcj3t6.cn/026885.htm
wqx.wfcj3t6.cn/268205.htm
wqx.wfcj3t6.cn/393995.htm
wqx.wfcj3t6.cn/573935.htm
wqz.wfcj3t6.cn/486445.htm
wqz.wfcj3t6.cn/595935.htm
wqz.wfcj3t6.cn/335575.htm
wqz.wfcj3t6.cn/466025.htm
wqz.wfcj3t6.cn/777195.htm
wqz.wfcj3t6.cn/024285.htm
wqz.wfcj3t6.cn/286825.htm
wqz.wfcj3t6.cn/600645.htm
wqz.wfcj3t6.cn/084265.htm
wqz.wfcj3t6.cn/208265.htm
wql.wfcj3t6.cn/428405.htm
wql.wfcj3t6.cn/335975.htm
wql.wfcj3t6.cn/860425.htm
wql.wfcj3t6.cn/664085.htm
wql.wfcj3t6.cn/022005.htm
wql.wfcj3t6.cn/460045.htm
wql.wfcj3t6.cn/731395.htm
wql.wfcj3t6.cn/004225.htm
wql.wfcj3t6.cn/844265.htm
wql.wfcj3t6.cn/319135.htm
wqk.wfcj3t6.cn/804085.htm
wqk.wfcj3t6.cn/995595.htm
wqk.wfcj3t6.cn/402425.htm
wqk.wfcj3t6.cn/191375.htm
wqk.wfcj3t6.cn/204245.htm
wqk.wfcj3t6.cn/486425.htm
wqk.wfcj3t6.cn/602005.htm
wqk.wfcj3t6.cn/286465.htm
wqk.wfcj3t6.cn/468025.htm
wqk.wfcj3t6.cn/408045.htm
wqj.wfcj3t6.cn/408445.htm
wqj.wfcj3t6.cn/440285.htm
wqj.wfcj3t6.cn/517935.htm
wqj.wfcj3t6.cn/486225.htm
wqj.wfcj3t6.cn/640805.htm
wqj.wfcj3t6.cn/000625.htm
wqj.wfcj3t6.cn/202225.htm
wqj.wfcj3t6.cn/606285.htm
wqj.wfcj3t6.cn/806865.htm
wqj.wfcj3t6.cn/608445.htm
wqh.wfcj3t6.cn/739335.htm
wqh.wfcj3t6.cn/137335.htm
wqh.wfcj3t6.cn/082825.htm
wqh.wfcj3t6.cn/468065.htm
wqh.wfcj3t6.cn/915535.htm
wqh.wfcj3t6.cn/404825.htm
wqh.wfcj3t6.cn/551315.htm
wqh.wfcj3t6.cn/331375.htm
wqh.wfcj3t6.cn/640025.htm
wqh.wfcj3t6.cn/268665.htm
wqg.wfcj3t6.cn/802265.htm
wqg.wfcj3t6.cn/682845.htm
wqg.wfcj3t6.cn/971195.htm
wqg.wfcj3t6.cn/208825.htm
wqg.wfcj3t6.cn/400205.htm
wqg.wfcj3t6.cn/620465.htm
wqg.wfcj3t6.cn/288205.htm
wqg.wfcj3t6.cn/957315.htm
wqg.wfcj3t6.cn/222065.htm
wqg.wfcj3t6.cn/444685.htm
wqf.wfcj3t6.cn/191115.htm
wqf.wfcj3t6.cn/779935.htm
wqf.wfcj3t6.cn/266045.htm
wqf.wfcj3t6.cn/882225.htm
wqf.wfcj3t6.cn/084885.htm
wqf.wfcj3t6.cn/642805.htm
wqf.wfcj3t6.cn/228685.htm
wqf.wfcj3t6.cn/600885.htm
wqf.wfcj3t6.cn/802005.htm
wqf.wfcj3t6.cn/119595.htm
wqd.wfcj3t6.cn/004205.htm
wqd.wfcj3t6.cn/404625.htm
wqd.wfcj3t6.cn/359335.htm
wqd.wfcj3t6.cn/284645.htm
wqd.wfcj3t6.cn/400665.htm
wqd.wfcj3t6.cn/195595.htm
wqd.wfcj3t6.cn/600085.htm
wqd.wfcj3t6.cn/802625.htm
wqd.wfcj3t6.cn/959935.htm
wqd.wfcj3t6.cn/024645.htm
wqs.wfcj3t6.cn/040405.htm
wqs.wfcj3t6.cn/602885.htm
wqs.wfcj3t6.cn/208825.htm
wqs.wfcj3t6.cn/371515.htm
wqs.wfcj3t6.cn/800265.htm
wqs.wfcj3t6.cn/448845.htm
wqs.wfcj3t6.cn/400265.htm
wqs.wfcj3t6.cn/404065.htm
wqs.wfcj3t6.cn/040445.htm
wqs.wfcj3t6.cn/395775.htm
wqa.wfcj3t6.cn/311955.htm
wqa.wfcj3t6.cn/646825.htm
wqa.wfcj3t6.cn/406225.htm
wqa.wfcj3t6.cn/824805.htm
wqa.wfcj3t6.cn/840425.htm
wqa.wfcj3t6.cn/733775.htm
wqa.wfcj3t6.cn/000085.htm
wqa.wfcj3t6.cn/771735.htm
wqa.wfcj3t6.cn/331795.htm
wqa.wfcj3t6.cn/440845.htm
wqp.wfcj3t6.cn/751395.htm
wqp.wfcj3t6.cn/082045.htm
wqp.wfcj3t6.cn/622265.htm
wqp.wfcj3t6.cn/359775.htm
wqp.wfcj3t6.cn/806825.htm
wqp.wfcj3t6.cn/393535.htm
wqp.wfcj3t6.cn/662205.htm
wqp.wfcj3t6.cn/848425.htm
wqp.wfcj3t6.cn/660465.htm
wqp.wfcj3t6.cn/846645.htm
wqo.wfcj3t6.cn/848085.htm
wqo.wfcj3t6.cn/868665.htm
wqo.wfcj3t6.cn/488265.htm
wqo.wfcj3t6.cn/119975.htm
wqo.wfcj3t6.cn/820085.htm
wqo.wfcj3t6.cn/046865.htm
wqo.wfcj3t6.cn/131935.htm
wqo.wfcj3t6.cn/822065.htm
wqo.wfcj3t6.cn/882045.htm
wqo.wfcj3t6.cn/004245.htm
wqi.wfcj3t6.cn/068005.htm
wqi.wfcj3t6.cn/828805.htm
wqi.wfcj3t6.cn/066865.htm
wqi.wfcj3t6.cn/391795.htm
wqi.wfcj3t6.cn/424605.htm
wqi.wfcj3t6.cn/577175.htm
wqi.wfcj3t6.cn/008485.htm
wqi.wfcj3t6.cn/208845.htm
wqi.wfcj3t6.cn/282085.htm
wqi.wfcj3t6.cn/862445.htm
wqu.wfcj3t6.cn/393595.htm
wqu.wfcj3t6.cn/440005.htm
wqu.wfcj3t6.cn/288425.htm
wqu.wfcj3t6.cn/519975.htm
wqu.wfcj3t6.cn/282885.htm
wqu.wfcj3t6.cn/820805.htm
wqu.wfcj3t6.cn/777955.htm
wqu.wfcj3t6.cn/046645.htm
wqu.wfcj3t6.cn/662485.htm
wqu.wfcj3t6.cn/620245.htm
wqy.wfcj3t6.cn/979935.htm
wqy.wfcj3t6.cn/264885.htm
wqy.wfcj3t6.cn/240885.htm
wqy.wfcj3t6.cn/266845.htm
wqy.wfcj3t6.cn/844085.htm
wqy.wfcj3t6.cn/579195.htm
wqy.wfcj3t6.cn/060025.htm
wqy.wfcj3t6.cn/551595.htm
wqy.wfcj3t6.cn/062225.htm
wqy.wfcj3t6.cn/606845.htm
wqt.wfcj3t6.cn/733975.htm
wqt.wfcj3t6.cn/084045.htm
wqt.wfcj3t6.cn/022265.htm
wqt.wfcj3t6.cn/420085.htm
wqt.wfcj3t6.cn/395735.htm
wqt.wfcj3t6.cn/597975.htm
wqt.wfcj3t6.cn/048645.htm
wqt.wfcj3t6.cn/668085.htm
wqt.wfcj3t6.cn/537375.htm
wqt.wfcj3t6.cn/262805.htm
wqr.wfcj3t6.cn/066425.htm
wqr.wfcj3t6.cn/882485.htm
wqr.wfcj3t6.cn/060445.htm
wqr.wfcj3t6.cn/317915.htm
wqr.wfcj3t6.cn/062865.htm
wqr.wfcj3t6.cn/151175.htm
wqr.wfcj3t6.cn/684625.htm
