胶盗看饭棺


========================================
 Python 配置管理与环境变量
========================================

一、为什么需要配置管理
----------------------------------------

应用程序通常需要在不同的环境（开发、测试、生产）中运行，每个环境需要不同的配置。
好的配置管理方案应满足：
- 敏感信息（密码、密钥）不提交到版本控制
- 不同环境加载不同配置
- 配置变更无需修改代码
- 配置值可进行类型验证


二、os.environ — 基础环境变量
----------------------------------------

import os

# 读取环境变量（不存在时返回 None）
db_host = os.environ.get("DB_HOST")
db_port = os.environ.get("DB_PORT", "5432")  # 提供默认值

# 获取所有环境变量（字典形式）
all_env = dict(os.environ)

# 设置环境变量（仅影响当前进程）
os.environ["APP_ENV"] = "production"

# 带类型转换的读取函数
def get_env_int(key: str, default: int = 0) -> int:
    """读取整数类型的环境变量"""
    value = os.environ.get(key)
    if value is None:
        return default
    try:
        return int(value)
    except ValueError:
        raise ValueError(f"环境变量 {key} 不是有效的整数: {value}")

# 实际使用
DEBUG = os.environ.get("DEBUG", "false").lower() == "true"
DATABASE_URL = os.environ.get(
    "DATABASE_URL",
    "sqlite:///default.db"  # 开发环境默认值
)


三、python-dotenv — 加载 .env 文件
----------------------------------------

# .env 文件内容（不要提交到 Git）：
# DB_HOST=localhost
# DB_PASSWORD=my_secret_password
# APP_ENV=development

from dotenv import load_dotenv, find_dotenv

# 自动查找并加载项目根目录的 .env 文件
dotenv_path = find_dotenv(usecwd=True)
if dotenv_path:
    load_dotenv(dotenv_path)
else:
    print("警告: 未找到 .env 文件，使用默认配置")

# 指定特定环境文件
load_dotenv(".env.development", override=True)
# override=True 表示覆盖已存在的环境变量

# 多个 .env 文件叠加（优先级从高到低）
load_dotenv(".env.local", override=True)       # 本地覆盖
load_dotenv(f".env.{os.environ.get('APP_ENV', 'development')}")  # 环境特定
load_dotenv(".env")                            # 公共默认值


四、pydantic-settings — 类型安全的配置
----------------------------------------

from pydantic_settings import BaseSettings
from pydantic import Field, SecretStr
from typing import Optional

class AppSettings(BaseSettings):
    """继承 BaseSettings 自动从环境变量读取配置"""

    # 字段名对应环境变量名（自动转换大写）
    app_name: str = Field(default="MyApp", alias="APP_NAME")
    debug: bool = Field(default=False, alias="DEBUG")
    database_url: SecretStr = Field(..., alias="DATABASE_URL")
    # SecretStr 确保日志中不会意外打印明文密码

    redis_host: str = "localhost"
    redis_port: int = 6379
    redis_password: Optional[str] = None

    # 嵌套配置
    class Config:
        env_file = ".env"          # 自动加载 .env 文件
        env_file_encoding = "utf-8"
        extra = "ignore"           # 忽略未定义的字段

    @property
    def redis_url(self) -> str:
        """计算属性：拼接 Redis 连接字符串"""
        if self.redis_password:
            return f"redis://:{self.redis_password}@{self.redis_host}:{self.redis_port}"
        return f"redis://{self.redis_host}:{self.redis_port}"

# 单例模式获取配置（整个应用共享）
settings = AppSettings()

# 使用示例
print(settings.app_name)          # MyApp
print(settings.database_url)      # SecretStr('******')


五、dataclasses 配置类
----------------------------------------

from dataclasses import dataclass, field, asdict
import json
import os

@dataclass
class DatabaseConfig:
    """数据库配置数据类"""
    host: str = "localhost"
    port: int = 5432
    username: str = "admin"
    password: str = field(default="", repr=False)  # repr=False 隐藏密码
    database: str = "myapp"
    pool_size: int = 10

    @property
    def connection_string(self) -> str:
        return f"postgresql://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"

@dataclass
class AppConfig:
    """应用总配置"""
    env: str = "development"
    debug: bool = False
    db: DatabaseConfig = field(default_factory=DatabaseConfig)
    secret_key: str = field(default="", repr=False)

    @classmethod
    def from_env(cls) -> "AppConfig":
        """从环境变量构建配置对象"""
        return cls(
            env=os.environ.get("APP_ENV", "development"),
            debug=os.environ.get("DEBUG", "false").lower() == "true",
            db=DatabaseConfig(
                host=os.environ.get("DB_HOST", "localhost"),
                port=int(os.environ.get("DB_PORT", "5432")),
            )
        )


六、YAML/JSON 配置文件加载
----------------------------------------

import yaml
from pathlib import Path

def load_yaml_config(config_path: str) -> dict:
    """加载 YAML 配置文件"""
    path = Path(config_path)
    if not path.exists():
        raise FileNotFoundError(f"配置文件不存在: {config_path}")
    with open(path, "r", encoding="utf-8") as f:
        return yaml.safe_load(f)

def load_json_config(config_path: str) -> dict:
    """加载 JSON 配置文件"""
    path = Path(config_path)
    if not path.exists():
        raise FileNotFoundError(f"配置文件不存在: {config_path}")
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)

# 分层配置文件加载（开发环境专用）
class DevConfigLoader:
    """根据环境自动选择配置"""
    _ENV_MAP = {
        "development": "config.dev.yaml",
        "staging": "config.staging.yaml",
        "production": "config.prod.yaml",
    }

    @classmethod
    def load(cls) -> dict:
        env = os.environ.get("APP_ENV", "development")
        config_file = cls._ENV_MAP.get(env, "config.dev.yaml")
        config = load_yaml_config(f"config/{config_file}")

        # 加载本地覆盖（开发环境专用）
        local_file = Path("config/config.local.yaml")
        if local_file.exists():
            local_config = yaml.safe_load(local_file.read_text())
            config.update(local_config)  # 本地配置覆盖公共配置
        return config


七、配置验证模式
----------------------------------------

from dataclasses import dataclass
from typing import Any

class ConfigValidationError(Exception):
    """配置验证异常"""
    pass

def validate_port(value: Any) -> int:
    """验证端口号范围"""
    port = int(value)
    if not 1 <= port <= 65535:
        raise ConfigValidationError(f"端口号超出范围: {port}")
    return port

def validate_url(value: str) -> str:
    """验证 URL 格式"""
    if not value.startswith(("http://", "https://", "postgresql://", "redis://")):
        raise ConfigValidationError(f"不支持的 URL 协议: {value}")
    return value

@dataclass
class ValidatedConfig:
    """带验证的配置类"""
    host: str = "localhost"
    port: int = 8080
    database_url: str = ""

    def __post_init__(self):
        """初始化后自动验证"""
        self.port = validate_port(self.port)
        if self.database_url:
            self.database_url = validate_url(self.database_url)


八、dynaconf 高级配置库
----------------------------------------

# dynaconf 提供了更强大的配置能力
try:
    from dynaconf import Dynaconf, Validator

    settings = Dynaconf(
        # 配置文件列表（按顺序加载，后加载的优先级更高）
        settings_files=["settings.toml", ".secrets.toml"],
        # 环境变量前缀
        envvar_prefix="MYAPP",
        # 支持多环境
        environments=True,
        # 默认环境
        default_env="default",
        # 当前环境
        env="development",
    )

    # 添加验证规则
    settings.validators.register(
        Validator("DATABASE_URL", must_exist=True),
        Validator("DEBUG", must_exist=True, eq=False, env="production"),
        Validator("PORT", must_exist=True, condition=lambda v: 1 <= int(v) <= 65535),
    )

    # 执行验证
    settings.validators.validate()

except ImportError:
    print("dynaconf 未安装，使用基本配置替代")


九、最佳实践总结
----------------------------------------

1. 敏感信息（密码、令牌）始终使用环境变量或 SecretStr
2. 避免将 .env 文件提交到 Git（添加至 .gitignore）
3. 提供合理的默认值，让开发环境开箱即用
4. 使用 dataclass 或 pydantic-settings 对配置进行类型验证
5. 分环境管理配置，但保持配置项的一致性
6. 配置变更应通过部署流程管理，而非运行时修改

晕檬九秦篮垦谔乜辟掠恋焊惭坏鞠

huf.eiyve.cn/668086.Doc
huf.eiyve.cn/088615.Doc
huf.eiyve.cn/446280.Doc
huf.eiyve.cn/080040.Doc
huf.eiyve.cn/468286.Doc
huf.eiyve.cn/800824.Doc
hud.eiyve.cn/220286.Doc
hud.eiyve.cn/324464.Doc
hud.eiyve.cn/684882.Doc
hud.eiyve.cn/345707.Doc
hud.eiyve.cn/822000.Doc
hud.eiyve.cn/684842.Doc
hud.eiyve.cn/202404.Doc
hud.eiyve.cn/808240.Doc
hud.eiyve.cn/286860.Doc
hud.eiyve.cn/553953.Doc
hus.eiyve.cn/880406.Doc
hus.eiyve.cn/864460.Doc
hus.eiyve.cn/044440.Doc
hus.eiyve.cn/008864.Doc
hus.eiyve.cn/642280.Doc
hus.eiyve.cn/664820.Doc
hus.eiyve.cn/408246.Doc
hus.eiyve.cn/317739.Doc
hus.eiyve.cn/804468.Doc
hus.eiyve.cn/426424.Doc
hua.eiyve.cn/868888.Doc
hua.eiyve.cn/484044.Doc
hua.eiyve.cn/240424.Doc
hua.eiyve.cn/620408.Doc
hua.eiyve.cn/460266.Doc
hua.eiyve.cn/860046.Doc
hua.eiyve.cn/202040.Doc
hua.eiyve.cn/244688.Doc
hua.eiyve.cn/628884.Doc
hua.eiyve.cn/424442.Doc
hup.eiyve.cn/261694.Doc
hup.eiyve.cn/828246.Doc
hup.eiyve.cn/064004.Doc
hup.eiyve.cn/242682.Doc
hup.eiyve.cn/606422.Doc
hup.eiyve.cn/480004.Doc
hup.eiyve.cn/240840.Doc
hup.eiyve.cn/618306.Doc
hup.eiyve.cn/348344.Doc
hup.eiyve.cn/006424.Doc
huo.eiyve.cn/080466.Doc
huo.eiyve.cn/222206.Doc
huo.eiyve.cn/408480.Doc
huo.eiyve.cn/608462.Doc
huo.eiyve.cn/283139.Doc
huo.eiyve.cn/806888.Doc
huo.eiyve.cn/026280.Doc
huo.eiyve.cn/707708.Doc
huo.eiyve.cn/468284.Doc
huo.eiyve.cn/804408.Doc
hui.eiyve.cn/066444.Doc
hui.eiyve.cn/999559.Doc
hui.eiyve.cn/060642.Doc
hui.eiyve.cn/488004.Doc
hui.eiyve.cn/317791.Doc
hui.eiyve.cn/107856.Doc
hui.eiyve.cn/402606.Doc
hui.eiyve.cn/886466.Doc
hui.eiyve.cn/202628.Doc
hui.eiyve.cn/628080.Doc
huu.eiyve.cn/539157.Doc
huu.eiyve.cn/719338.Doc
huu.eiyve.cn/008028.Doc
huu.eiyve.cn/208500.Doc
huu.eiyve.cn/600486.Doc
huu.eiyve.cn/979577.Doc
huu.eiyve.cn/862448.Doc
huu.eiyve.cn/060828.Doc
huu.eiyve.cn/804240.Doc
huu.eiyve.cn/800224.Doc
huy.eiyve.cn/325978.Doc
huy.eiyve.cn/440462.Doc
huy.eiyve.cn/531537.Doc
huy.eiyve.cn/684024.Doc
huy.eiyve.cn/717946.Doc
huy.eiyve.cn/311716.Doc
huy.eiyve.cn/677073.Doc
huy.eiyve.cn/959315.Doc
huy.eiyve.cn/002062.Doc
huy.eiyve.cn/575979.Doc
hut.eiyve.cn/391737.Doc
hut.eiyve.cn/484242.Doc
hut.eiyve.cn/133517.Doc
hut.eiyve.cn/802882.Doc
hut.eiyve.cn/437991.Doc
hut.eiyve.cn/020820.Doc
hut.eiyve.cn/020282.Doc
hut.eiyve.cn/488002.Doc
hut.eiyve.cn/641046.Doc
hut.eiyve.cn/733797.Doc
hur.eiyve.cn/999531.Doc
hur.eiyve.cn/940011.Doc
hur.eiyve.cn/040860.Doc
hur.eiyve.cn/840888.Doc
hur.eiyve.cn/400602.Doc
hur.eiyve.cn/084682.Doc
hur.eiyve.cn/080800.Doc
hur.eiyve.cn/662442.Doc
hur.eiyve.cn/222288.Doc
hur.eiyve.cn/244062.Doc
hue.eiyve.cn/268664.Doc
hue.eiyve.cn/682248.Doc
hue.eiyve.cn/933791.Doc
hue.eiyve.cn/826646.Doc
hue.eiyve.cn/604624.Doc
hue.eiyve.cn/080686.Doc
hue.eiyve.cn/386849.Doc
hue.eiyve.cn/240628.Doc
hue.eiyve.cn/422352.Doc
hue.eiyve.cn/446468.Doc
huw.eiyve.cn/317113.Doc
huw.eiyve.cn/907965.Doc
huw.eiyve.cn/682222.Doc
huw.eiyve.cn/686042.Doc
huw.eiyve.cn/268000.Doc
huw.eiyve.cn/977773.Doc
huw.eiyve.cn/488880.Doc
huw.eiyve.cn/040626.Doc
huw.eiyve.cn/082842.Doc
huw.eiyve.cn/208860.Doc
huq.eiyve.cn/537591.Doc
huq.eiyve.cn/804082.Doc
huq.eiyve.cn/460082.Doc
huq.eiyve.cn/824448.Doc
huq.eiyve.cn/608260.Doc
huq.eiyve.cn/244604.Doc
huq.eiyve.cn/664800.Doc
huq.eiyve.cn/731975.Doc
huq.eiyve.cn/464062.Doc
huq.eiyve.cn/660244.Doc
hym.eiyve.cn/660688.Doc
hym.eiyve.cn/604242.Doc
hym.eiyve.cn/440464.Doc
hym.eiyve.cn/468842.Doc
hym.eiyve.cn/268608.Doc
hym.eiyve.cn/640644.Doc
hym.eiyve.cn/666644.Doc
hym.eiyve.cn/591775.Doc
hym.eiyve.cn/844002.Doc
hym.eiyve.cn/268868.Doc
hyn.eiyve.cn/139153.Doc
hyn.eiyve.cn/408848.Doc
hyn.eiyve.cn/888628.Doc
hyn.eiyve.cn/999395.Doc
hyn.eiyve.cn/064620.Doc
hyn.eiyve.cn/208288.Doc
hyn.eiyve.cn/488244.Doc
hyn.eiyve.cn/222291.Doc
hyn.eiyve.cn/404860.Doc
hyn.eiyve.cn/688428.Doc
hyb.eiyve.cn/351593.Doc
hyb.eiyve.cn/880595.Doc
hyb.eiyve.cn/808406.Doc
hyb.eiyve.cn/440682.Doc
hyb.eiyve.cn/939793.Doc
hyb.eiyve.cn/868862.Doc
hyb.eiyve.cn/442779.Doc
hyb.eiyve.cn/640682.Doc
hyb.eiyve.cn/206888.Doc
hyb.eiyve.cn/678876.Doc
hyv.eiyve.cn/228284.Doc
hyv.eiyve.cn/804022.Doc
hyv.eiyve.cn/668040.Doc
hyv.eiyve.cn/244486.Doc
hyv.eiyve.cn/021888.Doc
hyv.eiyve.cn/862622.Doc
hyv.eiyve.cn/313672.Doc
hyv.eiyve.cn/066026.Doc
hyv.eiyve.cn/955517.Doc
hyv.eiyve.cn/282626.Doc
hyc.eiyve.cn/804068.Doc
hyc.eiyve.cn/295139.Doc
hyc.eiyve.cn/084068.Doc
hyc.eiyve.cn/468862.Doc
hyc.eiyve.cn/893446.Doc
hyc.eiyve.cn/826288.Doc
hyc.eiyve.cn/406284.Doc
hyc.eiyve.cn/260622.Doc
hyc.eiyve.cn/004642.Doc
hyc.eiyve.cn/282826.Doc
hyx.eiyve.cn/440468.Doc
hyx.eiyve.cn/646682.Doc
hyx.eiyve.cn/006264.Doc
hyx.eiyve.cn/260888.Doc
hyx.eiyve.cn/296546.Doc
hyx.eiyve.cn/616254.Doc
hyx.eiyve.cn/177449.Doc
hyx.eiyve.cn/757193.Doc
hyx.eiyve.cn/555379.Doc
hyx.eiyve.cn/642662.Doc
hyz.eiyve.cn/464608.Doc
hyz.eiyve.cn/804622.Doc
hyz.eiyve.cn/666266.Doc
hyz.eiyve.cn/660682.Doc
hyz.eiyve.cn/284002.Doc
hyz.eiyve.cn/686046.Doc
hyz.eiyve.cn/572370.Doc
hyz.eiyve.cn/254923.Doc
hyz.eiyve.cn/668024.Doc
hyz.eiyve.cn/286004.Doc
hyl.eiyve.cn/908017.Doc
hyl.eiyve.cn/004868.Doc
hyl.eiyve.cn/402264.Doc
hyl.eiyve.cn/884866.Doc
hyl.eiyve.cn/244444.Doc
hyl.eiyve.cn/404668.Doc
hyl.eiyve.cn/379795.Doc
hyl.eiyve.cn/555731.Doc
hyl.eiyve.cn/480002.Doc
hyl.eiyve.cn/882222.Doc
hyk.eiyve.cn/028400.Doc
hyk.eiyve.cn/882048.Doc
hyk.eiyve.cn/406284.Doc
hyk.eiyve.cn/846462.Doc
hyk.eiyve.cn/913339.Doc
hyk.eiyve.cn/539591.Doc
hyk.eiyve.cn/686266.Doc
hyk.eiyve.cn/448846.Doc
hyk.eiyve.cn/880062.Doc
hyk.eiyve.cn/428644.Doc
hyj.eiyve.cn/042688.Doc
hyj.eiyve.cn/013046.Doc
hyj.eiyve.cn/088484.Doc
hyj.eiyve.cn/088848.Doc
hyj.eiyve.cn/640664.Doc
hyj.eiyve.cn/282628.Doc
hyj.eiyve.cn/680686.Doc
hyj.eiyve.cn/373935.Doc
hyj.eiyve.cn/199151.Doc
hyj.eiyve.cn/848062.Doc
hyh.eiyve.cn/028684.Doc
hyh.eiyve.cn/028868.Doc
hyh.eiyve.cn/282280.Doc
hyh.eiyve.cn/884848.Doc
hyh.eiyve.cn/664840.Doc
hyh.eiyve.cn/002244.Doc
hyh.eiyve.cn/628800.Doc
hyh.eiyve.cn/022200.Doc
hyh.eiyve.cn/579171.Doc
hyh.eiyve.cn/606424.Doc
hyg.eiyve.cn/997579.Doc
hyg.eiyve.cn/575999.Doc
hyg.eiyve.cn/200442.Doc
hyg.eiyve.cn/842426.Doc
hyg.eiyve.cn/442428.Doc
hyg.eiyve.cn/460064.Doc
hyg.eiyve.cn/492215.Doc
hyg.eiyve.cn/684266.Doc
hyg.eiyve.cn/711557.Doc
hyg.eiyve.cn/642824.Doc
hyf.eiyve.cn/059613.Doc
hyf.eiyve.cn/833150.Doc
hyf.eiyve.cn/408420.Doc
hyf.eiyve.cn/286026.Doc
hyf.eiyve.cn/682640.Doc
hyf.eiyve.cn/315410.Doc
hyf.eiyve.cn/448666.Doc
hyf.eiyve.cn/088622.Doc
hyf.eiyve.cn/886442.Doc
hyf.eiyve.cn/048268.Doc
hyd.eiyve.cn/002024.Doc
hyd.eiyve.cn/200408.Doc
hyd.eiyve.cn/286888.Doc
hyd.eiyve.cn/206282.Doc
hyd.eiyve.cn/846022.Doc
hyd.eiyve.cn/036691.Doc
hyd.eiyve.cn/024042.Doc
hyd.eiyve.cn/804480.Doc
hyd.eiyve.cn/116600.Doc
hyd.eiyve.cn/022626.Doc
hys.eiyve.cn/317739.Doc
hys.eiyve.cn/648262.Doc
hys.eiyve.cn/668480.Doc
hys.eiyve.cn/759535.Doc
hys.eiyve.cn/662028.Doc
hys.eiyve.cn/608266.Doc
hys.eiyve.cn/175237.Doc
hys.eiyve.cn/375551.Doc
hys.eiyve.cn/488226.Doc
hys.eiyve.cn/882840.Doc
hya.eiyve.cn/564127.Doc
hya.eiyve.cn/224684.Doc
hya.eiyve.cn/446040.Doc
hya.eiyve.cn/236428.Doc
hya.eiyve.cn/888204.Doc
hya.eiyve.cn/842008.Doc
hya.eiyve.cn/020080.Doc
hya.eiyve.cn/402222.Doc
hya.eiyve.cn/420848.Doc
hya.eiyve.cn/022028.Doc
hyp.eiyve.cn/040028.Doc
hyp.eiyve.cn/442624.Doc
hyp.eiyve.cn/552267.Doc
hyp.eiyve.cn/288868.Doc
hyp.eiyve.cn/668660.Doc
hyp.eiyve.cn/684800.Doc
hyp.eiyve.cn/822006.Doc
hyp.eiyve.cn/602688.Doc
hyp.eiyve.cn/028868.Doc
hyp.eiyve.cn/421840.Doc
hyo.eiyve.cn/462402.Doc
hyo.eiyve.cn/228422.Doc
hyo.eiyve.cn/405538.Doc
hyo.eiyve.cn/026688.Doc
hyo.eiyve.cn/686002.Doc
hyo.eiyve.cn/340006.Doc
hyo.eiyve.cn/688602.Doc
hyo.eiyve.cn/220286.Doc
hyo.eiyve.cn/626206.Doc
hyo.eiyve.cn/006600.Doc
hyi.eiyve.cn/806226.Doc
hyi.eiyve.cn/797375.Doc
hyi.eiyve.cn/882222.Doc
hyi.eiyve.cn/028208.Doc
hyi.eiyve.cn/640628.Doc
hyi.eiyve.cn/480488.Doc
hyi.eiyve.cn/236091.Doc
hyi.eiyve.cn/486404.Doc
hyi.eiyve.cn/286662.Doc
hyi.eiyve.cn/824440.Doc
hyu.eiyve.cn/460828.Doc
hyu.eiyve.cn/440684.Doc
hyu.eiyve.cn/622008.Doc
hyu.eiyve.cn/080082.Doc
hyu.eiyve.cn/479468.Doc
hyu.eiyve.cn/933917.Doc
hyu.eiyve.cn/886480.Doc
hyu.eiyve.cn/488202.Doc
hyu.eiyve.cn/268868.Doc
hyu.eiyve.cn/664882.Doc
hyy.eiyve.cn/640884.Doc
hyy.eiyve.cn/084840.Doc
hyy.eiyve.cn/244082.Doc
hyy.eiyve.cn/963473.Doc
hyy.eiyve.cn/466642.Doc
hyy.eiyve.cn/759191.Doc
hyy.eiyve.cn/026088.Doc
hyy.eiyve.cn/486244.Doc
hyy.eiyve.cn/953731.Doc
hyy.eiyve.cn/828860.Doc
hyt.eiyve.cn/688684.Doc
hyt.eiyve.cn/884684.Doc
hyt.eiyve.cn/648202.Doc
hyt.eiyve.cn/446484.Doc
hyt.eiyve.cn/313939.Doc
hyt.eiyve.cn/804280.Doc
hyt.eiyve.cn/428406.Doc
hyt.eiyve.cn/426848.Doc
hyt.eiyve.cn/692874.Doc
hyt.eiyve.cn/171351.Doc
hyr.eiyve.cn/804266.Doc
hyr.eiyve.cn/658472.Doc
hyr.eiyve.cn/260682.Doc
hyr.eiyve.cn/252612.Doc
hyr.eiyve.cn/628008.Doc
hyr.eiyve.cn/191733.Doc
hyr.eiyve.cn/153077.Doc
hyr.eiyve.cn/868040.Doc
hyr.eiyve.cn/252678.Doc
hyr.eiyve.cn/422864.Doc
hye.eiyve.cn/608888.Doc
hye.eiyve.cn/846020.Doc
hye.eiyve.cn/664822.Doc
hye.eiyve.cn/484884.Doc
hye.eiyve.cn/884880.Doc
hye.eiyve.cn/242608.Doc
hye.eiyve.cn/682860.Doc
hye.eiyve.cn/939797.Doc
hye.eiyve.cn/268626.Doc
hye.eiyve.cn/402608.Doc
hyw.eiyve.cn/171951.Doc
hyw.eiyve.cn/002418.Doc
hyw.eiyve.cn/024844.Doc
hyw.eiyve.cn/342018.Doc
hyw.eiyve.cn/040024.Doc
hyw.eiyve.cn/915513.Doc
hyw.eiyve.cn/046840.Doc
hyw.eiyve.cn/864660.Doc
hyw.eiyve.cn/640080.Doc
hyw.eiyve.cn/688288.Doc
hyq.eiyve.cn/860413.Doc
hyq.eiyve.cn/802040.Doc
hyq.eiyve.cn/558269.Doc
hyq.eiyve.cn/226286.Doc
hyq.eiyve.cn/886884.Doc
hyq.eiyve.cn/440644.Doc
hyq.eiyve.cn/060846.Doc
hyq.eiyve.cn/026082.Doc
hyq.eiyve.cn/448682.Doc
