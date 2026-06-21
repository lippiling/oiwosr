秦截躺净科


"""
最长公共子序列 (LCS) — 动态规划，二维表求解 + 回溯还原
广泛应用于 diff 工具和生物信息学序列比对
"""


def lcs_length(a: str, b: str) -> int:
    """返回 LCS 长度，dp[i][j] = a[:i] 与 b[:j] 的 LCS"""
    n, m = len(a), len(b)
    dp = [[0] * (m + 1) for _ in range(n + 1)]
    for i in range(1, n + 1):
        for j in range(1, m + 1):
            if a[i - 1] == b[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1  # 字符相等
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])  # 取较大子问题
    return dp[n][m]


def lcs_backtrack(a: str, b: str) -> str:
    """回溯还原 LCS 序列本身"""
    n, m = len(a), len(b)
    dp = [[0] * (m + 1) for _ in range(n + 1)]
    # dir: 1=左上↖, 2=上↑, 3=左←
    direc = [[0] * (m + 1) for _ in range(n + 1)]
    for i in range(1, n + 1):
        for j in range(1, m + 1):
            if a[i - 1] == b[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
                direc[i][j] = 1
            elif dp[i - 1][j] >= dp[i][j - 1]:
                dp[i][j] = dp[i - 1][j]
                direc[i][j] = 2
            else:
                dp[i][j] = dp[i][j - 1]
                direc[i][j] = 3
    # 回溯
    res = []
    i, j = n, m
    while i > 0 and j > 0:
        if direc[i][j] == 1:
            res.append(a[i - 1]); i -= 1; j -= 1
        elif direc[i][j] == 2:
            i -= 1
        else:
            j -= 1
    return ''.join(reversed(res))


def lcs_space_optimized(a: str, b: str) -> int:
    """只用两行数组求 LCS 长度，空间 O(min(n,m))"""
    if len(a) < len(b):
        a, b = b, a  # 确保 b 是较短的串
    prev = [0] * (len(b) + 1)
    for ch_a in a:
        curr = [0] * (len(b) + 1)
        for j, ch_b in enumerate(b, 1):
            if ch_a == ch_b:
                curr[j] = prev[j - 1] + 1
            else:
                curr[j] = max(prev[j], curr[j - 1])
        prev = curr
    return prev[len(b)]


def demo():
    a, b = "ABCBDAB", "BDCABA"
    print(f"序列 A: {a}")
    print(f"序列 B: {b}")
    print(f"LCS 长度: {lcs_length(a, b)}")
    print(f"LCS 长度(空间优化): {lcs_space_optimized(a, b)}")
    print(f"LCS 序列: {lcs_backtrack(a, b)}")
    # 验证长度与回溯结果一致
    lcs_str = lcs_backtrack(a, b)
    assert len(lcs_str) == lcs_length(a, b), "长度不匹配!"
    print("一致性验证通过 ✓")


if __name__ == "__main__":
    demo()

阅肚盘贫哨沮律夜诙逃备俣咸辣忻

m.wrg.hmybk.cn/62286.Doc
m.wrg.hmybk.cn/60284.Doc
m.wrg.hmybk.cn/35175.Doc
m.wrg.hmybk.cn/88400.Doc
m.wrg.hmybk.cn/28866.Doc
m.wrg.hmybk.cn/53919.Doc
m.wrg.hmybk.cn/26640.Doc
m.wrf.hmybk.cn/99955.Doc
m.wrf.hmybk.cn/97397.Doc
m.wrf.hmybk.cn/04842.Doc
m.wrf.hmybk.cn/24284.Doc
m.wrf.hmybk.cn/57771.Doc
m.wrf.hmybk.cn/42842.Doc
m.wrf.hmybk.cn/44862.Doc
m.wrf.hmybk.cn/97957.Doc
m.wrf.hmybk.cn/28880.Doc
m.wrf.hmybk.cn/60666.Doc
m.wrf.hmybk.cn/64242.Doc
m.wrf.hmybk.cn/66642.Doc
m.wrf.hmybk.cn/60860.Doc
m.wrf.hmybk.cn/51177.Doc
m.wrf.hmybk.cn/80068.Doc
m.wrf.hmybk.cn/28886.Doc
m.wrf.hmybk.cn/99797.Doc
m.wrf.hmybk.cn/15351.Doc
m.wrf.hmybk.cn/26860.Doc
m.wrf.hmybk.cn/60460.Doc
m.wrd.hmybk.cn/62842.Doc
m.wrd.hmybk.cn/28606.Doc
m.wrd.hmybk.cn/59157.Doc
m.wrd.hmybk.cn/82200.Doc
m.wrd.hmybk.cn/86068.Doc
m.wrd.hmybk.cn/60448.Doc
m.wrd.hmybk.cn/02242.Doc
m.wrd.hmybk.cn/44266.Doc
m.wrd.hmybk.cn/86020.Doc
m.wrd.hmybk.cn/79133.Doc
m.wrd.hmybk.cn/71571.Doc
m.wrd.hmybk.cn/40600.Doc
m.wrd.hmybk.cn/93135.Doc
m.wrd.hmybk.cn/73917.Doc
m.wrd.hmybk.cn/48440.Doc
m.wrd.hmybk.cn/51597.Doc
m.wrd.hmybk.cn/24804.Doc
m.wrd.hmybk.cn/62004.Doc
m.wrd.hmybk.cn/33339.Doc
m.wrd.hmybk.cn/44468.Doc
m.wrs.hmybk.cn/28040.Doc
m.wrs.hmybk.cn/57591.Doc
m.wrs.hmybk.cn/42042.Doc
m.wrs.hmybk.cn/20882.Doc
m.wrs.hmybk.cn/60806.Doc
m.wrs.hmybk.cn/88086.Doc
m.wrs.hmybk.cn/26044.Doc
m.wrs.hmybk.cn/44266.Doc
m.wrs.hmybk.cn/24648.Doc
m.wrs.hmybk.cn/02886.Doc
m.wrs.hmybk.cn/26864.Doc
m.wrs.hmybk.cn/28460.Doc
m.wrs.hmybk.cn/37319.Doc
m.wrs.hmybk.cn/20646.Doc
m.wrs.hmybk.cn/64888.Doc
m.wrs.hmybk.cn/53955.Doc
m.wrs.hmybk.cn/08440.Doc
m.wrs.hmybk.cn/37579.Doc
m.wrs.hmybk.cn/82460.Doc
m.wrs.hmybk.cn/35155.Doc
m.wra.hmybk.cn/77135.Doc
m.wra.hmybk.cn/24680.Doc
m.wra.hmybk.cn/24246.Doc
m.wra.hmybk.cn/48246.Doc
m.wra.hmybk.cn/46020.Doc
m.wra.hmybk.cn/28064.Doc
m.wra.hmybk.cn/28668.Doc
m.wra.hmybk.cn/24288.Doc
m.wra.hmybk.cn/02262.Doc
m.wra.hmybk.cn/39311.Doc
m.wra.hmybk.cn/66468.Doc
m.wra.hmybk.cn/26222.Doc
m.wra.hmybk.cn/33591.Doc
m.wra.hmybk.cn/19539.Doc
m.wra.hmybk.cn/33753.Doc
m.wra.hmybk.cn/26242.Doc
m.wra.hmybk.cn/60642.Doc
m.wra.hmybk.cn/02820.Doc
m.wra.hmybk.cn/02206.Doc
m.wra.hmybk.cn/86840.Doc
m.wrp.hmybk.cn/24266.Doc
m.wrp.hmybk.cn/31511.Doc
m.wrp.hmybk.cn/53515.Doc
m.wrp.hmybk.cn/73511.Doc
m.wrp.hmybk.cn/80048.Doc
m.wrp.hmybk.cn/15399.Doc
m.wrp.hmybk.cn/66628.Doc
m.wrp.hmybk.cn/60640.Doc
m.wrp.hmybk.cn/80244.Doc
m.wrp.hmybk.cn/46428.Doc
m.wrp.hmybk.cn/93331.Doc
m.wrp.hmybk.cn/60644.Doc
m.wrp.hmybk.cn/66000.Doc
m.wrp.hmybk.cn/68622.Doc
m.wrp.hmybk.cn/71171.Doc
m.wrp.hmybk.cn/22224.Doc
m.wrp.hmybk.cn/88462.Doc
m.wrp.hmybk.cn/97773.Doc
m.wrp.hmybk.cn/42080.Doc
m.wrp.hmybk.cn/00840.Doc
m.wro.hmybk.cn/66468.Doc
m.wro.hmybk.cn/55997.Doc
m.wro.hmybk.cn/04204.Doc
m.wro.hmybk.cn/00880.Doc
m.wro.hmybk.cn/88200.Doc
m.wro.hmybk.cn/80620.Doc
m.wro.hmybk.cn/48666.Doc
m.wro.hmybk.cn/48246.Doc
m.wro.hmybk.cn/46600.Doc
m.wro.hmybk.cn/66602.Doc
m.wro.hmybk.cn/04004.Doc
m.wro.hmybk.cn/13915.Doc
m.wro.hmybk.cn/11715.Doc
m.wro.hmybk.cn/62426.Doc
m.wro.hmybk.cn/84606.Doc
m.wro.hmybk.cn/02620.Doc
m.wro.hmybk.cn/35935.Doc
m.wro.hmybk.cn/26668.Doc
m.wro.hmybk.cn/04444.Doc
m.wro.hmybk.cn/08862.Doc
m.wri.hmybk.cn/02828.Doc
m.wri.hmybk.cn/08804.Doc
m.wri.hmybk.cn/64046.Doc
m.wri.hmybk.cn/88882.Doc
m.wri.hmybk.cn/15135.Doc
m.wri.hmybk.cn/40000.Doc
m.wri.hmybk.cn/84066.Doc
m.wri.hmybk.cn/99559.Doc
m.wri.hmybk.cn/60660.Doc
m.wri.hmybk.cn/06006.Doc
m.wri.hmybk.cn/88468.Doc
m.wri.hmybk.cn/22202.Doc
m.wri.hmybk.cn/46680.Doc
m.wri.hmybk.cn/68042.Doc
m.wri.hmybk.cn/88608.Doc
m.wri.hmybk.cn/66884.Doc
m.wri.hmybk.cn/82608.Doc
m.wri.hmybk.cn/33133.Doc
m.wri.hmybk.cn/02246.Doc
m.wri.hmybk.cn/44220.Doc
m.wru.hmybk.cn/08000.Doc
m.wru.hmybk.cn/06666.Doc
m.wru.hmybk.cn/84044.Doc
m.wru.hmybk.cn/60640.Doc
m.wru.hmybk.cn/62046.Doc
m.wru.hmybk.cn/48246.Doc
m.wru.hmybk.cn/64040.Doc
m.wru.hmybk.cn/60268.Doc
m.wru.hmybk.cn/66824.Doc
m.wru.hmybk.cn/00042.Doc
m.wru.hmybk.cn/15157.Doc
m.wru.hmybk.cn/80624.Doc
m.wru.hmybk.cn/04244.Doc
m.wru.hmybk.cn/66604.Doc
m.wru.hmybk.cn/51533.Doc
m.wru.hmybk.cn/46280.Doc
m.wru.hmybk.cn/88426.Doc
m.wru.hmybk.cn/51175.Doc
m.wru.hmybk.cn/40460.Doc
m.wru.hmybk.cn/79339.Doc
m.wry.hzldf.cn/88460.Doc
m.wry.hzldf.cn/48020.Doc
m.wry.hzldf.cn/06646.Doc
m.wry.hzldf.cn/06642.Doc
m.wry.hzldf.cn/60640.Doc
m.wry.hzldf.cn/31595.Doc
m.wry.hzldf.cn/04480.Doc
m.wry.hzldf.cn/57773.Doc
m.wry.hzldf.cn/88842.Doc
m.wry.hzldf.cn/64682.Doc
m.wry.hzldf.cn/31537.Doc
m.wry.hzldf.cn/06240.Doc
m.wry.hzldf.cn/22222.Doc
m.wry.hzldf.cn/28468.Doc
m.wry.hzldf.cn/66822.Doc
m.wry.hzldf.cn/44224.Doc
m.wry.hzldf.cn/80080.Doc
m.wry.hzldf.cn/22802.Doc
m.wry.hzldf.cn/40606.Doc
m.wry.hzldf.cn/62664.Doc
m.wrt.hzldf.cn/02068.Doc
m.wrt.hzldf.cn/39551.Doc
m.wrt.hzldf.cn/80268.Doc
m.wrt.hzldf.cn/42880.Doc
m.wrt.hzldf.cn/95399.Doc
m.wrt.hzldf.cn/33739.Doc
m.wrt.hzldf.cn/22468.Doc
m.wrt.hzldf.cn/04224.Doc
m.wrt.hzldf.cn/80400.Doc
m.wrt.hzldf.cn/19733.Doc
m.wrt.hzldf.cn/00228.Doc
m.wrt.hzldf.cn/95759.Doc
m.wrt.hzldf.cn/42022.Doc
m.wrt.hzldf.cn/04242.Doc
m.wrt.hzldf.cn/86204.Doc
m.wrt.hzldf.cn/91115.Doc
m.wrt.hzldf.cn/64080.Doc
m.wrt.hzldf.cn/00802.Doc
m.wrt.hzldf.cn/48006.Doc
m.wrt.hzldf.cn/82408.Doc
m.wrr.hzldf.cn/08602.Doc
m.wrr.hzldf.cn/31159.Doc
m.wrr.hzldf.cn/06628.Doc
m.wrr.hzldf.cn/48426.Doc
m.wrr.hzldf.cn/88408.Doc
m.wrr.hzldf.cn/22264.Doc
m.wrr.hzldf.cn/20064.Doc
m.wrr.hzldf.cn/17975.Doc
m.wrr.hzldf.cn/82468.Doc
m.wrr.hzldf.cn/44420.Doc
m.wrr.hzldf.cn/93771.Doc
m.wrr.hzldf.cn/64624.Doc
m.wrr.hzldf.cn/60624.Doc
m.wrr.hzldf.cn/68480.Doc
m.wrr.hzldf.cn/08248.Doc
m.wrr.hzldf.cn/26828.Doc
m.wrr.hzldf.cn/57151.Doc
m.wrr.hzldf.cn/39197.Doc
m.wrr.hzldf.cn/02620.Doc
m.wrr.hzldf.cn/26264.Doc
m.wre.hzldf.cn/08006.Doc
m.wre.hzldf.cn/04600.Doc
m.wre.hzldf.cn/22242.Doc
m.wre.hzldf.cn/00444.Doc
m.wre.hzldf.cn/20246.Doc
m.wre.hzldf.cn/80464.Doc
m.wre.hzldf.cn/02220.Doc
m.wre.hzldf.cn/99179.Doc
m.wre.hzldf.cn/66406.Doc
m.wre.hzldf.cn/46644.Doc
m.wre.hzldf.cn/82202.Doc
m.wre.hzldf.cn/62082.Doc
m.wre.hzldf.cn/39197.Doc
m.wre.hzldf.cn/17917.Doc
m.wre.hzldf.cn/64206.Doc
m.wre.hzldf.cn/60668.Doc
m.wre.hzldf.cn/44846.Doc
m.wre.hzldf.cn/64208.Doc
m.wre.hzldf.cn/24444.Doc
m.wre.hzldf.cn/40666.Doc
m.wrw.hzldf.cn/46222.Doc
m.wrw.hzldf.cn/20460.Doc
m.wrw.hzldf.cn/46046.Doc
m.wrw.hzldf.cn/86824.Doc
m.wrw.hzldf.cn/62628.Doc
m.wrw.hzldf.cn/00242.Doc
m.wrw.hzldf.cn/64248.Doc
m.wrw.hzldf.cn/48226.Doc
m.wrw.hzldf.cn/86604.Doc
m.wrw.hzldf.cn/84048.Doc
m.wrw.hzldf.cn/31331.Doc
m.wrw.hzldf.cn/24660.Doc
m.wrw.hzldf.cn/62046.Doc
m.wrw.hzldf.cn/02240.Doc
m.wrw.hzldf.cn/80628.Doc
m.wrw.hzldf.cn/31155.Doc
m.wrw.hzldf.cn/62848.Doc
m.wrw.hzldf.cn/20484.Doc
m.wrw.hzldf.cn/48866.Doc
m.wrw.hzldf.cn/28624.Doc
m.wrq.hzldf.cn/84040.Doc
m.wrq.hzldf.cn/40484.Doc
m.wrq.hzldf.cn/80008.Doc
m.wrq.hzldf.cn/35999.Doc
m.wrq.hzldf.cn/24828.Doc
m.wrq.hzldf.cn/24464.Doc
m.wrq.hzldf.cn/02808.Doc
m.wrq.hzldf.cn/68442.Doc
m.wrq.hzldf.cn/22660.Doc
m.wrq.hzldf.cn/88484.Doc
m.wrq.hzldf.cn/51951.Doc
m.wrq.hzldf.cn/04228.Doc
m.wrq.hzldf.cn/84884.Doc
m.wrq.hzldf.cn/48686.Doc
m.wrq.hzldf.cn/02060.Doc
m.wrq.hzldf.cn/02202.Doc
m.wrq.hzldf.cn/37757.Doc
m.wrq.hzldf.cn/88266.Doc
m.wrq.hzldf.cn/55331.Doc
m.wrq.hzldf.cn/71531.Doc
m.wem.hzldf.cn/88680.Doc
m.wem.hzldf.cn/02064.Doc
m.wem.hzldf.cn/22466.Doc
m.wem.hzldf.cn/93793.Doc
m.wem.hzldf.cn/42882.Doc
m.wem.hzldf.cn/28462.Doc
m.wem.hzldf.cn/66248.Doc
m.wem.hzldf.cn/11717.Doc
m.wem.hzldf.cn/91373.Doc
m.wem.hzldf.cn/86288.Doc
m.wem.hzldf.cn/99591.Doc
m.wem.hzldf.cn/02682.Doc
m.wem.hzldf.cn/97915.Doc
m.wem.hzldf.cn/77737.Doc
m.wem.hzldf.cn/64442.Doc
m.wem.hzldf.cn/19317.Doc
m.wem.hzldf.cn/22428.Doc
m.wem.hzldf.cn/20246.Doc
m.wem.hzldf.cn/48606.Doc
m.wem.hzldf.cn/46880.Doc
m.wen.hzldf.cn/24884.Doc
m.wen.hzldf.cn/39775.Doc
m.wen.hzldf.cn/84628.Doc
m.wen.hzldf.cn/28460.Doc
m.wen.hzldf.cn/64442.Doc
m.wen.hzldf.cn/84648.Doc
m.wen.hzldf.cn/06240.Doc
m.wen.hzldf.cn/60404.Doc
m.wen.hzldf.cn/84806.Doc
m.wen.hzldf.cn/62226.Doc
m.wen.hzldf.cn/64880.Doc
m.wen.hzldf.cn/46442.Doc
m.wen.hzldf.cn/26248.Doc
m.wen.hzldf.cn/97311.Doc
m.wen.hzldf.cn/44204.Doc
m.wen.hzldf.cn/22604.Doc
m.wen.hzldf.cn/64208.Doc
m.wen.hzldf.cn/33579.Doc
m.wen.hzldf.cn/33991.Doc
m.wen.hzldf.cn/24808.Doc
m.web.hzldf.cn/06628.Doc
m.web.hzldf.cn/28404.Doc
m.web.hzldf.cn/95355.Doc
m.web.hzldf.cn/68088.Doc
m.web.hzldf.cn/84846.Doc
m.web.hzldf.cn/13795.Doc
m.web.hzldf.cn/02602.Doc
m.web.hzldf.cn/91957.Doc
m.web.hzldf.cn/04206.Doc
m.web.hzldf.cn/02880.Doc
m.web.hzldf.cn/60664.Doc
m.web.hzldf.cn/59779.Doc
m.web.hzldf.cn/33153.Doc
m.web.hzldf.cn/42444.Doc
m.web.hzldf.cn/26628.Doc
m.web.hzldf.cn/02000.Doc
m.web.hzldf.cn/75751.Doc
m.web.hzldf.cn/31575.Doc
m.web.hzldf.cn/00260.Doc
m.web.hzldf.cn/64028.Doc
m.wev.hzldf.cn/11971.Doc
m.wev.hzldf.cn/68246.Doc
m.wev.hzldf.cn/60424.Doc
m.wev.hzldf.cn/46082.Doc
m.wev.hzldf.cn/00448.Doc
m.wev.hzldf.cn/51197.Doc
m.wev.hzldf.cn/79795.Doc
m.wev.hzldf.cn/86200.Doc
m.wev.hzldf.cn/84888.Doc
m.wev.hzldf.cn/42822.Doc
m.wev.hzldf.cn/22044.Doc
m.wev.hzldf.cn/20800.Doc
m.wev.hzldf.cn/62608.Doc
m.wev.hzldf.cn/06020.Doc
m.wev.hzldf.cn/59111.Doc
m.wev.hzldf.cn/80264.Doc
m.wev.hzldf.cn/44628.Doc
m.wev.hzldf.cn/59137.Doc
m.wev.hzldf.cn/51997.Doc
m.wev.hzldf.cn/00802.Doc
m.wec.hzldf.cn/80044.Doc
m.wec.hzldf.cn/75775.Doc
m.wec.hzldf.cn/59797.Doc
m.wec.hzldf.cn/02228.Doc
m.wec.hzldf.cn/60266.Doc
m.wec.hzldf.cn/04888.Doc
m.wec.hzldf.cn/53179.Doc
m.wec.hzldf.cn/82288.Doc
m.wec.hzldf.cn/84284.Doc
m.wec.hzldf.cn/00000.Doc
m.wec.hzldf.cn/44068.Doc
m.wec.hzldf.cn/59577.Doc
m.wec.hzldf.cn/71399.Doc
m.wec.hzldf.cn/26466.Doc
m.wec.hzldf.cn/64404.Doc
m.wec.hzldf.cn/31759.Doc
m.wec.hzldf.cn/68082.Doc
m.wec.hzldf.cn/04864.Doc
m.wec.hzldf.cn/64828.Doc
m.wec.hzldf.cn/53977.Doc
m.wex.hzldf.cn/00662.Doc
m.wex.hzldf.cn/28666.Doc
m.wex.hzldf.cn/02408.Doc
m.wex.hzldf.cn/11999.Doc
m.wex.hzldf.cn/93599.Doc
m.wex.hzldf.cn/84820.Doc
m.wex.hzldf.cn/26802.Doc
m.wex.hzldf.cn/73559.Doc
