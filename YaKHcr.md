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

dhc.aa5uzL7.cn/51171.Doc
dhc.aa5uzL7.cn/55775.Doc
dhc.aa5uzL7.cn/86620.Doc
dhc.aa5uzL7.cn/48824.Doc
dhc.aa5uzL7.cn/77715.Doc
dhc.aa5uzL7.cn/59119.Doc
dhc.aa5uzL7.cn/31193.Doc
dhc.aa5uzL7.cn/59333.Doc
dhc.aa5uzL7.cn/11533.Doc
dhc.aa5uzL7.cn/97795.Doc
dhx.aa5uzL7.cn/51551.Doc
dhx.aa5uzL7.cn/91135.Doc
dhx.aa5uzL7.cn/55113.Doc
dhx.aa5uzL7.cn/33179.Doc
dhx.aa5uzL7.cn/57979.Doc
dhx.aa5uzL7.cn/95775.Doc
dhx.aa5uzL7.cn/33359.Doc
dhx.aa5uzL7.cn/73731.Doc
dhx.aa5uzL7.cn/13913.Doc
dhx.aa5uzL7.cn/57735.Doc
dhz.aa5uzL7.cn/59953.Doc
dhz.aa5uzL7.cn/35517.Doc
dhz.aa5uzL7.cn/75335.Doc
dhz.aa5uzL7.cn/55931.Doc
dhz.aa5uzL7.cn/91195.Doc
dhz.aa5uzL7.cn/39199.Doc
dhz.aa5uzL7.cn/11179.Doc
dhz.aa5uzL7.cn/11399.Doc
dhz.aa5uzL7.cn/75597.Doc
dhz.aa5uzL7.cn/71711.Doc
dhl.aa5uzL7.cn/71160.Doc
dhl.aa5uzL7.cn/75379.Doc
dhl.aa5uzL7.cn/73951.Doc
dhl.aa5uzL7.cn/11391.Doc
dhl.aa5uzL7.cn/17315.Doc
dhl.aa5uzL7.cn/31711.Doc
dhl.aa5uzL7.cn/75573.Doc
dhl.aa5uzL7.cn/57913.Doc
dhl.aa5uzL7.cn/19351.Doc
dhl.aa5uzL7.cn/91739.Doc
dhk.aa5uzL7.cn/73115.Doc
dhk.aa5uzL7.cn/57517.Doc
dhk.aa5uzL7.cn/39371.Doc
dhk.aa5uzL7.cn/73919.Doc
dhk.aa5uzL7.cn/31753.Doc
dhk.aa5uzL7.cn/1.Doc
dhk.aa5uzL7.cn/99113.Doc
dhk.aa5uzL7.cn/93535.Doc
dhk.aa5uzL7.cn/95137.Doc
dhk.aa5uzL7.cn/13131.Doc
dhj.aa5uzL7.cn/31751.Doc
dhj.aa5uzL7.cn/39395.Doc
dhj.aa5uzL7.cn/19957.Doc
dhj.aa5uzL7.cn/52284.Doc
dhj.aa5uzL7.cn/73755.Doc
dhj.aa5uzL7.cn/19599.Doc
dhj.aa5uzL7.cn/46226.Doc
dhj.aa5uzL7.cn/71537.Doc
dhj.aa5uzL7.cn/79795.Doc
dhj.aa5uzL7.cn/95573.Doc
dhh.aa5uzL7.cn/11915.Doc
dhh.aa5uzL7.cn/82460.Doc
dhh.aa5uzL7.cn/64846.Doc
dhh.aa5uzL7.cn/26800.Doc
dhh.aa5uzL7.cn/06286.Doc
dhh.aa5uzL7.cn/79317.Doc
dhh.aa5uzL7.cn/24646.Doc
dhh.aa5uzL7.cn/48008.Doc
dhh.aa5uzL7.cn/15133.Doc
dhh.aa5uzL7.cn/57337.Doc
dhg.aa5uzL7.cn/99331.Doc
dhg.aa5uzL7.cn/35775.Doc
dhg.aa5uzL7.cn/93571.Doc
dhg.aa5uzL7.cn/73133.Doc
dhg.aa5uzL7.cn/99193.Doc
dhg.aa5uzL7.cn/33911.Doc
dhg.aa5uzL7.cn/91559.Doc
dhg.aa5uzL7.cn/51577.Doc
dhg.aa5uzL7.cn/00246.Doc
dhg.aa5uzL7.cn/82482.Doc
dhf.aa5uzL7.cn/79351.Doc
dhf.aa5uzL7.cn/93391.Doc
dhf.aa5uzL7.cn/99399.Doc
dhf.aa5uzL7.cn/84424.Doc
dhf.aa5uzL7.cn/35975.Doc
dhf.aa5uzL7.cn/57937.Doc
dhf.aa5uzL7.cn/71731.Doc
dhf.aa5uzL7.cn/91371.Doc
dhf.aa5uzL7.cn/95153.Doc
dhf.aa5uzL7.cn/64484.Doc
dhd.aa5uzL7.cn/08264.Doc
dhd.aa5uzL7.cn/33551.Doc
dhd.aa5uzL7.cn/77979.Doc
dhd.aa5uzL7.cn/77135.Doc
dhd.aa5uzL7.cn/59577.Doc
dhd.aa5uzL7.cn/79593.Doc
dhd.aa5uzL7.cn/55119.Doc
dhd.aa5uzL7.cn/57737.Doc
dhd.aa5uzL7.cn/53797.Doc
dhd.aa5uzL7.cn/73955.Doc
dhs.aa5uzL7.cn/35315.Doc
dhs.aa5uzL7.cn/55775.Doc
dhs.aa5uzL7.cn/73993.Doc
dhs.aa5uzL7.cn/37599.Doc
dhs.aa5uzL7.cn/39717.Doc
dhs.aa5uzL7.cn/91795.Doc
dhs.aa5uzL7.cn/93931.Doc
dhs.aa5uzL7.cn/73133.Doc
dhs.aa5uzL7.cn/17357.Doc
dhs.aa5uzL7.cn/97559.Doc
dha.aa5uzL7.cn/79537.Doc
dha.aa5uzL7.cn/75591.Doc
dha.aa5uzL7.cn/39993.Doc
dha.aa5uzL7.cn/99935.Doc
dha.aa5uzL7.cn/11571.Doc
dha.aa5uzL7.cn/15579.Doc
dha.aa5uzL7.cn/51131.Doc
dha.aa5uzL7.cn/51595.Doc
dha.aa5uzL7.cn/51153.Doc
dha.aa5uzL7.cn/55775.Doc
dhp.aa5uzL7.cn/42424.Doc
dhp.aa5uzL7.cn/33119.Doc
dhp.aa5uzL7.cn/19359.Doc
dhp.aa5uzL7.cn/75331.Doc
dhp.aa5uzL7.cn/95953.Doc
dhp.aa5uzL7.cn/08082.Doc
dhp.aa5uzL7.cn/55173.Doc
dhp.aa5uzL7.cn/93771.Doc
dhp.aa5uzL7.cn/95599.Doc
dhp.aa5uzL7.cn/99111.Doc
dho.aa5uzL7.cn/55539.Doc
dho.aa5uzL7.cn/57733.Doc
dho.aa5uzL7.cn/93937.Doc
dho.aa5uzL7.cn/11951.Doc
dho.aa5uzL7.cn/93711.Doc
dho.aa5uzL7.cn/71795.Doc
dho.aa5uzL7.cn/37391.Doc
dho.aa5uzL7.cn/73115.Doc
dho.aa5uzL7.cn/59711.Doc
dho.aa5uzL7.cn/97357.Doc
dhi.aa5uzL7.cn/15975.Doc
dhi.aa5uzL7.cn/53917.Doc
dhi.aa5uzL7.cn/99537.Doc
dhi.aa5uzL7.cn/71957.Doc
dhi.aa5uzL7.cn/51359.Doc
dhi.aa5uzL7.cn/59537.Doc
dhi.aa5uzL7.cn/77517.Doc
dhi.aa5uzL7.cn/73175.Doc
dhi.aa5uzL7.cn/91311.Doc
dhi.aa5uzL7.cn/31771.Doc
dhu.aa5uzL7.cn/35759.Doc
dhu.aa5uzL7.cn/37533.Doc
dhu.aa5uzL7.cn/51799.Doc
dhu.aa5uzL7.cn/97919.Doc
dhu.aa5uzL7.cn/39915.Doc
dhu.aa5uzL7.cn/79519.Doc
dhu.aa5uzL7.cn/91597.Doc
dhu.aa5uzL7.cn/33395.Doc
dhu.aa5uzL7.cn/35199.Doc
dhu.aa5uzL7.cn/59577.Doc
dhy.aa5uzL7.cn/97111.Doc
dhy.aa5uzL7.cn/31513.Doc
dhy.aa5uzL7.cn/91771.Doc
dhy.aa5uzL7.cn/35715.Doc
dhy.aa5uzL7.cn/79731.Doc
dhy.aa5uzL7.cn/59791.Doc
dhy.aa5uzL7.cn/95731.Doc
dhy.aa5uzL7.cn/91517.Doc
dhy.aa5uzL7.cn/13157.Doc
dhy.aa5uzL7.cn/35711.Doc
dht.aa5uzL7.cn/73135.Doc
dht.aa5uzL7.cn/31753.Doc
dht.aa5uzL7.cn/17991.Doc
dht.aa5uzL7.cn/51335.Doc
dht.aa5uzL7.cn/71957.Doc
dht.aa5uzL7.cn/77975.Doc
dht.aa5uzL7.cn/93939.Doc
dht.aa5uzL7.cn/39395.Doc
dht.aa5uzL7.cn/95179.Doc
dht.aa5uzL7.cn/57771.Doc
dhr.aa5uzL7.cn/82688.Doc
dhr.aa5uzL7.cn/68002.Doc
dhr.aa5uzL7.cn/51573.Doc
dhr.aa5uzL7.cn/51131.Doc
dhr.aa5uzL7.cn/59777.Doc
dhr.aa5uzL7.cn/71355.Doc
dhr.aa5uzL7.cn/35313.Doc
dhr.aa5uzL7.cn/71593.Doc
dhr.aa5uzL7.cn/71539.Doc
dhr.aa5uzL7.cn/35757.Doc
dhe.aa5uzL7.cn/99991.Doc
dhe.aa5uzL7.cn/73777.Doc
dhe.aa5uzL7.cn/39779.Doc
dhe.aa5uzL7.cn/93133.Doc
dhe.aa5uzL7.cn/33979.Doc
dhe.aa5uzL7.cn/13955.Doc
dhe.aa5uzL7.cn/15751.Doc
dhe.aa5uzL7.cn/39337.Doc
dhe.aa5uzL7.cn/71917.Doc
dhe.aa5uzL7.cn/84866.Doc
dhw.aa5uzL7.cn/93171.Doc
dhw.aa5uzL7.cn/75535.Doc
dhw.aa5uzL7.cn/59997.Doc
dhw.aa5uzL7.cn/51559.Doc
dhw.aa5uzL7.cn/91973.Doc
dhw.aa5uzL7.cn/55337.Doc
dhw.aa5uzL7.cn/35733.Doc
dhw.aa5uzL7.cn/26462.Doc
dhw.aa5uzL7.cn/71313.Doc
dhw.aa5uzL7.cn/33139.Doc
dhq.aa5uzL7.cn/79111.Doc
dhq.aa5uzL7.cn/95935.Doc
dhq.aa5uzL7.cn/55595.Doc
dhq.aa5uzL7.cn/57571.Doc
dhq.aa5uzL7.cn/35531.Doc
dhq.aa5uzL7.cn/46066.Doc
dhq.aa5uzL7.cn/99753.Doc
dhq.aa5uzL7.cn/59517.Doc
dhq.aa5uzL7.cn/53711.Doc
dhq.aa5uzL7.cn/93579.Doc
dgm.aa5uzL7.cn/71579.Doc
dgm.aa5uzL7.cn/33513.Doc
dgm.aa5uzL7.cn/13395.Doc
dgm.aa5uzL7.cn/73351.Doc
dgm.aa5uzL7.cn/91199.Doc
dgm.aa5uzL7.cn/19399.Doc
dgm.aa5uzL7.cn/13935.Doc
dgm.aa5uzL7.cn/13131.Doc
dgm.aa5uzL7.cn/53579.Doc
dgm.aa5uzL7.cn/17975.Doc
dgn.aa5uzL7.cn/33191.Doc
dgn.aa5uzL7.cn/99537.Doc
dgn.aa5uzL7.cn/79719.Doc
dgn.aa5uzL7.cn/98304.Doc
dgn.aa5uzL7.cn/33599.Doc
dgn.aa5uzL7.cn/15511.Doc
dgn.aa5uzL7.cn/13517.Doc
dgn.aa5uzL7.cn/55951.Doc
dgn.aa5uzL7.cn/39733.Doc
dgn.aa5uzL7.cn/55357.Doc
dgb.aa5uzL7.cn/97157.Doc
dgb.aa5uzL7.cn/35579.Doc
dgb.aa5uzL7.cn/51119.Doc
dgb.aa5uzL7.cn/73973.Doc
dgb.aa5uzL7.cn/77753.Doc
dgb.aa5uzL7.cn/53553.Doc
dgb.aa5uzL7.cn/77593.Doc
dgb.aa5uzL7.cn/13513.Doc
dgb.aa5uzL7.cn/13573.Doc
dgb.aa5uzL7.cn/95915.Doc
dgv.aa5uzL7.cn/75157.Doc
dgv.aa5uzL7.cn/31711.Doc
dgv.aa5uzL7.cn/71957.Doc
dgv.aa5uzL7.cn/55979.Doc
dgv.aa5uzL7.cn/31359.Doc
dgv.aa5uzL7.cn/75353.Doc
dgv.aa5uzL7.cn/73353.Doc
dgv.aa5uzL7.cn/15759.Doc
dgv.aa5uzL7.cn/75979.Doc
dgv.aa5uzL7.cn/91193.Doc
dgc.aa5uzL7.cn/19735.Doc
dgc.aa5uzL7.cn/51339.Doc
dgc.aa5uzL7.cn/13191.Doc
dgc.aa5uzL7.cn/15975.Doc
dgc.aa5uzL7.cn/15315.Doc
dgc.aa5uzL7.cn/51195.Doc
dgc.aa5uzL7.cn/71955.Doc
dgc.aa5uzL7.cn/39593.Doc
dgc.aa5uzL7.cn/19595.Doc
dgc.aa5uzL7.cn/11773.Doc
dgx.aa5uzL7.cn/17311.Doc
dgx.aa5uzL7.cn/17597.Doc
dgx.aa5uzL7.cn/22224.Doc
dgx.aa5uzL7.cn/11713.Doc
dgx.aa5uzL7.cn/91533.Doc
dgx.aa5uzL7.cn/19957.Doc
dgx.aa5uzL7.cn/53753.Doc
dgx.aa5uzL7.cn/11171.Doc
dgx.aa5uzL7.cn/06420.Doc
dgx.aa5uzL7.cn/55931.Doc
dgz.aa5uzL7.cn/31375.Doc
dgz.aa5uzL7.cn/37915.Doc
dgz.aa5uzL7.cn/17715.Doc
dgz.aa5uzL7.cn/82880.Doc
dgz.aa5uzL7.cn/95753.Doc
dgz.aa5uzL7.cn/53319.Doc
dgz.aa5uzL7.cn/71557.Doc
dgz.aa5uzL7.cn/39731.Doc
dgz.aa5uzL7.cn/31315.Doc
dgz.aa5uzL7.cn/88226.Doc
dgl.aa5uzL7.cn/59397.Doc
dgl.aa5uzL7.cn/19137.Doc
dgl.aa5uzL7.cn/37771.Doc
dgl.aa5uzL7.cn/73195.Doc
dgl.aa5uzL7.cn/75773.Doc
dgl.aa5uzL7.cn/59595.Doc
dgl.aa5uzL7.cn/97319.Doc
dgl.aa5uzL7.cn/71537.Doc
dgl.aa5uzL7.cn/37195.Doc
dgl.aa5uzL7.cn/17777.Doc
dgk.aa5uzL7.cn/91377.Doc
dgk.aa5uzL7.cn/79155.Doc
dgk.aa5uzL7.cn/97977.Doc
dgk.aa5uzL7.cn/99993.Doc
dgk.aa5uzL7.cn/73973.Doc
dgk.aa5uzL7.cn/7.Doc
dgk.aa5uzL7.cn/53715.Doc
dgk.aa5uzL7.cn/31545.Doc
dgk.aa5uzL7.cn/39357.Doc
dgk.aa5uzL7.cn/51555.Doc
dgj.aa5uzL7.cn/17337.Doc
dgj.aa5uzL7.cn/75711.Doc
dgj.aa5uzL7.cn/99357.Doc
dgj.aa5uzL7.cn/57575.Doc
dgj.aa5uzL7.cn/17519.Doc
dgj.aa5uzL7.cn/39535.Doc
dgj.aa5uzL7.cn/55939.Doc
dgj.aa5uzL7.cn/95151.Doc
dgj.aa5uzL7.cn/59979.Doc
dgj.aa5uzL7.cn/99739.Doc
dgh.aa5uzL7.cn/15915.Doc
dgh.aa5uzL7.cn/13111.Doc
dgh.aa5uzL7.cn/35355.Doc
dgh.aa5uzL7.cn/35399.Doc
dgh.aa5uzL7.cn/99951.Doc
dgh.aa5uzL7.cn/17713.Doc
dgh.aa5uzL7.cn/77519.Doc
dgh.aa5uzL7.cn/77579.Doc
dgh.aa5uzL7.cn/77575.Doc
dgh.aa5uzL7.cn/33715.Doc
dgg.aa5uzL7.cn/33599.Doc
dgg.aa5uzL7.cn/55513.Doc
dgg.aa5uzL7.cn/82868.Doc
dgg.aa5uzL7.cn/73193.Doc
dgg.aa5uzL7.cn/37595.Doc
dgg.aa5uzL7.cn/75799.Doc
dgg.aa5uzL7.cn/71793.Doc
dgg.aa5uzL7.cn/99173.Doc
dgg.aa5uzL7.cn/08480.Doc
dgg.aa5uzL7.cn/75755.Doc
dgf.aa5uzL7.cn/39315.Doc
dgf.aa5uzL7.cn/57931.Doc
dgf.aa5uzL7.cn/33793.Doc
dgf.aa5uzL7.cn/75777.Doc
dgf.aa5uzL7.cn/77319.Doc
dgf.aa5uzL7.cn/95317.Doc
dgf.aa5uzL7.cn/11599.Doc
dgf.aa5uzL7.cn/91515.Doc
dgf.aa5uzL7.cn/53537.Doc
dgf.aa5uzL7.cn/55933.Doc
dgd.aa5uzL7.cn/57331.Doc
dgd.aa5uzL7.cn/39773.Doc
dgd.aa5uzL7.cn/99395.Doc
dgd.aa5uzL7.cn/77751.Doc
dgd.aa5uzL7.cn/77913.Doc
dgd.aa5uzL7.cn/00826.Doc
dgd.aa5uzL7.cn/60284.Doc
dgd.aa5uzL7.cn/77597.Doc
dgd.aa5uzL7.cn/53553.Doc
dgd.aa5uzL7.cn/71519.Doc
dgs.aa5uzL7.cn/17799.Doc
dgs.aa5uzL7.cn/73939.Doc
dgs.aa5uzL7.cn/68642.Doc
dgs.aa5uzL7.cn/39151.Doc
dgs.aa5uzL7.cn/19119.Doc
dgs.aa5uzL7.cn/37153.Doc
dgs.aa5uzL7.cn/57171.Doc
dgs.aa5uzL7.cn/93151.Doc
dgs.aa5uzL7.cn/91113.Doc
dgs.aa5uzL7.cn/75393.Doc
dga.aa5uzL7.cn/59937.Doc
dga.aa5uzL7.cn/93577.Doc
dga.aa5uzL7.cn/55917.Doc
dga.aa5uzL7.cn/91179.Doc
dga.aa5uzL7.cn/91157.Doc
dga.aa5uzL7.cn/97599.Doc
dga.aa5uzL7.cn/37313.Doc
dga.aa5uzL7.cn/15959.Doc
dga.aa5uzL7.cn/15197.Doc
dga.aa5uzL7.cn/51737.Doc
dgp.aa5uzL7.cn/53735.Doc
dgp.aa5uzL7.cn/73519.Doc
dgp.aa5uzL7.cn/33373.Doc
dgp.aa5uzL7.cn/53319.Doc
dgp.aa5uzL7.cn/06688.Doc
dgp.aa5uzL7.cn/26684.Doc
dgp.aa5uzL7.cn/51939.Doc
dgp.aa5uzL7.cn/19575.Doc
dgp.aa5uzL7.cn/91195.Doc
dgp.aa5uzL7.cn/51511.Doc
dgo.aa5uzL7.cn/59573.Doc
dgo.aa5uzL7.cn/59557.Doc
dgo.aa5uzL7.cn/19117.Doc
dgo.aa5uzL7.cn/77317.Doc
dgo.aa5uzL7.cn/17917.Doc
dgo.aa5uzL7.cn/51731.Doc
dgo.aa5uzL7.cn/75515.Doc
dgo.aa5uzL7.cn/95597.Doc
dgo.aa5uzL7.cn/68660.Doc
dgo.aa5uzL7.cn/95791.Doc
dgi.aa5uzL7.cn/24840.Doc
dgi.aa5uzL7.cn/08264.Doc
dgi.aa5uzL7.cn/77313.Doc
dgi.aa5uzL7.cn/59179.Doc
dgi.aa5uzL7.cn/55599.Doc
dgi.aa5uzL7.cn/51173.Doc
dgi.aa5uzL7.cn/19535.Doc
dgi.aa5uzL7.cn/93537.Doc
dgi.aa5uzL7.cn/51917.Doc
dgi.aa5uzL7.cn/51319.Doc
dgu.aa5uzL7.cn/33799.Doc
dgu.aa5uzL7.cn/06262.Doc
dgu.aa5uzL7.cn/97791.Doc
dgu.aa5uzL7.cn/13371.Doc
dgu.aa5uzL7.cn/35771.Doc
dgu.aa5uzL7.cn/99757.Doc
dgu.aa5uzL7.cn/91935.Doc
dgu.aa5uzL7.cn/31777.Doc
dgu.aa5uzL7.cn/31191.Doc
dgu.aa5uzL7.cn/79595.Doc
dgy.aa5uzL7.cn/73579.Doc
dgy.aa5uzL7.cn/91371.Doc
dgy.aa5uzL7.cn/44222.Doc
dgy.aa5uzL7.cn/13753.Doc
dgy.aa5uzL7.cn/37531.Doc
dgy.aa5uzL7.cn/39717.Doc
dgy.aa5uzL7.cn/39719.Doc
dgy.aa5uzL7.cn/37153.Doc
dgy.aa5uzL7.cn/35317.Doc
dgy.aa5uzL7.cn/26840.Doc
dgt.aa5uzL7.cn/71779.Doc
dgt.aa5uzL7.cn/95935.Doc
dgt.aa5uzL7.cn/37717.Doc
dgt.aa5uzL7.cn/35911.Doc
dgt.aa5uzL7.cn/11393.Doc
dgt.aa5uzL7.cn/11911.Doc
dgt.aa5uzL7.cn/44484.Doc
dgt.aa5uzL7.cn/55755.Doc
dgt.aa5uzL7.cn/75937.Doc
dgt.aa5uzL7.cn/31953.Doc
dgr.aa5uzL7.cn/15993.Doc
dgr.aa5uzL7.cn/33157.Doc
dgr.aa5uzL7.cn/08242.Doc
dgr.aa5uzL7.cn/33395.Doc
dgr.aa5uzL7.cn/77357.Doc
dgr.aa5uzL7.cn/11399.Doc
dgr.aa5uzL7.cn/99197.Doc
dgr.aa5uzL7.cn/95951.Doc
dgr.aa5uzL7.cn/57331.Doc
dgr.aa5uzL7.cn/35975.Doc
dge.aa5uzL7.cn/40022.Doc
dge.aa5uzL7.cn/17735.Doc
dge.aa5uzL7.cn/75711.Doc
dge.aa5uzL7.cn/57377.Doc
dge.aa5uzL7.cn/71919.Doc
dge.aa5uzL7.cn/37595.Doc
dge.aa5uzL7.cn/17551.Doc
dge.aa5uzL7.cn/20824.Doc
dge.aa5uzL7.cn/42008.Doc
dge.aa5uzL7.cn/86466.Doc
dgw.aa5uzL7.cn/31553.Doc
dgw.aa5uzL7.cn/15199.Doc
dgw.aa5uzL7.cn/57573.Doc
dgw.aa5uzL7.cn/31791.Doc
dgw.aa5uzL7.cn/77959.Doc
dgw.aa5uzL7.cn/31751.Doc
dgw.aa5uzL7.cn/15515.Doc
dgw.aa5uzL7.cn/53377.Doc
dgw.aa5uzL7.cn/11373.Doc
dgw.aa5uzL7.cn/71117.Doc
dgq.aa5uzL7.cn/53793.Doc
dgq.aa5uzL7.cn/71975.Doc
dgq.aa5uzL7.cn/24606.Doc
dgq.aa5uzL7.cn/15335.Doc
dgq.aa5uzL7.cn/93537.Doc
dgq.aa5uzL7.cn/13173.Doc
dgq.aa5uzL7.cn/51197.Doc
dgq.aa5uzL7.cn/51171.Doc
dgq.aa5uzL7.cn/55371.Doc
dgq.aa5uzL7.cn/37913.Doc
dfm.aa5uzL7.cn/35719.Doc
dfm.aa5uzL7.cn/79353.Doc
dfm.aa5uzL7.cn/31331.Doc
dfm.aa5uzL7.cn/75535.Doc
dfm.aa5uzL7.cn/17315.Doc
dfm.aa5uzL7.cn/17975.Doc
dfm.aa5uzL7.cn/39997.Doc
dfm.aa5uzL7.cn/39579.Doc
dfm.aa5uzL7.cn/19973.Doc
dfm.aa5uzL7.cn/75391.Doc
