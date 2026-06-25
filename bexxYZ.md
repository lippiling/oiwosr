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

dcd.baoquan026.cn/28646.Doc
dcd.baoquan026.cn/66608.Doc
dcd.baoquan026.cn/20808.Doc
dcd.baoquan026.cn/26624.Doc
dcd.baoquan026.cn/42882.Doc
dcd.baoquan026.cn/28400.Doc
dcd.baoquan026.cn/88262.Doc
dcd.baoquan026.cn/44844.Doc
dcd.baoquan026.cn/62426.Doc
dcd.baoquan026.cn/60486.Doc
dcs.baoquan026.cn/20828.Doc
dcs.baoquan026.cn/26426.Doc
dcs.baoquan026.cn/24220.Doc
dcs.baoquan026.cn/20862.Doc
dcs.baoquan026.cn/24820.Doc
dcs.baoquan026.cn/64820.Doc
dcs.baoquan026.cn/02602.Doc
dcs.baoquan026.cn/82406.Doc
dcs.baoquan026.cn/82602.Doc
dcs.baoquan026.cn/64420.Doc
dca.baoquan026.cn/08426.Doc
dca.baoquan026.cn/86886.Doc
dca.baoquan026.cn/84004.Doc
dca.baoquan026.cn/26844.Doc
dca.baoquan026.cn/00864.Doc
dca.baoquan026.cn/68660.Doc
dca.baoquan026.cn/26028.Doc
dca.baoquan026.cn/06886.Doc
dca.baoquan026.cn/48460.Doc
dca.baoquan026.cn/46082.Doc
dcp.baoquan026.cn/62028.Doc
dcp.baoquan026.cn/20044.Doc
dcp.baoquan026.cn/24224.Doc
dcp.baoquan026.cn/64622.Doc
dcp.baoquan026.cn/86442.Doc
dcp.baoquan026.cn/46426.Doc
dcp.baoquan026.cn/22628.Doc
dcp.baoquan026.cn/26222.Doc
dcp.baoquan026.cn/06844.Doc
dcp.baoquan026.cn/88226.Doc
dco.baoquan026.cn/08244.Doc
dco.baoquan026.cn/08448.Doc
dco.baoquan026.cn/46802.Doc
dco.baoquan026.cn/86684.Doc
dco.baoquan026.cn/26886.Doc
dco.baoquan026.cn/64688.Doc
dco.baoquan026.cn/60408.Doc
dco.baoquan026.cn/20062.Doc
dco.baoquan026.cn/62400.Doc
dco.baoquan026.cn/48266.Doc
dci.baoquan026.cn/20280.Doc
dci.baoquan026.cn/46662.Doc
dci.baoquan026.cn/42422.Doc
dci.baoquan026.cn/00684.Doc
dci.baoquan026.cn/60048.Doc
dci.baoquan026.cn/46428.Doc
dci.baoquan026.cn/48684.Doc
dci.baoquan026.cn/02820.Doc
dci.baoquan026.cn/86240.Doc
dci.baoquan026.cn/24008.Doc
dcu.baoquan026.cn/04682.Doc
dcu.baoquan026.cn/02846.Doc
dcu.baoquan026.cn/20426.Doc
dcu.baoquan026.cn/46200.Doc
dcu.baoquan026.cn/20224.Doc
dcu.baoquan026.cn/35773.Doc
dcu.baoquan026.cn/28666.Doc
dcu.baoquan026.cn/28040.Doc
dcu.baoquan026.cn/88860.Doc
dcu.baoquan026.cn/06824.Doc
dcy.baoquan026.cn/26888.Doc
dcy.baoquan026.cn/62424.Doc
dcy.baoquan026.cn/68088.Doc
dcy.baoquan026.cn/57793.Doc
dcy.baoquan026.cn/75137.Doc
dcy.baoquan026.cn/24646.Doc
dcy.baoquan026.cn/22668.Doc
dcy.baoquan026.cn/48840.Doc
dcy.baoquan026.cn/62460.Doc
dcy.baoquan026.cn/86022.Doc
dct.baoquan026.cn/22828.Doc
dct.baoquan026.cn/42200.Doc
dct.baoquan026.cn/60480.Doc
dct.baoquan026.cn/86224.Doc
dct.baoquan026.cn/28006.Doc
dct.baoquan026.cn/84244.Doc
dct.baoquan026.cn/08088.Doc
dct.baoquan026.cn/00482.Doc
dct.baoquan026.cn/22866.Doc
dct.baoquan026.cn/82044.Doc
dcr.baoquan026.cn/44408.Doc
dcr.baoquan026.cn/62664.Doc
dcr.baoquan026.cn/64468.Doc
dcr.baoquan026.cn/62082.Doc
dcr.baoquan026.cn/62048.Doc
dcr.baoquan026.cn/06860.Doc
dcr.baoquan026.cn/28260.Doc
dcr.baoquan026.cn/60484.Doc
dcr.baoquan026.cn/04008.Doc
dcr.baoquan026.cn/60880.Doc
dce.baoquan026.cn/68422.Doc
dce.baoquan026.cn/66400.Doc
dce.baoquan026.cn/82686.Doc
dce.baoquan026.cn/00060.Doc
dce.baoquan026.cn/86084.Doc
dce.baoquan026.cn/02080.Doc
dce.baoquan026.cn/64486.Doc
dce.baoquan026.cn/57177.Doc
dce.baoquan026.cn/04848.Doc
dce.baoquan026.cn/26664.Doc
dcw.baoquan026.cn/15115.Doc
dcw.baoquan026.cn/17171.Doc
dcw.baoquan026.cn/44824.Doc
dcw.baoquan026.cn/84440.Doc
dcw.baoquan026.cn/48020.Doc
dcw.baoquan026.cn/00668.Doc
dcw.baoquan026.cn/42646.Doc
dcw.baoquan026.cn/11177.Doc
dcw.baoquan026.cn/26240.Doc
dcw.baoquan026.cn/73931.Doc
dcq.baoquan026.cn/68622.Doc
dcq.baoquan026.cn/24206.Doc
dcq.baoquan026.cn/66424.Doc
dcq.baoquan026.cn/71771.Doc
dcq.baoquan026.cn/48642.Doc
dcq.baoquan026.cn/06280.Doc
dcq.baoquan026.cn/00826.Doc
dcq.baoquan026.cn/22646.Doc
dcq.baoquan026.cn/24640.Doc
dcq.baoquan026.cn/20268.Doc
dxm.baoquan026.cn/00868.Doc
dxm.baoquan026.cn/37151.Doc
dxm.baoquan026.cn/40044.Doc
dxm.baoquan026.cn/08646.Doc
dxm.baoquan026.cn/24688.Doc
dxm.baoquan026.cn/24002.Doc
dxm.baoquan026.cn/60026.Doc
dxm.baoquan026.cn/86446.Doc
dxm.baoquan026.cn/26626.Doc
dxm.baoquan026.cn/66062.Doc
dxn.baoquan026.cn/84688.Doc
dxn.baoquan026.cn/62622.Doc
dxn.baoquan026.cn/44482.Doc
dxn.baoquan026.cn/86080.Doc
dxn.baoquan026.cn/40808.Doc
dxn.baoquan026.cn/91519.Doc
dxn.baoquan026.cn/84468.Doc
dxn.baoquan026.cn/77355.Doc
dxn.baoquan026.cn/68682.Doc
dxn.baoquan026.cn/02264.Doc
dxb.baoquan026.cn/22206.Doc
dxb.baoquan026.cn/24202.Doc
dxb.baoquan026.cn/08248.Doc
dxb.baoquan026.cn/06888.Doc
dxb.baoquan026.cn/88044.Doc
dxb.baoquan026.cn/06220.Doc
dxb.baoquan026.cn/88466.Doc
dxb.baoquan026.cn/48464.Doc
dxb.baoquan026.cn/84808.Doc
dxb.baoquan026.cn/08424.Doc
dxv.baoquan026.cn/82242.Doc
dxv.baoquan026.cn/66888.Doc
dxv.baoquan026.cn/46880.Doc
dxv.baoquan026.cn/55959.Doc
dxv.baoquan026.cn/08208.Doc
dxv.baoquan026.cn/86462.Doc
dxv.baoquan026.cn/06448.Doc
dxv.baoquan026.cn/26248.Doc
dxv.baoquan026.cn/88688.Doc
dxv.baoquan026.cn/51799.Doc
dxc.baoquan026.cn/80080.Doc
dxc.baoquan026.cn/64668.Doc
dxc.baoquan026.cn/08486.Doc
dxc.baoquan026.cn/13793.Doc
dxc.baoquan026.cn/84288.Doc
dxc.baoquan026.cn/64462.Doc
dxc.baoquan026.cn/62440.Doc
dxc.baoquan026.cn/42068.Doc
dxc.baoquan026.cn/88206.Doc
dxc.baoquan026.cn/46888.Doc
dxx.baoquan026.cn/02624.Doc
dxx.baoquan026.cn/40426.Doc
dxx.baoquan026.cn/00088.Doc
dxx.baoquan026.cn/02400.Doc
dxx.baoquan026.cn/46884.Doc
dxx.baoquan026.cn/88224.Doc
dxx.baoquan026.cn/51197.Doc
dxx.baoquan026.cn/53951.Doc
dxx.baoquan026.cn/97315.Doc
dxx.baoquan026.cn/60028.Doc
dxz.baoquan026.cn/66424.Doc
dxz.baoquan026.cn/40806.Doc
dxz.baoquan026.cn/22604.Doc
dxz.baoquan026.cn/08004.Doc
dxz.baoquan026.cn/00466.Doc
dxz.baoquan026.cn/48240.Doc
dxz.baoquan026.cn/40026.Doc
dxz.baoquan026.cn/26484.Doc
dxz.baoquan026.cn/26800.Doc
dxz.baoquan026.cn/31337.Doc
dxl.baoquan026.cn/26242.Doc
dxl.baoquan026.cn/62480.Doc
dxl.baoquan026.cn/64886.Doc
dxl.baoquan026.cn/88224.Doc
dxl.baoquan026.cn/46400.Doc
dxl.baoquan026.cn/99311.Doc
dxl.baoquan026.cn/39557.Doc
dxl.baoquan026.cn/42226.Doc
dxl.baoquan026.cn/22062.Doc
dxl.baoquan026.cn/28660.Doc
dxk.baoquan026.cn/44062.Doc
dxk.baoquan026.cn/46802.Doc
dxk.baoquan026.cn/97539.Doc
dxk.baoquan026.cn/46246.Doc
dxk.baoquan026.cn/88066.Doc
dxk.baoquan026.cn/73199.Doc
dxk.baoquan026.cn/24620.Doc
dxk.baoquan026.cn/02260.Doc
dxk.baoquan026.cn/88686.Doc
dxk.baoquan026.cn/80048.Doc
dxj.baoquan026.cn/22022.Doc
dxj.baoquan026.cn/48402.Doc
dxj.baoquan026.cn/06604.Doc
dxj.baoquan026.cn/64688.Doc
dxj.baoquan026.cn/15919.Doc
dxj.baoquan026.cn/68426.Doc
dxj.baoquan026.cn/82402.Doc
dxj.baoquan026.cn/20284.Doc
dxj.baoquan026.cn/84222.Doc
dxj.baoquan026.cn/62288.Doc
dxh.baoquan026.cn/42460.Doc
dxh.baoquan026.cn/31977.Doc
dxh.baoquan026.cn/53553.Doc
dxh.baoquan026.cn/42466.Doc
dxh.baoquan026.cn/26280.Doc
dxh.baoquan026.cn/00868.Doc
dxh.baoquan026.cn/40402.Doc
dxh.baoquan026.cn/44622.Doc
dxh.baoquan026.cn/46662.Doc
dxh.baoquan026.cn/80886.Doc
dxg.baoquan026.cn/84444.Doc
dxg.baoquan026.cn/44824.Doc
dxg.baoquan026.cn/84620.Doc
dxg.baoquan026.cn/44846.Doc
dxg.baoquan026.cn/62228.Doc
dxg.baoquan026.cn/02484.Doc
dxg.baoquan026.cn/20686.Doc
dxg.baoquan026.cn/28660.Doc
dxg.baoquan026.cn/60228.Doc
dxg.baoquan026.cn/66686.Doc
dxf.baoquan026.cn/04884.Doc
dxf.baoquan026.cn/84642.Doc
dxf.baoquan026.cn/46208.Doc
dxf.baoquan026.cn/88408.Doc
dxf.baoquan026.cn/53511.Doc
dxf.baoquan026.cn/24888.Doc
dxf.baoquan026.cn/80484.Doc
dxf.baoquan026.cn/62044.Doc
dxf.baoquan026.cn/26804.Doc
dxf.baoquan026.cn/40044.Doc
dxd.baoquan026.cn/08606.Doc
dxd.baoquan026.cn/82048.Doc
dxd.baoquan026.cn/02028.Doc
dxd.baoquan026.cn/37135.Doc
dxd.baoquan026.cn/46644.Doc
dxd.baoquan026.cn/39977.Doc
dxd.baoquan026.cn/86480.Doc
dxd.baoquan026.cn/88640.Doc
dxd.baoquan026.cn/08248.Doc
dxd.baoquan026.cn/24424.Doc
dxs.baoquan026.cn/40082.Doc
dxs.baoquan026.cn/46848.Doc
dxs.baoquan026.cn/02068.Doc
dxs.baoquan026.cn/06000.Doc
dxs.baoquan026.cn/62622.Doc
dxs.baoquan026.cn/39795.Doc
dxs.baoquan026.cn/08402.Doc
dxs.baoquan026.cn/24448.Doc
dxs.baoquan026.cn/20884.Doc
dxs.baoquan026.cn/04802.Doc
dxa.baoquan026.cn/46640.Doc
dxa.baoquan026.cn/24422.Doc
dxa.baoquan026.cn/60866.Doc
dxa.baoquan026.cn/40848.Doc
dxa.baoquan026.cn/60084.Doc
dxa.baoquan026.cn/04086.Doc
dxa.baoquan026.cn/02684.Doc
dxa.baoquan026.cn/46806.Doc
dxa.baoquan026.cn/64624.Doc
dxa.baoquan026.cn/24080.Doc
dxp.baoquan026.cn/44864.Doc
dxp.baoquan026.cn/28846.Doc
dxp.baoquan026.cn/46280.Doc
dxp.baoquan026.cn/82080.Doc
dxp.baoquan026.cn/68262.Doc
dxp.baoquan026.cn/46084.Doc
dxp.baoquan026.cn/00006.Doc
dxp.baoquan026.cn/66880.Doc
dxp.baoquan026.cn/84222.Doc
dxp.baoquan026.cn/48088.Doc
dxo.baoquan026.cn/42204.Doc
dxo.baoquan026.cn/42066.Doc
dxo.baoquan026.cn/48402.Doc
dxo.baoquan026.cn/44228.Doc
dxo.baoquan026.cn/68880.Doc
dxo.baoquan026.cn/19595.Doc
dxo.baoquan026.cn/60448.Doc
dxo.baoquan026.cn/82220.Doc
dxo.baoquan026.cn/88084.Doc
dxo.baoquan026.cn/40646.Doc
dxi.baoquan026.cn/59171.Doc
dxi.baoquan026.cn/44240.Doc
dxi.baoquan026.cn/62688.Doc
dxi.baoquan026.cn/60282.Doc
dxi.baoquan026.cn/80886.Doc
dxi.baoquan026.cn/84644.Doc
dxi.baoquan026.cn/40408.Doc
dxi.baoquan026.cn/84460.Doc
dxi.baoquan026.cn/66004.Doc
dxi.baoquan026.cn/00822.Doc
dxu.baoquan026.cn/80064.Doc
dxu.baoquan026.cn/22626.Doc
dxu.baoquan026.cn/48248.Doc
dxu.baoquan026.cn/66884.Doc
dxu.baoquan026.cn/37771.Doc
dxu.baoquan026.cn/88860.Doc
dxu.baoquan026.cn/06220.Doc
dxu.baoquan026.cn/62286.Doc
dxu.baoquan026.cn/08602.Doc
dxu.baoquan026.cn/42604.Doc
dxy.baoquan026.cn/46422.Doc
dxy.baoquan026.cn/46226.Doc
dxy.baoquan026.cn/80460.Doc
dxy.baoquan026.cn/68080.Doc
dxy.baoquan026.cn/88228.Doc
dxy.baoquan026.cn/46202.Doc
dxy.baoquan026.cn/26044.Doc
dxy.baoquan026.cn/86028.Doc
dxy.baoquan026.cn/08602.Doc
dxy.baoquan026.cn/66042.Doc
dxt.baoquan026.cn/26420.Doc
dxt.baoquan026.cn/40460.Doc
dxt.baoquan026.cn/82422.Doc
dxt.baoquan026.cn/62004.Doc
dxt.baoquan026.cn/20402.Doc
dxt.baoquan026.cn/66644.Doc
dxt.baoquan026.cn/86026.Doc
dxt.baoquan026.cn/06262.Doc
dxt.baoquan026.cn/00260.Doc
dxt.baoquan026.cn/84686.Doc
dxr.baoquan026.cn/68802.Doc
dxr.baoquan026.cn/86804.Doc
dxr.baoquan026.cn/66208.Doc
dxr.baoquan026.cn/22080.Doc
dxr.baoquan026.cn/11971.Doc
dxr.baoquan026.cn/11513.Doc
dxr.baoquan026.cn/02282.Doc
dxr.baoquan026.cn/60466.Doc
dxr.baoquan026.cn/24062.Doc
dxr.baoquan026.cn/00284.Doc
dxe.baoquan026.cn/66602.Doc
dxe.baoquan026.cn/64026.Doc
dxe.baoquan026.cn/00002.Doc
dxe.baoquan026.cn/26668.Doc
dxe.baoquan026.cn/46264.Doc
dxe.baoquan026.cn/40884.Doc
dxe.baoquan026.cn/35915.Doc
dxe.baoquan026.cn/06260.Doc
dxe.baoquan026.cn/11579.Doc
dxe.baoquan026.cn/20824.Doc
dxw.baoquan026.cn/33911.Doc
dxw.baoquan026.cn/60004.Doc
dxw.baoquan026.cn/62888.Doc
dxw.baoquan026.cn/20028.Doc
dxw.baoquan026.cn/64846.Doc
dxw.baoquan026.cn/86640.Doc
dxw.baoquan026.cn/44062.Doc
dxw.baoquan026.cn/06868.Doc
dxw.baoquan026.cn/66042.Doc
dxw.baoquan026.cn/26662.Doc
dxq.baoquan026.cn/26468.Doc
dxq.baoquan026.cn/48826.Doc
dxq.baoquan026.cn/73313.Doc
dxq.baoquan026.cn/80482.Doc
dxq.baoquan026.cn/88008.Doc
dxq.baoquan026.cn/48082.Doc
dxq.baoquan026.cn/86442.Doc
dxq.baoquan026.cn/42444.Doc
dxq.baoquan026.cn/40602.Doc
dxq.baoquan026.cn/66664.Doc
dzm.baoquan026.cn/66888.Doc
dzm.baoquan026.cn/26260.Doc
dzm.baoquan026.cn/44866.Doc
dzm.baoquan026.cn/20682.Doc
dzm.baoquan026.cn/26666.Doc
dzm.baoquan026.cn/24024.Doc
dzm.baoquan026.cn/60248.Doc
dzm.baoquan026.cn/02048.Doc
dzm.baoquan026.cn/20880.Doc
dzm.baoquan026.cn/80282.Doc
dzn.baoquan026.cn/84846.Doc
dzn.baoquan026.cn/46666.Doc
dzn.baoquan026.cn/82404.Doc
dzn.baoquan026.cn/00240.Doc
dzn.baoquan026.cn/02862.Doc
dzn.baoquan026.cn/42208.Doc
dzn.baoquan026.cn/82224.Doc
dzn.baoquan026.cn/80248.Doc
dzn.baoquan026.cn/86864.Doc
dzn.baoquan026.cn/06220.Doc
dzb.baoquan026.cn/44842.Doc
dzb.baoquan026.cn/60002.Doc
dzb.baoquan026.cn/28608.Doc
dzb.baoquan026.cn/00042.Doc
dzb.baoquan026.cn/24240.Doc
dzb.baoquan026.cn/66268.Doc
dzb.baoquan026.cn/44022.Doc
dzb.baoquan026.cn/42082.Doc
dzb.baoquan026.cn/40648.Doc
dzb.baoquan026.cn/40468.Doc
dzv.baoquan026.cn/04444.Doc
dzv.baoquan026.cn/22220.Doc
dzv.baoquan026.cn/04224.Doc
dzv.baoquan026.cn/26482.Doc
dzv.baoquan026.cn/80668.Doc
dzv.baoquan026.cn/04806.Doc
dzv.baoquan026.cn/80444.Doc
dzv.baoquan026.cn/84264.Doc
dzv.baoquan026.cn/20282.Doc
dzv.baoquan026.cn/64400.Doc
dzc.baoquan026.cn/44480.Doc
dzc.baoquan026.cn/68882.Doc
dzc.baoquan026.cn/86264.Doc
dzc.baoquan026.cn/84800.Doc
dzc.baoquan026.cn/68486.Doc
dzc.baoquan026.cn/08080.Doc
dzc.baoquan026.cn/48204.Doc
dzc.baoquan026.cn/60080.Doc
dzc.baoquan026.cn/62662.Doc
dzc.baoquan026.cn/24866.Doc
dzx.baoquan026.cn/28606.Doc
dzx.baoquan026.cn/08008.Doc
dzx.baoquan026.cn/46420.Doc
dzx.baoquan026.cn/66002.Doc
dzx.baoquan026.cn/80020.Doc
dzx.baoquan026.cn/26088.Doc
dzx.baoquan026.cn/20022.Doc
dzx.baoquan026.cn/46606.Doc
dzx.baoquan026.cn/84040.Doc
dzx.baoquan026.cn/84024.Doc
dzz.baoquan026.cn/24602.Doc
dzz.baoquan026.cn/44248.Doc
dzz.baoquan026.cn/00826.Doc
dzz.baoquan026.cn/24086.Doc
dzz.baoquan026.cn/46846.Doc
dzz.baoquan026.cn/84404.Doc
dzz.baoquan026.cn/64402.Doc
dzz.baoquan026.cn/84448.Doc
dzz.baoquan026.cn/26064.Doc
dzz.baoquan026.cn/22000.Doc
dzl.baoquan026.cn/80088.Doc
dzl.baoquan026.cn/60822.Doc
dzl.baoquan026.cn/48000.Doc
dzl.baoquan026.cn/62048.Doc
dzl.baoquan026.cn/44648.Doc
dzl.baoquan026.cn/02888.Doc
dzl.baoquan026.cn/62680.Doc
dzl.baoquan026.cn/80006.Doc
dzl.baoquan026.cn/06406.Doc
dzl.baoquan026.cn/02266.Doc
dzk.baoquan026.cn/88248.Doc
dzk.baoquan026.cn/40806.Doc
dzk.baoquan026.cn/60680.Doc
dzk.baoquan026.cn/64266.Doc
dzk.baoquan026.cn/68406.Doc
dzk.baoquan026.cn/26668.Doc
dzk.baoquan026.cn/62442.Doc
dzk.baoquan026.cn/80868.Doc
dzk.baoquan026.cn/00284.Doc
dzk.baoquan026.cn/46444.Doc
dzj.baoquan026.cn/24628.Doc
dzj.baoquan026.cn/88880.Doc
dzj.baoquan026.cn/40802.Doc
dzj.baoquan026.cn/80286.Doc
dzj.baoquan026.cn/00842.Doc
dzj.baoquan026.cn/88024.Doc
dzj.baoquan026.cn/04444.Doc
dzj.baoquan026.cn/60000.Doc
dzj.baoquan026.cn/00882.Doc
dzj.baoquan026.cn/08600.Doc
