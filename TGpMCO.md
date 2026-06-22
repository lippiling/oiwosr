志乇仙良蕉


"""
Python 安全日志审计 —— 企业级安全事件记录与完整性保护
涵盖敏感数据脱敏、不可变存储、日志完整性验证等核心功能
"""

# 安装依赖：pip install cryptography
# 安全审计日志记录所有安全相关事件，是事后分析和合规审计的基础

import json
import time
import hmac
import hashlib
import re
from typing import Dict, Any, List, Optional
from dataclasses import dataclass, field, asdict
from datetime import datetime

# ========== 第一部分：敏感数据脱敏 ==========

class DataMasker:
    """
    敏感数据脱敏器 —— 在记录日志前对敏感信息进行处理。
    确保日志不会泄露密码、令牌、个人身份信息等敏感数据。
    """

    # 敏感字段名称模式
    SENSITIVE_FIELDS = {
        "password", "passwd", "secret", "token", "access_token",
        "refresh_token", "api_key", "apikey", "credit_card",
        "ssn", "phone", "phone_number", "email",
        "authorization", "cookie", "session_id"
    }

    # 信用卡号正则（简单匹配）
    CC_PATTERN = re.compile(r"\b(?:\d{4}[-\s]?){3}\d{4}\b")

    # 电子邮件正则
    EMAIL_PATTERN = re.compile(r"\b[\w.+-]+@[\w-]+\.[\w.-]+\b")

    # IP 地址正则（用于模糊化处理）
    IP_PATTERN = re.compile(r"\b(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})\b")

    @classmethod
    def mask_value(cls, key: str, value: Any) -> Any:
        """
        根据字段名对值进行脱敏处理。
        """
        key_lower = key.lower()

        # 检查是否为敏感字段
        if any(sensitive in key_lower for sensitive in cls.SENSITIVE_FIELDS):
            if isinstance(value, str) and len(value) > 4:
                return value[:2] + "****" + value[-2:]
            return "****"

        # 递归处理字典和列表
        if isinstance(value, dict):
            return {k: cls.mask_value(k, v) for k, v in value.items()}
        if isinstance(value, list):
            return [cls.mask_value(key, item) for item in value]
        return value

    @classmethod
    def mask_text(cls, text: str) -> str:
        """
        对自由文本中的敏感信息进行脱敏。
        """
        # 脱敏信用卡号
        text = cls.CC_PATTERN.sub("****-****-****-****", text)
        # 脱敏电子邮件（保留域名）
        text = cls.EMAIL_PATTERN.sub(lambda m: m.group()[0] + "***@" + m.group().split("@")[1], text)
        # 模糊化 IP 地址
        text = cls.IP_PATTERN.sub(r"\1.\2.\3.xxx", text)
        return text


# ========== 第二部分：审计日志条目 ==========

@dataclass
class AuditLogEntry:
    """
    审计日志条目 —— 记录安全事件的标准化格式。
    每条日志包含事件描述、来源、结果和时间信息。
    """
    timestamp: float                         # 事件发生时间
    event_type: str                          # 事件类型（如 LOGIN、DATA_ACCESS）
    user_id: str                             # 操作用户
    action: str                              # 具体操作描述
    resource: str                            # 访问的资源
    result: str                              # 结果（SUCCESS/FAILURE）
    ip_address: str                          # 来源 IP
    details: Dict[str, Any] = field(default_factory=dict)  # 附加详情（已脱敏）
    log_id: str = ""                         # 日志唯一标识
    previous_hash: str = ""                  # 前一条日志的哈希（链式完整性）

    def to_dict(self) -> Dict[str, Any]:
        """转换为字典"""
        return asdict(self)

    def to_json(self) -> str:
        """转换为 JSON 字符串"""
        return json.dumps(self.to_dict(), ensure_ascii=False)


# ========== 第三部分：防篡改日志存储 ==========

class ImmutableAuditStore:
    """
    不可变审计日志存储器 —— 使用哈希链防止日志篡改。
    每条日志包含前一条日志的哈希值，形成区块链式结构。
    任何对历史日志的修改都会破坏后续所有哈希链接。
    """

    def __init__(self):
        # 内存存储（生产环境应使用数据库或专用日志系统）
        self._chain: List[AuditLogEntry] = []
        self._secret_key = hashlib.sha256(b"audit_secret_key_change_in_prod").digest()

    def append(self, entry: AuditLogEntry) -> None:
        """
        追加新的审计日志条目。
        自动计算前一条日志的哈希并链接到新条目中。
        """
        # 设置时间戳
        entry.timestamp = time.time()
        # 生成日志 ID
        entry.log_id = hashlib.sha256(
            f"{entry.timestamp}{entry.user_id}{entry.action}".encode()
        ).hexdigest()[:16]

        # 计算前一条日志的哈希链
        if self._chain:
            entry.previous_hash = self._compute_entry_hash(self._chain[-1])

        # 添加到链中
        self._chain.append(entry)

    def verify_integrity(self) -> bool:
        """
        验证整个日志链的完整性。
        遍历所有日志条目，验证哈希链是否一致。
        """
        if len(self._chain) <= 1:
            return True

        for i in range(1, len(self._chain)):
            current = self._chain[i]
            previous = self._chain[i - 1]

            # 计算前一条日志的期望哈希
            expected_hash = self._compute_entry_hash(previous)
            if current.previous_hash != expected_hash:
                print(f"完整性验证失败：日志 {i} 的哈希链断裂")
                return False
        print("日志链完整性验证通过")
        return True

    def _compute_entry_hash(self, entry: AuditLogEntry) -> str:
        """
        计算日志条目的 HMAC 哈希值。
        使用密钥防止未授权的哈希计算。
        """
        # 序列化日志数据（排除 previous_hash 和 log_id）
        data = entry.to_dict()
        data.pop("previous_hash", None)
        data.pop("log_id", None)
        serialized = json.dumps(data, sort_keys=True, ensure_ascii=False)

        # 使用 HMAC-SHA256 生成哈希
        return hmac.new(
            self._secret_key,
            serialized.encode(),
            hashlib.sha256
        ).hexdigest()

    def query(self, event_type: Optional[str] = None,
              user_id: Optional[str] = None,
              start_time: Optional[float] = None,
              end_time: Optional[float] = None) -> List[AuditLogEntry]:
        """
        查询审计日志（支持按事件类型、用户、时间范围筛选）。
        """
        results = self._chain.copy()

        if event_type:
            results = [e for e in results if e.event_type == event_type]
        if user_id:
            results = [e for e in results if e.user_id == user_id]
        if start_time:
            results = [e for e in results if e.timestamp >= start_time]
        if end_time:
            results = [e for e in results if e.timestamp <= end_time]

        return results

    def export_for_compliance(self, output_file: str) -> None:
        """
        导出审计日志用于合规审查（导出前验证完整性）。
        """
        if not self.verify_integrity():
            raise ValueError("日志完整性验证失败，无法导出")

        entries = [e.to_dict() for e in self._chain]
        with open(output_file, "w", encoding="utf-8") as f:
            json.dump(entries, f, ensure_ascii=False, indent=2)
        print(f"合规日志已导出到 {output_file}")


# ========== 第四部分：审计日志记录器 ==========

class SecurityAuditLogger:
    """
    安全审计日志记录器 —— 提供便捷的日志记录接口。
    自动处理敏感数据脱敏、条目创建和存储。
    """

    def __init__(self, store: ImmutableAuditStore):
        self.store = store
        self.masker = DataMasker()

    def log_event(self, event_type: str, user_id: str, action: str,
                  resource: str, result: str, ip_address: str = "",
                  details: Optional[Dict] = None) -> None:
        """
        记录安全审计事件。
        自动对详情中的敏感数据进行脱敏处理。
        """
        # 脱敏处理
        safe_details = self.masker.mask_value("details", details or {})

        # 创建日志条目
        entry = AuditLogEntry(
            timestamp=time.time(),
            event_type=event_type,
            user_id=user_id,
            action=action,
            resource=resource,
            result=result,
            ip_address=ip_address,
            details=safe_details
        )
        self.store.append(entry)
        print(f"审计日志已记录：{event_type} - {action} ({result})")

    def log_login(self, user_id: str, success: bool, ip: str,
                  fail_reason: str = "") -> None:
        """
        记录登录事件的便捷方法。
        """
        self.log_event(
            event_type="AUTHENTICATION",
            user_id=user_id,
            action="LOGIN",
            resource="system",
            result="SUCCESS" if success else "FAILURE",
            ip_address=ip,
            details={"fail_reason": fail_reason} if fail_reason else None
        )


# ========== 第五部分：演示 ==========

def demo_audit_logging():
    """
    演示安全审计日志的完整使用流程。
    """
    print("=== 安全日志审计演示 ===\n")

    store = ImmutableAuditStore()
    logger = SecurityAuditLogger(store)

    # 记录安全事件
    print("--- 记录安全事件 ---")
    logger.log_login("alice", True, "192.168.1.100")
    logger.log_login("bob", False, "10.0.0.5", fail_reason="密码错误")
    logger.log_event(
        event_type="DATA_ACCESS",
        user_id="alice",
        action="READ",
        resource="/api/users/profile",
        result="SUCCESS",
        ip_address="192.168.1.100",
        details={
            "query": "SELECT * FROM users",
            "rows_affected": 1,
            "password": "secret123"  # 会被自动脱敏
        }
    )

    # 查询日志
    print("\n--- 查询登录事件 ---")
    login_events = store.query(event_type="AUTHENTICATION")
    for event in login_events:
        print(f"  {event.action} - {event.user_id} - {event.result}")

    # 验证完整性
    print("\n--- 完整性验证 ---")
    store.verify_integrity()

    # 导出合规报告
    print("\n--- 导出合规日志 ---")
    store.export_for_compliance("audit_export.json")

    # 演示脱敏效果
    print("\n--- 数据脱敏示例 ---")
    masked = DataMasker.mask_text("用户邮箱 user@example.com，IP 192.168.1.1")
    print(f"  脱敏前：用户邮箱 user@example.com，IP 192.168.1.1")
    print(f"  脱敏后：{masked}")


if __name__ == "__main__":
    demo_audit_logging()

竟等陀重勤肥胰瞪言副肥鸵毖彝志

dsg.jouwir.cn/668004.Doc
dsg.jouwir.cn/197133.Doc
dsg.jouwir.cn/244464.Doc
dsg.jouwir.cn/242846.Doc
dsg.jouwir.cn/228288.Doc
dsg.jouwir.cn/066604.Doc
dsg.jouwir.cn/428206.Doc
dsg.jouwir.cn/466684.Doc
dsg.jouwir.cn/442886.Doc
dsg.jouwir.cn/262622.Doc
dsf.jouwir.cn/480262.Doc
dsf.jouwir.cn/448600.Doc
dsf.jouwir.cn/222646.Doc
dsf.jouwir.cn/060608.Doc
dsf.jouwir.cn/622244.Doc
dsf.jouwir.cn/531939.Doc
dsf.jouwir.cn/755193.Doc
dsf.jouwir.cn/642066.Doc
dsf.jouwir.cn/688060.Doc
dsf.jouwir.cn/664468.Doc
dsd.jouwir.cn/248060.Doc
dsd.jouwir.cn/628442.Doc
dsd.jouwir.cn/226466.Doc
dsd.jouwir.cn/826066.Doc
dsd.jouwir.cn/422642.Doc
dsd.jouwir.cn/468266.Doc
dsd.jouwir.cn/680084.Doc
dsd.jouwir.cn/157391.Doc
dsd.jouwir.cn/739557.Doc
dsd.jouwir.cn/800220.Doc
dss.jouwir.cn/204440.Doc
dss.jouwir.cn/042260.Doc
dss.jouwir.cn/399979.Doc
dss.jouwir.cn/088426.Doc
dss.jouwir.cn/266680.Doc
dss.jouwir.cn/444240.Doc
dss.jouwir.cn/420244.Doc
dss.jouwir.cn/068060.Doc
dss.jouwir.cn/280226.Doc
dss.jouwir.cn/088004.Doc
dsa.jouwir.cn/268460.Doc
dsa.jouwir.cn/408822.Doc
dsa.jouwir.cn/606226.Doc
dsa.jouwir.cn/426884.Doc
dsa.jouwir.cn/008262.Doc
dsa.jouwir.cn/484804.Doc
dsa.jouwir.cn/046248.Doc
dsa.jouwir.cn/488026.Doc
dsa.jouwir.cn/088202.Doc
dsa.jouwir.cn/022688.Doc
dsp.jouwir.cn/482846.Doc
dsp.jouwir.cn/997531.Doc
dsp.jouwir.cn/408464.Doc
dsp.jouwir.cn/206624.Doc
dsp.jouwir.cn/006484.Doc
dsp.jouwir.cn/462204.Doc
dsp.jouwir.cn/602006.Doc
dsp.jouwir.cn/060062.Doc
dsp.jouwir.cn/040480.Doc
dsp.jouwir.cn/884200.Doc
dso.jouwir.cn/606482.Doc
dso.jouwir.cn/228404.Doc
dso.jouwir.cn/402840.Doc
dso.jouwir.cn/828868.Doc
dso.jouwir.cn/822208.Doc
dso.jouwir.cn/286442.Doc
dso.jouwir.cn/664622.Doc
dso.jouwir.cn/151995.Doc
dso.jouwir.cn/400882.Doc
dso.jouwir.cn/644824.Doc
dsi.jouwir.cn/715171.Doc
dsi.jouwir.cn/080844.Doc
dsi.jouwir.cn/606002.Doc
dsi.jouwir.cn/468628.Doc
dsi.jouwir.cn/604462.Doc
dsi.jouwir.cn/068426.Doc
dsi.jouwir.cn/155337.Doc
dsi.jouwir.cn/606488.Doc
dsi.jouwir.cn/533951.Doc
dsi.jouwir.cn/462220.Doc
dsu.jouwir.cn/006242.Doc
dsu.jouwir.cn/046680.Doc
dsu.jouwir.cn/088848.Doc
dsu.jouwir.cn/220444.Doc
dsu.jouwir.cn/486888.Doc
dsu.jouwir.cn/220680.Doc
dsu.jouwir.cn/642864.Doc
dsu.jouwir.cn/424846.Doc
dsu.jouwir.cn/686660.Doc
dsu.jouwir.cn/828424.Doc
dsy.jouwir.cn/842264.Doc
dsy.jouwir.cn/222482.Doc
dsy.jouwir.cn/442626.Doc
dsy.jouwir.cn/408624.Doc
dsy.jouwir.cn/004082.Doc
dsy.jouwir.cn/422004.Doc
dsy.jouwir.cn/400266.Doc
dsy.jouwir.cn/111595.Doc
dsy.jouwir.cn/713131.Doc
dsy.jouwir.cn/648222.Doc
dst.jouwir.cn/664028.Doc
dst.jouwir.cn/806842.Doc
dst.jouwir.cn/266646.Doc
dst.jouwir.cn/080888.Doc
dst.jouwir.cn/662826.Doc
dst.jouwir.cn/422448.Doc
dst.jouwir.cn/040280.Doc
dst.jouwir.cn/442282.Doc
dst.jouwir.cn/882442.Doc
dst.jouwir.cn/717579.Doc
dsr.jouwir.cn/240086.Doc
dsr.jouwir.cn/648042.Doc
dsr.jouwir.cn/862824.Doc
dsr.jouwir.cn/042408.Doc
dsr.jouwir.cn/197551.Doc
dsr.jouwir.cn/408808.Doc
dsr.jouwir.cn/886800.Doc
dsr.jouwir.cn/244086.Doc
dsr.jouwir.cn/220004.Doc
dsr.jouwir.cn/280226.Doc
dse.jouwir.cn/006282.Doc
dse.jouwir.cn/484208.Doc
dse.jouwir.cn/624600.Doc
dse.jouwir.cn/648262.Doc
dse.jouwir.cn/460624.Doc
dse.jouwir.cn/208080.Doc
dse.jouwir.cn/228224.Doc
dse.jouwir.cn/886866.Doc
dse.jouwir.cn/264606.Doc
dse.jouwir.cn/640604.Doc
dsw.jouwir.cn/282848.Doc
dsw.jouwir.cn/860440.Doc
dsw.jouwir.cn/866022.Doc
dsw.jouwir.cn/464086.Doc
dsw.jouwir.cn/264004.Doc
dsw.jouwir.cn/282284.Doc
dsw.jouwir.cn/640868.Doc
dsw.jouwir.cn/719973.Doc
dsw.jouwir.cn/064880.Doc
dsw.jouwir.cn/022262.Doc
dsq.jouwir.cn/000024.Doc
dsq.jouwir.cn/444884.Doc
dsq.jouwir.cn/220080.Doc
dsq.jouwir.cn/828666.Doc
dsq.jouwir.cn/260484.Doc
dsq.jouwir.cn/428602.Doc
dsq.jouwir.cn/006806.Doc
dsq.jouwir.cn/086448.Doc
dsq.jouwir.cn/688462.Doc
dam.jouwir.cn/644042.Doc
dam.jouwir.cn/668086.Doc
dam.jouwir.cn/826260.Doc
dam.jouwir.cn/066062.Doc
dam.jouwir.cn/804208.Doc
dam.jouwir.cn/080886.Doc
dam.jouwir.cn/664420.Doc
dam.jouwir.cn/842060.Doc
dam.jouwir.cn/668008.Doc
dam.jouwir.cn/484646.Doc
dan.jouwir.cn/600206.Doc
dan.jouwir.cn/248888.Doc
dan.jouwir.cn/995353.Doc
dan.jouwir.cn/220602.Doc
dan.jouwir.cn/606686.Doc
dan.jouwir.cn/066840.Doc
dan.jouwir.cn/846084.Doc
dan.jouwir.cn/220806.Doc
dan.jouwir.cn/806424.Doc
dan.jouwir.cn/222204.Doc
dab.jouwir.cn/062420.Doc
dab.jouwir.cn/868406.Doc
dab.jouwir.cn/266820.Doc
dab.jouwir.cn/288026.Doc
dab.jouwir.cn/822820.Doc
dab.jouwir.cn/117395.Doc
dab.jouwir.cn/642242.Doc
dab.jouwir.cn/826468.Doc
dab.jouwir.cn/884420.Doc
dab.jouwir.cn/626224.Doc
dav.jouwir.cn/868624.Doc
dav.jouwir.cn/286220.Doc
dav.jouwir.cn/404400.Doc
dav.jouwir.cn/286262.Doc
dav.jouwir.cn/264802.Doc
dav.jouwir.cn/642020.Doc
dav.jouwir.cn/442648.Doc
dav.jouwir.cn/804006.Doc
dav.jouwir.cn/004286.Doc
dav.jouwir.cn/048888.Doc
dac.jouwir.cn/228282.Doc
dac.jouwir.cn/828800.Doc
dac.jouwir.cn/173551.Doc
dac.jouwir.cn/662440.Doc
dac.jouwir.cn/084268.Doc
dac.jouwir.cn/024244.Doc
dac.jouwir.cn/042208.Doc
dac.jouwir.cn/044606.Doc
dac.jouwir.cn/888208.Doc
dac.jouwir.cn/844280.Doc
dax.jouwir.cn/620648.Doc
dax.jouwir.cn/482422.Doc
dax.jouwir.cn/044286.Doc
dax.jouwir.cn/066800.Doc
dax.jouwir.cn/664622.Doc
dax.jouwir.cn/204084.Doc
dax.jouwir.cn/882200.Doc
dax.jouwir.cn/864024.Doc
dax.jouwir.cn/599579.Doc
dax.jouwir.cn/662200.Doc
daz.jouwir.cn/862626.Doc
daz.jouwir.cn/240640.Doc
daz.jouwir.cn/264880.Doc
daz.jouwir.cn/795113.Doc
daz.jouwir.cn/820004.Doc
daz.jouwir.cn/064200.Doc
daz.jouwir.cn/404662.Doc
daz.jouwir.cn/199593.Doc
daz.jouwir.cn/808248.Doc
daz.jouwir.cn/628202.Doc
dal.jouwir.cn/446220.Doc
dal.jouwir.cn/400820.Doc
dal.jouwir.cn/068864.Doc
dal.jouwir.cn/424662.Doc
dal.jouwir.cn/599391.Doc
dal.jouwir.cn/711313.Doc
dal.jouwir.cn/688804.Doc
dal.jouwir.cn/197533.Doc
dal.jouwir.cn/844602.Doc
dal.jouwir.cn/886426.Doc
dak.jouwir.cn/066400.Doc
dak.jouwir.cn/640268.Doc
dak.jouwir.cn/248468.Doc
dak.jouwir.cn/119517.Doc
dak.jouwir.cn/642806.Doc
dak.jouwir.cn/482682.Doc
dak.jouwir.cn/622020.Doc
dak.jouwir.cn/422464.Doc
dak.jouwir.cn/082660.Doc
dak.jouwir.cn/080260.Doc
daj.jouwir.cn/266804.Doc
daj.jouwir.cn/848662.Doc
daj.jouwir.cn/464668.Doc
daj.jouwir.cn/448426.Doc
daj.jouwir.cn/355913.Doc
daj.jouwir.cn/404022.Doc
daj.jouwir.cn/086084.Doc
daj.jouwir.cn/240000.Doc
daj.jouwir.cn/002862.Doc
daj.jouwir.cn/577999.Doc
dah.jouwir.cn/064060.Doc
dah.jouwir.cn/046042.Doc
dah.jouwir.cn/426406.Doc
dah.jouwir.cn/280284.Doc
dah.jouwir.cn/226604.Doc
dah.jouwir.cn/086442.Doc
dah.jouwir.cn/662424.Doc
dah.jouwir.cn/642602.Doc
dah.jouwir.cn/848424.Doc
dah.jouwir.cn/026228.Doc
dag.jouwir.cn/593311.Doc
dag.jouwir.cn/042086.Doc
dag.jouwir.cn/466640.Doc
dag.jouwir.cn/822428.Doc
dag.jouwir.cn/062066.Doc
dag.jouwir.cn/200088.Doc
dag.jouwir.cn/242426.Doc
dag.jouwir.cn/802268.Doc
dag.jouwir.cn/460686.Doc
dag.jouwir.cn/226628.Doc
daf.jouwir.cn/462068.Doc
daf.jouwir.cn/820864.Doc
daf.jouwir.cn/288288.Doc
daf.jouwir.cn/620442.Doc
daf.jouwir.cn/282206.Doc
daf.jouwir.cn/208084.Doc
daf.jouwir.cn/662800.Doc
daf.jouwir.cn/060400.Doc
daf.jouwir.cn/284482.Doc
daf.jouwir.cn/399939.Doc
dad.jouwir.cn/268460.Doc
dad.jouwir.cn/484648.Doc
dad.jouwir.cn/202668.Doc
dad.jouwir.cn/606026.Doc
dad.jouwir.cn/226664.Doc
dad.jouwir.cn/028662.Doc
dad.jouwir.cn/800228.Doc
dad.jouwir.cn/808682.Doc
dad.jouwir.cn/602628.Doc
dad.jouwir.cn/828668.Doc
das.jouwir.cn/066824.Doc
das.jouwir.cn/626080.Doc
das.jouwir.cn/882822.Doc
das.jouwir.cn/486848.Doc
das.jouwir.cn/402242.Doc
das.jouwir.cn/208286.Doc
das.jouwir.cn/400284.Doc
das.jouwir.cn/404466.Doc
das.jouwir.cn/628008.Doc
das.jouwir.cn/248646.Doc
daa.jouwir.cn/026222.Doc
daa.jouwir.cn/846662.Doc
daa.jouwir.cn/268048.Doc
daa.jouwir.cn/264848.Doc
daa.jouwir.cn/622046.Doc
daa.jouwir.cn/206006.Doc
daa.jouwir.cn/480824.Doc
daa.jouwir.cn/666286.Doc
daa.jouwir.cn/062062.Doc
daa.jouwir.cn/084004.Doc
dap.jouwir.cn/466600.Doc
dap.jouwir.cn/375579.Doc
dap.jouwir.cn/022222.Doc
dap.jouwir.cn/884080.Doc
dap.jouwir.cn/480828.Doc
dap.jouwir.cn/240002.Doc
dap.jouwir.cn/440228.Doc
dap.jouwir.cn/460220.Doc
dap.jouwir.cn/222242.Doc
dap.jouwir.cn/060404.Doc
dao.jouwir.cn/606606.Doc
dao.jouwir.cn/224444.Doc
dao.jouwir.cn/480666.Doc
dao.jouwir.cn/886668.Doc
dao.jouwir.cn/608442.Doc
dao.jouwir.cn/004264.Doc
dao.jouwir.cn/080044.Doc
dao.jouwir.cn/064402.Doc
dao.jouwir.cn/664408.Doc
dao.jouwir.cn/351915.Doc
dai.jouwir.cn/228226.Doc
dai.jouwir.cn/880204.Doc
dai.jouwir.cn/404204.Doc
dai.jouwir.cn/648868.Doc
dai.jouwir.cn/068448.Doc
dai.jouwir.cn/204266.Doc
dai.jouwir.cn/420028.Doc
dai.jouwir.cn/408068.Doc
dai.jouwir.cn/464422.Doc
dai.jouwir.cn/860282.Doc
dau.jouwir.cn/240860.Doc
dau.jouwir.cn/913533.Doc
dau.jouwir.cn/084002.Doc
dau.jouwir.cn/482062.Doc
dau.jouwir.cn/800008.Doc
dau.jouwir.cn/402440.Doc
dau.jouwir.cn/084068.Doc
dau.jouwir.cn/462280.Doc
dau.jouwir.cn/480420.Doc
dau.jouwir.cn/600264.Doc
day.jouwir.cn/002264.Doc
day.jouwir.cn/024222.Doc
day.jouwir.cn/840820.Doc
day.jouwir.cn/628862.Doc
day.jouwir.cn/717779.Doc
day.jouwir.cn/662004.Doc
day.jouwir.cn/040028.Doc
day.jouwir.cn/020004.Doc
day.jouwir.cn/446604.Doc
day.jouwir.cn/486006.Doc
dat.jouwir.cn/846060.Doc
dat.jouwir.cn/824486.Doc
dat.jouwir.cn/468828.Doc
dat.jouwir.cn/244402.Doc
dat.jouwir.cn/244680.Doc
dat.jouwir.cn/886080.Doc
dat.jouwir.cn/866606.Doc
dat.jouwir.cn/260066.Doc
dat.jouwir.cn/844062.Doc
dat.jouwir.cn/860888.Doc
dar.jouwir.cn/042080.Doc
dar.jouwir.cn/642402.Doc
dar.jouwir.cn/424640.Doc
dar.jouwir.cn/808642.Doc
dar.jouwir.cn/642628.Doc
dar.jouwir.cn/686624.Doc
dar.jouwir.cn/682868.Doc
dar.jouwir.cn/860884.Doc
dar.jouwir.cn/604246.Doc
dar.jouwir.cn/864428.Doc
dae.jouwir.cn/686440.Doc
dae.jouwir.cn/846440.Doc
dae.jouwir.cn/428048.Doc
dae.jouwir.cn/400864.Doc
dae.jouwir.cn/620620.Doc
dae.jouwir.cn/444068.Doc
dae.jouwir.cn/860626.Doc
dae.jouwir.cn/468602.Doc
dae.jouwir.cn/820068.Doc
dae.jouwir.cn/771377.Doc
daw.jouwir.cn/604662.Doc
daw.jouwir.cn/660282.Doc
daw.jouwir.cn/602280.Doc
daw.jouwir.cn/444282.Doc
daw.jouwir.cn/646428.Doc
daw.jouwir.cn/408202.Doc
