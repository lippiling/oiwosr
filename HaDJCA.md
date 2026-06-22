兑爬套瞻孔


"""
Python 安全序列化 —— 规避 Pickle 风险，使用安全替代方案
涵盖 pickle 漏洞演示、JSON/msgpack 安全序列化、itsdangerous 签名加密
"""

# 安装依赖：pip install msgpack itsdangerous
# 序列化安全是 Python 安全中常被忽视但后果严重的领域

import pickle
import json
import msgpack
import hashlib
import hmac
import os
from typing import Any, Dict, Optional
from itsdangerous import URLSafeSerializer, TimedSerializer, BadSignature

# ========== 第一部分：Pickle 安全风险 ==========

class UnsafePickleDemo:
    """
    Pickle 反序列化漏洞演示。
    pickle.loads() 可以执行任意 Python 代码，这就是为什么
    永远不要对不可信数据使用 pickle 反序列化。
    """

    @staticmethod
    def create_malicious_payload() -> bytes:
        """
        创建一个恶意的 pickle 载荷。
        实际攻击中，攻击者会让服务器反序列化此载荷以执行恶意代码。
        """
        # 使用 __reduce__ 定义反序列化时执行的代码
        class EvilPickle:
            def __reduce__(self):
                # 反序列化时将执行 os.system 命令
                import os
                cmd = "echo '攻击成功！反序列化执行了任意命令'"
                return (os.system, (cmd,))

        evil = EvilPickle()
        malicious_data = pickle.dumps(evil)
        print(f"恶意 pickle 载荷已生成（{len(malicious_data)} 字节）")
        return malicious_data

    @staticmethod
    def dangerous_deserialize(data: bytes) -> Any:
        """
        不安全的反序列化 —— 直接调用 pickle.loads。
        这就是生产代码中绝对不能做的事情！
        """
        print("警告：正在反序列化不可信的 pickle 数据！")
        obj = pickle.loads(data)  # 这里会执行任意代码
        return obj


# ========== 第二部分：安全的序列化替代方案 ==========

class SafeSerialization:
    """
    安全的序列化方案 —— 使用不执行代码的格式。
    JSON 和 MessagePack 都是纯数据格式，不会执行代码。
    """

    @staticmethod
    def serialize_json(data: Any) -> str:
        """
        使用 JSON 进行安全序列化。
        JSON 是最通用的跨语言序列化格式。
        """
        return json.dumps(data, ensure_ascii=False, default=str)

    @staticmethod
    def deserialize_json(data: str) -> Any:
        """
        安全反序列化 JSON 数据。
        """
        return json.loads(data)

    @staticmethod
    def serialize_msgpack(data: Any) -> bytes:
        """
        使用 MessagePack 进行安全序列化。
        MessagePack 类似 JSON 但更紧凑，适合高性能场景。
        """
        return msgpack.packb(data)

    @staticmethod
    def deserialize_msgpack(data: bytes) -> Any:
        """
        安全反序列化 MessagePack 数据。
        """
        return msgpack.unpackb(data)

    @staticmethod
    def compare_formats(data: Dict) -> None:
        """
        比较不同序列化格式的效率和大小。
        """
        json_data = SafeSerialization.serialize_json(data)
        msgpack_data = SafeSerialization.serialize_msgpack(data)

        print(f"原始数据大小：{len(str(data))} 字节")
        print(f"JSON 序列化大小：{len(json_data)} 字节")
        print(f"MessagePack 序列化大小：{len(msgpack_data)} 字节")
        print(f"MessagePack 节省了 {len(json_data) - len(msgpack_data)} 字节")


# ========== 第三部分：带签名和加密的安全序列化 ==========

class SecureSerializer:
    """
    安全序列化器 —— 使用 itsdangerous 库对数据进行签名和加密。
    itsdangerous 提供：
    - 签名：防止数据被篡改
    - 时间戳：防止重放攻击
    - 加密：保护数据机密性
    """

    def __init__(self, secret_key: str):
        """
        初始化安全序列化器。
        secret_key 必须是高熵的密钥（建议使用 secrets.token_hex() 生成）。
        """
        self.secret_key = secret_key
        # URL 安全序列化器：对数据进行签名
        self.url_serializer = URLSafeSerializer(secret_key)
        # 时间戳序列化器：带过期时间的签名
        self.timed_serializer = TimedSerializer(secret_key)

    def sign_data(self, data: Any) -> str:
        """
        对数据进行签名，防止篡改。
        签名后的数据以 base64 编码的字符串形式返回。
        """
        # URLSafeSerializer 自动处理 JSON 序列化和签名
        signed = self.url_serializer.dumps(data)
        print(f"数据已签名（长度：{len(signed)} 字符）")
        return signed

    def verify_signed_data(self, signed_data: str) -> Optional[Any]:
        """
        验证签名并还原数据。
        如果签名无效（数据被篡改），抛出 BadSignature 异常。
        """
        try:
            data = self.url_serializer.loads(signed_data)
            print("签名验证通过，数据完整未被篡改")
            return data
        except BadSignature:
            print("警告：数据签名无效，数据可能被篡改！")
            return None

    def sign_with_expiry(self, data: Any,
                         max_age_seconds: int = 3600) -> str:
        """
        带过期时间的签名。
        适用于需要限时使用的令牌（如密码重置链接）。
        """
        signed = self.timed_serializer.dumps(data)
        print(f"数据已签名（有效期 {max_age_seconds} 秒）")
        return signed

    def verify_with_expiry(self, signed_data: str,
                           max_age_seconds: int = 3600) -> Optional[Any]:
        """
        验证签名并检查是否过期。
        """
        try:
            data = self.timed_serializer.loads(
                signed_data, max_age=max_age_seconds
            )
            print("签名验证通过，且在有效期内")
            return data
        except BadSignature:
            print("警告：签名无效或已过期！")
            return None


class SignEncryptSerializer:
    """
    签名 + 加密的序列化方案 —— 自己实现 HMAC 签名。
    确保数据的完整性和机密性。
    """

    def __init__(self, signing_key: str, encryption_key: Optional[str] = None):
        self.signing_key = signing_key.encode()
        self.encryption_key = encryption_key

    def serialize_secure(self, data: Any) -> bytes:
        """
        安全序列化：先编码 -> 再加密（可选）-> 最后签名。
        先签名后加密可以防止签名碰撞攻击。
        """
        # 第一步：序列化为 JSON
        payload = json.dumps(data, ensure_ascii=False).encode()

        # 第二步：计算 HMAC 签名
        signature = hmac.new(
            self.signing_key, payload, hashlib.sha256
        ).hexdigest()

        # 第三步：组合签名和数据
        result = json.dumps({
            "signature": signature,
            "payload": payload.decode()
        }).encode()
        return result

    def deserialize_secure(self, data: bytes) -> Optional[Any]:
        """
        安全反序列化：先验证签名 -> 再解密（可选）-> 最后解码。
        """
        try:
            # 第一步：解析外层 JSON
            wrapper = json.loads(data.decode())

            # 第二步：提取签名和载荷
            expected_sig = wrapper["signature"]
            payload_str = wrapper["payload"]
            payload = payload_str.encode()

            # 第三步：验证签名
            actual_sig = hmac.new(
                self.signing_key, payload, hashlib.sha256
            ).hexdigest()

            # 使用 hmac.compare_digest 防止时序攻击
            if not hmac.compare_digest(expected_sig, actual_sig):
                print("警告：HMAC 签名不匹配，数据被篡改！")
                return None

            # 第四步：解码 JSON
            return json.loads(payload)

        except (json.JSONDecodeError, KeyError, Exception) as e:
            print(f"反序列化失败：{e}")
            return None


# ========== 第四部分：Notrust 反序列化模式 ==========

class NotrustDeserializer:
    """
    "不信任"反序列化模式 —— 处理不可信数据时的安全最佳实践。
    即使使用安全的格式，也要验证数据的结构和内容。
    """

    @staticmethod
    def safe_deserialize(data: str,
                         expected_schema: Optional[Dict] = None) -> Optional[Dict]:
        """
        安全的反序列化：验证类型、结构和内容后再使用。
        """
        try:
            # 第一步：使用安全格式解析
            obj = json.loads(data)

            # 第二步：验证类型
            if not isinstance(obj, dict):
                print("错误：期望 JSON 对象，得到其他类型")
                return None

            # 第三步：验证必填字段
            if expected_schema:
                for field, field_type in expected_schema.items():
                    if field not in obj:
                        print(f"错误：缺少必填字段 '{field}'")
                        return None
                    if not isinstance(obj[field], field_type):
                        print(f"错误：字段 '{field}' 类型错误")
                        return None

            # 第四步：验证值范围
            for key, value in obj.items():
                if isinstance(value, str) and len(value) > 1000:
                    print(f"错误：字段 '{key}' 超过最大长度")
                    return None
                if isinstance(value, (int, float)) and abs(value) > 1e9:
                    print(f"错误：字段 '{key}' 数值超出范围")
                    return None

            print("反序列化验证通过")
            return obj

        except json.JSONDecodeError:
            print("错误：无效的 JSON 格式")
            return None


# ========== 第五部分：演示 ==========

def demo_secure_serialization():
    """
    演示各种序列化的安全特性和风险。
    """
    print("=== 安全序列化演示 ===\n")

    # 1. Pickle 风险演示
    print("--- Pickle 反序列化风险 ---")
    malicious = UnsafePickleDemo.create_malicious_payload()
    print("注意：在生产代码中绝不要对不可信数据调用 pickle.loads()")

    # 2. 安全格式比较
    print("\n--- 序列化格式比较 ---")
    test_data = {
        "user": "alice",
        "scores": [95, 87, 92],
        "metadata": {"version": "1.0", "timestamp": 1234567890}
    }
    SafeSerialization.compare_formats(test_data)

    # 3. itsdangerous 签名
    print("\n--- 带签名的安全序列化 ---")
    serializer = SecureSerializer(
        secret_key="my-secret-key-change-in-production"
    )
    user_data = {"user_id": 123, "role": "admin"}
    signed = serializer.sign_data(user_data)
    verified = serializer.verify_signed_data(signed)
    print(f"还原数据：{verified}")

    # 4. 篡改检测演示
    print("\n--- 篡改检测演示 ---")
    tampered = signed[:-1] + ("X" if signed[-1] != "X" else signed[-2])
    result = serializer.verify_signed_data(tampered)
    print(f"篡改数据还原结果：{result}")

    # 5. Schema 验证
    print("\n--- Notrust 反序列化验证 ---")
    safe_data = '{"name": "Alice", "age": 30}'
    schema = {"name": str, "age": int}
    result = NotrustDeserializer.safe_deserialize(safe_data, schema)
    print(f"验证通过的数据：{result}")

    # 6. 序列化安全建议
    print("\n--- 序列化安全最佳实践 ---")
    print("""
    1. 永远不要对不可信数据使用 pickle.loads()
    2. 优先使用 JSON 序列化（纯数据，不执行代码）
    3. 对序列化数据进行 HMAC 签名以防止篡改
    4. itsdangerous 提供了简洁的签名+序列化方案
    5. 反序列化后验证数据结构和值范围
    6. 敏感数据在序列化前应加密
    """)


if __name__ == "__main__":
    demo_secure_serialization()

缘蜗蛔饺葱士勺徊冻椅幕岩兄掏酉

amh.vwbnt.cn/464808.Doc
amh.vwbnt.cn/088684.Doc
amh.vwbnt.cn/200842.Doc
amg.vwbnt.cn/604244.Doc
amg.vwbnt.cn/264484.Doc
amg.vwbnt.cn/262404.Doc
amg.vwbnt.cn/822462.Doc
amg.vwbnt.cn/686264.Doc
amg.vwbnt.cn/046208.Doc
amg.vwbnt.cn/062666.Doc
amg.vwbnt.cn/446264.Doc
amg.vwbnt.cn/600426.Doc
amg.vwbnt.cn/006844.Doc
amf.vwbnt.cn/068444.Doc
amf.vwbnt.cn/628888.Doc
amf.vwbnt.cn/246866.Doc
amf.vwbnt.cn/442640.Doc
amf.vwbnt.cn/084822.Doc
amf.vwbnt.cn/606622.Doc
amf.vwbnt.cn/886684.Doc
amf.vwbnt.cn/028862.Doc
amf.vwbnt.cn/822660.Doc
amf.vwbnt.cn/428424.Doc
amd.vwbnt.cn/404060.Doc
amd.vwbnt.cn/446260.Doc
amd.vwbnt.cn/002244.Doc
amd.vwbnt.cn/080242.Doc
amd.vwbnt.cn/608204.Doc
amd.vwbnt.cn/066840.Doc
amd.vwbnt.cn/444648.Doc
amd.vwbnt.cn/268606.Doc
amd.vwbnt.cn/682642.Doc
amd.vwbnt.cn/622462.Doc
ams.vwbnt.cn/424828.Doc
ams.vwbnt.cn/220684.Doc
ams.vwbnt.cn/864082.Doc
ams.vwbnt.cn/086484.Doc
ams.vwbnt.cn/226022.Doc
ams.vwbnt.cn/228606.Doc
ams.vwbnt.cn/248468.Doc
ams.vwbnt.cn/846488.Doc
ams.vwbnt.cn/440446.Doc
ams.vwbnt.cn/248608.Doc
ama.vwbnt.cn/264244.Doc
ama.vwbnt.cn/800008.Doc
ama.vwbnt.cn/046404.Doc
ama.vwbnt.cn/840262.Doc
ama.vwbnt.cn/644264.Doc
ama.vwbnt.cn/880684.Doc
ama.vwbnt.cn/028402.Doc
ama.vwbnt.cn/882686.Doc
ama.vwbnt.cn/800664.Doc
ama.vwbnt.cn/646482.Doc
amp.vwbnt.cn/024064.Doc
amp.vwbnt.cn/806266.Doc
amp.vwbnt.cn/266826.Doc
amp.vwbnt.cn/826400.Doc
amp.vwbnt.cn/886046.Doc
amp.vwbnt.cn/262404.Doc
amp.vwbnt.cn/428002.Doc
amp.vwbnt.cn/480288.Doc
amp.vwbnt.cn/248206.Doc
amp.vwbnt.cn/444268.Doc
amo.vwbnt.cn/646448.Doc
amo.vwbnt.cn/220248.Doc
amo.vwbnt.cn/426260.Doc
amo.vwbnt.cn/886480.Doc
amo.vwbnt.cn/686684.Doc
amo.vwbnt.cn/060848.Doc
amo.vwbnt.cn/606600.Doc
amo.vwbnt.cn/482486.Doc
amo.vwbnt.cn/220806.Doc
amo.vwbnt.cn/488266.Doc
ami.vwbnt.cn/064864.Doc
ami.vwbnt.cn/864222.Doc
ami.vwbnt.cn/046486.Doc
ami.vwbnt.cn/286444.Doc
ami.vwbnt.cn/020002.Doc
ami.vwbnt.cn/884820.Doc
ami.vwbnt.cn/026686.Doc
ami.vwbnt.cn/820448.Doc
ami.vwbnt.cn/286068.Doc
ami.vwbnt.cn/820266.Doc
amu.vwbnt.cn/024064.Doc
amu.vwbnt.cn/888602.Doc
amu.vwbnt.cn/226226.Doc
amu.vwbnt.cn/426006.Doc
amu.vwbnt.cn/662666.Doc
amu.vwbnt.cn/864806.Doc
amu.vwbnt.cn/806224.Doc
amu.vwbnt.cn/684220.Doc
amu.vwbnt.cn/468886.Doc
amu.vwbnt.cn/468822.Doc
amy.vwbnt.cn/640628.Doc
amy.vwbnt.cn/088224.Doc
amy.vwbnt.cn/283355.Doc
amy.vwbnt.cn/020882.Doc
amy.vwbnt.cn/264840.Doc
amy.vwbnt.cn/820062.Doc
amy.vwbnt.cn/046468.Doc
amy.vwbnt.cn/048020.Doc
amy.vwbnt.cn/169416.Doc
amy.vwbnt.cn/006608.Doc
amt.vwbnt.cn/666006.Doc
amt.vwbnt.cn/450715.Doc
amt.vwbnt.cn/600000.Doc
amt.vwbnt.cn/064464.Doc
amt.vwbnt.cn/466044.Doc
amt.vwbnt.cn/686820.Doc
amt.vwbnt.cn/624480.Doc
amt.vwbnt.cn/802420.Doc
amt.vwbnt.cn/004066.Doc
amt.vwbnt.cn/400606.Doc
amr.vwbnt.cn/404264.Doc
amr.vwbnt.cn/000004.Doc
amr.vwbnt.cn/604846.Doc
amr.vwbnt.cn/220082.Doc
amr.vwbnt.cn/226468.Doc
amr.vwbnt.cn/426064.Doc
amr.vwbnt.cn/486284.Doc
amr.vwbnt.cn/682248.Doc
amr.vwbnt.cn/068824.Doc
amr.vwbnt.cn/008686.Doc
ame.vwbnt.cn/428462.Doc
ame.vwbnt.cn/060066.Doc
ame.vwbnt.cn/008606.Doc
ame.vwbnt.cn/602848.Doc
ame.vwbnt.cn/200460.Doc
ame.vwbnt.cn/086462.Doc
ame.vwbnt.cn/666862.Doc
ame.vwbnt.cn/404420.Doc
ame.vwbnt.cn/806642.Doc
ame.vwbnt.cn/228020.Doc
amw.vwbnt.cn/408880.Doc
amw.vwbnt.cn/420206.Doc
amw.vwbnt.cn/042666.Doc
amw.vwbnt.cn/680246.Doc
amw.vwbnt.cn/466066.Doc
amw.vwbnt.cn/260844.Doc
amw.vwbnt.cn/446064.Doc
amw.vwbnt.cn/406242.Doc
amw.vwbnt.cn/068888.Doc
amw.vwbnt.cn/002600.Doc
amq.vwbnt.cn/284080.Doc
amq.vwbnt.cn/206666.Doc
amq.vwbnt.cn/622226.Doc
amq.vwbnt.cn/668866.Doc
amq.vwbnt.cn/486822.Doc
amq.vwbnt.cn/840460.Doc
amq.vwbnt.cn/866622.Doc
amq.vwbnt.cn/480060.Doc
amq.vwbnt.cn/604486.Doc
amq.vwbnt.cn/648260.Doc
anm.vwbnt.cn/846684.Doc
anm.vwbnt.cn/204266.Doc
anm.vwbnt.cn/226800.Doc
anm.vwbnt.cn/844264.Doc
anm.vwbnt.cn/086606.Doc
anm.vwbnt.cn/620060.Doc
anm.vwbnt.cn/206404.Doc
anm.vwbnt.cn/486666.Doc
anm.vwbnt.cn/428620.Doc
anm.vwbnt.cn/069409.Doc
ann.vwbnt.cn/000462.Doc
ann.vwbnt.cn/026408.Doc
ann.vwbnt.cn/826842.Doc
ann.vwbnt.cn/442288.Doc
ann.vwbnt.cn/806440.Doc
ann.vwbnt.cn/488026.Doc
ann.vwbnt.cn/428686.Doc
ann.vwbnt.cn/642864.Doc
ann.vwbnt.cn/262088.Doc
ann.vwbnt.cn/020484.Doc
anb.vwbnt.cn/280008.Doc
anb.vwbnt.cn/228246.Doc
anb.vwbnt.cn/080660.Doc
anb.vwbnt.cn/042866.Doc
anb.vwbnt.cn/862402.Doc
anb.vwbnt.cn/644486.Doc
anb.vwbnt.cn/602626.Doc
anb.vwbnt.cn/002860.Doc
anb.vwbnt.cn/804460.Doc
anb.vwbnt.cn/642002.Doc
anv.vwbnt.cn/757773.Doc
anv.vwbnt.cn/462644.Doc
anv.vwbnt.cn/040248.Doc
anv.vwbnt.cn/260468.Doc
anv.vwbnt.cn/226642.Doc
anv.vwbnt.cn/866666.Doc
anv.vwbnt.cn/800284.Doc
anv.vwbnt.cn/770106.Doc
anv.vwbnt.cn/424600.Doc
anv.vwbnt.cn/286420.Doc
anc.vwbnt.cn/464228.Doc
anc.vwbnt.cn/820644.Doc
anc.vwbnt.cn/488668.Doc
anc.vwbnt.cn/866464.Doc
anc.vwbnt.cn/206882.Doc
anc.vwbnt.cn/222662.Doc
anc.vwbnt.cn/466428.Doc
anc.vwbnt.cn/222888.Doc
anc.vwbnt.cn/600024.Doc
anc.vwbnt.cn/864844.Doc
anx.vwbnt.cn/628064.Doc
anx.vwbnt.cn/404682.Doc
anx.vwbnt.cn/028422.Doc
anx.vwbnt.cn/802040.Doc
anx.vwbnt.cn/008204.Doc
anx.vwbnt.cn/042208.Doc
anx.vwbnt.cn/165284.Doc
anx.vwbnt.cn/826040.Doc
anx.vwbnt.cn/408240.Doc
anx.vwbnt.cn/808624.Doc
anz.vwbnt.cn/440060.Doc
anz.vwbnt.cn/846406.Doc
anz.vwbnt.cn/080882.Doc
anz.vwbnt.cn/464068.Doc
anz.vwbnt.cn/442802.Doc
anz.vwbnt.cn/442646.Doc
anz.vwbnt.cn/242886.Doc
anz.vwbnt.cn/168789.Doc
anz.vwbnt.cn/360254.Doc
anz.vwbnt.cn/468604.Doc
anl.vwbnt.cn/486246.Doc
anl.vwbnt.cn/082620.Doc
anl.vwbnt.cn/626886.Doc
anl.vwbnt.cn/040482.Doc
anl.vwbnt.cn/446842.Doc
anl.vwbnt.cn/088644.Doc
anl.vwbnt.cn/551090.Doc
anl.vwbnt.cn/286480.Doc
anl.vwbnt.cn/626606.Doc
anl.vwbnt.cn/408844.Doc
ank.vwbnt.cn/204442.Doc
ank.vwbnt.cn/692742.Doc
ank.vwbnt.cn/884448.Doc
ank.vwbnt.cn/424846.Doc
ank.vwbnt.cn/848204.Doc
ank.vwbnt.cn/204664.Doc
ank.vwbnt.cn/086004.Doc
ank.vwbnt.cn/028842.Doc
ank.vwbnt.cn/080882.Doc
ank.vwbnt.cn/484424.Doc
anj.vwbnt.cn/208840.Doc
anj.vwbnt.cn/062806.Doc
anj.vwbnt.cn/144562.Doc
anj.vwbnt.cn/406068.Doc
anj.vwbnt.cn/242460.Doc
anj.vwbnt.cn/600020.Doc
anj.vwbnt.cn/646664.Doc
anj.vwbnt.cn/486404.Doc
anj.vwbnt.cn/830609.Doc
anj.vwbnt.cn/602628.Doc
anh.vwbnt.cn/004642.Doc
anh.vwbnt.cn/860820.Doc
anh.vwbnt.cn/624028.Doc
anh.vwbnt.cn/888608.Doc
anh.vwbnt.cn/442604.Doc
anh.vwbnt.cn/808828.Doc
anh.vwbnt.cn/486844.Doc
anh.vwbnt.cn/430406.Doc
anh.vwbnt.cn/022408.Doc
anh.vwbnt.cn/820662.Doc
ang.vwbnt.cn/406802.Doc
ang.vwbnt.cn/428646.Doc
ang.vwbnt.cn/484268.Doc
ang.vwbnt.cn/600064.Doc
ang.vwbnt.cn/606242.Doc
ang.vwbnt.cn/082042.Doc
ang.vwbnt.cn/400422.Doc
ang.vwbnt.cn/424062.Doc
ang.vwbnt.cn/000802.Doc
ang.vwbnt.cn/648026.Doc
anf.vwbnt.cn/080262.Doc
anf.vwbnt.cn/022202.Doc
anf.vwbnt.cn/808224.Doc
anf.vwbnt.cn/060824.Doc
anf.vwbnt.cn/604002.Doc
anf.vwbnt.cn/680482.Doc
anf.vwbnt.cn/751062.Doc
anf.vwbnt.cn/620242.Doc
anf.vwbnt.cn/820824.Doc
and.vwbnt.cn/426048.Doc
and.vwbnt.cn/824248.Doc
and.vwbnt.cn/264404.Doc
and.vwbnt.cn/826808.Doc
and.vwbnt.cn/422488.Doc
and.vwbnt.cn/682602.Doc
and.vwbnt.cn/688602.Doc
and.vwbnt.cn/680882.Doc
and.vwbnt.cn/206717.Doc
and.vwbnt.cn/864868.Doc
ans.vwbnt.cn/864244.Doc
ans.vwbnt.cn/686022.Doc
ans.vwbnt.cn/006608.Doc
ans.vwbnt.cn/488624.Doc
ans.vwbnt.cn/800440.Doc
ans.vwbnt.cn/246460.Doc
ans.vwbnt.cn/600484.Doc
ans.vwbnt.cn/571012.Doc
ans.vwbnt.cn/046802.Doc
ans.vwbnt.cn/622228.Doc
ana.vwbnt.cn/363776.Doc
ana.vwbnt.cn/086448.Doc
ana.vwbnt.cn/000282.Doc
ana.vwbnt.cn/862264.Doc
ana.vwbnt.cn/222800.Doc
ana.vwbnt.cn/464402.Doc
ana.vwbnt.cn/620660.Doc
ana.vwbnt.cn/620008.Doc
ana.vwbnt.cn/488424.Doc
ana.vwbnt.cn/444400.Doc
anp.vwbnt.cn/408842.Doc
anp.vwbnt.cn/688024.Doc
anp.vwbnt.cn/844420.Doc
anp.vwbnt.cn/264400.Doc
anp.vwbnt.cn/240260.Doc
anp.vwbnt.cn/484844.Doc
anp.vwbnt.cn/084868.Doc
anp.vwbnt.cn/464864.Doc
anp.vwbnt.cn/608824.Doc
anp.vwbnt.cn/426048.Doc
ano.vwbnt.cn/840224.Doc
ano.vwbnt.cn/404648.Doc
ano.vwbnt.cn/640646.Doc
ano.vwbnt.cn/204082.Doc
ano.vwbnt.cn/808466.Doc
ano.vwbnt.cn/424888.Doc
ano.vwbnt.cn/046468.Doc
ano.vwbnt.cn/088600.Doc
ano.vwbnt.cn/448868.Doc
ano.vwbnt.cn/008204.Doc
ani.vwbnt.cn/848464.Doc
ani.vwbnt.cn/604866.Doc
ani.vwbnt.cn/204824.Doc
ani.vwbnt.cn/246424.Doc
ani.vwbnt.cn/488848.Doc
ani.vwbnt.cn/000822.Doc
ani.vwbnt.cn/880466.Doc
ani.vwbnt.cn/662048.Doc
ani.vwbnt.cn/646848.Doc
ani.vwbnt.cn/824840.Doc
anu.vwbnt.cn/604240.Doc
anu.vwbnt.cn/153055.Doc
anu.vwbnt.cn/306846.Doc
anu.vwbnt.cn/486088.Doc
anu.vwbnt.cn/406428.Doc
anu.vwbnt.cn/080400.Doc
anu.vwbnt.cn/884608.Doc
anu.vwbnt.cn/068686.Doc
anu.vwbnt.cn/046420.Doc
anu.vwbnt.cn/088824.Doc
any.vwbnt.cn/208028.Doc
any.vwbnt.cn/206228.Doc
any.vwbnt.cn/828668.Doc
any.vwbnt.cn/660622.Doc
any.vwbnt.cn/482026.Doc
any.vwbnt.cn/228804.Doc
any.vwbnt.cn/662408.Doc
any.vwbnt.cn/224769.Doc
any.vwbnt.cn/660666.Doc
any.vwbnt.cn/268248.Doc
ant.vwbnt.cn/020240.Doc
ant.vwbnt.cn/482262.Doc
ant.vwbnt.cn/440268.Doc
ant.vwbnt.cn/420240.Doc
ant.vwbnt.cn/626286.Doc
ant.vwbnt.cn/680862.Doc
ant.vwbnt.cn/044488.Doc
ant.vwbnt.cn/084840.Doc
ant.vwbnt.cn/064406.Doc
ant.vwbnt.cn/248448.Doc
anr.vwbnt.cn/480440.Doc
anr.vwbnt.cn/002866.Doc
anr.vwbnt.cn/420620.Doc
anr.vwbnt.cn/400426.Doc
anr.vwbnt.cn/084600.Doc
anr.vwbnt.cn/448042.Doc
anr.vwbnt.cn/322676.Doc
anr.vwbnt.cn/826262.Doc
anr.vwbnt.cn/620802.Doc
anr.vwbnt.cn/046448.Doc
ane.vwbnt.cn/866420.Doc
ane.vwbnt.cn/266664.Doc
ane.vwbnt.cn/288428.Doc
ane.vwbnt.cn/626024.Doc
ane.vwbnt.cn/664202.Doc
ane.vwbnt.cn/800682.Doc
ane.vwbnt.cn/024862.Doc
ane.vwbnt.cn/444646.Doc
ane.vwbnt.cn/800020.Doc
ane.vwbnt.cn/002042.Doc
anw.vwbnt.cn/842248.Doc
anw.vwbnt.cn/246684.Doc
anw.vwbnt.cn/866066.Doc
