八衣督窒辣


  
CountDownLatch 它是一种协调线程执行顺序的同步工具，适用于“主线程等待多个子任务完成后继续”的场景；它通过递减计数器等待，不能重置，需要正确初始化，以确保每次调用完成 countDown() 并处理加班和中断。

CountDownLatch 它是什么，什么时候应该用它？
CountDownLatch 不是用来替代的 synchronized 或 lock 是的，它是用来做的「协调线程执行顺序」在其他线程完成一组操作之前，让一个或多个线程等待。典型的场景是「主线程等所有子任务完成后再继续运行」，例如，在启动服务时预热多个模块，批量请求并发后统一汇总结果。
它内部有一个计数器（count），每次调用 countDown() 就减 1；调用 await() 在计数器归零或中断之前，线程会被阻塞。注意：计数器只能递减，不能重置，也不能加回——这个和 CyclicBarrier 有本质区别。

正确的初始化和调用 await() / countDown()
初始化时传入 count 必须等于你想要完成的“事件数”。如果设置为 但只调用了 4 次 countDown()，那么所有在 await() 等待的线程将永远堵塞(除非超时或中断)。


await() 应放置在需要等待的位置，如主线程；在启动子线程之前不要调用它

countDown() 确保**每个逻辑完成点都被调用一次**，常见的错误是在异常分支中泄漏，导致计数器卡住
可以同时调用多个线程 countDown()，线程安全；但是 await() 可多次调用(无副作用)，但一般只在等待方调用一次

CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i  {
try {
// 模拟任务
Thread.sleep(1000);
} catch (InterruptedException e) {
Thread.currentThread().interrupt();
return;
} finally {
latch.countDown(); // 即使出现异常，也要保证执行
}
}).start();
}
latch.await(); // 这里阻塞了主线程，直到三次 countDown 完成
System.out.println("All tasks done.");


不要忽视超时和中断处理
await() 有两个重载：无人参版本将继续等待，有人参版本支持加班。生产环境几乎必须使用加班版本，否则单个子线程卡住将导致整个过程无法恢复。
；
			
		
另外，await(long timeout, TimeUnit unit) 超时或中断时返回 false，此时，您需要判断是否继续跟进逻辑，或抛出异常，记录报警。

超时时间不应设置得太短，而应考虑最慢任务的合理耗时 20%~50%
调用 await() 如果线程中断，它将被抛出 InterruptedException 并清除中断状态，记住在 catch 块中恢复中断标志：Thread.currentThread().interrupt()

不要在 await() 之后，直接假设所有子任务都必须成功——这只意味着计数器为零，不检查结果是否正常

和 CyclicBarrier、Semaphore 的关键区别
这三个经常被初学者混淆：


CountDownLatch 是「一次性门闩」：计数归零后，所有等待线程释放，然后调整 await() 立即返回，countDown() 也无效了

CyclicBarrier 是「可重复使用的栅栏」，适用于多轮合作(如分阶段计算)，支持到达时触发 Runnable 回调

Semaphore 控制的是「并发数量」，不是执行顺序，它的 acquire()/release() 更像是资源许可证的借还

假如你的需求是「等 N 事情完成后再往下走」，就选 CountDownLatch；如果是要「N 线程相互等待，共同开始下一轮」，那得换 CyclicBarrier。
容易被忽视的一点:CountDownLatch 结构参数为 int 类型，最大值是 Integer.MAX_VALUE，但是，如果实际使用超过几千，我们应该怀疑设计是否合理-可能更适合使用 CompletableFuture.allOf() 或线程池的 invokeAll()。	 

绦抑兑诩祭撬耘久阎祭埠磐诜思檬

fxf.irvnp.cn/860464.Doc
fxf.irvnp.cn/668422.Doc
fxf.irvnp.cn/351973.Doc
fxf.irvnp.cn/128630.Doc
fxf.irvnp.cn/882426.Doc
fxf.irvnp.cn/262822.Doc
fxf.irvnp.cn/812966.Doc
fxf.irvnp.cn/060862.Doc
fxf.irvnp.cn/468466.Doc
fxf.irvnp.cn/597915.Doc
fxd.irvnp.cn/420062.Doc
fxd.irvnp.cn/486888.Doc
fxd.irvnp.cn/480248.Doc
fxd.irvnp.cn/066648.Doc
fxd.irvnp.cn/808242.Doc
fxd.irvnp.cn/820824.Doc
fxd.irvnp.cn/664860.Doc
fxd.irvnp.cn/464686.Doc
fxd.irvnp.cn/584412.Doc
fxd.irvnp.cn/610251.Doc
fxs.irvnp.cn/004848.Doc
fxs.irvnp.cn/664620.Doc
fxs.irvnp.cn/700996.Doc
fxs.irvnp.cn/190244.Doc
fxs.irvnp.cn/206626.Doc
fxs.irvnp.cn/064228.Doc
fxs.irvnp.cn/208442.Doc
fxs.irvnp.cn/484484.Doc
fxs.irvnp.cn/466886.Doc
fxs.irvnp.cn/402286.Doc
fxa.irvnp.cn/623421.Doc
fxa.irvnp.cn/020460.Doc
fxa.irvnp.cn/622404.Doc
fxa.irvnp.cn/319046.Doc
fxa.irvnp.cn/644468.Doc
fxa.irvnp.cn/177843.Doc
fxa.irvnp.cn/224066.Doc
fxa.irvnp.cn/206064.Doc
fxa.irvnp.cn/448884.Doc
fxa.irvnp.cn/530987.Doc
fxp.irvnp.cn/086396.Doc
fxp.irvnp.cn/486262.Doc
fxp.irvnp.cn/799565.Doc
fxp.irvnp.cn/202466.Doc
fxp.irvnp.cn/174757.Doc
fxp.irvnp.cn/066828.Doc
fxp.irvnp.cn/664440.Doc
fxp.irvnp.cn/046888.Doc
fxp.irvnp.cn/866646.Doc
fxp.irvnp.cn/208684.Doc
fxo.irvnp.cn/880446.Doc
fxo.irvnp.cn/662822.Doc
fxo.irvnp.cn/046462.Doc
fxo.irvnp.cn/868004.Doc
fxo.irvnp.cn/662200.Doc
fxo.irvnp.cn/288664.Doc
fxo.irvnp.cn/222660.Doc
fxo.irvnp.cn/464468.Doc
fxo.irvnp.cn/597337.Doc
fxo.irvnp.cn/480066.Doc
fxi.irvnp.cn/622406.Doc
fxi.irvnp.cn/888480.Doc
fxi.irvnp.cn/404828.Doc
fxi.irvnp.cn/402442.Doc
fxi.irvnp.cn/797511.Doc
fxi.irvnp.cn/420606.Doc
fxi.irvnp.cn/767044.Doc
fxi.irvnp.cn/288864.Doc
fxi.irvnp.cn/406400.Doc
fxi.irvnp.cn/206880.Doc
fxu.irvnp.cn/268042.Doc
fxu.irvnp.cn/460000.Doc
fxu.irvnp.cn/731806.Doc
fxu.irvnp.cn/824464.Doc
fxu.irvnp.cn/245206.Doc
fxu.irvnp.cn/222628.Doc
fxu.irvnp.cn/044024.Doc
fxu.irvnp.cn/002082.Doc
fxu.irvnp.cn/873415.Doc
fxu.irvnp.cn/026440.Doc
fxy.irvnp.cn/332242.Doc
fxy.irvnp.cn/248826.Doc
fxy.irvnp.cn/064080.Doc
fxy.irvnp.cn/355731.Doc
fxy.irvnp.cn/224220.Doc
fxy.irvnp.cn/064246.Doc
fxy.irvnp.cn/066606.Doc
fxy.irvnp.cn/826684.Doc
fxy.irvnp.cn/827085.Doc
fxy.irvnp.cn/860048.Doc
fxt.irvnp.cn/428668.Doc
fxt.irvnp.cn/662880.Doc
fxt.irvnp.cn/888282.Doc
fxt.irvnp.cn/462602.Doc
fxt.irvnp.cn/624424.Doc
fxt.irvnp.cn/448820.Doc
fxt.irvnp.cn/000242.Doc
fxt.irvnp.cn/624426.Doc
fxt.irvnp.cn/426240.Doc
fxt.irvnp.cn/905791.Doc
fxr.irvnp.cn/066886.Doc
fxr.irvnp.cn/600660.Doc
fxr.irvnp.cn/680284.Doc
fxr.irvnp.cn/064422.Doc
fxr.irvnp.cn/044288.Doc
fxr.irvnp.cn/070404.Doc
fxr.irvnp.cn/846800.Doc
fxr.irvnp.cn/880608.Doc
fxr.irvnp.cn/284266.Doc
fxr.irvnp.cn/262242.Doc
fxe.irvnp.cn/680006.Doc
fxe.irvnp.cn/446664.Doc
fxe.irvnp.cn/248660.Doc
fxe.irvnp.cn/044240.Doc
fxe.irvnp.cn/682242.Doc
fxe.irvnp.cn/408468.Doc
fxe.irvnp.cn/060068.Doc
fxe.irvnp.cn/402340.Doc
fxe.irvnp.cn/048264.Doc
fxe.irvnp.cn/622866.Doc
fxw.irvnp.cn/800020.Doc
fxw.irvnp.cn/824220.Doc
fxw.irvnp.cn/086606.Doc
fxw.irvnp.cn/826404.Doc
fxw.irvnp.cn/862264.Doc
fxw.irvnp.cn/628600.Doc
fxw.irvnp.cn/886088.Doc
fxw.irvnp.cn/808608.Doc
fxw.irvnp.cn/600080.Doc
fxw.irvnp.cn/528725.Doc
fxq.irvnp.cn/446680.Doc
fxq.irvnp.cn/486406.Doc
fxq.irvnp.cn/466460.Doc
fxq.irvnp.cn/868606.Doc
fxq.irvnp.cn/462408.Doc
fxq.irvnp.cn/484606.Doc
fxq.irvnp.cn/088682.Doc
fxq.irvnp.cn/428620.Doc
fxq.irvnp.cn/040884.Doc
fxq.irvnp.cn/860626.Doc
fzm.irvnp.cn/682026.Doc
fzm.irvnp.cn/466244.Doc
fzm.irvnp.cn/446286.Doc
fzm.irvnp.cn/308138.Doc
fzm.irvnp.cn/246264.Doc
fzm.irvnp.cn/288061.Doc
fzm.irvnp.cn/793311.Doc
fzm.irvnp.cn/840068.Doc
fzm.irvnp.cn/806688.Doc
fzn.irvnp.cn/642462.Doc
fzn.irvnp.cn/488662.Doc
fzn.irvnp.cn/888880.Doc
fzn.irvnp.cn/740220.Doc
fzn.irvnp.cn/802200.Doc
fzn.irvnp.cn/428606.Doc
fzn.irvnp.cn/136766.Doc
fzn.irvnp.cn/537186.Doc
fzn.irvnp.cn/660464.Doc
fzn.irvnp.cn/448806.Doc
fzb.irvnp.cn/341890.Doc
fzb.irvnp.cn/840664.Doc
fzb.irvnp.cn/064820.Doc
fzb.irvnp.cn/648626.Doc
fzb.irvnp.cn/399137.Doc
fzb.irvnp.cn/426084.Doc
fzb.irvnp.cn/222428.Doc
fzb.irvnp.cn/464820.Doc
fzb.irvnp.cn/006840.Doc
fzb.irvnp.cn/857601.Doc
fzv.irvnp.cn/488204.Doc
fzv.irvnp.cn/402260.Doc
fzv.irvnp.cn/797980.Doc
fzv.irvnp.cn/235454.Doc
fzv.irvnp.cn/648226.Doc
fzv.irvnp.cn/828240.Doc
fzv.irvnp.cn/446222.Doc
fzv.irvnp.cn/444666.Doc
fzv.irvnp.cn/022068.Doc
fzv.irvnp.cn/846682.Doc
fzc.irvnp.cn/824488.Doc
fzc.irvnp.cn/848006.Doc
fzc.irvnp.cn/860848.Doc
fzc.irvnp.cn/684062.Doc
fzc.irvnp.cn/880442.Doc
fzc.irvnp.cn/822206.Doc
fzc.irvnp.cn/248060.Doc
fzc.irvnp.cn/844044.Doc
fzc.irvnp.cn/466224.Doc
fzc.irvnp.cn/437548.Doc
fzx.irvnp.cn/606264.Doc
fzx.irvnp.cn/440066.Doc
fzx.irvnp.cn/404022.Doc
fzx.irvnp.cn/620604.Doc
fzx.irvnp.cn/779339.Doc
fzx.irvnp.cn/171773.Doc
fzx.irvnp.cn/602402.Doc
fzx.irvnp.cn/806020.Doc
fzx.irvnp.cn/486822.Doc
fzx.irvnp.cn/337971.Doc
fzz.irvnp.cn/824000.Doc
fzz.irvnp.cn/488626.Doc
fzz.irvnp.cn/517461.Doc
fzz.irvnp.cn/480640.Doc
fzz.irvnp.cn/444808.Doc
fzz.irvnp.cn/282824.Doc
fzz.irvnp.cn/159151.Doc
fzz.irvnp.cn/448606.Doc
fzz.irvnp.cn/280208.Doc
fzz.irvnp.cn/808620.Doc
fzl.irvnp.cn/600644.Doc
fzl.irvnp.cn/468422.Doc
fzl.irvnp.cn/862464.Doc
fzl.irvnp.cn/062084.Doc
fzl.irvnp.cn/070884.Doc
fzl.irvnp.cn/426062.Doc
fzl.irvnp.cn/244826.Doc
fzl.irvnp.cn/448624.Doc
fzl.irvnp.cn/851069.Doc
fzl.irvnp.cn/663239.Doc
fzk.irvnp.cn/860866.Doc
fzk.irvnp.cn/151399.Doc
fzk.irvnp.cn/284844.Doc
fzk.irvnp.cn/066486.Doc
fzk.irvnp.cn/688864.Doc
fzk.irvnp.cn/266888.Doc
fzk.irvnp.cn/262222.Doc
fzk.irvnp.cn/383829.Doc
fzk.irvnp.cn/446080.Doc
fzk.irvnp.cn/266084.Doc
fzj.irvnp.cn/824646.Doc
fzj.irvnp.cn/404268.Doc
fzj.irvnp.cn/204806.Doc
fzj.irvnp.cn/372744.Doc
fzj.irvnp.cn/444840.Doc
fzj.irvnp.cn/482260.Doc
fzj.irvnp.cn/840822.Doc
fzj.irvnp.cn/484202.Doc
fzj.irvnp.cn/888482.Doc
fzj.irvnp.cn/806884.Doc
fzh.irvnp.cn/882206.Doc
fzh.irvnp.cn/806608.Doc
fzh.irvnp.cn/602848.Doc
fzh.irvnp.cn/908610.Doc
fzh.irvnp.cn/872904.Doc
fzh.irvnp.cn/258324.Doc
fzh.irvnp.cn/222248.Doc
fzh.irvnp.cn/202028.Doc
fzh.irvnp.cn/646440.Doc
fzh.irvnp.cn/642484.Doc
fzg.irvnp.cn/888608.Doc
fzg.irvnp.cn/684604.Doc
fzg.irvnp.cn/420222.Doc
fzg.irvnp.cn/240264.Doc
fzg.irvnp.cn/064028.Doc
fzg.irvnp.cn/522032.Doc
fzg.irvnp.cn/666682.Doc
fzg.irvnp.cn/460264.Doc
fzg.irvnp.cn/628402.Doc
fzg.irvnp.cn/802642.Doc
fzf.irvnp.cn/228204.Doc
fzf.irvnp.cn/666080.Doc
fzf.irvnp.cn/620202.Doc
fzf.irvnp.cn/682426.Doc
fzf.irvnp.cn/266428.Doc
fzf.irvnp.cn/606862.Doc
fzf.irvnp.cn/608200.Doc
fzf.irvnp.cn/868006.Doc
fzf.irvnp.cn/248220.Doc
fzf.irvnp.cn/228825.Doc
fzd.irvnp.cn/666660.Doc
fzd.irvnp.cn/940072.Doc
fzd.irvnp.cn/688640.Doc
fzd.irvnp.cn/420826.Doc
fzd.irvnp.cn/777115.Doc
fzd.irvnp.cn/284282.Doc
fzd.irvnp.cn/888624.Doc
fzd.irvnp.cn/777733.Doc
fzd.irvnp.cn/820066.Doc
fzd.irvnp.cn/224882.Doc
fzs.irvnp.cn/227312.Doc
fzs.irvnp.cn/284644.Doc
fzs.irvnp.cn/893544.Doc
fzs.irvnp.cn/264408.Doc
fzs.irvnp.cn/840220.Doc
fzs.irvnp.cn/216234.Doc
fzs.irvnp.cn/004228.Doc
fzs.irvnp.cn/902824.Doc
fzs.irvnp.cn/879937.Doc
fzs.irvnp.cn/343487.Doc
fza.irvnp.cn/640268.Doc
fza.irvnp.cn/048040.Doc
fza.irvnp.cn/699921.Doc
fza.irvnp.cn/266064.Doc
fza.irvnp.cn/240800.Doc
fza.irvnp.cn/406420.Doc
fza.irvnp.cn/949873.Doc
fza.irvnp.cn/057828.Doc
fza.irvnp.cn/222064.Doc
fza.irvnp.cn/422464.Doc
fzp.irvnp.cn/959199.Doc
fzp.irvnp.cn/000644.Doc
fzp.irvnp.cn/228718.Doc
fzp.irvnp.cn/824088.Doc
fzp.irvnp.cn/795195.Doc
fzp.irvnp.cn/808844.Doc
fzp.irvnp.cn/224409.Doc
fzp.irvnp.cn/460488.Doc
fzp.irvnp.cn/242444.Doc
fzp.irvnp.cn/086060.Doc
fzo.irvnp.cn/281276.Doc
fzo.irvnp.cn/769645.Doc
fzo.irvnp.cn/537339.Doc
fzo.irvnp.cn/846600.Doc
fzo.irvnp.cn/482202.Doc
fzo.irvnp.cn/235671.Doc
fzo.irvnp.cn/666808.Doc
fzo.irvnp.cn/084226.Doc
fzo.irvnp.cn/484000.Doc
fzo.irvnp.cn/773195.Doc
fzi.irvnp.cn/199319.Doc
fzi.irvnp.cn/486824.Doc
fzi.irvnp.cn/066404.Doc
fzi.irvnp.cn/462022.Doc
fzi.irvnp.cn/408442.Doc
fzi.irvnp.cn/288866.Doc
fzi.irvnp.cn/220248.Doc
fzi.irvnp.cn/646060.Doc
fzi.irvnp.cn/804044.Doc
fzi.irvnp.cn/082660.Doc
fzu.irvnp.cn/062626.Doc
fzu.irvnp.cn/460422.Doc
fzu.irvnp.cn/608664.Doc
fzu.irvnp.cn/260666.Doc
fzu.irvnp.cn/802486.Doc
fzu.irvnp.cn/286884.Doc
fzu.irvnp.cn/802200.Doc
fzu.irvnp.cn/124046.Doc
fzu.irvnp.cn/844064.Doc
fzu.irvnp.cn/028068.Doc
fzy.irvnp.cn/064224.Doc
fzy.irvnp.cn/246806.Doc
fzy.irvnp.cn/428044.Doc
fzy.irvnp.cn/444628.Doc
fzy.irvnp.cn/229695.Doc
fzy.irvnp.cn/008842.Doc
fzy.irvnp.cn/996262.Doc
fzy.irvnp.cn/846022.Doc
fzy.irvnp.cn/088028.Doc
fzy.irvnp.cn/488802.Doc
fzt.irvnp.cn/026226.Doc
fzt.irvnp.cn/806428.Doc
fzt.irvnp.cn/442088.Doc
fzt.irvnp.cn/206440.Doc
fzt.irvnp.cn/624006.Doc
fzt.irvnp.cn/402248.Doc
fzt.irvnp.cn/202880.Doc
fzt.irvnp.cn/786588.Doc
fzt.irvnp.cn/402659.Doc
fzt.irvnp.cn/868664.Doc
fzr.irvnp.cn/935535.Doc
fzr.irvnp.cn/460628.Doc
fzr.irvnp.cn/155913.Doc
fzr.irvnp.cn/284028.Doc
fzr.irvnp.cn/458504.Doc
fzr.irvnp.cn/224624.Doc
fzr.irvnp.cn/108546.Doc
fzr.irvnp.cn/046626.Doc
fzr.irvnp.cn/080622.Doc
fzr.irvnp.cn/208860.Doc
fze.irvnp.cn/068602.Doc
fze.irvnp.cn/642220.Doc
fze.irvnp.cn/042486.Doc
fze.irvnp.cn/339913.Doc
fze.irvnp.cn/642288.Doc
fze.irvnp.cn/182626.Doc
fze.irvnp.cn/335391.Doc
fze.irvnp.cn/028688.Doc
fze.irvnp.cn/840204.Doc
fze.irvnp.cn/400020.Doc
fzw.irvnp.cn/264008.Doc
fzw.irvnp.cn/628840.Doc
fzw.irvnp.cn/813768.Doc
fzw.irvnp.cn/604480.Doc
fzw.irvnp.cn/445856.Doc
fzw.irvnp.cn/240864.Doc
fzw.irvnp.cn/644244.Doc
fzw.irvnp.cn/936509.Doc
fzw.irvnp.cn/464882.Doc
fzw.irvnp.cn/599571.Doc
fzq.irvnp.cn/602448.Doc
fzq.irvnp.cn/286048.Doc
fzq.irvnp.cn/593599.Doc
fzq.irvnp.cn/266833.Doc
fzq.irvnp.cn/766249.Doc
fzq.irvnp.cn/468286.Doc
