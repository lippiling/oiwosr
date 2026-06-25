Python内存管理与性能优化

一、内存模型

Python中一切皆对象，每个对象有引用计数、类型指针和值。

import sys
print(sys.getsizeof(0))       # 28 bytes
print(sys.getsizeof([]))      # 56 bytes
print(sys.getsizeof({}))      # 64 bytes
print(sys.getsizeof("hello")) # 54 bytes


二、引用计数与GC

import sys
a = [1, 2, 3]
print(sys.getrefcount(a))  # 2（a本身 + getrefcount参数）

# 循环引用
import gc
gc.collect()  # 手动触发垃圾回收

# 弱引用
import weakref

class Cache:
    def __init__(self):
        self._data = weakref.WeakValueDictionary()
    def get(self, key, factory):
        if key in self._data:
            return self._data[key]
        value = factory()
        self._data[key] = value
        return value


三、内存优化

3.1 __slots__

class Point:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

# 减少约70%内存（无__dict__）

3.2 生成器替代列表

# 大文件逐行处理
with open('large_file.txt') as f:
    for line in f:  # 文件对象本身是迭代器
        process(line)

3.3 array模块

import array
# list: 每元素8字节（指针）
# array('i'): 每元素4字节（整数值）


四、性能分析

import timeit

# 比较性能
t1 = timeit.timeit("[x**2 for x in range(1000)]", number=10000)
t2 = timeit.timeit("list(map(lambda x:x**2, range(1000)))", number=10000)

# cProfile
import cProfile, pstats
profiler = cProfile.Profile()
profiler.enable()
heavy_function()
profiler.disable()
pstats.Stats(profiler).sort_stats('cumulative').print_stats(10)


五、常见优化

5.1 字符串拼接

# 慢
s = ""
for item in items:
    s += str(item)

# 快
s = ",".join(str(item) for item in items)

5.2 查找优化

# list: O(n)  set/dict: O(1)
large_set = set(range(100000))
99999 in large_set  # 快

5.3 缓存

from functools import lru_cache

@lru_cache(maxsize=256)
def expensive(n):
    return n ** 2

5.4 局部变量

# 缓存内置函数和属性
def fast(items):
    local_len = len
    result = []
    for item in items:
        if local_len(item) > 3:
            result.append(item)
    return result


六、内存泄漏排查

import tracemalloc

tracemalloc.start()
# ... 执行代码 ...
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')
for stat in top_stats[:5]:
    print(stat)


七、优化原则

1. 先测量后优化（不要过早优化）
2. 优化热点代码（20%代码消耗80%时间）
3. 选择合适的数据结构是最大的优化
4. 利用标准库（C实现的函数)
5. 必要时用numpy/Cython/C扩展

wll.irampL.cn/00206.Doc
wll.irampL.cn/22846.Doc
wll.irampL.cn/37593.Doc
wll.irampL.cn/28440.Doc
wll.irampL.cn/66042.Doc
wll.irampL.cn/08426.Doc
wll.irampL.cn/22462.Doc
wll.irampL.cn/24804.Doc
wll.irampL.cn/00284.Doc
wll.irampL.cn/62222.Doc
wlk.irampL.cn/86408.Doc
wlk.irampL.cn/66828.Doc
wlk.irampL.cn/84422.Doc
wlk.irampL.cn/06646.Doc
wlk.irampL.cn/82620.Doc
wlk.irampL.cn/82200.Doc
wlk.irampL.cn/86048.Doc
wlk.irampL.cn/28860.Doc
wlk.irampL.cn/80242.Doc
wlk.irampL.cn/64426.Doc
wlj.irampL.cn/46280.Doc
wlj.irampL.cn/19173.Doc
wlj.irampL.cn/26000.Doc
wlj.irampL.cn/28868.Doc
wlj.irampL.cn/08486.Doc
wlj.irampL.cn/66802.Doc
wlj.irampL.cn/66640.Doc
wlj.irampL.cn/88022.Doc
wlj.irampL.cn/00202.Doc
wlj.irampL.cn/88460.Doc
wlh.irampL.cn/02408.Doc
wlh.irampL.cn/95753.Doc
wlh.irampL.cn/68488.Doc
wlh.irampL.cn/44022.Doc
wlh.irampL.cn/66046.Doc
wlh.irampL.cn/00602.Doc
wlh.irampL.cn/20606.Doc
wlh.irampL.cn/46426.Doc
wlh.irampL.cn/00484.Doc
wlh.irampL.cn/28246.Doc
wlg.irampL.cn/82046.Doc
wlg.irampL.cn/06688.Doc
wlg.irampL.cn/64826.Doc
wlg.irampL.cn/71731.Doc
wlg.irampL.cn/00604.Doc
wlg.irampL.cn/68060.Doc
wlg.irampL.cn/08884.Doc
wlg.irampL.cn/02046.Doc
wlg.irampL.cn/82866.Doc
wlg.irampL.cn/37991.Doc
wlf.irampL.cn/66062.Doc
wlf.irampL.cn/60482.Doc
wlf.irampL.cn/86842.Doc
wlf.irampL.cn/44408.Doc
wlf.irampL.cn/46000.Doc
wlf.irampL.cn/77197.Doc
wlf.irampL.cn/04264.Doc
wlf.irampL.cn/88664.Doc
wlf.irampL.cn/42660.Doc
wlf.irampL.cn/86846.Doc
wld.irampL.cn/88866.Doc
wld.irampL.cn/91339.Doc
wld.irampL.cn/48024.Doc
wld.irampL.cn/51317.Doc
wld.irampL.cn/68266.Doc
wld.irampL.cn/04204.Doc
wld.irampL.cn/24680.Doc
wld.irampL.cn/17737.Doc
wld.irampL.cn/62846.Doc
wld.irampL.cn/82228.Doc
wls.irampL.cn/26006.Doc
wls.irampL.cn/62000.Doc
wls.irampL.cn/06268.Doc
wls.irampL.cn/84482.Doc
wls.irampL.cn/64080.Doc
wls.irampL.cn/00244.Doc
wls.irampL.cn/80480.Doc
wls.irampL.cn/88646.Doc
wls.irampL.cn/44042.Doc
wls.irampL.cn/88828.Doc
wla.irampL.cn/46840.Doc
wla.irampL.cn/28002.Doc
wla.irampL.cn/84684.Doc
wla.irampL.cn/04244.Doc
wla.irampL.cn/84866.Doc
wla.irampL.cn/04806.Doc
wla.irampL.cn/55757.Doc
wla.irampL.cn/00484.Doc
wla.irampL.cn/48866.Doc
wla.irampL.cn/86820.Doc
wlp.irampL.cn/48200.Doc
wlp.irampL.cn/06024.Doc
wlp.irampL.cn/44600.Doc
wlp.irampL.cn/88488.Doc
wlp.irampL.cn/40686.Doc
wlp.irampL.cn/22004.Doc
wlp.irampL.cn/62424.Doc
wlp.irampL.cn/55375.Doc
wlp.irampL.cn/97757.Doc
wlp.irampL.cn/88248.Doc
wlo.irampL.cn/68028.Doc
wlo.irampL.cn/44422.Doc
wlo.irampL.cn/37131.Doc
wlo.irampL.cn/60424.Doc
wlo.irampL.cn/60666.Doc
wlo.irampL.cn/86242.Doc
wlo.irampL.cn/26842.Doc
wlo.irampL.cn/62224.Doc
wlo.irampL.cn/60888.Doc
wlo.irampL.cn/82408.Doc
wli.irampL.cn/20484.Doc
wli.irampL.cn/46040.Doc
wli.irampL.cn/35131.Doc
wli.irampL.cn/00022.Doc
wli.irampL.cn/08684.Doc
wli.irampL.cn/02468.Doc
wli.irampL.cn/66460.Doc
wli.irampL.cn/68466.Doc
wli.irampL.cn/02484.Doc
wli.irampL.cn/82444.Doc
wlu.irampL.cn/02608.Doc
wlu.irampL.cn/66024.Doc
wlu.irampL.cn/28404.Doc
wlu.irampL.cn/51193.Doc
wlu.irampL.cn/64464.Doc
wlu.irampL.cn/26444.Doc
wlu.irampL.cn/64082.Doc
wlu.irampL.cn/93537.Doc
wlu.irampL.cn/60602.Doc
wlu.irampL.cn/46208.Doc
wly.irampL.cn/22284.Doc
wly.irampL.cn/15119.Doc
wly.irampL.cn/04228.Doc
wly.irampL.cn/24266.Doc
wly.irampL.cn/62026.Doc
wly.irampL.cn/02824.Doc
wly.irampL.cn/84826.Doc
wly.irampL.cn/46888.Doc
wly.irampL.cn/28800.Doc
wly.irampL.cn/02204.Doc
wlt.irampL.cn/04684.Doc
wlt.irampL.cn/64644.Doc
wlt.irampL.cn/80084.Doc
wlt.irampL.cn/20660.Doc
wlt.irampL.cn/22064.Doc
wlt.irampL.cn/20280.Doc
wlt.irampL.cn/08888.Doc
wlt.irampL.cn/46662.Doc
wlt.irampL.cn/28680.Doc
wlt.irampL.cn/22240.Doc
wlr.irampL.cn/82066.Doc
wlr.irampL.cn/62062.Doc
wlr.irampL.cn/66244.Doc
wlr.irampL.cn/68264.Doc
wlr.irampL.cn/86824.Doc
wlr.irampL.cn/22048.Doc
wlr.irampL.cn/80448.Doc
wlr.irampL.cn/08440.Doc
wlr.irampL.cn/42426.Doc
wlr.irampL.cn/06422.Doc
wle.irampL.cn/04264.Doc
wle.irampL.cn/62086.Doc
wle.irampL.cn/82282.Doc
wle.irampL.cn/24682.Doc
wle.irampL.cn/88400.Doc
wle.irampL.cn/28248.Doc
wle.irampL.cn/93911.Doc
wle.irampL.cn/86486.Doc
wle.irampL.cn/42466.Doc
wle.irampL.cn/86462.Doc
wlw.irampL.cn/88822.Doc
wlw.irampL.cn/08028.Doc
wlw.irampL.cn/28460.Doc
wlw.irampL.cn/08806.Doc
wlw.irampL.cn/46484.Doc
wlw.irampL.cn/97397.Doc
wlw.irampL.cn/82004.Doc
wlw.irampL.cn/95515.Doc
wlw.irampL.cn/24022.Doc
wlw.irampL.cn/46644.Doc
wlq.irampL.cn/20206.Doc
wlq.irampL.cn/60280.Doc
wlq.irampL.cn/20480.Doc
wlq.irampL.cn/20886.Doc
wlq.irampL.cn/40286.Doc
wlq.irampL.cn/57311.Doc
wlq.irampL.cn/28200.Doc
wlq.irampL.cn/48028.Doc
wlq.irampL.cn/82808.Doc
wlq.irampL.cn/00628.Doc
wkm.irampL.cn/86680.Doc
wkm.irampL.cn/22026.Doc
wkm.irampL.cn/00840.Doc
wkm.irampL.cn/80844.Doc
wkm.irampL.cn/86824.Doc
wkm.irampL.cn/26828.Doc
wkm.irampL.cn/66606.Doc
wkm.irampL.cn/42260.Doc
wkm.irampL.cn/24240.Doc
wkm.irampL.cn/02244.Doc
wkn.irampL.cn/62424.Doc
wkn.irampL.cn/46246.Doc
wkn.irampL.cn/06662.Doc
wkn.irampL.cn/26802.Doc
wkn.irampL.cn/88400.Doc
wkn.irampL.cn/37711.Doc
wkn.irampL.cn/00668.Doc
wkn.irampL.cn/40864.Doc
wkn.irampL.cn/42202.Doc
wkn.irampL.cn/66862.Doc
wkb.irampL.cn/84408.Doc
wkb.irampL.cn/88066.Doc
wkb.irampL.cn/11599.Doc
wkb.irampL.cn/20004.Doc
wkb.irampL.cn/40684.Doc
wkb.irampL.cn/64808.Doc
wkb.irampL.cn/00462.Doc
wkb.irampL.cn/20400.Doc
wkb.irampL.cn/44848.Doc
wkb.irampL.cn/62840.Doc
wkv.irampL.cn/22444.Doc
wkv.irampL.cn/20886.Doc
wkv.irampL.cn/04824.Doc
wkv.irampL.cn/42484.Doc
wkv.irampL.cn/40404.Doc
wkv.irampL.cn/82604.Doc
wkv.irampL.cn/08622.Doc
wkv.irampL.cn/60444.Doc
wkv.irampL.cn/04464.Doc
wkv.irampL.cn/60640.Doc
wkc.irampL.cn/86668.Doc
wkc.irampL.cn/20662.Doc
wkc.irampL.cn/28000.Doc
wkc.irampL.cn/84066.Doc
wkc.irampL.cn/82842.Doc
wkc.irampL.cn/40006.Doc
wkc.irampL.cn/88680.Doc
wkc.irampL.cn/02824.Doc
wkc.irampL.cn/40668.Doc
wkc.irampL.cn/26044.Doc
wkx.irampL.cn/82084.Doc
wkx.irampL.cn/24806.Doc
wkx.irampL.cn/08402.Doc
wkx.irampL.cn/20204.Doc
wkx.irampL.cn/40044.Doc
wkx.irampL.cn/42820.Doc
wkx.irampL.cn/84662.Doc
wkx.irampL.cn/24860.Doc
wkx.irampL.cn/66802.Doc
wkx.irampL.cn/06246.Doc
wkz.irampL.cn/86442.Doc
wkz.irampL.cn/42286.Doc
wkz.irampL.cn/20624.Doc
wkz.irampL.cn/48604.Doc
wkz.irampL.cn/26488.Doc
wkz.irampL.cn/80422.Doc
wkz.irampL.cn/35519.Doc
wkz.irampL.cn/48840.Doc
wkz.irampL.cn/48282.Doc
wkz.irampL.cn/84208.Doc
wkl.irampL.cn/66882.Doc
wkl.irampL.cn/22882.Doc
wkl.irampL.cn/84488.Doc
wkl.irampL.cn/44240.Doc
wkl.irampL.cn/26846.Doc
wkl.irampL.cn/62208.Doc
wkl.irampL.cn/00280.Doc
wkl.irampL.cn/86020.Doc
wkl.irampL.cn/64804.Doc
wkl.irampL.cn/48820.Doc
wkk.irampL.cn/68420.Doc
wkk.irampL.cn/40804.Doc
wkk.irampL.cn/64444.Doc
wkk.irampL.cn/48404.Doc
wkk.irampL.cn/48264.Doc
wkk.irampL.cn/31115.Doc
wkk.irampL.cn/91935.Doc
wkk.irampL.cn/28680.Doc
wkk.irampL.cn/82002.Doc
wkk.irampL.cn/64606.Doc
wkj.irampL.cn/80268.Doc
wkj.irampL.cn/82008.Doc
wkj.irampL.cn/04284.Doc
wkj.irampL.cn/24046.Doc
wkj.irampL.cn/26844.Doc
wkj.irampL.cn/84460.Doc
wkj.irampL.cn/44602.Doc
wkj.irampL.cn/77397.Doc
wkj.irampL.cn/28084.Doc
wkj.irampL.cn/46886.Doc
wkh.irampL.cn/99537.Doc
wkh.irampL.cn/48862.Doc
wkh.irampL.cn/80404.Doc
wkh.irampL.cn/44880.Doc
wkh.irampL.cn/75933.Doc
wkh.irampL.cn/06060.Doc
wkh.irampL.cn/48622.Doc
wkh.irampL.cn/51115.Doc
wkh.irampL.cn/22082.Doc
wkh.irampL.cn/28608.Doc
wkg.irampL.cn/93933.Doc
wkg.irampL.cn/20262.Doc
wkg.irampL.cn/06402.Doc
wkg.irampL.cn/24686.Doc
wkg.irampL.cn/00802.Doc
wkg.irampL.cn/31557.Doc
wkg.irampL.cn/42244.Doc
wkg.irampL.cn/80664.Doc
wkg.irampL.cn/22488.Doc
wkg.irampL.cn/28046.Doc
wkf.irampL.cn/20000.Doc
wkf.irampL.cn/42408.Doc
wkf.irampL.cn/00220.Doc
wkf.irampL.cn/20668.Doc
wkf.irampL.cn/62686.Doc
wkf.irampL.cn/00208.Doc
wkf.irampL.cn/64086.Doc
wkf.irampL.cn/62604.Doc
wkf.irampL.cn/59713.Doc
wkf.irampL.cn/08448.Doc
wkd.irampL.cn/24068.Doc
wkd.irampL.cn/40422.Doc
wkd.irampL.cn/73591.Doc
wkd.irampL.cn/68840.Doc
wkd.irampL.cn/40882.Doc
wkd.irampL.cn/60648.Doc
wkd.irampL.cn/24488.Doc
wkd.irampL.cn/11797.Doc
wkd.irampL.cn/82826.Doc
wkd.irampL.cn/80648.Doc
wks.irampL.cn/60624.Doc
wks.irampL.cn/06068.Doc
wks.irampL.cn/08440.Doc
wks.irampL.cn/06042.Doc
wks.irampL.cn/06848.Doc
wks.irampL.cn/44044.Doc
wks.irampL.cn/24642.Doc
wks.irampL.cn/48008.Doc
wks.irampL.cn/22008.Doc
wks.irampL.cn/66044.Doc
wka.irampL.cn/04806.Doc
wka.irampL.cn/80644.Doc
wka.irampL.cn/51139.Doc
wka.irampL.cn/82202.Doc
wka.irampL.cn/97517.Doc
wka.irampL.cn/73391.Doc
wka.irampL.cn/88666.Doc
wka.irampL.cn/62604.Doc
wka.irampL.cn/86068.Doc
wka.irampL.cn/22488.Doc
wkp.irampL.cn/06288.Doc
wkp.irampL.cn/20206.Doc
wkp.irampL.cn/15155.Doc
wkp.irampL.cn/22224.Doc
wkp.irampL.cn/64624.Doc
wkp.irampL.cn/42646.Doc
wkp.irampL.cn/28288.Doc
wkp.irampL.cn/17795.Doc
wkp.irampL.cn/00484.Doc
wkp.irampL.cn/46840.Doc
wko.irampL.cn/66802.Doc
wko.irampL.cn/44246.Doc
wko.irampL.cn/44820.Doc
wko.irampL.cn/46284.Doc
wko.irampL.cn/51999.Doc
wko.irampL.cn/60002.Doc
wko.irampL.cn/80802.Doc
wko.irampL.cn/62644.Doc
wko.irampL.cn/04224.Doc
wko.irampL.cn/48228.Doc
wki.irampL.cn/64806.Doc
wki.irampL.cn/82684.Doc
wki.irampL.cn/86006.Doc
wki.irampL.cn/28466.Doc
wki.irampL.cn/06424.Doc
wki.irampL.cn/08082.Doc
wki.irampL.cn/46688.Doc
wki.irampL.cn/17515.Doc
wki.irampL.cn/26026.Doc
wki.irampL.cn/22044.Doc
wku.irampL.cn/57131.Doc
wku.irampL.cn/84808.Doc
wku.irampL.cn/88220.Doc
wku.irampL.cn/33153.Doc
wku.irampL.cn/46868.Doc
wku.irampL.cn/68822.Doc
wku.irampL.cn/42422.Doc
wku.irampL.cn/06048.Doc
wku.irampL.cn/28222.Doc
wku.irampL.cn/46620.Doc
wky.irampL.cn/86482.Doc
wky.irampL.cn/44266.Doc
wky.irampL.cn/88000.Doc
wky.irampL.cn/62424.Doc
wky.irampL.cn/46062.Doc
wky.irampL.cn/60684.Doc
wky.irampL.cn/26224.Doc
wky.irampL.cn/04464.Doc
wky.irampL.cn/00204.Doc
wky.irampL.cn/00084.Doc
wkt.irampL.cn/20646.Doc
wkt.irampL.cn/42846.Doc
wkt.irampL.cn/40840.Doc
wkt.irampL.cn/06002.Doc
wkt.irampL.cn/22648.Doc
wkt.irampL.cn/64028.Doc
wkt.irampL.cn/82208.Doc
wkt.irampL.cn/82866.Doc
wkt.irampL.cn/17139.Doc
wkt.irampL.cn/00888.Doc
wkr.irampL.cn/02028.Doc
wkr.irampL.cn/02608.Doc
wkr.irampL.cn/64200.Doc
wkr.irampL.cn/08600.Doc
wkr.irampL.cn/62826.Doc
wkr.irampL.cn/88666.Doc
wkr.irampL.cn/24866.Doc
wkr.irampL.cn/66686.Doc
wkr.irampL.cn/08808.Doc
wkr.irampL.cn/00268.Doc
wke.irampL.cn/08848.Doc
wke.irampL.cn/22022.Doc
wke.irampL.cn/64246.Doc
wke.irampL.cn/40280.Doc
wke.irampL.cn/44682.Doc
wke.irampL.cn/44448.Doc
wke.irampL.cn/80802.Doc
wke.irampL.cn/04028.Doc
wke.irampL.cn/68886.Doc
wke.irampL.cn/84086.Doc
wkw.irampL.cn/42006.Doc
wkw.irampL.cn/08040.Doc
wkw.irampL.cn/11791.Doc
wkw.irampL.cn/86886.Doc
wkw.irampL.cn/02262.Doc
wkw.irampL.cn/00400.Doc
wkw.irampL.cn/99795.Doc
wkw.irampL.cn/75197.Doc
wkw.irampL.cn/80220.Doc
wkw.irampL.cn/88280.Doc
wkq.irampL.cn/28204.Doc
wkq.irampL.cn/31959.Doc
wkq.irampL.cn/84466.Doc
wkq.irampL.cn/88646.Doc
wkq.irampL.cn/64228.Doc
wkq.irampL.cn/60222.Doc
wkq.irampL.cn/84246.Doc
wkq.irampL.cn/86280.Doc
wkq.irampL.cn/51597.Doc
wkq.irampL.cn/64224.Doc
wjm.irampL.cn/24240.Doc
wjm.irampL.cn/28260.Doc
wjm.irampL.cn/93395.Doc
wjm.irampL.cn/46068.Doc
wjm.irampL.cn/95719.Doc
wjm.irampL.cn/95517.Doc
wjm.irampL.cn/24622.Doc
wjm.irampL.cn/08806.Doc
wjm.irampL.cn/80206.Doc
wjm.irampL.cn/20486.Doc
wjn.irampL.cn/44846.Doc
wjn.irampL.cn/66084.Doc
wjn.irampL.cn/02288.Doc
wjn.irampL.cn/40880.Doc
wjn.irampL.cn/42428.Doc
wjn.irampL.cn/26464.Doc
wjn.irampL.cn/48866.Doc
wjn.irampL.cn/40868.Doc
wjn.irampL.cn/80428.Doc
wjn.irampL.cn/20648.Doc
wjb.irampL.cn/28264.Doc
wjb.irampL.cn/06820.Doc
wjb.irampL.cn/84844.Doc
wjb.irampL.cn/26862.Doc
wjb.irampL.cn/46026.Doc
wjb.irampL.cn/75939.Doc
wjb.irampL.cn/82446.Doc
wjb.irampL.cn/15915.Doc
wjb.irampL.cn/46808.Doc
wjb.irampL.cn/79711.Doc
wjv.irampL.cn/60886.Doc
wjv.irampL.cn/06806.Doc
wjv.irampL.cn/04080.Doc
wjv.irampL.cn/22462.Doc
wjv.irampL.cn/42600.Doc
wjv.irampL.cn/68426.Doc
wjv.irampL.cn/24464.Doc
wjv.irampL.cn/84468.Doc
wjv.irampL.cn/66044.Doc
wjv.irampL.cn/26248.Doc
