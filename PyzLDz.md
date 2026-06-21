壤胃粱饺撇


========================================
 Python 依赖注入与控制反转
========================================

一、什么是依赖注入与控制反转
----------------------------------------

控制反转（IoC）是一种设计原则，将组件创建和管理的控制权从组件内部转移到外部容器。
依赖注入（DI）是 IoC 的一种实现方式，通过外部传入依赖而非内部自行创建。

依赖注入的核心好处：
- 降低耦合度：组件不直接依赖具体实现
- 提升可测试性：可以轻松替换为 mock 对象
- 增强可维护性：依赖关系清晰可见


二、手动依赖注入（无框架）
----------------------------------------

# 不使用 DI 的传统方式——高耦合
class DatabaseConnection:
    def query(self, sql: str) -> list:
        return ["模拟数据1", "模拟数据2"]

class UserService:
    def __init__(self):
        # 硬编码创建依赖，难以替换
        self._db = DatabaseConnection()

    def get_users(self) -> list:
        return self._db.query("SELECT * FROM users")

# 使用构造函数注入——低耦合
class UserServiceDI:
    def __init__(self, db: DatabaseConnection):
        # 依赖由外部传入，而非内部创建
        self._db = db

    def get_users(self) -> list:
        return self._db.query("SELECT * FROM users")


三、使用 typing.Protocol 定义抽象依赖
----------------------------------------

from typing import Protocol, Any

# 定义抽象接口：任何拥有 query 方法的对象都符合
class Database(Protocol):
    def query(self, sql: str) -> list: ...

class ProductionDB:
    """生产环境数据库实现"""
    def query(self, sql: str) -> list:
        # 实际连接数据库执行 SQL
        return [{"id": 1, "name": "张三"}]

class FakeDB:
    """测试用伪实现"""
    def query(self, sql: str) -> list:
        return [{"id": 999, "name": "测试用户"}]

def create_user_service(db: Database) -> UserServiceDI:
    """工厂函数：显式组装依赖"""
    return UserServiceDI(db)


四、简易 DI 容器实现
----------------------------------------

class SimpleContainer:
    """一个极简的依赖注入容器"""

    def __init__(self):
        self._registry: dict[str, type] = {}
        self._instances: dict[str, object] = {}

    def register(self, name: str, cls: type) -> None:
        """注册类到容器"""
        self._registry[name] = cls

    def resolve(self, name: str) -> object:
        """从容器中解析依赖（自动注入）"""
        if name in self._instances:
            return self._instances[name]
        cls = self._registry[name]
        # 自动解析构造函数参数
        import inspect
        params = inspect.signature(cls.__init__).parameters
        deps = {}
        for param_name, param in params.items():
            if param_name == 'self':
                continue
            if param.annotation is not inspect.Parameter.empty:
                # 根据类型注解自动解析子依赖
                dep_name = param.annotation.__name__
                deps[param_name] = self.resolve(dep_name)
        instance = cls(**deps)
        self._instances[name] = instance
        return instance

    def clear(self) -> None:
        """清理所有实例（用于测试重置）"""
        self._instances.clear()

# 使用示例
container = SimpleContainer()
container.register("UserService", UserServiceDI)
container.register("Database", ProductionDB)
service = container.resolve("UserService")


五、FastAPI 中的 Depends
----------------------------------------

FastAPI 内置了强大的 DI 系统，通过 Depends 实现。

from fastapi import FastAPI, Depends, Header, HTTPException
from typing import Optional

app = FastAPI()

def get_db_connection():
    """数据库连接依赖（每次请求创建）"""
    db = ProductionDB()
    try:
        yield db
    finally:
        # 请求结束后清理资源
        pass

def verify_token(authorization: str = Header(...)) -> str:
    """验证令牌依赖"""
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="无效令牌")
    return authorization.split(" ")[1]

@app.get("/users")
def list_users(
    token: str = Depends(verify_token),  # 先验证令牌
    db: Database = Depends(get_db_connection)  # 再获取数据库连接
):
    """FastAPI 自动解析 Depends 链"""
    return {"users": db.query("SELECT * FROM users")}


六、unittest.mock.patch 实现测试 DI
----------------------------------------

即使代码没有使用 DI 模式，我们仍然可以在测试中通过 mock 来模拟依赖。

from unittest.mock import patch

def test_user_service():
    """展示如何对未使用 DI 的代码进行测试"""
    with patch("__main__.DatabaseConnection") as MockDB:
        # 配置 mock 行为
        mock_instance = MockDB.return_value
        mock_instance.query.return_value = ["模拟结果"]

        # 注入 mock 对象
        service = UserServiceDI(mock_instance)
        result = service.get_users()

        # 验证结果和行为
        assert result == ["模拟结果"]
        mock_instance.query.assert_called_once_with(
            "SELECT * FROM users"
        )


七、服务定位器模式 vs 依赖注入
----------------------------------------

# 服务定位器模式（反例）
class ServiceLocator:
    """全局服务定位器——隐藏了依赖关系"""
    _services: dict = {}

    @classmethod
    def register(cls, name: str, instance: object) -> None:
        cls._services[name] = instance

    @classmethod
    def get(cls, name: str) -> object:
        return cls._services[name]

# 使用服务定位器
class ReportGenerator:
    def generate(self) -> str:
        # 隐式依赖：使用时才去定位器查找
        db = ServiceLocator.get("Database")
        data = db.query("SELECT * FROM reports")
        return f"报告内容: {data}"

"""
服务定位器 vs DI 对比：
+------------------------+--------------------------+
| 服务定位器             | 依赖注入                 |
+------------------------+--------------------------+
| 隐式依赖，难以看出需求 | 显式依赖，一目了然       |
| 运行时可能失败         | 创建时保证依赖完整       |
| 测试时需配置定位器     | 测试时直接传入 mock      |
| 全局可变状态           | 无状态，纯参数传递       |
+------------------------+--------------------------+
"""


八、最佳实践总结
----------------------------------------

1. 对于简单项目，手动 DI（构造函数注入）足矣，无需引入框架
2. 使用 Protocol 定义接口，保持灵活性和类型安全
3. DI 容器适用于大型项目，但避免过度设计
4. FastAPI Depends 是 Python Web 框架中 DI 的最佳实践
5. 始终优先使用构造函数注入，而非服务定位器
6. 测试时用 mock.patch 补充而非替代 DI

新勾裂一币赘思永谜媒闻啄桌侗瞥

m.blog.lwsnr.cn/Article/details/2400080.htm
m.blog.lwsnr.cn/Article/details/0406404.htm
m.blog.lwsnr.cn/Article/details/1739599.htm
m.blog.lwsnr.cn/Article/details/7739997.htm
m.blog.lwsnr.cn/Article/details/4268688.htm
m.blog.lwsnr.cn/Article/details/9157197.htm
m.blog.lwsnr.cn/Article/details/3919999.htm
m.blog.lwsnr.cn/Article/details/6228668.htm
m.blog.lwsnr.cn/Article/details/8844442.htm
m.blog.lwsnr.cn/Article/details/7737735.htm
m.blog.lwsnr.cn/Article/details/8000886.htm
m.blog.lwsnr.cn/Article/details/4646248.htm
m.blog.lwsnr.cn/Article/details/7391935.htm
m.blog.lwsnr.cn/Article/details/5715513.htm
m.blog.lwsnr.cn/Article/details/1315193.htm
m.blog.lwsnr.cn/Article/details/1379951.htm
m.blog.lwsnr.cn/Article/details/2684460.htm
m.blog.lwsnr.cn/Article/details/0864888.htm
m.blog.lwsnr.cn/Article/details/3355333.htm
m.blog.lwsnr.cn/Article/details/8488846.htm
m.blog.lwsnr.cn/Article/details/1135773.htm
m.blog.lwsnr.cn/Article/details/9193357.htm
m.blog.lwsnr.cn/Article/details/8688002.htm
m.blog.lwsnr.cn/Article/details/3399595.htm
m.blog.lwsnr.cn/Article/details/9951515.htm
m.blog.lwsnr.cn/Article/details/0448448.htm
m.blog.lwsnr.cn/Article/details/5939777.htm
m.blog.lwsnr.cn/Article/details/1537959.htm
m.blog.lwsnr.cn/Article/details/9191797.htm
m.blog.lwsnr.cn/Article/details/5975351.htm
m.blog.lwsnr.cn/Article/details/8222082.htm
m.blog.lwsnr.cn/Article/details/0664462.htm
m.blog.lwsnr.cn/Article/details/3331155.htm
m.blog.lwsnr.cn/Article/details/9511557.htm
m.blog.lwsnr.cn/Article/details/7933977.htm
m.blog.lwsnr.cn/Article/details/8064086.htm
m.blog.lwsnr.cn/Article/details/5779975.htm
m.blog.lwsnr.cn/Article/details/7351939.htm
m.blog.lwsnr.cn/Article/details/7355715.htm
m.blog.lwsnr.cn/Article/details/7379155.htm
m.blog.lwsnr.cn/Article/details/5357359.htm
m.blog.lwsnr.cn/Article/details/0026444.htm
m.blog.lwsnr.cn/Article/details/1315373.htm
m.blog.lwsnr.cn/Article/details/2464408.htm
m.blog.lwsnr.cn/Article/details/1753753.htm
m.blog.lwsnr.cn/Article/details/1179953.htm
m.blog.lwsnr.cn/Article/details/7397759.htm
m.blog.lwsnr.cn/Article/details/7755999.htm
m.blog.lwsnr.cn/Article/details/6260022.htm
m.blog.lwsnr.cn/Article/details/0080884.htm
m.blog.lwsnr.cn/Article/details/2860082.htm
m.blog.lwsnr.cn/Article/details/7573513.htm
m.blog.lwsnr.cn/Article/details/1971519.htm
m.blog.lwsnr.cn/Article/details/3793775.htm
m.blog.lwsnr.cn/Article/details/2604264.htm
m.blog.lwsnr.cn/Article/details/8400248.htm
m.blog.lwsnr.cn/Article/details/5391533.htm
m.blog.lwsnr.cn/Article/details/3597371.htm
m.blog.lwsnr.cn/Article/details/7317571.htm
m.blog.lwsnr.cn/Article/details/7931799.htm
m.blog.lwsnr.cn/Article/details/6842060.htm
m.blog.lwsnr.cn/Article/details/3131775.htm
m.blog.lwsnr.cn/Article/details/2824040.htm
m.blog.lwsnr.cn/Article/details/5711719.htm
m.blog.lwsnr.cn/Article/details/5793555.htm
m.blog.lwsnr.cn/Article/details/5553731.htm
m.blog.lwsnr.cn/Article/details/6468060.htm
m.blog.lwsnr.cn/Article/details/2600628.htm
m.blog.lwsnr.cn/Article/details/5339557.htm
m.blog.lwsnr.cn/Article/details/9771533.htm
m.blog.lwsnr.cn/Article/details/0040806.htm
m.blog.lwsnr.cn/Article/details/7915797.htm
m.blog.lwsnr.cn/Article/details/7353799.htm
m.blog.lwsnr.cn/Article/details/7159771.htm
m.blog.lwsnr.cn/Article/details/9391555.htm
m.blog.lwsnr.cn/Article/details/1975315.htm
m.blog.lwsnr.cn/Article/details/0626080.htm
m.blog.lwsnr.cn/Article/details/0028026.htm
m.blog.lwsnr.cn/Article/details/8488608.htm
m.blog.lwsnr.cn/Article/details/1391959.htm
m.blog.lwsnr.cn/Article/details/4008006.htm
m.blog.lwsnr.cn/Article/details/1793551.htm
m.blog.lwsnr.cn/Article/details/1595937.htm
m.blog.lwsnr.cn/Article/details/9597991.htm
m.blog.lwsnr.cn/Article/details/7575379.htm
m.blog.lwsnr.cn/Article/details/9317513.htm
m.blog.lwsnr.cn/Article/details/3331191.htm
m.blog.lwsnr.cn/Article/details/5759719.htm
m.blog.lwsnr.cn/Article/details/5971173.htm
m.blog.lwsnr.cn/Article/details/6604682.htm
m.blog.lwsnr.cn/Article/details/9199977.htm
m.blog.lwsnr.cn/Article/details/2468868.htm
m.blog.lwsnr.cn/Article/details/3977579.htm
m.blog.lwsnr.cn/Article/details/3315715.htm
m.blog.lwsnr.cn/Article/details/4008220.htm
m.blog.lwsnr.cn/Article/details/6466040.htm
m.blog.lwsnr.cn/Article/details/7973371.htm
m.blog.lwsnr.cn/Article/details/2680468.htm
m.blog.lwsnr.cn/Article/details/8860448.htm
m.blog.lwsnr.cn/Article/details/6224248.htm
m.blog.lwsnr.cn/Article/details/2004400.htm
m.blog.lwsnr.cn/Article/details/8488800.htm
m.blog.lwsnr.cn/Article/details/9577959.htm
m.blog.lwsnr.cn/Article/details/6888028.htm
m.blog.lwsnr.cn/Article/details/8888820.htm
m.blog.lwsnr.cn/Article/details/2808680.htm
m.blog.lwsnr.cn/Article/details/2640824.htm
m.blog.lwsnr.cn/Article/details/8048802.htm
m.blog.lwsnr.cn/Article/details/6608680.htm
m.blog.lwsnr.cn/Article/details/7977975.htm
m.blog.lwsnr.cn/Article/details/7379599.htm
m.blog.lwsnr.cn/Article/details/3319911.htm
m.blog.lwsnr.cn/Article/details/4200622.htm
m.blog.lwsnr.cn/Article/details/4480884.htm
m.blog.lwsnr.cn/Article/details/9795951.htm
m.blog.lwsnr.cn/Article/details/2628808.htm
m.blog.lwsnr.cn/Article/details/5797557.htm
m.blog.lwsnr.cn/Article/details/6808284.htm
m.blog.lwsnr.cn/Article/details/0408442.htm
m.blog.lwsnr.cn/Article/details/3591153.htm
m.blog.lwsnr.cn/Article/details/0220048.htm
m.blog.lwsnr.cn/Article/details/1597317.htm
m.blog.lwsnr.cn/Article/details/1391577.htm
m.blog.lwsnr.cn/Article/details/9917377.htm
m.blog.lwsnr.cn/Article/details/7777979.htm
m.blog.lwsnr.cn/Article/details/2646240.htm
m.blog.lwsnr.cn/Article/details/1553557.htm
m.blog.lwsnr.cn/Article/details/0802648.htm
m.blog.lwsnr.cn/Article/details/8682084.htm
m.blog.lwsnr.cn/Article/details/3519535.htm
m.blog.lwsnr.cn/Article/details/6620408.htm
m.blog.lwsnr.cn/Article/details/0204082.htm
m.blog.lwsnr.cn/Article/details/6620426.htm
m.blog.lwsnr.cn/Article/details/7553957.htm
m.blog.lwsnr.cn/Article/details/4422846.htm
m.blog.lwsnr.cn/Article/details/5359115.htm
m.blog.lwsnr.cn/Article/details/5755333.htm
m.blog.lwsnr.cn/Article/details/5755193.htm
m.blog.lwsnr.cn/Article/details/9111931.htm
m.blog.lwsnr.cn/Article/details/6268020.htm
m.blog.lwsnr.cn/Article/details/7997931.htm
m.blog.lwsnr.cn/Article/details/5117731.htm
m.blog.lwsnr.cn/Article/details/0600822.htm
m.blog.lwsnr.cn/Article/details/6484022.htm
m.blog.lwsnr.cn/Article/details/8604260.htm
m.blog.lwsnr.cn/Article/details/2264488.htm
m.blog.lwsnr.cn/Article/details/9737739.htm
m.blog.lwsnr.cn/Article/details/8006448.htm
m.blog.lwsnr.cn/Article/details/1731553.htm
m.blog.lwsnr.cn/Article/details/5355199.htm
m.blog.lwsnr.cn/Article/details/2288064.htm
m.blog.lwsnr.cn/Article/details/5533351.htm
m.blog.lwsnr.cn/Article/details/4824682.htm
m.blog.lwsnr.cn/Article/details/2648480.htm
m.blog.lwsnr.cn/Article/details/1335339.htm
m.blog.lwsnr.cn/Article/details/9397715.htm
m.blog.lwsnr.cn/Article/details/9151357.htm
m.blog.lwsnr.cn/Article/details/6268662.htm
m.blog.lwsnr.cn/Article/details/3515359.htm
m.blog.lwsnr.cn/Article/details/6266468.htm
m.blog.lwsnr.cn/Article/details/9573551.htm
m.blog.lwsnr.cn/Article/details/4826628.htm
m.blog.lwsnr.cn/Article/details/0886662.htm
m.blog.lwsnr.cn/Article/details/3119357.htm
m.blog.lwsnr.cn/Article/details/3151935.htm
m.blog.lwsnr.cn/Article/details/5951575.htm
m.blog.lwsnr.cn/Article/details/7795377.htm
m.blog.lwsnr.cn/Article/details/6280080.htm
m.blog.lwsnr.cn/Article/details/6488262.htm
m.blog.lwsnr.cn/Article/details/7959735.htm
m.blog.lwsnr.cn/Article/details/5931151.htm
m.blog.lwsnr.cn/Article/details/2242002.htm
m.blog.lwsnr.cn/Article/details/5357153.htm
m.blog.lwsnr.cn/Article/details/5551577.htm
m.blog.lwsnr.cn/Article/details/0000400.htm
m.blog.lwsnr.cn/Article/details/9533739.htm
m.blog.lwsnr.cn/Article/details/6646600.htm
m.blog.lwsnr.cn/Article/details/6000260.htm
m.blog.lwsnr.cn/Article/details/8462484.htm
m.blog.lwsnr.cn/Article/details/5579799.htm
m.blog.lwsnr.cn/Article/details/2284664.htm
m.blog.lwsnr.cn/Article/details/5133311.htm
m.blog.lwsnr.cn/Article/details/9513771.htm
m.blog.lwsnr.cn/Article/details/2288082.htm
m.blog.lwsnr.cn/Article/details/9577117.htm
m.blog.lwsnr.cn/Article/details/0248824.htm
m.blog.lwsnr.cn/Article/details/1577177.htm
m.blog.lwsnr.cn/Article/details/3551751.htm
m.blog.lwsnr.cn/Article/details/3911717.htm
m.blog.lwsnr.cn/Article/details/7713575.htm
m.blog.lwsnr.cn/Article/details/3733197.htm
m.blog.lwsnr.cn/Article/details/5573195.htm
m.blog.lwsnr.cn/Article/details/7175939.htm
m.blog.lwsnr.cn/Article/details/4088064.htm
m.blog.lwsnr.cn/Article/details/1339339.htm
m.blog.lwsnr.cn/Article/details/1159571.htm
m.blog.lwsnr.cn/Article/details/0464244.htm
m.blog.lwsnr.cn/Article/details/7171171.htm
m.blog.lwsnr.cn/Article/details/8806046.htm
m.blog.lwsnr.cn/Article/details/9551733.htm
m.blog.lwsnr.cn/Article/details/8404444.htm
m.blog.lwsnr.cn/Article/details/4404442.htm
m.blog.lwsnr.cn/Article/details/3593137.htm
m.blog.lwsnr.cn/Article/details/9711713.htm
m.blog.lwsnr.cn/Article/details/9133715.htm
m.blog.lwsnr.cn/Article/details/7113197.htm
m.blog.lwsnr.cn/Article/details/4042044.htm
m.blog.lwsnr.cn/Article/details/3333531.htm
m.blog.lwsnr.cn/Article/details/1777779.htm
m.blog.lwsnr.cn/Article/details/3911931.htm
m.blog.lwsnr.cn/Article/details/9975737.htm
m.blog.lwsnr.cn/Article/details/7175971.htm
m.blog.lwsnr.cn/Article/details/3591511.htm
m.blog.lwsnr.cn/Article/details/3799111.htm
m.blog.lwsnr.cn/Article/details/4600048.htm
m.blog.lwsnr.cn/Article/details/2800468.htm
m.blog.lwsnr.cn/Article/details/9173311.htm
m.blog.lwsnr.cn/Article/details/4266682.htm
m.blog.lwsnr.cn/Article/details/9593975.htm
m.blog.lwsnr.cn/Article/details/7717933.htm
m.blog.lwsnr.cn/Article/details/7733137.htm
m.blog.lwsnr.cn/Article/details/7999511.htm
m.blog.lwsnr.cn/Article/details/1353593.htm
m.blog.lwsnr.cn/Article/details/1199357.htm
m.blog.lwsnr.cn/Article/details/3173737.htm
m.blog.lwsnr.cn/Article/details/5933771.htm
m.blog.lwsnr.cn/Article/details/5731595.htm
m.blog.lwsnr.cn/Article/details/6648222.htm
m.blog.lwsnr.cn/Article/details/1177957.htm
m.blog.lwsnr.cn/Article/details/1995157.htm
m.blog.lwsnr.cn/Article/details/1337337.htm
m.blog.lwsnr.cn/Article/details/5573159.htm
m.blog.lwsnr.cn/Article/details/6886602.htm
m.blog.lwsnr.cn/Article/details/3577333.htm
m.blog.lwsnr.cn/Article/details/9735713.htm
m.blog.lwsnr.cn/Article/details/9779153.htm
m.blog.lwsnr.cn/Article/details/8022464.htm
m.blog.lwsnr.cn/Article/details/7771391.htm
m.blog.lwsnr.cn/Article/details/1177775.htm
m.blog.lwsnr.cn/Article/details/5137911.htm
m.blog.lwsnr.cn/Article/details/8408208.htm
m.blog.lwsnr.cn/Article/details/5553931.htm
m.blog.lwsnr.cn/Article/details/5959731.htm
m.blog.lwsnr.cn/Article/details/9955977.htm
m.blog.lwsnr.cn/Article/details/5599913.htm
m.blog.lwsnr.cn/Article/details/1591339.htm
m.blog.lwsnr.cn/Article/details/5393175.htm
m.blog.lwsnr.cn/Article/details/3911977.htm
m.blog.lwsnr.cn/Article/details/9153911.htm
m.blog.lwsnr.cn/Article/details/3733935.htm
m.blog.lwsnr.cn/Article/details/2048462.htm
m.blog.lwsnr.cn/Article/details/1117391.htm
m.blog.lwsnr.cn/Article/details/1793931.htm
m.blog.lwsnr.cn/Article/details/3115955.htm
m.blog.lwsnr.cn/Article/details/5331595.htm
m.blog.lwsnr.cn/Article/details/9393573.htm
m.blog.lwsnr.cn/Article/details/5593739.htm
m.blog.lwsnr.cn/Article/details/5599733.htm
m.blog.lwsnr.cn/Article/details/7933333.htm
m.blog.lwsnr.cn/Article/details/7717331.htm
m.blog.lwsnr.cn/Article/details/0244064.htm
m.blog.lwsnr.cn/Article/details/1719575.htm
m.blog.lwsnr.cn/Article/details/3911951.htm
m.blog.lwsnr.cn/Article/details/7113319.htm
m.blog.lwsnr.cn/Article/details/5511715.htm
m.blog.lwsnr.cn/Article/details/1115157.htm
m.blog.lwsnr.cn/Article/details/5379537.htm
m.blog.lwsnr.cn/Article/details/3311155.htm
m.blog.lwsnr.cn/Article/details/5359535.htm
m.blog.lwsnr.cn/Article/details/0208220.htm
m.blog.lwsnr.cn/Article/details/4002046.htm
m.blog.lwsnr.cn/Article/details/9517159.htm
m.blog.lwsnr.cn/Article/details/4404480.htm
m.blog.lwsnr.cn/Article/details/7371115.htm
m.blog.lwsnr.cn/Article/details/9515751.htm
m.blog.lwsnr.cn/Article/details/2864028.htm
m.blog.lwsnr.cn/Article/details/7571995.htm
m.blog.lwsnr.cn/Article/details/1539311.htm
m.blog.lwsnr.cn/Article/details/6864688.htm
m.blog.lwsnr.cn/Article/details/4864088.htm
m.blog.lwsnr.cn/Article/details/9157991.htm
m.blog.lwsnr.cn/Article/details/1573197.htm
m.blog.lwsnr.cn/Article/details/5171577.htm
m.blog.lwsnr.cn/Article/details/3373399.htm
m.blog.lwsnr.cn/Article/details/5171577.htm
m.blog.lwsnr.cn/Article/details/6228482.htm
m.blog.lwsnr.cn/Article/details/9191171.htm
m.blog.lwsnr.cn/Article/details/9113793.htm
m.blog.lwsnr.cn/Article/details/4264000.htm
m.blog.lwsnr.cn/Article/details/5157539.htm
m.blog.lwsnr.cn/Article/details/0208880.htm
m.blog.lwsnr.cn/Article/details/3197753.htm
m.blog.lwsnr.cn/Article/details/1559553.htm
m.blog.lwsnr.cn/Article/details/8862042.htm
m.blog.lwsnr.cn/Article/details/3735793.htm
m.blog.lwsnr.cn/Article/details/3971173.htm
m.blog.lwsnr.cn/Article/details/1111711.htm
m.blog.lwsnr.cn/Article/details/7193537.htm
m.blog.lwsnr.cn/Article/details/1759395.htm
m.blog.lwsnr.cn/Article/details/0620406.htm
m.blog.lwsnr.cn/Article/details/9917553.htm
m.blog.lwsnr.cn/Article/details/5375113.htm
m.blog.lwsnr.cn/Article/details/4626886.htm
m.blog.lwsnr.cn/Article/details/5395335.htm
m.blog.lwsnr.cn/Article/details/3957739.htm
m.blog.lwsnr.cn/Article/details/4266880.htm
m.blog.lwsnr.cn/Article/details/1773773.htm
m.blog.lwsnr.cn/Article/details/5713977.htm
m.blog.lwsnr.cn/Article/details/4824268.htm
m.blog.lwsnr.cn/Article/details/5991359.htm
m.blog.lwsnr.cn/Article/details/7751719.htm
m.blog.lwsnr.cn/Article/details/5751135.htm
m.blog.lwsnr.cn/Article/details/3173353.htm
m.blog.lwsnr.cn/Article/details/3393353.htm
m.blog.lwsnr.cn/Article/details/9571799.htm
m.blog.lwsnr.cn/Article/details/6600622.htm
m.blog.lwsnr.cn/Article/details/5737159.htm
m.blog.lwsnr.cn/Article/details/8688424.htm
m.blog.lwsnr.cn/Article/details/0246860.htm
m.blog.lwsnr.cn/Article/details/7753771.htm
m.blog.lwsnr.cn/Article/details/1951371.htm
m.blog.lwsnr.cn/Article/details/3351179.htm
m.blog.lwsnr.cn/Article/details/1955559.htm
m.blog.lwsnr.cn/Article/details/0808088.htm
m.blog.lwsnr.cn/Article/details/1313313.htm
m.blog.lwsnr.cn/Article/details/3371399.htm
m.blog.lwsnr.cn/Article/details/3359551.htm
m.blog.lwsnr.cn/Article/details/5735915.htm
m.blog.lwsnr.cn/Article/details/9337977.htm
m.blog.lwsnr.cn/Article/details/7555739.htm
m.blog.lwsnr.cn/Article/details/2200080.htm
m.blog.lwsnr.cn/Article/details/6008800.htm
m.blog.lwsnr.cn/Article/details/7717511.htm
m.blog.lwsnr.cn/Article/details/6822680.htm
m.blog.lwsnr.cn/Article/details/2446264.htm
m.blog.lwsnr.cn/Article/details/7133317.htm
m.blog.lwsnr.cn/Article/details/6422088.htm
m.blog.lwsnr.cn/Article/details/5533713.htm
m.blog.lwsnr.cn/Article/details/9159133.htm
m.blog.lwsnr.cn/Article/details/1395371.htm
m.blog.lwsnr.cn/Article/details/2806882.htm
m.blog.lwsnr.cn/Article/details/5313913.htm
m.blog.lwsnr.cn/Article/details/8804680.htm
m.blog.lwsnr.cn/Article/details/5957957.htm
m.blog.lwsnr.cn/Article/details/1537775.htm
m.blog.lwsnr.cn/Article/details/2062008.htm
m.blog.lwsnr.cn/Article/details/2026040.htm
m.blog.lwsnr.cn/Article/details/7513915.htm
m.blog.lwsnr.cn/Article/details/6600822.htm
m.blog.lwsnr.cn/Article/details/3977779.htm
m.blog.lwsnr.cn/Article/details/7773193.htm
m.blog.lwsnr.cn/Article/details/7371773.htm
m.blog.lwsnr.cn/Article/details/7979579.htm
m.blog.lwsnr.cn/Article/details/7197175.htm
m.blog.lwsnr.cn/Article/details/0844262.htm
m.blog.lwsnr.cn/Article/details/4806648.htm
m.blog.lwsnr.cn/Article/details/5717513.htm
m.blog.lwsnr.cn/Article/details/3157919.htm
m.blog.lwsnr.cn/Article/details/8084866.htm
m.blog.lwsnr.cn/Article/details/2804280.htm
m.blog.lwsnr.cn/Article/details/5995177.htm
m.blog.lwsnr.cn/Article/details/3357313.htm
m.blog.lwsnr.cn/Article/details/5713577.htm
m.blog.lwsnr.cn/Article/details/0466628.htm
m.blog.lwsnr.cn/Article/details/7973935.htm
m.blog.lwsnr.cn/Article/details/9795117.htm
m.blog.lwsnr.cn/Article/details/3111337.htm
m.blog.lwsnr.cn/Article/details/5359919.htm
m.blog.lwsnr.cn/Article/details/7317311.htm
m.blog.lwsnr.cn/Article/details/0226666.htm
m.blog.lwsnr.cn/Article/details/4440406.htm
m.blog.lwsnr.cn/Article/details/5577553.htm
m.blog.lwsnr.cn/Article/details/6420064.htm
m.blog.lwsnr.cn/Article/details/1717359.htm
m.blog.lwsnr.cn/Article/details/7317533.htm
m.blog.lwsnr.cn/Article/details/3739511.htm
m.blog.lwsnr.cn/Article/details/5373337.htm
m.blog.lwsnr.cn/Article/details/5573791.htm
m.blog.lwsnr.cn/Article/details/9737735.htm
m.blog.lwsnr.cn/Article/details/8468446.htm
m.blog.lwsnr.cn/Article/details/5959715.htm
m.blog.lwsnr.cn/Article/details/9797597.htm
m.blog.lwsnr.cn/Article/details/5955719.htm
m.blog.lwsnr.cn/Article/details/9339531.htm
m.blog.lwsnr.cn/Article/details/5973513.htm
m.blog.lwsnr.cn/Article/details/0804284.htm
m.blog.lwsnr.cn/Article/details/5739357.htm
m.blog.lwsnr.cn/Article/details/0086440.htm
m.blog.lwsnr.cn/Article/details/7111739.htm
m.blog.lwsnr.cn/Article/details/5553553.htm
m.blog.lwsnr.cn/Article/details/7979519.htm
m.blog.lwsnr.cn/Article/details/9751759.htm
m.blog.lwsnr.cn/Article/details/3711353.htm
m.blog.lwsnr.cn/Article/details/3157771.htm
m.blog.lwsnr.cn/Article/details/6600866.htm
