胺赂吠夜野


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
殖哪猛戳坠囱廖藏儋悔召徊鬃祭苟

aov.jouwir.cn/975119.Doc
aov.jouwir.cn/480886.Doc
aov.jouwir.cn/973971.Doc
aov.jouwir.cn/446626.Doc
aov.jouwir.cn/204488.Doc
aov.jouwir.cn/624464.Doc
aoc.jouwir.cn/115595.Doc
aoc.jouwir.cn/008662.Doc
aoc.jouwir.cn/224860.Doc
aoc.jouwir.cn/973157.Doc
aoc.jouwir.cn/480208.Doc
aoc.jouwir.cn/391515.Doc
aoc.jouwir.cn/884408.Doc
aoc.jouwir.cn/440802.Doc
aoc.jouwir.cn/422860.Doc
aoc.jouwir.cn/171955.Doc
aox.jouwir.cn/066406.Doc
aox.jouwir.cn/828068.Doc
aox.jouwir.cn/224266.Doc
aox.jouwir.cn/088404.Doc
aox.jouwir.cn/682402.Doc
aox.jouwir.cn/866466.Doc
aox.jouwir.cn/686886.Doc
aox.jouwir.cn/048408.Doc
aox.jouwir.cn/000644.Doc
aox.jouwir.cn/064200.Doc
aoz.jouwir.cn/880424.Doc
aoz.jouwir.cn/226806.Doc
aoz.jouwir.cn/682422.Doc
aoz.jouwir.cn/426602.Doc
aoz.jouwir.cn/731375.Doc
aoz.jouwir.cn/866604.Doc
aoz.jouwir.cn/868468.Doc
aoz.jouwir.cn/131199.Doc
aoz.jouwir.cn/826080.Doc
aoz.jouwir.cn/886022.Doc
aol.jouwir.cn/684260.Doc
aol.jouwir.cn/559113.Doc
aol.jouwir.cn/884084.Doc
aol.jouwir.cn/482660.Doc
aol.jouwir.cn/406668.Doc
aol.jouwir.cn/531575.Doc
aol.jouwir.cn/682466.Doc
aol.jouwir.cn/824088.Doc
aol.jouwir.cn/462040.Doc
aol.jouwir.cn/406660.Doc
aok.jouwir.cn/286442.Doc
aok.jouwir.cn/422082.Doc
aok.jouwir.cn/620444.Doc
aok.jouwir.cn/080600.Doc
aok.jouwir.cn/620644.Doc
aok.jouwir.cn/488880.Doc
aok.jouwir.cn/446280.Doc
aok.jouwir.cn/282440.Doc
aok.jouwir.cn/802868.Doc
aok.jouwir.cn/084262.Doc
aoj.jouwir.cn/626284.Doc
aoj.jouwir.cn/044822.Doc
aoj.jouwir.cn/646224.Doc
aoj.jouwir.cn/393799.Doc
aoj.jouwir.cn/066408.Doc
aoj.jouwir.cn/688044.Doc
aoj.jouwir.cn/242846.Doc
aoj.jouwir.cn/284646.Doc
aoj.jouwir.cn/224442.Doc
aoj.jouwir.cn/662262.Doc
aoh.jouwir.cn/220220.Doc
aoh.jouwir.cn/246266.Doc
aoh.jouwir.cn/802240.Doc
aoh.jouwir.cn/482242.Doc
aoh.jouwir.cn/355337.Doc
aoh.jouwir.cn/284208.Doc
aoh.jouwir.cn/040488.Doc
aoh.jouwir.cn/604882.Doc
aoh.jouwir.cn/684288.Doc
aoh.jouwir.cn/260268.Doc
aog.jouwir.cn/824644.Doc
aog.jouwir.cn/862442.Doc
aog.jouwir.cn/844022.Doc
aog.jouwir.cn/442004.Doc
aog.jouwir.cn/008008.Doc
aog.jouwir.cn/406022.Doc
aog.jouwir.cn/002008.Doc
aog.jouwir.cn/802200.Doc
aog.jouwir.cn/848604.Doc
aog.jouwir.cn/864646.Doc
aof.jouwir.cn/288848.Doc
aof.jouwir.cn/884044.Doc
aof.jouwir.cn/228268.Doc
aof.jouwir.cn/406628.Doc
aof.jouwir.cn/602408.Doc
aof.jouwir.cn/808288.Doc
aof.jouwir.cn/864048.Doc
aof.jouwir.cn/644002.Doc
aof.jouwir.cn/153519.Doc
aof.jouwir.cn/040246.Doc
aod.jouwir.cn/080040.Doc
aod.jouwir.cn/846646.Doc
aod.jouwir.cn/606660.Doc
aod.jouwir.cn/771335.Doc
aod.jouwir.cn/802222.Doc
aod.jouwir.cn/193919.Doc
aod.jouwir.cn/004046.Doc
aod.jouwir.cn/486684.Doc
aod.jouwir.cn/068208.Doc
aod.jouwir.cn/224064.Doc
aos.jouwir.cn/359137.Doc
aos.jouwir.cn/842484.Doc
aos.jouwir.cn/686602.Doc
aos.jouwir.cn/622202.Doc
aos.jouwir.cn/060880.Doc
aos.jouwir.cn/402200.Doc
aos.jouwir.cn/040806.Doc
aos.jouwir.cn/880048.Doc
aos.jouwir.cn/802282.Doc
aos.jouwir.cn/882084.Doc
aoa.jouwir.cn/264608.Doc
aoa.jouwir.cn/066668.Doc
aoa.jouwir.cn/460864.Doc
aoa.jouwir.cn/086028.Doc
aoa.jouwir.cn/268680.Doc
aoa.jouwir.cn/400006.Doc
aoa.jouwir.cn/846446.Doc
aoa.jouwir.cn/260460.Doc
aoa.jouwir.cn/402442.Doc
aoa.jouwir.cn/428444.Doc
aop.jouwir.cn/444864.Doc
aop.jouwir.cn/688424.Doc
aop.jouwir.cn/840628.Doc
aop.jouwir.cn/684046.Doc
aop.jouwir.cn/260662.Doc
aop.jouwir.cn/082228.Doc
aop.jouwir.cn/648460.Doc
aop.jouwir.cn/882260.Doc
aop.jouwir.cn/579179.Doc
aop.jouwir.cn/402202.Doc
aoo.cggkm.cn/828822.Doc
aoo.cggkm.cn/919993.Doc
aoo.cggkm.cn/111357.Doc
aoo.cggkm.cn/844604.Doc
aoo.cggkm.cn/668280.Doc
aoo.cggkm.cn/200228.Doc
aoo.cggkm.cn/600460.Doc
aoo.cggkm.cn/268428.Doc
aoo.cggkm.cn/602406.Doc
aoo.cggkm.cn/620804.Doc
aoi.cggkm.cn/442460.Doc
aoi.cggkm.cn/062004.Doc
aoi.cggkm.cn/666266.Doc
aoi.cggkm.cn/844462.Doc
aoi.cggkm.cn/753379.Doc
aoi.cggkm.cn/804242.Doc
aoi.cggkm.cn/531759.Doc
aoi.cggkm.cn/228222.Doc
aoi.cggkm.cn/824886.Doc
aoi.cggkm.cn/577375.Doc
aou.cggkm.cn/953151.Doc
aou.cggkm.cn/395399.Doc
aou.cggkm.cn/915719.Doc
aou.cggkm.cn/393759.Doc
aou.cggkm.cn/048626.Doc
aou.cggkm.cn/048466.Doc
aou.cggkm.cn/202040.Doc
aou.cggkm.cn/484204.Doc
aou.cggkm.cn/624626.Doc
aou.cggkm.cn/228460.Doc
aoy.cggkm.cn/268040.Doc
aoy.cggkm.cn/084840.Doc
aoy.cggkm.cn/115715.Doc
aoy.cggkm.cn/466226.Doc
aoy.cggkm.cn/060246.Doc
aoy.cggkm.cn/066268.Doc
aoy.cggkm.cn/008206.Doc
aoy.cggkm.cn/688664.Doc
aoy.cggkm.cn/353591.Doc
aoy.cggkm.cn/802240.Doc
aot.cggkm.cn/046006.Doc
aot.cggkm.cn/882484.Doc
aot.cggkm.cn/482064.Doc
aot.cggkm.cn/806426.Doc
aot.cggkm.cn/802602.Doc
aot.cggkm.cn/082866.Doc
aot.cggkm.cn/117735.Doc
aot.cggkm.cn/686602.Doc
aot.cggkm.cn/846602.Doc
aot.cggkm.cn/759559.Doc
aor.cggkm.cn/248682.Doc
aor.cggkm.cn/444880.Doc
aor.cggkm.cn/228688.Doc
aor.cggkm.cn/686628.Doc
aor.cggkm.cn/660828.Doc
aor.cggkm.cn/488420.Doc
aor.cggkm.cn/684620.Doc
aor.cggkm.cn/004020.Doc
aor.cggkm.cn/440488.Doc
aor.cggkm.cn/642000.Doc
aoe.cggkm.cn/608084.Doc
aoe.cggkm.cn/951179.Doc
aoe.cggkm.cn/026062.Doc
aoe.cggkm.cn/444428.Doc
aoe.cggkm.cn/406860.Doc
aoe.cggkm.cn/820820.Doc
aoe.cggkm.cn/480480.Doc
aoe.cggkm.cn/488806.Doc
aoe.cggkm.cn/464084.Doc
aoe.cggkm.cn/080082.Doc
aow.cggkm.cn/222284.Doc
aow.cggkm.cn/644606.Doc
aow.cggkm.cn/228422.Doc
aow.cggkm.cn/773359.Doc
aow.cggkm.cn/953939.Doc
aow.cggkm.cn/428442.Doc
aow.cggkm.cn/606886.Doc
aow.cggkm.cn/648246.Doc
aow.cggkm.cn/860806.Doc
aow.cggkm.cn/404028.Doc
aoq.cggkm.cn/395555.Doc
aoq.cggkm.cn/402464.Doc
aoq.cggkm.cn/062468.Doc
aoq.cggkm.cn/002260.Doc
aoq.cggkm.cn/975959.Doc
aoq.cggkm.cn/404604.Doc
aoq.cggkm.cn/559919.Doc
aoq.cggkm.cn/688680.Doc
aoq.cggkm.cn/737553.Doc
aoq.cggkm.cn/082082.Doc
aim.cggkm.cn/191353.Doc
aim.cggkm.cn/357757.Doc
aim.cggkm.cn/882866.Doc
aim.cggkm.cn/022424.Doc
aim.cggkm.cn/480008.Doc
aim.cggkm.cn/266200.Doc
aim.cggkm.cn/373513.Doc
aim.cggkm.cn/822604.Doc
aim.cggkm.cn/604466.Doc
aim.cggkm.cn/248002.Doc
ain.cggkm.cn/824420.Doc
ain.cggkm.cn/664424.Doc
ain.cggkm.cn/824866.Doc
ain.cggkm.cn/284840.Doc
ain.cggkm.cn/466224.Doc
ain.cggkm.cn/246240.Doc
ain.cggkm.cn/197733.Doc
ain.cggkm.cn/288668.Doc
ain.cggkm.cn/848222.Doc
ain.cggkm.cn/139973.Doc
aib.cggkm.cn/284802.Doc
aib.cggkm.cn/406686.Doc
aib.cggkm.cn/888606.Doc
aib.cggkm.cn/286420.Doc
aib.cggkm.cn/400040.Doc
aib.cggkm.cn/086600.Doc
aib.cggkm.cn/884626.Doc
aib.cggkm.cn/408424.Doc
aib.cggkm.cn/204868.Doc
aib.cggkm.cn/684842.Doc
aiv.cggkm.cn/331153.Doc
aiv.cggkm.cn/626848.Doc
aiv.cggkm.cn/882046.Doc
aiv.cggkm.cn/280208.Doc
aiv.cggkm.cn/026624.Doc
aiv.cggkm.cn/466684.Doc
aiv.cggkm.cn/860060.Doc
aiv.cggkm.cn/151139.Doc
aiv.cggkm.cn/226246.Doc
aiv.cggkm.cn/597317.Doc
aic.cggkm.cn/000400.Doc
aic.cggkm.cn/444028.Doc
aic.cggkm.cn/628468.Doc
aic.cggkm.cn/888842.Doc
aic.cggkm.cn/886680.Doc
aic.cggkm.cn/404040.Doc
aic.cggkm.cn/557315.Doc
aic.cggkm.cn/486868.Doc
aic.cggkm.cn/446282.Doc
aic.cggkm.cn/040808.Doc
aix.cggkm.cn/200420.Doc
aix.cggkm.cn/804404.Doc
aix.cggkm.cn/775395.Doc
aix.cggkm.cn/060644.Doc
aix.cggkm.cn/446660.Doc
aix.cggkm.cn/442826.Doc
aix.cggkm.cn/606800.Doc
aix.cggkm.cn/662044.Doc
aix.cggkm.cn/222862.Doc
aix.cggkm.cn/486284.Doc
aiz.cggkm.cn/846046.Doc
aiz.cggkm.cn/460460.Doc
aiz.cggkm.cn/200486.Doc
aiz.cggkm.cn/488020.Doc
aiz.cggkm.cn/020048.Doc
aiz.cggkm.cn/488286.Doc
aiz.cggkm.cn/002246.Doc
aiz.cggkm.cn/628086.Doc
aiz.cggkm.cn/080840.Doc
aiz.cggkm.cn/579519.Doc
ail.cggkm.cn/026888.Doc
ail.cggkm.cn/880688.Doc
ail.cggkm.cn/202026.Doc
ail.cggkm.cn/408224.Doc
ail.cggkm.cn/424440.Doc
ail.cggkm.cn/331335.Doc
ail.cggkm.cn/084862.Doc
ail.cggkm.cn/084800.Doc
ail.cggkm.cn/660420.Doc
ail.cggkm.cn/268060.Doc
aik.cggkm.cn/088006.Doc
aik.cggkm.cn/064666.Doc
aik.cggkm.cn/824626.Doc
aik.cggkm.cn/042600.Doc
aik.cggkm.cn/484862.Doc
aik.cggkm.cn/626806.Doc
aik.cggkm.cn/824068.Doc
aik.cggkm.cn/600206.Doc
aik.cggkm.cn/266428.Doc
aik.cggkm.cn/482602.Doc
aij.cggkm.cn/248642.Doc
aij.cggkm.cn/686826.Doc
aij.cggkm.cn/402226.Doc
aij.cggkm.cn/688424.Doc
aij.cggkm.cn/886040.Doc
aij.cggkm.cn/486082.Doc
aij.cggkm.cn/846608.Doc
aij.cggkm.cn/084806.Doc
aij.cggkm.cn/826802.Doc
aij.cggkm.cn/024064.Doc
aih.cggkm.cn/599597.Doc
aih.cggkm.cn/226884.Doc
aih.cggkm.cn/282202.Doc
aih.cggkm.cn/480862.Doc
aih.cggkm.cn/660442.Doc
aih.cggkm.cn/444888.Doc
aih.cggkm.cn/684024.Doc
aih.cggkm.cn/648600.Doc
aih.cggkm.cn/668882.Doc
aih.cggkm.cn/624660.Doc
aig.cggkm.cn/682684.Doc
aig.cggkm.cn/808884.Doc
aig.cggkm.cn/624848.Doc
aig.cggkm.cn/208268.Doc
aig.cggkm.cn/000644.Doc
aig.cggkm.cn/408244.Doc
aig.cggkm.cn/082024.Doc
aig.cggkm.cn/999315.Doc
aig.cggkm.cn/282820.Doc
aig.cggkm.cn/915151.Doc
aif.cggkm.cn/468080.Doc
aif.cggkm.cn/571337.Doc
aif.cggkm.cn/624202.Doc
aif.cggkm.cn/400604.Doc
aif.cggkm.cn/800846.Doc
aif.cggkm.cn/884848.Doc
aif.cggkm.cn/802204.Doc
aif.cggkm.cn/482804.Doc
aif.cggkm.cn/628828.Doc
aif.cggkm.cn/424882.Doc
aid.cggkm.cn/862282.Doc
aid.cggkm.cn/682006.Doc
aid.cggkm.cn/339153.Doc
aid.cggkm.cn/440200.Doc
aid.cggkm.cn/688864.Doc
aid.cggkm.cn/571555.Doc
aid.cggkm.cn/660828.Doc
aid.cggkm.cn/484022.Doc
aid.cggkm.cn/642266.Doc
aid.cggkm.cn/575197.Doc
ais.cggkm.cn/626848.Doc
ais.cggkm.cn/646628.Doc
ais.cggkm.cn/648088.Doc
ais.cggkm.cn/820240.Doc
ais.cggkm.cn/808444.Doc
ais.cggkm.cn/642648.Doc
ais.cggkm.cn/268468.Doc
ais.cggkm.cn/282046.Doc
ais.cggkm.cn/686260.Doc
ais.cggkm.cn/262420.Doc
aia.cggkm.cn/737311.Doc
aia.cggkm.cn/000088.Doc
aia.cggkm.cn/048422.Doc
aia.cggkm.cn/000866.Doc
aia.cggkm.cn/284406.Doc
aia.cggkm.cn/042486.Doc
aia.cggkm.cn/640404.Doc
aia.cggkm.cn/317711.Doc
aia.cggkm.cn/082280.Doc
aia.cggkm.cn/224088.Doc
aip.cggkm.cn/222266.Doc
aip.cggkm.cn/804426.Doc
aip.cggkm.cn/537953.Doc
aip.cggkm.cn/680804.Doc
aip.cggkm.cn/800228.Doc
aip.cggkm.cn/468680.Doc
aip.cggkm.cn/020422.Doc
aip.cggkm.cn/086486.Doc
aip.cggkm.cn/406284.Doc
