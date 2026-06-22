瓶唐诤拭眯


=================================================================
 Python 多进程管理器：便捷的进程间数据共享
=================================================================

multiprocessing.Manager 提供了一种高级的进程间数据共享方式。
它通过代理对象在多个进程之间同步数据，无需手动处理锁和
共享内存。本文详解 Manager 的完整用法。

=================================================================
 一、Manager 基础：创建和使用
=================================================================

import multiprocessing as mp
import time
from multiprocessing.managers import (
    BaseManager, BaseProxy, SyncManager
)

def manager_basics():
    """Manager 的基本用法"""
    # 创建 Manager 对象（启动管理器服务器进程）
    manager = mp.Manager()

    # 创建各种共享数据结构
    shared_list = manager.list([1, 2, 3])
    shared_dict = manager.dict({"key": "value"})
    shared_set = manager.set([1, 2, 3])
    shared_namespace = manager.Namespace()

    # Namespace 可以通过属性访问
    shared_namespace.counter = 0
    shared_namespace.config = {"host": "localhost", "port": 8080}

    # 在子进程中修改共享数据
    def worker(lst, dct, ns):
        """子进程：修改 Manager 管理的共享对象"""
        lst.append(4)
        lst.append(5)
        dct["new_key"] = "new_value"
        ns.counter += 1
        print(f"子进程: list={list(lst)}, dict={dict(dct)}, counter={ns.counter}")

    processes = []
    for i in range(3):
        p = mp.Process(
            target=worker,
            args=(shared_list, shared_dict, shared_namespace)
        )
        p.start()
        processes.append(p)

    for p in processes:
        p.join()

    print(f"主进程: list={list(shared_list)}")
    print(f"主进程: dict={dict(shared_dict)}")
    print(f"主进程: counter={shared_namespace.counter}")

    # 使用完毕后必须 shutdown
    manager.shutdown()

=================================================================
 二、Manager 提供的核心数据类型
=================================================================

def manager_data_types():
    """Manager 支持的所有数据类型"""
    manager = mp.Manager()

    # 1. list: 类似 list 的代理
    lst = manager.list([1, 2, 3])
    lst.append(4)
    lst.extend([5, 6])
    print(f"list: {list(lst)}, 长度={len(lst)}")

    # 2. dict: 类似 dict 的代理
    dct = manager.dict({"a": 1})
    dct["b"] = 2
    dct.update({"c": 3, "d": 4})
    print(f"dict: {dict(dct)}")

    # 3. set: 类似 set 的代理（Python 3.6+）
    s = manager.set([1, 2, 3, 3, 2])  # 去重
    s.add(4)
    s.discard(1)
    print(f"set: {set(s)}")

    # 4. Namespace: 属性访问的代理
    ns = manager.Namespace()
    ns.x = 10
    ns.y = 20
    # Namespace 支持嵌套
    ns.sub = manager.Namespace()
    ns.sub.value = "nested"
    print(f"Namespace: x={ns.x}, sub.value={ns.sub.value}")

    # 5. Queue: 进程安全队列
    q = manager.Queue()
    q.put("task-1")
    q.put("task-2")
    print(f"Queue: get={q.get()}")

    # 6. Lock / RLock / Semaphore / Event / Condition
    lock = manager.Lock()
    with lock:
        print("获取了 Manager 提供的锁")

    manager.shutdown()

=================================================================
 三、自定义 Manager 类型
=================================================================

class MyCustomManager(BaseManager):
    """
    自定义 Manager，注册自定义类型。
    适用于需要共享自定义类的场景。
    """

class Counter:
    """一个简单的计数器类"""
    def __init__(self, initial: int = 0):
        self._count = initial

    def increment(self, amount: int = 1):
        self._count += amount

    def get_count(self) -> int:
        return self._count

    def __repr__(self) -> str:
        return f"Counter({self._count})"

# 注册自定义类型到 Manager
MyCustomManager.register("Counter", Counter)

def custom_manager_demo():
    """使用自定义 Manager 类型"""
    manager = MyCustomManager()
    manager.start()  # 启动管理器服务器

    # 创建共享的 Counter 实例
    shared_counter = manager.Counter(10)

    def worker(counter):
        """子进程：操作共享计数器"""
        for _ in range(100):
            counter.increment()

    processes = [
        mp.Process(target=worker, args=(shared_counter,))
        for _ in range(4)
    ]
    for p in processes: p.start()
    for p in processes: p.join()

    print(f"最终计数: {shared_counter.get_count()} (期望: 410)")
    manager.shutdown()

=================================================================
 四、代理对象的工作机制
=================================================================

def proxy_object_demo():
    """
    代理对象是 Manager 的核心机制。

    当通过 Manager 创建对象时，实际上是在 Manager 服务器
    进程中创建了一个真实对象，而客户端进程获得的是代理对象。

    对代理对象的操作会被序列化并通过管道发送到服务器进程
    执行，结果再反序列化返回。
    """
    manager = mp.Manager()
    shared_dict = manager.dict({"count": 0})

    def examine_proxy(proxy):
        """检查代理对象的内部机制"""
        # 代理对象类型
        print(f"代理类型: {type(proxy)}")
        print(f"代理 ID: {id(proxy)}")

        # 代理对象的自动同步
        proxy["count"] += 1
        # 上面的操作等价于（伪代码）：
        # 1. 序列化操作：("__setitem__", ("count", 1))
        # 2. 通过管道发送到 Manager 服务器
        # 3. 服务器执行真实字典的 __setitem__
        # 4. 返回确认

    examine_proxy(shared_dict)

    # 代理对象的局限性：
    # 1. 某些原地操作不会自动同步（如 list.sort()）
    # 2. 嵌套修改需要重新赋值
    # 3. 属性访问较慢（每次都有 IPC 开销）

    manager.shutdown()

=================================================================
 五、Manager vs 共享内存性能对比
=================================================================

def manager_vs_shared_memory():
    """
    Manager 使用 IPC（进程间通信），每次操作都有序列化开销。
    共享内存使用零拷贝，在大数据量场景下性能更优。

    选择依据：
    - 小数据量、频繁读写：Manager 更方便
    - 大数据量、批量操作：共享内存更高效
    - 需要复杂数据结构：Manager 更易用
    - 需要极致性能：共享内存
    """
    import array
    from multiprocessing import shared_memory

    DATA_SIZE = 100_000
    ITERATIONS = 100

    # === Manager 性能测试 ===
    manager = mp.Manager()
    shared_list = manager.list(range(DATA_SIZE))

    def manager_worker(lst):
        for i in range(len(lst)):
            lst[i] = lst[i] * 2

    start = time.perf_counter()
    processes = [
        mp.Process(target=manager_worker, args=(shared_list,))
        for _ in range(4)
    ]
    for p in processes: p.start()
    for p in processes: p.join()
    manager_time = time.perf_counter() - start
    manager.shutdown()

    # === 共享内存性能测试 ===
    data = array.array('i', range(DATA_SIZE))
    shm = shared_memory.SharedMemory(
        create=True, size=len(data) * data.itemsize
    )
    shm.buf[:len(data) * data.itemsize] = bytes(data)

    def shm_worker(shm_name, size):
        ex_shm = shared_memory.SharedMemory(name=shm_name)
        arr = array.array('i')
        arr.frombytes(ex_shm.buf[:size])
        for i in range(len(arr)):
            arr[i] *= 2
        ex_shm.buf[:size] = bytes(arr)
        ex_shm.close()

    start = time.perf_counter()
    processes = [
        mp.Process(target=shm_worker, args=(shm.name, len(data) * data.itemsize))
        for _ in range(4)
    ]
    for p in processes: p.start()
    for p in processes: p.join()
    shm_time = time.perf_counter() - start

    shm.close()
    shm.unlink()

    print(f"=== Manager vs 共享内存性能（{DATA_SIZE} 元素） ===")
    print(f"Manager 操作: {manager_time:.3f}s")
    print(f"共享内存: {shm_time:.3f}s")
    print(f"Manager 比共享内存慢 {manager_time / shm_time:.1f}x")

=================================================================
 六、分布式多进程
=================================================================

"""
Manager 支持网络访问。可以在一台机器上启动 Manager 服务器，
其他机器通过网络连接进行远程进程间通信。

适用场景：
1. 跨机器的分布式任务队列
2. 共享配置中心
3. 分布式锁
"""

def remote_manager_server():
    """启动可远程访问的 Manager 服务器"""
    manager = mp.Manager()
    # 获取 Manager 服务器地址
    # 默认只监听本地，可以配置为监听网络接口
    print(f"Manager 地址: {manager._address}")
    print(f"Manager 进程 PID: {manager._process.pid}")
    # 实际使用时，需要通过 multiprocessing.managers 自定义
    return manager

def custom_remote_manager():
    """创建可远程连接的自定义 Manager"""

    class RemoteManager(BaseManager):
        pass

    # 注册需要共享的类型
    RemoteManager.register("get_dict", callable=lambda: {})

    # 启动服务器（监听指定地址）
    manager = RemoteManager(address=("0.0.0.0", 50000), authkey=b"secret")
    manager.start()
    print(f"远程 Manager 服务器启动在 {manager._address}")

    # 客户端连接（其他机器上）
    # client = RemoteManager(address=("server_ip", 50000), authkey=b"secret")
    # client.connect()
    # shared_data = client.get_dict()

    return manager

=================================================================
 七、总结
=================================================================

# 1. Manager 提供了 list/dict/set/Namespace/Queue/Lock 等共享类型
# 2. 内部使用代理模式：操作被序列化后发送到 Manager 服务器执行
# 3. 自定义 Manager 可以注册自定义类来共享
# 4. 代理对象方便易用但每次操作都有 IPC 开销
# 5. 大数据量场景共享内存性能更优
# 6. Manager 支持网络分布式访问
# 7. 使用后必须调用 shutdown() 清理资源
# 8. 适合需要复杂数据结构的进程间通信

if __name__ == "__main__":
    manager_basics()

钠炊煽呕业几淮忌敬烟慌坟衫日部

asb.jouwir.cn/002808.Doc
asb.jouwir.cn/022204.Doc
asb.jouwir.cn/802208.Doc
asb.jouwir.cn/044648.Doc
asb.jouwir.cn/468222.Doc
asb.jouwir.cn/082208.Doc
asv.jouwir.cn/464882.Doc
asv.jouwir.cn/484826.Doc
asv.jouwir.cn/446862.Doc
asv.jouwir.cn/684206.Doc
asv.jouwir.cn/624866.Doc
asv.jouwir.cn/517353.Doc
asv.jouwir.cn/426686.Doc
asv.jouwir.cn/446442.Doc
asv.jouwir.cn/000808.Doc
asv.jouwir.cn/622486.Doc
asc.jouwir.cn/842406.Doc
asc.jouwir.cn/222608.Doc
asc.jouwir.cn/486222.Doc
asc.jouwir.cn/597991.Doc
asc.jouwir.cn/022222.Doc
asc.jouwir.cn/640664.Doc
asc.jouwir.cn/202828.Doc
asc.jouwir.cn/246826.Doc
asc.jouwir.cn/713829.Doc
asc.jouwir.cn/604280.Doc
asx.jouwir.cn/622648.Doc
asx.jouwir.cn/468286.Doc
asx.jouwir.cn/404846.Doc
asx.jouwir.cn/204640.Doc
asx.jouwir.cn/282206.Doc
asx.jouwir.cn/846220.Doc
asx.jouwir.cn/666866.Doc
asx.jouwir.cn/806866.Doc
asx.jouwir.cn/080860.Doc
asx.jouwir.cn/006046.Doc
asz.jouwir.cn/822448.Doc
asz.jouwir.cn/802606.Doc
asz.jouwir.cn/420626.Doc
asz.jouwir.cn/206268.Doc
asz.jouwir.cn/084080.Doc
asz.jouwir.cn/824228.Doc
asz.jouwir.cn/628640.Doc
asz.jouwir.cn/088208.Doc
asz.jouwir.cn/800804.Doc
asz.jouwir.cn/660246.Doc
asl.jouwir.cn/842424.Doc
asl.jouwir.cn/686402.Doc
asl.jouwir.cn/680866.Doc
asl.jouwir.cn/002444.Doc
asl.jouwir.cn/288804.Doc
asl.jouwir.cn/644446.Doc
asl.jouwir.cn/262820.Doc
asl.jouwir.cn/268868.Doc
asl.jouwir.cn/864640.Doc
asl.jouwir.cn/606402.Doc
ask.jouwir.cn/935193.Doc
ask.jouwir.cn/402242.Doc
ask.jouwir.cn/644266.Doc
ask.jouwir.cn/622620.Doc
ask.jouwir.cn/046080.Doc
ask.jouwir.cn/424442.Doc
ask.jouwir.cn/482242.Doc
ask.jouwir.cn/800004.Doc
ask.jouwir.cn/046666.Doc
ask.jouwir.cn/208222.Doc
asj.jouwir.cn/642668.Doc
asj.jouwir.cn/028040.Doc
asj.jouwir.cn/280088.Doc
asj.jouwir.cn/862266.Doc
asj.jouwir.cn/408648.Doc
asj.jouwir.cn/395799.Doc
asj.jouwir.cn/004042.Doc
asj.jouwir.cn/028424.Doc
asj.jouwir.cn/406880.Doc
asj.jouwir.cn/426024.Doc
ash.jouwir.cn/828482.Doc
ash.jouwir.cn/488222.Doc
ash.jouwir.cn/486668.Doc
ash.jouwir.cn/022622.Doc
ash.jouwir.cn/579399.Doc
ash.jouwir.cn/804240.Doc
ash.jouwir.cn/022266.Doc
ash.jouwir.cn/262888.Doc
ash.jouwir.cn/842460.Doc
ash.jouwir.cn/222668.Doc
asg.jouwir.cn/464422.Doc
asg.jouwir.cn/464882.Doc
asg.jouwir.cn/062024.Doc
asg.jouwir.cn/668280.Doc
asg.jouwir.cn/846848.Doc
asg.jouwir.cn/228886.Doc
asg.jouwir.cn/042248.Doc
asg.jouwir.cn/004680.Doc
asg.jouwir.cn/800248.Doc
asg.jouwir.cn/860860.Doc
asf.jouwir.cn/882824.Doc
asf.jouwir.cn/422446.Doc
asf.jouwir.cn/480448.Doc
asf.jouwir.cn/642246.Doc
asf.jouwir.cn/648468.Doc
asf.jouwir.cn/044846.Doc
asf.jouwir.cn/862882.Doc
asf.jouwir.cn/042200.Doc
asf.jouwir.cn/862880.Doc
asf.jouwir.cn/646460.Doc
asd.jouwir.cn/804044.Doc
asd.jouwir.cn/864668.Doc
asd.jouwir.cn/046686.Doc
asd.jouwir.cn/088866.Doc
asd.jouwir.cn/460026.Doc
asd.jouwir.cn/444208.Doc
asd.jouwir.cn/484826.Doc
asd.jouwir.cn/088868.Doc
asd.jouwir.cn/666280.Doc
asd.jouwir.cn/228426.Doc
ass.jouwir.cn/468248.Doc
ass.jouwir.cn/246440.Doc
ass.jouwir.cn/822202.Doc
ass.jouwir.cn/600686.Doc
ass.jouwir.cn/808088.Doc
ass.jouwir.cn/484068.Doc
ass.jouwir.cn/393973.Doc
ass.jouwir.cn/282820.Doc
ass.jouwir.cn/042286.Doc
ass.jouwir.cn/317311.Doc
asa.jouwir.cn/002086.Doc
asa.jouwir.cn/266420.Doc
asa.jouwir.cn/840822.Doc
asa.jouwir.cn/688040.Doc
asa.jouwir.cn/886264.Doc
asa.jouwir.cn/917335.Doc
asa.jouwir.cn/624842.Doc
asa.jouwir.cn/640820.Doc
asa.jouwir.cn/622260.Doc
asa.jouwir.cn/468040.Doc
asp.jouwir.cn/066624.Doc
asp.jouwir.cn/022666.Doc
asp.jouwir.cn/000426.Doc
asp.jouwir.cn/444288.Doc
asp.jouwir.cn/648406.Doc
asp.jouwir.cn/080000.Doc
asp.jouwir.cn/060824.Doc
asp.jouwir.cn/858067.Doc
asp.jouwir.cn/422240.Doc
asp.jouwir.cn/395713.Doc
aso.jouwir.cn/008024.Doc
aso.jouwir.cn/405522.Doc
aso.jouwir.cn/628048.Doc
aso.jouwir.cn/084288.Doc
aso.jouwir.cn/064004.Doc
aso.jouwir.cn/822240.Doc
aso.jouwir.cn/222820.Doc
aso.jouwir.cn/208640.Doc
aso.jouwir.cn/842264.Doc
aso.jouwir.cn/444064.Doc
asi.jouwir.cn/642608.Doc
asi.jouwir.cn/082801.Doc
asi.jouwir.cn/820460.Doc
asi.jouwir.cn/420804.Doc
asi.jouwir.cn/808840.Doc
asi.jouwir.cn/000844.Doc
asi.jouwir.cn/606712.Doc
asi.jouwir.cn/082002.Doc
asi.jouwir.cn/640002.Doc
asi.jouwir.cn/880880.Doc
asu.jouwir.cn/024804.Doc
asu.jouwir.cn/408680.Doc
asu.jouwir.cn/204264.Doc
asu.jouwir.cn/020666.Doc
asu.jouwir.cn/844084.Doc
asu.jouwir.cn/888846.Doc
asu.jouwir.cn/606286.Doc
asu.jouwir.cn/060402.Doc
asu.jouwir.cn/024404.Doc
asu.jouwir.cn/800224.Doc
asy.jouwir.cn/822480.Doc
asy.jouwir.cn/957356.Doc
asy.jouwir.cn/286628.Doc
asy.jouwir.cn/659802.Doc
asy.jouwir.cn/048620.Doc
asy.jouwir.cn/222422.Doc
asy.jouwir.cn/923984.Doc
asy.jouwir.cn/793511.Doc
asy.jouwir.cn/444282.Doc
asy.jouwir.cn/262826.Doc
ast.jouwir.cn/846420.Doc
ast.jouwir.cn/802422.Doc
ast.jouwir.cn/460240.Doc
ast.jouwir.cn/082066.Doc
ast.jouwir.cn/113797.Doc
ast.jouwir.cn/755355.Doc
ast.jouwir.cn/668220.Doc
ast.jouwir.cn/264684.Doc
ast.jouwir.cn/804828.Doc
ast.jouwir.cn/246680.Doc
asr.jouwir.cn/204468.Doc
asr.jouwir.cn/406408.Doc
asr.jouwir.cn/846800.Doc
asr.jouwir.cn/242664.Doc
asr.jouwir.cn/408066.Doc
asr.jouwir.cn/359151.Doc
asr.jouwir.cn/204864.Doc
asr.jouwir.cn/420208.Doc
asr.jouwir.cn/486800.Doc
asr.jouwir.cn/004008.Doc
ase.jouwir.cn/200266.Doc
ase.jouwir.cn/226680.Doc
ase.jouwir.cn/826688.Doc
ase.jouwir.cn/240806.Doc
ase.jouwir.cn/244888.Doc
ase.jouwir.cn/606808.Doc
ase.jouwir.cn/622660.Doc
ase.jouwir.cn/062628.Doc
ase.jouwir.cn/824062.Doc
ase.jouwir.cn/246062.Doc
asw.jouwir.cn/400246.Doc
asw.jouwir.cn/266242.Doc
asw.jouwir.cn/286866.Doc
asw.jouwir.cn/042862.Doc
asw.jouwir.cn/602006.Doc
asw.jouwir.cn/042404.Doc
asw.jouwir.cn/046882.Doc
asw.jouwir.cn/862660.Doc
asw.jouwir.cn/244866.Doc
asw.jouwir.cn/404208.Doc
asq.jouwir.cn/082246.Doc
asq.jouwir.cn/666604.Doc
asq.jouwir.cn/400060.Doc
asq.jouwir.cn/802004.Doc
asq.jouwir.cn/480408.Doc
asq.jouwir.cn/202860.Doc
asq.jouwir.cn/042660.Doc
asq.jouwir.cn/048004.Doc
asq.jouwir.cn/424280.Doc
asq.jouwir.cn/628866.Doc
aam.jouwir.cn/955597.Doc
aam.jouwir.cn/282684.Doc
aam.jouwir.cn/408868.Doc
aam.jouwir.cn/648282.Doc
aam.jouwir.cn/282688.Doc
aam.jouwir.cn/486062.Doc
aam.jouwir.cn/604404.Doc
aam.jouwir.cn/046208.Doc
aam.jouwir.cn/826446.Doc
aam.jouwir.cn/280400.Doc
aan.jouwir.cn/080800.Doc
aan.jouwir.cn/268084.Doc
aan.jouwir.cn/620004.Doc
aan.jouwir.cn/662440.Doc
aan.jouwir.cn/462488.Doc
aan.jouwir.cn/046682.Doc
aan.jouwir.cn/804404.Doc
aan.jouwir.cn/808406.Doc
aan.jouwir.cn/824268.Doc
aan.jouwir.cn/351759.Doc
aab.jouwir.cn/826240.Doc
aab.jouwir.cn/006422.Doc
aab.jouwir.cn/844264.Doc
aab.jouwir.cn/648222.Doc
aab.jouwir.cn/604820.Doc
aab.jouwir.cn/448666.Doc
aab.jouwir.cn/022060.Doc
aab.jouwir.cn/866462.Doc
aab.jouwir.cn/800222.Doc
aab.jouwir.cn/044040.Doc
aav.jouwir.cn/042868.Doc
aav.jouwir.cn/155957.Doc
aav.jouwir.cn/426260.Doc
aav.jouwir.cn/862866.Doc
aav.jouwir.cn/260800.Doc
aav.jouwir.cn/264684.Doc
aav.jouwir.cn/622062.Doc
aav.jouwir.cn/640228.Doc
aav.jouwir.cn/220444.Doc
aav.jouwir.cn/684028.Doc
aac.jouwir.cn/646428.Doc
aac.jouwir.cn/860282.Doc
aac.jouwir.cn/028802.Doc
aac.jouwir.cn/684224.Doc
aac.jouwir.cn/004240.Doc
aac.jouwir.cn/888202.Doc
aac.jouwir.cn/226024.Doc
aac.jouwir.cn/551137.Doc
aac.jouwir.cn/011713.Doc
aac.jouwir.cn/842862.Doc
aax.jouwir.cn/402886.Doc
aax.jouwir.cn/460602.Doc
aax.jouwir.cn/047111.Doc
aax.jouwir.cn/957130.Doc
aax.jouwir.cn/826884.Doc
aax.jouwir.cn/688004.Doc
aax.jouwir.cn/200622.Doc
aax.jouwir.cn/484680.Doc
aax.jouwir.cn/282248.Doc
aax.jouwir.cn/684202.Doc
aaz.jouwir.cn/684862.Doc
aaz.jouwir.cn/666606.Doc
aaz.jouwir.cn/206848.Doc
aaz.jouwir.cn/648608.Doc
aaz.jouwir.cn/208024.Doc
aaz.jouwir.cn/975535.Doc
aaz.jouwir.cn/664066.Doc
aaz.jouwir.cn/062040.Doc
aaz.jouwir.cn/686886.Doc
aaz.jouwir.cn/448048.Doc
aal.jouwir.cn/806080.Doc
aal.jouwir.cn/080080.Doc
aal.jouwir.cn/866246.Doc
aal.jouwir.cn/351935.Doc
aal.jouwir.cn/531511.Doc
aal.jouwir.cn/200260.Doc
aal.jouwir.cn/454432.Doc
aal.jouwir.cn/428262.Doc
aal.jouwir.cn/842668.Doc
aal.jouwir.cn/006628.Doc
aak.jouwir.cn/844864.Doc
aak.jouwir.cn/062086.Doc
aak.jouwir.cn/844226.Doc
aak.jouwir.cn/222422.Doc
aak.jouwir.cn/482406.Doc
aak.jouwir.cn/240866.Doc
aak.jouwir.cn/806268.Doc
aak.jouwir.cn/088602.Doc
aak.jouwir.cn/644862.Doc
aak.jouwir.cn/202840.Doc
aaj.jouwir.cn/208486.Doc
aaj.jouwir.cn/284486.Doc
aaj.jouwir.cn/606068.Doc
aaj.jouwir.cn/266004.Doc
aaj.jouwir.cn/026284.Doc
aaj.jouwir.cn/844482.Doc
aaj.jouwir.cn/066008.Doc
aaj.jouwir.cn/082866.Doc
aaj.jouwir.cn/060888.Doc
aaj.jouwir.cn/006262.Doc
aah.jouwir.cn/028202.Doc
aah.jouwir.cn/820844.Doc
aah.jouwir.cn/886686.Doc
aah.jouwir.cn/684442.Doc
aah.jouwir.cn/824026.Doc
aah.jouwir.cn/040442.Doc
aah.jouwir.cn/460802.Doc
aah.jouwir.cn/842064.Doc
aah.jouwir.cn/000428.Doc
aah.jouwir.cn/684606.Doc
aag.jouwir.cn/202604.Doc
aag.jouwir.cn/266002.Doc
aag.jouwir.cn/244640.Doc
aag.jouwir.cn/448200.Doc
aag.jouwir.cn/244800.Doc
aag.jouwir.cn/682802.Doc
aag.jouwir.cn/884428.Doc
aag.jouwir.cn/020244.Doc
aag.jouwir.cn/228240.Doc
aag.jouwir.cn/828868.Doc
aaf.jouwir.cn/408060.Doc
aaf.jouwir.cn/442424.Doc
aaf.jouwir.cn/379313.Doc
aaf.jouwir.cn/240806.Doc
aaf.jouwir.cn/448286.Doc
aaf.jouwir.cn/646662.Doc
aaf.jouwir.cn/044822.Doc
aaf.jouwir.cn/820406.Doc
aaf.jouwir.cn/688444.Doc
aaf.jouwir.cn/688088.Doc
aad.jouwir.cn/000844.Doc
aad.jouwir.cn/828662.Doc
aad.jouwir.cn/844266.Doc
aad.jouwir.cn/800424.Doc
aad.jouwir.cn/242848.Doc
aad.jouwir.cn/684286.Doc
aad.jouwir.cn/402842.Doc
aad.jouwir.cn/202642.Doc
aad.jouwir.cn/884864.Doc
aad.jouwir.cn/640006.Doc
aas.jouwir.cn/060826.Doc
aas.jouwir.cn/440028.Doc
aas.jouwir.cn/682040.Doc
aas.jouwir.cn/828284.Doc
aas.jouwir.cn/284402.Doc
aas.jouwir.cn/424828.Doc
aas.jouwir.cn/488842.Doc
aas.jouwir.cn/864040.Doc
aas.jouwir.cn/202448.Doc
aas.jouwir.cn/024480.Doc
aaa.jouwir.cn/202622.Doc
aaa.jouwir.cn/628664.Doc
aaa.jouwir.cn/800202.Doc
aaa.jouwir.cn/004486.Doc
aaa.jouwir.cn/020848.Doc
aaa.jouwir.cn/284622.Doc
aaa.jouwir.cn/466468.Doc
aaa.jouwir.cn/244042.Doc
aaa.jouwir.cn/464280.Doc
