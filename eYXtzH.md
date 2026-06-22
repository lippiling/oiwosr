释懒分邓珊


===== Python 火焰图分析 =====
===== 使用 py-spy 生成火焰图定位 CPU 热点 =====

"""
火焰图 (Flame Graph) 是一种可视化 CPU 时间分布的工具。
py-spy 是一个采样 profiler，无需修改代码即可生成火焰图。

安装: pip install py-spy
"""

import time
import random
import math


# ===== 1. 模拟 CPU 密集型任务 =====

def calculate_primes(limit: int) -> list:
    """
    计算素数，典型的 CPU 密集操作。
    这是火焰图中应当出现的 "热点" 函数。
    """
    primes = []
    for num in range(2, limit):
        is_prime = True
        for divisor in range(2, int(num ** 0.5) + 1):
            if num % divisor == 0:
                is_prime = False
                break
        if is_prime:
            primes.append(num)
    return primes


# ===== 2. 模拟 IO 密集型与 CPU 混合任务 =====

def process_data_chunk(chunk_id: int, size: int) -> float:
    """
    处理数据块，包含 CPU 计算和模拟 IO 等待。
    """
    time.sleep(0.005)                    # 模拟 IO 等待
    result = 0.0
    for i in range(size):
        result += math.sin(i) * math.cos(i)
        result += math.sqrt(i + 1)
    return result


def parallel_simulation(workers: int):
    """
    模拟多任务并行处理，每个任务独立计算。
    """
    results = []
    for wid in range(workers):
        res = process_data_chunk(wid, 5000)
        results.append(res)
    return sum(results)


# ===== 3. 递归函数调用分析 =====

def recursive_compute(n: int, depth: int = 0) -> int:
    """
    递归计算函数，火焰图会显示深层调用栈。
    """
    if depth > 10:
        return n
    # 每一层做一些计算
    total = 0
    for i in range(n):
        total += i * i
    # 递归调用
    return total + recursive_compute(n // 2, depth + 1)


# ===== 4. 复杂数据处理管道 =====

def transform_step(data: list) -> list:
    """管道第一步：数学变换"""
    return [x ** 2 - 3 * x + 1 for x in data]


def filter_step(data: list) -> list:
    """管道第二步：条件过滤"""
    return [x for x in data if x > 100]


def sort_step(data: list) -> list:
    """管道第三步：排序"""
    return sorted(data, reverse=True)


def pipeline(data: list) -> list:
    """
    完整数据处理管道。
    火焰图中可以看到每一步的耗时占比。
    """
    step1 = transform_step(data)
    step2 = filter_step(step1)
    step3 = sort_step(step2)
    return step3


# ===== 5. 长时间运行的主逻辑 =====

def heavy_workload():
    """
    主工作负载函数，火焰图的核心分析目标。
    它组合调用多个子函数，形成调用栈层次。
    """
    print("[WORK] 开始繁重计算任务...")

    # 任务 1：素数计算（CPU 密集）
    primes = calculate_primes(5000)
    print(f"[WORK] 找到 {len(primes)} 个素数")

    # 任务 2：并行模拟（混合型）
    sim_result = parallel_simulation(10)
    print(f"[WORK] 模拟结果: {sim_result:.2f}")

    # 任务 3：递归计算
    rec_result = recursive_compute(100)
    print(f"[WORK] 递归结果: {rec_result}")

    # 任务 4：数据处理管道
    raw_data = [random.randint(1, 1000) for _ in range(10000)]
    processed = pipeline(raw_data)
    print(f"[WORK] 管道处理了 {len(processed)} 个元素")


def background_overhead():
    """
    后台额外开销函数，模拟不定时触发的小任务。
    这类函数在火焰图中呈现为较窄的柱子。
    """
    for _ in range(50):
        _ = [math.sqrt(random.random()) for __ in range(200)]
        time.sleep(0.001)


# ===== 6. py-spy 使用方式 =====

"""
py-spy 的常用命令:

# 1. 实时采样并生成火焰图 SVG
py-spy record -o flamegraph.svg --duration 30 -- python 本文件.py

# 2. 实时查看调用栈（top 风格）
py-spy top -- python 本文件.py

# 3. 附加到正在运行的进程
py-spy record -o flame.svg --pid 12345

# 4. 指定采样频率（默认 100，即每秒 100 次）
py-spy record -o flame.svg --rate 50 -- python 本文件.py

# 5. 只分析特定线程
py-spy record -o flame.svg --subprocesses -- python 本文件.py

火焰图解读:
    - X 轴: 采样总数，不代表时间流逝
    - Y 轴: 调用栈深度，顶部是实际执行函数
    - 宽度: 函数占用 CPU 时间的比例
    - 颜色: 通常随机分配，无特殊含义
    - 从下往上看: 程序的执行路径

关键分析方法:
    1. 找最宽的 "平顶": 这是最耗 CPU 的热点函数
    2. 看调用路径: 追溯宽条是如何被调用的
    3. 山峰的形状:
       - 平坦宽阔: 循环或重复调用
       - 高耸细窄: 深递归或深层调用链
       - 锯齿状: 频繁的上下文切换
    4. 关注那些出乎意料的宽条，它们可能是性能问题
"""


# ===== 7. 使用 py-spy 记录采样到文件 =====

def save_sampling_data():
    """
    在被测脚本内部输出提示信息。
    实际采样由 py-spy record 命令在外部完成。
    """
    print("[PY-SPY] py-spy 正在采样中，请等待完成...")
    print("[PY-SPY] 采样结束后将生成 SVG 火焰图")
    print("[PY-SPY] 可以用浏览器直接打开 SVG 文件查看")


# ===== 主入口 =====

if __name__ == '__main__':
    import sys

    print("=" * 50)
    print("py-spy 火焰图分析示例")
    print("使用方法: py-spy record -o flame.svg -d 20 -- python 本文件")
    print("=" * 50)

    save_sampling_data()

    # 运行主工作负载
    heavy_workload()

    # 一些后台开销
    background_overhead()

    print("[DONE] 所有任务完成，请查看生成的火焰图 SVG 文件")

韧醒老绞缀档拔素旅示钙交剿闲辈

dzt.irvnp.cn/644288.Doc
dzt.irvnp.cn/138014.Doc
dzt.irvnp.cn/382586.Doc
dzt.irvnp.cn/264922.Doc
dzt.irvnp.cn/595547.Doc
dzr.irvnp.cn/472409.Doc
dzr.irvnp.cn/175975.Doc
dzr.irvnp.cn/226224.Doc
dzr.irvnp.cn/183682.Doc
dzr.irvnp.cn/319791.Doc
dzr.irvnp.cn/599914.Doc
dzr.irvnp.cn/830412.Doc
dzr.irvnp.cn/608424.Doc
dzr.irvnp.cn/919263.Doc
dzr.irvnp.cn/869028.Doc
dze.irvnp.cn/252413.Doc
dze.irvnp.cn/006468.Doc
dze.irvnp.cn/667389.Doc
dze.irvnp.cn/422060.Doc
dze.irvnp.cn/680284.Doc
dze.irvnp.cn/937599.Doc
dze.irvnp.cn/323543.Doc
dze.irvnp.cn/979537.Doc
dze.irvnp.cn/248204.Doc
dze.irvnp.cn/806642.Doc
dzw.irvnp.cn/488220.Doc
dzw.irvnp.cn/224866.Doc
dzw.irvnp.cn/173317.Doc
dzw.irvnp.cn/924985.Doc
dzw.irvnp.cn/882804.Doc
dzw.irvnp.cn/753179.Doc
dzw.irvnp.cn/181809.Doc
dzw.irvnp.cn/006848.Doc
dzw.irvnp.cn/380564.Doc
dzw.irvnp.cn/088844.Doc
dzq.irvnp.cn/042682.Doc
dzq.irvnp.cn/600804.Doc
dzq.irvnp.cn/880268.Doc
dzq.irvnp.cn/139553.Doc
dzq.irvnp.cn/026888.Doc
dzq.irvnp.cn/519391.Doc
dzq.irvnp.cn/137395.Doc
dzq.irvnp.cn/991319.Doc
dzq.irvnp.cn/020828.Doc
dzq.irvnp.cn/662844.Doc
dlm.irvnp.cn/620448.Doc
dlm.irvnp.cn/262626.Doc
dlm.irvnp.cn/228280.Doc
dlm.irvnp.cn/028282.Doc
dlm.irvnp.cn/359513.Doc
dlm.irvnp.cn/561530.Doc
dlm.irvnp.cn/604028.Doc
dlm.irvnp.cn/591313.Doc
dlm.irvnp.cn/284848.Doc
dlm.irvnp.cn/541797.Doc
dln.irvnp.cn/860828.Doc
dln.irvnp.cn/880226.Doc
dln.irvnp.cn/822066.Doc
dln.irvnp.cn/862882.Doc
dln.irvnp.cn/296545.Doc
dln.irvnp.cn/404640.Doc
dln.irvnp.cn/062886.Doc
dln.irvnp.cn/383858.Doc
dln.irvnp.cn/351571.Doc
dln.irvnp.cn/848484.Doc
dlb.irvnp.cn/866044.Doc
dlb.irvnp.cn/758451.Doc
dlb.irvnp.cn/627163.Doc
dlb.irvnp.cn/173771.Doc
dlb.irvnp.cn/884468.Doc
dlb.irvnp.cn/626488.Doc
dlb.irvnp.cn/693867.Doc
dlb.irvnp.cn/608408.Doc
dlb.irvnp.cn/799102.Doc
dlb.irvnp.cn/606088.Doc
dlv.irvnp.cn/335597.Doc
dlv.irvnp.cn/484062.Doc
dlv.irvnp.cn/951151.Doc
dlv.irvnp.cn/751171.Doc
dlv.irvnp.cn/084226.Doc
dlv.irvnp.cn/266000.Doc
dlv.irvnp.cn/406662.Doc
dlv.irvnp.cn/690700.Doc
dlv.irvnp.cn/658509.Doc
dlv.irvnp.cn/193171.Doc
dlc.irvnp.cn/464226.Doc
dlc.irvnp.cn/062828.Doc
dlc.irvnp.cn/686280.Doc
dlc.irvnp.cn/038161.Doc
dlc.irvnp.cn/535597.Doc
dlc.irvnp.cn/515791.Doc
dlc.irvnp.cn/646648.Doc
dlc.irvnp.cn/002246.Doc
dlc.irvnp.cn/222280.Doc
dlc.irvnp.cn/135759.Doc
dlx.irvnp.cn/320276.Doc
dlx.irvnp.cn/022220.Doc
dlx.irvnp.cn/400864.Doc
dlx.irvnp.cn/196027.Doc
dlx.irvnp.cn/615140.Doc
dlx.irvnp.cn/824682.Doc
dlx.irvnp.cn/624048.Doc
dlx.irvnp.cn/484268.Doc
dlx.irvnp.cn/626660.Doc
dlx.irvnp.cn/266844.Doc
dlz.irvnp.cn/200466.Doc
dlz.irvnp.cn/608066.Doc
dlz.irvnp.cn/088846.Doc
dlz.irvnp.cn/602048.Doc
dlz.irvnp.cn/820206.Doc
dlz.irvnp.cn/266228.Doc
dlz.irvnp.cn/842444.Doc
dlz.irvnp.cn/846440.Doc
dlz.irvnp.cn/640246.Doc
dlz.irvnp.cn/086282.Doc
dll.irvnp.cn/846648.Doc
dll.irvnp.cn/294685.Doc
dll.irvnp.cn/626004.Doc
dll.irvnp.cn/734208.Doc
dll.irvnp.cn/286680.Doc
dll.irvnp.cn/824240.Doc
dll.irvnp.cn/008660.Doc
dll.irvnp.cn/507961.Doc
dll.irvnp.cn/608804.Doc
dll.irvnp.cn/391715.Doc
dlk.irvnp.cn/658444.Doc
dlk.irvnp.cn/480420.Doc
dlk.irvnp.cn/844280.Doc
dlk.irvnp.cn/448668.Doc
dlk.irvnp.cn/862402.Doc
dlk.irvnp.cn/519595.Doc
dlk.irvnp.cn/351391.Doc
dlk.irvnp.cn/555937.Doc
dlk.irvnp.cn/806888.Doc
dlk.irvnp.cn/202606.Doc
dlj.irvnp.cn/520274.Doc
dlj.irvnp.cn/202125.Doc
dlj.irvnp.cn/808862.Doc
dlj.irvnp.cn/046464.Doc
dlj.irvnp.cn/139413.Doc
dlj.irvnp.cn/484846.Doc
dlj.irvnp.cn/227904.Doc
dlj.irvnp.cn/042062.Doc
dlj.irvnp.cn/088202.Doc
dlj.irvnp.cn/868866.Doc
dlh.irvnp.cn/931939.Doc
dlh.irvnp.cn/808086.Doc
dlh.irvnp.cn/911551.Doc
dlh.irvnp.cn/515915.Doc
dlh.irvnp.cn/964279.Doc
dlh.irvnp.cn/586988.Doc
dlh.irvnp.cn/955551.Doc
dlh.irvnp.cn/468860.Doc
dlh.irvnp.cn/464462.Doc
dlh.irvnp.cn/804624.Doc
dlg.irvnp.cn/781852.Doc
dlg.irvnp.cn/420086.Doc
dlg.irvnp.cn/155733.Doc
dlg.irvnp.cn/202602.Doc
dlg.irvnp.cn/097946.Doc
dlg.irvnp.cn/748398.Doc
dlg.irvnp.cn/022286.Doc
dlg.irvnp.cn/208248.Doc
dlg.irvnp.cn/482008.Doc
dlg.irvnp.cn/611824.Doc
dlf.irvnp.cn/468008.Doc
dlf.irvnp.cn/536849.Doc
dlf.irvnp.cn/202000.Doc
dlf.irvnp.cn/813990.Doc
dlf.irvnp.cn/844002.Doc
dlf.irvnp.cn/248808.Doc
dlf.irvnp.cn/462622.Doc
dlf.irvnp.cn/424682.Doc
dlf.irvnp.cn/917713.Doc
dlf.irvnp.cn/626006.Doc
dld.irvnp.cn/048420.Doc
dld.irvnp.cn/975135.Doc
dld.irvnp.cn/648026.Doc
dld.irvnp.cn/739111.Doc
dld.irvnp.cn/608284.Doc
dld.irvnp.cn/884404.Doc
dld.irvnp.cn/559931.Doc
dld.irvnp.cn/993539.Doc
dld.irvnp.cn/779957.Doc
dld.irvnp.cn/901152.Doc
dls.irvnp.cn/048408.Doc
dls.irvnp.cn/206082.Doc
dls.irvnp.cn/364087.Doc
dls.irvnp.cn/068440.Doc
dls.irvnp.cn/640048.Doc
dls.irvnp.cn/262228.Doc
dls.irvnp.cn/080884.Doc
dls.irvnp.cn/808086.Doc
dls.irvnp.cn/228626.Doc
dls.irvnp.cn/351359.Doc
dla.irvnp.cn/240602.Doc
dla.irvnp.cn/168841.Doc
dla.irvnp.cn/843330.Doc
dla.irvnp.cn/430448.Doc
dla.irvnp.cn/280028.Doc
dla.irvnp.cn/943147.Doc
dla.irvnp.cn/151957.Doc
dla.irvnp.cn/244622.Doc
dla.irvnp.cn/860204.Doc
dla.irvnp.cn/808602.Doc
dlp.irvnp.cn/395531.Doc
dlp.irvnp.cn/424180.Doc
dlp.irvnp.cn/487990.Doc
dlp.irvnp.cn/737111.Doc
dlp.irvnp.cn/138949.Doc
dlp.irvnp.cn/210963.Doc
dlp.irvnp.cn/406224.Doc
dlp.irvnp.cn/820020.Doc
dlp.irvnp.cn/208620.Doc
dlp.irvnp.cn/400068.Doc
dlo.irvnp.cn/559335.Doc
dlo.irvnp.cn/173517.Doc
dlo.irvnp.cn/240370.Doc
dlo.irvnp.cn/062260.Doc
dlo.irvnp.cn/353951.Doc
dlo.irvnp.cn/600886.Doc
dlo.irvnp.cn/183311.Doc
dlo.irvnp.cn/711513.Doc
dlo.irvnp.cn/775890.Doc
dlo.irvnp.cn/620715.Doc
dli.irvnp.cn/242984.Doc
dli.irvnp.cn/444600.Doc
dli.irvnp.cn/094338.Doc
dli.irvnp.cn/379939.Doc
dli.irvnp.cn/206246.Doc
dli.irvnp.cn/464260.Doc
dli.irvnp.cn/848428.Doc
dli.irvnp.cn/864826.Doc
dli.irvnp.cn/951977.Doc
dli.irvnp.cn/486066.Doc
dlu.irvnp.cn/806460.Doc
dlu.irvnp.cn/858856.Doc
dlu.irvnp.cn/944635.Doc
dlu.irvnp.cn/627463.Doc
dlu.irvnp.cn/444008.Doc
dlu.irvnp.cn/664026.Doc
dlu.irvnp.cn/004464.Doc
dlu.irvnp.cn/448868.Doc
dlu.irvnp.cn/337535.Doc
dlu.irvnp.cn/422808.Doc
dly.irvnp.cn/628466.Doc
dly.irvnp.cn/524376.Doc
dly.irvnp.cn/403016.Doc
dly.irvnp.cn/286966.Doc
dly.irvnp.cn/460006.Doc
dly.irvnp.cn/682359.Doc
dly.irvnp.cn/446842.Doc
dly.irvnp.cn/822680.Doc
dly.irvnp.cn/515366.Doc
dly.irvnp.cn/060086.Doc
dlt.irvnp.cn/208408.Doc
dlt.irvnp.cn/202484.Doc
dlt.irvnp.cn/040424.Doc
dlt.irvnp.cn/337328.Doc
dlt.irvnp.cn/973595.Doc
dlt.irvnp.cn/267725.Doc
dlt.irvnp.cn/155773.Doc
dlt.irvnp.cn/970255.Doc
dlt.irvnp.cn/153175.Doc
dlt.irvnp.cn/466242.Doc
dlr.irvnp.cn/440484.Doc
dlr.irvnp.cn/208428.Doc
dlr.irvnp.cn/703388.Doc
dlr.irvnp.cn/280684.Doc
dlr.irvnp.cn/004460.Doc
dlr.irvnp.cn/042460.Doc
dlr.irvnp.cn/684228.Doc
dlr.irvnp.cn/040668.Doc
dlr.irvnp.cn/737539.Doc
dlr.irvnp.cn/606248.Doc
dle.irvnp.cn/131971.Doc
dle.irvnp.cn/917713.Doc
dle.irvnp.cn/044282.Doc
dle.irvnp.cn/202440.Doc
dle.irvnp.cn/991733.Doc
dle.irvnp.cn/684848.Doc
dle.irvnp.cn/444494.Doc
dle.irvnp.cn/200286.Doc
dle.irvnp.cn/516530.Doc
dle.irvnp.cn/066484.Doc
dlw.irvnp.cn/222020.Doc
dlw.irvnp.cn/085484.Doc
dlw.irvnp.cn/202266.Doc
dlw.irvnp.cn/137446.Doc
dlw.irvnp.cn/600228.Doc
dlw.irvnp.cn/764172.Doc
dlw.irvnp.cn/493482.Doc
dlw.irvnp.cn/937733.Doc
dlw.irvnp.cn/537979.Doc
dlw.irvnp.cn/260244.Doc
dlq.irvnp.cn/188462.Doc
dlq.irvnp.cn/828788.Doc
dlq.irvnp.cn/260026.Doc
dlq.irvnp.cn/309095.Doc
dlq.irvnp.cn/795393.Doc
dlq.irvnp.cn/339337.Doc
dlq.irvnp.cn/225406.Doc
dlq.irvnp.cn/246424.Doc
dlq.irvnp.cn/286200.Doc
dlq.irvnp.cn/468666.Doc
dkm.irvnp.cn/515371.Doc
dkm.irvnp.cn/024804.Doc
dkm.irvnp.cn/288462.Doc
dkm.irvnp.cn/266426.Doc
dkm.irvnp.cn/226466.Doc
dkm.irvnp.cn/224882.Doc
dkm.irvnp.cn/395484.Doc
dkm.irvnp.cn/228664.Doc
dkm.irvnp.cn/332505.Doc
dkm.irvnp.cn/434748.Doc
dkn.irvnp.cn/649680.Doc
dkn.irvnp.cn/664886.Doc
dkn.irvnp.cn/462884.Doc
dkn.irvnp.cn/268022.Doc
dkn.irvnp.cn/484084.Doc
dkn.irvnp.cn/442468.Doc
dkn.irvnp.cn/644422.Doc
dkn.irvnp.cn/911797.Doc
dkn.irvnp.cn/593353.Doc
dkn.irvnp.cn/422314.Doc
dkb.irvnp.cn/919595.Doc
dkb.irvnp.cn/008688.Doc
dkb.irvnp.cn/266222.Doc
dkb.irvnp.cn/806468.Doc
dkb.irvnp.cn/064004.Doc
dkb.irvnp.cn/006608.Doc
dkb.irvnp.cn/790459.Doc
dkb.irvnp.cn/488806.Doc
dkb.irvnp.cn/446132.Doc
dkb.irvnp.cn/197122.Doc
dkv.irvnp.cn/517979.Doc
dkv.irvnp.cn/593779.Doc
dkv.irvnp.cn/080446.Doc
dkv.irvnp.cn/842480.Doc
dkv.irvnp.cn/374048.Doc
dkv.irvnp.cn/572185.Doc
dkv.irvnp.cn/600824.Doc
dkv.irvnp.cn/375397.Doc
dkv.irvnp.cn/644464.Doc
dkv.irvnp.cn/888882.Doc
dkc.irvnp.cn/191317.Doc
dkc.irvnp.cn/460840.Doc
dkc.irvnp.cn/042911.Doc
dkc.irvnp.cn/884282.Doc
dkc.irvnp.cn/664688.Doc
dkc.irvnp.cn/686526.Doc
dkc.irvnp.cn/599779.Doc
dkc.irvnp.cn/466202.Doc
dkc.irvnp.cn/769193.Doc
dkc.irvnp.cn/466228.Doc
dkx.irvnp.cn/373333.Doc
dkx.irvnp.cn/642228.Doc
dkx.irvnp.cn/460662.Doc
dkx.irvnp.cn/773755.Doc
dkx.irvnp.cn/014971.Doc
dkx.irvnp.cn/973139.Doc
dkx.irvnp.cn/644808.Doc
dkx.irvnp.cn/464484.Doc
dkx.irvnp.cn/826264.Doc
dkx.irvnp.cn/486244.Doc
dkz.irvnp.cn/131353.Doc
dkz.irvnp.cn/714106.Doc
dkz.irvnp.cn/208868.Doc
dkz.irvnp.cn/886488.Doc
dkz.irvnp.cn/993359.Doc
dkz.irvnp.cn/768934.Doc
dkz.irvnp.cn/042820.Doc
dkz.irvnp.cn/848068.Doc
dkz.irvnp.cn/204840.Doc
dkz.irvnp.cn/000866.Doc
dkl.irvnp.cn/409045.Doc
dkl.irvnp.cn/206862.Doc
dkl.irvnp.cn/682062.Doc
dkl.irvnp.cn/048882.Doc
dkl.irvnp.cn/084804.Doc
dkl.irvnp.cn/084266.Doc
dkl.irvnp.cn/868884.Doc
dkl.irvnp.cn/171571.Doc
dkl.irvnp.cn/206628.Doc
dkl.irvnp.cn/317456.Doc
dkk.irvnp.cn/280822.Doc
dkk.irvnp.cn/608820.Doc
dkk.irvnp.cn/677380.Doc
dkk.irvnp.cn/848802.Doc
dkk.irvnp.cn/024462.Doc
dkk.irvnp.cn/237905.Doc
dkk.irvnp.cn/994887.Doc
dkk.irvnp.cn/824220.Doc
dkk.irvnp.cn/117591.Doc
dkk.irvnp.cn/719739.Doc
