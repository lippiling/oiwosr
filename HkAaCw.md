恼罢自桓恼


  
Guava不可变集合无public结构器，必须通过静态工厂创建，如Immutablelistt等.of()或copyof()，修改操作抛unsupportedoperationexception；Multimap简化嵌套列表管理；Stream直接collecttttam(ImmutableList.toImmutableList())；JDK21Imutablelist没有实现 SequencedCollection。

ImmutableList、ImmutableSet 为什么不能直接？ new
Guava 不可变集合不依赖 final 装饰的普通集合，而是全新结构的只读实现。你写的 new ImmutableList() 会编译失败-它没有 public 构造器。
正确的入口只有静态工厂方法，核心是「构建即冻结」逻辑:一旦生成，任何修改操作(如 add()、remove()）都会抛 UnsupportedOperationException。

用 ImmutableList.of(&quot;a&quot;, &quot;b&quot;) 最多创建小固定集合(创建小固定集合( 5 个元素）
超过 5 个或需要从现有集合转换，使用 ImmutableList.copyOf(list)；注意传入 null 会直接 NPE，建议先判空

ImmutableSet.copyOf(collection) 自动去除重复元素，但不保证顺序(底层使用) ImmutableHashSet），要有序用 ImmutableSortedSet


Multimap 代替 Map> 的真实代价

写 Map&lt;string list&gt;&gt;&lt;/string&gt; 手动管理嵌套列表非常麻烦：每次 put 前得检查 key 是否存在、list 是空的，是否想要 new ArrayList。而 Multimap 包装这件事，但代价是它不继承 Map 接口，不能直接做 Map 用。

常用的是 ArrayListMultimap.create()(值按插入顺序)和 HashMultimap.create()(值无序，内存更省)

put(&quot;k1&quot;, 1) 和 put(&quot;k1&quot;, 2) 后，get(&quot;k1&quot;) 返回是不可修改的 Collection 视图，不能调整 clear() 或 remove() —— 想删除一个值得使用的 remove(&quot;k1&quot;, 1)

若需要支持 null key 或 null value，必须选 LinkedHashMultimap 或显式 wrap，因为默认禁止 null

ImmutableList.copyOf() 流式处理中容易误踩坑
很多人在 Stream 后接 collect(Collectors.toList())，再传给 ImmutableList.copyOf()，认为双重保险。事实上，没有必要再经历一次。


；

Stream 本身有 toList()（Java 16+)，返回是不可变的 List，但它是 JDK 实现，和 Guava 无关；Guava 的 ImmutableList.toImmutableList() 是配套收集器
错误写法：stream.collect(Collectors.toList()).stream().collect(ImmutableList.toImmutableList()) —— 白跑两轮
正确写法：stream.collect(ImmutableList.toImmutableList())，而且收集器内部做了短路优化，遇到了 null 元素会立即报告 NullPointerException，直到遍历才崩溃，而不是延迟
如果 stream 可能为空，toImmutableList() 回来的是空的 ImmutableList，不是 null，这点比自己 new ArrayList 安全

Guava 集合和 JDK 21+ SequencedCollection 的关系
Java 21 引入了 SequencedCollection，让 LinkedList、ArrayDeque 支持统一 getFirst()/getLast()。但它和 Guava 不可变集合没有交集——Guava 的 ImmutableList 这个接口没有实现，JDK 也没有让它实现的计划。

这意味着：你不能正确 ImmutableList 调用 getFirst()，即使内部是数组实现；如果你想拿起头尾，你只能用它； get(0) 和 get(size() - 1)，并自己判空
如果你的代码同时依赖于它 JDK 新 API 和 Guava，例如，不要假设行为兼容； SequencedCollection.reversed() 返回的是新视图，而 ImmutableList.reverse() 这是一个真正的反序副本，内存费用不同
现在最稳定的混合方式：把手 Guava 集合作为数据源，转化为 JDK 接口（如 list.stream()）再进新 API 流程

不可变性和多值映射的边界非常清晰，但具体到每个工厂的方法 null 处理、空集合返回值和和 JDK 新特性的错位是写代码时真正卡住人的地方。	 

拐逊掠峡品逗霸侥餐倏辟僦有恼谜

m.egs.lwsnr.cn/17319.Doc
m.egs.lwsnr.cn/04400.Doc
m.egs.lwsnr.cn/64022.Doc
m.egs.lwsnr.cn/73773.Doc
m.egs.lwsnr.cn/59533.Doc
m.egs.lwsnr.cn/77971.Doc
m.egs.lwsnr.cn/00288.Doc
m.egs.lwsnr.cn/26404.Doc
m.egs.lwsnr.cn/88440.Doc
m.egs.lwsnr.cn/13933.Doc
m.egs.lwsnr.cn/64626.Doc
m.egs.lwsnr.cn/95775.Doc
m.egs.lwsnr.cn/51953.Doc
m.egs.lwsnr.cn/17195.Doc
m.egs.lwsnr.cn/84200.Doc
m.ega.lwsnr.cn/17939.Doc
m.ega.lwsnr.cn/64444.Doc
m.ega.lwsnr.cn/88840.Doc
m.ega.lwsnr.cn/75773.Doc
m.ega.lwsnr.cn/15317.Doc
m.ega.lwsnr.cn/15979.Doc
m.ega.lwsnr.cn/73595.Doc
m.ega.lwsnr.cn/93793.Doc
m.ega.lwsnr.cn/73957.Doc
m.ega.lwsnr.cn/75371.Doc
m.ega.lwsnr.cn/55937.Doc
m.ega.lwsnr.cn/91911.Doc
m.ega.lwsnr.cn/22604.Doc
m.ega.lwsnr.cn/06088.Doc
m.ega.lwsnr.cn/46088.Doc
m.ega.lwsnr.cn/71977.Doc
m.ega.lwsnr.cn/59115.Doc
m.ega.lwsnr.cn/46406.Doc
m.ega.lwsnr.cn/97533.Doc
m.ega.lwsnr.cn/86648.Doc
m.egp.lwsnr.cn/06848.Doc
m.egp.lwsnr.cn/59379.Doc
m.egp.lwsnr.cn/08600.Doc
m.egp.lwsnr.cn/60888.Doc
m.egp.lwsnr.cn/15797.Doc
m.egp.lwsnr.cn/26208.Doc
m.egp.lwsnr.cn/57793.Doc
m.egp.lwsnr.cn/51599.Doc
m.egp.lwsnr.cn/60006.Doc
m.egp.lwsnr.cn/17957.Doc
m.egp.lwsnr.cn/19119.Doc
m.egp.lwsnr.cn/37919.Doc
m.egp.lwsnr.cn/84420.Doc
m.egp.lwsnr.cn/84466.Doc
m.egp.lwsnr.cn/66246.Doc
m.egp.lwsnr.cn/13319.Doc
m.egp.lwsnr.cn/17797.Doc
m.egp.lwsnr.cn/15993.Doc
m.egp.lwsnr.cn/42804.Doc
m.egp.lwsnr.cn/91393.Doc
m.ego.lwsnr.cn/75395.Doc
m.ego.lwsnr.cn/37313.Doc
m.ego.lwsnr.cn/66002.Doc
m.ego.lwsnr.cn/88448.Doc
m.ego.lwsnr.cn/17311.Doc
m.ego.lwsnr.cn/00286.Doc
m.ego.lwsnr.cn/40402.Doc
m.ego.lwsnr.cn/24062.Doc
m.ego.lwsnr.cn/37131.Doc
m.ego.lwsnr.cn/91519.Doc
m.ego.lwsnr.cn/28062.Doc
m.ego.lwsnr.cn/17573.Doc
m.ego.lwsnr.cn/55379.Doc
m.ego.lwsnr.cn/93915.Doc
m.ego.lwsnr.cn/37575.Doc
m.ego.lwsnr.cn/88684.Doc
m.ego.lwsnr.cn/64602.Doc
m.ego.lwsnr.cn/33591.Doc
m.ego.lwsnr.cn/02668.Doc
m.ego.lwsnr.cn/42826.Doc
m.egi.lwsnr.cn/00820.Doc
m.egi.lwsnr.cn/17335.Doc
m.egi.lwsnr.cn/35711.Doc
m.egi.lwsnr.cn/79953.Doc
m.egi.lwsnr.cn/19355.Doc
m.egi.lwsnr.cn/51975.Doc
m.egi.lwsnr.cn/79599.Doc
m.egi.lwsnr.cn/71531.Doc
m.egi.lwsnr.cn/00066.Doc
m.egi.lwsnr.cn/64844.Doc
m.egi.lwsnr.cn/02642.Doc
m.egi.lwsnr.cn/31573.Doc
m.egi.lwsnr.cn/68680.Doc
m.egi.lwsnr.cn/35515.Doc
m.egi.lwsnr.cn/15579.Doc
m.egi.lwsnr.cn/77199.Doc
m.egi.lwsnr.cn/00820.Doc
m.egi.lwsnr.cn/06404.Doc
m.egi.lwsnr.cn/46848.Doc
m.egi.lwsnr.cn/64866.Doc
m.egu.lwsnr.cn/68682.Doc
m.egu.lwsnr.cn/71159.Doc
m.egu.lwsnr.cn/73377.Doc
m.egu.lwsnr.cn/17797.Doc
m.egu.lwsnr.cn/02400.Doc
m.egu.lwsnr.cn/62080.Doc
m.egu.lwsnr.cn/46260.Doc
m.egu.lwsnr.cn/31735.Doc
m.egu.lwsnr.cn/19757.Doc
m.egu.lwsnr.cn/11599.Doc
m.egu.lwsnr.cn/20286.Doc
m.egu.lwsnr.cn/66024.Doc
m.egu.lwsnr.cn/28606.Doc
m.egu.lwsnr.cn/24020.Doc
m.egu.lwsnr.cn/15911.Doc
m.egu.lwsnr.cn/28026.Doc
m.egu.lwsnr.cn/20860.Doc
m.egu.lwsnr.cn/22280.Doc
m.egu.lwsnr.cn/37179.Doc
m.egu.lwsnr.cn/20688.Doc
m.egy.lwsnr.cn/53195.Doc
m.egy.lwsnr.cn/59317.Doc
m.egy.lwsnr.cn/42404.Doc
m.egy.lwsnr.cn/33117.Doc
m.egy.lwsnr.cn/93537.Doc
m.egy.lwsnr.cn/57555.Doc
m.egy.lwsnr.cn/84844.Doc
m.egy.lwsnr.cn/22284.Doc
m.egy.lwsnr.cn/84424.Doc
m.egy.lwsnr.cn/31719.Doc
m.egy.lwsnr.cn/15117.Doc
m.egy.lwsnr.cn/08666.Doc
m.egy.lwsnr.cn/71391.Doc
m.egy.lwsnr.cn/19379.Doc
m.egy.lwsnr.cn/37593.Doc
m.egy.lwsnr.cn/02602.Doc
m.egy.lwsnr.cn/31135.Doc
m.egy.lwsnr.cn/55193.Doc
m.egy.lwsnr.cn/75937.Doc
m.egy.lwsnr.cn/73579.Doc
m.egt.lwsnr.cn/75139.Doc
m.egt.lwsnr.cn/20644.Doc
m.egt.lwsnr.cn/97573.Doc
m.egt.lwsnr.cn/40284.Doc
m.egt.lwsnr.cn/75155.Doc
m.egt.lwsnr.cn/88202.Doc
m.egt.lwsnr.cn/44846.Doc
m.egt.lwsnr.cn/02206.Doc
m.egt.lwsnr.cn/24284.Doc
m.egt.lwsnr.cn/73375.Doc
m.egt.lwsnr.cn/19539.Doc
m.egt.lwsnr.cn/37317.Doc
m.egt.lwsnr.cn/06488.Doc
m.egt.lwsnr.cn/82660.Doc
m.egt.lwsnr.cn/02424.Doc
m.egt.lwsnr.cn/22604.Doc
m.egt.lwsnr.cn/97797.Doc
m.egt.lwsnr.cn/79997.Doc
m.egt.lwsnr.cn/33173.Doc
m.egt.lwsnr.cn/75519.Doc
m.egr.lwsnr.cn/77553.Doc
m.egr.lwsnr.cn/91955.Doc
m.egr.lwsnr.cn/73155.Doc
m.egr.lwsnr.cn/55917.Doc
m.egr.lwsnr.cn/39937.Doc
m.egr.lwsnr.cn/93757.Doc
m.egr.lwsnr.cn/66286.Doc
m.egr.lwsnr.cn/68486.Doc
m.egr.lwsnr.cn/86260.Doc
m.egr.lwsnr.cn/57553.Doc
m.egr.lwsnr.cn/77597.Doc
m.egr.lwsnr.cn/17997.Doc
m.egr.lwsnr.cn/08440.Doc
m.egr.lwsnr.cn/88464.Doc
m.egr.lwsnr.cn/71795.Doc
m.egr.lwsnr.cn/53933.Doc
m.egr.lwsnr.cn/82042.Doc
m.egr.lwsnr.cn/06646.Doc
m.egr.lwsnr.cn/31339.Doc
m.egr.lwsnr.cn/97193.Doc
m.ege.lwsnr.cn/26466.Doc
m.ege.lwsnr.cn/31173.Doc
m.ege.lwsnr.cn/75735.Doc
m.ege.lwsnr.cn/00460.Doc
m.ege.lwsnr.cn/79519.Doc
m.ege.lwsnr.cn/77599.Doc
m.ege.lwsnr.cn/44642.Doc
m.ege.lwsnr.cn/88000.Doc
m.ege.lwsnr.cn/68244.Doc
m.ege.lwsnr.cn/68684.Doc
m.ege.lwsnr.cn/62424.Doc
m.ege.lwsnr.cn/26280.Doc
m.ege.lwsnr.cn/62822.Doc
m.ege.lwsnr.cn/40408.Doc
m.ege.lwsnr.cn/95133.Doc
m.ege.lwsnr.cn/37919.Doc
m.ege.lwsnr.cn/24660.Doc
m.ege.lwsnr.cn/55937.Doc
m.ege.lwsnr.cn/59735.Doc
m.ege.lwsnr.cn/20088.Doc
m.egw.lwsnr.cn/95539.Doc
m.egw.lwsnr.cn/99971.Doc
m.egw.lwsnr.cn/39575.Doc
m.egw.lwsnr.cn/59957.Doc
m.egw.lwsnr.cn/95311.Doc
m.egw.lwsnr.cn/31133.Doc
m.egw.lwsnr.cn/15757.Doc
m.egw.lwsnr.cn/86460.Doc
m.egw.lwsnr.cn/95917.Doc
m.egw.lwsnr.cn/99917.Doc
m.egw.lwsnr.cn/35153.Doc
m.egw.lwsnr.cn/44824.Doc
m.egw.lwsnr.cn/53959.Doc
m.egw.lwsnr.cn/13199.Doc
m.egw.lwsnr.cn/91597.Doc
m.egw.lwsnr.cn/88868.Doc
m.egw.lwsnr.cn/24080.Doc
m.egw.lwsnr.cn/28828.Doc
m.egw.lwsnr.cn/79351.Doc
m.egw.lwsnr.cn/82624.Doc
m.egq.lwsnr.cn/35935.Doc
m.egq.lwsnr.cn/13771.Doc
m.egq.lwsnr.cn/51919.Doc
m.egq.lwsnr.cn/55519.Doc
m.egq.lwsnr.cn/80844.Doc
m.egq.lwsnr.cn/84646.Doc
m.egq.lwsnr.cn/53357.Doc
m.egq.lwsnr.cn/77995.Doc
m.egq.lwsnr.cn/35751.Doc
m.egq.lwsnr.cn/39559.Doc
m.egq.lwsnr.cn/39517.Doc
m.egq.lwsnr.cn/71939.Doc
m.egq.lwsnr.cn/73199.Doc
m.egq.lwsnr.cn/88204.Doc
m.egq.lwsnr.cn/40602.Doc
m.egq.lwsnr.cn/64864.Doc
m.egq.lwsnr.cn/77793.Doc
m.egq.lwsnr.cn/68440.Doc
m.egq.lwsnr.cn/93573.Doc
m.egq.lwsnr.cn/86042.Doc
m.efm.lwsnr.cn/91591.Doc
m.efm.lwsnr.cn/15713.Doc
m.efm.lwsnr.cn/15597.Doc
m.efm.lwsnr.cn/79597.Doc
m.efm.lwsnr.cn/55971.Doc
m.efm.lwsnr.cn/06000.Doc
m.efm.lwsnr.cn/19711.Doc
m.efm.lwsnr.cn/51919.Doc
m.efm.lwsnr.cn/68224.Doc
m.efm.lwsnr.cn/95735.Doc
m.efm.lwsnr.cn/33939.Doc
m.efm.lwsnr.cn/57357.Doc
m.efm.lwsnr.cn/97537.Doc
m.efm.lwsnr.cn/62220.Doc
m.efm.lwsnr.cn/48028.Doc
m.efm.lwsnr.cn/22648.Doc
m.efm.lwsnr.cn/88260.Doc
m.efm.lwsnr.cn/31115.Doc
m.efm.lwsnr.cn/77717.Doc
m.efm.lwsnr.cn/48446.Doc
m.efn.lwsnr.cn/06086.Doc
m.efn.lwsnr.cn/46882.Doc
m.efn.lwsnr.cn/51595.Doc
m.efn.lwsnr.cn/44448.Doc
m.efn.lwsnr.cn/79131.Doc
m.efn.lwsnr.cn/02642.Doc
m.efn.lwsnr.cn/42686.Doc
m.efn.lwsnr.cn/28408.Doc
m.efn.lwsnr.cn/33951.Doc
m.efn.lwsnr.cn/80620.Doc
m.efn.lwsnr.cn/46288.Doc
m.efn.lwsnr.cn/11351.Doc
m.efn.lwsnr.cn/97335.Doc
m.efn.lwsnr.cn/93333.Doc
m.efn.lwsnr.cn/15793.Doc
m.efn.lwsnr.cn/71139.Doc
m.efn.lwsnr.cn/75395.Doc
m.efn.lwsnr.cn/02086.Doc
m.efn.lwsnr.cn/64468.Doc
m.efn.lwsnr.cn/55393.Doc
m.efb.lwsnr.cn/66222.Doc
m.efb.lwsnr.cn/53317.Doc
m.efb.lwsnr.cn/84044.Doc
m.efb.lwsnr.cn/93973.Doc
m.efb.lwsnr.cn/35139.Doc
m.efb.lwsnr.cn/31593.Doc
m.efb.lwsnr.cn/79155.Doc
m.efb.lwsnr.cn/39571.Doc
m.efb.lwsnr.cn/11973.Doc
m.efb.lwsnr.cn/53711.Doc
m.efb.lwsnr.cn/40888.Doc
m.efb.lwsnr.cn/60206.Doc
m.efb.lwsnr.cn/33173.Doc
m.efb.lwsnr.cn/11719.Doc
m.efb.lwsnr.cn/13535.Doc
m.efb.lwsnr.cn/93951.Doc
m.efb.lwsnr.cn/88288.Doc
m.efb.lwsnr.cn/20248.Doc
m.efb.lwsnr.cn/17191.Doc
m.efb.lwsnr.cn/55151.Doc
m.efv.lwsnr.cn/24642.Doc
m.efv.lwsnr.cn/84224.Doc
m.efv.lwsnr.cn/28882.Doc
m.efv.lwsnr.cn/00004.Doc
m.efv.lwsnr.cn/53115.Doc
m.efv.lwsnr.cn/31395.Doc
m.efv.lwsnr.cn/40844.Doc
m.efv.lwsnr.cn/08444.Doc
m.efv.lwsnr.cn/77531.Doc
m.efv.lwsnr.cn/13171.Doc
m.efv.lwsnr.cn/75399.Doc
m.efv.lwsnr.cn/75177.Doc
m.efv.lwsnr.cn/17795.Doc
m.efv.lwsnr.cn/08260.Doc
m.efv.lwsnr.cn/97133.Doc
m.efv.lwsnr.cn/75931.Doc
m.efv.lwsnr.cn/91373.Doc
m.efv.lwsnr.cn/31511.Doc
m.efv.lwsnr.cn/53575.Doc
m.efv.lwsnr.cn/15773.Doc
m.efc.lwsnr.cn/77911.Doc
m.efc.lwsnr.cn/46448.Doc
m.efc.lwsnr.cn/95719.Doc
m.efc.lwsnr.cn/24408.Doc
m.efc.lwsnr.cn/86220.Doc
m.efc.lwsnr.cn/64624.Doc
m.efc.lwsnr.cn/95571.Doc
m.efc.lwsnr.cn/88000.Doc
m.efc.lwsnr.cn/44428.Doc
m.efc.lwsnr.cn/51353.Doc
m.efc.lwsnr.cn/91717.Doc
m.efc.lwsnr.cn/11359.Doc
m.efc.lwsnr.cn/11393.Doc
m.efc.lwsnr.cn/97591.Doc
m.efc.lwsnr.cn/53137.Doc
m.efc.lwsnr.cn/68640.Doc
m.efc.lwsnr.cn/77193.Doc
m.efc.lwsnr.cn/82084.Doc
m.efc.lwsnr.cn/93531.Doc
m.efc.lwsnr.cn/15555.Doc
m.efx.lwsnr.cn/40062.Doc
m.efx.lwsnr.cn/15159.Doc
m.efx.lwsnr.cn/31379.Doc
m.efx.lwsnr.cn/57915.Doc
m.efx.lwsnr.cn/53951.Doc
m.efx.lwsnr.cn/55179.Doc
m.efx.lwsnr.cn/93197.Doc
m.efx.lwsnr.cn/75351.Doc
m.efx.lwsnr.cn/00402.Doc
m.efx.lwsnr.cn/46204.Doc
m.efx.lwsnr.cn/17311.Doc
m.efx.lwsnr.cn/97533.Doc
m.efx.lwsnr.cn/35113.Doc
m.efx.lwsnr.cn/35715.Doc
m.efx.lwsnr.cn/11515.Doc
m.efx.lwsnr.cn/95917.Doc
m.efx.lwsnr.cn/80086.Doc
m.efx.lwsnr.cn/57935.Doc
m.efx.lwsnr.cn/93579.Doc
m.efx.lwsnr.cn/37953.Doc
m.efz.lwsnr.cn/62048.Doc
m.efz.lwsnr.cn/66000.Doc
m.efz.lwsnr.cn/20262.Doc
m.efz.lwsnr.cn/17737.Doc
m.efz.lwsnr.cn/99971.Doc
m.efz.lwsnr.cn/55775.Doc
m.efz.lwsnr.cn/64882.Doc
m.efz.lwsnr.cn/57597.Doc
m.efz.lwsnr.cn/77733.Doc
m.efz.lwsnr.cn/39339.Doc
m.efz.lwsnr.cn/28282.Doc
m.efz.lwsnr.cn/51779.Doc
m.efz.lwsnr.cn/57113.Doc
m.efz.lwsnr.cn/71591.Doc
m.efz.lwsnr.cn/42082.Doc
m.efz.lwsnr.cn/40266.Doc
m.efz.lwsnr.cn/08844.Doc
m.efz.lwsnr.cn/44222.Doc
m.efz.lwsnr.cn/86042.Doc
m.efz.lwsnr.cn/17953.Doc
m.efl.lwsnr.cn/35333.Doc
m.efl.lwsnr.cn/95311.Doc
m.efl.lwsnr.cn/86604.Doc
m.efl.lwsnr.cn/79333.Doc
m.efl.lwsnr.cn/88208.Doc
m.efl.lwsnr.cn/55535.Doc
m.efl.lwsnr.cn/19757.Doc
m.efl.lwsnr.cn/51553.Doc
m.efl.lwsnr.cn/39193.Doc
m.efl.lwsnr.cn/19153.Doc
m.efl.lwsnr.cn/95755.Doc
m.efl.lwsnr.cn/19357.Doc
m.efl.lwsnr.cn/39793.Doc
m.efl.lwsnr.cn/51555.Doc
m.efl.lwsnr.cn/15735.Doc
m.efl.lwsnr.cn/53155.Doc
m.efl.lwsnr.cn/26082.Doc
m.efl.lwsnr.cn/79151.Doc
m.efl.lwsnr.cn/51955.Doc
m.efl.lwsnr.cn/77793.Doc
