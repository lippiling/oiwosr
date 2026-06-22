读购嘉滔匙


Python contextvars 上下文变量
==================================

contextvars 模块提供上下文变量，用于在异步任务中安全传递状态，
避免显式传递参数。Python 3.7+ 引入，替代 threading.local 用于异步场景。

1. ContextVar 基本用法
---------------------------
ContextVar 在每个上下文（协程/任务）中独立存储值。

import contextvars

# 声明上下文变量（在模块级别定义）
request_id = contextvars.ContextVar("request_id", default="未知")


def handle_request():
    """模拟请求处理——读取当前上下文变量"""
    current = request_id.get()  # 获取当前上下文的值
    print(f"处理请求: {current}")
    return current


def process_task(task_name: str):
    """在上下文中设置变量"""
    token = request_id.set(f"REQ-{task_name}")  # 设置值，返回 token
    try:
        result = handle_request()
        return result
    finally:
        request_id.reset(token)  # 恢复之前的值


# 不同调用互不干扰
process_task("任务A")  # 输出：处理请求: REQ-任务A
process_task("任务B")  # 输出：处理请求: REQ-任务B
print(request_id.get())  # "未知"——已恢复为默认值


2. ContextVar 在 asyncio 中的使用
--------------------------------------
每个协程各自拥有独立的上下文变量。

import asyncio
import random

user_context = contextvars.ContextVar("user_context")


async def fetch_data(user_id: int):
    """模拟异步数据获取——每个协程独立上下文"""
    user_context.set(f"user_{user_id}")
    await asyncio.sleep(random.uniform(0.1, 0.3))
    current = user_context.get()
    print(f"协程 {user_id}: 当前用户 = {current}")
    return current


async def main():
    """并发运行多个协程——上下文变量各自独立"""
    tasks = [fetch_data(i) for i in range(1, 4)]
    await asyncio.gather(*tasks)

    # 注意：ContextVar 在线程中也有效
    print(f"主协程: {user_context.get('无用户')}")


# asyncio.run(main())
# 输出（顺序可能不同）：
# 协程 1: 当前用户 = user_1
# 协程 2: 当前用户 = user_2
# 协程 3: 当前用户 = user_3
# 主协程: 无用户


3. Context.run()——在指定上下文中执行
------------------------------------------
可以在特定上下文中运行代码块。

import contextvars

ctx_var = contextvars.ContextVar("ctx_var")
ctx_var.set("原始值")

# 创建一个新上下文并设置值
ctx = contextvars.copy_context()  # 复制当前上下文

def run_in_context(value: str) -> None:
    """在新上下文中运行的函数"""
    ctx_var.set(value)
    print(f"上下文内获取: {ctx_var.get()}")


# 在特定上下文中运行函数
ctx.run(run_in_context, "上下文A的值")
print(f"原始上下文: {ctx_var.get()}")  # "原始值"——未受影响


4. contextvars.copy_context() 高级用法
------------------------------------------
获取当前上下文的快照，用于任务间传递。

REQUEST_CTX = contextvars.ContextVar("request_ctx")

def process_in_context(ctx: contextvars.Context):
    """在给定的上下文中处理"""
    def worker():
        print(f"处理: {REQUEST_CTX.get()}")
    ctx.run(worker)


def create_request_context(request_data: dict) -> contextvars.Context:
    """为请求创建独立上下文"""
    ctx = contextvars.copy_context()  # 复制当前上下文
    ctx.run(lambda: REQUEST_CTX.set(request_data))
    return ctx


# 模拟两个请求各自独立
ctx_a = create_request_context({"id": 1, "path": "/api/a"})
ctx_b = create_request_context({"id": 2, "path": "/api/b"})

process_in_context(ctx_a)  # 处理: {'id': 1, 'path': '/api/a'}
process_in_context(ctx_b)  # 处理: {'id': 2, 'path': '/api/b'}


5. vs threading.local
--------------------------
threading.local 不适合异步场景——协程可能在线程间切换。

import threading

thread_data = threading.local()
async def bad_async_example():
    threading.local()  # 在线程之间共享，不适用于协程！


# ContextVar 的优势：
# 1. 支持 asyncio——协程切换时自动维护
# 2. 支持显式上下文传递（copy_context + run）
# 3. 类型安全——get()/set() 有明确语义

# 下面的代码展示 threading.local 在线程中工作但异步中出问题
def thread_worker(name: str):
    thread_data.name = name
    import time
    time.sleep(0.1)
    print(f"线程: {thread_data.name}")


from threading import Thread
t1 = Thread(target=thread_worker, args=("线程A",))
t2 = Thread(target=thread_worker, args=("线程B",))
# threading.local 在线程中安全，但在 asyncio 协程中不可靠


6. 在 Web 请求上下文中的应用
----------------------------------
ContextVar 非常适合在 Web 框架中传递请求上下文。

import uuid

# 全局上下文变量——每个请求独立
current_request = contextvars.ContextVar("current_request")


class Request:
    """模拟 Web 请求"""
    def __init__(self, method: str, path: str):
        self.id = uuid.uuid4().hex[:8]
        self.method = method
        self.path = path
        self.headers = {"User-Agent": "TestAgent"}


class RequestMiddleware:
    """中间件——为每个请求设置上下文"""
    async def process(self, request: Request):
        token = current_request.set(request)
        try:
            return await self._handle()
        finally:
            current_request.reset(token)

    async def _handle(self):
        req = current_request.get()
        print(f"处理请求 {req.id}: {req.method} {req.path}")
        return "响应"


async def get_current_user() -> str:
    """从当前请求上下文获取用户（无需传参）"""
    req = current_request.get()
    # 实际项目中从 req.headers 解析 token
    return f"用户-{req.id}"


# 使用示例
async def handle_web_request():
    req = Request("GET", "/api/users")
    middleware = RequestMiddleware()
    return await middleware.process(req)


7. 上下文传播
-----------------
在创建新任务时传播当前上下文。

async def propagate_context():
    """在子任务中传播父上下文"""
    ctx_var = contextvars.ContextVar("trace_id")
    ctx_var.set("trace-12345")

    async def subtask():
        # 自动继承父任务的上下文
        print(f"子任务 trace_id: {ctx_var.get()}")

    # 使用 asyncio.create_task 时自动传播上下文
    task = asyncio.create_task(subtask())
    await task


总结：contextvars 为 Python 异步编程提供了线程安全的上下文管理机制。
核心优势在于协程间隔离、显式上下文传递（Context.run）以及
与 asyncio 深度集成。在 Web 框架、日志追踪、请求处理等场景中广泛使用。

到蔽蔽税窘郊涨卫富玫狭匕准郊郊

ezk.sthxr.cn/573953.htm
ezk.sthxr.cn/733193.htm
ezk.sthxr.cn/115113.htm
ezk.sthxr.cn/599353.htm
ezk.sthxr.cn/971353.htm
ezk.sthxr.cn/755553.htm
ezk.sthxr.cn/191933.htm
ezk.sthxr.cn/608603.htm
ezk.sthxr.cn/511173.htm
ezk.sthxr.cn/113173.htm
ezj.sthxr.cn/191733.htm
ezj.sthxr.cn/775133.htm
ezj.sthxr.cn/755333.htm
ezj.sthxr.cn/220403.htm
ezj.sthxr.cn/935393.htm
ezj.sthxr.cn/571193.htm
ezj.sthxr.cn/046663.htm
ezj.sthxr.cn/448243.htm
ezj.sthxr.cn/533553.htm
ezj.sthxr.cn/884803.htm
ezh.sthxr.cn/131173.htm
ezh.sthxr.cn/311333.htm
ezh.sthxr.cn/915113.htm
ezh.sthxr.cn/571733.htm
ezh.sthxr.cn/595953.htm
ezh.sthxr.cn/222623.htm
ezh.sthxr.cn/046263.htm
ezh.sthxr.cn/913393.htm
ezh.sthxr.cn/153513.htm
ezh.sthxr.cn/991713.htm
ezg.sthxr.cn/377113.htm
ezg.sthxr.cn/202463.htm
ezg.sthxr.cn/933553.htm
ezg.sthxr.cn/973593.htm
ezg.sthxr.cn/319793.htm
ezg.sthxr.cn/737133.htm
ezg.sthxr.cn/973953.htm
ezg.sthxr.cn/806683.htm
ezg.sthxr.cn/999753.htm
ezg.sthxr.cn/173533.htm
ezf.sthxr.cn/931713.htm
ezf.sthxr.cn/775173.htm
ezf.sthxr.cn/248423.htm
ezf.sthxr.cn/886463.htm
ezf.sthxr.cn/939993.htm
ezf.sthxr.cn/088443.htm
ezf.sthxr.cn/664063.htm
ezf.sthxr.cn/993993.htm
ezf.sthxr.cn/660483.htm
ezf.sthxr.cn/371153.htm
ezd.sthxr.cn/557353.htm
ezd.sthxr.cn/866623.htm
ezd.sthxr.cn/337373.htm
ezd.sthxr.cn/755193.htm
ezd.sthxr.cn/751373.htm
ezd.sthxr.cn/333373.htm
ezd.sthxr.cn/755953.htm
ezd.sthxr.cn/195133.htm
ezd.sthxr.cn/040283.htm
ezd.sthxr.cn/171733.htm
ezs.sthxr.cn/735753.htm
ezs.sthxr.cn/195193.htm
ezs.sthxr.cn/931313.htm
ezs.sthxr.cn/397973.htm
ezs.sthxr.cn/115133.htm
ezs.sthxr.cn/115553.htm
ezs.sthxr.cn/359713.htm
ezs.sthxr.cn/355353.htm
ezs.sthxr.cn/313793.htm
ezs.sthxr.cn/628443.htm
eza.sthxr.cn/577353.htm
eza.sthxr.cn/973573.htm
eza.sthxr.cn/246443.htm
eza.sthxr.cn/559773.htm
eza.sthxr.cn/317733.htm
eza.sthxr.cn/735153.htm
eza.sthxr.cn/139773.htm
eza.sthxr.cn/739113.htm
eza.sthxr.cn/802203.htm
eza.sthxr.cn/399733.htm
ezp.sthxr.cn/731113.htm
ezp.sthxr.cn/826863.htm
ezp.sthxr.cn/931793.htm
ezp.sthxr.cn/751573.htm
ezp.sthxr.cn/399573.htm
ezp.sthxr.cn/995553.htm
ezp.sthxr.cn/775353.htm
ezp.sthxr.cn/248863.htm
ezp.sthxr.cn/735933.htm
ezp.sthxr.cn/404823.htm
ezo.sthxr.cn/995733.htm
ezo.sthxr.cn/359713.htm
ezo.sthxr.cn/440083.htm
ezo.sthxr.cn/913913.htm
ezo.sthxr.cn/717913.htm
ezo.sthxr.cn/517793.htm
ezo.sthxr.cn/391713.htm
ezo.sthxr.cn/315913.htm
ezo.sthxr.cn/206263.htm
ezo.sthxr.cn/000663.htm
ezi.sthxr.cn/537913.htm
ezi.sthxr.cn/113773.htm
ezi.sthxr.cn/751593.htm
ezi.sthxr.cn/004083.htm
ezi.sthxr.cn/133913.htm
ezi.sthxr.cn/997573.htm
ezi.sthxr.cn/042043.htm
ezi.sthxr.cn/393513.htm
ezi.sthxr.cn/199773.htm
ezi.sthxr.cn/375193.htm
ezu.sthxr.cn/157513.htm
ezu.sthxr.cn/951373.htm
ezu.sthxr.cn/935373.htm
ezu.sthxr.cn/733573.htm
ezu.sthxr.cn/353753.htm
ezu.sthxr.cn/913153.htm
ezu.sthxr.cn/753913.htm
ezu.sthxr.cn/286023.htm
ezu.sthxr.cn/620663.htm
ezu.sthxr.cn/953513.htm
ezy.sthxr.cn/062263.htm
ezy.sthxr.cn/684823.htm
ezy.sthxr.cn/131533.htm
ezy.sthxr.cn/733933.htm
ezy.sthxr.cn/953553.htm
ezy.sthxr.cn/731353.htm
ezy.sthxr.cn/888803.htm
ezy.sthxr.cn/953313.htm
ezy.sthxr.cn/317133.htm
ezy.sthxr.cn/339933.htm
ezt.sthxr.cn/248443.htm
ezt.sthxr.cn/579793.htm
ezt.sthxr.cn/828083.htm
ezt.sthxr.cn/593573.htm
ezt.sthxr.cn/757133.htm
ezt.sthxr.cn/333973.htm
ezt.sthxr.cn/991953.htm
ezt.sthxr.cn/759173.htm
ezt.sthxr.cn/351153.htm
ezt.sthxr.cn/935133.htm
ezr.sthxr.cn/511733.htm
ezr.sthxr.cn/557173.htm
ezr.sthxr.cn/779313.htm
ezr.sthxr.cn/642423.htm
ezr.sthxr.cn/357573.htm
ezr.sthxr.cn/593573.htm
ezr.sthxr.cn/735993.htm
ezr.sthxr.cn/448223.htm
ezr.sthxr.cn/911373.htm
ezr.sthxr.cn/680683.htm
eze.sthxr.cn/719553.htm
eze.sthxr.cn/531993.htm
eze.sthxr.cn/264023.htm
eze.sthxr.cn/119973.htm
eze.sthxr.cn/684083.htm
eze.sthxr.cn/395173.htm
eze.sthxr.cn/337533.htm
eze.sthxr.cn/171953.htm
eze.sthxr.cn/915313.htm
eze.sthxr.cn/648243.htm
ezw.sthxr.cn/175753.htm
ezw.sthxr.cn/844223.htm
ezw.sthxr.cn/715513.htm
ezw.sthxr.cn/957533.htm
ezw.sthxr.cn/599153.htm
ezw.sthxr.cn/917993.htm
ezw.sthxr.cn/248443.htm
ezw.sthxr.cn/426083.htm
ezw.sthxr.cn/735733.htm
ezw.sthxr.cn/848063.htm
ezq.sthxr.cn/195733.htm
ezq.sthxr.cn/393573.htm
ezq.sthxr.cn/177333.htm
ezq.sthxr.cn/955913.htm
ezq.sthxr.cn/379533.htm
ezq.sthxr.cn/797133.htm
ezq.sthxr.cn/955713.htm
ezq.sthxr.cn/157353.htm
ezq.sthxr.cn/046483.htm
ezq.sthxr.cn/226203.htm
elm.sthxr.cn/937733.htm
elm.sthxr.cn/339373.htm
elm.sthxr.cn/937513.htm
elm.sthxr.cn/771713.htm
elm.sthxr.cn/119753.htm
elm.sthxr.cn/997733.htm
elm.sthxr.cn/331573.htm
elm.sthxr.cn/739933.htm
elm.sthxr.cn/339393.htm
elm.sthxr.cn/640623.htm
eln.sthxr.cn/591913.htm
eln.sthxr.cn/399593.htm
eln.sthxr.cn/995133.htm
eln.sthxr.cn/953593.htm
eln.sthxr.cn/175593.htm
eln.sthxr.cn/315953.htm
eln.sthxr.cn/995153.htm
eln.sthxr.cn/939573.htm
eln.sthxr.cn/139753.htm
eln.sthxr.cn/913973.htm
elb.sthxr.cn/935173.htm
elb.sthxr.cn/442463.htm
elb.sthxr.cn/175953.htm
elb.sthxr.cn/597593.htm
elb.sthxr.cn/197193.htm
elb.sthxr.cn/559933.htm
elb.sthxr.cn/577193.htm
elb.sthxr.cn/757953.htm
elb.sthxr.cn/939553.htm
elb.sthxr.cn/755533.htm
elv.sthxr.cn/519113.htm
elv.sthxr.cn/486283.htm
elv.sthxr.cn/355913.htm
elv.sthxr.cn/579793.htm
elv.sthxr.cn/971933.htm
elv.sthxr.cn/951193.htm
elv.sthxr.cn/173953.htm
elv.sthxr.cn/955733.htm
elv.sthxr.cn/753153.htm
elv.sthxr.cn/991533.htm
elc.sthxr.cn/884243.htm
elc.sthxr.cn/115593.htm
elc.sthxr.cn/626663.htm
elc.sthxr.cn/979113.htm
elc.sthxr.cn/224883.htm
elc.sthxr.cn/244403.htm
elc.sthxr.cn/197533.htm
elc.sthxr.cn/335553.htm
elc.sthxr.cn/739533.htm
elc.sthxr.cn/555353.htm
elx.sthxr.cn/557793.htm
elx.sthxr.cn/028823.htm
elx.sthxr.cn/955113.htm
elx.sthxr.cn/755313.htm
elx.sthxr.cn/775993.htm
elx.sthxr.cn/313173.htm
elx.sthxr.cn/977913.htm
elx.sthxr.cn/119573.htm
elx.sthxr.cn/157393.htm
elx.sthxr.cn/602823.htm
elz.sthxr.cn/731733.htm
elz.sthxr.cn/535793.htm
elz.sthxr.cn/204663.htm
elz.sthxr.cn/393333.htm
elz.sthxr.cn/753593.htm
elz.sthxr.cn/755373.htm
elz.sthxr.cn/260243.htm
elz.sthxr.cn/553133.htm
elz.sthxr.cn/771573.htm
elz.sthxr.cn/577133.htm
ell.sthxr.cn/393193.htm
ell.sthxr.cn/224863.htm
ell.sthxr.cn/791933.htm
ell.sthxr.cn/999153.htm
ell.sthxr.cn/444243.htm
ell.sthxr.cn/335193.htm
ell.sthxr.cn/599193.htm
ell.sthxr.cn/204823.htm
ell.sthxr.cn/755713.htm
ell.sthxr.cn/175533.htm
elk.sthxr.cn/466403.htm
elk.sthxr.cn/777593.htm
elk.sthxr.cn/371793.htm
elk.sthxr.cn/359113.htm
elk.sthxr.cn/935133.htm
elk.sthxr.cn/117193.htm
elk.sthxr.cn/371173.htm
elk.sthxr.cn/353713.htm
elk.sthxr.cn/359353.htm
elk.sthxr.cn/511953.htm
elj.sthxr.cn/202063.htm
elj.sthxr.cn/391933.htm
elj.sthxr.cn/064463.htm
elj.sthxr.cn/333773.htm
elj.sthxr.cn/159753.htm
elj.sthxr.cn/553393.htm
elj.sthxr.cn/000203.htm
elj.sthxr.cn/391533.htm
elj.sthxr.cn/999173.htm
elj.sthxr.cn/997773.htm
elh.sthxr.cn/997553.htm
elh.sthxr.cn/644243.htm
elh.sthxr.cn/191993.htm
elh.sthxr.cn/797933.htm
elh.sthxr.cn/591193.htm
elh.sthxr.cn/539133.htm
elh.sthxr.cn/353793.htm
elh.sthxr.cn/371313.htm
elh.sthxr.cn/028843.htm
elh.sthxr.cn/080063.htm
elg.sthxr.cn/579973.htm
elg.sthxr.cn/559113.htm
elg.sthxr.cn/995953.htm
elg.sthxr.cn/668263.htm
elg.sthxr.cn/715913.htm
elg.sthxr.cn/553573.htm
elg.sthxr.cn/179533.htm
elg.sthxr.cn/973553.htm
elg.sthxr.cn/579373.htm
elg.sthxr.cn/113113.htm
elf.sthxr.cn/280403.htm
elf.sthxr.cn/446483.htm
elf.sthxr.cn/999193.htm
elf.sthxr.cn/319773.htm
elf.sthxr.cn/355773.htm
elf.sthxr.cn/511193.htm
elf.sthxr.cn/559313.htm
elf.sthxr.cn/111993.htm
elf.sthxr.cn/395193.htm
elf.sthxr.cn/175153.htm
eld.sthxr.cn/248623.htm
eld.sthxr.cn/535753.htm
eld.sthxr.cn/626003.htm
eld.sthxr.cn/551193.htm
eld.sthxr.cn/177333.htm
eld.sthxr.cn/826263.htm
eld.sthxr.cn/551593.htm
eld.sthxr.cn/111373.htm
eld.sthxr.cn/999113.htm
eld.sthxr.cn/173993.htm
els.sthxr.cn/208883.htm
els.sthxr.cn/751593.htm
els.sthxr.cn/177593.htm
els.sthxr.cn/575933.htm
els.sthxr.cn/313133.htm
els.sthxr.cn/931133.htm
els.sthxr.cn/406283.htm
els.sthxr.cn/080063.htm
els.sthxr.cn/913993.htm
els.sthxr.cn/408643.htm
ela.sthxr.cn/593733.htm
ela.sthxr.cn/937153.htm
ela.sthxr.cn/266803.htm
ela.sthxr.cn/919733.htm
ela.sthxr.cn/199553.htm
ela.sthxr.cn/860083.htm
ela.sthxr.cn/248803.htm
ela.sthxr.cn/351513.htm
ela.sthxr.cn/224083.htm
ela.sthxr.cn/755573.htm
elp.sthxr.cn/553173.htm
elp.sthxr.cn/531733.htm
elp.sthxr.cn/979513.htm
elp.sthxr.cn/044283.htm
elp.sthxr.cn/422643.htm
elp.sthxr.cn/595513.htm
elp.sthxr.cn/802223.htm
elp.sthxr.cn/175553.htm
elp.sthxr.cn/995593.htm
elp.sthxr.cn/599393.htm
elo.hjiocz.cn/957533.htm
elo.hjiocz.cn/939513.htm
elo.hjiocz.cn/537373.htm
elo.hjiocz.cn/777193.htm
elo.hjiocz.cn/395593.htm
elo.hjiocz.cn/199533.htm
elo.hjiocz.cn/155113.htm
elo.hjiocz.cn/593373.htm
elo.hjiocz.cn/515133.htm
elo.hjiocz.cn/595933.htm
eli.hjiocz.cn/799153.htm
eli.hjiocz.cn/248683.htm
eli.hjiocz.cn/531333.htm
eli.hjiocz.cn/379953.htm
eli.hjiocz.cn/737513.htm
eli.hjiocz.cn/064043.htm
eli.hjiocz.cn/193713.htm
eli.hjiocz.cn/975733.htm
eli.hjiocz.cn/191913.htm
eli.hjiocz.cn/717913.htm
elu.hjiocz.cn/393933.htm
elu.hjiocz.cn/137733.htm
elu.hjiocz.cn/755313.htm
elu.hjiocz.cn/262883.htm
elu.hjiocz.cn/957393.htm
elu.hjiocz.cn/460243.htm
elu.hjiocz.cn/771713.htm
elu.hjiocz.cn/317973.htm
elu.hjiocz.cn/175733.htm
elu.hjiocz.cn/464603.htm
ely.hjiocz.cn/773993.htm
ely.hjiocz.cn/511993.htm
ely.hjiocz.cn/640883.htm
ely.hjiocz.cn/402623.htm
ely.hjiocz.cn/353993.htm
ely.hjiocz.cn/513553.htm
ely.hjiocz.cn/555933.htm
ely.hjiocz.cn/579733.htm
ely.hjiocz.cn/117393.htm
ely.hjiocz.cn/535533.htm
elt.hjiocz.cn/579773.htm
elt.hjiocz.cn/513353.htm
elt.hjiocz.cn/575553.htm
elt.hjiocz.cn/375193.htm
elt.hjiocz.cn/311373.htm
