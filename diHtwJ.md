=================================================================
 Python asyncio 子进程管理：异步执行外部命令
=================================================================

asyncio 提供了 create_subprocess_exec 和 create_subprocess_shell
来异步管理子进程，避免阻塞事件循环。本文详解完整用法。

=================================================================
 一、create_subprocess_exec：执行外部程序
=================================================================

import asyncio
import sys

async def run_cmd_simple():
    """
    异步执行一个外部命令，等待其完成。
    create_subprocess_exec 直接执行程序（不经过 shell）。
    """
    print("开始执行 'ls -la' ...")
    # 创建子进程：第一个参数是可执行文件路径，后续是参数
    process = await asyncio.create_subprocess_exec(
        "ls" if sys.platform != "win32" else "dir",  # Linux/macOS 用 ls
        "-la",
        # 默认继承父进程的标准流
        stdout=asyncio.subprocess.PIPE,  # 捕获标准输出
        stderr=asyncio.subprocess.PIPE,  # 捕获标准错误
    )
    # communicate() 等待进程结束并返回 (stdout, stderr)
    stdout, stderr = await process.communicate()
    print(f"返回码: {process.returncode}")
    if stdout:
        print(f"标准输出 ({len(stdout)} 字节):")
        print(stdout.decode(errors="replace")[:500])
    if stderr:
        print(f"标准错误: {stderr.decode(errors='replace')}")

=================================================================
 二、create_subprocess_shell：通过 Shell 执行
=================================================================

async def run_cmd_shell():
    """
    通过系统 shell 执行命令。
    注意：存在 shell 注入风险，不要用于不可信的输入。
    """
    process = await asyncio.create_subprocess_shell(
        "echo 'Hello from asyncio' && sleep 1 && echo 'Done'",
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    stdout, stderr = await process.communicate()
    print(f"Shell 输出: {stdout.decode()}")

# create_subprocess_exec 与 create_subprocess_shell 的区别：
# - exec: 直接执行程序，更安全，不需要 shell 转义
# - shell: 通过 /bin/sh (Windows: cmd.exe) 执行，支持管道、重定向
# - exec 推荐用于所有可以避免使用 shell 的场景

=================================================================
 三、Process 类详解
=================================================================

async def process_class_demo():
    """
    展示 asyncio.subprocess.Process 的完整方法。
    Process 对象通过 create_subprocess_* 返回。
    """
    process = await asyncio.create_subprocess_exec(
        "python3", "--version",
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
        # 工作目录
        cwd="/tmp",
        # 环境变量
        env={"PYTHONUNBUFFERED": "1"},
    )

    # 进程标识
    print(f"PID: {process.pid}")

    # 等待进程结束，获取返回码
    returncode = await process.wait()
    print(f"返回码: {returncode}")

    # 检查进程状态
    if process.returncode == 0:
        print("执行成功")
    else:
        print(f"执行失败，返回码: {process.returncode}")

    # 发送信号（仅 Unix）
    # process.send_signal(signal.SIGTERM)
    # process.terminate()
    # process.kill()

=================================================================
 四、读取子进程输出：逐行读取
=================================================================

async def stream_subprocess_output():
    """
    逐行读取子进程输出，适用于长时间运行的进程。
    使用 stdout.readline() 而不是 communicate()，
    后者会等待进程结束才返回所有数据。
    """
    process = await asyncio.create_subprocess_exec(
        "ping", "-c", "5", "127.0.0.1",
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )

    print("开始逐行读取 ping 输出：")
    # 逐行读取标准输出（实时读取）
    while True:
        line = await process.stdout.readline()
        if not line:
            break  # EOF，进程已结束
        # 解码并去除换行符
        text = line.decode("utf-8", errors="replace").rstrip()
        print(f"[ping] {text}")

    # 确保进程已完全结束
    await process.wait()
    print(f"进程结束，返回码: {process.returncode}")

=================================================================
 五、超时控制
=================================================================

async def subprocess_with_timeout():
    """
    为子进程设置超时。
    使用 asyncio.wait_for 包装 communicate()。
    """
    process = await asyncio.create_subprocess_exec(
        "sleep", "10",  # 持续 10 秒
        stdout=asyncio.subprocess.PIPE,
    )

    try:
        # 设置 3 秒超时
        stdout, stderr = await asyncio.wait_for(
            process.communicate(), timeout=3.0
        )
    except asyncio.TimeoutError:
        print("子进程执行超时，正在终止...")
        process.terminate()  # 发送 SIGTERM
        # 等待进程真正结束
        await process.wait()
        print(f"进程已终止，返回码: {process.returncode}")
    except Exception as e:
        print(f"其他错误: {e}")
        process.kill()
        await process.wait()

=================================================================
 六、并发执行多个子进程
=================================================================

async def run_multiple_subprocesses():
    """
    使用 asyncio.gather 同时执行多个子进程。
    这比串行执行快得多。
    """
    async def run_one(cmd: str, delay: int):
        """运行一个子进程并返回结果"""
        print(f"开始执行: {cmd}")
        process = await asyncio.create_subprocess_shell(
            f"echo 'Starting {cmd}' && sleep {delay} && echo '{cmd} done'",
            stdout=asyncio.subprocess.PIPE,
            shell=True,
        )
        stdout, _ = await process.communicate()
        return stdout.decode().strip()

    # 并发执行 3 个任务
    results = await asyncio.gather(
        run_one("Task-1", 2),
        run_one("Task-2", 1),
        run_one("Task-3", 3),
    )

    for result in results:
        print(f"结果: {result}")

=================================================================
 七、与子进程交互：写入 stdin
=================================================================

async def interact_with_subprocess():
    """
    向子进程的标准输入写入数据，并读取输出。
    适用于交互式程序。
    """
    process = await asyncio.create_subprocess_exec(
        "python3", "-c",
        """
import sys
for line in sys.stdin:
    sys.stdout.write(f"处理: {line.strip().upper()}\\n")
    sys.stdout.flush()
""",
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )

    # 向 stdin 写入数据
    process.stdin.write(b"hello\nworld\nasyncio\n")
    # 务必调用 drain() 刷新缓冲区
    await process.stdin.drain()
    # 关闭 stdin 表示输入结束
    process.stdin.close()

    # 读取所有输出
    stdout, stderr = await process.communicate()
    print("子进程输出:")
    print(stdout.decode())

=================================================================
 八、总结
=================================================================

# 1. create_subprocess_exec 直接执行程序，更安全
# 2. create_subprocess_shell 经过 shell，有注入风险
# 3. communicate() 同时读写 stdin/stdout/stderr 并等待结束
# 4. wait() 仅等待进程结束，获取返回码
# 5. 逐行读取适用于实时流式输出场景
# 6. asyncio.wait_for 可为子进程设置超时
# 7. asyncio.gather 可并发管理多个子进程
# 8. 通过 stdin.write() 和 drain() 向子进程发送数据

if __name__ == "__main__":
    asyncio.run(stream_subprocess_output())

wfw.qjfkLsjdfkLaf.cn/46882.Doc
wfw.qjfkLsjdfkLaf.cn/26840.Doc
wfw.qjfkLsjdfkLaf.cn/86802.Doc
wfw.qjfkLsjdfkLaf.cn/20626.Doc
wfw.qjfkLsjdfkLaf.cn/88202.Doc
wfw.qjfkLsjdfkLaf.cn/02862.Doc
wfw.qjfkLsjdfkLaf.cn/68840.Doc
wfw.qjfkLsjdfkLaf.cn/60004.Doc
wfw.qjfkLsjdfkLaf.cn/88624.Doc
wfw.qjfkLsjdfkLaf.cn/88604.Doc
wfq.qjfkLsjdfkLaf.cn/68600.Doc
wfq.qjfkLsjdfkLaf.cn/06482.Doc
wfq.qjfkLsjdfkLaf.cn/02080.Doc
wfq.qjfkLsjdfkLaf.cn/44828.Doc
wfq.qjfkLsjdfkLaf.cn/42022.Doc
wfq.qjfkLsjdfkLaf.cn/00080.Doc
wfq.qjfkLsjdfkLaf.cn/24262.Doc
wfq.qjfkLsjdfkLaf.cn/04408.Doc
wfq.qjfkLsjdfkLaf.cn/86420.Doc
wfq.qjfkLsjdfkLaf.cn/46884.Doc
wdm.qjfkLsjdfkLaf.cn/51317.Doc
wdm.qjfkLsjdfkLaf.cn/31153.Doc
wdm.qjfkLsjdfkLaf.cn/66484.Doc
wdm.qjfkLsjdfkLaf.cn/48688.Doc
wdm.qjfkLsjdfkLaf.cn/20008.Doc
wdm.qjfkLsjdfkLaf.cn/24686.Doc
wdm.qjfkLsjdfkLaf.cn/46806.Doc
wdm.qjfkLsjdfkLaf.cn/11397.Doc
wdm.qjfkLsjdfkLaf.cn/04246.Doc
wdm.qjfkLsjdfkLaf.cn/66224.Doc
wdn.qjfkLsjdfkLaf.cn/42482.Doc
wdn.qjfkLsjdfkLaf.cn/82404.Doc
wdn.qjfkLsjdfkLaf.cn/08480.Doc
wdn.qjfkLsjdfkLaf.cn/06880.Doc
wdn.qjfkLsjdfkLaf.cn/48642.Doc
wdn.qjfkLsjdfkLaf.cn/84268.Doc
wdn.qjfkLsjdfkLaf.cn/40800.Doc
wdn.qjfkLsjdfkLaf.cn/06684.Doc
wdn.qjfkLsjdfkLaf.cn/00680.Doc
wdn.qjfkLsjdfkLaf.cn/04604.Doc
wdb.qjfkLsjdfkLaf.cn/22406.Doc
wdb.qjfkLsjdfkLaf.cn/33359.Doc
wdb.qjfkLsjdfkLaf.cn/44044.Doc
wdb.qjfkLsjdfkLaf.cn/57399.Doc
wdb.qjfkLsjdfkLaf.cn/48822.Doc
wdb.qjfkLsjdfkLaf.cn/00466.Doc
wdb.qjfkLsjdfkLaf.cn/22422.Doc
wdb.qjfkLsjdfkLaf.cn/66846.Doc
wdb.qjfkLsjdfkLaf.cn/19999.Doc
wdb.qjfkLsjdfkLaf.cn/86846.Doc
wdv.qjfkLsjdfkLaf.cn/00028.Doc
wdv.qjfkLsjdfkLaf.cn/22602.Doc
wdv.qjfkLsjdfkLaf.cn/15379.Doc
wdv.qjfkLsjdfkLaf.cn/80880.Doc
wdv.qjfkLsjdfkLaf.cn/04286.Doc
wdv.qjfkLsjdfkLaf.cn/00844.Doc
wdv.qjfkLsjdfkLaf.cn/02404.Doc
wdv.qjfkLsjdfkLaf.cn/60086.Doc
wdv.qjfkLsjdfkLaf.cn/82026.Doc
wdv.qjfkLsjdfkLaf.cn/04208.Doc
wdc.qjfkLsjdfkLaf.cn/66086.Doc
wdc.qjfkLsjdfkLaf.cn/24082.Doc
wdc.qjfkLsjdfkLaf.cn/88826.Doc
wdc.qjfkLsjdfkLaf.cn/60480.Doc
wdc.qjfkLsjdfkLaf.cn/82482.Doc
wdc.qjfkLsjdfkLaf.cn/42882.Doc
wdc.qjfkLsjdfkLaf.cn/82248.Doc
wdc.qjfkLsjdfkLaf.cn/22060.Doc
wdc.qjfkLsjdfkLaf.cn/40828.Doc
wdc.qjfkLsjdfkLaf.cn/62400.Doc
wdx.qjfkLsjdfkLaf.cn/24826.Doc
wdx.qjfkLsjdfkLaf.cn/06680.Doc
wdx.qjfkLsjdfkLaf.cn/64804.Doc
wdx.qjfkLsjdfkLaf.cn/02620.Doc
wdx.qjfkLsjdfkLaf.cn/24482.Doc
wdx.qjfkLsjdfkLaf.cn/93937.Doc
wdx.qjfkLsjdfkLaf.cn/86842.Doc
wdx.qjfkLsjdfkLaf.cn/64640.Doc
wdx.qjfkLsjdfkLaf.cn/68242.Doc
wdx.qjfkLsjdfkLaf.cn/64208.Doc
wdz.qjfkLsjdfkLaf.cn/00402.Doc
wdz.qjfkLsjdfkLaf.cn/60284.Doc
wdz.qjfkLsjdfkLaf.cn/60802.Doc
wdz.qjfkLsjdfkLaf.cn/04024.Doc
wdz.qjfkLsjdfkLaf.cn/06882.Doc
wdz.qjfkLsjdfkLaf.cn/24802.Doc
wdz.qjfkLsjdfkLaf.cn/68440.Doc
wdz.qjfkLsjdfkLaf.cn/80240.Doc
wdz.qjfkLsjdfkLaf.cn/75537.Doc
wdz.qjfkLsjdfkLaf.cn/24020.Doc
wdl.qjfkLsjdfkLaf.cn/22288.Doc
wdl.qjfkLsjdfkLaf.cn/44884.Doc
wdl.qjfkLsjdfkLaf.cn/82684.Doc
wdl.qjfkLsjdfkLaf.cn/62042.Doc
wdl.qjfkLsjdfkLaf.cn/82008.Doc
wdl.qjfkLsjdfkLaf.cn/40202.Doc
wdl.qjfkLsjdfkLaf.cn/20444.Doc
wdl.qjfkLsjdfkLaf.cn/84464.Doc
wdl.qjfkLsjdfkLaf.cn/00848.Doc
wdl.qjfkLsjdfkLaf.cn/62408.Doc
wdk.qjfkLsjdfkLaf.cn/84864.Doc
wdk.qjfkLsjdfkLaf.cn/46420.Doc
wdk.qjfkLsjdfkLaf.cn/93331.Doc
wdk.qjfkLsjdfkLaf.cn/68464.Doc
wdk.qjfkLsjdfkLaf.cn/28626.Doc
wdk.qjfkLsjdfkLaf.cn/00688.Doc
wdk.qjfkLsjdfkLaf.cn/68640.Doc
wdk.qjfkLsjdfkLaf.cn/62444.Doc
wdk.qjfkLsjdfkLaf.cn/42040.Doc
wdk.qjfkLsjdfkLaf.cn/48886.Doc
wdj.qjfkLsjdfkLaf.cn/40280.Doc
wdj.qjfkLsjdfkLaf.cn/15913.Doc
wdj.qjfkLsjdfkLaf.cn/88426.Doc
wdj.qjfkLsjdfkLaf.cn/68222.Doc
wdj.qjfkLsjdfkLaf.cn/86806.Doc
wdj.qjfkLsjdfkLaf.cn/84226.Doc
wdj.qjfkLsjdfkLaf.cn/68844.Doc
wdj.qjfkLsjdfkLaf.cn/55311.Doc
wdj.qjfkLsjdfkLaf.cn/88064.Doc
wdj.qjfkLsjdfkLaf.cn/66066.Doc
wdh.qjfkLsjdfkLaf.cn/80280.Doc
wdh.qjfkLsjdfkLaf.cn/57531.Doc
wdh.qjfkLsjdfkLaf.cn/48280.Doc
wdh.qjfkLsjdfkLaf.cn/60420.Doc
wdh.qjfkLsjdfkLaf.cn/88448.Doc
wdh.qjfkLsjdfkLaf.cn/66860.Doc
wdh.qjfkLsjdfkLaf.cn/24822.Doc
wdh.qjfkLsjdfkLaf.cn/40680.Doc
wdh.qjfkLsjdfkLaf.cn/48622.Doc
wdh.qjfkLsjdfkLaf.cn/84682.Doc
wdg.qjfkLsjdfkLaf.cn/88482.Doc
wdg.qjfkLsjdfkLaf.cn/73959.Doc
wdg.qjfkLsjdfkLaf.cn/20280.Doc
wdg.qjfkLsjdfkLaf.cn/06428.Doc
wdg.qjfkLsjdfkLaf.cn/28802.Doc
wdg.qjfkLsjdfkLaf.cn/44460.Doc
wdg.qjfkLsjdfkLaf.cn/46060.Doc
wdg.qjfkLsjdfkLaf.cn/00668.Doc
wdg.qjfkLsjdfkLaf.cn/42886.Doc
wdg.qjfkLsjdfkLaf.cn/42866.Doc
wdf.qjfkLsjdfkLaf.cn/26426.Doc
wdf.qjfkLsjdfkLaf.cn/80402.Doc
wdf.qjfkLsjdfkLaf.cn/84084.Doc
wdf.qjfkLsjdfkLaf.cn/44060.Doc
wdf.qjfkLsjdfkLaf.cn/82048.Doc
wdf.qjfkLsjdfkLaf.cn/53117.Doc
wdf.qjfkLsjdfkLaf.cn/06026.Doc
wdf.qjfkLsjdfkLaf.cn/80244.Doc
wdf.qjfkLsjdfkLaf.cn/68246.Doc
wdf.qjfkLsjdfkLaf.cn/60208.Doc
wdd.qjfkLsjdfkLaf.cn/88860.Doc
wdd.qjfkLsjdfkLaf.cn/88080.Doc
wdd.qjfkLsjdfkLaf.cn/62448.Doc
wdd.qjfkLsjdfkLaf.cn/62024.Doc
wdd.qjfkLsjdfkLaf.cn/28028.Doc
wdd.qjfkLsjdfkLaf.cn/71591.Doc
wdd.qjfkLsjdfkLaf.cn/46884.Doc
wdd.qjfkLsjdfkLaf.cn/20684.Doc
wdd.qjfkLsjdfkLaf.cn/06402.Doc
wdd.qjfkLsjdfkLaf.cn/82644.Doc
wds.qjfkLsjdfkLaf.cn/28604.Doc
wds.qjfkLsjdfkLaf.cn/02062.Doc
wds.qjfkLsjdfkLaf.cn/22688.Doc
wds.qjfkLsjdfkLaf.cn/42844.Doc
wds.qjfkLsjdfkLaf.cn/28080.Doc
wds.qjfkLsjdfkLaf.cn/00420.Doc
wds.qjfkLsjdfkLaf.cn/66822.Doc
wds.qjfkLsjdfkLaf.cn/95175.Doc
wds.qjfkLsjdfkLaf.cn/88228.Doc
wds.qjfkLsjdfkLaf.cn/93717.Doc
wda.qjfkLsjdfkLaf.cn/28064.Doc
wda.qjfkLsjdfkLaf.cn/44248.Doc
wda.qjfkLsjdfkLaf.cn/86240.Doc
wda.qjfkLsjdfkLaf.cn/48842.Doc
wda.qjfkLsjdfkLaf.cn/20880.Doc
wda.qjfkLsjdfkLaf.cn/40844.Doc
wda.qjfkLsjdfkLaf.cn/82088.Doc
wda.qjfkLsjdfkLaf.cn/66800.Doc
wda.qjfkLsjdfkLaf.cn/31593.Doc
wda.qjfkLsjdfkLaf.cn/84048.Doc
wdp.qjfkLsjdfkLaf.cn/40004.Doc
wdp.qjfkLsjdfkLaf.cn/60040.Doc
wdp.qjfkLsjdfkLaf.cn/26444.Doc
wdp.qjfkLsjdfkLaf.cn/64020.Doc
wdp.qjfkLsjdfkLaf.cn/86242.Doc
wdp.qjfkLsjdfkLaf.cn/82022.Doc
wdp.qjfkLsjdfkLaf.cn/64624.Doc
wdp.qjfkLsjdfkLaf.cn/37931.Doc
wdp.qjfkLsjdfkLaf.cn/68066.Doc
wdp.qjfkLsjdfkLaf.cn/39111.Doc
wdo.qjfkLsjdfkLaf.cn/04688.Doc
wdo.qjfkLsjdfkLaf.cn/80288.Doc
wdo.qjfkLsjdfkLaf.cn/80646.Doc
wdo.qjfkLsjdfkLaf.cn/26266.Doc
wdo.qjfkLsjdfkLaf.cn/22024.Doc
wdo.qjfkLsjdfkLaf.cn/08400.Doc
wdo.qjfkLsjdfkLaf.cn/35353.Doc
wdo.qjfkLsjdfkLaf.cn/02420.Doc
wdo.qjfkLsjdfkLaf.cn/88406.Doc
wdo.qjfkLsjdfkLaf.cn/48846.Doc
wdi.qjfkLsjdfkLaf.cn/22228.Doc
wdi.qjfkLsjdfkLaf.cn/64266.Doc
wdi.qjfkLsjdfkLaf.cn/04820.Doc
wdi.qjfkLsjdfkLaf.cn/46064.Doc
wdi.qjfkLsjdfkLaf.cn/82820.Doc
wdi.qjfkLsjdfkLaf.cn/88022.Doc
wdi.qjfkLsjdfkLaf.cn/04226.Doc
wdi.qjfkLsjdfkLaf.cn/20208.Doc
wdi.qjfkLsjdfkLaf.cn/26288.Doc
wdi.qjfkLsjdfkLaf.cn/71375.Doc
wdu.qjfkLsjdfkLaf.cn/55793.Doc
wdu.qjfkLsjdfkLaf.cn/26442.Doc
wdu.qjfkLsjdfkLaf.cn/51335.Doc
wdu.qjfkLsjdfkLaf.cn/62042.Doc
wdu.qjfkLsjdfkLaf.cn/86882.Doc
wdu.qjfkLsjdfkLaf.cn/68202.Doc
wdu.qjfkLsjdfkLaf.cn/68048.Doc
wdu.qjfkLsjdfkLaf.cn/82262.Doc
wdu.qjfkLsjdfkLaf.cn/08620.Doc
wdu.qjfkLsjdfkLaf.cn/08860.Doc
wdy.qjfkLsjdfkLaf.cn/88480.Doc
wdy.qjfkLsjdfkLaf.cn/20008.Doc
wdy.qjfkLsjdfkLaf.cn/02040.Doc
wdy.qjfkLsjdfkLaf.cn/48808.Doc
wdy.qjfkLsjdfkLaf.cn/02040.Doc
wdy.qjfkLsjdfkLaf.cn/22880.Doc
wdy.qjfkLsjdfkLaf.cn/24642.Doc
wdy.qjfkLsjdfkLaf.cn/08008.Doc
wdy.qjfkLsjdfkLaf.cn/86428.Doc
wdy.qjfkLsjdfkLaf.cn/00260.Doc
wdt.qjfkLsjdfkLaf.cn/64222.Doc
wdt.qjfkLsjdfkLaf.cn/44666.Doc
wdt.qjfkLsjdfkLaf.cn/60464.Doc
wdt.qjfkLsjdfkLaf.cn/62024.Doc
wdt.qjfkLsjdfkLaf.cn/22862.Doc
wdt.qjfkLsjdfkLaf.cn/95111.Doc
wdt.qjfkLsjdfkLaf.cn/24282.Doc
wdt.qjfkLsjdfkLaf.cn/24288.Doc
wdt.qjfkLsjdfkLaf.cn/26422.Doc
wdt.qjfkLsjdfkLaf.cn/06864.Doc
wdr.qjfkLsjdfkLaf.cn/22466.Doc
wdr.qjfkLsjdfkLaf.cn/02460.Doc
wdr.qjfkLsjdfkLaf.cn/46424.Doc
wdr.qjfkLsjdfkLaf.cn/62466.Doc
wdr.qjfkLsjdfkLaf.cn/79955.Doc
wdr.qjfkLsjdfkLaf.cn/42420.Doc
wdr.qjfkLsjdfkLaf.cn/02224.Doc
wdr.qjfkLsjdfkLaf.cn/00860.Doc
wdr.qjfkLsjdfkLaf.cn/22480.Doc
wdr.qjfkLsjdfkLaf.cn/24206.Doc
wde.qjfkLsjdfkLaf.cn/44888.Doc
wde.qjfkLsjdfkLaf.cn/62062.Doc
wde.qjfkLsjdfkLaf.cn/42804.Doc
wde.qjfkLsjdfkLaf.cn/82026.Doc
wde.qjfkLsjdfkLaf.cn/02820.Doc
wde.qjfkLsjdfkLaf.cn/77971.Doc
wde.qjfkLsjdfkLaf.cn/68266.Doc
wde.qjfkLsjdfkLaf.cn/59577.Doc
wde.qjfkLsjdfkLaf.cn/08220.Doc
wde.qjfkLsjdfkLaf.cn/28448.Doc
wdw.qjfkLsjdfkLaf.cn/86044.Doc
wdw.qjfkLsjdfkLaf.cn/20664.Doc
wdw.qjfkLsjdfkLaf.cn/80066.Doc
wdw.qjfkLsjdfkLaf.cn/80466.Doc
wdw.qjfkLsjdfkLaf.cn/28604.Doc
wdw.qjfkLsjdfkLaf.cn/60602.Doc
wdw.qjfkLsjdfkLaf.cn/59535.Doc
wdw.qjfkLsjdfkLaf.cn/64048.Doc
wdw.qjfkLsjdfkLaf.cn/80820.Doc
wdw.qjfkLsjdfkLaf.cn/24286.Doc
wdq.qjfkLsjdfkLaf.cn/20620.Doc
wdq.qjfkLsjdfkLaf.cn/48886.Doc
wdq.qjfkLsjdfkLaf.cn/08428.Doc
wdq.qjfkLsjdfkLaf.cn/08280.Doc
wdq.qjfkLsjdfkLaf.cn/82208.Doc
wdq.qjfkLsjdfkLaf.cn/53159.Doc
wdq.qjfkLsjdfkLaf.cn/44068.Doc
wdq.qjfkLsjdfkLaf.cn/24866.Doc
wdq.qjfkLsjdfkLaf.cn/28488.Doc
wdq.qjfkLsjdfkLaf.cn/46284.Doc
wsm.qjfkLsjdfkLaf.cn/66688.Doc
wsm.qjfkLsjdfkLaf.cn/48066.Doc
wsm.qjfkLsjdfkLaf.cn/04860.Doc
wsm.qjfkLsjdfkLaf.cn/80646.Doc
wsm.qjfkLsjdfkLaf.cn/02868.Doc
wsm.qjfkLsjdfkLaf.cn/26826.Doc
wsm.qjfkLsjdfkLaf.cn/64886.Doc
wsm.qjfkLsjdfkLaf.cn/39591.Doc
wsm.qjfkLsjdfkLaf.cn/04280.Doc
wsm.qjfkLsjdfkLaf.cn/66882.Doc
wsn.qjfkLsjdfkLaf.cn/20024.Doc
wsn.qjfkLsjdfkLaf.cn/64602.Doc
wsn.qjfkLsjdfkLaf.cn/24846.Doc
wsn.qjfkLsjdfkLaf.cn/04868.Doc
wsn.qjfkLsjdfkLaf.cn/00662.Doc
wsn.qjfkLsjdfkLaf.cn/24824.Doc
wsn.qjfkLsjdfkLaf.cn/51551.Doc
wsn.qjfkLsjdfkLaf.cn/24822.Doc
wsn.qjfkLsjdfkLaf.cn/68820.Doc
wsn.qjfkLsjdfkLaf.cn/40640.Doc
wsb.qjfkLsjdfkLaf.cn/66208.Doc
wsb.qjfkLsjdfkLaf.cn/62060.Doc
wsb.qjfkLsjdfkLaf.cn/26824.Doc
wsb.qjfkLsjdfkLaf.cn/24488.Doc
wsb.qjfkLsjdfkLaf.cn/97555.Doc
wsb.qjfkLsjdfkLaf.cn/88002.Doc
wsb.qjfkLsjdfkLaf.cn/73513.Doc
wsb.qjfkLsjdfkLaf.cn/46460.Doc
wsb.qjfkLsjdfkLaf.cn/62244.Doc
wsb.qjfkLsjdfkLaf.cn/88408.Doc
wsv.qjfkLsjdfkLaf.cn/86442.Doc
wsv.qjfkLsjdfkLaf.cn/06288.Doc
wsv.qjfkLsjdfkLaf.cn/02248.Doc
wsv.qjfkLsjdfkLaf.cn/48222.Doc
wsv.qjfkLsjdfkLaf.cn/00626.Doc
wsv.qjfkLsjdfkLaf.cn/77177.Doc
wsv.qjfkLsjdfkLaf.cn/86644.Doc
wsv.qjfkLsjdfkLaf.cn/48084.Doc
wsv.qjfkLsjdfkLaf.cn/08666.Doc
wsv.qjfkLsjdfkLaf.cn/73951.Doc
wsc.qjfkLsjdfkLaf.cn/84804.Doc
wsc.qjfkLsjdfkLaf.cn/62626.Doc
wsc.qjfkLsjdfkLaf.cn/24204.Doc
wsc.qjfkLsjdfkLaf.cn/80206.Doc
wsc.qjfkLsjdfkLaf.cn/88024.Doc
wsc.qjfkLsjdfkLaf.cn/40624.Doc
wsc.qjfkLsjdfkLaf.cn/06000.Doc
wsc.qjfkLsjdfkLaf.cn/26228.Doc
wsc.qjfkLsjdfkLaf.cn/26620.Doc
wsc.qjfkLsjdfkLaf.cn/06226.Doc
wsx.qjfkLsjdfkLaf.cn/84448.Doc
wsx.qjfkLsjdfkLaf.cn/66242.Doc
wsx.qjfkLsjdfkLaf.cn/66026.Doc
wsx.qjfkLsjdfkLaf.cn/64286.Doc
wsx.qjfkLsjdfkLaf.cn/20062.Doc
wsx.qjfkLsjdfkLaf.cn/08404.Doc
wsx.qjfkLsjdfkLaf.cn/20482.Doc
wsx.qjfkLsjdfkLaf.cn/22648.Doc
wsx.qjfkLsjdfkLaf.cn/82288.Doc
wsx.qjfkLsjdfkLaf.cn/40428.Doc
wsz.qjfkLsjdfkLaf.cn/04628.Doc
wsz.qjfkLsjdfkLaf.cn/88060.Doc
wsz.qjfkLsjdfkLaf.cn/51353.Doc
wsz.qjfkLsjdfkLaf.cn/00006.Doc
wsz.qjfkLsjdfkLaf.cn/62440.Doc
wsz.qjfkLsjdfkLaf.cn/22086.Doc
wsz.qjfkLsjdfkLaf.cn/60424.Doc
wsz.qjfkLsjdfkLaf.cn/86220.Doc
wsz.qjfkLsjdfkLaf.cn/86202.Doc
wsz.qjfkLsjdfkLaf.cn/84622.Doc
wsl.qjfkLsjdfkLaf.cn/64022.Doc
wsl.qjfkLsjdfkLaf.cn/82040.Doc
wsl.qjfkLsjdfkLaf.cn/66064.Doc
wsl.qjfkLsjdfkLaf.cn/42840.Doc
wsl.qjfkLsjdfkLaf.cn/04682.Doc
wsl.qjfkLsjdfkLaf.cn/86004.Doc
wsl.qjfkLsjdfkLaf.cn/40086.Doc
wsl.qjfkLsjdfkLaf.cn/44622.Doc
wsl.qjfkLsjdfkLaf.cn/66208.Doc
wsl.qjfkLsjdfkLaf.cn/28284.Doc
wsk.qjfkLsjdfkLaf.cn/40800.Doc
wsk.qjfkLsjdfkLaf.cn/00884.Doc
wsk.qjfkLsjdfkLaf.cn/79599.Doc
wsk.qjfkLsjdfkLaf.cn/00028.Doc
wsk.qjfkLsjdfkLaf.cn/28426.Doc
wsk.qjfkLsjdfkLaf.cn/40066.Doc
wsk.qjfkLsjdfkLaf.cn/86420.Doc
wsk.qjfkLsjdfkLaf.cn/60482.Doc
wsk.qjfkLsjdfkLaf.cn/06682.Doc
wsk.qjfkLsjdfkLaf.cn/55355.Doc
wsj.qjfkLsjdfkLaf.cn/95151.Doc
wsj.qjfkLsjdfkLaf.cn/60442.Doc
wsj.qjfkLsjdfkLaf.cn/40086.Doc
wsj.qjfkLsjdfkLaf.cn/26828.Doc
wsj.qjfkLsjdfkLaf.cn/04848.Doc
wsj.qjfkLsjdfkLaf.cn/33751.Doc
wsj.qjfkLsjdfkLaf.cn/04008.Doc
wsj.qjfkLsjdfkLaf.cn/62244.Doc
wsj.qjfkLsjdfkLaf.cn/84444.Doc
wsj.qjfkLsjdfkLaf.cn/33113.Doc
wsh.qjfkLsjdfkLaf.cn/71599.Doc
wsh.qjfkLsjdfkLaf.cn/88602.Doc
wsh.qjfkLsjdfkLaf.cn/06802.Doc
wsh.qjfkLsjdfkLaf.cn/48888.Doc
wsh.qjfkLsjdfkLaf.cn/04088.Doc
wsh.qjfkLsjdfkLaf.cn/04466.Doc
wsh.qjfkLsjdfkLaf.cn/24842.Doc
wsh.qjfkLsjdfkLaf.cn/08288.Doc
wsh.qjfkLsjdfkLaf.cn/62468.Doc
wsh.qjfkLsjdfkLaf.cn/08248.Doc
wsg.qjfkLsjdfkLaf.cn/24082.Doc
wsg.qjfkLsjdfkLaf.cn/88480.Doc
wsg.qjfkLsjdfkLaf.cn/42808.Doc
wsg.qjfkLsjdfkLaf.cn/44420.Doc
wsg.qjfkLsjdfkLaf.cn/62204.Doc
wsg.qjfkLsjdfkLaf.cn/60044.Doc
wsg.qjfkLsjdfkLaf.cn/02800.Doc
wsg.qjfkLsjdfkLaf.cn/02482.Doc
wsg.qjfkLsjdfkLaf.cn/95117.Doc
wsg.qjfkLsjdfkLaf.cn/06244.Doc
wsf.qjfkLsjdfkLaf.cn/11179.Doc
wsf.qjfkLsjdfkLaf.cn/44684.Doc
wsf.qjfkLsjdfkLaf.cn/80224.Doc
wsf.qjfkLsjdfkLaf.cn/15339.Doc
wsf.qjfkLsjdfkLaf.cn/44844.Doc
wsf.qjfkLsjdfkLaf.cn/24246.Doc
wsf.qjfkLsjdfkLaf.cn/26608.Doc
wsf.qjfkLsjdfkLaf.cn/22646.Doc
wsf.qjfkLsjdfkLaf.cn/80284.Doc
wsf.qjfkLsjdfkLaf.cn/60428.Doc
wsd.qjfkLsjdfkLaf.cn/13117.Doc
wsd.qjfkLsjdfkLaf.cn/86826.Doc
wsd.qjfkLsjdfkLaf.cn/68824.Doc
wsd.qjfkLsjdfkLaf.cn/26826.Doc
wsd.qjfkLsjdfkLaf.cn/42848.Doc
wsd.qjfkLsjdfkLaf.cn/26246.Doc
wsd.qjfkLsjdfkLaf.cn/00424.Doc
wsd.qjfkLsjdfkLaf.cn/86284.Doc
wsd.qjfkLsjdfkLaf.cn/84486.Doc
wsd.qjfkLsjdfkLaf.cn/82266.Doc
wss.qjfkLsjdfkLaf.cn/31755.Doc
wss.qjfkLsjdfkLaf.cn/02026.Doc
wss.qjfkLsjdfkLaf.cn/51515.Doc
wss.qjfkLsjdfkLaf.cn/48626.Doc
wss.qjfkLsjdfkLaf.cn/68668.Doc
wss.qjfkLsjdfkLaf.cn/33173.Doc
wss.qjfkLsjdfkLaf.cn/88446.Doc
wss.qjfkLsjdfkLaf.cn/44808.Doc
wss.qjfkLsjdfkLaf.cn/44002.Doc
wss.qjfkLsjdfkLaf.cn/20648.Doc
wsa.qjfkLsjdfkLaf.cn/68024.Doc
wsa.qjfkLsjdfkLaf.cn/24284.Doc
wsa.qjfkLsjdfkLaf.cn/80248.Doc
wsa.qjfkLsjdfkLaf.cn/06002.Doc
wsa.qjfkLsjdfkLaf.cn/86840.Doc
wsa.qjfkLsjdfkLaf.cn/02280.Doc
wsa.qjfkLsjdfkLaf.cn/86666.Doc
wsa.qjfkLsjdfkLaf.cn/02084.Doc
wsa.qjfkLsjdfkLaf.cn/28220.Doc
wsa.qjfkLsjdfkLaf.cn/86622.Doc
wsp.qjfkLsjdfkLaf.cn/08002.Doc
wsp.qjfkLsjdfkLaf.cn/08842.Doc
wsp.qjfkLsjdfkLaf.cn/48804.Doc
wsp.qjfkLsjdfkLaf.cn/04284.Doc
wsp.qjfkLsjdfkLaf.cn/82804.Doc
wsp.qjfkLsjdfkLaf.cn/40248.Doc
wsp.qjfkLsjdfkLaf.cn/20082.Doc
wsp.qjfkLsjdfkLaf.cn/00808.Doc
wsp.qjfkLsjdfkLaf.cn/24268.Doc
wsp.qjfkLsjdfkLaf.cn/68046.Doc
wso.qjfkLsjdfkLaf.cn/64000.Doc
wso.qjfkLsjdfkLaf.cn/00246.Doc
wso.qjfkLsjdfkLaf.cn/82824.Doc
wso.qjfkLsjdfkLaf.cn/26446.Doc
wso.qjfkLsjdfkLaf.cn/22408.Doc
wso.qjfkLsjdfkLaf.cn/62660.Doc
wso.qjfkLsjdfkLaf.cn/59575.Doc
wso.qjfkLsjdfkLaf.cn/59711.Doc
wso.qjfkLsjdfkLaf.cn/66424.Doc
wso.qjfkLsjdfkLaf.cn/88680.Doc
wsi.qjfkLsjdfkLaf.cn/08422.Doc
wsi.qjfkLsjdfkLaf.cn/71937.Doc
wsi.qjfkLsjdfkLaf.cn/77337.Doc
wsi.qjfkLsjdfkLaf.cn/06680.Doc
wsi.qjfkLsjdfkLaf.cn/33731.Doc
wsi.qjfkLsjdfkLaf.cn/82008.Doc
wsi.qjfkLsjdfkLaf.cn/20288.Doc
wsi.qjfkLsjdfkLaf.cn/86840.Doc
wsi.qjfkLsjdfkLaf.cn/42486.Doc
wsi.qjfkLsjdfkLaf.cn/82644.Doc
wsu.qjfkLsjdfkLaf.cn/99559.Doc
wsu.qjfkLsjdfkLaf.cn/48284.Doc
wsu.qjfkLsjdfkLaf.cn/86862.Doc
wsu.qjfkLsjdfkLaf.cn/24602.Doc
wsu.qjfkLsjdfkLaf.cn/66668.Doc
wsu.qjfkLsjdfkLaf.cn/44006.Doc
wsu.qjfkLsjdfkLaf.cn/28422.Doc
wsu.qjfkLsjdfkLaf.cn/91513.Doc
wsu.qjfkLsjdfkLaf.cn/68426.Doc
wsu.qjfkLsjdfkLaf.cn/68848.Doc
wsy.qjfkLsjdfkLaf.cn/66820.Doc
wsy.qjfkLsjdfkLaf.cn/97733.Doc
wsy.qjfkLsjdfkLaf.cn/40266.Doc
wsy.qjfkLsjdfkLaf.cn/86666.Doc
wsy.qjfkLsjdfkLaf.cn/60488.Doc
wsy.qjfkLsjdfkLaf.cn/44424.Doc
wsy.qjfkLsjdfkLaf.cn/60666.Doc
wsy.qjfkLsjdfkLaf.cn/64026.Doc
wsy.qjfkLsjdfkLaf.cn/46084.Doc
wsy.qjfkLsjdfkLaf.cn/86224.Doc
