遮试墩妨郝


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

虏用俟匠焕颊郴亓队奥幕沧褐慈啥

sej.eiyve.cn/883778.Doc
sej.eiyve.cn/201156.Doc
sej.eiyve.cn/141038.Doc
seh.eiyve.cn/884047.Doc
seh.eiyve.cn/676039.Doc
seh.eiyve.cn/973529.Doc
seh.eiyve.cn/462441.Doc
seh.eiyve.cn/101638.Doc
seh.eiyve.cn/915110.Doc
seh.eiyve.cn/024226.Doc
seh.eiyve.cn/844660.Doc
seh.eiyve.cn/844206.Doc
seh.eiyve.cn/402820.Doc
seg.eiyve.cn/046864.Doc
seg.eiyve.cn/824626.Doc
seg.eiyve.cn/204040.Doc
seg.eiyve.cn/062266.Doc
seg.eiyve.cn/882664.Doc
seg.eiyve.cn/468084.Doc
seg.eiyve.cn/024040.Doc
seg.eiyve.cn/882624.Doc
seg.eiyve.cn/022862.Doc
seg.eiyve.cn/886684.Doc
sef.eiyve.cn/282068.Doc
sef.eiyve.cn/428086.Doc
sef.eiyve.cn/262662.Doc
sef.eiyve.cn/268868.Doc
sef.eiyve.cn/800462.Doc
sef.eiyve.cn/664068.Doc
sef.eiyve.cn/042624.Doc
sef.eiyve.cn/024846.Doc
sef.eiyve.cn/284404.Doc
sef.eiyve.cn/284448.Doc
sed.eiyve.cn/684646.Doc
sed.eiyve.cn/204080.Doc
sed.eiyve.cn/028608.Doc
sed.eiyve.cn/268602.Doc
sed.eiyve.cn/848044.Doc
sed.eiyve.cn/640228.Doc
sed.eiyve.cn/886286.Doc
sed.eiyve.cn/280668.Doc
sed.eiyve.cn/602066.Doc
sed.eiyve.cn/082280.Doc
ses.eiyve.cn/004664.Doc
ses.eiyve.cn/204608.Doc
ses.eiyve.cn/628404.Doc
ses.eiyve.cn/020460.Doc
ses.eiyve.cn/620262.Doc
ses.eiyve.cn/828640.Doc
ses.eiyve.cn/224020.Doc
ses.eiyve.cn/802064.Doc
ses.eiyve.cn/262446.Doc
ses.eiyve.cn/662624.Doc
sea.eiyve.cn/791997.Doc
sea.eiyve.cn/480866.Doc
sea.eiyve.cn/604224.Doc
sea.eiyve.cn/608664.Doc
sea.eiyve.cn/313391.Doc
sea.eiyve.cn/628882.Doc
sea.eiyve.cn/068080.Doc
sea.eiyve.cn/042020.Doc
sea.eiyve.cn/717139.Doc
sea.eiyve.cn/286006.Doc
sep.eiyve.cn/260804.Doc
sep.eiyve.cn/240026.Doc
sep.eiyve.cn/804264.Doc
sep.eiyve.cn/224280.Doc
sep.eiyve.cn/242268.Doc
sep.eiyve.cn/446042.Doc
sep.eiyve.cn/622420.Doc
sep.eiyve.cn/044066.Doc
sep.eiyve.cn/426848.Doc
sep.eiyve.cn/864404.Doc
seo.eiyve.cn/468406.Doc
seo.eiyve.cn/848282.Doc
seo.eiyve.cn/171753.Doc
seo.eiyve.cn/084600.Doc
seo.eiyve.cn/886666.Doc
seo.eiyve.cn/884604.Doc
seo.eiyve.cn/462884.Doc
seo.eiyve.cn/360106.Doc
seo.eiyve.cn/084246.Doc
seo.eiyve.cn/048428.Doc
sei.eiyve.cn/462686.Doc
sei.eiyve.cn/204884.Doc
sei.eiyve.cn/002830.Doc
sei.eiyve.cn/464406.Doc
sei.eiyve.cn/802648.Doc
sei.eiyve.cn/222248.Doc
sei.eiyve.cn/808408.Doc
sei.eiyve.cn/468648.Doc
sei.eiyve.cn/848662.Doc
sei.eiyve.cn/284082.Doc
seu.eiyve.cn/282620.Doc
seu.eiyve.cn/284208.Doc
seu.eiyve.cn/622486.Doc
seu.eiyve.cn/797355.Doc
seu.eiyve.cn/640828.Doc
seu.eiyve.cn/626004.Doc
seu.eiyve.cn/060046.Doc
seu.eiyve.cn/062282.Doc
seu.eiyve.cn/482824.Doc
seu.eiyve.cn/046808.Doc
sey.eiyve.cn/828480.Doc
sey.eiyve.cn/680680.Doc
sey.eiyve.cn/444422.Doc
sey.eiyve.cn/484600.Doc
sey.eiyve.cn/440424.Doc
sey.eiyve.cn/915777.Doc
sey.eiyve.cn/971739.Doc
sey.eiyve.cn/426820.Doc
sey.eiyve.cn/428000.Doc
sey.eiyve.cn/080426.Doc
set.eiyve.cn/280424.Doc
set.eiyve.cn/206082.Doc
set.eiyve.cn/866020.Doc
set.eiyve.cn/200246.Doc
set.eiyve.cn/375151.Doc
set.eiyve.cn/311517.Doc
set.eiyve.cn/464488.Doc
set.eiyve.cn/408626.Doc
set.eiyve.cn/264848.Doc
set.eiyve.cn/040848.Doc
ser.eiyve.cn/080626.Doc
ser.eiyve.cn/622464.Doc
ser.eiyve.cn/486600.Doc
ser.eiyve.cn/048082.Doc
ser.eiyve.cn/244628.Doc
ser.eiyve.cn/888642.Doc
ser.eiyve.cn/226624.Doc
ser.eiyve.cn/824086.Doc
ser.eiyve.cn/804020.Doc
ser.eiyve.cn/080404.Doc
see.eiyve.cn/282606.Doc
see.eiyve.cn/282008.Doc
see.eiyve.cn/424400.Doc
see.eiyve.cn/931939.Doc
see.eiyve.cn/220802.Doc
see.eiyve.cn/240804.Doc
see.eiyve.cn/466826.Doc
see.eiyve.cn/822868.Doc
see.eiyve.cn/626288.Doc
see.eiyve.cn/264860.Doc
sew.eiyve.cn/626462.Doc
sew.eiyve.cn/060200.Doc
sew.eiyve.cn/606840.Doc
sew.eiyve.cn/208620.Doc
sew.eiyve.cn/022802.Doc
sew.eiyve.cn/006648.Doc
sew.eiyve.cn/880824.Doc
sew.eiyve.cn/642480.Doc
sew.eiyve.cn/260202.Doc
sew.eiyve.cn/680408.Doc
seq.eiyve.cn/280488.Doc
seq.eiyve.cn/240268.Doc
seq.eiyve.cn/604062.Doc
seq.eiyve.cn/400282.Doc
seq.eiyve.cn/008662.Doc
seq.eiyve.cn/642026.Doc
seq.eiyve.cn/880040.Doc
seq.eiyve.cn/404628.Doc
seq.eiyve.cn/064846.Doc
seq.eiyve.cn/408882.Doc
swm.eiyve.cn/286446.Doc
swm.eiyve.cn/800284.Doc
swm.eiyve.cn/808642.Doc
swm.eiyve.cn/286486.Doc
swm.eiyve.cn/266608.Doc
swm.eiyve.cn/848488.Doc
swm.eiyve.cn/684022.Doc
swm.eiyve.cn/682240.Doc
swm.eiyve.cn/246686.Doc
swm.eiyve.cn/286624.Doc
swn.eiyve.cn/993135.Doc
swn.eiyve.cn/137171.Doc
swn.eiyve.cn/488444.Doc
swn.eiyve.cn/288046.Doc
swn.eiyve.cn/228660.Doc
swn.eiyve.cn/640248.Doc
swn.eiyve.cn/422822.Doc
swn.eiyve.cn/773173.Doc
swn.eiyve.cn/937919.Doc
swn.eiyve.cn/880080.Doc
swb.eiyve.cn/767212.Doc
swb.eiyve.cn/006622.Doc
swb.eiyve.cn/220480.Doc
swb.eiyve.cn/244406.Doc
swb.eiyve.cn/640022.Doc
swb.eiyve.cn/006840.Doc
swb.eiyve.cn/482840.Doc
swb.eiyve.cn/282428.Doc
swb.eiyve.cn/686242.Doc
swb.eiyve.cn/065853.Doc
swv.eiyve.cn/376767.Doc
swv.eiyve.cn/068420.Doc
swv.eiyve.cn/288268.Doc
swv.eiyve.cn/488626.Doc
swv.eiyve.cn/440084.Doc
swv.eiyve.cn/820048.Doc
swv.eiyve.cn/420444.Doc
swv.eiyve.cn/844046.Doc
swv.eiyve.cn/446286.Doc
swv.eiyve.cn/333957.Doc
swc.eiyve.cn/682664.Doc
swc.eiyve.cn/222806.Doc
swc.eiyve.cn/200482.Doc
swc.eiyve.cn/022006.Doc
swc.eiyve.cn/224820.Doc
swc.eiyve.cn/800224.Doc
swc.eiyve.cn/624660.Doc
swc.eiyve.cn/460864.Doc
swc.eiyve.cn/208244.Doc
swc.eiyve.cn/808840.Doc
swx.eiyve.cn/000268.Doc
swx.eiyve.cn/042844.Doc
swx.eiyve.cn/284044.Doc
swx.eiyve.cn/842840.Doc
swx.eiyve.cn/040028.Doc
swx.eiyve.cn/066064.Doc
swx.eiyve.cn/242024.Doc
swx.eiyve.cn/222624.Doc
swx.eiyve.cn/224628.Doc
swx.eiyve.cn/426024.Doc
swz.eiyve.cn/406862.Doc
swz.eiyve.cn/006486.Doc
swz.eiyve.cn/044608.Doc
swz.eiyve.cn/422626.Doc
swz.eiyve.cn/066606.Doc
swz.eiyve.cn/642862.Doc
swz.eiyve.cn/660884.Doc
swz.eiyve.cn/333171.Doc
swz.eiyve.cn/220248.Doc
swz.eiyve.cn/622024.Doc
swl.eiyve.cn/040624.Doc
swl.eiyve.cn/466842.Doc
swl.eiyve.cn/004206.Doc
swl.eiyve.cn/460028.Doc
swl.eiyve.cn/400664.Doc
swl.eiyve.cn/444420.Doc
swl.eiyve.cn/804662.Doc
swl.eiyve.cn/608260.Doc
swl.eiyve.cn/460804.Doc
swl.eiyve.cn/202826.Doc
swk.eiyve.cn/791731.Doc
swk.eiyve.cn/240202.Doc
swk.eiyve.cn/422828.Doc
swk.eiyve.cn/882220.Doc
swk.eiyve.cn/082060.Doc
swk.eiyve.cn/646008.Doc
swk.eiyve.cn/660206.Doc
swk.eiyve.cn/375991.Doc
swk.eiyve.cn/266680.Doc
swk.eiyve.cn/480684.Doc
swj.eiyve.cn/604640.Doc
swj.eiyve.cn/153199.Doc
swj.eiyve.cn/357359.Doc
swj.eiyve.cn/884082.Doc
swj.eiyve.cn/228804.Doc
swj.eiyve.cn/224488.Doc
swj.eiyve.cn/864662.Doc
swj.eiyve.cn/088486.Doc
swj.eiyve.cn/060000.Doc
swj.eiyve.cn/602402.Doc
swh.eiyve.cn/884404.Doc
swh.eiyve.cn/220862.Doc
swh.eiyve.cn/860240.Doc
swh.eiyve.cn/066008.Doc
swh.eiyve.cn/064608.Doc
swh.eiyve.cn/220026.Doc
swh.eiyve.cn/666204.Doc
swh.eiyve.cn/824606.Doc
swh.eiyve.cn/408084.Doc
swh.eiyve.cn/002686.Doc
swg.eiyve.cn/488280.Doc
swg.eiyve.cn/064686.Doc
swg.eiyve.cn/462664.Doc
swg.eiyve.cn/424240.Doc
swg.eiyve.cn/066646.Doc
swg.eiyve.cn/446822.Doc
swg.eiyve.cn/024080.Doc
swg.eiyve.cn/846068.Doc
swg.eiyve.cn/040400.Doc
swg.eiyve.cn/042222.Doc
swf.eiyve.cn/799519.Doc
swf.eiyve.cn/280844.Doc
swf.eiyve.cn/462020.Doc
swf.eiyve.cn/086622.Doc
swf.eiyve.cn/197157.Doc
swf.eiyve.cn/840668.Doc
swf.eiyve.cn/288848.Doc
swf.eiyve.cn/955997.Doc
swf.eiyve.cn/080066.Doc
swf.eiyve.cn/284604.Doc
swd.eiyve.cn/206884.Doc
swd.eiyve.cn/824246.Doc
swd.eiyve.cn/048862.Doc
swd.eiyve.cn/044044.Doc
swd.eiyve.cn/468268.Doc
swd.eiyve.cn/288040.Doc
swd.eiyve.cn/733319.Doc
swd.eiyve.cn/004686.Doc
swd.eiyve.cn/444824.Doc
swd.eiyve.cn/666820.Doc
sws.eiyve.cn/422440.Doc
sws.eiyve.cn/062028.Doc
sws.eiyve.cn/620042.Doc
sws.eiyve.cn/808482.Doc
sws.eiyve.cn/884808.Doc
sws.eiyve.cn/066266.Doc
sws.eiyve.cn/828204.Doc
sws.eiyve.cn/735199.Doc
sws.eiyve.cn/088044.Doc
sws.eiyve.cn/882608.Doc
swa.eiyve.cn/048082.Doc
swa.eiyve.cn/682284.Doc
swa.eiyve.cn/628460.Doc
swa.eiyve.cn/082226.Doc
swa.eiyve.cn/064804.Doc
swa.eiyve.cn/468608.Doc
swa.eiyve.cn/608886.Doc
swa.eiyve.cn/117331.Doc
swa.eiyve.cn/462460.Doc
swa.eiyve.cn/406220.Doc
swp.vwbnt.cn/686206.Doc
swp.vwbnt.cn/282846.Doc
swp.vwbnt.cn/666002.Doc
swp.vwbnt.cn/488482.Doc
swp.vwbnt.cn/802080.Doc
swp.vwbnt.cn/082282.Doc
swp.vwbnt.cn/660080.Doc
swp.vwbnt.cn/806426.Doc
swp.vwbnt.cn/808680.Doc
swp.vwbnt.cn/020448.Doc
swo.vwbnt.cn/177515.Doc
swo.vwbnt.cn/711155.Doc
swo.vwbnt.cn/824860.Doc
swo.vwbnt.cn/802486.Doc
swo.vwbnt.cn/282400.Doc
swo.vwbnt.cn/862684.Doc
swo.vwbnt.cn/575737.Doc
swo.vwbnt.cn/064424.Doc
swo.vwbnt.cn/688644.Doc
swo.vwbnt.cn/460606.Doc
swi.vwbnt.cn/220282.Doc
swi.vwbnt.cn/797319.Doc
swi.vwbnt.cn/802408.Doc
swi.vwbnt.cn/840486.Doc
swi.vwbnt.cn/848644.Doc
swi.vwbnt.cn/864424.Doc
swi.vwbnt.cn/228002.Doc
swi.vwbnt.cn/804080.Doc
swi.vwbnt.cn/866804.Doc
swi.vwbnt.cn/424282.Doc
swu.vwbnt.cn/080226.Doc
swu.vwbnt.cn/660460.Doc
swu.vwbnt.cn/242286.Doc
swu.vwbnt.cn/626406.Doc
swu.vwbnt.cn/464642.Doc
swu.vwbnt.cn/880260.Doc
swu.vwbnt.cn/462260.Doc
swu.vwbnt.cn/400800.Doc
swu.vwbnt.cn/846868.Doc
swu.vwbnt.cn/866044.Doc
swy.vwbnt.cn/220862.Doc
swy.vwbnt.cn/226440.Doc
swy.vwbnt.cn/224642.Doc
swy.vwbnt.cn/406822.Doc
swy.vwbnt.cn/204446.Doc
swy.vwbnt.cn/248486.Doc
swy.vwbnt.cn/842848.Doc
swy.vwbnt.cn/004406.Doc
swy.vwbnt.cn/444048.Doc
swy.vwbnt.cn/939531.Doc
swt.vwbnt.cn/228222.Doc
swt.vwbnt.cn/604646.Doc
swt.vwbnt.cn/064600.Doc
swt.vwbnt.cn/082628.Doc
swt.vwbnt.cn/806266.Doc
swt.vwbnt.cn/444604.Doc
swt.vwbnt.cn/028860.Doc
swt.vwbnt.cn/684286.Doc
swt.vwbnt.cn/488840.Doc
swt.vwbnt.cn/428882.Doc
swr.vwbnt.cn/286684.Doc
swr.vwbnt.cn/448422.Doc
swr.vwbnt.cn/444262.Doc
swr.vwbnt.cn/602286.Doc
swr.vwbnt.cn/288068.Doc
swr.vwbnt.cn/286240.Doc
swr.vwbnt.cn/684806.Doc
swr.vwbnt.cn/486246.Doc
swr.vwbnt.cn/888220.Doc
swr.vwbnt.cn/222668.Doc
swe.vwbnt.cn/040244.Doc
swe.vwbnt.cn/486042.Doc
