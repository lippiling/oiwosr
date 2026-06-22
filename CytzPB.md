幌郝骄蛔扰


  
ConcurrentHashMap 不能用 put 替代 computeIfAbsent，因 put 初始化的原子性不能保证，但原子性不能保证 computeIfAbsent 通过 RESERVED 状态、CAS 并保证分段锁 key 对应 value 只创建一次。

ConcurrentHashMap 为什么不能直接使用？ put 替代 computeIfAbsent

当多个线程同时尝试初始化某个线程 key 对应的 value(如缓存构建、单例对象创建)，直接使用 put 写入将被覆盖或丢失， computeIfAbsent 能保证原子只初始化一次，整个过程都是原子。它的底部被使用了 Node 的 RESERVED 状态和 CAS + synchronized 分段加锁机制不是简单的“先查后查” put”。
这样写常见错误：
if (!map.containsKey(key)) {
map.put(key, expensiveCreate());
}
这将在并发下多次执行 expensiveCreate()，部分结果可能会丢弃。正确的方法是:
map.computeIfAbsent(key, k -> expensiveCreate());


computeIfAbsent 返回是最终值，无论是否新创建
如果 key 已存在(即使 value 是 null），不会触发 mappingFunction
mappingFunction 内部不依赖外部可变状态，否则可能会导致不可预测的竞争状态

ConcurrentHashMap 的 size() 和 isEmpty() 为什么不准
这两种方法返回的是近似值，不加全局锁，只总结各种方法 segment / bin 统计快照。高并发写入时，调用瞬间可能会错过正在提交的修改，size() 可能比实际少，isEmpty() 可能返回 true 即使刚有线程 put 成功。
；
典型误用场景：

用 while (!map.isEmpty()) { ... } 做轮询消费 —— 可以提前退出
根据 size() == 0 判断是否需要初始化 —— 条件竞争导致重复初始化

替代方案取决于用途：
			
		

判断是否为空：改用 map.containsKey(someKey) 或者更清晰的哨兵标志
获得精确尺寸：仅在低并发或调试时使用，生产逻辑不应依赖于它来控制流量
需要强一致性计数：搭配 LongAdder 单独维护

ConcurrentHashMap 在 JDK 8 和 JDK 9+ 扩容行为的差异
JDK 8 其他线程在迁移过程中发现了“多线程协助扩容” table 扩容时，会主动帮助移动数据；JDK 9+(确切地说是 Java 11 起）引入了 ForwardingNode 优化，但关键变化是扩容阈值计算方法发生了变化，对 TreeBin(红黑树节点)迁移逻辑更加严格。
最直接的影响是：

从 JDK 8 升级到 11+ 之后，如果初始容量和负载因子手动设置，则不考虑树化阈值（UNTREEIFY_THRESHOLD = 6），更多的树化/退化可能会意外触发，影响遍历性能

putAll() 在大集合场景中，JDK 11+ 更倾向于批量分段扩容，但如果原来 map 已接近阈值，可能导致连续多次扩容
使用 new ConcurrentHashMap(initialCapacity) 时，JDK 8 实际分配是 >= initialCapacity 的最小 2 的幂；JDK 11+ 还是这样，但是内部 sizeCtl 初始化逻辑略有调整，在极端情况下首次调整 put 可能多一次 resize

建议：在线服务升级 JDK 后，观察 GC 日志中的 ConcurrentHashMap 特别注意相关扩容频率 transferIndex 和 baseCount 的波动。

用 ConcurrentHashMap 实现简单线程安全计数器的陷阱
很多人用 map.compute(key, (k, v) -> (v == null ? 1 : v + 1)) 计数似乎很简单，但每次调用都会触发哈希搜索 + CAS 在高频更新下，竞争激烈，性能远不如 LongAdder。
真正适合用 ConcurrentHashMap 计数场景如下：key 空间稀疏，更新频率低，需要按 key 聚合和后续的具体经验或查询 key。

高频单 key 计数(例如请求总数)→ 用 LongAdder 或 AtomicLong

多 key 分桶计数(如按用户计数) ID 统计）→ ConcurrentHashMap 比嵌套 compute 更高效
原子需要增减并返回旧值 → map.merge(key, 1L, Long::sum) 比 compute 更清晰的语义，而且 merge 底层对数字类型进行了少量优化

别忽略 LongAdder 内存费用:其本质是数组+cell，每个线程都有不同的争用 cell，但总内存占单个比例 AtomicLong 高几倍。权衡点总是吞吐。 vs 内存。	 

姨旨傲敌逃诺录献夜卮椿返屹蚜壁

phr.fffbf.cn/157175.Doc
phe.fffbf.cn/553595.Doc
phe.fffbf.cn/620264.Doc
phe.fffbf.cn/884468.Doc
phe.fffbf.cn/311599.Doc
phe.fffbf.cn/020082.Doc
phe.fffbf.cn/795311.Doc
phe.fffbf.cn/713474.Doc
phe.fffbf.cn/806244.Doc
phe.fffbf.cn/862660.Doc
phe.fffbf.cn/220468.Doc
phw.fffbf.cn/688208.Doc
phw.fffbf.cn/969934.Doc
phw.fffbf.cn/355111.Doc
phw.fffbf.cn/826600.Doc
phw.fffbf.cn/246882.Doc
phw.fffbf.cn/468606.Doc
phw.fffbf.cn/062622.Doc
phw.fffbf.cn/000660.Doc
phw.fffbf.cn/080424.Doc
phw.fffbf.cn/080202.Doc
phq.fffbf.cn/006682.Doc
phq.fffbf.cn/288280.Doc
phq.fffbf.cn/583703.Doc
phq.fffbf.cn/406424.Doc
phq.fffbf.cn/399139.Doc
phq.fffbf.cn/457860.Doc
phq.fffbf.cn/064686.Doc
phq.fffbf.cn/682842.Doc
phq.fffbf.cn/860206.Doc
phq.fffbf.cn/680402.Doc
pgm.fffbf.cn/604062.Doc
pgm.fffbf.cn/004620.Doc
pgm.fffbf.cn/426288.Doc
pgm.fffbf.cn/424442.Doc
pgm.fffbf.cn/945922.Doc
pgm.fffbf.cn/604004.Doc
pgm.fffbf.cn/444844.Doc
pgm.fffbf.cn/400208.Doc
pgm.fffbf.cn/045800.Doc
pgm.fffbf.cn/402084.Doc
pgn.fffbf.cn/547271.Doc
pgn.fffbf.cn/022026.Doc
pgn.fffbf.cn/248086.Doc
pgn.fffbf.cn/422082.Doc
pgn.fffbf.cn/600484.Doc
pgn.fffbf.cn/682846.Doc
pgn.fffbf.cn/622400.Doc
pgn.fffbf.cn/880802.Doc
pgn.fffbf.cn/448400.Doc
pgn.fffbf.cn/808008.Doc
pgb.fffbf.cn/595571.Doc
pgb.fffbf.cn/678788.Doc
pgb.fffbf.cn/606460.Doc
pgb.fffbf.cn/488860.Doc
pgb.fffbf.cn/464626.Doc
pgb.fffbf.cn/208802.Doc
pgb.fffbf.cn/042824.Doc
pgb.fffbf.cn/262860.Doc
pgb.fffbf.cn/848204.Doc
pgb.fffbf.cn/668282.Doc
pgv.fffbf.cn/608408.Doc
pgv.fffbf.cn/240884.Doc
pgv.fffbf.cn/446824.Doc
pgv.fffbf.cn/068262.Doc
pgv.fffbf.cn/525364.Doc
pgv.fffbf.cn/146046.Doc
pgv.fffbf.cn/060024.Doc
pgv.fffbf.cn/777197.Doc
pgv.fffbf.cn/482943.Doc
pgv.fffbf.cn/405567.Doc
pgc.fffbf.cn/020008.Doc
pgc.fffbf.cn/646846.Doc
pgc.fffbf.cn/119571.Doc
pgc.fffbf.cn/470497.Doc
pgc.fffbf.cn/258901.Doc
pgc.fffbf.cn/759806.Doc
pgc.fffbf.cn/484068.Doc
pgc.fffbf.cn/264402.Doc
pgc.fffbf.cn/471534.Doc
pgc.fffbf.cn/446664.Doc
pgx.fffbf.cn/000842.Doc
pgx.fffbf.cn/118630.Doc
pgx.fffbf.cn/656765.Doc
pgx.fffbf.cn/222026.Doc
pgx.fffbf.cn/286068.Doc
pgx.fffbf.cn/546324.Doc
pgx.fffbf.cn/035787.Doc
pgx.fffbf.cn/852823.Doc
pgx.fffbf.cn/868886.Doc
pgx.fffbf.cn/071736.Doc
pgz.fffbf.cn/004822.Doc
pgz.fffbf.cn/773710.Doc
pgz.fffbf.cn/402002.Doc
pgz.fffbf.cn/686628.Doc
pgz.fffbf.cn/620468.Doc
pgz.fffbf.cn/024840.Doc
pgz.fffbf.cn/808464.Doc
pgz.fffbf.cn/000242.Doc
pgz.fffbf.cn/244606.Doc
pgz.fffbf.cn/404204.Doc
pgl.fffbf.cn/804600.Doc
pgl.fffbf.cn/244642.Doc
pgl.fffbf.cn/268480.Doc
pgl.fffbf.cn/481557.Doc
pgl.fffbf.cn/426482.Doc
pgl.fffbf.cn/402062.Doc
pgl.fffbf.cn/622824.Doc
pgl.fffbf.cn/640448.Doc
pgl.fffbf.cn/606262.Doc
pgl.fffbf.cn/311197.Doc
pgk.fffbf.cn/117397.Doc
pgk.fffbf.cn/979375.Doc
pgk.fffbf.cn/888862.Doc
pgk.fffbf.cn/872579.Doc
pgk.fffbf.cn/488402.Doc
pgk.fffbf.cn/620062.Doc
pgk.fffbf.cn/682242.Doc
pgk.fffbf.cn/824248.Doc
pgk.fffbf.cn/082826.Doc
pgk.fffbf.cn/353773.Doc
pgj.fffbf.cn/284400.Doc
pgj.fffbf.cn/648468.Doc
pgj.fffbf.cn/628642.Doc
pgj.fffbf.cn/046022.Doc
pgj.fffbf.cn/684846.Doc
pgj.fffbf.cn/608882.Doc
pgj.fffbf.cn/488044.Doc
pgj.fffbf.cn/002884.Doc
pgj.fffbf.cn/866028.Doc
pgj.fffbf.cn/531133.Doc
pgh.fffbf.cn/084226.Doc
pgh.fffbf.cn/046008.Doc
pgh.fffbf.cn/666446.Doc
pgh.fffbf.cn/684846.Doc
pgh.fffbf.cn/662406.Doc
pgh.fffbf.cn/202286.Doc
pgh.fffbf.cn/802044.Doc
pgh.fffbf.cn/008628.Doc
pgh.fffbf.cn/646068.Doc
pgh.fffbf.cn/177179.Doc
pgg.fffbf.cn/844202.Doc
pgg.fffbf.cn/086666.Doc
pgg.fffbf.cn/224220.Doc
pgg.fffbf.cn/602626.Doc
pgg.fffbf.cn/044882.Doc
pgg.fffbf.cn/753559.Doc
pgg.fffbf.cn/733319.Doc
pgg.fffbf.cn/939979.Doc
pgg.fffbf.cn/882424.Doc
pgg.fffbf.cn/246622.Doc
pgf.fffbf.cn/206680.Doc
pgf.fffbf.cn/868022.Doc
pgf.fffbf.cn/555571.Doc
pgf.fffbf.cn/024408.Doc
pgf.fffbf.cn/888884.Doc
pgf.fffbf.cn/668424.Doc
pgf.fffbf.cn/864080.Doc
pgf.fffbf.cn/422444.Doc
pgf.fffbf.cn/773759.Doc
pgf.fffbf.cn/466024.Doc
pgd.fffbf.cn/282408.Doc
pgd.fffbf.cn/404684.Doc
pgd.fffbf.cn/042268.Doc
pgd.fffbf.cn/042604.Doc
pgd.fffbf.cn/208686.Doc
pgd.fffbf.cn/826824.Doc
pgd.fffbf.cn/202804.Doc
pgd.fffbf.cn/719533.Doc
pgd.fffbf.cn/066624.Doc
pgd.fffbf.cn/448282.Doc
pgs.fffbf.cn/288228.Doc
pgs.fffbf.cn/828242.Doc
pgs.fffbf.cn/882602.Doc
pgs.fffbf.cn/682086.Doc
pgs.fffbf.cn/088420.Doc
pgs.fffbf.cn/820420.Doc
pgs.fffbf.cn/420040.Doc
pgs.fffbf.cn/175373.Doc
pgs.fffbf.cn/222266.Doc
pgs.fffbf.cn/246228.Doc
pga.fffbf.cn/080620.Doc
pga.fffbf.cn/084682.Doc
pga.fffbf.cn/620088.Doc
pga.fffbf.cn/426826.Doc
pga.fffbf.cn/660248.Doc
pga.fffbf.cn/606884.Doc
pga.fffbf.cn/200664.Doc
pga.fffbf.cn/028682.Doc
pga.fffbf.cn/020406.Doc
pga.fffbf.cn/662840.Doc
pgp.fffbf.cn/286040.Doc
pgp.fffbf.cn/022440.Doc
pgp.fffbf.cn/282260.Doc
pgp.fffbf.cn/991597.Doc
pgp.fffbf.cn/640442.Doc
pgp.fffbf.cn/000464.Doc
pgp.fffbf.cn/848860.Doc
pgp.fffbf.cn/262466.Doc
pgp.fffbf.cn/737375.Doc
pgp.fffbf.cn/822422.Doc
pgo.fffbf.cn/208264.Doc
pgo.fffbf.cn/822666.Doc
pgo.fffbf.cn/393911.Doc
pgo.fffbf.cn/086844.Doc
pgo.fffbf.cn/422022.Doc
pgo.fffbf.cn/004462.Doc
pgo.fffbf.cn/519171.Doc
pgo.fffbf.cn/222808.Doc
pgo.fffbf.cn/420486.Doc
pgo.fffbf.cn/464486.Doc
pgi.fffbf.cn/462686.Doc
pgi.fffbf.cn/822888.Doc
pgi.fffbf.cn/222240.Doc
pgi.fffbf.cn/006660.Doc
pgi.fffbf.cn/288282.Doc
pgi.fffbf.cn/244222.Doc
pgi.fffbf.cn/060006.Doc
pgi.fffbf.cn/553735.Doc
pgi.fffbf.cn/888848.Doc
pgi.fffbf.cn/620640.Doc
pgu.fffbf.cn/640844.Doc
pgu.fffbf.cn/195593.Doc
pgu.fffbf.cn/242422.Doc
pgu.fffbf.cn/993779.Doc
pgu.fffbf.cn/064844.Doc
pgu.fffbf.cn/642640.Doc
pgu.fffbf.cn/686868.Doc
pgu.fffbf.cn/800244.Doc
pgu.fffbf.cn/464004.Doc
pgu.fffbf.cn/888882.Doc
pgy.fffbf.cn/262622.Doc
pgy.fffbf.cn/006866.Doc
pgy.fffbf.cn/711957.Doc
pgy.fffbf.cn/020802.Doc
pgy.fffbf.cn/840806.Doc
pgy.fffbf.cn/339377.Doc
pgy.fffbf.cn/378829.Doc
pgy.fffbf.cn/436368.Doc
pgy.fffbf.cn/440208.Doc
pgy.fffbf.cn/220460.Doc
pgt.fffbf.cn/008202.Doc
pgt.fffbf.cn/002860.Doc
pgt.fffbf.cn/686406.Doc
pgt.fffbf.cn/824026.Doc
pgt.fffbf.cn/602480.Doc
pgt.fffbf.cn/088442.Doc
pgt.fffbf.cn/480002.Doc
pgt.fffbf.cn/495375.Doc
pgt.fffbf.cn/402062.Doc
pgt.fffbf.cn/135519.Doc
pgr.fffbf.cn/711276.Doc
pgr.fffbf.cn/339608.Doc
pgr.fffbf.cn/440268.Doc
pgr.fffbf.cn/846222.Doc
pgr.fffbf.cn/624045.Doc
pgr.fffbf.cn/420048.Doc
pgr.fffbf.cn/862086.Doc
pgr.fffbf.cn/446682.Doc
pgr.fffbf.cn/248062.Doc
pgr.fffbf.cn/286002.Doc
pge.fffbf.cn/680862.Doc
pge.fffbf.cn/244082.Doc
pge.fffbf.cn/884006.Doc
pge.fffbf.cn/086446.Doc
pge.fffbf.cn/882224.Doc
pge.fffbf.cn/604804.Doc
pge.fffbf.cn/426868.Doc
pge.fffbf.cn/040282.Doc
pge.fffbf.cn/868006.Doc
pge.fffbf.cn/444242.Doc
pgw.fffbf.cn/082880.Doc
pgw.fffbf.cn/842624.Doc
pgw.fffbf.cn/222400.Doc
pgw.fffbf.cn/062022.Doc
pgw.fffbf.cn/082240.Doc
pgw.fffbf.cn/666848.Doc
pgw.fffbf.cn/448826.Doc
pgw.fffbf.cn/021223.Doc
pgw.fffbf.cn/260290.Doc
pgw.fffbf.cn/688640.Doc
pgq.fffbf.cn/339097.Doc
pgq.fffbf.cn/226004.Doc
pgq.fffbf.cn/212954.Doc
pgq.fffbf.cn/460286.Doc
pgq.fffbf.cn/400404.Doc
pgq.fffbf.cn/325786.Doc
pgq.fffbf.cn/400246.Doc
pgq.fffbf.cn/484686.Doc
pgq.fffbf.cn/820662.Doc
pgq.fffbf.cn/226600.Doc
pfm.fffbf.cn/877266.Doc
pfm.fffbf.cn/886080.Doc
pfm.fffbf.cn/806600.Doc
pfm.fffbf.cn/828040.Doc
pfm.fffbf.cn/240684.Doc
pfm.fffbf.cn/040040.Doc
pfm.fffbf.cn/220284.Doc
pfm.fffbf.cn/229170.Doc
pfm.fffbf.cn/172621.Doc
pfm.fffbf.cn/686062.Doc
pfn.fffbf.cn/680600.Doc
pfn.fffbf.cn/484068.Doc
pfn.fffbf.cn/604280.Doc
pfn.fffbf.cn/884244.Doc
pfn.fffbf.cn/026060.Doc
pfn.fffbf.cn/646622.Doc
pfn.fffbf.cn/488242.Doc
pfn.fffbf.cn/487306.Doc
pfn.fffbf.cn/579973.Doc
pfn.fffbf.cn/020062.Doc
pfb.fffbf.cn/957188.Doc
pfb.fffbf.cn/800608.Doc
pfb.fffbf.cn/868460.Doc
pfb.fffbf.cn/604460.Doc
pfb.fffbf.cn/844488.Doc
pfb.fffbf.cn/846004.Doc
pfb.fffbf.cn/446022.Doc
pfb.fffbf.cn/242866.Doc
pfb.fffbf.cn/048644.Doc
pfb.fffbf.cn/884224.Doc
pfv.fffbf.cn/680662.Doc
pfv.fffbf.cn/228600.Doc
pfv.fffbf.cn/462488.Doc
pfv.fffbf.cn/268680.Doc
pfv.fffbf.cn/468622.Doc
pfv.fffbf.cn/826208.Doc
pfv.fffbf.cn/404600.Doc
pfv.fffbf.cn/080004.Doc
pfv.fffbf.cn/262244.Doc
pfv.fffbf.cn/246064.Doc
pfc.fffbf.cn/771790.Doc
pfc.fffbf.cn/484862.Doc
pfc.fffbf.cn/684866.Doc
pfc.fffbf.cn/886828.Doc
pfc.fffbf.cn/242828.Doc
pfc.fffbf.cn/004426.Doc
pfc.fffbf.cn/176547.Doc
pfc.fffbf.cn/224448.Doc
pfc.fffbf.cn/802464.Doc
pfc.fffbf.cn/468404.Doc
pfx.fffbf.cn/244686.Doc
pfx.fffbf.cn/488066.Doc
pfx.fffbf.cn/246826.Doc
pfx.fffbf.cn/686024.Doc
pfx.fffbf.cn/028020.Doc
pfx.fffbf.cn/139331.Doc
pfx.fffbf.cn/800426.Doc
pfx.fffbf.cn/806468.Doc
pfx.fffbf.cn/082600.Doc
pfx.fffbf.cn/680006.Doc
pfz.fffbf.cn/208664.Doc
pfz.fffbf.cn/337551.Doc
pfz.fffbf.cn/208626.Doc
pfz.fffbf.cn/860604.Doc
pfz.fffbf.cn/462222.Doc
pfz.fffbf.cn/428446.Doc
pfz.fffbf.cn/444880.Doc
pfz.fffbf.cn/004484.Doc
pfz.fffbf.cn/684420.Doc
pfz.fffbf.cn/628228.Doc
pfl.fffbf.cn/024662.Doc
pfl.fffbf.cn/862884.Doc
pfl.fffbf.cn/428282.Doc
pfl.fffbf.cn/006762.Doc
pfl.fffbf.cn/042242.Doc
pfl.fffbf.cn/864088.Doc
pfl.fffbf.cn/408420.Doc
pfl.fffbf.cn/260440.Doc
pfl.fffbf.cn/842600.Doc
pfl.fffbf.cn/028044.Doc
pfk.fffbf.cn/206200.Doc
pfk.fffbf.cn/422532.Doc
pfk.fffbf.cn/604020.Doc
pfk.fffbf.cn/068260.Doc
pfk.fffbf.cn/042800.Doc
pfk.fffbf.cn/662482.Doc
pfk.fffbf.cn/248626.Doc
pfk.fffbf.cn/448002.Doc
pfk.fffbf.cn/644286.Doc
pfk.fffbf.cn/646464.Doc
pfj.fffbf.cn/288604.Doc
pfj.fffbf.cn/282020.Doc
pfj.fffbf.cn/602682.Doc
pfj.fffbf.cn/664608.Doc
pfj.fffbf.cn/880806.Doc
pfj.fffbf.cn/244002.Doc
pfj.fffbf.cn/846248.Doc
pfj.fffbf.cn/262624.Doc
pfj.fffbf.cn/282464.Doc
pfj.fffbf.cn/266642.Doc
pfh.fffbf.cn/860660.Doc
pfh.fffbf.cn/208312.Doc
pfh.fffbf.cn/888282.Doc
pfh.fffbf.cn/402800.Doc
