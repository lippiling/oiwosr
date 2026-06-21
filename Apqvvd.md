苫锤澄萍霞


Python上下文管理器在测试中的应用——mock、资源管理与基准测试

上下文管理器在测试中有着广泛的应用，从模拟依赖到资源清理，再到性能基准测试。

import contextlib
import io
import sys
import time
from unittest import mock
from contextlib import contextmanager, ExitStack, redirect_stdout, nullcontext

# ========== contextmanager与mock.patch组合 ==========
@contextmanager
def mock_env_variable(key: str, value: str):
    """
    临时设置环境变量的上下文管理器。
    测试时模拟环境变量，退出时自动恢复。
    """
    import os
    original = os.environ.get(key)
    os.environ[key] = value
    try:
        yield  # 测试代码在此执行
    finally:
        if original is None:
            del os.environ[key]
        else:
            os.environ[key] = original


@contextmanager
def mock_database_connection(return_data=None):
    """
    模拟数据库连接的上下文管理器。
    避免测试中真正连接数据库。
    """
    fake_conn = mock.MagicMock()
    fake_conn.execute.return_value = return_data or []
    try:
        yield fake_conn
    finally:
        # 确保模拟连接被正确关闭
        fake_conn.close.assert_called_once()


# 演示组合使用的测试场景
class DataService:
    """依赖数据库连接的服务"""

    def __init__(self, db_conn):
        self.db = db_conn

    def get_users(self):
        cursor = self.db.execute("SELECT * FROM users")
        return cursor.fetchall()


def test_data_service():
    """演示mock上下文管理器如何简化测试"""
    expected = [{"id": 1, "name": "张三"}]
    with mock_database_connection(expected) as conn:
        service = DataService(conn)
        result = service.get_users()
        assert result == expected
        print(f"测试通过: 获取到 {len(result)} 个用户")


# ========== ExitStack 动态清理 ==========
def cleanup_resources_demo():
    """
    ExitStack允许动态注册清理回调。
    适用于不确定需要打开多少资源的情况。
    """
    cleanup_actions = []  # 记录清理动作，用于验证

    @contextmanager
    def managed_resource(name):
        """模拟一个需要清理的资源"""
        print(f"  打开资源: {name}")
        try:
            yield name
        finally:
            print(f"  关闭资源: {name}")
            cleanup_actions.append(f"cleaned:{name}")

    # ExitStack确保所有资源都被正确关闭
    with ExitStack() as stack:
        stack.enter_context(managed_resource("数据库连接"))
        stack.enter_context(managed_resource("文件句柄"))
        # 条件性注册资源
        if True:
            stack.enter_context(managed_resource("网络连接"))
        # 退出with块时，按后进先出顺序清理
    print(f"  清理记录: {cleanup_actions}")


# ========== redirect_stdout 在测试中 ==========
def greet(name: str) -> None:
    """直接打印到标准输出的函数"""
    print(f"你好, {name}!")


def test_greet_function():
    """
    测试打印输出的函数。
    使用redirect_stdout捕获print的输出，不污染测试控制台。
    """
    captured = io.StringIO()
    with redirect_stdout(captured):
        greet("测试用户")
    output = captured.getvalue().strip()
    assert output == "你好, 测试用户!"
    print(f"redirect_stdout测试通过: '{output}'")


# ========== nullcontext 简化条件分支 ==========
def optional_mock_demo(use_mock: bool):
    """
    根据条件决定是否使用mock。
    nullcontext在不需mock时返回原对象，避免if/else分支。
    """
    real_object = {"data": "真实数据"}

    # 如果use_mock为True则用mock，否则直接使用原对象
    mock_or_not = mock.MagicMock() if use_mock else nullcontext(real_object)

    with mock_or_not as ctx:
        if use_mock:
            ctx.get_data.return_value = "模拟数据"
            result = ctx.get_data()
        else:
            result = ctx["data"]
        return result


# ========== 计时上下文管理器用于基准测试 ==========
@contextmanager
def timing(name: str = "代码块"):
    """
    测量代码执行时间的上下文管理器。
    在基准测试中非常有用。
    """
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        print(f"[基准测试] {name} 耗时: {elapsed * 1000:.2f} 毫秒")


def benchmark_sort():
    """演示计时上下文管理器"""
    data = list(range(10000))
    with timing("列表排序"):
        sorted_data = sorted(data, reverse=True)
    assert len(sorted_data) == 10000
    print(f"排序完成，首元素: {sorted_data[0]}")


# ========== 综合测试示例 ==========
def run_all_tests():
    """运行所有演示"""
    print("=== mock数据库测试 ===")
    test_data_service()
    print("\n=== ExitStack资源清理 ===")
    cleanup_resources_demo()
    print("\n=== redirect_stdout ===")
    test_greet_function()
    print("\n=== 条件mock ===")
    r1 = optional_mock_demo(True)
    r2 = optional_mock_demo(False)
    print(f"Mock模式: {r1}, 真实模式: {r2}")
    print("\n=== 基准测试 ===")
    benchmark_sort()


if __name__ == "__main__":
    run_all_tests()

爸喝凰艘甘冻善诶壮估腊辛甭百胶

m.edn.mglwx.cn/75573.Doc
m.edn.mglwx.cn/53775.Doc
m.edn.mglwx.cn/99135.Doc
m.edn.mglwx.cn/20822.Doc
m.edn.mglwx.cn/20444.Doc
m.edb.mglwx.cn/60406.Doc
m.edb.mglwx.cn/77391.Doc
m.edb.mglwx.cn/73139.Doc
m.edb.mglwx.cn/77393.Doc
m.edb.mglwx.cn/64802.Doc
m.edb.mglwx.cn/57319.Doc
m.edb.mglwx.cn/48600.Doc
m.edb.mglwx.cn/08640.Doc
m.edb.mglwx.cn/48662.Doc
m.edb.mglwx.cn/26408.Doc
m.edb.mglwx.cn/40262.Doc
m.edb.mglwx.cn/66244.Doc
m.edb.mglwx.cn/95395.Doc
m.edb.mglwx.cn/91933.Doc
m.edb.mglwx.cn/95131.Doc
m.edb.mglwx.cn/64220.Doc
m.edb.mglwx.cn/33153.Doc
m.edb.mglwx.cn/68066.Doc
m.edb.mglwx.cn/86086.Doc
m.edb.mglwx.cn/86406.Doc
m.edv.mglwx.cn/00226.Doc
m.edv.mglwx.cn/99195.Doc
m.edv.mglwx.cn/19195.Doc
m.edv.mglwx.cn/59759.Doc
m.edv.mglwx.cn/66400.Doc
m.edv.mglwx.cn/95719.Doc
m.edv.mglwx.cn/55155.Doc
m.edv.mglwx.cn/00446.Doc
m.edv.mglwx.cn/75917.Doc
m.edv.mglwx.cn/00466.Doc
m.edv.mglwx.cn/53995.Doc
m.edv.mglwx.cn/93971.Doc
m.edv.mglwx.cn/53513.Doc
m.edv.mglwx.cn/68662.Doc
m.edv.mglwx.cn/37759.Doc
m.edv.mglwx.cn/66406.Doc
m.edv.mglwx.cn/59597.Doc
m.edv.mglwx.cn/75571.Doc
m.edv.mglwx.cn/77599.Doc
m.edv.mglwx.cn/51713.Doc
m.edc.mglwx.cn/97537.Doc
m.edc.mglwx.cn/57153.Doc
m.edc.mglwx.cn/37971.Doc
m.edc.mglwx.cn/53133.Doc
m.edc.mglwx.cn/35371.Doc
m.edc.mglwx.cn/17311.Doc
m.edc.mglwx.cn/88628.Doc
m.edc.mglwx.cn/31773.Doc
m.edc.mglwx.cn/93119.Doc
m.edc.mglwx.cn/31953.Doc
m.edc.mglwx.cn/93717.Doc
m.edc.mglwx.cn/02640.Doc
m.edc.mglwx.cn/20488.Doc
m.edc.mglwx.cn/82684.Doc
m.edc.mglwx.cn/48084.Doc
m.edc.mglwx.cn/59953.Doc
m.edc.mglwx.cn/06246.Doc
m.edc.mglwx.cn/86048.Doc
m.edc.mglwx.cn/59577.Doc
m.edc.mglwx.cn/15713.Doc
m.edx.mglwx.cn/19719.Doc
m.edx.mglwx.cn/59113.Doc
m.edx.mglwx.cn/79395.Doc
m.edx.mglwx.cn/91133.Doc
m.edx.mglwx.cn/97759.Doc
m.edx.mglwx.cn/22024.Doc
m.edx.mglwx.cn/06622.Doc
m.edx.mglwx.cn/22842.Doc
m.edx.mglwx.cn/99773.Doc
m.edx.mglwx.cn/17353.Doc
m.edx.mglwx.cn/99715.Doc
m.edx.mglwx.cn/73519.Doc
m.edx.mglwx.cn/17573.Doc
m.edx.mglwx.cn/73911.Doc
m.edx.mglwx.cn/48402.Doc
m.edx.mglwx.cn/08826.Doc
m.edx.mglwx.cn/97377.Doc
m.edx.mglwx.cn/82606.Doc
m.edx.mglwx.cn/33519.Doc
m.edx.mglwx.cn/75393.Doc
m.edz.mglwx.cn/15151.Doc
m.edz.mglwx.cn/51997.Doc
m.edz.mglwx.cn/97731.Doc
m.edz.mglwx.cn/19537.Doc
m.edz.mglwx.cn/53591.Doc
m.edz.mglwx.cn/75733.Doc
m.edz.mglwx.cn/80004.Doc
m.edz.mglwx.cn/57179.Doc
m.edz.mglwx.cn/19373.Doc
m.edz.mglwx.cn/11731.Doc
m.edz.mglwx.cn/55197.Doc
m.edz.mglwx.cn/68248.Doc
m.edz.mglwx.cn/26068.Doc
m.edz.mglwx.cn/82866.Doc
m.edz.mglwx.cn/53735.Doc
m.edz.mglwx.cn/46620.Doc
m.edz.mglwx.cn/57779.Doc
m.edz.mglwx.cn/44266.Doc
m.edz.mglwx.cn/31951.Doc
m.edz.mglwx.cn/20280.Doc
m.edl.mglwx.cn/64880.Doc
m.edl.mglwx.cn/59717.Doc
m.edl.mglwx.cn/93573.Doc
m.edl.mglwx.cn/17951.Doc
m.edl.mglwx.cn/13133.Doc
m.edl.mglwx.cn/77511.Doc
m.edl.mglwx.cn/17159.Doc
m.edl.mglwx.cn/73759.Doc
m.edl.mglwx.cn/66422.Doc
m.edl.mglwx.cn/82804.Doc
m.edl.mglwx.cn/93751.Doc
m.edl.mglwx.cn/28020.Doc
m.edl.mglwx.cn/44028.Doc
m.edl.mglwx.cn/71353.Doc
m.edl.mglwx.cn/82202.Doc
m.edl.mglwx.cn/91739.Doc
m.edl.mglwx.cn/60842.Doc
m.edl.mglwx.cn/33191.Doc
m.edl.mglwx.cn/93911.Doc
m.edl.mglwx.cn/59133.Doc
m.edk.mglwx.cn/91117.Doc
m.edk.mglwx.cn/35395.Doc
m.edk.mglwx.cn/97337.Doc
m.edk.mglwx.cn/95331.Doc
m.edk.mglwx.cn/97911.Doc
m.edk.mglwx.cn/55719.Doc
m.edk.mglwx.cn/15917.Doc
m.edk.mglwx.cn/79195.Doc
m.edk.mglwx.cn/28868.Doc
m.edk.mglwx.cn/75519.Doc
m.edk.mglwx.cn/99335.Doc
m.edk.mglwx.cn/95731.Doc
m.edk.mglwx.cn/57571.Doc
m.edk.mglwx.cn/26820.Doc
m.edk.mglwx.cn/15535.Doc
m.edk.mglwx.cn/39513.Doc
m.edk.mglwx.cn/60686.Doc
m.edk.mglwx.cn/24406.Doc
m.edk.mglwx.cn/75975.Doc
m.edk.mglwx.cn/86804.Doc
m.edj.mglwx.cn/06048.Doc
m.edj.mglwx.cn/13797.Doc
m.edj.mglwx.cn/5.Doc
m.edj.mglwx.cn/02846.Doc
m.edj.mglwx.cn/39173.Doc
m.edj.mglwx.cn/40082.Doc
m.edj.mglwx.cn/71113.Doc
m.edj.mglwx.cn/91971.Doc
m.edj.mglwx.cn/95713.Doc
m.edj.mglwx.cn/80648.Doc
m.edj.mglwx.cn/33519.Doc
m.edj.mglwx.cn/31313.Doc
m.edj.mglwx.cn/15155.Doc
m.edj.mglwx.cn/64822.Doc
m.edj.mglwx.cn/28426.Doc
m.edj.mglwx.cn/13959.Doc
m.edj.mglwx.cn/91199.Doc
m.edj.mglwx.cn/13757.Doc
m.edj.mglwx.cn/31715.Doc
m.edj.mglwx.cn/77739.Doc
m.edh.mglwx.cn/06802.Doc
m.edh.mglwx.cn/71171.Doc
m.edh.mglwx.cn/75373.Doc
m.edh.mglwx.cn/15177.Doc
m.edh.mglwx.cn/33173.Doc
m.edh.mglwx.cn/15157.Doc
m.edh.mglwx.cn/17519.Doc
m.edh.mglwx.cn/55573.Doc
m.edh.mglwx.cn/95111.Doc
m.edh.mglwx.cn/64086.Doc
m.edh.mglwx.cn/59373.Doc
m.edh.mglwx.cn/20606.Doc
m.edh.mglwx.cn/97579.Doc
m.edh.mglwx.cn/57395.Doc
m.edh.mglwx.cn/71597.Doc
m.edh.mglwx.cn/37733.Doc
m.edh.mglwx.cn/99775.Doc
m.edh.mglwx.cn/28444.Doc
m.edh.mglwx.cn/77155.Doc
m.edh.mglwx.cn/26620.Doc
m.edg.mglwx.cn/31759.Doc
m.edg.mglwx.cn/82466.Doc
m.edg.mglwx.cn/99593.Doc
m.edg.mglwx.cn/66608.Doc
m.edg.mglwx.cn/75711.Doc
m.edg.mglwx.cn/73759.Doc
m.edg.mglwx.cn/57775.Doc
m.edg.mglwx.cn/88846.Doc
m.edg.mglwx.cn/86640.Doc
m.edg.mglwx.cn/55751.Doc
m.edg.mglwx.cn/68820.Doc
m.edg.mglwx.cn/08804.Doc
m.edg.mglwx.cn/53335.Doc
m.edg.mglwx.cn/13551.Doc
m.edg.mglwx.cn/99719.Doc
m.edg.mglwx.cn/66666.Doc
m.edg.mglwx.cn/35999.Doc
m.edg.mglwx.cn/51517.Doc
m.edg.mglwx.cn/37159.Doc
m.edg.mglwx.cn/37955.Doc
m.edf.mglwx.cn/31991.Doc
m.edf.mglwx.cn/19939.Doc
m.edf.mglwx.cn/46660.Doc
m.edf.mglwx.cn/77777.Doc
m.edf.mglwx.cn/15335.Doc
m.edf.mglwx.cn/73715.Doc
m.edf.mglwx.cn/53357.Doc
m.edf.mglwx.cn/91977.Doc
m.edf.mglwx.cn/11571.Doc
m.edf.mglwx.cn/26082.Doc
m.edf.mglwx.cn/93517.Doc
m.edf.mglwx.cn/57139.Doc
m.edf.mglwx.cn/93311.Doc
m.edf.mglwx.cn/97337.Doc
m.edf.mglwx.cn/88000.Doc
m.edf.mglwx.cn/13197.Doc
m.edf.mglwx.cn/91775.Doc
m.edf.mglwx.cn/31375.Doc
m.edf.mglwx.cn/53913.Doc
m.edf.mglwx.cn/62684.Doc
m.edd.mglwx.cn/51553.Doc
m.edd.mglwx.cn/53937.Doc
m.edd.mglwx.cn/79111.Doc
m.edd.mglwx.cn/75393.Doc
m.edd.mglwx.cn/55539.Doc
m.edd.mglwx.cn/55177.Doc
m.edd.mglwx.cn/40488.Doc
m.edd.mglwx.cn/46646.Doc
m.edd.mglwx.cn/75775.Doc
m.edd.mglwx.cn/88862.Doc
m.edd.mglwx.cn/44664.Doc
m.edd.mglwx.cn/31595.Doc
m.edd.mglwx.cn/44866.Doc
m.edd.mglwx.cn/51933.Doc
m.edd.mglwx.cn/19391.Doc
m.edd.mglwx.cn/15557.Doc
m.edd.mglwx.cn/93937.Doc
m.edd.mglwx.cn/17711.Doc
m.edd.mglwx.cn/53737.Doc
m.edd.mglwx.cn/00440.Doc
m.eds.mglwx.cn/84204.Doc
m.eds.mglwx.cn/68064.Doc
m.eds.mglwx.cn/17939.Doc
m.eds.mglwx.cn/06808.Doc
m.eds.mglwx.cn/64000.Doc
m.eds.mglwx.cn/06800.Doc
m.eds.mglwx.cn/08006.Doc
m.eds.mglwx.cn/26442.Doc
m.eds.mglwx.cn/59511.Doc
m.eds.mglwx.cn/42028.Doc
m.eds.mglwx.cn/15119.Doc
m.eds.mglwx.cn/95955.Doc
m.eds.mglwx.cn/93375.Doc
m.eds.mglwx.cn/62082.Doc
m.eds.mglwx.cn/82200.Doc
m.eds.mglwx.cn/53599.Doc
m.eds.mglwx.cn/24428.Doc
m.eds.mglwx.cn/19951.Doc
m.eds.mglwx.cn/40088.Doc
m.eds.mglwx.cn/91913.Doc
m.eda.mglwx.cn/08486.Doc
m.eda.mglwx.cn/66482.Doc
m.eda.mglwx.cn/02664.Doc
m.eda.mglwx.cn/48804.Doc
m.eda.mglwx.cn/59999.Doc
m.eda.mglwx.cn/75319.Doc
m.eda.mglwx.cn/22482.Doc
m.eda.mglwx.cn/19917.Doc
m.eda.mglwx.cn/66644.Doc
m.eda.mglwx.cn/15951.Doc
m.eda.mglwx.cn/04604.Doc
m.eda.mglwx.cn/95973.Doc
m.eda.mglwx.cn/31355.Doc
m.eda.mglwx.cn/06604.Doc
m.eda.mglwx.cn/00426.Doc
m.eda.mglwx.cn/28444.Doc
m.eda.mglwx.cn/99575.Doc
m.eda.mglwx.cn/60206.Doc
m.eda.mglwx.cn/71993.Doc
m.eda.mglwx.cn/17357.Doc
m.edp.mglwx.cn/97799.Doc
m.edp.mglwx.cn/66624.Doc
m.edp.mglwx.cn/79139.Doc
m.edp.mglwx.cn/75795.Doc
m.edp.mglwx.cn/06400.Doc
m.edp.mglwx.cn/22602.Doc
m.edp.mglwx.cn/57511.Doc
m.edp.mglwx.cn/46208.Doc
m.edp.mglwx.cn/97951.Doc
m.edp.mglwx.cn/13339.Doc
m.edp.mglwx.cn/00604.Doc
m.edp.mglwx.cn/53339.Doc
m.edp.mglwx.cn/82040.Doc
m.edp.mglwx.cn/31157.Doc
m.edp.mglwx.cn/93377.Doc
m.edp.mglwx.cn/77953.Doc
m.edp.mglwx.cn/13973.Doc
m.edp.mglwx.cn/24826.Doc
m.edp.mglwx.cn/13919.Doc
m.edp.mglwx.cn/00602.Doc
m.edo.mglwx.cn/57171.Doc
m.edo.mglwx.cn/59771.Doc
m.edo.mglwx.cn/57535.Doc
m.edo.mglwx.cn/99511.Doc
m.edo.mglwx.cn/57753.Doc
m.edo.mglwx.cn/42242.Doc
m.edo.mglwx.cn/77577.Doc
m.edo.mglwx.cn/15519.Doc
m.edo.mglwx.cn/55331.Doc
m.edo.mglwx.cn/59577.Doc
m.edo.mglwx.cn/24280.Doc
m.edo.mglwx.cn/79717.Doc
m.edo.mglwx.cn/95595.Doc
m.edo.mglwx.cn/35377.Doc
m.edo.mglwx.cn/31135.Doc
m.edo.mglwx.cn/97551.Doc
m.edo.mglwx.cn/40002.Doc
m.edo.mglwx.cn/33797.Doc
m.edo.mglwx.cn/97919.Doc
m.edo.mglwx.cn/82028.Doc
m.edi.mglwx.cn/68602.Doc
m.edi.mglwx.cn/57511.Doc
m.edi.mglwx.cn/22282.Doc
m.edi.mglwx.cn/59937.Doc
m.edi.mglwx.cn/17777.Doc
m.edi.mglwx.cn/99119.Doc
m.edi.mglwx.cn/46860.Doc
m.edi.mglwx.cn/62804.Doc
m.edi.mglwx.cn/40444.Doc
m.edi.mglwx.cn/57135.Doc
m.edi.mglwx.cn/40242.Doc
m.edi.mglwx.cn/31571.Doc
m.edi.mglwx.cn/86240.Doc
m.edi.mglwx.cn/11991.Doc
m.edi.mglwx.cn/39971.Doc
m.edi.mglwx.cn/73513.Doc
m.edi.mglwx.cn/93113.Doc
m.edi.mglwx.cn/68008.Doc
m.edi.mglwx.cn/31377.Doc
m.edi.mglwx.cn/26446.Doc
m.edu.mglwx.cn/60648.Doc
m.edu.mglwx.cn/91753.Doc
m.edu.mglwx.cn/55931.Doc
m.edu.mglwx.cn/91157.Doc
m.edu.mglwx.cn/44408.Doc
m.edu.mglwx.cn/95957.Doc
m.edu.mglwx.cn/35993.Doc
m.edu.mglwx.cn/80624.Doc
m.edu.mglwx.cn/20602.Doc
m.edu.mglwx.cn/33759.Doc
m.edu.mglwx.cn/64006.Doc
m.edu.mglwx.cn/59393.Doc
m.edu.mglwx.cn/77911.Doc
m.edu.mglwx.cn/35591.Doc
m.edu.mglwx.cn/35319.Doc
m.edu.mglwx.cn/77171.Doc
m.edu.mglwx.cn/57515.Doc
m.edu.mglwx.cn/80644.Doc
m.edu.mglwx.cn/33315.Doc
m.edu.mglwx.cn/44060.Doc
m.edy.mglwx.cn/19593.Doc
m.edy.mglwx.cn/95933.Doc
m.edy.mglwx.cn/24422.Doc
m.edy.mglwx.cn/93599.Doc
m.edy.mglwx.cn/40264.Doc
m.edy.mglwx.cn/13339.Doc
m.edy.mglwx.cn/64480.Doc
m.edy.mglwx.cn/93971.Doc
m.edy.mglwx.cn/22260.Doc
m.edy.mglwx.cn/62680.Doc
m.edy.mglwx.cn/77175.Doc
m.edy.mglwx.cn/79979.Doc
m.edy.mglwx.cn/95737.Doc
m.edy.mglwx.cn/15579.Doc
m.edy.mglwx.cn/31751.Doc
m.edy.mglwx.cn/57159.Doc
m.edy.mglwx.cn/22262.Doc
m.edy.mglwx.cn/80806.Doc
m.edy.mglwx.cn/62044.Doc
m.edy.mglwx.cn/20208.Doc
m.edt.mglwx.cn/62846.Doc
m.edt.mglwx.cn/86460.Doc
m.edt.mglwx.cn/15575.Doc
m.edt.mglwx.cn/53951.Doc
m.edt.mglwx.cn/13559.Doc
m.edt.mglwx.cn/48082.Doc
m.edt.mglwx.cn/15931.Doc
m.edt.mglwx.cn/19797.Doc
m.edt.mglwx.cn/19751.Doc
m.edt.mglwx.cn/44804.Doc
