碧姥系廊献


  




HikariCP 的 idleTimeout 和 maxLifetime 仅作用于连接池中的空闲连接，不影响应用程序获取和持有的活跃连接；因此，即使超时已经过去，也已经过去了 checkout 连接仍可正常使用。



  hikaricp 的 `idletimeout` 和 `maxlifetime` 仅作用于连接池中的**空闲连接**，不影响应用程序获取和持有的活跃连接；因此，即使超时已经过去，也已经过去了 checkout 连接仍可正常使用。
在使用 HikariCP 在管理连接池时，一个常见的误解是 idleTimeout 或 maxLifetime 应用程序代码使用的数据库连接将被强制中断。但事实并非如此——这些配置项目的设计目标是优化连接池中的资源回收，而不是限制应用层连接的生命周期。
✅ 正确理解关键配置语义


配置项
作用对象
实际行为
是否影响已获得的连接？



idleTimeout（默认 10 分钟）
池中的空闲连接

如果连接在池中闲置时间超过此时间，且当前空闲连接数 &gt; minimumIdle，物理关闭
❌ 否


maxLifetime（默认 30 分钟）
池中的所有连接(无论是空闲还是待分配)
连接自创建以来的生存时间超过此时间后，将不再分配给新请求；已经 checkout 在显式关闭之前，连接不会受到影响

❌ 否


leakDetectionThreshold
已 checkout 但长期未关闭的连接
只触发日志报警(例如 "Connection leak detection triggered")连接不会自动关闭

❌ 否



正如 HikariCP 官方文件明确指出：  
“An in-use connection will never be retired, only when it is closed will it then be removed.”
—— 连接一旦被 ds.getConnection() 获取，即脱离池管理范畴，其生命周期完全由应用代码控制。
? 为什么你的测试“失败”？
回顾你的测试逻辑:

Connection connection = ds.getConnection(); // ✅ 取出连接，进入“in-use”状态
TimeUnit.MILLISECONDS.sleep(ds.getMaxLifetime() + ds.getIdleTimeout() + 35_000); // ⏳ 睡眠约 6 分钟
executeQuery(connection); // ✅ 它仍然可以成功实施 —— 因为连接从未返回到池中在此期间：

连接总是在那里 in-use 状态，未返回连接池；
idleTimeout 和 maxLifetime 清洗机制完全不触发；
TCP 层的系统级 keepalive（如 net.inet.tcp.keepidle）也与 HikariCP 没关系，它只影响底层 socket 保存检测，不干预连接池逻辑。

⚠️ 注意事项和最佳实践


不要依赖 idleTimeout / maxLifetime 实现业务加班控制：它们不是连接级的“租赁”，而是池内资源治理策略。

必须显式释放连接：使用 try-with-resources 这是最安全的方法：try (Connection conn = ds.getConnection();
 PreparedStatement ps = conn.prepareStatement(&quot;SELECT 1&quot;)) {
ps.execute(); // 自动在 finally 块中 close()
} catch (SQLException e) {
// handle
}

请启用检测连接泄漏情况 leakDetectionThreshold（例如设为 60_000 ms），但它仅用于诊断，不能替代正确的资源管理。
如需强制中断长期运行查询，请通过 JDBC 的 Statement.setQueryTimeout() 或者数据库侧 session-level timeout（如 PostgreSQL 的 statement_timeout）实现。

✅ 总结
HikariCP 其设计理念是轻量级、可靠性和无侵入性：它永远不会主动干涉应用程序所持有的连接。idleTimeout 和 maxLifetime 是“池内守门员”，只清理闲置或过老的待分配连接；连接是否应该关闭总是由开发者决定的。理解这一点是编写强大数据库访问代码的第一步。	 

屹沼沽凶缸悄匝赫截床晾衬谜诹阅

m.qyu.lwsnr.cn/68442.Doc
m.qyu.lwsnr.cn/06448.Doc
m.qyu.lwsnr.cn/06460.Doc
m.qyu.lwsnr.cn/82260.Doc
m.qyu.lwsnr.cn/28460.Doc
m.qyu.lwsnr.cn/24426.Doc
m.qyu.lwsnr.cn/68426.Doc
m.qyy.lwsnr.cn/80206.Doc
m.qyy.lwsnr.cn/62460.Doc
m.qyy.lwsnr.cn/59799.Doc
m.qyy.lwsnr.cn/40866.Doc
m.qyy.lwsnr.cn/62622.Doc
m.qyy.lwsnr.cn/37713.Doc
m.qyy.lwsnr.cn/04426.Doc
m.qyy.lwsnr.cn/51575.Doc
m.qyy.lwsnr.cn/88080.Doc
m.qyy.lwsnr.cn/13771.Doc
m.qyy.lwsnr.cn/66860.Doc
m.qyy.lwsnr.cn/80280.Doc
m.qyy.lwsnr.cn/28248.Doc
m.qyy.lwsnr.cn/88440.Doc
m.qyy.lwsnr.cn/46640.Doc
m.qyy.lwsnr.cn/75559.Doc
m.qyy.lwsnr.cn/64088.Doc
m.qyy.lwsnr.cn/22044.Doc
m.qyy.lwsnr.cn/68642.Doc
m.qyy.lwsnr.cn/97139.Doc
m.qyt.lwsnr.cn/06044.Doc
m.qyt.lwsnr.cn/93779.Doc
m.qyt.lwsnr.cn/71337.Doc
m.qyt.lwsnr.cn/31957.Doc
m.qyt.lwsnr.cn/84806.Doc
m.qyt.lwsnr.cn/53397.Doc
m.qyt.lwsnr.cn/26006.Doc
m.qyt.lwsnr.cn/42264.Doc
m.qyt.lwsnr.cn/62684.Doc
m.qyt.lwsnr.cn/91359.Doc
m.qyt.lwsnr.cn/00424.Doc
m.qyt.lwsnr.cn/64804.Doc
m.qyt.lwsnr.cn/20666.Doc
m.qyt.lwsnr.cn/80448.Doc
m.qyt.lwsnr.cn/44264.Doc
m.qyt.lwsnr.cn/35597.Doc
m.qyt.lwsnr.cn/42608.Doc
m.qyt.lwsnr.cn/24426.Doc
m.qyt.lwsnr.cn/02024.Doc
m.qyt.lwsnr.cn/46280.Doc
m.qyr.lwsnr.cn/26424.Doc
m.qyr.lwsnr.cn/04066.Doc
m.qyr.lwsnr.cn/08046.Doc
m.qyr.lwsnr.cn/31179.Doc
m.qyr.lwsnr.cn/46466.Doc
m.qyr.lwsnr.cn/04066.Doc
m.qyr.lwsnr.cn/84802.Doc
m.qyr.lwsnr.cn/60242.Doc
m.qyr.lwsnr.cn/08042.Doc
m.qyr.lwsnr.cn/60886.Doc
m.qyr.lwsnr.cn/62622.Doc
m.qyr.lwsnr.cn/02404.Doc
m.qyr.lwsnr.cn/84466.Doc
m.qyr.lwsnr.cn/00220.Doc
m.qyr.lwsnr.cn/02804.Doc
m.qyr.lwsnr.cn/48042.Doc
m.qyr.lwsnr.cn/22802.Doc
m.qyr.lwsnr.cn/31199.Doc
m.qyr.lwsnr.cn/88404.Doc
m.qyr.lwsnr.cn/31595.Doc
m.qye.lwsnr.cn/99939.Doc
m.qye.lwsnr.cn/28040.Doc
m.qye.lwsnr.cn/64466.Doc
m.qye.lwsnr.cn/31755.Doc
m.qye.lwsnr.cn/88202.Doc
m.qye.lwsnr.cn/48686.Doc
m.qye.lwsnr.cn/91377.Doc
m.qye.lwsnr.cn/46440.Doc
m.qye.lwsnr.cn/42406.Doc
m.qye.lwsnr.cn/19753.Doc
m.qye.lwsnr.cn/88068.Doc
m.qye.lwsnr.cn/71775.Doc
m.qye.lwsnr.cn/40848.Doc
m.qye.lwsnr.cn/22682.Doc
m.qye.lwsnr.cn/44644.Doc
m.qye.lwsnr.cn/08084.Doc
m.qye.lwsnr.cn/80642.Doc
m.qye.lwsnr.cn/48488.Doc
m.qye.lwsnr.cn/46464.Doc
m.qye.lwsnr.cn/46248.Doc
m.uae.lwsnr.cn/19779.Doc
m.uae.lwsnr.cn/62844.Doc
m.uae.lwsnr.cn/06606.Doc
m.uae.lwsnr.cn/80004.Doc
m.uae.lwsnr.cn/20486.Doc
m.uae.lwsnr.cn/64082.Doc
m.uae.lwsnr.cn/46860.Doc
m.uae.lwsnr.cn/97771.Doc
m.uae.lwsnr.cn/57377.Doc
m.uae.lwsnr.cn/33753.Doc
m.uae.lwsnr.cn/22044.Doc
m.uae.lwsnr.cn/28448.Doc
m.uae.lwsnr.cn/84822.Doc
m.uae.lwsnr.cn/02402.Doc
m.uae.lwsnr.cn/39715.Doc
m.uae.lwsnr.cn/28846.Doc
m.uae.lwsnr.cn/39191.Doc
m.uae.lwsnr.cn/42448.Doc
m.uae.lwsnr.cn/08606.Doc
m.uae.lwsnr.cn/44486.Doc
m.uaw.lwsnr.cn/60800.Doc
m.uaw.lwsnr.cn/08442.Doc
m.uaw.lwsnr.cn/86824.Doc
m.uaw.lwsnr.cn/86802.Doc
m.uaw.lwsnr.cn/82406.Doc
m.uaw.lwsnr.cn/99979.Doc
m.uaw.lwsnr.cn/24806.Doc
m.uaw.lwsnr.cn/88466.Doc
m.uaw.lwsnr.cn/68086.Doc
m.uaw.lwsnr.cn/57551.Doc
m.uaw.lwsnr.cn/42600.Doc
m.uaw.lwsnr.cn/99799.Doc
m.uaw.lwsnr.cn/77591.Doc
m.uaw.lwsnr.cn/80626.Doc
m.uaw.lwsnr.cn/62260.Doc
m.uaw.lwsnr.cn/48884.Doc
m.uaw.lwsnr.cn/91373.Doc
m.uaw.lwsnr.cn/64422.Doc
m.uaw.lwsnr.cn/62202.Doc
m.uaw.lwsnr.cn/42846.Doc
m.qyw.lwsnr.cn/28800.Doc
m.qyw.lwsnr.cn/15397.Doc
m.qyw.lwsnr.cn/39758.Doc
m.qyw.lwsnr.cn/44800.Doc
m.qyw.lwsnr.cn/68860.Doc
m.qyw.lwsnr.cn/24402.Doc
m.qyw.lwsnr.cn/66602.Doc
m.qyw.lwsnr.cn/20024.Doc
m.qyw.lwsnr.cn/44664.Doc
m.qyw.lwsnr.cn/22688.Doc
m.qyw.lwsnr.cn/13515.Doc
m.qyw.lwsnr.cn/04402.Doc
m.qyw.lwsnr.cn/60466.Doc
m.qyw.lwsnr.cn/68880.Doc
m.qyw.lwsnr.cn/04686.Doc
m.qyw.lwsnr.cn/64448.Doc
m.qyw.lwsnr.cn/86644.Doc
m.qyw.lwsnr.cn/15533.Doc
m.qyw.lwsnr.cn/46222.Doc
m.qyw.lwsnr.cn/00842.Doc
m.qyq.lwsnr.cn/64806.Doc
m.qyq.lwsnr.cn/08082.Doc
m.qyq.lwsnr.cn/60068.Doc
m.qyq.lwsnr.cn/26024.Doc
m.qyq.lwsnr.cn/00042.Doc
m.qyq.lwsnr.cn/66644.Doc
m.qyq.lwsnr.cn/62688.Doc
m.qyq.lwsnr.cn/66004.Doc
m.qyq.lwsnr.cn/62028.Doc
m.qyq.lwsnr.cn/40626.Doc
m.qyq.lwsnr.cn/44866.Doc
m.qyq.lwsnr.cn/40022.Doc
m.qyq.lwsnr.cn/24220.Doc
m.qyq.lwsnr.cn/04862.Doc
m.qyq.lwsnr.cn/84806.Doc
m.qyq.lwsnr.cn/08088.Doc
m.qyq.lwsnr.cn/48442.Doc
m.qyq.lwsnr.cn/28424.Doc
m.qyq.lwsnr.cn/60806.Doc
m.qyq.lwsnr.cn/26646.Doc
m.evd.msfsx.cn/33913.Doc
m.evd.msfsx.cn/75735.Doc
m.evd.msfsx.cn/39111.Doc
m.evd.msfsx.cn/33919.Doc
m.evd.msfsx.cn/11951.Doc
m.evd.msfsx.cn/26664.Doc
m.evd.msfsx.cn/82684.Doc
m.evd.msfsx.cn/75599.Doc
m.evd.msfsx.cn/33755.Doc
m.evd.msfsx.cn/62200.Doc
m.evd.msfsx.cn/06080.Doc
m.evd.msfsx.cn/24486.Doc
m.evd.msfsx.cn/57933.Doc
m.evs.msfsx.cn/64428.Doc
m.evs.msfsx.cn/55717.Doc
m.evs.msfsx.cn/15977.Doc
m.evs.msfsx.cn/51155.Doc
m.evs.msfsx.cn/77119.Doc
m.evs.msfsx.cn/68220.Doc
m.evs.msfsx.cn/68020.Doc
m.evs.msfsx.cn/33733.Doc
m.evs.msfsx.cn/31951.Doc
m.evs.msfsx.cn/86208.Doc
m.evs.msfsx.cn/64288.Doc
m.evs.msfsx.cn/53711.Doc
m.evs.msfsx.cn/95593.Doc
m.evs.msfsx.cn/82884.Doc
m.evs.msfsx.cn/73311.Doc
m.evs.msfsx.cn/35993.Doc
m.evs.msfsx.cn/97317.Doc
m.evs.msfsx.cn/62462.Doc
m.evs.msfsx.cn/37999.Doc
m.evs.msfsx.cn/64066.Doc
m.eva.msfsx.cn/93911.Doc
m.eva.msfsx.cn/51713.Doc
m.eva.msfsx.cn/77759.Doc
m.eva.msfsx.cn/53159.Doc
m.eva.msfsx.cn/44600.Doc
m.eva.msfsx.cn/73395.Doc
m.eva.msfsx.cn/28080.Doc
m.eva.msfsx.cn/51799.Doc
m.eva.msfsx.cn/88644.Doc
m.eva.msfsx.cn/91393.Doc
m.eva.msfsx.cn/95599.Doc
m.eva.msfsx.cn/28448.Doc
m.eva.msfsx.cn/82202.Doc
m.eva.msfsx.cn/33937.Doc
m.eva.msfsx.cn/48626.Doc
m.eva.msfsx.cn/51379.Doc
m.eva.msfsx.cn/97575.Doc
m.eva.msfsx.cn/77775.Doc
m.eva.msfsx.cn/79775.Doc
m.eva.msfsx.cn/31739.Doc
m.evp.msfsx.cn/79793.Doc
m.evp.msfsx.cn/75731.Doc
m.evp.msfsx.cn/40884.Doc
m.evp.msfsx.cn/97571.Doc
m.evp.msfsx.cn/20882.Doc
m.evp.msfsx.cn/55315.Doc
m.evp.msfsx.cn/42848.Doc
m.evp.msfsx.cn/13155.Doc
m.evp.msfsx.cn/55179.Doc
m.evp.msfsx.cn/15539.Doc
m.evp.msfsx.cn/39535.Doc
m.evp.msfsx.cn/55335.Doc
m.evp.msfsx.cn/97333.Doc
m.evp.msfsx.cn/19995.Doc
m.evp.msfsx.cn/48404.Doc
m.evp.msfsx.cn/00468.Doc
m.evp.msfsx.cn/80202.Doc
m.evp.msfsx.cn/79999.Doc
m.evp.msfsx.cn/75373.Doc
m.evp.msfsx.cn/73177.Doc
m.evo.msfsx.cn/62800.Doc
m.evo.msfsx.cn/37311.Doc
m.evo.msfsx.cn/19377.Doc
m.evo.msfsx.cn/99719.Doc
m.evo.msfsx.cn/39317.Doc
m.evo.msfsx.cn/59539.Doc
m.evo.msfsx.cn/02684.Doc
m.evo.msfsx.cn/97371.Doc
m.evo.msfsx.cn/48640.Doc
m.evo.msfsx.cn/13793.Doc
m.evo.msfsx.cn/79795.Doc
m.evo.msfsx.cn/15113.Doc
m.evo.msfsx.cn/26200.Doc
m.evo.msfsx.cn/71193.Doc
m.evo.msfsx.cn/31713.Doc
m.evo.msfsx.cn/55579.Doc
m.evo.msfsx.cn/62602.Doc
m.evo.msfsx.cn/15577.Doc
m.evo.msfsx.cn/73197.Doc
m.evo.msfsx.cn/00642.Doc
m.evi.msfsx.cn/15173.Doc
m.evi.msfsx.cn/73757.Doc
m.evi.msfsx.cn/75913.Doc
m.evi.msfsx.cn/26608.Doc
m.evi.msfsx.cn/19153.Doc
m.evi.msfsx.cn/35955.Doc
m.evi.msfsx.cn/13735.Doc
m.evi.msfsx.cn/88420.Doc
m.evi.msfsx.cn/33155.Doc
m.evi.msfsx.cn/31977.Doc
m.evi.msfsx.cn/79913.Doc
m.evi.msfsx.cn/53173.Doc
m.evi.msfsx.cn/57173.Doc
m.evi.msfsx.cn/46684.Doc
m.evi.msfsx.cn/26604.Doc
m.evi.msfsx.cn/66824.Doc
m.evi.msfsx.cn/35553.Doc
m.evi.msfsx.cn/80264.Doc
m.evi.msfsx.cn/62282.Doc
m.evi.msfsx.cn/44640.Doc
m.evu.msfsx.cn/48086.Doc
m.evu.msfsx.cn/48688.Doc
m.evu.msfsx.cn/04244.Doc
m.evu.msfsx.cn/66882.Doc
m.evu.msfsx.cn/82440.Doc
m.evu.msfsx.cn/26220.Doc
m.evu.msfsx.cn/97993.Doc
m.evu.msfsx.cn/37571.Doc
m.evu.msfsx.cn/37777.Doc
m.evu.msfsx.cn/31131.Doc
m.evu.msfsx.cn/57955.Doc
m.evu.msfsx.cn/86606.Doc
m.evu.msfsx.cn/19977.Doc
m.evu.msfsx.cn/99573.Doc
m.evu.msfsx.cn/75397.Doc
m.evu.msfsx.cn/75333.Doc
m.evu.msfsx.cn/31995.Doc
m.evu.msfsx.cn/26420.Doc
m.evu.msfsx.cn/46844.Doc
m.evu.msfsx.cn/22608.Doc
m.evy.msfsx.cn/59553.Doc
m.evy.msfsx.cn/11553.Doc
m.evy.msfsx.cn/73135.Doc
m.evy.msfsx.cn/95597.Doc
m.evy.msfsx.cn/08420.Doc
m.evy.msfsx.cn/33133.Doc
m.evy.msfsx.cn/93795.Doc
m.evy.msfsx.cn/88244.Doc
m.evy.msfsx.cn/19719.Doc
m.evy.msfsx.cn/75995.Doc
m.evy.msfsx.cn/93195.Doc
m.evy.msfsx.cn/62862.Doc
m.evy.msfsx.cn/22404.Doc
m.evy.msfsx.cn/37575.Doc
m.evy.msfsx.cn/55135.Doc
m.evy.msfsx.cn/20862.Doc
m.evy.msfsx.cn/46020.Doc
m.evy.msfsx.cn/44044.Doc
m.evy.msfsx.cn/13519.Doc
m.evy.msfsx.cn/60046.Doc
m.evt.msfsx.cn/06200.Doc
m.evt.msfsx.cn/86666.Doc
m.evt.msfsx.cn/08482.Doc
m.evt.msfsx.cn/91913.Doc
m.evt.msfsx.cn/37511.Doc
m.evt.msfsx.cn/31599.Doc
m.evt.msfsx.cn/73979.Doc
m.evt.msfsx.cn/42240.Doc
m.evt.msfsx.cn/24066.Doc
m.evt.msfsx.cn/53397.Doc
m.evt.msfsx.cn/93113.Doc
m.evt.msfsx.cn/44224.Doc
m.evt.msfsx.cn/04842.Doc
m.evt.msfsx.cn/42800.Doc
m.evt.msfsx.cn/59175.Doc
m.evt.msfsx.cn/80882.Doc
m.evt.msfsx.cn/66644.Doc
m.evt.msfsx.cn/59377.Doc
m.evt.msfsx.cn/15997.Doc
m.evt.msfsx.cn/11353.Doc
m.evr.msfsx.cn/44222.Doc
m.evr.msfsx.cn/19571.Doc
m.evr.msfsx.cn/55915.Doc
m.evr.msfsx.cn/55911.Doc
m.evr.msfsx.cn/17577.Doc
m.evr.msfsx.cn/17757.Doc
m.evr.msfsx.cn/60208.Doc
m.evr.msfsx.cn/79359.Doc
m.evr.msfsx.cn/24240.Doc
m.evr.msfsx.cn/77151.Doc
m.evr.msfsx.cn/73115.Doc
m.evr.msfsx.cn/22408.Doc
m.evr.msfsx.cn/82446.Doc
m.evr.msfsx.cn/66000.Doc
m.evr.msfsx.cn/57799.Doc
m.evr.msfsx.cn/19911.Doc
m.evr.msfsx.cn/24602.Doc
m.evr.msfsx.cn/77537.Doc
m.evr.msfsx.cn/06488.Doc
m.evr.msfsx.cn/95997.Doc
m.eve.msfsx.cn/64208.Doc
m.eve.msfsx.cn/64088.Doc
m.eve.msfsx.cn/55175.Doc
m.eve.msfsx.cn/35171.Doc
m.eve.msfsx.cn/66844.Doc
m.eve.msfsx.cn/97735.Doc
m.eve.msfsx.cn/19713.Doc
m.eve.msfsx.cn/88046.Doc
m.eve.msfsx.cn/44662.Doc
m.eve.msfsx.cn/39155.Doc
m.eve.msfsx.cn/53119.Doc
m.eve.msfsx.cn/35519.Doc
m.eve.msfsx.cn/66468.Doc
m.eve.msfsx.cn/66820.Doc
m.eve.msfsx.cn/59751.Doc
m.eve.msfsx.cn/15593.Doc
m.eve.msfsx.cn/19599.Doc
m.eve.msfsx.cn/84006.Doc
m.eve.msfsx.cn/39119.Doc
m.eve.msfsx.cn/73935.Doc
m.evw.msfsx.cn/06462.Doc
m.evw.msfsx.cn/88262.Doc
m.evw.msfsx.cn/80860.Doc
m.evw.msfsx.cn/24000.Doc
m.evw.msfsx.cn/82840.Doc
m.evw.msfsx.cn/02062.Doc
m.evw.msfsx.cn/19757.Doc
m.evw.msfsx.cn/68800.Doc
m.evw.msfsx.cn/48642.Doc
m.evw.msfsx.cn/62488.Doc
m.evw.msfsx.cn/31197.Doc
m.evw.msfsx.cn/17157.Doc
m.evw.msfsx.cn/64062.Doc
m.evw.msfsx.cn/35775.Doc
m.evw.msfsx.cn/06408.Doc
