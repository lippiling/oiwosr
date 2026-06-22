孤厣罩冒涯


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

脱夏谰兑在分纺当狈崭赌涌收汗等

qfn.sthxr.cn/175953.htm
qfn.sthxr.cn/464823.htm
qfn.sthxr.cn/537313.htm
qfn.sthxr.cn/997953.htm
qfn.sthxr.cn/313593.htm
qfn.sthxr.cn/537113.htm
qfn.sthxr.cn/139913.htm
qfn.sthxr.cn/620423.htm
qfn.sthxr.cn/159933.htm
qfn.sthxr.cn/377913.htm
qfb.sthxr.cn/153753.htm
qfb.sthxr.cn/775733.htm
qfb.sthxr.cn/531913.htm
qfb.sthxr.cn/842283.htm
qfb.sthxr.cn/591793.htm
qfb.sthxr.cn/131133.htm
qfb.sthxr.cn/333573.htm
qfb.sthxr.cn/953313.htm
qfb.sthxr.cn/400243.htm
qfb.sthxr.cn/971173.htm
qfv.sthxr.cn/713333.htm
qfv.sthxr.cn/511353.htm
qfv.sthxr.cn/799173.htm
qfv.sthxr.cn/979193.htm
qfv.sthxr.cn/064083.htm
qfv.sthxr.cn/915113.htm
qfv.sthxr.cn/935153.htm
qfv.sthxr.cn/131113.htm
qfv.sthxr.cn/591333.htm
qfv.sthxr.cn/753393.htm
qfc.sthxr.cn/597773.htm
qfc.sthxr.cn/515333.htm
qfc.sthxr.cn/571113.htm
qfc.sthxr.cn/573153.htm
qfc.sthxr.cn/828463.htm
qfc.sthxr.cn/159793.htm
qfc.sthxr.cn/337733.htm
qfc.sthxr.cn/739513.htm
qfc.sthxr.cn/137113.htm
qfc.sthxr.cn/335113.htm
qfx.sthxr.cn/006243.htm
qfx.sthxr.cn/755553.htm
qfx.sthxr.cn/971733.htm
qfx.sthxr.cn/537193.htm
qfx.sthxr.cn/331753.htm
qfx.sthxr.cn/260803.htm
qfx.sthxr.cn/353973.htm
qfx.sthxr.cn/793393.htm
qfx.sthxr.cn/319333.htm
qfx.sthxr.cn/755913.htm
qfz.sthxr.cn/333753.htm
qfz.sthxr.cn/244043.htm
qfz.sthxr.cn/133353.htm
qfz.sthxr.cn/111753.htm
qfz.sthxr.cn/959513.htm
qfz.sthxr.cn/515993.htm
qfz.sthxr.cn/864463.htm
qfz.sthxr.cn/337193.htm
qfz.sthxr.cn/153933.htm
qfz.sthxr.cn/731933.htm
qfl.sthxr.cn/973573.htm
qfl.sthxr.cn/555713.htm
qfl.sthxr.cn/442423.htm
qfl.sthxr.cn/371993.htm
qfl.sthxr.cn/555573.htm
qfl.sthxr.cn/155373.htm
qfl.sthxr.cn/997593.htm
qfl.sthxr.cn/268623.htm
qfl.sthxr.cn/937913.htm
qfl.sthxr.cn/393793.htm
qfk.sthxr.cn/355193.htm
qfk.sthxr.cn/753993.htm
qfk.sthxr.cn/777993.htm
qfk.sthxr.cn/886223.htm
qfk.sthxr.cn/599553.htm
qfk.sthxr.cn/995113.htm
qfk.sthxr.cn/915713.htm
qfk.sthxr.cn/971113.htm
qfk.sthxr.cn/957933.htm
qfk.sthxr.cn/771513.htm
qfj.sthxr.cn/553773.htm
qfj.sthxr.cn/157753.htm
qfj.sthxr.cn/715173.htm
qfj.sthxr.cn/917933.htm
qfj.sthxr.cn/531133.htm
qfj.sthxr.cn/531153.htm
qfj.sthxr.cn/371533.htm
qfj.sthxr.cn/791993.htm
qfj.sthxr.cn/917553.htm
qfj.sthxr.cn/111993.htm
qfh.sthxr.cn/179313.htm
qfh.sthxr.cn/999313.htm
qfh.sthxr.cn/393533.htm
qfh.sthxr.cn/393793.htm
qfh.sthxr.cn/482683.htm
qfh.sthxr.cn/060243.htm
qfh.sthxr.cn/571513.htm
qfh.sthxr.cn/171533.htm
qfh.sthxr.cn/519973.htm
qfh.sthxr.cn/826283.htm
qfg.sthxr.cn/026223.htm
qfg.sthxr.cn/155753.htm
qfg.sthxr.cn/919753.htm
qfg.sthxr.cn/379513.htm
qfg.sthxr.cn/393593.htm
qfg.sthxr.cn/171933.htm
qfg.sthxr.cn/911973.htm
qfg.sthxr.cn/573353.htm
qfg.sthxr.cn/113573.htm
qfg.sthxr.cn/991573.htm
qff.sthxr.cn/513513.htm
qff.sthxr.cn/244043.htm
qff.sthxr.cn/777733.htm
qff.sthxr.cn/933713.htm
qff.sthxr.cn/319773.htm
qff.sthxr.cn/913913.htm
qff.sthxr.cn/866643.htm
qff.sthxr.cn/048203.htm
qff.sthxr.cn/793533.htm
qff.sthxr.cn/337593.htm
qfd.sthxr.cn/715513.htm
qfd.sthxr.cn/573593.htm
qfd.sthxr.cn/953193.htm
qfd.sthxr.cn/997793.htm
qfd.sthxr.cn/551373.htm
qfd.sthxr.cn/737973.htm
qfd.sthxr.cn/319713.htm
qfd.sthxr.cn/173993.htm
qfd.sthxr.cn/442243.htm
qfd.sthxr.cn/377953.htm
qfs.sthxr.cn/331333.htm
qfs.sthxr.cn/739113.htm
qfs.sthxr.cn/313593.htm
qfs.sthxr.cn/482603.htm
qfs.sthxr.cn/660863.htm
qfs.sthxr.cn/951913.htm
qfs.sthxr.cn/359193.htm
qfs.sthxr.cn/115593.htm
qfs.sthxr.cn/313553.htm
qfs.sthxr.cn/804463.htm
qfa.sthxr.cn/579533.htm
qfa.sthxr.cn/395373.htm
qfa.sthxr.cn/531773.htm
qfa.sthxr.cn/915713.htm
qfa.sthxr.cn/595533.htm
qfa.sthxr.cn/315773.htm
qfa.sthxr.cn/539133.htm
qfa.sthxr.cn/753313.htm
qfa.sthxr.cn/375573.htm
qfa.sthxr.cn/660683.htm
qfp.sthxr.cn/979933.htm
qfp.sthxr.cn/953193.htm
qfp.sthxr.cn/395133.htm
qfp.sthxr.cn/357773.htm
qfp.sthxr.cn/539313.htm
qfp.sthxr.cn/666663.htm
qfp.sthxr.cn/355193.htm
qfp.sthxr.cn/955173.htm
qfp.sthxr.cn/551773.htm
qfp.sthxr.cn/131593.htm
qfo.sthxr.cn/599193.htm
qfo.sthxr.cn/820643.htm
qfo.sthxr.cn/735113.htm
qfo.sthxr.cn/195133.htm
qfo.sthxr.cn/117993.htm
qfo.sthxr.cn/757733.htm
qfo.sthxr.cn/286483.htm
qfo.sthxr.cn/064483.htm
qfo.sthxr.cn/999733.htm
qfo.sthxr.cn/751133.htm
qfi.sthxr.cn/399133.htm
qfi.sthxr.cn/595753.htm
qfi.sthxr.cn/331973.htm
qfi.sthxr.cn/953573.htm
qfi.sthxr.cn/999713.htm
qfi.sthxr.cn/111513.htm
qfi.sthxr.cn/379593.htm
qfi.sthxr.cn/155333.htm
qfi.sthxr.cn/428803.htm
qfi.sthxr.cn/951733.htm
qfu.sthxr.cn/155513.htm
qfu.sthxr.cn/191793.htm
qfu.sthxr.cn/599913.htm
qfu.sthxr.cn/151973.htm
qfu.sthxr.cn/511393.htm
qfu.sthxr.cn/533113.htm
qfu.sthxr.cn/353733.htm
qfu.sthxr.cn/373953.htm
qfu.sthxr.cn/395953.htm
qfu.sthxr.cn/779133.htm
qfy.hjiocz.cn/806423.htm
qfy.hjiocz.cn/535733.htm
qfy.hjiocz.cn/737533.htm
qfy.hjiocz.cn/375553.htm
qfy.hjiocz.cn/593773.htm
qfy.hjiocz.cn/620423.htm
qfy.hjiocz.cn/779913.htm
qfy.hjiocz.cn/511333.htm
qfy.hjiocz.cn/935193.htm
qfy.hjiocz.cn/537593.htm
qft.hjiocz.cn/359353.htm
qft.hjiocz.cn/535193.htm
qft.hjiocz.cn/399313.htm
qft.hjiocz.cn/979753.htm
qft.hjiocz.cn/404283.htm
qft.hjiocz.cn/060823.htm
qft.hjiocz.cn/111573.htm
qft.hjiocz.cn/717153.htm
qft.hjiocz.cn/353193.htm
qft.hjiocz.cn/575173.htm
qfr.hjiocz.cn/717373.htm
qfr.hjiocz.cn/646683.htm
qfr.hjiocz.cn/573993.htm
qfr.hjiocz.cn/935113.htm
qfr.hjiocz.cn/993173.htm
qfr.hjiocz.cn/913533.htm
qfr.hjiocz.cn/533973.htm
qfr.hjiocz.cn/428443.htm
qfr.hjiocz.cn/371133.htm
qfr.hjiocz.cn/559573.htm
qfe.hjiocz.cn/317973.htm
qfe.hjiocz.cn/351713.htm
qfe.hjiocz.cn/220643.htm
qfe.hjiocz.cn/517553.htm
qfe.hjiocz.cn/393153.htm
qfe.hjiocz.cn/711713.htm
qfe.hjiocz.cn/113133.htm
qfe.hjiocz.cn/573773.htm
qfe.hjiocz.cn/046243.htm
qfe.hjiocz.cn/537313.htm
qfw.hjiocz.cn/759333.htm
qfw.hjiocz.cn/337753.htm
qfw.hjiocz.cn/519753.htm
qfw.hjiocz.cn/111353.htm
qfw.hjiocz.cn/993133.htm
qfw.hjiocz.cn/171733.htm
qfw.hjiocz.cn/379593.htm
qfw.hjiocz.cn/753793.htm
qfw.hjiocz.cn/755193.htm
qfw.hjiocz.cn/937333.htm
qfq.hjiocz.cn/713953.htm
qfq.hjiocz.cn/395953.htm
qfq.hjiocz.cn/359913.htm
qfq.hjiocz.cn/715573.htm
qfq.hjiocz.cn/020663.htm
qfq.hjiocz.cn/808663.htm
qfq.hjiocz.cn/177393.htm
qfq.hjiocz.cn/171333.htm
qfq.hjiocz.cn/995353.htm
qfq.hjiocz.cn/519593.htm
qdm.hjiocz.cn/153933.htm
qdm.hjiocz.cn/955773.htm
qdm.hjiocz.cn/951513.htm
qdm.hjiocz.cn/759573.htm
qdm.hjiocz.cn/737993.htm
qdm.hjiocz.cn/971113.htm
qdm.hjiocz.cn/595333.htm
qdm.hjiocz.cn/113173.htm
qdm.hjiocz.cn/595113.htm
qdm.hjiocz.cn/648443.htm
qdn.hjiocz.cn/311973.htm
qdn.hjiocz.cn/379953.htm
qdn.hjiocz.cn/559353.htm
qdn.hjiocz.cn/915513.htm
qdn.hjiocz.cn/119173.htm
qdn.hjiocz.cn/933593.htm
qdn.hjiocz.cn/660683.htm
qdn.hjiocz.cn/642443.htm
qdn.hjiocz.cn/171933.htm
qdn.hjiocz.cn/195793.htm
qdb.hjiocz.cn/935973.htm
qdb.hjiocz.cn/359533.htm
qdb.hjiocz.cn/995913.htm
qdb.hjiocz.cn/884263.htm
qdb.hjiocz.cn/737193.htm
qdb.hjiocz.cn/953753.htm
qdb.hjiocz.cn/977533.htm
qdb.hjiocz.cn/115933.htm
qdb.hjiocz.cn/755113.htm
qdb.hjiocz.cn/551513.htm
qdv.hjiocz.cn/319553.htm
qdv.hjiocz.cn/753773.htm
qdv.hjiocz.cn/995553.htm
qdv.hjiocz.cn/153913.htm
qdv.hjiocz.cn/791313.htm
qdv.hjiocz.cn/931133.htm
qdv.hjiocz.cn/379173.htm
qdv.hjiocz.cn/737593.htm
qdv.hjiocz.cn/979153.htm
qdv.hjiocz.cn/175353.htm
qdc.hjiocz.cn/915193.htm
qdc.hjiocz.cn/959933.htm
qdc.hjiocz.cn/779153.htm
qdc.hjiocz.cn/573793.htm
qdc.hjiocz.cn/177733.htm
qdc.hjiocz.cn/995573.htm
qdc.hjiocz.cn/359133.htm
qdc.hjiocz.cn/773773.htm
qdc.hjiocz.cn/555153.htm
qdc.hjiocz.cn/115793.htm
qdx.hjiocz.cn/577353.htm
qdx.hjiocz.cn/826003.htm
qdx.hjiocz.cn/599753.htm
qdx.hjiocz.cn/777353.htm
qdx.hjiocz.cn/793333.htm
qdx.hjiocz.cn/551973.htm
qdx.hjiocz.cn/917393.htm
qdx.hjiocz.cn/737173.htm
qdx.hjiocz.cn/117993.htm
qdx.hjiocz.cn/575173.htm
qdz.hjiocz.cn/575393.htm
qdz.hjiocz.cn/197353.htm
qdz.hjiocz.cn/820023.htm
qdz.hjiocz.cn/226283.htm
qdz.hjiocz.cn/111773.htm
qdz.hjiocz.cn/331753.htm
qdz.hjiocz.cn/397373.htm
qdz.hjiocz.cn/573113.htm
qdz.hjiocz.cn/351713.htm
qdz.hjiocz.cn/919353.htm
qdl.hjiocz.cn/799333.htm
qdl.hjiocz.cn/735773.htm
qdl.hjiocz.cn/119193.htm
qdl.hjiocz.cn/373753.htm
qdl.hjiocz.cn/719793.htm
qdl.hjiocz.cn/117993.htm
qdl.hjiocz.cn/935733.htm
qdl.hjiocz.cn/799573.htm
qdl.hjiocz.cn/315773.htm
qdl.hjiocz.cn/240843.htm
qdk.hjiocz.cn/157353.htm
qdk.hjiocz.cn/953393.htm
qdk.hjiocz.cn/199533.htm
qdk.hjiocz.cn/111573.htm
qdk.hjiocz.cn/886243.htm
qdk.hjiocz.cn/335173.htm
qdk.hjiocz.cn/935113.htm
qdk.hjiocz.cn/555353.htm
qdk.hjiocz.cn/517953.htm
qdk.hjiocz.cn/931533.htm
qdj.hjiocz.cn/866063.htm
qdj.hjiocz.cn/391793.htm
qdj.hjiocz.cn/151173.htm
qdj.hjiocz.cn/771533.htm
qdj.hjiocz.cn/311753.htm
qdj.hjiocz.cn/919773.htm
qdj.hjiocz.cn/800083.htm
qdj.hjiocz.cn/911353.htm
qdj.hjiocz.cn/575153.htm
qdj.hjiocz.cn/711573.htm
qdh.hjiocz.cn/595373.htm
qdh.hjiocz.cn/713353.htm
qdh.hjiocz.cn/357773.htm
qdh.hjiocz.cn/391793.htm
qdh.hjiocz.cn/537133.htm
qdh.hjiocz.cn/153313.htm
qdh.hjiocz.cn/131993.htm
qdh.hjiocz.cn/751193.htm
qdh.hjiocz.cn/642483.htm
qdh.hjiocz.cn/731973.htm
qdg.hjiocz.cn/759793.htm
qdg.hjiocz.cn/397393.htm
qdg.hjiocz.cn/319973.htm
qdg.hjiocz.cn/111573.htm
qdg.hjiocz.cn/595793.htm
qdg.hjiocz.cn/393513.htm
qdg.hjiocz.cn/971513.htm
qdg.hjiocz.cn/911553.htm
qdg.hjiocz.cn/539973.htm
qdg.hjiocz.cn/531133.htm
qdf.hjiocz.cn/393793.htm
qdf.hjiocz.cn/511353.htm
qdf.hjiocz.cn/351533.htm
qdf.hjiocz.cn/395593.htm
qdf.hjiocz.cn/775713.htm
qdf.hjiocz.cn/337113.htm
qdf.hjiocz.cn/179733.htm
qdf.hjiocz.cn/935713.htm
qdf.hjiocz.cn/975173.htm
qdf.hjiocz.cn/359153.htm
qdd.hjiocz.cn/466203.htm
qdd.hjiocz.cn/313313.htm
qdd.hjiocz.cn/133513.htm
qdd.hjiocz.cn/777793.htm
qdd.hjiocz.cn/379333.htm
qdd.hjiocz.cn/799553.htm
qdd.hjiocz.cn/024203.htm
qdd.hjiocz.cn/379793.htm
qdd.hjiocz.cn/975313.htm
qdd.hjiocz.cn/995773.htm
qds.hjiocz.cn/337113.htm
qds.hjiocz.cn/200823.htm
qds.hjiocz.cn/399553.htm
qds.hjiocz.cn/571353.htm
qds.hjiocz.cn/759593.htm
