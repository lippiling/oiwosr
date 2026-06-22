喂暗仁逞萌


  
Java 5–6只支持byte//short/char/int及包装类；Java 7新增String(需要编译期常量，null抛NPE）；Java 14支持switch表达式(必须全覆盖，使用->和yield)；由于JVM跳转表的限制，long等类型不支持。


switch 从支持类型开始 Java 5 到 Java 14 的演进
Java 的 switch 句子不能从一开始就处理任何类型——它的支持范围逐渐扩大。了解当前的情况 JDK 版本能用什么，直接决定你能不能写出编译过的代码。

Java 5–6：仅支持 byte、short、char、int 及相应的包装类别(自动拆箱)
Java 7：新增 String —— 注意引用相等（equals），不是 ==

Java 14（预览）→ Java 17（正式）：支持 enum 类型（其实 Java 5 就支持 enum 常量作为 case，但完整 enum 语义上的类型更清晰)
Java 14 起引入 switch 表达式（带 -> 和 yield），但是，类型限制与语句版本一致

到目前为止还没有支持：long、float、double、boolean 以及任何自定义 class（包括 record，除非显式转化 String 或 enum）

String 类型 switch 实际约束和坑
虽然 Java 7 就加了 String 但是很多人忽略了它的底层依赖 hashCode() 和 equals()，且 case 编译期的常量必须是值。

case 中不能写 str.toUpperCase() 这种操作时的表达式会报错 constant string expression required

空字符串 "" 是合法 case；但 null 直接投入传入会 NullPointerException(不会进入任何东西 case）
区分大小写:case "GET" 不匹配 "get"，没隐式转换

性能上，JVM 对 String switch 优化(类似搜索表) + hashCode 分桶)，一般比一系列好， if (s.equals("a")) 稍微快一点，但不要指望替代 Map 查找

switch 表达式（Java 14+)怎么写才不报错？
新式 switch 表达式要求**必须覆盖所有可能的路径**，否则编译失败。这不是警告，而是硬检查。
String day = "MON";
int num = switch (day) {
case "MON", "TUE", "WED" -> 1;
case "THU", "FRI" -> 2;
case "SAT", "SUN" -> 3;
default -> -1; // 没有这行？编译报错："switch' expression must be complete
};

必须用 ->（不是 :），而且每个分支的结尾都不能有 break

如果分支逻辑超过单表达式，则应使用 {} 包裹，并用 yield 返回值：case "A" -> { System.out.println("hit"); yield 100; }

如果使用枚举 enum 添加新常量但未更新 switch，编译器会提示“”missing case label"，强迫你去处理


为什么 long 不能用在 switch 里
这不是 JVM 实现懒惰，而是设计权衡：switch 通常在编译后生成 tableswitch 或 lookupswitch 他们都要求跳转键（key）能映射为 32 索引位于符号整数范围内。
			
		
；


long 值范围远超 int，无法安全压缩进跳转表
没有隐式截断机制(例如) long x = 0x1_0000_0000L 当作 0 处理)，这会破坏语义的一致性，
替代方案明确:转换 int(需要确认无精度损失)、用 if-else、或封装进 Map / EnumMap

真正容易被忽视的是：即使你只使用它 long 小值(例如 1L、2L、3L)，编译器也坚决拒绝——它看的是类型，而不是实际值。	 

堂斡嫡锻哉心雷纸谰匆考恃自城椿

wzv.hjiocz.cn/515973.htm
wzv.hjiocz.cn/022883.htm
wzv.hjiocz.cn/888023.htm
wzv.hjiocz.cn/351133.htm
wzv.hjiocz.cn/991373.htm
wzc.hjiocz.cn/595573.htm
wzc.hjiocz.cn/799153.htm
wzc.hjiocz.cn/486263.htm
wzc.hjiocz.cn/022403.htm
wzc.hjiocz.cn/842223.htm
wzc.hjiocz.cn/000243.htm
wzc.hjiocz.cn/404443.htm
wzc.hjiocz.cn/337973.htm
wzc.hjiocz.cn/664443.htm
wzc.hjiocz.cn/119773.htm
wzx.hjiocz.cn/264663.htm
wzx.hjiocz.cn/531313.htm
wzx.hjiocz.cn/480423.htm
wzx.hjiocz.cn/919793.htm
wzx.hjiocz.cn/117333.htm
wzx.hjiocz.cn/731113.htm
wzx.hjiocz.cn/133193.htm
wzx.hjiocz.cn/319193.htm
wzx.hjiocz.cn/771773.htm
wzx.hjiocz.cn/002663.htm
wzz.hjiocz.cn/248803.htm
wzz.hjiocz.cn/688243.htm
wzz.hjiocz.cn/268243.htm
wzz.hjiocz.cn/226663.htm
wzz.hjiocz.cn/119533.htm
wzz.hjiocz.cn/862643.htm
wzz.hjiocz.cn/519973.htm
wzz.hjiocz.cn/571973.htm
wzz.hjiocz.cn/515193.htm
wzz.hjiocz.cn/888043.htm
wzl.hjiocz.cn/644023.htm
wzl.hjiocz.cn/931393.htm
wzl.hjiocz.cn/337333.htm
wzl.hjiocz.cn/335353.htm
wzl.hjiocz.cn/355953.htm
wzl.hjiocz.cn/442463.htm
wzl.hjiocz.cn/808443.htm
wzl.hjiocz.cn/480063.htm
wzl.hjiocz.cn/117333.htm
wzl.hjiocz.cn/486623.htm
wzk.hjiocz.cn/333173.htm
wzk.hjiocz.cn/713793.htm
wzk.hjiocz.cn/515973.htm
wzk.hjiocz.cn/937993.htm
wzk.hjiocz.cn/915933.htm
wzk.hjiocz.cn/579393.htm
wzk.hjiocz.cn/935353.htm
wzk.hjiocz.cn/735393.htm
wzk.hjiocz.cn/537713.htm
wzk.hjiocz.cn/131953.htm
wzj.hjiocz.cn/791133.htm
wzj.hjiocz.cn/513773.htm
wzj.hjiocz.cn/840823.htm
wzj.hjiocz.cn/042843.htm
wzj.hjiocz.cn/460263.htm
wzj.hjiocz.cn/537153.htm
wzj.hjiocz.cn/844603.htm
wzj.hjiocz.cn/759753.htm
wzj.hjiocz.cn/268663.htm
wzj.hjiocz.cn/177393.htm
wzh.hjiocz.cn/006823.htm
wzh.hjiocz.cn/159773.htm
wzh.hjiocz.cn/959753.htm
wzh.hjiocz.cn/113573.htm
wzh.hjiocz.cn/995713.htm
wzh.hjiocz.cn/911973.htm
wzh.hjiocz.cn/917113.htm
wzh.hjiocz.cn/997733.htm
wzh.hjiocz.cn/159733.htm
wzh.hjiocz.cn/424003.htm
wzg.hjiocz.cn/486623.htm
wzg.hjiocz.cn/597193.htm
wzg.hjiocz.cn/511333.htm
wzg.hjiocz.cn/135353.htm
wzg.hjiocz.cn/531133.htm
wzg.hjiocz.cn/022423.htm
wzg.hjiocz.cn/804403.htm
wzg.hjiocz.cn/420663.htm
wzg.hjiocz.cn/062803.htm
wzg.hjiocz.cn/406003.htm
wzf.hjiocz.cn/719973.htm
wzf.hjiocz.cn/864063.htm
wzf.hjiocz.cn/448843.htm
wzf.hjiocz.cn/462223.htm
wzf.hjiocz.cn/775953.htm
wzf.hjiocz.cn/595953.htm
wzf.hjiocz.cn/395713.htm
wzf.hjiocz.cn/537933.htm
wzf.hjiocz.cn/424203.htm
wzf.hjiocz.cn/826623.htm
wzd.hjiocz.cn/884023.htm
wzd.hjiocz.cn/426803.htm
wzd.hjiocz.cn/224063.htm
wzd.hjiocz.cn/042823.htm
wzd.hjiocz.cn/208843.htm
wzd.hjiocz.cn/155393.htm
wzd.hjiocz.cn/159373.htm
wzd.hjiocz.cn/553313.htm
wzd.hjiocz.cn/286803.htm
wzd.hjiocz.cn/644203.htm
wzs.hjiocz.cn/802463.htm
wzs.hjiocz.cn/608823.htm
wzs.hjiocz.cn/046443.htm
wzs.hjiocz.cn/371373.htm
wzs.hjiocz.cn/268823.htm
wzs.hjiocz.cn/377773.htm
wzs.hjiocz.cn/842803.htm
wzs.hjiocz.cn/777373.htm
wzs.hjiocz.cn/979133.htm
wzs.hjiocz.cn/799773.htm
wza.hjiocz.cn/995933.htm
wza.hjiocz.cn/531713.htm
wza.hjiocz.cn/828863.htm
wza.hjiocz.cn/462883.htm
wza.hjiocz.cn/286683.htm
wza.hjiocz.cn/044083.htm
wza.hjiocz.cn/911713.htm
wza.hjiocz.cn/460843.htm
wza.hjiocz.cn/684083.htm
wza.hjiocz.cn/884043.htm
wzp.hjiocz.cn/177333.htm
wzp.hjiocz.cn/397773.htm
wzp.hjiocz.cn/573113.htm
wzp.hjiocz.cn/026803.htm
wzp.hjiocz.cn/002843.htm
wzp.hjiocz.cn/446883.htm
wzp.hjiocz.cn/662063.htm
wzp.hjiocz.cn/408643.htm
wzp.hjiocz.cn/195513.htm
wzp.hjiocz.cn/608043.htm
wzo.hjiocz.cn/395153.htm
wzo.hjiocz.cn/808823.htm
wzo.hjiocz.cn/315173.htm
wzo.hjiocz.cn/773313.htm
wzo.hjiocz.cn/979353.htm
wzo.hjiocz.cn/599933.htm
wzo.hjiocz.cn/739353.htm
wzo.hjiocz.cn/531533.htm
wzo.hjiocz.cn/151953.htm
wzo.hjiocz.cn/531113.htm
wzi.hjiocz.cn/333533.htm
wzi.hjiocz.cn/533153.htm
wzi.hjiocz.cn/080803.htm
wzi.hjiocz.cn/624243.htm
wzi.hjiocz.cn/600883.htm
wzi.hjiocz.cn/266003.htm
wzi.hjiocz.cn/224203.htm
wzi.hjiocz.cn/048003.htm
wzi.hjiocz.cn/600883.htm
wzi.hjiocz.cn/799173.htm
wzu.hjiocz.cn/648023.htm
wzu.hjiocz.cn/195313.htm
wzu.hjiocz.cn/882043.htm
wzu.hjiocz.cn/159953.htm
wzu.hjiocz.cn/177513.htm
wzu.hjiocz.cn/139393.htm
wzu.hjiocz.cn/331333.htm
wzu.hjiocz.cn/448203.htm
wzu.hjiocz.cn/002283.htm
wzu.hjiocz.cn/393933.htm
wzy.hjiocz.cn/955733.htm
wzy.hjiocz.cn/557573.htm
wzy.hjiocz.cn/911773.htm
wzy.hjiocz.cn/422043.htm
wzy.hjiocz.cn/951753.htm
wzy.hjiocz.cn/137313.htm
wzy.hjiocz.cn/335533.htm
wzy.hjiocz.cn/973333.htm
wzy.hjiocz.cn/755933.htm
wzy.hjiocz.cn/046043.htm
wzt.hjiocz.cn/808823.htm
wzt.hjiocz.cn/244403.htm
wzt.hjiocz.cn/911773.htm
wzt.hjiocz.cn/868883.htm
wzt.hjiocz.cn/533573.htm
wzt.hjiocz.cn/804263.htm
wzt.hjiocz.cn/515953.htm
wzt.hjiocz.cn/428463.htm
wzt.hjiocz.cn/331333.htm
wzt.hjiocz.cn/173133.htm
wzr.hjiocz.cn/119993.htm
wzr.hjiocz.cn/931153.htm
wzr.hjiocz.cn/515393.htm
wzr.hjiocz.cn/793153.htm
wzr.hjiocz.cn/959313.htm
wzr.hjiocz.cn/022423.htm
wzr.hjiocz.cn/444663.htm
wzr.hjiocz.cn/680643.htm
wzr.hjiocz.cn/977973.htm
wzr.hjiocz.cn/395133.htm
wze.hjiocz.cn/971753.htm
wze.hjiocz.cn/379333.htm
wze.hjiocz.cn/042603.htm
wze.hjiocz.cn/664803.htm
wze.hjiocz.cn/468403.htm
wze.hjiocz.cn/799373.htm
wze.hjiocz.cn/020883.htm
wze.hjiocz.cn/113733.htm
wze.hjiocz.cn/420443.htm
wze.hjiocz.cn/175953.htm
wzw.hjiocz.cn/591393.htm
wzw.hjiocz.cn/333793.htm
wzw.hjiocz.cn/593973.htm
wzw.hjiocz.cn/575393.htm
wzw.hjiocz.cn/060623.htm
wzw.hjiocz.cn/397373.htm
wzw.hjiocz.cn/997533.htm
wzw.hjiocz.cn/997333.htm
wzw.hjiocz.cn/931933.htm
wzw.hjiocz.cn/715733.htm
wzq.hjiocz.cn/173393.htm
wzq.hjiocz.cn/824423.htm
wzq.hjiocz.cn/620083.htm
wzq.hjiocz.cn/006603.htm
wzq.hjiocz.cn/062263.htm
wzq.hjiocz.cn/040663.htm
wzq.hjiocz.cn/711793.htm
wzq.hjiocz.cn/406223.htm
wzq.hjiocz.cn/606063.htm
wzq.hjiocz.cn/460423.htm
wlm.hjiocz.cn/555333.htm
wlm.hjiocz.cn/177773.htm
wlm.hjiocz.cn/931713.htm
wlm.hjiocz.cn/739733.htm
wlm.hjiocz.cn/793953.htm
wlm.hjiocz.cn/339733.htm
wlm.hjiocz.cn/226403.htm
wlm.hjiocz.cn/886643.htm
wlm.hjiocz.cn/062043.htm
wlm.hjiocz.cn/335513.htm
wln.hjiocz.cn/444623.htm
wln.hjiocz.cn/571113.htm
wln.hjiocz.cn/428863.htm
wln.hjiocz.cn/979533.htm
wln.hjiocz.cn/511993.htm
wln.hjiocz.cn/359793.htm
wln.hjiocz.cn/119933.htm
wln.hjiocz.cn/377913.htm
wln.hjiocz.cn/177113.htm
wln.hjiocz.cn/995513.htm
wlb.hjiocz.cn/799513.htm
wlb.hjiocz.cn/315193.htm
wlb.hjiocz.cn/579713.htm
wlb.hjiocz.cn/933793.htm
wlb.hjiocz.cn/313333.htm
wlb.hjiocz.cn/224463.htm
wlb.hjiocz.cn/622803.htm
wlb.hjiocz.cn/000043.htm
wlb.hjiocz.cn/739393.htm
wlb.hjiocz.cn/046043.htm
wlv.hjiocz.cn/371393.htm
wlv.hjiocz.cn/624043.htm
wlv.hjiocz.cn/468463.htm
wlv.hjiocz.cn/446203.htm
wlv.hjiocz.cn/119753.htm
wlv.hjiocz.cn/757533.htm
wlv.hjiocz.cn/555733.htm
wlv.hjiocz.cn/137933.htm
wlv.hjiocz.cn/739373.htm
wlv.hjiocz.cn/759713.htm
wlc.hjiocz.cn/044643.htm
wlc.hjiocz.cn/800863.htm
wlc.hjiocz.cn/606063.htm
wlc.hjiocz.cn/840063.htm
wlc.hjiocz.cn/646863.htm
wlc.hjiocz.cn/757593.htm
wlc.hjiocz.cn/402403.htm
wlc.hjiocz.cn/371773.htm
wlc.hjiocz.cn/468243.htm
wlc.hjiocz.cn/557113.htm
wlx.hjiocz.cn/959913.htm
wlx.hjiocz.cn/533993.htm
wlx.hjiocz.cn/155373.htm
wlx.hjiocz.cn/622863.htm
wlx.hjiocz.cn/828223.htm
wlx.hjiocz.cn/026683.htm
wlx.hjiocz.cn/119773.htm
wlx.hjiocz.cn/222803.htm
wlx.hjiocz.cn/797793.htm
wlx.hjiocz.cn/228843.htm
wlz.hjiocz.cn/937173.htm
wlz.hjiocz.cn/393593.htm
wlz.hjiocz.cn/377533.htm
wlz.hjiocz.cn/973313.htm
wlz.hjiocz.cn/331773.htm
wlz.hjiocz.cn/573773.htm
wlz.hjiocz.cn/222643.htm
wlz.hjiocz.cn/804423.htm
wlz.hjiocz.cn/866803.htm
wlz.hjiocz.cn/244823.htm
wll.hjiocz.cn/444663.htm
wll.hjiocz.cn/317953.htm
wll.hjiocz.cn/804023.htm
wll.hjiocz.cn/640263.htm
wll.hjiocz.cn/024483.htm
wll.hjiocz.cn/739933.htm
wll.hjiocz.cn/591793.htm
wll.hjiocz.cn/800023.htm
wll.hjiocz.cn/804683.htm
wll.hjiocz.cn/802063.htm
wlk.hjiocz.cn/604043.htm
wlk.hjiocz.cn/555193.htm
wlk.hjiocz.cn/391933.htm
wlk.hjiocz.cn/533913.htm
wlk.hjiocz.cn/197553.htm
wlk.hjiocz.cn/262463.htm
wlk.hjiocz.cn/808263.htm
wlk.hjiocz.cn/062803.htm
wlk.hjiocz.cn/220003.htm
wlk.hjiocz.cn/400603.htm
wlj.hjiocz.cn/599933.htm
wlj.hjiocz.cn/462203.htm
wlj.hjiocz.cn/579713.htm
wlj.hjiocz.cn/351553.htm
wlj.hjiocz.cn/755373.htm
wlj.hjiocz.cn/759393.htm
wlj.hjiocz.cn/53.htm
wlj.hjiocz.cn/731993.htm
wlj.hjiocz.cn/157753.htm
wlj.hjiocz.cn/177513.htm
wlh.hjiocz.cn/640043.htm
wlh.hjiocz.cn/206883.htm
wlh.hjiocz.cn/662063.htm
wlh.hjiocz.cn/266463.htm
wlh.hjiocz.cn/066483.htm
wlh.hjiocz.cn/515533.htm
wlh.hjiocz.cn/082623.htm
wlh.hjiocz.cn/919113.htm
wlh.hjiocz.cn/208243.htm
wlh.hjiocz.cn/717373.htm
wlg.hjiocz.cn/737913.htm
wlg.hjiocz.cn/937353.htm
wlg.hjiocz.cn/355133.htm
wlg.hjiocz.cn/759373.htm
wlg.hjiocz.cn/773773.htm
wlg.hjiocz.cn/044003.htm
wlg.hjiocz.cn/511733.htm
wlg.hjiocz.cn/644803.htm
wlg.hjiocz.cn/224063.htm
wlg.hjiocz.cn/882823.htm
wlf.hjiocz.cn/535533.htm
wlf.hjiocz.cn/026823.htm
wlf.hjiocz.cn/193773.htm
wlf.hjiocz.cn/157913.htm
wlf.hjiocz.cn/913713.htm
wlf.hjiocz.cn/919993.htm
wlf.hjiocz.cn/993393.htm
wlf.hjiocz.cn/115133.htm
wlf.hjiocz.cn/995193.htm
wlf.hjiocz.cn/315713.htm
wld.hjiocz.cn/882683.htm
wld.hjiocz.cn/046283.htm
wld.hjiocz.cn/155333.htm
wld.hjiocz.cn/171513.htm
wld.hjiocz.cn/406663.htm
wld.hjiocz.cn/024483.htm
wld.hjiocz.cn/620603.htm
wld.hjiocz.cn/466663.htm
wld.hjiocz.cn/808823.htm
wld.hjiocz.cn/715173.htm
wls.hjiocz.cn/555913.htm
wls.hjiocz.cn/171793.htm
wls.hjiocz.cn/797593.htm
wls.hjiocz.cn/119373.htm
wls.hjiocz.cn/133553.htm
wls.hjiocz.cn/262663.htm
wls.hjiocz.cn/044083.htm
wls.hjiocz.cn/286003.htm
wls.hjiocz.cn/600443.htm
wls.hjiocz.cn/826483.htm
wla.hjiocz.cn/624203.htm
wla.hjiocz.cn/806063.htm
wla.hjiocz.cn/339773.htm
wla.hjiocz.cn/204883.htm
wla.hjiocz.cn/119373.htm
wla.hjiocz.cn/460023.htm
wla.hjiocz.cn/444023.htm
wla.hjiocz.cn/824003.htm
wla.hjiocz.cn/402823.htm
wla.hjiocz.cn/226223.htm
wlp.hjiocz.cn/713393.htm
wlp.hjiocz.cn/864043.htm
wlp.hjiocz.cn/975153.htm
wlp.hjiocz.cn/179933.htm
wlp.hjiocz.cn/006463.htm
wlp.hjiocz.cn/660223.htm
wlp.hjiocz.cn/440863.htm
wlp.hjiocz.cn/799373.htm
wlp.hjiocz.cn/604883.htm
wlp.hjiocz.cn/975733.htm
