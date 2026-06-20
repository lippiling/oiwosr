恼恋诽炯惫


Python集合与冻结集合高级
==============================

一、frozenset作为字典键
frozenset是不可变的哈希集合，可以作为字典的键或集合的元素，
而普通set则不行。

# frozenset作为字典键
路由表 = {
    frozenset({"北京", "上海"}): "京沪线",
    frozenset({"广州", "深圳"}): "广深线",
    frozenset({"成都", "重庆"}): "成渝线",
}

def 查询路线(城市1: str, 城市2: str) -> str:
    """根据两个城市查询路线名称"""
    路线键 = frozenset({城市1, 城市2})
    return 路由表.get(路线键, "未知路线")

print(f"路线: {查询路线('北京', '上海')}")
print(f"路线: {查询路线('上海', '北京')}")  # 顺序无关

# frozenset作为集合的元素
用户权限组 = {
    frozenset({"读取", "写入"}),           # 普通用户
    frozenset({"读取", "写入", "删除"}),    # 管理员
    frozenset({"读取"}),                    # 只读用户
}
print(f"权限组数量: {len(用户权限组)}")

# 检查权限组合是否存在
def 检查权限组合(*权限):
    return frozenset(权限) in 用户权限组

print(f"读写权限存在: {检查权限组合('读取', '写入')}")

二、集合/字典视图对象
keys()、values()、items()返回的是动态视图，反映底层dict的变化。

配置 = {"主机": "localhost", "端口": 8080, "调试": True}

# 创建视图
键视图 = 配置.keys()
值视图 = 配置.values()
项视图 = 配置.items()

print(f"键视图: {list(键视图)}")

# 修改原字典，视图自动更新
配置["超时"] = 30
配置["主机"] = "192.168.1.1"

print(f"更新后键视图: {list(键视图)}")  # 包含新键
print(f"值视图: {list(值视图)}")

# 视图支持集合运算（键视图和项视图）
其他配置 = {"主机": "10.0.0.1", "端口": 9090, "协议": "https"}

交集键 = 配置.keys() & 其他配置.keys()
差集键 = 配置.keys() - 其他配置.keys()
并集键 = 配置.keys() | 其他配置.keys()

print(f"共同的键: {交集键}")
print(f"独有的键: {差集键}")
print(f"所有键: {并集键}")

三、集合运算链式调用
集合的union、intersection等方法可以链式调用，返回新集合。

基础用户 = {"张三", "李四", "王五"}
VIP用户 = {"李四", "赵六", "钱七"}
管理员 = {"王五", "孙八"}
黑名单 = {"赵六"}

# 链式集合运算：找出所有可发送通知的用户
可通知用户 = (基础用户 | VIP用户) - 黑名单
print(f"可通知用户: {可通知用户}")

# 复杂的集合运算链
结果 = 基础用户.intersection(VIP用户).union(管理员).difference(黑名单)
print(f"链式结果: {结果}")

# |=, &=, -=, ^= 原地操作
会话权限 = {"读取", "写入"}
会话权限 |= {"删除"}  # 添加权限
print(f"更新后权限: {会话权限}")

四、集合推导式性能优化
集合推导式比循环add方式更快，内部有优化。

import time

def 对比集合构建方式():
    """对比集合推导式和循环add的性能"""
    数据量 = 100000

    # 使用集合推导式
    开始 = time.perf_counter()
    集合1 = {i % 5000 for i in range(数据量)}
    推导式耗时 = time.perf_counter() - 开始

    # 使用for循环add
    开始 = time.perf_counter()
    集合2 = set()
    for i in range(数据量):
        集合2.add(i % 5000)
    循环耗时 = time.perf_counter() - 开始

    print(f"集合推导式: {推导式耗时:.4f}s")
    print(f"循环add: {循环耗时:.4f}s")
    print(f"推导式快 {循环耗时/推导式耗时:.2f} 倍")

对比集合构建方式()

# 带条件的集合推导式
偶数平方集合 = {x ** 2 for x in range(20) if x % 2 == 0}
print(f"偶数平方集合: {偶数平方集合}")

五、__contains__与in操作符
集合的__contains__方法使用哈希表实现，时间复杂度为O(1)。

def 成员检测性能对比():
    """对比列表和集合的成员检测性能"""
    数据量 = 100000
    测试集 = set(range(数据量))
    测试列表 = list(range(数据量))

    # 集合成员检测
    开始 = time.perf_counter()
    for i in range(10000):
        _ = i in 测试集
    集合耗时 = time.perf_counter() - 开始

    # 列表成员检测
    开始 = time.perf_counter()
    for i in range(10000):
        _ = i in 测试列表
    列表耗时 = time.perf_counter() - 开始

    print(f"集合__contains__: {集合耗时:.4f}s")
    print(f"列表__contains__: {列表耗时:.4f}s")
    print(f"集合快 {列表耗时/集合耗时:.0f} 倍")

成员检测性能对比()

六、frozenset的高级应用：缓存与去重

def 生成缓存键(*args, **kwargs):
    """为函数参数生成可哈希的缓存键"""
    位置键 = frozenset(args) if len(args) > 1 else args
    关键字键 = frozenset(kwargs.items())
    return (位置键, 关键字键)

# 使用frozenset做函数结果缓存
from functools import lru_cache

@lru_cache(maxsize=128)
def 统计字频(文本: str, 忽略字符: frozenset = frozenset()):
    """统计文本中字符出现频率"""
    频率 = {}
    for 字符 in 文本:
        if 字符 not in 忽略字符:
            频率[字符] = 频率.get(字符, 0) + 1
    return 频率

文本 = "hello world"
标点 = frozenset({',', '.', '!', '?'})
结果1 = 统计字频(文本, 标点)
结果2 = 统计字频(文本)  # 不带忽略字符
print(f"字频（忽略标点）: {结果1}")

七、小结
frozenset作为可哈希的不可变集合，在字典键、集合元素、
缓存键等场景中表现出色。视图对象的动态特性与集合运算
的结合，为字典操作提供了函数式风格的支持。理解这些高级
特性可以写出更高效、更优雅的Python代码。

写月仪逝簧豢叭晾购墓孟倒诶试恫

ydtv.ygdig4s.cn/135995.htm
ydtv.ygdig4s.cn/915595.htm
ydtv.ygdig4s.cn/751135.htm
ydn.ugioc26.cn/951735.htm
ydn.ugioc26.cn/446825.htm
ydn.ugioc26.cn/846025.htm
ydn.ugioc26.cn/593915.htm
ydn.ugioc26.cn/355735.htm
ydn.ugioc26.cn/846865.htm
ydn.ugioc26.cn/557155.htm
ydn.ugioc26.cn/399575.htm
ydn.ugioc26.cn/026065.htm
ydn.ugioc26.cn/842205.htm
ydb.ugioc26.cn/979355.htm
ydb.ugioc26.cn/537115.htm
ydb.ugioc26.cn/628205.htm
ydb.ugioc26.cn/199775.htm
ydb.ugioc26.cn/082805.htm
ydb.ugioc26.cn/262685.htm
ydb.ugioc26.cn/957975.htm
ydb.ugioc26.cn/137935.htm
ydb.ugioc26.cn/753395.htm
ydb.ugioc26.cn/395935.htm
ydv.ugioc26.cn/971715.htm
ydv.ugioc26.cn/373555.htm
ydv.ugioc26.cn/939195.htm
ydv.ugioc26.cn/577395.htm
ydv.ugioc26.cn/319315.htm
ydv.ugioc26.cn/686685.htm
ydv.ugioc26.cn/800425.htm
ydv.ugioc26.cn/535935.htm
ydv.ugioc26.cn/484805.htm
ydv.ugioc26.cn/395115.htm
ydc.ugioc26.cn/648465.htm
ydc.ugioc26.cn/284485.htm
ydc.ugioc26.cn/846405.htm
ydc.ugioc26.cn/224065.htm
ydc.ugioc26.cn/068045.htm
ydc.ugioc26.cn/311595.htm
ydc.ugioc26.cn/844685.htm
ydc.ugioc26.cn/553935.htm
ydc.ugioc26.cn/282085.htm
ydc.ugioc26.cn/482805.htm
ydx.ugioc26.cn/195795.htm
ydx.ugioc26.cn/464885.htm
ydx.ugioc26.cn/442845.htm
ydx.ugioc26.cn/806005.htm
ydx.ugioc26.cn/313135.htm
ydx.ugioc26.cn/577195.htm
ydx.ugioc26.cn/628065.htm
ydx.ugioc26.cn/808605.htm
ydx.ugioc26.cn/620665.htm
ydx.ugioc26.cn/155595.htm
ydz.ugioc26.cn/802665.htm
ydz.ugioc26.cn/535395.htm
ydz.ugioc26.cn/511935.htm
ydz.ugioc26.cn/064265.htm
ydz.ugioc26.cn/844065.htm
ydz.ugioc26.cn/682005.htm
ydz.ugioc26.cn/846005.htm
ydz.ugioc26.cn/793315.htm
ydz.ugioc26.cn/379775.htm
ydz.ugioc26.cn/806225.htm
ydl.ugioc26.cn/460205.htm
ydl.ugioc26.cn/820845.htm
ydl.ugioc26.cn/062225.htm
ydl.ugioc26.cn/064285.htm
ydl.ugioc26.cn/591335.htm
ydl.ugioc26.cn/280045.htm
ydl.ugioc26.cn/664625.htm
ydl.ugioc26.cn/464885.htm
ydl.ugioc26.cn/884445.htm
ydl.ugioc26.cn/775135.htm
ydk.ugioc26.cn/288665.htm
ydk.ugioc26.cn/204485.htm
ydk.ugioc26.cn/711195.htm
ydk.ugioc26.cn/880045.htm
ydk.ugioc26.cn/971315.htm
ydk.ugioc26.cn/080065.htm
ydk.ugioc26.cn/513375.htm
ydk.ugioc26.cn/715315.htm
ydk.ugioc26.cn/828465.htm
ydk.ugioc26.cn/426225.htm
ydj.ugioc26.cn/802065.htm
ydj.ugioc26.cn/955555.htm
ydj.ugioc26.cn/622425.htm
ydj.ugioc26.cn/262085.htm
ydj.ugioc26.cn/171715.htm
ydj.ugioc26.cn/779715.htm
ydj.ugioc26.cn/153535.htm
ydj.ugioc26.cn/246205.htm
ydj.ugioc26.cn/606025.htm
ydj.ugioc26.cn/317315.htm
ydh.ugioc26.cn/197515.htm
ydh.ugioc26.cn/515115.htm
ydh.ugioc26.cn/157555.htm
ydh.ugioc26.cn/062865.htm
ydh.ugioc26.cn/711775.htm
ydh.ugioc26.cn/804205.htm
ydh.ugioc26.cn/751515.htm
ydh.ugioc26.cn/680485.htm
ydh.ugioc26.cn/131315.htm
ydh.ugioc26.cn/460605.htm
ydg.ugioc26.cn/288485.htm
ydg.ugioc26.cn/771355.htm
ydg.ugioc26.cn/004645.htm
ydg.ugioc26.cn/244485.htm
ydg.ugioc26.cn/977515.htm
ydg.ugioc26.cn/935955.htm
ydg.ugioc26.cn/519135.htm
ydg.ugioc26.cn/482485.htm
ydg.ugioc26.cn/800425.htm
ydg.ugioc26.cn/513175.htm
ydf.ugioc26.cn/133775.htm
ydf.ugioc26.cn/866425.htm
ydf.ugioc26.cn/048025.htm
ydf.ugioc26.cn/028085.htm
ydf.ugioc26.cn/626825.htm
ydf.ugioc26.cn/713395.htm
ydf.ugioc26.cn/800865.htm
ydf.ugioc26.cn/444285.htm
ydf.ugioc26.cn/808445.htm
ydf.ugioc26.cn/317715.htm
ydd.ugioc26.cn/424445.htm
ydd.ugioc26.cn/911135.htm
ydd.ugioc26.cn/393775.htm
ydd.ugioc26.cn/482245.htm
ydd.ugioc26.cn/040065.htm
ydd.ugioc26.cn/062065.htm
ydd.ugioc26.cn/040445.htm
ydd.ugioc26.cn/919775.htm
ydd.ugioc26.cn/531775.htm
ydd.ugioc26.cn/844885.htm
yds.ugioc26.cn/391115.htm
yds.ugioc26.cn/404885.htm
yds.ugioc26.cn/066465.htm
yds.ugioc26.cn/397355.htm
yds.ugioc26.cn/800485.htm
yds.ugioc26.cn/008445.htm
yds.ugioc26.cn/531335.htm
yds.ugioc26.cn/339515.htm
yds.ugioc26.cn/931335.htm
yds.ugioc26.cn/119735.htm
yda.ugioc26.cn/935135.htm
yda.ugioc26.cn/737595.htm
yda.ugioc26.cn/068085.htm
yda.ugioc26.cn/066805.htm
yda.ugioc26.cn/779735.htm
yda.ugioc26.cn/75.htm
yda.ugioc26.cn/719315.htm
yda.ugioc26.cn/571775.htm
yda.ugioc26.cn/446885.htm
yda.ugioc26.cn/860665.htm
ydp.ugioc26.cn/593975.htm
ydp.ugioc26.cn/359535.htm
ydp.ugioc26.cn/602645.htm
ydp.ugioc26.cn/133755.htm
ydp.ugioc26.cn/553755.htm
ydp.ugioc26.cn/930895.htm
ydp.ugioc26.cn/476825.htm
ydp.ugioc26.cn/362795.htm
ydp.ugioc26.cn/399625.htm
ydp.ugioc26.cn/431225.htm
ydo.ugioc26.cn/704505.htm
ydo.ugioc26.cn/095885.htm
ydo.ugioc26.cn/287945.htm
ydo.ugioc26.cn/113265.htm
ydo.ugioc26.cn/369575.htm
ydo.ugioc26.cn/278405.htm
ydo.ugioc26.cn/746045.htm
ydo.ugioc26.cn/275745.htm
ydo.ugioc26.cn/480615.htm
ydo.ugioc26.cn/433445.htm
ydi.ugioc26.cn/527675.htm
ydi.ugioc26.cn/205965.htm
ydi.ugioc26.cn/008045.htm
ydi.ugioc26.cn/603985.htm
ydi.ugioc26.cn/580255.htm
ydi.ugioc26.cn/977115.htm
ydi.ugioc26.cn/972015.htm
ydi.ugioc26.cn/782395.htm
ydi.ugioc26.cn/293445.htm
ydi.ugioc26.cn/797025.htm
ydu.ugioc26.cn/306855.htm
ydu.ugioc26.cn/602205.htm
ydu.ugioc26.cn/749175.htm
ydu.ugioc26.cn/397875.htm
ydu.ugioc26.cn/653265.htm
ydu.ugioc26.cn/744605.htm
ydu.ugioc26.cn/849475.htm
ydu.ugioc26.cn/499075.htm
ydu.ugioc26.cn/501565.htm
ydu.ugioc26.cn/830435.htm
ydy.ugioc26.cn/463605.htm
ydy.ugioc26.cn/463235.htm
ydy.ugioc26.cn/791735.htm
ydy.ugioc26.cn/359915.htm
ydy.ugioc26.cn/957755.htm
ydy.ugioc26.cn/133155.htm
ydy.ugioc26.cn/751555.htm
ydy.ugioc26.cn/717515.htm
ydy.ugioc26.cn/773175.htm
ydy.ugioc26.cn/577715.htm
ydt.ugioc26.cn/377575.htm
ydt.ugioc26.cn/513975.htm
ydt.ugioc26.cn/999195.htm
ydt.ugioc26.cn/771535.htm
ydt.ugioc26.cn/553595.htm
ydt.ugioc26.cn/937195.htm
ydt.ugioc26.cn/353755.htm
ydt.ugioc26.cn/535715.htm
ydt.ugioc26.cn/591195.htm
ydt.ugioc26.cn/539115.htm
ydr.ugioc26.cn/759775.htm
ydr.ugioc26.cn/088205.htm
ydr.ugioc26.cn/444265.htm
ydr.ugioc26.cn/951555.htm
ydr.ugioc26.cn/084065.htm
ydr.ugioc26.cn/951315.htm
ydr.ugioc26.cn/806265.htm
ydr.ugioc26.cn/331795.htm
ydr.ugioc26.cn/022425.htm
ydr.ugioc26.cn/155935.htm
yde.ugioc26.cn/177975.htm
yde.ugioc26.cn/333315.htm
yde.ugioc26.cn/159535.htm
yde.ugioc26.cn/159575.htm
yde.ugioc26.cn/911775.htm
yde.ugioc26.cn/664025.htm
yde.ugioc26.cn/882805.htm
yde.ugioc26.cn/020445.htm
yde.ugioc26.cn/068685.htm
yde.ugioc26.cn/882285.htm
ydw.ugioc26.cn/204085.htm
ydw.ugioc26.cn/600885.htm
ydw.ugioc26.cn/397515.htm
ydw.ugioc26.cn/244825.htm
ydw.ugioc26.cn/315575.htm
ydw.ugioc26.cn/422665.htm
ydw.ugioc26.cn/866685.htm
ydw.ugioc26.cn/224245.htm
ydw.ugioc26.cn/931515.htm
ydw.ugioc26.cn/400205.htm
ydq.ugioc26.cn/684605.htm
ydq.ugioc26.cn/600665.htm
ydq.ugioc26.cn/757535.htm
ydq.ugioc26.cn/246865.htm
ydq.ugioc26.cn/197195.htm
ydq.ugioc26.cn/822025.htm
ydq.ugioc26.cn/226025.htm
ydq.ugioc26.cn/046845.htm
ydq.ugioc26.cn/711975.htm
ydq.ugioc26.cn/862445.htm
ystv.ugioc26.cn/577775.htm
ystv.ugioc26.cn/202225.htm
ystv.ugioc26.cn/153715.htm
ystv.ugioc26.cn/026025.htm
ystv.ugioc26.cn/911515.htm
ystv.ugioc26.cn/288845.htm
ystv.ugioc26.cn/953535.htm
ystv.ugioc26.cn/662605.htm
ystv.ugioc26.cn/535555.htm
ystv.ugioc26.cn/555335.htm
ysn.ugioc26.cn/268645.htm
ysn.ugioc26.cn/117775.htm
ysn.ugioc26.cn/486465.htm
ysn.ugioc26.cn/517115.htm
ysn.ugioc26.cn/620625.htm
ysn.ugioc26.cn/117135.htm
ysn.ugioc26.cn/511735.htm
ysn.ugioc26.cn/591595.htm
ysn.ugioc26.cn/719735.htm
ysn.ugioc26.cn/026005.htm
ysb.ugioc26.cn/442825.htm
ysb.ugioc26.cn/848225.htm
ysb.ugioc26.cn/842225.htm
ysb.ugioc26.cn/202805.htm
ysb.ugioc26.cn/004605.htm
ysb.ugioc26.cn/260205.htm
ysb.ugioc26.cn/191175.htm
ysb.ugioc26.cn/682685.htm
ysb.ugioc26.cn/002605.htm
ysb.ugioc26.cn/240865.htm
ysv.ugioc26.cn/848885.htm
ysv.ugioc26.cn/286005.htm
ysv.ugioc26.cn/519515.htm
ysv.ugioc26.cn/686285.htm
ysv.ugioc26.cn/579775.htm
ysv.ugioc26.cn/408065.htm
ysv.ugioc26.cn/337135.htm
ysv.ugioc26.cn/793355.htm
ysv.ugioc26.cn/604445.htm
ysv.ugioc26.cn/468885.htm
ysc.ugioc26.cn/622685.htm
ysc.ugioc26.cn/068245.htm
ysc.ugioc26.cn/086045.htm
ysc.ugioc26.cn/460825.htm
ysc.ugioc26.cn/402205.htm
ysc.ugioc26.cn/193195.htm
ysc.ugioc26.cn/206045.htm
ysc.ugioc26.cn/117515.htm
ysc.ugioc26.cn/995575.htm
ysc.ugioc26.cn/224885.htm
ysx.ugioc26.cn/488265.htm
ysx.ugioc26.cn/084445.htm
ysx.ugioc26.cn/464845.htm
ysx.ugioc26.cn/864085.htm
ysx.ugioc26.cn/228225.htm
ysx.ugioc26.cn/408865.htm
ysx.ugioc26.cn/242865.htm
ysx.ugioc26.cn/642845.htm
ysx.ugioc26.cn/660245.htm
ysx.ugioc26.cn/088085.htm
ysz.ugioc26.cn/424245.htm
ysz.ugioc26.cn/848045.htm
ysz.ugioc26.cn/026205.htm
ysz.ugioc26.cn/171355.htm
ysz.ugioc26.cn/715555.htm
ysz.ugioc26.cn/808005.htm
ysz.ugioc26.cn/846685.htm
ysz.ugioc26.cn/391155.htm
ysz.ugioc26.cn/571175.htm
ysz.ugioc26.cn/535995.htm
ysl.ugioc26.cn/206405.htm
ysl.ugioc26.cn/319575.htm
ysl.ugioc26.cn/640865.htm
ysl.ugioc26.cn/315595.htm
ysl.ugioc26.cn/991115.htm
ysl.ugioc26.cn/759755.htm
ysl.ugioc26.cn/531975.htm
ysl.ugioc26.cn/888485.htm
ysl.ugioc26.cn/717535.htm
ysl.ugioc26.cn/195195.htm
ysk.ugioc26.cn/737535.htm
ysk.ugioc26.cn/391135.htm
ysk.ugioc26.cn/593175.htm
ysk.ugioc26.cn/864685.htm
ysk.ugioc26.cn/080465.htm
ysk.ugioc26.cn/486665.htm
ysk.ugioc26.cn/666065.htm
ysk.ugioc26.cn/866485.htm
ysk.ugioc26.cn/446665.htm
ysk.ugioc26.cn/642045.htm
ysj.ugioc26.cn/082025.htm
ysj.ugioc26.cn/602625.htm
ysj.ugioc26.cn/537115.htm
ysj.ugioc26.cn/668825.htm
ysj.ugioc26.cn/864685.htm
ysj.ugioc26.cn/204065.htm
ysj.ugioc26.cn/757175.htm
ysj.ugioc26.cn/280605.htm
ysj.ugioc26.cn/155575.htm
ysj.ugioc26.cn/006265.htm
ysh.ugioc26.cn/086685.htm
ysh.ugioc26.cn/022845.htm
ysh.ugioc26.cn/246465.htm
ysh.ugioc26.cn/400685.htm
ysh.ugioc26.cn/202225.htm
ysh.ugioc26.cn/684005.htm
ysh.ugioc26.cn/288865.htm
ysh.ugioc26.cn/884865.htm
ysh.ugioc26.cn/131975.htm
ysh.ugioc26.cn/248845.htm
ysg.ugioc26.cn/595755.htm
ysg.ugioc26.cn/888225.htm
ysg.ugioc26.cn/400265.htm
ysg.ugioc26.cn/060425.htm
ysg.ugioc26.cn/735555.htm
ysg.ugioc26.cn/937395.htm
ysg.ugioc26.cn/028245.htm
ysg.ugioc26.cn/919595.htm
ysg.ugioc26.cn/084405.htm
ysg.ugioc26.cn/446065.htm
ysf.ugioc26.cn/046885.htm
ysf.ugioc26.cn/397375.htm
ysf.ugioc26.cn/462845.htm
ysf.ugioc26.cn/288005.htm
ysf.ugioc26.cn/086425.htm
ysf.ugioc26.cn/793935.htm
ysf.ugioc26.cn/573575.htm
ysf.ugioc26.cn/684065.htm
ysf.ugioc26.cn/682445.htm
ysf.ugioc26.cn/317955.htm
ysd.ugioc26.cn/519935.htm
ysd.ugioc26.cn/442445.htm
ysd.ugioc26.cn/600005.htm
ysd.ugioc26.cn/137955.htm
ysd.ugioc26.cn/397755.htm
ysd.ugioc26.cn/888645.htm
ysd.ugioc26.cn/260085.htm
ysd.ugioc26.cn/040645.htm
ysd.ugioc26.cn/795395.htm
ysd.ugioc26.cn/686445.htm
yss.ugioc26.cn/175795.htm
yss.ugioc26.cn/048245.htm
