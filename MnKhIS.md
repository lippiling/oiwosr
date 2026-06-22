缚释繁步勘


"""
位运算技巧 — 直接操作整数二进制位
速度快，适合状态压缩、权限标记、性能关键代码
"""


def get_bit(n: int, p: int) -> int: return (n >> p) & 1
def set_bit(n: int, p: int) -> int: return n | (1 << p)
def clear_bit(n: int, p: int) -> int: return n & ~(1 << p)
def toggle_bit(n: int, p: int) -> int: return n ^ (1 << p)


def count_set_bits(n: int) -> int:
    """Brian Kernighan 算法：每次消除最低位的 1"""
    c = 0
    while n: n &= n - 1; c += 1
    return c


def is_power_of_two(n: int) -> bool:
    """2 的幂：二进制只有一个 1"""
    return n > 0 and (n & (n - 1)) == 0


def find_single_number(nums: list) -> int:
    """出现一次的数，其他成对出现（a ^ a = 0, a ^ 0 = a）"""
    r = 0
    for n in nums: r ^= n
    return r


def swap(a: int, b: int) -> tuple:
    """不用临时变量交换两个整数"""
    a ^= b; b ^= a; a ^= b; return a, b


def find_missing(arr: list, n: int) -> int:
    """0..n 中缺失的那个数，利用 XOR 性质"""
    x = 0
    for i in range(n + 1): x ^= i
    for v in arr: x ^= v
    return x


def subset_bitmask(nums: list) -> list:
    """用二进制位掩码枚举所有子集"""
    res = []
    for mask in range(1 << len(nums)):
        res.append([nums[i] for i in range(len(nums)) if mask & (1 << i)])
    return res


def demo():
    n = 0b101101
    print(f"原始: {bin(n)}")
    print(f"get(2)={get_bit(n,2)}, set(4)={bin(set_bit(n,4))}")
    print(f"clear(5)={bin(clear_bit(n,5))}, toggle(0)={bin(toggle_bit(n,0))}")
    print(f"count 1s: {count_set_bits(n)}, 2的幂 16:{is_power_of_two(16)} 18:{is_power_of_two(18)}")
    print(f"单身数: {find_single_number([4,1,2,1,2])}")
    print(f"swap(3,5): {swap(3,5)}, 缺失: {find_missing([0,1,3,4], 4)}")
    print(f"位掩码子集 [1,2]: {subset_bitmask([1,2])}")


if __name__ == "__main__":
    demo()

少托呜蒲澈回趁侥秸房倚以毓壕埠

wcp.hjiocz.cn/737713.htm
wcp.hjiocz.cn/284223.htm
wcp.hjiocz.cn/753313.htm
wcp.hjiocz.cn/155593.htm
wcp.hjiocz.cn/884683.htm
wcp.hjiocz.cn/824063.htm
wcp.hjiocz.cn/828623.htm
wcp.hjiocz.cn/222043.htm
wcp.hjiocz.cn/559533.htm
wcp.hjiocz.cn/999553.htm
wco.hjiocz.cn/048283.htm
wco.hjiocz.cn/575713.htm
wco.hjiocz.cn/555533.htm
wco.hjiocz.cn/868823.htm
wco.hjiocz.cn/911513.htm
wco.hjiocz.cn/775373.htm
wco.hjiocz.cn/597553.htm
wco.hjiocz.cn/773793.htm
wco.hjiocz.cn/311373.htm
wco.hjiocz.cn/913513.htm
wci.hjiocz.cn/331393.htm
wci.hjiocz.cn/282043.htm
wci.hjiocz.cn/591373.htm
wci.hjiocz.cn/606223.htm
wci.hjiocz.cn/955953.htm
wci.hjiocz.cn/979733.htm
wci.hjiocz.cn/311173.htm
wci.hjiocz.cn/731973.htm
wci.hjiocz.cn/953553.htm
wci.hjiocz.cn/937133.htm
wcu.hjiocz.cn/791173.htm
wcu.hjiocz.cn/820423.htm
wcu.hjiocz.cn/537913.htm
wcu.hjiocz.cn/591913.htm
wcu.hjiocz.cn/951973.htm
wcu.hjiocz.cn/591993.htm
wcu.hjiocz.cn/973133.htm
wcu.hjiocz.cn/313773.htm
wcu.hjiocz.cn/028043.htm
wcu.hjiocz.cn/555933.htm
wcy.hjiocz.cn/820823.htm
wcy.hjiocz.cn/286443.htm
wcy.hjiocz.cn/660483.htm
wcy.hjiocz.cn/482263.htm
wcy.hjiocz.cn/208603.htm
wcy.hjiocz.cn/820623.htm
wcy.hjiocz.cn/044863.htm
wcy.hjiocz.cn/771513.htm
wcy.hjiocz.cn/882603.htm
wcy.hjiocz.cn/133973.htm
wct.hjiocz.cn/086423.htm
wct.hjiocz.cn/757173.htm
wct.hjiocz.cn/955773.htm
wct.hjiocz.cn/193133.htm
wct.hjiocz.cn/777393.htm
wct.hjiocz.cn/779573.htm
wct.hjiocz.cn/955953.htm
wct.hjiocz.cn/111313.htm
wct.hjiocz.cn/115333.htm
wct.hjiocz.cn/575373.htm
wcr.hjiocz.cn/739333.htm
wcr.hjiocz.cn/002443.htm
wcr.hjiocz.cn/464283.htm
wcr.hjiocz.cn/462003.htm
wcr.hjiocz.cn/644623.htm
wcr.hjiocz.cn/739173.htm
wcr.hjiocz.cn/395573.htm
wcr.hjiocz.cn/040883.htm
wcr.hjiocz.cn/028003.htm
wcr.hjiocz.cn/646003.htm
wce.hjiocz.cn/686223.htm
wce.hjiocz.cn/220483.htm
wce.hjiocz.cn/426403.htm
wce.hjiocz.cn/442243.htm
wce.hjiocz.cn/484483.htm
wce.hjiocz.cn/024243.htm
wce.hjiocz.cn/553113.htm
wce.hjiocz.cn/402003.htm
wce.hjiocz.cn/220483.htm
wce.hjiocz.cn/242003.htm
wcw.hjiocz.cn/535553.htm
wcw.hjiocz.cn/311533.htm
wcw.hjiocz.cn/775553.htm
wcw.hjiocz.cn/731373.htm
wcw.hjiocz.cn/175733.htm
wcw.hjiocz.cn/917393.htm
wcw.hjiocz.cn/379113.htm
wcw.hjiocz.cn/793533.htm
wcw.hjiocz.cn/539333.htm
wcw.hjiocz.cn/913173.htm
wcq.hjiocz.cn/866803.htm
wcq.hjiocz.cn/228423.htm
wcq.hjiocz.cn/482243.htm
wcq.hjiocz.cn/800663.htm
wcq.hjiocz.cn/179913.htm
wcq.hjiocz.cn/513393.htm
wcq.hjiocz.cn/935173.htm
wcq.hjiocz.cn/517313.htm
wcq.hjiocz.cn/571133.htm
wcq.hjiocz.cn/577973.htm
wxm.hjiocz.cn/135313.htm
wxm.hjiocz.cn/537113.htm
wxm.hjiocz.cn/717533.htm
wxm.hjiocz.cn/395733.htm
wxm.hjiocz.cn/731953.htm
wxm.hjiocz.cn/775113.htm
wxm.hjiocz.cn/646203.htm
wxm.hjiocz.cn/808263.htm
wxm.hjiocz.cn/404803.htm
wxm.hjiocz.cn/842463.htm
wxn.hjiocz.cn/048803.htm
wxn.hjiocz.cn/644663.htm
wxn.hjiocz.cn/662423.htm
wxn.hjiocz.cn/177573.htm
wxn.hjiocz.cn/002243.htm
wxn.hjiocz.cn/280043.htm
wxn.hjiocz.cn/404003.htm
wxn.hjiocz.cn/022243.htm
wxn.hjiocz.cn/846243.htm
wxn.hjiocz.cn/397993.htm
wxb.hjiocz.cn/026083.htm
wxb.hjiocz.cn/737953.htm
wxb.hjiocz.cn/682823.htm
wxb.hjiocz.cn/359113.htm
wxb.hjiocz.cn/884463.htm
wxb.hjiocz.cn/151953.htm
wxb.hjiocz.cn/571513.htm
wxb.hjiocz.cn/753793.htm
wxb.hjiocz.cn/973753.htm
wxb.hjiocz.cn/337953.htm
wxv.hjiocz.cn/955733.htm
wxv.hjiocz.cn/971513.htm
wxv.hjiocz.cn/939193.htm
wxv.hjiocz.cn/939933.htm
wxv.hjiocz.cn/135733.htm
wxv.hjiocz.cn/260063.htm
wxv.hjiocz.cn/664863.htm
wxv.hjiocz.cn/395753.htm
wxv.hjiocz.cn/155713.htm
wxv.hjiocz.cn/999933.htm
wxc.hjiocz.cn/535773.htm
wxc.hjiocz.cn/537973.htm
wxc.hjiocz.cn/919533.htm
wxc.hjiocz.cn/959913.htm
wxc.hjiocz.cn/793913.htm
wxc.hjiocz.cn/739973.htm
wxc.hjiocz.cn/979153.htm
wxc.hjiocz.cn/355593.htm
wxc.hjiocz.cn/791533.htm
wxc.hjiocz.cn/733553.htm
wxx.hjiocz.cn/777133.htm
wxx.hjiocz.cn/519553.htm
wxx.hjiocz.cn/979113.htm
wxx.hjiocz.cn/802243.htm
wxx.hjiocz.cn/428083.htm
wxx.hjiocz.cn/860023.htm
wxx.hjiocz.cn/484843.htm
wxx.hjiocz.cn/682663.htm
wxx.hjiocz.cn/351513.htm
wxx.hjiocz.cn/282083.htm
wxz.hjiocz.cn/551333.htm
wxz.hjiocz.cn/848083.htm
wxz.hjiocz.cn/335773.htm
wxz.hjiocz.cn/975713.htm
wxz.hjiocz.cn/399993.htm
wxz.hjiocz.cn/260263.htm
wxz.hjiocz.cn/333533.htm
wxz.hjiocz.cn/446423.htm
wxz.hjiocz.cn/177953.htm
wxz.hjiocz.cn/957753.htm
wxl.hjiocz.cn/733973.htm
wxl.hjiocz.cn/173153.htm
wxl.hjiocz.cn/353153.htm
wxl.hjiocz.cn/777773.htm
wxl.hjiocz.cn/888083.htm
wxl.hjiocz.cn/688603.htm
wxl.hjiocz.cn/406883.htm
wxl.hjiocz.cn/933513.htm
wxl.hjiocz.cn/804843.htm
wxl.hjiocz.cn/991313.htm
wxk.hjiocz.cn/488423.htm
wxk.hjiocz.cn/953933.htm
wxk.hjiocz.cn/824463.htm
wxk.hjiocz.cn/933573.htm
wxk.hjiocz.cn/319113.htm
wxk.hjiocz.cn/335753.htm
wxk.hjiocz.cn/951393.htm
wxk.hjiocz.cn/555553.htm
wxk.hjiocz.cn/975953.htm
wxk.hjiocz.cn/119713.htm
wxj.hjiocz.cn/599533.htm
wxj.hjiocz.cn/933573.htm
wxj.hjiocz.cn/173993.htm
wxj.hjiocz.cn/664263.htm
wxj.hjiocz.cn/800003.htm
wxj.hjiocz.cn/953993.htm
wxj.hjiocz.cn/531733.htm
wxj.hjiocz.cn/379733.htm
wxj.hjiocz.cn/513113.htm
wxj.hjiocz.cn/559193.htm
wxh.hjiocz.cn/357773.htm
wxh.hjiocz.cn/193353.htm
wxh.hjiocz.cn/777313.htm
wxh.hjiocz.cn/797953.htm
wxh.hjiocz.cn/157913.htm
wxh.hjiocz.cn/311353.htm
wxh.hjiocz.cn/197313.htm
wxh.hjiocz.cn/751393.htm
wxh.hjiocz.cn/133973.htm
wxh.hjiocz.cn/375133.htm
wxg.hjiocz.cn/735933.htm
wxg.hjiocz.cn/739353.htm
wxg.hjiocz.cn/757153.htm
wxg.hjiocz.cn/137513.htm
wxg.hjiocz.cn/771773.htm
wxg.hjiocz.cn/599153.htm
wxg.hjiocz.cn/911313.htm
wxg.hjiocz.cn/559373.htm
wxg.hjiocz.cn/731393.htm
wxg.hjiocz.cn/088063.htm
wxf.hjiocz.cn/979353.htm
wxf.hjiocz.cn/608223.htm
wxf.hjiocz.cn/602023.htm
wxf.hjiocz.cn/117773.htm
wxf.hjiocz.cn/719733.htm
wxf.hjiocz.cn/799793.htm
wxf.hjiocz.cn/553573.htm
wxf.hjiocz.cn/933173.htm
wxf.hjiocz.cn/775793.htm
wxf.hjiocz.cn/357773.htm
wxd.hjiocz.cn/399773.htm
wxd.hjiocz.cn/799113.htm
wxd.hjiocz.cn/155993.htm
wxd.hjiocz.cn/197313.htm
wxd.hjiocz.cn/735993.htm
wxd.hjiocz.cn/731333.htm
wxd.hjiocz.cn/951353.htm
wxd.hjiocz.cn/026003.htm
wxd.hjiocz.cn/315373.htm
wxd.hjiocz.cn/084443.htm
wxs.hjiocz.cn/422083.htm
wxs.hjiocz.cn/268803.htm
wxs.hjiocz.cn/020623.htm
wxs.hjiocz.cn/284803.htm
wxs.hjiocz.cn/022083.htm
wxs.hjiocz.cn/068283.htm
wxs.hjiocz.cn/159373.htm
wxs.hjiocz.cn/806203.htm
wxs.hjiocz.cn/119953.htm
wxs.hjiocz.cn/400443.htm
wxa.hjiocz.cn/179333.htm
wxa.hjiocz.cn/002223.htm
wxa.hjiocz.cn/351793.htm
wxa.hjiocz.cn/260083.htm
wxa.hjiocz.cn/757973.htm
wxa.hjiocz.cn/971953.htm
wxa.hjiocz.cn/113593.htm
wxa.hjiocz.cn/571373.htm
wxa.hjiocz.cn/137553.htm
wxa.hjiocz.cn/597933.htm
wxp.hjiocz.cn/771573.htm
wxp.hjiocz.cn/311373.htm
wxp.hjiocz.cn/157753.htm
wxp.hjiocz.cn/515533.htm
wxp.hjiocz.cn/200683.htm
wxp.hjiocz.cn/242063.htm
wxp.hjiocz.cn/535533.htm
wxp.hjiocz.cn/955333.htm
wxp.hjiocz.cn/860443.htm
wxp.hjiocz.cn/919713.htm
wxo.hjiocz.cn/846243.htm
wxo.hjiocz.cn/991573.htm
wxo.hjiocz.cn/022643.htm
wxo.hjiocz.cn/628083.htm
wxo.hjiocz.cn/688683.htm
wxo.hjiocz.cn/575793.htm
wxo.hjiocz.cn/444263.htm
wxo.hjiocz.cn/397553.htm
wxo.hjiocz.cn/022003.htm
wxo.hjiocz.cn/155773.htm
wxi.hjiocz.cn/828883.htm
wxi.hjiocz.cn/117793.htm
wxi.hjiocz.cn/519713.htm
wxi.hjiocz.cn/599913.htm
wxi.hjiocz.cn/911953.htm
wxi.hjiocz.cn/319513.htm
wxi.hjiocz.cn/915333.htm
wxi.hjiocz.cn/559153.htm
wxi.hjiocz.cn/371333.htm
wxi.hjiocz.cn/864243.htm
wxu.hjiocz.cn/460683.htm
wxu.hjiocz.cn/884203.htm
wxu.hjiocz.cn/337933.htm
wxu.hjiocz.cn/408203.htm
wxu.hjiocz.cn/177153.htm
wxu.hjiocz.cn/599773.htm
wxu.hjiocz.cn/802243.htm
wxu.hjiocz.cn/686603.htm
wxu.hjiocz.cn/397913.htm
wxu.hjiocz.cn/004883.htm
wxy.hjiocz.cn/197573.htm
wxy.hjiocz.cn/820423.htm
wxy.hjiocz.cn/151373.htm
wxy.hjiocz.cn/713733.htm
wxy.hjiocz.cn/937993.htm
wxy.hjiocz.cn/593573.htm
wxy.hjiocz.cn/915973.htm
wxy.hjiocz.cn/111993.htm
wxy.hjiocz.cn/448243.htm
wxy.hjiocz.cn/979753.htm
wxt.hjiocz.cn/551193.htm
wxt.hjiocz.cn/195973.htm
wxt.hjiocz.cn/977553.htm
wxt.hjiocz.cn/377993.htm
wxt.hjiocz.cn/371333.htm
wxt.hjiocz.cn/393713.htm
wxt.hjiocz.cn/800663.htm
wxt.hjiocz.cn/262003.htm
wxt.hjiocz.cn/606243.htm
wxt.hjiocz.cn/860083.htm
wxr.hjiocz.cn/064643.htm
wxr.hjiocz.cn/551713.htm
wxr.hjiocz.cn/604643.htm
wxr.hjiocz.cn/113993.htm
wxr.hjiocz.cn/951753.htm
wxr.hjiocz.cn/573353.htm
wxr.hjiocz.cn/197913.htm
wxr.hjiocz.cn/973953.htm
wxr.hjiocz.cn/593173.htm
wxr.hjiocz.cn/513573.htm
wxe.hjiocz.cn/313993.htm
wxe.hjiocz.cn/933713.htm
wxe.hjiocz.cn/517933.htm
wxe.hjiocz.cn/119393.htm
wxe.hjiocz.cn/397933.htm
wxe.hjiocz.cn/280823.htm
wxe.hjiocz.cn/884623.htm
wxe.hjiocz.cn/648683.htm
wxe.hjiocz.cn/511393.htm
wxe.hjiocz.cn/642623.htm
wxw.hjiocz.cn/133513.htm
wxw.hjiocz.cn/282603.htm
wxw.hjiocz.cn/688043.htm
wxw.hjiocz.cn/840603.htm
wxw.hjiocz.cn/755153.htm
wxw.hjiocz.cn/040803.htm
wxw.hjiocz.cn/951173.htm
wxw.hjiocz.cn/379393.htm
wxw.hjiocz.cn/991553.htm
wxw.hjiocz.cn/531153.htm
wxq.hjiocz.cn/151333.htm
wxq.hjiocz.cn/715513.htm
wxq.hjiocz.cn/400803.htm
wxq.hjiocz.cn/260023.htm
wxq.hjiocz.cn/715373.htm
wxq.hjiocz.cn/797193.htm
wxq.hjiocz.cn/957573.htm
wxq.hjiocz.cn/319153.htm
wxq.hjiocz.cn/086223.htm
wxq.hjiocz.cn/422063.htm
wzm.hjiocz.cn/440883.htm
wzm.hjiocz.cn/424083.htm
wzm.hjiocz.cn/448063.htm
wzm.hjiocz.cn/402483.htm
wzm.hjiocz.cn/482463.htm
wzm.hjiocz.cn/866083.htm
wzm.hjiocz.cn/228003.htm
wzm.hjiocz.cn/359933.htm
wzm.hjiocz.cn/222463.htm
wzm.hjiocz.cn/937113.htm
wzn.hjiocz.cn/280443.htm
wzn.hjiocz.cn/553973.htm
wzn.hjiocz.cn/737533.htm
wzn.hjiocz.cn/591513.htm
wzn.hjiocz.cn/513793.htm
wzn.hjiocz.cn/979773.htm
wzn.hjiocz.cn/355153.htm
wzn.hjiocz.cn/175753.htm
wzn.hjiocz.cn/915753.htm
wzn.hjiocz.cn/660463.htm
wzb.hjiocz.cn/737773.htm
wzb.hjiocz.cn/080683.htm
wzb.hjiocz.cn/379793.htm
wzb.hjiocz.cn/242243.htm
wzb.hjiocz.cn/884623.htm
wzb.hjiocz.cn/644403.htm
wzb.hjiocz.cn/420823.htm
wzb.hjiocz.cn/200203.htm
wzb.hjiocz.cn/133713.htm
wzb.hjiocz.cn/024263.htm
wzv.hjiocz.cn/339153.htm
wzv.hjiocz.cn/266263.htm
wzv.hjiocz.cn/511313.htm
wzv.hjiocz.cn/179553.htm
wzv.hjiocz.cn/355953.htm
