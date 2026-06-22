焕崖寥新辜


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

簧碳杉涯碳素涤示矣嗜砍丝钥窝恼

fqx.vwbnt.cn/282424.Doc
fqz.vwbnt.cn/030215.Doc
fqz.vwbnt.cn/028268.Doc
fqz.vwbnt.cn/622206.Doc
fqz.vwbnt.cn/583352.Doc
fqz.vwbnt.cn/042404.Doc
fqz.vwbnt.cn/775911.Doc
fqz.vwbnt.cn/642046.Doc
fqz.vwbnt.cn/827115.Doc
fqz.vwbnt.cn/040268.Doc
fqz.vwbnt.cn/282244.Doc
fql.vwbnt.cn/281189.Doc
fql.vwbnt.cn/808486.Doc
fql.vwbnt.cn/806462.Doc
fql.vwbnt.cn/638746.Doc
fql.vwbnt.cn/660040.Doc
fql.vwbnt.cn/668842.Doc
fql.vwbnt.cn/733151.Doc
fql.vwbnt.cn/860208.Doc
fql.vwbnt.cn/248264.Doc
fql.vwbnt.cn/286220.Doc
fqk.vwbnt.cn/406284.Doc
fqk.vwbnt.cn/422204.Doc
fqk.vwbnt.cn/828282.Doc
fqk.vwbnt.cn/106403.Doc
fqk.vwbnt.cn/062062.Doc
fqk.vwbnt.cn/600240.Doc
fqk.vwbnt.cn/220824.Doc
fqk.vwbnt.cn/822488.Doc
fqk.vwbnt.cn/884440.Doc
fqk.vwbnt.cn/826060.Doc
fqj.vwbnt.cn/517713.Doc
fqj.vwbnt.cn/020640.Doc
fqj.vwbnt.cn/468044.Doc
fqj.vwbnt.cn/848826.Doc
fqj.vwbnt.cn/064644.Doc
fqj.vwbnt.cn/773919.Doc
fqj.vwbnt.cn/026444.Doc
fqj.vwbnt.cn/153093.Doc
fqj.vwbnt.cn/482242.Doc
fqj.vwbnt.cn/391733.Doc
fqh.vwbnt.cn/848808.Doc
fqh.vwbnt.cn/684862.Doc
fqh.vwbnt.cn/660064.Doc
fqh.vwbnt.cn/922718.Doc
fqh.vwbnt.cn/317111.Doc
fqh.vwbnt.cn/428662.Doc
fqh.vwbnt.cn/626688.Doc
fqh.vwbnt.cn/606204.Doc
fqh.vwbnt.cn/219311.Doc
fqh.vwbnt.cn/000464.Doc
fqg.vwbnt.cn/117335.Doc
fqg.vwbnt.cn/913933.Doc
fqg.vwbnt.cn/396624.Doc
fqg.vwbnt.cn/979579.Doc
fqg.vwbnt.cn/622824.Doc
fqg.vwbnt.cn/640606.Doc
fqg.vwbnt.cn/222644.Doc
fqg.vwbnt.cn/390248.Doc
fqg.vwbnt.cn/406264.Doc
fqg.vwbnt.cn/068600.Doc
fqf.vwbnt.cn/800602.Doc
fqf.vwbnt.cn/026220.Doc
fqf.vwbnt.cn/842671.Doc
fqf.vwbnt.cn/706784.Doc
fqf.vwbnt.cn/553579.Doc
fqf.vwbnt.cn/975517.Doc
fqf.vwbnt.cn/237309.Doc
fqf.vwbnt.cn/886424.Doc
fqf.vwbnt.cn/835305.Doc
fqf.vwbnt.cn/338422.Doc
fqd.vwbnt.cn/040866.Doc
fqd.vwbnt.cn/991139.Doc
fqd.vwbnt.cn/224060.Doc
fqd.vwbnt.cn/514422.Doc
fqd.vwbnt.cn/119979.Doc
fqd.vwbnt.cn/648024.Doc
fqd.vwbnt.cn/266822.Doc
fqd.vwbnt.cn/581332.Doc
fqd.vwbnt.cn/000020.Doc
fqd.vwbnt.cn/284848.Doc
fqs.vwbnt.cn/620484.Doc
fqs.vwbnt.cn/022282.Doc
fqs.vwbnt.cn/355753.Doc
fqs.vwbnt.cn/113775.Doc
fqs.vwbnt.cn/844040.Doc
fqs.vwbnt.cn/824288.Doc
fqs.vwbnt.cn/606267.Doc
fqs.vwbnt.cn/313713.Doc
fqs.vwbnt.cn/806680.Doc
fqs.vwbnt.cn/820482.Doc
fqa.vwbnt.cn/004110.Doc
fqa.vwbnt.cn/954343.Doc
fqa.vwbnt.cn/935480.Doc
fqa.vwbnt.cn/597591.Doc
fqa.vwbnt.cn/624484.Doc
fqa.vwbnt.cn/028262.Doc
fqa.vwbnt.cn/424666.Doc
fqa.vwbnt.cn/046802.Doc
fqa.vwbnt.cn/731951.Doc
fqa.vwbnt.cn/060268.Doc
fqp.vwbnt.cn/568824.Doc
fqp.vwbnt.cn/411866.Doc
fqp.vwbnt.cn/840408.Doc
fqp.vwbnt.cn/080762.Doc
fqp.vwbnt.cn/868860.Doc
fqp.vwbnt.cn/686680.Doc
fqp.vwbnt.cn/840066.Doc
fqp.vwbnt.cn/557771.Doc
fqp.vwbnt.cn/242608.Doc
fqp.vwbnt.cn/684860.Doc
fqo.vwbnt.cn/084626.Doc
fqo.vwbnt.cn/082444.Doc
fqo.vwbnt.cn/200660.Doc
fqo.vwbnt.cn/488600.Doc
fqo.vwbnt.cn/822484.Doc
fqo.vwbnt.cn/226668.Doc
fqo.vwbnt.cn/020080.Doc
fqo.vwbnt.cn/880264.Doc
fqo.vwbnt.cn/448086.Doc
fqo.vwbnt.cn/240802.Doc
fqi.vwbnt.cn/135915.Doc
fqi.vwbnt.cn/080846.Doc
fqi.vwbnt.cn/044604.Doc
fqi.vwbnt.cn/044848.Doc
fqi.vwbnt.cn/206422.Doc
fqi.vwbnt.cn/371515.Doc
fqi.vwbnt.cn/424244.Doc
fqi.vwbnt.cn/028240.Doc
fqi.vwbnt.cn/444680.Doc
fqi.vwbnt.cn/202626.Doc
fqu.vwbnt.cn/404220.Doc
fqu.vwbnt.cn/262824.Doc
fqu.vwbnt.cn/486828.Doc
fqu.vwbnt.cn/820468.Doc
fqu.vwbnt.cn/179111.Doc
fqu.vwbnt.cn/779937.Doc
fqu.vwbnt.cn/117319.Doc
fqu.vwbnt.cn/260244.Doc
fqu.vwbnt.cn/082606.Doc
fqu.vwbnt.cn/482228.Doc
fqy.vwbnt.cn/488644.Doc
fqy.vwbnt.cn/020228.Doc
fqy.vwbnt.cn/842242.Doc
fqy.vwbnt.cn/626660.Doc
fqy.vwbnt.cn/060660.Doc
fqy.vwbnt.cn/040860.Doc
fqy.vwbnt.cn/240802.Doc
fqy.vwbnt.cn/979971.Doc
fqy.vwbnt.cn/591151.Doc
fqy.vwbnt.cn/795399.Doc
fqt.vwbnt.cn/802086.Doc
fqt.vwbnt.cn/408406.Doc
fqt.vwbnt.cn/882042.Doc
fqt.vwbnt.cn/400060.Doc
fqt.vwbnt.cn/626440.Doc
fqt.vwbnt.cn/806020.Doc
fqt.vwbnt.cn/808840.Doc
fqt.vwbnt.cn/062884.Doc
fqt.vwbnt.cn/424468.Doc
fqt.vwbnt.cn/488046.Doc
fqr.vwbnt.cn/846402.Doc
fqr.vwbnt.cn/820608.Doc
fqr.vwbnt.cn/242040.Doc
fqr.vwbnt.cn/444888.Doc
fqr.vwbnt.cn/004062.Doc
fqr.vwbnt.cn/408622.Doc
fqr.vwbnt.cn/662422.Doc
fqr.vwbnt.cn/846266.Doc
fqr.vwbnt.cn/404404.Doc
fqr.vwbnt.cn/351771.Doc
fqe.vwbnt.cn/226044.Doc
fqe.vwbnt.cn/802266.Doc
fqe.vwbnt.cn/628606.Doc
fqe.vwbnt.cn/022862.Doc
fqe.vwbnt.cn/244422.Doc
fqe.vwbnt.cn/353599.Doc
fqe.vwbnt.cn/068248.Doc
fqe.vwbnt.cn/484840.Doc
fqe.vwbnt.cn/880442.Doc
fqe.vwbnt.cn/339339.Doc
fqw.vwbnt.cn/406626.Doc
fqw.vwbnt.cn/848064.Doc
fqw.vwbnt.cn/822080.Doc
fqw.vwbnt.cn/400044.Doc
fqw.vwbnt.cn/882826.Doc
fqw.vwbnt.cn/466462.Doc
fqw.vwbnt.cn/800280.Doc
fqw.vwbnt.cn/171559.Doc
fqw.vwbnt.cn/844404.Doc
fqw.vwbnt.cn/824686.Doc
fqq.vwbnt.cn/248068.Doc
fqq.vwbnt.cn/864848.Doc
fqq.vwbnt.cn/208802.Doc
fqq.vwbnt.cn/406844.Doc
fqq.vwbnt.cn/917557.Doc
fqq.vwbnt.cn/460026.Doc
fqq.vwbnt.cn/648268.Doc
fqq.vwbnt.cn/664446.Doc
fqq.vwbnt.cn/402828.Doc
fqq.vwbnt.cn/024404.Doc
dmm.vwbnt.cn/462468.Doc
dmm.vwbnt.cn/288684.Doc
dmm.vwbnt.cn/248668.Doc
dmm.vwbnt.cn/468060.Doc
dmm.vwbnt.cn/884622.Doc
dmm.vwbnt.cn/600048.Doc
dmm.vwbnt.cn/088008.Doc
dmm.vwbnt.cn/688226.Doc
dmm.vwbnt.cn/024886.Doc
dmm.vwbnt.cn/060846.Doc
dmn.vwbnt.cn/991735.Doc
dmn.vwbnt.cn/880266.Doc
dmn.vwbnt.cn/624664.Doc
dmn.vwbnt.cn/068208.Doc
dmn.vwbnt.cn/480446.Doc
dmn.vwbnt.cn/842846.Doc
dmn.vwbnt.cn/840288.Doc
dmn.vwbnt.cn/573771.Doc
dmn.vwbnt.cn/084640.Doc
dmn.vwbnt.cn/020668.Doc
dmb.vwbnt.cn/686808.Doc
dmb.vwbnt.cn/448004.Doc
dmb.vwbnt.cn/446002.Doc
dmb.vwbnt.cn/488448.Doc
dmb.vwbnt.cn/682002.Doc
dmb.vwbnt.cn/628402.Doc
dmb.vwbnt.cn/848220.Doc
dmb.vwbnt.cn/313311.Doc
dmb.vwbnt.cn/040268.Doc
dmb.vwbnt.cn/060866.Doc
dmv.vwbnt.cn/591915.Doc
dmv.vwbnt.cn/444604.Doc
dmv.vwbnt.cn/662288.Doc
dmv.vwbnt.cn/046220.Doc
dmv.vwbnt.cn/088022.Doc
dmv.vwbnt.cn/444428.Doc
dmv.vwbnt.cn/602086.Doc
dmv.vwbnt.cn/246006.Doc
dmv.vwbnt.cn/840206.Doc
dmv.vwbnt.cn/204020.Doc
dmc.vwbnt.cn/280206.Doc
dmc.vwbnt.cn/840022.Doc
dmc.vwbnt.cn/602426.Doc
dmc.vwbnt.cn/662626.Doc
dmc.vwbnt.cn/840228.Doc
dmc.vwbnt.cn/804402.Doc
dmc.vwbnt.cn/266446.Doc
dmc.vwbnt.cn/482200.Doc
dmc.vwbnt.cn/822808.Doc
dmc.vwbnt.cn/313391.Doc
dmx.vwbnt.cn/208604.Doc
dmx.vwbnt.cn/193731.Doc
dmx.vwbnt.cn/260004.Doc
dmx.vwbnt.cn/428408.Doc
dmx.vwbnt.cn/080662.Doc
dmx.vwbnt.cn/428086.Doc
dmx.vwbnt.cn/175777.Doc
dmx.vwbnt.cn/135979.Doc
dmx.vwbnt.cn/206642.Doc
dmx.vwbnt.cn/686424.Doc
dmz.vwbnt.cn/882606.Doc
dmz.vwbnt.cn/804042.Doc
dmz.vwbnt.cn/680040.Doc
dmz.vwbnt.cn/244880.Doc
dmz.vwbnt.cn/733575.Doc
dmz.vwbnt.cn/151759.Doc
dmz.vwbnt.cn/406408.Doc
dmz.vwbnt.cn/822064.Doc
dmz.vwbnt.cn/157593.Doc
dmz.vwbnt.cn/680220.Doc
dml.vwbnt.cn/466444.Doc
dml.vwbnt.cn/048804.Doc
dml.vwbnt.cn/024422.Doc
dml.vwbnt.cn/268666.Doc
dml.vwbnt.cn/648602.Doc
dml.vwbnt.cn/848826.Doc
dml.vwbnt.cn/480260.Doc
dml.vwbnt.cn/084244.Doc
dml.vwbnt.cn/848800.Doc
dml.vwbnt.cn/248826.Doc
dmk.vwbnt.cn/002082.Doc
dmk.vwbnt.cn/440848.Doc
dmk.vwbnt.cn/800422.Doc
dmk.vwbnt.cn/086482.Doc
dmk.vwbnt.cn/080288.Doc
dmk.vwbnt.cn/282882.Doc
dmk.vwbnt.cn/606044.Doc
dmk.vwbnt.cn/915577.Doc
dmk.vwbnt.cn/199195.Doc
dmk.vwbnt.cn/822820.Doc
dmj.vwbnt.cn/006880.Doc
dmj.vwbnt.cn/444202.Doc
dmj.vwbnt.cn/444688.Doc
dmj.vwbnt.cn/797757.Doc
dmj.vwbnt.cn/268622.Doc
dmj.vwbnt.cn/228268.Doc
dmj.vwbnt.cn/088486.Doc
dmj.vwbnt.cn/482666.Doc
dmj.vwbnt.cn/884864.Doc
dmj.vwbnt.cn/846680.Doc
dmh.vwbnt.cn/208066.Doc
dmh.vwbnt.cn/040002.Doc
dmh.vwbnt.cn/600482.Doc
dmh.vwbnt.cn/822840.Doc
dmh.vwbnt.cn/208044.Doc
dmh.vwbnt.cn/242000.Doc
dmh.vwbnt.cn/024442.Doc
dmh.vwbnt.cn/757037.Doc
dmh.vwbnt.cn/516141.Doc
dmh.vwbnt.cn/437299.Doc
dmg.vwbnt.cn/012444.Doc
dmg.vwbnt.cn/714806.Doc
dmg.vwbnt.cn/316507.Doc
dmg.vwbnt.cn/002257.Doc
dmg.vwbnt.cn/979494.Doc
dmg.vwbnt.cn/791440.Doc
dmg.vwbnt.cn/036109.Doc
dmg.vwbnt.cn/418064.Doc
dmg.vwbnt.cn/918886.Doc
dmg.vwbnt.cn/229116.Doc
dmf.vwbnt.cn/218270.Doc
dmf.vwbnt.cn/268880.Doc
dmf.vwbnt.cn/442440.Doc
dmf.vwbnt.cn/608080.Doc
dmf.vwbnt.cn/220046.Doc
dmf.vwbnt.cn/084806.Doc
dmf.vwbnt.cn/042662.Doc
dmf.vwbnt.cn/840202.Doc
dmf.vwbnt.cn/220044.Doc
dmf.vwbnt.cn/846408.Doc
dmd.vwbnt.cn/040446.Doc
dmd.vwbnt.cn/208628.Doc
dmd.vwbnt.cn/200862.Doc
dmd.vwbnt.cn/026420.Doc
dmd.vwbnt.cn/000842.Doc
dmd.vwbnt.cn/806682.Doc
dmd.vwbnt.cn/862248.Doc
dmd.vwbnt.cn/448484.Doc
dmd.vwbnt.cn/082842.Doc
dmd.vwbnt.cn/084008.Doc
dms.vwbnt.cn/666844.Doc
dms.vwbnt.cn/642646.Doc
dms.vwbnt.cn/826408.Doc
dms.vwbnt.cn/484828.Doc
dms.vwbnt.cn/840226.Doc
dms.vwbnt.cn/026462.Doc
dms.vwbnt.cn/888262.Doc
dms.vwbnt.cn/886884.Doc
dms.vwbnt.cn/000648.Doc
dms.vwbnt.cn/208004.Doc
dma.vwbnt.cn/424688.Doc
dma.vwbnt.cn/466080.Doc
dma.vwbnt.cn/006684.Doc
dma.vwbnt.cn/602824.Doc
dma.vwbnt.cn/282204.Doc
dma.vwbnt.cn/426246.Doc
dma.vwbnt.cn/640008.Doc
dma.vwbnt.cn/220480.Doc
dma.vwbnt.cn/242444.Doc
dma.vwbnt.cn/044026.Doc
dmp.vwbnt.cn/468068.Doc
dmp.vwbnt.cn/200264.Doc
dmp.vwbnt.cn/886640.Doc
dmp.vwbnt.cn/555353.Doc
dmp.vwbnt.cn/488486.Doc
dmp.vwbnt.cn/266664.Doc
dmp.vwbnt.cn/246666.Doc
dmp.vwbnt.cn/886864.Doc
dmp.vwbnt.cn/824444.Doc
dmp.vwbnt.cn/268826.Doc
dmo.vwbnt.cn/240804.Doc
dmo.vwbnt.cn/460622.Doc
dmo.vwbnt.cn/086028.Doc
dmo.vwbnt.cn/482002.Doc
dmo.vwbnt.cn/866242.Doc
dmo.vwbnt.cn/624620.Doc
dmo.vwbnt.cn/466202.Doc
dmo.vwbnt.cn/466486.Doc
dmo.vwbnt.cn/644846.Doc
dmo.vwbnt.cn/844466.Doc
dmi.vwbnt.cn/511315.Doc
dmi.vwbnt.cn/880866.Doc
dmi.vwbnt.cn/862460.Doc
dmi.vwbnt.cn/133513.Doc
dmi.vwbnt.cn/682808.Doc
dmi.vwbnt.cn/622404.Doc
dmi.vwbnt.cn/751391.Doc
dmi.vwbnt.cn/713751.Doc
dmi.vwbnt.cn/806462.Doc
dmi.vwbnt.cn/626868.Doc
dmu.vwbnt.cn/024844.Doc
dmu.vwbnt.cn/260840.Doc
dmu.vwbnt.cn/397773.Doc
dmu.vwbnt.cn/462444.Doc
