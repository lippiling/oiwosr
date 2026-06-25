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

wjc.irampL.cn/62228.Doc
wjc.irampL.cn/28224.Doc
wjc.irampL.cn/84248.Doc
wjc.irampL.cn/88002.Doc
wjc.irampL.cn/40444.Doc
wjc.irampL.cn/02086.Doc
wjc.irampL.cn/06602.Doc
wjc.irampL.cn/02240.Doc
wjc.irampL.cn/08608.Doc
wjc.irampL.cn/26440.Doc
wjx.irampL.cn/46044.Doc
wjx.irampL.cn/48666.Doc
wjx.irampL.cn/39531.Doc
wjx.irampL.cn/88442.Doc
wjx.irampL.cn/68462.Doc
wjx.irampL.cn/46448.Doc
wjx.irampL.cn/22860.Doc
wjx.irampL.cn/08600.Doc
wjx.irampL.cn/00040.Doc
wjx.irampL.cn/44424.Doc
wjz.irampL.cn/40088.Doc
wjz.irampL.cn/22206.Doc
wjz.irampL.cn/08626.Doc
wjz.irampL.cn/62404.Doc
wjz.irampL.cn/26464.Doc
wjz.irampL.cn/26000.Doc
wjz.irampL.cn/00280.Doc
wjz.irampL.cn/35151.Doc
wjz.irampL.cn/64808.Doc
wjz.irampL.cn/24044.Doc
wjl.irampL.cn/42020.Doc
wjl.irampL.cn/13319.Doc
wjl.irampL.cn/20880.Doc
wjl.irampL.cn/46484.Doc
wjl.irampL.cn/60028.Doc
wjl.irampL.cn/80424.Doc
wjl.irampL.cn/86466.Doc
wjl.irampL.cn/06004.Doc
wjl.irampL.cn/80800.Doc
wjl.irampL.cn/06280.Doc
wjk.irampL.cn/71159.Doc
wjk.irampL.cn/46688.Doc
wjk.irampL.cn/66262.Doc
wjk.irampL.cn/48226.Doc
wjk.irampL.cn/42864.Doc
wjk.irampL.cn/28620.Doc
wjk.irampL.cn/00006.Doc
wjk.irampL.cn/84286.Doc
wjk.irampL.cn/26822.Doc
wjk.irampL.cn/57771.Doc
wjj.irampL.cn/97953.Doc
wjj.irampL.cn/20808.Doc
wjj.irampL.cn/99935.Doc
wjj.irampL.cn/88000.Doc
wjj.irampL.cn/44444.Doc
wjj.irampL.cn/22880.Doc
wjj.irampL.cn/22664.Doc
wjj.irampL.cn/22600.Doc
wjj.irampL.cn/06444.Doc
wjj.irampL.cn/40266.Doc
wjh.irampL.cn/60240.Doc
wjh.irampL.cn/84884.Doc
wjh.irampL.cn/64428.Doc
wjh.irampL.cn/00842.Doc
wjh.irampL.cn/44482.Doc
wjh.irampL.cn/44000.Doc
wjh.irampL.cn/08682.Doc
wjh.irampL.cn/04600.Doc
wjh.irampL.cn/33937.Doc
wjh.irampL.cn/80806.Doc
wjg.irampL.cn/04284.Doc
wjg.irampL.cn/68224.Doc
wjg.irampL.cn/80462.Doc
wjg.irampL.cn/48682.Doc
wjg.irampL.cn/20406.Doc
wjg.irampL.cn/51795.Doc
wjg.irampL.cn/44008.Doc
wjg.irampL.cn/82444.Doc
wjg.irampL.cn/88002.Doc
wjg.irampL.cn/06280.Doc
wjf.irampL.cn/00688.Doc
wjf.irampL.cn/93915.Doc
wjf.irampL.cn/64204.Doc
wjf.irampL.cn/68826.Doc
wjf.irampL.cn/64862.Doc
wjf.irampL.cn/64226.Doc
wjf.irampL.cn/82424.Doc
wjf.irampL.cn/00444.Doc
wjf.irampL.cn/44406.Doc
wjf.irampL.cn/22466.Doc
wjd.irampL.cn/08442.Doc
wjd.irampL.cn/53717.Doc
wjd.irampL.cn/40886.Doc
wjd.irampL.cn/99157.Doc
wjd.irampL.cn/02068.Doc
wjd.irampL.cn/68468.Doc
wjd.irampL.cn/26666.Doc
wjd.irampL.cn/46464.Doc
wjd.irampL.cn/44464.Doc
wjd.irampL.cn/40026.Doc
wjs.irampL.cn/84820.Doc
wjs.irampL.cn/62204.Doc
wjs.irampL.cn/20468.Doc
wjs.irampL.cn/46200.Doc
wjs.irampL.cn/66008.Doc
wjs.irampL.cn/06024.Doc
wjs.irampL.cn/20400.Doc
wjs.irampL.cn/00460.Doc
wjs.irampL.cn/42224.Doc
wjs.irampL.cn/06046.Doc
wja.irampL.cn/62046.Doc
wja.irampL.cn/46042.Doc
wja.irampL.cn/82024.Doc
wja.irampL.cn/00688.Doc
wja.irampL.cn/28040.Doc
wja.irampL.cn/28802.Doc
wja.irampL.cn/28426.Doc
wja.irampL.cn/95571.Doc
wja.irampL.cn/66682.Doc
wja.irampL.cn/88664.Doc
wjp.irampL.cn/46420.Doc
wjp.irampL.cn/42002.Doc
wjp.irampL.cn/84888.Doc
wjp.irampL.cn/82844.Doc
wjp.irampL.cn/20800.Doc
wjp.irampL.cn/19339.Doc
wjp.irampL.cn/42488.Doc
wjp.irampL.cn/64406.Doc
wjp.irampL.cn/06488.Doc
wjp.irampL.cn/84026.Doc
wjo.irampL.cn/80082.Doc
wjo.irampL.cn/82448.Doc
wjo.irampL.cn/91535.Doc
wjo.irampL.cn/48206.Doc
wjo.irampL.cn/62640.Doc
wjo.irampL.cn/00046.Doc
wjo.irampL.cn/71553.Doc
wjo.irampL.cn/13339.Doc
wjo.irampL.cn/88082.Doc
wjo.irampL.cn/53537.Doc
wji.irampL.cn/20820.Doc
wji.irampL.cn/28242.Doc
wji.irampL.cn/64008.Doc
wji.irampL.cn/73131.Doc
wji.irampL.cn/77157.Doc
wji.irampL.cn/40064.Doc
wji.irampL.cn/64666.Doc
wji.irampL.cn/79539.Doc
wji.irampL.cn/04206.Doc
wji.irampL.cn/97515.Doc
wju.irampL.cn/06862.Doc
wju.irampL.cn/02668.Doc
wju.irampL.cn/62626.Doc
wju.irampL.cn/24004.Doc
wju.irampL.cn/57171.Doc
wju.irampL.cn/62226.Doc
wju.irampL.cn/71757.Doc
wju.irampL.cn/60268.Doc
wju.irampL.cn/06428.Doc
wju.irampL.cn/20208.Doc
wjy.irampL.cn/22420.Doc
wjy.irampL.cn/93773.Doc
wjy.irampL.cn/24008.Doc
wjy.irampL.cn/33375.Doc
wjy.irampL.cn/40624.Doc
wjy.irampL.cn/66460.Doc
wjy.irampL.cn/04288.Doc
wjy.irampL.cn/24604.Doc
wjy.irampL.cn/86244.Doc
wjy.irampL.cn/60204.Doc
wjt.irampL.cn/60842.Doc
wjt.irampL.cn/64262.Doc
wjt.irampL.cn/82284.Doc
wjt.irampL.cn/82008.Doc
wjt.irampL.cn/44408.Doc
wjt.irampL.cn/84000.Doc
wjt.irampL.cn/28242.Doc
wjt.irampL.cn/42402.Doc
wjt.irampL.cn/88640.Doc
wjt.irampL.cn/46248.Doc
wjr.irampL.cn/06604.Doc
wjr.irampL.cn/91333.Doc
wjr.irampL.cn/28428.Doc
wjr.irampL.cn/40842.Doc
wjr.irampL.cn/80684.Doc
wjr.irampL.cn/40488.Doc
wjr.irampL.cn/08006.Doc
wjr.irampL.cn/24622.Doc
wjr.irampL.cn/04866.Doc
wjr.irampL.cn/66464.Doc
wje.irampL.cn/40426.Doc
wje.irampL.cn/44402.Doc
wje.irampL.cn/86422.Doc
wje.irampL.cn/04608.Doc
wje.irampL.cn/82002.Doc
wje.irampL.cn/68420.Doc
wje.irampL.cn/88842.Doc
wje.irampL.cn/84406.Doc
wje.irampL.cn/24024.Doc
wje.irampL.cn/13557.Doc
wjw.irampL.cn/48048.Doc
wjw.irampL.cn/62802.Doc
wjw.irampL.cn/80848.Doc
wjw.irampL.cn/64424.Doc
wjw.irampL.cn/80224.Doc
wjw.irampL.cn/00646.Doc
wjw.irampL.cn/88062.Doc
wjw.irampL.cn/46448.Doc
wjw.irampL.cn/48202.Doc
wjw.irampL.cn/08844.Doc
wjq.irampL.cn/00266.Doc
wjq.irampL.cn/40048.Doc
wjq.irampL.cn/86840.Doc
wjq.irampL.cn/40020.Doc
wjq.irampL.cn/24222.Doc
wjq.irampL.cn/62622.Doc
wjq.irampL.cn/86608.Doc
wjq.irampL.cn/48088.Doc
wjq.irampL.cn/00624.Doc
wjq.irampL.cn/84064.Doc
whm.irampL.cn/55551.Doc
whm.irampL.cn/24828.Doc
whm.irampL.cn/48664.Doc
whm.irampL.cn/60066.Doc
whm.irampL.cn/68228.Doc
whm.irampL.cn/84462.Doc
whm.irampL.cn/39319.Doc
whm.irampL.cn/68008.Doc
whm.irampL.cn/06046.Doc
whm.irampL.cn/86820.Doc
whn.irampL.cn/80604.Doc
whn.irampL.cn/13159.Doc
whn.irampL.cn/40842.Doc
whn.irampL.cn/80884.Doc
whn.irampL.cn/68824.Doc
whn.irampL.cn/66640.Doc
whn.irampL.cn/28444.Doc
whn.irampL.cn/42042.Doc
whn.irampL.cn/28082.Doc
whn.irampL.cn/82864.Doc
whb.irampL.cn/66282.Doc
whb.irampL.cn/13131.Doc
whb.irampL.cn/42224.Doc
whb.irampL.cn/64086.Doc
whb.irampL.cn/00266.Doc
whb.irampL.cn/28664.Doc
whb.irampL.cn/53375.Doc
whb.irampL.cn/91353.Doc
whb.irampL.cn/99797.Doc
whb.irampL.cn/86664.Doc
whv.irampL.cn/51535.Doc
whv.irampL.cn/24864.Doc
whv.irampL.cn/88804.Doc
whv.irampL.cn/62022.Doc
whv.irampL.cn/06426.Doc
whv.irampL.cn/88228.Doc
whv.irampL.cn/66064.Doc
whv.irampL.cn/46824.Doc
whv.irampL.cn/88080.Doc
whv.irampL.cn/42862.Doc
whc.irampL.cn/68864.Doc
whc.irampL.cn/48864.Doc
whc.irampL.cn/86248.Doc
whc.irampL.cn/88226.Doc
whc.irampL.cn/95775.Doc
whc.irampL.cn/46820.Doc
whc.irampL.cn/28806.Doc
whc.irampL.cn/66486.Doc
whc.irampL.cn/68804.Doc
whc.irampL.cn/02804.Doc
whx.irampL.cn/24042.Doc
whx.irampL.cn/19175.Doc
whx.irampL.cn/37519.Doc
whx.irampL.cn/35184.Doc
whx.irampL.cn/04242.Doc
whx.irampL.cn/24486.Doc
whx.irampL.cn/06880.Doc
whx.irampL.cn/80604.Doc
whx.irampL.cn/66240.Doc
whx.irampL.cn/24446.Doc
whz.irampL.cn/00862.Doc
whz.irampL.cn/68462.Doc
whz.irampL.cn/00604.Doc
whz.irampL.cn/17159.Doc
whz.irampL.cn/80466.Doc
whz.irampL.cn/04008.Doc
whz.irampL.cn/20084.Doc
whz.irampL.cn/48642.Doc
whz.irampL.cn/48402.Doc
whz.irampL.cn/68244.Doc
whl.irampL.cn/26280.Doc
whl.irampL.cn/20426.Doc
whl.irampL.cn/06648.Doc
whl.irampL.cn/86268.Doc
whl.irampL.cn/48028.Doc
whl.irampL.cn/44440.Doc
whl.irampL.cn/91735.Doc
whl.irampL.cn/40288.Doc
whl.irampL.cn/06262.Doc
whl.irampL.cn/97551.Doc
whk.irampL.cn/04840.Doc
whk.irampL.cn/44648.Doc
whk.irampL.cn/42244.Doc
whk.irampL.cn/88208.Doc
whk.irampL.cn/62806.Doc
whk.irampL.cn/82686.Doc
whk.irampL.cn/04220.Doc
whk.irampL.cn/88646.Doc
whk.irampL.cn/26628.Doc
whk.irampL.cn/48046.Doc
whj.irampL.cn/86288.Doc
whj.irampL.cn/86006.Doc
whj.irampL.cn/55173.Doc
whj.irampL.cn/97979.Doc
whj.irampL.cn/60420.Doc
whj.irampL.cn/71755.Doc
whj.irampL.cn/06028.Doc
whj.irampL.cn/64662.Doc
whj.irampL.cn/24040.Doc
whj.irampL.cn/68444.Doc
whh.irampL.cn/06280.Doc
whh.irampL.cn/80608.Doc
whh.irampL.cn/00804.Doc
whh.irampL.cn/04824.Doc
whh.irampL.cn/35551.Doc
whh.irampL.cn/15917.Doc
whh.irampL.cn/08862.Doc
whh.irampL.cn/64882.Doc
whh.irampL.cn/91917.Doc
whh.irampL.cn/15717.Doc
whg.irampL.cn/80042.Doc
whg.irampL.cn/68886.Doc
whg.irampL.cn/66266.Doc
whg.irampL.cn/64246.Doc
whg.irampL.cn/48800.Doc
whg.irampL.cn/44646.Doc
whg.irampL.cn/46246.Doc
whg.irampL.cn/48668.Doc
whg.irampL.cn/62024.Doc
whg.irampL.cn/59995.Doc
whf.irampL.cn/57953.Doc
whf.irampL.cn/40620.Doc
whf.irampL.cn/40666.Doc
whf.irampL.cn/28284.Doc
whf.irampL.cn/24440.Doc
whf.irampL.cn/08224.Doc
whf.irampL.cn/68266.Doc
whf.irampL.cn/68462.Doc
whf.irampL.cn/42042.Doc
whf.irampL.cn/26680.Doc
whd.irampL.cn/40808.Doc
whd.irampL.cn/80286.Doc
whd.irampL.cn/60822.Doc
whd.irampL.cn/00008.Doc
whd.irampL.cn/88482.Doc
whd.irampL.cn/06046.Doc
whd.irampL.cn/00406.Doc
whd.irampL.cn/22468.Doc
whd.irampL.cn/44044.Doc
whd.irampL.cn/42446.Doc
whs.irampL.cn/22262.Doc
whs.irampL.cn/68008.Doc
whs.irampL.cn/22488.Doc
whs.irampL.cn/00228.Doc
whs.irampL.cn/66242.Doc
whs.irampL.cn/60280.Doc
whs.irampL.cn/95571.Doc
whs.irampL.cn/64640.Doc
whs.irampL.cn/62844.Doc
whs.irampL.cn/28880.Doc
wha.irampL.cn/06464.Doc
wha.irampL.cn/44228.Doc
wha.irampL.cn/77391.Doc
wha.irampL.cn/40886.Doc
wha.irampL.cn/48402.Doc
wha.irampL.cn/88066.Doc
wha.irampL.cn/26088.Doc
wha.irampL.cn/00404.Doc
wha.irampL.cn/80808.Doc
wha.irampL.cn/00488.Doc
whp.irampL.cn/20420.Doc
whp.irampL.cn/17753.Doc
whp.irampL.cn/88846.Doc
whp.irampL.cn/84044.Doc
whp.irampL.cn/24826.Doc
whp.irampL.cn/06222.Doc
whp.irampL.cn/28660.Doc
whp.irampL.cn/08042.Doc
whp.irampL.cn/08426.Doc
whp.irampL.cn/82600.Doc
who.irampL.cn/26246.Doc
who.irampL.cn/24686.Doc
who.irampL.cn/60466.Doc
who.irampL.cn/44022.Doc
who.irampL.cn/20884.Doc
who.irampL.cn/33599.Doc
who.irampL.cn/66286.Doc
who.irampL.cn/66400.Doc
who.irampL.cn/13911.Doc
who.irampL.cn/44622.Doc
whi.irampL.cn/06622.Doc
whi.irampL.cn/48444.Doc
whi.irampL.cn/88024.Doc
whi.irampL.cn/06840.Doc
whi.irampL.cn/68468.Doc
whi.irampL.cn/02206.Doc
whi.irampL.cn/28088.Doc
whi.irampL.cn/24204.Doc
whi.irampL.cn/00242.Doc
whi.irampL.cn/80824.Doc
whu.irampL.cn/20660.Doc
whu.irampL.cn/46826.Doc
whu.irampL.cn/62068.Doc
whu.irampL.cn/86428.Doc
whu.irampL.cn/00800.Doc
whu.irampL.cn/62262.Doc
whu.irampL.cn/82448.Doc
whu.irampL.cn/62280.Doc
whu.irampL.cn/44022.Doc
whu.irampL.cn/68808.Doc
why.irampL.cn/64646.Doc
why.irampL.cn/42282.Doc
why.irampL.cn/19133.Doc
why.irampL.cn/04846.Doc
why.irampL.cn/64268.Doc
why.irampL.cn/37795.Doc
why.irampL.cn/55771.Doc
why.irampL.cn/48688.Doc
why.irampL.cn/11993.Doc
why.irampL.cn/02028.Doc
wht.irampL.cn/06028.Doc
wht.irampL.cn/46248.Doc
wht.irampL.cn/84220.Doc
wht.irampL.cn/64040.Doc
wht.irampL.cn/02808.Doc
wht.irampL.cn/08480.Doc
wht.irampL.cn/64606.Doc
wht.irampL.cn/48426.Doc
wht.irampL.cn/26886.Doc
wht.irampL.cn/06620.Doc
whr.irampL.cn/86204.Doc
whr.irampL.cn/86626.Doc
whr.irampL.cn/26060.Doc
whr.irampL.cn/06288.Doc
whr.irampL.cn/68404.Doc
whr.irampL.cn/46082.Doc
whr.irampL.cn/40860.Doc
whr.irampL.cn/51373.Doc
whr.irampL.cn/80868.Doc
whr.irampL.cn/66600.Doc
whe.irampL.cn/02444.Doc
whe.irampL.cn/40808.Doc
whe.irampL.cn/73179.Doc
whe.irampL.cn/08464.Doc
whe.irampL.cn/42804.Doc
whe.irampL.cn/60404.Doc
whe.irampL.cn/82086.Doc
whe.irampL.cn/80886.Doc
whe.irampL.cn/48042.Doc
whe.irampL.cn/35571.Doc
whw.irampL.cn/80824.Doc
whw.irampL.cn/88080.Doc
whw.irampL.cn/55531.Doc
whw.irampL.cn/22844.Doc
whw.irampL.cn/22442.Doc
whw.irampL.cn/88840.Doc
whw.irampL.cn/66444.Doc
whw.irampL.cn/20020.Doc
whw.irampL.cn/02204.Doc
whw.irampL.cn/77773.Doc
whq.irampL.cn/04426.Doc
whq.irampL.cn/24060.Doc
whq.irampL.cn/44802.Doc
whq.irampL.cn/60846.Doc
whq.irampL.cn/33717.Doc
whq.irampL.cn/08222.Doc
whq.irampL.cn/82602.Doc
whq.irampL.cn/84400.Doc
whq.irampL.cn/40664.Doc
whq.irampL.cn/46680.Doc
wgm.irampL.cn/33139.Doc
wgm.irampL.cn/40462.Doc
wgm.irampL.cn/24246.Doc
wgm.irampL.cn/82844.Doc
wgm.irampL.cn/39319.Doc
wgm.irampL.cn/68602.Doc
wgm.irampL.cn/48840.Doc
wgm.irampL.cn/46646.Doc
wgm.irampL.cn/06688.Doc
wgm.irampL.cn/20220.Doc
