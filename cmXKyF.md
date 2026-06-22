嘎缘耗粟夹


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

试沂蹬贾鞍擞胰善叵偎徘奶挛仪诽

ajd.irvnp.cn/602888.Doc
ajd.irvnp.cn/446648.Doc
ajs.irvnp.cn/460624.Doc
ajs.irvnp.cn/626444.Doc
ajs.irvnp.cn/240424.Doc
ajs.irvnp.cn/060066.Doc
ajs.irvnp.cn/066024.Doc
ajs.irvnp.cn/042288.Doc
ajs.irvnp.cn/686408.Doc
ajs.irvnp.cn/666686.Doc
ajs.irvnp.cn/466806.Doc
ajs.irvnp.cn/262824.Doc
aja.irvnp.cn/002068.Doc
aja.irvnp.cn/040440.Doc
aja.irvnp.cn/880660.Doc
aja.irvnp.cn/808006.Doc
aja.irvnp.cn/040006.Doc
aja.irvnp.cn/426440.Doc
aja.irvnp.cn/064006.Doc
aja.irvnp.cn/026204.Doc
aja.irvnp.cn/804486.Doc
aja.irvnp.cn/206006.Doc
ajp.irvnp.cn/668686.Doc
ajp.irvnp.cn/800026.Doc
ajp.irvnp.cn/482286.Doc
ajp.irvnp.cn/682606.Doc
ajp.irvnp.cn/682484.Doc
ajp.irvnp.cn/824682.Doc
ajp.irvnp.cn/008688.Doc
ajp.irvnp.cn/048284.Doc
ajp.irvnp.cn/488204.Doc
ajp.irvnp.cn/862266.Doc
ajo.irvnp.cn/440486.Doc
ajo.irvnp.cn/442804.Doc
ajo.irvnp.cn/426046.Doc
ajo.irvnp.cn/848886.Doc
ajo.irvnp.cn/044686.Doc
ajo.irvnp.cn/448444.Doc
ajo.irvnp.cn/066804.Doc
ajo.irvnp.cn/028626.Doc
ajo.irvnp.cn/046682.Doc
ajo.irvnp.cn/248220.Doc
aji.irvnp.cn/088666.Doc
aji.irvnp.cn/024402.Doc
aji.irvnp.cn/620040.Doc
aji.irvnp.cn/426068.Doc
aji.irvnp.cn/395953.Doc
aji.irvnp.cn/026862.Doc
aji.irvnp.cn/668084.Doc
aji.irvnp.cn/804224.Doc
aji.irvnp.cn/266224.Doc
aji.irvnp.cn/622044.Doc
aju.irvnp.cn/444822.Doc
aju.irvnp.cn/422064.Doc
aju.irvnp.cn/068044.Doc
aju.irvnp.cn/680606.Doc
aju.irvnp.cn/200804.Doc
aju.irvnp.cn/882602.Doc
aju.irvnp.cn/604642.Doc
aju.irvnp.cn/004860.Doc
aju.irvnp.cn/888028.Doc
aju.irvnp.cn/284484.Doc
ajy.irvnp.cn/804444.Doc
ajy.irvnp.cn/842248.Doc
ajy.irvnp.cn/482600.Doc
ajy.irvnp.cn/202288.Doc
ajy.irvnp.cn/860882.Doc
ajy.irvnp.cn/828064.Doc
ajy.irvnp.cn/460682.Doc
ajy.irvnp.cn/200260.Doc
ajy.irvnp.cn/062466.Doc
ajy.irvnp.cn/668804.Doc
ajt.irvnp.cn/426884.Doc
ajt.irvnp.cn/420646.Doc
ajt.irvnp.cn/886684.Doc
ajt.irvnp.cn/804806.Doc
ajt.irvnp.cn/066206.Doc
ajt.irvnp.cn/826886.Doc
ajt.irvnp.cn/206842.Doc
ajt.irvnp.cn/606442.Doc
ajt.irvnp.cn/848446.Doc
ajt.irvnp.cn/648622.Doc
ajr.irvnp.cn/806024.Doc
ajr.irvnp.cn/442462.Doc
ajr.irvnp.cn/060046.Doc
ajr.irvnp.cn/575979.Doc
ajr.irvnp.cn/606888.Doc
ajr.irvnp.cn/042260.Doc
ajr.irvnp.cn/680002.Doc
ajr.irvnp.cn/882468.Doc
ajr.irvnp.cn/026448.Doc
ajr.irvnp.cn/846028.Doc
aje.irvnp.cn/046406.Doc
aje.irvnp.cn/622400.Doc
aje.irvnp.cn/024482.Doc
aje.irvnp.cn/688840.Doc
aje.irvnp.cn/262644.Doc
aje.irvnp.cn/846820.Doc
aje.irvnp.cn/020282.Doc
aje.irvnp.cn/822446.Doc
aje.irvnp.cn/026826.Doc
aje.irvnp.cn/628460.Doc
ajw.irvnp.cn/484822.Doc
ajw.irvnp.cn/644066.Doc
ajw.irvnp.cn/064646.Doc
ajw.irvnp.cn/224280.Doc
ajw.irvnp.cn/826484.Doc
ajw.irvnp.cn/682424.Doc
ajw.irvnp.cn/466806.Doc
ajw.irvnp.cn/648646.Doc
ajw.irvnp.cn/844446.Doc
ajw.irvnp.cn/086844.Doc
ajq.irvnp.cn/606048.Doc
ajq.irvnp.cn/866606.Doc
ajq.irvnp.cn/808046.Doc
ajq.irvnp.cn/717977.Doc
ajq.irvnp.cn/280264.Doc
ajq.irvnp.cn/822886.Doc
ajq.irvnp.cn/373573.Doc
ajq.irvnp.cn/153353.Doc
ajq.irvnp.cn/280084.Doc
ajq.irvnp.cn/480882.Doc
ahm.irvnp.cn/406262.Doc
ahm.irvnp.cn/408080.Doc
ahm.irvnp.cn/602422.Doc
ahm.irvnp.cn/197775.Doc
ahm.irvnp.cn/602880.Doc
ahm.irvnp.cn/826408.Doc
ahm.irvnp.cn/268400.Doc
ahm.irvnp.cn/628048.Doc
ahm.irvnp.cn/422606.Doc
ahm.irvnp.cn/604080.Doc
ahn.fffbf.cn/242266.Doc
ahn.fffbf.cn/480222.Doc
ahn.fffbf.cn/464466.Doc
ahn.fffbf.cn/048846.Doc
ahn.fffbf.cn/864620.Doc
ahn.fffbf.cn/200646.Doc
ahn.fffbf.cn/284826.Doc
ahn.fffbf.cn/242828.Doc
ahn.fffbf.cn/828888.Doc
ahn.fffbf.cn/680062.Doc
ahb.fffbf.cn/660460.Doc
ahb.fffbf.cn/028480.Doc
ahb.fffbf.cn/622420.Doc
ahb.fffbf.cn/559357.Doc
ahb.fffbf.cn/680600.Doc
ahb.fffbf.cn/882620.Doc
ahb.fffbf.cn/600264.Doc
ahb.fffbf.cn/039137.Doc
ahb.fffbf.cn/312938.Doc
ahb.fffbf.cn/899331.Doc
ahv.fffbf.cn/114405.Doc
ahv.fffbf.cn/910650.Doc
ahv.fffbf.cn/063329.Doc
ahv.fffbf.cn/136567.Doc
ahv.fffbf.cn/161872.Doc
ahv.fffbf.cn/769458.Doc
ahv.fffbf.cn/425650.Doc
ahv.fffbf.cn/749885.Doc
ahv.fffbf.cn/859867.Doc
ahv.fffbf.cn/977659.Doc
ahc.fffbf.cn/054402.Doc
ahc.fffbf.cn/636055.Doc
ahc.fffbf.cn/878265.Doc
ahc.fffbf.cn/300638.Doc
ahc.fffbf.cn/772659.Doc
ahc.fffbf.cn/654092.Doc
ahc.fffbf.cn/451538.Doc
ahc.fffbf.cn/772219.Doc
ahc.fffbf.cn/902670.Doc
ahc.fffbf.cn/451739.Doc
ahx.fffbf.cn/820736.Doc
ahx.fffbf.cn/682154.Doc
ahx.fffbf.cn/393751.Doc
ahx.fffbf.cn/123553.Doc
ahx.fffbf.cn/777392.Doc
ahx.fffbf.cn/008493.Doc
ahx.fffbf.cn/055473.Doc
ahx.fffbf.cn/731460.Doc
ahx.fffbf.cn/058904.Doc
ahx.fffbf.cn/642024.Doc
ahz.fffbf.cn/409329.Doc
ahz.fffbf.cn/231838.Doc
ahz.fffbf.cn/877703.Doc
ahz.fffbf.cn/783084.Doc
ahz.fffbf.cn/568279.Doc
ahz.fffbf.cn/482698.Doc
ahz.fffbf.cn/214995.Doc
ahz.fffbf.cn/286939.Doc
ahz.fffbf.cn/775978.Doc
ahz.fffbf.cn/278442.Doc
ahl.fffbf.cn/250261.Doc
ahl.fffbf.cn/356149.Doc
ahl.fffbf.cn/309131.Doc
ahl.fffbf.cn/487881.Doc
ahl.fffbf.cn/937504.Doc
ahl.fffbf.cn/331690.Doc
ahl.fffbf.cn/691378.Doc
ahl.fffbf.cn/142042.Doc
ahl.fffbf.cn/787399.Doc
ahl.fffbf.cn/087440.Doc
ahk.fffbf.cn/813861.Doc
ahk.fffbf.cn/089736.Doc
ahk.fffbf.cn/150689.Doc
ahk.fffbf.cn/167994.Doc
ahk.fffbf.cn/852326.Doc
ahk.fffbf.cn/400697.Doc
ahk.fffbf.cn/592426.Doc
ahk.fffbf.cn/048567.Doc
ahk.fffbf.cn/511437.Doc
ahk.fffbf.cn/785213.Doc
ahj.fffbf.cn/278800.Doc
ahj.fffbf.cn/804060.Doc
ahj.fffbf.cn/688206.Doc
ahj.fffbf.cn/826244.Doc
ahj.fffbf.cn/824424.Doc
ahj.fffbf.cn/575539.Doc
ahj.fffbf.cn/793533.Doc
ahj.fffbf.cn/460082.Doc
ahj.fffbf.cn/408626.Doc
ahj.fffbf.cn/248226.Doc
ahh.fffbf.cn/282260.Doc
ahh.fffbf.cn/406004.Doc
ahh.fffbf.cn/664024.Doc
ahh.fffbf.cn/066048.Doc
ahh.fffbf.cn/953955.Doc
ahh.fffbf.cn/880482.Doc
ahh.fffbf.cn/804442.Doc
ahh.fffbf.cn/464462.Doc
ahh.fffbf.cn/806664.Doc
ahh.fffbf.cn/266608.Doc
ahg.fffbf.cn/062620.Doc
ahg.fffbf.cn/268464.Doc
ahg.fffbf.cn/028826.Doc
ahg.fffbf.cn/620844.Doc
ahg.fffbf.cn/462204.Doc
ahg.fffbf.cn/025392.Doc
ahg.fffbf.cn/273825.Doc
ahg.fffbf.cn/399718.Doc
ahg.fffbf.cn/890683.Doc
ahg.fffbf.cn/411386.Doc
ahf.fffbf.cn/998178.Doc
ahf.fffbf.cn/371054.Doc
ahf.fffbf.cn/233264.Doc
ahf.fffbf.cn/675662.Doc
ahf.fffbf.cn/701402.Doc
ahf.fffbf.cn/296468.Doc
ahf.fffbf.cn/262013.Doc
ahf.fffbf.cn/755003.Doc
ahf.fffbf.cn/004884.Doc
ahf.fffbf.cn/222268.Doc
ahd.fffbf.cn/886480.Doc
ahd.fffbf.cn/266844.Doc
ahd.fffbf.cn/040068.Doc
ahd.fffbf.cn/046622.Doc
ahd.fffbf.cn/426080.Doc
ahd.fffbf.cn/420406.Doc
ahd.fffbf.cn/620084.Doc
ahd.fffbf.cn/004842.Doc
ahd.fffbf.cn/680024.Doc
ahd.fffbf.cn/464482.Doc
ahs.fffbf.cn/666888.Doc
ahs.fffbf.cn/464440.Doc
ahs.fffbf.cn/082424.Doc
ahs.fffbf.cn/446862.Doc
ahs.fffbf.cn/686846.Doc
ahs.fffbf.cn/264664.Doc
ahs.fffbf.cn/488262.Doc
ahs.fffbf.cn/804248.Doc
ahs.fffbf.cn/535397.Doc
ahs.fffbf.cn/444884.Doc
aha.fffbf.cn/002462.Doc
aha.fffbf.cn/424428.Doc
aha.fffbf.cn/426482.Doc
aha.fffbf.cn/886404.Doc
aha.fffbf.cn/642004.Doc
aha.fffbf.cn/204800.Doc
aha.fffbf.cn/751717.Doc
aha.fffbf.cn/880486.Doc
aha.fffbf.cn/022448.Doc
aha.fffbf.cn/462228.Doc
ahp.fffbf.cn/935919.Doc
ahp.fffbf.cn/840040.Doc
ahp.fffbf.cn/428864.Doc
ahp.fffbf.cn/406022.Doc
ahp.fffbf.cn/088482.Doc
ahp.fffbf.cn/662824.Doc
ahp.fffbf.cn/428404.Doc
ahp.fffbf.cn/844406.Doc
ahp.fffbf.cn/606020.Doc
ahp.fffbf.cn/444024.Doc
aho.fffbf.cn/228260.Doc
aho.fffbf.cn/686004.Doc
aho.fffbf.cn/642208.Doc
aho.fffbf.cn/426240.Doc
aho.fffbf.cn/020466.Doc
aho.fffbf.cn/226664.Doc
aho.fffbf.cn/426842.Doc
aho.fffbf.cn/408022.Doc
aho.fffbf.cn/048842.Doc
aho.fffbf.cn/024064.Doc
ahi.fffbf.cn/688204.Doc
ahi.fffbf.cn/660004.Doc
ahi.fffbf.cn/202406.Doc
ahi.fffbf.cn/006482.Doc
ahi.fffbf.cn/646204.Doc
ahi.fffbf.cn/260884.Doc
ahi.fffbf.cn/240686.Doc
ahi.fffbf.cn/426224.Doc
ahi.fffbf.cn/844486.Doc
ahi.fffbf.cn/068402.Doc
ahu.fffbf.cn/173377.Doc
ahu.fffbf.cn/666804.Doc
ahu.fffbf.cn/828048.Doc
ahu.fffbf.cn/804020.Doc
ahu.fffbf.cn/226280.Doc
ahu.fffbf.cn/068228.Doc
ahu.fffbf.cn/393775.Doc
ahu.fffbf.cn/153773.Doc
ahu.fffbf.cn/286888.Doc
ahu.fffbf.cn/424240.Doc
ahy.fffbf.cn/848026.Doc
ahy.fffbf.cn/715757.Doc
ahy.fffbf.cn/088400.Doc
ahy.fffbf.cn/886422.Doc
ahy.fffbf.cn/848688.Doc
ahy.fffbf.cn/226866.Doc
ahy.fffbf.cn/135915.Doc
ahy.fffbf.cn/531715.Doc
ahy.fffbf.cn/240084.Doc
ahy.fffbf.cn/400668.Doc
aht.fffbf.cn/240424.Doc
aht.fffbf.cn/440284.Doc
aht.fffbf.cn/828660.Doc
aht.fffbf.cn/228688.Doc
aht.fffbf.cn/820844.Doc
aht.fffbf.cn/204484.Doc
aht.fffbf.cn/244848.Doc
aht.fffbf.cn/004008.Doc
aht.fffbf.cn/088644.Doc
aht.fffbf.cn/206268.Doc
ahr.fffbf.cn/660420.Doc
ahr.fffbf.cn/068486.Doc
ahr.fffbf.cn/751591.Doc
ahr.fffbf.cn/006822.Doc
ahr.fffbf.cn/353917.Doc
ahr.fffbf.cn/115511.Doc
ahr.fffbf.cn/662468.Doc
ahr.fffbf.cn/084024.Doc
ahr.fffbf.cn/517119.Doc
ahr.fffbf.cn/668826.Doc
ahe.fffbf.cn/268084.Doc
ahe.fffbf.cn/846464.Doc
ahe.fffbf.cn/260282.Doc
ahe.fffbf.cn/224404.Doc
ahe.fffbf.cn/981577.Doc
ahe.fffbf.cn/000602.Doc
ahe.fffbf.cn/822662.Doc
ahe.fffbf.cn/624404.Doc
ahe.fffbf.cn/088066.Doc
ahe.fffbf.cn/446862.Doc
ahw.fffbf.cn/402262.Doc
ahw.fffbf.cn/538434.Doc
ahw.fffbf.cn/688042.Doc
ahw.fffbf.cn/666046.Doc
ahw.fffbf.cn/220068.Doc
ahw.fffbf.cn/224408.Doc
ahw.fffbf.cn/444402.Doc
ahw.fffbf.cn/246866.Doc
ahw.fffbf.cn/828808.Doc
ahw.fffbf.cn/280626.Doc
ahq.fffbf.cn/862200.Doc
ahq.fffbf.cn/804204.Doc
ahq.fffbf.cn/868422.Doc
ahq.fffbf.cn/246648.Doc
ahq.fffbf.cn/248626.Doc
ahq.fffbf.cn/666808.Doc
ahq.fffbf.cn/484044.Doc
ahq.fffbf.cn/860664.Doc
ahq.fffbf.cn/240206.Doc
ahq.fffbf.cn/606048.Doc
agm.fffbf.cn/151599.Doc
agm.fffbf.cn/040802.Doc
agm.fffbf.cn/993571.Doc
agm.fffbf.cn/531935.Doc
agm.fffbf.cn/200826.Doc
agm.fffbf.cn/682606.Doc
agm.fffbf.cn/006400.Doc
agm.fffbf.cn/868286.Doc
agm.fffbf.cn/002604.Doc
agm.fffbf.cn/242488.Doc
agn.fffbf.cn/408028.Doc
agn.fffbf.cn/020288.Doc
agn.fffbf.cn/826604.Doc
