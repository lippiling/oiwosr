肚口卣是毡


Python SQLAlchemy 关系映射与高级模式详解
============================================

一、relationship() 与 back_populates 双向关系
---------------------------------------------

在 SQLAlchemy 中，relationship() 用于定义模型之间的对象关系。
back_populates 参数用于建立双向关系，让两端都能感知对方。

from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base, relationship, Session
from sqlalchemy.orm import joinedload, selectinload, subqueryload

Base = declarative_base()


# 一对多关系：一个作者有多篇文章
class Author(Base):
    __tablename__ = 'authors'
    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)

    # 定义反向引用，Article 通过 back_populates 关联回来
    articles = relationship("Article", back_populates="author")


class Article(Base):
    __tablename__ = 'articles'
    id = Column(Integer, primary_key=True)
    title = Column(String(100), nullable=False)
    author_id = Column(Integer, ForeignKey('authors.id'))

    # 定义正向引用，指向 Author
    author = relationship("Author", back_populates="articles")


# 使用示例
engine = create_engine("sqlite:///:memory:", echo=True)
Base.metadata.create_all(engine)

with Session(engine) as session:
    author = Author(name="张三")
    article = Article(title="SQLAlchemy 入门教程", author=author)
    session.add(author)
    session.add(article)
    session.commit()

    # 双向访问：通过作者找文章，或通过文章找作者
    saved_author = session.get(Author, 1)
    print(saved_author.articles[0].title)  # 通过关系访问文章
    print(saved_author.articles[0].author.name)  # 反向访问作者


二、加载策略：懒加载 vs 预加载
-------------------------------

SQLAlchemy 默认使用懒加载（lazy loading），即在访问关系属性时才发送查询。
预加载（eager loading）通过在一次查询中 JOIN 或使用额外 SELECT 来减少查询次数。

# --- 预加载方式对比 ---

# 1. selectinload：发出额外的 SELECT … WHERE id IN (…) 查询
# 优点：不会产生笛卡尔积，适合加载一对多关系
# 缺点：多条 SQL 语句
from sqlalchemy.orm import selectinload

with Session(engine) as session:
    authors = session.query(Author).options(
        selectinload(Author.articles)
    ).all()
    for a in authors:
        print(f"作者 {a.name} 有 {len(a.articles)} 篇文章")

# 2. joinedload：使用 LEFT OUTER JOIN 一次查询
# 优点：单条 SQL，适合多对一关系
# 缺点：一对多时会产生笛卡尔积，数据量大会重复
from sqlalchemy.orm import joinedload

with Session(engine) as session:
    articles = session.query(Article).options(
        joinedload(Article.author)
    ).all()
    for article in articles:
        print(f"文章《{article.title}》作者是 {article.author.name}")

# 3. subqueryload：使用子查询的方式加载
# 优点：避免笛卡尔积，适合复杂查询
# 缺点：子查询可能性能较差
from sqlalchemy.orm import subqueryload

with Session(engine) as session:
    authors = session.query(Author).options(
        subqueryload(Author.articles)
    ).all()


三、多对多关系与 secondary 参数
-------------------------------

多对多关系通过中间表实现，secondary 参数指定中间表的名称。

# 文章和标签的多对多关系
article_tag = Table(
    'article_tags', Base.metadata,
    Column('article_id', Integer, ForeignKey('articles.id'), primary_key=True),
    Column('tag_id', Integer, ForeignKey('tags.id'), primary_key=True)
)


class Tag(Base):
    __tablename__ = 'tags'
    id = Column(Integer, primary_key=True)
    name = Column(String(30), unique=True, nullable=False)

    articles = relationship("Article", secondary=article_tag, back_populates="tags")


# 注意：需要在 Article 类中补充 tags 关系
Article.tags = relationship("Tag", secondary=article_tag, back_populates="articles")

with Session(engine) as session:
    tag_python = Tag(name="Python")
    tag_db = Tag(name="数据库")
    article = Article(title="SQLAlchemy 高级用法")
    article.tags.extend([tag_python, tag_db])
    session.add(article)
    session.commit()

    # 查询时自动 JOIN 中间表
    article = session.query(Article).first()
    print([tag.name for tag in article.tags])  # ['Python', '数据库']


四、关联对象模式（Association Object）
--------------------------------------

当中间表需要存储额外字段时，使用关联对象模式代替简单的 secondary。
例如：记录用户加入群组的时间和角色。

class Membership(Base):
    """关联对象，存储用户和群组之间的额外信息"""
    __tablename__ = 'memberships'

    user_id = Column(Integer, ForeignKey('users.id'), primary_key=True)
    group_id = Column(Integer, ForeignKey('groups.id'), primary_key=True)
    joined_at = Column(DateTime, default=func.now())  # 加入时间
    role = Column(String(20), default='member')       # 角色：member/admin

    # 双向关联到 User 和 Group
    user = relationship("User", back_populates="memberships")
    group = relationship("Group", back_populates="memberships")


class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String(50))
    memberships = relationship("Membership", back_populates="user")


class Group(Base):
    __tablename__ = 'groups'
    id = Column(Integer, primary_key=True)
    name = Column(String(50))
    memberships = relationship("Membership", back_populates="group")

# 使用关联对象创建关系
with Session(engine) as session:
    user = User(name="李四")
    group = Group(name="Python学习群")
    membership = Membership(user=user, group=group, role="admin")
    session.add_all([user, group, membership])
    session.commit()


五、hybrid_property 计算属性
----------------------------

hybrid_property 允许在 Python 层面和 SQL 层面都参与计算的属性。
它在类上定义，既可以在实例上访问，也可以在查询中作为过滤条件。

from sqlalchemy.ext.hybrid import hybrid_property

class Order(Base):
    __tablename__ = 'orders'
    id = Column(Integer, primary_key=True)
    quantity = Column(Integer, nullable=False, default=1)
    unit_price = Column(Integer, nullable=False)  # 单位：分

    @hybrid_property
    def total_amount(self):
        """计算总金额（元），将分转换为元"""
        return self.quantity * self.unit_price / 100.0

    @total_amount.expression
    def total_amount(cls):
        """SQL 层面的表达式，可直接用于查询过滤"""
        return (cls.quantity * cls.unit_price) / 100.0

# 实例级别使用
order = Order(quantity=3, unit_price=5000)
print(f"总金额: {order.total_amount} 元")  # 150.0 元

# SQL 级别使用 —— 在 WHERE 中过滤
expensive_orders = session.query(Order).filter(
    Order.total_amount > 100
).all()  # 生成 SQL: WHERE (quantity * unit_price / 100.0) > 100


六、@validates 属性验证
-----------------------

@validates 装饰器用于在设置属性值时进行验证，可以在数据持久化前拦截并修改值。

from sqlalchemy.orm import validates

class Product(Base):
    __tablename__ = 'products'
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    price = Column(Integer, nullable=False)  # 单位：分
    stock = Column(Integer, default=0)

    @validates('price')
    def validate_price(self, key, value):
        """价格不能为负数，且必须是整数（单位：分）"""
        if value is None or value < 0:
            raise ValueError(f"价格无效: {value}")
        return int(value)

    @validates('name')
    def validate_name(self, key, value):
        """名称不能为空，且自动去除首尾空格"""
        if not value or not value.strip():
            raise ValueError("产品名称不能为空")
        return value.strip()

    @validates('stock')
    def validate_stock(self, key, value):
        """库存不能为负数"""
        if value < 0:
            raise ValueError("库存不能为负数")
        return value

# 测试验证逻辑
try:
    product = Product(name="  笔记本电脑  ", price=-100, stock=10)
    # price 验证会触发 ValueError
except ValueError as e:
    print(f"验证失败: {e}")

# 正确创建
product = Product(name="  笔记本电脑  ", price=500000, stock=10)
print(f"名称自动去空格: '{product.name}'")  # '笔记本电脑'


七、属性事件监听
-----------------

SQLAlchemy 提供 attribute events 来监听属性的 set/append/remove 操作。
这对于审计日志、自动更新缓存等场景非常有用。

from sqlalchemy import event

class ShoppingCart(Base):
    __tablename__ = 'shopping_carts'
    id = Column(Integer, primary_key=True)
    items = relationship("CartItem", back_populates="cart")


class CartItem(Base):
    __tablename__ = 'cart_items'
    id = Column(Integer, primary_key=True)
    cart_id = Column(Integer, ForeignKey('shopping_carts.id'))
    product_name = Column(String(100))
    quantity = Column(Integer, default=1)
    cart = relationship("ShoppingCart", back_populates="items")


# --- 监听属性设置事件 ---
@event.listens_for(Product.stock, 'set')
def on_stock_change(target, value, oldvalue, initiator):
    """库存变更时记录日志"""
    print(f"[审计] 产品 {target.id} 库存变更: {oldvalue} -> {value}")
    # 可在此处写入审计日志表


# --- 监听集合的 append/remove 事件 ---
@event.listens_for(ShoppingCart.items, 'append')
def on_item_added(target, value, initiator):
    """购物车添加商品时触发"""
    print(f"[购物车] 添加商品: {value.product_name} x {value.quantity}")
    # 可触发价格重新计算等逻辑


@event.listens_for(ShoppingCart.items, 'remove')
def on_item_removed(target, value, initiator):
    """购物车移除商品时触发"""
    print(f"[购物车] 移除商品: {value.product_name}")
    # 可触发库存恢复等逻辑

# 事件触发演示
with Session(engine) as session:
    cart = ShoppingCart()
    session.add(cart)

    item = CartItem(product_name="键盘", quantity=2)
    cart.items.append(item)  # 触发 append 事件
    cart.items.remove(item)  # 触发 remove 事件
    session.commit()

系荣驴烈备擞倚谰桥恢詹飞刃浊檬

m.erw.cccnt.cn/55915.Doc
m.erw.cccnt.cn/79557.Doc
m.erw.cccnt.cn/42008.Doc
m.erw.cccnt.cn/93393.Doc
m.erw.cccnt.cn/31711.Doc
m.erw.cccnt.cn/31931.Doc
m.erw.cccnt.cn/26000.Doc
m.erw.cccnt.cn/55391.Doc
m.erw.cccnt.cn/75931.Doc
m.erw.cccnt.cn/75759.Doc
m.erq.cccnt.cn/19115.Doc
m.erq.cccnt.cn/46428.Doc
m.erq.cccnt.cn/13731.Doc
m.erq.cccnt.cn/06066.Doc
m.erq.cccnt.cn/55311.Doc
m.erq.cccnt.cn/13579.Doc
m.erq.cccnt.cn/64240.Doc
m.erq.cccnt.cn/13777.Doc
m.erq.cccnt.cn/62022.Doc
m.erq.cccnt.cn/59915.Doc
m.erq.cccnt.cn/11591.Doc
m.erq.cccnt.cn/88044.Doc
m.erq.cccnt.cn/68406.Doc
m.erq.cccnt.cn/99137.Doc
m.erq.cccnt.cn/75157.Doc
m.erq.cccnt.cn/00406.Doc
m.erq.cccnt.cn/88422.Doc
m.erq.cccnt.cn/59753.Doc
m.erq.cccnt.cn/06428.Doc
m.erq.cccnt.cn/88428.Doc
m.eem.cccnt.cn/37359.Doc
m.eem.cccnt.cn/75559.Doc
m.eem.cccnt.cn/35577.Doc
m.eem.cccnt.cn/55193.Doc
m.eem.cccnt.cn/68082.Doc
m.eem.cccnt.cn/04268.Doc
m.eem.cccnt.cn/26684.Doc
m.eem.cccnt.cn/15771.Doc
m.eem.cccnt.cn/04266.Doc
m.eem.cccnt.cn/82082.Doc
m.eem.cccnt.cn/31373.Doc
m.eem.cccnt.cn/08200.Doc
m.eem.cccnt.cn/82268.Doc
m.eem.cccnt.cn/68424.Doc
m.eem.cccnt.cn/33311.Doc
m.eem.cccnt.cn/26660.Doc
m.eem.cccnt.cn/08446.Doc
m.eem.cccnt.cn/40280.Doc
m.eem.cccnt.cn/91597.Doc
m.eem.cccnt.cn/55533.Doc
m.een.cccnt.cn/42640.Doc
m.een.cccnt.cn/17735.Doc
m.een.cccnt.cn/31735.Doc
m.een.cccnt.cn/31713.Doc
m.een.cccnt.cn/08240.Doc
m.een.cccnt.cn/13133.Doc
m.een.cccnt.cn/84680.Doc
m.een.cccnt.cn/51579.Doc
m.een.cccnt.cn/64662.Doc
m.een.cccnt.cn/60048.Doc
m.een.cccnt.cn/46208.Doc
m.een.cccnt.cn/17591.Doc
m.een.cccnt.cn/04040.Doc
m.een.cccnt.cn/75595.Doc
m.een.cccnt.cn/44826.Doc
m.een.cccnt.cn/39531.Doc
m.een.cccnt.cn/57317.Doc
m.een.cccnt.cn/64402.Doc
m.een.cccnt.cn/37513.Doc
m.een.cccnt.cn/68204.Doc
m.eeb.cccnt.cn/80002.Doc
m.eeb.cccnt.cn/42244.Doc
m.eeb.cccnt.cn/84486.Doc
m.eeb.cccnt.cn/80462.Doc
m.eeb.cccnt.cn/22800.Doc
m.eeb.cccnt.cn/11595.Doc
m.eeb.cccnt.cn/33795.Doc
m.eeb.cccnt.cn/77539.Doc
m.eeb.cccnt.cn/93531.Doc
m.eeb.cccnt.cn/15133.Doc
m.eeb.cccnt.cn/33999.Doc
m.eeb.cccnt.cn/60268.Doc
m.eeb.cccnt.cn/19799.Doc
m.eeb.cccnt.cn/13711.Doc
m.eeb.cccnt.cn/15333.Doc
m.eeb.cccnt.cn/31399.Doc
m.eeb.cccnt.cn/57779.Doc
m.eeb.cccnt.cn/17759.Doc
m.eeb.cccnt.cn/15793.Doc
m.eeb.cccnt.cn/82404.Doc
m.eev.cccnt.cn/19159.Doc
m.eev.cccnt.cn/37197.Doc
m.eev.cccnt.cn/84028.Doc
m.eev.cccnt.cn/48044.Doc
m.eev.cccnt.cn/17935.Doc
m.eev.cccnt.cn/88444.Doc
m.eev.cccnt.cn/79137.Doc
m.eev.cccnt.cn/06424.Doc
m.eev.cccnt.cn/93351.Doc
m.eev.cccnt.cn/59151.Doc
m.eev.cccnt.cn/39355.Doc
m.eev.cccnt.cn/58248.Doc
m.eev.cccnt.cn/97113.Doc
m.eev.cccnt.cn/42482.Doc
m.eev.cccnt.cn/62008.Doc
m.eev.cccnt.cn/22828.Doc
m.eev.cccnt.cn/60842.Doc
m.eev.cccnt.cn/00688.Doc
m.eev.cccnt.cn/55913.Doc
m.eev.cccnt.cn/35739.Doc
m.eec.cccnt.cn/13597.Doc
m.eec.cccnt.cn/57157.Doc
m.eec.cccnt.cn/15571.Doc
m.eec.cccnt.cn/97175.Doc
m.eec.cccnt.cn/28062.Doc
m.eec.cccnt.cn/40608.Doc
m.eec.cccnt.cn/46826.Doc
m.eec.cccnt.cn/24604.Doc
m.eec.cccnt.cn/04646.Doc
m.eec.cccnt.cn/62420.Doc
m.eec.cccnt.cn/22600.Doc
m.eec.cccnt.cn/86626.Doc
m.eec.cccnt.cn/55195.Doc
m.eec.cccnt.cn/97777.Doc
m.eec.cccnt.cn/04602.Doc
m.eec.cccnt.cn/86088.Doc
m.eec.cccnt.cn/20020.Doc
m.eec.cccnt.cn/02200.Doc
m.eec.cccnt.cn/40266.Doc
m.eec.cccnt.cn/93551.Doc
m.eex.cccnt.cn/04208.Doc
m.eex.cccnt.cn/42088.Doc
m.eex.cccnt.cn/71553.Doc
m.eex.cccnt.cn/79971.Doc
m.eex.cccnt.cn/80266.Doc
m.eex.cccnt.cn/33795.Doc
m.eex.cccnt.cn/48864.Doc
m.eex.cccnt.cn/13557.Doc
m.eex.cccnt.cn/62686.Doc
m.eex.cccnt.cn/53711.Doc
m.eex.cccnt.cn/13959.Doc
m.eex.cccnt.cn/39577.Doc
m.eex.cccnt.cn/77973.Doc
m.eex.cccnt.cn/64468.Doc
m.eex.cccnt.cn/53599.Doc
m.eex.cccnt.cn/22064.Doc
m.eex.cccnt.cn/95599.Doc
m.eex.cccnt.cn/20680.Doc
m.eex.cccnt.cn/57353.Doc
m.eex.cccnt.cn/15933.Doc
m.eez.cccnt.cn/39175.Doc
m.eez.cccnt.cn/26684.Doc
m.eez.cccnt.cn/06226.Doc
m.eez.cccnt.cn/40604.Doc
m.eez.cccnt.cn/71397.Doc
m.eez.cccnt.cn/39173.Doc
m.eez.cccnt.cn/82824.Doc
m.eez.cccnt.cn/33115.Doc
m.eez.cccnt.cn/79993.Doc
m.eez.cccnt.cn/82088.Doc
m.eez.cccnt.cn/88248.Doc
m.eez.cccnt.cn/40004.Doc
m.eez.cccnt.cn/59515.Doc
m.eez.cccnt.cn/13959.Doc
m.eez.cccnt.cn/66022.Doc
m.eez.cccnt.cn/37795.Doc
m.eez.cccnt.cn/02020.Doc
m.eez.cccnt.cn/91353.Doc
m.eez.cccnt.cn/19715.Doc
m.eez.cccnt.cn/48446.Doc
m.eel.cccnt.cn/86824.Doc
m.eel.cccnt.cn/88466.Doc
m.eel.cccnt.cn/82282.Doc
m.eel.cccnt.cn/97999.Doc
m.eel.cccnt.cn/46026.Doc
m.eel.cccnt.cn/68880.Doc
m.eel.cccnt.cn/39395.Doc
m.eel.cccnt.cn/48008.Doc
m.eel.cccnt.cn/51357.Doc
m.eel.cccnt.cn/91195.Doc
m.eel.cccnt.cn/57319.Doc
m.eel.cccnt.cn/33951.Doc
m.eel.cccnt.cn/84608.Doc
m.eel.cccnt.cn/51559.Doc
m.eel.cccnt.cn/28668.Doc
m.eel.cccnt.cn/17771.Doc
m.eel.cccnt.cn/20622.Doc
m.eel.cccnt.cn/19317.Doc
m.eel.cccnt.cn/97575.Doc
m.eel.cccnt.cn/00642.Doc
m.eek.cccnt.cn/59993.Doc
m.eek.cccnt.cn/04484.Doc
m.eek.cccnt.cn/08466.Doc
m.eek.cccnt.cn/37799.Doc
m.eek.cccnt.cn/00220.Doc
m.eek.cccnt.cn/08206.Doc
m.eek.cccnt.cn/97171.Doc
m.eek.cccnt.cn/06286.Doc
m.eek.cccnt.cn/68086.Doc
m.eek.cccnt.cn/60288.Doc
m.eek.cccnt.cn/80006.Doc
m.eek.cccnt.cn/51977.Doc
m.eek.cccnt.cn/84642.Doc
m.eek.cccnt.cn/44206.Doc
m.eek.cccnt.cn/08602.Doc
m.eek.cccnt.cn/55715.Doc
m.eek.cccnt.cn/06082.Doc
m.eek.cccnt.cn/59739.Doc
m.eek.cccnt.cn/93391.Doc
m.eek.cccnt.cn/40280.Doc
m.eej.cccnt.cn/82288.Doc
m.eej.cccnt.cn/39519.Doc
m.eej.cccnt.cn/37793.Doc
m.eej.cccnt.cn/06004.Doc
m.eej.cccnt.cn/86682.Doc
m.eej.cccnt.cn/77519.Doc
m.eej.cccnt.cn/00824.Doc
m.eej.cccnt.cn/02668.Doc
m.eej.cccnt.cn/59313.Doc
m.eej.cccnt.cn/75313.Doc
m.eej.cccnt.cn/33111.Doc
m.eej.cccnt.cn/44842.Doc
m.eej.cccnt.cn/00862.Doc
m.eej.cccnt.cn/93335.Doc
m.eej.cccnt.cn/57959.Doc
m.eej.cccnt.cn/57933.Doc
m.eej.cccnt.cn/33777.Doc
m.eej.cccnt.cn/57975.Doc
m.eej.cccnt.cn/62840.Doc
m.eej.cccnt.cn/91557.Doc
m.eeh.cccnt.cn/64282.Doc
m.eeh.cccnt.cn/20884.Doc
m.eeh.cccnt.cn/55359.Doc
m.eeh.cccnt.cn/75191.Doc
m.eeh.cccnt.cn/48060.Doc
m.eeh.cccnt.cn/77197.Doc
m.eeh.cccnt.cn/66226.Doc
m.eeh.cccnt.cn/13991.Doc
m.eeh.cccnt.cn/33153.Doc
m.eeh.cccnt.cn/75537.Doc
m.eeh.cccnt.cn/59911.Doc
m.eeh.cccnt.cn/60400.Doc
m.eeh.cccnt.cn/79559.Doc
m.eeh.cccnt.cn/11959.Doc
m.eeh.cccnt.cn/71573.Doc
m.eeh.cccnt.cn/88460.Doc
m.eeh.cccnt.cn/79919.Doc
m.eeh.cccnt.cn/91953.Doc
m.eeh.cccnt.cn/15559.Doc
m.eeh.cccnt.cn/00404.Doc
m.eeg.cccnt.cn/57319.Doc
m.eeg.cccnt.cn/02262.Doc
m.eeg.cccnt.cn/11971.Doc
m.eeg.cccnt.cn/00206.Doc
m.eeg.cccnt.cn/13517.Doc
m.eeg.cccnt.cn/39117.Doc
m.eeg.cccnt.cn/88666.Doc
m.eeg.cccnt.cn/08084.Doc
m.eeg.cccnt.cn/13779.Doc
m.eeg.cccnt.cn/04088.Doc
m.eeg.cccnt.cn/33591.Doc
m.eeg.cccnt.cn/51371.Doc
m.eeg.cccnt.cn/37973.Doc
m.eeg.cccnt.cn/13311.Doc
m.eeg.cccnt.cn/46084.Doc
m.eeg.cccnt.cn/24808.Doc
m.eeg.cccnt.cn/37397.Doc
m.eeg.cccnt.cn/59911.Doc
m.eeg.cccnt.cn/17535.Doc
m.eeg.cccnt.cn/33137.Doc
m.eef.cccnt.cn/15119.Doc
m.eef.cccnt.cn/06402.Doc
m.eef.cccnt.cn/06646.Doc
m.eef.cccnt.cn/11193.Doc
m.eef.cccnt.cn/93379.Doc
m.eef.cccnt.cn/66844.Doc
m.eef.cccnt.cn/57935.Doc
m.eef.cccnt.cn/55599.Doc
m.eef.cccnt.cn/84668.Doc
m.eef.cccnt.cn/71993.Doc
m.eef.cccnt.cn/51131.Doc
m.eef.cccnt.cn/75939.Doc
m.eef.cccnt.cn/11515.Doc
m.eef.cccnt.cn/51717.Doc
m.eef.cccnt.cn/71195.Doc
m.eef.cccnt.cn/73553.Doc
m.eef.cccnt.cn/97335.Doc
m.eef.cccnt.cn/46282.Doc
m.eef.cccnt.cn/37377.Doc
m.eef.cccnt.cn/55735.Doc
m.eed.cccnt.cn/17917.Doc
m.eed.cccnt.cn/93377.Doc
m.eed.cccnt.cn/79571.Doc
m.eed.cccnt.cn/82248.Doc
m.eed.cccnt.cn/51375.Doc
m.eed.cccnt.cn/35779.Doc
m.eed.cccnt.cn/57711.Doc
m.eed.cccnt.cn/51151.Doc
m.eed.cccnt.cn/55539.Doc
m.eed.cccnt.cn/08466.Doc
m.eed.cccnt.cn/15937.Doc
m.eed.cccnt.cn/40622.Doc
m.eed.cccnt.cn/60282.Doc
m.eed.cccnt.cn/97573.Doc
m.eed.cccnt.cn/55335.Doc
m.eed.cccnt.cn/35397.Doc
m.eed.cccnt.cn/73391.Doc
m.eed.cccnt.cn/15199.Doc
m.eed.cccnt.cn/31157.Doc
m.eed.cccnt.cn/77353.Doc
m.ees.cccnt.cn/11597.Doc
m.ees.cccnt.cn/66628.Doc
m.ees.cccnt.cn/35915.Doc
m.ees.cccnt.cn/77353.Doc
m.ees.cccnt.cn/79317.Doc
m.ees.cccnt.cn/19311.Doc
m.ees.cccnt.cn/73975.Doc
m.ees.cccnt.cn/95999.Doc
m.ees.cccnt.cn/51179.Doc
m.ees.cccnt.cn/28888.Doc
m.ees.cccnt.cn/31793.Doc
m.ees.cccnt.cn/82044.Doc
m.ees.cccnt.cn/51511.Doc
m.ees.cccnt.cn/71139.Doc
m.ees.cccnt.cn/04042.Doc
m.ees.cccnt.cn/80286.Doc
m.ees.cccnt.cn/19555.Doc
m.ees.cccnt.cn/73793.Doc
m.ees.cccnt.cn/75135.Doc
m.ees.cccnt.cn/68808.Doc
m.eea.cccnt.cn/82020.Doc
m.eea.cccnt.cn/88260.Doc
m.eea.cccnt.cn/57759.Doc
m.eea.cccnt.cn/15119.Doc
m.eea.cccnt.cn/44040.Doc
m.eea.cccnt.cn/75371.Doc
m.eea.cccnt.cn/46662.Doc
m.eea.cccnt.cn/71171.Doc
m.eea.cccnt.cn/33975.Doc
m.eea.cccnt.cn/02806.Doc
m.eea.cccnt.cn/40004.Doc
m.eea.cccnt.cn/04648.Doc
m.eea.cccnt.cn/13531.Doc
m.eea.cccnt.cn/19597.Doc
m.eea.cccnt.cn/53355.Doc
m.eea.cccnt.cn/75917.Doc
m.eea.cccnt.cn/00080.Doc
m.eea.cccnt.cn/51933.Doc
m.eea.cccnt.cn/46046.Doc
m.eea.cccnt.cn/51513.Doc
m.eep.cccnt.cn/77575.Doc
m.eep.cccnt.cn/33153.Doc
m.eep.cccnt.cn/71137.Doc
m.eep.cccnt.cn/19799.Doc
m.eep.cccnt.cn/48846.Doc
m.eep.cccnt.cn/06484.Doc
m.eep.cccnt.cn/88284.Doc
m.eep.cccnt.cn/55515.Doc
m.eep.cccnt.cn/08688.Doc
m.eep.cccnt.cn/06846.Doc
m.eep.cccnt.cn/37339.Doc
m.eep.cccnt.cn/66428.Doc
m.eep.cccnt.cn/86680.Doc
m.eep.cccnt.cn/31375.Doc
m.eep.cccnt.cn/86022.Doc
m.eep.cccnt.cn/48684.Doc
m.eep.cccnt.cn/79357.Doc
m.eep.cccnt.cn/75791.Doc
m.eep.cccnt.cn/79519.Doc
m.eep.cccnt.cn/19791.Doc
m.eeo.cccnt.cn/33513.Doc
m.eeo.cccnt.cn/35937.Doc
m.eeo.cccnt.cn/11751.Doc
m.eeo.cccnt.cn/91739.Doc
m.eeo.cccnt.cn/48844.Doc
m.eeo.cccnt.cn/84628.Doc
m.eeo.cccnt.cn/19531.Doc
m.eeo.cccnt.cn/40844.Doc
m.eeo.cccnt.cn/82040.Doc
m.eeo.cccnt.cn/19555.Doc
m.eeo.cccnt.cn/66208.Doc
m.eeo.cccnt.cn/51531.Doc
m.eeo.cccnt.cn/64086.Doc
m.eeo.cccnt.cn/13593.Doc
m.eeo.cccnt.cn/31395.Doc
m.eeo.cccnt.cn/06202.Doc
m.eeo.cccnt.cn/11179.Doc
m.eeo.cccnt.cn/75179.Doc
m.eeo.cccnt.cn/59139.Doc
m.eeo.cccnt.cn/22068.Doc
m.eei.cccnt.cn/79779.Doc
m.eei.cccnt.cn/13791.Doc
m.eei.cccnt.cn/39151.Doc
m.eei.cccnt.cn/73359.Doc
m.eei.cccnt.cn/77737.Doc
