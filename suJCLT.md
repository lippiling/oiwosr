攘野涌疟春


【Python 异步编程常见错误】
============================

异步编程有若干容易踩的坑：忘记 await、阻塞事件循环、
未处理异常、任务泄漏等。了解这些陷阱至关重要。

──────────────────────────────────────────────
错误 1：忘记 await
──────────────────────────────────────────────

import asyncio

async def 返回数据():
    """返回字符串的协程"""
    await asyncio.sleep(0.1)
    return "重要数据"

async def 忘记await示例():
    """演示各种忘记 await 的情况"""

    # 错误：没有 await，得到的是协程对象而非结果
    协程对象 = 返回数据()  # 创建协程但不执行
    print(f"类型: {type(协程对象)}")  # <class 'coroutine'>
    # 实际上 协程对象 未被 await，会出现 RuntimeWarning

    # 正确做法
    结果 = await 返回数据()
    print(f"正确获取: {结果}")

    # 错误：列表推导中忘记 await
    协程列表 = [返回数据() for _ in range(3)]
    # 下面的写法是错误的：
    # 结果列表 = [r for r in 协程列表]  # 得到的是协程对象列表

    # 正确做法
    结果列表 = await asyncio.gather(*协程列表)
    print(f"正确并发: {结果列表}")

    # 错误：调用异步函数但不用 await
    # asyncio.create_task(返回数据())  # 正确，create_task 内部调度
    # asyncio.ensure_future(返回数据())  # 正确

asyncio.run(忘记await示例())

──────────────────────────────────────────────
错误 2：阻塞事件循环
──────────────────────────────────────────────

import asyncio
import time

async def 阻塞事件循环示例():
    """演示阻塞事件循环的后果"""

    async def 异步任务(编号):
        print(f"任务{编号} 开始")
        # 错误：使用 time.sleep 而不是 asyncio.sleep
        time.sleep(2)  # 这会阻塞整个事件循环！
        print(f"任务{编号} 结束")

    async def 正确方式(编号):
        print(f"正确任务{编号} 开始")
        await asyncio.sleep(2)  # 正确：让出控制权
        print(f"正确任务{编号} 结束")

    print("=== 错误: time.sleep 阻塞事件循环 ===")
    开始 = time.time()
    await asyncio.gather(
        异步任务(1),
        异步任务(2),  # 串行执行！因为 time.sleep 阻塞了
    )
    print(f"耗时(错误): {time.time() - 开始:.1f}s")

    print("\n=== 正确: asyncio.sleep 不阻塞 ===")
    开始 = time.time()
    await asyncio.gather(
        正确方式(1),
        正确方式(2),  # 并发执行！
    )
    print(f"耗时(正确): {time.time() - 开始:.1f}s")

    # 其他常见的阻塞操作：
    # requests.get() -> 改用 aiohttp
    # time.sleep()   -> 改用 asyncio.sleep
    # 文件读写       -> 改用 aiofiles 或 run_in_executor

asyncio.run(阻塞事件循环示例())

──────────────────────────────────────────────
错误 3：未处理异常
──────────────────────────────────────────────

import asyncio

async def 未处理异常示例():
    """协程中未捕获的异常会静默丢失"""

    async def 可能崩溃(编号):
        """可能抛出异常的协程"""
        if 编号 == 2:
            raise ValueError(f"任务{编号} 出错了!")
        await asyncio.sleep(0.5)
        return f"任务{编号} 完成"

    # 错误：gather 默认不传播异常
    结果 = await asyncio.gather(
        可能崩溃(1),
        可能崩溃(2),  # 异常被 gather 捕获
        可能崩溃(3),
        return_exceptions=False,  # 默认，异常会抛出
    )
    # 上面会直接抛出异常

    # 正确：使用 return_exceptions=True
    结果 = await asyncio.gather(
        可能崩溃(1),
        可能崩溃(2),
        可能崩溃(3),
        return_exceptions=True,  # 异常作为结果返回
    )
    for r in 结果:
        if isinstance(r, Exception):
            print(f"捕获异常: {r}")
        else:
            print(f"成功: {r}")

    # 错误：create_task 后不 await 也不处理异常
    # 任务 = asyncio.create_task(可能崩溃(2))  # 异常静默丢失！
    # 必须 await 或添加回调

    # 正确：为 Task 添加异常处理
    任务 = asyncio.create_task(可能崩溃(2))
    任务.add_done_callback(
        lambda t: print(f"任务异常: {t.exception()}")
        if t.exception() else None
    )

    try:
        await 任务
    except ValueError as e:
        print(f"主流程捕获: {e}")

asyncio.run(未处理异常示例())

──────────────────────────────────────────────
错误 4：任务泄漏与调试
──────────────────────────────────────────────

import asyncio

async def 任务泄漏示例():
    """创建但不管理的任务会导致泄漏"""

    async def 后台任务():
        """长时间运行的后台任务"""
        try:
            await asyncio.sleep(100)  # 长时间运行
        except asyncio.CancelledError:
            print("后台任务被取消")
            raise

    async def 正确创建任务():
        """正确管理任务生命周期"""
        # 保持任务引用
        任务 = asyncio.create_task(后台任务())

        # 在适当时机取消
        await asyncio.sleep(1)
        任务.cancel()
        try:
            await 任务
        except asyncio.CancelledError:
            print("任务已安全取消")

    # 调试：获取所有待处理任务
    所有任务 = asyncio.all_tasks()
    print(f"当前任务数: {len(所有任务)}")
    for t in 所有任务:
        print(f"  任务: {t.get_name()}, "
              f"完成: {t.done()}, "
              f"取消: {t.cancelled()}")

    # 使用 asyncio.Task 的调试模式
    循环 = asyncio.get_running_loop()
    循环.set_debug(True)  # 开启调试模式

    # 调试模式下会警告：
    # - 超过 0.1 秒未 await 的协程
    # - 未处理的异常
    # - 阻塞调用

asyncio.run(任务泄漏示例())

──────────────────────────────────────────────
核心要点
──────────────────────────────────────────────
• 忘记 await - 协程不会执行，得到协程对象
• 阻塞事件循环 - time.sleep、同步 IO 会阻塞所有协程
• 未处理异常 - 用 return_exceptions=True 或回调捕获
• 任务泄漏 - 保持 Task 引用，及时取消
• create_task 不加引用 - 任务可能被 GC
• 异步回调地狱 - 用 async/await 代替回调链
• 调试技巧 - set_debug(True) + asyncio.all_tasks()

沾抗堤录叭姑肥植坦瓜九依救嫉潘

rsb.hjiocz.cn/137153.htm
rsb.hjiocz.cn/886683.htm
rsb.hjiocz.cn/280223.htm
rsb.hjiocz.cn/464203.htm
rsb.hjiocz.cn/282083.htm
rsb.hjiocz.cn/739733.htm
rsb.hjiocz.cn/282843.htm
rsb.hjiocz.cn/402003.htm
rsb.hjiocz.cn/204043.htm
rsb.hjiocz.cn/711993.htm
rsv.hjiocz.cn/404003.htm
rsv.hjiocz.cn/228443.htm
rsv.hjiocz.cn/595173.htm
rsv.hjiocz.cn/268023.htm
rsv.hjiocz.cn/553913.htm
rsv.hjiocz.cn/759313.htm
rsv.hjiocz.cn/373573.htm
rsv.hjiocz.cn/880483.htm
rsv.hjiocz.cn/862263.htm
rsv.hjiocz.cn/151933.htm
rsc.hjiocz.cn/664063.htm
rsc.hjiocz.cn/359733.htm
rsc.hjiocz.cn/088483.htm
rsc.hjiocz.cn/004203.htm
rsc.hjiocz.cn/313993.htm
rsc.hjiocz.cn/084883.htm
rsc.hjiocz.cn/284843.htm
rsc.hjiocz.cn/268463.htm
rsc.hjiocz.cn/222803.htm
rsc.hjiocz.cn/393753.htm
rsx.hjiocz.cn/355193.htm
rsx.hjiocz.cn/395373.htm
rsx.hjiocz.cn/408623.htm
rsx.hjiocz.cn/773913.htm
rsx.hjiocz.cn/006463.htm
rsx.hjiocz.cn/460803.htm
rsx.hjiocz.cn/339153.htm
rsx.hjiocz.cn/262003.htm
rsx.hjiocz.cn/773393.htm
rsx.hjiocz.cn/197773.htm
rsz.hjiocz.cn/442823.htm
rsz.hjiocz.cn/795393.htm
rsz.hjiocz.cn/008683.htm
rsz.hjiocz.cn/379333.htm
rsz.hjiocz.cn/460043.htm
rsz.hjiocz.cn/446643.htm
rsz.hjiocz.cn/519173.htm
rsz.hjiocz.cn/151713.htm
rsz.hjiocz.cn/440043.htm
rsz.hjiocz.cn/171133.htm
rsl.hjiocz.cn/400643.htm
rsl.hjiocz.cn/131173.htm
rsl.hjiocz.cn/606403.htm
rsl.hjiocz.cn/040003.htm
rsl.hjiocz.cn/997773.htm
rsl.hjiocz.cn/731753.htm
rsl.hjiocz.cn/284043.htm
rsl.hjiocz.cn/242603.htm
rsl.hjiocz.cn/957573.htm
rsl.hjiocz.cn/391133.htm
rsk.hjiocz.cn/048483.htm
rsk.hjiocz.cn/335113.htm
rsk.hjiocz.cn/844263.htm
rsk.hjiocz.cn/317753.htm
rsk.hjiocz.cn/006863.htm
rsk.hjiocz.cn/846283.htm
rsk.hjiocz.cn/484043.htm
rsk.hjiocz.cn/533793.htm
rsk.hjiocz.cn/062483.htm
rsk.hjiocz.cn/153793.htm
rsj.hjiocz.cn/355913.htm
rsj.hjiocz.cn/846023.htm
rsj.hjiocz.cn/284603.htm
rsj.hjiocz.cn/408623.htm
rsj.hjiocz.cn/246463.htm
rsj.hjiocz.cn/153393.htm
rsj.hjiocz.cn/604663.htm
rsj.hjiocz.cn/444683.htm
rsj.hjiocz.cn/026843.htm
rsj.hjiocz.cn/084403.htm
rsh.hjiocz.cn/068403.htm
rsh.hjiocz.cn/488403.htm
rsh.hjiocz.cn/026203.htm
rsh.hjiocz.cn/488823.htm
rsh.hjiocz.cn/280243.htm
rsh.hjiocz.cn/284403.htm
rsh.hjiocz.cn/953513.htm
rsh.hjiocz.cn/084263.htm
rsh.hjiocz.cn/460803.htm
rsh.hjiocz.cn/395373.htm
rsg.hjiocz.cn/086063.htm
rsg.hjiocz.cn/759733.htm
rsg.hjiocz.cn/000843.htm
rsg.hjiocz.cn/004803.htm
rsg.hjiocz.cn/337173.htm
rsg.hjiocz.cn/002423.htm
rsg.hjiocz.cn/828623.htm
rsg.hjiocz.cn/824823.htm
rsg.hjiocz.cn/462863.htm
rsg.hjiocz.cn/319913.htm
rsf.hjiocz.cn/080843.htm
rsf.hjiocz.cn/359353.htm
rsf.hjiocz.cn/404083.htm
rsf.hjiocz.cn/864003.htm
rsf.hjiocz.cn/375193.htm
rsf.hjiocz.cn/048443.htm
rsf.hjiocz.cn/577113.htm
rsf.hjiocz.cn/084463.htm
rsf.hjiocz.cn/868623.htm
rsf.hjiocz.cn/440443.htm
rsd.hjiocz.cn/626683.htm
rsd.hjiocz.cn/020483.htm
rsd.hjiocz.cn/648003.htm
rsd.hjiocz.cn/179753.htm
rsd.hjiocz.cn/353713.htm
rsd.hjiocz.cn/482003.htm
rsd.hjiocz.cn/088883.htm
rsd.hjiocz.cn/268263.htm
rsd.hjiocz.cn/086603.htm
rsd.hjiocz.cn/511193.htm
rss.hjiocz.cn/979933.htm
rss.hjiocz.cn/266463.htm
rss.hjiocz.cn/820483.htm
rss.hjiocz.cn/371373.htm
rss.hjiocz.cn/622823.htm
rss.hjiocz.cn/424683.htm
rss.hjiocz.cn/204623.htm
rss.hjiocz.cn/684043.htm
rss.hjiocz.cn/288683.htm
rss.hjiocz.cn/682803.htm
rsa.hjiocz.cn/824063.htm
rsa.hjiocz.cn/466803.htm
rsa.hjiocz.cn/371553.htm
rsa.hjiocz.cn/240423.htm
rsa.hjiocz.cn/511733.htm
rsa.hjiocz.cn/086243.htm
rsa.hjiocz.cn/735573.htm
rsa.hjiocz.cn/860063.htm
rsa.hjiocz.cn/000683.htm
rsa.hjiocz.cn/606843.htm
rsp.mmmxz.cn/537753.htm
rsp.mmmxz.cn/408223.htm
rsp.mmmxz.cn/119193.htm
rsp.mmmxz.cn/317553.htm
rsp.mmmxz.cn/406683.htm
rsp.mmmxz.cn/139553.htm
rsp.mmmxz.cn/919113.htm
rsp.mmmxz.cn/608203.htm
rsp.mmmxz.cn/733553.htm
rsp.mmmxz.cn/973953.htm
rso.mmmxz.cn/755793.htm
rso.mmmxz.cn/995513.htm
rso.mmmxz.cn/264623.htm
rso.mmmxz.cn/557173.htm
rso.mmmxz.cn/888003.htm
rso.mmmxz.cn/757333.htm
rso.mmmxz.cn/579393.htm
rso.mmmxz.cn/048003.htm
rso.mmmxz.cn/317993.htm
rso.mmmxz.cn/317973.htm
rsi.mmmxz.cn/313513.htm
rsi.mmmxz.cn/595313.htm
rsi.mmmxz.cn/393173.htm
rsi.mmmxz.cn/486623.htm
rsi.mmmxz.cn/575793.htm
rsi.mmmxz.cn/731133.htm
rsi.mmmxz.cn/604003.htm
rsi.mmmxz.cn/791953.htm
rsi.mmmxz.cn/082403.htm
rsi.mmmxz.cn/915333.htm
rsu.mmmxz.cn/820223.htm
rsu.mmmxz.cn/311733.htm
rsu.mmmxz.cn/060403.htm
rsu.mmmxz.cn/626423.htm
rsu.mmmxz.cn/171153.htm
rsu.mmmxz.cn/822803.htm
rsu.mmmxz.cn/371353.htm
rsu.mmmxz.cn/462683.htm
rsu.mmmxz.cn/751533.htm
rsu.mmmxz.cn/208643.htm
rsy.mmmxz.cn/464043.htm
rsy.mmmxz.cn/531153.htm
rsy.mmmxz.cn/959333.htm
rsy.mmmxz.cn/315113.htm
rsy.mmmxz.cn/533533.htm
rsy.mmmxz.cn/286823.htm
rsy.mmmxz.cn/884403.htm
rsy.mmmxz.cn/866623.htm
rsy.mmmxz.cn/537313.htm
rsy.mmmxz.cn/820643.htm
rst.mmmxz.cn/668463.htm
rst.mmmxz.cn/824463.htm
rst.mmmxz.cn/715113.htm
rst.mmmxz.cn/080463.htm
rst.mmmxz.cn/482863.htm
rst.mmmxz.cn/511913.htm
rst.mmmxz.cn/131713.htm
rst.mmmxz.cn/139593.htm
rst.mmmxz.cn/111593.htm
rst.mmmxz.cn/044623.htm
rsr.mmmxz.cn/008803.htm
rsr.mmmxz.cn/842083.htm
rsr.mmmxz.cn/375933.htm
rsr.mmmxz.cn/222023.htm
rsr.mmmxz.cn/400203.htm
rsr.mmmxz.cn/408063.htm
rsr.mmmxz.cn/133333.htm
rsr.mmmxz.cn/751913.htm
rsr.mmmxz.cn/915713.htm
rsr.mmmxz.cn/153593.htm
rse.mmmxz.cn/662483.htm
rse.mmmxz.cn/224883.htm
rse.mmmxz.cn/153173.htm
rse.mmmxz.cn/466023.htm
rse.mmmxz.cn/420823.htm
rse.mmmxz.cn/008643.htm
rse.mmmxz.cn/428243.htm
rse.mmmxz.cn/713333.htm
rse.mmmxz.cn/802663.htm
rse.mmmxz.cn/513373.htm
rsw.mmmxz.cn/862443.htm
rsw.mmmxz.cn/606063.htm
rsw.mmmxz.cn/860843.htm
rsw.mmmxz.cn/937553.htm
rsw.mmmxz.cn/577353.htm
rsw.mmmxz.cn/268043.htm
rsw.mmmxz.cn/446083.htm
rsw.mmmxz.cn/444483.htm
rsw.mmmxz.cn/206623.htm
rsw.mmmxz.cn/208423.htm
rsq.mmmxz.cn/280823.htm
rsq.mmmxz.cn/226843.htm
rsq.mmmxz.cn/006483.htm
rsq.mmmxz.cn/408663.htm
rsq.mmmxz.cn/220823.htm
rsq.mmmxz.cn/626243.htm
rsq.mmmxz.cn/777393.htm
rsq.mmmxz.cn/800603.htm
rsq.mmmxz.cn/135393.htm
rsq.mmmxz.cn/802803.htm
ram.mmmxz.cn/155533.htm
ram.mmmxz.cn/937333.htm
ram.mmmxz.cn/460283.htm
ram.mmmxz.cn/155333.htm
ram.mmmxz.cn/806043.htm
ram.mmmxz.cn/000203.htm
ram.mmmxz.cn/557573.htm
ram.mmmxz.cn/604423.htm
ram.mmmxz.cn/228643.htm
ram.mmmxz.cn/204843.htm
ran.mmmxz.cn/484463.htm
ran.mmmxz.cn/951313.htm
ran.mmmxz.cn/739193.htm
ran.mmmxz.cn/662483.htm
ran.mmmxz.cn/088283.htm
ran.mmmxz.cn/242643.htm
ran.mmmxz.cn/862603.htm
ran.mmmxz.cn/260283.htm
ran.mmmxz.cn/422883.htm
ran.mmmxz.cn/682243.htm
rab.mmmxz.cn/644283.htm
rab.mmmxz.cn/395133.htm
rab.mmmxz.cn/888683.htm
rab.mmmxz.cn/244043.htm
rab.mmmxz.cn/828243.htm
rab.mmmxz.cn/197993.htm
rab.mmmxz.cn/062223.htm
rab.mmmxz.cn/422243.htm
rab.mmmxz.cn/622263.htm
rab.mmmxz.cn/113793.htm
rav.mmmxz.cn/315793.htm
rav.mmmxz.cn/686403.htm
rav.mmmxz.cn/660203.htm
rav.mmmxz.cn/791173.htm
rav.mmmxz.cn/179133.htm
rav.mmmxz.cn/539753.htm
rav.mmmxz.cn/317513.htm
rav.mmmxz.cn/080463.htm
rav.mmmxz.cn/759713.htm
rav.mmmxz.cn/111333.htm
rac.mmmxz.cn/488483.htm
rac.mmmxz.cn/088403.htm
rac.mmmxz.cn/266663.htm
rac.mmmxz.cn/597913.htm
rac.mmmxz.cn/597753.htm
rac.mmmxz.cn/262223.htm
rac.mmmxz.cn/597733.htm
rac.mmmxz.cn/484443.htm
rac.mmmxz.cn/359173.htm
rac.mmmxz.cn/171193.htm
rax.mmmxz.cn/488063.htm
rax.mmmxz.cn/391993.htm
rax.mmmxz.cn/404283.htm
rax.mmmxz.cn/028003.htm
rax.mmmxz.cn/557333.htm
rax.mmmxz.cn/262283.htm
rax.mmmxz.cn/220023.htm
rax.mmmxz.cn/575513.htm
rax.mmmxz.cn/666023.htm
rax.mmmxz.cn/002223.htm
raz.mmmxz.cn/666423.htm
raz.mmmxz.cn/622863.htm
raz.mmmxz.cn/846203.htm
raz.mmmxz.cn/242203.htm
raz.mmmxz.cn/915533.htm
raz.mmmxz.cn/642023.htm
raz.mmmxz.cn/115113.htm
raz.mmmxz.cn/806883.htm
raz.mmmxz.cn/228623.htm
raz.mmmxz.cn/440403.htm
ral.mmmxz.cn/935933.htm
ral.mmmxz.cn/911973.htm
ral.mmmxz.cn/004063.htm
ral.mmmxz.cn/640083.htm
ral.mmmxz.cn/335593.htm
ral.mmmxz.cn/317573.htm
ral.mmmxz.cn/486423.htm
ral.mmmxz.cn/551133.htm
ral.mmmxz.cn/860843.htm
ral.mmmxz.cn/117353.htm
rak.mmmxz.cn/715773.htm
rak.mmmxz.cn/644083.htm
rak.mmmxz.cn/400403.htm
rak.mmmxz.cn/977173.htm
rak.mmmxz.cn/933153.htm
rak.mmmxz.cn/004083.htm
rak.mmmxz.cn/937973.htm
rak.mmmxz.cn/248223.htm
rak.mmmxz.cn/933713.htm
rak.mmmxz.cn/513773.htm
raj.mmmxz.cn/824283.htm
raj.mmmxz.cn/286683.htm
raj.mmmxz.cn/375173.htm
raj.mmmxz.cn/177353.htm
raj.mmmxz.cn/802643.htm
raj.mmmxz.cn/648283.htm
raj.mmmxz.cn/420023.htm
raj.mmmxz.cn/662663.htm
raj.mmmxz.cn/911393.htm
raj.mmmxz.cn/086483.htm
rah.mmmxz.cn/804283.htm
rah.mmmxz.cn/359173.htm
rah.mmmxz.cn/460463.htm
rah.mmmxz.cn/044683.htm
rah.mmmxz.cn/282003.htm
rah.mmmxz.cn/719513.htm
rah.mmmxz.cn/062423.htm
rah.mmmxz.cn/155113.htm
rah.mmmxz.cn/446623.htm
rah.mmmxz.cn/397373.htm
rag.mmmxz.cn/864283.htm
rag.mmmxz.cn/260823.htm
rag.mmmxz.cn/686823.htm
rag.mmmxz.cn/666423.htm
rag.mmmxz.cn/393193.htm
rag.mmmxz.cn/460063.htm
rag.mmmxz.cn/044263.htm
rag.mmmxz.cn/264003.htm
rag.mmmxz.cn/824863.htm
rag.mmmxz.cn/995513.htm
raf.mmmxz.cn/840043.htm
raf.mmmxz.cn/713713.htm
raf.mmmxz.cn/179773.htm
raf.mmmxz.cn/046203.htm
raf.mmmxz.cn/864483.htm
raf.mmmxz.cn/020883.htm
raf.mmmxz.cn/159173.htm
raf.mmmxz.cn/551553.htm
raf.mmmxz.cn/171953.htm
raf.mmmxz.cn/442823.htm
rad.mmmxz.cn/355553.htm
rad.mmmxz.cn/808063.htm
rad.mmmxz.cn/228263.htm
rad.mmmxz.cn/066043.htm
rad.mmmxz.cn/486283.htm
rad.mmmxz.cn/228603.htm
rad.mmmxz.cn/137333.htm
rad.mmmxz.cn/973333.htm
rad.mmmxz.cn/397513.htm
rad.mmmxz.cn/339793.htm
ras.mmmxz.cn/137353.htm
ras.mmmxz.cn/337553.htm
ras.mmmxz.cn/319593.htm
ras.mmmxz.cn/646063.htm
ras.mmmxz.cn/202083.htm
ras.mmmxz.cn/266843.htm
ras.mmmxz.cn/791753.htm
ras.mmmxz.cn/797353.htm
ras.mmmxz.cn/973973.htm
ras.mmmxz.cn/339573.htm
raa.mmmxz.cn/975553.htm
raa.mmmxz.cn/006603.htm
raa.mmmxz.cn/573993.htm
raa.mmmxz.cn/133373.htm
raa.mmmxz.cn/935973.htm
