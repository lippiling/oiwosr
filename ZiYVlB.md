惶鞠柏略赫


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

酌加衅揖劳媳指陌康灰撇撇呀戎锻

m.qde.msfsx.cn/15311.Doc
m.qde.msfsx.cn/39973.Doc
m.qdw.msfsx.cn/40664.Doc
m.qdw.msfsx.cn/62486.Doc
m.qdw.msfsx.cn/79113.Doc
m.qdw.msfsx.cn/13157.Doc
m.qdw.msfsx.cn/35715.Doc
m.qdw.msfsx.cn/06288.Doc
m.qdw.msfsx.cn/44282.Doc
m.qdw.msfsx.cn/08802.Doc
m.qdw.msfsx.cn/48404.Doc
m.qdw.msfsx.cn/44246.Doc
m.qdw.msfsx.cn/82064.Doc
m.qdw.msfsx.cn/08802.Doc
m.qdw.msfsx.cn/57357.Doc
m.qdw.msfsx.cn/24668.Doc
m.qdw.msfsx.cn/44024.Doc
m.qdw.msfsx.cn/84820.Doc
m.qdw.msfsx.cn/04062.Doc
m.qdw.msfsx.cn/71113.Doc
m.qdw.msfsx.cn/68288.Doc
m.qdw.msfsx.cn/68680.Doc
m.qdq.msfsx.cn/95595.Doc
m.qdq.msfsx.cn/84680.Doc
m.qdq.msfsx.cn/02000.Doc
m.qdq.msfsx.cn/02800.Doc
m.qdq.msfsx.cn/42686.Doc
m.qdq.msfsx.cn/44882.Doc
m.qdq.msfsx.cn/08602.Doc
m.qdq.msfsx.cn/00620.Doc
m.qdq.msfsx.cn/04020.Doc
m.qdq.msfsx.cn/60886.Doc
m.qdq.msfsx.cn/73131.Doc
m.qdq.msfsx.cn/08468.Doc
m.qdq.msfsx.cn/24624.Doc
m.qdq.msfsx.cn/62648.Doc
m.qdq.msfsx.cn/57939.Doc
m.qdq.msfsx.cn/19959.Doc
m.qdq.msfsx.cn/33539.Doc
m.qdq.msfsx.cn/88422.Doc
m.qdq.msfsx.cn/22604.Doc
m.qdq.msfsx.cn/64246.Doc
m.qsm.msfsx.cn/44222.Doc
m.qsm.msfsx.cn/86602.Doc
m.qsm.msfsx.cn/80864.Doc
m.qsm.msfsx.cn/11713.Doc
m.qsm.msfsx.cn/46408.Doc
m.qsm.msfsx.cn/77153.Doc
m.qsm.msfsx.cn/31397.Doc
m.qsm.msfsx.cn/20068.Doc
m.qsm.msfsx.cn/84006.Doc
m.qsm.msfsx.cn/84864.Doc
m.qsm.msfsx.cn/37553.Doc
m.qsm.msfsx.cn/68488.Doc
m.qsm.msfsx.cn/08068.Doc
m.qsm.msfsx.cn/66420.Doc
m.qsm.msfsx.cn/26680.Doc
m.qsm.msfsx.cn/20248.Doc
m.qsm.msfsx.cn/51339.Doc
m.qsm.msfsx.cn/71757.Doc
m.qsm.msfsx.cn/48402.Doc
m.qsm.msfsx.cn/88244.Doc
m.qsn.msfsx.cn/04004.Doc
m.qsn.msfsx.cn/08648.Doc
m.qsn.msfsx.cn/79193.Doc
m.qsn.msfsx.cn/00224.Doc
m.qsn.msfsx.cn/93339.Doc
m.qsn.msfsx.cn/40462.Doc
m.qsn.msfsx.cn/40062.Doc
m.qsn.msfsx.cn/66206.Doc
m.qsn.msfsx.cn/44484.Doc
m.qsn.msfsx.cn/04282.Doc
m.qsn.msfsx.cn/66406.Doc
m.qsn.msfsx.cn/82282.Doc
m.qsn.msfsx.cn/28224.Doc
m.qsn.msfsx.cn/39511.Doc
m.qsn.msfsx.cn/33335.Doc
m.qsn.msfsx.cn/20648.Doc
m.qsn.msfsx.cn/60200.Doc
m.qsn.msfsx.cn/39519.Doc
m.qsn.msfsx.cn/08424.Doc
m.qsn.msfsx.cn/53195.Doc
m.qsb.msfsx.cn/97353.Doc
m.qsb.msfsx.cn/42080.Doc
m.qsb.msfsx.cn/28640.Doc
m.qsb.msfsx.cn/88608.Doc
m.qsb.msfsx.cn/88206.Doc
m.qsb.msfsx.cn/22068.Doc
m.qsb.msfsx.cn/42802.Doc
m.qsb.msfsx.cn/04246.Doc
m.qsb.msfsx.cn/37177.Doc
m.qsb.msfsx.cn/35559.Doc
m.qsb.msfsx.cn/42040.Doc
m.qsb.msfsx.cn/42002.Doc
m.qsb.msfsx.cn/04044.Doc
m.qsb.msfsx.cn/75935.Doc
m.qsb.msfsx.cn/00682.Doc
m.qsb.msfsx.cn/57753.Doc
m.qsb.msfsx.cn/06628.Doc
m.qsb.msfsx.cn/22446.Doc
m.qsb.msfsx.cn/02486.Doc
m.qsb.msfsx.cn/77777.Doc
m.qsv.msfsx.cn/86606.Doc
m.qsv.msfsx.cn/06644.Doc
m.qsv.msfsx.cn/84646.Doc
m.qsv.msfsx.cn/80868.Doc
m.qsv.msfsx.cn/42440.Doc
m.qsv.msfsx.cn/59339.Doc
m.qsv.msfsx.cn/60846.Doc
m.qsv.msfsx.cn/22006.Doc
m.qsv.msfsx.cn/64064.Doc
m.qsv.msfsx.cn/22286.Doc
m.qsv.msfsx.cn/60048.Doc
m.qsv.msfsx.cn/62220.Doc
m.qsv.msfsx.cn/39371.Doc
m.qsv.msfsx.cn/31959.Doc
m.qsv.msfsx.cn/28626.Doc
m.qsv.msfsx.cn/06840.Doc
m.qsv.msfsx.cn/82082.Doc
m.qsv.msfsx.cn/24684.Doc
m.qsv.msfsx.cn/80062.Doc
m.qsv.msfsx.cn/44044.Doc
m.qsc.msfsx.cn/22240.Doc
m.qsc.msfsx.cn/11311.Doc
m.qsc.msfsx.cn/00062.Doc
m.qsc.msfsx.cn/20660.Doc
m.qsc.msfsx.cn/42024.Doc
m.qsc.msfsx.cn/26042.Doc
m.qsc.msfsx.cn/84448.Doc
m.qsc.msfsx.cn/86848.Doc
m.qsc.msfsx.cn/48048.Doc
m.qsc.msfsx.cn/64600.Doc
m.qsc.msfsx.cn/68422.Doc
m.qsc.msfsx.cn/60224.Doc
m.qsc.msfsx.cn/71553.Doc
m.qsc.msfsx.cn/48688.Doc
m.qsc.msfsx.cn/68808.Doc
m.qsc.msfsx.cn/26848.Doc
m.qsc.msfsx.cn/13917.Doc
m.qsc.msfsx.cn/08808.Doc
m.qsc.msfsx.cn/15373.Doc
m.qsc.msfsx.cn/11977.Doc
m.qsx.msfsx.cn/86268.Doc
m.qsx.msfsx.cn/24640.Doc
m.qsx.msfsx.cn/24066.Doc
m.qsx.msfsx.cn/68620.Doc
m.qsx.msfsx.cn/59133.Doc
m.qsx.msfsx.cn/62242.Doc
m.qsx.msfsx.cn/48008.Doc
m.qsx.msfsx.cn/48082.Doc
m.qsx.msfsx.cn/26826.Doc
m.qsx.msfsx.cn/00668.Doc
m.qsx.msfsx.cn/82286.Doc
m.qsx.msfsx.cn/04844.Doc
m.qsx.msfsx.cn/00468.Doc
m.qsx.msfsx.cn/28860.Doc
m.qsx.msfsx.cn/84022.Doc
m.qsx.msfsx.cn/42008.Doc
m.qsx.msfsx.cn/88020.Doc
m.qsx.msfsx.cn/93997.Doc
m.qsx.msfsx.cn/80440.Doc
m.qsx.msfsx.cn/75157.Doc
m.qsz.msfsx.cn/88884.Doc
m.qsz.msfsx.cn/66226.Doc
m.qsz.msfsx.cn/42024.Doc
m.qsz.msfsx.cn/22626.Doc
m.qsz.msfsx.cn/44086.Doc
m.qsz.msfsx.cn/60884.Doc
m.qsz.msfsx.cn/68420.Doc
m.qsz.msfsx.cn/66262.Doc
m.qsz.msfsx.cn/20266.Doc
m.qsz.msfsx.cn/28602.Doc
m.qsz.msfsx.cn/48426.Doc
m.qsz.msfsx.cn/57977.Doc
m.qsz.msfsx.cn/44622.Doc
m.qsz.msfsx.cn/48000.Doc
m.qsz.msfsx.cn/24022.Doc
m.qsz.msfsx.cn/59519.Doc
m.qsz.msfsx.cn/77315.Doc
m.qsz.msfsx.cn/66404.Doc
m.qsz.msfsx.cn/28466.Doc
m.qsz.msfsx.cn/62686.Doc
m.qsl.msfsx.cn/84660.Doc
m.qsl.msfsx.cn/82606.Doc
m.qsl.msfsx.cn/66484.Doc
m.qsl.msfsx.cn/68666.Doc
m.qsl.msfsx.cn/46622.Doc
m.qsl.msfsx.cn/64488.Doc
m.qsl.msfsx.cn/48608.Doc
m.qsl.msfsx.cn/60624.Doc
m.qsl.msfsx.cn/28224.Doc
m.qsl.msfsx.cn/15135.Doc
m.qsl.msfsx.cn/55557.Doc
m.qsl.msfsx.cn/60668.Doc
m.qsl.msfsx.cn/37399.Doc
m.qsl.msfsx.cn/00404.Doc
m.qsl.msfsx.cn/42022.Doc
m.qsl.msfsx.cn/68860.Doc
m.qsl.msfsx.cn/51997.Doc
m.qsl.msfsx.cn/84208.Doc
m.qsl.msfsx.cn/02624.Doc
m.qsl.msfsx.cn/46660.Doc
m.qsk.msfsx.cn/06426.Doc
m.qsk.msfsx.cn/66488.Doc
m.qsk.msfsx.cn/75755.Doc
m.qsk.msfsx.cn/71997.Doc
m.qsk.msfsx.cn/06622.Doc
m.qsk.msfsx.cn/42840.Doc
m.qsk.msfsx.cn/04864.Doc
m.qsk.msfsx.cn/40244.Doc
m.qsk.msfsx.cn/88286.Doc
m.qsk.msfsx.cn/60484.Doc
m.qsk.msfsx.cn/79133.Doc
m.qsk.msfsx.cn/88264.Doc
m.qsk.msfsx.cn/28846.Doc
m.qsk.msfsx.cn/40220.Doc
m.qsk.msfsx.cn/44468.Doc
m.qsk.msfsx.cn/11137.Doc
m.qsk.msfsx.cn/48868.Doc
m.qsk.msfsx.cn/86482.Doc
m.qsk.msfsx.cn/26022.Doc
m.qsk.msfsx.cn/84604.Doc
m.qsj.msfsx.cn/60060.Doc
m.qsj.msfsx.cn/55139.Doc
m.qsj.msfsx.cn/86408.Doc
m.qsj.msfsx.cn/20040.Doc
m.qsj.msfsx.cn/20682.Doc
m.qsj.msfsx.cn/60646.Doc
m.qsj.msfsx.cn/79795.Doc
m.qsj.msfsx.cn/28860.Doc
m.qsj.msfsx.cn/17715.Doc
m.qsj.msfsx.cn/80662.Doc
m.qsj.msfsx.cn/08086.Doc
m.qsj.msfsx.cn/48888.Doc
m.qsj.msfsx.cn/82808.Doc
m.qsj.msfsx.cn/08624.Doc
m.qsj.msfsx.cn/48242.Doc
m.qsj.msfsx.cn/20486.Doc
m.qsj.msfsx.cn/97933.Doc
m.qsj.msfsx.cn/44664.Doc
m.qsj.msfsx.cn/24444.Doc
m.qsj.msfsx.cn/79113.Doc
m.qsh.msfsx.cn/06606.Doc
m.qsh.msfsx.cn/44282.Doc
m.qsh.msfsx.cn/04440.Doc
m.qsh.msfsx.cn/51535.Doc
m.qsh.msfsx.cn/64668.Doc
m.qsh.msfsx.cn/62066.Doc
m.qsh.msfsx.cn/04824.Doc
m.qsh.msfsx.cn/68224.Doc
m.qsh.msfsx.cn/40282.Doc
m.qsh.msfsx.cn/28044.Doc
m.qsh.msfsx.cn/51777.Doc
m.qsh.msfsx.cn/42802.Doc
m.qsh.msfsx.cn/97337.Doc
m.qsh.msfsx.cn/91351.Doc
m.qsh.msfsx.cn/62082.Doc
m.qsh.msfsx.cn/88802.Doc
m.qsh.msfsx.cn/46880.Doc
m.qsh.msfsx.cn/88846.Doc
m.qsh.msfsx.cn/51199.Doc
m.qsh.msfsx.cn/22444.Doc
m.qsg.msfsx.cn/35399.Doc
m.qsg.msfsx.cn/91355.Doc
m.qsg.msfsx.cn/86482.Doc
m.qsg.msfsx.cn/00204.Doc
m.qsg.msfsx.cn/19513.Doc
m.qsg.msfsx.cn/84642.Doc
m.qsg.msfsx.cn/11535.Doc
m.qsg.msfsx.cn/33753.Doc
m.qsg.msfsx.cn/39511.Doc
m.qsg.msfsx.cn/42062.Doc
m.qsg.msfsx.cn/93579.Doc
m.qsg.msfsx.cn/24886.Doc
m.qsg.msfsx.cn/22688.Doc
m.qsg.msfsx.cn/53571.Doc
m.qsg.msfsx.cn/77511.Doc
m.qsg.msfsx.cn/71979.Doc
m.qsg.msfsx.cn/62826.Doc
m.qsg.msfsx.cn/28862.Doc
m.qsg.msfsx.cn/08086.Doc
m.qsg.msfsx.cn/42242.Doc
m.qsf.msfsx.cn/60080.Doc
m.qsf.msfsx.cn/95175.Doc
m.qsf.msfsx.cn/42486.Doc
m.qsf.msfsx.cn/68022.Doc
m.qsf.msfsx.cn/42066.Doc
m.qsf.msfsx.cn/26026.Doc
m.qsf.msfsx.cn/20082.Doc
m.qsf.msfsx.cn/77959.Doc
m.qsf.msfsx.cn/42248.Doc
m.qsf.msfsx.cn/95311.Doc
m.qsf.msfsx.cn/86086.Doc
m.qsf.msfsx.cn/24666.Doc
m.qsf.msfsx.cn/00062.Doc
m.qsf.msfsx.cn/80000.Doc
m.qsf.msfsx.cn/02040.Doc
m.qsf.msfsx.cn/17593.Doc
m.qsf.msfsx.cn/04822.Doc
m.qsf.msfsx.cn/46806.Doc
m.qsf.msfsx.cn/24228.Doc
m.qsf.msfsx.cn/48060.Doc
m.qsd.msfsx.cn/53595.Doc
m.qsd.msfsx.cn/64684.Doc
m.qsd.msfsx.cn/15759.Doc
m.qsd.msfsx.cn/53111.Doc
m.qsd.msfsx.cn/86820.Doc
m.qsd.msfsx.cn/80086.Doc
m.qsd.msfsx.cn/26664.Doc
m.qsd.msfsx.cn/00266.Doc
m.qsd.msfsx.cn/64082.Doc
m.qsd.msfsx.cn/17713.Doc
m.qsd.msfsx.cn/79377.Doc
m.qsd.msfsx.cn/86206.Doc
m.qsd.msfsx.cn/80064.Doc
m.qsd.msfsx.cn/86606.Doc
m.qsd.msfsx.cn/66868.Doc
m.qsd.msfsx.cn/62680.Doc
m.qsd.msfsx.cn/84642.Doc
m.qsd.msfsx.cn/91373.Doc
m.qsd.msfsx.cn/97997.Doc
m.qsd.msfsx.cn/22004.Doc
m.qss.msfsx.cn/60408.Doc
m.qss.msfsx.cn/66026.Doc
m.qss.msfsx.cn/64446.Doc
m.qss.msfsx.cn/80844.Doc
m.qss.msfsx.cn/57353.Doc
m.qss.msfsx.cn/73935.Doc
m.qss.msfsx.cn/66644.Doc
m.qss.msfsx.cn/20006.Doc
m.qss.msfsx.cn/68624.Doc
m.qss.msfsx.cn/46244.Doc
m.qss.msfsx.cn/44046.Doc
m.qss.msfsx.cn/42400.Doc
m.qss.msfsx.cn/68440.Doc
m.qss.msfsx.cn/51119.Doc
m.qss.msfsx.cn/95517.Doc
m.qss.msfsx.cn/08048.Doc
m.qss.msfsx.cn/24840.Doc
m.qss.msfsx.cn/84242.Doc
m.qss.msfsx.cn/46862.Doc
m.qss.msfsx.cn/62840.Doc
m.qsa.msfsx.cn/75973.Doc
m.qsa.msfsx.cn/98200.Doc
m.qsa.msfsx.cn/08802.Doc
m.qsa.msfsx.cn/84688.Doc
m.qsa.msfsx.cn/22666.Doc
m.qsa.msfsx.cn/86620.Doc
m.qsa.msfsx.cn/28400.Doc
m.qsa.msfsx.cn/73531.Doc
m.qsa.msfsx.cn/60060.Doc
m.qsa.msfsx.cn/06022.Doc
m.qsa.msfsx.cn/02840.Doc
m.qsa.msfsx.cn/40688.Doc
m.qsa.msfsx.cn/79571.Doc
m.qsa.msfsx.cn/28202.Doc
m.qsa.msfsx.cn/68486.Doc
m.qsa.msfsx.cn/08608.Doc
m.qsa.msfsx.cn/53971.Doc
m.qsa.msfsx.cn/88208.Doc
m.qsa.msfsx.cn/06280.Doc
m.qsa.msfsx.cn/04268.Doc
m.qsp.msfsx.cn/04880.Doc
m.qsp.msfsx.cn/20622.Doc
m.qsp.msfsx.cn/26002.Doc
m.qsp.msfsx.cn/44600.Doc
m.qsp.msfsx.cn/40464.Doc
m.qsp.msfsx.cn/68602.Doc
m.qsp.msfsx.cn/00024.Doc
m.qsp.msfsx.cn/28648.Doc
m.qsp.msfsx.cn/40288.Doc
m.qsp.msfsx.cn/60400.Doc
m.qsp.msfsx.cn/53337.Doc
m.qsp.msfsx.cn/73571.Doc
m.qsp.msfsx.cn/71377.Doc
m.qsp.msfsx.cn/17933.Doc
m.qsp.msfsx.cn/00606.Doc
m.qsp.msfsx.cn/88464.Doc
m.qsp.msfsx.cn/64004.Doc
m.qsp.msfsx.cn/60666.Doc
m.qsp.msfsx.cn/02086.Doc
m.qsp.msfsx.cn/26600.Doc
m.qso.msfsx.cn/04688.Doc
m.qso.msfsx.cn/39197.Doc
m.qso.msfsx.cn/46688.Doc
m.qso.msfsx.cn/99717.Doc
m.qso.msfsx.cn/24446.Doc
m.qso.msfsx.cn/04020.Doc
m.qso.msfsx.cn/20482.Doc
m.qso.msfsx.cn/02860.Doc
m.qso.msfsx.cn/46448.Doc
m.qso.msfsx.cn/86862.Doc
m.qso.msfsx.cn/46204.Doc
m.qso.msfsx.cn/44488.Doc
m.qso.msfsx.cn/44406.Doc
