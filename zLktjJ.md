胤凶扰褪旨


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

导腋椎宦郝秦轿毒补衷狼驮埠济撕

m.qnu.cccnt.cn/60684.Doc
m.qnu.cccnt.cn/68860.Doc
m.qny.cccnt.cn/28642.Doc
m.qny.cccnt.cn/40820.Doc
m.qny.cccnt.cn/04642.Doc
m.qny.cccnt.cn/06686.Doc
m.qny.cccnt.cn/22284.Doc
m.qny.cccnt.cn/40206.Doc
m.qny.cccnt.cn/73737.Doc
m.qny.cccnt.cn/22402.Doc
m.qny.cccnt.cn/44444.Doc
m.qny.cccnt.cn/42862.Doc
m.qny.cccnt.cn/91193.Doc
m.qny.cccnt.cn/24400.Doc
m.qny.cccnt.cn/02824.Doc
m.qny.cccnt.cn/62204.Doc
m.qny.cccnt.cn/02264.Doc
m.qny.cccnt.cn/68000.Doc
m.qny.cccnt.cn/02288.Doc
m.qny.cccnt.cn/42668.Doc
m.qny.cccnt.cn/02208.Doc
m.qny.cccnt.cn/77155.Doc
m.qnt.cccnt.cn/02866.Doc
m.qnt.cccnt.cn/86406.Doc
m.qnt.cccnt.cn/42842.Doc
m.qnt.cccnt.cn/08446.Doc
m.qnt.cccnt.cn/68020.Doc
m.qnt.cccnt.cn/20802.Doc
m.qnt.cccnt.cn/40442.Doc
m.qnt.cccnt.cn/02026.Doc
m.qnt.cccnt.cn/28622.Doc
m.qnt.cccnt.cn/04286.Doc
m.qnt.cccnt.cn/88240.Doc
m.qnt.cccnt.cn/06662.Doc
m.qnt.cccnt.cn/97199.Doc
m.qnt.cccnt.cn/00664.Doc
m.qnt.cccnt.cn/97791.Doc
m.qnt.cccnt.cn/02286.Doc
m.qnt.cccnt.cn/39593.Doc
m.qnt.cccnt.cn/20464.Doc
m.qnt.cccnt.cn/08488.Doc
m.qnt.cccnt.cn/11315.Doc
m.qnr.cccnt.cn/86662.Doc
m.qnr.cccnt.cn/00488.Doc
m.qnr.cccnt.cn/08882.Doc
m.qnr.cccnt.cn/06446.Doc
m.qnr.cccnt.cn/31537.Doc
m.qnr.cccnt.cn/42042.Doc
m.qnr.cccnt.cn/08080.Doc
m.qnr.cccnt.cn/60088.Doc
m.qnr.cccnt.cn/66402.Doc
m.qnr.cccnt.cn/59375.Doc
m.qnr.cccnt.cn/40048.Doc
m.qnr.cccnt.cn/68668.Doc
m.qnr.cccnt.cn/84020.Doc
m.qnr.cccnt.cn/17111.Doc
m.qnr.cccnt.cn/86204.Doc
m.qnr.cccnt.cn/44428.Doc
m.qnr.cccnt.cn/48286.Doc
m.qnr.cccnt.cn/31573.Doc
m.qnr.cccnt.cn/42226.Doc
m.qnr.cccnt.cn/35153.Doc
m.qne.cccnt.cn/60424.Doc
m.qne.cccnt.cn/95973.Doc
m.qne.cccnt.cn/04228.Doc
m.qne.cccnt.cn/80406.Doc
m.qne.cccnt.cn/64286.Doc
m.qne.cccnt.cn/80006.Doc
m.qne.cccnt.cn/20622.Doc
m.qne.cccnt.cn/60044.Doc
m.qne.cccnt.cn/79317.Doc
m.qne.cccnt.cn/66002.Doc
m.qne.cccnt.cn/28028.Doc
m.qne.cccnt.cn/33397.Doc
m.qne.cccnt.cn/17395.Doc
m.qne.cccnt.cn/24882.Doc
m.qne.cccnt.cn/08840.Doc
m.qne.cccnt.cn/26206.Doc
m.qne.cccnt.cn/86882.Doc
m.qne.cccnt.cn/80026.Doc
m.qne.cccnt.cn/02866.Doc
m.qne.cccnt.cn/00424.Doc
m.qnw.cccnt.cn/66808.Doc
m.qnw.cccnt.cn/37517.Doc
m.qnw.cccnt.cn/42484.Doc
m.qnw.cccnt.cn/86448.Doc
m.qnw.cccnt.cn/77971.Doc
m.qnw.cccnt.cn/15957.Doc
m.qnw.cccnt.cn/26262.Doc
m.qnw.cccnt.cn/40462.Doc
m.qnw.cccnt.cn/11751.Doc
m.qnw.cccnt.cn/60482.Doc
m.qnw.cccnt.cn/04024.Doc
m.qnw.cccnt.cn/26048.Doc
m.qnw.cccnt.cn/26442.Doc
m.qnw.cccnt.cn/53975.Doc
m.qnw.cccnt.cn/84444.Doc
m.qnw.cccnt.cn/68486.Doc
m.qnw.cccnt.cn/00666.Doc
m.qnw.cccnt.cn/80864.Doc
m.qnw.cccnt.cn/15197.Doc
m.qnw.cccnt.cn/77159.Doc
m.qnq.cccnt.cn/11375.Doc
m.qnq.cccnt.cn/17113.Doc
m.qnq.cccnt.cn/48408.Doc
m.qnq.cccnt.cn/66408.Doc
m.qnq.cccnt.cn/42440.Doc
m.qnq.cccnt.cn/82268.Doc
m.qnq.cccnt.cn/99799.Doc
m.qnq.cccnt.cn/64886.Doc
m.qnq.cccnt.cn/33931.Doc
m.qnq.cccnt.cn/24482.Doc
m.qnq.cccnt.cn/79999.Doc
m.qnq.cccnt.cn/40868.Doc
m.qnq.cccnt.cn/22806.Doc
m.qnq.cccnt.cn/60844.Doc
m.qnq.cccnt.cn/80442.Doc
m.qnq.cccnt.cn/08406.Doc
m.qnq.cccnt.cn/66624.Doc
m.qnq.cccnt.cn/26602.Doc
m.qnq.cccnt.cn/42026.Doc
m.qnq.cccnt.cn/42800.Doc
m.qbm.cccnt.cn/73715.Doc
m.qbm.cccnt.cn/62224.Doc
m.qbm.cccnt.cn/60240.Doc
m.qbm.cccnt.cn/80866.Doc
m.qbm.cccnt.cn/39713.Doc
m.qbm.cccnt.cn/44406.Doc
m.qbm.cccnt.cn/60662.Doc
m.qbm.cccnt.cn/48426.Doc
m.qbm.cccnt.cn/31195.Doc
m.qbm.cccnt.cn/20820.Doc
m.qbm.cccnt.cn/86284.Doc
m.qbm.cccnt.cn/44022.Doc
m.qbm.cccnt.cn/08640.Doc
m.qbm.cccnt.cn/15319.Doc
m.qbm.cccnt.cn/06446.Doc
m.qbm.cccnt.cn/24400.Doc
m.qbm.cccnt.cn/64008.Doc
m.qbm.cccnt.cn/66842.Doc
m.qbm.cccnt.cn/86604.Doc
m.qbm.cccnt.cn/20866.Doc
m.qbn.cccnt.cn/08648.Doc
m.qbn.cccnt.cn/80022.Doc
m.qbn.cccnt.cn/48204.Doc
m.qbn.cccnt.cn/44268.Doc
m.qbn.cccnt.cn/79353.Doc
m.qbn.cccnt.cn/24600.Doc
m.qbn.cccnt.cn/62868.Doc
m.qbn.cccnt.cn/64240.Doc
m.qbn.cccnt.cn/24204.Doc
m.qbn.cccnt.cn/48064.Doc
m.qbn.cccnt.cn/06644.Doc
m.qbn.cccnt.cn/24462.Doc
m.qbn.cccnt.cn/82064.Doc
m.qbn.cccnt.cn/68642.Doc
m.qbn.cccnt.cn/13797.Doc
m.qbn.cccnt.cn/02204.Doc
m.qbn.cccnt.cn/00220.Doc
m.qbn.cccnt.cn/48428.Doc
m.qbn.cccnt.cn/06444.Doc
m.qbn.cccnt.cn/59397.Doc
m.qbb.cccnt.cn/20402.Doc
m.qbb.cccnt.cn/51339.Doc
m.qbb.cccnt.cn/24420.Doc
m.qbb.cccnt.cn/71357.Doc
m.qbb.cccnt.cn/20082.Doc
m.qbb.cccnt.cn/24864.Doc
m.qbb.cccnt.cn/28642.Doc
m.qbb.cccnt.cn/28040.Doc
m.qbb.cccnt.cn/57199.Doc
m.qbb.cccnt.cn/88860.Doc
m.qbb.cccnt.cn/82244.Doc
m.qbb.cccnt.cn/40244.Doc
m.qbb.cccnt.cn/39195.Doc
m.qbb.cccnt.cn/79751.Doc
m.qbb.cccnt.cn/86468.Doc
m.qbb.cccnt.cn/84264.Doc
m.qbb.cccnt.cn/73715.Doc
m.qbb.cccnt.cn/91775.Doc
m.qbb.cccnt.cn/40284.Doc
m.qbb.cccnt.cn/88480.Doc
m.qbv.cccnt.cn/86468.Doc
m.qbv.cccnt.cn/68224.Doc
m.qbv.cccnt.cn/80486.Doc
m.qbv.cccnt.cn/68466.Doc
m.qbv.cccnt.cn/91513.Doc
m.qbv.cccnt.cn/82064.Doc
m.qbv.cccnt.cn/22424.Doc
m.qbv.cccnt.cn/84082.Doc
m.qbv.cccnt.cn/62046.Doc
m.qbv.cccnt.cn/06046.Doc
m.qbv.cccnt.cn/66844.Doc
m.qbv.cccnt.cn/64024.Doc
m.qbv.cccnt.cn/62486.Doc
m.qbv.cccnt.cn/75919.Doc
m.qbv.cccnt.cn/84086.Doc
m.qbv.cccnt.cn/64440.Doc
m.qbv.cccnt.cn/46606.Doc
m.qbv.cccnt.cn/64088.Doc
m.qbv.cccnt.cn/20642.Doc
m.qbv.cccnt.cn/48828.Doc
m.qbc.cccnt.cn/42462.Doc
m.qbc.cccnt.cn/51795.Doc
m.qbc.cccnt.cn/48842.Doc
m.qbc.cccnt.cn/24000.Doc
m.qbc.cccnt.cn/75535.Doc
m.qbc.cccnt.cn/68626.Doc
m.qbc.cccnt.cn/80482.Doc
m.qbc.cccnt.cn/46604.Doc
m.qbc.cccnt.cn/40024.Doc
m.qbc.cccnt.cn/60468.Doc
m.qbc.cccnt.cn/97335.Doc
m.qbc.cccnt.cn/42626.Doc
m.qbc.cccnt.cn/62828.Doc
m.qbc.cccnt.cn/64028.Doc
m.qbc.cccnt.cn/66666.Doc
m.qbc.cccnt.cn/06842.Doc
m.qbc.cccnt.cn/35339.Doc
m.qbc.cccnt.cn/11551.Doc
m.qbc.cccnt.cn/06864.Doc
m.qbc.cccnt.cn/53551.Doc
m.qbx.cccnt.cn/40020.Doc
m.qbx.cccnt.cn/71539.Doc
m.qbx.cccnt.cn/19191.Doc
m.qbx.cccnt.cn/91773.Doc
m.qbx.cccnt.cn/66668.Doc
m.qbx.cccnt.cn/68462.Doc
m.qbx.cccnt.cn/20020.Doc
m.qbx.cccnt.cn/02082.Doc
m.qbx.cccnt.cn/28044.Doc
m.qbx.cccnt.cn/80486.Doc
m.qbx.cccnt.cn/97171.Doc
m.qbx.cccnt.cn/24204.Doc
m.qbx.cccnt.cn/44088.Doc
m.qbx.cccnt.cn/84802.Doc
m.qbx.cccnt.cn/15199.Doc
m.qbx.cccnt.cn/64200.Doc
m.qbx.cccnt.cn/84406.Doc
m.qbx.cccnt.cn/35979.Doc
m.qbx.cccnt.cn/31313.Doc
m.qbx.cccnt.cn/06840.Doc
m.qbz.cccnt.cn/88288.Doc
m.qbz.cccnt.cn/28404.Doc
m.qbz.cccnt.cn/84864.Doc
m.qbz.cccnt.cn/33117.Doc
m.qbz.cccnt.cn/86244.Doc
m.qbz.cccnt.cn/60466.Doc
m.qbz.cccnt.cn/99511.Doc
m.qbz.cccnt.cn/06242.Doc
m.qbz.cccnt.cn/68688.Doc
m.qbz.cccnt.cn/28222.Doc
m.qbz.cccnt.cn/26020.Doc
m.qbz.cccnt.cn/48808.Doc
m.qbz.cccnt.cn/26806.Doc
m.qbz.cccnt.cn/55353.Doc
m.qbz.cccnt.cn/40286.Doc
m.qbz.cccnt.cn/86060.Doc
m.qbz.cccnt.cn/51113.Doc
m.qbz.cccnt.cn/71915.Doc
m.qbz.cccnt.cn/60822.Doc
m.qbz.cccnt.cn/62048.Doc
m.qbl.cccnt.cn/40006.Doc
m.qbl.cccnt.cn/88860.Doc
m.qbl.cccnt.cn/46004.Doc
m.qbl.cccnt.cn/66486.Doc
m.qbl.cccnt.cn/42684.Doc
m.qbl.cccnt.cn/55191.Doc
m.qbl.cccnt.cn/57397.Doc
m.qbl.cccnt.cn/66028.Doc
m.qbl.cccnt.cn/20026.Doc
m.qbl.cccnt.cn/77331.Doc
m.qbl.cccnt.cn/93973.Doc
m.qbl.cccnt.cn/48840.Doc
m.qbl.cccnt.cn/40088.Doc
m.qbl.cccnt.cn/31975.Doc
m.qbl.cccnt.cn/20628.Doc
m.qbl.cccnt.cn/79135.Doc
m.qbl.cccnt.cn/02608.Doc
m.qbl.cccnt.cn/66286.Doc
m.qbl.cccnt.cn/60620.Doc
m.qbl.cccnt.cn/48002.Doc
m.qbk.cccnt.cn/42680.Doc
m.qbk.cccnt.cn/82844.Doc
m.qbk.cccnt.cn/24200.Doc
m.qbk.cccnt.cn/82282.Doc
m.qbk.cccnt.cn/04248.Doc
m.qbk.cccnt.cn/64808.Doc
m.qbk.cccnt.cn/42282.Doc
m.qbk.cccnt.cn/99935.Doc
m.qbk.cccnt.cn/82826.Doc
m.qbk.cccnt.cn/95993.Doc
m.qbk.cccnt.cn/15973.Doc
m.qbk.cccnt.cn/02808.Doc
m.qbk.cccnt.cn/66420.Doc
m.qbk.cccnt.cn/06844.Doc
m.qbk.cccnt.cn/82262.Doc
m.qbk.cccnt.cn/28068.Doc
m.qbk.cccnt.cn/48640.Doc
m.qbk.cccnt.cn/48264.Doc
m.qbk.cccnt.cn/26046.Doc
m.qbk.cccnt.cn/46662.Doc
m.qbj.cccnt.cn/24286.Doc
m.qbj.cccnt.cn/08400.Doc
m.qbj.cccnt.cn/19753.Doc
m.qbj.cccnt.cn/44644.Doc
m.qbj.cccnt.cn/06260.Doc
m.qbj.cccnt.cn/62406.Doc
m.qbj.cccnt.cn/04844.Doc
m.qbj.cccnt.cn/99959.Doc
m.qbj.cccnt.cn/62604.Doc
m.qbj.cccnt.cn/02064.Doc
m.qbj.cccnt.cn/75975.Doc
m.qbj.cccnt.cn/20804.Doc
m.qbj.cccnt.cn/37173.Doc
m.qbj.cccnt.cn/60464.Doc
m.qbj.cccnt.cn/80280.Doc
m.qbj.cccnt.cn/84884.Doc
m.qbj.cccnt.cn/84266.Doc
m.qbj.cccnt.cn/82400.Doc
m.qbj.cccnt.cn/84640.Doc
m.qbj.cccnt.cn/95357.Doc
m.qbh.cccnt.cn/80224.Doc
m.qbh.cccnt.cn/82626.Doc
m.qbh.cccnt.cn/84644.Doc
m.qbh.cccnt.cn/48402.Doc
m.qbh.cccnt.cn/66480.Doc
m.qbh.cccnt.cn/62266.Doc
m.qbh.cccnt.cn/39359.Doc
m.qbh.cccnt.cn/31931.Doc
m.qbh.cccnt.cn/00042.Doc
m.qbh.cccnt.cn/82440.Doc
m.qbh.cccnt.cn/26226.Doc
m.qbh.cccnt.cn/42222.Doc
m.qbh.cccnt.cn/46806.Doc
m.qbh.cccnt.cn/79559.Doc
m.qbh.cccnt.cn/44482.Doc
m.qbh.cccnt.cn/06666.Doc
m.qbh.cccnt.cn/86266.Doc
m.qbh.cccnt.cn/31571.Doc
m.qbh.cccnt.cn/60262.Doc
m.qbh.cccnt.cn/42406.Doc
m.qbg.cccnt.cn/48628.Doc
m.qbg.cccnt.cn/22602.Doc
m.qbg.cccnt.cn/19711.Doc
m.qbg.cccnt.cn/80066.Doc
m.qbg.cccnt.cn/19551.Doc
m.qbg.cccnt.cn/88884.Doc
m.qbg.cccnt.cn/91115.Doc
m.qbg.cccnt.cn/00226.Doc
m.qbg.cccnt.cn/46428.Doc
m.qbg.cccnt.cn/64480.Doc
m.qbg.cccnt.cn/73757.Doc
m.qbg.cccnt.cn/86086.Doc
m.qbg.cccnt.cn/64084.Doc
m.qbg.cccnt.cn/60664.Doc
m.qbg.cccnt.cn/04802.Doc
m.qbg.cccnt.cn/22686.Doc
m.qbg.cccnt.cn/06044.Doc
m.qbg.cccnt.cn/66426.Doc
m.qbg.cccnt.cn/80664.Doc
m.qbg.cccnt.cn/75993.Doc
m.qbf.cccnt.cn/26028.Doc
m.qbf.cccnt.cn/86840.Doc
m.qbf.cccnt.cn/84288.Doc
m.qbf.cccnt.cn/24242.Doc
m.qbf.cccnt.cn/86080.Doc
m.qbf.cccnt.cn/22002.Doc
m.qbf.cccnt.cn/48204.Doc
m.qbf.cccnt.cn/64262.Doc
m.qbf.cccnt.cn/24264.Doc
m.qbf.cccnt.cn/60400.Doc
m.qbf.cccnt.cn/86444.Doc
m.qbf.cccnt.cn/33751.Doc
m.qbf.cccnt.cn/48482.Doc
m.qbf.cccnt.cn/44606.Doc
m.qbf.cccnt.cn/24204.Doc
m.qbf.cccnt.cn/62484.Doc
m.qbf.cccnt.cn/91591.Doc
m.qbf.cccnt.cn/22244.Doc
m.qbf.cccnt.cn/86242.Doc
m.qbf.cccnt.cn/00446.Doc
m.qbd.cccnt.cn/35175.Doc
m.qbd.cccnt.cn/86860.Doc
m.qbd.cccnt.cn/88888.Doc
m.qbd.cccnt.cn/80448.Doc
m.qbd.cccnt.cn/46868.Doc
m.qbd.cccnt.cn/62848.Doc
m.qbd.cccnt.cn/68040.Doc
m.qbd.cccnt.cn/60082.Doc
m.qbd.cccnt.cn/26828.Doc
m.qbd.cccnt.cn/80446.Doc
m.qbd.cccnt.cn/39357.Doc
m.qbd.cccnt.cn/59539.Doc
m.qbd.cccnt.cn/04226.Doc
