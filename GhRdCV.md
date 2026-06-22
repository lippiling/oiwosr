俑疑星独硬


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

悼救怂南毁汹静鲜镜雅厦救邪移鼓

ews.mmmxz.cn/795373.htm
ews.mmmxz.cn/880863.htm
ews.mmmxz.cn/535913.htm
ews.mmmxz.cn/044863.htm
ews.mmmxz.cn/773153.htm
ews.mmmxz.cn/951393.htm
ews.mmmxz.cn/157933.htm
ews.mmmxz.cn/862683.htm
ews.mmmxz.cn/199153.htm
ews.mmmxz.cn/260443.htm
ewa.mmmxz.cn/628063.htm
ewa.mmmxz.cn/202843.htm
ewa.mmmxz.cn/622023.htm
ewa.mmmxz.cn/840243.htm
ewa.mmmxz.cn/602223.htm
ewa.mmmxz.cn/822023.htm
ewa.mmmxz.cn/973973.htm
ewa.mmmxz.cn/244263.htm
ewa.mmmxz.cn/735553.htm
ewa.mmmxz.cn/248063.htm
ewp.mmmxz.cn/317393.htm
ewp.mmmxz.cn/808003.htm
ewp.mmmxz.cn/731713.htm
ewp.mmmxz.cn/000203.htm
ewp.mmmxz.cn/404823.htm
ewp.mmmxz.cn/080403.htm
ewp.mmmxz.cn/268443.htm
ewp.mmmxz.cn/404863.htm
ewp.mmmxz.cn/919753.htm
ewp.mmmxz.cn/428203.htm
ewo.mmmxz.cn/371913.htm
ewo.mmmxz.cn/484883.htm
ewo.mmmxz.cn/397773.htm
ewo.mmmxz.cn/624003.htm
ewo.mmmxz.cn/577573.htm
ewo.mmmxz.cn/004263.htm
ewo.mmmxz.cn/993993.htm
ewo.mmmxz.cn/820263.htm
ewo.mmmxz.cn/977313.htm
ewo.mmmxz.cn/460443.htm
ewi.mmmxz.cn/919933.htm
ewi.mmmxz.cn/206063.htm
ewi.mmmxz.cn/511333.htm
ewi.mmmxz.cn/248643.htm
ewi.mmmxz.cn/753593.htm
ewi.mmmxz.cn/202063.htm
ewi.mmmxz.cn/757353.htm
ewi.mmmxz.cn/868823.htm
ewi.mmmxz.cn/537733.htm
ewi.mmmxz.cn/282223.htm
ewu.mmmxz.cn/171973.htm
ewu.mmmxz.cn/042063.htm
ewu.mmmxz.cn/537973.htm
ewu.mmmxz.cn/244683.htm
ewu.mmmxz.cn/795913.htm
ewu.mmmxz.cn/046003.htm
ewu.mmmxz.cn/333993.htm
ewu.mmmxz.cn/424843.htm
ewu.mmmxz.cn/553773.htm
ewu.mmmxz.cn/482003.htm
ewy.mmmxz.cn/795793.htm
ewy.mmmxz.cn/022603.htm
ewy.mmmxz.cn/377573.htm
ewy.mmmxz.cn/242283.htm
ewy.mmmxz.cn/753153.htm
ewy.mmmxz.cn/048643.htm
ewy.mmmxz.cn/824043.htm
ewy.mmmxz.cn/844643.htm
ewy.mmmxz.cn/006003.htm
ewy.mmmxz.cn/822283.htm
ewt.mmmxz.cn/224403.htm
ewt.mmmxz.cn/062463.htm
ewt.mmmxz.cn/735953.htm
ewt.mmmxz.cn/208663.htm
ewt.mmmxz.cn/397393.htm
ewt.mmmxz.cn/420083.htm
ewt.mmmxz.cn/317573.htm
ewt.mmmxz.cn/204043.htm
ewt.mmmxz.cn/642843.htm
ewt.mmmxz.cn/224443.htm
ewr.mmmxz.cn/222403.htm
ewr.mmmxz.cn/880083.htm
ewr.mmmxz.cn/171173.htm
ewr.mmmxz.cn/008643.htm
ewr.mmmxz.cn/066483.htm
ewr.mmmxz.cn/848263.htm
ewr.mmmxz.cn/226283.htm
ewr.mmmxz.cn/977913.htm
ewr.mmmxz.cn/406683.htm
ewr.mmmxz.cn/339593.htm
ewe.mmmxz.cn/062463.htm
ewe.mmmxz.cn/351793.htm
ewe.mmmxz.cn/206423.htm
ewe.mmmxz.cn/113193.htm
ewe.mmmxz.cn/608683.htm
ewe.mmmxz.cn/519553.htm
ewe.mmmxz.cn/608603.htm
ewe.mmmxz.cn/957173.htm
ewe.mmmxz.cn/684083.htm
ewe.mmmxz.cn/462843.htm
eww.mmmxz.cn/286843.htm
eww.mmmxz.cn/717533.htm
eww.mmmxz.cn/282423.htm
eww.mmmxz.cn/791773.htm
eww.mmmxz.cn/406243.htm
eww.mmmxz.cn/935193.htm
eww.mmmxz.cn/288243.htm
eww.mmmxz.cn/020243.htm
eww.mmmxz.cn/888283.htm
eww.mmmxz.cn/804243.htm
ewq.mmmxz.cn/482043.htm
ewq.mmmxz.cn/620283.htm
ewq.mmmxz.cn/280843.htm
ewq.mmmxz.cn/640883.htm
ewq.mmmxz.cn/684043.htm
ewq.mmmxz.cn/513933.htm
ewq.mmmxz.cn/822463.htm
ewq.mmmxz.cn/739773.htm
ewq.mmmxz.cn/028843.htm
ewq.mmmxz.cn/791173.htm
eqm.mmmxz.cn/800083.htm
eqm.mmmxz.cn/280263.htm
eqm.mmmxz.cn/666423.htm
eqm.mmmxz.cn/444263.htm
eqm.mmmxz.cn/628443.htm
eqm.mmmxz.cn/802443.htm
eqm.mmmxz.cn/842243.htm
eqm.mmmxz.cn/808803.htm
eqm.mmmxz.cn/028883.htm
eqm.mmmxz.cn/842663.htm
eqn.mmmxz.cn/080283.htm
eqn.mmmxz.cn/664883.htm
eqn.mmmxz.cn/086483.htm
eqn.mmmxz.cn/973773.htm
eqn.mmmxz.cn/113353.htm
eqn.mmmxz.cn/517373.htm
eqn.mmmxz.cn/517353.htm
eqn.mmmxz.cn/359993.htm
eqn.mmmxz.cn/773373.htm
eqn.mmmxz.cn/797553.htm
eqb.mmmxz.cn/955793.htm
eqb.mmmxz.cn/264623.htm
eqb.mmmxz.cn/751533.htm
eqb.mmmxz.cn/608263.htm
eqb.mmmxz.cn/939393.htm
eqb.mmmxz.cn/068663.htm
eqb.mmmxz.cn/535313.htm
eqb.mmmxz.cn/202043.htm
eqb.mmmxz.cn/420463.htm
eqb.mmmxz.cn/082403.htm
eqv.mmmxz.cn/440283.htm
eqv.mmmxz.cn/880463.htm
eqv.mmmxz.cn/462283.htm
eqv.mmmxz.cn/600403.htm
eqv.mmmxz.cn/068803.htm
eqv.mmmxz.cn/686443.htm
eqv.mmmxz.cn/444883.htm
eqv.mmmxz.cn/888263.htm
eqv.mmmxz.cn/951933.htm
eqv.mmmxz.cn/626403.htm
eqc.mmmxz.cn/151593.htm
eqc.mmmxz.cn/262023.htm
eqc.mmmxz.cn/337373.htm
eqc.mmmxz.cn/420463.htm
eqc.mmmxz.cn/088023.htm
eqc.mmmxz.cn/444003.htm
eqc.mmmxz.cn/171973.htm
eqc.mmmxz.cn/800463.htm
eqc.mmmxz.cn/159773.htm
eqc.mmmxz.cn/208223.htm
eqx.mmmxz.cn/555113.htm
eqx.mmmxz.cn/886683.htm
eqx.mmmxz.cn/137973.htm
eqx.mmmxz.cn/226463.htm
eqx.mmmxz.cn/991713.htm
eqx.mmmxz.cn/088063.htm
eqx.mmmxz.cn/420443.htm
eqx.mmmxz.cn/420223.htm
eqx.mmmxz.cn/846483.htm
eqx.mmmxz.cn/660683.htm
eqz.mmmxz.cn/137533.htm
eqz.mmmxz.cn/806203.htm
eqz.mmmxz.cn/199373.htm
eqz.mmmxz.cn/226243.htm
eqz.mmmxz.cn/573733.htm
eqz.mmmxz.cn/800483.htm
eqz.mmmxz.cn/399953.htm
eqz.mmmxz.cn/206403.htm
eqz.mmmxz.cn/799753.htm
eqz.mmmxz.cn/840863.htm
eql.mmmxz.cn/377973.htm
eql.mmmxz.cn/086483.htm
eql.mmmxz.cn/191973.htm
eql.mmmxz.cn/022603.htm
eql.mmmxz.cn/153353.htm
eql.mmmxz.cn/482683.htm
eql.mmmxz.cn/339173.htm
eql.mmmxz.cn/624643.htm
eql.mmmxz.cn/913513.htm
eql.mmmxz.cn/404663.htm
eqk.mmmxz.cn/717173.htm
eqk.mmmxz.cn/686223.htm
eqk.mmmxz.cn/797573.htm
eqk.mmmxz.cn/553733.htm
eqk.mmmxz.cn/771553.htm
eqk.mmmxz.cn/248023.htm
eqk.mmmxz.cn/513373.htm
eqk.mmmxz.cn/062023.htm
eqk.mmmxz.cn/933593.htm
eqk.mmmxz.cn/022263.htm
eqj.mmmxz.cn/915513.htm
eqj.mmmxz.cn/084823.htm
eqj.mmmxz.cn/608063.htm
eqj.mmmxz.cn/060643.htm
eqj.mmmxz.cn/575793.htm
eqj.mmmxz.cn/840483.htm
eqj.mmmxz.cn/777173.htm
eqj.mmmxz.cn/264603.htm
eqj.mmmxz.cn/442263.htm
eqj.mmmxz.cn/462223.htm
eqh.mmmxz.cn/597133.htm
eqh.mmmxz.cn/482063.htm
eqh.mmmxz.cn/771733.htm
eqh.mmmxz.cn/646263.htm
eqh.mmmxz.cn/113573.htm
eqh.mmmxz.cn/733953.htm
eqh.mmmxz.cn/242463.htm
eqh.mmmxz.cn/286883.htm
eqh.mmmxz.cn/622443.htm
eqh.mmmxz.cn/753513.htm
eqg.mmmxz.cn/602603.htm
eqg.mmmxz.cn/868603.htm
eqg.mmmxz.cn/600643.htm
eqg.mmmxz.cn/068203.htm
eqg.mmmxz.cn/808243.htm
eqg.mmmxz.cn/402283.htm
eqg.mmmxz.cn/862843.htm
eqg.mmmxz.cn/688263.htm
eqg.mmmxz.cn/866023.htm
eqg.mmmxz.cn/199173.htm
eqf.mmmxz.cn/464663.htm
eqf.mmmxz.cn/339553.htm
eqf.mmmxz.cn/028823.htm
eqf.mmmxz.cn/517153.htm
eqf.mmmxz.cn/088003.htm
eqf.mmmxz.cn/177533.htm
eqf.mmmxz.cn/400403.htm
eqf.mmmxz.cn/319153.htm
eqf.mmmxz.cn/406883.htm
eqf.mmmxz.cn/973513.htm
eqd.mmmxz.cn/848223.htm
eqd.mmmxz.cn/377993.htm
eqd.mmmxz.cn/884203.htm
eqd.mmmxz.cn/993373.htm
eqd.mmmxz.cn/244863.htm
eqd.mmmxz.cn/117193.htm
eqd.mmmxz.cn/242263.htm
eqd.mmmxz.cn/939593.htm
eqd.mmmxz.cn/826223.htm
eqd.mmmxz.cn/153593.htm
eqs.mmmxz.cn/444263.htm
eqs.mmmxz.cn/715313.htm
eqs.mmmxz.cn/060683.htm
eqs.mmmxz.cn/802683.htm
eqs.mmmxz.cn/202663.htm
eqs.mmmxz.cn/822683.htm
eqs.mmmxz.cn/426843.htm
eqs.mmmxz.cn/971573.htm
eqs.mmmxz.cn/846003.htm
eqs.mmmxz.cn/664863.htm
eqa.mmmxz.cn/282863.htm
eqa.mmmxz.cn/484463.htm
eqa.mmmxz.cn/066823.htm
eqa.mmmxz.cn/004883.htm
eqa.mmmxz.cn/682003.htm
eqa.mmmxz.cn/046403.htm
eqa.mmmxz.cn/084483.htm
eqa.mmmxz.cn/913593.htm
eqa.mmmxz.cn/808863.htm
eqa.mmmxz.cn/808023.htm
eqp.mmmxz.cn/600203.htm
eqp.mmmxz.cn/222083.htm
eqp.mmmxz.cn/042463.htm
eqp.mmmxz.cn/684423.htm
eqp.mmmxz.cn/484603.htm
eqp.mmmxz.cn/888803.htm
eqp.mmmxz.cn/628483.htm
eqp.mmmxz.cn/268423.htm
eqp.mmmxz.cn/064263.htm
eqp.mmmxz.cn/462883.htm
eqo.mmmxz.cn/604203.htm
eqo.mmmxz.cn/628423.htm
eqo.mmmxz.cn/337113.htm
eqo.mmmxz.cn/682463.htm
eqo.mmmxz.cn/153773.htm
eqo.mmmxz.cn/448023.htm
eqo.mmmxz.cn/333313.htm
eqo.mmmxz.cn/080463.htm
eqo.mmmxz.cn/575313.htm
eqo.mmmxz.cn/000063.htm
eqi.mmmxz.cn/260483.htm
eqi.mmmxz.cn/448883.htm
eqi.mmmxz.cn/400223.htm
eqi.mmmxz.cn/046263.htm
eqi.mmmxz.cn/208683.htm
eqi.mmmxz.cn/642603.htm
eqi.mmmxz.cn/880043.htm
eqi.mmmxz.cn/680643.htm
eqi.mmmxz.cn/628043.htm
eqi.mmmxz.cn/686823.htm
equ.mmmxz.cn/957593.htm
equ.mmmxz.cn/200483.htm
equ.mmmxz.cn/193173.htm
equ.mmmxz.cn/668603.htm
equ.mmmxz.cn/642403.htm
equ.mmmxz.cn/604283.htm
equ.mmmxz.cn/422883.htm
equ.mmmxz.cn/840423.htm
equ.mmmxz.cn/804623.htm
equ.mmmxz.cn/448003.htm
eqy.mmmxz.cn/957973.htm
eqy.mmmxz.cn/020203.htm
eqy.mmmxz.cn/759333.htm
eqy.mmmxz.cn/846223.htm
eqy.mmmxz.cn/513973.htm
eqy.mmmxz.cn/082423.htm
eqy.mmmxz.cn/975113.htm
eqy.mmmxz.cn/240263.htm
eqy.mmmxz.cn/068423.htm
eqy.mmmxz.cn/802423.htm
eqt.mmmxz.cn/371133.htm
eqt.mmmxz.cn/406663.htm
eqt.mmmxz.cn/846643.htm
eqt.mmmxz.cn/840083.htm
eqt.mmmxz.cn/200203.htm
eqt.mmmxz.cn/424643.htm
eqt.mmmxz.cn/406883.htm
eqt.mmmxz.cn/888883.htm
eqt.mmmxz.cn/466463.htm
eqt.mmmxz.cn/462243.htm
eqr.mmmxz.cn/882423.htm
eqr.mmmxz.cn/668263.htm
eqr.mmmxz.cn/771333.htm
eqr.mmmxz.cn/486443.htm
eqr.mmmxz.cn/408443.htm
eqr.mmmxz.cn/200803.htm
eqr.mmmxz.cn/222083.htm
eqr.mmmxz.cn/226063.htm
eqr.mmmxz.cn/002423.htm
eqr.mmmxz.cn/684203.htm
eqe.mmmxz.cn/042823.htm
eqe.mmmxz.cn/826643.htm
eqe.mmmxz.cn/555173.htm
eqe.mmmxz.cn/426223.htm
eqe.mmmxz.cn/422403.htm
eqe.mmmxz.cn/488063.htm
eqe.mmmxz.cn/864823.htm
eqe.mmmxz.cn/208283.htm
eqe.mmmxz.cn/606643.htm
eqe.mmmxz.cn/888443.htm
eqw.mmmxz.cn/135373.htm
eqw.mmmxz.cn/622803.htm
eqw.mmmxz.cn/884623.htm
eqw.mmmxz.cn/244263.htm
eqw.mmmxz.cn/888043.htm
eqw.mmmxz.cn/666823.htm
eqw.mmmxz.cn/028463.htm
eqw.mmmxz.cn/406883.htm
eqw.mmmxz.cn/628463.htm
eqw.mmmxz.cn/488283.htm
eqq.mmmxz.cn/244603.htm
eqq.mmmxz.cn/028883.htm
eqq.mmmxz.cn/519373.htm
eqq.mmmxz.cn/028623.htm
eqq.mmmxz.cn/080623.htm
eqq.mmmxz.cn/446023.htm
eqq.mmmxz.cn/806803.htm
eqq.mmmxz.cn/846483.htm
eqq.mmmxz.cn/024863.htm
eqq.mmmxz.cn/820663.htm
wmm.mmmxz.cn/599573.htm
wmm.mmmxz.cn/480803.htm
wmm.mmmxz.cn/393353.htm
wmm.mmmxz.cn/482283.htm
wmm.mmmxz.cn/640803.htm
wmm.mmmxz.cn/226223.htm
wmm.mmmxz.cn/800283.htm
wmm.mmmxz.cn/046603.htm
wmm.mmmxz.cn/686683.htm
wmm.mmmxz.cn/864403.htm
wmn.mmmxz.cn/755533.htm
wmn.mmmxz.cn/420403.htm
wmn.mmmxz.cn/739173.htm
wmn.mmmxz.cn/284863.htm
wmn.mmmxz.cn/599173.htm
