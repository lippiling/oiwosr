Python struct 二进制协议
===============================

struct 模块在 Python 值和 C 结构体之间打包/解包, 适合二进制协议和文件格式。

1. pack / unpack — 基本用法
-------------------------------

import struct

# pack: 将 Python 值打包为字节串
# 格式: "<H" 表示小端 2 字节无符号整数
packed = struct.pack("<H", 1024)
print("pack 结果:", packed)                   # b'\x00\x04'
print("长度:", len(packed))                   # 2

# unpack: 将字节串解包为 Python 值
value = struct.unpack("<H", packed)
print("unpack 结果:", value)                  # (1024,)

# 同时打包多个值
packed_multi = struct.pack("<I H B", 1, 2, 3)
print("多个值打包:", packed_multi.hex())       # 01000000 0200 03
# 解析: 4字节(I) + 2字节(H) + 1字节(B) = 7字节

# 解包多个值
a, b, c = struct.unpack("<I H B", packed_multi)
print(f"多值解包: I={a}, H={b}, B={c}")        # I=1, H=2, B=3

2. 格式代码详解
------------------

# 字节顺序前缀:
# @   native 字节序及对齐 (默认)
# =   native 标准大小
# <   小端序 (little-endian)
# >   大端序 (big-endian, 网络字节序)
# !   网络字节序 (同 >)

# 基本格式代码:
# x   填充字节 (无对应 Python 值)
# b   signed char (1字节) -> int
# B   unsigned char -> int
# h   short (2字节) -> int
# H   unsigned short -> int
# i   int (4字节) -> int
# I   unsigned int -> int
# l   long (4/8字节) -> int
# L   unsigned long -> int
# q   long long (8字节) -> int
# Q   unsigned long long -> int
# f   float (4字节) -> float
# d   double (8字节) -> float
# s   char[] -> bytes (需要指定长度, 如 10s)
# p   Pascal 字符串
# ?   _Bool -> bool

# 示例:
print("int 打包:", struct.pack(">i", -100))    # 大端 4 字节
print("float 打包:", struct.pack(">f", 3.14))  # 大端 4 字节浮点
print("double 打包:", struct.pack(">d", 3.14159265358979))
print("布尔打包:", struct.pack(">?", True))     # b'\x01'

# 字符串打包
name = b"Python"
# 4s 表示固定 4 字节字符串 (截断或补齐空格)
print("固定长度字符串:", struct.pack(">4s", name))    # b'Pyth'
print("完整字符串(带长度):", struct.pack(">6s", name)) # b'Python'

3. 字节序 — 小端与大端
--------------------------

# 小端序: 低地址存低位字节 (x86 架构)
# 大端序: 高地址存低位字节 (网络协议)

val = 0x12345678

# 小端: 78 56 34 12
little = struct.pack("<I", val)
print("小端:", little.hex(" "))                # 78 56 34 12

# 大端: 12 34 56 78
big = struct.pack(">I", val)
print("大端:", big.hex(" "))                   # 12 34 56 78

# 实战: 从网络数据包读取 2 字节大端端口号
raw_port = b"\x08\xAE"                         # 端口 2222
port = struct.unpack("!H", raw_port)[0]        # ! 和 > 等价
print("网络端口号:", port)                      # 2222

4. 位字段处理
----------------

# struct 不直接支持位字段, 但可通过掩码和移位手动处理
# 场景: 解析 IP 首部 (4位版本 + 4位首部长度)
def parse_ip_header_first_byte(data):
    """解析 IP 首部第一个字节: 版本(4bit) + 首部长度(4bit)"""
    raw = struct.unpack("!B", data)[0]
    version = (raw >> 4) & 0x0F               # 高4位: 版本号
    header_len = raw & 0x0F                    # 低4位: 首部长度(×4字节)
    return version, header_len * 4

ip_header_first = b"\x45"                      # IPv4, 首部长度20字节
ver, hdr_len = parse_ip_header_first_byte(ip_header_first)
print(f"IP版本={ver}, 首部长度={hdr_len}字节")  # IP版本=4, 首部长度=20字节

# 示例: 解析 2 字节标志位
flags_raw = struct.pack(">H", 0b1011000000000000)
flags = struct.unpack(">H", flags_raw)[0]
flag_a = (flags >> 15) & 1                    # 最高位
flag_b = (flags >> 14) & 1                    # 次高位
flag_c = (flags >> 13) & 1                    # 第三位
reserved = (flags >> 12) & 1                  # 第四位
print(f"标志位: A={flag_a}, B={flag_b}, C={flag_c}, R={reserved}")

5. 网络数据包打包/解包
--------------------------

import struct
from collections import namedtuple

# 自定义协议: 固定头部 + 可变负载
# 头部: 版本(1B) + 类型(1B) + 长度(2B, 大端) + 序列号(4B, 大端)
PacketHeader = namedtuple("PacketHeader", ["version", "type", "length", "seq"])

def pack_packet(version, ptype, seq, payload):
    """打包网络包"""
    length = len(payload)
    header = struct.pack("!BB HI", version, ptype, length, seq)
    return header + payload

def unpack_packet(data):
    """解包网络包"""
    # 头部固定 8 字节: B+B+pad(2)+H+I = 1+1+2+2+4 = 8? 不对
    # H 占 2 字节, I 占 4 字节, 但 B+B+2pad+ H 是 4 再 I 是 4, 共 8
    # 实际上格式 "!BB HI" 中的空格忽略: B(1)+B(1)+pad(2)+H(2)+I(4) = 10?
    # 用 calcsize 确认:
    hdr_size = struct.calcsize("!BB HI")
    print("包头部大小:", hdr_size)              # 8
    version, ptype, length, seq = struct.unpack("!BB HI", data[:hdr_size])
    payload = data[hdr_size:hdr_size + length]
    return PacketHeader(version, ptype, length, seq), payload

# 测试发包/解包
packet = pack_packet(1, 2, 100, b"Hello Protocol!")
header, payload = unpack_packet(packet)
print(f"版本={header.version}, 类型={header.type}, 长度={header.length}, 序列号={header.seq}")
print(f"负载内容: {payload}")

# 注意: struct 在打包 B B 后可能插入 2 字节填充以满足 H 的对齐要求
# 用 @ (native) 前缀时填充有平台相关性, ! 或 < 或 > 则不填充
print("无填充格式大小:", struct.calcsize("!BBHI"))  # 8

6. 读取/写入二进制文件格式
-----------------------------

import struct

# 场景: 读取 BMP 位图文件头
# BMP 文件头 (14字节):
#   bfType(2B) + bfSize(4B) + bfReserved1(2B) + bfReserved2(2B) + bfOffBits(4B)

bmp_header_format = "<H I H H I"               # 小端
bmp_header_size = struct.calcsize(bmp_header_format)
print("BMP 头部大小:", bmp_header_size)         # 14

# 模拟读取 BMP 头
def parse_bmp_header(data):
    """解析 BMP 文件头"""
    if len(data) < 14:
        raise ValueError("数据不足14字节")
    bf_type, bf_size, _, _, bf_off_bits = struct.unpack_from(bmp_header_format, data, 0)
    # bf_type 应为 0x4D42 (即 "BM")
    if bf_type != 0x4D42:
        raise ValueError(f"不是有效的 BMP 文件: 签名={bf_type:04X}")
    return {"大小": bf_size, "数据偏移": bf_off_bits}

# 构造模拟 BMP 数据
mock_bmp = struct.pack(bmp_header_format, 0x4D42, 1024, 0, 0, 54)
info = parse_bmp_header(mock_bmp)
print("BMP 文件信息:", info)                    # {'大小': 1024, '数据偏移': 54}

7. struct.calcsize — 计算大小与对齐
---------------------------------------

import struct

# calcsize 根据格式字符串计算打包后的字节大小
print("!i2f 大小:", struct.calcsize("!i2f"))    # 4+4+4=12
print("@i2f 大小(原生):", struct.calcsize("@i2f"))  # 可能不同 (平台相关)

# native 对齐规则:
#   char (B)    对齐 1 字节
#   short (H)   对齐 2 字节
#   int (I)     对齐 4 字节
#   long long   对齐 8 字节
print("@iB 大小:", struct.calcsize("@iB"))      # 8 (int 4 + char 1 + 填充3)
print("=iB 大小(标准):", struct.calcsize("=iB"))  # 5 (无填充)
print("!iB 大小:", struct.calcsize("!iB"))      # 5 (无填充)

# 使用 pack_into / unpack_from 避免创建中间 bytes 对象
import ctypes
buf = ctypes.create_string_buffer(20)           # 预分配缓冲区
struct.pack_into("!i H B", buf, 0, 100, 200, 50)  # 打包到指定偏移
i, h, b = struct.unpack_from("!i H B", buf, 0)
print(f"pack_into/unpack_from: i={i}, h={h}, b={b}")

# 性能对比: 频繁打包时应复用格式字符串
# 缓存格式字符串可避免重复编译
import time
fmt = "!I H B 10s"
compiled = struct.Struct(fmt)                  # 预编译格式
start = time.perf_counter()
for _ in range(10000):
    compiled.pack(1, 2, 3, b"test")            # 使用预编译对象
print("预编译 Struct 性能: 10000 次用时(ms)",
      (time.perf_counter() - start) * 1000)

# Struct 对象也支持 pack_into / unpack_from
buf = bytearray(compiled.size)
compiled.pack_into(buf, 0, 42, 7, 1, b"data")
values = compiled.unpack_from(buf, 0)
print("Struct 对象解析:", values)

总结: struct 是二进制协议开发的核心模块, 配合 BytesIO / memoryview 可实现高效数据处理.

wyf.7nvyou1.cn/46480.Doc
wyf.7nvyou1.cn/95131.Doc
wyf.7nvyou1.cn/66640.Doc
wyf.7nvyou1.cn/04066.Doc
wyf.7nvyou1.cn/88442.Doc
wyf.7nvyou1.cn/02480.Doc
wyf.7nvyou1.cn/42840.Doc
wyf.7nvyou1.cn/44404.Doc
wyf.7nvyou1.cn/44002.Doc
wyf.7nvyou1.cn/46808.Doc
wyd.7nvyou1.cn/86260.Doc
wyd.7nvyou1.cn/44688.Doc
wyd.7nvyou1.cn/88268.Doc
wyd.7nvyou1.cn/80820.Doc
wyd.7nvyou1.cn/48600.Doc
wyd.7nvyou1.cn/22628.Doc
wyd.7nvyou1.cn/68860.Doc
wyd.7nvyou1.cn/62642.Doc
wyd.7nvyou1.cn/82668.Doc
wyd.7nvyou1.cn/88220.Doc
wys.7nvyou1.cn/28246.Doc
wys.7nvyou1.cn/82628.Doc
wys.7nvyou1.cn/42664.Doc
wys.7nvyou1.cn/22028.Doc
wys.7nvyou1.cn/20848.Doc
wys.7nvyou1.cn/42602.Doc
wys.7nvyou1.cn/68600.Doc
wys.7nvyou1.cn/84606.Doc
wys.7nvyou1.cn/24804.Doc
wys.7nvyou1.cn/60822.Doc
wya.7nvyou1.cn/19397.Doc
wya.7nvyou1.cn/20686.Doc
wya.7nvyou1.cn/02288.Doc
wya.7nvyou1.cn/64024.Doc
wya.7nvyou1.cn/82662.Doc
wya.7nvyou1.cn/93757.Doc
wya.7nvyou1.cn/00242.Doc
wya.7nvyou1.cn/00206.Doc
wya.7nvyou1.cn/46200.Doc
wya.7nvyou1.cn/15551.Doc
wyp.7nvyou1.cn/82284.Doc
wyp.7nvyou1.cn/40424.Doc
wyp.7nvyou1.cn/40448.Doc
wyp.7nvyou1.cn/11193.Doc
wyp.7nvyou1.cn/48846.Doc
wyp.7nvyou1.cn/80642.Doc
wyp.7nvyou1.cn/02828.Doc
wyp.7nvyou1.cn/86224.Doc
wyp.7nvyou1.cn/04822.Doc
wyp.7nvyou1.cn/48060.Doc
wyo.7nvyou1.cn/66246.Doc
wyo.7nvyou1.cn/42888.Doc
wyo.7nvyou1.cn/33777.Doc
wyo.7nvyou1.cn/28664.Doc
wyo.7nvyou1.cn/80844.Doc
wyo.7nvyou1.cn/86860.Doc
wyo.7nvyou1.cn/86626.Doc
wyo.7nvyou1.cn/00620.Doc
wyo.7nvyou1.cn/82688.Doc
wyo.7nvyou1.cn/26264.Doc
wyi.7nvyou1.cn/82880.Doc
wyi.7nvyou1.cn/26086.Doc
wyi.7nvyou1.cn/04660.Doc
wyi.7nvyou1.cn/40600.Doc
wyi.7nvyou1.cn/48428.Doc
wyi.7nvyou1.cn/73757.Doc
wyi.7nvyou1.cn/06426.Doc
wyi.7nvyou1.cn/22462.Doc
wyi.7nvyou1.cn/44482.Doc
wyi.7nvyou1.cn/22208.Doc
wyu.7nvyou1.cn/66828.Doc
wyu.7nvyou1.cn/88244.Doc
wyu.7nvyou1.cn/44642.Doc
wyu.7nvyou1.cn/55391.Doc
wyu.7nvyou1.cn/19771.Doc
wyu.7nvyou1.cn/99395.Doc
wyu.7nvyou1.cn/22220.Doc
wyu.7nvyou1.cn/08004.Doc
wyu.7nvyou1.cn/62848.Doc
wyu.7nvyou1.cn/82608.Doc
wyy.7nvyou1.cn/66466.Doc
wyy.7nvyou1.cn/22844.Doc
wyy.7nvyou1.cn/44080.Doc
wyy.7nvyou1.cn/40262.Doc
wyy.7nvyou1.cn/44884.Doc
wyy.7nvyou1.cn/28268.Doc
wyy.7nvyou1.cn/44044.Doc
wyy.7nvyou1.cn/26446.Doc
wyy.7nvyou1.cn/82882.Doc
wyy.7nvyou1.cn/19931.Doc
wyt.7nvyou1.cn/73933.Doc
wyt.7nvyou1.cn/04868.Doc
wyt.7nvyou1.cn/24022.Doc
wyt.7nvyou1.cn/86286.Doc
wyt.7nvyou1.cn/68868.Doc
wyt.7nvyou1.cn/20626.Doc
wyt.7nvyou1.cn/66426.Doc
wyt.7nvyou1.cn/60646.Doc
wyt.7nvyou1.cn/00080.Doc
wyt.7nvyou1.cn/04840.Doc
wyr.7nvyou1.cn/26402.Doc
wyr.7nvyou1.cn/88608.Doc
wyr.7nvyou1.cn/88808.Doc
wyr.7nvyou1.cn/79757.Doc
wyr.7nvyou1.cn/62268.Doc
wyr.7nvyou1.cn/08406.Doc
wyr.7nvyou1.cn/48062.Doc
wyr.7nvyou1.cn/53737.Doc
wyr.7nvyou1.cn/26620.Doc
wyr.7nvyou1.cn/00620.Doc
wye.7nvyou1.cn/68420.Doc
wye.7nvyou1.cn/04828.Doc
wye.7nvyou1.cn/44004.Doc
wye.7nvyou1.cn/97771.Doc
wye.7nvyou1.cn/28406.Doc
wye.7nvyou1.cn/60802.Doc
wye.7nvyou1.cn/60088.Doc
wye.7nvyou1.cn/57197.Doc
wye.7nvyou1.cn/06200.Doc
wye.7nvyou1.cn/40200.Doc
wyw.7nvyou1.cn/62680.Doc
wyw.7nvyou1.cn/08866.Doc
wyw.7nvyou1.cn/00222.Doc
wyw.7nvyou1.cn/62266.Doc
wyw.7nvyou1.cn/60046.Doc
wyw.7nvyou1.cn/91175.Doc
wyw.7nvyou1.cn/64428.Doc
wyw.7nvyou1.cn/24202.Doc
wyw.7nvyou1.cn/82442.Doc
wyw.7nvyou1.cn/42262.Doc
wyq.7nvyou1.cn/28282.Doc
wyq.7nvyou1.cn/60888.Doc
wyq.7nvyou1.cn/86088.Doc
wyq.7nvyou1.cn/24868.Doc
wyq.7nvyou1.cn/64606.Doc
wyq.7nvyou1.cn/60062.Doc
wyq.7nvyou1.cn/02608.Doc
wyq.7nvyou1.cn/26024.Doc
wyq.7nvyou1.cn/26824.Doc
wyq.7nvyou1.cn/42664.Doc
wtm.7nvyou1.cn/22884.Doc
wtm.7nvyou1.cn/44622.Doc
wtm.7nvyou1.cn/93797.Doc
wtm.7nvyou1.cn/84802.Doc
wtm.7nvyou1.cn/84644.Doc
wtm.7nvyou1.cn/44808.Doc
wtm.7nvyou1.cn/64004.Doc
wtm.7nvyou1.cn/40248.Doc
wtm.7nvyou1.cn/64048.Doc
wtm.7nvyou1.cn/44268.Doc
wtn.7nvyou1.cn/48806.Doc
wtn.7nvyou1.cn/26844.Doc
wtn.7nvyou1.cn/24428.Doc
wtn.7nvyou1.cn/84422.Doc
wtn.7nvyou1.cn/88006.Doc
wtn.7nvyou1.cn/04444.Doc
wtn.7nvyou1.cn/26260.Doc
wtn.7nvyou1.cn/60286.Doc
wtn.7nvyou1.cn/71933.Doc
wtn.7nvyou1.cn/28440.Doc
wtb.7nvyou1.cn/75397.Doc
wtb.7nvyou1.cn/39377.Doc
wtb.7nvyou1.cn/88428.Doc
wtb.7nvyou1.cn/88682.Doc
wtb.7nvyou1.cn/46046.Doc
wtb.7nvyou1.cn/26260.Doc
wtb.7nvyou1.cn/79395.Doc
wtb.7nvyou1.cn/22022.Doc
wtb.7nvyou1.cn/39173.Doc
wtb.7nvyou1.cn/22844.Doc
wtv.7nvyou1.cn/82668.Doc
wtv.7nvyou1.cn/26284.Doc
wtv.7nvyou1.cn/02242.Doc
wtv.7nvyou1.cn/06208.Doc
wtv.7nvyou1.cn/40666.Doc
wtv.7nvyou1.cn/68022.Doc
wtv.7nvyou1.cn/15735.Doc
wtv.7nvyou1.cn/44426.Doc
wtv.7nvyou1.cn/64606.Doc
wtv.7nvyou1.cn/28664.Doc
wtc.7nvyou1.cn/42404.Doc
wtc.7nvyou1.cn/08686.Doc
wtc.7nvyou1.cn/60026.Doc
wtc.7nvyou1.cn/48622.Doc
wtc.7nvyou1.cn/22848.Doc
wtc.7nvyou1.cn/80244.Doc
wtc.7nvyou1.cn/62662.Doc
wtc.7nvyou1.cn/39713.Doc
wtc.7nvyou1.cn/11159.Doc
wtc.7nvyou1.cn/82808.Doc
wtx.7nvyou1.cn/08246.Doc
wtx.7nvyou1.cn/42080.Doc
wtx.7nvyou1.cn/60248.Doc
wtx.7nvyou1.cn/57179.Doc
wtx.7nvyou1.cn/64284.Doc
wtx.7nvyou1.cn/35157.Doc
wtx.7nvyou1.cn/80422.Doc
wtx.7nvyou1.cn/64642.Doc
wtx.7nvyou1.cn/19137.Doc
wtx.7nvyou1.cn/02882.Doc
wtz.7nvyou1.cn/80084.Doc
wtz.7nvyou1.cn/64426.Doc
wtz.7nvyou1.cn/40602.Doc
wtz.7nvyou1.cn/00284.Doc
wtz.7nvyou1.cn/46888.Doc
wtz.7nvyou1.cn/42466.Doc
wtz.7nvyou1.cn/00044.Doc
wtz.7nvyou1.cn/22686.Doc
wtz.7nvyou1.cn/42668.Doc
wtz.7nvyou1.cn/08282.Doc
wtl.7nvyou1.cn/93315.Doc
wtl.7nvyou1.cn/68086.Doc
wtl.7nvyou1.cn/02066.Doc
wtl.7nvyou1.cn/26486.Doc
wtl.7nvyou1.cn/80460.Doc
wtl.7nvyou1.cn/84646.Doc
wtl.7nvyou1.cn/82466.Doc
wtl.7nvyou1.cn/28062.Doc
wtl.7nvyou1.cn/66264.Doc
wtl.7nvyou1.cn/04222.Doc
wtk.7nvyou1.cn/64428.Doc
wtk.7nvyou1.cn/46868.Doc
wtk.7nvyou1.cn/62808.Doc
wtk.7nvyou1.cn/80200.Doc
wtk.7nvyou1.cn/02868.Doc
wtk.7nvyou1.cn/42428.Doc
wtk.7nvyou1.cn/44842.Doc
wtk.7nvyou1.cn/80246.Doc
wtk.7nvyou1.cn/64025.Doc
wtk.7nvyou1.cn/28822.Doc
wtj.7nvyou1.cn/68842.Doc
wtj.7nvyou1.cn/39971.Doc
wtj.7nvyou1.cn/64264.Doc
wtj.7nvyou1.cn/86622.Doc
wtj.7nvyou1.cn/68262.Doc
wtj.7nvyou1.cn/48684.Doc
wtj.7nvyou1.cn/04288.Doc
wtj.7nvyou1.cn/02844.Doc
wtj.7nvyou1.cn/26448.Doc
wtj.7nvyou1.cn/42208.Doc
wth.7nvyou1.cn/04888.Doc
wth.7nvyou1.cn/28400.Doc
wth.7nvyou1.cn/00066.Doc
wth.7nvyou1.cn/97977.Doc
wth.7nvyou1.cn/28208.Doc
wth.7nvyou1.cn/46002.Doc
wth.7nvyou1.cn/48686.Doc
wth.7nvyou1.cn/22060.Doc
wth.7nvyou1.cn/22662.Doc
wth.7nvyou1.cn/80444.Doc
wtg.7nvyou1.cn/33753.Doc
wtg.7nvyou1.cn/00484.Doc
wtg.7nvyou1.cn/46024.Doc
wtg.7nvyou1.cn/04244.Doc
wtg.7nvyou1.cn/77753.Doc
wtg.7nvyou1.cn/19759.Doc
wtg.7nvyou1.cn/88624.Doc
wtg.7nvyou1.cn/44002.Doc
wtg.7nvyou1.cn/88624.Doc
wtg.7nvyou1.cn/46620.Doc
wtf.7nvyou1.cn/64884.Doc
wtf.7nvyou1.cn/11537.Doc
wtf.7nvyou1.cn/62682.Doc
wtf.7nvyou1.cn/28204.Doc
wtf.7nvyou1.cn/62204.Doc
wtf.7nvyou1.cn/60806.Doc
wtf.7nvyou1.cn/20206.Doc
wtf.7nvyou1.cn/80048.Doc
wtf.7nvyou1.cn/80020.Doc
wtf.7nvyou1.cn/06864.Doc
wtd.7nvyou1.cn/79313.Doc
wtd.7nvyou1.cn/42604.Doc
wtd.7nvyou1.cn/04028.Doc
wtd.7nvyou1.cn/06028.Doc
wtd.7nvyou1.cn/77179.Doc
wtd.7nvyou1.cn/20422.Doc
wtd.7nvyou1.cn/57153.Doc
wtd.7nvyou1.cn/20880.Doc
wtd.7nvyou1.cn/20206.Doc
wtd.7nvyou1.cn/95571.Doc
wts.7nvyou1.cn/86466.Doc
wts.7nvyou1.cn/06620.Doc
wts.7nvyou1.cn/20202.Doc
wts.7nvyou1.cn/04644.Doc
wts.7nvyou1.cn/95737.Doc
wts.7nvyou1.cn/57933.Doc
wts.7nvyou1.cn/22220.Doc
wts.7nvyou1.cn/08404.Doc
wts.7nvyou1.cn/20246.Doc
wts.7nvyou1.cn/68660.Doc
wta.7nvyou1.cn/88860.Doc
wta.7nvyou1.cn/08028.Doc
wta.7nvyou1.cn/48004.Doc
wta.7nvyou1.cn/24840.Doc
wta.7nvyou1.cn/64422.Doc
wta.7nvyou1.cn/24224.Doc
wta.7nvyou1.cn/22464.Doc
wta.7nvyou1.cn/86028.Doc
wta.7nvyou1.cn/68680.Doc
wta.7nvyou1.cn/93573.Doc
wtp.7nvyou1.cn/39595.Doc
wtp.7nvyou1.cn/84660.Doc
wtp.7nvyou1.cn/95131.Doc
wtp.7nvyou1.cn/60802.Doc
wtp.7nvyou1.cn/24462.Doc
wtp.7nvyou1.cn/82228.Doc
wtp.7nvyou1.cn/08668.Doc
wtp.7nvyou1.cn/82420.Doc
wtp.7nvyou1.cn/08868.Doc
wtp.7nvyou1.cn/02286.Doc
wto.7nvyou1.cn/04040.Doc
wto.7nvyou1.cn/26606.Doc
wto.7nvyou1.cn/20888.Doc
wto.7nvyou1.cn/68426.Doc
wto.7nvyou1.cn/60688.Doc
wto.7nvyou1.cn/71537.Doc
wto.7nvyou1.cn/62422.Doc
wto.7nvyou1.cn/68264.Doc
wto.7nvyou1.cn/39115.Doc
wto.7nvyou1.cn/62802.Doc
wti.7nvyou1.cn/68420.Doc
wti.7nvyou1.cn/84066.Doc
wti.7nvyou1.cn/04600.Doc
wti.7nvyou1.cn/28802.Doc
wti.7nvyou1.cn/80620.Doc
wti.7nvyou1.cn/60262.Doc
wti.7nvyou1.cn/06060.Doc
wti.7nvyou1.cn/48028.Doc
wti.7nvyou1.cn/24848.Doc
wti.7nvyou1.cn/82446.Doc
wtu.7nvyou1.cn/60082.Doc
wtu.7nvyou1.cn/02262.Doc
wtu.7nvyou1.cn/02206.Doc
wtu.7nvyou1.cn/48040.Doc
wtu.7nvyou1.cn/42442.Doc
wtu.7nvyou1.cn/26824.Doc
wtu.7nvyou1.cn/86660.Doc
wtu.7nvyou1.cn/86246.Doc
wtu.7nvyou1.cn/86842.Doc
wtu.7nvyou1.cn/60480.Doc
wty.7nvyou1.cn/04066.Doc
wty.7nvyou1.cn/91531.Doc
wty.7nvyou1.cn/22222.Doc
wty.7nvyou1.cn/00264.Doc
wty.7nvyou1.cn/84680.Doc
wty.7nvyou1.cn/93333.Doc
wty.7nvyou1.cn/51115.Doc
wty.7nvyou1.cn/68486.Doc
wty.7nvyou1.cn/66800.Doc
wty.7nvyou1.cn/46206.Doc
wtt.7nvyou1.cn/02802.Doc
wtt.7nvyou1.cn/28648.Doc
wtt.7nvyou1.cn/06864.Doc
wtt.7nvyou1.cn/04204.Doc
wtt.7nvyou1.cn/02844.Doc
wtt.7nvyou1.cn/06062.Doc
wtt.7nvyou1.cn/22068.Doc
wtt.7nvyou1.cn/53191.Doc
wtt.7nvyou1.cn/95577.Doc
wtt.7nvyou1.cn/24008.Doc
wtr.7nvyou1.cn/66486.Doc
wtr.7nvyou1.cn/46228.Doc
wtr.7nvyou1.cn/33571.Doc
wtr.7nvyou1.cn/26804.Doc
wtr.7nvyou1.cn/00679.Doc
wtr.7nvyou1.cn/20040.Doc
wtr.7nvyou1.cn/91733.Doc
wtr.7nvyou1.cn/04820.Doc
wtr.7nvyou1.cn/40626.Doc
wtr.7nvyou1.cn/68466.Doc
wte.7nvyou1.cn/17557.Doc
wte.7nvyou1.cn/88086.Doc
wte.7nvyou1.cn/02888.Doc
wte.7nvyou1.cn/44646.Doc
wte.7nvyou1.cn/28424.Doc
wte.7nvyou1.cn/62028.Doc
wte.7nvyou1.cn/93355.Doc
wte.7nvyou1.cn/08824.Doc
wte.7nvyou1.cn/26040.Doc
wte.7nvyou1.cn/86840.Doc
wtw.7nvyou1.cn/26246.Doc
wtw.7nvyou1.cn/82424.Doc
wtw.7nvyou1.cn/39517.Doc
wtw.7nvyou1.cn/42884.Doc
wtw.7nvyou1.cn/42262.Doc
wtw.7nvyou1.cn/28468.Doc
wtw.7nvyou1.cn/28262.Doc
wtw.7nvyou1.cn/28206.Doc
wtw.7nvyou1.cn/35735.Doc
wtw.7nvyou1.cn/48060.Doc
wtq.7nvyou1.cn/88280.Doc
wtq.7nvyou1.cn/40408.Doc
wtq.7nvyou1.cn/00462.Doc
wtq.7nvyou1.cn/26224.Doc
wtq.7nvyou1.cn/59711.Doc
wtq.7nvyou1.cn/06648.Doc
wtq.7nvyou1.cn/48864.Doc
wtq.7nvyou1.cn/02866.Doc
wtq.7nvyou1.cn/20264.Doc
wtq.7nvyou1.cn/15557.Doc
wrm.7nvyou1.cn/91595.Doc
wrm.7nvyou1.cn/08020.Doc
wrm.7nvyou1.cn/88606.Doc
wrm.7nvyou1.cn/00848.Doc
wrm.7nvyou1.cn/42884.Doc
wrm.7nvyou1.cn/40448.Doc
wrm.7nvyou1.cn/02066.Doc
wrm.7nvyou1.cn/66082.Doc
wrm.7nvyou1.cn/26408.Doc
wrm.7nvyou1.cn/06822.Doc
wrn.7nvyou1.cn/00048.Doc
wrn.7nvyou1.cn/08064.Doc
wrn.7nvyou1.cn/53171.Doc
wrn.7nvyou1.cn/80620.Doc
wrn.7nvyou1.cn/22608.Doc
wrn.7nvyou1.cn/04268.Doc
wrn.7nvyou1.cn/06882.Doc
wrn.7nvyou1.cn/48408.Doc
wrn.7nvyou1.cn/22480.Doc
wrn.7nvyou1.cn/04688.Doc
wrb.7nvyou1.cn/46486.Doc
wrb.7nvyou1.cn/26084.Doc
wrb.7nvyou1.cn/20204.Doc
wrb.7nvyou1.cn/75777.Doc
wrb.7nvyou1.cn/20004.Doc
wrb.7nvyou1.cn/11951.Doc
wrb.7nvyou1.cn/02644.Doc
wrb.7nvyou1.cn/24242.Doc
wrb.7nvyou1.cn/48684.Doc
wrb.7nvyou1.cn/88880.Doc
wrv.7nvyou1.cn/62600.Doc
wrv.7nvyou1.cn/02022.Doc
wrv.7nvyou1.cn/51515.Doc
wrv.7nvyou1.cn/60266.Doc
wrv.7nvyou1.cn/64222.Doc
wrv.7nvyou1.cn/60282.Doc
wrv.7nvyou1.cn/86642.Doc
wrv.7nvyou1.cn/64464.Doc
wrv.7nvyou1.cn/46624.Doc
wrv.7nvyou1.cn/86802.Doc
wrc.7nvyou1.cn/68840.Doc
wrc.7nvyou1.cn/80608.Doc
wrc.7nvyou1.cn/06642.Doc
wrc.7nvyou1.cn/68046.Doc
wrc.7nvyou1.cn/28424.Doc
wrc.7nvyou1.cn/55177.Doc
wrc.7nvyou1.cn/44480.Doc
wrc.7nvyou1.cn/11173.Doc
wrc.7nvyou1.cn/11511.Doc
wrc.7nvyou1.cn/40084.Doc
wrx.7nvyou1.cn/86460.Doc
wrx.7nvyou1.cn/02662.Doc
wrx.7nvyou1.cn/20862.Doc
wrx.7nvyou1.cn/44628.Doc
wrx.7nvyou1.cn/24464.Doc
wrx.7nvyou1.cn/86480.Doc
wrx.7nvyou1.cn/20206.Doc
wrx.7nvyou1.cn/28480.Doc
wrx.7nvyou1.cn/86880.Doc
wrx.7nvyou1.cn/91951.Doc
wrz.7nvyou1.cn/44648.Doc
wrz.7nvyou1.cn/22866.Doc
wrz.7nvyou1.cn/66202.Doc
wrz.7nvyou1.cn/88842.Doc
wrz.7nvyou1.cn/08262.Doc
wrz.7nvyou1.cn/62642.Doc
wrz.7nvyou1.cn/44084.Doc
wrz.7nvyou1.cn/88426.Doc
wrz.7nvyou1.cn/60462.Doc
wrz.7nvyou1.cn/08800.Doc
wrl.7nvyou1.cn/28604.Doc
wrl.7nvyou1.cn/68824.Doc
wrl.7nvyou1.cn/06648.Doc
wrl.7nvyou1.cn/20242.Doc
wrl.7nvyou1.cn/60088.Doc
wrl.7nvyou1.cn/39175.Doc
wrl.7nvyou1.cn/71153.Doc
wrl.7nvyou1.cn/44800.Doc
wrl.7nvyou1.cn/00446.Doc
wrl.7nvyou1.cn/24440.Doc
wrk.7nvyou1.cn/20068.Doc
wrk.7nvyou1.cn/22640.Doc
wrk.7nvyou1.cn/66688.Doc
wrk.7nvyou1.cn/88442.Doc
wrk.7nvyou1.cn/68224.Doc
wrk.7nvyou1.cn/02088.Doc
wrk.7nvyou1.cn/66402.Doc
wrk.7nvyou1.cn/22800.Doc
wrk.7nvyou1.cn/80480.Doc
wrk.7nvyou1.cn/06444.Doc
