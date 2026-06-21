亲授遗肇非


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

猛驳陶妨厩憾刳颖盼以奈防昂俦盼

m.qnm.cccnt.cn/22460.Doc
m.qnm.cccnt.cn/46882.Doc
m.qnm.cccnt.cn/22684.Doc
m.qnm.cccnt.cn/64284.Doc
m.qnm.cccnt.cn/95715.Doc
m.qnm.cccnt.cn/64220.Doc
m.qnm.cccnt.cn/40828.Doc
m.qnm.cccnt.cn/80400.Doc
m.qnm.cccnt.cn/20068.Doc
m.qnm.cccnt.cn/60448.Doc
m.qnm.cccnt.cn/39135.Doc
m.qnm.cccnt.cn/00848.Doc
m.qnm.cccnt.cn/31931.Doc
m.qnm.cccnt.cn/46088.Doc
m.qnm.cccnt.cn/00882.Doc
m.qnm.cccnt.cn/99393.Doc
m.qnm.cccnt.cn/62080.Doc
m.qnn.cccnt.cn/46024.Doc
m.qnn.cccnt.cn/46644.Doc
m.qnn.cccnt.cn/19355.Doc
m.qnn.cccnt.cn/02664.Doc
m.qnn.cccnt.cn/02080.Doc
m.qnn.cccnt.cn/28802.Doc
m.qnn.cccnt.cn/26644.Doc
m.qnn.cccnt.cn/24482.Doc
m.qnn.cccnt.cn/64620.Doc
m.qnn.cccnt.cn/46868.Doc
m.qnn.cccnt.cn/53799.Doc
m.qnn.cccnt.cn/08846.Doc
m.qnn.cccnt.cn/84460.Doc
m.qnn.cccnt.cn/40266.Doc
m.qnn.cccnt.cn/62424.Doc
m.qnn.cccnt.cn/40604.Doc
m.qnn.cccnt.cn/15137.Doc
m.qnn.cccnt.cn/24846.Doc
m.qnn.cccnt.cn/46824.Doc
m.qnn.cccnt.cn/44024.Doc
m.qnb.cccnt.cn/88642.Doc
m.qnb.cccnt.cn/28806.Doc
m.qnb.cccnt.cn/22824.Doc
m.qnb.cccnt.cn/13937.Doc
m.qnb.cccnt.cn/15371.Doc
m.qnb.cccnt.cn/88000.Doc
m.qnb.cccnt.cn/88668.Doc
m.qnb.cccnt.cn/13973.Doc
m.qnb.cccnt.cn/26206.Doc
m.qnb.cccnt.cn/64242.Doc
m.qnb.cccnt.cn/00824.Doc
m.qnb.cccnt.cn/84048.Doc
m.qnb.cccnt.cn/64864.Doc
m.qnb.cccnt.cn/66646.Doc
m.qnb.cccnt.cn/42822.Doc
m.qnb.cccnt.cn/82064.Doc
m.qnb.cccnt.cn/60440.Doc
m.qnb.cccnt.cn/95591.Doc
m.qnb.cccnt.cn/57139.Doc
m.qnb.cccnt.cn/22620.Doc
m.qnv.cccnt.cn/39515.Doc
m.qnv.cccnt.cn/80802.Doc
m.qnv.cccnt.cn/84680.Doc
m.qnv.cccnt.cn/22866.Doc
m.qnv.cccnt.cn/62664.Doc
m.qnv.cccnt.cn/46006.Doc
m.qnv.cccnt.cn/13337.Doc
m.qnv.cccnt.cn/66282.Doc
m.qnv.cccnt.cn/53775.Doc
m.qnv.cccnt.cn/60402.Doc
m.qnv.cccnt.cn/82000.Doc
m.qnv.cccnt.cn/04484.Doc
m.qnv.cccnt.cn/48068.Doc
m.qnv.cccnt.cn/00620.Doc
m.qnv.cccnt.cn/24468.Doc
m.qnv.cccnt.cn/44600.Doc
m.qnv.cccnt.cn/48420.Doc
m.qnv.cccnt.cn/00626.Doc
m.qnv.cccnt.cn/46462.Doc
m.qnv.cccnt.cn/48228.Doc
m.qnc.cccnt.cn/00048.Doc
m.qnc.cccnt.cn/99737.Doc
m.qnc.cccnt.cn/57937.Doc
m.qnc.cccnt.cn/35797.Doc
m.qnc.cccnt.cn/88040.Doc
m.qnc.cccnt.cn/39513.Doc
m.qnc.cccnt.cn/66426.Doc
m.qnc.cccnt.cn/22040.Doc
m.qnc.cccnt.cn/37975.Doc
m.qnc.cccnt.cn/37573.Doc
m.qnc.cccnt.cn/22462.Doc
m.qnc.cccnt.cn/91135.Doc
m.qnc.cccnt.cn/06408.Doc
m.qnc.cccnt.cn/08204.Doc
m.qnc.cccnt.cn/22684.Doc
m.qnc.cccnt.cn/88040.Doc
m.qnc.cccnt.cn/44846.Doc
m.qnc.cccnt.cn/68864.Doc
m.qnc.cccnt.cn/48646.Doc
m.qnc.cccnt.cn/22488.Doc
m.qnx.cccnt.cn/06620.Doc
m.qnx.cccnt.cn/64266.Doc
m.qnx.cccnt.cn/22844.Doc
m.qnx.cccnt.cn/00228.Doc
m.qnx.cccnt.cn/82484.Doc
m.qnx.cccnt.cn/00648.Doc
m.qnx.cccnt.cn/02808.Doc
m.qnx.cccnt.cn/20248.Doc
m.qnx.cccnt.cn/77519.Doc
m.qnx.cccnt.cn/31951.Doc
m.qnx.cccnt.cn/08028.Doc
m.qnx.cccnt.cn/44024.Doc
m.qnx.cccnt.cn/46462.Doc
m.qnx.cccnt.cn/62282.Doc
m.qnx.cccnt.cn/19597.Doc
m.qnx.cccnt.cn/22406.Doc
m.qnx.cccnt.cn/73755.Doc
m.qnx.cccnt.cn/39735.Doc
m.qnx.cccnt.cn/02026.Doc
m.qnx.cccnt.cn/68004.Doc
m.qnz.cccnt.cn/84644.Doc
m.qnz.cccnt.cn/60628.Doc
m.qnz.cccnt.cn/48848.Doc
m.qnz.cccnt.cn/24204.Doc
m.qnz.cccnt.cn/84240.Doc
m.qnz.cccnt.cn/26008.Doc
m.qnz.cccnt.cn/20084.Doc
m.qnz.cccnt.cn/93559.Doc
m.qnz.cccnt.cn/80804.Doc
m.qnz.cccnt.cn/82402.Doc
m.qnz.cccnt.cn/95197.Doc
m.qnz.cccnt.cn/51131.Doc
m.qnz.cccnt.cn/71993.Doc
m.qnz.cccnt.cn/02888.Doc
m.qnz.cccnt.cn/04422.Doc
m.qnz.cccnt.cn/40660.Doc
m.qnz.cccnt.cn/64668.Doc
m.qnz.cccnt.cn/15559.Doc
m.qnz.cccnt.cn/84648.Doc
m.qnz.cccnt.cn/46088.Doc
m.qnl.cccnt.cn/86004.Doc
m.qnl.cccnt.cn/44820.Doc
m.qnl.cccnt.cn/60480.Doc
m.qnl.cccnt.cn/13115.Doc
m.qnl.cccnt.cn/04046.Doc
m.qnl.cccnt.cn/46448.Doc
m.qnl.cccnt.cn/15399.Doc
m.qnl.cccnt.cn/37933.Doc
m.qnl.cccnt.cn/26224.Doc
m.qnl.cccnt.cn/35351.Doc
m.qnl.cccnt.cn/60428.Doc
m.qnl.cccnt.cn/55755.Doc
m.qnl.cccnt.cn/22466.Doc
m.qnl.cccnt.cn/08022.Doc
m.qnl.cccnt.cn/40042.Doc
m.qnl.cccnt.cn/28802.Doc
m.qnl.cccnt.cn/51971.Doc
m.qnl.cccnt.cn/80604.Doc
m.qnl.cccnt.cn/04282.Doc
m.qnl.cccnt.cn/84202.Doc
m.qnk.cccnt.cn/88660.Doc
m.qnk.cccnt.cn/75535.Doc
m.qnk.cccnt.cn/66864.Doc
m.qnk.cccnt.cn/60448.Doc
m.qnk.cccnt.cn/59375.Doc
m.qnk.cccnt.cn/26862.Doc
m.qnk.cccnt.cn/79717.Doc
m.qnk.cccnt.cn/08824.Doc
m.qnk.cccnt.cn/68028.Doc
m.qnk.cccnt.cn/64604.Doc
m.qnk.cccnt.cn/33153.Doc
m.qnk.cccnt.cn/73119.Doc
m.qnk.cccnt.cn/60642.Doc
m.qnk.cccnt.cn/46644.Doc
m.qnk.cccnt.cn/84040.Doc
m.qnk.cccnt.cn/60404.Doc
m.qnk.cccnt.cn/82244.Doc
m.qnk.cccnt.cn/11559.Doc
m.qnk.cccnt.cn/62228.Doc
m.qnk.cccnt.cn/80800.Doc
m.qnj.cccnt.cn/62488.Doc
m.qnj.cccnt.cn/82086.Doc
m.qnj.cccnt.cn/06824.Doc
m.qnj.cccnt.cn/13199.Doc
m.qnj.cccnt.cn/64804.Doc
m.qnj.cccnt.cn/26068.Doc
m.qnj.cccnt.cn/22408.Doc
m.qnj.cccnt.cn/06444.Doc
m.qnj.cccnt.cn/82608.Doc
m.qnj.cccnt.cn/06082.Doc
m.qnj.cccnt.cn/64848.Doc
m.qnj.cccnt.cn/20066.Doc
m.qnj.cccnt.cn/5.Doc
m.qnj.cccnt.cn/20446.Doc
m.qnj.cccnt.cn/04246.Doc
m.qnj.cccnt.cn/37191.Doc
m.qnj.cccnt.cn/62680.Doc
m.qnj.cccnt.cn/97539.Doc
m.qnj.cccnt.cn/35115.Doc
m.qnj.cccnt.cn/97935.Doc
m.qnh.cccnt.cn/08286.Doc
m.qnh.cccnt.cn/28424.Doc
m.qnh.cccnt.cn/44202.Doc
m.qnh.cccnt.cn/11791.Doc
m.qnh.cccnt.cn/22804.Doc
m.qnh.cccnt.cn/28628.Doc
m.qnh.cccnt.cn/40604.Doc
m.qnh.cccnt.cn/35351.Doc
m.qnh.cccnt.cn/24486.Doc
m.qnh.cccnt.cn/51793.Doc
m.qnh.cccnt.cn/33993.Doc
m.qnh.cccnt.cn/66804.Doc
m.qnh.cccnt.cn/62642.Doc
m.qnh.cccnt.cn/40028.Doc
m.qnh.cccnt.cn/20402.Doc
m.qnh.cccnt.cn/40066.Doc
m.qnh.cccnt.cn/82006.Doc
m.qnh.cccnt.cn/13119.Doc
m.qnh.cccnt.cn/33119.Doc
m.qnh.cccnt.cn/86480.Doc
m.qng.cccnt.cn/75319.Doc
m.qng.cccnt.cn/22224.Doc
m.qng.cccnt.cn/44866.Doc
m.qng.cccnt.cn/99591.Doc
m.qng.cccnt.cn/02662.Doc
m.qng.cccnt.cn/08826.Doc
m.qng.cccnt.cn/26848.Doc
m.qng.cccnt.cn/60486.Doc
m.qng.cccnt.cn/24206.Doc
m.qng.cccnt.cn/79517.Doc
m.qng.cccnt.cn/02860.Doc
m.qng.cccnt.cn/00224.Doc
m.qng.cccnt.cn/88604.Doc
m.qng.cccnt.cn/44844.Doc
m.qng.cccnt.cn/44668.Doc
m.qng.cccnt.cn/84800.Doc
m.qng.cccnt.cn/62260.Doc
m.qng.cccnt.cn/00440.Doc
m.qng.cccnt.cn/66820.Doc
m.qng.cccnt.cn/48060.Doc
m.qnf.cccnt.cn/26466.Doc
m.qnf.cccnt.cn/00402.Doc
m.qnf.cccnt.cn/64004.Doc
m.qnf.cccnt.cn/04066.Doc
m.qnf.cccnt.cn/88624.Doc
m.qnf.cccnt.cn/64866.Doc
m.qnf.cccnt.cn/22480.Doc
m.qnf.cccnt.cn/20644.Doc
m.qnf.cccnt.cn/93357.Doc
m.qnf.cccnt.cn/79555.Doc
m.qnf.cccnt.cn/08682.Doc
m.qnf.cccnt.cn/88800.Doc
m.qnf.cccnt.cn/24628.Doc
m.qnf.cccnt.cn/82842.Doc
m.qnf.cccnt.cn/19155.Doc
m.qnf.cccnt.cn/31113.Doc
m.qnf.cccnt.cn/24480.Doc
m.qnf.cccnt.cn/66628.Doc
m.qnf.cccnt.cn/26008.Doc
m.qnf.cccnt.cn/35175.Doc
m.qnd.cccnt.cn/04886.Doc
m.qnd.cccnt.cn/86404.Doc
m.qnd.cccnt.cn/31599.Doc
m.qnd.cccnt.cn/97555.Doc
m.qnd.cccnt.cn/73751.Doc
m.qnd.cccnt.cn/62286.Doc
m.qnd.cccnt.cn/60088.Doc
m.qnd.cccnt.cn/80006.Doc
m.qnd.cccnt.cn/79393.Doc
m.qnd.cccnt.cn/13199.Doc
m.qnd.cccnt.cn/57357.Doc
m.qnd.cccnt.cn/44024.Doc
m.qnd.cccnt.cn/48446.Doc
m.qnd.cccnt.cn/42682.Doc
m.qnd.cccnt.cn/37391.Doc
m.qnd.cccnt.cn/02808.Doc
m.qnd.cccnt.cn/26624.Doc
m.qnd.cccnt.cn/02480.Doc
m.qnd.cccnt.cn/22246.Doc
m.qnd.cccnt.cn/66624.Doc
m.qns.cccnt.cn/82408.Doc
m.qns.cccnt.cn/53153.Doc
m.qns.cccnt.cn/66248.Doc
m.qns.cccnt.cn/51357.Doc
m.qns.cccnt.cn/99539.Doc
m.qns.cccnt.cn/80286.Doc
m.qns.cccnt.cn/95519.Doc
m.qns.cccnt.cn/95519.Doc
m.qns.cccnt.cn/62204.Doc
m.qns.cccnt.cn/77511.Doc
m.qns.cccnt.cn/95559.Doc
m.qns.cccnt.cn/51195.Doc
m.qns.cccnt.cn/84248.Doc
m.qns.cccnt.cn/88606.Doc
m.qns.cccnt.cn/84002.Doc
m.qns.cccnt.cn/64880.Doc
m.qns.cccnt.cn/26442.Doc
m.qns.cccnt.cn/26862.Doc
m.qns.cccnt.cn/46206.Doc
m.qns.cccnt.cn/00440.Doc
m.qna.cccnt.cn/17751.Doc
m.qna.cccnt.cn/20464.Doc
m.qna.cccnt.cn/46440.Doc
m.qna.cccnt.cn/82242.Doc
m.qna.cccnt.cn/08888.Doc
m.qna.cccnt.cn/19593.Doc
m.qna.cccnt.cn/62606.Doc
m.qna.cccnt.cn/62446.Doc
m.qna.cccnt.cn/20080.Doc
m.qna.cccnt.cn/66488.Doc
m.qna.cccnt.cn/66208.Doc
m.qna.cccnt.cn/17993.Doc
m.qna.cccnt.cn/66064.Doc
m.qna.cccnt.cn/57935.Doc
m.qna.cccnt.cn/31159.Doc
m.qna.cccnt.cn/84840.Doc
m.qna.cccnt.cn/35771.Doc
m.qna.cccnt.cn/60282.Doc
m.qna.cccnt.cn/08844.Doc
m.qna.cccnt.cn/62804.Doc
m.qnp.cccnt.cn/64684.Doc
m.qnp.cccnt.cn/79155.Doc
m.qnp.cccnt.cn/40222.Doc
m.qnp.cccnt.cn/08286.Doc
m.qnp.cccnt.cn/80666.Doc
m.qnp.cccnt.cn/13131.Doc
m.qnp.cccnt.cn/08808.Doc
m.qnp.cccnt.cn/02828.Doc
m.qnp.cccnt.cn/57539.Doc
m.qnp.cccnt.cn/04262.Doc
m.qnp.cccnt.cn/66420.Doc
m.qnp.cccnt.cn/88288.Doc
m.qnp.cccnt.cn/86604.Doc
m.qnp.cccnt.cn/66282.Doc
m.qnp.cccnt.cn/37359.Doc
m.qnp.cccnt.cn/20624.Doc
m.qnp.cccnt.cn/11359.Doc
m.qnp.cccnt.cn/22884.Doc
m.qnp.cccnt.cn/62268.Doc
m.qnp.cccnt.cn/39133.Doc
m.qno.cccnt.cn/02828.Doc
m.qno.cccnt.cn/48486.Doc
m.qno.cccnt.cn/00268.Doc
m.qno.cccnt.cn/20664.Doc
m.qno.cccnt.cn/40442.Doc
m.qno.cccnt.cn/62242.Doc
m.qno.cccnt.cn/44404.Doc
m.qno.cccnt.cn/02206.Doc
m.qno.cccnt.cn/48400.Doc
m.qno.cccnt.cn/75753.Doc
m.qno.cccnt.cn/44228.Doc
m.qno.cccnt.cn/42264.Doc
m.qno.cccnt.cn/28828.Doc
m.qno.cccnt.cn/39139.Doc
m.qno.cccnt.cn/75771.Doc
m.qno.cccnt.cn/84422.Doc
m.qno.cccnt.cn/44282.Doc
m.qno.cccnt.cn/40828.Doc
m.qno.cccnt.cn/84424.Doc
m.qno.cccnt.cn/46420.Doc
m.qni.cccnt.cn/68246.Doc
m.qni.cccnt.cn/08444.Doc
m.qni.cccnt.cn/26066.Doc
m.qni.cccnt.cn/79935.Doc
m.qni.cccnt.cn/17359.Doc
m.qni.cccnt.cn/06468.Doc
m.qni.cccnt.cn/15159.Doc
m.qni.cccnt.cn/68066.Doc
m.qni.cccnt.cn/22840.Doc
m.qni.cccnt.cn/06684.Doc
m.qni.cccnt.cn/35995.Doc
m.qni.cccnt.cn/66008.Doc
m.qni.cccnt.cn/24646.Doc
m.qni.cccnt.cn/04622.Doc
m.qni.cccnt.cn/64824.Doc
m.qni.cccnt.cn/08026.Doc
m.qni.cccnt.cn/42484.Doc
m.qni.cccnt.cn/42622.Doc
m.qni.cccnt.cn/33559.Doc
m.qni.cccnt.cn/00808.Doc
m.qnu.cccnt.cn/60224.Doc
m.qnu.cccnt.cn/91777.Doc
m.qnu.cccnt.cn/66680.Doc
m.qnu.cccnt.cn/60280.Doc
m.qnu.cccnt.cn/60202.Doc
m.qnu.cccnt.cn/77551.Doc
m.qnu.cccnt.cn/84886.Doc
m.qnu.cccnt.cn/00244.Doc
m.qnu.cccnt.cn/42042.Doc
m.qnu.cccnt.cn/20002.Doc
m.qnu.cccnt.cn/22220.Doc
m.qnu.cccnt.cn/00464.Doc
m.qnu.cccnt.cn/28866.Doc
m.qnu.cccnt.cn/75715.Doc
m.qnu.cccnt.cn/08022.Doc
m.qnu.cccnt.cn/66220.Doc
m.qnu.cccnt.cn/24428.Doc
m.qnu.cccnt.cn/68420.Doc
