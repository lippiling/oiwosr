欣职敢僚邓


Python字节数组与内存视图
==============================

一、bytearray——可变的字节序列
bytearray是bytes的可变版本，支持就地修改操作。
与bytes不同，bytearray的元素可以直接赋值修改。

# 创建bytearray的多种方式
空数组 = bytearray()           # 空
指定长度 = bytearray(10)       # 10个零字节
从列表 = bytearray([65, 66, 67])  # 从整数列表
从字符串 = bytearray("Hello", "utf-8")  # 从字符串编码
从bytes = bytearray(b"World")  # 从bytes对象

print(f"从列表创建: {从列表}")  # bytearray(b'ABC')

# bytearray支持就地修改
可变数据 = bytearray(b"Python")
可变数据[0] = 74  # 修改第一个字节为 'J'
print(f"修改后: {可变数据}")  # bytearray(b'Jython')

# 支持切片赋值
可变数据[1:4] = b"AV"
print(f"切片赋值后: {可变数据}")  # bytearray(b'JAvon')

# append和extend操作
可变数据.append(33)  # 添加 '!'
print(f"append后: {可变数据}")

二、memoryview——零拷贝内存视图
memoryview以零拷贝的方式共享内存数据，避免大数据的复制开销。

import struct

# 从bytearray创建memoryview
原始数据 = bytearray(b"\x01\x02\x03\x04\x05\x06\x07\x08")
视图 = memoryview(原始数据)
print(f"视图: {视图}")
print(f"视图长度: {视图.nbytes}")
print(f"视图格式: {视图.format}")  # 默认'B'（无符号字节）

# 通过视图修改原始数据
视图[0] = 0xFF
print(f"原始数据被修改: {原始数据}")  # bytearray(b'\xff\x02...')

三、memoryview的格式转换
memoryview支持cast方法，将数据解释为不同的格式。

# 将字节数据转换为不同格式查看
数据 = bytearray(b"\x01\x00\x00\x00\x02\x00\x00\x00")
视图 = memoryview(数据)

# 转换为无符号16位整数数组
十六位视图 = 视图.cast("H")  # 'H' = unsigned short (2字节)
print(f"16位视图: {list(十六位视图)}")
# 小端序输出: [1, 2]

# 转换为32位整数
三十二位视图 = 视图.cast("I")  # 'I' = unsigned int (4字节)
print(f"32位视图: {list(三十二位视图)}")
# 小端序输出: [1, 2]

# 注意：cast要求总字节数能被目标格式大小整除
浮点数据 = bytearray(b"\x00\x00\x00\x00\x00\x00\xF0\x3F")
浮点视图 = memoryview(浮点数据).cast("d")  # 'd' = double (8字节)
print(f"浮点值: {浮点视图[0]:.1f}")  # 输出: 1.0

四、使用struct模块通过memoryview访问

包数据 = bytearray(12)  # 3个32位整数
视图mv = memoryview(包数据)

# 使用struct.pack_into将数据打包到指定位置
struct.pack_into("<I", 包数据, 0, 100)   # 偏移0：写入100
struct.pack_into("<I", 包数据, 4, 200)   # 偏移4：写入200
struct.pack_into("<I", 包数据, 8, 300)   # 偏移8：写入300

# 使用struct.unpack_from从指定位置读取
值1 = struct.unpack_from("<I", 包数据, 0)[0]
值2 = struct.unpack_from("<I", 包数据, 4)[0]
值3 = struct.unpack_from("<I", 包数据, 8)[0]
print(f"解包值: {值1}, {值2}, {值3}")

五、bytes vs bytearray vs memoryview对比

import time

# bytes：不可变字节序列，哈希可用
不可变 = b"hello"
哈希值 = hash(不可变)  # bytes可哈希

# bytearray：可变字节序列，不可哈希
可变 = bytearray(b"hello")
# hash(可变)  # 会抛出TypeError

# memoryview：零拷贝视图，适合大块数据处理
大数据 = bytearray(100 * 1024 * 1024)  # 100MB
开始 = time.perf_counter()
视图副本 = memoryview(大数据)
用时 = time.perf_counter() - 开始
print(f"memoryview创建用时: {用时:.6f}秒")

# 对比切片复制
开始 = time.perf_counter()
切片副本 = 大数据[:1024]
用时 = time.perf_counter() - 开始
print(f"bytes切片复制用时: {用时:.6f}秒")

# memoryview切片是零拷贝
小视图 = 视图副本[:1024]
print(f"memoryview切片类型: {type(小视图)}")

六、实际应用示例：解析二进制文件头

def 解析PNG头部(数据: bytes) -> dict:
    """解析PNG文件头部信息"""
    mv = memoryview(数据)
    签名 = bytes(mv[:8])
    if 签名 != b"\x89PNG\r\n\x1a\n":
        raise ValueError("不是有效的PNG文件")
    # 读取IHDR块中的宽度和高度
    宽度 = struct.unpack_from(">I", 数据, 16)[0]
    高度 = struct.unpack_from(">I", 数据, 20)[0]
    return {"宽度": 宽度, "高度": 高度}

模拟PNG = (b"\x89PNG\r\n\x1a\n" + b"\x00" * 8 +
           b"\x00\x00\x00\x10\x00\x00\x00\x20")
信息 = 解析PNG头部(模拟PNG)
print(f"PNG尺寸: {信息}")

七、小结
bytearray适合需要原地修改字节数据的场景；memoryview是处理
大数据块时的性能利器，零拷贝特性能显著减少内存开销。三者
各有适用场景，合理选用能极大提升数据处理效率。

扔沃谴吧懦门谋季仁叭虏娇涎钠履

qds.ugioc26.cn/559575.htm
qds.ugioc26.cn/557735.htm
qds.ugioc26.cn/951955.htm
qds.ugioc26.cn/557935.htm
qds.ugioc26.cn/717375.htm
qds.ugioc26.cn/393355.htm
qds.ugioc26.cn/799175.htm
qds.ugioc26.cn/137915.htm
qda.ugioc26.cn/533535.htm
qda.ugioc26.cn/511195.htm
qda.ugioc26.cn/773735.htm
qda.ugioc26.cn/975995.htm
qda.ugioc26.cn/157375.htm
qda.ugioc26.cn/713775.htm
qda.ugioc26.cn/519555.htm
qda.ugioc26.cn/177195.htm
qda.ugioc26.cn/539995.htm
qda.ugioc26.cn/595795.htm
qdp.ugioc26.cn/311975.htm
qdp.ugioc26.cn/175115.htm
qdp.ugioc26.cn/975375.htm
qdp.ugioc26.cn/195755.htm
qdp.ugioc26.cn/757115.htm
qdp.ugioc26.cn/711995.htm
qdp.ugioc26.cn/339395.htm
qdp.ugioc26.cn/919375.htm
qdp.ugioc26.cn/331935.htm
qdp.ugioc26.cn/159735.htm
qdo.ugioc26.cn/751175.htm
qdo.ugioc26.cn/355395.htm
qdo.ugioc26.cn/177915.htm
qdo.ugioc26.cn/717555.htm
qdo.ugioc26.cn/597535.htm
qdo.ugioc26.cn/319335.htm
qdo.ugioc26.cn/571955.htm
qdo.ugioc26.cn/953955.htm
qdo.ugioc26.cn/313335.htm
qdo.ugioc26.cn/733935.htm
qdi.ugioc26.cn/399995.htm
qdi.ugioc26.cn/517915.htm
qdi.ugioc26.cn/331155.htm
qdi.ugioc26.cn/131335.htm
qdi.ugioc26.cn/719335.htm
qdi.ugioc26.cn/179735.htm
qdi.ugioc26.cn/317315.htm
qdi.ugioc26.cn/533355.htm
qdi.ugioc26.cn/513155.htm
qdi.ugioc26.cn/535195.htm
qdu.ugioc26.cn/979155.htm
qdu.ugioc26.cn/771355.htm
qdu.ugioc26.cn/797735.htm
qdu.ugioc26.cn/157375.htm
qdu.ugioc26.cn/797575.htm
qdu.ugioc26.cn/535175.htm
qdu.ugioc26.cn/593775.htm
qdu.ugioc26.cn/915195.htm
qdu.ugioc26.cn/331715.htm
qdu.ugioc26.cn/579115.htm
qdy.ugioc26.cn/157595.htm
qdy.ugioc26.cn/799115.htm
qdy.ugioc26.cn/139535.htm
qdy.ugioc26.cn/135135.htm
qdy.ugioc26.cn/173135.htm
qdy.ugioc26.cn/731935.htm
qdy.ugioc26.cn/735355.htm
qdy.ugioc26.cn/537955.htm
qdy.ugioc26.cn/137355.htm
qdy.ugioc26.cn/159355.htm
qdt.ugioc26.cn/537195.htm
qdt.ugioc26.cn/971175.htm
qdt.ugioc26.cn/531755.htm
qdt.ugioc26.cn/951935.htm
qdt.ugioc26.cn/155315.htm
qdt.ugioc26.cn/935135.htm
qdt.ugioc26.cn/511935.htm
qdt.ugioc26.cn/533315.htm
qdt.ugioc26.cn/939535.htm
qdt.ugioc26.cn/515115.htm
qdr.ugioc26.cn/751955.htm
qdr.ugioc26.cn/973955.htm
qdr.ugioc26.cn/339395.htm
qdr.ugioc26.cn/777395.htm
qdr.ugioc26.cn/953195.htm
qdr.ugioc26.cn/995515.htm
qdr.ugioc26.cn/593735.htm
qdr.ugioc26.cn/979175.htm
qdr.ugioc26.cn/395995.htm
qdr.ugioc26.cn/175395.htm
qde.ugioc26.cn/719715.htm
qde.ugioc26.cn/997575.htm
qde.ugioc26.cn/575155.htm
qde.ugioc26.cn/759515.htm
qde.ugioc26.cn/395115.htm
qde.ugioc26.cn/333315.htm
qde.ugioc26.cn/557735.htm
qde.ugioc26.cn/751555.htm
qde.ugioc26.cn/313935.htm
qde.ugioc26.cn/397195.htm
qdw.ugioc26.cn/193515.htm
qdw.ugioc26.cn/979535.htm
qdw.ugioc26.cn/191935.htm
qdw.ugioc26.cn/355395.htm
qdw.ugioc26.cn/913755.htm
qdw.ugioc26.cn/139555.htm
qdw.ugioc26.cn/771155.htm
qdw.ugioc26.cn/917355.htm
qdw.ugioc26.cn/953735.htm
qdw.ugioc26.cn/737195.htm
qdq.ugioc26.cn/533395.htm
qdq.ugioc26.cn/317915.htm
qdq.ugioc26.cn/937735.htm
qdq.ugioc26.cn/933355.htm
qdq.ugioc26.cn/179795.htm
qdq.ugioc26.cn/151975.htm
qdq.ugioc26.cn/995575.htm
qdq.ugioc26.cn/991755.htm
qdq.ugioc26.cn/597715.htm
qdq.ugioc26.cn/939155.htm
qstv.ugioc26.cn/313175.htm
qstv.ugioc26.cn/771575.htm
qstv.ugioc26.cn/751715.htm
qstv.ugioc26.cn/331775.htm
qstv.ugioc26.cn/579775.htm
qstv.ugioc26.cn/793795.htm
qstv.ugioc26.cn/191175.htm
qstv.ugioc26.cn/119135.htm
qstv.ugioc26.cn/573975.htm
qstv.ugioc26.cn/111755.htm
qsn.ugioc26.cn/317195.htm
qsn.ugioc26.cn/753935.htm
qsn.ugioc26.cn/757575.htm
qsn.ugioc26.cn/991755.htm
qsn.ugioc26.cn/391315.htm
qsn.ugioc26.cn/155955.htm
qsn.ugioc26.cn/973155.htm
qsn.ugioc26.cn/513135.htm
qsn.ugioc26.cn/733755.htm
qsn.ugioc26.cn/579155.htm
qsb.ugioc26.cn/593315.htm
qsb.ugioc26.cn/959915.htm
qsb.ugioc26.cn/193955.htm
qsb.ugioc26.cn/333335.htm
qsb.ugioc26.cn/317515.htm
qsb.ugioc26.cn/597735.htm
qsb.ugioc26.cn/775355.htm
qsb.ugioc26.cn/755975.htm
qsb.ugioc26.cn/917355.htm
qsb.ugioc26.cn/373155.htm
qsv.ugioc26.cn/139355.htm
qsv.ugioc26.cn/517175.htm
qsv.ugioc26.cn/997355.htm
qsv.ugioc26.cn/793115.htm
qsv.ugioc26.cn/999755.htm
qsv.ugioc26.cn/191375.htm
qsv.ugioc26.cn/731915.htm
qsv.ugioc26.cn/517175.htm
qsv.ugioc26.cn/571195.htm
qsv.ugioc26.cn/777535.htm
qsc.ugioc26.cn/939935.htm
qsc.ugioc26.cn/953735.htm
qsc.ugioc26.cn/117775.htm
qsc.ugioc26.cn/737995.htm
qsc.ugioc26.cn/757595.htm
qsc.ugioc26.cn/397575.htm
qsc.ugioc26.cn/513555.htm
qsc.ugioc26.cn/715955.htm
qsc.ugioc26.cn/199915.htm
qsc.ugioc26.cn/751735.htm
qsx.ugioc26.cn/771795.htm
qsx.ugioc26.cn/777315.htm
qsx.ugioc26.cn/993735.htm
qsx.ugioc26.cn/315535.htm
qsx.ugioc26.cn/511595.htm
qsx.ugioc26.cn/351575.htm
qsx.ugioc26.cn/111355.htm
qsx.ugioc26.cn/35.htm
qsx.ugioc26.cn/331595.htm
qsx.ugioc26.cn/113715.htm
qsz.ugioc26.cn/359975.htm
qsz.ugioc26.cn/975795.htm
qsz.ugioc26.cn/911135.htm
qsz.ugioc26.cn/131935.htm
qsz.ugioc26.cn/319915.htm
qsz.ugioc26.cn/115115.htm
qsz.ugioc26.cn/179995.htm
qsz.ugioc26.cn/571555.htm
qsz.ugioc26.cn/135115.htm
qsz.ugioc26.cn/377975.htm
qsl.ugioc26.cn/515755.htm
qsl.ugioc26.cn/119335.htm
qsl.ugioc26.cn/771555.htm
qsl.ugioc26.cn/393775.htm
qsl.ugioc26.cn/391975.htm
qsl.ugioc26.cn/531375.htm
qsl.ugioc26.cn/993775.htm
qsl.ugioc26.cn/139975.htm
qsl.ugioc26.cn/591175.htm
qsl.ugioc26.cn/195915.htm
qsk.ugioc26.cn/995795.htm
qsk.ugioc26.cn/551775.htm
qsk.ugioc26.cn/175555.htm
qsk.ugioc26.cn/919715.htm
qsk.ugioc26.cn/777935.htm
qsk.ugioc26.cn/759715.htm
qsk.ugioc26.cn/773995.htm
qsk.ugioc26.cn/539775.htm
qsk.ugioc26.cn/319375.htm
qsk.ugioc26.cn/575795.htm
qsj.ugioc26.cn/935575.htm
qsj.ugioc26.cn/953575.htm
qsj.ugioc26.cn/919195.htm
qsj.ugioc26.cn/197155.htm
qsj.ugioc26.cn/737155.htm
qsj.ugioc26.cn/713555.htm
qsj.ugioc26.cn/577335.htm
qsj.ugioc26.cn/713575.htm
qsj.ugioc26.cn/35.htm
qsj.ugioc26.cn/197355.htm
qsh.ugioc26.cn/179395.htm
qsh.ugioc26.cn/113915.htm
qsh.ugioc26.cn/573515.htm
qsh.ugioc26.cn/359195.htm
qsh.ugioc26.cn/553735.htm
qsh.ugioc26.cn/919195.htm
qsh.ugioc26.cn/599195.htm
qsh.ugioc26.cn/157975.htm
qsh.ugioc26.cn/979955.htm
qsh.ugioc26.cn/157155.htm
qsg.ugioc26.cn/317755.htm
qsg.ugioc26.cn/115115.htm
qsg.ugioc26.cn/713555.htm
qsg.ugioc26.cn/973335.htm
qsg.ugioc26.cn/791315.htm
qsg.ugioc26.cn/573575.htm
qsg.ugioc26.cn/193935.htm
qsg.ugioc26.cn/591955.htm
qsg.ugioc26.cn/191355.htm
qsg.ugioc26.cn/575555.htm
qsf.ugioc26.cn/519935.htm
qsf.ugioc26.cn/753995.htm
qsf.ugioc26.cn/153955.htm
qsf.ugioc26.cn/111995.htm
qsf.ugioc26.cn/159975.htm
qsf.ugioc26.cn/737775.htm
qsf.ugioc26.cn/913375.htm
qsf.ugioc26.cn/711715.htm
qsf.ugioc26.cn/511195.htm
qsf.ugioc26.cn/737735.htm
qsd.ugioc26.cn/171755.htm
qsd.ugioc26.cn/553595.htm
qsd.ugioc26.cn/957395.htm
qsd.ugioc26.cn/515775.htm
qsd.ugioc26.cn/191995.htm
qsd.ugioc26.cn/371115.htm
qsd.ugioc26.cn/177555.htm
qsd.ugioc26.cn/991535.htm
qsd.ugioc26.cn/159555.htm
qsd.ugioc26.cn/159395.htm
qss.ugioc26.cn/717775.htm
qss.ugioc26.cn/571715.htm
qss.ugioc26.cn/559195.htm
qss.ugioc26.cn/733975.htm
qss.ugioc26.cn/193755.htm
qss.ugioc26.cn/133535.htm
qss.ugioc26.cn/119195.htm
qss.ugioc26.cn/911735.htm
qss.ugioc26.cn/397955.htm
qss.ugioc26.cn/733775.htm
qsa.ugioc26.cn/913975.htm
qsa.ugioc26.cn/797935.htm
qsa.ugioc26.cn/993995.htm
qsa.ugioc26.cn/315535.htm
qsa.ugioc26.cn/917955.htm
qsa.ugioc26.cn/977575.htm
qsa.ugioc26.cn/739775.htm
qsa.ugioc26.cn/131595.htm
qsa.ugioc26.cn/919335.htm
qsa.ugioc26.cn/553155.htm
qsp.ugioc26.cn/937195.htm
qsp.ugioc26.cn/311535.htm
qsp.ugioc26.cn/359575.htm
qsp.ugioc26.cn/535135.htm
qsp.ugioc26.cn/911315.htm
qsp.ugioc26.cn/157955.htm
qsp.ugioc26.cn/799515.htm
qsp.ugioc26.cn/533975.htm
qsp.ugioc26.cn/537395.htm
qsp.ugioc26.cn/975595.htm
qso.ugioc26.cn/133915.htm
qso.ugioc26.cn/511575.htm
qso.ugioc26.cn/311975.htm
qso.ugioc26.cn/915195.htm
qso.ugioc26.cn/151115.htm
qso.ugioc26.cn/131795.htm
qso.ugioc26.cn/333115.htm
qso.ugioc26.cn/591935.htm
qso.ugioc26.cn/313915.htm
qso.ugioc26.cn/753115.htm
qsi.ugioc26.cn/559555.htm
qsi.ugioc26.cn/133375.htm
qsi.ugioc26.cn/715975.htm
qsi.ugioc26.cn/535515.htm
qsi.ugioc26.cn/119755.htm
qsi.ugioc26.cn/753755.htm
qsi.ugioc26.cn/195115.htm
qsi.ugioc26.cn/937515.htm
qsi.ugioc26.cn/791555.htm
qsi.ugioc26.cn/159595.htm
qsu.ugioc26.cn/511355.htm
qsu.ugioc26.cn/997955.htm
qsu.ugioc26.cn/191335.htm
qsu.ugioc26.cn/513535.htm
qsu.ugioc26.cn/919135.htm
qsu.ugioc26.cn/113555.htm
qsu.ugioc26.cn/793395.htm
qsu.ugioc26.cn/175315.htm
qsu.ugioc26.cn/715795.htm
qsu.ugioc26.cn/373395.htm
qsy.ugioc26.cn/759355.htm
qsy.ugioc26.cn/997995.htm
qsy.ugioc26.cn/317595.htm
qsy.ugioc26.cn/337395.htm
qsy.ugioc26.cn/193195.htm
qsy.ugioc26.cn/355955.htm
qsy.ugioc26.cn/715395.htm
qsy.ugioc26.cn/155915.htm
qsy.ugioc26.cn/553715.htm
qsy.ugioc26.cn/517115.htm
qst.ugioc26.cn/599595.htm
qst.ugioc26.cn/731795.htm
qst.ugioc26.cn/939995.htm
qst.ugioc26.cn/157395.htm
qst.ugioc26.cn/315735.htm
qst.ugioc26.cn/357595.htm
qst.ugioc26.cn/793135.htm
qst.ugioc26.cn/599735.htm
qst.ugioc26.cn/513595.htm
qst.ugioc26.cn/397315.htm
qsr.ugioc26.cn/555995.htm
qsr.ugioc26.cn/919515.htm
qsr.ugioc26.cn/511395.htm
qsr.ugioc26.cn/751975.htm
qsr.ugioc26.cn/353355.htm
qsr.ugioc26.cn/751735.htm
qsr.ugioc26.cn/280485.htm
qsr.ugioc26.cn/737155.htm
qsr.ugioc26.cn/139335.htm
qsr.ugioc26.cn/822825.htm
qse.ugioc26.cn/337555.htm
qse.ugioc26.cn/919995.htm
qse.ugioc26.cn/977375.htm
qse.ugioc26.cn/511755.htm
qse.ugioc26.cn/595335.htm
qse.ugioc26.cn/399575.htm
qse.ugioc26.cn/804025.htm
qse.ugioc26.cn/573555.htm
qse.ugioc26.cn/339735.htm
qse.ugioc26.cn/957995.htm
qsw.ugioc26.cn/357515.htm
qsw.ugioc26.cn/979515.htm
qsw.ugioc26.cn/113155.htm
qsw.ugioc26.cn/355775.htm
qsw.ugioc26.cn/933175.htm
qsw.ugioc26.cn/373755.htm
qsw.ugioc26.cn/719775.htm
qsw.ugioc26.cn/335915.htm
qsw.ugioc26.cn/733955.htm
qsw.ugioc26.cn/713115.htm
qsq.ugioc26.cn/951755.htm
qsq.ugioc26.cn/577735.htm
qsq.ugioc26.cn/999515.htm
qsq.ugioc26.cn/135555.htm
qsq.ugioc26.cn/228845.htm
qsq.ugioc26.cn/315595.htm
qsq.ugioc26.cn/997355.htm
qsq.ugioc26.cn/331555.htm
qsq.ugioc26.cn/284045.htm
qsq.ugioc26.cn/771535.htm
qatv.ugioc26.cn/795955.htm
qatv.ugioc26.cn/911735.htm
qatv.ugioc26.cn/844445.htm
qatv.ugioc26.cn/513975.htm
qatv.ugioc26.cn/575575.htm
qatv.ugioc26.cn/175715.htm
qatv.ugioc26.cn/793335.htm
qatv.ugioc26.cn/973975.htm
qatv.ugioc26.cn/535395.htm
qatv.ugioc26.cn/751315.htm
qan.ugioc26.cn/331935.htm
qan.ugioc26.cn/795555.htm
qan.ugioc26.cn/371755.htm
qan.ugioc26.cn/591595.htm
qan.ugioc26.cn/884065.htm
qan.ugioc26.cn/555395.htm
qan.ugioc26.cn/177195.htm
