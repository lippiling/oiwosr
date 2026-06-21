谱厝泼嘶胶


  
该用 CyclicBarrier 而不是 CountDownLatch 场景是多线程需要反复同步到达某一点，然后共同推进，如批处理、多轮迭代或并发压力测量；因为它可以重复使用，支持屏障动作，可以响应中断和加班， CountDownLatch 一次性使用。

该用什么时候？ CyclicBarrier 而不是 CountDownLatch

需要重复多个线程「等待彼此达到某一点」再往下走，就该选了 CyclicBarrier。例如，在多线程分批处理数据、并行计算中的每一轮迭代同步、压力测试中的模拟和并发用户同时发起请求——在这些场景中，您不仅等待一次，而且可以循环多次。
CountDownLatch 是一次性门槛，倒数后永久打开；CyclicBarrier 每次都是可重置的旋转门 await() 它会被卡住，直到指定数量的线程被调用，然后集体释放。

如果你需要「复用」同一屏障(例如 10 轮计算，每轮等待 4 线程齐活)，CyclicBarrier 天然支持 reset() 或自动复用，CountDownLatch 得新建实例

CyclicBarrier 允许设置一个 Runnable 作为“屏障动作”，在最后一个动作中 N 线程到达时，放行前执行(如汇总本轮结果)，CountDownLatch 没这能力
注：如果线程在 await() 当中断时，整个屏障将被打破，其他等待线程将被抛出 BrokenBarrierException ——这不是 bug，由于设计，必须主动捕获处理


await() 调用后线程到底卡在哪里，等等
await() 在满足两个条件之一之前，它不仅仅是“睡一会儿”，它会阻挡当前线程的屏障点：调用指定数量的线程 await()，或超时/中断/屏障损坏。
关键是所有参与线程必须每次调用一次 await()，只是“到达”。漏调，多调，或者某个线程根本没有进入这种方法，都会导致其他线程永远卡住(除非设置加班)。
；

默认无超时 await() 会一直等待，强烈建议生产环境使用带超时版本:await(long timeout, TimeUnit unit)

返回值是 int：表示当前线程到达的次数是多少(从) 0 开始计数)，可用于做“线程干点第一到初始化”的逻辑
如果一个线程在等待时被等待 interrupt()，它会立即抛出 InterruptedException，屏障状态变为 broken，后续所有 await() 都会立刻抛 BrokenBarrierException


如何安全处理？ BrokenBarrierException

这种异常不是意外的错误，而是屏障被迫失效后的明确信号——可能是因为有人中断，有人加班，或者有人调用 reset()。忽略它，往往意味着后续逻辑仍然假设“每个人都同步完成了”，结果数据混乱。


只有典型的错误写法 catch InterruptedException，却把 BrokenBarrierException 往上扔，或者根本不要 catch。

必须显式 catch BrokenBarrierException，判断是否重新测试、清理资源或退出任务
如果要重用屏障，必须先调用 getNumberWaiting() 看看还有谁卡着，然后决定是否卡着。 reset()；但注意 reset() 它会唤醒所有的等待线程，并让它们抛出 BrokenBarrierException

常见坑:屏障运动（Runnable）内抛异常也会导致屏障 broken，而且异常会包装成 RuntimeException 再次触发 BrokenBarrierException


实际写作方法：最小可用的例子，具有超时和屏障动作
下面的代码模拟 3 完成一轮计算的线程协作：
CyclicBarrier barrier = new CyclicBarrier(3, () -&gt; {
System.out.println(&quot;所有线程到齐，开始汇总&quot;);
// 这里可以更新共享状态，写日志，触发下一步
});
// 每个线程：
try {
Thread.sleep(100); // 不同的耗时模拟
System.out.println(Thread.currentThread().getName() + &quot; 到达屏障&quot;);
barrier.await(5, TimeUnit.SECONDS); // 带超时，避免死等
} catch (TimeoutException e) {
System.err.println(&quot;等不及别人了，超时退出&quot;);
} catch (BrokenBarrierException e) {
System.err.println(&quot;屏障损坏，有些人可能会中断或出错&quot;);
} catch (InterruptedException e) {
Thread.currentThread().interrupt();
}
注意 barrier.await(5, TimeUnit.SECONDS) 的超时是「单次等待」上限不是整个循环的总耗时；当最后一个线程到达时，屏障动作中的逻辑只会执行一次——这很容易被误认为每轮运行多次。
真正困难的永远不会被调用 API，相反，预测哪个线程可能失败，如何防止一个线程拖累整个组，以及 broken 之后要不要重建屏障？这些没写 Javadoc 但是每天都在网上发生。	 

谠膛悍考蚁毡碧百晾褐较烈崖橇敌

m.wfv.hxbsg.cn/71715.Doc
m.wfv.hxbsg.cn/88868.Doc
m.wfv.hxbsg.cn/82442.Doc
m.wfv.hxbsg.cn/48800.Doc
m.wfv.hxbsg.cn/02202.Doc
m.wfv.hxbsg.cn/80204.Doc
m.wfv.hxbsg.cn/44248.Doc
m.wfv.hxbsg.cn/22020.Doc
m.wfv.hxbsg.cn/60422.Doc
m.wfv.hxbsg.cn/60048.Doc
m.wfc.hxbsg.cn/60406.Doc
m.wfc.hxbsg.cn/08862.Doc
m.wfc.hxbsg.cn/97759.Doc
m.wfc.hxbsg.cn/19793.Doc
m.wfc.hxbsg.cn/33131.Doc
m.wfc.hxbsg.cn/33111.Doc
m.wfc.hxbsg.cn/84624.Doc
m.wfc.hxbsg.cn/31939.Doc
m.wfc.hxbsg.cn/26208.Doc
m.wfc.hxbsg.cn/02068.Doc
m.wfc.hxbsg.cn/86668.Doc
m.wfc.hxbsg.cn/24002.Doc
m.wfc.hxbsg.cn/37135.Doc
m.wfc.hxbsg.cn/04442.Doc
m.wfc.hxbsg.cn/19959.Doc
m.wfc.hxbsg.cn/97791.Doc
m.wfc.hxbsg.cn/31733.Doc
m.wfc.hxbsg.cn/15159.Doc
m.wfc.hxbsg.cn/08044.Doc
m.wfc.hxbsg.cn/17577.Doc
m.wfx.hxbsg.cn/88004.Doc
m.wfx.hxbsg.cn/31915.Doc
m.wfx.hxbsg.cn/19713.Doc
m.wfx.hxbsg.cn/64642.Doc
m.wfx.hxbsg.cn/33755.Doc
m.wfx.hxbsg.cn/40262.Doc
m.wfx.hxbsg.cn/57515.Doc
m.wfx.hxbsg.cn/99719.Doc
m.wfx.hxbsg.cn/48622.Doc
m.wfx.hxbsg.cn/04426.Doc
m.wfx.hxbsg.cn/24200.Doc
m.wfx.hxbsg.cn/79375.Doc
m.wfx.hxbsg.cn/99737.Doc
m.wfx.hxbsg.cn/15737.Doc
m.wfx.hxbsg.cn/22862.Doc
m.wfx.hxbsg.cn/53597.Doc
m.wfx.hxbsg.cn/40484.Doc
m.wfx.hxbsg.cn/51551.Doc
m.wfx.hxbsg.cn/93319.Doc
m.wfx.hxbsg.cn/37391.Doc
m.wfz.hxbsg.cn/57595.Doc
m.wfz.hxbsg.cn/35793.Doc
m.wfz.hxbsg.cn/06060.Doc
m.wfz.hxbsg.cn/59393.Doc
m.wfz.hxbsg.cn/02260.Doc
m.wfz.hxbsg.cn/13993.Doc
m.wfz.hxbsg.cn/95371.Doc
m.wfz.hxbsg.cn/95511.Doc
m.wfz.hxbsg.cn/22800.Doc
m.wfz.hxbsg.cn/46660.Doc
m.wfz.hxbsg.cn/66262.Doc
m.wfz.hxbsg.cn/40208.Doc
m.wfz.hxbsg.cn/08466.Doc
m.wfz.hxbsg.cn/28666.Doc
m.wfz.hxbsg.cn/75399.Doc
m.wfz.hxbsg.cn/40844.Doc
m.wfz.hxbsg.cn/04608.Doc
m.wfz.hxbsg.cn/44862.Doc
m.wfz.hxbsg.cn/82826.Doc
m.wfz.hxbsg.cn/95177.Doc
m.wfl.hxbsg.cn/46662.Doc
m.wfl.hxbsg.cn/57775.Doc
m.wfl.hxbsg.cn/79557.Doc
m.wfl.hxbsg.cn/06402.Doc
m.wfl.hxbsg.cn/80008.Doc
m.wfl.hxbsg.cn/93939.Doc
m.wfl.hxbsg.cn/51775.Doc
m.wfl.hxbsg.cn/66484.Doc
m.wfl.hxbsg.cn/60006.Doc
m.wfl.hxbsg.cn/15351.Doc
m.wfl.hxbsg.cn/00440.Doc
m.wfl.hxbsg.cn/66268.Doc
m.wfl.hxbsg.cn/73517.Doc
m.wfl.hxbsg.cn/95757.Doc
m.wfl.hxbsg.cn/40620.Doc
m.wfl.hxbsg.cn/33339.Doc
m.wfl.hxbsg.cn/53715.Doc
m.wfl.hxbsg.cn/20024.Doc
m.wfl.hxbsg.cn/46222.Doc
m.wfl.hxbsg.cn/75511.Doc
m.wfk.hxbsg.cn/64066.Doc
m.wfk.hxbsg.cn/48604.Doc
m.wfk.hxbsg.cn/46026.Doc
m.wfk.hxbsg.cn/99351.Doc
m.wfk.hxbsg.cn/20682.Doc
m.wfk.hxbsg.cn/80220.Doc
m.wfk.hxbsg.cn/64044.Doc
m.wfk.hxbsg.cn/31317.Doc
m.wfk.hxbsg.cn/31175.Doc
m.wfk.hxbsg.cn/88866.Doc
m.wfk.hxbsg.cn/46000.Doc
m.wfk.hxbsg.cn/15953.Doc
m.wfk.hxbsg.cn/40244.Doc
m.wfk.hxbsg.cn/46842.Doc
m.wfk.hxbsg.cn/75333.Doc
m.wfk.hxbsg.cn/60088.Doc
m.wfk.hxbsg.cn/73991.Doc
m.wfk.hxbsg.cn/97137.Doc
m.wfk.hxbsg.cn/93155.Doc
m.wfk.hxbsg.cn/82004.Doc
m.wfj.hxbsg.cn/51333.Doc
m.wfj.hxbsg.cn/31913.Doc
m.wfj.hxbsg.cn/84064.Doc
m.wfj.hxbsg.cn/02486.Doc
m.wfj.hxbsg.cn/28644.Doc
m.wfj.hxbsg.cn/59997.Doc
m.wfj.hxbsg.cn/99315.Doc
m.wfj.hxbsg.cn/55559.Doc
m.wfj.hxbsg.cn/60860.Doc
m.wfj.hxbsg.cn/22022.Doc
m.wfj.hxbsg.cn/99991.Doc
m.wfj.hxbsg.cn/40602.Doc
m.wfj.hxbsg.cn/08022.Doc
m.wfj.hxbsg.cn/59559.Doc
m.wfj.hxbsg.cn/20646.Doc
m.wfj.hxbsg.cn/48266.Doc
m.wfj.hxbsg.cn/37959.Doc
m.wfj.hxbsg.cn/59777.Doc
m.wfj.hxbsg.cn/11737.Doc
m.wfj.hxbsg.cn/64482.Doc
m.wfh.hxbsg.cn/04826.Doc
m.wfh.hxbsg.cn/62800.Doc
m.wfh.hxbsg.cn/24888.Doc
m.wfh.hxbsg.cn/82648.Doc
m.wfh.hxbsg.cn/66408.Doc
m.wfh.hxbsg.cn/62022.Doc
m.wfh.hxbsg.cn/40806.Doc
m.wfh.hxbsg.cn/59999.Doc
m.wfh.hxbsg.cn/64022.Doc
m.wfh.hxbsg.cn/04282.Doc
m.wfh.hxbsg.cn/59137.Doc
m.wfh.hxbsg.cn/20244.Doc
m.wfh.hxbsg.cn/02020.Doc
m.wfh.hxbsg.cn/04824.Doc
m.wfh.hxbsg.cn/08626.Doc
m.wfh.hxbsg.cn/62464.Doc
m.wfh.hxbsg.cn/17159.Doc
m.wfh.hxbsg.cn/46682.Doc
m.wfh.hxbsg.cn/11177.Doc
m.wfh.hxbsg.cn/88060.Doc
m.wfg.hxbsg.cn/11139.Doc
m.wfg.hxbsg.cn/28006.Doc
m.wfg.hxbsg.cn/53357.Doc
m.wfg.hxbsg.cn/53755.Doc
m.wfg.hxbsg.cn/39919.Doc
m.wfg.hxbsg.cn/86444.Doc
m.wfg.hxbsg.cn/80666.Doc
m.wfg.hxbsg.cn/06068.Doc
m.wfg.hxbsg.cn/26806.Doc
m.wfg.hxbsg.cn/00226.Doc
m.wfg.hxbsg.cn/55133.Doc
m.wfg.hxbsg.cn/68222.Doc
m.wfg.hxbsg.cn/35517.Doc
m.wfg.hxbsg.cn/26244.Doc
m.wfg.hxbsg.cn/39351.Doc
m.wfg.hxbsg.cn/40040.Doc
m.wfg.hxbsg.cn/37531.Doc
m.wfg.hxbsg.cn/44046.Doc
m.wfg.hxbsg.cn/66402.Doc
m.wfg.hxbsg.cn/68842.Doc
m.wff.hxbsg.cn/48260.Doc
m.wff.hxbsg.cn/44486.Doc
m.wff.hxbsg.cn/39111.Doc
m.wff.hxbsg.cn/17975.Doc
m.wff.hxbsg.cn/55537.Doc
m.wff.hxbsg.cn/93395.Doc
m.wff.hxbsg.cn/68642.Doc
m.wff.hxbsg.cn/82666.Doc
m.wff.hxbsg.cn/40404.Doc
m.wff.hxbsg.cn/66626.Doc
m.wff.hxbsg.cn/00666.Doc
m.wff.hxbsg.cn/99177.Doc
m.wff.hxbsg.cn/66604.Doc
m.wff.hxbsg.cn/20022.Doc
m.wff.hxbsg.cn/62646.Doc
m.wff.hxbsg.cn/95173.Doc
m.wff.hxbsg.cn/68426.Doc
m.wff.hxbsg.cn/40260.Doc
m.wff.hxbsg.cn/24486.Doc
m.wff.hxbsg.cn/66408.Doc
m.wfd.hxbsg.cn/53139.Doc
m.wfd.hxbsg.cn/51193.Doc
m.wfd.hxbsg.cn/04406.Doc
m.wfd.hxbsg.cn/11937.Doc
m.wfd.hxbsg.cn/62006.Doc
m.wfd.hxbsg.cn/99551.Doc
m.wfd.hxbsg.cn/66866.Doc
m.wfd.hxbsg.cn/11171.Doc
m.wfd.hxbsg.cn/84228.Doc
m.wfd.hxbsg.cn/66840.Doc
m.wfd.hxbsg.cn/77955.Doc
m.wfd.hxbsg.cn/66604.Doc
m.wfd.hxbsg.cn/11937.Doc
m.wfd.hxbsg.cn/44840.Doc
m.wfd.hxbsg.cn/31979.Doc
m.wfd.hxbsg.cn/00682.Doc
m.wfd.hxbsg.cn/04486.Doc
m.wfd.hxbsg.cn/51119.Doc
m.wfd.hxbsg.cn/53119.Doc
m.wfd.hxbsg.cn/73135.Doc
m.wfs.hxbsg.cn/51711.Doc
m.wfs.hxbsg.cn/64288.Doc
m.wfs.hxbsg.cn/66806.Doc
m.wfs.hxbsg.cn/64660.Doc
m.wfs.hxbsg.cn/82620.Doc
m.wfs.hxbsg.cn/75353.Doc
m.wfs.hxbsg.cn/35179.Doc
m.wfs.hxbsg.cn/51551.Doc
m.wfs.hxbsg.cn/42444.Doc
m.wfs.hxbsg.cn/44408.Doc
m.wfs.hxbsg.cn/15951.Doc
m.wfs.hxbsg.cn/86440.Doc
m.wfs.hxbsg.cn/59195.Doc
m.wfs.hxbsg.cn/57571.Doc
m.wfs.hxbsg.cn/55133.Doc
m.wfs.hxbsg.cn/82664.Doc
m.wfs.hxbsg.cn/31933.Doc
m.wfs.hxbsg.cn/20040.Doc
m.wfs.hxbsg.cn/60662.Doc
m.wfs.hxbsg.cn/44064.Doc
m.wfa.hxbsg.cn/60220.Doc
m.wfa.hxbsg.cn/08420.Doc
m.wfa.hxbsg.cn/75557.Doc
m.wfa.hxbsg.cn/64062.Doc
m.wfa.hxbsg.cn/77919.Doc
m.wfa.hxbsg.cn/19155.Doc
m.wfa.hxbsg.cn/64060.Doc
m.wfa.hxbsg.cn/37395.Doc
m.wfa.hxbsg.cn/06604.Doc
m.wfa.hxbsg.cn/39759.Doc
m.wfa.hxbsg.cn/22826.Doc
m.wfa.hxbsg.cn/28442.Doc
m.wfa.hxbsg.cn/33119.Doc
m.wfa.hxbsg.cn/46024.Doc
m.wfa.hxbsg.cn/84862.Doc
m.wfa.hxbsg.cn/15955.Doc
m.wfa.hxbsg.cn/88884.Doc
m.wfa.hxbsg.cn/82244.Doc
m.wfa.hxbsg.cn/57795.Doc
m.wfa.hxbsg.cn/28486.Doc
m.wfp.hxbsg.cn/95153.Doc
m.wfp.hxbsg.cn/20648.Doc
m.wfp.hxbsg.cn/46882.Doc
m.wfp.hxbsg.cn/11991.Doc
m.wfp.hxbsg.cn/88040.Doc
m.wfp.hxbsg.cn/71711.Doc
m.wfp.hxbsg.cn/80400.Doc
m.wfp.hxbsg.cn/20028.Doc
m.wfp.hxbsg.cn/86488.Doc
m.wfp.hxbsg.cn/75511.Doc
m.wfp.hxbsg.cn/26662.Doc
m.wfp.hxbsg.cn/28040.Doc
m.wfp.hxbsg.cn/26862.Doc
m.wfp.hxbsg.cn/53355.Doc
m.wfp.hxbsg.cn/53373.Doc
m.wfp.hxbsg.cn/06884.Doc
m.wfp.hxbsg.cn/48660.Doc
m.wfp.hxbsg.cn/68446.Doc
m.wfp.hxbsg.cn/39979.Doc
m.wfp.hxbsg.cn/19797.Doc
m.wfo.hxbsg.cn/22060.Doc
m.wfo.hxbsg.cn/06086.Doc
m.wfo.hxbsg.cn/24844.Doc
m.wfo.hxbsg.cn/26084.Doc
m.wfo.hxbsg.cn/97193.Doc
m.wfo.hxbsg.cn/33399.Doc
m.wfo.hxbsg.cn/57797.Doc
m.wfo.hxbsg.cn/00600.Doc
m.wfo.hxbsg.cn/17919.Doc
m.wfo.hxbsg.cn/53313.Doc
m.wfo.hxbsg.cn/71577.Doc
m.wfo.hxbsg.cn/99193.Doc
m.wfo.hxbsg.cn/13715.Doc
m.wfo.hxbsg.cn/71331.Doc
m.wfo.hxbsg.cn/24646.Doc
m.wfo.hxbsg.cn/77953.Doc
m.wfo.hxbsg.cn/37171.Doc
m.wfo.hxbsg.cn/66646.Doc
m.wfo.hxbsg.cn/31355.Doc
m.wfo.hxbsg.cn/84264.Doc
m.wfi.hxbsg.cn/64486.Doc
m.wfi.hxbsg.cn/13195.Doc
m.wfi.hxbsg.cn/60406.Doc
m.wfi.hxbsg.cn/39513.Doc
m.wfi.hxbsg.cn/17333.Doc
m.wfi.hxbsg.cn/26686.Doc
m.wfi.hxbsg.cn/39513.Doc
m.wfi.hxbsg.cn/84842.Doc
m.wfi.hxbsg.cn/86086.Doc
m.wfi.hxbsg.cn/42040.Doc
m.wfi.hxbsg.cn/66606.Doc
m.wfi.hxbsg.cn/95313.Doc
m.wfi.hxbsg.cn/53397.Doc
m.wfi.hxbsg.cn/97555.Doc
m.wfi.hxbsg.cn/51737.Doc
m.wfi.hxbsg.cn/02444.Doc
m.wfi.hxbsg.cn/71519.Doc
m.wfi.hxbsg.cn/22606.Doc
m.wfi.hxbsg.cn/33999.Doc
m.wfi.hxbsg.cn/28820.Doc
m.wfu.hxbsg.cn/22286.Doc
m.wfu.hxbsg.cn/53975.Doc
m.wfu.hxbsg.cn/06024.Doc
m.wfu.hxbsg.cn/39555.Doc
m.wfu.hxbsg.cn/77993.Doc
m.wfu.hxbsg.cn/57333.Doc
m.wfu.hxbsg.cn/62440.Doc
m.wfu.hxbsg.cn/46062.Doc
m.wfu.hxbsg.cn/26006.Doc
m.wfu.hxbsg.cn/55131.Doc
m.wfu.hxbsg.cn/37797.Doc
m.wfu.hxbsg.cn/95139.Doc
m.wfu.hxbsg.cn/51353.Doc
m.wfu.hxbsg.cn/84682.Doc
m.wfu.hxbsg.cn/77979.Doc
m.wfu.hxbsg.cn/64262.Doc
m.wfu.hxbsg.cn/84028.Doc
m.wfu.hxbsg.cn/44226.Doc
m.wfu.hxbsg.cn/33359.Doc
m.wfu.hxbsg.cn/28664.Doc
m.wfy.hxbsg.cn/11793.Doc
m.wfy.hxbsg.cn/86468.Doc
m.wfy.hxbsg.cn/77197.Doc
m.wfy.hxbsg.cn/97539.Doc
m.wfy.hxbsg.cn/22202.Doc
m.wfy.hxbsg.cn/28644.Doc
m.wfy.hxbsg.cn/00486.Doc
m.wfy.hxbsg.cn/97333.Doc
m.wfy.hxbsg.cn/31375.Doc
m.wfy.hxbsg.cn/26020.Doc
m.wfy.hxbsg.cn/11735.Doc
m.wfy.hxbsg.cn/82802.Doc
m.wfy.hxbsg.cn/73559.Doc
m.wfy.hxbsg.cn/59733.Doc
m.wfy.hxbsg.cn/82826.Doc
m.wfy.hxbsg.cn/15595.Doc
m.wfy.hxbsg.cn/42204.Doc
m.wfy.hxbsg.cn/82222.Doc
m.wfy.hxbsg.cn/37573.Doc
m.wfy.hxbsg.cn/75139.Doc
m.wft.hxbsg.cn/88064.Doc
m.wft.hxbsg.cn/73953.Doc
m.wft.hxbsg.cn/39511.Doc
m.wft.hxbsg.cn/11195.Doc
m.wft.hxbsg.cn/11919.Doc
m.wft.hxbsg.cn/08846.Doc
m.wft.hxbsg.cn/68000.Doc
m.wft.hxbsg.cn/93595.Doc
m.wft.hxbsg.cn/55937.Doc
m.wft.hxbsg.cn/71133.Doc
m.wft.hxbsg.cn/26606.Doc
m.wft.hxbsg.cn/13599.Doc
m.wft.hxbsg.cn/24860.Doc
m.wft.hxbsg.cn/48200.Doc
m.wft.hxbsg.cn/91735.Doc
m.wft.hxbsg.cn/53539.Doc
m.wft.hxbsg.cn/86462.Doc
m.wft.hxbsg.cn/77119.Doc
m.wft.hxbsg.cn/68682.Doc
m.wft.hxbsg.cn/40224.Doc
m.wfr.hxbsg.cn/79991.Doc
m.wfr.hxbsg.cn/80480.Doc
m.wfr.hxbsg.cn/84200.Doc
m.wfr.hxbsg.cn/00662.Doc
m.wfr.hxbsg.cn/59777.Doc
m.wfr.hxbsg.cn/00604.Doc
m.wfr.hxbsg.cn/35793.Doc
m.wfr.hxbsg.cn/26248.Doc
m.wfr.hxbsg.cn/80044.Doc
m.wfr.hxbsg.cn/26062.Doc
m.wfr.hxbsg.cn/64824.Doc
m.wfr.hxbsg.cn/93753.Doc
m.wfr.hxbsg.cn/42822.Doc
m.wfr.hxbsg.cn/99517.Doc
m.wfr.hxbsg.cn/48002.Doc
m.wfr.hxbsg.cn/08086.Doc
m.wfr.hxbsg.cn/71595.Doc
m.wfr.hxbsg.cn/08046.Doc
m.wfr.hxbsg.cn/37733.Doc
m.wfr.hxbsg.cn/46420.Doc
m.wfe.hxbsg.cn/26400.Doc
m.wfe.hxbsg.cn/35533.Doc
m.wfe.hxbsg.cn/11719.Doc
m.wfe.hxbsg.cn/1.Doc
m.wfe.hxbsg.cn/62084.Doc
