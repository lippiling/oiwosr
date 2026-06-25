========================================================
 Python 代码调试与问题排查 — 从 print 到专业工具
========================================================

调试是开发中不可避免的环节。Python 提供了从交互式
调试器到性能分析工具的完整工具链，掌握它们能显著
提升排查效率。

--------------------------------------------------------
1. pdb — 标准库调试器
--------------------------------------------------------

import pdb

def divide(a, b):
    result = a / b  # 如果 b=0 会抛 ZeroDivisionError
    return result

def process_data(data):
    total = 0
    for i, val in enumerate(data):
        # 在此处设置断点，进入交互式调试
        # pdb.set_trace()
        total += val
    return total

# pdb.set_trace() 会暂停执行并启动交互式命令行
# 常用命令:
#   n (next)     -> 执行下一行
#   c (continue) -> 继续执行到下一个断点
#   l (list)     -> 显示当前行周围的源码
#   p (print)    -> 打印表达式值，如 p total
#   pp           -> 漂亮打印
#   s (step)     -> 进入函数内部
#   r (return)   -> 执行到当前函数返回
#   q (quit)     -> 退出调试器
#   h (help)     -> 查看帮助

# 取消下面的注释即可体验：
# pdb.set_trace()
# print(divide(10, 2))

--------------------------------------------------------
2. breakpoint() — Python 3.7+ 内置调试入口
--------------------------------------------------------

def calculate_discount(price, level):
    """根据会员等级计算折扣"""
    breakpoint()  # 比 pdb.set_trace() 更简洁

    discounts = {"normal": 0, "silver": 0.1, "gold": 0.2}
    rate = discounts.get(level, 0)
    final = price * (1 - rate)
    return final

# breakpoint() 默认调用 pdb.set_trace()
# 可通过环境变量 PYTHONBREAKPOINT 自定义调试器：
#   export PYTHONBREAKPOINT=ipdb.set_trace  # 使用 ipdb
#   export PYTHONBREAKPOINT=0               # 全局禁用所有断点

# price = calculate_discount(100, "gold")

--------------------------------------------------------
3. logging — 替代 print 的正确做法
--------------------------------------------------------

import logging
import sys

# 配置日志系统
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    handlers=[
        logging.StreamHandler(sys.stdout),  # 输出到终端
        logging.FileHandler("app.log", encoding="utf-8"),  # 写入文件
    ]
)

logger = logging.getLogger("myapp")

def process_order(order_id, items):
    logger.info(f"开始处理订单: {order_id}")  # 信息级别
    logger.debug(f"订单详情: {items}")        # 调试级别，生产环境不输出

    if not items:
        logger.warning(f"订单 {order_id} 为空")  # 警告
        return False

    try:
        # 业务逻辑...
        result = 1 / 0  # 模拟错误
    except Exception as e:
        logger.exception(f"订单 {order_id} 处理失败")  # 自动包含异常堆栈
        return False

    logger.info(f"订单 {order_id} 处理成功")
    return True

# process_order("ORD-001", ["商品A", "商品B"])

--------------------------------------------------------
4. traceback — 异常栈深度分析
--------------------------------------------------------

import traceback

def deep_func():
    raise ValueError("深层错误")

def middle_func():
    deep_func()

def top_func():
    try:
        middle_func()
    except Exception:
        # 获取完整的异常堆栈字符串
        tb_str = traceback.format_exc()
        print("=== 完整堆栈 ===")
        print(tb_str)

        # 提取栈帧信息用于分析
        tb = traceback.extract_tb(sys.exc_info()[2])
        print("=== 栈帧摘要 ===")
        for filename, lineno, funcname, text in tb:
            print(f"  文件 {filename}:{lineno} 在 {funcname}()")
            print(f"    -> {text}")

# top_func()

--------------------------------------------------------
5. timeit — 微基准测试
--------------------------------------------------------

import timeit

# 测试列表推导式 vs for 循环的性能
setup_code = "data = list(range(1000))"
list_comp = "[x * 2 for x in data]"
for_loop = """
result = []
for x in data:
    result.append(x * 2)
"""

t1 = timeit.timeit(list_comp, setup=setup_code, number=10000)
t2 = timeit.timeit(for_loop, setup=setup_code, number=10000)

print(f"列表推导式: {t1:.4f}s")
print(f"For 循环:   {t2:.4f}s")
print(f"列表推导式快 {t2/t1:.2f} 倍")

# 使用命令行: python -m timeit "[x*2 for x in range(1000)]"

--------------------------------------------------------
6. cProfile — 函数级性能分析
--------------------------------------------------------

import cProfile
import pstats

def slow_function():
    total = 0
    for i in range(1000000):
        total += i ** 2
    return total

def fast_function():
    return sum(i ** 2 for i in range(1000000))

def main():
    slow_function()
    fast_function()

# 方式一：命令行分析
# python -m cProfile -o profile.prof script.py

# 方式二：代码内分析
profiler = cProfile.Profile()
profiler.enable()
main()
profiler.disable()

# 保存分析结果
profiler.dump_stats("profile.prof")

# 读取并排序输出（按累计时间排序）
stats = pstats.Stats("profile.prof")
stats.sort_stats("cumulative")
stats.print_stats(10)  # 只打印前 10 行

--------------------------------------------------------
7. memory_profiler — 内存使用分析
--------------------------------------------------------

# 安装：pip install memory_profiler psutil

@profile
def memory_intensive():
    """使用 @profile 装饰器逐行分析内存（需命令行执行）"""
    a = [i for i in range(100000)]     # 列表占用 ~0.8MB
    b = {i: i * 2 for i in range(50000)}  # 字典
    c = [x * 1.5 for x in a]            # 浮点数列表
    del a
    return b, c

# 命令行运行：
# python -m memory_profiler this_script.py
# 每行会显示 Mem usage 和 Increment 信息

import tracemalloc

# tracemalloc 可以跟踪内存分配的堆栈
tracemalloc.start()

def leak_function():
    """模拟内存泄漏"""
    big_data = [bytearray(1024 * 1024) for _ in range(10)]

snapshot = tracemalloc.take_snapshot()
leak_function()
snapshot2 = tracemalloc.take_snapshot()

# 显示内存分配最多的代码位置
top_stats = snapshot2.compare_to(snapshot, "lineno")
for stat in top_stats[:3]:
    print(stat)

--------------------------------------------------------
8. faulthandler — 排查段错误
--------------------------------------------------------

import faulthandler
import signal
import sys

# 启用故障处理器，在发生段错误时打印 Python 堆栈
faulthandler.enable()

# 也可以注册信号，按 Ctrl+\ 时打印所有线程的堆栈
faulthandler.register(signal.SIGUSR1)

# 模拟一个段错误（通常在 C 扩展中发生）
# import ctypes
# ctypes.string_at(0)  # 访问地址 0，触发 SIGSEGV

# faulthandler 会捕获并输出类似：
#   Fatal Python error: Segmentation fault
#   Current thread 0x...:
#     File "script.py", line 22 in buggy_extension
#     File "script.py", line 30 in main

# 常见调试工作流总结：
# 1. 简单问题 -> print / logging
# 2. 复杂逻辑 -> breakpoint() + pdb 逐行跟踪
# 3. 异常栈 -> traceback.format_exc()
# 4. 性能瓶颈 -> cProfile + timeit
# 5. 内存问题 -> tracemalloc / memory_profiler
# 6. 段错误 -> faulthandler

qmv.www669hq.cn/86444.Doc
qmv.www669hq.cn/22860.Doc
qmv.www669hq.cn/42082.Doc
qmv.www669hq.cn/04608.Doc
qmv.www669hq.cn/42604.Doc
qmv.www669hq.cn/62660.Doc
qmv.www669hq.cn/20000.Doc
qmv.www669hq.cn/02640.Doc
qmv.www669hq.cn/68266.Doc
qmv.www669hq.cn/28064.Doc
qmc.www669hq.cn/68220.Doc
qmc.www669hq.cn/82806.Doc
qmc.www669hq.cn/55777.Doc
qmc.www669hq.cn/64648.Doc
qmc.www669hq.cn/02806.Doc
qmc.www669hq.cn/60266.Doc
qmc.www669hq.cn/68888.Doc
qmc.www669hq.cn/88024.Doc
qmc.www669hq.cn/60462.Doc
qmc.www669hq.cn/60686.Doc
qmx.www669hq.cn/02282.Doc
qmx.www669hq.cn/62620.Doc
qmx.www669hq.cn/48608.Doc
qmx.www669hq.cn/77775.Doc
qmx.www669hq.cn/68206.Doc
qmx.www669hq.cn/26208.Doc
qmx.www669hq.cn/46444.Doc
qmx.www669hq.cn/06460.Doc
qmx.www669hq.cn/42426.Doc
qmx.www669hq.cn/06008.Doc
qmz.www669hq.cn/08424.Doc
qmz.www669hq.cn/04662.Doc
qmz.www669hq.cn/91977.Doc
qmz.www669hq.cn/64042.Doc
qmz.www669hq.cn/86482.Doc
qmz.www669hq.cn/60246.Doc
qmz.www669hq.cn/08646.Doc
qmz.www669hq.cn/08628.Doc
qmz.www669hq.cn/02820.Doc
qmz.www669hq.cn/60002.Doc
qml.www669hq.cn/28606.Doc
qml.www669hq.cn/44666.Doc
qml.www669hq.cn/24844.Doc
qml.www669hq.cn/08486.Doc
qml.www669hq.cn/26802.Doc
qml.www669hq.cn/28406.Doc
qml.www669hq.cn/06482.Doc
qml.www669hq.cn/26888.Doc
qml.www669hq.cn/24408.Doc
qml.www669hq.cn/33795.Doc
qmk.www669hq.cn/00664.Doc
qmk.www669hq.cn/06420.Doc
qmk.www669hq.cn/46820.Doc
qmk.www669hq.cn/48460.Doc
qmk.www669hq.cn/64840.Doc
qmk.www669hq.cn/62848.Doc
qmk.www669hq.cn/20222.Doc
qmk.www669hq.cn/86822.Doc
qmk.www669hq.cn/82460.Doc
qmk.www669hq.cn/04840.Doc
qmj.www669hq.cn/40044.Doc
qmj.www669hq.cn/86468.Doc
qmj.www669hq.cn/00440.Doc
qmj.www669hq.cn/77571.Doc
qmj.www669hq.cn/40820.Doc
qmj.www669hq.cn/86240.Doc
qmj.www669hq.cn/44460.Doc
qmj.www669hq.cn/62466.Doc
qmj.www669hq.cn/66062.Doc
qmj.www669hq.cn/00468.Doc
qmh.www669hq.cn/24488.Doc
qmh.www669hq.cn/40408.Doc
qmh.www669hq.cn/57975.Doc
qmh.www669hq.cn/66004.Doc
qmh.www669hq.cn/66804.Doc
qmh.www669hq.cn/20600.Doc
qmh.www669hq.cn/80820.Doc
qmh.www669hq.cn/28224.Doc
qmh.www669hq.cn/22400.Doc
qmh.www669hq.cn/04606.Doc
qmg.www669hq.cn/64286.Doc
qmg.www669hq.cn/80880.Doc
qmg.www669hq.cn/40426.Doc
qmg.www669hq.cn/44282.Doc
qmg.www669hq.cn/73171.Doc
qmg.www669hq.cn/06066.Doc
qmg.www669hq.cn/68864.Doc
qmg.www669hq.cn/82222.Doc
qmg.www669hq.cn/42222.Doc
qmg.www669hq.cn/86606.Doc
qmf.www669hq.cn/88426.Doc
qmf.www669hq.cn/60862.Doc
qmf.www669hq.cn/82628.Doc
qmf.www669hq.cn/26000.Doc
qmf.www669hq.cn/06088.Doc
qmf.www669hq.cn/48484.Doc
qmf.www669hq.cn/02482.Doc
qmf.www669hq.cn/20684.Doc
qmf.www669hq.cn/66808.Doc
qmf.www669hq.cn/66884.Doc
qmd.www669hq.cn/44448.Doc
qmd.www669hq.cn/91117.Doc
qmd.www669hq.cn/77919.Doc
qmd.www669hq.cn/64468.Doc
qmd.www669hq.cn/62464.Doc
qmd.www669hq.cn/62280.Doc
qmd.www669hq.cn/60228.Doc
qmd.www669hq.cn/33951.Doc
qmd.www669hq.cn/64226.Doc
qmd.www669hq.cn/26822.Doc
qms.www669hq.cn/02460.Doc
qms.www669hq.cn/66484.Doc
qms.www669hq.cn/22802.Doc
qms.www669hq.cn/46068.Doc
qms.www669hq.cn/86680.Doc
qms.www669hq.cn/40806.Doc
qms.www669hq.cn/44842.Doc
qms.www669hq.cn/42484.Doc
qms.www669hq.cn/80468.Doc
qms.www669hq.cn/86684.Doc
qma.www669hq.cn/08226.Doc
qma.www669hq.cn/26820.Doc
qma.www669hq.cn/84200.Doc
qma.www669hq.cn/26402.Doc
qma.www669hq.cn/28660.Doc
qma.www669hq.cn/26204.Doc
qma.www669hq.cn/44664.Doc
qma.www669hq.cn/88648.Doc
qma.www669hq.cn/08888.Doc
qma.www669hq.cn/28066.Doc
qmp.www669hq.cn/88822.Doc
qmp.www669hq.cn/80648.Doc
qmp.www669hq.cn/88242.Doc
qmp.www669hq.cn/46644.Doc
qmp.www669hq.cn/08460.Doc
qmp.www669hq.cn/04042.Doc
qmp.www669hq.cn/08640.Doc
qmp.www669hq.cn/28646.Doc
qmp.www669hq.cn/00620.Doc
qmp.www669hq.cn/28864.Doc
qmo.www669hq.cn/04222.Doc
qmo.www669hq.cn/46026.Doc
qmo.www669hq.cn/20408.Doc
qmo.www669hq.cn/86462.Doc
qmo.www669hq.cn/26406.Doc
qmo.www669hq.cn/60800.Doc
qmo.www669hq.cn/62824.Doc
qmo.www669hq.cn/48220.Doc
qmo.www669hq.cn/88486.Doc
qmo.www669hq.cn/82046.Doc
qmi.www669hq.cn/26208.Doc
qmi.www669hq.cn/48400.Doc
qmi.www669hq.cn/28244.Doc
qmi.www669hq.cn/26024.Doc
qmi.www669hq.cn/31775.Doc
qmi.www669hq.cn/22486.Doc
qmi.www669hq.cn/28008.Doc
qmi.www669hq.cn/02640.Doc
qmi.www669hq.cn/80848.Doc
qmi.www669hq.cn/04026.Doc
qmu.www669hq.cn/22848.Doc
qmu.www669hq.cn/68884.Doc
qmu.www669hq.cn/66806.Doc
qmu.www669hq.cn/66486.Doc
qmu.www669hq.cn/35317.Doc
qmu.www669hq.cn/84442.Doc
qmu.www669hq.cn/08602.Doc
qmu.www669hq.cn/60644.Doc
qmu.www669hq.cn/22804.Doc
qmu.www669hq.cn/84424.Doc
qmy.www669hq.cn/84286.Doc
qmy.www669hq.cn/68002.Doc
qmy.www669hq.cn/66666.Doc
qmy.www669hq.cn/40240.Doc
qmy.www669hq.cn/82484.Doc
qmy.www669hq.cn/48422.Doc
qmy.www669hq.cn/06688.Doc
qmy.www669hq.cn/48048.Doc
qmy.www669hq.cn/46028.Doc
qmy.www669hq.cn/80046.Doc
qmt.www669hq.cn/86688.Doc
qmt.www669hq.cn/19975.Doc
qmt.www669hq.cn/26068.Doc
qmt.www669hq.cn/26688.Doc
qmt.www669hq.cn/42462.Doc
qmt.www669hq.cn/88046.Doc
qmt.www669hq.cn/75151.Doc
qmt.www669hq.cn/80068.Doc
qmt.www669hq.cn/93333.Doc
qmt.www669hq.cn/40222.Doc
qmr.www669hq.cn/40224.Doc
qmr.www669hq.cn/08824.Doc
qmr.www669hq.cn/20284.Doc
qmr.www669hq.cn/60222.Doc
qmr.www669hq.cn/22888.Doc
qmr.www669hq.cn/68244.Doc
qmr.www669hq.cn/37135.Doc
qmr.www669hq.cn/48022.Doc
qmr.www669hq.cn/26022.Doc
qmr.www669hq.cn/04488.Doc
qme.www669hq.cn/66602.Doc
qme.www669hq.cn/24062.Doc
qme.www669hq.cn/40842.Doc
qme.www669hq.cn/06840.Doc
qme.www669hq.cn/20804.Doc
qme.www669hq.cn/06420.Doc
qme.www669hq.cn/84426.Doc
qme.www669hq.cn/26080.Doc
qme.www669hq.cn/84844.Doc
qme.www669hq.cn/28222.Doc
qmw.www669hq.cn/88208.Doc
qmw.www669hq.cn/06644.Doc
qmw.www669hq.cn/08422.Doc
qmw.www669hq.cn/88826.Doc
qmw.www669hq.cn/46688.Doc
qmw.www669hq.cn/88444.Doc
qmw.www669hq.cn/06660.Doc
qmw.www669hq.cn/22848.Doc
qmw.www669hq.cn/11371.Doc
qmw.www669hq.cn/04462.Doc
qmq.www669hq.cn/64066.Doc
qmq.www669hq.cn/80600.Doc
qmq.www669hq.cn/66422.Doc
qmq.www669hq.cn/42428.Doc
qmq.www669hq.cn/62002.Doc
qmq.www669hq.cn/88088.Doc
qmq.www669hq.cn/66604.Doc
qmq.www669hq.cn/97399.Doc
qmq.www669hq.cn/24284.Doc
qmq.www669hq.cn/06226.Doc
qnm.www669hq.cn/39119.Doc
qnm.www669hq.cn/97119.Doc
qnm.www669hq.cn/84086.Doc
qnm.www669hq.cn/84808.Doc
qnm.www669hq.cn/28468.Doc
qnm.www669hq.cn/64428.Doc
qnm.www669hq.cn/86648.Doc
qnm.www669hq.cn/39159.Doc
qnm.www669hq.cn/26842.Doc
qnm.www669hq.cn/46462.Doc
qnn.www669hq.cn/26246.Doc
qnn.www669hq.cn/00882.Doc
qnn.www669hq.cn/71373.Doc
qnn.www669hq.cn/86260.Doc
qnn.www669hq.cn/62462.Doc
qnn.www669hq.cn/06066.Doc
qnn.www669hq.cn/11951.Doc
qnn.www669hq.cn/82840.Doc
qnn.www669hq.cn/44222.Doc
qnn.www669hq.cn/66662.Doc
qnb.www669hq.cn/84662.Doc
qnb.www669hq.cn/84442.Doc
qnb.www669hq.cn/46842.Doc
qnb.www669hq.cn/40642.Doc
qnb.www669hq.cn/02242.Doc
qnb.www669hq.cn/68262.Doc
qnb.www669hq.cn/42242.Doc
qnb.www669hq.cn/33791.Doc
qnb.www669hq.cn/20422.Doc
qnb.www669hq.cn/39995.Doc
qnv.www669hq.cn/60002.Doc
qnv.www669hq.cn/00080.Doc
qnv.www669hq.cn/04668.Doc
qnv.www669hq.cn/28080.Doc
qnv.www669hq.cn/28460.Doc
qnv.www669hq.cn/95155.Doc
qnv.www669hq.cn/68268.Doc
qnv.www669hq.cn/20808.Doc
qnv.www669hq.cn/48060.Doc
qnv.www669hq.cn/08402.Doc
qnc.www669hq.cn/02420.Doc
qnc.www669hq.cn/04000.Doc
qnc.www669hq.cn/46044.Doc
qnc.www669hq.cn/80880.Doc
qnc.www669hq.cn/06200.Doc
qnc.www669hq.cn/60260.Doc
qnc.www669hq.cn/06424.Doc
qnc.www669hq.cn/68606.Doc
qnc.www669hq.cn/28022.Doc
qnc.www669hq.cn/88802.Doc
qnx.www669hq.cn/60646.Doc
qnx.www669hq.cn/17935.Doc
qnx.www669hq.cn/48820.Doc
qnx.www669hq.cn/80600.Doc
qnx.www669hq.cn/86602.Doc
qnx.www669hq.cn/82264.Doc
qnx.www669hq.cn/86642.Doc
qnx.www669hq.cn/06484.Doc
qnx.www669hq.cn/48682.Doc
qnx.www669hq.cn/51771.Doc
qnz.www669hq.cn/28242.Doc
qnz.www669hq.cn/60448.Doc
qnz.www669hq.cn/88842.Doc
qnz.www669hq.cn/06406.Doc
qnz.www669hq.cn/60020.Doc
qnz.www669hq.cn/82600.Doc
qnz.www669hq.cn/46688.Doc
qnz.www669hq.cn/64688.Doc
qnz.www669hq.cn/13533.Doc
qnz.www669hq.cn/19179.Doc
qnl.www669hq.cn/60028.Doc
qnl.www669hq.cn/64822.Doc
qnl.www669hq.cn/84848.Doc
qnl.www669hq.cn/64468.Doc
qnl.www669hq.cn/42882.Doc
qnl.www669hq.cn/80244.Doc
qnl.www669hq.cn/46660.Doc
qnl.www669hq.cn/62624.Doc
qnl.www669hq.cn/59179.Doc
qnl.www669hq.cn/62026.Doc
qnk.www669hq.cn/97759.Doc
qnk.www669hq.cn/44422.Doc
qnk.www669hq.cn/24688.Doc
qnk.www669hq.cn/82666.Doc
qnk.www669hq.cn/20682.Doc
qnk.www669hq.cn/66882.Doc
qnk.www669hq.cn/84842.Doc
qnk.www669hq.cn/08284.Doc
qnk.www669hq.cn/40482.Doc
qnk.www669hq.cn/02226.Doc
qnj.www669hq.cn/84080.Doc
qnj.www669hq.cn/60266.Doc
qnj.www669hq.cn/48088.Doc
qnj.www669hq.cn/44428.Doc
qnj.www669hq.cn/40668.Doc
qnj.www669hq.cn/66082.Doc
qnj.www669hq.cn/04240.Doc
qnj.www669hq.cn/02648.Doc
qnj.www669hq.cn/68464.Doc
qnj.www669hq.cn/20408.Doc
qnh.www669hq.cn/02088.Doc
qnh.www669hq.cn/15757.Doc
qnh.www669hq.cn/26840.Doc
qnh.www669hq.cn/40268.Doc
qnh.www669hq.cn/93131.Doc
qnh.www669hq.cn/48806.Doc
qnh.www669hq.cn/80866.Doc
qnh.www669hq.cn/31593.Doc
qnh.www669hq.cn/20242.Doc
qnh.www669hq.cn/24684.Doc
qng.www669hq.cn/82244.Doc
qng.www669hq.cn/40622.Doc
qng.www669hq.cn/59331.Doc
qng.www669hq.cn/08408.Doc
qng.www669hq.cn/44260.Doc
qng.www669hq.cn/66828.Doc
qng.www669hq.cn/26426.Doc
qng.www669hq.cn/26680.Doc
qng.www669hq.cn/02684.Doc
qng.www669hq.cn/82680.Doc
qnf.www669hq.cn/40860.Doc
qnf.www669hq.cn/46480.Doc
qnf.www669hq.cn/39579.Doc
qnf.www669hq.cn/46204.Doc
qnf.www669hq.cn/64826.Doc
qnf.www669hq.cn/22666.Doc
qnf.www669hq.cn/80884.Doc
qnf.www669hq.cn/82080.Doc
qnf.www669hq.cn/26844.Doc
qnf.www669hq.cn/06828.Doc
qnd.www669hq.cn/88444.Doc
qnd.www669hq.cn/97379.Doc
qnd.www669hq.cn/66428.Doc
qnd.www669hq.cn/80466.Doc
qnd.www669hq.cn/28208.Doc
qnd.www669hq.cn/42060.Doc
qnd.www669hq.cn/08880.Doc
qnd.www669hq.cn/62268.Doc
qnd.www669hq.cn/86220.Doc
qnd.www669hq.cn/24266.Doc
qns.www669hq.cn/02666.Doc
qns.www669hq.cn/62442.Doc
qns.www669hq.cn/86048.Doc
qns.www669hq.cn/42804.Doc
qns.www669hq.cn/82880.Doc
qns.www669hq.cn/15159.Doc
qns.www669hq.cn/71795.Doc
qns.www669hq.cn/62428.Doc
qns.www669hq.cn/68048.Doc
qns.www669hq.cn/06004.Doc
qna.www669hq.cn/04208.Doc
qna.www669hq.cn/88246.Doc
qna.www669hq.cn/66620.Doc
qna.www669hq.cn/04026.Doc
qna.www669hq.cn/22880.Doc
qna.www669hq.cn/42226.Doc
qna.www669hq.cn/48400.Doc
qna.www669hq.cn/62440.Doc
qna.www669hq.cn/60624.Doc
qna.www669hq.cn/21581.Doc
qnp.www669hq.cn/82364.Doc
qnp.www669hq.cn/52637.Doc
qnp.www669hq.cn/86775.Doc
qnp.www669hq.cn/52912.Doc
qnp.www669hq.cn/95888.Doc
qnp.www669hq.cn/76299.Doc
qnp.www669hq.cn/58990.Doc
qnp.www669hq.cn/87348.Doc
qnp.www669hq.cn/30179.Doc
qnp.www669hq.cn/20539.Doc
qno.www669hq.cn/48744.Doc
qno.www669hq.cn/28844.Doc
qno.www669hq.cn/08064.Doc
qno.www669hq.cn/42068.Doc
qno.www669hq.cn/80446.Doc
qno.www669hq.cn/66844.Doc
qno.www669hq.cn/80820.Doc
qno.www669hq.cn/00884.Doc
qno.www669hq.cn/66424.Doc
qno.www669hq.cn/62202.Doc
qni.www669hq.cn/66462.Doc
qni.www669hq.cn/06848.Doc
qni.www669hq.cn/84880.Doc
qni.www669hq.cn/86244.Doc
qni.www669hq.cn/80646.Doc
qni.www669hq.cn/26448.Doc
qni.www669hq.cn/08080.Doc
qni.www669hq.cn/62286.Doc
qni.www669hq.cn/80622.Doc
qni.www669hq.cn/08668.Doc
qnu.www669hq.cn/86062.Doc
qnu.www669hq.cn/71555.Doc
qnu.www669hq.cn/60402.Doc
qnu.www669hq.cn/02602.Doc
qnu.www669hq.cn/44606.Doc
qnu.www669hq.cn/84226.Doc
qnu.www669hq.cn/86002.Doc
qnu.www669hq.cn/00428.Doc
qnu.www669hq.cn/80228.Doc
qnu.www669hq.cn/28446.Doc
qny.www669hq.cn/88888.Doc
qny.www669hq.cn/06422.Doc
qny.www669hq.cn/00004.Doc
qny.www669hq.cn/37751.Doc
qny.www669hq.cn/28626.Doc
qny.www669hq.cn/06448.Doc
qny.www669hq.cn/71119.Doc
qny.www669hq.cn/88484.Doc
qny.www669hq.cn/20806.Doc
qny.www669hq.cn/66240.Doc
qnt.www669hq.cn/64688.Doc
qnt.www669hq.cn/00024.Doc
qnt.www669hq.cn/20220.Doc
qnt.www669hq.cn/40024.Doc
qnt.www669hq.cn/42242.Doc
qnt.www669hq.cn/66628.Doc
qnt.www669hq.cn/26204.Doc
qnt.www669hq.cn/62460.Doc
qnt.www669hq.cn/02404.Doc
qnt.www669hq.cn/02800.Doc
qnr.www669hq.cn/64002.Doc
qnr.www669hq.cn/40644.Doc
qnr.www669hq.cn/08842.Doc
qnr.www669hq.cn/00428.Doc
qnr.www669hq.cn/28628.Doc
qnr.www669hq.cn/28448.Doc
qnr.www669hq.cn/42686.Doc
qnr.www669hq.cn/22684.Doc
qnr.www669hq.cn/22808.Doc
qnr.www669hq.cn/86266.Doc
qne.www669hq.cn/06446.Doc
qne.www669hq.cn/60680.Doc
qne.www669hq.cn/80468.Doc
qne.www669hq.cn/86446.Doc
qne.www669hq.cn/88668.Doc
qne.www669hq.cn/00688.Doc
qne.www669hq.cn/66648.Doc
qne.www669hq.cn/22822.Doc
qne.www669hq.cn/66600.Doc
qne.www669hq.cn/08222.Doc
qnw.www669hq.cn/66448.Doc
qnw.www669hq.cn/00040.Doc
qnw.www669hq.cn/48880.Doc
qnw.www669hq.cn/66822.Doc
qnw.www669hq.cn/60862.Doc
qnw.www669hq.cn/26806.Doc
qnw.www669hq.cn/46868.Doc
qnw.www669hq.cn/48802.Doc
qnw.www669hq.cn/88862.Doc
qnw.www669hq.cn/06200.Doc
qnq.www669hq.cn/46084.Doc
qnq.www669hq.cn/26806.Doc
qnq.www669hq.cn/15153.Doc
qnq.www669hq.cn/00660.Doc
qnq.www669hq.cn/28482.Doc
qnq.www669hq.cn/20426.Doc
qnq.www669hq.cn/66642.Doc
qnq.www669hq.cn/82004.Doc
qnq.www669hq.cn/66440.Doc
qnq.www669hq.cn/22028.Doc
