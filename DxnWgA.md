懊菜乱敛夹


Python Redis 高级数据结构实战
===============================

本文深入 Redis 的高级数据结构，涵盖 Streams、Geospatial、HyperLogLog、
Bitmap、RedisJSON、RediSearch 以及 Pipeline/事务等核心技术。

一、Redis Streams 消息队列
--------------------------

Stream 是 Redis 5.0 引入的日志式数据结构，支持消费者组，
适合做消息队列、事件溯源等场景。

import redis

# 连接 Redis（生产环境建议使用连接池）
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# --- 生产者：添加消息到 Stream ---
def produce_messages():
    """模拟订单消息的生产者"""
    for i in range(5):
        # XADD：向 Stream 添加消息，* 表示自动生成消息 ID
        msg_id = r.xadd(
            "order_stream",          # Stream 的 key
            {"order_id": f"ORD{i:05d}", "amount": 100 * i, "status": "created"},
            id="*"                   # 自动生成毫秒级时间戳 ID
        )
        print(f"消息已发送，ID: {msg_id}")  # ID 格式: 1678901234567-0

    # 限制 Stream 最大长度，防止无限增长
    r.xadd("log_stream", {"level": "INFO", "msg": "系统启动"},
           maxlen=1000)  # 只保留最近 1000 条


# --- 消费者：读取消息 ---
def consume_messages():
    """从 Stream 读取消息"""
    # XREAD：阻塞读取，等待新消息
    messages = r.xread(
        {"order_stream": "0"},      # 从 ID 为 0 开始（从头读取）
        count=10,                    # 最多读取 10 条
        block=5000                   # 阻塞 5 秒，无消息则超时返回
    )
    for stream_name, msgs in messages:
        for msg_id, msg_data in msgs:
            print(f"[Stream: {stream_name}] ID: {msg_id}, 数据: {msg_data}")

    # XRANGE：按时间范围读取（ID 基于时间戳）
    # 例如读取 2026-05-27 00:00:00 到 23:59:59 的消息
    start_id = "1716768000000-0"    # 起始时间戳
    end_id = "1716854399000-0"      # 结束时间戳
    ranged_msgs = r.xrange("order_stream", min=start_id, max=end_id)
    print(f"时间范围内共 {len(ranged_msgs)} 条消息")

    # XDEL：按 ID 删除消息
    r.xdel("order_stream", "1716768000000-0")  # 删除指定消息

    # XLEN：获取 Stream 长度
    stream_length = r.xlen("order_stream")
    print(f"Stream 当前长度: {stream_length}")


# --- 消费者组模式（实现消息的可靠消费）---
def consumer_group_demo():
    """使用消费者组实现消息的负载均衡"""
    # 创建消费者组
    try:
        r.xgroup_create("order_stream", "order_processor", id="0", mkstream=True)
    except redis.exceptions.ResponseError:
        print("消费者组已存在，跳过创建")

    # 消费者从组中读取消息（每条消息只被一个消费者处理）
    messages = r.xreadgroup(
        "order_processor",           # 消费者组名
        "worker_1",                  # 消费者名
        {"order_stream": ">"},       # ">" 表示只读未投递过的消息
        count=1,
        block=2000
    )

    for stream_name, msgs in messages:
        for msg_id, msg_data in msgs:
            print(f"处理订单: {msg_data}")

            # 处理完成后确认消息（XACK），防止消息被重复投递
            r.xack("order_stream", "order_processor", msg_id)

    # XPENDING：查看未确认的消息（处理失败需重试的消息）
    pending = r.xpending("order_stream", "order_processor")
    print(f"未确认消息数: {pending.get('pending', 0)}")


二、Geospatial 地理位置
-----------------------

Geo 模块支持存储经纬度坐标，计算距离和范围查询。

def geo_demo():
    """地理位置功能演示"""
    # GEOADD：添加位置（经度、纬度、名称）
    r.geoadd("stores:北京",
             (116.397128, 39.916527, "天安门"),    # (经度, 纬度, 名称)
             (116.326744, 39.913315, "故宫"),
             (116.395645, 39.929986, "雍和宫"))

    r.geoadd("stores:上海",
             (121.473701, 31.230416, "外滩"),
             (121.511621, 31.240092, "东方明珠"))

    # GEODIST：计算两个地点之间的距离
    # 默认单位是米，支持 m/km/mi/ft
    distance = r.geodist("stores:北京", "天安门", "故宫", unit="km")
    print(f"天安门到故宫距离: {distance:.2f} 公里")

    # GEORADIUS：以给定坐标为中心，查找范围内的其他位置
    # 查找天安门附近 5 公里内的所有地点
    near_by = r.georadius(
        "stores:北京",               # key
        116.397128, 39.916527,       # 中心点经纬度
        5,                           # 半径
        unit="km",                   # 单位
        withdist=True,               # 同时返回距离
        withcoord=True,              # 同时返回坐标
        sort="ASC"                   # 按距离升序
    )
    print("天安门附近 5km 内的地点:")
    for name, dist, coord in near_by:
        print(f"  {name}: {dist:.2f}km ({coord})")

    # GEORADIUSBYMEMBER：以已有成员为中心查找
    near_tiananmen = r.georadiusbymember("stores:北京", "天安门", 10, unit="km")
    print(f"天安门 10km 内的地点: {near_tiananmen}")

    # GEOHASH：获取 geohash 字符串（可用于地理编码）
    geohash = r.geohash("stores:北京", "天安门")
    print(f"天安门的 geohash: {geohash[0]}")


三、HyperLogLog 基数统计
------------------------

HyperLogLog 使用极少量内存（约 12KB）统计巨大数据集的基数（去重计数），
误差约 0.81%。适合统计 UV（独立访客）等场景。

def hyperloglog_demo():
    """HyperLogLog 基数统计"""
    # PFADD：添加元素
    r.pfadd("uv:article:1001", "user_001", "user_002", "user_003")
    r.pfadd("uv:article:1001", "user_001")  # 重复添加不会增加基数

    # PFCOUNT：获取基数估算值
    uv_article = r.pfcount("uv:article:1001")
    print(f"文章 1001 UV 估算值: {uv_article}")

    # 模拟批量 UV 统计（每天一个 key）
    import datetime
    today = datetime.date.today().isoformat()
    for user_id in [f"user_{i:03d}" for i in range(1000)]:
        r.pfadd(f"uv:daily:{today}", user_id)

    daily_uv = r.pfcount(f"uv:daily:{today}")
    print(f"今日 UV: {daily_uv}（理论值 1000，误差约 0.81%）")

    # PFMERGE：合并多个 HyperLogLog
    # 用于计算多天去重 UV（周/月 UV）
    r.pfmerge("uv:weekly:2026-W22",
              "uv:daily:2026-05-25",
              "uv:daily:2026-05-26",
              "uv:daily:2026-05-27")
    weekly_uv = r.pfcount("uv:weekly:2026-W22")
    print(f"本周 UV: {weekly_uv}")


四、Bitmap 位图操作
-------------------

Bitmap 是二进制位数组，适合做布尔型海量数据标记，如签到、登录状态。

def bitmap_demo():
    """位图操作：用户签到系统"""
    user_id = 10086

    # SETBIT：设置某一位（用户第 N 天签到）
    # 第 1 天签到
    r.setbit(f"sign:{user_id}:202605", 0, 1)   # 偏移量 0 表示第 1 天
    # 第 3 天签到
    r.setbit(f"sign:{user_id}:202605", 2, 1)   # 偏移量 2 表示第 3 天
    # 第 15 天签到
    r.setbit(f"sign:{user_id}:202605", 14, 1)  # 偏移量 14 表示第 15 天

    # GETBIT：查询某天是否签到
    day1 = r.getbit(f"sign:{user_id}:202605", 0)
    day2 = r.getbit(f"sign:{user_id}:202605", 1)
    print(f"第1天签到了吗: {'是' if day1 else '否'}")
    print(f"第2天签到了吗: {'是' if day2 else '否'}")

    # BITCOUNT：统计签到总天数
    total_days = r.bitcount(f"sign:{user_id}:202605")
    print(f"5月共签到 {total_days} 天")

    # BITOP：位运算，计算连续签到（AND/OR/XOR/NOT）
    # 示例：计算一群用户的共同签到天数
    r.setbit("team_sign:202605", 0, 1)
    r.bitop("AND", "common_sign:202605", f"sign:{user_id}:202605", "team_sign:202605")
    common_days = r.bitcount("common_sign:202605")
    print(f"和团队共同的签到天数: {common_days}")


五、RedisJSON 模块
------------------

RedisJSON 让 Redis 原生支持 JSON 文档的存储和操作。
需要 Redis 加载 RedisJSON 模块（redis-py >= 4.0 支持）。

def redisjson_demo():
    """RedisJSON 操作"""
    # 注意：需要 redis-py >= 4.0 且 Redis 加载了 RedisJSON 模块
    # JSON.SET：存储 JSON 文档
    r.execute_command("JSON.SET", "user:1001", "$",
                      '{"name": "王五", "age": 30, "address": {"city": "北京", "district": "海淀"}}')

    # JSON.GET：获取 JSON 文档（支持 JSONPath 语法）
    name = r.execute_command("JSON.GET", "user:1001", "$.name")
    print(f"用户名: {name}")  # ["王五"]

    # 获取嵌套字段
    city = r.execute_command("JSON.GET", "user:1001", "$.address.city")
    print(f"城市: {city}")

    # JSON.SET：更新特定字段
    r.execute_command("JSON.SET", "user:1001", "$.age", "31")
    print("年龄已更新为 31")

    # JSON.ARRAPPEND：向数组追加元素
    r.execute_command("JSON.SET", "user:1001", "$.tags", '["Python", "Redis"]')
    r.execute_command("JSON.ARRAPPEND", "user:1001", "$.tags", '"Golang"')
    tags = r.execute_command("JSON.GET", "user:1001", "$.tags")
    print(f"标签: {tags}")

    # JSON.DEL：删除 JSON 字段
    r.execute_command("JSON.DEL", "user:1001", "$.address.district")


六、RediSearch 全文搜索
-----------------------

RediSearch 是 Redis 的全文搜索模块，支持索引、搜索、聚合等。

def redisearch_demo():
    """全文搜索"""
    # 注意：需要 Redis 加载 RediSearch 模块
    try:
        # FT.CREATE：创建索引
        r.execute_command(
            "FT.CREATE", "idx:products",
            "ON", "HASH",              # 在 Hash 上建索引
            "PREFIX", "1", "product:", # 索引前缀为 product: 的 key
            "SCHEMA",
            "name", "TEXT", "WEIGHT", "5.0",      # name 字段权重 5
            "description", "TEXT", "WEIGHT", "1.0",
            "price", "NUMERIC",
            "tags", "TAG"
        )
    except redis.exceptions.ResponseError as e:
        if "already exists" in str(e):
            print("索引已存在")
        else:
            raise

    # 存入测试数据
    r.hset("product:1", mapping={
        "name": "Python 编程入门", "description": "适合初学者的 Python 教程",
        "price": 5900, "tags": "Python,编程"
    })
    r.hset("product:2", mapping={
        "name": "Redis 实战", "description": "Redis 数据结构与案例",
        "price": 7900, "tags": "Redis,数据库"
    })

    # FT.SEARCH：搜索
    result = r.execute_command(
        "FT.SEARCH", "idx:products", "@name:Python",  # 在 name 字段搜索 Python
        "RETURN", "3", "name", "price", "description"
    )
    print(f"搜索结果: {result}")


七、Pipeline 管道与事务
-----------------------

Pipeline 将多个命令打包发送，减少网络往返。
MULTI/EXEC 保证原子性执行。

def pipeline_and_transaction():
    """管道与事务"""
    # --- Pipeline：批量操作提升性能 ---
    pipe = r.pipeline(transaction=False)  # transaction=False 只打包不保证原子性

    # 批量设置多个 key
    for i in range(1000):
        pipe.set(f"batch:key:{i}", f"value_{i}")
    pipe.execute()  # 一次性发送所有命令，减少网络开销

    print("1000 个 key 批量写入完成")

    # --- Pipeline + 事务：保证原子性 ---
    tx_pipe = r.pipeline(transaction=True)

    # MULTI/EXEC 事务：要么全部成功，要么全部失败
    tx_pipe.multi()
    tx_pipe.decrby("account:1001", 100)  # 扣减余额
    tx_pipe.incrby("account:1002", 100)  # 增加余额
    tx_pipe.execute()  # 原子执行

    print("转账事务完成")

普兜载诙缘轮男弦仪影颂献刚烈寄

https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/YDyuYg.md
https://github.com/lippiling/oiwosr/blob/main/sXgWCk.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
