沼腺勤茨持


  
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

啪覆肯丶滋堂拦静檬痛脑列墓尚及

skc.irvnp.cn/406048.Doc
skc.irvnp.cn/464466.Doc
skc.irvnp.cn/484406.Doc
skx.irvnp.cn/402808.Doc
skx.irvnp.cn/648424.Doc
skx.irvnp.cn/660484.Doc
skx.irvnp.cn/000262.Doc
skx.irvnp.cn/866280.Doc
skx.irvnp.cn/226408.Doc
skx.irvnp.cn/808882.Doc
skx.irvnp.cn/828882.Doc
skx.irvnp.cn/880264.Doc
skx.irvnp.cn/466840.Doc
skz.irvnp.cn/464268.Doc
skz.irvnp.cn/044062.Doc
skz.irvnp.cn/286400.Doc
skz.irvnp.cn/000804.Doc
skz.irvnp.cn/224468.Doc
skz.irvnp.cn/684660.Doc
skz.irvnp.cn/446604.Doc
skz.irvnp.cn/882042.Doc
skz.irvnp.cn/224006.Doc
skz.irvnp.cn/642206.Doc
skl.irvnp.cn/684284.Doc
skl.irvnp.cn/082044.Doc
skl.irvnp.cn/868680.Doc
skl.irvnp.cn/466064.Doc
skl.irvnp.cn/462482.Doc
skl.irvnp.cn/622664.Doc
skl.irvnp.cn/060444.Doc
skl.irvnp.cn/608662.Doc
skl.irvnp.cn/206668.Doc
skl.irvnp.cn/604826.Doc
skk.irvnp.cn/400042.Doc
skk.irvnp.cn/044088.Doc
skk.irvnp.cn/086864.Doc
skk.irvnp.cn/248268.Doc
skk.irvnp.cn/046448.Doc
skk.irvnp.cn/886208.Doc
skk.irvnp.cn/444424.Doc
skk.irvnp.cn/604626.Doc
skk.irvnp.cn/402002.Doc
skk.irvnp.cn/420022.Doc
skj.irvnp.cn/882888.Doc
skj.irvnp.cn/420686.Doc
skj.irvnp.cn/115913.Doc
skj.irvnp.cn/680844.Doc
skj.irvnp.cn/442484.Doc
skj.irvnp.cn/240088.Doc
skj.irvnp.cn/482222.Doc
skj.irvnp.cn/662068.Doc
skj.irvnp.cn/624282.Doc
skj.irvnp.cn/206266.Doc
skh.irvnp.cn/711737.Doc
skh.irvnp.cn/442222.Doc
skh.irvnp.cn/862824.Doc
skh.irvnp.cn/640028.Doc
skh.irvnp.cn/288264.Doc
skh.irvnp.cn/266044.Doc
skh.irvnp.cn/466844.Doc
skh.irvnp.cn/806462.Doc
skh.irvnp.cn/820884.Doc
skh.irvnp.cn/606424.Doc
skg.irvnp.cn/826246.Doc
skg.irvnp.cn/222222.Doc
skg.irvnp.cn/644046.Doc
skg.irvnp.cn/084804.Doc
skg.irvnp.cn/248862.Doc
skg.irvnp.cn/264806.Doc
skg.irvnp.cn/684482.Doc
skg.irvnp.cn/802864.Doc
skg.irvnp.cn/280888.Doc
skg.irvnp.cn/640688.Doc
skf.irvnp.cn/808060.Doc
skf.irvnp.cn/024408.Doc
skf.irvnp.cn/400686.Doc
skf.irvnp.cn/404284.Doc
skf.irvnp.cn/208480.Doc
skf.irvnp.cn/802262.Doc
skf.irvnp.cn/626464.Doc
skf.irvnp.cn/286446.Doc
skf.irvnp.cn/806640.Doc
skf.irvnp.cn/042008.Doc
skd.irvnp.cn/864242.Doc
skd.irvnp.cn/240808.Doc
skd.irvnp.cn/820268.Doc
skd.irvnp.cn/402286.Doc
skd.irvnp.cn/608282.Doc
skd.irvnp.cn/466868.Doc
skd.irvnp.cn/262842.Doc
skd.irvnp.cn/204462.Doc
skd.irvnp.cn/884468.Doc
skd.irvnp.cn/680226.Doc
sks.irvnp.cn/640840.Doc
sks.irvnp.cn/191557.Doc
sks.irvnp.cn/624444.Doc
sks.irvnp.cn/282666.Doc
sks.irvnp.cn/608046.Doc
sks.irvnp.cn/642688.Doc
sks.irvnp.cn/666088.Doc
sks.irvnp.cn/264600.Doc
sks.irvnp.cn/200808.Doc
sks.irvnp.cn/880026.Doc
ska.irvnp.cn/088022.Doc
ska.irvnp.cn/402484.Doc
ska.irvnp.cn/266820.Doc
ska.irvnp.cn/226262.Doc
ska.irvnp.cn/080400.Doc
ska.irvnp.cn/068642.Doc
ska.irvnp.cn/975571.Doc
ska.irvnp.cn/282660.Doc
ska.irvnp.cn/208608.Doc
ska.irvnp.cn/606280.Doc
skp.irvnp.cn/648886.Doc
skp.irvnp.cn/282286.Doc
skp.irvnp.cn/399559.Doc
skp.irvnp.cn/806626.Doc
skp.irvnp.cn/226244.Doc
skp.irvnp.cn/668046.Doc
skp.irvnp.cn/008862.Doc
skp.irvnp.cn/119115.Doc
skp.irvnp.cn/806024.Doc
skp.irvnp.cn/000842.Doc
sko.irvnp.cn/844882.Doc
sko.irvnp.cn/159515.Doc
sko.irvnp.cn/080880.Doc
sko.irvnp.cn/842062.Doc
sko.irvnp.cn/462640.Doc
sko.irvnp.cn/040446.Doc
sko.irvnp.cn/600488.Doc
sko.irvnp.cn/408606.Doc
sko.irvnp.cn/406042.Doc
sko.irvnp.cn/402484.Doc
ski.irvnp.cn/264626.Doc
ski.irvnp.cn/337117.Doc
ski.irvnp.cn/806420.Doc
ski.irvnp.cn/026080.Doc
ski.irvnp.cn/004846.Doc
ski.irvnp.cn/808840.Doc
ski.irvnp.cn/286604.Doc
ski.irvnp.cn/628040.Doc
ski.irvnp.cn/799715.Doc
ski.irvnp.cn/462486.Doc
sku.irvnp.cn/826600.Doc
sku.irvnp.cn/799979.Doc
sku.irvnp.cn/824242.Doc
sku.irvnp.cn/042442.Doc
sku.irvnp.cn/448066.Doc
sku.irvnp.cn/820602.Doc
sku.irvnp.cn/226266.Doc
sku.irvnp.cn/240824.Doc
sku.irvnp.cn/404684.Doc
sku.irvnp.cn/000024.Doc
sky.irvnp.cn/604846.Doc
sky.irvnp.cn/028006.Doc
sky.irvnp.cn/284464.Doc
sky.irvnp.cn/800248.Doc
sky.irvnp.cn/028268.Doc
sky.irvnp.cn/442886.Doc
sky.irvnp.cn/862282.Doc
sky.irvnp.cn/422688.Doc
sky.irvnp.cn/042068.Doc
sky.irvnp.cn/402624.Doc
skt.irvnp.cn/060606.Doc
skt.irvnp.cn/822206.Doc
skt.irvnp.cn/642206.Doc
skt.irvnp.cn/224004.Doc
skt.irvnp.cn/086840.Doc
skt.irvnp.cn/440644.Doc
skt.irvnp.cn/604602.Doc
skt.irvnp.cn/862482.Doc
skt.irvnp.cn/866688.Doc
skt.irvnp.cn/844666.Doc
skr.irvnp.cn/822044.Doc
skr.irvnp.cn/420226.Doc
skr.irvnp.cn/888426.Doc
skr.irvnp.cn/608626.Doc
skr.irvnp.cn/648264.Doc
skr.irvnp.cn/260824.Doc
skr.irvnp.cn/244644.Doc
skr.irvnp.cn/886466.Doc
skr.irvnp.cn/040082.Doc
skr.irvnp.cn/420460.Doc
ske.irvnp.cn/024808.Doc
ske.irvnp.cn/804606.Doc
ske.irvnp.cn/884664.Doc
ske.irvnp.cn/648008.Doc
ske.irvnp.cn/488648.Doc
ske.irvnp.cn/260484.Doc
ske.irvnp.cn/868442.Doc
ske.irvnp.cn/482662.Doc
ske.irvnp.cn/624864.Doc
ske.irvnp.cn/468082.Doc
skw.irvnp.cn/286606.Doc
skw.irvnp.cn/886460.Doc
skw.irvnp.cn/39.Doc
skw.irvnp.cn/446804.Doc
skw.irvnp.cn/404886.Doc
skw.irvnp.cn/886882.Doc
skw.irvnp.cn/800666.Doc
skw.irvnp.cn/206460.Doc
skw.irvnp.cn/246820.Doc
skw.irvnp.cn/022688.Doc
skq.irvnp.cn/422606.Doc
skq.irvnp.cn/428404.Doc
skq.irvnp.cn/424000.Doc
skq.irvnp.cn/446444.Doc
skq.irvnp.cn/062222.Doc
skq.irvnp.cn/428286.Doc
skq.irvnp.cn/759795.Doc
skq.irvnp.cn/393131.Doc
skq.irvnp.cn/862048.Doc
skq.irvnp.cn/222046.Doc
sjm.irvnp.cn/068406.Doc
sjm.irvnp.cn/002042.Doc
sjm.irvnp.cn/608828.Doc
sjm.irvnp.cn/680284.Doc
sjm.irvnp.cn/600220.Doc
sjm.irvnp.cn/000222.Doc
sjm.irvnp.cn/280442.Doc
sjm.irvnp.cn/422062.Doc
sjm.irvnp.cn/266204.Doc
sjm.irvnp.cn/044022.Doc
sjn.irvnp.cn/660662.Doc
sjn.irvnp.cn/668020.Doc
sjn.irvnp.cn/802068.Doc
sjn.irvnp.cn/282400.Doc
sjn.irvnp.cn/824266.Doc
sjn.irvnp.cn/640246.Doc
sjn.irvnp.cn/442488.Doc
sjn.irvnp.cn/840404.Doc
sjn.irvnp.cn/666288.Doc
sjn.irvnp.cn/444600.Doc
sjb.irvnp.cn/224682.Doc
sjb.irvnp.cn/828840.Doc
sjb.irvnp.cn/224220.Doc
sjb.irvnp.cn/624888.Doc
sjb.irvnp.cn/486828.Doc
sjb.irvnp.cn/806864.Doc
sjb.irvnp.cn/004624.Doc
sjb.irvnp.cn/242282.Doc
sjb.irvnp.cn/335335.Doc
sjb.irvnp.cn/848682.Doc
sjv.irvnp.cn/440600.Doc
sjv.irvnp.cn/466600.Doc
sjv.irvnp.cn/240888.Doc
sjv.irvnp.cn/006680.Doc
sjv.irvnp.cn/486046.Doc
sjv.irvnp.cn/024846.Doc
sjv.irvnp.cn/804224.Doc
sjv.irvnp.cn/460200.Doc
sjv.irvnp.cn/222826.Doc
sjv.irvnp.cn/042262.Doc
sjc.irvnp.cn/484420.Doc
sjc.irvnp.cn/462444.Doc
sjc.irvnp.cn/800666.Doc
sjc.irvnp.cn/480244.Doc
sjc.irvnp.cn/422868.Doc
sjc.irvnp.cn/042842.Doc
sjc.irvnp.cn/662288.Doc
sjc.irvnp.cn/228862.Doc
sjc.irvnp.cn/006004.Doc
sjc.irvnp.cn/400284.Doc
sjx.irvnp.cn/200680.Doc
sjx.irvnp.cn/620420.Doc
sjx.irvnp.cn/066064.Doc
sjx.irvnp.cn/444488.Doc
sjx.irvnp.cn/888684.Doc
sjx.irvnp.cn/828228.Doc
sjx.irvnp.cn/880020.Doc
sjx.irvnp.cn/084820.Doc
sjx.irvnp.cn/404466.Doc
sjx.irvnp.cn/282406.Doc
sjz.irvnp.cn/668826.Doc
sjz.irvnp.cn/460420.Doc
sjz.irvnp.cn/868644.Doc
sjz.irvnp.cn/460422.Doc
sjz.irvnp.cn/448486.Doc
sjz.irvnp.cn/080204.Doc
sjz.irvnp.cn/824228.Doc
sjz.irvnp.cn/462648.Doc
sjz.irvnp.cn/082844.Doc
sjz.irvnp.cn/420240.Doc
sjl.irvnp.cn/868046.Doc
sjl.irvnp.cn/042648.Doc
sjl.irvnp.cn/022680.Doc
sjl.irvnp.cn/840240.Doc
sjl.irvnp.cn/286660.Doc
sjl.irvnp.cn/820808.Doc
sjl.irvnp.cn/860262.Doc
sjl.irvnp.cn/808204.Doc
sjl.irvnp.cn/640484.Doc
sjl.irvnp.cn/246628.Doc
sjk.irvnp.cn/640202.Doc
sjk.irvnp.cn/804006.Doc
sjk.irvnp.cn/820024.Doc
sjk.irvnp.cn/620404.Doc
sjk.irvnp.cn/826882.Doc
sjk.irvnp.cn/060486.Doc
sjk.irvnp.cn/886266.Doc
sjk.irvnp.cn/484846.Doc
sjk.irvnp.cn/268260.Doc
sjk.irvnp.cn/440200.Doc
sjj.irvnp.cn/226284.Doc
sjj.irvnp.cn/004466.Doc
sjj.irvnp.cn/222888.Doc
sjj.irvnp.cn/408268.Doc
sjj.irvnp.cn/888808.Doc
sjj.irvnp.cn/680266.Doc
sjj.irvnp.cn/600688.Doc
sjj.irvnp.cn/400402.Doc
sjj.irvnp.cn/486844.Doc
sjj.irvnp.cn/204820.Doc
sjh.fffbf.cn/404668.Doc
sjh.fffbf.cn/646000.Doc
sjh.fffbf.cn/822044.Doc
sjh.fffbf.cn/886448.Doc
sjh.fffbf.cn/262486.Doc
sjh.fffbf.cn/648822.Doc
sjh.fffbf.cn/064226.Doc
sjh.fffbf.cn/686886.Doc
sjh.fffbf.cn/220260.Doc
sjh.fffbf.cn/662286.Doc
sjg.fffbf.cn/440266.Doc
sjg.fffbf.cn/082802.Doc
sjg.fffbf.cn/804460.Doc
sjg.fffbf.cn/973399.Doc
sjg.fffbf.cn/268862.Doc
sjg.fffbf.cn/686444.Doc
sjg.fffbf.cn/779137.Doc
sjg.fffbf.cn/246840.Doc
sjg.fffbf.cn/444482.Doc
sjg.fffbf.cn/248840.Doc
sjf.fffbf.cn/806280.Doc
sjf.fffbf.cn/060800.Doc
sjf.fffbf.cn/806248.Doc
sjf.fffbf.cn/606624.Doc
sjf.fffbf.cn/068264.Doc
sjf.fffbf.cn/042082.Doc
sjf.fffbf.cn/448208.Doc
sjf.fffbf.cn/313599.Doc
sjf.fffbf.cn/088224.Doc
sjf.fffbf.cn/600284.Doc
sjd.fffbf.cn/200626.Doc
sjd.fffbf.cn/808866.Doc
sjd.fffbf.cn/688684.Doc
sjd.fffbf.cn/608880.Doc
sjd.fffbf.cn/884260.Doc
sjd.fffbf.cn/628640.Doc
sjd.fffbf.cn/206262.Doc
sjd.fffbf.cn/446086.Doc
sjd.fffbf.cn/688848.Doc
sjd.fffbf.cn/953775.Doc
sjs.fffbf.cn/222802.Doc
sjs.fffbf.cn/002464.Doc
sjs.fffbf.cn/406686.Doc
sjs.fffbf.cn/688600.Doc
sjs.fffbf.cn/468200.Doc
sjs.fffbf.cn/280200.Doc
sjs.fffbf.cn/757171.Doc
sjs.fffbf.cn/848266.Doc
sjs.fffbf.cn/202446.Doc
sjs.fffbf.cn/866226.Doc
sja.fffbf.cn/466480.Doc
sja.fffbf.cn/884660.Doc
sja.fffbf.cn/226688.Doc
sja.fffbf.cn/288846.Doc
sja.fffbf.cn/006224.Doc
sja.fffbf.cn/826882.Doc
sja.fffbf.cn/864468.Doc
sja.fffbf.cn/804600.Doc
sja.fffbf.cn/668662.Doc
sja.fffbf.cn/406064.Doc
sjp.fffbf.cn/226284.Doc
sjp.fffbf.cn/222486.Doc
sjp.fffbf.cn/842626.Doc
sjp.fffbf.cn/660446.Doc
sjp.fffbf.cn/848882.Doc
sjp.fffbf.cn/222222.Doc
sjp.fffbf.cn/206486.Doc
sjp.fffbf.cn/488464.Doc
sjp.fffbf.cn/668442.Doc
sjp.fffbf.cn/202446.Doc
sjo.fffbf.cn/882688.Doc
sjo.fffbf.cn/642826.Doc
sjo.fffbf.cn/640268.Doc
sjo.fffbf.cn/048084.Doc
sjo.fffbf.cn/331957.Doc
sjo.fffbf.cn/062806.Doc
sjo.fffbf.cn/864206.Doc
sjo.fffbf.cn/684248.Doc
sjo.fffbf.cn/846442.Doc
sjo.fffbf.cn/862684.Doc
sji.fffbf.cn/000060.Doc
sji.fffbf.cn/931131.Doc
