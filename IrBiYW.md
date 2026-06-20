肆绿膳槐冉


========================================================
 Python 进程间通信与消息队列 — 多进程协作详解
========================================================

当程序需要跨进程传输数据或协调工作时，IPC（进程间通信）
就是核心技术。Python 提供了从简单的队列到专业的消息
中间件的多层次方案。

--------------------------------------------------------
1. multiprocessing.Queue — 进程安全队列
--------------------------------------------------------

from multiprocessing import Process, Queue
import time

def worker(q, name):
    """消费者：从队列中获取任务并处理"""
    while True:
        task = q.get()
        if task is None:  # 哨兵值，表示结束
            print(f"[{name}] 收到退出信号")
            break
        print(f"[{name}] 处理任务: {task}")
        time.sleep(0.5)

if __name__ == "__main__":
    task_queue = Queue()

    # 启动两个消费者进程
    workers = []
    for i in range(2):
        p = Process(target=worker, args=(task_queue, f"Worker-{i}"))
        p.start()
        workers.append(p)

    # 生产者：派发 5 个任务
    for i in range(5):
        task_queue.put(f"Task-{i}")

    # 发送退出信号（每个 worker 一个）
    for _ in workers:
        task_queue.put(None)

    for p in workers:
        p.join()

--------------------------------------------------------
2. Pipe — 双向通信管道
--------------------------------------------------------

from multiprocessing import Process, Pipe

def child_process(conn):
    """子进程：通过管道收发消息"""
    data = conn.recv()  # 阻塞等待接收
    print(f"子进程收到: {data}")
    conn.send({"reply": "已收到", "original": data})
    conn.close()

if __name__ == "__main__":
    # Pipe() 返回 (parent_conn, child_conn)
    parent_conn, child_conn = Pipe()

    p = Process(target=child_process, args=(child_conn,))
    p.start()

    # 父进程发送数据
    parent_conn.send({"command": "ping", "seq": 1})

    # 接收子进程回复
    reply = parent_conn.recv()
    print(f"父进程收到回复: {reply}")

    parent_conn.close()
    p.join()

--------------------------------------------------------
3. 共享内存 — Value 与 Array
--------------------------------------------------------

from multiprocessing import Process, Value, Array

def increment(shared_val, shared_arr):
    """子进程修改共享内存中的数据"""
    with shared_val.get_lock():  # 加锁保证原子操作
        shared_val.value += 1

    for i in range(len(shared_arr)):
        shared_arr[i] = shared_arr[i] * 2

if __name__ == "__main__":
    # 创建共享的整型和数组
    counter = Value("i", 0)       # 'i' 表示 signed int
    data = Array("d", [1.0, 2.0, 3.0])  # 'd' 表示 double

    processes = []
    for _ in range(5):
        p = Process(target=increment, args=(counter, data))
        p.start()
        processes.append(p)

    for p in processes:
        p.join()

    print(f"最终计数: {counter.value}")  # 5
    print(f"数组数据: {list(data)}")    # [32.0, 64.0, 128.0]

--------------------------------------------------------
4. Redis Pub/Sub — 发布订阅模式
--------------------------------------------------------

# 安装：pip install redis
import redis
import threading
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def subscriber(channel_name):
    """订阅者：在单独的线程中运行"""
    pubsub = r.pubsub()
    pubsub.subscribe(channel_name)
    print(f"已订阅频道: {channel_name}")
    for message in pubsub.listen():
        if message["type"] == "message":
            print(f"[订阅者] 收到: {message['data']}")
            if message["data"] == "STOP":
                break

def publisher(channel_name):
    """发布者：发送消息"""
    for msg in ["消息1", "消息2", "消息3", "STOP"]:
        time.sleep(1)
        r.publish(channel_name, msg)
        print(f"[发布者] 发送: {msg}")

# 测试发布订阅
# t = threading.Thread(target=subscriber, args=("news",), daemon=True)
# t.start()
# time.sleep(0.5)
# publisher("news")

--------------------------------------------------------
5. Pika / RabbitMQ — 专业消息队列
--------------------------------------------------------

# 安装：pip install pika
import pika
import json

def send_message(queue_name, message):
    """生产者：发送消息到 RabbitMQ"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters("localhost"))
    channel = connection.channel()
    channel.queue_declare(queue=queue_name, durable=True)

    channel.basic_publish(
        exchange="",
        routing_key=queue_name,
        body=json.dumps(message),
        properties=pika.BasicProperties(
            delivery_mode=2,  # 持久化消息
        ))
    connection.close()

def start_consumer(queue_name, callback):
    """消费者：持续消费队列消息"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters("localhost"))
    channel = connection.channel()
    channel.queue_declare(queue=queue_name, durable=True)

    # 每次只处理一条消息，处理完才接受下一条
    channel.basic_qos(prefetch_count=1)
    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback,
        auto_ack=False)

    print(f"等待 {queue_name} 的消息...")
    channel.start_consuming()

def process_task(ch, method, properties, body):
    """处理消息的回调函数"""
    task = json.loads(body)
    print(f"处理任务: {task}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

# send_message("task_queue", {"action": "resize", "size": 1024})
# start_consumer("task_queue", process_task)

--------------------------------------------------------
6. ZeroMQ (zmq) — 多种通信模式
--------------------------------------------------------

# 安装：pip install pyzmq
import zmq
import time

def zmq_server():
    """REP/REQ 模式服务端"""
    context = zmq.Context()
    socket = context.socket(zmq.REP)
    socket.bind("tcp://*:5555")

    while True:
        message = socket.recv_json()
        print(f"服务端收到: {message}")
        socket.send_json({"status": "ok", "echo": message})
        if message.get("cmd") == "exit":
            break

def zmq_client():
    """REP/REQ 模式客户端"""
    context = zmq.Context()
    socket = context.socket(zmq.REQ)
    socket.connect("tcp://localhost:5555")

    socket.send_json({"cmd": "ping", "ts": time.time()})
    reply = socket.recv_json()
    print(f"客户端收到: {reply}")

def zmq_pub_sub():
    """PUB/SUB 模式"""
    context = zmq.Context()

    # 发布者
    pub = context.socket(zmq.PUB)
    pub.bind("tcp://*:5556")
    time.sleep(0.5)  # 等待订阅者连接

    # 订阅者
    sub = context.socket(zmq.SUB)
    sub.connect("tcp://localhost:5556")
    sub.setsockopt_string(zmq.SUBSCRIBE, "news")  # 只订阅 news 前缀

    pub.send_string("news 今日头条")
    msg = sub.recv_string()
    print(f"订阅者收到: {msg}")

--------------------------------------------------------
7. socket — 底层 IPC
--------------------------------------------------------

import socket
import threading

def unix_server(sock_path):
    """Unix 域套接字服务器"""
    server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    server.bind(sock_path)
    server.listen(1)
    conn, _ = server.accept()
    data = conn.recv(1024)
    print(f"Socket 服务端收到: {data.decode()}")
    conn.send(b"ACK")
    conn.close()
    server.close()

# 不同场景选择建议：
# - 单机多进程 -> multiprocessing.Queue 或 Pipe（零依赖）
# - 跨语言/网络 -> RabbitMQ 或 ZeroMQ
# - 简单发布订阅 -> Redis Pub/Sub
# - 高性能低延迟 -> ZeroMQ 或 Unix Socket

缮汤济形躺有厮返秘嘿贤倚倜汤式

qpa.24d38z2.cn/599375.htm
qpa.24d38z2.cn/713975.htm
qpa.24d38z2.cn/379335.htm
qpa.24d38z2.cn/113555.htm
qpa.24d38z2.cn/860625.htm
qpa.24d38z2.cn/913715.htm
qpa.24d38z2.cn/377395.htm
qpa.24d38z2.cn/539115.htm
qpp.24d38z2.cn/535955.htm
qpp.24d38z2.cn/280405.htm
qpp.24d38z2.cn/737795.htm
qpp.24d38z2.cn/375195.htm
qpp.24d38z2.cn/593315.htm
qpp.24d38z2.cn/066605.htm
qpp.24d38z2.cn/024405.htm
qpp.24d38z2.cn/171375.htm
qpp.24d38z2.cn/797375.htm
qpp.24d38z2.cn/751935.htm
qpo.24d38z2.cn/539735.htm
qpo.24d38z2.cn/357315.htm
qpo.24d38z2.cn/755335.htm
qpo.24d38z2.cn/373395.htm
qpo.24d38z2.cn/244645.htm
qpo.24d38z2.cn/551175.htm
qpo.24d38z2.cn/719315.htm
qpo.24d38z2.cn/737375.htm
qpo.24d38z2.cn/193535.htm
qpo.24d38z2.cn/133195.htm
qpi.24d38z2.cn/719375.htm
qpi.24d38z2.cn/260205.htm
qpi.24d38z2.cn/046285.htm
qpi.24d38z2.cn/519955.htm
qpi.24d38z2.cn/933375.htm
qpi.24d38z2.cn/737535.htm
qpi.24d38z2.cn/913155.htm
qpi.24d38z2.cn/555115.htm
qpi.24d38z2.cn/391955.htm
qpi.24d38z2.cn/366865.htm
qpu.24d38z2.cn/951715.htm
qpu.24d38z2.cn/977355.htm
qpu.24d38z2.cn/975195.htm
qpu.24d38z2.cn/800285.htm
qpu.24d38z2.cn/535195.htm
qpu.24d38z2.cn/684025.htm
qpu.24d38z2.cn/155995.htm
qpu.24d38z2.cn/397775.htm
qpu.24d38z2.cn/991975.htm
qpu.24d38z2.cn/513955.htm
qpy.24d38z2.cn/917155.htm
qpy.24d38z2.cn/399155.htm
qpy.24d38z2.cn/193995.htm
qpy.24d38z2.cn/999375.htm
qpy.24d38z2.cn/775175.htm
qpy.24d38z2.cn/553195.htm
qpy.24d38z2.cn/991515.htm
qpy.24d38z2.cn/957595.htm
qpy.24d38z2.cn/751775.htm
qpy.24d38z2.cn/131535.htm
qpt.24d38z2.cn/553355.htm
qpt.24d38z2.cn/577915.htm
qpt.24d38z2.cn/919315.htm
qpt.24d38z2.cn/397155.htm
qpt.24d38z2.cn/991155.htm
qpt.24d38z2.cn/608205.htm
qpt.24d38z2.cn/711955.htm
qpt.24d38z2.cn/359995.htm
qpt.24d38z2.cn/662665.htm
qpt.24d38z2.cn/280285.htm
qpr.24d38z2.cn/842805.htm
qpr.24d38z2.cn/280445.htm
qpr.24d38z2.cn/668045.htm
qpr.24d38z2.cn/197175.htm
qpr.24d38z2.cn/975135.htm
qpr.24d38z2.cn/680865.htm
qpr.24d38z2.cn/111315.htm
qpr.24d38z2.cn/488805.htm
qpr.24d38z2.cn/115995.htm
qpr.24d38z2.cn/280425.htm
qpe.24d38z2.cn/482865.htm
qpe.24d38z2.cn/195995.htm
qpe.24d38z2.cn/911775.htm
qpe.24d38z2.cn/002465.htm
qpe.24d38z2.cn/395995.htm
qpe.24d38z2.cn/713995.htm
qpe.24d38z2.cn/286005.htm
qpe.24d38z2.cn/602825.htm
qpe.24d38z2.cn/395155.htm
qpe.24d38z2.cn/048205.htm
qpw.24d38z2.cn/533995.htm
qpw.24d38z2.cn/860205.htm
qpw.24d38z2.cn/482405.htm
qpw.24d38z2.cn/664225.htm
qpw.24d38z2.cn/202685.htm
qpw.24d38z2.cn/195355.htm
qpw.24d38z2.cn/408465.htm
qpw.24d38z2.cn/024445.htm
qpw.24d38z2.cn/713575.htm
qpw.24d38z2.cn/935735.htm
qpq.24d38z2.cn/151335.htm
qpq.24d38z2.cn/797195.htm
qpq.24d38z2.cn/337355.htm
qpq.24d38z2.cn/359355.htm
qpq.24d38z2.cn/575955.htm
qpq.24d38z2.cn/559375.htm
qpq.24d38z2.cn/575535.htm
qpq.24d38z2.cn/195735.htm
qpq.24d38z2.cn/539135.htm
qpq.24d38z2.cn/935395.htm
qotv.24d38z2.cn/199175.htm
qotv.24d38z2.cn/915195.htm
qotv.24d38z2.cn/399175.htm
qotv.24d38z2.cn/333995.htm
qotv.24d38z2.cn/779735.htm
qotv.24d38z2.cn/537935.htm
qotv.24d38z2.cn/666005.htm
qotv.24d38z2.cn/559335.htm
qotv.24d38z2.cn/444025.htm
qotv.24d38z2.cn/913115.htm
qon.24d38z2.cn/377555.htm
qon.24d38z2.cn/917915.htm
qon.24d38z2.cn/753115.htm
qon.24d38z2.cn/177975.htm
qon.24d38z2.cn/115115.htm
qon.24d38z2.cn/971175.htm
qon.24d38z2.cn/995715.htm
qon.24d38z2.cn/339975.htm
qon.24d38z2.cn/333515.htm
qon.24d38z2.cn/579355.htm
qob.24d38z2.cn/735715.htm
qob.24d38z2.cn/804485.htm
qob.24d38z2.cn/357995.htm
qob.24d38z2.cn/799955.htm
qob.24d38z2.cn/377355.htm
qob.24d38z2.cn/111935.htm
qob.24d38z2.cn/957555.htm
qob.24d38z2.cn/775195.htm
qob.24d38z2.cn/591775.htm
qob.24d38z2.cn/913995.htm
qov.24d38z2.cn/193135.htm
qov.24d38z2.cn/395355.htm
qov.24d38z2.cn/915175.htm
qov.24d38z2.cn/195195.htm
qov.24d38z2.cn/173955.htm
qov.24d38z2.cn/739195.htm
qov.24d38z2.cn/951555.htm
qov.24d38z2.cn/753395.htm
qov.24d38z2.cn/935795.htm
qov.24d38z2.cn/735335.htm
qoc.24d38z2.cn/735135.htm
qoc.24d38z2.cn/519515.htm
qoc.24d38z2.cn/351995.htm
qoc.24d38z2.cn/953755.htm
qoc.24d38z2.cn/371755.htm
qoc.24d38z2.cn/606885.htm
qoc.24d38z2.cn/731795.htm
qoc.24d38z2.cn/975175.htm
qoc.24d38z2.cn/119975.htm
qoc.24d38z2.cn/351995.htm
qox.24d38z2.cn/680025.htm
qox.24d38z2.cn/917355.htm
qox.24d38z2.cn/537135.htm
qox.24d38z2.cn/088265.htm
qox.24d38z2.cn/315375.htm
qox.24d38z2.cn/717375.htm
qox.24d38z2.cn/599575.htm
qox.24d38z2.cn/911535.htm
qox.24d38z2.cn/917575.htm
qox.24d38z2.cn/395755.htm
qoz.24d38z2.cn/117795.htm
qoz.24d38z2.cn/199715.htm
qoz.24d38z2.cn/006625.htm
qoz.24d38z2.cn/953995.htm
qoz.24d38z2.cn/593775.htm
qoz.24d38z2.cn/753935.htm
qoz.24d38z2.cn/175995.htm
qoz.24d38z2.cn/719135.htm
qoz.24d38z2.cn/139595.htm
qoz.24d38z2.cn/915975.htm
qol.24d38z2.cn/971575.htm
qol.24d38z2.cn/377935.htm
qol.24d38z2.cn/319155.htm
qol.24d38z2.cn/577915.htm
qol.24d38z2.cn/571315.htm
qol.24d38z2.cn/391535.htm
qol.24d38z2.cn/955935.htm
qol.24d38z2.cn/371975.htm
qol.24d38z2.cn/159155.htm
qol.24d38z2.cn/517315.htm
qok.24d38z2.cn/773335.htm
qok.24d38z2.cn/311135.htm
qok.24d38z2.cn/311595.htm
qok.24d38z2.cn/131515.htm
qok.24d38z2.cn/175955.htm
qok.24d38z2.cn/355515.htm
qok.24d38z2.cn/351175.htm
qok.24d38z2.cn/751575.htm
qok.24d38z2.cn/797915.htm
qok.24d38z2.cn/393195.htm
qoj.24d38z2.cn/515595.htm
qoj.24d38z2.cn/919375.htm
qoj.24d38z2.cn/591315.htm
qoj.24d38z2.cn/773395.htm
qoj.24d38z2.cn/713715.htm
qoj.24d38z2.cn/755755.htm
qoj.24d38z2.cn/315935.htm
qoj.24d38z2.cn/573175.htm
qoj.24d38z2.cn/919935.htm
qoj.24d38z2.cn/311995.htm
qoh.24d38z2.cn/333575.htm
qoh.24d38z2.cn/535595.htm
qoh.24d38z2.cn/35.htm
qoh.24d38z2.cn/179335.htm
qoh.24d38z2.cn/351795.htm
qoh.24d38z2.cn/397535.htm
qoh.24d38z2.cn/775975.htm
qoh.24d38z2.cn/959975.htm
qoh.24d38z2.cn/979735.htm
qoh.24d38z2.cn/957955.htm
qog.24d38z2.cn/713515.htm
qog.24d38z2.cn/535335.htm
qog.24d38z2.cn/935135.htm
qog.24d38z2.cn/793195.htm
qog.24d38z2.cn/151915.htm
qog.24d38z2.cn/791935.htm
qog.24d38z2.cn/735795.htm
qog.24d38z2.cn/735115.htm
qog.24d38z2.cn/579515.htm
qog.24d38z2.cn/137355.htm
qof.24d38z2.cn/717575.htm
qof.24d38z2.cn/155595.htm
qof.24d38z2.cn/771955.htm
qof.24d38z2.cn/379395.htm
qof.24d38z2.cn/315795.htm
qof.24d38z2.cn/191755.htm
qof.24d38z2.cn/537555.htm
qof.24d38z2.cn/999795.htm
qof.24d38z2.cn/577375.htm
qof.24d38z2.cn/951155.htm
qod.24d38z2.cn/353755.htm
qod.24d38z2.cn/535955.htm
qod.24d38z2.cn/791555.htm
qod.24d38z2.cn/777535.htm
qod.24d38z2.cn/571935.htm
qod.24d38z2.cn/551315.htm
qod.24d38z2.cn/555195.htm
qod.24d38z2.cn/197595.htm
qod.24d38z2.cn/397395.htm
qod.24d38z2.cn/939995.htm
qos.24d38z2.cn/917315.htm
qos.24d38z2.cn/353315.htm
qos.24d38z2.cn/915515.htm
qos.24d38z2.cn/159715.htm
qos.24d38z2.cn/791715.htm
qos.24d38z2.cn/919795.htm
qos.24d38z2.cn/991335.htm
qos.24d38z2.cn/791755.htm
qos.24d38z2.cn/559335.htm
qos.24d38z2.cn/993135.htm
qoa.24d38z2.cn/515515.htm
qoa.24d38z2.cn/553515.htm
qoa.24d38z2.cn/319535.htm
qoa.24d38z2.cn/757195.htm
qoa.24d38z2.cn/915755.htm
qoa.24d38z2.cn/395955.htm
qoa.24d38z2.cn/991375.htm
qoa.24d38z2.cn/979315.htm
qoa.24d38z2.cn/135515.htm
qoa.24d38z2.cn/577175.htm
qop.24d38z2.cn/797935.htm
qop.24d38z2.cn/759735.htm
qop.24d38z2.cn/335595.htm
qop.24d38z2.cn/131195.htm
qop.24d38z2.cn/951735.htm
qop.24d38z2.cn/393375.htm
qop.24d38z2.cn/799335.htm
qop.24d38z2.cn/595315.htm
qop.24d38z2.cn/333555.htm
qop.24d38z2.cn/519115.htm
qoo.24d38z2.cn/371755.htm
qoo.24d38z2.cn/919115.htm
qoo.24d38z2.cn/731955.htm
qoo.24d38z2.cn/511375.htm
qoo.24d38z2.cn/171955.htm
qoo.24d38z2.cn/351775.htm
qoo.24d38z2.cn/391975.htm
qoo.24d38z2.cn/993315.htm
qoo.24d38z2.cn/535995.htm
qoo.24d38z2.cn/533935.htm
qoi.24d38z2.cn/937715.htm
qoi.24d38z2.cn/513375.htm
qoi.24d38z2.cn/777195.htm
qoi.24d38z2.cn/315395.htm
qoi.24d38z2.cn/935515.htm
qoi.24d38z2.cn/599515.htm
qoi.24d38z2.cn/357755.htm
qoi.24d38z2.cn/371995.htm
qoi.24d38z2.cn/135795.htm
qoi.24d38z2.cn/395715.htm
qou.24d38z2.cn/335335.htm
qou.24d38z2.cn/559515.htm
qou.24d38z2.cn/719955.htm
qou.24d38z2.cn/377935.htm
qou.24d38z2.cn/975195.htm
qou.24d38z2.cn/179955.htm
qou.24d38z2.cn/919195.htm
qou.24d38z2.cn/733395.htm
qou.24d38z2.cn/713735.htm
qou.24d38z2.cn/599135.htm
qoy.24d38z2.cn/975735.htm
qoy.24d38z2.cn/773335.htm
qoy.24d38z2.cn/711175.htm
qoy.24d38z2.cn/591555.htm
qoy.24d38z2.cn/779335.htm
qoy.24d38z2.cn/531735.htm
qoy.24d38z2.cn/533995.htm
qoy.24d38z2.cn/313175.htm
qoy.24d38z2.cn/511355.htm
qoy.24d38z2.cn/739715.htm
qot.24d38z2.cn/357395.htm
qot.24d38z2.cn/797175.htm
qot.24d38z2.cn/559195.htm
qot.24d38z2.cn/317775.htm
qot.24d38z2.cn/115935.htm
qot.24d38z2.cn/175135.htm
qot.24d38z2.cn/991515.htm
qot.24d38z2.cn/535355.htm
qot.24d38z2.cn/593335.htm
qot.24d38z2.cn/757355.htm
qor.24d38z2.cn/539715.htm
qor.24d38z2.cn/715375.htm
qor.24d38z2.cn/913175.htm
qor.24d38z2.cn/377375.htm
qor.24d38z2.cn/339375.htm
qor.24d38z2.cn/713395.htm
qor.24d38z2.cn/537935.htm
qor.24d38z2.cn/357315.htm
qor.24d38z2.cn/911795.htm
qor.24d38z2.cn/359135.htm
qoe.24d38z2.cn/399115.htm
qoe.24d38z2.cn/955555.htm
qoe.24d38z2.cn/333395.htm
qoe.24d38z2.cn/711195.htm
qoe.24d38z2.cn/397735.htm
qoe.24d38z2.cn/715975.htm
qoe.24d38z2.cn/753935.htm
qoe.24d38z2.cn/715195.htm
qoe.24d38z2.cn/793375.htm
qoe.24d38z2.cn/191175.htm
qow.24d38z2.cn/919755.htm
qow.24d38z2.cn/151575.htm
qow.24d38z2.cn/731795.htm
qow.24d38z2.cn/537115.htm
qow.24d38z2.cn/937355.htm
qow.24d38z2.cn/313135.htm
qow.24d38z2.cn/753395.htm
qow.24d38z2.cn/393395.htm
qow.24d38z2.cn/733995.htm
qow.24d38z2.cn/151335.htm
qoq.24d38z2.cn/755575.htm
qoq.24d38z2.cn/919795.htm
qoq.24d38z2.cn/359575.htm
qoq.24d38z2.cn/535355.htm
qoq.24d38z2.cn/375195.htm
qoq.24d38z2.cn/731335.htm
qoq.24d38z2.cn/919175.htm
qoq.24d38z2.cn/119955.htm
qoq.24d38z2.cn/513115.htm
qoq.24d38z2.cn/595155.htm
qitv.24d38z2.cn/593535.htm
qitv.24d38z2.cn/135375.htm
qitv.24d38z2.cn/377175.htm
qitv.24d38z2.cn/119935.htm
qitv.24d38z2.cn/937155.htm
qitv.24d38z2.cn/335735.htm
qitv.24d38z2.cn/177935.htm
qitv.24d38z2.cn/131975.htm
qitv.24d38z2.cn/371355.htm
qitv.24d38z2.cn/393315.htm
qin.24d38z2.cn/931715.htm
qin.24d38z2.cn/791115.htm
qin.24d38z2.cn/793775.htm
qin.24d38z2.cn/973175.htm
qin.24d38z2.cn/991575.htm
qin.24d38z2.cn/751755.htm
qin.24d38z2.cn/551515.htm
qin.24d38z2.cn/137595.htm
qin.24d38z2.cn/533115.htm
qin.24d38z2.cn/535155.htm
qib.24d38z2.cn/795575.htm
qib.24d38z2.cn/371555.htm
qib.24d38z2.cn/533795.htm
qib.24d38z2.cn/553755.htm
qib.24d38z2.cn/973175.htm
qib.24d38z2.cn/995795.htm
qib.24d38z2.cn/511795.htm
