Python Alembic 数据库迁移完全指南
====================================

一、Alembic 环境初始化
----------------------

Alembic 是 SQLAlchemy 的数据库迁移工具，用于版本化管理数据库 Schema。
首先使用命令行初始化迁移环境。

"""
# 在项目根目录执行以下命令：
# alembic init alembic          # 创建 alembic 目录和配置文件
# 初始化后生成目录结构：
# alembic/
#   env.py          # 运行环境配置，连接数据库和模型
#   script.py.mako  # 迁移脚本模板
#   versions/       # 存放生成的迁移脚本
# alembic.ini       # 主配置文件，设置数据库 URL 等
"""

# --- alembic/env.py 核心配置 ---
"""
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context

# 导入你的 Base 模型，让 Alembic 能检测模型变更
from app.models import Base

config = context.config

# 设置数据库 URL（可从环境变量读取，避免密码硬编码）
# config.set_main_option("sqlalchemy.url", "postgresql://user:pass@localhost/db")

# 设置目标元数据 —— Alembic 通过对比它检测变更
target_metadata = Base.metadata

def run_migrations_offline():
    '''离线模式：生成 SQL 脚本而不连接数据库'''
    url = config.get_main_option("sqlalchemy.url")
    context.configure(url=url, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online():
    '''在线模式：连接数据库执行迁移'''
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool)
    with connectable.connect() as connection:
        context.configure(connection=connection,
                          target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
"""


二、自动生成迁移脚本
--------------------

修改模型后，使用 --autogenerate 自动生成迁移脚本。

"""
# 命令格式：
# alembic revision --autogenerate -m "添加用户表"

# 自动生成的迁移脚本位于 alembic/versions/ 目录下
# 脚本包含 upgrade() 和 downgrade() 两个函数
"""

# --- 示例：自动生成的迁移脚本 ---
"""
\"\"\"添加用户表

Revision ID: a1b2c3d4e5f6
Revises: None  # 前一个版本的 ID，None 表示第一个迁移
Create Date: 2026-05-27 10:30:00.000000
\"\"\"
from alembic import op
import sqlalchemy as sa

# revision identifiers, used by Alembic
revision = 'a1b2c3d4e5f6'
down_revision = None  # 依赖的上一个版本
branch_labels = None
depends_on = None

def upgrade():
    '''升级操作：创建用户表'''
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('name', sa.String(length=50), nullable=False),
        sa.Column('email', sa.String(length=120), nullable=True),
        sa.Column('created_at', sa.DateTime(), server_default=sa.text('NOW()')),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('email')
    )
    # 创建索引加速查询
    op.create_index('idx_users_name', 'users', ['name'])

def downgrade():
    '''降级操作：删除用户表'''
    op.drop_index('idx_users_name', table_name='users')
    op.drop_table('users')
"""


三、自定义迁移操作
------------------

自动生成无法覆盖所有场景，需要手写自定义迁移逻辑。

def upgrade():
    """自定义迁移：添加字段并设置默认值"""
    # 添加可为空的字段
    op.add_column('users', sa.Column('avatar_url', sa.String(256), nullable=True))

    # 添加非空字段：先添加可为空，填充数据，再设为非空
    op.add_column('users', sa.Column('nickname', sa.String(50), nullable=True))

    # 执行 SQL 更新已有数据的 nickname 为 name 的值
    op.execute("UPDATE users SET nickname = name WHERE nickname IS NULL")

    # 修改字段为 NOT NULL
    op.alter_column('users', 'nickname', nullable=False)

    # 修改字段类型（注意数据兼容性）
    op.alter_column('users', 'email',
                    type_=sa.String(200),
                    existing_type=sa.String(120),
                    nullable=True)

    # 添加复合唯一约束
    op.create_unique_constraint('uq_users_name_email', 'users', ['name', 'email'])

    # 添加外键约束
    op.create_foreign_key('fk_articles_authors', 'articles', 'authors',
                          ['author_id'], ['id'],
                          ondelete='CASCADE')

def downgrade():
    """回滚操作：与 upgrade 完全相反"""
    op.drop_constraint('fk_articles_authors', 'articles', type_='foreignkey')
    op.drop_constraint('uq_users_name_email', 'users', type_='unique')
    op.alter_column('users', 'email', type_=sa.String(120), nullable=True)
    op.drop_column('users', 'nickname')
    op.drop_column('users', 'avatar_url')


四、数据迁移（在迁移脚本中操作数据）
------------------------------------

Schema 变更后常常需要数据迁移（backfill）。
在同一个迁移脚本中，upgrade() 里先改 Schema，再迁移数据。

def upgrade():
    """Schema + 数据迁移"""
    # 第一步：Schema 变更 —— 添加角色字段
    op.add_column('users', sa.Column('role', sa.String(20),
                  server_default='user'))  # 设置默认值避免已有数据为 NULL

    # 第二步：数据迁移 —— 根据条件更新角色
    # 从数据库中获取连接执行数据操作
    op.execute(
        "UPDATE users SET role = 'admin' "
        "WHERE email LIKE '%@admin.com'"
    )
    op.execute(
        "UPDATE users SET role = 'vip' "
        "WHERE email LIKE '%@vip.com'"
    )

    # 第三步：可以增加索引来优化新的查询
    op.create_index('idx_users_role', 'users', ['role'])

def downgrade():
    """回滚时倒序执行"""
    op.drop_index('idx_users_role', table_name='users')
    op.drop_column('users', 'role')


五、分支标签与合并
------------------

在多人协作时，可能出现多个迁移分支，需要用 merge 合并。

"""
# 两个人各自创建迁移：
# Alice:  alembic revision --autogenerate -m "添加 age 字段"
# Bob:    alembic revision --autogenerate -m "添加 address 字段"
# 此时两个迁移都基于同一个父版本，形成分支

# 创建合并迁移：
# alembic merge -m "合并 Alice 和 Bob 的迁移" heads

# 合并后的迁移脚本：
revision = 'm1n2o3p4q5r6'
down_revision = ('a1b2c3d4e5f6', 'b2c3d4e5f6a7')  # 多个父版本
branch_labels = None
depends_on = None

def upgrade():
    '''合并迁移不需要做任何操作，只需将两个分支合并'''
    pass

def downgrade():
    pass
"""

# 使用 branch_labels 创建命名分支
"""
# 在 revision 命令中指定分支标签：
# alembic revision --autogenerate -m "特性分支迁移" --branch-label feature_x
# 分支标签在跨分支协作时用于标识不同开发线的迁移
"""


六、在应用启动时自动执行迁移
----------------------------

在生产环境中，常在应用启动时自动执行迁移，确保数据库 Schema 最新。

from alembic.config import Config as AlembicConfig
from alembic import command
import os


def run_database_migrations():
    """在应用启动时自动运行数据库迁移"""
    # 根据环境选择数据库 URL
    db_url = os.getenv("DATABASE_URL", "postgresql://localhost/mydb")

    # 创建 Alembic 配置对象
    alembic_cfg = AlembicConfig("alembic.ini")
    alembic_cfg.set_main_option("sqlalchemy.url", db_url)

    try:
        # 执行迁移到最新版本
        command.upgrade(alembic_cfg, "head")
        print("数据库迁移完成，当前版本: head")
    except Exception as e:
        print(f"数据库迁移失败: {e}")
        # 根据策略决定是否阻止应用启动
        raise


七、离线迁移 SQL 生成
---------------------

离线模式（offline migration）生成 SQL 脚本而不直接执行，适用于：
1. 生产环境数据库无直接写权限
2. 需要 DBA 审查 SQL 后再执行
3. 生成可重复执行的部署脚本

"""
# 生成从当前版本到最新版本的 SQL 脚本：
# alembic upgrade head --sql > migration.sql

# 生成指定版本范围的 SQL：
# alembic upgrade a1b2c3d4e5f6:m1n2o3p4q5r6 --sql > upgrade.sql

# 生成的 SQL 文件内容示例：
# -- Running upgrade a1b2c3d4e5f6 -> m1n2o3p4q5r6
# ALTER TABLE users ADD COLUMN role VARCHAR(20) DEFAULT 'user';
# CREATE INDEX idx_users_role ON users (role);
# UPDATE users SET role = 'admin' WHERE email LIKE '%@admin.com';
"""

# 代码中获取离线 SQL
from alembic.config import Config
from alembic.script import ScriptDirectory
from alembic.runtime.environment import EnvironmentContext
from io import StringIO


def generate_offline_sql(start_rev="base", end_rev="head"):
    """生成离线迁移 SQL 到字符串"""
    config = Config("alembic.ini")
    script = ScriptDirectory.from_config(config)
    output = StringIO()

    def process_revision(direction, revision_context, filename):
        """收集生成的 SQL 到 StringIO"""
        output.write(f"-- {revision_context.get('revision')}\n")
        return []

    with EnvironmentContext(
        config,
        script,
        fn=process_revision,
        start=start_rev,
        end=end_rev,
    ):
        script.run_env()

    return output.getvalue()

dkl.aa5uzL7.cn/55799.Doc
dkl.aa5uzL7.cn/17335.Doc
dkl.aa5uzL7.cn/37917.Doc
dkl.aa5uzL7.cn/57991.Doc
dkl.aa5uzL7.cn/06086.Doc
dkl.aa5uzL7.cn/15373.Doc
dkl.aa5uzL7.cn/99937.Doc
dkl.aa5uzL7.cn/59311.Doc
dkl.aa5uzL7.cn/97775.Doc
dkl.aa5uzL7.cn/91771.Doc
dkk.aa5uzL7.cn/95591.Doc
dkk.aa5uzL7.cn/15179.Doc
dkk.aa5uzL7.cn/95373.Doc
dkk.aa5uzL7.cn/77933.Doc
dkk.aa5uzL7.cn/73753.Doc
dkk.aa5uzL7.cn/15553.Doc
dkk.aa5uzL7.cn/77995.Doc
dkk.aa5uzL7.cn/97139.Doc
dkk.aa5uzL7.cn/17573.Doc
dkk.aa5uzL7.cn/77753.Doc
dkj.aa5uzL7.cn/57777.Doc
dkj.aa5uzL7.cn/73151.Doc
dkj.aa5uzL7.cn/55135.Doc
dkj.aa5uzL7.cn/77379.Doc
dkj.aa5uzL7.cn/51719.Doc
dkj.aa5uzL7.cn/11937.Doc
dkj.aa5uzL7.cn/08800.Doc
dkj.aa5uzL7.cn/53959.Doc
dkj.aa5uzL7.cn/13751.Doc
dkj.aa5uzL7.cn/99311.Doc
dkh.aa5uzL7.cn/17393.Doc
dkh.aa5uzL7.cn/93593.Doc
dkh.aa5uzL7.cn/15953.Doc
dkh.aa5uzL7.cn/13595.Doc
dkh.aa5uzL7.cn/95913.Doc
dkh.aa5uzL7.cn/59913.Doc
dkh.aa5uzL7.cn/75599.Doc
dkh.aa5uzL7.cn/97379.Doc
dkh.aa5uzL7.cn/33977.Doc
dkh.aa5uzL7.cn/97593.Doc
dkg.aa5uzL7.cn/35591.Doc
dkg.aa5uzL7.cn/95713.Doc
dkg.aa5uzL7.cn/95755.Doc
dkg.aa5uzL7.cn/95533.Doc
dkg.aa5uzL7.cn/77773.Doc
dkg.aa5uzL7.cn/19979.Doc
dkg.aa5uzL7.cn/71399.Doc
dkg.aa5uzL7.cn/68848.Doc
dkg.aa5uzL7.cn/77377.Doc
dkg.aa5uzL7.cn/97999.Doc
dkf.aa5uzL7.cn/31793.Doc
dkf.aa5uzL7.cn/11773.Doc
dkf.aa5uzL7.cn/15573.Doc
dkf.aa5uzL7.cn/15199.Doc
dkf.aa5uzL7.cn/15117.Doc
dkf.aa5uzL7.cn/39135.Doc
dkf.aa5uzL7.cn/93775.Doc
dkf.aa5uzL7.cn/15151.Doc
dkf.aa5uzL7.cn/73939.Doc
dkf.aa5uzL7.cn/75755.Doc
dkd.aa5uzL7.cn/37939.Doc
dkd.aa5uzL7.cn/57791.Doc
dkd.aa5uzL7.cn/51593.Doc
dkd.aa5uzL7.cn/35171.Doc
dkd.aa5uzL7.cn/59513.Doc
dkd.aa5uzL7.cn/71551.Doc
dkd.aa5uzL7.cn/55131.Doc
dkd.aa5uzL7.cn/57735.Doc
dkd.aa5uzL7.cn/13339.Doc
dkd.aa5uzL7.cn/55911.Doc
dks.aa5uzL7.cn/77719.Doc
dks.aa5uzL7.cn/93339.Doc
dks.aa5uzL7.cn/53991.Doc
dks.aa5uzL7.cn/55937.Doc
dks.aa5uzL7.cn/93573.Doc
dks.aa5uzL7.cn/71799.Doc
dks.aa5uzL7.cn/13173.Doc
dks.aa5uzL7.cn/99513.Doc
dks.aa5uzL7.cn/44868.Doc
dks.aa5uzL7.cn/91555.Doc
dka.aa5uzL7.cn/99171.Doc
dka.aa5uzL7.cn/31551.Doc
dka.aa5uzL7.cn/53375.Doc
dka.aa5uzL7.cn/97115.Doc
dka.aa5uzL7.cn/97517.Doc
dka.aa5uzL7.cn/35911.Doc
dka.aa5uzL7.cn/39153.Doc
dka.aa5uzL7.cn/95333.Doc
dka.aa5uzL7.cn/75999.Doc
dka.aa5uzL7.cn/37531.Doc
dkp.aa5uzL7.cn/95539.Doc
dkp.aa5uzL7.cn/06404.Doc
dkp.aa5uzL7.cn/55179.Doc
dkp.aa5uzL7.cn/11991.Doc
dkp.aa5uzL7.cn/57711.Doc
dkp.aa5uzL7.cn/57731.Doc
dkp.aa5uzL7.cn/91717.Doc
dkp.aa5uzL7.cn/91379.Doc
dkp.aa5uzL7.cn/77397.Doc
dkp.aa5uzL7.cn/82866.Doc
dko.aa5uzL7.cn/97113.Doc
dko.aa5uzL7.cn/31573.Doc
dko.aa5uzL7.cn/20822.Doc
dko.aa5uzL7.cn/82688.Doc
dko.aa5uzL7.cn/68600.Doc
dko.aa5uzL7.cn/71757.Doc
dko.aa5uzL7.cn/31379.Doc
dko.aa5uzL7.cn/99957.Doc
dko.aa5uzL7.cn/33933.Doc
dko.aa5uzL7.cn/13939.Doc
dki.aa5uzL7.cn/51911.Doc
dki.aa5uzL7.cn/95577.Doc
dki.aa5uzL7.cn/19351.Doc
dki.aa5uzL7.cn/57339.Doc
dki.aa5uzL7.cn/35593.Doc
dki.aa5uzL7.cn/91911.Doc
dki.aa5uzL7.cn/19791.Doc
dki.aa5uzL7.cn/55777.Doc
dki.aa5uzL7.cn/97711.Doc
dki.aa5uzL7.cn/59773.Doc
dku.aa5uzL7.cn/99751.Doc
dku.aa5uzL7.cn/13733.Doc
dku.aa5uzL7.cn/39171.Doc
dku.aa5uzL7.cn/39319.Doc
dku.aa5uzL7.cn/15535.Doc
dku.aa5uzL7.cn/59511.Doc
dku.aa5uzL7.cn/93977.Doc
dku.aa5uzL7.cn/44044.Doc
dku.aa5uzL7.cn/31157.Doc
dku.aa5uzL7.cn/71997.Doc
dky.aa5uzL7.cn/11777.Doc
dky.aa5uzL7.cn/13157.Doc
dky.aa5uzL7.cn/31119.Doc
dky.aa5uzL7.cn/99979.Doc
dky.aa5uzL7.cn/73911.Doc
dky.aa5uzL7.cn/19135.Doc
dky.aa5uzL7.cn/71179.Doc
dky.aa5uzL7.cn/77339.Doc
dky.aa5uzL7.cn/73759.Doc
dky.aa5uzL7.cn/35597.Doc
dkt.aa5uzL7.cn/08462.Doc
dkt.aa5uzL7.cn/93517.Doc
dkt.aa5uzL7.cn/53917.Doc
dkt.aa5uzL7.cn/11175.Doc
dkt.aa5uzL7.cn/95313.Doc
dkt.aa5uzL7.cn/75591.Doc
dkt.aa5uzL7.cn/37379.Doc
dkt.aa5uzL7.cn/33971.Doc
dkt.aa5uzL7.cn/19773.Doc
dkt.aa5uzL7.cn/13971.Doc
dkr.aa5uzL7.cn/59979.Doc
dkr.aa5uzL7.cn/51919.Doc
dkr.aa5uzL7.cn/35331.Doc
dkr.aa5uzL7.cn/93737.Doc
dkr.aa5uzL7.cn/31533.Doc
dkr.aa5uzL7.cn/97579.Doc
dkr.aa5uzL7.cn/33775.Doc
dkr.aa5uzL7.cn/19513.Doc
dkr.aa5uzL7.cn/19711.Doc
dkr.aa5uzL7.cn/91913.Doc
dke.aa5uzL7.cn/19177.Doc
dke.aa5uzL7.cn/93139.Doc
dke.aa5uzL7.cn/51195.Doc
dke.aa5uzL7.cn/39593.Doc
dke.aa5uzL7.cn/71977.Doc
dke.aa5uzL7.cn/73517.Doc
dke.aa5uzL7.cn/39337.Doc
dke.aa5uzL7.cn/99317.Doc
dke.aa5uzL7.cn/33557.Doc
dke.aa5uzL7.cn/79179.Doc
dkw.aa5uzL7.cn/93917.Doc
dkw.aa5uzL7.cn/79793.Doc
dkw.aa5uzL7.cn/79177.Doc
dkw.aa5uzL7.cn/91911.Doc
dkw.aa5uzL7.cn/55317.Doc
dkw.aa5uzL7.cn/71535.Doc
dkw.aa5uzL7.cn/71917.Doc
dkw.aa5uzL7.cn/79737.Doc
dkw.aa5uzL7.cn/35915.Doc
dkw.aa5uzL7.cn/35719.Doc
dkq.aa5uzL7.cn/75931.Doc
dkq.aa5uzL7.cn/37977.Doc
dkq.aa5uzL7.cn/57515.Doc
dkq.aa5uzL7.cn/19337.Doc
dkq.aa5uzL7.cn/79517.Doc
dkq.aa5uzL7.cn/55993.Doc
dkq.aa5uzL7.cn/15915.Doc
dkq.aa5uzL7.cn/73957.Doc
dkq.aa5uzL7.cn/59557.Doc
dkq.aa5uzL7.cn/59953.Doc
djm.aa5uzL7.cn/55953.Doc
djm.aa5uzL7.cn/79939.Doc
djm.aa5uzL7.cn/95137.Doc
djm.aa5uzL7.cn/40200.Doc
djm.aa5uzL7.cn/93997.Doc
djm.aa5uzL7.cn/51517.Doc
djm.aa5uzL7.cn/26842.Doc
djm.aa5uzL7.cn/33539.Doc
djm.aa5uzL7.cn/33591.Doc
djm.aa5uzL7.cn/35775.Doc
djn.aa5uzL7.cn/53575.Doc
djn.aa5uzL7.cn/53997.Doc
djn.aa5uzL7.cn/75151.Doc
djn.aa5uzL7.cn/97599.Doc
djn.aa5uzL7.cn/99113.Doc
djn.aa5uzL7.cn/88868.Doc
djn.aa5uzL7.cn/51917.Doc
djn.aa5uzL7.cn/35775.Doc
djn.aa5uzL7.cn/71915.Doc
djn.aa5uzL7.cn/57531.Doc
djb.aa5uzL7.cn/77731.Doc
djb.aa5uzL7.cn/71315.Doc
djb.aa5uzL7.cn/71955.Doc
djb.aa5uzL7.cn/11511.Doc
djb.aa5uzL7.cn/28004.Doc
djb.aa5uzL7.cn/40840.Doc
djb.aa5uzL7.cn/91751.Doc
djb.aa5uzL7.cn/19551.Doc
djb.aa5uzL7.cn/33193.Doc
djb.aa5uzL7.cn/19997.Doc
djv.aa5uzL7.cn/91133.Doc
djv.aa5uzL7.cn/99599.Doc
djv.aa5uzL7.cn/11597.Doc
djv.aa5uzL7.cn/73135.Doc
djv.aa5uzL7.cn/13135.Doc
djv.aa5uzL7.cn/95915.Doc
djv.aa5uzL7.cn/17373.Doc
djv.aa5uzL7.cn/48202.Doc
djv.aa5uzL7.cn/95311.Doc
djv.aa5uzL7.cn/73775.Doc
djc.aa5uzL7.cn/51131.Doc
djc.aa5uzL7.cn/77955.Doc
djc.aa5uzL7.cn/62002.Doc
djc.aa5uzL7.cn/33173.Doc
djc.aa5uzL7.cn/17719.Doc
djc.aa5uzL7.cn/51335.Doc
djc.aa5uzL7.cn/33197.Doc
djc.aa5uzL7.cn/37377.Doc
djc.aa5uzL7.cn/59713.Doc
djc.aa5uzL7.cn/39151.Doc
djx.aa5uzL7.cn/71919.Doc
djx.aa5uzL7.cn/51733.Doc
djx.aa5uzL7.cn/35511.Doc
djx.aa5uzL7.cn/95591.Doc
djx.aa5uzL7.cn/37115.Doc
djx.aa5uzL7.cn/95177.Doc
djx.aa5uzL7.cn/79151.Doc
djx.aa5uzL7.cn/17339.Doc
djx.aa5uzL7.cn/35397.Doc
djx.aa5uzL7.cn/31935.Doc
djz.aa5uzL7.cn/93979.Doc
djz.aa5uzL7.cn/71137.Doc
djz.aa5uzL7.cn/55979.Doc
djz.aa5uzL7.cn/73953.Doc
djz.aa5uzL7.cn/71793.Doc
djz.aa5uzL7.cn/93993.Doc
djz.aa5uzL7.cn/97795.Doc
djz.aa5uzL7.cn/71379.Doc
djz.aa5uzL7.cn/75979.Doc
djz.aa5uzL7.cn/55997.Doc
djl.aa5uzL7.cn/31975.Doc
djl.aa5uzL7.cn/73315.Doc
djl.aa5uzL7.cn/95119.Doc
djl.aa5uzL7.cn/99351.Doc
djl.aa5uzL7.cn/33779.Doc
djl.aa5uzL7.cn/11371.Doc
djl.aa5uzL7.cn/13117.Doc
djl.aa5uzL7.cn/35359.Doc
djl.aa5uzL7.cn/13175.Doc
djl.aa5uzL7.cn/51315.Doc
djk.aa5uzL7.cn/99115.Doc
djk.aa5uzL7.cn/31971.Doc
djk.aa5uzL7.cn/97733.Doc
djk.aa5uzL7.cn/91339.Doc
djk.aa5uzL7.cn/53993.Doc
djk.aa5uzL7.cn/19551.Doc
djk.aa5uzL7.cn/57175.Doc
djk.aa5uzL7.cn/19377.Doc
djk.aa5uzL7.cn/55973.Doc
djk.aa5uzL7.cn/53599.Doc
djj.aa5uzL7.cn/95353.Doc
djj.aa5uzL7.cn/55919.Doc
djj.aa5uzL7.cn/35517.Doc
djj.aa5uzL7.cn/33179.Doc
djj.aa5uzL7.cn/39935.Doc
djj.aa5uzL7.cn/53911.Doc
djj.aa5uzL7.cn/97335.Doc
djj.aa5uzL7.cn/71959.Doc
djj.aa5uzL7.cn/15793.Doc
djj.aa5uzL7.cn/73131.Doc
djh.aa5uzL7.cn/68404.Doc
djh.aa5uzL7.cn/15711.Doc
djh.aa5uzL7.cn/55137.Doc
djh.aa5uzL7.cn/17731.Doc
djh.aa5uzL7.cn/93711.Doc
djh.aa5uzL7.cn/55915.Doc
djh.aa5uzL7.cn/93115.Doc
djh.aa5uzL7.cn/73315.Doc
djh.aa5uzL7.cn/53137.Doc
djh.aa5uzL7.cn/91551.Doc
djg.aa5uzL7.cn/99555.Doc
djg.aa5uzL7.cn/99995.Doc
djg.aa5uzL7.cn/33199.Doc
djg.aa5uzL7.cn/13117.Doc
djg.aa5uzL7.cn/59999.Doc
djg.aa5uzL7.cn/31593.Doc
djg.aa5uzL7.cn/53713.Doc
djg.aa5uzL7.cn/33131.Doc
djg.aa5uzL7.cn/31957.Doc
djg.aa5uzL7.cn/37551.Doc
djf.aa5uzL7.cn/57195.Doc
djf.aa5uzL7.cn/15117.Doc
djf.aa5uzL7.cn/15319.Doc
djf.aa5uzL7.cn/31713.Doc
djf.aa5uzL7.cn/97571.Doc
djf.aa5uzL7.cn/77591.Doc
djf.aa5uzL7.cn/97739.Doc
djf.aa5uzL7.cn/00200.Doc
djf.aa5uzL7.cn/13117.Doc
djf.aa5uzL7.cn/51533.Doc
djd.aa5uzL7.cn/73191.Doc
djd.aa5uzL7.cn/39795.Doc
djd.aa5uzL7.cn/97719.Doc
djd.aa5uzL7.cn/31553.Doc
djd.aa5uzL7.cn/99717.Doc
djd.aa5uzL7.cn/59771.Doc
djd.aa5uzL7.cn/15117.Doc
djd.aa5uzL7.cn/95193.Doc
djd.aa5uzL7.cn/59119.Doc
djd.aa5uzL7.cn/26200.Doc
djs.aa5uzL7.cn/99577.Doc
djs.aa5uzL7.cn/93333.Doc
djs.aa5uzL7.cn/24426.Doc
djs.aa5uzL7.cn/95793.Doc
djs.aa5uzL7.cn/77159.Doc
djs.aa5uzL7.cn/13753.Doc
djs.aa5uzL7.cn/77179.Doc
djs.aa5uzL7.cn/51113.Doc
djs.aa5uzL7.cn/51573.Doc
djs.aa5uzL7.cn/53173.Doc
dja.aa5uzL7.cn/13777.Doc
dja.aa5uzL7.cn/97113.Doc
dja.aa5uzL7.cn/95995.Doc
dja.aa5uzL7.cn/79193.Doc
dja.aa5uzL7.cn/39137.Doc
dja.aa5uzL7.cn/35755.Doc
dja.aa5uzL7.cn/51977.Doc
dja.aa5uzL7.cn/66066.Doc
dja.aa5uzL7.cn/35773.Doc
dja.aa5uzL7.cn/97757.Doc
djp.aa5uzL7.cn/39797.Doc
djp.aa5uzL7.cn/93175.Doc
djp.aa5uzL7.cn/79531.Doc
djp.aa5uzL7.cn/73511.Doc
djp.aa5uzL7.cn/55153.Doc
djp.aa5uzL7.cn/59171.Doc
djp.aa5uzL7.cn/55933.Doc
djp.aa5uzL7.cn/93193.Doc
djp.aa5uzL7.cn/19135.Doc
djp.aa5uzL7.cn/19137.Doc
djo.aa5uzL7.cn/37537.Doc
djo.aa5uzL7.cn/95597.Doc
djo.aa5uzL7.cn/93533.Doc
djo.aa5uzL7.cn/71915.Doc
djo.aa5uzL7.cn/37375.Doc
djo.aa5uzL7.cn/31533.Doc
djo.aa5uzL7.cn/37799.Doc
djo.aa5uzL7.cn/91175.Doc
djo.aa5uzL7.cn/91315.Doc
djo.aa5uzL7.cn/37971.Doc
dji.aa5uzL7.cn/77173.Doc
dji.aa5uzL7.cn/91517.Doc
dji.aa5uzL7.cn/97511.Doc
dji.aa5uzL7.cn/79153.Doc
dji.aa5uzL7.cn/75733.Doc
dji.aa5uzL7.cn/99535.Doc
dji.aa5uzL7.cn/51337.Doc
dji.aa5uzL7.cn/99993.Doc
dji.aa5uzL7.cn/59959.Doc
dji.aa5uzL7.cn/77335.Doc
dju.aa5uzL7.cn/13151.Doc
dju.aa5uzL7.cn/77397.Doc
dju.aa5uzL7.cn/15197.Doc
dju.aa5uzL7.cn/53535.Doc
dju.aa5uzL7.cn/57919.Doc
dju.aa5uzL7.cn/73557.Doc
dju.aa5uzL7.cn/73777.Doc
dju.aa5uzL7.cn/11171.Doc
dju.aa5uzL7.cn/79311.Doc
dju.aa5uzL7.cn/86468.Doc
djy.aa5uzL7.cn/75953.Doc
djy.aa5uzL7.cn/57397.Doc
djy.aa5uzL7.cn/71313.Doc
djy.aa5uzL7.cn/71935.Doc
djy.aa5uzL7.cn/53717.Doc
djy.aa5uzL7.cn/35755.Doc
djy.aa5uzL7.cn/93755.Doc
djy.aa5uzL7.cn/55153.Doc
djy.aa5uzL7.cn/77311.Doc
djy.aa5uzL7.cn/93937.Doc
djt.aa5uzL7.cn/55355.Doc
djt.aa5uzL7.cn/97753.Doc
djt.aa5uzL7.cn/93577.Doc
djt.aa5uzL7.cn/75193.Doc
djt.aa5uzL7.cn/71373.Doc
djt.aa5uzL7.cn/55139.Doc
djt.aa5uzL7.cn/19793.Doc
djt.aa5uzL7.cn/53355.Doc
djt.aa5uzL7.cn/51971.Doc
djt.aa5uzL7.cn/91733.Doc
djr.aa5uzL7.cn/17515.Doc
djr.aa5uzL7.cn/13917.Doc
djr.aa5uzL7.cn/71973.Doc
djr.aa5uzL7.cn/31119.Doc
djr.aa5uzL7.cn/97519.Doc
djr.aa5uzL7.cn/75911.Doc
djr.aa5uzL7.cn/39751.Doc
djr.aa5uzL7.cn/77575.Doc
djr.aa5uzL7.cn/75131.Doc
djr.aa5uzL7.cn/37351.Doc
dje.aa5uzL7.cn/19593.Doc
dje.aa5uzL7.cn/31971.Doc
dje.aa5uzL7.cn/53919.Doc
dje.aa5uzL7.cn/73751.Doc
dje.aa5uzL7.cn/55773.Doc
dje.aa5uzL7.cn/57597.Doc
dje.aa5uzL7.cn/55939.Doc
dje.aa5uzL7.cn/97979.Doc
dje.aa5uzL7.cn/37977.Doc
dje.aa5uzL7.cn/15917.Doc
djw.aa5uzL7.cn/15199.Doc
djw.aa5uzL7.cn/11511.Doc
djw.aa5uzL7.cn/28402.Doc
djw.aa5uzL7.cn/73319.Doc
djw.aa5uzL7.cn/51579.Doc
djw.aa5uzL7.cn/91795.Doc
djw.aa5uzL7.cn/33799.Doc
djw.aa5uzL7.cn/95355.Doc
djw.aa5uzL7.cn/88406.Doc
djw.aa5uzL7.cn/55533.Doc
djq.aa5uzL7.cn/19333.Doc
djq.aa5uzL7.cn/59371.Doc
djq.aa5uzL7.cn/95575.Doc
djq.aa5uzL7.cn/31533.Doc
djq.aa5uzL7.cn/35933.Doc
djq.aa5uzL7.cn/53197.Doc
djq.aa5uzL7.cn/91117.Doc
djq.aa5uzL7.cn/11517.Doc
djq.aa5uzL7.cn/57773.Doc
djq.aa5uzL7.cn/35117.Doc
dhm.aa5uzL7.cn/19153.Doc
dhm.aa5uzL7.cn/95775.Doc
dhm.aa5uzL7.cn/79793.Doc
dhm.aa5uzL7.cn/79991.Doc
dhm.aa5uzL7.cn/93575.Doc
dhm.aa5uzL7.cn/11919.Doc
dhm.aa5uzL7.cn/51557.Doc
dhm.aa5uzL7.cn/17139.Doc
dhm.aa5uzL7.cn/51171.Doc
dhm.aa5uzL7.cn/13593.Doc
dhn.aa5uzL7.cn/73371.Doc
dhn.aa5uzL7.cn/57199.Doc
dhn.aa5uzL7.cn/95591.Doc
dhn.aa5uzL7.cn/33375.Doc
dhn.aa5uzL7.cn/59777.Doc
dhn.aa5uzL7.cn/51713.Doc
dhn.aa5uzL7.cn/51359.Doc
dhn.aa5uzL7.cn/75735.Doc
dhn.aa5uzL7.cn/59373.Doc
dhn.aa5uzL7.cn/57557.Doc
dhb.aa5uzL7.cn/55991.Doc
dhb.aa5uzL7.cn/99999.Doc
dhb.aa5uzL7.cn/33517.Doc
dhb.aa5uzL7.cn/13577.Doc
dhb.aa5uzL7.cn/19113.Doc
dhb.aa5uzL7.cn/17771.Doc
dhb.aa5uzL7.cn/39739.Doc
dhb.aa5uzL7.cn/31177.Doc
dhb.aa5uzL7.cn/51113.Doc
dhb.aa5uzL7.cn/11191.Doc
dhv.aa5uzL7.cn/31577.Doc
dhv.aa5uzL7.cn/17593.Doc
dhv.aa5uzL7.cn/75957.Doc
dhv.aa5uzL7.cn/77777.Doc
dhv.aa5uzL7.cn/11977.Doc
dhv.aa5uzL7.cn/99115.Doc
dhv.aa5uzL7.cn/11975.Doc
dhv.aa5uzL7.cn/33333.Doc
dhv.aa5uzL7.cn/35597.Doc
dhv.aa5uzL7.cn/35335.Doc
