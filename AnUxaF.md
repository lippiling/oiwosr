============================================
pytest 钩子函数与插件体系深入解析
============================================

pytest 的钩子（Hook）系统允许我们在测试生命周期的
特定阶段插入自定义逻辑，是扩展 pytest 功能的核心机制。

一、pytest 测试生命周期
-----------------------

测试执行的主要阶段：
1. 配置阶段：pytest_configure
2. 收集阶段：pytest_collection_modifyitems
3. 执行阶段：pytest_runtest_setup/call/teardown
4. 报告阶段：pytest_terminal_summary

二、代码示例
------------

import pytest
import time
import os


# =============================================
# 第一部分：在 conftest.py 中实现钩子函数
# =============================================
# 以下钩子函数通常写在 conftest.py 中，会自动生效


def pytest_configure(config):
    """
    钩子1：pytest 配置完成后调用
    用于注册自定义标记、初始化全局资源
    """
    # 注册自定义标记，避免使用未注册标记时的警告
    config.addinivalue_line(
        "markers",
        "slow: 标记为慢速测试，可用 -m 'not slow' 跳过"
    )
    config.addinivalue_line(
        "markers",
        "integration: 集成测试标记，需要外部服务"
    )
    config.addinivalue_line(
        "markers",
        "smoke: 冒烟测试，快速验证核心功能"
    )
    print("\n[pytest_configure] 配置完成，已注册自定义标记")


def pytest_collection_modifyitems(config, items):
    """
    钩子2：测试用例收集完成后调用
    用于修改、排序、过滤测试用例
    """
    print(f"\n[pytest_collection_modifyitems] 共收集 {len(items)} 个测试")

    # 将标记了 slow 的测试移到末尾执行
    items.sort(key=lambda item: "slow" in item.keywords)

    # 为所有测试添加自定义属性
    for item in items:
        item.user_properties.append(("收集时间", time.strftime("%H:%M:%S")))

    print("[pytest_collection_modifyitems] 测试排序完成")


def pytest_runtest_setup(item):
    """
    钩子3：每个测试用例执行前调用
    用于前置条件检查、动态跳过
    """
    print(f"\n[pytest_runtest_setup] 准备执行: {item.name}")

    # 检查环境条件，不满足则跳过
    if "integration" in item.keywords:
        if not os.environ.get("INTEGRATION_TEST"):
            pytest.skip("需要设置 INTEGRATION_TEST 环境变量")
    print(f"[pytest_runtest_setup] {item.name} 前置条件满足")


def pytest_runtest_teardown(item, nextitem):
    """
    钩子4：每个测试用例执行后调用
    用于清理资源、记录日志
    """
    print(f"\n[pytest_runtest_teardown] 清理: {item.name}")
    if nextitem:
        print(f"[pytest_runtest_teardown] 下一个测试: {nextitem.name}")


def pytest_terminal_summary(terminalreporter, exitstatus, config):
    """
    钩子5：全部测试执行完毕后调用
    用于生成自定义摘要报告、统计信息
    """
    print("\n" + "=" * 50)
    print("[pytest_terminal_summary] 自定义测试总结报告")
    print("=" * 50)

    # 获取测试结果统计
    passed = len(terminalreporter.stats.get("passed", []))
    failed = len(terminalreporter.stats.get("failed", []))
    skipped = len(terminalreporter.stats.get("skipped", []))
    error = len(terminalreporter.stats.get("error", []))
    total = passed + failed + skipped + error

    print(f"总用例数: {total}")
    print(f"通过: {passed}")
    print(f"失败: {failed}")
    print(f"跳过: {skipped}")
    print(f"错误: {error}")

    if failed > 0:
        print("\n--- 失败用例详情 ---")
        for test in terminalreporter.stats.get("failed", []):
            print(f"  - {test.nodeid}")
            print(f"    错误: {test.longreprtext.split(chr(10))[-1]}")

    # 计算通过率
    if total > 0:
        pass_rate = (passed / total) * 100
        print(f"\n通过率: {pass_rate:.1f}%")

        if pass_rate < 80:
            print("警告：通过率低于 80%，请检查代码质量！")


# =============================================
# 第二部分：使用流行的 pytest 插件
# =============================================
# 1. pytest-cov：测试覆盖率
# 安装：pip install pytest-cov
# 运行：pytest --cov=myproject --cov-report=html


def test_coverage_plugin_example():
    """
    使用 pytest-cov 后，运行测试时添加参数：
    pytest --cov=. --cov-report=term-missing
    会显示每行代码的覆盖情况
    """
    result = sum([1, 2, 3, 4, 5])
    assert result == 15


# 2. pytest-xdist：并行执行测试
# 安装：pip install pytest-xdist
# 运行：pytest -n auto           (自动检测 CPU 核心数)
# 运行：pytest -n 4              (使用 4 个 worker)


@pytest.mark.slow
def test_slow_operation():
    """模拟慢速测试，xdist 可以并行加速此类测试"""
    time.sleep(0.5)
    assert True


# 3. pytest-timeout：测试超时控制
# 安装：pip install pytest-timeout
# 运行：pytest --timeout=30      (全局超时 30 秒)
@pytest.mark.timeout(5)  # 单个测试超时 5 秒
def test_with_timeout():
    """超时控制在防止测试死循环时非常有用"""
    result = 2 + 2
    assert result == 4


# =============================================
# 第三部分：编写自定义 pytest 插件
# =============================================


class RepeatPlugin:
    """
    自定义插件：允许重复执行测试
    使用方式：pytest --repeat 3 test_file.py
    """

    def __init__(self):
        self.repeat_count = 1

    def pytest_addoption(self, parser):
        """添加命令行选项"""
        parser.addoption(
            "--repeat",
            action="store",
            default=1,
            type=int,
            help="指定测试重复执行次数"
        )

    def pytest_configure(self, config):
        """读取命令行选项值"""
        self.repeat_count = config.getoption("--repeat")
        if self.repeat_count > 1:
            print(f"\n[RepeatPlugin] 测试将重复执行 {self.repeat_count} 次")

    def pytest_runtest_protocol(self, item, nextitem):
        """
        拦截测试执行协议，实现重复执行
        """
        if self.repeat_count <= 1:
            return None  # 交给默认处理

        # 手动重复执行测试
        for i in range(self.repeat_count):
            print(f"\n[RepeatPlugin] 第 {i+1}/{self.repeat_count} 次执行")
            item.config.hook.pytest_runtest_logstart(
                nodeid=item.nodeid, location=item.location
            )
            reports = self.run_test(item)
            for report in reports:
                item.config.hook.pytest_runtest_logreport(report=report)

        # 返回非 None 表示已处理，阻止默认执行
        return True

    def run_test(self, item):
        """执行单次测试并收集报告"""
        from _pytest.runner import runtestprotocol
        return runtestprotocol(item, log=False)


class TimingPlugin:
    """
    自定义插件：记录每个测试的执行时间
    """

    def __init__(self):
        self.timings = {}

    def pytest_runtest_setup(self, item):
        """记录测试开始时间"""
        self.timings[item.nodeid] = {"start": time.time()}

    def pytest_runtest_teardown(self, item):
        """计算并记录测试耗时"""
        if item.nodeid in self.timings:
            elapsed = time.time() - self.timings[item.nodeid]["start"]
            self.timings[item.nodeid]["elapsed"] = elapsed

    def pytest_terminal_summary(self, terminalreporter):
        """在终端输出最慢的测试"""
        print("\n" + "=" * 50)
        print("[TimingPlugin] 测试执行时间排名（Top 5）")
        print("=" * 50)

        # 按耗时排序
        sorted_tests = sorted(
            self.timings.items(),
            key=lambda x: x[1].get("elapsed", 0),
            reverse=True
        )

        for nodeid, data in sorted_tests[:5]:
            elapsed = data.get("elapsed", 0)
            print(f"  {elapsed:.3f}s  -  {nodeid}")


# =============================================
# 第四部分：插件注册方式
# =============================================

# 方式1：在 conftest.py 中注册
"""
# conftest.py
pytest_plugins = [
    "pytest_cov",
    "pytest_xdist",
    "my_custom_plugin",
]
"""

# 方式2：通过 entry points 注册（setup.cfg / pyproject.toml）
"""
# setup.cfg
[options.entry_points]
pytest11 =
    my_plugin = my_package.my_plugin

# pyproject.toml
[project.entry-points."pytest11"]
my_plugin = "my_package.my_plugin"
"""

# 方式3：在测试文件中直接使用
# pytest_plugins = ["pytest_cov", "my_plugin"]


# 测试自定义插件的功能
@pytest.mark.smoke
def test_smoke():
    """冒烟测试示例"""
    assert True


@pytest.mark.slow
def test_heavy_computation():
    """慢速测试示例"""
    time.sleep(0.1)
    result = sum(range(1000))
    assert result == 499500

wvd.wfuyuu27.cn/24488.Doc
wvd.wfuyuu27.cn/04208.Doc
wvd.wfuyuu27.cn/20888.Doc
wvd.wfuyuu27.cn/22264.Doc
wvd.wfuyuu27.cn/24828.Doc
wvd.wfuyuu27.cn/62626.Doc
wvd.wfuyuu27.cn/79733.Doc
wvd.wfuyuu27.cn/19995.Doc
wvd.wfuyuu27.cn/44424.Doc
wvd.wfuyuu27.cn/68046.Doc
wvs.wfuyuu27.cn/48266.Doc
wvs.wfuyuu27.cn/48804.Doc
wvs.wfuyuu27.cn/26666.Doc
wvs.wfuyuu27.cn/31571.Doc
wvs.wfuyuu27.cn/62846.Doc
wvs.wfuyuu27.cn/82402.Doc
wvs.wfuyuu27.cn/84000.Doc
wvs.wfuyuu27.cn/80228.Doc
wvs.wfuyuu27.cn/51537.Doc
wvs.wfuyuu27.cn/00200.Doc
wva.wfuyuu27.cn/40086.Doc
wva.wfuyuu27.cn/20686.Doc
wva.wfuyuu27.cn/04242.Doc
wva.wfuyuu27.cn/04628.Doc
wva.wfuyuu27.cn/80422.Doc
wva.wfuyuu27.cn/24422.Doc
wva.wfuyuu27.cn/84280.Doc
wva.wfuyuu27.cn/62204.Doc
wva.wfuyuu27.cn/04488.Doc
wva.wfuyuu27.cn/08282.Doc
wvp.wfuyuu27.cn/80024.Doc
wvp.wfuyuu27.cn/80408.Doc
wvp.wfuyuu27.cn/80022.Doc
wvp.wfuyuu27.cn/64608.Doc
wvp.wfuyuu27.cn/86460.Doc
wvp.wfuyuu27.cn/75557.Doc
wvp.wfuyuu27.cn/06280.Doc
wvp.wfuyuu27.cn/00642.Doc
wvp.wfuyuu27.cn/06666.Doc
wvp.wfuyuu27.cn/84488.Doc
wvo.wfuyuu27.cn/08620.Doc
wvo.wfuyuu27.cn/64060.Doc
wvo.wfuyuu27.cn/93799.Doc
wvo.wfuyuu27.cn/08848.Doc
wvo.wfuyuu27.cn/08446.Doc
wvo.wfuyuu27.cn/48860.Doc
wvo.wfuyuu27.cn/26842.Doc
wvo.wfuyuu27.cn/22266.Doc
wvo.wfuyuu27.cn/62888.Doc
wvo.wfuyuu27.cn/42884.Doc
wvi.wfuyuu27.cn/62608.Doc
wvi.wfuyuu27.cn/02602.Doc
wvi.wfuyuu27.cn/08888.Doc
wvi.wfuyuu27.cn/68444.Doc
wvi.wfuyuu27.cn/00842.Doc
wvi.wfuyuu27.cn/48424.Doc
wvi.wfuyuu27.cn/68064.Doc
wvi.wfuyuu27.cn/80666.Doc
wvi.wfuyuu27.cn/62626.Doc
wvi.wfuyuu27.cn/00208.Doc
wvu.wfuyuu27.cn/66226.Doc
wvu.wfuyuu27.cn/88224.Doc
wvu.wfuyuu27.cn/17915.Doc
wvu.wfuyuu27.cn/86640.Doc
wvu.wfuyuu27.cn/82846.Doc
wvu.wfuyuu27.cn/42062.Doc
wvu.wfuyuu27.cn/46226.Doc
wvu.wfuyuu27.cn/28440.Doc
wvu.wfuyuu27.cn/28208.Doc
wvu.wfuyuu27.cn/62808.Doc
wvy.wfuyuu27.cn/00660.Doc
wvy.wfuyuu27.cn/46628.Doc
wvy.wfuyuu27.cn/84222.Doc
wvy.wfuyuu27.cn/84648.Doc
wvy.wfuyuu27.cn/11535.Doc
wvy.wfuyuu27.cn/48024.Doc
wvy.wfuyuu27.cn/60620.Doc
wvy.wfuyuu27.cn/88640.Doc
wvy.wfuyuu27.cn/84408.Doc
wvy.wfuyuu27.cn/24260.Doc
wvt.wfuyuu27.cn/48046.Doc
wvt.wfuyuu27.cn/24424.Doc
wvt.wfuyuu27.cn/68428.Doc
wvt.wfuyuu27.cn/42226.Doc
wvt.wfuyuu27.cn/02880.Doc
wvt.wfuyuu27.cn/80006.Doc
wvt.wfuyuu27.cn/42826.Doc
wvt.wfuyuu27.cn/80420.Doc
wvt.wfuyuu27.cn/80048.Doc
wvt.wfuyuu27.cn/42026.Doc
wvr.wfuyuu27.cn/00028.Doc
wvr.wfuyuu27.cn/88824.Doc
wvr.wfuyuu27.cn/46024.Doc
wvr.wfuyuu27.cn/08868.Doc
wvr.wfuyuu27.cn/11559.Doc
wvr.wfuyuu27.cn/04200.Doc
wvr.wfuyuu27.cn/26424.Doc
wvr.wfuyuu27.cn/22402.Doc
wvr.wfuyuu27.cn/80208.Doc
wvr.wfuyuu27.cn/60840.Doc
wve.wfuyuu27.cn/46446.Doc
wve.wfuyuu27.cn/88602.Doc
wve.wfuyuu27.cn/08426.Doc
wve.wfuyuu27.cn/28642.Doc
wve.wfuyuu27.cn/79791.Doc
wve.wfuyuu27.cn/62202.Doc
wve.wfuyuu27.cn/68002.Doc
wve.wfuyuu27.cn/26240.Doc
wve.wfuyuu27.cn/82626.Doc
wve.wfuyuu27.cn/40840.Doc
wvw.wfuyuu27.cn/08204.Doc
wvw.wfuyuu27.cn/80820.Doc
wvw.wfuyuu27.cn/22880.Doc
wvw.wfuyuu27.cn/04402.Doc
wvw.wfuyuu27.cn/28668.Doc
wvw.wfuyuu27.cn/33115.Doc
wvw.wfuyuu27.cn/59995.Doc
wvw.wfuyuu27.cn/62802.Doc
wvw.wfuyuu27.cn/00882.Doc
wvw.wfuyuu27.cn/88642.Doc
wvq.wfuyuu27.cn/64248.Doc
wvq.wfuyuu27.cn/24864.Doc
wvq.wfuyuu27.cn/20228.Doc
wvq.wfuyuu27.cn/02286.Doc
wvq.wfuyuu27.cn/37739.Doc
wvq.wfuyuu27.cn/82084.Doc
wvq.wfuyuu27.cn/80082.Doc
wvq.wfuyuu27.cn/28620.Doc
wvq.wfuyuu27.cn/48426.Doc
wvq.wfuyuu27.cn/46848.Doc
wcm.wfuyuu27.cn/13777.Doc
wcm.wfuyuu27.cn/39755.Doc
wcm.wfuyuu27.cn/26000.Doc
wcm.wfuyuu27.cn/02820.Doc
wcm.wfuyuu27.cn/97797.Doc
wcm.wfuyuu27.cn/08462.Doc
wcm.wfuyuu27.cn/62866.Doc
wcm.wfuyuu27.cn/00400.Doc
wcm.wfuyuu27.cn/88668.Doc
wcm.wfuyuu27.cn/68406.Doc
wcn.wfuyuu27.cn/48024.Doc
wcn.wfuyuu27.cn/86806.Doc
wcn.wfuyuu27.cn/66606.Doc
wcn.wfuyuu27.cn/06002.Doc
wcn.wfuyuu27.cn/26820.Doc
wcn.wfuyuu27.cn/20660.Doc
wcn.wfuyuu27.cn/66006.Doc
wcn.wfuyuu27.cn/44646.Doc
wcn.wfuyuu27.cn/02428.Doc
wcn.wfuyuu27.cn/20880.Doc
wcb.wfuyuu27.cn/42644.Doc
wcb.wfuyuu27.cn/60464.Doc
wcb.wfuyuu27.cn/66406.Doc
wcb.wfuyuu27.cn/02446.Doc
wcb.wfuyuu27.cn/66884.Doc
wcb.wfuyuu27.cn/24486.Doc
wcb.wfuyuu27.cn/08466.Doc
wcb.wfuyuu27.cn/28804.Doc
wcb.wfuyuu27.cn/68242.Doc
wcb.wfuyuu27.cn/62068.Doc
wcv.wfuyuu27.cn/00622.Doc
wcv.wfuyuu27.cn/39313.Doc
wcv.wfuyuu27.cn/08886.Doc
wcv.wfuyuu27.cn/08666.Doc
wcv.wfuyuu27.cn/28844.Doc
wcv.wfuyuu27.cn/66248.Doc
wcv.wfuyuu27.cn/64026.Doc
wcv.wfuyuu27.cn/13391.Doc
wcv.wfuyuu27.cn/28420.Doc
wcv.wfuyuu27.cn/42844.Doc
wcc.wfuyuu27.cn/88646.Doc
wcc.wfuyuu27.cn/06460.Doc
wcc.wfuyuu27.cn/99715.Doc
wcc.wfuyuu27.cn/26080.Doc
wcc.wfuyuu27.cn/62020.Doc
wcc.wfuyuu27.cn/02280.Doc
wcc.wfuyuu27.cn/20024.Doc
wcc.wfuyuu27.cn/40222.Doc
wcc.wfuyuu27.cn/00842.Doc
wcc.wfuyuu27.cn/88464.Doc
wcx.wfuyuu27.cn/68264.Doc
wcx.wfuyuu27.cn/24426.Doc
wcx.wfuyuu27.cn/00804.Doc
wcx.wfuyuu27.cn/40828.Doc
wcx.wfuyuu27.cn/22266.Doc
wcx.wfuyuu27.cn/48060.Doc
wcx.wfuyuu27.cn/91137.Doc
wcx.wfuyuu27.cn/40482.Doc
wcx.wfuyuu27.cn/88802.Doc
wcx.wfuyuu27.cn/62408.Doc
wcz.wfuyuu27.cn/64008.Doc
wcz.wfuyuu27.cn/00680.Doc
wcz.wfuyuu27.cn/60224.Doc
wcz.wfuyuu27.cn/28624.Doc
wcz.wfuyuu27.cn/26868.Doc
wcz.wfuyuu27.cn/42688.Doc
wcz.wfuyuu27.cn/68802.Doc
wcz.wfuyuu27.cn/20806.Doc
wcz.wfuyuu27.cn/08044.Doc
wcz.wfuyuu27.cn/84444.Doc
wcl.wfuyuu27.cn/44806.Doc
wcl.wfuyuu27.cn/28000.Doc
wcl.wfuyuu27.cn/26000.Doc
wcl.wfuyuu27.cn/48268.Doc
wcl.wfuyuu27.cn/62026.Doc
wcl.wfuyuu27.cn/80008.Doc
wcl.wfuyuu27.cn/77351.Doc
wcl.wfuyuu27.cn/04800.Doc
wcl.wfuyuu27.cn/28884.Doc
wcl.wfuyuu27.cn/08860.Doc
wck.wfuyuu27.cn/86426.Doc
wck.wfuyuu27.cn/66404.Doc
wck.wfuyuu27.cn/68608.Doc
wck.wfuyuu27.cn/06066.Doc
wck.wfuyuu27.cn/46022.Doc
wck.wfuyuu27.cn/86268.Doc
wck.wfuyuu27.cn/26222.Doc
wck.wfuyuu27.cn/42648.Doc
wck.wfuyuu27.cn/71759.Doc
wck.wfuyuu27.cn/28806.Doc
wcj.wfuyuu27.cn/86840.Doc
wcj.wfuyuu27.cn/66868.Doc
wcj.wfuyuu27.cn/06006.Doc
wcj.wfuyuu27.cn/26426.Doc
wcj.wfuyuu27.cn/00004.Doc
wcj.wfuyuu27.cn/24686.Doc
wcj.wfuyuu27.cn/22082.Doc
wcj.wfuyuu27.cn/40488.Doc
wcj.wfuyuu27.cn/82880.Doc
wcj.wfuyuu27.cn/84468.Doc
wch.wfuyuu27.cn/40208.Doc
wch.wfuyuu27.cn/88482.Doc
wch.wfuyuu27.cn/02808.Doc
wch.wfuyuu27.cn/48660.Doc
wch.wfuyuu27.cn/42266.Doc
wch.wfuyuu27.cn/66240.Doc
wch.wfuyuu27.cn/28268.Doc
wch.wfuyuu27.cn/77373.Doc
wch.wfuyuu27.cn/28440.Doc
wch.wfuyuu27.cn/00448.Doc
wcg.wfuyuu27.cn/13557.Doc
wcg.wfuyuu27.cn/60820.Doc
wcg.wfuyuu27.cn/59775.Doc
wcg.wfuyuu27.cn/06288.Doc
wcg.wfuyuu27.cn/08884.Doc
wcg.wfuyuu27.cn/22422.Doc
wcg.wfuyuu27.cn/24286.Doc
wcg.wfuyuu27.cn/60820.Doc
wcg.wfuyuu27.cn/48628.Doc
wcg.wfuyuu27.cn/06828.Doc
wcf.wfuyuu27.cn/26268.Doc
wcf.wfuyuu27.cn/20680.Doc
wcf.wfuyuu27.cn/06020.Doc
wcf.wfuyuu27.cn/80224.Doc
wcf.wfuyuu27.cn/80268.Doc
wcf.wfuyuu27.cn/64626.Doc
wcf.wfuyuu27.cn/22600.Doc
wcf.wfuyuu27.cn/84464.Doc
wcf.wfuyuu27.cn/00202.Doc
wcf.wfuyuu27.cn/28240.Doc
wcd.wfuyuu27.cn/84464.Doc
wcd.wfuyuu27.cn/02622.Doc
wcd.wfuyuu27.cn/80862.Doc
wcd.wfuyuu27.cn/28680.Doc
wcd.wfuyuu27.cn/99777.Doc
wcd.wfuyuu27.cn/93715.Doc
wcd.wfuyuu27.cn/88608.Doc
wcd.wfuyuu27.cn/62446.Doc
wcd.wfuyuu27.cn/88266.Doc
wcd.wfuyuu27.cn/13335.Doc
wcs.wfuyuu27.cn/40886.Doc
wcs.wfuyuu27.cn/06288.Doc
wcs.wfuyuu27.cn/88824.Doc
wcs.wfuyuu27.cn/46624.Doc
wcs.wfuyuu27.cn/00400.Doc
wcs.wfuyuu27.cn/20206.Doc
wcs.wfuyuu27.cn/22406.Doc
wcs.wfuyuu27.cn/82288.Doc
wcs.wfuyuu27.cn/88268.Doc
wcs.wfuyuu27.cn/75377.Doc
wca.wfuyuu27.cn/24060.Doc
wca.wfuyuu27.cn/82242.Doc
wca.wfuyuu27.cn/26862.Doc
wca.wfuyuu27.cn/64680.Doc
wca.wfuyuu27.cn/02682.Doc
wca.wfuyuu27.cn/00660.Doc
wca.wfuyuu27.cn/40884.Doc
wca.wfuyuu27.cn/82486.Doc
wca.wfuyuu27.cn/00040.Doc
wca.wfuyuu27.cn/46002.Doc
wcp.wfuyuu27.cn/26242.Doc
wcp.wfuyuu27.cn/88006.Doc
wcp.wfuyuu27.cn/26400.Doc
wcp.wfuyuu27.cn/88686.Doc
wcp.wfuyuu27.cn/44422.Doc
wcp.wfuyuu27.cn/82806.Doc
wcp.wfuyuu27.cn/24804.Doc
wcp.wfuyuu27.cn/59391.Doc
wcp.wfuyuu27.cn/46820.Doc
wcp.wfuyuu27.cn/48040.Doc
wco.wfuyuu27.cn/48442.Doc
wco.wfuyuu27.cn/48400.Doc
wco.wfuyuu27.cn/42080.Doc
wco.wfuyuu27.cn/66668.Doc
wco.wfuyuu27.cn/48000.Doc
wco.wfuyuu27.cn/88626.Doc
wco.wfuyuu27.cn/80486.Doc
wco.wfuyuu27.cn/99139.Doc
wco.wfuyuu27.cn/44206.Doc
wco.wfuyuu27.cn/40806.Doc
wci.wfuyuu27.cn/88242.Doc
wci.wfuyuu27.cn/48626.Doc
wci.wfuyuu27.cn/88866.Doc
wci.wfuyuu27.cn/28822.Doc
wci.wfuyuu27.cn/48822.Doc
wci.wfuyuu27.cn/88088.Doc
wci.wfuyuu27.cn/20620.Doc
wci.wfuyuu27.cn/40800.Doc
wci.wfuyuu27.cn/42246.Doc
wci.wfuyuu27.cn/68246.Doc
wcu.wfuyuu27.cn/41156.Doc
wcu.wfuyuu27.cn/31248.Doc
wcu.wfuyuu27.cn/82289.Doc
wcu.wfuyuu27.cn/85577.Doc
wcu.wfuyuu27.cn/02653.Doc
wcu.wfuyuu27.cn/46000.Doc
wcu.wfuyuu27.cn/52387.Doc
wcu.wfuyuu27.cn/97705.Doc
wcu.wfuyuu27.cn/00522.Doc
wcu.wfuyuu27.cn/27127.Doc
wcy.wfuyuu27.cn/95550.Doc
wcy.wfuyuu27.cn/86078.Doc
wcy.wfuyuu27.cn/02324.Doc
wcy.wfuyuu27.cn/08194.Doc
wcy.wfuyuu27.cn/17293.Doc
wcy.wfuyuu27.cn/38330.Doc
wcy.wfuyuu27.cn/58938.Doc
wcy.wfuyuu27.cn/37941.Doc
wcy.wfuyuu27.cn/20002.Doc
wcy.wfuyuu27.cn/66574.Doc
wct.wfuyuu27.cn/80078.Doc
wct.wfuyuu27.cn/93285.Doc
wct.wfuyuu27.cn/68625.Doc
wct.wfuyuu27.cn/67078.Doc
wct.wfuyuu27.cn/71759.Doc
wct.wfuyuu27.cn/92833.Doc
wct.wfuyuu27.cn/46635.Doc
wct.wfuyuu27.cn/92938.Doc
wct.wfuyuu27.cn/26646.Doc
wct.wfuyuu27.cn/01310.Doc
wcr.wfuyuu27.cn/67437.Doc
wcr.wfuyuu27.cn/92195.Doc
wcr.wfuyuu27.cn/84419.Doc
wcr.wfuyuu27.cn/75393.Doc
wcr.wfuyuu27.cn/00701.Doc
wcr.wfuyuu27.cn/70510.Doc
wcr.wfuyuu27.cn/12991.Doc
wcr.wfuyuu27.cn/04836.Doc
wcr.wfuyuu27.cn/66876.Doc
wcr.wfuyuu27.cn/65743.Doc
wce.wfuyuu27.cn/96780.Doc
wce.wfuyuu27.cn/16105.Doc
wce.wfuyuu27.cn/51379.Doc
wce.wfuyuu27.cn/26020.Doc
wce.wfuyuu27.cn/19442.Doc
wce.wfuyuu27.cn/04205.Doc
wce.wfuyuu27.cn/62293.Doc
wce.wfuyuu27.cn/84335.Doc
wce.wfuyuu27.cn/46024.Doc
wce.wfuyuu27.cn/24446.Doc
wcw.wfuyuu27.cn/00662.Doc
wcw.wfuyuu27.cn/42026.Doc
wcw.wfuyuu27.cn/24800.Doc
wcw.wfuyuu27.cn/62886.Doc
wcw.wfuyuu27.cn/24848.Doc
wcw.wfuyuu27.cn/82048.Doc
wcw.wfuyuu27.cn/08220.Doc
wcw.wfuyuu27.cn/88022.Doc
wcw.wfuyuu27.cn/68686.Doc
wcw.wfuyuu27.cn/42884.Doc
wcq.wfuyuu27.cn/24286.Doc
wcq.wfuyuu27.cn/24240.Doc
wcq.wfuyuu27.cn/26604.Doc
wcq.wfuyuu27.cn/46248.Doc
wcq.wfuyuu27.cn/04446.Doc
wcq.wfuyuu27.cn/22404.Doc
wcq.wfuyuu27.cn/82022.Doc
wcq.wfuyuu27.cn/28486.Doc
wcq.wfuyuu27.cn/40000.Doc
wcq.wfuyuu27.cn/84844.Doc
wxm.wfuyuu27.cn/00246.Doc
wxm.wfuyuu27.cn/20402.Doc
wxm.wfuyuu27.cn/08082.Doc
wxm.wfuyuu27.cn/06208.Doc
wxm.wfuyuu27.cn/62480.Doc
wxm.wfuyuu27.cn/84840.Doc
wxm.wfuyuu27.cn/46000.Doc
wxm.wfuyuu27.cn/82802.Doc
wxm.wfuyuu27.cn/60260.Doc
wxm.wfuyuu27.cn/68408.Doc
wxn.wfuyuu27.cn/06882.Doc
wxn.wfuyuu27.cn/06406.Doc
wxn.wfuyuu27.cn/42648.Doc
wxn.wfuyuu27.cn/02880.Doc
wxn.wfuyuu27.cn/08828.Doc
wxn.wfuyuu27.cn/46660.Doc
wxn.wfuyuu27.cn/84028.Doc
wxn.wfuyuu27.cn/80486.Doc
wxn.wfuyuu27.cn/84828.Doc
wxn.wfuyuu27.cn/80664.Doc
wxb.wfuyuu27.cn/42066.Doc
wxb.wfuyuu27.cn/00468.Doc
wxb.wfuyuu27.cn/00462.Doc
wxb.wfuyuu27.cn/02688.Doc
wxb.wfuyuu27.cn/60486.Doc
wxb.wfuyuu27.cn/60042.Doc
wxb.wfuyuu27.cn/20442.Doc
wxb.wfuyuu27.cn/42888.Doc
wxb.wfuyuu27.cn/40286.Doc
wxb.wfuyuu27.cn/66448.Doc
wxv.wfuyuu27.cn/77751.Doc
wxv.wfuyuu27.cn/40422.Doc
wxv.wfuyuu27.cn/68666.Doc
wxv.wfuyuu27.cn/24268.Doc
wxv.wfuyuu27.cn/68428.Doc
wxv.wfuyuu27.cn/24624.Doc
wxv.wfuyuu27.cn/80606.Doc
wxv.wfuyuu27.cn/86800.Doc
wxv.wfuyuu27.cn/42284.Doc
wxv.wfuyuu27.cn/60462.Doc
wxc.wfuyuu27.cn/00224.Doc
wxc.wfuyuu27.cn/66202.Doc
wxc.wfuyuu27.cn/00064.Doc
wxc.wfuyuu27.cn/04268.Doc
wxc.wfuyuu27.cn/00080.Doc
wxc.wfuyuu27.cn/02644.Doc
wxc.wfuyuu27.cn/02068.Doc
wxc.wfuyuu27.cn/80242.Doc
wxc.wfuyuu27.cn/19757.Doc
wxc.wfuyuu27.cn/82600.Doc
wxx.wfuyuu27.cn/02424.Doc
wxx.wfuyuu27.cn/02406.Doc
wxx.wfuyuu27.cn/26020.Doc
wxx.wfuyuu27.cn/64664.Doc
wxx.wfuyuu27.cn/66884.Doc
wxx.wfuyuu27.cn/06440.Doc
wxx.wfuyuu27.cn/02288.Doc
wxx.wfuyuu27.cn/48046.Doc
wxx.wfuyuu27.cn/84086.Doc
wxx.wfuyuu27.cn/42688.Doc
wxz.wfuyuu27.cn/42066.Doc
wxz.wfuyuu27.cn/08840.Doc
wxz.wfuyuu27.cn/20244.Doc
wxz.wfuyuu27.cn/44240.Doc
wxz.wfuyuu27.cn/88022.Doc
wxz.wfuyuu27.cn/44442.Doc
wxz.wfuyuu27.cn/64668.Doc
wxz.wfuyuu27.cn/86682.Doc
wxz.wfuyuu27.cn/06226.Doc
wxz.wfuyuu27.cn/80624.Doc
wxl.wfuyuu27.cn/11977.Doc
wxl.wfuyuu27.cn/88222.Doc
wxl.wfuyuu27.cn/28660.Doc
wxl.wfuyuu27.cn/22488.Doc
wxl.wfuyuu27.cn/24260.Doc
wxl.wfuyuu27.cn/48028.Doc
wxl.wfuyuu27.cn/20046.Doc
wxl.wfuyuu27.cn/68426.Doc
wxl.wfuyuu27.cn/06648.Doc
wxl.wfuyuu27.cn/48260.Doc
wxk.wfuyuu27.cn/66844.Doc
wxk.wfuyuu27.cn/19957.Doc
wxk.wfuyuu27.cn/42608.Doc
wxk.wfuyuu27.cn/06682.Doc
wxk.wfuyuu27.cn/62640.Doc
wxk.wfuyuu27.cn/64226.Doc
wxk.wfuyuu27.cn/26804.Doc
wxk.wfuyuu27.cn/84466.Doc
wxk.wfuyuu27.cn/80620.Doc
wxk.wfuyuu27.cn/48688.Doc
wxj.wfuyuu27.cn/24020.Doc
wxj.wfuyuu27.cn/28682.Doc
wxj.wfuyuu27.cn/84406.Doc
wxj.wfuyuu27.cn/06040.Doc
wxj.wfuyuu27.cn/22806.Doc
wxj.wfuyuu27.cn/42240.Doc
wxj.wfuyuu27.cn/48682.Doc
wxj.wfuyuu27.cn/80000.Doc
wxj.wfuyuu27.cn/42420.Doc
wxj.wfuyuu27.cn/04842.Doc
