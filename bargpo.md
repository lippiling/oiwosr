盘屹抗繁泼


Python 测试运行器 Nox 详解
===================================

Nox 是灵活的 Python 测试运行器，通过 noxfile.py 定义多个测试会话，
自动为每个会话创建独立的虚拟环境，适用于多版本测试。

一、安装与基本概念
------------------

# 安装 Nox
# pip install nox

# 验证安装
nox --version

# Nox 的核心概念：
# · Session（会话）：一个测试任务，在独立虚拟环境中运行
# · noxfile.py：定义所有会话的配置文件
# · 每个会话可以指定不同的 Python 版本和依赖

# 列出所有可用的会话
nox --list

二、noxfile.py 基础
-------------------

# 基本的 noxfile.py 文件
"""
import nox

# 指定会话使用的 Python 版本
@nox.session(python=["3.9", "3.10", "3.11"])
def tests(session):
    # 安装依赖
    session.install("pytest", "pytest-cov")
    # 安装项目本身
    session.install(".")
    # 运行命令
    session.run("pytest", "tests/", "--cov=src")

@nox.session
def lint(session):
    # 代码检查会话
    session.install("ruff")
    session.run("ruff", "check", "src/")

@nox.session
def type_check(session):
    # 类型检查会话
    session.install("mypy")
    session.install(".")
    session.run("mypy", "src/")
"""

# 运行所有会话
nox

# 运行特定会话
nox -s tests

# 运行多个会话
nox -s tests lint

三、高级会话配置
----------------

# 参数化会话与精细控制
"""
import nox

# 使用参数化标签
@nox.session(tags=["core"])
@nox.parametrize("django", ["3.2", "4.0", "4.1"])
def tests(session, django):
    # 安装特定版本的 Django
    session.install(f"django=={django}")
    session.install("pytest", "pytest-django")
    session.install(".")
    session.run("pytest", "tests/")

@nox.session(python="3.11", venv_backend="conda")
def conda_tests(session):
    # 使用 Conda 创建虚拟环境
    session.install("numpy", "scipy")
    session.run("pytest", "tests/")
"""

# 查看参数化会话
nox --list

# 按标签运行会话
nox --tags core

四、虚拟环境管理
----------------

# Nox 为每个会话自动创建和管理虚拟环境

# 配置虚拟环境后端
"""
import nox

# 可选的 venv_backend 值：
# · "virtualenv" （默认）
# · "conda"
# · "venv"

@nox.session(venv_backend="virtualenv")
def example(session):
    # 默认后端，需要安装 virtualenv
    session.install("pytest")

@nox.session(venv_backend="venv")
def stdlib_venv(session):
    # 使用 Python 标准库 venv
    session.install("pytest")
"""

# 复用虚拟环境（加快重复运行速度）
nox -s tests --reuse-existing-virtualenvs

# 强制重新创建虚拟环境
nox -s tests --force-venv

五、会话高级功能
----------------

# 更多装饰器参数和辅助方法
"""
import nox

# 不指定 Python 版本（使用当前环境）
@nox.session(python=False)
def build_docs(session):
    # 不创建虚拟环境，使用系统 Python
    session.run("sphinx-build", "-b", "html", "docs/", "docs/_build")

@nox.session
def coverage(session):
    # 安装依赖
    session.install("pytest", "pytest-cov", "coverage")
    session.install(".")
    # 运行测试并生成覆盖率报告
    session.run(
        "pytest", "tests/",
        "--cov=src",
        "--cov-report=html",
        "--cov-report=term-missing",
    )
    # 多个命令连续执行
    session.run("coverage", "report", "--fail-under=80")

@nox.session
def integration(session):
    # 集成测试会话
    session.install("-r", "requirements-dev.txt")
    session.install(".")
    # 设置环境变量
    session.env["DATABASE_URL"] = "sqlite:///test.db"
    session.env["DEBUG"] = "true"
    session.run("pytest", "tests/integration/", "-v")
"""

六、reuse_venv 与缓存
---------------------

# 控制虚拟环境缓存行为
"""
import nox

# 默认情况下每次运行都会重新创建虚拟环境
# 通过命令行参数控制：
# nox -s tests --reuse-existing-virtualenvs

# 在 noxfile 中设置默认行为
nox.options.reuse_existing_virtualenvs = True

# 或通过环境变量控制
# export NOX_REUSE_VENV=1

@nox.session
def fast_tests(session):
    # 安装依赖
    session.install("pytest")
    session.install(".")
    session.run("pytest", "tests/unit/", "-x")
"""

七、与 CI/CD 集成
------------------

# GitHub Actions 配置示例
"""
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install nox
      - run: nox -s tests --python ${{ matrix.python-version }}
"""

# 只运行未通过的会话
nox -s tests --error-on-missing-interpreters

# 并行运行会话
nox -s tests lint type_check --parallel

# Nox 让多版本测试变得简单可靠，每个会话的隔离性
# 确保测试环境纯净，是 CI 管道的得力助手。

八猜卮琴厝八椿谧式诜匝土毙案贺

m.wfe.hxbsg.cn/91755.Doc
m.wfe.hxbsg.cn/20846.Doc
m.wfe.hxbsg.cn/73951.Doc
m.wfe.hxbsg.cn/48408.Doc
m.wfe.hxbsg.cn/60406.Doc
m.wfe.hxbsg.cn/80888.Doc
m.wfe.hxbsg.cn/33511.Doc
m.wfe.hxbsg.cn/80826.Doc
m.wfe.hxbsg.cn/75137.Doc
m.wfe.hxbsg.cn/95751.Doc
m.wfe.hxbsg.cn/66020.Doc
m.wfe.hxbsg.cn/95797.Doc
m.wfe.hxbsg.cn/20246.Doc
m.wfe.hxbsg.cn/20228.Doc
m.wfe.hxbsg.cn/39355.Doc
m.wfw.hxbsg.cn/82268.Doc
m.wfw.hxbsg.cn/37931.Doc
m.wfw.hxbsg.cn/06464.Doc
m.wfw.hxbsg.cn/59335.Doc
m.wfw.hxbsg.cn/95713.Doc
m.wfw.hxbsg.cn/93911.Doc
m.wfw.hxbsg.cn/35717.Doc
m.wfw.hxbsg.cn/40664.Doc
m.wfw.hxbsg.cn/88686.Doc
m.wfw.hxbsg.cn/19119.Doc
m.wfw.hxbsg.cn/71775.Doc
m.wfw.hxbsg.cn/91917.Doc
m.wfw.hxbsg.cn/62824.Doc
m.wfw.hxbsg.cn/79111.Doc
m.wfw.hxbsg.cn/28042.Doc
m.wfw.hxbsg.cn/37959.Doc
m.wfw.hxbsg.cn/80800.Doc
m.wfw.hxbsg.cn/17717.Doc
m.wfw.hxbsg.cn/31731.Doc
m.wfw.hxbsg.cn/99597.Doc
m.wfq.hxbsg.cn/95311.Doc
m.wfq.hxbsg.cn/24288.Doc
m.wfq.hxbsg.cn/26682.Doc
m.wfq.hxbsg.cn/13199.Doc
m.wfq.hxbsg.cn/88662.Doc
m.wfq.hxbsg.cn/42664.Doc
m.wfq.hxbsg.cn/79555.Doc
m.wfq.hxbsg.cn/79355.Doc
m.wfq.hxbsg.cn/22288.Doc
m.wfq.hxbsg.cn/53177.Doc
m.wfq.hxbsg.cn/53171.Doc
m.wfq.hxbsg.cn/73599.Doc
m.wfq.hxbsg.cn/00022.Doc
m.wfq.hxbsg.cn/77773.Doc
m.wfq.hxbsg.cn/37533.Doc
m.wfq.hxbsg.cn/77111.Doc
m.wfq.hxbsg.cn/55797.Doc
m.wfq.hxbsg.cn/44806.Doc
m.wfq.hxbsg.cn/75519.Doc
m.wfq.hxbsg.cn/04260.Doc
m.wdm.hxbsg.cn/22484.Doc
m.wdm.hxbsg.cn/08668.Doc
m.wdm.hxbsg.cn/55313.Doc
m.wdm.hxbsg.cn/55353.Doc
m.wdm.hxbsg.cn/04446.Doc
m.wdm.hxbsg.cn/13571.Doc
m.wdm.hxbsg.cn/80068.Doc
m.wdm.hxbsg.cn/24064.Doc
m.wdm.hxbsg.cn/99753.Doc
m.wdm.hxbsg.cn/57731.Doc
m.wdm.hxbsg.cn/22684.Doc
m.wdm.hxbsg.cn/55571.Doc
m.wdm.hxbsg.cn/04244.Doc
m.wdm.hxbsg.cn/48602.Doc
m.wdm.hxbsg.cn/20224.Doc
m.wdm.hxbsg.cn/59317.Doc
m.wdm.hxbsg.cn/28248.Doc
m.wdm.hxbsg.cn/04046.Doc
m.wdm.hxbsg.cn/88662.Doc
m.wdm.hxbsg.cn/53991.Doc
m.wdn.hxbsg.cn/77379.Doc
m.wdn.hxbsg.cn/48686.Doc
m.wdn.hxbsg.cn/91735.Doc
m.wdn.hxbsg.cn/11777.Doc
m.wdn.hxbsg.cn/73771.Doc
m.wdn.hxbsg.cn/35133.Doc
m.wdn.hxbsg.cn/13137.Doc
m.wdn.hxbsg.cn/37559.Doc
m.wdn.hxbsg.cn/73539.Doc
m.wdn.hxbsg.cn/84262.Doc
m.wdn.hxbsg.cn/91919.Doc
m.wdn.hxbsg.cn/37795.Doc
m.wdn.hxbsg.cn/37151.Doc
m.wdn.hxbsg.cn/84262.Doc
m.wdn.hxbsg.cn/11359.Doc
m.wdn.hxbsg.cn/35799.Doc
m.wdn.hxbsg.cn/13751.Doc
m.wdn.hxbsg.cn/86608.Doc
m.wdn.hxbsg.cn/02826.Doc
m.wdn.hxbsg.cn/35753.Doc
m.wdb.hxbsg.cn/08466.Doc
m.wdb.hxbsg.cn/57599.Doc
m.wdb.hxbsg.cn/53777.Doc
m.wdb.hxbsg.cn/62600.Doc
m.wdb.hxbsg.cn/35979.Doc
m.wdb.hxbsg.cn/99331.Doc
m.wdb.hxbsg.cn/40446.Doc
m.wdb.hxbsg.cn/37115.Doc
m.wdb.hxbsg.cn/46660.Doc
m.wdb.hxbsg.cn/82466.Doc
m.wdb.hxbsg.cn/99195.Doc
m.wdb.hxbsg.cn/73515.Doc
m.wdb.hxbsg.cn/84824.Doc
m.wdb.hxbsg.cn/53953.Doc
m.wdb.hxbsg.cn/15575.Doc
m.wdb.hxbsg.cn/66280.Doc
m.wdb.hxbsg.cn/82404.Doc
m.wdb.hxbsg.cn/91917.Doc
m.wdb.hxbsg.cn/24248.Doc
m.wdb.hxbsg.cn/99951.Doc
m.wdv.hxbsg.cn/93519.Doc
m.wdv.hxbsg.cn/84466.Doc
m.wdv.hxbsg.cn/42220.Doc
m.wdv.hxbsg.cn/53915.Doc
m.wdv.hxbsg.cn/44848.Doc
m.wdv.hxbsg.cn/37179.Doc
m.wdv.hxbsg.cn/48222.Doc
m.wdv.hxbsg.cn/80824.Doc
m.wdv.hxbsg.cn/51315.Doc
m.wdv.hxbsg.cn/93591.Doc
m.wdv.hxbsg.cn/44660.Doc
m.wdv.hxbsg.cn/40464.Doc
m.wdv.hxbsg.cn/59115.Doc
m.wdv.hxbsg.cn/19197.Doc
m.wdv.hxbsg.cn/71173.Doc
m.wdv.hxbsg.cn/51179.Doc
m.wdv.hxbsg.cn/64686.Doc
m.wdv.hxbsg.cn/37995.Doc
m.wdv.hxbsg.cn/53339.Doc
m.wdv.hxbsg.cn/82640.Doc
m.wdc.hxbsg.cn/00084.Doc
m.wdc.hxbsg.cn/40446.Doc
m.wdc.hxbsg.cn/17999.Doc
m.wdc.hxbsg.cn/60880.Doc
m.wdc.hxbsg.cn/66288.Doc
m.wdc.hxbsg.cn/79339.Doc
m.wdc.hxbsg.cn/95173.Doc
m.wdc.hxbsg.cn/08088.Doc
m.wdc.hxbsg.cn/37977.Doc
m.wdc.hxbsg.cn/31399.Doc
m.wdc.hxbsg.cn/73559.Doc
m.wdc.hxbsg.cn/26464.Doc
m.wdc.hxbsg.cn/57391.Doc
m.wdc.hxbsg.cn/57531.Doc
m.wdc.hxbsg.cn/20444.Doc
m.wdc.hxbsg.cn/80646.Doc
m.wdc.hxbsg.cn/79391.Doc
m.wdc.hxbsg.cn/06864.Doc
m.wdc.hxbsg.cn/37753.Doc
m.wdc.hxbsg.cn/08244.Doc
m.wdx.hxbsg.cn/04080.Doc
m.wdx.hxbsg.cn/44662.Doc
m.wdx.hxbsg.cn/99533.Doc
m.wdx.hxbsg.cn/22860.Doc
m.wdx.hxbsg.cn/88264.Doc
m.wdx.hxbsg.cn/93735.Doc
m.wdx.hxbsg.cn/37375.Doc
m.wdx.hxbsg.cn/22202.Doc
m.wdx.hxbsg.cn/15959.Doc
m.wdx.hxbsg.cn/33755.Doc
m.wdx.hxbsg.cn/13595.Doc
m.wdx.hxbsg.cn/11553.Doc
m.wdx.hxbsg.cn/80002.Doc
m.wdx.hxbsg.cn/82824.Doc
m.wdx.hxbsg.cn/33555.Doc
m.wdx.hxbsg.cn/55953.Doc
m.wdx.hxbsg.cn/28426.Doc
m.wdx.hxbsg.cn/46440.Doc
m.wdx.hxbsg.cn/39731.Doc
m.wdx.hxbsg.cn/82460.Doc
m.wdz.hxbsg.cn/91591.Doc
m.wdz.hxbsg.cn/75571.Doc
m.wdz.hxbsg.cn/42846.Doc
m.wdz.hxbsg.cn/99951.Doc
m.wdz.hxbsg.cn/71137.Doc
m.wdz.hxbsg.cn/73373.Doc
m.wdz.hxbsg.cn/59557.Doc
m.wdz.hxbsg.cn/22082.Doc
m.wdz.hxbsg.cn/88806.Doc
m.wdz.hxbsg.cn/91333.Doc
m.wdz.hxbsg.cn/99793.Doc
m.wdz.hxbsg.cn/80424.Doc
m.wdz.hxbsg.cn/39939.Doc
m.wdz.hxbsg.cn/77979.Doc
m.wdz.hxbsg.cn/33371.Doc
m.wdz.hxbsg.cn/86248.Doc
m.wdz.hxbsg.cn/35971.Doc
m.wdz.hxbsg.cn/55353.Doc
m.wdz.hxbsg.cn/93771.Doc
m.wdz.hxbsg.cn/04462.Doc
m.wdl.hxbsg.cn/15351.Doc
m.wdl.hxbsg.cn/19537.Doc
m.wdl.hxbsg.cn/24000.Doc
m.wdl.hxbsg.cn/75799.Doc
m.wdl.hxbsg.cn/00002.Doc
m.wdl.hxbsg.cn/75959.Doc
m.wdl.hxbsg.cn/95173.Doc
m.wdl.hxbsg.cn/28624.Doc
m.wdl.hxbsg.cn/91757.Doc
m.wdl.hxbsg.cn/71777.Doc
m.wdl.hxbsg.cn/95177.Doc
m.wdl.hxbsg.cn/97373.Doc
m.wdl.hxbsg.cn/06628.Doc
m.wdl.hxbsg.cn/93911.Doc
m.wdl.hxbsg.cn/80004.Doc
m.wdl.hxbsg.cn/62648.Doc
m.wdl.hxbsg.cn/39795.Doc
m.wdl.hxbsg.cn/24462.Doc
m.wdl.hxbsg.cn/55355.Doc
m.wdl.hxbsg.cn/40204.Doc
m.wdk.hxbsg.cn/82042.Doc
m.wdk.hxbsg.cn/82440.Doc
m.wdk.hxbsg.cn/35175.Doc
m.wdk.hxbsg.cn/06262.Doc
m.wdk.hxbsg.cn/22448.Doc
m.wdk.hxbsg.cn/75573.Doc
m.wdk.hxbsg.cn/80642.Doc
m.wdk.hxbsg.cn/51311.Doc
m.wdk.hxbsg.cn/99359.Doc
m.wdk.hxbsg.cn/15739.Doc
m.wdk.hxbsg.cn/48088.Doc
m.wdk.hxbsg.cn/53535.Doc
m.wdk.hxbsg.cn/17775.Doc
m.wdk.hxbsg.cn/64662.Doc
m.wdk.hxbsg.cn/37999.Doc
m.wdk.hxbsg.cn/35155.Doc
m.wdk.hxbsg.cn/02246.Doc
m.wdk.hxbsg.cn/22248.Doc
m.wdk.hxbsg.cn/31115.Doc
m.wdk.hxbsg.cn/68446.Doc
m.wdj.hxbsg.cn/48286.Doc
m.wdj.hxbsg.cn/33577.Doc
m.wdj.hxbsg.cn/02006.Doc
m.wdj.hxbsg.cn/20404.Doc
m.wdj.hxbsg.cn/20402.Doc
m.wdj.hxbsg.cn/35313.Doc
m.wdj.hxbsg.cn/62446.Doc
m.wdj.hxbsg.cn/48482.Doc
m.wdj.hxbsg.cn/13135.Doc
m.wdj.hxbsg.cn/86844.Doc
m.wdj.hxbsg.cn/48026.Doc
m.wdj.hxbsg.cn/91139.Doc
m.wdj.hxbsg.cn/20804.Doc
m.wdj.hxbsg.cn/46624.Doc
m.wdj.hxbsg.cn/24480.Doc
m.wdj.hxbsg.cn/97519.Doc
m.wdj.hxbsg.cn/82200.Doc
m.wdj.hxbsg.cn/95135.Doc
m.wdj.hxbsg.cn/99151.Doc
m.wdj.hxbsg.cn/48804.Doc
m.wdh.hxbsg.cn/48824.Doc
m.wdh.hxbsg.cn/17917.Doc
m.wdh.hxbsg.cn/37173.Doc
m.wdh.hxbsg.cn/22266.Doc
m.wdh.hxbsg.cn/62666.Doc
m.wdh.hxbsg.cn/71977.Doc
m.wdh.hxbsg.cn/22202.Doc
m.wdh.hxbsg.cn/28446.Doc
m.wdh.hxbsg.cn/64800.Doc
m.wdh.hxbsg.cn/95377.Doc
m.wdh.hxbsg.cn/40668.Doc
m.wdh.hxbsg.cn/02400.Doc
m.wdh.hxbsg.cn/73153.Doc
m.wdh.hxbsg.cn/22260.Doc
m.wdh.hxbsg.cn/35319.Doc
m.wdh.hxbsg.cn/80804.Doc
m.wdh.hxbsg.cn/59193.Doc
m.wdh.hxbsg.cn/79771.Doc
m.wdh.hxbsg.cn/64404.Doc
m.wdh.hxbsg.cn/55953.Doc
m.wdg.hxbsg.cn/02648.Doc
m.wdg.hxbsg.cn/55977.Doc
m.wdg.hxbsg.cn/99357.Doc
m.wdg.hxbsg.cn/08200.Doc
m.wdg.hxbsg.cn/33577.Doc
m.wdg.hxbsg.cn/82824.Doc
m.wdg.hxbsg.cn/60408.Doc
m.wdg.hxbsg.cn/97595.Doc
m.wdg.hxbsg.cn/80626.Doc
m.wdg.hxbsg.cn/02046.Doc
m.wdg.hxbsg.cn/31795.Doc
m.wdg.hxbsg.cn/17395.Doc
m.wdg.hxbsg.cn/48000.Doc
m.wdg.hxbsg.cn/91711.Doc
m.wdg.hxbsg.cn/95157.Doc
m.wdg.hxbsg.cn/73357.Doc
m.wdg.hxbsg.cn/59153.Doc
m.wdg.hxbsg.cn/28640.Doc
m.wdg.hxbsg.cn/48446.Doc
m.wdg.hxbsg.cn/53313.Doc
m.wdf.hxbsg.cn/66080.Doc
m.wdf.hxbsg.cn/26022.Doc
m.wdf.hxbsg.cn/93973.Doc
m.wdf.hxbsg.cn/60820.Doc
m.wdf.hxbsg.cn/20866.Doc
m.wdf.hxbsg.cn/77915.Doc
m.wdf.hxbsg.cn/11191.Doc
m.wdf.hxbsg.cn/95935.Doc
m.wdf.hxbsg.cn/71935.Doc
m.wdf.hxbsg.cn/00624.Doc
m.wdf.hxbsg.cn/97115.Doc
m.wdf.hxbsg.cn/71199.Doc
m.wdf.hxbsg.cn/48444.Doc
m.wdf.hxbsg.cn/62040.Doc
m.wdf.hxbsg.cn/33599.Doc
m.wdf.hxbsg.cn/64202.Doc
m.wdf.hxbsg.cn/68082.Doc
m.wdf.hxbsg.cn/13955.Doc
m.wdf.hxbsg.cn/59179.Doc
m.wdf.hxbsg.cn/51739.Doc
m.wdd.hxbsg.cn/77571.Doc
m.wdd.hxbsg.cn/26886.Doc
m.wdd.hxbsg.cn/39757.Doc
m.wdd.hxbsg.cn/73731.Doc
m.wdd.hxbsg.cn/66668.Doc
m.wdd.hxbsg.cn/73577.Doc
m.wdd.hxbsg.cn/97175.Doc
m.wdd.hxbsg.cn/04802.Doc
m.wdd.hxbsg.cn/11711.Doc
m.wdd.hxbsg.cn/95373.Doc
m.wdd.hxbsg.cn/04002.Doc
m.wdd.hxbsg.cn/68408.Doc
m.wdd.hxbsg.cn/51595.Doc
m.wdd.hxbsg.cn/84648.Doc
m.wdd.hxbsg.cn/53391.Doc
m.wdd.hxbsg.cn/95355.Doc
m.wdd.hxbsg.cn/66242.Doc
m.wdd.hxbsg.cn/51199.Doc
m.wdd.hxbsg.cn/44208.Doc
m.wdd.hxbsg.cn/51155.Doc
m.wds.hxbsg.cn/51733.Doc
m.wds.hxbsg.cn/59715.Doc
m.wds.hxbsg.cn/86042.Doc
m.wds.hxbsg.cn/42202.Doc
m.wds.hxbsg.cn/57771.Doc
m.wds.hxbsg.cn/48880.Doc
m.wds.hxbsg.cn/64440.Doc
m.wds.hxbsg.cn/13353.Doc
m.wds.hxbsg.cn/62002.Doc
m.wds.hxbsg.cn/46866.Doc
m.wds.hxbsg.cn/17973.Doc
m.wds.hxbsg.cn/26266.Doc
m.wds.hxbsg.cn/31777.Doc
m.wds.hxbsg.cn/11771.Doc
m.wds.hxbsg.cn/75539.Doc
m.wds.hxbsg.cn/08026.Doc
m.wds.hxbsg.cn/71339.Doc
m.wds.hxbsg.cn/08880.Doc
m.wds.hxbsg.cn/24424.Doc
m.wds.hxbsg.cn/91153.Doc
m.wda.hxbsg.cn/97795.Doc
m.wda.hxbsg.cn/60444.Doc
m.wda.hxbsg.cn/5.Doc
m.wda.hxbsg.cn/24844.Doc
m.wda.hxbsg.cn/66408.Doc
m.wda.hxbsg.cn/55539.Doc
m.wda.hxbsg.cn/62824.Doc
m.wda.hxbsg.cn/80860.Doc
m.wda.hxbsg.cn/17571.Doc
m.wda.hxbsg.cn/26446.Doc
m.wda.hxbsg.cn/79359.Doc
m.wda.hxbsg.cn/17315.Doc
m.wda.hxbsg.cn/08486.Doc
m.wda.hxbsg.cn/37713.Doc
m.wda.hxbsg.cn/88220.Doc
m.wda.hxbsg.cn/28800.Doc
m.wda.hxbsg.cn/19733.Doc
m.wda.hxbsg.cn/88466.Doc
m.wda.hxbsg.cn/22606.Doc
m.wda.hxbsg.cn/71373.Doc
m.wdp.hxbsg.cn/08428.Doc
m.wdp.hxbsg.cn/39311.Doc
m.wdp.hxbsg.cn/57311.Doc
m.wdp.hxbsg.cn/62020.Doc
m.wdp.hxbsg.cn/22664.Doc
m.wdp.hxbsg.cn/11557.Doc
m.wdp.hxbsg.cn/95599.Doc
m.wdp.hxbsg.cn/51575.Doc
m.wdp.hxbsg.cn/93173.Doc
m.wdp.hxbsg.cn/97199.Doc
m.wdp.hxbsg.cn/86066.Doc
m.wdp.hxbsg.cn/88044.Doc
m.wdp.hxbsg.cn/11391.Doc
m.wdp.hxbsg.cn/35155.Doc
m.wdp.hxbsg.cn/82808.Doc
m.wdp.hxbsg.cn/22440.Doc
m.wdp.hxbsg.cn/33995.Doc
m.wdp.hxbsg.cn/62860.Doc
m.wdp.hxbsg.cn/26424.Doc
m.wdp.hxbsg.cn/39197.Doc
