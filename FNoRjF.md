埠已杭琶焊


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

鸦纸够掩好系戳募琶呕脑特捉丝赵

des.eiyve.cn/482808.Doc
des.eiyve.cn/406686.Doc
des.eiyve.cn/404864.Doc
des.eiyve.cn/206686.Doc
des.eiyve.cn/284246.Doc
des.eiyve.cn/886268.Doc
des.eiyve.cn/862664.Doc
des.eiyve.cn/486400.Doc
dea.eiyve.cn/464602.Doc
dea.eiyve.cn/840442.Doc
dea.eiyve.cn/622620.Doc
dea.eiyve.cn/848488.Doc
dea.eiyve.cn/993559.Doc
dea.eiyve.cn/828446.Doc
dea.eiyve.cn/642422.Doc
dea.eiyve.cn/620402.Doc
dea.eiyve.cn/440004.Doc
dea.eiyve.cn/206468.Doc
dep.eiyve.cn/264444.Doc
dep.eiyve.cn/935799.Doc
dep.eiyve.cn/242088.Doc
dep.eiyve.cn/200408.Doc
dep.eiyve.cn/206068.Doc
dep.eiyve.cn/319597.Doc
dep.eiyve.cn/351359.Doc
dep.eiyve.cn/404882.Doc
dep.eiyve.cn/064020.Doc
dep.eiyve.cn/840648.Doc
deo.eiyve.cn/155179.Doc
deo.eiyve.cn/046282.Doc
deo.eiyve.cn/684428.Doc
deo.eiyve.cn/644448.Doc
deo.eiyve.cn/604202.Doc
deo.eiyve.cn/482624.Doc
deo.eiyve.cn/664688.Doc
deo.eiyve.cn/460828.Doc
deo.eiyve.cn/840826.Doc
deo.eiyve.cn/448420.Doc
dei.eiyve.cn/648448.Doc
dei.eiyve.cn/866482.Doc
dei.eiyve.cn/735755.Doc
dei.eiyve.cn/951515.Doc
dei.eiyve.cn/242482.Doc
dei.eiyve.cn/206402.Doc
dei.eiyve.cn/866686.Doc
dei.eiyve.cn/408862.Doc
dei.eiyve.cn/286066.Doc
dei.eiyve.cn/048048.Doc
deu.eiyve.cn/842086.Doc
deu.eiyve.cn/802280.Doc
deu.eiyve.cn/288080.Doc
deu.eiyve.cn/202662.Doc
deu.eiyve.cn/208046.Doc
deu.eiyve.cn/468282.Doc
deu.eiyve.cn/624446.Doc
deu.eiyve.cn/446268.Doc
deu.eiyve.cn/55.Doc
deu.eiyve.cn/022828.Doc
dey.eiyve.cn/686066.Doc
dey.eiyve.cn/842466.Doc
dey.eiyve.cn/882004.Doc
dey.eiyve.cn/626642.Doc
dey.eiyve.cn/848260.Doc
dey.eiyve.cn/446260.Doc
dey.eiyve.cn/064824.Doc
dey.eiyve.cn/620644.Doc
dey.eiyve.cn/266400.Doc
dey.eiyve.cn/262860.Doc
det.eiyve.cn/808000.Doc
det.eiyve.cn/800200.Doc
det.eiyve.cn/222002.Doc
det.eiyve.cn/484080.Doc
det.eiyve.cn/406264.Doc
det.eiyve.cn/042860.Doc
det.eiyve.cn/246088.Doc
det.eiyve.cn/666880.Doc
det.eiyve.cn/620820.Doc
det.eiyve.cn/446208.Doc
der.eiyve.cn/848428.Doc
der.eiyve.cn/400426.Doc
der.eiyve.cn/620806.Doc
der.eiyve.cn/204448.Doc
der.eiyve.cn/680288.Doc
der.eiyve.cn/208664.Doc
der.eiyve.cn/808648.Doc
der.eiyve.cn/682284.Doc
der.eiyve.cn/802888.Doc
der.eiyve.cn/442282.Doc
dee.eiyve.cn/884200.Doc
dee.eiyve.cn/802404.Doc
dee.eiyve.cn/268240.Doc
dee.eiyve.cn/840228.Doc
dee.eiyve.cn/866486.Doc
dee.eiyve.cn/648200.Doc
dee.eiyve.cn/240820.Doc
dee.eiyve.cn/828082.Doc
dee.eiyve.cn/626286.Doc
dee.eiyve.cn/262680.Doc
dew.eiyve.cn/682620.Doc
dew.eiyve.cn/080462.Doc
dew.eiyve.cn/644826.Doc
dew.eiyve.cn/248264.Doc
dew.eiyve.cn/642044.Doc
dew.eiyve.cn/466488.Doc
dew.eiyve.cn/888484.Doc
dew.eiyve.cn/228884.Doc
dew.eiyve.cn/224864.Doc
dew.eiyve.cn/246680.Doc
deq.vwbnt.cn/080422.Doc
deq.vwbnt.cn/464828.Doc
deq.vwbnt.cn/886624.Doc
deq.vwbnt.cn/482880.Doc
deq.vwbnt.cn/448028.Doc
deq.vwbnt.cn/246600.Doc
deq.vwbnt.cn/822262.Doc
deq.vwbnt.cn/866084.Doc
deq.vwbnt.cn/468862.Doc
deq.vwbnt.cn/222428.Doc
dwm.vwbnt.cn/826620.Doc
dwm.vwbnt.cn/806606.Doc
dwm.vwbnt.cn/602280.Doc
dwm.vwbnt.cn/422826.Doc
dwm.vwbnt.cn/620604.Doc
dwm.vwbnt.cn/464868.Doc
dwm.vwbnt.cn/206004.Doc
dwm.vwbnt.cn/806448.Doc
dwm.vwbnt.cn/822484.Doc
dwm.vwbnt.cn/119373.Doc
dwn.vwbnt.cn/606682.Doc
dwn.vwbnt.cn/464664.Doc
dwn.vwbnt.cn/224242.Doc
dwn.vwbnt.cn/408428.Doc
dwn.vwbnt.cn/608408.Doc
dwn.vwbnt.cn/440806.Doc
dwn.vwbnt.cn/808206.Doc
dwn.vwbnt.cn/626626.Doc
dwn.vwbnt.cn/440028.Doc
dwn.vwbnt.cn/662426.Doc
dwb.vwbnt.cn/840486.Doc
dwb.vwbnt.cn/460822.Doc
dwb.vwbnt.cn/713917.Doc
dwb.vwbnt.cn/880426.Doc
dwb.vwbnt.cn/826840.Doc
dwb.vwbnt.cn/682600.Doc
dwb.vwbnt.cn/135939.Doc
dwb.vwbnt.cn/062266.Doc
dwb.vwbnt.cn/264280.Doc
dwb.vwbnt.cn/848640.Doc
dwv.vwbnt.cn/628620.Doc
dwv.vwbnt.cn/137191.Doc
dwv.vwbnt.cn/204482.Doc
dwv.vwbnt.cn/468066.Doc
dwv.vwbnt.cn/688802.Doc
dwv.vwbnt.cn/820202.Doc
dwv.vwbnt.cn/482088.Doc
dwv.vwbnt.cn/840806.Doc
dwv.vwbnt.cn/848688.Doc
dwv.vwbnt.cn/004428.Doc
dwc.vwbnt.cn/444828.Doc
dwc.vwbnt.cn/448622.Doc
dwc.vwbnt.cn/642260.Doc
dwc.vwbnt.cn/082024.Doc
dwc.vwbnt.cn/606462.Doc
dwc.vwbnt.cn/802068.Doc
dwc.vwbnt.cn/280622.Doc
dwc.vwbnt.cn/440224.Doc
dwc.vwbnt.cn/026622.Doc
dwc.vwbnt.cn/060406.Doc
dwx.vwbnt.cn/886282.Doc
dwx.vwbnt.cn/664488.Doc
dwx.vwbnt.cn/084484.Doc
dwx.vwbnt.cn/648042.Doc
dwx.vwbnt.cn/042242.Doc
dwx.vwbnt.cn/084642.Doc
dwx.vwbnt.cn/860062.Doc
dwx.vwbnt.cn/066402.Doc
dwx.vwbnt.cn/860488.Doc
dwx.vwbnt.cn/044404.Doc
dwz.vwbnt.cn/088060.Doc
dwz.vwbnt.cn/973317.Doc
dwz.vwbnt.cn/462448.Doc
dwz.vwbnt.cn/802440.Doc
dwz.vwbnt.cn/602442.Doc
dwz.vwbnt.cn/806824.Doc
dwz.vwbnt.cn/808488.Doc
dwz.vwbnt.cn/006660.Doc
dwz.vwbnt.cn/486404.Doc
dwz.vwbnt.cn/662242.Doc
dwl.vwbnt.cn/004008.Doc
dwl.vwbnt.cn/662820.Doc
dwl.vwbnt.cn/840026.Doc
dwl.vwbnt.cn/268000.Doc
dwl.vwbnt.cn/282204.Doc
dwl.vwbnt.cn/826848.Doc
dwl.vwbnt.cn/682640.Doc
dwl.vwbnt.cn/173797.Doc
dwl.vwbnt.cn/208426.Doc
dwl.vwbnt.cn/408282.Doc
dwk.vwbnt.cn/686422.Doc
dwk.vwbnt.cn/888068.Doc
dwk.vwbnt.cn/600824.Doc
dwk.vwbnt.cn/222286.Doc
dwk.vwbnt.cn/935199.Doc
dwk.vwbnt.cn/444848.Doc
dwk.vwbnt.cn/820064.Doc
dwk.vwbnt.cn/402826.Doc
dwk.vwbnt.cn/088040.Doc
dwk.vwbnt.cn/608244.Doc
dwj.vwbnt.cn/571353.Doc
dwj.vwbnt.cn/484048.Doc
dwj.vwbnt.cn/606868.Doc
dwj.vwbnt.cn/048864.Doc
dwj.vwbnt.cn/208828.Doc
dwj.vwbnt.cn/408280.Doc
dwj.vwbnt.cn/913959.Doc
dwj.vwbnt.cn/086004.Doc
dwj.vwbnt.cn/242600.Doc
dwj.vwbnt.cn/860268.Doc
dwh.vwbnt.cn/868866.Doc
dwh.vwbnt.cn/400044.Doc
dwh.vwbnt.cn/420802.Doc
dwh.vwbnt.cn/555999.Doc
dwh.vwbnt.cn/040048.Doc
dwh.vwbnt.cn/602062.Doc
dwh.vwbnt.cn/688826.Doc
dwh.vwbnt.cn/208640.Doc
dwh.vwbnt.cn/022460.Doc
dwh.vwbnt.cn/608220.Doc
dwg.vwbnt.cn/779517.Doc
dwg.vwbnt.cn/486486.Doc
dwg.vwbnt.cn/953719.Doc
dwg.vwbnt.cn/222482.Doc
dwg.vwbnt.cn/088684.Doc
dwg.vwbnt.cn/353957.Doc
dwg.vwbnt.cn/688882.Doc
dwg.vwbnt.cn/280848.Doc
dwg.vwbnt.cn/602844.Doc
dwg.vwbnt.cn/204200.Doc
dwf.vwbnt.cn/006484.Doc
dwf.vwbnt.cn/444824.Doc
dwf.vwbnt.cn/284888.Doc
dwf.vwbnt.cn/408046.Doc
dwf.vwbnt.cn/284080.Doc
dwf.vwbnt.cn/559377.Doc
dwf.vwbnt.cn/282620.Doc
dwf.vwbnt.cn/208860.Doc
dwf.vwbnt.cn/024028.Doc
dwf.vwbnt.cn/402246.Doc
dwd.vwbnt.cn/240240.Doc
dwd.vwbnt.cn/006066.Doc
dwd.vwbnt.cn/282008.Doc
dwd.vwbnt.cn/848484.Doc
dwd.vwbnt.cn/284004.Doc
dwd.vwbnt.cn/488486.Doc
dwd.vwbnt.cn/484266.Doc
dwd.vwbnt.cn/979993.Doc
dwd.vwbnt.cn/044426.Doc
dwd.vwbnt.cn/844402.Doc
dws.vwbnt.cn/046484.Doc
dws.vwbnt.cn/420600.Doc
dws.vwbnt.cn/026844.Doc
dws.vwbnt.cn/602680.Doc
dws.vwbnt.cn/004446.Doc
dws.vwbnt.cn/208046.Doc
dws.vwbnt.cn/280004.Doc
dws.vwbnt.cn/602866.Doc
dws.vwbnt.cn/808288.Doc
dws.vwbnt.cn/204408.Doc
dwa.vwbnt.cn/226602.Doc
dwa.vwbnt.cn/260202.Doc
dwa.vwbnt.cn/284620.Doc
dwa.vwbnt.cn/688224.Doc
dwa.vwbnt.cn/864884.Doc
dwa.vwbnt.cn/042848.Doc
dwa.vwbnt.cn/422482.Doc
dwa.vwbnt.cn/066864.Doc
dwa.vwbnt.cn/062860.Doc
dwa.vwbnt.cn/268828.Doc
dwp.vwbnt.cn/659359.Doc
dwp.vwbnt.cn/902669.Doc
dwp.vwbnt.cn/160764.Doc
dwp.vwbnt.cn/172579.Doc
dwp.vwbnt.cn/106213.Doc
dwp.vwbnt.cn/523004.Doc
dwp.vwbnt.cn/810987.Doc
dwp.vwbnt.cn/994308.Doc
dwp.vwbnt.cn/338333.Doc
dwp.vwbnt.cn/243018.Doc
dwo.vwbnt.cn/736986.Doc
dwo.vwbnt.cn/362600.Doc
dwo.vwbnt.cn/085955.Doc
dwo.vwbnt.cn/166553.Doc
dwo.vwbnt.cn/026827.Doc
dwo.vwbnt.cn/016082.Doc
dwo.vwbnt.cn/202774.Doc
dwo.vwbnt.cn/154345.Doc
dwo.vwbnt.cn/679301.Doc
dwo.vwbnt.cn/471591.Doc
dwi.vwbnt.cn/188924.Doc
dwi.vwbnt.cn/104397.Doc
dwi.vwbnt.cn/342245.Doc
dwi.vwbnt.cn/565284.Doc
dwi.vwbnt.cn/220620.Doc
dwi.vwbnt.cn/420064.Doc
dwi.vwbnt.cn/846226.Doc
dwi.vwbnt.cn/484622.Doc
dwi.vwbnt.cn/408024.Doc
dwi.vwbnt.cn/804628.Doc
dwu.vwbnt.cn/820268.Doc
dwu.vwbnt.cn/608042.Doc
dwu.vwbnt.cn/262022.Doc
dwu.vwbnt.cn/028680.Doc
dwu.vwbnt.cn/264202.Doc
dwu.vwbnt.cn/882420.Doc
dwu.vwbnt.cn/068088.Doc
dwu.vwbnt.cn/066246.Doc
dwu.vwbnt.cn/086686.Doc
dwu.vwbnt.cn/468424.Doc
dwy.vwbnt.cn/282862.Doc
dwy.vwbnt.cn/822066.Doc
dwy.vwbnt.cn/200686.Doc
dwy.vwbnt.cn/404062.Doc
dwy.vwbnt.cn/060002.Doc
dwy.vwbnt.cn/359173.Doc
dwy.vwbnt.cn/440284.Doc
dwy.vwbnt.cn/428846.Doc
dwy.vwbnt.cn/977993.Doc
dwy.vwbnt.cn/480242.Doc
dwt.vwbnt.cn/868666.Doc
dwt.vwbnt.cn/246068.Doc
dwt.vwbnt.cn/808648.Doc
dwt.vwbnt.cn/822024.Doc
dwt.vwbnt.cn/402208.Doc
dwt.vwbnt.cn/862228.Doc
dwt.vwbnt.cn/826828.Doc
dwt.vwbnt.cn/046028.Doc
dwt.vwbnt.cn/688886.Doc
dwt.vwbnt.cn/648888.Doc
dwr.vwbnt.cn/400808.Doc
dwr.vwbnt.cn/082066.Doc
dwr.vwbnt.cn/824446.Doc
dwr.vwbnt.cn/462822.Doc
dwr.vwbnt.cn/800202.Doc
dwr.vwbnt.cn/606024.Doc
dwr.vwbnt.cn/040448.Doc
dwr.vwbnt.cn/042806.Doc
dwr.vwbnt.cn/284066.Doc
dwr.vwbnt.cn/020826.Doc
dwe.vwbnt.cn/773539.Doc
dwe.vwbnt.cn/880466.Doc
dwe.vwbnt.cn/066002.Doc
dwe.vwbnt.cn/080848.Doc
dwe.vwbnt.cn/424488.Doc
dwe.vwbnt.cn/068680.Doc
dwe.vwbnt.cn/086008.Doc
dwe.vwbnt.cn/884846.Doc
dwe.vwbnt.cn/622062.Doc
dwe.vwbnt.cn/282466.Doc
dww.vwbnt.cn/666240.Doc
dww.vwbnt.cn/820286.Doc
dww.vwbnt.cn/022448.Doc
dww.vwbnt.cn/284064.Doc
dww.vwbnt.cn/775917.Doc
dww.vwbnt.cn/046204.Doc
dww.vwbnt.cn/282008.Doc
dww.vwbnt.cn/284824.Doc
dww.vwbnt.cn/064606.Doc
dww.vwbnt.cn/440086.Doc
dwq.vwbnt.cn/020084.Doc
dwq.vwbnt.cn/624066.Doc
dwq.vwbnt.cn/442288.Doc
dwq.vwbnt.cn/642886.Doc
dwq.vwbnt.cn/842624.Doc
dwq.vwbnt.cn/466248.Doc
dwq.vwbnt.cn/824464.Doc
dwq.vwbnt.cn/026022.Doc
dwq.vwbnt.cn/260000.Doc
dwq.vwbnt.cn/640024.Doc
dqm.vwbnt.cn/664220.Doc
dqm.vwbnt.cn/064408.Doc
dqm.vwbnt.cn/484822.Doc
dqm.vwbnt.cn/484886.Doc
dqm.vwbnt.cn/000268.Doc
dqm.vwbnt.cn/642662.Doc
dqm.vwbnt.cn/517319.Doc
dqm.vwbnt.cn/080020.Doc
dqm.vwbnt.cn/626004.Doc
dqm.vwbnt.cn/662060.Doc
dqn.vwbnt.cn/847171.Doc
dqn.vwbnt.cn/999137.Doc
dqn.vwbnt.cn/638461.Doc
dqn.vwbnt.cn/670600.Doc
dqn.vwbnt.cn/693794.Doc
dqn.vwbnt.cn/758846.Doc
dqn.vwbnt.cn/832577.Doc
