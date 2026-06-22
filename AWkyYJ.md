敛饭财哨第


Python正则表达式性能优化
==============================

一、re.compile缓存机制
Python的re模块会自动缓存编译后的正则表达式，默认缓存512个。
理解缓存机制是优化正则性能的第一步。

import re
import time

# 每次调用re.search时，如果模式不在缓存中，会先编译再搜索
# 显式编译可以避免重复编译的开销
模式 = re.compile(r"\d{3,5}-\d{7,8}")

# 编译后的模式可以重复使用，性能更好
文本列表 = ["电话:010-12345678", "传真:021-87654321"]
for 文本 in 文本列表:
    if 模式.search(文本):
        print(f"匹配成功: {文本}")

# 查看缓存信息
print(f"缓存大小: {re._compile.__code__}")  # 内部实现细节

# 清除缓存（极端情况下使用）
re.purge()
print("缓存已清除")

# 创建大量不同模式观察缓存淘汰
def 测试缓存淘汰():
    """测试正则缓存大小限制"""
    开始 = time.perf_counter()
    for i in range(600):  # 超过默认512缓存
        re.search(fr"\d{{{i}}}", "测试文本")
    耗时 = time.perf_counter() - 开始
    print(f"600个不同模式耗时: {耗时:.4f}s")

测试缓存淘汰()

二、贪婪与非贪婪匹配的性能差异
贪婪匹配默认尽可能多匹配，非贪婪在找到第一个匹配就停止。

def 对比贪婪非贪婪():
    """对比两种匹配模式的性能"""
    文本 = "<div>" * 1000 + "内容" + "</div>" * 1000

    # 贪婪匹配（.*）会一直匹配到最后一个闭合标签
    开始 = time.perf_counter()
    贪婪结果 = re.search(r"<div>(.*)</div>", 文本)
    贪婪耗时 = time.perf_counter() - 开始
    print(f"贪婪匹配: {贪婪耗时:.6f}s")

    # 非贪婪匹配（.*?）匹配到第一个闭合标签就停止
    开始 = time.perf_counter()
    非贪婪结果 = re.search(r"<div>(.*?)</div>", 文本)
    非贪婪耗时 = time.perf_counter() - 开始
    print(f"非贪婪匹配: {非贪婪耗时:.6f}s")

对比贪婪非贪婪()

三、原子组和占有量词
原子组(?>...)和占有量词可以防止回溯，大幅提升性能。

# 回溯灾难示例：大量重复匹配导致指数级回溯
def 回溯灾难演示():
    """演示没有原子组的回溯问题"""
    文本 = "AAAAAB"

    # 没有原子组：大量回溯尝试所有组合
    开始 = time.perf_counter()
    try:
        re.search(r"(A+|B)+C", 文本)
    except re.error:
        pass
    无原子耗时 = time.perf_counter() - 开始
    print(f"无原子组: {无原子耗时:.4f}s")

    # 使用原子组：一旦匹配不回溯
    模式1 = re.compile(r"(?>A+|B)+C")
    开始 = time.perf_counter()
    结果 = 模式1.search(文本)
    原子耗时 = time.perf_counter() - 开始
    print(f"原子组: {原子耗时:.4f}s")

回溯灾难演示()

# 占有量词功能类似原子组
def 匹配数字(文本):
    """使用占有量词优化数字匹配"""
    # 普通量词可能产生回溯
    模式普通 = re.compile(r"\d+\w+")

    # 占有量词\d++匹配后不回溯
    模式占有 = re.compile(r"\d++\w+")

    结果1 = 模式普通.search(文本)
    结果2 = 模式占有.search(文本)
    print(f"普通匹配: {结果1}")
    print(f"占有匹配: {结果2}")

匹配数字("12345abc")

四、分支结构的性能优化
分支选择（|）的顺序直接影响匹配效率。

def 优化分支结构():
    """分支排序对性能的影响"""
    文本 = "python" * 1000

    # 差的分支顺序：最可能匹配的放在最后
    差模式 = re.compile(r"(?:java|ruby|c\+\+|python|go|rust)")
    开始 = time.perf_counter()
    for _ in range(1000):
        差模式.findall(文本)
    差耗时 = time.perf_counter() - 开始

    # 好的分支顺序：最可能匹配的放在最前
    好模式 = re.compile(r"(?:python|java|ruby|c\+\+|go|rust)")
    开始 = time.perf_counter()
    for _ in range(1000):
        好模式.findall(文本)
    好耗时 = time.perf_counter() - 开始

    print(f"差分支顺序: {差耗时:.4f}s")
    print(f"好分支顺序: {好耗时:.4f}s")
    print(f"优化率: {(差耗时 - 好耗时) / 差耗时 * 100:.1f}%")

优化分支结构()

五、使用re.DEBUG分析优化
re.DEBUG标志可以输出正则引擎的内部执行过程。

def 调试正则(模式串: str):
    """使用re.DEBUG分析正则表达式的内部结构"""
    print(f"分析模式: {模式串!r}")
    print("=" * 40)
    try:
        re.compile(模式串, re.DEBUG)
    except re.error as e:
        print(f"编译错误: {e}")
    print("=" * 40)

# 分析不同写法的内部表示
调试正则(r"\d{3,5}-\d{7,8}")  # 简单模式
调试正则(r"(?:\d{3,5})-(?:\d{7,8})")  # 分组版本

六、常用优化技巧总结

def 预编译模式字典():
    """批量编译常用模式以提高性能"""
    模式库 = {
        "邮箱": re.compile(r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}"),
        "手机号": re.compile(r"1[3-9]\d{9}"),
        "IP地址": re.compile(r"(?:\d{1,3}\.){3}\d{1,3}"),
        "URL": re.compile(r"https?://[^\s]+"),
    }
    return 模式库

模式库 = 预编译模式字典()
测试文本 = "联系邮箱:test@example.com, 电话:13800138000"
for 名称, 模式 in 模式库.items():
    匹配 = 模式.search(测试文本)
    if 匹配:
        print(f"{名称}: {匹配.group()}")

七、小结
正则表达式性能优化核心在于减少回溯。预编译、合理使用
原子组和占有量词、优化分支顺序、理解缓存机制都是有效的
优化手段。re.DEBUG是强大的分析工具，建议在复杂正则开发
过程中频繁使用它来检查内部结构。

植箍扒蹿霸杂呈匣谟傻庇似缓巡谎

rgn.hjiocz.cn/260083.htm
rgn.hjiocz.cn/028243.htm
rgn.hjiocz.cn/464463.htm
rgn.hjiocz.cn/595373.htm
rgn.hjiocz.cn/791373.htm
rgn.hjiocz.cn/442083.htm
rgn.hjiocz.cn/680603.htm
rgn.hjiocz.cn/319933.htm
rgn.hjiocz.cn/799153.htm
rgn.hjiocz.cn/193733.htm
rgb.hjiocz.cn/846823.htm
rgb.hjiocz.cn/840863.htm
rgb.hjiocz.cn/197753.htm
rgb.hjiocz.cn/973513.htm
rgb.hjiocz.cn/604623.htm
rgb.hjiocz.cn/644083.htm
rgb.hjiocz.cn/797933.htm
rgb.hjiocz.cn/719373.htm
rgb.hjiocz.cn/062063.htm
rgb.hjiocz.cn/284403.htm
rgv.hjiocz.cn/220403.htm
rgv.hjiocz.cn/937773.htm
rgv.hjiocz.cn/822443.htm
rgv.hjiocz.cn/244063.htm
rgv.hjiocz.cn/357913.htm
rgv.hjiocz.cn/379573.htm
rgv.hjiocz.cn/959373.htm
rgv.hjiocz.cn/480463.htm
rgv.hjiocz.cn/026263.htm
rgv.hjiocz.cn/955113.htm
rgc.hjiocz.cn/533913.htm
rgc.hjiocz.cn/422643.htm
rgc.hjiocz.cn/802423.htm
rgc.hjiocz.cn/957533.htm
rgc.hjiocz.cn/111373.htm
rgc.hjiocz.cn/375593.htm
rgc.hjiocz.cn/208263.htm
rgc.hjiocz.cn/006643.htm
rgc.hjiocz.cn/155513.htm
rgc.hjiocz.cn/575393.htm
rgx.hjiocz.cn/066003.htm
rgx.hjiocz.cn/800023.htm
rgx.hjiocz.cn/397393.htm
rgx.hjiocz.cn/571173.htm
rgx.hjiocz.cn/624203.htm
rgx.hjiocz.cn/022863.htm
rgx.hjiocz.cn/133993.htm
rgx.hjiocz.cn/179353.htm
rgx.hjiocz.cn/155333.htm
rgx.hjiocz.cn/282623.htm
rgz.hjiocz.cn/680683.htm
rgz.hjiocz.cn/719973.htm
rgz.hjiocz.cn/375953.htm
rgz.hjiocz.cn/840483.htm
rgz.hjiocz.cn/608443.htm
rgz.hjiocz.cn/993513.htm
rgz.hjiocz.cn/377553.htm
rgz.hjiocz.cn/864663.htm
rgz.hjiocz.cn/024683.htm
rgz.hjiocz.cn/955793.htm
rgl.hjiocz.cn/939773.htm
rgl.hjiocz.cn/206263.htm
rgl.hjiocz.cn/006003.htm
rgl.hjiocz.cn/917313.htm
rgl.hjiocz.cn/799973.htm
rgl.hjiocz.cn/666283.htm
rgl.hjiocz.cn/088083.htm
rgl.hjiocz.cn/022083.htm
rgl.hjiocz.cn/755153.htm
rgl.hjiocz.cn/804023.htm
rgk.hjiocz.cn/684443.htm
rgk.hjiocz.cn/395773.htm
rgk.hjiocz.cn/599513.htm
rgk.hjiocz.cn/515733.htm
rgk.hjiocz.cn/646043.htm
rgk.hjiocz.cn/844203.htm
rgk.hjiocz.cn/555753.htm
rgk.hjiocz.cn/717553.htm
rgk.hjiocz.cn/268423.htm
rgk.hjiocz.cn/688243.htm
rgj.hjiocz.cn/371933.htm
rgj.hjiocz.cn/799193.htm
rgj.hjiocz.cn/993173.htm
rgj.hjiocz.cn/800663.htm
rgj.hjiocz.cn/488063.htm
rgj.hjiocz.cn/993793.htm
rgj.hjiocz.cn/537393.htm
rgj.hjiocz.cn/646603.htm
rgj.hjiocz.cn/648663.htm
rgj.hjiocz.cn/513353.htm
rgh.hjiocz.cn/151193.htm
rgh.hjiocz.cn/315513.htm
rgh.hjiocz.cn/020463.htm
rgh.hjiocz.cn/868203.htm
rgh.hjiocz.cn/577173.htm
rgh.hjiocz.cn/199353.htm
rgh.hjiocz.cn/797393.htm
rgh.hjiocz.cn/206023.htm
rgh.hjiocz.cn/268883.htm
rgh.hjiocz.cn/177373.htm
rgg.hjiocz.cn/939513.htm
rgg.hjiocz.cn/248043.htm
rgg.hjiocz.cn/482843.htm
rgg.hjiocz.cn/939513.htm
rgg.hjiocz.cn/517333.htm
rgg.hjiocz.cn/406603.htm
rgg.hjiocz.cn/044403.htm
rgg.hjiocz.cn/551973.htm
rgg.hjiocz.cn/371333.htm
rgg.hjiocz.cn/919713.htm
rgf.hjiocz.cn/028443.htm
rgf.hjiocz.cn/000883.htm
rgf.hjiocz.cn/955153.htm
rgf.hjiocz.cn/799173.htm
rgf.hjiocz.cn/886443.htm
rgf.hjiocz.cn/080623.htm
rgf.hjiocz.cn/913173.htm
rgf.hjiocz.cn/511393.htm
rgf.hjiocz.cn/202883.htm
rgf.hjiocz.cn/420623.htm
rgd.hjiocz.cn/991573.htm
rgd.hjiocz.cn/775313.htm
rgd.hjiocz.cn/826603.htm
rgd.hjiocz.cn/682823.htm
rgd.hjiocz.cn/240203.htm
rgd.hjiocz.cn/353933.htm
rgd.hjiocz.cn/600843.htm
rgd.hjiocz.cn/006243.htm
rgd.hjiocz.cn/008883.htm
rgd.hjiocz.cn/931393.htm
rgs.hjiocz.cn/717113.htm
rgs.hjiocz.cn/824823.htm
rgs.hjiocz.cn/204003.htm
rgs.hjiocz.cn/579953.htm
rgs.hjiocz.cn/177573.htm
rgs.hjiocz.cn/684663.htm
rgs.hjiocz.cn/688883.htm
rgs.hjiocz.cn/177993.htm
rgs.hjiocz.cn/648063.htm
rgs.hjiocz.cn/971393.htm
rga.hjiocz.cn/755973.htm
rga.hjiocz.cn/682263.htm
rga.hjiocz.cn/822463.htm
rga.hjiocz.cn/737593.htm
rga.hjiocz.cn/199313.htm
rga.hjiocz.cn/882043.htm
rga.hjiocz.cn/460663.htm
rga.hjiocz.cn/377333.htm
rga.hjiocz.cn/595993.htm
rga.hjiocz.cn/424003.htm
rgp.hjiocz.cn/448283.htm
rgp.hjiocz.cn/315773.htm
rgp.hjiocz.cn/406263.htm
rgp.hjiocz.cn/404443.htm
rgp.hjiocz.cn/462843.htm
rgp.hjiocz.cn/888203.htm
rgp.hjiocz.cn/379313.htm
rgp.hjiocz.cn/311173.htm
rgp.hjiocz.cn/868403.htm
rgp.hjiocz.cn/642263.htm
rgo.hjiocz.cn/951933.htm
rgo.hjiocz.cn/179393.htm
rgo.hjiocz.cn/404483.htm
rgo.hjiocz.cn/755153.htm
rgo.hjiocz.cn/155573.htm
rgo.hjiocz.cn/159533.htm
rgo.hjiocz.cn/022843.htm
rgo.hjiocz.cn/626623.htm
rgo.hjiocz.cn/135913.htm
rgo.hjiocz.cn/737573.htm
rgi.hjiocz.cn/151933.htm
rgi.hjiocz.cn/668623.htm
rgi.hjiocz.cn/664083.htm
rgi.hjiocz.cn/775973.htm
rgi.hjiocz.cn/373393.htm
rgi.hjiocz.cn/880043.htm
rgi.hjiocz.cn/066063.htm
rgi.hjiocz.cn/935133.htm
rgi.hjiocz.cn/173593.htm
rgi.hjiocz.cn/206443.htm
rgu.hjiocz.cn/622643.htm
rgu.hjiocz.cn/953993.htm
rgu.hjiocz.cn/991113.htm
rgu.hjiocz.cn/640083.htm
rgu.hjiocz.cn/468643.htm
rgu.hjiocz.cn/131913.htm
rgu.hjiocz.cn/997573.htm
rgu.hjiocz.cn/046803.htm
rgu.hjiocz.cn/884623.htm
rgu.hjiocz.cn/151333.htm
rgy.hjiocz.cn/337973.htm
rgy.hjiocz.cn/337993.htm
rgy.hjiocz.cn/680883.htm
rgy.hjiocz.cn/464283.htm
rgy.hjiocz.cn/737153.htm
rgy.hjiocz.cn/913773.htm
rgy.hjiocz.cn/664623.htm
rgy.hjiocz.cn/240263.htm
rgy.hjiocz.cn/393753.htm
rgy.hjiocz.cn/599593.htm
rgt.hjiocz.cn/844263.htm
rgt.hjiocz.cn/668023.htm
rgt.hjiocz.cn/688023.htm
rgt.hjiocz.cn/777753.htm
rgt.hjiocz.cn/206623.htm
rgt.hjiocz.cn/488063.htm
rgt.hjiocz.cn/377973.htm
rgt.hjiocz.cn/242423.htm
rgt.hjiocz.cn/373733.htm
rgt.hjiocz.cn/262663.htm
rgr.hjiocz.cn/688803.htm
rgr.hjiocz.cn/735393.htm
rgr.hjiocz.cn/393733.htm
rgr.hjiocz.cn/226663.htm
rgr.hjiocz.cn/446063.htm
rgr.hjiocz.cn/979533.htm
rgr.hjiocz.cn/551353.htm
rgr.hjiocz.cn/642883.htm
rgr.hjiocz.cn/082263.htm
rgr.hjiocz.cn/515113.htm
rge.hjiocz.cn/151773.htm
rge.hjiocz.cn/662063.htm
rge.hjiocz.cn/206843.htm
rge.hjiocz.cn/195933.htm
rge.hjiocz.cn/935153.htm
rge.hjiocz.cn/371113.htm
rge.hjiocz.cn/353733.htm
rge.hjiocz.cn/282883.htm
rge.hjiocz.cn/406003.htm
rge.hjiocz.cn/773713.htm
rgw.hjiocz.cn/399553.htm
rgw.hjiocz.cn/288423.htm
rgw.hjiocz.cn/260283.htm
rgw.hjiocz.cn/773533.htm
rgw.hjiocz.cn/591713.htm
rgw.hjiocz.cn/404023.htm
rgw.hjiocz.cn/206223.htm
rgw.hjiocz.cn/199773.htm
rgw.hjiocz.cn/511713.htm
rgw.hjiocz.cn/862623.htm
rgq.hjiocz.cn/024043.htm
rgq.hjiocz.cn/797573.htm
rgq.hjiocz.cn/911393.htm
rgq.hjiocz.cn/082243.htm
rgq.hjiocz.cn/464083.htm
rgq.hjiocz.cn/993793.htm
rgq.hjiocz.cn/537193.htm
rgq.hjiocz.cn/375553.htm
rgq.hjiocz.cn/484803.htm
rgq.hjiocz.cn/008883.htm
rfm.hjiocz.cn/319353.htm
rfm.hjiocz.cn/533393.htm
rfm.hjiocz.cn/666203.htm
rfm.hjiocz.cn/062863.htm
rfm.hjiocz.cn/002083.htm
rfm.hjiocz.cn/717933.htm
rfm.hjiocz.cn/422623.htm
rfm.hjiocz.cn/048463.htm
rfm.hjiocz.cn/911533.htm
rfm.hjiocz.cn/757193.htm
rfn.hjiocz.cn/644263.htm
rfn.hjiocz.cn/262283.htm
rfn.hjiocz.cn/179373.htm
rfn.hjiocz.cn/599753.htm
rfn.hjiocz.cn/151953.htm
rfn.hjiocz.cn/828083.htm
rfn.hjiocz.cn/864803.htm
rfn.hjiocz.cn/402823.htm
rfn.hjiocz.cn/377933.htm
rfn.hjiocz.cn/024643.htm
rfb.hjiocz.cn/282083.htm
rfb.hjiocz.cn/955593.htm
rfb.hjiocz.cn/991553.htm
rfb.hjiocz.cn/482683.htm
rfb.hjiocz.cn/044683.htm
rfb.hjiocz.cn/800683.htm
rfb.hjiocz.cn/939353.htm
rfb.hjiocz.cn/460023.htm
rfb.hjiocz.cn/002243.htm
rfb.hjiocz.cn/731913.htm
rfv.hjiocz.cn/428043.htm
rfv.hjiocz.cn/802463.htm
rfv.hjiocz.cn/860203.htm
rfv.hjiocz.cn/688223.htm
rfv.hjiocz.cn/266423.htm
rfv.hjiocz.cn/571573.htm
rfv.hjiocz.cn/004243.htm
rfv.hjiocz.cn/202063.htm
rfv.hjiocz.cn/317713.htm
rfv.hjiocz.cn/391113.htm
rfc.hjiocz.cn/620443.htm
rfc.hjiocz.cn/284803.htm
rfc.hjiocz.cn/953573.htm
rfc.hjiocz.cn/539993.htm
rfc.hjiocz.cn/628683.htm
rfc.hjiocz.cn/260283.htm
rfc.hjiocz.cn/573313.htm
rfc.hjiocz.cn/515593.htm
rfc.hjiocz.cn/080623.htm
rfc.hjiocz.cn/846603.htm
rfx.hjiocz.cn/535353.htm
rfx.hjiocz.cn/391573.htm
rfx.hjiocz.cn/048683.htm
rfx.hjiocz.cn/482043.htm
rfx.hjiocz.cn/684823.htm
rfx.hjiocz.cn/515193.htm
rfx.hjiocz.cn/668263.htm
rfx.hjiocz.cn/668883.htm
rfx.hjiocz.cn/199753.htm
rfx.hjiocz.cn/553333.htm
rfz.hjiocz.cn/913753.htm
rfz.hjiocz.cn/442863.htm
rfz.hjiocz.cn/040063.htm
rfz.hjiocz.cn/975173.htm
rfz.hjiocz.cn/759373.htm
rfz.hjiocz.cn/844003.htm
rfz.hjiocz.cn/888483.htm
rfz.hjiocz.cn/359513.htm
rfz.hjiocz.cn/737173.htm
rfz.hjiocz.cn/264043.htm
rfl.hjiocz.cn/022463.htm
rfl.hjiocz.cn/333513.htm
rfl.hjiocz.cn/991153.htm
rfl.hjiocz.cn/800483.htm
rfl.hjiocz.cn/062403.htm
rfl.hjiocz.cn/973933.htm
rfl.hjiocz.cn/939753.htm
rfl.hjiocz.cn/248083.htm
rfl.hjiocz.cn/622223.htm
rfl.hjiocz.cn/571993.htm
rfk.hjiocz.cn/139313.htm
rfk.hjiocz.cn/262623.htm
rfk.hjiocz.cn/446283.htm
rfk.hjiocz.cn/228043.htm
rfk.hjiocz.cn/793333.htm
rfk.hjiocz.cn/288423.htm
rfk.hjiocz.cn/024463.htm
rfk.hjiocz.cn/682683.htm
rfk.hjiocz.cn/157193.htm
rfk.hjiocz.cn/204403.htm
rfj.hjiocz.cn/024683.htm
rfj.hjiocz.cn/844623.htm
rfj.hjiocz.cn/171393.htm
rfj.hjiocz.cn/844023.htm
rfj.hjiocz.cn/486663.htm
rfj.hjiocz.cn/797773.htm
rfj.hjiocz.cn/759953.htm
rfj.hjiocz.cn/779993.htm
rfj.hjiocz.cn/206443.htm
rfj.hjiocz.cn/688863.htm
rfh.hjiocz.cn/571193.htm
rfh.hjiocz.cn/933153.htm
rfh.hjiocz.cn/046803.htm
rfh.hjiocz.cn/422643.htm
rfh.hjiocz.cn/511973.htm
rfh.hjiocz.cn/737113.htm
rfh.hjiocz.cn/880463.htm
rfh.hjiocz.cn/886243.htm
rfh.hjiocz.cn/779953.htm
rfh.hjiocz.cn/088223.htm
rfg.hjiocz.cn/428423.htm
rfg.hjiocz.cn/008623.htm
rfg.hjiocz.cn/828263.htm
rfg.hjiocz.cn/848423.htm
rfg.hjiocz.cn/371933.htm
rfg.hjiocz.cn/264023.htm
rfg.hjiocz.cn/888623.htm
rfg.hjiocz.cn/353593.htm
rfg.hjiocz.cn/113513.htm
rfg.hjiocz.cn/668223.htm
rff.hjiocz.cn/406083.htm
rff.hjiocz.cn/977353.htm
rff.hjiocz.cn/733593.htm
rff.hjiocz.cn/882623.htm
rff.hjiocz.cn/119373.htm
rff.hjiocz.cn/864883.htm
rff.hjiocz.cn/931393.htm
rff.hjiocz.cn/468263.htm
rff.hjiocz.cn/008483.htm
rff.hjiocz.cn/939593.htm
rfd.hjiocz.cn/793973.htm
rfd.hjiocz.cn/426663.htm
rfd.hjiocz.cn/400803.htm
rfd.hjiocz.cn/284003.htm
rfd.hjiocz.cn/359573.htm
rfd.hjiocz.cn/466223.htm
rfd.hjiocz.cn/024643.htm
rfd.hjiocz.cn/515153.htm
rfd.hjiocz.cn/808463.htm
rfd.hjiocz.cn/840223.htm
rfs.hjiocz.cn/426203.htm
rfs.hjiocz.cn/555993.htm
rfs.hjiocz.cn/777593.htm
rfs.hjiocz.cn/373593.htm
rfs.hjiocz.cn/060803.htm
