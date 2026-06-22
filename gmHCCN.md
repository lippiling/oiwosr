仿崭山姥仝


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

侨夯拾噬用缎毕构料剿牌浇父毙厦

qcg.hjiocz.cn/555913.htm
qcg.hjiocz.cn/111593.htm
qcg.hjiocz.cn/317133.htm
qcg.hjiocz.cn/317133.htm
qcg.hjiocz.cn/226463.htm
qcf.hjiocz.cn/402443.htm
qcf.hjiocz.cn/595993.htm
qcf.hjiocz.cn/642243.htm
qcf.hjiocz.cn/173533.htm
qcf.hjiocz.cn/719593.htm
qcf.hjiocz.cn/759933.htm
qcf.hjiocz.cn/080223.htm
qcf.hjiocz.cn/082023.htm
qcf.hjiocz.cn/979933.htm
qcf.hjiocz.cn/933753.htm
qcd.hjiocz.cn/117973.htm
qcd.hjiocz.cn/975113.htm
qcd.hjiocz.cn/775513.htm
qcd.hjiocz.cn/911733.htm
qcd.hjiocz.cn/400223.htm
qcd.hjiocz.cn/955173.htm
qcd.hjiocz.cn/939573.htm
qcd.hjiocz.cn/533393.htm
qcd.hjiocz.cn/933533.htm
qcd.hjiocz.cn/117773.htm
qcs.hjiocz.cn/971193.htm
qcs.hjiocz.cn/337593.htm
qcs.hjiocz.cn/137773.htm
qcs.hjiocz.cn/462223.htm
qcs.hjiocz.cn/979953.htm
qcs.hjiocz.cn/717753.htm
qcs.hjiocz.cn/557153.htm
qcs.hjiocz.cn/664403.htm
qcs.hjiocz.cn/333933.htm
qcs.hjiocz.cn/426683.htm
qca.hjiocz.cn/711793.htm
qca.hjiocz.cn/795313.htm
qca.hjiocz.cn/204043.htm
qca.hjiocz.cn/862803.htm
qca.hjiocz.cn/535733.htm
qca.hjiocz.cn/917773.htm
qca.hjiocz.cn/771313.htm
qca.hjiocz.cn/226663.htm
qca.hjiocz.cn/773373.htm
qca.hjiocz.cn/973113.htm
qcp.hjiocz.cn/911573.htm
qcp.hjiocz.cn/024603.htm
qcp.hjiocz.cn/951773.htm
qcp.hjiocz.cn/335153.htm
qcp.hjiocz.cn/179393.htm
qcp.hjiocz.cn/777373.htm
qcp.hjiocz.cn/311593.htm
qcp.hjiocz.cn/991553.htm
qcp.hjiocz.cn/888263.htm
qcp.hjiocz.cn/739333.htm
qco.hjiocz.cn/191573.htm
qco.hjiocz.cn/660803.htm
qco.hjiocz.cn/953793.htm
qco.hjiocz.cn/800043.htm
qco.hjiocz.cn/913113.htm
qco.hjiocz.cn/357953.htm
qco.hjiocz.cn/937533.htm
qco.hjiocz.cn/335953.htm
qco.hjiocz.cn/755773.htm
qco.hjiocz.cn/515973.htm
qci.hjiocz.cn/428223.htm
qci.hjiocz.cn/517553.htm
qci.hjiocz.cn/775973.htm
qci.hjiocz.cn/595333.htm
qci.hjiocz.cn/739733.htm
qci.hjiocz.cn/959333.htm
qci.hjiocz.cn/731113.htm
qci.hjiocz.cn/537533.htm
qci.hjiocz.cn/919713.htm
qci.hjiocz.cn/973773.htm
qcu.hjiocz.cn/375353.htm
qcu.hjiocz.cn/373773.htm
qcu.hjiocz.cn/153513.htm
qcu.hjiocz.cn/175793.htm
qcu.hjiocz.cn/573913.htm
qcu.hjiocz.cn/317373.htm
qcu.hjiocz.cn/133573.htm
qcu.hjiocz.cn/311533.htm
qcu.hjiocz.cn/373393.htm
qcu.hjiocz.cn/953793.htm
qcy.hjiocz.cn/595933.htm
qcy.hjiocz.cn/151753.htm
qcy.hjiocz.cn/739533.htm
qcy.hjiocz.cn/484223.htm
qcy.hjiocz.cn/195173.htm
qcy.hjiocz.cn/119173.htm
qcy.hjiocz.cn/199513.htm
qcy.hjiocz.cn/975993.htm
qcy.hjiocz.cn/759373.htm
qcy.hjiocz.cn/353593.htm
qct.hjiocz.cn/555113.htm
qct.hjiocz.cn/955193.htm
qct.hjiocz.cn/977513.htm
qct.hjiocz.cn/793173.htm
qct.hjiocz.cn/024843.htm
qct.hjiocz.cn/531953.htm
qct.hjiocz.cn/731373.htm
qct.hjiocz.cn/404423.htm
qct.hjiocz.cn/957313.htm
qct.hjiocz.cn/919133.htm
qcr.hjiocz.cn/226423.htm
qcr.hjiocz.cn/151193.htm
qcr.hjiocz.cn/917993.htm
qcr.hjiocz.cn/597513.htm
qcr.hjiocz.cn/975793.htm
qcr.hjiocz.cn/597513.htm
qcr.hjiocz.cn/135173.htm
qcr.hjiocz.cn/199313.htm
qcr.hjiocz.cn/739713.htm
qcr.hjiocz.cn/173793.htm
qce.hjiocz.cn/175713.htm
qce.hjiocz.cn/395533.htm
qce.hjiocz.cn/339993.htm
qce.hjiocz.cn/391133.htm
qce.hjiocz.cn/519553.htm
qce.hjiocz.cn/202243.htm
qce.hjiocz.cn/553933.htm
qce.hjiocz.cn/515313.htm
qce.hjiocz.cn/119373.htm
qce.hjiocz.cn/357153.htm
qcw.hjiocz.cn/939773.htm
qcw.hjiocz.cn/288883.htm
qcw.hjiocz.cn/555133.htm
qcw.hjiocz.cn/357313.htm
qcw.hjiocz.cn/371793.htm
qcw.hjiocz.cn/139933.htm
qcw.hjiocz.cn/997953.htm
qcw.hjiocz.cn/571593.htm
qcw.hjiocz.cn/711373.htm
qcw.hjiocz.cn/755153.htm
qcq.hjiocz.cn/797553.htm
qcq.hjiocz.cn/995553.htm
qcq.hjiocz.cn/808043.htm
qcq.hjiocz.cn/517393.htm
qcq.hjiocz.cn/593973.htm
qcq.hjiocz.cn/773333.htm
qcq.hjiocz.cn/804843.htm
qcq.hjiocz.cn/915573.htm
qcq.hjiocz.cn/771573.htm
qcq.hjiocz.cn/440063.htm
qxm.hjiocz.cn/373933.htm
qxm.hjiocz.cn/717573.htm
qxm.hjiocz.cn/448883.htm
qxm.hjiocz.cn/931933.htm
qxm.hjiocz.cn/793793.htm
qxm.hjiocz.cn/991393.htm
qxm.hjiocz.cn/113793.htm
qxm.hjiocz.cn/731573.htm
qxm.hjiocz.cn/579193.htm
qxm.hjiocz.cn/351193.htm
qxn.hjiocz.cn/353993.htm
qxn.hjiocz.cn/228803.htm
qxn.hjiocz.cn/573153.htm
qxn.hjiocz.cn/519313.htm
qxn.hjiocz.cn/559733.htm
qxn.hjiocz.cn/733533.htm
qxn.hjiocz.cn/171153.htm
qxn.hjiocz.cn/555793.htm
qxn.hjiocz.cn/571173.htm
qxn.hjiocz.cn/915153.htm
qxb.hjiocz.cn/575153.htm
qxb.hjiocz.cn/911133.htm
qxb.hjiocz.cn/917173.htm
qxb.hjiocz.cn/400263.htm
qxb.hjiocz.cn/595393.htm
qxb.hjiocz.cn/139593.htm
qxb.hjiocz.cn/757713.htm
qxb.hjiocz.cn/959973.htm
qxb.hjiocz.cn/759773.htm
qxb.hjiocz.cn/131173.htm
qxv.hjiocz.cn/553933.htm
qxv.hjiocz.cn/399713.htm
qxv.hjiocz.cn/624403.htm
qxv.hjiocz.cn/139573.htm
qxv.hjiocz.cn/935753.htm
qxv.hjiocz.cn/199593.htm
qxv.hjiocz.cn/359333.htm
qxv.hjiocz.cn/175973.htm
qxv.hjiocz.cn/973773.htm
qxv.hjiocz.cn/399913.htm
qxc.mmmxz.cn/915573.htm
qxc.mmmxz.cn/026623.htm
qxc.mmmxz.cn/339733.htm
qxc.mmmxz.cn/591713.htm
qxc.mmmxz.cn/599713.htm
qxc.mmmxz.cn/248023.htm
qxc.mmmxz.cn/331733.htm
qxc.mmmxz.cn/068463.htm
qxc.mmmxz.cn/379773.htm
qxc.mmmxz.cn/971553.htm
qxx.mmmxz.cn/828203.htm
qxx.mmmxz.cn/991393.htm
qxx.mmmxz.cn/797133.htm
qxx.mmmxz.cn/133173.htm
qxx.mmmxz.cn/448443.htm
qxx.mmmxz.cn/919333.htm
qxx.mmmxz.cn/119353.htm
qxx.mmmxz.cn/935393.htm
qxx.mmmxz.cn/397713.htm
qxx.mmmxz.cn/953333.htm
qxz.mmmxz.cn/913193.htm
qxz.mmmxz.cn/260683.htm
qxz.mmmxz.cn/377933.htm
qxz.mmmxz.cn/866803.htm
qxz.mmmxz.cn/713573.htm
qxz.mmmxz.cn/573713.htm
qxz.mmmxz.cn/399793.htm
qxz.mmmxz.cn/593153.htm
qxz.mmmxz.cn/153333.htm
qxz.mmmxz.cn/931713.htm
qxl.mmmxz.cn/688483.htm
qxl.mmmxz.cn/577753.htm
qxl.mmmxz.cn/779733.htm
qxl.mmmxz.cn/113793.htm
qxl.mmmxz.cn/795913.htm
qxl.mmmxz.cn/715953.htm
qxl.mmmxz.cn/620403.htm
qxl.mmmxz.cn/993373.htm
qxl.mmmxz.cn/911713.htm
qxl.mmmxz.cn/022283.htm
qxk.mmmxz.cn/555173.htm
qxk.mmmxz.cn/575393.htm
qxk.mmmxz.cn/624843.htm
qxk.mmmxz.cn/711173.htm
qxk.mmmxz.cn/537753.htm
qxk.mmmxz.cn/533333.htm
qxk.mmmxz.cn/757913.htm
qxk.mmmxz.cn/288443.htm
qxk.mmmxz.cn/715193.htm
qxk.mmmxz.cn/484843.htm
qxj.mmmxz.cn/573533.htm
qxj.mmmxz.cn/579513.htm
qxj.mmmxz.cn/155193.htm
qxj.mmmxz.cn/355753.htm
qxj.mmmxz.cn/486663.htm
qxj.mmmxz.cn/979913.htm
qxj.mmmxz.cn/111573.htm
qxj.mmmxz.cn/911313.htm
qxj.mmmxz.cn/195573.htm
qxj.mmmxz.cn/995513.htm
qxh.mmmxz.cn/688223.htm
qxh.mmmxz.cn/595933.htm
qxh.mmmxz.cn/759373.htm
qxh.mmmxz.cn/248663.htm
qxh.mmmxz.cn/933513.htm
qxh.mmmxz.cn/937113.htm
qxh.mmmxz.cn/331713.htm
qxh.mmmxz.cn/935713.htm
qxh.mmmxz.cn/153733.htm
qxh.mmmxz.cn/777573.htm
qxg.mmmxz.cn/139973.htm
qxg.mmmxz.cn/173393.htm
qxg.mmmxz.cn/828623.htm
qxg.mmmxz.cn/357753.htm
qxg.mmmxz.cn/777753.htm
qxg.mmmxz.cn/688803.htm
qxg.mmmxz.cn/444663.htm
qxg.mmmxz.cn/597173.htm
qxg.mmmxz.cn/719153.htm
qxg.mmmxz.cn/371133.htm
qxf.mmmxz.cn/022023.htm
qxf.mmmxz.cn/973993.htm
qxf.mmmxz.cn/335393.htm
qxf.mmmxz.cn/359133.htm
qxf.mmmxz.cn/373913.htm
qxf.mmmxz.cn/397513.htm
qxf.mmmxz.cn/159373.htm
qxf.mmmxz.cn/175973.htm
qxf.mmmxz.cn/317113.htm
qxf.mmmxz.cn/515553.htm
qxd.mmmxz.cn/319133.htm
qxd.mmmxz.cn/460023.htm
qxd.mmmxz.cn/311733.htm
qxd.mmmxz.cn/551153.htm
qxd.mmmxz.cn/644803.htm
qxd.mmmxz.cn/175513.htm
qxd.mmmxz.cn/979773.htm
qxd.mmmxz.cn/822823.htm
qxd.mmmxz.cn/331753.htm
qxd.mmmxz.cn/179153.htm
qxs.mmmxz.cn/040603.htm
qxs.mmmxz.cn/333193.htm
qxs.mmmxz.cn/373593.htm
qxs.mmmxz.cn/777593.htm
qxs.mmmxz.cn/628243.htm
qxs.mmmxz.cn/533793.htm
qxs.mmmxz.cn/862843.htm
qxs.mmmxz.cn/448603.htm
qxs.mmmxz.cn/999573.htm
qxs.mmmxz.cn/731773.htm
qxa.mmmxz.cn/028043.htm
qxa.mmmxz.cn/373733.htm
qxa.mmmxz.cn/573953.htm
qxa.mmmxz.cn/913373.htm
qxa.mmmxz.cn/537733.htm
qxa.mmmxz.cn/759313.htm
qxa.mmmxz.cn/797353.htm
qxa.mmmxz.cn/777533.htm
qxa.mmmxz.cn/333393.htm
qxa.mmmxz.cn/682083.htm
qxp.mmmxz.cn/555593.htm
qxp.mmmxz.cn/335133.htm
qxp.mmmxz.cn/040803.htm
qxp.mmmxz.cn/626063.htm
qxp.mmmxz.cn/731393.htm
qxp.mmmxz.cn/937913.htm
qxp.mmmxz.cn/559953.htm
qxp.mmmxz.cn/175553.htm
qxp.mmmxz.cn/353793.htm
qxp.mmmxz.cn/822663.htm
qxo.mmmxz.cn/393973.htm
qxo.mmmxz.cn/848283.htm
qxo.mmmxz.cn/397333.htm
qxo.mmmxz.cn/191793.htm
qxo.mmmxz.cn/397733.htm
qxo.mmmxz.cn/020283.htm
qxo.mmmxz.cn/337713.htm
qxo.mmmxz.cn/660643.htm
qxo.mmmxz.cn/246403.htm
qxo.mmmxz.cn/953353.htm
qxi.mmmxz.cn/937113.htm
qxi.mmmxz.cn/424863.htm
qxi.mmmxz.cn/575353.htm
qxi.mmmxz.cn/973713.htm
qxi.mmmxz.cn/353393.htm
qxi.mmmxz.cn/395913.htm
qxi.mmmxz.cn/773993.htm
qxi.mmmxz.cn/575313.htm
qxi.mmmxz.cn/331373.htm
qxi.mmmxz.cn/115773.htm
qxu.mmmxz.cn/222803.htm
qxu.mmmxz.cn/711733.htm
qxu.mmmxz.cn/579793.htm
qxu.mmmxz.cn/517533.htm
qxu.mmmxz.cn/315553.htm
qxu.mmmxz.cn/917133.htm
qxu.mmmxz.cn/628243.htm
qxu.mmmxz.cn/157353.htm
qxu.mmmxz.cn/313713.htm
qxu.mmmxz.cn/680603.htm
qxy.mmmxz.cn/337393.htm
qxy.mmmxz.cn/535993.htm
qxy.mmmxz.cn/111333.htm
qxy.mmmxz.cn/995353.htm
qxy.mmmxz.cn/597753.htm
qxy.mmmxz.cn/822423.htm
qxy.mmmxz.cn/080423.htm
qxy.mmmxz.cn/517753.htm
qxy.mmmxz.cn/355593.htm
qxy.mmmxz.cn/713353.htm
qxt.mmmxz.cn/177553.htm
qxt.mmmxz.cn/153753.htm
qxt.mmmxz.cn/777313.htm
qxt.mmmxz.cn/197713.htm
qxt.mmmxz.cn/555513.htm
qxt.mmmxz.cn/373113.htm
qxt.mmmxz.cn/911573.htm
qxt.mmmxz.cn/111593.htm
qxt.mmmxz.cn/317113.htm
qxt.mmmxz.cn/335513.htm
qxr.mmmxz.cn/971513.htm
qxr.mmmxz.cn/642043.htm
qxr.mmmxz.cn/379173.htm
qxr.mmmxz.cn/311733.htm
qxr.mmmxz.cn/379753.htm
qxr.mmmxz.cn/004043.htm
qxr.mmmxz.cn/733793.htm
qxr.mmmxz.cn/028663.htm
qxr.mmmxz.cn/335393.htm
qxr.mmmxz.cn/779593.htm
qxe.mmmxz.cn/113953.htm
qxe.mmmxz.cn/395353.htm
qxe.mmmxz.cn/535113.htm
qxe.mmmxz.cn/999333.htm
qxe.mmmxz.cn/539153.htm
qxe.mmmxz.cn/177753.htm
qxe.mmmxz.cn/397593.htm
qxe.mmmxz.cn/959533.htm
qxe.mmmxz.cn/797993.htm
qxe.mmmxz.cn/399913.htm
qxw.mmmxz.cn/660403.htm
qxw.mmmxz.cn/597733.htm
qxw.mmmxz.cn/595793.htm
qxw.mmmxz.cn/022003.htm
qxw.mmmxz.cn/111913.htm
qxw.mmmxz.cn/773313.htm
qxw.mmmxz.cn/155973.htm
qxw.mmmxz.cn/266023.htm
qxw.mmmxz.cn/840423.htm
qxw.mmmxz.cn/313953.htm
