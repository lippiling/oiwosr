贺穆贤鼐局


"""
Python 输入验证模式 —— 基于 Pydantic V2 的字段/模型验证
涵盖类型校验、自定义规则、消毒处理和注入防护
"""

# 安装依赖：pip install pydantic
# Pydantic 是 Python 最流行的数据验证库，利用类型注解进行运行时校验

from pydantic import (
    BaseModel,
    field_validator,
    model_validator,
    Field,
    EmailStr,
    HttpUrl,
    ConfigDict
)
from typing import Optional, List
import re

# ========== 第一部分：基础字段验证 ==========

class UserRegistration(BaseModel):
    """
    用户注册模型 —— 演示各种字段级别的验证器。
    Pydantic V2 使用 field_validator 替代旧的 validator 装饰器。
    """
    # 类型注解自带基础类型检查
    username: str = Field(..., min_length=3, max_length=20)
    email: EmailStr                               # 自动验证电子邮件格式
    age: int = Field(ge=0, le=150)                # 范围限制：0-150 岁
    website: Optional[HttpUrl] = None             # 自动验证 URL 格式
    password: str = Field(..., min_length=8)
    confirm_password: str
    bio: Optional[str] = Field(None, max_length=500)

    # model_config 用于设置模型级别的行为
    model_config = ConfigDict(
        str_strip_whitespace=True,                # 自动去除字符串首尾空格
        str_min_length=1,                          # 字符串最小长度
        extra="forbid"                             # 禁止传递未定义的字段
    )

    # ---------- 字段级别验证器 ----------

    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        """
        验证用户名的合法性：
        - 只能包含字母、数字和下划线
        - 不能以数字开头
        - 防止 SQL 注入和 XSS 攻击
        """
        # 消毒处理：移除可能的危险字符
        v = v.strip()
        # 正则校验：只允许字母数字和下划线
        if not re.match(r"^[a-zA-Z][a-zA-Z0-9_]*$", v):
            raise ValueError(
                "用户名只能以字母开头，包含字母、数字和下划线"
            )
        # 检查是否包含危险模式
        dangerous_patterns = [
            r"<[^>]*>",          # HTML 标签
            r"script",           # JavaScript 关键字
            r"'.*OR.*'",         # SQL 注入模式
            r"--",               # SQL 注释
            r"/\*.*\*/",         # SQL 块注释
        ]
        for pattern in dangerous_patterns:
            if re.search(pattern, v, re.IGNORECASE):
                raise ValueError("用户名包含不允许的字符模式")
        return v

    @field_validator("password")
    @classmethod
    def validate_password_strength(cls, v: str) -> str:
        """
        验证密码强度：
        - 至少 8 个字符
        - 包含大写字母、小写字母、数字和特殊字符
        """
        if not re.search(r"[A-Z]", v):
            raise ValueError("密码必须包含至少一个大写字母")
        if not re.search(r"[a-z]", v):
            raise ValueError("密码必须包含至少一个小写字母")
        if not re.search(r"\d", v):
            raise ValueError("密码必须包含至少一个数字")
        if not re.search(r"[!@#$%^&*(),.?\":{}|<>]", v):
            raise ValueError("密码必须包含至少一个特殊字符")
        return v

    @field_validator("age")
    @classmethod
    def validate_age_reasonable(cls, v: int) -> int:
        """
        验证年龄的合理性（业务规则验证）。
        """
        if v < 18 and v > 0:
            print("警告：未成年用户需要家长同意")
        return v

    # ---------- 模型级别验证器 ----------

    @model_validator(mode="after")
    def validate_passwords_match(self):
        """
        模型验证器：验证两个密码字段是否匹配。
        mode="after" 表示在字段验证完成后执行，可以访问所有字段值。
        """
        if self.password != self.confirm_password:
            raise ValueError("两次输入的密码不一致")
        return self

    @model_validator(mode="after")
    def check_bio_injection(self):
        """
        模型验证器：对个人简介进行注入防护检查。
        """
        if self.bio:
            # 检测并移除潜在的 XSS 攻击载荷
            xss_patterns = [
                r"<script[^>]*>.*?</script>",
                r"javascript:",
                r"on\w+\s*=",
                r"<iframe[^>]*>",
                r"<embed[^>]*>",
                r"<object[^>]*>",
            ]
            for pattern in xss_patterns:
                if re.search(pattern, self.bio, re.IGNORECASE | re.DOTALL):
                    raise ValueError("个人简介包含不允许的 HTML/JavaScript 内容")
            # 转义剩余 HTML 标签
            self.bio = self.bio.replace("<", "&lt;").replace(">", "&gt;")
        return self


# ========== 第二部分：枚举值与允许值验证 ==========

from enum import Enum

class UserRole(str, Enum):
    """用户角色枚举 —— 限定允许值的范围"""
    ADMIN = "admin"
    USER = "user"
    MODERATOR = "moderator"
    GUEST = "guest"

class UserProfile(BaseModel):
    """使用枚举限制字段的可选值"""
    role: UserRole                                # 只能取枚举中定义的值
    status: str = Field(
        default="active",
        pattern=r"^(active|inactive|suspended)$"  # 正则限定可选值
    )
    tags: List[str] = Field(default_factory=list)

    @field_validator("tags")
    @classmethod
    def validate_tags(cls, v: List[str]) -> List[str]:
        """验证标签列表：每个标签消毒且去重"""
        sanitized = []
        seen = set()
        for tag in v:
            # 只保留字母数字和连字符
            clean = re.sub(r"[^\w\-]", "", tag).strip().lower()
            if clean and clean not in seen and len(clean) <= 20:
                sanitized.append(clean)
                seen.add(clean)
        return sanitized


# ========== 第三部分：综合示例 ==========

def demo_input_validation():
    """
    演示输入验证的完整流程：合法输入与恶意输入对比。
    """
    print("=== Python 输入验证模式演示 ===\n")

    # 合法的输入数据
    print("--- 场景一：合法输入 ---")
    valid_data = {
        "username": "Alice_wang",
        "email": "alice@example.com",
        "age": 28,
        "password": "StrongP@ss1",
        "confirm_password": "StrongP@ss1",
        "bio": "Hello, I am Alice!",
    }
    try:
        user = UserRegistration(**valid_data)
        print(f"验证通过！用户名：{user.username}")
        print(f"邮箱：{user.email}")
        print(f"年龄：{user.age}")
    except Exception as e:
        print(f"验证失败：{e}")

    # 恶意的输入数据
    print("\n--- 场景二：恶意输入（XSS 攻击尝试）---")
    malicious_data = {
        "username": "hacker_007",
        "email": "hacker@evil.com",
        "age": 25,
        "password": "StrongP@ss2",
        "confirm_password": "StrongP@ss2",
        "bio": "<script>alert('xss')</script>Click me!",
    }
    try:
        hacker = UserRegistration(**malicious_data)
        print(f"验证通过！经过消毒的个人简介：{hacker.bio}")
    except Exception as e:
        print(f"恶意输入被拦截：{e}")

    # SQL 注入尝试
    print("\n--- 场景三：SQL 注入尝试 ---")
    sql_injection_data = {**valid_data, "username": "admin' OR '1'='1"}
    try:
        UserRegistration(**sql_injection_data)
    except Exception as e:
        print(f"SQL 注入被拦截：{e}")


if __name__ == "__main__":
    demo_input_validation()

严僮犹煞蚁剐锹染特韵嚷刚偬兜汉

qxq.mmmxz.cn/537373.htm
qxq.mmmxz.cn/351173.htm
qxq.mmmxz.cn/955933.htm
qxq.mmmxz.cn/197973.htm
qxq.mmmxz.cn/177753.htm
qxq.mmmxz.cn/579953.htm
qxq.mmmxz.cn/197333.htm
qxq.mmmxz.cn/911333.htm
qxq.mmmxz.cn/559593.htm
qxq.mmmxz.cn/844463.htm
qzm.mmmxz.cn/711393.htm
qzm.mmmxz.cn/777933.htm
qzm.mmmxz.cn/557753.htm
qzm.mmmxz.cn/991113.htm
qzm.mmmxz.cn/139313.htm
qzm.mmmxz.cn/799913.htm
qzm.mmmxz.cn/195973.htm
qzm.mmmxz.cn/931553.htm
qzm.mmmxz.cn/797393.htm
qzm.mmmxz.cn/022023.htm
qzn.mmmxz.cn/971573.htm
qzn.mmmxz.cn/315353.htm
qzn.mmmxz.cn/179513.htm
qzn.mmmxz.cn/755573.htm
qzn.mmmxz.cn/531733.htm
qzn.mmmxz.cn/864463.htm
qzn.mmmxz.cn/359333.htm
qzn.mmmxz.cn/317953.htm
qzn.mmmxz.cn/157533.htm
qzn.mmmxz.cn/715753.htm
qzb.mmmxz.cn/315353.htm
qzb.mmmxz.cn/751113.htm
qzb.mmmxz.cn/135993.htm
qzb.mmmxz.cn/357553.htm
qzb.mmmxz.cn/759133.htm
qzb.mmmxz.cn/791993.htm
qzb.mmmxz.cn/713553.htm
qzb.mmmxz.cn/222463.htm
qzb.mmmxz.cn/171393.htm
qzb.mmmxz.cn/197773.htm
qzv.mmmxz.cn/440463.htm
qzv.mmmxz.cn/399793.htm
qzv.mmmxz.cn/111393.htm
qzv.mmmxz.cn/377333.htm
qzv.mmmxz.cn/755713.htm
qzv.mmmxz.cn/337753.htm
qzv.mmmxz.cn/404483.htm
qzv.mmmxz.cn/153113.htm
qzv.mmmxz.cn/971173.htm
qzv.mmmxz.cn/319773.htm
qzc.mmmxz.cn/379173.htm
qzc.mmmxz.cn/353953.htm
qzc.mmmxz.cn/800083.htm
qzc.mmmxz.cn/373133.htm
qzc.mmmxz.cn/557773.htm
qzc.mmmxz.cn/555553.htm
qzc.mmmxz.cn/511133.htm
qzc.mmmxz.cn/759393.htm
qzc.mmmxz.cn/757973.htm
qzc.mmmxz.cn/822623.htm
qzx.mmmxz.cn/391593.htm
qzx.mmmxz.cn/593113.htm
qzx.mmmxz.cn/133313.htm
qzx.mmmxz.cn/737593.htm
qzx.mmmxz.cn/353973.htm
qzx.mmmxz.cn/991573.htm
qzx.mmmxz.cn/375353.htm
qzx.mmmxz.cn/979173.htm
qzx.mmmxz.cn/73.htm
qzx.mmmxz.cn/177173.htm
qzz.mmmxz.cn/111313.htm
qzz.mmmxz.cn/575773.htm
qzz.mmmxz.cn/515533.htm
qzz.mmmxz.cn/377993.htm
qzz.mmmxz.cn/173913.htm
qzz.mmmxz.cn/393573.htm
qzz.mmmxz.cn/339393.htm
qzz.mmmxz.cn/379173.htm
qzz.mmmxz.cn/262283.htm
qzz.mmmxz.cn/133113.htm
qzl.mmmxz.cn/993773.htm
qzl.mmmxz.cn/155913.htm
qzl.mmmxz.cn/379513.htm
qzl.mmmxz.cn/339513.htm
qzl.mmmxz.cn/775953.htm
qzl.mmmxz.cn/311133.htm
qzl.mmmxz.cn/773373.htm
qzl.mmmxz.cn/171933.htm
qzl.mmmxz.cn/266663.htm
qzl.mmmxz.cn/915753.htm
qzk.mmmxz.cn/777993.htm
qzk.mmmxz.cn/715793.htm
qzk.mmmxz.cn/951113.htm
qzk.mmmxz.cn/911313.htm
qzk.mmmxz.cn/802223.htm
qzk.mmmxz.cn/911153.htm
qzk.mmmxz.cn/755193.htm
qzk.mmmxz.cn/191173.htm
qzk.mmmxz.cn/779153.htm
qzk.mmmxz.cn/195393.htm
qzj.mmmxz.cn/280063.htm
qzj.mmmxz.cn/533553.htm
qzj.mmmxz.cn/199173.htm
qzj.mmmxz.cn/355513.htm
qzj.mmmxz.cn/155753.htm
qzj.mmmxz.cn/337133.htm
qzj.mmmxz.cn/511133.htm
qzj.mmmxz.cn/551713.htm
qzj.mmmxz.cn/175973.htm
qzj.mmmxz.cn/755153.htm
qzh.mmmxz.cn/488023.htm
qzh.mmmxz.cn/791113.htm
qzh.mmmxz.cn/373973.htm
qzh.mmmxz.cn/975573.htm
qzh.mmmxz.cn/531153.htm
qzh.mmmxz.cn/331173.htm
qzh.mmmxz.cn/111553.htm
qzh.mmmxz.cn/111933.htm
qzh.mmmxz.cn/999993.htm
qzh.mmmxz.cn/137793.htm
qzg.mmmxz.cn/197193.htm
qzg.mmmxz.cn/571933.htm
qzg.mmmxz.cn/888423.htm
qzg.mmmxz.cn/197713.htm
qzg.mmmxz.cn/371513.htm
qzg.mmmxz.cn/553573.htm
qzg.mmmxz.cn/995913.htm
qzg.mmmxz.cn/842243.htm
qzg.mmmxz.cn/995953.htm
qzg.mmmxz.cn/042623.htm
qzf.mmmxz.cn/371773.htm
qzf.mmmxz.cn/537133.htm
qzf.mmmxz.cn/933593.htm
qzf.mmmxz.cn/999913.htm
qzf.mmmxz.cn/755113.htm
qzf.mmmxz.cn/604423.htm
qzf.mmmxz.cn/913353.htm
qzf.mmmxz.cn/73.htm
qzf.mmmxz.cn/804403.htm
qzf.mmmxz.cn/917533.htm
qzd.mmmxz.cn/373793.htm
qzd.mmmxz.cn/202083.htm
qzd.mmmxz.cn/311713.htm
qzd.mmmxz.cn/757173.htm
qzd.mmmxz.cn/179133.htm
qzd.mmmxz.cn/977133.htm
qzd.mmmxz.cn/193513.htm
qzd.mmmxz.cn/244063.htm
qzd.mmmxz.cn/773133.htm
qzd.mmmxz.cn/731313.htm
qzs.mmmxz.cn/086003.htm
qzs.mmmxz.cn/155513.htm
qzs.mmmxz.cn/313793.htm
qzs.mmmxz.cn/919333.htm
qzs.mmmxz.cn/333393.htm
qzs.mmmxz.cn/717973.htm
qzs.mmmxz.cn/515373.htm
qzs.mmmxz.cn/173993.htm
qzs.mmmxz.cn/593373.htm
qzs.mmmxz.cn/357113.htm
qza.mmmxz.cn/884823.htm
qza.mmmxz.cn/913533.htm
qza.mmmxz.cn/197513.htm
qza.mmmxz.cn/159733.htm
qza.mmmxz.cn/973373.htm
qza.mmmxz.cn/597193.htm
qza.mmmxz.cn/93.htm
qza.mmmxz.cn/951373.htm
qza.mmmxz.cn/559193.htm
qza.mmmxz.cn/111993.htm
qzp.mmmxz.cn/951573.htm
qzp.mmmxz.cn/117353.htm
qzp.mmmxz.cn/193513.htm
qzp.mmmxz.cn/804083.htm
qzp.mmmxz.cn/155193.htm
qzp.mmmxz.cn/117513.htm
qzp.mmmxz.cn/799513.htm
qzp.mmmxz.cn/599133.htm
qzp.mmmxz.cn/137993.htm
qzp.mmmxz.cn/991933.htm
qzo.mmmxz.cn/995973.htm
qzo.mmmxz.cn/171513.htm
qzo.mmmxz.cn/391193.htm
qzo.mmmxz.cn/559973.htm
qzo.mmmxz.cn/193733.htm
qzo.mmmxz.cn/088823.htm
qzo.mmmxz.cn/993773.htm
qzo.mmmxz.cn/533193.htm
qzo.mmmxz.cn/868603.htm
qzo.mmmxz.cn/486643.htm
qzi.mmmxz.cn/171353.htm
qzi.mmmxz.cn/135913.htm
qzi.mmmxz.cn/571533.htm
qzi.mmmxz.cn/397993.htm
qzi.mmmxz.cn/533373.htm
qzi.mmmxz.cn/684203.htm
qzi.mmmxz.cn/757553.htm
qzi.mmmxz.cn/997373.htm
qzi.mmmxz.cn/195573.htm
qzi.mmmxz.cn/775773.htm
qzu.mmmxz.cn/771533.htm
qzu.mmmxz.cn/248643.htm
qzu.mmmxz.cn/399753.htm
qzu.mmmxz.cn/917393.htm
qzu.mmmxz.cn/153133.htm
qzu.mmmxz.cn/004263.htm
qzu.mmmxz.cn/377933.htm
qzu.mmmxz.cn/971953.htm
qzu.mmmxz.cn/179733.htm
qzu.mmmxz.cn/799513.htm
qzy.mmmxz.cn/159173.htm
qzy.mmmxz.cn/688483.htm
qzy.mmmxz.cn/771733.htm
qzy.mmmxz.cn/335553.htm
qzy.mmmxz.cn/513373.htm
qzy.mmmxz.cn/355593.htm
qzy.mmmxz.cn/195713.htm
qzy.mmmxz.cn/880023.htm
qzy.mmmxz.cn/262083.htm
qzy.mmmxz.cn/375533.htm
qzt.mmmxz.cn/551193.htm
qzt.mmmxz.cn/202603.htm
qzt.mmmxz.cn/157333.htm
qzt.mmmxz.cn/139193.htm
qzt.mmmxz.cn/199533.htm
qzt.mmmxz.cn/371733.htm
qzt.mmmxz.cn/735313.htm
qzt.mmmxz.cn/917953.htm
qzt.mmmxz.cn/317533.htm
qzt.mmmxz.cn/179953.htm
qzr.mmmxz.cn/642403.htm
qzr.mmmxz.cn/333993.htm
qzr.mmmxz.cn/513153.htm
qzr.mmmxz.cn/535353.htm
qzr.mmmxz.cn/513733.htm
qzr.mmmxz.cn/111773.htm
qzr.mmmxz.cn/644403.htm
qzr.mmmxz.cn/339173.htm
qzr.mmmxz.cn/379553.htm
qzr.mmmxz.cn/840043.htm
qze.mmmxz.cn/997933.htm
qze.mmmxz.cn/555113.htm
qze.mmmxz.cn/951953.htm
qze.mmmxz.cn/533953.htm
qze.mmmxz.cn/773373.htm
qze.mmmxz.cn/860003.htm
qze.mmmxz.cn/828663.htm
qze.mmmxz.cn/975353.htm
qze.mmmxz.cn/311713.htm
qze.mmmxz.cn/482843.htm
qzw.mmmxz.cn/993193.htm
qzw.mmmxz.cn/739733.htm
qzw.mmmxz.cn/315353.htm
qzw.mmmxz.cn/157133.htm
qzw.mmmxz.cn/735313.htm
qzw.mmmxz.cn/997133.htm
qzw.mmmxz.cn/599393.htm
qzw.mmmxz.cn/355933.htm
qzw.mmmxz.cn/000423.htm
qzw.mmmxz.cn/535773.htm
qzq.mmmxz.cn/993933.htm
qzq.mmmxz.cn/175393.htm
qzq.mmmxz.cn/355773.htm
qzq.mmmxz.cn/751393.htm
qzq.mmmxz.cn/573713.htm
qzq.mmmxz.cn/642663.htm
qzq.mmmxz.cn/597773.htm
qzq.mmmxz.cn/973933.htm
qzq.mmmxz.cn/551793.htm
qzq.mmmxz.cn/915733.htm
qlm.mmmxz.cn/751393.htm
qlm.mmmxz.cn/864283.htm
qlm.mmmxz.cn/026263.htm
qlm.mmmxz.cn/911153.htm
qlm.mmmxz.cn/379313.htm
qlm.mmmxz.cn/280283.htm
qlm.mmmxz.cn/573933.htm
qlm.mmmxz.cn/195533.htm
qlm.mmmxz.cn/151573.htm
qlm.mmmxz.cn/373973.htm
qln.mmmxz.cn/517513.htm
qln.mmmxz.cn/426063.htm
qln.mmmxz.cn/715333.htm
qln.mmmxz.cn/777753.htm
qln.mmmxz.cn/622203.htm
qln.mmmxz.cn/399793.htm
qln.mmmxz.cn/226203.htm
qln.mmmxz.cn/733993.htm
qln.mmmxz.cn/995393.htm
qln.mmmxz.cn/513133.htm
qlb.mmmxz.cn/111993.htm
qlb.mmmxz.cn/311913.htm
qlb.mmmxz.cn/979933.htm
qlb.mmmxz.cn/351713.htm
qlb.mmmxz.cn/682623.htm
qlb.mmmxz.cn/313513.htm
qlb.mmmxz.cn/953593.htm
qlb.mmmxz.cn/026643.htm
qlb.mmmxz.cn/519173.htm
qlb.mmmxz.cn/397133.htm
qlv.mmmxz.cn/379373.htm
qlv.mmmxz.cn/593153.htm
qlv.mmmxz.cn/371993.htm
qlv.mmmxz.cn/311573.htm
qlv.mmmxz.cn/826423.htm
qlv.mmmxz.cn/391153.htm
qlv.mmmxz.cn/359393.htm
qlv.mmmxz.cn/260483.htm
qlv.mmmxz.cn/515933.htm
qlv.mmmxz.cn/648603.htm
qlc.mmmxz.cn/484483.htm
qlc.mmmxz.cn/759393.htm
qlc.mmmxz.cn/159913.htm
qlc.mmmxz.cn/606883.htm
qlc.mmmxz.cn/399173.htm
qlc.mmmxz.cn/311353.htm
qlc.mmmxz.cn/804483.htm
qlc.mmmxz.cn/113153.htm
qlc.mmmxz.cn/193733.htm
qlc.mmmxz.cn/113533.htm
qlx.mmmxz.cn/420083.htm
qlx.mmmxz.cn/339793.htm
qlx.mmmxz.cn/173993.htm
qlx.mmmxz.cn/197993.htm
qlx.mmmxz.cn/155573.htm
qlx.mmmxz.cn/373973.htm
qlx.mmmxz.cn/840423.htm
qlx.mmmxz.cn/915973.htm
qlx.mmmxz.cn/933753.htm
qlx.mmmxz.cn/791553.htm
qlz.mmmxz.cn/717773.htm
qlz.mmmxz.cn/951153.htm
qlz.mmmxz.cn/179133.htm
qlz.mmmxz.cn/195953.htm
qlz.mmmxz.cn/977193.htm
qlz.mmmxz.cn/117333.htm
qlz.mmmxz.cn/226263.htm
qlz.mmmxz.cn/599573.htm
qlz.mmmxz.cn/155793.htm
qlz.mmmxz.cn/397333.htm
qll.mmmxz.cn/713733.htm
qll.mmmxz.cn/517373.htm
qll.mmmxz.cn/391513.htm
qll.mmmxz.cn/597733.htm
qll.mmmxz.cn/911333.htm
qll.mmmxz.cn/666283.htm
qll.mmmxz.cn/557773.htm
qll.mmmxz.cn/733193.htm
qll.mmmxz.cn/066623.htm
qll.mmmxz.cn/775933.htm
qlk.mmmxz.cn/339773.htm
qlk.mmmxz.cn/462283.htm
qlk.mmmxz.cn/260043.htm
qlk.mmmxz.cn/404603.htm
qlk.mmmxz.cn/535593.htm
qlk.mmmxz.cn/404043.htm
qlk.mmmxz.cn/171713.htm
qlk.mmmxz.cn/799113.htm
qlk.mmmxz.cn/913553.htm
qlk.mmmxz.cn/195513.htm
qlj.mmmxz.cn/397113.htm
qlj.mmmxz.cn/157733.htm
qlj.mmmxz.cn/595533.htm
qlj.mmmxz.cn/199313.htm
qlj.mmmxz.cn/593113.htm
qlj.mmmxz.cn/997953.htm
qlj.mmmxz.cn/399533.htm
qlj.mmmxz.cn/159353.htm
qlj.mmmxz.cn/004063.htm
qlj.mmmxz.cn/977513.htm
qlh.mmmxz.cn/197153.htm
qlh.mmmxz.cn/606603.htm
qlh.mmmxz.cn/193193.htm
qlh.mmmxz.cn/515373.htm
qlh.mmmxz.cn/911333.htm
qlh.mmmxz.cn/751573.htm
qlh.mmmxz.cn/537973.htm
qlh.mmmxz.cn/917773.htm
qlh.mmmxz.cn/959953.htm
qlh.mmmxz.cn/999333.htm
qlg.mmmxz.cn/159713.htm
qlg.mmmxz.cn/999973.htm
qlg.mmmxz.cn/333733.htm
qlg.mmmxz.cn/777313.htm
qlg.mmmxz.cn/955153.htm
qlg.mmmxz.cn/880883.htm
qlg.mmmxz.cn/591393.htm
qlg.mmmxz.cn/377933.htm
qlg.mmmxz.cn/868663.htm
qlg.mmmxz.cn/337153.htm
qlf.mmmxz.cn/731133.htm
qlf.mmmxz.cn/577313.htm
qlf.mmmxz.cn/004223.htm
qlf.mmmxz.cn/240443.htm
qlf.mmmxz.cn/193373.htm
