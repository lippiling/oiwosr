股欧勇薪拾


Python生成器表达式深度解析
==============================

一、生成器表达式内部机制
生成器表达式是惰性求值的迭代器，与列表推导式有本质区别。
它在迭代时才逐个产生值，不会一次性创建整个序列。

import sys
import time
import memory_profiler  # 可选，仅用于演示

# 基本语法对比
列表结果 = [x ** 2 for x in range(10)]   # 列表推导式：立即求值
生成器结果 = (x ** 2 for x in range(10))  # 生成器表达式：惰性求值

print(f"列表推导式类型: {type(列表结果)}")
print(f"生成器表达式类型: {type(生成器结果)}")

# 生成器表达式本质上是简化的生成器函数
def 等价生成器函数():
    for x in range(10):
        yield x ** 2

# 逐值获取
print("生成器输出:", end=" ")
for 值 in 生成器结果:
    print(值, end=" ")
print()

二、内存消耗对比
生成器表达式最大的优势在于内存效率，尤其处理大数据时。

# 查看对象大小
大列表 = [x for x in range(100000)]
大生成器 = (x for x in range(100000))

print(f"列表大小: {sys.getsizeof(大列表)} 字节")
print(f"生成器大小: {sys.getsizeof(大生成器)} 字节")
# 生成器的大小固定，不随数据量增长

# 内存占用对比演示
def 内存对比():
    列表数据 = [i * 2 for i in range(1000000)]
    print(f"列表内存: {sys.getsizeof(列表数据) / 1024 / 1024:.2f} MB")

    生成器数据 = (i * 2 for i in range(1000000))
    print(f"生成器内存: {sys.getsizeof(生成器数据)} 字节")

内存对比()

三、嵌套生成器表达式
生成器表达式可以嵌套使用，形成管道式数据处理。

# 三层嵌套：读取 → 过滤 → 转换
原始数据 = range(100)
过滤后 = (x for x in 原始数据 if x % 2 == 0)       # 偶数
转换后 = (x * 10 for x in 过滤后)                   # 扩大
聚合后 = (x + 1 for x in 转换后)                     # 偏移

# 只有遍历时才真正执行计算
结果列表 = list(聚合后)
print(f"嵌套管道结果（前10个）: {结果列表[:10]}")

# 等价于链式调用
链式结果 = list((x + 1 for x in (x * 10 for x in (x for x in range(100) if x % 2 == 0))))
print(f"链式结果（前10个）: {链式结果[:10]}")

四、生成器表达式作为函数参数
当生成器表达式作为唯一的函数参数时，可以省略一层括号。

def 处理数据(迭代器):
    """处理迭代数据"""
    return sum(迭代器)

# 标准写法
结果1 = 处理数据((x ** 2 for x in range(10)))

# 省略括号的简洁写法（只有一个参数时）
结果2 = 处理数据(x ** 2 for x in range(10))

print(f"求和结果: {结果1}")

# 多个参数时必须加括号
def 分析数据(数据源, 转换函数):
    return [转换函数(x) for x in 数据源]

# 必须加括号的情况
结果3 = 分析数据((x * 2 for x in range(5)), lambda x: x + 1)
print(f"多参情况: {结果3}")

五、any/all与生成器表达式的短路优化
any和all配合生成器表达式可以实现短路求值，提前终止。

def 耗时检查(n):
    """模拟耗时检查函数"""
    print(f"  检查 {n}")
    return n > 5

# 列表推导式：全部计算完才判断
print("列表推导式 + any（非短路）:")
结果列表式 = any([耗时检查(x) for x in [1, 2, 3, 6, 7, 8]])
print(f"结果: {结果列表式}")

# 生成器表达式：短路求值
print("\n生成器表达式 + any（短路）:")
结果生成器式 = any(耗时检查(x) for x in [1, 2, 3, 6, 7, 8])
print(f"结果: {结果生成器式}")

六、生成器表达式与列表推导式性能对比

def 性能对比():
    """对比生成器表达式和列表推导式的执行时间"""
    import timeit

    # 列表推导式：一次性创建所有数据
    列表时间 = timeit.timeit(
        'sum([x * x for x in range(1000)])',
        number=10000
    )

    # 生成器表达式：逐个求值
    生成器时间 = timeit.timeit(
        'sum(x * x for x in range(1000))',
        number=10000
    )

    print(f"列表推导式总耗时: {列表时间:.4f}s")
    print(f"生成器表达式总耗时: {生成器时间:.4f}s")
    print(f"生成器快 {列表时间/生成器时间:.2f} 倍")

性能对比()

七、生成器表达式的关闭与异常
生成器表达式支持close()方法，可以提前终止迭代。

def 监控生成器():
    生成器 = (i for i in range(1000))
    for idx, 值 in enumerate(生成器):
        print(f"取值: {值}")
        if idx >= 2:
            生成器.close()  # 提前关闭
            print("生成器已关闭")
            break
    # 关闭后再取值
    try:
        next(生成器)
    except StopIteration:
        print("已停止迭代")

监控生成器()

八、小结
生成器表达式以极小的内存开销实现了流式数据处理，适合大数据集、
无限序列、管道式数据处理等场景。结合any/all等短路函数使用时
能进一步提升性能。但需要多次遍历时，应转为列表。

澈抢掩姑鹊呈都膛油永握箍乘自桃

evl.mmmxz.cn/024043.htm
evl.mmmxz.cn/862263.htm
evl.mmmxz.cn/733753.htm
evl.mmmxz.cn/353913.htm
evl.mmmxz.cn/666423.htm
evl.mmmxz.cn/482263.htm
evl.mmmxz.cn/997353.htm
evl.mmmxz.cn/664223.htm
evl.mmmxz.cn/713353.htm
evl.mmmxz.cn/711153.htm
evk.mmmxz.cn/931173.htm
evk.mmmxz.cn/486283.htm
evk.mmmxz.cn/202263.htm
evk.mmmxz.cn/553513.htm
evk.mmmxz.cn/979993.htm
evk.mmmxz.cn/620623.htm
evk.mmmxz.cn/846063.htm
evk.mmmxz.cn/820403.htm
evk.mmmxz.cn/822843.htm
evk.mmmxz.cn/846803.htm
evj.mmmxz.cn/600883.htm
evj.mmmxz.cn/793553.htm
evj.mmmxz.cn/755953.htm
evj.mmmxz.cn/620243.htm
evj.mmmxz.cn/755593.htm
evj.mmmxz.cn/553753.htm
evj.mmmxz.cn/248063.htm
evj.mmmxz.cn/597313.htm
evj.mmmxz.cn/713953.htm
evj.mmmxz.cn/555793.htm
evh.mmmxz.cn/482483.htm
evh.mmmxz.cn/757113.htm
evh.mmmxz.cn/402883.htm
evh.mmmxz.cn/860043.htm
evh.mmmxz.cn/866643.htm
evh.mmmxz.cn/771113.htm
evh.mmmxz.cn/408803.htm
evh.mmmxz.cn/468043.htm
evh.mmmxz.cn/466483.htm
evh.mmmxz.cn/975133.htm
evg.mmmxz.cn/959593.htm
evg.mmmxz.cn/826623.htm
evg.mmmxz.cn/002003.htm
evg.mmmxz.cn/666663.htm
evg.mmmxz.cn/151113.htm
evg.mmmxz.cn/319353.htm
evg.mmmxz.cn/913333.htm
evg.mmmxz.cn/426463.htm
evg.mmmxz.cn/666443.htm
evg.mmmxz.cn/937573.htm
evf.mmmxz.cn/802283.htm
evf.mmmxz.cn/488043.htm
evf.mmmxz.cn/139153.htm
evf.mmmxz.cn/426863.htm
evf.mmmxz.cn/577773.htm
evf.mmmxz.cn/844603.htm
evf.mmmxz.cn/466083.htm
evf.mmmxz.cn/197953.htm
evf.mmmxz.cn/642023.htm
evf.mmmxz.cn/026643.htm
evd.mmmxz.cn/240283.htm
evd.mmmxz.cn/884043.htm
evd.mmmxz.cn/868283.htm
evd.mmmxz.cn/646863.htm
evd.mmmxz.cn/537993.htm
evd.mmmxz.cn/046223.htm
evd.mmmxz.cn/884063.htm
evd.mmmxz.cn/535173.htm
evd.mmmxz.cn/559993.htm
evd.mmmxz.cn/555793.htm
evs.mmmxz.cn/137993.htm
evs.mmmxz.cn/480003.htm
evs.mmmxz.cn/971153.htm
evs.mmmxz.cn/933153.htm
evs.mmmxz.cn/997313.htm
evs.mmmxz.cn/153393.htm
evs.mmmxz.cn/517393.htm
evs.mmmxz.cn/602223.htm
evs.mmmxz.cn/753393.htm
evs.mmmxz.cn/713173.htm
eva.mmmxz.cn/991753.htm
eva.mmmxz.cn/664223.htm
eva.mmmxz.cn/913573.htm
eva.mmmxz.cn/424083.htm
eva.mmmxz.cn/191573.htm
eva.mmmxz.cn/597973.htm
eva.mmmxz.cn/862803.htm
eva.mmmxz.cn/559313.htm
eva.mmmxz.cn/937373.htm
eva.mmmxz.cn/595333.htm
evp.mmmxz.cn/606803.htm
evp.mmmxz.cn/420043.htm
evp.mmmxz.cn/404663.htm
evp.mmmxz.cn/486683.htm
evp.mmmxz.cn/408263.htm
evp.mmmxz.cn/228083.htm
evp.mmmxz.cn/715313.htm
evp.mmmxz.cn/319573.htm
evp.mmmxz.cn/242803.htm
evp.mmmxz.cn/519733.htm
evo.mmmxz.cn/595553.htm
evo.mmmxz.cn/688043.htm
evo.mmmxz.cn/753193.htm
evo.mmmxz.cn/315173.htm
evo.mmmxz.cn/137713.htm
evo.mmmxz.cn/159373.htm
evo.mmmxz.cn/155153.htm
evo.mmmxz.cn/888843.htm
evo.mmmxz.cn/775173.htm
evo.mmmxz.cn/660843.htm
evi.mmmxz.cn/062423.htm
evi.mmmxz.cn/597333.htm
evi.mmmxz.cn/604803.htm
evi.mmmxz.cn/793793.htm
evi.mmmxz.cn/888483.htm
evi.mmmxz.cn/397173.htm
evi.mmmxz.cn/660883.htm
evi.mmmxz.cn/860823.htm
evi.mmmxz.cn/935553.htm
evi.mmmxz.cn/862063.htm
evu.mmmxz.cn/448603.htm
evu.mmmxz.cn/442443.htm
evu.mmmxz.cn/842243.htm
evu.mmmxz.cn/775793.htm
evu.mmmxz.cn/686283.htm
evu.mmmxz.cn/775713.htm
evu.mmmxz.cn/337373.htm
evu.mmmxz.cn/204683.htm
evu.mmmxz.cn/339713.htm
evu.mmmxz.cn/577533.htm
evy.mmmxz.cn/757933.htm
evy.mmmxz.cn/486643.htm
evy.mmmxz.cn/800663.htm
evy.mmmxz.cn/575993.htm
evy.mmmxz.cn/288443.htm
evy.mmmxz.cn/975313.htm
evy.mmmxz.cn/086443.htm
evy.mmmxz.cn/886603.htm
evy.mmmxz.cn/331333.htm
evy.mmmxz.cn/266603.htm
evt.mmmxz.cn/917713.htm
evt.mmmxz.cn/422823.htm
evt.mmmxz.cn/660283.htm
evt.mmmxz.cn/175933.htm
evt.mmmxz.cn/680683.htm
evt.mmmxz.cn/840803.htm
evt.mmmxz.cn/555113.htm
evt.mmmxz.cn/260023.htm
evt.mmmxz.cn/682423.htm
evt.mmmxz.cn/220243.htm
evr.sthxr.cn/208023.htm
evr.sthxr.cn/266043.htm
evr.sthxr.cn/284843.htm
evr.sthxr.cn/082083.htm
evr.sthxr.cn/175193.htm
evr.sthxr.cn/000043.htm
evr.sthxr.cn/391933.htm
evr.sthxr.cn/995513.htm
evr.sthxr.cn/155913.htm
evr.sthxr.cn/086063.htm
eve.sthxr.cn/644623.htm
eve.sthxr.cn/999713.htm
eve.sthxr.cn/979953.htm
eve.sthxr.cn/791533.htm
eve.sthxr.cn/597793.htm
eve.sthxr.cn/648223.htm
eve.sthxr.cn/755593.htm
eve.sthxr.cn/268043.htm
eve.sthxr.cn/773773.htm
eve.sthxr.cn/680063.htm
evw.sthxr.cn/511733.htm
evw.sthxr.cn/868223.htm
evw.sthxr.cn/155793.htm
evw.sthxr.cn/799113.htm
evw.sthxr.cn/717513.htm
evw.sthxr.cn/406663.htm
evw.sthxr.cn/864643.htm
evw.sthxr.cn/402223.htm
evw.sthxr.cn/060023.htm
evw.sthxr.cn/357713.htm
evq.sthxr.cn/420683.htm
evq.sthxr.cn/737733.htm
evq.sthxr.cn/400223.htm
evq.sthxr.cn/202403.htm
evq.sthxr.cn/004643.htm
evq.sthxr.cn/331513.htm
evq.sthxr.cn/044203.htm
evq.sthxr.cn/113793.htm
evq.sthxr.cn/240643.htm
evq.sthxr.cn/602863.htm
ecm.sthxr.cn/357393.htm
ecm.sthxr.cn/884843.htm
ecm.sthxr.cn/484663.htm
ecm.sthxr.cn/937793.htm
ecm.sthxr.cn/088683.htm
ecm.sthxr.cn/537533.htm
ecm.sthxr.cn/777393.htm
ecm.sthxr.cn/822803.htm
ecm.sthxr.cn/642423.htm
ecm.sthxr.cn/753973.htm
ecn.sthxr.cn/400863.htm
ecn.sthxr.cn/171973.htm
ecn.sthxr.cn/288463.htm
ecn.sthxr.cn/175513.htm
ecn.sthxr.cn/711733.htm
ecn.sthxr.cn/799933.htm
ecn.sthxr.cn/713993.htm
ecn.sthxr.cn/880063.htm
ecn.sthxr.cn/377333.htm
ecn.sthxr.cn/975513.htm
ecb.sthxr.cn/026683.htm
ecb.sthxr.cn/626263.htm
ecb.sthxr.cn/753773.htm
ecb.sthxr.cn/195533.htm
ecb.sthxr.cn/220643.htm
ecb.sthxr.cn/422003.htm
ecb.sthxr.cn/604643.htm
ecb.sthxr.cn/202823.htm
ecb.sthxr.cn/777993.htm
ecb.sthxr.cn/557913.htm
ecv.sthxr.cn/575953.htm
ecv.sthxr.cn/779553.htm
ecv.sthxr.cn/735553.htm
ecv.sthxr.cn/311793.htm
ecv.sthxr.cn/335153.htm
ecv.sthxr.cn/755773.htm
ecv.sthxr.cn/795173.htm
ecv.sthxr.cn/977773.htm
ecv.sthxr.cn/628643.htm
ecv.sthxr.cn/884023.htm
ecc.sthxr.cn/399313.htm
ecc.sthxr.cn/002843.htm
ecc.sthxr.cn/111773.htm
ecc.sthxr.cn/448623.htm
ecc.sthxr.cn/024683.htm
ecc.sthxr.cn/022863.htm
ecc.sthxr.cn/428803.htm
ecc.sthxr.cn/482063.htm
ecc.sthxr.cn/820203.htm
ecc.sthxr.cn/840843.htm
ecx.sthxr.cn/159153.htm
ecx.sthxr.cn/440263.htm
ecx.sthxr.cn/379573.htm
ecx.sthxr.cn/773373.htm
ecx.sthxr.cn/622823.htm
ecx.sthxr.cn/620823.htm
ecx.sthxr.cn/886463.htm
ecx.sthxr.cn/860283.htm
ecx.sthxr.cn/513173.htm
ecx.sthxr.cn/244083.htm
ecz.sthxr.cn/157353.htm
ecz.sthxr.cn/355513.htm
ecz.sthxr.cn/319353.htm
ecz.sthxr.cn/973313.htm
ecz.sthxr.cn/111753.htm
ecz.sthxr.cn/173773.htm
ecz.sthxr.cn/915353.htm
ecz.sthxr.cn/442883.htm
ecz.sthxr.cn/775313.htm
ecz.sthxr.cn/799573.htm
ecl.sthxr.cn/222483.htm
ecl.sthxr.cn/177593.htm
ecl.sthxr.cn/028483.htm
ecl.sthxr.cn/137793.htm
ecl.sthxr.cn/002003.htm
ecl.sthxr.cn/044043.htm
ecl.sthxr.cn/086803.htm
ecl.sthxr.cn/866223.htm
ecl.sthxr.cn/884063.htm
ecl.sthxr.cn/888483.htm
eck.sthxr.cn/622603.htm
eck.sthxr.cn/359713.htm
eck.sthxr.cn/953133.htm
eck.sthxr.cn/115173.htm
eck.sthxr.cn/137393.htm
eck.sthxr.cn/640863.htm
eck.sthxr.cn/844463.htm
eck.sthxr.cn/533353.htm
eck.sthxr.cn/337553.htm
eck.sthxr.cn/795173.htm
ecj.sthxr.cn/515353.htm
ecj.sthxr.cn/195313.htm
ecj.sthxr.cn/406083.htm
ecj.sthxr.cn/208063.htm
ecj.sthxr.cn/195133.htm
ecj.sthxr.cn/242423.htm
ecj.sthxr.cn/022463.htm
ecj.sthxr.cn/339573.htm
ecj.sthxr.cn/519913.htm
ecj.sthxr.cn/359393.htm
ech.sthxr.cn/688083.htm
ech.sthxr.cn/802083.htm
ech.sthxr.cn/240843.htm
ech.sthxr.cn/808063.htm
ech.sthxr.cn/179753.htm
ech.sthxr.cn/424403.htm
ech.sthxr.cn/448883.htm
ech.sthxr.cn/175793.htm
ech.sthxr.cn/953933.htm
ech.sthxr.cn/224683.htm
ecg.sthxr.cn/977373.htm
ecg.sthxr.cn/042403.htm
ecg.sthxr.cn/404683.htm
ecg.sthxr.cn/377333.htm
ecg.sthxr.cn/880403.htm
ecg.sthxr.cn/688403.htm
ecg.sthxr.cn/624463.htm
ecg.sthxr.cn/000043.htm
ecg.sthxr.cn/002663.htm
ecg.sthxr.cn/353173.htm
ecf.sthxr.cn/462223.htm
ecf.sthxr.cn/442003.htm
ecf.sthxr.cn/000823.htm
ecf.sthxr.cn/795173.htm
ecf.sthxr.cn/466223.htm
ecf.sthxr.cn/204483.htm
ecf.sthxr.cn/044623.htm
ecf.sthxr.cn/042223.htm
ecf.sthxr.cn/284403.htm
ecf.sthxr.cn/555173.htm
ecd.sthxr.cn/084203.htm
ecd.sthxr.cn/280043.htm
ecd.sthxr.cn/933733.htm
ecd.sthxr.cn/951953.htm
ecd.sthxr.cn/668463.htm
ecd.sthxr.cn/602403.htm
ecd.sthxr.cn/406043.htm
ecd.sthxr.cn/268023.htm
ecd.sthxr.cn/842883.htm
ecd.sthxr.cn/517513.htm
ecs.sthxr.cn/931173.htm
ecs.sthxr.cn/028203.htm
ecs.sthxr.cn/848063.htm
ecs.sthxr.cn/375573.htm
ecs.sthxr.cn/533533.htm
ecs.sthxr.cn/955753.htm
ecs.sthxr.cn/224083.htm
ecs.sthxr.cn/222283.htm
ecs.sthxr.cn/844663.htm
ecs.sthxr.cn/046283.htm
eca.sthxr.cn/668043.htm
eca.sthxr.cn/864843.htm
eca.sthxr.cn/028263.htm
eca.sthxr.cn/331593.htm
eca.sthxr.cn/006603.htm
eca.sthxr.cn/915993.htm
eca.sthxr.cn/860803.htm
eca.sthxr.cn/606883.htm
eca.sthxr.cn/448843.htm
eca.sthxr.cn/935313.htm
ecp.sthxr.cn/971553.htm
ecp.sthxr.cn/591353.htm
ecp.sthxr.cn/517973.htm
ecp.sthxr.cn/951773.htm
ecp.sthxr.cn/244603.htm
ecp.sthxr.cn/404463.htm
ecp.sthxr.cn/222023.htm
ecp.sthxr.cn/377333.htm
ecp.sthxr.cn/151933.htm
ecp.sthxr.cn/282003.htm
eco.sthxr.cn/137193.htm
eco.sthxr.cn/311773.htm
eco.sthxr.cn/179353.htm
eco.sthxr.cn/993593.htm
eco.sthxr.cn/973133.htm
eco.sthxr.cn/535733.htm
eco.sthxr.cn/515593.htm
eco.sthxr.cn/933973.htm
eco.sthxr.cn/951133.htm
eco.sthxr.cn/975573.htm
eci.sthxr.cn/599773.htm
eci.sthxr.cn/173913.htm
eci.sthxr.cn/779373.htm
eci.sthxr.cn/808863.htm
eci.sthxr.cn/600263.htm
eci.sthxr.cn/991993.htm
eci.sthxr.cn/644483.htm
eci.sthxr.cn/026043.htm
eci.sthxr.cn/286623.htm
eci.sthxr.cn/422003.htm
ecu.sthxr.cn/282683.htm
ecu.sthxr.cn/008003.htm
ecu.sthxr.cn/440663.htm
ecu.sthxr.cn/604063.htm
ecu.sthxr.cn/375513.htm
ecu.sthxr.cn/373133.htm
ecu.sthxr.cn/266643.htm
ecu.sthxr.cn/244083.htm
ecu.sthxr.cn/808863.htm
ecu.sthxr.cn/866263.htm
ecy.sthxr.cn/119553.htm
ecy.sthxr.cn/462083.htm
ecy.sthxr.cn/159553.htm
ecy.sthxr.cn/424263.htm
ecy.sthxr.cn/519933.htm
