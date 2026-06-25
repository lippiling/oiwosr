# Python Trampoline 与递归优化 —— 打破递归限制
# Python 不支持尾递归优化，Trampoline 模式将递归转化为循环

import sys
from functools import wraps, partial

# 1. 递归限制问题
def factorial_recursive(n):
    """普通递归阶乘，受限于调用栈深度"""
    if n <= 1: return 1
    return n * factorial_recursive(n - 1)

print("默认递归限制:", sys.getrecursionlimit())
try:
    factorial_recursive(2000)
except RecursionError as e:
    print("递归超出限制:", e)

# 2. Trampoline 基础模式
def trampoline(func):
    """蹦床函数：反复调用返回的函数直到返回非函数值"""
    result = func
    while callable(result):
        result = result()
    return result

def factorial_trampoline(n, acc=1):
    """蹦床版阶乘 —— 返回函数而非递归调用"""
    if n <= 1: return acc
    return lambda: factorial_trampoline(n - 1, n * acc)

print("蹦床阶乘(10):", trampoline(factorial_trampoline(10)))
print("蹦床阶乘(5000) 位数:", len(str(trampoline(factorial_trampoline(5000)))))

# 3. 通用蹦床装饰器
def trampoline_decorator(func):
    """自动将递归函数转换为栈安全版本"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        while callable(result):
            result = result()
        return result
    return wrapper

@trampoline_decorator
def even(n):
    """判断偶数（相互递归）"""
    return True if n == 0 else lambda: odd(n - 1)

@trampoline_decorator
def odd(n):
    """判断奇数（与 even 相互递归）"""
    return False if n == 0 else lambda: even(n - 1)

print("1000 是偶数?", even(1000))
print("100000 是偶数?", even(100000))  # 栈安全

# 4. 使用 partial 实现蹦床（比 lambda 更高效）
@trampoline_decorator
def fibonacci(n, a=0, b=1):
    """斐波那契数列的蹦床版本"""
    if n == 0: return a
    if n == 1: return b
    return partial(fibonacci, n - 1, b, a + b)

print("斐波那契(10):", fibonacci(10))
print("斐波那契(100):", fibonacci(100))

# 5. 递归转迭代（显式栈）
def tree_depth_iterative(root):
    """树的深度计算 —— 使用显式栈替代递归"""
    if root is None: return 0
    stack = [(root, 1)]
    max_depth = 0
    while stack:
        node, depth = stack.pop()
        if node is not None:
            max_depth = max(max_depth, depth)
            if hasattr(node, 'right'):
                stack.append((node.right, depth + 1))
            if hasattr(node, 'left'):
                stack.append((node.left, depth + 1))
    return max_depth

class TreeNode:
    def __init__(self, v, left=None, right=None):
        self.value = v; self.left = left; self.right = right

tree = TreeNode(1,
    TreeNode(2, TreeNode(4)),
    TreeNode(3, None, TreeNode(5, None, TreeNode(6))))
print("树深度（迭代）:", tree_depth_iterative(tree))

# 6. 三种技术对比
"""
sys.setrecursionlimit() —— 临时提升限制，治标不治本
Trampoline —— 保持递归风格，但每次"弹跳"有函数调用开销
递归转迭代 —— 性能最优，但代码复杂度增加
"""
import time

def bench(desc, func):
    start = time.perf_counter()
    result = func()
    print(f"{desc}: {result}, 耗时 {time.perf_counter()-start:.4f}s")

def fib_iterative(n):
    """迭代版斐波那契（最高效）"""
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a

bench("蹦床fib(1000)", lambda: trampoline(fibonacci(1000)))
bench("迭代fib(1000)", lambda: fib_iterative(1000))

eix.xkzjnc.cn/68062.Doc
eix.xkzjnc.cn/08264.Doc
eix.xkzjnc.cn/42488.Doc
eix.xkzjnc.cn/26202.Doc
eix.xkzjnc.cn/28002.Doc
eix.xkzjnc.cn/44026.Doc
eix.xkzjnc.cn/42802.Doc
eix.xkzjnc.cn/64640.Doc
eix.xkzjnc.cn/42048.Doc
eix.xkzjnc.cn/46800.Doc
eiz.xkzjnc.cn/44402.Doc
eiz.xkzjnc.cn/88462.Doc
eiz.xkzjnc.cn/64422.Doc
eiz.xkzjnc.cn/68626.Doc
eiz.xkzjnc.cn/86882.Doc
eiz.xkzjnc.cn/00208.Doc
eiz.xkzjnc.cn/40628.Doc
eiz.xkzjnc.cn/26822.Doc
eiz.xkzjnc.cn/44826.Doc
eiz.xkzjnc.cn/88662.Doc
eil.xkzjnc.cn/44468.Doc
eil.xkzjnc.cn/64640.Doc
eil.xkzjnc.cn/04240.Doc
eil.xkzjnc.cn/86866.Doc
eil.xkzjnc.cn/28200.Doc
eil.xkzjnc.cn/42420.Doc
eil.xkzjnc.cn/84864.Doc
eil.xkzjnc.cn/42844.Doc
eil.xkzjnc.cn/66866.Doc
eil.xkzjnc.cn/02240.Doc
eik.xkzjnc.cn/22866.Doc
eik.xkzjnc.cn/02242.Doc
eik.xkzjnc.cn/86048.Doc
eik.xkzjnc.cn/00844.Doc
eik.xkzjnc.cn/46460.Doc
eik.xkzjnc.cn/86820.Doc
eik.xkzjnc.cn/68024.Doc
eik.xkzjnc.cn/44224.Doc
eik.xkzjnc.cn/44460.Doc
eik.xkzjnc.cn/00242.Doc
eij.xkzjnc.cn/28006.Doc
eij.xkzjnc.cn/00044.Doc
eij.xkzjnc.cn/80648.Doc
eij.xkzjnc.cn/60668.Doc
eij.xkzjnc.cn/86000.Doc
eij.xkzjnc.cn/86442.Doc
eij.xkzjnc.cn/86622.Doc
eij.xkzjnc.cn/04084.Doc
eij.xkzjnc.cn/24226.Doc
eij.xkzjnc.cn/80044.Doc
eih.xkzjnc.cn/84028.Doc
eih.xkzjnc.cn/40000.Doc
eih.xkzjnc.cn/62020.Doc
eih.xkzjnc.cn/02444.Doc
eih.xkzjnc.cn/86682.Doc
eih.xkzjnc.cn/04422.Doc
eih.xkzjnc.cn/84028.Doc
eih.xkzjnc.cn/68593.Doc
eih.xkzjnc.cn/00845.Doc
eih.xkzjnc.cn/64969.Doc
eig.xkzjnc.cn/46682.Doc
eig.xkzjnc.cn/14781.Doc
eig.xkzjnc.cn/15106.Doc
eig.xkzjnc.cn/96722.Doc
eig.xkzjnc.cn/35542.Doc
eig.xkzjnc.cn/40924.Doc
eig.xkzjnc.cn/64153.Doc
eig.xkzjnc.cn/31889.Doc
eig.xkzjnc.cn/27271.Doc
eig.xkzjnc.cn/33814.Doc
eif.xkzjnc.cn/27009.Doc
eif.xkzjnc.cn/37942.Doc
eif.xkzjnc.cn/94873.Doc
eif.xkzjnc.cn/25691.Doc
eif.xkzjnc.cn/56216.Doc
eif.xkzjnc.cn/37984.Doc
eif.xkzjnc.cn/47790.Doc
eif.xkzjnc.cn/90023.Doc
eif.xkzjnc.cn/38971.Doc
eif.xkzjnc.cn/21990.Doc
eid.xkzjnc.cn/16806.Doc
eid.xkzjnc.cn/59142.Doc
eid.xkzjnc.cn/41460.Doc
eid.xkzjnc.cn/18163.Doc
eid.xkzjnc.cn/12326.Doc
eid.xkzjnc.cn/99068.Doc
eid.xkzjnc.cn/42802.Doc
eid.xkzjnc.cn/59247.Doc
eid.xkzjnc.cn/86488.Doc
eid.xkzjnc.cn/20280.Doc
eis.xkzjnc.cn/46848.Doc
eis.xkzjnc.cn/06446.Doc
eis.xkzjnc.cn/86622.Doc
eis.xkzjnc.cn/26200.Doc
eis.xkzjnc.cn/42804.Doc
eis.xkzjnc.cn/28848.Doc
eis.xkzjnc.cn/06602.Doc
eis.xkzjnc.cn/86024.Doc
eis.xkzjnc.cn/66260.Doc
eis.xkzjnc.cn/08822.Doc
eia.xkzjnc.cn/40868.Doc
eia.xkzjnc.cn/40666.Doc
eia.xkzjnc.cn/82022.Doc
eia.xkzjnc.cn/20422.Doc
eia.xkzjnc.cn/44806.Doc
eia.xkzjnc.cn/08286.Doc
eia.xkzjnc.cn/28806.Doc
eia.xkzjnc.cn/08204.Doc
eia.xkzjnc.cn/28064.Doc
eia.xkzjnc.cn/00248.Doc
eip.xkzjnc.cn/62868.Doc
eip.xkzjnc.cn/08882.Doc
eip.xkzjnc.cn/42462.Doc
eip.xkzjnc.cn/00060.Doc
eip.xkzjnc.cn/15759.Doc
eip.xkzjnc.cn/22288.Doc
eip.xkzjnc.cn/68402.Doc
eip.xkzjnc.cn/04640.Doc
eip.xkzjnc.cn/08800.Doc
eip.xkzjnc.cn/26482.Doc
eio.xkzjnc.cn/28020.Doc
eio.xkzjnc.cn/66488.Doc
eio.xkzjnc.cn/91791.Doc
eio.xkzjnc.cn/42006.Doc
eio.xkzjnc.cn/48402.Doc
eio.xkzjnc.cn/48864.Doc
eio.xkzjnc.cn/22888.Doc
eio.xkzjnc.cn/46024.Doc
eio.xkzjnc.cn/26448.Doc
eio.xkzjnc.cn/04648.Doc
eii.xkzjnc.cn/60608.Doc
eii.xkzjnc.cn/33911.Doc
eii.xkzjnc.cn/40488.Doc
eii.xkzjnc.cn/24444.Doc
eii.xkzjnc.cn/71153.Doc
eii.xkzjnc.cn/46682.Doc
eii.xkzjnc.cn/88028.Doc
eii.xkzjnc.cn/44862.Doc
eii.xkzjnc.cn/44884.Doc
eii.xkzjnc.cn/22862.Doc
eiu.xkzjnc.cn/02600.Doc
eiu.xkzjnc.cn/84686.Doc
eiu.xkzjnc.cn/64602.Doc
eiu.xkzjnc.cn/22462.Doc
eiu.xkzjnc.cn/46426.Doc
eiu.xkzjnc.cn/84842.Doc
eiu.xkzjnc.cn/06204.Doc
eiu.xkzjnc.cn/80042.Doc
eiu.xkzjnc.cn/46884.Doc
eiu.xkzjnc.cn/08626.Doc
eiy.xkzjnc.cn/06880.Doc
eiy.xkzjnc.cn/66260.Doc
eiy.xkzjnc.cn/02024.Doc
eiy.xkzjnc.cn/60402.Doc
eiy.xkzjnc.cn/28642.Doc
eiy.xkzjnc.cn/26406.Doc
eiy.xkzjnc.cn/35317.Doc
eiy.xkzjnc.cn/60680.Doc
eiy.xkzjnc.cn/22426.Doc
eiy.xkzjnc.cn/80040.Doc
eit.xkzjnc.cn/80468.Doc
eit.xkzjnc.cn/66604.Doc
eit.xkzjnc.cn/28440.Doc
eit.xkzjnc.cn/40044.Doc
eit.xkzjnc.cn/02446.Doc
eit.xkzjnc.cn/60480.Doc
eit.xkzjnc.cn/51915.Doc
eit.xkzjnc.cn/11731.Doc
eit.xkzjnc.cn/24666.Doc
eit.xkzjnc.cn/82044.Doc
eir.xkzjnc.cn/60822.Doc
eir.xkzjnc.cn/20686.Doc
eir.xkzjnc.cn/35533.Doc
eir.xkzjnc.cn/60846.Doc
eir.xkzjnc.cn/80826.Doc
eir.xkzjnc.cn/84264.Doc
eir.xkzjnc.cn/28022.Doc
eir.xkzjnc.cn/84468.Doc
eir.xkzjnc.cn/77535.Doc
eir.xkzjnc.cn/02820.Doc
eie.xkzjnc.cn/84284.Doc
eie.xkzjnc.cn/59515.Doc
eie.xkzjnc.cn/53971.Doc
eie.xkzjnc.cn/40420.Doc
eie.xkzjnc.cn/42422.Doc
eie.xkzjnc.cn/84244.Doc
eie.xkzjnc.cn/62666.Doc
eie.xkzjnc.cn/80488.Doc
eie.xkzjnc.cn/22848.Doc
eie.xkzjnc.cn/88886.Doc
eiw.xkzjnc.cn/28862.Doc
eiw.xkzjnc.cn/82086.Doc
eiw.xkzjnc.cn/59359.Doc
eiw.xkzjnc.cn/84080.Doc
eiw.xkzjnc.cn/62862.Doc
eiw.xkzjnc.cn/00064.Doc
eiw.xkzjnc.cn/28688.Doc
eiw.xkzjnc.cn/53133.Doc
eiw.xkzjnc.cn/46860.Doc
eiw.xkzjnc.cn/42888.Doc
eiq.xkzjnc.cn/66088.Doc
eiq.xkzjnc.cn/04044.Doc
eiq.xkzjnc.cn/26868.Doc
eiq.xkzjnc.cn/00828.Doc
eiq.xkzjnc.cn/79971.Doc
eiq.xkzjnc.cn/24860.Doc
eiq.xkzjnc.cn/28268.Doc
eiq.xkzjnc.cn/82004.Doc
eiq.xkzjnc.cn/91379.Doc
eiq.xkzjnc.cn/04864.Doc
eum.xkzjnc.cn/06462.Doc
eum.xkzjnc.cn/00446.Doc
eum.xkzjnc.cn/64440.Doc
eum.xkzjnc.cn/68442.Doc
eum.xkzjnc.cn/79959.Doc
eum.xkzjnc.cn/62226.Doc
eum.xkzjnc.cn/28084.Doc
eum.xkzjnc.cn/80824.Doc
eum.xkzjnc.cn/02206.Doc
eum.xkzjnc.cn/40604.Doc
eun.xkzjnc.cn/06862.Doc
eun.xkzjnc.cn/88666.Doc
eun.xkzjnc.cn/44806.Doc
eun.xkzjnc.cn/68224.Doc
eun.xkzjnc.cn/66860.Doc
eun.xkzjnc.cn/42482.Doc
eun.xkzjnc.cn/85255.Doc
eun.xkzjnc.cn/85740.Doc
eun.xkzjnc.cn/56139.Doc
eun.xkzjnc.cn/38928.Doc
eub.xkzjnc.cn/69955.Doc
eub.xkzjnc.cn/77677.Doc
eub.xkzjnc.cn/57842.Doc
eub.xkzjnc.cn/05363.Doc
eub.xkzjnc.cn/06082.Doc
eub.xkzjnc.cn/82359.Doc
eub.xkzjnc.cn/67810.Doc
eub.xkzjnc.cn/13492.Doc
eub.xkzjnc.cn/65632.Doc
eub.xkzjnc.cn/66874.Doc
euv.xkzjnc.cn/96477.Doc
euv.xkzjnc.cn/93766.Doc
euv.xkzjnc.cn/21250.Doc
euv.xkzjnc.cn/58015.Doc
euv.xkzjnc.cn/62446.Doc
euv.xkzjnc.cn/81212.Doc
euv.xkzjnc.cn/48686.Doc
euv.xkzjnc.cn/50381.Doc
euv.xkzjnc.cn/86200.Doc
euv.xkzjnc.cn/25493.Doc
euc.xkzjnc.cn/75548.Doc
euc.xkzjnc.cn/83365.Doc
euc.xkzjnc.cn/66763.Doc
euc.xkzjnc.cn/99055.Doc
euc.xkzjnc.cn/92383.Doc
euc.xkzjnc.cn/51826.Doc
euc.xkzjnc.cn/78071.Doc
euc.xkzjnc.cn/61184.Doc
euc.xkzjnc.cn/26745.Doc
euc.xkzjnc.cn/79705.Doc
eux.xkzjnc.cn/67025.Doc
eux.xkzjnc.cn/47910.Doc
eux.xkzjnc.cn/32544.Doc
eux.xkzjnc.cn/85727.Doc
eux.xkzjnc.cn/56100.Doc
eux.xkzjnc.cn/76290.Doc
eux.xkzjnc.cn/62742.Doc
eux.xkzjnc.cn/74893.Doc
eux.xkzjnc.cn/67960.Doc
eux.xkzjnc.cn/04416.Doc
euz.xkzjnc.cn/85125.Doc
euz.xkzjnc.cn/30002.Doc
euz.xkzjnc.cn/15139.Doc
euz.xkzjnc.cn/58783.Doc
euz.xkzjnc.cn/10235.Doc
euz.xkzjnc.cn/32230.Doc
euz.xkzjnc.cn/04211.Doc
euz.xkzjnc.cn/65554.Doc
euz.xkzjnc.cn/85417.Doc
euz.xkzjnc.cn/54880.Doc
eul.xkzjnc.cn/95685.Doc
eul.xkzjnc.cn/66824.Doc
eul.xkzjnc.cn/74241.Doc
eul.xkzjnc.cn/71909.Doc
eul.xkzjnc.cn/27500.Doc
eul.xkzjnc.cn/99812.Doc
eul.xkzjnc.cn/43423.Doc
eul.xkzjnc.cn/72556.Doc
eul.xkzjnc.cn/80767.Doc
eul.xkzjnc.cn/59544.Doc
euk.xkzjnc.cn/62333.Doc
euk.xkzjnc.cn/00798.Doc
euk.xkzjnc.cn/85538.Doc
euk.xkzjnc.cn/10946.Doc
euk.xkzjnc.cn/96105.Doc
euk.xkzjnc.cn/47566.Doc
euk.xkzjnc.cn/97985.Doc
euk.xkzjnc.cn/92004.Doc
euk.xkzjnc.cn/67893.Doc
euk.xkzjnc.cn/95160.Doc
euj.xkzjnc.cn/39494.Doc
euj.xkzjnc.cn/40657.Doc
euj.xkzjnc.cn/93498.Doc
euj.xkzjnc.cn/68414.Doc
euj.xkzjnc.cn/37779.Doc
euj.xkzjnc.cn/11135.Doc
euj.xkzjnc.cn/46084.Doc
euj.xkzjnc.cn/22026.Doc
euj.xkzjnc.cn/51995.Doc
euj.xkzjnc.cn/31537.Doc
euh.xkzjnc.cn/86486.Doc
euh.xkzjnc.cn/99535.Doc
euh.xkzjnc.cn/08228.Doc
euh.xkzjnc.cn/82200.Doc
euh.xkzjnc.cn/42046.Doc
euh.xkzjnc.cn/88662.Doc
euh.xkzjnc.cn/08686.Doc
euh.xkzjnc.cn/44228.Doc
euh.xkzjnc.cn/44482.Doc
euh.xkzjnc.cn/06286.Doc
eug.xkzjnc.cn/46002.Doc
eug.xkzjnc.cn/66642.Doc
eug.xkzjnc.cn/22864.Doc
eug.xkzjnc.cn/55119.Doc
eug.xkzjnc.cn/22842.Doc
eug.xkzjnc.cn/46606.Doc
eug.xkzjnc.cn/48868.Doc
eug.xkzjnc.cn/08802.Doc
eug.xkzjnc.cn/22224.Doc
eug.xkzjnc.cn/84884.Doc
euf.xkzjnc.cn/02426.Doc
euf.xkzjnc.cn/15199.Doc
euf.xkzjnc.cn/24446.Doc
euf.xkzjnc.cn/06000.Doc
euf.xkzjnc.cn/02622.Doc
euf.xkzjnc.cn/88088.Doc
euf.xkzjnc.cn/22060.Doc
euf.xkzjnc.cn/00220.Doc
euf.xkzjnc.cn/48248.Doc
euf.xkzjnc.cn/88466.Doc
eud.xkzjnc.cn/46266.Doc
eud.xkzjnc.cn/64844.Doc
eud.xkzjnc.cn/42286.Doc
eud.xkzjnc.cn/02020.Doc
eud.xkzjnc.cn/57399.Doc
eud.xkzjnc.cn/26082.Doc
eud.xkzjnc.cn/24802.Doc
eud.xkzjnc.cn/15117.Doc
eud.xkzjnc.cn/40882.Doc
eud.xkzjnc.cn/26642.Doc
eus.xkzjnc.cn/22404.Doc
eus.xkzjnc.cn/02486.Doc
eus.xkzjnc.cn/04606.Doc
eus.xkzjnc.cn/40824.Doc
eus.xkzjnc.cn/88888.Doc
eus.xkzjnc.cn/22828.Doc
eus.xkzjnc.cn/84640.Doc
eus.xkzjnc.cn/04444.Doc
eus.xkzjnc.cn/46826.Doc
eus.xkzjnc.cn/48422.Doc
eua.xkzjnc.cn/20882.Doc
eua.xkzjnc.cn/42482.Doc
eua.xkzjnc.cn/46002.Doc
eua.xkzjnc.cn/86626.Doc
eua.xkzjnc.cn/02022.Doc
eua.xkzjnc.cn/60060.Doc
eua.xkzjnc.cn/88824.Doc
eua.xkzjnc.cn/22204.Doc
eua.xkzjnc.cn/88442.Doc
eua.xkzjnc.cn/82422.Doc
eup.xkzjnc.cn/42228.Doc
eup.xkzjnc.cn/28666.Doc
eup.xkzjnc.cn/24688.Doc
eup.xkzjnc.cn/86824.Doc
eup.xkzjnc.cn/64284.Doc
eup.xkzjnc.cn/40044.Doc
eup.xkzjnc.cn/55579.Doc
eup.xkzjnc.cn/20880.Doc
eup.xkzjnc.cn/82068.Doc
eup.xkzjnc.cn/08248.Doc
euo.xkzjnc.cn/35731.Doc
euo.xkzjnc.cn/42882.Doc
euo.xkzjnc.cn/24000.Doc
euo.xkzjnc.cn/60824.Doc
euo.xkzjnc.cn/86866.Doc
euo.xkzjnc.cn/60624.Doc
euo.xkzjnc.cn/46420.Doc
euo.xkzjnc.cn/86000.Doc
euo.xkzjnc.cn/60000.Doc
euo.xkzjnc.cn/66226.Doc
eui.xkzjnc.cn/20026.Doc
eui.xkzjnc.cn/44040.Doc
eui.xkzjnc.cn/00448.Doc
eui.xkzjnc.cn/20606.Doc
eui.xkzjnc.cn/28862.Doc
eui.xkzjnc.cn/26640.Doc
eui.xkzjnc.cn/26202.Doc
eui.xkzjnc.cn/26408.Doc
eui.xkzjnc.cn/62006.Doc
eui.xkzjnc.cn/99953.Doc
euu.xkzjnc.cn/88226.Doc
euu.xkzjnc.cn/86440.Doc
euu.xkzjnc.cn/46602.Doc
euu.xkzjnc.cn/26486.Doc
euu.xkzjnc.cn/48000.Doc
euu.xkzjnc.cn/20282.Doc
euu.xkzjnc.cn/46244.Doc
euu.xkzjnc.cn/66626.Doc
euu.xkzjnc.cn/00824.Doc
euu.xkzjnc.cn/46424.Doc
euy.xkzjnc.cn/82486.Doc
euy.xkzjnc.cn/80424.Doc
euy.xkzjnc.cn/60446.Doc
euy.xkzjnc.cn/26688.Doc
euy.xkzjnc.cn/60280.Doc
euy.xkzjnc.cn/46244.Doc
euy.xkzjnc.cn/80468.Doc
euy.xkzjnc.cn/86880.Doc
euy.xkzjnc.cn/91977.Doc
euy.xkzjnc.cn/06862.Doc
eut.xkzjnc.cn/68480.Doc
eut.xkzjnc.cn/42400.Doc
eut.xkzjnc.cn/68028.Doc
eut.xkzjnc.cn/40808.Doc
eut.xkzjnc.cn/24840.Doc
eut.xkzjnc.cn/06408.Doc
eut.xkzjnc.cn/86266.Doc
eut.xkzjnc.cn/40688.Doc
eut.xkzjnc.cn/60608.Doc
eut.xkzjnc.cn/79319.Doc
eur.xkzjnc.cn/20864.Doc
eur.xkzjnc.cn/44484.Doc
eur.xkzjnc.cn/77531.Doc
eur.xkzjnc.cn/04620.Doc
eur.xkzjnc.cn/40000.Doc
eur.xkzjnc.cn/42440.Doc
eur.xkzjnc.cn/46804.Doc
eur.xkzjnc.cn/06842.Doc
eur.xkzjnc.cn/44082.Doc
eur.xkzjnc.cn/79917.Doc
eue.xkzjnc.cn/73911.Doc
eue.xkzjnc.cn/95553.Doc
eue.xkzjnc.cn/26408.Doc
eue.xkzjnc.cn/24426.Doc
eue.xkzjnc.cn/64866.Doc
eue.xkzjnc.cn/84082.Doc
eue.xkzjnc.cn/62622.Doc
eue.xkzjnc.cn/37537.Doc
eue.xkzjnc.cn/84884.Doc
eue.xkzjnc.cn/82428.Doc
euw.xkzjnc.cn/68642.Doc
euw.xkzjnc.cn/82462.Doc
euw.xkzjnc.cn/26804.Doc
euw.xkzjnc.cn/53599.Doc
euw.xkzjnc.cn/71315.Doc
euw.xkzjnc.cn/48280.Doc
euw.xkzjnc.cn/28668.Doc
euw.xkzjnc.cn/40260.Doc
euw.xkzjnc.cn/62482.Doc
euw.xkzjnc.cn/28082.Doc
euq.xkzjnc.cn/04628.Doc
euq.xkzjnc.cn/19959.Doc
euq.xkzjnc.cn/28684.Doc
euq.xkzjnc.cn/42604.Doc
euq.xkzjnc.cn/64064.Doc
euq.xkzjnc.cn/28802.Doc
euq.xkzjnc.cn/60080.Doc
euq.xkzjnc.cn/20068.Doc
euq.xkzjnc.cn/40008.Doc
euq.xkzjnc.cn/66668.Doc
eym.xkzjnc.cn/64246.Doc
eym.xkzjnc.cn/06684.Doc
eym.xkzjnc.cn/46606.Doc
eym.xkzjnc.cn/00064.Doc
eym.xkzjnc.cn/26486.Doc
eym.xkzjnc.cn/66244.Doc
eym.xkzjnc.cn/20622.Doc
eym.xkzjnc.cn/20862.Doc
eym.xkzjnc.cn/64444.Doc
eym.xkzjnc.cn/75333.Doc
eyn.xkzjnc.cn/64408.Doc
eyn.xkzjnc.cn/15379.Doc
eyn.xkzjnc.cn/44604.Doc
eyn.xkzjnc.cn/00062.Doc
eyn.xkzjnc.cn/06688.Doc
eyn.xkzjnc.cn/86844.Doc
eyn.xkzjnc.cn/00242.Doc
eyn.xkzjnc.cn/04048.Doc
eyn.xkzjnc.cn/62044.Doc
eyn.xkzjnc.cn/60066.Doc
