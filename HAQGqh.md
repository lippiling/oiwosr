郊烈柏肚聘


========================================================
 Python 元编程与代码生成 — 深入理解运行时的代码操控
========================================================

元编程（Metaprogramming）是指编写能够操作代码本身的代码。
Python 作为动态语言，提供了极为丰富的元编程工具。本文
将逐一剖析这些工具的实际用法与典型场景。

--------------------------------------------------------
1. 使用 type() 动态创建类
--------------------------------------------------------

除了检查对象类型，type() 还可以用三参数形式动态创建类：

# 定义父类
class Base:
    def greet(self):
        return f"Hello from {self.name}"

# 使用 type(name, bases, dict) 动态创建类
DynamicClass = type(
    "DynamicClass",          # 类名
    (Base,),                 # 父类元组
    {
        "name": "动态类",
        "add": lambda self, a, b: a + b,  # 动态添加方法
    }
)

obj = DynamicClass()
print(obj.greet())   # Hello from 动态类
print(obj.add(3, 4)) # 7

这在实现工厂模式或 ORM 字段映射时非常有用。

--------------------------------------------------------
2. setattr / getattr 动态属性处理
--------------------------------------------------------

class Config:
    pass

cfg = Config()
# 动态设置属性名（属性名来自变量或用户输入）
for key, val in [("host", "localhost"), ("port", 5432)]:
    setattr(cfg, key, val)

# 动态读取，提供默认值避免 AttributeError
host = getattr(cfg, "host", "127.0.0.1")
timeout = getattr(cfg, "timeout", 30)  # 不存在则返回默认值
print(host, timeout)  # localhost 30

--------------------------------------------------------
3. exec / eval 的安全使用模式
--------------------------------------------------------

直接使用 exec/eval 执行用户输入是危险的。推荐用限定
命名空间的方式来控制可访问的范围：

SAFE_GLOBALS = {"__builtins__": {}}  # 禁用所有内置函数
SAFE_LOCALS = {"math": __import__("math")}

expr = "math.sqrt(16) + 9"
result = eval(expr, SAFE_GLOBALS, SAFE_LOCALS)
print(result)  # 13.0

# exec 执行多行代码同样适用
code = """
result = []
for i in range(5):
    result.append(i ** 2)
"""
exec(code, SAFE_GLOBALS, SAFE_LOCALS)
print(SAFE_LOCALS["result"])  # [0, 1, 4, 9, 16]

--------------------------------------------------------
4. inspect 模块 — 源代码内省
--------------------------------------------------------

import inspect

def example_func(a: int, b: str = "default") -> bool:
    """示例函数"""
    return True

# 获取函数签名
sig = inspect.signature(example_func)
for name, param in sig.parameters.items():
    print(f"参数 {name}: 类型={param.annotation}, 默认={param.default}")
# 输出:
#   参数 a: 类型=<class 'int'>, 默认=<class 'inspect._empty'>
#   参数 b: 类型=<class 'str'>, 默认=default

# 获取源代码
print(inspect.getsource(example_func))

# 检查是否类/函数/模块
print(inspect.isfunction(example_func))  # True

--------------------------------------------------------
5. AST 操作入门
--------------------------------------------------------

AST（抽象语法树）让我们在编译前分析和修改代码。

import ast

code_str = "x = 1 + 2 * 3"
tree = ast.parse(code_str)

# 遍历 AST 节点
class NodeVisitor(ast.NodeVisitor):
    def visit_BinOp(self, node):
        print(f"二元运算: {type(node.op).__name__}")
        self.generic_visit(node)  # 继续访问子节点

NodeVisitor().visit(tree)

# 动态编译 AST 并执行
compiled = compile(tree, filename="<ast>", mode="exec")
exec(compiled)
print(x)  # 7

--------------------------------------------------------
6. __init_subclass__ 自动注册子类
--------------------------------------------------------

class PluginRegistry(dict):
    """插件注册表，自动收集所有子类"""
    def register(self, cls):
        self[cls.__name__] = cls
        return cls

registry = PluginRegistry()

class BasePlugin:
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        registry.register(cls)
        # 自动为子类添加元信息
        cls.plugin_name = cls.__name__.lower()

class LogPlugin(BasePlugin):
    def run(self): print("日志插件运行")

class AlertPlugin(BasePlugin):
    def run(self): print("告警插件运行")

print(list(registry.keys()))  # ['LogPlugin', 'AlertPlugin']
print(LogPlugin.plugin_name)  # 'logplugin'

--------------------------------------------------------
7. 类装饰器 — 运行时修改类
--------------------------------------------------------

def add_method(cls):
    """类装饰器：为类动态添加方法"""
    def new_method(self):
        return f"{self.__class__.__name__} 的附加方法"
    cls.extra = new_method
    return cls

def singleton(cls):
    """类装饰器：实现单例模式"""
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
@add_method
class MyService:
    def __init__(self):
        self.name = "服务"

s1 = MyService()
s2 = MyService()
print(s1 is s2)       # True（单例生效）
print(s1.extra())     # 'MyService 的附加方法'

--------------------------------------------------------
8. 代码生成实战：动态构建 API 客户端
--------------------------------------------------------

def build_api_client(base_url, endpoints):
    """根据配置动态生成 API 客户端类"""
    def make_request(method, path):
        def request(self, **params):
            url = f"{base_url}{path}"
            print(f"[{method.upper()}] {url} params={params}")
            # 实际可用 requests 库发送 HTTP 请求
            return {"status": 200, "data": params}
        return request

    methods = {}
    for name, config in endpoints.items():
        methods[name] = make_request(config["method"], config["path"])

    return type("ApiClient", (), methods)

# 使用示例
endpoints = {
    "get_user": {"method": "get", "path": "/users/{id}"},
    "create_post": {"method": "post", "path": "/posts"},
}
Client = build_api_client("https://api.example.com", endpoints)
client = Client()
client.get_user(id=123)
client.create_post(title="Hello", body="World")

元编程是一把双刃剑：适当使用可以大幅减少重复代码，
过度使用则会让代码难以理解和调试。建议仅在框架、
库或工具类代码中使用，业务逻辑层保持直白即可。

反盖几斩砂档惺型谙挡棠谔裁悍刹

fhm.fffbf.cn/595999.Doc
fhm.fffbf.cn/688222.Doc
fhm.fffbf.cn/286248.Doc
fhn.fffbf.cn/531599.Doc
fhn.fffbf.cn/064628.Doc
fhn.fffbf.cn/888026.Doc
fhn.fffbf.cn/482648.Doc
fhn.fffbf.cn/688800.Doc
fhn.fffbf.cn/408286.Doc
fhn.fffbf.cn/026408.Doc
fhn.fffbf.cn/642084.Doc
fhn.fffbf.cn/044224.Doc
fhn.fffbf.cn/088482.Doc
fhb.fffbf.cn/402002.Doc
fhb.fffbf.cn/739935.Doc
fhb.fffbf.cn/262084.Doc
fhb.fffbf.cn/200888.Doc
fhb.fffbf.cn/711973.Doc
fhb.fffbf.cn/424620.Doc
fhb.fffbf.cn/088608.Doc
fhb.fffbf.cn/682080.Doc
fhb.fffbf.cn/264808.Doc
fhb.fffbf.cn/822800.Doc
fhv.fffbf.cn/246488.Doc
fhv.fffbf.cn/004848.Doc
fhv.fffbf.cn/662426.Doc
fhv.fffbf.cn/680866.Doc
fhv.fffbf.cn/226662.Doc
fhv.fffbf.cn/022064.Doc
fhv.fffbf.cn/042444.Doc
fhv.fffbf.cn/666224.Doc
fhv.fffbf.cn/622482.Doc
fhv.fffbf.cn/155731.Doc
fhc.fffbf.cn/648422.Doc
fhc.fffbf.cn/866840.Doc
fhc.fffbf.cn/620028.Doc
fhc.fffbf.cn/060486.Doc
fhc.fffbf.cn/040462.Doc
fhc.fffbf.cn/006680.Doc
fhc.fffbf.cn/660428.Doc
fhc.fffbf.cn/173953.Doc
fhc.fffbf.cn/466240.Doc
fhc.fffbf.cn/600828.Doc
fhx.fffbf.cn/682408.Doc
fhx.fffbf.cn/751571.Doc
fhx.fffbf.cn/046424.Doc
fhx.fffbf.cn/822064.Doc
fhx.fffbf.cn/284468.Doc
fhx.fffbf.cn/408804.Doc
fhx.fffbf.cn/802646.Doc
fhx.fffbf.cn/400826.Doc
fhx.fffbf.cn/268224.Doc
fhx.fffbf.cn/244042.Doc
fhz.fffbf.cn/173339.Doc
fhz.fffbf.cn/888628.Doc
fhz.fffbf.cn/066244.Doc
fhz.fffbf.cn/240064.Doc
fhz.fffbf.cn/604664.Doc
fhz.fffbf.cn/804080.Doc
fhz.fffbf.cn/662222.Doc
fhz.fffbf.cn/848022.Doc
fhz.fffbf.cn/771191.Doc
fhz.fffbf.cn/620404.Doc
fhl.fffbf.cn/884880.Doc
fhl.fffbf.cn/860828.Doc
fhl.fffbf.cn/088082.Doc
fhl.fffbf.cn/284640.Doc
fhl.fffbf.cn/480288.Doc
fhl.fffbf.cn/080004.Doc
fhl.fffbf.cn/680798.Doc
fhl.fffbf.cn/537711.Doc
fhl.fffbf.cn/155304.Doc
fhl.fffbf.cn/648280.Doc
fhk.fffbf.cn/486640.Doc
fhk.fffbf.cn/840606.Doc
fhk.fffbf.cn/228068.Doc
fhk.fffbf.cn/460608.Doc
fhk.fffbf.cn/206282.Doc
fhk.fffbf.cn/262482.Doc
fhk.fffbf.cn/026046.Doc
fhk.fffbf.cn/841225.Doc
fhk.fffbf.cn/248620.Doc
fhk.fffbf.cn/002628.Doc
fhj.fffbf.cn/840004.Doc
fhj.fffbf.cn/206420.Doc
fhj.fffbf.cn/088800.Doc
fhj.fffbf.cn/575753.Doc
fhj.fffbf.cn/246860.Doc
fhj.fffbf.cn/684484.Doc
fhj.fffbf.cn/646244.Doc
fhj.fffbf.cn/975131.Doc
fhj.fffbf.cn/224884.Doc
fhj.fffbf.cn/602062.Doc
fhh.fffbf.cn/028406.Doc
fhh.fffbf.cn/086066.Doc
fhh.fffbf.cn/066040.Doc
fhh.fffbf.cn/820204.Doc
fhh.fffbf.cn/537199.Doc
fhh.fffbf.cn/884486.Doc
fhh.fffbf.cn/553391.Doc
fhh.fffbf.cn/173573.Doc
fhh.fffbf.cn/682000.Doc
fhh.fffbf.cn/154062.Doc
fhg.fffbf.cn/173113.Doc
fhg.fffbf.cn/444442.Doc
fhg.fffbf.cn/153733.Doc
fhg.fffbf.cn/840468.Doc
fhg.fffbf.cn/262402.Doc
fhg.fffbf.cn/315195.Doc
fhg.fffbf.cn/482422.Doc
fhg.fffbf.cn/480646.Doc
fhg.fffbf.cn/664240.Doc
fhg.fffbf.cn/709240.Doc
fhf.fffbf.cn/262282.Doc
fhf.fffbf.cn/042422.Doc
fhf.fffbf.cn/606208.Doc
fhf.fffbf.cn/200020.Doc
fhf.fffbf.cn/559911.Doc
fhf.fffbf.cn/206626.Doc
fhf.fffbf.cn/628624.Doc
fhf.fffbf.cn/000420.Doc
fhf.fffbf.cn/204624.Doc
fhf.fffbf.cn/844464.Doc
fhd.fffbf.cn/395331.Doc
fhd.fffbf.cn/840282.Doc
fhd.fffbf.cn/262020.Doc
fhd.fffbf.cn/804622.Doc
fhd.fffbf.cn/975391.Doc
fhd.fffbf.cn/604226.Doc
fhd.fffbf.cn/066464.Doc
fhd.fffbf.cn/008286.Doc
fhd.fffbf.cn/377575.Doc
fhd.fffbf.cn/266860.Doc
fhs.fffbf.cn/355757.Doc
fhs.fffbf.cn/983026.Doc
fhs.fffbf.cn/438360.Doc
fhs.fffbf.cn/739915.Doc
fhs.fffbf.cn/682229.Doc
fhs.fffbf.cn/951573.Doc
fhs.fffbf.cn/000084.Doc
fhs.fffbf.cn/468622.Doc
fhs.fffbf.cn/165040.Doc
fhs.fffbf.cn/028042.Doc
fha.fffbf.cn/042248.Doc
fha.fffbf.cn/977599.Doc
fha.fffbf.cn/422680.Doc
fha.fffbf.cn/284866.Doc
fha.fffbf.cn/460206.Doc
fha.fffbf.cn/084608.Doc
fha.fffbf.cn/999757.Doc
fha.fffbf.cn/620806.Doc
fha.fffbf.cn/024408.Doc
fha.fffbf.cn/026826.Doc
fhp.fffbf.cn/060260.Doc
fhp.fffbf.cn/604822.Doc
fhp.fffbf.cn/046088.Doc
fhp.fffbf.cn/682826.Doc
fhp.fffbf.cn/466048.Doc
fhp.fffbf.cn/515106.Doc
fhp.fffbf.cn/973074.Doc
fhp.fffbf.cn/488606.Doc
fhp.fffbf.cn/088468.Doc
fhp.fffbf.cn/379315.Doc
fho.fffbf.cn/862060.Doc
fho.fffbf.cn/355753.Doc
fho.fffbf.cn/440066.Doc
fho.fffbf.cn/626224.Doc
fho.fffbf.cn/660004.Doc
fho.fffbf.cn/562650.Doc
fho.fffbf.cn/460882.Doc
fho.fffbf.cn/131971.Doc
fho.fffbf.cn/666048.Doc
fho.fffbf.cn/420240.Doc
fhi.fffbf.cn/680660.Doc
fhi.fffbf.cn/800860.Doc
fhi.fffbf.cn/228442.Doc
fhi.fffbf.cn/006880.Doc
fhi.fffbf.cn/619747.Doc
fhi.fffbf.cn/757331.Doc
fhi.fffbf.cn/448662.Doc
fhi.fffbf.cn/086606.Doc
fhi.fffbf.cn/082460.Doc
fhi.fffbf.cn/319393.Doc
fhu.fffbf.cn/800020.Doc
fhu.fffbf.cn/644220.Doc
fhu.fffbf.cn/888202.Doc
fhu.fffbf.cn/204202.Doc
fhu.fffbf.cn/442644.Doc
fhu.fffbf.cn/175155.Doc
fhu.fffbf.cn/620420.Doc
fhu.fffbf.cn/915795.Doc
fhu.fffbf.cn/864642.Doc
fhu.fffbf.cn/868602.Doc
fhy.fffbf.cn/000468.Doc
fhy.fffbf.cn/577517.Doc
fhy.fffbf.cn/531731.Doc
fhy.fffbf.cn/684840.Doc
fhy.fffbf.cn/604842.Doc
fhy.fffbf.cn/848066.Doc
fhy.fffbf.cn/822006.Doc
fhy.fffbf.cn/046464.Doc
fhy.fffbf.cn/984194.Doc
fhy.fffbf.cn/820044.Doc
fht.fffbf.cn/242044.Doc
fht.fffbf.cn/648664.Doc
fht.fffbf.cn/646224.Doc
fht.fffbf.cn/453159.Doc
fht.fffbf.cn/088084.Doc
fht.fffbf.cn/420880.Doc
fht.fffbf.cn/026622.Doc
fht.fffbf.cn/286408.Doc
fht.fffbf.cn/084808.Doc
fht.fffbf.cn/406848.Doc
fhr.fffbf.cn/602206.Doc
fhr.fffbf.cn/596712.Doc
fhr.fffbf.cn/866286.Doc
fhr.fffbf.cn/266268.Doc
fhr.fffbf.cn/995711.Doc
fhr.fffbf.cn/404004.Doc
fhr.fffbf.cn/020404.Doc
fhr.fffbf.cn/688406.Doc
fhr.fffbf.cn/802226.Doc
fhr.fffbf.cn/628022.Doc
fhe.fffbf.cn/648840.Doc
fhe.fffbf.cn/042624.Doc
fhe.fffbf.cn/069743.Doc
fhe.fffbf.cn/040224.Doc
fhe.fffbf.cn/816287.Doc
fhe.fffbf.cn/884682.Doc
fhe.fffbf.cn/648282.Doc
fhe.fffbf.cn/028460.Doc
fhe.fffbf.cn/466646.Doc
fhe.fffbf.cn/424224.Doc
fhw.fffbf.cn/629187.Doc
fhw.fffbf.cn/616284.Doc
fhw.fffbf.cn/860486.Doc
fhw.fffbf.cn/130210.Doc
fhw.fffbf.cn/228046.Doc
fhw.fffbf.cn/286860.Doc
fhw.fffbf.cn/004804.Doc
fhw.fffbf.cn/404062.Doc
fhw.fffbf.cn/800662.Doc
fhw.fffbf.cn/240486.Doc
fhq.fffbf.cn/282468.Doc
fhq.fffbf.cn/124109.Doc
fhq.fffbf.cn/488024.Doc
fhq.fffbf.cn/464646.Doc
fhq.fffbf.cn/244662.Doc
fhq.fffbf.cn/444644.Doc
fhq.fffbf.cn/802606.Doc
fhq.fffbf.cn/640884.Doc
fhq.fffbf.cn/408668.Doc
fhq.fffbf.cn/688462.Doc
fgm.fffbf.cn/468288.Doc
fgm.fffbf.cn/408046.Doc
fgm.fffbf.cn/062860.Doc
fgm.fffbf.cn/006240.Doc
fgm.fffbf.cn/913937.Doc
fgm.fffbf.cn/864080.Doc
fgm.fffbf.cn/042426.Doc
fgm.fffbf.cn/482604.Doc
fgm.fffbf.cn/268428.Doc
fgm.fffbf.cn/993399.Doc
fgn.fffbf.cn/424048.Doc
fgn.fffbf.cn/753151.Doc
fgn.fffbf.cn/786691.Doc
fgn.fffbf.cn/117559.Doc
fgn.fffbf.cn/446846.Doc
fgn.fffbf.cn/266206.Doc
fgn.fffbf.cn/284084.Doc
fgn.fffbf.cn/604644.Doc
fgn.fffbf.cn/797799.Doc
fgn.fffbf.cn/248086.Doc
fgb.fffbf.cn/956140.Doc
fgb.fffbf.cn/882408.Doc
fgb.fffbf.cn/824286.Doc
fgb.fffbf.cn/995933.Doc
fgb.fffbf.cn/826644.Doc
fgb.fffbf.cn/620248.Doc
fgb.fffbf.cn/648420.Doc
fgb.fffbf.cn/066008.Doc
fgb.fffbf.cn/155939.Doc
fgb.fffbf.cn/946302.Doc
fgv.fffbf.cn/648406.Doc
fgv.fffbf.cn/422084.Doc
fgv.fffbf.cn/046224.Doc
fgv.fffbf.cn/602848.Doc
fgv.fffbf.cn/042604.Doc
fgv.fffbf.cn/731357.Doc
fgv.fffbf.cn/628620.Doc
fgv.fffbf.cn/600268.Doc
fgv.fffbf.cn/886024.Doc
fgv.fffbf.cn/288644.Doc
fgc.fffbf.cn/772935.Doc
fgc.fffbf.cn/777917.Doc
fgc.fffbf.cn/446608.Doc
fgc.fffbf.cn/669268.Doc
fgc.fffbf.cn/800084.Doc
fgc.fffbf.cn/882068.Doc
fgc.fffbf.cn/842460.Doc
fgc.fffbf.cn/406866.Doc
fgc.fffbf.cn/060240.Doc
fgc.fffbf.cn/351991.Doc
fgx.fffbf.cn/068082.Doc
fgx.fffbf.cn/026680.Doc
fgx.fffbf.cn/684820.Doc
fgx.fffbf.cn/862284.Doc
fgx.fffbf.cn/164561.Doc
fgx.fffbf.cn/868862.Doc
fgx.fffbf.cn/044020.Doc
fgx.fffbf.cn/402800.Doc
fgx.fffbf.cn/819105.Doc
fgx.fffbf.cn/973573.Doc
fgz.fffbf.cn/824680.Doc
fgz.fffbf.cn/022460.Doc
fgz.fffbf.cn/850424.Doc
fgz.fffbf.cn/666408.Doc
fgz.fffbf.cn/060060.Doc
fgz.fffbf.cn/204406.Doc
fgz.fffbf.cn/882224.Doc
fgz.fffbf.cn/682448.Doc
fgz.fffbf.cn/650409.Doc
fgz.fffbf.cn/064008.Doc
fgl.fffbf.cn/226468.Doc
fgl.fffbf.cn/244480.Doc
fgl.fffbf.cn/868028.Doc
fgl.fffbf.cn/408604.Doc
fgl.fffbf.cn/391399.Doc
fgl.fffbf.cn/820282.Doc
fgl.fffbf.cn/963759.Doc
fgl.fffbf.cn/404062.Doc
fgl.fffbf.cn/971971.Doc
fgl.fffbf.cn/240442.Doc
fgk.fffbf.cn/371195.Doc
fgk.fffbf.cn/195397.Doc
fgk.fffbf.cn/222222.Doc
fgk.fffbf.cn/608622.Doc
fgk.fffbf.cn/020422.Doc
fgk.fffbf.cn/793551.Doc
fgk.fffbf.cn/202866.Doc
fgk.fffbf.cn/022668.Doc
fgk.fffbf.cn/837173.Doc
fgk.fffbf.cn/640468.Doc
fgj.fffbf.cn/240020.Doc
fgj.fffbf.cn/422462.Doc
fgj.fffbf.cn/355795.Doc
fgj.fffbf.cn/680064.Doc
fgj.fffbf.cn/820886.Doc
fgj.fffbf.cn/797955.Doc
fgj.fffbf.cn/820626.Doc
fgj.fffbf.cn/004088.Doc
fgj.fffbf.cn/224886.Doc
fgj.fffbf.cn/884282.Doc
fgh.fffbf.cn/806404.Doc
fgh.fffbf.cn/244446.Doc
fgh.fffbf.cn/642808.Doc
fgh.fffbf.cn/622044.Doc
fgh.fffbf.cn/640624.Doc
fgh.fffbf.cn/547523.Doc
fgh.fffbf.cn/531573.Doc
fgh.fffbf.cn/244640.Doc
fgh.fffbf.cn/957571.Doc
fgh.fffbf.cn/268026.Doc
fgg.fffbf.cn/117713.Doc
fgg.fffbf.cn/048482.Doc
fgg.fffbf.cn/822482.Doc
fgg.fffbf.cn/577593.Doc
fgg.fffbf.cn/193177.Doc
fgg.fffbf.cn/666860.Doc
fgg.fffbf.cn/484082.Doc
fgg.fffbf.cn/066888.Doc
fgg.fffbf.cn/042646.Doc
fgg.fffbf.cn/204840.Doc
fgf.fffbf.cn/868822.Doc
fgf.fffbf.cn/286626.Doc
fgf.fffbf.cn/606024.Doc
fgf.fffbf.cn/577807.Doc
fgf.fffbf.cn/822642.Doc
fgf.fffbf.cn/111939.Doc
fgf.fffbf.cn/424826.Doc
fgf.fffbf.cn/917177.Doc
fgf.fffbf.cn/060648.Doc
fgf.fffbf.cn/426482.Doc
fgd.fffbf.cn/446202.Doc
fgd.fffbf.cn/068840.Doc
fgd.fffbf.cn/399351.Doc
fgd.fffbf.cn/242026.Doc
fgd.fffbf.cn/464828.Doc
fgd.fffbf.cn/260842.Doc
fgd.fffbf.cn/220024.Doc
fgd.fffbf.cn/428222.Doc
fgd.fffbf.cn/408004.Doc
fgd.fffbf.cn/664248.Doc
fgs.fffbf.cn/406282.Doc
fgs.fffbf.cn/815628.Doc
