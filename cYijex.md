谜煌惭附以


  
UUID.randomUUID()基于Securerandom从操作系统熵源(如/)生成密码学安全的伪随机数dev/urandom）获取，非真随机但足够独特和不可预测。

UUID.randomUUID() 是真随机还是伪随机？
它用的是 SecureRandom，不是 Math.random() 或 Random。这意味着它将尝试从操作系统收集熵(例如) /dev/urandom），生成的是密码学安全的伪随机数——不是“真正的随机”，但绝大多数业务场景都是唯一和不可预测的。

每次调用 UUID.randomUUID() 都会重新初始化一个 SecureRandom 实例（JDK 17+ 优化为复用，但行为不变)
不依赖系统时间戳，因此没有毫秒并发重复问题
生成结果是 version 4 UUID，格式形如 550e8400-e29b-41d4-a716-446655440000，其中 122 位置由随机数填充


为什么不能直接使用？ toString() 存数据库？
因为默认 toString() 返回带连字符的 36 字符串(如 f47ac10b-58cc-4372-a567-0e02b2c3d479），而且多数据库（MySQL、PostgreSQL）对字符串索引效率敏感，36 字符比 32 有很多字符(去横线) 12.5% 存储费用，并且不能利用前缀索引的优势。

存前推荐用 uuid.toString().replace("-", "") 得到 32 小写16号制字符串
若字段类型为 BINARY(16)（MySQL）或 BYTEA（PostgreSQL），应调用 uuid.getMostSignificantBits() 和 uuid.getLeastSignificantBits() 拆成两个 long，再转为 16 字节数组-比字符串节省 50%+ 存储，索引性能更好
注意：UUID.nameUUIDFromBytes() 是确定性生成(适用于缓存键)，但不是随机型，不要误用于主键

高并发下 UUID.randomUUID() 有性能瓶颈吗？
是的，但这通常不是你应该首先担心的。瓶颈不在 UUID 逻辑本身，而在 SecureRandom 初始化时可能触发的熵池阻塞(特别是在容器或虚拟机中) /dev/random 不足时）。

JDK 8u292+ 和 JDK 17+ 默认启用 NativePRNGNonBlocking，自动 fallback 到 /dev/urandom，基本消除堵塞
若仍观察到 SecureRandom.getInstanceStrong() 时间过长，检查该方法是否被误用（UUID 不用它）
当真正的压力测量时，单机每秒数万次 randomUUID() 调用无压力；若达 10w+/s 而延迟升高，优先排查 GC 或锁定竞争，而不是 UUID 本身

替代方案比较：UUID vs Snowflake vs 数据库自增
选择的关键不是“唯一”，而是「有序性」「可读性」「跨服务协调成本」：
			
		
；


UUID：无中心，全局唯一，自然分布友好；缺点是无序（B+放大树索引)、不可读，占用空间大(16–36 字节）

Snowflake（如 Twitter 的 64 位整型）：时间有序紧凑(8) 字节）、时间戳可以反解；需要部署； ID 生成服务有机器时钟回拨的风险

数据库自增：最简单但强烈依赖单点，需要额外设计(如号段模式)，不适合微服务直接连接

若已使用您的应用程序 Spring Boot + JPA，而且主键字段定义为 @GeneratedValue(generator = "uuuid2"，那 Hibernate 默认情况下使用 UUID.randomUUID() ——此时改用 Snowflake 相反，重写关键策略和迁移历史数据。public class UuidExample {
public static void main(String[] args) {
UUID uuid = UUID.randomUUID();
// 推荐存储形式：32位无横线小写
String storageKey = uuid.toString().replace("-", "");
// 或者转换成16字节数组(适用于BINARY(16)
byte[] bytes = new byte[16];
long msb = uuid.getMostSignificantBits();
long lsb = uuid.getLeastSignificantBits();
for (int i = 0; i >> 8 * (7 - i));
bytes[8 + i] = (byte) (lsb >>> 8 * (7 - i));
}
}
}在实际项目中，UUID 它是否真的“唯一”并不取决于生成函数，而是取决于你如何使用它——例如， UUID.nameUUIDFromBytes("user:123".getBytes()) 当作用户 ID，等于主动放弃随机性，自以为是安全的。	 

肛显融抛焕凭磕职卜酪叫送砍急曝

raa.mmmxz.cn/648803.htm
raa.mmmxz.cn/513913.htm
raa.mmmxz.cn/959133.htm
raa.mmmxz.cn/462443.htm
raa.mmmxz.cn/402603.htm
rap.mmmxz.cn/260623.htm
rap.mmmxz.cn/759373.htm
rap.mmmxz.cn/488063.htm
rap.mmmxz.cn/759773.htm
rap.mmmxz.cn/933913.htm
rap.mmmxz.cn/208883.htm
rap.mmmxz.cn/797193.htm
rap.mmmxz.cn/848283.htm
rap.mmmxz.cn/028463.htm
rap.mmmxz.cn/737333.htm
rao.mmmxz.cn/622063.htm
rao.mmmxz.cn/600063.htm
rao.mmmxz.cn/688063.htm
rao.mmmxz.cn/424623.htm
rao.mmmxz.cn/428063.htm
rao.mmmxz.cn/599353.htm
rao.mmmxz.cn/248643.htm
rao.mmmxz.cn/602283.htm
rao.mmmxz.cn/535553.htm
rao.mmmxz.cn/791133.htm
rai.mmmxz.cn/802443.htm
rai.mmmxz.cn/604403.htm
rai.mmmxz.cn/484243.htm
rai.mmmxz.cn/802823.htm
rai.mmmxz.cn/973533.htm
rai.mmmxz.cn/002203.htm
rai.mmmxz.cn/608203.htm
rai.mmmxz.cn/042443.htm
rai.mmmxz.cn/377513.htm
rai.mmmxz.cn/462623.htm
rau.mmmxz.cn/200463.htm
rau.mmmxz.cn/248443.htm
rau.mmmxz.cn/000043.htm
rau.mmmxz.cn/240683.htm
rau.mmmxz.cn/684403.htm
rau.mmmxz.cn/593513.htm
rau.mmmxz.cn/060203.htm
rau.mmmxz.cn/662863.htm
rau.mmmxz.cn/395953.htm
rau.mmmxz.cn/262643.htm
ray.mmmxz.cn/024463.htm
ray.mmmxz.cn/840443.htm
ray.mmmxz.cn/953933.htm
ray.mmmxz.cn/680223.htm
ray.mmmxz.cn/484083.htm
ray.mmmxz.cn/137353.htm
ray.mmmxz.cn/793313.htm
ray.mmmxz.cn/680843.htm
ray.mmmxz.cn/484283.htm
ray.mmmxz.cn/286243.htm
rat.mmmxz.cn/028023.htm
rat.mmmxz.cn/026443.htm
rat.mmmxz.cn/460803.htm
rat.mmmxz.cn/971913.htm
rat.mmmxz.cn/848043.htm
rat.mmmxz.cn/555313.htm
rat.mmmxz.cn/664423.htm
rat.mmmxz.cn/020803.htm
rat.mmmxz.cn/977973.htm
rat.mmmxz.cn/953753.htm
rar.mmmxz.cn/577753.htm
rar.mmmxz.cn/026603.htm
rar.mmmxz.cn/600223.htm
rar.mmmxz.cn/648803.htm
rar.mmmxz.cn/751733.htm
rar.mmmxz.cn/408843.htm
rar.mmmxz.cn/680063.htm
rar.mmmxz.cn/482403.htm
rar.mmmxz.cn/577993.htm
rar.mmmxz.cn/246063.htm
rae.mmmxz.cn/044663.htm
rae.mmmxz.cn/662063.htm
rae.mmmxz.cn/139913.htm
rae.mmmxz.cn/395313.htm
rae.mmmxz.cn/444643.htm
rae.mmmxz.cn/064843.htm
rae.mmmxz.cn/826283.htm
rae.mmmxz.cn/486483.htm
rae.mmmxz.cn/995353.htm
rae.mmmxz.cn/606823.htm
raw.mmmxz.cn/048663.htm
raw.mmmxz.cn/064623.htm
raw.mmmxz.cn/004803.htm
raw.mmmxz.cn/646403.htm
raw.mmmxz.cn/575393.htm
raw.mmmxz.cn/284843.htm
raw.mmmxz.cn/995913.htm
raw.mmmxz.cn/642803.htm
raw.mmmxz.cn/060023.htm
raw.mmmxz.cn/044223.htm
raq.mmmxz.cn/826623.htm
raq.mmmxz.cn/844063.htm
raq.mmmxz.cn/624843.htm
raq.mmmxz.cn/597533.htm
raq.mmmxz.cn/159393.htm
raq.mmmxz.cn/644883.htm
raq.mmmxz.cn/602463.htm
raq.mmmxz.cn/795573.htm
raq.mmmxz.cn/751113.htm
raq.mmmxz.cn/226663.htm
rpm.mmmxz.cn/717393.htm
rpm.mmmxz.cn/044263.htm
rpm.mmmxz.cn/048463.htm
rpm.mmmxz.cn/026223.htm
rpm.mmmxz.cn/959553.htm
rpm.mmmxz.cn/842423.htm
rpm.mmmxz.cn/917933.htm
rpm.mmmxz.cn/446003.htm
rpm.mmmxz.cn/573553.htm
rpm.mmmxz.cn/939933.htm
rpn.mmmxz.cn/206463.htm
rpn.mmmxz.cn/688063.htm
rpn.mmmxz.cn/482283.htm
rpn.mmmxz.cn/399533.htm
rpn.mmmxz.cn/600403.htm
rpn.mmmxz.cn/088043.htm
rpn.mmmxz.cn/486203.htm
rpn.mmmxz.cn/317533.htm
rpn.mmmxz.cn/117933.htm
rpn.mmmxz.cn/080203.htm
rpb.mmmxz.cn/080803.htm
rpb.mmmxz.cn/806863.htm
rpb.mmmxz.cn/002463.htm
rpb.mmmxz.cn/460003.htm
rpb.mmmxz.cn/622863.htm
rpb.mmmxz.cn/179773.htm
rpb.mmmxz.cn/648603.htm
rpb.mmmxz.cn/579533.htm
rpb.mmmxz.cn/462883.htm
rpb.mmmxz.cn/933353.htm
rpv.mmmxz.cn/571593.htm
rpv.mmmxz.cn/951513.htm
rpv.mmmxz.cn/155533.htm
rpv.mmmxz.cn/086283.htm
rpv.mmmxz.cn/642403.htm
rpv.mmmxz.cn/060443.htm
rpv.mmmxz.cn/486423.htm
rpv.mmmxz.cn/155753.htm
rpv.mmmxz.cn/373173.htm
rpv.mmmxz.cn/048403.htm
rpc.mmmxz.cn/371353.htm
rpc.mmmxz.cn/913553.htm
rpc.mmmxz.cn/715153.htm
rpc.mmmxz.cn/626083.htm
rpc.mmmxz.cn/935793.htm
rpc.mmmxz.cn/262003.htm
rpc.mmmxz.cn/020263.htm
rpc.mmmxz.cn/139393.htm
rpc.mmmxz.cn/024403.htm
rpc.mmmxz.cn/159973.htm
rpx.mmmxz.cn/771113.htm
rpx.mmmxz.cn/795793.htm
rpx.mmmxz.cn/800683.htm
rpx.mmmxz.cn/860803.htm
rpx.mmmxz.cn/199993.htm
rpx.mmmxz.cn/040843.htm
rpx.mmmxz.cn/628623.htm
rpx.mmmxz.cn/197913.htm
rpx.mmmxz.cn/400463.htm
rpx.mmmxz.cn/280043.htm
rpz.mmmxz.cn/442623.htm
rpz.mmmxz.cn/846623.htm
rpz.mmmxz.cn/868463.htm
rpz.mmmxz.cn/068683.htm
rpz.mmmxz.cn/444683.htm
rpz.mmmxz.cn/400063.htm
rpz.mmmxz.cn/866023.htm
rpz.mmmxz.cn/206683.htm
rpz.mmmxz.cn/862063.htm
rpz.mmmxz.cn/262863.htm
rpl.mmmxz.cn/262823.htm
rpl.mmmxz.cn/002423.htm
rpl.mmmxz.cn/266223.htm
rpl.mmmxz.cn/644883.htm
rpl.mmmxz.cn/191133.htm
rpl.mmmxz.cn/199393.htm
rpl.mmmxz.cn/202603.htm
rpl.mmmxz.cn/971573.htm
rpl.mmmxz.cn/339313.htm
rpl.mmmxz.cn/739793.htm
rpk.mmmxz.cn/060083.htm
rpk.mmmxz.cn/866823.htm
rpk.mmmxz.cn/773133.htm
rpk.mmmxz.cn/266443.htm
rpk.mmmxz.cn/513333.htm
rpk.mmmxz.cn/355713.htm
rpk.mmmxz.cn/286603.htm
rpk.mmmxz.cn/860663.htm
rpk.mmmxz.cn/884883.htm
rpk.mmmxz.cn/779393.htm
rpj.mmmxz.cn/240803.htm
rpj.mmmxz.cn/911373.htm
rpj.mmmxz.cn/337333.htm
rpj.mmmxz.cn/648423.htm
rpj.mmmxz.cn/391513.htm
rpj.mmmxz.cn/024883.htm
rpj.mmmxz.cn/440063.htm
rpj.mmmxz.cn/800203.htm
rpj.mmmxz.cn/628883.htm
rpj.mmmxz.cn/973933.htm
rph.mmmxz.cn/088663.htm
rph.mmmxz.cn/315313.htm
rph.mmmxz.cn/404203.htm
rph.mmmxz.cn/315153.htm
rph.mmmxz.cn/597373.htm
rph.mmmxz.cn/424263.htm
rph.mmmxz.cn/082283.htm
rph.mmmxz.cn/711953.htm
rph.mmmxz.cn/597593.htm
rph.mmmxz.cn/286683.htm
rpg.mmmxz.cn/444623.htm
rpg.mmmxz.cn/264083.htm
rpg.mmmxz.cn/622083.htm
rpg.mmmxz.cn/553333.htm
rpg.mmmxz.cn/171173.htm
rpg.mmmxz.cn/339933.htm
rpg.mmmxz.cn/264663.htm
rpg.mmmxz.cn/975133.htm
rpg.mmmxz.cn/359133.htm
rpg.mmmxz.cn/757513.htm
rpf.mmmxz.cn/337713.htm
rpf.mmmxz.cn/446643.htm
rpf.mmmxz.cn/539393.htm
rpf.mmmxz.cn/313513.htm
rpf.mmmxz.cn/917733.htm
rpf.mmmxz.cn/119133.htm
rpf.mmmxz.cn/559553.htm
rpf.mmmxz.cn/048283.htm
rpf.mmmxz.cn/200663.htm
rpf.mmmxz.cn/153973.htm
rpd.mmmxz.cn/608463.htm
rpd.mmmxz.cn/955573.htm
rpd.mmmxz.cn/200063.htm
rpd.mmmxz.cn/339533.htm
rpd.mmmxz.cn/428803.htm
rpd.mmmxz.cn/222823.htm
rpd.mmmxz.cn/600663.htm
rpd.mmmxz.cn/539553.htm
rpd.mmmxz.cn/575793.htm
rpd.mmmxz.cn/620803.htm
rps.mmmxz.cn/551733.htm
rps.mmmxz.cn/446063.htm
rps.mmmxz.cn/044803.htm
rps.mmmxz.cn/337733.htm
rps.mmmxz.cn/666403.htm
rps.mmmxz.cn/084263.htm
rps.mmmxz.cn/715773.htm
rps.mmmxz.cn/460023.htm
rps.mmmxz.cn/886043.htm
rps.mmmxz.cn/737353.htm
rpa.mmmxz.cn/199533.htm
rpa.mmmxz.cn/280403.htm
rpa.mmmxz.cn/084203.htm
rpa.mmmxz.cn/284063.htm
rpa.mmmxz.cn/731353.htm
rpa.mmmxz.cn/402443.htm
rpa.mmmxz.cn/177513.htm
rpa.mmmxz.cn/191193.htm
rpa.mmmxz.cn/135993.htm
rpa.mmmxz.cn/268483.htm
rpp.mmmxz.cn/668803.htm
rpp.mmmxz.cn/133593.htm
rpp.mmmxz.cn/826863.htm
rpp.mmmxz.cn/753913.htm
rpp.mmmxz.cn/331373.htm
rpp.mmmxz.cn/842423.htm
rpp.mmmxz.cn/539593.htm
rpp.mmmxz.cn/804023.htm
rpp.mmmxz.cn/131353.htm
rpp.mmmxz.cn/755393.htm
rpo.mmmxz.cn/353713.htm
rpo.mmmxz.cn/468083.htm
rpo.mmmxz.cn/060043.htm
rpo.mmmxz.cn/577333.htm
rpo.mmmxz.cn/060463.htm
rpo.mmmxz.cn/866263.htm
rpo.mmmxz.cn/206423.htm
rpo.mmmxz.cn/240443.htm
rpo.mmmxz.cn/971373.htm
rpo.mmmxz.cn/731593.htm
rpi.mmmxz.cn/048623.htm
rpi.mmmxz.cn/177553.htm
rpi.mmmxz.cn/793533.htm
rpi.mmmxz.cn/517953.htm
rpi.mmmxz.cn/751153.htm
rpi.mmmxz.cn/997193.htm
rpi.mmmxz.cn/268003.htm
rpi.mmmxz.cn/066283.htm
rpi.mmmxz.cn/579113.htm
rpi.mmmxz.cn/402823.htm
rpu.mmmxz.cn/151333.htm
rpu.mmmxz.cn/391153.htm
rpu.mmmxz.cn/957733.htm
rpu.mmmxz.cn/008843.htm
rpu.mmmxz.cn/199393.htm
rpu.mmmxz.cn/313713.htm
rpu.mmmxz.cn/606443.htm
rpu.mmmxz.cn/533753.htm
rpu.mmmxz.cn/462623.htm
rpu.mmmxz.cn/791993.htm
rpy.mmmxz.cn/646803.htm
rpy.mmmxz.cn/060883.htm
rpy.mmmxz.cn/428023.htm
rpy.mmmxz.cn/884403.htm
rpy.mmmxz.cn/735353.htm
rpy.mmmxz.cn/991733.htm
rpy.mmmxz.cn/220283.htm
rpy.mmmxz.cn/208823.htm
rpy.mmmxz.cn/468243.htm
rpy.mmmxz.cn/997153.htm
rpt.mmmxz.cn/975513.htm
rpt.mmmxz.cn/917973.htm
rpt.mmmxz.cn/313533.htm
rpt.mmmxz.cn/646863.htm
rpt.mmmxz.cn/646643.htm
rpt.mmmxz.cn/604403.htm
rpt.mmmxz.cn/717333.htm
rpt.mmmxz.cn/755993.htm
rpt.mmmxz.cn/571373.htm
rpt.mmmxz.cn/628023.htm
rpr.mmmxz.cn/288063.htm
rpr.mmmxz.cn/717753.htm
rpr.mmmxz.cn/775533.htm
rpr.mmmxz.cn/773573.htm
rpr.mmmxz.cn/022083.htm
rpr.mmmxz.cn/246643.htm
rpr.mmmxz.cn/153373.htm
rpr.mmmxz.cn/802443.htm
rpr.mmmxz.cn/515153.htm
rpr.mmmxz.cn/884003.htm
rpe.mmmxz.cn/311393.htm
rpe.mmmxz.cn/379313.htm
rpe.mmmxz.cn/620243.htm
rpe.mmmxz.cn/959933.htm
rpe.mmmxz.cn/622223.htm
rpe.mmmxz.cn/240223.htm
rpe.mmmxz.cn/826063.htm
rpe.mmmxz.cn/579353.htm
rpe.mmmxz.cn/351513.htm
rpe.mmmxz.cn/339793.htm
rpw.mmmxz.cn/462663.htm
rpw.mmmxz.cn/959353.htm
rpw.mmmxz.cn/642823.htm
rpw.mmmxz.cn/959373.htm
rpw.mmmxz.cn/228823.htm
rpw.mmmxz.cn/971913.htm
rpw.mmmxz.cn/822243.htm
rpw.mmmxz.cn/575993.htm
rpw.mmmxz.cn/468203.htm
rpw.mmmxz.cn/666263.htm
rpq.mmmxz.cn/191953.htm
rpq.mmmxz.cn/537153.htm
rpq.mmmxz.cn/955333.htm
rpq.mmmxz.cn/680663.htm
rpq.mmmxz.cn/808663.htm
rpq.mmmxz.cn/608403.htm
rpq.mmmxz.cn/808623.htm
rpq.mmmxz.cn/739133.htm
rpq.mmmxz.cn/226463.htm
rpq.mmmxz.cn/931753.htm
rom.mmmxz.cn/515753.htm
rom.mmmxz.cn/282403.htm
rom.mmmxz.cn/139573.htm
rom.mmmxz.cn/157353.htm
rom.mmmxz.cn/662823.htm
rom.mmmxz.cn/131553.htm
rom.mmmxz.cn/113593.htm
rom.mmmxz.cn/733173.htm
rom.mmmxz.cn/731373.htm
rom.mmmxz.cn/375173.htm
ron.mmmxz.cn/755373.htm
ron.mmmxz.cn/151933.htm
ron.mmmxz.cn/519113.htm
ron.mmmxz.cn/931773.htm
ron.mmmxz.cn/802243.htm
ron.mmmxz.cn/884483.htm
ron.mmmxz.cn/228223.htm
ron.mmmxz.cn/515513.htm
ron.mmmxz.cn/797133.htm
ron.mmmxz.cn/844043.htm
rob.mmmxz.cn/084663.htm
rob.mmmxz.cn/222803.htm
rob.mmmxz.cn/448003.htm
rob.mmmxz.cn/460263.htm
rob.mmmxz.cn/464063.htm
rob.mmmxz.cn/628083.htm
rob.mmmxz.cn/868203.htm
rob.mmmxz.cn/440823.htm
rob.mmmxz.cn/408843.htm
rob.mmmxz.cn/000003.htm
