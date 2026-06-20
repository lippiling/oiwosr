途幽姓恼匦


  
ThreadMXBean 只能检测 monitor 锁死锁，返回死锁线程 ID 数组；对 ReentrantLock 默认无效，公平模式和手动遍历需要显式打开；jstack 的“Found N deadlock.“是确定性结论，需要检查 waiting to lock 与 locked 地址一致性。

死锁发生时 ThreadMXBean 能查到什么
Java 自带的 ThreadMXBean 它是唯一一种在运行过程中不侵入检测死锁的官方机制。它不预测或拦截，只扫描所有线程的锁持有/等待关系，然后在死锁形成后返回一个 long[] 数组-里面有一个死锁线程 threadId。
关键点是:它只能发现:「相互拥有对方需要的东西 monitor 锁」等待循环，是的 ReentrantLock 默认不生效(除非公平模式明确开启，否则配合 getOwner() 等待手动遍历)。


ThreadMXBean.findDeadlockedThreads() 返回 null 说明死锁没有检测到；返回空数组表示有死锁，但不包括同步器(如 ReentrantLock）
必须通过 ManagementFactory.getThreadMXBean() 获取例子，而且 JVM 需要启用监控(默认开启)
该方法可触发全局线程状态快照，高并发下有轻微性能抖动，不适合高频轮询

ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] deadlockedIds = bean.findDeadlockedThreads(); // 注意：仅限 monitor 锁
if (deadlockedIds != null && deadlockedIds.length > 0) {
for (long id : deadlockedIds) {
ThreadInfo info = bean.getThreadInfo(id, Integer.MAX_VALUE);
System.out.println("Deadlocked thread: " + info.getThreadName());
}
}

用 jstack 定位死锁现场的真实输出特征
jstack  它是最可靠的在线死锁诊断命令，其输出「Found 1 deadlock.」段落不是提示，而是结论性证据——只要出现，就代表 JVM 已确认设置死锁。
在真实输出中，我们关注三种信息：线程名waiting to lock 后16进制地址和同一地址出现在另一个地方 locked 好的。这两个地址是一致的，形成闭环。
			
		
；

不要只看「java.lang.Thread.State: BLOCKED」——大量的阻塞不等于死锁，必须有锁地址链等待

jstack -l 会额外打印 AbstractOwnableSynchronizer 信息，对 ReentrantLock 死锁有效
如果 jstack 输出里没有「Found N deadlock.」但有多处 parking to wait for，可能是 LockSupport.park() 活锁或资源饥饿不是死锁造成的


synchronized 为什么嵌套顺序不一致会导致死锁的风险？
“循环等待”在死锁四的必要条件下 Java 嵌套是最常见的 synchronized 锁定顺序不一致触发。这不是概率问题，而是逻辑不可避免的：只要两个线程以不同的顺序申请同一组对象锁，并在释放前申请下一个，就满足死锁的所有条件。

典型错误模式：threadA synchronized(obj1) → synchronized(obj2)，而 threadB synchronized(obj2) → synchronized(obj1)

即使 obj1 和 obj2 这是一个不同类型的例子。只要多个线程交叉访问，风险就会存在
加锁顺序不能通过“先判断再加锁”来避免——因为判断本身也需要同步，否则判断结果可能会过期

// 危险：依赖调用方的顺序不能保证整体一致
void transfer(Account from, Account to, int amount) {
synchronized(from) {
synchronized(to) { // ← 顺序由参数决定，不可控
from.withdraw(amount);
to.deposit(amount);
}
}
}

用 tryLock() 主动打破死锁循环的实际操作要点
ReentrantLock.tryLock(long, TimeUnit) 这是为数不多的能够从代码层面主动防止死锁的手段之一。它不强制阻塞，加班时放弃，从而切断循环等待链。但这不是一种通用的解药，在使用时必须处理商业语义回报。

必须指定超时间(例如(例如) 100, TimeUnit.MILLISECONDS），设为 0 等于非阻塞试验，设置为非阻塞试验 -1 或不带参的 tryLock() 仍然是阻碍行为
锁失败后不能简单重试——重试可能再次进入相同的竞争路径；记录日志、降级处理或抛出上级决策的具体异常
如果涉及多把锁，应按固定顺序申请，并在任何失败时释放已持有的锁（请注意） try-finally 保证）

ReentrantLock lock1 = new ReentrantLock();
ReentrantLock lock2 = new ReentrantLock();

boolean transfer(Account from, Account to, int amount) throws InterruptedException {
if (!lock1.tryLock(100, TimeUnit.MILLISECONDS)) return false;
try {
if (!lock2.tryLock(100, TimeUnit.MILLISECONDS)) return false;
try {
from.withdraw(amount);
to.deposit(amount);
return true;
} finally {
lock2.unlock();
}
} finally {
lock1.unlock();
}
}

死锁不是偶然的异常，而是设计缺陷的确定性暴露。真正难以防范的不是 monitor 锁冲突是混合使用的 synchronized、ReentrantLock、StampedLock 甚至 wait()/notify() 当锁的边界和所有权转移变得模糊时 jstack 只能显示部分线索。	 

慕赌膳日难呕练北蝗钨苑诵貉滦丶

tv.blog.yqvyet.cn/Article/details/428024.sHtML
tv.blog.yqvyet.cn/Article/details/200202.sHtML
tv.blog.yqvyet.cn/Article/details/466660.sHtML
tv.blog.yqvyet.cn/Article/details/242862.sHtML
tv.blog.yqvyet.cn/Article/details/644020.sHtML
tv.blog.yqvyet.cn/Article/details/915357.sHtML
tv.blog.yqvyet.cn/Article/details/886880.sHtML
tv.blog.yqvyet.cn/Article/details/648246.sHtML
tv.blog.yqvyet.cn/Article/details/266862.sHtML
tv.blog.yqvyet.cn/Article/details/779933.sHtML
tv.blog.yqvyet.cn/Article/details/480882.sHtML
tv.blog.yqvyet.cn/Article/details/008286.sHtML
tv.blog.yqvyet.cn/Article/details/200262.sHtML
tv.blog.yqvyet.cn/Article/details/115335.sHtML
tv.blog.yqvyet.cn/Article/details/913971.sHtML
tv.blog.yqvyet.cn/Article/details/519759.sHtML
tv.blog.yqvyet.cn/Article/details/482224.sHtML
tv.blog.yqvyet.cn/Article/details/868264.sHtML
tv.blog.yqvyet.cn/Article/details/860200.sHtML
tv.blog.yqvyet.cn/Article/details/428420.sHtML
tv.blog.yqvyet.cn/Article/details/313931.sHtML
tv.blog.yqvyet.cn/Article/details/973311.sHtML
tv.blog.yqvyet.cn/Article/details/808622.sHtML
tv.blog.yqvyet.cn/Article/details/937593.sHtML
tv.blog.yqvyet.cn/Article/details/626882.sHtML
tv.blog.yqvyet.cn/Article/details/311511.sHtML
tv.blog.yqvyet.cn/Article/details/888042.sHtML
tv.blog.yqvyet.cn/Article/details/591777.sHtML
tv.blog.yqvyet.cn/Article/details/060282.sHtML
tv.blog.yqvyet.cn/Article/details/535975.sHtML
tv.blog.yqvyet.cn/Article/details/995959.sHtML
tv.blog.yqvyet.cn/Article/details/688446.sHtML
tv.blog.yqvyet.cn/Article/details/913573.sHtML
tv.blog.yqvyet.cn/Article/details/797957.sHtML
tv.blog.yqvyet.cn/Article/details/088464.sHtML
tv.blog.yqvyet.cn/Article/details/911331.sHtML
tv.blog.yqvyet.cn/Article/details/575993.sHtML
tv.blog.yqvyet.cn/Article/details/393911.sHtML
tv.blog.yqvyet.cn/Article/details/020806.sHtML
tv.blog.yqvyet.cn/Article/details/046080.sHtML
tv.blog.yqvyet.cn/Article/details/606820.sHtML
tv.blog.yqvyet.cn/Article/details/240002.sHtML
tv.blog.yqvyet.cn/Article/details/199973.sHtML
tv.blog.yqvyet.cn/Article/details/422666.sHtML
tv.blog.yqvyet.cn/Article/details/648204.sHtML
tv.blog.yqvyet.cn/Article/details/397939.sHtML
tv.blog.yqvyet.cn/Article/details/771171.sHtML
tv.blog.yqvyet.cn/Article/details/995775.sHtML
tv.blog.yqvyet.cn/Article/details/111131.sHtML
tv.blog.yqvyet.cn/Article/details/224800.sHtML
tv.blog.yqvyet.cn/Article/details/264086.sHtML
tv.blog.yqvyet.cn/Article/details/448426.sHtML
tv.blog.yqvyet.cn/Article/details/840020.sHtML
tv.blog.yqvyet.cn/Article/details/559377.sHtML
tv.blog.yqvyet.cn/Article/details/842200.sHtML
tv.blog.yqvyet.cn/Article/details/339353.sHtML
tv.blog.yqvyet.cn/Article/details/177193.sHtML
tv.blog.yqvyet.cn/Article/details/199779.sHtML
tv.blog.yqvyet.cn/Article/details/426628.sHtML
tv.blog.yqvyet.cn/Article/details/620460.sHtML
tv.blog.yqvyet.cn/Article/details/640626.sHtML
tv.blog.yqvyet.cn/Article/details/599917.sHtML
tv.blog.yqvyet.cn/Article/details/991153.sHtML
tv.blog.yqvyet.cn/Article/details/626422.sHtML
tv.blog.yqvyet.cn/Article/details/240202.sHtML
tv.blog.yqvyet.cn/Article/details/824868.sHtML
tv.blog.yqvyet.cn/Article/details/937553.sHtML
tv.blog.yqvyet.cn/Article/details/642224.sHtML
tv.blog.yqvyet.cn/Article/details/559995.sHtML
tv.blog.yqvyet.cn/Article/details/262608.sHtML
tv.blog.yqvyet.cn/Article/details/048864.sHtML
tv.blog.yqvyet.cn/Article/details/024026.sHtML
tv.blog.yqvyet.cn/Article/details/319117.sHtML
tv.blog.yqvyet.cn/Article/details/828642.sHtML
tv.blog.yqvyet.cn/Article/details/993779.sHtML
tv.blog.yqvyet.cn/Article/details/246622.sHtML
tv.blog.yqvyet.cn/Article/details/717795.sHtML
tv.blog.yqvyet.cn/Article/details/335311.sHtML
tv.blog.yqvyet.cn/Article/details/757713.sHtML
tv.blog.yqvyet.cn/Article/details/206824.sHtML
tv.blog.yqvyet.cn/Article/details/824824.sHtML
tv.blog.yqvyet.cn/Article/details/260404.sHtML
tv.blog.yqvyet.cn/Article/details/731353.sHtML
tv.blog.yqvyet.cn/Article/details/733959.sHtML
tv.blog.yqvyet.cn/Article/details/282284.sHtML
tv.blog.yqvyet.cn/Article/details/191535.sHtML
tv.blog.yqvyet.cn/Article/details/795559.sHtML
tv.blog.yqvyet.cn/Article/details/244220.sHtML
tv.blog.yqvyet.cn/Article/details/399133.sHtML
tv.blog.yqvyet.cn/Article/details/317977.sHtML
tv.blog.yqvyet.cn/Article/details/513519.sHtML
tv.blog.yqvyet.cn/Article/details/173911.sHtML
tv.blog.yqvyet.cn/Article/details/979339.sHtML
tv.blog.yqvyet.cn/Article/details/448222.sHtML
tv.blog.yqvyet.cn/Article/details/820464.sHtML
tv.blog.yqvyet.cn/Article/details/068228.sHtML
tv.blog.yqvyet.cn/Article/details/575519.sHtML
tv.blog.yqvyet.cn/Article/details/995531.sHtML
tv.blog.yqvyet.cn/Article/details/777531.sHtML
tv.blog.yqvyet.cn/Article/details/820886.sHtML
tv.blog.yqvyet.cn/Article/details/199177.sHtML
tv.blog.yqvyet.cn/Article/details/284666.sHtML
tv.blog.yqvyet.cn/Article/details/042888.sHtML
tv.blog.yqvyet.cn/Article/details/573111.sHtML
tv.blog.yqvyet.cn/Article/details/755539.sHtML
tv.blog.yqvyet.cn/Article/details/062488.sHtML
tv.blog.yqvyet.cn/Article/details/282880.sHtML
tv.blog.yqvyet.cn/Article/details/911579.sHtML
tv.blog.yqvyet.cn/Article/details/644284.sHtML
tv.blog.yqvyet.cn/Article/details/222208.sHtML
tv.blog.yqvyet.cn/Article/details/844602.sHtML
tv.blog.yqvyet.cn/Article/details/171591.sHtML
tv.blog.yqvyet.cn/Article/details/191759.sHtML
tv.blog.yqvyet.cn/Article/details/371959.sHtML
tv.blog.yqvyet.cn/Article/details/159355.sHtML
tv.blog.yqvyet.cn/Article/details/533933.sHtML
tv.blog.yqvyet.cn/Article/details/664840.sHtML
tv.blog.yqvyet.cn/Article/details/915939.sHtML
tv.blog.yqvyet.cn/Article/details/573559.sHtML
tv.blog.yqvyet.cn/Article/details/644680.sHtML
tv.blog.yqvyet.cn/Article/details/868268.sHtML
tv.blog.yqvyet.cn/Article/details/444044.sHtML
tv.blog.yqvyet.cn/Article/details/559953.sHtML
tv.blog.yqvyet.cn/Article/details/448460.sHtML
tv.blog.yqvyet.cn/Article/details/993997.sHtML
tv.blog.yqvyet.cn/Article/details/424066.sHtML
tv.blog.yqvyet.cn/Article/details/002822.sHtML
tv.blog.yqvyet.cn/Article/details/200006.sHtML
tv.blog.yqvyet.cn/Article/details/822446.sHtML
tv.blog.yqvyet.cn/Article/details/686822.sHtML
tv.blog.yqvyet.cn/Article/details/220460.sHtML
tv.blog.yqvyet.cn/Article/details/028626.sHtML
tv.blog.yqvyet.cn/Article/details/468602.sHtML
tv.blog.yqvyet.cn/Article/details/442844.sHtML
tv.blog.yqvyet.cn/Article/details/606462.sHtML
tv.blog.yqvyet.cn/Article/details/424880.sHtML
tv.blog.yqvyet.cn/Article/details/088268.sHtML
tv.blog.yqvyet.cn/Article/details/866206.sHtML
tv.blog.yqvyet.cn/Article/details/024460.sHtML
tv.blog.yqvyet.cn/Article/details/004200.sHtML
tv.blog.yqvyet.cn/Article/details/482206.sHtML
tv.blog.yqvyet.cn/Article/details/202200.sHtML
tv.blog.yqvyet.cn/Article/details/084208.sHtML
tv.blog.yqvyet.cn/Article/details/284248.sHtML
tv.blog.yqvyet.cn/Article/details/795753.sHtML
tv.blog.yqvyet.cn/Article/details/266064.sHtML
tv.blog.yqvyet.cn/Article/details/840642.sHtML
tv.blog.yqvyet.cn/Article/details/113359.sHtML
tv.blog.yqvyet.cn/Article/details/448428.sHtML
tv.blog.yqvyet.cn/Article/details/664402.sHtML
tv.blog.yqvyet.cn/Article/details/602462.sHtML
tv.blog.yqvyet.cn/Article/details/791759.sHtML
tv.blog.yqvyet.cn/Article/details/422220.sHtML
tv.blog.yqvyet.cn/Article/details/373575.sHtML
tv.blog.yqvyet.cn/Article/details/082424.sHtML
tv.blog.yqvyet.cn/Article/details/791777.sHtML
tv.blog.yqvyet.cn/Article/details/737357.sHtML
tv.blog.yqvyet.cn/Article/details/799371.sHtML
tv.blog.yqvyet.cn/Article/details/004288.sHtML
tv.blog.yqvyet.cn/Article/details/880824.sHtML
tv.blog.yqvyet.cn/Article/details/737131.sHtML
tv.blog.yqvyet.cn/Article/details/111135.sHtML
tv.blog.yqvyet.cn/Article/details/442600.sHtML
tv.blog.yqvyet.cn/Article/details/717953.sHtML
tv.blog.yqvyet.cn/Article/details/337919.sHtML
tv.blog.yqvyet.cn/Article/details/917995.sHtML
tv.blog.yqvyet.cn/Article/details/775379.sHtML
tv.blog.yqvyet.cn/Article/details/606240.sHtML
tv.blog.yqvyet.cn/Article/details/402642.sHtML
tv.blog.yqvyet.cn/Article/details/179173.sHtML
tv.blog.yqvyet.cn/Article/details/933731.sHtML
tv.blog.yqvyet.cn/Article/details/513131.sHtML
tv.blog.yqvyet.cn/Article/details/024840.sHtML
tv.blog.yqvyet.cn/Article/details/339719.sHtML
tv.blog.yqvyet.cn/Article/details/624840.sHtML
tv.blog.yqvyet.cn/Article/details/571959.sHtML
tv.blog.yqvyet.cn/Article/details/137151.sHtML
tv.blog.yqvyet.cn/Article/details/535799.sHtML
tv.blog.yqvyet.cn/Article/details/064020.sHtML
tv.blog.yqvyet.cn/Article/details/204224.sHtML
tv.blog.yqvyet.cn/Article/details/448428.sHtML
tv.blog.yqvyet.cn/Article/details/806886.sHtML
tv.blog.yqvyet.cn/Article/details/042086.sHtML
tv.blog.yqvyet.cn/Article/details/755115.sHtML
tv.blog.yqvyet.cn/Article/details/608084.sHtML
tv.blog.yqvyet.cn/Article/details/680804.sHtML
tv.blog.yqvyet.cn/Article/details/682088.sHtML
tv.blog.yqvyet.cn/Article/details/531999.sHtML
tv.blog.yqvyet.cn/Article/details/664044.sHtML
tv.blog.yqvyet.cn/Article/details/606484.sHtML
tv.blog.yqvyet.cn/Article/details/991793.sHtML
tv.blog.yqvyet.cn/Article/details/402022.sHtML
tv.blog.yqvyet.cn/Article/details/284804.sHtML
tv.blog.yqvyet.cn/Article/details/006448.sHtML
tv.blog.yqvyet.cn/Article/details/886266.sHtML
tv.blog.yqvyet.cn/Article/details/359313.sHtML
tv.blog.yqvyet.cn/Article/details/139757.sHtML
tv.blog.yqvyet.cn/Article/details/666024.sHtML
tv.blog.yqvyet.cn/Article/details/828680.sHtML
tv.blog.yqvyet.cn/Article/details/915317.sHtML
tv.blog.yqvyet.cn/Article/details/359755.sHtML
tv.blog.yqvyet.cn/Article/details/042842.sHtML
tv.blog.yqvyet.cn/Article/details/591115.sHtML
tv.blog.yqvyet.cn/Article/details/848864.sHtML
tv.blog.yqvyet.cn/Article/details/682864.sHtML
tv.blog.yqvyet.cn/Article/details/973395.sHtML
tv.blog.yqvyet.cn/Article/details/264044.sHtML
tv.blog.yqvyet.cn/Article/details/208068.sHtML
tv.blog.yqvyet.cn/Article/details/826022.sHtML
tv.blog.yqvyet.cn/Article/details/337735.sHtML
tv.blog.yqvyet.cn/Article/details/319397.sHtML
tv.blog.yqvyet.cn/Article/details/680064.sHtML
tv.blog.yqvyet.cn/Article/details/264684.sHtML
tv.blog.yqvyet.cn/Article/details/080244.sHtML
tv.blog.yqvyet.cn/Article/details/557799.sHtML
tv.blog.yqvyet.cn/Article/details/333111.sHtML
tv.blog.yqvyet.cn/Article/details/773171.sHtML
tv.blog.yqvyet.cn/Article/details/595171.sHtML
tv.blog.yqvyet.cn/Article/details/959337.sHtML
tv.blog.yqvyet.cn/Article/details/804246.sHtML
tv.blog.yqvyet.cn/Article/details/022060.sHtML
tv.blog.yqvyet.cn/Article/details/028062.sHtML
tv.blog.yqvyet.cn/Article/details/331913.sHtML
tv.blog.yqvyet.cn/Article/details/797777.sHtML
tv.blog.yqvyet.cn/Article/details/955955.sHtML
tv.blog.yqvyet.cn/Article/details/460442.sHtML
tv.blog.yqvyet.cn/Article/details/420624.sHtML
tv.blog.yqvyet.cn/Article/details/064224.sHtML
tv.blog.yqvyet.cn/Article/details/660406.sHtML
tv.blog.yqvyet.cn/Article/details/177371.sHtML
tv.blog.yqvyet.cn/Article/details/684068.sHtML
tv.blog.yqvyet.cn/Article/details/404862.sHtML
tv.blog.yqvyet.cn/Article/details/155797.sHtML
tv.blog.yqvyet.cn/Article/details/288606.sHtML
tv.blog.yqvyet.cn/Article/details/208020.sHtML
tv.blog.yqvyet.cn/Article/details/682080.sHtML
tv.blog.yqvyet.cn/Article/details/737137.sHtML
tv.blog.yqvyet.cn/Article/details/048284.sHtML
tv.blog.yqvyet.cn/Article/details/608820.sHtML
tv.blog.yqvyet.cn/Article/details/713755.sHtML
tv.blog.yqvyet.cn/Article/details/193911.sHtML
tv.blog.yqvyet.cn/Article/details/420024.sHtML
tv.blog.yqvyet.cn/Article/details/462282.sHtML
tv.blog.yqvyet.cn/Article/details/666882.sHtML
tv.blog.yqvyet.cn/Article/details/317391.sHtML
tv.blog.yqvyet.cn/Article/details/648604.sHtML
tv.blog.yqvyet.cn/Article/details/648228.sHtML
tv.blog.yqvyet.cn/Article/details/646682.sHtML
tv.blog.yqvyet.cn/Article/details/228488.sHtML
tv.blog.yqvyet.cn/Article/details/773117.sHtML
tv.blog.yqvyet.cn/Article/details/666642.sHtML
tv.blog.yqvyet.cn/Article/details/113757.sHtML
tv.blog.yqvyet.cn/Article/details/666846.sHtML
tv.blog.yqvyet.cn/Article/details/573391.sHtML
tv.blog.yqvyet.cn/Article/details/644424.sHtML
tv.blog.yqvyet.cn/Article/details/488884.sHtML
tv.blog.yqvyet.cn/Article/details/193957.sHtML
tv.blog.yqvyet.cn/Article/details/680688.sHtML
tv.blog.yqvyet.cn/Article/details/115315.sHtML
tv.blog.yqvyet.cn/Article/details/868624.sHtML
tv.blog.yqvyet.cn/Article/details/286440.sHtML
tv.blog.yqvyet.cn/Article/details/777797.sHtML
tv.blog.yqvyet.cn/Article/details/460488.sHtML
tv.blog.yqvyet.cn/Article/details/624406.sHtML
tv.blog.yqvyet.cn/Article/details/555959.sHtML
tv.blog.yqvyet.cn/Article/details/153735.sHtML
tv.blog.yqvyet.cn/Article/details/622486.sHtML
tv.blog.yqvyet.cn/Article/details/640048.sHtML
tv.blog.yqvyet.cn/Article/details/397957.sHtML
tv.blog.yqvyet.cn/Article/details/795791.sHtML
tv.blog.yqvyet.cn/Article/details/339377.sHtML
tv.blog.yqvyet.cn/Article/details/773599.sHtML
tv.blog.yqvyet.cn/Article/details/020064.sHtML
tv.blog.yqvyet.cn/Article/details/757113.sHtML
tv.blog.yqvyet.cn/Article/details/335957.sHtML
tv.blog.yqvyet.cn/Article/details/133157.sHtML
tv.blog.yqvyet.cn/Article/details/913937.sHtML
tv.blog.yqvyet.cn/Article/details/339173.sHtML
tv.blog.yqvyet.cn/Article/details/248426.sHtML
tv.blog.yqvyet.cn/Article/details/157777.sHtML
tv.blog.yqvyet.cn/Article/details/884402.sHtML
tv.blog.yqvyet.cn/Article/details/200440.sHtML
tv.blog.yqvyet.cn/Article/details/959791.sHtML
tv.blog.yqvyet.cn/Article/details/026026.sHtML
tv.blog.yqvyet.cn/Article/details/717535.sHtML
tv.blog.yqvyet.cn/Article/details/844480.sHtML
tv.blog.yqvyet.cn/Article/details/804460.sHtML
tv.blog.yqvyet.cn/Article/details/208004.sHtML
tv.blog.yqvyet.cn/Article/details/640624.sHtML
tv.blog.yqvyet.cn/Article/details/446864.sHtML
tv.blog.yqvyet.cn/Article/details/595375.sHtML
tv.blog.yqvyet.cn/Article/details/028000.sHtML
tv.blog.yqvyet.cn/Article/details/266662.sHtML
tv.blog.yqvyet.cn/Article/details/719115.sHtML
tv.blog.yqvyet.cn/Article/details/119551.sHtML
tv.blog.yqvyet.cn/Article/details/408646.sHtML
tv.blog.yqvyet.cn/Article/details/373317.sHtML
tv.blog.yqvyet.cn/Article/details/868466.sHtML
tv.blog.yqvyet.cn/Article/details/468002.sHtML
tv.blog.yqvyet.cn/Article/details/351517.sHtML
tv.blog.yqvyet.cn/Article/details/313791.sHtML
tv.blog.yqvyet.cn/Article/details/800640.sHtML
tv.blog.yqvyet.cn/Article/details/660406.sHtML
tv.blog.yqvyet.cn/Article/details/911753.sHtML
tv.blog.yqvyet.cn/Article/details/042044.sHtML
tv.blog.yqvyet.cn/Article/details/822026.sHtML
tv.blog.yqvyet.cn/Article/details/242244.sHtML
tv.blog.yqvyet.cn/Article/details/153597.sHtML
tv.blog.yqvyet.cn/Article/details/551399.sHtML
tv.blog.yqvyet.cn/Article/details/260060.sHtML
tv.blog.yqvyet.cn/Article/details/208820.sHtML
tv.blog.yqvyet.cn/Article/details/244826.sHtML
tv.blog.yqvyet.cn/Article/details/482468.sHtML
tv.blog.yqvyet.cn/Article/details/806048.sHtML
tv.blog.yqvyet.cn/Article/details/993935.sHtML
tv.blog.yqvyet.cn/Article/details/068880.sHtML
tv.blog.yqvyet.cn/Article/details/155539.sHtML
tv.blog.yqvyet.cn/Article/details/557793.sHtML
tv.blog.yqvyet.cn/Article/details/739199.sHtML
tv.blog.yqvyet.cn/Article/details/624606.sHtML
tv.blog.yqvyet.cn/Article/details/733537.sHtML
tv.blog.yqvyet.cn/Article/details/315135.sHtML
tv.blog.yqvyet.cn/Article/details/771993.sHtML
tv.blog.yqvyet.cn/Article/details/977553.sHtML
tv.blog.yqvyet.cn/Article/details/135731.sHtML
tv.blog.yqvyet.cn/Article/details/640262.sHtML
tv.blog.yqvyet.cn/Article/details/759991.sHtML
tv.blog.yqvyet.cn/Article/details/228462.sHtML
tv.blog.yqvyet.cn/Article/details/624240.sHtML
tv.blog.yqvyet.cn/Article/details/868022.sHtML
tv.blog.yqvyet.cn/Article/details/866462.sHtML
tv.blog.yqvyet.cn/Article/details/622484.sHtML
tv.blog.yqvyet.cn/Article/details/379591.sHtML
tv.blog.yqvyet.cn/Article/details/868284.sHtML
tv.blog.yqvyet.cn/Article/details/040844.sHtML
tv.blog.yqvyet.cn/Article/details/664446.sHtML
tv.blog.yqvyet.cn/Article/details/886680.sHtML
tv.blog.yqvyet.cn/Article/details/800222.sHtML
tv.blog.yqvyet.cn/Article/details/208802.sHtML
tv.blog.yqvyet.cn/Article/details/111195.sHtML
tv.blog.yqvyet.cn/Article/details/822884.sHtML
tv.blog.yqvyet.cn/Article/details/662626.sHtML
tv.blog.yqvyet.cn/Article/details/082086.sHtML
tv.blog.yqvyet.cn/Article/details/933939.sHtML
tv.blog.yqvyet.cn/Article/details/864644.sHtML
tv.blog.yqvyet.cn/Article/details/197199.sHtML
tv.blog.yqvyet.cn/Article/details/068860.sHtML
tv.blog.yqvyet.cn/Article/details/173999.sHtML
tv.blog.yqvyet.cn/Article/details/244208.sHtML
tv.blog.yqvyet.cn/Article/details/737711.sHtML
tv.blog.yqvyet.cn/Article/details/399135.sHtML
tv.blog.yqvyet.cn/Article/details/448084.sHtML
tv.blog.yqvyet.cn/Article/details/808424.sHtML
tv.blog.yqvyet.cn/Article/details/202442.sHtML
tv.blog.yqvyet.cn/Article/details/442262.sHtML
tv.blog.yqvyet.cn/Article/details/202268.sHtML
tv.blog.yqvyet.cn/Article/details/571199.sHtML
tv.blog.yqvyet.cn/Article/details/008484.sHtML
tv.blog.yqvyet.cn/Article/details/220688.sHtML
tv.blog.yqvyet.cn/Article/details/002062.sHtML
tv.blog.yqvyet.cn/Article/details/222460.sHtML
tv.blog.yqvyet.cn/Article/details/684002.sHtML
tv.blog.yqvyet.cn/Article/details/559711.sHtML
tv.blog.yqvyet.cn/Article/details/139359.sHtML
tv.blog.yqvyet.cn/Article/details/606420.sHtML
tv.blog.yqvyet.cn/Article/details/846640.sHtML
tv.blog.yqvyet.cn/Article/details/602268.sHtML
tv.blog.yqvyet.cn/Article/details/735779.sHtML
tv.blog.yqvyet.cn/Article/details/191715.sHtML
tv.blog.yqvyet.cn/Article/details/282480.sHtML
tv.blog.yqvyet.cn/Article/details/680408.sHtML
tv.blog.yqvyet.cn/Article/details/753919.sHtML
tv.blog.yqvyet.cn/Article/details/824044.sHtML
tv.blog.yqvyet.cn/Article/details/486288.sHtML
tv.blog.yqvyet.cn/Article/details/575311.sHtML
tv.blog.yqvyet.cn/Article/details/602606.sHtML
tv.blog.yqvyet.cn/Article/details/311111.sHtML
tv.blog.yqvyet.cn/Article/details/553937.sHtML
tv.blog.yqvyet.cn/Article/details/991535.sHtML
tv.blog.yqvyet.cn/Article/details/733737.sHtML
tv.blog.yqvyet.cn/Article/details/315311.sHtML
tv.blog.yqvyet.cn/Article/details/068208.sHtML
tv.blog.yqvyet.cn/Article/details/571157.sHtML
tv.blog.yqvyet.cn/Article/details/046660.sHtML
tv.blog.yqvyet.cn/Article/details/777911.sHtML
tv.blog.yqvyet.cn/Article/details/931775.sHtML
tv.blog.yqvyet.cn/Article/details/680642.sHtML
tv.blog.yqvyet.cn/Article/details/404462.sHtML
tv.blog.yqvyet.cn/Article/details/773357.sHtML
tv.blog.yqvyet.cn/Article/details/480282.sHtML
tv.blog.yqvyet.cn/Article/details/042208.sHtML
tv.blog.yqvyet.cn/Article/details/244626.sHtML
tv.blog.yqvyet.cn/Article/details/731179.sHtML
tv.blog.yqvyet.cn/Article/details/719113.sHtML
tv.blog.yqvyet.cn/Article/details/991315.sHtML
