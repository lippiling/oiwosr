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

evv.jyw669.cn/62464.Doc
evv.jyw669.cn/02606.Doc
evv.jyw669.cn/28482.Doc
evv.jyw669.cn/44448.Doc
evv.jyw669.cn/11357.Doc
evv.jyw669.cn/02628.Doc
evv.jyw669.cn/06026.Doc
evv.jyw669.cn/22242.Doc
evv.jyw669.cn/48828.Doc
evv.jyw669.cn/48806.Doc
evc.jyw669.cn/84848.Doc
evc.jyw669.cn/80246.Doc
evc.jyw669.cn/60602.Doc
evc.jyw669.cn/82688.Doc
evc.jyw669.cn/48826.Doc
evc.jyw669.cn/62804.Doc
evc.jyw669.cn/48800.Doc
evc.jyw669.cn/17939.Doc
evc.jyw669.cn/22226.Doc
evc.jyw669.cn/26266.Doc
evx.jyw669.cn/46880.Doc
evx.jyw669.cn/80880.Doc
evx.jyw669.cn/42882.Doc
evx.jyw669.cn/06422.Doc
evx.jyw669.cn/64280.Doc
evx.jyw669.cn/00626.Doc
evx.jyw669.cn/04804.Doc
evx.jyw669.cn/80424.Doc
evx.jyw669.cn/02240.Doc
evx.jyw669.cn/55393.Doc
evz.jyw669.cn/22448.Doc
evz.jyw669.cn/73599.Doc
evz.jyw669.cn/02246.Doc
evz.jyw669.cn/84204.Doc
evz.jyw669.cn/17777.Doc
evz.jyw669.cn/62226.Doc
evz.jyw669.cn/42084.Doc
evz.jyw669.cn/64484.Doc
evz.jyw669.cn/42262.Doc
evz.jyw669.cn/46462.Doc
evl.jyw669.cn/84046.Doc
evl.jyw669.cn/60206.Doc
evl.jyw669.cn/82628.Doc
evl.jyw669.cn/88020.Doc
evl.jyw669.cn/86420.Doc
evl.jyw669.cn/62420.Doc
evl.jyw669.cn/48068.Doc
evl.jyw669.cn/60204.Doc
evl.jyw669.cn/86422.Doc
evl.jyw669.cn/24622.Doc
evk.jyw669.cn/82428.Doc
evk.jyw669.cn/06440.Doc
evk.jyw669.cn/84046.Doc
evk.jyw669.cn/04600.Doc
evk.jyw669.cn/22488.Doc
evk.jyw669.cn/04686.Doc
evk.jyw669.cn/62422.Doc
evk.jyw669.cn/68888.Doc
evk.jyw669.cn/28280.Doc
evk.jyw669.cn/00022.Doc
evj.jyw669.cn/00844.Doc
evj.jyw669.cn/82624.Doc
evj.jyw669.cn/15511.Doc
evj.jyw669.cn/13191.Doc
evj.jyw669.cn/80808.Doc
evj.jyw669.cn/64482.Doc
evj.jyw669.cn/26604.Doc
evj.jyw669.cn/64480.Doc
evj.jyw669.cn/46048.Doc
evj.jyw669.cn/20208.Doc
evh.jyw669.cn/64484.Doc
evh.jyw669.cn/02264.Doc
evh.jyw669.cn/20440.Doc
evh.jyw669.cn/06608.Doc
evh.jyw669.cn/06666.Doc
evh.jyw669.cn/33151.Doc
evh.jyw669.cn/62802.Doc
evh.jyw669.cn/40446.Doc
evh.jyw669.cn/46680.Doc
evh.jyw669.cn/60842.Doc
evg.jyw669.cn/08842.Doc
evg.jyw669.cn/91713.Doc
evg.jyw669.cn/00444.Doc
evg.jyw669.cn/08468.Doc
evg.jyw669.cn/22682.Doc
evg.jyw669.cn/60088.Doc
evg.jyw669.cn/64642.Doc
evg.jyw669.cn/00240.Doc
evg.jyw669.cn/28686.Doc
evg.jyw669.cn/62424.Doc
evf.jyw669.cn/64286.Doc
evf.jyw669.cn/26204.Doc
evf.jyw669.cn/22424.Doc
evf.jyw669.cn/26444.Doc
evf.jyw669.cn/02888.Doc
evf.jyw669.cn/53973.Doc
evf.jyw669.cn/26224.Doc
evf.jyw669.cn/02688.Doc
evf.jyw669.cn/40608.Doc
evf.jyw669.cn/86046.Doc
evd.jyw669.cn/77913.Doc
evd.jyw669.cn/40822.Doc
evd.jyw669.cn/55199.Doc
evd.jyw669.cn/46000.Doc
evd.jyw669.cn/20866.Doc
evd.jyw669.cn/02088.Doc
evd.jyw669.cn/80868.Doc
evd.jyw669.cn/86888.Doc
evd.jyw669.cn/62008.Doc
evd.jyw669.cn/06882.Doc
evs.jyw669.cn/15739.Doc
evs.jyw669.cn/15151.Doc
evs.jyw669.cn/48446.Doc
evs.jyw669.cn/60062.Doc
evs.jyw669.cn/62220.Doc
evs.jyw669.cn/40046.Doc
evs.jyw669.cn/08200.Doc
evs.jyw669.cn/46646.Doc
evs.jyw669.cn/62688.Doc
evs.jyw669.cn/20282.Doc
eva.jyw669.cn/06226.Doc
eva.jyw669.cn/88424.Doc
eva.jyw669.cn/62002.Doc
eva.jyw669.cn/20862.Doc
eva.jyw669.cn/42048.Doc
eva.jyw669.cn/13191.Doc
eva.jyw669.cn/71139.Doc
eva.jyw669.cn/95715.Doc
eva.jyw669.cn/55155.Doc
eva.jyw669.cn/08466.Doc
evp.jyw669.cn/64068.Doc
evp.jyw669.cn/88006.Doc
evp.jyw669.cn/08460.Doc
evp.jyw669.cn/08668.Doc
evp.jyw669.cn/42284.Doc
evp.jyw669.cn/88688.Doc
evp.jyw669.cn/66822.Doc
evp.jyw669.cn/06282.Doc
evp.jyw669.cn/24482.Doc
evp.jyw669.cn/84004.Doc
evo.jyw669.cn/42668.Doc
evo.jyw669.cn/00686.Doc
evo.jyw669.cn/86600.Doc
evo.jyw669.cn/31913.Doc
evo.jyw669.cn/71331.Doc
evo.jyw669.cn/08286.Doc
evo.jyw669.cn/24444.Doc
evo.jyw669.cn/60048.Doc
evo.jyw669.cn/26200.Doc
evo.jyw669.cn/62200.Doc
evi.jyw669.cn/24228.Doc
evi.jyw669.cn/22402.Doc
evi.jyw669.cn/44020.Doc
evi.jyw669.cn/46646.Doc
evi.jyw669.cn/00268.Doc
evi.jyw669.cn/28880.Doc
evi.jyw669.cn/80622.Doc
evi.jyw669.cn/48228.Doc
evi.jyw669.cn/46004.Doc
evi.jyw669.cn/22244.Doc
evu.jyw669.cn/88288.Doc
evu.jyw669.cn/66882.Doc
evu.jyw669.cn/68042.Doc
evu.jyw669.cn/80260.Doc
evu.jyw669.cn/60026.Doc
evu.jyw669.cn/60840.Doc
evu.jyw669.cn/62286.Doc
evu.jyw669.cn/06488.Doc
evu.jyw669.cn/00260.Doc
evu.jyw669.cn/84468.Doc
evy.jyw669.cn/08084.Doc
evy.jyw669.cn/02420.Doc
evy.jyw669.cn/82060.Doc
evy.jyw669.cn/06202.Doc
evy.jyw669.cn/04424.Doc
evy.jyw669.cn/40824.Doc
evy.jyw669.cn/08842.Doc
evy.jyw669.cn/84804.Doc
evy.jyw669.cn/08064.Doc
evy.jyw669.cn/80666.Doc
evt.jyw669.cn/60040.Doc
evt.jyw669.cn/60202.Doc
evt.jyw669.cn/84020.Doc
evt.jyw669.cn/66488.Doc
evt.jyw669.cn/26040.Doc
evt.jyw669.cn/75117.Doc
evt.jyw669.cn/88468.Doc
evt.jyw669.cn/82026.Doc
evt.jyw669.cn/39555.Doc
evt.jyw669.cn/48884.Doc
evr.jyw669.cn/75191.Doc
evr.jyw669.cn/64680.Doc
evr.jyw669.cn/20060.Doc
evr.jyw669.cn/84468.Doc
evr.jyw669.cn/57391.Doc
evr.jyw669.cn/60226.Doc
evr.jyw669.cn/13937.Doc
evr.jyw669.cn/02480.Doc
evr.jyw669.cn/84040.Doc
evr.jyw669.cn/88282.Doc
eve.jyw669.cn/44468.Doc
eve.jyw669.cn/02242.Doc
eve.jyw669.cn/84082.Doc
eve.jyw669.cn/22820.Doc
eve.jyw669.cn/02086.Doc
eve.jyw669.cn/64246.Doc
eve.jyw669.cn/62408.Doc
eve.jyw669.cn/02620.Doc
eve.jyw669.cn/60064.Doc
eve.jyw669.cn/40400.Doc
evw.jyw669.cn/93319.Doc
evw.jyw669.cn/26680.Doc
evw.jyw669.cn/64024.Doc
evw.jyw669.cn/66640.Doc
evw.jyw669.cn/68842.Doc
evw.jyw669.cn/11757.Doc
evw.jyw669.cn/02680.Doc
evw.jyw669.cn/66408.Doc
evw.jyw669.cn/00008.Doc
evw.jyw669.cn/28420.Doc
evq.jyw669.cn/15533.Doc
evq.jyw669.cn/48448.Doc
evq.jyw669.cn/82602.Doc
evq.jyw669.cn/48044.Doc
evq.jyw669.cn/86882.Doc
evq.jyw669.cn/40486.Doc
evq.jyw669.cn/06262.Doc
evq.jyw669.cn/22668.Doc
evq.jyw669.cn/64282.Doc
evq.jyw669.cn/28026.Doc
ecm.jyw669.cn/26626.Doc
ecm.jyw669.cn/97553.Doc
ecm.jyw669.cn/37731.Doc
ecm.jyw669.cn/06222.Doc
ecm.jyw669.cn/64688.Doc
ecm.jyw669.cn/86008.Doc
ecm.jyw669.cn/04424.Doc
ecm.jyw669.cn/86646.Doc
ecm.jyw669.cn/28860.Doc
ecm.jyw669.cn/06082.Doc
ecn.jyw669.cn/40882.Doc
ecn.jyw669.cn/64266.Doc
ecn.jyw669.cn/79775.Doc
ecn.jyw669.cn/64464.Doc
ecn.jyw669.cn/37315.Doc
ecn.jyw669.cn/88426.Doc
ecn.jyw669.cn/26486.Doc
ecn.jyw669.cn/42686.Doc
ecn.jyw669.cn/60640.Doc
ecn.jyw669.cn/40735.Doc
ecb.jyw669.cn/48242.Doc
ecb.jyw669.cn/06664.Doc
ecb.jyw669.cn/60402.Doc
ecb.jyw669.cn/35155.Doc
ecb.jyw669.cn/37153.Doc
ecb.jyw669.cn/46808.Doc
ecb.jyw669.cn/68840.Doc
ecb.jyw669.cn/84820.Doc
ecb.jyw669.cn/86662.Doc
ecb.jyw669.cn/59377.Doc
ecv.jyw669.cn/64484.Doc
ecv.jyw669.cn/51317.Doc
ecv.jyw669.cn/64426.Doc
ecv.jyw669.cn/44260.Doc
ecv.jyw669.cn/40048.Doc
ecv.jyw669.cn/44802.Doc
ecv.jyw669.cn/48400.Doc
ecv.jyw669.cn/08606.Doc
ecv.jyw669.cn/68422.Doc
ecv.jyw669.cn/20808.Doc
ecc.jyw669.cn/46468.Doc
ecc.jyw669.cn/84422.Doc
ecc.jyw669.cn/06266.Doc
ecc.jyw669.cn/06844.Doc
ecc.jyw669.cn/06882.Doc
ecc.jyw669.cn/04204.Doc
ecc.jyw669.cn/46008.Doc
ecc.jyw669.cn/28460.Doc
ecc.jyw669.cn/88460.Doc
ecc.jyw669.cn/66646.Doc
ecx.jyw669.cn/40268.Doc
ecx.jyw669.cn/88444.Doc
ecx.jyw669.cn/42088.Doc
ecx.jyw669.cn/62824.Doc
ecx.jyw669.cn/46666.Doc
ecx.jyw669.cn/68206.Doc
ecx.jyw669.cn/62462.Doc
ecx.jyw669.cn/44688.Doc
ecx.jyw669.cn/44206.Doc
ecx.jyw669.cn/20622.Doc
ecz.jyw669.cn/02040.Doc
ecz.jyw669.cn/82444.Doc
ecz.jyw669.cn/06640.Doc
ecz.jyw669.cn/20846.Doc
ecz.jyw669.cn/26440.Doc
ecz.jyw669.cn/48840.Doc
ecz.jyw669.cn/40626.Doc
ecz.jyw669.cn/29151.Doc
ecz.jyw669.cn/86204.Doc
ecz.jyw669.cn/02664.Doc
ecl.jyw669.cn/82466.Doc
ecl.jyw669.cn/88628.Doc
ecl.jyw669.cn/40888.Doc
ecl.jyw669.cn/80880.Doc
ecl.jyw669.cn/22602.Doc
ecl.jyw669.cn/06268.Doc
ecl.jyw669.cn/44002.Doc
ecl.jyw669.cn/28008.Doc
ecl.jyw669.cn/62660.Doc
ecl.jyw669.cn/02464.Doc
eck.jyw669.cn/28046.Doc
eck.jyw669.cn/62060.Doc
eck.jyw669.cn/08660.Doc
eck.jyw669.cn/82028.Doc
eck.jyw669.cn/80662.Doc
eck.jyw669.cn/64864.Doc
eck.jyw669.cn/82480.Doc
eck.jyw669.cn/04802.Doc
eck.jyw669.cn/62848.Doc
eck.jyw669.cn/84062.Doc
ecj.jyw669.cn/40686.Doc
ecj.jyw669.cn/66284.Doc
ecj.jyw669.cn/95239.Doc
ecj.jyw669.cn/40864.Doc
ecj.jyw669.cn/68604.Doc
ecj.jyw669.cn/82684.Doc
ecj.jyw669.cn/48442.Doc
ecj.jyw669.cn/60686.Doc
ecj.jyw669.cn/48828.Doc
ecj.jyw669.cn/48006.Doc
ech.jyw669.cn/26022.Doc
ech.jyw669.cn/48006.Doc
ech.jyw669.cn/86240.Doc
ech.jyw669.cn/80080.Doc
ech.jyw669.cn/86400.Doc
ech.jyw669.cn/88420.Doc
ech.jyw669.cn/42688.Doc
ech.jyw669.cn/66040.Doc
ech.jyw669.cn/24404.Doc
ech.jyw669.cn/42624.Doc
ecg.jyw669.cn/04486.Doc
ecg.jyw669.cn/08424.Doc
ecg.jyw669.cn/02864.Doc
ecg.jyw669.cn/08484.Doc
ecg.jyw669.cn/40026.Doc
ecg.jyw669.cn/44848.Doc
ecg.jyw669.cn/00293.Doc
ecg.jyw669.cn/40264.Doc
ecg.jyw669.cn/00862.Doc
ecg.jyw669.cn/48226.Doc
ecf.jyw669.cn/20446.Doc
ecf.jyw669.cn/60828.Doc
ecf.jyw669.cn/22422.Doc
ecf.jyw669.cn/60822.Doc
ecf.jyw669.cn/88068.Doc
ecf.jyw669.cn/46806.Doc
ecf.jyw669.cn/86884.Doc
ecf.jyw669.cn/44042.Doc
ecf.jyw669.cn/82648.Doc
ecf.jyw669.cn/66064.Doc
ecd.jyw669.cn/42282.Doc
ecd.jyw669.cn/24424.Doc
ecd.jyw669.cn/42228.Doc
ecd.jyw669.cn/84222.Doc
ecd.jyw669.cn/46200.Doc
ecd.jyw669.cn/26642.Doc
ecd.jyw669.cn/02408.Doc
ecd.jyw669.cn/66088.Doc
ecd.jyw669.cn/04220.Doc
ecd.jyw669.cn/28884.Doc
ecs.jyw669.cn/20000.Doc
ecs.jyw669.cn/21840.Doc
ecs.jyw669.cn/84268.Doc
ecs.jyw669.cn/62624.Doc
ecs.jyw669.cn/64844.Doc
ecs.jyw669.cn/60440.Doc
ecs.jyw669.cn/84260.Doc
ecs.jyw669.cn/60486.Doc
ecs.jyw669.cn/62862.Doc
ecs.jyw669.cn/80680.Doc
eca.jyw669.cn/40020.Doc
eca.jyw669.cn/22640.Doc
eca.jyw669.cn/66666.Doc
eca.jyw669.cn/46642.Doc
eca.jyw669.cn/22606.Doc
eca.jyw669.cn/22268.Doc
eca.jyw669.cn/44622.Doc
eca.jyw669.cn/22208.Doc
eca.jyw669.cn/28404.Doc
eca.jyw669.cn/40024.Doc
ecp.jyw669.cn/48024.Doc
ecp.jyw669.cn/62460.Doc
ecp.jyw669.cn/46868.Doc
ecp.jyw669.cn/22606.Doc
ecp.jyw669.cn/66286.Doc
ecp.jyw669.cn/48662.Doc
ecp.jyw669.cn/28824.Doc
ecp.jyw669.cn/86666.Doc
ecp.jyw669.cn/82804.Doc
ecp.jyw669.cn/80248.Doc
eco.jyw669.cn/20426.Doc
eco.jyw669.cn/02228.Doc
eco.jyw669.cn/44264.Doc
eco.jyw669.cn/46484.Doc
eco.jyw669.cn/68608.Doc
eco.jyw669.cn/46480.Doc
eco.jyw669.cn/62422.Doc
eco.jyw669.cn/66644.Doc
eco.jyw669.cn/66806.Doc
eco.jyw669.cn/02280.Doc
eci.jyw669.cn/02880.Doc
eci.jyw669.cn/66428.Doc
eci.jyw669.cn/28246.Doc
eci.jyw669.cn/48448.Doc
eci.jyw669.cn/06242.Doc
eci.jyw669.cn/86200.Doc
eci.jyw669.cn/08446.Doc
eci.jyw669.cn/08084.Doc
eci.jyw669.cn/88200.Doc
eci.jyw669.cn/62486.Doc
ecu.jyw669.cn/08840.Doc
ecu.jyw669.cn/86424.Doc
ecu.jyw669.cn/60820.Doc
ecu.jyw669.cn/66068.Doc
ecu.jyw669.cn/22448.Doc
ecu.jyw669.cn/48240.Doc
ecu.jyw669.cn/22620.Doc
ecu.jyw669.cn/06628.Doc
ecu.jyw669.cn/48882.Doc
ecu.jyw669.cn/26462.Doc
ecy.jyw669.cn/22208.Doc
ecy.jyw669.cn/68202.Doc
ecy.jyw669.cn/66224.Doc
ecy.jyw669.cn/22466.Doc
ecy.jyw669.cn/82086.Doc
ecy.jyw669.cn/48422.Doc
ecy.jyw669.cn/22244.Doc
ecy.jyw669.cn/80480.Doc
ecy.jyw669.cn/84280.Doc
ecy.jyw669.cn/04880.Doc
ect.jyw669.cn/24242.Doc
ect.jyw669.cn/06422.Doc
ect.jyw669.cn/20268.Doc
ect.jyw669.cn/42068.Doc
ect.jyw669.cn/48080.Doc
ect.jyw669.cn/22860.Doc
ect.jyw669.cn/00646.Doc
ect.jyw669.cn/82262.Doc
ect.jyw669.cn/20464.Doc
ect.jyw669.cn/00048.Doc
ecr.jyw669.cn/82264.Doc
ecr.jyw669.cn/22066.Doc
ecr.jyw669.cn/86220.Doc
ecr.jyw669.cn/22606.Doc
ecr.jyw669.cn/04204.Doc
ecr.jyw669.cn/06846.Doc
ecr.jyw669.cn/28022.Doc
ecr.jyw669.cn/06062.Doc
ecr.jyw669.cn/64444.Doc
ecr.jyw669.cn/69176.Doc
ece.jyw669.cn/64222.Doc
ece.jyw669.cn/00482.Doc
ece.jyw669.cn/80604.Doc
ece.jyw669.cn/82648.Doc
ece.jyw669.cn/00244.Doc
ece.jyw669.cn/42159.Doc
ece.jyw669.cn/60024.Doc
ece.jyw669.cn/82266.Doc
ece.jyw669.cn/66028.Doc
ece.jyw669.cn/06466.Doc
ecw.jyw669.cn/20800.Doc
ecw.jyw669.cn/40802.Doc
ecw.jyw669.cn/28204.Doc
ecw.jyw669.cn/24864.Doc
ecw.jyw669.cn/04200.Doc
ecw.jyw669.cn/96690.Doc
ecw.jyw669.cn/08446.Doc
ecw.jyw669.cn/64868.Doc
ecw.jyw669.cn/04242.Doc
ecw.jyw669.cn/44260.Doc
ecq.jyw669.cn/84482.Doc
ecq.jyw669.cn/24826.Doc
ecq.jyw669.cn/62466.Doc
ecq.jyw669.cn/86046.Doc
ecq.jyw669.cn/66642.Doc
ecq.jyw669.cn/82842.Doc
ecq.jyw669.cn/44895.Doc
ecq.jyw669.cn/04923.Doc
ecq.jyw669.cn/68002.Doc
ecq.jyw669.cn/46026.Doc
