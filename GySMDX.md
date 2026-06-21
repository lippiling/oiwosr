饰俨捞涝照


  
-Xmx应根据可用内存和环境限制合理设置：物理机≤70%的可用内存可用内存～80%的容器小于memory limit预留10%～20%缓冲；-Xms和-Xmx必须相等，以避免性能抖动；还需要预留元空间、直接内存等非堆积费用，并通过RSS和GC日志验证实际效果。

看看物理内存和系统费用 -Xmx

不要一上来就设置 -Xmx16g，首先要看机器还剩多少真正可用的内存。使用 free -m 看 available 列，不是 total；在容器里要多加小心——Kubernetes 的 memory limit 是硬上限，JVM 超了会被 OOMKilled，而 -Xmx 必须严格小于(建议留住) 10%～20% 缓冲)。例如容器 limit 是 16Gi，-Xmx 最多设到 12g～14g。

物理机：推荐 -Xmx ≤ 可用内存的 70%～80%，给 OS、页面缓存，当地过程留出足够的空间
容器环境：必要 -Xmx &lt; memory limit，否则 GC 还没来得及跑，cgroup 已经杀进程
混合场景(如和 Redis、Nginx 同机）：按实际压测后 RSS 峰值反推，不要相信“理论价值”


-Xms 和 -Xmx 一定要相等
不等于主动引入性能抖动：堆从 -Xms 当扩容开始时，会触发额外的 Full GC 或 STW（尤其 ParallelGC），且 JVM 需反复向 OS 申请内存页面，延迟无法控制。G1 虽然稍微好一点，但还是不推荐动态伸缩。

所有的生产服务都设置相同的值，例如：-Xms8g -Xmx8g

小内存服务（≤2g)可以放宽，但也要避免 -Xms512m -Xmx2g 这种跨度过大的组合
Spring Boot 注：若使用 spring-boot-starter-actuator + management.endpoint.heapdump.show-info=true，堆越大，heap dump 文件生成越慢，可能会阻塞线程

不要忽视元空间和直接内存的“隐形费用”
-Xmx 只管堆，但 JVM 还要吃元空间（-XX:MaxMetaspaceSize）、线程栈（-Xss）、直接内存（-XX:MaxDirectMemorySize）、JIT 编译代码缓存等。这些加起来可能比堆多，特别是在大量反射、动态代理、Netty 场景下。



典型搭配：-Xmx8g -XX:MaxMetaspaceSize=512m -XX:MaxDirectMemorySize=2g

未设 -XX:MaxMetaspaceSize？当类加载器泄漏时，元空间无限增长，最终触发 java.lang.OutOfMemoryError: Metaspace

Netty 常用坑的应用：-Xmx4g 却没调 -XX:MaxDirectMemorySize，结果堆不满，直接内存先爆，报告 OutOfMemoryError: Direct buffer memory


验证是否正确：盯着看 GC 日志和 RSS 实际值
写参数不等于生效，更不用说合理了。真正要看的是 JVM 进程的 RSS（Resident Set Size）是否稳定、Full GC 是否频繁，老年占用率是否长期， >70%。

启动参数：-Xlog:gc*:file=gc.log:time,tags,level（JDK 11+)，不要用旧的 -XX:+PrintGCDetails

在操作过程中检查真实内存：ps -o pid,rss,comm -p &lt;pid&gt;，对比 rss/1024（MB）和 -Xmx 差值太大，说明非堆内存吃多了。
老年代持续 >85%？说明 -Xmx 其实还不够，或者有内存泄漏，光调大是没有用的

跳过的最常见步骤是未确认容器 cgroup memory.max 根据经验设定实际限制值 -Xmx。一上线就被 kill，连 GC 打出日志已经太晚了。	 

爸聊中独亢济诶庸彩陨兜段趟俸仔

m.etg.cccnt.cn/68880.Doc
m.etg.cccnt.cn/08288.Doc
m.etg.cccnt.cn/51173.Doc
m.etg.cccnt.cn/75713.Doc
m.etg.cccnt.cn/28424.Doc
m.etg.cccnt.cn/15137.Doc
m.etg.cccnt.cn/91515.Doc
m.etg.cccnt.cn/37737.Doc
m.etg.cccnt.cn/39735.Doc
m.etg.cccnt.cn/40220.Doc
m.etg.cccnt.cn/11939.Doc
m.etg.cccnt.cn/91791.Doc
m.etg.cccnt.cn/37935.Doc
m.etg.cccnt.cn/00640.Doc
m.etg.cccnt.cn/11999.Doc
m.etg.cccnt.cn/40200.Doc
m.etg.cccnt.cn/31911.Doc
m.etg.cccnt.cn/17915.Doc
m.etg.cccnt.cn/95777.Doc
m.etg.cccnt.cn/44006.Doc
m.etf.cccnt.cn/31517.Doc
m.etf.cccnt.cn/84068.Doc
m.etf.cccnt.cn/42266.Doc
m.etf.cccnt.cn/51953.Doc
m.etf.cccnt.cn/60846.Doc
m.etf.cccnt.cn/26648.Doc
m.etf.cccnt.cn/15791.Doc
m.etf.cccnt.cn/40248.Doc
m.etf.cccnt.cn/37137.Doc
m.etf.cccnt.cn/26864.Doc
m.etf.cccnt.cn/37591.Doc
m.etf.cccnt.cn/66842.Doc
m.etf.cccnt.cn/97337.Doc
m.etf.cccnt.cn/04206.Doc
m.etf.cccnt.cn/19755.Doc
m.etf.cccnt.cn/35357.Doc
m.etf.cccnt.cn/53117.Doc
m.etf.cccnt.cn/97311.Doc
m.etf.cccnt.cn/55373.Doc
m.etf.cccnt.cn/11155.Doc
m.etd.cccnt.cn/55751.Doc
m.etd.cccnt.cn/79331.Doc
m.etd.cccnt.cn/68280.Doc
m.etd.cccnt.cn/57197.Doc
m.etd.cccnt.cn/80604.Doc
m.etd.cccnt.cn/71195.Doc
m.etd.cccnt.cn/02280.Doc
m.etd.cccnt.cn/88606.Doc
m.etd.cccnt.cn/59937.Doc
m.etd.cccnt.cn/97131.Doc
m.etd.cccnt.cn/26626.Doc
m.etd.cccnt.cn/97377.Doc
m.etd.cccnt.cn/00442.Doc
m.etd.cccnt.cn/31917.Doc
m.etd.cccnt.cn/73133.Doc
m.etd.cccnt.cn/17313.Doc
m.etd.cccnt.cn/91751.Doc
m.etd.cccnt.cn/20600.Doc
m.etd.cccnt.cn/33955.Doc
m.etd.cccnt.cn/53731.Doc
m.ets.cccnt.cn/17157.Doc
m.ets.cccnt.cn/62226.Doc
m.ets.cccnt.cn/39373.Doc
m.ets.cccnt.cn/59917.Doc
m.ets.cccnt.cn/37153.Doc
m.ets.cccnt.cn/95593.Doc
m.ets.cccnt.cn/07175.Doc
m.ets.cccnt.cn/75539.Doc
m.ets.cccnt.cn/82244.Doc
m.ets.cccnt.cn/91793.Doc
m.ets.cccnt.cn/64084.Doc
m.ets.cccnt.cn/80804.Doc
m.ets.cccnt.cn/35153.Doc
m.ets.cccnt.cn/93351.Doc
m.ets.cccnt.cn/84464.Doc
m.ets.cccnt.cn/46244.Doc
m.ets.cccnt.cn/53535.Doc
m.ets.cccnt.cn/59551.Doc
m.ets.cccnt.cn/22824.Doc
m.ets.cccnt.cn/79537.Doc
m.eta.cccnt.cn/93915.Doc
m.eta.cccnt.cn/79159.Doc
m.eta.cccnt.cn/55153.Doc
m.eta.cccnt.cn/26424.Doc
m.eta.cccnt.cn/44846.Doc
m.eta.cccnt.cn/80408.Doc
m.eta.cccnt.cn/79531.Doc
m.eta.cccnt.cn/80808.Doc
m.eta.cccnt.cn/13331.Doc
m.eta.cccnt.cn/86244.Doc
m.eta.cccnt.cn/91717.Doc
m.eta.cccnt.cn/04026.Doc
m.eta.cccnt.cn/24604.Doc
m.eta.cccnt.cn/68480.Doc
m.eta.cccnt.cn/93555.Doc
m.eta.cccnt.cn/08020.Doc
m.eta.cccnt.cn/42680.Doc
m.eta.cccnt.cn/91591.Doc
m.eta.cccnt.cn/44406.Doc
m.eta.cccnt.cn/35173.Doc
m.etp.cccnt.cn/97731.Doc
m.etp.cccnt.cn/66624.Doc
m.etp.cccnt.cn/20622.Doc
m.etp.cccnt.cn/75577.Doc
m.etp.cccnt.cn/33577.Doc
m.etp.cccnt.cn/82684.Doc
m.etp.cccnt.cn/26606.Doc
m.etp.cccnt.cn/37553.Doc
m.etp.cccnt.cn/93935.Doc
m.etp.cccnt.cn/42206.Doc
m.etp.cccnt.cn/73399.Doc
m.etp.cccnt.cn/13737.Doc
m.etp.cccnt.cn/33199.Doc
m.etp.cccnt.cn/59117.Doc
m.etp.cccnt.cn/37999.Doc
m.etp.cccnt.cn/84866.Doc
m.etp.cccnt.cn/46244.Doc
m.etp.cccnt.cn/55979.Doc
m.etp.cccnt.cn/46666.Doc
m.etp.cccnt.cn/82482.Doc
m.eto.cccnt.cn/17593.Doc
m.eto.cccnt.cn/95115.Doc
m.eto.cccnt.cn/53751.Doc
m.eto.cccnt.cn/00680.Doc
m.eto.cccnt.cn/84002.Doc
m.eto.cccnt.cn/06408.Doc
m.eto.cccnt.cn/93311.Doc
m.eto.cccnt.cn/28040.Doc
m.eto.cccnt.cn/34232.Doc
m.eto.cccnt.cn/00401.Doc
m.eto.cccnt.cn/24209.Doc
m.eto.cccnt.cn/40015.Doc
m.eto.cccnt.cn/13847.Doc
m.eto.cccnt.cn/83362.Doc
m.eto.cccnt.cn/60647.Doc
m.eto.cccnt.cn/66914.Doc
m.eto.cccnt.cn/57958.Doc
m.eto.cccnt.cn/99777.Doc
m.eto.cccnt.cn/76145.Doc
m.eto.cccnt.cn/65239.Doc
m.eti.cccnt.cn/17234.Doc
m.eti.cccnt.cn/44901.Doc
m.eti.cccnt.cn/91682.Doc
m.eti.cccnt.cn/03724.Doc
m.eti.cccnt.cn/82257.Doc
m.eti.cccnt.cn/51293.Doc
m.eti.cccnt.cn/08361.Doc
m.eti.cccnt.cn/30677.Doc
m.eti.cccnt.cn/36879.Doc
m.eti.cccnt.cn/06491.Doc
m.eti.cccnt.cn/44682.Doc
m.eti.cccnt.cn/52148.Doc
m.eti.cccnt.cn/57265.Doc
m.eti.cccnt.cn/88152.Doc
m.eti.cccnt.cn/07716.Doc
m.eti.cccnt.cn/79012.Doc
m.eti.cccnt.cn/84563.Doc
m.eti.cccnt.cn/35046.Doc
m.eti.cccnt.cn/11822.Doc
m.eti.cccnt.cn/65641.Doc
m.etu.cccnt.cn/03561.Doc
m.etu.cccnt.cn/59857.Doc
m.etu.cccnt.cn/92438.Doc
m.etu.cccnt.cn/81888.Doc
m.etu.cccnt.cn/18434.Doc
m.etu.cccnt.cn/42052.Doc
m.etu.cccnt.cn/77419.Doc
m.etu.cccnt.cn/84619.Doc
m.etu.cccnt.cn/67942.Doc
m.etu.cccnt.cn/74871.Doc
m.etu.cccnt.cn/50852.Doc
m.etu.cccnt.cn/78640.Doc
m.etu.cccnt.cn/99503.Doc
m.etu.cccnt.cn/04212.Doc
m.etu.cccnt.cn/24232.Doc
m.etu.cccnt.cn/83082.Doc
m.etu.cccnt.cn/78966.Doc
m.etu.cccnt.cn/71050.Doc
m.etu.cccnt.cn/28008.Doc
m.etu.cccnt.cn/93731.Doc
m.ety.cccnt.cn/55713.Doc
m.ety.cccnt.cn/44062.Doc
m.ety.cccnt.cn/00484.Doc
m.ety.cccnt.cn/93795.Doc
m.ety.cccnt.cn/00044.Doc
m.ety.cccnt.cn/64242.Doc
m.ety.cccnt.cn/84428.Doc
m.ety.cccnt.cn/11391.Doc
m.ety.cccnt.cn/17719.Doc
m.ety.cccnt.cn/31537.Doc
m.ety.cccnt.cn/91377.Doc
m.ety.cccnt.cn/11133.Doc
m.ety.cccnt.cn/19995.Doc
m.ety.cccnt.cn/66228.Doc
m.ety.cccnt.cn/68428.Doc
m.ety.cccnt.cn/82864.Doc
m.ety.cccnt.cn/82226.Doc
m.ety.cccnt.cn/04660.Doc
m.ety.cccnt.cn/95711.Doc
m.ety.cccnt.cn/02002.Doc
m.ett.cccnt.cn/62824.Doc
m.ett.cccnt.cn/62428.Doc
m.ett.cccnt.cn/80400.Doc
m.ett.cccnt.cn/35939.Doc
m.ett.cccnt.cn/75111.Doc
m.ett.cccnt.cn/40040.Doc
m.ett.cccnt.cn/40424.Doc
m.ett.cccnt.cn/31759.Doc
m.ett.cccnt.cn/44044.Doc
m.ett.cccnt.cn/66628.Doc
m.ett.cccnt.cn/22004.Doc
m.ett.cccnt.cn/40422.Doc
m.ett.cccnt.cn/42648.Doc
m.ett.cccnt.cn/62408.Doc
m.ett.cccnt.cn/66688.Doc
m.ett.cccnt.cn/80686.Doc
m.ett.cccnt.cn/71557.Doc
m.ett.cccnt.cn/35975.Doc
m.ett.cccnt.cn/73515.Doc
m.ett.cccnt.cn/86204.Doc
m.etr.cccnt.cn/46844.Doc
m.etr.cccnt.cn/44666.Doc
m.etr.cccnt.cn/08280.Doc
m.etr.cccnt.cn/88206.Doc
m.etr.cccnt.cn/97757.Doc
m.etr.cccnt.cn/82084.Doc
m.etr.cccnt.cn/35139.Doc
m.etr.cccnt.cn/19379.Doc
m.etr.cccnt.cn/48666.Doc
m.etr.cccnt.cn/59919.Doc
m.etr.cccnt.cn/15517.Doc
m.etr.cccnt.cn/22862.Doc
m.etr.cccnt.cn/22024.Doc
m.etr.cccnt.cn/22844.Doc
m.etr.cccnt.cn/80282.Doc
m.etr.cccnt.cn/88228.Doc
m.etr.cccnt.cn/68024.Doc
m.etr.cccnt.cn/80444.Doc
m.etr.cccnt.cn/55993.Doc
m.etr.cccnt.cn/04008.Doc
m.ete.cccnt.cn/26204.Doc
m.ete.cccnt.cn/15513.Doc
m.ete.cccnt.cn/40284.Doc
m.ete.cccnt.cn/13935.Doc
m.ete.cccnt.cn/57359.Doc
m.ete.cccnt.cn/24828.Doc
m.ete.cccnt.cn/75117.Doc
m.ete.cccnt.cn/33117.Doc
m.ete.cccnt.cn/17317.Doc
m.ete.cccnt.cn/51917.Doc
m.ete.cccnt.cn/77795.Doc
m.ete.cccnt.cn/93777.Doc
m.ete.cccnt.cn/42222.Doc
m.ete.cccnt.cn/97135.Doc
m.ete.cccnt.cn/55957.Doc
m.ete.cccnt.cn/28846.Doc
m.ete.cccnt.cn/15113.Doc
m.ete.cccnt.cn/33511.Doc
m.ete.cccnt.cn/57191.Doc
m.ete.cccnt.cn/53195.Doc
m.etw.cccnt.cn/77591.Doc
m.etw.cccnt.cn/82808.Doc
m.etw.cccnt.cn/64288.Doc
m.etw.cccnt.cn/33177.Doc
m.etw.cccnt.cn/17957.Doc
m.etw.cccnt.cn/13535.Doc
m.etw.cccnt.cn/06268.Doc
m.etw.cccnt.cn/91915.Doc
m.etw.cccnt.cn/86248.Doc
m.etw.cccnt.cn/95117.Doc
m.etw.cccnt.cn/26800.Doc
m.etw.cccnt.cn/64204.Doc
m.etw.cccnt.cn/15339.Doc
m.etw.cccnt.cn/82444.Doc
m.etw.cccnt.cn/00062.Doc
m.etw.cccnt.cn/33515.Doc
m.etw.cccnt.cn/08868.Doc
m.etw.cccnt.cn/02808.Doc
m.etw.cccnt.cn/88044.Doc
m.etw.cccnt.cn/20262.Doc
m.etq.cccnt.cn/82662.Doc
m.etq.cccnt.cn/57173.Doc
m.etq.cccnt.cn/19739.Doc
m.etq.cccnt.cn/88084.Doc
m.etq.cccnt.cn/06088.Doc
m.etq.cccnt.cn/55991.Doc
m.etq.cccnt.cn/19137.Doc
m.etq.cccnt.cn/31513.Doc
m.etq.cccnt.cn/00482.Doc
m.etq.cccnt.cn/48828.Doc
m.etq.cccnt.cn/39113.Doc
m.etq.cccnt.cn/60806.Doc
m.etq.cccnt.cn/84086.Doc
m.etq.cccnt.cn/93799.Doc
m.etq.cccnt.cn/28402.Doc
m.etq.cccnt.cn/57559.Doc
m.etq.cccnt.cn/84600.Doc
m.etq.cccnt.cn/95599.Doc
m.etq.cccnt.cn/20444.Doc
m.etq.cccnt.cn/86440.Doc
m.erm.cccnt.cn/66868.Doc
m.erm.cccnt.cn/79393.Doc
m.erm.cccnt.cn/11373.Doc
m.erm.cccnt.cn/11975.Doc
m.erm.cccnt.cn/73375.Doc
m.erm.cccnt.cn/02002.Doc
m.erm.cccnt.cn/35511.Doc
m.erm.cccnt.cn/04268.Doc
m.erm.cccnt.cn/02082.Doc
m.erm.cccnt.cn/20062.Doc
m.erm.cccnt.cn/02048.Doc
m.erm.cccnt.cn/53157.Doc
m.erm.cccnt.cn/17313.Doc
m.erm.cccnt.cn/71331.Doc
m.erm.cccnt.cn/06802.Doc
m.erm.cccnt.cn/33511.Doc
m.erm.cccnt.cn/11919.Doc
m.erm.cccnt.cn/59319.Doc
m.erm.cccnt.cn/33399.Doc
m.erm.cccnt.cn/79319.Doc
m.ern.cccnt.cn/62486.Doc
m.ern.cccnt.cn/84804.Doc
m.ern.cccnt.cn/62444.Doc
m.ern.cccnt.cn/99735.Doc
m.ern.cccnt.cn/95791.Doc
m.ern.cccnt.cn/06860.Doc
m.ern.cccnt.cn/53317.Doc
m.ern.cccnt.cn/62444.Doc
m.ern.cccnt.cn/13931.Doc
m.ern.cccnt.cn/33179.Doc
m.ern.cccnt.cn/57937.Doc
m.ern.cccnt.cn/13959.Doc
m.ern.cccnt.cn/51573.Doc
m.ern.cccnt.cn/97553.Doc
m.ern.cccnt.cn/02286.Doc
m.ern.cccnt.cn/44428.Doc
m.ern.cccnt.cn/48668.Doc
m.ern.cccnt.cn/55573.Doc
m.ern.cccnt.cn/48626.Doc
m.ern.cccnt.cn/99739.Doc
m.erb.cccnt.cn/08026.Doc
m.erb.cccnt.cn/86000.Doc
m.erb.cccnt.cn/71393.Doc
m.erb.cccnt.cn/13171.Doc
m.erb.cccnt.cn/97797.Doc
m.erb.cccnt.cn/75957.Doc
m.erb.cccnt.cn/37515.Doc
m.erb.cccnt.cn/73531.Doc
m.erb.cccnt.cn/55533.Doc
m.erb.cccnt.cn/24602.Doc
m.erb.cccnt.cn/37975.Doc
m.erb.cccnt.cn/95951.Doc
m.erb.cccnt.cn/88624.Doc
m.erb.cccnt.cn/28866.Doc
m.erb.cccnt.cn/71935.Doc
m.erb.cccnt.cn/20088.Doc
m.erb.cccnt.cn/44000.Doc
m.erb.cccnt.cn/28464.Doc
m.erb.cccnt.cn/13311.Doc
m.erb.cccnt.cn/62808.Doc
m.erv.cccnt.cn/51595.Doc
m.erv.cccnt.cn/19533.Doc
m.erv.cccnt.cn/35319.Doc
m.erv.cccnt.cn/44420.Doc
m.erv.cccnt.cn/75111.Doc
m.erv.cccnt.cn/02820.Doc
m.erv.cccnt.cn/40426.Doc
m.erv.cccnt.cn/80866.Doc
m.erv.cccnt.cn/79913.Doc
m.erv.cccnt.cn/93955.Doc
m.erv.cccnt.cn/95351.Doc
m.erv.cccnt.cn/84042.Doc
m.erv.cccnt.cn/62662.Doc
m.erv.cccnt.cn/71193.Doc
m.erv.cccnt.cn/71553.Doc
m.erv.cccnt.cn/13119.Doc
m.erv.cccnt.cn/35319.Doc
m.erv.cccnt.cn/99191.Doc
m.erv.cccnt.cn/20668.Doc
m.erv.cccnt.cn/19551.Doc
m.erc.cccnt.cn/40864.Doc
m.erc.cccnt.cn/77537.Doc
m.erc.cccnt.cn/08466.Doc
m.erc.cccnt.cn/13359.Doc
m.erc.cccnt.cn/55193.Doc
m.erc.cccnt.cn/08682.Doc
m.erc.cccnt.cn/06862.Doc
m.erc.cccnt.cn/86468.Doc
m.erc.cccnt.cn/02068.Doc
m.erc.cccnt.cn/80042.Doc
m.erc.cccnt.cn/37797.Doc
m.erc.cccnt.cn/24624.Doc
m.erc.cccnt.cn/15931.Doc
m.erc.cccnt.cn/48028.Doc
m.erc.cccnt.cn/11795.Doc
