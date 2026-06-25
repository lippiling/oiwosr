Python Web 开发基础与框架对比
================================

Python 在 Web 开发领域拥有丰富的生态，从内置模块到成熟的框架，各有特点。
本文从底层协议讲起，逐步深入到主流框架的对比，帮助你理解 Web 开发的全貌。

一、WSGI 与 ASGI —— Python Web 的基石
--------------------------------------

WSGI（Web Server Gateway Interface）是 Python 传统的同步 Web 接口标准。
ASGI（Asynchronous Server Gateway Interface）是它的异步升级版。

# WSGI 应用示例：一个最简单的 Web 应用
def wsgi_app(environ, start_response):
    # environ: 包含所有 HTTP 请求信息的字典
    # start_response: 用于发送响应状态码和头的回调函数
    status = "200 OK"
    headers = [("Content-Type", "text/plain; charset=utf-8")]
    start_response(status, headers)
    # 必须返回可迭代的字节数据
    return [b"Hello, WSGI World!"]

# 使用 wsgiref 内置服务器运行
from wsgiref.simple_server import make_server
server = make_server("localhost", 8000, wsgi_app)
print("WSGI 服务器运行在 http://localhost:8000")
# server.serve_forever()  # 取消注释即可启动

# ASGI 应用示例：支持异步的 Web 接口
async def asgi_app(scope, receive, send):
    # scope: 连接信息字典，类似 WSGI 的 environ
    # receive: 用于接收消息的异步函数
    # send: 用于发送消息的异步函数
    assert scope["type"] == "http"
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain")],
    })
    await send({
        "type": "http.response.body",
        "body": b"Hello, ASGI World!",
    })

二、使用 http.server 快速搭建简易服务器
-----------------------------------------

Python 内置的 http.server 模块可用于快速搭建开发用服务器。

from http.server import HTTPServer, BaseHTTPRequestHandler
import json

# 自定义请求处理器
class SimpleHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        """处理 GET 请求"""
        if self.path == "/":
            self.send_response(200)
            self.send_header("Content-Type", "text/html; charset=utf-8")
            self.end_headers()
            self.wfile.write("<h1>欢迎来到简易服务器</h1>".encode("utf-8"))
        elif self.path == "/api/health":
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            response = {"status": "ok", "message": "服务运行中"}
            self.wfile.write(json.dumps(response).encode("utf-8"))
        else:
            self.send_response(404)
            self.end_headers()
            self.wfile.write(b"404 Not Found")

    def do_POST(self):
        """处理 POST 请求（读取请求体）"""
        content_length = int(self.headers.get("Content-Length", 0))
        body = self.rfile.read(content_length)
        # 解析 JSON 请求体
        data = json.loads(body.decode("utf-8"))
        print(f"收到数据: {data}")

        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        response = {"received": data, "echo": "success"}
        self.wfile.write(json.dumps(response).encode("utf-8"))

    def log_message(self, format, *args):
        """自定义日志输出"""
        print(f"[{self.log_date_time_string()}] {args[0]} {args[1]} {args[2]}")

server = HTTPServer(("localhost", 8080), SimpleHandler)
print("简易 HTTP 服务器已启动: http://localhost:8080")
# server.serve_forever()  # 取消注释启动服务器

三、三大 Web 框架对比
---------------------

| 特性         | Flask                  | FastAPI               | Django                  |
|-------------|------------------------|-----------------------|-------------------------|
| 类型        | 微框架（简约灵活）     | 异步高性能框架        | 全能框架（电池全包）    |
| Python 版本 | 2.7+ / 3.x            | 3.6+（强类型依赖）    | 3.x                     |
| 异步支持    | 有限（可通过插件）     | 原生异步支持          | 3.1+ 开始支持异步       |
| 性能        | 中等（同步）           | 高（基于 Starlette）  | 中等（同步为主）        |
| 路由方式    | 装饰器                 | 装饰器 + 路径参数类型 | urls.py 集中配置        |
| ORM         | 无（可集成 SQLAlchemy)| 无（可集成 SQLAlchemy)| 内置 Django ORM         |
| 序列化      | 手写或 marshmallow     | Pydantic 自动处理     | Django REST Framework   |
| 学习曲线    | 低（极易上手）         | 中（需理解类型提示）  | 高（概念较多）          |
| 适合场景    | 小项目/API 原型        | 高性能 API/微服务     | 大型项目/内容管理       |

四、路由与请求处理
-----------------

# Flask 风格路由（使用装饰器）
from flask import Flask, request, jsonify
app = Flask(__name__)

@app.route("/users/<int:user_id>", methods=["GET"])
def get_user(user_id):
    """路径参数会自动转换类型"""
    return jsonify({"user_id": user_id, "name": f"User_{user_id}"})

# FastAPI 风格路由（利用类型提示）
from fastapi import FastAPI, Path
from pydantic import BaseModel
import uvicorn

fastapi_app = FastAPI()

class Item(BaseModel):
    """Pydantic 模型自动处理请求体验证和序列化"""
    name: str
    price: float
    tags: list[str] = []

@fastapi_app.get("/items/{item_id}")
async def read_item(item_id: int = Path(..., title="项目ID")):
    """FastAPI 自动生成 API 文档，自动验证参数类型"""
    return {"item_id": item_id, "message": "商品信息"}

# 启动 FastAPI 应用: uvicorn.run(fastapi_app, host="0.0.0.0", port=8000)

五、中间件概念
-------------

中间件是请求/响应处理管道中的一个环节，可以在请求到达路由之前或响应返回
给客户端之前执行额外的处理逻辑（如日志记录、认证检查、跨域处理等）。

from flask import Flask, g
import time

app = Flask(__name__)

@app.before_request
def before_request():
    """请求到达路由前的预处理：记录开始时间"""
    g.start_time = time.time()
    # g 对象是 Flask 的请求全局上下文，可在同一请求的各个函数间共享数据

@app.after_request
def after_request(response):
    """响应发送给客户端后的后处理：计算请求耗时"""
    elapsed = time.time() - g.start_time
    print(f"[{request.method}] {request.path} - 耗时: {elapsed:.4f}s")
    # 可以统一添加响应头
    response.headers["X-Response-Time"] = str(elapsed)
    return response

# FastAPI 中间件示例
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
import time

class TimingMiddleware(BaseHTTPMiddleware):
    """自定义 FastAPI 中间件：统计每个请求的耗时"""
    async def dispatch(self, request: Request, call_next):
        start = time.time()
        response = await call_next(request)  # 调用下一个中间件或路由
        elapsed = time.time() - start
        response.headers["X-Process-Time"] = str(elapsed)
        return response

fastapi_app = FastAPI()
fastapi_app.add_middleware(TimingMiddleware)

六、请求/响应周期
-----------------

一个典型的 Web 请求生命周期：

1. 客户端发送 HTTP 请求到服务器
2. WSGI/ASGI 服务器解析请求，构建环境字典（environ/scope）
3. 中间件链依次处理请求（认证、日志、限流等）
4. 路由系统根据 URL 路径匹配对应的视图函数
5. 视图函数执行业务逻辑（数据库查询、外部 API 调用等）
6. 构建响应对象（状态码、响应头、响应体）
7. 中间件链反向处理响应
8. 服务器将响应发送回客户端

七、REST API 基础
-----------------

RESTful API 设计遵循以下原则：

# 典型的 REST API 端点设计
GET    /api/users          # 获取用户列表
POST   /api/users          # 创建新用户
GET    /api/users/{id}     # 获取单个用户
PUT    /api/users/{id}     # 更新用户（全量替换）
PATCH  /api/users/{id}     # 部分更新用户
DELETE /api/users/{id}     # 删除用户

# FastAPI 实现 REST API 示例
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class UserCreate(BaseModel):
    name: str
    email: str

# 模拟数据库
fake_db = {}
counter = 0

@app.post("/api/users")
async def create_user(user: UserCreate):
    global counter
    counter += 1
    fake_db[counter] = user.model_dump()  # Pydantic v2 使用 model_dump
    return {"id": counter, **user.model_dump()}

@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    if user_id not in fake_db:
        raise HTTPException(status_code=404, detail="用户不存在")
    return {"id": user_id, **fake_db[user_id]}

八、序列化与反序列化
-------------------

序列化是将 Python 对象转换为 JSON/XML 等传输格式的过程，反序列化则相反。

import json
from datetime import datetime

# 自定义 JSON 编码器处理 datetime 类型
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()  # datetime 转为 ISO 8601 字符串
        if isinstance(obj, bytes):
            return obj.decode("utf-8")  # bytes 转为字符串
        return super().default(obj)

data = {
    "name": "测试",
    "created_at": datetime.now(),
    "token": b"binary_data"
}
json_str = json.dumps(data, cls=CustomEncoder, ensure_ascii=False)
print(json_str)  # datetime 和 bytes 都被正确序列化

# FastAPI/Pydantic 自动处理序列化
class ResponseModel(BaseModel):
    id: int
    name: str
    created_at: datetime

# 返回 Pydantic 模型实例时，FastAPI 自动调用 model_dump() 序列化为 JSON

总结：从 WSGI/ASGI 底层协议到 Flask/FastAPI/Django 三大框架，Python Web
开发生态提供了从简单到复杂的完整选择。理解请求/响应周期、中间件机制和
REST API 设计原则，是构建高质量 Web 服务的基础。

rui.by1792.cn/00606.Doc
rui.by1792.cn/26642.Doc
rui.by1792.cn/40806.Doc
rui.by1792.cn/88066.Doc
rui.by1792.cn/44420.Doc
rui.by1792.cn/02424.Doc
rui.by1792.cn/08280.Doc
rui.by1792.cn/04624.Doc
rui.by1792.cn/08802.Doc
rui.by1792.cn/88802.Doc
ruu.by1792.cn/95531.Doc
ruu.by1792.cn/44622.Doc
ruu.by1792.cn/64042.Doc
ruu.by1792.cn/20608.Doc
ruu.by1792.cn/11319.Doc
ruu.by1792.cn/20002.Doc
ruu.by1792.cn/42284.Doc
ruu.by1792.cn/28608.Doc
ruu.by1792.cn/86004.Doc
ruu.by1792.cn/80086.Doc
ruy.by1792.cn/00268.Doc
ruy.by1792.cn/66684.Doc
ruy.by1792.cn/42820.Doc
ruy.by1792.cn/82404.Doc
ruy.by1792.cn/44680.Doc
ruy.by1792.cn/26488.Doc
ruy.by1792.cn/53971.Doc
ruy.by1792.cn/62804.Doc
ruy.by1792.cn/99373.Doc
ruy.by1792.cn/82206.Doc
rut.by1792.cn/93513.Doc
rut.by1792.cn/22480.Doc
rut.by1792.cn/42026.Doc
rut.by1792.cn/20688.Doc
rut.by1792.cn/99997.Doc
rut.by1792.cn/57753.Doc
rut.by1792.cn/46644.Doc
rut.by1792.cn/37575.Doc
rut.by1792.cn/08084.Doc
rut.by1792.cn/68642.Doc
rur.by1792.cn/84820.Doc
rur.by1792.cn/62482.Doc
rur.by1792.cn/86662.Doc
rur.by1792.cn/06026.Doc
rur.by1792.cn/24602.Doc
rur.by1792.cn/46044.Doc
rur.by1792.cn/00084.Doc
rur.by1792.cn/22666.Doc
rur.by1792.cn/33553.Doc
rur.by1792.cn/68248.Doc
rue.by1792.cn/24264.Doc
rue.by1792.cn/88242.Doc
rue.by1792.cn/37379.Doc
rue.by1792.cn/00260.Doc
rue.by1792.cn/24444.Doc
rue.by1792.cn/84242.Doc
rue.by1792.cn/66682.Doc
rue.by1792.cn/80260.Doc
rue.by1792.cn/35991.Doc
rue.by1792.cn/66822.Doc
ruw.by1792.cn/48660.Doc
ruw.by1792.cn/86826.Doc
ruw.by1792.cn/20066.Doc
ruw.by1792.cn/66664.Doc
ruw.by1792.cn/82842.Doc
ruw.by1792.cn/66880.Doc
ruw.by1792.cn/95319.Doc
ruw.by1792.cn/68064.Doc
ruw.by1792.cn/62284.Doc
ruw.by1792.cn/08264.Doc
ruq.by1792.cn/19315.Doc
ruq.by1792.cn/22604.Doc
ruq.by1792.cn/46662.Doc
ruq.by1792.cn/59553.Doc
ruq.by1792.cn/64448.Doc
ruq.by1792.cn/39577.Doc
ruq.by1792.cn/28046.Doc
ruq.by1792.cn/48268.Doc
ruq.by1792.cn/62844.Doc
ruq.by1792.cn/22088.Doc
rym.by1792.cn/00468.Doc
rym.by1792.cn/02882.Doc
rym.by1792.cn/20288.Doc
rym.by1792.cn/71733.Doc
rym.by1792.cn/04604.Doc
rym.by1792.cn/42622.Doc
rym.by1792.cn/26286.Doc
rym.by1792.cn/48288.Doc
rym.by1792.cn/26422.Doc
rym.by1792.cn/80686.Doc
ryn.by1792.cn/57759.Doc
ryn.by1792.cn/60462.Doc
ryn.by1792.cn/20644.Doc
ryn.by1792.cn/42488.Doc
ryn.by1792.cn/82062.Doc
ryn.by1792.cn/06208.Doc
ryn.by1792.cn/93351.Doc
ryn.by1792.cn/08088.Doc
ryn.by1792.cn/08686.Doc
ryn.by1792.cn/91353.Doc
ryb.by1792.cn/82022.Doc
ryb.by1792.cn/19997.Doc
ryb.by1792.cn/00684.Doc
ryb.by1792.cn/04040.Doc
ryb.by1792.cn/06866.Doc
ryb.by1792.cn/68668.Doc
ryb.by1792.cn/40044.Doc
ryb.by1792.cn/42268.Doc
ryb.by1792.cn/02462.Doc
ryb.by1792.cn/00686.Doc
ryv.by1792.cn/88028.Doc
ryv.by1792.cn/84246.Doc
ryv.by1792.cn/22842.Doc
ryv.by1792.cn/00080.Doc
ryv.by1792.cn/82062.Doc
ryv.by1792.cn/06626.Doc
ryv.by1792.cn/66888.Doc
ryv.by1792.cn/44406.Doc
ryv.by1792.cn/68628.Doc
ryv.by1792.cn/42646.Doc
ryc.by1792.cn/64026.Doc
ryc.by1792.cn/97733.Doc
ryc.by1792.cn/28048.Doc
ryc.by1792.cn/00088.Doc
ryc.by1792.cn/20680.Doc
ryc.by1792.cn/40004.Doc
ryc.by1792.cn/26268.Doc
ryc.by1792.cn/28026.Doc
ryc.by1792.cn/48488.Doc
ryc.by1792.cn/68286.Doc
ryx.by1792.cn/22480.Doc
ryx.by1792.cn/00064.Doc
ryx.by1792.cn/62042.Doc
ryx.by1792.cn/60400.Doc
ryx.by1792.cn/06882.Doc
ryx.by1792.cn/00822.Doc
ryx.by1792.cn/28800.Doc
ryx.by1792.cn/48860.Doc
ryx.by1792.cn/26208.Doc
ryx.by1792.cn/26846.Doc
ryz.by1792.cn/22262.Doc
ryz.by1792.cn/40606.Doc
ryz.by1792.cn/20228.Doc
ryz.by1792.cn/84062.Doc
ryz.by1792.cn/20800.Doc
ryz.by1792.cn/20240.Doc
ryz.by1792.cn/26402.Doc
ryz.by1792.cn/91919.Doc
ryz.by1792.cn/91997.Doc
ryz.by1792.cn/20002.Doc
ryl.by1792.cn/02288.Doc
ryl.by1792.cn/44060.Doc
ryl.by1792.cn/08266.Doc
ryl.by1792.cn/02804.Doc
ryl.by1792.cn/59979.Doc
ryl.by1792.cn/91937.Doc
ryl.by1792.cn/28460.Doc
ryl.by1792.cn/04422.Doc
ryl.by1792.cn/84626.Doc
ryl.by1792.cn/66066.Doc
ryk.by1792.cn/40002.Doc
ryk.by1792.cn/68080.Doc
ryk.by1792.cn/60004.Doc
ryk.by1792.cn/00848.Doc
ryk.by1792.cn/40468.Doc
ryk.by1792.cn/08804.Doc
ryk.by1792.cn/48620.Doc
ryk.by1792.cn/86466.Doc
ryk.by1792.cn/06620.Doc
ryk.by1792.cn/15991.Doc
ryj.by1792.cn/31771.Doc
ryj.by1792.cn/02046.Doc
ryj.by1792.cn/08028.Doc
ryj.by1792.cn/00844.Doc
ryj.by1792.cn/48866.Doc
ryj.by1792.cn/82280.Doc
ryj.by1792.cn/80446.Doc
ryj.by1792.cn/04886.Doc
ryj.by1792.cn/62404.Doc
ryj.by1792.cn/68866.Doc
ryh.by1792.cn/40080.Doc
ryh.by1792.cn/02266.Doc
ryh.by1792.cn/08440.Doc
ryh.by1792.cn/80486.Doc
ryh.by1792.cn/40440.Doc
ryh.by1792.cn/08620.Doc
ryh.by1792.cn/08824.Doc
ryh.by1792.cn/22240.Doc
ryh.by1792.cn/22402.Doc
ryh.by1792.cn/00424.Doc
ryg.by1792.cn/08840.Doc
ryg.by1792.cn/28244.Doc
ryg.by1792.cn/11539.Doc
ryg.by1792.cn/84644.Doc
ryg.by1792.cn/40888.Doc
ryg.by1792.cn/04860.Doc
ryg.by1792.cn/64204.Doc
ryg.by1792.cn/64080.Doc
ryg.by1792.cn/04044.Doc
ryg.by1792.cn/24208.Doc
ryf.by1792.cn/42004.Doc
ryf.by1792.cn/88860.Doc
ryf.by1792.cn/00486.Doc
ryf.by1792.cn/26448.Doc
ryf.by1792.cn/51759.Doc
ryf.by1792.cn/17375.Doc
ryf.by1792.cn/15975.Doc
ryf.by1792.cn/48844.Doc
ryf.by1792.cn/68862.Doc
ryf.by1792.cn/06460.Doc
ryd.by1792.cn/19999.Doc
ryd.by1792.cn/00608.Doc
ryd.by1792.cn/17991.Doc
ryd.by1792.cn/04442.Doc
ryd.by1792.cn/44082.Doc
ryd.by1792.cn/62644.Doc
ryd.by1792.cn/02862.Doc
ryd.by1792.cn/31593.Doc
ryd.by1792.cn/46268.Doc
ryd.by1792.cn/26884.Doc
rys.by1792.cn/88444.Doc
rys.by1792.cn/44886.Doc
rys.by1792.cn/84204.Doc
rys.by1792.cn/88448.Doc
rys.by1792.cn/44662.Doc
rys.by1792.cn/44828.Doc
rys.by1792.cn/62404.Doc
rys.by1792.cn/15593.Doc
rys.by1792.cn/42668.Doc
rys.by1792.cn/08466.Doc
rya.by1792.cn/88408.Doc
rya.by1792.cn/22468.Doc
rya.by1792.cn/00026.Doc
rya.by1792.cn/84020.Doc
rya.by1792.cn/04020.Doc
rya.by1792.cn/06022.Doc
rya.by1792.cn/44244.Doc
rya.by1792.cn/08266.Doc
rya.by1792.cn/88482.Doc
rya.by1792.cn/80228.Doc
ryp.by1792.cn/86444.Doc
ryp.by1792.cn/86640.Doc
ryp.by1792.cn/26220.Doc
ryp.by1792.cn/40406.Doc
ryp.by1792.cn/80440.Doc
ryp.by1792.cn/80680.Doc
ryp.by1792.cn/46402.Doc
ryp.by1792.cn/00466.Doc
ryp.by1792.cn/68466.Doc
ryp.by1792.cn/68822.Doc
ryo.by1792.cn/62684.Doc
ryo.by1792.cn/44462.Doc
ryo.by1792.cn/82064.Doc
ryo.by1792.cn/13355.Doc
ryo.by1792.cn/86404.Doc
ryo.by1792.cn/46042.Doc
ryo.by1792.cn/60626.Doc
ryo.by1792.cn/40080.Doc
ryo.by1792.cn/20068.Doc
ryo.by1792.cn/40620.Doc
ryi.by1792.cn/06860.Doc
ryi.by1792.cn/00628.Doc
ryi.by1792.cn/57135.Doc
ryi.by1792.cn/20006.Doc
ryi.by1792.cn/82846.Doc
ryi.by1792.cn/24004.Doc
ryi.by1792.cn/84648.Doc
ryi.by1792.cn/80888.Doc
ryi.by1792.cn/64286.Doc
ryi.by1792.cn/44422.Doc
ryu.by1792.cn/04024.Doc
ryu.by1792.cn/88440.Doc
ryu.by1792.cn/82486.Doc
ryu.by1792.cn/15393.Doc
ryu.by1792.cn/86028.Doc
ryu.by1792.cn/80282.Doc
ryu.by1792.cn/88462.Doc
ryu.by1792.cn/60800.Doc
ryu.by1792.cn/46462.Doc
ryu.by1792.cn/80026.Doc
ryy.by1792.cn/88026.Doc
ryy.by1792.cn/86042.Doc
ryy.by1792.cn/20224.Doc
ryy.by1792.cn/42868.Doc
ryy.by1792.cn/20046.Doc
ryy.by1792.cn/55339.Doc
ryy.by1792.cn/80802.Doc
ryy.by1792.cn/22860.Doc
ryy.by1792.cn/75937.Doc
ryy.by1792.cn/00488.Doc
ryt.by1792.cn/35795.Doc
ryt.by1792.cn/88800.Doc
ryt.by1792.cn/42004.Doc
ryt.by1792.cn/60860.Doc
ryt.by1792.cn/24220.Doc
ryt.by1792.cn/17391.Doc
ryt.by1792.cn/26488.Doc
ryt.by1792.cn/88680.Doc
ryt.by1792.cn/46806.Doc
ryt.by1792.cn/48800.Doc
ryr.by1792.cn/06824.Doc
ryr.by1792.cn/20248.Doc
ryr.by1792.cn/62684.Doc
ryr.by1792.cn/48620.Doc
ryr.by1792.cn/06848.Doc
ryr.by1792.cn/59937.Doc
ryr.by1792.cn/06486.Doc
ryr.by1792.cn/08664.Doc
ryr.by1792.cn/71959.Doc
ryr.by1792.cn/20886.Doc
rye.by1792.cn/64668.Doc
rye.by1792.cn/28868.Doc
rye.by1792.cn/24024.Doc
rye.by1792.cn/82400.Doc
rye.by1792.cn/44244.Doc
rye.by1792.cn/06060.Doc
rye.by1792.cn/04488.Doc
rye.by1792.cn/42402.Doc
rye.by1792.cn/46268.Doc
rye.by1792.cn/22286.Doc
ryw.by1792.cn/82684.Doc
ryw.by1792.cn/44666.Doc
ryw.by1792.cn/64642.Doc
ryw.by1792.cn/08866.Doc
ryw.by1792.cn/46800.Doc
ryw.by1792.cn/31915.Doc
ryw.by1792.cn/80682.Doc
ryw.by1792.cn/04040.Doc
ryw.by1792.cn/24080.Doc
ryw.by1792.cn/68868.Doc
ryq.by1792.cn/62602.Doc
ryq.by1792.cn/02686.Doc
ryq.by1792.cn/60262.Doc
ryq.by1792.cn/24484.Doc
ryq.by1792.cn/20428.Doc
ryq.by1792.cn/68220.Doc
ryq.by1792.cn/86660.Doc
ryq.by1792.cn/66088.Doc
ryq.by1792.cn/64240.Doc
ryq.by1792.cn/20284.Doc
rtm.by1792.cn/02048.Doc
rtm.by1792.cn/26044.Doc
rtm.by1792.cn/00880.Doc
rtm.by1792.cn/20062.Doc
rtm.by1792.cn/64408.Doc
rtm.by1792.cn/86204.Doc
rtm.by1792.cn/60488.Doc
rtm.by1792.cn/66644.Doc
rtm.by1792.cn/24824.Doc
rtm.by1792.cn/82088.Doc
rtn.by1792.cn/40286.Doc
rtn.by1792.cn/20022.Doc
rtn.by1792.cn/86226.Doc
rtn.by1792.cn/24282.Doc
rtn.by1792.cn/97757.Doc
rtn.by1792.cn/44822.Doc
rtn.by1792.cn/20440.Doc
rtn.by1792.cn/86048.Doc
rtn.by1792.cn/46620.Doc
rtn.by1792.cn/1.Doc
rtb.by1792.cn/28626.Doc
rtb.by1792.cn/88242.Doc
rtb.by1792.cn/04228.Doc
rtb.by1792.cn/13713.Doc
rtb.by1792.cn/60888.Doc
rtb.by1792.cn/64060.Doc
rtb.by1792.cn/11939.Doc
rtb.by1792.cn/75595.Doc
rtb.by1792.cn/06822.Doc
rtb.by1792.cn/66402.Doc
rtv.by1792.cn/68622.Doc
rtv.by1792.cn/40242.Doc
rtv.by1792.cn/04844.Doc
rtv.by1792.cn/28006.Doc
rtv.by1792.cn/24806.Doc
rtv.by1792.cn/86280.Doc
rtv.by1792.cn/24640.Doc
rtv.by1792.cn/20848.Doc
rtv.by1792.cn/48602.Doc
rtv.by1792.cn/48808.Doc
rtc.by1792.cn/62800.Doc
rtc.by1792.cn/86624.Doc
rtc.by1792.cn/22246.Doc
rtc.by1792.cn/73955.Doc
rtc.by1792.cn/64264.Doc
rtc.by1792.cn/48680.Doc
rtc.by1792.cn/20664.Doc
rtc.by1792.cn/26086.Doc
rtc.by1792.cn/64860.Doc
rtc.by1792.cn/26682.Doc
rtx.by1792.cn/46226.Doc
rtx.by1792.cn/02820.Doc
rtx.by1792.cn/08804.Doc
rtx.by1792.cn/40064.Doc
rtx.by1792.cn/04620.Doc
rtx.by1792.cn/00022.Doc
rtx.by1792.cn/39351.Doc
rtx.by1792.cn/13195.Doc
rtx.by1792.cn/66040.Doc
rtx.by1792.cn/82044.Doc
rtz.by1792.cn/00262.Doc
rtz.by1792.cn/64664.Doc
rtz.by1792.cn/86066.Doc
rtz.by1792.cn/39735.Doc
rtz.by1792.cn/22642.Doc
rtz.by1792.cn/26262.Doc
rtz.by1792.cn/46442.Doc
rtz.by1792.cn/60620.Doc
rtz.by1792.cn/06840.Doc
rtz.by1792.cn/68468.Doc
rtl.by1792.cn/31911.Doc
rtl.by1792.cn/86462.Doc
rtl.by1792.cn/04448.Doc
rtl.by1792.cn/00882.Doc
rtl.by1792.cn/62440.Doc
rtl.by1792.cn/24048.Doc
rtl.by1792.cn/86044.Doc
rtl.by1792.cn/40248.Doc
rtl.by1792.cn/20080.Doc
rtl.by1792.cn/26020.Doc
rtk.by1792.cn/04684.Doc
rtk.by1792.cn/77513.Doc
rtk.by1792.cn/60468.Doc
rtk.by1792.cn/44666.Doc
rtk.by1792.cn/82026.Doc
rtk.by1792.cn/80404.Doc
rtk.by1792.cn/20248.Doc
rtk.by1792.cn/17737.Doc
rtk.by1792.cn/02664.Doc
rtk.by1792.cn/80886.Doc
rtj.by1792.cn/22620.Doc
rtj.by1792.cn/40622.Doc
rtj.by1792.cn/68286.Doc
rtj.by1792.cn/44846.Doc
rtj.by1792.cn/35799.Doc
rtj.by1792.cn/44668.Doc
rtj.by1792.cn/62644.Doc
rtj.by1792.cn/75913.Doc
rtj.by1792.cn/68882.Doc
rtj.by1792.cn/55775.Doc
rth.by1792.cn/66488.Doc
rth.by1792.cn/66460.Doc
rth.by1792.cn/46886.Doc
rth.by1792.cn/00226.Doc
rth.by1792.cn/64440.Doc
rth.by1792.cn/60860.Doc
rth.by1792.cn/51357.Doc
rth.by1792.cn/28846.Doc
rth.by1792.cn/60882.Doc
rth.by1792.cn/40400.Doc
rtg.by1792.cn/82822.Doc
rtg.by1792.cn/02482.Doc
rtg.by1792.cn/26024.Doc
rtg.by1792.cn/22822.Doc
rtg.by1792.cn/48844.Doc
rtg.by1792.cn/60602.Doc
rtg.by1792.cn/73793.Doc
rtg.by1792.cn/68842.Doc
rtg.by1792.cn/00486.Doc
rtg.by1792.cn/55135.Doc
rtf.by1792.cn/64044.Doc
rtf.by1792.cn/42404.Doc
rtf.by1792.cn/22042.Doc
rtf.by1792.cn/44880.Doc
rtf.by1792.cn/11595.Doc
rtf.by1792.cn/02600.Doc
rtf.by1792.cn/82680.Doc
rtf.by1792.cn/62808.Doc
rtf.by1792.cn/86644.Doc
rtf.by1792.cn/40404.Doc
rtd.by1792.cn/60648.Doc
rtd.by1792.cn/44220.Doc
rtd.by1792.cn/28660.Doc
rtd.by1792.cn/04622.Doc
rtd.by1792.cn/84088.Doc
rtd.by1792.cn/42088.Doc
rtd.by1792.cn/26004.Doc
rtd.by1792.cn/66448.Doc
rtd.by1792.cn/62446.Doc
rtd.by1792.cn/86862.Doc
rts.by1792.cn/02642.Doc
rts.by1792.cn/20844.Doc
rts.by1792.cn/28268.Doc
rts.by1792.cn/15159.Doc
rts.by1792.cn/82848.Doc
rts.by1792.cn/68282.Doc
rts.by1792.cn/51397.Doc
rts.by1792.cn/06224.Doc
rts.by1792.cn/68804.Doc
rts.by1792.cn/26824.Doc
