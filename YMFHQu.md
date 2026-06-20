闭陀榔录彝


Python 枚举类型高级用法
=============================

enum 模块提供了类型安全的枚举类，远比使用字符串或整数常量更可靠。
本文覆盖从基础到高级的全部特性。

1. Enum 基础
-----------------
定义符号常量，成员名称唯一，值可重复（此时为别名）。

from enum import Enum

class Color(Enum):
    """颜色枚举——最基础的枚举使用"""
    RED = 1
    GREEN = 2
    BLUE = 3

# 访问方式
print(Color.RED)           # Color.RED
print(Color.RED.name)      # "RED"
print(Color.RED.value)     # 1

# 通过值反向查找
print(Color(1))            # Color.RED——ValueError 如果不存在
print(Color["RED"])        # Color.RED——KeyError 如果不存在

# 枚举成员是单例的
print(Color.RED is Color(1))  # True


2. IntEnum / StrEnum
-------------------------
IntEnum 成员可直接用于整数运算；StrEnum (3.11+) 成员是字符串子类。

from enum import IntEnum

class HttpStatus(IntEnum):
    """HTTP 状态码——IntEnum 成员可直接参与整数运算"""
    OK = 200
    NOT_FOUND = 404
    INTERNAL_ERROR = 500


# IntEnum 可以直接和整数比较
print(HttpStatus.OK == 200)     # True——普通 Enum 不允许
print(HttpStatus.OK + 100)      # 300——可以用在数学运算中
print(HttpStatus(404))          # HttpStatus.NOT_FOUND


# StrEnum (Python 3.11+)
try:
    from enum import StrEnum

    class LogLevel(StrEnum):
        """日志级别——StrEnum 成员是字符串子类"""
        DEBUG = "DEBUG"
        INFO = "INFO"
        WARNING = "WARNING"
        ERROR = "ERROR"

    level = LogLevel.INFO
    print(level.lower())       # "info"——可以调用字符串方法
    print(level == "INFO")    # True——可直接与字符串比较

except ImportError:
    print("StrEnum 需要 Python 3.11+")


3. Flag / IntFlag——位掩码枚举
---------------------------------
Flag 支持位运算组合多个枚举值。

from enum import Flag, auto

class Permission(Flag):
    """权限——使用 Flag 实现位掩码"""
    NONE = 0
    READ = auto()     # 1
    WRITE = auto()    # 2
    EXECUTE = auto()  # 4
    ALL = READ | WRITE | EXECUTE  # 7


# 组合权限
user_perm = Permission.READ | Permission.WRITE
print(user_perm)           # Permission.READ|WRITE

# 检查权限
print(user_perm & Permission.READ)     # Permission.READ (非零→有权限)
print(bool(user_perm & Permission.READ))   # True
print(bool(user_perm & Permission.EXECUTE))  # False

# 增删权限
user_perm |= Permission.EXECUTE   # 添加执行权限
user_perm &= ~Permission.WRITE    # 移除写入权限
print(user_perm)                  # Permission.READ|EXECUTE

# 遍历组合中的每个 Flag
for perm in Permission:
    print(f"{perm.name} = {perm.value}")


class AccessMode(IntFlag):
    """IntFlag——兼容整数的位标志"""
    READ = 4
    WRITE = 2
    EXECUTE = 1

print(AccessMode.READ | AccessMode.WRITE)  # 6
print((AccessMode.READ | AccessMode.WRITE) == 6)  # True


4. auto() 自动赋值
--------------------
auto() 自动为成员分配值，规则是依次加 1。

from enum import auto, Enum

class Priority(Enum):
    """优先级——auto 自动赋值"""
    LOW = auto()       # 1
    MEDIUM = auto()    # 2
    HIGH = auto()      # 3
    CRITICAL = auto()  # 4

for p in Priority:
    print(f"{p.name} = {p.value}")

# 对于 Flag，auto() 使用位移（1, 2, 4, 8...）
class AnimalFeature(Flag):
    """动物特征——auto 在 Flag 中自动位移"""
    HAS_FUR = auto()       # 1
    HAS_FEATHERS = auto()  # 2
    HAS_SCALES = auto()    # 4
    LAYS_EGGS = auto()     # 8


5. unique() 与别名
---------------------
@unique 装饰器禁止重复值，默认情况下重复值创建别名。

from enum import unique

# 默认：重复值会变成别名
class Status(Enum):
    PENDING = 1
    WAITING = 1   # 这是 PENDING 的别名
    DONE = 2

print(Status.WAITING)           # Status.PENDING——指向同一成员
print(Status.WAITING is Status.PENDING)  # True——是同一个对象
print(list(Status))             # [Status.PENDING, Status.DONE]——WAITING 被跳过


# @unique：禁止重复值
try:
    @unique
    class StrictStatus(Enum):
        PENDING = 1
        WAITING = 1   # ValueError: duplicate values!
        DONE = 2
except ValueError as e:
    print(f"重复值被禁止: {e}")


6. 枚举添加方法
-------------------
枚举类可以定义方法，每个成员共享。

class OrderStatus(Enum):
    """订单状态——带有业务方法的枚举"""
    PENDING = 1
    PAID = 2
    SHIPPED = 3
    DELIVERED = 4
    CANCELLED = 5

    def can_cancel(self) -> bool:
        """是否可以取消——根据状态判断业务规则"""
        return self in (OrderStatus.PENDING, OrderStatus.PAID)

    @property
    def description(self) -> str:
        """属性——返回状态描述"""
        descriptions = {
            OrderStatus.PENDING: "等待支付",
            OrderStatus.PAID: "已付款",
            OrderStatus.SHIPPED: "已发货",
            OrderStatus.DELIVERED: "已送达",
            OrderStatus.CANCELLED: "已取消",
        }
        return descriptions[self]


s = OrderStatus.PENDING
print(s.description)    # "等待支付"
print(s.can_cancel())   # True

s = OrderStatus.SHIPPED
print(s.can_cancel())   # False


7. __init__ 自定义值
-----------------------
通过 __init__ 为每个成员附加更多属性。

class Planet(Enum):
    """行星——__init__ 自定义更多属性"""
    MERCURY = (3.303e+23, 2.4397e6)
    VENUS = (4.869e+24, 6.0518e6)
    EARTH = (5.976e+24, 6.37814e6)
    MARS = (6.421e+23, 3.3972e6)

    def __init__(self, mass: float, radius: float):
        self.mass = mass       # 质量（kg）
        self.radius = radius   # 半径（m）

    @property
    def surface_gravity(self) -> float:
        """计算表面重力"""
        G = 6.67430e-11       # 万有引力常数
        return G * self.mass / (self.radius ** 2)


print(Planet.EARTH.mass)            # 5.976e+24
print(f"{Planet.EARTH.surface_gravity:.2f} m/s²")  # 9.80 m/s²


8. 比较与哈希
----------------
枚举成员可比较相等性，可作为字典键和集合成员。

# 枚举成员可哈希
status_set = {OrderStatus.PENDING, OrderStatus.PAID, OrderStatus.PENDING}
print(len(status_set))              # 2——重复被去重

status_dict = {
    OrderStatus.PENDING: "等待中",
    OrderStatus.PAID: "已完成付款",
}
print(status_dict[OrderStatus.PENDING])  # "等待中"

# 比较：按成员身份比较（不是按值）
print(OrderStatus.PENDING == OrderStatus.PENDING)  # True
print(OrderStatus.PENDING == 1)                    # False（普通 Enum）


9. 类型提示中的枚举
-----------------------
枚举在类型提示中提供精确的值约束。

from typing import Literal

def set_status(status: OrderStatus) -> None:
    """函数签名明确指定了允许的值范围"""
    if status == OrderStatus.PENDING:
        print("订单已创建")
    elif status == OrderStatus.PAID:
        print("订单已付款")


# 枚举与 Literal 类型结合
def set_mode(mode: Literal[Color.RED, Color.BLUE]) -> None:
    """更精确的值约束"""
    print(f"设置模式: {mode}")


总结：enum 模块提供了比普通常量更强的类型安全性。Enum 适合符号常量，
IntEnum/StrEnum 兼容原始类型，Flag 实现位掩码。通过添加方法和
__init__ 可以附加业务逻辑，unique() 确保值唯一。

严任古迟涨谌粕诓粱永囱品惩什汲

tv.blog.cvvliy.cn/Article/details/395311.sHtML
tv.blog.cvvliy.cn/Article/details/115917.sHtML
tv.blog.cvvliy.cn/Article/details/191731.sHtML
tv.blog.cvvliy.cn/Article/details/640266.sHtML
tv.blog.cvvliy.cn/Article/details/915335.sHtML
tv.blog.cvvliy.cn/Article/details/719133.sHtML
tv.blog.cvvliy.cn/Article/details/517559.sHtML
tv.blog.cvvliy.cn/Article/details/193997.sHtML
tv.blog.cvvliy.cn/Article/details/373199.sHtML
tv.blog.cvvliy.cn/Article/details/131773.sHtML
tv.blog.cvvliy.cn/Article/details/442486.sHtML
tv.blog.cvvliy.cn/Article/details/175173.sHtML
tv.blog.cvvliy.cn/Article/details/537999.sHtML
tv.blog.cvvliy.cn/Article/details/868048.sHtML
tv.blog.cvvliy.cn/Article/details/117111.sHtML
tv.blog.cvvliy.cn/Article/details/155557.sHtML
tv.blog.cvvliy.cn/Article/details/515791.sHtML
tv.blog.cvvliy.cn/Article/details/288824.sHtML
tv.blog.cvvliy.cn/Article/details/319373.sHtML
tv.blog.cvvliy.cn/Article/details/082620.sHtML
tv.blog.cvvliy.cn/Article/details/622248.sHtML
tv.blog.cvvliy.cn/Article/details/420062.sHtML
tv.blog.cvvliy.cn/Article/details/517157.sHtML
tv.blog.cvvliy.cn/Article/details/820282.sHtML
tv.blog.cvvliy.cn/Article/details/935751.sHtML
tv.blog.cvvliy.cn/Article/details/668042.sHtML
tv.blog.cvvliy.cn/Article/details/862004.sHtML
tv.blog.cvvliy.cn/Article/details/440066.sHtML
tv.blog.cvvliy.cn/Article/details/357997.sHtML
tv.blog.cvvliy.cn/Article/details/004680.sHtML
tv.blog.cvvliy.cn/Article/details/626224.sHtML
tv.blog.cvvliy.cn/Article/details/868286.sHtML
tv.blog.cvvliy.cn/Article/details/626608.sHtML
tv.blog.cvvliy.cn/Article/details/919713.sHtML
tv.blog.cvvliy.cn/Article/details/260848.sHtML
tv.blog.cvvliy.cn/Article/details/571739.sHtML
tv.blog.cvvliy.cn/Article/details/404442.sHtML
tv.blog.cvvliy.cn/Article/details/868224.sHtML
tv.blog.cvvliy.cn/Article/details/997753.sHtML
tv.blog.cvvliy.cn/Article/details/642884.sHtML
tv.blog.cvvliy.cn/Article/details/933759.sHtML
tv.blog.cvvliy.cn/Article/details/337139.sHtML
tv.blog.cvvliy.cn/Article/details/422260.sHtML
tv.blog.cvvliy.cn/Article/details/824008.sHtML
tv.blog.cvvliy.cn/Article/details/422664.sHtML
tv.blog.cvvliy.cn/Article/details/468020.sHtML
tv.blog.cvvliy.cn/Article/details/731939.sHtML
tv.blog.cvvliy.cn/Article/details/315519.sHtML
tv.blog.cvvliy.cn/Article/details/519317.sHtML
tv.blog.cvvliy.cn/Article/details/397317.sHtML
tv.blog.cvvliy.cn/Article/details/595513.sHtML
tv.blog.cvvliy.cn/Article/details/666486.sHtML
tv.blog.cvvliy.cn/Article/details/537955.sHtML
tv.blog.cvvliy.cn/Article/details/268280.sHtML
tv.blog.cvvliy.cn/Article/details/579157.sHtML
tv.blog.cvvliy.cn/Article/details/131317.sHtML
tv.blog.cvvliy.cn/Article/details/004682.sHtML
tv.blog.cvvliy.cn/Article/details/268066.sHtML
tv.blog.cvvliy.cn/Article/details/379399.sHtML
tv.blog.cvvliy.cn/Article/details/133791.sHtML
tv.blog.cvvliy.cn/Article/details/111511.sHtML
tv.blog.cvvliy.cn/Article/details/446000.sHtML
tv.blog.cvvliy.cn/Article/details/206062.sHtML
tv.blog.cvvliy.cn/Article/details/004822.sHtML
tv.blog.cvvliy.cn/Article/details/060840.sHtML
tv.blog.cvvliy.cn/Article/details/202286.sHtML
tv.blog.cvvliy.cn/Article/details/042840.sHtML
tv.blog.cvvliy.cn/Article/details/462664.sHtML
tv.blog.cvvliy.cn/Article/details/204804.sHtML
tv.blog.cvvliy.cn/Article/details/868680.sHtML
tv.blog.cvvliy.cn/Article/details/668660.sHtML
tv.blog.cvvliy.cn/Article/details/357173.sHtML
tv.blog.cvvliy.cn/Article/details/937155.sHtML
tv.blog.cvvliy.cn/Article/details/333335.sHtML
tv.blog.cvvliy.cn/Article/details/997391.sHtML
tv.blog.cvvliy.cn/Article/details/377313.sHtML
tv.blog.cvvliy.cn/Article/details/359717.sHtML
tv.blog.cvvliy.cn/Article/details/680404.sHtML
tv.blog.cvvliy.cn/Article/details/884260.sHtML
tv.blog.cvvliy.cn/Article/details/171711.sHtML
tv.blog.cvvliy.cn/Article/details/313713.sHtML
tv.blog.cvvliy.cn/Article/details/173595.sHtML
tv.blog.cvvliy.cn/Article/details/975559.sHtML
tv.blog.cvvliy.cn/Article/details/684802.sHtML
tv.blog.cvvliy.cn/Article/details/888202.sHtML
tv.blog.cvvliy.cn/Article/details/933959.sHtML
tv.blog.cvvliy.cn/Article/details/684448.sHtML
tv.blog.cvvliy.cn/Article/details/442486.sHtML
tv.blog.cvvliy.cn/Article/details/739339.sHtML
tv.blog.cvvliy.cn/Article/details/179593.sHtML
tv.blog.cvvliy.cn/Article/details/084286.sHtML
tv.blog.cvvliy.cn/Article/details/991177.sHtML
tv.blog.cvvliy.cn/Article/details/086682.sHtML
tv.blog.cvvliy.cn/Article/details/577737.sHtML
tv.blog.cvvliy.cn/Article/details/319797.sHtML
tv.blog.cvvliy.cn/Article/details/048688.sHtML
tv.blog.cvvliy.cn/Article/details/717397.sHtML
tv.blog.cvvliy.cn/Article/details/028804.sHtML
tv.blog.cvvliy.cn/Article/details/731119.sHtML
tv.blog.cvvliy.cn/Article/details/062844.sHtML
tv.blog.cvvliy.cn/Article/details/053191.sHtML
tv.blog.cvvliy.cn/Article/details/735157.sHtML
tv.blog.cvvliy.cn/Article/details/262044.sHtML
tv.blog.cvvliy.cn/Article/details/975993.sHtML
tv.blog.cvvliy.cn/Article/details/579115.sHtML
tv.blog.cvvliy.cn/Article/details/648260.sHtML
tv.blog.cvvliy.cn/Article/details/826844.sHtML
tv.blog.cvvliy.cn/Article/details/551353.sHtML
tv.blog.cvvliy.cn/Article/details/353133.sHtML
tv.blog.cvvliy.cn/Article/details/226066.sHtML
tv.blog.cvvliy.cn/Article/details/557715.sHtML
tv.blog.cvvliy.cn/Article/details/757959.sHtML
tv.blog.cvvliy.cn/Article/details/088466.sHtML
tv.blog.cvvliy.cn/Article/details/820800.sHtML
tv.blog.cvvliy.cn/Article/details/711751.sHtML
tv.blog.cvvliy.cn/Article/details/860068.sHtML
tv.blog.cvvliy.cn/Article/details/088620.sHtML
tv.blog.cvvliy.cn/Article/details/040848.sHtML
tv.blog.cvvliy.cn/Article/details/975535.sHtML
tv.blog.cvvliy.cn/Article/details/191195.sHtML
tv.blog.cvvliy.cn/Article/details/335573.sHtML
tv.blog.cvvliy.cn/Article/details/420624.sHtML
tv.blog.cvvliy.cn/Article/details/288642.sHtML
tv.blog.cvvliy.cn/Article/details/371537.sHtML
tv.blog.cvvliy.cn/Article/details/820286.sHtML
tv.blog.cvvliy.cn/Article/details/602042.sHtML
tv.blog.cvvliy.cn/Article/details/799753.sHtML
tv.blog.cvvliy.cn/Article/details/977195.sHtML
tv.blog.cvvliy.cn/Article/details/462440.sHtML
tv.blog.cvvliy.cn/Article/details/220828.sHtML
tv.blog.cvvliy.cn/Article/details/662844.sHtML
tv.blog.cvvliy.cn/Article/details/733591.sHtML
tv.blog.cvvliy.cn/Article/details/208886.sHtML
tv.blog.cvvliy.cn/Article/details/602000.sHtML
tv.blog.cvvliy.cn/Article/details/793317.sHtML
tv.blog.cvvliy.cn/Article/details/464262.sHtML
tv.blog.cvvliy.cn/Article/details/975539.sHtML
tv.blog.cvvliy.cn/Article/details/911913.sHtML
tv.blog.cvvliy.cn/Article/details/979993.sHtML
tv.blog.cvvliy.cn/Article/details/131513.sHtML
tv.blog.cvvliy.cn/Article/details/866620.sHtML
tv.blog.cvvliy.cn/Article/details/917797.sHtML
tv.blog.cvvliy.cn/Article/details/519737.sHtML
tv.blog.cvvliy.cn/Article/details/804640.sHtML
tv.blog.cvvliy.cn/Article/details/642828.sHtML
tv.blog.cvvliy.cn/Article/details/602060.sHtML
tv.blog.cvvliy.cn/Article/details/975559.sHtML
tv.blog.cvvliy.cn/Article/details/686686.sHtML
tv.blog.cvvliy.cn/Article/details/353759.sHtML
tv.blog.cvvliy.cn/Article/details/680464.sHtML
tv.blog.cvvliy.cn/Article/details/133575.sHtML
tv.blog.cvvliy.cn/Article/details/262862.sHtML
tv.blog.cvvliy.cn/Article/details/048828.sHtML
tv.blog.cvvliy.cn/Article/details/535991.sHtML
tv.blog.cvvliy.cn/Article/details/802448.sHtML
tv.blog.cvvliy.cn/Article/details/195737.sHtML
tv.blog.cvvliy.cn/Article/details/220226.sHtML
tv.blog.cvvliy.cn/Article/details/537371.sHtML
tv.blog.cvvliy.cn/Article/details/911313.sHtML
tv.blog.cvvliy.cn/Article/details/080822.sHtML
tv.blog.cvvliy.cn/Article/details/333351.sHtML
tv.blog.cvvliy.cn/Article/details/826664.sHtML
tv.blog.cvvliy.cn/Article/details/042408.sHtML
tv.blog.cvvliy.cn/Article/details/260006.sHtML
tv.blog.cvvliy.cn/Article/details/317511.sHtML
tv.blog.cvvliy.cn/Article/details/799373.sHtML
tv.blog.cvvliy.cn/Article/details/406828.sHtML
tv.blog.cvvliy.cn/Article/details/282026.sHtML
tv.blog.cvvliy.cn/Article/details/935995.sHtML
tv.blog.cvvliy.cn/Article/details/848682.sHtML
tv.blog.cvvliy.cn/Article/details/288442.sHtML
tv.blog.cvvliy.cn/Article/details/175931.sHtML
tv.blog.cvvliy.cn/Article/details/537313.sHtML
tv.blog.cvvliy.cn/Article/details/555795.sHtML
tv.blog.cvvliy.cn/Article/details/202400.sHtML
tv.blog.cvvliy.cn/Article/details/620226.sHtML
tv.blog.cvvliy.cn/Article/details/973393.sHtML
tv.blog.cvvliy.cn/Article/details/086202.sHtML
tv.blog.cvvliy.cn/Article/details/173591.sHtML
tv.blog.cvvliy.cn/Article/details/868064.sHtML
tv.blog.cvvliy.cn/Article/details/753979.sHtML
tv.blog.cvvliy.cn/Article/details/660608.sHtML
tv.blog.cvvliy.cn/Article/details/284222.sHtML
tv.blog.cvvliy.cn/Article/details/779773.sHtML
tv.blog.cvvliy.cn/Article/details/311797.sHtML
tv.blog.cvvliy.cn/Article/details/248026.sHtML
tv.blog.cvvliy.cn/Article/details/888682.sHtML
tv.blog.cvvliy.cn/Article/details/137913.sHtML
tv.blog.cvvliy.cn/Article/details/195717.sHtML
tv.blog.cvvliy.cn/Article/details/755137.sHtML
tv.blog.cvvliy.cn/Article/details/040864.sHtML
tv.blog.cvvliy.cn/Article/details/420820.sHtML
tv.blog.cvvliy.cn/Article/details/242064.sHtML
tv.blog.cvvliy.cn/Article/details/606840.sHtML
tv.blog.cvvliy.cn/Article/details/046008.sHtML
tv.blog.cvvliy.cn/Article/details/282402.sHtML
tv.blog.cvvliy.cn/Article/details/288428.sHtML
tv.blog.cvvliy.cn/Article/details/048882.sHtML
tv.blog.cvvliy.cn/Article/details/828442.sHtML
tv.blog.cvvliy.cn/Article/details/533931.sHtML
tv.blog.cvvliy.cn/Article/details/771311.sHtML
tv.blog.cvvliy.cn/Article/details/480066.sHtML
tv.blog.cvvliy.cn/Article/details/771591.sHtML
tv.blog.cvvliy.cn/Article/details/806244.sHtML
tv.blog.cvvliy.cn/Article/details/999159.sHtML
tv.blog.cvvliy.cn/Article/details/460220.sHtML
tv.blog.cvvliy.cn/Article/details/939199.sHtML
tv.blog.cvvliy.cn/Article/details/137979.sHtML
tv.blog.cvvliy.cn/Article/details/153953.sHtML
tv.blog.cvvliy.cn/Article/details/555357.sHtML
tv.blog.cvvliy.cn/Article/details/284286.sHtML
tv.blog.cvvliy.cn/Article/details/913919.sHtML
tv.blog.cvvliy.cn/Article/details/717313.sHtML
tv.blog.cvvliy.cn/Article/details/888082.sHtML
tv.blog.cvvliy.cn/Article/details/006000.sHtML
tv.blog.cvvliy.cn/Article/details/597311.sHtML
tv.blog.cvvliy.cn/Article/details/422284.sHtML
tv.blog.cvvliy.cn/Article/details/884082.sHtML
tv.blog.cvvliy.cn/Article/details/571579.sHtML
tv.blog.cvvliy.cn/Article/details/468042.sHtML
tv.blog.cvvliy.cn/Article/details/735919.sHtML
tv.blog.cvvliy.cn/Article/details/133313.sHtML
tv.blog.cvvliy.cn/Article/details/393171.sHtML
tv.blog.cvvliy.cn/Article/details/802462.sHtML
tv.blog.cvvliy.cn/Article/details/664242.sHtML
tv.blog.cvvliy.cn/Article/details/135731.sHtML
tv.blog.cvvliy.cn/Article/details/462668.sHtML
tv.blog.cvvliy.cn/Article/details/842086.sHtML
tv.blog.cvvliy.cn/Article/details/428204.sHtML
tv.blog.cvvliy.cn/Article/details/060420.sHtML
tv.blog.cvvliy.cn/Article/details/779713.sHtML
tv.blog.cvvliy.cn/Article/details/420420.sHtML
tv.blog.cvvliy.cn/Article/details/802626.sHtML
tv.blog.cvvliy.cn/Article/details/999111.sHtML
tv.blog.cvvliy.cn/Article/details/151137.sHtML
tv.blog.cvvliy.cn/Article/details/395155.sHtML
tv.blog.cvvliy.cn/Article/details/626644.sHtML
tv.blog.cvvliy.cn/Article/details/640260.sHtML
tv.blog.cvvliy.cn/Article/details/115931.sHtML
tv.blog.cvvliy.cn/Article/details/155595.sHtML
tv.blog.cvvliy.cn/Article/details/595357.sHtML
tv.blog.cvvliy.cn/Article/details/399795.sHtML
tv.blog.cvvliy.cn/Article/details/086848.sHtML
tv.blog.cvvliy.cn/Article/details/539173.sHtML
tv.blog.cvvliy.cn/Article/details/284606.sHtML
tv.blog.cvvliy.cn/Article/details/953135.sHtML
tv.blog.cvvliy.cn/Article/details/260004.sHtML
tv.blog.cvvliy.cn/Article/details/282668.sHtML
tv.blog.cvvliy.cn/Article/details/371313.sHtML
tv.blog.cvvliy.cn/Article/details/664402.sHtML
tv.blog.cvvliy.cn/Article/details/004206.sHtML
tv.blog.cvvliy.cn/Article/details/331559.sHtML
tv.blog.cvvliy.cn/Article/details/480240.sHtML
tv.blog.cvvliy.cn/Article/details/199157.sHtML
tv.blog.cvvliy.cn/Article/details/486020.sHtML
tv.blog.cvvliy.cn/Article/details/680460.sHtML
tv.blog.cvvliy.cn/Article/details/046684.sHtML
tv.blog.cvvliy.cn/Article/details/971913.sHtML
tv.blog.cvvliy.cn/Article/details/753915.sHtML
tv.blog.cvvliy.cn/Article/details/286626.sHtML
tv.blog.cvvliy.cn/Article/details/151377.sHtML
tv.blog.cvvliy.cn/Article/details/533373.sHtML
tv.blog.cvvliy.cn/Article/details/460466.sHtML
tv.blog.cvvliy.cn/Article/details/442882.sHtML
tv.blog.cvvliy.cn/Article/details/680864.sHtML
tv.blog.cvvliy.cn/Article/details/664886.sHtML
tv.blog.cvvliy.cn/Article/details/519799.sHtML
tv.blog.cvvliy.cn/Article/details/222806.sHtML
tv.blog.cvvliy.cn/Article/details/462486.sHtML
tv.blog.cvvliy.cn/Article/details/400642.sHtML
tv.blog.cvvliy.cn/Article/details/426448.sHtML
tv.blog.cvvliy.cn/Article/details/997399.sHtML
tv.blog.cvvliy.cn/Article/details/426608.sHtML
tv.blog.cvvliy.cn/Article/details/226608.sHtML
tv.blog.cvvliy.cn/Article/details/468226.sHtML
tv.blog.cvvliy.cn/Article/details/068686.sHtML
tv.blog.cvvliy.cn/Article/details/604840.sHtML
tv.blog.cvvliy.cn/Article/details/195933.sHtML
tv.blog.cvvliy.cn/Article/details/644660.sHtML
tv.blog.cvvliy.cn/Article/details/682682.sHtML
tv.blog.cvvliy.cn/Article/details/868620.sHtML
tv.blog.cvvliy.cn/Article/details/264444.sHtML
tv.blog.cvvliy.cn/Article/details/755937.sHtML
tv.blog.cvvliy.cn/Article/details/173759.sHtML
tv.blog.cvvliy.cn/Article/details/006286.sHtML
tv.blog.cvvliy.cn/Article/details/808840.sHtML
tv.blog.cvvliy.cn/Article/details/628684.sHtML
tv.blog.cvvliy.cn/Article/details/959317.sHtML
tv.blog.cvvliy.cn/Article/details/022008.sHtML
tv.blog.cvvliy.cn/Article/details/284082.sHtML
tv.blog.cvvliy.cn/Article/details/066680.sHtML
tv.blog.cvvliy.cn/Article/details/466882.sHtML
tv.blog.cvvliy.cn/Article/details/713593.sHtML
tv.blog.cvvliy.cn/Article/details/755555.sHtML
tv.blog.cvvliy.cn/Article/details/397979.sHtML
tv.blog.cvvliy.cn/Article/details/622466.sHtML
tv.blog.cvvliy.cn/Article/details/480484.sHtML
tv.blog.cvvliy.cn/Article/details/860804.sHtML
tv.blog.cvvliy.cn/Article/details/551919.sHtML
tv.blog.cvvliy.cn/Article/details/731135.sHtML
tv.blog.cvvliy.cn/Article/details/111179.sHtML
tv.blog.cvvliy.cn/Article/details/351197.sHtML
tv.blog.cvvliy.cn/Article/details/402268.sHtML
tv.blog.cvvliy.cn/Article/details/648422.sHtML
tv.blog.cvvliy.cn/Article/details/840268.sHtML
tv.blog.cvvliy.cn/Article/details/468484.sHtML
tv.blog.cvvliy.cn/Article/details/775159.sHtML
tv.blog.cvvliy.cn/Article/details/519971.sHtML
tv.blog.cvvliy.cn/Article/details/624040.sHtML
tv.blog.cvvliy.cn/Article/details/848640.sHtML
tv.blog.cvvliy.cn/Article/details/280480.sHtML
tv.blog.cvvliy.cn/Article/details/591751.sHtML
tv.blog.cvvliy.cn/Article/details/513371.sHtML
tv.blog.cvvliy.cn/Article/details/131559.sHtML
tv.blog.cvvliy.cn/Article/details/937799.sHtML
tv.blog.cvvliy.cn/Article/details/662442.sHtML
tv.blog.cvvliy.cn/Article/details/626420.sHtML
tv.blog.cvvliy.cn/Article/details/440088.sHtML
tv.blog.cvvliy.cn/Article/details/628288.sHtML
tv.blog.cvvliy.cn/Article/details/408200.sHtML
tv.blog.cvvliy.cn/Article/details/240646.sHtML
tv.blog.cvvliy.cn/Article/details/046200.sHtML
tv.blog.cvvliy.cn/Article/details/971551.sHtML
tv.blog.cvvliy.cn/Article/details/266488.sHtML
tv.blog.cvvliy.cn/Article/details/264864.sHtML
tv.blog.cvvliy.cn/Article/details/444682.sHtML
tv.blog.cvvliy.cn/Article/details/482864.sHtML
tv.blog.cvvliy.cn/Article/details/391971.sHtML
tv.blog.cvvliy.cn/Article/details/088824.sHtML
tv.blog.cvvliy.cn/Article/details/206262.sHtML
tv.blog.cvvliy.cn/Article/details/288242.sHtML
tv.blog.cvvliy.cn/Article/details/193779.sHtML
tv.blog.cvvliy.cn/Article/details/731559.sHtML
tv.blog.cvvliy.cn/Article/details/517573.sHtML
tv.blog.cvvliy.cn/Article/details/139117.sHtML
tv.blog.cvvliy.cn/Article/details/913139.sHtML
tv.blog.cvvliy.cn/Article/details/551119.sHtML
tv.blog.cvvliy.cn/Article/details/757571.sHtML
tv.blog.cvvliy.cn/Article/details/268068.sHtML
tv.blog.cvvliy.cn/Article/details/753511.sHtML
tv.blog.cvvliy.cn/Article/details/191119.sHtML
tv.blog.cvvliy.cn/Article/details/400064.sHtML
tv.blog.cvvliy.cn/Article/details/422682.sHtML
tv.blog.cvvliy.cn/Article/details/573733.sHtML
tv.blog.cvvliy.cn/Article/details/537959.sHtML
tv.blog.cvvliy.cn/Article/details/246806.sHtML
tv.blog.cvvliy.cn/Article/details/806240.sHtML
tv.blog.cvvliy.cn/Article/details/068860.sHtML
tv.blog.cvvliy.cn/Article/details/917953.sHtML
tv.blog.cvvliy.cn/Article/details/208482.sHtML
tv.blog.cvvliy.cn/Article/details/422020.sHtML
tv.blog.cvvliy.cn/Article/details/911971.sHtML
tv.blog.cvvliy.cn/Article/details/179351.sHtML
tv.blog.cvvliy.cn/Article/details/200644.sHtML
tv.blog.cvvliy.cn/Article/details/028448.sHtML
tv.blog.cvvliy.cn/Article/details/606808.sHtML
tv.blog.cvvliy.cn/Article/details/804400.sHtML
tv.blog.cvvliy.cn/Article/details/553979.sHtML
tv.blog.cvvliy.cn/Article/details/404464.sHtML
tv.blog.cvvliy.cn/Article/details/428048.sHtML
tv.blog.cvvliy.cn/Article/details/951179.sHtML
tv.blog.cvvliy.cn/Article/details/006462.sHtML
tv.blog.cvvliy.cn/Article/details/975959.sHtML
tv.blog.cvvliy.cn/Article/details/480060.sHtML
tv.blog.cvvliy.cn/Article/details/480644.sHtML
tv.blog.cvvliy.cn/Article/details/937991.sHtML
tv.blog.cvvliy.cn/Article/details/842822.sHtML
tv.blog.cvvliy.cn/Article/details/977991.sHtML
tv.blog.cvvliy.cn/Article/details/171193.sHtML
tv.blog.cvvliy.cn/Article/details/402020.sHtML
tv.blog.cvvliy.cn/Article/details/800844.sHtML
tv.blog.cvvliy.cn/Article/details/155759.sHtML
tv.blog.cvvliy.cn/Article/details/751795.sHtML
tv.blog.cvvliy.cn/Article/details/866426.sHtML
tv.blog.cvvliy.cn/Article/details/860664.sHtML
tv.blog.cvvliy.cn/Article/details/266460.sHtML
tv.blog.cvvliy.cn/Article/details/468046.sHtML
tv.blog.cvvliy.cn/Article/details/008028.sHtML
tv.blog.cvvliy.cn/Article/details/084882.sHtML
tv.blog.cvvliy.cn/Article/details/424406.sHtML
tv.blog.cvvliy.cn/Article/details/995935.sHtML
tv.blog.cvvliy.cn/Article/details/393773.sHtML
tv.blog.cvvliy.cn/Article/details/608426.sHtML
tv.blog.cvvliy.cn/Article/details/517971.sHtML
tv.blog.cvvliy.cn/Article/details/042204.sHtML
tv.blog.cvvliy.cn/Article/details/024866.sHtML
tv.blog.cvvliy.cn/Article/details/044606.sHtML
tv.blog.cvvliy.cn/Article/details/791195.sHtML
tv.blog.cvvliy.cn/Article/details/624024.sHtML
tv.blog.cvvliy.cn/Article/details/644606.sHtML
tv.blog.cvvliy.cn/Article/details/442268.sHtML
tv.blog.cvvliy.cn/Article/details/082200.sHtML
tv.blog.cvvliy.cn/Article/details/040866.sHtML
tv.blog.cvvliy.cn/Article/details/200664.sHtML
tv.blog.cvvliy.cn/Article/details/559733.sHtML
