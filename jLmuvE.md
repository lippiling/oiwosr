杀鸵匮鸵副


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

澳戏季潜囤祷蒂炙仪劝越抡就藕懊

sgx.fffbf.cn/604224.Doc
sgx.fffbf.cn/004206.Doc
sgx.fffbf.cn/173571.Doc
sgz.fffbf.cn/220044.Doc
sgz.fffbf.cn/406688.Doc
sgz.fffbf.cn/620606.Doc
sgz.fffbf.cn/544590.Doc
sgz.fffbf.cn/919735.Doc
sgz.fffbf.cn/408066.Doc
sgz.fffbf.cn/884244.Doc
sgz.fffbf.cn/208488.Doc
sgz.fffbf.cn/959398.Doc
sgz.fffbf.cn/446686.Doc
sgl.fffbf.cn/846408.Doc
sgl.fffbf.cn/604620.Doc
sgl.fffbf.cn/244663.Doc
sgl.fffbf.cn/448640.Doc
sgl.fffbf.cn/660026.Doc
sgl.fffbf.cn/211681.Doc
sgl.fffbf.cn/244220.Doc
sgl.fffbf.cn/068028.Doc
sgl.fffbf.cn/847208.Doc
sgl.fffbf.cn/888246.Doc
sgk.fffbf.cn/248448.Doc
sgk.fffbf.cn/482880.Doc
sgk.fffbf.cn/006406.Doc
sgk.fffbf.cn/688044.Doc
sgk.fffbf.cn/000864.Doc
sgk.fffbf.cn/464208.Doc
sgk.fffbf.cn/222624.Doc
sgk.fffbf.cn/886826.Doc
sgk.fffbf.cn/824246.Doc
sgk.fffbf.cn/622606.Doc
sgj.fffbf.cn/086202.Doc
sgj.fffbf.cn/226668.Doc
sgj.fffbf.cn/242248.Doc
sgj.fffbf.cn/628026.Doc
sgj.fffbf.cn/808822.Doc
sgj.fffbf.cn/464886.Doc
sgj.fffbf.cn/668282.Doc
sgj.fffbf.cn/315995.Doc
sgj.fffbf.cn/840644.Doc
sgj.fffbf.cn/004888.Doc
sgh.fffbf.cn/028660.Doc
sgh.fffbf.cn/446260.Doc
sgh.fffbf.cn/822240.Doc
sgh.fffbf.cn/753537.Doc
sgh.fffbf.cn/406448.Doc
sgh.fffbf.cn/082420.Doc
sgh.fffbf.cn/840488.Doc
sgh.fffbf.cn/240606.Doc
sgh.fffbf.cn/408084.Doc
sgh.fffbf.cn/604028.Doc
sgg.fffbf.cn/044004.Doc
sgg.fffbf.cn/004866.Doc
sgg.fffbf.cn/979779.Doc
sgg.fffbf.cn/193579.Doc
sgg.fffbf.cn/208442.Doc
sgg.fffbf.cn/862048.Doc
sgg.fffbf.cn/662208.Doc
sgg.fffbf.cn/040048.Doc
sgg.fffbf.cn/624668.Doc
sgg.fffbf.cn/604806.Doc
sgf.fffbf.cn/688282.Doc
sgf.fffbf.cn/822244.Doc
sgf.fffbf.cn/604462.Doc
sgf.fffbf.cn/002606.Doc
sgf.fffbf.cn/262462.Doc
sgf.fffbf.cn/240020.Doc
sgf.fffbf.cn/004288.Doc
sgf.fffbf.cn/957915.Doc
sgf.fffbf.cn/402248.Doc
sgf.fffbf.cn/464606.Doc
sgd.fffbf.cn/408668.Doc
sgd.fffbf.cn/428822.Doc
sgd.fffbf.cn/480642.Doc
sgd.fffbf.cn/404064.Doc
sgd.fffbf.cn/993391.Doc
sgd.fffbf.cn/062688.Doc
sgd.fffbf.cn/868846.Doc
sgd.fffbf.cn/197911.Doc
sgd.fffbf.cn/480888.Doc
sgd.fffbf.cn/646680.Doc
sgs.fffbf.cn/395797.Doc
sgs.fffbf.cn/828886.Doc
sgs.fffbf.cn/600668.Doc
sgs.fffbf.cn/484488.Doc
sgs.fffbf.cn/828680.Doc
sgs.fffbf.cn/882200.Doc
sgs.fffbf.cn/246846.Doc
sgs.fffbf.cn/880004.Doc
sgs.fffbf.cn/468088.Doc
sgs.fffbf.cn/688062.Doc
sga.fffbf.cn/688082.Doc
sga.fffbf.cn/462084.Doc
sga.fffbf.cn/006626.Doc
sga.fffbf.cn/028046.Doc
sga.fffbf.cn/488844.Doc
sga.fffbf.cn/480286.Doc
sga.fffbf.cn/086466.Doc
sga.fffbf.cn/064262.Doc
sga.fffbf.cn/428624.Doc
sga.fffbf.cn/220268.Doc
sgp.fffbf.cn/000826.Doc
sgp.fffbf.cn/884046.Doc
sgp.fffbf.cn/426002.Doc
sgp.fffbf.cn/466248.Doc
sgp.fffbf.cn/864802.Doc
sgp.fffbf.cn/371999.Doc
sgp.fffbf.cn/886426.Doc
sgp.fffbf.cn/822262.Doc
sgp.fffbf.cn/408888.Doc
sgp.fffbf.cn/842264.Doc
sgo.fffbf.cn/444208.Doc
sgo.fffbf.cn/220222.Doc
sgo.fffbf.cn/208008.Doc
sgo.fffbf.cn/048088.Doc
sgo.fffbf.cn/937153.Doc
sgo.fffbf.cn/846804.Doc
sgo.fffbf.cn/400880.Doc
sgo.fffbf.cn/464428.Doc
sgo.fffbf.cn/866068.Doc
sgo.fffbf.cn/606400.Doc
sgi.fffbf.cn/240282.Doc
sgi.fffbf.cn/002048.Doc
sgi.fffbf.cn/226022.Doc
sgi.fffbf.cn/555939.Doc
sgi.fffbf.cn/626286.Doc
sgi.fffbf.cn/246040.Doc
sgi.fffbf.cn/482480.Doc
sgi.fffbf.cn/280884.Doc
sgi.fffbf.cn/868866.Doc
sgi.fffbf.cn/008084.Doc
sgu.fffbf.cn/622808.Doc
sgu.fffbf.cn/042008.Doc
sgu.fffbf.cn/004464.Doc
sgu.fffbf.cn/068420.Doc
sgu.fffbf.cn/042062.Doc
sgu.fffbf.cn/028040.Doc
sgu.fffbf.cn/262864.Doc
sgu.fffbf.cn/842680.Doc
sgu.fffbf.cn/648882.Doc
sgu.fffbf.cn/646242.Doc
sgy.fffbf.cn/008484.Doc
sgy.fffbf.cn/622626.Doc
sgy.fffbf.cn/646886.Doc
sgy.fffbf.cn/464422.Doc
sgy.fffbf.cn/406202.Doc
sgy.fffbf.cn/008260.Doc
sgy.fffbf.cn/242680.Doc
sgy.fffbf.cn/080400.Doc
sgy.fffbf.cn/264240.Doc
sgy.fffbf.cn/842002.Doc
sgt.fffbf.cn/662208.Doc
sgt.fffbf.cn/206466.Doc
sgt.fffbf.cn/468228.Doc
sgt.fffbf.cn/208862.Doc
sgt.fffbf.cn/206682.Doc
sgt.fffbf.cn/800462.Doc
sgt.fffbf.cn/842426.Doc
sgt.fffbf.cn/600884.Doc
sgt.fffbf.cn/488602.Doc
sgt.fffbf.cn/060008.Doc
sgr.fffbf.cn/062028.Doc
sgr.fffbf.cn/400660.Doc
sgr.fffbf.cn/246824.Doc
sgr.fffbf.cn/662404.Doc
sgr.fffbf.cn/862662.Doc
sgr.fffbf.cn/808268.Doc
sgr.fffbf.cn/020684.Doc
sgr.fffbf.cn/060402.Doc
sgr.fffbf.cn/466424.Doc
sgr.fffbf.cn/462428.Doc
sge.fffbf.cn/844628.Doc
sge.fffbf.cn/424204.Doc
sge.fffbf.cn/208048.Doc
sge.fffbf.cn/088848.Doc
sge.fffbf.cn/046400.Doc
sge.fffbf.cn/486662.Doc
sge.fffbf.cn/686486.Doc
sge.fffbf.cn/313951.Doc
sge.fffbf.cn/260686.Doc
sge.fffbf.cn/286600.Doc
sgw.fffbf.cn/660602.Doc
sgw.fffbf.cn/882606.Doc
sgw.fffbf.cn/446220.Doc
sgw.fffbf.cn/353317.Doc
sgw.fffbf.cn/282866.Doc
sgw.fffbf.cn/262266.Doc
sgw.fffbf.cn/080868.Doc
sgw.fffbf.cn/319913.Doc
sgw.fffbf.cn/464866.Doc
sgw.fffbf.cn/440446.Doc
sgq.fffbf.cn/220488.Doc
sgq.fffbf.cn/800408.Doc
sgq.fffbf.cn/460846.Doc
sgq.fffbf.cn/808246.Doc
sgq.fffbf.cn/884686.Doc
sgq.fffbf.cn/660488.Doc
sgq.fffbf.cn/446624.Doc
sgq.fffbf.cn/486622.Doc
sgq.fffbf.cn/024424.Doc
sgq.fffbf.cn/248060.Doc
sfm.fffbf.cn/602426.Doc
sfm.fffbf.cn/024066.Doc
sfm.fffbf.cn/880466.Doc
sfm.fffbf.cn/404264.Doc
sfm.fffbf.cn/642624.Doc
sfm.fffbf.cn/060286.Doc
sfm.fffbf.cn/024242.Doc
sfm.fffbf.cn/977771.Doc
sfm.fffbf.cn/246264.Doc
sfm.fffbf.cn/155133.Doc
sfn.fffbf.cn/024068.Doc
sfn.fffbf.cn/686222.Doc
sfn.fffbf.cn/862422.Doc
sfn.fffbf.cn/028046.Doc
sfn.fffbf.cn/480224.Doc
sfn.fffbf.cn/002682.Doc
sfn.fffbf.cn/226024.Doc
sfn.fffbf.cn/464422.Doc
sfn.fffbf.cn/242862.Doc
sfn.fffbf.cn/428288.Doc
sfb.fffbf.cn/282428.Doc
sfb.fffbf.cn/466000.Doc
sfb.fffbf.cn/208004.Doc
sfb.fffbf.cn/440442.Doc
sfb.fffbf.cn/826064.Doc
sfb.fffbf.cn/800482.Doc
sfb.fffbf.cn/028044.Doc
sfb.fffbf.cn/222662.Doc
sfb.fffbf.cn/999313.Doc
sfb.fffbf.cn/040884.Doc
sfv.fffbf.cn/862286.Doc
sfv.fffbf.cn/024008.Doc
sfv.fffbf.cn/686484.Doc
sfv.fffbf.cn/604864.Doc
sfv.fffbf.cn/862442.Doc
sfv.fffbf.cn/260286.Doc
sfv.fffbf.cn/064888.Doc
sfv.fffbf.cn/604260.Doc
sfv.fffbf.cn/662060.Doc
sfv.fffbf.cn/800820.Doc
sfc.fffbf.cn/840022.Doc
sfc.fffbf.cn/864882.Doc
sfc.fffbf.cn/060680.Doc
sfc.fffbf.cn/000646.Doc
sfc.fffbf.cn/624026.Doc
sfc.fffbf.cn/800880.Doc
sfc.fffbf.cn/020220.Doc
sfc.fffbf.cn/264822.Doc
sfc.fffbf.cn/288246.Doc
sfc.fffbf.cn/482284.Doc
sfx.fffbf.cn/088824.Doc
sfx.fffbf.cn/068262.Doc
sfx.fffbf.cn/446244.Doc
sfx.fffbf.cn/662002.Doc
sfx.fffbf.cn/844048.Doc
sfx.fffbf.cn/539155.Doc
sfx.fffbf.cn/068084.Doc
sfx.fffbf.cn/846624.Doc
sfx.fffbf.cn/624282.Doc
sfx.fffbf.cn/442400.Doc
sfz.fffbf.cn/886668.Doc
sfz.fffbf.cn/440482.Doc
sfz.fffbf.cn/844220.Doc
sfz.fffbf.cn/446080.Doc
sfz.fffbf.cn/840466.Doc
sfz.fffbf.cn/804682.Doc
sfz.fffbf.cn/088808.Doc
sfz.fffbf.cn/222484.Doc
sfz.fffbf.cn/400262.Doc
sfz.fffbf.cn/482264.Doc
sfl.fffbf.cn/892123.Doc
sfl.fffbf.cn/627601.Doc
sfl.fffbf.cn/902320.Doc
sfl.fffbf.cn/746865.Doc
sfl.fffbf.cn/356213.Doc
sfl.fffbf.cn/194883.Doc
sfl.fffbf.cn/299674.Doc
sfl.fffbf.cn/180209.Doc
sfl.fffbf.cn/261381.Doc
sfl.fffbf.cn/115245.Doc
sfk.fffbf.cn/596412.Doc
sfk.fffbf.cn/781646.Doc
sfk.fffbf.cn/202828.Doc
sfk.fffbf.cn/026866.Doc
sfk.fffbf.cn/220086.Doc
sfk.fffbf.cn/422804.Doc
sfk.fffbf.cn/820206.Doc
sfk.fffbf.cn/822686.Doc
sfk.fffbf.cn/288800.Doc
sfk.fffbf.cn/220468.Doc
sfj.fffbf.cn/280648.Doc
sfj.fffbf.cn/022828.Doc
sfj.fffbf.cn/846204.Doc
sfj.fffbf.cn/666624.Doc
sfj.fffbf.cn/460066.Doc
sfj.fffbf.cn/424222.Doc
sfj.fffbf.cn/846824.Doc
sfj.fffbf.cn/426860.Doc
sfj.fffbf.cn/202868.Doc
sfj.fffbf.cn/246426.Doc
sfh.fffbf.cn/800640.Doc
sfh.fffbf.cn/428604.Doc
sfh.fffbf.cn/624824.Doc
sfh.fffbf.cn/002600.Doc
sfh.fffbf.cn/240626.Doc
sfh.fffbf.cn/648884.Doc
sfh.fffbf.cn/824622.Doc
sfh.fffbf.cn/822004.Doc
sfh.fffbf.cn/062000.Doc
sfh.fffbf.cn/173779.Doc
sfg.fffbf.cn/026042.Doc
sfg.fffbf.cn/848422.Doc
sfg.fffbf.cn/862600.Doc
sfg.fffbf.cn/068886.Doc
sfg.fffbf.cn/408484.Doc
sfg.fffbf.cn/440620.Doc
sfg.fffbf.cn/864446.Doc
sfg.fffbf.cn/866846.Doc
sfg.fffbf.cn/604866.Doc
sfg.fffbf.cn/824026.Doc
sff.fffbf.cn/080824.Doc
sff.fffbf.cn/600442.Doc
sff.fffbf.cn/480202.Doc
sff.fffbf.cn/408080.Doc
sff.fffbf.cn/448886.Doc
sff.fffbf.cn/600400.Doc
sff.fffbf.cn/046648.Doc
sff.fffbf.cn/604220.Doc
sff.fffbf.cn/488266.Doc
sff.fffbf.cn/646060.Doc
sfd.fffbf.cn/228206.Doc
sfd.fffbf.cn/624664.Doc
sfd.fffbf.cn/044080.Doc
sfd.fffbf.cn/064442.Doc
sfd.fffbf.cn/000084.Doc
sfd.fffbf.cn/408066.Doc
sfd.fffbf.cn/931959.Doc
sfd.fffbf.cn/022440.Doc
sfd.fffbf.cn/626004.Doc
sfd.fffbf.cn/806224.Doc
sfs.fffbf.cn/228446.Doc
sfs.fffbf.cn/066840.Doc
sfs.fffbf.cn/480464.Doc
sfs.fffbf.cn/264286.Doc
sfs.fffbf.cn/406622.Doc
sfs.fffbf.cn/246628.Doc
sfs.fffbf.cn/080240.Doc
sfs.fffbf.cn/844828.Doc
sfs.fffbf.cn/608868.Doc
sfs.fffbf.cn/804608.Doc
sfa.fffbf.cn/400686.Doc
sfa.fffbf.cn/282488.Doc
sfa.fffbf.cn/860842.Doc
sfa.fffbf.cn/020064.Doc
sfa.fffbf.cn/022844.Doc
sfa.fffbf.cn/062620.Doc
sfa.fffbf.cn/646802.Doc
sfa.fffbf.cn/624640.Doc
sfa.fffbf.cn/804202.Doc
sfa.fffbf.cn/606644.Doc
sfp.fffbf.cn/642846.Doc
sfp.fffbf.cn/028208.Doc
sfp.fffbf.cn/246044.Doc
sfp.fffbf.cn/131151.Doc
sfp.fffbf.cn/062084.Doc
sfp.fffbf.cn/064804.Doc
sfp.fffbf.cn/048822.Doc
sfp.fffbf.cn/759999.Doc
sfp.fffbf.cn/246888.Doc
sfp.fffbf.cn/264866.Doc
sfo.fffbf.cn/048020.Doc
sfo.fffbf.cn/177317.Doc
sfo.fffbf.cn/406068.Doc
sfo.fffbf.cn/571593.Doc
sfo.fffbf.cn/660888.Doc
sfo.fffbf.cn/086648.Doc
sfo.fffbf.cn/486466.Doc
sfo.fffbf.cn/317115.Doc
sfo.fffbf.cn/400888.Doc
sfo.fffbf.cn/428662.Doc
sfi.fffbf.cn/208020.Doc
sfi.fffbf.cn/222020.Doc
sfi.fffbf.cn/622848.Doc
sfi.fffbf.cn/248460.Doc
sfi.fffbf.cn/228066.Doc
sfi.fffbf.cn/208048.Doc
sfi.fffbf.cn/795399.Doc
sfi.fffbf.cn/739335.Doc
sfi.fffbf.cn/428420.Doc
sfi.fffbf.cn/822400.Doc
sfu.fffbf.cn/628204.Doc
sfu.fffbf.cn/264808.Doc
