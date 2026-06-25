Python TOTP 双因素认证实战
=============================

双因素认证（2FA）为账户增加一层安全保障。TOTP（基于时间的一次性密码）
是最流行的 2FA 方案，Google Authenticator 和 Authy 都支持此标准。

1. 安装依赖
------------

# pip install pyotp qrcode pillow

import pyotp
import qrcode
import qrcode.image.svg
import io
import base64
import time
from typing import List, Optional

2. TOTP 基础：生成密钥与验证
-----------------------------

def generate_totp_secret() -> str:
    """生成 TOTP 共享密钥（Base32 编码）"""
    # pyotp.random_base32() 生成一个随机的 Base32 密钥
    # 长度默认 32 字符（160 位熵）
    secret = pyotp.random_base32()
    return secret

def create_totp(secret: str) -> pyotp.TOTP:
    """使用密钥创建 TOTP 对象"""
    # TOTP 默认使用 SHA1、30 秒时间窗口、6 位数字
    return pyotp.TOTP(secret)

# 演示：生成密钥并获取当前一次性密码
secret = generate_totp_secret()
print(f"TOTP 密钥: {secret}")

totp = create_totp(secret)
current_code = totp.now()  # 获取当前时间窗口的验证码
print(f"当前验证码: {current_code}")

# 等待几秒后验证（模拟用户输入）
time.sleep(1)
is_valid = totp.verify(current_code)
print(f"验证码有效: {is_valid}")

# 验证过期的验证码
expired_code = totp.at(for_time=int(time.time()) - 120)  # 2 分钟前
print(f"过期验证码: {expired_code}, 有效: {totp.verify(expired_code)}")

3. TOTP 参数定制
-----------------

def create_custom_totp(secret: str) -> pyotp.TOTP:
    """创建自定义参数的 TOTP 实例"""
    return pyotp.TOTP(
        secret,
        digits=8,         # 验证码位数（默认 6）
        interval=60,      # 时间窗口（秒，默认 30）
        digest=pyotp.SHA256  # 哈希算法（默认 SHA1）
    )

custom_totp = create_custom_totp(secret)
print(f"自定义 TOTP: {custom_totp.now()} (8 位, 60 秒)")

4. HOTP（基于计数器的哈希）
---------------------------

# HOTP 使用计数器而非时间，每次验证成功后计数器递增
def create_hotp(secret: str, initial_count: int = 0) -> pyotp.HOTP:
    """创建 HOTP 实例"""
    return pyotp.HOTP(secret, initial_count=initial_count)

hotp = create_hotp(secret)
# 生成第 1、2、3 个验证码
for i in range(3):
    code = hotp.at(i)
    print(f"HOTP 第 {i+1} 次: {code}")
    # 验证时会自动使用当前计数器
    print(f"  验证: {hotp.verify(code, counter=i)}")

# HOTP 适用于离线场景或无法同步时间的设备
# 但需要维护计数器同步，比 TOTP 更复杂

5. 生成二维码（用于用户绑定）
------------------------------

def generate_qr_code_uri(secret: str, username: str,
                         issuer: str = "MyApp") -> str:
    """生成 otpauth URI 供二维码使用"""
    # 标准格式: otpauth://totp/ISSUER:USERNAME?secret=SECRET&issuer=ISSUER
    totp = pyotp.TOTP(secret)
    uri = totp.provisioning_uri(name=username, issuer_name=issuer)
    return uri

def create_qr_code_image(uri: str) -> str:
    """生成二维码并返回 Base64 编码的 SVG"""
    qr = qrcode.make(uri, image_factory=qrcode.image.svg.SvgImage)
    buf = io.BytesIO()
    qr.save(buf)
    svg_data = buf.getvalue().decode()
    # 转为 HTML 可内联的 Base64（实际使用）
    b64 = base64.b64encode(buf.getvalue()).decode()
    return f"data:image/svg+xml;base64,{b64}"

# 生成用户的绑定 URI
uri = generate_qr_code_uri(secret, "user@example.com", "MySecureApp")
print(f"Provisioning URI:")
print(f"  {uri}")

# 二维码生成（实际部署时显示给用户扫描）
# 用户使用 Google Authenticator / Authy 扫码即可完成绑定

6. 验证流程（服务端核心逻辑）
-----------------------------

class TOTPAuthenticator:
    """TOTP 双因素认证管理器"""

    def __init__(self, secret: str):
        self.secret = secret
        self.totp = pyotp.TOTP(secret)
        # 允许前后各 1 个时间窗口的偏移（共 3 个窗口）
        self.allowed_drift = 1

    def verify_code(self, code: str) -> bool:
        """
        验证用户输入的验证码，允许一定的时间偏移。
        valid_window 参数允许前后各 N 个时间窗口的偏移
        """
        return self.totp.verify(code, valid_window=self.allowed_drift)

    def get_current_code(self) -> str:
        """获取当前验证码（仅用于调试或测试）"""
        return self.totp.now()

# 模拟完整的验证流程
auth = TOTPAuthenticator(secret)
user_input = auth.get_current_code()
print(f"\n用户输入验证码: {user_input}")
print(f"服务端验证结果: {auth.verify_code(user_input)}")

7. 备份码（Recovery Codes）
---------------------------

def generate_backup_codes(count: int = 8) -> List[str]:
    """生成一次性备份恢复码"""
    import secrets
    codes = []
    for _ in range(count):
        # 生成 10 位十六进制编码
        code = secrets.token_hex(5).upper()
        # 格式化为分组形式，便于输入
        formatted = f"{code[:5]}-{code[5:]}"
        codes.append(formatted)
    return codes

def store_backup_codes(codes: List[str]) -> List[dict]:
    """返回带哈希的备份码用于存储"""
    import hashlib
    stored = []
    for code in codes:
        # 存储哈希值而非明文（一次验证后立即销毁）
        code_hash = hashlib.sha256(code.encode()).hexdigest()
        stored.append({"hash": code_hash, "used": False})
    return stored

# 生成 8 个备份码
backup_codes = generate_backup_codes(8)
print("\n备份恢复码（请立即保存到安全位置）:")
for i, code in enumerate(backup_codes, 1):
    print(f"  {i}. {code}")

# 备份码应显示一次后就不可再查看
# 每个备份码只能使用一次，用过后立即作废

8. 时间偏移处理
----------------

def handle_time_drift(secret: str, user_code: str) -> dict:
    """
    处理客户端与服务器之间的时间偏差。
    某些手机时间可能不准确，导致验证失败。
    """
    totp = pyotp.TOTP(secret)
    # pyotp 的 verify 已内置 valid_window 参数
    # 但可以手动检测偏量用于日志
    expected_time = int(time.time())
    matched = False
    for offset in range(-3, 4):
        check_time = expected_time + (offset * 30)
        if totp.at(for_time=check_time) == user_code:
            matched = True
            print(f"  [调试] 在偏移 {offset} 个窗口处匹配")
            break

    return {
        "verified": matched,
        "server_time": expected_time,
        "drift_detected": matched
    }

# 任何成功验证都可以记录当前时间偏移，用于后续优化
drift_result = handle_time_drift(secret, auth.get_current_code())
print(f"\n时间偏移检测: {drift_result}")

9. 生产环境集成注意事项
------------------------

def totp_setup_for_new_user(username: str) -> dict:
    """为新用户初始化 TOTP 的完整流程"""
    # 1. 生成密钥
    secret = generate_totp_secret()

    # 2. 生成二维码 URI
    uri = generate_qr_code_uri(secret, username)

    # 3. 生成备份码
    recovery_codes = generate_backup_codes(8)

    # 4. 返回设置信息（实际项目中需要绑定到用户）
    return {
        "secret": secret,            # 存入用户表的 totp_secret 字段
        "qr_uri": uri,              # 显示给用户扫码
        "recovery_codes": recovery_codes,  # 显示给用户保存
        "enabled": False             # 待用户验证后才启用
    }

def verify_and_enable_totp(secret: str, user_code: str) -> bool:
    """用户首次设置时，验证一次以确认绑定成功"""
    totp = pyotp.TOTP(secret)
    return totp.verify(user_code)

# 示例流程
new_user = totp_setup_for_new_user("alice@example.com")
print(f"\n新用户 TOTP 设置完成，密钥: {new_user['secret']}")

10. 安全建议
------------

# 1. 密钥传输必须使用 HTTPS
# 2. 二维码只在首次设置时显示一次
# 3. 备份码显示后应立即存储到安全位置
# 4. 设置速率限制防止暴力枚举 6 位验证码（百万分之一概率）
# 5. 监控异常的大量验证失败尝试
# 6. 记录 TOTP 验证成功的设备信息和 IP
# 7. 支持用户撤销所有已绑定的 TOTP 设备

11. 总结
---------

TOTP 为应用提供了强大的第二层认证。pyotp 库实现简洁，与主流
验证器应用完全兼容。结合备份码和合理的时间偏移容错，可以构建
安全可靠的 2FA 系统，显著提升账户安全性。

eer.yghdsxcnbza.cn/46266.Doc
eer.yghdsxcnbza.cn/06888.Doc
eer.yghdsxcnbza.cn/80400.Doc
eer.yghdsxcnbza.cn/66408.Doc
eer.yghdsxcnbza.cn/08820.Doc
eer.yghdsxcnbza.cn/20088.Doc
eer.yghdsxcnbza.cn/82260.Doc
eer.yghdsxcnbza.cn/26688.Doc
eer.yghdsxcnbza.cn/82680.Doc
eer.yghdsxcnbza.cn/40008.Doc
eee.yghdsxcnbza.cn/20226.Doc
eee.yghdsxcnbza.cn/22242.Doc
eee.yghdsxcnbza.cn/26022.Doc
eee.yghdsxcnbza.cn/24686.Doc
eee.yghdsxcnbza.cn/08604.Doc
eee.yghdsxcnbza.cn/26622.Doc
eee.yghdsxcnbza.cn/00480.Doc
eee.yghdsxcnbza.cn/22420.Doc
eee.yghdsxcnbza.cn/24042.Doc
eee.yghdsxcnbza.cn/24462.Doc
eew.yghdsxcnbza.cn/26804.Doc
eew.yghdsxcnbza.cn/44844.Doc
eew.yghdsxcnbza.cn/62468.Doc
eew.yghdsxcnbza.cn/66064.Doc
eew.yghdsxcnbza.cn/62828.Doc
eew.yghdsxcnbza.cn/75935.Doc
eew.yghdsxcnbza.cn/80840.Doc
eew.yghdsxcnbza.cn/26880.Doc
eew.yghdsxcnbza.cn/40204.Doc
eew.yghdsxcnbza.cn/62466.Doc
eeq.yghdsxcnbza.cn/82660.Doc
eeq.yghdsxcnbza.cn/31317.Doc
eeq.yghdsxcnbza.cn/42840.Doc
eeq.yghdsxcnbza.cn/48280.Doc
eeq.yghdsxcnbza.cn/66400.Doc
eeq.yghdsxcnbza.cn/66280.Doc
eeq.yghdsxcnbza.cn/08842.Doc
eeq.yghdsxcnbza.cn/48284.Doc
eeq.yghdsxcnbza.cn/42628.Doc
eeq.yghdsxcnbza.cn/08060.Doc
ewm.yghdsxcnbza.cn/82804.Doc
ewm.yghdsxcnbza.cn/68226.Doc
ewm.yghdsxcnbza.cn/86068.Doc
ewm.yghdsxcnbza.cn/66000.Doc
ewm.yghdsxcnbza.cn/84000.Doc
ewm.yghdsxcnbza.cn/15913.Doc
ewm.yghdsxcnbza.cn/66202.Doc
ewm.yghdsxcnbza.cn/22486.Doc
ewm.yghdsxcnbza.cn/42628.Doc
ewm.yghdsxcnbza.cn/71515.Doc
ewn.yghdsxcnbza.cn/68640.Doc
ewn.yghdsxcnbza.cn/40642.Doc
ewn.yghdsxcnbza.cn/28224.Doc
ewn.yghdsxcnbza.cn/53337.Doc
ewn.yghdsxcnbza.cn/68004.Doc
ewn.yghdsxcnbza.cn/62642.Doc
ewn.yghdsxcnbza.cn/26080.Doc
ewn.yghdsxcnbza.cn/28880.Doc
ewn.yghdsxcnbza.cn/40680.Doc
ewn.yghdsxcnbza.cn/60228.Doc
ewb.yghdsxcnbza.cn/40480.Doc
ewb.yghdsxcnbza.cn/40488.Doc
ewb.yghdsxcnbza.cn/86460.Doc
ewb.yghdsxcnbza.cn/06086.Doc
ewb.yghdsxcnbza.cn/60422.Doc
ewb.yghdsxcnbza.cn/40004.Doc
ewb.yghdsxcnbza.cn/26246.Doc
ewb.yghdsxcnbza.cn/86284.Doc
ewb.yghdsxcnbza.cn/04666.Doc
ewb.yghdsxcnbza.cn/68822.Doc
ewv.yghdsxcnbza.cn/46606.Doc
ewv.yghdsxcnbza.cn/08662.Doc
ewv.yghdsxcnbza.cn/48404.Doc
ewv.yghdsxcnbza.cn/73577.Doc
ewv.yghdsxcnbza.cn/46846.Doc
ewv.yghdsxcnbza.cn/40804.Doc
ewv.yghdsxcnbza.cn/84426.Doc
ewv.yghdsxcnbza.cn/59559.Doc
ewv.yghdsxcnbza.cn/62088.Doc
ewv.yghdsxcnbza.cn/24004.Doc
ewc.yghdsxcnbza.cn/06046.Doc
ewc.yghdsxcnbza.cn/02404.Doc
ewc.yghdsxcnbza.cn/71551.Doc
ewc.yghdsxcnbza.cn/04206.Doc
ewc.yghdsxcnbza.cn/80688.Doc
ewc.yghdsxcnbza.cn/73155.Doc
ewc.yghdsxcnbza.cn/68400.Doc
ewc.yghdsxcnbza.cn/02426.Doc
ewc.yghdsxcnbza.cn/80446.Doc
ewc.yghdsxcnbza.cn/60642.Doc
ewx.yghdsxcnbza.cn/24260.Doc
ewx.yghdsxcnbza.cn/86066.Doc
ewx.yghdsxcnbza.cn/42228.Doc
ewx.yghdsxcnbza.cn/24664.Doc
ewx.yghdsxcnbza.cn/66608.Doc
ewx.yghdsxcnbza.cn/86806.Doc
ewx.yghdsxcnbza.cn/31995.Doc
ewx.yghdsxcnbza.cn/80882.Doc
ewx.yghdsxcnbza.cn/64068.Doc
ewx.yghdsxcnbza.cn/40884.Doc
ewz.yghdsxcnbza.cn/22266.Doc
ewz.yghdsxcnbza.cn/48286.Doc
ewz.yghdsxcnbza.cn/22082.Doc
ewz.yghdsxcnbza.cn/82622.Doc
ewz.yghdsxcnbza.cn/33353.Doc
ewz.yghdsxcnbza.cn/82888.Doc
ewz.yghdsxcnbza.cn/88602.Doc
ewz.yghdsxcnbza.cn/28240.Doc
ewz.yghdsxcnbza.cn/48804.Doc
ewz.yghdsxcnbza.cn/24608.Doc
ewl.yghdsxcnbza.cn/82060.Doc
ewl.yghdsxcnbza.cn/24624.Doc
ewl.yghdsxcnbza.cn/99117.Doc
ewl.yghdsxcnbza.cn/04668.Doc
ewl.yghdsxcnbza.cn/80842.Doc
ewl.yghdsxcnbza.cn/06048.Doc
ewl.yghdsxcnbza.cn/66402.Doc
ewl.yghdsxcnbza.cn/00484.Doc
ewl.yghdsxcnbza.cn/84284.Doc
ewl.yghdsxcnbza.cn/48806.Doc
ewk.yghdsxcnbza.cn/86684.Doc
ewk.yghdsxcnbza.cn/24066.Doc
ewk.yghdsxcnbza.cn/82086.Doc
ewk.yghdsxcnbza.cn/44244.Doc
ewk.yghdsxcnbza.cn/28822.Doc
ewk.yghdsxcnbza.cn/86664.Doc
ewk.yghdsxcnbza.cn/48848.Doc
ewk.yghdsxcnbza.cn/60840.Doc
ewk.yghdsxcnbza.cn/55979.Doc
ewk.yghdsxcnbza.cn/08088.Doc
ewj.yghdsxcnbza.cn/06868.Doc
ewj.yghdsxcnbza.cn/13751.Doc
ewj.yghdsxcnbza.cn/62662.Doc
ewj.yghdsxcnbza.cn/46208.Doc
ewj.yghdsxcnbza.cn/48808.Doc
ewj.yghdsxcnbza.cn/22060.Doc
ewj.yghdsxcnbza.cn/20628.Doc
ewj.yghdsxcnbza.cn/39515.Doc
ewj.yghdsxcnbza.cn/46240.Doc
ewj.yghdsxcnbza.cn/08486.Doc
ewh.yghdsxcnbza.cn/44284.Doc
ewh.yghdsxcnbza.cn/04882.Doc
ewh.yghdsxcnbza.cn/59999.Doc
ewh.yghdsxcnbza.cn/46246.Doc
ewh.yghdsxcnbza.cn/48224.Doc
ewh.yghdsxcnbza.cn/20002.Doc
ewh.yghdsxcnbza.cn/64684.Doc
ewh.yghdsxcnbza.cn/46886.Doc
ewh.yghdsxcnbza.cn/00440.Doc
ewh.yghdsxcnbza.cn/44662.Doc
ewg.yghdsxcnbza.cn/22820.Doc
ewg.yghdsxcnbza.cn/42044.Doc
ewg.yghdsxcnbza.cn/64646.Doc
ewg.yghdsxcnbza.cn/02248.Doc
ewg.yghdsxcnbza.cn/28402.Doc
ewg.yghdsxcnbza.cn/86824.Doc
ewg.yghdsxcnbza.cn/13911.Doc
ewg.yghdsxcnbza.cn/46086.Doc
ewg.yghdsxcnbza.cn/66644.Doc
ewg.yghdsxcnbza.cn/20444.Doc
ewf.yghdsxcnbza.cn/28062.Doc
ewf.yghdsxcnbza.cn/02228.Doc
ewf.yghdsxcnbza.cn/62086.Doc
ewf.yghdsxcnbza.cn/68044.Doc
ewf.yghdsxcnbza.cn/04660.Doc
ewf.yghdsxcnbza.cn/88020.Doc
ewf.yghdsxcnbza.cn/42880.Doc
ewf.yghdsxcnbza.cn/44204.Doc
ewf.yghdsxcnbza.cn/91331.Doc
ewf.yghdsxcnbza.cn/04800.Doc
ewd.yghdsxcnbza.cn/48406.Doc
ewd.yghdsxcnbza.cn/60844.Doc
ewd.yghdsxcnbza.cn/86640.Doc
ewd.yghdsxcnbza.cn/22486.Doc
ewd.yghdsxcnbza.cn/66824.Doc
ewd.yghdsxcnbza.cn/08060.Doc
ewd.yghdsxcnbza.cn/22024.Doc
ewd.yghdsxcnbza.cn/60684.Doc
ewd.yghdsxcnbza.cn/02884.Doc
ewd.yghdsxcnbza.cn/86680.Doc
ews.yghdsxcnbza.cn/24026.Doc
ews.yghdsxcnbza.cn/35739.Doc
ews.yghdsxcnbza.cn/66624.Doc
ews.yghdsxcnbza.cn/60246.Doc
ews.yghdsxcnbza.cn/40420.Doc
ews.yghdsxcnbza.cn/24244.Doc
ews.yghdsxcnbza.cn/82604.Doc
ews.yghdsxcnbza.cn/26284.Doc
ews.yghdsxcnbza.cn/68646.Doc
ews.yghdsxcnbza.cn/02004.Doc
ewa.yghdsxcnbza.cn/55995.Doc
ewa.yghdsxcnbza.cn/39557.Doc
ewa.yghdsxcnbza.cn/26088.Doc
ewa.yghdsxcnbza.cn/84002.Doc
ewa.yghdsxcnbza.cn/88684.Doc
ewa.yghdsxcnbza.cn/48224.Doc
ewa.yghdsxcnbza.cn/37957.Doc
ewa.yghdsxcnbza.cn/40686.Doc
ewa.yghdsxcnbza.cn/46264.Doc
ewa.yghdsxcnbza.cn/46200.Doc
ewp.yghdsxcnbza.cn/44228.Doc
ewp.yghdsxcnbza.cn/24664.Doc
ewp.yghdsxcnbza.cn/20202.Doc
ewp.yghdsxcnbza.cn/39173.Doc
ewp.yghdsxcnbza.cn/44224.Doc
ewp.yghdsxcnbza.cn/26244.Doc
ewp.yghdsxcnbza.cn/82600.Doc
ewp.yghdsxcnbza.cn/20806.Doc
ewp.yghdsxcnbza.cn/17555.Doc
ewp.yghdsxcnbza.cn/66444.Doc
ewo.yghdsxcnbza.cn/24422.Doc
ewo.yghdsxcnbza.cn/64242.Doc
ewo.yghdsxcnbza.cn/40864.Doc
ewo.yghdsxcnbza.cn/39737.Doc
ewo.yghdsxcnbza.cn/46866.Doc
ewo.yghdsxcnbza.cn/88464.Doc
ewo.yghdsxcnbza.cn/86626.Doc
ewo.yghdsxcnbza.cn/86080.Doc
ewo.yghdsxcnbza.cn/86886.Doc
ewo.yghdsxcnbza.cn/00264.Doc
ewi.yghdsxcnbza.cn/86826.Doc
ewi.yghdsxcnbza.cn/42662.Doc
ewi.yghdsxcnbza.cn/20204.Doc
ewi.yghdsxcnbza.cn/86248.Doc
ewi.yghdsxcnbza.cn/08024.Doc
ewi.yghdsxcnbza.cn/77977.Doc
ewi.yghdsxcnbza.cn/64806.Doc
ewi.yghdsxcnbza.cn/62048.Doc
ewi.yghdsxcnbza.cn/11935.Doc
ewi.yghdsxcnbza.cn/62020.Doc
ewu.yghdsxcnbza.cn/68066.Doc
ewu.yghdsxcnbza.cn/20048.Doc
ewu.yghdsxcnbza.cn/40240.Doc
ewu.yghdsxcnbza.cn/46280.Doc
ewu.yghdsxcnbza.cn/02844.Doc
ewu.yghdsxcnbza.cn/26228.Doc
ewu.yghdsxcnbza.cn/48866.Doc
ewu.yghdsxcnbza.cn/88642.Doc
ewu.yghdsxcnbza.cn/24422.Doc
ewu.yghdsxcnbza.cn/42882.Doc
ewy.yghdsxcnbza.cn/48446.Doc
ewy.yghdsxcnbza.cn/80424.Doc
ewy.yghdsxcnbza.cn/64002.Doc
ewy.yghdsxcnbza.cn/64644.Doc
ewy.yghdsxcnbza.cn/82822.Doc
ewy.yghdsxcnbza.cn/66280.Doc
ewy.yghdsxcnbza.cn/26284.Doc
ewy.yghdsxcnbza.cn/64668.Doc
ewy.yghdsxcnbza.cn/88628.Doc
ewy.yghdsxcnbza.cn/97717.Doc
ewt.yghdsxcnbza.cn/68642.Doc
ewt.yghdsxcnbza.cn/00400.Doc
ewt.yghdsxcnbza.cn/39933.Doc
ewt.yghdsxcnbza.cn/48646.Doc
ewt.yghdsxcnbza.cn/84062.Doc
ewt.yghdsxcnbza.cn/00280.Doc
ewt.yghdsxcnbza.cn/35173.Doc
ewt.yghdsxcnbza.cn/44448.Doc
ewt.yghdsxcnbza.cn/02024.Doc
ewt.yghdsxcnbza.cn/73175.Doc
ewr.yghdsxcnbza.cn/88048.Doc
ewr.yghdsxcnbza.cn/79111.Doc
ewr.yghdsxcnbza.cn/66046.Doc
ewr.yghdsxcnbza.cn/46268.Doc
ewr.yghdsxcnbza.cn/86040.Doc
ewr.yghdsxcnbza.cn/82240.Doc
ewr.yghdsxcnbza.cn/28080.Doc
ewr.yghdsxcnbza.cn/08282.Doc
ewr.yghdsxcnbza.cn/26020.Doc
ewr.yghdsxcnbza.cn/59373.Doc
ewe.yghdsxcnbza.cn/39195.Doc
ewe.yghdsxcnbza.cn/80288.Doc
ewe.yghdsxcnbza.cn/00242.Doc
ewe.yghdsxcnbza.cn/46002.Doc
ewe.yghdsxcnbza.cn/00208.Doc
ewe.yghdsxcnbza.cn/62846.Doc
ewe.yghdsxcnbza.cn/44006.Doc
ewe.yghdsxcnbza.cn/00066.Doc
ewe.yghdsxcnbza.cn/00686.Doc
ewe.yghdsxcnbza.cn/57555.Doc
eww.yghdsxcnbza.cn/60688.Doc
eww.yghdsxcnbza.cn/60606.Doc
eww.yghdsxcnbza.cn/40224.Doc
eww.yghdsxcnbza.cn/06400.Doc
eww.yghdsxcnbza.cn/60026.Doc
eww.yghdsxcnbza.cn/64242.Doc
eww.yghdsxcnbza.cn/40280.Doc
eww.yghdsxcnbza.cn/40624.Doc
eww.yghdsxcnbza.cn/46468.Doc
eww.yghdsxcnbza.cn/00880.Doc
ewq.yghdsxcnbza.cn/19937.Doc
ewq.yghdsxcnbza.cn/20228.Doc
ewq.yghdsxcnbza.cn/88064.Doc
ewq.yghdsxcnbza.cn/24000.Doc
ewq.yghdsxcnbza.cn/55179.Doc
ewq.yghdsxcnbza.cn/88000.Doc
ewq.yghdsxcnbza.cn/06286.Doc
ewq.yghdsxcnbza.cn/86482.Doc
ewq.yghdsxcnbza.cn/42824.Doc
ewq.yghdsxcnbza.cn/28628.Doc
eqm.yghdsxcnbza.cn/68608.Doc
eqm.yghdsxcnbza.cn/22222.Doc
eqm.yghdsxcnbza.cn/60480.Doc
eqm.yghdsxcnbza.cn/19973.Doc
eqm.yghdsxcnbza.cn/48244.Doc
eqm.yghdsxcnbza.cn/84468.Doc
eqm.yghdsxcnbza.cn/68048.Doc
eqm.yghdsxcnbza.cn/42428.Doc
eqm.yghdsxcnbza.cn/40004.Doc
eqm.yghdsxcnbza.cn/64228.Doc
eqn.yghdsxcnbza.cn/22862.Doc
eqn.yghdsxcnbza.cn/82464.Doc
eqn.yghdsxcnbza.cn/42268.Doc
eqn.yghdsxcnbza.cn/04046.Doc
eqn.yghdsxcnbza.cn/82020.Doc
eqn.yghdsxcnbza.cn/22666.Doc
eqn.yghdsxcnbza.cn/40646.Doc
eqn.yghdsxcnbza.cn/80488.Doc
eqn.yghdsxcnbza.cn/60006.Doc
eqn.yghdsxcnbza.cn/00602.Doc
eqb.yghdsxcnbza.cn/24068.Doc
eqb.yghdsxcnbza.cn/71599.Doc
eqb.yghdsxcnbza.cn/82822.Doc
eqb.yghdsxcnbza.cn/66040.Doc
eqb.yghdsxcnbza.cn/40242.Doc
eqb.yghdsxcnbza.cn/95759.Doc
eqb.yghdsxcnbza.cn/84846.Doc
eqb.yghdsxcnbza.cn/40464.Doc
eqb.yghdsxcnbza.cn/40028.Doc
eqb.yghdsxcnbza.cn/02262.Doc
eqv.yghdsxcnbza.cn/22824.Doc
eqv.yghdsxcnbza.cn/82488.Doc
eqv.yghdsxcnbza.cn/84806.Doc
eqv.yghdsxcnbza.cn/91391.Doc
eqv.yghdsxcnbza.cn/59359.Doc
eqv.yghdsxcnbza.cn/44224.Doc
eqv.yghdsxcnbza.cn/55573.Doc
eqv.yghdsxcnbza.cn/08606.Doc
eqv.yghdsxcnbza.cn/42462.Doc
eqv.yghdsxcnbza.cn/24868.Doc
eqc.yghdsxcnbza.cn/22802.Doc
eqc.yghdsxcnbza.cn/04462.Doc
eqc.yghdsxcnbza.cn/26620.Doc
eqc.yghdsxcnbza.cn/64468.Doc
eqc.yghdsxcnbza.cn/62086.Doc
eqc.yghdsxcnbza.cn/80088.Doc
eqc.yghdsxcnbza.cn/48462.Doc
eqc.yghdsxcnbza.cn/79555.Doc
eqc.yghdsxcnbza.cn/40400.Doc
eqc.yghdsxcnbza.cn/11179.Doc
eqx.yghdsxcnbza.cn/40688.Doc
eqx.yghdsxcnbza.cn/46848.Doc
eqx.yghdsxcnbza.cn/40040.Doc
eqx.yghdsxcnbza.cn/86062.Doc
eqx.yghdsxcnbza.cn/60228.Doc
eqx.yghdsxcnbza.cn/60844.Doc
eqx.yghdsxcnbza.cn/08662.Doc
eqx.yghdsxcnbza.cn/08642.Doc
eqx.yghdsxcnbza.cn/80268.Doc
eqx.yghdsxcnbza.cn/48888.Doc
eqz.yghdsxcnbza.cn/31757.Doc
eqz.yghdsxcnbza.cn/08200.Doc
eqz.yghdsxcnbza.cn/33579.Doc
eqz.yghdsxcnbza.cn/46886.Doc
eqz.yghdsxcnbza.cn/66002.Doc
eqz.yghdsxcnbza.cn/73717.Doc
eqz.yghdsxcnbza.cn/82062.Doc
eqz.yghdsxcnbza.cn/64240.Doc
eqz.yghdsxcnbza.cn/11197.Doc
eqz.yghdsxcnbza.cn/68846.Doc
eql.yghdsxcnbza.cn/73591.Doc
eql.yghdsxcnbza.cn/44422.Doc
eql.yghdsxcnbza.cn/66428.Doc
eql.yghdsxcnbza.cn/26426.Doc
eql.yghdsxcnbza.cn/99515.Doc
eql.yghdsxcnbza.cn/86680.Doc
eql.yghdsxcnbza.cn/08066.Doc
eql.yghdsxcnbza.cn/00088.Doc
eql.yghdsxcnbza.cn/22086.Doc
eql.yghdsxcnbza.cn/08488.Doc
eqk.yghdsxcnbza.cn/44848.Doc
eqk.yghdsxcnbza.cn/17575.Doc
eqk.yghdsxcnbza.cn/04624.Doc
eqk.yghdsxcnbza.cn/22604.Doc
eqk.yghdsxcnbza.cn/31973.Doc
eqk.yghdsxcnbza.cn/44842.Doc
eqk.yghdsxcnbza.cn/46040.Doc
eqk.yghdsxcnbza.cn/37939.Doc
eqk.yghdsxcnbza.cn/44064.Doc
eqk.yghdsxcnbza.cn/44026.Doc
eqj.yghdsxcnbza.cn/51593.Doc
eqj.yghdsxcnbza.cn/64880.Doc
eqj.yghdsxcnbza.cn/84428.Doc
eqj.yghdsxcnbza.cn/28202.Doc
eqj.yghdsxcnbza.cn/80882.Doc
eqj.yghdsxcnbza.cn/64208.Doc
eqj.yghdsxcnbza.cn/04844.Doc
eqj.yghdsxcnbza.cn/55379.Doc
eqj.yghdsxcnbza.cn/86244.Doc
eqj.yghdsxcnbza.cn/26644.Doc
eqh.yghdsxcnbza.cn/82640.Doc
eqh.yghdsxcnbza.cn/40840.Doc
eqh.yghdsxcnbza.cn/28826.Doc
eqh.yghdsxcnbza.cn/06864.Doc
eqh.yghdsxcnbza.cn/79151.Doc
eqh.yghdsxcnbza.cn/64624.Doc
eqh.yghdsxcnbza.cn/40222.Doc
eqh.yghdsxcnbza.cn/51959.Doc
eqh.yghdsxcnbza.cn/15115.Doc
eqh.yghdsxcnbza.cn/66022.Doc
eqg.yghdsxcnbza.cn/71591.Doc
eqg.yghdsxcnbza.cn/46486.Doc
eqg.yghdsxcnbza.cn/24086.Doc
eqg.yghdsxcnbza.cn/60202.Doc
eqg.yghdsxcnbza.cn/40200.Doc
eqg.yghdsxcnbza.cn/46220.Doc
eqg.yghdsxcnbza.cn/46466.Doc
eqg.yghdsxcnbza.cn/26866.Doc
eqg.yghdsxcnbza.cn/60664.Doc
eqg.yghdsxcnbza.cn/08606.Doc
eqf.yghdsxcnbza.cn/42884.Doc
eqf.yghdsxcnbza.cn/46864.Doc
eqf.yghdsxcnbza.cn/84688.Doc
eqf.yghdsxcnbza.cn/22064.Doc
eqf.yghdsxcnbza.cn/02608.Doc
eqf.yghdsxcnbza.cn/22202.Doc
eqf.yghdsxcnbza.cn/42466.Doc
eqf.yghdsxcnbza.cn/22862.Doc
eqf.yghdsxcnbza.cn/22266.Doc
eqf.yghdsxcnbza.cn/80844.Doc
eqd.yghdsxcnbza.cn/20066.Doc
eqd.yghdsxcnbza.cn/06242.Doc
eqd.yghdsxcnbza.cn/80886.Doc
eqd.yghdsxcnbza.cn/60428.Doc
eqd.yghdsxcnbza.cn/06866.Doc
eqd.yghdsxcnbza.cn/20804.Doc
eqd.yghdsxcnbza.cn/66820.Doc
eqd.yghdsxcnbza.cn/02242.Doc
eqd.yghdsxcnbza.cn/46068.Doc
eqd.yghdsxcnbza.cn/84886.Doc
eqs.yghdsxcnbza.cn/17751.Doc
eqs.yghdsxcnbza.cn/06028.Doc
eqs.yghdsxcnbza.cn/64288.Doc
eqs.yghdsxcnbza.cn/66806.Doc
eqs.yghdsxcnbza.cn/00460.Doc
eqs.yghdsxcnbza.cn/82684.Doc
eqs.yghdsxcnbza.cn/64644.Doc
eqs.yghdsxcnbza.cn/26822.Doc
eqs.yghdsxcnbza.cn/60284.Doc
eqs.yghdsxcnbza.cn/04864.Doc
eqa.yghdsxcnbza.cn/46480.Doc
eqa.yghdsxcnbza.cn/66646.Doc
eqa.yghdsxcnbza.cn/82002.Doc
eqa.yghdsxcnbza.cn/06406.Doc
eqa.yghdsxcnbza.cn/60680.Doc
eqa.yghdsxcnbza.cn/66260.Doc
eqa.yghdsxcnbza.cn/39173.Doc
eqa.yghdsxcnbza.cn/35553.Doc
eqa.yghdsxcnbza.cn/26662.Doc
eqa.yghdsxcnbza.cn/68242.Doc
eqp.yghdsxcnbza.cn/22286.Doc
eqp.yghdsxcnbza.cn/40480.Doc
eqp.yghdsxcnbza.cn/44260.Doc
eqp.yghdsxcnbza.cn/88028.Doc
eqp.yghdsxcnbza.cn/68808.Doc
eqp.yghdsxcnbza.cn/60868.Doc
eqp.yghdsxcnbza.cn/64280.Doc
eqp.yghdsxcnbza.cn/20284.Doc
eqp.yghdsxcnbza.cn/08222.Doc
eqp.yghdsxcnbza.cn/91731.Doc
eqo.yghdsxcnbza.cn/48426.Doc
eqo.yghdsxcnbza.cn/40880.Doc
eqo.yghdsxcnbza.cn/42468.Doc
eqo.yghdsxcnbza.cn/44004.Doc
eqo.yghdsxcnbza.cn/75133.Doc
eqo.yghdsxcnbza.cn/00064.Doc
eqo.yghdsxcnbza.cn/86464.Doc
eqo.yghdsxcnbza.cn/82066.Doc
eqo.yghdsxcnbza.cn/26466.Doc
eqo.yghdsxcnbza.cn/42068.Doc
eqi.yghdsxcnbza.cn/00280.Doc
eqi.yghdsxcnbza.cn/26066.Doc
eqi.yghdsxcnbza.cn/82608.Doc
eqi.yghdsxcnbza.cn/42620.Doc
eqi.yghdsxcnbza.cn/02020.Doc
eqi.yghdsxcnbza.cn/20884.Doc
eqi.yghdsxcnbza.cn/62668.Doc
eqi.yghdsxcnbza.cn/88444.Doc
eqi.yghdsxcnbza.cn/84440.Doc
eqi.yghdsxcnbza.cn/86042.Doc
