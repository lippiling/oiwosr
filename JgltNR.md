侥世加扒园


Python 项目生成器 Copier 详解
=======================================

Copier 是现代化的项目脚手架工具，基于 Jinja2 模板引擎，支持嵌套
模板、问答文件、任务系统等功能，比 Cookiecutter 更灵活强大。

一、安装与基本使用
------------------

# 安装 Copier
# pip install copier

# 验证安装
copier --version

# 从 Git 仓库模板创建项目
copier copy https://github.com/example/python-template.git myproject

# 从本地模板创建项目
copier copy ./my-template ./new-project

# 查看帮助
copier copy --help

# 使用已有答案文件跳过交互
copier copy -f ./my-template ./new-project -d project_name=MyApp

二、模板目录结构
----------------

# Copier 模板的标准目录结构
"""
my-template/
├── copier.yml                     # 模板配置与变量定义
├── {{project_name}}/               # 使用 {{变量名}} 语法
│   ├── src/
│   │   └── main.py.jinja           # .jinja 后缀表示需要渲染
│   ├── tests/
│   │   └── test_app.py.jinja
│   ├── README.md.jinja
│   ├── pyproject.toml.jinja
│   └── .gitignore.jinja
├── tasks.py                       # 生成后任务
└── .copier-answers.yml            # 自动生成的答案文件
"""

# 注意 Copier 使用 {{变量名}} 而不是 {{cookiecutter.变量名}}
# 不需要 cookiecutter. 前缀

三、copier.yml 配置
-------------------

# copier.yml 定义模板变量和配置
"""
# 变量定义和类型
project_name:
    type: str
    help: 项目名称
    default: myproject
    validator: >-
        ^[a-zA-Z][a-zA-Z0-9_-]+$

description:
    type: str
    help: 项目描述
    default: 一个 Python 项目

author_name:
    type: str
    help: 作者姓名
    default: 开发者

python_version:
    type: float
    help: Python 版本
    choices:
        - 3.9
        - 3.10
        - 3.11
    default: 3.11

use_mypy:
    type: bool
    help: 是否使用 Mypy 类型检查？
    default: yes

license:
    type: str
    help: 选择许可证
    choices:
        MIT: MIT 许可证
        APACHE: Apache 2.0
        GPL: GPL v3
    default: MIT

# 排除的文件
_exclude:
    - "*.pyc"
    - "__pycache__"

# 模板后缀
_templates_suffix: ".jinja"
"""

四、Jinja2 模板渲染
-------------------

# 在模板文件中使用 Jinja2 语法

# pyproject.toml.jinja
"""
[project]
name = "{{ project_name }}"
version = "0.1.0"
description = "{{ description }}"
authors = [{name = "{{ author_name }}"}]
requires-python = ">= {{ python_version }}"
dependencies = [
    "click>=8.1.0",
]

{% if use_mypy %}
[tool.mypy]
strict = true
{% endif %}

{% if license == "MIT" %}
[tool.poetry.license]
text = "MIT"
{% endif %}
"""

# src/main.py.jinja
"""
def main() -> None:
    {{ greeting }}
    print("{{ project_name }} 已启动!")

if __name__ == "__main__":
    main()
"""

# README.md.jinja
"""
# {{ project_name }}

{{ description }}

## 安装

```bash
pip install {{ project_name }}
```

## 使用

```python
import {{ project_name }}
```

## 许可证

{{ license }}
"""

五、answers 文件
----------------

# .copier-answers.yml 记录用户的每次选择
"""
# 自动生成的文件，记录模板版本和用户选择
# 用于后续更新模板

_commit: v1.2.0
_src_path: gh:example/python-template
project_name: myproject
description: 一个 Web API 项目
author_name: 开发者
python_version: 3.11
use_mypy: true
license: MIT
"""

# 使用 answers 文件更新已有项目
copier update
# 基于 .copier-answers.yml 中的记录进行差异化更新

# 检查更新但不应用
copier update --pretend

# 强制使用 answers 文件
copier copy -f ./template ./project --trust

六、任务系统（tasks.py）
-----------------------

# tasks.py 定义生成后需要执行的任务
"""
import subprocess
from pathlib import Path

def setup_git(repo_path: Path, project_name: str) -> None:
    # 初始化 Git 仓库
    subprocess.run(["git", "init"], cwd=repo_path)
    subprocess.run(["git", "add", "."], cwd=repo_path)
    subprocess.run(
        ["git", "commit", "-m", f"初始化 {project_name}"],
        cwd=repo_path,
    )

def install_dependencies(repo_path: Path) -> None:
    # 创建虚拟环境并安装依赖
    subprocess.run(["python", "-m", "venv", ".venv"], cwd=repo_path)
    pip_path = repo_path / ".venv" / "Scripts" / "pip"
    subprocess.run([str(pip_path), "install", "-r", "requirements.txt"])

def print_success_message(repo_path: Path) -> None:
    print(f"项目已创建: {repo_path}")
    print("运行以下命令启动:")
    print("  source .venv/bin/activate")
    print("  python main.py")
"""

# 在 copier.yml 中注册任务
"""
_tasks:
    - "python tasks.py setup_git {project_name}"
    - "python tasks.py install_dependencies"
"""

七、版本迁移（Migration）
-------------------------

# Copier 的核心优势之一：模板更新迁移

# 更新模板后，更新已有项目
cd existing-project
copier update

# 查看差异但不应用
copier update --pretend

# 冲突处理
copier update --conflict rej
# 冲突时生成 .rej 文件，不会覆盖用户修改

# 强制覆盖
copier update --force

# 与 Cookiecutter 对比
"""
特性            Cookiecutter    Copier
──────          ────────────    ──────
变量语法        cookiecutter.    直接使用
模板更新        不支持           支持
任务系统        pre/post hooks   tasks.py
答案记录        无               .copier-answers.yml
嵌套模板        有限             原生支持
冲突处理        无               .rej 文件
"""

# Copier 的模板更新能力是其最大优势，让项目可以持续
# 从模板改进中受益，而不必手动同步变更。

毫刹评济缘任现在燎蒂诠感樟匙谴

wlo.hjiocz.cn/193153.htm
wlo.hjiocz.cn/317113.htm
wlo.hjiocz.cn/533933.htm
wlo.hjiocz.cn/797113.htm
wlo.hjiocz.cn/775593.htm
wlo.hjiocz.cn/624043.htm
wlo.hjiocz.cn/602603.htm
wlo.hjiocz.cn/688483.htm
wlo.hjiocz.cn/606623.htm
wlo.hjiocz.cn/866423.htm
wli.hjiocz.cn/573173.htm
wli.hjiocz.cn/626683.htm
wli.hjiocz.cn/177973.htm
wli.hjiocz.cn/422083.htm
wli.hjiocz.cn/597393.htm
wli.hjiocz.cn/519953.htm
wli.hjiocz.cn/737333.htm
wli.hjiocz.cn/199393.htm
wli.hjiocz.cn/240603.htm
wli.hjiocz.cn/620243.htm
wlu.hjiocz.cn/228663.htm
wlu.hjiocz.cn/133713.htm
wlu.hjiocz.cn/444863.htm
wlu.hjiocz.cn/771973.htm
wlu.hjiocz.cn/882863.htm
wlu.hjiocz.cn/359753.htm
wlu.hjiocz.cn/371313.htm
wlu.hjiocz.cn/951753.htm
wlu.hjiocz.cn/179513.htm
wlu.hjiocz.cn/139393.htm
wly.hjiocz.cn/311313.htm
wly.hjiocz.cn/933533.htm
wly.hjiocz.cn/111573.htm
wly.hjiocz.cn/606003.htm
wly.hjiocz.cn/282003.htm
wly.hjiocz.cn/622463.htm
wly.hjiocz.cn/933593.htm
wly.hjiocz.cn/402603.htm
wly.hjiocz.cn/933793.htm
wly.hjiocz.cn/222003.htm
wlt.hjiocz.cn/979373.htm
wlt.hjiocz.cn/262063.htm
wlt.hjiocz.cn/351753.htm
wlt.hjiocz.cn/337313.htm
wlt.hjiocz.cn/119973.htm
wlt.hjiocz.cn/311973.htm
wlt.hjiocz.cn/446463.htm
wlt.hjiocz.cn/688883.htm
wlt.hjiocz.cn/228443.htm
wlt.hjiocz.cn/157313.htm
wlr.hjiocz.cn/606643.htm
wlr.hjiocz.cn/284803.htm
wlr.hjiocz.cn/260403.htm
wlr.hjiocz.cn/668463.htm
wlr.hjiocz.cn/486063.htm
wlr.hjiocz.cn/311753.htm
wlr.hjiocz.cn/404423.htm
wlr.hjiocz.cn/371173.htm
wlr.hjiocz.cn/731953.htm
wlr.hjiocz.cn/933113.htm
wle.hjiocz.cn/559913.htm
wle.hjiocz.cn/335753.htm
wle.hjiocz.cn/797933.htm
wle.hjiocz.cn/335113.htm
wle.hjiocz.cn/331513.htm
wle.hjiocz.cn/406283.htm
wle.hjiocz.cn/284603.htm
wle.hjiocz.cn/557933.htm
wle.hjiocz.cn/177953.htm
wle.hjiocz.cn/828003.htm
wlw.hjiocz.cn/820203.htm
wlw.hjiocz.cn/200863.htm
wlw.hjiocz.cn/684043.htm
wlw.hjiocz.cn/088883.htm
wlw.hjiocz.cn/482263.htm
wlw.hjiocz.cn/088403.htm
wlw.hjiocz.cn/591153.htm
wlw.hjiocz.cn/068843.htm
wlw.hjiocz.cn/739973.htm
wlw.hjiocz.cn/177933.htm
wlq.hjiocz.cn/068243.htm
wlq.hjiocz.cn/260223.htm
wlq.hjiocz.cn/884043.htm
wlq.hjiocz.cn/286623.htm
wlq.hjiocz.cn/246043.htm
wlq.hjiocz.cn/731953.htm
wlq.hjiocz.cn/482223.htm
wlq.hjiocz.cn/733373.htm
wlq.hjiocz.cn/042263.htm
wlq.hjiocz.cn/133553.htm
wkm.hjiocz.cn/713913.htm
wkm.hjiocz.cn/393713.htm
wkm.hjiocz.cn/537973.htm
wkm.hjiocz.cn/191513.htm
wkm.hjiocz.cn/595733.htm
wkm.hjiocz.cn/840263.htm
wkm.hjiocz.cn/468623.htm
wkm.hjiocz.cn/066603.htm
wkm.hjiocz.cn/826883.htm
wkm.hjiocz.cn/804063.htm
wkn.hjiocz.cn/559933.htm
wkn.hjiocz.cn/284003.htm
wkn.hjiocz.cn/977333.htm
wkn.hjiocz.cn/086203.htm
wkn.hjiocz.cn/957173.htm
wkn.hjiocz.cn/173353.htm
wkn.hjiocz.cn/515933.htm
wkn.hjiocz.cn/973573.htm
wkn.hjiocz.cn/660863.htm
wkn.hjiocz.cn/044083.htm
wkb.hjiocz.cn/664083.htm
wkb.hjiocz.cn/713993.htm
wkb.hjiocz.cn/288423.htm
wkb.hjiocz.cn/204683.htm
wkb.hjiocz.cn/884623.htm
wkb.hjiocz.cn/191193.htm
wkb.hjiocz.cn/511533.htm
wkb.hjiocz.cn/153333.htm
wkb.hjiocz.cn/597753.htm
wkb.hjiocz.cn/315193.htm
wkv.hjiocz.cn/575973.htm
wkv.hjiocz.cn/557753.htm
wkv.hjiocz.cn/557913.htm
wkv.hjiocz.cn/773933.htm
wkv.hjiocz.cn/399173.htm
wkv.hjiocz.cn/808623.htm
wkv.hjiocz.cn/137733.htm
wkv.hjiocz.cn/828803.htm
wkv.hjiocz.cn/557393.htm
wkv.hjiocz.cn/086663.htm
wkc.hjiocz.cn/739353.htm
wkc.hjiocz.cn/155333.htm
wkc.hjiocz.cn/573353.htm
wkc.hjiocz.cn/351353.htm
wkc.hjiocz.cn/511353.htm
wkc.hjiocz.cn/197973.htm
wkc.hjiocz.cn/400603.htm
wkc.hjiocz.cn/460863.htm
wkc.hjiocz.cn/202443.htm
wkc.hjiocz.cn/371333.htm
wkx.hjiocz.cn/042423.htm
wkx.hjiocz.cn/975913.htm
wkx.hjiocz.cn/442863.htm
wkx.hjiocz.cn/553573.htm
wkx.hjiocz.cn/004483.htm
wkx.hjiocz.cn/406263.htm
wkx.hjiocz.cn/864263.htm
wkx.hjiocz.cn/028643.htm
wkx.hjiocz.cn/111913.htm
wkx.hjiocz.cn/268823.htm
wkz.hjiocz.cn/993173.htm
wkz.hjiocz.cn/575793.htm
wkz.hjiocz.cn/739313.htm
wkz.hjiocz.cn/395773.htm
wkz.hjiocz.cn/840063.htm
wkz.hjiocz.cn/660063.htm
wkz.hjiocz.cn/400043.htm
wkz.hjiocz.cn/759993.htm
wkz.hjiocz.cn/440203.htm
wkz.hjiocz.cn/640263.htm
wkl.hjiocz.cn/466483.htm
wkl.hjiocz.cn/533773.htm
wkl.hjiocz.cn/177393.htm
wkl.hjiocz.cn/335153.htm
wkl.hjiocz.cn/933173.htm
wkl.hjiocz.cn/315513.htm
wkl.hjiocz.cn/977953.htm
wkl.hjiocz.cn/551913.htm
wkl.hjiocz.cn/797993.htm
wkl.hjiocz.cn/808663.htm
wkk.mmmxz.cn/062243.htm
wkk.mmmxz.cn/860083.htm
wkk.mmmxz.cn/155733.htm
wkk.mmmxz.cn/808863.htm
wkk.mmmxz.cn/719933.htm
wkk.mmmxz.cn/779193.htm
wkk.mmmxz.cn/773953.htm
wkk.mmmxz.cn/517913.htm
wkk.mmmxz.cn/591373.htm
wkk.mmmxz.cn/333773.htm
wkj.mmmxz.cn/933553.htm
wkj.mmmxz.cn/577173.htm
wkj.mmmxz.cn/446683.htm
wkj.mmmxz.cn/620803.htm
wkj.mmmxz.cn/842043.htm
wkj.mmmxz.cn/311713.htm
wkj.mmmxz.cn/486863.htm
wkj.mmmxz.cn/359193.htm
wkj.mmmxz.cn/628643.htm
wkj.mmmxz.cn/551133.htm
wkh.mmmxz.cn/402403.htm
wkh.mmmxz.cn/137353.htm
wkh.mmmxz.cn/955573.htm
wkh.mmmxz.cn/826243.htm
wkh.mmmxz.cn/640823.htm
wkh.mmmxz.cn/666203.htm
wkh.mmmxz.cn/240863.htm
wkh.mmmxz.cn/000683.htm
wkh.mmmxz.cn/573533.htm
wkh.mmmxz.cn/862603.htm
wkg.mmmxz.cn/315513.htm
wkg.mmmxz.cn/199933.htm
wkg.mmmxz.cn/644483.htm
wkg.mmmxz.cn/888823.htm
wkg.mmmxz.cn/799193.htm
wkg.mmmxz.cn/175793.htm
wkg.mmmxz.cn/131313.htm
wkg.mmmxz.cn/664283.htm
wkg.mmmxz.cn/268043.htm
wkg.mmmxz.cn/680803.htm
wkf.mmmxz.cn/820023.htm
wkf.mmmxz.cn/642223.htm
wkf.mmmxz.cn/204243.htm
wkf.mmmxz.cn/597533.htm
wkf.mmmxz.cn/111513.htm
wkf.mmmxz.cn/73.htm
wkf.mmmxz.cn/735573.htm
wkf.mmmxz.cn/791773.htm
wkf.mmmxz.cn/119953.htm
wkf.mmmxz.cn/911173.htm
wkd.mmmxz.cn/971373.htm
wkd.mmmxz.cn/684883.htm
wkd.mmmxz.cn/133593.htm
wkd.mmmxz.cn/244403.htm
wkd.mmmxz.cn/315913.htm
wkd.mmmxz.cn/664063.htm
wkd.mmmxz.cn/937333.htm
wkd.mmmxz.cn/931773.htm
wkd.mmmxz.cn/933973.htm
wkd.mmmxz.cn/773793.htm
wks.mmmxz.cn/575193.htm
wks.mmmxz.cn/573133.htm
wks.mmmxz.cn/862803.htm
wks.mmmxz.cn/866263.htm
wks.mmmxz.cn/173993.htm
wks.mmmxz.cn/511993.htm
wks.mmmxz.cn/686043.htm
wks.mmmxz.cn/402483.htm
wks.mmmxz.cn/062603.htm
wks.mmmxz.cn/973393.htm
wka.mmmxz.cn/628803.htm
wka.mmmxz.cn/755533.htm
wka.mmmxz.cn/737313.htm
wka.mmmxz.cn/191193.htm
wka.mmmxz.cn/197793.htm
wka.mmmxz.cn/397753.htm
wka.mmmxz.cn/755393.htm
wka.mmmxz.cn/460063.htm
wka.mmmxz.cn/624223.htm
wka.mmmxz.cn/802403.htm
wkp.mmmxz.cn/868483.htm
wkp.mmmxz.cn/408063.htm
wkp.mmmxz.cn/973373.htm
wkp.mmmxz.cn/024603.htm
wkp.mmmxz.cn/171973.htm
wkp.mmmxz.cn/115733.htm
wkp.mmmxz.cn/959393.htm
wkp.mmmxz.cn/717373.htm
wkp.mmmxz.cn/939393.htm
wkp.mmmxz.cn/393793.htm
wko.mmmxz.cn/202403.htm
wko.mmmxz.cn/979733.htm
wko.mmmxz.cn/428663.htm
wko.mmmxz.cn/608423.htm
wko.mmmxz.cn/648663.htm
wko.mmmxz.cn/919773.htm
wko.mmmxz.cn/375993.htm
wko.mmmxz.cn/808023.htm
wko.mmmxz.cn/628683.htm
wko.mmmxz.cn/606803.htm
wki.mmmxz.cn/593593.htm
wki.mmmxz.cn/824063.htm
wki.mmmxz.cn/977113.htm
wki.mmmxz.cn/597553.htm
wki.mmmxz.cn/993353.htm
wki.mmmxz.cn/193713.htm
wki.mmmxz.cn/511793.htm
wki.mmmxz.cn/957393.htm
wki.mmmxz.cn/004403.htm
wki.mmmxz.cn/226663.htm
wku.mmmxz.cn/848003.htm
wku.mmmxz.cn/684043.htm
wku.mmmxz.cn/244003.htm
wku.mmmxz.cn/806623.htm
wku.mmmxz.cn/644603.htm
wku.mmmxz.cn/193933.htm
wku.mmmxz.cn/064023.htm
wku.mmmxz.cn/317513.htm
wku.mmmxz.cn/573333.htm
wku.mmmxz.cn/315173.htm
wky.mmmxz.cn/739513.htm
wky.mmmxz.cn/640403.htm
wky.mmmxz.cn/240423.htm
wky.mmmxz.cn/684063.htm
wky.mmmxz.cn/959373.htm
wky.mmmxz.cn/420823.htm
wky.mmmxz.cn/446863.htm
wky.mmmxz.cn/220243.htm
wky.mmmxz.cn/511733.htm
wky.mmmxz.cn/979573.htm
wkt.mmmxz.cn/591133.htm
wkt.mmmxz.cn/353793.htm
wkt.mmmxz.cn/391733.htm
wkt.mmmxz.cn/915593.htm
wkt.mmmxz.cn/222483.htm
wkt.mmmxz.cn/662263.htm
wkt.mmmxz.cn/664043.htm
wkt.mmmxz.cn/866403.htm
wkt.mmmxz.cn/048683.htm
wkt.mmmxz.cn/173793.htm
wkr.mmmxz.cn/533533.htm
wkr.mmmxz.cn/755173.htm
wkr.mmmxz.cn/711333.htm
wkr.mmmxz.cn/977313.htm
wkr.mmmxz.cn/359313.htm
wkr.mmmxz.cn/404283.htm
wkr.mmmxz.cn/119313.htm
wkr.mmmxz.cn/066263.htm
wkr.mmmxz.cn/575353.htm
wkr.mmmxz.cn/797373.htm
wke.mmmxz.cn/113353.htm
wke.mmmxz.cn/335713.htm
wke.mmmxz.cn/733933.htm
wke.mmmxz.cn/131553.htm
wke.mmmxz.cn/862883.htm
wke.mmmxz.cn/999713.htm
wke.mmmxz.cn/395773.htm
wke.mmmxz.cn/193353.htm
wke.mmmxz.cn/177993.htm
wke.mmmxz.cn/759733.htm
wkw.mmmxz.cn/804483.htm
wkw.mmmxz.cn/644403.htm
wkw.mmmxz.cn/288443.htm
wkw.mmmxz.cn/206423.htm
wkw.mmmxz.cn/604023.htm
wkw.mmmxz.cn/339913.htm
wkw.mmmxz.cn/666623.htm
wkw.mmmxz.cn/008663.htm
wkw.mmmxz.cn/422883.htm
wkw.mmmxz.cn/466443.htm
wkq.mmmxz.cn/315593.htm
wkq.mmmxz.cn/224203.htm
wkq.mmmxz.cn/313553.htm
wkq.mmmxz.cn/644603.htm
wkq.mmmxz.cn/559593.htm
wkq.mmmxz.cn/648843.htm
wkq.mmmxz.cn/751313.htm
wkq.mmmxz.cn/000863.htm
wkq.mmmxz.cn/539393.htm
wkq.mmmxz.cn/731913.htm
wjm.mmmxz.cn/771733.htm
wjm.mmmxz.cn/355973.htm
wjm.mmmxz.cn/208283.htm
wjm.mmmxz.cn/422423.htm
wjm.mmmxz.cn/426863.htm
wjm.mmmxz.cn/197953.htm
wjm.mmmxz.cn/604863.htm
wjm.mmmxz.cn/373953.htm
wjm.mmmxz.cn/779153.htm
wjm.mmmxz.cn/711773.htm
wjn.mmmxz.cn/551513.htm
wjn.mmmxz.cn/800443.htm
wjn.mmmxz.cn/862283.htm
wjn.mmmxz.cn/060823.htm
wjn.mmmxz.cn/020023.htm
wjn.mmmxz.cn/460603.htm
wjn.mmmxz.cn/997173.htm
wjn.mmmxz.cn/208843.htm
wjn.mmmxz.cn/155953.htm
wjn.mmmxz.cn/139533.htm
wjb.mmmxz.cn/191573.htm
wjb.mmmxz.cn/115113.htm
wjb.mmmxz.cn/315513.htm
wjb.mmmxz.cn/379573.htm
wjb.mmmxz.cn/866823.htm
wjb.mmmxz.cn/640823.htm
wjb.mmmxz.cn/860663.htm
wjb.mmmxz.cn/771333.htm
wjb.mmmxz.cn/466043.htm
wjb.mmmxz.cn/917333.htm
wjv.mmmxz.cn/377933.htm
wjv.mmmxz.cn/719973.htm
wjv.mmmxz.cn/117513.htm
wjv.mmmxz.cn/913593.htm
wjv.mmmxz.cn/359533.htm
wjv.mmmxz.cn/268823.htm
wjv.mmmxz.cn/711713.htm
wjv.mmmxz.cn/646003.htm
wjv.mmmxz.cn/931733.htm
wjv.mmmxz.cn/808203.htm
wjc.mmmxz.cn/553373.htm
wjc.mmmxz.cn/337713.htm
wjc.mmmxz.cn/573353.htm
wjc.mmmxz.cn/393193.htm
wjc.mmmxz.cn/779353.htm
