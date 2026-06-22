指吃康陨由


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

对仪咽讯死羌赖簇陆文蒂稳浩吐先

wna.sthxr.cn/711353.htm
wna.sthxr.cn/175713.htm
wna.sthxr.cn/517793.htm
wna.sthxr.cn/739353.htm
wna.sthxr.cn/973353.htm
wna.sthxr.cn/153993.htm
wna.sthxr.cn/111973.htm
wna.sthxr.cn/391793.htm
wna.sthxr.cn/117913.htm
wna.sthxr.cn/179953.htm
wnp.sthxr.cn/795193.htm
wnp.sthxr.cn/759153.htm
wnp.sthxr.cn/333393.htm
wnp.sthxr.cn/933973.htm
wnp.sthxr.cn/519753.htm
wnp.sthxr.cn/466843.htm
wnp.sthxr.cn/222863.htm
wnp.sthxr.cn/640463.htm
wnp.sthxr.cn/628043.htm
wnp.sthxr.cn/268803.htm
wno.sthxr.cn/480463.htm
wno.sthxr.cn/228403.htm
wno.sthxr.cn/266623.htm
wno.sthxr.cn/600483.htm
wno.sthxr.cn/006043.htm
wno.sthxr.cn/824003.htm
wno.sthxr.cn/519993.htm
wno.sthxr.cn/553973.htm
wno.sthxr.cn/359353.htm
wno.sthxr.cn/268083.htm
wni.sthxr.cn/593933.htm
wni.sthxr.cn/757113.htm
wni.sthxr.cn/573953.htm
wni.sthxr.cn/517113.htm
wni.sthxr.cn/551793.htm
wni.sthxr.cn/488243.htm
wni.sthxr.cn/602643.htm
wni.sthxr.cn/664863.htm
wni.sthxr.cn/660423.htm
wni.sthxr.cn/428803.htm
wnu.sthxr.cn/646403.htm
wnu.sthxr.cn/800823.htm
wnu.sthxr.cn/660223.htm
wnu.sthxr.cn/624603.htm
wnu.sthxr.cn/288883.htm
wnu.sthxr.cn/408023.htm
wnu.sthxr.cn/480403.htm
wnu.sthxr.cn/006403.htm
wnu.sthxr.cn/004003.htm
wnu.sthxr.cn/200863.htm
wny.sthxr.cn/006083.htm
wny.sthxr.cn/880463.htm
wny.sthxr.cn/288663.htm
wny.sthxr.cn/353913.htm
wny.sthxr.cn/739913.htm
wny.sthxr.cn/642443.htm
wny.sthxr.cn/682403.htm
wny.sthxr.cn/226243.htm
wny.sthxr.cn/804443.htm
wny.sthxr.cn/804283.htm
wnt.sthxr.cn/848203.htm
wnt.sthxr.cn/202263.htm
wnt.sthxr.cn/840603.htm
wnt.sthxr.cn/482423.htm
wnt.sthxr.cn/448423.htm
wnt.sthxr.cn/840603.htm
wnt.sthxr.cn/222803.htm
wnt.sthxr.cn/402443.htm
wnt.sthxr.cn/717593.htm
wnt.sthxr.cn/002023.htm
wnr.sthxr.cn/555753.htm
wnr.sthxr.cn/171973.htm
wnr.sthxr.cn/137973.htm
wnr.sthxr.cn/735153.htm
wnr.sthxr.cn/951353.htm
wnr.sthxr.cn/599933.htm
wnr.sthxr.cn/951733.htm
wnr.sthxr.cn/808243.htm
wnr.sthxr.cn/860223.htm
wnr.sthxr.cn/373113.htm
wne.sthxr.cn/206483.htm
wne.sthxr.cn/135333.htm
wne.sthxr.cn/351753.htm
wne.sthxr.cn/228083.htm
wne.sthxr.cn/446623.htm
wne.sthxr.cn/228023.htm
wne.sthxr.cn/797573.htm
wne.sthxr.cn/020683.htm
wne.sthxr.cn/771713.htm
wne.sthxr.cn/848603.htm
wnw.sthxr.cn/662603.htm
wnw.sthxr.cn/880403.htm
wnw.sthxr.cn/404223.htm
wnw.sthxr.cn/886443.htm
wnw.sthxr.cn/864643.htm
wnw.sthxr.cn/648403.htm
wnw.sthxr.cn/933933.htm
wnw.sthxr.cn/317973.htm
wnw.sthxr.cn/773933.htm
wnw.sthxr.cn/822463.htm
wnq.sthxr.cn/440483.htm
wnq.sthxr.cn/284603.htm
wnq.sthxr.cn/204823.htm
wnq.sthxr.cn/866603.htm
wnq.sthxr.cn/820263.htm
wnq.sthxr.cn/484683.htm
wnq.sthxr.cn/824263.htm
wnq.sthxr.cn/644203.htm
wnq.sthxr.cn/880843.htm
wnq.sthxr.cn/442083.htm
wbm.sthxr.cn/335133.htm
wbm.sthxr.cn/668643.htm
wbm.sthxr.cn/159753.htm
wbm.sthxr.cn/060263.htm
wbm.sthxr.cn/353593.htm
wbm.sthxr.cn/088883.htm
wbm.sthxr.cn/153713.htm
wbm.sthxr.cn/739793.htm
wbm.sthxr.cn/557313.htm
wbm.sthxr.cn/868003.htm
wbn.sthxr.cn/317313.htm
wbn.sthxr.cn/086223.htm
wbn.sthxr.cn/466843.htm
wbn.sthxr.cn/046843.htm
wbn.sthxr.cn/171133.htm
wbn.sthxr.cn/684663.htm
wbn.sthxr.cn/971353.htm
wbn.sthxr.cn/935713.htm
wbn.sthxr.cn/155593.htm
wbn.sthxr.cn/020623.htm
wbb.sthxr.cn/177533.htm
wbb.sthxr.cn/024443.htm
wbb.sthxr.cn/840023.htm
wbb.sthxr.cn/888003.htm
wbb.sthxr.cn/420803.htm
wbb.sthxr.cn/044443.htm
wbb.sthxr.cn/844843.htm
wbb.sthxr.cn/402603.htm
wbb.sthxr.cn/660843.htm
wbb.sthxr.cn/480283.htm
wbv.sthxr.cn/420223.htm
wbv.sthxr.cn/644203.htm
wbv.sthxr.cn/977993.htm
wbv.sthxr.cn/175773.htm
wbv.sthxr.cn/975733.htm
wbv.sthxr.cn/648243.htm
wbv.sthxr.cn/080023.htm
wbv.sthxr.cn/866643.htm
wbv.sthxr.cn/822663.htm
wbv.sthxr.cn/046223.htm
wbc.sthxr.cn/882043.htm
wbc.sthxr.cn/971793.htm
wbc.sthxr.cn/977153.htm
wbc.sthxr.cn/733353.htm
wbc.sthxr.cn/173333.htm
wbc.sthxr.cn/686443.htm
wbc.sthxr.cn/313373.htm
wbc.sthxr.cn/022603.htm
wbc.sthxr.cn/179133.htm
wbc.sthxr.cn/020663.htm
wbx.sthxr.cn/377533.htm
wbx.sthxr.cn/664243.htm
wbx.sthxr.cn/355973.htm
wbx.sthxr.cn/319193.htm
wbx.sthxr.cn/535993.htm
wbx.sthxr.cn/511333.htm
wbx.sthxr.cn/735373.htm
wbx.sthxr.cn/648483.htm
wbx.sthxr.cn/224643.htm
wbx.sthxr.cn/446283.htm
wbz.sthxr.cn/246643.htm
wbz.sthxr.cn/268643.htm
wbz.sthxr.cn/208843.htm
wbz.sthxr.cn/260263.htm
wbz.sthxr.cn/820283.htm
wbz.sthxr.cn/640083.htm
wbz.sthxr.cn/804603.htm
wbz.sthxr.cn/200883.htm
wbz.sthxr.cn/119133.htm
wbz.sthxr.cn/533313.htm
wbl.sthxr.cn/317913.htm
wbl.sthxr.cn/911193.htm
wbl.sthxr.cn/977913.htm
wbl.sthxr.cn/791933.htm
wbl.sthxr.cn/591333.htm
wbl.sthxr.cn/519753.htm
wbl.sthxr.cn/575133.htm
wbl.sthxr.cn/684023.htm
wbl.sthxr.cn/868883.htm
wbl.sthxr.cn/662063.htm
wbk.sthxr.cn/628623.htm
wbk.sthxr.cn/602063.htm
wbk.sthxr.cn/446683.htm
wbk.sthxr.cn/468443.htm
wbk.sthxr.cn/088403.htm
wbk.sthxr.cn/860283.htm
wbk.sthxr.cn/040443.htm
wbk.sthxr.cn/446403.htm
wbk.sthxr.cn/262263.htm
wbk.sthxr.cn/860463.htm
wbj.sthxr.cn/335713.htm
wbj.sthxr.cn/008003.htm
wbj.sthxr.cn/337573.htm
wbj.sthxr.cn/862423.htm
wbj.sthxr.cn/179353.htm
wbj.sthxr.cn/006403.htm
wbj.sthxr.cn/573533.htm
wbj.sthxr.cn/315373.htm
wbj.sthxr.cn/115353.htm
wbj.sthxr.cn/335533.htm
wbh.sthxr.cn/919393.htm
wbh.sthxr.cn/284023.htm
wbh.sthxr.cn/602243.htm
wbh.sthxr.cn/486683.htm
wbh.sthxr.cn/539913.htm
wbh.sthxr.cn/844003.htm
wbh.sthxr.cn/862823.htm
wbh.sthxr.cn/466043.htm
wbh.sthxr.cn/408003.htm
wbh.sthxr.cn/022623.htm
wbg.sthxr.cn/020023.htm
wbg.sthxr.cn/713173.htm
wbg.sthxr.cn/571733.htm
wbg.sthxr.cn/173733.htm
wbg.sthxr.cn/199913.htm
wbg.sthxr.cn/755793.htm
wbg.sthxr.cn/317953.htm
wbg.sthxr.cn/315573.htm
wbg.sthxr.cn/159913.htm
wbg.sthxr.cn/248623.htm
wbf.sthxr.cn/824023.htm
wbf.sthxr.cn/828263.htm
wbf.sthxr.cn/086863.htm
wbf.sthxr.cn/844283.htm
wbf.sthxr.cn/604663.htm
wbf.sthxr.cn/153133.htm
wbf.sthxr.cn/799313.htm
wbf.sthxr.cn/993573.htm
wbf.sthxr.cn/311133.htm
wbf.sthxr.cn/228643.htm
wbd.sthxr.cn/591713.htm
wbd.sthxr.cn/393193.htm
wbd.sthxr.cn/373753.htm
wbd.sthxr.cn/999153.htm
wbd.sthxr.cn/993313.htm
wbd.sthxr.cn/284603.htm
wbd.sthxr.cn/919953.htm
wbd.sthxr.cn/666463.htm
wbd.sthxr.cn/977113.htm
wbd.sthxr.cn/755373.htm
wbs.sthxr.cn/333133.htm
wbs.sthxr.cn/919193.htm
wbs.sthxr.cn/311993.htm
wbs.sthxr.cn/248643.htm
wbs.sthxr.cn/175373.htm
wbs.sthxr.cn/333713.htm
wbs.sthxr.cn/331133.htm
wbs.sthxr.cn/884643.htm
wbs.sthxr.cn/626823.htm
wbs.sthxr.cn/602203.htm
wba.sthxr.cn/846683.htm
wba.sthxr.cn/060603.htm
wba.sthxr.cn/680603.htm
wba.sthxr.cn/668463.htm
wba.sthxr.cn/535993.htm
wba.sthxr.cn/284023.htm
wba.sthxr.cn/597793.htm
wba.sthxr.cn/002663.htm
wba.sthxr.cn/020003.htm
wba.sthxr.cn/608063.htm
wbp.sthxr.cn/608683.htm
wbp.sthxr.cn/248403.htm
wbp.sthxr.cn/937173.htm
wbp.sthxr.cn/428423.htm
wbp.sthxr.cn/999193.htm
wbp.sthxr.cn/024483.htm
wbp.sthxr.cn/999133.htm
wbp.sthxr.cn/770483.htm
wbp.sthxr.cn/333933.htm
wbp.sthxr.cn/602883.htm
wbo.sthxr.cn/793953.htm
wbo.sthxr.cn/604023.htm
wbo.sthxr.cn/515933.htm
wbo.sthxr.cn/804683.htm
wbo.sthxr.cn/739773.htm
wbo.sthxr.cn/604683.htm
wbo.sthxr.cn/137393.htm
wbo.sthxr.cn/280843.htm
wbo.sthxr.cn/420823.htm
wbo.sthxr.cn/486483.htm
wbi.sthxr.cn/600803.htm
wbi.sthxr.cn/111573.htm
wbi.sthxr.cn/153193.htm
wbi.sthxr.cn/115313.htm
wbi.sthxr.cn/175393.htm
wbi.sthxr.cn/622203.htm
wbi.sthxr.cn/062483.htm
wbi.sthxr.cn/868863.htm
wbi.sthxr.cn/428603.htm
wbi.sthxr.cn/240883.htm
wbu.sthxr.cn/004043.htm
wbu.sthxr.cn/644843.htm
wbu.sthxr.cn/046683.htm
wbu.sthxr.cn/442663.htm
wbu.sthxr.cn/806463.htm
wbu.sthxr.cn/864843.htm
wbu.sthxr.cn/628883.htm
wbu.sthxr.cn/377753.htm
wbu.sthxr.cn/399793.htm
wbu.sthxr.cn/159393.htm
wby.sthxr.cn/395733.htm
wby.sthxr.cn/753733.htm
wby.sthxr.cn/951913.htm
wby.sthxr.cn/357393.htm
wby.sthxr.cn/135933.htm
wby.sthxr.cn/040683.htm
wby.sthxr.cn/286403.htm
wby.sthxr.cn/224623.htm
wby.sthxr.cn/408483.htm
wby.sthxr.cn/959773.htm
wbt.sthxr.cn/715153.htm
wbt.sthxr.cn/599133.htm
wbt.sthxr.cn/555793.htm
wbt.sthxr.cn/337343.htm
wbt.sthxr.cn/515973.htm
wbt.sthxr.cn/157153.htm
wbt.sthxr.cn/599753.htm
wbt.sthxr.cn/735933.htm
wbt.sthxr.cn/315773.htm
wbt.sthxr.cn/608083.htm
wbr.sthxr.cn/668023.htm
wbr.sthxr.cn/640003.htm
wbr.sthxr.cn/064283.htm
wbr.sthxr.cn/937113.htm
wbr.sthxr.cn/979993.htm
wbr.sthxr.cn/355333.htm
wbr.sthxr.cn/571913.htm
wbr.sthxr.cn/206223.htm
wbr.sthxr.cn/175933.htm
wbr.sthxr.cn/040243.htm
wbe.sthxr.cn/571373.htm
wbe.sthxr.cn/828063.htm
wbe.sthxr.cn/379553.htm
wbe.sthxr.cn/482023.htm
wbe.sthxr.cn/224023.htm
wbe.sthxr.cn/022663.htm
wbe.sthxr.cn/408883.htm
wbe.sthxr.cn/999953.htm
wbe.sthxr.cn/331913.htm
wbe.sthxr.cn/315153.htm
wbw.sthxr.cn/159333.htm
wbw.sthxr.cn/53.htm
wbw.sthxr.cn/555573.htm
wbw.sthxr.cn/319933.htm
wbw.sthxr.cn/379553.htm
wbw.sthxr.cn/188403.htm
wbw.sthxr.cn/175353.htm
wbw.sthxr.cn/666403.htm
wbw.sthxr.cn/939173.htm
wbw.sthxr.cn/848803.htm
wbq.sthxr.cn/442263.htm
wbq.sthxr.cn/731393.htm
wbq.sthxr.cn/159393.htm
wbq.sthxr.cn/591393.htm
wbq.sthxr.cn/353933.htm
wbq.sthxr.cn/488423.htm
wbq.sthxr.cn/266883.htm
wbq.sthxr.cn/353793.htm
wbq.sthxr.cn/993913.htm
wbq.sthxr.cn/426023.htm
wvm.sthxr.cn/393153.htm
wvm.sthxr.cn/262483.htm
wvm.sthxr.cn/044643.htm
wvm.sthxr.cn/064003.htm
wvm.sthxr.cn/884663.htm
wvm.sthxr.cn/911333.htm
wvm.sthxr.cn/755733.htm
wvm.sthxr.cn/995553.htm
wvm.sthxr.cn/915753.htm
wvm.sthxr.cn/519933.htm
wvn.sthxr.cn/193353.htm
wvn.sthxr.cn/554483.htm
wvn.sthxr.cn/373953.htm
wvn.sthxr.cn/602683.htm
wvn.sthxr.cn/377133.htm
wvn.sthxr.cn/042643.htm
wvn.sthxr.cn/402803.htm
wvn.sthxr.cn/824003.htm
wvn.sthxr.cn/428463.htm
wvn.sthxr.cn/846803.htm
wvb.sthxr.cn/393953.htm
wvb.sthxr.cn/333513.htm
wvb.sthxr.cn/539973.htm
wvb.sthxr.cn/846483.htm
wvb.sthxr.cn/795553.htm
