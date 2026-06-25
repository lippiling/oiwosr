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

wgn.irampL.cn/08204.Doc
wgn.irampL.cn/13957.Doc
wgn.irampL.cn/66842.Doc
wgn.irampL.cn/20284.Doc
wgn.irampL.cn/64840.Doc
wgn.irampL.cn/04082.Doc
wgn.irampL.cn/44820.Doc
wgn.irampL.cn/82866.Doc
wgn.irampL.cn/40640.Doc
wgn.irampL.cn/22606.Doc
wgb.irampL.cn/20468.Doc
wgb.irampL.cn/51595.Doc
wgb.irampL.cn/22620.Doc
wgb.irampL.cn/26446.Doc
wgb.irampL.cn/08608.Doc
wgb.irampL.cn/88644.Doc
wgb.irampL.cn/44064.Doc
wgb.irampL.cn/40062.Doc
wgb.irampL.cn/28048.Doc
wgb.irampL.cn/20282.Doc
wgv.irampL.cn/64486.Doc
wgv.irampL.cn/95371.Doc
wgv.irampL.cn/86400.Doc
wgv.irampL.cn/93195.Doc
wgv.irampL.cn/71333.Doc
wgv.irampL.cn/17379.Doc
wgv.irampL.cn/44248.Doc
wgv.irampL.cn/15999.Doc
wgv.irampL.cn/40220.Doc
wgv.irampL.cn/04042.Doc
wgc.irampL.cn/82082.Doc
wgc.irampL.cn/44026.Doc
wgc.irampL.cn/68264.Doc
wgc.irampL.cn/08266.Doc
wgc.irampL.cn/06080.Doc
wgc.irampL.cn/80248.Doc
wgc.irampL.cn/06824.Doc
wgc.irampL.cn/60644.Doc
wgc.irampL.cn/00020.Doc
wgc.irampL.cn/46222.Doc
wgx.irampL.cn/08004.Doc
wgx.irampL.cn/28802.Doc
wgx.irampL.cn/82000.Doc
wgx.irampL.cn/82228.Doc
wgx.irampL.cn/08204.Doc
wgx.irampL.cn/68208.Doc
wgx.irampL.cn/02406.Doc
wgx.irampL.cn/53391.Doc
wgx.irampL.cn/66840.Doc
wgx.irampL.cn/42644.Doc
wgz.irampL.cn/84600.Doc
wgz.irampL.cn/28668.Doc
wgz.irampL.cn/00006.Doc
wgz.irampL.cn/28246.Doc
wgz.irampL.cn/86868.Doc
wgz.irampL.cn/68282.Doc
wgz.irampL.cn/00644.Doc
wgz.irampL.cn/06408.Doc
wgz.irampL.cn/22260.Doc
wgz.irampL.cn/26424.Doc
wgl.irampL.cn/37975.Doc
wgl.irampL.cn/53157.Doc
wgl.irampL.cn/28880.Doc
wgl.irampL.cn/64846.Doc
wgl.irampL.cn/84282.Doc
wgl.irampL.cn/42642.Doc
wgl.irampL.cn/82880.Doc
wgl.irampL.cn/42280.Doc
wgl.irampL.cn/28880.Doc
wgl.irampL.cn/48844.Doc
wgk.irampL.cn/46204.Doc
wgk.irampL.cn/42864.Doc
wgk.irampL.cn/40080.Doc
wgk.irampL.cn/93157.Doc
wgk.irampL.cn/42820.Doc
wgk.irampL.cn/28046.Doc
wgk.irampL.cn/80624.Doc
wgk.irampL.cn/24462.Doc
wgk.irampL.cn/04640.Doc
wgk.irampL.cn/64244.Doc
wgj.irampL.cn/00202.Doc
wgj.irampL.cn/24022.Doc
wgj.irampL.cn/26224.Doc
wgj.irampL.cn/57939.Doc
wgj.irampL.cn/40600.Doc
wgj.irampL.cn/40042.Doc
wgj.irampL.cn/46846.Doc
wgj.irampL.cn/91979.Doc
wgj.irampL.cn/06062.Doc
wgj.irampL.cn/60480.Doc
wgh.irampL.cn/04848.Doc
wgh.irampL.cn/06464.Doc
wgh.irampL.cn/22466.Doc
wgh.irampL.cn/48424.Doc
wgh.irampL.cn/68642.Doc
wgh.irampL.cn/86668.Doc
wgh.irampL.cn/46204.Doc
wgh.irampL.cn/68884.Doc
wgh.irampL.cn/46686.Doc
wgh.irampL.cn/48860.Doc
wgg.irampL.cn/22248.Doc
wgg.irampL.cn/08024.Doc
wgg.irampL.cn/22880.Doc
wgg.irampL.cn/26404.Doc
wgg.irampL.cn/22084.Doc
wgg.irampL.cn/24884.Doc
wgg.irampL.cn/02648.Doc
wgg.irampL.cn/24848.Doc
wgg.irampL.cn/82048.Doc
wgg.irampL.cn/26464.Doc
wgf.irampL.cn/42248.Doc
wgf.irampL.cn/48882.Doc
wgf.irampL.cn/00662.Doc
wgf.irampL.cn/26044.Doc
wgf.irampL.cn/46228.Doc
wgf.irampL.cn/19511.Doc
wgf.irampL.cn/68204.Doc
wgf.irampL.cn/62828.Doc
wgf.irampL.cn/46280.Doc
wgf.irampL.cn/40484.Doc
wgd.irampL.cn/00220.Doc
wgd.irampL.cn/88886.Doc
wgd.irampL.cn/02042.Doc
wgd.irampL.cn/40680.Doc
wgd.irampL.cn/46064.Doc
wgd.irampL.cn/28086.Doc
wgd.irampL.cn/22486.Doc
wgd.irampL.cn/44260.Doc
wgd.irampL.cn/46604.Doc
wgd.irampL.cn/40660.Doc
wgs.irampL.cn/80204.Doc
wgs.irampL.cn/80202.Doc
wgs.irampL.cn/22280.Doc
wgs.irampL.cn/60862.Doc
wgs.irampL.cn/24680.Doc
wgs.irampL.cn/22220.Doc
wgs.irampL.cn/26826.Doc
wgs.irampL.cn/00628.Doc
wgs.irampL.cn/62066.Doc
wgs.irampL.cn/08204.Doc
wga.irampL.cn/53779.Doc
wga.irampL.cn/42226.Doc
wga.irampL.cn/00662.Doc
wga.irampL.cn/26286.Doc
wga.irampL.cn/08640.Doc
wga.irampL.cn/44406.Doc
wga.irampL.cn/80622.Doc
wga.irampL.cn/46086.Doc
wga.irampL.cn/24468.Doc
wga.irampL.cn/64240.Doc
wgp.irampL.cn/24420.Doc
wgp.irampL.cn/46486.Doc
wgp.irampL.cn/48422.Doc
wgp.irampL.cn/35359.Doc
wgp.irampL.cn/06480.Doc
wgp.irampL.cn/42444.Doc
wgp.irampL.cn/06242.Doc
wgp.irampL.cn/93713.Doc
wgp.irampL.cn/42242.Doc
wgp.irampL.cn/88204.Doc
wgo.irampL.cn/40080.Doc
wgo.irampL.cn/24642.Doc
wgo.irampL.cn/62620.Doc
wgo.irampL.cn/48884.Doc
wgo.irampL.cn/04686.Doc
wgo.irampL.cn/62820.Doc
wgo.irampL.cn/04426.Doc
wgo.irampL.cn/24268.Doc
wgo.irampL.cn/40680.Doc
wgo.irampL.cn/04208.Doc
wgi.irampL.cn/60062.Doc
wgi.irampL.cn/20044.Doc
wgi.irampL.cn/77391.Doc
wgi.irampL.cn/00886.Doc
wgi.irampL.cn/82802.Doc
wgi.irampL.cn/62860.Doc
wgi.irampL.cn/39197.Doc
wgi.irampL.cn/40048.Doc
wgi.irampL.cn/60626.Doc
wgi.irampL.cn/62222.Doc
wgu.irampL.cn/60228.Doc
wgu.irampL.cn/08868.Doc
wgu.irampL.cn/06466.Doc
wgu.irampL.cn/26028.Doc
wgu.irampL.cn/42248.Doc
wgu.irampL.cn/68444.Doc
wgu.irampL.cn/08042.Doc
wgu.irampL.cn/20828.Doc
wgu.irampL.cn/46004.Doc
wgu.irampL.cn/82820.Doc
wgy.irampL.cn/08804.Doc
wgy.irampL.cn/02200.Doc
wgy.irampL.cn/00688.Doc
wgy.irampL.cn/06846.Doc
wgy.irampL.cn/48020.Doc
wgy.irampL.cn/24822.Doc
wgy.irampL.cn/00664.Doc
wgy.irampL.cn/42020.Doc
wgy.irampL.cn/44844.Doc
wgy.irampL.cn/75513.Doc
wgt.irampL.cn/84048.Doc
wgt.irampL.cn/04824.Doc
wgt.irampL.cn/62886.Doc
wgt.irampL.cn/46484.Doc
wgt.irampL.cn/88486.Doc
wgt.irampL.cn/80642.Doc
wgt.irampL.cn/42208.Doc
wgt.irampL.cn/06688.Doc
wgt.irampL.cn/88206.Doc
wgt.irampL.cn/42022.Doc
wgr.irampL.cn/80064.Doc
wgr.irampL.cn/40602.Doc
wgr.irampL.cn/66866.Doc
wgr.irampL.cn/62664.Doc
wgr.irampL.cn/06406.Doc
wgr.irampL.cn/00824.Doc
wgr.irampL.cn/68006.Doc
wgr.irampL.cn/64442.Doc
wgr.irampL.cn/28406.Doc
wgr.irampL.cn/46886.Doc
wge.irampL.cn/00062.Doc
wge.irampL.cn/22444.Doc
wge.irampL.cn/46222.Doc
wge.irampL.cn/84606.Doc
wge.irampL.cn/88222.Doc
wge.irampL.cn/02606.Doc
wge.irampL.cn/46660.Doc
wge.irampL.cn/48606.Doc
wge.irampL.cn/86260.Doc
wge.irampL.cn/64488.Doc
wgw.irampL.cn/66466.Doc
wgw.irampL.cn/51575.Doc
wgw.irampL.cn/00082.Doc
wgw.irampL.cn/62242.Doc
wgw.irampL.cn/22626.Doc
wgw.irampL.cn/00060.Doc
wgw.irampL.cn/24628.Doc
wgw.irampL.cn/66882.Doc
wgw.irampL.cn/88882.Doc
wgw.irampL.cn/80860.Doc
wgq.irampL.cn/42420.Doc
wgq.irampL.cn/82620.Doc
wgq.irampL.cn/88068.Doc
wgq.irampL.cn/73379.Doc
wgq.irampL.cn/40268.Doc
wgq.irampL.cn/26046.Doc
wgq.irampL.cn/08664.Doc
wgq.irampL.cn/77395.Doc
wgq.irampL.cn/82862.Doc
wgq.irampL.cn/42226.Doc
wfm.irampL.cn/00240.Doc
wfm.irampL.cn/28402.Doc
wfm.irampL.cn/53953.Doc
wfm.irampL.cn/26244.Doc
wfm.irampL.cn/00000.Doc
wfm.irampL.cn/00088.Doc
wfm.irampL.cn/44466.Doc
wfm.irampL.cn/26060.Doc
wfm.irampL.cn/48648.Doc
wfm.irampL.cn/44002.Doc
wfn.irampL.cn/26844.Doc
wfn.irampL.cn/44404.Doc
wfn.irampL.cn/08808.Doc
wfn.irampL.cn/84624.Doc
wfn.irampL.cn/86820.Doc
wfn.irampL.cn/00008.Doc
wfn.irampL.cn/64026.Doc
wfn.irampL.cn/42866.Doc
wfn.irampL.cn/68688.Doc
wfn.irampL.cn/37733.Doc
wfb.irampL.cn/02042.Doc
wfb.irampL.cn/68600.Doc
wfb.irampL.cn/82022.Doc
wfb.irampL.cn/22622.Doc
wfb.irampL.cn/86060.Doc
wfb.irampL.cn/44884.Doc
wfb.irampL.cn/53599.Doc
wfb.irampL.cn/24822.Doc
wfb.irampL.cn/80244.Doc
wfb.irampL.cn/35593.Doc
wfv.irampL.cn/84228.Doc
wfv.irampL.cn/68048.Doc
wfv.irampL.cn/84040.Doc
wfv.irampL.cn/88428.Doc
wfv.irampL.cn/04048.Doc
wfv.irampL.cn/48864.Doc
wfv.irampL.cn/04846.Doc
wfv.irampL.cn/82622.Doc
wfv.irampL.cn/60408.Doc
wfv.irampL.cn/66866.Doc
wfc.irampL.cn/24828.Doc
wfc.irampL.cn/93771.Doc
wfc.irampL.cn/66822.Doc
wfc.irampL.cn/59599.Doc
wfc.irampL.cn/20886.Doc
wfc.irampL.cn/86244.Doc
wfc.irampL.cn/62882.Doc
wfc.irampL.cn/42022.Doc
wfc.irampL.cn/46660.Doc
wfc.irampL.cn/42026.Doc
wfx.irampL.cn/88202.Doc
wfx.irampL.cn/20424.Doc
wfx.irampL.cn/55737.Doc
wfx.irampL.cn/64484.Doc
wfx.irampL.cn/13733.Doc
wfx.irampL.cn/06008.Doc
wfx.irampL.cn/26682.Doc
wfx.irampL.cn/08626.Doc
wfx.irampL.cn/26404.Doc
wfx.irampL.cn/24888.Doc
wfz.irampL.cn/44866.Doc
wfz.irampL.cn/24420.Doc
wfz.irampL.cn/91931.Doc
wfz.irampL.cn/64408.Doc
wfz.irampL.cn/20064.Doc
wfz.irampL.cn/22842.Doc
wfz.irampL.cn/68646.Doc
wfz.irampL.cn/22640.Doc
wfz.irampL.cn/64668.Doc
wfz.irampL.cn/46224.Doc
wfl.irampL.cn/48440.Doc
wfl.irampL.cn/44840.Doc
wfl.irampL.cn/62662.Doc
wfl.irampL.cn/68422.Doc
wfl.irampL.cn/24424.Doc
wfl.irampL.cn/66020.Doc
wfl.irampL.cn/22242.Doc
wfl.irampL.cn/62422.Doc
wfl.irampL.cn/99991.Doc
wfl.irampL.cn/11535.Doc
wfk.irampL.cn/60260.Doc
wfk.irampL.cn/35533.Doc
wfk.irampL.cn/28662.Doc
wfk.irampL.cn/82404.Doc
wfk.irampL.cn/62068.Doc
wfk.irampL.cn/26200.Doc
wfk.irampL.cn/53317.Doc
wfk.irampL.cn/82802.Doc
wfk.irampL.cn/60408.Doc
wfk.irampL.cn/33559.Doc
wfj.irampL.cn/08488.Doc
wfj.irampL.cn/64600.Doc
wfj.irampL.cn/73395.Doc
wfj.irampL.cn/04002.Doc
wfj.irampL.cn/86242.Doc
wfj.irampL.cn/00268.Doc
wfj.irampL.cn/20248.Doc
wfj.irampL.cn/46880.Doc
wfj.irampL.cn/97355.Doc
wfj.irampL.cn/28248.Doc
wfh.irampL.cn/82886.Doc
wfh.irampL.cn/04860.Doc
wfh.irampL.cn/57559.Doc
wfh.irampL.cn/88028.Doc
wfh.irampL.cn/46046.Doc
wfh.irampL.cn/84626.Doc
wfh.irampL.cn/28026.Doc
wfh.irampL.cn/13593.Doc
wfh.irampL.cn/39393.Doc
wfh.irampL.cn/28868.Doc
wfg.irampL.cn/59593.Doc
wfg.irampL.cn/68600.Doc
wfg.irampL.cn/06482.Doc
wfg.irampL.cn/26686.Doc
wfg.irampL.cn/84806.Doc
wfg.irampL.cn/59735.Doc
wfg.irampL.cn/40820.Doc
wfg.irampL.cn/88220.Doc
wfg.irampL.cn/39395.Doc
wfg.irampL.cn/37933.Doc
wff.irampL.cn/48884.Doc
wff.irampL.cn/91915.Doc
wff.irampL.cn/66268.Doc
wff.irampL.cn/97599.Doc
wff.irampL.cn/60624.Doc
wff.irampL.cn/42026.Doc
wff.irampL.cn/60288.Doc
wff.irampL.cn/88482.Doc
wff.irampL.cn/48620.Doc
wff.irampL.cn/88280.Doc
wfd.irampL.cn/59373.Doc
wfd.irampL.cn/80028.Doc
wfd.irampL.cn/82662.Doc
wfd.irampL.cn/42482.Doc
wfd.irampL.cn/44646.Doc
wfd.irampL.cn/04624.Doc
wfd.irampL.cn/19711.Doc
wfd.irampL.cn/42680.Doc
wfd.irampL.cn/88840.Doc
wfd.irampL.cn/88086.Doc
wfs.irampL.cn/51751.Doc
wfs.irampL.cn/62404.Doc
wfs.irampL.cn/66086.Doc
wfs.irampL.cn/20644.Doc
wfs.irampL.cn/97135.Doc
wfs.irampL.cn/26866.Doc
wfs.irampL.cn/48862.Doc
wfs.irampL.cn/62224.Doc
wfs.irampL.cn/28628.Doc
wfs.irampL.cn/48608.Doc
wfa.irampL.cn/44024.Doc
wfa.irampL.cn/48248.Doc
wfa.irampL.cn/20646.Doc
wfa.irampL.cn/33317.Doc
wfa.irampL.cn/77111.Doc
wfa.irampL.cn/95795.Doc
wfa.irampL.cn/26468.Doc
wfa.irampL.cn/06826.Doc
wfa.irampL.cn/48646.Doc
wfa.irampL.cn/28462.Doc
wfp.irampL.cn/80444.Doc
wfp.irampL.cn/64420.Doc
wfp.irampL.cn/08262.Doc
wfp.irampL.cn/42604.Doc
wfp.irampL.cn/46664.Doc
wfp.irampL.cn/00408.Doc
wfp.irampL.cn/00422.Doc
wfp.irampL.cn/39339.Doc
wfp.irampL.cn/60228.Doc
wfp.irampL.cn/68048.Doc
wfo.irampL.cn/00048.Doc
wfo.irampL.cn/28426.Doc
wfo.irampL.cn/62686.Doc
wfo.irampL.cn/35115.Doc
wfo.irampL.cn/20428.Doc
wfo.irampL.cn/22640.Doc
wfo.irampL.cn/08662.Doc
wfo.irampL.cn/82202.Doc
wfo.irampL.cn/02426.Doc
wfo.irampL.cn/64644.Doc
wfi.irampL.cn/44448.Doc
wfi.irampL.cn/80866.Doc
wfi.irampL.cn/37373.Doc
wfi.irampL.cn/42404.Doc
wfi.irampL.cn/62866.Doc
wfi.irampL.cn/40668.Doc
wfi.irampL.cn/68880.Doc
wfi.irampL.cn/24246.Doc
wfi.irampL.cn/64880.Doc
wfi.irampL.cn/60068.Doc
wfu.irampL.cn/82426.Doc
wfu.irampL.cn/73111.Doc
wfu.irampL.cn/44466.Doc
wfu.irampL.cn/22424.Doc
wfu.irampL.cn/62260.Doc
wfu.irampL.cn/62084.Doc
wfu.irampL.cn/20246.Doc
wfu.irampL.cn/24684.Doc
wfu.irampL.cn/64246.Doc
wfu.irampL.cn/00886.Doc
wfy.irampL.cn/39351.Doc
wfy.irampL.cn/68042.Doc
wfy.irampL.cn/82608.Doc
wfy.irampL.cn/28266.Doc
wfy.irampL.cn/64626.Doc
wfy.irampL.cn/20648.Doc
wfy.irampL.cn/44220.Doc
wfy.irampL.cn/26664.Doc
wfy.irampL.cn/20462.Doc
wfy.irampL.cn/19971.Doc
wft.irampL.cn/22040.Doc
wft.irampL.cn/44240.Doc
wft.irampL.cn/97757.Doc
wft.irampL.cn/22880.Doc
wft.irampL.cn/68642.Doc
wft.irampL.cn/20482.Doc
wft.irampL.cn/28244.Doc
wft.irampL.cn/53319.Doc
wft.irampL.cn/86600.Doc
wft.irampL.cn/68464.Doc
wfr.irampL.cn/82828.Doc
wfr.irampL.cn/26260.Doc
wfr.irampL.cn/35993.Doc
wfr.irampL.cn/64666.Doc
wfr.irampL.cn/22202.Doc
wfr.irampL.cn/08886.Doc
wfr.irampL.cn/22286.Doc
wfr.irampL.cn/26040.Doc
wfr.irampL.cn/22804.Doc
wfr.irampL.cn/20628.Doc
wfe.irampL.cn/80002.Doc
wfe.irampL.cn/68864.Doc
wfe.irampL.cn/60204.Doc
wfe.irampL.cn/80204.Doc
wfe.irampL.cn/00466.Doc
wfe.irampL.cn/91757.Doc
wfe.irampL.cn/84202.Doc
wfe.irampL.cn/42802.Doc
wfe.irampL.cn/64046.Doc
wfe.irampL.cn/46280.Doc
