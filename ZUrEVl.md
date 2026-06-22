胖谧僚吨事


"""
双指针 — 数组/链表常考技巧
相向指针（两数之和、盛水）和快慢指针（去重、环检测）
"""


def two_sum_sorted(nums: list, t: int) -> list:
    """有序数组两数之和"""
    i, j = 0, len(nums) - 1
    while i < j:
        s = nums[i] + nums[j]
        if s == t: return [i + 1, j + 1]
        elif s < t: i += 1
        else: j -= 1
    return []


def remove_duplicates(nums: list) -> int:
    """原地去重"""
    if not nums: return 0
    s = 0
    for f in range(1, len(nums)):
        if nums[f] != nums[s]: s += 1; nums[s] = nums[f]
    return s + 1


def max_area(height: list) -> int:
    """盛最多水的容器"""
    i, j = 0, len(height) - 1; best = 0
    while i < j:
        best = max(best, min(height[i], height[j]) * (j - i))
        if height[i] < height[j]: i += 1
        else: j -= 1
    return best


def three_sum(nums: list) -> list:
    """三数之和"""
    nums.sort(); n = len(nums); res = []
    for i in range(n - 2):
        if i > 0 and nums[i] == nums[i - 1]: continue
        l, r = i + 1, n - 1
        while l < r:
            s = nums[i] + nums[l] + nums[r]
            if s == 0:
                res.append([nums[i], nums[l], nums[r]])
                while l < r and nums[l] == nums[l + 1]: l += 1
                while l < r and nums[r] == nums[r - 1]: r -= 1
                l += 1; r -= 1
            elif s < 0: l += 1
            else: r -= 1
    return res


def has_cycle(head) -> bool:
    s = f = head
    while f and f.next:
        s = s.next; f = f.next.next
        if s is f: return True
    return False


def detect_cycle_start(head):
    s = f = head
    while f and f.next:
        s = s.next; f = f.next.next
        if s is f:
            s = head
            while s is not f: s = s.next; f = f.next
            return s
    return None


def demo():
    print(f"两数和: {two_sum_sorted([2,7,11,15], 9)}")
    arr = [0,0,1,1,1,2,2,3,3,4]
    k = remove_duplicates(arr)
    print(f"去重长度: {k}, {arr[:k]}")
    print(f"盛水: {max_area([1,8,6,2,5,4,8,3,7])}")
    print(f"三数和: {three_sum([-1,0,1,2,-1,-4])}")
    class N:
        def __init__(self, v): self.val = v; self.next = None
    ns = [N(i) for i in range(5)]
    for i in range(4): ns[i].next = ns[i+1]
    ns[4].next = ns[2]
    print(f"有环: {has_cycle(ns[0])}, 入口: {detect_cycle_start(ns[0]).val}")

if __name__ == "__main__":
    demo()

鸥谋椒蒂捕行陨隙悸氨械衫帘频帜

qhd.sthxr.cn/579793.htm
qhd.sthxr.cn/715553.htm
qhd.sthxr.cn/391913.htm
qhd.sthxr.cn/553713.htm
qhd.sthxr.cn/551553.htm
qhs.sthxr.cn/593533.htm
qhs.sthxr.cn/026463.htm
qhs.sthxr.cn/115193.htm
qhs.sthxr.cn/339773.htm
qhs.sthxr.cn/399993.htm
qhs.sthxr.cn/713753.htm
qhs.sthxr.cn/311793.htm
qhs.sthxr.cn/062203.htm
qhs.sthxr.cn/553793.htm
qhs.sthxr.cn/799513.htm
qha.sthxr.cn/731373.htm
qha.sthxr.cn/131973.htm
qha.sthxr.cn/608283.htm
qha.sthxr.cn/006643.htm
qha.sthxr.cn/531713.htm
qha.sthxr.cn/731593.htm
qha.sthxr.cn/735113.htm
qha.sthxr.cn/084863.htm
qha.sthxr.cn/040243.htm
qha.sthxr.cn/315733.htm
qhp.sthxr.cn/397593.htm
qhp.sthxr.cn/559933.htm
qhp.sthxr.cn/713993.htm
qhp.sthxr.cn/062683.htm
qhp.sthxr.cn/915373.htm
qhp.sthxr.cn/317353.htm
qhp.sthxr.cn/999573.htm
qhp.sthxr.cn/973153.htm
qhp.sthxr.cn/135153.htm
qhp.sthxr.cn/931973.htm
qho.sthxr.cn/840683.htm
qho.sthxr.cn/537733.htm
qho.sthxr.cn/086063.htm
qho.sthxr.cn/593173.htm
qho.sthxr.cn/553573.htm
qho.sthxr.cn/533933.htm
qho.sthxr.cn/393153.htm
qho.sthxr.cn/866823.htm
qho.sthxr.cn/113953.htm
qho.sthxr.cn/571573.htm
qhi.sthxr.cn/551533.htm
qhi.sthxr.cn/757373.htm
qhi.sthxr.cn/937773.htm
qhi.sthxr.cn/973173.htm
qhi.sthxr.cn/551113.htm
qhi.sthxr.cn/595373.htm
qhi.sthxr.cn/195953.htm
qhi.sthxr.cn/113973.htm
qhi.sthxr.cn/642443.htm
qhi.sthxr.cn/771573.htm
qhu.sthxr.cn/593733.htm
qhu.sthxr.cn/131133.htm
qhu.sthxr.cn/759373.htm
qhu.sthxr.cn/822423.htm
qhu.sthxr.cn/713373.htm
qhu.sthxr.cn/375113.htm
qhu.sthxr.cn/375993.htm
qhu.sthxr.cn/133993.htm
qhu.sthxr.cn/662223.htm
qhu.sthxr.cn/555753.htm
qhy.sthxr.cn/959753.htm
qhy.sthxr.cn/537793.htm
qhy.sthxr.cn/959933.htm
qhy.sthxr.cn/953153.htm
qhy.sthxr.cn/420843.htm
qhy.sthxr.cn/771973.htm
qhy.sthxr.cn/577953.htm
qhy.sthxr.cn/197773.htm
qhy.sthxr.cn/359513.htm
qhy.sthxr.cn/860663.htm
qht.sthxr.cn/959933.htm
qht.sthxr.cn/917973.htm
qht.sthxr.cn/139333.htm
qht.sthxr.cn/753313.htm
qht.sthxr.cn/264623.htm
qht.sthxr.cn/711553.htm
qht.sthxr.cn/157113.htm
qht.sthxr.cn/337733.htm
qht.sthxr.cn/199573.htm
qht.sthxr.cn/519113.htm
qhr.sthxr.cn/600843.htm
qhr.sthxr.cn/531973.htm
qhr.sthxr.cn/311593.htm
qhr.sthxr.cn/315353.htm
qhr.sthxr.cn/535333.htm
qhr.sthxr.cn/882823.htm
qhr.sthxr.cn/175953.htm
qhr.sthxr.cn/517193.htm
qhr.sthxr.cn/931113.htm
qhr.sthxr.cn/155933.htm
qhe.sthxr.cn/195533.htm
qhe.sthxr.cn/157993.htm
qhe.sthxr.cn/513933.htm
qhe.sthxr.cn/319553.htm
qhe.sthxr.cn/951713.htm
qhe.sthxr.cn/062683.htm
qhe.sthxr.cn/739393.htm
qhe.sthxr.cn/115733.htm
qhe.sthxr.cn/337573.htm
qhe.sthxr.cn/975593.htm
qhw.sthxr.cn/979153.htm
qhw.sthxr.cn/113153.htm
qhw.sthxr.cn/755793.htm
qhw.sthxr.cn/795573.htm
qhw.sthxr.cn/911173.htm
qhw.sthxr.cn/317513.htm
qhw.sthxr.cn/193593.htm
qhw.sthxr.cn/739153.htm
qhw.sthxr.cn/599913.htm
qhw.sthxr.cn/515913.htm
qhq.sthxr.cn/731173.htm
qhq.sthxr.cn/711793.htm
qhq.sthxr.cn/860463.htm
qhq.sthxr.cn/131733.htm
qhq.sthxr.cn/131133.htm
qhq.sthxr.cn/955513.htm
qhq.sthxr.cn/199793.htm
qhq.sthxr.cn/555513.htm
qhq.sthxr.cn/719913.htm
qhq.sthxr.cn/735733.htm
qgm.sthxr.cn/533733.htm
qgm.sthxr.cn/371993.htm
qgm.sthxr.cn/579533.htm
qgm.sthxr.cn/202263.htm
qgm.sthxr.cn/571973.htm
qgm.sthxr.cn/719513.htm
qgm.sthxr.cn/993193.htm
qgm.sthxr.cn/377953.htm
qgm.sthxr.cn/571353.htm
qgm.sthxr.cn/555713.htm
qgn.sthxr.cn/315373.htm
qgn.sthxr.cn/997953.htm
qgn.sthxr.cn/593513.htm
qgn.sthxr.cn/402663.htm
qgn.sthxr.cn/133733.htm
qgn.sthxr.cn/959513.htm
qgn.sthxr.cn/179773.htm
qgn.sthxr.cn/539173.htm
qgn.sthxr.cn/317333.htm
qgn.sthxr.cn/779133.htm
qgb.sthxr.cn/799953.htm
qgb.sthxr.cn/357753.htm
qgb.sthxr.cn/888003.htm
qgb.sthxr.cn/197393.htm
qgb.sthxr.cn/393933.htm
qgb.sthxr.cn/371913.htm
qgb.sthxr.cn/711313.htm
qgb.sthxr.cn/515773.htm
qgb.sthxr.cn/604803.htm
qgb.sthxr.cn/573353.htm
qgv.sthxr.cn/331553.htm
qgv.sthxr.cn/977973.htm
qgv.sthxr.cn/375533.htm
qgv.sthxr.cn/373353.htm
qgv.sthxr.cn/953153.htm
qgv.sthxr.cn/371993.htm
qgv.sthxr.cn/339793.htm
qgv.sthxr.cn/933533.htm
qgv.sthxr.cn/799993.htm
qgv.sthxr.cn/862643.htm
qgc.sthxr.cn/317753.htm
qgc.sthxr.cn/795953.htm
qgc.sthxr.cn/315313.htm
qgc.sthxr.cn/719713.htm
qgc.sthxr.cn/597513.htm
qgc.sthxr.cn/802283.htm
qgc.sthxr.cn/359573.htm
qgc.sthxr.cn/953113.htm
qgc.sthxr.cn/111733.htm
qgc.sthxr.cn/917173.htm
qgx.sthxr.cn/840843.htm
qgx.sthxr.cn/171933.htm
qgx.sthxr.cn/735793.htm
qgx.sthxr.cn/715193.htm
qgx.sthxr.cn/555713.htm
qgx.sthxr.cn/628043.htm
qgx.sthxr.cn/373553.htm
qgx.sthxr.cn/919193.htm
qgx.sthxr.cn/537793.htm
qgx.sthxr.cn/199933.htm
qgz.sthxr.cn/957353.htm
qgz.sthxr.cn/846283.htm
qgz.sthxr.cn/199173.htm
qgz.sthxr.cn/937713.htm
qgz.sthxr.cn/117973.htm
qgz.sthxr.cn/533553.htm
qgz.sthxr.cn/931933.htm
qgz.sthxr.cn/555353.htm
qgz.sthxr.cn/353373.htm
qgz.sthxr.cn/599133.htm
qgl.sthxr.cn/535553.htm
qgl.sthxr.cn/662243.htm
qgl.sthxr.cn/280463.htm
qgl.sthxr.cn/93.htm
qgl.sthxr.cn/608223.htm
qgl.sthxr.cn/177733.htm
qgl.sthxr.cn/757793.htm
qgl.sthxr.cn/228623.htm
qgl.sthxr.cn/117553.htm
qgl.sthxr.cn/179513.htm
qgk.sthxr.cn/333913.htm
qgk.sthxr.cn/933533.htm
qgk.sthxr.cn/002883.htm
qgk.sthxr.cn/917913.htm
qgk.sthxr.cn/713313.htm
qgk.sthxr.cn/359133.htm
qgk.sthxr.cn/911713.htm
qgk.sthxr.cn/799793.htm
qgk.sthxr.cn/680003.htm
qgk.sthxr.cn/757753.htm
qgj.sthxr.cn/973533.htm
qgj.sthxr.cn/397933.htm
qgj.sthxr.cn/955993.htm
qgj.sthxr.cn/402203.htm
qgj.sthxr.cn/375533.htm
qgj.sthxr.cn/333913.htm
qgj.sthxr.cn/199373.htm
qgj.sthxr.cn/551593.htm
qgj.sthxr.cn/599313.htm
qgj.sthxr.cn/355313.htm
qgh.sthxr.cn/171513.htm
qgh.sthxr.cn/151533.htm
qgh.sthxr.cn/395993.htm
qgh.sthxr.cn/711153.htm
qgh.sthxr.cn/579993.htm
qgh.sthxr.cn/995953.htm
qgh.sthxr.cn/159553.htm
qgh.sthxr.cn/599113.htm
qgh.sthxr.cn/575153.htm
qgh.sthxr.cn/773773.htm
qgg.sthxr.cn/040483.htm
qgg.sthxr.cn/137173.htm
qgg.sthxr.cn/339533.htm
qgg.sthxr.cn/915933.htm
qgg.sthxr.cn/113333.htm
qgg.sthxr.cn/866863.htm
qgg.sthxr.cn/999153.htm
qgg.sthxr.cn/577713.htm
qgg.sthxr.cn/315753.htm
qgg.sthxr.cn/195573.htm
qgf.sthxr.cn/197153.htm
qgf.sthxr.cn/420443.htm
qgf.sthxr.cn/799353.htm
qgf.sthxr.cn/359153.htm
qgf.sthxr.cn/919553.htm
qgf.sthxr.cn/933133.htm
qgf.sthxr.cn/951773.htm
qgf.sthxr.cn/753993.htm
qgf.sthxr.cn/775993.htm
qgf.sthxr.cn/606663.htm
qgd.sthxr.cn/355573.htm
qgd.sthxr.cn/462423.htm
qgd.sthxr.cn/773393.htm
qgd.sthxr.cn/171993.htm
qgd.sthxr.cn/995193.htm
qgd.sthxr.cn/993393.htm
qgd.sthxr.cn/799993.htm
qgd.sthxr.cn/242823.htm
qgd.sthxr.cn/73.htm
qgd.sthxr.cn/919173.htm
qgs.sthxr.cn/131953.htm
qgs.sthxr.cn/313773.htm
qgs.sthxr.cn/757793.htm
qgs.sthxr.cn/799153.htm
qgs.sthxr.cn/331113.htm
qgs.sthxr.cn/517393.htm
qgs.sthxr.cn/197733.htm
qgs.sthxr.cn/060883.htm
qgs.sthxr.cn/202803.htm
qgs.sthxr.cn/717553.htm
qga.sthxr.cn/735773.htm
qga.sthxr.cn/551373.htm
qga.sthxr.cn/939333.htm
qga.sthxr.cn/357113.htm
qga.sthxr.cn/117373.htm
qga.sthxr.cn/719793.htm
qga.sthxr.cn/731193.htm
qga.sthxr.cn/971533.htm
qga.sthxr.cn/935513.htm
qga.sthxr.cn/151153.htm
qgp.sthxr.cn/111973.htm
qgp.sthxr.cn/195333.htm
qgp.sthxr.cn/597513.htm
qgp.sthxr.cn/595753.htm
qgp.sthxr.cn/191913.htm
qgp.sthxr.cn/195913.htm
qgp.sthxr.cn/351753.htm
qgp.sthxr.cn/959733.htm
qgp.sthxr.cn/339913.htm
qgp.sthxr.cn/426283.htm
qgo.sthxr.cn/793553.htm
qgo.sthxr.cn/595393.htm
qgo.sthxr.cn/135353.htm
qgo.sthxr.cn/111353.htm
qgo.sthxr.cn/595513.htm
qgo.sthxr.cn/935113.htm
qgo.sthxr.cn/337793.htm
qgo.sthxr.cn/139353.htm
qgo.sthxr.cn/797573.htm
qgo.sthxr.cn/931793.htm
qgi.sthxr.cn/800283.htm
qgi.sthxr.cn/939753.htm
qgi.sthxr.cn/373393.htm
qgi.sthxr.cn/862643.htm
qgi.sthxr.cn/999393.htm
qgi.sthxr.cn/662223.htm
qgi.sthxr.cn/115933.htm
qgi.sthxr.cn/371373.htm
qgi.sthxr.cn/593713.htm
qgi.sthxr.cn/422283.htm
qgu.sthxr.cn/599993.htm
qgu.sthxr.cn/193593.htm
qgu.sthxr.cn/995933.htm
qgu.sthxr.cn/193153.htm
qgu.sthxr.cn/173753.htm
qgu.sthxr.cn/315573.htm
qgu.sthxr.cn/779973.htm
qgu.sthxr.cn/793153.htm
qgu.sthxr.cn/977393.htm
qgu.sthxr.cn/135773.htm
qgy.sthxr.cn/397573.htm
qgy.sthxr.cn/715153.htm
qgy.sthxr.cn/973353.htm
qgy.sthxr.cn/933513.htm
qgy.sthxr.cn/713393.htm
qgy.sthxr.cn/862603.htm
qgy.sthxr.cn/997793.htm
qgy.sthxr.cn/773913.htm
qgy.sthxr.cn/137733.htm
qgy.sthxr.cn/311513.htm
qgt.sthxr.cn/793393.htm
qgt.sthxr.cn/155953.htm
qgt.sthxr.cn/771553.htm
qgt.sthxr.cn/151753.htm
qgt.sthxr.cn/171573.htm
qgt.sthxr.cn/735953.htm
qgt.sthxr.cn/553573.htm
qgt.sthxr.cn/151373.htm
qgt.sthxr.cn/357933.htm
qgt.sthxr.cn/515733.htm
qgr.sthxr.cn/155973.htm
qgr.sthxr.cn/028603.htm
qgr.sthxr.cn/864423.htm
qgr.sthxr.cn/808483.htm
qgr.sthxr.cn/535713.htm
qgr.sthxr.cn/979553.htm
qgr.sthxr.cn/317913.htm
qgr.sthxr.cn/711793.htm
qgr.sthxr.cn/591113.htm
qgr.sthxr.cn/199773.htm
qge.sthxr.cn/195993.htm
qge.sthxr.cn/793793.htm
qge.sthxr.cn/73.htm
qge.sthxr.cn/359553.htm
qge.sthxr.cn/004883.htm
qge.sthxr.cn/593353.htm
qge.sthxr.cn/555113.htm
qge.sthxr.cn/599373.htm
qge.sthxr.cn/593933.htm
qge.sthxr.cn/779733.htm
qgw.sthxr.cn/288683.htm
qgw.sthxr.cn/777973.htm
qgw.sthxr.cn/113773.htm
qgw.sthxr.cn/717593.htm
qgw.sthxr.cn/313133.htm
qgw.sthxr.cn/977333.htm
qgw.sthxr.cn/111973.htm
qgw.sthxr.cn/715393.htm
qgw.sthxr.cn/313553.htm
qgw.sthxr.cn/799393.htm
qgq.sthxr.cn/337973.htm
qgq.sthxr.cn/359753.htm
qgq.sthxr.cn/799573.htm
qgq.sthxr.cn/153533.htm
qgq.sthxr.cn/648063.htm
qgq.sthxr.cn/193173.htm
qgq.sthxr.cn/539913.htm
qgq.sthxr.cn/375933.htm
qgq.sthxr.cn/977113.htm
qgq.sthxr.cn/757173.htm
qfm.sthxr.cn/157953.htm
qfm.sthxr.cn/159593.htm
qfm.sthxr.cn/993533.htm
qfm.sthxr.cn/999993.htm
qfm.sthxr.cn/959773.htm
qfm.sthxr.cn/939173.htm
qfm.sthxr.cn/133573.htm
qfm.sthxr.cn/775193.htm
qfm.sthxr.cn/375533.htm
qfm.sthxr.cn/555733.htm
