晒啃撼啃辟


Python 数字签名与证书实战
============================

数字签名确保证据来源的真实性和数据的完整性。本章深入 RSA/ECDSA 签名、
X.509 证书链验证以及自签名证书生成。

1. 基础环境
------------

from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa, ec, padding, utils
from cryptography.hazmat.backends import default_backend
from cryptography import x509
from cryptography.x509.oid import NameOID
from cryptography.exceptions import InvalidSignature
import datetime
import os

2. RSA PKCS#1 v1.5 签名
-----------------------

# 生成 RSA 密钥对
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048,
    backend=default_backend()
)
public_key = private_key.public_key()

def rsa_sign_pkcs1v15(data: bytes, priv_key) -> bytes:
    """使用 RSA PKCS#1 v1.5 方案对数据进行签名"""
    signature = priv_key.sign(
        data,
        padding.PKCS1v15(),
        hashes.SHA256()
    )
    return signature

def rsa_verify_pkcs1v15(data: bytes, signature: bytes, pub_key) -> bool:
    """验证 RSA PKCS#1 v1.5 签名"""
    try:
        pub_key.verify(
            signature,
            data,
            padding.PKCS1v15(),
            hashes.SHA256()
        )
        return True
    except InvalidSignature:
        return False

# 测试签名与验证
message = b"这是一条需要签名的关键消息，比如合同或交易记录"
sig_v15 = rsa_sign_pkcs1v15(message, private_key)
print(f"PKCS1v15 签名长度: {len(sig_v15)} 字节")
print(f"验证结果: {rsa_verify_pkcs1v15(message, sig_v15, public_key)}")
# 篡改数据后验证应失败
print(f"篡改后验证: {rsa_verify_pkcs1v15(b"篡改的消息", sig_v15, public_key)}")

3. RSA PSS 签名（更安全的填充）
-------------------------------

def rsa_sign_pss(data: bytes, priv_key) -> bytes:
    """使用 RSA-PSS 方案签名，PSS 比 PKCS1v15 有更强的安全保证"""
    signature = priv_key.sign(
        data,
        padding.PSS(
            mgf=padding.MGF1(hashes.SHA256()),
            salt_length=padding.PSS.MAX_LENGTH  # 最大盐长度，增强安全性
        ),
        hashes.SHA256()
    )
    return signature

def rsa_verify_pss(data: bytes, signature: bytes, pub_key) -> bool:
    """验证 RSA-PSS 签名"""
    try:
        pub_key.verify(
            signature,
            data,
            padding.PSS(
                mgf=padding.MGF1(hashes.SHA256()),
                salt_length=padding.PSS.MAX_LENGTH
            ),
            hashes.SHA256()
        )
        return True
    except InvalidSignature:
        return False

sig_pss = rsa_sign_pss(message, private_key)
print(f"\nPSS 签名验证: {rsa_verify_pss(message, sig_pss, public_key)}")

# PSS 的优势：随机化签名使得同一消息多次签名结果不同
# PKCS1v15 的优势：广泛兼容，所有平台都支持

4. ECDSA 签名（椭圆曲线）
-------------------------

def generate_ec_keypair():
    """生成 ECDSA 密钥对（使用 P-256 曲线）"""
    private_key = ec.generate_private_key(
        ec.SECP256R1(),  # P-256 曲线，等效于 prime256v1
        default_backend()
    )
    return private_key, private_key.public_key()

ec_priv, ec_pub = generate_ec_keypair()

def ecdsa_sign(data: bytes, priv_key) -> bytes:
    """使用 ECDSA 签名"""
    signature = priv_key.sign(
        data,
        ec.ECDSA(hashes.SHA256())
    )
    return signature

def ecdsa_verify(data: bytes, signature: bytes, pub_key) -> bool:
    """验证 ECDSA 签名"""
    try:
        pub_key.verify(
            signature,
            data,
            ec.ECDSA(hashes.SHA256())
        )
        return True
    except InvalidSignature:
        return False

ec_sig = ecdsa_sign(message, ec_priv)
print(f"\nECDSA 签名长度: {len(ec_sig)} 字节（比 RSA 短很多）")
print(f"ECDSA 验证: {ecdsa_verify(message, ec_sig, ec_pub)}")

# ECDSA 优势：同等安全强度下密钥更短、签名更小、性能更好
# 256 位 ECC ≈ 3072 位 RSA

5. 签名序列化与传输
-------------------

def serialize_signature(signature: bytes, format: str = "DER") -> bytes:
    """将签名格式化为可传输的 DER 编码"""
    # ECDSA 的签名在 cryptography 中默认是 DER 编码
    # RSA 签名本身就是固定长度字节串，无需特殊编码
    return signature

def export_public_key(pub_key) -> bytes:
    """将公钥导出为 PEM 格式供分发"""
    return pub_key.public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    )

def load_public_key(pem_data: bytes):
    """从 PEM 数据加载公钥"""
    return serialization.load_pem_public_key(pem_data, default_backend())

# 导出与加载测试
pub_pem = export_public_key(public_key)
reloaded_pub = load_public_key(pub_pem)
print(f"\n公钥 PEM（前 50 字符）: {pub_pem[:50]}")
print(f"重新加载后验证: {rsa_verify_pss(message, sig_pss, reloaded_pub)}")

6. X.509 证书链验证
-------------------

def create_self_signed_ca(ca_private_key, subject_name: str) -> x509.Certificate:
    """创建自签名 CA 证书"""
    subject = issuer = x509.Name([
        x509.NameAttribute(NameOID.COUNTRY_NAME, "CN"),
        x509.NameAttribute(NameOID.ORGANIZATION_NAME, "Demo CA"),
        x509.NameAttribute(NameOID.COMMON_NAME, subject_name),
    ])
    cert = (x509.CertificateBuilder()
        .subject_name(subject)
        .issuer_name(issuer)
        .public_key(ca_private_key.public_key())
        .serial_number(x509.random_serial_number())
        .not_valid_before(datetime.datetime.utcnow())
        .not_valid_after(datetime.datetime.utcnow() + datetime.timedelta(days=3650))
        .add_extension(x509.BasicConstraints(ca=True, path_length=None), critical=True)
        .sign(ca_private_key, hashes.SHA256(), default_backend()))
    return cert

def issue_certificate(ca_cert, ca_private_key, subject_pub_key,
                      subject_name: str) -> x509.Certificate:
    """使用 CA 证书签发子证书"""
    subject = x509.Name([
        x509.NameAttribute(NameOID.COUNTRY_NAME, "CN"),
        x509.NameAttribute(NameOID.ORGANIZATION_NAME, "Demo Sub"),
        x509.NameAttribute(NameOID.COMMON_NAME, subject_name),
    ])
    cert = (x509.CertificateBuilder()
        .subject_name(subject)
        .issuer_name(ca_cert.subject)
        .public_key(subject_pub_key)
        .serial_number(x509.random_serial_number())
        .not_valid_before(datetime.datetime.utcnow())
        .not_valid_after(datetime.datetime.utcnow() + datetime.timedelta(days=365))
        .add_extension(x509.BasicConstraints(ca=False, path_length=None), critical=True)
        .sign(ca_private_key, hashes.SHA256(), default_backend()))
    return cert

# 创建 CA 和子证书
ca_priv, ca_pub = generate_ec_keypair()
ca_cert = create_self_signed_ca(ca_priv, "My Root CA")
sub_priv, sub_pub = generate_ec_keypair()
sub_cert = issue_certificate(ca_cert, ca_priv, sub_pub, "server.example.com")

# 验证证书链（验证 sub_cert 是由 ca_cert 签发的）
def verify_cert_chain(cert: x509.Certificate, issuer_cert: x509.Certificate) -> bool:
    """验证证书链：cert 是否由 issuer_cert 签发"""
    try:
        issuer_pub = issuer_cert.public_key()
        issuer_pub.verify(
            cert.signature,
            cert.tbs_certificate_bytes,
            padding.PKCS1v15(),  # 取决于签名算法
            cert.signature_hash_algorithm
        )
        return True
    except InvalidSignature:
        return False

print(f"\n证书链验证: {verify_cert_chain(sub_cert, ca_cert)}")

7. 自签名证书生成
------------------

def generate_self_signed_cert(common_name: str, days_valid: int = 365):
    """生成自签名证书（适用于内部测试）"""
    key = rsa.generate_private_key(65537, 2048, default_backend())
    subject = issuer = x509.Name([
        x509.NameAttribute(NameOID.COUNTRY_NAME, "CN"),
        x509.NameAttribute(NameOID.COMMON_NAME, common_name),
    ])
    cert = (x509.CertificateBuilder()
        .subject_name(subject)
        .issuer_name(issuer)
        .public_key(key.public_key())
        .serial_number(x509.random_serial_number())
        .not_valid_before(datetime.datetime.utcnow())
        .not_valid_after(datetime.datetime.utcnow() + datetime.timedelta(days=days_valid))
        .add_extension(x509.SubjectAlternativeName([
            x509.DNSName(common_name),
            x509.DNSName(f"*.{common_name}"),
        ]), critical=False)
        .sign(key, hashes.SHA256(), default_backend()))
    return cert, key

self_cert, self_key = generate_self_signed_cert("example.com")
print(f"自签名证书主题: {self_cert.subject}")
print(f"SAN 扩展: {self_cert.extensions}")

8. 证书序列化（PEM / DER）
---------------------------

def serialize_certificate(cert: x509.Certificate, fmt: str = "PEM") -> bytes:
    """将证书序列化为 PEM 或 DER 格式"""
    encoding = serialization.Encoding.PEM if fmt == "PEM" else serialization.Encoding.DER
    return cert.public_bytes(encoding)

def save_cert_and_key(cert: x509.Certificate, key, path_prefix: str):
    """保存证书和私钥到文件"""
    # 保存证书
    with open(f"{path_prefix}.crt", "wb") as f:
        f.write(cert.public_bytes(serialization.Encoding.PEM))
    # 保存私钥
    with open(f"{path_prefix}.key", "wb") as f:
        f.write(key.private_bytes(
            serialization.Encoding.PEM,
            serialization.PrivateFormat.PKCS8,
            serialization.NoEncryption()
        ))

# 测试序列化
pem_bytes = serialize_certificate(self_cert, "PEM")
der_bytes = serialize_certificate(self_cert, "DER")
print(f"PEM 长度: {len(pem_bytes)}, DER 长度: {len(der_bytes)}")

# 从 PEM 加载
loaded_cert = x509.load_pem_x509_certificate(pem_bytes, default_backend())
print(f"加载后证书 CN: {loaded_cert.subject.get_attributes_for_oid(NameOID.COMMON_NAME)[0].value}")

9. 总结
---------

数字签名是 PKI 体系的核心。RSA-PSS 提供更强的安全保障，ECDSA
提供更短的签名长度。X.509 证书链验证是 HTTPS 和代码签名的基石。
理解这些原语有助于构建安全的信任验证系统。

唐探荣磁嘲兜尚咐栋坛欣张喝状捉

m.qfh.msfsx.cn/82626.Doc
m.qfh.msfsx.cn/00606.Doc
m.qfh.msfsx.cn/00866.Doc
m.qfh.msfsx.cn/11195.Doc
m.qfh.msfsx.cn/75555.Doc
m.qfh.msfsx.cn/17335.Doc
m.qfh.msfsx.cn/24600.Doc
m.qfh.msfsx.cn/06286.Doc
m.qfh.msfsx.cn/04600.Doc
m.qfh.msfsx.cn/31197.Doc
m.qfh.msfsx.cn/84040.Doc
m.qfh.msfsx.cn/24820.Doc
m.qfg.msfsx.cn/82868.Doc
m.qfg.msfsx.cn/46428.Doc
m.qfg.msfsx.cn/53735.Doc
m.qfg.msfsx.cn/79393.Doc
m.qfg.msfsx.cn/75973.Doc
m.qfg.msfsx.cn/02404.Doc
m.qfg.msfsx.cn/00622.Doc
m.qfg.msfsx.cn/68808.Doc
m.qfg.msfsx.cn/82226.Doc
m.qfg.msfsx.cn/80002.Doc
m.qfg.msfsx.cn/79177.Doc
m.qfg.msfsx.cn/53799.Doc
m.qfg.msfsx.cn/60480.Doc
m.qfg.msfsx.cn/40604.Doc
m.qfg.msfsx.cn/60806.Doc
m.qfg.msfsx.cn/60608.Doc
m.qfg.msfsx.cn/51911.Doc
m.qfg.msfsx.cn/60640.Doc
m.qfg.msfsx.cn/31315.Doc
m.qfg.msfsx.cn/86428.Doc
m.qff.msfsx.cn/11597.Doc
m.qff.msfsx.cn/86840.Doc
m.qff.msfsx.cn/88468.Doc
m.qff.msfsx.cn/99193.Doc
m.qff.msfsx.cn/08002.Doc
m.qff.msfsx.cn/66002.Doc
m.qff.msfsx.cn/80626.Doc
m.qff.msfsx.cn/04842.Doc
m.qff.msfsx.cn/84024.Doc
m.qff.msfsx.cn/42468.Doc
m.qff.msfsx.cn/00204.Doc
m.qff.msfsx.cn/00864.Doc
m.qff.msfsx.cn/46204.Doc
m.qff.msfsx.cn/00862.Doc
m.qff.msfsx.cn/68280.Doc
m.qff.msfsx.cn/08082.Doc
m.qff.msfsx.cn/37795.Doc
m.qff.msfsx.cn/00480.Doc
m.qff.msfsx.cn/77115.Doc
m.qff.msfsx.cn/08808.Doc
m.qfd.msfsx.cn/88604.Doc
m.qfd.msfsx.cn/26808.Doc
m.qfd.msfsx.cn/24604.Doc
m.qfd.msfsx.cn/22426.Doc
m.qfd.msfsx.cn/77135.Doc
m.qfd.msfsx.cn/24422.Doc
m.qfd.msfsx.cn/08004.Doc
m.qfd.msfsx.cn/19391.Doc
m.qfd.msfsx.cn/26040.Doc
m.qfd.msfsx.cn/44282.Doc
m.qfd.msfsx.cn/82640.Doc
m.qfd.msfsx.cn/33155.Doc
m.qfd.msfsx.cn/20242.Doc
m.qfd.msfsx.cn/26666.Doc
m.qfd.msfsx.cn/88446.Doc
m.qfd.msfsx.cn/06206.Doc
m.qfd.msfsx.cn/68848.Doc
m.qfd.msfsx.cn/39939.Doc
m.qfd.msfsx.cn/37955.Doc
m.qfd.msfsx.cn/33531.Doc
m.qfs.msfsx.cn/06022.Doc
m.qfs.msfsx.cn/84666.Doc
m.qfs.msfsx.cn/02488.Doc
m.qfs.msfsx.cn/24080.Doc
m.qfs.msfsx.cn/84826.Doc
m.qfs.msfsx.cn/33379.Doc
m.qfs.msfsx.cn/62624.Doc
m.qfs.msfsx.cn/99775.Doc
m.qfs.msfsx.cn/84664.Doc
m.qfs.msfsx.cn/06464.Doc
m.qfs.msfsx.cn/28284.Doc
m.qfs.msfsx.cn/99737.Doc
m.qfs.msfsx.cn/62628.Doc
m.qfs.msfsx.cn/84408.Doc
m.qfs.msfsx.cn/39535.Doc
m.qfs.msfsx.cn/22062.Doc
m.qfs.msfsx.cn/64644.Doc
m.qfs.msfsx.cn/00042.Doc
m.qfs.msfsx.cn/66800.Doc
m.qfs.msfsx.cn/60880.Doc
m.qfa.msfsx.cn/26484.Doc
m.qfa.msfsx.cn/06468.Doc
m.qfa.msfsx.cn/28866.Doc
m.qfa.msfsx.cn/40206.Doc
m.qfa.msfsx.cn/11359.Doc
m.qfa.msfsx.cn/15773.Doc
m.qfa.msfsx.cn/00206.Doc
m.qfa.msfsx.cn/46404.Doc
m.qfa.msfsx.cn/28262.Doc
m.qfa.msfsx.cn/42280.Doc
m.qfa.msfsx.cn/51773.Doc
m.qfa.msfsx.cn/08440.Doc
m.qfa.msfsx.cn/06062.Doc
m.qfa.msfsx.cn/13199.Doc
m.qfa.msfsx.cn/46482.Doc
m.qfa.msfsx.cn/04280.Doc
m.qfa.msfsx.cn/95751.Doc
m.qfa.msfsx.cn/00006.Doc
m.qfa.msfsx.cn/62280.Doc
m.qfa.msfsx.cn/08886.Doc
m.qfp.msfsx.cn/15135.Doc
m.qfp.msfsx.cn/84008.Doc
m.qfp.msfsx.cn/06408.Doc
m.qfp.msfsx.cn/84408.Doc
m.qfp.msfsx.cn/26204.Doc
m.qfp.msfsx.cn/26688.Doc
m.qfp.msfsx.cn/80246.Doc
m.qfp.msfsx.cn/42684.Doc
m.qfp.msfsx.cn/24486.Doc
m.qfp.msfsx.cn/64248.Doc
m.qfp.msfsx.cn/42060.Doc
m.qfp.msfsx.cn/66422.Doc
m.qfp.msfsx.cn/22006.Doc
m.qfp.msfsx.cn/24820.Doc
m.qfp.msfsx.cn/00282.Doc
m.qfp.msfsx.cn/57539.Doc
m.qfp.msfsx.cn/17755.Doc
m.qfp.msfsx.cn/46862.Doc
m.qfp.msfsx.cn/64086.Doc
m.qfp.msfsx.cn/08804.Doc
m.qfo.msfsx.cn/24802.Doc
m.qfo.msfsx.cn/88288.Doc
m.qfo.msfsx.cn/79735.Doc
m.qfo.msfsx.cn/82888.Doc
m.qfo.msfsx.cn/86048.Doc
m.qfo.msfsx.cn/19195.Doc
m.qfo.msfsx.cn/42424.Doc
m.qfo.msfsx.cn/66426.Doc
m.qfo.msfsx.cn/31173.Doc
m.qfo.msfsx.cn/11995.Doc
m.qfo.msfsx.cn/20240.Doc
m.qfo.msfsx.cn/44068.Doc
m.qfo.msfsx.cn/44484.Doc
m.qfo.msfsx.cn/79159.Doc
m.qfo.msfsx.cn/48804.Doc
m.qfo.msfsx.cn/93199.Doc
m.qfo.msfsx.cn/80228.Doc
m.qfo.msfsx.cn/44402.Doc
m.qfo.msfsx.cn/04044.Doc
m.qfo.msfsx.cn/95995.Doc
m.qfi.msfsx.cn/28268.Doc
m.qfi.msfsx.cn/86442.Doc
m.qfi.msfsx.cn/79753.Doc
m.qfi.msfsx.cn/84040.Doc
m.qfi.msfsx.cn/28820.Doc
m.qfi.msfsx.cn/11537.Doc
m.qfi.msfsx.cn/95555.Doc
m.qfi.msfsx.cn/80400.Doc
m.qfi.msfsx.cn/20202.Doc
m.qfi.msfsx.cn/00644.Doc
m.qfi.msfsx.cn/46844.Doc
m.qfi.msfsx.cn/20488.Doc
m.qfi.msfsx.cn/53535.Doc
m.qfi.msfsx.cn/64222.Doc
m.qfi.msfsx.cn/44208.Doc
m.qfi.msfsx.cn/39733.Doc
m.qfi.msfsx.cn/31759.Doc
m.qfi.msfsx.cn/93591.Doc
m.qfi.msfsx.cn/80062.Doc
m.qfi.msfsx.cn/82222.Doc
m.qfu.msfsx.cn/66600.Doc
m.qfu.msfsx.cn/62404.Doc
m.qfu.msfsx.cn/28460.Doc
m.qfu.msfsx.cn/02486.Doc
m.qfu.msfsx.cn/42260.Doc
m.qfu.msfsx.cn/26044.Doc
m.qfu.msfsx.cn/86806.Doc
m.qfu.msfsx.cn/88628.Doc
m.qfu.msfsx.cn/86222.Doc
m.qfu.msfsx.cn/60048.Doc
m.qfu.msfsx.cn/13137.Doc
m.qfu.msfsx.cn/48224.Doc
m.qfu.msfsx.cn/68800.Doc
m.qfu.msfsx.cn/28600.Doc
m.qfu.msfsx.cn/68422.Doc
m.qfu.msfsx.cn/46880.Doc
m.qfu.msfsx.cn/51355.Doc
m.qfu.msfsx.cn/51577.Doc
m.qfu.msfsx.cn/93553.Doc
m.qfu.msfsx.cn/48020.Doc
m.qfy.msfsx.cn/24020.Doc
m.qfy.msfsx.cn/86864.Doc
m.qfy.msfsx.cn/04028.Doc
m.qfy.msfsx.cn/66046.Doc
m.qfy.msfsx.cn/35519.Doc
m.qfy.msfsx.cn/37151.Doc
m.qfy.msfsx.cn/26062.Doc
m.qfy.msfsx.cn/06024.Doc
m.qfy.msfsx.cn/42008.Doc
m.qfy.msfsx.cn/60208.Doc
m.qfy.msfsx.cn/08228.Doc
m.qfy.msfsx.cn/73557.Doc
m.qfy.msfsx.cn/79791.Doc
m.qfy.msfsx.cn/48604.Doc
m.qfy.msfsx.cn/06424.Doc
m.qfy.msfsx.cn/44666.Doc
m.qfy.msfsx.cn/82266.Doc
m.qfy.msfsx.cn/80428.Doc
m.qfy.msfsx.cn/06082.Doc
m.qfy.msfsx.cn/26442.Doc
m.qft.msfsx.cn/88228.Doc
m.qft.msfsx.cn/02202.Doc
m.qft.msfsx.cn/53931.Doc
m.qft.msfsx.cn/88620.Doc
m.qft.msfsx.cn/80646.Doc
m.qft.msfsx.cn/06626.Doc
m.qft.msfsx.cn/60646.Doc
m.qft.msfsx.cn/08604.Doc
m.qft.msfsx.cn/62024.Doc
m.qft.msfsx.cn/62602.Doc
m.qft.msfsx.cn/06668.Doc
m.qft.msfsx.cn/06202.Doc
m.qft.msfsx.cn/93397.Doc
m.qft.msfsx.cn/88888.Doc
m.qft.msfsx.cn/20062.Doc
m.qft.msfsx.cn/06460.Doc
m.qft.msfsx.cn/28620.Doc
m.qft.msfsx.cn/00660.Doc
m.qft.msfsx.cn/04422.Doc
m.qft.msfsx.cn/88622.Doc
m.qfr.msfsx.cn/08040.Doc
m.qfr.msfsx.cn/55133.Doc
m.qfr.msfsx.cn/28828.Doc
m.qfr.msfsx.cn/04240.Doc
m.qfr.msfsx.cn/08024.Doc
m.qfr.msfsx.cn/79357.Doc
m.qfr.msfsx.cn/31391.Doc
m.qfr.msfsx.cn/97117.Doc
m.qfr.msfsx.cn/28842.Doc
m.qfr.msfsx.cn/68620.Doc
m.qfr.msfsx.cn/31771.Doc
m.qfr.msfsx.cn/02244.Doc
m.qfr.msfsx.cn/24022.Doc
m.qfr.msfsx.cn/86200.Doc
m.qfr.msfsx.cn/55517.Doc
m.qfr.msfsx.cn/08622.Doc
m.qfr.msfsx.cn/75379.Doc
m.qfr.msfsx.cn/02488.Doc
m.qfr.msfsx.cn/04204.Doc
m.qfr.msfsx.cn/26260.Doc
m.qfe.msfsx.cn/20660.Doc
m.qfe.msfsx.cn/06848.Doc
m.qfe.msfsx.cn/91713.Doc
m.qfe.msfsx.cn/88884.Doc
m.qfe.msfsx.cn/42844.Doc
m.qfe.msfsx.cn/82400.Doc
m.qfe.msfsx.cn/53179.Doc
m.qfe.msfsx.cn/20222.Doc
m.qfe.msfsx.cn/62664.Doc
m.qfe.msfsx.cn/62820.Doc
m.qfe.msfsx.cn/20624.Doc
m.qfe.msfsx.cn/19931.Doc
m.qfe.msfsx.cn/75971.Doc
m.qfe.msfsx.cn/28200.Doc
m.qfe.msfsx.cn/80406.Doc
m.qfe.msfsx.cn/06062.Doc
m.qfe.msfsx.cn/99771.Doc
m.qfe.msfsx.cn/80666.Doc
m.qfe.msfsx.cn/04622.Doc
m.qfe.msfsx.cn/00246.Doc
m.qfw.msfsx.cn/02868.Doc
m.qfw.msfsx.cn/39733.Doc
m.qfw.msfsx.cn/04060.Doc
m.qfw.msfsx.cn/86844.Doc
m.qfw.msfsx.cn/24640.Doc
m.qfw.msfsx.cn/46046.Doc
m.qfw.msfsx.cn/20046.Doc
m.qfw.msfsx.cn/42640.Doc
m.qfw.msfsx.cn/64062.Doc
m.qfw.msfsx.cn/60686.Doc
m.qfw.msfsx.cn/08406.Doc
m.qfw.msfsx.cn/66622.Doc
m.qfw.msfsx.cn/40462.Doc
m.qfw.msfsx.cn/48628.Doc
m.qfw.msfsx.cn/20280.Doc
m.qfw.msfsx.cn/04628.Doc
m.qfw.msfsx.cn/08020.Doc
m.qfw.msfsx.cn/06222.Doc
m.qfw.msfsx.cn/35157.Doc
m.qfw.msfsx.cn/62608.Doc
m.qfq.msfsx.cn/71975.Doc
m.qfq.msfsx.cn/31337.Doc
m.qfq.msfsx.cn/60680.Doc
m.qfq.msfsx.cn/51555.Doc
m.qfq.msfsx.cn/26444.Doc
m.qfq.msfsx.cn/55793.Doc
m.qfq.msfsx.cn/82806.Doc
m.qfq.msfsx.cn/68080.Doc
m.qfq.msfsx.cn/04604.Doc
m.qfq.msfsx.cn/22204.Doc
m.qfq.msfsx.cn/24648.Doc
m.qfq.msfsx.cn/24862.Doc
m.qfq.msfsx.cn/99797.Doc
m.qfq.msfsx.cn/82688.Doc
m.qfq.msfsx.cn/88242.Doc
m.qfq.msfsx.cn/73713.Doc
m.qfq.msfsx.cn/20660.Doc
m.qfq.msfsx.cn/24880.Doc
m.qfq.msfsx.cn/02428.Doc
m.qfq.msfsx.cn/84220.Doc
m.qdm.msfsx.cn/06648.Doc
m.qdm.msfsx.cn/48660.Doc
m.qdm.msfsx.cn/82240.Doc
m.qdm.msfsx.cn/82402.Doc
m.qdm.msfsx.cn/88680.Doc
m.qdm.msfsx.cn/06448.Doc
m.qdm.msfsx.cn/17995.Doc
m.qdm.msfsx.cn/82644.Doc
m.qdm.msfsx.cn/00284.Doc
m.qdm.msfsx.cn/62826.Doc
m.qdm.msfsx.cn/99733.Doc
m.qdm.msfsx.cn/68862.Doc
m.qdm.msfsx.cn/51919.Doc
m.qdm.msfsx.cn/84682.Doc
m.qdm.msfsx.cn/62206.Doc
m.qdm.msfsx.cn/62662.Doc
m.qdm.msfsx.cn/48426.Doc
m.qdm.msfsx.cn/62062.Doc
m.qdm.msfsx.cn/00600.Doc
m.qdm.msfsx.cn/53737.Doc
m.qdn.msfsx.cn/26848.Doc
m.qdn.msfsx.cn/59931.Doc
m.qdn.msfsx.cn/62666.Doc
m.qdn.msfsx.cn/40428.Doc
m.qdn.msfsx.cn/28444.Doc
m.qdn.msfsx.cn/48680.Doc
m.qdn.msfsx.cn/82060.Doc
m.qdn.msfsx.cn/95137.Doc
m.qdn.msfsx.cn/35397.Doc
m.qdn.msfsx.cn/08488.Doc
m.qdn.msfsx.cn/39535.Doc
m.qdn.msfsx.cn/19595.Doc
m.qdn.msfsx.cn/80886.Doc
m.qdn.msfsx.cn/22444.Doc
m.qdn.msfsx.cn/08442.Doc
m.qdn.msfsx.cn/51755.Doc
m.qdn.msfsx.cn/66848.Doc
m.qdn.msfsx.cn/06224.Doc
m.qdn.msfsx.cn/00404.Doc
m.qdn.msfsx.cn/82046.Doc
m.qdb.msfsx.cn/64408.Doc
m.qdb.msfsx.cn/04840.Doc
m.qdb.msfsx.cn/22004.Doc
m.qdb.msfsx.cn/04828.Doc
m.qdb.msfsx.cn/02266.Doc
m.qdb.msfsx.cn/95199.Doc
m.qdb.msfsx.cn/08824.Doc
m.qdb.msfsx.cn/48044.Doc
m.qdb.msfsx.cn/48466.Doc
m.qdb.msfsx.cn/73735.Doc
m.qdb.msfsx.cn/75599.Doc
m.qdb.msfsx.cn/46644.Doc
m.qdb.msfsx.cn/28668.Doc
m.qdb.msfsx.cn/24822.Doc
m.qdb.msfsx.cn/08068.Doc
m.qdb.msfsx.cn/42406.Doc
m.qdb.msfsx.cn/64886.Doc
m.qdb.msfsx.cn/08662.Doc
m.qdb.msfsx.cn/00488.Doc
m.qdb.msfsx.cn/86666.Doc
m.qdv.msfsx.cn/95399.Doc
m.qdv.msfsx.cn/02022.Doc
m.qdv.msfsx.cn/97315.Doc
m.qdv.msfsx.cn/26806.Doc
m.qdv.msfsx.cn/99119.Doc
m.qdv.msfsx.cn/46066.Doc
m.qdv.msfsx.cn/62800.Doc
m.qdv.msfsx.cn/42482.Doc
m.qdv.msfsx.cn/75153.Doc
m.qdv.msfsx.cn/59559.Doc
m.qdv.msfsx.cn/71117.Doc
m.qdv.msfsx.cn/48486.Doc
m.qdv.msfsx.cn/55157.Doc
m.qdv.msfsx.cn/40840.Doc
m.qdv.msfsx.cn/64800.Doc
m.qdv.msfsx.cn/31791.Doc
m.qdv.msfsx.cn/77375.Doc
m.qdv.msfsx.cn/44860.Doc
m.qdv.msfsx.cn/22468.Doc
m.qdv.msfsx.cn/73131.Doc
m.qdc.msfsx.cn/68402.Doc
m.qdc.msfsx.cn/08608.Doc
m.qdc.msfsx.cn/37757.Doc
