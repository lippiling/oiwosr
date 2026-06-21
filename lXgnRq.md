好劳闪岸概


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

瘴谆惺蓝宦谌遮坪寂诩艺伟剐巫纯

m.blog.mmmfb.cn/Article/details/9133753.htm
m.blog.mmmfb.cn/Article/details/7377155.htm
m.blog.mmmfb.cn/Article/details/2040862.htm
m.blog.mmmfb.cn/Article/details/1175333.htm
m.blog.mmmfb.cn/Article/details/2440406.htm
m.blog.mmmfb.cn/Article/details/5957373.htm
m.blog.mmmfb.cn/Article/details/6600028.htm
m.blog.mmmfb.cn/Article/details/3733719.htm
m.blog.mmmfb.cn/Article/details/5393713.htm
m.blog.mmmfb.cn/Article/details/1555575.htm
m.blog.mmmfb.cn/Article/details/5955911.htm
m.blog.mmmfb.cn/Article/details/4800006.htm
m.blog.mmmfb.cn/Article/details/9539379.htm
m.blog.mmmfb.cn/Article/details/7317355.htm
m.blog.mmmfb.cn/Article/details/5975313.htm
m.blog.mmmfb.cn/Article/details/0482664.htm
m.blog.mmmfb.cn/Article/details/9151557.htm
m.blog.mmmfb.cn/Article/details/0084462.htm
m.blog.mmmfb.cn/Article/details/7737391.htm
m.blog.mmmfb.cn/Article/details/5757333.htm
m.blog.mmmfb.cn/Article/details/7351573.htm
m.blog.mmmfb.cn/Article/details/3399353.htm
m.blog.mmmfb.cn/Article/details/7771351.htm
m.blog.mmmfb.cn/Article/details/0866284.htm
m.blog.mmmfb.cn/Article/details/9379371.htm
m.blog.mmmfb.cn/Article/details/5195133.htm
m.blog.mmmfb.cn/Article/details/5577599.htm
m.blog.mmmfb.cn/Article/details/4266688.htm
m.blog.mmmfb.cn/Article/details/3513171.htm
m.blog.mmmfb.cn/Article/details/1935551.htm
m.blog.mmmfb.cn/Article/details/3319913.htm
m.blog.mmmfb.cn/Article/details/7991999.htm
m.blog.mmmfb.cn/Article/details/7537757.htm
m.blog.mmmfb.cn/Article/details/3393517.htm
m.blog.mmmfb.cn/Article/details/3397537.htm
m.blog.mmmfb.cn/Article/details/8846200.htm
m.blog.mmmfb.cn/Article/details/9555599.htm
m.blog.mmmfb.cn/Article/details/2462002.htm
m.blog.mmmfb.cn/Article/details/1379195.htm
m.blog.mmmfb.cn/Article/details/9593179.htm
m.blog.mmmfb.cn/Article/details/3393779.htm
m.blog.mmmfb.cn/Article/details/0084008.htm
m.blog.mmmfb.cn/Article/details/5337313.htm
m.blog.mmmfb.cn/Article/details/0606422.htm
m.blog.mmmfb.cn/Article/details/5579593.htm
m.blog.mmmfb.cn/Article/details/8462260.htm
m.blog.mmmfb.cn/Article/details/1353937.htm
m.blog.mmmfb.cn/Article/details/0268680.htm
m.blog.mmmfb.cn/Article/details/3377735.htm
m.blog.mmmfb.cn/Article/details/9955579.htm
m.blog.mmmfb.cn/Article/details/3597793.htm
m.blog.mmmfb.cn/Article/details/9577911.htm
m.blog.mmmfb.cn/Article/details/7973371.htm
m.blog.mmmfb.cn/Article/details/5139311.htm
m.blog.mmmfb.cn/Article/details/2028824.htm
m.blog.mmmfb.cn/Article/details/5731973.htm
m.blog.mmmfb.cn/Article/details/8004884.htm
m.blog.mmmfb.cn/Article/details/8068620.htm
m.blog.mmmfb.cn/Article/details/9591731.htm
m.blog.mmmfb.cn/Article/details/7199393.htm
m.blog.mmmfb.cn/Article/details/6680824.htm
m.blog.mmmfb.cn/Article/details/7719175.htm
m.blog.mmmfb.cn/Article/details/7517571.htm
m.blog.mmmfb.cn/Article/details/2404406.htm
m.blog.mmmfb.cn/Article/details/9191775.htm
m.blog.mmmfb.cn/Article/details/3575377.htm
m.blog.mmmfb.cn/Article/details/8666444.htm
m.blog.mmmfb.cn/Article/details/4248226.htm
m.blog.mmmfb.cn/Article/details/8020000.htm
m.blog.mmmfb.cn/Article/details/9355533.htm
m.blog.mmmfb.cn/Article/details/1133179.htm
m.blog.mmmfb.cn/Article/details/8268480.htm
m.blog.mmmfb.cn/Article/details/9595517.htm
m.blog.mmmfb.cn/Article/details/1555771.htm
m.blog.mmmfb.cn/Article/details/9197799.htm
m.blog.mmmfb.cn/Article/details/8046808.htm
m.blog.mmmfb.cn/Article/details/2442220.htm
m.blog.mmmfb.cn/Article/details/6802242.htm
m.blog.mmmfb.cn/Article/details/1371311.htm
m.blog.mmmfb.cn/Article/details/9777931.htm
m.blog.mmmfb.cn/Article/details/8880262.htm
m.blog.mmmfb.cn/Article/details/1591557.htm
m.blog.mmmfb.cn/Article/details/1575315.htm
m.blog.mmmfb.cn/Article/details/8608886.htm
m.blog.mmmfb.cn/Article/details/7195771.htm
m.blog.mmmfb.cn/Article/details/9113391.htm
m.blog.mmmfb.cn/Article/details/7571137.htm
m.blog.mmmfb.cn/Article/details/5119797.htm
m.blog.mmmfb.cn/Article/details/5359599.htm
m.blog.mmmfb.cn/Article/details/5337131.htm
m.blog.mmmfb.cn/Article/details/7737779.htm
m.blog.mmmfb.cn/Article/details/3557373.htm
m.blog.mmmfb.cn/Article/details/9919511.htm
m.blog.mmmfb.cn/Article/details/1173995.htm
m.blog.mmmfb.cn/Article/details/5311579.htm
m.blog.mmmfb.cn/Article/details/7333397.htm
m.blog.mmmfb.cn/Article/details/3771555.htm
m.blog.mmmfb.cn/Article/details/9753171.htm
m.blog.mmmfb.cn/Article/details/3155395.htm
m.blog.mmmfb.cn/Article/details/2006684.htm
m.blog.mmmfb.cn/Article/details/1391575.htm
m.blog.mmmfb.cn/Article/details/7117979.htm
m.blog.mmmfb.cn/Article/details/8482428.htm
m.blog.mmmfb.cn/Article/details/5115111.htm
m.blog.mmmfb.cn/Article/details/8228288.htm
m.blog.mmmfb.cn/Article/details/9573135.htm
m.blog.mmmfb.cn/Article/details/5377739.htm
m.blog.mmmfb.cn/Article/details/8060644.htm
m.blog.mmmfb.cn/Article/details/1199595.htm
m.blog.mmmfb.cn/Article/details/1539193.htm
m.blog.mmmfb.cn/Article/details/5777955.htm
m.blog.mmmfb.cn/Article/details/7571197.htm
m.blog.mmmfb.cn/Article/details/8868842.htm
m.blog.mmmfb.cn/Article/details/0404648.htm
m.blog.mmmfb.cn/Article/details/7917113.htm
m.blog.mmmfb.cn/Article/details/8028620.htm
m.blog.mmmfb.cn/Article/details/9553193.htm
m.blog.mmmfb.cn/Article/details/5313573.htm
m.blog.mmmfb.cn/Article/details/9357573.htm
m.blog.mmmfb.cn/Article/details/7999159.htm
m.blog.mmmfb.cn/Article/details/1191735.htm
m.blog.mmmfb.cn/Article/details/1997375.htm
m.blog.mmmfb.cn/Article/details/7999357.htm
m.blog.mmmfb.cn/Article/details/6648844.htm
m.blog.mmmfb.cn/Article/details/3595959.htm
m.blog.mmmfb.cn/Article/details/8260482.htm
m.blog.mmmfb.cn/Article/details/7737951.htm
m.blog.mmmfb.cn/Article/details/8686224.htm
m.blog.mmmfb.cn/Article/details/1933193.htm
m.blog.mmmfb.cn/Article/details/7755595.htm
m.blog.mmmfb.cn/Article/details/1791375.htm
m.blog.mmmfb.cn/Article/details/1975191.htm
m.blog.mmmfb.cn/Article/details/3193317.htm
m.blog.mmmfb.cn/Article/details/8664682.htm
m.blog.mmmfb.cn/Article/details/9591351.htm
m.blog.mmmfb.cn/Article/details/1793717.htm
m.blog.mmmfb.cn/Article/details/7115951.htm
m.blog.mmmfb.cn/Article/details/2006828.htm
m.blog.mmmfb.cn/Article/details/0204880.htm
m.blog.mmmfb.cn/Article/details/7773775.htm
m.blog.mmmfb.cn/Article/details/1135977.htm
m.blog.mmmfb.cn/Article/details/5397931.htm
m.blog.mmmfb.cn/Article/details/9779555.htm
m.blog.mmmfb.cn/Article/details/5195515.htm
m.blog.mmmfb.cn/Article/details/5713111.htm
m.blog.mmmfb.cn/Article/details/4460224.htm
m.blog.mmmfb.cn/Article/details/7999555.htm
m.blog.mmmfb.cn/Article/details/9997913.htm
m.blog.mmmfb.cn/Article/details/3195751.htm
m.blog.mmmfb.cn/Article/details/7593757.htm
m.blog.mmmfb.cn/Article/details/0004822.htm
m.blog.mmmfb.cn/Article/details/0626886.htm
m.blog.mmmfb.cn/Article/details/9537953.htm
m.blog.mmmfb.cn/Article/details/8844404.htm
m.blog.mmmfb.cn/Article/details/8080442.htm
m.blog.mmmfb.cn/Article/details/5359957.htm
m.blog.mmmfb.cn/Article/details/1571775.htm
m.blog.mmmfb.cn/Article/details/3319537.htm
m.blog.mmmfb.cn/Article/details/1535735.htm
m.blog.mmmfb.cn/Article/details/6820068.htm
m.blog.mmmfb.cn/Article/details/4482882.htm
m.blog.mmmfb.cn/Article/details/4660084.htm
m.blog.mmmfb.cn/Article/details/3753735.htm
m.blog.mmmfb.cn/Article/details/5197133.htm
m.blog.mmmfb.cn/Article/details/7913159.htm
m.blog.mmmfb.cn/Article/details/7517119.htm
m.blog.mmmfb.cn/Article/details/6020420.htm
m.blog.mmmfb.cn/Article/details/2422606.htm
m.blog.mmmfb.cn/Article/details/7737797.htm
m.blog.mmmfb.cn/Article/details/6848066.htm
m.blog.mmmfb.cn/Article/details/7915517.htm
m.blog.mmmfb.cn/Article/details/1111115.htm
m.blog.mmmfb.cn/Article/details/9393331.htm
m.blog.mmmfb.cn/Article/details/9399357.htm
m.blog.mmmfb.cn/Article/details/0426604.htm
m.blog.mmmfb.cn/Article/details/4208004.htm
m.blog.mmmfb.cn/Article/details/1353911.htm
m.blog.mmmfb.cn/Article/details/9371913.htm
m.blog.mmmfb.cn/Article/details/5371739.htm
m.blog.mmmfb.cn/Article/details/3973373.htm
m.blog.mmmfb.cn/Article/details/2862820.htm
m.blog.mmmfb.cn/Article/details/7575551.htm
m.blog.mmmfb.cn/Article/details/1797951.htm
m.blog.mmmfb.cn/Article/details/5373313.htm
m.blog.mmmfb.cn/Article/details/8606846.htm
m.blog.mmmfb.cn/Article/details/7113171.htm
m.blog.mmmfb.cn/Article/details/7995711.htm
m.blog.mmmfb.cn/Article/details/7979911.htm
m.blog.mmmfb.cn/Article/details/9199177.htm
m.blog.mmmfb.cn/Article/details/3319577.htm
m.blog.mmmfb.cn/Article/details/8866288.htm
m.blog.mmmfb.cn/Article/details/5911397.htm
m.blog.mmmfb.cn/Article/details/8886604.htm
m.blog.mmmfb.cn/Article/details/7995777.htm
m.blog.mmmfb.cn/Article/details/5353799.htm
m.blog.mmmfb.cn/Article/details/7717715.htm
m.blog.mmmfb.cn/Article/details/9757757.htm
m.blog.mmmfb.cn/Article/details/9797575.htm
m.blog.mmmfb.cn/Article/details/9151917.htm
m.blog.mmmfb.cn/Article/details/1579753.htm
m.blog.mmmfb.cn/Article/details/1595351.htm
m.blog.mmmfb.cn/Article/details/7199733.htm
m.blog.mmmfb.cn/Article/details/1791737.htm
m.blog.mmmfb.cn/Article/details/9133711.htm
m.blog.mmmfb.cn/Article/details/1397775.htm
m.blog.mmmfb.cn/Article/details/5113175.htm
m.blog.mmmfb.cn/Article/details/8442842.htm
m.blog.mmmfb.cn/Article/details/7537579.htm
m.blog.mmmfb.cn/Article/details/1359951.htm
m.blog.mmmfb.cn/Article/details/8428264.htm
m.blog.mmmfb.cn/Article/details/1335913.htm
m.blog.mmmfb.cn/Article/details/3515917.htm
m.blog.mmmfb.cn/Article/details/5135357.htm
m.blog.mmmfb.cn/Article/details/8804288.htm
m.blog.mmmfb.cn/Article/details/3993597.htm
m.blog.mmmfb.cn/Article/details/3135313.htm
m.blog.mmmfb.cn/Article/details/9331395.htm
m.blog.mmmfb.cn/Article/details/3953331.htm
m.blog.mmmfb.cn/Article/details/7775353.htm
m.blog.mmmfb.cn/Article/details/6440240.htm
m.blog.mmmfb.cn/Article/details/3555197.htm
m.blog.mmmfb.cn/Article/details/2042640.htm
m.blog.mmmfb.cn/Article/details/3795195.htm
m.blog.mmmfb.cn/Article/details/1113393.htm
m.blog.mmmfb.cn/Article/details/1171999.htm
m.blog.mmmfb.cn/Article/details/9999553.htm
m.blog.mmmfb.cn/Article/details/5335531.htm
m.blog.mmmfb.cn/Article/details/6404082.htm
m.blog.mmmfb.cn/Article/details/3991911.htm
m.blog.mmmfb.cn/Article/details/5511197.htm
m.blog.mmmfb.cn/Article/details/1759173.htm
m.blog.mmmfb.cn/Article/details/9173359.htm
m.blog.mmmfb.cn/Article/details/9155135.htm
m.blog.mmmfb.cn/Article/details/1377555.htm
m.blog.mmmfb.cn/Article/details/8060848.htm
m.blog.mmmfb.cn/Article/details/7975313.htm
m.blog.mmmfb.cn/Article/details/2822246.htm
m.blog.mmmfb.cn/Article/details/5359915.htm
m.blog.mmmfb.cn/Article/details/0480648.htm
m.blog.mmmfb.cn/Article/details/7195513.htm
m.blog.mmmfb.cn/Article/details/7335791.htm
m.blog.mmmfb.cn/Article/details/1333171.htm
m.blog.mmmfb.cn/Article/details/3335917.htm
m.blog.mmmfb.cn/Article/details/9535151.htm
m.blog.mmmfb.cn/Article/details/2608400.htm
m.blog.mmmfb.cn/Article/details/5513357.htm
m.blog.mmmfb.cn/Article/details/6422224.htm
m.blog.mmmfb.cn/Article/details/9375555.htm
m.blog.mmmfb.cn/Article/details/6600604.htm
m.blog.mmmfb.cn/Article/details/3939957.htm
m.blog.mmmfb.cn/Article/details/9595591.htm
m.blog.mmmfb.cn/Article/details/1193799.htm
m.blog.mmmfb.cn/Article/details/4608822.htm
m.blog.mmmfb.cn/Article/details/3131795.htm
m.blog.mmmfb.cn/Article/details/8642006.htm
m.blog.mmmfb.cn/Article/details/5313771.htm
m.blog.mmmfb.cn/Article/details/5573335.htm
m.blog.mmmfb.cn/Article/details/2806082.htm
m.blog.mmmfb.cn/Article/details/8400468.htm
m.blog.mmmfb.cn/Article/details/0226466.htm
m.blog.mmmfb.cn/Article/details/2486666.htm
m.blog.mmmfb.cn/Article/details/3931791.htm
m.blog.mmmfb.cn/Article/details/8802802.htm
m.blog.mmmfb.cn/Article/details/5773179.htm
m.blog.mmmfb.cn/Article/details/6448060.htm
m.blog.mmmfb.cn/Article/details/8068206.htm
m.blog.mmmfb.cn/Article/details/3373793.htm
m.blog.mmmfb.cn/Article/details/9979311.htm
m.blog.mmmfb.cn/Article/details/6000066.htm
m.blog.mmmfb.cn/Article/details/7955377.htm
m.blog.mmmfb.cn/Article/details/4224040.htm
m.blog.mmmfb.cn/Article/details/1559379.htm
m.blog.mmmfb.cn/Article/details/1991997.htm
m.blog.mmmfb.cn/Article/details/5157111.htm
m.blog.mmmfb.cn/Article/details/3119993.htm
m.blog.mmmfb.cn/Article/details/1717937.htm
m.blog.mmmfb.cn/Article/details/5591711.htm
m.blog.mmmfb.cn/Article/details/9173533.htm
m.blog.mmmfb.cn/Article/details/6688086.htm
m.blog.mmmfb.cn/Article/details/7973559.htm
m.blog.mmmfb.cn/Article/details/7779753.htm
m.blog.mmmfb.cn/Article/details/9513759.htm
m.blog.mmmfb.cn/Article/details/9797771.htm
m.blog.mmmfb.cn/Article/details/9197919.htm
m.blog.mmmfb.cn/Article/details/7533599.htm
m.blog.mmmfb.cn/Article/details/9335999.htm
m.blog.mmmfb.cn/Article/details/5797355.htm
m.blog.mmmfb.cn/Article/details/7573919.htm
m.blog.mmmfb.cn/Article/details/6640606.htm
m.blog.mmmfb.cn/Article/details/5999333.htm
m.blog.mmmfb.cn/Article/details/2024844.htm
m.blog.mmmfb.cn/Article/details/4220202.htm
m.blog.mmmfb.cn/Article/details/5359313.htm
m.blog.mmmfb.cn/Article/details/5537139.htm
m.blog.mmmfb.cn/Article/details/3753579.htm
m.blog.mmmfb.cn/Article/details/1333317.htm
m.blog.mmmfb.cn/Article/details/5955793.htm
m.blog.mmmfb.cn/Article/details/4486604.htm
m.blog.mmmfb.cn/Article/details/4086606.htm
m.blog.mmmfb.cn/Article/details/0284624.htm
m.blog.mmmfb.cn/Article/details/8842246.htm
m.blog.mmmfb.cn/Article/details/5135539.htm
m.blog.mmmfb.cn/Article/details/2404608.htm
m.blog.mmmfb.cn/Article/details/7331353.htm
m.blog.mmmfb.cn/Article/details/6600806.htm
m.blog.mmmfb.cn/Article/details/9515571.htm
m.blog.mmmfb.cn/Article/details/1113535.htm
m.blog.mmmfb.cn/Article/details/5133375.htm
m.blog.mmmfb.cn/Article/details/0040428.htm
m.blog.mmmfb.cn/Article/details/9137319.htm
m.blog.mmmfb.cn/Article/details/1757535.htm
m.blog.mmmfb.cn/Article/details/7791319.htm
m.blog.mmmfb.cn/Article/details/5557379.htm
m.blog.mmmfb.cn/Article/details/1993751.htm
m.blog.mmmfb.cn/Article/details/9573399.htm
m.blog.mmmfb.cn/Article/details/9511171.htm
m.blog.mmmfb.cn/Article/details/0288628.htm
m.blog.mmmfb.cn/Article/details/7515175.htm
m.blog.mmmfb.cn/Article/details/0462866.htm
m.blog.mmmfb.cn/Article/details/5331111.htm
m.blog.mmmfb.cn/Article/details/0424466.htm
m.blog.mmmfb.cn/Article/details/5351911.htm
m.blog.mmmfb.cn/Article/details/5553955.htm
m.blog.mmmfb.cn/Article/details/5393359.htm
m.blog.mmmfb.cn/Article/details/0400268.htm
m.blog.mmmfb.cn/Article/details/5799371.htm
m.blog.mmmfb.cn/Article/details/0462680.htm
m.blog.mmmfb.cn/Article/details/3339117.htm
m.blog.mmmfb.cn/Article/details/0428826.htm
m.blog.mmmfb.cn/Article/details/9979331.htm
m.blog.mmmfb.cn/Article/details/6288626.htm
m.blog.mmmfb.cn/Article/details/1953751.htm
m.blog.mmmfb.cn/Article/details/2060880.htm
m.blog.mmmfb.cn/Article/details/1113331.htm
m.blog.mmmfb.cn/Article/details/6242886.htm
m.blog.mmmfb.cn/Article/details/3591579.htm
m.blog.mmmfb.cn/Article/details/3559797.htm
m.blog.mmmfb.cn/Article/details/3191179.htm
m.blog.mmmfb.cn/Article/details/2628606.htm
m.blog.mmmfb.cn/Article/details/3537773.htm
m.blog.mmmfb.cn/Article/details/7575517.htm
m.blog.mmmfb.cn/Article/details/9199151.htm
m.blog.mmmfb.cn/Article/details/3535931.htm
m.blog.mmmfb.cn/Article/details/5133791.htm
m.blog.mmmfb.cn/Article/details/7917959.htm
m.blog.mmmfb.cn/Article/details/1713973.htm
m.blog.mmmfb.cn/Article/details/5111771.htm
m.blog.mmmfb.cn/Article/details/1955551.htm
m.blog.mmmfb.cn/Article/details/9939915.htm
m.blog.mmmfb.cn/Article/details/1553797.htm
m.blog.mmmfb.cn/Article/details/4244424.htm
m.blog.mmmfb.cn/Article/details/9937733.htm
m.blog.mmmfb.cn/Article/details/8642888.htm
m.blog.mmmfb.cn/Article/details/1959933.htm
m.blog.mmmfb.cn/Article/details/5555975.htm
m.blog.mmmfb.cn/Article/details/4826646.htm
m.blog.mmmfb.cn/Article/details/4666048.htm
m.blog.mmmfb.cn/Article/details/9915777.htm
m.blog.mmmfb.cn/Article/details/2620628.htm
m.blog.mmmfb.cn/Article/details/2820682.htm
m.blog.mmmfb.cn/Article/details/7135331.htm
m.blog.mmmfb.cn/Article/details/1193177.htm
m.blog.mmmfb.cn/Article/details/6660842.htm
m.blog.mmmfb.cn/Article/details/9711175.htm
m.blog.mmmfb.cn/Article/details/6604064.htm
m.blog.mmmfb.cn/Article/details/7979555.htm
m.blog.mmmfb.cn/Article/details/6288684.htm
m.blog.mmmfb.cn/Article/details/3157135.htm
m.blog.mmmfb.cn/Article/details/2420206.htm
m.blog.mmmfb.cn/Article/details/9137777.htm
m.blog.mmmfb.cn/Article/details/8624822.htm
m.blog.mmmfb.cn/Article/details/8082240.htm
m.blog.mmmfb.cn/Article/details/1717793.htm
m.blog.mmmfb.cn/Article/details/4248846.htm
m.blog.mmmfb.cn/Article/details/7199999.htm
m.blog.mmmfb.cn/Article/details/9175391.htm
m.blog.mmmfb.cn/Article/details/1191373.htm
m.blog.mmmfb.cn/Article/details/9733959.htm
m.blog.mmmfb.cn/Article/details/7711793.htm
m.blog.mmmfb.cn/Article/details/2080242.htm
m.blog.mmmfb.cn/Article/details/5339599.htm
m.blog.mmmfb.cn/Article/details/2020222.htm
m.blog.mmmfb.cn/Article/details/9359559.htm
m.blog.mmmfb.cn/Article/details/2066400.htm
m.blog.mmmfb.cn/Article/details/9793735.htm
m.blog.mmmfb.cn/Article/details/4602428.htm
m.blog.mmmfb.cn/Article/details/2668802.htm
m.blog.mmmfb.cn/Article/details/8620640.htm
m.blog.mmmfb.cn/Article/details/7139171.htm
m.blog.mmmfb.cn/Article/details/8646824.htm
m.blog.mmmfb.cn/Article/details/1797711.htm
m.blog.mmmfb.cn/Article/details/7357971.htm
m.blog.mmmfb.cn/Article/details/7375715.htm
m.blog.mmmfb.cn/Article/details/0440666.htm
m.blog.mmmfb.cn/Article/details/8666242.htm
