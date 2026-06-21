敝官肚餐苟


========================================================
 Python 缓存策略与实现 — 从内存到持久化存储
========================================================

缓存是提升程序性能最有效的手段之一。Python 生态中
有多种缓存方案，从标准库到第三方工具，各有适用场景。

--------------------------------------------------------
1. functools.lru_cache — 最近最少使用缓存
--------------------------------------------------------

from functools import lru_cache
import time

# maxsize 限制缓存条目数，typed 区分参数类型
@lru_cache(maxsize=128, typed=True)
def fibonacci(n):
    """带缓存的计算斐波那契数列"""
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

start = time.time()
print(fibonacci(35))  # 第一次计算，慢
print(f"耗时: {time.time() - start:.4f}s")

start = time.time()
print(fibonacci(35))  # 直接从缓存读取，极快
print(f"耗时: {time.time() - start:.6f}s")

# 查看缓存统计信息
print(fibonacci.cache_info())
# CacheInfo(hits=33, misses=36, maxsize=128, currsize=36)

# 手动清除缓存
fibonacci.cache_clear()

--------------------------------------------------------
2. functools.cache — Python 3.9+ 简化版
--------------------------------------------------------

from functools import cache

@cache
def compute_expensive(n):
    """无参配置的 LRU 缓存，等价于 lru_cache(maxsize=None)"""
    print(f"计算 compute_expensive({n})...")
    return sum(i * i for i in range(n))

print(compute_expensive(10_000))  # 第一次真实计算
print(compute_expensive(10_000))  # 命中缓存，无输出

--------------------------------------------------------
3. 手写 TTLCache — 带过期时间的缓存
--------------------------------------------------------

import time
import threading

class TTLCache:
    """带有生存时间（TTL）的本地缓存实现"""

    def __init__(self, ttl_seconds=60):
        self._data = {}        # 存储键值对
        self._expires = {}     # 存储过期时间戳
        self._ttl = ttl_seconds
        self._lock = threading.Lock()

    def set(self, key, value):
        """设置缓存项并记录过期时间"""
        with self._lock:
            self._data[key] = value
            self._expires[key] = time.time() + self._ttl

    def get(self, key, default=None):
        """获取缓存项，已过期则删除并返回默认值"""
        with self._lock:
            if key not in self._data:
                return default
            if time.time() > self._expires[key]:
                # 缓存已过期，清理
                del self._data[key]
                del self._expires[key]
                return default
            return self._data[key]

    def clear_expired(self):
        """清理所有过期项（可由后台线程定时调用）"""
        now = time.time()
        with self._lock:
            expired = [k for k, v in self._expires.items()
                       if now > v]
            for k in expired:
                del self._data[k]
                del self._expires[k]

# 使用示例
cache = TTLCache(ttl_seconds=5)
cache.set("token", "abc123")
print(cache.get("token"))  # abc123
time.sleep(6)
print(cache.get("token"))  # None（已过期）

--------------------------------------------------------
4. Redis 缓存基本模式
--------------------------------------------------------

# 安装：pip install redis
import redis

r = redis.Redis(host="localhost", port=6379, db=0,
                decode_responses=True)

def get_user(user_id):
    """带 Redis 缓存的用户查询"""
    cache_key = f"user:{user_id}"

    # 1. 尝试从缓存读取
    cached = r.get(cache_key)
    if cached is not None:
        print(f"[缓存命中] user:{user_id}")
        return cached  # 实际应用应反序列化 JSON

    # 2. 缓存未命中，从数据库加载
    print(f"[缓存未命中] 从数据库加载 user:{user_id}")
    user_data = {"id": user_id, "name": "张三"}

    # 3. 写入缓存，设置 300 秒过期
    # setex = set + expire 的原子操作
    r.setex(cache_key, 300, str(user_data))
    return user_data

# get_user(42)  # 首次：查数据库
# get_user(42)  # 第二次：缓存命中

--------------------------------------------------------
5. 缓存失效策略
--------------------------------------------------------

class CacheManager:
    """管理多种缓存失效策略"""

    def __init__(self, redis_client):
        self.r = redis_client

    def evict_pattern(self, pattern):
        """按模式批量失效（如清除所有 user: 前缀的缓存）"""
        cursor = 0
        while True:
            cursor, keys = self.r.scan(cursor, match=pattern,
                                       count=100)
            if keys:
                self.r.delete(*keys)
            if cursor == 0:
                break

    def write_through(self, key, data, ttl=300):
        """写穿透：同时写入缓存和数据库"""
        # 写入数据库
        self._save_to_db(key, data)
        # 同步更新缓存
        self.r.setex(key, ttl, str(data))

    def write_behind(self, key, data, ttl=300):
        """写后缓存：先写数据库，再异步更新缓存"""
        self._save_to_db(key, data)
        # 这里可以用消息队列异步更新缓存
        self.r.setex(key, ttl, str(data))

    def _save_to_db(self, key, data):
        pass  # 模拟数据库写入

--------------------------------------------------------
6. diskcache — 持久化缓存
--------------------------------------------------------

# 安装：pip install diskcache
from diskcache import Cache

# 缓存存储在本地磁盘目录中，重启不丢失
disk_cache = Cache("/tmp/my_cache")

@disk_cache.memoize(expire=60, tag="weather")
def fetch_weather(city):
    """获取天气数据（结果会被缓存到磁盘）"""
    print(f"正在获取 {city} 的天气...")
    return {"city": city, "temp": 25, "humidity": 60}

# 第一次：真实获取
print(fetch_weather("北京"))

# 第二次：从磁盘缓存读取
print(fetch_weather("北京"))

# 按标签清除
disk_cache.evict(tag="weather")

# 缓存统计
print(disk_cache.stats())

--------------------------------------------------------
7. 记忆化（Memoization）通用模式
--------------------------------------------------------

def memoize(func):
    """通用的记忆化装饰器，支持自定义键生成"""
    cache = {}

    def wrapper(*args, **kwargs):
        # 使用参数作为缓存键
        key = str(args) + str(sorted(kwargs.items()))
        if key not in cache:
            cache[key] = func(*args, **kwargs)
        return cache[key]

    wrapper.cache = cache
    wrapper.clear = cache.clear
    return wrapper

@memoize
def query_database(sql):
    """模拟数据库查询"""
    print(f"执行查询: {sql}")
    return {"result": "data"}

print(query_database("SELECT * FROM users"))
print(query_database("SELECT * FROM users"))  # 走缓存
print(len(query_database.cache))              # 缓存条目数

选择合适的缓存策略需要权衡一致性、可用性和性能：
- 读多写少 -> LRU + TTL
- 需要持久化 -> diskcache 或 Redis
- 数据必须一致 -> 写穿透策略
- 可接受短暂不一致 -> 写后缓存 + TTL

欣聪美型仙侥幽冻胸幢刎室谜诹僮

m.wjb.msfsx.cn/24864.Doc
m.wjb.msfsx.cn/75791.Doc
m.wjb.msfsx.cn/37173.Doc
m.wjb.msfsx.cn/06864.Doc
m.wjb.msfsx.cn/55919.Doc
m.wjb.msfsx.cn/19111.Doc
m.wjb.msfsx.cn/66822.Doc
m.wjb.msfsx.cn/86044.Doc
m.wjb.msfsx.cn/68028.Doc
m.wjb.msfsx.cn/88480.Doc
m.wjv.msfsx.cn/17733.Doc
m.wjv.msfsx.cn/33513.Doc
m.wjv.msfsx.cn/39173.Doc
m.wjv.msfsx.cn/97791.Doc
m.wjv.msfsx.cn/13179.Doc
m.wjv.msfsx.cn/17393.Doc
m.wjv.msfsx.cn/35511.Doc
m.wjv.msfsx.cn/28008.Doc
m.wjv.msfsx.cn/82204.Doc
m.wjv.msfsx.cn/19735.Doc
m.wjv.msfsx.cn/17997.Doc
m.wjv.msfsx.cn/79173.Doc
m.wjv.msfsx.cn/75751.Doc
m.wjv.msfsx.cn/93113.Doc
m.wjv.msfsx.cn/19759.Doc
m.wjv.msfsx.cn/51397.Doc
m.wjv.msfsx.cn/04682.Doc
m.wjv.msfsx.cn/53153.Doc
m.wjv.msfsx.cn/99179.Doc
m.wjv.msfsx.cn/91359.Doc
m.wjc.msfsx.cn/71771.Doc
m.wjc.msfsx.cn/77795.Doc
m.wjc.msfsx.cn/35115.Doc
m.wjc.msfsx.cn/79179.Doc
m.wjc.msfsx.cn/39577.Doc
m.wjc.msfsx.cn/64842.Doc
m.wjc.msfsx.cn/53933.Doc
m.wjc.msfsx.cn/26844.Doc
m.wjc.msfsx.cn/28682.Doc
m.wjc.msfsx.cn/64080.Doc
m.wjc.msfsx.cn/33799.Doc
m.wjc.msfsx.cn/66406.Doc
m.wjc.msfsx.cn/77355.Doc
m.wjc.msfsx.cn/35795.Doc
m.wjc.msfsx.cn/48680.Doc
m.wjc.msfsx.cn/20244.Doc
m.wjc.msfsx.cn/15555.Doc
m.wjc.msfsx.cn/48806.Doc
m.wjc.msfsx.cn/77539.Doc
m.wjc.msfsx.cn/59335.Doc
m.wjx.msfsx.cn/04264.Doc
m.wjx.msfsx.cn/44400.Doc
m.wjx.msfsx.cn/55571.Doc
m.wjx.msfsx.cn/57377.Doc
m.wjx.msfsx.cn/35999.Doc
m.wjx.msfsx.cn/42246.Doc
m.wjx.msfsx.cn/79337.Doc
m.wjx.msfsx.cn/91977.Doc
m.wjx.msfsx.cn/62082.Doc
m.wjx.msfsx.cn/22086.Doc
m.wjx.msfsx.cn/37977.Doc
m.wjx.msfsx.cn/97551.Doc
m.wjx.msfsx.cn/40800.Doc
m.wjx.msfsx.cn/20082.Doc
m.wjx.msfsx.cn/20828.Doc
m.wjx.msfsx.cn/35779.Doc
m.wjx.msfsx.cn/44268.Doc
m.wjx.msfsx.cn/08080.Doc
m.wjx.msfsx.cn/35775.Doc
m.wjx.msfsx.cn/04444.Doc
m.wjz.msfsx.cn/00480.Doc
m.wjz.msfsx.cn/84002.Doc
m.wjz.msfsx.cn/08424.Doc
m.wjz.msfsx.cn/28088.Doc
m.wjz.msfsx.cn/57355.Doc
m.wjz.msfsx.cn/00202.Doc
m.wjz.msfsx.cn/31513.Doc
m.wjz.msfsx.cn/55939.Doc
m.wjz.msfsx.cn/22086.Doc
m.wjz.msfsx.cn/46464.Doc
m.wjz.msfsx.cn/40080.Doc
m.wjz.msfsx.cn/19195.Doc
m.wjz.msfsx.cn/13575.Doc
m.wjz.msfsx.cn/11993.Doc
m.wjz.msfsx.cn/11719.Doc
m.wjz.msfsx.cn/20082.Doc
m.wjz.msfsx.cn/02860.Doc
m.wjz.msfsx.cn/31197.Doc
m.wjz.msfsx.cn/44220.Doc
m.wjz.msfsx.cn/28262.Doc
m.wjl.msfsx.cn/66666.Doc
m.wjl.msfsx.cn/60666.Doc
m.wjl.msfsx.cn/40000.Doc
m.wjl.msfsx.cn/44824.Doc
m.wjl.msfsx.cn/06644.Doc
m.wjl.msfsx.cn/00464.Doc
m.wjl.msfsx.cn/33515.Doc
m.wjl.msfsx.cn/13159.Doc
m.wjl.msfsx.cn/22202.Doc
m.wjl.msfsx.cn/68242.Doc
m.wjl.msfsx.cn/31937.Doc
m.wjl.msfsx.cn/62880.Doc
m.wjl.msfsx.cn/53555.Doc
m.wjl.msfsx.cn/08446.Doc
m.wjl.msfsx.cn/88482.Doc
m.wjl.msfsx.cn/48868.Doc
m.wjl.msfsx.cn/95397.Doc
m.wjl.msfsx.cn/60264.Doc
m.wjl.msfsx.cn/00228.Doc
m.wjl.msfsx.cn/73971.Doc
m.wjk.msfsx.cn/00686.Doc
m.wjk.msfsx.cn/73571.Doc
m.wjk.msfsx.cn/11537.Doc
m.wjk.msfsx.cn/11335.Doc
m.wjk.msfsx.cn/00466.Doc
m.wjk.msfsx.cn/75771.Doc
m.wjk.msfsx.cn/79311.Doc
m.wjk.msfsx.cn/57755.Doc
m.wjk.msfsx.cn/40080.Doc
m.wjk.msfsx.cn/95773.Doc
m.wjk.msfsx.cn/64426.Doc
m.wjk.msfsx.cn/17393.Doc
m.wjk.msfsx.cn/11177.Doc
m.wjk.msfsx.cn/86022.Doc
m.wjk.msfsx.cn/46648.Doc
m.wjk.msfsx.cn/84222.Doc
m.wjk.msfsx.cn/28602.Doc
m.wjk.msfsx.cn/60488.Doc
m.wjk.msfsx.cn/11317.Doc
m.wjk.msfsx.cn/99333.Doc
m.wjj.msfsx.cn/02884.Doc
m.wjj.msfsx.cn/26040.Doc
m.wjj.msfsx.cn/66044.Doc
m.wjj.msfsx.cn/19995.Doc
m.wjj.msfsx.cn/37995.Doc
m.wjj.msfsx.cn/59777.Doc
m.wjj.msfsx.cn/31733.Doc
m.wjj.msfsx.cn/68444.Doc
m.wjj.msfsx.cn/17759.Doc
m.wjj.msfsx.cn/66242.Doc
m.wjj.msfsx.cn/44260.Doc
m.wjj.msfsx.cn/22682.Doc
m.wjj.msfsx.cn/71135.Doc
m.wjj.msfsx.cn/11155.Doc
m.wjj.msfsx.cn/59937.Doc
m.wjj.msfsx.cn/79371.Doc
m.wjj.msfsx.cn/06686.Doc
m.wjj.msfsx.cn/60486.Doc
m.wjj.msfsx.cn/80260.Doc
m.wjj.msfsx.cn/97115.Doc
m.wjh.msfsx.cn/64024.Doc
m.wjh.msfsx.cn/00846.Doc
m.wjh.msfsx.cn/46864.Doc
m.wjh.msfsx.cn/79713.Doc
m.wjh.msfsx.cn/20460.Doc
m.wjh.msfsx.cn/53179.Doc
m.wjh.msfsx.cn/39355.Doc
m.wjh.msfsx.cn/31771.Doc
m.wjh.msfsx.cn/99957.Doc
m.wjh.msfsx.cn/55195.Doc
m.wjh.msfsx.cn/08084.Doc
m.wjh.msfsx.cn/33755.Doc
m.wjh.msfsx.cn/84260.Doc
m.wjh.msfsx.cn/06844.Doc
m.wjh.msfsx.cn/28046.Doc
m.wjh.msfsx.cn/71573.Doc
m.wjh.msfsx.cn/20808.Doc
m.wjh.msfsx.cn/26260.Doc
m.wjh.msfsx.cn/00826.Doc
m.wjh.msfsx.cn/53195.Doc
m.wjg.msfsx.cn/20462.Doc
m.wjg.msfsx.cn/62000.Doc
m.wjg.msfsx.cn/55533.Doc
m.wjg.msfsx.cn/57799.Doc
m.wjg.msfsx.cn/13531.Doc
m.wjg.msfsx.cn/31971.Doc
m.wjg.msfsx.cn/40884.Doc
m.wjg.msfsx.cn/22222.Doc
m.wjg.msfsx.cn/97337.Doc
m.wjg.msfsx.cn/93737.Doc
m.wjg.msfsx.cn/44404.Doc
m.wjg.msfsx.cn/88402.Doc
m.wjg.msfsx.cn/46228.Doc
m.wjg.msfsx.cn/86468.Doc
m.wjg.msfsx.cn/00882.Doc
m.wjg.msfsx.cn/79159.Doc
m.wjg.msfsx.cn/91971.Doc
m.wjg.msfsx.cn/82046.Doc
m.wjg.msfsx.cn/77955.Doc
m.wjg.msfsx.cn/82406.Doc
m.wjf.msfsx.cn/91139.Doc
m.wjf.msfsx.cn/95155.Doc
m.wjf.msfsx.cn/75553.Doc
m.wjf.msfsx.cn/19595.Doc
m.wjf.msfsx.cn/20662.Doc
m.wjf.msfsx.cn/28868.Doc
m.wjf.msfsx.cn/88268.Doc
m.wjf.msfsx.cn/26206.Doc
m.wjf.msfsx.cn/35571.Doc
m.wjf.msfsx.cn/20848.Doc
m.wjf.msfsx.cn/53357.Doc
m.wjf.msfsx.cn/60666.Doc
m.wjf.msfsx.cn/88860.Doc
m.wjf.msfsx.cn/33999.Doc
m.wjf.msfsx.cn/26668.Doc
m.wjf.msfsx.cn/08082.Doc
m.wjf.msfsx.cn/99935.Doc
m.wjf.msfsx.cn/48862.Doc
m.wjf.msfsx.cn/84424.Doc
m.wjf.msfsx.cn/48260.Doc
m.wjd.msfsx.cn/75557.Doc
m.wjd.msfsx.cn/48266.Doc
m.wjd.msfsx.cn/11733.Doc
m.wjd.msfsx.cn/28006.Doc
m.wjd.msfsx.cn/75175.Doc
m.wjd.msfsx.cn/79715.Doc
m.wjd.msfsx.cn/68608.Doc
m.wjd.msfsx.cn/71535.Doc
m.wjd.msfsx.cn/48486.Doc
m.wjd.msfsx.cn/68668.Doc
m.wjd.msfsx.cn/53197.Doc
m.wjd.msfsx.cn/57197.Doc
m.wjd.msfsx.cn/55113.Doc
m.wjd.msfsx.cn/55555.Doc
m.wjd.msfsx.cn/06484.Doc
m.wjd.msfsx.cn/19557.Doc
m.wjd.msfsx.cn/13951.Doc
m.wjd.msfsx.cn/77997.Doc
m.wjd.msfsx.cn/82482.Doc
m.wjd.msfsx.cn/66260.Doc
m.wjs.msfsx.cn/37131.Doc
m.wjs.msfsx.cn/57733.Doc
m.wjs.msfsx.cn/17571.Doc
m.wjs.msfsx.cn/91791.Doc
m.wjs.msfsx.cn/53939.Doc
m.wjs.msfsx.cn/08046.Doc
m.wjs.msfsx.cn/22642.Doc
m.wjs.msfsx.cn/26426.Doc
m.wjs.msfsx.cn/77377.Doc
m.wjs.msfsx.cn/64242.Doc
m.wjs.msfsx.cn/02842.Doc
m.wjs.msfsx.cn/71173.Doc
m.wjs.msfsx.cn/31911.Doc
m.wjs.msfsx.cn/40444.Doc
m.wjs.msfsx.cn/44424.Doc
m.wjs.msfsx.cn/73931.Doc
m.wjs.msfsx.cn/77555.Doc
m.wjs.msfsx.cn/55395.Doc
m.wjs.msfsx.cn/19131.Doc
m.wjs.msfsx.cn/02404.Doc
m.wja.msfsx.cn/15359.Doc
m.wja.msfsx.cn/13537.Doc
m.wja.msfsx.cn/06042.Doc
m.wja.msfsx.cn/04260.Doc
m.wja.msfsx.cn/44804.Doc
m.wja.msfsx.cn/13133.Doc
m.wja.msfsx.cn/71353.Doc
m.wja.msfsx.cn/15359.Doc
m.wja.msfsx.cn/00086.Doc
m.wja.msfsx.cn/53171.Doc
m.wja.msfsx.cn/08642.Doc
m.wja.msfsx.cn/55533.Doc
m.wja.msfsx.cn/04068.Doc
m.wja.msfsx.cn/26420.Doc
m.wja.msfsx.cn/26244.Doc
m.wja.msfsx.cn/60824.Doc
m.wja.msfsx.cn/82682.Doc
m.wja.msfsx.cn/28868.Doc
m.wja.msfsx.cn/60480.Doc
m.wja.msfsx.cn/71757.Doc
m.wjp.msfsx.cn/71997.Doc
m.wjp.msfsx.cn/15379.Doc
m.wjp.msfsx.cn/84002.Doc
m.wjp.msfsx.cn/19995.Doc
m.wjp.msfsx.cn/80862.Doc
m.wjp.msfsx.cn/51173.Doc
m.wjp.msfsx.cn/84088.Doc
m.wjp.msfsx.cn/40628.Doc
m.wjp.msfsx.cn/77351.Doc
m.wjp.msfsx.cn/59979.Doc
m.wjp.msfsx.cn/02406.Doc
m.wjp.msfsx.cn/06686.Doc
m.wjp.msfsx.cn/28662.Doc
m.wjp.msfsx.cn/82086.Doc
m.wjp.msfsx.cn/60228.Doc
m.wjp.msfsx.cn/08600.Doc
m.wjp.msfsx.cn/35555.Doc
m.wjp.msfsx.cn/86008.Doc
m.wjp.msfsx.cn/48068.Doc
m.wjp.msfsx.cn/57353.Doc
m.wjo.msfsx.cn/31333.Doc
m.wjo.msfsx.cn/66884.Doc
m.wjo.msfsx.cn/57737.Doc
m.wjo.msfsx.cn/37755.Doc
m.wjo.msfsx.cn/08208.Doc
m.wjo.msfsx.cn/51317.Doc
m.wjo.msfsx.cn/73335.Doc
m.wjo.msfsx.cn/39373.Doc
m.wjo.msfsx.cn/20662.Doc
m.wjo.msfsx.cn/31731.Doc
m.wjo.msfsx.cn/79733.Doc
m.wjo.msfsx.cn/39933.Doc
m.wjo.msfsx.cn/71119.Doc
m.wjo.msfsx.cn/80488.Doc
m.wjo.msfsx.cn/22080.Doc
m.wjo.msfsx.cn/95135.Doc
m.wjo.msfsx.cn/62424.Doc
m.wjo.msfsx.cn/39131.Doc
m.wjo.msfsx.cn/40604.Doc
m.wjo.msfsx.cn/93599.Doc
m.wji.msfsx.cn/19977.Doc
m.wji.msfsx.cn/88862.Doc
m.wji.msfsx.cn/66642.Doc
m.wji.msfsx.cn/15199.Doc
m.wji.msfsx.cn/95759.Doc
m.wji.msfsx.cn/71399.Doc
m.wji.msfsx.cn/04842.Doc
m.wji.msfsx.cn/31513.Doc
m.wji.msfsx.cn/28642.Doc
m.wji.msfsx.cn/42284.Doc
m.wji.msfsx.cn/00228.Doc
m.wji.msfsx.cn/57195.Doc
m.wji.msfsx.cn/95915.Doc
m.wji.msfsx.cn/95795.Doc
m.wji.msfsx.cn/28480.Doc
m.wji.msfsx.cn/06022.Doc
m.wji.msfsx.cn/22464.Doc
m.wji.msfsx.cn/60264.Doc
m.wji.msfsx.cn/00264.Doc
m.wji.msfsx.cn/20802.Doc
m.wju.msfsx.cn/91599.Doc
m.wju.msfsx.cn/53579.Doc
m.wju.msfsx.cn/82820.Doc
m.wju.msfsx.cn/66080.Doc
m.wju.msfsx.cn/46664.Doc
m.wju.msfsx.cn/33579.Doc
m.wju.msfsx.cn/04048.Doc
m.wju.msfsx.cn/51971.Doc
m.wju.msfsx.cn/68820.Doc
m.wju.msfsx.cn/77955.Doc
m.wju.msfsx.cn/04420.Doc
m.wju.msfsx.cn/28248.Doc
m.wju.msfsx.cn/28086.Doc
m.wju.msfsx.cn/04220.Doc
m.wju.msfsx.cn/53939.Doc
m.wju.msfsx.cn/28668.Doc
m.wju.msfsx.cn/99331.Doc
m.wju.msfsx.cn/84202.Doc
m.wju.msfsx.cn/48486.Doc
m.wju.msfsx.cn/31919.Doc
m.wjy.msfsx.cn/33519.Doc
m.wjy.msfsx.cn/15175.Doc
m.wjy.msfsx.cn/88482.Doc
m.wjy.msfsx.cn/44204.Doc
m.wjy.msfsx.cn/37159.Doc
m.wjy.msfsx.cn/53373.Doc
m.wjy.msfsx.cn/08420.Doc
m.wjy.msfsx.cn/80086.Doc
m.wjy.msfsx.cn/68642.Doc
m.wjy.msfsx.cn/64884.Doc
m.wjy.msfsx.cn/73975.Doc
m.wjy.msfsx.cn/57317.Doc
m.wjy.msfsx.cn/22804.Doc
m.wjy.msfsx.cn/91535.Doc
m.wjy.msfsx.cn/82686.Doc
m.wjy.msfsx.cn/19313.Doc
m.wjy.msfsx.cn/66686.Doc
m.wjy.msfsx.cn/39799.Doc
m.wjy.msfsx.cn/19799.Doc
m.wjy.msfsx.cn/17513.Doc
m.wjt.msfsx.cn/24266.Doc
m.wjt.msfsx.cn/35155.Doc
m.wjt.msfsx.cn/62286.Doc
m.wjt.msfsx.cn/31395.Doc
m.wjt.msfsx.cn/60400.Doc
m.wjt.msfsx.cn/44426.Doc
m.wjt.msfsx.cn/84408.Doc
m.wjt.msfsx.cn/73395.Doc
m.wjt.msfsx.cn/44040.Doc
m.wjt.msfsx.cn/59795.Doc
m.wjt.msfsx.cn/71991.Doc
m.wjt.msfsx.cn/20848.Doc
m.wjt.msfsx.cn/15757.Doc
m.wjt.msfsx.cn/62882.Doc
m.wjt.msfsx.cn/20022.Doc
m.wjt.msfsx.cn/64820.Doc
m.wjt.msfsx.cn/04640.Doc
m.wjt.msfsx.cn/53191.Doc
m.wjt.msfsx.cn/68666.Doc
m.wjt.msfsx.cn/48246.Doc
m.wjr.msfsx.cn/91377.Doc
m.wjr.msfsx.cn/55511.Doc
m.wjr.msfsx.cn/79975.Doc
m.wjr.msfsx.cn/62028.Doc
m.wjr.msfsx.cn/39577.Doc
