赵冻焚眯椿


# Python 策略模式 (Strategy Pattern)
# Strategy 接口，运行时选择策略，策略注册表，验证策略
# -----------------------------------------------------------
# 策略模式定义一系列可互换的算法，使算法独立于使用它的
# 客户端。核心：策略接口 + 具体策略 + 策略上下文。
# -----------------------------------------------------------

from abc import ABC, abstractmethod
from typing import Any

# ===================== 策略接口 =====================
class ValidationStrategy(ABC):
    @abstractmethod
    def validate(self, data: dict) -> list[str]: ...

# ===================== 具体策略 =====================
class RequiredFieldStrategy(ValidationStrategy):
    def __init__(self, fields: list[str]):
        self._fields = fields
    def validate(self, data: dict) -> list[str]:
        return [f"{f} 为必填项" for f in self._fields
                if f not in data or not data.get(f)]

class RangeStrategy(ValidationStrategy):
    def __init__(self, field: str, min_v: float, max_v: float):
        self._field = field; self._min = min_v; self._max = max_v
    def validate(self, data: dict) -> list[str]:
        v = data.get(self._field)
        if v is not None:
            try:
                fv = float(v)
                if fv < self._min or fv > self._max:
                    return [f"{self._field} 需在 {self._min}-{self._max}"]
            except (ValueError, TypeError):
                return [f"{self._field} 必须为数字"]
        return []

class EmailStrategy(ValidationStrategy):
    def validate(self, data: dict) -> list[str]:
        email = data.get("email", "")
        return ["邮箱格式无效"] if email and "@" not in email else []

class LengthStrategy(ValidationStrategy):
    def __init__(self, field: str, min_l: int, max_l: int):
        self._field = field; self._min = min_l; self._max = max_l
    def validate(self, data: dict) -> list[str]:
        val = str(data.get(self._field, ""))
        if not (self._min <= len(val) <= self._max):
            return [f"{self._field} 长度需 {self._min}-{self._max} 字符"]
        return []

# ===================== 策略上下文 =====================
class Validator:
    """验证上下文：组合多个策略进行校验"""
    def __init__(self):
        self._strategies: list[ValidationStrategy] = []
    def add(self, s: ValidationStrategy) -> "Validator":
        self._strategies.append(s); return self
    def validate(self, data: dict) -> list[str]:
        errors = []
        for s in self._strategies:
            errors.extend(s.validate(data))
        return errors

# ===================== 策略注册表 =====================
class StrategyRegistry:
    """按名称动态查找和创建策略"""
    def __init__(self):
        self._reg: dict[str, type[ValidationStrategy]] = {}
    def register(self, n: str, s: type[ValidationStrategy]) -> None:
        self._reg[n] = s
    def create(self, n: str, **kw) -> ValidationStrategy:
        cls = self._reg.get(n)
        if cls is None:
            raise ValueError(f"未知策略: {n}")
        return cls(**kw)

if __name__ == "__main__":
    v = Validator()
    v.add(RequiredFieldStrategy(["name", "email", "age"])) \
     .add(EmailStrategy()) \
     .add(RangeStrategy("age", 18, 120)) \
     .add(LengthStrategy("name", 2, 50))
    errs = v.validate({"name": "A", "email": "bad", "age": "200"})
    for e in errs:
        print(f"验证失败: {e}")

欠卸渭压覆睦怖舜履腿谛巢恐砍山

ede.mmmxz.cn/311793.htm
ede.mmmxz.cn/759773.htm
ede.mmmxz.cn/444803.htm
ede.mmmxz.cn/337993.htm
ede.mmmxz.cn/397953.htm
edw.mmmxz.cn/599553.htm
edw.mmmxz.cn/737373.htm
edw.mmmxz.cn/864003.htm
edw.mmmxz.cn/488463.htm
edw.mmmxz.cn/626843.htm
edw.mmmxz.cn/379993.htm
edw.mmmxz.cn/375773.htm
edw.mmmxz.cn/464023.htm
edw.mmmxz.cn/935133.htm
edw.mmmxz.cn/175313.htm
edq.mmmxz.cn/975753.htm
edq.mmmxz.cn/513153.htm
edq.mmmxz.cn/911393.htm
edq.mmmxz.cn/517153.htm
edq.mmmxz.cn/959993.htm
edq.mmmxz.cn/551173.htm
edq.mmmxz.cn/373133.htm
edq.mmmxz.cn/159553.htm
edq.mmmxz.cn/373313.htm
edq.mmmxz.cn/911553.htm
esm.mmmxz.cn/404483.htm
esm.mmmxz.cn/408463.htm
esm.mmmxz.cn/577773.htm
esm.mmmxz.cn/117793.htm
esm.mmmxz.cn/602063.htm
esm.mmmxz.cn/862003.htm
esm.mmmxz.cn/977973.htm
esm.mmmxz.cn/199193.htm
esm.mmmxz.cn/911913.htm
esm.mmmxz.cn/591753.htm
esn.mmmxz.cn/022643.htm
esn.mmmxz.cn/577773.htm
esn.mmmxz.cn/862003.htm
esn.mmmxz.cn/939733.htm
esn.mmmxz.cn/955193.htm
esn.mmmxz.cn/551593.htm
esn.mmmxz.cn/997533.htm
esn.mmmxz.cn/719733.htm
esn.mmmxz.cn/628203.htm
esn.mmmxz.cn/848623.htm
esb.mmmxz.cn/880283.htm
esb.mmmxz.cn/599153.htm
esb.mmmxz.cn/759953.htm
esb.mmmxz.cn/517733.htm
esb.mmmxz.cn/080823.htm
esb.mmmxz.cn/113713.htm
esb.mmmxz.cn/339553.htm
esb.mmmxz.cn/137973.htm
esb.mmmxz.cn/777313.htm
esb.mmmxz.cn/175153.htm
esv.mmmxz.cn/206063.htm
esv.mmmxz.cn/335113.htm
esv.mmmxz.cn/286063.htm
esv.mmmxz.cn/193153.htm
esv.mmmxz.cn/375753.htm
esv.mmmxz.cn/424063.htm
esv.mmmxz.cn/428063.htm
esv.mmmxz.cn/955333.htm
esv.mmmxz.cn/317193.htm
esv.mmmxz.cn/911533.htm
esc.mmmxz.cn/664463.htm
esc.mmmxz.cn/682443.htm
esc.mmmxz.cn/048203.htm
esc.mmmxz.cn/759113.htm
esc.mmmxz.cn/993533.htm
esc.mmmxz.cn/886063.htm
esc.mmmxz.cn/462463.htm
esc.mmmxz.cn/246643.htm
esc.mmmxz.cn/337113.htm
esc.mmmxz.cn/313133.htm
esx.mmmxz.cn/284283.htm
esx.mmmxz.cn/795193.htm
esx.mmmxz.cn/399513.htm
esx.mmmxz.cn/375753.htm
esx.mmmxz.cn/973533.htm
esx.mmmxz.cn/666223.htm
esx.mmmxz.cn/313733.htm
esx.mmmxz.cn/991553.htm
esx.mmmxz.cn/862863.htm
esx.mmmxz.cn/319773.htm
esz.mmmxz.cn/448863.htm
esz.mmmxz.cn/133373.htm
esz.mmmxz.cn/311553.htm
esz.mmmxz.cn/204203.htm
esz.mmmxz.cn/979953.htm
esz.mmmxz.cn/804623.htm
esz.mmmxz.cn/797973.htm
esz.mmmxz.cn/131553.htm
esz.mmmxz.cn/597753.htm
esz.mmmxz.cn/800643.htm
esl.mmmxz.cn/357133.htm
esl.mmmxz.cn/137913.htm
esl.mmmxz.cn/557973.htm
esl.mmmxz.cn/202043.htm
esl.mmmxz.cn/951133.htm
esl.mmmxz.cn/000003.htm
esl.mmmxz.cn/517353.htm
esl.mmmxz.cn/117793.htm
esl.mmmxz.cn/999393.htm
esl.mmmxz.cn/575993.htm
esk.mmmxz.cn/466883.htm
esk.mmmxz.cn/171153.htm
esk.mmmxz.cn/628883.htm
esk.mmmxz.cn/551973.htm
esk.mmmxz.cn/937793.htm
esk.mmmxz.cn/593193.htm
esk.mmmxz.cn/626283.htm
esk.mmmxz.cn/797773.htm
esk.mmmxz.cn/866003.htm
esk.mmmxz.cn/751193.htm
esj.mmmxz.cn/604643.htm
esj.mmmxz.cn/371713.htm
esj.mmmxz.cn/579793.htm
esj.mmmxz.cn/539193.htm
esj.mmmxz.cn/175193.htm
esj.mmmxz.cn/911933.htm
esj.mmmxz.cn/999513.htm
esj.mmmxz.cn/139713.htm
esj.mmmxz.cn/064243.htm
esj.mmmxz.cn/391793.htm
esh.mmmxz.cn/868283.htm
esh.mmmxz.cn/913393.htm
esh.mmmxz.cn/626083.htm
esh.mmmxz.cn/804043.htm
esh.mmmxz.cn/224063.htm
esh.mmmxz.cn/913593.htm
esh.mmmxz.cn/882283.htm
esh.mmmxz.cn/971393.htm
esh.mmmxz.cn/571573.htm
esh.mmmxz.cn/260623.htm
esg.mmmxz.cn/791553.htm
esg.mmmxz.cn/022463.htm
esg.mmmxz.cn/971153.htm
esg.mmmxz.cn/135513.htm
esg.mmmxz.cn/371373.htm
esg.mmmxz.cn/913713.htm
esg.mmmxz.cn/004023.htm
esg.mmmxz.cn/759933.htm
esg.mmmxz.cn/799793.htm
esg.mmmxz.cn/288423.htm
esf.mmmxz.cn/246203.htm
esf.mmmxz.cn/286223.htm
esf.mmmxz.cn/377933.htm
esf.mmmxz.cn/177733.htm
esf.mmmxz.cn/315133.htm
esf.mmmxz.cn/959993.htm
esf.mmmxz.cn/117133.htm
esf.mmmxz.cn/379513.htm
esf.mmmxz.cn/771393.htm
esf.mmmxz.cn/155373.htm
esd.mmmxz.cn/113993.htm
esd.mmmxz.cn/579533.htm
esd.mmmxz.cn/119733.htm
esd.mmmxz.cn/173113.htm
esd.mmmxz.cn/733593.htm
esd.mmmxz.cn/139113.htm
esd.mmmxz.cn/482463.htm
esd.mmmxz.cn/119113.htm
esd.mmmxz.cn/591553.htm
esd.mmmxz.cn/911773.htm
ess.mmmxz.cn/377973.htm
ess.mmmxz.cn/537993.htm
ess.mmmxz.cn/957533.htm
ess.mmmxz.cn/640803.htm
ess.mmmxz.cn/155393.htm
ess.mmmxz.cn/535313.htm
ess.mmmxz.cn/260403.htm
ess.mmmxz.cn/957733.htm
ess.mmmxz.cn/286663.htm
ess.mmmxz.cn/608643.htm
esa.mmmxz.cn/335713.htm
esa.mmmxz.cn/024823.htm
esa.mmmxz.cn/955193.htm
esa.mmmxz.cn/408483.htm
esa.mmmxz.cn/739513.htm
esa.mmmxz.cn/884643.htm
esa.mmmxz.cn/979933.htm
esa.mmmxz.cn/793573.htm
esa.mmmxz.cn/551953.htm
esa.mmmxz.cn/177753.htm
esp.mmmxz.cn/155193.htm
esp.mmmxz.cn/464483.htm
esp.mmmxz.cn/979773.htm
esp.mmmxz.cn/642803.htm
esp.mmmxz.cn/197553.htm
esp.mmmxz.cn/517353.htm
esp.mmmxz.cn/488023.htm
esp.mmmxz.cn/119793.htm
esp.mmmxz.cn/202443.htm
esp.mmmxz.cn/842223.htm
eso.mmmxz.cn/351773.htm
eso.mmmxz.cn/686003.htm
eso.mmmxz.cn/131333.htm
eso.mmmxz.cn/555713.htm
eso.mmmxz.cn/739713.htm
eso.mmmxz.cn/519133.htm
eso.mmmxz.cn/440803.htm
eso.mmmxz.cn/997953.htm
eso.mmmxz.cn/808043.htm
eso.mmmxz.cn/735913.htm
esi.mmmxz.cn/260863.htm
esi.mmmxz.cn/468243.htm
esi.mmmxz.cn/753953.htm
esi.mmmxz.cn/002003.htm
esi.mmmxz.cn/795593.htm
esi.mmmxz.cn/579373.htm
esi.mmmxz.cn/315373.htm
esi.mmmxz.cn/268623.htm
esi.mmmxz.cn/599353.htm
esi.mmmxz.cn/33.htm
esu.mmmxz.cn/319373.htm
esu.mmmxz.cn/284603.htm
esu.mmmxz.cn/739733.htm
esu.mmmxz.cn/606483.htm
esu.mmmxz.cn/737313.htm
esu.mmmxz.cn/719393.htm
esu.mmmxz.cn/228843.htm
esu.mmmxz.cn/139133.htm
esu.mmmxz.cn/599973.htm
esu.mmmxz.cn/515133.htm
esy.mmmxz.cn/199753.htm
esy.mmmxz.cn/428223.htm
esy.mmmxz.cn/315593.htm
esy.mmmxz.cn/804883.htm
esy.mmmxz.cn/626403.htm
esy.mmmxz.cn/993573.htm
esy.mmmxz.cn/175133.htm
esy.mmmxz.cn/339153.htm
esy.mmmxz.cn/824003.htm
esy.mmmxz.cn/713773.htm
est.mmmxz.cn/220263.htm
est.mmmxz.cn/804803.htm
est.mmmxz.cn/357713.htm
est.mmmxz.cn/535553.htm
est.mmmxz.cn/993793.htm
est.mmmxz.cn/915553.htm
est.mmmxz.cn/884003.htm
est.mmmxz.cn/197713.htm
est.mmmxz.cn/397973.htm
est.mmmxz.cn/599133.htm
esr.mmmxz.cn/642403.htm
esr.mmmxz.cn/664003.htm
esr.mmmxz.cn/739313.htm
esr.mmmxz.cn/757113.htm
esr.mmmxz.cn/199193.htm
esr.mmmxz.cn/711173.htm
esr.mmmxz.cn/359393.htm
esr.mmmxz.cn/408023.htm
esr.mmmxz.cn/977393.htm
esr.mmmxz.cn/266283.htm
ese.mmmxz.cn/331773.htm
ese.mmmxz.cn/173373.htm
ese.mmmxz.cn/820423.htm
ese.mmmxz.cn/539373.htm
ese.mmmxz.cn/339193.htm
ese.mmmxz.cn/157533.htm
ese.mmmxz.cn/628083.htm
ese.mmmxz.cn/977333.htm
ese.mmmxz.cn/840243.htm
ese.mmmxz.cn/424623.htm
esw.mmmxz.cn/377993.htm
esw.mmmxz.cn/680623.htm
esw.mmmxz.cn/153793.htm
esw.mmmxz.cn/775113.htm
esw.mmmxz.cn/777713.htm
esw.mmmxz.cn/151933.htm
esw.mmmxz.cn/000283.htm
esw.mmmxz.cn/333793.htm
esw.mmmxz.cn/315793.htm
esw.mmmxz.cn/797333.htm
esq.mmmxz.cn/935773.htm
esq.mmmxz.cn/771793.htm
esq.mmmxz.cn/171793.htm
esq.mmmxz.cn/793753.htm
esq.mmmxz.cn/53.htm
esq.mmmxz.cn/353953.htm
esq.mmmxz.cn/359913.htm
esq.mmmxz.cn/466863.htm
esq.mmmxz.cn/288423.htm
esq.mmmxz.cn/517173.htm
eam.mmmxz.cn/600283.htm
eam.mmmxz.cn/717733.htm
eam.mmmxz.cn/377353.htm
eam.mmmxz.cn/000603.htm
eam.mmmxz.cn/577393.htm
eam.mmmxz.cn/222203.htm
eam.mmmxz.cn/755173.htm
eam.mmmxz.cn/571753.htm
eam.mmmxz.cn/226823.htm
eam.mmmxz.cn/735973.htm
ean.mmmxz.cn/711713.htm
ean.mmmxz.cn/937713.htm
ean.mmmxz.cn/133373.htm
ean.mmmxz.cn/995113.htm
ean.mmmxz.cn/795193.htm
ean.mmmxz.cn/739113.htm
ean.mmmxz.cn/517313.htm
ean.mmmxz.cn/139173.htm
ean.mmmxz.cn/771513.htm
ean.mmmxz.cn/759713.htm
eab.mmmxz.cn/555573.htm
eab.mmmxz.cn/913593.htm
eab.mmmxz.cn/551553.htm
eab.mmmxz.cn/797993.htm
eab.mmmxz.cn/915913.htm
eab.mmmxz.cn/040203.htm
eab.mmmxz.cn/331313.htm
eab.mmmxz.cn/064423.htm
eab.mmmxz.cn/955333.htm
eab.mmmxz.cn/424803.htm
eav.mmmxz.cn/204263.htm
eav.mmmxz.cn/559353.htm
eav.mmmxz.cn/046223.htm
eav.mmmxz.cn/028243.htm
eav.mmmxz.cn/559133.htm
eav.mmmxz.cn/155553.htm
eav.mmmxz.cn/179513.htm
eav.mmmxz.cn/802243.htm
eav.mmmxz.cn/733533.htm
eav.mmmxz.cn/117993.htm
eac.mmmxz.cn/973173.htm
eac.mmmxz.cn/757913.htm
eac.mmmxz.cn/884423.htm
eac.mmmxz.cn/997113.htm
eac.mmmxz.cn/555793.htm
eac.mmmxz.cn/442043.htm
eac.mmmxz.cn/686063.htm
eac.mmmxz.cn/468203.htm
eac.mmmxz.cn/955193.htm
eac.mmmxz.cn/911193.htm
eax.mmmxz.cn/115373.htm
eax.mmmxz.cn/933973.htm
eax.mmmxz.cn/599133.htm
eax.mmmxz.cn/733193.htm
eax.mmmxz.cn/826003.htm
eax.mmmxz.cn/797793.htm
eax.mmmxz.cn/135593.htm
eax.mmmxz.cn/137373.htm
eax.mmmxz.cn/117713.htm
eax.mmmxz.cn/642883.htm
eaz.mmmxz.cn/715313.htm
eaz.mmmxz.cn/111313.htm
eaz.mmmxz.cn/799753.htm
eaz.mmmxz.cn/793953.htm
eaz.mmmxz.cn/480643.htm
eaz.mmmxz.cn/173353.htm
eaz.mmmxz.cn/846243.htm
eaz.mmmxz.cn/086243.htm
eaz.mmmxz.cn/753933.htm
eaz.mmmxz.cn/648403.htm
eal.sthxr.cn/668443.htm
eal.sthxr.cn/519393.htm
eal.sthxr.cn/202003.htm
eal.sthxr.cn/979753.htm
eal.sthxr.cn/559173.htm
eal.sthxr.cn/331133.htm
eal.sthxr.cn/424223.htm
eal.sthxr.cn/335553.htm
eal.sthxr.cn/551113.htm
eal.sthxr.cn/226603.htm
eak.sthxr.cn/591793.htm
eak.sthxr.cn/468823.htm
eak.sthxr.cn/088623.htm
eak.sthxr.cn/111573.htm
eak.sthxr.cn/357773.htm
eak.sthxr.cn/931393.htm
eak.sthxr.cn/119153.htm
eak.sthxr.cn/779713.htm
eak.sthxr.cn/262443.htm
eak.sthxr.cn/933333.htm
eaj.sthxr.cn/953773.htm
eaj.sthxr.cn/284603.htm
eaj.sthxr.cn/779393.htm
eaj.sthxr.cn/040443.htm
eaj.sthxr.cn/513313.htm
eaj.sthxr.cn/531353.htm
eaj.sthxr.cn/866243.htm
eaj.sthxr.cn/799753.htm
eaj.sthxr.cn/391573.htm
eaj.sthxr.cn/884843.htm
eah.sthxr.cn/335733.htm
eah.sthxr.cn/222043.htm
eah.sthxr.cn/799913.htm
eah.sthxr.cn/006603.htm
eah.sthxr.cn/575373.htm
eah.sthxr.cn/933973.htm
eah.sthxr.cn/979173.htm
eah.sthxr.cn/519773.htm
eah.sthxr.cn/973553.htm
eah.sthxr.cn/357513.htm
