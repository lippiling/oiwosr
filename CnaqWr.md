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

dmu.0wp4aoe.cn/99133.Doc
dmu.0wp4aoe.cn/75713.Doc
dmu.0wp4aoe.cn/99955.Doc
dmu.0wp4aoe.cn/33339.Doc
dmu.0wp4aoe.cn/00482.Doc
dmu.0wp4aoe.cn/33575.Doc
dmu.0wp4aoe.cn/77519.Doc
dmu.0wp4aoe.cn/19551.Doc
dmu.0wp4aoe.cn/73951.Doc
dmu.0wp4aoe.cn/51513.Doc
dmy.0wp4aoe.cn/71191.Doc
dmy.0wp4aoe.cn/59317.Doc
dmy.0wp4aoe.cn/53959.Doc
dmy.0wp4aoe.cn/37515.Doc
dmy.0wp4aoe.cn/75517.Doc
dmy.0wp4aoe.cn/75315.Doc
dmy.0wp4aoe.cn/37559.Doc
dmy.0wp4aoe.cn/95319.Doc
dmy.0wp4aoe.cn/53913.Doc
dmy.0wp4aoe.cn/97119.Doc
dmt.0wp4aoe.cn/40208.Doc
dmt.0wp4aoe.cn/02442.Doc
dmt.0wp4aoe.cn/91571.Doc
dmt.0wp4aoe.cn/59751.Doc
dmt.0wp4aoe.cn/97751.Doc
dmt.0wp4aoe.cn/37739.Doc
dmt.0wp4aoe.cn/79957.Doc
dmt.0wp4aoe.cn/59915.Doc
dmt.0wp4aoe.cn/95515.Doc
dmt.0wp4aoe.cn/99933.Doc
dmr.0wp4aoe.cn/33353.Doc
dmr.0wp4aoe.cn/33711.Doc
dmr.0wp4aoe.cn/77155.Doc
dmr.0wp4aoe.cn/59379.Doc
dmr.0wp4aoe.cn/06864.Doc
dmr.0wp4aoe.cn/93117.Doc
dmr.0wp4aoe.cn/66244.Doc
dmr.0wp4aoe.cn/15173.Doc
dmr.0wp4aoe.cn/99377.Doc
dmr.0wp4aoe.cn/37591.Doc
dme.0wp4aoe.cn/11139.Doc
dme.0wp4aoe.cn/91395.Doc
dme.0wp4aoe.cn/95973.Doc
dme.0wp4aoe.cn/11517.Doc
dme.0wp4aoe.cn/37757.Doc
dme.0wp4aoe.cn/15739.Doc
dme.0wp4aoe.cn/39339.Doc
dme.0wp4aoe.cn/19119.Doc
dme.0wp4aoe.cn/93711.Doc
dme.0wp4aoe.cn/75113.Doc
dmw.0wp4aoe.cn/95175.Doc
dmw.0wp4aoe.cn/71151.Doc
dmw.0wp4aoe.cn/17595.Doc
dmw.0wp4aoe.cn/71977.Doc
dmw.0wp4aoe.cn/79917.Doc
dmw.0wp4aoe.cn/93591.Doc
dmw.0wp4aoe.cn/55751.Doc
dmw.0wp4aoe.cn/31739.Doc
dmw.0wp4aoe.cn/11595.Doc
dmw.0wp4aoe.cn/35593.Doc
dmq.0wp4aoe.cn/71359.Doc
dmq.0wp4aoe.cn/39553.Doc
dmq.0wp4aoe.cn/35919.Doc
dmq.0wp4aoe.cn/91733.Doc
dmq.0wp4aoe.cn/37539.Doc
dmq.0wp4aoe.cn/57751.Doc
dmq.0wp4aoe.cn/55159.Doc
dmq.0wp4aoe.cn/71119.Doc
dmq.0wp4aoe.cn/35333.Doc
dmq.0wp4aoe.cn/79115.Doc
dnm.0wp4aoe.cn/33575.Doc
dnm.0wp4aoe.cn/97515.Doc
dnm.0wp4aoe.cn/31793.Doc
dnm.0wp4aoe.cn/59911.Doc
dnm.0wp4aoe.cn/77739.Doc
dnm.0wp4aoe.cn/91379.Doc
dnm.0wp4aoe.cn/39797.Doc
dnm.0wp4aoe.cn/31711.Doc
dnm.0wp4aoe.cn/35599.Doc
dnm.0wp4aoe.cn/37977.Doc
dnn.0wp4aoe.cn/17731.Doc
dnn.0wp4aoe.cn/39597.Doc
dnn.0wp4aoe.cn/59171.Doc
dnn.0wp4aoe.cn/13915.Doc
dnn.0wp4aoe.cn/73179.Doc
dnn.0wp4aoe.cn/3.Doc
dnn.0wp4aoe.cn/77353.Doc
dnn.0wp4aoe.cn/53557.Doc
dnn.0wp4aoe.cn/73751.Doc
dnn.0wp4aoe.cn/77171.Doc
dnb.0wp4aoe.cn/33535.Doc
dnb.0wp4aoe.cn/75911.Doc
dnb.0wp4aoe.cn/57771.Doc
dnb.0wp4aoe.cn/79375.Doc
dnb.0wp4aoe.cn/73995.Doc
dnb.0wp4aoe.cn/75593.Doc
dnb.0wp4aoe.cn/93353.Doc
dnb.0wp4aoe.cn/31599.Doc
dnb.0wp4aoe.cn/53771.Doc
dnb.0wp4aoe.cn/57177.Doc
dnv.0wp4aoe.cn/31357.Doc
dnv.0wp4aoe.cn/28484.Doc
dnv.0wp4aoe.cn/35191.Doc
dnv.0wp4aoe.cn/13733.Doc
dnv.0wp4aoe.cn/31137.Doc
dnv.0wp4aoe.cn/91113.Doc
dnv.0wp4aoe.cn/33591.Doc
dnv.0wp4aoe.cn/22020.Doc
dnv.0wp4aoe.cn/99191.Doc
dnv.0wp4aoe.cn/57735.Doc
dnc.0wp4aoe.cn/95915.Doc
dnc.0wp4aoe.cn/37333.Doc
dnc.0wp4aoe.cn/51975.Doc
dnc.0wp4aoe.cn/33159.Doc
dnc.0wp4aoe.cn/91759.Doc
dnc.0wp4aoe.cn/57159.Doc
dnc.0wp4aoe.cn/11933.Doc
dnc.0wp4aoe.cn/17555.Doc
dnc.0wp4aoe.cn/11999.Doc
dnc.0wp4aoe.cn/11973.Doc
dnx.0wp4aoe.cn/77339.Doc
dnx.0wp4aoe.cn/59797.Doc
dnx.0wp4aoe.cn/15573.Doc
dnx.0wp4aoe.cn/35935.Doc
dnx.0wp4aoe.cn/93919.Doc
dnx.0wp4aoe.cn/95139.Doc
dnx.0wp4aoe.cn/99173.Doc
dnx.0wp4aoe.cn/59715.Doc
dnx.0wp4aoe.cn/97371.Doc
dnx.0wp4aoe.cn/39171.Doc
dnz.0wp4aoe.cn/99755.Doc
dnz.0wp4aoe.cn/60826.Doc
dnz.0wp4aoe.cn/17571.Doc
dnz.0wp4aoe.cn/93179.Doc
dnz.0wp4aoe.cn/15795.Doc
dnz.0wp4aoe.cn/59319.Doc
dnz.0wp4aoe.cn/91931.Doc
dnz.0wp4aoe.cn/35551.Doc
dnz.0wp4aoe.cn/19913.Doc
dnz.0wp4aoe.cn/79175.Doc
dnl.0wp4aoe.cn/97375.Doc
dnl.0wp4aoe.cn/31797.Doc
dnl.0wp4aoe.cn/39515.Doc
dnl.0wp4aoe.cn/55173.Doc
dnl.0wp4aoe.cn/17779.Doc
dnl.0wp4aoe.cn/19939.Doc
dnl.0wp4aoe.cn/35955.Doc
dnl.0wp4aoe.cn/51315.Doc
dnl.0wp4aoe.cn/53157.Doc
dnl.0wp4aoe.cn/11775.Doc
dnk.0wp4aoe.cn/97377.Doc
dnk.0wp4aoe.cn/37757.Doc
dnk.0wp4aoe.cn/93159.Doc
dnk.0wp4aoe.cn/55739.Doc
dnk.0wp4aoe.cn/79795.Doc
dnk.0wp4aoe.cn/11937.Doc
dnk.0wp4aoe.cn/37519.Doc
dnk.0wp4aoe.cn/13555.Doc
dnk.0wp4aoe.cn/37997.Doc
dnk.0wp4aoe.cn/71539.Doc
dnj.0wp4aoe.cn/35173.Doc
dnj.0wp4aoe.cn/19379.Doc
dnj.0wp4aoe.cn/37771.Doc
dnj.0wp4aoe.cn/77511.Doc
dnj.0wp4aoe.cn/53911.Doc
dnj.0wp4aoe.cn/13997.Doc
dnj.0wp4aoe.cn/91591.Doc
dnj.0wp4aoe.cn/64224.Doc
dnj.0wp4aoe.cn/37179.Doc
dnj.0wp4aoe.cn/99777.Doc
dnh.0wp4aoe.cn/73333.Doc
dnh.0wp4aoe.cn/55919.Doc
dnh.0wp4aoe.cn/39153.Doc
dnh.0wp4aoe.cn/37357.Doc
dnh.0wp4aoe.cn/17379.Doc
dnh.0wp4aoe.cn/91517.Doc
dnh.0wp4aoe.cn/88046.Doc
dnh.0wp4aoe.cn/91555.Doc
dnh.0wp4aoe.cn/71573.Doc
dnh.0wp4aoe.cn/31399.Doc
dng.0wp4aoe.cn/57373.Doc
dng.0wp4aoe.cn/35577.Doc
dng.0wp4aoe.cn/35771.Doc
dng.0wp4aoe.cn/53359.Doc
dng.0wp4aoe.cn/33139.Doc
dng.0wp4aoe.cn/51313.Doc
dng.0wp4aoe.cn/31797.Doc
dng.0wp4aoe.cn/13733.Doc
dng.0wp4aoe.cn/71931.Doc
dng.0wp4aoe.cn/93553.Doc
dnf.0wp4aoe.cn/62888.Doc
dnf.0wp4aoe.cn/13553.Doc
dnf.0wp4aoe.cn/91313.Doc
dnf.0wp4aoe.cn/33159.Doc
dnf.0wp4aoe.cn/64066.Doc
dnf.0wp4aoe.cn/35977.Doc
dnf.0wp4aoe.cn/55557.Doc
dnf.0wp4aoe.cn/99191.Doc
dnf.0wp4aoe.cn/88208.Doc
dnf.0wp4aoe.cn/46628.Doc
dnd.0wp4aoe.cn/33931.Doc
dnd.0wp4aoe.cn/35579.Doc
dnd.0wp4aoe.cn/55375.Doc
dnd.0wp4aoe.cn/95351.Doc
dnd.0wp4aoe.cn/97137.Doc
dnd.0wp4aoe.cn/35937.Doc
dnd.0wp4aoe.cn/79735.Doc
dnd.0wp4aoe.cn/17751.Doc
dnd.0wp4aoe.cn/15113.Doc
dnd.0wp4aoe.cn/53519.Doc
dns.0wp4aoe.cn/91539.Doc
dns.0wp4aoe.cn/71313.Doc
dns.0wp4aoe.cn/11733.Doc
dns.0wp4aoe.cn/33771.Doc
dns.0wp4aoe.cn/73975.Doc
dns.0wp4aoe.cn/79951.Doc
dns.0wp4aoe.cn/57713.Doc
dns.0wp4aoe.cn/73917.Doc
dns.0wp4aoe.cn/17315.Doc
dns.0wp4aoe.cn/91197.Doc
dna.0wp4aoe.cn/33999.Doc
dna.0wp4aoe.cn/37319.Doc
dna.0wp4aoe.cn/17311.Doc
dna.0wp4aoe.cn/37391.Doc
dna.0wp4aoe.cn/31199.Doc
dna.0wp4aoe.cn/55971.Doc
dna.0wp4aoe.cn/35353.Doc
dna.0wp4aoe.cn/97571.Doc
dna.0wp4aoe.cn/37131.Doc
dna.0wp4aoe.cn/71391.Doc
dnp.0wp4aoe.cn/19179.Doc
dnp.0wp4aoe.cn/15355.Doc
dnp.0wp4aoe.cn/77195.Doc
dnp.0wp4aoe.cn/77559.Doc
dnp.0wp4aoe.cn/19917.Doc
dnp.0wp4aoe.cn/59979.Doc
dnp.0wp4aoe.cn/79353.Doc
dnp.0wp4aoe.cn/37331.Doc
dnp.0wp4aoe.cn/77791.Doc
dnp.0wp4aoe.cn/77755.Doc
dno.0wp4aoe.cn/77139.Doc
dno.0wp4aoe.cn/55715.Doc
dno.0wp4aoe.cn/99399.Doc
dno.0wp4aoe.cn/51951.Doc
dno.0wp4aoe.cn/33131.Doc
dno.0wp4aoe.cn/31757.Doc
dno.0wp4aoe.cn/71399.Doc
dno.0wp4aoe.cn/13753.Doc
dno.0wp4aoe.cn/77997.Doc
dno.0wp4aoe.cn/31573.Doc
dni.0wp4aoe.cn/51779.Doc
dni.0wp4aoe.cn/11359.Doc
dni.0wp4aoe.cn/15797.Doc
dni.0wp4aoe.cn/31335.Doc
dni.0wp4aoe.cn/75595.Doc
dni.0wp4aoe.cn/95753.Doc
dni.0wp4aoe.cn/71371.Doc
dni.0wp4aoe.cn/95931.Doc
dni.0wp4aoe.cn/51315.Doc
dni.0wp4aoe.cn/15177.Doc
dnu.0wp4aoe.cn/35357.Doc
dnu.0wp4aoe.cn/37953.Doc
dnu.0wp4aoe.cn/77737.Doc
dnu.0wp4aoe.cn/99173.Doc
dnu.0wp4aoe.cn/53531.Doc
dnu.0wp4aoe.cn/17519.Doc
dnu.0wp4aoe.cn/55331.Doc
dnu.0wp4aoe.cn/35711.Doc
dnu.0wp4aoe.cn/93139.Doc
dnu.0wp4aoe.cn/95511.Doc
dny.0wp4aoe.cn/97939.Doc
dny.0wp4aoe.cn/99791.Doc
dny.0wp4aoe.cn/11771.Doc
dny.0wp4aoe.cn/53171.Doc
dny.0wp4aoe.cn/19977.Doc
dny.0wp4aoe.cn/51935.Doc
dny.0wp4aoe.cn/91913.Doc
dny.0wp4aoe.cn/55735.Doc
dny.0wp4aoe.cn/53379.Doc
dny.0wp4aoe.cn/39591.Doc
dnt.0wp4aoe.cn/48202.Doc
dnt.0wp4aoe.cn/77755.Doc
dnt.0wp4aoe.cn/13793.Doc
dnt.0wp4aoe.cn/33313.Doc
dnt.0wp4aoe.cn/57593.Doc
dnt.0wp4aoe.cn/33755.Doc
dnt.0wp4aoe.cn/93553.Doc
dnt.0wp4aoe.cn/53937.Doc
dnt.0wp4aoe.cn/59995.Doc
dnt.0wp4aoe.cn/44840.Doc
dnr.0wp4aoe.cn/37757.Doc
dnr.0wp4aoe.cn/93959.Doc
dnr.0wp4aoe.cn/77775.Doc
dnr.0wp4aoe.cn/11311.Doc
dnr.0wp4aoe.cn/93955.Doc
dnr.0wp4aoe.cn/19751.Doc
dnr.0wp4aoe.cn/77959.Doc
dnr.0wp4aoe.cn/33997.Doc
dnr.0wp4aoe.cn/11771.Doc
dnr.0wp4aoe.cn/95357.Doc
dne.0wp4aoe.cn/75731.Doc
dne.0wp4aoe.cn/46882.Doc
dne.0wp4aoe.cn/35959.Doc
dne.0wp4aoe.cn/79973.Doc
dne.0wp4aoe.cn/79937.Doc
dne.0wp4aoe.cn/17917.Doc
dne.0wp4aoe.cn/71155.Doc
dne.0wp4aoe.cn/19775.Doc
dne.0wp4aoe.cn/31179.Doc
dne.0wp4aoe.cn/93593.Doc
dnw.0wp4aoe.cn/73377.Doc
dnw.0wp4aoe.cn/77739.Doc
dnw.0wp4aoe.cn/97975.Doc
dnw.0wp4aoe.cn/39119.Doc
dnw.0wp4aoe.cn/51597.Doc
dnw.0wp4aoe.cn/17351.Doc
dnw.0wp4aoe.cn/51753.Doc
dnw.0wp4aoe.cn/59593.Doc
dnw.0wp4aoe.cn/37733.Doc
dnw.0wp4aoe.cn/31391.Doc
dnq.0wp4aoe.cn/53379.Doc
dnq.0wp4aoe.cn/73991.Doc
dnq.0wp4aoe.cn/79713.Doc
dnq.0wp4aoe.cn/35977.Doc
dnq.0wp4aoe.cn/95555.Doc
dnq.0wp4aoe.cn/17993.Doc
dnq.0wp4aoe.cn/17117.Doc
dnq.0wp4aoe.cn/15715.Doc
dnq.0wp4aoe.cn/51133.Doc
dnq.0wp4aoe.cn/33119.Doc
dbm.0wp4aoe.cn/13173.Doc
dbm.0wp4aoe.cn/60664.Doc
dbm.0wp4aoe.cn/99199.Doc
dbm.0wp4aoe.cn/55939.Doc
dbm.0wp4aoe.cn/17933.Doc
dbm.0wp4aoe.cn/11317.Doc
dbm.0wp4aoe.cn/91579.Doc
dbm.0wp4aoe.cn/97713.Doc
dbm.0wp4aoe.cn/99791.Doc
dbm.0wp4aoe.cn/15155.Doc
dbn.0wp4aoe.cn/59713.Doc
dbn.0wp4aoe.cn/59577.Doc
dbn.0wp4aoe.cn/91553.Doc
dbn.0wp4aoe.cn/75311.Doc
dbn.0wp4aoe.cn/71513.Doc
dbn.0wp4aoe.cn/91371.Doc
dbn.0wp4aoe.cn/59775.Doc
dbn.0wp4aoe.cn/48208.Doc
dbn.0wp4aoe.cn/73331.Doc
dbn.0wp4aoe.cn/91797.Doc
dbb.0wp4aoe.cn/97319.Doc
dbb.0wp4aoe.cn/99771.Doc
dbb.0wp4aoe.cn/55337.Doc
dbb.0wp4aoe.cn/31955.Doc
dbb.0wp4aoe.cn/17339.Doc
dbb.0wp4aoe.cn/13973.Doc
dbb.0wp4aoe.cn/99977.Doc
dbb.0wp4aoe.cn/93759.Doc
dbb.0wp4aoe.cn/33555.Doc
dbb.0wp4aoe.cn/55397.Doc
dbv.0wp4aoe.cn/97737.Doc
dbv.0wp4aoe.cn/71337.Doc
dbv.0wp4aoe.cn/93959.Doc
dbv.0wp4aoe.cn/93571.Doc
dbv.0wp4aoe.cn/35379.Doc
dbv.0wp4aoe.cn/35197.Doc
dbv.0wp4aoe.cn/75533.Doc
dbv.0wp4aoe.cn/15353.Doc
dbv.0wp4aoe.cn/57937.Doc
dbv.0wp4aoe.cn/06806.Doc
dbc.0wp4aoe.cn/99511.Doc
dbc.0wp4aoe.cn/19131.Doc
dbc.0wp4aoe.cn/71971.Doc
dbc.0wp4aoe.cn/15757.Doc
dbc.0wp4aoe.cn/71711.Doc
dbc.0wp4aoe.cn/75173.Doc
dbc.0wp4aoe.cn/88224.Doc
dbc.0wp4aoe.cn/11999.Doc
dbc.0wp4aoe.cn/31197.Doc
dbc.0wp4aoe.cn/31157.Doc
dbx.0wp4aoe.cn/95111.Doc
dbx.0wp4aoe.cn/51139.Doc
dbx.0wp4aoe.cn/19179.Doc
dbx.0wp4aoe.cn/11791.Doc
dbx.0wp4aoe.cn/73573.Doc
dbx.0wp4aoe.cn/91393.Doc
dbx.0wp4aoe.cn/59713.Doc
dbx.0wp4aoe.cn/44608.Doc
dbx.0wp4aoe.cn/40288.Doc
dbx.0wp4aoe.cn/55995.Doc
dbz.0wp4aoe.cn/97175.Doc
dbz.0wp4aoe.cn/75115.Doc
dbz.0wp4aoe.cn/57919.Doc
dbz.0wp4aoe.cn/93573.Doc
dbz.0wp4aoe.cn/57551.Doc
dbz.0wp4aoe.cn/97335.Doc
dbz.0wp4aoe.cn/13791.Doc
dbz.0wp4aoe.cn/75731.Doc
dbz.0wp4aoe.cn/19779.Doc
dbz.0wp4aoe.cn/91793.Doc
dbl.0wp4aoe.cn/95735.Doc
dbl.0wp4aoe.cn/93379.Doc
dbl.0wp4aoe.cn/33735.Doc
dbl.0wp4aoe.cn/99559.Doc
dbl.0wp4aoe.cn/75719.Doc
dbl.0wp4aoe.cn/97715.Doc
dbl.0wp4aoe.cn/55179.Doc
dbl.0wp4aoe.cn/75355.Doc
dbl.0wp4aoe.cn/19731.Doc
dbl.0wp4aoe.cn/99719.Doc
dbk.0wp4aoe.cn/13197.Doc
dbk.0wp4aoe.cn/97719.Doc
dbk.0wp4aoe.cn/37937.Doc
dbk.0wp4aoe.cn/79955.Doc
dbk.0wp4aoe.cn/68806.Doc
dbk.0wp4aoe.cn/17997.Doc
dbk.0wp4aoe.cn/91915.Doc
dbk.0wp4aoe.cn/35339.Doc
dbk.0wp4aoe.cn/79375.Doc
dbk.0wp4aoe.cn/33759.Doc
dbj.0wp4aoe.cn/71175.Doc
dbj.0wp4aoe.cn/39799.Doc
dbj.0wp4aoe.cn/33999.Doc
dbj.0wp4aoe.cn/99315.Doc
dbj.0wp4aoe.cn/35731.Doc
dbj.0wp4aoe.cn/13551.Doc
dbj.0wp4aoe.cn/31737.Doc
dbj.0wp4aoe.cn/79395.Doc
dbj.0wp4aoe.cn/57339.Doc
dbj.0wp4aoe.cn/77377.Doc
dbh.0wp4aoe.cn/11333.Doc
dbh.0wp4aoe.cn/37713.Doc
dbh.0wp4aoe.cn/15333.Doc
dbh.0wp4aoe.cn/71335.Doc
dbh.0wp4aoe.cn/59517.Doc
dbh.0wp4aoe.cn/15139.Doc
dbh.0wp4aoe.cn/11759.Doc
dbh.0wp4aoe.cn/55355.Doc
dbh.0wp4aoe.cn/77393.Doc
dbh.0wp4aoe.cn/33713.Doc
dbg.0wp4aoe.cn/35151.Doc
dbg.0wp4aoe.cn/37197.Doc
dbg.0wp4aoe.cn/17591.Doc
dbg.0wp4aoe.cn/55137.Doc
dbg.0wp4aoe.cn/95731.Doc
dbg.0wp4aoe.cn/95391.Doc
dbg.0wp4aoe.cn/91557.Doc
dbg.0wp4aoe.cn/77159.Doc
dbg.0wp4aoe.cn/51579.Doc
dbg.0wp4aoe.cn/33759.Doc
dbf.0wp4aoe.cn/17775.Doc
dbf.0wp4aoe.cn/97555.Doc
dbf.0wp4aoe.cn/17737.Doc
dbf.0wp4aoe.cn/11591.Doc
dbf.0wp4aoe.cn/57197.Doc
dbf.0wp4aoe.cn/53593.Doc
dbf.0wp4aoe.cn/75735.Doc
dbf.0wp4aoe.cn/71333.Doc
dbf.0wp4aoe.cn/71311.Doc
dbf.0wp4aoe.cn/11935.Doc
dbd.0wp4aoe.cn/22040.Doc
dbd.0wp4aoe.cn/75791.Doc
dbd.0wp4aoe.cn/77719.Doc
dbd.0wp4aoe.cn/51797.Doc
dbd.0wp4aoe.cn/51157.Doc
dbd.0wp4aoe.cn/91113.Doc
dbd.0wp4aoe.cn/57533.Doc
dbd.0wp4aoe.cn/99919.Doc
dbd.0wp4aoe.cn/15177.Doc
dbd.0wp4aoe.cn/31959.Doc
dbs.0wp4aoe.cn/57555.Doc
dbs.0wp4aoe.cn/39751.Doc
dbs.0wp4aoe.cn/15131.Doc
dbs.0wp4aoe.cn/68682.Doc
dbs.0wp4aoe.cn/79555.Doc
dbs.0wp4aoe.cn/37315.Doc
dbs.0wp4aoe.cn/19513.Doc
dbs.0wp4aoe.cn/06404.Doc
dbs.0wp4aoe.cn/55951.Doc
dbs.0wp4aoe.cn/11315.Doc
dba.0wp4aoe.cn/55931.Doc
dba.0wp4aoe.cn/17795.Doc
dba.0wp4aoe.cn/31159.Doc
dba.0wp4aoe.cn/77593.Doc
dba.0wp4aoe.cn/59139.Doc
dba.0wp4aoe.cn/55951.Doc
dba.0wp4aoe.cn/57591.Doc
dba.0wp4aoe.cn/31197.Doc
dba.0wp4aoe.cn/02082.Doc
dba.0wp4aoe.cn/93315.Doc
