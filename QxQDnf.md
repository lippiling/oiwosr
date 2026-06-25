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

qjs.zhuzhoujiudazxL1517.cn/26868.Doc
qjs.zhuzhoujiudazxL1517.cn/80082.Doc
qjs.zhuzhoujiudazxL1517.cn/02648.Doc
qjs.zhuzhoujiudazxL1517.cn/24648.Doc
qjs.zhuzhoujiudazxL1517.cn/42884.Doc
qjs.zhuzhoujiudazxL1517.cn/26482.Doc
qjs.zhuzhoujiudazxL1517.cn/80420.Doc
qjs.zhuzhoujiudazxL1517.cn/44228.Doc
qjs.zhuzhoujiudazxL1517.cn/84884.Doc
qjs.zhuzhoujiudazxL1517.cn/80600.Doc
qja.zhuzhoujiudazxL1517.cn/62264.Doc
qja.zhuzhoujiudazxL1517.cn/40642.Doc
qja.zhuzhoujiudazxL1517.cn/26606.Doc
qja.zhuzhoujiudazxL1517.cn/84284.Doc
qja.zhuzhoujiudazxL1517.cn/44840.Doc
qja.zhuzhoujiudazxL1517.cn/28802.Doc
qja.zhuzhoujiudazxL1517.cn/88802.Doc
qja.zhuzhoujiudazxL1517.cn/40022.Doc
qja.zhuzhoujiudazxL1517.cn/00844.Doc
qja.zhuzhoujiudazxL1517.cn/82246.Doc
qjp.zhuzhoujiudazxL1517.cn/24848.Doc
qjp.zhuzhoujiudazxL1517.cn/62428.Doc
qjp.zhuzhoujiudazxL1517.cn/68404.Doc
qjp.zhuzhoujiudazxL1517.cn/46228.Doc
qjp.zhuzhoujiudazxL1517.cn/84000.Doc
qjp.zhuzhoujiudazxL1517.cn/82404.Doc
qjp.zhuzhoujiudazxL1517.cn/86406.Doc
qjp.zhuzhoujiudazxL1517.cn/46020.Doc
qjp.zhuzhoujiudazxL1517.cn/24022.Doc
qjp.zhuzhoujiudazxL1517.cn/24882.Doc
qjo.zhuzhoujiudazxL1517.cn/40008.Doc
qjo.zhuzhoujiudazxL1517.cn/28444.Doc
qjo.zhuzhoujiudazxL1517.cn/26048.Doc
qjo.zhuzhoujiudazxL1517.cn/62064.Doc
qjo.zhuzhoujiudazxL1517.cn/65339.Doc
qjo.zhuzhoujiudazxL1517.cn/84666.Doc
qjo.zhuzhoujiudazxL1517.cn/86204.Doc
qjo.zhuzhoujiudazxL1517.cn/68666.Doc
qjo.zhuzhoujiudazxL1517.cn/44088.Doc
qjo.zhuzhoujiudazxL1517.cn/86866.Doc
qji.zhuzhoujiudazxL1517.cn/64008.Doc
qji.zhuzhoujiudazxL1517.cn/62066.Doc
qji.zhuzhoujiudazxL1517.cn/88840.Doc
qji.zhuzhoujiudazxL1517.cn/24682.Doc
qji.zhuzhoujiudazxL1517.cn/24608.Doc
qji.zhuzhoujiudazxL1517.cn/26024.Doc
qji.zhuzhoujiudazxL1517.cn/04442.Doc
qji.zhuzhoujiudazxL1517.cn/46640.Doc
qji.zhuzhoujiudazxL1517.cn/20260.Doc
qji.zhuzhoujiudazxL1517.cn/08260.Doc
qju.zhuzhoujiudazxL1517.cn/08240.Doc
qju.zhuzhoujiudazxL1517.cn/20822.Doc
qju.zhuzhoujiudazxL1517.cn/80020.Doc
qju.zhuzhoujiudazxL1517.cn/64686.Doc
qju.zhuzhoujiudazxL1517.cn/46886.Doc
qju.zhuzhoujiudazxL1517.cn/88808.Doc
qju.zhuzhoujiudazxL1517.cn/86406.Doc
qju.zhuzhoujiudazxL1517.cn/80264.Doc
qju.zhuzhoujiudazxL1517.cn/04860.Doc
qju.zhuzhoujiudazxL1517.cn/64888.Doc
qjy.zhuzhoujiudazxL1517.cn/80000.Doc
qjy.zhuzhoujiudazxL1517.cn/86226.Doc
qjy.zhuzhoujiudazxL1517.cn/82642.Doc
qjy.zhuzhoujiudazxL1517.cn/00804.Doc
qjy.zhuzhoujiudazxL1517.cn/46606.Doc
qjy.zhuzhoujiudazxL1517.cn/46600.Doc
qjy.zhuzhoujiudazxL1517.cn/60408.Doc
qjy.zhuzhoujiudazxL1517.cn/62880.Doc
qjy.zhuzhoujiudazxL1517.cn/60224.Doc
qjy.zhuzhoujiudazxL1517.cn/22228.Doc
qjt.zhuzhoujiudazxL1517.cn/88644.Doc
qjt.zhuzhoujiudazxL1517.cn/84642.Doc
qjt.zhuzhoujiudazxL1517.cn/82480.Doc
qjt.zhuzhoujiudazxL1517.cn/44460.Doc
qjt.zhuzhoujiudazxL1517.cn/82686.Doc
qjt.zhuzhoujiudazxL1517.cn/66286.Doc
qjt.zhuzhoujiudazxL1517.cn/28406.Doc
qjt.zhuzhoujiudazxL1517.cn/44648.Doc
qjt.zhuzhoujiudazxL1517.cn/44288.Doc
qjt.zhuzhoujiudazxL1517.cn/48220.Doc
qjr.zhuzhoujiudazxL1517.cn/80200.Doc
qjr.zhuzhoujiudazxL1517.cn/24226.Doc
qjr.zhuzhoujiudazxL1517.cn/64484.Doc
qjr.zhuzhoujiudazxL1517.cn/00840.Doc
qjr.zhuzhoujiudazxL1517.cn/28066.Doc
qjr.zhuzhoujiudazxL1517.cn/44006.Doc
qjr.zhuzhoujiudazxL1517.cn/42246.Doc
qjr.zhuzhoujiudazxL1517.cn/62204.Doc
qjr.zhuzhoujiudazxL1517.cn/28866.Doc
qjr.zhuzhoujiudazxL1517.cn/24860.Doc
qje.zhuzhoujiudazxL1517.cn/68420.Doc
qje.zhuzhoujiudazxL1517.cn/51995.Doc
qje.zhuzhoujiudazxL1517.cn/02280.Doc
qje.zhuzhoujiudazxL1517.cn/46628.Doc
qje.zhuzhoujiudazxL1517.cn/04024.Doc
qje.zhuzhoujiudazxL1517.cn/77959.Doc
qje.zhuzhoujiudazxL1517.cn/46688.Doc
qje.zhuzhoujiudazxL1517.cn/60644.Doc
qje.zhuzhoujiudazxL1517.cn/00020.Doc
qje.zhuzhoujiudazxL1517.cn/44048.Doc
qjw.zhuzhoujiudazxL1517.cn/08246.Doc
qjw.zhuzhoujiudazxL1517.cn/66066.Doc
qjw.zhuzhoujiudazxL1517.cn/22280.Doc
qjw.zhuzhoujiudazxL1517.cn/08866.Doc
qjw.zhuzhoujiudazxL1517.cn/60666.Doc
qjw.zhuzhoujiudazxL1517.cn/46240.Doc
qjw.zhuzhoujiudazxL1517.cn/64284.Doc
qjw.zhuzhoujiudazxL1517.cn/46686.Doc
qjw.zhuzhoujiudazxL1517.cn/40422.Doc
qjw.zhuzhoujiudazxL1517.cn/08604.Doc
qjq.zhuzhoujiudazxL1517.cn/66624.Doc
qjq.zhuzhoujiudazxL1517.cn/42286.Doc
qjq.zhuzhoujiudazxL1517.cn/06886.Doc
qjq.zhuzhoujiudazxL1517.cn/79313.Doc
qjq.zhuzhoujiudazxL1517.cn/84824.Doc
qjq.zhuzhoujiudazxL1517.cn/80084.Doc
qjq.zhuzhoujiudazxL1517.cn/80848.Doc
qjq.zhuzhoujiudazxL1517.cn/68286.Doc
qjq.zhuzhoujiudazxL1517.cn/08084.Doc
qjq.zhuzhoujiudazxL1517.cn/42620.Doc
qhm.zhuzhoujiudazxL1517.cn/66668.Doc
qhm.zhuzhoujiudazxL1517.cn/48466.Doc
qhm.zhuzhoujiudazxL1517.cn/60462.Doc
qhm.zhuzhoujiudazxL1517.cn/28404.Doc
qhm.zhuzhoujiudazxL1517.cn/31393.Doc
qhm.zhuzhoujiudazxL1517.cn/88642.Doc
qhm.zhuzhoujiudazxL1517.cn/42448.Doc
qhm.zhuzhoujiudazxL1517.cn/93797.Doc
qhm.zhuzhoujiudazxL1517.cn/44684.Doc
qhm.zhuzhoujiudazxL1517.cn/44446.Doc
qhn.zhuzhoujiudazxL1517.cn/08080.Doc
qhn.zhuzhoujiudazxL1517.cn/91331.Doc
qhn.zhuzhoujiudazxL1517.cn/02248.Doc
qhn.zhuzhoujiudazxL1517.cn/24800.Doc
qhn.zhuzhoujiudazxL1517.cn/62684.Doc
qhn.zhuzhoujiudazxL1517.cn/20688.Doc
qhn.zhuzhoujiudazxL1517.cn/00846.Doc
qhn.zhuzhoujiudazxL1517.cn/48862.Doc
qhn.zhuzhoujiudazxL1517.cn/88264.Doc
qhn.zhuzhoujiudazxL1517.cn/28804.Doc
qhb.zhuzhoujiudazxL1517.cn/80246.Doc
qhb.zhuzhoujiudazxL1517.cn/66246.Doc
qhb.zhuzhoujiudazxL1517.cn/80600.Doc
qhb.zhuzhoujiudazxL1517.cn/04280.Doc
qhb.zhuzhoujiudazxL1517.cn/66066.Doc
qhb.zhuzhoujiudazxL1517.cn/82242.Doc
qhb.zhuzhoujiudazxL1517.cn/48484.Doc
qhb.zhuzhoujiudazxL1517.cn/20444.Doc
qhb.zhuzhoujiudazxL1517.cn/44426.Doc
qhb.zhuzhoujiudazxL1517.cn/82402.Doc
qhv.zhuzhoujiudazxL1517.cn/22068.Doc
qhv.zhuzhoujiudazxL1517.cn/68482.Doc
qhv.zhuzhoujiudazxL1517.cn/04084.Doc
qhv.zhuzhoujiudazxL1517.cn/00860.Doc
qhv.zhuzhoujiudazxL1517.cn/80446.Doc
qhv.zhuzhoujiudazxL1517.cn/48066.Doc
qhv.zhuzhoujiudazxL1517.cn/40284.Doc
qhv.zhuzhoujiudazxL1517.cn/28406.Doc
qhv.zhuzhoujiudazxL1517.cn/82460.Doc
qhv.zhuzhoujiudazxL1517.cn/86286.Doc
qhc.zhuzhoujiudazxL1517.cn/08026.Doc
qhc.zhuzhoujiudazxL1517.cn/02666.Doc
qhc.zhuzhoujiudazxL1517.cn/06105.Doc
qhc.zhuzhoujiudazxL1517.cn/92502.Doc
qhc.zhuzhoujiudazxL1517.cn/46501.Doc
qhc.zhuzhoujiudazxL1517.cn/33675.Doc
qhc.zhuzhoujiudazxL1517.cn/37782.Doc
qhc.zhuzhoujiudazxL1517.cn/61663.Doc
qhc.zhuzhoujiudazxL1517.cn/39448.Doc
qhc.zhuzhoujiudazxL1517.cn/84826.Doc
qhx.zhuzhoujiudazxL1517.cn/29270.Doc
qhx.zhuzhoujiudazxL1517.cn/13437.Doc
qhx.zhuzhoujiudazxL1517.cn/95481.Doc
qhx.zhuzhoujiudazxL1517.cn/06700.Doc
qhx.zhuzhoujiudazxL1517.cn/35541.Doc
qhx.zhuzhoujiudazxL1517.cn/27658.Doc
qhx.zhuzhoujiudazxL1517.cn/26227.Doc
qhx.zhuzhoujiudazxL1517.cn/30226.Doc
qhx.zhuzhoujiudazxL1517.cn/51350.Doc
qhx.zhuzhoujiudazxL1517.cn/44032.Doc
qhz.zhuzhoujiudazxL1517.cn/54118.Doc
qhz.zhuzhoujiudazxL1517.cn/10551.Doc
qhz.zhuzhoujiudazxL1517.cn/46354.Doc
qhz.zhuzhoujiudazxL1517.cn/60892.Doc
qhz.zhuzhoujiudazxL1517.cn/86688.Doc
qhz.zhuzhoujiudazxL1517.cn/72464.Doc
qhz.zhuzhoujiudazxL1517.cn/65973.Doc
qhz.zhuzhoujiudazxL1517.cn/96148.Doc
qhz.zhuzhoujiudazxL1517.cn/41893.Doc
qhz.zhuzhoujiudazxL1517.cn/11689.Doc
qhl.zhuzhoujiudazxL1517.cn/40042.Doc
qhl.zhuzhoujiudazxL1517.cn/24466.Doc
qhl.zhuzhoujiudazxL1517.cn/66680.Doc
qhl.zhuzhoujiudazxL1517.cn/40444.Doc
qhl.zhuzhoujiudazxL1517.cn/82888.Doc
qhl.zhuzhoujiudazxL1517.cn/82604.Doc
qhl.zhuzhoujiudazxL1517.cn/02006.Doc
qhl.zhuzhoujiudazxL1517.cn/84248.Doc
qhl.zhuzhoujiudazxL1517.cn/42660.Doc
qhl.zhuzhoujiudazxL1517.cn/68046.Doc
qhk.zhuzhoujiudazxL1517.cn/80080.Doc
qhk.zhuzhoujiudazxL1517.cn/84646.Doc
qhk.zhuzhoujiudazxL1517.cn/46282.Doc
qhk.zhuzhoujiudazxL1517.cn/48640.Doc
qhk.zhuzhoujiudazxL1517.cn/40888.Doc
qhk.zhuzhoujiudazxL1517.cn/26240.Doc
qhk.zhuzhoujiudazxL1517.cn/22868.Doc
qhk.zhuzhoujiudazxL1517.cn/42066.Doc
qhk.zhuzhoujiudazxL1517.cn/84808.Doc
qhk.zhuzhoujiudazxL1517.cn/80624.Doc
qhj.zhuzhoujiudazxL1517.cn/84246.Doc
qhj.zhuzhoujiudazxL1517.cn/86028.Doc
qhj.zhuzhoujiudazxL1517.cn/24446.Doc
qhj.zhuzhoujiudazxL1517.cn/64666.Doc
qhj.zhuzhoujiudazxL1517.cn/02826.Doc
qhj.zhuzhoujiudazxL1517.cn/22422.Doc
qhj.zhuzhoujiudazxL1517.cn/42026.Doc
qhj.zhuzhoujiudazxL1517.cn/44200.Doc
qhj.zhuzhoujiudazxL1517.cn/68044.Doc
qhj.zhuzhoujiudazxL1517.cn/40886.Doc
qhh.zhuzhoujiudazxL1517.cn/60602.Doc
qhh.zhuzhoujiudazxL1517.cn/84862.Doc
qhh.zhuzhoujiudazxL1517.cn/80444.Doc
qhh.zhuzhoujiudazxL1517.cn/48800.Doc
qhh.zhuzhoujiudazxL1517.cn/62824.Doc
qhh.zhuzhoujiudazxL1517.cn/20002.Doc
qhh.zhuzhoujiudazxL1517.cn/06048.Doc
qhh.zhuzhoujiudazxL1517.cn/00862.Doc
qhh.zhuzhoujiudazxL1517.cn/84626.Doc
qhh.zhuzhoujiudazxL1517.cn/04442.Doc
qhg.zhuzhoujiudazxL1517.cn/20680.Doc
qhg.zhuzhoujiudazxL1517.cn/08066.Doc
qhg.zhuzhoujiudazxL1517.cn/68886.Doc
qhg.zhuzhoujiudazxL1517.cn/40064.Doc
qhg.zhuzhoujiudazxL1517.cn/84482.Doc
qhg.zhuzhoujiudazxL1517.cn/86004.Doc
qhg.zhuzhoujiudazxL1517.cn/68620.Doc
qhg.zhuzhoujiudazxL1517.cn/06864.Doc
qhg.zhuzhoujiudazxL1517.cn/48264.Doc
qhg.zhuzhoujiudazxL1517.cn/46442.Doc
qhf.zhuzhoujiudazxL1517.cn/62482.Doc
qhf.zhuzhoujiudazxL1517.cn/88222.Doc
qhf.zhuzhoujiudazxL1517.cn/22284.Doc
qhf.zhuzhoujiudazxL1517.cn/22084.Doc
qhf.zhuzhoujiudazxL1517.cn/86826.Doc
qhf.zhuzhoujiudazxL1517.cn/00440.Doc
qhf.zhuzhoujiudazxL1517.cn/40088.Doc
qhf.zhuzhoujiudazxL1517.cn/02604.Doc
qhf.zhuzhoujiudazxL1517.cn/42480.Doc
qhf.zhuzhoujiudazxL1517.cn/46266.Doc
qhd.zhuzhoujiudazxL1517.cn/88824.Doc
qhd.zhuzhoujiudazxL1517.cn/06046.Doc
qhd.zhuzhoujiudazxL1517.cn/62280.Doc
qhd.zhuzhoujiudazxL1517.cn/42448.Doc
qhd.zhuzhoujiudazxL1517.cn/86828.Doc
qhd.zhuzhoujiudazxL1517.cn/82800.Doc
qhd.zhuzhoujiudazxL1517.cn/40888.Doc
qhd.zhuzhoujiudazxL1517.cn/84202.Doc
qhd.zhuzhoujiudazxL1517.cn/40448.Doc
qhd.zhuzhoujiudazxL1517.cn/86622.Doc
qhs.zhuzhoujiudazxL1517.cn/84206.Doc
qhs.zhuzhoujiudazxL1517.cn/42686.Doc
qhs.zhuzhoujiudazxL1517.cn/62220.Doc
qhs.zhuzhoujiudazxL1517.cn/60662.Doc
qhs.zhuzhoujiudazxL1517.cn/40408.Doc
qhs.zhuzhoujiudazxL1517.cn/02264.Doc
qhs.zhuzhoujiudazxL1517.cn/44044.Doc
qhs.zhuzhoujiudazxL1517.cn/22284.Doc
qhs.zhuzhoujiudazxL1517.cn/28260.Doc
qhs.zhuzhoujiudazxL1517.cn/80204.Doc
qha.zhuzhoujiudazxL1517.cn/22884.Doc
qha.zhuzhoujiudazxL1517.cn/06804.Doc
qha.zhuzhoujiudazxL1517.cn/60804.Doc
qha.zhuzhoujiudazxL1517.cn/28420.Doc
qha.zhuzhoujiudazxL1517.cn/68288.Doc
qha.zhuzhoujiudazxL1517.cn/06000.Doc
qha.zhuzhoujiudazxL1517.cn/68640.Doc
qha.zhuzhoujiudazxL1517.cn/08220.Doc
qha.zhuzhoujiudazxL1517.cn/04286.Doc
qha.zhuzhoujiudazxL1517.cn/20864.Doc
qhp.zhuzhoujiudazxL1517.cn/04444.Doc
qhp.zhuzhoujiudazxL1517.cn/02402.Doc
qhp.zhuzhoujiudazxL1517.cn/42822.Doc
qhp.zhuzhoujiudazxL1517.cn/26208.Doc
qhp.zhuzhoujiudazxL1517.cn/04640.Doc
qhp.zhuzhoujiudazxL1517.cn/20480.Doc
qhp.zhuzhoujiudazxL1517.cn/28866.Doc
qhp.zhuzhoujiudazxL1517.cn/42424.Doc
qhp.zhuzhoujiudazxL1517.cn/00222.Doc
qhp.zhuzhoujiudazxL1517.cn/62800.Doc
qho.zhuzhoujiudazxL1517.cn/00820.Doc
qho.zhuzhoujiudazxL1517.cn/62442.Doc
qho.zhuzhoujiudazxL1517.cn/02662.Doc
qho.zhuzhoujiudazxL1517.cn/46844.Doc
qho.zhuzhoujiudazxL1517.cn/04866.Doc
qho.zhuzhoujiudazxL1517.cn/42086.Doc
qho.zhuzhoujiudazxL1517.cn/04660.Doc
qho.zhuzhoujiudazxL1517.cn/06608.Doc
qho.zhuzhoujiudazxL1517.cn/24004.Doc
qho.zhuzhoujiudazxL1517.cn/44482.Doc
qhi.zhuzhoujiudazxL1517.cn/46804.Doc
qhi.zhuzhoujiudazxL1517.cn/64422.Doc
qhi.zhuzhoujiudazxL1517.cn/44848.Doc
qhi.zhuzhoujiudazxL1517.cn/60026.Doc
qhi.zhuzhoujiudazxL1517.cn/04804.Doc
qhi.zhuzhoujiudazxL1517.cn/62280.Doc
qhi.zhuzhoujiudazxL1517.cn/42688.Doc
qhi.zhuzhoujiudazxL1517.cn/68806.Doc
qhi.zhuzhoujiudazxL1517.cn/48688.Doc
qhi.zhuzhoujiudazxL1517.cn/24828.Doc
qhu.zhuzhoujiudazxL1517.cn/26480.Doc
qhu.zhuzhoujiudazxL1517.cn/06866.Doc
qhu.zhuzhoujiudazxL1517.cn/80884.Doc
qhu.zhuzhoujiudazxL1517.cn/48606.Doc
qhu.zhuzhoujiudazxL1517.cn/46028.Doc
qhu.zhuzhoujiudazxL1517.cn/82228.Doc
qhu.zhuzhoujiudazxL1517.cn/48086.Doc
qhu.zhuzhoujiudazxL1517.cn/26868.Doc
qhu.zhuzhoujiudazxL1517.cn/46808.Doc
qhu.zhuzhoujiudazxL1517.cn/28624.Doc
qhy.zhuzhoujiudazxL1517.cn/46068.Doc
qhy.zhuzhoujiudazxL1517.cn/64800.Doc
qhy.zhuzhoujiudazxL1517.cn/22402.Doc
qhy.zhuzhoujiudazxL1517.cn/64860.Doc
qhy.zhuzhoujiudazxL1517.cn/48888.Doc
qhy.zhuzhoujiudazxL1517.cn/28624.Doc
qhy.zhuzhoujiudazxL1517.cn/66060.Doc
qhy.zhuzhoujiudazxL1517.cn/80086.Doc
qhy.zhuzhoujiudazxL1517.cn/22086.Doc
qhy.zhuzhoujiudazxL1517.cn/80262.Doc
qht.zhuzhoujiudazxL1517.cn/24068.Doc
qht.zhuzhoujiudazxL1517.cn/26068.Doc
qht.zhuzhoujiudazxL1517.cn/26682.Doc
qht.zhuzhoujiudazxL1517.cn/48664.Doc
qht.zhuzhoujiudazxL1517.cn/28446.Doc
qht.zhuzhoujiudazxL1517.cn/86622.Doc
qht.zhuzhoujiudazxL1517.cn/46062.Doc
qht.zhuzhoujiudazxL1517.cn/42248.Doc
qht.zhuzhoujiudazxL1517.cn/68224.Doc
qht.zhuzhoujiudazxL1517.cn/46848.Doc
qhr.zhuzhoujiudazxL1517.cn/06260.Doc
qhr.zhuzhoujiudazxL1517.cn/26244.Doc
qhr.zhuzhoujiudazxL1517.cn/42002.Doc
qhr.zhuzhoujiudazxL1517.cn/80846.Doc
qhr.zhuzhoujiudazxL1517.cn/68806.Doc
qhr.zhuzhoujiudazxL1517.cn/66600.Doc
qhr.zhuzhoujiudazxL1517.cn/04866.Doc
qhr.zhuzhoujiudazxL1517.cn/82804.Doc
qhr.zhuzhoujiudazxL1517.cn/60882.Doc
qhr.zhuzhoujiudazxL1517.cn/02482.Doc
qhe.zhuzhoujiudazxL1517.cn/62008.Doc
qhe.zhuzhoujiudazxL1517.cn/68642.Doc
qhe.zhuzhoujiudazxL1517.cn/20488.Doc
qhe.zhuzhoujiudazxL1517.cn/62208.Doc
qhe.zhuzhoujiudazxL1517.cn/64888.Doc
qhe.zhuzhoujiudazxL1517.cn/06084.Doc
qhe.zhuzhoujiudazxL1517.cn/46064.Doc
qhe.zhuzhoujiudazxL1517.cn/44646.Doc
qhe.zhuzhoujiudazxL1517.cn/46602.Doc
qhe.zhuzhoujiudazxL1517.cn/04024.Doc
qhw.zhuzhoujiudazxL1517.cn/42086.Doc
qhw.zhuzhoujiudazxL1517.cn/84664.Doc
qhw.zhuzhoujiudazxL1517.cn/46062.Doc
qhw.zhuzhoujiudazxL1517.cn/86884.Doc
qhw.zhuzhoujiudazxL1517.cn/00844.Doc
qhw.zhuzhoujiudazxL1517.cn/84000.Doc
qhw.zhuzhoujiudazxL1517.cn/60622.Doc
qhw.zhuzhoujiudazxL1517.cn/08604.Doc
qhw.zhuzhoujiudazxL1517.cn/20804.Doc
qhw.zhuzhoujiudazxL1517.cn/06068.Doc
qhq.zhuzhoujiudazxL1517.cn/24042.Doc
qhq.zhuzhoujiudazxL1517.cn/08680.Doc
qhq.zhuzhoujiudazxL1517.cn/40084.Doc
qhq.zhuzhoujiudazxL1517.cn/20820.Doc
qhq.zhuzhoujiudazxL1517.cn/19377.Doc
qhq.zhuzhoujiudazxL1517.cn/00204.Doc
qhq.zhuzhoujiudazxL1517.cn/20882.Doc
qhq.zhuzhoujiudazxL1517.cn/48244.Doc
qhq.zhuzhoujiudazxL1517.cn/68060.Doc
qhq.zhuzhoujiudazxL1517.cn/79933.Doc
qgm.zhuzhoujiudazxL1517.cn/04488.Doc
qgm.zhuzhoujiudazxL1517.cn/28684.Doc
qgm.zhuzhoujiudazxL1517.cn/95771.Doc
qgm.zhuzhoujiudazxL1517.cn/84828.Doc
qgm.zhuzhoujiudazxL1517.cn/06824.Doc
qgm.zhuzhoujiudazxL1517.cn/68064.Doc
qgm.zhuzhoujiudazxL1517.cn/26468.Doc
qgm.zhuzhoujiudazxL1517.cn/64204.Doc
qgm.zhuzhoujiudazxL1517.cn/48086.Doc
qgm.zhuzhoujiudazxL1517.cn/68286.Doc
qgn.zhuzhoujiudazxL1517.cn/68060.Doc
qgn.zhuzhoujiudazxL1517.cn/22086.Doc
qgn.zhuzhoujiudazxL1517.cn/02022.Doc
qgn.zhuzhoujiudazxL1517.cn/26440.Doc
qgn.zhuzhoujiudazxL1517.cn/64828.Doc
qgn.zhuzhoujiudazxL1517.cn/40680.Doc
qgn.zhuzhoujiudazxL1517.cn/06402.Doc
qgn.zhuzhoujiudazxL1517.cn/28646.Doc
qgn.zhuzhoujiudazxL1517.cn/37315.Doc
qgn.zhuzhoujiudazxL1517.cn/66420.Doc
qgb.zhuzhoujiudazxL1517.cn/46066.Doc
qgb.zhuzhoujiudazxL1517.cn/68840.Doc
qgb.zhuzhoujiudazxL1517.cn/08864.Doc
qgb.zhuzhoujiudazxL1517.cn/82048.Doc
qgb.zhuzhoujiudazxL1517.cn/64408.Doc
qgb.zhuzhoujiudazxL1517.cn/46228.Doc
qgb.zhuzhoujiudazxL1517.cn/40208.Doc
qgb.zhuzhoujiudazxL1517.cn/08646.Doc
qgb.zhuzhoujiudazxL1517.cn/80060.Doc
qgb.zhuzhoujiudazxL1517.cn/42402.Doc
qgv.zhuzhoujiudazxL1517.cn/28084.Doc
qgv.zhuzhoujiudazxL1517.cn/37931.Doc
qgv.zhuzhoujiudazxL1517.cn/59915.Doc
qgv.zhuzhoujiudazxL1517.cn/40420.Doc
qgv.zhuzhoujiudazxL1517.cn/66824.Doc
qgv.zhuzhoujiudazxL1517.cn/80648.Doc
qgv.zhuzhoujiudazxL1517.cn/84282.Doc
qgv.zhuzhoujiudazxL1517.cn/60460.Doc
qgv.zhuzhoujiudazxL1517.cn/06062.Doc
qgv.zhuzhoujiudazxL1517.cn/28462.Doc
qgc.zhuzhoujiudazxL1517.cn/66464.Doc
qgc.zhuzhoujiudazxL1517.cn/24064.Doc
qgc.zhuzhoujiudazxL1517.cn/42462.Doc
qgc.zhuzhoujiudazxL1517.cn/82662.Doc
qgc.zhuzhoujiudazxL1517.cn/88464.Doc
qgc.zhuzhoujiudazxL1517.cn/02868.Doc
qgc.zhuzhoujiudazxL1517.cn/80866.Doc
qgc.zhuzhoujiudazxL1517.cn/26684.Doc
qgc.zhuzhoujiudazxL1517.cn/84028.Doc
qgc.zhuzhoujiudazxL1517.cn/86028.Doc
qgx.zhuzhoujiudazxL1517.cn/80208.Doc
qgx.zhuzhoujiudazxL1517.cn/44664.Doc
qgx.zhuzhoujiudazxL1517.cn/84042.Doc
qgx.zhuzhoujiudazxL1517.cn/60682.Doc
qgx.zhuzhoujiudazxL1517.cn/20046.Doc
qgx.zhuzhoujiudazxL1517.cn/13319.Doc
qgx.zhuzhoujiudazxL1517.cn/02648.Doc
qgx.zhuzhoujiudazxL1517.cn/80288.Doc
qgx.zhuzhoujiudazxL1517.cn/46862.Doc
qgx.zhuzhoujiudazxL1517.cn/64424.Doc
qgz.zhuzhoujiudazxL1517.cn/48680.Doc
qgz.zhuzhoujiudazxL1517.cn/00048.Doc
qgz.zhuzhoujiudazxL1517.cn/42486.Doc
qgz.zhuzhoujiudazxL1517.cn/22044.Doc
qgz.zhuzhoujiudazxL1517.cn/20044.Doc
qgz.zhuzhoujiudazxL1517.cn/64804.Doc
qgz.zhuzhoujiudazxL1517.cn/08846.Doc
qgz.zhuzhoujiudazxL1517.cn/60206.Doc
qgz.zhuzhoujiudazxL1517.cn/68084.Doc
qgz.zhuzhoujiudazxL1517.cn/66402.Doc
qgl.zhuzhoujiudazxL1517.cn/71155.Doc
qgl.zhuzhoujiudazxL1517.cn/20886.Doc
qgl.zhuzhoujiudazxL1517.cn/86444.Doc
qgl.zhuzhoujiudazxL1517.cn/60044.Doc
qgl.zhuzhoujiudazxL1517.cn/04646.Doc
qgl.zhuzhoujiudazxL1517.cn/22822.Doc
qgl.zhuzhoujiudazxL1517.cn/66486.Doc
qgl.zhuzhoujiudazxL1517.cn/44480.Doc
qgl.zhuzhoujiudazxL1517.cn/13577.Doc
qgl.zhuzhoujiudazxL1517.cn/57331.Doc
qgk.zhuzhoujiudazxL1517.cn/26200.Doc
qgk.zhuzhoujiudazxL1517.cn/44422.Doc
qgk.zhuzhoujiudazxL1517.cn/00460.Doc
qgk.zhuzhoujiudazxL1517.cn/24040.Doc
qgk.zhuzhoujiudazxL1517.cn/24062.Doc
qgk.zhuzhoujiudazxL1517.cn/82068.Doc
qgk.zhuzhoujiudazxL1517.cn/26802.Doc
qgk.zhuzhoujiudazxL1517.cn/68848.Doc
qgk.zhuzhoujiudazxL1517.cn/88266.Doc
qgk.zhuzhoujiudazxL1517.cn/28064.Doc
qgj.zhuzhoujiudazxL1517.cn/08264.Doc
qgj.zhuzhoujiudazxL1517.cn/82008.Doc
qgj.zhuzhoujiudazxL1517.cn/55179.Doc
qgj.zhuzhoujiudazxL1517.cn/48402.Doc
qgj.zhuzhoujiudazxL1517.cn/60022.Doc
qgj.zhuzhoujiudazxL1517.cn/22606.Doc
qgj.zhuzhoujiudazxL1517.cn/46464.Doc
qgj.zhuzhoujiudazxL1517.cn/86408.Doc
qgj.zhuzhoujiudazxL1517.cn/57137.Doc
qgj.zhuzhoujiudazxL1517.cn/28286.Doc
qgh.zhuzhoujiudazxL1517.cn/60486.Doc
qgh.zhuzhoujiudazxL1517.cn/86062.Doc
qgh.zhuzhoujiudazxL1517.cn/28262.Doc
qgh.zhuzhoujiudazxL1517.cn/66226.Doc
qgh.zhuzhoujiudazxL1517.cn/86420.Doc
qgh.zhuzhoujiudazxL1517.cn/08224.Doc
qgh.zhuzhoujiudazxL1517.cn/40482.Doc
qgh.zhuzhoujiudazxL1517.cn/26484.Doc
qgh.zhuzhoujiudazxL1517.cn/60682.Doc
qgh.zhuzhoujiudazxL1517.cn/08266.Doc
