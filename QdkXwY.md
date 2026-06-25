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

esg.xkfrnc.cn/64282.Doc
esg.xkfrnc.cn/02448.Doc
esg.xkfrnc.cn/40884.Doc
esg.xkfrnc.cn/08406.Doc
esg.xkfrnc.cn/46462.Doc
esg.xkfrnc.cn/84862.Doc
esg.xkfrnc.cn/40084.Doc
esg.xkfrnc.cn/22646.Doc
esg.xkfrnc.cn/02686.Doc
esg.xkfrnc.cn/20668.Doc
esf.xkfrnc.cn/60828.Doc
esf.xkfrnc.cn/20648.Doc
esf.xkfrnc.cn/44600.Doc
esf.xkfrnc.cn/24824.Doc
esf.xkfrnc.cn/62002.Doc
esf.xkfrnc.cn/04664.Doc
esf.xkfrnc.cn/26440.Doc
esf.xkfrnc.cn/82248.Doc
esf.xkfrnc.cn/80462.Doc
esf.xkfrnc.cn/80466.Doc
esd.xkfrnc.cn/42420.Doc
esd.xkfrnc.cn/46626.Doc
esd.xkfrnc.cn/62662.Doc
esd.xkfrnc.cn/08664.Doc
esd.xkfrnc.cn/88804.Doc
esd.xkfrnc.cn/62648.Doc
esd.xkfrnc.cn/95401.Doc
esd.xkfrnc.cn/84844.Doc
esd.xkfrnc.cn/88044.Doc
esd.xkfrnc.cn/80882.Doc
ess.xkfrnc.cn/88024.Doc
ess.xkfrnc.cn/24062.Doc
ess.xkfrnc.cn/48884.Doc
ess.xkfrnc.cn/42486.Doc
ess.xkfrnc.cn/08620.Doc
ess.xkfrnc.cn/68628.Doc
ess.xkfrnc.cn/44824.Doc
ess.xkfrnc.cn/20008.Doc
ess.xkfrnc.cn/50956.Doc
ess.xkfrnc.cn/28048.Doc
esa.xkfrnc.cn/00660.Doc
esa.xkfrnc.cn/46048.Doc
esa.xkfrnc.cn/22000.Doc
esa.xkfrnc.cn/48428.Doc
esa.xkfrnc.cn/02448.Doc
esa.xkfrnc.cn/84446.Doc
esa.xkfrnc.cn/42422.Doc
esa.xkfrnc.cn/86642.Doc
esa.xkfrnc.cn/80202.Doc
esa.xkfrnc.cn/26662.Doc
esp.xkfrnc.cn/86402.Doc
esp.xkfrnc.cn/48688.Doc
esp.xkfrnc.cn/06626.Doc
esp.xkfrnc.cn/26095.Doc
esp.xkfrnc.cn/62933.Doc
esp.xkfrnc.cn/02800.Doc
esp.xkfrnc.cn/08600.Doc
esp.xkfrnc.cn/24040.Doc
esp.xkfrnc.cn/08488.Doc
esp.xkfrnc.cn/06284.Doc
eso.xkfrnc.cn/22460.Doc
eso.xkfrnc.cn/84806.Doc
eso.xkfrnc.cn/86684.Doc
eso.xkfrnc.cn/42680.Doc
eso.xkfrnc.cn/64422.Doc
eso.xkfrnc.cn/80020.Doc
eso.xkfrnc.cn/24866.Doc
eso.xkfrnc.cn/22228.Doc
eso.xkfrnc.cn/64800.Doc
eso.xkfrnc.cn/48406.Doc
esi.xkfrnc.cn/06444.Doc
esi.xkfrnc.cn/60608.Doc
esi.xkfrnc.cn/62242.Doc
esi.xkfrnc.cn/84046.Doc
esi.xkfrnc.cn/04842.Doc
esi.xkfrnc.cn/04066.Doc
esi.xkfrnc.cn/42684.Doc
esi.xkfrnc.cn/66486.Doc
esi.xkfrnc.cn/68480.Doc
esi.xkfrnc.cn/68648.Doc
esu.xkfrnc.cn/48042.Doc
esu.xkfrnc.cn/82484.Doc
esu.xkfrnc.cn/84848.Doc
esu.xkfrnc.cn/67212.Doc
esu.xkfrnc.cn/00668.Doc
esu.xkfrnc.cn/68242.Doc
esu.xkfrnc.cn/42200.Doc
esu.xkfrnc.cn/82048.Doc
esu.xkfrnc.cn/88064.Doc
esu.xkfrnc.cn/26066.Doc
esy.xkfrnc.cn/89012.Doc
esy.xkfrnc.cn/80808.Doc
esy.xkfrnc.cn/24226.Doc
esy.xkfrnc.cn/44420.Doc
esy.xkfrnc.cn/80262.Doc
esy.xkfrnc.cn/26848.Doc
esy.xkfrnc.cn/20802.Doc
esy.xkfrnc.cn/06660.Doc
esy.xkfrnc.cn/40404.Doc
esy.xkfrnc.cn/48240.Doc
est.xkfrnc.cn/42060.Doc
est.xkfrnc.cn/24480.Doc
est.xkfrnc.cn/42686.Doc
est.xkfrnc.cn/02208.Doc
est.xkfrnc.cn/62028.Doc
est.xkfrnc.cn/26442.Doc
est.xkfrnc.cn/64084.Doc
est.xkfrnc.cn/62466.Doc
est.xkfrnc.cn/15212.Doc
est.xkfrnc.cn/62628.Doc
esr.xkfrnc.cn/08226.Doc
esr.xkfrnc.cn/06248.Doc
esr.xkfrnc.cn/48871.Doc
esr.xkfrnc.cn/22664.Doc
esr.xkfrnc.cn/66608.Doc
esr.xkfrnc.cn/60862.Doc
esr.xkfrnc.cn/46086.Doc
esr.xkfrnc.cn/26206.Doc
esr.xkfrnc.cn/42640.Doc
esr.xkfrnc.cn/04228.Doc
ese.xkfrnc.cn/26000.Doc
ese.xkfrnc.cn/46488.Doc
ese.xkfrnc.cn/04080.Doc
ese.xkfrnc.cn/22848.Doc
ese.xkfrnc.cn/48024.Doc
ese.xkfrnc.cn/80826.Doc
ese.xkfrnc.cn/86462.Doc
ese.xkfrnc.cn/08400.Doc
ese.xkfrnc.cn/24862.Doc
ese.xkfrnc.cn/64028.Doc
esw.xkfrnc.cn/22266.Doc
esw.xkfrnc.cn/40600.Doc
esw.xkfrnc.cn/63484.Doc
esw.xkfrnc.cn/62402.Doc
esw.xkfrnc.cn/66244.Doc
esw.xkfrnc.cn/46608.Doc
esw.xkfrnc.cn/42480.Doc
esw.xkfrnc.cn/88406.Doc
esw.xkfrnc.cn/86048.Doc
esw.xkfrnc.cn/68068.Doc
esq.xkfrnc.cn/64488.Doc
esq.xkfrnc.cn/62604.Doc
esq.xkfrnc.cn/15793.Doc
esq.xkfrnc.cn/22082.Doc
esq.xkfrnc.cn/62084.Doc
esq.xkfrnc.cn/02862.Doc
esq.xkfrnc.cn/48000.Doc
esq.xkfrnc.cn/60846.Doc
esq.xkfrnc.cn/24202.Doc
esq.xkfrnc.cn/84488.Doc
eam.xkfrnc.cn/76062.Doc
eam.xkfrnc.cn/28600.Doc
eam.xkfrnc.cn/02202.Doc
eam.xkfrnc.cn/22684.Doc
eam.xkfrnc.cn/20408.Doc
eam.xkfrnc.cn/64224.Doc
eam.xkfrnc.cn/62408.Doc
eam.xkfrnc.cn/60804.Doc
eam.xkfrnc.cn/60844.Doc
eam.xkfrnc.cn/08200.Doc
ean.xkfrnc.cn/02868.Doc
ean.xkfrnc.cn/20404.Doc
ean.xkfrnc.cn/42888.Doc
ean.xkfrnc.cn/22123.Doc
ean.xkfrnc.cn/06446.Doc
ean.xkfrnc.cn/66026.Doc
ean.xkfrnc.cn/42644.Doc
ean.xkfrnc.cn/64426.Doc
ean.xkfrnc.cn/44680.Doc
ean.xkfrnc.cn/48846.Doc
eab.xkfrnc.cn/24624.Doc
eab.xkfrnc.cn/08448.Doc
eab.xkfrnc.cn/82420.Doc
eab.xkfrnc.cn/88268.Doc
eab.xkfrnc.cn/40680.Doc
eab.xkfrnc.cn/88888.Doc
eab.xkfrnc.cn/84208.Doc
eab.xkfrnc.cn/42226.Doc
eab.xkfrnc.cn/06488.Doc
eab.xkfrnc.cn/00626.Doc
eav.xkfrnc.cn/00400.Doc
eav.xkfrnc.cn/60022.Doc
eav.xkfrnc.cn/86424.Doc
eav.xkfrnc.cn/60408.Doc
eav.xkfrnc.cn/44080.Doc
eav.xkfrnc.cn/06022.Doc
eav.xkfrnc.cn/42666.Doc
eav.xkfrnc.cn/80866.Doc
eav.xkfrnc.cn/09173.Doc
eav.xkfrnc.cn/68640.Doc
eac.xkfrnc.cn/20882.Doc
eac.xkfrnc.cn/62680.Doc
eac.xkfrnc.cn/84808.Doc
eac.xkfrnc.cn/60844.Doc
eac.xkfrnc.cn/60260.Doc
eac.xkfrnc.cn/20426.Doc
eac.xkfrnc.cn/60622.Doc
eac.xkfrnc.cn/64082.Doc
eac.xkfrnc.cn/86684.Doc
eac.xkfrnc.cn/46442.Doc
eax.xkfrnc.cn/02620.Doc
eax.xkfrnc.cn/42660.Doc
eax.xkfrnc.cn/08400.Doc
eax.xkfrnc.cn/02848.Doc
eax.xkfrnc.cn/88060.Doc
eax.xkfrnc.cn/00084.Doc
eax.xkfrnc.cn/14873.Doc
eax.xkfrnc.cn/46244.Doc
eax.xkfrnc.cn/46864.Doc
eax.xkfrnc.cn/64066.Doc
eaz.xkfrnc.cn/24064.Doc
eaz.xkfrnc.cn/80820.Doc
eaz.xkfrnc.cn/40622.Doc
eaz.xkfrnc.cn/60204.Doc
eaz.xkfrnc.cn/08226.Doc
eaz.xkfrnc.cn/68400.Doc
eaz.xkfrnc.cn/80888.Doc
eaz.xkfrnc.cn/44620.Doc
eaz.xkfrnc.cn/08688.Doc
eaz.xkfrnc.cn/82266.Doc
eal.xkfrnc.cn/22204.Doc
eal.xkfrnc.cn/22046.Doc
eal.xkfrnc.cn/26224.Doc
eal.xkfrnc.cn/06862.Doc
eal.xkfrnc.cn/48802.Doc
eal.xkfrnc.cn/06846.Doc
eal.xkfrnc.cn/02802.Doc
eal.xkfrnc.cn/86260.Doc
eal.xkfrnc.cn/62642.Doc
eal.xkfrnc.cn/24064.Doc
eak.xkfrnc.cn/00220.Doc
eak.xkfrnc.cn/48228.Doc
eak.xkfrnc.cn/80688.Doc
eak.xkfrnc.cn/06835.Doc
eak.xkfrnc.cn/28002.Doc
eak.xkfrnc.cn/20220.Doc
eak.xkfrnc.cn/46644.Doc
eak.xkfrnc.cn/64040.Doc
eak.xkfrnc.cn/60242.Doc
eak.xkfrnc.cn/28682.Doc
eaj.xkfrnc.cn/22820.Doc
eaj.xkfrnc.cn/06062.Doc
eaj.xkfrnc.cn/06860.Doc
eaj.xkfrnc.cn/04220.Doc
eaj.xkfrnc.cn/24662.Doc
eaj.xkfrnc.cn/84026.Doc
eaj.xkfrnc.cn/46002.Doc
eaj.xkfrnc.cn/40024.Doc
eaj.xkfrnc.cn/68882.Doc
eaj.xkfrnc.cn/68240.Doc
eah.xkfrnc.cn/86824.Doc
eah.xkfrnc.cn/68266.Doc
eah.xkfrnc.cn/26802.Doc
eah.xkfrnc.cn/88020.Doc
eah.xkfrnc.cn/44202.Doc
eah.xkfrnc.cn/68648.Doc
eah.xkfrnc.cn/68048.Doc
eah.xkfrnc.cn/08464.Doc
eah.xkfrnc.cn/40404.Doc
eah.xkfrnc.cn/68660.Doc
eag.xkfrnc.cn/79395.Doc
eag.xkfrnc.cn/00402.Doc
eag.xkfrnc.cn/42668.Doc
eag.xkfrnc.cn/04426.Doc
eag.xkfrnc.cn/24062.Doc
eag.xkfrnc.cn/64248.Doc
eag.xkfrnc.cn/82208.Doc
eag.xkfrnc.cn/86446.Doc
eag.xkfrnc.cn/00848.Doc
eag.xkfrnc.cn/04486.Doc
eaf.xkfrnc.cn/04006.Doc
eaf.xkfrnc.cn/08480.Doc
eaf.xkfrnc.cn/24220.Doc
eaf.xkfrnc.cn/64480.Doc
eaf.xkfrnc.cn/02660.Doc
eaf.xkfrnc.cn/88004.Doc
eaf.xkfrnc.cn/60620.Doc
eaf.xkfrnc.cn/91828.Doc
eaf.xkfrnc.cn/44844.Doc
eaf.xkfrnc.cn/48804.Doc
ead.xkfrnc.cn/64260.Doc
ead.xkfrnc.cn/01506.Doc
ead.xkfrnc.cn/42482.Doc
ead.xkfrnc.cn/80680.Doc
ead.xkfrnc.cn/86468.Doc
ead.xkfrnc.cn/22488.Doc
ead.xkfrnc.cn/06400.Doc
ead.xkfrnc.cn/22622.Doc
ead.xkfrnc.cn/20206.Doc
ead.xkfrnc.cn/46066.Doc
eas.xkfrnc.cn/30560.Doc
eas.xkfrnc.cn/68512.Doc
eas.xkfrnc.cn/62848.Doc
eas.xkfrnc.cn/24880.Doc
eas.xkfrnc.cn/26644.Doc
eas.xkfrnc.cn/06888.Doc
eas.xkfrnc.cn/82868.Doc
eas.xkfrnc.cn/64468.Doc
eas.xkfrnc.cn/88046.Doc
eas.xkfrnc.cn/46895.Doc
eaa.xkfrnc.cn/46664.Doc
eaa.xkfrnc.cn/24686.Doc
eaa.xkfrnc.cn/20862.Doc
eaa.xkfrnc.cn/60068.Doc
eaa.xkfrnc.cn/68820.Doc
eaa.xkfrnc.cn/28040.Doc
eaa.xkfrnc.cn/24673.Doc
eaa.xkfrnc.cn/26200.Doc
eaa.xkfrnc.cn/62242.Doc
eaa.xkfrnc.cn/99073.Doc
eap.xkfrnc.cn/88068.Doc
eap.xkfrnc.cn/64284.Doc
eap.xkfrnc.cn/28042.Doc
eap.xkfrnc.cn/44606.Doc
eap.xkfrnc.cn/46448.Doc
eap.xkfrnc.cn/66084.Doc
eap.xkfrnc.cn/68022.Doc
eap.xkfrnc.cn/64822.Doc
eap.xkfrnc.cn/24480.Doc
eap.xkfrnc.cn/64488.Doc
eao.xkfrnc.cn/46484.Doc
eao.xkfrnc.cn/20046.Doc
eao.xkfrnc.cn/40066.Doc
eao.xkfrnc.cn/80680.Doc
eao.xkfrnc.cn/82222.Doc
eao.xkfrnc.cn/60084.Doc
eao.xkfrnc.cn/46800.Doc
eao.xkfrnc.cn/64622.Doc
eao.xkfrnc.cn/04442.Doc
eao.xkfrnc.cn/86860.Doc
eai.xkfrnc.cn/22806.Doc
eai.xkfrnc.cn/40420.Doc
eai.xkfrnc.cn/04880.Doc
eai.xkfrnc.cn/68628.Doc
eai.xkfrnc.cn/04442.Doc
eai.xkfrnc.cn/62646.Doc
eai.xkfrnc.cn/66828.Doc
eai.xkfrnc.cn/00668.Doc
eai.xkfrnc.cn/48644.Doc
eai.xkfrnc.cn/88420.Doc
eau.xkfrnc.cn/82202.Doc
eau.xkfrnc.cn/86460.Doc
eau.xkfrnc.cn/84604.Doc
eau.xkfrnc.cn/88288.Doc
eau.xkfrnc.cn/48268.Doc
eau.xkfrnc.cn/24086.Doc
eau.xkfrnc.cn/22044.Doc
eau.xkfrnc.cn/04248.Doc
eau.xkfrnc.cn/46240.Doc
eau.xkfrnc.cn/08402.Doc
eay.xkfrnc.cn/40006.Doc
eay.xkfrnc.cn/80408.Doc
eay.xkfrnc.cn/04684.Doc
eay.xkfrnc.cn/00840.Doc
eay.xkfrnc.cn/68868.Doc
eay.xkfrnc.cn/88220.Doc
eay.xkfrnc.cn/62260.Doc
eay.xkfrnc.cn/86824.Doc
eay.xkfrnc.cn/02262.Doc
eay.xkfrnc.cn/68480.Doc
eat.xkfrnc.cn/02688.Doc
eat.xkfrnc.cn/80422.Doc
eat.xkfrnc.cn/44028.Doc
eat.xkfrnc.cn/86860.Doc
eat.xkfrnc.cn/62600.Doc
eat.xkfrnc.cn/20402.Doc
eat.xkfrnc.cn/80602.Doc
eat.xkfrnc.cn/86826.Doc
eat.xkfrnc.cn/22066.Doc
eat.xkfrnc.cn/04844.Doc
ear.xkfrnc.cn/60084.Doc
ear.xkfrnc.cn/48204.Doc
ear.xkfrnc.cn/62662.Doc
ear.xkfrnc.cn/22400.Doc
ear.xkfrnc.cn/26286.Doc
ear.xkfrnc.cn/04426.Doc
ear.xkfrnc.cn/00408.Doc
ear.xkfrnc.cn/48026.Doc
ear.xkfrnc.cn/40824.Doc
ear.xkfrnc.cn/62646.Doc
eae.xkfrnc.cn/66480.Doc
eae.xkfrnc.cn/64242.Doc
eae.xkfrnc.cn/60400.Doc
eae.xkfrnc.cn/68206.Doc
eae.xkfrnc.cn/80482.Doc
eae.xkfrnc.cn/68402.Doc
eae.xkfrnc.cn/04084.Doc
eae.xkfrnc.cn/80666.Doc
eae.xkfrnc.cn/66206.Doc
eae.xkfrnc.cn/82240.Doc
eaw.xkfrnc.cn/46646.Doc
eaw.xkfrnc.cn/84240.Doc
eaw.xkfrnc.cn/40002.Doc
eaw.xkfrnc.cn/48222.Doc
eaw.xkfrnc.cn/88028.Doc
eaw.xkfrnc.cn/82480.Doc
eaw.xkfrnc.cn/04060.Doc
eaw.xkfrnc.cn/48068.Doc
eaw.xkfrnc.cn/28204.Doc
eaw.xkfrnc.cn/46828.Doc
eaq.xkfrnc.cn/66848.Doc
eaq.xkfrnc.cn/28020.Doc
eaq.xkfrnc.cn/84200.Doc
eaq.xkfrnc.cn/26602.Doc
eaq.xkfrnc.cn/08422.Doc
eaq.xkfrnc.cn/08882.Doc
eaq.xkfrnc.cn/62642.Doc
eaq.xkfrnc.cn/20042.Doc
eaq.xkfrnc.cn/22422.Doc
eaq.xkfrnc.cn/28828.Doc
epm.xkfrnc.cn/88208.Doc
epm.xkfrnc.cn/60220.Doc
epm.xkfrnc.cn/66864.Doc
epm.xkfrnc.cn/68886.Doc
epm.xkfrnc.cn/24840.Doc
epm.xkfrnc.cn/64682.Doc
epm.xkfrnc.cn/66400.Doc
epm.xkfrnc.cn/88266.Doc
epm.xkfrnc.cn/66262.Doc
epm.xkfrnc.cn/60662.Doc
epn.xkfrnc.cn/66060.Doc
epn.xkfrnc.cn/62400.Doc
epn.xkfrnc.cn/80062.Doc
epn.xkfrnc.cn/28864.Doc
epn.xkfrnc.cn/02080.Doc
epn.xkfrnc.cn/04628.Doc
epn.xkfrnc.cn/42060.Doc
epn.xkfrnc.cn/40006.Doc
epn.xkfrnc.cn/66448.Doc
epn.xkfrnc.cn/60286.Doc
epb.xkfrnc.cn/64426.Doc
epb.xkfrnc.cn/26202.Doc
epb.xkfrnc.cn/42482.Doc
epb.xkfrnc.cn/04024.Doc
epb.xkfrnc.cn/26862.Doc
epb.xkfrnc.cn/66842.Doc
epb.xkfrnc.cn/66040.Doc
epb.xkfrnc.cn/80626.Doc
epb.xkfrnc.cn/82044.Doc
epb.xkfrnc.cn/82244.Doc
epv.xkfrnc.cn/20226.Doc
epv.xkfrnc.cn/84642.Doc
epv.xkfrnc.cn/42040.Doc
epv.xkfrnc.cn/60462.Doc
epv.xkfrnc.cn/88426.Doc
epv.xkfrnc.cn/46242.Doc
epv.xkfrnc.cn/24088.Doc
epv.xkfrnc.cn/82440.Doc
epv.xkfrnc.cn/88220.Doc
epv.xkfrnc.cn/00086.Doc
epc.xkfrnc.cn/04082.Doc
epc.xkfrnc.cn/26000.Doc
epc.xkfrnc.cn/02420.Doc
epc.xkfrnc.cn/64202.Doc
epc.xkfrnc.cn/60460.Doc
epc.xkfrnc.cn/08862.Doc
epc.xkfrnc.cn/84646.Doc
epc.xkfrnc.cn/64686.Doc
epc.xkfrnc.cn/04286.Doc
epc.xkfrnc.cn/28866.Doc
epx.xkfrnc.cn/60442.Doc
epx.xkfrnc.cn/66288.Doc
epx.xkfrnc.cn/44806.Doc
epx.xkfrnc.cn/44880.Doc
epx.xkfrnc.cn/24848.Doc
epx.xkfrnc.cn/84806.Doc
epx.xkfrnc.cn/22484.Doc
epx.xkfrnc.cn/26600.Doc
epx.xkfrnc.cn/42448.Doc
epx.xkfrnc.cn/22260.Doc
epz.xkfrnc.cn/68686.Doc
epz.xkfrnc.cn/26004.Doc
epz.xkfrnc.cn/48284.Doc
epz.xkfrnc.cn/42668.Doc
epz.xkfrnc.cn/26482.Doc
epz.xkfrnc.cn/46684.Doc
epz.xkfrnc.cn/80022.Doc
epz.xkfrnc.cn/62480.Doc
epz.xkfrnc.cn/88060.Doc
epz.xkfrnc.cn/28804.Doc
epl.xkfrnc.cn/04044.Doc
epl.xkfrnc.cn/00602.Doc
epl.xkfrnc.cn/02684.Doc
epl.xkfrnc.cn/40408.Doc
epl.xkfrnc.cn/66044.Doc
epl.xkfrnc.cn/46202.Doc
epl.xkfrnc.cn/02200.Doc
epl.xkfrnc.cn/60206.Doc
epl.xkfrnc.cn/20228.Doc
epl.xkfrnc.cn/46228.Doc
