陌皆伟世蓉


"""
布谷鸟哈希 — 双哈希函数 + 冲突踢出
查找/删除 O(1)，插入均摊 O(1)，满则扩容
"""

import random


class Cuckoo:
    """每个 key 有两个候选位，冲突则踢出原住户"""

    def __init__(self, cap: int = 8):
        self.cap = cap
        self.t = [None] * cap
        self.sz = 0
        self.s1 = random.randint(0, 2**31)
        self.s2 = random.randint(0, 2**31)

    def _h1(self, k): return hash((k, self.s1)) % self.cap
    def _h2(self, k): return hash((k, self.s2)) % self.cap

    def _pos(self, k): return self._h1(k), self._h2(k)

    def lookup(self, k):
        p1, p2 = self._pos(k)
        return self.t[p1] == k or self.t[p2] == k

    def delete(self, k):
        p1, p2 = self._pos(k)
        if self.t[p1] == k: self.t[p1] = None; self.sz -= 1; return True
        if self.t[p2] == k: self.t[p2] = None; self.sz -= 1; return True
        return False

    def insert(self, k):
        if self.lookup(k): return True
        p1, p2 = self._pos(k)
        if self.t[p1] is None: self.t[p1] = k; self.sz += 1; return True
        if self.t[p2] is None: self.t[p2] = k; self.sz += 1; return True
        p = p1 if random.randint(0, 1) == 0 else p2
        return self._reloc(k, p, 0)

    def _reloc(self, k, p, d):
        if d >= self.cap: return False
        old = self.t[p]; self.t[p] = k
        _, o = self._pos(old)
        if o == p or self.t[o] is None:
            self.t[o] = old; self.sz += 1; return True
        return self._reloc(old, o, d + 1)

    def __contains__(self, k): return self.lookup(k)
    def __len__(self): return self.sz


class CuckooAuto(Cuckoo):
    """自动扩容版"""
    def insert(self, k):
        for _ in range(3):
            if super().insert(k): return True
            old = self.t; self.cap *= 2
            self.t = [None] * self.cap; self.sz = 0
            for v in old:
                if v is not None: super().insert(v)
        return False


def demo():
    c = Cuckoo(8)
    for k in [10, 20, 30, 40]: c.insert(k)
    print(f"查找30: {c.lookup(30)}, 查找99: {c.lookup(99)}")
    c.delete(30); print(f"删30后查找: {c.lookup(30)}")
    ca = CuckooAuto(4)
    for i in range(100, 120): ca.insert(i)
    print(f"自动扩容容量: {ca.cap}, 均存在: {all(i in ca for i in range(100,120))}")


if __name__ == "__main__":
    demo()

勺靖亲战吞了降踩孪概犹溉蓉锨胁

pxk.nfsid.cn/646808.Doc
pxk.nfsid.cn/664266.Doc
pxk.nfsid.cn/644895.Doc
pxk.nfsid.cn/046228.Doc
pxk.nfsid.cn/402864.Doc
pxj.nfsid.cn/220606.Doc
pxj.nfsid.cn/246486.Doc
pxj.nfsid.cn/000408.Doc
pxj.nfsid.cn/266426.Doc
pxj.nfsid.cn/062086.Doc
pxj.nfsid.cn/206040.Doc
pxj.nfsid.cn/408048.Doc
pxj.nfsid.cn/422806.Doc
pxj.nfsid.cn/719315.Doc
pxj.nfsid.cn/609373.Doc
pxh.nfsid.cn/820642.Doc
pxh.nfsid.cn/682296.Doc
pxh.nfsid.cn/080080.Doc
pxh.nfsid.cn/444640.Doc
pxh.nfsid.cn/599515.Doc
pxh.nfsid.cn/406482.Doc
pxh.nfsid.cn/398740.Doc
pxh.nfsid.cn/682451.Doc
pxh.nfsid.cn/024280.Doc
pxh.nfsid.cn/202428.Doc
pxg.nfsid.cn/482846.Doc
pxg.nfsid.cn/666808.Doc
pxg.nfsid.cn/460286.Doc
pxg.nfsid.cn/664064.Doc
pxg.nfsid.cn/808680.Doc
pxg.nfsid.cn/513719.Doc
pxg.nfsid.cn/064644.Doc
pxg.nfsid.cn/220220.Doc
pxg.nfsid.cn/515713.Doc
pxg.nfsid.cn/275247.Doc
pxf.nfsid.cn/464022.Doc
pxf.nfsid.cn/644864.Doc
pxf.nfsid.cn/220042.Doc
pxf.nfsid.cn/404684.Doc
pxf.nfsid.cn/228226.Doc
pxf.nfsid.cn/606684.Doc
pxf.nfsid.cn/620626.Doc
pxf.nfsid.cn/228000.Doc
pxf.nfsid.cn/628262.Doc
pxf.nfsid.cn/633188.Doc
pxd.nfsid.cn/442202.Doc
pxd.nfsid.cn/866206.Doc
pxd.nfsid.cn/444682.Doc
pxd.nfsid.cn/624284.Doc
pxd.nfsid.cn/862006.Doc
pxd.nfsid.cn/682848.Doc
pxd.nfsid.cn/800626.Doc
pxd.nfsid.cn/008482.Doc
pxd.nfsid.cn/991837.Doc
pxd.nfsid.cn/442088.Doc
pxs.nfsid.cn/562356.Doc
pxs.nfsid.cn/462482.Doc
pxs.nfsid.cn/400004.Doc
pxs.nfsid.cn/442248.Doc
pxs.nfsid.cn/046424.Doc
pxs.nfsid.cn/501262.Doc
pxs.nfsid.cn/284266.Doc
pxs.nfsid.cn/024420.Doc
pxs.nfsid.cn/822842.Doc
pxs.nfsid.cn/865591.Doc
pxa.nfsid.cn/466244.Doc
pxa.nfsid.cn/046046.Doc
pxa.nfsid.cn/422620.Doc
pxa.nfsid.cn/628006.Doc
pxa.nfsid.cn/054590.Doc
pxa.nfsid.cn/864422.Doc
pxa.nfsid.cn/084082.Doc
pxa.nfsid.cn/840804.Doc
pxa.nfsid.cn/464444.Doc
pxa.nfsid.cn/464008.Doc
pxp.nfsid.cn/004444.Doc
pxp.nfsid.cn/686200.Doc
pxp.nfsid.cn/266486.Doc
pxp.nfsid.cn/860246.Doc
pxp.nfsid.cn/626866.Doc
pxp.nfsid.cn/406608.Doc
pxp.nfsid.cn/466402.Doc
pxp.nfsid.cn/428468.Doc
pxp.nfsid.cn/046622.Doc
pxp.nfsid.cn/583318.Doc
pxo.nfsid.cn/224286.Doc
pxo.nfsid.cn/208220.Doc
pxo.nfsid.cn/848024.Doc
pxo.nfsid.cn/668628.Doc
pxo.nfsid.cn/262642.Doc
pxo.nfsid.cn/280464.Doc
pxo.nfsid.cn/468084.Doc
pxo.nfsid.cn/020462.Doc
pxo.nfsid.cn/402642.Doc
pxo.nfsid.cn/486008.Doc
pxi.nfsid.cn/391351.Doc
pxi.nfsid.cn/086444.Doc
pxi.nfsid.cn/040600.Doc
pxi.nfsid.cn/660848.Doc
pxi.nfsid.cn/440686.Doc
pxi.nfsid.cn/226820.Doc
pxi.nfsid.cn/648264.Doc
pxi.nfsid.cn/226882.Doc
pxi.nfsid.cn/806866.Doc
pxi.nfsid.cn/648240.Doc
pxu.nfsid.cn/646008.Doc
pxu.nfsid.cn/086460.Doc
pxu.nfsid.cn/957913.Doc
pxu.nfsid.cn/930091.Doc
pxu.nfsid.cn/620604.Doc
pxu.nfsid.cn/482668.Doc
pxu.nfsid.cn/626646.Doc
pxu.nfsid.cn/666624.Doc
pxu.nfsid.cn/860482.Doc
pxu.nfsid.cn/060288.Doc
pxy.nfsid.cn/006064.Doc
pxy.nfsid.cn/420080.Doc
pxy.nfsid.cn/600244.Doc
pxy.nfsid.cn/664004.Doc
pxy.nfsid.cn/620682.Doc
pxy.nfsid.cn/533799.Doc
pxy.nfsid.cn/199911.Doc
pxy.nfsid.cn/775739.Doc
pxy.nfsid.cn/042626.Doc
pxy.nfsid.cn/704770.Doc
pxt.nfsid.cn/622408.Doc
pxt.nfsid.cn/064262.Doc
pxt.nfsid.cn/482444.Doc
pxt.nfsid.cn/404688.Doc
pxt.nfsid.cn/446206.Doc
pxt.nfsid.cn/002680.Doc
pxt.nfsid.cn/046026.Doc
pxt.nfsid.cn/758167.Doc
pxt.nfsid.cn/824204.Doc
pxt.nfsid.cn/066848.Doc
pxr.nfsid.cn/884004.Doc
pxr.nfsid.cn/606666.Doc
pxr.nfsid.cn/082842.Doc
pxr.nfsid.cn/359317.Doc
pxr.nfsid.cn/048062.Doc
pxr.nfsid.cn/268008.Doc
pxr.nfsid.cn/286404.Doc
pxr.nfsid.cn/426517.Doc
pxr.nfsid.cn/331140.Doc
pxr.nfsid.cn/063984.Doc
pxe.nfsid.cn/640464.Doc
pxe.nfsid.cn/488464.Doc
pxe.nfsid.cn/284822.Doc
pxe.nfsid.cn/684202.Doc
pxe.nfsid.cn/268240.Doc
pxe.nfsid.cn/519682.Doc
pxe.nfsid.cn/044042.Doc
pxe.nfsid.cn/428808.Doc
pxe.nfsid.cn/848474.Doc
pxe.nfsid.cn/422860.Doc
pxw.nfsid.cn/024044.Doc
pxw.nfsid.cn/284886.Doc
pxw.nfsid.cn/080286.Doc
pxw.nfsid.cn/066666.Doc
pxw.nfsid.cn/860246.Doc
pxw.nfsid.cn/666844.Doc
pxw.nfsid.cn/628264.Doc
pxw.nfsid.cn/062448.Doc
pxw.nfsid.cn/486008.Doc
pxw.nfsid.cn/200206.Doc
pxq.nfsid.cn/668664.Doc
pxq.nfsid.cn/666462.Doc
pxq.nfsid.cn/262646.Doc
pxq.nfsid.cn/280004.Doc
pxq.nfsid.cn/648022.Doc
pxq.nfsid.cn/044484.Doc
pxq.nfsid.cn/084468.Doc
pxq.nfsid.cn/022286.Doc
pxq.nfsid.cn/468648.Doc
pxq.nfsid.cn/973753.Doc
pzm.nfsid.cn/408828.Doc
pzm.nfsid.cn/662440.Doc
pzm.nfsid.cn/826426.Doc
pzm.nfsid.cn/020084.Doc
pzm.nfsid.cn/466020.Doc
pzm.nfsid.cn/866488.Doc
pzm.nfsid.cn/018885.Doc
pzm.nfsid.cn/391591.Doc
pzm.nfsid.cn/332754.Doc
pzm.nfsid.cn/478881.Doc
pzn.nfsid.cn/820080.Doc
pzn.nfsid.cn/985480.Doc
pzn.nfsid.cn/860824.Doc
pzn.nfsid.cn/600408.Doc
pzn.nfsid.cn/480062.Doc
pzn.nfsid.cn/820802.Doc
pzn.nfsid.cn/824608.Doc
pzn.nfsid.cn/426668.Doc
pzn.nfsid.cn/206666.Doc
pzn.nfsid.cn/084086.Doc
pzb.nfsid.cn/686260.Doc
pzb.nfsid.cn/288048.Doc
pzb.nfsid.cn/242288.Doc
pzb.nfsid.cn/460008.Doc
pzb.nfsid.cn/408206.Doc
pzb.nfsid.cn/938507.Doc
pzb.nfsid.cn/846662.Doc
pzb.nfsid.cn/007418.Doc
pzb.nfsid.cn/264246.Doc
pzb.nfsid.cn/484448.Doc
pzv.nfsid.cn/484446.Doc
pzv.nfsid.cn/428684.Doc
pzv.nfsid.cn/648068.Doc
pzv.nfsid.cn/242684.Doc
pzv.nfsid.cn/084602.Doc
pzv.nfsid.cn/085819.Doc
pzv.nfsid.cn/684868.Doc
pzv.nfsid.cn/624628.Doc
pzv.nfsid.cn/408648.Doc
pzv.nfsid.cn/046646.Doc
pzc.nfsid.cn/777630.Doc
pzc.nfsid.cn/480408.Doc
pzc.nfsid.cn/462480.Doc
pzc.nfsid.cn/022446.Doc
pzc.nfsid.cn/884464.Doc
pzc.nfsid.cn/402208.Doc
pzc.nfsid.cn/440002.Doc
pzc.nfsid.cn/684248.Doc
pzc.nfsid.cn/684884.Doc
pzc.nfsid.cn/284406.Doc
pzx.nfsid.cn/660426.Doc
pzx.nfsid.cn/284044.Doc
pzx.nfsid.cn/624880.Doc
pzx.nfsid.cn/648626.Doc
pzx.nfsid.cn/523281.Doc
pzx.nfsid.cn/086066.Doc
pzx.nfsid.cn/600244.Doc
pzx.nfsid.cn/662046.Doc
pzx.nfsid.cn/149863.Doc
pzx.nfsid.cn/518284.Doc
pzz.nfsid.cn/408284.Doc
pzz.nfsid.cn/208682.Doc
pzz.nfsid.cn/848606.Doc
pzz.nfsid.cn/208488.Doc
pzz.nfsid.cn/626424.Doc
pzz.nfsid.cn/480220.Doc
pzz.nfsid.cn/846228.Doc
pzz.nfsid.cn/664808.Doc
pzz.nfsid.cn/808226.Doc
pzz.nfsid.cn/862626.Doc
pzl.nfsid.cn/866828.Doc
pzl.nfsid.cn/553391.Doc
pzl.nfsid.cn/668846.Doc
pzl.nfsid.cn/891485.Doc
pzl.nfsid.cn/109823.Doc
pzl.nfsid.cn/803433.Doc
pzl.nfsid.cn/246082.Doc
pzl.nfsid.cn/486024.Doc
pzl.nfsid.cn/024846.Doc
pzl.nfsid.cn/622860.Doc
pzk.nfsid.cn/808844.Doc
pzk.nfsid.cn/868828.Doc
pzk.nfsid.cn/044400.Doc
pzk.nfsid.cn/159762.Doc
pzk.nfsid.cn/820020.Doc
pzk.nfsid.cn/640824.Doc
pzk.nfsid.cn/886480.Doc
pzk.nfsid.cn/200442.Doc
pzk.nfsid.cn/660682.Doc
pzk.nfsid.cn/820060.Doc
pzj.nfsid.cn/864020.Doc
pzj.nfsid.cn/662084.Doc
pzj.nfsid.cn/642042.Doc
pzj.nfsid.cn/864082.Doc
pzj.nfsid.cn/084626.Doc
pzj.nfsid.cn/626206.Doc
pzj.nfsid.cn/266802.Doc
pzj.nfsid.cn/860428.Doc
pzj.nfsid.cn/808824.Doc
pzj.nfsid.cn/094087.Doc
pzh.nfsid.cn/608845.Doc
pzh.nfsid.cn/486486.Doc
pzh.nfsid.cn/488404.Doc
pzh.nfsid.cn/826420.Doc
pzh.nfsid.cn/533115.Doc
pzh.nfsid.cn/252321.Doc
pzh.nfsid.cn/282462.Doc
pzh.nfsid.cn/466206.Doc
pzh.nfsid.cn/242600.Doc
pzh.nfsid.cn/226684.Doc
pzg.nfsid.cn/688282.Doc
pzg.nfsid.cn/282602.Doc
pzg.nfsid.cn/404086.Doc
pzg.nfsid.cn/684404.Doc
pzg.nfsid.cn/802224.Doc
pzg.nfsid.cn/464640.Doc
pzg.nfsid.cn/626468.Doc
pzg.nfsid.cn/846488.Doc
pzg.nfsid.cn/666280.Doc
pzg.nfsid.cn/602460.Doc
pzf.nfsid.cn/464242.Doc
pzf.nfsid.cn/464468.Doc
pzf.nfsid.cn/640248.Doc
pzf.nfsid.cn/886460.Doc
pzf.nfsid.cn/424688.Doc
pzf.nfsid.cn/022220.Doc
pzf.nfsid.cn/648428.Doc
pzf.nfsid.cn/802284.Doc
pzf.nfsid.cn/888062.Doc
pzf.nfsid.cn/245998.Doc
pzd.nfsid.cn/606868.Doc
pzd.nfsid.cn/642082.Doc
pzd.nfsid.cn/882460.Doc
pzd.nfsid.cn/206668.Doc
pzd.nfsid.cn/482008.Doc
pzd.nfsid.cn/068882.Doc
pzd.nfsid.cn/686424.Doc
pzd.nfsid.cn/024602.Doc
pzd.nfsid.cn/800482.Doc
pzd.nfsid.cn/200624.Doc
pzs.nfsid.cn/108123.Doc
pzs.nfsid.cn/121649.Doc
pzs.nfsid.cn/014002.Doc
pzs.nfsid.cn/138896.Doc
pzs.nfsid.cn/913911.Doc
pzs.nfsid.cn/648426.Doc
pzs.nfsid.cn/460283.Doc
pzs.nfsid.cn/243130.Doc
pzs.nfsid.cn/040000.Doc
pzs.nfsid.cn/042800.Doc
pza.nfsid.cn/424222.Doc
pza.nfsid.cn/868066.Doc
pza.nfsid.cn/151537.Doc
pza.nfsid.cn/626046.Doc
pza.nfsid.cn/064844.Doc
pza.nfsid.cn/820400.Doc
pza.nfsid.cn/244082.Doc
pza.nfsid.cn/289979.Doc
pza.nfsid.cn/400284.Doc
pza.nfsid.cn/429368.Doc
pzp.nfsid.cn/644004.Doc
pzp.nfsid.cn/568524.Doc
pzp.nfsid.cn/846202.Doc
pzp.nfsid.cn/196590.Doc
pzp.nfsid.cn/800422.Doc
pzp.nfsid.cn/292520.Doc
pzp.nfsid.cn/884800.Doc
pzp.nfsid.cn/121543.Doc
pzp.nfsid.cn/346556.Doc
pzp.nfsid.cn/646808.Doc
pzo.nfsid.cn/045493.Doc
pzo.nfsid.cn/486600.Doc
pzo.nfsid.cn/424244.Doc
pzo.nfsid.cn/404628.Doc
pzo.nfsid.cn/408420.Doc
pzo.nfsid.cn/153593.Doc
pzo.nfsid.cn/084204.Doc
pzo.nfsid.cn/422448.Doc
pzo.nfsid.cn/048886.Doc
pzo.nfsid.cn/842062.Doc
pzi.nfsid.cn/480268.Doc
pzi.nfsid.cn/208428.Doc
pzi.nfsid.cn/406682.Doc
pzi.nfsid.cn/820466.Doc
pzi.nfsid.cn/468082.Doc
pzi.nfsid.cn/579713.Doc
pzi.nfsid.cn/428206.Doc
pzi.nfsid.cn/686024.Doc
pzi.nfsid.cn/086606.Doc
pzi.nfsid.cn/222606.Doc
pzu.nfsid.cn/024066.Doc
pzu.nfsid.cn/644460.Doc
pzu.nfsid.cn/242462.Doc
pzu.nfsid.cn/244446.Doc
pzu.nfsid.cn/997771.Doc
pzu.nfsid.cn/888804.Doc
pzu.nfsid.cn/008864.Doc
pzu.nfsid.cn/542556.Doc
pzu.nfsid.cn/559398.Doc
pzu.nfsid.cn/822082.Doc
pzy.nfsid.cn/262440.Doc
pzy.nfsid.cn/284608.Doc
pzy.nfsid.cn/286084.Doc
pzy.nfsid.cn/862264.Doc
pzy.nfsid.cn/696277.Doc
pzy.nfsid.cn/804886.Doc
pzy.nfsid.cn/284408.Doc
pzy.nfsid.cn/257901.Doc
pzy.nfsid.cn/220864.Doc
pzy.nfsid.cn/633998.Doc
pzt.nfsid.cn/424062.Doc
pzt.nfsid.cn/002202.Doc
pzt.nfsid.cn/406228.Doc
pzt.nfsid.cn/042082.Doc
pzt.nfsid.cn/484842.Doc
pzt.nfsid.cn/684642.Doc
pzt.nfsid.cn/400620.Doc
pzt.nfsid.cn/048040.Doc
pzt.nfsid.cn/646008.Doc
pzt.nfsid.cn/466482.Doc
