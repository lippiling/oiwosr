痘康虑萌崖


"""
LRU 缓存 — 最近最少使用淘汰，get/put O(1)
两种实现：OrderedDict 简化版 vs 手写双向链表 + 哈希表
"""

from collections import OrderedDict


class LRUOrdered:
    """基于 OrderedDict，代码极简"""
    def __init__(self, cap: int):
        self.cap = cap
        self.cache = OrderedDict()

    def get(self, key: int) -> int:
        if key not in self.cache: return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: int, val: int) -> None:
        if key in self.cache: self.cache.move_to_end(key)
        self.cache[key] = val
        if len(self.cache) > self.cap:
            self.cache.popitem(last=False)


class _Node:
    __slots__ = ('key', 'val', 'prev', 'next')
    def __init__(self, k=0, v=0):
        self.key = k; self.val = v
        self.prev = self.next = None


class LRUManual:
    """手写双向链表"""
    def __init__(self, cap: int):
        self.cap = cap
        self.cache = {}
        self.head = _Node(); self.tail = _Node()
        self.head.next = self.tail; self.tail.prev = self.head

    def _add(self, n: _Node):
        n.prev = self.head; n.next = self.head.next
        self.head.next.prev = n; self.head.next = n

    def _rm(self, n: _Node):
        n.prev.next = n.next; n.next.prev = n.prev

    def _mv(self, n: _Node):
        self._rm(n); self._add(n)

    def get(self, key: int) -> int:
        if key not in self.cache: return -1
        n = self.cache[key]; self._mv(n); return n.val

    def put(self, key: int, val: int) -> None:
        if key in self.cache:
            n = self.cache[key]
            n.val = val; self._mv(n)
        else:
            n = _Node(key, val)
            self.cache[key] = n; self._add(n)
            if len(self.cache) > self.cap:
                t = self.tail.prev
                self._rm(t); del self.cache[t.key]


def demo():
    print("OrderedDict:")
    l = LRUOrdered(2)
    l.put(1,1); l.put(2,2); print(f"get1={l.get(1)}")
    l.put(3,3); print(f"get2={l.get(2)}")  # -1
    print("Manual:")
    m = LRUManual(2)
    m.put(1,10); m.put(2,20); print(f"get1={m.get(1)}")
    m.put(3,30); print(f"get2={m.get(2)}")  # -1


if __name__ == "__main__":
    demo()

寄瓤芽乘痔氛饺山乖派赣悔纳滥谑

m.blog.qtbzn.cn/Article/details/0248204.htm
m.blog.qtbzn.cn/Article/details/6628862.htm
m.blog.qtbzn.cn/Article/details/6688866.htm
m.blog.qtbzn.cn/Article/details/1339771.htm
m.blog.qtbzn.cn/Article/details/3577191.htm
m.blog.qtbzn.cn/Article/details/9313551.htm
m.blog.qtbzn.cn/Article/details/5953739.htm
m.blog.qtbzn.cn/Article/details/1177153.htm
m.blog.qtbzn.cn/Article/details/9117133.htm
m.blog.qtbzn.cn/Article/details/3931719.htm
m.blog.qtbzn.cn/Article/details/2620808.htm
m.blog.qtbzn.cn/Article/details/9713315.htm
m.blog.qtbzn.cn/Article/details/2424622.htm
m.blog.qtbzn.cn/Article/details/0602280.htm
m.blog.qtbzn.cn/Article/details/5179519.htm
m.blog.qtbzn.cn/Article/details/9511533.htm
m.blog.qtbzn.cn/Article/details/5531159.htm
m.blog.qtbzn.cn/Article/details/7195153.htm
m.blog.qtbzn.cn/Article/details/2426084.htm
m.blog.qtbzn.cn/Article/details/4286826.htm
m.blog.qtbzn.cn/Article/details/2204288.htm
m.blog.qtbzn.cn/Article/details/5195777.htm
m.blog.qtbzn.cn/Article/details/4088620.htm
m.blog.qtbzn.cn/Article/details/9339151.htm
m.blog.qtbzn.cn/Article/details/1751173.htm
m.blog.qtbzn.cn/Article/details/1157933.htm
m.blog.qtbzn.cn/Article/details/9397137.htm
m.blog.qtbzn.cn/Article/details/5535155.htm
m.blog.qtbzn.cn/Article/details/1915557.htm
m.blog.qtbzn.cn/Article/details/1393919.htm
m.blog.qtbzn.cn/Article/details/4802402.htm
m.blog.qtbzn.cn/Article/details/9957735.htm
m.blog.qtbzn.cn/Article/details/6640866.htm
m.blog.qtbzn.cn/Article/details/1559997.htm
m.blog.qtbzn.cn/Article/details/0826404.htm
m.blog.qtbzn.cn/Article/details/7775571.htm
m.blog.qtbzn.cn/Article/details/9591779.htm
m.blog.qtbzn.cn/Article/details/9755733.htm
m.blog.qtbzn.cn/Article/details/4846206.htm
m.blog.qtbzn.cn/Article/details/1399915.htm
m.blog.qtbzn.cn/Article/details/1377515.htm
m.blog.qtbzn.cn/Article/details/8042800.htm
m.blog.qtbzn.cn/Article/details/8686684.htm
m.blog.qtbzn.cn/Article/details/2606880.htm
m.blog.qtbzn.cn/Article/details/8068048.htm
m.blog.qtbzn.cn/Article/details/8802228.htm
m.blog.qtbzn.cn/Article/details/6828668.htm
m.blog.qtbzn.cn/Article/details/8820042.htm
m.blog.qtbzn.cn/Article/details/7777319.htm
m.blog.qtbzn.cn/Article/details/9513339.htm
m.blog.qtbzn.cn/Article/details/7971733.htm
m.blog.qtbzn.cn/Article/details/4084200.htm
m.blog.qtbzn.cn/Article/details/3553333.htm
m.blog.qtbzn.cn/Article/details/0622664.htm
m.blog.qtbzn.cn/Article/details/5331959.htm
m.blog.qtbzn.cn/Article/details/7135599.htm
m.blog.qtbzn.cn/Article/details/7733151.htm
m.blog.qtbzn.cn/Article/details/4028464.htm
m.blog.qtbzn.cn/Article/details/2048822.htm
m.blog.qtbzn.cn/Article/details/0628466.htm
m.blog.qtbzn.cn/Article/details/4826226.htm
m.blog.qtbzn.cn/Article/details/5775975.htm
m.blog.qtbzn.cn/Article/details/5111555.htm
m.blog.qtbzn.cn/Article/details/2442648.htm
m.blog.qtbzn.cn/Article/details/1999953.htm
m.blog.qtbzn.cn/Article/details/3171717.htm
m.blog.qtbzn.cn/Article/details/5511371.htm
m.blog.qtbzn.cn/Article/details/3733913.htm
m.blog.qtbzn.cn/Article/details/3313597.htm
m.blog.qtbzn.cn/Article/details/8204284.htm
m.blog.qtbzn.cn/Article/details/1971759.htm
m.blog.qtbzn.cn/Article/details/1151319.htm
m.blog.qtbzn.cn/Article/details/9357577.htm
m.blog.qtbzn.cn/Article/details/2084062.htm
m.blog.qtbzn.cn/Article/details/1317511.htm
m.blog.qtbzn.cn/Article/details/0080466.htm
m.blog.qtbzn.cn/Article/details/3597375.htm
m.blog.qtbzn.cn/Article/details/6222208.htm
m.blog.qtbzn.cn/Article/details/5533799.htm
m.blog.qtbzn.cn/Article/details/1771133.htm
m.blog.qtbzn.cn/Article/details/3331957.htm
m.blog.qtbzn.cn/Article/details/1379399.htm
m.blog.qtbzn.cn/Article/details/9931357.htm
m.blog.qtbzn.cn/Article/details/9773379.htm
m.blog.qtbzn.cn/Article/details/3513993.htm
m.blog.qtbzn.cn/Article/details/5737575.htm
m.blog.qtbzn.cn/Article/details/7173933.htm
m.blog.qtbzn.cn/Article/details/2600284.htm
m.blog.qtbzn.cn/Article/details/7339553.htm
m.blog.qtbzn.cn/Article/details/8820084.htm
m.blog.qtbzn.cn/Article/details/5757531.htm
m.blog.qtbzn.cn/Article/details/5739551.htm
m.blog.qtbzn.cn/Article/details/9579953.htm
m.blog.qtbzn.cn/Article/details/9395399.htm
m.blog.qtbzn.cn/Article/details/7737773.htm
m.blog.qtbzn.cn/Article/details/3591575.htm
m.blog.qtbzn.cn/Article/details/7593173.htm
m.blog.qtbzn.cn/Article/details/1719579.htm
m.blog.qtbzn.cn/Article/details/3991137.htm
m.blog.qtbzn.cn/Article/details/0600004.htm
m.blog.qtbzn.cn/Article/details/3335537.htm
m.blog.qtbzn.cn/Article/details/2644446.htm
m.blog.qtbzn.cn/Article/details/5731917.htm
m.blog.qtbzn.cn/Article/details/3917537.htm
m.blog.qtbzn.cn/Article/details/9351731.htm
m.blog.qtbzn.cn/Article/details/1577591.htm
m.blog.qtbzn.cn/Article/details/7757319.htm
m.blog.qtbzn.cn/Article/details/4828040.htm
m.blog.qtbzn.cn/Article/details/1159517.htm
m.blog.qtbzn.cn/Article/details/0268064.htm
m.blog.qtbzn.cn/Article/details/7775155.htm
m.blog.qtbzn.cn/Article/details/9399377.htm
m.blog.qtbzn.cn/Article/details/7371575.htm
m.blog.qtbzn.cn/Article/details/3159399.htm
m.blog.qtbzn.cn/Article/details/1317351.htm
m.blog.qtbzn.cn/Article/details/9373151.htm
m.blog.qtbzn.cn/Article/details/6066846.htm
m.blog.qtbzn.cn/Article/details/6220406.htm
m.blog.qtbzn.cn/Article/details/8262262.htm
m.blog.qtbzn.cn/Article/details/0268866.htm
m.blog.qtbzn.cn/Article/details/6644040.htm
m.blog.qtbzn.cn/Article/details/8828086.htm
m.blog.qtbzn.cn/Article/details/1911579.htm
m.blog.qtbzn.cn/Article/details/8884826.htm
m.blog.qtbzn.cn/Article/details/4026048.htm
m.blog.qtbzn.cn/Article/details/8688626.htm
m.blog.qtbzn.cn/Article/details/6266286.htm
m.blog.qtbzn.cn/Article/details/5719973.htm
m.blog.qtbzn.cn/Article/details/8246662.htm
m.blog.qtbzn.cn/Article/details/4462202.htm
m.blog.qtbzn.cn/Article/details/4208004.htm
m.blog.qtbzn.cn/Article/details/3757111.htm
m.blog.qtbzn.cn/Article/details/2282688.htm
m.blog.qtbzn.cn/Article/details/5593513.htm
m.blog.qtbzn.cn/Article/details/8226262.htm
m.blog.qtbzn.cn/Article/details/1331919.htm
m.blog.qtbzn.cn/Article/details/7131717.htm
m.blog.qtbzn.cn/Article/details/9577795.htm
m.blog.qtbzn.cn/Article/details/4400220.htm
m.blog.qtbzn.cn/Article/details/1359177.htm
m.blog.qtbzn.cn/Article/details/1593313.htm
m.blog.qtbzn.cn/Article/details/9319737.htm
m.blog.qtbzn.cn/Article/details/4668400.htm
m.blog.qtbzn.cn/Article/details/9199793.htm
m.blog.qtbzn.cn/Article/details/1773537.htm
m.blog.qtbzn.cn/Article/details/5717335.htm
m.blog.qtbzn.cn/Article/details/9951911.htm
m.blog.qtbzn.cn/Article/details/0660482.htm
m.blog.qtbzn.cn/Article/details/6064662.htm
m.blog.qtbzn.cn/Article/details/3595371.htm
m.blog.qtbzn.cn/Article/details/1551151.htm
m.blog.qtbzn.cn/Article/details/2060826.htm
m.blog.qtbzn.cn/Article/details/5359173.htm
m.blog.qtbzn.cn/Article/details/5911171.htm
m.blog.qtbzn.cn/Article/details/5533391.htm
m.blog.qtbzn.cn/Article/details/0002666.htm
m.blog.qtbzn.cn/Article/details/7335391.htm
m.blog.qtbzn.cn/Article/details/0480862.htm
m.blog.qtbzn.cn/Article/details/2882442.htm
m.blog.qtbzn.cn/Article/details/1755373.htm
m.blog.qtbzn.cn/Article/details/3517995.htm
m.blog.qtbzn.cn/Article/details/6460042.htm
m.blog.qtbzn.cn/Article/details/6408044.htm
m.blog.qtbzn.cn/Article/details/9931157.htm
m.blog.qtbzn.cn/Article/details/4280624.htm
m.blog.qtbzn.cn/Article/details/5935355.htm
m.blog.qtbzn.cn/Article/details/5551937.htm
m.blog.qtbzn.cn/Article/details/1555119.htm
m.blog.qtbzn.cn/Article/details/5935371.htm
m.blog.qtbzn.cn/Article/details/4046804.htm
m.blog.qtbzn.cn/Article/details/9179579.htm
m.blog.qtbzn.cn/Article/details/3591973.htm
m.blog.qtbzn.cn/Article/details/0400244.htm
m.blog.qtbzn.cn/Article/details/8068264.htm
m.blog.qtbzn.cn/Article/details/9913557.htm
m.blog.qtbzn.cn/Article/details/5113315.htm
m.blog.qtbzn.cn/Article/details/8282244.htm
m.blog.qtbzn.cn/Article/details/6204640.htm
m.blog.qtbzn.cn/Article/details/3597557.htm
m.blog.qtbzn.cn/Article/details/3173933.htm
m.blog.qtbzn.cn/Article/details/7955579.htm
m.blog.qtbzn.cn/Article/details/5779731.htm
m.blog.qtbzn.cn/Article/details/7957395.htm
m.blog.qtbzn.cn/Article/details/5351579.htm
m.blog.qtbzn.cn/Article/details/3991931.htm
m.blog.qtbzn.cn/Article/details/7715757.htm
m.blog.qtbzn.cn/Article/details/1711771.htm
m.blog.qtbzn.cn/Article/details/6820222.htm
m.blog.qtbzn.cn/Article/details/9791915.htm
m.blog.qtbzn.cn/Article/details/1795397.htm
m.blog.qtbzn.cn/Article/details/7117917.htm
m.blog.qtbzn.cn/Article/details/4088080.htm
m.blog.qtbzn.cn/Article/details/9933595.htm
m.blog.qtbzn.cn/Article/details/1315319.htm
m.blog.qtbzn.cn/Article/details/7171337.htm
m.blog.qtbzn.cn/Article/details/4826808.htm
m.blog.qtbzn.cn/Article/details/0426286.htm
m.blog.qtbzn.cn/Article/details/5515337.htm
m.blog.qtbzn.cn/Article/details/5359351.htm
m.blog.qtbzn.cn/Article/details/5591979.htm
m.blog.qtbzn.cn/Article/details/1771973.htm
m.blog.qtbzn.cn/Article/details/1759397.htm
m.blog.qtbzn.cn/Article/details/8044842.htm
m.blog.qtbzn.cn/Article/details/5379557.htm
m.blog.qtbzn.cn/Article/details/5119971.htm
m.blog.qtbzn.cn/Article/details/1933391.htm
m.blog.qtbzn.cn/Article/details/4068268.htm
m.blog.qtbzn.cn/Article/details/7173559.htm
m.blog.qtbzn.cn/Article/details/0040822.htm
m.blog.qtbzn.cn/Article/details/3919573.htm
m.blog.qtbzn.cn/Article/details/3711757.htm
m.blog.qtbzn.cn/Article/details/9195557.htm
m.blog.qtbzn.cn/Article/details/8024224.htm
m.blog.qtbzn.cn/Article/details/1731197.htm
m.blog.qtbzn.cn/Article/details/0484068.htm
m.blog.qtbzn.cn/Article/details/3511573.htm
m.blog.qtbzn.cn/Article/details/3551913.htm
m.blog.qtbzn.cn/Article/details/7599913.htm
m.blog.qtbzn.cn/Article/details/4008880.htm
m.blog.qtbzn.cn/Article/details/9739515.htm
m.blog.qtbzn.cn/Article/details/1151551.htm
m.blog.qtbzn.cn/Article/details/6480224.htm
m.blog.qtbzn.cn/Article/details/3357311.htm
m.blog.qtbzn.cn/Article/details/5753757.htm
m.blog.qtbzn.cn/Article/details/7555393.htm
m.blog.qtbzn.cn/Article/details/2204664.htm
m.blog.qtbzn.cn/Article/details/7713915.htm
m.blog.qtbzn.cn/Article/details/9577595.htm
m.blog.qtbzn.cn/Article/details/7537793.htm
m.blog.qtbzn.cn/Article/details/8066462.htm
m.blog.qtbzn.cn/Article/details/0086842.htm
m.blog.qtbzn.cn/Article/details/0268888.htm
m.blog.qtbzn.cn/Article/details/5371719.htm
m.blog.qtbzn.cn/Article/details/0826884.htm
m.blog.qtbzn.cn/Article/details/3377551.htm
m.blog.qtbzn.cn/Article/details/7591599.htm
m.blog.qtbzn.cn/Article/details/8064402.htm
m.blog.qtbzn.cn/Article/details/7737993.htm
m.blog.qtbzn.cn/Article/details/6008280.htm
m.blog.qtbzn.cn/Article/details/0248662.htm
m.blog.qtbzn.cn/Article/details/6602804.htm
m.blog.qtbzn.cn/Article/details/1139553.htm
m.blog.qtbzn.cn/Article/details/4066086.htm
m.blog.qtbzn.cn/Article/details/7519337.htm
m.blog.qtbzn.cn/Article/details/1113971.htm
m.blog.qtbzn.cn/Article/details/4848206.htm
m.blog.qtbzn.cn/Article/details/8420066.htm
m.blog.qtbzn.cn/Article/details/3797771.htm
m.blog.qtbzn.cn/Article/details/4402648.htm
m.blog.qtbzn.cn/Article/details/7575533.htm
m.blog.qtbzn.cn/Article/details/6888664.htm
m.blog.qtbzn.cn/Article/details/4022244.htm
m.blog.qtbzn.cn/Article/details/1991151.htm
m.blog.qtbzn.cn/Article/details/1533113.htm
m.blog.qtbzn.cn/Article/details/3931513.htm
m.blog.qtbzn.cn/Article/details/7917199.htm
m.blog.qtbzn.cn/Article/details/3995391.htm
m.blog.qtbzn.cn/Article/details/2626406.htm
m.blog.qtbzn.cn/Article/details/5517591.htm
m.blog.qtbzn.cn/Article/details/6860244.htm
m.blog.qtbzn.cn/Article/details/3355933.htm
m.blog.qtbzn.cn/Article/details/9359779.htm
m.blog.qtbzn.cn/Article/details/1799355.htm
m.blog.qtbzn.cn/Article/details/9917533.htm
m.blog.qtbzn.cn/Article/details/9939913.htm
m.blog.qtbzn.cn/Article/details/6468684.htm
m.blog.qtbzn.cn/Article/details/7979959.htm
m.blog.qtbzn.cn/Article/details/9339753.htm
m.blog.qtbzn.cn/Article/details/3379117.htm
m.blog.qtbzn.cn/Article/details/9757513.htm
m.blog.qtbzn.cn/Article/details/5713133.htm
m.blog.qtbzn.cn/Article/details/3153391.htm
m.blog.qtbzn.cn/Article/details/3595337.htm
m.blog.qtbzn.cn/Article/details/6028048.htm
m.blog.qtbzn.cn/Article/details/2202884.htm
m.blog.qtbzn.cn/Article/details/2862206.htm
m.blog.qtbzn.cn/Article/details/9519973.htm
m.blog.qtbzn.cn/Article/details/9571917.htm
m.blog.qtbzn.cn/Article/details/2648642.htm
m.blog.qtbzn.cn/Article/details/5971771.htm
m.blog.qtbzn.cn/Article/details/7377735.htm
m.blog.qtbzn.cn/Article/details/5139339.htm
m.blog.qtbzn.cn/Article/details/4422026.htm
m.blog.qtbzn.cn/Article/details/3735555.htm
m.blog.qtbzn.cn/Article/details/6062068.htm
m.blog.qtbzn.cn/Article/details/7915319.htm
m.blog.qtbzn.cn/Article/details/0208446.htm
m.blog.qtbzn.cn/Article/details/1737191.htm
m.blog.qtbzn.cn/Article/details/2020842.htm
m.blog.qtbzn.cn/Article/details/7955353.htm
m.blog.qtbzn.cn/Article/details/6046846.htm
m.blog.qtbzn.cn/Article/details/7591779.htm
m.blog.qtbzn.cn/Article/details/7951379.htm
m.blog.qtbzn.cn/Article/details/3591315.htm
m.blog.qtbzn.cn/Article/details/6606044.htm
m.blog.qtbzn.cn/Article/details/9395777.htm
m.blog.qtbzn.cn/Article/details/3779713.htm
m.blog.qtbzn.cn/Article/details/5535171.htm
m.blog.qtbzn.cn/Article/details/1393973.htm
m.blog.qtbzn.cn/Article/details/7397953.htm
m.blog.qtbzn.cn/Article/details/2882004.htm
m.blog.qtbzn.cn/Article/details/3335315.htm
m.blog.qtbzn.cn/Article/details/5959717.htm
m.blog.qtbzn.cn/Article/details/2842206.htm
m.blog.qtbzn.cn/Article/details/5517975.htm
m.blog.qtbzn.cn/Article/details/1133753.htm
m.blog.qtbzn.cn/Article/details/2042266.htm
m.blog.qtbzn.cn/Article/details/4002462.htm
m.blog.qtbzn.cn/Article/details/6446484.htm
m.blog.qtbzn.cn/Article/details/3593573.htm
m.blog.qtbzn.cn/Article/details/3777973.htm
m.blog.qtbzn.cn/Article/details/5335319.htm
m.blog.qtbzn.cn/Article/details/1315717.htm
m.blog.qtbzn.cn/Article/details/8280264.htm
m.blog.qtbzn.cn/Article/details/7911775.htm
m.blog.qtbzn.cn/Article/details/7539119.htm
m.blog.qtbzn.cn/Article/details/1935373.htm
m.blog.qtbzn.cn/Article/details/6484086.htm
m.blog.qtbzn.cn/Article/details/8264242.htm
m.blog.qtbzn.cn/Article/details/1917739.htm
m.blog.qtbzn.cn/Article/details/1339395.htm
m.blog.qtbzn.cn/Article/details/5393377.htm
m.blog.qtbzn.cn/Article/details/1135191.htm
m.blog.qtbzn.cn/Article/details/1971355.htm
m.blog.qtbzn.cn/Article/details/3937733.htm
m.blog.qtbzn.cn/Article/details/9519937.htm
m.blog.qtbzn.cn/Article/details/1397111.htm
m.blog.qtbzn.cn/Article/details/9137595.htm
m.blog.qtbzn.cn/Article/details/0844644.htm
m.blog.qtbzn.cn/Article/details/5917379.htm
m.blog.qtbzn.cn/Article/details/9977575.htm
m.blog.qtbzn.cn/Article/details/4404608.htm
m.blog.qtbzn.cn/Article/details/4822202.htm
m.blog.qtbzn.cn/Article/details/9973753.htm
m.blog.qtbzn.cn/Article/details/0806888.htm
m.blog.qtbzn.cn/Article/details/8282844.htm
m.blog.qtbzn.cn/Article/details/6882080.htm
m.blog.qtbzn.cn/Article/details/8044260.htm
m.blog.qtbzn.cn/Article/details/3757975.htm
m.blog.qtbzn.cn/Article/details/4242226.htm
m.blog.qtbzn.cn/Article/details/2060268.htm
m.blog.qtbzn.cn/Article/details/8628860.htm
m.blog.qtbzn.cn/Article/details/0602860.htm
m.blog.qtbzn.cn/Article/details/8622004.htm
m.blog.qtbzn.cn/Article/details/8060420.htm
m.blog.qtbzn.cn/Article/details/7793737.htm
m.blog.qtbzn.cn/Article/details/0820640.htm
m.blog.qtbzn.cn/Article/details/5139533.htm
m.blog.qtbzn.cn/Article/details/5913111.htm
m.blog.qtbzn.cn/Article/details/1715739.htm
m.blog.qtbzn.cn/Article/details/1559593.htm
m.blog.qtbzn.cn/Article/details/5533177.htm
m.blog.qtbzn.cn/Article/details/7557995.htm
m.blog.qtbzn.cn/Article/details/7395951.htm
m.blog.qtbzn.cn/Article/details/7115931.htm
m.blog.qtbzn.cn/Article/details/1717999.htm
m.blog.qtbzn.cn/Article/details/7153557.htm
m.blog.qtbzn.cn/Article/details/0866688.htm
m.blog.qtbzn.cn/Article/details/6442462.htm
m.blog.qtbzn.cn/Article/details/0086882.htm
m.blog.qtbzn.cn/Article/details/5159579.htm
m.blog.qtbzn.cn/Article/details/2484444.htm
m.blog.qtbzn.cn/Article/details/3791739.htm
m.blog.qtbzn.cn/Article/details/1139937.htm
m.blog.qtbzn.cn/Article/details/3733939.htm
m.blog.qtbzn.cn/Article/details/2202242.htm
m.blog.qtbzn.cn/Article/details/5375591.htm
m.blog.qtbzn.cn/Article/details/9597939.htm
m.blog.qtbzn.cn/Article/details/1113355.htm
m.blog.qtbzn.cn/Article/details/4224844.htm
m.blog.qtbzn.cn/Article/details/7917155.htm
m.blog.qtbzn.cn/Article/details/1137797.htm
m.blog.qtbzn.cn/Article/details/2448860.htm
m.blog.qtbzn.cn/Article/details/3137551.htm
m.blog.qtbzn.cn/Article/details/2042644.htm
m.blog.qtbzn.cn/Article/details/7357175.htm
m.blog.qtbzn.cn/Article/details/5915939.htm
m.blog.qtbzn.cn/Article/details/3717135.htm
m.blog.qtbzn.cn/Article/details/9939179.htm
m.blog.qtbzn.cn/Article/details/7731593.htm
m.blog.qtbzn.cn/Article/details/7731133.htm
m.blog.qtbzn.cn/Article/details/2668620.htm
m.blog.qtbzn.cn/Article/details/0620224.htm
m.blog.qtbzn.cn/Article/details/8866844.htm
m.blog.qtbzn.cn/Article/details/9399599.htm
m.blog.qtbzn.cn/Article/details/2680400.htm
m.blog.qtbzn.cn/Article/details/8024646.htm
m.blog.qtbzn.cn/Article/details/4628426.htm
m.blog.qtbzn.cn/Article/details/7775331.htm
m.blog.qtbzn.cn/Article/details/6628602.htm
m.blog.qtbzn.cn/Article/details/7939333.htm
m.blog.qtbzn.cn/Article/details/1757911.htm
m.blog.qtbzn.cn/Article/details/7113719.htm
m.blog.qtbzn.cn/Article/details/9131395.htm
m.blog.qtbzn.cn/Article/details/6608444.htm
