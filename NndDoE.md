咳肚尚沮歉


Python 虚拟环境管理详解
=================================

虚拟环境是 Python 项目隔离依赖的核心手段，确保不同项目使用
互不干扰的包版本。本文介绍多种虚拟环境管理工具。

一、venv —— Python 内置虚拟环境
--------------------------------

# venv 是 Python 3.3+ 自带的虚拟环境模块，无需额外安装

# 创建虚拟环境
python -m venv .venv
# 在 .venv 目录下创建独立的 Python 环境

# 激活虚拟环境
# Windows 系统：
# .venv\\Scripts\\activate

# Linux / macOS 系统：
# source .venv/bin/activate

# 激活后提示符前出现 (.venv) 标志
# 此时 pip install 安装的包将进入虚拟环境

# 退出虚拟环境
deactivate

# 删除虚拟环境（直接删除目录）
rm -rf .venv
# 或 Windows: rmdir /s .venv

# 指定 Python 版本创建
python3.11 -m venv .venv

# 创建时不安装 pip 和 setuptools
python -m venv .venv --without-pip

二、pyenv —— Python 版本管理
-----------------------------

# pyenv 用于管理多个 Python 版本，而非虚拟环境
# 安装 pyenv（Linux/macOS）
# curl https://pyenv.run | bash

# 安装特定 Python 版本
pyenv install 3.11.5

# 列出所有可安装的版本
pyenv install --list

# 查看已安装的版本
pyenv versions

# 设置全局 Python 版本
pyenv global 3.11.5

# 设置当前目录的 Python 版本
pyenv local 3.10.0
# 会在目录中创建 .python-version 文件

# 设置 shell 会话的 Python 版本
pyenv shell 3.11.5

# pyenv 与虚拟环境配合使用
# 安装 pyenv-virtualenv 插件
# pyenv virtualenv 3.11.5 myproject-env
# pyenv activate myproject-env

三、virtualenvwrapper —— 增强的虚拟环境管理
--------------------------------------------

# virtualenvwrapper 提供集中化的虚拟环境管理
# pip install virtualenvwrapper  # Linux/macOS
# pip install virtualenvwrapper-win  # Windows

# 设置工作目录
# export WORKON_HOME=~/.virtualenvs
# source /usr/local/bin/virtualenvwrapper.sh

# 创建虚拟环境
mkvirtualenv myproject
# 自动激活新创建的环境

# 列出所有虚拟环境
workon

# 切换到指定环境
workon myproject

# 退出当前环境
deactivate

# 删除虚拟环境
rmvirtualenv myproject

# 在虚拟环境中运行命令
runvirtualenv myproject python script.py

# 查看当前环境的 site-packages 路径
showvirtualenv myproject

# 复制虚拟环境
cpvirtualenv myproject myproject-backup

四、pipenv —— 一站式工具
-------------------------

# pipenv 融合了 pip 和 virtualenv 的功能
# pip install pipenv

# 初始化项目并创建虚拟环境
pipenv install
# 自动检测并创建虚拟环境，生成 Pipfile

# 安装包
pipenv install requests

# 安装开发依赖
pipenv install --dev pytest

# 安装指定版本
pipenv install "django>=4.0,<5.0"

# 生成锁定文件
pipenv lock

# 安装锁定文件中的依赖
pipenv sync

# 激活虚拟环境
pipenv shell

# 运行命令而不进入环境
pipenv run python main.py

# 删除虚拟环境
pipenv --rm

# 查看依赖关系
pipenv graph

五、虚拟环境最佳实践
--------------------

# 1. 总是使用虚拟环境
# 每个项目应有独立的虚拟环境

# 2. 命名规范
# · 使用 .venv 作为虚拟环境目录名（标准约定）
# · 用于 .gitignore 排除
echo ".venv/" >> .gitignore

# 3. 保存依赖清单
pip freeze > requirements.txt
# 或使用 pipenv/poetry 自动管理

# 4. 恢复环境
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 5. pyenv + venv 组合使用
"""
# 设置 Python 版本
pyenv local 3.11.5

# 创建虚拟环境
python -m venv .venv

# 激活
source .venv/bin/activate

# 确认 Python 版本
python --version
# 输出: Python 3.11.5
"""

六、常见问题与排查
------------------

# 检查当前 Python 和 pip 路径
which python
which pip
# 确认是否指向虚拟环境内的路径

# 确认虚拟环境是否激活
python -c "import sys; print(sys.prefix)"

# pip 安装包时报错权限问题
# 确保虚拟环境已激活，不要使用 sudo pip

# requirements.txt 中固定版本
"""
# requirements.txt 示例
flask==2.3.3
requests==2.31.0
pydantic==2.5.0
"""

# 生成精确版本清单
pip list --format=freeze > requirements.txt

# 虚拟环境是现代 Python 开发的基石，掌握多种管理工具
# 可以根据项目需求选择最合适的方案。

握俏滩律现嘲肚诙裁对导乃侥凶九

m.wwl.hzldf.cn/40480.Doc
m.wwl.hzldf.cn/86882.Doc
m.wwl.hzldf.cn/42828.Doc
m.wwl.hzldf.cn/06842.Doc
m.wwl.hzldf.cn/08860.Doc
m.wwl.hzldf.cn/95397.Doc
m.wwl.hzldf.cn/64620.Doc
m.wwl.hzldf.cn/66220.Doc
m.wwl.hzldf.cn/55539.Doc
m.wwl.hzldf.cn/97515.Doc
m.wwk.hzldf.cn/08884.Doc
m.wwk.hzldf.cn/44826.Doc
m.wwk.hzldf.cn/31171.Doc
m.wwk.hzldf.cn/66426.Doc
m.wwk.hzldf.cn/24262.Doc
m.wwk.hzldf.cn/57391.Doc
m.wwk.hzldf.cn/00800.Doc
m.wwk.hzldf.cn/59339.Doc
m.wwk.hzldf.cn/82048.Doc
m.wwk.hzldf.cn/40808.Doc
m.wwk.hzldf.cn/55393.Doc
m.wwk.hzldf.cn/93519.Doc
m.wwk.hzldf.cn/42826.Doc
m.wwk.hzldf.cn/46426.Doc
m.wwk.hzldf.cn/06280.Doc
m.wwk.hzldf.cn/46248.Doc
m.wwk.hzldf.cn/08248.Doc
m.wwk.hzldf.cn/84044.Doc
m.wwk.hzldf.cn/66428.Doc
m.wwk.hzldf.cn/06240.Doc
m.wwj.hzldf.cn/64626.Doc
m.wwj.hzldf.cn/48846.Doc
m.wwj.hzldf.cn/88040.Doc
m.wwj.hzldf.cn/77993.Doc
m.wwj.hzldf.cn/33331.Doc
m.wwj.hzldf.cn/44640.Doc
m.wwj.hzldf.cn/20680.Doc
m.wwj.hzldf.cn/31159.Doc
m.wwj.hzldf.cn/48248.Doc
m.wwj.hzldf.cn/26860.Doc
m.wwj.hzldf.cn/06022.Doc
m.wwj.hzldf.cn/86408.Doc
m.wwj.hzldf.cn/35173.Doc
m.wwj.hzldf.cn/88826.Doc
m.wwj.hzldf.cn/00286.Doc
m.wwj.hzldf.cn/86028.Doc
m.wwj.hzldf.cn/48604.Doc
m.wwj.hzldf.cn/95393.Doc
m.wwj.hzldf.cn/40222.Doc
m.wwj.hzldf.cn/06822.Doc
m.wwh.hzldf.cn/02822.Doc
m.wwh.hzldf.cn/20080.Doc
m.wwh.hzldf.cn/35739.Doc
m.wwh.hzldf.cn/57991.Doc
m.wwh.hzldf.cn/88888.Doc
m.wwh.hzldf.cn/08400.Doc
m.wwh.hzldf.cn/66000.Doc
m.wwh.hzldf.cn/20480.Doc
m.wwh.hzldf.cn/55395.Doc
m.wwh.hzldf.cn/77579.Doc
m.wwh.hzldf.cn/37775.Doc
m.wwh.hzldf.cn/64804.Doc
m.wwh.hzldf.cn/28042.Doc
m.wwh.hzldf.cn/62208.Doc
m.wwh.hzldf.cn/86440.Doc
m.wwh.hzldf.cn/84266.Doc
m.wwh.hzldf.cn/08640.Doc
m.wwh.hzldf.cn/51555.Doc
m.wwh.hzldf.cn/04086.Doc
m.wwh.hzldf.cn/24660.Doc
m.wwg.hzldf.cn/22844.Doc
m.wwg.hzldf.cn/15997.Doc
m.wwg.hzldf.cn/39397.Doc
m.wwg.hzldf.cn/95351.Doc
m.wwg.hzldf.cn/88224.Doc
m.wwg.hzldf.cn/26480.Doc
m.wwg.hzldf.cn/06848.Doc
m.wwg.hzldf.cn/86648.Doc
m.wwg.hzldf.cn/66002.Doc
m.wwg.hzldf.cn/55937.Doc
m.wwg.hzldf.cn/44828.Doc
m.wwg.hzldf.cn/42668.Doc
m.wwg.hzldf.cn/06208.Doc
m.wwg.hzldf.cn/22064.Doc
m.wwg.hzldf.cn/42686.Doc
m.wwg.hzldf.cn/24408.Doc
m.wwg.hzldf.cn/44226.Doc
m.wwg.hzldf.cn/62848.Doc
m.wwg.hzldf.cn/28400.Doc
m.wwg.hzldf.cn/91795.Doc
m.wwf.hzldf.cn/40808.Doc
m.wwf.hzldf.cn/00006.Doc
m.wwf.hzldf.cn/22046.Doc
m.wwf.hzldf.cn/22226.Doc
m.wwf.hzldf.cn/57755.Doc
m.wwf.hzldf.cn/00028.Doc
m.wwf.hzldf.cn/48848.Doc
m.wwf.hzldf.cn/28886.Doc
m.wwf.hzldf.cn/37517.Doc
m.wwf.hzldf.cn/22004.Doc
m.wwf.hzldf.cn/66660.Doc
m.wwf.hzldf.cn/40220.Doc
m.wwf.hzldf.cn/40640.Doc
m.wwf.hzldf.cn/91353.Doc
m.wwf.hzldf.cn/97197.Doc
m.wwf.hzldf.cn/59711.Doc
m.wwf.hzldf.cn/04684.Doc
m.wwf.hzldf.cn/35719.Doc
m.wwf.hzldf.cn/62082.Doc
m.wwf.hzldf.cn/13737.Doc
m.wwd.hzldf.cn/06844.Doc
m.wwd.hzldf.cn/44460.Doc
m.wwd.hzldf.cn/57153.Doc
m.wwd.hzldf.cn/11733.Doc
m.wwd.hzldf.cn/91731.Doc
m.wwd.hzldf.cn/44668.Doc
m.wwd.hzldf.cn/62044.Doc
m.wwd.hzldf.cn/91317.Doc
m.wwd.hzldf.cn/93171.Doc
m.wwd.hzldf.cn/00002.Doc
m.wwd.hzldf.cn/82648.Doc
m.wwd.hzldf.cn/60400.Doc
m.wwd.hzldf.cn/00080.Doc
m.wwd.hzldf.cn/20642.Doc
m.wwd.hzldf.cn/75593.Doc
m.wwd.hzldf.cn/62260.Doc
m.wwd.hzldf.cn/80808.Doc
m.wwd.hzldf.cn/04020.Doc
m.wwd.hzldf.cn/48288.Doc
m.wwd.hzldf.cn/46004.Doc
m.wws.hzldf.cn/80828.Doc
m.wws.hzldf.cn/64486.Doc
m.wws.hzldf.cn/57555.Doc
m.wws.hzldf.cn/24666.Doc
m.wws.hzldf.cn/79519.Doc
m.wws.hzldf.cn/82468.Doc
m.wws.hzldf.cn/62800.Doc
m.wws.hzldf.cn/02424.Doc
m.wws.hzldf.cn/06820.Doc
m.wws.hzldf.cn/82226.Doc
m.wws.hzldf.cn/62202.Doc
m.wws.hzldf.cn/82886.Doc
m.wws.hzldf.cn/68462.Doc
m.wws.hzldf.cn/75171.Doc
m.wws.hzldf.cn/99119.Doc
m.wws.hzldf.cn/15797.Doc
m.wws.hzldf.cn/80042.Doc
m.wws.hzldf.cn/40624.Doc
m.wws.hzldf.cn/68040.Doc
m.wws.hzldf.cn/71917.Doc
m.wwa.hzldf.cn/00688.Doc
m.wwa.hzldf.cn/28282.Doc
m.wwa.hzldf.cn/84462.Doc
m.wwa.hzldf.cn/08446.Doc
m.wwa.hzldf.cn/66888.Doc
m.wwa.hzldf.cn/46666.Doc
m.wwa.hzldf.cn/48686.Doc
m.wwa.hzldf.cn/39575.Doc
m.wwa.hzldf.cn/28846.Doc
m.wwa.hzldf.cn/84202.Doc
m.wwa.hzldf.cn/59977.Doc
m.wwa.hzldf.cn/02042.Doc
m.wwa.hzldf.cn/64644.Doc
m.wwa.hzldf.cn/82202.Doc
m.wwa.hzldf.cn/06606.Doc
m.wwa.hzldf.cn/19117.Doc
m.wwa.hzldf.cn/95995.Doc
m.wwa.hzldf.cn/40826.Doc
m.wwa.hzldf.cn/31155.Doc
m.wwa.hzldf.cn/82680.Doc
m.wwp.hzldf.cn/24644.Doc
m.wwp.hzldf.cn/82406.Doc
m.wwp.hzldf.cn/17757.Doc
m.wwp.hzldf.cn/99711.Doc
m.wwp.hzldf.cn/31319.Doc
m.wwp.hzldf.cn/42800.Doc
m.wwp.hzldf.cn/86884.Doc
m.wwp.hzldf.cn/00620.Doc
m.wwp.hzldf.cn/68482.Doc
m.wwp.hzldf.cn/80042.Doc
m.wwp.hzldf.cn/84648.Doc
m.wwp.hzldf.cn/95995.Doc
m.wwp.hzldf.cn/48608.Doc
m.wwp.hzldf.cn/35735.Doc
m.wwp.hzldf.cn/33799.Doc
m.wwp.hzldf.cn/24808.Doc
m.wwp.hzldf.cn/95131.Doc
m.wwp.hzldf.cn/08640.Doc
m.wwp.hzldf.cn/33739.Doc
m.wwp.hzldf.cn/33999.Doc
m.wwo.hzldf.cn/06406.Doc
m.wwo.hzldf.cn/88802.Doc
m.wwo.hzldf.cn/40062.Doc
m.wwo.hzldf.cn/66008.Doc
m.wwo.hzldf.cn/20680.Doc
m.wwo.hzldf.cn/55737.Doc
m.wwo.hzldf.cn/68246.Doc
m.wwo.hzldf.cn/82640.Doc
m.wwo.hzldf.cn/40422.Doc
m.wwo.hzldf.cn/60266.Doc
m.wwo.hzldf.cn/80402.Doc
m.wwo.hzldf.cn/71717.Doc
m.wwo.hzldf.cn/46440.Doc
m.wwo.hzldf.cn/02260.Doc
m.wwo.hzldf.cn/86808.Doc
m.wwo.hzldf.cn/62200.Doc
m.wwo.hzldf.cn/60604.Doc
m.wwo.hzldf.cn/20644.Doc
m.wwo.hzldf.cn/93711.Doc
m.wwo.hzldf.cn/15315.Doc
m.wwi.hzldf.cn/95153.Doc
m.wwi.hzldf.cn/08200.Doc
m.wwi.hzldf.cn/77599.Doc
m.wwi.hzldf.cn/82444.Doc
m.wwi.hzldf.cn/55175.Doc
m.wwi.hzldf.cn/13591.Doc
m.wwi.hzldf.cn/31559.Doc
m.wwi.hzldf.cn/20688.Doc
m.wwi.hzldf.cn/46860.Doc
m.wwi.hzldf.cn/33339.Doc
m.wwi.hzldf.cn/33179.Doc
m.wwi.hzldf.cn/39575.Doc
m.wwi.hzldf.cn/55315.Doc
m.wwi.hzldf.cn/26222.Doc
m.wwi.hzldf.cn/82868.Doc
m.wwi.hzldf.cn/64868.Doc
m.wwi.hzldf.cn/35311.Doc
m.wwi.hzldf.cn/02842.Doc
m.wwi.hzldf.cn/11331.Doc
m.wwi.hzldf.cn/55571.Doc
m.wwu.hzldf.cn/64628.Doc
m.wwu.hzldf.cn/22662.Doc
m.wwu.hzldf.cn/39735.Doc
m.wwu.hzldf.cn/91751.Doc
m.wwu.hzldf.cn/20484.Doc
m.wwu.hzldf.cn/68460.Doc
m.wwu.hzldf.cn/08482.Doc
m.wwu.hzldf.cn/31995.Doc
m.wwu.hzldf.cn/40642.Doc
m.wwu.hzldf.cn/84046.Doc
m.wwu.hzldf.cn/26046.Doc
m.wwu.hzldf.cn/26884.Doc
m.wwu.hzldf.cn/84468.Doc
m.wwu.hzldf.cn/44628.Doc
m.wwu.hzldf.cn/60064.Doc
m.wwu.hzldf.cn/71397.Doc
m.wwu.hzldf.cn/93999.Doc
m.wwu.hzldf.cn/64840.Doc
m.wwu.hzldf.cn/62066.Doc
m.wwu.hzldf.cn/66046.Doc
m.wwy.hzldf.cn/79917.Doc
m.wwy.hzldf.cn/46004.Doc
m.wwy.hzldf.cn/88424.Doc
m.wwy.hzldf.cn/28260.Doc
m.wwy.hzldf.cn/40642.Doc
m.wwy.hzldf.cn/04680.Doc
m.wwy.hzldf.cn/22082.Doc
m.wwy.hzldf.cn/20606.Doc
m.wwy.hzldf.cn/06284.Doc
m.wwy.hzldf.cn/88426.Doc
m.wwy.hzldf.cn/68488.Doc
m.wwy.hzldf.cn/22084.Doc
m.wwy.hzldf.cn/42888.Doc
m.wwy.hzldf.cn/26246.Doc
m.wwy.hzldf.cn/91337.Doc
m.wwy.hzldf.cn/28848.Doc
m.wwy.hzldf.cn/08862.Doc
m.wwy.hzldf.cn/55151.Doc
m.wwy.hzldf.cn/00800.Doc
m.wwy.hzldf.cn/42208.Doc
m.wwt.hzldf.cn/17119.Doc
m.wwt.hzldf.cn/62266.Doc
m.wwt.hzldf.cn/28406.Doc
m.wwt.hzldf.cn/06624.Doc
m.wwt.hzldf.cn/22440.Doc
m.wwt.hzldf.cn/11513.Doc
m.wwt.hzldf.cn/77957.Doc
m.wwt.hzldf.cn/00064.Doc
m.wwt.hzldf.cn/35335.Doc
m.wwt.hzldf.cn/80806.Doc
m.wwt.hzldf.cn/88802.Doc
m.wwt.hzldf.cn/86862.Doc
m.wwt.hzldf.cn/26284.Doc
m.wwt.hzldf.cn/44464.Doc
m.wwt.hzldf.cn/02080.Doc
m.wwt.hzldf.cn/86468.Doc
m.wwt.hzldf.cn/00440.Doc
m.wwt.hzldf.cn/97957.Doc
m.wwt.hzldf.cn/28446.Doc
m.wwt.hzldf.cn/20200.Doc
m.wwr.hzldf.cn/79539.Doc
m.wwr.hzldf.cn/77999.Doc
m.wwr.hzldf.cn/20068.Doc
m.wwr.hzldf.cn/26066.Doc
m.wwr.hzldf.cn/28664.Doc
m.wwr.hzldf.cn/75557.Doc
m.wwr.hzldf.cn/00680.Doc
m.wwr.hzldf.cn/66880.Doc
m.wwr.hzldf.cn/02068.Doc
m.wwr.hzldf.cn/60800.Doc
m.wwr.hzldf.cn/35755.Doc
m.wwr.hzldf.cn/22224.Doc
m.wwr.hzldf.cn/86628.Doc
m.wwr.hzldf.cn/66284.Doc
m.wwr.hzldf.cn/82600.Doc
m.wwr.hzldf.cn/88624.Doc
m.wwr.hzldf.cn/24266.Doc
m.wwr.hzldf.cn/84820.Doc
m.wwr.hzldf.cn/79313.Doc
m.wwr.hzldf.cn/91719.Doc
m.wwe.hzldf.cn/88260.Doc
m.wwe.hzldf.cn/84288.Doc
m.wwe.hzldf.cn/86048.Doc
m.wwe.hzldf.cn/04824.Doc
m.wwe.hzldf.cn/46808.Doc
m.wwe.hzldf.cn/86264.Doc
m.wwe.hzldf.cn/04266.Doc
m.wwe.hzldf.cn/17351.Doc
m.wwe.hzldf.cn/00428.Doc
m.wwe.hzldf.cn/04484.Doc
m.wwe.hzldf.cn/26440.Doc
m.wwe.hzldf.cn/15977.Doc
m.wwe.hzldf.cn/31391.Doc
m.wwe.hzldf.cn/48288.Doc
m.wwe.hzldf.cn/15913.Doc
m.wwe.hzldf.cn/24844.Doc
m.wwe.hzldf.cn/57917.Doc
m.wwe.hzldf.cn/71591.Doc
m.wwe.hzldf.cn/60248.Doc
m.wwe.hzldf.cn/22048.Doc
m.www.hzldf.cn/91179.Doc
m.www.hzldf.cn/73195.Doc
m.www.hzldf.cn/22220.Doc
m.www.hzldf.cn/88404.Doc
m.www.hzldf.cn/64826.Doc
m.www.hzldf.cn/13119.Doc
m.www.hzldf.cn/44244.Doc
m.www.hzldf.cn/68082.Doc
m.www.hzldf.cn/95919.Doc
m.www.hzldf.cn/51119.Doc
m.www.hzldf.cn/46420.Doc
m.www.hzldf.cn/88826.Doc
m.www.hzldf.cn/44042.Doc
m.www.hzldf.cn/02268.Doc
m.www.hzldf.cn/86028.Doc
m.www.hzldf.cn/82688.Doc
m.www.hzldf.cn/22202.Doc
m.www.hzldf.cn/62244.Doc
m.www.hzldf.cn/66882.Doc
m.www.hzldf.cn/60202.Doc
m.wwq.hzldf.cn/22028.Doc
m.wwq.hzldf.cn/82060.Doc
m.wwq.hzldf.cn/00200.Doc
m.wwq.hzldf.cn/46846.Doc
m.wwq.hzldf.cn/59935.Doc
m.wwq.hzldf.cn/46462.Doc
m.wwq.hzldf.cn/02866.Doc
m.wwq.hzldf.cn/88842.Doc
m.wwq.hzldf.cn/44208.Doc
m.wwq.hzldf.cn/00600.Doc
m.wwq.hzldf.cn/00608.Doc
m.wwq.hzldf.cn/31733.Doc
m.wwq.hzldf.cn/39177.Doc
m.wwq.hzldf.cn/48468.Doc
m.wwq.hzldf.cn/26628.Doc
m.wwq.hzldf.cn/66886.Doc
m.wwq.hzldf.cn/71573.Doc
m.wwq.hzldf.cn/42448.Doc
m.wwq.hzldf.cn/62262.Doc
m.wwq.hzldf.cn/26044.Doc
m.wqm.hzldf.cn/37993.Doc
m.wqm.hzldf.cn/35773.Doc
m.wqm.hzldf.cn/84628.Doc
m.wqm.hzldf.cn/26286.Doc
m.wqm.hzldf.cn/04008.Doc
m.wqm.hzldf.cn/28004.Doc
m.wqm.hzldf.cn/42242.Doc
m.wqm.hzldf.cn/48426.Doc
m.wqm.hzldf.cn/37351.Doc
m.wqm.hzldf.cn/88604.Doc
m.wqm.hzldf.cn/75539.Doc
m.wqm.hzldf.cn/75959.Doc
m.wqm.hzldf.cn/64008.Doc
m.wqm.hzldf.cn/48226.Doc
m.wqm.hzldf.cn/24446.Doc
m.wqm.hzldf.cn/33173.Doc
m.wqm.hzldf.cn/88220.Doc
m.wqm.hzldf.cn/08882.Doc
m.wqm.hzldf.cn/95315.Doc
m.wqm.hzldf.cn/99337.Doc
m.wqn.hzldf.cn/68228.Doc
m.wqn.hzldf.cn/55997.Doc
m.wqn.hzldf.cn/00086.Doc
m.wqn.hzldf.cn/40600.Doc
m.wqn.hzldf.cn/55371.Doc
