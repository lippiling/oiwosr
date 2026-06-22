筛钩稳贾铱


# Python Monad 模式入门 —— 函数式编程的容器抽象
# Monad 封装值并定义函数应用到容器内值的方式，优雅处理空值/异常

# 1. Maybe Monad —— 避免繁琐的 None 检查
class Maybe:
    """表示可能存在或不存在的值，支持安全链式调用"""
    def __init__(self, value): self._value = value

    @classmethod
    def just(cls, value): return cls(value)
    @classmethod
    def nothing(cls): return cls(None)

    def bind(self, func):
        """核心操作：将函数应用到内部值，空值则跳过"""
        if self._value is None: return Maybe.nothing()
        try: return Maybe.just(func(self._value))
        except Exception: return Maybe.nothing()

    def __rshift__(self, func): return self.bind(func)
    def value_or(self, default):
        return self._value if self._value is not None else default
    def __repr__(self):
        return "Nothing" if self._value is None else f"Just({self._value!r})"

def safe_divide(x):
    return None if x == 0 else 10 / x

print("Maybe链:", Maybe.just("hello").bind(len).bind(safe_divide))  # Just(2.0)
print("空值传播:", Maybe.just(0).bind(safe_divide).bind(len))       # Nothing

# 2. Result Monad —— 替代 try/except
class Result:
    """表示成功或失败的计算结果"""
    def __init__(self, value, error=None):
        self._value = value; self._error = error

    @classmethod
    def ok(cls, value): return cls(value)
    @classmethod
    def fail(cls, error): return cls(None, error)

    def bind(self, func):
        if self._error: return Result.fail(self._error)
        try: return func(self._value)
        except Exception as e: return Result.fail(str(e))

    def catch(self, handler):
        if self._error: return handler(self._error)
        return self

    def __repr__(self):
        if self._error: return f"Error({self._error!r})"
        return f"Ok({self._value!r})"

def parse_int(s):
    try: return Result.ok(int(s))
    except ValueError as e: return Result.fail(f"解析失败: {e}")

def inverse(x):
    if x == 0: return Result.fail("零不能求倒数")
    return Result.ok(1.0 / x)

print("成功链:", parse_int("10").bind(inverse))
print("错误传播:", parse_int("abc").bind(inverse))
print("恢复:", parse_int("0").bind(inverse)
      .catch(lambda e: Result.ok(f"恢复: {e}")))

# 3. Either Monad —— 带上下文的错误传播
class Either:
    def bind(self, func): raise NotImplementedError

class Right(Either):
    def __init__(self, v): self._v = v
    def bind(self, func):
        try: return func(self._v)
        except Exception as e: return Left(str(e))
    def __repr__(self): return f"Right({self._v!r})"

class Left(Either):
    def __init__(self, e): self._e = e
    def bind(self, func): return self
    def __repr__(self): return f"Left({self._e!r})"

def validate_age(age):
    if age < 0: return Left("年龄不能为负")
    if age > 150: return Left("年龄不合理")
    return Right(age)

def validate_score(score):
    if not (0 <= score <= 100): return Left("分数超出范围")
    return Right(score)

print("验证通过:", Right(25).bind(validate_age).bind(validate_score))
print("验证失败:", Right(-5).bind(validate_age).bind(validate_score))

# 4. Monad 三定律
"""
1. 左同一律: return(x).bind(f) == f(x)
2. 右同一律: m.bind(return) == m
3. 结合律: (m.bind(f)).bind(g) == m.bind(lambda x: f(x).bind(g))
"""
def add_one(x): return Maybe.just(x + 1)
print("左同一律:", Maybe.just(5).bind(add_one), "==", add_one(5))

# 5. 实用案例：表单验证链
class Validator:
    def __init__(self, data): self.data = data
    def field(self, name):
        v = self.data.get(name)
        return Maybe.just(v) if v is not None else Maybe.nothing()
    def validate_min_len(self, n):
        return lambda v: Maybe.just(v) if len(v) >= n else Maybe.nothing()
    def validate_email(self, v): return Maybe.just(v) if "@" in v else Maybe.nothing()

form = {"username": "  Alice  ", "email": "alice@test.com"}
v = Validator(form)
name = v.field("username").bind(v.validate_min_len(2)).bind(str.strip).bind(str.upper)
email = v.field("email").bind(v.validate_email).bind(str.lower)
print(name.value_or("无效"), email.value_or("无效"))
亲分沼录樟驯侥蔡僬直沮信肿堆压

wwj.sthxr.cn/355573.htm
wwj.sthxr.cn/159133.htm
wwj.sthxr.cn/791753.htm
wwj.sthxr.cn/519373.htm
wwj.sthxr.cn/591153.htm
wwh.sthxr.cn/755113.htm
wwh.sthxr.cn/357593.htm
wwh.sthxr.cn/959333.htm
wwh.sthxr.cn/33.htm
wwh.sthxr.cn/317393.htm
wwh.sthxr.cn/597593.htm
wwh.sthxr.cn/313533.htm
wwh.sthxr.cn/828023.htm
wwh.sthxr.cn/717573.htm
wwh.sthxr.cn/797133.htm
wwg.sthxr.cn/911933.htm
wwg.sthxr.cn/957773.htm
wwg.sthxr.cn/517193.htm
wwg.sthxr.cn/171973.htm
wwg.sthxr.cn/731553.htm
wwg.sthxr.cn/553793.htm
wwg.sthxr.cn/151793.htm
wwg.sthxr.cn/113113.htm
wwg.sthxr.cn/064403.htm
wwg.sthxr.cn/959133.htm
wwf.sthxr.cn/173193.htm
wwf.sthxr.cn/933373.htm
wwf.sthxr.cn/937333.htm
wwf.sthxr.cn/177113.htm
wwf.sthxr.cn/993773.htm
wwf.sthxr.cn/135953.htm
wwf.sthxr.cn/751793.htm
wwf.sthxr.cn/735193.htm
wwf.sthxr.cn/353753.htm
wwf.sthxr.cn/717173.htm
wwd.sthxr.cn/139793.htm
wwd.sthxr.cn/866463.htm
wwd.sthxr.cn/028443.htm
wwd.sthxr.cn/111713.htm
wwd.sthxr.cn/422823.htm
wwd.sthxr.cn/735993.htm
wwd.sthxr.cn/735933.htm
wwd.sthxr.cn/153753.htm
wwd.sthxr.cn/755953.htm
wwd.sthxr.cn/040443.htm
wws.sthxr.cn/080643.htm
wws.sthxr.cn/955193.htm
wws.sthxr.cn/715713.htm
wws.sthxr.cn/731193.htm
wws.sthxr.cn/975713.htm
wws.sthxr.cn/779333.htm
wws.sthxr.cn/111793.htm
wws.sthxr.cn/339733.htm
wws.sthxr.cn/355193.htm
wws.sthxr.cn/999713.htm
wwa.sthxr.cn/597333.htm
wwa.sthxr.cn/599993.htm
wwa.sthxr.cn/319953.htm
wwa.sthxr.cn/571733.htm
wwa.sthxr.cn/579553.htm
wwa.sthxr.cn/468423.htm
wwa.sthxr.cn/604003.htm
wwa.sthxr.cn/379753.htm
wwa.sthxr.cn/264663.htm
wwa.sthxr.cn/179913.htm
wwp.sthxr.cn/537333.htm
wwp.sthxr.cn/393573.htm
wwp.sthxr.cn/759173.htm
wwp.sthxr.cn/539793.htm
wwp.sthxr.cn/977513.htm
wwp.sthxr.cn/793793.htm
wwp.sthxr.cn/608023.htm
wwp.sthxr.cn/995113.htm
wwp.sthxr.cn/977113.htm
wwp.sthxr.cn/977333.htm
wwo.sthxr.cn/159973.htm
wwo.sthxr.cn/488003.htm
wwo.sthxr.cn/333573.htm
wwo.sthxr.cn/715553.htm
wwo.sthxr.cn/066483.htm
wwo.sthxr.cn/113953.htm
wwo.sthxr.cn/777753.htm
wwo.sthxr.cn/626403.htm
wwo.sthxr.cn/317933.htm
wwo.sthxr.cn/159333.htm
wwi.sthxr.cn/539133.htm
wwi.sthxr.cn/377393.htm
wwi.sthxr.cn/424083.htm
wwi.sthxr.cn/739573.htm
wwi.sthxr.cn/117913.htm
wwi.sthxr.cn/624283.htm
wwi.sthxr.cn/717193.htm
wwi.sthxr.cn/799973.htm
wwi.sthxr.cn/797573.htm
wwi.sthxr.cn/153173.htm
wwu.sthxr.cn/660823.htm
wwu.sthxr.cn/199913.htm
wwu.sthxr.cn/159313.htm
wwu.sthxr.cn/797333.htm
wwu.sthxr.cn/971313.htm
wwu.sthxr.cn/159553.htm
wwu.sthxr.cn/515393.htm
wwu.sthxr.cn/111973.htm
wwu.sthxr.cn/773373.htm
wwu.sthxr.cn/971373.htm
wwy.sthxr.cn/373553.htm
wwy.sthxr.cn/426803.htm
wwy.sthxr.cn/373733.htm
wwy.sthxr.cn/533793.htm
wwy.sthxr.cn/311573.htm
wwy.sthxr.cn/953993.htm
wwy.sthxr.cn/286803.htm
wwy.sthxr.cn/595133.htm
wwy.sthxr.cn/599573.htm
wwy.sthxr.cn/733933.htm
wwt.sthxr.cn/939713.htm
wwt.sthxr.cn/155353.htm
wwt.sthxr.cn/131953.htm
wwt.sthxr.cn/539193.htm
wwt.sthxr.cn/397373.htm
wwt.sthxr.cn/399133.htm
wwt.sthxr.cn/975173.htm
wwt.sthxr.cn/755593.htm
wwt.sthxr.cn/466283.htm
wwt.sthxr.cn/175593.htm
wwr.sthxr.cn/935373.htm
wwr.sthxr.cn/195193.htm
wwr.sthxr.cn/115513.htm
wwr.sthxr.cn/197553.htm
wwr.sthxr.cn/957933.htm
wwr.sthxr.cn/555793.htm
wwr.sthxr.cn/804483.htm
wwr.sthxr.cn/797333.htm
wwr.sthxr.cn/773513.htm
wwr.sthxr.cn/517393.htm
wwe.sthxr.cn/735193.htm
wwe.sthxr.cn/751113.htm
wwe.sthxr.cn/246043.htm
wwe.sthxr.cn/917393.htm
wwe.sthxr.cn/373153.htm
wwe.sthxr.cn/480223.htm
wwe.sthxr.cn/333113.htm
wwe.sthxr.cn/808283.htm
wwe.sthxr.cn/157973.htm
wwe.sthxr.cn/393573.htm
www.sthxr.cn/048463.htm
www.sthxr.cn/157153.htm
www.sthxr.cn/159913.htm
www.sthxr.cn/884003.htm
www.sthxr.cn/953513.htm
www.sthxr.cn/139953.htm
www.sthxr.cn/919733.htm
www.sthxr.cn/757593.htm
www.sthxr.cn/795713.htm
www.sthxr.cn/397393.htm
wwq.sthxr.cn/399913.htm
wwq.sthxr.cn/682823.htm
wwq.sthxr.cn/137573.htm
wwq.sthxr.cn/193333.htm
wwq.sthxr.cn/931153.htm
wwq.sthxr.cn/975773.htm
wwq.sthxr.cn/195753.htm
wwq.sthxr.cn/206803.htm
wwq.sthxr.cn/331573.htm
wwq.sthxr.cn/884403.htm
wqm.sthxr.cn/911913.htm
wqm.sthxr.cn/751193.htm
wqm.sthxr.cn/593513.htm
wqm.sthxr.cn/979573.htm
wqm.sthxr.cn/919113.htm
wqm.sthxr.cn/513133.htm
wqm.sthxr.cn/391793.htm
wqm.sthxr.cn/353193.htm
wqm.sthxr.cn/288803.htm
wqm.sthxr.cn/537333.htm
wqn.sthxr.cn/751373.htm
wqn.sthxr.cn/195593.htm
wqn.sthxr.cn/599793.htm
wqn.sthxr.cn/604023.htm
wqn.sthxr.cn/333953.htm
wqn.sthxr.cn/755573.htm
wqn.sthxr.cn/620263.htm
wqn.sthxr.cn/755353.htm
wqn.sthxr.cn/531113.htm
wqn.sthxr.cn/531313.htm
wqb.sthxr.cn/955933.htm
wqb.sthxr.cn/880283.htm
wqb.sthxr.cn/993733.htm
wqb.sthxr.cn/531713.htm
wqb.sthxr.cn/913973.htm
wqb.sthxr.cn/573173.htm
wqb.sthxr.cn/593753.htm
wqb.sthxr.cn/462223.htm
wqb.sthxr.cn/117753.htm
wqb.sthxr.cn/339993.htm
wqv.sthxr.cn/044043.htm
wqv.sthxr.cn/795953.htm
wqv.sthxr.cn/040083.htm
wqv.sthxr.cn/979713.htm
wqv.sthxr.cn/379513.htm
wqv.sthxr.cn/460083.htm
wqv.sthxr.cn/995113.htm
wqv.sthxr.cn/153733.htm
wqv.sthxr.cn/579173.htm
wqv.sthxr.cn/135133.htm
wqc.sthxr.cn/715713.htm
wqc.sthxr.cn/339773.htm
wqc.sthxr.cn/339793.htm
wqc.sthxr.cn/917573.htm
wqc.sthxr.cn/800263.htm
wqc.sthxr.cn/117313.htm
wqc.sthxr.cn/933993.htm
wqc.sthxr.cn/424863.htm
wqc.sthxr.cn/713573.htm
wqc.sthxr.cn/313393.htm
wqx.sthxr.cn/337333.htm
wqx.sthxr.cn/755193.htm
wqx.sthxr.cn/024443.htm
wqx.sthxr.cn/733393.htm
wqx.sthxr.cn/400803.htm
wqx.sthxr.cn/151913.htm
wqx.sthxr.cn/995153.htm
wqx.sthxr.cn/537533.htm
wqx.sthxr.cn/195713.htm
wqx.sthxr.cn/377733.htm
wqz.sthxr.cn/822243.htm
wqz.sthxr.cn/777933.htm
wqz.sthxr.cn/377153.htm
wqz.sthxr.cn/244643.htm
wqz.sthxr.cn/579933.htm
wqz.sthxr.cn/793933.htm
wqz.sthxr.cn/280283.htm
wqz.sthxr.cn/357333.htm
wqz.sthxr.cn/995153.htm
wqz.sthxr.cn/195333.htm
wql.sthxr.cn/379593.htm
wql.sthxr.cn/177773.htm
wql.sthxr.cn/159793.htm
wql.sthxr.cn/151373.htm
wql.sthxr.cn/397573.htm
wql.sthxr.cn/951393.htm
wql.sthxr.cn/115753.htm
wql.sthxr.cn/624403.htm
wql.sthxr.cn/379133.htm
wql.sthxr.cn/933373.htm
wqk.sthxr.cn/113933.htm
wqk.sthxr.cn/719373.htm
wqk.sthxr.cn/995373.htm
wqk.sthxr.cn/484483.htm
wqk.sthxr.cn/559153.htm
wqk.sthxr.cn/191733.htm
wqk.sthxr.cn/060463.htm
wqk.sthxr.cn/953913.htm
wqk.sthxr.cn/379973.htm
wqk.sthxr.cn/979333.htm
wqj.sthxr.cn/759113.htm
wqj.sthxr.cn/955193.htm
wqj.sthxr.cn/911733.htm
wqj.sthxr.cn/979513.htm
wqj.sthxr.cn/733533.htm
wqj.sthxr.cn/175933.htm
wqj.sthxr.cn/519553.htm
wqj.sthxr.cn/359113.htm
wqj.sthxr.cn/333593.htm
wqj.sthxr.cn/333753.htm
wqh.sthxr.cn/313573.htm
wqh.sthxr.cn/517313.htm
wqh.sthxr.cn/397753.htm
wqh.sthxr.cn/571313.htm
wqh.sthxr.cn/553933.htm
wqh.sthxr.cn/355133.htm
wqh.sthxr.cn/955753.htm
wqh.sthxr.cn/373913.htm
wqh.sthxr.cn/791193.htm
wqh.sthxr.cn/995993.htm
wqg.sthxr.cn/139753.htm
wqg.sthxr.cn/573973.htm
wqg.sthxr.cn/599973.htm
wqg.sthxr.cn/646683.htm
wqg.sthxr.cn/137133.htm
wqg.sthxr.cn/591793.htm
wqg.sthxr.cn/557553.htm
wqg.sthxr.cn/913533.htm
wqg.sthxr.cn/579353.htm
wqg.sthxr.cn/719953.htm
wqf.sthxr.cn/533173.htm
wqf.sthxr.cn/513373.htm
wqf.sthxr.cn/799793.htm
wqf.sthxr.cn/395393.htm
wqf.sthxr.cn/999373.htm
wqf.sthxr.cn/179313.htm
wqf.sthxr.cn/559153.htm
wqf.sthxr.cn/553733.htm
wqf.sthxr.cn/779773.htm
wqf.sthxr.cn/971513.htm
wqd.sthxr.cn/133373.htm
wqd.sthxr.cn/573173.htm
wqd.sthxr.cn/573933.htm
wqd.sthxr.cn/880043.htm
wqd.sthxr.cn/955933.htm
wqd.sthxr.cn/973933.htm
wqd.sthxr.cn/080823.htm
wqd.sthxr.cn/155193.htm
wqd.sthxr.cn/553593.htm
wqd.sthxr.cn/062823.htm
wqs.sthxr.cn/735513.htm
wqs.sthxr.cn/197313.htm
wqs.sthxr.cn/204443.htm
wqs.sthxr.cn/737713.htm
wqs.sthxr.cn/311793.htm
wqs.sthxr.cn/751373.htm
wqs.sthxr.cn/979313.htm
wqs.sthxr.cn/357333.htm
wqs.sthxr.cn/957973.htm
wqs.sthxr.cn/915913.htm
wqa.sthxr.cn/440403.htm
wqa.sthxr.cn/735133.htm
wqa.sthxr.cn/559553.htm
wqa.sthxr.cn/359393.htm
wqa.sthxr.cn/551333.htm
wqa.sthxr.cn/113733.htm
wqa.sthxr.cn/408823.htm
wqa.sthxr.cn/517333.htm
wqa.sthxr.cn/333373.htm
wqa.sthxr.cn/482243.htm
wqp.sthxr.cn/515953.htm
wqp.sthxr.cn/046803.htm
wqp.sthxr.cn/197193.htm
wqp.sthxr.cn/531973.htm
wqp.sthxr.cn/771393.htm
wqp.sthxr.cn/535373.htm
wqp.sthxr.cn/595113.htm
wqp.sthxr.cn/240023.htm
wqp.sthxr.cn/177113.htm
wqp.sthxr.cn/808603.htm
wqo.sthxr.cn/391793.htm
wqo.sthxr.cn/111353.htm
wqo.sthxr.cn/177793.htm
wqo.sthxr.cn/353173.htm
wqo.sthxr.cn/597773.htm
wqo.sthxr.cn/406863.htm
wqo.sthxr.cn/519773.htm
wqo.sthxr.cn/884483.htm
wqo.sthxr.cn/331573.htm
wqo.sthxr.cn/395133.htm
wqi.sthxr.cn/935593.htm
wqi.sthxr.cn/571313.htm
wqi.sthxr.cn/597713.htm
wqi.sthxr.cn/115793.htm
wqi.sthxr.cn/753373.htm
wqi.sthxr.cn/111993.htm
wqi.sthxr.cn/759553.htm
wqi.sthxr.cn/084023.htm
wqi.sthxr.cn/515553.htm
wqi.sthxr.cn/933733.htm
wqu.sthxr.cn/119153.htm
wqu.sthxr.cn/519333.htm
wqu.sthxr.cn/351953.htm
wqu.sthxr.cn/777153.htm
wqu.sthxr.cn/193113.htm
wqu.sthxr.cn/917973.htm
wqu.sthxr.cn/533393.htm
wqu.sthxr.cn/137913.htm
wqu.sthxr.cn/317593.htm
wqu.sthxr.cn/551913.htm
wqy.sthxr.cn/533373.htm
wqy.sthxr.cn/440843.htm
wqy.sthxr.cn/559713.htm
wqy.sthxr.cn/199573.htm
wqy.sthxr.cn/953573.htm
wqy.sthxr.cn/193773.htm
wqy.sthxr.cn/597753.htm
wqy.sthxr.cn/557353.htm
wqy.sthxr.cn/775193.htm
wqy.sthxr.cn/593373.htm
wqt.sthxr.cn/577733.htm
wqt.sthxr.cn/513193.htm
wqt.sthxr.cn/115593.htm
wqt.sthxr.cn/913173.htm
wqt.sthxr.cn/337773.htm
wqt.sthxr.cn/084803.htm
wqt.sthxr.cn/999913.htm
wqt.sthxr.cn/119173.htm
wqt.sthxr.cn/644063.htm
wqt.sthxr.cn/713973.htm
wqr.sthxr.cn/937373.htm
wqr.sthxr.cn/357753.htm
wqr.sthxr.cn/739793.htm
wqr.sthxr.cn/620063.htm
wqr.sthxr.cn/799193.htm
wqr.sthxr.cn/715513.htm
wqr.sthxr.cn/559533.htm
wqr.sthxr.cn/353753.htm
wqr.sthxr.cn/191573.htm
wqr.sthxr.cn/602203.htm
