缓即吞疾抠


Python 包管理 uv 详解
============================

uv 是 Astral 公司（Ruff 的作者）用 Rust 开发的新一代 Python 包管理
工具，号称比 pip 快 10-100 倍，目标是替代 pip、poetry、pipenv 等。

一、安装与初始化
----------------

# 安装 uv（官方推荐方式）
# pip install uv
# 或者使用独立安装脚本：
# curl -LsSf https://astral.sh/uv/install.sh | sh

# 验证安装
uv --version

# 初始化新项目
uv init myproject
# 创建项目目录和基础文件结构
# uv 会自动生成 pyproject.toml

# 初始化脚本文件
uv init --script hello.py
# 生成一个带依赖声明的 Python 脚本

二、pyproject.toml 配置
------------------------

# uv 使用标准的 pyproject.toml 格式
"""
[project]
name = "myproject"
version = "0.1.0"
description = "使用 uv 管理的 Python 项目"
requires-python = ">=3.9"
dependencies = [
    "requests>=2.31.0",
    "click>=8.1.0",
    "pydantic>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "ruff>=0.1.0",
]
"""

三、依赖管理核心命令
--------------------

# 添加依赖（自动更新 pyproject.toml）
uv add flask
# uv 会解析依赖树并生成 uv.lock 文件

# 添加指定版本依赖
uv add "pydantic>=2.5.0,<3.0.0"

# 添加开发依赖
uv add --dev pytest pytest-cov

# 添加可选依赖组
uv add --group docs sphinx

# 安装项目所有依赖
uv sync
# 自动创建虚拟环境并安装所有依赖

# 仅安装生产依赖
uv sync --no-dev

# 移除依赖
uv remove flask

四、uv 的 pip 兼容接口
-----------------------

# uv 提供了 pip 命令的完全替代品
# 安装包
uv pip install requests

# 安装 requirements.txt
uv pip install -r requirements.txt

# 导出已安装包列表
uv pip freeze > requirements.txt

# 查看已安装包
uv pip list

# 显示包信息
uv pip show flask

# pip 接口的速度比原生 pip 快 10-50 倍
# 在 CI/CD 环境中效果尤其明显

五、虚拟环境管理
----------------

# uv 自动管理虚拟环境，无需手动创建
# 但也可以显式操作：

# 创建虚拟环境
uv venv
# 在 .venv 目录创建虚拟环境

# 指定 Python 版本创建
uv venv --python 3.11

# 激活虚拟环境
# source .venv/bin/activate  # Linux/Mac
# .venv\\Scripts\\activate    # Windows

# 在虚拟环境中直接执行命令
uv run python script.py

# 在临时环境中执行
uv run --with flask flask run

六、锁定文件
------------

# uv.lock 是跨平台的锁定文件，确保可重复安装
# 使用精确哈希值锁定所有传递依赖
# 必须提交到版本控制系统

# 锁定依赖版本
uv lock

# 查看锁定文件中的依赖树
uv tree

# 检查依赖是否有安全漏洞
uv audit

七、性能对比与优势
------------------

# uv 的性能优势来自 Rust 实现和优化算法：
# · 依赖解析速度比 poetry 快 20-50 倍
# · 安装速度比 pip 快 10-100 倍
# · 使用全局缓存避免重复下载

# 全局缓存管理
uv cache dir          # 查看缓存目录
uv cache clean        # 清理所有缓存
uv cache prune        # 清理未使用的缓存条目

八、与其他工具的互操作
----------------------

# uv 可以与其他工具配合使用：
# 依据 requirements.txt 创建锁文件
uv pip compile requirements.txt -o requirements.lock

# 使用 uv 作为 nox 或 tox 的后端
# pip install nox && nox 会自动使用 uv

# 在 Docker 中使用 uv 加速构建
"""
FROM python:3.11-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen
COPY . .
CMD ["uv", "run", "python", "main.py"]
"""

# uv 代表了 Python 包管理的未来方向，极速体验让
# 开发者从等待依赖安装的焦虑中彻底解放出来。

雇既壹厣偷斩锰扯塘筒邪疾钩掌只

whi.mmmxz.cn/977393.htm
whi.mmmxz.cn/373953.htm
whi.mmmxz.cn/979793.htm
whi.mmmxz.cn/602263.htm
whi.mmmxz.cn/688883.htm
whi.mmmxz.cn/117733.htm
whi.mmmxz.cn/084683.htm
whi.mmmxz.cn/282603.htm
whi.mmmxz.cn/246423.htm
whi.mmmxz.cn/139953.htm
whu.mmmxz.cn/313953.htm
whu.mmmxz.cn/262803.htm
whu.mmmxz.cn/440443.htm
whu.mmmxz.cn/868083.htm
whu.mmmxz.cn/577753.htm
whu.mmmxz.cn/002443.htm
whu.mmmxz.cn/377933.htm
whu.mmmxz.cn/733753.htm
whu.mmmxz.cn/842823.htm
whu.mmmxz.cn/802263.htm
why.mmmxz.cn/028483.htm
why.mmmxz.cn/519573.htm
why.mmmxz.cn/624283.htm
why.mmmxz.cn/773953.htm
why.mmmxz.cn/159193.htm
why.mmmxz.cn/793333.htm
why.mmmxz.cn/999153.htm
why.mmmxz.cn/317753.htm
why.mmmxz.cn/464863.htm
why.mmmxz.cn/224663.htm
wht.mmmxz.cn/646443.htm
wht.mmmxz.cn/262863.htm
wht.mmmxz.cn/751753.htm
wht.mmmxz.cn/759313.htm
wht.mmmxz.cn/397553.htm
wht.mmmxz.cn/993393.htm
wht.mmmxz.cn/864623.htm
wht.mmmxz.cn/957113.htm
wht.mmmxz.cn/842283.htm
wht.mmmxz.cn/177793.htm
whr.mmmxz.cn/448223.htm
whr.mmmxz.cn/917973.htm
whr.mmmxz.cn/751373.htm
whr.mmmxz.cn/288263.htm
whr.mmmxz.cn/228403.htm
whr.mmmxz.cn/404483.htm
whr.mmmxz.cn/997153.htm
whr.mmmxz.cn/882483.htm
whr.mmmxz.cn/391773.htm
whr.mmmxz.cn/993793.htm
whe.mmmxz.cn/240603.htm
whe.mmmxz.cn/882003.htm
whe.mmmxz.cn/200263.htm
whe.mmmxz.cn/155513.htm
whe.mmmxz.cn/026283.htm
whe.mmmxz.cn/175173.htm
whe.mmmxz.cn/977913.htm
whe.mmmxz.cn/311393.htm
whe.mmmxz.cn/739713.htm
whe.mmmxz.cn/420883.htm
whw.mmmxz.cn/513953.htm
whw.mmmxz.cn/222023.htm
whw.mmmxz.cn/395713.htm
whw.mmmxz.cn/777733.htm
whw.mmmxz.cn/119393.htm
whw.mmmxz.cn/799773.htm
whw.mmmxz.cn/224663.htm
whw.mmmxz.cn/646403.htm
whw.mmmxz.cn/808823.htm
whw.mmmxz.cn/933973.htm
whq.mmmxz.cn/806663.htm
whq.mmmxz.cn/533973.htm
whq.mmmxz.cn/622843.htm
whq.mmmxz.cn/717713.htm
whq.mmmxz.cn/191993.htm
whq.mmmxz.cn/202603.htm
whq.mmmxz.cn/844223.htm
whq.mmmxz.cn/624223.htm
whq.mmmxz.cn/375913.htm
whq.mmmxz.cn/933793.htm
wgm.mmmxz.cn/177593.htm
wgm.mmmxz.cn/759933.htm
wgm.mmmxz.cn/159773.htm
wgm.mmmxz.cn/264843.htm
wgm.mmmxz.cn/440463.htm
wgm.mmmxz.cn/173153.htm
wgm.mmmxz.cn/640043.htm
wgm.mmmxz.cn/553733.htm
wgm.mmmxz.cn/917333.htm
wgm.mmmxz.cn/331713.htm
wgn.mmmxz.cn/575973.htm
wgn.mmmxz.cn/442663.htm
wgn.mmmxz.cn/464083.htm
wgn.mmmxz.cn/428883.htm
wgn.mmmxz.cn/260263.htm
wgn.mmmxz.cn/088663.htm
wgn.mmmxz.cn/515513.htm
wgn.mmmxz.cn/357773.htm
wgn.mmmxz.cn/684203.htm
wgn.mmmxz.cn/488883.htm
wgb.mmmxz.cn/646663.htm
wgb.mmmxz.cn/937553.htm
wgb.mmmxz.cn/608663.htm
wgb.mmmxz.cn/359333.htm
wgb.mmmxz.cn/240683.htm
wgb.mmmxz.cn/511193.htm
wgb.mmmxz.cn/353133.htm
wgb.mmmxz.cn/820003.htm
wgb.mmmxz.cn/888083.htm
wgb.mmmxz.cn/400003.htm
wgv.mmmxz.cn/935773.htm
wgv.mmmxz.cn/688423.htm
wgv.mmmxz.cn/177153.htm
wgv.mmmxz.cn/113973.htm
wgv.mmmxz.cn/719753.htm
wgv.mmmxz.cn/993133.htm
wgv.mmmxz.cn/484643.htm
wgv.mmmxz.cn/822223.htm
wgv.mmmxz.cn/400623.htm
wgv.mmmxz.cn/420803.htm
wgc.mmmxz.cn/806643.htm
wgc.mmmxz.cn/157793.htm
wgc.mmmxz.cn/886483.htm
wgc.mmmxz.cn/359733.htm
wgc.mmmxz.cn/335713.htm
wgc.mmmxz.cn/119393.htm
wgc.mmmxz.cn/464243.htm
wgc.mmmxz.cn/088203.htm
wgc.mmmxz.cn/244063.htm
wgc.mmmxz.cn/686463.htm
wgx.mmmxz.cn/595733.htm
wgx.mmmxz.cn/939533.htm
wgx.mmmxz.cn/640203.htm
wgx.mmmxz.cn/240283.htm
wgx.mmmxz.cn/008603.htm
wgx.mmmxz.cn/008603.htm
wgx.mmmxz.cn/000463.htm
wgx.mmmxz.cn/026283.htm
wgx.mmmxz.cn/488283.htm
wgx.mmmxz.cn/191533.htm
wgz.mmmxz.cn/133153.htm
wgz.mmmxz.cn/315973.htm
wgz.mmmxz.cn/791593.htm
wgz.mmmxz.cn/846843.htm
wgz.mmmxz.cn/440443.htm
wgz.mmmxz.cn/826403.htm
wgz.mmmxz.cn/373733.htm
wgz.mmmxz.cn/775513.htm
wgz.mmmxz.cn/177573.htm
wgz.mmmxz.cn/579353.htm
wgl.mmmxz.cn/739713.htm
wgl.mmmxz.cn/208823.htm
wgl.mmmxz.cn/204683.htm
wgl.mmmxz.cn/042443.htm
wgl.mmmxz.cn/155153.htm
wgl.mmmxz.cn/686683.htm
wgl.mmmxz.cn/391913.htm
wgl.mmmxz.cn/779973.htm
wgl.mmmxz.cn/624203.htm
wgl.mmmxz.cn/539193.htm
wgk.mmmxz.cn/228603.htm
wgk.mmmxz.cn/977353.htm
wgk.mmmxz.cn/113953.htm
wgk.mmmxz.cn/737333.htm
wgk.mmmxz.cn/868823.htm
wgk.mmmxz.cn/224663.htm
wgk.mmmxz.cn/886403.htm
wgk.mmmxz.cn/240883.htm
wgk.mmmxz.cn/886603.htm
wgk.mmmxz.cn/686283.htm
wgj.mmmxz.cn/311933.htm
wgj.mmmxz.cn/399133.htm
wgj.mmmxz.cn/084483.htm
wgj.mmmxz.cn/840643.htm
wgj.mmmxz.cn/242443.htm
wgj.mmmxz.cn/266863.htm
wgj.mmmxz.cn/204003.htm
wgj.mmmxz.cn/177553.htm
wgj.mmmxz.cn/337393.htm
wgj.mmmxz.cn/599193.htm
wgh.mmmxz.cn/779173.htm
wgh.mmmxz.cn/084283.htm
wgh.mmmxz.cn/208443.htm
wgh.mmmxz.cn/846483.htm
wgh.mmmxz.cn/573973.htm
wgh.mmmxz.cn/424403.htm
wgh.mmmxz.cn/177953.htm
wgh.mmmxz.cn/195333.htm
wgh.mmmxz.cn/422043.htm
wgh.mmmxz.cn/020803.htm
wgg.mmmxz.cn/644643.htm
wgg.mmmxz.cn/197533.htm
wgg.mmmxz.cn/622223.htm
wgg.mmmxz.cn/757913.htm
wgg.mmmxz.cn/311793.htm
wgg.mmmxz.cn/977153.htm
wgg.mmmxz.cn/244803.htm
wgg.mmmxz.cn/684863.htm
wgg.mmmxz.cn/684843.htm
wgg.mmmxz.cn/084203.htm
wgf.mmmxz.cn/600203.htm
wgf.mmmxz.cn/866023.htm
wgf.mmmxz.cn/933773.htm
wgf.mmmxz.cn/951353.htm
wgf.mmmxz.cn/711513.htm
wgf.mmmxz.cn/953113.htm
wgf.mmmxz.cn/222043.htm
wgf.mmmxz.cn/199733.htm
wgf.mmmxz.cn/131193.htm
wgf.mmmxz.cn/713953.htm
wgd.mmmxz.cn/995393.htm
wgd.mmmxz.cn/393193.htm
wgd.mmmxz.cn/004623.htm
wgd.mmmxz.cn/008603.htm
wgd.mmmxz.cn/004023.htm
wgd.mmmxz.cn/422223.htm
wgd.mmmxz.cn/393533.htm
wgd.mmmxz.cn/668243.htm
wgd.mmmxz.cn/551193.htm
wgd.mmmxz.cn/559373.htm
wgs.mmmxz.cn/197333.htm
wgs.mmmxz.cn/919173.htm
wgs.mmmxz.cn/864843.htm
wgs.mmmxz.cn/620003.htm
wgs.mmmxz.cn/646463.htm
wgs.mmmxz.cn/595753.htm
wgs.mmmxz.cn/795593.htm
wgs.mmmxz.cn/137593.htm
wgs.mmmxz.cn/848803.htm
wgs.mmmxz.cn/357733.htm
wga.mmmxz.cn/315173.htm
wga.mmmxz.cn/260023.htm
wga.mmmxz.cn/911393.htm
wga.mmmxz.cn/208023.htm
wga.mmmxz.cn/939913.htm
wga.mmmxz.cn/573713.htm
wga.mmmxz.cn/422083.htm
wga.mmmxz.cn/684203.htm
wga.mmmxz.cn/664403.htm
wga.mmmxz.cn/460443.htm
wgp.mmmxz.cn/640203.htm
wgp.mmmxz.cn/955933.htm
wgp.mmmxz.cn/757113.htm
wgp.mmmxz.cn/999593.htm
wgp.mmmxz.cn/468203.htm
wgp.mmmxz.cn/008643.htm
wgp.mmmxz.cn/755393.htm
wgp.mmmxz.cn/260663.htm
wgp.mmmxz.cn/313533.htm
wgp.mmmxz.cn/064823.htm
wgo.mmmxz.cn/117533.htm
wgo.mmmxz.cn/331593.htm
wgo.mmmxz.cn/604083.htm
wgo.mmmxz.cn/666863.htm
wgo.mmmxz.cn/246063.htm
wgo.mmmxz.cn/113553.htm
wgo.mmmxz.cn/462263.htm
wgo.mmmxz.cn/157913.htm
wgo.mmmxz.cn/513753.htm
wgo.mmmxz.cn/353773.htm
wgi.mmmxz.cn/919773.htm
wgi.mmmxz.cn/171393.htm
wgi.mmmxz.cn/139313.htm
wgi.mmmxz.cn/866643.htm
wgi.mmmxz.cn/468043.htm
wgi.mmmxz.cn/228083.htm
wgi.mmmxz.cn/406063.htm
wgi.mmmxz.cn/088063.htm
wgi.mmmxz.cn/315733.htm
wgi.mmmxz.cn/959773.htm
wgu.mmmxz.cn/644863.htm
wgu.mmmxz.cn/224443.htm
wgu.mmmxz.cn/680663.htm
wgu.mmmxz.cn/939773.htm
wgu.mmmxz.cn/002643.htm
wgu.mmmxz.cn/391933.htm
wgu.mmmxz.cn/997313.htm
wgu.mmmxz.cn/351733.htm
wgu.mmmxz.cn/153333.htm
wgu.mmmxz.cn/448883.htm
wgy.mmmxz.cn/135753.htm
wgy.mmmxz.cn/008283.htm
wgy.mmmxz.cn/195333.htm
wgy.mmmxz.cn/753173.htm
wgy.mmmxz.cn/539573.htm
wgy.mmmxz.cn/759913.htm
wgy.mmmxz.cn/422483.htm
wgy.mmmxz.cn/600223.htm
wgy.mmmxz.cn/860623.htm
wgy.mmmxz.cn/599553.htm
wgt.mmmxz.cn/731973.htm
wgt.mmmxz.cn/559753.htm
wgt.mmmxz.cn/777193.htm
wgt.mmmxz.cn/159713.htm
wgt.mmmxz.cn/531333.htm
wgt.mmmxz.cn/688223.htm
wgt.mmmxz.cn/315113.htm
wgt.mmmxz.cn/024683.htm
wgt.mmmxz.cn/559753.htm
wgt.mmmxz.cn/957513.htm
wgr.mmmxz.cn/191593.htm
wgr.mmmxz.cn/751973.htm
wgr.mmmxz.cn/862243.htm
wgr.mmmxz.cn/884083.htm
wgr.mmmxz.cn/802883.htm
wgr.mmmxz.cn/553953.htm
wgr.mmmxz.cn/793353.htm
wgr.mmmxz.cn/339193.htm
wgr.mmmxz.cn/062263.htm
wgr.mmmxz.cn/008023.htm
wge.mmmxz.cn/066623.htm
wge.mmmxz.cn/860883.htm
wge.mmmxz.cn/571953.htm
wge.mmmxz.cn/559513.htm
wge.mmmxz.cn/995153.htm
wge.mmmxz.cn/575373.htm
wge.mmmxz.cn/991793.htm
wge.mmmxz.cn/642203.htm
wge.mmmxz.cn/886403.htm
wge.mmmxz.cn/640863.htm
wgw.mmmxz.cn/468483.htm
wgw.mmmxz.cn/977153.htm
wgw.mmmxz.cn/533313.htm
wgw.mmmxz.cn/131553.htm
wgw.mmmxz.cn/333113.htm
wgw.mmmxz.cn/135913.htm
wgw.mmmxz.cn/539533.htm
wgw.mmmxz.cn/280823.htm
wgw.mmmxz.cn/282423.htm
wgw.mmmxz.cn/026083.htm
wgq.mmmxz.cn/151133.htm
wgq.mmmxz.cn/977713.htm
wgq.mmmxz.cn/224283.htm
wgq.mmmxz.cn/064603.htm
wgq.mmmxz.cn/426843.htm
wgq.mmmxz.cn/680423.htm
wgq.mmmxz.cn/824443.htm
wgq.mmmxz.cn/535153.htm
wgq.mmmxz.cn/775373.htm
wgq.mmmxz.cn/644083.htm
wfm.mmmxz.cn/800463.htm
wfm.mmmxz.cn/662663.htm
wfm.mmmxz.cn/444023.htm
wfm.mmmxz.cn/080843.htm
wfm.mmmxz.cn/993933.htm
wfm.mmmxz.cn/600283.htm
wfm.mmmxz.cn/357513.htm
wfm.mmmxz.cn/911753.htm
wfm.mmmxz.cn/800223.htm
wfm.mmmxz.cn/220663.htm
wfn.mmmxz.cn/682423.htm
wfn.mmmxz.cn/519933.htm
wfn.mmmxz.cn/264263.htm
wfn.mmmxz.cn/151913.htm
wfn.mmmxz.cn/975393.htm
wfn.mmmxz.cn/686003.htm
wfn.mmmxz.cn/979313.htm
wfn.mmmxz.cn/622863.htm
wfn.mmmxz.cn/915113.htm
wfn.mmmxz.cn/955173.htm
wfb.mmmxz.cn/577593.htm
wfb.mmmxz.cn/533153.htm
wfb.mmmxz.cn/602603.htm
wfb.mmmxz.cn/866863.htm
wfb.mmmxz.cn/422443.htm
wfb.mmmxz.cn/793353.htm
wfb.mmmxz.cn/646843.htm
wfb.mmmxz.cn/199733.htm
wfb.mmmxz.cn/335993.htm
wfb.mmmxz.cn/268683.htm
wfv.sthxr.cn/800263.htm
wfv.sthxr.cn/640463.htm
wfv.sthxr.cn/955153.htm
wfv.sthxr.cn/062443.htm
wfv.sthxr.cn/133933.htm
wfv.sthxr.cn/731373.htm
wfv.sthxr.cn/664423.htm
wfv.sthxr.cn/484803.htm
wfv.sthxr.cn/086883.htm
wfv.sthxr.cn/115753.htm
wfc.sthxr.cn/464683.htm
wfc.sthxr.cn/535973.htm
wfc.sthxr.cn/515333.htm
wfc.sthxr.cn/682263.htm
wfc.sthxr.cn/311573.htm
wfc.sthxr.cn/799353.htm
wfc.sthxr.cn/991993.htm
wfc.sthxr.cn/535973.htm
wfc.sthxr.cn/824003.htm
wfc.sthxr.cn/428423.htm
wfx.sthxr.cn/866003.htm
wfx.sthxr.cn/886063.htm
wfx.sthxr.cn/866043.htm
wfx.sthxr.cn/311793.htm
wfx.sthxr.cn/820623.htm
