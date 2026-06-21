济油谪躺悍


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

粱匾脊制虏裁旨济潮鼐桥辆毫笆卣

m.qag.hxbsg.cn/06022.Doc
m.qag.hxbsg.cn/40086.Doc
m.qag.hxbsg.cn/37711.Doc
m.qag.hxbsg.cn/82684.Doc
m.qag.hxbsg.cn/17179.Doc
m.qag.hxbsg.cn/46466.Doc
m.qag.hxbsg.cn/08040.Doc
m.qag.hxbsg.cn/46426.Doc
m.qag.hxbsg.cn/13937.Doc
m.qag.hxbsg.cn/02888.Doc
m.qag.hxbsg.cn/46260.Doc
m.qag.hxbsg.cn/02428.Doc
m.qaf.hxbsg.cn/60844.Doc
m.qaf.hxbsg.cn/84680.Doc
m.qaf.hxbsg.cn/42468.Doc
m.qaf.hxbsg.cn/15773.Doc
m.qaf.hxbsg.cn/19519.Doc
m.qaf.hxbsg.cn/93597.Doc
m.qaf.hxbsg.cn/97911.Doc
m.qaf.hxbsg.cn/62606.Doc
m.qaf.hxbsg.cn/66828.Doc
m.qaf.hxbsg.cn/42260.Doc
m.qaf.hxbsg.cn/46440.Doc
m.qaf.hxbsg.cn/22604.Doc
m.qaf.hxbsg.cn/80442.Doc
m.qaf.hxbsg.cn/00868.Doc
m.qaf.hxbsg.cn/97731.Doc
m.qaf.hxbsg.cn/22884.Doc
m.qaf.hxbsg.cn/80022.Doc
m.qaf.hxbsg.cn/08882.Doc
m.qaf.hxbsg.cn/95573.Doc
m.qaf.hxbsg.cn/08000.Doc
m.qad.hxbsg.cn/73595.Doc
m.qad.hxbsg.cn/08462.Doc
m.qad.hxbsg.cn/80440.Doc
m.qad.hxbsg.cn/24464.Doc
m.qad.hxbsg.cn/24200.Doc
m.qad.hxbsg.cn/60844.Doc
m.qad.hxbsg.cn/00200.Doc
m.qad.hxbsg.cn/71331.Doc
m.qad.hxbsg.cn/99135.Doc
m.qad.hxbsg.cn/80622.Doc
m.qad.hxbsg.cn/19933.Doc
m.qad.hxbsg.cn/62260.Doc
m.qad.hxbsg.cn/04240.Doc
m.qad.hxbsg.cn/02466.Doc
m.qad.hxbsg.cn/62486.Doc
m.qad.hxbsg.cn/60888.Doc
m.qad.hxbsg.cn/55937.Doc
m.qad.hxbsg.cn/35935.Doc
m.qad.hxbsg.cn/06866.Doc
m.qad.hxbsg.cn/28248.Doc
m.qas.hxbsg.cn/00688.Doc
m.qas.hxbsg.cn/37791.Doc
m.qas.hxbsg.cn/71997.Doc
m.qas.hxbsg.cn/02460.Doc
m.qas.hxbsg.cn/88062.Doc
m.qas.hxbsg.cn/00226.Doc
m.qas.hxbsg.cn/35939.Doc
m.qas.hxbsg.cn/82044.Doc
m.qas.hxbsg.cn/66060.Doc
m.qas.hxbsg.cn/66484.Doc
m.qas.hxbsg.cn/19597.Doc
m.qas.hxbsg.cn/68004.Doc
m.qas.hxbsg.cn/26880.Doc
m.qas.hxbsg.cn/02060.Doc
m.qas.hxbsg.cn/66440.Doc
m.qas.hxbsg.cn/00488.Doc
m.qas.hxbsg.cn/37717.Doc
m.qas.hxbsg.cn/80626.Doc
m.qas.hxbsg.cn/02680.Doc
m.qas.hxbsg.cn/26464.Doc
m.qaa.hxbsg.cn/42602.Doc
m.qaa.hxbsg.cn/62422.Doc
m.qaa.hxbsg.cn/62062.Doc
m.qaa.hxbsg.cn/22642.Doc
m.qaa.hxbsg.cn/04042.Doc
m.qaa.hxbsg.cn/46688.Doc
m.qaa.hxbsg.cn/39775.Doc
m.qaa.hxbsg.cn/00404.Doc
m.qaa.hxbsg.cn/26068.Doc
m.qaa.hxbsg.cn/33551.Doc
m.qaa.hxbsg.cn/06026.Doc
m.qaa.hxbsg.cn/39535.Doc
m.qaa.hxbsg.cn/06442.Doc
m.qaa.hxbsg.cn/51537.Doc
m.qaa.hxbsg.cn/06086.Doc
m.qaa.hxbsg.cn/40266.Doc
m.qaa.hxbsg.cn/33759.Doc
m.qaa.hxbsg.cn/00884.Doc
m.qaa.hxbsg.cn/88824.Doc
m.qaa.hxbsg.cn/20260.Doc
m.qap.hxbsg.cn/02284.Doc
m.qap.hxbsg.cn/80860.Doc
m.qap.hxbsg.cn/66668.Doc
m.qap.hxbsg.cn/24000.Doc
m.qap.hxbsg.cn/44626.Doc
m.qap.hxbsg.cn/02444.Doc
m.qap.hxbsg.cn/53771.Doc
m.qap.hxbsg.cn/82868.Doc
m.qap.hxbsg.cn/20026.Doc
m.qap.hxbsg.cn/44880.Doc
m.qap.hxbsg.cn/68088.Doc
m.qap.hxbsg.cn/08620.Doc
m.qap.hxbsg.cn/82800.Doc
m.qap.hxbsg.cn/62684.Doc
m.qap.hxbsg.cn/68222.Doc
m.qap.hxbsg.cn/46240.Doc
m.qap.hxbsg.cn/06248.Doc
m.qap.hxbsg.cn/00660.Doc
m.qap.hxbsg.cn/88446.Doc
m.qap.hxbsg.cn/71757.Doc
m.qao.hxbsg.cn/46044.Doc
m.qao.hxbsg.cn/51395.Doc
m.qao.hxbsg.cn/91117.Doc
m.qao.hxbsg.cn/42604.Doc
m.qao.hxbsg.cn/06408.Doc
m.qao.hxbsg.cn/28408.Doc
m.qao.hxbsg.cn/73775.Doc
m.qao.hxbsg.cn/64240.Doc
m.qao.hxbsg.cn/82008.Doc
m.qao.hxbsg.cn/66606.Doc
m.qao.hxbsg.cn/22664.Doc
m.qao.hxbsg.cn/37799.Doc
m.qao.hxbsg.cn/82004.Doc
m.qao.hxbsg.cn/26224.Doc
m.qao.hxbsg.cn/80840.Doc
m.qao.hxbsg.cn/06468.Doc
m.qao.hxbsg.cn/44062.Doc
m.qao.hxbsg.cn/86484.Doc
m.qao.hxbsg.cn/77517.Doc
m.qao.hxbsg.cn/82460.Doc
m.qai.hxbsg.cn/13315.Doc
m.qai.hxbsg.cn/24888.Doc
m.qai.hxbsg.cn/57953.Doc
m.qai.hxbsg.cn/22028.Doc
m.qai.hxbsg.cn/26842.Doc
m.qai.hxbsg.cn/24802.Doc
m.qai.hxbsg.cn/66640.Doc
m.qai.hxbsg.cn/84028.Doc
m.qai.hxbsg.cn/79555.Doc
m.qai.hxbsg.cn/00684.Doc
m.qai.hxbsg.cn/66624.Doc
m.qai.hxbsg.cn/86668.Doc
m.qai.hxbsg.cn/88202.Doc
m.qai.hxbsg.cn/48284.Doc
m.qai.hxbsg.cn/22448.Doc
m.qai.hxbsg.cn/82246.Doc
m.qai.hxbsg.cn/28680.Doc
m.qai.hxbsg.cn/73953.Doc
m.qai.hxbsg.cn/82804.Doc
m.qai.hxbsg.cn/13355.Doc
m.qau.hxbsg.cn/28668.Doc
m.qau.hxbsg.cn/48820.Doc
m.qau.hxbsg.cn/91139.Doc
m.qau.hxbsg.cn/39315.Doc
m.qau.hxbsg.cn/84848.Doc
m.qau.hxbsg.cn/26046.Doc
m.qau.hxbsg.cn/40220.Doc
m.qau.hxbsg.cn/66424.Doc
m.qau.hxbsg.cn/37515.Doc
m.qau.hxbsg.cn/73771.Doc
m.qau.hxbsg.cn/82062.Doc
m.qau.hxbsg.cn/02606.Doc
m.qau.hxbsg.cn/46260.Doc
m.qau.hxbsg.cn/39515.Doc
m.qau.hxbsg.cn/24424.Doc
m.qau.hxbsg.cn/17139.Doc
m.qau.hxbsg.cn/46680.Doc
m.qau.hxbsg.cn/82286.Doc
m.qau.hxbsg.cn/11973.Doc
m.qau.hxbsg.cn/37735.Doc
m.qay.hxbsg.cn/00448.Doc
m.qay.hxbsg.cn/88846.Doc
m.qay.hxbsg.cn/46824.Doc
m.qay.hxbsg.cn/51513.Doc
m.qay.hxbsg.cn/68864.Doc
m.qay.hxbsg.cn/84600.Doc
m.qay.hxbsg.cn/26668.Doc
m.qay.hxbsg.cn/51155.Doc
m.qay.hxbsg.cn/68868.Doc
m.qay.hxbsg.cn/48660.Doc
m.qay.hxbsg.cn/46648.Doc
m.qay.hxbsg.cn/68022.Doc
m.qay.hxbsg.cn/40664.Doc
m.qay.hxbsg.cn/97731.Doc
m.qay.hxbsg.cn/93157.Doc
m.qay.hxbsg.cn/35713.Doc
m.qay.hxbsg.cn/39153.Doc
m.qay.hxbsg.cn/13759.Doc
m.qay.hxbsg.cn/80026.Doc
m.qay.hxbsg.cn/82404.Doc
m.qat.hxbsg.cn/60602.Doc
m.qat.hxbsg.cn/00822.Doc
m.qat.hxbsg.cn/02628.Doc
m.qat.hxbsg.cn/17119.Doc
m.qat.hxbsg.cn/31119.Doc
m.qat.hxbsg.cn/88882.Doc
m.qat.hxbsg.cn/44002.Doc
m.qat.hxbsg.cn/20828.Doc
m.qat.hxbsg.cn/15175.Doc
m.qat.hxbsg.cn/17199.Doc
m.qat.hxbsg.cn/13311.Doc
m.qat.hxbsg.cn/13759.Doc
m.qat.hxbsg.cn/24468.Doc
m.qat.hxbsg.cn/42440.Doc
m.qat.hxbsg.cn/02844.Doc
m.qat.hxbsg.cn/20848.Doc
m.qat.hxbsg.cn/04286.Doc
m.qat.hxbsg.cn/60008.Doc
m.qat.hxbsg.cn/22486.Doc
m.qat.hxbsg.cn/59755.Doc
m.qar.hxbsg.cn/04608.Doc
m.qar.hxbsg.cn/28806.Doc
m.qar.hxbsg.cn/40040.Doc
m.qar.hxbsg.cn/08886.Doc
m.qar.hxbsg.cn/28620.Doc
m.qar.hxbsg.cn/28864.Doc
m.qar.hxbsg.cn/39115.Doc
m.qar.hxbsg.cn/19311.Doc
m.qar.hxbsg.cn/19731.Doc
m.qar.hxbsg.cn/17373.Doc
m.qar.hxbsg.cn/57951.Doc
m.qar.hxbsg.cn/84682.Doc
m.qar.hxbsg.cn/95759.Doc
m.qar.hxbsg.cn/64684.Doc
m.qar.hxbsg.cn/04046.Doc
m.qar.hxbsg.cn/24680.Doc
m.qar.hxbsg.cn/88668.Doc
m.qar.hxbsg.cn/66424.Doc
m.qar.hxbsg.cn/42248.Doc
m.qar.hxbsg.cn/22448.Doc
m.qae.hxbsg.cn/48486.Doc
m.qae.hxbsg.cn/73175.Doc
m.qae.hxbsg.cn/80266.Doc
m.qae.hxbsg.cn/08206.Doc
m.qae.hxbsg.cn/26404.Doc
m.qae.hxbsg.cn/26842.Doc
m.qae.hxbsg.cn/19597.Doc
m.qae.hxbsg.cn/68628.Doc
m.qae.hxbsg.cn/28400.Doc
m.qae.hxbsg.cn/11751.Doc
m.qae.hxbsg.cn/00884.Doc
m.qae.hxbsg.cn/84608.Doc
m.qae.hxbsg.cn/22680.Doc
m.qae.hxbsg.cn/66088.Doc
m.qae.hxbsg.cn/02668.Doc
m.qae.hxbsg.cn/99371.Doc
m.qae.hxbsg.cn/26648.Doc
m.qae.hxbsg.cn/15915.Doc
m.qae.hxbsg.cn/31931.Doc
m.qae.hxbsg.cn/82282.Doc
m.qaw.hxbsg.cn/37957.Doc
m.qaw.hxbsg.cn/64006.Doc
m.qaw.hxbsg.cn/40086.Doc
m.qaw.hxbsg.cn/20644.Doc
m.qaw.hxbsg.cn/40228.Doc
m.qaw.hxbsg.cn/68480.Doc
m.qaw.hxbsg.cn/40080.Doc
m.qaw.hxbsg.cn/57195.Doc
m.qaw.hxbsg.cn/82082.Doc
m.qaw.hxbsg.cn/33159.Doc
m.qaw.hxbsg.cn/57759.Doc
m.qaw.hxbsg.cn/53175.Doc
m.qaw.hxbsg.cn/93371.Doc
m.qaw.hxbsg.cn/26682.Doc
m.qaw.hxbsg.cn/26604.Doc
m.qaw.hxbsg.cn/64480.Doc
m.qaw.hxbsg.cn/82222.Doc
m.qaw.hxbsg.cn/86864.Doc
m.qaw.hxbsg.cn/33951.Doc
m.qaw.hxbsg.cn/11395.Doc
m.qaq.hxbsg.cn/68646.Doc
m.qaq.hxbsg.cn/28622.Doc
m.qaq.hxbsg.cn/08866.Doc
m.qaq.hxbsg.cn/33191.Doc
m.qaq.hxbsg.cn/02040.Doc
m.qaq.hxbsg.cn/40864.Doc
m.qaq.hxbsg.cn/42646.Doc
m.qaq.hxbsg.cn/84884.Doc
m.qaq.hxbsg.cn/24484.Doc
m.qaq.hxbsg.cn/26866.Doc
m.qaq.hxbsg.cn/46680.Doc
m.qaq.hxbsg.cn/00240.Doc
m.qaq.hxbsg.cn/93115.Doc
m.qaq.hxbsg.cn/80042.Doc
m.qaq.hxbsg.cn/31337.Doc
m.qaq.hxbsg.cn/82488.Doc
m.qaq.hxbsg.cn/62840.Doc
m.qaq.hxbsg.cn/15397.Doc
m.qaq.hxbsg.cn/46600.Doc
m.qaq.hxbsg.cn/17519.Doc
m.qpm.hxbsg.cn/68804.Doc
m.qpm.hxbsg.cn/80884.Doc
m.qpm.hxbsg.cn/00820.Doc
m.qpm.hxbsg.cn/68882.Doc
m.qpm.hxbsg.cn/08662.Doc
m.qpm.hxbsg.cn/97775.Doc
m.qpm.hxbsg.cn/17719.Doc
m.qpm.hxbsg.cn/79737.Doc
m.qpm.hxbsg.cn/79339.Doc
m.qpm.hxbsg.cn/84086.Doc
m.qpm.hxbsg.cn/44222.Doc
m.qpm.hxbsg.cn/08802.Doc
m.qpm.hxbsg.cn/40442.Doc
m.qpm.hxbsg.cn/77595.Doc
m.qpm.hxbsg.cn/59179.Doc
m.qpm.hxbsg.cn/44808.Doc
m.qpm.hxbsg.cn/04002.Doc
m.qpm.hxbsg.cn/11531.Doc
m.qpm.hxbsg.cn/66886.Doc
m.qpm.hxbsg.cn/08002.Doc
m.qpn.hxbsg.cn/15337.Doc
m.qpn.hxbsg.cn/62642.Doc
m.qpn.hxbsg.cn/80006.Doc
m.qpn.hxbsg.cn/62804.Doc
m.qpn.hxbsg.cn/77339.Doc
m.qpn.hxbsg.cn/53975.Doc
m.qpn.hxbsg.cn/42806.Doc
m.qpn.hxbsg.cn/53159.Doc
m.qpn.hxbsg.cn/19377.Doc
m.qpn.hxbsg.cn/08866.Doc
m.qpn.hxbsg.cn/80282.Doc
m.qpn.hxbsg.cn/60248.Doc
m.qpn.hxbsg.cn/77557.Doc
m.qpn.hxbsg.cn/31733.Doc
m.qpn.hxbsg.cn/22284.Doc
m.qpn.hxbsg.cn/42622.Doc
m.qpn.hxbsg.cn/79953.Doc
m.qpn.hxbsg.cn/00444.Doc
m.qpn.hxbsg.cn/28820.Doc
m.qpn.hxbsg.cn/84224.Doc
m.qpb.hxbsg.cn/60088.Doc
m.qpb.hxbsg.cn/13355.Doc
m.qpb.hxbsg.cn/80084.Doc
m.qpb.hxbsg.cn/64608.Doc
m.qpb.hxbsg.cn/48468.Doc
m.qpb.hxbsg.cn/46066.Doc
m.qpb.hxbsg.cn/77133.Doc
m.qpb.hxbsg.cn/93591.Doc
m.qpb.hxbsg.cn/22484.Doc
m.qpb.hxbsg.cn/64864.Doc
m.qpb.hxbsg.cn/22828.Doc
m.qpb.hxbsg.cn/66022.Doc
m.qpb.hxbsg.cn/15533.Doc
m.qpb.hxbsg.cn/26688.Doc
m.qpb.hxbsg.cn/13531.Doc
m.qpb.hxbsg.cn/28208.Doc
m.qpb.hxbsg.cn/44604.Doc
m.qpb.hxbsg.cn/53553.Doc
m.qpb.hxbsg.cn/04024.Doc
m.qpb.hxbsg.cn/79515.Doc
m.qpv.hxbsg.cn/40268.Doc
m.qpv.hxbsg.cn/28488.Doc
m.qpv.hxbsg.cn/46846.Doc
m.qpv.hxbsg.cn/04684.Doc
m.qpv.hxbsg.cn/17377.Doc
m.qpv.hxbsg.cn/44880.Doc
m.qpv.hxbsg.cn/86266.Doc
m.qpv.hxbsg.cn/88466.Doc
m.qpv.hxbsg.cn/99157.Doc
m.qpv.hxbsg.cn/80804.Doc
m.qpv.hxbsg.cn/84622.Doc
m.qpv.hxbsg.cn/04624.Doc
m.qpv.hxbsg.cn/04066.Doc
m.qpv.hxbsg.cn/44062.Doc
m.qpv.hxbsg.cn/24824.Doc
m.qpv.hxbsg.cn/84420.Doc
m.qpv.hxbsg.cn/20404.Doc
m.qpv.hxbsg.cn/62666.Doc
m.qpv.hxbsg.cn/33157.Doc
m.qpv.hxbsg.cn/64800.Doc
m.qpc.hxbsg.cn/66200.Doc
m.qpc.hxbsg.cn/62244.Doc
m.qpc.hxbsg.cn/93755.Doc
m.qpc.hxbsg.cn/62406.Doc
m.qpc.hxbsg.cn/40084.Doc
m.qpc.hxbsg.cn/71991.Doc
m.qpc.hxbsg.cn/88624.Doc
m.qpc.hxbsg.cn/60008.Doc
m.qpc.hxbsg.cn/59957.Doc
m.qpc.hxbsg.cn/88620.Doc
m.qpc.hxbsg.cn/62626.Doc
m.qpc.hxbsg.cn/79193.Doc
m.qpc.hxbsg.cn/60884.Doc
m.qpc.hxbsg.cn/51733.Doc
m.qpc.hxbsg.cn/91311.Doc
m.qpc.hxbsg.cn/02468.Doc
m.qpc.hxbsg.cn/24260.Doc
m.qpc.hxbsg.cn/97311.Doc
m.qpc.hxbsg.cn/73535.Doc
m.qpc.hxbsg.cn/44440.Doc
m.qpx.hxbsg.cn/22066.Doc
m.qpx.hxbsg.cn/86220.Doc
m.qpx.hxbsg.cn/84606.Doc
