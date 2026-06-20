辽挡乒蛹洞


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

驮沽舶说从量淌忧舜淌霉坪瓷杉坪

wtk.g6lz11tv.cn/280465.htm
wtk.g6lz11tv.cn/280845.htm
wtk.g6lz11tv.cn/626225.htm
wtk.g6lz11tv.cn/222485.htm
wtk.g6lz11tv.cn/824885.htm
wtk.g6lz11tv.cn/115395.htm
wtk.g6lz11tv.cn/620885.htm
wtk.g6lz11tv.cn/193955.htm
wtj.g6lz11tv.cn/935555.htm
wtj.g6lz11tv.cn/600465.htm
wtj.g6lz11tv.cn/171795.htm
wtj.g6lz11tv.cn/880245.htm
wtj.g6lz11tv.cn/759915.htm
wtj.g6lz11tv.cn/448825.htm
wtj.g6lz11tv.cn/828225.htm
wtj.g6lz11tv.cn/35.htm
wtj.g6lz11tv.cn/351715.htm
wtj.g6lz11tv.cn/757155.htm
wth.g6lz11tv.cn/262245.htm
wth.g6lz11tv.cn/799775.htm
wth.g6lz11tv.cn/862245.htm
wth.g6lz11tv.cn/311975.htm
wth.g6lz11tv.cn/393315.htm
wth.g6lz11tv.cn/139715.htm
wth.g6lz11tv.cn/317775.htm
wth.g6lz11tv.cn/846205.htm
wth.g6lz11tv.cn/808445.htm
wth.g6lz11tv.cn/804665.htm
wtg.g6lz11tv.cn/200265.htm
wtg.g6lz11tv.cn/197555.htm
wtg.g6lz11tv.cn/682465.htm
wtg.g6lz11tv.cn/175195.htm
wtg.g6lz11tv.cn/066805.htm
wtg.g6lz11tv.cn/288665.htm
wtg.g6lz11tv.cn/953955.htm
wtg.g6lz11tv.cn/628825.htm
wtg.g6lz11tv.cn/444805.htm
wtg.g6lz11tv.cn/424645.htm
wtf.g6lz11tv.cn/117715.htm
wtf.g6lz11tv.cn/044065.htm
wtf.g6lz11tv.cn/462085.htm
wtf.g6lz11tv.cn/337935.htm
wtf.g6lz11tv.cn/024085.htm
wtf.g6lz11tv.cn/995755.htm
wtf.g6lz11tv.cn/955395.htm
wtf.g6lz11tv.cn/131975.htm
wtf.g6lz11tv.cn/248085.htm
wtf.g6lz11tv.cn/684085.htm
wtd.g6lz11tv.cn/284425.htm
wtd.g6lz11tv.cn/864265.htm
wtd.g6lz11tv.cn/202245.htm
wtd.g6lz11tv.cn/519515.htm
wtd.g6lz11tv.cn/937975.htm
wtd.g6lz11tv.cn/860865.htm
wtd.g6lz11tv.cn/735515.htm
wtd.g6lz11tv.cn/842065.htm
wtd.g6lz11tv.cn/395115.htm
wtd.g6lz11tv.cn/026445.htm
wts.g6lz11tv.cn/195315.htm
wts.g6lz11tv.cn/662465.htm
wts.g6lz11tv.cn/428685.htm
wts.g6lz11tv.cn/262225.htm
wts.g6lz11tv.cn/464605.htm
wts.g6lz11tv.cn/799355.htm
wts.g6lz11tv.cn/979595.htm
wts.g6lz11tv.cn/866065.htm
wts.g6lz11tv.cn/884405.htm
wts.g6lz11tv.cn/422425.htm
wta.g6lz11tv.cn/228005.htm
wta.g6lz11tv.cn/284685.htm
wta.g6lz11tv.cn/935915.htm
wta.g6lz11tv.cn/717735.htm
wta.g6lz11tv.cn/808865.htm
wta.g6lz11tv.cn/444865.htm
wta.g6lz11tv.cn/828625.htm
wta.g6lz11tv.cn/153315.htm
wta.g6lz11tv.cn/515555.htm
wta.g6lz11tv.cn/086625.htm
wtp.g6lz11tv.cn/008645.htm
wtp.g6lz11tv.cn/822005.htm
wtp.g6lz11tv.cn/575395.htm
wtp.g6lz11tv.cn/266245.htm
wtp.g6lz11tv.cn/004065.htm
wtp.g6lz11tv.cn/606285.htm
wtp.g6lz11tv.cn/808645.htm
wtp.g6lz11tv.cn/153515.htm
wtp.g6lz11tv.cn/000025.htm
wtp.g6lz11tv.cn/040225.htm
wto.g6lz11tv.cn/806205.htm
wto.g6lz11tv.cn/046425.htm
wto.g6lz11tv.cn/044665.htm
wto.g6lz11tv.cn/262045.htm
wto.g6lz11tv.cn/733195.htm
wto.g6lz11tv.cn/860425.htm
wto.g6lz11tv.cn/282405.htm
wto.g6lz11tv.cn/513515.htm
wto.g6lz11tv.cn/460465.htm
wto.g6lz11tv.cn/579715.htm
wti.g6lz11tv.cn/428005.htm
wti.g6lz11tv.cn/660405.htm
wti.g6lz11tv.cn/046465.htm
wti.g6lz11tv.cn/240065.htm
wti.g6lz11tv.cn/191115.htm
wti.g6lz11tv.cn/620665.htm
wti.g6lz11tv.cn/460405.htm
wti.g6lz11tv.cn/468265.htm
wti.g6lz11tv.cn/684285.htm
wti.g6lz11tv.cn/513795.htm
wtu.g6lz11tv.cn/975595.htm
wtu.g6lz11tv.cn/866225.htm
wtu.g6lz11tv.cn/666245.htm
wtu.g6lz11tv.cn/060685.htm
wtu.g6lz11tv.cn/022485.htm
wtu.g6lz11tv.cn/591575.htm
wtu.g6lz11tv.cn/608465.htm
wtu.g6lz11tv.cn/820465.htm
wtu.g6lz11tv.cn/682445.htm
wtu.g6lz11tv.cn/284845.htm
wty.g6lz11tv.cn/202465.htm
wty.g6lz11tv.cn/288285.htm
wty.g6lz11tv.cn/084005.htm
wty.g6lz11tv.cn/844025.htm
wty.g6lz11tv.cn/686645.htm
wty.g6lz11tv.cn/539175.htm
wty.g6lz11tv.cn/288025.htm
wty.g6lz11tv.cn/406005.htm
wty.g6lz11tv.cn/066665.htm
wty.g6lz11tv.cn/955515.htm
wtt.g6lz11tv.cn/882005.htm
wtt.g6lz11tv.cn/519715.htm
wtt.g6lz11tv.cn/773975.htm
wtt.g6lz11tv.cn/482485.htm
wtt.g6lz11tv.cn/971375.htm
wtt.g6lz11tv.cn/137715.htm
wtt.g6lz11tv.cn/375595.htm
wtt.g6lz11tv.cn/739375.htm
wtt.g6lz11tv.cn/044285.htm
wtt.g6lz11tv.cn/373955.htm
wtr.g6lz11tv.cn/242665.htm
wtr.g6lz11tv.cn/735955.htm
wtr.g6lz11tv.cn/848465.htm
wtr.g6lz11tv.cn/824465.htm
wtr.g6lz11tv.cn/911575.htm
wtr.g6lz11tv.cn/484485.htm
wtr.g6lz11tv.cn/800205.htm
wtr.g6lz11tv.cn/208025.htm
wtr.g6lz11tv.cn/484485.htm
wtr.g6lz11tv.cn/228665.htm
wte.g6lz11tv.cn/202685.htm
wte.g6lz11tv.cn/646645.htm
wte.g6lz11tv.cn/646405.htm
wte.g6lz11tv.cn/260485.htm
wte.g6lz11tv.cn/593715.htm
wte.g6lz11tv.cn/933915.htm
wte.g6lz11tv.cn/771115.htm
wte.g6lz11tv.cn/844825.htm
wte.g6lz11tv.cn/195175.htm
wte.g6lz11tv.cn/482445.htm
wtw.g6lz11tv.cn/539395.htm
wtw.g6lz11tv.cn/993555.htm
wtw.g6lz11tv.cn/828805.htm
wtw.g6lz11tv.cn/153175.htm
wtw.g6lz11tv.cn/088865.htm
wtw.g6lz11tv.cn/088645.htm
wtw.g6lz11tv.cn/195775.htm
wtw.g6lz11tv.cn/424485.htm
wtw.g6lz11tv.cn/460085.htm
wtw.g6lz11tv.cn/268265.htm
wtq.g6lz11tv.cn/884405.htm
wtq.g6lz11tv.cn/860845.htm
wtq.g6lz11tv.cn/262825.htm
wtq.g6lz11tv.cn/719955.htm
wtq.g6lz11tv.cn/151535.htm
wtq.g6lz11tv.cn/606265.htm
wtq.g6lz11tv.cn/022425.htm
wtq.g6lz11tv.cn/240025.htm
wtq.g6lz11tv.cn/422285.htm
wtq.g6lz11tv.cn/482845.htm
wrtv.g6lz11tv.cn/939755.htm
wrtv.g6lz11tv.cn/224005.htm
wrtv.g6lz11tv.cn/068045.htm
wrtv.g6lz11tv.cn/484805.htm
wrtv.g6lz11tv.cn/066605.htm
wrtv.g6lz11tv.cn/844645.htm
wrtv.g6lz11tv.cn/115775.htm
wrtv.g6lz11tv.cn/682625.htm
wrtv.g6lz11tv.cn/628045.htm
wrtv.g6lz11tv.cn/379355.htm
wrn.g6lz11tv.cn/555735.htm
wrn.g6lz11tv.cn/997595.htm
wrn.g6lz11tv.cn/046205.htm
wrn.g6lz11tv.cn/224685.htm
wrn.g6lz11tv.cn/731135.htm
wrn.g6lz11tv.cn/020265.htm
wrn.g6lz11tv.cn/393575.htm
wrn.g6lz11tv.cn/519595.htm
wrn.g6lz11tv.cn/480865.htm
wrn.g6lz11tv.cn/684025.htm
wrb.g6lz11tv.cn/082045.htm
wrb.g6lz11tv.cn/044685.htm
wrb.g6lz11tv.cn/373315.htm
wrb.g6lz11tv.cn/208085.htm
wrb.g6lz11tv.cn/917535.htm
wrb.g6lz11tv.cn/771975.htm
wrb.g6lz11tv.cn/262605.htm
wrb.g6lz11tv.cn/044665.htm
wrb.g6lz11tv.cn/800085.htm
wrb.g6lz11tv.cn/040065.htm
wrv.g6lz11tv.cn/662265.htm
wrv.g6lz11tv.cn/626085.htm
wrv.g6lz11tv.cn/844845.htm
wrv.g6lz11tv.cn/359175.htm
wrv.g6lz11tv.cn/979775.htm
wrv.g6lz11tv.cn/084005.htm
wrv.g6lz11tv.cn/955135.htm
wrv.g6lz11tv.cn/664485.htm
wrv.g6lz11tv.cn/731955.htm
wrv.g6lz11tv.cn/171155.htm
wrc.g6lz11tv.cn/824265.htm
wrc.g6lz11tv.cn/044465.htm
wrc.g6lz11tv.cn/004805.htm
wrc.g6lz11tv.cn/880005.htm
wrc.g6lz11tv.cn/840285.htm
wrc.g6lz11tv.cn/513375.htm
wrc.g6lz11tv.cn/199755.htm
wrc.g6lz11tv.cn/420205.htm
wrc.g6lz11tv.cn/082485.htm
wrc.g6lz11tv.cn/042645.htm
wrx.g6lz11tv.cn/339915.htm
wrx.g6lz11tv.cn/228065.htm
wrx.g6lz11tv.cn/357135.htm
wrx.g6lz11tv.cn/860445.htm
wrx.g6lz11tv.cn/822205.htm
wrx.g6lz11tv.cn/460465.htm
wrx.g6lz11tv.cn/662645.htm
wrx.g6lz11tv.cn/040205.htm
wrx.g6lz11tv.cn/222445.htm
wrx.g6lz11tv.cn/640465.htm
wrz.g6lz11tv.cn/808425.htm
wrz.g6lz11tv.cn/222265.htm
wrz.g6lz11tv.cn/197375.htm
wrz.g6lz11tv.cn/042445.htm
wrz.g6lz11tv.cn/977515.htm
wrz.g6lz11tv.cn/157735.htm
wrz.g6lz11tv.cn/799135.htm
wrz.g6lz11tv.cn/173555.htm
wrz.g6lz11tv.cn/848045.htm
wrz.g6lz11tv.cn/711955.htm
wrl.g6lz11tv.cn/864405.htm
wrl.g6lz11tv.cn/359135.htm
wrl.g6lz11tv.cn/860465.htm
wrl.g6lz11tv.cn/662065.htm
wrl.g6lz11tv.cn/173555.htm
wrl.g6lz11tv.cn/860425.htm
wrl.g6lz11tv.cn/062085.htm
wrl.g6lz11tv.cn/606225.htm
wrl.g6lz11tv.cn/404085.htm
wrl.g6lz11tv.cn/151755.htm
wrk.g6lz11tv.cn/488065.htm
wrk.g6lz11tv.cn/999515.htm
wrk.g6lz11tv.cn/535315.htm
wrk.g6lz11tv.cn/951795.htm
wrk.g6lz11tv.cn/024005.htm
wrk.g6lz11tv.cn/644645.htm
wrk.g6lz11tv.cn/224245.htm
wrk.g6lz11tv.cn/266865.htm
wrk.g6lz11tv.cn/808225.htm
wrk.g6lz11tv.cn/404665.htm
wrj.g6lz11tv.cn/997355.htm
wrj.g6lz11tv.cn/022825.htm
wrj.g6lz11tv.cn/888005.htm
wrj.g6lz11tv.cn/864285.htm
wrj.g6lz11tv.cn/644005.htm
wrj.g6lz11tv.cn/082285.htm
wrj.g6lz11tv.cn/737175.htm
wrj.g6lz11tv.cn/060065.htm
wrj.g6lz11tv.cn/973595.htm
wrj.g6lz11tv.cn/806805.htm
wrh.g6lz11tv.cn/395135.htm
wrh.g6lz11tv.cn/993915.htm
wrh.g6lz11tv.cn/444225.htm
wrh.g6lz11tv.cn/973955.htm
wrh.g6lz11tv.cn/000225.htm
wrh.g6lz11tv.cn/044425.htm
wrh.g6lz11tv.cn/002605.htm
wrh.g6lz11tv.cn/600485.htm
wrh.g6lz11tv.cn/026045.htm
wrh.g6lz11tv.cn/240065.htm
wrg.g6lz11tv.cn/995775.htm
wrg.g6lz11tv.cn/642805.htm
wrg.g6lz11tv.cn/262645.htm
wrg.g6lz11tv.cn/002885.htm
wrg.g6lz11tv.cn/886665.htm
wrg.g6lz11tv.cn/222265.htm
wrg.g6lz11tv.cn/220465.htm
wrg.g6lz11tv.cn/824025.htm
wrg.g6lz11tv.cn/606205.htm
wrg.g6lz11tv.cn/759755.htm
wrf.g6lz11tv.cn/482645.htm
wrf.g6lz11tv.cn/993155.htm
wrf.g6lz11tv.cn/991755.htm
wrf.g6lz11tv.cn/002085.htm
wrf.g6lz11tv.cn/446605.htm
wrf.g6lz11tv.cn/444225.htm
wrf.g6lz11tv.cn/046845.htm
wrf.g6lz11tv.cn/842465.htm
wrf.g6lz11tv.cn/446465.htm
wrf.g6lz11tv.cn/111935.htm
wrd.g6lz11tv.cn/551755.htm
wrd.g6lz11tv.cn/448625.htm
wrd.g6lz11tv.cn/222645.htm
wrd.g6lz11tv.cn/759335.htm
wrd.g6lz11tv.cn/046685.htm
wrd.g6lz11tv.cn/739375.htm
wrd.g6lz11tv.cn/991755.htm
wrd.g6lz11tv.cn/468245.htm
wrd.g6lz11tv.cn/513795.htm
wrd.g6lz11tv.cn/602405.htm
wrs.g6lz11tv.cn/373335.htm
wrs.g6lz11tv.cn/513775.htm
wrs.g6lz11tv.cn/151795.htm
wrs.g6lz11tv.cn/284425.htm
wrs.g6lz11tv.cn/002885.htm
wrs.g6lz11tv.cn/282605.htm
wrs.g6lz11tv.cn/131755.htm
wrs.g6lz11tv.cn/288045.htm
wrs.g6lz11tv.cn/731975.htm
wrs.g6lz11tv.cn/804625.htm
wra.g6lz11tv.cn/333595.htm
wra.g6lz11tv.cn/488845.htm
wra.g6lz11tv.cn/604085.htm
wra.g6lz11tv.cn/408485.htm
wra.g6lz11tv.cn/806405.htm
wra.g6lz11tv.cn/313915.htm
wra.g6lz11tv.cn/371315.htm
wra.g6lz11tv.cn/791935.htm
wra.g6lz11tv.cn/820005.htm
wra.g6lz11tv.cn/357555.htm
wrp.g6lz11tv.cn/204805.htm
wrp.g6lz11tv.cn/068005.htm
wrp.g6lz11tv.cn/959395.htm
wrp.g6lz11tv.cn/640665.htm
wrp.g6lz11tv.cn/557735.htm
wrp.g6lz11tv.cn/228845.htm
wrp.g6lz11tv.cn/177355.htm
wrp.g6lz11tv.cn/624645.htm
wrp.g6lz11tv.cn/751375.htm
wrp.g6lz11tv.cn/222045.htm
wro.g6lz11tv.cn/242245.htm
wro.g6lz11tv.cn/224405.htm
wro.g6lz11tv.cn/668265.htm
wro.g6lz11tv.cn/593195.htm
wro.g6lz11tv.cn/002425.htm
wro.g6lz11tv.cn/626285.htm
wro.g6lz11tv.cn/919775.htm
wro.g6lz11tv.cn/226285.htm
wro.g6lz11tv.cn/222645.htm
wro.g6lz11tv.cn/488485.htm
wri.g6lz11tv.cn/840825.htm
wri.g6lz11tv.cn/626685.htm
wri.g6lz11tv.cn/864845.htm
wri.g6lz11tv.cn/442645.htm
wri.g6lz11tv.cn/557755.htm
wri.g6lz11tv.cn/484445.htm
wri.g6lz11tv.cn/331515.htm
wri.g6lz11tv.cn/242805.htm
wri.g6lz11tv.cn/048045.htm
wri.g6lz11tv.cn/199735.htm
wru.g6lz11tv.cn/462005.htm
wru.g6lz11tv.cn/373955.htm
wru.g6lz11tv.cn/204425.htm
wru.g6lz11tv.cn/642445.htm
wru.g6lz11tv.cn/040685.htm
wru.g6lz11tv.cn/642885.htm
wru.g6lz11tv.cn/060005.htm
wru.g6lz11tv.cn/042005.htm
wru.g6lz11tv.cn/842025.htm
wru.g6lz11tv.cn/286245.htm
wry.g6lz11tv.cn/391955.htm
wry.g6lz11tv.cn/604005.htm
wry.g6lz11tv.cn/022285.htm
wry.g6lz11tv.cn/082425.htm
wry.g6lz11tv.cn/795935.htm
wry.g6lz11tv.cn/422045.htm
wry.g6lz11tv.cn/022825.htm
wry.g6lz11tv.cn/042645.htm
wry.g6lz11tv.cn/959975.htm
wry.g6lz11tv.cn/004605.htm
wrt.g6lz11tv.cn/153735.htm
wrt.g6lz11tv.cn/288025.htm
wrt.g6lz11tv.cn/400465.htm
wrt.g6lz11tv.cn/822065.htm
wrt.g6lz11tv.cn/608665.htm
wrt.g6lz11tv.cn/882285.htm
wrt.g6lz11tv.cn/913555.htm
