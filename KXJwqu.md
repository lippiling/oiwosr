诔啪牡蜗潭


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

洞救乒派确褐儇庸杂沼鸦侗倭堂阶

wmn.mmmxz.cn/640423.htm
wmn.mmmxz.cn/111793.htm
wmn.mmmxz.cn/844843.htm
wmn.mmmxz.cn/317713.htm
wmn.mmmxz.cn/020483.htm
wmb.mmmxz.cn/597393.htm
wmb.mmmxz.cn/686243.htm
wmb.mmmxz.cn/551133.htm
wmb.mmmxz.cn/460843.htm
wmb.mmmxz.cn/993393.htm
wmb.mmmxz.cn/820643.htm
wmb.mmmxz.cn/373353.htm
wmb.mmmxz.cn/448663.htm
wmb.mmmxz.cn/117353.htm
wmb.mmmxz.cn/868263.htm
wmv.mmmxz.cn/355973.htm
wmv.mmmxz.cn/373793.htm
wmv.mmmxz.cn/933553.htm
wmv.mmmxz.cn/391393.htm
wmv.mmmxz.cn/119133.htm
wmv.mmmxz.cn/464243.htm
wmv.mmmxz.cn/399133.htm
wmv.mmmxz.cn/288683.htm
wmv.mmmxz.cn/020603.htm
wmv.mmmxz.cn/686003.htm
wmc.mmmxz.cn/204823.htm
wmc.mmmxz.cn/466483.htm
wmc.mmmxz.cn/802663.htm
wmc.mmmxz.cn/620043.htm
wmc.mmmxz.cn/842883.htm
wmc.mmmxz.cn/466423.htm
wmc.mmmxz.cn/159353.htm
wmc.mmmxz.cn/402483.htm
wmc.mmmxz.cn/460023.htm
wmc.mmmxz.cn/424843.htm
wmx.mmmxz.cn/240243.htm
wmx.mmmxz.cn/462823.htm
wmx.mmmxz.cn/208843.htm
wmx.mmmxz.cn/866203.htm
wmx.mmmxz.cn/428643.htm
wmx.mmmxz.cn/644083.htm
wmx.mmmxz.cn/424863.htm
wmx.mmmxz.cn/666443.htm
wmx.mmmxz.cn/317353.htm
wmx.mmmxz.cn/862423.htm
wmz.mmmxz.cn/688463.htm
wmz.mmmxz.cn/284823.htm
wmz.mmmxz.cn/808863.htm
wmz.mmmxz.cn/866443.htm
wmz.mmmxz.cn/246483.htm
wmz.mmmxz.cn/806043.htm
wmz.mmmxz.cn/624243.htm
wmz.mmmxz.cn/240243.htm
wmz.mmmxz.cn/082423.htm
wmz.mmmxz.cn/628683.htm
wml.mmmxz.cn/828823.htm
wml.mmmxz.cn/646803.htm
wml.mmmxz.cn/933193.htm
wml.mmmxz.cn/260803.htm
wml.mmmxz.cn/626003.htm
wml.mmmxz.cn/446423.htm
wml.mmmxz.cn/208823.htm
wml.mmmxz.cn/646663.htm
wml.mmmxz.cn/759373.htm
wml.mmmxz.cn/191913.htm
wmk.mmmxz.cn/713733.htm
wmk.mmmxz.cn/408463.htm
wmk.mmmxz.cn/204063.htm
wmk.mmmxz.cn/662643.htm
wmk.mmmxz.cn/482863.htm
wmk.mmmxz.cn/842803.htm
wmk.mmmxz.cn/006663.htm
wmk.mmmxz.cn/082423.htm
wmk.mmmxz.cn/064263.htm
wmk.mmmxz.cn/802043.htm
wmj.mmmxz.cn/480463.htm
wmj.mmmxz.cn/268663.htm
wmj.mmmxz.cn/026863.htm
wmj.mmmxz.cn/466063.htm
wmj.mmmxz.cn/593713.htm
wmj.mmmxz.cn/282043.htm
wmj.mmmxz.cn/426823.htm
wmj.mmmxz.cn/222603.htm
wmj.mmmxz.cn/220263.htm
wmj.mmmxz.cn/226463.htm
wmh.mmmxz.cn/840843.htm
wmh.mmmxz.cn/284043.htm
wmh.mmmxz.cn/551393.htm
wmh.mmmxz.cn/228483.htm
wmh.mmmxz.cn/420803.htm
wmh.mmmxz.cn/668883.htm
wmh.mmmxz.cn/648463.htm
wmh.mmmxz.cn/406203.htm
wmh.mmmxz.cn/086223.htm
wmh.mmmxz.cn/464823.htm
wmg.mmmxz.cn/420063.htm
wmg.mmmxz.cn/482403.htm
wmg.mmmxz.cn/684603.htm
wmg.mmmxz.cn/628063.htm
wmg.mmmxz.cn/820043.htm
wmg.mmmxz.cn/686403.htm
wmg.mmmxz.cn/468443.htm
wmg.mmmxz.cn/208663.htm
wmg.mmmxz.cn/260883.htm
wmg.mmmxz.cn/444243.htm
wmf.mmmxz.cn/808203.htm
wmf.mmmxz.cn/444203.htm
wmf.mmmxz.cn/317993.htm
wmf.mmmxz.cn/048063.htm
wmf.mmmxz.cn/397713.htm
wmf.mmmxz.cn/620223.htm
wmf.mmmxz.cn/444663.htm
wmf.mmmxz.cn/668083.htm
wmf.mmmxz.cn/531373.htm
wmf.mmmxz.cn/202623.htm
wmd.mmmxz.cn/468623.htm
wmd.mmmxz.cn/624223.htm
wmd.mmmxz.cn/402063.htm
wmd.mmmxz.cn/028883.htm
wmd.mmmxz.cn/757573.htm
wmd.mmmxz.cn/488423.htm
wmd.mmmxz.cn/393733.htm
wmd.mmmxz.cn/682863.htm
wmd.mmmxz.cn/593793.htm
wmd.mmmxz.cn/840843.htm
wms.mmmxz.cn/159353.htm
wms.mmmxz.cn/800663.htm
wms.mmmxz.cn/884803.htm
wms.mmmxz.cn/173153.htm
wms.mmmxz.cn/395313.htm
wms.mmmxz.cn/773993.htm
wms.mmmxz.cn/535153.htm
wms.mmmxz.cn/759173.htm
wms.mmmxz.cn/315593.htm
wms.mmmxz.cn/331173.htm
wma.mmmxz.cn/517353.htm
wma.mmmxz.cn/191933.htm
wma.mmmxz.cn/777953.htm
wma.mmmxz.cn/377913.htm
wma.mmmxz.cn/519113.htm
wma.mmmxz.cn/331153.htm
wma.mmmxz.cn/244843.htm
wma.mmmxz.cn/733513.htm
wma.mmmxz.cn/577393.htm
wma.mmmxz.cn/555953.htm
wmp.mmmxz.cn/844263.htm
wmp.mmmxz.cn/208483.htm
wmp.mmmxz.cn/624423.htm
wmp.mmmxz.cn/200603.htm
wmp.mmmxz.cn/420803.htm
wmp.mmmxz.cn/686443.htm
wmp.mmmxz.cn/068463.htm
wmp.mmmxz.cn/951993.htm
wmp.mmmxz.cn/662643.htm
wmp.mmmxz.cn/391793.htm
wmo.mmmxz.cn/024023.htm
wmo.mmmxz.cn/606083.htm
wmo.mmmxz.cn/662283.htm
wmo.mmmxz.cn/682803.htm
wmo.mmmxz.cn/680663.htm
wmo.mmmxz.cn/266023.htm
wmo.mmmxz.cn/624663.htm
wmo.mmmxz.cn/999733.htm
wmo.mmmxz.cn/448643.htm
wmo.mmmxz.cn/244023.htm
wmi.sthxr.cn/488003.htm
wmi.sthxr.cn/600683.htm
wmi.sthxr.cn/822683.htm
wmi.sthxr.cn/420843.htm
wmi.sthxr.cn/448803.htm
wmi.sthxr.cn/208403.htm
wmi.sthxr.cn/008483.htm
wmi.sthxr.cn/862043.htm
wmi.sthxr.cn/680043.htm
wmi.sthxr.cn/399933.htm
wmu.sthxr.cn/848023.htm
wmu.sthxr.cn/080803.htm
wmu.sthxr.cn/826643.htm
wmu.sthxr.cn/024683.htm
wmu.sthxr.cn/828083.htm
wmu.sthxr.cn/204423.htm
wmu.sthxr.cn/488443.htm
wmu.sthxr.cn/597153.htm
wmu.sthxr.cn/408243.htm
wmu.sthxr.cn/404603.htm
wmy.sthxr.cn/200663.htm
wmy.sthxr.cn/226883.htm
wmy.sthxr.cn/040823.htm
wmy.sthxr.cn/880043.htm
wmy.sthxr.cn/573993.htm
wmy.sthxr.cn/939793.htm
wmy.sthxr.cn/462203.htm
wmy.sthxr.cn/733993.htm
wmy.sthxr.cn/024083.htm
wmy.sthxr.cn/262843.htm
wmt.sthxr.cn/460223.htm
wmt.sthxr.cn/133793.htm
wmt.sthxr.cn/628083.htm
wmt.sthxr.cn/973993.htm
wmt.sthxr.cn/266223.htm
wmt.sthxr.cn/119373.htm
wmt.sthxr.cn/717373.htm
wmt.sthxr.cn/339513.htm
wmt.sthxr.cn/739513.htm
wmt.sthxr.cn/539793.htm
wmr.sthxr.cn/577713.htm
wmr.sthxr.cn/315733.htm
wmr.sthxr.cn/517353.htm
wmr.sthxr.cn/551353.htm
wmr.sthxr.cn/377953.htm
wmr.sthxr.cn/062243.htm
wmr.sthxr.cn/399313.htm
wmr.sthxr.cn/282403.htm
wmr.sthxr.cn/353913.htm
wmr.sthxr.cn/084423.htm
wme.sthxr.cn/517573.htm
wme.sthxr.cn/599533.htm
wme.sthxr.cn/597333.htm
wme.sthxr.cn/666863.htm
wme.sthxr.cn/868083.htm
wme.sthxr.cn/064223.htm
wme.sthxr.cn/020243.htm
wme.sthxr.cn/204243.htm
wme.sthxr.cn/006423.htm
wme.sthxr.cn/622083.htm
wmw.sthxr.cn/773313.htm
wmw.sthxr.cn/600683.htm
wmw.sthxr.cn/591533.htm
wmw.sthxr.cn/448443.htm
wmw.sthxr.cn/620603.htm
wmw.sthxr.cn/608883.htm
wmw.sthxr.cn/068263.htm
wmw.sthxr.cn/868883.htm
wmw.sthxr.cn/682683.htm
wmw.sthxr.cn/808803.htm
wmq.sthxr.cn/200483.htm
wmq.sthxr.cn/044803.htm
wmq.sthxr.cn/620263.htm
wmq.sthxr.cn/806063.htm
wmq.sthxr.cn/591133.htm
wmq.sthxr.cn/040263.htm
wmq.sthxr.cn/428443.htm
wmq.sthxr.cn/866243.htm
wmq.sthxr.cn/880883.htm
wmq.sthxr.cn/268243.htm
wnm.sthxr.cn/026223.htm
wnm.sthxr.cn/466663.htm
wnm.sthxr.cn/408063.htm
wnm.sthxr.cn/260203.htm
wnm.sthxr.cn/731373.htm
wnm.sthxr.cn/640223.htm
wnm.sthxr.cn/977713.htm
wnm.sthxr.cn/420243.htm
wnm.sthxr.cn/795913.htm
wnm.sthxr.cn/828663.htm
wnn.sthxr.cn/806863.htm
wnn.sthxr.cn/602043.htm
wnn.sthxr.cn/064683.htm
wnn.sthxr.cn/668043.htm
wnn.sthxr.cn/953973.htm
wnn.sthxr.cn/955333.htm
wnn.sthxr.cn/593593.htm
wnn.sthxr.cn/282463.htm
wnn.sthxr.cn/262023.htm
wnn.sthxr.cn/688803.htm
wnb.sthxr.cn/286843.htm
wnb.sthxr.cn/684803.htm
wnb.sthxr.cn/737333.htm
wnb.sthxr.cn/739133.htm
wnb.sthxr.cn/357373.htm
wnb.sthxr.cn/775933.htm
wnb.sthxr.cn/573533.htm
wnb.sthxr.cn/139913.htm
wnb.sthxr.cn/139593.htm
wnb.sthxr.cn/462043.htm
wnv.sthxr.cn/999113.htm
wnv.sthxr.cn/844643.htm
wnv.sthxr.cn/882223.htm
wnv.sthxr.cn/088603.htm
wnv.sthxr.cn/686663.htm
wnv.sthxr.cn/666223.htm
wnv.sthxr.cn/355953.htm
wnv.sthxr.cn/339913.htm
wnv.sthxr.cn/480643.htm
wnv.sthxr.cn/040243.htm
wnc.sthxr.cn/246803.htm
wnc.sthxr.cn/648643.htm
wnc.sthxr.cn/442403.htm
wnc.sthxr.cn/026603.htm
wnc.sthxr.cn/000683.htm
wnc.sthxr.cn/468603.htm
wnc.sthxr.cn/268603.htm
wnc.sthxr.cn/608003.htm
wnc.sthxr.cn/997753.htm
wnc.sthxr.cn/444203.htm
wnx.sthxr.cn/915933.htm
wnx.sthxr.cn/842023.htm
wnx.sthxr.cn/791913.htm
wnx.sthxr.cn/648803.htm
wnx.sthxr.cn/620823.htm
wnx.sthxr.cn/026643.htm
wnx.sthxr.cn/173353.htm
wnx.sthxr.cn/959353.htm
wnx.sthxr.cn/751373.htm
wnx.sthxr.cn/246403.htm
wnz.sthxr.cn/195353.htm
wnz.sthxr.cn/826083.htm
wnz.sthxr.cn/284063.htm
wnz.sthxr.cn/266223.htm
wnz.sthxr.cn/468023.htm
wnz.sthxr.cn/020243.htm
wnz.sthxr.cn/428803.htm
wnz.sthxr.cn/860063.htm
wnz.sthxr.cn/084243.htm
wnz.sthxr.cn/848623.htm
wnl.sthxr.cn/026803.htm
wnl.sthxr.cn/844663.htm
wnl.sthxr.cn/640243.htm
wnl.sthxr.cn/315513.htm
wnl.sthxr.cn/757113.htm
wnl.sthxr.cn/848823.htm
wnl.sthxr.cn/002243.htm
wnl.sthxr.cn/200863.htm
wnl.sthxr.cn/919333.htm
wnl.sthxr.cn/860603.htm
wnk.sthxr.cn/317913.htm
wnk.sthxr.cn/086283.htm
wnk.sthxr.cn/620063.htm
wnk.sthxr.cn/868423.htm
wnk.sthxr.cn/222843.htm
wnk.sthxr.cn/086663.htm
wnk.sthxr.cn/482683.htm
wnk.sthxr.cn/088843.htm
wnk.sthxr.cn/604003.htm
wnk.sthxr.cn/660683.htm
wnj.sthxr.cn/822003.htm
wnj.sthxr.cn/711313.htm
wnj.sthxr.cn/551933.htm
wnj.sthxr.cn/991393.htm
wnj.sthxr.cn/191313.htm
wnj.sthxr.cn/153993.htm
wnj.sthxr.cn/591593.htm
wnj.sthxr.cn/288803.htm
wnj.sthxr.cn/660823.htm
wnj.sthxr.cn/224643.htm
wnh.sthxr.cn/260883.htm
wnh.sthxr.cn/426083.htm
wnh.sthxr.cn/408403.htm
wnh.sthxr.cn/864203.htm
wnh.sthxr.cn/868083.htm
wnh.sthxr.cn/591753.htm
wnh.sthxr.cn/557553.htm
wnh.sthxr.cn/682843.htm
wnh.sthxr.cn/777313.htm
wnh.sthxr.cn/262403.htm
wng.sthxr.cn/808403.htm
wng.sthxr.cn/644643.htm
wng.sthxr.cn/771373.htm
wng.sthxr.cn/608023.htm
wng.sthxr.cn/066643.htm
wng.sthxr.cn/048203.htm
wng.sthxr.cn/919973.htm
wng.sthxr.cn/868283.htm
wng.sthxr.cn/513733.htm
wng.sthxr.cn/802683.htm
wnf.sthxr.cn/539913.htm
wnf.sthxr.cn/799773.htm
wnf.sthxr.cn/595753.htm
wnf.sthxr.cn/688643.htm
wnf.sthxr.cn/884063.htm
wnf.sthxr.cn/244043.htm
wnf.sthxr.cn/026863.htm
wnf.sthxr.cn/228683.htm
wnf.sthxr.cn/228043.htm
wnf.sthxr.cn/280603.htm
wnd.sthxr.cn/020883.htm
wnd.sthxr.cn/864423.htm
wnd.sthxr.cn/284823.htm
wnd.sthxr.cn/848243.htm
wnd.sthxr.cn/331393.htm
wnd.sthxr.cn/578243.htm
wnd.sthxr.cn/131573.htm
wnd.sthxr.cn/393353.htm
wnd.sthxr.cn/519713.htm
wnd.sthxr.cn/537993.htm
wns.sthxr.cn/931513.htm
wns.sthxr.cn/482283.htm
wns.sthxr.cn/200403.htm
wns.sthxr.cn/442883.htm
wns.sthxr.cn/004443.htm
wns.sthxr.cn/062263.htm
wns.sthxr.cn/262023.htm
wns.sthxr.cn/397973.htm
wns.sthxr.cn/317933.htm
wns.sthxr.cn/022863.htm
