奔胰鞍吃平


=================================================================
 Python 线程安全模式：同步原语与并发控制
=================================================================

多线程编程中，线程安全是核心挑战。Python 提供了丰富的同步
原语来协调线程间的访问。本文深入讲解每个原语的用法。

=================================================================
 一、Lock：互斥锁
=================================================================

import threading
import time
import random
import queue

# 共享资源：银行账户余额
balance = 0
# 创建互斥锁
lock = threading.Lock()

def deposit(amount: int):
    """存款操作（线程不安全版本）"""
    global balance
    # 问题：这里不是原子操作，多个线程会互相覆盖
    # balance = balance + amount 实际分三步：
    # 1. 读取 balance  2. CPU 计算  3. 写回 balance
    temp = balance
    time.sleep(0.0001)  # 模拟 IO 延迟，增大竞争概率
    balance = temp + amount

def deposit_safe(amount: int):
    """使用 Lock 保证线程安全"""
    global balance
    # 方式一：使用 acquire/release（不推荐，容易忘记 release）
    lock.acquire()
    try:
        temp = balance
        time.sleep(0.0001)
        balance = temp + amount
    finally:
        lock.release()

def deposit_safe_context(amount: int):
    """使用上下文管理器（推荐方式）"""
    global balance
    # with 语句自动 acquire 和 release
    with lock:
        temp = balance
        time.sleep(0.0001)
        balance = temp + amount

=================================================================
 二、RLock：可重入锁
=================================================================

"""
RLock（可重入锁）允许同一个线程多次 acquire 而不会死锁。
内部维护了 owner 线程和递归计数。
适用场景：递归函数或一个函数内部调用另一个需要锁的函数。
"""

rlock = threading.RLock()

class Counter:
    """使用 RLock 实现可重入的计数器"""

    def __init__(self):
        self.count = 0
        self.lock = rlock

    def increment(self):
        """增加计数"""
        with self.lock:
            self.count += 1

    def increment_by(self, n: int):
        """
        多次增加计数（递归使用锁）。
        如果使用普通 Lock，这里会死锁。
        """
        with self.lock:
            for _ in range(n):
                self.increment()  # 再次获取同一把锁

    def get_value(self):
        """安全地获取当前值"""
        with self.lock:
            return self.count

# Lock vs RLock 选择：
# 同级锁操作使用 Lock（性能更好，语义清晰）
# 需要同一个线程重入时使用 RLock

=================================================================
 三、Semaphore 和 BoundedSemaphore
=================================================================

"""
信号量维护一个计数器，acquire 递减，release 递增。
计数器为 0 时 acquire 阻塞。
适用场景：限制并发访问数量（连接池、限流）。
"""

class ConnectionPool:
    """使用 BoundedSemaphore 实现的连接池"""

    def __init__(self, max_connections: int = 3):
        # BoundedSemaphore 会检查 release 次数不超过初始值
        # 防止 bug 导致信号量被过度 release
        self.semaphore = threading.BoundedSemaphore(max_connections)
        self.connections = list(range(max_connections))

    def acquire(self) -> int:
        """获取一个连接"""
        self.semaphore.acquire()  # 如果池已满则阻塞
        conn = self.connections.pop()
        print(f"获取连接 {conn}，剩余: {len(self.connections)}")
        return conn

    def release(self, conn: int):
        """归还一个连接"""
        self.connections.append(conn)
        self.semaphore.release()
        print(f"归还连接 {conn}，可用: {len(self.connections)}")

# Semaphore vs BoundedSemaphore：
# BoundedSemaphore 在 release 次数超过初始值时抛出 ValueError
# 这可以检测到程序中多调用了 release 的 bug

=================================================================
 四、Condition：条件变量
=================================================================

"""
Condition 结合了 Lock 和事件通知机制。
线程可以等待某个条件满足，另一个线程可以通知条件变化。
经典应用：生产者-消费者模式。
"""

class BoundedBuffer:
    """使用 Condition 实现有界缓冲区"""

    def __init__(self, max_size: int = 5):
        self.buffer = []
        self.max_size = max_size
        self.condition = threading.Condition()

    def produce(self, item: int):
        """生产一个物品"""
        with self.condition:
            # 当缓冲区满时等待
            while len(self.buffer) >= self.max_size:
                print(f"缓冲区已满，生产者等待...")
                self.condition.wait()  # 释放锁并阻塞
            # 生产物品
            self.buffer.append(item)
            print(f"生产: {item}，缓冲区大小: {len(self.buffer)}")
            # 通知等待的消费者
            self.condition.notify()  # 唤醒一个等待线程
            # self.condition.notify_all()  # 唤醒所有等待线程

    def consume(self) -> int:
        """消费一个物品"""
        with self.condition:
            # 当缓冲区空时等待
            while not self.buffer:
                print(f"缓冲区为空，消费者等待...")
                self.condition.wait()
            # 消费物品
            item = self.buffer.pop(0)
            print(f"消费: {item}，缓冲区大小: {len(self.buffer)}")
            # 通知等待的生产者
            self.condition.notify()
            return item

=================================================================
 五、Event：事件通知
=================================================================

"""
Event 用于线程间的简单信号通信。
一个线程设置事件，另一个线程等待事件。
特点：一次性使用（set 后所有 wait 立即返回）。
"""

event = threading.Event()

def worker(event: threading.Event, name: str):
    """等待主线程信号的工人线程"""
    print(f"{name} 等待开始信号...")
    # 阻塞直到事件被设置
    event.wait()
    print(f"{name} 收到信号，开始工作！")
    # 模拟工作
    time.sleep(random.uniform(0.5, 1.5))
    print(f"{name} 工作完成")

def boss():
    """主线程：发号施令"""
    threads = []
    for i in range(3):
        t = threading.Thread(target=worker, args=(event, f"工人-{i}"))
        t.start()
        threads.append(t)

    print("老板准备中...")
    time.sleep(2)
    print("老板发出开始信号！")
    event.set()  # 所有工人同时开始

    for t in threads:
        t.join()

# Event vs Condition：
# Event：简单的一对多一次性通知
# Condition：复杂的重复条件等待

=================================================================
 六、Barrier：屏障同步
=================================================================

"""
Barrier 让多个线程相互等待，直到所有线程都到达屏障点。
适用场景：并行计算中的分阶段同步。
"""

def worker_barrier(barrier: threading.Barrier, name: str):
    """使用 Barrier 同步的工作线程"""
    print(f"{name} 开始阶段 1 的工作...")
    time.sleep(random.uniform(0.5, 2))
    print(f"{name} 完成阶段 1，等待其他线程...")
    # 等待所有线程完成阶段 1
    barrier.wait()
    print(f"{name} 开始阶段 2 的工作...")
    time.sleep(random.uniform(0.5, 1))
    print(f"{name} 完成所有工作！")

def barrier_demo():
    """演示 Barrier 的同步效果"""
    # 3 个线程，超时 5 秒
    barrier = threading.Barrier(3, timeout=5)
    threads = []
    for i in range(3):
        t = threading.Thread(
            target=worker_barrier, args=(barrier, f"Worker-{i}")
        )
        t.start()
        threads.append(t)
    for t in threads:
        t.join()

=================================================================
 七、线程安全队列
=================================================================

"""
queue.Queue 是线程安全的 FIFO 队列，内部使用了 Condition。
比手动使用 Lock + list 更可靠。
"""

def queue_demo():
    """使用 Queue 实现生产者消费者"""
    q = queue.Queue(maxsize=5)

    def producer():
        for i in range(10):
            item = f"数据-{i}"
            q.put(item)  # 如果队列满则阻塞
            print(f"生产: {item}")
            time.sleep(random.uniform(0.1, 0.3))

    def consumer(name: str):
        while True:
            item = q.get()  # 如果队列空则阻塞
            if item is None:  # 哨兵值，表示结束
                break
            print(f"[{name}] 消费: {item}")
            q.task_done()  # 标记任务完成
            time.sleep(random.uniform(0.2, 0.5))

    threads = [
        threading.Thread(target=producer, daemon=True),
        threading.Thread(target=consumer, args=("C1",), daemon=True),
        threading.Thread(target=consumer, args=("C2",), daemon=True),
    ]
    for t in threads:
        t.start()
    producer.join()
    q.join()  # 等待所有任务完成

=================================================================
 八、死锁预防
=================================================================

class DeadlockPrevention:
    """死锁预防策略"""

    @staticmethod
    def example_fixed_order():
        """策略 1：固定锁顺序"""
        lock_a = threading.Lock()
        lock_b = threading.Lock()

        def worker_1():
            for _ in range(1000):
                # 总是先获取 lock_a，再获取 lock_b
                with lock_a:
                    with lock_b:
                        pass  # 业务逻辑

        def worker_2():
            for _ in range(1000):
                # 也按相同顺序获取锁
                with lock_a:  # 而不是先拿 lock_b
                    with lock_b:
                        pass  # 业务逻辑

@staticmethod
def example_timeout():
    """策略 2：使用超时获取锁"""
    lock = threading.Lock()
    if lock.acquire(timeout=1.0):  # 最多等待 1 秒
        try:
            pass  # 业务逻辑
        finally:
            lock.release()
    else:
        print("获取锁超时，执行回退策略")

# 其他预防策略：
# 3. 锁粒度尽量小（只保护必要的代码段）
# 4. 尽量使用 queue.Queue 替代手动锁
# 5. 避免在持锁时调用外部代码
# 6. 优先使用 threading 高级工具而非 Lock

=================================================================
 九、总结
=================================================================

# 1. Lock：基本互斥锁，不可重入
# 2. RLock：可重入锁，适用于递归场景
# 3. Semaphore/BoundedSemaphore：控制并发访问数量
# 4. Condition：复杂条件等待与通知
# 5. Event：一次性信号通知
# 6. Barrier：多阶段同步屏障
# 7. Queue：线程安全队列，优先于手动锁
# 8. 死锁预防：固定顺序、超时、细粒度、使用高级工具

if __name__ == "__main__":
    barrier_demo()

藤掖惫段谆鼐壳还粤倨脱匪床杀部

qjm.sthxr.cn/711393.htm
qjm.sthxr.cn/539953.htm
qjm.sthxr.cn/739993.htm
qjm.sthxr.cn/064443.htm
qjm.sthxr.cn/933113.htm
qjm.sthxr.cn/379753.htm
qjm.sthxr.cn/955193.htm
qjm.sthxr.cn/151593.htm
qjm.sthxr.cn/955993.htm
qjm.sthxr.cn/577773.htm
qjn.sthxr.cn/080403.htm
qjn.sthxr.cn/575373.htm
qjn.sthxr.cn/311753.htm
qjn.sthxr.cn/971953.htm
qjn.sthxr.cn/133113.htm
qjn.sthxr.cn/379173.htm
qjn.sthxr.cn/197953.htm
qjn.sthxr.cn/137573.htm
qjn.sthxr.cn/117133.htm
qjn.sthxr.cn/395593.htm
qjb.sthxr.cn/226063.htm
qjb.sthxr.cn/393153.htm
qjb.sthxr.cn/317373.htm
qjb.sthxr.cn/355153.htm
qjb.sthxr.cn/462823.htm
qjb.sthxr.cn/155393.htm
qjb.sthxr.cn/713993.htm
qjb.sthxr.cn/406483.htm
qjb.sthxr.cn/191913.htm
qjb.sthxr.cn/539773.htm
qjv.sthxr.cn/571593.htm
qjv.sthxr.cn/779193.htm
qjv.sthxr.cn/953173.htm
qjv.sthxr.cn/155573.htm
qjv.sthxr.cn/804203.htm
qjv.sthxr.cn/111593.htm
qjv.sthxr.cn/751193.htm
qjv.sthxr.cn/608443.htm
qjv.sthxr.cn/155753.htm
qjv.sthxr.cn/751733.htm
qjc.sthxr.cn/315153.htm
qjc.sthxr.cn/551333.htm
qjc.sthxr.cn/153333.htm
qjc.sthxr.cn/848663.htm
qjc.sthxr.cn/135393.htm
qjc.sthxr.cn/339753.htm
qjc.sthxr.cn/537713.htm
qjc.sthxr.cn/397153.htm
qjc.sthxr.cn/175333.htm
qjc.sthxr.cn/379353.htm
qjx.sthxr.cn/226223.htm
qjx.sthxr.cn/793573.htm
qjx.sthxr.cn/333373.htm
qjx.sthxr.cn/155953.htm
qjx.sthxr.cn/424063.htm
qjx.sthxr.cn/517353.htm
qjx.sthxr.cn/991953.htm
qjx.sthxr.cn/957953.htm
qjx.sthxr.cn/315133.htm
qjx.sthxr.cn/551373.htm
qjz.sthxr.cn/464643.htm
qjz.sthxr.cn/008443.htm
qjz.sthxr.cn/535113.htm
qjz.sthxr.cn/462463.htm
qjz.sthxr.cn/460203.htm
qjz.sthxr.cn/573773.htm
qjz.sthxr.cn/797793.htm
qjz.sthxr.cn/753973.htm
qjz.sthxr.cn/597173.htm
qjz.sthxr.cn/359993.htm
qjl.sthxr.cn/373113.htm
qjl.sthxr.cn/482463.htm
qjl.sthxr.cn/179353.htm
qjl.sthxr.cn/911713.htm
qjl.sthxr.cn/111393.htm
qjl.sthxr.cn/200243.htm
qjl.sthxr.cn/133153.htm
qjl.sthxr.cn/157753.htm
qjl.sthxr.cn/288243.htm
qjl.sthxr.cn/959553.htm
qjk.sthxr.cn/557593.htm
qjk.sthxr.cn/242443.htm
qjk.sthxr.cn/515793.htm
qjk.sthxr.cn/551113.htm
qjk.sthxr.cn/955993.htm
qjk.sthxr.cn/806043.htm
qjk.sthxr.cn/937773.htm
qjk.sthxr.cn/779373.htm
qjk.sthxr.cn/804283.htm
qjk.sthxr.cn/715933.htm
qjj.sthxr.cn/375333.htm
qjj.sthxr.cn/282223.htm
qjj.sthxr.cn/591573.htm
qjj.sthxr.cn/975513.htm
qjj.sthxr.cn/593773.htm
qjj.sthxr.cn/422463.htm
qjj.sthxr.cn/397933.htm
qjj.sthxr.cn/464803.htm
qjj.sthxr.cn/006423.htm
qjj.sthxr.cn/333993.htm
qjh.sthxr.cn/913393.htm
qjh.sthxr.cn/735913.htm
qjh.sthxr.cn/911793.htm
qjh.sthxr.cn/537573.htm
qjh.sthxr.cn/331793.htm
qjh.sthxr.cn/573353.htm
qjh.sthxr.cn/153713.htm
qjh.sthxr.cn/133313.htm
qjh.sthxr.cn/486863.htm
qjh.sthxr.cn/357113.htm
qjg.sthxr.cn/971593.htm
qjg.sthxr.cn/993993.htm
qjg.sthxr.cn/599513.htm
qjg.sthxr.cn/715973.htm
qjg.sthxr.cn/151793.htm
qjg.sthxr.cn/371533.htm
qjg.sthxr.cn/179953.htm
qjg.sthxr.cn/959133.htm
qjg.sthxr.cn/482403.htm
qjg.sthxr.cn/575953.htm
qjf.sthxr.cn/559553.htm
qjf.sthxr.cn/264603.htm
qjf.sthxr.cn/808003.htm
qjf.sthxr.cn/957553.htm
qjf.sthxr.cn/513173.htm
qjf.sthxr.cn/600423.htm
qjf.sthxr.cn/151573.htm
qjf.sthxr.cn/151533.htm
qjf.sthxr.cn/995133.htm
qjf.sthxr.cn/195373.htm
qjd.sthxr.cn/799993.htm
qjd.sthxr.cn/137933.htm
qjd.sthxr.cn/595353.htm
qjd.sthxr.cn/591573.htm
qjd.sthxr.cn/202083.htm
qjd.sthxr.cn/531973.htm
qjd.sthxr.cn/395173.htm
qjd.sthxr.cn/571973.htm
qjd.sthxr.cn/519913.htm
qjd.sthxr.cn/355153.htm
qjs.sthxr.cn/991753.htm
qjs.sthxr.cn/777313.htm
qjs.sthxr.cn/402623.htm
qjs.sthxr.cn/591173.htm
qjs.sthxr.cn/337933.htm
qjs.sthxr.cn/139353.htm
qjs.sthxr.cn/717993.htm
qjs.sthxr.cn/335753.htm
qjs.sthxr.cn/040643.htm
qjs.sthxr.cn/379733.htm
qja.sthxr.cn/779333.htm
qja.sthxr.cn/391373.htm
qja.sthxr.cn/151773.htm
qja.sthxr.cn/917773.htm
qja.sthxr.cn/713553.htm
qja.sthxr.cn/179393.htm
qja.sthxr.cn/408063.htm
qja.sthxr.cn/113773.htm
qja.sthxr.cn/402683.htm
qja.sthxr.cn/284063.htm
qjp.sthxr.cn/357593.htm
qjp.sthxr.cn/375513.htm
qjp.sthxr.cn/911193.htm
qjp.sthxr.cn/955993.htm
qjp.sthxr.cn/331573.htm
qjp.sthxr.cn/242683.htm
qjp.sthxr.cn/759553.htm
qjp.sthxr.cn/333313.htm
qjp.sthxr.cn/000483.htm
qjp.sthxr.cn/286043.htm
qjo.sthxr.cn/537133.htm
qjo.sthxr.cn/717113.htm
qjo.sthxr.cn/995753.htm
qjo.sthxr.cn/559793.htm
qjo.sthxr.cn/937373.htm
qjo.sthxr.cn/006443.htm
qjo.sthxr.cn/884203.htm
qjo.sthxr.cn/713773.htm
qjo.sthxr.cn/313533.htm
qjo.sthxr.cn/424603.htm
qji.sthxr.cn/333753.htm
qji.sthxr.cn/575913.htm
qji.sthxr.cn/822623.htm
qji.sthxr.cn/971953.htm
qji.sthxr.cn/519753.htm
qji.sthxr.cn/391153.htm
qji.sthxr.cn/559153.htm
qji.sthxr.cn/715573.htm
qji.sthxr.cn/593333.htm
qji.sthxr.cn/315113.htm
qju.sthxr.cn/008603.htm
qju.sthxr.cn/939173.htm
qju.sthxr.cn/195153.htm
qju.sthxr.cn/248623.htm
qju.sthxr.cn/713353.htm
qju.sthxr.cn/515773.htm
qju.sthxr.cn/173913.htm
qju.sthxr.cn/068683.htm
qju.sthxr.cn/359953.htm
qju.sthxr.cn/553353.htm
qjy.sthxr.cn/575393.htm
qjy.sthxr.cn/755553.htm
qjy.sthxr.cn/971713.htm
qjy.sthxr.cn/771153.htm
qjy.sthxr.cn/406843.htm
qjy.sthxr.cn/115733.htm
qjy.sthxr.cn/953193.htm
qjy.sthxr.cn/119973.htm
qjy.sthxr.cn/579993.htm
qjy.sthxr.cn/977713.htm
qjt.sthxr.cn/511933.htm
qjt.sthxr.cn/391533.htm
qjt.sthxr.cn/771553.htm
qjt.sthxr.cn/135533.htm
qjt.sthxr.cn/608683.htm
qjt.sthxr.cn/117913.htm
qjt.sthxr.cn/717713.htm
qjt.sthxr.cn/577993.htm
qjt.sthxr.cn/802603.htm
qjt.sthxr.cn/791393.htm
qjr.sthxr.cn/317373.htm
qjr.sthxr.cn/448483.htm
qjr.sthxr.cn/597113.htm
qjr.sthxr.cn/799553.htm
qjr.sthxr.cn/022243.htm
qjr.sthxr.cn/024223.htm
qjr.sthxr.cn/573173.htm
qjr.sthxr.cn/53.htm
qjr.sthxr.cn/462223.htm
qjr.sthxr.cn/151133.htm
qje.sthxr.cn/973173.htm
qje.sthxr.cn/860403.htm
qje.sthxr.cn/397333.htm
qje.sthxr.cn/333153.htm
qje.sthxr.cn/197933.htm
qje.sthxr.cn/862483.htm
qje.sthxr.cn/959753.htm
qje.sthxr.cn/375753.htm
qje.sthxr.cn/515113.htm
qje.sthxr.cn/933133.htm
qjw.sthxr.cn/571773.htm
qjw.sthxr.cn/715573.htm
qjw.sthxr.cn/195153.htm
qjw.sthxr.cn/719953.htm
qjw.sthxr.cn/311333.htm
qjw.sthxr.cn/266663.htm
qjw.sthxr.cn/315933.htm
qjw.sthxr.cn/171553.htm
qjw.sthxr.cn/466283.htm
qjw.sthxr.cn/595513.htm
qjq.sthxr.cn/973153.htm
qjq.sthxr.cn/573593.htm
qjq.sthxr.cn/377173.htm
qjq.sthxr.cn/513913.htm
qjq.sthxr.cn/157373.htm
qjq.sthxr.cn/937353.htm
qjq.sthxr.cn/402643.htm
qjq.sthxr.cn/993733.htm
qjq.sthxr.cn/353973.htm
qjq.sthxr.cn/991793.htm
qhm.sthxr.cn/733333.htm
qhm.sthxr.cn/842623.htm
qhm.sthxr.cn/973753.htm
qhm.sthxr.cn/248023.htm
qhm.sthxr.cn/117313.htm
qhm.sthxr.cn/915533.htm
qhm.sthxr.cn/339373.htm
qhm.sthxr.cn/757713.htm
qhm.sthxr.cn/191973.htm
qhm.sthxr.cn/533773.htm
qhn.sthxr.cn/446083.htm
qhn.sthxr.cn/155993.htm
qhn.sthxr.cn/591173.htm
qhn.sthxr.cn/573513.htm
qhn.sthxr.cn/751933.htm
qhn.sthxr.cn/991173.htm
qhn.sthxr.cn/375113.htm
qhn.sthxr.cn/331793.htm
qhn.sthxr.cn/888463.htm
qhn.sthxr.cn/391173.htm
qhb.sthxr.cn/191333.htm
qhb.sthxr.cn/404683.htm
qhb.sthxr.cn/793353.htm
qhb.sthxr.cn/111333.htm
qhb.sthxr.cn/175333.htm
qhb.sthxr.cn/064023.htm
qhb.sthxr.cn/153753.htm
qhb.sthxr.cn/177713.htm
qhb.sthxr.cn/888863.htm
qhb.sthxr.cn/917913.htm
qhv.sthxr.cn/713513.htm
qhv.sthxr.cn/600603.htm
qhv.sthxr.cn/806883.htm
qhv.sthxr.cn/513753.htm
qhv.sthxr.cn/151113.htm
qhv.sthxr.cn/555973.htm
qhv.sthxr.cn/955933.htm
qhv.sthxr.cn/959533.htm
qhv.sthxr.cn/577153.htm
qhv.sthxr.cn/268823.htm
qhc.sthxr.cn/177953.htm
qhc.sthxr.cn/757933.htm
qhc.sthxr.cn/391753.htm
qhc.sthxr.cn/662283.htm
qhc.sthxr.cn/735393.htm
qhc.sthxr.cn/973153.htm
qhc.sthxr.cn/573373.htm
qhc.sthxr.cn/915373.htm
qhc.sthxr.cn/917173.htm
qhc.sthxr.cn/224203.htm
qhx.sthxr.cn/191753.htm
qhx.sthxr.cn/280803.htm
qhx.sthxr.cn/153793.htm
qhx.sthxr.cn/117993.htm
qhx.sthxr.cn/315793.htm
qhx.sthxr.cn/795993.htm
qhx.sthxr.cn/579593.htm
qhx.sthxr.cn/351733.htm
qhx.sthxr.cn/751173.htm
qhx.sthxr.cn/515153.htm
qhz.sthxr.cn/359773.htm
qhz.sthxr.cn/935553.htm
qhz.sthxr.cn/131153.htm
qhz.sthxr.cn/868823.htm
qhz.sthxr.cn/640063.htm
qhz.sthxr.cn/113333.htm
qhz.sthxr.cn/351773.htm
qhz.sthxr.cn/593953.htm
qhz.sthxr.cn/153313.htm
qhz.sthxr.cn/399753.htm
qhl.sthxr.cn/246283.htm
qhl.sthxr.cn/068423.htm
qhl.sthxr.cn/795573.htm
qhl.sthxr.cn/113733.htm
qhl.sthxr.cn/064403.htm
qhl.sthxr.cn/377333.htm
qhl.sthxr.cn/73.htm
qhl.sthxr.cn/288283.htm
qhl.sthxr.cn/646823.htm
qhl.sthxr.cn/315113.htm
qhk.sthxr.cn/955193.htm
qhk.sthxr.cn/088463.htm
qhk.sthxr.cn/957153.htm
qhk.sthxr.cn/553933.htm
qhk.sthxr.cn/953933.htm
qhk.sthxr.cn/757313.htm
qhk.sthxr.cn/975973.htm
qhk.sthxr.cn/959913.htm
qhk.sthxr.cn/575133.htm
qhk.sthxr.cn/373353.htm
qhj.sthxr.cn/997313.htm
qhj.sthxr.cn/537953.htm
qhj.sthxr.cn/537793.htm
qhj.sthxr.cn/737173.htm
qhj.sthxr.cn/551953.htm
qhj.sthxr.cn/579113.htm
qhj.sthxr.cn/000083.htm
qhj.sthxr.cn/993393.htm
qhj.sthxr.cn/575793.htm
qhj.sthxr.cn/222663.htm
qhh.sthxr.cn/391153.htm
qhh.sthxr.cn/573373.htm
qhh.sthxr.cn/424683.htm
qhh.sthxr.cn/002643.htm
qhh.sthxr.cn/535193.htm
qhh.sthxr.cn/355133.htm
qhh.sthxr.cn/777593.htm
qhh.sthxr.cn/151373.htm
qhh.sthxr.cn/373333.htm
qhh.sthxr.cn/820603.htm
qhg.sthxr.cn/462223.htm
qhg.sthxr.cn/993733.htm
qhg.sthxr.cn/153593.htm
qhg.sthxr.cn/533173.htm
qhg.sthxr.cn/759753.htm
qhg.sthxr.cn/539973.htm
qhg.sthxr.cn/426863.htm
qhg.sthxr.cn/531573.htm
qhg.sthxr.cn/913113.htm
qhg.sthxr.cn/177533.htm
qhf.sthxr.cn/008863.htm
qhf.sthxr.cn/139993.htm
qhf.sthxr.cn/155373.htm
qhf.sthxr.cn/977133.htm
qhf.sthxr.cn/573313.htm
qhf.sthxr.cn/913113.htm
qhf.sthxr.cn/177313.htm
qhf.sthxr.cn/824223.htm
qhf.sthxr.cn/393313.htm
qhf.sthxr.cn/991393.htm
qhd.sthxr.cn/220603.htm
qhd.sthxr.cn/648423.htm
qhd.sthxr.cn/371773.htm
qhd.sthxr.cn/593193.htm
qhd.sthxr.cn/753793.htm
