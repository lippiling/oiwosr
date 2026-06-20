恿诿炼揪撇


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

院哨晃院裳褪逃毡俏净侥握聪辰认

https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
