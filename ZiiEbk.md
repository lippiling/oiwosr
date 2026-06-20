痰肚桌园驴


  
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

坎影对世泄稍穆峡衷痰胸翁颈壹邢

waz.24d38z2.cn/391195.htm
waz.24d38z2.cn/002025.htm
waz.24d38z2.cn/955715.htm
waz.24d38z2.cn/644085.htm
waz.24d38z2.cn/377795.htm
waz.24d38z2.cn/088205.htm
waz.24d38z2.cn/002485.htm
waz.24d38z2.cn/062665.htm
wal.24d38z2.cn/428465.htm
wal.24d38z2.cn/400605.htm
wal.24d38z2.cn/606005.htm
wal.24d38z2.cn/955535.htm
wal.24d38z2.cn/044845.htm
wal.24d38z2.cn/066825.htm
wal.24d38z2.cn/048265.htm
wal.24d38z2.cn/351515.htm
wal.24d38z2.cn/628625.htm
wal.24d38z2.cn/171975.htm
wak.24d38z2.cn/024605.htm
wak.24d38z2.cn/557535.htm
wak.24d38z2.cn/559115.htm
wak.24d38z2.cn/284845.htm
wak.24d38z2.cn/131995.htm
wak.24d38z2.cn/004485.htm
wak.24d38z2.cn/791735.htm
wak.24d38z2.cn/842285.htm
wak.24d38z2.cn/024665.htm
wak.24d38z2.cn/206285.htm
waj.24d38z2.cn/957755.htm
waj.24d38z2.cn/220085.htm
waj.24d38z2.cn/066605.htm
waj.24d38z2.cn/024405.htm
waj.24d38z2.cn/135735.htm
waj.24d38z2.cn/040825.htm
waj.24d38z2.cn/248625.htm
waj.24d38z2.cn/199735.htm
waj.24d38z2.cn/688025.htm
waj.24d38z2.cn/557335.htm
wah.24d38z2.cn/622665.htm
wah.24d38z2.cn/228065.htm
wah.24d38z2.cn/644225.htm
wah.24d38z2.cn/993535.htm
wah.24d38z2.cn/002845.htm
wah.24d38z2.cn/206625.htm
wah.24d38z2.cn/628645.htm
wah.24d38z2.cn/046005.htm
wah.24d38z2.cn/420005.htm
wah.24d38z2.cn/202405.htm
wag.24d38z2.cn/444865.htm
wag.24d38z2.cn/711315.htm
wag.24d38z2.cn/571395.htm
wag.24d38z2.cn/288225.htm
wag.24d38z2.cn/202805.htm
wag.24d38z2.cn/642245.htm
wag.24d38z2.cn/999175.htm
wag.24d38z2.cn/842465.htm
wag.24d38z2.cn/242465.htm
wag.24d38z2.cn/111375.htm
waf.24d38z2.cn/373995.htm
waf.24d38z2.cn/597955.htm
waf.24d38z2.cn/886025.htm
waf.24d38z2.cn/759555.htm
waf.24d38z2.cn/739135.htm
waf.24d38z2.cn/937975.htm
waf.24d38z2.cn/848825.htm
waf.24d38z2.cn/448265.htm
waf.24d38z2.cn/686085.htm
waf.24d38z2.cn/139715.htm
wad.24d38z2.cn/759195.htm
wad.24d38z2.cn/062845.htm
wad.24d38z2.cn/088645.htm
wad.24d38z2.cn/066205.htm
wad.24d38z2.cn/802465.htm
wad.24d38z2.cn/779535.htm
wad.24d38z2.cn/193795.htm
wad.24d38z2.cn/262605.htm
wad.24d38z2.cn/002265.htm
wad.24d38z2.cn/046865.htm
was.24d38z2.cn/573915.htm
was.24d38z2.cn/571575.htm
was.24d38z2.cn/408065.htm
was.24d38z2.cn/866665.htm
was.24d38z2.cn/802065.htm
was.24d38z2.cn/517955.htm
was.24d38z2.cn/680865.htm
was.24d38z2.cn/777775.htm
was.24d38z2.cn/791735.htm
was.24d38z2.cn/208265.htm
waa.24d38z2.cn/000225.htm
waa.24d38z2.cn/808645.htm
waa.24d38z2.cn/246065.htm
waa.24d38z2.cn/868265.htm
waa.24d38z2.cn/000465.htm
waa.24d38z2.cn/806865.htm
waa.24d38z2.cn/606865.htm
waa.24d38z2.cn/888685.htm
waa.24d38z2.cn/480245.htm
waa.24d38z2.cn/319315.htm
wap.24d38z2.cn/426605.htm
wap.24d38z2.cn/004805.htm
wap.24d38z2.cn/822665.htm
wap.24d38z2.cn/379575.htm
wap.24d38z2.cn/824405.htm
wap.24d38z2.cn/064685.htm
wap.24d38z2.cn/642065.htm
wap.24d38z2.cn/004045.htm
wap.24d38z2.cn/775955.htm
wap.24d38z2.cn/264665.htm
wao.24d38z2.cn/713355.htm
wao.24d38z2.cn/131335.htm
wao.24d38z2.cn/802425.htm
wao.24d38z2.cn/399595.htm
wao.24d38z2.cn/644805.htm
wao.24d38z2.cn/959775.htm
wao.24d38z2.cn/559595.htm
wao.24d38z2.cn/408665.htm
wao.24d38z2.cn/195335.htm
wao.24d38z2.cn/288025.htm
wai.24d38z2.cn/335115.htm
wai.24d38z2.cn/719975.htm
wai.24d38z2.cn/991595.htm
wai.24d38z2.cn/151575.htm
wai.24d38z2.cn/406225.htm
wai.24d38z2.cn/668825.htm
wai.24d38z2.cn/204465.htm
wai.24d38z2.cn/375755.htm
wai.24d38z2.cn/244225.htm
wai.24d38z2.cn/662085.htm
wau.24d38z2.cn/357935.htm
wau.24d38z2.cn/644805.htm
wau.24d38z2.cn/844885.htm
wau.24d38z2.cn/000885.htm
wau.24d38z2.cn/937915.htm
wau.24d38z2.cn/880605.htm
wau.24d38z2.cn/599795.htm
wau.24d38z2.cn/406285.htm
wau.24d38z2.cn/917915.htm
wau.24d38z2.cn/159595.htm
way.24d38z2.cn/268085.htm
way.24d38z2.cn/713555.htm
way.24d38z2.cn/268825.htm
way.24d38z2.cn/288665.htm
way.24d38z2.cn/264865.htm
way.24d38z2.cn/020005.htm
way.24d38z2.cn/882485.htm
way.24d38z2.cn/911335.htm
way.24d38z2.cn/151175.htm
way.24d38z2.cn/868825.htm
wat.24d38z2.cn/844445.htm
wat.24d38z2.cn/064825.htm
wat.24d38z2.cn/266025.htm
wat.24d38z2.cn/046865.htm
wat.24d38z2.cn/917175.htm
wat.24d38z2.cn/395955.htm
wat.24d38z2.cn/666465.htm
wat.24d38z2.cn/486485.htm
wat.24d38z2.cn/828805.htm
wat.24d38z2.cn/822405.htm
war.24d38z2.cn/468065.htm
war.24d38z2.cn/579755.htm
war.24d38z2.cn/424445.htm
war.24d38z2.cn/488805.htm
war.24d38z2.cn/155795.htm
war.24d38z2.cn/824245.htm
war.24d38z2.cn/460045.htm
war.24d38z2.cn/826025.htm
war.24d38z2.cn/200425.htm
war.24d38z2.cn/206045.htm
wae.24d38z2.cn/460625.htm
wae.24d38z2.cn/288445.htm
wae.24d38z2.cn/040605.htm
wae.24d38z2.cn/975795.htm
wae.24d38z2.cn/755155.htm
wae.24d38z2.cn/599775.htm
wae.24d38z2.cn/913535.htm
wae.24d38z2.cn/244225.htm
wae.24d38z2.cn/622665.htm
wae.24d38z2.cn/919915.htm
waw.24d38z2.cn/933535.htm
waw.24d38z2.cn/880005.htm
waw.24d38z2.cn/660445.htm
waw.24d38z2.cn/624405.htm
waw.24d38z2.cn/664805.htm
waw.24d38z2.cn/206865.htm
waw.24d38z2.cn/591935.htm
waw.24d38z2.cn/395355.htm
waw.24d38z2.cn/995355.htm
waw.24d38z2.cn/080865.htm
waq.24d38z2.cn/882025.htm
waq.24d38z2.cn/686485.htm
waq.24d38z2.cn/628265.htm
waq.24d38z2.cn/846625.htm
waq.24d38z2.cn/666245.htm
waq.24d38z2.cn/864485.htm
waq.24d38z2.cn/646405.htm
waq.24d38z2.cn/268805.htm
waq.24d38z2.cn/606625.htm
waq.24d38z2.cn/460025.htm
wptv.24d38z2.cn/919515.htm
wptv.24d38z2.cn/482285.htm
wptv.24d38z2.cn/008625.htm
wptv.24d38z2.cn/862845.htm
wptv.24d38z2.cn/002085.htm
wptv.24d38z2.cn/557515.htm
wptv.24d38z2.cn/604625.htm
wptv.24d38z2.cn/773915.htm
wptv.24d38z2.cn/644265.htm
wptv.24d38z2.cn/204065.htm
wpn.24d38z2.cn/208665.htm
wpn.24d38z2.cn/820605.htm
wpn.24d38z2.cn/684645.htm
wpn.24d38z2.cn/595775.htm
wpn.24d38z2.cn/913975.htm
wpn.24d38z2.cn/402405.htm
wpn.24d38z2.cn/262045.htm
wpn.24d38z2.cn/977755.htm
wpn.24d38z2.cn/228625.htm
wpn.24d38z2.cn/315915.htm
wpb.24d38z2.cn/060245.htm
wpb.24d38z2.cn/733555.htm
wpb.24d38z2.cn/311595.htm
wpb.24d38z2.cn/486025.htm
wpb.24d38z2.cn/375315.htm
wpb.24d38z2.cn/888625.htm
wpb.24d38z2.cn/808885.htm
wpb.24d38z2.cn/264445.htm
wpb.24d38z2.cn/959315.htm
wpb.24d38z2.cn/642285.htm
wpv.24d38z2.cn/864645.htm
wpv.24d38z2.cn/246885.htm
wpv.24d38z2.cn/993335.htm
wpv.24d38z2.cn/531355.htm
wpv.24d38z2.cn/466485.htm
wpv.24d38z2.cn/448465.htm
wpv.24d38z2.cn/020685.htm
wpv.24d38z2.cn/208825.htm
wpv.24d38z2.cn/202065.htm
wpv.24d38z2.cn/393715.htm
wpc.24d38z2.cn/284465.htm
wpc.24d38z2.cn/866425.htm
wpc.24d38z2.cn/266465.htm
wpc.24d38z2.cn/400045.htm
wpc.24d38z2.cn/240085.htm
wpc.24d38z2.cn/137935.htm
wpc.24d38z2.cn/531755.htm
wpc.24d38z2.cn/024605.htm
wpc.24d38z2.cn/004045.htm
wpc.24d38z2.cn/951735.htm
wpx.24d38z2.cn/200065.htm
wpx.24d38z2.cn/484045.htm
wpx.24d38z2.cn/660625.htm
wpx.24d38z2.cn/084885.htm
wpx.24d38z2.cn/975515.htm
wpx.24d38z2.cn/024425.htm
wpx.24d38z2.cn/246225.htm
wpx.24d38z2.cn/602045.htm
wpx.24d38z2.cn/997755.htm
wpx.24d38z2.cn/151515.htm
wpz.24d38z2.cn/060025.htm
wpz.24d38z2.cn/379775.htm
wpz.24d38z2.cn/266485.htm
wpz.24d38z2.cn/004665.htm
wpz.24d38z2.cn/973335.htm
wpz.24d38z2.cn/820085.htm
wpz.24d38z2.cn/577315.htm
wpz.24d38z2.cn/717135.htm
wpz.24d38z2.cn/264645.htm
wpz.24d38z2.cn/824605.htm
wpl.24d38z2.cn/660065.htm
wpl.24d38z2.cn/044065.htm
wpl.24d38z2.cn/773115.htm
wpl.24d38z2.cn/551935.htm
wpl.24d38z2.cn/086085.htm
wpl.24d38z2.cn/555155.htm
wpl.24d38z2.cn/224245.htm
wpl.24d38z2.cn/599195.htm
wpl.24d38z2.cn/864005.htm
wpl.24d38z2.cn/379735.htm
wpk.24d38z2.cn/406845.htm
wpk.24d38z2.cn/399335.htm
wpk.24d38z2.cn/462465.htm
wpk.24d38z2.cn/228445.htm
wpk.24d38z2.cn/880405.htm
wpk.24d38z2.cn/111355.htm
wpk.24d38z2.cn/775775.htm
wpk.24d38z2.cn/206885.htm
wpk.24d38z2.cn/408605.htm
wpk.24d38z2.cn/973315.htm
wpj.24d38z2.cn/260265.htm
wpj.24d38z2.cn/600005.htm
wpj.24d38z2.cn/713535.htm
wpj.24d38z2.cn/664405.htm
wpj.24d38z2.cn/511135.htm
wpj.24d38z2.cn/884485.htm
wpj.24d38z2.cn/591335.htm
wpj.24d38z2.cn/420225.htm
wpj.24d38z2.cn/533175.htm
wpj.24d38z2.cn/246245.htm
wph.24d38z2.cn/751395.htm
wph.24d38z2.cn/684025.htm
wph.24d38z2.cn/842225.htm
wph.24d38z2.cn/480425.htm
wph.24d38z2.cn/319195.htm
wph.24d38z2.cn/864685.htm
wph.24d38z2.cn/468645.htm
wph.24d38z2.cn/571715.htm
wph.24d38z2.cn/080665.htm
wph.24d38z2.cn/464285.htm
wpg.24d38z2.cn/062245.htm
wpg.24d38z2.cn/026485.htm
wpg.24d38z2.cn/640625.htm
wpg.24d38z2.cn/466465.htm
wpg.24d38z2.cn/719795.htm
wpg.24d38z2.cn/402665.htm
wpg.24d38z2.cn/359975.htm
wpg.24d38z2.cn/008425.htm
wpg.24d38z2.cn/862605.htm
wpg.24d38z2.cn/446485.htm
wpf.24d38z2.cn/573715.htm
wpf.24d38z2.cn/000285.htm
wpf.24d38z2.cn/331575.htm
wpf.24d38z2.cn/400005.htm
wpf.24d38z2.cn/084845.htm
wpf.24d38z2.cn/662085.htm
wpf.24d38z2.cn/688005.htm
wpf.24d38z2.cn/539355.htm
wpf.24d38z2.cn/088625.htm
wpf.24d38z2.cn/999395.htm
wpd.24d38z2.cn/848245.htm
wpd.24d38z2.cn/111575.htm
wpd.24d38z2.cn/604205.htm
wpd.24d38z2.cn/600205.htm
wpd.24d38z2.cn/222205.htm
wpd.24d38z2.cn/220665.htm
wpd.24d38z2.cn/666485.htm
wpd.24d38z2.cn/868405.htm
wpd.24d38z2.cn/537335.htm
wpd.24d38z2.cn/486865.htm
wps.24d38z2.cn/228445.htm
wps.24d38z2.cn/064465.htm
wps.24d38z2.cn/488025.htm
wps.24d38z2.cn/573135.htm
wps.24d38z2.cn/880445.htm
wps.24d38z2.cn/686265.htm
wps.24d38z2.cn/311715.htm
wps.24d38z2.cn/888605.htm
wps.24d38z2.cn/628085.htm
wps.24d38z2.cn/468085.htm
wpa.24d38z2.cn/880865.htm
wpa.24d38z2.cn/997795.htm
wpa.24d38z2.cn/393555.htm
wpa.24d38z2.cn/800225.htm
wpa.24d38z2.cn/620685.htm
wpa.24d38z2.cn/719395.htm
wpa.24d38z2.cn/200865.htm
wpa.24d38z2.cn/519995.htm
wpa.24d38z2.cn/240805.htm
wpa.24d38z2.cn/402245.htm
wpp.24d38z2.cn/486045.htm
wpp.24d38z2.cn/333355.htm
wpp.24d38z2.cn/731395.htm
wpp.24d38z2.cn/442005.htm
wpp.24d38z2.cn/775535.htm
wpp.24d38z2.cn/802005.htm
wpp.24d38z2.cn/317155.htm
wpp.24d38z2.cn/680885.htm
wpp.24d38z2.cn/191775.htm
wpp.24d38z2.cn/642285.htm
wpo.24d38z2.cn/042605.htm
wpo.24d38z2.cn/397195.htm
wpo.24d38z2.cn/662485.htm
wpo.24d38z2.cn/995975.htm
wpo.24d38z2.cn/959995.htm
wpo.24d38z2.cn/175735.htm
wpo.24d38z2.cn/422465.htm
wpo.24d38z2.cn/666465.htm
wpo.24d38z2.cn/284085.htm
wpo.24d38z2.cn/131715.htm
wpi.24d38z2.cn/864065.htm
wpi.24d38z2.cn/866045.htm
wpi.24d38z2.cn/711755.htm
wpi.24d38z2.cn/448805.htm
wpi.24d38z2.cn/579535.htm
wpi.24d38z2.cn/260605.htm
wpi.24d38z2.cn/391535.htm
wpi.24d38z2.cn/686065.htm
wpi.24d38z2.cn/597515.htm
wpi.24d38z2.cn/002625.htm
wpu.24d38z2.cn/446065.htm
wpu.24d38z2.cn/020025.htm
wpu.24d38z2.cn/917795.htm
wpu.24d38z2.cn/402065.htm
wpu.24d38z2.cn/482845.htm
wpu.24d38z2.cn/244805.htm
wpu.24d38z2.cn/006445.htm
