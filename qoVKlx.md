防附挠炙渍


=================================================================
 Python 多进程共享内存：高效数据交换方案
=================================================================

多进程间数据传递通常通过序列化（pickle）进行，开销较大。
共享内存允许多个进程直接访问同一块内存区域，实现零拷贝
数据交换。本文详解各种共享内存技术。

=================================================================
 一、multiprocessing.Value / Array：基础共享内存
=================================================================

import multiprocessing as mp
import time
import numpy as np

"""
Value 和 Array 使用 ctypes 在共享内存中分配数据。
内部使用 mmap 或系统共享内存 API。
"""

def value_array_demo():
    """使用 Value 和 Array 在进程间共享数据"""

    # 创建共享整数（使用 'i' 类型码）
    shared_counter = mp.Value('i', 0)

    # 创建共享双精度浮点数数组（10 个元素）
    shared_array = mp.Array('d', [0.0] * 10)

    def worker(counter, arr, worker_id: int):
        """工作进程：修改共享数据"""
        # 修改共享值
        with counter.get_lock():  # Value 自带锁
            counter.value += 1
            print(f"工作进程 {worker_id}: counter = {counter.value}")

        # 修改共享数组
        for i in range(len(arr)):
            arr[i] += worker_id * 10

    processes = []
    for i in range(4):
        p = mp.Process(target=worker, args=(
            shared_counter, shared_array, i
        ))
        p.start()
        processes.append(p)

    for p in processes:
        p.join()

    print(f"最终 counter: {shared_counter.value}")
    print(f"最终 array: {list(shared_array)}")

    # 常用类型码：
    # 'c': char, 'b': signed char, 'B': unsigned char
    # 'h': short, 'H': unsigned short
    # 'i': int, 'I': unsigned int
    # 'l': long, 'L': unsigned long
    # 'f': float, 'd': double

=================================================================
 二、multiprocessing.shared_memory（Python 3.8+）
=================================================================

"""
shared_memory 模块提供了更底层和灵活的共享内存接口。
支持任意大小的内存块，可以存储任何二进制数据。
"""

from multiprocessing import shared_memory

def shared_memory_basic():
    """使用 SharedMemory 在进程间共享数据"""

    # 创建 100 字节的共享内存块
    shm = shared_memory.SharedMemory(
        name="my_shared_mem",  # 命名（None 则自动生成）
        create=True,
        size=100,  # 字节数
    )

    try:
        # 通过缓冲区接口写入数据
        buffer = shm.buf  # memoryview 对象
        buffer[:5] = b"Hello"
        buffer[5:13] = b" World! "
        # 写入整数（4 字节，小端序）
        buffer[13:17] = (42).to_bytes(4, 'little')

        # 从另一个进程读取（在另一个进程中通过名称打开）
        def reader_process(shm_name: str):
            """读取共享内存的子进程"""
            existing_shm = shared_memory.SharedMemory(
                name=shm_name, create=False
            )
            data = existing_shm.buf[:17].tobytes()
            print(f"读取到: {data}")
            text = data[:13].decode()
            number = int.from_bytes(data[13:17], 'little')
            print(f"文本: {text}, 数字: {number}")
            existing_shm.close()

        p = mp.Process(target=reader_process, args=(shm.name,))
        p.start()
        p.join()

        print(f"原始数据: {shm.buf[:17].tobytes()}")
    finally:
        # 清理共享内存
        shm.close()
        shm.unlink()  # 删除共享内存块

=================================================================
 三、SharedMemoryManager：安全管理器
=================================================================

from multiprocessing.managers import SharedMemoryManager

def shared_memory_manager_demo():
    """
    SharedMemoryManager 自动管理共享内存的生命周期。
    所有创建的共享内存在管理器退出时自动清理。
    """
    with SharedMemoryManager() as smm:
        # 创建共享内存段
        shm_a = smm.SharedMemory(size=64)
        shm_b = smm.SharedMemory(size=128)

        print(f"共享内存名: {shm_a.name}")
        print(f"共享内存大小: {shm_a.size} 字节")

        # 使用 ShareableList 存储 Python 对象
        sl = smm.ShareableList(
            ["hello", 42, 3.14, b"bytes", None]
        )
        print(f"共享列表: {sl}")
        print(f"sl[0] = {sl[0]}, sl[1] = {sl[1]}")

        # 在子进程中读写
        def process_func(name: str):
            """子进程访问共享内存"""
            shm = shared_memory.SharedMemory(name=name)
            shm.buf[:6] = b"World!"
            shm.close()

        p = mp.Process(target=process_func, args=(shm_a.name,))
        p.start()
        p.join()

        print(f"子进程写入后: {shm_a.buf[:6].tobytes()}")

        # 管理器退出时自动清理所有共享内存

=================================================================
 四、共享内存 vs Queue 性能对比
=================================================================

def shared_memory_vs_queue():
    """
    对比共享内存和 Queue 在大数据量传输时的性能。
    共享内存避免了序列化/反序列化的开销。
    """
    DATA_SIZE = 1_000_000  # 100 万个整数
    data = list(range(DATA_SIZE))

    # === 使用 Queue 传输 ===
    start = time.perf_counter()
    q = mp.Queue()

    def queue_sender(q, data):
        q.put(data)

    def queue_receiver(q):
        _ = q.get()

    p1 = mp.Process(target=queue_sender, args=(q, data))
    p2 = mp.Process(target=queue_receiver, args=(q,))

    p1.start()
    p2.start()
    p1.join()
    p2.join()
    queue_time = time.perf_counter() - start

    # === 使用共享内存传输 ===
    start = time.perf_counter()
    import array

    # 将数据写入共享内存
    arr = array.array('i', data)
    shm = shared_memory.SharedMemory(
        create=True, size=len(arr) * arr.itemsize
    )
    shm.buf[:len(arr) * arr.itemsize] = bytes(arr)

    def shm_receiver(shm_name, size):
        ex_shm = shared_memory.SharedMemory(name=shm_name)
        result = array.array('i')
        result.frombytes(ex_shm.buf[:size])
        _ = result.tolist()
        ex_shm.close()

    p = mp.Process(target=shm_receiver, args=(shm.name, len(arr) * arr.itemsize))
    p.start()
    p.join()
    shm_time = time.perf_counter() - start

    shm.close()
    shm.unlink()

    print(f"=== 性能对比（{DATA_SIZE} 个整数） ===")
    print(f"Queue 传输: {queue_time:.3f}s")
    print(f"共享内存: {shm_time:.3f}s")
    print(f"加速比: {queue_time / shm_time:.1f}x")
    # 数据量越大，共享内存的优势越明显

=================================================================
 五、在共享内存中使用 NumPy
=================================================================

def shared_memory_numpy():
    """
    在共享内存中创建 numpy 数组。
    多个进程可以零拷贝地操作同一份数据。
    """
    with SharedMemoryManager() as smm:
        # 创建共享内存
        shape = (1000, 1000)  # 1000x1000 的矩阵
        dtype = np.float64
        # 计算需要的字节数
        nbytes = np.prod(shape) * np.dtype(dtype).itemsize
        shm = smm.SharedMemory(size=nbytes)

        # 创建 numpy 数组指向共享内存
        shared_array = np.ndarray(
            shape, dtype=dtype, buffer=shm.buf
        )

        # 初始化数据
        shared_array[:] = np.random.rand(*shape)
        print(f"共享数组形状: {shared_array.shape}")
        print(f"共享数组大小: {nbytes / 1024 / 1024:.2f} MB")

        def compute_sum(shm_name, shape, dtype):
            """在子进程中计算数组和"""
            ex_shm = shared_memory.SharedMemory(name=shm_name)
            arr = np.ndarray(shape, dtype=dtype, buffer=ex_shm.buf)
            total = np.sum(arr)
            print(f"子进程计算结果: {total:.2f}")
            ex_shm.close()

        p = mp.Process(
            target=compute_sum,
            args=(shm.name, shape, dtype),
        )
        p.start()
        p.join()

        print(f"主进程验证: {np.sum(shared_array):.2f}")

=================================================================
 六、数据同步：避免竞态条件
=================================================================

"""
共享内存本身不提供同步机制。当多个进程同时修改同一块
内存时，需要使用 Lock 来避免竞态条件。
"""

def race_condition_demo():
    """演示共享内存的竞态条件及解决方案"""
    shm = shared_memory.SharedMemory(
        create=True, size=8  # 存储一个 64 位整数
    )
    lock = mp.Lock()  # 进程间锁
    NUM_INCREMENTS = 10000

    # 初始化计数器
    shm.buf[:8] = (0).to_bytes(8, 'little')

    def unsafe_increment(shm_name):
        """不安全的递增（没有锁）"""
        ex_shm = shared_memory.SharedMemory(name=shm_name)
        for _ in range(NUM_INCREMENTS):
            # 读取-修改-写入（非原子操作）
            val = int.from_bytes(ex_shm.buf[:8], 'little')
            val += 1
            ex_shm.buf[:8] = val.to_bytes(8, 'little')
        ex_shm.close()

    def safe_increment(shm_name, lock):
        """安全的递增（使用锁）"""
        ex_shm = shared_memory.SharedMemory(name=shm_name)
        for _ in range(NUM_INCREMENTS):
            with lock:
                val = int.from_bytes(ex_shm.buf[:8], 'little')
                val += 1
                ex_shm.buf[:8] = val.to_bytes(8, 'little')
        ex_shm.close()

    # 启动多个进程（不使用锁）
    procs = [
        mp.Process(target=unsafe_increment, args=(shm.name,))
        for _ in range(4)
    ]
    for p in procs: p.start()
    for p in procs: p.join()

    result = int.from_bytes(shm.buf[:8], 'little')
    expected = NUM_INCREMENTS * 4
    print(f"无锁: 实际={result}, 期望={expected}, 误差={expected-result}")

    # 重置并测试有锁版本
    shm.buf[:8] = (0).to_bytes(8, 'little')
    procs = [
        mp.Process(target=safe_increment, args=(shm.name, lock))
        for _ in range(4)
    ]
    for p in procs: p.start()
    for p in procs: p.join()

    result = int.from_bytes(shm.buf[:8], 'little')
    print(f"有锁: 实际={result}, 期望={expected}")

    shm.close()
    shm.unlink()

=================================================================
 七、总结
=================================================================

# 1. Value/Array：简单的 ctypes 共享内存，自带锁
# 2. SharedMemory（3.8+）：灵活的低层共享内存 API
# 3. SharedMemoryManager：自动管理共享内存生命周期
# 4. 大数据量时共享内存比 Queue 快（避免序列化）
# 5. numpy 数组可零拷贝映射到共享内存
# 6. 共享内存需要手动同步（Lock）避免竞态
# 7. 使用完毕后必须 unlink() 清理系统资源
# 8. SharedMemoryManager 的 ShareableList 可存混合类型

if __name__ == "__main__":
    shared_memory_vs_queue()

垂亩抵急急拿陶囟磷嚎亚谙厩诤泳

m.eop.hmybk.cn/75171.Doc
m.eop.hmybk.cn/77997.Doc
m.eop.hmybk.cn/11913.Doc
m.eop.hmybk.cn/11371.Doc
m.eop.hmybk.cn/40868.Doc
m.eop.hmybk.cn/33555.Doc
m.eop.hmybk.cn/04400.Doc
m.eop.hmybk.cn/82684.Doc
m.eop.hmybk.cn/08080.Doc
m.eop.hmybk.cn/66004.Doc
m.eop.hmybk.cn/20264.Doc
m.eop.hmybk.cn/59911.Doc
m.eop.hmybk.cn/91951.Doc
m.eop.hmybk.cn/99773.Doc
m.eop.hmybk.cn/79319.Doc
m.eoo.hmybk.cn/39775.Doc
m.eoo.hmybk.cn/22268.Doc
m.eoo.hmybk.cn/73111.Doc
m.eoo.hmybk.cn/51375.Doc
m.eoo.hmybk.cn/24668.Doc
m.eoo.hmybk.cn/91593.Doc
m.eoo.hmybk.cn/15171.Doc
m.eoo.hmybk.cn/53713.Doc
m.eoo.hmybk.cn/35971.Doc
m.eoo.hmybk.cn/35779.Doc
m.eoo.hmybk.cn/93955.Doc
m.eoo.hmybk.cn/42404.Doc
m.eoo.hmybk.cn/37937.Doc
m.eoo.hmybk.cn/97539.Doc
m.eoo.hmybk.cn/33595.Doc
m.eoo.hmybk.cn/80620.Doc
m.eoo.hmybk.cn/79733.Doc
m.eoo.hmybk.cn/20644.Doc
m.eoo.hmybk.cn/13379.Doc
m.eoo.hmybk.cn/86864.Doc
m.eoi.hmybk.cn/68488.Doc
m.eoi.hmybk.cn/62006.Doc
m.eoi.hmybk.cn/82886.Doc
m.eoi.hmybk.cn/84462.Doc
m.eoi.hmybk.cn/99395.Doc
m.eoi.hmybk.cn/53111.Doc
m.eoi.hmybk.cn/31353.Doc
m.eoi.hmybk.cn/73913.Doc
m.eoi.hmybk.cn/88046.Doc
m.eoi.hmybk.cn/79157.Doc
m.eoi.hmybk.cn/93593.Doc
m.eoi.hmybk.cn/17913.Doc
m.eoi.hmybk.cn/79177.Doc
m.eoi.hmybk.cn/91917.Doc
m.eoi.hmybk.cn/84840.Doc
m.eoi.hmybk.cn/73375.Doc
m.eoi.hmybk.cn/11733.Doc
m.eoi.hmybk.cn/91359.Doc
m.eoi.hmybk.cn/00404.Doc
m.eoi.hmybk.cn/93711.Doc
m.eou.hmybk.cn/04688.Doc
m.eou.hmybk.cn/73977.Doc
m.eou.hmybk.cn/48440.Doc
m.eou.hmybk.cn/11731.Doc
m.eou.hmybk.cn/79759.Doc
m.eou.hmybk.cn/55159.Doc
m.eou.hmybk.cn/00202.Doc
m.eou.hmybk.cn/73379.Doc
m.eou.hmybk.cn/40448.Doc
m.eou.hmybk.cn/42042.Doc
m.eou.hmybk.cn/02282.Doc
m.eou.hmybk.cn/77573.Doc
m.eou.hmybk.cn/11799.Doc
m.eou.hmybk.cn/19333.Doc
m.eou.hmybk.cn/51115.Doc
m.eou.hmybk.cn/93557.Doc
m.eou.hmybk.cn/26820.Doc
m.eou.hmybk.cn/71159.Doc
m.eou.hmybk.cn/99531.Doc
m.eou.hmybk.cn/33795.Doc
m.eoy.hmybk.cn/75717.Doc
m.eoy.hmybk.cn/39317.Doc
m.eoy.hmybk.cn/35973.Doc
m.eoy.hmybk.cn/66462.Doc
m.eoy.hmybk.cn/79175.Doc
m.eoy.hmybk.cn/31391.Doc
m.eoy.hmybk.cn/35919.Doc
m.eoy.hmybk.cn/17755.Doc
m.eoy.hmybk.cn/77977.Doc
m.eoy.hmybk.cn/08224.Doc
m.eoy.hmybk.cn/13795.Doc
m.eoy.hmybk.cn/51137.Doc
m.eoy.hmybk.cn/73779.Doc
m.eoy.hmybk.cn/71775.Doc
m.eoy.hmybk.cn/55159.Doc
m.eoy.hmybk.cn/22464.Doc
m.eoy.hmybk.cn/99513.Doc
m.eoy.hmybk.cn/77599.Doc
m.eoy.hmybk.cn/44068.Doc
m.eoy.hmybk.cn/40462.Doc
m.eot.hmybk.cn/66420.Doc
m.eot.hmybk.cn/46280.Doc
m.eot.hmybk.cn/64244.Doc
m.eot.hmybk.cn/11135.Doc
m.eot.hmybk.cn/80662.Doc
m.eot.hmybk.cn/37771.Doc
m.eot.hmybk.cn/75795.Doc
m.eot.hmybk.cn/37171.Doc
m.eot.hmybk.cn/51959.Doc
m.eot.hmybk.cn/02426.Doc
m.eot.hmybk.cn/62464.Doc
m.eot.hmybk.cn/62464.Doc
m.eot.hmybk.cn/33797.Doc
m.eot.hmybk.cn/44424.Doc
m.eot.hmybk.cn/19751.Doc
m.eot.hmybk.cn/33395.Doc
m.eot.hmybk.cn/19339.Doc
m.eot.hmybk.cn/73533.Doc
m.eot.hmybk.cn/33191.Doc
m.eot.hmybk.cn/9.Doc
m.eor.hmybk.cn/71391.Doc
m.eor.hmybk.cn/13597.Doc
m.eor.hmybk.cn/24020.Doc
m.eor.hmybk.cn/00642.Doc
m.eor.hmybk.cn/73975.Doc
m.eor.hmybk.cn/19359.Doc
m.eor.hmybk.cn/11591.Doc
m.eor.hmybk.cn/55313.Doc
m.eor.hmybk.cn/48064.Doc
m.eor.hmybk.cn/33195.Doc
m.eor.hmybk.cn/37157.Doc
m.eor.hmybk.cn/73137.Doc
m.eor.hmybk.cn/57355.Doc
m.eor.hmybk.cn/93971.Doc
m.eor.hmybk.cn/31531.Doc
m.eor.hmybk.cn/95973.Doc
m.eor.hmybk.cn/35357.Doc
m.eor.hmybk.cn/28080.Doc
m.eor.hmybk.cn/93797.Doc
m.eor.hmybk.cn/97771.Doc
m.eoe.hmybk.cn/73993.Doc
m.eoe.hmybk.cn/79313.Doc
m.eoe.hmybk.cn/35539.Doc
m.eoe.hmybk.cn/48422.Doc
m.eoe.hmybk.cn/44602.Doc
m.eoe.hmybk.cn/73919.Doc
m.eoe.hmybk.cn/31955.Doc
m.eoe.hmybk.cn/91171.Doc
m.eoe.hmybk.cn/20826.Doc
m.eoe.hmybk.cn/02484.Doc
m.eoe.hmybk.cn/79195.Doc
m.eoe.hmybk.cn/60086.Doc
m.eoe.hmybk.cn/26220.Doc
m.eoe.hmybk.cn/55919.Doc
m.eoe.hmybk.cn/68684.Doc
m.eoe.hmybk.cn/59571.Doc
m.eoe.hmybk.cn/44866.Doc
m.eoe.hmybk.cn/06840.Doc
m.eoe.hmybk.cn/84248.Doc
m.eoe.hmybk.cn/82400.Doc
m.eow.hmybk.cn/62822.Doc
m.eow.hmybk.cn/17755.Doc
m.eow.hmybk.cn/35393.Doc
m.eow.hmybk.cn/73973.Doc
m.eow.hmybk.cn/75755.Doc
m.eow.hmybk.cn/19795.Doc
m.eow.hmybk.cn/24660.Doc
m.eow.hmybk.cn/75953.Doc
m.eow.hmybk.cn/66606.Doc
m.eow.hmybk.cn/11555.Doc
m.eow.hmybk.cn/55793.Doc
m.eow.hmybk.cn/17999.Doc
m.eow.hmybk.cn/31553.Doc
m.eow.hmybk.cn/99193.Doc
m.eow.hmybk.cn/42206.Doc
m.eow.hmybk.cn/55935.Doc
m.eow.hmybk.cn/71399.Doc
m.eow.hmybk.cn/11559.Doc
m.eow.hmybk.cn/17193.Doc
m.eow.hmybk.cn/31959.Doc
m.eoq.hmybk.cn/11779.Doc
m.eoq.hmybk.cn/93135.Doc
m.eoq.hmybk.cn/28644.Doc
m.eoq.hmybk.cn/73979.Doc
m.eoq.hmybk.cn/35539.Doc
m.eoq.hmybk.cn/06226.Doc
m.eoq.hmybk.cn/00464.Doc
m.eoq.hmybk.cn/95313.Doc
m.eoq.hmybk.cn/77113.Doc
m.eoq.hmybk.cn/62488.Doc
m.eoq.hmybk.cn/68486.Doc
m.eoq.hmybk.cn/48206.Doc
m.eoq.hmybk.cn/59733.Doc
m.eoq.hmybk.cn/22888.Doc
m.eoq.hmybk.cn/44422.Doc
m.eoq.hmybk.cn/51937.Doc
m.eoq.hmybk.cn/79939.Doc
m.eoq.hmybk.cn/84860.Doc
m.eoq.hmybk.cn/31795.Doc
m.eoq.hmybk.cn/55515.Doc
m.eim.hmybk.cn/57531.Doc
m.eim.hmybk.cn/51739.Doc
m.eim.hmybk.cn/57151.Doc
m.eim.hmybk.cn/00042.Doc
m.eim.hmybk.cn/97771.Doc
m.eim.hmybk.cn/88028.Doc
m.eim.hmybk.cn/60240.Doc
m.eim.hmybk.cn/77975.Doc
m.eim.hmybk.cn/35553.Doc
m.eim.hmybk.cn/59313.Doc
m.eim.hmybk.cn/26828.Doc
m.eim.hmybk.cn/28884.Doc
m.eim.hmybk.cn/37931.Doc
m.eim.hmybk.cn/77333.Doc
m.eim.hmybk.cn/08008.Doc
m.eim.hmybk.cn/39157.Doc
m.eim.hmybk.cn/37317.Doc
m.eim.hmybk.cn/35535.Doc
m.eim.hmybk.cn/53317.Doc
m.eim.hmybk.cn/28846.Doc
m.ein.hmybk.cn/73797.Doc
m.ein.hmybk.cn/42660.Doc
m.ein.hmybk.cn/64622.Doc
m.ein.hmybk.cn/66026.Doc
m.ein.hmybk.cn/33319.Doc
m.ein.hmybk.cn/20806.Doc
m.ein.hmybk.cn/06602.Doc
m.ein.hmybk.cn/13995.Doc
m.ein.hmybk.cn/48004.Doc
m.ein.hmybk.cn/19515.Doc
m.ein.hmybk.cn/35553.Doc
m.ein.hmybk.cn/42404.Doc
m.ein.hmybk.cn/35559.Doc
m.ein.hmybk.cn/57139.Doc
m.ein.hmybk.cn/57339.Doc
m.ein.hmybk.cn/15373.Doc
m.ein.hmybk.cn/37919.Doc
m.ein.hmybk.cn/51137.Doc
m.ein.hmybk.cn/62464.Doc
m.ein.hmybk.cn/84460.Doc
m.eib.hmybk.cn/37311.Doc
m.eib.hmybk.cn/99199.Doc
m.eib.hmybk.cn/04222.Doc
m.eib.hmybk.cn/71575.Doc
m.eib.hmybk.cn/82004.Doc
m.eib.hmybk.cn/15515.Doc
m.eib.hmybk.cn/22282.Doc
m.eib.hmybk.cn/26004.Doc
m.eib.hmybk.cn/24220.Doc
m.eib.hmybk.cn/95771.Doc
m.eib.hmybk.cn/95571.Doc
m.eib.hmybk.cn/31351.Doc
m.eib.hmybk.cn/77115.Doc
m.eib.hmybk.cn/13133.Doc
m.eib.hmybk.cn/08688.Doc
m.eib.hmybk.cn/82442.Doc
m.eib.hmybk.cn/57337.Doc
m.eib.hmybk.cn/99599.Doc
m.eib.hmybk.cn/97339.Doc
m.eib.hmybk.cn/64662.Doc
m.eiv.hmybk.cn/11399.Doc
m.eiv.hmybk.cn/91397.Doc
m.eiv.hmybk.cn/06068.Doc
m.eiv.hmybk.cn/93515.Doc
m.eiv.hmybk.cn/28082.Doc
m.eiv.hmybk.cn/24246.Doc
m.eiv.hmybk.cn/59115.Doc
m.eiv.hmybk.cn/40828.Doc
m.eiv.hmybk.cn/97331.Doc
m.eiv.hmybk.cn/31173.Doc
m.eiv.hmybk.cn/91977.Doc
m.eiv.hmybk.cn/31731.Doc
m.eiv.hmybk.cn/95575.Doc
m.eiv.hmybk.cn/79715.Doc
m.eiv.hmybk.cn/40826.Doc
m.eiv.hmybk.cn/13935.Doc
m.eiv.hmybk.cn/53535.Doc
m.eiv.hmybk.cn/79173.Doc
m.eiv.hmybk.cn/71555.Doc
m.eiv.hmybk.cn/51939.Doc
m.eic.hmybk.cn/17773.Doc
m.eic.hmybk.cn/42026.Doc
m.eic.hmybk.cn/31737.Doc
m.eic.hmybk.cn/24620.Doc
m.eic.hmybk.cn/66642.Doc
m.eic.hmybk.cn/73771.Doc
m.eic.hmybk.cn/64228.Doc
m.eic.hmybk.cn/59391.Doc
m.eic.hmybk.cn/17391.Doc
m.eic.hmybk.cn/20088.Doc
m.eic.hmybk.cn/08208.Doc
m.eic.hmybk.cn/44004.Doc
m.eic.hmybk.cn/40686.Doc
m.eic.hmybk.cn/95971.Doc
m.eic.hmybk.cn/35119.Doc
m.eic.hmybk.cn/59171.Doc
m.eic.hmybk.cn/97357.Doc
m.eic.hmybk.cn/17917.Doc
m.eic.hmybk.cn/17919.Doc
m.eic.hmybk.cn/46048.Doc
m.eix.hmybk.cn/46404.Doc
m.eix.hmybk.cn/33595.Doc
m.eix.hmybk.cn/26222.Doc
m.eix.hmybk.cn/51715.Doc
m.eix.hmybk.cn/22424.Doc
m.eix.hmybk.cn/28842.Doc
m.eix.hmybk.cn/79771.Doc
m.eix.hmybk.cn/22860.Doc
m.eix.hmybk.cn/37731.Doc
m.eix.hmybk.cn/33179.Doc
m.eix.hmybk.cn/39117.Doc
m.eix.hmybk.cn/15377.Doc
m.eix.hmybk.cn/06284.Doc
m.eix.hmybk.cn/35999.Doc
m.eix.hmybk.cn/82006.Doc
m.eix.hmybk.cn/66248.Doc
m.eix.hmybk.cn/86842.Doc
m.eix.hmybk.cn/97179.Doc
m.eix.hmybk.cn/17111.Doc
m.eix.hmybk.cn/53533.Doc
m.eiz.hmybk.cn/79197.Doc
m.eiz.hmybk.cn/71997.Doc
m.eiz.hmybk.cn/31195.Doc
m.eiz.hmybk.cn/17115.Doc
m.eiz.hmybk.cn/51359.Doc
m.eiz.hmybk.cn/84822.Doc
m.eiz.hmybk.cn/95737.Doc
m.eiz.hmybk.cn/95531.Doc
m.eiz.hmybk.cn/95957.Doc
m.eiz.hmybk.cn/79151.Doc
m.eiz.hmybk.cn/62240.Doc
m.eiz.hmybk.cn/19151.Doc
m.eiz.hmybk.cn/08420.Doc
m.eiz.hmybk.cn/11913.Doc
m.eiz.hmybk.cn/95373.Doc
m.eiz.hmybk.cn/77791.Doc
m.eiz.hmybk.cn/79711.Doc
m.eiz.hmybk.cn/15933.Doc
m.eiz.hmybk.cn/93553.Doc
m.eiz.hmybk.cn/17995.Doc
m.eil.hmybk.cn/59931.Doc
m.eil.hmybk.cn/62660.Doc
m.eil.hmybk.cn/11331.Doc
m.eil.hmybk.cn/57975.Doc
m.eil.hmybk.cn/35753.Doc
m.eil.hmybk.cn/62688.Doc
m.eil.hmybk.cn/53351.Doc
m.eil.hmybk.cn/86024.Doc
m.eil.hmybk.cn/19179.Doc
m.eil.hmybk.cn/79151.Doc
m.eil.hmybk.cn/48648.Doc
m.eil.hmybk.cn/88284.Doc
m.eil.hmybk.cn/53591.Doc
m.eil.hmybk.cn/93333.Doc
m.eil.hmybk.cn/37753.Doc
m.eil.hmybk.cn/99131.Doc
m.eil.hmybk.cn/39399.Doc
m.eil.hmybk.cn/73331.Doc
m.eil.hmybk.cn/84208.Doc
m.eil.hmybk.cn/35773.Doc
m.eik.hmybk.cn/26806.Doc
m.eik.hmybk.cn/57397.Doc
m.eik.hmybk.cn/35931.Doc
m.eik.hmybk.cn/02440.Doc
m.eik.hmybk.cn/11195.Doc
m.eik.hmybk.cn/75515.Doc
m.eik.hmybk.cn/40420.Doc
m.eik.hmybk.cn/35131.Doc
m.eik.hmybk.cn/33311.Doc
m.eik.hmybk.cn/19393.Doc
m.eik.hmybk.cn/35353.Doc
m.eik.hmybk.cn/79159.Doc
m.eik.hmybk.cn/26620.Doc
m.eik.hmybk.cn/73933.Doc
m.eik.hmybk.cn/31757.Doc
m.eik.hmybk.cn/62824.Doc
m.eik.hmybk.cn/97155.Doc
m.eik.hmybk.cn/11997.Doc
m.eik.hmybk.cn/13739.Doc
m.eik.hmybk.cn/31511.Doc
m.eij.hmybk.cn/35153.Doc
m.eij.hmybk.cn/44488.Doc
m.eij.hmybk.cn/42466.Doc
m.eij.hmybk.cn/97779.Doc
m.eij.hmybk.cn/60642.Doc
m.eij.hmybk.cn/31773.Doc
m.eij.hmybk.cn/79937.Doc
m.eij.hmybk.cn/75719.Doc
m.eij.hmybk.cn/82662.Doc
m.eij.hmybk.cn/88068.Doc
m.eij.hmybk.cn/20004.Doc
m.eij.hmybk.cn/35717.Doc
m.eij.hmybk.cn/51919.Doc
m.eij.hmybk.cn/15331.Doc
m.eij.hmybk.cn/93775.Doc
m.eij.hmybk.cn/42604.Doc
m.eij.hmybk.cn/60604.Doc
m.eij.hmybk.cn/95133.Doc
m.eij.hmybk.cn/66428.Doc
m.eij.hmybk.cn/40628.Doc
