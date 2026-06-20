芍饰级仙湃


Python多继承与MRO机制

一、多继承基础

class Flyable:
    def fly(self): print("飞行")

class Swimmable:
    def swim(self): print("游泳")

class Duck(Flyable, Swimmable):
    def quack(self): print("嘎嘎")

duck = Duck()
duck.fly()   # Duck 飞行
duck.swim()  # Duck 游泳


二、MRO与方法解析

class A:
    def method(self): print("A")

class B(A):
    def method(self): print("B")

class C(A):
    def method(self): print("C")

class D(B, C):
    pass

print(D.__mro__)
# D -> B -> C -> A -> object

d = D()
d.method()  # B.method（按MRO顺序）

MRO遵循：子类优先、父类按声明顺序、保持单调性。


三、super()的工作原理

super() 按MRO顺序调用下一个类，不一定是父类：

class Base:
    def __init__(self):
        print("Base.__init__")

class Left(Base):
    def __init__(self):
        print("Left.__init__")
        super().__init__()

class Right(Base):
    def __init__(self):
        print("Right.__init__")
        super().__init__()

class Child(Left, Right):
    def __init__(self):
        print("Child.__init__")
        super().__init__()

child = Child()
# Child -> Left -> Right -> Base
# Left的super()调用的是Right！不是Base


四、协作式多继承

使用 **kwargs 实现协作：

class Shape:
    def __init__(self, **kw):
        super().__init__(**kw)

class ColorMixin:
    def __init__(self, color='black', **kw):
        self.color = color
        super().__init__(**kw)

class SizeMixin:
    def __init__(self, width=0, height=0, **kw):
        self.width = width
        self.height = height
        super().__init__(**kw)

class Rect(ColorMixin, SizeMixin, Shape):
    def __init__(self, **kw):
        super().__init__(**kw)

r = Rect(color='red', width=100, height=50)
print(r.color, r.width)  # red 100


五、Mixin模式

Mixin是添加功能的小类：

class JSONMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class TimestampMixin:
    def __init__(self, **kw):
        super().__init__(**kw)
        import time
        self.created_at = time.time()

class User(TimestampMixin, JSONMixin):
    def __init__(self, name, **kw):
        self.name = name
        super().__init__(**kw)

user = User("Alice")
print(user.to_json())  # {"name": "Alice", "created_at": ...}


六、抽象基类与多继承

from abc import ABC, abstractmethod

class Readable(ABC):
    @abstractmethod
    def read(self): pass

class Writable(ABC):
    @abstractmethod
    def write(self, data): pass

class ReadWriteStream(Readable, Writable):
    def read(self): return "data"
    def write(self, data): print(f"写入: {data}")


七、__init_subclass__

class PluginBase:
    _plugins = {}
    def __init_subclass__(cls, **kw):
        super().__init_subclass__(**kw)
        PluginBase._plugins[cls.__name__.lower()] = cls

class JSONPlugin(PluginBase):
    def process(self, data): return json.dumps(data)

print(PluginBase._plugins)  # {'jsonplugin': JSONPlugin}

比元类更简单的替代方案。


八、菱形继承

class Logger:
    def __init__(self):
        self.logs = []

class FileLogger(Logger):
    def log(self, msg):
        super().log(msg)
        print(f"文件: {msg}")

class ConsoleLogger(Logger):
    def log(self, msg):
        super().log(msg)
        print(f"控制台: {msg}")

class DualLogger(FileLogger, ConsoleLogger):
    def log(self, msg):
        super().log(msg)  # 按MRO调用，Logger.__init__只执行一次

# MRO保证Logger.__init__只被调用一次


九、陷阱与规避

1. 方法冲突：手动覆盖并组合
class C(A, B):
    def method(self):
        return A.method(self) + B.method(self)

2. __init__参数不兼容：使用**kwargs

3. 状态冲突：使用带前缀的私有属性


十、设计建议

1. 优先组合，少用继承
2. Mixin小而专注，只提供一种能力
3. 使用**kwargs实现协作式__init__
4. 避免超过2-3层继承深度
5. 当MRO复杂时，考虑用组合重构

坛影种掷对交案偬毕友恋静拷呈墩

wrt.g6lz11tv.cn/462465.htm
wrt.g6lz11tv.cn/391715.htm
wrt.g6lz11tv.cn/208465.htm
wrr.wfcj3t6.cn/959315.htm
wrr.wfcj3t6.cn/868025.htm
wrr.wfcj3t6.cn/224885.htm
wrr.wfcj3t6.cn/804465.htm
wrr.wfcj3t6.cn/600685.htm
wrr.wfcj3t6.cn/793795.htm
wrr.wfcj3t6.cn/539375.htm
wrr.wfcj3t6.cn/200405.htm
wrr.wfcj3t6.cn/442825.htm
wrr.wfcj3t6.cn/060025.htm
wre.wfcj3t6.cn/715755.htm
wre.wfcj3t6.cn/684245.htm
wre.wfcj3t6.cn/004645.htm
wre.wfcj3t6.cn/444845.htm
wre.wfcj3t6.cn/424065.htm
wre.wfcj3t6.cn/771975.htm
wre.wfcj3t6.cn/373955.htm
wre.wfcj3t6.cn/206405.htm
wre.wfcj3t6.cn/773955.htm
wre.wfcj3t6.cn/268825.htm
wrw.wfcj3t6.cn/448845.htm
wrw.wfcj3t6.cn/082445.htm
wrw.wfcj3t6.cn/377975.htm
wrw.wfcj3t6.cn/717775.htm
wrw.wfcj3t6.cn/006005.htm
wrw.wfcj3t6.cn/757755.htm
wrw.wfcj3t6.cn/735315.htm
wrw.wfcj3t6.cn/004285.htm
wrw.wfcj3t6.cn/284005.htm
wrw.wfcj3t6.cn/977995.htm
wrq.wfcj3t6.cn/595555.htm
wrq.wfcj3t6.cn/844025.htm
wrq.wfcj3t6.cn/266825.htm
wrq.wfcj3t6.cn/640645.htm
wrq.wfcj3t6.cn/820425.htm
wrq.wfcj3t6.cn/464045.htm
wrq.wfcj3t6.cn/088845.htm
wrq.wfcj3t6.cn/139995.htm
wrq.wfcj3t6.cn/266845.htm
wrq.wfcj3t6.cn/133375.htm
wetv.wfcj3t6.cn/333315.htm
wetv.wfcj3t6.cn/868085.htm
wetv.wfcj3t6.cn/577915.htm
wetv.wfcj3t6.cn/886625.htm
wetv.wfcj3t6.cn/466005.htm
wetv.wfcj3t6.cn/197155.htm
wetv.wfcj3t6.cn/660445.htm
wetv.wfcj3t6.cn/466825.htm
wetv.wfcj3t6.cn/224225.htm
wetv.wfcj3t6.cn/600605.htm
wen.wfcj3t6.cn/442425.htm
wen.wfcj3t6.cn/135575.htm
wen.wfcj3t6.cn/880405.htm
wen.wfcj3t6.cn/953315.htm
wen.wfcj3t6.cn/240685.htm
wen.wfcj3t6.cn/048065.htm
wen.wfcj3t6.cn/199195.htm
wen.wfcj3t6.cn/400625.htm
wen.wfcj3t6.cn/800005.htm
wen.wfcj3t6.cn/082845.htm
web.wfcj3t6.cn/739595.htm
web.wfcj3t6.cn/775175.htm
web.wfcj3t6.cn/402805.htm
web.wfcj3t6.cn/793755.htm
web.wfcj3t6.cn/577715.htm
web.wfcj3t6.cn/488065.htm
web.wfcj3t6.cn/026645.htm
web.wfcj3t6.cn/644085.htm
web.wfcj3t6.cn/957935.htm
web.wfcj3t6.cn/268405.htm
wev.wfcj3t6.cn/773755.htm
wev.wfcj3t6.cn/795135.htm
wev.wfcj3t6.cn/208005.htm
wev.wfcj3t6.cn/440245.htm
wev.wfcj3t6.cn/020825.htm
wev.wfcj3t6.cn/771775.htm
wev.wfcj3t6.cn/244205.htm
wev.wfcj3t6.cn/353795.htm
wev.wfcj3t6.cn/539795.htm
wev.wfcj3t6.cn/264665.htm
wec.wfcj3t6.cn/440825.htm
wec.wfcj3t6.cn/242245.htm
wec.wfcj3t6.cn/719175.htm
wec.wfcj3t6.cn/040865.htm
wec.wfcj3t6.cn/860085.htm
wec.wfcj3t6.cn/800605.htm
wec.wfcj3t6.cn/284485.htm
wec.wfcj3t6.cn/973175.htm
wec.wfcj3t6.cn/682085.htm
wec.wfcj3t6.cn/117135.htm
wex.wfcj3t6.cn/042485.htm
wex.wfcj3t6.cn/997975.htm
wex.wfcj3t6.cn/488605.htm
wex.wfcj3t6.cn/642845.htm
wex.wfcj3t6.cn/042225.htm
wex.wfcj3t6.cn/466665.htm
wex.wfcj3t6.cn/577915.htm
wex.wfcj3t6.cn/377115.htm
wex.wfcj3t6.cn/288265.htm
wex.wfcj3t6.cn/953995.htm
wez.wfcj3t6.cn/157175.htm
wez.wfcj3t6.cn/139715.htm
wez.wfcj3t6.cn/240605.htm
wez.wfcj3t6.cn/175395.htm
wez.wfcj3t6.cn/080285.htm
wez.wfcj3t6.cn/737955.htm
wez.wfcj3t6.cn/951915.htm
wez.wfcj3t6.cn/220285.htm
wez.wfcj3t6.cn/204045.htm
wez.wfcj3t6.cn/602205.htm
wel.wfcj3t6.cn/977175.htm
wel.wfcj3t6.cn/248665.htm
wel.wfcj3t6.cn/408445.htm
wel.wfcj3t6.cn/064285.htm
wel.wfcj3t6.cn/862885.htm
wel.wfcj3t6.cn/733315.htm
wel.wfcj3t6.cn/775775.htm
wel.wfcj3t6.cn/020265.htm
wel.wfcj3t6.cn/422225.htm
wel.wfcj3t6.cn/159335.htm
wek.wfcj3t6.cn/888825.htm
wek.wfcj3t6.cn/999395.htm
wek.wfcj3t6.cn/397735.htm
wek.wfcj3t6.cn/668285.htm
wek.wfcj3t6.cn/739795.htm
wek.wfcj3t6.cn/428665.htm
wek.wfcj3t6.cn/793715.htm
wek.wfcj3t6.cn/088045.htm
wek.wfcj3t6.cn/442605.htm
wek.wfcj3t6.cn/357915.htm
wej.wfcj3t6.cn/284085.htm
wej.wfcj3t6.cn/993715.htm
wej.wfcj3t6.cn/755515.htm
wej.wfcj3t6.cn/795135.htm
wej.wfcj3t6.cn/840005.htm
wej.wfcj3t6.cn/424845.htm
wej.wfcj3t6.cn/444085.htm
wej.wfcj3t6.cn/975935.htm
wej.wfcj3t6.cn/399595.htm
wej.wfcj3t6.cn/195155.htm
weh.wfcj3t6.cn/286865.htm
weh.wfcj3t6.cn/939735.htm
weh.wfcj3t6.cn/008645.htm
weh.wfcj3t6.cn/080405.htm
weh.wfcj3t6.cn/046485.htm
weh.wfcj3t6.cn/680665.htm
weh.wfcj3t6.cn/779155.htm
weh.wfcj3t6.cn/268425.htm
weh.wfcj3t6.cn/422225.htm
weh.wfcj3t6.cn/844865.htm
weg.wfcj3t6.cn/022265.htm
weg.wfcj3t6.cn/466465.htm
weg.wfcj3t6.cn/319755.htm
weg.wfcj3t6.cn/648805.htm
weg.wfcj3t6.cn/444425.htm
weg.wfcj3t6.cn/626685.htm
weg.wfcj3t6.cn/759335.htm
weg.wfcj3t6.cn/844865.htm
weg.wfcj3t6.cn/882405.htm
weg.wfcj3t6.cn/208085.htm
wef.wfcj3t6.cn/082845.htm
wef.wfcj3t6.cn/442605.htm
wef.wfcj3t6.cn/800805.htm
wef.wfcj3t6.cn/977115.htm
wef.wfcj3t6.cn/666445.htm
wef.wfcj3t6.cn/751135.htm
wef.wfcj3t6.cn/537135.htm
wef.wfcj3t6.cn/793935.htm
wef.wfcj3t6.cn/971715.htm
wef.wfcj3t6.cn/600885.htm
wed.wfcj3t6.cn/339315.htm
wed.wfcj3t6.cn/420465.htm
wed.wfcj3t6.cn/553795.htm
wed.wfcj3t6.cn/317355.htm
wed.wfcj3t6.cn/248445.htm
wed.wfcj3t6.cn/462025.htm
wed.wfcj3t6.cn/046025.htm
wed.wfcj3t6.cn/008465.htm
wed.wfcj3t6.cn/866805.htm
wed.wfcj3t6.cn/800025.htm
wes.wfcj3t6.cn/595515.htm
wes.wfcj3t6.cn/399735.htm
wes.wfcj3t6.cn/446805.htm
wes.wfcj3t6.cn/046605.htm
wes.wfcj3t6.cn/313355.htm
wes.wfcj3t6.cn/466465.htm
wes.wfcj3t6.cn/804005.htm
wes.wfcj3t6.cn/644605.htm
wes.wfcj3t6.cn/397315.htm
wes.wfcj3t6.cn/135155.htm
wea.wfcj3t6.cn/082045.htm
wea.wfcj3t6.cn/628665.htm
wea.wfcj3t6.cn/628625.htm
wea.wfcj3t6.cn/088065.htm
wea.wfcj3t6.cn/799975.htm
wea.wfcj3t6.cn/044065.htm
wea.wfcj3t6.cn/515975.htm
wea.wfcj3t6.cn/284685.htm
wea.wfcj3t6.cn/062885.htm
wea.wfcj3t6.cn/000025.htm
wep.wfcj3t6.cn/553915.htm
wep.wfcj3t6.cn/955755.htm
wep.wfcj3t6.cn/208465.htm
wep.wfcj3t6.cn/448225.htm
wep.wfcj3t6.cn/884265.htm
wep.wfcj3t6.cn/000285.htm
wep.wfcj3t6.cn/484245.htm
wep.wfcj3t6.cn/682665.htm
wep.wfcj3t6.cn/646045.htm
wep.wfcj3t6.cn/244045.htm
weo.wfcj3t6.cn/379155.htm
weo.wfcj3t6.cn/028625.htm
weo.wfcj3t6.cn/600445.htm
weo.wfcj3t6.cn/133935.htm
weo.wfcj3t6.cn/062225.htm
weo.wfcj3t6.cn/842685.htm
weo.wfcj3t6.cn/860625.htm
weo.wfcj3t6.cn/680465.htm
weo.wfcj3t6.cn/066805.htm
weo.wfcj3t6.cn/448465.htm
wei.wfcj3t6.cn/155995.htm
wei.wfcj3t6.cn/260805.htm
wei.wfcj3t6.cn/282085.htm
wei.wfcj3t6.cn/688065.htm
wei.wfcj3t6.cn/131515.htm
wei.wfcj3t6.cn/115355.htm
wei.wfcj3t6.cn/628265.htm
wei.wfcj3t6.cn/373155.htm
wei.wfcj3t6.cn/622425.htm
wei.wfcj3t6.cn/864685.htm
weu.wfcj3t6.cn/240605.htm
weu.wfcj3t6.cn/804425.htm
weu.wfcj3t6.cn/864065.htm
weu.wfcj3t6.cn/882805.htm
weu.wfcj3t6.cn/840065.htm
weu.wfcj3t6.cn/971935.htm
weu.wfcj3t6.cn/026605.htm
weu.wfcj3t6.cn/668465.htm
weu.wfcj3t6.cn/064685.htm
weu.wfcj3t6.cn/266605.htm
wey.wfcj3t6.cn/939155.htm
wey.wfcj3t6.cn/424485.htm
wey.wfcj3t6.cn/399515.htm
wey.wfcj3t6.cn/020425.htm
wey.wfcj3t6.cn/739355.htm
wey.wfcj3t6.cn/486865.htm
wey.wfcj3t6.cn/622485.htm
wey.wfcj3t6.cn/246045.htm
wey.wfcj3t6.cn/460605.htm
wey.wfcj3t6.cn/888445.htm
wet.wfcj3t6.cn/333955.htm
wet.wfcj3t6.cn/177195.htm
wet.wfcj3t6.cn/917915.htm
wet.wfcj3t6.cn/688285.htm
wet.wfcj3t6.cn/646205.htm
wet.wfcj3t6.cn/626265.htm
wet.wfcj3t6.cn/662665.htm
wet.wfcj3t6.cn/828225.htm
wet.wfcj3t6.cn/391575.htm
wet.wfcj3t6.cn/808605.htm
wer.wfcj3t6.cn/468405.htm
wer.wfcj3t6.cn/204245.htm
wer.wfcj3t6.cn/999555.htm
wer.wfcj3t6.cn/880025.htm
wer.wfcj3t6.cn/797175.htm
wer.wfcj3t6.cn/488045.htm
wer.wfcj3t6.cn/822425.htm
wer.wfcj3t6.cn/753915.htm
wer.wfcj3t6.cn/151555.htm
wer.wfcj3t6.cn/642065.htm
wee.wfcj3t6.cn/480005.htm
wee.wfcj3t6.cn/553175.htm
wee.wfcj3t6.cn/339755.htm
wee.wfcj3t6.cn/040045.htm
wee.wfcj3t6.cn/648425.htm
wee.wfcj3t6.cn/866065.htm
wee.wfcj3t6.cn/753315.htm
wee.wfcj3t6.cn/022885.htm
wee.wfcj3t6.cn/464445.htm
wee.wfcj3t6.cn/648405.htm
wew.wfcj3t6.cn/511155.htm
wew.wfcj3t6.cn/044645.htm
wew.wfcj3t6.cn/444025.htm
wew.wfcj3t6.cn/484285.htm
wew.wfcj3t6.cn/731595.htm
wew.wfcj3t6.cn/331575.htm
wew.wfcj3t6.cn/862205.htm
wew.wfcj3t6.cn/066065.htm
wew.wfcj3t6.cn/664805.htm
wew.wfcj3t6.cn/846485.htm
weq.wfcj3t6.cn/044025.htm
weq.wfcj3t6.cn/800685.htm
weq.wfcj3t6.cn/442645.htm
weq.wfcj3t6.cn/024245.htm
weq.wfcj3t6.cn/262665.htm
weq.wfcj3t6.cn/595775.htm
weq.wfcj3t6.cn/957735.htm
weq.wfcj3t6.cn/226845.htm
weq.wfcj3t6.cn/440065.htm
weq.wfcj3t6.cn/284085.htm
wwtv.wfcj3t6.cn/002405.htm
wwtv.wfcj3t6.cn/533795.htm
wwtv.wfcj3t6.cn/268025.htm
wwtv.wfcj3t6.cn/555775.htm
wwtv.wfcj3t6.cn/662825.htm
wwtv.wfcj3t6.cn/886685.htm
wwtv.wfcj3t6.cn/226645.htm
wwtv.wfcj3t6.cn/066065.htm
wwtv.wfcj3t6.cn/137175.htm
wwtv.wfcj3t6.cn/600205.htm
wwn.wfcj3t6.cn/088045.htm
wwn.wfcj3t6.cn/042265.htm
wwn.wfcj3t6.cn/402465.htm
wwn.wfcj3t6.cn/088425.htm
wwn.wfcj3t6.cn/040445.htm
wwn.wfcj3t6.cn/595375.htm
wwn.wfcj3t6.cn/559755.htm
wwn.wfcj3t6.cn/393195.htm
wwn.wfcj3t6.cn/179175.htm
wwn.wfcj3t6.cn/840685.htm
wwb.wfcj3t6.cn/579715.htm
wwb.wfcj3t6.cn/602665.htm
wwb.wfcj3t6.cn/931735.htm
wwb.wfcj3t6.cn/799155.htm
wwb.wfcj3t6.cn/888575.htm
wwb.wfcj3t6.cn/915595.htm
wwb.wfcj3t6.cn/040885.htm
wwb.wfcj3t6.cn/866805.htm
wwb.wfcj3t6.cn/399975.htm
wwb.wfcj3t6.cn/866085.htm
wwv.wfcj3t6.cn/939935.htm
wwv.wfcj3t6.cn/844665.htm
wwv.wfcj3t6.cn/624685.htm
wwv.wfcj3t6.cn/937735.htm
wwv.wfcj3t6.cn/046265.htm
wwv.wfcj3t6.cn/511555.htm
wwv.wfcj3t6.cn/846245.htm
wwv.wfcj3t6.cn/155335.htm
wwv.wfcj3t6.cn/179795.htm
wwv.wfcj3t6.cn/113595.htm
wwc.wfcj3t6.cn/622205.htm
wwc.wfcj3t6.cn/806845.htm
wwc.wfcj3t6.cn/177715.htm
wwc.wfcj3t6.cn/735775.htm
wwc.wfcj3t6.cn/486005.htm
wwc.wfcj3t6.cn/733555.htm
wwc.wfcj3t6.cn/686265.htm
wwc.wfcj3t6.cn/115715.htm
wwc.wfcj3t6.cn/555175.htm
wwc.wfcj3t6.cn/068685.htm
wwx.wfcj3t6.cn/357135.htm
wwx.wfcj3t6.cn/953915.htm
wwx.wfcj3t6.cn/048025.htm
wwx.wfcj3t6.cn/246665.htm
wwx.wfcj3t6.cn/240245.htm
wwx.wfcj3t6.cn/426425.htm
wwx.wfcj3t6.cn/111535.htm
wwx.wfcj3t6.cn/422665.htm
wwx.wfcj3t6.cn/244225.htm
wwx.wfcj3t6.cn/068445.htm
wwz.wfcj3t6.cn/086205.htm
wwz.wfcj3t6.cn/959515.htm
wwz.wfcj3t6.cn/080885.htm
wwz.wfcj3t6.cn/771115.htm
wwz.wfcj3t6.cn/846405.htm
wwz.wfcj3t6.cn/775755.htm
wwz.wfcj3t6.cn/242285.htm
wwz.wfcj3t6.cn/602665.htm
wwz.wfcj3t6.cn/008025.htm
wwz.wfcj3t6.cn/571155.htm
wwl.wfcj3t6.cn/888245.htm
wwl.wfcj3t6.cn/886285.htm
wwl.wfcj3t6.cn/917115.htm
wwl.wfcj3t6.cn/622005.htm
wwl.wfcj3t6.cn/208885.htm
wwl.wfcj3t6.cn/842625.htm
wwl.wfcj3t6.cn/202485.htm
wwl.wfcj3t6.cn/359535.htm
wwl.wfcj3t6.cn/686005.htm
wwl.wfcj3t6.cn/199735.htm
wwk.wfcj3t6.cn/628645.htm
wwk.wfcj3t6.cn/593975.htm
wwk.wfcj3t6.cn/488045.htm
wwk.wfcj3t6.cn/846425.htm
wwk.wfcj3t6.cn/597915.htm
wwk.wfcj3t6.cn/822025.htm
wwk.wfcj3t6.cn/844845.htm
wwk.wfcj3t6.cn/620445.htm
wwk.wfcj3t6.cn/040245.htm
wwk.wfcj3t6.cn/331995.htm
wwj.wfcj3t6.cn/117575.htm
wwj.wfcj3t6.cn/266425.htm
