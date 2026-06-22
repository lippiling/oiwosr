诺亿按口恼


=================================================================
 Python asyncio 协议与传输：低层网络编程详解
=================================================================

asyncio 提供了两层 API：高层流 API（Streams）和低层协议/传输 API
（Protocol/Transport）。本文深入讲解低层协议传输模型。

=================================================================
 一、Protocol 基础：事件驱动的网络处理
=================================================================

import asyncio
import logging

logging.basicConfig(level=logging.INFO, format="%(message)s")
logger = logging.getLogger(__name__)

class EchoProtocol(asyncio.Protocol):
    """
    回显协议：将收到的数据原样返回。
    Protocol 是事件驱动的，需要重写特定方法。
    """

    def __init__(self):
        self.transport = None  # 传输对象，稍后绑定
        self.buffer = b""      # 内部缓冲区

    def connection_made(self, transport: asyncio.Transport):
        """
        当连接建立时调用。
        transport 参数是对端连接的传输层对象。
        """
        self.transport = transport
        peername = transport.get_extra_info("peername")
        logger.info(f"连接建立：{peername}")
        # 获取 socket 相关信息
        sockname = transport.get_extra_info("sockname")
        logger.info(f"本地地址：{sockname}")

    def data_received(self, data: bytes):
        """
        当收到数据时调用。这是最核心的方法。
        data 是从对端接收到的原始字节。
        """
        logger.info(f"收到数据：{data!r}")
        # 使用 transport.write() 发送数据（非阻塞）
        self.transport.write(data)
        logger.info(f"已回显 {len(data)} 字节")

    def eof_received(self) -> bool:
        """
        当对端发送 EOF（结束标记）时调用。
        返回 True 表示仍然想保持传输方向打开。
        返回 False（默认）表示关闭传输方向。
        """
        logger.info("收到 EOF")
        # 如果需要半关闭连接，返回 True
        return False

    def connection_lost(self, exc: Exception | None):
        """
        当连接关闭或发生异常时调用。
        exc 为 None 表示正常关闭，否则为异常对象。
        """
        if exc:
            logger.error(f"连接异常关闭：{exc}")
        else:
            logger.info("连接正常关闭")
        self.transport = None

=================================================================
 二、Transport：数据传输的底层接口
=================================================================

class TransportDemo:
    """展示 Transport 的核心方法"""

    def __init__(self, protocol: asyncio.Protocol):
        self.protocol = protocol
        self.transport = protocol.transport

    def send_data(self, data: bytes):
        """非阻塞写入数据到传输层"""
        self.transport.write(data)
        # write 不保证立即发送，数据可能被缓冲

    def send_eof(self):
        """发送 EOF 标记，表示写半关闭"""
        self.transport.write_eof()

    def abort_connection(self):
        """立即终止连接，丢弃缓冲数据"""
        self.transport.abort()

    def check_status(self):
        """检查传输状态"""
        if self.transport.is_closing():
            logger.warning("传输层正在关闭")
        # 获取缓冲区中待发送的数据量
        size = self.transport.get_write_buffer_size()
        logger.info(f"写缓冲区大小：{size}")

# Transport 关键方法总结：
#   write(data)      - 非阻塞写入
#   writelines(list) - 写入多个数据片段
#   write_eof()      - 关闭写半连接
#   close()          - 优雅关闭
#   abort()          - 立即终止
#   can_write_eof()  - 是否支持 EOF
#   get_write_buffer_size() - 获取缓冲区大小
#   get_write_buffer_limits() - 获取缓冲区高低水位线
#   set_write_buffer_limits() - 设置缓冲区高低水位线
#   pause_reading()  - 暂停读取
#   resume_reading() - 恢复读取

=================================================================
 三、使用协议工厂创建 TCP 服务器
=================================================================

class EchoServer:
    """完整的回显 TCP 服务器"""

    def __init__(self, host: str = "127.0.0.1", port: int = 8888):
        self.host = host
        self.port = port

    async def start(self):
        """
        使用 asyncio.create_server 创建服务器。
        protocol_factory 是一个可调用对象，每次新连接都会调用它。
        """
        loop = asyncio.get_running_loop()
        # 创建服务器：传入协议工厂而非协程
        server = await loop.create_server(
            lambda: EchoProtocol(),  # 每次连接创建新的协议实例
            self.host,
            self.port,
            reuse_address=True,
            reuse_port=False,
            backlog=128,  # 连接队列大小
        )

        addr = server.sockets[0].getsockname()
        logger.info(f"服务器运行在 {addr}")

        # server.serve_forever() 启动服务
        # 内部调用 loop.run_forever() 的行为
        async with server:
            await server.serve_forever()

    async def stop(self, server):
        """优雅关闭服务器"""
        server.close()
        # 等待所有连接处理完毕
        await server.wait_closed()
        logger.info("服务器已关闭")

async def run_echo_client():
    """测试用的回显客户端"""
    reader, writer = await asyncio.open_connection("127.0.0.1", 8888)
    writer.write(b"Hello, Protocol!\n")
    await writer.drain()

    data = await reader.read(100)
    logger.info(f"客户端收到：{data!r}")

    writer.close()
    await writer.wait_closed()

=================================================================
 四、低层协议 vs 高层流 API 对比
=================================================================

"""
=== 低层协议 API ===
优点：
  - 更精细的控制（缓冲区、EOF、半关闭）
  - 适合自定义协议实现
  - 性能略高（减少一层抽象）
  - 适合高吞吐场景

缺点：
  - 代码更复杂
  - 需要手动管理协议状态
  - 不适合简单的请求-响应模式

=== 高层流 API ===
优点：
  - 代码简洁，类似同步编程
  - 自动处理缓冲和分割
  - 适合大多数应用

缺点：
  - 灵活性较低
  - 无法直接处理半关闭场景
  - 额外的抽象层带来轻微开销

=== 何时使用协议 ===
1. 实现自定义二进制协议（如 HTTP/2, WebSocket 框架）
2. 需要精细控制 TCP 连接生命周期
3. 需要处理部分数据包（协议分帧）
4. 需要直接操作文件描述符的事件
5. 低延迟高吞吐场景（如游戏服务器）
"""

=================================================================
 五、自定义行协议示例
=================================================================

class LineProtocol(asyncio.Protocol):
    """
    行协议：每次接收一行数据并处理。
    演示如何处理粘包（数据分片）。
    """

    def __init__(self):
        self.buf = b""

    def data_received(self, data: bytes):
        """处理粘包：将数据追加到缓冲区后逐行处理"""
        self.buf += data
        while b"\n" in self.buf:
            line, self.buf = self.buf.split(b"\n", 1)
            self.handle_line(line)

    def handle_line(self, line: bytes):
        """处理完整的一行数据"""
        if not line.strip():
            return
        logger.info(f"处理行：{line.decode(errors='replace')}")
        # 在这里添加业务逻辑

    def connection_lost(self, exc):
        """连接关闭时处理缓冲区中的残留数据"""
        if self.buf:
            logger.warning(f"连接关闭，有未处理数据：{self.buf!r}")
        super().connection_lost(exc)

=================================================================
 六、总结
=================================================================

# 1. Protocol 是事件驱动的接口，核心方法包括：
#    connection_made / data_received / eof_received / connection_lost
# 2. Transport 负责实际的数据发送和连接管理
# 3. create_server 使用协议工厂模式创建 TCP 服务
# 4. 低层协议提供了最高灵活性，适合复杂协议实现
# 5. 简单场景优先选择高层 Streams API

if __name__ == "__main__":
    async def main():
        server = EchoServer()
        # 生产环境应使用 asyncio.run(main())
        await server.start()
    asyncio.run(main())

毡反狼科荣创故颇狼褪仆艘腋严毡

dqn.vwbnt.cn/972805.Doc
dqn.vwbnt.cn/241419.Doc
dqn.vwbnt.cn/025515.Doc
dqb.vwbnt.cn/722548.Doc
dqb.vwbnt.cn/678470.Doc
dqb.vwbnt.cn/235235.Doc
dqb.vwbnt.cn/193563.Doc
dqb.vwbnt.cn/226129.Doc
dqb.vwbnt.cn/409447.Doc
dqb.vwbnt.cn/298325.Doc
dqb.vwbnt.cn/012046.Doc
dqb.vwbnt.cn/521469.Doc
dqb.vwbnt.cn/348428.Doc
dqv.vwbnt.cn/821650.Doc
dqv.vwbnt.cn/335208.Doc
dqv.vwbnt.cn/909917.Doc
dqv.vwbnt.cn/914234.Doc
dqv.vwbnt.cn/017493.Doc
dqv.vwbnt.cn/483695.Doc
dqv.vwbnt.cn/520145.Doc
dqv.vwbnt.cn/243640.Doc
dqv.vwbnt.cn/365824.Doc
dqv.vwbnt.cn/058803.Doc
dqc.vwbnt.cn/304583.Doc
dqc.vwbnt.cn/433188.Doc
dqc.vwbnt.cn/020643.Doc
dqc.vwbnt.cn/656998.Doc
dqc.vwbnt.cn/614983.Doc
dqc.vwbnt.cn/480861.Doc
dqc.vwbnt.cn/611990.Doc
dqc.vwbnt.cn/645764.Doc
dqc.vwbnt.cn/631440.Doc
dqc.vwbnt.cn/540256.Doc
dqx.vwbnt.cn/836137.Doc
dqx.vwbnt.cn/535539.Doc
dqx.vwbnt.cn/916268.Doc
dqx.vwbnt.cn/007782.Doc
dqx.vwbnt.cn/818426.Doc
dqx.vwbnt.cn/376475.Doc
dqx.vwbnt.cn/452875.Doc
dqx.vwbnt.cn/437713.Doc
dqx.vwbnt.cn/085966.Doc
dqx.vwbnt.cn/335321.Doc
dqz.vwbnt.cn/503147.Doc
dqz.vwbnt.cn/959568.Doc
dqz.vwbnt.cn/188540.Doc
dqz.vwbnt.cn/750007.Doc
dqz.vwbnt.cn/907308.Doc
dqz.vwbnt.cn/945876.Doc
dqz.vwbnt.cn/394108.Doc
dqz.vwbnt.cn/367124.Doc
dqz.vwbnt.cn/024431.Doc
dqz.vwbnt.cn/353233.Doc
dql.vwbnt.cn/936707.Doc
dql.vwbnt.cn/238324.Doc
dql.vwbnt.cn/809772.Doc
dql.vwbnt.cn/799705.Doc
dql.vwbnt.cn/897800.Doc
dql.vwbnt.cn/365624.Doc
dql.vwbnt.cn/094613.Doc
dql.vwbnt.cn/777823.Doc
dql.vwbnt.cn/521631.Doc
dql.vwbnt.cn/004664.Doc
dqk.vwbnt.cn/974164.Doc
dqk.vwbnt.cn/824844.Doc
dqk.vwbnt.cn/962393.Doc
dqk.vwbnt.cn/705023.Doc
dqk.vwbnt.cn/277758.Doc
dqk.vwbnt.cn/269205.Doc
dqk.vwbnt.cn/420624.Doc
dqk.vwbnt.cn/442446.Doc
dqk.vwbnt.cn/446868.Doc
dqk.vwbnt.cn/044806.Doc
dqj.vwbnt.cn/602428.Doc
dqj.vwbnt.cn/228200.Doc
dqj.vwbnt.cn/284626.Doc
dqj.vwbnt.cn/482448.Doc
dqj.vwbnt.cn/602844.Doc
dqj.vwbnt.cn/624020.Doc
dqj.vwbnt.cn/288688.Doc
dqj.vwbnt.cn/088884.Doc
dqj.vwbnt.cn/733553.Doc
dqj.vwbnt.cn/066826.Doc
dqh.vwbnt.cn/604282.Doc
dqh.vwbnt.cn/624862.Doc
dqh.vwbnt.cn/228828.Doc
dqh.vwbnt.cn/666206.Doc
dqh.vwbnt.cn/424446.Doc
dqh.vwbnt.cn/248642.Doc
dqh.vwbnt.cn/606620.Doc
dqh.vwbnt.cn/246604.Doc
dqh.vwbnt.cn/040466.Doc
dqh.vwbnt.cn/200684.Doc
dqg.vwbnt.cn/820208.Doc
dqg.vwbnt.cn/246406.Doc
dqg.vwbnt.cn/606686.Doc
dqg.vwbnt.cn/882084.Doc
dqg.vwbnt.cn/626822.Doc
dqg.vwbnt.cn/022042.Doc
dqg.vwbnt.cn/882624.Doc
dqg.vwbnt.cn/068444.Doc
dqg.vwbnt.cn/886420.Doc
dqg.vwbnt.cn/551177.Doc
dqf.vwbnt.cn/800802.Doc
dqf.vwbnt.cn/755717.Doc
dqf.vwbnt.cn/648062.Doc
dqf.vwbnt.cn/620866.Doc
dqf.vwbnt.cn/646268.Doc
dqf.vwbnt.cn/280026.Doc
dqf.vwbnt.cn/208608.Doc
dqf.vwbnt.cn/220284.Doc
dqf.vwbnt.cn/084000.Doc
dqf.vwbnt.cn/062882.Doc
dqd.vwbnt.cn/828804.Doc
dqd.vwbnt.cn/886446.Doc
dqd.vwbnt.cn/888808.Doc
dqd.vwbnt.cn/040628.Doc
dqd.vwbnt.cn/806686.Doc
dqd.vwbnt.cn/028400.Doc
dqd.vwbnt.cn/486442.Doc
dqd.vwbnt.cn/97.Doc
dqd.vwbnt.cn/886040.Doc
dqd.vwbnt.cn/955159.Doc
dqs.vwbnt.cn/393333.Doc
dqs.vwbnt.cn/488662.Doc
dqs.vwbnt.cn/080084.Doc
dqs.vwbnt.cn/004682.Doc
dqs.vwbnt.cn/866462.Doc
dqs.vwbnt.cn/488848.Doc
dqs.vwbnt.cn/040600.Doc
dqs.vwbnt.cn/208260.Doc
dqs.vwbnt.cn/024248.Doc
dqs.vwbnt.cn/884688.Doc
dqa.vwbnt.cn/082864.Doc
dqa.vwbnt.cn/420442.Doc
dqa.vwbnt.cn/044888.Doc
dqa.vwbnt.cn/464800.Doc
dqa.vwbnt.cn/408642.Doc
dqa.vwbnt.cn/682644.Doc
dqa.vwbnt.cn/486048.Doc
dqa.vwbnt.cn/600244.Doc
dqa.vwbnt.cn/602480.Doc
dqa.vwbnt.cn/468660.Doc
dqp.vwbnt.cn/200664.Doc
dqp.vwbnt.cn/606280.Doc
dqp.vwbnt.cn/002282.Doc
dqp.vwbnt.cn/226662.Doc
dqp.vwbnt.cn/046804.Doc
dqp.vwbnt.cn/688648.Doc
dqp.vwbnt.cn/068604.Doc
dqp.vwbnt.cn/175937.Doc
dqp.vwbnt.cn/228644.Doc
dqp.vwbnt.cn/684644.Doc
dqo.vwbnt.cn/468044.Doc
dqo.vwbnt.cn/842648.Doc
dqo.vwbnt.cn/606626.Doc
dqo.vwbnt.cn/084004.Doc
dqo.vwbnt.cn/688664.Doc
dqo.vwbnt.cn/048044.Doc
dqo.vwbnt.cn/642884.Doc
dqo.vwbnt.cn/848264.Doc
dqo.vwbnt.cn/848884.Doc
dqo.vwbnt.cn/442042.Doc
dqi.vwbnt.cn/228880.Doc
dqi.vwbnt.cn/244684.Doc
dqi.vwbnt.cn/686222.Doc
dqi.vwbnt.cn/791919.Doc
dqi.vwbnt.cn/880626.Doc
dqi.vwbnt.cn/020282.Doc
dqi.vwbnt.cn/842026.Doc
dqi.vwbnt.cn/666860.Doc
dqi.vwbnt.cn/866828.Doc
dqi.vwbnt.cn/648408.Doc
dqu.vwbnt.cn/466024.Doc
dqu.vwbnt.cn/688224.Doc
dqu.vwbnt.cn/000004.Doc
dqu.vwbnt.cn/248200.Doc
dqu.vwbnt.cn/826806.Doc
dqu.vwbnt.cn/244826.Doc
dqu.vwbnt.cn/484824.Doc
dqu.vwbnt.cn/608004.Doc
dqu.vwbnt.cn/820244.Doc
dqu.vwbnt.cn/644268.Doc
dqy.vwbnt.cn/088222.Doc
dqy.vwbnt.cn/242408.Doc
dqy.vwbnt.cn/420604.Doc
dqy.vwbnt.cn/420226.Doc
dqy.vwbnt.cn/826804.Doc
dqy.vwbnt.cn/000002.Doc
dqy.vwbnt.cn/688626.Doc
dqy.vwbnt.cn/266682.Doc
dqy.vwbnt.cn/628480.Doc
dqy.vwbnt.cn/682408.Doc
dqt.vwbnt.cn/044600.Doc
dqt.vwbnt.cn/846480.Doc
dqt.vwbnt.cn/062060.Doc
dqt.vwbnt.cn/937557.Doc
dqt.vwbnt.cn/040042.Doc
dqt.vwbnt.cn/286682.Doc
dqt.vwbnt.cn/800440.Doc
dqt.vwbnt.cn/844442.Doc
dqt.vwbnt.cn/042082.Doc
dqt.vwbnt.cn/486066.Doc
dqr.vwbnt.cn/862062.Doc
dqr.vwbnt.cn/226846.Doc
dqr.vwbnt.cn/026608.Doc
dqr.vwbnt.cn/422428.Doc
dqr.vwbnt.cn/644228.Doc
dqr.vwbnt.cn/266866.Doc
dqr.vwbnt.cn/046204.Doc
dqr.vwbnt.cn/664242.Doc
dqr.vwbnt.cn/822088.Doc
dqr.vwbnt.cn/800422.Doc
dqe.vwbnt.cn/046060.Doc
dqe.vwbnt.cn/379933.Doc
dqe.vwbnt.cn/206426.Doc
dqe.vwbnt.cn/826662.Doc
dqe.vwbnt.cn/226228.Doc
dqe.vwbnt.cn/648468.Doc
dqe.vwbnt.cn/248080.Doc
dqe.vwbnt.cn/648408.Doc
dqe.vwbnt.cn/422428.Doc
dqe.vwbnt.cn/440268.Doc
dqw.vwbnt.cn/600660.Doc
dqw.vwbnt.cn/062268.Doc
dqw.vwbnt.cn/620648.Doc
dqw.vwbnt.cn/840086.Doc
dqw.vwbnt.cn/442006.Doc
dqw.vwbnt.cn/688284.Doc
dqw.vwbnt.cn/260666.Doc
dqw.vwbnt.cn/048448.Doc
dqw.vwbnt.cn/866288.Doc
dqw.vwbnt.cn/022400.Doc
dqq.vwbnt.cn/915959.Doc
dqq.vwbnt.cn/244846.Doc
dqq.vwbnt.cn/228402.Doc
dqq.vwbnt.cn/248848.Doc
dqq.vwbnt.cn/464060.Doc
dqq.vwbnt.cn/020020.Doc
dqq.vwbnt.cn/000288.Doc
dqq.vwbnt.cn/866222.Doc
dqq.vwbnt.cn/559379.Doc
dqq.vwbnt.cn/642822.Doc
smm.vwbnt.cn/806604.Doc
smm.vwbnt.cn/860246.Doc
smm.vwbnt.cn/333739.Doc
smm.vwbnt.cn/604286.Doc
smm.vwbnt.cn/268080.Doc
smm.vwbnt.cn/046220.Doc
smm.vwbnt.cn/244264.Doc
smm.vwbnt.cn/000808.Doc
smm.vwbnt.cn/408600.Doc
smm.vwbnt.cn/066826.Doc
smn.vwbnt.cn/086600.Doc
smn.vwbnt.cn/680666.Doc
smn.vwbnt.cn/268666.Doc
smn.vwbnt.cn/086402.Doc
smn.vwbnt.cn/448842.Doc
smn.vwbnt.cn/800202.Doc
smn.vwbnt.cn/408682.Doc
smn.vwbnt.cn/400824.Doc
smn.vwbnt.cn/266400.Doc
smn.vwbnt.cn/220822.Doc
smb.vwbnt.cn/173977.Doc
smb.vwbnt.cn/002022.Doc
smb.vwbnt.cn/224446.Doc
smb.vwbnt.cn/866028.Doc
smb.vwbnt.cn/155551.Doc
smb.vwbnt.cn/068668.Doc
smb.vwbnt.cn/066220.Doc
smb.vwbnt.cn/446684.Doc
smb.vwbnt.cn/044288.Doc
smb.vwbnt.cn/848426.Doc
smv.vwbnt.cn/426062.Doc
smv.vwbnt.cn/686288.Doc
smv.vwbnt.cn/600264.Doc
smv.vwbnt.cn/866868.Doc
smv.vwbnt.cn/844084.Doc
smv.vwbnt.cn/660084.Doc
smv.vwbnt.cn/640084.Doc
smv.vwbnt.cn/513577.Doc
smv.vwbnt.cn/404082.Doc
smv.vwbnt.cn/420644.Doc
smc.vwbnt.cn/666844.Doc
smc.vwbnt.cn/206846.Doc
smc.vwbnt.cn/006884.Doc
smc.vwbnt.cn/848046.Doc
smc.vwbnt.cn/444848.Doc
smc.vwbnt.cn/204264.Doc
smc.vwbnt.cn/959191.Doc
smc.vwbnt.cn/024662.Doc
smc.vwbnt.cn/648862.Doc
smc.vwbnt.cn/579993.Doc
smx.vwbnt.cn/422848.Doc
smx.vwbnt.cn/080086.Doc
smx.vwbnt.cn/042248.Doc
smx.vwbnt.cn/486804.Doc
smx.vwbnt.cn/262464.Doc
smx.vwbnt.cn/420808.Doc
smx.vwbnt.cn/802404.Doc
smx.vwbnt.cn/660886.Doc
smx.vwbnt.cn/593333.Doc
smx.vwbnt.cn/777911.Doc
smz.vwbnt.cn/068866.Doc
smz.vwbnt.cn/084424.Doc
smz.vwbnt.cn/826842.Doc
smz.vwbnt.cn/426082.Doc
smz.vwbnt.cn/228828.Doc
smz.vwbnt.cn/666846.Doc
smz.vwbnt.cn/868668.Doc
smz.vwbnt.cn/048824.Doc
smz.vwbnt.cn/884628.Doc
smz.vwbnt.cn/480428.Doc
sml.vwbnt.cn/426464.Doc
sml.vwbnt.cn/640282.Doc
sml.vwbnt.cn/088644.Doc
sml.vwbnt.cn/482444.Doc
sml.vwbnt.cn/028082.Doc
sml.vwbnt.cn/191513.Doc
sml.vwbnt.cn/115513.Doc
sml.vwbnt.cn/822244.Doc
sml.vwbnt.cn/422826.Doc
sml.vwbnt.cn/480022.Doc
smk.vwbnt.cn/884448.Doc
smk.vwbnt.cn/868486.Doc
smk.vwbnt.cn/066082.Doc
smk.vwbnt.cn/284408.Doc
smk.vwbnt.cn/668844.Doc
smk.vwbnt.cn/244840.Doc
smk.vwbnt.cn/880808.Doc
smk.vwbnt.cn/640664.Doc
smk.vwbnt.cn/484460.Doc
smk.vwbnt.cn/428448.Doc
smj.vwbnt.cn/284424.Doc
smj.vwbnt.cn/537735.Doc
smj.vwbnt.cn/686608.Doc
smj.vwbnt.cn/424684.Doc
smj.vwbnt.cn/008828.Doc
smj.vwbnt.cn/844080.Doc
smj.vwbnt.cn/264000.Doc
smj.vwbnt.cn/628480.Doc
smj.vwbnt.cn/517155.Doc
smj.vwbnt.cn/008682.Doc
smh.vwbnt.cn/426446.Doc
smh.vwbnt.cn/482648.Doc
smh.vwbnt.cn/864208.Doc
smh.vwbnt.cn/860800.Doc
smh.vwbnt.cn/402084.Doc
smh.vwbnt.cn/682606.Doc
smh.vwbnt.cn/002666.Doc
smh.vwbnt.cn/682864.Doc
smh.vwbnt.cn/248022.Doc
smh.vwbnt.cn/646862.Doc
smg.vwbnt.cn/311171.Doc
smg.vwbnt.cn/622084.Doc
smg.vwbnt.cn/682684.Doc
smg.vwbnt.cn/848086.Doc
smg.vwbnt.cn/226020.Doc
smg.vwbnt.cn/002642.Doc
smg.vwbnt.cn/822484.Doc
smg.vwbnt.cn/268860.Doc
smg.vwbnt.cn/644020.Doc
smg.vwbnt.cn/002444.Doc
smf.vwbnt.cn/626848.Doc
smf.vwbnt.cn/286806.Doc
smf.vwbnt.cn/628428.Doc
smf.vwbnt.cn/804628.Doc
smf.vwbnt.cn/804686.Doc
smf.vwbnt.cn/266624.Doc
smf.vwbnt.cn/408048.Doc
smf.vwbnt.cn/842884.Doc
smf.vwbnt.cn/020480.Doc
smf.vwbnt.cn/844004.Doc
smd.vwbnt.cn/484208.Doc
smd.vwbnt.cn/828680.Doc
smd.vwbnt.cn/602620.Doc
smd.vwbnt.cn/626022.Doc
smd.vwbnt.cn/420448.Doc
smd.vwbnt.cn/006288.Doc
smd.vwbnt.cn/240284.Doc
smd.vwbnt.cn/240200.Doc
smd.vwbnt.cn/000428.Doc
smd.vwbnt.cn/080000.Doc
sms.vwbnt.cn/662606.Doc
sms.vwbnt.cn/644480.Doc
sms.vwbnt.cn/462408.Doc
sms.vwbnt.cn/048488.Doc
sms.vwbnt.cn/024800.Doc
sms.vwbnt.cn/462044.Doc
sms.vwbnt.cn/604646.Doc
sms.vwbnt.cn/646200.Doc
sms.vwbnt.cn/202640.Doc
sms.vwbnt.cn/642042.Doc
sma.vwbnt.cn/486220.Doc
sma.vwbnt.cn/804220.Doc
