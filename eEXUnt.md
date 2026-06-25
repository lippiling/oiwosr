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

eze.wandk5.cn/80880.Doc
eze.wandk5.cn/02486.Doc
eze.wandk5.cn/02006.Doc
eze.wandk5.cn/84026.Doc
eze.wandk5.cn/06280.Doc
eze.wandk5.cn/42101.Doc
eze.wandk5.cn/80882.Doc
eze.wandk5.cn/86426.Doc
eze.wandk5.cn/48024.Doc
eze.wandk5.cn/22224.Doc
ezw.wandk5.cn/66028.Doc
ezw.wandk5.cn/82868.Doc
ezw.wandk5.cn/08600.Doc
ezw.wandk5.cn/08282.Doc
ezw.wandk5.cn/24202.Doc
ezw.wandk5.cn/26624.Doc
ezw.wandk5.cn/02246.Doc
ezw.wandk5.cn/04020.Doc
ezw.wandk5.cn/62226.Doc
ezw.wandk5.cn/40062.Doc
ezq.wandk5.cn/78262.Doc
ezq.wandk5.cn/06862.Doc
ezq.wandk5.cn/04448.Doc
ezq.wandk5.cn/88442.Doc
ezq.wandk5.cn/60828.Doc
ezq.wandk5.cn/42466.Doc
ezq.wandk5.cn/28848.Doc
ezq.wandk5.cn/64220.Doc
ezq.wandk5.cn/00088.Doc
ezq.wandk5.cn/40848.Doc
elm.wandk5.cn/66022.Doc
elm.wandk5.cn/86220.Doc
elm.wandk5.cn/82240.Doc
elm.wandk5.cn/66662.Doc
elm.wandk5.cn/26022.Doc
elm.wandk5.cn/40006.Doc
elm.wandk5.cn/64420.Doc
elm.wandk5.cn/82600.Doc
elm.wandk5.cn/48286.Doc
elm.wandk5.cn/84086.Doc
eln.wandk5.cn/80820.Doc
eln.wandk5.cn/46442.Doc
eln.wandk5.cn/60068.Doc
eln.wandk5.cn/20400.Doc
eln.wandk5.cn/80046.Doc
eln.wandk5.cn/24682.Doc
eln.wandk5.cn/24460.Doc
eln.wandk5.cn/44406.Doc
eln.wandk5.cn/20268.Doc
eln.wandk5.cn/00688.Doc
elb.wandk5.cn/88448.Doc
elb.wandk5.cn/46220.Doc
elb.wandk5.cn/08026.Doc
elb.wandk5.cn/82848.Doc
elb.wandk5.cn/60084.Doc
elb.wandk5.cn/22260.Doc
elb.wandk5.cn/68802.Doc
elb.wandk5.cn/04286.Doc
elb.wandk5.cn/88280.Doc
elb.wandk5.cn/42040.Doc
elv.wandk5.cn/28062.Doc
elv.wandk5.cn/64242.Doc
elv.wandk5.cn/66246.Doc
elv.wandk5.cn/28640.Doc
elv.wandk5.cn/24266.Doc
elv.wandk5.cn/68082.Doc
elv.wandk5.cn/60026.Doc
elv.wandk5.cn/60266.Doc
elv.wandk5.cn/88448.Doc
elv.wandk5.cn/60806.Doc
elc.wandk5.cn/08804.Doc
elc.wandk5.cn/02062.Doc
elc.wandk5.cn/84668.Doc
elc.wandk5.cn/40808.Doc
elc.wandk5.cn/48668.Doc
elc.wandk5.cn/22280.Doc
elc.wandk5.cn/06066.Doc
elc.wandk5.cn/26086.Doc
elc.wandk5.cn/40842.Doc
elc.wandk5.cn/20286.Doc
elx.wandk5.cn/40048.Doc
elx.wandk5.cn/02066.Doc
elx.wandk5.cn/02868.Doc
elx.wandk5.cn/88448.Doc
elx.wandk5.cn/62402.Doc
elx.wandk5.cn/04202.Doc
elx.wandk5.cn/80088.Doc
elx.wandk5.cn/63512.Doc
elx.wandk5.cn/08880.Doc
elx.wandk5.cn/62806.Doc
elz.wandk5.cn/06444.Doc
elz.wandk5.cn/42040.Doc
elz.wandk5.cn/40028.Doc
elz.wandk5.cn/24220.Doc
elz.wandk5.cn/68764.Doc
elz.wandk5.cn/08026.Doc
elz.wandk5.cn/06888.Doc
elz.wandk5.cn/60026.Doc
elz.wandk5.cn/64080.Doc
elz.wandk5.cn/20202.Doc
ell.wandk5.cn/82422.Doc
ell.wandk5.cn/00824.Doc
ell.wandk5.cn/02686.Doc
ell.wandk5.cn/02888.Doc
ell.wandk5.cn/04402.Doc
ell.wandk5.cn/06808.Doc
ell.wandk5.cn/86642.Doc
ell.wandk5.cn/26800.Doc
ell.wandk5.cn/42642.Doc
ell.wandk5.cn/48846.Doc
elk.wandk5.cn/88260.Doc
elk.wandk5.cn/68222.Doc
elk.wandk5.cn/02464.Doc
elk.wandk5.cn/02026.Doc
elk.wandk5.cn/84062.Doc
elk.wandk5.cn/26266.Doc
elk.wandk5.cn/60084.Doc
elk.wandk5.cn/60800.Doc
elk.wandk5.cn/68862.Doc
elk.wandk5.cn/84248.Doc
elj.wandk5.cn/06242.Doc
elj.wandk5.cn/40268.Doc
elj.wandk5.cn/22228.Doc
elj.wandk5.cn/88220.Doc
elj.wandk5.cn/60888.Doc
elj.wandk5.cn/48466.Doc
elj.wandk5.cn/64006.Doc
elj.wandk5.cn/08826.Doc
elj.wandk5.cn/42048.Doc
elj.wandk5.cn/42862.Doc
elh.wandk5.cn/88448.Doc
elh.wandk5.cn/06642.Doc
elh.wandk5.cn/66884.Doc
elh.wandk5.cn/28020.Doc
elh.wandk5.cn/22600.Doc
elh.wandk5.cn/40682.Doc
elh.wandk5.cn/22868.Doc
elh.wandk5.cn/46066.Doc
elh.wandk5.cn/64864.Doc
elh.wandk5.cn/84202.Doc
elg.wandk5.cn/00000.Doc
elg.wandk5.cn/42242.Doc
elg.wandk5.cn/02444.Doc
elg.wandk5.cn/88048.Doc
elg.wandk5.cn/66444.Doc
elg.wandk5.cn/40486.Doc
elg.wandk5.cn/68646.Doc
elg.wandk5.cn/28226.Doc
elg.wandk5.cn/88268.Doc
elg.wandk5.cn/06060.Doc
elf.wandk5.cn/26046.Doc
elf.wandk5.cn/62202.Doc
elf.wandk5.cn/84408.Doc
elf.wandk5.cn/24006.Doc
elf.wandk5.cn/24228.Doc
elf.wandk5.cn/48668.Doc
elf.wandk5.cn/88880.Doc
elf.wandk5.cn/22880.Doc
elf.wandk5.cn/00648.Doc
elf.wandk5.cn/08882.Doc
eld.wandk5.cn/40042.Doc
eld.wandk5.cn/06244.Doc
eld.wandk5.cn/26226.Doc
eld.wandk5.cn/04488.Doc
eld.wandk5.cn/82460.Doc
eld.wandk5.cn/64666.Doc
eld.wandk5.cn/26260.Doc
eld.wandk5.cn/82824.Doc
eld.wandk5.cn/31390.Doc
eld.wandk5.cn/64086.Doc
els.wandk5.cn/62480.Doc
els.wandk5.cn/24842.Doc
els.wandk5.cn/46042.Doc
els.wandk5.cn/42484.Doc
els.wandk5.cn/48620.Doc
els.wandk5.cn/64242.Doc
els.wandk5.cn/37271.Doc
els.wandk5.cn/22442.Doc
els.wandk5.cn/66420.Doc
els.wandk5.cn/00880.Doc
ela.wandk5.cn/20266.Doc
ela.wandk5.cn/28246.Doc
ela.wandk5.cn/88868.Doc
ela.wandk5.cn/40886.Doc
ela.wandk5.cn/02846.Doc
ela.wandk5.cn/22068.Doc
ela.wandk5.cn/28426.Doc
ela.wandk5.cn/62286.Doc
ela.wandk5.cn/60622.Doc
ela.wandk5.cn/32659.Doc
elp.wandk5.cn/60642.Doc
elp.wandk5.cn/06006.Doc
elp.wandk5.cn/28402.Doc
elp.wandk5.cn/60202.Doc
elp.wandk5.cn/42004.Doc
elp.wandk5.cn/00480.Doc
elp.wandk5.cn/00200.Doc
elp.wandk5.cn/26006.Doc
elp.wandk5.cn/86828.Doc
elp.wandk5.cn/62228.Doc
elo.wandk5.cn/06426.Doc
elo.wandk5.cn/84226.Doc
elo.wandk5.cn/68660.Doc
elo.wandk5.cn/02268.Doc
elo.wandk5.cn/68842.Doc
elo.wandk5.cn/80842.Doc
elo.wandk5.cn/66440.Doc
elo.wandk5.cn/60028.Doc
elo.wandk5.cn/26240.Doc
elo.wandk5.cn/08802.Doc
eli.wandk5.cn/26622.Doc
eli.wandk5.cn/00628.Doc
eli.wandk5.cn/60868.Doc
eli.wandk5.cn/82646.Doc
eli.wandk5.cn/26442.Doc
eli.wandk5.cn/11062.Doc
eli.wandk5.cn/88644.Doc
eli.wandk5.cn/62602.Doc
eli.wandk5.cn/66224.Doc
eli.wandk5.cn/00864.Doc
elu.wandk5.cn/88684.Doc
elu.wandk5.cn/46202.Doc
elu.wandk5.cn/60046.Doc
elu.wandk5.cn/20082.Doc
elu.wandk5.cn/66480.Doc
elu.wandk5.cn/68484.Doc
elu.wandk5.cn/80826.Doc
elu.wandk5.cn/66602.Doc
elu.wandk5.cn/60824.Doc
elu.wandk5.cn/26802.Doc
ely.wandk5.cn/22602.Doc
ely.wandk5.cn/88806.Doc
ely.wandk5.cn/46084.Doc
ely.wandk5.cn/06266.Doc
ely.wandk5.cn/44482.Doc
ely.wandk5.cn/64060.Doc
ely.wandk5.cn/68604.Doc
ely.wandk5.cn/40842.Doc
ely.wandk5.cn/80286.Doc
ely.wandk5.cn/08642.Doc
elt.wandk5.cn/24408.Doc
elt.wandk5.cn/64600.Doc
elt.wandk5.cn/02422.Doc
elt.wandk5.cn/24024.Doc
elt.wandk5.cn/44082.Doc
elt.wandk5.cn/28640.Doc
elt.wandk5.cn/82646.Doc
elt.wandk5.cn/82842.Doc
elt.wandk5.cn/08628.Doc
elt.wandk5.cn/46244.Doc
elr.wandk5.cn/88080.Doc
elr.wandk5.cn/28444.Doc
elr.wandk5.cn/82064.Doc
elr.wandk5.cn/02444.Doc
elr.wandk5.cn/88600.Doc
elr.wandk5.cn/62626.Doc
elr.wandk5.cn/44048.Doc
elr.wandk5.cn/46646.Doc
elr.wandk5.cn/44024.Doc
elr.wandk5.cn/86888.Doc
ele.wandk5.cn/82048.Doc
ele.wandk5.cn/66442.Doc
ele.wandk5.cn/62682.Doc
ele.wandk5.cn/04002.Doc
ele.wandk5.cn/88240.Doc
ele.wandk5.cn/44028.Doc
ele.wandk5.cn/80220.Doc
ele.wandk5.cn/48802.Doc
ele.wandk5.cn/04862.Doc
ele.wandk5.cn/68666.Doc
elw.wandk5.cn/40020.Doc
elw.wandk5.cn/24244.Doc
elw.wandk5.cn/28428.Doc
elw.wandk5.cn/64264.Doc
elw.wandk5.cn/62220.Doc
elw.wandk5.cn/68222.Doc
elw.wandk5.cn/46844.Doc
elw.wandk5.cn/60846.Doc
elw.wandk5.cn/66824.Doc
elw.wandk5.cn/66842.Doc
elq.wandk5.cn/08882.Doc
elq.wandk5.cn/88064.Doc
elq.wandk5.cn/08624.Doc
elq.wandk5.cn/04646.Doc
elq.wandk5.cn/84828.Doc
elq.wandk5.cn/26600.Doc
elq.wandk5.cn/66024.Doc
elq.wandk5.cn/48608.Doc
elq.wandk5.cn/64026.Doc
elq.wandk5.cn/62880.Doc
ekm.wandk5.cn/80886.Doc
ekm.wandk5.cn/80668.Doc
ekm.wandk5.cn/68440.Doc
ekm.wandk5.cn/84666.Doc
ekm.wandk5.cn/88226.Doc
ekm.wandk5.cn/06248.Doc
ekm.wandk5.cn/64260.Doc
ekm.wandk5.cn/40064.Doc
ekm.wandk5.cn/06024.Doc
ekm.wandk5.cn/68842.Doc
ekn.wandk5.cn/04262.Doc
ekn.wandk5.cn/06046.Doc
ekn.wandk5.cn/24008.Doc
ekn.wandk5.cn/66040.Doc
ekn.wandk5.cn/64868.Doc
ekn.wandk5.cn/62664.Doc
ekn.wandk5.cn/82060.Doc
ekn.wandk5.cn/82248.Doc
ekn.wandk5.cn/84420.Doc
ekn.wandk5.cn/88406.Doc
ekb.wandk5.cn/86222.Doc
ekb.wandk5.cn/46060.Doc
ekb.wandk5.cn/66024.Doc
ekb.wandk5.cn/44248.Doc
ekb.wandk5.cn/82806.Doc
ekb.wandk5.cn/62862.Doc
ekb.wandk5.cn/48426.Doc
ekb.wandk5.cn/00866.Doc
ekb.wandk5.cn/84204.Doc
ekb.wandk5.cn/84062.Doc
ekv.wandk5.cn/82482.Doc
ekv.wandk5.cn/62088.Doc
ekv.wandk5.cn/73289.Doc
ekv.wandk5.cn/88400.Doc
ekv.wandk5.cn/26424.Doc
ekv.wandk5.cn/44242.Doc
ekv.wandk5.cn/46020.Doc
ekv.wandk5.cn/00680.Doc
ekv.wandk5.cn/66086.Doc
ekv.wandk5.cn/26200.Doc
ekc.wandk5.cn/08240.Doc
ekc.wandk5.cn/48606.Doc
ekc.wandk5.cn/24602.Doc
ekc.wandk5.cn/64442.Doc
ekc.wandk5.cn/02860.Doc
ekc.wandk5.cn/48624.Doc
ekc.wandk5.cn/20206.Doc
ekc.wandk5.cn/04866.Doc
ekc.wandk5.cn/40668.Doc
ekc.wandk5.cn/02840.Doc
ekx.wandk5.cn/64620.Doc
ekx.wandk5.cn/60088.Doc
ekx.wandk5.cn/64608.Doc
ekx.wandk5.cn/00864.Doc
ekx.wandk5.cn/08200.Doc
ekx.wandk5.cn/08060.Doc
ekx.wandk5.cn/62428.Doc
ekx.wandk5.cn/22462.Doc
ekx.wandk5.cn/88006.Doc
ekx.wandk5.cn/24864.Doc
ekz.wandk5.cn/62402.Doc
ekz.wandk5.cn/82886.Doc
ekz.wandk5.cn/88660.Doc
ekz.wandk5.cn/06264.Doc
ekz.wandk5.cn/04862.Doc
ekz.wandk5.cn/24882.Doc
ekz.wandk5.cn/26406.Doc
ekz.wandk5.cn/62602.Doc
ekz.wandk5.cn/20026.Doc
ekz.wandk5.cn/22046.Doc
ekl.wandk5.cn/48046.Doc
ekl.wandk5.cn/46408.Doc
ekl.wandk5.cn/22442.Doc
ekl.wandk5.cn/80860.Doc
ekl.wandk5.cn/02088.Doc
ekl.wandk5.cn/66820.Doc
ekl.wandk5.cn/08860.Doc
ekl.wandk5.cn/04408.Doc
ekl.wandk5.cn/46662.Doc
ekl.wandk5.cn/44264.Doc
ekk.wandk5.cn/04464.Doc
ekk.wandk5.cn/20426.Doc
ekk.wandk5.cn/66262.Doc
ekk.wandk5.cn/02442.Doc
ekk.wandk5.cn/84204.Doc
ekk.wandk5.cn/06620.Doc
ekk.wandk5.cn/60660.Doc
ekk.wandk5.cn/26604.Doc
ekk.wandk5.cn/46642.Doc
ekk.wandk5.cn/60022.Doc
ekj.wandk5.cn/68068.Doc
ekj.wandk5.cn/28040.Doc
ekj.wandk5.cn/06422.Doc
ekj.wandk5.cn/20262.Doc
ekj.wandk5.cn/88042.Doc
ekj.wandk5.cn/46284.Doc
ekj.wandk5.cn/20906.Doc
ekj.wandk5.cn/02064.Doc
ekj.wandk5.cn/20002.Doc
ekj.wandk5.cn/40060.Doc
ekh.wandk5.cn/28688.Doc
ekh.wandk5.cn/88046.Doc
ekh.wandk5.cn/04026.Doc
ekh.wandk5.cn/68880.Doc
ekh.wandk5.cn/28480.Doc
ekh.wandk5.cn/60046.Doc
ekh.wandk5.cn/08808.Doc
ekh.wandk5.cn/48046.Doc
ekh.wandk5.cn/44600.Doc
ekh.wandk5.cn/64804.Doc
ekg.wandk5.cn/82480.Doc
ekg.wandk5.cn/62086.Doc
ekg.wandk5.cn/06660.Doc
ekg.wandk5.cn/66264.Doc
ekg.wandk5.cn/44884.Doc
ekg.wandk5.cn/00864.Doc
ekg.wandk5.cn/62866.Doc
ekg.wandk5.cn/00266.Doc
ekg.wandk5.cn/86828.Doc
ekg.wandk5.cn/40020.Doc
ekf.wandk5.cn/04680.Doc
ekf.wandk5.cn/06020.Doc
ekf.wandk5.cn/20262.Doc
ekf.wandk5.cn/60242.Doc
ekf.wandk5.cn/06082.Doc
ekf.wandk5.cn/22644.Doc
ekf.wandk5.cn/28866.Doc
ekf.wandk5.cn/68024.Doc
ekf.wandk5.cn/82604.Doc
ekf.wandk5.cn/04024.Doc
ekd.wandk5.cn/26600.Doc
ekd.wandk5.cn/62804.Doc
ekd.wandk5.cn/08622.Doc
ekd.wandk5.cn/24626.Doc
ekd.wandk5.cn/00828.Doc
ekd.wandk5.cn/68604.Doc
ekd.wandk5.cn/40424.Doc
ekd.wandk5.cn/66820.Doc
ekd.wandk5.cn/48484.Doc
ekd.wandk5.cn/28800.Doc
eks.wandk5.cn/00822.Doc
eks.wandk5.cn/02648.Doc
eks.wandk5.cn/62822.Doc
eks.wandk5.cn/04040.Doc
eks.wandk5.cn/44008.Doc
eks.wandk5.cn/08262.Doc
eks.wandk5.cn/08460.Doc
eks.wandk5.cn/06408.Doc
eks.wandk5.cn/84466.Doc
eks.wandk5.cn/28624.Doc
eka.wandk5.cn/48282.Doc
eka.wandk5.cn/86244.Doc
eka.wandk5.cn/40484.Doc
eka.wandk5.cn/48680.Doc
eka.wandk5.cn/06048.Doc
eka.wandk5.cn/44446.Doc
eka.wandk5.cn/48480.Doc
eka.wandk5.cn/00460.Doc
eka.wandk5.cn/24028.Doc
eka.wandk5.cn/44022.Doc
ekp.wandk5.cn/60460.Doc
ekp.wandk5.cn/88400.Doc
ekp.wandk5.cn/66660.Doc
ekp.wandk5.cn/26402.Doc
ekp.wandk5.cn/60644.Doc
ekp.wandk5.cn/48264.Doc
ekp.wandk5.cn/46284.Doc
ekp.wandk5.cn/20426.Doc
ekp.wandk5.cn/40884.Doc
ekp.wandk5.cn/46600.Doc
eko.wandk5.cn/20000.Doc
eko.wandk5.cn/08800.Doc
eko.wandk5.cn/42428.Doc
eko.wandk5.cn/26262.Doc
eko.wandk5.cn/00860.Doc
eko.wandk5.cn/86082.Doc
eko.wandk5.cn/84262.Doc
eko.wandk5.cn/60888.Doc
eko.wandk5.cn/88904.Doc
eko.wandk5.cn/60460.Doc
eki.wandk5.cn/46642.Doc
eki.wandk5.cn/82602.Doc
eki.wandk5.cn/44846.Doc
eki.wandk5.cn/46224.Doc
eki.wandk5.cn/02626.Doc
eki.wandk5.cn/48244.Doc
eki.wandk5.cn/60808.Doc
eki.wandk5.cn/82868.Doc
eki.wandk5.cn/48682.Doc
eki.wandk5.cn/80224.Doc
eku.wandk5.cn/88426.Doc
eku.wandk5.cn/06604.Doc
eku.wandk5.cn/86646.Doc
eku.wandk5.cn/40608.Doc
eku.wandk5.cn/60404.Doc
eku.wandk5.cn/80844.Doc
eku.wandk5.cn/80220.Doc
eku.wandk5.cn/48240.Doc
eku.wandk5.cn/86220.Doc
eku.wandk5.cn/28776.Doc
