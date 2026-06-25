Python uuid 与唯一标识
==============================

uuid 模块生成统一唯一标识符 (UUID), 有多种版本满足不同场景。

1. uuid1 — 基于时间的 UUID
-----------------------------

import uuid

# UUID1: 基于当前时间戳 + 节点(MAC地址) + 时钟序列
# 优势: 时间有序, 可追踪生成时间
# 劣势: 会暴露 MAC 地址 (隐私问题)
u1 = uuid.uuid1()
print("UUID1:", u1)
print("UUID1 十六进制:", u1.hex)               # 32 字符十六进制
print("UUID1 版本:", u1.version)               # 1

# UUID1 字段解析
print("时间戳字段:", u1.time)                   # 100纳秒为单位的时间戳
print("时钟序列:", u1.clock_seq)                # 时钟序列 (14位)
print("节点(MAC):", u1.node)                    # 48位 MAC 地址

# 自定义节点: 避免暴露真实 MAC 地址
custom_node = 0x123456789ABC
u1_custom = uuid.uuid1(node=custom_node)
print("自定义节点 UUID1:", u1_custom)

# 自定义时钟序列
u1_clock = uuid.uuid1(clock_seq=0x3FFF)        # 时钟序列范围 0-0x3FFF
print("自定义时钟 UUID1:", u1_clock)

# 从 UUID1 提取时间
from datetime import datetime, timezone
time_100ns = u1.time - 0x01b21dd213814000      # UUID1 时间的偏移量
timestamp_sec = time_100ns / 10000000
dt = datetime.fromtimestamp(timestamp_sec, tz=timezone.utc)
print("UUID1 生成时间 (UTC):", dt)

2. uuid4 — 随机 UUID
----------------------

import uuid

# UUID4: 完全随机生成 (122位随机数 + 6位版本/变体位)
# 优势: 无隐私泄露, 适合大多数应用场景
# 劣势: 无序, 数据库索引性能稍差
u4 = uuid.uuid4()
print("UUID4:", u4)
print("UUID4 版本:", u4.version)               # 4
print("UUID4 字节:", u4.bytes)                 # 16字节原始数据
print("UUID4 整数表示:", u4.int)                # 128位整数

# 生成批量 UUID (列表推导式)
batch = [uuid.uuid4() for _ in range(5)]
print("批量 UUID4:")
for i, uid in enumerate(batch):
    print(f"  {i+1}: {uid}")

# UUID4 碰撞概率: 极其微小
# 每秒生成 10 亿个 UUID4, 约 100 年后才可能发生一次碰撞
# 公式: 碰撞概率 ~ n^2 / (2 * 2^122)

3. uuid3 / uuid5 — 基于名称的 UUID
--------------------------------------

import uuid

# UUID3: 基于命名空间 + 名称的 MD5 哈希 (128位)
# UUID5: 基于命名空间 + 名称的 SHA-1 哈希 (128位)

# 预定义命名空间:
# NAMESPACE_DNS — 域名
# NAMESPACE_URL — URL
# NAMESPACE_OID — ISO OID
# NAMESPACE_X500 — X.500 DN

# UUID5: SHA-1 版 (推荐)
ns_dns = uuid.NAMESPACE_DNS
u5_domain = uuid.uuid5(ns_dns, "example.com")
print("UUID5 (DNS):", u5_domain)              # 对 example.com 的一致 UUID

# 同一名称在同一命名空间下总是生成相同 UUID
u5_domain2 = uuid.uuid5(ns_dns, "example.com")
print("UUID5 相同:", u5_domain == u5_domain2)  # True

# UUID3: MD5 版 (兼容旧系统)
u3_domain = uuid.uuid3(ns_dns, "example.com")
print("UUID3 (DNS):", u3_domain)
print("UUID3 长度:", len(u3_domain.hex))       # 32

# 不同命名空间下相同名称不同结果
u5_url = uuid.uuid5(uuid.NAMESPACE_URL, "example.com")
print("UUID5 (URL):", u5_url)                 # 与 DNS 版不同

# 实用场景: 为资源生成确定性 UUID
def resource_uuid(resource_type, resource_id):
    """为资源生成稳定的 UUID5 标识"""
    ns = uuid.uuid5(uuid.NAMESPACE_DNS, "myapp.example.com")
    return uuid.uuid5(ns, f"{resource_type}:{resource_id}")

user_uuid = resource_uuid("user", "alice")
post_uuid = resource_uuid("post", "12345")
print("用户 UUID:", user_uuid)
print("文章 UUID:", post_uuid)

4. UUID 对象字段与操作
-------------------------

import uuid

u = uuid.uuid4()

# 基本字段
print("UUID 字符串:", str(u))                  # 标准格式: 8-4-4-4-12
print("UUID hex:", u.hex)                      # 32字符十六进制
print("UUID 字节:", u.bytes)                   # 16字节 (bytes)
print("UUID 字节序 LE:", u.bytes_le)           # 混合字节序 (小端 time 域)
print("UUID int:", u.int)                      # 128位整数表示
print("UUID 变体:", u.variant)                 # RFC 4122
print("UUID 版本:", u.version)                 # 4

# UUID 比较和排序
u1 = uuid.uuid4()
u2 = uuid.uuid4()
print("u1 < u2:", u1 < u2)                     # 按整数值比较
print("相等判断:", u1 == u1)                    # True

# 从不同格式创建 UUID
u_from_hex = uuid.UUID(hex="550e8400e29b41d4a716446655440000")
u_from_bytes = uuid.UUID(bytes=b"\x55\x0e\x84\x00\xe2\x9b\x41\xd4\xa7\x16\x44\x66\x55\x44\x00\x00")
u_from_int = uuid.UUID(int=0x550e8400e29b41d4a716446655440000)
print("从 hex 创建:", u_from_hex)
print("从 bytes 创建:", u_from_bytes)
print("从 int 创建:", u_from_int)

# UUID 的不可变性
u_frozen = uuid.uuid4()
# u_frozen.int = 123   # 会报错! UUID 不可变

# 将 UUID 用于字典键
cache = {uuid.uuid4(): "value1", uuid.uuid4(): "value2"}
print("UUID 作为字典键:", len(cache))

5. UUID 与数据库主键策略
----------------------------

import uuid

# 场景: 使用 UUID 作为分布式数据库主键
# 优势: 全局唯一, 无需中心化序列, 安全 (不暴露数据量)
# 劣势: 16字节 vs 4字节 int, 索引效率较低

# 策略1: 直接用 UUID 字符串
class ModelWithUUID:
    def __init__(self, name: str):
        self.id = uuid.uuid4()
        self.name = name

    def to_db(self):
        """存储到数据库: UUID 转 hex 或 bytes"""
        return {
            "id_hex": self.id.hex,              # 32 字符字符串
            "id_bytes": self.id.bytes,          # 16 字节 (需 BINARY(16) 类型)
            "id_str": str(self.id),             # 36 字符字符串
        }

entity = ModelWithUUID("测试数据")
print("数据库存储准备:", entity.to_db())

# 策略2: 有序 UUID (类似 UUID1 但更安全)
# 可结合时间戳 + 随机数自行生成有序 UUID
import time
import random

def ordered_uuid():
    """生成时间有序的 UUID (类似 UUID7 方案)"""
    # 自定义: 毫秒时间戳(高48位) + 随机数(低80位)
    timestamp_ms = int(time.time() * 1000)
    rand_bits = random.getrandbits(80)
    # 组合成 128 位 UUID
    value = (timestamp_ms << 80) | rand_bits
    # 设置版本位 (4) 和变体位
    value &= ~(0xC000 << 64)                   # 清除变体位
    value |= 0x8000 << 64                      # 设置变体 (RFC 4122)
    value &= ~(0xF000 << 76)                   # 清除版本位
    value |= 4 << 76                           # 设置版本 4
    return uuid.UUID(int=value)

print("有序 UUID:", ordered_uuid())

# 策略3: PostgreSQL 的 uuid-ossp 扩展兼容
# PSQL: CREATE EXTENSION "uuid-ossp";
#       uuid_generate_v4() -> UUID 类型列

6. Snowflake ID 模式
-------------------------

import time
import threading

class SnowflakeGenerator:
    """简易雪花 ID 生成器 (64位整数, 非 UUID)"""

    def __init__(self, worker_id: int, datacenter_id: int, sequence: int = 0):
        self.worker_id = worker_id & 0x1F              # 5位 工作节点ID
        self.datacenter_id = datacenter_id & 0x1F      # 5位 数据中心ID
        self.sequence = sequence & 0xFFF               # 12位 序列号
        self.tw_epoch = 1609459200000                   # 自定义纪元 (2021-01-01)
        self.last_timestamp = -1
        self.lock = threading.Lock()

    def _current_millis(self) -> int:
        return int(time.time() * 1000)

    def _wait_next_millis(self, last_ts: int) -> int:
        ts = self._current_millis()
        while ts <= last_ts:
            ts = self._current_millis()
        return ts

    def next_id(self) -> int:
        """生成下一个 64 位雪花 ID"""
        with self.lock:
            timestamp = self._current_millis()

            # 时钟回拨处理 (简化版, 仅等待)
            if timestamp < self.last_timestamp:
                timestamp = self._wait_next_millis(self.last_timestamp)

            if timestamp == self.last_timestamp:
                self.sequence = (self.sequence + 1) & 0xFFF
                if self.sequence == 0:                  # 序列号用完
                    timestamp = self._wait_next_millis(self.last_timestamp)
            else:
                self.sequence = 0

            self.last_timestamp = timestamp

            # 组合: 时间戳(41位) + 数据中心(5位) + 工作节点(5位) + 序列号(12位)
            snowflake_id = (
                ((timestamp - self.tw_epoch) << 22) |
                (self.datacenter_id << 17) |
                (self.worker_id << 12) |
                self.sequence
            )
            return snowflake_id

    def parse(self, snowflake_id: int) -> dict:
        """解析雪花 ID 中的各字段"""
        sequence = snowflake_id & 0xFFF
        worker_id = (snowflake_id >> 12) & 0x1F
        datacenter_id = (snowflake_id >> 17) & 0x1F
        timestamp = (snowflake_id >> 22) + self.tw_epoch
        return {
            "id": snowflake_id,
            "timestamp_ms": timestamp,
            "datacenter_id": datacenter_id,
            "worker_id": worker_id,
            "sequence": sequence,
        }

# 测试雪花 ID
snowflake = SnowflakeGenerator(worker_id=1, datacenter_id=0)
ids = [snowflake.next_id() for _ in range(3)]
print("雪花 ID 列表:", ids)
for sid in ids:
    info = snowflake.parse(sid)
    dt = datetime.fromtimestamp(info["timestamp_ms"] / 1000, tz=timezone.utc)
    print(f"  ID={sid}, 时间={dt}, 数据中心={info['datacenter_id']}, 节点={info['worker_id']}, 序列={info['sequence']}")

7. UUID 的转换与编码
-----------------------

import uuid

# UUID 转不同格式
u = uuid.uuid4()

# 标准 36 字符格式
print("标准格式:", str(u))           # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# 32 字符十六进制 (不带连字符)
print("Hex 格式:", u.hex)           # 32 字符

# 16 字节 (二进制紧凑格式)
print("Bytes 格式:", u.bytes)       # 16 字节

# Base64 编码 (更紧凑的字符串表示, 22 字符)
import base64
base64_str = base64.urlsafe_b64encode(u.bytes).rstrip(b"=").decode()
print("Base64 格式:", base64_str)   # 22 字符

# Base64 解码回 UUID
decoded_bytes = base64.urlsafe_b64decode(base64_str + "==")
restored_uuid = uuid.UUID(bytes=decoded_bytes)
print("恢复 UUID:", restored_uuid)
print("一致?", u == restored_uuid)

# 短 UUID 方案: Base62 编码
BASE62 = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

def uuid_to_base62(uid: uuid.UUID) -> str:
    """将 UUID 编码为 Base62 字符串 (约 22 字符)"""
    num = uid.int
    if num == 0:
        return BASE62[0]
    result = []
    while num > 0:
        result.append(BASE62[num % 62])
        num //= 62
    return "".join(reversed(result))

short_id = uuid_to_base62(u)
print("Base62 短 ID:", short_id)
print("长度:", len(short_id))                     # 约 22 字符

总结: uuid4 最适合通用场景; uuid5 用于确定性标识; uuid1 用于时间有序; 雪花 ID 适合高性能分布式系统.

dmu.0wp4aoe.cn/99133.Doc
dmu.0wp4aoe.cn/75713.Doc
dmu.0wp4aoe.cn/99955.Doc
dmu.0wp4aoe.cn/33339.Doc
dmu.0wp4aoe.cn/00482.Doc
dmu.0wp4aoe.cn/33575.Doc
dmu.0wp4aoe.cn/77519.Doc
dmu.0wp4aoe.cn/19551.Doc
dmu.0wp4aoe.cn/73951.Doc
dmu.0wp4aoe.cn/51513.Doc
dmy.0wp4aoe.cn/71191.Doc
dmy.0wp4aoe.cn/59317.Doc
dmy.0wp4aoe.cn/53959.Doc
dmy.0wp4aoe.cn/37515.Doc
dmy.0wp4aoe.cn/75517.Doc
dmy.0wp4aoe.cn/75315.Doc
dmy.0wp4aoe.cn/37559.Doc
dmy.0wp4aoe.cn/95319.Doc
dmy.0wp4aoe.cn/53913.Doc
dmy.0wp4aoe.cn/97119.Doc
dmt.0wp4aoe.cn/40208.Doc
dmt.0wp4aoe.cn/02442.Doc
dmt.0wp4aoe.cn/91571.Doc
dmt.0wp4aoe.cn/59751.Doc
dmt.0wp4aoe.cn/97751.Doc
dmt.0wp4aoe.cn/37739.Doc
dmt.0wp4aoe.cn/79957.Doc
dmt.0wp4aoe.cn/59915.Doc
dmt.0wp4aoe.cn/95515.Doc
dmt.0wp4aoe.cn/99933.Doc
dmr.0wp4aoe.cn/33353.Doc
dmr.0wp4aoe.cn/33711.Doc
dmr.0wp4aoe.cn/77155.Doc
dmr.0wp4aoe.cn/59379.Doc
dmr.0wp4aoe.cn/06864.Doc
dmr.0wp4aoe.cn/93117.Doc
dmr.0wp4aoe.cn/66244.Doc
dmr.0wp4aoe.cn/15173.Doc
dmr.0wp4aoe.cn/99377.Doc
dmr.0wp4aoe.cn/37591.Doc
dme.0wp4aoe.cn/11139.Doc
dme.0wp4aoe.cn/91395.Doc
dme.0wp4aoe.cn/95973.Doc
dme.0wp4aoe.cn/11517.Doc
dme.0wp4aoe.cn/37757.Doc
dme.0wp4aoe.cn/15739.Doc
dme.0wp4aoe.cn/39339.Doc
dme.0wp4aoe.cn/19119.Doc
dme.0wp4aoe.cn/93711.Doc
dme.0wp4aoe.cn/75113.Doc
dmw.0wp4aoe.cn/95175.Doc
dmw.0wp4aoe.cn/71151.Doc
dmw.0wp4aoe.cn/17595.Doc
dmw.0wp4aoe.cn/71977.Doc
dmw.0wp4aoe.cn/79917.Doc
dmw.0wp4aoe.cn/93591.Doc
dmw.0wp4aoe.cn/55751.Doc
dmw.0wp4aoe.cn/31739.Doc
dmw.0wp4aoe.cn/11595.Doc
dmw.0wp4aoe.cn/35593.Doc
dmq.0wp4aoe.cn/71359.Doc
dmq.0wp4aoe.cn/39553.Doc
dmq.0wp4aoe.cn/35919.Doc
dmq.0wp4aoe.cn/91733.Doc
dmq.0wp4aoe.cn/37539.Doc
dmq.0wp4aoe.cn/57751.Doc
dmq.0wp4aoe.cn/55159.Doc
dmq.0wp4aoe.cn/71119.Doc
dmq.0wp4aoe.cn/35333.Doc
dmq.0wp4aoe.cn/79115.Doc
dnm.0wp4aoe.cn/33575.Doc
dnm.0wp4aoe.cn/97515.Doc
dnm.0wp4aoe.cn/31793.Doc
dnm.0wp4aoe.cn/59911.Doc
dnm.0wp4aoe.cn/77739.Doc
dnm.0wp4aoe.cn/91379.Doc
dnm.0wp4aoe.cn/39797.Doc
dnm.0wp4aoe.cn/31711.Doc
dnm.0wp4aoe.cn/35599.Doc
dnm.0wp4aoe.cn/37977.Doc
dnn.0wp4aoe.cn/17731.Doc
dnn.0wp4aoe.cn/39597.Doc
dnn.0wp4aoe.cn/59171.Doc
dnn.0wp4aoe.cn/13915.Doc
dnn.0wp4aoe.cn/73179.Doc
dnn.0wp4aoe.cn/3.Doc
dnn.0wp4aoe.cn/77353.Doc
dnn.0wp4aoe.cn/53557.Doc
dnn.0wp4aoe.cn/73751.Doc
dnn.0wp4aoe.cn/77171.Doc
dnb.0wp4aoe.cn/33535.Doc
dnb.0wp4aoe.cn/75911.Doc
dnb.0wp4aoe.cn/57771.Doc
dnb.0wp4aoe.cn/79375.Doc
dnb.0wp4aoe.cn/73995.Doc
dnb.0wp4aoe.cn/75593.Doc
dnb.0wp4aoe.cn/93353.Doc
dnb.0wp4aoe.cn/31599.Doc
dnb.0wp4aoe.cn/53771.Doc
dnb.0wp4aoe.cn/57177.Doc
dnv.0wp4aoe.cn/31357.Doc
dnv.0wp4aoe.cn/28484.Doc
dnv.0wp4aoe.cn/35191.Doc
dnv.0wp4aoe.cn/13733.Doc
dnv.0wp4aoe.cn/31137.Doc
dnv.0wp4aoe.cn/91113.Doc
dnv.0wp4aoe.cn/33591.Doc
dnv.0wp4aoe.cn/22020.Doc
dnv.0wp4aoe.cn/99191.Doc
dnv.0wp4aoe.cn/57735.Doc
dnc.0wp4aoe.cn/95915.Doc
dnc.0wp4aoe.cn/37333.Doc
dnc.0wp4aoe.cn/51975.Doc
dnc.0wp4aoe.cn/33159.Doc
dnc.0wp4aoe.cn/91759.Doc
dnc.0wp4aoe.cn/57159.Doc
dnc.0wp4aoe.cn/11933.Doc
dnc.0wp4aoe.cn/17555.Doc
dnc.0wp4aoe.cn/11999.Doc
dnc.0wp4aoe.cn/11973.Doc
dnx.0wp4aoe.cn/77339.Doc
dnx.0wp4aoe.cn/59797.Doc
dnx.0wp4aoe.cn/15573.Doc
dnx.0wp4aoe.cn/35935.Doc
dnx.0wp4aoe.cn/93919.Doc
dnx.0wp4aoe.cn/95139.Doc
dnx.0wp4aoe.cn/99173.Doc
dnx.0wp4aoe.cn/59715.Doc
dnx.0wp4aoe.cn/97371.Doc
dnx.0wp4aoe.cn/39171.Doc
dnz.0wp4aoe.cn/99755.Doc
dnz.0wp4aoe.cn/60826.Doc
dnz.0wp4aoe.cn/17571.Doc
dnz.0wp4aoe.cn/93179.Doc
dnz.0wp4aoe.cn/15795.Doc
dnz.0wp4aoe.cn/59319.Doc
dnz.0wp4aoe.cn/91931.Doc
dnz.0wp4aoe.cn/35551.Doc
dnz.0wp4aoe.cn/19913.Doc
dnz.0wp4aoe.cn/79175.Doc
dnl.0wp4aoe.cn/97375.Doc
dnl.0wp4aoe.cn/31797.Doc
dnl.0wp4aoe.cn/39515.Doc
dnl.0wp4aoe.cn/55173.Doc
dnl.0wp4aoe.cn/17779.Doc
dnl.0wp4aoe.cn/19939.Doc
dnl.0wp4aoe.cn/35955.Doc
dnl.0wp4aoe.cn/51315.Doc
dnl.0wp4aoe.cn/53157.Doc
dnl.0wp4aoe.cn/11775.Doc
dnk.0wp4aoe.cn/97377.Doc
dnk.0wp4aoe.cn/37757.Doc
dnk.0wp4aoe.cn/93159.Doc
dnk.0wp4aoe.cn/55739.Doc
dnk.0wp4aoe.cn/79795.Doc
dnk.0wp4aoe.cn/11937.Doc
dnk.0wp4aoe.cn/37519.Doc
dnk.0wp4aoe.cn/13555.Doc
dnk.0wp4aoe.cn/37997.Doc
dnk.0wp4aoe.cn/71539.Doc
dnj.0wp4aoe.cn/35173.Doc
dnj.0wp4aoe.cn/19379.Doc
dnj.0wp4aoe.cn/37771.Doc
dnj.0wp4aoe.cn/77511.Doc
dnj.0wp4aoe.cn/53911.Doc
dnj.0wp4aoe.cn/13997.Doc
dnj.0wp4aoe.cn/91591.Doc
dnj.0wp4aoe.cn/64224.Doc
dnj.0wp4aoe.cn/37179.Doc
dnj.0wp4aoe.cn/99777.Doc
dnh.0wp4aoe.cn/73333.Doc
dnh.0wp4aoe.cn/55919.Doc
dnh.0wp4aoe.cn/39153.Doc
dnh.0wp4aoe.cn/37357.Doc
dnh.0wp4aoe.cn/17379.Doc
dnh.0wp4aoe.cn/91517.Doc
dnh.0wp4aoe.cn/88046.Doc
dnh.0wp4aoe.cn/91555.Doc
dnh.0wp4aoe.cn/71573.Doc
dnh.0wp4aoe.cn/31399.Doc
dng.0wp4aoe.cn/57373.Doc
dng.0wp4aoe.cn/35577.Doc
dng.0wp4aoe.cn/35771.Doc
dng.0wp4aoe.cn/53359.Doc
dng.0wp4aoe.cn/33139.Doc
dng.0wp4aoe.cn/51313.Doc
dng.0wp4aoe.cn/31797.Doc
dng.0wp4aoe.cn/13733.Doc
dng.0wp4aoe.cn/71931.Doc
dng.0wp4aoe.cn/93553.Doc
dnf.0wp4aoe.cn/62888.Doc
dnf.0wp4aoe.cn/13553.Doc
dnf.0wp4aoe.cn/91313.Doc
dnf.0wp4aoe.cn/33159.Doc
dnf.0wp4aoe.cn/64066.Doc
dnf.0wp4aoe.cn/35977.Doc
dnf.0wp4aoe.cn/55557.Doc
dnf.0wp4aoe.cn/99191.Doc
dnf.0wp4aoe.cn/88208.Doc
dnf.0wp4aoe.cn/46628.Doc
dnd.0wp4aoe.cn/33931.Doc
dnd.0wp4aoe.cn/35579.Doc
dnd.0wp4aoe.cn/55375.Doc
dnd.0wp4aoe.cn/95351.Doc
dnd.0wp4aoe.cn/97137.Doc
dnd.0wp4aoe.cn/35937.Doc
dnd.0wp4aoe.cn/79735.Doc
dnd.0wp4aoe.cn/17751.Doc
dnd.0wp4aoe.cn/15113.Doc
dnd.0wp4aoe.cn/53519.Doc
dns.0wp4aoe.cn/91539.Doc
dns.0wp4aoe.cn/71313.Doc
dns.0wp4aoe.cn/11733.Doc
dns.0wp4aoe.cn/33771.Doc
dns.0wp4aoe.cn/73975.Doc
dns.0wp4aoe.cn/79951.Doc
dns.0wp4aoe.cn/57713.Doc
dns.0wp4aoe.cn/73917.Doc
dns.0wp4aoe.cn/17315.Doc
dns.0wp4aoe.cn/91197.Doc
dna.0wp4aoe.cn/33999.Doc
dna.0wp4aoe.cn/37319.Doc
dna.0wp4aoe.cn/17311.Doc
dna.0wp4aoe.cn/37391.Doc
dna.0wp4aoe.cn/31199.Doc
dna.0wp4aoe.cn/55971.Doc
dna.0wp4aoe.cn/35353.Doc
dna.0wp4aoe.cn/97571.Doc
dna.0wp4aoe.cn/37131.Doc
dna.0wp4aoe.cn/71391.Doc
dnp.0wp4aoe.cn/19179.Doc
dnp.0wp4aoe.cn/15355.Doc
dnp.0wp4aoe.cn/77195.Doc
dnp.0wp4aoe.cn/77559.Doc
dnp.0wp4aoe.cn/19917.Doc
dnp.0wp4aoe.cn/59979.Doc
dnp.0wp4aoe.cn/79353.Doc
dnp.0wp4aoe.cn/37331.Doc
dnp.0wp4aoe.cn/77791.Doc
dnp.0wp4aoe.cn/77755.Doc
dno.0wp4aoe.cn/77139.Doc
dno.0wp4aoe.cn/55715.Doc
dno.0wp4aoe.cn/99399.Doc
dno.0wp4aoe.cn/51951.Doc
dno.0wp4aoe.cn/33131.Doc
dno.0wp4aoe.cn/31757.Doc
dno.0wp4aoe.cn/71399.Doc
dno.0wp4aoe.cn/13753.Doc
dno.0wp4aoe.cn/77997.Doc
dno.0wp4aoe.cn/31573.Doc
dni.0wp4aoe.cn/51779.Doc
dni.0wp4aoe.cn/11359.Doc
dni.0wp4aoe.cn/15797.Doc
dni.0wp4aoe.cn/31335.Doc
dni.0wp4aoe.cn/75595.Doc
dni.0wp4aoe.cn/95753.Doc
dni.0wp4aoe.cn/71371.Doc
dni.0wp4aoe.cn/95931.Doc
dni.0wp4aoe.cn/51315.Doc
dni.0wp4aoe.cn/15177.Doc
dnu.0wp4aoe.cn/35357.Doc
dnu.0wp4aoe.cn/37953.Doc
dnu.0wp4aoe.cn/77737.Doc
dnu.0wp4aoe.cn/99173.Doc
dnu.0wp4aoe.cn/53531.Doc
dnu.0wp4aoe.cn/17519.Doc
dnu.0wp4aoe.cn/55331.Doc
dnu.0wp4aoe.cn/35711.Doc
dnu.0wp4aoe.cn/93139.Doc
dnu.0wp4aoe.cn/95511.Doc
dny.0wp4aoe.cn/97939.Doc
dny.0wp4aoe.cn/99791.Doc
dny.0wp4aoe.cn/11771.Doc
dny.0wp4aoe.cn/53171.Doc
dny.0wp4aoe.cn/19977.Doc
dny.0wp4aoe.cn/51935.Doc
dny.0wp4aoe.cn/91913.Doc
dny.0wp4aoe.cn/55735.Doc
dny.0wp4aoe.cn/53379.Doc
dny.0wp4aoe.cn/39591.Doc
dnt.0wp4aoe.cn/48202.Doc
dnt.0wp4aoe.cn/77755.Doc
dnt.0wp4aoe.cn/13793.Doc
dnt.0wp4aoe.cn/33313.Doc
dnt.0wp4aoe.cn/57593.Doc
dnt.0wp4aoe.cn/33755.Doc
dnt.0wp4aoe.cn/93553.Doc
dnt.0wp4aoe.cn/53937.Doc
dnt.0wp4aoe.cn/59995.Doc
dnt.0wp4aoe.cn/44840.Doc
dnr.0wp4aoe.cn/37757.Doc
dnr.0wp4aoe.cn/93959.Doc
dnr.0wp4aoe.cn/77775.Doc
dnr.0wp4aoe.cn/11311.Doc
dnr.0wp4aoe.cn/93955.Doc
dnr.0wp4aoe.cn/19751.Doc
dnr.0wp4aoe.cn/77959.Doc
dnr.0wp4aoe.cn/33997.Doc
dnr.0wp4aoe.cn/11771.Doc
dnr.0wp4aoe.cn/95357.Doc
dne.0wp4aoe.cn/75731.Doc
dne.0wp4aoe.cn/46882.Doc
dne.0wp4aoe.cn/35959.Doc
dne.0wp4aoe.cn/79973.Doc
dne.0wp4aoe.cn/79937.Doc
dne.0wp4aoe.cn/17917.Doc
dne.0wp4aoe.cn/71155.Doc
dne.0wp4aoe.cn/19775.Doc
dne.0wp4aoe.cn/31179.Doc
dne.0wp4aoe.cn/93593.Doc
dnw.0wp4aoe.cn/73377.Doc
dnw.0wp4aoe.cn/77739.Doc
dnw.0wp4aoe.cn/97975.Doc
dnw.0wp4aoe.cn/39119.Doc
dnw.0wp4aoe.cn/51597.Doc
dnw.0wp4aoe.cn/17351.Doc
dnw.0wp4aoe.cn/51753.Doc
dnw.0wp4aoe.cn/59593.Doc
dnw.0wp4aoe.cn/37733.Doc
dnw.0wp4aoe.cn/31391.Doc
dnq.0wp4aoe.cn/53379.Doc
dnq.0wp4aoe.cn/73991.Doc
dnq.0wp4aoe.cn/79713.Doc
dnq.0wp4aoe.cn/35977.Doc
dnq.0wp4aoe.cn/95555.Doc
dnq.0wp4aoe.cn/17993.Doc
dnq.0wp4aoe.cn/17117.Doc
dnq.0wp4aoe.cn/15715.Doc
dnq.0wp4aoe.cn/51133.Doc
dnq.0wp4aoe.cn/33119.Doc
dbm.0wp4aoe.cn/13173.Doc
dbm.0wp4aoe.cn/60664.Doc
dbm.0wp4aoe.cn/99199.Doc
dbm.0wp4aoe.cn/55939.Doc
dbm.0wp4aoe.cn/17933.Doc
dbm.0wp4aoe.cn/11317.Doc
dbm.0wp4aoe.cn/91579.Doc
dbm.0wp4aoe.cn/97713.Doc
dbm.0wp4aoe.cn/99791.Doc
dbm.0wp4aoe.cn/15155.Doc
dbn.0wp4aoe.cn/59713.Doc
dbn.0wp4aoe.cn/59577.Doc
dbn.0wp4aoe.cn/91553.Doc
dbn.0wp4aoe.cn/75311.Doc
dbn.0wp4aoe.cn/71513.Doc
dbn.0wp4aoe.cn/91371.Doc
dbn.0wp4aoe.cn/59775.Doc
dbn.0wp4aoe.cn/48208.Doc
dbn.0wp4aoe.cn/73331.Doc
dbn.0wp4aoe.cn/91797.Doc
dbb.0wp4aoe.cn/97319.Doc
dbb.0wp4aoe.cn/99771.Doc
dbb.0wp4aoe.cn/55337.Doc
dbb.0wp4aoe.cn/31955.Doc
dbb.0wp4aoe.cn/17339.Doc
dbb.0wp4aoe.cn/13973.Doc
dbb.0wp4aoe.cn/99977.Doc
dbb.0wp4aoe.cn/93759.Doc
dbb.0wp4aoe.cn/33555.Doc
dbb.0wp4aoe.cn/55397.Doc
dbv.0wp4aoe.cn/97737.Doc
dbv.0wp4aoe.cn/71337.Doc
dbv.0wp4aoe.cn/93959.Doc
dbv.0wp4aoe.cn/93571.Doc
dbv.0wp4aoe.cn/35379.Doc
dbv.0wp4aoe.cn/35197.Doc
dbv.0wp4aoe.cn/75533.Doc
dbv.0wp4aoe.cn/15353.Doc
dbv.0wp4aoe.cn/57937.Doc
dbv.0wp4aoe.cn/06806.Doc
dbc.0wp4aoe.cn/99511.Doc
dbc.0wp4aoe.cn/19131.Doc
dbc.0wp4aoe.cn/71971.Doc
dbc.0wp4aoe.cn/15757.Doc
dbc.0wp4aoe.cn/71711.Doc
dbc.0wp4aoe.cn/75173.Doc
dbc.0wp4aoe.cn/88224.Doc
dbc.0wp4aoe.cn/11999.Doc
dbc.0wp4aoe.cn/31197.Doc
dbc.0wp4aoe.cn/31157.Doc
dbx.0wp4aoe.cn/95111.Doc
dbx.0wp4aoe.cn/51139.Doc
dbx.0wp4aoe.cn/19179.Doc
dbx.0wp4aoe.cn/11791.Doc
dbx.0wp4aoe.cn/73573.Doc
dbx.0wp4aoe.cn/91393.Doc
dbx.0wp4aoe.cn/59713.Doc
dbx.0wp4aoe.cn/44608.Doc
dbx.0wp4aoe.cn/40288.Doc
dbx.0wp4aoe.cn/55995.Doc
dbz.0wp4aoe.cn/97175.Doc
dbz.0wp4aoe.cn/75115.Doc
dbz.0wp4aoe.cn/57919.Doc
dbz.0wp4aoe.cn/93573.Doc
dbz.0wp4aoe.cn/57551.Doc
dbz.0wp4aoe.cn/97335.Doc
dbz.0wp4aoe.cn/13791.Doc
dbz.0wp4aoe.cn/75731.Doc
dbz.0wp4aoe.cn/19779.Doc
dbz.0wp4aoe.cn/91793.Doc
dbl.0wp4aoe.cn/95735.Doc
dbl.0wp4aoe.cn/93379.Doc
dbl.0wp4aoe.cn/33735.Doc
dbl.0wp4aoe.cn/99559.Doc
dbl.0wp4aoe.cn/75719.Doc
dbl.0wp4aoe.cn/97715.Doc
dbl.0wp4aoe.cn/55179.Doc
dbl.0wp4aoe.cn/75355.Doc
dbl.0wp4aoe.cn/19731.Doc
dbl.0wp4aoe.cn/99719.Doc
dbk.0wp4aoe.cn/13197.Doc
dbk.0wp4aoe.cn/97719.Doc
dbk.0wp4aoe.cn/37937.Doc
dbk.0wp4aoe.cn/79955.Doc
dbk.0wp4aoe.cn/68806.Doc
dbk.0wp4aoe.cn/17997.Doc
dbk.0wp4aoe.cn/91915.Doc
dbk.0wp4aoe.cn/35339.Doc
dbk.0wp4aoe.cn/79375.Doc
dbk.0wp4aoe.cn/33759.Doc
dbj.0wp4aoe.cn/71175.Doc
dbj.0wp4aoe.cn/39799.Doc
dbj.0wp4aoe.cn/33999.Doc
dbj.0wp4aoe.cn/99315.Doc
dbj.0wp4aoe.cn/35731.Doc
dbj.0wp4aoe.cn/13551.Doc
dbj.0wp4aoe.cn/31737.Doc
dbj.0wp4aoe.cn/79395.Doc
dbj.0wp4aoe.cn/57339.Doc
dbj.0wp4aoe.cn/77377.Doc
dbh.0wp4aoe.cn/11333.Doc
dbh.0wp4aoe.cn/37713.Doc
dbh.0wp4aoe.cn/15333.Doc
dbh.0wp4aoe.cn/71335.Doc
dbh.0wp4aoe.cn/59517.Doc
dbh.0wp4aoe.cn/15139.Doc
dbh.0wp4aoe.cn/11759.Doc
dbh.0wp4aoe.cn/55355.Doc
dbh.0wp4aoe.cn/77393.Doc
dbh.0wp4aoe.cn/33713.Doc
dbg.0wp4aoe.cn/35151.Doc
dbg.0wp4aoe.cn/37197.Doc
dbg.0wp4aoe.cn/17591.Doc
dbg.0wp4aoe.cn/55137.Doc
dbg.0wp4aoe.cn/95731.Doc
dbg.0wp4aoe.cn/95391.Doc
dbg.0wp4aoe.cn/91557.Doc
dbg.0wp4aoe.cn/77159.Doc
dbg.0wp4aoe.cn/51579.Doc
dbg.0wp4aoe.cn/33759.Doc
dbf.0wp4aoe.cn/17775.Doc
dbf.0wp4aoe.cn/97555.Doc
dbf.0wp4aoe.cn/17737.Doc
dbf.0wp4aoe.cn/11591.Doc
dbf.0wp4aoe.cn/57197.Doc
dbf.0wp4aoe.cn/53593.Doc
dbf.0wp4aoe.cn/75735.Doc
dbf.0wp4aoe.cn/71333.Doc
dbf.0wp4aoe.cn/71311.Doc
dbf.0wp4aoe.cn/11935.Doc
dbd.0wp4aoe.cn/22040.Doc
dbd.0wp4aoe.cn/75791.Doc
dbd.0wp4aoe.cn/77719.Doc
dbd.0wp4aoe.cn/51797.Doc
dbd.0wp4aoe.cn/51157.Doc
dbd.0wp4aoe.cn/91113.Doc
dbd.0wp4aoe.cn/57533.Doc
dbd.0wp4aoe.cn/99919.Doc
dbd.0wp4aoe.cn/15177.Doc
dbd.0wp4aoe.cn/31959.Doc
dbs.0wp4aoe.cn/57555.Doc
dbs.0wp4aoe.cn/39751.Doc
dbs.0wp4aoe.cn/15131.Doc
dbs.0wp4aoe.cn/68682.Doc
dbs.0wp4aoe.cn/79555.Doc
dbs.0wp4aoe.cn/37315.Doc
dbs.0wp4aoe.cn/19513.Doc
dbs.0wp4aoe.cn/06404.Doc
dbs.0wp4aoe.cn/55951.Doc
dbs.0wp4aoe.cn/11315.Doc
dba.0wp4aoe.cn/55931.Doc
dba.0wp4aoe.cn/17795.Doc
dba.0wp4aoe.cn/31159.Doc
dba.0wp4aoe.cn/77593.Doc
dba.0wp4aoe.cn/59139.Doc
dba.0wp4aoe.cn/55951.Doc
dba.0wp4aoe.cn/57591.Doc
dba.0wp4aoe.cn/31197.Doc
dba.0wp4aoe.cn/02082.Doc
dba.0wp4aoe.cn/93315.Doc
