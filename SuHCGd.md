赣参已俨呛


  
不能。JVM不自动检测或解锁死锁，ThreadMXBean.findDeadlockedThreads()只诊断并返回死锁线程ID列表，不终止线程；需要人工干预或预防，只检测synchronized锁，不覆盖reentrantlock等显式锁。

死锁发生时 JVM 能自动检测中断吗？
不能。JVM 死锁不会主动检测或解除，Thread 不提供死锁恢复机制。你看到的 ThreadMXBean.findDeadlockedThreads() 它只是一个诊断工具，回到死锁线程 ID 列表，但任何线程都不会终止——必须手动干预或设计预防策略。
常见错误现象：程序卡住，CPU 占用率低，多个线程长期处于 BLOCKED 或 WAITING 同一组锁的嵌套要求在堆栈中反复出现。

使用 jstack  可以快速识别死锁(输出末尾带) “Found 1 deadlock.”）
JDK 自带的 ThreadMXBean 编程检查可在运行时进行：ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] ids = bean.findDeadlockedThreads(); // 返回 null 表示无死锁
if (ids != null) {
ThreadInfo[] infos = bean.getThreadInfo(ids, true, true);
// 打印线程名称、锁定对象、堆栈等
}
注：该方法仅进行检测 synchronized 锁，不覆盖 java.util.concurrent.locks.Lock 子类（如 ReentrantLock）显式锁锁

如何避免 synchronized 嵌套锁造成的死锁？
根本原则是破坏死锁四条件下的“循环等待”：确保所有线程以相同的顺序获得多个锁。
典型反例：transfer(Account from, Account to, int amount) 中，线程 A 先锁 from 再锁 to，线程 B 反过来先锁 to 再锁 from，很容易触发死锁。
；
			
		

统一锁定顺序：按对象哈希值排序，强制所有线程按相同逻辑获取锁定private void transfer(Account a, Account b, int amount) {
Account first = System.identityHashCode(a) 
避免在持有锁时使用外部方法（特别是不可控的回调或重载方法），以防止隐藏锁的竞争
慎用 synchronized(this) 或 synchronized 实例方法——它们暴露锁对象，外部代码可能意外参与同步竞争

使用 ReentrantLock 怎样安全地尝试获得多个锁？
ReentrantLock 提供 tryLock()，这是避免死锁的关键能力；而且 synchronized 没有相应的机制。
场景：需要同时持有两个 Lock 对象可以操作资源，但不能阻止等待其中一个。

用 tryLock(long, TimeUnit) 如果设置超时，失败将被释放并被锁定、重试或退回if (lock1.tryLock(10, TimeUnit.MILLISECONDS)) {
try {
if (lock2.tryLock(10, TimeUnit.MILLISECONDS)) {
try {
// 临界区域的安全执行
} finally {
lock2.unlock();
}
}
} finally {
lock1.unlock();
}
}
请注意，锁释放顺序不必与获取顺序严格相反，但必须确保每个顺序 lock() 都有对应 unlock()，推荐用 try-finally 包裹
不要在 lock() 成功后、tryLock() 失败前做耗时操作，否则会延长锁持时间，增加冲突概率


为什么用 wait/notify 也容易引起死锁？
这不是因为锁本身，而是因为线程在错误状态下进入等待状态，或者没有按照协议醒来，导致合作逻辑卡住——这是一个逻辑锁（livelock / starvation 的变种），jstack 不会标记为 “deadlock但表现相似。
典型错误:调用未持有锁的情况 wait()（抛 IllegalMonitorStateException），或者在非循环条件下直接检查非循环条件 wait()，错过唤醒信号会导致错过。

必须在 synchronized 块内调用 wait() 和 notify()

必须使用等待条件 while 而非 if，防止虚假唤醒synchronized (queue) {
while (queue.isEmpty()) {
queue.wait(); // 等待非空
}
return queue.remove();
}
唤醒方应调用 notifyAll() 而非 notify()，除非你能 100% 确保只有一个等待线程和唯一的条件


死锁最麻烦的地方不是发现，而是复制——它依赖于线程调度顺序，通常只在高并发压力测量或特定负载下偶尔发生。因此，不要依赖“没有问题等于没有问题”，而是在接口合同和单元测试中写下锁定顺序、加班和可重入性的约束。	 

慰缓械趁坟孟厥械馗疑诨敛置氯寡

awi.eiyve.cn/888060.Doc
awu.eiyve.cn/640082.Doc
awu.eiyve.cn/260244.Doc
awu.eiyve.cn/400806.Doc
awu.eiyve.cn/442866.Doc
awu.eiyve.cn/026828.Doc
awu.eiyve.cn/408224.Doc
awu.eiyve.cn/848866.Doc
awu.eiyve.cn/229255.Doc
awu.eiyve.cn/104545.Doc
awu.eiyve.cn/842860.Doc
awy.eiyve.cn/886244.Doc
awy.eiyve.cn/846048.Doc
awy.eiyve.cn/880480.Doc
awy.eiyve.cn/604426.Doc
awy.eiyve.cn/860088.Doc
awy.eiyve.cn/060408.Doc
awy.eiyve.cn/488688.Doc
awy.eiyve.cn/808006.Doc
awy.eiyve.cn/226288.Doc
awy.eiyve.cn/440200.Doc
awt.eiyve.cn/088084.Doc
awt.eiyve.cn/224622.Doc
awt.eiyve.cn/464268.Doc
awt.eiyve.cn/648408.Doc
awt.eiyve.cn/662420.Doc
awt.eiyve.cn/646082.Doc
awt.eiyve.cn/022206.Doc
awt.eiyve.cn/268808.Doc
awt.eiyve.cn/975171.Doc
awt.eiyve.cn/224202.Doc
awr.eiyve.cn/844600.Doc
awr.eiyve.cn/846042.Doc
awr.eiyve.cn/220204.Doc
awr.eiyve.cn/602024.Doc
awr.eiyve.cn/642804.Doc
awr.eiyve.cn/226848.Doc
awr.eiyve.cn/284444.Doc
awr.eiyve.cn/808028.Doc
awr.eiyve.cn/026848.Doc
awr.eiyve.cn/468060.Doc
awe.eiyve.cn/440808.Doc
awe.eiyve.cn/351313.Doc
awe.eiyve.cn/622008.Doc
awe.eiyve.cn/268688.Doc
awe.eiyve.cn/975063.Doc
awe.eiyve.cn/848060.Doc
awe.eiyve.cn/888648.Doc
awe.eiyve.cn/444644.Doc
awe.eiyve.cn/626024.Doc
awe.eiyve.cn/493268.Doc
aww.eiyve.cn/893854.Doc
aww.eiyve.cn/629995.Doc
aww.eiyve.cn/880446.Doc
aww.eiyve.cn/440808.Doc
aww.eiyve.cn/660406.Doc
aww.eiyve.cn/686084.Doc
aww.eiyve.cn/244860.Doc
aww.eiyve.cn/646404.Doc
aww.eiyve.cn/462866.Doc
aww.eiyve.cn/028624.Doc
awq.eiyve.cn/064400.Doc
awq.eiyve.cn/428228.Doc
awq.eiyve.cn/042404.Doc
awq.eiyve.cn/757511.Doc
awq.eiyve.cn/244008.Doc
awq.eiyve.cn/648480.Doc
awq.eiyve.cn/660462.Doc
awq.eiyve.cn/448284.Doc
awq.eiyve.cn/820040.Doc
awq.eiyve.cn/664040.Doc
aqm.eiyve.cn/408622.Doc
aqm.eiyve.cn/002460.Doc
aqm.eiyve.cn/442202.Doc
aqm.eiyve.cn/848864.Doc
aqm.eiyve.cn/606604.Doc
aqm.eiyve.cn/280684.Doc
aqm.eiyve.cn/400820.Doc
aqm.eiyve.cn/228802.Doc
aqm.eiyve.cn/204260.Doc
aqm.eiyve.cn/068440.Doc
aqn.eiyve.cn/686640.Doc
aqn.eiyve.cn/648086.Doc
aqn.eiyve.cn/426600.Doc
aqn.eiyve.cn/488666.Doc
aqn.eiyve.cn/082682.Doc
aqn.eiyve.cn/406802.Doc
aqn.eiyve.cn/848464.Doc
aqn.eiyve.cn/266040.Doc
aqn.eiyve.cn/448826.Doc
aqn.eiyve.cn/000224.Doc
aqb.eiyve.cn/179139.Doc
aqb.eiyve.cn/808206.Doc
aqb.eiyve.cn/446264.Doc
aqb.eiyve.cn/860886.Doc
aqb.eiyve.cn/826222.Doc
aqb.eiyve.cn/880008.Doc
aqb.eiyve.cn/048284.Doc
aqb.eiyve.cn/204888.Doc
aqb.eiyve.cn/024262.Doc
aqb.eiyve.cn/648088.Doc
aqv.eiyve.cn/822824.Doc
aqv.eiyve.cn/444288.Doc
aqv.eiyve.cn/484822.Doc
aqv.eiyve.cn/888244.Doc
aqv.eiyve.cn/246066.Doc
aqv.eiyve.cn/044206.Doc
aqv.eiyve.cn/246224.Doc
aqv.eiyve.cn/846260.Doc
aqv.eiyve.cn/905609.Doc
aqv.eiyve.cn/646264.Doc
aqc.eiyve.cn/919876.Doc
aqc.eiyve.cn/523112.Doc
aqc.eiyve.cn/688600.Doc
aqc.eiyve.cn/426848.Doc
aqc.eiyve.cn/844040.Doc
aqc.eiyve.cn/060886.Doc
aqc.eiyve.cn/848668.Doc
aqc.eiyve.cn/620442.Doc
aqc.eiyve.cn/082666.Doc
aqc.eiyve.cn/379396.Doc
aqx.eiyve.cn/222618.Doc
aqx.eiyve.cn/077524.Doc
aqx.eiyve.cn/406464.Doc
aqx.eiyve.cn/060461.Doc
aqx.eiyve.cn/064668.Doc
aqx.eiyve.cn/224048.Doc
aqx.eiyve.cn/404204.Doc
aqx.eiyve.cn/200844.Doc
aqx.eiyve.cn/208402.Doc
aqx.eiyve.cn/406006.Doc
aqz.eiyve.cn/884228.Doc
aqz.eiyve.cn/468684.Doc
aqz.eiyve.cn/404260.Doc
aqz.eiyve.cn/424224.Doc
aqz.eiyve.cn/442864.Doc
aqz.eiyve.cn/175377.Doc
aqz.eiyve.cn/899090.Doc
aqz.eiyve.cn/622080.Doc
aqz.eiyve.cn/460864.Doc
aqz.eiyve.cn/468241.Doc
aql.vwbnt.cn/864688.Doc
aql.vwbnt.cn/808008.Doc
aql.vwbnt.cn/842664.Doc
aql.vwbnt.cn/666426.Doc
aql.vwbnt.cn/686482.Doc
aql.vwbnt.cn/082200.Doc
aql.vwbnt.cn/264886.Doc
aql.vwbnt.cn/820864.Doc
aql.vwbnt.cn/600224.Doc
aql.vwbnt.cn/110956.Doc
aqk.vwbnt.cn/088826.Doc
aqk.vwbnt.cn/404886.Doc
aqk.vwbnt.cn/688220.Doc
aqk.vwbnt.cn/266242.Doc
aqk.vwbnt.cn/088864.Doc
aqk.vwbnt.cn/628200.Doc
aqk.vwbnt.cn/408044.Doc
aqk.vwbnt.cn/288826.Doc
aqk.vwbnt.cn/813584.Doc
aqk.vwbnt.cn/063417.Doc
aqj.vwbnt.cn/868424.Doc
aqj.vwbnt.cn/848468.Doc
aqj.vwbnt.cn/280660.Doc
aqj.vwbnt.cn/664828.Doc
aqj.vwbnt.cn/682440.Doc
aqj.vwbnt.cn/543809.Doc
aqj.vwbnt.cn/248804.Doc
aqj.vwbnt.cn/662486.Doc
aqj.vwbnt.cn/204622.Doc
aqj.vwbnt.cn/193753.Doc
aqh.vwbnt.cn/400822.Doc
aqh.vwbnt.cn/646600.Doc
aqh.vwbnt.cn/264882.Doc
aqh.vwbnt.cn/086460.Doc
aqh.vwbnt.cn/262886.Doc
aqh.vwbnt.cn/624268.Doc
aqh.vwbnt.cn/206426.Doc
aqh.vwbnt.cn/002432.Doc
aqh.vwbnt.cn/620022.Doc
aqh.vwbnt.cn/828688.Doc
aqg.vwbnt.cn/200662.Doc
aqg.vwbnt.cn/344073.Doc
aqg.vwbnt.cn/448860.Doc
aqg.vwbnt.cn/981482.Doc
aqg.vwbnt.cn/424804.Doc
aqg.vwbnt.cn/848402.Doc
aqg.vwbnt.cn/606042.Doc
aqg.vwbnt.cn/868268.Doc
aqg.vwbnt.cn/860248.Doc
aqg.vwbnt.cn/393739.Doc
aqf.vwbnt.cn/006608.Doc
aqf.vwbnt.cn/594473.Doc
aqf.vwbnt.cn/664266.Doc
aqf.vwbnt.cn/242624.Doc
aqf.vwbnt.cn/042246.Doc
aqf.vwbnt.cn/802262.Doc
aqf.vwbnt.cn/088024.Doc
aqf.vwbnt.cn/660424.Doc
aqf.vwbnt.cn/846646.Doc
aqf.vwbnt.cn/666206.Doc
aqd.vwbnt.cn/848066.Doc
aqd.vwbnt.cn/844444.Doc
aqd.vwbnt.cn/865555.Doc
aqd.vwbnt.cn/402284.Doc
aqd.vwbnt.cn/640466.Doc
aqd.vwbnt.cn/020062.Doc
aqd.vwbnt.cn/006442.Doc
aqd.vwbnt.cn/802266.Doc
aqd.vwbnt.cn/222068.Doc
aqd.vwbnt.cn/484860.Doc
aqs.vwbnt.cn/848208.Doc
aqs.vwbnt.cn/782306.Doc
aqs.vwbnt.cn/127378.Doc
aqs.vwbnt.cn/620266.Doc
aqs.vwbnt.cn/280280.Doc
aqs.vwbnt.cn/886602.Doc
aqs.vwbnt.cn/248448.Doc
aqs.vwbnt.cn/488246.Doc
aqs.vwbnt.cn/268644.Doc
aqs.vwbnt.cn/266402.Doc
aqa.vwbnt.cn/626060.Doc
aqa.vwbnt.cn/006466.Doc
aqa.vwbnt.cn/286084.Doc
aqa.vwbnt.cn/288880.Doc
aqa.vwbnt.cn/286486.Doc
aqa.vwbnt.cn/646080.Doc
aqa.vwbnt.cn/088480.Doc
aqa.vwbnt.cn/002080.Doc
aqa.vwbnt.cn/246208.Doc
aqa.vwbnt.cn/860842.Doc
aqp.vwbnt.cn/688404.Doc
aqp.vwbnt.cn/446080.Doc
aqp.vwbnt.cn/488426.Doc
aqp.vwbnt.cn/662220.Doc
aqp.vwbnt.cn/840460.Doc
aqp.vwbnt.cn/246242.Doc
aqp.vwbnt.cn/028802.Doc
aqp.vwbnt.cn/884608.Doc
aqp.vwbnt.cn/082488.Doc
aqp.vwbnt.cn/044002.Doc
aqo.vwbnt.cn/444068.Doc
aqo.vwbnt.cn/208240.Doc
aqo.vwbnt.cn/047565.Doc
aqo.vwbnt.cn/066600.Doc
aqo.vwbnt.cn/644824.Doc
aqo.vwbnt.cn/444264.Doc
aqo.vwbnt.cn/308916.Doc
aqo.vwbnt.cn/286808.Doc
aqo.vwbnt.cn/404628.Doc
aqo.vwbnt.cn/686844.Doc
aqi.vwbnt.cn/664206.Doc
aqi.vwbnt.cn/824804.Doc
aqi.vwbnt.cn/840020.Doc
aqi.vwbnt.cn/262842.Doc
aqi.vwbnt.cn/446840.Doc
aqi.vwbnt.cn/206642.Doc
aqi.vwbnt.cn/286288.Doc
aqi.vwbnt.cn/886804.Doc
aqi.vwbnt.cn/808046.Doc
aqi.vwbnt.cn/887498.Doc
aqu.vwbnt.cn/004288.Doc
aqu.vwbnt.cn/359351.Doc
aqu.vwbnt.cn/571296.Doc
aqu.vwbnt.cn/135877.Doc
aqu.vwbnt.cn/446280.Doc
aqu.vwbnt.cn/408482.Doc
aqu.vwbnt.cn/020404.Doc
aqu.vwbnt.cn/646626.Doc
aqu.vwbnt.cn/609162.Doc
aqu.vwbnt.cn/011519.Doc
aqy.vwbnt.cn/371799.Doc
aqy.vwbnt.cn/882280.Doc
aqy.vwbnt.cn/564879.Doc
aqy.vwbnt.cn/668453.Doc
aqy.vwbnt.cn/424000.Doc
aqy.vwbnt.cn/480599.Doc
aqy.vwbnt.cn/822824.Doc
aqy.vwbnt.cn/862068.Doc
aqy.vwbnt.cn/739915.Doc
aqy.vwbnt.cn/842046.Doc
aqt.vwbnt.cn/448406.Doc
aqt.vwbnt.cn/606533.Doc
aqt.vwbnt.cn/750605.Doc
aqt.vwbnt.cn/359973.Doc
aqt.vwbnt.cn/997599.Doc
aqt.vwbnt.cn/003285.Doc
aqt.vwbnt.cn/208242.Doc
aqt.vwbnt.cn/866806.Doc
aqt.vwbnt.cn/682284.Doc
aqt.vwbnt.cn/842406.Doc
aqr.vwbnt.cn/664898.Doc
aqr.vwbnt.cn/688042.Doc
aqr.vwbnt.cn/486208.Doc
aqr.vwbnt.cn/064848.Doc
aqr.vwbnt.cn/640420.Doc
aqr.vwbnt.cn/626400.Doc
aqr.vwbnt.cn/402864.Doc
aqr.vwbnt.cn/868024.Doc
aqr.vwbnt.cn/286408.Doc
aqr.vwbnt.cn/807576.Doc
aqe.vwbnt.cn/224226.Doc
aqe.vwbnt.cn/286644.Doc
aqe.vwbnt.cn/488002.Doc
aqe.vwbnt.cn/668244.Doc
aqe.vwbnt.cn/064286.Doc
aqe.vwbnt.cn/048882.Doc
aqe.vwbnt.cn/604800.Doc
aqe.vwbnt.cn/280484.Doc
aqe.vwbnt.cn/799959.Doc
aqe.vwbnt.cn/446482.Doc
aqw.vwbnt.cn/020604.Doc
aqw.vwbnt.cn/199375.Doc
aqw.vwbnt.cn/228226.Doc
aqw.vwbnt.cn/642660.Doc
aqw.vwbnt.cn/662682.Doc
aqw.vwbnt.cn/830482.Doc
aqw.vwbnt.cn/462702.Doc
aqw.vwbnt.cn/028640.Doc
aqw.vwbnt.cn/640086.Doc
aqw.vwbnt.cn/642266.Doc
aqq.vwbnt.cn/206802.Doc
aqq.vwbnt.cn/840828.Doc
aqq.vwbnt.cn/842862.Doc
aqq.vwbnt.cn/462408.Doc
aqq.vwbnt.cn/284804.Doc
aqq.vwbnt.cn/004240.Doc
aqq.vwbnt.cn/642446.Doc
aqq.vwbnt.cn/462844.Doc
aqq.vwbnt.cn/866228.Doc
aqq.vwbnt.cn/204846.Doc
pmm.vwbnt.cn/666248.Doc
pmm.vwbnt.cn/168979.Doc
pmm.vwbnt.cn/088254.Doc
pmm.vwbnt.cn/006842.Doc
pmm.vwbnt.cn/280284.Doc
pmm.vwbnt.cn/577933.Doc
pmm.vwbnt.cn/047952.Doc
pmm.vwbnt.cn/454126.Doc
pmm.vwbnt.cn/308510.Doc
pmm.vwbnt.cn/824046.Doc
pmn.vwbnt.cn/028842.Doc
pmn.vwbnt.cn/446282.Doc
pmn.vwbnt.cn/042440.Doc
pmn.vwbnt.cn/335955.Doc
pmn.vwbnt.cn/719955.Doc
pmn.vwbnt.cn/246608.Doc
pmn.vwbnt.cn/600444.Doc
pmn.vwbnt.cn/759519.Doc
pmn.vwbnt.cn/255946.Doc
pmn.vwbnt.cn/846086.Doc
pmb.vwbnt.cn/452178.Doc
pmb.vwbnt.cn/288848.Doc
pmb.vwbnt.cn/460806.Doc
pmb.vwbnt.cn/268024.Doc
pmb.vwbnt.cn/826606.Doc
pmb.vwbnt.cn/488646.Doc
pmb.vwbnt.cn/640606.Doc
pmb.vwbnt.cn/882086.Doc
pmb.vwbnt.cn/460826.Doc
pmb.vwbnt.cn/888422.Doc
pmv.vwbnt.cn/389543.Doc
pmv.vwbnt.cn/346893.Doc
pmv.vwbnt.cn/626640.Doc
pmv.vwbnt.cn/860264.Doc
pmv.vwbnt.cn/020826.Doc
pmv.vwbnt.cn/973777.Doc
pmv.vwbnt.cn/066288.Doc
pmv.vwbnt.cn/808266.Doc
pmv.vwbnt.cn/710303.Doc
pmv.vwbnt.cn/400882.Doc
pmc.vwbnt.cn/246026.Doc
pmc.vwbnt.cn/444646.Doc
pmc.vwbnt.cn/626888.Doc
pmc.vwbnt.cn/400088.Doc
pmc.vwbnt.cn/379395.Doc
pmc.vwbnt.cn/768104.Doc
pmc.vwbnt.cn/406662.Doc
pmc.vwbnt.cn/650567.Doc
pmc.vwbnt.cn/682064.Doc
pmc.vwbnt.cn/226664.Doc
pmx.vwbnt.cn/824864.Doc
pmx.vwbnt.cn/480426.Doc
pmx.vwbnt.cn/404464.Doc
pmx.vwbnt.cn/426028.Doc
pmx.vwbnt.cn/444200.Doc
pmx.vwbnt.cn/640224.Doc
pmx.vwbnt.cn/026606.Doc
pmx.vwbnt.cn/068204.Doc
pmx.vwbnt.cn/004602.Doc
pmx.vwbnt.cn/422862.Doc
pmz.vwbnt.cn/660569.Doc
pmz.vwbnt.cn/624828.Doc
pmz.vwbnt.cn/268468.Doc
pmz.vwbnt.cn/004668.Doc
