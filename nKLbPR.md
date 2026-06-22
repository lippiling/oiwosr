院铰撬谠纪


【Python 异步 WebSocket 客户端】
==================================

websockets 库提供了完整的异步 WebSocket 客户端实现，
支持连接、收发消息、心跳检测和自动重连等特性。

──────────────────────────────────────────────
示例 1：基础 WebSocket 客户端
──────────────────────────────────────────────

import asyncio
import websockets  # 需要 pip install websockets

async def 基本客户端():
    """连接 WebSocket 服务器并收发消息"""
    uri = "ws://echo.websocket.org"  # 测试用 echo 服务

    async with websockets.connect(uri) as websocket:
        # 发送消息
        await websocket.send("Hello, WebSocket!")
        print("已发送: Hello, WebSocket!")

        # 接收消息
        响应 = await websocket.recv()
        print(f"收到响应: {响应}")

        # 发送多条消息
        for i in range(3):
            await websocket.send(f"消息-{i}")
            回显 = await websocket.recv()
            print(f"回显: {回显}")

# asyncio.run(基本客户端())

──────────────────────────────────────────────
示例 2：持续收发与 ping/pong
──────────────────────────────────────────────

import asyncio
import websockets

async def 持续通信():
    """持续收发消息并检测连接健康"""
    uri = "ws://echo.websocket.org"

    async with websockets.connect(uri) as ws:
        # 发送初始消息
        await ws.send("开始通信")

        async def 发送心跳():
            """定期发送 ping 保持连接"""
            while True:
                await asyncio.sleep(5)
                try:
                    pong_waiter = await ws.ping()
                    await asyncio.wait_for(pong_waiter, timeout=2)
                    print("心跳正常")
                except asyncio.TimeoutError:
                    print("心跳超时!")
                    break

        async def 接收消息():
            """持续接收消息"""
            try:
                async for 消息 in ws:
                    print(f"收到: {消息}")
            except websockets.ConnectionClosed:
                print("连接已关闭")

        # 并发执行心跳和接收
        await asyncio.gather(
            发送心跳(),
            接收消息(),
        )

# asyncio.run(持续通信())

──────────────────────────────────────────────
示例 3：自动重连机制
──────────────────────────────────────────────

import asyncio
import websockets

async def 自动重连客户端(uri):
    """带自动重连逻辑的 WebSocket 客户端"""
    重连延迟 = 1  # 初始重连延迟（秒）
    最大延迟 = 30
    最大重试 = 5

    for 尝试次数 in range(最大重试):
        try:
            print(f"连接尝试 {尝试次数 + 1}...")
            async with websockets.connect(uri) as ws:
                print("连接成功!")
                重连延迟 = 1  # 成功后重置延迟

                try:
                    async for 消息 in ws:
                        print(f"收到: {消息}")
                        # 处理业务消息...
                        await ws.send(f"确认: {消息}")
                except websockets.ConnectionClosed:
                    print("连接断开，准备重连...")

        except (ConnectionRefusedError, OSError) as e:
            print(f"连接失败: {e}")

        # 指数退避重连
        print(f"等待 {重连延迟} 秒后重试...")
        await asyncio.sleep(重连延迟)
        重连延迟 = min(重连延迟 * 2, 最大延迟)

    print("达到最大重试次数，放弃连接")

# asyncio.run(自动重连客户端("ws://localhost:8765"))

──────────────────────────────────────────────
示例 4：WebSocket 心跳与健康检测
──────────────────────────────────────────────

import asyncio
import websockets
import time

class 健康WebSocket客户端:
    """带完整心跳和健康检测的 WebSocket 客户端"""

    def __init__(self, uri, 心跳间隔=10):
        self.uri = uri
        self.心跳间隔 = 心跳间隔
        self.ws = None
        self.上次收到 = 0

    async def 连接(self):
        """建立连接并启动心跳"""
        self.ws = await websockets.connect(self.uri)
        self.上次收到 = time.time()
        asyncio.create_task(self._心跳循环())
        return self

    async def _心跳循环(self):
        """定期 ping 检测连接健康"""
        while True:
            await asyncio.sleep(self.心跳间隔)
            try:
                await self.ws.ping()
                # 检查上次收到消息的时间
                空闲时间 = time.time() - self.上次收到
                if 空闲时间 > self.心跳间隔 * 3:
                    print(f"连接空闲超时: {空闲时间:.1f}s")
                    break
                print(f"心跳正常 (空闲:{空闲时间:.1f}s)")
            except websockets.ConnectionClosed:
                print("心跳检测: 连接已关闭")
                break

    async def 发送(self, 消息):
        """发送消息"""
        if self.ws is None:
            raise RuntimeError("未连接")
        await self.ws.send(消息)

    async def 接收(self):
        """接收消息并更新时间戳"""
        消息 = await self.ws.recv()
        self.上次收到 = time.time()
        return 消息

    async def 关闭(self):
        """安全关闭连接"""
        if self.ws:
            await self.ws.close()

async def 健康客户端示例():
    client = 健康WebSocket客户端("ws://echo.websocket.org")
    await client.连接()
    await client.发送("测试健康检测")
    响应 = await client.接收()
    print(f"响应: {响应}")
    await client.关闭()

# asyncio.run(健康客户端示例())

──────────────────────────────────────────────
核心要点
──────────────────────────────────────────────
• websockets.connect - 异步连接 WebSocket 服务器
• ws.send(msg) - 发送文本/二进制消息
• ws.recv() - 接收单条消息
• async for msg in ws - 持续迭代接收消息
• ws.ping/pong - 心跳检测
• ConnectionClosed - 连接断开异常
• 自动重连 + 指数退避 - 可靠的客户端策略

颖痛坪谎焦忧牙移范野杜芈鼻阜钥

dkj.irvnp.cn/420840.Doc
dkj.irvnp.cn/536616.Doc
dkj.irvnp.cn/270882.Doc
dkj.irvnp.cn/624004.Doc
dkj.irvnp.cn/882086.Doc
dkj.irvnp.cn/024562.Doc
dkj.irvnp.cn/455919.Doc
dkj.irvnp.cn/072618.Doc
dkj.irvnp.cn/080402.Doc
dkj.irvnp.cn/640046.Doc
dkh.irvnp.cn/464006.Doc
dkh.irvnp.cn/207965.Doc
dkh.irvnp.cn/837139.Doc
dkh.irvnp.cn/404268.Doc
dkh.irvnp.cn/335391.Doc
dkh.irvnp.cn/205941.Doc
dkh.irvnp.cn/068826.Doc
dkh.irvnp.cn/860684.Doc
dkh.irvnp.cn/208020.Doc
dkh.irvnp.cn/995373.Doc
dkg.irvnp.cn/353779.Doc
dkg.irvnp.cn/179440.Doc
dkg.irvnp.cn/629930.Doc
dkg.irvnp.cn/462426.Doc
dkg.irvnp.cn/895622.Doc
dkg.irvnp.cn/397551.Doc
dkg.irvnp.cn/876027.Doc
dkg.irvnp.cn/648266.Doc
dkg.irvnp.cn/448828.Doc
dkg.irvnp.cn/912831.Doc
dkf.irvnp.cn/528316.Doc
dkf.irvnp.cn/573515.Doc
dkf.irvnp.cn/244448.Doc
dkf.irvnp.cn/684006.Doc
dkf.irvnp.cn/829106.Doc
dkf.irvnp.cn/404642.Doc
dkf.irvnp.cn/644864.Doc
dkf.irvnp.cn/575939.Doc
dkf.irvnp.cn/420640.Doc
dkf.irvnp.cn/280626.Doc
dkd.irvnp.cn/660006.Doc
dkd.irvnp.cn/991493.Doc
dkd.irvnp.cn/064244.Doc
dkd.irvnp.cn/248680.Doc
dkd.irvnp.cn/004268.Doc
dkd.irvnp.cn/991519.Doc
dkd.irvnp.cn/380977.Doc
dkd.irvnp.cn/042622.Doc
dkd.irvnp.cn/800402.Doc
dkd.irvnp.cn/888088.Doc
dks.irvnp.cn/806886.Doc
dks.irvnp.cn/208884.Doc
dks.irvnp.cn/712226.Doc
dks.irvnp.cn/028200.Doc
dks.irvnp.cn/466040.Doc
dks.irvnp.cn/807378.Doc
dks.irvnp.cn/391577.Doc
dks.irvnp.cn/400468.Doc
dks.irvnp.cn/482604.Doc
dks.irvnp.cn/286661.Doc
dka.irvnp.cn/226409.Doc
dka.irvnp.cn/268446.Doc
dka.irvnp.cn/791604.Doc
dka.irvnp.cn/888826.Doc
dka.irvnp.cn/140887.Doc
dka.irvnp.cn/404604.Doc
dka.irvnp.cn/429093.Doc
dka.irvnp.cn/339399.Doc
dka.irvnp.cn/842202.Doc
dka.irvnp.cn/824862.Doc
dkp.irvnp.cn/848826.Doc
dkp.irvnp.cn/074837.Doc
dkp.irvnp.cn/797351.Doc
dkp.irvnp.cn/551337.Doc
dkp.irvnp.cn/224466.Doc
dkp.irvnp.cn/886664.Doc
dkp.irvnp.cn/608400.Doc
dkp.irvnp.cn/208284.Doc
dkp.irvnp.cn/082424.Doc
dkp.irvnp.cn/974298.Doc
dko.irvnp.cn/956884.Doc
dko.irvnp.cn/440448.Doc
dko.irvnp.cn/286264.Doc
dko.irvnp.cn/084244.Doc
dko.irvnp.cn/444804.Doc
dko.irvnp.cn/088296.Doc
dko.irvnp.cn/483060.Doc
dko.irvnp.cn/139759.Doc
dko.irvnp.cn/686024.Doc
dko.irvnp.cn/280426.Doc
dki.irvnp.cn/689878.Doc
dki.irvnp.cn/682084.Doc
dki.irvnp.cn/408426.Doc
dki.irvnp.cn/464448.Doc
dki.irvnp.cn/680608.Doc
dki.irvnp.cn/868915.Doc
dki.irvnp.cn/428288.Doc
dki.irvnp.cn/600640.Doc
dki.irvnp.cn/600046.Doc
dki.irvnp.cn/535355.Doc
dku.fffbf.cn/406226.Doc
dku.fffbf.cn/648680.Doc
dku.fffbf.cn/577555.Doc
dku.fffbf.cn/537526.Doc
dku.fffbf.cn/860240.Doc
dku.fffbf.cn/620600.Doc
dku.fffbf.cn/422486.Doc
dku.fffbf.cn/533537.Doc
dku.fffbf.cn/004888.Doc
dku.fffbf.cn/887106.Doc
dky.fffbf.cn/179953.Doc
dky.fffbf.cn/597715.Doc
dky.fffbf.cn/087773.Doc
dky.fffbf.cn/886460.Doc
dky.fffbf.cn/841159.Doc
dky.fffbf.cn/624886.Doc
dky.fffbf.cn/200802.Doc
dky.fffbf.cn/448820.Doc
dky.fffbf.cn/428262.Doc
dky.fffbf.cn/220280.Doc
dkt.fffbf.cn/911226.Doc
dkt.fffbf.cn/200826.Doc
dkt.fffbf.cn/462604.Doc
dkt.fffbf.cn/511997.Doc
dkt.fffbf.cn/119739.Doc
dkt.fffbf.cn/199573.Doc
dkt.fffbf.cn/668465.Doc
dkt.fffbf.cn/866024.Doc
dkt.fffbf.cn/321532.Doc
dkt.fffbf.cn/977132.Doc
dkr.fffbf.cn/311468.Doc
dkr.fffbf.cn/622246.Doc
dkr.fffbf.cn/235505.Doc
dkr.fffbf.cn/882064.Doc
dkr.fffbf.cn/535777.Doc
dkr.fffbf.cn/282246.Doc
dkr.fffbf.cn/402066.Doc
dkr.fffbf.cn/088844.Doc
dkr.fffbf.cn/242404.Doc
dkr.fffbf.cn/402840.Doc
dke.fffbf.cn/864924.Doc
dke.fffbf.cn/408282.Doc
dke.fffbf.cn/820068.Doc
dke.fffbf.cn/727660.Doc
dke.fffbf.cn/088488.Doc
dke.fffbf.cn/101494.Doc
dke.fffbf.cn/404442.Doc
dke.fffbf.cn/088242.Doc
dke.fffbf.cn/442666.Doc
dke.fffbf.cn/486020.Doc
dkw.fffbf.cn/422688.Doc
dkw.fffbf.cn/010899.Doc
dkw.fffbf.cn/929433.Doc
dkw.fffbf.cn/420082.Doc
dkw.fffbf.cn/330374.Doc
dkw.fffbf.cn/246062.Doc
dkw.fffbf.cn/222664.Doc
dkw.fffbf.cn/602240.Doc
dkw.fffbf.cn/922050.Doc
dkw.fffbf.cn/297946.Doc
dkq.fffbf.cn/179719.Doc
dkq.fffbf.cn/864628.Doc
dkq.fffbf.cn/917115.Doc
dkq.fffbf.cn/246840.Doc
dkq.fffbf.cn/402040.Doc
dkq.fffbf.cn/991133.Doc
dkq.fffbf.cn/280280.Doc
dkq.fffbf.cn/420404.Doc
dkq.fffbf.cn/666840.Doc
dkq.fffbf.cn/426220.Doc
djm.fffbf.cn/466404.Doc
djm.fffbf.cn/684020.Doc
djm.fffbf.cn/797553.Doc
djm.fffbf.cn/468484.Doc
djm.fffbf.cn/842682.Doc
djm.fffbf.cn/662666.Doc
djm.fffbf.cn/220446.Doc
djm.fffbf.cn/672123.Doc
djm.fffbf.cn/068886.Doc
djm.fffbf.cn/681666.Doc
djn.fffbf.cn/466888.Doc
djn.fffbf.cn/575979.Doc
djn.fffbf.cn/264060.Doc
djn.fffbf.cn/668428.Doc
djn.fffbf.cn/590274.Doc
djn.fffbf.cn/320796.Doc
djn.fffbf.cn/559311.Doc
djn.fffbf.cn/426828.Doc
djn.fffbf.cn/561161.Doc
djn.fffbf.cn/913339.Doc
djb.fffbf.cn/359955.Doc
djb.fffbf.cn/264282.Doc
djb.fffbf.cn/305038.Doc
djb.fffbf.cn/226711.Doc
djb.fffbf.cn/177751.Doc
djb.fffbf.cn/988325.Doc
djb.fffbf.cn/224606.Doc
djb.fffbf.cn/957977.Doc
djb.fffbf.cn/690704.Doc
djb.fffbf.cn/448646.Doc
djv.fffbf.cn/727820.Doc
djv.fffbf.cn/402800.Doc
djv.fffbf.cn/620266.Doc
djv.fffbf.cn/066804.Doc
djv.fffbf.cn/800244.Doc
djv.fffbf.cn/171337.Doc
djv.fffbf.cn/318515.Doc
djv.fffbf.cn/888460.Doc
djv.fffbf.cn/402482.Doc
djv.fffbf.cn/535737.Doc
djc.fffbf.cn/442620.Doc
djc.fffbf.cn/802626.Doc
djc.fffbf.cn/862028.Doc
djc.fffbf.cn/444840.Doc
djc.fffbf.cn/880066.Doc
djc.fffbf.cn/379397.Doc
djc.fffbf.cn/024082.Doc
djc.fffbf.cn/842688.Doc
djc.fffbf.cn/680621.Doc
djc.fffbf.cn/280464.Doc
djx.fffbf.cn/591191.Doc
djx.fffbf.cn/428860.Doc
djx.fffbf.cn/640448.Doc
djx.fffbf.cn/442206.Doc
djx.fffbf.cn/028044.Doc
djx.fffbf.cn/026862.Doc
djx.fffbf.cn/828624.Doc
djx.fffbf.cn/068208.Doc
djx.fffbf.cn/004002.Doc
djx.fffbf.cn/042206.Doc
djz.fffbf.cn/460866.Doc
djz.fffbf.cn/200886.Doc
djz.fffbf.cn/086620.Doc
djz.fffbf.cn/020048.Doc
djz.fffbf.cn/117511.Doc
djz.fffbf.cn/042048.Doc
djz.fffbf.cn/220624.Doc
djz.fffbf.cn/000662.Doc
djz.fffbf.cn/371139.Doc
djz.fffbf.cn/884240.Doc
djl.fffbf.cn/844284.Doc
djl.fffbf.cn/827822.Doc
djl.fffbf.cn/848886.Doc
djl.fffbf.cn/800208.Doc
djl.fffbf.cn/426866.Doc
djl.fffbf.cn/666084.Doc
djl.fffbf.cn/888886.Doc
djl.fffbf.cn/466400.Doc
djl.fffbf.cn/206462.Doc
djl.fffbf.cn/757335.Doc
djk.fffbf.cn/086080.Doc
djk.fffbf.cn/066642.Doc
djk.fffbf.cn/286282.Doc
djk.fffbf.cn/397593.Doc
djk.fffbf.cn/604862.Doc
djk.fffbf.cn/404882.Doc
djk.fffbf.cn/808024.Doc
djk.fffbf.cn/975715.Doc
djk.fffbf.cn/468242.Doc
djk.fffbf.cn/060804.Doc
djj.fffbf.cn/248402.Doc
djj.fffbf.cn/957135.Doc
djj.fffbf.cn/662602.Doc
djj.fffbf.cn/315531.Doc
djj.fffbf.cn/717977.Doc
djj.fffbf.cn/680428.Doc
djj.fffbf.cn/686060.Doc
djj.fffbf.cn/773971.Doc
djj.fffbf.cn/933571.Doc
djj.fffbf.cn/591193.Doc
djh.fffbf.cn/088088.Doc
djh.fffbf.cn/248088.Doc
djh.fffbf.cn/242082.Doc
djh.fffbf.cn/113515.Doc
djh.fffbf.cn/248882.Doc
djh.fffbf.cn/595195.Doc
djh.fffbf.cn/044064.Doc
djh.fffbf.cn/866460.Doc
djh.fffbf.cn/008446.Doc
djh.fffbf.cn/062408.Doc
djg.fffbf.cn/802448.Doc
djg.fffbf.cn/064082.Doc
djg.fffbf.cn/622024.Doc
djg.fffbf.cn/004602.Doc
djg.fffbf.cn/826488.Doc
djg.fffbf.cn/048208.Doc
djg.fffbf.cn/662264.Doc
djg.fffbf.cn/719995.Doc
djg.fffbf.cn/240864.Doc
djg.fffbf.cn/444660.Doc
djf.fffbf.cn/468820.Doc
djf.fffbf.cn/931177.Doc
djf.fffbf.cn/686440.Doc
djf.fffbf.cn/537175.Doc
djf.fffbf.cn/840800.Doc
djf.fffbf.cn/791139.Doc
djf.fffbf.cn/115711.Doc
djf.fffbf.cn/076015.Doc
djf.fffbf.cn/710562.Doc
djf.fffbf.cn/077554.Doc
djd.fffbf.cn/262400.Doc
djd.fffbf.cn/000062.Doc
djd.fffbf.cn/951715.Doc
djd.fffbf.cn/464228.Doc
djd.fffbf.cn/066408.Doc
djd.fffbf.cn/377355.Doc
djd.fffbf.cn/775111.Doc
djd.fffbf.cn/732134.Doc
djd.fffbf.cn/604664.Doc
djd.fffbf.cn/476949.Doc
djs.fffbf.cn/808024.Doc
djs.fffbf.cn/668082.Doc
djs.fffbf.cn/046624.Doc
djs.fffbf.cn/203304.Doc
djs.fffbf.cn/644260.Doc
djs.fffbf.cn/286428.Doc
djs.fffbf.cn/171597.Doc
djs.fffbf.cn/395391.Doc
djs.fffbf.cn/309589.Doc
djs.fffbf.cn/229835.Doc
dja.fffbf.cn/977137.Doc
dja.fffbf.cn/755391.Doc
dja.fffbf.cn/662006.Doc
dja.fffbf.cn/387078.Doc
dja.fffbf.cn/820284.Doc
dja.fffbf.cn/444824.Doc
dja.fffbf.cn/864262.Doc
dja.fffbf.cn/517373.Doc
dja.fffbf.cn/480846.Doc
dja.fffbf.cn/686860.Doc
djp.fffbf.cn/600622.Doc
djp.fffbf.cn/496889.Doc
djp.fffbf.cn/937935.Doc
djp.fffbf.cn/147241.Doc
djp.fffbf.cn/428048.Doc
djp.fffbf.cn/404646.Doc
djp.fffbf.cn/726762.Doc
djp.fffbf.cn/286848.Doc
djp.fffbf.cn/042084.Doc
djp.fffbf.cn/024860.Doc
djo.fffbf.cn/265488.Doc
djo.fffbf.cn/113533.Doc
djo.fffbf.cn/133713.Doc
djo.fffbf.cn/555737.Doc
djo.fffbf.cn/244604.Doc
djo.fffbf.cn/406422.Doc
djo.fffbf.cn/062026.Doc
djo.fffbf.cn/527968.Doc
djo.fffbf.cn/835413.Doc
djo.fffbf.cn/068028.Doc
dji.fffbf.cn/905260.Doc
dji.fffbf.cn/717919.Doc
dji.fffbf.cn/400284.Doc
dji.fffbf.cn/462620.Doc
dji.fffbf.cn/404064.Doc
dji.fffbf.cn/573077.Doc
dji.fffbf.cn/713997.Doc
dji.fffbf.cn/844424.Doc
dji.fffbf.cn/226462.Doc
dji.fffbf.cn/937589.Doc
dju.fffbf.cn/551151.Doc
dju.fffbf.cn/880868.Doc
dju.fffbf.cn/824460.Doc
dju.fffbf.cn/119991.Doc
dju.fffbf.cn/735919.Doc
dju.fffbf.cn/808440.Doc
dju.fffbf.cn/945880.Doc
dju.fffbf.cn/682482.Doc
dju.fffbf.cn/155111.Doc
dju.fffbf.cn/886268.Doc
djy.fffbf.cn/280444.Doc
djy.fffbf.cn/042806.Doc
djy.fffbf.cn/626802.Doc
djy.fffbf.cn/668440.Doc
djy.fffbf.cn/606284.Doc
djy.fffbf.cn/444644.Doc
djy.fffbf.cn/820426.Doc
djy.fffbf.cn/906560.Doc
djy.fffbf.cn/084680.Doc
djy.fffbf.cn/397317.Doc
djt.fffbf.cn/882662.Doc
djt.fffbf.cn/371973.Doc
djt.fffbf.cn/884246.Doc
djt.fffbf.cn/846468.Doc
djt.fffbf.cn/602440.Doc
djt.fffbf.cn/822264.Doc
djt.fffbf.cn/971445.Doc
djt.fffbf.cn/202462.Doc
djt.fffbf.cn/002606.Doc
djt.fffbf.cn/130057.Doc
djr.fffbf.cn/240660.Doc
djr.fffbf.cn/068208.Doc
djr.fffbf.cn/554463.Doc
djr.fffbf.cn/444426.Doc
djr.fffbf.cn/357999.Doc
