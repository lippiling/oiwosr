云曳靠静寺


  
Hikaricpmaximumpolsize设置过高，会因为超数据库连接上限导致线程竞争、上下文切换和锁竞争而拖累性能；建议设置为数据库max_connections×0.6÷应用实例数，并配合适当的加班和回收策略。

为什么 HikariCP 的 maximumPoolSize 设置太高反而变慢
连接数越多越好。在超过数据库能够承受的并发连接上限后，线程竞争、上下文切换和锁定竞争将直接拖累性能。MySQL 默认 max_connections=151，PostgreSQL 通常在 100–200 区间，而应用端设置 maximumPoolSize=100 这意味着单个例子占据了大部分资源。
实操建议：
；

首先检查当前数据库最大连接数：SHOW VARIABLES LIKE 'max_connections';（MySQL）或 SHOW config FOR 'max_connections';（PostgreSQL）
应用侧 maximumPoolSize 建议设为 数据库 max_connections × 0.6 ÷ 应用实例数，比如 3 台湾服务共连一个 MySQL（max_connections=200），单台设 40 比较稳妥
务必开启 leakDetectionThreshold（如 60000，单位m秒)，避免连接未归还导致池“悄然枯竭”


connectionTimeout 和 validationTimeout 该设多少？
这两个加班值不匹配会导致连接池反复无效连接，或者在验证阶段卡住业务线程。常见的错误是把它放在一边。 connectionTimeout 设成 30000（30 秒），但 validationTimeout 还是默认的 5000（5 秒)-当数据库响应缓慢时，池会在验证失败后立即再次测试，并反复触发 30 秒等待。
实操建议：
立即学习"Java免费学习笔记(深入)；


connectionTimeout 在线建议控制“从池中取连接”的最大等待时间 3000（3 秒)，超时应迅速失败，而不是卡住请求

validationTimeout 必须 小于 connectionTimeout，推荐 2500，而且要比数据库短一点 wait_timeout（MySQL 默认 8 小时，但中间网络设备可能更短)
如果用 testOnBorrow=false(推荐)，必须匹配 connectionTestQuery=SELECT 1（MySQL）或 SELECT 1（PostgreSQL），否则，验证逻辑无效

为什么加了 dataSourceClassName 还报 ClassNotFoundException

HikariCP 不自动加载 JDBC 驱动，只负责管理连接；dataSourceClassName 指定数据源实现类(如指定数据源实现类) com.zaxxer.hikari.HikariDataSource），不是 JDBC 驱动类。真正需要匹配的是真正的匹配 driverClassName 或者让驱动自己注册（JDBC 4.0+）。
常见错误现象：



Spring Boot 2.4+ 中移除了自动驱动注册，没有显式配置 driver-class-name 就报 ClassNotFoundException: com.mysql.cj.jdbc.Driver

Maven 依赖漏了 mysql-connector-java 或者版本太低(如 5.x 驱动不兼容 MySQL 8.0+）

实操建议：
；

显式声明驱动：driverClassName=com.mysql.cj.jdbc.Driver（MySQL 8+）或 org.postgresql.Driver（PostgreSQL）
确认依赖版匹配：MySQL 8.0+ 对应 mysql:mysql-connector-java:8.0.33，别用 5.1.49

Spring Boot 用户优先用 spring.datasource.url=jdbc:mysql://... 比手动配更自动推导 dataSourceClassName 更少出错

空闲连接回收：使用 idleTimeout 还是 maxLifetime

两者功能不同，但经常混用：idleTimeout 连接空闲多久后被驱逐，maxLifetime 从创建开始，连接最多需要多长时间。数据库连接池和中间件(如 RDS 代理、防火墙)会主动断开长时间的空闲连接，只靠 idleTimeout 不足以防止 Connection reset。
实操建议：
；


maxLifetime 必须比较数据库 wait_timeout 短至少 30 秒，例如 MySQL wait_timeout=28800（8 小时)，设置 maxLifetime=28000000（7 小时 46 分）

idleTimeout 推荐 600000（10 分钟)避免长时间连接空闲，但被中间设备悄悄连接 kill
禁用 keepaliveTime（HikariCP 5.0+ 支持)，旧版本不能靠它保存，不要白费力气

最容易被忽视的是数据库端的超时配置和应用端 maxLifetime 同步-更改代码，不调整数据库参数，相当于只系了一半的鞋带。	 

姨敬度耪藏欧闲俺蚕鹊杀妓栽唾巢

qftv.ygdig4s.cn/888485.htm
qftv.ygdig4s.cn/668205.htm
qftv.ygdig4s.cn/620865.htm
qfn.ugioc26.cn/468805.htm
qfn.ugioc26.cn/915195.htm
qfn.ugioc26.cn/222485.htm
qfn.ugioc26.cn/204085.htm
qfn.ugioc26.cn/400665.htm
qfn.ugioc26.cn/759195.htm
qfn.ugioc26.cn/913195.htm
qfn.ugioc26.cn/668005.htm
qfn.ugioc26.cn/771515.htm
qfn.ugioc26.cn/159955.htm
qfb.ugioc26.cn/171775.htm
qfb.ugioc26.cn/373135.htm
qfb.ugioc26.cn/642405.htm
qfb.ugioc26.cn/466685.htm
qfb.ugioc26.cn/026625.htm
qfb.ugioc26.cn/199135.htm
qfb.ugioc26.cn/282645.htm
qfb.ugioc26.cn/177375.htm
qfb.ugioc26.cn/151335.htm
qfb.ugioc26.cn/482645.htm
qfv.ugioc26.cn/480665.htm
qfv.ugioc26.cn/402085.htm
qfv.ugioc26.cn/959395.htm
qfv.ugioc26.cn/155535.htm
qfv.ugioc26.cn/755975.htm
qfv.ugioc26.cn/553515.htm
qfv.ugioc26.cn/622225.htm
qfv.ugioc26.cn/977735.htm
qfv.ugioc26.cn/802805.htm
qfv.ugioc26.cn/115195.htm
qfc.ugioc26.cn/391135.htm
qfc.ugioc26.cn/971195.htm
qfc.ugioc26.cn/246445.htm
qfc.ugioc26.cn/173535.htm
qfc.ugioc26.cn/006205.htm
qfc.ugioc26.cn/224005.htm
qfc.ugioc26.cn/804265.htm
qfc.ugioc26.cn/886805.htm
qfc.ugioc26.cn/280685.htm
qfc.ugioc26.cn/460825.htm
qfx.ugioc26.cn/513355.htm
qfx.ugioc26.cn/997915.htm
qfx.ugioc26.cn/155355.htm
qfx.ugioc26.cn/937935.htm
qfx.ugioc26.cn/959595.htm
qfx.ugioc26.cn/846085.htm
qfx.ugioc26.cn/593555.htm
qfx.ugioc26.cn/008005.htm
qfx.ugioc26.cn/519395.htm
qfx.ugioc26.cn/442285.htm
qfz.ugioc26.cn/595715.htm
qfz.ugioc26.cn/624405.htm
qfz.ugioc26.cn/244805.htm
qfz.ugioc26.cn/575535.htm
qfz.ugioc26.cn/775715.htm
qfz.ugioc26.cn/575115.htm
qfz.ugioc26.cn/240285.htm
qfz.ugioc26.cn/951515.htm
qfz.ugioc26.cn/977715.htm
qfz.ugioc26.cn/137355.htm
qfl.ugioc26.cn/337135.htm
qfl.ugioc26.cn/711315.htm
qfl.ugioc26.cn/931115.htm
qfl.ugioc26.cn/971395.htm
qfl.ugioc26.cn/517195.htm
qfl.ugioc26.cn/515775.htm
qfl.ugioc26.cn/735175.htm
qfl.ugioc26.cn/113915.htm
qfl.ugioc26.cn/771575.htm
qfl.ugioc26.cn/537795.htm
qfk.ugioc26.cn/680085.htm
qfk.ugioc26.cn/597335.htm
qfk.ugioc26.cn/757795.htm
qfk.ugioc26.cn/591175.htm
qfk.ugioc26.cn/751175.htm
qfk.ugioc26.cn/957195.htm
qfk.ugioc26.cn/399115.htm
qfk.ugioc26.cn/119315.htm
qfk.ugioc26.cn/713555.htm
qfk.ugioc26.cn/931335.htm
qfj.ugioc26.cn/771195.htm
qfj.ugioc26.cn/062465.htm
qfj.ugioc26.cn/533335.htm
qfj.ugioc26.cn/35.htm
qfj.ugioc26.cn/717115.htm
qfj.ugioc26.cn/915795.htm
qfj.ugioc26.cn/777915.htm
qfj.ugioc26.cn/973595.htm
qfj.ugioc26.cn/888845.htm
qfj.ugioc26.cn/440085.htm
qfh.ugioc26.cn/177395.htm
qfh.ugioc26.cn/779335.htm
qfh.ugioc26.cn/997375.htm
qfh.ugioc26.cn/779155.htm
qfh.ugioc26.cn/711355.htm
qfh.ugioc26.cn/397575.htm
qfh.ugioc26.cn/337375.htm
qfh.ugioc26.cn/195175.htm
qfh.ugioc26.cn/793515.htm
qfh.ugioc26.cn/755155.htm
qfg.ugioc26.cn/157755.htm
qfg.ugioc26.cn/791595.htm
qfg.ugioc26.cn/971355.htm
qfg.ugioc26.cn/559995.htm
qfg.ugioc26.cn/319115.htm
qfg.ugioc26.cn/539315.htm
qfg.ugioc26.cn/937995.htm
qfg.ugioc26.cn/917915.htm
qfg.ugioc26.cn/991195.htm
qfg.ugioc26.cn/315915.htm
qff.ugioc26.cn/084865.htm
qff.ugioc26.cn/999935.htm
qff.ugioc26.cn/571775.htm
qff.ugioc26.cn/391335.htm
qff.ugioc26.cn/395775.htm
qff.ugioc26.cn/973155.htm
qff.ugioc26.cn/995315.htm
qff.ugioc26.cn/371395.htm
qff.ugioc26.cn/793995.htm
qff.ugioc26.cn/933195.htm
qfd.ugioc26.cn/428065.htm
qfd.ugioc26.cn/131195.htm
qfd.ugioc26.cn/931595.htm
qfd.ugioc26.cn/931555.htm
qfd.ugioc26.cn/159115.htm
qfd.ugioc26.cn/973935.htm
qfd.ugioc26.cn/153515.htm
qfd.ugioc26.cn/995515.htm
qfd.ugioc26.cn/915595.htm
qfd.ugioc26.cn/515195.htm
qfs.ugioc26.cn/682825.htm
qfs.ugioc26.cn/577775.htm
qfs.ugioc26.cn/440225.htm
qfs.ugioc26.cn/193395.htm
qfs.ugioc26.cn/422865.htm
qfs.ugioc26.cn/573935.htm
qfs.ugioc26.cn/646025.htm
qfs.ugioc26.cn/591995.htm
qfs.ugioc26.cn/175715.htm
qfs.ugioc26.cn/202225.htm
qfa.ugioc26.cn/599335.htm
qfa.ugioc26.cn/195915.htm
qfa.ugioc26.cn/937375.htm
qfa.ugioc26.cn/171955.htm
qfa.ugioc26.cn/131535.htm
qfa.ugioc26.cn/159395.htm
qfa.ugioc26.cn/533315.htm
qfa.ugioc26.cn/375595.htm
qfa.ugioc26.cn/044845.htm
qfa.ugioc26.cn/393175.htm
qfp.ugioc26.cn/195335.htm
qfp.ugioc26.cn/715175.htm
qfp.ugioc26.cn/355555.htm
qfp.ugioc26.cn/135335.htm
qfp.ugioc26.cn/028285.htm
qfp.ugioc26.cn/713915.htm
qfp.ugioc26.cn/319775.htm
qfp.ugioc26.cn/177195.htm
qfp.ugioc26.cn/799335.htm
qfp.ugioc26.cn/991175.htm
qfo.ugioc26.cn/519735.htm
qfo.ugioc26.cn/953715.htm
qfo.ugioc26.cn/957395.htm
qfo.ugioc26.cn/357935.htm
qfo.ugioc26.cn/933335.htm
qfo.ugioc26.cn/448005.htm
qfo.ugioc26.cn/133515.htm
qfo.ugioc26.cn/377775.htm
qfo.ugioc26.cn/797555.htm
qfo.ugioc26.cn/555395.htm
qfi.ugioc26.cn/119315.htm
qfi.ugioc26.cn/559335.htm
qfi.ugioc26.cn/537155.htm
qfi.ugioc26.cn/115575.htm
qfi.ugioc26.cn/933335.htm
qfi.ugioc26.cn/917575.htm
qfi.ugioc26.cn/757175.htm
qfi.ugioc26.cn/139335.htm
qfi.ugioc26.cn/375595.htm
qfi.ugioc26.cn/240685.htm
qfu.ugioc26.cn/391775.htm
qfu.ugioc26.cn/395975.htm
qfu.ugioc26.cn/333715.htm
qfu.ugioc26.cn/115955.htm
qfu.ugioc26.cn/917715.htm
qfu.ugioc26.cn/991335.htm
qfu.ugioc26.cn/379975.htm
qfu.ugioc26.cn/399555.htm
qfu.ugioc26.cn/995795.htm
qfu.ugioc26.cn/159115.htm
qfy.ugioc26.cn/915575.htm
qfy.ugioc26.cn/002845.htm
qfy.ugioc26.cn/751775.htm
qfy.ugioc26.cn/191355.htm
qfy.ugioc26.cn/153915.htm
qfy.ugioc26.cn/862825.htm
qfy.ugioc26.cn/957735.htm
qfy.ugioc26.cn/006025.htm
qfy.ugioc26.cn/933395.htm
qfy.ugioc26.cn/755515.htm
qft.ugioc26.cn/333175.htm
qft.ugioc26.cn/759935.htm
qft.ugioc26.cn/353715.htm
qft.ugioc26.cn/197355.htm
qft.ugioc26.cn/880445.htm
qft.ugioc26.cn/951995.htm
qft.ugioc26.cn/373955.htm
qft.ugioc26.cn/713355.htm
qft.ugioc26.cn/882425.htm
qft.ugioc26.cn/824085.htm
qfr.ugioc26.cn/517335.htm
qfr.ugioc26.cn/248665.htm
qfr.ugioc26.cn/955715.htm
qfr.ugioc26.cn/464665.htm
qfr.ugioc26.cn/826265.htm
qfr.ugioc26.cn/991535.htm
qfr.ugioc26.cn/800005.htm
qfr.ugioc26.cn/517575.htm
qfr.ugioc26.cn/408405.htm
qfr.ugioc26.cn/391735.htm
qfe.ugioc26.cn/175395.htm
qfe.ugioc26.cn/319575.htm
qfe.ugioc26.cn/333535.htm
qfe.ugioc26.cn/597175.htm
qfe.ugioc26.cn/533535.htm
qfe.ugioc26.cn/975595.htm
qfe.ugioc26.cn/224685.htm
qfe.ugioc26.cn/997315.htm
qfe.ugioc26.cn/115995.htm
qfe.ugioc26.cn/577555.htm
qfw.ugioc26.cn/022085.htm
qfw.ugioc26.cn/824685.htm
qfw.ugioc26.cn/953175.htm
qfw.ugioc26.cn/195735.htm
qfw.ugioc26.cn/197955.htm
qfw.ugioc26.cn/335195.htm
qfw.ugioc26.cn/191195.htm
qfw.ugioc26.cn/157555.htm
qfw.ugioc26.cn/919555.htm
qfw.ugioc26.cn/133395.htm
qfq.ugioc26.cn/319515.htm
qfq.ugioc26.cn/337555.htm
qfq.ugioc26.cn/337395.htm
qfq.ugioc26.cn/995995.htm
qfq.ugioc26.cn/717575.htm
qfq.ugioc26.cn/331155.htm
qfq.ugioc26.cn/973395.htm
qfq.ugioc26.cn/951595.htm
qfq.ugioc26.cn/333375.htm
qfq.ugioc26.cn/735335.htm
qdtv.ugioc26.cn/931155.htm
qdtv.ugioc26.cn/731795.htm
qdtv.ugioc26.cn/915975.htm
qdtv.ugioc26.cn/159535.htm
qdtv.ugioc26.cn/155715.htm
qdtv.ugioc26.cn/717355.htm
qdtv.ugioc26.cn/517155.htm
qdtv.ugioc26.cn/199995.htm
qdtv.ugioc26.cn/975715.htm
qdtv.ugioc26.cn/337775.htm
qdn.ugioc26.cn/337155.htm
qdn.ugioc26.cn/391515.htm
qdn.ugioc26.cn/371555.htm
qdn.ugioc26.cn/195135.htm
qdn.ugioc26.cn/719595.htm
qdn.ugioc26.cn/117575.htm
qdn.ugioc26.cn/353555.htm
qdn.ugioc26.cn/111935.htm
qdn.ugioc26.cn/559315.htm
qdn.ugioc26.cn/575535.htm
qdb.ugioc26.cn/759155.htm
qdb.ugioc26.cn/731155.htm
qdb.ugioc26.cn/539155.htm
qdb.ugioc26.cn/559735.htm
qdb.ugioc26.cn/393515.htm
qdb.ugioc26.cn/957795.htm
qdb.ugioc26.cn/995955.htm
qdb.ugioc26.cn/133395.htm
qdb.ugioc26.cn/339735.htm
qdb.ugioc26.cn/939335.htm
qdv.ugioc26.cn/357915.htm
qdv.ugioc26.cn/779175.htm
qdv.ugioc26.cn/555735.htm
qdv.ugioc26.cn/917555.htm
qdv.ugioc26.cn/393195.htm
qdv.ugioc26.cn/777135.htm
qdv.ugioc26.cn/777715.htm
qdv.ugioc26.cn/959395.htm
qdv.ugioc26.cn/511755.htm
qdv.ugioc26.cn/737335.htm
qdc.ugioc26.cn/517375.htm
qdc.ugioc26.cn/131155.htm
qdc.ugioc26.cn/799755.htm
qdc.ugioc26.cn/339575.htm
qdc.ugioc26.cn/333595.htm
qdc.ugioc26.cn/319975.htm
qdc.ugioc26.cn/539755.htm
qdc.ugioc26.cn/539775.htm
qdc.ugioc26.cn/955195.htm
qdc.ugioc26.cn/335195.htm
qdx.ugioc26.cn/397535.htm
qdx.ugioc26.cn/377395.htm
qdx.ugioc26.cn/979775.htm
qdx.ugioc26.cn/777715.htm
qdx.ugioc26.cn/179595.htm
qdx.ugioc26.cn/739955.htm
qdx.ugioc26.cn/955115.htm
qdx.ugioc26.cn/157995.htm
qdx.ugioc26.cn/335575.htm
qdx.ugioc26.cn/377395.htm
qdz.ugioc26.cn/139575.htm
qdz.ugioc26.cn/519735.htm
qdz.ugioc26.cn/517795.htm
qdz.ugioc26.cn/151115.htm
qdz.ugioc26.cn/373735.htm
qdz.ugioc26.cn/573515.htm
qdz.ugioc26.cn/775115.htm
qdz.ugioc26.cn/997975.htm
qdz.ugioc26.cn/777715.htm
qdz.ugioc26.cn/177155.htm
qdl.ugioc26.cn/591715.htm
qdl.ugioc26.cn/551995.htm
qdl.ugioc26.cn/353375.htm
qdl.ugioc26.cn/993775.htm
qdl.ugioc26.cn/939335.htm
qdl.ugioc26.cn/377955.htm
qdl.ugioc26.cn/357315.htm
qdl.ugioc26.cn/537975.htm
qdl.ugioc26.cn/559755.htm
qdl.ugioc26.cn/151395.htm
qdk.ugioc26.cn/955735.htm
qdk.ugioc26.cn/337515.htm
qdk.ugioc26.cn/917995.htm
qdk.ugioc26.cn/595575.htm
qdk.ugioc26.cn/995315.htm
qdk.ugioc26.cn/957115.htm
qdk.ugioc26.cn/593735.htm
qdk.ugioc26.cn/931355.htm
qdk.ugioc26.cn/797515.htm
qdk.ugioc26.cn/319595.htm
qdj.ugioc26.cn/991935.htm
qdj.ugioc26.cn/533535.htm
qdj.ugioc26.cn/995935.htm
qdj.ugioc26.cn/339135.htm
qdj.ugioc26.cn/951775.htm
qdj.ugioc26.cn/793115.htm
qdj.ugioc26.cn/95.htm
qdj.ugioc26.cn/955595.htm
qdj.ugioc26.cn/797995.htm
qdj.ugioc26.cn/995535.htm
qdh.ugioc26.cn/979195.htm
qdh.ugioc26.cn/755155.htm
qdh.ugioc26.cn/315575.htm
qdh.ugioc26.cn/711955.htm
qdh.ugioc26.cn/133775.htm
qdh.ugioc26.cn/115355.htm
qdh.ugioc26.cn/599955.htm
qdh.ugioc26.cn/171115.htm
qdh.ugioc26.cn/911175.htm
qdh.ugioc26.cn/315715.htm
qdg.ugioc26.cn/533155.htm
qdg.ugioc26.cn/351335.htm
qdg.ugioc26.cn/315715.htm
qdg.ugioc26.cn/351575.htm
qdg.ugioc26.cn/375915.htm
qdg.ugioc26.cn/171575.htm
qdg.ugioc26.cn/155555.htm
qdg.ugioc26.cn/577775.htm
qdg.ugioc26.cn/973595.htm
qdg.ugioc26.cn/117715.htm
qdf.ugioc26.cn/995155.htm
qdf.ugioc26.cn/999955.htm
qdf.ugioc26.cn/151515.htm
qdf.ugioc26.cn/999335.htm
qdf.ugioc26.cn/775175.htm
qdf.ugioc26.cn/971315.htm
qdf.ugioc26.cn/957915.htm
qdf.ugioc26.cn/115135.htm
qdf.ugioc26.cn/159955.htm
qdf.ugioc26.cn/719555.htm
qdd.ugioc26.cn/773535.htm
qdd.ugioc26.cn/555935.htm
qdd.ugioc26.cn/951955.htm
qdd.ugioc26.cn/191955.htm
qdd.ugioc26.cn/959355.htm
qdd.ugioc26.cn/919355.htm
qdd.ugioc26.cn/373715.htm
qdd.ugioc26.cn/573535.htm
qdd.ugioc26.cn/953595.htm
qdd.ugioc26.cn/331175.htm
qds.ugioc26.cn/719355.htm
qds.ugioc26.cn/915335.htm
