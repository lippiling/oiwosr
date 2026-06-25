========================================
pytest 固件（Fixture）高级用法详解
========================================

一、Fixture 基础与作用域
------------------------

pytest 的 fixture 是测试前准备和清理工作的核心机制。
通过 @pytest.fixture 装饰器定义，scope 参数控制生命周期。

二、代码示例
------------

import pytest
import os
import tempfile


# 1. 不同作用域的 fixture
# ------------------------
# function 级别：每个测试函数都执行一次
@pytest.fixture(scope="function")
def func_fixture():
    """每个测试函数都会获得一个新的实例"""
    data = {"key": "value", "count": 0}
    print("\n[function fixture] 创建数据")
    return data


# class 级别：每个测试类执行一次
@pytest.fixture(scope="class")
def class_fixture():
    """同一个测试类中的所有方法共享此 fixture"""
    print("\n[class fixture] 初始化类级资源")
    yield {"shared": "class_data"}
    print("\n[class fixture] 清理类级资源")


# module 级别：每个模块执行一次
@pytest.fixture(scope="module")
def module_fixture():
    """整个模块共享此 fixture"""
    print("\n[module fixture] 初始化模块级资源")
    yield {"module_config": "production"}
    print("\n[module fixture] 清理模块级资源")


# session 级别：整个测试会话执行一次
@pytest.fixture(scope="session")
def session_fixture():
    """所有测试共享，通常用于数据库连接等重量级资源"""
    print("\n[session fixture] 初始化会话级资源（如数据库连接池）")
    yield {"db_url": "postgresql://localhost/test"}
    print("\n[session fixture] 清理会话级资源")


# 2. autouse=True：自动应用的 fixture
# ------------------------------------
@pytest.fixture(autouse=True)
def auto_timer():
    """自动为每个测试计时，无需显式引用"""
    import time
    start = time.time()
    yield
    elapsed = time.time() - start
    print(f"\n[autouse] 测试耗时: {elapsed:.4f} 秒")


# 3. yield fixture：setup + teardown 模式
# -----------------------------------------
@pytest.fixture
def temp_file():
    """使用 yield 实现 setup/teardown 模式"""
    # ---- setup 阶段 ----
    tmp = tempfile.NamedTemporaryFile(delete=False, suffix=".txt")
    tmp.write(b"hello fixture")
    tmp.close()
    print(f"\n[yield fixture] 创建临时文件: {tmp.name}")

    # 将资源交给测试函数使用
    yield tmp.name

    # ---- teardown 阶段 ----
    os.unlink(tmp.name)
    print(f"\n[yield fixture] 删除临时文件: {tmp.name}")


# 4. 使用 conftest.py 共享 fixture
# ---------------------------------
# 本文件同级目录下的 conftest.py 中定义的 fixture
# 会自动被所有测试找到并使用，无需 import
# 示例：conftest.py 内容如下（实际测试中写在 conftest.py 中）
"""
# conftest.py
@pytest.fixture
def db_connection():
    \"\"\"提供数据库连接，供多个测试文件共享\"\"\"
    conn = create_database_connection()
    yield conn
    conn.close()
"""


# 5. Fixture 参数化
# ------------------
@pytest.fixture(params=["mysql", "postgresql", "sqlite"])
def database_backend(request):
    """通过 params 参数化 fixture，每个参数值执行一次测试"""
    backend = request.param
    print(f"\n[参数化 fixture] 使用数据库后端: {backend}")
    return backend


# 6. 使用 request 对象获取上下文信息
# ------------------------------------
@pytest.fixture
def config(request):
    """request 对象提供测试函数的名称、类名、模块等上下文信息"""
    test_name = request.node.name
    test_module = request.module.__name__
    test_class = request.cls  # 如果测试在类中则返回类

    print(f"\n[request 对象] 当前测试: {test_name}")
    print(f"[request 对象] 所属模块: {test_module}")

    # 根据调用者返回不同配置
    if "integration" in test_name:
        return {"mode": "integration", "timeout": 30}
    return {"mode": "unit", "timeout": 5}


# 7. 内置 fixture 使用示例
# --------------------------

def test_tempdir(tmpdir):
    """tmpdir 提供临时目录，测试结束后自动清理"""
    test_file = tmpdir.join("test.txt")
    test_file.write("临时数据")
    content = test_file.read()
    assert content == "临时数据"
    print(f"\n[tmpdir] 临时目录路径: {tmpdir}")


def test_capsys(capsys):
    """capsys 捕获 stdout/stderr 输出"""
    print("标准输出内容")
    import sys
    sys.stderr.write("错误输出内容\n")

    captured = capsys.readouterr()
    assert captured.out == "标准输出内容\n"
    assert captured.err == "错误输出内容\n"
    print("[capsys] 成功捕获标准输出和错误输出")


def test_monkeypatch(monkeypatch):
    """monkeypatch 动态修改对象、环境变量等"""
    # 模拟环境变量
    monkeypatch.setenv("DATABASE_URL", "sqlite:///test.db")
    assert os.environ["DATABASE_URL"] == "sqlite:///test.db"

    # 模拟函数返回值
    def mock_get_data():
        return {"id": 1, "name": "mock_data"}

    monkeypatch.setattr("builtins.input", lambda x: "模拟输入")
    print(f"\n[monkeypatch] 环境变量: {os.environ['DATABASE_URL']}")


# 8. 组合使用 fixture
# --------------------
def test_combined_fixtures(func_fixture, temp_file, config):
    """测试可以同时注入多个 fixture"""
    print(f"\n[组合] 数据: {func_fixture}")
    print(f"[组合] 文件: {temp_file}")
    print(f"[组合] 配置: {config}")
    assert func_fixture["key"] == "value"


def test_backends(database_backend):
    """此测试会对每个数据库后端分别执行一次"""
    print(f"\n测试后端: {database_backend}")


# 9. Fixture 的返回值和作用域缓存
# --------------------------------
@pytest.fixture(scope="session")
def cached_data():
    """session 级的 fixture 结果会被缓存，提升性能"""
    print("\n[cached] 计算大量数据（仅一次）")
    return [i * i for i in range(1000)]


def test_use_cache(cached_data):
    """使用 session 级缓存数据"""
    assert len(cached_data) == 1000
    assert cached_data[10] == 100


# 10. 使用 fixture 进行依赖注入
# ------------------------------
@pytest.fixture
def user_service(config):
    """根据配置创建不同的服务实例"""
    class UserService:
        def __init__(self, mode, timeout):
            self.mode = mode
            self.timeout = timeout

        def get_user(self, user_id):
            if self.mode == "integration":
                return {"id": user_id, "name": "真实用户"}
            return {"id": user_id, "name": "模拟用户"}

    return UserService(mode=config["mode"], timeout=config["timeout"])


def test_user_service(user_service):
    """测试依赖注入的服务"""
    user = user_service.get_user(1)
    assert user["id"] == 1
    print(f"\n[依赖注入] 获取用户: {user}")

wxh.wfuyuu27.cn/71935.Doc
wxh.wfuyuu27.cn/42462.Doc
wxh.wfuyuu27.cn/64268.Doc
wxh.wfuyuu27.cn/39773.Doc
wxh.wfuyuu27.cn/86222.Doc
wxh.wfuyuu27.cn/28806.Doc
wxh.wfuyuu27.cn/08068.Doc
wxh.wfuyuu27.cn/26246.Doc
wxh.wfuyuu27.cn/40626.Doc
wxh.wfuyuu27.cn/44606.Doc
wxg.wfuyuu27.cn/22028.Doc
wxg.wfuyuu27.cn/88224.Doc
wxg.wfuyuu27.cn/35977.Doc
wxg.wfuyuu27.cn/06866.Doc
wxg.wfuyuu27.cn/84000.Doc
wxg.wfuyuu27.cn/77937.Doc
wxg.wfuyuu27.cn/04242.Doc
wxg.wfuyuu27.cn/86260.Doc
wxg.wfuyuu27.cn/62448.Doc
wxg.wfuyuu27.cn/75375.Doc
wxf.wfuyuu27.cn/24286.Doc
wxf.wfuyuu27.cn/86428.Doc
wxf.wfuyuu27.cn/46608.Doc
wxf.wfuyuu27.cn/66860.Doc
wxf.wfuyuu27.cn/46260.Doc
wxf.wfuyuu27.cn/55711.Doc
wxf.wfuyuu27.cn/42866.Doc
wxf.wfuyuu27.cn/40880.Doc
wxf.wfuyuu27.cn/84686.Doc
wxf.wfuyuu27.cn/84044.Doc
wxd.wfuyuu27.cn/44066.Doc
wxd.wfuyuu27.cn/20040.Doc
wxd.wfuyuu27.cn/62046.Doc
wxd.wfuyuu27.cn/82082.Doc
wxd.wfuyuu27.cn/00606.Doc
wxd.wfuyuu27.cn/06404.Doc
wxd.wfuyuu27.cn/60086.Doc
wxd.wfuyuu27.cn/26242.Doc
wxd.wfuyuu27.cn/17717.Doc
wxd.wfuyuu27.cn/82622.Doc
wxs.wfuyuu27.cn/24068.Doc
wxs.wfuyuu27.cn/82868.Doc
wxs.wfuyuu27.cn/02062.Doc
wxs.wfuyuu27.cn/00462.Doc
wxs.wfuyuu27.cn/62806.Doc
wxs.wfuyuu27.cn/80460.Doc
wxs.wfuyuu27.cn/82026.Doc
wxs.wfuyuu27.cn/28644.Doc
wxs.wfuyuu27.cn/08244.Doc
wxs.wfuyuu27.cn/77379.Doc
wxa.wfuyuu27.cn/44242.Doc
wxa.wfuyuu27.cn/04288.Doc
wxa.wfuyuu27.cn/28226.Doc
wxa.wfuyuu27.cn/20282.Doc
wxa.wfuyuu27.cn/42800.Doc
wxa.wfuyuu27.cn/00488.Doc
wxa.wfuyuu27.cn/86442.Doc
wxa.wfuyuu27.cn/82606.Doc
wxa.wfuyuu27.cn/40088.Doc
wxa.wfuyuu27.cn/86068.Doc
wxp.wfuyuu27.cn/42442.Doc
wxp.wfuyuu27.cn/22224.Doc
wxp.wfuyuu27.cn/60266.Doc
wxp.wfuyuu27.cn/80260.Doc
wxp.wfuyuu27.cn/24684.Doc
wxp.wfuyuu27.cn/04862.Doc
wxp.wfuyuu27.cn/20680.Doc
wxp.wfuyuu27.cn/48640.Doc
wxp.wfuyuu27.cn/04020.Doc
wxp.wfuyuu27.cn/68280.Doc
wxo.wfuyuu27.cn/48826.Doc
wxo.wfuyuu27.cn/37193.Doc
wxo.wfuyuu27.cn/97375.Doc
wxo.wfuyuu27.cn/86800.Doc
wxo.wfuyuu27.cn/02482.Doc
wxo.wfuyuu27.cn/88208.Doc
wxo.wfuyuu27.cn/00444.Doc
wxo.wfuyuu27.cn/80682.Doc
wxo.wfuyuu27.cn/02480.Doc
wxo.wfuyuu27.cn/24826.Doc
wxi.wfuyuu27.cn/62686.Doc
wxi.wfuyuu27.cn/44026.Doc
wxi.wfuyuu27.cn/64022.Doc
wxi.wfuyuu27.cn/26402.Doc
wxi.wfuyuu27.cn/66088.Doc
wxi.wfuyuu27.cn/46026.Doc
wxi.wfuyuu27.cn/26684.Doc
wxi.wfuyuu27.cn/64462.Doc
wxi.wfuyuu27.cn/64806.Doc
wxi.wfuyuu27.cn/42800.Doc
wxu.wfuyuu27.cn/06408.Doc
wxu.wfuyuu27.cn/80828.Doc
wxu.wfuyuu27.cn/22642.Doc
wxu.wfuyuu27.cn/24484.Doc
wxu.wfuyuu27.cn/02222.Doc
wxu.wfuyuu27.cn/08804.Doc
wxu.wfuyuu27.cn/64420.Doc
wxu.wfuyuu27.cn/04200.Doc
wxu.wfuyuu27.cn/00424.Doc
wxu.wfuyuu27.cn/62864.Doc
wxy.wfuyuu27.cn/48860.Doc
wxy.wfuyuu27.cn/20620.Doc
wxy.wfuyuu27.cn/42268.Doc
wxy.wfuyuu27.cn/46406.Doc
wxy.wfuyuu27.cn/17797.Doc
wxy.wfuyuu27.cn/42668.Doc
wxy.wfuyuu27.cn/44622.Doc
wxy.wfuyuu27.cn/66060.Doc
wxy.wfuyuu27.cn/42028.Doc
wxy.wfuyuu27.cn/20002.Doc
wxt.wfuyuu27.cn/20068.Doc
wxt.wfuyuu27.cn/42842.Doc
wxt.wfuyuu27.cn/64824.Doc
wxt.wfuyuu27.cn/02062.Doc
wxt.wfuyuu27.cn/26026.Doc
wxt.wfuyuu27.cn/26064.Doc
wxt.wfuyuu27.cn/22820.Doc
wxt.wfuyuu27.cn/77971.Doc
wxt.wfuyuu27.cn/28068.Doc
wxt.wfuyuu27.cn/51397.Doc
wxr.wfuyuu27.cn/46022.Doc
wxr.wfuyuu27.cn/20286.Doc
wxr.wfuyuu27.cn/68662.Doc
wxr.wfuyuu27.cn/26664.Doc
wxr.wfuyuu27.cn/60448.Doc
wxr.wfuyuu27.cn/84464.Doc
wxr.wfuyuu27.cn/62462.Doc
wxr.wfuyuu27.cn/93953.Doc
wxr.wfuyuu27.cn/62662.Doc
wxr.wfuyuu27.cn/42208.Doc
wxe.wfuyuu27.cn/44466.Doc
wxe.wfuyuu27.cn/59537.Doc
wxe.wfuyuu27.cn/80426.Doc
wxe.wfuyuu27.cn/48888.Doc
wxe.wfuyuu27.cn/55393.Doc
wxe.wfuyuu27.cn/06620.Doc
wxe.wfuyuu27.cn/22862.Doc
wxe.wfuyuu27.cn/42644.Doc
wxe.wfuyuu27.cn/26442.Doc
wxe.wfuyuu27.cn/24866.Doc
wxw.wfuyuu27.cn/91755.Doc
wxw.wfuyuu27.cn/86802.Doc
wxw.wfuyuu27.cn/20660.Doc
wxw.wfuyuu27.cn/44668.Doc
wxw.wfuyuu27.cn/02040.Doc
wxw.wfuyuu27.cn/64008.Doc
wxw.wfuyuu27.cn/64644.Doc
wxw.wfuyuu27.cn/46202.Doc
wxw.wfuyuu27.cn/04820.Doc
wxw.wfuyuu27.cn/24666.Doc
wxq.wfuyuu27.cn/88004.Doc
wxq.wfuyuu27.cn/08266.Doc
wxq.wfuyuu27.cn/48804.Doc
wxq.wfuyuu27.cn/64884.Doc
wxq.wfuyuu27.cn/88008.Doc
wxq.wfuyuu27.cn/59733.Doc
wxq.wfuyuu27.cn/88088.Doc
wxq.wfuyuu27.cn/20248.Doc
wxq.wfuyuu27.cn/80844.Doc
wxq.wfuyuu27.cn/64480.Doc
wzm.wfuyuu27.cn/08642.Doc
wzm.wfuyuu27.cn/48428.Doc
wzm.wfuyuu27.cn/48084.Doc
wzm.wfuyuu27.cn/44206.Doc
wzm.wfuyuu27.cn/91375.Doc
wzm.wfuyuu27.cn/84400.Doc
wzm.wfuyuu27.cn/40800.Doc
wzm.wfuyuu27.cn/66820.Doc
wzm.wfuyuu27.cn/06486.Doc
wzm.wfuyuu27.cn/28066.Doc
wzn.wfuyuu27.cn/04408.Doc
wzn.wfuyuu27.cn/44888.Doc
wzn.wfuyuu27.cn/42266.Doc
wzn.wfuyuu27.cn/02442.Doc
wzn.wfuyuu27.cn/53911.Doc
wzn.wfuyuu27.cn/17353.Doc
wzn.wfuyuu27.cn/24082.Doc
wzn.wfuyuu27.cn/64860.Doc
wzn.wfuyuu27.cn/66622.Doc
wzn.wfuyuu27.cn/40460.Doc
wzb.wfuyuu27.cn/24888.Doc
wzb.wfuyuu27.cn/06066.Doc
wzb.wfuyuu27.cn/46804.Doc
wzb.wfuyuu27.cn/40646.Doc
wzb.wfuyuu27.cn/88868.Doc
wzb.wfuyuu27.cn/24824.Doc
wzb.wfuyuu27.cn/08668.Doc
wzb.wfuyuu27.cn/42400.Doc
wzb.wfuyuu27.cn/42460.Doc
wzb.wfuyuu27.cn/28804.Doc
wzv.wfuyuu27.cn/42000.Doc
wzv.wfuyuu27.cn/82680.Doc
wzv.wfuyuu27.cn/66644.Doc
wzv.wfuyuu27.cn/04668.Doc
wzv.wfuyuu27.cn/66822.Doc
wzv.wfuyuu27.cn/22068.Doc
wzv.wfuyuu27.cn/40008.Doc
wzv.wfuyuu27.cn/86420.Doc
wzv.wfuyuu27.cn/28040.Doc
wzv.wfuyuu27.cn/73597.Doc
wzc.wfuyuu27.cn/88426.Doc
wzc.wfuyuu27.cn/40840.Doc
wzc.wfuyuu27.cn/06644.Doc
wzc.wfuyuu27.cn/04648.Doc
wzc.wfuyuu27.cn/60660.Doc
wzc.wfuyuu27.cn/20086.Doc
wzc.wfuyuu27.cn/24604.Doc
wzc.wfuyuu27.cn/28462.Doc
wzc.wfuyuu27.cn/64644.Doc
wzc.wfuyuu27.cn/99579.Doc
wzx.wfuyuu27.cn/42864.Doc
wzx.wfuyuu27.cn/91195.Doc
wzx.wfuyuu27.cn/66248.Doc
wzx.wfuyuu27.cn/00668.Doc
wzx.wfuyuu27.cn/66404.Doc
wzx.wfuyuu27.cn/06080.Doc
wzx.wfuyuu27.cn/20440.Doc
wzx.wfuyuu27.cn/08826.Doc
wzx.wfuyuu27.cn/42222.Doc
wzx.wfuyuu27.cn/28208.Doc
wzz.wfuyuu27.cn/64246.Doc
wzz.wfuyuu27.cn/64048.Doc
wzz.wfuyuu27.cn/02668.Doc
wzz.wfuyuu27.cn/28604.Doc
wzz.wfuyuu27.cn/66040.Doc
wzz.wfuyuu27.cn/64266.Doc
wzz.wfuyuu27.cn/88606.Doc
wzz.wfuyuu27.cn/62224.Doc
wzz.wfuyuu27.cn/06600.Doc
wzz.wfuyuu27.cn/82024.Doc
wzl.wfuyuu27.cn/08040.Doc
wzl.wfuyuu27.cn/59313.Doc
wzl.wfuyuu27.cn/80422.Doc
wzl.wfuyuu27.cn/24428.Doc
wzl.wfuyuu27.cn/77531.Doc
wzl.wfuyuu27.cn/84020.Doc
wzl.wfuyuu27.cn/99133.Doc
wzl.wfuyuu27.cn/22000.Doc
wzl.wfuyuu27.cn/62866.Doc
wzl.wfuyuu27.cn/88022.Doc
wzk.wfuyuu27.cn/08880.Doc
wzk.wfuyuu27.cn/86286.Doc
wzk.wfuyuu27.cn/68086.Doc
wzk.wfuyuu27.cn/04640.Doc
wzk.wfuyuu27.cn/26222.Doc
wzk.wfuyuu27.cn/66080.Doc
wzk.wfuyuu27.cn/99537.Doc
wzk.wfuyuu27.cn/22682.Doc
wzk.wfuyuu27.cn/11737.Doc
wzk.wfuyuu27.cn/40244.Doc
wzj.wfuyuu27.cn/48844.Doc
wzj.wfuyuu27.cn/60024.Doc
wzj.wfuyuu27.cn/86880.Doc
wzj.wfuyuu27.cn/88680.Doc
wzj.wfuyuu27.cn/91151.Doc
wzj.wfuyuu27.cn/22224.Doc
wzj.wfuyuu27.cn/42808.Doc
wzj.wfuyuu27.cn/04840.Doc
wzj.wfuyuu27.cn/86040.Doc
wzj.wfuyuu27.cn/60806.Doc
wzh.wfuyuu27.cn/60242.Doc
wzh.wfuyuu27.cn/22622.Doc
wzh.wfuyuu27.cn/02488.Doc
wzh.wfuyuu27.cn/48406.Doc
wzh.wfuyuu27.cn/04866.Doc
wzh.wfuyuu27.cn/80442.Doc
wzh.wfuyuu27.cn/68824.Doc
wzh.wfuyuu27.cn/24622.Doc
wzh.wfuyuu27.cn/20482.Doc
wzh.wfuyuu27.cn/28080.Doc
wzg.wfuyuu27.cn/46624.Doc
wzg.wfuyuu27.cn/42866.Doc
wzg.wfuyuu27.cn/44004.Doc
wzg.wfuyuu27.cn/46626.Doc
wzg.wfuyuu27.cn/48406.Doc
wzg.wfuyuu27.cn/26446.Doc
wzg.wfuyuu27.cn/46688.Doc
wzg.wfuyuu27.cn/22680.Doc
wzg.wfuyuu27.cn/84468.Doc
wzg.wfuyuu27.cn/64862.Doc
wzf.wfuyuu27.cn/20204.Doc
wzf.wfuyuu27.cn/22800.Doc
wzf.wfuyuu27.cn/28420.Doc
wzf.wfuyuu27.cn/84660.Doc
wzf.wfuyuu27.cn/82442.Doc
wzf.wfuyuu27.cn/60646.Doc
wzf.wfuyuu27.cn/00242.Doc
wzf.wfuyuu27.cn/11979.Doc
wzf.wfuyuu27.cn/46084.Doc
wzf.wfuyuu27.cn/24822.Doc
wzd.wfuyuu27.cn/48224.Doc
wzd.wfuyuu27.cn/26684.Doc
wzd.wfuyuu27.cn/64660.Doc
wzd.wfuyuu27.cn/66488.Doc
wzd.wfuyuu27.cn/04282.Doc
wzd.wfuyuu27.cn/80664.Doc
wzd.wfuyuu27.cn/86828.Doc
wzd.wfuyuu27.cn/00402.Doc
wzd.wfuyuu27.cn/48684.Doc
wzd.wfuyuu27.cn/64886.Doc
wzs.wfuyuu27.cn/59733.Doc
wzs.wfuyuu27.cn/02206.Doc
wzs.wfuyuu27.cn/00088.Doc
wzs.wfuyuu27.cn/06264.Doc
wzs.wfuyuu27.cn/40002.Doc
wzs.wfuyuu27.cn/42002.Doc
wzs.wfuyuu27.cn/95339.Doc
wzs.wfuyuu27.cn/04646.Doc
wzs.wfuyuu27.cn/59513.Doc
wzs.wfuyuu27.cn/48868.Doc
wza.wfuyuu27.cn/88808.Doc
wza.wfuyuu27.cn/37513.Doc
wza.wfuyuu27.cn/26422.Doc
wza.wfuyuu27.cn/91379.Doc
wza.wfuyuu27.cn/08068.Doc
wza.wfuyuu27.cn/86280.Doc
wza.wfuyuu27.cn/46608.Doc
wza.wfuyuu27.cn/93913.Doc
wza.wfuyuu27.cn/19119.Doc
wza.wfuyuu27.cn/82866.Doc
wzp.wfuyuu27.cn/82240.Doc
wzp.wfuyuu27.cn/46082.Doc
wzp.wfuyuu27.cn/46068.Doc
wzp.wfuyuu27.cn/08222.Doc
wzp.wfuyuu27.cn/28644.Doc
wzp.wfuyuu27.cn/22024.Doc
wzp.wfuyuu27.cn/26628.Doc
wzp.wfuyuu27.cn/68462.Doc
wzp.wfuyuu27.cn/82800.Doc
wzp.wfuyuu27.cn/86084.Doc
wzo.wfuyuu27.cn/08820.Doc
wzo.wfuyuu27.cn/28220.Doc
wzo.wfuyuu27.cn/20684.Doc
wzo.wfuyuu27.cn/17753.Doc
wzo.wfuyuu27.cn/68686.Doc
wzo.wfuyuu27.cn/00226.Doc
wzo.wfuyuu27.cn/00282.Doc
wzo.wfuyuu27.cn/68866.Doc
wzo.wfuyuu27.cn/44042.Doc
wzo.wfuyuu27.cn/62240.Doc
wzi.wfuyuu27.cn/97579.Doc
wzi.wfuyuu27.cn/06022.Doc
wzi.wfuyuu27.cn/00062.Doc
wzi.wfuyuu27.cn/28882.Doc
wzi.wfuyuu27.cn/68864.Doc
wzi.wfuyuu27.cn/28666.Doc
wzi.wfuyuu27.cn/88008.Doc
wzi.wfuyuu27.cn/66882.Doc
wzi.wfuyuu27.cn/24466.Doc
wzi.wfuyuu27.cn/02860.Doc
wzu.wfuyuu27.cn/80866.Doc
wzu.wfuyuu27.cn/20842.Doc
wzu.wfuyuu27.cn/80466.Doc
wzu.wfuyuu27.cn/02022.Doc
wzu.wfuyuu27.cn/26064.Doc
wzu.wfuyuu27.cn/24446.Doc
wzu.wfuyuu27.cn/64624.Doc
wzu.wfuyuu27.cn/82200.Doc
wzu.wfuyuu27.cn/22884.Doc
wzu.wfuyuu27.cn/00820.Doc
wzy.wfuyuu27.cn/26206.Doc
wzy.wfuyuu27.cn/06822.Doc
wzy.wfuyuu27.cn/20084.Doc
wzy.wfuyuu27.cn/46886.Doc
wzy.wfuyuu27.cn/06222.Doc
wzy.wfuyuu27.cn/82048.Doc
wzy.wfuyuu27.cn/68286.Doc
wzy.wfuyuu27.cn/08464.Doc
wzy.wfuyuu27.cn/66484.Doc
wzy.wfuyuu27.cn/42202.Doc
wzt.wfuyuu27.cn/64080.Doc
wzt.wfuyuu27.cn/02806.Doc
wzt.wfuyuu27.cn/24448.Doc
wzt.wfuyuu27.cn/42802.Doc
wzt.wfuyuu27.cn/93793.Doc
wzt.wfuyuu27.cn/06646.Doc
wzt.wfuyuu27.cn/55711.Doc
wzt.wfuyuu27.cn/20868.Doc
wzt.wfuyuu27.cn/20806.Doc
wzt.wfuyuu27.cn/51711.Doc
wzr.wfuyuu27.cn/88608.Doc
wzr.wfuyuu27.cn/44046.Doc
wzr.wfuyuu27.cn/77351.Doc
wzr.wfuyuu27.cn/00826.Doc
wzr.wfuyuu27.cn/24020.Doc
wzr.wfuyuu27.cn/42060.Doc
wzr.wfuyuu27.cn/02480.Doc
wzr.wfuyuu27.cn/31957.Doc
wzr.wfuyuu27.cn/26480.Doc
wzr.wfuyuu27.cn/28068.Doc
wze.wfuyuu27.cn/88082.Doc
wze.wfuyuu27.cn/80680.Doc
wze.wfuyuu27.cn/46442.Doc
wze.wfuyuu27.cn/46620.Doc
wze.wfuyuu27.cn/62266.Doc
wze.wfuyuu27.cn/44408.Doc
wze.wfuyuu27.cn/66026.Doc
wze.wfuyuu27.cn/11975.Doc
wze.wfuyuu27.cn/44824.Doc
wze.wfuyuu27.cn/28082.Doc
wzw.wfuyuu27.cn/80240.Doc
wzw.wfuyuu27.cn/80682.Doc
wzw.wfuyuu27.cn/82842.Doc
wzw.wfuyuu27.cn/46422.Doc
wzw.wfuyuu27.cn/60466.Doc
wzw.wfuyuu27.cn/51117.Doc
wzw.wfuyuu27.cn/20280.Doc
wzw.wfuyuu27.cn/66064.Doc
wzw.wfuyuu27.cn/84028.Doc
wzw.wfuyuu27.cn/64064.Doc
wzq.wfuyuu27.cn/60068.Doc
wzq.wfuyuu27.cn/80688.Doc
wzq.wfuyuu27.cn/91199.Doc
wzq.wfuyuu27.cn/40442.Doc
wzq.wfuyuu27.cn/64286.Doc
wzq.wfuyuu27.cn/11511.Doc
wzq.wfuyuu27.cn/48628.Doc
wzq.wfuyuu27.cn/20880.Doc
wzq.wfuyuu27.cn/86426.Doc
wzq.wfuyuu27.cn/80828.Doc
wlm.wfuyuu27.cn/57315.Doc
wlm.wfuyuu27.cn/46448.Doc
wlm.wfuyuu27.cn/64644.Doc
wlm.wfuyuu27.cn/80406.Doc
wlm.wfuyuu27.cn/84022.Doc
wlm.wfuyuu27.cn/28624.Doc
wlm.wfuyuu27.cn/06444.Doc
wlm.wfuyuu27.cn/11517.Doc
wlm.wfuyuu27.cn/39517.Doc
wlm.wfuyuu27.cn/06026.Doc
wln.wfuyuu27.cn/20202.Doc
wln.wfuyuu27.cn/02664.Doc
wln.wfuyuu27.cn/46020.Doc
wln.wfuyuu27.cn/82468.Doc
wln.wfuyuu27.cn/42042.Doc
wln.wfuyuu27.cn/88204.Doc
wln.wfuyuu27.cn/04020.Doc
wln.wfuyuu27.cn/04460.Doc
wln.wfuyuu27.cn/04008.Doc
wln.wfuyuu27.cn/62884.Doc
wlb.wfuyuu27.cn/88064.Doc
wlb.wfuyuu27.cn/24882.Doc
wlb.wfuyuu27.cn/48888.Doc
wlb.wfuyuu27.cn/00828.Doc
wlb.wfuyuu27.cn/82246.Doc
wlb.wfuyuu27.cn/28664.Doc
wlb.wfuyuu27.cn/82086.Doc
wlb.wfuyuu27.cn/28286.Doc
wlb.wfuyuu27.cn/31771.Doc
wlb.wfuyuu27.cn/40460.Doc
wlv.wfuyuu27.cn/04260.Doc
wlv.wfuyuu27.cn/46082.Doc
wlv.wfuyuu27.cn/42448.Doc
wlv.wfuyuu27.cn/04482.Doc
wlv.wfuyuu27.cn/31317.Doc
wlv.wfuyuu27.cn/99755.Doc
wlv.wfuyuu27.cn/84284.Doc
wlv.wfuyuu27.cn/11773.Doc
wlv.wfuyuu27.cn/84284.Doc
wlv.wfuyuu27.cn/86682.Doc
wlc.wfuyuu27.cn/00444.Doc
wlc.wfuyuu27.cn/40202.Doc
wlc.wfuyuu27.cn/22448.Doc
wlc.wfuyuu27.cn/68264.Doc
wlc.wfuyuu27.cn/42802.Doc
wlc.wfuyuu27.cn/04246.Doc
wlc.wfuyuu27.cn/20600.Doc
wlc.wfuyuu27.cn/86880.Doc
wlc.wfuyuu27.cn/82424.Doc
wlc.wfuyuu27.cn/20666.Doc
wlx.wfuyuu27.cn/19951.Doc
wlx.wfuyuu27.cn/44864.Doc
wlx.wfuyuu27.cn/46462.Doc
wlx.wfuyuu27.cn/82268.Doc
wlx.wfuyuu27.cn/24462.Doc
wlx.wfuyuu27.cn/26222.Doc
wlx.wfuyuu27.cn/88604.Doc
wlx.wfuyuu27.cn/60826.Doc
wlx.wfuyuu27.cn/46624.Doc
wlx.wfuyuu27.cn/33173.Doc
wlz.wfuyuu27.cn/82266.Doc
wlz.wfuyuu27.cn/66660.Doc
wlz.wfuyuu27.cn/48626.Doc
wlz.wfuyuu27.cn/22688.Doc
wlz.wfuyuu27.cn/68202.Doc
wlz.wfuyuu27.cn/26428.Doc
wlz.wfuyuu27.cn/62840.Doc
wlz.wfuyuu27.cn/40480.Doc
wlz.wfuyuu27.cn/20226.Doc
wlz.wfuyuu27.cn/48208.Doc
