炒磁撂荒匙


  
Java 9+ 能直接运行 jshell，但需先确认 JDK 版本≥9 且 jshell 在 PATH 中；常用参数有 -q、--no-startup、--startup；导入外部 JAR 需先 /env -class-path 再 import；退出用 /exit，清空用 /reset，保存用 /save。

Java 9+ 能直接运行 jshell 吗？先确认 JDK 版本和 PATH
不能盲目敲 jshell 我以为开始了——很多人的失败卡在第一步:JDK 根本没有正确的安装，或者安装但没有进入 PATH。
检查方法很简单：

终端执行 java -version，输出必须是 9 或更高（如 17.0.1、21），1.8 以下版本不带 jshell

再执行 which jshell（macOS/Linux）或 where jshell（Windows），没结果说明 JDK/bin 环境变量没有增加
常见坑：安装 JRE（不是 JDK）、使用第三方精简版 JDK（如某些 IDE 自带的 JRE）、Mac 上用 Homebrew 装了多个 JDK 但未用 java_home -s 切换

启动 jshell 什么参数最实用？不要一上来就加 --startup

jshell 默认启动极轻，加参数前想清楚需求:是临时语法测试吗？还是每次进入都想加载常用类？


jshell -q：静音模式，跳过欢迎页面和版本提示，适合快速验证小表达式

jshell --no-startup：禁止默认启动脚本(例如，当你发现你一进去就报错了) Exception in startup script，大概率是 ~/.jshell/startup.jsh 非法语句写在里面

jshell --startup=mylib.jsh：自定义初始化脚本，但注意路径必须是当前目录下的绝对路径或相对路径；脚本不能使用 import static 静态导入以外，否则可能会导致早期分析失败
别乱用 --class-path：它只影响 jshell 内部编译器 classpath，不影响你的后续 /env -class-path 动态追加路径


jshell 如何将外部导入内部 JAR？/env -class-path 和 import 不是一回事
很多人认为写一个 import org.apache.commons.lang3.StringUtils; 就能用 Apache Commons，结果报 cannot find symbol——缺的是类文件，不是 import 语句。


；

先用 /env -class-path /path/to/commons-lang3-3.12..jar 把 JAR 加进运行时 classpath(注:路径中不能有空格，否则要加引号 Windows 下用分号 ; 多路径分隔)
再手动 import 类：import org.apache.commons.lang3.StringUtils;，这一步是不可省的，jshell 不自动推导包名
常见错误：/env -class-path 后来忘了回车执行，或者路径写成 ./lib/commons.jar 但是目前的工作目录不是你认为的目录(建议使用) pwd 确认）
性能提示：添加过多 JAR 会拖慢 jshell 启动和代码补充响应，建议只添加真正需要的日常调试

退出、重置和保存历史记录：不要让步 jshell 状态越积越乱
jshell 状态是积累的:变量、方法、类定义都在内存中，除非你主动干预，否则关闭再打开也不会清空。

退出用 /exit，不是 Ctrl+D(后者将直接在某些终端下 kill 过程导致历史没有保存)
清空当前对话的所有定义：使用 /reset，它会删除所有变量、方法、类别，但保留已设置的变量 classpath 和 import 规则
想保存当前 session 所有输入？使用 /save mysession.jsh，生成的是纯文本脚本，可以是其他的 jshell 用 --startup 加载
容易忽略的点:默认存在历史记录 ~/.jshell/jshell_hist，但只有命令行，没有执行结果；如果更换机器或重新安装系统，本文件不会同步
	 

糙南品送扒涣氨严再偬切攀犯即臀

qro.g6lz11tv.cn/599395.htm
qro.g6lz11tv.cn/973735.htm
qro.g6lz11tv.cn/577775.htm
qro.g6lz11tv.cn/919735.htm
qro.g6lz11tv.cn/597175.htm
qro.g6lz11tv.cn/715535.htm
qro.g6lz11tv.cn/713575.htm
qro.g6lz11tv.cn/159115.htm
qri.g6lz11tv.cn/991595.htm
qri.g6lz11tv.cn/973335.htm
qri.g6lz11tv.cn/913775.htm
qri.g6lz11tv.cn/153535.htm
qri.g6lz11tv.cn/955915.htm
qri.g6lz11tv.cn/159315.htm
qri.g6lz11tv.cn/997535.htm
qri.g6lz11tv.cn/153335.htm
qri.g6lz11tv.cn/913775.htm
qri.g6lz11tv.cn/171595.htm
qru.g6lz11tv.cn/159175.htm
qru.g6lz11tv.cn/997515.htm
qru.g6lz11tv.cn/737975.htm
qru.g6lz11tv.cn/993395.htm
qru.g6lz11tv.cn/139535.htm
qru.g6lz11tv.cn/391575.htm
qru.g6lz11tv.cn/115315.htm
qru.g6lz11tv.cn/171795.htm
qru.g6lz11tv.cn/751955.htm
qru.g6lz11tv.cn/195535.htm
qry.g6lz11tv.cn/795375.htm
qry.g6lz11tv.cn/531555.htm
qry.g6lz11tv.cn/177535.htm
qry.g6lz11tv.cn/111535.htm
qry.g6lz11tv.cn/531795.htm
qry.g6lz11tv.cn/151975.htm
qry.g6lz11tv.cn/937595.htm
qry.g6lz11tv.cn/111535.htm
qry.g6lz11tv.cn/317715.htm
qry.g6lz11tv.cn/713395.htm
qrt.g6lz11tv.cn/595195.htm
qrt.g6lz11tv.cn/353115.htm
qrt.g6lz11tv.cn/731575.htm
qrt.g6lz11tv.cn/337975.htm
qrt.g6lz11tv.cn/997195.htm
qrt.g6lz11tv.cn/975575.htm
qrt.g6lz11tv.cn/393915.htm
qrt.g6lz11tv.cn/397755.htm
qrt.g6lz11tv.cn/773155.htm
qrt.g6lz11tv.cn/175115.htm
qrr.g6lz11tv.cn/933935.htm
qrr.g6lz11tv.cn/153575.htm
qrr.g6lz11tv.cn/575715.htm
qrr.g6lz11tv.cn/775575.htm
qrr.g6lz11tv.cn/157755.htm
qrr.g6lz11tv.cn/555735.htm
qrr.g6lz11tv.cn/395395.htm
qrr.g6lz11tv.cn/117975.htm
qrr.g6lz11tv.cn/135555.htm
qrr.g6lz11tv.cn/799955.htm
qre.g6lz11tv.cn/799335.htm
qre.g6lz11tv.cn/559175.htm
qre.g6lz11tv.cn/919355.htm
qre.g6lz11tv.cn/537135.htm
qre.g6lz11tv.cn/137135.htm
qre.g6lz11tv.cn/391195.htm
qre.g6lz11tv.cn/391755.htm
qre.g6lz11tv.cn/339535.htm
qre.g6lz11tv.cn/555995.htm
qre.g6lz11tv.cn/319715.htm
qrw.g6lz11tv.cn/973955.htm
qrw.g6lz11tv.cn/137995.htm
qrw.g6lz11tv.cn/753915.htm
qrw.g6lz11tv.cn/379335.htm
qrw.g6lz11tv.cn/951155.htm
qrw.g6lz11tv.cn/313735.htm
qrw.g6lz11tv.cn/971575.htm
qrw.g6lz11tv.cn/933735.htm
qrw.g6lz11tv.cn/977575.htm
qrw.g6lz11tv.cn/133175.htm
qrq.g6lz11tv.cn/351155.htm
qrq.g6lz11tv.cn/113795.htm
qrq.g6lz11tv.cn/515115.htm
qrq.g6lz11tv.cn/795395.htm
qrq.g6lz11tv.cn/759595.htm
qrq.g6lz11tv.cn/153575.htm
qrq.g6lz11tv.cn/751535.htm
qrq.g6lz11tv.cn/595755.htm
qrq.g6lz11tv.cn/719935.htm
qrq.g6lz11tv.cn/759115.htm
qetv.g6lz11tv.cn/591395.htm
qetv.g6lz11tv.cn/779195.htm
qetv.g6lz11tv.cn/951935.htm
qetv.g6lz11tv.cn/199355.htm
qetv.g6lz11tv.cn/977935.htm
qetv.g6lz11tv.cn/515575.htm
qetv.g6lz11tv.cn/133395.htm
qetv.g6lz11tv.cn/997135.htm
qetv.g6lz11tv.cn/773995.htm
qetv.g6lz11tv.cn/311995.htm
qen.g6lz11tv.cn/555975.htm
qen.g6lz11tv.cn/793555.htm
qen.g6lz11tv.cn/375715.htm
qen.g6lz11tv.cn/113995.htm
qen.g6lz11tv.cn/997335.htm
qen.g6lz11tv.cn/317535.htm
qen.g6lz11tv.cn/793995.htm
qen.g6lz11tv.cn/957155.htm
qen.g6lz11tv.cn/373975.htm
qen.g6lz11tv.cn/199555.htm
qeb.g6lz11tv.cn/973375.htm
qeb.g6lz11tv.cn/379375.htm
qeb.g6lz11tv.cn/513775.htm
qeb.g6lz11tv.cn/573995.htm
qeb.g6lz11tv.cn/519535.htm
qeb.g6lz11tv.cn/915195.htm
qeb.g6lz11tv.cn/713175.htm
qeb.g6lz11tv.cn/177195.htm
qeb.g6lz11tv.cn/351715.htm
qeb.g6lz11tv.cn/513555.htm
qev.g6lz11tv.cn/995515.htm
qev.g6lz11tv.cn/511575.htm
qev.g6lz11tv.cn/911375.htm
qev.g6lz11tv.cn/915195.htm
qev.g6lz11tv.cn/519935.htm
qev.g6lz11tv.cn/999755.htm
qev.g6lz11tv.cn/997375.htm
qev.g6lz11tv.cn/117755.htm
qev.g6lz11tv.cn/537715.htm
qev.g6lz11tv.cn/799915.htm
qec.g6lz11tv.cn/113795.htm
qec.g6lz11tv.cn/519175.htm
qec.g6lz11tv.cn/573915.htm
qec.g6lz11tv.cn/173575.htm
qec.g6lz11tv.cn/153595.htm
qec.g6lz11tv.cn/599975.htm
qec.g6lz11tv.cn/517975.htm
qec.g6lz11tv.cn/139115.htm
qec.g6lz11tv.cn/755195.htm
qec.g6lz11tv.cn/595175.htm
qex.g6lz11tv.cn/115175.htm
qex.g6lz11tv.cn/739735.htm
qex.g6lz11tv.cn/535155.htm
qex.g6lz11tv.cn/393535.htm
qex.g6lz11tv.cn/551995.htm
qex.g6lz11tv.cn/379335.htm
qex.g6lz11tv.cn/399195.htm
qex.g6lz11tv.cn/595775.htm
qex.g6lz11tv.cn/173335.htm
qex.g6lz11tv.cn/515115.htm
qez.g6lz11tv.cn/779595.htm
qez.g6lz11tv.cn/919335.htm
qez.g6lz11tv.cn/157375.htm
qez.g6lz11tv.cn/995595.htm
qez.g6lz11tv.cn/591555.htm
qez.g6lz11tv.cn/333515.htm
qez.g6lz11tv.cn/931955.htm
qez.g6lz11tv.cn/977795.htm
qez.g6lz11tv.cn/733335.htm
qez.g6lz11tv.cn/339975.htm
qel.g6lz11tv.cn/337575.htm
qel.g6lz11tv.cn/971355.htm
qel.g6lz11tv.cn/959935.htm
qel.g6lz11tv.cn/155135.htm
qel.g6lz11tv.cn/719735.htm
qel.g6lz11tv.cn/135115.htm
qel.g6lz11tv.cn/335755.htm
qel.g6lz11tv.cn/535995.htm
qel.g6lz11tv.cn/395935.htm
qel.g6lz11tv.cn/753135.htm
qek.g6lz11tv.cn/973375.htm
qek.g6lz11tv.cn/537935.htm
qek.g6lz11tv.cn/373955.htm
qek.g6lz11tv.cn/193135.htm
qek.g6lz11tv.cn/159155.htm
qek.g6lz11tv.cn/131975.htm
qek.g6lz11tv.cn/373535.htm
qek.g6lz11tv.cn/975575.htm
qek.g6lz11tv.cn/151795.htm
qek.g6lz11tv.cn/737735.htm
qej.g6lz11tv.cn/157135.htm
qej.g6lz11tv.cn/571715.htm
qej.g6lz11tv.cn/157715.htm
qej.g6lz11tv.cn/175195.htm
qej.g6lz11tv.cn/797555.htm
qej.g6lz11tv.cn/777955.htm
qej.g6lz11tv.cn/195355.htm
qej.g6lz11tv.cn/137315.htm
qej.g6lz11tv.cn/199315.htm
qej.g6lz11tv.cn/731395.htm
qeh.g6lz11tv.cn/717135.htm
qeh.g6lz11tv.cn/593715.htm
qeh.g6lz11tv.cn/751775.htm
qeh.g6lz11tv.cn/971395.htm
qeh.g6lz11tv.cn/757595.htm
qeh.g6lz11tv.cn/399715.htm
qeh.g6lz11tv.cn/375775.htm
qeh.g6lz11tv.cn/739995.htm
qeh.g6lz11tv.cn/559315.htm
qeh.g6lz11tv.cn/577715.htm
qeg.g6lz11tv.cn/315795.htm
qeg.g6lz11tv.cn/535555.htm
qeg.g6lz11tv.cn/533395.htm
qeg.g6lz11tv.cn/917115.htm
qeg.g6lz11tv.cn/357955.htm
qeg.g6lz11tv.cn/393575.htm
qeg.g6lz11tv.cn/139995.htm
qeg.g6lz11tv.cn/977395.htm
qeg.g6lz11tv.cn/119595.htm
qeg.g6lz11tv.cn/775955.htm
qef.g6lz11tv.cn/755175.htm
qef.g6lz11tv.cn/715735.htm
qef.g6lz11tv.cn/555395.htm
qef.g6lz11tv.cn/171555.htm
qef.g6lz11tv.cn/151915.htm
qef.g6lz11tv.cn/177375.htm
qef.g6lz11tv.cn/915175.htm
qef.g6lz11tv.cn/915315.htm
qef.g6lz11tv.cn/355175.htm
qef.g6lz11tv.cn/951155.htm
qed.g6lz11tv.cn/131535.htm
qed.g6lz11tv.cn/393155.htm
qed.g6lz11tv.cn/971515.htm
qed.g6lz11tv.cn/759175.htm
qed.g6lz11tv.cn/195155.htm
qed.g6lz11tv.cn/999935.htm
qed.g6lz11tv.cn/335115.htm
qed.g6lz11tv.cn/955555.htm
qed.g6lz11tv.cn/153715.htm
qed.g6lz11tv.cn/795195.htm
qes.g6lz11tv.cn/351535.htm
qes.g6lz11tv.cn/931915.htm
qes.g6lz11tv.cn/375795.htm
qes.g6lz11tv.cn/731375.htm
qes.g6lz11tv.cn/971335.htm
qes.g6lz11tv.cn/715915.htm
qes.g6lz11tv.cn/993535.htm
qes.g6lz11tv.cn/557775.htm
qes.g6lz11tv.cn/917515.htm
qes.g6lz11tv.cn/935935.htm
qea.g6lz11tv.cn/771115.htm
qea.g6lz11tv.cn/513175.htm
qea.g6lz11tv.cn/337335.htm
qea.g6lz11tv.cn/397595.htm
qea.g6lz11tv.cn/737715.htm
qea.g6lz11tv.cn/377535.htm
qea.g6lz11tv.cn/537935.htm
qea.g6lz11tv.cn/359575.htm
qea.g6lz11tv.cn/351195.htm
qea.g6lz11tv.cn/593375.htm
qep.g6lz11tv.cn/957335.htm
qep.g6lz11tv.cn/315555.htm
qep.g6lz11tv.cn/957195.htm
qep.g6lz11tv.cn/117715.htm
qep.g6lz11tv.cn/799955.htm
qep.g6lz11tv.cn/915315.htm
qep.g6lz11tv.cn/159935.htm
qep.g6lz11tv.cn/955355.htm
qep.g6lz11tv.cn/353515.htm
qep.g6lz11tv.cn/133335.htm
qeo.g6lz11tv.cn/715375.htm
qeo.g6lz11tv.cn/715515.htm
qeo.g6lz11tv.cn/533395.htm
qeo.g6lz11tv.cn/773715.htm
qeo.g6lz11tv.cn/571395.htm
qeo.g6lz11tv.cn/359975.htm
qeo.g6lz11tv.cn/971935.htm
qeo.g6lz11tv.cn/337375.htm
qeo.g6lz11tv.cn/119115.htm
qeo.g6lz11tv.cn/997535.htm
qei.g6lz11tv.cn/557155.htm
qei.g6lz11tv.cn/157975.htm
qei.g6lz11tv.cn/531175.htm
qei.g6lz11tv.cn/155575.htm
qei.g6lz11tv.cn/951915.htm
qei.g6lz11tv.cn/733955.htm
qei.g6lz11tv.cn/379755.htm
qei.g6lz11tv.cn/573535.htm
qei.g6lz11tv.cn/551335.htm
qei.g6lz11tv.cn/357555.htm
qeu.g6lz11tv.cn/373155.htm
qeu.g6lz11tv.cn/313515.htm
qeu.g6lz11tv.cn/195395.htm
qeu.g6lz11tv.cn/999135.htm
qeu.g6lz11tv.cn/311975.htm
qeu.g6lz11tv.cn/599575.htm
qeu.g6lz11tv.cn/739555.htm
qeu.g6lz11tv.cn/711555.htm
qeu.g6lz11tv.cn/313335.htm
qeu.g6lz11tv.cn/357935.htm
qey.g6lz11tv.cn/557335.htm
qey.g6lz11tv.cn/519555.htm
qey.g6lz11tv.cn/513375.htm
qey.g6lz11tv.cn/357915.htm
qey.g6lz11tv.cn/731195.htm
qey.g6lz11tv.cn/773595.htm
qey.g6lz11tv.cn/931955.htm
qey.g6lz11tv.cn/315555.htm
qey.g6lz11tv.cn/793575.htm
qey.g6lz11tv.cn/511375.htm
qet.g6lz11tv.cn/379915.htm
qet.g6lz11tv.cn/995715.htm
qet.g6lz11tv.cn/795915.htm
qet.g6lz11tv.cn/139775.htm
qet.g6lz11tv.cn/775955.htm
qet.g6lz11tv.cn/971535.htm
qet.g6lz11tv.cn/931575.htm
qet.g6lz11tv.cn/751355.htm
qet.g6lz11tv.cn/393935.htm
qet.g6lz11tv.cn/315755.htm
qer.g6lz11tv.cn/559775.htm
qer.g6lz11tv.cn/111575.htm
qer.g6lz11tv.cn/953335.htm
qer.g6lz11tv.cn/739315.htm
qer.g6lz11tv.cn/535515.htm
qer.g6lz11tv.cn/913575.htm
qer.g6lz11tv.cn/599355.htm
qer.g6lz11tv.cn/975955.htm
qer.g6lz11tv.cn/373395.htm
qer.g6lz11tv.cn/591535.htm
qee.g6lz11tv.cn/333915.htm
qee.g6lz11tv.cn/757395.htm
qee.g6lz11tv.cn/519515.htm
qee.g6lz11tv.cn/111935.htm
qee.g6lz11tv.cn/111515.htm
qee.g6lz11tv.cn/177115.htm
qee.g6lz11tv.cn/331515.htm
qee.g6lz11tv.cn/113775.htm
qee.g6lz11tv.cn/319375.htm
qee.g6lz11tv.cn/537915.htm
qew.g6lz11tv.cn/595575.htm
qew.g6lz11tv.cn/559395.htm
qew.g6lz11tv.cn/959915.htm
qew.g6lz11tv.cn/991115.htm
qew.g6lz11tv.cn/393575.htm
qew.g6lz11tv.cn/131915.htm
qew.g6lz11tv.cn/779575.htm
qew.g6lz11tv.cn/939375.htm
qew.g6lz11tv.cn/759535.htm
qew.g6lz11tv.cn/359915.htm
qeq.g6lz11tv.cn/951155.htm
qeq.g6lz11tv.cn/753555.htm
qeq.g6lz11tv.cn/537535.htm
qeq.g6lz11tv.cn/159315.htm
qeq.g6lz11tv.cn/575535.htm
qeq.g6lz11tv.cn/377775.htm
qeq.g6lz11tv.cn/553915.htm
qeq.g6lz11tv.cn/117715.htm
qeq.g6lz11tv.cn/331515.htm
qeq.g6lz11tv.cn/999975.htm
qwtv.g6lz11tv.cn/113135.htm
qwtv.g6lz11tv.cn/373195.htm
qwtv.g6lz11tv.cn/739575.htm
qwtv.g6lz11tv.cn/331155.htm
qwtv.g6lz11tv.cn/119715.htm
qwtv.g6lz11tv.cn/991595.htm
qwtv.g6lz11tv.cn/793715.htm
qwtv.g6lz11tv.cn/395515.htm
qwtv.g6lz11tv.cn/911995.htm
qwtv.g6lz11tv.cn/959955.htm
qwn.g6lz11tv.cn/739515.htm
qwn.g6lz11tv.cn/311915.htm
qwn.g6lz11tv.cn/919375.htm
qwn.g6lz11tv.cn/917775.htm
qwn.g6lz11tv.cn/739155.htm
qwn.g6lz11tv.cn/579375.htm
qwn.g6lz11tv.cn/397975.htm
qwn.g6lz11tv.cn/551535.htm
qwn.g6lz11tv.cn/979935.htm
qwn.g6lz11tv.cn/553175.htm
qwb.g6lz11tv.cn/577515.htm
qwb.g6lz11tv.cn/771335.htm
qwb.g6lz11tv.cn/915955.htm
qwb.g6lz11tv.cn/155355.htm
qwb.g6lz11tv.cn/135535.htm
qwb.g6lz11tv.cn/713395.htm
qwb.g6lz11tv.cn/119715.htm
qwb.g6lz11tv.cn/313315.htm
qwb.g6lz11tv.cn/599595.htm
qwb.g6lz11tv.cn/993535.htm
qwv.g6lz11tv.cn/959535.htm
qwv.g6lz11tv.cn/795575.htm
qwv.g6lz11tv.cn/159315.htm
qwv.g6lz11tv.cn/137555.htm
qwv.g6lz11tv.cn/713315.htm
qwv.g6lz11tv.cn/597315.htm
qwv.g6lz11tv.cn/797775.htm
qwv.g6lz11tv.cn/519995.htm
qwv.g6lz11tv.cn/951795.htm
qwv.g6lz11tv.cn/197775.htm
wc.g6lz11tv.cn/737555.htm
wc.g6lz11tv.cn/397175.htm
wc.g6lz11tv.cn/355775.htm
wc.g6lz11tv.cn/933195.htm
wc.g6lz11tv.cn/559595.htm
wc.g6lz11tv.cn/171175.htm
wc.g6lz11tv.cn/995975.htm
