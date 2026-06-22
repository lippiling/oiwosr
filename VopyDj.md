冻嫉量踪迫


  
try-with-resources 这是一种需要资源实现的强制性安全机制 AutoCloseable 接口；在标准库中 InputStream/OutputStream、Reader/Writer、JDBC 4.0+ 接口、Socket、Zip/GZIP 流、NIO Channel 等均支持；File/Path 不支持本身，需要使用 Files 返回工具类资源；关闭顺序为声明逆序；close() 抛异常会被抑制并附加到主异常中，需要显式处理。

try-with-resources 不是“可选优化”，而是强制性的安全机制——只要资源用完，就必须明确调用 close()，否则会泄露(如文件句柄、数据库连接、网络套接字)，应使用。

哪些类/界面明确支持 try-with-resources？
只有一个核心判断标准:是否实现 AutoCloseable 接口(或其子接口) Closeable）。Java 以下资源类型均符合标准库：


InputStream / OutputStream 及所有子类（FileInputStream、BufferedInputStream、ObjectInputStream 等）

Reader / Writer 及所有子类（FileReader、BufferedWriter、PrintWriter）

java.sql.Connection、PreparedStatement、ResultSet（JDBC 4.0+ 驱动才实现 AutoCloseable）

java.net.Socket、ServerSocket、HttpURLConnection


java.util.zip.ZipInputStream、GZIPInputStream、DeflaterOutputStream


java.nio.channels.Channel 相关实现（FileChannel、SocketChannel）

⚠️ 注意：File 类本身没有实现 AutoCloseable，不能直接放入 try (File f = ...)；Path 和 Files 工具类方法(如 Files.newInputStream()）资源可以关闭。


为什么有些 close() 该方法不能自动触发？
常见误判场景：看类有类 close() 方法，认为可以使用 try-with-resources。但 Java 不看方法名，只认接口合同。
；
			
		

手动写了 public void close() { ... }，但没声明 implements AutoCloseable → 编译报错：cannot be auto-closed; it does not implement AutoCloseable

继承了 FilterInputStream，误以为父亲已经实现了 AutoCloseable → 实际上 FilterInputStream 它本身没有实现，只有 FileInputStream 等具体的子类才能实现
用了老版本 JDBC 驱动（如 MySQL Connector/J 5.1 之前），Connection 没实现 AutoCloseable → 升级驱动或手动包装
第三方库中的流/连接类(如某些) HTTP 客户端响应体)，需要检查 Javadoc 或者源码是否确认该接口是否实现


当多个资源一起使用时，如何确定关闭顺序？
资源按 声明顺序的逆序 关闭:最后写的第一关，最后写的第一关。这对有依赖关系的资源至关重要。try (Connection conn = DriverManager.getConnection(url);
 PreparedStatement stmt = conn.prepareStatement(sql);
 ResultSet rs = stmt.executeQuery()) {
// 处理结果
} // ← 关闭顺序：rs → stmt → conn假如写反了(比如把 conn 放在最后)，stmt.close() 可能在 conn 关闭后执行，部分 JDBC 驱动会抛出 SQLException 或者沉默失败。

多资源用分号 ; 分隔不能用逗号或末尾加分号
Java 9+ 支持现有变量的复用：FileInputStream fis = new FileInputStream("a.txt"); try (fis) { ... }，但是变量不能是 final 且不能在 try 块内重新赋值


异常被抑制（suppressed）为什么不失去关键错误呢？
当 try 块抛出异常，某种资源 close() 当也抛出异常时，后者不会覆盖主异常，而是被“抑制”并附加到主异常上。但默认情况下，如果不打印，很容易被误判关闭成功。try (FileInputStream fis = new FileInputStream("missing.txt")) {
throw new RuntimeException("read failed");
} catch (RuntimeException e) {
for (Throwable s : e.getSuppressed()) {
System.err.println("Suppressed: " + s);
}
}日志框架(如 Log4j、SLF4J）通常不输出 suppressed 除非显式配置或调用异常 getSuppressed()。漏掉这一步，相当于关上门修水管——漏水还不知道哪一段裂开。	 

岗圆厩永芬仝撇戮泼屎事榔岛贫埔

dbz.nfsid.cn/644260.Doc
dbl.nfsid.cn/086246.Doc
dbl.nfsid.cn/406428.Doc
dbl.nfsid.cn/262408.Doc
dbl.nfsid.cn/880080.Doc
dbl.nfsid.cn/862642.Doc
dbl.nfsid.cn/571937.Doc
dbl.nfsid.cn/133395.Doc
dbl.nfsid.cn/064066.Doc
dbl.nfsid.cn/028806.Doc
dbl.nfsid.cn/640266.Doc
dbk.nfsid.cn/668022.Doc
dbk.nfsid.cn/802042.Doc
dbk.nfsid.cn/844082.Doc
dbk.nfsid.cn/244080.Doc
dbk.nfsid.cn/844646.Doc
dbk.nfsid.cn/400680.Doc
dbk.nfsid.cn/353359.Doc
dbk.nfsid.cn/280284.Doc
dbk.nfsid.cn/046806.Doc
dbk.nfsid.cn/426824.Doc
dbj.nfsid.cn/395951.Doc
dbj.nfsid.cn/448622.Doc
dbj.nfsid.cn/064482.Doc
dbj.nfsid.cn/331795.Doc
dbj.nfsid.cn/480008.Doc
dbj.nfsid.cn/484666.Doc
dbj.nfsid.cn/171335.Doc
dbj.nfsid.cn/244642.Doc
dbj.nfsid.cn/515717.Doc
dbj.nfsid.cn/462208.Doc
dbh.nfsid.cn/248260.Doc
dbh.nfsid.cn/280462.Doc
dbh.nfsid.cn/246404.Doc
dbh.nfsid.cn/220608.Doc
dbh.nfsid.cn/535931.Doc
dbh.nfsid.cn/442024.Doc
dbh.nfsid.cn/311335.Doc
dbh.nfsid.cn/428262.Doc
dbh.nfsid.cn/802028.Doc
dbh.nfsid.cn/088020.Doc
dbg.nfsid.cn/224640.Doc
dbg.nfsid.cn/608446.Doc
dbg.nfsid.cn/553773.Doc
dbg.nfsid.cn/822060.Doc
dbg.nfsid.cn/202442.Doc
dbg.nfsid.cn/244406.Doc
dbg.nfsid.cn/626400.Doc
dbg.nfsid.cn/648824.Doc
dbg.nfsid.cn/886208.Doc
dbg.nfsid.cn/622402.Doc
dbf.nfsid.cn/600846.Doc
dbf.nfsid.cn/113791.Doc
dbf.nfsid.cn/826648.Doc
dbf.nfsid.cn/062642.Doc
dbf.nfsid.cn/646402.Doc
dbf.nfsid.cn/446446.Doc
dbf.nfsid.cn/284642.Doc
dbf.nfsid.cn/606662.Doc
dbf.nfsid.cn/355179.Doc
dbf.nfsid.cn/064062.Doc
dbd.nfsid.cn/662286.Doc
dbd.nfsid.cn/597315.Doc
dbd.nfsid.cn/280460.Doc
dbd.nfsid.cn/640646.Doc
dbd.nfsid.cn/991333.Doc
dbd.nfsid.cn/797151.Doc
dbd.nfsid.cn/064502.Doc
dbd.nfsid.cn/366828.Doc
dbd.nfsid.cn/555166.Doc
dbd.nfsid.cn/862356.Doc
dbs.nfsid.cn/854523.Doc
dbs.nfsid.cn/554006.Doc
dbs.nfsid.cn/328625.Doc
dbs.nfsid.cn/217702.Doc
dbs.nfsid.cn/167396.Doc
dbs.nfsid.cn/158102.Doc
dbs.nfsid.cn/173227.Doc
dbs.nfsid.cn/395404.Doc
dbs.nfsid.cn/009393.Doc
dbs.nfsid.cn/760273.Doc
dba.nfsid.cn/248335.Doc
dba.nfsid.cn/872946.Doc
dba.nfsid.cn/661503.Doc
dba.nfsid.cn/228148.Doc
dba.nfsid.cn/597581.Doc
dba.nfsid.cn/248120.Doc
dba.nfsid.cn/650736.Doc
dba.nfsid.cn/807260.Doc
dba.nfsid.cn/455347.Doc
dba.nfsid.cn/662724.Doc
dbp.nfsid.cn/259243.Doc
dbp.nfsid.cn/578156.Doc
dbp.nfsid.cn/799518.Doc
dbp.nfsid.cn/517695.Doc
dbp.nfsid.cn/362961.Doc
dbp.nfsid.cn/614975.Doc
dbp.nfsid.cn/103595.Doc
dbp.nfsid.cn/029198.Doc
dbp.nfsid.cn/764235.Doc
dbp.nfsid.cn/987219.Doc
dbo.nfsid.cn/825095.Doc
dbo.nfsid.cn/491601.Doc
dbo.nfsid.cn/179370.Doc
dbo.nfsid.cn/418681.Doc
dbo.nfsid.cn/111220.Doc
dbo.nfsid.cn/407907.Doc
dbo.nfsid.cn/191685.Doc
dbo.nfsid.cn/666316.Doc
dbo.nfsid.cn/249819.Doc
dbo.nfsid.cn/425321.Doc
dbi.nfsid.cn/472641.Doc
dbi.nfsid.cn/480650.Doc
dbi.nfsid.cn/432676.Doc
dbi.nfsid.cn/919580.Doc
dbi.nfsid.cn/164703.Doc
dbi.nfsid.cn/247369.Doc
dbi.nfsid.cn/996527.Doc
dbi.nfsid.cn/391573.Doc
dbi.nfsid.cn/660630.Doc
dbi.nfsid.cn/565667.Doc
dbu.nfsid.cn/458004.Doc
dbu.nfsid.cn/992867.Doc
dbu.nfsid.cn/019293.Doc
dbu.nfsid.cn/495723.Doc
dbu.nfsid.cn/124819.Doc
dbu.nfsid.cn/998733.Doc
dbu.nfsid.cn/913903.Doc
dbu.nfsid.cn/946545.Doc
dbu.nfsid.cn/707806.Doc
dbu.nfsid.cn/293951.Doc
dby.nfsid.cn/902506.Doc
dby.nfsid.cn/590130.Doc
dby.nfsid.cn/123770.Doc
dby.nfsid.cn/868957.Doc
dby.nfsid.cn/846812.Doc
dby.nfsid.cn/912870.Doc
dby.nfsid.cn/419546.Doc
dby.nfsid.cn/866544.Doc
dby.nfsid.cn/565215.Doc
dby.nfsid.cn/569254.Doc
dbt.nfsid.cn/764235.Doc
dbt.nfsid.cn/677097.Doc
dbt.nfsid.cn/414808.Doc
dbt.nfsid.cn/893340.Doc
dbt.nfsid.cn/738519.Doc
dbt.nfsid.cn/700466.Doc
dbt.nfsid.cn/085053.Doc
dbt.nfsid.cn/554129.Doc
dbt.nfsid.cn/082845.Doc
dbt.nfsid.cn/482311.Doc
dbr.nfsid.cn/910849.Doc
dbr.nfsid.cn/541280.Doc
dbr.nfsid.cn/251017.Doc
dbr.nfsid.cn/133649.Doc
dbr.nfsid.cn/483497.Doc
dbr.nfsid.cn/212023.Doc
dbr.nfsid.cn/515281.Doc
dbr.nfsid.cn/503588.Doc
dbr.nfsid.cn/062305.Doc
dbr.nfsid.cn/331058.Doc
dbe.nfsid.cn/090474.Doc
dbe.nfsid.cn/371704.Doc
dbe.nfsid.cn/549051.Doc
dbe.nfsid.cn/607831.Doc
dbe.nfsid.cn/267198.Doc
dbe.nfsid.cn/475272.Doc
dbe.nfsid.cn/530466.Doc
dbe.nfsid.cn/206174.Doc
dbe.nfsid.cn/748169.Doc
dbe.nfsid.cn/716867.Doc
dbw.nfsid.cn/431080.Doc
dbw.nfsid.cn/976049.Doc
dbw.nfsid.cn/884927.Doc
dbw.nfsid.cn/922974.Doc
dbw.nfsid.cn/372936.Doc
dbw.nfsid.cn/772399.Doc
dbw.nfsid.cn/101957.Doc
dbw.nfsid.cn/601751.Doc
dbw.nfsid.cn/962345.Doc
dbw.nfsid.cn/187354.Doc
dbq.nfsid.cn/810286.Doc
dbq.nfsid.cn/965640.Doc
dbq.nfsid.cn/573662.Doc
dbq.nfsid.cn/644579.Doc
dbq.nfsid.cn/846080.Doc
dbq.nfsid.cn/464228.Doc
dbq.nfsid.cn/868042.Doc
dbq.nfsid.cn/606044.Doc
dbq.nfsid.cn/422062.Doc
dbq.nfsid.cn/539351.Doc
dvm.nfsid.cn/842846.Doc
dvm.nfsid.cn/880640.Doc
dvm.nfsid.cn/882602.Doc
dvm.nfsid.cn/044668.Doc
dvm.nfsid.cn/228248.Doc
dvm.nfsid.cn/644444.Doc
dvm.nfsid.cn/624846.Doc
dvm.nfsid.cn/062262.Doc
dvm.nfsid.cn/008420.Doc
dvm.nfsid.cn/262088.Doc
dvn.nfsid.cn/317553.Doc
dvn.nfsid.cn/335577.Doc
dvn.nfsid.cn/711993.Doc
dvn.nfsid.cn/999959.Doc
dvn.nfsid.cn/264860.Doc
dvn.nfsid.cn/204004.Doc
dvn.nfsid.cn/868824.Doc
dvn.nfsid.cn/460442.Doc
dvn.nfsid.cn/468602.Doc
dvn.nfsid.cn/420660.Doc
dvb.nfsid.cn/288482.Doc
dvb.nfsid.cn/642000.Doc
dvb.nfsid.cn/246884.Doc
dvb.nfsid.cn/826680.Doc
dvb.nfsid.cn/644842.Doc
dvb.nfsid.cn/797791.Doc
dvb.nfsid.cn/931991.Doc
dvb.nfsid.cn/242002.Doc
dvb.nfsid.cn/804840.Doc
dvb.nfsid.cn/844264.Doc
dvv.nfsid.cn/424400.Doc
dvv.nfsid.cn/939173.Doc
dvv.nfsid.cn/064648.Doc
dvv.nfsid.cn/824422.Doc
dvv.nfsid.cn/066242.Doc
dvv.nfsid.cn/046886.Doc
dvv.nfsid.cn/488648.Doc
dvv.nfsid.cn/208402.Doc
dvv.nfsid.cn/177917.Doc
dvv.nfsid.cn/886626.Doc
dvc.nfsid.cn/777719.Doc
dvc.nfsid.cn/264882.Doc
dvc.nfsid.cn/062628.Doc
dvc.nfsid.cn/846640.Doc
dvc.nfsid.cn/880866.Doc
dvc.nfsid.cn/977993.Doc
dvc.nfsid.cn/133915.Doc
dvc.nfsid.cn/020284.Doc
dvc.nfsid.cn/600844.Doc
dvc.nfsid.cn/822484.Doc
dvx.nfsid.cn/222408.Doc
dvx.nfsid.cn/624486.Doc
dvx.nfsid.cn/531311.Doc
dvx.nfsid.cn/511393.Doc
dvx.nfsid.cn/993991.Doc
dvx.nfsid.cn/420488.Doc
dvx.nfsid.cn/042020.Doc
dvx.nfsid.cn/060682.Doc
dvx.nfsid.cn/224480.Doc
dvx.nfsid.cn/048460.Doc
dvz.nfsid.cn/248844.Doc
dvz.nfsid.cn/644426.Doc
dvz.nfsid.cn/424660.Doc
dvz.nfsid.cn/480448.Doc
dvz.nfsid.cn/713959.Doc
dvz.nfsid.cn/995551.Doc
dvz.nfsid.cn/284480.Doc
dvz.nfsid.cn/060268.Doc
dvz.nfsid.cn/686008.Doc
dvz.nfsid.cn/042222.Doc
dvl.nfsid.cn/482668.Doc
dvl.nfsid.cn/842026.Doc
dvl.nfsid.cn/824886.Doc
dvl.nfsid.cn/460846.Doc
dvl.nfsid.cn/337357.Doc
dvl.nfsid.cn/608846.Doc
dvl.nfsid.cn/040062.Doc
dvl.nfsid.cn/622200.Doc
dvl.nfsid.cn/608400.Doc
dvl.nfsid.cn/268828.Doc
dvk.nfsid.cn/420802.Doc
dvk.nfsid.cn/824288.Doc
dvk.nfsid.cn/911135.Doc
dvk.nfsid.cn/208660.Doc
dvk.nfsid.cn/531313.Doc
dvk.nfsid.cn/844602.Doc
dvk.nfsid.cn/026084.Doc
dvk.nfsid.cn/480446.Doc
dvk.nfsid.cn/820644.Doc
dvk.nfsid.cn/739753.Doc
dvj.nfsid.cn/068642.Doc
dvj.nfsid.cn/440602.Doc
dvj.nfsid.cn/660804.Doc
dvj.nfsid.cn/004882.Doc
dvj.nfsid.cn/084004.Doc
dvj.nfsid.cn/597195.Doc
dvj.nfsid.cn/024864.Doc
dvj.nfsid.cn/113537.Doc
dvj.nfsid.cn/266600.Doc
dvj.nfsid.cn/020648.Doc
dvh.nfsid.cn/226842.Doc
dvh.nfsid.cn/884464.Doc
dvh.nfsid.cn/042262.Doc
dvh.nfsid.cn/040062.Doc
dvh.nfsid.cn/795533.Doc
dvh.nfsid.cn/024488.Doc
dvh.nfsid.cn/240200.Doc
dvh.nfsid.cn/444488.Doc
dvh.nfsid.cn/660448.Doc
dvh.nfsid.cn/826206.Doc
dvg.nfsid.cn/248646.Doc
dvg.nfsid.cn/131117.Doc
dvg.nfsid.cn/393131.Doc
dvg.nfsid.cn/608208.Doc
dvg.nfsid.cn/642606.Doc
dvg.nfsid.cn/604602.Doc
dvg.nfsid.cn/262084.Doc
dvg.nfsid.cn/602042.Doc
dvg.nfsid.cn/262622.Doc
dvg.nfsid.cn/422604.Doc
dvf.nfsid.cn/393959.Doc
dvf.nfsid.cn/626488.Doc
dvf.nfsid.cn/846844.Doc
dvf.nfsid.cn/064060.Doc
dvf.nfsid.cn/466642.Doc
dvf.nfsid.cn/206200.Doc
dvf.nfsid.cn/642002.Doc
dvf.nfsid.cn/846046.Doc
dvf.nfsid.cn/002606.Doc
dvf.nfsid.cn/888260.Doc
dvd.nfsid.cn/886846.Doc
dvd.nfsid.cn/062202.Doc
dvd.nfsid.cn/088824.Doc
dvd.nfsid.cn/262844.Doc
dvd.nfsid.cn/242824.Doc
dvd.nfsid.cn/640424.Doc
dvd.nfsid.cn/777955.Doc
dvd.nfsid.cn/400846.Doc
dvd.nfsid.cn/608022.Doc
dvd.nfsid.cn/084282.Doc
dvs.nfsid.cn/020880.Doc
dvs.nfsid.cn/882000.Doc
dvs.nfsid.cn/264242.Doc
dvs.nfsid.cn/266482.Doc
dvs.nfsid.cn/462068.Doc
dvs.nfsid.cn/971335.Doc
dvs.nfsid.cn/717937.Doc
dvs.nfsid.cn/868226.Doc
dvs.nfsid.cn/224640.Doc
dvs.nfsid.cn/002440.Doc
dva.nfsid.cn/648642.Doc
dva.nfsid.cn/208402.Doc
dva.nfsid.cn/393735.Doc
dva.nfsid.cn/246224.Doc
dva.nfsid.cn/022886.Doc
dva.nfsid.cn/602048.Doc
dva.nfsid.cn/008022.Doc
dva.nfsid.cn/860664.Doc
dva.nfsid.cn/222404.Doc
dva.nfsid.cn/806022.Doc
dvp.nfsid.cn/682882.Doc
dvp.nfsid.cn/084048.Doc
dvp.nfsid.cn/444600.Doc
dvp.nfsid.cn/717199.Doc
dvp.nfsid.cn/331955.Doc
dvp.nfsid.cn/444844.Doc
dvp.nfsid.cn/082606.Doc
dvp.nfsid.cn/408484.Doc
dvp.nfsid.cn/406064.Doc
dvp.nfsid.cn/282648.Doc
dvo.nfsid.cn/226282.Doc
dvo.nfsid.cn/535551.Doc
dvo.nfsid.cn/240420.Doc
dvo.nfsid.cn/040220.Doc
dvo.nfsid.cn/866882.Doc
dvo.nfsid.cn/999379.Doc
dvo.nfsid.cn/068668.Doc
dvo.nfsid.cn/608226.Doc
dvo.nfsid.cn/828006.Doc
dvo.nfsid.cn/408888.Doc
dvi.nfsid.cn/179957.Doc
dvi.nfsid.cn/840468.Doc
dvi.nfsid.cn/886824.Doc
dvi.nfsid.cn/628240.Doc
dvi.nfsid.cn/826004.Doc
dvi.nfsid.cn/248264.Doc
dvi.nfsid.cn/159359.Doc
dvi.nfsid.cn/626266.Doc
dvi.nfsid.cn/464060.Doc
dvi.nfsid.cn/733733.Doc
dvu.nfsid.cn/028482.Doc
dvu.nfsid.cn/117335.Doc
dvu.nfsid.cn/680246.Doc
dvu.nfsid.cn/868424.Doc
dvu.nfsid.cn/020282.Doc
dvu.nfsid.cn/402628.Doc
dvu.nfsid.cn/351193.Doc
dvu.nfsid.cn/044448.Doc
dvu.nfsid.cn/404286.Doc
dvu.nfsid.cn/202004.Doc
dvy.nfsid.cn/664824.Doc
dvy.nfsid.cn/173131.Doc
dvy.nfsid.cn/064046.Doc
dvy.nfsid.cn/082406.Doc
