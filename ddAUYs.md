=================================================================
 Python asyncio 流操作：基于 Stream 的高层网络编程
=================================================================

Streams API 是 asyncio 的高层网络抽象，基于 Protocol/Transport
构建，提供了类似同步编程的读写接口，适用于大多数网络应用。

=================================================================
 一、StreamReader 和 StreamWriter 核心用法
=================================================================

import asyncio
import logging

logging.basicConfig(level=logging.INFO, format="%(message)s")
logger = logging.getLogger(__name__)

async def stream_basic_demo():
    """
    StreamReader 和 StreamWriter 的基本操作。
    open_connection 返回 (reader, writer) 元组。
    """
    # 建立 TCP 连接
    reader, writer = await asyncio.open_connection(
        "httpbin.org", 80,
        # 限制读取缓冲区大小（字节）
        limit=2 ** 16,  # 64KB
    )

    # writer.write() - 将数据写入缓冲区（非阻塞）
    request = (
        "GET /get HTTP/1.1\r\n"
        "Host: httpbin.org\r\n"
        "Connection: close\r\n"
        "\r\n"
    )
    writer.write(request.encode())

    # drain() - 刷新缓冲区，等待数据真正发送
    # 当写缓冲区超过高水位线时，drain() 会阻塞直到数据被发送
    await writer.drain()
    logger.info("请求已发送")

    # reader.read() - 读取指定字节数
    # read(n=-1): n=-1 表示读取直到 EOF
    response = await reader.read(4096)
    logger.info(f"收到 {len(response)} 字节")
    logger.info(f"响应前 200 字符:\n{response[:200].decode()}")

    # 优雅关闭连接
    writer.close()
    await writer.wait_closed()

=================================================================
 二、read / readline / readexactly 的区别
=================================================================

async def read_methods_demo():
    """演示三种读取方式的不同用途"""
    reader, writer = await asyncio.open_connection(
        "httpbin.org", 80
    )

    writer.write(b"GET /get HTTP/1.1\r\nHost: httpbin.org\r\n\r\n")
    await writer.drain()

    # 1. read(n): 读取最多 n 字节（可能少于 n）
    header = await reader.read(100)  # 读取前 100 字节
    logger.info(f"read(100): 实际读取 {len(header)} 字节")

    # 2. readline(): 读取直到遇到换行符（包含）
    # 适合逐行解析的文本协议（HTTP、SMTP、FTP）
    line = await reader.readline()
    logger.info(f"readline(): {line.decode().rstrip()}")

    # 3. readexactly(n): 精确读取 n 字节，否则抛出异常
    # 适合固定长度消息头的二进制协议
    try:
        chunk = await asyncio.wait_for(
            reader.readexactly(50), timeout=2.0
        )
        logger.info(f"readexactly(50): 精确读取 {len(chunk)} 字节")
    except asyncio.IncompleteReadError as e:
        # 在读完 n 字节之前遇到 EOF
        logger.warning(f"数据不足: 需要 {e.expected} 字节，收到 {len(e.partial)}")
    except asyncio.TimeoutError:
        logger.warning("读取超时")

    writer.close()
    await writer.wait_closed()

=================================================================
 三、使用 asyncio.start_server 创建 TCP 服务
=================================================================

async def handle_echo_client(
    reader: asyncio.StreamReader,
    writer: asyncio.StreamWriter,
):
    """
    处理每个客户端连接的协程。
    start_server 对每个连接调用此函数。
    """
    peername = writer.get_extra_info("peername")
    logger.info(f"客户端连接: {peername}")

    try:
        while True:
            # 读取一行数据
            data = await reader.readline()
            if not data:
                break  # EOF，客户端关闭了连接

            message = data.decode().strip()
            logger.info(f"收到: {message}")

            # 处理消息并回显
            response = f"Echo: {message}\n".encode()
            writer.write(response)
            await writer.drain()

    except asyncio.CancelledError:
        logger.info(f"客户端 {peername} 连接被取消")
    except Exception as e:
        logger.error(f"处理客户端 {peername} 时出错: {e}")
    finally:
        logger.info(f"关闭连接: {peername}")
        writer.close()
        await writer.wait_closed()

async def stream_echo_server():
    """
    使用 asyncio.start_server 创建 TCP 回显服务器。
    比低层 Protocol API 更简洁。
    """
    server = await asyncio.start_server(
        handle_echo_client,  # 客户端连接处理函数
        "127.0.0.1",
        9999,
        # 限制读取缓冲区大小
        limit=2 ** 16,
        # 允许地址重用
        reuse_address=True,
        # 最大同时挂起连接数
        backlog=128,
    )

    addr = server.sockets[0].getsockname()
    logger.info(f"流式服务器启动在 {addr}")

    # 使用异步上下文管理器运行服务器
    async with server:
        await server.serve_forever()

=================================================================
 四、完整 TCP 回显客户端
=================================================================

async def stream_echo_client():
    """连接到回显服务器的客户端"""
    reader, writer = await asyncio.open_connection(
        "127.0.0.1", 9999
    )

    messages = ["Hello", "World", "asyncio", "Streams", "exit"]

    for msg in messages:
        # 发送消息（行协议：每行一条消息）
        writer.write(f"{msg}\n".encode())
        await writer.drain()

        # 读取响应
        response = await reader.readline()
        if response:
            logger.info(f"服务器响应: {response.decode().strip()}")

        await asyncio.sleep(0.5)

    logger.info("客户端关闭")
    writer.close()
    await writer.wait_closed()

=================================================================
 五、行协议解析器
=================================================================

class LineBasedProtocol:
    """
    基于行的协议处理工具。
    适用于聊天协议、简单命令协议等。
    """

    @staticmethod
    def encode_message(msg: str) -> bytes:
        """将消息编码为行协议格式"""
        return f"{msg}\n".encode("utf-8")

    @staticmethod
    def decode_line(data: bytes) -> str:
        """解码一行数据"""
        return data.decode("utf-8").rstrip("\r\n")

async def line_protocol_server():
    """基于行协议的聊天服务器"""

    async def handler(reader, writer):
        name = None
        while True:
            data = await reader.readline()
            if not data:
                break
            msg = LineBasedProtocol.decode_line(data)

            if name is None:
                name = msg  # 第一行作为用户名
                logger.info(f"用户 {name} 加入")
                continue

            logger.info(f"[{name}] {msg}")
            # 广播给其他客户端（简化实现：仅回显）
            response = f"[{name}] {msg}\n"
            writer.write(response.encode())
            await writer.drain()

        logger.info(f"用户 {name} 离开")

    server = await asyncio.start_server(handler, "127.0.0.1", 9998)
    async with server:
        await server.serve_forever()

=================================================================
 六、Stream API 最佳实践
=================================================================

# 1. 始终调用 drain()：write() 后调用 drain() 确保数据发送
# 2. 使用 wait_closed()：close() 后调用等待连接完全关闭
# 3. 选择正确的读取方法：
#    - read(): 不确定长度的数据
#    - readline(): 文本协议（HTTP、SMTP）
#    - readexactly(): 固定长度消息头
# 4. 设置合理的缓冲区 limit：避免内存溢出
# 5. 异常处理：捕获 CancelledError、IncompleteReadError
# 6. 不要在线程中使用：Stream API 只能在事件循环线程中调用

# 性能提示：
# - Streams API 比 Protocol API 慢约 10-20%
# - 高吞吐场景（>10K RPS）考虑使用 Protocol API
# - 使用 uvloop 可显著提升 Streams 性能

=================================================================
 七、总结
=================================================================

# 1. open_connection() 创建客户端连接
# 2. start_server() 创建服务器
# 3. reader.read/readline/readexactly 适应不同读取场景
# 4. write + drain 确保数据的可靠发送
# 5. Streams API 适合多数网络应用
# 6. 行协议是文本类网络协议的基础模式

if __name__ == "__main__":
    # 启动服务器
    asyncio.run(stream_echo_server())

qax.wwwcao1314c.cn/84688.Doc
qax.wwwcao1314c.cn/66640.Doc
qax.wwwcao1314c.cn/71571.Doc
qax.wwwcao1314c.cn/46028.Doc
qax.wwwcao1314c.cn/46444.Doc
qax.wwwcao1314c.cn/20440.Doc
qax.wwwcao1314c.cn/26666.Doc
qax.wwwcao1314c.cn/48282.Doc
qax.wwwcao1314c.cn/84064.Doc
qax.wwwcao1314c.cn/79153.Doc
qaz.wwwcao1314c.cn/08442.Doc
qaz.wwwcao1314c.cn/48026.Doc
qaz.wwwcao1314c.cn/95591.Doc
qaz.wwwcao1314c.cn/48602.Doc
qaz.wwwcao1314c.cn/88848.Doc
qaz.wwwcao1314c.cn/80824.Doc
qaz.wwwcao1314c.cn/24440.Doc
qaz.wwwcao1314c.cn/82464.Doc
qaz.wwwcao1314c.cn/46806.Doc
qaz.wwwcao1314c.cn/80864.Doc
qal.wwwcao1314c.cn/00806.Doc
qal.wwwcao1314c.cn/88648.Doc
qal.wwwcao1314c.cn/44400.Doc
qal.wwwcao1314c.cn/02640.Doc
qal.wwwcao1314c.cn/24200.Doc
qal.wwwcao1314c.cn/75195.Doc
qal.wwwcao1314c.cn/60008.Doc
qal.wwwcao1314c.cn/80268.Doc
qal.wwwcao1314c.cn/88408.Doc
qal.wwwcao1314c.cn/84482.Doc
qak.wwwcao1314c.cn/44004.Doc
qak.wwwcao1314c.cn/77357.Doc
qak.wwwcao1314c.cn/62642.Doc
qak.wwwcao1314c.cn/40468.Doc
qak.wwwcao1314c.cn/20248.Doc
qak.wwwcao1314c.cn/86048.Doc
qak.wwwcao1314c.cn/44886.Doc
qak.wwwcao1314c.cn/40826.Doc
qak.wwwcao1314c.cn/17755.Doc
qak.wwwcao1314c.cn/00006.Doc
qaj.wwwcao1314c.cn/20468.Doc
qaj.wwwcao1314c.cn/46848.Doc
qaj.wwwcao1314c.cn/62682.Doc
qaj.wwwcao1314c.cn/60442.Doc
qaj.wwwcao1314c.cn/08680.Doc
qaj.wwwcao1314c.cn/28662.Doc
qaj.wwwcao1314c.cn/66622.Doc
qaj.wwwcao1314c.cn/28626.Doc
qaj.wwwcao1314c.cn/97711.Doc
qaj.wwwcao1314c.cn/64280.Doc
qah.wwwcao1314c.cn/04428.Doc
qah.wwwcao1314c.cn/53175.Doc
qah.wwwcao1314c.cn/62404.Doc
qah.wwwcao1314c.cn/04226.Doc
qah.wwwcao1314c.cn/48868.Doc
qah.wwwcao1314c.cn/46648.Doc
qah.wwwcao1314c.cn/68866.Doc
qah.wwwcao1314c.cn/68662.Doc
qah.wwwcao1314c.cn/62066.Doc
qah.wwwcao1314c.cn/62808.Doc
qag.wwwcao1314c.cn/08684.Doc
qag.wwwcao1314c.cn/20022.Doc
qag.wwwcao1314c.cn/44220.Doc
qag.wwwcao1314c.cn/64462.Doc
qag.wwwcao1314c.cn/84444.Doc
qag.wwwcao1314c.cn/08286.Doc
qag.wwwcao1314c.cn/64448.Doc
qag.wwwcao1314c.cn/28406.Doc
qag.wwwcao1314c.cn/42446.Doc
qag.wwwcao1314c.cn/82828.Doc
qaf.wwwcao1314c.cn/42222.Doc
qaf.wwwcao1314c.cn/44086.Doc
qaf.wwwcao1314c.cn/48486.Doc
qaf.wwwcao1314c.cn/22088.Doc
qaf.wwwcao1314c.cn/62846.Doc
qaf.wwwcao1314c.cn/04868.Doc
qaf.wwwcao1314c.cn/44884.Doc
qaf.wwwcao1314c.cn/42848.Doc
qaf.wwwcao1314c.cn/95353.Doc
qaf.wwwcao1314c.cn/20624.Doc
qad.wwwcao1314c.cn/02468.Doc
qad.wwwcao1314c.cn/80460.Doc
qad.wwwcao1314c.cn/86606.Doc
qad.wwwcao1314c.cn/06426.Doc
qad.wwwcao1314c.cn/08262.Doc
qad.wwwcao1314c.cn/13515.Doc
qad.wwwcao1314c.cn/17711.Doc
qad.wwwcao1314c.cn/88688.Doc
qad.wwwcao1314c.cn/44222.Doc
qad.wwwcao1314c.cn/22064.Doc
qas.wwwcao1314c.cn/86084.Doc
qas.wwwcao1314c.cn/84600.Doc
qas.wwwcao1314c.cn/44422.Doc
qas.wwwcao1314c.cn/48248.Doc
qas.wwwcao1314c.cn/48820.Doc
qas.wwwcao1314c.cn/62224.Doc
qas.wwwcao1314c.cn/48440.Doc
qas.wwwcao1314c.cn/04808.Doc
qas.wwwcao1314c.cn/86068.Doc
qas.wwwcao1314c.cn/86824.Doc
qaa.wwwcao1314c.cn/68040.Doc
qaa.wwwcao1314c.cn/68268.Doc
qaa.wwwcao1314c.cn/04042.Doc
qaa.wwwcao1314c.cn/26046.Doc
qaa.wwwcao1314c.cn/88020.Doc
qaa.wwwcao1314c.cn/73317.Doc
qaa.wwwcao1314c.cn/24484.Doc
qaa.wwwcao1314c.cn/82620.Doc
qaa.wwwcao1314c.cn/08240.Doc
qaa.wwwcao1314c.cn/26420.Doc
qap.wwwcao1314c.cn/60888.Doc
qap.wwwcao1314c.cn/17739.Doc
qap.wwwcao1314c.cn/88022.Doc
qap.wwwcao1314c.cn/48606.Doc
qap.wwwcao1314c.cn/04204.Doc
qap.wwwcao1314c.cn/28644.Doc
qap.wwwcao1314c.cn/04466.Doc
qap.wwwcao1314c.cn/73939.Doc
qap.wwwcao1314c.cn/84882.Doc
qap.wwwcao1314c.cn/60840.Doc
qao.wwwcao1314c.cn/46806.Doc
qao.wwwcao1314c.cn/53397.Doc
qao.wwwcao1314c.cn/20028.Doc
qao.wwwcao1314c.cn/26646.Doc
qao.wwwcao1314c.cn/46628.Doc
qao.wwwcao1314c.cn/84600.Doc
qao.wwwcao1314c.cn/20880.Doc
qao.wwwcao1314c.cn/17395.Doc
qao.wwwcao1314c.cn/24288.Doc
qao.wwwcao1314c.cn/46400.Doc
qai.wwwcao1314c.cn/24228.Doc
qai.wwwcao1314c.cn/66828.Doc
qai.wwwcao1314c.cn/02844.Doc
qai.wwwcao1314c.cn/68822.Doc
qai.wwwcao1314c.cn/60626.Doc
qai.wwwcao1314c.cn/48444.Doc
qai.wwwcao1314c.cn/68604.Doc
qai.wwwcao1314c.cn/06644.Doc
qai.wwwcao1314c.cn/64000.Doc
qai.wwwcao1314c.cn/00084.Doc
qau.wwwcao1314c.cn/80082.Doc
qau.wwwcao1314c.cn/20042.Doc
qau.wwwcao1314c.cn/82408.Doc
qau.wwwcao1314c.cn/84464.Doc
qau.wwwcao1314c.cn/44628.Doc
qau.wwwcao1314c.cn/48668.Doc
qau.wwwcao1314c.cn/88464.Doc
qau.wwwcao1314c.cn/80620.Doc
qau.wwwcao1314c.cn/86602.Doc
qau.wwwcao1314c.cn/42448.Doc
qay.wwwcao1314c.cn/82642.Doc
qay.wwwcao1314c.cn/28064.Doc
qay.wwwcao1314c.cn/08442.Doc
qay.wwwcao1314c.cn/79917.Doc
qay.wwwcao1314c.cn/24062.Doc
qay.wwwcao1314c.cn/08866.Doc
qay.wwwcao1314c.cn/79131.Doc
qay.wwwcao1314c.cn/64260.Doc
qay.wwwcao1314c.cn/80004.Doc
qay.wwwcao1314c.cn/48860.Doc
qat.wwwcao1314c.cn/64046.Doc
qat.wwwcao1314c.cn/44284.Doc
qat.wwwcao1314c.cn/40808.Doc
qat.wwwcao1314c.cn/20620.Doc
qat.wwwcao1314c.cn/44026.Doc
qat.wwwcao1314c.cn/44208.Doc
qat.wwwcao1314c.cn/06040.Doc
qat.wwwcao1314c.cn/33173.Doc
qat.wwwcao1314c.cn/20484.Doc
qat.wwwcao1314c.cn/84860.Doc
qar.wwwcao1314c.cn/06248.Doc
qar.wwwcao1314c.cn/00642.Doc
qar.wwwcao1314c.cn/40880.Doc
qar.wwwcao1314c.cn/64802.Doc
qar.wwwcao1314c.cn/60668.Doc
qar.wwwcao1314c.cn/22062.Doc
qar.wwwcao1314c.cn/66420.Doc
qar.wwwcao1314c.cn/02486.Doc
qar.wwwcao1314c.cn/84642.Doc
qar.wwwcao1314c.cn/48840.Doc
qae.wwwcao1314c.cn/84622.Doc
qae.wwwcao1314c.cn/60842.Doc
qae.wwwcao1314c.cn/28040.Doc
qae.wwwcao1314c.cn/20088.Doc
qae.wwwcao1314c.cn/82286.Doc
qae.wwwcao1314c.cn/62824.Doc
qae.wwwcao1314c.cn/64800.Doc
qae.wwwcao1314c.cn/68086.Doc
qae.wwwcao1314c.cn/42420.Doc
qae.wwwcao1314c.cn/42824.Doc
qaw.wwwcao1314c.cn/88666.Doc
qaw.wwwcao1314c.cn/24086.Doc
qaw.wwwcao1314c.cn/60408.Doc
qaw.wwwcao1314c.cn/22682.Doc
qaw.wwwcao1314c.cn/24088.Doc
qaw.wwwcao1314c.cn/37991.Doc
qaw.wwwcao1314c.cn/46620.Doc
qaw.wwwcao1314c.cn/26442.Doc
qaw.wwwcao1314c.cn/22008.Doc
qaw.wwwcao1314c.cn/26684.Doc
qaq.wwwcao1314c.cn/22684.Doc
qaq.wwwcao1314c.cn/64666.Doc
qaq.wwwcao1314c.cn/86084.Doc
qaq.wwwcao1314c.cn/44240.Doc
qaq.wwwcao1314c.cn/62006.Doc
qaq.wwwcao1314c.cn/22604.Doc
qaq.wwwcao1314c.cn/08822.Doc
qaq.wwwcao1314c.cn/04022.Doc
qaq.wwwcao1314c.cn/02682.Doc
qaq.wwwcao1314c.cn/59931.Doc
qpm.wwwcao1314c.cn/82084.Doc
qpm.wwwcao1314c.cn/86444.Doc
qpm.wwwcao1314c.cn/82044.Doc
qpm.wwwcao1314c.cn/62864.Doc
qpm.wwwcao1314c.cn/57951.Doc
qpm.wwwcao1314c.cn/71511.Doc
qpm.wwwcao1314c.cn/48668.Doc
qpm.wwwcao1314c.cn/20842.Doc
qpm.wwwcao1314c.cn/68260.Doc
qpm.wwwcao1314c.cn/80646.Doc
qpn.wwwcao1314c.cn/44048.Doc
qpn.wwwcao1314c.cn/28408.Doc
qpn.wwwcao1314c.cn/86240.Doc
qpn.wwwcao1314c.cn/42488.Doc
qpn.wwwcao1314c.cn/48284.Doc
qpn.wwwcao1314c.cn/02420.Doc
qpn.wwwcao1314c.cn/44848.Doc
qpn.wwwcao1314c.cn/04268.Doc
qpn.wwwcao1314c.cn/68468.Doc
qpn.wwwcao1314c.cn/48246.Doc
qpb.wwwcao1314c.cn/44884.Doc
qpb.wwwcao1314c.cn/20806.Doc
qpb.wwwcao1314c.cn/88826.Doc
qpb.wwwcao1314c.cn/60884.Doc
qpb.wwwcao1314c.cn/48260.Doc
qpb.wwwcao1314c.cn/00602.Doc
qpb.wwwcao1314c.cn/64200.Doc
qpb.wwwcao1314c.cn/88688.Doc
qpb.wwwcao1314c.cn/40220.Doc
qpb.wwwcao1314c.cn/80464.Doc
qpv.wwwcao1314c.cn/60884.Doc
qpv.wwwcao1314c.cn/44026.Doc
qpv.wwwcao1314c.cn/68264.Doc
qpv.wwwcao1314c.cn/88626.Doc
qpv.wwwcao1314c.cn/86604.Doc
qpv.wwwcao1314c.cn/88424.Doc
qpv.wwwcao1314c.cn/62860.Doc
qpv.wwwcao1314c.cn/80622.Doc
qpv.wwwcao1314c.cn/06806.Doc
qpv.wwwcao1314c.cn/02482.Doc
qpc.wwwcao1314c.cn/66084.Doc
qpc.wwwcao1314c.cn/28062.Doc
qpc.wwwcao1314c.cn/88802.Doc
qpc.wwwcao1314c.cn/44204.Doc
qpc.wwwcao1314c.cn/80400.Doc
qpc.wwwcao1314c.cn/24626.Doc
qpc.wwwcao1314c.cn/44280.Doc
qpc.wwwcao1314c.cn/62804.Doc
qpc.wwwcao1314c.cn/08846.Doc
qpc.wwwcao1314c.cn/02402.Doc
qpx.wwwcao1314c.cn/48288.Doc
qpx.wwwcao1314c.cn/66620.Doc
qpx.wwwcao1314c.cn/00286.Doc
qpx.wwwcao1314c.cn/22226.Doc
qpx.wwwcao1314c.cn/04224.Doc
qpx.wwwcao1314c.cn/04880.Doc
qpx.wwwcao1314c.cn/60084.Doc
qpx.wwwcao1314c.cn/60206.Doc
qpx.wwwcao1314c.cn/53717.Doc
qpx.wwwcao1314c.cn/86020.Doc
qpz.wwwcao1314c.cn/48082.Doc
qpz.wwwcao1314c.cn/24248.Doc
qpz.wwwcao1314c.cn/20002.Doc
qpz.wwwcao1314c.cn/06662.Doc
qpz.wwwcao1314c.cn/48846.Doc
qpz.wwwcao1314c.cn/82082.Doc
qpz.wwwcao1314c.cn/02440.Doc
qpz.wwwcao1314c.cn/84828.Doc
qpz.wwwcao1314c.cn/44446.Doc
qpz.wwwcao1314c.cn/22822.Doc
qpl.wwwcao1314c.cn/40624.Doc
qpl.wwwcao1314c.cn/64442.Doc
qpl.wwwcao1314c.cn/84264.Doc
qpl.wwwcao1314c.cn/04020.Doc
qpl.wwwcao1314c.cn/24604.Doc
qpl.wwwcao1314c.cn/46826.Doc
qpl.wwwcao1314c.cn/62200.Doc
qpl.wwwcao1314c.cn/26684.Doc
qpl.wwwcao1314c.cn/44282.Doc
qpl.wwwcao1314c.cn/22882.Doc
qpk.wwwcao1314c.cn/40048.Doc
qpk.wwwcao1314c.cn/26806.Doc
qpk.wwwcao1314c.cn/48020.Doc
qpk.wwwcao1314c.cn/84600.Doc
qpk.wwwcao1314c.cn/22220.Doc
qpk.wwwcao1314c.cn/46682.Doc
qpk.wwwcao1314c.cn/60006.Doc
qpk.wwwcao1314c.cn/42242.Doc
qpk.wwwcao1314c.cn/84044.Doc
qpk.wwwcao1314c.cn/62822.Doc
qpj.wwwcao1314c.cn/84280.Doc
qpj.wwwcao1314c.cn/84468.Doc
qpj.wwwcao1314c.cn/86264.Doc
qpj.wwwcao1314c.cn/80480.Doc
qpj.wwwcao1314c.cn/44606.Doc
qpj.wwwcao1314c.cn/64444.Doc
qpj.wwwcao1314c.cn/46040.Doc
qpj.wwwcao1314c.cn/26646.Doc
qpj.wwwcao1314c.cn/44040.Doc
qpj.wwwcao1314c.cn/24868.Doc
qph.wwwcao1314c.cn/08266.Doc
qph.wwwcao1314c.cn/26042.Doc
qph.wwwcao1314c.cn/42062.Doc
qph.wwwcao1314c.cn/02062.Doc
qph.wwwcao1314c.cn/26220.Doc
qph.wwwcao1314c.cn/88442.Doc
qph.wwwcao1314c.cn/84440.Doc
qph.wwwcao1314c.cn/00446.Doc
qph.wwwcao1314c.cn/42822.Doc
qph.wwwcao1314c.cn/46022.Doc
qpg.wwwcao1314c.cn/82486.Doc
qpg.wwwcao1314c.cn/24400.Doc
qpg.wwwcao1314c.cn/40228.Doc
qpg.wwwcao1314c.cn/40486.Doc
qpg.wwwcao1314c.cn/20602.Doc
qpg.wwwcao1314c.cn/08264.Doc
qpg.wwwcao1314c.cn/20262.Doc
qpg.wwwcao1314c.cn/20080.Doc
qpg.wwwcao1314c.cn/88442.Doc
qpg.wwwcao1314c.cn/82048.Doc
qpf.wwwcao1314c.cn/80440.Doc
qpf.wwwcao1314c.cn/42826.Doc
qpf.wwwcao1314c.cn/26826.Doc
qpf.wwwcao1314c.cn/66840.Doc
qpf.wwwcao1314c.cn/62600.Doc
qpf.wwwcao1314c.cn/84264.Doc
qpf.wwwcao1314c.cn/22280.Doc
qpf.wwwcao1314c.cn/22662.Doc
qpf.wwwcao1314c.cn/22448.Doc
qpf.wwwcao1314c.cn/64602.Doc
qpd.wwwcao1314c.cn/08868.Doc
qpd.wwwcao1314c.cn/42080.Doc
qpd.wwwcao1314c.cn/71599.Doc
qpd.wwwcao1314c.cn/40626.Doc
qpd.wwwcao1314c.cn/06226.Doc
qpd.wwwcao1314c.cn/82460.Doc
qpd.wwwcao1314c.cn/44028.Doc
qpd.wwwcao1314c.cn/15577.Doc
qpd.wwwcao1314c.cn/40026.Doc
qpd.wwwcao1314c.cn/22020.Doc
qps.wwwcao1314c.cn/68642.Doc
qps.wwwcao1314c.cn/00644.Doc
qps.wwwcao1314c.cn/80620.Doc
qps.wwwcao1314c.cn/24406.Doc
qps.wwwcao1314c.cn/95973.Doc
qps.wwwcao1314c.cn/53315.Doc
qps.wwwcao1314c.cn/28226.Doc
qps.wwwcao1314c.cn/04808.Doc
qps.wwwcao1314c.cn/84862.Doc
qps.wwwcao1314c.cn/42820.Doc
qpa.wwwcao1314c.cn/08000.Doc
qpa.wwwcao1314c.cn/84246.Doc
qpa.wwwcao1314c.cn/00602.Doc
qpa.wwwcao1314c.cn/20682.Doc
qpa.wwwcao1314c.cn/86802.Doc
qpa.wwwcao1314c.cn/64884.Doc
qpa.wwwcao1314c.cn/62062.Doc
qpa.wwwcao1314c.cn/02655.Doc
qpa.wwwcao1314c.cn/95939.Doc
qpa.wwwcao1314c.cn/48600.Doc
qpp.wwwcao1314c.cn/28686.Doc
qpp.wwwcao1314c.cn/20084.Doc
qpp.wwwcao1314c.cn/02684.Doc
qpp.wwwcao1314c.cn/64664.Doc
qpp.wwwcao1314c.cn/48282.Doc
qpp.wwwcao1314c.cn/66886.Doc
qpp.wwwcao1314c.cn/44080.Doc
qpp.wwwcao1314c.cn/26406.Doc
qpp.wwwcao1314c.cn/26688.Doc
qpp.wwwcao1314c.cn/64462.Doc
qpo.wwwcao1314c.cn/19977.Doc
qpo.wwwcao1314c.cn/22608.Doc
qpo.wwwcao1314c.cn/44406.Doc
qpo.wwwcao1314c.cn/75371.Doc
qpo.wwwcao1314c.cn/28800.Doc
qpo.wwwcao1314c.cn/86600.Doc
qpo.wwwcao1314c.cn/48848.Doc
qpo.wwwcao1314c.cn/20400.Doc
qpo.wwwcao1314c.cn/00622.Doc
qpo.wwwcao1314c.cn/88628.Doc
qpi.wwwcao1314c.cn/02242.Doc
qpi.wwwcao1314c.cn/40688.Doc
qpi.wwwcao1314c.cn/80640.Doc
qpi.wwwcao1314c.cn/28422.Doc
qpi.wwwcao1314c.cn/95517.Doc
qpi.wwwcao1314c.cn/24866.Doc
qpi.wwwcao1314c.cn/44802.Doc
qpi.wwwcao1314c.cn/68662.Doc
qpi.wwwcao1314c.cn/04668.Doc
qpi.wwwcao1314c.cn/28602.Doc
qpu.wwwcao1314c.cn/86224.Doc
qpu.wwwcao1314c.cn/02088.Doc
qpu.wwwcao1314c.cn/02860.Doc
qpu.wwwcao1314c.cn/84048.Doc
qpu.wwwcao1314c.cn/02604.Doc
qpu.wwwcao1314c.cn/66800.Doc
qpu.wwwcao1314c.cn/28444.Doc
qpu.wwwcao1314c.cn/11735.Doc
qpu.wwwcao1314c.cn/06604.Doc
qpu.wwwcao1314c.cn/84480.Doc
qpy.wwwcao1314c.cn/88066.Doc
qpy.wwwcao1314c.cn/46220.Doc
qpy.wwwcao1314c.cn/46426.Doc
qpy.wwwcao1314c.cn/82288.Doc
qpy.wwwcao1314c.cn/82286.Doc
qpy.wwwcao1314c.cn/97115.Doc
qpy.wwwcao1314c.cn/42284.Doc
qpy.wwwcao1314c.cn/82468.Doc
qpy.wwwcao1314c.cn/28840.Doc
qpy.wwwcao1314c.cn/68066.Doc
qpt.wwwcao1314c.cn/48606.Doc
qpt.wwwcao1314c.cn/44242.Doc
qpt.wwwcao1314c.cn/04082.Doc
qpt.wwwcao1314c.cn/00442.Doc
qpt.wwwcao1314c.cn/62448.Doc
qpt.wwwcao1314c.cn/02840.Doc
qpt.wwwcao1314c.cn/20022.Doc
qpt.wwwcao1314c.cn/84208.Doc
qpt.wwwcao1314c.cn/44266.Doc
qpt.wwwcao1314c.cn/82642.Doc
qpr.wwwcao1314c.cn/20606.Doc
qpr.wwwcao1314c.cn/08222.Doc
qpr.wwwcao1314c.cn/29518.Doc
qpr.wwwcao1314c.cn/77882.Doc
qpr.wwwcao1314c.cn/95092.Doc
qpr.wwwcao1314c.cn/72174.Doc
qpr.wwwcao1314c.cn/14693.Doc
qpr.wwwcao1314c.cn/42975.Doc
qpr.wwwcao1314c.cn/67550.Doc
qpr.wwwcao1314c.cn/94605.Doc
qpe.wwwcao1314c.cn/16594.Doc
qpe.wwwcao1314c.cn/03929.Doc
qpe.wwwcao1314c.cn/30620.Doc
qpe.wwwcao1314c.cn/87074.Doc
qpe.wwwcao1314c.cn/18977.Doc
qpe.wwwcao1314c.cn/66649.Doc
qpe.wwwcao1314c.cn/20970.Doc
qpe.wwwcao1314c.cn/22620.Doc
qpe.wwwcao1314c.cn/28853.Doc
qpe.wwwcao1314c.cn/89187.Doc
qpw.wwwcao1314c.cn/61276.Doc
qpw.wwwcao1314c.cn/12440.Doc
qpw.wwwcao1314c.cn/46461.Doc
qpw.wwwcao1314c.cn/92930.Doc
qpw.wwwcao1314c.cn/40933.Doc
qpw.wwwcao1314c.cn/95603.Doc
qpw.wwwcao1314c.cn/86608.Doc
qpw.wwwcao1314c.cn/84604.Doc
qpw.wwwcao1314c.cn/86628.Doc
qpw.wwwcao1314c.cn/60488.Doc
qpq.wwwcao1314c.cn/04482.Doc
qpq.wwwcao1314c.cn/86246.Doc
qpq.wwwcao1314c.cn/04844.Doc
qpq.wwwcao1314c.cn/86002.Doc
qpq.wwwcao1314c.cn/04282.Doc
qpq.wwwcao1314c.cn/24602.Doc
qpq.wwwcao1314c.cn/22246.Doc
qpq.wwwcao1314c.cn/20468.Doc
qpq.wwwcao1314c.cn/24884.Doc
qpq.wwwcao1314c.cn/86602.Doc
qom.wwwcao1314c.cn/42884.Doc
qom.wwwcao1314c.cn/40440.Doc
qom.wwwcao1314c.cn/88624.Doc
qom.wwwcao1314c.cn/62280.Doc
qom.wwwcao1314c.cn/62664.Doc
qom.wwwcao1314c.cn/79993.Doc
qom.wwwcao1314c.cn/48420.Doc
qom.wwwcao1314c.cn/60048.Doc
qom.wwwcao1314c.cn/24284.Doc
qom.wwwcao1314c.cn/02020.Doc
qon.wwwcao1314c.cn/80284.Doc
qon.wwwcao1314c.cn/46880.Doc
qon.wwwcao1314c.cn/82860.Doc
qon.wwwcao1314c.cn/84024.Doc
qon.wwwcao1314c.cn/08460.Doc
qon.wwwcao1314c.cn/42282.Doc
qon.wwwcao1314c.cn/06028.Doc
qon.wwwcao1314c.cn/84488.Doc
qon.wwwcao1314c.cn/37391.Doc
qon.wwwcao1314c.cn/06604.Doc
