Python数据库编程与ORM

一、SQLite基础

import sqlite3

conn = sqlite3.connect('example.db')
cursor = conn.cursor()

cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE,
        age INTEGER
    )
''')

# 参数化查询（防SQL注入）
cursor.execute(
    "INSERT INTO users (name, email, age) VALUES (?, ?, ?)",
    ("Alice", "alice@example.com", 30)
)

# 批量插入
users = [("Bob", "bob@e.com", 25), ("Charlie", "charlie@e.com", 35)]
cursor.executemany("INSERT INTO users (name, email, age) VALUES (?, ?, ?)", users)

conn.commit()

# 查询
cursor.execute("SELECT * FROM users WHERE age > ?", (25,))
for row in cursor.fetchall():
    print(row)

conn.close()


二、连接上下文管理

from contextlib import contextmanager

@contextmanager
def get_db(path):
    conn = sqlite3.connect(path)
    conn.row_factory = sqlite3.Row
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

with get_db('example.db') as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users")
    for row in cursor.fetchall():
        print(dict(row))


三、SQLAlchemy Core

from sqlalchemy import create_engine, MetaData, Table, Column
from sqlalchemy import Integer, String, select, insert

engine = create_engine('sqlite:///example.db', echo=True)
metadata = MetaData()

users = Table('users', metadata,
    Column('id', Integer, primary_key=True),
    Column('name', String(50)),
    Column('email', String(100)),
)

metadata.create_all(engine)

with engine.connect() as conn:
    conn.execute(insert(users).values(name='Eve', email='eve@e.com'))

    stmt = select(users).where(users.c.name.like('%E%'))
    for row in conn.execute(stmt):
        print(row.name)

    conn.commit()


四、SQLAlchemy ORM

from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = 'users'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    email: Mapped[str] = mapped_column(String(100), unique=True)

Base.metadata.create_all(engine)

with Session(engine) as session:
    # 创建
    user = User(name='Frank', email='frank@e.com')
    session.add(user)
    session.commit()

    # 查询
    users = session.query(User).filter(User.name.like('%F%')).all()
    for u in users:
        print(u.name)

    # 关系
    class Order(Base):
        __tablename__ = 'orders'
        id: Mapped[int] = mapped_column(primary_key=True)
        user_id: Mapped[int] = mapped_column(ForeignKey('users.id'))
        amount: Mapped[float]

    # 预加载避免N+1
    from sqlalchemy.orm import joinedload
    stmt = select(User).options(joinedload(User.orders))


五、数据库迁移（Alembic）

# alembic init migrations
# alembic revision --autogenerate -m "add phone column"
# alembic upgrade head


六、Repository模式

class UserRepo:
    def __init__(self, session):
        self.session = session
    def get_by_id(self, id):
        return self.session.get(User, id)
    def get_by_email(self, email):
        return self.session.query(User).filter(User.email == email).first()
    def add(self, user):
        self.session.add(user)
        self.session.flush()
        return user


七、查询优化

# 只查需要的列
stmt = select(User.name, User.email)

# 分页
stmt = select(User).offset(20).limit(10)

# 索引
class User(Base):
    __table_args__ = (Index('idx_email', 'email'),)

总结：参数化查询是基本安全要求。ORM方便但注意N+1问题。使用连接池管理连接。

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
