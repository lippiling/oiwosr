磁艘形牧檬


  
ConcurrentHashMap 由于采用分段锁，更适合高并发场景（JDK 7）或 CAS + synchronized（JDK 8+)，只锁定修改后的桶，读操作无锁；而且 Hashtable 所有方法用 synchronized 修饰，竞争全局锁。

ConcurrentHashMap 为什么比 Hashtable 更适合高并发场景
因为 ConcurrentHashMap 不仅仅是锁定整个表，而是使用分段锁（JDK 7)或更轻 CAS + synchronized（JDK 8+)策略，只锁定修改后的桶（bucket），阅读操作完全无锁。和 Hashtable 使用所有方法 synchronized 修改，意味着每次 get() 或 put() 都要竞争同样的全局锁。
实操建议：

如果项目用 JDK 8+，ConcurrentHashMap 是默认的首选；它支持 computeIfAbsent()、merge() 等原子操作，避免手动同步
不要用 ConcurrentHashMap 的 size() 做出准确的计数判断——它返回估计值，在多线程下可能不一致；需要精确的大小时才能改用 mappingCount()


ConcurrentHashMap 不允许 null 作为 key 或 value，否则抛 NullPointerException；而 HashMap 允许一个 null key

CopyOnWriteArrayList 什么场景真的有用？
它适用于「读多写少」可能在遍历中修改的场景，如事件监控器列表和配置白名单缓存。每次写作操作（add()、remove()）整个底层数组都会被复制，所以写作成本很高，但迭代器永远不会被抛弃 ConcurrentModificationException，也不需要额外的同步。
常见误用：
；

在 for-each 循环里调用 remove() —— 实际删除的是旧副本。目前，迭代器仍指向原始数组，逻辑错误
把它当普通 ArrayList 用于高频增删场景，CPU 内存压力急剧增加
期望它保证「实时一致性」：写入后，遍历的旧迭代器看不到新元素，这是设计造成的，不是 bug

使用 Collections.synchronizedList() 为什么以后还会出现？ ConcurrentModificationException
因为 Collections.synchronizedList(new ArrayList()) 只保证单一方法调用是线程安全的，不保证复合操作的原子性。例如 if (list.isEmpty()) list.add(x) 中，isEmpty() 和 add() 其他线程可能会插入元素，导致逻辑错误；更严重的是，如果迭代没有手动同步，它仍然会触发 ConcurrentModificationException。
			
		
正确的写作方法必须显式加锁：
List list = Collections.synchronizedList(new ArrayList());
// 必须手动同步迭代
synchronized (list) {
Iterator it = list.iterator();
while (it.hasNext()) {
System.out.println(it.next());
}
}
关键点：


synchronizedList 包装类只是“语法糖”
它不能防止迭代过程中的结构修改(除非你总是用同步块包裹整个遍历)
相比 CopyOnWriteArrayList，更适合写作操作少但需要强一致性读写的场景

BlockingQueue 如何配合线程池实现生产者-消费者解耦
典型组合是 ThreadPoolExecutor + ArrayBlockingQueue 或 LinkedBlockingQueue。线程池的 workQueue 参数用于接收提交但尚未执行的任务，本质上是阻塞队列。
参数差异影响明显：


ArrayBlockingQueue 它是一个具有固定容量的有界队列，可以防止内存溢出，但当任务提交超限时，会触发线程池的拒绝策略(如 AbortPolicy 抛 RejectedExecutionException）

LinkedBlockingQueue 默认无界（Integer.MAX_VALUE），看似“不满”，实际上可能会导致 OOM；有界更可控
别直接用 PriorityBlockingQueue 当线程池队列-不保证公平性，任务执行顺序与优先级不一致

容易忽略的一点：线程池 corePoolSize 资源水位由队列容量共同决定。例如 corePoolSize=2、队列容量=10，意味着前面 12 任务不会创建新的线程；第一 13 可能触发扩容(取决于扩容(取决于) maxPoolSize）。这种组合逻辑经常被低估。	 

悄率腋轿嗜堆椭唐掠植聪栈侥脊导

m.qgp.kxnxh.cn/86480.Doc
m.qgp.kxnxh.cn/71335.Doc
m.qgp.kxnxh.cn/13371.Doc
m.qgp.kxnxh.cn/62626.Doc
m.qgp.kxnxh.cn/82608.Doc
m.qgp.kxnxh.cn/88864.Doc
m.qgp.kxnxh.cn/84846.Doc
m.qgo.kxnxh.cn/02426.Doc
m.qgo.kxnxh.cn/62884.Doc
m.qgo.kxnxh.cn/26422.Doc
m.qgo.kxnxh.cn/97971.Doc
m.qgo.kxnxh.cn/28846.Doc
m.qgo.kxnxh.cn/20008.Doc
m.qgo.kxnxh.cn/84262.Doc
m.qgo.kxnxh.cn/88864.Doc
m.qgo.kxnxh.cn/00820.Doc
m.qgo.kxnxh.cn/88048.Doc
m.qgo.kxnxh.cn/82222.Doc
m.qgo.kxnxh.cn/26220.Doc
m.qgo.kxnxh.cn/99395.Doc
m.qgo.kxnxh.cn/60048.Doc
m.qgo.kxnxh.cn/00066.Doc
m.qgo.kxnxh.cn/62684.Doc
m.qgo.kxnxh.cn/68802.Doc
m.qgo.kxnxh.cn/51595.Doc
m.qgo.kxnxh.cn/93719.Doc
m.qgo.kxnxh.cn/46864.Doc
m.qgi.kxnxh.cn/51399.Doc
m.qgi.kxnxh.cn/80620.Doc
m.qgi.kxnxh.cn/82024.Doc
m.qgi.kxnxh.cn/08268.Doc
m.qgi.kxnxh.cn/40626.Doc
m.qgi.kxnxh.cn/06282.Doc
m.qgi.kxnxh.cn/86020.Doc
m.qgi.kxnxh.cn/64448.Doc
m.qgi.kxnxh.cn/66040.Doc
m.qgi.kxnxh.cn/60286.Doc
m.qgi.kxnxh.cn/99175.Doc
m.qgi.kxnxh.cn/86060.Doc
m.qgi.kxnxh.cn/80644.Doc
m.qgi.kxnxh.cn/80008.Doc
m.qgi.kxnxh.cn/86268.Doc
m.qgi.kxnxh.cn/40464.Doc
m.qgi.kxnxh.cn/42628.Doc
m.qgi.kxnxh.cn/11197.Doc
m.qgi.kxnxh.cn/44080.Doc
m.qgi.kxnxh.cn/80288.Doc
m.qgu.kxnxh.cn/93733.Doc
m.qgu.kxnxh.cn/28226.Doc
m.qgu.kxnxh.cn/28668.Doc
m.qgu.kxnxh.cn/08204.Doc
m.qgu.kxnxh.cn/06222.Doc
m.qgu.kxnxh.cn/97513.Doc
m.qgu.kxnxh.cn/60026.Doc
m.qgu.kxnxh.cn/28868.Doc
m.qgu.kxnxh.cn/84644.Doc
m.qgu.kxnxh.cn/00626.Doc
m.qgu.kxnxh.cn/24224.Doc
m.qgu.kxnxh.cn/42262.Doc
m.qgu.kxnxh.cn/57371.Doc
m.qgu.kxnxh.cn/95199.Doc
m.qgu.kxnxh.cn/08288.Doc
m.qgu.kxnxh.cn/22080.Doc
m.qgu.kxnxh.cn/06042.Doc
m.qgu.kxnxh.cn/79355.Doc
m.qgu.kxnxh.cn/82022.Doc
m.qgu.kxnxh.cn/46406.Doc
m.qgy.kxnxh.cn/39779.Doc
m.qgy.kxnxh.cn/26806.Doc
m.qgy.kxnxh.cn/20082.Doc
m.qgy.kxnxh.cn/28666.Doc
m.qgy.kxnxh.cn/44844.Doc
m.qgy.kxnxh.cn/48220.Doc
m.qgy.kxnxh.cn/62002.Doc
m.qgy.kxnxh.cn/40406.Doc
m.qgy.kxnxh.cn/24020.Doc
m.qgy.kxnxh.cn/59395.Doc
m.qgy.kxnxh.cn/62424.Doc
m.qgy.kxnxh.cn/48060.Doc
m.qgy.kxnxh.cn/84866.Doc
m.qgy.kxnxh.cn/28840.Doc
m.qgy.kxnxh.cn/46626.Doc
m.qgy.kxnxh.cn/62424.Doc
m.qgy.kxnxh.cn/82406.Doc
m.qgy.kxnxh.cn/53911.Doc
m.qgy.kxnxh.cn/22422.Doc
m.qgy.kxnxh.cn/46446.Doc
m.qgt.kxnxh.cn/64048.Doc
m.qgt.kxnxh.cn/00022.Doc
m.qgt.kxnxh.cn/06608.Doc
m.qgt.kxnxh.cn/22084.Doc
m.qgt.kxnxh.cn/42626.Doc
m.qgt.kxnxh.cn/88004.Doc
m.qgt.kxnxh.cn/51753.Doc
m.qgt.kxnxh.cn/68080.Doc
m.qgt.kxnxh.cn/75533.Doc
m.qgt.kxnxh.cn/24860.Doc
m.qgt.kxnxh.cn/75179.Doc
m.qgt.kxnxh.cn/62888.Doc
m.qgt.kxnxh.cn/48620.Doc
m.qgt.kxnxh.cn/26280.Doc
m.qgt.kxnxh.cn/88224.Doc
m.qgt.kxnxh.cn/24442.Doc
m.qgt.kxnxh.cn/60264.Doc
m.qgt.kxnxh.cn/24082.Doc
m.qgt.kxnxh.cn/97353.Doc
m.qgt.kxnxh.cn/60026.Doc
m.qgr.kxnxh.cn/40468.Doc
m.qgr.kxnxh.cn/04668.Doc
m.qgr.kxnxh.cn/06200.Doc
m.qgr.kxnxh.cn/39957.Doc
m.qgr.kxnxh.cn/71173.Doc
m.qgr.kxnxh.cn/80046.Doc
m.qgr.kxnxh.cn/46868.Doc
m.qgr.kxnxh.cn/55311.Doc
m.qgr.kxnxh.cn/93311.Doc
m.qgr.kxnxh.cn/46462.Doc
m.qgr.kxnxh.cn/26848.Doc
m.qgr.kxnxh.cn/84400.Doc
m.qgr.kxnxh.cn/46428.Doc
m.qgr.kxnxh.cn/06240.Doc
m.qgr.kxnxh.cn/80088.Doc
m.qgr.kxnxh.cn/86288.Doc
m.qgr.kxnxh.cn/46406.Doc
m.qgr.kxnxh.cn/86622.Doc
m.qgr.kxnxh.cn/53171.Doc
m.qgr.kxnxh.cn/33737.Doc
m.qge.kxnxh.cn/64402.Doc
m.qge.kxnxh.cn/80648.Doc
m.qge.kxnxh.cn/06044.Doc
m.qge.kxnxh.cn/26466.Doc
m.qge.kxnxh.cn/22804.Doc
m.qge.kxnxh.cn/02866.Doc
m.qge.kxnxh.cn/86046.Doc
m.qge.kxnxh.cn/66284.Doc
m.qge.kxnxh.cn/82848.Doc
m.qge.kxnxh.cn/00644.Doc
m.qge.kxnxh.cn/02684.Doc
m.qge.kxnxh.cn/68024.Doc
m.qge.kxnxh.cn/73351.Doc
m.qge.kxnxh.cn/57339.Doc
m.qge.kxnxh.cn/88400.Doc
m.qge.kxnxh.cn/84282.Doc
m.qge.kxnxh.cn/60822.Doc
m.qge.kxnxh.cn/08286.Doc
m.qge.kxnxh.cn/80808.Doc
m.qge.kxnxh.cn/20866.Doc
m.qgw.kxnxh.cn/88646.Doc
m.qgw.kxnxh.cn/35339.Doc
m.qgw.kxnxh.cn/86204.Doc
m.qgw.kxnxh.cn/26220.Doc
m.qgw.kxnxh.cn/26022.Doc
m.qgw.kxnxh.cn/66082.Doc
m.qgw.kxnxh.cn/17939.Doc
m.qgw.kxnxh.cn/48426.Doc
m.qgw.kxnxh.cn/11197.Doc
m.qgw.kxnxh.cn/55573.Doc
m.qgw.kxnxh.cn/24246.Doc
m.qgw.kxnxh.cn/60644.Doc
m.qgw.kxnxh.cn/51533.Doc
m.qgw.kxnxh.cn/06684.Doc
m.qgw.kxnxh.cn/19511.Doc
m.qgw.kxnxh.cn/26668.Doc
m.qgw.kxnxh.cn/04488.Doc
m.qgw.kxnxh.cn/68260.Doc
m.qgw.kxnxh.cn/46828.Doc
m.qgw.kxnxh.cn/64826.Doc
m.qgq.msfsx.cn/80000.Doc
m.qgq.msfsx.cn/91595.Doc
m.qgq.msfsx.cn/33399.Doc
m.qgq.msfsx.cn/40628.Doc
m.qgq.msfsx.cn/44642.Doc
m.qgq.msfsx.cn/86004.Doc
m.qgq.msfsx.cn/46246.Doc
m.qgq.msfsx.cn/97337.Doc
m.qgq.msfsx.cn/42004.Doc
m.qgq.msfsx.cn/93357.Doc
m.qgq.msfsx.cn/99199.Doc
m.qgq.msfsx.cn/68626.Doc
m.qgq.msfsx.cn/80828.Doc
m.qgq.msfsx.cn/88022.Doc
m.qgq.msfsx.cn/08040.Doc
m.qgq.msfsx.cn/79575.Doc
m.qgq.msfsx.cn/40062.Doc
m.qgq.msfsx.cn/79175.Doc
m.qgq.msfsx.cn/20444.Doc
m.qgq.msfsx.cn/44428.Doc
m.qfm.msfsx.cn/22200.Doc
m.qfm.msfsx.cn/42668.Doc
m.qfm.msfsx.cn/22848.Doc
m.qfm.msfsx.cn/97179.Doc
m.qfm.msfsx.cn/62426.Doc
m.qfm.msfsx.cn/64082.Doc
m.qfm.msfsx.cn/64484.Doc
m.qfm.msfsx.cn/46486.Doc
m.qfm.msfsx.cn/66268.Doc
m.qfm.msfsx.cn/02204.Doc
m.qfm.msfsx.cn/48406.Doc
m.qfm.msfsx.cn/28882.Doc
m.qfm.msfsx.cn/64226.Doc
m.qfm.msfsx.cn/08068.Doc
m.qfm.msfsx.cn/86806.Doc
m.qfm.msfsx.cn/86464.Doc
m.qfm.msfsx.cn/62488.Doc
m.qfm.msfsx.cn/28666.Doc
m.qfm.msfsx.cn/28804.Doc
m.qfm.msfsx.cn/02264.Doc
m.qfn.msfsx.cn/62806.Doc
m.qfn.msfsx.cn/11519.Doc
m.qfn.msfsx.cn/08026.Doc
m.qfn.msfsx.cn/68024.Doc
m.qfn.msfsx.cn/00626.Doc
m.qfn.msfsx.cn/62862.Doc
m.qfn.msfsx.cn/48048.Doc
m.qfn.msfsx.cn/04020.Doc
m.qfn.msfsx.cn/88068.Doc
m.qfn.msfsx.cn/22266.Doc
m.qfn.msfsx.cn/75313.Doc
m.qfn.msfsx.cn/44646.Doc
m.qfn.msfsx.cn/46000.Doc
m.qfn.msfsx.cn/88686.Doc
m.qfn.msfsx.cn/39395.Doc
m.qfn.msfsx.cn/24006.Doc
m.qfn.msfsx.cn/39797.Doc
m.qfn.msfsx.cn/33997.Doc
m.qfn.msfsx.cn/11913.Doc
m.qfn.msfsx.cn/35319.Doc
m.qfb.msfsx.cn/68486.Doc
m.qfb.msfsx.cn/66280.Doc
m.qfb.msfsx.cn/33319.Doc
m.qfb.msfsx.cn/44282.Doc
m.qfb.msfsx.cn/00248.Doc
m.qfb.msfsx.cn/60024.Doc
m.qfb.msfsx.cn/75971.Doc
m.qfb.msfsx.cn/86620.Doc
m.qfb.msfsx.cn/42800.Doc
m.qfb.msfsx.cn/84426.Doc
m.qfb.msfsx.cn/37591.Doc
m.qfb.msfsx.cn/66444.Doc
m.qfb.msfsx.cn/28040.Doc
m.qfb.msfsx.cn/66284.Doc
m.qfb.msfsx.cn/08820.Doc
m.qfb.msfsx.cn/46488.Doc
m.qfb.msfsx.cn/24282.Doc
m.qfb.msfsx.cn/66466.Doc
m.qfb.msfsx.cn/59157.Doc
m.qfb.msfsx.cn/24606.Doc
m.qfv.msfsx.cn/86242.Doc
m.qfv.msfsx.cn/06648.Doc
m.qfv.msfsx.cn/28482.Doc
m.qfv.msfsx.cn/35757.Doc
m.qfv.msfsx.cn/84602.Doc
m.qfv.msfsx.cn/13531.Doc
m.qfv.msfsx.cn/48262.Doc
m.qfv.msfsx.cn/40406.Doc
m.qfv.msfsx.cn/80442.Doc
m.qfv.msfsx.cn/79375.Doc
m.qfv.msfsx.cn/40844.Doc
m.qfv.msfsx.cn/68088.Doc
m.qfv.msfsx.cn/84628.Doc
m.qfv.msfsx.cn/55979.Doc
m.qfv.msfsx.cn/00222.Doc
m.qfv.msfsx.cn/75939.Doc
m.qfv.msfsx.cn/84228.Doc
m.qfv.msfsx.cn/91579.Doc
m.qfv.msfsx.cn/82404.Doc
m.qfv.msfsx.cn/55777.Doc
m.qfc.msfsx.cn/62044.Doc
m.qfc.msfsx.cn/44682.Doc
m.qfc.msfsx.cn/44606.Doc
m.qfc.msfsx.cn/88220.Doc
m.qfc.msfsx.cn/59539.Doc
m.qfc.msfsx.cn/44460.Doc
m.qfc.msfsx.cn/19751.Doc
m.qfc.msfsx.cn/48626.Doc
m.qfc.msfsx.cn/40840.Doc
m.qfc.msfsx.cn/48260.Doc
m.qfc.msfsx.cn/77113.Doc
m.qfc.msfsx.cn/80008.Doc
m.qfc.msfsx.cn/82400.Doc
m.qfc.msfsx.cn/64086.Doc
m.qfc.msfsx.cn/26842.Doc
m.qfc.msfsx.cn/24884.Doc
m.qfc.msfsx.cn/33157.Doc
m.qfc.msfsx.cn/24042.Doc
m.qfc.msfsx.cn/46222.Doc
m.qfc.msfsx.cn/95993.Doc
m.qfx.msfsx.cn/91571.Doc
m.qfx.msfsx.cn/46604.Doc
m.qfx.msfsx.cn/66680.Doc
m.qfx.msfsx.cn/86444.Doc
m.qfx.msfsx.cn/33331.Doc
m.qfx.msfsx.cn/71395.Doc
m.qfx.msfsx.cn/20064.Doc
m.qfx.msfsx.cn/66664.Doc
m.qfx.msfsx.cn/42208.Doc
m.qfx.msfsx.cn/20284.Doc
m.qfx.msfsx.cn/20422.Doc
m.qfx.msfsx.cn/66888.Doc
m.qfx.msfsx.cn/80804.Doc
m.qfx.msfsx.cn/44484.Doc
m.qfx.msfsx.cn/28400.Doc
m.qfx.msfsx.cn/00242.Doc
m.qfx.msfsx.cn/26864.Doc
m.qfx.msfsx.cn/24468.Doc
m.qfx.msfsx.cn/37791.Doc
m.qfx.msfsx.cn/59313.Doc
m.qfz.msfsx.cn/68862.Doc
m.qfz.msfsx.cn/82042.Doc
m.qfz.msfsx.cn/46202.Doc
m.qfz.msfsx.cn/00606.Doc
m.qfz.msfsx.cn/77931.Doc
m.qfz.msfsx.cn/48666.Doc
m.qfz.msfsx.cn/62604.Doc
m.qfz.msfsx.cn/48868.Doc
m.qfz.msfsx.cn/44062.Doc
m.qfz.msfsx.cn/64680.Doc
m.qfz.msfsx.cn/44228.Doc
m.qfz.msfsx.cn/66664.Doc
m.qfz.msfsx.cn/93511.Doc
m.qfz.msfsx.cn/55753.Doc
m.qfz.msfsx.cn/64684.Doc
m.qfz.msfsx.cn/62246.Doc
m.qfz.msfsx.cn/46088.Doc
m.qfz.msfsx.cn/84402.Doc
m.qfz.msfsx.cn/46800.Doc
m.qfz.msfsx.cn/44228.Doc
m.qfl.msfsx.cn/13939.Doc
m.qfl.msfsx.cn/84462.Doc
m.qfl.msfsx.cn/88620.Doc
m.qfl.msfsx.cn/22620.Doc
m.qfl.msfsx.cn/57777.Doc
m.qfl.msfsx.cn/46086.Doc
m.qfl.msfsx.cn/44822.Doc
m.qfl.msfsx.cn/51517.Doc
m.qfl.msfsx.cn/88604.Doc
m.qfl.msfsx.cn/28228.Doc
m.qfl.msfsx.cn/28802.Doc
m.qfl.msfsx.cn/68088.Doc
m.qfl.msfsx.cn/60620.Doc
m.qfl.msfsx.cn/86884.Doc
m.qfl.msfsx.cn/53977.Doc
m.qfl.msfsx.cn/48242.Doc
m.qfl.msfsx.cn/82424.Doc
m.qfl.msfsx.cn/20440.Doc
m.qfl.msfsx.cn/28008.Doc
m.qfl.msfsx.cn/91391.Doc
m.qfk.msfsx.cn/57597.Doc
m.qfk.msfsx.cn/64042.Doc
m.qfk.msfsx.cn/68460.Doc
m.qfk.msfsx.cn/46848.Doc
m.qfk.msfsx.cn/04846.Doc
m.qfk.msfsx.cn/08244.Doc
m.qfk.msfsx.cn/28242.Doc
m.qfk.msfsx.cn/84660.Doc
m.qfk.msfsx.cn/39571.Doc
m.qfk.msfsx.cn/02446.Doc
m.qfk.msfsx.cn/64400.Doc
m.qfk.msfsx.cn/82208.Doc
m.qfk.msfsx.cn/68488.Doc
m.qfk.msfsx.cn/24462.Doc
m.qfk.msfsx.cn/84228.Doc
m.qfk.msfsx.cn/88428.Doc
m.qfk.msfsx.cn/28026.Doc
m.qfk.msfsx.cn/93315.Doc
m.qfk.msfsx.cn/88642.Doc
m.qfk.msfsx.cn/20068.Doc
m.qfj.msfsx.cn/51959.Doc
m.qfj.msfsx.cn/66204.Doc
m.qfj.msfsx.cn/62040.Doc
m.qfj.msfsx.cn/06420.Doc
m.qfj.msfsx.cn/20620.Doc
m.qfj.msfsx.cn/86640.Doc
m.qfj.msfsx.cn/48840.Doc
m.qfj.msfsx.cn/20622.Doc
m.qfj.msfsx.cn/66428.Doc
m.qfj.msfsx.cn/71995.Doc
m.qfj.msfsx.cn/42864.Doc
m.qfj.msfsx.cn/80004.Doc
m.qfj.msfsx.cn/06204.Doc
m.qfj.msfsx.cn/60802.Doc
m.qfj.msfsx.cn/22604.Doc
m.qfj.msfsx.cn/77599.Doc
m.qfj.msfsx.cn/97977.Doc
m.qfj.msfsx.cn/46282.Doc
m.qfj.msfsx.cn/46220.Doc
m.qfj.msfsx.cn/68464.Doc
m.qfh.msfsx.cn/08044.Doc
m.qfh.msfsx.cn/73557.Doc
m.qfh.msfsx.cn/99375.Doc
m.qfh.msfsx.cn/15135.Doc
m.qfh.msfsx.cn/24222.Doc
m.qfh.msfsx.cn/00028.Doc
m.qfh.msfsx.cn/68224.Doc
m.qfh.msfsx.cn/93977.Doc
