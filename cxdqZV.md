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

dzh.baoquan026.cn/64864.Doc
dzh.baoquan026.cn/24242.Doc
dzh.baoquan026.cn/88406.Doc
dzh.baoquan026.cn/26408.Doc
dzh.baoquan026.cn/62886.Doc
dzh.baoquan026.cn/84688.Doc
dzh.baoquan026.cn/28064.Doc
dzh.baoquan026.cn/64024.Doc
dzh.baoquan026.cn/08248.Doc
dzh.baoquan026.cn/02608.Doc
dzg.baoquan026.cn/22466.Doc
dzg.baoquan026.cn/02264.Doc
dzg.baoquan026.cn/08682.Doc
dzg.baoquan026.cn/40246.Doc
dzg.baoquan026.cn/20006.Doc
dzg.baoquan026.cn/84660.Doc
dzg.baoquan026.cn/00402.Doc
dzg.baoquan026.cn/04228.Doc
dzg.baoquan026.cn/84006.Doc
dzg.baoquan026.cn/40062.Doc
dzf.baoquan026.cn/46886.Doc
dzf.baoquan026.cn/46686.Doc
dzf.baoquan026.cn/46804.Doc
dzf.baoquan026.cn/46864.Doc
dzf.baoquan026.cn/20620.Doc
dzf.baoquan026.cn/20442.Doc
dzf.baoquan026.cn/06206.Doc
dzf.baoquan026.cn/20480.Doc
dzf.baoquan026.cn/08488.Doc
dzf.baoquan026.cn/62820.Doc
dzd.baoquan026.cn/40440.Doc
dzd.baoquan026.cn/00044.Doc
dzd.baoquan026.cn/86626.Doc
dzd.baoquan026.cn/44002.Doc
dzd.baoquan026.cn/86882.Doc
dzd.baoquan026.cn/62600.Doc
dzd.baoquan026.cn/60206.Doc
dzd.baoquan026.cn/02204.Doc
dzd.baoquan026.cn/20662.Doc
dzd.baoquan026.cn/88208.Doc
dzs.baoquan026.cn/66808.Doc
dzs.baoquan026.cn/66626.Doc
dzs.baoquan026.cn/66242.Doc
dzs.baoquan026.cn/06648.Doc
dzs.baoquan026.cn/24860.Doc
dzs.baoquan026.cn/60480.Doc
dzs.baoquan026.cn/04266.Doc
dzs.baoquan026.cn/40406.Doc
dzs.baoquan026.cn/28606.Doc
dzs.baoquan026.cn/60002.Doc
dza.baoquan026.cn/04220.Doc
dza.baoquan026.cn/02606.Doc
dza.baoquan026.cn/80844.Doc
dza.baoquan026.cn/84800.Doc
dza.baoquan026.cn/48468.Doc
dza.baoquan026.cn/64022.Doc
dza.baoquan026.cn/44008.Doc
dza.baoquan026.cn/42622.Doc
dza.baoquan026.cn/66002.Doc
dza.baoquan026.cn/04802.Doc
dzp.baoquan026.cn/46806.Doc
dzp.baoquan026.cn/46428.Doc
dzp.baoquan026.cn/42484.Doc
dzp.baoquan026.cn/66480.Doc
dzp.baoquan026.cn/24082.Doc
dzp.baoquan026.cn/60680.Doc
dzp.baoquan026.cn/60008.Doc
dzp.baoquan026.cn/44246.Doc
dzp.baoquan026.cn/28040.Doc
dzp.baoquan026.cn/82624.Doc
dzo.baoquan026.cn/82624.Doc
dzo.baoquan026.cn/66484.Doc
dzo.baoquan026.cn/84286.Doc
dzo.baoquan026.cn/88024.Doc
dzo.baoquan026.cn/66426.Doc
dzo.baoquan026.cn/60844.Doc
dzo.baoquan026.cn/04808.Doc
dzo.baoquan026.cn/60066.Doc
dzo.baoquan026.cn/20282.Doc
dzo.baoquan026.cn/48444.Doc
dzi.baoquan026.cn/84008.Doc
dzi.baoquan026.cn/68606.Doc
dzi.baoquan026.cn/24486.Doc
dzi.baoquan026.cn/08684.Doc
dzi.baoquan026.cn/86284.Doc
dzi.baoquan026.cn/86840.Doc
dzi.baoquan026.cn/08028.Doc
dzi.baoquan026.cn/80420.Doc
dzi.baoquan026.cn/60888.Doc
dzi.baoquan026.cn/68848.Doc
dzu.baoquan026.cn/08606.Doc
dzu.baoquan026.cn/08288.Doc
dzu.baoquan026.cn/08080.Doc
dzu.baoquan026.cn/00402.Doc
dzu.baoquan026.cn/40222.Doc
dzu.baoquan026.cn/82862.Doc
dzu.baoquan026.cn/62002.Doc
dzu.baoquan026.cn/42882.Doc
dzu.baoquan026.cn/82400.Doc
dzu.baoquan026.cn/44042.Doc
dzy.baoquan026.cn/24822.Doc
dzy.baoquan026.cn/80826.Doc
dzy.baoquan026.cn/02662.Doc
dzy.baoquan026.cn/82842.Doc
dzy.baoquan026.cn/24204.Doc
dzy.baoquan026.cn/40404.Doc
dzy.baoquan026.cn/02244.Doc
dzy.baoquan026.cn/24222.Doc
dzy.baoquan026.cn/22288.Doc
dzy.baoquan026.cn/66244.Doc
dzt.baoquan026.cn/86266.Doc
dzt.baoquan026.cn/48862.Doc
dzt.baoquan026.cn/22424.Doc
dzt.baoquan026.cn/64884.Doc
dzt.baoquan026.cn/44066.Doc
dzt.baoquan026.cn/88264.Doc
dzt.baoquan026.cn/88860.Doc
dzt.baoquan026.cn/48066.Doc
dzt.baoquan026.cn/48422.Doc
dzt.baoquan026.cn/06282.Doc
dzr.baoquan026.cn/04240.Doc
dzr.baoquan026.cn/62842.Doc
dzr.baoquan026.cn/62620.Doc
dzr.baoquan026.cn/42060.Doc
dzr.baoquan026.cn/66002.Doc
dzr.baoquan026.cn/66286.Doc
dzr.baoquan026.cn/62602.Doc
dzr.baoquan026.cn/44486.Doc
dzr.baoquan026.cn/86060.Doc
dzr.baoquan026.cn/00842.Doc
dze.baoquan026.cn/82422.Doc
dze.baoquan026.cn/02086.Doc
dze.baoquan026.cn/20846.Doc
dze.baoquan026.cn/24202.Doc
dze.baoquan026.cn/64020.Doc
dze.baoquan026.cn/64680.Doc
dze.baoquan026.cn/68008.Doc
dze.baoquan026.cn/82228.Doc
dze.baoquan026.cn/26460.Doc
dze.baoquan026.cn/68882.Doc
dzw.baoquan026.cn/88046.Doc
dzw.baoquan026.cn/86204.Doc
dzw.baoquan026.cn/40068.Doc
dzw.baoquan026.cn/64848.Doc
dzw.baoquan026.cn/68802.Doc
dzw.baoquan026.cn/42688.Doc
dzw.baoquan026.cn/04880.Doc
dzw.baoquan026.cn/80284.Doc
dzw.baoquan026.cn/06084.Doc
dzw.baoquan026.cn/26666.Doc
dzq.baoquan026.cn/26080.Doc
dzq.baoquan026.cn/68420.Doc
dzq.baoquan026.cn/46060.Doc
dzq.baoquan026.cn/20666.Doc
dzq.baoquan026.cn/62022.Doc
dzq.baoquan026.cn/42020.Doc
dzq.baoquan026.cn/82086.Doc
dzq.baoquan026.cn/48444.Doc
dzq.baoquan026.cn/68444.Doc
dzq.baoquan026.cn/86662.Doc
dlm.baoquan026.cn/60448.Doc
dlm.baoquan026.cn/82662.Doc
dlm.baoquan026.cn/40644.Doc
dlm.baoquan026.cn/84200.Doc
dlm.baoquan026.cn/60080.Doc
dlm.baoquan026.cn/66486.Doc
dlm.baoquan026.cn/80688.Doc
dlm.baoquan026.cn/86640.Doc
dlm.baoquan026.cn/66806.Doc
dlm.baoquan026.cn/80282.Doc
dln.baoquan026.cn/04026.Doc
dln.baoquan026.cn/20426.Doc
dln.baoquan026.cn/48404.Doc
dln.baoquan026.cn/06266.Doc
dln.baoquan026.cn/28486.Doc
dln.baoquan026.cn/66020.Doc
dln.baoquan026.cn/26208.Doc
dln.baoquan026.cn/00260.Doc
dln.baoquan026.cn/02262.Doc
dln.baoquan026.cn/24868.Doc
dlb.baoquan026.cn/62624.Doc
dlb.baoquan026.cn/64464.Doc
dlb.baoquan026.cn/68040.Doc
dlb.baoquan026.cn/60026.Doc
dlb.baoquan026.cn/04008.Doc
dlb.baoquan026.cn/40064.Doc
dlb.baoquan026.cn/84048.Doc
dlb.baoquan026.cn/28660.Doc
dlb.baoquan026.cn/60606.Doc
dlb.baoquan026.cn/80668.Doc
dlv.baoquan026.cn/73133.Doc
dlv.baoquan026.cn/66060.Doc
dlv.baoquan026.cn/84626.Doc
dlv.baoquan026.cn/04080.Doc
dlv.baoquan026.cn/26048.Doc
dlv.baoquan026.cn/04604.Doc
dlv.baoquan026.cn/57759.Doc
dlv.baoquan026.cn/62028.Doc
dlv.baoquan026.cn/48806.Doc
dlv.baoquan026.cn/00228.Doc
dlc.baoquan026.cn/42486.Doc
dlc.baoquan026.cn/75757.Doc
dlc.baoquan026.cn/48642.Doc
dlc.baoquan026.cn/84400.Doc
dlc.baoquan026.cn/80682.Doc
dlc.baoquan026.cn/20886.Doc
dlc.baoquan026.cn/04202.Doc
dlc.baoquan026.cn/66628.Doc
dlc.baoquan026.cn/08646.Doc
dlc.baoquan026.cn/42284.Doc
dlx.baoquan026.cn/86464.Doc
dlx.baoquan026.cn/95991.Doc
dlx.baoquan026.cn/40462.Doc
dlx.baoquan026.cn/26200.Doc
dlx.baoquan026.cn/08000.Doc
dlx.baoquan026.cn/46044.Doc
dlx.baoquan026.cn/60624.Doc
dlx.baoquan026.cn/28288.Doc
dlx.baoquan026.cn/06222.Doc
dlx.baoquan026.cn/02060.Doc
dlz.baoquan026.cn/02448.Doc
dlz.baoquan026.cn/64048.Doc
dlz.baoquan026.cn/00648.Doc
dlz.baoquan026.cn/71371.Doc
dlz.baoquan026.cn/60826.Doc
dlz.baoquan026.cn/26268.Doc
dlz.baoquan026.cn/64800.Doc
dlz.baoquan026.cn/62604.Doc
dlz.baoquan026.cn/22204.Doc
dlz.baoquan026.cn/64040.Doc
dll.baoquan026.cn/60646.Doc
dll.baoquan026.cn/68444.Doc
dll.baoquan026.cn/66448.Doc
dll.baoquan026.cn/31917.Doc
dll.baoquan026.cn/48486.Doc
dll.baoquan026.cn/44402.Doc
dll.baoquan026.cn/40424.Doc
dll.baoquan026.cn/06204.Doc
dll.baoquan026.cn/06862.Doc
dll.baoquan026.cn/62020.Doc
dlk.baoquan026.cn/68682.Doc
dlk.baoquan026.cn/22648.Doc
dlk.baoquan026.cn/08286.Doc
dlk.baoquan026.cn/46620.Doc
dlk.baoquan026.cn/40220.Doc
dlk.baoquan026.cn/20882.Doc
dlk.baoquan026.cn/77959.Doc
dlk.baoquan026.cn/84868.Doc
dlk.baoquan026.cn/26622.Doc
dlk.baoquan026.cn/02206.Doc
dlj.baoquan026.cn/99977.Doc
dlj.baoquan026.cn/84680.Doc
dlj.baoquan026.cn/20862.Doc
dlj.baoquan026.cn/66088.Doc
dlj.baoquan026.cn/60080.Doc
dlj.baoquan026.cn/80682.Doc
dlj.baoquan026.cn/44682.Doc
dlj.baoquan026.cn/24066.Doc
dlj.baoquan026.cn/46024.Doc
dlj.baoquan026.cn/24684.Doc
dlh.baoquan026.cn/06884.Doc
dlh.baoquan026.cn/28686.Doc
dlh.baoquan026.cn/11311.Doc
dlh.baoquan026.cn/02226.Doc
dlh.baoquan026.cn/42046.Doc
dlh.baoquan026.cn/42044.Doc
dlh.baoquan026.cn/48228.Doc
dlh.baoquan026.cn/22480.Doc
dlh.baoquan026.cn/48480.Doc
dlh.baoquan026.cn/44224.Doc
dlg.baoquan026.cn/00482.Doc
dlg.baoquan026.cn/22664.Doc
dlg.baoquan026.cn/66460.Doc
dlg.baoquan026.cn/22842.Doc
dlg.baoquan026.cn/68886.Doc
dlg.baoquan026.cn/42028.Doc
dlg.baoquan026.cn/08686.Doc
dlg.baoquan026.cn/53133.Doc
dlg.baoquan026.cn/86002.Doc
dlg.baoquan026.cn/68606.Doc
dlf.baoquan026.cn/68088.Doc
dlf.baoquan026.cn/99359.Doc
dlf.baoquan026.cn/06862.Doc
dlf.baoquan026.cn/22402.Doc
dlf.baoquan026.cn/68642.Doc
dlf.baoquan026.cn/04042.Doc
dlf.baoquan026.cn/82688.Doc
dlf.baoquan026.cn/04460.Doc
dlf.baoquan026.cn/28262.Doc
dlf.baoquan026.cn/08062.Doc
dld.baoquan026.cn/37975.Doc
dld.baoquan026.cn/46268.Doc
dld.baoquan026.cn/48622.Doc
dld.baoquan026.cn/46004.Doc
dld.baoquan026.cn/62626.Doc
dld.baoquan026.cn/39537.Doc
dld.baoquan026.cn/42802.Doc
dld.baoquan026.cn/44062.Doc
dld.baoquan026.cn/24002.Doc
dld.baoquan026.cn/40888.Doc
dls.baoquan026.cn/20620.Doc
dls.baoquan026.cn/11319.Doc
dls.baoquan026.cn/20402.Doc
dls.baoquan026.cn/08468.Doc
dls.baoquan026.cn/02682.Doc
dls.baoquan026.cn/62440.Doc
dls.baoquan026.cn/42866.Doc
dls.baoquan026.cn/40620.Doc
dls.baoquan026.cn/42826.Doc
dls.baoquan026.cn/00466.Doc
dla.baoquan026.cn/06488.Doc
dla.baoquan026.cn/80648.Doc
dla.baoquan026.cn/26404.Doc
dla.baoquan026.cn/08002.Doc
dla.baoquan026.cn/62008.Doc
dla.baoquan026.cn/80886.Doc
dla.baoquan026.cn/66804.Doc
dla.baoquan026.cn/64648.Doc
dla.baoquan026.cn/22628.Doc
dla.baoquan026.cn/64448.Doc
dlp.baoquan026.cn/42086.Doc
dlp.baoquan026.cn/31377.Doc
dlp.baoquan026.cn/19933.Doc
dlp.baoquan026.cn/20648.Doc
dlp.baoquan026.cn/20204.Doc
dlp.baoquan026.cn/04426.Doc
dlp.baoquan026.cn/84244.Doc
dlp.baoquan026.cn/44680.Doc
dlp.baoquan026.cn/22048.Doc
dlp.baoquan026.cn/40466.Doc
dlo.baoquan026.cn/06668.Doc
dlo.baoquan026.cn/66800.Doc
dlo.baoquan026.cn/00044.Doc
dlo.baoquan026.cn/08064.Doc
dlo.baoquan026.cn/82644.Doc
dlo.baoquan026.cn/46400.Doc
dlo.baoquan026.cn/40626.Doc
dlo.baoquan026.cn/48660.Doc
dlo.baoquan026.cn/62202.Doc
dlo.baoquan026.cn/46840.Doc
dli.baoquan026.cn/35755.Doc
dli.baoquan026.cn/84066.Doc
dli.baoquan026.cn/44206.Doc
dli.baoquan026.cn/08680.Doc
dli.baoquan026.cn/77137.Doc
dli.baoquan026.cn/04666.Doc
dli.baoquan026.cn/26884.Doc
dli.baoquan026.cn/26286.Doc
dli.baoquan026.cn/62482.Doc
dli.baoquan026.cn/88000.Doc
dlu.baoquan026.cn/64608.Doc
dlu.baoquan026.cn/44426.Doc
dlu.baoquan026.cn/22046.Doc
dlu.baoquan026.cn/59597.Doc
dlu.baoquan026.cn/86422.Doc
dlu.baoquan026.cn/44464.Doc
dlu.baoquan026.cn/24806.Doc
dlu.baoquan026.cn/02846.Doc
dlu.baoquan026.cn/24002.Doc
dlu.baoquan026.cn/28860.Doc
dly.baoquan026.cn/04244.Doc
dly.baoquan026.cn/46444.Doc
dly.baoquan026.cn/59373.Doc
dly.baoquan026.cn/77335.Doc
dly.baoquan026.cn/02626.Doc
dly.baoquan026.cn/91995.Doc
dly.baoquan026.cn/55511.Doc
dly.baoquan026.cn/02480.Doc
dly.baoquan026.cn/20866.Doc
dly.baoquan026.cn/48022.Doc
dlt.baoquan026.cn/24206.Doc
dlt.baoquan026.cn/02088.Doc
dlt.baoquan026.cn/04006.Doc
dlt.baoquan026.cn/64088.Doc
dlt.baoquan026.cn/40442.Doc
dlt.baoquan026.cn/80648.Doc
dlt.baoquan026.cn/60840.Doc
dlt.baoquan026.cn/42006.Doc
dlt.baoquan026.cn/60460.Doc
dlt.baoquan026.cn/84468.Doc
dlr.baoquan026.cn/68002.Doc
dlr.baoquan026.cn/15951.Doc
dlr.baoquan026.cn/82824.Doc
dlr.baoquan026.cn/44846.Doc
dlr.baoquan026.cn/00288.Doc
dlr.baoquan026.cn/35511.Doc
dlr.baoquan026.cn/68420.Doc
dlr.baoquan026.cn/62408.Doc
dlr.baoquan026.cn/24684.Doc
dlr.baoquan026.cn/06284.Doc
dle.baoquan026.cn/26286.Doc
dle.baoquan026.cn/02080.Doc
dle.baoquan026.cn/00062.Doc
dle.baoquan026.cn/60266.Doc
dle.baoquan026.cn/66802.Doc
dle.baoquan026.cn/64804.Doc
dle.baoquan026.cn/48288.Doc
dle.baoquan026.cn/80044.Doc
dle.baoquan026.cn/08248.Doc
dle.baoquan026.cn/24406.Doc
dlw.baoquan026.cn/40802.Doc
dlw.baoquan026.cn/46840.Doc
dlw.baoquan026.cn/40244.Doc
dlw.baoquan026.cn/48822.Doc
dlw.baoquan026.cn/20882.Doc
dlw.baoquan026.cn/06004.Doc
dlw.baoquan026.cn/42220.Doc
dlw.baoquan026.cn/28262.Doc
dlw.baoquan026.cn/82668.Doc
dlw.baoquan026.cn/64486.Doc
dlq.baoquan026.cn/73351.Doc
dlq.baoquan026.cn/62004.Doc
dlq.baoquan026.cn/00208.Doc
dlq.baoquan026.cn/88400.Doc
dlq.baoquan026.cn/68682.Doc
dlq.baoquan026.cn/62888.Doc
dlq.baoquan026.cn/46280.Doc
dlq.baoquan026.cn/28202.Doc
dlq.baoquan026.cn/68628.Doc
dlq.baoquan026.cn/08622.Doc
dkm.baoquan026.cn/80882.Doc
dkm.baoquan026.cn/95777.Doc
dkm.baoquan026.cn/04866.Doc
dkm.baoquan026.cn/55357.Doc
dkm.baoquan026.cn/64824.Doc
dkm.baoquan026.cn/08220.Doc
dkm.baoquan026.cn/04004.Doc
dkm.baoquan026.cn/26868.Doc
dkm.baoquan026.cn/66608.Doc
dkm.baoquan026.cn/00420.Doc
dkn.baoquan026.cn/02600.Doc
dkn.baoquan026.cn/13315.Doc
dkn.baoquan026.cn/00264.Doc
dkn.baoquan026.cn/46606.Doc
dkn.baoquan026.cn/84082.Doc
dkn.baoquan026.cn/95111.Doc
dkn.baoquan026.cn/06686.Doc
dkn.baoquan026.cn/88000.Doc
dkn.baoquan026.cn/15977.Doc
dkn.baoquan026.cn/04602.Doc
dkb.baoquan026.cn/48440.Doc
dkb.baoquan026.cn/77117.Doc
dkb.baoquan026.cn/00480.Doc
dkb.baoquan026.cn/86066.Doc
dkb.baoquan026.cn/82648.Doc
dkb.baoquan026.cn/28200.Doc
dkb.baoquan026.cn/68848.Doc
dkb.baoquan026.cn/40824.Doc
dkb.baoquan026.cn/88422.Doc
dkb.baoquan026.cn/44620.Doc
dkv.baoquan026.cn/84660.Doc
dkv.baoquan026.cn/64028.Doc
dkv.baoquan026.cn/84800.Doc
dkv.baoquan026.cn/64000.Doc
dkv.baoquan026.cn/46686.Doc
dkv.baoquan026.cn/04866.Doc
dkv.baoquan026.cn/84006.Doc
dkv.baoquan026.cn/60660.Doc
dkv.baoquan026.cn/84282.Doc
dkv.baoquan026.cn/84288.Doc
dkc.baoquan026.cn/80800.Doc
dkc.baoquan026.cn/28040.Doc
dkc.baoquan026.cn/44482.Doc
dkc.baoquan026.cn/86428.Doc
dkc.baoquan026.cn/28086.Doc
dkc.baoquan026.cn/46406.Doc
dkc.baoquan026.cn/46042.Doc
dkc.baoquan026.cn/08282.Doc
dkc.baoquan026.cn/68222.Doc
dkc.baoquan026.cn/31335.Doc
dkx.baoquan026.cn/48428.Doc
dkx.baoquan026.cn/28246.Doc
dkx.baoquan026.cn/48468.Doc
dkx.baoquan026.cn/84428.Doc
dkx.baoquan026.cn/82866.Doc
dkx.baoquan026.cn/75595.Doc
dkx.baoquan026.cn/53959.Doc
dkx.baoquan026.cn/55137.Doc
dkx.baoquan026.cn/57937.Doc
dkx.baoquan026.cn/15597.Doc
dkz.baoquan026.cn/80240.Doc
dkz.baoquan026.cn/37999.Doc
dkz.baoquan026.cn/00808.Doc
dkz.baoquan026.cn/79995.Doc
dkz.baoquan026.cn/79355.Doc
dkz.baoquan026.cn/59777.Doc
dkz.baoquan026.cn/53313.Doc
dkz.baoquan026.cn/91917.Doc
dkz.baoquan026.cn/71559.Doc
dkz.baoquan026.cn/59773.Doc
