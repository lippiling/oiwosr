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

eky.wandk5.cn/82026.Doc
eky.wandk5.cn/62880.Doc
eky.wandk5.cn/26844.Doc
eky.wandk5.cn/08022.Doc
eky.wandk5.cn/08482.Doc
eky.wandk5.cn/28628.Doc
eky.wandk5.cn/24282.Doc
eky.wandk5.cn/86480.Doc
eky.wandk5.cn/00860.Doc
eky.wandk5.cn/48202.Doc
ekt.wandk5.cn/44282.Doc
ekt.wandk5.cn/04006.Doc
ekt.wandk5.cn/48002.Doc
ekt.wandk5.cn/44288.Doc
ekt.wandk5.cn/08406.Doc
ekt.wandk5.cn/02248.Doc
ekt.wandk5.cn/86804.Doc
ekt.wandk5.cn/26242.Doc
ekt.wandk5.cn/46266.Doc
ekt.wandk5.cn/64664.Doc
ekr.wandk5.cn/66442.Doc
ekr.wandk5.cn/23312.Doc
ekr.wandk5.cn/44062.Doc
ekr.wandk5.cn/08468.Doc
ekr.wandk5.cn/24884.Doc
ekr.wandk5.cn/86620.Doc
ekr.wandk5.cn/24666.Doc
ekr.wandk5.cn/50987.Doc
ekr.wandk5.cn/10431.Doc
ekr.wandk5.cn/82464.Doc
eke.wandk5.cn/24808.Doc
eke.wandk5.cn/26226.Doc
eke.wandk5.cn/82686.Doc
eke.wandk5.cn/42448.Doc
eke.wandk5.cn/80420.Doc
eke.wandk5.cn/86040.Doc
eke.wandk5.cn/20628.Doc
eke.wandk5.cn/46220.Doc
eke.wandk5.cn/60046.Doc
eke.wandk5.cn/60806.Doc
ekw.wandk5.cn/06264.Doc
ekw.wandk5.cn/46868.Doc
ekw.wandk5.cn/68286.Doc
ekw.wandk5.cn/24862.Doc
ekw.wandk5.cn/06002.Doc
ekw.wandk5.cn/06282.Doc
ekw.wandk5.cn/62600.Doc
ekw.wandk5.cn/28820.Doc
ekw.wandk5.cn/64080.Doc
ekw.wandk5.cn/22202.Doc
ekq.wandk5.cn/24840.Doc
ekq.wandk5.cn/62824.Doc
ekq.wandk5.cn/02620.Doc
ekq.wandk5.cn/04200.Doc
ekq.wandk5.cn/22600.Doc
ekq.wandk5.cn/62644.Doc
ekq.wandk5.cn/42684.Doc
ekq.wandk5.cn/02688.Doc
ekq.wandk5.cn/24442.Doc
ekq.wandk5.cn/80862.Doc
ejm.wandk5.cn/68464.Doc
ejm.wandk5.cn/46660.Doc
ejm.wandk5.cn/42268.Doc
ejm.wandk5.cn/86644.Doc
ejm.wandk5.cn/00466.Doc
ejm.wandk5.cn/80028.Doc
ejm.wandk5.cn/28020.Doc
ejm.wandk5.cn/04020.Doc
ejm.wandk5.cn/24606.Doc
ejm.wandk5.cn/42282.Doc
ejn.wandk5.cn/84226.Doc
ejn.wandk5.cn/60068.Doc
ejn.wandk5.cn/24064.Doc
ejn.wandk5.cn/44468.Doc
ejn.wandk5.cn/68268.Doc
ejn.wandk5.cn/40200.Doc
ejn.wandk5.cn/04066.Doc
ejn.wandk5.cn/00006.Doc
ejn.wandk5.cn/02682.Doc
ejn.wandk5.cn/24248.Doc
ejb.wandk5.cn/22240.Doc
ejb.wandk5.cn/86402.Doc
ejb.wandk5.cn/46862.Doc
ejb.wandk5.cn/24220.Doc
ejb.wandk5.cn/60602.Doc
ejb.wandk5.cn/64288.Doc
ejb.wandk5.cn/44846.Doc
ejb.wandk5.cn/66684.Doc
ejb.wandk5.cn/20684.Doc
ejb.wandk5.cn/20206.Doc
ejv.wandk5.cn/84666.Doc
ejv.wandk5.cn/88062.Doc
ejv.wandk5.cn/40488.Doc
ejv.wandk5.cn/60000.Doc
ejv.wandk5.cn/11739.Doc
ejv.wandk5.cn/06880.Doc
ejv.wandk5.cn/08628.Doc
ejv.wandk5.cn/44622.Doc
ejv.wandk5.cn/04444.Doc
ejv.wandk5.cn/22002.Doc
ejc.wandk5.cn/04482.Doc
ejc.wandk5.cn/60040.Doc
ejc.wandk5.cn/62208.Doc
ejc.wandk5.cn/62000.Doc
ejc.wandk5.cn/46848.Doc
ejc.wandk5.cn/86466.Doc
ejc.wandk5.cn/46082.Doc
ejc.wandk5.cn/84846.Doc
ejc.wandk5.cn/66264.Doc
ejc.wandk5.cn/82800.Doc
ejx.wandk5.cn/84622.Doc
ejx.wandk5.cn/68640.Doc
ejx.wandk5.cn/64082.Doc
ejx.wandk5.cn/80482.Doc
ejx.wandk5.cn/22866.Doc
ejx.wandk5.cn/20000.Doc
ejx.wandk5.cn/42688.Doc
ejx.wandk5.cn/04068.Doc
ejx.wandk5.cn/46662.Doc
ejx.wandk5.cn/48228.Doc
ejz.wandk5.cn/06424.Doc
ejz.wandk5.cn/28442.Doc
ejz.wandk5.cn/02406.Doc
ejz.wandk5.cn/68426.Doc
ejz.wandk5.cn/66042.Doc
ejz.wandk5.cn/60848.Doc
ejz.wandk5.cn/84628.Doc
ejz.wandk5.cn/08082.Doc
ejz.wandk5.cn/28806.Doc
ejz.wandk5.cn/08060.Doc
ejl.wandk5.cn/80822.Doc
ejl.wandk5.cn/66802.Doc
ejl.wandk5.cn/06460.Doc
ejl.wandk5.cn/88866.Doc
ejl.wandk5.cn/00228.Doc
ejl.wandk5.cn/06064.Doc
ejl.wandk5.cn/44860.Doc
ejl.wandk5.cn/80668.Doc
ejl.wandk5.cn/42444.Doc
ejl.wandk5.cn/64024.Doc
ejk.wandk5.cn/84888.Doc
ejk.wandk5.cn/40026.Doc
ejk.wandk5.cn/48422.Doc
ejk.wandk5.cn/24220.Doc
ejk.wandk5.cn/22482.Doc
ejk.wandk5.cn/66088.Doc
ejk.wandk5.cn/08068.Doc
ejk.wandk5.cn/04246.Doc
ejk.wandk5.cn/42822.Doc
ejk.wandk5.cn/04046.Doc
ejj.wandk5.cn/42202.Doc
ejj.wandk5.cn/26088.Doc
ejj.wandk5.cn/60280.Doc
ejj.wandk5.cn/62402.Doc
ejj.wandk5.cn/24624.Doc
ejj.wandk5.cn/40886.Doc
ejj.wandk5.cn/44426.Doc
ejj.wandk5.cn/24046.Doc
ejj.wandk5.cn/48044.Doc
ejj.wandk5.cn/44060.Doc
ejh.wandk5.cn/66064.Doc
ejh.wandk5.cn/08064.Doc
ejh.wandk5.cn/44806.Doc
ejh.wandk5.cn/06808.Doc
ejh.wandk5.cn/80026.Doc
ejh.wandk5.cn/20862.Doc
ejh.wandk5.cn/20208.Doc
ejh.wandk5.cn/24664.Doc
ejh.wandk5.cn/00220.Doc
ejh.wandk5.cn/04440.Doc
ejg.wandk5.cn/62046.Doc
ejg.wandk5.cn/82440.Doc
ejg.wandk5.cn/24640.Doc
ejg.wandk5.cn/68882.Doc
ejg.wandk5.cn/04844.Doc
ejg.wandk5.cn/82020.Doc
ejg.wandk5.cn/64886.Doc
ejg.wandk5.cn/20868.Doc
ejg.wandk5.cn/62846.Doc
ejg.wandk5.cn/24020.Doc
ejf.wandk5.cn/04442.Doc
ejf.wandk5.cn/06024.Doc
ejf.wandk5.cn/28608.Doc
ejf.wandk5.cn/46448.Doc
ejf.wandk5.cn/22268.Doc
ejf.wandk5.cn/60086.Doc
ejf.wandk5.cn/02202.Doc
ejf.wandk5.cn/44888.Doc
ejf.wandk5.cn/42868.Doc
ejf.wandk5.cn/07956.Doc
ejd.wandk5.cn/66466.Doc
ejd.wandk5.cn/04628.Doc
ejd.wandk5.cn/02242.Doc
ejd.wandk5.cn/08080.Doc
ejd.wandk5.cn/46848.Doc
ejd.wandk5.cn/04622.Doc
ejd.wandk5.cn/82862.Doc
ejd.wandk5.cn/80608.Doc
ejd.wandk5.cn/28737.Doc
ejd.wandk5.cn/84626.Doc
ejs.wandk5.cn/88806.Doc
ejs.wandk5.cn/42440.Doc
ejs.wandk5.cn/06606.Doc
ejs.wandk5.cn/86660.Doc
ejs.wandk5.cn/24882.Doc
ejs.wandk5.cn/84404.Doc
ejs.wandk5.cn/08266.Doc
ejs.wandk5.cn/26440.Doc
ejs.wandk5.cn/02062.Doc
ejs.wandk5.cn/44266.Doc
eja.wandk5.cn/44246.Doc
eja.wandk5.cn/86062.Doc
eja.wandk5.cn/24482.Doc
eja.wandk5.cn/28442.Doc
eja.wandk5.cn/40884.Doc
eja.wandk5.cn/08068.Doc
eja.wandk5.cn/40084.Doc
eja.wandk5.cn/40820.Doc
eja.wandk5.cn/08840.Doc
eja.wandk5.cn/40408.Doc
ejp.wandk5.cn/26602.Doc
ejp.wandk5.cn/48826.Doc
ejp.wandk5.cn/60668.Doc
ejp.wandk5.cn/24042.Doc
ejp.wandk5.cn/42004.Doc
ejp.wandk5.cn/02280.Doc
ejp.wandk5.cn/08048.Doc
ejp.wandk5.cn/66460.Doc
ejp.wandk5.cn/08608.Doc
ejp.wandk5.cn/66400.Doc
ejo.wandk5.cn/88884.Doc
ejo.wandk5.cn/22662.Doc
ejo.wandk5.cn/02086.Doc
ejo.wandk5.cn/62226.Doc
ejo.wandk5.cn/44084.Doc
ejo.wandk5.cn/20460.Doc
ejo.wandk5.cn/88824.Doc
ejo.wandk5.cn/08288.Doc
ejo.wandk5.cn/68802.Doc
ejo.wandk5.cn/82628.Doc
eji.wandk5.cn/00028.Doc
eji.wandk5.cn/00866.Doc
eji.wandk5.cn/02628.Doc
eji.wandk5.cn/64022.Doc
eji.wandk5.cn/66468.Doc
eji.wandk5.cn/86082.Doc
eji.wandk5.cn/20824.Doc
eji.wandk5.cn/88804.Doc
eji.wandk5.cn/44006.Doc
eji.wandk5.cn/06244.Doc
eju.wandk5.cn/24824.Doc
eju.wandk5.cn/26246.Doc
eju.wandk5.cn/04020.Doc
eju.wandk5.cn/64204.Doc
eju.wandk5.cn/04020.Doc
eju.wandk5.cn/22462.Doc
eju.wandk5.cn/22406.Doc
eju.wandk5.cn/22606.Doc
eju.wandk5.cn/44640.Doc
eju.wandk5.cn/04888.Doc
ejy.wandk5.cn/62608.Doc
ejy.wandk5.cn/60068.Doc
ejy.wandk5.cn/66444.Doc
ejy.wandk5.cn/88248.Doc
ejy.wandk5.cn/80628.Doc
ejy.wandk5.cn/28802.Doc
ejy.wandk5.cn/84820.Doc
ejy.wandk5.cn/48080.Doc
ejy.wandk5.cn/82482.Doc
ejy.wandk5.cn/86022.Doc
ejt.wandk5.cn/42228.Doc
ejt.wandk5.cn/20246.Doc
ejt.wandk5.cn/20244.Doc
ejt.wandk5.cn/64804.Doc
ejt.wandk5.cn/06448.Doc
ejt.wandk5.cn/06624.Doc
ejt.wandk5.cn/00402.Doc
ejt.wandk5.cn/66080.Doc
ejt.wandk5.cn/46206.Doc
ejt.wandk5.cn/08828.Doc
ejr.wandk5.cn/06288.Doc
ejr.wandk5.cn/02208.Doc
ejr.wandk5.cn/84804.Doc
ejr.wandk5.cn/28608.Doc
ejr.wandk5.cn/62082.Doc
ejr.wandk5.cn/20428.Doc
ejr.wandk5.cn/20824.Doc
ejr.wandk5.cn/40282.Doc
ejr.wandk5.cn/46866.Doc
ejr.wandk5.cn/86288.Doc
eje.wandk5.cn/46600.Doc
eje.wandk5.cn/46064.Doc
eje.wandk5.cn/04488.Doc
eje.wandk5.cn/48446.Doc
eje.wandk5.cn/66444.Doc
eje.wandk5.cn/22802.Doc
eje.wandk5.cn/02400.Doc
eje.wandk5.cn/68286.Doc
eje.wandk5.cn/02848.Doc
eje.wandk5.cn/02880.Doc
ejw.wandk5.cn/66648.Doc
ejw.wandk5.cn/02808.Doc
ejw.wandk5.cn/66404.Doc
ejw.wandk5.cn/08468.Doc
ejw.wandk5.cn/24048.Doc
ejw.wandk5.cn/24604.Doc
ejw.wandk5.cn/84406.Doc
ejw.wandk5.cn/40608.Doc
ejw.wandk5.cn/44268.Doc
ejw.wandk5.cn/20800.Doc
ejq.wandk5.cn/20840.Doc
ejq.wandk5.cn/08646.Doc
ejq.wandk5.cn/88408.Doc
ejq.wandk5.cn/02208.Doc
ejq.wandk5.cn/46440.Doc
ejq.wandk5.cn/40024.Doc
ejq.wandk5.cn/88466.Doc
ejq.wandk5.cn/22460.Doc
ejq.wandk5.cn/42002.Doc
ejq.wandk5.cn/28648.Doc
ehm.wandk5.cn/28402.Doc
ehm.wandk5.cn/22042.Doc
ehm.wandk5.cn/66420.Doc
ehm.wandk5.cn/26200.Doc
ehm.wandk5.cn/02244.Doc
ehm.wandk5.cn/40640.Doc
ehm.wandk5.cn/48406.Doc
ehm.wandk5.cn/42864.Doc
ehm.wandk5.cn/02264.Doc
ehm.wandk5.cn/42668.Doc
ehn.wandk5.cn/26266.Doc
ehn.wandk5.cn/40226.Doc
ehn.wandk5.cn/04002.Doc
ehn.wandk5.cn/64222.Doc
ehn.wandk5.cn/82086.Doc
ehn.wandk5.cn/46644.Doc
ehn.wandk5.cn/08048.Doc
ehn.wandk5.cn/88226.Doc
ehn.wandk5.cn/88402.Doc
ehn.wandk5.cn/08082.Doc
ehb.wandk5.cn/47451.Doc
ehb.wandk5.cn/80242.Doc
ehb.wandk5.cn/66666.Doc
ehb.wandk5.cn/48266.Doc
ehb.wandk5.cn/62020.Doc
ehb.wandk5.cn/24868.Doc
ehb.wandk5.cn/20240.Doc
ehb.wandk5.cn/62622.Doc
ehb.wandk5.cn/64686.Doc
ehb.wandk5.cn/68860.Doc
ehv.wandk5.cn/24424.Doc
ehv.wandk5.cn/40626.Doc
ehv.wandk5.cn/08622.Doc
ehv.wandk5.cn/42620.Doc
ehv.wandk5.cn/46420.Doc
ehv.wandk5.cn/20860.Doc
ehv.wandk5.cn/28660.Doc
ehv.wandk5.cn/80640.Doc
ehv.wandk5.cn/46626.Doc
ehv.wandk5.cn/26020.Doc
ehc.wandk5.cn/48462.Doc
ehc.wandk5.cn/00008.Doc
ehc.wandk5.cn/08404.Doc
ehc.wandk5.cn/06868.Doc
ehc.wandk5.cn/48406.Doc
ehc.wandk5.cn/42004.Doc
ehc.wandk5.cn/64242.Doc
ehc.wandk5.cn/88286.Doc
ehc.wandk5.cn/44246.Doc
ehc.wandk5.cn/81826.Doc
ehx.wandk5.cn/64462.Doc
ehx.wandk5.cn/82284.Doc
ehx.wandk5.cn/82646.Doc
ehx.wandk5.cn/66822.Doc
ehx.wandk5.cn/62202.Doc
ehx.wandk5.cn/22862.Doc
ehx.wandk5.cn/84822.Doc
ehx.wandk5.cn/66404.Doc
ehx.wandk5.cn/02606.Doc
ehx.wandk5.cn/02288.Doc
ehz.wandk5.cn/28822.Doc
ehz.wandk5.cn/00680.Doc
ehz.wandk5.cn/88620.Doc
ehz.wandk5.cn/46246.Doc
ehz.wandk5.cn/00228.Doc
ehz.wandk5.cn/26408.Doc
ehz.wandk5.cn/64646.Doc
ehz.wandk5.cn/44468.Doc
ehz.wandk5.cn/68828.Doc
ehz.wandk5.cn/40222.Doc
ehl.wandk5.cn/84828.Doc
ehl.wandk5.cn/82604.Doc
ehl.wandk5.cn/80208.Doc
ehl.wandk5.cn/26688.Doc
ehl.wandk5.cn/46886.Doc
ehl.wandk5.cn/28864.Doc
ehl.wandk5.cn/40284.Doc
ehl.wandk5.cn/00048.Doc
ehl.wandk5.cn/26282.Doc
ehl.wandk5.cn/20242.Doc
ehk.wandk5.cn/44808.Doc
ehk.wandk5.cn/02464.Doc
ehk.wandk5.cn/44202.Doc
ehk.wandk5.cn/24020.Doc
ehk.wandk5.cn/24424.Doc
ehk.wandk5.cn/62682.Doc
ehk.wandk5.cn/00080.Doc
ehk.wandk5.cn/00482.Doc
ehk.wandk5.cn/28048.Doc
ehk.wandk5.cn/22860.Doc
ehj.wandk5.cn/82284.Doc
ehj.wandk5.cn/46666.Doc
ehj.wandk5.cn/24204.Doc
ehj.wandk5.cn/04066.Doc
ehj.wandk5.cn/20886.Doc
ehj.wandk5.cn/60082.Doc
ehj.wandk5.cn/62264.Doc
ehj.wandk5.cn/44048.Doc
ehj.wandk5.cn/68660.Doc
ehj.wandk5.cn/08026.Doc
ehh.wandk5.cn/22080.Doc
ehh.wandk5.cn/06824.Doc
ehh.wandk5.cn/62220.Doc
ehh.wandk5.cn/48424.Doc
ehh.wandk5.cn/24684.Doc
ehh.wandk5.cn/46004.Doc
ehh.wandk5.cn/22042.Doc
ehh.wandk5.cn/04404.Doc
ehh.wandk5.cn/28048.Doc
ehh.wandk5.cn/24666.Doc
ehg.wandk5.cn/86466.Doc
ehg.wandk5.cn/66400.Doc
ehg.wandk5.cn/66068.Doc
ehg.wandk5.cn/82620.Doc
ehg.wandk5.cn/82680.Doc
ehg.wandk5.cn/02026.Doc
ehg.wandk5.cn/86442.Doc
ehg.wandk5.cn/04422.Doc
ehg.wandk5.cn/02026.Doc
ehg.wandk5.cn/44688.Doc
ehf.wandk5.cn/84626.Doc
ehf.wandk5.cn/08028.Doc
ehf.wandk5.cn/26222.Doc
ehf.wandk5.cn/80680.Doc
ehf.wandk5.cn/22640.Doc
ehf.wandk5.cn/04082.Doc
ehf.wandk5.cn/22208.Doc
ehf.wandk5.cn/84082.Doc
ehf.wandk5.cn/60042.Doc
ehf.wandk5.cn/48408.Doc
ehd.wandk5.cn/86846.Doc
ehd.wandk5.cn/26084.Doc
ehd.wandk5.cn/84682.Doc
ehd.wandk5.cn/42844.Doc
ehd.wandk5.cn/42400.Doc
ehd.wandk5.cn/24460.Doc
ehd.wandk5.cn/48860.Doc
ehd.wandk5.cn/84642.Doc
ehd.wandk5.cn/20422.Doc
ehd.wandk5.cn/66286.Doc
ehs.wandk5.cn/84000.Doc
ehs.wandk5.cn/64022.Doc
ehs.wandk5.cn/46622.Doc
ehs.wandk5.cn/64408.Doc
ehs.wandk5.cn/00822.Doc
ehs.wandk5.cn/08848.Doc
ehs.wandk5.cn/66866.Doc
ehs.wandk5.cn/22222.Doc
ehs.wandk5.cn/80060.Doc
ehs.wandk5.cn/02664.Doc
eha.wandk5.cn/86886.Doc
eha.wandk5.cn/48264.Doc
eha.wandk5.cn/06682.Doc
eha.wandk5.cn/26260.Doc
eha.wandk5.cn/22426.Doc
eha.wandk5.cn/62488.Doc
eha.wandk5.cn/24820.Doc
eha.wandk5.cn/42646.Doc
eha.wandk5.cn/22204.Doc
eha.wandk5.cn/20424.Doc
ehp.wandk5.cn/60408.Doc
ehp.wandk5.cn/60604.Doc
ehp.wandk5.cn/24242.Doc
ehp.wandk5.cn/66828.Doc
ehp.wandk5.cn/54628.Doc
ehp.wandk5.cn/80260.Doc
ehp.wandk5.cn/44426.Doc
ehp.wandk5.cn/06622.Doc
ehp.wandk5.cn/62608.Doc
ehp.wandk5.cn/68600.Doc
