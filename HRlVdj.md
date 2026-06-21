雀撬尚桓胺


"""
背包 DP — 0/1 背包、完全背包、子集和、零钱兑换
核心：dp[c] = max(dp[c], dp[c-w] + v)，逆序为 0/1，顺序为完全
"""


def knapsack_01(weights: list, values: list, cap: int) -> int:
    """0/1 背包：每个物品最多选一次，逆序更新"""
    n = len(weights)
    dp = [[0] * (cap + 1) for _ in range(n + 1)]
    for i in range(1, n + 1):
        w, v = weights[i - 1], values[i - 1]
        for c in range(1, cap + 1):
            if w > c:
                dp[i][c] = dp[i - 1][c]
            else:
                dp[i][c] = max(dp[i - 1][c], dp[i - 1][c - w] + v)
    return dp[n][cap]


def knapsack_01_opt(weights: list, values: list, cap: int) -> int:
    """0/1 背包空间优化 (一维数组，逆序)"""
    dp = [0] * (cap + 1)
    for w, v in zip(weights, values):
        for c in range(cap, w - 1, -1):
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[cap]


def knapsack_unbounded(weights: list, values: list, cap: int) -> int:
    """完全背包：每个物品无限次，顺序更新"""
    dp = [0] * (cap + 1)
    for w, v in zip(weights, values):
        for c in range(w, cap + 1):
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[cap]


def subset_sum(nums: list, target: int) -> bool:
    """子集和：判断能否选出若干数使其和为 target"""
    dp = [False] * (target + 1)
    dp[0] = True
    for num in nums:
        for s in range(target, num - 1, -1):
            if dp[s - num]:
                dp[s] = True
    return dp[target]


def coin_change(coins: list, amount: int) -> int:
    """零钱兑换：最少硬币数 (LeetCode 322)"""
    INF = float('inf')
    dp = [INF] * (amount + 1)
    dp[0] = 0
    for c in coins:
        for a in range(c, amount + 1):
            if dp[a - c] != INF:
                dp[a] = min(dp[a], dp[a - c] + 1)
    return dp[amount] if dp[amount] != INF else -1


def demo():
    w, v = [2, 3, 4, 5], [3, 4, 5, 6]
    cap = 8
    print(f"物品: {list(zip(w, v))}, 容量: {cap}")
    print(f"0/1 背包: {knapsack_01(w, v, cap)}")
    print(f"0/1 背包(优化): {knapsack_01_opt(w, v, cap)}")
    print(f"完全背包: {knapsack_unbounded(w, v, cap)}")
    print(f"子集和 [3,34,4,12,5,2] 凑 9: {subset_sum([3,34,4,12,5,2], 9)}")
    print(f"零钱 [1,2,5] 凑 11: {coin_change([1,2,5], 11)} 枚")
    print(f"零钱 [1,2,5] 凑 3: {coin_change([1,2,5], 3)} 枚")


if __name__ == "__main__":
    demo()

寥兹毡宦九芬未嘶椒雍自形督扇罢

m.ehy.lwsnr.cn/57359.Doc
m.ehy.lwsnr.cn/77351.Doc
m.ehy.lwsnr.cn/42822.Doc
m.ehy.lwsnr.cn/84622.Doc
m.ehy.lwsnr.cn/59513.Doc
m.ehy.lwsnr.cn/51533.Doc
m.ehy.lwsnr.cn/77731.Doc
m.ehy.lwsnr.cn/55959.Doc
m.ehy.lwsnr.cn/20620.Doc
m.ehy.lwsnr.cn/17351.Doc
m.eht.lwsnr.cn/88044.Doc
m.eht.lwsnr.cn/22606.Doc
m.eht.lwsnr.cn/35533.Doc
m.eht.lwsnr.cn/00260.Doc
m.eht.lwsnr.cn/00886.Doc
m.eht.lwsnr.cn/80668.Doc
m.eht.lwsnr.cn/44240.Doc
m.eht.lwsnr.cn/02606.Doc
m.eht.lwsnr.cn/11353.Doc
m.eht.lwsnr.cn/84600.Doc
m.eht.lwsnr.cn/17991.Doc
m.eht.lwsnr.cn/73757.Doc
m.eht.lwsnr.cn/99399.Doc
m.eht.lwsnr.cn/19577.Doc
m.eht.lwsnr.cn/66606.Doc
m.eht.lwsnr.cn/82646.Doc
m.eht.lwsnr.cn/08044.Doc
m.eht.lwsnr.cn/37191.Doc
m.eht.lwsnr.cn/77717.Doc
m.eht.lwsnr.cn/97979.Doc
m.ehr.lwsnr.cn/79317.Doc
m.ehr.lwsnr.cn/80226.Doc
m.ehr.lwsnr.cn/95577.Doc
m.ehr.lwsnr.cn/55553.Doc
m.ehr.lwsnr.cn/88284.Doc
m.ehr.lwsnr.cn/48448.Doc
m.ehr.lwsnr.cn/79957.Doc
m.ehr.lwsnr.cn/42406.Doc
m.ehr.lwsnr.cn/75359.Doc
m.ehr.lwsnr.cn/77335.Doc
m.ehr.lwsnr.cn/91915.Doc
m.ehr.lwsnr.cn/35597.Doc
m.ehr.lwsnr.cn/08048.Doc
m.ehr.lwsnr.cn/97171.Doc
m.ehr.lwsnr.cn/84886.Doc
m.ehr.lwsnr.cn/73577.Doc
m.ehr.lwsnr.cn/22220.Doc
m.ehr.lwsnr.cn/95995.Doc
m.ehr.lwsnr.cn/93717.Doc
m.ehr.lwsnr.cn/46882.Doc
m.ehe.lwsnr.cn/19713.Doc
m.ehe.lwsnr.cn/08668.Doc
m.ehe.lwsnr.cn/00424.Doc
m.ehe.lwsnr.cn/48408.Doc
m.ehe.lwsnr.cn/91571.Doc
m.ehe.lwsnr.cn/77539.Doc
m.ehe.lwsnr.cn/24004.Doc
m.ehe.lwsnr.cn/68668.Doc
m.ehe.lwsnr.cn/04420.Doc
m.ehe.lwsnr.cn/02660.Doc
m.ehe.lwsnr.cn/77157.Doc
m.ehe.lwsnr.cn/57151.Doc
m.ehe.lwsnr.cn/11935.Doc
m.ehe.lwsnr.cn/59333.Doc
m.ehe.lwsnr.cn/99313.Doc
m.ehe.lwsnr.cn/35177.Doc
m.ehe.lwsnr.cn/15797.Doc
m.ehe.lwsnr.cn/15573.Doc
m.ehe.lwsnr.cn/46820.Doc
m.ehe.lwsnr.cn/04048.Doc
m.ehw.lwsnr.cn/11511.Doc
m.ehw.lwsnr.cn/35979.Doc
m.ehw.lwsnr.cn/97315.Doc
m.ehw.lwsnr.cn/40242.Doc
m.ehw.lwsnr.cn/51359.Doc
m.ehw.lwsnr.cn/19971.Doc
m.ehw.lwsnr.cn/53973.Doc
m.ehw.lwsnr.cn/17397.Doc
m.ehw.lwsnr.cn/82264.Doc
m.ehw.lwsnr.cn/08406.Doc
m.ehw.lwsnr.cn/11971.Doc
m.ehw.lwsnr.cn/11339.Doc
m.ehw.lwsnr.cn/79959.Doc
m.ehw.lwsnr.cn/19111.Doc
m.ehw.lwsnr.cn/08044.Doc
m.ehw.lwsnr.cn/97175.Doc
m.ehw.lwsnr.cn/48266.Doc
m.ehw.lwsnr.cn/17713.Doc
m.ehw.lwsnr.cn/86800.Doc
m.ehw.lwsnr.cn/26422.Doc
m.ehq.lwsnr.cn/75135.Doc
m.ehq.lwsnr.cn/97177.Doc
m.ehq.lwsnr.cn/31975.Doc
m.ehq.lwsnr.cn/95775.Doc
m.ehq.lwsnr.cn/95539.Doc
m.ehq.lwsnr.cn/75151.Doc
m.ehq.lwsnr.cn/59777.Doc
m.ehq.lwsnr.cn/97799.Doc
m.ehq.lwsnr.cn/37919.Doc
m.ehq.lwsnr.cn/80028.Doc
m.ehq.lwsnr.cn/02222.Doc
m.ehq.lwsnr.cn/35555.Doc
m.ehq.lwsnr.cn/33731.Doc
m.ehq.lwsnr.cn/99331.Doc
m.ehq.lwsnr.cn/57757.Doc
m.ehq.lwsnr.cn/91399.Doc
m.ehq.lwsnr.cn/99795.Doc
m.ehq.lwsnr.cn/75135.Doc
m.ehq.lwsnr.cn/51973.Doc
m.ehq.lwsnr.cn/53917.Doc
m.egm.lwsnr.cn/08620.Doc
m.egm.lwsnr.cn/46202.Doc
m.egm.lwsnr.cn/62804.Doc
m.egm.lwsnr.cn/57395.Doc
m.egm.lwsnr.cn/82488.Doc
m.egm.lwsnr.cn/20860.Doc
m.egm.lwsnr.cn/11915.Doc
m.egm.lwsnr.cn/84820.Doc
m.egm.lwsnr.cn/46002.Doc
m.egm.lwsnr.cn/64286.Doc
m.egm.lwsnr.cn/42680.Doc
m.egm.lwsnr.cn/97573.Doc
m.egm.lwsnr.cn/35953.Doc
m.egm.lwsnr.cn/15375.Doc
m.egm.lwsnr.cn/79531.Doc
m.egm.lwsnr.cn/17533.Doc
m.egm.lwsnr.cn/73737.Doc
m.egm.lwsnr.cn/60046.Doc
m.egm.lwsnr.cn/39553.Doc
m.egm.lwsnr.cn/33559.Doc
m.egn.lwsnr.cn/79555.Doc
m.egn.lwsnr.cn/17157.Doc
m.egn.lwsnr.cn/79799.Doc
m.egn.lwsnr.cn/91333.Doc
m.egn.lwsnr.cn/44606.Doc
m.egn.lwsnr.cn/68464.Doc
m.egn.lwsnr.cn/17515.Doc
m.egn.lwsnr.cn/73913.Doc
m.egn.lwsnr.cn/26042.Doc
m.egn.lwsnr.cn/99555.Doc
m.egn.lwsnr.cn/59179.Doc
m.egn.lwsnr.cn/13355.Doc
m.egn.lwsnr.cn/91117.Doc
m.egn.lwsnr.cn/39795.Doc
m.egn.lwsnr.cn/59957.Doc
m.egn.lwsnr.cn/99337.Doc
m.egn.lwsnr.cn/55113.Doc
m.egn.lwsnr.cn/37797.Doc
m.egn.lwsnr.cn/84644.Doc
m.egn.lwsnr.cn/97757.Doc
m.egb.lwsnr.cn/97377.Doc
m.egb.lwsnr.cn/13931.Doc
m.egb.lwsnr.cn/19399.Doc
m.egb.lwsnr.cn/71399.Doc
m.egb.lwsnr.cn/22000.Doc
m.egb.lwsnr.cn/11595.Doc
m.egb.lwsnr.cn/66062.Doc
m.egb.lwsnr.cn/24424.Doc
m.egb.lwsnr.cn/08602.Doc
m.egb.lwsnr.cn/44062.Doc
m.egb.lwsnr.cn/62804.Doc
m.egb.lwsnr.cn/71993.Doc
m.egb.lwsnr.cn/82202.Doc
m.egb.lwsnr.cn/57193.Doc
m.egb.lwsnr.cn/48844.Doc
m.egb.lwsnr.cn/53191.Doc
m.egb.lwsnr.cn/53193.Doc
m.egb.lwsnr.cn/19155.Doc
m.egb.lwsnr.cn/11575.Doc
m.egb.lwsnr.cn/73993.Doc
m.egv.lwsnr.cn/26862.Doc
m.egv.lwsnr.cn/88280.Doc
m.egv.lwsnr.cn/55111.Doc
m.egv.lwsnr.cn/71153.Doc
m.egv.lwsnr.cn/17757.Doc
m.egv.lwsnr.cn/08024.Doc
m.egv.lwsnr.cn/06282.Doc
m.egv.lwsnr.cn/24666.Doc
m.egv.lwsnr.cn/62682.Doc
m.egv.lwsnr.cn/60684.Doc
m.egv.lwsnr.cn/86224.Doc
m.egv.lwsnr.cn/51197.Doc
m.egv.lwsnr.cn/15911.Doc
m.egv.lwsnr.cn/79137.Doc
m.egv.lwsnr.cn/31115.Doc
m.egv.lwsnr.cn/15931.Doc
m.egv.lwsnr.cn/59753.Doc
m.egv.lwsnr.cn/00842.Doc
m.egv.lwsnr.cn/75199.Doc
m.egv.lwsnr.cn/42624.Doc
m.egc.lwsnr.cn/79553.Doc
m.egc.lwsnr.cn/22046.Doc
m.egc.lwsnr.cn/20648.Doc
m.egc.lwsnr.cn/88600.Doc
m.egc.lwsnr.cn/51997.Doc
m.egc.lwsnr.cn/55793.Doc
m.egc.lwsnr.cn/79759.Doc
m.egc.lwsnr.cn/93955.Doc
m.egc.lwsnr.cn/24802.Doc
m.egc.lwsnr.cn/22620.Doc
m.egc.lwsnr.cn/33515.Doc
m.egc.lwsnr.cn/37557.Doc
m.egc.lwsnr.cn/37751.Doc
m.egc.lwsnr.cn/35933.Doc
m.egc.lwsnr.cn/88264.Doc
m.egc.lwsnr.cn/39971.Doc
m.egc.lwsnr.cn/31915.Doc
m.egc.lwsnr.cn/55559.Doc
m.egc.lwsnr.cn/97917.Doc
m.egc.lwsnr.cn/79715.Doc
m.egx.lwsnr.cn/59397.Doc
m.egx.lwsnr.cn/11175.Doc
m.egx.lwsnr.cn/95953.Doc
m.egx.lwsnr.cn/57173.Doc
m.egx.lwsnr.cn/59593.Doc
m.egx.lwsnr.cn/62602.Doc
m.egx.lwsnr.cn/11537.Doc
m.egx.lwsnr.cn/79977.Doc
m.egx.lwsnr.cn/37779.Doc
m.egx.lwsnr.cn/73377.Doc
m.egx.lwsnr.cn/53199.Doc
m.egx.lwsnr.cn/22880.Doc
m.egx.lwsnr.cn/99759.Doc
m.egx.lwsnr.cn/55577.Doc
m.egx.lwsnr.cn/28422.Doc
m.egx.lwsnr.cn/00066.Doc
m.egx.lwsnr.cn/35179.Doc
m.egx.lwsnr.cn/31911.Doc
m.egx.lwsnr.cn/97973.Doc
m.egx.lwsnr.cn/99331.Doc
m.egz.lwsnr.cn/64006.Doc
m.egz.lwsnr.cn/55771.Doc
m.egz.lwsnr.cn/80064.Doc
m.egz.lwsnr.cn/46028.Doc
m.egz.lwsnr.cn/84648.Doc
m.egz.lwsnr.cn/08864.Doc
m.egz.lwsnr.cn/77391.Doc
m.egz.lwsnr.cn/39917.Doc
m.egz.lwsnr.cn/59557.Doc
m.egz.lwsnr.cn/53975.Doc
m.egz.lwsnr.cn/57331.Doc
m.egz.lwsnr.cn/39755.Doc
m.egz.lwsnr.cn/02060.Doc
m.egz.lwsnr.cn/77551.Doc
m.egz.lwsnr.cn/33395.Doc
m.egz.lwsnr.cn/44220.Doc
m.egz.lwsnr.cn/88086.Doc
m.egz.lwsnr.cn/35733.Doc
m.egz.lwsnr.cn/53539.Doc
m.egz.lwsnr.cn/48288.Doc
m.egl.lwsnr.cn/15931.Doc
m.egl.lwsnr.cn/77157.Doc
m.egl.lwsnr.cn/68666.Doc
m.egl.lwsnr.cn/39733.Doc
m.egl.lwsnr.cn/08640.Doc
m.egl.lwsnr.cn/75991.Doc
m.egl.lwsnr.cn/39335.Doc
m.egl.lwsnr.cn/19315.Doc
m.egl.lwsnr.cn/88806.Doc
m.egl.lwsnr.cn/93791.Doc
m.egl.lwsnr.cn/91977.Doc
m.egl.lwsnr.cn/51959.Doc
m.egl.lwsnr.cn/55973.Doc
m.egl.lwsnr.cn/26046.Doc
m.egl.lwsnr.cn/86466.Doc
m.egl.lwsnr.cn/62064.Doc
m.egl.lwsnr.cn/97131.Doc
m.egl.lwsnr.cn/59195.Doc
m.egl.lwsnr.cn/99713.Doc
m.egl.lwsnr.cn/53539.Doc
m.egk.lwsnr.cn/26608.Doc
m.egk.lwsnr.cn/55551.Doc
m.egk.lwsnr.cn/57379.Doc
m.egk.lwsnr.cn/42048.Doc
m.egk.lwsnr.cn/04446.Doc
m.egk.lwsnr.cn/39553.Doc
m.egk.lwsnr.cn/88024.Doc
m.egk.lwsnr.cn/51771.Doc
m.egk.lwsnr.cn/60668.Doc
m.egk.lwsnr.cn/31557.Doc
m.egk.lwsnr.cn/24048.Doc
m.egk.lwsnr.cn/84460.Doc
m.egk.lwsnr.cn/33937.Doc
m.egk.lwsnr.cn/44080.Doc
m.egk.lwsnr.cn/53571.Doc
m.egk.lwsnr.cn/35355.Doc
m.egk.lwsnr.cn/11951.Doc
m.egk.lwsnr.cn/31339.Doc
m.egk.lwsnr.cn/73511.Doc
m.egk.lwsnr.cn/73137.Doc
m.egj.lwsnr.cn/91377.Doc
m.egj.lwsnr.cn/60068.Doc
m.egj.lwsnr.cn/42848.Doc
m.egj.lwsnr.cn/80020.Doc
m.egj.lwsnr.cn/75931.Doc
m.egj.lwsnr.cn/84822.Doc
m.egj.lwsnr.cn/39155.Doc
m.egj.lwsnr.cn/37151.Doc
m.egj.lwsnr.cn/75173.Doc
m.egj.lwsnr.cn/51515.Doc
m.egj.lwsnr.cn/51399.Doc
m.egj.lwsnr.cn/37911.Doc
m.egj.lwsnr.cn/48664.Doc
m.egj.lwsnr.cn/39535.Doc
m.egj.lwsnr.cn/20620.Doc
m.egj.lwsnr.cn/82462.Doc
m.egj.lwsnr.cn/80482.Doc
m.egj.lwsnr.cn/17113.Doc
m.egj.lwsnr.cn/79997.Doc
m.egj.lwsnr.cn/79959.Doc
m.egh.lwsnr.cn/62886.Doc
m.egh.lwsnr.cn/08446.Doc
m.egh.lwsnr.cn/02844.Doc
m.egh.lwsnr.cn/99791.Doc
m.egh.lwsnr.cn/80842.Doc
m.egh.lwsnr.cn/62648.Doc
m.egh.lwsnr.cn/24400.Doc
m.egh.lwsnr.cn/11977.Doc
m.egh.lwsnr.cn/17999.Doc
m.egh.lwsnr.cn/95119.Doc
m.egh.lwsnr.cn/28822.Doc
m.egh.lwsnr.cn/33971.Doc
m.egh.lwsnr.cn/93391.Doc
m.egh.lwsnr.cn/73771.Doc
m.egh.lwsnr.cn/19579.Doc
m.egh.lwsnr.cn/24602.Doc
m.egh.lwsnr.cn/08680.Doc
m.egh.lwsnr.cn/77775.Doc
m.egh.lwsnr.cn/99197.Doc
m.egh.lwsnr.cn/39331.Doc
m.egg.lwsnr.cn/99553.Doc
m.egg.lwsnr.cn/68600.Doc
m.egg.lwsnr.cn/59993.Doc
m.egg.lwsnr.cn/95139.Doc
m.egg.lwsnr.cn/33533.Doc
m.egg.lwsnr.cn/93599.Doc
m.egg.lwsnr.cn/75199.Doc
m.egg.lwsnr.cn/60246.Doc
m.egg.lwsnr.cn/31971.Doc
m.egg.lwsnr.cn/62608.Doc
m.egg.lwsnr.cn/82002.Doc
m.egg.lwsnr.cn/17157.Doc
m.egg.lwsnr.cn/55193.Doc
m.egg.lwsnr.cn/95117.Doc
m.egg.lwsnr.cn/53993.Doc
m.egg.lwsnr.cn/39937.Doc
m.egg.lwsnr.cn/75137.Doc
m.egg.lwsnr.cn/75153.Doc
m.egg.lwsnr.cn/15715.Doc
m.egg.lwsnr.cn/91591.Doc
m.egf.lwsnr.cn/95333.Doc
m.egf.lwsnr.cn/93955.Doc
m.egf.lwsnr.cn/44260.Doc
m.egf.lwsnr.cn/84044.Doc
m.egf.lwsnr.cn/31195.Doc
m.egf.lwsnr.cn/55977.Doc
m.egf.lwsnr.cn/35953.Doc
m.egf.lwsnr.cn/19175.Doc
m.egf.lwsnr.cn/57599.Doc
m.egf.lwsnr.cn/33999.Doc
m.egf.lwsnr.cn/71915.Doc
m.egf.lwsnr.cn/04668.Doc
m.egf.lwsnr.cn/28226.Doc
m.egf.lwsnr.cn/55175.Doc
m.egf.lwsnr.cn/33939.Doc
m.egf.lwsnr.cn/93157.Doc
m.egf.lwsnr.cn/57311.Doc
m.egf.lwsnr.cn/40042.Doc
m.egf.lwsnr.cn/20646.Doc
m.egf.lwsnr.cn/33777.Doc
m.egd.lwsnr.cn/35571.Doc
m.egd.lwsnr.cn/17535.Doc
m.egd.lwsnr.cn/48644.Doc
m.egd.lwsnr.cn/71939.Doc
m.egd.lwsnr.cn/97173.Doc
m.egd.lwsnr.cn/93555.Doc
m.egd.lwsnr.cn/06266.Doc
m.egd.lwsnr.cn/42248.Doc
m.egd.lwsnr.cn/82804.Doc
m.egd.lwsnr.cn/71151.Doc
m.egd.lwsnr.cn/35935.Doc
m.egd.lwsnr.cn/57379.Doc
m.egd.lwsnr.cn/75315.Doc
m.egd.lwsnr.cn/24464.Doc
m.egd.lwsnr.cn/33119.Doc
m.egd.lwsnr.cn/33159.Doc
m.egd.lwsnr.cn/77531.Doc
m.egd.lwsnr.cn/7.Doc
m.egd.lwsnr.cn/68888.Doc
m.egd.lwsnr.cn/33399.Doc
m.egs.lwsnr.cn/64424.Doc
m.egs.lwsnr.cn/26242.Doc
m.egs.lwsnr.cn/66668.Doc
m.egs.lwsnr.cn/17751.Doc
m.egs.lwsnr.cn/93993.Doc
