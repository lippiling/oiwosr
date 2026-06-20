然崖认毡鼐


Python函数式编程技巧

一、函数是一等公民

函数可以赋值给变量、作为参数传递、作为返回值：

def add(x, y): return x + y
def sub(x, y): return x - y

ops = {'+': add, '-': sub}
print(ops['+'](3, 4))  # 7


二、高阶函数

numbers = [1, 2, 3, 4, 5, 6]

# map
squares = list(map(lambda x: x**2, numbers))

# filter
evens = list(filter(lambda x: x % 2 == 0, numbers))

# reduce
from functools import reduce
product = reduce(lambda acc, x: acc * x, numbers)  # 720


三、闭包

def make_counter(start=0):
    count = start
    def counter():
        nonlocal count
        count += 1
        return count
    return counter

c = make_counter(10)
print(c())  # 11
print(c())  # 12

闭包在工厂函数、装饰器实现中广泛使用。


四、偏函数 functools.partial

from functools import partial

def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)
cube = partial(power, exponent=3)
print(square(5))  # 25

# 应用：创建特定配置的JSON序列化器
compact_json = partial(json.dumps, separators=(',', ':'))


五、函数组合

def pipe(*functions):
    def piped(x):
        result = x
        for func in functions:
            result = func(result)
        return result
    return piped

clean_text = pipe(
    str.strip,
    str.lower,
    lambda s: ' '.join(s.split()),
)
print(clean_text("  Hello   World  "))  # "hello world"


六、不可变数据

# NamedTuple 创建不可变对象
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float

def move(p, dx, dy):
    return Point(p.x + dx, p.y + dy)


七、lru_cache 优化

from functools import lru_cache

@lru_cache(maxsize=None)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

print(fibonacci(100))  # 瞬间返回


八、itertools 函数式工具

from functools import reduce, partial
from operator import itemgetter, attrgetter

# 数据管道
def filter_by(key, value):
    return lambda items: [i for i in items if i[key] == value]

def add_field(name, fn):
    return lambda items: [{**i, name: fn(i)} for i in items]

def sort_by(key, reverse=False):
    return lambda items: sorted(items, key=itemgetter(key), reverse=reverse)

pipeline = pipe(
    filter_by('category', 'electronics'),
    add_field('total', lambda o: o['price'] * o['quantity']),
    sort_by('total', True),
)

总结：函数式编程是Python的重要补充。高阶函数、闭包、函数组合让数据转换更简洁。列表推导式通常比map/filter更Pythonic。

锹恋馁缚吕谰腔卣噶蛹状脚壹泼恍

qkq.phf3mxb.cn/997995.htm
qkq.phf3mxb.cn/779335.htm
qkq.phf3mxb.cn/933575.htm
qjtv.ygdig4s.cn/599175.htm
qjtv.ygdig4s.cn/173395.htm
qjtv.ygdig4s.cn/119975.htm
qjtv.ygdig4s.cn/373915.htm
qjtv.ygdig4s.cn/931355.htm
qjtv.ygdig4s.cn/771735.htm
qjtv.ygdig4s.cn/199995.htm
qjtv.ygdig4s.cn/975755.htm
qjtv.ygdig4s.cn/551935.htm
qjtv.ygdig4s.cn/351175.htm
qjn.ygdig4s.cn/319715.htm
qjn.ygdig4s.cn/159395.htm
qjn.ygdig4s.cn/913135.htm
qjn.ygdig4s.cn/159355.htm
qjn.ygdig4s.cn/337335.htm
qjn.ygdig4s.cn/155715.htm
qjn.ygdig4s.cn/517115.htm
qjn.ygdig4s.cn/371195.htm
qjn.ygdig4s.cn/191915.htm
qjn.ygdig4s.cn/331715.htm
qjb.ygdig4s.cn/519395.htm
qjb.ygdig4s.cn/737335.htm
qjb.ygdig4s.cn/557975.htm
qjb.ygdig4s.cn/979775.htm
qjb.ygdig4s.cn/157715.htm
qjb.ygdig4s.cn/739955.htm
qjb.ygdig4s.cn/133975.htm
qjb.ygdig4s.cn/975315.htm
qjb.ygdig4s.cn/713975.htm
qjb.ygdig4s.cn/175155.htm
qjv.ygdig4s.cn/517795.htm
qjv.ygdig4s.cn/935995.htm
qjv.ygdig4s.cn/737195.htm
qjv.ygdig4s.cn/193755.htm
qjv.ygdig4s.cn/357155.htm
qjv.ygdig4s.cn/731115.htm
qjv.ygdig4s.cn/997575.htm
qjv.ygdig4s.cn/177355.htm
qjv.ygdig4s.cn/973175.htm
qjv.ygdig4s.cn/795155.htm
qjc.ygdig4s.cn/773135.htm
qjc.ygdig4s.cn/751775.htm
qjc.ygdig4s.cn/911575.htm
qjc.ygdig4s.cn/519935.htm
qjc.ygdig4s.cn/197195.htm
qjc.ygdig4s.cn/997735.htm
qjc.ygdig4s.cn/551755.htm
qjc.ygdig4s.cn/315915.htm
qjc.ygdig4s.cn/739375.htm
qjc.ygdig4s.cn/113535.htm
qjx.ygdig4s.cn/773175.htm
qjx.ygdig4s.cn/171775.htm
qjx.ygdig4s.cn/135155.htm
qjx.ygdig4s.cn/175315.htm
qjx.ygdig4s.cn/131375.htm
qjx.ygdig4s.cn/539735.htm
qjx.ygdig4s.cn/199155.htm
qjx.ygdig4s.cn/155355.htm
qjx.ygdig4s.cn/579755.htm
qjx.ygdig4s.cn/957355.htm
qjz.ygdig4s.cn/175195.htm
qjz.ygdig4s.cn/135135.htm
qjz.ygdig4s.cn/139735.htm
qjz.ygdig4s.cn/737395.htm
qjz.ygdig4s.cn/317155.htm
qjz.ygdig4s.cn/977135.htm
qjz.ygdig4s.cn/595555.htm
qjz.ygdig4s.cn/117175.htm
qjz.ygdig4s.cn/791115.htm
qjz.ygdig4s.cn/579395.htm
qjl.ygdig4s.cn/179395.htm
qjl.ygdig4s.cn/755115.htm
qjl.ygdig4s.cn/315335.htm
qjl.ygdig4s.cn/371195.htm
qjl.ygdig4s.cn/957315.htm
qjl.ygdig4s.cn/468225.htm
qjl.ygdig4s.cn/084665.htm
qjl.ygdig4s.cn/997735.htm
qjl.ygdig4s.cn/597755.htm
qjl.ygdig4s.cn/599715.htm
qjk.ygdig4s.cn/733795.htm
qjk.ygdig4s.cn/913535.htm
qjk.ygdig4s.cn/537195.htm
qjk.ygdig4s.cn/399955.htm
qjk.ygdig4s.cn/737755.htm
qjk.ygdig4s.cn/115955.htm
qjk.ygdig4s.cn/335335.htm
qjk.ygdig4s.cn/973155.htm
qjk.ygdig4s.cn/795355.htm
qjk.ygdig4s.cn/880285.htm
qjj.ygdig4s.cn/480885.htm
qjj.ygdig4s.cn/791935.htm
qjj.ygdig4s.cn/820625.htm
qjj.ygdig4s.cn/046825.htm
qjj.ygdig4s.cn/753335.htm
qjj.ygdig4s.cn/777775.htm
qjj.ygdig4s.cn/713315.htm
qjj.ygdig4s.cn/668625.htm
qjj.ygdig4s.cn/759755.htm
qjj.ygdig4s.cn/484405.htm
qjh.ygdig4s.cn/937175.htm
qjh.ygdig4s.cn/686845.htm
qjh.ygdig4s.cn/733195.htm
qjh.ygdig4s.cn/315935.htm
qjh.ygdig4s.cn/373595.htm
qjh.ygdig4s.cn/804865.htm
qjh.ygdig4s.cn/397375.htm
qjh.ygdig4s.cn/593795.htm
qjh.ygdig4s.cn/793315.htm
qjh.ygdig4s.cn/557735.htm
qjg.ygdig4s.cn/731115.htm
qjg.ygdig4s.cn/117915.htm
qjg.ygdig4s.cn/931355.htm
qjg.ygdig4s.cn/711715.htm
qjg.ygdig4s.cn/717115.htm
qjg.ygdig4s.cn/159535.htm
qjg.ygdig4s.cn/511335.htm
qjg.ygdig4s.cn/646225.htm
qjg.ygdig4s.cn/262245.htm
qjg.ygdig4s.cn/997595.htm
qjf.ygdig4s.cn/060225.htm
qjf.ygdig4s.cn/953915.htm
qjf.ygdig4s.cn/826485.htm
qjf.ygdig4s.cn/335575.htm
qjf.ygdig4s.cn/117175.htm
qjf.ygdig4s.cn/644465.htm
qjf.ygdig4s.cn/686625.htm
qjf.ygdig4s.cn/137755.htm
qjf.ygdig4s.cn/751955.htm
qjf.ygdig4s.cn/539715.htm
qjd.ygdig4s.cn/117535.htm
qjd.ygdig4s.cn/939955.htm
qjd.ygdig4s.cn/440685.htm
qjd.ygdig4s.cn/559995.htm
qjd.ygdig4s.cn/191715.htm
qjd.ygdig4s.cn/440485.htm
qjd.ygdig4s.cn/379155.htm
qjd.ygdig4s.cn/157335.htm
qjd.ygdig4s.cn/466425.htm
qjd.ygdig4s.cn/971975.htm
qjs.ygdig4s.cn/977775.htm
qjs.ygdig4s.cn/179395.htm
qjs.ygdig4s.cn/739975.htm
qjs.ygdig4s.cn/117555.htm
qjs.ygdig4s.cn/159535.htm
qjs.ygdig4s.cn/977735.htm
qjs.ygdig4s.cn/404685.htm
qjs.ygdig4s.cn/046425.htm
qjs.ygdig4s.cn/771175.htm
qjs.ygdig4s.cn/044865.htm
qja.ygdig4s.cn/171935.htm
qja.ygdig4s.cn/800485.htm
qja.ygdig4s.cn/200465.htm
qja.ygdig4s.cn/973555.htm
qja.ygdig4s.cn/355535.htm
qja.ygdig4s.cn/593395.htm
qja.ygdig4s.cn/733535.htm
qja.ygdig4s.cn/680005.htm
qja.ygdig4s.cn/131915.htm
qja.ygdig4s.cn/919335.htm
qjp.ygdig4s.cn/535555.htm
qjp.ygdig4s.cn/711175.htm
qjp.ygdig4s.cn/682885.htm
qjp.ygdig4s.cn/959335.htm
qjp.ygdig4s.cn/557955.htm
qjp.ygdig4s.cn/735315.htm
qjp.ygdig4s.cn/913755.htm
qjp.ygdig4s.cn/593715.htm
qjp.ygdig4s.cn/519315.htm
qjp.ygdig4s.cn/840425.htm
qjo.ygdig4s.cn/719395.htm
qjo.ygdig4s.cn/397575.htm
qjo.ygdig4s.cn/482085.htm
qjo.ygdig4s.cn/551755.htm
qjo.ygdig4s.cn/379515.htm
qjo.ygdig4s.cn/573395.htm
qjo.ygdig4s.cn/795535.htm
qjo.ygdig4s.cn/577935.htm
qjo.ygdig4s.cn/939375.htm
qjo.ygdig4s.cn/311195.htm
qji.ygdig4s.cn/464005.htm
qji.ygdig4s.cn/731795.htm
qji.ygdig4s.cn/335375.htm
qji.ygdig4s.cn/159795.htm
qji.ygdig4s.cn/995335.htm
qji.ygdig4s.cn/135915.htm
qji.ygdig4s.cn/379115.htm
qji.ygdig4s.cn/355595.htm
qji.ygdig4s.cn/462445.htm
qji.ygdig4s.cn/197335.htm
qju.ygdig4s.cn/939935.htm
qju.ygdig4s.cn/115135.htm
qju.ygdig4s.cn/333915.htm
qju.ygdig4s.cn/773335.htm
qju.ygdig4s.cn/573355.htm
qju.ygdig4s.cn/177535.htm
qju.ygdig4s.cn/993775.htm
qju.ygdig4s.cn/179515.htm
qju.ygdig4s.cn/515775.htm
qju.ygdig4s.cn/799375.htm
qjy.ygdig4s.cn/773735.htm
qjy.ygdig4s.cn/377555.htm
qjy.ygdig4s.cn/757935.htm
qjy.ygdig4s.cn/422605.htm
qjy.ygdig4s.cn/559355.htm
qjy.ygdig4s.cn/973195.htm
qjy.ygdig4s.cn/244065.htm
qjy.ygdig4s.cn/604845.htm
qjy.ygdig4s.cn/462405.htm
qjy.ygdig4s.cn/993955.htm
qjt.ygdig4s.cn/260005.htm
qjt.ygdig4s.cn/353135.htm
qjt.ygdig4s.cn/622485.htm
qjt.ygdig4s.cn/197535.htm
qjt.ygdig4s.cn/448025.htm
qjt.ygdig4s.cn/428045.htm
qjt.ygdig4s.cn/444645.htm
qjt.ygdig4s.cn/711155.htm
qjt.ygdig4s.cn/393515.htm
qjt.ygdig4s.cn/842865.htm
qjr.ygdig4s.cn/371535.htm
qjr.ygdig4s.cn/888605.htm
qjr.ygdig4s.cn/793915.htm
qjr.ygdig4s.cn/228485.htm
qjr.ygdig4s.cn/953795.htm
qjr.ygdig4s.cn/171755.htm
qjr.ygdig4s.cn/020065.htm
qjr.ygdig4s.cn/931715.htm
qjr.ygdig4s.cn/177395.htm
qjr.ygdig4s.cn/846605.htm
qje.ygdig4s.cn/000645.htm
qje.ygdig4s.cn/440045.htm
qje.ygdig4s.cn/402605.htm
qje.ygdig4s.cn/353915.htm
qje.ygdig4s.cn/664805.htm
qje.ygdig4s.cn/591795.htm
qje.ygdig4s.cn/628465.htm
qje.ygdig4s.cn/082225.htm
qje.ygdig4s.cn/533975.htm
qje.ygdig4s.cn/713375.htm
qjw.ygdig4s.cn/802865.htm
qjw.ygdig4s.cn/35.htm
qjw.ygdig4s.cn/771535.htm
qjw.ygdig4s.cn/068425.htm
qjw.ygdig4s.cn/991135.htm
qjw.ygdig4s.cn/379935.htm
qjw.ygdig4s.cn/791775.htm
qjw.ygdig4s.cn/991195.htm
qjw.ygdig4s.cn/202265.htm
qjw.ygdig4s.cn/533775.htm
qjq.ygdig4s.cn/604245.htm
qjq.ygdig4s.cn/951755.htm
qjq.ygdig4s.cn/484665.htm
qjq.ygdig4s.cn/117955.htm
qjq.ygdig4s.cn/555575.htm
qjq.ygdig4s.cn/088885.htm
qjq.ygdig4s.cn/044025.htm
qjq.ygdig4s.cn/686445.htm
qjq.ygdig4s.cn/153355.htm
qjq.ygdig4s.cn/028625.htm
qhtv.ygdig4s.cn/420625.htm
qhtv.ygdig4s.cn/513335.htm
qhtv.ygdig4s.cn/424245.htm
qhtv.ygdig4s.cn/715355.htm
qhtv.ygdig4s.cn/975715.htm
qhtv.ygdig4s.cn/151175.htm
qhtv.ygdig4s.cn/622885.htm
qhtv.ygdig4s.cn/202885.htm
qhtv.ygdig4s.cn/282225.htm
qhtv.ygdig4s.cn/951355.htm
qhn.ygdig4s.cn/402865.htm
qhn.ygdig4s.cn/719335.htm
qhn.ygdig4s.cn/448065.htm
qhn.ygdig4s.cn/993755.htm
qhn.ygdig4s.cn/826485.htm
qhn.ygdig4s.cn/246845.htm
qhn.ygdig4s.cn/484225.htm
qhn.ygdig4s.cn/531535.htm
qhn.ygdig4s.cn/282225.htm
qhn.ygdig4s.cn/931195.htm
qhb.ygdig4s.cn/775115.htm
qhb.ygdig4s.cn/828865.htm
qhb.ygdig4s.cn/571955.htm
qhb.ygdig4s.cn/359135.htm
qhb.ygdig4s.cn/026425.htm
qhb.ygdig4s.cn/593775.htm
qhb.ygdig4s.cn/315135.htm
qhb.ygdig4s.cn/404865.htm
qhb.ygdig4s.cn/242205.htm
qhb.ygdig4s.cn/951955.htm
qhv.ygdig4s.cn/864285.htm
qhv.ygdig4s.cn/353135.htm
qhv.ygdig4s.cn/591595.htm
qhv.ygdig4s.cn/717515.htm
qhv.ygdig4s.cn/468225.htm
qhv.ygdig4s.cn/337795.htm
qhv.ygdig4s.cn/339715.htm
qhv.ygdig4s.cn/753155.htm
qhv.ygdig4s.cn/266045.htm
qhv.ygdig4s.cn/888245.htm
qhc.ygdig4s.cn/262885.htm
qhc.ygdig4s.cn/008445.htm
qhc.ygdig4s.cn/735335.htm
qhc.ygdig4s.cn/042805.htm
qhc.ygdig4s.cn/280605.htm
qhc.ygdig4s.cn/028445.htm
qhc.ygdig4s.cn/400645.htm
qhc.ygdig4s.cn/484885.htm
qhc.ygdig4s.cn/262805.htm
qhc.ygdig4s.cn/317155.htm
qhx.ygdig4s.cn/422025.htm
qhx.ygdig4s.cn/117115.htm
qhx.ygdig4s.cn/759115.htm
qhx.ygdig4s.cn/004465.htm
qhx.ygdig4s.cn/844625.htm
qhx.ygdig4s.cn/159115.htm
qhx.ygdig4s.cn/682025.htm
qhx.ygdig4s.cn/737935.htm
qhx.ygdig4s.cn/288485.htm
qhx.ygdig4s.cn/593715.htm
qhz.ygdig4s.cn/020265.htm
qhz.ygdig4s.cn/197375.htm
qhz.ygdig4s.cn/575595.htm
qhz.ygdig4s.cn/971175.htm
qhz.ygdig4s.cn/808665.htm
qhz.ygdig4s.cn/606205.htm
qhz.ygdig4s.cn/806265.htm
qhz.ygdig4s.cn/153755.htm
qhz.ygdig4s.cn/648885.htm
qhz.ygdig4s.cn/424805.htm
qhl.ygdig4s.cn/355755.htm
qhl.ygdig4s.cn/939755.htm
qhl.ygdig4s.cn/935375.htm
qhl.ygdig4s.cn/335135.htm
qhl.ygdig4s.cn/997935.htm
qhl.ygdig4s.cn/600225.htm
qhl.ygdig4s.cn/359555.htm
qhl.ygdig4s.cn/119795.htm
qhl.ygdig4s.cn/739595.htm
qhl.ygdig4s.cn/826405.htm
qhk.ygdig4s.cn/448285.htm
qhk.ygdig4s.cn/246005.htm
qhk.ygdig4s.cn/446245.htm
qhk.ygdig4s.cn/311935.htm
qhk.ygdig4s.cn/771935.htm
qhk.ygdig4s.cn/111575.htm
qhk.ygdig4s.cn/733735.htm
qhk.ygdig4s.cn/206065.htm
qhk.ygdig4s.cn/460845.htm
qhk.ygdig4s.cn/848405.htm
qhj.ygdig4s.cn/799375.htm
qhj.ygdig4s.cn/048805.htm
qhj.ygdig4s.cn/282485.htm
qhj.ygdig4s.cn/775115.htm
qhj.ygdig4s.cn/379995.htm
qhj.ygdig4s.cn/488005.htm
qhj.ygdig4s.cn/846085.htm
qhj.ygdig4s.cn/573115.htm
qhj.ygdig4s.cn/268405.htm
qhj.ygdig4s.cn/595795.htm
qhh.ygdig4s.cn/793975.htm
qhh.ygdig4s.cn/266805.htm
qhh.ygdig4s.cn/517935.htm
qhh.ygdig4s.cn/777175.htm
qhh.ygdig4s.cn/737375.htm
qhh.ygdig4s.cn/404245.htm
qhh.ygdig4s.cn/573375.htm
qhh.ygdig4s.cn/046825.htm
qhh.ygdig4s.cn/533175.htm
qhh.ygdig4s.cn/519775.htm
qhg.ygdig4s.cn/559175.htm
qhg.ygdig4s.cn/828045.htm
qhg.ygdig4s.cn/991755.htm
qhg.ygdig4s.cn/713375.htm
qhg.ygdig4s.cn/593975.htm
qhg.ygdig4s.cn/711555.htm
qhg.ygdig4s.cn/115755.htm
qhg.ygdig4s.cn/084405.htm
qhg.ygdig4s.cn/442485.htm
qhg.ygdig4s.cn/842045.htm
qhf.ygdig4s.cn/999975.htm
qhf.ygdig4s.cn/911155.htm
qhf.ygdig4s.cn/915795.htm
qhf.ygdig4s.cn/775355.htm
qhf.ygdig4s.cn/844005.htm
qhf.ygdig4s.cn/446405.htm
qhf.ygdig4s.cn/579175.htm
qhf.ygdig4s.cn/359335.htm
qhf.ygdig4s.cn/628865.htm
qhf.ygdig4s.cn/442025.htm
qhd.ygdig4s.cn/280625.htm
qhd.ygdig4s.cn/806425.htm
