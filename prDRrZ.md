啥猜亟滴馗


Python代码注释规范——自文档化代码、类型注解、docstring约定

好的注释不是解释"做了什么"，而是说明"为什么这样做"。Python提供了多层次的文档机制。

import os
from typing import List, Optional, Union

# ========== 自文档化代码：命名即注释 ==========
# 糟糕的命名——需要注释来解释
def proc(d, u, p):
    """处理用户数据（糟糕的命名，必须看文档才知道参数含义）"""
    return [x for x in d if x.get(u) == p]

# 好的命名——代码自解释
def filter_users_by_status(users: list, status_field: str, target_status: str) -> list:
    """筛选指定状态的所有用户"""
    return [u for u in users if u.get(status_field) == target_status]

# ========== 类型注解 vs 注释 ==========
def calculate_discount(
    price: float,
    user_level: str,
    coupon: Optional[float] = None
) -> float:
    """
    计算用户最终折扣价。

    类型注解直接写在签名中，比注释更精确，IDE支持更好。
    """
    level_discounts = {"普通": 1.0, "银卡": 0.95, "金卡": 0.85}
    base = price * level_discounts.get(user_level, 1.0)
    if coupon is not None:
        base = max(0, base - coupon)  # 扣减优惠券，不低于零
    return round(base, 2)

# ========== Google风格docstring ==========
def fetch_user_profile(user_id: int, include_deleted: bool = False) -> dict:
    """
    获取用户详细资料。

    Google风格docstring将参数、返回值、异常分节清晰。

    Args:
        user_id: 用户唯一标识符，必须为正整数。
        include_deleted: 是否包含已删除的用户，默认为False。

    Returns:
        包含用户信息的字典，键包括 'id', 'name', 'email', 'created_at'。
        如果用户不存在，返回空字典。

    Raises:
        ValueError: 当user_id为负数或零时抛出。
        ConnectionError: 当无法连接数据库时抛出。

    Example:
        >>> profile = fetch_user_profile(1)
        >>> profile.get('name')
        '张三'
    """
    if user_id <= 0:
        raise ValueError("用户ID必须为正整数")
    # 模拟数据库查询——实际项目中这里是SQL查询
    return {"id": user_id, "name": "张三", "email": "zhangsan@example.com"}

# ========== NumPy风格docstring ==========
def analyze_scores(scores: List[float]) -> dict:
    """
    分析分数列表的统计特征。

    Parameters
    ----------
    scores : List[float]
        需要分析的分数列表，每个元素应在0-100之间。

    Returns
    -------
    dict
        包含以下键的字典：
        - mean: 平均分
        - max: 最高分
        - min: 最低分
        - passed: 及格人数（>=60）
    """
    if not scores:
        return {"mean": 0.0, "max": 0.0, "min": 0.0, "passed": 0}
    return {
        "mean": sum(scores) / len(scores),
        "max": max(scores),
        "min": min(scores),
        "passed": sum(1 for s in scores if s >= 60),
    }

# ========== 什么时候使用行内注释 ==========
class ConfigLoader:
    """配置文件加载器——只在需要解释WHY时写注释"""

    def __init__(self, config_path: str):
        self.config_path = config_path
        self._settings = {}

    def load(self) -> dict:
        """加载配置文件并处理环境变量覆盖"""
        raw = self._read_file()
        for key, value in raw.items():
            # 使用环境变量覆盖配置项（如果有的话）
            # 这是为了让运维人员可以在不修改配置文件的情况下调整参数
            env_value = os.environ.get(f"APP_{key.upper()}")
            if env_value is not None:
                self._settings[key] = self._convert_type(env_value, type(value))
            else:
                self._settings[key] = value
        return self._settings

    def _read_file(self) -> dict:
        """读取配置文件（简化实现）"""
        return {"host": "localhost", "port": 8080, "debug": True}

    @staticmethod
    def _convert_type(value: str, target_type: type) -> Union[str, int, bool]:
        """将字符串转换为目标类型——环境变量都是字符串"""
        if target_type == bool:
            return value.lower() in ("true", "1", "yes")
        if target_type == int:
            return int(value)
        return value  # 字符串类型直接返回


# ========== doctest使用示例 ==========
def add(a: int, b: int) -> int:
    """
    返回两个整数的和。

    使用doctest嵌入测试用例，既是文档也是测试。

    >>> add(1, 2)
    3
    >>> add(-1, 1)
    0
    >>> add(0, 0)
    0
    """
    return a + b


# 关键原则总结：
# 1. 好的代码本身应该是文档——用有意义的命名
# 2. 类型注解替代"参数是什么"类型的注释
# 3. docstring描述"做什么"和"为什么"，而非"怎么做"
# 4. 行内注释只在WHY不明确时才使用

聘夷鸥闯哪逗科厍俚氨侥峙文图垦

uqj.wfcj3t6.cn/595715.htm
uqj.wfcj3t6.cn/971135.htm
uqj.wfcj3t6.cn/315555.htm
uqj.wfcj3t6.cn/771595.htm
uqj.wfcj3t6.cn/199155.htm
uqj.wfcj3t6.cn/955735.htm
uqj.wfcj3t6.cn/739155.htm
uqj.wfcj3t6.cn/591135.htm
uqh.wfcj3t6.cn/199375.htm
uqh.wfcj3t6.cn/511935.htm
uqh.wfcj3t6.cn/739375.htm
uqh.wfcj3t6.cn/799195.htm
uqh.wfcj3t6.cn/533795.htm
uqh.wfcj3t6.cn/991935.htm
uqh.wfcj3t6.cn/539735.htm
uqh.wfcj3t6.cn/777735.htm
uqh.wfcj3t6.cn/113315.htm
uqh.wfcj3t6.cn/159195.htm
uqg.wfcj3t6.cn/979355.htm
uqg.wfcj3t6.cn/571315.htm
uqg.wfcj3t6.cn/751195.htm
uqg.wfcj3t6.cn/933795.htm
uqg.wfcj3t6.cn/151335.htm
uqg.wfcj3t6.cn/911195.htm
uqg.wfcj3t6.cn/135935.htm
uqg.wfcj3t6.cn/539795.htm
uqg.wfcj3t6.cn/357155.htm
uqg.wfcj3t6.cn/117535.htm
uqf.wfcj3t6.cn/951155.htm
uqf.wfcj3t6.cn/573915.htm
uqf.wfcj3t6.cn/999755.htm
uqf.wfcj3t6.cn/371715.htm
uqf.wfcj3t6.cn/953355.htm
uqf.wfcj3t6.cn/375915.htm
uqf.wfcj3t6.cn/757535.htm
uqf.wfcj3t6.cn/555375.htm
uqf.wfcj3t6.cn/373175.htm
uqf.wfcj3t6.cn/753315.htm
uqd.wfcj3t6.cn/397775.htm
uqd.wfcj3t6.cn/151535.htm
uqd.wfcj3t6.cn/317335.htm
uqd.wfcj3t6.cn/137555.htm
uqd.wfcj3t6.cn/795575.htm
uqd.wfcj3t6.cn/775355.htm
uqd.wfcj3t6.cn/719975.htm
uqd.wfcj3t6.cn/571175.htm
uqd.wfcj3t6.cn/779935.htm
uqd.wfcj3t6.cn/937555.htm
uqs.wfcj3t6.cn/113595.htm
uqs.wfcj3t6.cn/771155.htm
uqs.wfcj3t6.cn/771795.htm
uqs.wfcj3t6.cn/539175.htm
uqs.wfcj3t6.cn/735955.htm
uqs.wfcj3t6.cn/111735.htm
uqs.wfcj3t6.cn/137395.htm
uqs.wfcj3t6.cn/151595.htm
uqs.wfcj3t6.cn/533135.htm
uqs.wfcj3t6.cn/377975.htm
uqa.wfcj3t6.cn/355715.htm
uqa.wfcj3t6.cn/197975.htm
uqa.wfcj3t6.cn/391515.htm
uqa.wfcj3t6.cn/339335.htm
uqa.wfcj3t6.cn/737535.htm
uqa.wfcj3t6.cn/157935.htm
uqa.wfcj3t6.cn/177995.htm
uqa.wfcj3t6.cn/513515.htm
uqa.wfcj3t6.cn/157175.htm
uqa.wfcj3t6.cn/151995.htm
uqp.wfcj3t6.cn/119575.htm
uqp.wfcj3t6.cn/773775.htm
uqp.wfcj3t6.cn/115195.htm
uqp.wfcj3t6.cn/955155.htm
uqp.wfcj3t6.cn/393195.htm
uqp.wfcj3t6.cn/311175.htm
uqp.wfcj3t6.cn/999735.htm
uqp.wfcj3t6.cn/713795.htm
uqp.wfcj3t6.cn/971715.htm
uqp.wfcj3t6.cn/111535.htm
uqo.wfcj3t6.cn/359935.htm
uqo.wfcj3t6.cn/337935.htm
uqo.wfcj3t6.cn/513535.htm
uqo.wfcj3t6.cn/757795.htm
uqo.wfcj3t6.cn/719135.htm
uqo.wfcj3t6.cn/737775.htm
uqo.wfcj3t6.cn/175555.htm
uqo.wfcj3t6.cn/951175.htm
uqo.wfcj3t6.cn/537115.htm
uqo.wfcj3t6.cn/359375.htm
uqi.wfcj3t6.cn/573315.htm
uqi.wfcj3t6.cn/593115.htm
uqi.wfcj3t6.cn/997595.htm
uqi.wfcj3t6.cn/173995.htm
uqi.wfcj3t6.cn/573515.htm
uqi.wfcj3t6.cn/991175.htm
uqi.wfcj3t6.cn/519935.htm
uqi.wfcj3t6.cn/919715.htm
uqi.wfcj3t6.cn/777135.htm
uqi.wfcj3t6.cn/511915.htm
uqu.wfcj3t6.cn/173995.htm
uqu.wfcj3t6.cn/757115.htm
uqu.wfcj3t6.cn/359915.htm
uqu.wfcj3t6.cn/957175.htm
uqu.wfcj3t6.cn/311775.htm
uqu.wfcj3t6.cn/771175.htm
uqu.wfcj3t6.cn/155155.htm
uqu.wfcj3t6.cn/737115.htm
uqu.wfcj3t6.cn/173375.htm
uqu.wfcj3t6.cn/799535.htm
uqy.wfcj3t6.cn/793375.htm
uqy.wfcj3t6.cn/935375.htm
uqy.wfcj3t6.cn/353195.htm
uqy.wfcj3t6.cn/577915.htm
uqy.wfcj3t6.cn/599715.htm
uqy.wfcj3t6.cn/959555.htm
uqy.wfcj3t6.cn/773715.htm
uqy.wfcj3t6.cn/397515.htm
uqy.wfcj3t6.cn/719375.htm
uqy.wfcj3t6.cn/391715.htm
uqt.wfcj3t6.cn/173175.htm
uqt.wfcj3t6.cn/115755.htm
uqt.wfcj3t6.cn/779395.htm
uqt.wfcj3t6.cn/977955.htm
uqt.wfcj3t6.cn/953375.htm
uqt.wfcj3t6.cn/997395.htm
uqt.wfcj3t6.cn/131115.htm
uqt.wfcj3t6.cn/797975.htm
uqt.wfcj3t6.cn/317375.htm
uqt.wfcj3t6.cn/919755.htm
uqr.wfcj3t6.cn/557715.htm
uqr.wfcj3t6.cn/919315.htm
uqr.wfcj3t6.cn/551595.htm
uqr.wfcj3t6.cn/317775.htm
uqr.wfcj3t6.cn/159195.htm
uqr.wfcj3t6.cn/717135.htm
uqr.wfcj3t6.cn/519995.htm
uqr.wfcj3t6.cn/151335.htm
uqr.wfcj3t6.cn/597395.htm
uqr.wfcj3t6.cn/397595.htm
uqe.wfcj3t6.cn/359555.htm
uqe.wfcj3t6.cn/751335.htm
uqe.wfcj3t6.cn/713355.htm
uqe.wfcj3t6.cn/175375.htm
uqe.wfcj3t6.cn/373315.htm
uqe.wfcj3t6.cn/911395.htm
uqe.wfcj3t6.cn/137175.htm
uqe.wfcj3t6.cn/393595.htm
uqe.wfcj3t6.cn/177595.htm
uqe.wfcj3t6.cn/911555.htm
uqw.wfcj3t6.cn/399315.htm
uqw.wfcj3t6.cn/399935.htm
uqw.wfcj3t6.cn/397395.htm
uqw.wfcj3t6.cn/797355.htm
uqw.wfcj3t6.cn/757595.htm
uqw.wfcj3t6.cn/191975.htm
uqw.wfcj3t6.cn/379315.htm
uqw.wfcj3t6.cn/933775.htm
uqw.wfcj3t6.cn/519775.htm
uqw.wfcj3t6.cn/933555.htm
uqq.wfcj3t6.cn/319735.htm
uqq.wfcj3t6.cn/555335.htm
uqq.wfcj3t6.cn/955155.htm
uqq.wfcj3t6.cn/137135.htm
uqq.wfcj3t6.cn/137175.htm
uqq.wfcj3t6.cn/353995.htm
uqq.wfcj3t6.cn/791735.htm
uqq.wfcj3t6.cn/793775.htm
uqq.wfcj3t6.cn/975195.htm
uqq.wfcj3t6.cn/975395.htm
ymtv.wfcj3t6.cn/711775.htm
ymtv.wfcj3t6.cn/751395.htm
ymtv.wfcj3t6.cn/955915.htm
ymtv.wfcj3t6.cn/557555.htm
ymtv.wfcj3t6.cn/177315.htm
ymtv.wfcj3t6.cn/531315.htm
ymtv.wfcj3t6.cn/517975.htm
ymtv.wfcj3t6.cn/955175.htm
ymtv.wfcj3t6.cn/915715.htm
ymtv.wfcj3t6.cn/191395.htm
ymn.wfcj3t6.cn/755575.htm
ymn.wfcj3t6.cn/173555.htm
ymn.wfcj3t6.cn/395555.htm
ymn.wfcj3t6.cn/973915.htm
ymn.wfcj3t6.cn/313115.htm
ymn.wfcj3t6.cn/135335.htm
ymn.wfcj3t6.cn/937795.htm
ymn.wfcj3t6.cn/973315.htm
ymn.wfcj3t6.cn/539595.htm
ymn.wfcj3t6.cn/531975.htm
ymb.wfcj3t6.cn/993395.htm
ymb.wfcj3t6.cn/353395.htm
ymb.wfcj3t6.cn/397935.htm
ymb.wfcj3t6.cn/795935.htm
ymb.wfcj3t6.cn/755375.htm
ymb.wfcj3t6.cn/531175.htm
ymb.wfcj3t6.cn/915735.htm
ymb.wfcj3t6.cn/713595.htm
ymb.wfcj3t6.cn/997515.htm
ymb.wfcj3t6.cn/579755.htm
ymv.wfcj3t6.cn/519715.htm
ymv.wfcj3t6.cn/531795.htm
ymv.wfcj3t6.cn/513735.htm
ymv.wfcj3t6.cn/995595.htm
ymv.wfcj3t6.cn/139395.htm
ymv.wfcj3t6.cn/519315.htm
ymv.wfcj3t6.cn/739795.htm
ymv.wfcj3t6.cn/795775.htm
ymv.wfcj3t6.cn/199355.htm
ymv.wfcj3t6.cn/357195.htm
ymc.wfcj3t6.cn/537535.htm
ymc.wfcj3t6.cn/155915.htm
ymc.wfcj3t6.cn/575355.htm
ymc.wfcj3t6.cn/757915.htm
ymc.wfcj3t6.cn/917955.htm
ymc.wfcj3t6.cn/195115.htm
ymc.wfcj3t6.cn/193135.htm
ymc.wfcj3t6.cn/177915.htm
ymc.wfcj3t6.cn/511395.htm
ymc.wfcj3t6.cn/955915.htm
ymx.wfcj3t6.cn/117775.htm
ymx.wfcj3t6.cn/975975.htm
ymx.wfcj3t6.cn/595755.htm
ymx.wfcj3t6.cn/715135.htm
ymx.wfcj3t6.cn/371175.htm
ymx.wfcj3t6.cn/795535.htm
ymx.wfcj3t6.cn/359375.htm
ymx.wfcj3t6.cn/933935.htm
ymx.wfcj3t6.cn/171515.htm
ymx.wfcj3t6.cn/375735.htm
ymz.wfcj3t6.cn/153955.htm
ymz.wfcj3t6.cn/715795.htm
ymz.wfcj3t6.cn/517975.htm
ymz.wfcj3t6.cn/755715.htm
ymz.wfcj3t6.cn/939715.htm
ymz.wfcj3t6.cn/517375.htm
ymz.wfcj3t6.cn/539375.htm
ymz.wfcj3t6.cn/551995.htm
ymz.wfcj3t6.cn/559175.htm
ymz.wfcj3t6.cn/315195.htm
yml.wfcj3t6.cn/919315.htm
yml.wfcj3t6.cn/779355.htm
yml.wfcj3t6.cn/755395.htm
yml.wfcj3t6.cn/933335.htm
yml.wfcj3t6.cn/993915.htm
yml.wfcj3t6.cn/559335.htm
yml.wfcj3t6.cn/977995.htm
yml.wfcj3t6.cn/115555.htm
yml.wfcj3t6.cn/717795.htm
yml.wfcj3t6.cn/717375.htm
ymk.wfcj3t6.cn/577395.htm
ymk.wfcj3t6.cn/753795.htm
ymk.wfcj3t6.cn/571355.htm
ymk.wfcj3t6.cn/175175.htm
ymk.wfcj3t6.cn/319715.htm
ymk.wfcj3t6.cn/135555.htm
ymk.wfcj3t6.cn/133955.htm
ymk.wfcj3t6.cn/973355.htm
ymk.wfcj3t6.cn/795755.htm
ymk.wfcj3t6.cn/319515.htm
ymj.wfcj3t6.cn/971155.htm
ymj.wfcj3t6.cn/553935.htm
ymj.wfcj3t6.cn/391175.htm
ymj.wfcj3t6.cn/719915.htm
ymj.wfcj3t6.cn/715135.htm
ymj.wfcj3t6.cn/979555.htm
ymj.wfcj3t6.cn/355955.htm
ymj.wfcj3t6.cn/117955.htm
ymj.wfcj3t6.cn/515955.htm
ymj.wfcj3t6.cn/115755.htm
ymh.wfcj3t6.cn/911715.htm
ymh.wfcj3t6.cn/593975.htm
ymh.wfcj3t6.cn/977975.htm
ymh.wfcj3t6.cn/753735.htm
ymh.wfcj3t6.cn/993955.htm
ymh.wfcj3t6.cn/795995.htm
ymh.wfcj3t6.cn/771975.htm
ymh.wfcj3t6.cn/715935.htm
ymh.wfcj3t6.cn/713935.htm
ymh.wfcj3t6.cn/777595.htm
ymg.wfcj3t6.cn/753395.htm
ymg.wfcj3t6.cn/119595.htm
ymg.wfcj3t6.cn/913555.htm
ymg.wfcj3t6.cn/793515.htm
ymg.wfcj3t6.cn/139355.htm
ymg.wfcj3t6.cn/951935.htm
ymg.wfcj3t6.cn/173375.htm
ymg.wfcj3t6.cn/357315.htm
ymg.wfcj3t6.cn/795935.htm
ymg.wfcj3t6.cn/111915.htm
ymf.wfcj3t6.cn/115115.htm
ymf.wfcj3t6.cn/753735.htm
ymf.wfcj3t6.cn/715595.htm
ymf.wfcj3t6.cn/573375.htm
ymf.wfcj3t6.cn/715115.htm
ymf.wfcj3t6.cn/319515.htm
ymf.wfcj3t6.cn/937555.htm
ymf.wfcj3t6.cn/599315.htm
ymf.wfcj3t6.cn/333135.htm
ymf.wfcj3t6.cn/135115.htm
ymd.wfcj3t6.cn/973155.htm
ymd.wfcj3t6.cn/115715.htm
ymd.wfcj3t6.cn/993935.htm
ymd.wfcj3t6.cn/737535.htm
ymd.wfcj3t6.cn/559375.htm
ymd.wfcj3t6.cn/135315.htm
ymd.wfcj3t6.cn/555915.htm
ymd.wfcj3t6.cn/931515.htm
ymd.wfcj3t6.cn/513315.htm
ymd.wfcj3t6.cn/957575.htm
yms.wfcj3t6.cn/997955.htm
yms.wfcj3t6.cn/313595.htm
yms.wfcj3t6.cn/331515.htm
yms.wfcj3t6.cn/973375.htm
yms.wfcj3t6.cn/555915.htm
yms.wfcj3t6.cn/535335.htm
yms.wfcj3t6.cn/357935.htm
yms.wfcj3t6.cn/577375.htm
yms.wfcj3t6.cn/375795.htm
yms.wfcj3t6.cn/973395.htm
yma.wfcj3t6.cn/195135.htm
yma.wfcj3t6.cn/919335.htm
yma.wfcj3t6.cn/357375.htm
yma.wfcj3t6.cn/935975.htm
yma.wfcj3t6.cn/979335.htm
yma.wfcj3t6.cn/711395.htm
yma.wfcj3t6.cn/775735.htm
yma.wfcj3t6.cn/573135.htm
yma.wfcj3t6.cn/771775.htm
yma.wfcj3t6.cn/739955.htm
ymp.wfcj3t6.cn/755975.htm
ymp.wfcj3t6.cn/937115.htm
ymp.wfcj3t6.cn/397955.htm
ymp.wfcj3t6.cn/711375.htm
ymp.wfcj3t6.cn/773915.htm
ymp.wfcj3t6.cn/717995.htm
ymp.wfcj3t6.cn/979335.htm
ymp.wfcj3t6.cn/155155.htm
ymp.wfcj3t6.cn/797715.htm
ymp.wfcj3t6.cn/173515.htm
ymo.wfcj3t6.cn/159795.htm
ymo.wfcj3t6.cn/171735.htm
ymo.wfcj3t6.cn/193555.htm
ymo.wfcj3t6.cn/579335.htm
ymo.wfcj3t6.cn/595555.htm
ymo.wfcj3t6.cn/539155.htm
ymo.wfcj3t6.cn/591155.htm
ymo.wfcj3t6.cn/937375.htm
ymo.wfcj3t6.cn/979795.htm
ymo.wfcj3t6.cn/353155.htm
ymi.wfcj3t6.cn/917115.htm
ymi.wfcj3t6.cn/151555.htm
ymi.wfcj3t6.cn/931775.htm
ymi.wfcj3t6.cn/197195.htm
ymi.wfcj3t6.cn/799335.htm
ymi.wfcj3t6.cn/793195.htm
ymi.wfcj3t6.cn/733555.htm
ymi.wfcj3t6.cn/153575.htm
ymi.wfcj3t6.cn/719775.htm
ymi.wfcj3t6.cn/793715.htm
ymu.wfcj3t6.cn/393535.htm
ymu.wfcj3t6.cn/153355.htm
ymu.wfcj3t6.cn/715935.htm
ymu.wfcj3t6.cn/173595.htm
ymu.wfcj3t6.cn/999735.htm
ymu.wfcj3t6.cn/731795.htm
ymu.wfcj3t6.cn/791715.htm
ymu.wfcj3t6.cn/399735.htm
ymu.wfcj3t6.cn/973775.htm
ymu.wfcj3t6.cn/573935.htm
ymy.wfcj3t6.cn/933515.htm
ymy.wfcj3t6.cn/995335.htm
ymy.wfcj3t6.cn/911135.htm
ymy.wfcj3t6.cn/517735.htm
ymy.wfcj3t6.cn/797595.htm
ymy.wfcj3t6.cn/355135.htm
ymy.wfcj3t6.cn/517735.htm
ymy.wfcj3t6.cn/735135.htm
ymy.wfcj3t6.cn/311335.htm
ymy.wfcj3t6.cn/959115.htm
ymt.wfcj3t6.cn/393975.htm
ymt.wfcj3t6.cn/351515.htm
ymt.wfcj3t6.cn/531355.htm
ymt.wfcj3t6.cn/171195.htm
ymt.wfcj3t6.cn/791935.htm
ymt.wfcj3t6.cn/379175.htm
ymt.wfcj3t6.cn/995195.htm
ymt.wfcj3t6.cn/735935.htm
ymt.wfcj3t6.cn/715955.htm
ymt.wfcj3t6.cn/719335.htm
ymr.wfcj3t6.cn/937775.htm
ymr.wfcj3t6.cn/333575.htm
ymr.wfcj3t6.cn/135155.htm
ymr.wfcj3t6.cn/133115.htm
ymr.wfcj3t6.cn/539395.htm
ymr.wfcj3t6.cn/531555.htm
ymr.wfcj3t6.cn/715755.htm
