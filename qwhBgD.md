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

wia.7nvyou1.cn/06448.Doc
wia.7nvyou1.cn/40604.Doc
wia.7nvyou1.cn/13135.Doc
wia.7nvyou1.cn/35713.Doc
wia.7nvyou1.cn/20664.Doc
wia.7nvyou1.cn/82082.Doc
wia.7nvyou1.cn/88446.Doc
wia.7nvyou1.cn/64268.Doc
wia.7nvyou1.cn/48420.Doc
wia.7nvyou1.cn/28204.Doc
wip.7nvyou1.cn/51137.Doc
wip.7nvyou1.cn/00088.Doc
wip.7nvyou1.cn/48240.Doc
wip.7nvyou1.cn/40862.Doc
wip.7nvyou1.cn/48460.Doc
wip.7nvyou1.cn/26460.Doc
wip.7nvyou1.cn/64424.Doc
wip.7nvyou1.cn/60624.Doc
wip.7nvyou1.cn/60068.Doc
wip.7nvyou1.cn/80404.Doc
wio.7nvyou1.cn/02062.Doc
wio.7nvyou1.cn/22662.Doc
wio.7nvyou1.cn/11955.Doc
wio.7nvyou1.cn/08004.Doc
wio.7nvyou1.cn/00004.Doc
wio.7nvyou1.cn/80840.Doc
wio.7nvyou1.cn/20224.Doc
wio.7nvyou1.cn/24626.Doc
wio.7nvyou1.cn/42404.Doc
wio.7nvyou1.cn/04404.Doc
wii.7nvyou1.cn/02004.Doc
wii.7nvyou1.cn/24682.Doc
wii.7nvyou1.cn/28422.Doc
wii.7nvyou1.cn/46088.Doc
wii.7nvyou1.cn/40062.Doc
wii.7nvyou1.cn/64422.Doc
wii.7nvyou1.cn/82266.Doc
wii.7nvyou1.cn/82888.Doc
wii.7nvyou1.cn/08200.Doc
wii.7nvyou1.cn/44640.Doc
wiu.7nvyou1.cn/73379.Doc
wiu.7nvyou1.cn/26862.Doc
wiu.7nvyou1.cn/88460.Doc
wiu.7nvyou1.cn/08244.Doc
wiu.7nvyou1.cn/82648.Doc
wiu.7nvyou1.cn/60808.Doc
wiu.7nvyou1.cn/06464.Doc
wiu.7nvyou1.cn/88840.Doc
wiu.7nvyou1.cn/60288.Doc
wiu.7nvyou1.cn/24688.Doc
wiy.7nvyou1.cn/60286.Doc
wiy.7nvyou1.cn/08288.Doc
wiy.7nvyou1.cn/62046.Doc
wiy.7nvyou1.cn/48484.Doc
wiy.7nvyou1.cn/40866.Doc
wiy.7nvyou1.cn/66826.Doc
wiy.7nvyou1.cn/79315.Doc
wiy.7nvyou1.cn/53937.Doc
wiy.7nvyou1.cn/04042.Doc
wiy.7nvyou1.cn/02004.Doc
wit.7nvyou1.cn/06668.Doc
wit.7nvyou1.cn/04466.Doc
wit.7nvyou1.cn/88826.Doc
wit.7nvyou1.cn/22466.Doc
wit.7nvyou1.cn/08624.Doc
wit.7nvyou1.cn/42806.Doc
wit.7nvyou1.cn/86046.Doc
wit.7nvyou1.cn/06200.Doc
wit.7nvyou1.cn/46624.Doc
wit.7nvyou1.cn/60060.Doc
wir.7nvyou1.cn/08662.Doc
wir.7nvyou1.cn/48626.Doc
wir.7nvyou1.cn/44444.Doc
wir.7nvyou1.cn/62202.Doc
wir.7nvyou1.cn/11337.Doc
wir.7nvyou1.cn/00024.Doc
wir.7nvyou1.cn/82262.Doc
wir.7nvyou1.cn/84060.Doc
wir.7nvyou1.cn/84060.Doc
wir.7nvyou1.cn/20862.Doc
wie.7nvyou1.cn/48404.Doc
wie.7nvyou1.cn/84044.Doc
wie.7nvyou1.cn/22446.Doc
wie.7nvyou1.cn/00884.Doc
wie.7nvyou1.cn/59133.Doc
wie.7nvyou1.cn/06884.Doc
wie.7nvyou1.cn/04842.Doc
wie.7nvyou1.cn/24668.Doc
wie.7nvyou1.cn/80464.Doc
wie.7nvyou1.cn/22220.Doc
wiw.7nvyou1.cn/00048.Doc
wiw.7nvyou1.cn/04626.Doc
wiw.7nvyou1.cn/80600.Doc
wiw.7nvyou1.cn/86006.Doc
wiw.7nvyou1.cn/22680.Doc
wiw.7nvyou1.cn/86888.Doc
wiw.7nvyou1.cn/11991.Doc
wiw.7nvyou1.cn/00688.Doc
wiw.7nvyou1.cn/64880.Doc
wiw.7nvyou1.cn/66286.Doc
wiq.7nvyou1.cn/39579.Doc
wiq.7nvyou1.cn/00060.Doc
wiq.7nvyou1.cn/68000.Doc
wiq.7nvyou1.cn/26200.Doc
wiq.7nvyou1.cn/80846.Doc
wiq.7nvyou1.cn/62428.Doc
wiq.7nvyou1.cn/20486.Doc
wiq.7nvyou1.cn/22204.Doc
wiq.7nvyou1.cn/48824.Doc
wiq.7nvyou1.cn/51719.Doc
wum.7nvyou1.cn/82606.Doc
wum.7nvyou1.cn/28862.Doc
wum.7nvyou1.cn/02444.Doc
wum.7nvyou1.cn/00608.Doc
wum.7nvyou1.cn/40840.Doc
wum.7nvyou1.cn/80284.Doc
wum.7nvyou1.cn/60428.Doc
wum.7nvyou1.cn/24888.Doc
wum.7nvyou1.cn/84400.Doc
wum.7nvyou1.cn/26802.Doc
wun.7nvyou1.cn/73179.Doc
wun.7nvyou1.cn/88448.Doc
wun.7nvyou1.cn/66040.Doc
wun.7nvyou1.cn/64824.Doc
wun.7nvyou1.cn/68480.Doc
wun.7nvyou1.cn/86020.Doc
wun.7nvyou1.cn/44840.Doc
wun.7nvyou1.cn/60882.Doc
wun.7nvyou1.cn/08080.Doc
wun.7nvyou1.cn/99393.Doc
wub.7nvyou1.cn/46882.Doc
wub.7nvyou1.cn/60246.Doc
wub.7nvyou1.cn/24662.Doc
wub.7nvyou1.cn/60064.Doc
wub.7nvyou1.cn/62222.Doc
wub.7nvyou1.cn/13139.Doc
wub.7nvyou1.cn/28886.Doc
wub.7nvyou1.cn/08642.Doc
wub.7nvyou1.cn/26460.Doc
wub.7nvyou1.cn/22062.Doc
wuv.7nvyou1.cn/28842.Doc
wuv.7nvyou1.cn/62000.Doc
wuv.7nvyou1.cn/04002.Doc
wuv.7nvyou1.cn/60220.Doc
wuv.7nvyou1.cn/88206.Doc
wuv.7nvyou1.cn/06400.Doc
wuv.7nvyou1.cn/06042.Doc
wuv.7nvyou1.cn/48242.Doc
wuv.7nvyou1.cn/48444.Doc
wuv.7nvyou1.cn/26680.Doc
wuc.7nvyou1.cn/60604.Doc
wuc.7nvyou1.cn/40842.Doc
wuc.7nvyou1.cn/48884.Doc
wuc.7nvyou1.cn/48402.Doc
wuc.7nvyou1.cn/42268.Doc
wuc.7nvyou1.cn/68082.Doc
wuc.7nvyou1.cn/04020.Doc
wuc.7nvyou1.cn/75999.Doc
wuc.7nvyou1.cn/44200.Doc
wuc.7nvyou1.cn/24608.Doc
wux.7nvyou1.cn/82006.Doc
wux.7nvyou1.cn/48200.Doc
wux.7nvyou1.cn/06668.Doc
wux.7nvyou1.cn/42880.Doc
wux.7nvyou1.cn/17597.Doc
wux.7nvyou1.cn/40246.Doc
wux.7nvyou1.cn/06808.Doc
wux.7nvyou1.cn/42022.Doc
wux.7nvyou1.cn/02622.Doc
wux.7nvyou1.cn/04804.Doc
wuz.7nvyou1.cn/84286.Doc
wuz.7nvyou1.cn/68662.Doc
wuz.7nvyou1.cn/08842.Doc
wuz.7nvyou1.cn/88660.Doc
wuz.7nvyou1.cn/06686.Doc
wuz.7nvyou1.cn/88248.Doc
wuz.7nvyou1.cn/93357.Doc
wuz.7nvyou1.cn/46008.Doc
wuz.7nvyou1.cn/62866.Doc
wuz.7nvyou1.cn/62240.Doc
wul.7nvyou1.cn/06040.Doc
wul.7nvyou1.cn/11193.Doc
wul.7nvyou1.cn/84244.Doc
wul.7nvyou1.cn/39311.Doc
wul.7nvyou1.cn/22406.Doc
wul.7nvyou1.cn/80206.Doc
wul.7nvyou1.cn/62842.Doc
wul.7nvyou1.cn/59111.Doc
wul.7nvyou1.cn/48820.Doc
wul.7nvyou1.cn/11973.Doc
wuk.7nvyou1.cn/37331.Doc
wuk.7nvyou1.cn/35757.Doc
wuk.7nvyou1.cn/13951.Doc
wuk.7nvyou1.cn/66604.Doc
wuk.7nvyou1.cn/40662.Doc
wuk.7nvyou1.cn/00626.Doc
wuk.7nvyou1.cn/26806.Doc
wuk.7nvyou1.cn/11553.Doc
wuk.7nvyou1.cn/42466.Doc
wuk.7nvyou1.cn/66626.Doc
wuj.7nvyou1.cn/64804.Doc
wuj.7nvyou1.cn/40868.Doc
wuj.7nvyou1.cn/28682.Doc
wuj.7nvyou1.cn/88204.Doc
wuj.7nvyou1.cn/24806.Doc
wuj.7nvyou1.cn/24644.Doc
wuj.7nvyou1.cn/48046.Doc
wuj.7nvyou1.cn/64020.Doc
wuj.7nvyou1.cn/15733.Doc
wuj.7nvyou1.cn/40608.Doc
wuh.7nvyou1.cn/24826.Doc
wuh.7nvyou1.cn/00006.Doc
wuh.7nvyou1.cn/80888.Doc
wuh.7nvyou1.cn/82224.Doc
wuh.7nvyou1.cn/04422.Doc
wuh.7nvyou1.cn/46004.Doc
wuh.7nvyou1.cn/22224.Doc
wuh.7nvyou1.cn/15359.Doc
wuh.7nvyou1.cn/22086.Doc
wuh.7nvyou1.cn/04446.Doc
wug.7nvyou1.cn/00844.Doc
wug.7nvyou1.cn/26842.Doc
wug.7nvyou1.cn/82222.Doc
wug.7nvyou1.cn/06866.Doc
wug.7nvyou1.cn/91773.Doc
wug.7nvyou1.cn/80426.Doc
wug.7nvyou1.cn/84820.Doc
wug.7nvyou1.cn/22448.Doc
wug.7nvyou1.cn/28064.Doc
wug.7nvyou1.cn/40062.Doc
wuf.7nvyou1.cn/06626.Doc
wuf.7nvyou1.cn/44622.Doc
wuf.7nvyou1.cn/40084.Doc
wuf.7nvyou1.cn/84822.Doc
wuf.7nvyou1.cn/20248.Doc
wuf.7nvyou1.cn/40200.Doc
wuf.7nvyou1.cn/82628.Doc
wuf.7nvyou1.cn/53777.Doc
wuf.7nvyou1.cn/08246.Doc
wuf.7nvyou1.cn/95515.Doc
wud.7nvyou1.cn/02606.Doc
wud.7nvyou1.cn/93333.Doc
wud.7nvyou1.cn/55591.Doc
wud.7nvyou1.cn/06686.Doc
wud.7nvyou1.cn/40068.Doc
wud.7nvyou1.cn/55971.Doc
wud.7nvyou1.cn/44444.Doc
wud.7nvyou1.cn/97977.Doc
wud.7nvyou1.cn/06460.Doc
wud.7nvyou1.cn/46864.Doc
wus.7nvyou1.cn/99391.Doc
wus.7nvyou1.cn/80024.Doc
wus.7nvyou1.cn/22662.Doc
wus.7nvyou1.cn/28004.Doc
wus.7nvyou1.cn/46846.Doc
wus.7nvyou1.cn/04006.Doc
wus.7nvyou1.cn/04822.Doc
wus.7nvyou1.cn/20242.Doc
wus.7nvyou1.cn/60426.Doc
wus.7nvyou1.cn/79139.Doc
wua.7nvyou1.cn/60844.Doc
wua.7nvyou1.cn/28028.Doc
wua.7nvyou1.cn/80822.Doc
wua.7nvyou1.cn/62464.Doc
wua.7nvyou1.cn/00604.Doc
wua.7nvyou1.cn/20804.Doc
wua.7nvyou1.cn/20626.Doc
wua.7nvyou1.cn/40664.Doc
wua.7nvyou1.cn/46622.Doc
wua.7nvyou1.cn/48244.Doc
wup.7nvyou1.cn/02000.Doc
wup.7nvyou1.cn/17513.Doc
wup.7nvyou1.cn/26260.Doc
wup.7nvyou1.cn/60846.Doc
wup.7nvyou1.cn/62600.Doc
wup.7nvyou1.cn/75773.Doc
wup.7nvyou1.cn/84426.Doc
wup.7nvyou1.cn/46484.Doc
wup.7nvyou1.cn/64008.Doc
wup.7nvyou1.cn/24800.Doc
wuo.7nvyou1.cn/42828.Doc
wuo.7nvyou1.cn/06042.Doc
wuo.7nvyou1.cn/68000.Doc
wuo.7nvyou1.cn/08826.Doc
wuo.7nvyou1.cn/44606.Doc
wuo.7nvyou1.cn/95113.Doc
wuo.7nvyou1.cn/88004.Doc
wuo.7nvyou1.cn/24826.Doc
wuo.7nvyou1.cn/80484.Doc
wuo.7nvyou1.cn/82448.Doc
wui.7nvyou1.cn/06444.Doc
wui.7nvyou1.cn/20060.Doc
wui.7nvyou1.cn/53177.Doc
wui.7nvyou1.cn/44040.Doc
wui.7nvyou1.cn/24884.Doc
wui.7nvyou1.cn/62224.Doc
wui.7nvyou1.cn/64648.Doc
wui.7nvyou1.cn/60822.Doc
wui.7nvyou1.cn/80024.Doc
wui.7nvyou1.cn/22286.Doc
wuu.7nvyou1.cn/00606.Doc
wuu.7nvyou1.cn/82828.Doc
wuu.7nvyou1.cn/06884.Doc
wuu.7nvyou1.cn/99375.Doc
wuu.7nvyou1.cn/48880.Doc
wuu.7nvyou1.cn/48684.Doc
wuu.7nvyou1.cn/40248.Doc
wuu.7nvyou1.cn/28200.Doc
wuu.7nvyou1.cn/84484.Doc
wuu.7nvyou1.cn/20408.Doc
wuy.7nvyou1.cn/44864.Doc
wuy.7nvyou1.cn/08286.Doc
wuy.7nvyou1.cn/22208.Doc
wuy.7nvyou1.cn/44282.Doc
wuy.7nvyou1.cn/62482.Doc
wuy.7nvyou1.cn/08860.Doc
wuy.7nvyou1.cn/84484.Doc
wuy.7nvyou1.cn/06442.Doc
wuy.7nvyou1.cn/02868.Doc
wuy.7nvyou1.cn/26426.Doc
wut.7nvyou1.cn/80844.Doc
wut.7nvyou1.cn/42682.Doc
wut.7nvyou1.cn/86062.Doc
wut.7nvyou1.cn/82466.Doc
wut.7nvyou1.cn/93137.Doc
wut.7nvyou1.cn/28602.Doc
wut.7nvyou1.cn/91133.Doc
wut.7nvyou1.cn/28460.Doc
wut.7nvyou1.cn/64640.Doc
wut.7nvyou1.cn/64280.Doc
wur.7nvyou1.cn/04684.Doc
wur.7nvyou1.cn/42846.Doc
wur.7nvyou1.cn/24808.Doc
wur.7nvyou1.cn/22642.Doc
wur.7nvyou1.cn/24086.Doc
wur.7nvyou1.cn/24208.Doc
wur.7nvyou1.cn/00408.Doc
wur.7nvyou1.cn/08440.Doc
wur.7nvyou1.cn/84262.Doc
wur.7nvyou1.cn/35317.Doc
wue.7nvyou1.cn/24642.Doc
wue.7nvyou1.cn/64246.Doc
wue.7nvyou1.cn/68402.Doc
wue.7nvyou1.cn/00008.Doc
wue.7nvyou1.cn/28826.Doc
wue.7nvyou1.cn/80484.Doc
wue.7nvyou1.cn/71597.Doc
wue.7nvyou1.cn/00460.Doc
wue.7nvyou1.cn/64240.Doc
wue.7nvyou1.cn/60044.Doc
wuw.7nvyou1.cn/46820.Doc
wuw.7nvyou1.cn/80206.Doc
wuw.7nvyou1.cn/26420.Doc
wuw.7nvyou1.cn/00622.Doc
wuw.7nvyou1.cn/02404.Doc
wuw.7nvyou1.cn/44006.Doc
wuw.7nvyou1.cn/22284.Doc
wuw.7nvyou1.cn/04482.Doc
wuw.7nvyou1.cn/02202.Doc
wuw.7nvyou1.cn/24864.Doc
wuq.7nvyou1.cn/26860.Doc
wuq.7nvyou1.cn/82626.Doc
wuq.7nvyou1.cn/04464.Doc
wuq.7nvyou1.cn/08428.Doc
wuq.7nvyou1.cn/46848.Doc
wuq.7nvyou1.cn/84846.Doc
wuq.7nvyou1.cn/42042.Doc
wuq.7nvyou1.cn/84280.Doc
wuq.7nvyou1.cn/84620.Doc
wuq.7nvyou1.cn/22684.Doc
wym.7nvyou1.cn/04066.Doc
wym.7nvyou1.cn/33119.Doc
wym.7nvyou1.cn/40226.Doc
wym.7nvyou1.cn/26082.Doc
wym.7nvyou1.cn/13957.Doc
wym.7nvyou1.cn/66424.Doc
wym.7nvyou1.cn/97937.Doc
wym.7nvyou1.cn/00868.Doc
wym.7nvyou1.cn/04886.Doc
wym.7nvyou1.cn/82662.Doc
wyn.7nvyou1.cn/40862.Doc
wyn.7nvyou1.cn/48228.Doc
wyn.7nvyou1.cn/24480.Doc
wyn.7nvyou1.cn/64862.Doc
wyn.7nvyou1.cn/40688.Doc
wyn.7nvyou1.cn/00400.Doc
wyn.7nvyou1.cn/42286.Doc
wyn.7nvyou1.cn/00040.Doc
wyn.7nvyou1.cn/26866.Doc
wyn.7nvyou1.cn/20806.Doc
wyb.7nvyou1.cn/28868.Doc
wyb.7nvyou1.cn/88842.Doc
wyb.7nvyou1.cn/68048.Doc
wyb.7nvyou1.cn/75337.Doc
wyb.7nvyou1.cn/00022.Doc
wyb.7nvyou1.cn/86284.Doc
wyb.7nvyou1.cn/84084.Doc
wyb.7nvyou1.cn/46440.Doc
wyb.7nvyou1.cn/17119.Doc
wyb.7nvyou1.cn/66084.Doc
wyv.7nvyou1.cn/84468.Doc
wyv.7nvyou1.cn/82466.Doc
wyv.7nvyou1.cn/60484.Doc
wyv.7nvyou1.cn/26202.Doc
wyv.7nvyou1.cn/68200.Doc
wyv.7nvyou1.cn/66888.Doc
wyv.7nvyou1.cn/22446.Doc
wyv.7nvyou1.cn/00200.Doc
wyv.7nvyou1.cn/48468.Doc
wyv.7nvyou1.cn/26282.Doc
wyc.7nvyou1.cn/28842.Doc
wyc.7nvyou1.cn/02240.Doc
wyc.7nvyou1.cn/22820.Doc
wyc.7nvyou1.cn/08620.Doc
wyc.7nvyou1.cn/20248.Doc
wyc.7nvyou1.cn/71911.Doc
wyc.7nvyou1.cn/62444.Doc
wyc.7nvyou1.cn/33175.Doc
wyc.7nvyou1.cn/20006.Doc
wyc.7nvyou1.cn/86826.Doc
wyx.7nvyou1.cn/02844.Doc
wyx.7nvyou1.cn/44242.Doc
wyx.7nvyou1.cn/08048.Doc
wyx.7nvyou1.cn/39337.Doc
wyx.7nvyou1.cn/44226.Doc
wyx.7nvyou1.cn/28842.Doc
wyx.7nvyou1.cn/42282.Doc
wyx.7nvyou1.cn/37351.Doc
wyx.7nvyou1.cn/08062.Doc
wyx.7nvyou1.cn/40888.Doc
wyz.7nvyou1.cn/95777.Doc
wyz.7nvyou1.cn/02664.Doc
wyz.7nvyou1.cn/40668.Doc
wyz.7nvyou1.cn/64648.Doc
wyz.7nvyou1.cn/20000.Doc
wyz.7nvyou1.cn/66608.Doc
wyz.7nvyou1.cn/60642.Doc
wyz.7nvyou1.cn/86028.Doc
wyz.7nvyou1.cn/20040.Doc
wyz.7nvyou1.cn/60202.Doc
wyl.7nvyou1.cn/26806.Doc
wyl.7nvyou1.cn/88222.Doc
wyl.7nvyou1.cn/24884.Doc
wyl.7nvyou1.cn/11197.Doc
wyl.7nvyou1.cn/95773.Doc
wyl.7nvyou1.cn/42206.Doc
wyl.7nvyou1.cn/46646.Doc
wyl.7nvyou1.cn/08042.Doc
wyl.7nvyou1.cn/06664.Doc
wyl.7nvyou1.cn/82428.Doc
wyk.7nvyou1.cn/22440.Doc
wyk.7nvyou1.cn/40802.Doc
wyk.7nvyou1.cn/66428.Doc
wyk.7nvyou1.cn/42264.Doc
wyk.7nvyou1.cn/48066.Doc
wyk.7nvyou1.cn/08662.Doc
wyk.7nvyou1.cn/04466.Doc
wyk.7nvyou1.cn/84668.Doc
wyk.7nvyou1.cn/24242.Doc
wyk.7nvyou1.cn/44202.Doc
wyj.7nvyou1.cn/62884.Doc
wyj.7nvyou1.cn/51939.Doc
wyj.7nvyou1.cn/02260.Doc
wyj.7nvyou1.cn/22488.Doc
wyj.7nvyou1.cn/66466.Doc
wyj.7nvyou1.cn/42602.Doc
wyj.7nvyou1.cn/86244.Doc
wyj.7nvyou1.cn/40024.Doc
wyj.7nvyou1.cn/28400.Doc
wyj.7nvyou1.cn/44602.Doc
wyh.7nvyou1.cn/17535.Doc
wyh.7nvyou1.cn/42468.Doc
wyh.7nvyou1.cn/28464.Doc
wyh.7nvyou1.cn/60622.Doc
wyh.7nvyou1.cn/82606.Doc
wyh.7nvyou1.cn/86846.Doc
wyh.7nvyou1.cn/24468.Doc
wyh.7nvyou1.cn/80882.Doc
wyh.7nvyou1.cn/15111.Doc
wyh.7nvyou1.cn/20082.Doc
wyg.7nvyou1.cn/26440.Doc
wyg.7nvyou1.cn/59719.Doc
wyg.7nvyou1.cn/60080.Doc
wyg.7nvyou1.cn/99971.Doc
wyg.7nvyou1.cn/60620.Doc
wyg.7nvyou1.cn/28000.Doc
wyg.7nvyou1.cn/02602.Doc
wyg.7nvyou1.cn/08088.Doc
wyg.7nvyou1.cn/40068.Doc
wyg.7nvyou1.cn/62228.Doc
