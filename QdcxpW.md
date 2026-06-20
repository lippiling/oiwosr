蟹氛沙屠腊


"""
Python 安全密码策略 —— 企业级密码强度管理与账户锁定保护
涵盖复杂度规则、强度评估、历史追踪和暴力破解防护
"""

# 安装依赖：pip install zxcvbn
# 密码安全是账户保护的第一道防线，需要综合运用多种策略

import re
import hashlib
import time
import json
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass, field
from collections import defaultdict, deque

try:
    from zxcvbn import zxcvbn
    HAS_ZXCVBN = True
except ImportError:
    HAS_ZXCVBN = False
    print("提示：安装 zxcvbn 可获得更好的密码强度评估（pip install zxcvbn）")

# ========== 第一部分：密码复杂度规则 ==========

class PasswordComplexityPolicy:
    """
    密码复杂度策略 —— 定义并验证密码必须满足的规则。
    遵循 NIST SP 800-63B 指南：更关注长度而非复杂的字符组合要求。
    """

    def __init__(self):
        # 最小长度（NIST 建议至少 8 字符）
        self.min_length: int = 8
        # 最大长度
        self.max_length: int = 128
        # 是否需要大写字母
        self.require_uppercase: bool = True
        # 是否需要小写字母
        self.require_lowercase: bool = True
        # 是否需要数字
        self.require_digit: bool = True
        # 是否需要特殊字符
        self.require_special: bool = False
        # 不允许的常见密码列表
        self.common_passwords: set = set()
        # 不允许包含用户个人信息
        self.check_personal_info: bool = True

    def validate(self, password: str, user_info: Optional[Dict] = None) -> Tuple[bool, List[str]]:
        """
        验证密码是否满足所有复杂度要求。
        返回 (是否通过, 错误信息列表)。
        """
        errors = []

        # 检查长度
        if len(password) < self.min_length:
            errors.append(f"密码长度不能少于 {self.min_length} 个字符")
        if len(password) > self.max_length:
            errors.append(f"密码长度不能超过 {self.max_length} 个字符")

        # 检查字符类型要求
        if self.require_uppercase and not re.search(r"[A-Z]", password):
            errors.append("密码必须包含至少一个大写字母")
        if self.require_lowercase and not re.search(r"[a-z]", password):
            errors.append("密码必须包含至少一个小写字母")
        if self.require_digit and not re.search(r"\d", password):
            errors.append("密码必须包含至少一个数字")
        if self.require_special and not re.search(r"[!@#$%^&*(),.?\":{}|<>_\-+]", password):
            errors.append("密码必须包含至少一个特殊字符")

        # 检查常见密码
        if password.lower() in self.common_passwords:
            errors.append("密码过于常见，请更换")

        # 检查是否包含用户个人信息
        if self.check_personal_info and user_info:
            for field in ["username", "email", "name", "phone"]:
                value = user_info.get(field, "")
                if value and value.lower() in password.lower():
                    errors.append(f"密码不能包含您的{field}信息")
                    break

        return len(errors) == 0, errors


# ========== 第二部分：zxcvbn 强度评估 ==========

class PasswordStrengthEstimator:
    """
    使用 zxcvbn 库评估密码强度。
    zxcvbn 是 Dropbox 开发的密码强度估算器，
    基于真实世界的密码破解模式进行评估。
    """

    @staticmethod
    def estimate(password: str, user_inputs: Optional[List[str]] = None) -> Dict:
        """
        评估密码强度并返回详细的评估报告。
        分数范围 0-4：0=非常弱，1=弱，2=一般，3=强，4=非常强
        """
        if not HAS_ZXCVBN:
            # 降级方案：使用简单的启发式评估
            return PasswordStrengthEstimator._fallback_estimate(password)

        result = zxcvbn(password, user_inputs or [])
        return {
            "score": result["score"],
            "score_label": ["非常弱", "弱", "一般", "强", "非常强"][result["score"]],
            "crack_times_seconds": result["crack_times_seconds"],
            "crack_times_display": result["crack_times_display"],
            "feedback": result.get("feedback", {}),
            "suggestions": result["feedback"].get("suggestions", []),
            "warning": result["feedback"].get("warning", ""),
        }

    @staticmethod
    def _fallback_estimate(password: str) -> Dict:
        """
        降级方案：基于密码长度和字符集的简单评估。
        """
        score = 0
        length = len(password)
        char_types = 0

        if re.search(r"[a-z]", password):
            char_types += 1
        if re.search(r"[A-Z]", password):
            char_types += 1
        if re.search(r"\d", password):
            char_types += 1
        if re.search(r"[^a-zA-Z0-9]", password):
            char_types += 1

        # 简单评分逻辑
        if length >= 16 and char_types >= 3:
            score = 4
        elif length >= 12 and char_types >= 3:
            score = 3
        elif length >= 10 and char_types >= 2:
            score = 2
        elif length >= 8:
            score = 1

        return {
            "score": score,
            "score_label": ["非常弱", "弱", "一般", "强", "非常强"][score],
            "crack_times_seconds": {"online_throttling_100_per_hour": 99999},
            "crack_times_display": {"online_throttling_100_per_hour": "数百年"},
            "feedback": {},
            "suggestions": ["考虑使用更长的密码"] if score < 3 else [],
            "warning": "" if score >= 2 else "密码太弱",
        }


# ========== 第三部分：密码历史追踪 ==========

class PasswordHistory:
    """
    密码历史追踪器 —— 防止用户重复使用最近使用过的密码。
    建议保存最近 5-10 次的历史记录。
    """

    def __init__(self, history_size: int = 5):
        self.history_size = history_size
        # user_id -> deque of hashed passwords
        self._histories: Dict[str, deque] = defaultdict(
            lambda: deque(maxlen=history_size)
        )

    def add_password(self, user_id: str, password: str) -> None:
        """
        记录用户的新密码（存储哈希值，不存明文）。
        """
        pw_hash = self._hash_password(password)
        self._histories[user_id].append(pw_hash)

    def is_password_reused(self, user_id: str, new_password: str) -> bool:
        """
        检查新密码是否在历史记录中（是否被重复使用）。
        """
        new_hash = self._hash_password(new_password)
        return new_hash in self._histories.get(user_id, [])

    def get_history_count(self, user_id: str) -> int:
        """
        获取用户的密码历史记录数量。
        """
        return len(self._histories.get(user_id, []))

    @staticmethod
    def _hash_password(password: str) -> str:
        """
        对密码进行哈希处理（仅用于历史检查，非最终存储）。
        """
        return hashlib.sha256(password.encode()).hexdigest()


# ========== 第四部分：账户锁定机制 ==========

class AccountLockoutManager:
    """
    账户锁定管理器 —— 防止暴力破解攻击。
    策略：连续失败 N 次后锁定账户 T 分钟。
    """

    def __init__(self, max_attempts: int = 5, lockout_minutes: int = 15):
        self.max_attempts = max_attempts
        self.lockout_seconds = lockout_minutes * 60
        # user_id -> [timestamp1, timestamp2, ...] 失败记录时间戳
        self._attempts: Dict[str, List[float]] = defaultdict(list)
        # user_id -> unlock_time 锁定时间
        self._locks: Dict[str, float] = {}

    def record_failed_attempt(self, user_id: str) -> bool:
        """
        记录一次登录失败。
        如果失败次数达到上限，锁定账户。
        返回是否已锁定。
        """
        now = time.time()
        # 清理过期的失败记录
        self._attempts[user_id] = [
            t for t in self._attempts[user_id]
            if now - t < self.lockout_seconds
        ]
        self._attempts[user_id].append(now)

        if len(self._attempts[user_id]) >= self.max_attempts:
            self._locks[user_id] = now + self.lockout_seconds
            print(f"账户 {user_id} 已被锁定 {self.lockout_seconds // 60} 分钟")
            return True
        return False

    def is_locked(self, user_id: str) -> bool:
        """
        检查账户是否被锁定。
        如果锁定时间已过，自动解除锁定。
        """
        if user_id not in self._locks:
            return False
        if time.time() >= self._locks[user_id]:
            del self._locks[user_id]
            self._attempts[user_id] = []
            return False
        remaining = int(self._locks[user_id] - time.time())
        print(f"账户 {user_id} 锁定中（剩余 {remaining // 60} 分钟）")
        return True

    def reset_attempts(self, user_id: str) -> None:
        """
        登录成功后重置失败次数。
        """
        self._attempts[user_id] = []


# ========== 第五部分：综合演示 ==========

def demo_password_policy():
    """
    演示完整的密码策略管理流程。
    """
    print("=== 安全密码策略演示 ===\n")

    # 密码复杂度检查
    print("--- 密码复杂度检查 ---")
    policy = PasswordComplexityPolicy()
    test_passwords = ["123456", "Password1", "MyStr0ng!Pass#2024"]
    for pw in test_passwords:
        valid, errors = policy.validate(pw)
        print(f"  {pw:25s} -> {'通过' if valid else '拒绝'}：{', '.join(errors) if errors else '合规'}")

    # zxcvbn 强度评估
    print("\n--- 密码强度评估 ---")
    estimator = PasswordStrengthEstimator()
    test_pws = ["password", "Tr0ub4dor&3", "correct-horse-battery-staple"]
    for pw in test_pws:
        result = estimator.estimate(pw)
        print(f"  {pw:30s} -> 强度 {result['score']}/4：{result['score_label']}")

    # 密码历史检查
    print("\n--- 密码历史追踪 ---")
    history = PasswordHistory(history_size=3)
    user_id = "user_001"
    history.add_password(user_id, "FirstP@ss1")
    history.add_password(user_id, "SecondP@ss2")
    print(f"  重复使用 Firs...：{history.is_password_reused(user_id, 'FirstP@ss1')}")
    print(f"  新密码 NewP@ss3：{history.is_password_reused(user_id, 'NewP@ss3')}")

    # 账户锁定机制
    print("\n--- 账户锁定保护 ---")
    lockout = AccountLockoutManager(max_attempts=3, lockout_minutes=1)
    for i in range(4):
        locked = lockout.record_failed_attempt(user_id)
        print(f"  第 {i+1} 次失败 -> {'已锁定' if locked else '未锁定'}")
    print(f"  检查锁定状态：{'锁定中' if lockout.is_locked(user_id) else '未锁定'}")


if __name__ == "__main__":
    demo_password_policy()

郧砂导埠惫己墙佳百悔睬房靥说拷

qbe.aira2hc.cn/088685.htm
qbe.aira2hc.cn/240285.htm
qbe.aira2hc.cn/971155.htm
qbw.lmlgyi9.cn/262825.htm
qbw.lmlgyi9.cn/737715.htm
qbw.lmlgyi9.cn/513115.htm
qbw.lmlgyi9.cn/084625.htm
qbw.lmlgyi9.cn/991975.htm
qbw.lmlgyi9.cn/193775.htm
qbw.lmlgyi9.cn/284645.htm
qbw.lmlgyi9.cn/797335.htm
qbw.lmlgyi9.cn/420665.htm
qbw.lmlgyi9.cn/997155.htm
qbq.lmlgyi9.cn/408645.htm
qbq.lmlgyi9.cn/608825.htm
qbq.lmlgyi9.cn/662245.htm
qbq.lmlgyi9.cn/951775.htm
qbq.lmlgyi9.cn/020205.htm
qbq.lmlgyi9.cn/391935.htm
qbq.lmlgyi9.cn/800465.htm
qbq.lmlgyi9.cn/597355.htm
qbq.lmlgyi9.cn/022425.htm
qbq.lmlgyi9.cn/460465.htm
qvtv.lmlgyi9.cn/717395.htm
qvtv.lmlgyi9.cn/133755.htm
qvtv.lmlgyi9.cn/042885.htm
qvtv.lmlgyi9.cn/402865.htm
qvtv.lmlgyi9.cn/080685.htm
qvtv.lmlgyi9.cn/624065.htm
qvtv.lmlgyi9.cn/464645.htm
qvtv.lmlgyi9.cn/759935.htm
qvtv.lmlgyi9.cn/371795.htm
qvtv.lmlgyi9.cn/028845.htm
qvn.lmlgyi9.cn/682645.htm
qvn.lmlgyi9.cn/133395.htm
qvn.lmlgyi9.cn/026445.htm
qvn.lmlgyi9.cn/688405.htm
qvn.lmlgyi9.cn/488445.htm
qvn.lmlgyi9.cn/882685.htm
qvn.lmlgyi9.cn/957975.htm
qvn.lmlgyi9.cn/208605.htm
qvn.lmlgyi9.cn/408605.htm
qvn.lmlgyi9.cn/864065.htm
qvb.lmlgyi9.cn/400405.htm
qvb.lmlgyi9.cn/408205.htm
qvb.lmlgyi9.cn/404425.htm
qvb.lmlgyi9.cn/175335.htm
qvb.lmlgyi9.cn/355575.htm
qvb.lmlgyi9.cn/600265.htm
qvb.lmlgyi9.cn/282025.htm
qvb.lmlgyi9.cn/622025.htm
qvb.lmlgyi9.cn/975535.htm
qvb.lmlgyi9.cn/317735.htm
qvv.lmlgyi9.cn/806025.htm
qvv.lmlgyi9.cn/911335.htm
qvv.lmlgyi9.cn/280425.htm
qvv.lmlgyi9.cn/244845.htm
qvv.lmlgyi9.cn/426205.htm
qvv.lmlgyi9.cn/824025.htm
qvv.lmlgyi9.cn/808225.htm
qvv.lmlgyi9.cn/820285.htm
qvv.lmlgyi9.cn/333335.htm
qvv.lmlgyi9.cn/448685.htm
qvc.lmlgyi9.cn/806245.htm
qvc.lmlgyi9.cn/282605.htm
qvc.lmlgyi9.cn/224665.htm
qvc.lmlgyi9.cn/913315.htm
qvc.lmlgyi9.cn/719775.htm
qvc.lmlgyi9.cn/424685.htm
qvc.lmlgyi9.cn/155175.htm
qvc.lmlgyi9.cn/202665.htm
qvc.lmlgyi9.cn/995375.htm
qvc.lmlgyi9.cn/600465.htm
qvx.lmlgyi9.cn/808885.htm
qvx.lmlgyi9.cn/044425.htm
qvx.lmlgyi9.cn/119115.htm
qvx.lmlgyi9.cn/737955.htm
qvx.lmlgyi9.cn/737395.htm
qvx.lmlgyi9.cn/993175.htm
qvx.lmlgyi9.cn/793535.htm
qvx.lmlgyi9.cn/915935.htm
qvx.lmlgyi9.cn/624845.htm
qvx.lmlgyi9.cn/866405.htm
qvz.lmlgyi9.cn/462865.htm
qvz.lmlgyi9.cn/066205.htm
qvz.lmlgyi9.cn/642625.htm
qvz.lmlgyi9.cn/226485.htm
qvz.lmlgyi9.cn/644205.htm
qvz.lmlgyi9.cn/824485.htm
qvz.lmlgyi9.cn/511955.htm
qvz.lmlgyi9.cn/179935.htm
qvz.lmlgyi9.cn/515535.htm
qvz.lmlgyi9.cn/002245.htm
qvl.lmlgyi9.cn/460685.htm
qvl.lmlgyi9.cn/937395.htm
qvl.lmlgyi9.cn/931355.htm
qvl.lmlgyi9.cn/935395.htm
qvl.lmlgyi9.cn/248885.htm
qvl.lmlgyi9.cn/377735.htm
qvl.lmlgyi9.cn/886825.htm
qvl.lmlgyi9.cn/753335.htm
qvl.lmlgyi9.cn/888225.htm
qvl.lmlgyi9.cn/135555.htm
qvk.lmlgyi9.cn/337735.htm
qvk.lmlgyi9.cn/880225.htm
qvk.lmlgyi9.cn/644885.htm
qvk.lmlgyi9.cn/844205.htm
qvk.lmlgyi9.cn/179595.htm
qvk.lmlgyi9.cn/979355.htm
qvk.lmlgyi9.cn/602465.htm
qvk.lmlgyi9.cn/313335.htm
qvk.lmlgyi9.cn/422025.htm
qvk.lmlgyi9.cn/399975.htm
qvj.lmlgyi9.cn/080605.htm
qvj.lmlgyi9.cn/006425.htm
qvj.lmlgyi9.cn/486665.htm
qvj.lmlgyi9.cn/042605.htm
qvj.lmlgyi9.cn/933315.htm
qvj.lmlgyi9.cn/513335.htm
qvj.lmlgyi9.cn/260805.htm
qvj.lmlgyi9.cn/757755.htm
qvj.lmlgyi9.cn/648685.htm
qvj.lmlgyi9.cn/379335.htm
qvh.lmlgyi9.cn/791935.htm
qvh.lmlgyi9.cn/066225.htm
qvh.lmlgyi9.cn/913135.htm
qvh.lmlgyi9.cn/608445.htm
qvh.lmlgyi9.cn/313715.htm
qvh.lmlgyi9.cn/999195.htm
qvh.lmlgyi9.cn/628445.htm
qvh.lmlgyi9.cn/048665.htm
qvh.lmlgyi9.cn/822625.htm
qvh.lmlgyi9.cn/668885.htm
qvg.lmlgyi9.cn/519155.htm
qvg.lmlgyi9.cn/539115.htm
qvg.lmlgyi9.cn/806665.htm
qvg.lmlgyi9.cn/022405.htm
qvg.lmlgyi9.cn/117515.htm
qvg.lmlgyi9.cn/060685.htm
qvg.lmlgyi9.cn/626645.htm
qvg.lmlgyi9.cn/622625.htm
qvg.lmlgyi9.cn/482245.htm
qvg.lmlgyi9.cn/266825.htm
qvf.lmlgyi9.cn/660085.htm
qvf.lmlgyi9.cn/400025.htm
qvf.lmlgyi9.cn/197115.htm
qvf.lmlgyi9.cn/400845.htm
qvf.lmlgyi9.cn/791935.htm
qvf.lmlgyi9.cn/866805.htm
qvf.lmlgyi9.cn/648445.htm
qvf.lmlgyi9.cn/717775.htm
qvf.lmlgyi9.cn/155195.htm
qvf.lmlgyi9.cn/860825.htm
qvd.lmlgyi9.cn/880665.htm
qvd.lmlgyi9.cn/802445.htm
qvd.lmlgyi9.cn/204265.htm
qvd.lmlgyi9.cn/068685.htm
qvd.lmlgyi9.cn/604085.htm
qvd.lmlgyi9.cn/000405.htm
qvd.lmlgyi9.cn/062445.htm
qvd.lmlgyi9.cn/084225.htm
qvd.lmlgyi9.cn/711155.htm
qvd.lmlgyi9.cn/755195.htm
qvs.lmlgyi9.cn/917375.htm
qvs.lmlgyi9.cn/973935.htm
qvs.lmlgyi9.cn/888865.htm
qvs.lmlgyi9.cn/204845.htm
qvs.lmlgyi9.cn/664265.htm
qvs.lmlgyi9.cn/420685.htm
qvs.lmlgyi9.cn/406605.htm
qvs.lmlgyi9.cn/442645.htm
qvs.lmlgyi9.cn/177115.htm
qvs.lmlgyi9.cn/517395.htm
qva.lmlgyi9.cn/662245.htm
qva.lmlgyi9.cn/668005.htm
qva.lmlgyi9.cn/400085.htm
qva.lmlgyi9.cn/931715.htm
qva.lmlgyi9.cn/660225.htm
qva.lmlgyi9.cn/088225.htm
qva.lmlgyi9.cn/408065.htm
qva.lmlgyi9.cn/624245.htm
qva.lmlgyi9.cn/208485.htm
qva.lmlgyi9.cn/771355.htm
qvp.lmlgyi9.cn/284665.htm
qvp.lmlgyi9.cn/044205.htm
qvp.lmlgyi9.cn/975955.htm
qvp.lmlgyi9.cn/204205.htm
qvp.lmlgyi9.cn/595355.htm
qvp.lmlgyi9.cn/137195.htm
qvp.lmlgyi9.cn/846685.htm
qvp.lmlgyi9.cn/002885.htm
qvp.lmlgyi9.cn/391195.htm
qvp.lmlgyi9.cn/026625.htm
qvo.lmlgyi9.cn/777955.htm
qvo.lmlgyi9.cn/828485.htm
qvo.lmlgyi9.cn/048245.htm
qvo.lmlgyi9.cn/626085.htm
qvo.lmlgyi9.cn/717535.htm
qvo.lmlgyi9.cn/464045.htm
qvo.lmlgyi9.cn/480605.htm
qvo.lmlgyi9.cn/420845.htm
qvo.lmlgyi9.cn/739975.htm
qvo.lmlgyi9.cn/064625.htm
qvi.lmlgyi9.cn/280265.htm
qvi.lmlgyi9.cn/426645.htm
qvi.lmlgyi9.cn/402065.htm
qvi.lmlgyi9.cn/804405.htm
qvi.lmlgyi9.cn/244205.htm
qvi.lmlgyi9.cn/442225.htm
qvi.lmlgyi9.cn/359735.htm
qvi.lmlgyi9.cn/404245.htm
qvi.lmlgyi9.cn/400665.htm
qvi.lmlgyi9.cn/628465.htm
qvu.lmlgyi9.cn/282485.htm
qvu.lmlgyi9.cn/628085.htm
qvu.lmlgyi9.cn/448425.htm
qvu.lmlgyi9.cn/640245.htm
qvu.lmlgyi9.cn/484805.htm
qvu.lmlgyi9.cn/448805.htm
qvu.lmlgyi9.cn/066445.htm
qvu.lmlgyi9.cn/046465.htm
qvu.lmlgyi9.cn/026605.htm
qvu.lmlgyi9.cn/955315.htm
qvy.lmlgyi9.cn/640425.htm
qvy.lmlgyi9.cn/115515.htm
qvy.lmlgyi9.cn/933795.htm
qvy.lmlgyi9.cn/648465.htm
qvy.lmlgyi9.cn/684485.htm
qvy.lmlgyi9.cn/975335.htm
qvy.lmlgyi9.cn/602825.htm
qvy.lmlgyi9.cn/660245.htm
qvy.lmlgyi9.cn/280265.htm
qvy.lmlgyi9.cn/333555.htm
qvt.lmlgyi9.cn/220845.htm
qvt.lmlgyi9.cn/802625.htm
qvt.lmlgyi9.cn/626485.htm
qvt.lmlgyi9.cn/608805.htm
qvt.lmlgyi9.cn/402425.htm
qvt.lmlgyi9.cn/444245.htm
qvt.lmlgyi9.cn/886885.htm
qvt.lmlgyi9.cn/513115.htm
qvt.lmlgyi9.cn/866665.htm
qvt.lmlgyi9.cn/975375.htm
qvr.lmlgyi9.cn/420005.htm
qvr.lmlgyi9.cn/886045.htm
qvr.lmlgyi9.cn/240445.htm
qvr.lmlgyi9.cn/373775.htm
qvr.lmlgyi9.cn/608225.htm
qvr.lmlgyi9.cn/024085.htm
qvr.lmlgyi9.cn/228045.htm
qvr.lmlgyi9.cn/024085.htm
qvr.lmlgyi9.cn/266625.htm
qvr.lmlgyi9.cn/006065.htm
qve.lmlgyi9.cn/642245.htm
qve.lmlgyi9.cn/975395.htm
qve.lmlgyi9.cn/242285.htm
qve.lmlgyi9.cn/595515.htm
qve.lmlgyi9.cn/622685.htm
qve.lmlgyi9.cn/200605.htm
qve.lmlgyi9.cn/428605.htm
qve.lmlgyi9.cn/519755.htm
qve.lmlgyi9.cn/484825.htm
qve.lmlgyi9.cn/351735.htm
qvw.lmlgyi9.cn/355775.htm
qvw.lmlgyi9.cn/848245.htm
qvw.lmlgyi9.cn/464025.htm
qvw.lmlgyi9.cn/064465.htm
qvw.lmlgyi9.cn/593155.htm
qvw.lmlgyi9.cn/626825.htm
qvw.lmlgyi9.cn/286245.htm
qvw.lmlgyi9.cn/482285.htm
qvw.lmlgyi9.cn/206685.htm
qvw.lmlgyi9.cn/535735.htm
qvq.lmlgyi9.cn/224825.htm
qvq.lmlgyi9.cn/424445.htm
qvq.lmlgyi9.cn/000845.htm
qvq.lmlgyi9.cn/042285.htm
qvq.lmlgyi9.cn/622045.htm
qvq.lmlgyi9.cn/757935.htm
qvq.lmlgyi9.cn/882205.htm
qvq.lmlgyi9.cn/646085.htm
qvq.lmlgyi9.cn/666225.htm
qvq.lmlgyi9.cn/662825.htm
qctv.lmlgyi9.cn/313395.htm
qctv.lmlgyi9.cn/622065.htm
qctv.lmlgyi9.cn/046485.htm
qctv.lmlgyi9.cn/244605.htm
qctv.lmlgyi9.cn/866045.htm
qctv.lmlgyi9.cn/773715.htm
qctv.lmlgyi9.cn/402425.htm
qctv.lmlgyi9.cn/062065.htm
qctv.lmlgyi9.cn/224005.htm
qctv.lmlgyi9.cn/399595.htm
qcn.lmlgyi9.cn/628845.htm
qcn.lmlgyi9.cn/446425.htm
qcn.lmlgyi9.cn/624245.htm
qcn.lmlgyi9.cn/399195.htm
qcn.lmlgyi9.cn/846665.htm
qcn.lmlgyi9.cn/682045.htm
qcn.lmlgyi9.cn/751935.htm
qcn.lmlgyi9.cn/577115.htm
qcn.lmlgyi9.cn/460285.htm
qcn.lmlgyi9.cn/020665.htm
qcb.lmlgyi9.cn/062225.htm
qcb.lmlgyi9.cn/266225.htm
qcb.lmlgyi9.cn/197315.htm
qcb.lmlgyi9.cn/208825.htm
qcb.lmlgyi9.cn/337995.htm
qcb.lmlgyi9.cn/997915.htm
qcb.lmlgyi9.cn/666625.htm
qcb.lmlgyi9.cn/191515.htm
qcb.lmlgyi9.cn/446205.htm
qcb.lmlgyi9.cn/020005.htm
qcv.lmlgyi9.cn/802445.htm
qcv.lmlgyi9.cn/377395.htm
qcv.lmlgyi9.cn/379175.htm
qcv.lmlgyi9.cn/026425.htm
qcv.lmlgyi9.cn/444665.htm
qcv.lmlgyi9.cn/571335.htm
qcv.lmlgyi9.cn/606085.htm
qcv.lmlgyi9.cn/240205.htm
qcv.lmlgyi9.cn/199155.htm
qcv.lmlgyi9.cn/151995.htm
qcc.lmlgyi9.cn/200225.htm
qcc.lmlgyi9.cn/757555.htm
qcc.lmlgyi9.cn/800625.htm
qcc.lmlgyi9.cn/820645.htm
qcc.lmlgyi9.cn/137995.htm
qcc.lmlgyi9.cn/597535.htm
qcc.lmlgyi9.cn/008465.htm
qcc.lmlgyi9.cn/268645.htm
qcc.lmlgyi9.cn/604085.htm
qcc.lmlgyi9.cn/379595.htm
qcx.lmlgyi9.cn/440485.htm
qcx.lmlgyi9.cn/535115.htm
qcx.lmlgyi9.cn/264425.htm
qcx.lmlgyi9.cn/759115.htm
qcx.lmlgyi9.cn/153715.htm
qcx.lmlgyi9.cn/046065.htm
qcx.lmlgyi9.cn/993375.htm
qcx.lmlgyi9.cn/931395.htm
qcx.lmlgyi9.cn/268425.htm
qcx.lmlgyi9.cn/686205.htm
qcz.lmlgyi9.cn/373375.htm
qcz.lmlgyi9.cn/408245.htm
qcz.lmlgyi9.cn/426845.htm
qcz.lmlgyi9.cn/226025.htm
qcz.lmlgyi9.cn/000605.htm
qcz.lmlgyi9.cn/771735.htm
qcz.lmlgyi9.cn/775115.htm
qcz.lmlgyi9.cn/022665.htm
qcz.lmlgyi9.cn/555315.htm
qcz.lmlgyi9.cn/359375.htm
qcl.lmlgyi9.cn/640485.htm
qcl.lmlgyi9.cn/004485.htm
qcl.lmlgyi9.cn/802825.htm
qcl.lmlgyi9.cn/828285.htm
qcl.lmlgyi9.cn/608865.htm
qcl.lmlgyi9.cn/426405.htm
qcl.lmlgyi9.cn/464065.htm
qcl.lmlgyi9.cn/666005.htm
qcl.lmlgyi9.cn/864685.htm
qcl.lmlgyi9.cn/535195.htm
qck.lmlgyi9.cn/573595.htm
qck.lmlgyi9.cn/393395.htm
qck.lmlgyi9.cn/442205.htm
qck.lmlgyi9.cn/559915.htm
qck.lmlgyi9.cn/246065.htm
qck.lmlgyi9.cn/840225.htm
qck.lmlgyi9.cn/391355.htm
qck.lmlgyi9.cn/559355.htm
qck.lmlgyi9.cn/268425.htm
qck.lmlgyi9.cn/315575.htm
qcj.lmlgyi9.cn/248405.htm
qcj.lmlgyi9.cn/628465.htm
qcj.lmlgyi9.cn/466845.htm
qcj.lmlgyi9.cn/113155.htm
qcj.lmlgyi9.cn/466425.htm
qcj.lmlgyi9.cn/822065.htm
qcj.lmlgyi9.cn/060025.htm
qcj.lmlgyi9.cn/848625.htm
qcj.lmlgyi9.cn/820045.htm
qcj.lmlgyi9.cn/262605.htm
qch.lmlgyi9.cn/880845.htm
qch.lmlgyi9.cn/319355.htm
qch.lmlgyi9.cn/200205.htm
qch.lmlgyi9.cn/828485.htm
qch.lmlgyi9.cn/513395.htm
qch.lmlgyi9.cn/462425.htm
qch.lmlgyi9.cn/868425.htm
qch.lmlgyi9.cn/202025.htm
qch.lmlgyi9.cn/064205.htm
qch.lmlgyi9.cn/753515.htm
qcg.lmlgyi9.cn/337515.htm
qcg.lmlgyi9.cn/040425.htm
