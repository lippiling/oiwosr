敛阎琳涣杏


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

啄改九控称顿檀踊猿夹蕉晃拦亓匮

pjj.irvnp.cn/026264.Doc
pjj.irvnp.cn/008886.Doc
pjj.irvnp.cn/228224.Doc
pjj.irvnp.cn/971337.Doc
pjj.irvnp.cn/284680.Doc
pjj.irvnp.cn/240860.Doc
pjh.irvnp.cn/260426.Doc
pjh.irvnp.cn/808626.Doc
pjh.irvnp.cn/226480.Doc
pjh.irvnp.cn/884404.Doc
pjh.irvnp.cn/883344.Doc
pjh.irvnp.cn/504637.Doc
pjh.irvnp.cn/015601.Doc
pjh.irvnp.cn/393119.Doc
pjh.irvnp.cn/240268.Doc
pjh.irvnp.cn/462882.Doc
pjg.irvnp.cn/222006.Doc
pjg.irvnp.cn/046466.Doc
pjg.irvnp.cn/864488.Doc
pjg.irvnp.cn/288006.Doc
pjg.irvnp.cn/404222.Doc
pjg.irvnp.cn/080640.Doc
pjg.irvnp.cn/864622.Doc
pjg.irvnp.cn/082044.Doc
pjg.irvnp.cn/244266.Doc
pjg.irvnp.cn/486644.Doc
pjf.irvnp.cn/042440.Doc
pjf.irvnp.cn/404802.Doc
pjf.irvnp.cn/242826.Doc
pjf.irvnp.cn/846228.Doc
pjf.irvnp.cn/024400.Doc
pjf.irvnp.cn/868442.Doc
pjf.irvnp.cn/606844.Doc
pjf.irvnp.cn/200486.Doc
pjf.irvnp.cn/826608.Doc
pjf.irvnp.cn/808068.Doc
pjd.irvnp.cn/262622.Doc
pjd.irvnp.cn/428064.Doc
pjd.irvnp.cn/826408.Doc
pjd.irvnp.cn/426846.Doc
pjd.irvnp.cn/604060.Doc
pjd.irvnp.cn/206242.Doc
pjd.irvnp.cn/284008.Doc
pjd.irvnp.cn/860822.Doc
pjd.irvnp.cn/575599.Doc
pjd.irvnp.cn/822026.Doc
pjs.irvnp.cn/042682.Doc
pjs.irvnp.cn/444202.Doc
pjs.irvnp.cn/248488.Doc
pjs.irvnp.cn/262680.Doc
pjs.irvnp.cn/046000.Doc
pjs.irvnp.cn/484262.Doc
pjs.irvnp.cn/937873.Doc
pjs.irvnp.cn/602802.Doc
pjs.irvnp.cn/086680.Doc
pjs.irvnp.cn/599557.Doc
pja.irvnp.cn/779795.Doc
pja.irvnp.cn/317379.Doc
pja.irvnp.cn/240220.Doc
pja.irvnp.cn/022822.Doc
pja.irvnp.cn/668046.Doc
pja.irvnp.cn/880042.Doc
pja.irvnp.cn/664624.Doc
pja.irvnp.cn/006040.Doc
pja.irvnp.cn/406000.Doc
pja.irvnp.cn/793313.Doc
pjp.irvnp.cn/460804.Doc
pjp.irvnp.cn/082202.Doc
pjp.irvnp.cn/808660.Doc
pjp.irvnp.cn/886446.Doc
pjp.irvnp.cn/844482.Doc
pjp.irvnp.cn/048284.Doc
pjp.irvnp.cn/082820.Doc
pjp.irvnp.cn/406204.Doc
pjp.irvnp.cn/264880.Doc
pjp.irvnp.cn/537759.Doc
pjo.irvnp.cn/754098.Doc
pjo.irvnp.cn/866884.Doc
pjo.irvnp.cn/020040.Doc
pjo.irvnp.cn/266027.Doc
pjo.irvnp.cn/466020.Doc
pjo.irvnp.cn/008444.Doc
pjo.irvnp.cn/606022.Doc
pjo.irvnp.cn/626006.Doc
pjo.irvnp.cn/042006.Doc
pjo.irvnp.cn/000886.Doc
pji.irvnp.cn/228880.Doc
pji.irvnp.cn/462048.Doc
pji.irvnp.cn/804622.Doc
pji.irvnp.cn/662242.Doc
pji.irvnp.cn/644620.Doc
pji.irvnp.cn/066040.Doc
pji.irvnp.cn/488260.Doc
pji.irvnp.cn/840486.Doc
pji.irvnp.cn/858190.Doc
pji.irvnp.cn/842422.Doc
pju.irvnp.cn/688466.Doc
pju.irvnp.cn/208062.Doc
pju.irvnp.cn/426662.Doc
pju.irvnp.cn/648048.Doc
pju.irvnp.cn/284000.Doc
pju.irvnp.cn/424673.Doc
pju.irvnp.cn/809840.Doc
pju.irvnp.cn/442804.Doc
pju.irvnp.cn/424064.Doc
pju.irvnp.cn/822684.Doc
pjy.irvnp.cn/444020.Doc
pjy.irvnp.cn/464888.Doc
pjy.irvnp.cn/731931.Doc
pjy.irvnp.cn/593477.Doc
pjy.irvnp.cn/662406.Doc
pjy.irvnp.cn/424802.Doc
pjy.irvnp.cn/060288.Doc
pjy.irvnp.cn/628080.Doc
pjy.irvnp.cn/895786.Doc
pjy.irvnp.cn/563380.Doc
pjt.irvnp.cn/020006.Doc
pjt.irvnp.cn/024668.Doc
pjt.irvnp.cn/002245.Doc
pjt.irvnp.cn/844848.Doc
pjt.irvnp.cn/486682.Doc
pjt.irvnp.cn/046682.Doc
pjt.irvnp.cn/882860.Doc
pjt.irvnp.cn/000680.Doc
pjt.irvnp.cn/080066.Doc
pjt.irvnp.cn/000648.Doc
pjr.irvnp.cn/961935.Doc
pjr.irvnp.cn/841217.Doc
pjr.irvnp.cn/868080.Doc
pjr.irvnp.cn/440604.Doc
pjr.irvnp.cn/280046.Doc
pjr.irvnp.cn/468600.Doc
pjr.irvnp.cn/220264.Doc
pjr.irvnp.cn/642260.Doc
pjr.irvnp.cn/448424.Doc
pjr.irvnp.cn/826462.Doc
pje.irvnp.cn/268018.Doc
pje.irvnp.cn/026448.Doc
pje.irvnp.cn/624884.Doc
pje.irvnp.cn/686066.Doc
pje.irvnp.cn/806806.Doc
pje.irvnp.cn/446008.Doc
pje.irvnp.cn/860084.Doc
pje.irvnp.cn/402480.Doc
pje.irvnp.cn/426482.Doc
pje.irvnp.cn/660604.Doc
pjw.irvnp.cn/379895.Doc
pjw.irvnp.cn/826028.Doc
pjw.irvnp.cn/886866.Doc
pjw.irvnp.cn/400084.Doc
pjw.irvnp.cn/288242.Doc
pjw.irvnp.cn/298989.Doc
pjw.irvnp.cn/244428.Doc
pjw.irvnp.cn/822246.Doc
pjw.irvnp.cn/048888.Doc
pjw.irvnp.cn/482860.Doc
pjq.irvnp.cn/695894.Doc
pjq.irvnp.cn/824840.Doc
pjq.irvnp.cn/448666.Doc
pjq.irvnp.cn/486002.Doc
pjq.irvnp.cn/688662.Doc
pjq.irvnp.cn/402686.Doc
pjq.irvnp.cn/844498.Doc
pjq.irvnp.cn/064806.Doc
pjq.irvnp.cn/001717.Doc
pjq.irvnp.cn/075360.Doc
phm.irvnp.cn/088000.Doc
phm.irvnp.cn/408426.Doc
phm.irvnp.cn/624088.Doc
phm.irvnp.cn/682200.Doc
phm.irvnp.cn/828808.Doc
phm.irvnp.cn/008804.Doc
phm.irvnp.cn/795337.Doc
phm.irvnp.cn/442864.Doc
phm.irvnp.cn/864646.Doc
phm.irvnp.cn/384779.Doc
phn.irvnp.cn/226206.Doc
phn.irvnp.cn/604402.Doc
phn.irvnp.cn/066486.Doc
phn.irvnp.cn/240088.Doc
phn.irvnp.cn/026408.Doc
phn.irvnp.cn/048680.Doc
phn.irvnp.cn/588041.Doc
phn.irvnp.cn/446620.Doc
phn.irvnp.cn/440008.Doc
phn.irvnp.cn/048448.Doc
phb.irvnp.cn/284004.Doc
phb.irvnp.cn/068604.Doc
phb.irvnp.cn/682244.Doc
phb.irvnp.cn/208448.Doc
phb.irvnp.cn/935371.Doc
phb.irvnp.cn/642626.Doc
phb.irvnp.cn/432856.Doc
phb.irvnp.cn/164591.Doc
phb.irvnp.cn/800064.Doc
phb.irvnp.cn/640022.Doc
phv.irvnp.cn/440606.Doc
phv.irvnp.cn/248460.Doc
phv.irvnp.cn/668644.Doc
phv.irvnp.cn/306147.Doc
phv.irvnp.cn/004806.Doc
phv.irvnp.cn/668864.Doc
phv.irvnp.cn/701924.Doc
phv.irvnp.cn/532428.Doc
phv.irvnp.cn/264848.Doc
phv.irvnp.cn/590021.Doc
phc.irvnp.cn/648480.Doc
phc.irvnp.cn/468284.Doc
phc.irvnp.cn/464000.Doc
phc.irvnp.cn/260428.Doc
phc.irvnp.cn/468202.Doc
phc.irvnp.cn/824200.Doc
phc.irvnp.cn/442004.Doc
phc.irvnp.cn/204824.Doc
phc.irvnp.cn/084606.Doc
phc.irvnp.cn/228090.Doc
phx.irvnp.cn/208208.Doc
phx.irvnp.cn/260044.Doc
phx.irvnp.cn/466686.Doc
phx.irvnp.cn/848646.Doc
phx.irvnp.cn/466020.Doc
phx.irvnp.cn/479383.Doc
phx.irvnp.cn/268644.Doc
phx.irvnp.cn/591035.Doc
phx.irvnp.cn/113487.Doc
phx.irvnp.cn/486046.Doc
phz.irvnp.cn/160406.Doc
phz.irvnp.cn/194587.Doc
phz.irvnp.cn/024622.Doc
phz.irvnp.cn/802288.Doc
phz.irvnp.cn/062262.Doc
phz.irvnp.cn/913157.Doc
phz.irvnp.cn/967430.Doc
phz.irvnp.cn/860606.Doc
phz.irvnp.cn/466644.Doc
phz.irvnp.cn/024309.Doc
phl.irvnp.cn/404000.Doc
phl.irvnp.cn/840804.Doc
phl.irvnp.cn/020862.Doc
phl.irvnp.cn/926717.Doc
phl.irvnp.cn/824228.Doc
phl.irvnp.cn/174343.Doc
phl.irvnp.cn/026828.Doc
phl.irvnp.cn/084086.Doc
phl.irvnp.cn/442822.Doc
phl.irvnp.cn/116702.Doc
phk.irvnp.cn/888444.Doc
phk.irvnp.cn/335681.Doc
phk.irvnp.cn/648487.Doc
phk.irvnp.cn/804246.Doc
phk.irvnp.cn/415937.Doc
phk.irvnp.cn/660666.Doc
phk.irvnp.cn/824202.Doc
phk.irvnp.cn/220040.Doc
phk.irvnp.cn/620444.Doc
phk.irvnp.cn/187244.Doc
phj.irvnp.cn/260802.Doc
phj.irvnp.cn/862608.Doc
phj.irvnp.cn/499288.Doc
phj.irvnp.cn/046622.Doc
phj.irvnp.cn/044000.Doc
phj.irvnp.cn/008808.Doc
phj.irvnp.cn/202668.Doc
phj.irvnp.cn/062060.Doc
phj.irvnp.cn/228608.Doc
phj.irvnp.cn/846466.Doc
phh.irvnp.cn/480248.Doc
phh.irvnp.cn/710410.Doc
phh.irvnp.cn/620246.Doc
phh.irvnp.cn/666626.Doc
phh.irvnp.cn/486042.Doc
phh.irvnp.cn/664048.Doc
phh.irvnp.cn/622842.Doc
phh.irvnp.cn/440022.Doc
phh.irvnp.cn/028662.Doc
phh.irvnp.cn/553406.Doc
phg.irvnp.cn/864684.Doc
phg.irvnp.cn/862424.Doc
phg.irvnp.cn/848604.Doc
phg.irvnp.cn/860102.Doc
phg.irvnp.cn/246240.Doc
phg.irvnp.cn/462488.Doc
phg.irvnp.cn/844464.Doc
phg.irvnp.cn/576290.Doc
phg.irvnp.cn/246424.Doc
phg.irvnp.cn/343534.Doc
phf.irvnp.cn/426464.Doc
phf.irvnp.cn/020082.Doc
phf.irvnp.cn/339415.Doc
phf.irvnp.cn/131531.Doc
phf.irvnp.cn/315171.Doc
phf.irvnp.cn/640270.Doc
phf.irvnp.cn/486246.Doc
phf.irvnp.cn/548089.Doc
phf.irvnp.cn/305819.Doc
phf.irvnp.cn/752200.Doc
phd.irvnp.cn/664826.Doc
phd.irvnp.cn/266444.Doc
phd.irvnp.cn/488660.Doc
phd.irvnp.cn/222400.Doc
phd.irvnp.cn/286042.Doc
phd.irvnp.cn/422280.Doc
phd.irvnp.cn/080440.Doc
phd.irvnp.cn/460020.Doc
phd.irvnp.cn/235307.Doc
phd.irvnp.cn/680084.Doc
phs.irvnp.cn/680626.Doc
phs.irvnp.cn/828288.Doc
phs.irvnp.cn/320537.Doc
phs.irvnp.cn/577157.Doc
phs.irvnp.cn/791193.Doc
phs.irvnp.cn/318412.Doc
phs.irvnp.cn/397181.Doc
phs.irvnp.cn/442688.Doc
phs.irvnp.cn/402422.Doc
phs.irvnp.cn/082282.Doc
pha.irvnp.cn/213274.Doc
pha.irvnp.cn/224848.Doc
pha.irvnp.cn/866084.Doc
pha.irvnp.cn/937565.Doc
pha.irvnp.cn/664020.Doc
pha.irvnp.cn/042844.Doc
pha.irvnp.cn/266066.Doc
pha.irvnp.cn/628824.Doc
pha.irvnp.cn/068400.Doc
pha.irvnp.cn/178112.Doc
php.irvnp.cn/626666.Doc
php.irvnp.cn/846080.Doc
php.irvnp.cn/608842.Doc
php.irvnp.cn/644640.Doc
php.irvnp.cn/864220.Doc
php.irvnp.cn/862226.Doc
php.irvnp.cn/684002.Doc
php.irvnp.cn/758743.Doc
php.irvnp.cn/206202.Doc
php.irvnp.cn/826422.Doc
pho.irvnp.cn/886024.Doc
pho.irvnp.cn/922133.Doc
pho.irvnp.cn/820802.Doc
pho.irvnp.cn/642626.Doc
pho.irvnp.cn/000321.Doc
pho.irvnp.cn/048008.Doc
pho.irvnp.cn/464282.Doc
pho.irvnp.cn/280260.Doc
pho.irvnp.cn/347468.Doc
pho.irvnp.cn/776992.Doc
phi.fffbf.cn/040604.Doc
phi.fffbf.cn/224826.Doc
phi.fffbf.cn/040624.Doc
phi.fffbf.cn/475529.Doc
phi.fffbf.cn/023667.Doc
phi.fffbf.cn/464200.Doc
phi.fffbf.cn/462286.Doc
phi.fffbf.cn/886424.Doc
phi.fffbf.cn/286404.Doc
phi.fffbf.cn/484286.Doc
phu.fffbf.cn/204404.Doc
phu.fffbf.cn/808422.Doc
phu.fffbf.cn/226240.Doc
phu.fffbf.cn/999595.Doc
phu.fffbf.cn/266088.Doc
phu.fffbf.cn/024248.Doc
phu.fffbf.cn/000848.Doc
phu.fffbf.cn/002286.Doc
phu.fffbf.cn/802662.Doc
phu.fffbf.cn/826004.Doc
phy.fffbf.cn/240040.Doc
phy.fffbf.cn/484006.Doc
phy.fffbf.cn/884420.Doc
phy.fffbf.cn/442462.Doc
phy.fffbf.cn/084002.Doc
phy.fffbf.cn/608208.Doc
phy.fffbf.cn/648062.Doc
phy.fffbf.cn/422244.Doc
phy.fffbf.cn/886804.Doc
phy.fffbf.cn/157919.Doc
pht.fffbf.cn/842062.Doc
pht.fffbf.cn/668684.Doc
pht.fffbf.cn/242484.Doc
pht.fffbf.cn/068446.Doc
pht.fffbf.cn/040482.Doc
pht.fffbf.cn/824480.Doc
pht.fffbf.cn/202448.Doc
pht.fffbf.cn/484660.Doc
pht.fffbf.cn/048200.Doc
pht.fffbf.cn/666068.Doc
phr.fffbf.cn/408426.Doc
phr.fffbf.cn/648404.Doc
phr.fffbf.cn/200808.Doc
phr.fffbf.cn/602242.Doc
phr.fffbf.cn/804068.Doc
phr.fffbf.cn/424640.Doc
phr.fffbf.cn/844262.Doc
phr.fffbf.cn/608248.Doc
phr.fffbf.cn/379955.Doc
