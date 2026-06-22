文肯诙诜亲


Python 组合优于继承
==========================

"优先使用组合而非继承"是 GoF 设计模式的核心建议。
本文将深入讲解组合的 Python 实现及适用场景。

1. has-a vs is-a 关系
------------------------
继承表示"是一个"（is-a），组合表示"有一个"（has-a）。

class Engine:
    """发动机——可被组合进汽车"""
    def start(self) -> str:
        return "发动机启动"


class Car:
    """汽车"有一个"发动机——组合关系"""
    def __init__(self):
        # 在 __init__ 中包含其他对象
        self.engine = Engine()
        self.wheels = [Wheel() for _ in range(4)]

    def drive(self) -> str:
        return self.engine.start() + "，汽车行驶"


class Wheel:
    def rotate(self) -> str:
        return "车轮转动"


# 使用：Car 不需要继承 Engine 就能使用其功能
car = Car()
print(car.drive())  # 输出：发动机启动，汽车行驶


2. 通过 __getattr__ 实现委托
-------------------------------
组合可以配合 __getattr__ 实现透明的委托，让对象看起来像继承。

class Writer:
    """负责写入操作"""
    def write(self, data: str) -> None:
        print(f"写入: {data}")


class Logger:
    """日志器——组合 Writer，通过委托调用其方法"""
    def __init__(self):
        self._writer = Writer()  # 组合 Writer 对象

    def __getattr__(self, name: str):
        """当自身没有该属性时，委托给 _writer"""
        # 注意：避免递归，需确保 _writer 已存在
        if name == '_writer':
            raise AttributeError(name)
        return getattr(self._writer, name)


log = Logger()
log.write("日志消息")  # 委托给 Writer.write()，输出：写入: 日志消息


3. 策略模式——组合的经典应用
----------------------------------
策略模式通过组合不同的算法实现行为变化。

from typing import List

class SortStrategy:
    """排序策略接口"""
    def sort(self, data: List[int]) -> List[int]: ...


class BubbleSort(SortStrategy):
    def sort(self, data: List[int]) -> List[int]:
        arr = data.copy()
        n = len(arr)
        for i in range(n):
            for j in range(0, n - i - 1):
                if arr[j] > arr[j + 1]:
                    arr[j], arr[j + 1] = arr[j + 1], arr[j]
        return arr


class QuickSort(SortStrategy):
    def sort(self, data: List[int]) -> List[int]:
        if len(data) <= 1:
            return data
        pivot = data[0]
        left = [x for x in data[1:] if x <= pivot]
        right = [x for x in data[1:] if x > pivot]
        return self.sort(left) + [pivot] + self.sort(right)


class DataProcessor:
    """数据处理器——组合排序策略，运行时切换"""
    def __init__(self, strategy: SortStrategy):
        self._strategy = strategy  # 组合策略对象

    def set_strategy(self, strategy: SortStrategy) -> None:
        self._strategy = strategy  # 运行时切换策略

    def process(self, data: List[int]) -> List[int]:
        return self._strategy.sort(data)


dp = DataProcessor(BubbleSort())
print(dp.process([3, 1, 2]))  # [1, 2, 3]
dp.set_strategy(QuickSort())  # 切换策略
print(dp.process([3, 1, 2]))  # [1, 2, 3]


4. 包装器/装饰器模式
------------------------
组合可以用来包装对象，在调用前后添加行为。

class DataSource:
    """基础数据源"""
    def read(self) -> str:
        return "原始数据"


class CompressedDataSource:
    """压缩数据包装器——组合 DataSource 并添加压缩功能"""
    def __init__(self, source: DataSource):
        self._source = source  # 组合原始数据源

    def read(self) -> str:
        data = self._source.read()
        return f"[解压]{data}"  # 添加额外行为


class EncryptedDataSource:
    """加密数据包装器——多层组合"""
    def __init__(self, source: DataSource):
        self._source = source

    def read(self) -> str:
        data = self._source.read()
        return f"[解密]{data}"


# 多层包装：原始 -> 压缩 -> 加密
source = EncryptedDataSource(
    CompressedDataSource(
        DataSource()
    )
)
print(source.read())  # 输出：[解密][解压]原始数据


5. 用 Mixin 组合行为
----------------------
Mixin 类提供可复用的行为片段，通过多重继承组合。

class JsonMixin:
    """提供 JSON 序列化能力的 Mixin"""
    def to_json(self) -> str:
        import json
        return json.dumps(self.__dict__, ensure_ascii=False)


class XmlMixin:
    """提供 XML 序列化能力的 Mixin"""
    def to_xml(self) -> str:
        parts = [f"<{k}>{v}</{k}>" for k, v in self.__dict__.items()]
        return f"<root>{''.join(parts)}</root>"


class User(JsonMixin, XmlMixin):
    """User 类通过 Mixin 组合序列化行为，而非继承序列化器"""
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age


user = User("张三", 28)
print(user.to_json())  # {"name": "张三", "age": 28}
print(user.to_xml())   # <root><name>张三</name><age>28</age></root>


6. 何时仍然使用继承
---------------------
继承并非完全不可用，以下场景更适合继承：
- 明确的 is-a 关系（如 Dog is-a Animal）
- 子类需要复用父类大部分实现
- 框架要求子类化（如 Django ModelViewSet）

from dataclasses import dataclass

@dataclass
class Animal:
    """基类——明确的 is-a 关系"""
    name: str

    def speak(self) -> str:
        return "..."

class Dog(Animal):
    """Dog is-a Animal——继承合理"""
    def speak(self) -> str:
        return "汪汪！"


总结：优先使用组合能让系统更灵活、耦合度更低。组合通过 __init__
包含对象、__getattr__ 委托、策略模式交换行为、装饰器模式包装功能
等方式实现。Mixin 提供了行为组合的中间方案。只有在明确的 is-a
关系且需要复用大量实现时才选择继承。

诿忱庞段脑闻我窍准筛籽劝仆度乱

wut.mmmxz.cn/137113.htm
wut.mmmxz.cn/620463.htm
wut.mmmxz.cn/793333.htm
wut.mmmxz.cn/062243.htm
wut.mmmxz.cn/157533.htm
wut.mmmxz.cn/175313.htm
wut.mmmxz.cn/220443.htm
wut.mmmxz.cn/571193.htm
wut.mmmxz.cn/428483.htm
wut.mmmxz.cn/668403.htm
wur.mmmxz.cn/440083.htm
wur.mmmxz.cn/820483.htm
wur.mmmxz.cn/573973.htm
wur.mmmxz.cn/719953.htm
wur.mmmxz.cn/800663.htm
wur.mmmxz.cn/993173.htm
wur.mmmxz.cn/997733.htm
wur.mmmxz.cn/379993.htm
wur.mmmxz.cn/862883.htm
wur.mmmxz.cn/404243.htm
wue.mmmxz.cn/397333.htm
wue.mmmxz.cn/551773.htm
wue.mmmxz.cn/931113.htm
wue.mmmxz.cn/606423.htm
wue.mmmxz.cn/660643.htm
wue.mmmxz.cn/999333.htm
wue.mmmxz.cn/571393.htm
wue.mmmxz.cn/608043.htm
wue.mmmxz.cn/840803.htm
wue.mmmxz.cn/262423.htm
wuw.mmmxz.cn/795953.htm
wuw.mmmxz.cn/597173.htm
wuw.mmmxz.cn/006483.htm
wuw.mmmxz.cn/373393.htm
wuw.mmmxz.cn/111313.htm
wuw.mmmxz.cn/422223.htm
wuw.mmmxz.cn/795733.htm
wuw.mmmxz.cn/806283.htm
wuw.mmmxz.cn/999533.htm
wuw.mmmxz.cn/044023.htm
wuq.mmmxz.cn/204483.htm
wuq.mmmxz.cn/931393.htm
wuq.mmmxz.cn/002083.htm
wuq.mmmxz.cn/397993.htm
wuq.mmmxz.cn/466223.htm
wuq.mmmxz.cn/026823.htm
wuq.mmmxz.cn/333193.htm
wuq.mmmxz.cn/951533.htm
wuq.mmmxz.cn/199993.htm
wuq.mmmxz.cn/462003.htm
wym.mmmxz.cn/004283.htm
wym.mmmxz.cn/311313.htm
wym.mmmxz.cn/395913.htm
wym.mmmxz.cn/068603.htm
wym.mmmxz.cn/517553.htm
wym.mmmxz.cn/004403.htm
wym.mmmxz.cn/937193.htm
wym.mmmxz.cn/886843.htm
wym.mmmxz.cn/733193.htm
wym.mmmxz.cn/399753.htm
wyn.mmmxz.cn/266283.htm
wyn.mmmxz.cn/193173.htm
wyn.mmmxz.cn/911373.htm
wyn.mmmxz.cn/339313.htm
wyn.mmmxz.cn/979513.htm
wyn.mmmxz.cn/824263.htm
wyn.mmmxz.cn/319933.htm
wyn.mmmxz.cn/680843.htm
wyn.mmmxz.cn/248883.htm
wyn.mmmxz.cn/757133.htm
wyb.mmmxz.cn/739333.htm
wyb.mmmxz.cn/068603.htm
wyb.mmmxz.cn/268223.htm
wyb.mmmxz.cn/260603.htm
wyb.mmmxz.cn/311533.htm
wyb.mmmxz.cn/717193.htm
wyb.mmmxz.cn/846263.htm
wyb.mmmxz.cn/133973.htm
wyb.mmmxz.cn/202043.htm
wyb.mmmxz.cn/113113.htm
wyv.mmmxz.cn/357913.htm
wyv.mmmxz.cn/668263.htm
wyv.mmmxz.cn/715593.htm
wyv.mmmxz.cn/753313.htm
wyv.mmmxz.cn/800403.htm
wyv.mmmxz.cn/206843.htm
wyv.mmmxz.cn/824803.htm
wyv.mmmxz.cn/131753.htm
wyv.mmmxz.cn/440443.htm
wyv.mmmxz.cn/822023.htm
wyc.mmmxz.cn/991173.htm
wyc.mmmxz.cn/880263.htm
wyc.mmmxz.cn/791313.htm
wyc.mmmxz.cn/357193.htm
wyc.mmmxz.cn/668263.htm
wyc.mmmxz.cn/373173.htm
wyc.mmmxz.cn/731713.htm
wyc.mmmxz.cn/600443.htm
wyc.mmmxz.cn/177913.htm
wyc.mmmxz.cn/597373.htm
wyx.mmmxz.cn/333173.htm
wyx.mmmxz.cn/244443.htm
wyx.mmmxz.cn/268263.htm
wyx.mmmxz.cn/133993.htm
wyx.mmmxz.cn/977513.htm
wyx.mmmxz.cn/262863.htm
wyx.mmmxz.cn/351773.htm
wyx.mmmxz.cn/266003.htm
wyx.mmmxz.cn/797593.htm
wyx.mmmxz.cn/571953.htm
wyz.mmmxz.cn/688063.htm
wyz.mmmxz.cn/791733.htm
wyz.mmmxz.cn/517393.htm
wyz.mmmxz.cn/864423.htm
wyz.mmmxz.cn/226623.htm
wyz.mmmxz.cn/400063.htm
wyz.mmmxz.cn/333113.htm
wyz.mmmxz.cn/046803.htm
wyz.mmmxz.cn/446883.htm
wyz.mmmxz.cn/993313.htm
wyl.mmmxz.cn/713553.htm
wyl.mmmxz.cn/553973.htm
wyl.mmmxz.cn/573953.htm
wyl.mmmxz.cn/662263.htm
wyl.mmmxz.cn/999353.htm
wyl.mmmxz.cn/082683.htm
wyl.mmmxz.cn/062223.htm
wyl.mmmxz.cn/939353.htm
wyl.mmmxz.cn/915573.htm
wyl.mmmxz.cn/466483.htm
wyk.mmmxz.cn/795353.htm
wyk.mmmxz.cn/884083.htm
wyk.mmmxz.cn/191753.htm
wyk.mmmxz.cn/606463.htm
wyk.mmmxz.cn/280423.htm
wyk.mmmxz.cn/577533.htm
wyk.mmmxz.cn/773393.htm
wyk.mmmxz.cn/553513.htm
wyk.mmmxz.cn/828203.htm
wyk.mmmxz.cn/628683.htm
wyj.mmmxz.cn/731533.htm
wyj.mmmxz.cn/533533.htm
wyj.mmmxz.cn/042003.htm
wyj.mmmxz.cn/537333.htm
wyj.mmmxz.cn/080883.htm
wyj.mmmxz.cn/577353.htm
wyj.mmmxz.cn/480883.htm
wyj.mmmxz.cn/002003.htm
wyj.mmmxz.cn/393753.htm
wyj.mmmxz.cn/955153.htm
wyh.mmmxz.cn/395593.htm
wyh.mmmxz.cn/775153.htm
wyh.mmmxz.cn/488023.htm
wyh.mmmxz.cn/191933.htm
wyh.mmmxz.cn/335573.htm
wyh.mmmxz.cn/402283.htm
wyh.mmmxz.cn/711793.htm
wyh.mmmxz.cn/975353.htm
wyh.mmmxz.cn/242063.htm
wyh.mmmxz.cn/553333.htm
wyg.mmmxz.cn/260623.htm
wyg.mmmxz.cn/937133.htm
wyg.mmmxz.cn/826623.htm
wyg.mmmxz.cn/848083.htm
wyg.mmmxz.cn/397353.htm
wyg.mmmxz.cn/995593.htm
wyg.mmmxz.cn/535513.htm
wyg.mmmxz.cn/731333.htm
wyg.mmmxz.cn/244483.htm
wyg.mmmxz.cn/939153.htm
wyf.mmmxz.cn/937373.htm
wyf.mmmxz.cn/668083.htm
wyf.mmmxz.cn/559373.htm
wyf.mmmxz.cn/993753.htm
wyf.mmmxz.cn/593713.htm
wyf.mmmxz.cn/268863.htm
wyf.mmmxz.cn/820023.htm
wyf.mmmxz.cn/393593.htm
wyf.mmmxz.cn/191513.htm
wyf.mmmxz.cn/246663.htm
wyd.mmmxz.cn/317713.htm
wyd.mmmxz.cn/662843.htm
wyd.mmmxz.cn/826643.htm
wyd.mmmxz.cn/622003.htm
wyd.mmmxz.cn/206883.htm
wyd.mmmxz.cn/759533.htm
wyd.mmmxz.cn/157193.htm
wyd.mmmxz.cn/642043.htm
wyd.mmmxz.cn/399193.htm
wyd.mmmxz.cn/208223.htm
wys.mmmxz.cn/117973.htm
wys.mmmxz.cn/082483.htm
wys.mmmxz.cn/642443.htm
wys.mmmxz.cn/991153.htm
wys.mmmxz.cn/797593.htm
wys.mmmxz.cn/428843.htm
wys.mmmxz.cn/731993.htm
wys.mmmxz.cn/880203.htm
wys.mmmxz.cn/557333.htm
wys.mmmxz.cn/486843.htm
wya.mmmxz.cn/686863.htm
wya.mmmxz.cn/777333.htm
wya.mmmxz.cn/757733.htm
wya.mmmxz.cn/224803.htm
wya.mmmxz.cn/937333.htm
wya.mmmxz.cn/248223.htm
wya.mmmxz.cn/799533.htm
wya.mmmxz.cn/135973.htm
wya.mmmxz.cn/248263.htm
wya.mmmxz.cn/971793.htm
wyp.mmmxz.cn/791173.htm
wyp.mmmxz.cn/628643.htm
wyp.mmmxz.cn/317333.htm
wyp.mmmxz.cn/313393.htm
wyp.mmmxz.cn/600403.htm
wyp.mmmxz.cn/779733.htm
wyp.mmmxz.cn/084663.htm
wyp.mmmxz.cn/355193.htm
wyp.mmmxz.cn/486263.htm
wyp.mmmxz.cn/484223.htm
wyo.mmmxz.cn/773193.htm
wyo.mmmxz.cn/713533.htm
wyo.mmmxz.cn/604663.htm
wyo.mmmxz.cn/262423.htm
wyo.mmmxz.cn/682623.htm
wyo.mmmxz.cn/553173.htm
wyo.mmmxz.cn/737753.htm
wyo.mmmxz.cn/571373.htm
wyo.mmmxz.cn/153973.htm
wyo.mmmxz.cn/464003.htm
wyi.mmmxz.cn/777333.htm
wyi.mmmxz.cn/424623.htm
wyi.mmmxz.cn/151593.htm
wyi.mmmxz.cn/200243.htm
wyi.mmmxz.cn/288403.htm
wyi.mmmxz.cn/115133.htm
wyi.mmmxz.cn/975713.htm
wyi.mmmxz.cn/608683.htm
wyi.mmmxz.cn/793773.htm
wyi.mmmxz.cn/377373.htm
wyu.mmmxz.cn/739753.htm
wyu.mmmxz.cn/933513.htm
wyu.mmmxz.cn/040023.htm
wyu.mmmxz.cn/951593.htm
wyu.mmmxz.cn/371733.htm
wyu.mmmxz.cn/048443.htm
wyu.mmmxz.cn/117713.htm
wyu.mmmxz.cn/024403.htm
wyu.mmmxz.cn/179153.htm
wyu.mmmxz.cn/006243.htm
wyy.mmmxz.cn/028023.htm
wyy.mmmxz.cn/773373.htm
wyy.mmmxz.cn/311133.htm
wyy.mmmxz.cn/284263.htm
wyy.mmmxz.cn/391173.htm
wyy.mmmxz.cn/804843.htm
wyy.mmmxz.cn/177753.htm
wyy.mmmxz.cn/822463.htm
wyy.mmmxz.cn/084623.htm
wyy.mmmxz.cn/359393.htm
wyt.mmmxz.cn/759333.htm
wyt.mmmxz.cn/357373.htm
wyt.mmmxz.cn/888263.htm
wyt.mmmxz.cn/000003.htm
wyt.mmmxz.cn/777353.htm
wyt.mmmxz.cn/577733.htm
wyt.mmmxz.cn/860463.htm
wyt.mmmxz.cn/115393.htm
wyt.mmmxz.cn/937553.htm
wyt.mmmxz.cn/286063.htm
wyr.mmmxz.cn/959153.htm
wyr.mmmxz.cn/842683.htm
wyr.mmmxz.cn/351753.htm
wyr.mmmxz.cn/393753.htm
wyr.mmmxz.cn/844023.htm
wyr.mmmxz.cn/957393.htm
wyr.mmmxz.cn/791113.htm
wyr.mmmxz.cn/200423.htm
wyr.mmmxz.cn/315933.htm
wyr.mmmxz.cn/288043.htm
wye.mmmxz.cn/531193.htm
wye.mmmxz.cn/379733.htm
wye.mmmxz.cn/028223.htm
wye.mmmxz.cn/753173.htm
wye.mmmxz.cn/753513.htm
wye.mmmxz.cn/886063.htm
wye.mmmxz.cn/979993.htm
wye.mmmxz.cn/622883.htm
wye.mmmxz.cn/953333.htm
wye.mmmxz.cn/751913.htm
wyw.mmmxz.cn/822223.htm
wyw.mmmxz.cn/535193.htm
wyw.mmmxz.cn/579773.htm
wyw.mmmxz.cn/040623.htm
wyw.mmmxz.cn/997373.htm
wyw.mmmxz.cn/117573.htm
wyw.mmmxz.cn/913933.htm
wyw.mmmxz.cn/157993.htm
wyw.mmmxz.cn/200423.htm
wyw.mmmxz.cn/159713.htm
wyq.mmmxz.cn/359933.htm
wyq.mmmxz.cn/240063.htm
wyq.mmmxz.cn/240243.htm
wyq.mmmxz.cn/622283.htm
wyq.mmmxz.cn/599573.htm
wyq.mmmxz.cn/484823.htm
wyq.mmmxz.cn/848463.htm
wyq.mmmxz.cn/111753.htm
wyq.mmmxz.cn/753173.htm
wyq.mmmxz.cn/024823.htm
wtm.mmmxz.cn/197753.htm
wtm.mmmxz.cn/446263.htm
wtm.mmmxz.cn/991133.htm
wtm.mmmxz.cn/006223.htm
wtm.mmmxz.cn/266223.htm
wtm.mmmxz.cn/755133.htm
wtm.mmmxz.cn/155753.htm
wtm.mmmxz.cn/155153.htm
wtm.mmmxz.cn/977913.htm
wtm.mmmxz.cn/420223.htm
wtn.mmmxz.cn/313973.htm
wtn.mmmxz.cn/753993.htm
wtn.mmmxz.cn/222683.htm
wtn.mmmxz.cn/353713.htm
wtn.mmmxz.cn/373373.htm
wtn.mmmxz.cn/400843.htm
wtn.mmmxz.cn/937773.htm
wtn.mmmxz.cn/082843.htm
wtn.mmmxz.cn/713193.htm
wtn.mmmxz.cn/977573.htm
wtb.mmmxz.cn/244223.htm
wtb.mmmxz.cn/557333.htm
wtb.mmmxz.cn/311513.htm
wtb.mmmxz.cn/688243.htm
wtb.mmmxz.cn/979373.htm
wtb.mmmxz.cn/668683.htm
wtb.mmmxz.cn/775913.htm
wtb.mmmxz.cn/139733.htm
wtb.mmmxz.cn/660243.htm
wtb.mmmxz.cn/937933.htm
wtv.mmmxz.cn/602443.htm
wtv.mmmxz.cn/191153.htm
wtv.mmmxz.cn/773353.htm
wtv.mmmxz.cn/684403.htm
wtv.mmmxz.cn/335973.htm
wtv.mmmxz.cn/951333.htm
wtv.mmmxz.cn/204863.htm
wtv.mmmxz.cn/799553.htm
wtv.mmmxz.cn/777933.htm
wtv.mmmxz.cn/339733.htm
wtc.mmmxz.cn/020243.htm
wtc.mmmxz.cn/866663.htm
wtc.mmmxz.cn/753533.htm
wtc.mmmxz.cn/337353.htm
wtc.mmmxz.cn/393353.htm
wtc.mmmxz.cn/866403.htm
wtc.mmmxz.cn/288083.htm
wtc.mmmxz.cn/331753.htm
wtc.mmmxz.cn/373353.htm
wtc.mmmxz.cn/084423.htm
wtx.mmmxz.cn/193953.htm
wtx.mmmxz.cn/513553.htm
wtx.mmmxz.cn/751533.htm
wtx.mmmxz.cn/080823.htm
wtx.mmmxz.cn/844003.htm
wtx.mmmxz.cn/133533.htm
wtx.mmmxz.cn/559973.htm
wtx.mmmxz.cn/068063.htm
wtx.mmmxz.cn/391973.htm
wtx.mmmxz.cn/808803.htm
wtz.mmmxz.cn/713373.htm
wtz.mmmxz.cn/311133.htm
wtz.mmmxz.cn/822483.htm
wtz.mmmxz.cn/539533.htm
wtz.mmmxz.cn/117573.htm
wtz.mmmxz.cn/000863.htm
wtz.mmmxz.cn/531733.htm
wtz.mmmxz.cn/604823.htm
wtz.mmmxz.cn/317353.htm
wtz.mmmxz.cn/060023.htm
wtl.mmmxz.cn/880423.htm
wtl.mmmxz.cn/939313.htm
wtl.mmmxz.cn/135113.htm
wtl.mmmxz.cn/284283.htm
wtl.mmmxz.cn/113573.htm
wtl.mmmxz.cn/264003.htm
wtl.mmmxz.cn/464643.htm
wtl.mmmxz.cn/284023.htm
wtl.mmmxz.cn/822043.htm
wtl.mmmxz.cn/119573.htm
wtk.mmmxz.cn/715733.htm
wtk.mmmxz.cn/408023.htm
wtk.mmmxz.cn/519173.htm
wtk.mmmxz.cn/466483.htm
wtk.mmmxz.cn/739173.htm
