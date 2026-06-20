认隙奖温碌


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

蔷椿粗婪傥兔菇刳僦饲掳目幢彩蒂

yss.ugioc26.cn/951155.htm
yss.ugioc26.cn/593715.htm
yss.ugioc26.cn/393975.htm
yss.ugioc26.cn/377315.htm
yss.ugioc26.cn/402645.htm
yss.ugioc26.cn/400805.htm
yss.ugioc26.cn/064865.htm
yss.ugioc26.cn/339715.htm
ysa.ugioc26.cn/800425.htm
ysa.ugioc26.cn/684485.htm
ysa.ugioc26.cn/086625.htm
ysa.ugioc26.cn/842485.htm
ysa.ugioc26.cn/602465.htm
ysa.ugioc26.cn/804265.htm
ysa.ugioc26.cn/626445.htm
ysa.ugioc26.cn/357555.htm
ysa.ugioc26.cn/317735.htm
ysa.ugioc26.cn/600065.htm
ysp.ugioc26.cn/684485.htm
ysp.ugioc26.cn/933355.htm
ysp.ugioc26.cn/206845.htm
ysp.ugioc26.cn/848205.htm
ysp.ugioc26.cn/175315.htm
ysp.ugioc26.cn/484025.htm
ysp.ugioc26.cn/822885.htm
ysp.ugioc26.cn/208065.htm
ysp.ugioc26.cn/513315.htm
ysp.ugioc26.cn/482205.htm
yso.ugioc26.cn/773575.htm
yso.ugioc26.cn/220825.htm
yso.ugioc26.cn/795175.htm
yso.ugioc26.cn/468865.htm
yso.ugioc26.cn/951535.htm
yso.ugioc26.cn/648465.htm
yso.ugioc26.cn/624445.htm
yso.ugioc26.cn/422805.htm
yso.ugioc26.cn/886285.htm
yso.ugioc26.cn/828445.htm
ysi.ugioc26.cn/955795.htm
ysi.ugioc26.cn/399335.htm
ysi.ugioc26.cn/335595.htm
ysi.ugioc26.cn/339195.htm
ysi.ugioc26.cn/842085.htm
ysi.ugioc26.cn/977375.htm
ysi.ugioc26.cn/822265.htm
ysi.ugioc26.cn/373355.htm
ysi.ugioc26.cn/393915.htm
ysi.ugioc26.cn/022485.htm
ysu.ugioc26.cn/771995.htm
ysu.ugioc26.cn/131795.htm
ysu.ugioc26.cn/060045.htm
ysu.ugioc26.cn/917775.htm
ysu.ugioc26.cn/820285.htm
ysu.ugioc26.cn/797755.htm
ysu.ugioc26.cn/268025.htm
ysu.ugioc26.cn/359395.htm
ysu.ugioc26.cn/682665.htm
ysu.ugioc26.cn/648885.htm
ysy.ugioc26.cn/264465.htm
ysy.ugioc26.cn/608605.htm
ysy.ugioc26.cn/084245.htm
ysy.ugioc26.cn/000205.htm
ysy.ugioc26.cn/626665.htm
ysy.ugioc26.cn/222045.htm
ysy.ugioc26.cn/397595.htm
ysy.ugioc26.cn/353115.htm
ysy.ugioc26.cn/953555.htm
ysy.ugioc26.cn/482225.htm
yst.ugioc26.cn/735315.htm
yst.ugioc26.cn/608625.htm
yst.ugioc26.cn/088605.htm
yst.ugioc26.cn/668285.htm
yst.ugioc26.cn/379915.htm
yst.ugioc26.cn/595995.htm
yst.ugioc26.cn/826205.htm
yst.ugioc26.cn/020245.htm
yst.ugioc26.cn/444405.htm
yst.ugioc26.cn/862865.htm
ysr.ugioc26.cn/000245.htm
ysr.ugioc26.cn/731595.htm
ysr.ugioc26.cn/264425.htm
ysr.ugioc26.cn/113915.htm
ysr.ugioc26.cn/755535.htm
ysr.ugioc26.cn/791955.htm
ysr.ugioc26.cn/206485.htm
ysr.ugioc26.cn/242645.htm
ysr.ugioc26.cn/422285.htm
ysr.ugioc26.cn/680845.htm
yse.ugioc26.cn/482245.htm
yse.ugioc26.cn/737755.htm
yse.ugioc26.cn/264425.htm
yse.ugioc26.cn/860865.htm
yse.ugioc26.cn/264265.htm
yse.ugioc26.cn/513375.htm
yse.ugioc26.cn/622445.htm
yse.ugioc26.cn/664605.htm
yse.ugioc26.cn/139175.htm
yse.ugioc26.cn/882005.htm
ysw.ugioc26.cn/773135.htm
ysw.ugioc26.cn/062285.htm
ysw.ugioc26.cn/468805.htm
ysw.ugioc26.cn/595575.htm
ysw.ugioc26.cn/686085.htm
ysw.ugioc26.cn/464885.htm
ysw.ugioc26.cn/048865.htm
ysw.ugioc26.cn/208285.htm
ysw.ugioc26.cn/444845.htm
ysw.ugioc26.cn/606265.htm
ysq.ugioc26.cn/684425.htm
ysq.ugioc26.cn/260245.htm
ysq.ugioc26.cn/139955.htm
ysq.ugioc26.cn/153555.htm
ysq.ugioc26.cn/797195.htm
ysq.ugioc26.cn/395515.htm
ysq.ugioc26.cn/482405.htm
ysq.ugioc26.cn/959755.htm
ysq.ugioc26.cn/680465.htm
ysq.ugioc26.cn/737755.htm
yatv.ugioc26.cn/284025.htm
yatv.ugioc26.cn/488665.htm
yatv.ugioc26.cn/008285.htm
yatv.ugioc26.cn/975935.htm
yatv.ugioc26.cn/391335.htm
yatv.ugioc26.cn/428625.htm
yatv.ugioc26.cn/733975.htm
yatv.ugioc26.cn/173575.htm
yatv.ugioc26.cn/359715.htm
yatv.ugioc26.cn/317575.htm
yan.ugioc26.cn/260825.htm
yan.ugioc26.cn/444265.htm
yan.ugioc26.cn/377535.htm
yan.ugioc26.cn/517975.htm
yan.ugioc26.cn/624225.htm
yan.ugioc26.cn/888445.htm
yan.ugioc26.cn/442245.htm
yan.ugioc26.cn/511735.htm
yan.ugioc26.cn/406485.htm
yan.ugioc26.cn/842205.htm
yab.ugioc26.cn/557735.htm
yab.ugioc26.cn/593975.htm
yab.ugioc26.cn/999715.htm
yab.ugioc26.cn/666645.htm
yab.ugioc26.cn/206045.htm
yab.ugioc26.cn/028025.htm
yab.ugioc26.cn/282805.htm
yab.ugioc26.cn/464485.htm
yab.ugioc26.cn/264805.htm
yab.ugioc26.cn/404445.htm
yav.ugioc26.cn/913155.htm
yav.ugioc26.cn/424285.htm
yav.ugioc26.cn/557195.htm
yav.ugioc26.cn/048005.htm
yav.ugioc26.cn/686845.htm
yav.ugioc26.cn/888645.htm
yav.ugioc26.cn/999355.htm
yav.ugioc26.cn/997315.htm
yav.ugioc26.cn/248625.htm
yav.ugioc26.cn/193335.htm
yac.ugioc26.cn/759575.htm
yac.ugioc26.cn/246205.htm
yac.ugioc26.cn/115775.htm
yac.ugioc26.cn/628005.htm
yac.ugioc26.cn/062065.htm
yac.ugioc26.cn/080465.htm
yac.ugioc26.cn/046605.htm
yac.ugioc26.cn/660845.htm
yac.ugioc26.cn/911195.htm
yac.ugioc26.cn/468645.htm
yax.ugioc26.cn/682625.htm
yax.ugioc26.cn/537375.htm
yax.ugioc26.cn/620865.htm
yax.ugioc26.cn/222885.htm
yax.ugioc26.cn/420885.htm
yax.ugioc26.cn/377775.htm
yax.ugioc26.cn/842225.htm
yax.ugioc26.cn/939575.htm
yax.ugioc26.cn/004425.htm
yax.ugioc26.cn/826825.htm
yaz.ugioc26.cn/062445.htm
yaz.ugioc26.cn/620045.htm
yaz.ugioc26.cn/008065.htm
yaz.ugioc26.cn/735395.htm
yaz.ugioc26.cn/284625.htm
yaz.ugioc26.cn/626825.htm
yaz.ugioc26.cn/426245.htm
yaz.ugioc26.cn/246425.htm
yaz.ugioc26.cn/662405.htm
yaz.ugioc26.cn/200425.htm
yal.ugioc26.cn/535515.htm
yal.ugioc26.cn/866625.htm
yal.ugioc26.cn/622205.htm
yal.ugioc26.cn/860005.htm
yal.ugioc26.cn/002225.htm
yal.ugioc26.cn/468605.htm
yal.ugioc26.cn/668225.htm
yal.ugioc26.cn/866205.htm
yal.ugioc26.cn/662445.htm
yal.ugioc26.cn/240865.htm
yak.ugioc26.cn/442665.htm
yak.ugioc26.cn/480425.htm
yak.ugioc26.cn/068805.htm
yak.ugioc26.cn/444465.htm
yak.ugioc26.cn/375775.htm
yak.ugioc26.cn/111375.htm
yak.ugioc26.cn/646025.htm
yak.ugioc26.cn/264085.htm
yak.ugioc26.cn/262065.htm
yak.ugioc26.cn/537975.htm
yaj.ugioc26.cn/202085.htm
yaj.ugioc26.cn/684665.htm
yaj.ugioc26.cn/042885.htm
yaj.ugioc26.cn/113395.htm
yaj.ugioc26.cn/220065.htm
yaj.ugioc26.cn/488045.htm
yaj.ugioc26.cn/288625.htm
yaj.ugioc26.cn/282865.htm
yaj.ugioc26.cn/228425.htm
yaj.ugioc26.cn/840265.htm
yah.ugioc26.cn/373995.htm
yah.ugioc26.cn/622865.htm
yah.ugioc26.cn/642625.htm
yah.ugioc26.cn/862885.htm
yah.ugioc26.cn/484685.htm
yah.ugioc26.cn/397935.htm
yah.ugioc26.cn/246225.htm
yah.ugioc26.cn/220405.htm
yah.ugioc26.cn/282685.htm
yah.ugioc26.cn/680825.htm
yag.ugioc26.cn/866445.htm
yag.ugioc26.cn/931195.htm
yag.ugioc26.cn/688465.htm
yag.ugioc26.cn/151135.htm
yag.ugioc26.cn/800425.htm
yag.ugioc26.cn/117535.htm
yag.ugioc26.cn/464885.htm
yag.ugioc26.cn/175135.htm
yag.ugioc26.cn/242265.htm
yag.ugioc26.cn/111335.htm
yas.ugioc26.cn/206845.htm
yas.ugioc26.cn/886685.htm
yas.ugioc26.cn/082825.htm
yas.ugioc26.cn/937575.htm
yas.ugioc26.cn/599195.htm
yas.ugioc26.cn/068245.htm
yas.ugioc26.cn/288885.htm
yas.ugioc26.cn/737775.htm
yas.ugioc26.cn/200685.htm
yas.ugioc26.cn/313395.htm
yaa.ugioc26.cn/460805.htm
yaa.ugioc26.cn/999195.htm
yaa.ugioc26.cn/884485.htm
yaa.ugioc26.cn/682205.htm
yaa.ugioc26.cn/822225.htm
yaa.ugioc26.cn/886025.htm
yaa.ugioc26.cn/371715.htm
yaa.ugioc26.cn/393935.htm
yaa.ugioc26.cn/664005.htm
yaa.ugioc26.cn/864605.htm
yap.ugioc26.cn/779395.htm
yap.ugioc26.cn/599595.htm
yap.ugioc26.cn/684445.htm
yap.ugioc26.cn/319755.htm
yap.ugioc26.cn/640465.htm
yap.ugioc26.cn/531535.htm
yap.ugioc26.cn/640205.htm
yap.ugioc26.cn/482805.htm
yap.ugioc26.cn/062025.htm
yap.ugioc26.cn/024645.htm
yao.ugioc26.cn/402005.htm
yao.ugioc26.cn/131735.htm
yao.ugioc26.cn/713955.htm
yao.ugioc26.cn/808045.htm
yao.ugioc26.cn/008405.htm
yao.ugioc26.cn/573315.htm
yao.ugioc26.cn/339935.htm
yao.ugioc26.cn/684825.htm
yao.ugioc26.cn/171995.htm
yao.ugioc26.cn/242825.htm
yai.ugioc26.cn/933735.htm
yai.ugioc26.cn/480825.htm
yai.ugioc26.cn/628805.htm
yai.ugioc26.cn/682465.htm
yai.ugioc26.cn/379155.htm
yai.ugioc26.cn/175395.htm
yai.ugioc26.cn/040805.htm
yai.ugioc26.cn/028865.htm
yai.ugioc26.cn/080805.htm
yai.ugioc26.cn/717195.htm
yau.ugioc26.cn/608225.htm
yau.ugioc26.cn/139335.htm
yau.ugioc26.cn/268865.htm
yau.ugioc26.cn/713375.htm
yau.ugioc26.cn/426645.htm
yau.ugioc26.cn/684265.htm
yau.ugioc26.cn/979335.htm
yau.ugioc26.cn/153935.htm
yau.ugioc26.cn/022865.htm
yau.ugioc26.cn/844425.htm
yay.ugioc26.cn/404225.htm
yay.ugioc26.cn/282645.htm
yay.ugioc26.cn/442425.htm
yay.ugioc26.cn/624245.htm
yay.ugioc26.cn/002425.htm
yay.ugioc26.cn/862605.htm
yay.ugioc26.cn/179775.htm
yay.ugioc26.cn/440005.htm
yay.ugioc26.cn/177935.htm
yay.ugioc26.cn/062405.htm
yat.ugioc26.cn/202025.htm
yat.ugioc26.cn/020225.htm
yat.ugioc26.cn/157955.htm
yat.ugioc26.cn/557955.htm
yat.ugioc26.cn/795955.htm
yat.ugioc26.cn/268285.htm
yat.ugioc26.cn/284845.htm
yat.ugioc26.cn/713575.htm
yat.ugioc26.cn/711975.htm
yat.ugioc26.cn/339995.htm
yar.ugioc26.cn/353535.htm
yar.ugioc26.cn/795355.htm
yar.ugioc26.cn/820625.htm
yar.ugioc26.cn/802605.htm
yar.ugioc26.cn/066025.htm
yar.ugioc26.cn/373775.htm
yar.ugioc26.cn/026285.htm
yar.ugioc26.cn/559715.htm
yar.ugioc26.cn/628805.htm
yar.ugioc26.cn/793315.htm
yae.ugioc26.cn/202205.htm
yae.ugioc26.cn/311995.htm
yae.ugioc26.cn/024025.htm
yae.ugioc26.cn/353395.htm
yae.ugioc26.cn/624405.htm
yae.ugioc26.cn/620425.htm
yae.ugioc26.cn/864025.htm
yae.ugioc26.cn/286625.htm
yae.ugioc26.cn/222025.htm
yae.ugioc26.cn/644845.htm
yaw.ugioc26.cn/137775.htm
yaw.ugioc26.cn/402225.htm
yaw.ugioc26.cn/959575.htm
yaw.ugioc26.cn/400025.htm
yaw.ugioc26.cn/646025.htm
yaw.ugioc26.cn/600885.htm
yaw.ugioc26.cn/519535.htm
yaw.ugioc26.cn/080485.htm
yaw.ugioc26.cn/288845.htm
yaw.ugioc26.cn/357115.htm
yaq.ugioc26.cn/082825.htm
yaq.ugioc26.cn/337375.htm
yaq.ugioc26.cn/626205.htm
yaq.ugioc26.cn/337535.htm
yaq.ugioc26.cn/262845.htm
yaq.ugioc26.cn/064645.htm
yaq.ugioc26.cn/806285.htm
yaq.ugioc26.cn/337975.htm
yaq.ugioc26.cn/826425.htm
yaq.ugioc26.cn/046685.htm
yptv.ugioc26.cn/464485.htm
yptv.ugioc26.cn/644245.htm
yptv.ugioc26.cn/317315.htm
yptv.ugioc26.cn/466465.htm
yptv.ugioc26.cn/971915.htm
yptv.ugioc26.cn/933955.htm
yptv.ugioc26.cn/339755.htm
yptv.ugioc26.cn/115535.htm
yptv.ugioc26.cn/193175.htm
yptv.ugioc26.cn/999355.htm
ypn.ugioc26.cn/448885.htm
ypn.ugioc26.cn/957955.htm
ypn.ugioc26.cn/995555.htm
ypn.ugioc26.cn/535175.htm
ypn.ugioc26.cn/911335.htm
ypn.ugioc26.cn/224685.htm
ypn.ugioc26.cn/848865.htm
ypn.ugioc26.cn/842485.htm
ypn.ugioc26.cn/979915.htm
ypn.ugioc26.cn/513135.htm
wdo.ugioc26.cn/599195.htm
wdo.ugioc26.cn/624625.htm
wdo.ugioc26.cn/55.htm
wdo.ugioc26.cn/397935.htm
wdo.ugioc26.cn/800885.htm
wdo.ugioc26.cn/153195.htm
wdo.ugioc26.cn/991755.htm
wdo.ugioc26.cn/397595.htm
wdo.ugioc26.cn/171335.htm
wdo.ugioc26.cn/135515.htm
wdi.ugioc26.cn/115575.htm
wdi.ugioc26.cn/840025.htm
wdi.ugioc26.cn/248245.htm
wdi.ugioc26.cn/222645.htm
wdi.ugioc26.cn/664265.htm
wdi.ugioc26.cn/662865.htm
wdi.ugioc26.cn/519775.htm
