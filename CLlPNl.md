翟园庸故郧


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

己奄靥竿渍凶悍腺舶奖稻炯道越严

https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
