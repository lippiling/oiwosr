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

qgg.zhuzhoujiudazxL1517.cn/86888.Doc
qgg.zhuzhoujiudazxL1517.cn/84682.Doc
qgg.zhuzhoujiudazxL1517.cn/55595.Doc
qgg.zhuzhoujiudazxL1517.cn/04844.Doc
qgg.zhuzhoujiudazxL1517.cn/80448.Doc
qgg.zhuzhoujiudazxL1517.cn/60044.Doc
qgg.zhuzhoujiudazxL1517.cn/88080.Doc
qgg.zhuzhoujiudazxL1517.cn/28228.Doc
qgg.zhuzhoujiudazxL1517.cn/88062.Doc
qgg.zhuzhoujiudazxL1517.cn/00428.Doc
qgf.zhuzhoujiudazxL1517.cn/80228.Doc
qgf.zhuzhoujiudazxL1517.cn/88426.Doc
qgf.zhuzhoujiudazxL1517.cn/00242.Doc
qgf.zhuzhoujiudazxL1517.cn/00224.Doc
qgf.zhuzhoujiudazxL1517.cn/64882.Doc
qgf.zhuzhoujiudazxL1517.cn/44648.Doc
qgf.zhuzhoujiudazxL1517.cn/06804.Doc
qgf.zhuzhoujiudazxL1517.cn/93511.Doc
qgf.zhuzhoujiudazxL1517.cn/62282.Doc
qgf.zhuzhoujiudazxL1517.cn/48842.Doc
qgd.zhuzhoujiudazxL1517.cn/53313.Doc
qgd.zhuzhoujiudazxL1517.cn/68244.Doc
qgd.zhuzhoujiudazxL1517.cn/02460.Doc
qgd.zhuzhoujiudazxL1517.cn/44400.Doc
qgd.zhuzhoujiudazxL1517.cn/46260.Doc
qgd.zhuzhoujiudazxL1517.cn/28862.Doc
qgd.zhuzhoujiudazxL1517.cn/88844.Doc
qgd.zhuzhoujiudazxL1517.cn/42660.Doc
qgd.zhuzhoujiudazxL1517.cn/00442.Doc
qgd.zhuzhoujiudazxL1517.cn/88602.Doc
qgs.zhuzhoujiudazxL1517.cn/82608.Doc
qgs.zhuzhoujiudazxL1517.cn/59971.Doc
qgs.zhuzhoujiudazxL1517.cn/24480.Doc
qgs.zhuzhoujiudazxL1517.cn/60686.Doc
qgs.zhuzhoujiudazxL1517.cn/22008.Doc
qgs.zhuzhoujiudazxL1517.cn/84448.Doc
qgs.zhuzhoujiudazxL1517.cn/06688.Doc
qgs.zhuzhoujiudazxL1517.cn/42642.Doc
qgs.zhuzhoujiudazxL1517.cn/24486.Doc
qgs.zhuzhoujiudazxL1517.cn/80844.Doc
qga.zhuzhoujiudazxL1517.cn/28462.Doc
qga.zhuzhoujiudazxL1517.cn/48042.Doc
qga.zhuzhoujiudazxL1517.cn/88048.Doc
qga.zhuzhoujiudazxL1517.cn/44084.Doc
qga.zhuzhoujiudazxL1517.cn/46442.Doc
qga.zhuzhoujiudazxL1517.cn/42644.Doc
qga.zhuzhoujiudazxL1517.cn/08628.Doc
qga.zhuzhoujiudazxL1517.cn/20880.Doc
qga.zhuzhoujiudazxL1517.cn/42684.Doc
qga.zhuzhoujiudazxL1517.cn/00064.Doc
qgp.zhuzhoujiudazxL1517.cn/26802.Doc
qgp.zhuzhoujiudazxL1517.cn/04068.Doc
qgp.zhuzhoujiudazxL1517.cn/80468.Doc
qgp.zhuzhoujiudazxL1517.cn/42288.Doc
qgp.zhuzhoujiudazxL1517.cn/20086.Doc
qgp.zhuzhoujiudazxL1517.cn/22604.Doc
qgp.zhuzhoujiudazxL1517.cn/04802.Doc
qgp.zhuzhoujiudazxL1517.cn/42224.Doc
qgp.zhuzhoujiudazxL1517.cn/26868.Doc
qgp.zhuzhoujiudazxL1517.cn/64642.Doc
qgo.zhuzhoujiudazxL1517.cn/46466.Doc
qgo.zhuzhoujiudazxL1517.cn/28686.Doc
qgo.zhuzhoujiudazxL1517.cn/08222.Doc
qgo.zhuzhoujiudazxL1517.cn/26686.Doc
qgo.zhuzhoujiudazxL1517.cn/08804.Doc
qgo.zhuzhoujiudazxL1517.cn/84202.Doc
qgo.zhuzhoujiudazxL1517.cn/26404.Doc
qgo.zhuzhoujiudazxL1517.cn/86088.Doc
qgo.zhuzhoujiudazxL1517.cn/48608.Doc
qgo.zhuzhoujiudazxL1517.cn/04224.Doc
qgi.zhuzhoujiudazxL1517.cn/60822.Doc
qgi.zhuzhoujiudazxL1517.cn/02404.Doc
qgi.zhuzhoujiudazxL1517.cn/86244.Doc
qgi.zhuzhoujiudazxL1517.cn/48864.Doc
qgi.zhuzhoujiudazxL1517.cn/86220.Doc
qgi.zhuzhoujiudazxL1517.cn/51759.Doc
qgi.zhuzhoujiudazxL1517.cn/40804.Doc
qgi.zhuzhoujiudazxL1517.cn/22686.Doc
qgi.zhuzhoujiudazxL1517.cn/28868.Doc
qgi.zhuzhoujiudazxL1517.cn/88802.Doc
qgu.zhuzhoujiudazxL1517.cn/04624.Doc
qgu.zhuzhoujiudazxL1517.cn/02682.Doc
qgu.zhuzhoujiudazxL1517.cn/68204.Doc
qgu.zhuzhoujiudazxL1517.cn/86224.Doc
qgu.zhuzhoujiudazxL1517.cn/97791.Doc
qgu.zhuzhoujiudazxL1517.cn/53379.Doc
qgu.zhuzhoujiudazxL1517.cn/62264.Doc
qgu.zhuzhoujiudazxL1517.cn/64040.Doc
qgu.zhuzhoujiudazxL1517.cn/62840.Doc
qgu.zhuzhoujiudazxL1517.cn/40020.Doc
qgy.zhuzhoujiudazxL1517.cn/64686.Doc
qgy.zhuzhoujiudazxL1517.cn/46824.Doc
qgy.zhuzhoujiudazxL1517.cn/46068.Doc
qgy.zhuzhoujiudazxL1517.cn/62200.Doc
qgy.zhuzhoujiudazxL1517.cn/00208.Doc
qgy.zhuzhoujiudazxL1517.cn/64460.Doc
qgy.zhuzhoujiudazxL1517.cn/06442.Doc
qgy.zhuzhoujiudazxL1517.cn/42044.Doc
qgy.zhuzhoujiudazxL1517.cn/64688.Doc
qgy.zhuzhoujiudazxL1517.cn/86620.Doc
qgt.zhuzhoujiudazxL1517.cn/44242.Doc
qgt.zhuzhoujiudazxL1517.cn/26260.Doc
qgt.zhuzhoujiudazxL1517.cn/42040.Doc
qgt.zhuzhoujiudazxL1517.cn/22688.Doc
qgt.zhuzhoujiudazxL1517.cn/28460.Doc
qgt.zhuzhoujiudazxL1517.cn/88864.Doc
qgt.zhuzhoujiudazxL1517.cn/66206.Doc
qgt.zhuzhoujiudazxL1517.cn/40628.Doc
qgt.zhuzhoujiudazxL1517.cn/28480.Doc
qgt.zhuzhoujiudazxL1517.cn/48886.Doc
qgr.zhuzhoujiudazxL1517.cn/22420.Doc
qgr.zhuzhoujiudazxL1517.cn/04268.Doc
qgr.zhuzhoujiudazxL1517.cn/08000.Doc
qgr.zhuzhoujiudazxL1517.cn/20844.Doc
qgr.zhuzhoujiudazxL1517.cn/22268.Doc
qgr.zhuzhoujiudazxL1517.cn/86626.Doc
qgr.zhuzhoujiudazxL1517.cn/44462.Doc
qgr.zhuzhoujiudazxL1517.cn/20686.Doc
qgr.zhuzhoujiudazxL1517.cn/40622.Doc
qgr.zhuzhoujiudazxL1517.cn/08664.Doc
qge.zhuzhoujiudazxL1517.cn/48824.Doc
qge.zhuzhoujiudazxL1517.cn/39135.Doc
qge.zhuzhoujiudazxL1517.cn/24206.Doc
qge.zhuzhoujiudazxL1517.cn/06666.Doc
qge.zhuzhoujiudazxL1517.cn/44246.Doc
qge.zhuzhoujiudazxL1517.cn/82486.Doc
qge.zhuzhoujiudazxL1517.cn/64680.Doc
qge.zhuzhoujiudazxL1517.cn/84066.Doc
qge.zhuzhoujiudazxL1517.cn/22608.Doc
qge.zhuzhoujiudazxL1517.cn/86848.Doc
qgw.zhuzhoujiudazxL1517.cn/20684.Doc
qgw.zhuzhoujiudazxL1517.cn/06626.Doc
qgw.zhuzhoujiudazxL1517.cn/84880.Doc
qgw.zhuzhoujiudazxL1517.cn/02204.Doc
qgw.zhuzhoujiudazxL1517.cn/04400.Doc
qgw.zhuzhoujiudazxL1517.cn/08440.Doc
qgw.zhuzhoujiudazxL1517.cn/62644.Doc
qgw.zhuzhoujiudazxL1517.cn/20682.Doc
qgw.zhuzhoujiudazxL1517.cn/28262.Doc
qgw.zhuzhoujiudazxL1517.cn/37113.Doc
qgq.zhuzhoujiudazxL1517.cn/62664.Doc
qgq.zhuzhoujiudazxL1517.cn/28280.Doc
qgq.zhuzhoujiudazxL1517.cn/28622.Doc
qgq.zhuzhoujiudazxL1517.cn/88624.Doc
qgq.zhuzhoujiudazxL1517.cn/62400.Doc
qgq.zhuzhoujiudazxL1517.cn/84000.Doc
qgq.zhuzhoujiudazxL1517.cn/33715.Doc
qgq.zhuzhoujiudazxL1517.cn/60288.Doc
qgq.zhuzhoujiudazxL1517.cn/62682.Doc
qgq.zhuzhoujiudazxL1517.cn/04868.Doc
qfm.zhuzhoujiudazxL1517.cn/00822.Doc
qfm.zhuzhoujiudazxL1517.cn/28660.Doc
qfm.zhuzhoujiudazxL1517.cn/62222.Doc
qfm.zhuzhoujiudazxL1517.cn/02222.Doc
qfm.zhuzhoujiudazxL1517.cn/04268.Doc
qfm.zhuzhoujiudazxL1517.cn/82228.Doc
qfm.zhuzhoujiudazxL1517.cn/66862.Doc
qfm.zhuzhoujiudazxL1517.cn/48864.Doc
qfm.zhuzhoujiudazxL1517.cn/48266.Doc
qfm.zhuzhoujiudazxL1517.cn/24808.Doc
qfn.zhuzhoujiudazxL1517.cn/68864.Doc
qfn.zhuzhoujiudazxL1517.cn/02624.Doc
qfn.zhuzhoujiudazxL1517.cn/82886.Doc
qfn.zhuzhoujiudazxL1517.cn/86402.Doc
qfn.zhuzhoujiudazxL1517.cn/02668.Doc
qfn.zhuzhoujiudazxL1517.cn/66484.Doc
qfn.zhuzhoujiudazxL1517.cn/37991.Doc
qfn.zhuzhoujiudazxL1517.cn/20668.Doc
qfn.zhuzhoujiudazxL1517.cn/22068.Doc
qfn.zhuzhoujiudazxL1517.cn/64222.Doc
qfb.zhuzhoujiudazxL1517.cn/24206.Doc
qfb.zhuzhoujiudazxL1517.cn/86688.Doc
qfb.zhuzhoujiudazxL1517.cn/19553.Doc
qfb.zhuzhoujiudazxL1517.cn/82002.Doc
qfb.zhuzhoujiudazxL1517.cn/24060.Doc
qfb.zhuzhoujiudazxL1517.cn/26628.Doc
qfb.zhuzhoujiudazxL1517.cn/08284.Doc
qfb.zhuzhoujiudazxL1517.cn/62882.Doc
qfb.zhuzhoujiudazxL1517.cn/66488.Doc
qfb.zhuzhoujiudazxL1517.cn/68286.Doc
qfv.zhuzhoujiudazxL1517.cn/68444.Doc
qfv.zhuzhoujiudazxL1517.cn/82824.Doc
qfv.zhuzhoujiudazxL1517.cn/86620.Doc
qfv.zhuzhoujiudazxL1517.cn/75579.Doc
qfv.zhuzhoujiudazxL1517.cn/22844.Doc
qfv.zhuzhoujiudazxL1517.cn/24800.Doc
qfv.zhuzhoujiudazxL1517.cn/84086.Doc
qfv.zhuzhoujiudazxL1517.cn/80400.Doc
qfv.zhuzhoujiudazxL1517.cn/44640.Doc
qfv.zhuzhoujiudazxL1517.cn/40064.Doc
qfc.zhuzhoujiudazxL1517.cn/84222.Doc
qfc.zhuzhoujiudazxL1517.cn/20066.Doc
qfc.zhuzhoujiudazxL1517.cn/57311.Doc
qfc.zhuzhoujiudazxL1517.cn/86880.Doc
qfc.zhuzhoujiudazxL1517.cn/64668.Doc
qfc.zhuzhoujiudazxL1517.cn/88602.Doc
qfc.zhuzhoujiudazxL1517.cn/66648.Doc
qfc.zhuzhoujiudazxL1517.cn/46846.Doc
qfc.zhuzhoujiudazxL1517.cn/48002.Doc
qfc.zhuzhoujiudazxL1517.cn/22228.Doc
qfx.zhuzhoujiudazxL1517.cn/02444.Doc
qfx.zhuzhoujiudazxL1517.cn/06604.Doc
qfx.zhuzhoujiudazxL1517.cn/02260.Doc
qfx.zhuzhoujiudazxL1517.cn/46004.Doc
qfx.zhuzhoujiudazxL1517.cn/04844.Doc
qfx.zhuzhoujiudazxL1517.cn/64262.Doc
qfx.zhuzhoujiudazxL1517.cn/08022.Doc
qfx.zhuzhoujiudazxL1517.cn/02640.Doc
qfx.zhuzhoujiudazxL1517.cn/24686.Doc
qfx.zhuzhoujiudazxL1517.cn/28620.Doc
qfz.zhuzhoujiudazxL1517.cn/28020.Doc
qfz.zhuzhoujiudazxL1517.cn/11991.Doc
qfz.zhuzhoujiudazxL1517.cn/84884.Doc
qfz.zhuzhoujiudazxL1517.cn/60226.Doc
qfz.zhuzhoujiudazxL1517.cn/20228.Doc
qfz.zhuzhoujiudazxL1517.cn/48080.Doc
qfz.zhuzhoujiudazxL1517.cn/26446.Doc
qfz.zhuzhoujiudazxL1517.cn/60606.Doc
qfz.zhuzhoujiudazxL1517.cn/82042.Doc
qfz.zhuzhoujiudazxL1517.cn/42880.Doc
qfl.zhuzhoujiudazxL1517.cn/80442.Doc
qfl.zhuzhoujiudazxL1517.cn/40608.Doc
qfl.zhuzhoujiudazxL1517.cn/48402.Doc
qfl.zhuzhoujiudazxL1517.cn/00802.Doc
qfl.zhuzhoujiudazxL1517.cn/22446.Doc
qfl.zhuzhoujiudazxL1517.cn/17337.Doc
qfl.zhuzhoujiudazxL1517.cn/00202.Doc
qfl.zhuzhoujiudazxL1517.cn/46608.Doc
qfl.zhuzhoujiudazxL1517.cn/42080.Doc
qfl.zhuzhoujiudazxL1517.cn/44226.Doc
qfk.zhuzhoujiudazxL1517.cn/80402.Doc
qfk.zhuzhoujiudazxL1517.cn/06820.Doc
qfk.zhuzhoujiudazxL1517.cn/93971.Doc
qfk.zhuzhoujiudazxL1517.cn/68480.Doc
qfk.zhuzhoujiudazxL1517.cn/00266.Doc
qfk.zhuzhoujiudazxL1517.cn/82880.Doc
qfk.zhuzhoujiudazxL1517.cn/22668.Doc
qfk.zhuzhoujiudazxL1517.cn/24448.Doc
qfk.zhuzhoujiudazxL1517.cn/26662.Doc
qfk.zhuzhoujiudazxL1517.cn/86266.Doc
qfj.zhuzhoujiudazxL1517.cn/88840.Doc
qfj.zhuzhoujiudazxL1517.cn/24442.Doc
qfj.zhuzhoujiudazxL1517.cn/40064.Doc
qfj.zhuzhoujiudazxL1517.cn/64628.Doc
qfj.zhuzhoujiudazxL1517.cn/82662.Doc
qfj.zhuzhoujiudazxL1517.cn/26426.Doc
qfj.zhuzhoujiudazxL1517.cn/42826.Doc
qfj.zhuzhoujiudazxL1517.cn/24224.Doc
qfj.zhuzhoujiudazxL1517.cn/31573.Doc
qfj.zhuzhoujiudazxL1517.cn/79591.Doc
qfh.zhuzhoujiudazxL1517.cn/68088.Doc
qfh.zhuzhoujiudazxL1517.cn/60626.Doc
qfh.zhuzhoujiudazxL1517.cn/40862.Doc
qfh.zhuzhoujiudazxL1517.cn/06284.Doc
qfh.zhuzhoujiudazxL1517.cn/84428.Doc
qfh.zhuzhoujiudazxL1517.cn/48008.Doc
qfh.zhuzhoujiudazxL1517.cn/86044.Doc
qfh.zhuzhoujiudazxL1517.cn/62262.Doc
qfh.zhuzhoujiudazxL1517.cn/33159.Doc
qfh.zhuzhoujiudazxL1517.cn/71575.Doc
qfg.zhuzhoujiudazxL1517.cn/53517.Doc
qfg.zhuzhoujiudazxL1517.cn/62422.Doc
qfg.zhuzhoujiudazxL1517.cn/44666.Doc
qfg.zhuzhoujiudazxL1517.cn/24626.Doc
qfg.zhuzhoujiudazxL1517.cn/08206.Doc
qfg.zhuzhoujiudazxL1517.cn/08840.Doc
qfg.zhuzhoujiudazxL1517.cn/22482.Doc
qfg.zhuzhoujiudazxL1517.cn/00260.Doc
qfg.zhuzhoujiudazxL1517.cn/88624.Doc
qfg.zhuzhoujiudazxL1517.cn/31973.Doc
qff.zhuzhoujiudazxL1517.cn/40804.Doc
qff.zhuzhoujiudazxL1517.cn/44664.Doc
qff.zhuzhoujiudazxL1517.cn/46202.Doc
qff.zhuzhoujiudazxL1517.cn/20608.Doc
qff.zhuzhoujiudazxL1517.cn/08002.Doc
qff.zhuzhoujiudazxL1517.cn/82840.Doc
qff.zhuzhoujiudazxL1517.cn/62004.Doc
qff.zhuzhoujiudazxL1517.cn/68288.Doc
qff.zhuzhoujiudazxL1517.cn/80088.Doc
qff.zhuzhoujiudazxL1517.cn/86466.Doc
qfd.zhuzhoujiudazxL1517.cn/26600.Doc
qfd.zhuzhoujiudazxL1517.cn/22444.Doc
qfd.zhuzhoujiudazxL1517.cn/06402.Doc
qfd.zhuzhoujiudazxL1517.cn/66468.Doc
qfd.zhuzhoujiudazxL1517.cn/24808.Doc
qfd.zhuzhoujiudazxL1517.cn/13915.Doc
qfd.zhuzhoujiudazxL1517.cn/26080.Doc
qfd.zhuzhoujiudazxL1517.cn/40068.Doc
qfd.zhuzhoujiudazxL1517.cn/37117.Doc
qfd.zhuzhoujiudazxL1517.cn/04628.Doc
qfs.zhuzhoujiudazxL1517.cn/28086.Doc
qfs.zhuzhoujiudazxL1517.cn/46888.Doc
qfs.zhuzhoujiudazxL1517.cn/60424.Doc
qfs.zhuzhoujiudazxL1517.cn/64044.Doc
qfs.zhuzhoujiudazxL1517.cn/08488.Doc
qfs.zhuzhoujiudazxL1517.cn/31973.Doc
qfs.zhuzhoujiudazxL1517.cn/97533.Doc
qfs.zhuzhoujiudazxL1517.cn/86648.Doc
qfs.zhuzhoujiudazxL1517.cn/84260.Doc
qfs.zhuzhoujiudazxL1517.cn/62882.Doc
qfa.zhuzhoujiudazxL1517.cn/66228.Doc
qfa.zhuzhoujiudazxL1517.cn/00286.Doc
qfa.zhuzhoujiudazxL1517.cn/86000.Doc
qfa.zhuzhoujiudazxL1517.cn/20204.Doc
qfa.zhuzhoujiudazxL1517.cn/04604.Doc
qfa.zhuzhoujiudazxL1517.cn/22680.Doc
qfa.zhuzhoujiudazxL1517.cn/06622.Doc
qfa.zhuzhoujiudazxL1517.cn/68264.Doc
qfa.zhuzhoujiudazxL1517.cn/62082.Doc
qfa.zhuzhoujiudazxL1517.cn/42260.Doc
qfp.zhuzhoujiudazxL1517.cn/22282.Doc
qfp.zhuzhoujiudazxL1517.cn/86848.Doc
qfp.zhuzhoujiudazxL1517.cn/95751.Doc
qfp.zhuzhoujiudazxL1517.cn/28020.Doc
qfp.zhuzhoujiudazxL1517.cn/06046.Doc
qfp.zhuzhoujiudazxL1517.cn/15199.Doc
qfp.zhuzhoujiudazxL1517.cn/62624.Doc
qfp.zhuzhoujiudazxL1517.cn/20286.Doc
qfp.zhuzhoujiudazxL1517.cn/40800.Doc
qfp.zhuzhoujiudazxL1517.cn/26648.Doc
qfo.zhuzhoujiudazxL1517.cn/68480.Doc
qfo.zhuzhoujiudazxL1517.cn/84442.Doc
qfo.zhuzhoujiudazxL1517.cn/40080.Doc
qfo.zhuzhoujiudazxL1517.cn/40802.Doc
qfo.zhuzhoujiudazxL1517.cn/59995.Doc
qfo.zhuzhoujiudazxL1517.cn/04648.Doc
qfo.zhuzhoujiudazxL1517.cn/88404.Doc
qfo.zhuzhoujiudazxL1517.cn/86826.Doc
qfo.zhuzhoujiudazxL1517.cn/08648.Doc
qfo.zhuzhoujiudazxL1517.cn/04486.Doc
qfi.zhuzhoujiudazxL1517.cn/28620.Doc
qfi.zhuzhoujiudazxL1517.cn/35337.Doc
qfi.zhuzhoujiudazxL1517.cn/42420.Doc
qfi.zhuzhoujiudazxL1517.cn/06842.Doc
qfi.zhuzhoujiudazxL1517.cn/88602.Doc
qfi.zhuzhoujiudazxL1517.cn/46062.Doc
qfi.zhuzhoujiudazxL1517.cn/06442.Doc
qfi.zhuzhoujiudazxL1517.cn/64464.Doc
qfi.zhuzhoujiudazxL1517.cn/60240.Doc
qfi.zhuzhoujiudazxL1517.cn/84246.Doc
qfu.zhuzhoujiudazxL1517.cn/22268.Doc
qfu.zhuzhoujiudazxL1517.cn/22484.Doc
qfu.zhuzhoujiudazxL1517.cn/44064.Doc
qfu.zhuzhoujiudazxL1517.cn/00644.Doc
qfu.zhuzhoujiudazxL1517.cn/20622.Doc
qfu.zhuzhoujiudazxL1517.cn/46684.Doc
qfu.zhuzhoujiudazxL1517.cn/06008.Doc
qfu.zhuzhoujiudazxL1517.cn/06846.Doc
qfu.zhuzhoujiudazxL1517.cn/22048.Doc
qfu.zhuzhoujiudazxL1517.cn/37753.Doc
qfy.zhuzhoujiudazxL1517.cn/64680.Doc
qfy.zhuzhoujiudazxL1517.cn/68440.Doc
qfy.zhuzhoujiudazxL1517.cn/04866.Doc
qfy.zhuzhoujiudazxL1517.cn/88686.Doc
qfy.zhuzhoujiudazxL1517.cn/62282.Doc
qfy.zhuzhoujiudazxL1517.cn/02406.Doc
qfy.zhuzhoujiudazxL1517.cn/02222.Doc
qfy.zhuzhoujiudazxL1517.cn/42046.Doc
qfy.zhuzhoujiudazxL1517.cn/57135.Doc
qfy.zhuzhoujiudazxL1517.cn/24262.Doc
qft.zhuzhoujiudazxL1517.cn/64886.Doc
qft.zhuzhoujiudazxL1517.cn/20480.Doc
qft.zhuzhoujiudazxL1517.cn/02408.Doc
qft.zhuzhoujiudazxL1517.cn/48004.Doc
qft.zhuzhoujiudazxL1517.cn/68640.Doc
qft.zhuzhoujiudazxL1517.cn/00062.Doc
qft.zhuzhoujiudazxL1517.cn/68442.Doc
qft.zhuzhoujiudazxL1517.cn/48246.Doc
qft.zhuzhoujiudazxL1517.cn/82286.Doc
qft.zhuzhoujiudazxL1517.cn/60404.Doc
qfr.zhuzhoujiudazxL1517.cn/44268.Doc
qfr.zhuzhoujiudazxL1517.cn/46806.Doc
qfr.zhuzhoujiudazxL1517.cn/88202.Doc
qfr.zhuzhoujiudazxL1517.cn/84688.Doc
qfr.zhuzhoujiudazxL1517.cn/26640.Doc
qfr.zhuzhoujiudazxL1517.cn/44048.Doc
qfr.zhuzhoujiudazxL1517.cn/00824.Doc
qfr.zhuzhoujiudazxL1517.cn/00422.Doc
qfr.zhuzhoujiudazxL1517.cn/24820.Doc
qfr.zhuzhoujiudazxL1517.cn/80062.Doc
qfe.zhuzhoujiudazxL1517.cn/48084.Doc
qfe.zhuzhoujiudazxL1517.cn/64842.Doc
qfe.zhuzhoujiudazxL1517.cn/28422.Doc
qfe.zhuzhoujiudazxL1517.cn/06460.Doc
qfe.zhuzhoujiudazxL1517.cn/57153.Doc
qfe.zhuzhoujiudazxL1517.cn/22820.Doc
qfe.zhuzhoujiudazxL1517.cn/84826.Doc
qfe.zhuzhoujiudazxL1517.cn/48604.Doc
qfe.zhuzhoujiudazxL1517.cn/68462.Doc
qfe.zhuzhoujiudazxL1517.cn/40242.Doc
qfw.zhuzhoujiudazxL1517.cn/64202.Doc
qfw.zhuzhoujiudazxL1517.cn/42060.Doc
qfw.zhuzhoujiudazxL1517.cn/66808.Doc
qfw.zhuzhoujiudazxL1517.cn/08464.Doc
qfw.zhuzhoujiudazxL1517.cn/80022.Doc
qfw.zhuzhoujiudazxL1517.cn/28242.Doc
qfw.zhuzhoujiudazxL1517.cn/06242.Doc
qfw.zhuzhoujiudazxL1517.cn/24228.Doc
qfw.zhuzhoujiudazxL1517.cn/28800.Doc
qfw.zhuzhoujiudazxL1517.cn/88806.Doc
qfq.zhuzhoujiudazxL1517.cn/44046.Doc
qfq.zhuzhoujiudazxL1517.cn/04282.Doc
qfq.zhuzhoujiudazxL1517.cn/00646.Doc
qfq.zhuzhoujiudazxL1517.cn/06406.Doc
qfq.zhuzhoujiudazxL1517.cn/84826.Doc
qfq.zhuzhoujiudazxL1517.cn/28040.Doc
qfq.zhuzhoujiudazxL1517.cn/60224.Doc
qfq.zhuzhoujiudazxL1517.cn/60464.Doc
qfq.zhuzhoujiudazxL1517.cn/80680.Doc
qfq.zhuzhoujiudazxL1517.cn/26282.Doc
qdm.zhuzhoujiudazxL1517.cn/04244.Doc
qdm.zhuzhoujiudazxL1517.cn/97115.Doc
qdm.zhuzhoujiudazxL1517.cn/40022.Doc
qdm.zhuzhoujiudazxL1517.cn/00840.Doc
qdm.zhuzhoujiudazxL1517.cn/08680.Doc
qdm.zhuzhoujiudazxL1517.cn/26288.Doc
qdm.zhuzhoujiudazxL1517.cn/04028.Doc
qdm.zhuzhoujiudazxL1517.cn/51179.Doc
qdm.zhuzhoujiudazxL1517.cn/06842.Doc
qdm.zhuzhoujiudazxL1517.cn/22048.Doc
qdn.zhuzhoujiudazxL1517.cn/48882.Doc
qdn.zhuzhoujiudazxL1517.cn/20208.Doc
qdn.zhuzhoujiudazxL1517.cn/88064.Doc
qdn.zhuzhoujiudazxL1517.cn/84624.Doc
qdn.zhuzhoujiudazxL1517.cn/44462.Doc
qdn.zhuzhoujiudazxL1517.cn/06688.Doc
qdn.zhuzhoujiudazxL1517.cn/40402.Doc
qdn.zhuzhoujiudazxL1517.cn/08084.Doc
qdn.zhuzhoujiudazxL1517.cn/75731.Doc
qdn.zhuzhoujiudazxL1517.cn/77997.Doc
qdb.zhuzhoujiudazxL1517.cn/40848.Doc
qdb.zhuzhoujiudazxL1517.cn/08086.Doc
qdb.zhuzhoujiudazxL1517.cn/68884.Doc
qdb.zhuzhoujiudazxL1517.cn/04424.Doc
qdb.zhuzhoujiudazxL1517.cn/20440.Doc
qdb.zhuzhoujiudazxL1517.cn/04486.Doc
qdb.zhuzhoujiudazxL1517.cn/64244.Doc
qdb.zhuzhoujiudazxL1517.cn/86428.Doc
qdb.zhuzhoujiudazxL1517.cn/62860.Doc
qdb.zhuzhoujiudazxL1517.cn/86666.Doc
qdv.zhuzhoujiudazxL1517.cn/60840.Doc
qdv.zhuzhoujiudazxL1517.cn/28888.Doc
qdv.zhuzhoujiudazxL1517.cn/08828.Doc
qdv.zhuzhoujiudazxL1517.cn/68282.Doc
qdv.zhuzhoujiudazxL1517.cn/88428.Doc
qdv.zhuzhoujiudazxL1517.cn/02260.Doc
qdv.zhuzhoujiudazxL1517.cn/60408.Doc
qdv.zhuzhoujiudazxL1517.cn/64244.Doc
qdv.zhuzhoujiudazxL1517.cn/84824.Doc
qdv.zhuzhoujiudazxL1517.cn/46844.Doc
qdc.zhuzhoujiudazxL1517.cn/06880.Doc
qdc.zhuzhoujiudazxL1517.cn/02868.Doc
qdc.zhuzhoujiudazxL1517.cn/04626.Doc
qdc.zhuzhoujiudazxL1517.cn/02648.Doc
qdc.zhuzhoujiudazxL1517.cn/86240.Doc
qdc.zhuzhoujiudazxL1517.cn/02624.Doc
qdc.zhuzhoujiudazxL1517.cn/06202.Doc
qdc.zhuzhoujiudazxL1517.cn/3.Doc
qdc.zhuzhoujiudazxL1517.cn/28088.Doc
qdc.zhuzhoujiudazxL1517.cn/22648.Doc
qdx.zhuzhoujiudazxL1517.cn/84446.Doc
qdx.zhuzhoujiudazxL1517.cn/00602.Doc
qdx.zhuzhoujiudazxL1517.cn/80642.Doc
qdx.zhuzhoujiudazxL1517.cn/46288.Doc
qdx.zhuzhoujiudazxL1517.cn/79977.Doc
qdx.zhuzhoujiudazxL1517.cn/40262.Doc
qdx.zhuzhoujiudazxL1517.cn/46284.Doc
qdx.zhuzhoujiudazxL1517.cn/84428.Doc
qdx.zhuzhoujiudazxL1517.cn/66428.Doc
qdx.zhuzhoujiudazxL1517.cn/26482.Doc
qdz.zhuzhoujiudazxL1517.cn/64402.Doc
qdz.zhuzhoujiudazxL1517.cn/28684.Doc
qdz.zhuzhoujiudazxL1517.cn/80668.Doc
qdz.zhuzhoujiudazxL1517.cn/62262.Doc
qdz.zhuzhoujiudazxL1517.cn/68624.Doc
qdz.zhuzhoujiudazxL1517.cn/00880.Doc
qdz.zhuzhoujiudazxL1517.cn/28026.Doc
qdz.zhuzhoujiudazxL1517.cn/68060.Doc
qdz.zhuzhoujiudazxL1517.cn/64606.Doc
qdz.zhuzhoujiudazxL1517.cn/26808.Doc
qdl.zhuzhoujiudazxL1517.cn/00280.Doc
qdl.zhuzhoujiudazxL1517.cn/44080.Doc
qdl.zhuzhoujiudazxL1517.cn/00488.Doc
qdl.zhuzhoujiudazxL1517.cn/00626.Doc
qdl.zhuzhoujiudazxL1517.cn/28000.Doc
qdl.zhuzhoujiudazxL1517.cn/62826.Doc
qdl.zhuzhoujiudazxL1517.cn/48244.Doc
qdl.zhuzhoujiudazxL1517.cn/66686.Doc
qdl.zhuzhoujiudazxL1517.cn/11151.Doc
qdl.zhuzhoujiudazxL1517.cn/44042.Doc
