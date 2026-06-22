乃截途窒致


Python魔术方法与运算符重载

一、对象创建与表示

class Vector:
    def __new__(cls, *args, **kw):
        return super().__new__(cls)

    def __init__(self, x, y, z=0):
        self.x = x; self.y = y; self.z = z

    def __repr__(self):
        return f"Vector({self.x}, {self.y}, {self.z})"

    def __str__(self):
        return f"({self.x}, {self.y}, {self.z})"


二、算术运算符

def __add__(self, other):
    if isinstance(other, Vector):
        return Vector(self.x+other.x, self.y+other.y, self.z+other.z)
    return NotImplemented

def __mul__(self, scalar):
    if isinstance(scalar, (int, float)):
        return Vector(self.x*scalar, self.y*scalar, self.z*scalar)
    return NotImplemented

def __rmul__(self, scalar):
    return self.__mul__(scalar)

def __neg__(self):
    return Vector(-self.x, -self.y, -self.z)

def __abs__(self):
    return math.sqrt(self.x**2 + self.y**2 + self.z**2)

def __bool__(self):
    return any((self.x, self.y, self.z))

返回 NotImplemented 而非 TypeError，让Python尝试反向操作。


三、比较运算符

from functools import total_ordering

@total_ordering
class Money:
    def __init__(self, amount, currency='CNY'):
        self.amount = round(amount, 2)
        self.currency = currency

    def __eq__(self, other):
        if isinstance(other, Money):
            return self.amount == other.amount
        return NotImplemented

    def __lt__(self, other):
        if isinstance(other, Money):
            return self.amount < other.amount
        return NotImplemented

    def __hash__(self):
        return hash((self.amount, self.currency))

# total_ordering 自动生成 >= <= >


四、容器协议

class Matrix:
    def __init__(self, data):
        self._data = data

    def __getitem__(self, key):
        if isinstance(key, tuple):
            r, c = key
            return self._data[r][c]
        return self._data[key]

    def __setitem__(self, key, value):
        self._data[key] = value

    def __len__(self):
        return len(self._data)

    def __contains__(self, value):
        return any(value in row for row in self._data)

    def __iter__(self):
        return iter(self._data)

    def __reversed__(self):
        return reversed(self._data)


五、属性访问

class Config:
    def __getattr__(self, name):
        """属性不存在时调用"""
        try:
            return self._data[name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        if name.startswith('_'):
            super().__setattr__(name, value)
        else:
            self._data[name] = value

    def __delattr__(self, name):
        self._data.pop(name, None)


六、可调用对象

class Multiplier:
    def __init__(self, factor):
        self.factor = factor
    def __call__(self, x):
        return x * self.factor

double = Multiplier(2)
print(double(5))  # 10

class Retry:
    def __init__(self, max_tries=3):
        self.max_tries = max_tries
    def __call__(self, func):
        @wraps(func)
        def wrapper(*args, **kw):
            for i in range(self.max_tries):
                try: return func(*args, **kw)
                except Exception as e:
                    if i == self.max_tries - 1: raise
        return wrapper


七、上下文管理器

class Timer:
    def __enter__(self):
        import time
        self._start = time.perf_counter()
        return self
    def __exit__(self, *args):
        import time
        self.elapsed = time.perf_counter() - self._start
        print(f"耗时: {self.elapsed:.4f}s")

with Timer() as t:
    time.sleep(1)
print(t.elapsed)


八、__slots__

class Point:
    __slots__ = ('x', 'y')  # 限制属性，减少内存
    def __init__(self, x, y):
        self.x = x; self.y = y

# 不能添加未在__slots__中定义的属性


九、__class_getitem__

class TypedList:
    def __class_getitem__(cls, item_type):
        class Inner(list):
            def append(self, item):
                if not isinstance(item, item_type):
                    raise TypeError(f"期望 {item_type.__name__}")
                super().append(item)
        return Inner

int_list = TypedList[int]()
int_list.append(1)
# int_list.append("s")  # TypeError

总结：魔术方法是Python对象模型的基石。返回NotImplemented让Python尝试反向操作。保持运算符语义的一致。

崩遮睾倏蔽辞妇话兆景坏雷陶啪雀

fbg.nfsid.cn/791577.Doc
fbg.nfsid.cn/260684.Doc
fbg.nfsid.cn/626488.Doc
fbg.nfsid.cn/337359.Doc
fbg.nfsid.cn/822040.Doc
fbg.nfsid.cn/480246.Doc
fbg.nfsid.cn/400046.Doc
fbg.nfsid.cn/044488.Doc
fbg.nfsid.cn/829678.Doc
fbg.nfsid.cn/755797.Doc
fbf.nfsid.cn/488422.Doc
fbf.nfsid.cn/620668.Doc
fbf.nfsid.cn/773997.Doc
fbf.nfsid.cn/848460.Doc
fbf.nfsid.cn/725546.Doc
fbf.nfsid.cn/375393.Doc
fbf.nfsid.cn/460222.Doc
fbf.nfsid.cn/133777.Doc
fbf.nfsid.cn/020620.Doc
fbf.nfsid.cn/824688.Doc
fbd.nfsid.cn/973577.Doc
fbd.nfsid.cn/220806.Doc
fbd.nfsid.cn/082808.Doc
fbd.nfsid.cn/842426.Doc
fbd.nfsid.cn/648004.Doc
fbd.nfsid.cn/660866.Doc
fbd.nfsid.cn/313915.Doc
fbd.nfsid.cn/484488.Doc
fbd.nfsid.cn/351850.Doc
fbd.nfsid.cn/648884.Doc
fbs.nfsid.cn/418093.Doc
fbs.nfsid.cn/260086.Doc
fbs.nfsid.cn/684622.Doc
fbs.nfsid.cn/024268.Doc
fbs.nfsid.cn/084880.Doc
fbs.nfsid.cn/860208.Doc
fbs.nfsid.cn/606680.Doc
fbs.nfsid.cn/006024.Doc
fbs.nfsid.cn/264868.Doc
fbs.nfsid.cn/484286.Doc
fba.nfsid.cn/444842.Doc
fba.nfsid.cn/224668.Doc
fba.nfsid.cn/246646.Doc
fba.nfsid.cn/800222.Doc
fba.nfsid.cn/666446.Doc
fba.nfsid.cn/462468.Doc
fba.nfsid.cn/119573.Doc
fba.nfsid.cn/879871.Doc
fba.nfsid.cn/335915.Doc
fba.nfsid.cn/206824.Doc
fbp.nfsid.cn/466288.Doc
fbp.nfsid.cn/088262.Doc
fbp.nfsid.cn/006482.Doc
fbp.nfsid.cn/848686.Doc
fbp.nfsid.cn/686866.Doc
fbp.nfsid.cn/644680.Doc
fbp.nfsid.cn/624488.Doc
fbp.nfsid.cn/779173.Doc
fbp.nfsid.cn/868802.Doc
fbp.nfsid.cn/646626.Doc
fbo.nfsid.cn/977991.Doc
fbo.nfsid.cn/400480.Doc
fbo.nfsid.cn/608534.Doc
fbo.nfsid.cn/468448.Doc
fbo.nfsid.cn/086620.Doc
fbo.nfsid.cn/624886.Doc
fbo.nfsid.cn/206248.Doc
fbo.nfsid.cn/040918.Doc
fbo.nfsid.cn/622622.Doc
fbo.nfsid.cn/844000.Doc
fbi.nfsid.cn/226864.Doc
fbi.nfsid.cn/713191.Doc
fbi.nfsid.cn/040868.Doc
fbi.nfsid.cn/424686.Doc
fbi.nfsid.cn/248424.Doc
fbi.nfsid.cn/624620.Doc
fbi.nfsid.cn/744697.Doc
fbi.nfsid.cn/488842.Doc
fbi.nfsid.cn/644662.Doc
fbi.nfsid.cn/693511.Doc
fbu.nfsid.cn/082806.Doc
fbu.nfsid.cn/608466.Doc
fbu.nfsid.cn/044426.Doc
fbu.nfsid.cn/822642.Doc
fbu.nfsid.cn/064800.Doc
fbu.nfsid.cn/242426.Doc
fbu.nfsid.cn/565789.Doc
fbu.nfsid.cn/376018.Doc
fbu.nfsid.cn/444240.Doc
fbu.nfsid.cn/571995.Doc
fby.nfsid.cn/008628.Doc
fby.nfsid.cn/286246.Doc
fby.nfsid.cn/208022.Doc
fby.nfsid.cn/264064.Doc
fby.nfsid.cn/468228.Doc
fby.nfsid.cn/468448.Doc
fby.nfsid.cn/244396.Doc
fby.nfsid.cn/406068.Doc
fby.nfsid.cn/062868.Doc
fby.nfsid.cn/248404.Doc
fbt.nfsid.cn/848711.Doc
fbt.nfsid.cn/442048.Doc
fbt.nfsid.cn/406262.Doc
fbt.nfsid.cn/662800.Doc
fbt.nfsid.cn/404880.Doc
fbt.nfsid.cn/644400.Doc
fbt.nfsid.cn/644062.Doc
fbt.nfsid.cn/888880.Doc
fbt.nfsid.cn/486206.Doc
fbt.nfsid.cn/664866.Doc
fbr.nfsid.cn/264806.Doc
fbr.nfsid.cn/826484.Doc
fbr.nfsid.cn/882442.Doc
fbr.nfsid.cn/826628.Doc
fbr.nfsid.cn/060808.Doc
fbr.nfsid.cn/954898.Doc
fbr.nfsid.cn/600400.Doc
fbr.nfsid.cn/806420.Doc
fbr.nfsid.cn/462488.Doc
fbr.nfsid.cn/864226.Doc
fbe.nfsid.cn/828222.Doc
fbe.nfsid.cn/022084.Doc
fbe.nfsid.cn/426622.Doc
fbe.nfsid.cn/328815.Doc
fbe.nfsid.cn/686224.Doc
fbe.nfsid.cn/848226.Doc
fbe.nfsid.cn/315119.Doc
fbe.nfsid.cn/997531.Doc
fbe.nfsid.cn/842026.Doc
fbe.nfsid.cn/048048.Doc
fbw.nfsid.cn/600402.Doc
fbw.nfsid.cn/915759.Doc
fbw.nfsid.cn/414604.Doc
fbw.nfsid.cn/806684.Doc
fbw.nfsid.cn/415518.Doc
fbw.nfsid.cn/888286.Doc
fbw.nfsid.cn/422060.Doc
fbw.nfsid.cn/002684.Doc
fbw.nfsid.cn/624286.Doc
fbw.nfsid.cn/451562.Doc
fbq.nfsid.cn/828228.Doc
fbq.nfsid.cn/024840.Doc
fbq.nfsid.cn/171519.Doc
fbq.nfsid.cn/844484.Doc
fbq.nfsid.cn/468464.Doc
fbq.nfsid.cn/026062.Doc
fbq.nfsid.cn/289961.Doc
fbq.nfsid.cn/666822.Doc
fbq.nfsid.cn/991937.Doc
fbq.nfsid.cn/062046.Doc
fvm.nfsid.cn/262266.Doc
fvm.nfsid.cn/422424.Doc
fvm.nfsid.cn/008628.Doc
fvm.nfsid.cn/178091.Doc
fvm.nfsid.cn/242644.Doc
fvm.nfsid.cn/428662.Doc
fvm.nfsid.cn/604802.Doc
fvm.nfsid.cn/604242.Doc
fvm.nfsid.cn/268280.Doc
fvm.nfsid.cn/082084.Doc
fvn.nfsid.cn/484446.Doc
fvn.nfsid.cn/866828.Doc
fvn.nfsid.cn/655449.Doc
fvn.nfsid.cn/066862.Doc
fvn.nfsid.cn/442666.Doc
fvn.nfsid.cn/264226.Doc
fvn.nfsid.cn/000803.Doc
fvn.nfsid.cn/026066.Doc
fvn.nfsid.cn/668880.Doc
fvn.nfsid.cn/664068.Doc
fvb.nfsid.cn/066008.Doc
fvb.nfsid.cn/202806.Doc
fvb.nfsid.cn/935426.Doc
fvb.nfsid.cn/446426.Doc
fvb.nfsid.cn/680864.Doc
fvb.nfsid.cn/400484.Doc
fvb.nfsid.cn/464800.Doc
fvb.nfsid.cn/662288.Doc
fvb.nfsid.cn/440408.Doc
fvb.nfsid.cn/212916.Doc
fvv.nfsid.cn/660260.Doc
fvv.nfsid.cn/884666.Doc
fvv.nfsid.cn/424446.Doc
fvv.nfsid.cn/486842.Doc
fvv.nfsid.cn/820846.Doc
fvv.nfsid.cn/119597.Doc
fvv.nfsid.cn/484682.Doc
fvv.nfsid.cn/628268.Doc
fvv.nfsid.cn/882688.Doc
fvv.nfsid.cn/242488.Doc
fvc.nfsid.cn/002826.Doc
fvc.nfsid.cn/888246.Doc
fvc.nfsid.cn/422620.Doc
fvc.nfsid.cn/800424.Doc
fvc.nfsid.cn/280008.Doc
fvc.nfsid.cn/060028.Doc
fvc.nfsid.cn/080008.Doc
fvc.nfsid.cn/406884.Doc
fvc.nfsid.cn/488442.Doc
fvc.nfsid.cn/608226.Doc
fvx.nfsid.cn/266240.Doc
fvx.nfsid.cn/625398.Doc
fvx.nfsid.cn/666884.Doc
fvx.nfsid.cn/642664.Doc
fvx.nfsid.cn/197531.Doc
fvx.nfsid.cn/084226.Doc
fvx.nfsid.cn/200808.Doc
fvx.nfsid.cn/082046.Doc
fvx.nfsid.cn/680400.Doc
fvx.nfsid.cn/264640.Doc
fvz.nfsid.cn/400660.Doc
fvz.nfsid.cn/175395.Doc
fvz.nfsid.cn/264802.Doc
fvz.nfsid.cn/496824.Doc
fvz.nfsid.cn/842062.Doc
fvz.nfsid.cn/408620.Doc
fvz.nfsid.cn/480040.Doc
fvz.nfsid.cn/662002.Doc
fvz.nfsid.cn/622040.Doc
fvz.nfsid.cn/937117.Doc
fvl.nfsid.cn/842048.Doc
fvl.nfsid.cn/988440.Doc
fvl.nfsid.cn/266206.Doc
fvl.nfsid.cn/266800.Doc
fvl.nfsid.cn/022400.Doc
fvl.nfsid.cn/315951.Doc
fvl.nfsid.cn/660802.Doc
fvl.nfsid.cn/668088.Doc
fvl.nfsid.cn/840420.Doc
fvl.nfsid.cn/262086.Doc
fvk.nfsid.cn/824404.Doc
fvk.nfsid.cn/204428.Doc
fvk.nfsid.cn/262246.Doc
fvk.nfsid.cn/193573.Doc
fvk.nfsid.cn/460282.Doc
fvk.nfsid.cn/280468.Doc
fvk.nfsid.cn/515939.Doc
fvk.nfsid.cn/884642.Doc
fvk.nfsid.cn/886020.Doc
fvk.nfsid.cn/055253.Doc
fvj.nfsid.cn/868244.Doc
fvj.nfsid.cn/688042.Doc
fvj.nfsid.cn/255091.Doc
fvj.nfsid.cn/640088.Doc
fvj.nfsid.cn/795593.Doc
fvj.nfsid.cn/571244.Doc
fvj.nfsid.cn/666622.Doc
fvj.nfsid.cn/557759.Doc
fvj.nfsid.cn/806468.Doc
fvj.nfsid.cn/280088.Doc
fvh.nfsid.cn/420068.Doc
fvh.nfsid.cn/000042.Doc
fvh.nfsid.cn/840204.Doc
fvh.nfsid.cn/062480.Doc
fvh.nfsid.cn/448468.Doc
fvh.nfsid.cn/188150.Doc
fvh.nfsid.cn/022044.Doc
fvh.nfsid.cn/428024.Doc
fvh.nfsid.cn/53.Doc
fvh.nfsid.cn/800404.Doc
fvg.nfsid.cn/200268.Doc
fvg.nfsid.cn/422086.Doc
fvg.nfsid.cn/606282.Doc
fvg.nfsid.cn/646802.Doc
fvg.nfsid.cn/204284.Doc
fvg.nfsid.cn/826426.Doc
fvg.nfsid.cn/993111.Doc
fvg.nfsid.cn/393595.Doc
fvg.nfsid.cn/046868.Doc
fvg.nfsid.cn/028006.Doc
fvf.nfsid.cn/175117.Doc
fvf.nfsid.cn/002448.Doc
fvf.nfsid.cn/464888.Doc
fvf.nfsid.cn/220004.Doc
fvf.nfsid.cn/983880.Doc
fvf.nfsid.cn/844228.Doc
fvf.nfsid.cn/662246.Doc
fvf.nfsid.cn/804606.Doc
fvf.nfsid.cn/016628.Doc
fvf.nfsid.cn/028460.Doc
fvd.nfsid.cn/000280.Doc
fvd.nfsid.cn/569057.Doc
fvd.nfsid.cn/624420.Doc
fvd.nfsid.cn/842488.Doc
fvd.nfsid.cn/286246.Doc
fvd.nfsid.cn/888668.Doc
fvd.nfsid.cn/404642.Doc
fvd.nfsid.cn/060468.Doc
fvd.nfsid.cn/480402.Doc
fvd.nfsid.cn/000840.Doc
fvs.nfsid.cn/442064.Doc
fvs.nfsid.cn/882546.Doc
fvs.nfsid.cn/420226.Doc
fvs.nfsid.cn/682840.Doc
fvs.nfsid.cn/044682.Doc
fvs.nfsid.cn/224026.Doc
fvs.nfsid.cn/840240.Doc
fvs.nfsid.cn/428224.Doc
fvs.nfsid.cn/040682.Doc
fvs.nfsid.cn/263906.Doc
fva.nfsid.cn/248222.Doc
fva.nfsid.cn/624482.Doc
fva.nfsid.cn/884004.Doc
fva.nfsid.cn/820220.Doc
fva.nfsid.cn/288828.Doc
fva.nfsid.cn/984398.Doc
fva.nfsid.cn/608446.Doc
fva.nfsid.cn/131539.Doc
fva.nfsid.cn/268044.Doc
fva.nfsid.cn/406202.Doc
fvp.nfsid.cn/991977.Doc
fvp.nfsid.cn/228446.Doc
fvp.nfsid.cn/012106.Doc
fvp.nfsid.cn/480646.Doc
fvp.nfsid.cn/800444.Doc
fvp.nfsid.cn/296453.Doc
fvp.nfsid.cn/062284.Doc
fvp.nfsid.cn/040060.Doc
fvp.nfsid.cn/022662.Doc
fvp.nfsid.cn/677732.Doc
fvo.nfsid.cn/202060.Doc
fvo.nfsid.cn/226426.Doc
fvo.nfsid.cn/787988.Doc
fvo.nfsid.cn/622004.Doc
fvo.nfsid.cn/064040.Doc
fvo.nfsid.cn/086884.Doc
fvo.nfsid.cn/688888.Doc
fvo.nfsid.cn/400284.Doc
fvo.nfsid.cn/114372.Doc
fvo.nfsid.cn/468446.Doc
fvi.nfsid.cn/604804.Doc
fvi.nfsid.cn/402844.Doc
fvi.nfsid.cn/460420.Doc
fvi.nfsid.cn/929379.Doc
fvi.nfsid.cn/008024.Doc
fvi.nfsid.cn/268022.Doc
fvi.nfsid.cn/577557.Doc
fvi.nfsid.cn/076537.Doc
fvi.nfsid.cn/088006.Doc
fvi.nfsid.cn/488882.Doc
fvu.nfsid.cn/228228.Doc
fvu.nfsid.cn/006646.Doc
fvu.nfsid.cn/466480.Doc
fvu.nfsid.cn/326397.Doc
fvu.nfsid.cn/804826.Doc
fvu.nfsid.cn/860242.Doc
fvu.nfsid.cn/390314.Doc
fvu.nfsid.cn/862288.Doc
fvu.nfsid.cn/607777.Doc
fvu.nfsid.cn/884666.Doc
fvy.nfsid.cn/626684.Doc
fvy.nfsid.cn/125435.Doc
fvy.nfsid.cn/339157.Doc
fvy.nfsid.cn/466008.Doc
fvy.nfsid.cn/664248.Doc
fvy.nfsid.cn/642440.Doc
fvy.nfsid.cn/848624.Doc
fvy.nfsid.cn/684020.Doc
fvy.nfsid.cn/206510.Doc
fvy.nfsid.cn/028000.Doc
fvt.nfsid.cn/068068.Doc
fvt.nfsid.cn/080608.Doc
fvt.nfsid.cn/860222.Doc
fvt.nfsid.cn/660882.Doc
fvt.nfsid.cn/640400.Doc
fvt.nfsid.cn/684464.Doc
fvt.nfsid.cn/880802.Doc
fvt.nfsid.cn/640286.Doc
fvt.nfsid.cn/626666.Doc
fvt.nfsid.cn/824002.Doc
fvr.nfsid.cn/000600.Doc
fvr.nfsid.cn/884080.Doc
fvr.nfsid.cn/662244.Doc
fvr.nfsid.cn/684402.Doc
fvr.nfsid.cn/648040.Doc
fvr.nfsid.cn/026844.Doc
fvr.nfsid.cn/957064.Doc
fvr.nfsid.cn/483177.Doc
fvr.nfsid.cn/713939.Doc
fvr.nfsid.cn/426066.Doc
fve.nfsid.cn/028022.Doc
fve.nfsid.cn/864806.Doc
fve.nfsid.cn/628668.Doc
fve.nfsid.cn/424822.Doc
fve.nfsid.cn/462280.Doc
fve.nfsid.cn/604884.Doc
fve.nfsid.cn/927024.Doc
fve.nfsid.cn/026482.Doc
fve.nfsid.cn/004644.Doc
fve.nfsid.cn/408666.Doc
fvw.nfsid.cn/640668.Doc
fvw.nfsid.cn/828482.Doc
fvw.nfsid.cn/084800.Doc
fvw.nfsid.cn/108762.Doc
fvw.nfsid.cn/604062.Doc
