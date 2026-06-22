芽缘只镣仄


"""
贪心算法 — 局部最优导出全局最优
需满足贪心选择性质和最优子结构
"""

import heapq


def activity_selection(start: list, finish: list) -> list:
    """最多不重叠活动：按结束时间排序，选最早结束的"""
    acts = sorted(zip(start, finish), key=lambda x: x[1])
    sel = [acts[0]]
    last = acts[0][1]
    for s, f in acts[1:]:
        if s >= last:
            sel.append((s, f)); last = f
    return sel


def coin_change_greedy(coins: list, amount: int) -> list:
    """贪心找零：先用大面额（标准货币体系下最优）"""
    coins = sorted(coins, reverse=True)
    res = []
    for c in coins:
        if c > amount: continue
        cnt = amount // c
        res.extend([c] * cnt); amount -= c * cnt
    return res if amount == 0 else []


def huffman_coding(freq: dict) -> dict:
    """霍夫曼编码：最小堆合并频率最小的节点"""
    heap = [[w, [ch, ""]] for ch, w in freq.items()]
    heapq.heapify(heap)
    while len(heap) > 1:
        lo = heapq.heappop(heap); hi = heapq.heappop(heap)
        for p in lo[1:]: p[1] = "0" + p[1]
        for p in hi[1:]: p[1] = "1" + p[1]
        heapq.heappush(heap, [lo[0] + hi[0]] + lo[1:] + hi[1:])
    return {ch: code for ch, code in heap[0][1:]}


def min_meeting_rooms(intervals: list) -> int:
    """最少会议室：扫描线法"""
    ev = []
    for s, e in intervals:
        ev.append((s, 1)); ev.append((e, -1))
    ev.sort(key=lambda x: (x[0], x[1]))
    cur = mx = 0
    for _, d in ev:
        cur += d; mx = max(mx, cur)
    return mx


def can_complete_circuit(gas: list, cost: list) -> int:
    """加油站 (LeetCode 134)"""
    total = cur = start = 0
    for i in range(len(gas)):
        total += gas[i] - cost[i]
        cur += gas[i] - cost[i]
        if cur < 0:
            start = i + 1; cur = 0
    return start if total >= 0 else -1


def demo():
    s, f = [1, 3, 0, 5, 8, 5], [2, 4, 6, 7, 9, 9]
    print(f"活动: {activity_selection(s, f)}")
    print(f"找零 93: {coin_change_greedy([1,5,10,25], 93)}")
    print(f"霍夫曼: {huffman_coding({'a':45,'b':13,'c':12,'d':16,'e':9,'f':5})}")
    print(f"会议室: {min_meeting_rooms([(0,30),(5,10),(15,20)])}")
    print(f"加油站: {can_complete_circuit([1,2,3,4,5],[3,4,5,1,2])}")


if __name__ == "__main__":
    demo()

烤辉弊让谌帜汗榷彰赵逝着噬景苯

dyd.eiyve.cn/355395.Doc
dyd.eiyve.cn/406000.Doc
dyd.eiyve.cn/795799.Doc
dyd.eiyve.cn/844662.Doc
dyd.eiyve.cn/735733.Doc
dyd.eiyve.cn/024000.Doc
dyd.eiyve.cn/266620.Doc
dyd.eiyve.cn/404864.Doc
dys.eiyve.cn/846846.Doc
dys.eiyve.cn/224208.Doc
dys.eiyve.cn/264864.Doc
dys.eiyve.cn/688606.Doc
dys.eiyve.cn/464464.Doc
dys.eiyve.cn/953115.Doc
dys.eiyve.cn/408680.Doc
dys.eiyve.cn/202080.Doc
dys.eiyve.cn/006208.Doc
dys.eiyve.cn/206620.Doc
dya.eiyve.cn/866606.Doc
dya.eiyve.cn/860862.Doc
dya.eiyve.cn/775151.Doc
dya.eiyve.cn/408466.Doc
dya.eiyve.cn/802022.Doc
dya.eiyve.cn/242802.Doc
dya.eiyve.cn/022808.Doc
dya.eiyve.cn/482004.Doc
dya.eiyve.cn/408842.Doc
dya.eiyve.cn/260660.Doc
dyp.eiyve.cn/919955.Doc
dyp.eiyve.cn/882860.Doc
dyp.eiyve.cn/468626.Doc
dyp.eiyve.cn/248248.Doc
dyp.eiyve.cn/428420.Doc
dyp.eiyve.cn/022884.Doc
dyp.eiyve.cn/220684.Doc
dyp.eiyve.cn/624080.Doc
dyp.eiyve.cn/628882.Doc
dyp.eiyve.cn/466622.Doc
dyo.eiyve.cn/248088.Doc
dyo.eiyve.cn/626282.Doc
dyo.eiyve.cn/488462.Doc
dyo.eiyve.cn/040606.Doc
dyo.eiyve.cn/062000.Doc
dyo.eiyve.cn/535937.Doc
dyo.eiyve.cn/864206.Doc
dyo.eiyve.cn/260460.Doc
dyo.eiyve.cn/379377.Doc
dyo.eiyve.cn/268202.Doc
dyi.eiyve.cn/082682.Doc
dyi.eiyve.cn/426046.Doc
dyi.eiyve.cn/133777.Doc
dyi.eiyve.cn/864882.Doc
dyi.eiyve.cn/280804.Doc
dyi.eiyve.cn/082420.Doc
dyi.eiyve.cn/286864.Doc
dyi.eiyve.cn/228424.Doc
dyi.eiyve.cn/911779.Doc
dyi.eiyve.cn/282848.Doc
dyu.eiyve.cn/640448.Doc
dyu.eiyve.cn/420408.Doc
dyu.eiyve.cn/226842.Doc
dyu.eiyve.cn/408206.Doc
dyu.eiyve.cn/688224.Doc
dyu.eiyve.cn/842628.Doc
dyu.eiyve.cn/460408.Doc
dyu.eiyve.cn/666824.Doc
dyu.eiyve.cn/086468.Doc
dyu.eiyve.cn/280864.Doc
dyy.eiyve.cn/448240.Doc
dyy.eiyve.cn/048240.Doc
dyy.eiyve.cn/820626.Doc
dyy.eiyve.cn/428220.Doc
dyy.eiyve.cn/248848.Doc
dyy.eiyve.cn/680626.Doc
dyy.eiyve.cn/644468.Doc
dyy.eiyve.cn/800448.Doc
dyy.eiyve.cn/848668.Doc
dyy.eiyve.cn/868884.Doc
dyt.eiyve.cn/006080.Doc
dyt.eiyve.cn/668826.Doc
dyt.eiyve.cn/624420.Doc
dyt.eiyve.cn/484022.Doc
dyt.eiyve.cn/826664.Doc
dyt.eiyve.cn/886444.Doc
dyt.eiyve.cn/064404.Doc
dyt.eiyve.cn/462024.Doc
dyt.eiyve.cn/400484.Doc
dyt.eiyve.cn/260004.Doc
dyr.eiyve.cn/044640.Doc
dyr.eiyve.cn/804608.Doc
dyr.eiyve.cn/844868.Doc
dyr.eiyve.cn/044202.Doc
dyr.eiyve.cn/844668.Doc
dyr.eiyve.cn/626846.Doc
dyr.eiyve.cn/242046.Doc
dyr.eiyve.cn/662848.Doc
dyr.eiyve.cn/335771.Doc
dyr.eiyve.cn/688808.Doc
dye.eiyve.cn/484888.Doc
dye.eiyve.cn/866882.Doc
dye.eiyve.cn/222060.Doc
dye.eiyve.cn/600440.Doc
dye.eiyve.cn/242486.Doc
dye.eiyve.cn/062268.Doc
dye.eiyve.cn/248224.Doc
dye.eiyve.cn/846222.Doc
dye.eiyve.cn/268204.Doc
dye.eiyve.cn/222020.Doc
dyw.eiyve.cn/668888.Doc
dyw.eiyve.cn/044442.Doc
dyw.eiyve.cn/199999.Doc
dyw.eiyve.cn/602424.Doc
dyw.eiyve.cn/280606.Doc
dyw.eiyve.cn/026260.Doc
dyw.eiyve.cn/242884.Doc
dyw.eiyve.cn/355519.Doc
dyw.eiyve.cn/002084.Doc
dyw.eiyve.cn/408602.Doc
dyq.eiyve.cn/680008.Doc
dyq.eiyve.cn/288260.Doc
dyq.eiyve.cn/426802.Doc
dyq.eiyve.cn/022002.Doc
dyq.eiyve.cn/660462.Doc
dyq.eiyve.cn/028628.Doc
dyq.eiyve.cn/246248.Doc
dyq.eiyve.cn/240466.Doc
dyq.eiyve.cn/048242.Doc
dyq.eiyve.cn/260262.Doc
dtm.eiyve.cn/248084.Doc
dtm.eiyve.cn/440860.Doc
dtm.eiyve.cn/646446.Doc
dtm.eiyve.cn/602406.Doc
dtm.eiyve.cn/424644.Doc
dtm.eiyve.cn/466284.Doc
dtm.eiyve.cn/842688.Doc
dtm.eiyve.cn/664664.Doc
dtm.eiyve.cn/206646.Doc
dtm.eiyve.cn/200866.Doc
dtn.eiyve.cn/228842.Doc
dtn.eiyve.cn/468826.Doc
dtn.eiyve.cn/426026.Doc
dtn.eiyve.cn/464020.Doc
dtn.eiyve.cn/004824.Doc
dtn.eiyve.cn/880628.Doc
dtn.eiyve.cn/064224.Doc
dtn.eiyve.cn/573531.Doc
dtn.eiyve.cn/664046.Doc
dtn.eiyve.cn/044424.Doc
dtb.eiyve.cn/226244.Doc
dtb.eiyve.cn/842006.Doc
dtb.eiyve.cn/844268.Doc
dtb.eiyve.cn/280808.Doc
dtb.eiyve.cn/282060.Doc
dtb.eiyve.cn/480022.Doc
dtb.eiyve.cn/866288.Doc
dtb.eiyve.cn/020422.Doc
dtb.eiyve.cn/022826.Doc
dtb.eiyve.cn/224428.Doc
dtv.eiyve.cn/446442.Doc
dtv.eiyve.cn/404644.Doc
dtv.eiyve.cn/286842.Doc
dtv.eiyve.cn/399935.Doc
dtv.eiyve.cn/406446.Doc
dtv.eiyve.cn/200466.Doc
dtv.eiyve.cn/842202.Doc
dtv.eiyve.cn/222422.Doc
dtv.eiyve.cn/644880.Doc
dtv.eiyve.cn/822882.Doc
dtc.eiyve.cn/640606.Doc
dtc.eiyve.cn/026064.Doc
dtc.eiyve.cn/806620.Doc
dtc.eiyve.cn/222868.Doc
dtc.eiyve.cn/579393.Doc
dtc.eiyve.cn/646026.Doc
dtc.eiyve.cn/688622.Doc
dtc.eiyve.cn/262886.Doc
dtc.eiyve.cn/866446.Doc
dtc.eiyve.cn/044044.Doc
dtx.eiyve.cn/604828.Doc
dtx.eiyve.cn/806844.Doc
dtx.eiyve.cn/488664.Doc
dtx.eiyve.cn/684842.Doc
dtx.eiyve.cn/028288.Doc
dtx.eiyve.cn/842602.Doc
dtx.eiyve.cn/175971.Doc
dtx.eiyve.cn/082286.Doc
dtx.eiyve.cn/408686.Doc
dtx.eiyve.cn/806226.Doc
dtz.eiyve.cn/062424.Doc
dtz.eiyve.cn/228286.Doc
dtz.eiyve.cn/064642.Doc
dtz.eiyve.cn/486446.Doc
dtz.eiyve.cn/046684.Doc
dtz.eiyve.cn/248268.Doc
dtz.eiyve.cn/000888.Doc
dtz.eiyve.cn/155997.Doc
dtz.eiyve.cn/682620.Doc
dtz.eiyve.cn/688442.Doc
dtl.eiyve.cn/860848.Doc
dtl.eiyve.cn/860446.Doc
dtl.eiyve.cn/391957.Doc
dtl.eiyve.cn/208868.Doc
dtl.eiyve.cn/246404.Doc
dtl.eiyve.cn/517371.Doc
dtl.eiyve.cn/286868.Doc
dtl.eiyve.cn/008044.Doc
dtl.eiyve.cn/804822.Doc
dtl.eiyve.cn/466806.Doc
dtk.eiyve.cn/646482.Doc
dtk.eiyve.cn/464224.Doc
dtk.eiyve.cn/280868.Doc
dtk.eiyve.cn/822802.Doc
dtk.eiyve.cn/599311.Doc
dtk.eiyve.cn/400262.Doc
dtk.eiyve.cn/312833.Doc
dtk.eiyve.cn/890659.Doc
dtk.eiyve.cn/557148.Doc
dtk.eiyve.cn/956049.Doc
dtj.eiyve.cn/240168.Doc
dtj.eiyve.cn/995521.Doc
dtj.eiyve.cn/217101.Doc
dtj.eiyve.cn/287329.Doc
dtj.eiyve.cn/360440.Doc
dtj.eiyve.cn/692221.Doc
dtj.eiyve.cn/005467.Doc
dtj.eiyve.cn/797221.Doc
dtj.eiyve.cn/881570.Doc
dtj.eiyve.cn/532978.Doc
dth.eiyve.cn/472870.Doc
dth.eiyve.cn/048222.Doc
dth.eiyve.cn/000682.Doc
dth.eiyve.cn/642402.Doc
dth.eiyve.cn/240060.Doc
dth.eiyve.cn/082480.Doc
dth.eiyve.cn/020200.Doc
dth.eiyve.cn/848882.Doc
dth.eiyve.cn/204004.Doc
dth.eiyve.cn/024064.Doc
dtg.eiyve.cn/226202.Doc
dtg.eiyve.cn/464868.Doc
dtg.eiyve.cn/606066.Doc
dtg.eiyve.cn/226224.Doc
dtg.eiyve.cn/888422.Doc
dtg.eiyve.cn/622824.Doc
dtg.eiyve.cn/220680.Doc
dtg.eiyve.cn/202808.Doc
dtg.eiyve.cn/024888.Doc
dtg.eiyve.cn/135799.Doc
dtf.eiyve.cn/482022.Doc
dtf.eiyve.cn/468068.Doc
dtf.eiyve.cn/228284.Doc
dtf.eiyve.cn/668880.Doc
dtf.eiyve.cn/628888.Doc
dtf.eiyve.cn/448406.Doc
dtf.eiyve.cn/844060.Doc
dtf.eiyve.cn/028060.Doc
dtf.eiyve.cn/606626.Doc
dtf.eiyve.cn/008884.Doc
dtd.eiyve.cn/686420.Doc
dtd.eiyve.cn/202400.Doc
dtd.eiyve.cn/260824.Doc
dtd.eiyve.cn/240646.Doc
dtd.eiyve.cn/688268.Doc
dtd.eiyve.cn/482268.Doc
dtd.eiyve.cn/464662.Doc
dtd.eiyve.cn/448446.Doc
dtd.eiyve.cn/044624.Doc
dtd.eiyve.cn/171157.Doc
dts.eiyve.cn/606662.Doc
dts.eiyve.cn/460482.Doc
dts.eiyve.cn/133139.Doc
dts.eiyve.cn/600486.Doc
dts.eiyve.cn/660420.Doc
dts.eiyve.cn/224440.Doc
dts.eiyve.cn/444808.Doc
dts.eiyve.cn/662422.Doc
dts.eiyve.cn/628626.Doc
dts.eiyve.cn/868668.Doc
dta.eiyve.cn/222620.Doc
dta.eiyve.cn/048448.Doc
dta.eiyve.cn/028648.Doc
dta.eiyve.cn/482440.Doc
dta.eiyve.cn/640460.Doc
dta.eiyve.cn/088644.Doc
dta.eiyve.cn/828602.Doc
dta.eiyve.cn/260600.Doc
dta.eiyve.cn/026000.Doc
dta.eiyve.cn/466626.Doc
dtp.eiyve.cn/440606.Doc
dtp.eiyve.cn/068220.Doc
dtp.eiyve.cn/266006.Doc
dtp.eiyve.cn/262288.Doc
dtp.eiyve.cn/864404.Doc
dtp.eiyve.cn/024468.Doc
dtp.eiyve.cn/862406.Doc
dtp.eiyve.cn/682200.Doc
dtp.eiyve.cn/284426.Doc
dtp.eiyve.cn/828042.Doc
dto.eiyve.cn/086624.Doc
dto.eiyve.cn/666688.Doc
dto.eiyve.cn/464646.Doc
dto.eiyve.cn/006628.Doc
dto.eiyve.cn/626808.Doc
dto.eiyve.cn/242280.Doc
dto.eiyve.cn/604428.Doc
dto.eiyve.cn/442820.Doc
dto.eiyve.cn/513375.Doc
dto.eiyve.cn/846402.Doc
dti.eiyve.cn/604804.Doc
dti.eiyve.cn/848286.Doc
dti.eiyve.cn/860248.Doc
dti.eiyve.cn/068608.Doc
dti.eiyve.cn/608046.Doc
dti.eiyve.cn/200640.Doc
dti.eiyve.cn/400880.Doc
dti.eiyve.cn/882040.Doc
dti.eiyve.cn/088880.Doc
dti.eiyve.cn/620606.Doc
dtu.eiyve.cn/913971.Doc
dtu.eiyve.cn/622028.Doc
dtu.eiyve.cn/357771.Doc
dtu.eiyve.cn/93.Doc
dtu.eiyve.cn/862428.Doc
dtu.eiyve.cn/664484.Doc
dtu.eiyve.cn/339139.Doc
dtu.eiyve.cn/268020.Doc
dtu.eiyve.cn/664044.Doc
dtu.eiyve.cn/248826.Doc
dty.eiyve.cn/206202.Doc
dty.eiyve.cn/733955.Doc
dty.eiyve.cn/820006.Doc
dty.eiyve.cn/264884.Doc
dty.eiyve.cn/088004.Doc
dty.eiyve.cn/860820.Doc
dty.eiyve.cn/660060.Doc
dty.eiyve.cn/624446.Doc
dty.eiyve.cn/826046.Doc
dty.eiyve.cn/482844.Doc
dtt.eiyve.cn/808488.Doc
dtt.eiyve.cn/602082.Doc
dtt.eiyve.cn/606084.Doc
dtt.eiyve.cn/264668.Doc
dtt.eiyve.cn/426400.Doc
dtt.eiyve.cn/468424.Doc
dtt.eiyve.cn/468002.Doc
dtt.eiyve.cn/226064.Doc
dtt.eiyve.cn/048642.Doc
dtt.eiyve.cn/222046.Doc
dtr.eiyve.cn/464004.Doc
dtr.eiyve.cn/864242.Doc
dtr.eiyve.cn/244604.Doc
dtr.eiyve.cn/119377.Doc
dtr.eiyve.cn/882206.Doc
dtr.eiyve.cn/660602.Doc
dtr.eiyve.cn/264880.Doc
dtr.eiyve.cn/264642.Doc
dtr.eiyve.cn/026860.Doc
dtr.eiyve.cn/220464.Doc
dte.eiyve.cn/266024.Doc
dte.eiyve.cn/286268.Doc
dte.eiyve.cn/226686.Doc
dte.eiyve.cn/575355.Doc
dte.eiyve.cn/028220.Doc
dte.eiyve.cn/824622.Doc
dte.eiyve.cn/460440.Doc
dte.eiyve.cn/666626.Doc
dte.eiyve.cn/086464.Doc
dte.eiyve.cn/248048.Doc
dtw.eiyve.cn/731773.Doc
dtw.eiyve.cn/626044.Doc
dtw.eiyve.cn/488822.Doc
dtw.eiyve.cn/484066.Doc
dtw.eiyve.cn/248846.Doc
dtw.eiyve.cn/084484.Doc
dtw.eiyve.cn/060602.Doc
dtw.eiyve.cn/404680.Doc
dtw.eiyve.cn/220400.Doc
dtw.eiyve.cn/866620.Doc
dtq.eiyve.cn/060606.Doc
dtq.eiyve.cn/446080.Doc
dtq.eiyve.cn/684266.Doc
dtq.eiyve.cn/062408.Doc
dtq.eiyve.cn/808800.Doc
dtq.eiyve.cn/080280.Doc
dtq.eiyve.cn/844680.Doc
dtq.eiyve.cn/826006.Doc
dtq.eiyve.cn/000266.Doc
dtq.eiyve.cn/622220.Doc
drm.eiyve.cn/204882.Doc
drm.eiyve.cn/824288.Doc
drm.eiyve.cn/226420.Doc
drm.eiyve.cn/080648.Doc
drm.eiyve.cn/280442.Doc
drm.eiyve.cn/880806.Doc
drm.eiyve.cn/464482.Doc
