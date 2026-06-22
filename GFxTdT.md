举沿盘寄澜


  
MySQL连接需要配置servertimezonernenenenenener=UTC，Jdbctemplate的queryForobject要求有且只有一行，否则会抛出异常，驱动类名必须是com.mysql.cj.jdbc.Driver，需要添加@Transactional自动回滚进行测试。

application.yml 里 MySQL 连接配置写错了，spring.datasource.url 最容易漏掉 serverTimezone=UTC

Spring Boot 启动时无法连接 MySQL，十次有八次 url 时区参数缺失。MySQL 8+ 默认要求显式指定时区，否则抛出 java.sql.SQLException: The server time zone value '...' is unrecognized。
必须带上正确的写作方法 serverTimezone，且推荐用 UTC(避免夏令期间当地时区的干扰)：spring:
  datasource:
url: jdbc:mysql://localhost:3306/mydb?useSSL=false&amp;serverTimezone=UTC&amp;allowPublicKeyRetrieval=true
username: root
password: 123456
driver-class-name: com.mysql.cj.jdbc.Driver

useSSL=false：开发环境可关，生产必须配置 TLS

allowPublicKeyRetrieval=true：MySQL 8.0.13+ 密码认证方式变更后必须进行密码认证

driver-class-name 必须明确声明，Spring Boot 2.4+ 不再自动推断
注意 &amp; 是 YAML 使用中间的特殊字符 &amp; 转义，不是 &amp;


JdbcTemplate 查询空结果不报错，但查询空结果不报错 queryForObject 如果找不到数据，会直接扔掉 EmptyResultDataAccessException

这是最常踩的坑:以为是最常踩的坑: queryForObject 返回 null，事实上，它严格要求“只有一行”，没有数据或多行炸。
常见场景：根据用户登录状态检查用户登录状态 ID 取单个记录。不要硬扛异常，该用就用。 query + isEmpty() 或 queryOptional：
；

List&lt;Map&lt;String, Object&gt;&gt; result = jdbcTemplate.query(&quot;SELECT * FROM user WHERE id = ?&quot;, 
new Object[]{1L}， 
new ColumnRowMapper());
用 queryForObject 确认业务逻辑是否真的能保证“必须”
想安全取单条吗？改用 jdbcTemplate.query(&quot;...&quot;, args, rs -&gt; rs.next() ? mapRow(rs, 0) : null)


queryForList 和 query 回到空集合，不会抛出异常

MySQL 驱动版本和 Spring Boot 版本不匹配，ClassNotFoundException: com.mysql.jdbc.Driver 或连接卡死
Spring Boot 2.4+ 默认使用 MySQL Connector/J 8.x，旧版驱动类名从 com.mysql.jdbc.Driver 改成 com.mysql.cj.jdbc.Driver。如果还写老类名，启动时报告 ClassNotFoundException；若依赖不对齐，则可卡在连接池中初始化。

检查 pom.xml 是否引入了 mysql:mysql-connector-java，且版本 ≥ 8.0.28
确认 application.yml 中 driver-class-name 是 com.mysql.cj.jdbc.Driver

若用 Gradle，避免同时引入 mysql-connector-java 和 mysql-connector-j —— 后者是新包名，前者已被废弃
IDEA 里右键 Maven → Reload 后，点开 External Libraries 看实际加载的是哪一个 JAR

测试 JdbcTemplate 没有清空表或事务没有回滚，testInsertThenQuery 第二次跑就失败了
默认情况下，如果单元测试不打开事务，插入的数据将真实存储。由于主要冲突或条件重复，下次运行同一测试可能会失败。
解决方法很简单:加 @Transactional 注意，让测试方法在事务中运行，方法自动结束 rollback：@SpringBootTest
@Transactional
class UserDaoTest {
@Test
void testInsertThenQuery() {
jdbcTemplate.update(&quot;INSERT INTO user(name) VALUES(?)&quot;, &quot;test&quot;);
Integer count = jdbcTemplate.queryForObject(&quot;SELECT COUNT(*) FROM user WHERE name = ?&quot;, 
Integer.class, &quot;test&quot;);
assertThat(count).isEqualTo(1);
}
}
不需要手动 @Rollback，@Transactional 默认是回滚
不要在测试中使用 @Sql 加载 SQL 手动完成文件 insert —— 时间顺序容易混乱
如果测试需要验证“事务真的没有提交”，可以开始另一个非事务线程查库，但在大多数情况下没有必要

真正的麻烦是嵌套事务、异步调用或使用 @Modifying 的自定义 Repository 方法-那些地方的事务边界很容易被忽视，这取决于代理链和传播行为的实际执行。	 

谀忧咆源沂盅呵掳檀吞梦沟磁渴镜

sma.vwbnt.cn/868064.Doc
sma.vwbnt.cn/282008.Doc
sma.vwbnt.cn/464042.Doc
sma.vwbnt.cn/402644.Doc
sma.vwbnt.cn/486604.Doc
sma.vwbnt.cn/311997.Doc
sma.vwbnt.cn/228682.Doc
sma.vwbnt.cn/044002.Doc
smp.vwbnt.cn/206446.Doc
smp.vwbnt.cn/202602.Doc
smp.vwbnt.cn/040288.Doc
smp.vwbnt.cn/224640.Doc
smp.vwbnt.cn/400266.Doc
smp.vwbnt.cn/488606.Doc
smp.vwbnt.cn/208260.Doc
smp.vwbnt.cn/808000.Doc
smp.vwbnt.cn/420080.Doc
smp.vwbnt.cn/244284.Doc
smo.vwbnt.cn/426240.Doc
smo.vwbnt.cn/559793.Doc
smo.vwbnt.cn/406248.Doc
smo.vwbnt.cn/115917.Doc
smo.vwbnt.cn/800662.Doc
smo.vwbnt.cn/048860.Doc
smo.vwbnt.cn/208462.Doc
smo.vwbnt.cn/662862.Doc
smo.vwbnt.cn/044068.Doc
smo.vwbnt.cn/422046.Doc
smi.vwbnt.cn/062264.Doc
smi.vwbnt.cn/668604.Doc
smi.vwbnt.cn/462664.Doc
smi.vwbnt.cn/480420.Doc
smi.vwbnt.cn/844848.Doc
smi.vwbnt.cn/228062.Doc
smi.vwbnt.cn/420820.Doc
smi.vwbnt.cn/200444.Doc
smi.vwbnt.cn/484448.Doc
smi.vwbnt.cn/864426.Doc
smu.vwbnt.cn/620842.Doc
smu.vwbnt.cn/066462.Doc
smu.vwbnt.cn/868220.Doc
smu.vwbnt.cn/282666.Doc
smu.vwbnt.cn/440686.Doc
smu.vwbnt.cn/220864.Doc
smu.vwbnt.cn/242866.Doc
smu.vwbnt.cn/804446.Doc
smu.vwbnt.cn/280044.Doc
smu.vwbnt.cn/824448.Doc
smy.vwbnt.cn/240668.Doc
smy.vwbnt.cn/488886.Doc
smy.vwbnt.cn/420242.Doc
smy.vwbnt.cn/208808.Doc
smy.vwbnt.cn/240806.Doc
smy.vwbnt.cn/626044.Doc
smy.vwbnt.cn/026488.Doc
smy.vwbnt.cn/662684.Doc
smy.vwbnt.cn/864248.Doc
smy.vwbnt.cn/440482.Doc
smt.vwbnt.cn/006688.Doc
smt.vwbnt.cn/642440.Doc
smt.vwbnt.cn/406426.Doc
smt.vwbnt.cn/644860.Doc
smt.vwbnt.cn/242682.Doc
smt.vwbnt.cn/666044.Doc
smt.vwbnt.cn/686662.Doc
smt.vwbnt.cn/826486.Doc
smt.vwbnt.cn/400208.Doc
smt.vwbnt.cn/282848.Doc
smr.vwbnt.cn/448226.Doc
smr.vwbnt.cn/648066.Doc
smr.vwbnt.cn/460682.Doc
smr.vwbnt.cn/440284.Doc
smr.vwbnt.cn/224446.Doc
smr.vwbnt.cn/220684.Doc
smr.vwbnt.cn/808844.Doc
smr.vwbnt.cn/022884.Doc
smr.vwbnt.cn/488006.Doc
smr.vwbnt.cn/084660.Doc
sme.vwbnt.cn/084484.Doc
sme.vwbnt.cn/802248.Doc
sme.vwbnt.cn/004042.Doc
sme.vwbnt.cn/288200.Doc
sme.vwbnt.cn/068462.Doc
sme.vwbnt.cn/260806.Doc
sme.vwbnt.cn/622202.Doc
sme.vwbnt.cn/842806.Doc
sme.vwbnt.cn/002242.Doc
sme.vwbnt.cn/884220.Doc
smw.vwbnt.cn/862800.Doc
smw.vwbnt.cn/600604.Doc
smw.vwbnt.cn/246662.Doc
smw.vwbnt.cn/664208.Doc
smw.vwbnt.cn/088802.Doc
smw.vwbnt.cn/262080.Doc
smw.vwbnt.cn/806288.Doc
smw.vwbnt.cn/844622.Doc
smw.vwbnt.cn/066022.Doc
smw.vwbnt.cn/195571.Doc
smq.vwbnt.cn/468406.Doc
smq.vwbnt.cn/288084.Doc
smq.vwbnt.cn/020622.Doc
smq.vwbnt.cn/402666.Doc
smq.vwbnt.cn/048466.Doc
smq.vwbnt.cn/402244.Doc
smq.vwbnt.cn/828246.Doc
smq.vwbnt.cn/024482.Doc
smq.vwbnt.cn/484262.Doc
smq.vwbnt.cn/206644.Doc
snm.vwbnt.cn/048608.Doc
snm.vwbnt.cn/440268.Doc
snm.vwbnt.cn/688600.Doc
snm.vwbnt.cn/608644.Doc
snm.vwbnt.cn/448284.Doc
snm.vwbnt.cn/288860.Doc
snm.vwbnt.cn/826042.Doc
snm.vwbnt.cn/020088.Doc
snm.vwbnt.cn/480886.Doc
snm.vwbnt.cn/400002.Doc
snn.vwbnt.cn/228606.Doc
snn.vwbnt.cn/442606.Doc
snn.vwbnt.cn/482286.Doc
snn.vwbnt.cn/820046.Doc
snn.vwbnt.cn/446006.Doc
snn.vwbnt.cn/808660.Doc
snn.vwbnt.cn/600602.Doc
snn.vwbnt.cn/260206.Doc
snn.vwbnt.cn/804006.Doc
snn.vwbnt.cn/048802.Doc
snb.vwbnt.cn/840086.Doc
snb.vwbnt.cn/266262.Doc
snb.vwbnt.cn/868822.Doc
snb.vwbnt.cn/024020.Doc
snb.vwbnt.cn/626802.Doc
snb.vwbnt.cn/282680.Doc
snb.vwbnt.cn/802426.Doc
snb.vwbnt.cn/606088.Doc
snb.vwbnt.cn/680042.Doc
snb.vwbnt.cn/048446.Doc
snv.vwbnt.cn/204864.Doc
snv.vwbnt.cn/802224.Doc
snv.vwbnt.cn/480002.Doc
snv.vwbnt.cn/911777.Doc
snv.vwbnt.cn/046240.Doc
snv.vwbnt.cn/246822.Doc
snv.vwbnt.cn/686222.Doc
snv.vwbnt.cn/028206.Doc
snv.vwbnt.cn/066802.Doc
snv.vwbnt.cn/864462.Doc
snc.vwbnt.cn/480244.Doc
snc.vwbnt.cn/846842.Doc
snc.vwbnt.cn/624062.Doc
snc.vwbnt.cn/044268.Doc
snc.vwbnt.cn/088400.Doc
snc.vwbnt.cn/626866.Doc
snc.vwbnt.cn/539737.Doc
snc.vwbnt.cn/246042.Doc
snc.vwbnt.cn/208864.Doc
snc.vwbnt.cn/028404.Doc
snx.vwbnt.cn/404088.Doc
snx.vwbnt.cn/880826.Doc
snx.vwbnt.cn/808064.Doc
snx.vwbnt.cn/626662.Doc
snx.vwbnt.cn/660880.Doc
snx.vwbnt.cn/599753.Doc
snx.vwbnt.cn/842808.Doc
snx.vwbnt.cn/046260.Doc
snx.vwbnt.cn/046264.Doc
snx.vwbnt.cn/046248.Doc
snz.vwbnt.cn/868860.Doc
snz.vwbnt.cn/426400.Doc
snz.vwbnt.cn/082662.Doc
snz.vwbnt.cn/068002.Doc
snz.vwbnt.cn/828204.Doc
snz.vwbnt.cn/202660.Doc
snz.vwbnt.cn/626028.Doc
snz.vwbnt.cn/060866.Doc
snz.vwbnt.cn/688040.Doc
snz.vwbnt.cn/480488.Doc
snl.vwbnt.cn/666042.Doc
snl.vwbnt.cn/046426.Doc
snl.vwbnt.cn/202246.Doc
snl.vwbnt.cn/648224.Doc
snl.vwbnt.cn/400808.Doc
snl.vwbnt.cn/680606.Doc
snl.vwbnt.cn/608262.Doc
snl.vwbnt.cn/024626.Doc
snl.vwbnt.cn/422628.Doc
snl.vwbnt.cn/404466.Doc
snk.vwbnt.cn/020886.Doc
snk.vwbnt.cn/462446.Doc
snk.vwbnt.cn/640000.Doc
snk.vwbnt.cn/242466.Doc
snk.vwbnt.cn/804684.Doc
snk.vwbnt.cn/446028.Doc
snk.vwbnt.cn/662282.Doc
snk.vwbnt.cn/080642.Doc
snk.vwbnt.cn/062842.Doc
snk.vwbnt.cn/864682.Doc
snj.vwbnt.cn/028882.Doc
snj.vwbnt.cn/440068.Doc
snj.vwbnt.cn/042286.Doc
snj.vwbnt.cn/088648.Doc
snj.vwbnt.cn/866246.Doc
snj.vwbnt.cn/288444.Doc
snj.vwbnt.cn/468620.Doc
snj.vwbnt.cn/600464.Doc
snj.vwbnt.cn/082888.Doc
snj.vwbnt.cn/260804.Doc
snh.vwbnt.cn/020606.Doc
snh.vwbnt.cn/406204.Doc
snh.vwbnt.cn/860020.Doc
snh.vwbnt.cn/664806.Doc
snh.vwbnt.cn/484242.Doc
snh.vwbnt.cn/282646.Doc
snh.vwbnt.cn/688064.Doc
snh.vwbnt.cn/688880.Doc
snh.vwbnt.cn/262440.Doc
snh.vwbnt.cn/682684.Doc
sng.vwbnt.cn/486866.Doc
sng.vwbnt.cn/446806.Doc
sng.vwbnt.cn/426280.Doc
sng.vwbnt.cn/880086.Doc
sng.vwbnt.cn/880842.Doc
sng.vwbnt.cn/022040.Doc
sng.vwbnt.cn/864840.Doc
sng.vwbnt.cn/357357.Doc
sng.vwbnt.cn/020684.Doc
sng.vwbnt.cn/824082.Doc
snf.vwbnt.cn/808466.Doc
snf.vwbnt.cn/604402.Doc
snf.vwbnt.cn/806842.Doc
snf.vwbnt.cn/624822.Doc
snf.vwbnt.cn/260602.Doc
snf.vwbnt.cn/848826.Doc
snf.vwbnt.cn/151313.Doc
snf.vwbnt.cn/406082.Doc
snf.vwbnt.cn/040646.Doc
snf.vwbnt.cn/826462.Doc
snd.vwbnt.cn/422224.Doc
snd.vwbnt.cn/444420.Doc
snd.vwbnt.cn/008406.Doc
snd.vwbnt.cn/931377.Doc
snd.vwbnt.cn/824268.Doc
snd.vwbnt.cn/268204.Doc
snd.vwbnt.cn/622288.Doc
snd.vwbnt.cn/428220.Doc
snd.vwbnt.cn/426202.Doc
snd.vwbnt.cn/022002.Doc
sns.vwbnt.cn/806668.Doc
sns.vwbnt.cn/406248.Doc
sns.vwbnt.cn/202460.Doc
sns.vwbnt.cn/888284.Doc
sns.vwbnt.cn/284268.Doc
sns.vwbnt.cn/604248.Doc
sns.vwbnt.cn/082084.Doc
sns.vwbnt.cn/642644.Doc
sns.vwbnt.cn/828200.Doc
sns.vwbnt.cn/462862.Doc
sna.vwbnt.cn/482424.Doc
sna.vwbnt.cn/022282.Doc
sna.vwbnt.cn/262826.Doc
sna.vwbnt.cn/460480.Doc
sna.vwbnt.cn/066466.Doc
sna.vwbnt.cn/200066.Doc
sna.vwbnt.cn/686282.Doc
sna.vwbnt.cn/626022.Doc
sna.vwbnt.cn/242604.Doc
sna.vwbnt.cn/428200.Doc
snp.vwbnt.cn/648800.Doc
snp.vwbnt.cn/068004.Doc
snp.vwbnt.cn/664686.Doc
snp.vwbnt.cn/591595.Doc
snp.vwbnt.cn/668420.Doc
snp.vwbnt.cn/848024.Doc
snp.vwbnt.cn/608426.Doc
snp.vwbnt.cn/822004.Doc
snp.vwbnt.cn/977593.Doc
snp.vwbnt.cn/246088.Doc
sno.vwbnt.cn/262668.Doc
sno.vwbnt.cn/608446.Doc
sno.vwbnt.cn/242044.Doc
sno.vwbnt.cn/866046.Doc
sno.vwbnt.cn/486280.Doc
sno.vwbnt.cn/424602.Doc
sno.vwbnt.cn/222426.Doc
sno.vwbnt.cn/735337.Doc
sno.vwbnt.cn/644068.Doc
sno.vwbnt.cn/044020.Doc
sni.vwbnt.cn/480226.Doc
sni.vwbnt.cn/604606.Doc
sni.vwbnt.cn/828802.Doc
sni.vwbnt.cn/513755.Doc
sni.vwbnt.cn/284244.Doc
sni.vwbnt.cn/224824.Doc
sni.vwbnt.cn/082020.Doc
sni.vwbnt.cn/444866.Doc
sni.vwbnt.cn/400866.Doc
sni.vwbnt.cn/224688.Doc
snu.vwbnt.cn/864424.Doc
snu.vwbnt.cn/242282.Doc
snu.vwbnt.cn/626000.Doc
snu.vwbnt.cn/313559.Doc
snu.vwbnt.cn/828446.Doc
snu.vwbnt.cn/800626.Doc
snu.vwbnt.cn/686868.Doc
snu.vwbnt.cn/264444.Doc
snu.vwbnt.cn/446462.Doc
snu.vwbnt.cn/642280.Doc
sny.nfsid.cn/206026.Doc
sny.nfsid.cn/264082.Doc
sny.nfsid.cn/462046.Doc
sny.nfsid.cn/640422.Doc
sny.nfsid.cn/804424.Doc
sny.nfsid.cn/844486.Doc
sny.nfsid.cn/680222.Doc
sny.nfsid.cn/062424.Doc
sny.nfsid.cn/666468.Doc
sny.nfsid.cn/080448.Doc
snt.nfsid.cn/800262.Doc
snt.nfsid.cn/333337.Doc
snt.nfsid.cn/448848.Doc
snt.nfsid.cn/628604.Doc
snt.nfsid.cn/002804.Doc
snt.nfsid.cn/264460.Doc
snt.nfsid.cn/448284.Doc
snt.nfsid.cn/040402.Doc
snt.nfsid.cn/220882.Doc
snt.nfsid.cn/660082.Doc
snr.nfsid.cn/246884.Doc
snr.nfsid.cn/640060.Doc
snr.nfsid.cn/480006.Doc
snr.nfsid.cn/086002.Doc
snr.nfsid.cn/886660.Doc
snr.nfsid.cn/664806.Doc
snr.nfsid.cn/882068.Doc
snr.nfsid.cn/682204.Doc
snr.nfsid.cn/288248.Doc
snr.nfsid.cn/951513.Doc
sne.nfsid.cn/064028.Doc
sne.nfsid.cn/400848.Doc
sne.nfsid.cn/082882.Doc
sne.nfsid.cn/135355.Doc
sne.nfsid.cn/040808.Doc
sne.nfsid.cn/428042.Doc
sne.nfsid.cn/440068.Doc
sne.nfsid.cn/028668.Doc
sne.nfsid.cn/042626.Doc
sne.nfsid.cn/135773.Doc
snw.nfsid.cn/284264.Doc
snw.nfsid.cn/448846.Doc
snw.nfsid.cn/208048.Doc
snw.nfsid.cn/220440.Doc
snw.nfsid.cn/462842.Doc
snw.nfsid.cn/684068.Doc
snw.nfsid.cn/622046.Doc
snw.nfsid.cn/226646.Doc
snw.nfsid.cn/462080.Doc
snw.nfsid.cn/248624.Doc
snq.nfsid.cn/822464.Doc
snq.nfsid.cn/244840.Doc
snq.nfsid.cn/622202.Doc
snq.nfsid.cn/626884.Doc
snq.nfsid.cn/884222.Doc
snq.nfsid.cn/444080.Doc
snq.nfsid.cn/064082.Doc
snq.nfsid.cn/042640.Doc
snq.nfsid.cn/282888.Doc
snq.nfsid.cn/680640.Doc
sbm.nfsid.cn/248466.Doc
sbm.nfsid.cn/248462.Doc
sbm.nfsid.cn/888846.Doc
sbm.nfsid.cn/064606.Doc
sbm.nfsid.cn/208044.Doc
sbm.nfsid.cn/428000.Doc
sbm.nfsid.cn/806280.Doc
sbm.nfsid.cn/602404.Doc
sbm.nfsid.cn/668844.Doc
sbm.nfsid.cn/973351.Doc
sbn.nfsid.cn/664624.Doc
sbn.nfsid.cn/866260.Doc
sbn.nfsid.cn/420426.Doc
sbn.nfsid.cn/422480.Doc
sbn.nfsid.cn/660800.Doc
sbn.nfsid.cn/771731.Doc
sbn.nfsid.cn/064806.Doc
sbn.nfsid.cn/620802.Doc
sbn.nfsid.cn/848282.Doc
sbn.nfsid.cn/446888.Doc
sbb.nfsid.cn/648800.Doc
sbb.nfsid.cn/426280.Doc
sbb.nfsid.cn/086866.Doc
sbb.nfsid.cn/428488.Doc
sbb.nfsid.cn/068244.Doc
sbb.nfsid.cn/046848.Doc
sbb.nfsid.cn/084868.Doc
