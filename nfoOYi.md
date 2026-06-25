
============================================================
 Python LRU缓存实现 — OrderedDict/手动双链表/functools/线程安全
============================================================

LRU（Least Recently Used）缓存淘汰最近最少使用的数据，是缓存算法的经典实现。
Python提供了多种方式实现LRU缓存，从内置装饰器到手写数据结构。

============================================================
1. 基于 OrderedDict 的 LRU 缓存
============================================================

from collections import OrderedDict

class LRUCacheOrderedDict:
    """使用OrderedDict实现LRU缓存，简洁高效"""
    def __init__(self, capacity):
        self.capacity = capacity          # 缓存容量
        self.cache = OrderedDict()        # 有序字典保持访问顺序

    def get(self, key):
        """获取键对应的值，若不存在返回-1"""
        if key not in self.cache:
            return -1
        # 将访问的键移到末尾（表示最近使用）
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key, value):
        """插入或更新键值对"""
        if key in self.cache:
            # 已存在：更新值并移到末尾
            self.cache.move_to_end(key)
        elif len(self.cache) >= self.capacity:
            # 缓存已满：弹出最久未使用的（第一个元素）
            self.cache.popitem(last=False)
        self.cache[key] = value

    def __repr__(self):
        return f"LRUCache({dict(self.cache)})"

# 测试
lru = LRUCacheOrderedDict(3)
lru.put('A', 1); lru.put('B', 2); lru.put('C', 3)
print(lru)                               # {'A':1, 'B':2, 'C':3}
lru.get('A')                             # 访问A，A移到末尾
lru.put('D', 4)                          # 新增D，淘汰最久未用的B
print("淘汰B后:", lru)                   # {'C':3, 'A':1, 'D':4}

============================================================
2. 手动实现：双向链表 + 字典
============================================================

class DLinkedNode:
    """双向链表节点，用于手动LRU"""
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCacheManual:
    """手写双向链表+字典实现LRU，面试常考"""
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}                   # {key: DLinkedNode}
        # 虚拟头尾节点简化边界处理
        self.head = DLinkedNode()         # 最近使用的
        self.tail = DLinkedNode()         # 最久未使用的
        self.head.next = self.tail
        self.tail.prev = self.head

    def _add_node(self, node):
        """将节点添加到头部（最近使用位置）"""
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def _remove_node(self, node):
        """从链表中移除一个节点"""
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node

    def _move_to_head(self, node):
        """将节点移动到头部（表示刚被访问）"""
        self._remove_node(node)
        self._add_node(node)

    def _pop_tail(self):
        """弹出尾部节点（最久未使用的）"""
        node = self.tail.prev
        self._remove_node(node)
        return node

    def get(self, key):
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._move_to_head(node)          # 标记为最近使用
        return node.value

    def put(self, key, value):
        if key in self.cache:
            node = self.cache[key]
            node.value = value
            self._move_to_head(node)
        else:
            if len(self.cache) >= self.capacity:
                tail = self._pop_tail()   # 淘汰最久未使用的
                del self.cache[tail.key]
            new_node = DLinkedNode(key, value)
            self.cache[key] = new_node
            self._add_node(new_node)

============================================================
3. functools.lru_cache 装饰器
============================================================

from functools import lru_cache
import time

@lru_cache(maxsize=128)
def fibonacci(n):
    """带缓存的斐波那契数列计算"""
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# lru_cache的参数：
# - maxsize: 最大缓存条目数（设为None表示无上限）
# - typed: 是否区分不同类型（如3和3.0）

# 第一次调用会递归计算，后续调用直接返回缓存结果
start = time.time()
result = fibonacci(35)
print(f"fib(35) = {result}, 耗时: {time.time() - start:.4f}s")

# 查看缓存统计信息
print("缓存统计:", fibonacci.cache_info())
# CacheInfo(hits=..., misses=..., maxsize=128, currsize=...)

# 清空缓存
fibonacci.cache_clear()

# 使用lru_cache实现函数结果的自动过期并不是内置支持
# 但对于固定输入输出的纯函数，lru_cache是非常好用的工具

============================================================
4. LRU vs LFU 对比
============================================================
# LRU（最近最少使用）：
#   - 淘汰最久未被访问的条目
#   - 实现相对简单
#   - 对突发性大量访问敏感（可能淘汰真正常用的数据）
#
# LFU（最不经常使用）：
#   - 淘汰访问频率最低的条目
#   - 需要维护访问计数
#   - 对历史访问模式有"记忆"
#
# 选择建议：
#   - 数据访问模式相对稳定 → LRU
#   - 有明显的热点数据和冷门数据 → LFU

============================================================
5. 基于时间的过期缓存
============================================================

import time as time_module

class ExpiringCache:
    """带过期时间的缓存"""
    def __init__(self, default_ttl=60):
        self.cache = {}                   # {key: (value, expire_time)}
        self.default_ttl = default_ttl    # 默认过期时间（秒）

    def get(self, key):
        if key not in self.cache:
            return None
        value, expire = self.cache[key]
        if time_module.time() > expire:   # 已过期
            del self.cache[key]
            return None
        return value

    def set(self, key, value, ttl=None):
        expire = time_module.time() + (ttl or self.default_ttl)
        self.cache[key] = (value, expire)

    def cleanup(self):
        """清理所有过期条目"""
        now = time_module.time()
        expired = [k for k, (_, t) in self.cache.items() if now > t]
        for k in expired:
            del self.cache[k]

============================================================
6. 大小受限缓存（Size-Bounded Cache）
============================================================

class SizeBoundedCache:
    """限制内容总大小的缓存（按字节计数）"""
    def __init__(self, max_size_bytes=1024*1024):
        self.cache = OrderedDict()
        self.max_size = max_size_bytes
        self.current_size = 0

    def get(self, key):
        if key not in self.cache:
            return None
        self.cache.move_to_end(key)
        return self.cache[key][0]

    def set(self, key, value):
        value_size = len(str(value))      # 估算大小
        if key in self.cache:
            old_value, old_size = self.cache[key]
            self.current_size -= old_size
        # 如果单个值就超过缓存限制，拒绝缓存
        if value_size > self.max_size:
            return
        # 腾出空间
        while self.current_size + value_size > self.max_size:
            _, (_, removed_size) = self.cache.popitem(last=False)
            self.current_size -= removed_size
        self.cache[key] = (value, value_size)
        self.current_size += value_size

============================================================
7. 线程安全的LRU缓存
============================================================

from threading import Lock

class ThreadSafeLRUCache:
    """使用Lock保证线程安全的LRU缓存"""
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()
        self.lock = Lock()                # 保证线程安全

    def get(self, key):
        with self.lock:                   # 加锁保护临界区
            if key not in self.cache:
                return -1
            self.cache.move_to_end(key)
            return self.cache[key]

    def put(self, key, value):
        with self.lock:
            if key in self.cache:
                self.cache.move_to_end(key)
            elif len(self.cache) >= self.capacity:
                self.cache.popitem(last=False)
            self.cache[key] = value

doi.kumchen.cn/85768.Doc
doi.kumchen.cn/45246.Doc
doi.kumchen.cn/19777.Doc
doi.kumchen.cn/10767.Doc
doi.kumchen.cn/59091.Doc
doi.kumchen.cn/23793.Doc
doi.kumchen.cn/55818.Doc
doi.kumchen.cn/52404.Doc
doi.kumchen.cn/24528.Doc
doi.kumchen.cn/48792.Doc
dou.kumchen.cn/52614.Doc
dou.kumchen.cn/68031.Doc
dou.kumchen.cn/06992.Doc
dou.kumchen.cn/54326.Doc
dou.kumchen.cn/95325.Doc
dou.kumchen.cn/73479.Doc
dou.kumchen.cn/94532.Doc
dou.kumchen.cn/31676.Doc
dou.kumchen.cn/35675.Doc
dou.kumchen.cn/23470.Doc
doy.kumchen.cn/18342.Doc
doy.kumchen.cn/61256.Doc
doy.kumchen.cn/30966.Doc
doy.kumchen.cn/87644.Doc
doy.kumchen.cn/43233.Doc
doy.kumchen.cn/52283.Doc
doy.kumchen.cn/56004.Doc
doy.kumchen.cn/14382.Doc
doy.kumchen.cn/57082.Doc
doy.kumchen.cn/00594.Doc
dot.kumchen.cn/53428.Doc
dot.kumchen.cn/18953.Doc
dot.kumchen.cn/80493.Doc
dot.kumchen.cn/61418.Doc
dot.kumchen.cn/88004.Doc
dot.kumchen.cn/27172.Doc
dot.kumchen.cn/59929.Doc
dot.kumchen.cn/46829.Doc
dot.kumchen.cn/70904.Doc
dot.kumchen.cn/30815.Doc
dor.kumchen.cn/08146.Doc
dor.kumchen.cn/68063.Doc
dor.kumchen.cn/93203.Doc
dor.kumchen.cn/30135.Doc
dor.kumchen.cn/68102.Doc
dor.kumchen.cn/07498.Doc
dor.kumchen.cn/26965.Doc
dor.kumchen.cn/44024.Doc
dor.kumchen.cn/18979.Doc
dor.kumchen.cn/39345.Doc
doe.kumchen.cn/73366.Doc
doe.kumchen.cn/29259.Doc
doe.kumchen.cn/64592.Doc
doe.kumchen.cn/33240.Doc
doe.kumchen.cn/01570.Doc
doe.kumchen.cn/09457.Doc
doe.kumchen.cn/88938.Doc
doe.kumchen.cn/34922.Doc
doe.kumchen.cn/23378.Doc
doe.kumchen.cn/14400.Doc
dow.kumchen.cn/55770.Doc
dow.kumchen.cn/39868.Doc
dow.kumchen.cn/43678.Doc
dow.kumchen.cn/37635.Doc
dow.kumchen.cn/31556.Doc
dow.kumchen.cn/97105.Doc
dow.kumchen.cn/64435.Doc
dow.kumchen.cn/66205.Doc
dow.kumchen.cn/23027.Doc
dow.kumchen.cn/80487.Doc
doq.kumchen.cn/80357.Doc
doq.kumchen.cn/31267.Doc
doq.kumchen.cn/99240.Doc
doq.kumchen.cn/86031.Doc
doq.kumchen.cn/22332.Doc
doq.kumchen.cn/27124.Doc
doq.kumchen.cn/89896.Doc
doq.kumchen.cn/76513.Doc
doq.kumchen.cn/13135.Doc
doq.kumchen.cn/42044.Doc
dim.kumchen.cn/48715.Doc
dim.kumchen.cn/48722.Doc
dim.kumchen.cn/51127.Doc
dim.kumchen.cn/24503.Doc
dim.kumchen.cn/51020.Doc
dim.kumchen.cn/78797.Doc
dim.kumchen.cn/61614.Doc
dim.kumchen.cn/06291.Doc
dim.kumchen.cn/59363.Doc
dim.kumchen.cn/52663.Doc
din.kumchen.cn/19799.Doc
din.kumchen.cn/23708.Doc
din.kumchen.cn/21808.Doc
din.kumchen.cn/51935.Doc
din.kumchen.cn/86488.Doc
din.kumchen.cn/96119.Doc
din.kumchen.cn/34578.Doc
din.kumchen.cn/66057.Doc
din.kumchen.cn/28175.Doc
din.kumchen.cn/57136.Doc
dib.kumchen.cn/66365.Doc
dib.kumchen.cn/86022.Doc
dib.kumchen.cn/29112.Doc
dib.kumchen.cn/73545.Doc
dib.kumchen.cn/80841.Doc
dib.kumchen.cn/79213.Doc
dib.kumchen.cn/08682.Doc
dib.kumchen.cn/80222.Doc
dib.kumchen.cn/86787.Doc
dib.kumchen.cn/90677.Doc
div.kumchen.cn/45567.Doc
div.kumchen.cn/88810.Doc
div.kumchen.cn/57013.Doc
div.kumchen.cn/60735.Doc
div.kumchen.cn/10891.Doc
div.kumchen.cn/34346.Doc
div.kumchen.cn/93260.Doc
div.kumchen.cn/00666.Doc
div.kumchen.cn/65250.Doc
div.kumchen.cn/21630.Doc
dic.kumchen.cn/23366.Doc
dic.kumchen.cn/94419.Doc
dic.kumchen.cn/48581.Doc
dic.kumchen.cn/51278.Doc
dic.kumchen.cn/24257.Doc
dic.kumchen.cn/71017.Doc
dic.kumchen.cn/46568.Doc
dic.kumchen.cn/54888.Doc
dic.kumchen.cn/17025.Doc
dic.kumchen.cn/92859.Doc
dix.kumchen.cn/08778.Doc
dix.kumchen.cn/81378.Doc
dix.kumchen.cn/27218.Doc
dix.kumchen.cn/42100.Doc
dix.kumchen.cn/48701.Doc
dix.kumchen.cn/96494.Doc
dix.kumchen.cn/49955.Doc
dix.kumchen.cn/99577.Doc
dix.kumchen.cn/42090.Doc
dix.kumchen.cn/72464.Doc
diz.kumchen.cn/64965.Doc
diz.kumchen.cn/17245.Doc
diz.kumchen.cn/25627.Doc
diz.kumchen.cn/54639.Doc
diz.kumchen.cn/33674.Doc
diz.kumchen.cn/86280.Doc
diz.kumchen.cn/06624.Doc
diz.kumchen.cn/65160.Doc
diz.kumchen.cn/16659.Doc
diz.kumchen.cn/59247.Doc
dil.kumchen.cn/33686.Doc
dil.kumchen.cn/10175.Doc
dil.kumchen.cn/88000.Doc
dil.kumchen.cn/67789.Doc
dil.kumchen.cn/70158.Doc
dil.kumchen.cn/40511.Doc
dil.kumchen.cn/86991.Doc
dil.kumchen.cn/76910.Doc
dil.kumchen.cn/71246.Doc
dil.kumchen.cn/00714.Doc
dik.kumchen.cn/32380.Doc
dik.kumchen.cn/85113.Doc
dik.kumchen.cn/52681.Doc
dik.kumchen.cn/94471.Doc
dik.kumchen.cn/71401.Doc
dik.kumchen.cn/87510.Doc
dik.kumchen.cn/06601.Doc
dik.kumchen.cn/33592.Doc
dik.kumchen.cn/32528.Doc
dik.kumchen.cn/10995.Doc
dij.kumchen.cn/02213.Doc
dij.kumchen.cn/86998.Doc
dij.kumchen.cn/79907.Doc
dij.kumchen.cn/93851.Doc
dij.kumchen.cn/39413.Doc
dij.kumchen.cn/06705.Doc
dij.kumchen.cn/82561.Doc
dij.kumchen.cn/88595.Doc
dij.kumchen.cn/28900.Doc
dij.kumchen.cn/82094.Doc
dih.kumchen.cn/08068.Doc
dih.kumchen.cn/07092.Doc
dih.kumchen.cn/69598.Doc
dih.kumchen.cn/43611.Doc
dih.kumchen.cn/23970.Doc
dih.kumchen.cn/64834.Doc
dih.kumchen.cn/91498.Doc
dih.kumchen.cn/00611.Doc
dih.kumchen.cn/27135.Doc
dih.kumchen.cn/42601.Doc
dig.kumchen.cn/29936.Doc
dig.kumchen.cn/75845.Doc
dig.kumchen.cn/90926.Doc
dig.kumchen.cn/99136.Doc
dig.kumchen.cn/87137.Doc
dig.kumchen.cn/11484.Doc
dig.kumchen.cn/37811.Doc
dig.kumchen.cn/26345.Doc
dig.kumchen.cn/73999.Doc
dig.kumchen.cn/34843.Doc
dif.kumchen.cn/69453.Doc
dif.kumchen.cn/50018.Doc
dif.kumchen.cn/00879.Doc
dif.kumchen.cn/08454.Doc
dif.kumchen.cn/85909.Doc
dif.kumchen.cn/69047.Doc
dif.kumchen.cn/75175.Doc
dif.kumchen.cn/38008.Doc
dif.kumchen.cn/24636.Doc
dif.kumchen.cn/03621.Doc
did.kumchen.cn/37528.Doc
did.kumchen.cn/29201.Doc
did.kumchen.cn/64892.Doc
did.kumchen.cn/94033.Doc
did.kumchen.cn/72887.Doc
did.kumchen.cn/37787.Doc
did.kumchen.cn/27906.Doc
did.kumchen.cn/94015.Doc
did.kumchen.cn/98910.Doc
did.kumchen.cn/64814.Doc
dis.kumchen.cn/63402.Doc
dis.kumchen.cn/66787.Doc
dis.kumchen.cn/90859.Doc
dis.kumchen.cn/66229.Doc
dis.kumchen.cn/16487.Doc
dis.kumchen.cn/98569.Doc
dis.kumchen.cn/81815.Doc
dis.kumchen.cn/20241.Doc
dis.kumchen.cn/06422.Doc
dis.kumchen.cn/92998.Doc
dia.kumchen.cn/29284.Doc
dia.kumchen.cn/35770.Doc
dia.kumchen.cn/36228.Doc
dia.kumchen.cn/26855.Doc
dia.kumchen.cn/32344.Doc
dia.kumchen.cn/29403.Doc
dia.kumchen.cn/24634.Doc
dia.kumchen.cn/04934.Doc
dia.kumchen.cn/50357.Doc
dia.kumchen.cn/55138.Doc
dip.kumchen.cn/96176.Doc
dip.kumchen.cn/43604.Doc
dip.kumchen.cn/66081.Doc
dip.kumchen.cn/89062.Doc
dip.kumchen.cn/63635.Doc
dip.kumchen.cn/21577.Doc
dip.kumchen.cn/57770.Doc
dip.kumchen.cn/62523.Doc
dip.kumchen.cn/09452.Doc
dip.kumchen.cn/00609.Doc
dio.kumchen.cn/26410.Doc
dio.kumchen.cn/17200.Doc
dio.kumchen.cn/88634.Doc
dio.kumchen.cn/00279.Doc
dio.kumchen.cn/72387.Doc
dio.kumchen.cn/85973.Doc
dio.kumchen.cn/06572.Doc
dio.kumchen.cn/51590.Doc
dio.kumchen.cn/33269.Doc
dio.kumchen.cn/31502.Doc
dii.kumchen.cn/23547.Doc
dii.kumchen.cn/08024.Doc
dii.kumchen.cn/79034.Doc
dii.kumchen.cn/61258.Doc
dii.kumchen.cn/60146.Doc
dii.kumchen.cn/78760.Doc
dii.kumchen.cn/84604.Doc
dii.kumchen.cn/10244.Doc
dii.kumchen.cn/69105.Doc
dii.kumchen.cn/82044.Doc
diu.kumchen.cn/00839.Doc
diu.kumchen.cn/27242.Doc
diu.kumchen.cn/50196.Doc
diu.kumchen.cn/63404.Doc
diu.kumchen.cn/25998.Doc
diu.kumchen.cn/06298.Doc
diu.kumchen.cn/35730.Doc
diu.kumchen.cn/04382.Doc
diu.kumchen.cn/99452.Doc
diu.kumchen.cn/24724.Doc
diy.kumchen.cn/83487.Doc
diy.kumchen.cn/03708.Doc
diy.kumchen.cn/63162.Doc
diy.kumchen.cn/73502.Doc
diy.kumchen.cn/77301.Doc
diy.kumchen.cn/99665.Doc
diy.kumchen.cn/23360.Doc
diy.kumchen.cn/69420.Doc
diy.kumchen.cn/50706.Doc
diy.kumchen.cn/93980.Doc
dit.kumchen.cn/42749.Doc
dit.kumchen.cn/36159.Doc
dit.kumchen.cn/99246.Doc
dit.kumchen.cn/70389.Doc
dit.kumchen.cn/16032.Doc
dit.kumchen.cn/36914.Doc
dit.kumchen.cn/52576.Doc
dit.kumchen.cn/89091.Doc
dit.kumchen.cn/02506.Doc
dit.kumchen.cn/90147.Doc
dir.kumchen.cn/02473.Doc
dir.kumchen.cn/00560.Doc
dir.kumchen.cn/59117.Doc
dir.kumchen.cn/40698.Doc
dir.kumchen.cn/80864.Doc
dir.kumchen.cn/27027.Doc
dir.kumchen.cn/70279.Doc
dir.kumchen.cn/74265.Doc
dir.kumchen.cn/57568.Doc
dir.kumchen.cn/92218.Doc
die.kumchen.cn/76646.Doc
die.kumchen.cn/47691.Doc
die.kumchen.cn/21942.Doc
die.kumchen.cn/04842.Doc
die.kumchen.cn/97898.Doc
die.kumchen.cn/68043.Doc
die.kumchen.cn/94553.Doc
die.kumchen.cn/09814.Doc
die.kumchen.cn/06280.Doc
die.kumchen.cn/22579.Doc
diw.kumchen.cn/42404.Doc
diw.kumchen.cn/18366.Doc
diw.kumchen.cn/63815.Doc
diw.kumchen.cn/81386.Doc
diw.kumchen.cn/93009.Doc
diw.kumchen.cn/71842.Doc
diw.kumchen.cn/52693.Doc
diw.kumchen.cn/99601.Doc
diw.kumchen.cn/90847.Doc
diw.kumchen.cn/99236.Doc
diq.kumchen.cn/74252.Doc
diq.kumchen.cn/33192.Doc
diq.kumchen.cn/05266.Doc
diq.kumchen.cn/96575.Doc
diq.kumchen.cn/46857.Doc
diq.kumchen.cn/05801.Doc
diq.kumchen.cn/35579.Doc
diq.kumchen.cn/58156.Doc
diq.kumchen.cn/22376.Doc
diq.kumchen.cn/29156.Doc
dum.kumchen.cn/89027.Doc
dum.kumchen.cn/82001.Doc
dum.kumchen.cn/88472.Doc
dum.kumchen.cn/12634.Doc
dum.kumchen.cn/08572.Doc
dum.kumchen.cn/50153.Doc
dum.kumchen.cn/57548.Doc
dum.kumchen.cn/84185.Doc
dum.kumchen.cn/18319.Doc
dum.kumchen.cn/69931.Doc
dun.kumchen.cn/62983.Doc
dun.kumchen.cn/13856.Doc
dun.kumchen.cn/00137.Doc
dun.kumchen.cn/50592.Doc
dun.kumchen.cn/59722.Doc
dun.kumchen.cn/84606.Doc
dun.kumchen.cn/96349.Doc
dun.kumchen.cn/62509.Doc
dun.kumchen.cn/72298.Doc
dun.kumchen.cn/24529.Doc
dub.kumchen.cn/33348.Doc
dub.kumchen.cn/37098.Doc
dub.kumchen.cn/93426.Doc
dub.kumchen.cn/50512.Doc
dub.kumchen.cn/65825.Doc
dub.kumchen.cn/83536.Doc
dub.kumchen.cn/64648.Doc
dub.kumchen.cn/04983.Doc
dub.kumchen.cn/59127.Doc
dub.kumchen.cn/30677.Doc
duv.kumchen.cn/28209.Doc
duv.kumchen.cn/42581.Doc
duv.kumchen.cn/00697.Doc
duv.kumchen.cn/12979.Doc
duv.kumchen.cn/99118.Doc
duv.kumchen.cn/65609.Doc
duv.kumchen.cn/04112.Doc
duv.kumchen.cn/78117.Doc
duv.kumchen.cn/48471.Doc
duv.kumchen.cn/55877.Doc
duc.kumchen.cn/08586.Doc
duc.kumchen.cn/11269.Doc
duc.kumchen.cn/14288.Doc
duc.kumchen.cn/48890.Doc
duc.kumchen.cn/66967.Doc
duc.kumchen.cn/07170.Doc
duc.kumchen.cn/68142.Doc
duc.kumchen.cn/02132.Doc
duc.kumchen.cn/79022.Doc
duc.kumchen.cn/67415.Doc
dux.kumchen.cn/11251.Doc
dux.kumchen.cn/34672.Doc
dux.kumchen.cn/26908.Doc
dux.kumchen.cn/09253.Doc
dux.kumchen.cn/53958.Doc
dux.kumchen.cn/03595.Doc
dux.kumchen.cn/84511.Doc
dux.kumchen.cn/83063.Doc
dux.kumchen.cn/85782.Doc
dux.kumchen.cn/62903.Doc
duz.kumchen.cn/23600.Doc
duz.kumchen.cn/53384.Doc
duz.kumchen.cn/83109.Doc
duz.kumchen.cn/73846.Doc
duz.kumchen.cn/94898.Doc
duz.kumchen.cn/67498.Doc
duz.kumchen.cn/98273.Doc
duz.kumchen.cn/27966.Doc
duz.kumchen.cn/99890.Doc
duz.kumchen.cn/80245.Doc
dul.kumchen.cn/78764.Doc
dul.kumchen.cn/63197.Doc
dul.kumchen.cn/87594.Doc
dul.kumchen.cn/71416.Doc
dul.kumchen.cn/76177.Doc
dul.kumchen.cn/49642.Doc
dul.kumchen.cn/79456.Doc
dul.kumchen.cn/01742.Doc
dul.kumchen.cn/66132.Doc
dul.kumchen.cn/65486.Doc
duk.kumchen.cn/19573.Doc
duk.kumchen.cn/92898.Doc
duk.kumchen.cn/99978.Doc
duk.kumchen.cn/59333.Doc
duk.kumchen.cn/04362.Doc
duk.kumchen.cn/64387.Doc
duk.kumchen.cn/21380.Doc
duk.kumchen.cn/70499.Doc
duk.kumchen.cn/09879.Doc
duk.kumchen.cn/00225.Doc
duj.kumchen.cn/63663.Doc
duj.kumchen.cn/65813.Doc
duj.kumchen.cn/18957.Doc
duj.kumchen.cn/36602.Doc
duj.kumchen.cn/67486.Doc
duj.kumchen.cn/47400.Doc
duj.kumchen.cn/74024.Doc
duj.kumchen.cn/78389.Doc
duj.kumchen.cn/00634.Doc
duj.kumchen.cn/15500.Doc
duh.kumchen.cn/46998.Doc
duh.kumchen.cn/11035.Doc
duh.kumchen.cn/63249.Doc
duh.kumchen.cn/38599.Doc
duh.kumchen.cn/52225.Doc
duh.kumchen.cn/63799.Doc
duh.kumchen.cn/79938.Doc
duh.kumchen.cn/16962.Doc
duh.kumchen.cn/44342.Doc
duh.kumchen.cn/46606.Doc
dug.kumchen.cn/46767.Doc
dug.kumchen.cn/68310.Doc
dug.kumchen.cn/32846.Doc
dug.kumchen.cn/91576.Doc
dug.kumchen.cn/80906.Doc
dug.kumchen.cn/18059.Doc
dug.kumchen.cn/38793.Doc
dug.kumchen.cn/08142.Doc
dug.kumchen.cn/75324.Doc
dug.kumchen.cn/06499.Doc
duf.kumchen.cn/06226.Doc
duf.kumchen.cn/24083.Doc
duf.kumchen.cn/62294.Doc
duf.kumchen.cn/73415.Doc
duf.kumchen.cn/24403.Doc
duf.kumchen.cn/24416.Doc
duf.kumchen.cn/32757.Doc
duf.kumchen.cn/58803.Doc
duf.kumchen.cn/34578.Doc
duf.kumchen.cn/39595.Doc
dud.kumchen.cn/75059.Doc
dud.kumchen.cn/67076.Doc
dud.kumchen.cn/36722.Doc
dud.kumchen.cn/26099.Doc
dud.kumchen.cn/33335.Doc
dud.kumchen.cn/24328.Doc
dud.kumchen.cn/80480.Doc
dud.kumchen.cn/50778.Doc
dud.kumchen.cn/97429.Doc
dud.kumchen.cn/25982.Doc
dus.kumchen.cn/70554.Doc
dus.kumchen.cn/51851.Doc
dus.kumchen.cn/66479.Doc
dus.kumchen.cn/48728.Doc
dus.kumchen.cn/60225.Doc
dus.kumchen.cn/97348.Doc
dus.kumchen.cn/11509.Doc
dus.kumchen.cn/71797.Doc
dus.kumchen.cn/72721.Doc
dus.kumchen.cn/72212.Doc
