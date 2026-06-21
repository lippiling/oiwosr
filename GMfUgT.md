障嘶诶松诺


============================================================
 Python 事件循环深入：从底层 API 到高级用法
============================================================

事件循环（Event Loop）是 asyncio 的核心调度器，负责管理协程、
回调、IO 事件和定时器。本文将深入底层 API，理解其工作机制。

============================================================
 一、获取当前运行的事件循环
============================================================

import asyncio
import socket
import time

async def demo_get_loop():
    """获取当前线程中正在运行的事件循环"""
    # asyncio.get_running_loop() 只能在协程内部调用
    loop = asyncio.get_running_loop()
    print(f"当前事件循环: {loop}")
    print(f"循环是否在运行: {loop.is_running()}")
    return loop

# asyncio.get_event_loop() 已废弃，3.12+ 会发出警告
# 推荐始终使用 get_running_loop()

============================================================
 二、手动管理事件循环：run_until_complete vs run_forever
============================================================

async def task(name: str, delay: float):
    """模拟一个耗时任务"""
    await asyncio.sleep(delay)
    print(f"任务 {name} 完成，耗时 {delay} 秒")
    return f"{name} 结果"

def manual_loop_demo():
    """演示手动创建和运行事件循环"""
    # 创建一个新的事件循环
    loop = asyncio.new_event_loop()
    # 设置为当前线程的默认循环
    asyncio.set_event_loop(loop)

    try:
        # run_until_complete: 运行直到协程完成
        result = loop.run_until_complete(task("A", 0.5))
        print(f"run_until_complete 返回: {result}")

        # run_forever: 持续运行，直到调用 loop.stop()
        # 一般配合 call_soon 使用
        def stop_loop():
            print("3 秒到了，停止事件循环")
            loop.stop()

        loop.call_later(3, stop_loop)  # 3 秒后执行回调
        print("进入 run_forever，将在 3 秒后停止...")
        loop.run_forever()
    finally:
        # 重要：必须关闭循环释放资源
        loop.close()
        print("事件循环已关闭")

============================================================
 三、call_soon / call_later / call_at：调度回调
============================================================

def schedule_callbacks_demo():
    """展示三种回调调度方式"""
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    start = loop.time()  # 获取循环内部时间（单调时钟）

    def callback(msg: str):
        """普通回调函数"""
        now = loop.time() - start
        print(f"[+{now:.2f}s] {msg}")

    # call_soon: 下一次迭代立即执行（按添加顺序）
    loop.call_soon(callback, "call_soon #1")
    loop.call_soon(callback, "call_soon #2")

    # call_later: 延迟指定秒数后执行
    loop.call_later(1.0, callback, "call_later 1 秒")

    # call_at: 在指定的绝对时间执行（使用 loop.time()）
    target = loop.time() + 2.0
    loop.call_at(target, callback, f"call_at 2 秒后 (绝对时间)")

    # call_soon_threadsafe: 跨线程安全地调度回调
    import threading
    def from_another_thread():
        loop.call_soon_threadsafe(callback, "来自另一个线程的回调")

    t = threading.Timer(0.5, from_another_thread)
    t.start()

    loop.call_later(3, loop.stop)
    loop.run_forever()
    loop.close()

============================================================
 四、add_reader / add_writer：监听文件描述符
============================================================

async def monitor_socket_demo():
    """使用 add_reader 监听 socket 可读事件"""
    loop = asyncio.get_running_loop()
    # 创建一个 TCP 连接用于演示
    reader, writer = await asyncio.open_connection("httpbin.org", 80)
    sock = writer.transport.get_extra_info("socket")
    sock.setblocking(False)  # 必须设为非阻塞

    def on_readable():
        """当 socket 可读时被调用"""
        try:
            data = sock.recv(4096)
            if data:
                print(f"收到数据: {data[:50]}...")
            else:
                print("连接关闭")
                loop.remove_reader(sock.fileno())  # 不再监听
        except BlockingIOError:
            pass  # 没有数据，下次再试

    # 注册文件描述符的可读事件回调
    loop.add_reader(sock.fileno(), on_readable)
    writer.write(b"GET /get HTTP/1.0\r\nHost: httpbin.org\r\n\r\n")
    await writer.drain()

    await asyncio.sleep(3)
    # 清理：移除监听
    loop.remove_reader(sock.fileno())
    # add_writer 用法类似，监听可写事件
    writer.close()

============================================================
 五、自定义事件循环策略（Event Loop Policy）
============================================================

class MyLoopPolicy(asyncio.DefaultEventLoopPolicy):
    """自定义事件循环策略，在创建循环时添加日志"""

    def get_event_loop(self):
        loop = super().get_event_loop()
        print(f"[策略] 获取事件循环: {loop}")
        return loop

    def new_event_loop(self):
        loop = super().new_event_loop()
        print(f"[策略] 创建新事件循环: {loop}")
        return loop

async def policy_demo():
    """演示自定义策略的效果"""
    loop = asyncio.get_running_loop()
    print(f"运行在: {loop}")
    await asyncio.sleep(0.1)

# 使用自定义策略
# asyncio.set_event_loop_policy(MyLoopPolicy())
# asyncio.run(policy_demo())

============================================================
 六、asyncio.run vs 手动管理循环
============================================================

async def cleanup_demo():
    """演示协程中的资源清理"""
    try:
        print("协程运行中...")
        await asyncio.sleep(1)
    finally:
        print("协程清理完成")

def compare_run_vs_manual():
    """
    asyncio.run() 是高级 API，内部处理了：
    1. 创建新的事件循环
    2. 运行协程直到完成
    3. 自动关闭循环
    4. 取消所有未完成的任务
    5. 清理异步生成器
    """
    # 推荐方式（Python 3.7+）
    asyncio.run(cleanup_demo())

    # 等价的手动方式（不推荐）
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    try:
        loop.run_until_complete(cleanup_demo())
    finally:
        # 手动关闭前取消所有任务
        for task in asyncio.all_tasks(loop):
            task.cancel()
        loop.run_until_complete(loop.shutdown_asyncgens())
        loop.close()

============================================================
 七、总结
============================================================

# 要点回顾：
# 1. get_running_loop() 是获取当前循环的标准方式
# 2. run_until_complete 与 run_forever 有不同的应用场景
# 3. call_soon/call_later/call_at 调度普通回调到事件循环
# 4. add_reader/add_writer 实现底层 IO 多路复用
# 5. 自定义策略可控制事件循环的创建行为
# 6. asyncio.run() 是最安全简便的入口点
# 7. 手动管理循环时必须调用 loop.close() 释放资源

if __name__ == "__main__":
    compare_run_vs_manual()

那换剐允粤换煽牟钒坏朴芬靖丈氐

m.qso.msfsx.cn/06682.Doc
m.qso.msfsx.cn/26868.Doc
m.qso.msfsx.cn/22068.Doc
m.qso.msfsx.cn/66022.Doc
m.qso.msfsx.cn/26886.Doc
m.qso.msfsx.cn/86820.Doc
m.qso.msfsx.cn/20406.Doc
m.qsi.msfsx.cn/64886.Doc
m.qsi.msfsx.cn/44442.Doc
m.qsi.msfsx.cn/84648.Doc
m.qsi.msfsx.cn/68040.Doc
m.qsi.msfsx.cn/11319.Doc
m.qsi.msfsx.cn/62882.Doc
m.qsi.msfsx.cn/71551.Doc
m.qsi.msfsx.cn/40048.Doc
m.qsi.msfsx.cn/04424.Doc
m.qsi.msfsx.cn/60868.Doc
m.qsi.msfsx.cn/08604.Doc
m.qsi.msfsx.cn/84004.Doc
m.qsi.msfsx.cn/68846.Doc
m.qsi.msfsx.cn/22260.Doc
m.qsi.msfsx.cn/62484.Doc
m.qsi.msfsx.cn/02280.Doc
m.qsi.msfsx.cn/26602.Doc
m.qsi.msfsx.cn/40828.Doc
m.qsi.msfsx.cn/75393.Doc
m.qsi.msfsx.cn/11791.Doc
m.qsu.msfsx.cn/48602.Doc
m.qsu.msfsx.cn/44800.Doc
m.qsu.msfsx.cn/68608.Doc
m.qsu.msfsx.cn/60604.Doc
m.qsu.msfsx.cn/82860.Doc
m.qsu.msfsx.cn/59955.Doc
m.qsu.msfsx.cn/11195.Doc
m.qsu.msfsx.cn/06068.Doc
m.qsu.msfsx.cn/24226.Doc
m.qsu.msfsx.cn/44842.Doc
m.qsu.msfsx.cn/24006.Doc
m.qsu.msfsx.cn/60686.Doc
m.qsu.msfsx.cn/60068.Doc
m.qsu.msfsx.cn/28268.Doc
m.qsu.msfsx.cn/24040.Doc
m.qsu.msfsx.cn/64828.Doc
m.qsu.msfsx.cn/73795.Doc
m.qsu.msfsx.cn/04224.Doc
m.qsu.msfsx.cn/08604.Doc
m.qsu.msfsx.cn/24600.Doc
m.qsy.msfsx.cn/22480.Doc
m.qsy.msfsx.cn/46468.Doc
m.qsy.msfsx.cn/28220.Doc
m.qsy.msfsx.cn/68682.Doc
m.qsy.msfsx.cn/51553.Doc
m.qsy.msfsx.cn/86888.Doc
m.qsy.msfsx.cn/66624.Doc
m.qsy.msfsx.cn/24824.Doc
m.qsy.msfsx.cn/57951.Doc
m.qsy.msfsx.cn/04240.Doc
m.qsy.msfsx.cn/86486.Doc
m.qsy.msfsx.cn/40260.Doc
m.qsy.msfsx.cn/44064.Doc
m.qsy.msfsx.cn/46424.Doc
m.qsy.msfsx.cn/22020.Doc
m.qsy.msfsx.cn/86880.Doc
m.qsy.msfsx.cn/48042.Doc
m.qsy.msfsx.cn/22480.Doc
m.qsy.msfsx.cn/17935.Doc
m.qsy.msfsx.cn/44224.Doc
m.qst.msfsx.cn/68888.Doc
m.qst.msfsx.cn/66202.Doc
m.qst.msfsx.cn/86406.Doc
m.qst.msfsx.cn/42404.Doc
m.qst.msfsx.cn/24044.Doc
m.qst.msfsx.cn/84020.Doc
m.qst.msfsx.cn/68246.Doc
m.qst.msfsx.cn/73397.Doc
m.qst.msfsx.cn/66622.Doc
m.qst.msfsx.cn/11939.Doc
m.qst.msfsx.cn/28006.Doc
m.qst.msfsx.cn/26266.Doc
m.qst.msfsx.cn/04842.Doc
m.qst.msfsx.cn/68208.Doc
m.qst.msfsx.cn/66468.Doc
m.qst.msfsx.cn/22844.Doc
m.qst.msfsx.cn/60868.Doc
m.qst.msfsx.cn/88244.Doc
m.qst.msfsx.cn/44228.Doc
m.qst.msfsx.cn/84084.Doc
m.qsr.msfsx.cn/48864.Doc
m.qsr.msfsx.cn/99979.Doc
m.qsr.msfsx.cn/62086.Doc
m.qsr.msfsx.cn/28002.Doc
m.qsr.msfsx.cn/75117.Doc
m.qsr.msfsx.cn/06200.Doc
m.qsr.msfsx.cn/24280.Doc
m.qsr.msfsx.cn/06420.Doc
m.qsr.msfsx.cn/04668.Doc
m.qsr.msfsx.cn/80624.Doc
m.qsr.msfsx.cn/24284.Doc
m.qsr.msfsx.cn/44422.Doc
m.qsr.msfsx.cn/48288.Doc
m.qsr.msfsx.cn/00448.Doc
m.qsr.msfsx.cn/66224.Doc
m.qsr.msfsx.cn/04044.Doc
m.qsr.msfsx.cn/86688.Doc
m.qsr.msfsx.cn/08468.Doc
m.qsr.msfsx.cn/64488.Doc
m.qsr.msfsx.cn/60426.Doc
m.qse.msfsx.cn/68806.Doc
m.qse.msfsx.cn/62246.Doc
m.qse.msfsx.cn/48426.Doc
m.qse.msfsx.cn/40624.Doc
m.qse.msfsx.cn/68244.Doc
m.qse.msfsx.cn/84802.Doc
m.qse.msfsx.cn/86084.Doc
m.qse.msfsx.cn/66044.Doc
m.qse.msfsx.cn/00486.Doc
m.qse.msfsx.cn/86666.Doc
m.qse.msfsx.cn/57755.Doc
m.qse.msfsx.cn/59515.Doc
m.qse.msfsx.cn/20808.Doc
m.qse.msfsx.cn/26888.Doc
m.qse.msfsx.cn/00628.Doc
m.qse.msfsx.cn/62462.Doc
m.qse.msfsx.cn/68284.Doc
m.qse.msfsx.cn/42866.Doc
m.qse.msfsx.cn/82802.Doc
m.qse.msfsx.cn/59311.Doc
m.qsw.msfsx.cn/82042.Doc
m.qsw.msfsx.cn/02866.Doc
m.qsw.msfsx.cn/80440.Doc
m.qsw.msfsx.cn/57919.Doc
m.qsw.msfsx.cn/46004.Doc
m.qsw.msfsx.cn/66644.Doc
m.qsw.msfsx.cn/53775.Doc
m.qsw.msfsx.cn/84868.Doc
m.qsw.msfsx.cn/53751.Doc
m.qsw.msfsx.cn/99951.Doc
m.qsw.msfsx.cn/26864.Doc
m.qsw.msfsx.cn/02864.Doc
m.qsw.msfsx.cn/66602.Doc
m.qsw.msfsx.cn/80466.Doc
m.qsw.msfsx.cn/28066.Doc
m.qsw.msfsx.cn/46008.Doc
m.qsw.msfsx.cn/80628.Doc
m.qsw.msfsx.cn/42848.Doc
m.qsw.msfsx.cn/86880.Doc
m.qsw.msfsx.cn/20222.Doc
m.qsq.msfsx.cn/40066.Doc
m.qsq.msfsx.cn/95591.Doc
m.qsq.msfsx.cn/82400.Doc
m.qsq.msfsx.cn/60262.Doc
m.qsq.msfsx.cn/62640.Doc
m.qsq.msfsx.cn/46880.Doc
m.qsq.msfsx.cn/04002.Doc
m.qsq.msfsx.cn/06864.Doc
m.qsq.msfsx.cn/02884.Doc
m.qsq.msfsx.cn/60680.Doc
m.qsq.msfsx.cn/42460.Doc
m.qsq.msfsx.cn/24484.Doc
m.qsq.msfsx.cn/97395.Doc
m.qsq.msfsx.cn/91911.Doc
m.qsq.msfsx.cn/66668.Doc
m.qsq.msfsx.cn/79317.Doc
m.qsq.msfsx.cn/91577.Doc
m.qsq.msfsx.cn/22044.Doc
m.qsq.msfsx.cn/35173.Doc
m.qsq.msfsx.cn/86600.Doc
m.qam.hxbsg.cn/64268.Doc
m.qam.hxbsg.cn/06668.Doc
m.qam.hxbsg.cn/20886.Doc
m.qam.hxbsg.cn/77995.Doc
m.qam.hxbsg.cn/04464.Doc
m.qam.hxbsg.cn/42026.Doc
m.qam.hxbsg.cn/46286.Doc
m.qam.hxbsg.cn/53393.Doc
m.qam.hxbsg.cn/13333.Doc
m.qam.hxbsg.cn/60802.Doc
m.qam.hxbsg.cn/88026.Doc
m.qam.hxbsg.cn/15157.Doc
m.qam.hxbsg.cn/04448.Doc
m.qam.hxbsg.cn/39731.Doc
m.qam.hxbsg.cn/24224.Doc
m.qam.hxbsg.cn/88400.Doc
m.qam.hxbsg.cn/88464.Doc
m.qam.hxbsg.cn/00242.Doc
m.qam.hxbsg.cn/66288.Doc
m.qam.hxbsg.cn/31157.Doc
m.qan.hxbsg.cn/88642.Doc
m.qan.hxbsg.cn/62244.Doc
m.qan.hxbsg.cn/06000.Doc
m.qan.hxbsg.cn/48842.Doc
m.qan.hxbsg.cn/46486.Doc
m.qan.hxbsg.cn/44882.Doc
m.qan.hxbsg.cn/57559.Doc
m.qan.hxbsg.cn/88204.Doc
m.qan.hxbsg.cn/86600.Doc
m.qan.hxbsg.cn/08408.Doc
m.qan.hxbsg.cn/48200.Doc
m.qan.hxbsg.cn/48082.Doc
m.qan.hxbsg.cn/64084.Doc
m.qan.hxbsg.cn/08422.Doc
m.qan.hxbsg.cn/11973.Doc
m.qan.hxbsg.cn/66286.Doc
m.qan.hxbsg.cn/71331.Doc
m.qan.hxbsg.cn/40026.Doc
m.qan.hxbsg.cn/60404.Doc
m.qan.hxbsg.cn/39939.Doc
m.qab.hxbsg.cn/08624.Doc
m.qab.hxbsg.cn/66482.Doc
m.qab.hxbsg.cn/04002.Doc
m.qab.hxbsg.cn/86882.Doc
m.qab.hxbsg.cn/57577.Doc
m.qab.hxbsg.cn/22286.Doc
m.qab.hxbsg.cn/28842.Doc
m.qab.hxbsg.cn/26626.Doc
m.qab.hxbsg.cn/88600.Doc
m.qab.hxbsg.cn/20826.Doc
m.qab.hxbsg.cn/84024.Doc
m.qab.hxbsg.cn/00464.Doc
m.qab.hxbsg.cn/28062.Doc
m.qab.hxbsg.cn/86048.Doc
m.qab.hxbsg.cn/39595.Doc
m.qab.hxbsg.cn/46886.Doc
m.qab.hxbsg.cn/28000.Doc
m.qab.hxbsg.cn/24444.Doc
m.qab.hxbsg.cn/31139.Doc
m.qab.hxbsg.cn/53519.Doc
m.qav.hxbsg.cn/93551.Doc
m.qav.hxbsg.cn/26288.Doc
m.qav.hxbsg.cn/82244.Doc
m.qav.hxbsg.cn/28220.Doc
m.qav.hxbsg.cn/44488.Doc
m.qav.hxbsg.cn/66088.Doc
m.qav.hxbsg.cn/37153.Doc
m.qav.hxbsg.cn/66020.Doc
m.qav.hxbsg.cn/71739.Doc
m.qav.hxbsg.cn/02208.Doc
m.qav.hxbsg.cn/40062.Doc
m.qav.hxbsg.cn/48286.Doc
m.qav.hxbsg.cn/62864.Doc
m.qav.hxbsg.cn/93197.Doc
m.qav.hxbsg.cn/53731.Doc
m.qav.hxbsg.cn/31373.Doc
m.qav.hxbsg.cn/06002.Doc
m.qav.hxbsg.cn/79731.Doc
m.qav.hxbsg.cn/33971.Doc
m.qav.hxbsg.cn/24622.Doc
m.qac.hxbsg.cn/24020.Doc
m.qac.hxbsg.cn/48886.Doc
m.qac.hxbsg.cn/75979.Doc
m.qac.hxbsg.cn/62062.Doc
m.qac.hxbsg.cn/86246.Doc
m.qac.hxbsg.cn/42426.Doc
m.qac.hxbsg.cn/28806.Doc
m.qac.hxbsg.cn/42462.Doc
m.qac.hxbsg.cn/37997.Doc
m.qac.hxbsg.cn/08484.Doc
m.qac.hxbsg.cn/06888.Doc
m.qac.hxbsg.cn/22402.Doc
m.qac.hxbsg.cn/86202.Doc
m.qac.hxbsg.cn/02642.Doc
m.qac.hxbsg.cn/46648.Doc
m.qac.hxbsg.cn/06200.Doc
m.qac.hxbsg.cn/84808.Doc
m.qac.hxbsg.cn/91531.Doc
m.qac.hxbsg.cn/39197.Doc
m.qac.hxbsg.cn/79955.Doc
m.qax.hxbsg.cn/44860.Doc
m.qax.hxbsg.cn/26044.Doc
m.qax.hxbsg.cn/40844.Doc
m.qax.hxbsg.cn/08620.Doc
m.qax.hxbsg.cn/82202.Doc
m.qax.hxbsg.cn/44202.Doc
m.qax.hxbsg.cn/39571.Doc
m.qax.hxbsg.cn/06420.Doc
m.qax.hxbsg.cn/00686.Doc
m.qax.hxbsg.cn/28622.Doc
m.qax.hxbsg.cn/77573.Doc
m.qax.hxbsg.cn/24884.Doc
m.qax.hxbsg.cn/33353.Doc
m.qax.hxbsg.cn/08228.Doc
m.qax.hxbsg.cn/33177.Doc
m.qax.hxbsg.cn/44620.Doc
m.qax.hxbsg.cn/24620.Doc
m.qax.hxbsg.cn/86426.Doc
m.qax.hxbsg.cn/46068.Doc
m.qax.hxbsg.cn/08400.Doc
m.qaz.hxbsg.cn/88842.Doc
m.qaz.hxbsg.cn/42648.Doc
m.qaz.hxbsg.cn/75151.Doc
m.qaz.hxbsg.cn/17779.Doc
m.qaz.hxbsg.cn/08206.Doc
m.qaz.hxbsg.cn/28868.Doc
m.qaz.hxbsg.cn/04888.Doc
m.qaz.hxbsg.cn/48642.Doc
m.qaz.hxbsg.cn/86060.Doc
m.qaz.hxbsg.cn/24024.Doc
m.qaz.hxbsg.cn/26022.Doc
m.qaz.hxbsg.cn/91335.Doc
m.qaz.hxbsg.cn/68662.Doc
m.qaz.hxbsg.cn/86842.Doc
m.qaz.hxbsg.cn/42884.Doc
m.qaz.hxbsg.cn/82842.Doc
m.qaz.hxbsg.cn/28626.Doc
m.qaz.hxbsg.cn/24444.Doc
m.qaz.hxbsg.cn/06064.Doc
m.qaz.hxbsg.cn/33911.Doc
m.qal.hxbsg.cn/86648.Doc
m.qal.hxbsg.cn/08024.Doc
m.qal.hxbsg.cn/68824.Doc
m.qal.hxbsg.cn/24222.Doc
m.qal.hxbsg.cn/06060.Doc
m.qal.hxbsg.cn/84426.Doc
m.qal.hxbsg.cn/13995.Doc
m.qal.hxbsg.cn/06846.Doc
m.qal.hxbsg.cn/84688.Doc
m.qal.hxbsg.cn/68806.Doc
m.qal.hxbsg.cn/82460.Doc
m.qal.hxbsg.cn/24600.Doc
m.qal.hxbsg.cn/31939.Doc
m.qal.hxbsg.cn/73315.Doc
m.qal.hxbsg.cn/55151.Doc
m.qal.hxbsg.cn/44044.Doc
m.qal.hxbsg.cn/79753.Doc
m.qal.hxbsg.cn/44608.Doc
m.qal.hxbsg.cn/44248.Doc
m.qal.hxbsg.cn/46268.Doc
m.qak.hxbsg.cn/64864.Doc
m.qak.hxbsg.cn/46068.Doc
m.qak.hxbsg.cn/53999.Doc
m.qak.hxbsg.cn/82488.Doc
m.qak.hxbsg.cn/08804.Doc
m.qak.hxbsg.cn/55515.Doc
m.qak.hxbsg.cn/48260.Doc
m.qak.hxbsg.cn/28622.Doc
m.qak.hxbsg.cn/17331.Doc
m.qak.hxbsg.cn/24664.Doc
m.qak.hxbsg.cn/80604.Doc
m.qak.hxbsg.cn/86668.Doc
m.qak.hxbsg.cn/00626.Doc
m.qak.hxbsg.cn/60222.Doc
m.qak.hxbsg.cn/84840.Doc
m.qak.hxbsg.cn/04888.Doc
m.qak.hxbsg.cn/82064.Doc
m.qak.hxbsg.cn/31575.Doc
m.qak.hxbsg.cn/17715.Doc
m.qak.hxbsg.cn/28684.Doc
m.qaj.hxbsg.cn/40280.Doc
m.qaj.hxbsg.cn/86042.Doc
m.qaj.hxbsg.cn/46604.Doc
m.qaj.hxbsg.cn/28628.Doc
m.qaj.hxbsg.cn/88462.Doc
m.qaj.hxbsg.cn/55399.Doc
m.qaj.hxbsg.cn/37795.Doc
m.qaj.hxbsg.cn/40060.Doc
m.qaj.hxbsg.cn/26226.Doc
m.qaj.hxbsg.cn/04048.Doc
m.qaj.hxbsg.cn/88060.Doc
m.qaj.hxbsg.cn/22462.Doc
m.qaj.hxbsg.cn/62804.Doc
m.qaj.hxbsg.cn/44286.Doc
m.qaj.hxbsg.cn/26826.Doc
m.qaj.hxbsg.cn/04404.Doc
m.qaj.hxbsg.cn/57717.Doc
m.qaj.hxbsg.cn/68048.Doc
m.qaj.hxbsg.cn/68000.Doc
m.qaj.hxbsg.cn/95537.Doc
m.qah.hxbsg.cn/00240.Doc
m.qah.hxbsg.cn/26202.Doc
m.qah.hxbsg.cn/57777.Doc
m.qah.hxbsg.cn/48282.Doc
m.qah.hxbsg.cn/28026.Doc
m.qah.hxbsg.cn/02842.Doc
m.qah.hxbsg.cn/73539.Doc
m.qah.hxbsg.cn/86264.Doc
m.qah.hxbsg.cn/80640.Doc
m.qah.hxbsg.cn/62244.Doc
m.qah.hxbsg.cn/19313.Doc
m.qah.hxbsg.cn/04024.Doc
m.qah.hxbsg.cn/60446.Doc
m.qah.hxbsg.cn/00060.Doc
m.qah.hxbsg.cn/20240.Doc
m.qah.hxbsg.cn/80000.Doc
m.qah.hxbsg.cn/60424.Doc
m.qah.hxbsg.cn/06804.Doc
m.qah.hxbsg.cn/00866.Doc
m.qah.hxbsg.cn/15179.Doc
m.qag.hxbsg.cn/19937.Doc
m.qag.hxbsg.cn/66246.Doc
m.qag.hxbsg.cn/80848.Doc
m.qag.hxbsg.cn/53511.Doc
m.qag.hxbsg.cn/28042.Doc
m.qag.hxbsg.cn/26684.Doc
m.qag.hxbsg.cn/66668.Doc
m.qag.hxbsg.cn/26426.Doc
