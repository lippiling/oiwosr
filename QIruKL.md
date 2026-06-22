炔琶植毫偬


Python 代码格式化 Ruff 详解
====================================

Ruff 是 Astral 公司用 Rust 开发的极速 Python 静态检查与格式化工具，
兼容 Flake8 规则集，目标是统一替代 Black、isort、Flake8 等工具。

一、安装与基本用法
------------------

# 安装 Ruff
# pip install ruff

# 验证安装
ruff --version

# 检查当前目录所有 Python 文件
ruff check .

# 自动修复可修复的问题
ruff check --fix .

# 格式化代码（替代 Black）
ruff format .

# 同时检查并格式化
ruff check . && ruff format .

二、pyproject.toml 配置
------------------------

# 在 pyproject.toml 中配置 Ruff
"""
[tool.ruff]
# 目标 Python 版本
target-version = "py311"

# 每行最大字符数
line-length = 88

# 排除的文件和目录
exclude = ["__pycache__", ".git", "node_modules"]

# 允许使用制表符
indent-width = 4

[tool.ruff.lint]
# 启用的规则集
select = [
    "E",    # pycodestyle 错误
    "W",    # pycodestyle 警告
    "F",    # Pyflakes
    "I",    # isort 导入排序
    "N",    # 命名约定
    "D",    # pydocstyle 文档字符串
    "UP",   # pyupgrade 升级语法
    "B",    # flake8-bugbear 潜在 bug
]

# 忽略的特定规则
ignore = [
    "D100",  # 模块缺少文档字符串
    "D101",  # 类缺少文档字符串
]

# 允许特定规则用于测试文件
[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["D", "N"]
"""

三、规则选择与忽略
------------------

# Ruff 内置了超过 800 条检查规则，按前缀分组

# 命令行选择规则
ruff check --select E,W,F --ignore E501 .

# E501 表示行太长（默认忽略行长度检查）

# 在代码中局部忽略
# 在代码行末尾使用 noqa 注释
import os  # noqa: F401  忽略未使用的导入

# 忽略整个文件的特定规则
"""
# file: example.py
# ruff: noqa: D100, D101
"""

# 在配置中按文件类型忽略
"""
[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]
"tests/*.py" = ["D", "S101"]
"""

四、与 Black 的对比
--------------------

# Ruff 格式化和 Black 的主要区别：

# 1. 速度：Ruff 快 10-100 倍（Rust vs Python）
# 2. 配置：Ruff 提供更多配置选项，Black 近乎零配置
# 3. 兼容性：Ruff 尽可能兼容 Black 的输出风格

# 使用 Black 兼容风格（默认已启用）
"""
[tool.ruff.format]
# 使用与 Black 兼容的引号风格
quote-style = "double"

# 与 Black 一致的缩进风格
indent-style = "space"

# 行尾换行符风格
line-ending = "auto"

# 文档字符串格式化为与 Black 一致
docstring-code-format = true
"""

五、导入排序（替代 isort）
--------------------------

# Ruff 内置了导入排序功能，完全替代 isort

# 配置导入排序风格
"""
[tool.ruff.lint.isort]
# 按长度排序：即将常导入放在前面
length-sort = false

# 强制单行导入（每个 import 独占一行）
force-single-line = false

# 强制将导入包裹在括号内
force-wrap-aliases = false

# 合并同名模块的导入
combine-as-imports = true

# 第三方库已知列表
known-third-party = ["pandas", "numpy", "django"]
"""

# 检查导入排序
ruff check --select I .

# 自动修复导入排序
ruff check --select I --fix .

六、与 pre-commit 集成
----------------------

# 在 pre-commit 配置中使用 Ruff
"""
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format
"""

七、持续集成中的使用
--------------------

# GitHub Actions 配置示例
"""
name: Lint
on: [push, pull_request]
jobs:
  ruff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
      - run: pip install ruff
      - run: ruff check --output-format=github .
      - run: ruff format --check .
"""

# CI 中检查格式化而不修改
ruff format --check .

# 输出 GitHub Annotation 格式的错误
ruff check --output-format=github .

八、缓存与性能优化
------------------

# Ruff 使用增量缓存加速重复运行

# 查看缓存目录
ruff cache dir

# 清理缓存
ruff clean

# 使用较少的 CPU 核心以节省资源
ruff check --threads 2 .

# 只检查修改过的文件（自动检测）
ruff check --show-fixes .

# 监控模式：文件变化时自动检查
ruff check --watch .

# Ruff 的出现彻底改变了 Python 代码检查的效率体验，
# 将原本需要几秒甚至十几秒的检查缩短到毫秒级。

僖舱纬烧恍兹刮分沸采瓮好老沦泻

rwi.hjiocz.cn/517393.htm
rwi.hjiocz.cn/460283.htm
rwi.hjiocz.cn/404463.htm
rwi.hjiocz.cn/155953.htm
rwi.hjiocz.cn/373133.htm
rwu.hjiocz.cn/333173.htm
rwu.hjiocz.cn/246683.htm
rwu.hjiocz.cn/444843.htm
rwu.hjiocz.cn/266803.htm
rwu.hjiocz.cn/157153.htm
rwu.hjiocz.cn/959373.htm
rwu.hjiocz.cn/999113.htm
rwu.hjiocz.cn/597313.htm
rwu.hjiocz.cn/555113.htm
rwu.hjiocz.cn/53.htm
rwy.hjiocz.cn/313193.htm
rwy.hjiocz.cn/553973.htm
rwy.hjiocz.cn/046003.htm
rwy.hjiocz.cn/220223.htm
rwy.hjiocz.cn/224403.htm
rwy.hjiocz.cn/731153.htm
rwy.hjiocz.cn/357373.htm
rwy.hjiocz.cn/406403.htm
rwy.hjiocz.cn/951713.htm
rwy.hjiocz.cn/799793.htm
rwt.hjiocz.cn/602003.htm
rwt.hjiocz.cn/537593.htm
rwt.hjiocz.cn/400823.htm
rwt.hjiocz.cn/044023.htm
rwt.hjiocz.cn/066203.htm
rwt.hjiocz.cn/048803.htm
rwt.hjiocz.cn/024243.htm
rwt.hjiocz.cn/393573.htm
rwt.hjiocz.cn/175913.htm
rwt.hjiocz.cn/040223.htm
rwr.hjiocz.cn/137193.htm
rwr.hjiocz.cn/511173.htm
rwr.hjiocz.cn/240203.htm
rwr.hjiocz.cn/977733.htm
rwr.hjiocz.cn/002263.htm
rwr.hjiocz.cn/266803.htm
rwr.hjiocz.cn/202443.htm
rwr.hjiocz.cn/399713.htm
rwr.hjiocz.cn/808463.htm
rwr.hjiocz.cn/602023.htm
rwe.hjiocz.cn/244003.htm
rwe.hjiocz.cn/771393.htm
rwe.hjiocz.cn/446663.htm
rwe.hjiocz.cn/868843.htm
rwe.hjiocz.cn/062623.htm
rwe.hjiocz.cn/975573.htm
rwe.hjiocz.cn/840843.htm
rwe.hjiocz.cn/591533.htm
rwe.hjiocz.cn/519753.htm
rwe.hjiocz.cn/402443.htm
rww.hjiocz.cn/286423.htm
rww.hjiocz.cn/620023.htm
rww.hjiocz.cn/533173.htm
rww.hjiocz.cn/408623.htm
rww.hjiocz.cn/377153.htm
rww.hjiocz.cn/559133.htm
rww.hjiocz.cn/846823.htm
rww.hjiocz.cn/880003.htm
rww.hjiocz.cn/395113.htm
rww.hjiocz.cn/959933.htm
rwq.hjiocz.cn/080283.htm
rwq.hjiocz.cn/339393.htm
rwq.hjiocz.cn/048623.htm
rwq.hjiocz.cn/286203.htm
rwq.hjiocz.cn/733533.htm
rwq.hjiocz.cn/000603.htm
rwq.hjiocz.cn/333193.htm
rwq.hjiocz.cn/668683.htm
rwq.hjiocz.cn/224883.htm
rwq.hjiocz.cn/408263.htm
rqm.hjiocz.cn/068063.htm
rqm.hjiocz.cn/688003.htm
rqm.hjiocz.cn/080603.htm
rqm.hjiocz.cn/737973.htm
rqm.hjiocz.cn/088663.htm
rqm.hjiocz.cn/795913.htm
rqm.hjiocz.cn/882083.htm
rqm.hjiocz.cn/660463.htm
rqm.hjiocz.cn/115133.htm
rqm.hjiocz.cn/373513.htm
rqn.hjiocz.cn/751113.htm
rqn.hjiocz.cn/428043.htm
rqn.hjiocz.cn/113713.htm
rqn.hjiocz.cn/884203.htm
rqn.hjiocz.cn/682863.htm
rqn.hjiocz.cn/060403.htm
rqn.hjiocz.cn/802463.htm
rqn.hjiocz.cn/551313.htm
rqn.hjiocz.cn/448483.htm
rqn.hjiocz.cn/240003.htm
rqb.hjiocz.cn/084223.htm
rqb.hjiocz.cn/860263.htm
rqb.hjiocz.cn/888623.htm
rqb.hjiocz.cn/622443.htm
rqb.hjiocz.cn/957773.htm
rqb.hjiocz.cn/757973.htm
rqb.hjiocz.cn/933133.htm
rqb.hjiocz.cn/355713.htm
rqb.hjiocz.cn/620063.htm
rqb.hjiocz.cn/199153.htm
rqv.hjiocz.cn/804663.htm
rqv.hjiocz.cn/620443.htm
rqv.hjiocz.cn/197773.htm
rqv.hjiocz.cn/880243.htm
rqv.hjiocz.cn/155153.htm
rqv.hjiocz.cn/646063.htm
rqv.hjiocz.cn/400843.htm
rqv.hjiocz.cn/024423.htm
rqv.hjiocz.cn/711773.htm
rqv.hjiocz.cn/202063.htm
rqc.hjiocz.cn/513733.htm
rqc.hjiocz.cn/113733.htm
rqc.hjiocz.cn/646483.htm
rqc.hjiocz.cn/624603.htm
rqc.hjiocz.cn/791913.htm
rqc.hjiocz.cn/008663.htm
rqc.hjiocz.cn/004263.htm
rqc.hjiocz.cn/793773.htm
rqc.hjiocz.cn/026263.htm
rqc.hjiocz.cn/000603.htm
rqx.hjiocz.cn/208403.htm
rqx.hjiocz.cn/428843.htm
rqx.hjiocz.cn/933573.htm
rqx.hjiocz.cn/240883.htm
rqx.hjiocz.cn/979953.htm
rqx.hjiocz.cn/262083.htm
rqx.hjiocz.cn/244083.htm
rqx.hjiocz.cn/044403.htm
rqx.hjiocz.cn/535913.htm
rqx.hjiocz.cn/620683.htm
rqz.hjiocz.cn/020403.htm
rqz.hjiocz.cn/595353.htm
rqz.hjiocz.cn/537393.htm
rqz.hjiocz.cn/286463.htm
rqz.hjiocz.cn/713973.htm
rqz.hjiocz.cn/624063.htm
rqz.hjiocz.cn/204243.htm
rqz.hjiocz.cn/599753.htm
rqz.hjiocz.cn/866863.htm
rqz.hjiocz.cn/042243.htm
rql.hjiocz.cn/399333.htm
rql.hjiocz.cn/646023.htm
rql.hjiocz.cn/917333.htm
rql.hjiocz.cn/228063.htm
rql.hjiocz.cn/313973.htm
rql.hjiocz.cn/197553.htm
rql.hjiocz.cn/604863.htm
rql.hjiocz.cn/000283.htm
rql.hjiocz.cn/820003.htm
rql.hjiocz.cn/793153.htm
rqk.hjiocz.cn/802263.htm
rqk.hjiocz.cn/159393.htm
rqk.hjiocz.cn/448483.htm
rqk.hjiocz.cn/339953.htm
rqk.hjiocz.cn/086263.htm
rqk.hjiocz.cn/040083.htm
rqk.hjiocz.cn/337793.htm
rqk.hjiocz.cn/642283.htm
rqk.hjiocz.cn/917353.htm
rqk.hjiocz.cn/426663.htm
rqj.hjiocz.cn/824463.htm
rqj.hjiocz.cn/226463.htm
rqj.hjiocz.cn/486603.htm
rqj.hjiocz.cn/640803.htm
rqj.hjiocz.cn/240663.htm
rqj.hjiocz.cn/842283.htm
rqj.hjiocz.cn/991113.htm
rqj.hjiocz.cn/266223.htm
rqj.hjiocz.cn/468483.htm
rqj.hjiocz.cn/751373.htm
rqh.hjiocz.cn/913973.htm
rqh.hjiocz.cn/402643.htm
rqh.hjiocz.cn/000223.htm
rqh.hjiocz.cn/082883.htm
rqh.hjiocz.cn/066443.htm
rqh.hjiocz.cn/262823.htm
rqh.hjiocz.cn/008083.htm
rqh.hjiocz.cn/191773.htm
rqh.hjiocz.cn/008403.htm
rqh.hjiocz.cn/846603.htm
rqg.hjiocz.cn/862023.htm
rqg.hjiocz.cn/002803.htm
rqg.hjiocz.cn/086883.htm
rqg.hjiocz.cn/197993.htm
rqg.hjiocz.cn/779733.htm
rqg.hjiocz.cn/642883.htm
rqg.hjiocz.cn/862823.htm
rqg.hjiocz.cn/684223.htm
rqg.hjiocz.cn/484883.htm
rqg.hjiocz.cn/537393.htm
rqf.hjiocz.cn/280663.htm
rqf.hjiocz.cn/848683.htm
rqf.hjiocz.cn/268263.htm
rqf.hjiocz.cn/080003.htm
rqf.hjiocz.cn/446423.htm
rqf.hjiocz.cn/886843.htm
rqf.hjiocz.cn/020483.htm
rqf.hjiocz.cn/264483.htm
rqf.hjiocz.cn/868803.htm
rqf.hjiocz.cn/882463.htm
rqd.hjiocz.cn/864223.htm
rqd.hjiocz.cn/195513.htm
rqd.hjiocz.cn/139193.htm
rqd.hjiocz.cn/684203.htm
rqd.hjiocz.cn/993173.htm
rqd.hjiocz.cn/646443.htm
rqd.hjiocz.cn/026823.htm
rqd.hjiocz.cn/828043.htm
rqd.hjiocz.cn/044823.htm
rqd.hjiocz.cn/020863.htm
rqs.hjiocz.cn/028283.htm
rqs.hjiocz.cn/400203.htm
rqs.hjiocz.cn/020823.htm
rqs.hjiocz.cn/040003.htm
rqs.hjiocz.cn/531373.htm
rqs.hjiocz.cn/286603.htm
rqs.hjiocz.cn/600823.htm
rqs.hjiocz.cn/422483.htm
rqs.hjiocz.cn/866063.htm
rqs.hjiocz.cn/682443.htm
rqa.hjiocz.cn/864683.htm
rqa.hjiocz.cn/579733.htm
rqa.hjiocz.cn/715953.htm
rqa.hjiocz.cn/864443.htm
rqa.hjiocz.cn/624083.htm
rqa.hjiocz.cn/868483.htm
rqa.hjiocz.cn/228203.htm
rqa.hjiocz.cn/995593.htm
rqa.hjiocz.cn/517393.htm
rqa.hjiocz.cn/157553.htm
rqp.hjiocz.cn/486623.htm
rqp.hjiocz.cn/442803.htm
rqp.hjiocz.cn/824263.htm
rqp.hjiocz.cn/795533.htm
rqp.hjiocz.cn/406223.htm
rqp.hjiocz.cn/046683.htm
rqp.hjiocz.cn/131913.htm
rqp.hjiocz.cn/886863.htm
rqp.hjiocz.cn/062203.htm
rqp.hjiocz.cn/591573.htm
rqo.hjiocz.cn/822603.htm
rqo.hjiocz.cn/448403.htm
rqo.hjiocz.cn/175913.htm
rqo.hjiocz.cn/519793.htm
rqo.hjiocz.cn/426683.htm
rqo.hjiocz.cn/460823.htm
rqo.hjiocz.cn/197333.htm
rqo.hjiocz.cn/686263.htm
rqo.hjiocz.cn/028823.htm
rqo.hjiocz.cn/799313.htm
rqi.hjiocz.cn/371753.htm
rqi.hjiocz.cn/026263.htm
rqi.hjiocz.cn/133113.htm
rqi.hjiocz.cn/062623.htm
rqi.hjiocz.cn/991133.htm
rqi.hjiocz.cn/486223.htm
rqi.hjiocz.cn/937933.htm
rqi.hjiocz.cn/735973.htm
rqi.hjiocz.cn/060663.htm
rqi.hjiocz.cn/068463.htm
rqu.hjiocz.cn/511753.htm
rqu.hjiocz.cn/999313.htm
rqu.hjiocz.cn/882483.htm
rqu.hjiocz.cn/086063.htm
rqu.hjiocz.cn/620023.htm
rqu.hjiocz.cn/286003.htm
rqu.hjiocz.cn/511793.htm
rqu.hjiocz.cn/777753.htm
rqu.hjiocz.cn/808483.htm
rqu.hjiocz.cn/133153.htm
rqy.hjiocz.cn/200423.htm
rqy.hjiocz.cn/359753.htm
rqy.hjiocz.cn/024203.htm
rqy.hjiocz.cn/282463.htm
rqy.hjiocz.cn/773753.htm
rqy.hjiocz.cn/084063.htm
rqy.hjiocz.cn/179993.htm
rqy.hjiocz.cn/844643.htm
rqy.hjiocz.cn/040263.htm
rqy.hjiocz.cn/820403.htm
rqt.hjiocz.cn/937333.htm
rqt.hjiocz.cn/688843.htm
rqt.hjiocz.cn/624843.htm
rqt.hjiocz.cn/480443.htm
rqt.hjiocz.cn/240463.htm
rqt.hjiocz.cn/626663.htm
rqt.hjiocz.cn/062683.htm
rqt.hjiocz.cn/359193.htm
rqt.hjiocz.cn/357373.htm
rqt.hjiocz.cn/753193.htm
rqr.hjiocz.cn/488083.htm
rqr.hjiocz.cn/842043.htm
rqr.hjiocz.cn/224243.htm
rqr.hjiocz.cn/840803.htm
rqr.hjiocz.cn/608443.htm
rqr.hjiocz.cn/626643.htm
rqr.hjiocz.cn/800403.htm
rqr.hjiocz.cn/593573.htm
rqr.hjiocz.cn/151113.htm
rqr.hjiocz.cn/826023.htm
rqe.hjiocz.cn/464603.htm
rqe.hjiocz.cn/775373.htm
rqe.hjiocz.cn/666043.htm
rqe.hjiocz.cn/046843.htm
rqe.hjiocz.cn/008243.htm
rqe.hjiocz.cn/779573.htm
rqe.hjiocz.cn/284483.htm
rqe.hjiocz.cn/537313.htm
rqe.hjiocz.cn/404043.htm
rqe.hjiocz.cn/006883.htm
rqw.hjiocz.cn/828403.htm
rqw.hjiocz.cn/533913.htm
rqw.hjiocz.cn/684843.htm
rqw.hjiocz.cn/537333.htm
rqw.hjiocz.cn/264823.htm
rqw.hjiocz.cn/995173.htm
rqw.hjiocz.cn/975553.htm
rqw.hjiocz.cn/311533.htm
rqw.hjiocz.cn/551393.htm
rqw.hjiocz.cn/604203.htm
rqq.hjiocz.cn/313573.htm
rqq.hjiocz.cn/424823.htm
rqq.hjiocz.cn/137933.htm
rqq.hjiocz.cn/262003.htm
rqq.hjiocz.cn/488043.htm
rqq.hjiocz.cn/688443.htm
rqq.hjiocz.cn/826863.htm
rqq.hjiocz.cn/264603.htm
rqq.hjiocz.cn/157333.htm
rqq.hjiocz.cn/200483.htm
emm.hjiocz.cn/311373.htm
emm.hjiocz.cn/626683.htm
emm.hjiocz.cn/260243.htm
emm.hjiocz.cn/359333.htm
emm.hjiocz.cn/844463.htm
emm.hjiocz.cn/260843.htm
emm.hjiocz.cn/620003.htm
emm.hjiocz.cn/888063.htm
emm.hjiocz.cn/353593.htm
emm.hjiocz.cn/204063.htm
emn.mmmxz.cn/735553.htm
emn.mmmxz.cn/246023.htm
emn.mmmxz.cn/800243.htm
emn.mmmxz.cn/393773.htm
emn.mmmxz.cn/206623.htm
emn.mmmxz.cn/591193.htm
emn.mmmxz.cn/608223.htm
emn.mmmxz.cn/002003.htm
emn.mmmxz.cn/804243.htm
emn.mmmxz.cn/062603.htm
emb.mmmxz.cn/084043.htm
emb.mmmxz.cn/404843.htm
emb.mmmxz.cn/288843.htm
emb.mmmxz.cn/022463.htm
emb.mmmxz.cn/640863.htm
emb.mmmxz.cn/351533.htm
emb.mmmxz.cn/517193.htm
emb.mmmxz.cn/484443.htm
emb.mmmxz.cn/884243.htm
emb.mmmxz.cn/884283.htm
emv.mmmxz.cn/317993.htm
emv.mmmxz.cn/288643.htm
emv.mmmxz.cn/224803.htm
emv.mmmxz.cn/975553.htm
emv.mmmxz.cn/624823.htm
emv.mmmxz.cn/913513.htm
emv.mmmxz.cn/957533.htm
emv.mmmxz.cn/800803.htm
emv.mmmxz.cn/440823.htm
emv.mmmxz.cn/200463.htm
emc.mmmxz.cn/466443.htm
emc.mmmxz.cn/626263.htm
emc.mmmxz.cn/462403.htm
emc.mmmxz.cn/735153.htm
emc.mmmxz.cn/086603.htm
emc.mmmxz.cn/424083.htm
emc.mmmxz.cn/062403.htm
emc.mmmxz.cn/197393.htm
emc.mmmxz.cn/222243.htm
emc.mmmxz.cn/424083.htm
emx.mmmxz.cn/062843.htm
emx.mmmxz.cn/080823.htm
emx.mmmxz.cn/391373.htm
emx.mmmxz.cn/046603.htm
emx.mmmxz.cn/931133.htm
emx.mmmxz.cn/268063.htm
emx.mmmxz.cn/426663.htm
emx.mmmxz.cn/488463.htm
emx.mmmxz.cn/711733.htm
emx.mmmxz.cn/517553.htm
