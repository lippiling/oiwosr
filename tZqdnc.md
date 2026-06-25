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

eyb.xkzjnc.cn/84024.Doc
eyb.xkzjnc.cn/06868.Doc
eyb.xkzjnc.cn/64260.Doc
eyb.xkzjnc.cn/22646.Doc
eyb.xkzjnc.cn/26800.Doc
eyb.xkzjnc.cn/28046.Doc
eyb.xkzjnc.cn/82824.Doc
eyb.xkzjnc.cn/04622.Doc
eyb.xkzjnc.cn/17115.Doc
eyb.xkzjnc.cn/26402.Doc
eyv.xkzjnc.cn/62244.Doc
eyv.xkzjnc.cn/77333.Doc
eyv.xkzjnc.cn/84404.Doc
eyv.xkzjnc.cn/82208.Doc
eyv.xkzjnc.cn/20040.Doc
eyv.xkzjnc.cn/20848.Doc
eyv.xkzjnc.cn/97715.Doc
eyv.xkzjnc.cn/04442.Doc
eyv.xkzjnc.cn/24024.Doc
eyv.xkzjnc.cn/20624.Doc
eyc.xkzjnc.cn/46644.Doc
eyc.xkzjnc.cn/86402.Doc
eyc.xkzjnc.cn/39791.Doc
eyc.xkzjnc.cn/62866.Doc
eyc.xkzjnc.cn/40640.Doc
eyc.xkzjnc.cn/04280.Doc
eyc.xkzjnc.cn/42220.Doc
eyc.xkzjnc.cn/60882.Doc
eyc.xkzjnc.cn/82628.Doc
eyc.xkzjnc.cn/64864.Doc
eyx.xkzjnc.cn/44482.Doc
eyx.xkzjnc.cn/66620.Doc
eyx.xkzjnc.cn/79993.Doc
eyx.xkzjnc.cn/02426.Doc
eyx.xkzjnc.cn/40228.Doc
eyx.xkzjnc.cn/91977.Doc
eyx.xkzjnc.cn/82088.Doc
eyx.xkzjnc.cn/26680.Doc
eyx.xkzjnc.cn/22882.Doc
eyx.xkzjnc.cn/48688.Doc
eyz.xkzjnc.cn/22244.Doc
eyz.xkzjnc.cn/04268.Doc
eyz.xkzjnc.cn/06408.Doc
eyz.xkzjnc.cn/20680.Doc
eyz.xkzjnc.cn/04468.Doc
eyz.xkzjnc.cn/08044.Doc
eyz.xkzjnc.cn/57197.Doc
eyz.xkzjnc.cn/04262.Doc
eyz.xkzjnc.cn/55517.Doc
eyz.xkzjnc.cn/26842.Doc
eyl.xkzjnc.cn/93159.Doc
eyl.xkzjnc.cn/02284.Doc
eyl.xkzjnc.cn/66828.Doc
eyl.xkzjnc.cn/48286.Doc
eyl.xkzjnc.cn/62284.Doc
eyl.xkzjnc.cn/48808.Doc
eyl.xkzjnc.cn/00086.Doc
eyl.xkzjnc.cn/20626.Doc
eyl.xkzjnc.cn/55131.Doc
eyl.xkzjnc.cn/46860.Doc
eyk.xkzjnc.cn/15111.Doc
eyk.xkzjnc.cn/46042.Doc
eyk.xkzjnc.cn/00828.Doc
eyk.xkzjnc.cn/08644.Doc
eyk.xkzjnc.cn/71113.Doc
eyk.xkzjnc.cn/48086.Doc
eyk.xkzjnc.cn/33933.Doc
eyk.xkzjnc.cn/26420.Doc
eyk.xkzjnc.cn/64804.Doc
eyk.xkzjnc.cn/64608.Doc
eyj.xkzjnc.cn/22248.Doc
eyj.xkzjnc.cn/88606.Doc
eyj.xkzjnc.cn/24220.Doc
eyj.xkzjnc.cn/86626.Doc
eyj.xkzjnc.cn/66406.Doc
eyj.xkzjnc.cn/60622.Doc
eyj.xkzjnc.cn/06882.Doc
eyj.xkzjnc.cn/64800.Doc
eyj.xkzjnc.cn/42244.Doc
eyj.xkzjnc.cn/48220.Doc
eyh.xkzjnc.cn/28086.Doc
eyh.xkzjnc.cn/02442.Doc
eyh.xkzjnc.cn/00860.Doc
eyh.xkzjnc.cn/84026.Doc
eyh.xkzjnc.cn/24062.Doc
eyh.xkzjnc.cn/91711.Doc
eyh.xkzjnc.cn/02200.Doc
eyh.xkzjnc.cn/88060.Doc
eyh.xkzjnc.cn/28866.Doc
eyh.xkzjnc.cn/62248.Doc
eyg.xkzjnc.cn/44686.Doc
eyg.xkzjnc.cn/62864.Doc
eyg.xkzjnc.cn/40862.Doc
eyg.xkzjnc.cn/26608.Doc
eyg.xkzjnc.cn/66422.Doc
eyg.xkzjnc.cn/73515.Doc
eyg.xkzjnc.cn/20662.Doc
eyg.xkzjnc.cn/40226.Doc
eyg.xkzjnc.cn/02488.Doc
eyg.xkzjnc.cn/64242.Doc
eyf.xkzjnc.cn/28644.Doc
eyf.xkzjnc.cn/64280.Doc
eyf.xkzjnc.cn/48262.Doc
eyf.xkzjnc.cn/68288.Doc
eyf.xkzjnc.cn/68020.Doc
eyf.xkzjnc.cn/66680.Doc
eyf.xkzjnc.cn/66466.Doc
eyf.xkzjnc.cn/71779.Doc
eyf.xkzjnc.cn/04846.Doc
eyf.xkzjnc.cn/42088.Doc
eyd.xkzjnc.cn/00244.Doc
eyd.xkzjnc.cn/19357.Doc
eyd.xkzjnc.cn/08240.Doc
eyd.xkzjnc.cn/40446.Doc
eyd.xkzjnc.cn/20820.Doc
eyd.xkzjnc.cn/88020.Doc
eyd.xkzjnc.cn/66866.Doc
eyd.xkzjnc.cn/06082.Doc
eyd.xkzjnc.cn/08200.Doc
eyd.xkzjnc.cn/60468.Doc
eys.xkzjnc.cn/44242.Doc
eys.xkzjnc.cn/99791.Doc
eys.xkzjnc.cn/22404.Doc
eys.xkzjnc.cn/02420.Doc
eys.xkzjnc.cn/06802.Doc
eys.xkzjnc.cn/97379.Doc
eys.xkzjnc.cn/26420.Doc
eys.xkzjnc.cn/82426.Doc
eys.xkzjnc.cn/88046.Doc
eys.xkzjnc.cn/26402.Doc
eya.xkzjnc.cn/48262.Doc
eya.xkzjnc.cn/28000.Doc
eya.xkzjnc.cn/62824.Doc
eya.xkzjnc.cn/19971.Doc
eya.xkzjnc.cn/46444.Doc
eya.xkzjnc.cn/71335.Doc
eya.xkzjnc.cn/44482.Doc
eya.xkzjnc.cn/46206.Doc
eya.xkzjnc.cn/62264.Doc
eya.xkzjnc.cn/88666.Doc
eyp.xkzjnc.cn/80204.Doc
eyp.xkzjnc.cn/04222.Doc
eyp.xkzjnc.cn/86422.Doc
eyp.xkzjnc.cn/22402.Doc
eyp.xkzjnc.cn/93111.Doc
eyp.xkzjnc.cn/44468.Doc
eyp.xkzjnc.cn/86602.Doc
eyp.xkzjnc.cn/86000.Doc
eyp.xkzjnc.cn/84606.Doc
eyp.xkzjnc.cn/42204.Doc
eyo.xkzjnc.cn/22624.Doc
eyo.xkzjnc.cn/08448.Doc
eyo.xkzjnc.cn/40404.Doc
eyo.xkzjnc.cn/28640.Doc
eyo.xkzjnc.cn/64828.Doc
eyo.xkzjnc.cn/15377.Doc
eyo.xkzjnc.cn/04660.Doc
eyo.xkzjnc.cn/88806.Doc
eyo.xkzjnc.cn/82420.Doc
eyo.xkzjnc.cn/20626.Doc
eyi.xkzjnc.cn/80082.Doc
eyi.xkzjnc.cn/00462.Doc
eyi.xkzjnc.cn/20666.Doc
eyi.xkzjnc.cn/66646.Doc
eyi.xkzjnc.cn/20648.Doc
eyi.xkzjnc.cn/42084.Doc
eyi.xkzjnc.cn/88004.Doc
eyi.xkzjnc.cn/46840.Doc
eyi.xkzjnc.cn/00486.Doc
eyi.xkzjnc.cn/24642.Doc
eyu.xkzjnc.cn/06424.Doc
eyu.xkzjnc.cn/42242.Doc
eyu.xkzjnc.cn/91355.Doc
eyu.xkzjnc.cn/88828.Doc
eyu.xkzjnc.cn/66488.Doc
eyu.xkzjnc.cn/82248.Doc
eyu.xkzjnc.cn/00822.Doc
eyu.xkzjnc.cn/82240.Doc
eyu.xkzjnc.cn/22260.Doc
eyu.xkzjnc.cn/02040.Doc
eyy.xkzjnc.cn/26004.Doc
eyy.xkzjnc.cn/68620.Doc
eyy.xkzjnc.cn/68426.Doc
eyy.xkzjnc.cn/28204.Doc
eyy.xkzjnc.cn/60828.Doc
eyy.xkzjnc.cn/02628.Doc
eyy.xkzjnc.cn/84046.Doc
eyy.xkzjnc.cn/64020.Doc
eyy.xkzjnc.cn/73373.Doc
eyy.xkzjnc.cn/46840.Doc
eyt.xkzjnc.cn/84042.Doc
eyt.xkzjnc.cn/02062.Doc
eyt.xkzjnc.cn/40046.Doc
eyt.xkzjnc.cn/22284.Doc
eyt.xkzjnc.cn/20248.Doc
eyt.xkzjnc.cn/22202.Doc
eyt.xkzjnc.cn/86488.Doc
eyt.xkzjnc.cn/97915.Doc
eyt.xkzjnc.cn/64020.Doc
eyt.xkzjnc.cn/75759.Doc
eyr.xkzjnc.cn/22222.Doc
eyr.xkzjnc.cn/20044.Doc
eyr.xkzjnc.cn/06602.Doc
eyr.xkzjnc.cn/88088.Doc
eyr.xkzjnc.cn/57537.Doc
eyr.xkzjnc.cn/60244.Doc
eyr.xkzjnc.cn/62086.Doc
eyr.xkzjnc.cn/88200.Doc
eyr.xkzjnc.cn/08484.Doc
eyr.xkzjnc.cn/22000.Doc
eye.xkzjnc.cn/28082.Doc
eye.xkzjnc.cn/40404.Doc
eye.xkzjnc.cn/46082.Doc
eye.xkzjnc.cn/08068.Doc
eye.xkzjnc.cn/64268.Doc
eye.xkzjnc.cn/31337.Doc
eye.xkzjnc.cn/02668.Doc
eye.xkzjnc.cn/28466.Doc
eye.xkzjnc.cn/88028.Doc
eye.xkzjnc.cn/39517.Doc
eyw.xkzjnc.cn/04246.Doc
eyw.xkzjnc.cn/24006.Doc
eyw.xkzjnc.cn/13151.Doc
eyw.xkzjnc.cn/68480.Doc
eyw.xkzjnc.cn/75779.Doc
eyw.xkzjnc.cn/60426.Doc
eyw.xkzjnc.cn/06642.Doc
eyw.xkzjnc.cn/06886.Doc
eyw.xkzjnc.cn/48024.Doc
eyw.xkzjnc.cn/84688.Doc
eyq.xkzjnc.cn/26068.Doc
eyq.xkzjnc.cn/95551.Doc
eyq.xkzjnc.cn/42228.Doc
eyq.xkzjnc.cn/84222.Doc
eyq.xkzjnc.cn/88428.Doc
eyq.xkzjnc.cn/66482.Doc
eyq.xkzjnc.cn/60484.Doc
eyq.xkzjnc.cn/80044.Doc
eyq.xkzjnc.cn/88884.Doc
eyq.xkzjnc.cn/42604.Doc
etm.xkzjnc.cn/11539.Doc
etm.xkzjnc.cn/86460.Doc
etm.xkzjnc.cn/60824.Doc
etm.xkzjnc.cn/48088.Doc
etm.xkzjnc.cn/82826.Doc
etm.xkzjnc.cn/24620.Doc
etm.xkzjnc.cn/84840.Doc
etm.xkzjnc.cn/46060.Doc
etm.xkzjnc.cn/91317.Doc
etm.xkzjnc.cn/22820.Doc
etn.xkzjnc.cn/68466.Doc
etn.xkzjnc.cn/64804.Doc
etn.xkzjnc.cn/80222.Doc
etn.xkzjnc.cn/68028.Doc
etn.xkzjnc.cn/68428.Doc
etn.xkzjnc.cn/80622.Doc
etn.xkzjnc.cn/42628.Doc
etn.xkzjnc.cn/62222.Doc
etn.xkzjnc.cn/26844.Doc
etn.xkzjnc.cn/42404.Doc
etb.xkzjnc.cn/28628.Doc
etb.xkzjnc.cn/24000.Doc
etb.xkzjnc.cn/66660.Doc
etb.xkzjnc.cn/02806.Doc
etb.xkzjnc.cn/53795.Doc
etb.xkzjnc.cn/28424.Doc
etb.xkzjnc.cn/06626.Doc
etb.xkzjnc.cn/04626.Doc
etb.xkzjnc.cn/60422.Doc
etb.xkzjnc.cn/40044.Doc
etv.xkzjnc.cn/04008.Doc
etv.xkzjnc.cn/59999.Doc
etv.xkzjnc.cn/04222.Doc
etv.xkzjnc.cn/62004.Doc
etv.xkzjnc.cn/22624.Doc
etv.xkzjnc.cn/84680.Doc
etv.xkzjnc.cn/26842.Doc
etv.xkzjnc.cn/15179.Doc
etv.xkzjnc.cn/64480.Doc
etv.xkzjnc.cn/88022.Doc
etc.xkzjnc.cn/46826.Doc
etc.xkzjnc.cn/26646.Doc
etc.xkzjnc.cn/88022.Doc
etc.xkzjnc.cn/08404.Doc
etc.xkzjnc.cn/20286.Doc
etc.xkzjnc.cn/64288.Doc
etc.xkzjnc.cn/02608.Doc
etc.xkzjnc.cn/86002.Doc
etc.xkzjnc.cn/64648.Doc
etc.xkzjnc.cn/93779.Doc
etx.xkzjnc.cn/84280.Doc
etx.xkzjnc.cn/46086.Doc
etx.xkzjnc.cn/80666.Doc
etx.xkzjnc.cn/60884.Doc
etx.xkzjnc.cn/20082.Doc
etx.xkzjnc.cn/00008.Doc
etx.xkzjnc.cn/24680.Doc
etx.xkzjnc.cn/40628.Doc
etx.xkzjnc.cn/42646.Doc
etx.xkzjnc.cn/55993.Doc
etz.xkzjnc.cn/86202.Doc
etz.xkzjnc.cn/35795.Doc
etz.xkzjnc.cn/64202.Doc
etz.xkzjnc.cn/60206.Doc
etz.xkzjnc.cn/73351.Doc
etz.xkzjnc.cn/17559.Doc
etz.xkzjnc.cn/06244.Doc
etz.xkzjnc.cn/40068.Doc
etz.xkzjnc.cn/40824.Doc
etz.xkzjnc.cn/82400.Doc
etl.xkzjnc.cn/84668.Doc
etl.xkzjnc.cn/80604.Doc
etl.xkzjnc.cn/86626.Doc
etl.xkzjnc.cn/40084.Doc
etl.xkzjnc.cn/48240.Doc
etl.xkzjnc.cn/62062.Doc
etl.xkzjnc.cn/86846.Doc
etl.xkzjnc.cn/73311.Doc
etl.xkzjnc.cn/28008.Doc
etl.xkzjnc.cn/40488.Doc
etk.xkzjnc.cn/64482.Doc
etk.xkzjnc.cn/82402.Doc
etk.xkzjnc.cn/02822.Doc
etk.xkzjnc.cn/64224.Doc
etk.xkzjnc.cn/28424.Doc
etk.xkzjnc.cn/04842.Doc
etk.xkzjnc.cn/64484.Doc
etk.xkzjnc.cn/66402.Doc
etk.xkzjnc.cn/51739.Doc
etk.xkzjnc.cn/68264.Doc
etj.xkzjnc.cn/60822.Doc
etj.xkzjnc.cn/46422.Doc
etj.xkzjnc.cn/00444.Doc
etj.xkzjnc.cn/00066.Doc
etj.xkzjnc.cn/00608.Doc
etj.xkzjnc.cn/20240.Doc
etj.xkzjnc.cn/24280.Doc
etj.xkzjnc.cn/20088.Doc
etj.xkzjnc.cn/88826.Doc
etj.xkzjnc.cn/00008.Doc
eth.xkzjnc.cn/68840.Doc
eth.xkzjnc.cn/17979.Doc
eth.xkzjnc.cn/66222.Doc
eth.xkzjnc.cn/46288.Doc
eth.xkzjnc.cn/22884.Doc
eth.xkzjnc.cn/95511.Doc
eth.xkzjnc.cn/62406.Doc
eth.xkzjnc.cn/08848.Doc
eth.xkzjnc.cn/06684.Doc
eth.xkzjnc.cn/06808.Doc
etg.xkzjnc.cn/48208.Doc
etg.xkzjnc.cn/59339.Doc
etg.xkzjnc.cn/20262.Doc
etg.xkzjnc.cn/88806.Doc
etg.xkzjnc.cn/62600.Doc
etg.xkzjnc.cn/53917.Doc
etg.xkzjnc.cn/64282.Doc
etg.xkzjnc.cn/39995.Doc
etg.xkzjnc.cn/66604.Doc
etg.xkzjnc.cn/26684.Doc
etf.xkzjnc.cn/68682.Doc
etf.xkzjnc.cn/88240.Doc
etf.xkzjnc.cn/24080.Doc
etf.xkzjnc.cn/64844.Doc
etf.xkzjnc.cn/28608.Doc
etf.xkzjnc.cn/08828.Doc
etf.xkzjnc.cn/42662.Doc
etf.xkzjnc.cn/60086.Doc
etf.xkzjnc.cn/42062.Doc
etf.xkzjnc.cn/42240.Doc
etd.xkzjnc.cn/46402.Doc
etd.xkzjnc.cn/26426.Doc
etd.xkzjnc.cn/62846.Doc
etd.xkzjnc.cn/40282.Doc
etd.xkzjnc.cn/06268.Doc
etd.xkzjnc.cn/06224.Doc
etd.xkzjnc.cn/42088.Doc
etd.xkzjnc.cn/80082.Doc
etd.xkzjnc.cn/24260.Doc
etd.xkzjnc.cn/24028.Doc
ets.xkzjnc.cn/24468.Doc
ets.xkzjnc.cn/42626.Doc
ets.xkzjnc.cn/48404.Doc
ets.xkzjnc.cn/22008.Doc
ets.xkzjnc.cn/86008.Doc
ets.xkzjnc.cn/44848.Doc
ets.xkzjnc.cn/28640.Doc
ets.xkzjnc.cn/02662.Doc
ets.xkzjnc.cn/04008.Doc
ets.xkzjnc.cn/06462.Doc
eta.xkzjnc.cn/24864.Doc
eta.xkzjnc.cn/42826.Doc
eta.xkzjnc.cn/40844.Doc
eta.xkzjnc.cn/82406.Doc
eta.xkzjnc.cn/71751.Doc
eta.xkzjnc.cn/40868.Doc
eta.xkzjnc.cn/60204.Doc
eta.xkzjnc.cn/48608.Doc
eta.xkzjnc.cn/73913.Doc
eta.xkzjnc.cn/62460.Doc
etp.xkzjnc.cn/00222.Doc
etp.xkzjnc.cn/39951.Doc
etp.xkzjnc.cn/02048.Doc
etp.xkzjnc.cn/20660.Doc
etp.xkzjnc.cn/66484.Doc
etp.xkzjnc.cn/22446.Doc
etp.xkzjnc.cn/68006.Doc
etp.xkzjnc.cn/55337.Doc
etp.xkzjnc.cn/08648.Doc
etp.xkzjnc.cn/84662.Doc
eto.xkzjnc.cn/24088.Doc
eto.xkzjnc.cn/42840.Doc
eto.xkzjnc.cn/68204.Doc
eto.xkzjnc.cn/46864.Doc
eto.xkzjnc.cn/28408.Doc
eto.xkzjnc.cn/64268.Doc
eto.xkzjnc.cn/80444.Doc
eto.xkzjnc.cn/60444.Doc
eto.xkzjnc.cn/02086.Doc
eto.xkzjnc.cn/00480.Doc
eti.xkzjnc.cn/88826.Doc
eti.xkzjnc.cn/44004.Doc
eti.xkzjnc.cn/48886.Doc
eti.xkzjnc.cn/84660.Doc
eti.xkzjnc.cn/26840.Doc
eti.xkzjnc.cn/84624.Doc
eti.xkzjnc.cn/82842.Doc
eti.xkzjnc.cn/46288.Doc
eti.xkzjnc.cn/88062.Doc
eti.xkzjnc.cn/44882.Doc
etu.xkzjnc.cn/24464.Doc
etu.xkzjnc.cn/40468.Doc
etu.xkzjnc.cn/06642.Doc
etu.xkzjnc.cn/71519.Doc
etu.xkzjnc.cn/62484.Doc
etu.xkzjnc.cn/75571.Doc
etu.xkzjnc.cn/26406.Doc
etu.xkzjnc.cn/88086.Doc
etu.xkzjnc.cn/84662.Doc
etu.xkzjnc.cn/40680.Doc
ety.xkzjnc.cn/22624.Doc
ety.xkzjnc.cn/08226.Doc
ety.xkzjnc.cn/04646.Doc
ety.xkzjnc.cn/40480.Doc
ety.xkzjnc.cn/48620.Doc
ety.xkzjnc.cn/06202.Doc
ety.xkzjnc.cn/04440.Doc
ety.xkzjnc.cn/59715.Doc
ety.xkzjnc.cn/46662.Doc
ety.xkzjnc.cn/44684.Doc
ett.xkzjnc.cn/06266.Doc
ett.xkzjnc.cn/22284.Doc
ett.xkzjnc.cn/26600.Doc
ett.xkzjnc.cn/26428.Doc
ett.xkzjnc.cn/48266.Doc
ett.xkzjnc.cn/9.Doc
ett.xkzjnc.cn/62446.Doc
ett.xkzjnc.cn/82404.Doc
ett.xkzjnc.cn/19555.Doc
ett.xkzjnc.cn/82802.Doc
etr.xkzjnc.cn/39319.Doc
etr.xkzjnc.cn/62222.Doc
etr.xkzjnc.cn/86648.Doc
etr.xkzjnc.cn/02842.Doc
etr.xkzjnc.cn/33951.Doc
etr.xkzjnc.cn/75759.Doc
etr.xkzjnc.cn/13117.Doc
etr.xkzjnc.cn/82240.Doc
etr.xkzjnc.cn/06484.Doc
etr.xkzjnc.cn/44062.Doc
ete.xkzjnc.cn/24622.Doc
ete.xkzjnc.cn/84800.Doc
ete.xkzjnc.cn/00664.Doc
ete.xkzjnc.cn/28626.Doc
ete.xkzjnc.cn/82400.Doc
ete.xkzjnc.cn/64408.Doc
ete.xkzjnc.cn/64260.Doc
ete.xkzjnc.cn/08882.Doc
ete.xkzjnc.cn/00668.Doc
ete.xkzjnc.cn/68406.Doc
etw.xkzjnc.cn/04622.Doc
etw.xkzjnc.cn/02426.Doc
etw.xkzjnc.cn/02022.Doc
etw.xkzjnc.cn/88488.Doc
etw.xkzjnc.cn/84880.Doc
etw.xkzjnc.cn/48688.Doc
etw.xkzjnc.cn/68062.Doc
etw.xkzjnc.cn/86222.Doc
etw.xkzjnc.cn/80046.Doc
etw.xkzjnc.cn/04200.Doc
