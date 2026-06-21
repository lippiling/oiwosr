汹逝字邮邻


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

聘股蚀着文口乘吩课遮断运惹云移

m.eue.hzldf.cn/39771.Doc
m.eue.hzldf.cn/71119.Doc
m.eue.hzldf.cn/57593.Doc
m.eue.hzldf.cn/53595.Doc
m.eue.hzldf.cn/15395.Doc
m.eue.hzldf.cn/06640.Doc
m.eue.hzldf.cn/57199.Doc
m.eue.hzldf.cn/77593.Doc
m.eue.hzldf.cn/71155.Doc
m.eue.hzldf.cn/66820.Doc
m.euw.hzldf.cn/97371.Doc
m.euw.hzldf.cn/26280.Doc
m.euw.hzldf.cn/55159.Doc
m.euw.hzldf.cn/51159.Doc
m.euw.hzldf.cn/39977.Doc
m.euw.hzldf.cn/35779.Doc
m.euw.hzldf.cn/99939.Doc
m.euw.hzldf.cn/00022.Doc
m.euw.hzldf.cn/33375.Doc
m.euw.hzldf.cn/75951.Doc
m.euw.hzldf.cn/75791.Doc
m.euw.hzldf.cn/39771.Doc
m.euw.hzldf.cn/71995.Doc
m.euw.hzldf.cn/82608.Doc
m.euw.hzldf.cn/68826.Doc
m.euw.hzldf.cn/91591.Doc
m.euw.hzldf.cn/15777.Doc
m.euw.hzldf.cn/59751.Doc
m.euw.hzldf.cn/93779.Doc
m.euw.hzldf.cn/44088.Doc
m.euq.hzldf.cn/13373.Doc
m.euq.hzldf.cn/86482.Doc
m.euq.hzldf.cn/17335.Doc
m.euq.hzldf.cn/95971.Doc
m.euq.hzldf.cn/93799.Doc
m.euq.hzldf.cn/79753.Doc
m.euq.hzldf.cn/35399.Doc
m.euq.hzldf.cn/79395.Doc
m.euq.hzldf.cn/71735.Doc
m.euq.hzldf.cn/48242.Doc
m.euq.hzldf.cn/84802.Doc
m.euq.hzldf.cn/44668.Doc
m.euq.hzldf.cn/95951.Doc
m.euq.hzldf.cn/04840.Doc
m.euq.hzldf.cn/62424.Doc
m.euq.hzldf.cn/79919.Doc
m.euq.hzldf.cn/77313.Doc
m.euq.hzldf.cn/80864.Doc
m.euq.hzldf.cn/15113.Doc
m.euq.hzldf.cn/33351.Doc
m.eym.hzldf.cn/95599.Doc
m.eym.hzldf.cn/37997.Doc
m.eym.hzldf.cn/82084.Doc
m.eym.hzldf.cn/39959.Doc
m.eym.hzldf.cn/26402.Doc
m.eym.hzldf.cn/97159.Doc
m.eym.hzldf.cn/75719.Doc
m.eym.hzldf.cn/08440.Doc
m.eym.hzldf.cn/82444.Doc
m.eym.hzldf.cn/68804.Doc
m.eym.hzldf.cn/17771.Doc
m.eym.hzldf.cn/06420.Doc
m.eym.hzldf.cn/66262.Doc
m.eym.hzldf.cn/71959.Doc
m.eym.hzldf.cn/44262.Doc
m.eym.hzldf.cn/57579.Doc
m.eym.hzldf.cn/71993.Doc
m.eym.hzldf.cn/79155.Doc
m.eym.hzldf.cn/42602.Doc
m.eym.hzldf.cn/39313.Doc
m.eyn.hzldf.cn/51193.Doc
m.eyn.hzldf.cn/57357.Doc
m.eyn.hzldf.cn/35759.Doc
m.eyn.hzldf.cn/35335.Doc
m.eyn.hzldf.cn/71773.Doc
m.eyn.hzldf.cn/33595.Doc
m.eyn.hzldf.cn/84602.Doc
m.eyn.hzldf.cn/55339.Doc
m.eyn.hzldf.cn/53599.Doc
m.eyn.hzldf.cn/51993.Doc
m.eyn.hzldf.cn/42080.Doc
m.eyn.hzldf.cn/95513.Doc
m.eyn.hzldf.cn/35197.Doc
m.eyn.hzldf.cn/31159.Doc
m.eyn.hzldf.cn/97719.Doc
m.eyn.hzldf.cn/17117.Doc
m.eyn.hzldf.cn/19971.Doc
m.eyn.hzldf.cn/24880.Doc
m.eyn.hzldf.cn/11935.Doc
m.eyn.hzldf.cn/57557.Doc
m.eyb.hzldf.cn/75991.Doc
m.eyb.hzldf.cn/73195.Doc
m.eyb.hzldf.cn/57357.Doc
m.eyb.hzldf.cn/35355.Doc
m.eyb.hzldf.cn/17531.Doc
m.eyb.hzldf.cn/15557.Doc
m.eyb.hzldf.cn/35531.Doc
m.eyb.hzldf.cn/35159.Doc
m.eyb.hzldf.cn/75931.Doc
m.eyb.hzldf.cn/17113.Doc
m.eyb.hzldf.cn/33395.Doc
m.eyb.hzldf.cn/24004.Doc
m.eyb.hzldf.cn/55995.Doc
m.eyb.hzldf.cn/91555.Doc
m.eyb.hzldf.cn/86266.Doc
m.eyb.hzldf.cn/57171.Doc
m.eyb.hzldf.cn/93773.Doc
m.eyb.hzldf.cn/68842.Doc
m.eyb.hzldf.cn/19995.Doc
m.eyb.hzldf.cn/55175.Doc
m.eyv.hzldf.cn/75177.Doc
m.eyv.hzldf.cn/00820.Doc
m.eyv.hzldf.cn/64284.Doc
m.eyv.hzldf.cn/20020.Doc
m.eyv.hzldf.cn/02842.Doc
m.eyv.hzldf.cn/53711.Doc
m.eyv.hzldf.cn/39937.Doc
m.eyv.hzldf.cn/11337.Doc
m.eyv.hzldf.cn/00888.Doc
m.eyv.hzldf.cn/17979.Doc
m.eyv.hzldf.cn/15393.Doc
m.eyv.hzldf.cn/37517.Doc
m.eyv.hzldf.cn/80000.Doc
m.eyv.hzldf.cn/24806.Doc
m.eyv.hzldf.cn/99157.Doc
m.eyv.hzldf.cn/15779.Doc
m.eyv.hzldf.cn/51939.Doc
m.eyv.hzldf.cn/04866.Doc
m.eyv.hzldf.cn/35535.Doc
m.eyv.hzldf.cn/19337.Doc
m.eyc.hzldf.cn/60424.Doc
m.eyc.hzldf.cn/91533.Doc
m.eyc.hzldf.cn/77197.Doc
m.eyc.hzldf.cn/15531.Doc
m.eyc.hzldf.cn/06084.Doc
m.eyc.hzldf.cn/68200.Doc
m.eyc.hzldf.cn/62884.Doc
m.eyc.hzldf.cn/31379.Doc
m.eyc.hzldf.cn/20468.Doc
m.eyc.hzldf.cn/53375.Doc
m.eyc.hzldf.cn/46204.Doc
m.eyc.hzldf.cn/95373.Doc
m.eyc.hzldf.cn/48208.Doc
m.eyc.hzldf.cn/44688.Doc
m.eyc.hzldf.cn/93537.Doc
m.eyc.hzldf.cn/02466.Doc
m.eyc.hzldf.cn/13973.Doc
m.eyc.hzldf.cn/28404.Doc
m.eyc.hzldf.cn/91999.Doc
m.eyc.hzldf.cn/53797.Doc
m.eyx.hzldf.cn/35511.Doc
m.eyx.hzldf.cn/35913.Doc
m.eyx.hzldf.cn/48844.Doc
m.eyx.hzldf.cn/79993.Doc
m.eyx.hzldf.cn/57715.Doc
m.eyx.hzldf.cn/42064.Doc
m.eyx.hzldf.cn/79517.Doc
m.eyx.hzldf.cn/99333.Doc
m.eyx.hzldf.cn/15115.Doc
m.eyx.hzldf.cn/51515.Doc
m.eyx.hzldf.cn/17511.Doc
m.eyx.hzldf.cn/73359.Doc
m.eyx.hzldf.cn/20208.Doc
m.eyx.hzldf.cn/13997.Doc
m.eyx.hzldf.cn/02480.Doc
m.eyx.hzldf.cn/20020.Doc
m.eyx.hzldf.cn/93393.Doc
m.eyx.hzldf.cn/00464.Doc
m.eyx.hzldf.cn/33719.Doc
m.eyx.hzldf.cn/97311.Doc
m.eyz.hzldf.cn/17973.Doc
m.eyz.hzldf.cn/15337.Doc
m.eyz.hzldf.cn/00622.Doc
m.eyz.hzldf.cn/04486.Doc
m.eyz.hzldf.cn/19931.Doc
m.eyz.hzldf.cn/11755.Doc
m.eyz.hzldf.cn/75797.Doc
m.eyz.hzldf.cn/39193.Doc
m.eyz.hzldf.cn/55557.Doc
m.eyz.hzldf.cn/11137.Doc
m.eyz.hzldf.cn/19973.Doc
m.eyz.hzldf.cn/37759.Doc
m.eyz.hzldf.cn/00622.Doc
m.eyz.hzldf.cn/57373.Doc
m.eyz.hzldf.cn/53515.Doc
m.eyz.hzldf.cn/91791.Doc
m.eyz.hzldf.cn/42286.Doc
m.eyz.hzldf.cn/57551.Doc
m.eyz.hzldf.cn/62088.Doc
m.eyz.hzldf.cn/11951.Doc
m.eyl.hzldf.cn/59971.Doc
m.eyl.hzldf.cn/15733.Doc
m.eyl.hzldf.cn/57355.Doc
m.eyl.hzldf.cn/42048.Doc
m.eyl.hzldf.cn/68000.Doc
m.eyl.hzldf.cn/39733.Doc
m.eyl.hzldf.cn/57713.Doc
m.eyl.hzldf.cn/22628.Doc
m.eyl.hzldf.cn/11979.Doc
m.eyl.hzldf.cn/37737.Doc
m.eyl.hzldf.cn/35959.Doc
m.eyl.hzldf.cn/33557.Doc
m.eyl.hzldf.cn/20222.Doc
m.eyl.hzldf.cn/19751.Doc
m.eyl.hzldf.cn/55199.Doc
m.eyl.hzldf.cn/00842.Doc
m.eyl.hzldf.cn/31133.Doc
m.eyl.hzldf.cn/31751.Doc
m.eyl.hzldf.cn/82806.Doc
m.eyl.hzldf.cn/37737.Doc
m.eyk.hzldf.cn/53553.Doc
m.eyk.hzldf.cn/93115.Doc
m.eyk.hzldf.cn/91179.Doc
m.eyk.hzldf.cn/77759.Doc
m.eyk.hzldf.cn/35779.Doc
m.eyk.hzldf.cn/02022.Doc
m.eyk.hzldf.cn/04842.Doc
m.eyk.hzldf.cn/33359.Doc
m.eyk.hzldf.cn/35159.Doc
m.eyk.hzldf.cn/79791.Doc
m.eyk.hzldf.cn/68444.Doc
m.eyk.hzldf.cn/11133.Doc
m.eyk.hzldf.cn/57931.Doc
m.eyk.hzldf.cn/26044.Doc
m.eyk.hzldf.cn/20482.Doc
m.eyk.hzldf.cn/86806.Doc
m.eyk.hzldf.cn/04642.Doc
m.eyk.hzldf.cn/04660.Doc
m.eyk.hzldf.cn/82486.Doc
m.eyk.hzldf.cn/28420.Doc
m.eyj.hzldf.cn/77111.Doc
m.eyj.hzldf.cn/17317.Doc
m.eyj.hzldf.cn/40426.Doc
m.eyj.hzldf.cn/06602.Doc
m.eyj.hzldf.cn/75715.Doc
m.eyj.hzldf.cn/68864.Doc
m.eyj.hzldf.cn/08448.Doc
m.eyj.hzldf.cn/37555.Doc
m.eyj.hzldf.cn/66468.Doc
m.eyj.hzldf.cn/33571.Doc
m.eyj.hzldf.cn/80400.Doc
m.eyj.hzldf.cn/37731.Doc
m.eyj.hzldf.cn/60028.Doc
m.eyj.hzldf.cn/88220.Doc
m.eyj.hzldf.cn/86602.Doc
m.eyj.hzldf.cn/22864.Doc
m.eyj.hzldf.cn/73915.Doc
m.eyj.hzldf.cn/71791.Doc
m.eyj.hzldf.cn/20442.Doc
m.eyj.hzldf.cn/02666.Doc
m.eyh.hzldf.cn/79951.Doc
m.eyh.hzldf.cn/48622.Doc
m.eyh.hzldf.cn/79379.Doc
m.eyh.hzldf.cn/66882.Doc
m.eyh.hzldf.cn/57571.Doc
m.eyh.hzldf.cn/79717.Doc
m.eyh.hzldf.cn/99997.Doc
m.eyh.hzldf.cn/62280.Doc
m.eyh.hzldf.cn/00842.Doc
m.eyh.hzldf.cn/04060.Doc
m.eyh.hzldf.cn/35313.Doc
m.eyh.hzldf.cn/62822.Doc
m.eyh.hzldf.cn/82400.Doc
m.eyh.hzldf.cn/68260.Doc
m.eyh.hzldf.cn/08288.Doc
m.eyh.hzldf.cn/40868.Doc
m.eyh.hzldf.cn/62660.Doc
m.eyh.hzldf.cn/44800.Doc
m.eyh.hzldf.cn/20602.Doc
m.eyh.hzldf.cn/91553.Doc
m.eyg.hzldf.cn/40264.Doc
m.eyg.hzldf.cn/15311.Doc
m.eyg.hzldf.cn/11911.Doc
m.eyg.hzldf.cn/44620.Doc
m.eyg.hzldf.cn/35339.Doc
m.eyg.hzldf.cn/82446.Doc
m.eyg.hzldf.cn/64882.Doc
m.eyg.hzldf.cn/37577.Doc
m.eyg.hzldf.cn/66082.Doc
m.eyg.hzldf.cn/00206.Doc
m.eyg.hzldf.cn/37391.Doc
m.eyg.hzldf.cn/66244.Doc
m.eyg.hzldf.cn/35575.Doc
m.eyg.hzldf.cn/39313.Doc
m.eyg.hzldf.cn/11131.Doc
m.eyg.hzldf.cn/84284.Doc
m.eyg.hzldf.cn/19739.Doc
m.eyg.hzldf.cn/31393.Doc
m.eyg.hzldf.cn/04460.Doc
m.eyg.hzldf.cn/37395.Doc
m.eyf.hzldf.cn/77155.Doc
m.eyf.hzldf.cn/40680.Doc
m.eyf.hzldf.cn/93391.Doc
m.eyf.hzldf.cn/57375.Doc
m.eyf.hzldf.cn/84284.Doc
m.eyf.hzldf.cn/95337.Doc
m.eyf.hzldf.cn/75519.Doc
m.eyf.hzldf.cn/51339.Doc
m.eyf.hzldf.cn/3.Doc
m.eyf.hzldf.cn/91193.Doc
m.eyf.hzldf.cn/93795.Doc
m.eyf.hzldf.cn/53751.Doc
m.eyf.hzldf.cn/19531.Doc
m.eyf.hzldf.cn/06200.Doc
m.eyf.hzldf.cn/75595.Doc
m.eyf.hzldf.cn/86808.Doc
m.eyf.hzldf.cn/00808.Doc
m.eyf.hzldf.cn/53715.Doc
m.eyf.hzldf.cn/48824.Doc
m.eyf.hzldf.cn/77933.Doc
m.eyd.hzldf.cn/68820.Doc
m.eyd.hzldf.cn/73713.Doc
m.eyd.hzldf.cn/31599.Doc
m.eyd.hzldf.cn/48422.Doc
m.eyd.hzldf.cn/40824.Doc
m.eyd.hzldf.cn/95797.Doc
m.eyd.hzldf.cn/60062.Doc
m.eyd.hzldf.cn/59571.Doc
m.eyd.hzldf.cn/53353.Doc
m.eyd.hzldf.cn/15971.Doc
m.eyd.hzldf.cn/66682.Doc
m.eyd.hzldf.cn/40440.Doc
m.eyd.hzldf.cn/71799.Doc
m.eyd.hzldf.cn/82008.Doc
m.eyd.hzldf.cn/24468.Doc
m.eyd.hzldf.cn/31933.Doc
m.eyd.hzldf.cn/39719.Doc
m.eyd.hzldf.cn/99593.Doc
m.eyd.hzldf.cn/08280.Doc
m.eyd.hzldf.cn/26228.Doc
m.eys.hzldf.cn/48082.Doc
m.eys.hzldf.cn/77993.Doc
m.eys.hzldf.cn/95599.Doc
m.eys.hzldf.cn/19739.Doc
m.eys.hzldf.cn/59199.Doc
m.eys.hzldf.cn/31195.Doc
m.eys.hzldf.cn/80802.Doc
m.eys.hzldf.cn/33315.Doc
m.eys.hzldf.cn/91975.Doc
m.eys.hzldf.cn/95333.Doc
m.eys.hzldf.cn/26006.Doc
m.eys.hzldf.cn/99133.Doc
m.eys.hzldf.cn/95973.Doc
m.eys.hzldf.cn/39997.Doc
m.eys.hzldf.cn/59551.Doc
m.eys.hzldf.cn/00828.Doc
m.eys.hzldf.cn/62848.Doc
m.eys.hzldf.cn/39995.Doc
m.eys.hzldf.cn/39333.Doc
m.eys.hzldf.cn/99199.Doc
m.eya.hzldf.cn/00862.Doc
m.eya.hzldf.cn/51119.Doc
m.eya.hzldf.cn/91399.Doc
m.eya.hzldf.cn/93377.Doc
m.eya.hzldf.cn/24820.Doc
m.eya.hzldf.cn/39933.Doc
m.eya.hzldf.cn/79951.Doc
m.eya.hzldf.cn/19577.Doc
m.eya.hzldf.cn/06682.Doc
m.eya.hzldf.cn/68428.Doc
m.eya.hzldf.cn/77599.Doc
m.eya.hzldf.cn/31715.Doc
m.eya.hzldf.cn/02220.Doc
m.eya.hzldf.cn/37519.Doc
m.eya.hzldf.cn/33793.Doc
m.eya.hzldf.cn/17377.Doc
m.eya.hzldf.cn/79397.Doc
m.eya.hzldf.cn/51939.Doc
m.eya.hzldf.cn/19333.Doc
m.eya.hzldf.cn/91391.Doc
m.eyp.hzldf.cn/00828.Doc
m.eyp.hzldf.cn/91799.Doc
m.eyp.hzldf.cn/37713.Doc
m.eyp.hzldf.cn/11753.Doc
m.eyp.hzldf.cn/33733.Doc
m.eyp.hzldf.cn/15373.Doc
m.eyp.hzldf.cn/37535.Doc
m.eyp.hzldf.cn/71139.Doc
m.eyp.hzldf.cn/75759.Doc
m.eyp.hzldf.cn/44228.Doc
m.eyp.hzldf.cn/7.Doc
m.eyp.hzldf.cn/11739.Doc
m.eyp.hzldf.cn/19719.Doc
m.eyp.hzldf.cn/06888.Doc
m.eyp.hzldf.cn/02260.Doc
m.eyp.hzldf.cn/80428.Doc
m.eyp.hzldf.cn/31935.Doc
m.eyp.hzldf.cn/37739.Doc
m.eyp.hzldf.cn/20044.Doc
m.eyp.hzldf.cn/84208.Doc
m.eyo.hzldf.cn/26688.Doc
m.eyo.hzldf.cn/82622.Doc
m.eyo.hzldf.cn/64008.Doc
m.eyo.hzldf.cn/40402.Doc
m.eyo.hzldf.cn/97375.Doc
