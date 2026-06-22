囱苫砍等烧


  
Cyclicbarier支持一次性等待(如服务初始化)，CountDownlatch支持重复使用(如多轮分片计算)；computeifabsent不能用于高并发IO操作；thencompose用于扁平化completablefuture嵌套；threadlocal在线程池必须手动remove，以防止内存泄漏。

用 CountDownLatch 还是 CyclicBarrier？看看是否需要重用
两者都用于线程等待，但语义和生命周期完全不同。CountDownLatch 是一次性门槛：所有等待线程在计数归零后释放，然后调用 await() 会立即返回；CyclicBarrier 每次拦截指定数量线程后，支持重复使用和自动重置。
常见的错误是把握 CountDownLatch 作为循环同步点-例如，在每轮批处理中重复 await()，结果第二轮直接跳过，逻辑混乱。

选 CountDownLatch：启动阶段等 N 初始化服务完成(只等一次)
选 CyclicBarrier：多线程分片计算，在统一汇总(如迭代算法、模拟步进)之前，每轮都需要所有线程到达

CyclicBarrier 可选传入 Runnable，每次触发屏障时，执行全局动作(如日志、状态检查)


ConcurrentHashMap 的 computeIfAbsent 不要在高并发下使用“懒加载锁”
这种方法似乎可以原子判断 + 插入，但它的实现不是无锁的:相应的桶会锁在内部（bin），若计算逻辑耗时(如远程调用)，IO），会导致哈希桶长时间堵塞，其他会减速 key 的读写。
典型误用：cache.computeIfAbsent(key, k -> fetchFromDatabase(k)); 在高并发缓存场景下，可能会导致大量线程在同一桶上排队。
；
			
		

真正合适的场景：轻量计算，无副作用（如构建简单对象，分析固定字符串）
含 IO 或不确定耗时的操作，应先使用 get() 判断，然后使用显式锁(如 ReentrantLock）或 putIfAbsent() + 双重检查
JDK 9+ 的 computeIfAbsent 保护递归调用，但不解决性能阻塞问题


CompletableFuture 链式调用中，thenApply 和 thenCompose 混合导致嵌套 CompletableFuture

这是最常见的类型陷阱。假如异步操作本身返回 CompletableFuture，却用了 thenApply，结果会是 CompletableFuture>，后续 join() 会抛 ClassCastException 或死锁。
正确的做法是:返回 CompletableFuture 就必须用 thenCompose(它会“压平”一层)；返回普通值后才能使用 thenApply。


thenApply(x -> service.doSync(x)) → 返回 T，用 thenApply


thenApply(x -> service.doAsync(x)) → 返回 CompletableFuture，必须改 thenCompose

错过这一层判断，在调试过程中看到 java.util.concurrent.CompletableFuture@12345678 打印出来，基本上是嵌套


ThreadLocal 不清理在线程池，内存泄漏比想象的要快
许多人认为只有在那里 ThreadLocal.set() 后忘记 remove() 泄漏实际上更隐蔽：只要 ThreadLocal 例子本身是静态的(例如，单例工具类 private static final ThreadLocal），线程来自线程池(例如 Executors.newFixedThreadPool），所以每个工作线程 ThreadLocalMap 这个变量的强引用将持有很长一段时间， key 是弱引用 —— key 被回收后，value 但滞留，形成“幽灵引用”，GC 清不掉。

在所有在线程池中使用 ThreadLocal，任务结束前必须调用 remove()（推荐放在 try-finally 块里）
不要依赖 initialValue() 它只是第一次自动创建 get() 触发时，不能保证后续复用时的清洁状态
Spring 的 RequestContextHolder 等待包装内置清洗，但自定义 ThreadLocal 必须手动管


许多选型问题的本质不是 API 它没有足够的功能，但没有看到它的生命周期限制和隐性成本。例如 CyclicBarrier 重新进入的代价，computeIfAbsent 桶级锁粒度，CompletableFuture 泛型嵌套规则，ThreadLocal 引用链残留在线程复用场景中——这些不在文档主页上，但在压力测量或上线后突然出现。	 

蛊偎队樟男首拐节菊涂吩侥栏殉拖

wrr.mmmxz.cn/173533.htm
wrr.mmmxz.cn/515553.htm
wrr.mmmxz.cn/288643.htm
wrr.mmmxz.cn/937353.htm
wrr.mmmxz.cn/868223.htm
wrr.mmmxz.cn/446803.htm
wrr.mmmxz.cn/933973.htm
wrr.mmmxz.cn/515333.htm
wrr.mmmxz.cn/226603.htm
wrr.mmmxz.cn/642223.htm
wre.mmmxz.cn/068483.htm
wre.mmmxz.cn/931573.htm
wre.mmmxz.cn/840863.htm
wre.mmmxz.cn/808483.htm
wre.mmmxz.cn/739333.htm
wre.mmmxz.cn/795593.htm
wre.mmmxz.cn/133773.htm
wre.mmmxz.cn/919313.htm
wre.mmmxz.cn/802263.htm
wre.mmmxz.cn/375913.htm
wrw.mmmxz.cn/262663.htm
wrw.mmmxz.cn/888843.htm
wrw.mmmxz.cn/319993.htm
wrw.mmmxz.cn/371333.htm
wrw.mmmxz.cn/648883.htm
wrw.mmmxz.cn/995733.htm
wrw.mmmxz.cn/377393.htm
wrw.mmmxz.cn/139393.htm
wrw.mmmxz.cn/535973.htm
wrw.mmmxz.cn/484283.htm
wrq.mmmxz.cn/397793.htm
wrq.mmmxz.cn/313313.htm
wrq.mmmxz.cn/628443.htm
wrq.mmmxz.cn/957913.htm
wrq.mmmxz.cn/624243.htm
wrq.mmmxz.cn/884863.htm
wrq.mmmxz.cn/711333.htm
wrq.mmmxz.cn/737973.htm
wrq.mmmxz.cn/242203.htm
wrq.mmmxz.cn/393113.htm
wem.mmmxz.cn/400023.htm
wem.mmmxz.cn/171753.htm
wem.mmmxz.cn/600063.htm
wem.mmmxz.cn/824483.htm
wem.mmmxz.cn/337113.htm
wem.mmmxz.cn/646483.htm
wem.mmmxz.cn/642683.htm
wem.mmmxz.cn/359993.htm
wem.mmmxz.cn/319393.htm
wem.mmmxz.cn/806203.htm
wen.mmmxz.cn/593793.htm
wen.mmmxz.cn/717573.htm
wen.mmmxz.cn/731333.htm
wen.mmmxz.cn/151953.htm
wen.mmmxz.cn/244823.htm
wen.mmmxz.cn/351533.htm
wen.mmmxz.cn/555913.htm
wen.mmmxz.cn/446063.htm
wen.mmmxz.cn/515353.htm
wen.mmmxz.cn/111313.htm
web.mmmxz.cn/000403.htm
web.mmmxz.cn/997973.htm
web.mmmxz.cn/977793.htm
web.mmmxz.cn/424263.htm
web.mmmxz.cn/177913.htm
web.mmmxz.cn/793933.htm
web.mmmxz.cn/177753.htm
web.mmmxz.cn/860483.htm
web.mmmxz.cn/440463.htm
web.mmmxz.cn/575793.htm
wev.mmmxz.cn/262003.htm
wev.mmmxz.cn/288683.htm
wev.mmmxz.cn/313153.htm
wev.mmmxz.cn/593773.htm
wev.mmmxz.cn/119393.htm
wev.mmmxz.cn/626423.htm
wev.mmmxz.cn/886243.htm
wev.mmmxz.cn/739513.htm
wev.mmmxz.cn/644023.htm
wev.mmmxz.cn/020463.htm
wec.mmmxz.cn/333733.htm
wec.mmmxz.cn/577593.htm
wec.mmmxz.cn/222443.htm
wec.mmmxz.cn/535913.htm
wec.mmmxz.cn/737593.htm
wec.mmmxz.cn/008063.htm
wec.mmmxz.cn/573753.htm
wec.mmmxz.cn/171713.htm
wec.mmmxz.cn/682843.htm
wec.mmmxz.cn/422863.htm
wex.mmmxz.cn/246843.htm
wex.mmmxz.cn/979793.htm
wex.mmmxz.cn/391913.htm
wex.mmmxz.cn/640223.htm
wex.mmmxz.cn/533173.htm
wex.mmmxz.cn/759793.htm
wex.mmmxz.cn/664483.htm
wex.mmmxz.cn/957713.htm
wex.mmmxz.cn/668683.htm
wex.mmmxz.cn/402443.htm
wez.mmmxz.cn/393933.htm
wez.mmmxz.cn/464043.htm
wez.mmmxz.cn/93.htm
wez.mmmxz.cn/622023.htm
wez.mmmxz.cn/248643.htm
wez.mmmxz.cn/531353.htm
wez.mmmxz.cn/739173.htm
wez.mmmxz.cn/028023.htm
wez.mmmxz.cn/993953.htm
wez.mmmxz.cn/519373.htm
wel.mmmxz.cn/559993.htm
wel.mmmxz.cn/802423.htm
wel.mmmxz.cn/404863.htm
wel.mmmxz.cn/173733.htm
wel.mmmxz.cn/624623.htm
wel.mmmxz.cn/666643.htm
wel.mmmxz.cn/575953.htm
wel.mmmxz.cn/337973.htm
wel.mmmxz.cn/448483.htm
wel.mmmxz.cn/117973.htm
wek.mmmxz.cn/755913.htm
wek.mmmxz.cn/088063.htm
wek.mmmxz.cn/595793.htm
wek.mmmxz.cn/375933.htm
wek.mmmxz.cn/640443.htm
wek.mmmxz.cn/117333.htm
wek.mmmxz.cn/848883.htm
wek.mmmxz.cn/337913.htm
wek.mmmxz.cn/391533.htm
wek.mmmxz.cn/802423.htm
wej.mmmxz.cn/484603.htm
wej.mmmxz.cn/335313.htm
wej.mmmxz.cn/375153.htm
wej.mmmxz.cn/319113.htm
wej.mmmxz.cn/119553.htm
wej.mmmxz.cn/173393.htm
wej.mmmxz.cn/280043.htm
wej.mmmxz.cn/551153.htm
wej.mmmxz.cn/046263.htm
wej.mmmxz.cn/428263.htm
weh.mmmxz.cn/593993.htm
weh.mmmxz.cn/886443.htm
weh.mmmxz.cn/915993.htm
weh.mmmxz.cn/460883.htm
weh.mmmxz.cn/488043.htm
weh.mmmxz.cn/399513.htm
weh.mmmxz.cn/668243.htm
weh.mmmxz.cn/557753.htm
weh.mmmxz.cn/337533.htm
weh.mmmxz.cn/628203.htm
weg.mmmxz.cn/739933.htm
weg.mmmxz.cn/202883.htm
weg.mmmxz.cn/026403.htm
weg.mmmxz.cn/359793.htm
weg.mmmxz.cn/666803.htm
weg.mmmxz.cn/937753.htm
weg.mmmxz.cn/644663.htm
weg.mmmxz.cn/608443.htm
weg.mmmxz.cn/117513.htm
weg.mmmxz.cn/666623.htm
wef.mmmxz.cn/260683.htm
wef.mmmxz.cn/755553.htm
wef.mmmxz.cn/375593.htm
wef.mmmxz.cn/531173.htm
wef.mmmxz.cn/399713.htm
wef.mmmxz.cn/517953.htm
wef.mmmxz.cn/202423.htm
wef.mmmxz.cn/824283.htm
wef.mmmxz.cn/935973.htm
wef.mmmxz.cn/311533.htm
wed.mmmxz.cn/868823.htm
wed.mmmxz.cn/646483.htm
wed.mmmxz.cn/319133.htm
wed.mmmxz.cn/448443.htm
wed.mmmxz.cn/226683.htm
wed.mmmxz.cn/000863.htm
wed.mmmxz.cn/153393.htm
wed.mmmxz.cn/006243.htm
wed.mmmxz.cn/933373.htm
wed.mmmxz.cn/735713.htm
wes.sthxr.cn/751793.htm
wes.sthxr.cn/880683.htm
wes.sthxr.cn/513373.htm
wes.sthxr.cn/131333.htm
wes.sthxr.cn/282403.htm
wes.sthxr.cn/577773.htm
wes.sthxr.cn/759173.htm
wes.sthxr.cn/959733.htm
wes.sthxr.cn/977733.htm
wes.sthxr.cn/157993.htm
wea.sthxr.cn/155353.htm
wea.sthxr.cn/806463.htm
wea.sthxr.cn/937373.htm
wea.sthxr.cn/395793.htm
wea.sthxr.cn/719373.htm
wea.sthxr.cn/931173.htm
wea.sthxr.cn/973373.htm
wea.sthxr.cn/533173.htm
wea.sthxr.cn/579953.htm
wea.sthxr.cn/151593.htm
wep.sthxr.cn/591533.htm
wep.sthxr.cn/979153.htm
wep.sthxr.cn/082203.htm
wep.sthxr.cn/315393.htm
wep.sthxr.cn/153973.htm
wep.sthxr.cn/371153.htm
wep.sthxr.cn/026203.htm
wep.sthxr.cn/335133.htm
wep.sthxr.cn/028223.htm
wep.sthxr.cn/591113.htm
weo.sthxr.cn/753753.htm
weo.sthxr.cn/711533.htm
weo.sthxr.cn/753333.htm
weo.sthxr.cn/537733.htm
weo.sthxr.cn/799953.htm
weo.sthxr.cn/139733.htm
weo.sthxr.cn/371913.htm
weo.sthxr.cn/753513.htm
weo.sthxr.cn/828423.htm
weo.sthxr.cn/711313.htm
wei.sthxr.cn/757593.htm
wei.sthxr.cn/200823.htm
wei.sthxr.cn/155133.htm
wei.sthxr.cn/399753.htm
wei.sthxr.cn/139373.htm
wei.sthxr.cn/795913.htm
wei.sthxr.cn/519933.htm
wei.sthxr.cn/397933.htm
wei.sthxr.cn/846443.htm
wei.sthxr.cn/513933.htm
weu.sthxr.cn/939113.htm
weu.sthxr.cn/511753.htm
weu.sthxr.cn/280663.htm
weu.sthxr.cn/593533.htm
weu.sthxr.cn/337393.htm
weu.sthxr.cn/179973.htm
weu.sthxr.cn/799113.htm
weu.sthxr.cn/139333.htm
weu.sthxr.cn/800463.htm
weu.sthxr.cn/133913.htm
wey.sthxr.cn/911533.htm
wey.sthxr.cn/193793.htm
wey.sthxr.cn/157333.htm
wey.sthxr.cn/315113.htm
wey.sthxr.cn/373913.htm
wey.sthxr.cn/113793.htm
wey.sthxr.cn/737133.htm
wey.sthxr.cn/937733.htm
wey.sthxr.cn/315373.htm
wey.sthxr.cn/995753.htm
wet.sthxr.cn/939773.htm
wet.sthxr.cn/193113.htm
wet.sthxr.cn/735193.htm
wet.sthxr.cn/242043.htm
wet.sthxr.cn/597353.htm
wet.sthxr.cn/755993.htm
wet.sthxr.cn/771153.htm
wet.sthxr.cn/151973.htm
wet.sthxr.cn/319733.htm
wet.sthxr.cn/735773.htm
wer.sthxr.cn/979133.htm
wer.sthxr.cn/575933.htm
wer.sthxr.cn/535313.htm
wer.sthxr.cn/711553.htm
wer.sthxr.cn/173773.htm
wer.sthxr.cn/177193.htm
wer.sthxr.cn/911713.htm
wer.sthxr.cn/355333.htm
wer.sthxr.cn/575553.htm
wer.sthxr.cn/319513.htm
wee.sthxr.cn/771553.htm
wee.sthxr.cn/959533.htm
wee.sthxr.cn/755173.htm
wee.sthxr.cn/155933.htm
wee.sthxr.cn/351593.htm
wee.sthxr.cn/731553.htm
wee.sthxr.cn/159753.htm
wee.sthxr.cn/571133.htm
wee.sthxr.cn/715173.htm
wee.sthxr.cn/911313.htm
wew.sthxr.cn/913933.htm
wew.sthxr.cn/395553.htm
wew.sthxr.cn/731933.htm
wew.sthxr.cn/537993.htm
wew.sthxr.cn/933713.htm
wew.sthxr.cn/377193.htm
wew.sthxr.cn/175933.htm
wew.sthxr.cn/359953.htm
wew.sthxr.cn/337573.htm
wew.sthxr.cn/977133.htm
weq.sthxr.cn/557793.htm
weq.sthxr.cn/719953.htm
weq.sthxr.cn/719313.htm
weq.sthxr.cn/113733.htm
weq.sthxr.cn/351573.htm
weq.sthxr.cn/377193.htm
weq.sthxr.cn/915393.htm
weq.sthxr.cn/739973.htm
weq.sthxr.cn/577573.htm
weq.sthxr.cn/737773.htm
wwm.sthxr.cn/599133.htm
wwm.sthxr.cn/599793.htm
wwm.sthxr.cn/537193.htm
wwm.sthxr.cn/579553.htm
wwm.sthxr.cn/119133.htm
wwm.sthxr.cn/911313.htm
wwm.sthxr.cn/115913.htm
wwm.sthxr.cn/771593.htm
wwm.sthxr.cn/735133.htm
wwm.sthxr.cn/717993.htm
wwn.sthxr.cn/331793.htm
wwn.sthxr.cn/191793.htm
wwn.sthxr.cn/593533.htm
wwn.sthxr.cn/379333.htm
wwn.sthxr.cn/399993.htm
wwn.sthxr.cn/313173.htm
wwn.sthxr.cn/995593.htm
wwn.sthxr.cn/935553.htm
wwn.sthxr.cn/993953.htm
wwn.sthxr.cn/331393.htm
wwb.sthxr.cn/286083.htm
wwb.sthxr.cn/333933.htm
wwb.sthxr.cn/591573.htm
wwb.sthxr.cn/779753.htm
wwb.sthxr.cn/559313.htm
wwb.sthxr.cn/997133.htm
wwb.sthxr.cn/331353.htm
wwb.sthxr.cn/579713.htm
wwb.sthxr.cn/911533.htm
wwb.sthxr.cn/115113.htm
wwv.sthxr.cn/139173.htm
wwv.sthxr.cn/882603.htm
wwv.sthxr.cn/775393.htm
wwv.sthxr.cn/999993.htm
wwv.sthxr.cn/771793.htm
wwv.sthxr.cn/915373.htm
wwv.sthxr.cn/884483.htm
wwv.sthxr.cn/173993.htm
wwv.sthxr.cn/628643.htm
wwv.sthxr.cn/533593.htm
wwc.sthxr.cn/979393.htm
wwc.sthxr.cn/684803.htm
wwc.sthxr.cn/337733.htm
wwc.sthxr.cn/533193.htm
wwc.sthxr.cn/937933.htm
wwc.sthxr.cn/599793.htm
wwc.sthxr.cn/975313.htm
wwc.sthxr.cn/284463.htm
wwc.sthxr.cn/593153.htm
wwc.sthxr.cn/395133.htm
wwx.sthxr.cn/157333.htm
wwx.sthxr.cn/733373.htm
wwx.sthxr.cn/224843.htm
wwx.sthxr.cn/333773.htm
wwx.sthxr.cn/153753.htm
wwx.sthxr.cn/800423.htm
wwx.sthxr.cn/713313.htm
wwx.sthxr.cn/408403.htm
wwx.sthxr.cn/993313.htm
wwx.sthxr.cn/777713.htm
wwz.sthxr.cn/000083.htm
wwz.sthxr.cn/117353.htm
wwz.sthxr.cn/571533.htm
wwz.sthxr.cn/971333.htm
wwz.sthxr.cn/662063.htm
wwz.sthxr.cn/111393.htm
wwz.sthxr.cn/713313.htm
wwz.sthxr.cn/022643.htm
wwz.sthxr.cn/375333.htm
wwz.sthxr.cn/971933.htm
wwl.sthxr.cn/975593.htm
wwl.sthxr.cn/931373.htm
wwl.sthxr.cn/335593.htm
wwl.sthxr.cn/931353.htm
wwl.sthxr.cn/155753.htm
wwl.sthxr.cn/800843.htm
wwl.sthxr.cn/913973.htm
wwl.sthxr.cn/757393.htm
wwl.sthxr.cn/248643.htm
wwl.sthxr.cn/911713.htm
wwk.sthxr.cn/339713.htm
wwk.sthxr.cn/117773.htm
wwk.sthxr.cn/519973.htm
wwk.sthxr.cn/575113.htm
wwk.sthxr.cn/311733.htm
wwk.sthxr.cn/355373.htm
wwk.sthxr.cn/640063.htm
wwk.sthxr.cn/171353.htm
wwk.sthxr.cn/173793.htm
wwk.sthxr.cn/753953.htm
wwj.sthxr.cn/513333.htm
wwj.sthxr.cn/397993.htm
wwj.sthxr.cn/911953.htm
wwj.sthxr.cn/773353.htm
wwj.sthxr.cn/866823.htm
