========================================
 Python 反射与自省机制
========================================

一、什么是反射与自省
----------------------------------------

自省（Introspection）是指程序在运行时检查自身结构和状态的能能力。
反射（Reflection）则更进一步，允许程序在运行时动态地创建对象、调用方法和修改属性。

Python 提供了强大的自省和反射机制，这正是它被称为"可自省的"语言的原因。


二、type() 和 isinstance() — 运行时类型检查
----------------------------------------

# type() —— 获取对象的类型
class Animal:
    pass

class Dog(Animal):
    pass

dog = Dog()
print(type(dog))            # <class '__main__.Dog'>
print(type(dog) is Dog)     # True（精确类型检查）
print(type(dog) is Animal)  # False（type 不考虑继承）

# isinstance() —— 检查类型（考虑继承链）
print(isinstance(dog, Dog))      # True
print(isinstance(dog, Animal))   # True（Dog 继承自 Animal）
print(isinstance(dog, object))   # True（所有类都继承自 object）

# issubclass() —— 检查类的继承关系
print(issubclass(Dog, Animal))   # True
print(issubclass(Animal, object)) # True

# 实用示例：类型分发
def process_data(data):
    """根据不同类型执行不同处理"""
    if isinstance(data, str):
        return f"处理字符串: {data.upper()}"
    elif isinstance(data, (int, float)):
        return f"处理数字: {data * 2}"
    elif isinstance(data, list):
        return f"处理列表: 共 {len(data)} 个元素"
    else:
        return f"未知类型: {type(data).__name__}"


三、dir() 和 vars() — 属性自省
----------------------------------------

class Person:
    """人类——用于展示属性自省的示例类"""

    species = "智人"  # 类属性

    def __init__(self, name: str, age: int):
        self.name = name   # 实例属性
        self.age = age     # 实例属性
        self._secret = "隐藏属性"  # 私有属性约定

    def greet(self) -> str:
        return f"你好，我是{self.name}"

p = Person("李四", 30)

# dir() —— 列出对象的所有属性和方法名
print(dir(p))
# 输出包含了 __init__, greet, name, age, _secret 等

# vars() —— 返回 __dict__（实例的属性字典）
print(vars(p))
# 输出: {'name': '李四', 'age': 30, '_secret': '隐藏属性'}

# 过滤出用户自定义的属性（非特殊方法）
user_attrs = {
    k: v for k, v in vars(p).items()
    if not k.startswith("_")  # 排除私有属性
}
print(user_attrs)  # {'name': '李四', 'age': 30}


四、getattr/setattr/hasattr/delattr — 属性反射
----------------------------------------

# hasattr() —— 检查对象是否有某个属性
print(hasattr(p, "name"))      # True
print(hasattr(p, "address"))   # False

# getattr() —— 获取属性（可提供默认值）
name = getattr(p, "name")           # 李四
address = getattr(p, "address", "未知地址")  # 默认值
method = getattr(p, "greet")        # 获取方法对象
print(method())                      # 你好，我是李四

# setattr() —— 动态设置属性
setattr(p, "age", 31)
setattr(p, "address", "北京市海淀区")
print(p.age)       # 31

# delattr() —— 删除属性
if hasattr(p, "_secret"):
    delattr(p, "_secret")

# 实用示例：动态配置对象
def apply_config(obj, config: dict) -> None:
    """将字典配置动态应用到对象"""
    for key, value in config.items():
        if hasattr(obj, key):
            setattr(obj, key, value)
        else:
            print(f"警告: 对象没有属性 '{key}'，已跳过")


五、inspect 模块 — 深入自省工具箱
----------------------------------------

import inspect
from typing import get_type_hints

class Calculator:
    """计算器类——用于展示 inspect 模块的用法"""

    def __init__(self, precision: int = 2):
        self.precision = precision

    def add(self, a: float, b: float) -> float:
        """将两个数字相加并返回结果"""
        return round(a + b, self.precision)

    def subtract(self, a: float, b: float) -> float:
        """将两个数字相减"""
        return round(a - b, self.precision)

calc = Calculator()

# getmembers() —— 获取对象的所有成员（带类型过滤）
methods = inspect.getmembers(calc, predicate=inspect.ismethod)
print(methods)
# 输出: [('add', ...), ('subtract', ...), ...]

# signature() —— 获取函数签名信息
sig = inspect.signature(calc.add)
print(sig)                # (a: float, b: float) -> float
for name, param in sig.parameters.items():
    print(f"参数: {name}, 类型: {param.annotation}, 默认值: {param.default}")

# getsource() —— 获取函数源代码
source = inspect.getsource(calc.add)
print(source)
# 输出 add 方法的完整源代码

# isfunction/isclass/ismethod —— 类型谓词
print(inspect.isfunction(calc.add))    # False（绑定方法）
print(inspect.ismethod(calc.add))      # True
print(inspect.isclass(Calculator))     # True
print(inspect.isfunction(Calculator.add))  # True（未绑定的函数）

# getfile() —— 获取定义对象的文件路径
print(inspect.getfile(Calculator))
# 输出当前文件的完整路径


六、__name__ 和 __qualname__ — 名称自省
----------------------------------------

class Outer:
    """外层类——展示 __name__ 和 __qualname__ 的区别"""

    class Inner:
        """内层类"""

        def method(self):
            pass

def top_level_func():
    pass

# __name__ —— 仅类名/函数名
print(Outer.__name__)              # Outer
print(Outer.Inner.__name__)        # Inner
print(top_level_func.__name__)     # top_level_func

# __qualname__ —— 限定名（包含所在类的完整路径）
print(Outer.__qualname__)          # Outer
print(Outer.Inner.__qualname__)    # Outer.Inner
print(Outer.Inner.method.__qualname__)  # Outer.Inner.method

# 应用场景：日志记录器自动推断名称
import logging
logger = logging.getLogger(__name__)  # 自动使用模块名


七、函数注解运行时检查
----------------------------------------

from typing import get_type_hints, Annotated

def fetch_data(
    url: str,
    timeout: int = 30,
    retry: bool = True
) -> dict[str, any]:
    """从指定 URL 获取数据"""
    return {"status": "ok"}

# 获取所有类型注解
hints = get_type_hints(fetch_data)
print(hints)
# 输出: {'url': str, 'timeout': int, 'retry': bool, 'return': dict[str, any]}

# __annotations__ 原始字典
print(fetch_data.__annotations__)
# 输出: {'url': <class 'str'>, 'timeout': <class 'int'>, ...}

# 实用：基于注解的参数验证
def validate_call(func):
    """装饰器：运行时验证函数参数类型"""
    from functools import wraps
    hints = get_type_hints(func)

    @wraps(func)
    def wrapper(*args, **kwargs):
        # 简单类型检查（实际应更完善）
        for name, value in kwargs.items():
            if name in hints and not isinstance(value, hints[name]):
                raise TypeError(f"参数 {name} 应为 {hints[name]}, 实际为 {type(value)}")
        return func(*args, **kwargs)
    return wrapper


八、__dict__ 与 __slots__ — 属性存储
----------------------------------------

class WithDict:
    """使用 __dict__ 存储属性（默认方式）"""
    def __init__(self):
        self.x = 1
        self.y = 2

class WithSlots:
    """使用 __slots__ 优化内存"""
    __slots__ = ("x", "y")  # 固定属性列表

    def __init__(self):
        self.x = 1
        self.y = 2

d = WithDict()
s = WithSlots()

print(type(d.__dict__))   # <class 'dict'> —— 动态字典
print(d.__dict__)         # {'x': 1, 'y': 2}

# WithSlots 没有 __dict__，属性存储在固定大小的数组中
# print(s.__dict__)  # AttributeError: 'WithSlots' 没有 __dict__

# __slots__ 优势：内存更小，访问更快
# __slots__ 劣势：不能动态添加属性
# setattr(s, "z", 3)  # AttributeError!

# 也可以混用 __slots__ 和 __dict__
class MixedSlots:
    __slots__ = ("fixed",)  # 快速访问固定属性
    # 未在 __slots__ 中的属性仍然走 __dict__

    def __init__(self):
        self.fixed = "固定属性"
        self.dynamic = "动态属性"  # 存在 __dict__ 中

frq.0wp4aoe.cn/79291.Doc
frq.0wp4aoe.cn/49137.Doc
frq.0wp4aoe.cn/10548.Doc
frq.0wp4aoe.cn/30156.Doc
frq.0wp4aoe.cn/82020.Doc
frq.0wp4aoe.cn/22265.Doc
frq.0wp4aoe.cn/31133.Doc
frq.0wp4aoe.cn/65372.Doc
frq.0wp4aoe.cn/36688.Doc
frq.0wp4aoe.cn/97036.Doc
fem.0wp4aoe.cn/17200.Doc
fem.0wp4aoe.cn/47581.Doc
fem.0wp4aoe.cn/05680.Doc
fem.0wp4aoe.cn/59340.Doc
fem.0wp4aoe.cn/19553.Doc
fem.0wp4aoe.cn/05140.Doc
fem.0wp4aoe.cn/56480.Doc
fem.0wp4aoe.cn/09136.Doc
fem.0wp4aoe.cn/11678.Doc
fem.0wp4aoe.cn/86000.Doc
fen.0wp4aoe.cn/01310.Doc
fen.0wp4aoe.cn/98643.Doc
fen.0wp4aoe.cn/12228.Doc
fen.0wp4aoe.cn/96560.Doc
fen.0wp4aoe.cn/41105.Doc
fen.0wp4aoe.cn/07073.Doc
fen.0wp4aoe.cn/15359.Doc
fen.0wp4aoe.cn/29272.Doc
fen.0wp4aoe.cn/02211.Doc
fen.0wp4aoe.cn/04085.Doc
feb.0wp4aoe.cn/66376.Doc
feb.0wp4aoe.cn/10579.Doc
feb.0wp4aoe.cn/72015.Doc
feb.0wp4aoe.cn/79185.Doc
feb.0wp4aoe.cn/18899.Doc
feb.0wp4aoe.cn/02797.Doc
feb.0wp4aoe.cn/46496.Doc
feb.0wp4aoe.cn/38847.Doc
feb.0wp4aoe.cn/65063.Doc
feb.0wp4aoe.cn/97989.Doc
fev.0wp4aoe.cn/89218.Doc
fev.0wp4aoe.cn/34811.Doc
fev.0wp4aoe.cn/08288.Doc
fev.0wp4aoe.cn/78536.Doc
fev.0wp4aoe.cn/47900.Doc
fev.0wp4aoe.cn/38003.Doc
fev.0wp4aoe.cn/42757.Doc
fev.0wp4aoe.cn/77932.Doc
fev.0wp4aoe.cn/57909.Doc
fev.0wp4aoe.cn/50802.Doc
fec.0wp4aoe.cn/52567.Doc
fec.0wp4aoe.cn/98257.Doc
fec.0wp4aoe.cn/36255.Doc
fec.0wp4aoe.cn/91186.Doc
fec.0wp4aoe.cn/77228.Doc
fec.0wp4aoe.cn/01827.Doc
fec.0wp4aoe.cn/24505.Doc
fec.0wp4aoe.cn/30049.Doc
fec.0wp4aoe.cn/73567.Doc
fec.0wp4aoe.cn/63303.Doc
fex.0wp4aoe.cn/23800.Doc
fex.0wp4aoe.cn/52206.Doc
fex.0wp4aoe.cn/35818.Doc
fex.0wp4aoe.cn/62800.Doc
fex.0wp4aoe.cn/52443.Doc
fex.0wp4aoe.cn/57147.Doc
fex.0wp4aoe.cn/48109.Doc
fex.0wp4aoe.cn/79477.Doc
fex.0wp4aoe.cn/89202.Doc
fex.0wp4aoe.cn/01290.Doc
fez.0wp4aoe.cn/88636.Doc
fez.0wp4aoe.cn/61471.Doc
fez.0wp4aoe.cn/48734.Doc
fez.0wp4aoe.cn/54081.Doc
fez.0wp4aoe.cn/53494.Doc
fez.0wp4aoe.cn/09624.Doc
fez.0wp4aoe.cn/18468.Doc
fez.0wp4aoe.cn/99646.Doc
fez.0wp4aoe.cn/91163.Doc
fez.0wp4aoe.cn/36696.Doc
fel.0wp4aoe.cn/74699.Doc
fel.0wp4aoe.cn/30709.Doc
fel.0wp4aoe.cn/95027.Doc
fel.0wp4aoe.cn/52815.Doc
fel.0wp4aoe.cn/21381.Doc
fel.0wp4aoe.cn/32278.Doc
fel.0wp4aoe.cn/76900.Doc
fel.0wp4aoe.cn/50699.Doc
fel.0wp4aoe.cn/84711.Doc
fel.0wp4aoe.cn/72797.Doc
fek.0wp4aoe.cn/82398.Doc
fek.0wp4aoe.cn/06906.Doc
fek.0wp4aoe.cn/20843.Doc
fek.0wp4aoe.cn/93912.Doc
fek.0wp4aoe.cn/44416.Doc
fek.0wp4aoe.cn/35028.Doc
fek.0wp4aoe.cn/59636.Doc
fek.0wp4aoe.cn/57176.Doc
fek.0wp4aoe.cn/49135.Doc
fek.0wp4aoe.cn/65356.Doc
fej.0wp4aoe.cn/36442.Doc
fej.0wp4aoe.cn/98258.Doc
fej.0wp4aoe.cn/06798.Doc
fej.0wp4aoe.cn/78025.Doc
fej.0wp4aoe.cn/27226.Doc
fej.0wp4aoe.cn/15390.Doc
fej.0wp4aoe.cn/08890.Doc
fej.0wp4aoe.cn/14492.Doc
fej.0wp4aoe.cn/16727.Doc
fej.0wp4aoe.cn/54535.Doc
feh.0wp4aoe.cn/21475.Doc
feh.0wp4aoe.cn/14374.Doc
feh.0wp4aoe.cn/88659.Doc
feh.0wp4aoe.cn/10502.Doc
feh.0wp4aoe.cn/00027.Doc
feh.0wp4aoe.cn/55990.Doc
feh.0wp4aoe.cn/96786.Doc
feh.0wp4aoe.cn/94811.Doc
feh.0wp4aoe.cn/50904.Doc
feh.0wp4aoe.cn/41648.Doc
feg.0wp4aoe.cn/69829.Doc
feg.0wp4aoe.cn/13541.Doc
feg.0wp4aoe.cn/62306.Doc
feg.0wp4aoe.cn/30132.Doc
feg.0wp4aoe.cn/97513.Doc
feg.0wp4aoe.cn/14940.Doc
feg.0wp4aoe.cn/08158.Doc
feg.0wp4aoe.cn/37381.Doc
feg.0wp4aoe.cn/06150.Doc
feg.0wp4aoe.cn/99580.Doc
fef.0wp4aoe.cn/45103.Doc
fef.0wp4aoe.cn/50111.Doc
fef.0wp4aoe.cn/04982.Doc
fef.0wp4aoe.cn/38477.Doc
fef.0wp4aoe.cn/43542.Doc
fef.0wp4aoe.cn/68958.Doc
fef.0wp4aoe.cn/00570.Doc
fef.0wp4aoe.cn/04747.Doc
fef.0wp4aoe.cn/54524.Doc
fef.0wp4aoe.cn/51103.Doc
fed.0wp4aoe.cn/85319.Doc
fed.0wp4aoe.cn/46944.Doc
fed.0wp4aoe.cn/29139.Doc
fed.0wp4aoe.cn/31630.Doc
fed.0wp4aoe.cn/33039.Doc
fed.0wp4aoe.cn/46521.Doc
fed.0wp4aoe.cn/06171.Doc
fed.0wp4aoe.cn/27476.Doc
fed.0wp4aoe.cn/09809.Doc
fed.0wp4aoe.cn/69288.Doc
fes.0wp4aoe.cn/89905.Doc
fes.0wp4aoe.cn/16279.Doc
fes.0wp4aoe.cn/55782.Doc
fes.0wp4aoe.cn/98049.Doc
fes.0wp4aoe.cn/30701.Doc
fes.0wp4aoe.cn/41938.Doc
fes.0wp4aoe.cn/66104.Doc
fes.0wp4aoe.cn/32056.Doc
fes.0wp4aoe.cn/93768.Doc
fes.0wp4aoe.cn/10043.Doc
fea.0wp4aoe.cn/92463.Doc
fea.0wp4aoe.cn/34944.Doc
fea.0wp4aoe.cn/22075.Doc
fea.0wp4aoe.cn/11164.Doc
fea.0wp4aoe.cn/51985.Doc
fea.0wp4aoe.cn/67631.Doc
fea.0wp4aoe.cn/97140.Doc
fea.0wp4aoe.cn/17469.Doc
fea.0wp4aoe.cn/08829.Doc
fea.0wp4aoe.cn/44748.Doc
fep.0wp4aoe.cn/88924.Doc
fep.0wp4aoe.cn/32304.Doc
fep.0wp4aoe.cn/06499.Doc
fep.0wp4aoe.cn/42377.Doc
fep.0wp4aoe.cn/66923.Doc
fep.0wp4aoe.cn/73800.Doc
fep.0wp4aoe.cn/24429.Doc
fep.0wp4aoe.cn/44009.Doc
fep.0wp4aoe.cn/55570.Doc
fep.0wp4aoe.cn/80761.Doc
feo.0wp4aoe.cn/11275.Doc
feo.0wp4aoe.cn/67944.Doc
feo.0wp4aoe.cn/19725.Doc
feo.0wp4aoe.cn/00561.Doc
feo.0wp4aoe.cn/57928.Doc
feo.0wp4aoe.cn/57449.Doc
feo.0wp4aoe.cn/85020.Doc
feo.0wp4aoe.cn/94924.Doc
feo.0wp4aoe.cn/16413.Doc
feo.0wp4aoe.cn/06157.Doc
fei.0wp4aoe.cn/08782.Doc
fei.0wp4aoe.cn/75620.Doc
fei.0wp4aoe.cn/37928.Doc
fei.0wp4aoe.cn/58164.Doc
fei.0wp4aoe.cn/05008.Doc
fei.0wp4aoe.cn/27229.Doc
fei.0wp4aoe.cn/17523.Doc
fei.0wp4aoe.cn/05839.Doc
fei.0wp4aoe.cn/42743.Doc
fei.0wp4aoe.cn/31214.Doc
feu.0wp4aoe.cn/00452.Doc
feu.0wp4aoe.cn/04281.Doc
feu.0wp4aoe.cn/98294.Doc
feu.0wp4aoe.cn/76886.Doc
feu.0wp4aoe.cn/22958.Doc
feu.0wp4aoe.cn/43222.Doc
feu.0wp4aoe.cn/74688.Doc
feu.0wp4aoe.cn/10124.Doc
feu.0wp4aoe.cn/42244.Doc
feu.0wp4aoe.cn/26480.Doc
fey.0wp4aoe.cn/59990.Doc
fey.0wp4aoe.cn/42140.Doc
fey.0wp4aoe.cn/54837.Doc
fey.0wp4aoe.cn/82503.Doc
fey.0wp4aoe.cn/76977.Doc
fey.0wp4aoe.cn/93068.Doc
fey.0wp4aoe.cn/01585.Doc
fey.0wp4aoe.cn/32530.Doc
fey.0wp4aoe.cn/86954.Doc
fey.0wp4aoe.cn/28677.Doc
fet.0wp4aoe.cn/22085.Doc
fet.0wp4aoe.cn/48010.Doc
fet.0wp4aoe.cn/66849.Doc
fet.0wp4aoe.cn/61224.Doc
fet.0wp4aoe.cn/04853.Doc
fet.0wp4aoe.cn/15117.Doc
fet.0wp4aoe.cn/53771.Doc
fet.0wp4aoe.cn/15519.Doc
fet.0wp4aoe.cn/71719.Doc
fet.0wp4aoe.cn/42802.Doc
fer.0wp4aoe.cn/75719.Doc
fer.0wp4aoe.cn/57733.Doc
fer.0wp4aoe.cn/31955.Doc
fer.0wp4aoe.cn/79195.Doc
fer.0wp4aoe.cn/73939.Doc
fer.0wp4aoe.cn/91599.Doc
fer.0wp4aoe.cn/15335.Doc
fer.0wp4aoe.cn/35799.Doc
fer.0wp4aoe.cn/19151.Doc
fer.0wp4aoe.cn/28682.Doc
fee.0wp4aoe.cn/97933.Doc
fee.0wp4aoe.cn/93159.Doc
fee.0wp4aoe.cn/80868.Doc
fee.0wp4aoe.cn/51935.Doc
fee.0wp4aoe.cn/37319.Doc
fee.0wp4aoe.cn/79991.Doc
fee.0wp4aoe.cn/13739.Doc
fee.0wp4aoe.cn/57555.Doc
fee.0wp4aoe.cn/39115.Doc
fee.0wp4aoe.cn/77715.Doc
few.0wp4aoe.cn/95375.Doc
few.0wp4aoe.cn/77799.Doc
few.0wp4aoe.cn/77157.Doc
few.0wp4aoe.cn/22828.Doc
few.0wp4aoe.cn/71993.Doc
few.0wp4aoe.cn/37131.Doc
few.0wp4aoe.cn/17513.Doc
few.0wp4aoe.cn/95973.Doc
few.0wp4aoe.cn/31799.Doc
few.0wp4aoe.cn/39379.Doc
feq.0wp4aoe.cn/73753.Doc
feq.0wp4aoe.cn/55715.Doc
feq.0wp4aoe.cn/17535.Doc
feq.0wp4aoe.cn/79133.Doc
feq.0wp4aoe.cn/15977.Doc
feq.0wp4aoe.cn/39531.Doc
feq.0wp4aoe.cn/97999.Doc
feq.0wp4aoe.cn/57937.Doc
feq.0wp4aoe.cn/71359.Doc
feq.0wp4aoe.cn/33951.Doc
fwm.0wp4aoe.cn/11371.Doc
fwm.0wp4aoe.cn/35119.Doc
fwm.0wp4aoe.cn/95537.Doc
fwm.0wp4aoe.cn/24660.Doc
fwm.0wp4aoe.cn/99999.Doc
fwm.0wp4aoe.cn/79551.Doc
fwm.0wp4aoe.cn/19357.Doc
fwm.0wp4aoe.cn/93531.Doc
fwm.0wp4aoe.cn/73711.Doc
fwm.0wp4aoe.cn/53575.Doc
fwn.0wp4aoe.cn/55575.Doc
fwn.0wp4aoe.cn/93915.Doc
fwn.0wp4aoe.cn/57771.Doc
fwn.0wp4aoe.cn/9.Doc
fwn.0wp4aoe.cn/71791.Doc
fwn.0wp4aoe.cn/11571.Doc
fwn.0wp4aoe.cn/57715.Doc
fwn.0wp4aoe.cn/31737.Doc
fwn.0wp4aoe.cn/91997.Doc
fwn.0wp4aoe.cn/99117.Doc
fwb.0wp4aoe.cn/97359.Doc
fwb.0wp4aoe.cn/86248.Doc
fwb.0wp4aoe.cn/35995.Doc
fwb.0wp4aoe.cn/93115.Doc
fwb.0wp4aoe.cn/93539.Doc
fwb.0wp4aoe.cn/99311.Doc
fwb.0wp4aoe.cn/37993.Doc
fwb.0wp4aoe.cn/55771.Doc
fwb.0wp4aoe.cn/75713.Doc
fwb.0wp4aoe.cn/26244.Doc
fwv.0wp4aoe.cn/39399.Doc
fwv.0wp4aoe.cn/11135.Doc
fwv.0wp4aoe.cn/31179.Doc
fwv.0wp4aoe.cn/39797.Doc
fwv.0wp4aoe.cn/11959.Doc
fwv.0wp4aoe.cn/31771.Doc
fwv.0wp4aoe.cn/51395.Doc
fwv.0wp4aoe.cn/59911.Doc
fwv.0wp4aoe.cn/37357.Doc
fwv.0wp4aoe.cn/55393.Doc
fwc.0wp4aoe.cn/53799.Doc
fwc.0wp4aoe.cn/59953.Doc
fwc.0wp4aoe.cn/51339.Doc
fwc.0wp4aoe.cn/97513.Doc
fwc.0wp4aoe.cn/77973.Doc
fwc.0wp4aoe.cn/35573.Doc
fwc.0wp4aoe.cn/71119.Doc
fwc.0wp4aoe.cn/31993.Doc
fwc.0wp4aoe.cn/79575.Doc
fwc.0wp4aoe.cn/95391.Doc
fwx.0wp4aoe.cn/11317.Doc
fwx.0wp4aoe.cn/33535.Doc
fwx.0wp4aoe.cn/19139.Doc
fwx.0wp4aoe.cn/73993.Doc
fwx.0wp4aoe.cn/91957.Doc
fwx.0wp4aoe.cn/11153.Doc
fwx.0wp4aoe.cn/35151.Doc
fwx.0wp4aoe.cn/73593.Doc
fwx.0wp4aoe.cn/91717.Doc
fwx.0wp4aoe.cn/55371.Doc
fwz.0wp4aoe.cn/19175.Doc
fwz.0wp4aoe.cn/11533.Doc
fwz.0wp4aoe.cn/17971.Doc
fwz.0wp4aoe.cn/77991.Doc
fwz.0wp4aoe.cn/91191.Doc
fwz.0wp4aoe.cn/95551.Doc
fwz.0wp4aoe.cn/77999.Doc
fwz.0wp4aoe.cn/17377.Doc
fwz.0wp4aoe.cn/35737.Doc
fwz.0wp4aoe.cn/79335.Doc
fwl.0wp4aoe.cn/31955.Doc
fwl.0wp4aoe.cn/73719.Doc
fwl.0wp4aoe.cn/33197.Doc
fwl.0wp4aoe.cn/19179.Doc
fwl.0wp4aoe.cn/93519.Doc
fwl.0wp4aoe.cn/13333.Doc
fwl.0wp4aoe.cn/71913.Doc
fwl.0wp4aoe.cn/31553.Doc
fwl.0wp4aoe.cn/42640.Doc
fwl.0wp4aoe.cn/91911.Doc
fwk.0wp4aoe.cn/55579.Doc
fwk.0wp4aoe.cn/59391.Doc
fwk.0wp4aoe.cn/97911.Doc
fwk.0wp4aoe.cn/33915.Doc
fwk.0wp4aoe.cn/88286.Doc
fwk.0wp4aoe.cn/77311.Doc
fwk.0wp4aoe.cn/57771.Doc
fwk.0wp4aoe.cn/71777.Doc
fwk.0wp4aoe.cn/75753.Doc
fwk.0wp4aoe.cn/57777.Doc
fwj.0wp4aoe.cn/44082.Doc
fwj.0wp4aoe.cn/99935.Doc
fwj.0wp4aoe.cn/95931.Doc
fwj.0wp4aoe.cn/79379.Doc
fwj.0wp4aoe.cn/22046.Doc
fwj.0wp4aoe.cn/93351.Doc
fwj.0wp4aoe.cn/33313.Doc
fwj.0wp4aoe.cn/75513.Doc
fwj.0wp4aoe.cn/51971.Doc
fwj.0wp4aoe.cn/11715.Doc
fwh.0wp4aoe.cn/33517.Doc
fwh.0wp4aoe.cn/31375.Doc
fwh.0wp4aoe.cn/15153.Doc
fwh.0wp4aoe.cn/33773.Doc
fwh.0wp4aoe.cn/31937.Doc
fwh.0wp4aoe.cn/93311.Doc
fwh.0wp4aoe.cn/17193.Doc
fwh.0wp4aoe.cn/91179.Doc
fwh.0wp4aoe.cn/17939.Doc
fwh.0wp4aoe.cn/55551.Doc
fwg.0wp4aoe.cn/51795.Doc
fwg.0wp4aoe.cn/59515.Doc
fwg.0wp4aoe.cn/75575.Doc
fwg.0wp4aoe.cn/57371.Doc
fwg.0wp4aoe.cn/37395.Doc
fwg.0wp4aoe.cn/99735.Doc
fwg.0wp4aoe.cn/75599.Doc
fwg.0wp4aoe.cn/11997.Doc
fwg.0wp4aoe.cn/93997.Doc
fwg.0wp4aoe.cn/53353.Doc
fwf.0wp4aoe.cn/57593.Doc
fwf.0wp4aoe.cn/75133.Doc
fwf.0wp4aoe.cn/59719.Doc
fwf.0wp4aoe.cn/08686.Doc
fwf.0wp4aoe.cn/73335.Doc
fwf.0wp4aoe.cn/31539.Doc
fwf.0wp4aoe.cn/75935.Doc
fwf.0wp4aoe.cn/77997.Doc
fwf.0wp4aoe.cn/66668.Doc
fwf.0wp4aoe.cn/77139.Doc
fwd.0wp4aoe.cn/59755.Doc
fwd.0wp4aoe.cn/60846.Doc
fwd.0wp4aoe.cn/53591.Doc
fwd.0wp4aoe.cn/51175.Doc
fwd.0wp4aoe.cn/19715.Doc
fwd.0wp4aoe.cn/88840.Doc
fwd.0wp4aoe.cn/31917.Doc
fwd.0wp4aoe.cn/86026.Doc
fwd.0wp4aoe.cn/71171.Doc
fwd.0wp4aoe.cn/93937.Doc
fws.0wp4aoe.cn/53957.Doc
fws.0wp4aoe.cn/55759.Doc
fws.0wp4aoe.cn/3.Doc
fws.0wp4aoe.cn/13913.Doc
fws.0wp4aoe.cn/57959.Doc
fws.0wp4aoe.cn/59115.Doc
fws.0wp4aoe.cn/99379.Doc
fws.0wp4aoe.cn/08040.Doc
fws.0wp4aoe.cn/71393.Doc
fws.0wp4aoe.cn/79953.Doc
fwa.0wp4aoe.cn/73953.Doc
fwa.0wp4aoe.cn/35335.Doc
fwa.0wp4aoe.cn/11975.Doc
fwa.0wp4aoe.cn/37999.Doc
fwa.0wp4aoe.cn/91515.Doc
fwa.0wp4aoe.cn/55117.Doc
fwa.0wp4aoe.cn/57953.Doc
fwa.0wp4aoe.cn/06066.Doc
fwa.0wp4aoe.cn/75371.Doc
fwa.0wp4aoe.cn/77719.Doc
fwp.0wp4aoe.cn/93359.Doc
fwp.0wp4aoe.cn/71131.Doc
fwp.0wp4aoe.cn/15735.Doc
fwp.0wp4aoe.cn/15931.Doc
fwp.0wp4aoe.cn/59915.Doc
fwp.0wp4aoe.cn/99351.Doc
fwp.0wp4aoe.cn/57757.Doc
fwp.0wp4aoe.cn/77757.Doc
fwp.0wp4aoe.cn/77935.Doc
fwp.0wp4aoe.cn/93931.Doc
fwo.0wp4aoe.cn/93375.Doc
fwo.0wp4aoe.cn/11195.Doc
fwo.0wp4aoe.cn/79377.Doc
fwo.0wp4aoe.cn/71791.Doc
fwo.0wp4aoe.cn/39999.Doc
fwo.0wp4aoe.cn/39997.Doc
fwo.0wp4aoe.cn/15591.Doc
fwo.0wp4aoe.cn/57375.Doc
fwo.0wp4aoe.cn/79191.Doc
fwo.0wp4aoe.cn/15399.Doc
fwi.0wp4aoe.cn/55591.Doc
fwi.0wp4aoe.cn/79373.Doc
fwi.0wp4aoe.cn/15553.Doc
fwi.0wp4aoe.cn/59339.Doc
fwi.0wp4aoe.cn/35135.Doc
fwi.0wp4aoe.cn/11777.Doc
fwi.0wp4aoe.cn/95975.Doc
fwi.0wp4aoe.cn/95555.Doc
fwi.0wp4aoe.cn/59371.Doc
fwi.0wp4aoe.cn/39777.Doc
fwu.0wp4aoe.cn/53771.Doc
fwu.0wp4aoe.cn/59117.Doc
fwu.0wp4aoe.cn/93537.Doc
fwu.0wp4aoe.cn/28400.Doc
fwu.0wp4aoe.cn/73315.Doc
fwu.0wp4aoe.cn/97159.Doc
fwu.0wp4aoe.cn/11739.Doc
fwu.0wp4aoe.cn/53971.Doc
fwu.0wp4aoe.cn/59357.Doc
fwu.0wp4aoe.cn/93155.Doc
fwy.0wp4aoe.cn/37713.Doc
fwy.0wp4aoe.cn/35719.Doc
fwy.0wp4aoe.cn/75153.Doc
fwy.0wp4aoe.cn/15153.Doc
fwy.0wp4aoe.cn/71399.Doc
fwy.0wp4aoe.cn/59199.Doc
fwy.0wp4aoe.cn/15759.Doc
fwy.0wp4aoe.cn/33991.Doc
fwy.0wp4aoe.cn/77573.Doc
fwy.0wp4aoe.cn/55595.Doc
fwt.0wp4aoe.cn/95153.Doc
fwt.0wp4aoe.cn/39191.Doc
fwt.0wp4aoe.cn/95375.Doc
fwt.0wp4aoe.cn/17139.Doc
fwt.0wp4aoe.cn/06020.Doc
fwt.0wp4aoe.cn/13133.Doc
fwt.0wp4aoe.cn/33331.Doc
fwt.0wp4aoe.cn/39597.Doc
fwt.0wp4aoe.cn/51795.Doc
fwt.0wp4aoe.cn/75391.Doc
