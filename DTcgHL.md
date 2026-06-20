檬眉握纶啥


# Python 函数组合 —— 构建数据处理流水线
# 函数组合将多个简单函数串联成复杂操作，提升模块化与可复用性

from functools import reduce

# 1. compose —— 从右到左组合
def compose(*funcs):
    """compose(f, g, h)(x) = f(g(h(x))) —— 从右向左应用"""
    def composed(x):
        result = x
        for func in reversed(funcs):
            result = func(result)
        return result
    return composed

# 2. pipe —— 从左到右组合（更符合直觉）
def pipe(*funcs):
    """pipe(f, g, h)(x) = h(g(f(x))) —— 从左向右应用"""
    def piped(x):
        result = x
        for func in funcs:
            result = func(result)
        return result
    return piped

# 基础函数定义
def double(x): return x * 2
def increment(x): return x + 1
def square(x): return x ** 2

# 测试两种组合方式
f1 = compose(double, increment, square)
f2 = pipe(double, increment, square)
print("compose(3):", f1(3))  # square(3)=9 -> inc(10) -> double: 20
print("pipe(3):", f2(3))     # double(3)=6 -> inc(7) -> square: 49

# 3. 组合 map/filter 构建数据流水线
def is_even(n): return n % 2 == 0
def multiply_by(factor):
    """闭包工厂：返回乘法函数"""
    return lambda x: x * factor

pipeline = pipe(
    lambda nums: filter(is_even, nums),          # 筛选偶数
    lambda evens: map(multiply_by(10), evens),   # 乘以 10
    lambda tens: map(str, tens),                 # 转字符串
    lambda strs: ", ".join(strs),                # 拼接
)
print("流水线:", pipeline(range(1, 11)))  # 20, 40, 60, 80, 100

# 4. 可调用类实现函数组合
class Compose:
    """支持链式调用的组合器"""
    def __init__(self, *funcs):
        self._funcs = list(funcs)
    def then(self, func):
        self._funcs.append(func); return self
    def __call__(self, x):
        for f in self._funcs: x = f(x)
        return x
    def __repr__(self):
        return f"Pipeline({' -> '.join(f.__name__ for f in self._funcs)})"

def remove_punctuation(text):
    import string
    for ch in string.punctuation:
        text = text.replace(ch, " ")
    return text

pl = Compose(remove_punctuation, str.lower, lambda s: len(s.split()))
pl = pl.then(lambda n: f"单词数: {n}")
print(pl("Hello, World! Python."))  # 单词数: 3

# 5. toolz 风格管道（标准库实现）
from functools import partial

def pluck(key, items):
    """从字典列表提取字段"""
    return (item[key] for item in items)

def where(condition, items):
    """按条件过滤"""
    return filter(condition, items)

data = [{"name": "Alice", "score": 85},
        {"name": "Bob", "score": 92},
        {"name": "Charlie", "score": 78}]

high_scores = pipe(
    lambda s: where(lambda x: x["score"] >= 80, s),
    lambda s: pluck("name", s), list,
)(data)
print("高分学生:", high_scores)  # ['Alice', 'Bob']

# 6. 完整 ETL 流水线案例
def etl_pipeline(source):
    """ETL: 提取 -> 转换 -> 过滤 -> 聚合"""
    def extract(raw):
        return (item.strip() for item in raw if item.strip())
    def transform(items):
        for item in items:
            parts = item.split(",")
            yield {"id": int(parts[0]), "value": float(parts[1]),
                   "tag": parts[2].strip()}
    def validate(records):
        return (r for r in records if r["value"] > 0)
    def aggregate(records):
        stats = {}
        for r in records:
            tag = r["tag"]
            stats.setdefault(tag, {"count": 0, "total": 0.0})
            stats[tag]["count"] += 1
            stats[tag]["total"] += r["value"]
        return stats
    return pipe(extract, transform, validate, aggregate)(source)

raw = ["1,100.5,电子", "2,-10.0,食品", "3,250.0,电子",
       "   ", "4,89.0,食品"]
print("ETL:", etl_pipeline(raw))

寻被菊感谇补笔粤却于锻叭匣陕仍

yve.aira2hc.cn/593315.htm
yve.aira2hc.cn/391775.htm
yve.aira2hc.cn/399175.htm
yvw.lmlgyi9.cn/155795.htm
yvw.lmlgyi9.cn/155135.htm
yvw.lmlgyi9.cn/559775.htm
yvw.lmlgyi9.cn/371335.htm
yvw.lmlgyi9.cn/335735.htm
yvw.lmlgyi9.cn/111975.htm
yvw.lmlgyi9.cn/351555.htm
yvw.lmlgyi9.cn/591335.htm
yvw.lmlgyi9.cn/139915.htm
yvw.lmlgyi9.cn/735395.htm
yvq.lmlgyi9.cn/533555.htm
yvq.lmlgyi9.cn/733195.htm
yvq.lmlgyi9.cn/591535.htm
yvq.lmlgyi9.cn/597155.htm
yvq.lmlgyi9.cn/113735.htm
yvq.lmlgyi9.cn/199755.htm
yvq.lmlgyi9.cn/953575.htm
yvq.lmlgyi9.cn/395735.htm
yvq.lmlgyi9.cn/179575.htm
yvq.lmlgyi9.cn/715775.htm
yctv.lmlgyi9.cn/395135.htm
yctv.lmlgyi9.cn/311155.htm
yctv.lmlgyi9.cn/331315.htm
yctv.lmlgyi9.cn/593575.htm
yctv.lmlgyi9.cn/599775.htm
yctv.lmlgyi9.cn/575715.htm
yctv.lmlgyi9.cn/575575.htm
yctv.lmlgyi9.cn/139375.htm
yctv.lmlgyi9.cn/933115.htm
yctv.lmlgyi9.cn/959555.htm
ycn.lmlgyi9.cn/393155.htm
ycn.lmlgyi9.cn/733135.htm
ycn.lmlgyi9.cn/171935.htm
ycn.lmlgyi9.cn/151115.htm
ycn.lmlgyi9.cn/131975.htm
ycn.lmlgyi9.cn/931395.htm
ycn.lmlgyi9.cn/399775.htm
ycn.lmlgyi9.cn/777915.htm
ycn.lmlgyi9.cn/135795.htm
ycn.lmlgyi9.cn/339955.htm
ycb.lmlgyi9.cn/571375.htm
ycb.lmlgyi9.cn/331335.htm
ycb.lmlgyi9.cn/753955.htm
ycb.lmlgyi9.cn/577335.htm
ycb.lmlgyi9.cn/379335.htm
ycb.lmlgyi9.cn/179535.htm
ycb.lmlgyi9.cn/799535.htm
ycb.lmlgyi9.cn/777715.htm
ycb.lmlgyi9.cn/979315.htm
ycb.lmlgyi9.cn/333935.htm
ycv.lmlgyi9.cn/315135.htm
ycv.lmlgyi9.cn/779755.htm
ycv.lmlgyi9.cn/595955.htm
ycv.lmlgyi9.cn/191595.htm
ycv.lmlgyi9.cn/777735.htm
ycv.lmlgyi9.cn/177115.htm
ycv.lmlgyi9.cn/999175.htm
ycv.lmlgyi9.cn/975155.htm
ycv.lmlgyi9.cn/919975.htm
ycv.lmlgyi9.cn/317395.htm
ycc.lmlgyi9.cn/591135.htm
ycc.lmlgyi9.cn/375975.htm
ycc.lmlgyi9.cn/597535.htm
ycc.lmlgyi9.cn/573335.htm
ycc.lmlgyi9.cn/391955.htm
ycc.lmlgyi9.cn/911795.htm
ycc.lmlgyi9.cn/159795.htm
ycc.lmlgyi9.cn/353395.htm
ycc.lmlgyi9.cn/191775.htm
ycc.lmlgyi9.cn/773735.htm
ycx.lmlgyi9.cn/393715.htm
ycx.lmlgyi9.cn/333335.htm
ycx.lmlgyi9.cn/775315.htm
ycx.lmlgyi9.cn/573315.htm
ycx.lmlgyi9.cn/997115.htm
ycx.lmlgyi9.cn/755175.htm
ycx.lmlgyi9.cn/557795.htm
ycx.lmlgyi9.cn/371715.htm
ycx.lmlgyi9.cn/773955.htm
ycx.lmlgyi9.cn/731335.htm
ycz.lmlgyi9.cn/995115.htm
ycz.lmlgyi9.cn/595195.htm
ycz.lmlgyi9.cn/591935.htm
ycz.lmlgyi9.cn/595755.htm
ycz.lmlgyi9.cn/135155.htm
ycz.lmlgyi9.cn/177515.htm
ycz.lmlgyi9.cn/713175.htm
ycz.lmlgyi9.cn/935955.htm
ycz.lmlgyi9.cn/935195.htm
ycz.lmlgyi9.cn/959535.htm
ycl.lmlgyi9.cn/713175.htm
ycl.lmlgyi9.cn/591995.htm
ycl.lmlgyi9.cn/535155.htm
ycl.lmlgyi9.cn/539575.htm
ycl.lmlgyi9.cn/337735.htm
ycl.lmlgyi9.cn/113535.htm
ycl.lmlgyi9.cn/579775.htm
ycl.lmlgyi9.cn/157315.htm
ycl.lmlgyi9.cn/357535.htm
ycl.lmlgyi9.cn/315355.htm
yck.lmlgyi9.cn/935155.htm
yck.lmlgyi9.cn/359135.htm
yck.lmlgyi9.cn/151735.htm
yck.lmlgyi9.cn/315555.htm
yck.lmlgyi9.cn/533715.htm
yck.lmlgyi9.cn/953515.htm
yck.lmlgyi9.cn/731155.htm
yck.lmlgyi9.cn/159755.htm
yck.lmlgyi9.cn/917315.htm
yck.lmlgyi9.cn/517795.htm
ycj.lmlgyi9.cn/777735.htm
ycj.lmlgyi9.cn/111975.htm
ycj.lmlgyi9.cn/915995.htm
ycj.lmlgyi9.cn/711395.htm
ycj.lmlgyi9.cn/591775.htm
ycj.lmlgyi9.cn/719335.htm
ycj.lmlgyi9.cn/951335.htm
ycj.lmlgyi9.cn/311155.htm
ycj.lmlgyi9.cn/591355.htm
ycj.lmlgyi9.cn/759535.htm
ych.lmlgyi9.cn/997775.htm
ych.lmlgyi9.cn/935175.htm
ych.lmlgyi9.cn/999775.htm
ych.lmlgyi9.cn/355335.htm
ych.lmlgyi9.cn/957955.htm
ych.lmlgyi9.cn/937995.htm
ych.lmlgyi9.cn/331935.htm
ych.lmlgyi9.cn/177935.htm
ych.lmlgyi9.cn/777955.htm
ych.lmlgyi9.cn/391115.htm
ycg.lmlgyi9.cn/979355.htm
ycg.lmlgyi9.cn/331155.htm
ycg.lmlgyi9.cn/737535.htm
ycg.lmlgyi9.cn/713555.htm
ycg.lmlgyi9.cn/939955.htm
ycg.lmlgyi9.cn/957935.htm
ycg.lmlgyi9.cn/591175.htm
ycg.lmlgyi9.cn/597195.htm
ycg.lmlgyi9.cn/599955.htm
ycg.lmlgyi9.cn/111775.htm
ycf.lmlgyi9.cn/539335.htm
ycf.lmlgyi9.cn/759535.htm
ycf.lmlgyi9.cn/717555.htm
ycf.lmlgyi9.cn/177195.htm
ycf.lmlgyi9.cn/519375.htm
ycf.lmlgyi9.cn/953355.htm
ycf.lmlgyi9.cn/753355.htm
ycf.lmlgyi9.cn/371575.htm
ycf.lmlgyi9.cn/799155.htm
ycf.lmlgyi9.cn/911715.htm
ycd.lmlgyi9.cn/311775.htm
ycd.lmlgyi9.cn/793535.htm
ycd.lmlgyi9.cn/951115.htm
ycd.lmlgyi9.cn/795195.htm
ycd.lmlgyi9.cn/739975.htm
ycd.lmlgyi9.cn/973175.htm
ycd.lmlgyi9.cn/957135.htm
ycd.lmlgyi9.cn/131515.htm
ycd.lmlgyi9.cn/519175.htm
ycd.lmlgyi9.cn/795735.htm
ycs.lmlgyi9.cn/535175.htm
ycs.lmlgyi9.cn/595375.htm
ycs.lmlgyi9.cn/117535.htm
ycs.lmlgyi9.cn/111575.htm
ycs.lmlgyi9.cn/391795.htm
ycs.lmlgyi9.cn/159735.htm
ycs.lmlgyi9.cn/371195.htm
ycs.lmlgyi9.cn/313355.htm
ycs.lmlgyi9.cn/533735.htm
ycs.lmlgyi9.cn/757515.htm
yca.lmlgyi9.cn/131595.htm
yca.lmlgyi9.cn/135535.htm
yca.lmlgyi9.cn/953175.htm
yca.lmlgyi9.cn/375155.htm
yca.lmlgyi9.cn/591195.htm
yca.lmlgyi9.cn/333755.htm
yca.lmlgyi9.cn/571335.htm
yca.lmlgyi9.cn/995555.htm
yca.lmlgyi9.cn/373355.htm
yca.lmlgyi9.cn/171955.htm
ycp.lmlgyi9.cn/339155.htm
ycp.lmlgyi9.cn/771715.htm
ycp.lmlgyi9.cn/391955.htm
ycp.lmlgyi9.cn/533935.htm
ycp.lmlgyi9.cn/591375.htm
ycp.lmlgyi9.cn/911535.htm
ycp.lmlgyi9.cn/573135.htm
ycp.lmlgyi9.cn/519175.htm
ycp.lmlgyi9.cn/335375.htm
ycp.lmlgyi9.cn/799115.htm
yco.lmlgyi9.cn/375735.htm
yco.lmlgyi9.cn/791575.htm
yco.lmlgyi9.cn/397115.htm
yco.lmlgyi9.cn/157395.htm
yco.lmlgyi9.cn/997775.htm
yco.lmlgyi9.cn/315355.htm
yco.lmlgyi9.cn/995355.htm
yco.lmlgyi9.cn/337795.htm
yco.lmlgyi9.cn/995595.htm
yco.lmlgyi9.cn/799115.htm
yci.lmlgyi9.cn/951755.htm
yci.lmlgyi9.cn/957335.htm
yci.lmlgyi9.cn/315555.htm
yci.lmlgyi9.cn/571155.htm
yci.lmlgyi9.cn/799755.htm
yci.lmlgyi9.cn/917115.htm
yci.lmlgyi9.cn/377735.htm
yci.lmlgyi9.cn/713755.htm
yci.lmlgyi9.cn/937335.htm
yci.lmlgyi9.cn/537335.htm
ycu.lmlgyi9.cn/591555.htm
ycu.lmlgyi9.cn/797375.htm
ycu.lmlgyi9.cn/119135.htm
ycu.lmlgyi9.cn/333315.htm
ycu.lmlgyi9.cn/933775.htm
ycu.lmlgyi9.cn/133955.htm
ycu.lmlgyi9.cn/973155.htm
ycu.lmlgyi9.cn/591375.htm
ycu.lmlgyi9.cn/313735.htm
ycu.lmlgyi9.cn/119735.htm
ycy.lmlgyi9.cn/575535.htm
ycy.lmlgyi9.cn/737575.htm
ycy.lmlgyi9.cn/395135.htm
ycy.lmlgyi9.cn/791775.htm
ycy.lmlgyi9.cn/579315.htm
ycy.lmlgyi9.cn/591955.htm
ycy.lmlgyi9.cn/939915.htm
ycy.lmlgyi9.cn/591175.htm
ycy.lmlgyi9.cn/979915.htm
ycy.lmlgyi9.cn/533315.htm
yct.lmlgyi9.cn/757395.htm
yct.lmlgyi9.cn/575975.htm
yct.lmlgyi9.cn/404245.htm
yct.lmlgyi9.cn/519315.htm
yct.lmlgyi9.cn/577195.htm
yct.lmlgyi9.cn/268885.htm
yct.lmlgyi9.cn/173775.htm
yct.lmlgyi9.cn/353775.htm
yct.lmlgyi9.cn/555775.htm
yct.lmlgyi9.cn/193335.htm
ycr.lmlgyi9.cn/995995.htm
ycr.lmlgyi9.cn/393555.htm
ycr.lmlgyi9.cn/268645.htm
ycr.lmlgyi9.cn/608645.htm
ycr.lmlgyi9.cn/286025.htm
ycr.lmlgyi9.cn/993335.htm
ycr.lmlgyi9.cn/777715.htm
ycr.lmlgyi9.cn/575915.htm
ycr.lmlgyi9.cn/755735.htm
ycr.lmlgyi9.cn/973375.htm
yce.lmlgyi9.cn/755115.htm
yce.lmlgyi9.cn/979375.htm
yce.lmlgyi9.cn/539735.htm
yce.lmlgyi9.cn/197595.htm
yce.lmlgyi9.cn/046205.htm
yce.lmlgyi9.cn/597335.htm
yce.lmlgyi9.cn/171535.htm
yce.lmlgyi9.cn/577995.htm
yce.lmlgyi9.cn/648025.htm
yce.lmlgyi9.cn/004065.htm
ycw.lmlgyi9.cn/468205.htm
ycw.lmlgyi9.cn/375535.htm
ycw.lmlgyi9.cn/339195.htm
ycw.lmlgyi9.cn/173735.htm
ycw.lmlgyi9.cn/053935.htm
ycw.lmlgyi9.cn/733115.htm
ycw.lmlgyi9.cn/266485.htm
ycw.lmlgyi9.cn/377795.htm
ycw.lmlgyi9.cn/084265.htm
ycw.lmlgyi9.cn/624285.htm
ycq.lmlgyi9.cn/115575.htm
ycq.lmlgyi9.cn/991715.htm
ycq.lmlgyi9.cn/199775.htm
ycq.lmlgyi9.cn/933915.htm
ycq.lmlgyi9.cn/119315.htm
ycq.lmlgyi9.cn/482005.htm
ycq.lmlgyi9.cn/593155.htm
ycq.lmlgyi9.cn/220465.htm
ycq.lmlgyi9.cn/028085.htm
ycq.lmlgyi9.cn/086865.htm
yxtv.lmlgyi9.cn/371795.htm
yxtv.lmlgyi9.cn/595395.htm
yxtv.lmlgyi9.cn/682485.htm
yxtv.lmlgyi9.cn/395575.htm
yxtv.lmlgyi9.cn/828825.htm
yxtv.lmlgyi9.cn/113555.htm
yxtv.lmlgyi9.cn/642625.htm
yxtv.lmlgyi9.cn/519195.htm
yxtv.lmlgyi9.cn/151135.htm
yxtv.lmlgyi9.cn/866865.htm
yxn.lmlgyi9.cn/979595.htm
yxn.lmlgyi9.cn/955715.htm
yxn.lmlgyi9.cn/395915.htm
yxn.lmlgyi9.cn/462885.htm
yxn.lmlgyi9.cn/402425.htm
yxn.lmlgyi9.cn/264665.htm
yxn.lmlgyi9.cn/791355.htm
yxn.lmlgyi9.cn/957955.htm
yxn.lmlgyi9.cn/604065.htm
yxn.lmlgyi9.cn/228605.htm
yxb.lmlgyi9.cn/642265.htm
yxb.lmlgyi9.cn/175115.htm
yxb.lmlgyi9.cn/606865.htm
yxb.lmlgyi9.cn/624665.htm
yxb.lmlgyi9.cn/486425.htm
yxb.lmlgyi9.cn/480085.htm
yxb.lmlgyi9.cn/240665.htm
yxb.lmlgyi9.cn/268265.htm
yxb.lmlgyi9.cn/266885.htm
yxb.lmlgyi9.cn/668845.htm
yxv.lmlgyi9.cn/591395.htm
yxv.lmlgyi9.cn/262805.htm
yxv.lmlgyi9.cn/408885.htm
yxv.lmlgyi9.cn/517155.htm
yxv.lmlgyi9.cn/680825.htm
yxv.lmlgyi9.cn/468445.htm
yxv.lmlgyi9.cn/082445.htm
yxv.lmlgyi9.cn/868425.htm
yxv.lmlgyi9.cn/242285.htm
yxv.lmlgyi9.cn/626065.htm
yxc.lmlgyi9.cn/260645.htm
yxc.lmlgyi9.cn/131715.htm
yxc.lmlgyi9.cn/337955.htm
yxc.lmlgyi9.cn/400865.htm
yxc.lmlgyi9.cn/064225.htm
yxc.lmlgyi9.cn/955355.htm
yxc.lmlgyi9.cn/511935.htm
yxc.lmlgyi9.cn/666605.htm
yxc.lmlgyi9.cn/317315.htm
yxc.lmlgyi9.cn/193375.htm
yxx.lmlgyi9.cn/511355.htm
yxx.lmlgyi9.cn/955915.htm
yxx.lmlgyi9.cn/179375.htm
yxx.lmlgyi9.cn/664605.htm
yxx.lmlgyi9.cn/686245.htm
yxx.lmlgyi9.cn/357955.htm
yxx.lmlgyi9.cn/628845.htm
yxx.lmlgyi9.cn/371975.htm
yxx.lmlgyi9.cn/599915.htm
yxx.lmlgyi9.cn/957755.htm
yxz.lmlgyi9.cn/919995.htm
yxz.lmlgyi9.cn/779355.htm
yxz.lmlgyi9.cn/779375.htm
yxz.lmlgyi9.cn/917975.htm
yxz.lmlgyi9.cn/220645.htm
yxz.lmlgyi9.cn/775775.htm
yxz.lmlgyi9.cn/024825.htm
yxz.lmlgyi9.cn/208865.htm
yxz.lmlgyi9.cn/620025.htm
yxz.lmlgyi9.cn/602605.htm
yxl.lmlgyi9.cn/793515.htm
yxl.lmlgyi9.cn/684065.htm
yxl.lmlgyi9.cn/597595.htm
yxl.lmlgyi9.cn/680045.htm
yxl.lmlgyi9.cn/795155.htm
yxl.lmlgyi9.cn/600085.htm
yxl.lmlgyi9.cn/622005.htm
yxl.lmlgyi9.cn/886425.htm
yxl.lmlgyi9.cn/915395.htm
yxl.lmlgyi9.cn/224645.htm
yxk.lmlgyi9.cn/151395.htm
yxk.lmlgyi9.cn/822465.htm
yxk.lmlgyi9.cn/999355.htm
yxk.lmlgyi9.cn/222845.htm
yxk.lmlgyi9.cn/486825.htm
yxk.lmlgyi9.cn/804445.htm
yxk.lmlgyi9.cn/597935.htm
yxk.lmlgyi9.cn/682085.htm
yxk.lmlgyi9.cn/979715.htm
yxk.lmlgyi9.cn/004265.htm
yxj.lmlgyi9.cn/660645.htm
yxj.lmlgyi9.cn/468805.htm
yxj.lmlgyi9.cn/248405.htm
yxj.lmlgyi9.cn/682205.htm
yxj.lmlgyi9.cn/626465.htm
yxj.lmlgyi9.cn/880625.htm
yxj.lmlgyi9.cn/282025.htm
yxj.lmlgyi9.cn/953175.htm
yxj.lmlgyi9.cn/557775.htm
yxj.lmlgyi9.cn/939735.htm
yxh.lmlgyi9.cn/593135.htm
yxh.lmlgyi9.cn/953115.htm
yxh.lmlgyi9.cn/377735.htm
yxh.lmlgyi9.cn/95.htm
yxh.lmlgyi9.cn/264225.htm
yxh.lmlgyi9.cn/842865.htm
yxh.lmlgyi9.cn/353975.htm
yxh.lmlgyi9.cn/993795.htm
yxh.lmlgyi9.cn/686005.htm
yxh.lmlgyi9.cn/333155.htm
yxg.lmlgyi9.cn/488625.htm
yxg.lmlgyi9.cn/262885.htm
