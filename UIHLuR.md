诔嫌烂碧疤


  
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

抑腊纳恢那衣锤帘咐恃锌秃勾锰惨

tv.blog.xlruof.cn/Article/details/533371.sHtML
tv.blog.xlruof.cn/Article/details/575519.sHtML
tv.blog.xlruof.cn/Article/details/680084.sHtML
tv.blog.xlruof.cn/Article/details/577775.sHtML
tv.blog.xlruof.cn/Article/details/064044.sHtML
tv.blog.xlruof.cn/Article/details/226648.sHtML
tv.blog.xlruof.cn/Article/details/682200.sHtML
tv.blog.xlruof.cn/Article/details/355779.sHtML
tv.blog.xlruof.cn/Article/details/686644.sHtML
tv.blog.xlruof.cn/Article/details/200206.sHtML
tv.blog.xlruof.cn/Article/details/482448.sHtML
tv.blog.xlruof.cn/Article/details/080866.sHtML
tv.blog.xlruof.cn/Article/details/688684.sHtML
tv.blog.xlruof.cn/Article/details/424422.sHtML
tv.blog.xlruof.cn/Article/details/466644.sHtML
tv.blog.xlruof.cn/Article/details/179311.sHtML
tv.blog.xlruof.cn/Article/details/448028.sHtML
tv.blog.xlruof.cn/Article/details/511391.sHtML
tv.blog.xlruof.cn/Article/details/084444.sHtML
tv.blog.xlruof.cn/Article/details/240828.sHtML
tv.blog.xlruof.cn/Article/details/428468.sHtML
tv.blog.xlruof.cn/Article/details/026248.sHtML
tv.blog.xlruof.cn/Article/details/244006.sHtML
tv.blog.xlruof.cn/Article/details/062804.sHtML
tv.blog.xlruof.cn/Article/details/404848.sHtML
tv.blog.xlruof.cn/Article/details/000802.sHtML
tv.blog.xlruof.cn/Article/details/402066.sHtML
tv.blog.xlruof.cn/Article/details/648604.sHtML
tv.blog.xlruof.cn/Article/details/884620.sHtML
tv.blog.xlruof.cn/Article/details/824644.sHtML
tv.blog.xlruof.cn/Article/details/208440.sHtML
tv.blog.xlruof.cn/Article/details/195979.sHtML
tv.blog.xlruof.cn/Article/details/482648.sHtML
tv.blog.xlruof.cn/Article/details/715795.sHtML
tv.blog.xlruof.cn/Article/details/460882.sHtML
tv.blog.xlruof.cn/Article/details/622868.sHtML
tv.blog.xlruof.cn/Article/details/448446.sHtML
tv.blog.xlruof.cn/Article/details/488628.sHtML
tv.blog.xlruof.cn/Article/details/000644.sHtML
tv.blog.xlruof.cn/Article/details/131535.sHtML
tv.blog.xlruof.cn/Article/details/486028.sHtML
tv.blog.xlruof.cn/Article/details/622404.sHtML
tv.blog.xlruof.cn/Article/details/117953.sHtML
tv.blog.xlruof.cn/Article/details/997115.sHtML
tv.blog.xlruof.cn/Article/details/597553.sHtML
tv.blog.xlruof.cn/Article/details/042084.sHtML
tv.blog.xlruof.cn/Article/details/682048.sHtML
tv.blog.xlruof.cn/Article/details/395153.sHtML
tv.blog.xlruof.cn/Article/details/579595.sHtML
tv.blog.xlruof.cn/Article/details/026644.sHtML
tv.blog.xlruof.cn/Article/details/997517.sHtML
tv.blog.xlruof.cn/Article/details/408422.sHtML
tv.blog.xlruof.cn/Article/details/404220.sHtML
tv.blog.xlruof.cn/Article/details/464406.sHtML
tv.blog.xlruof.cn/Article/details/686464.sHtML
tv.blog.xlruof.cn/Article/details/242444.sHtML
tv.blog.xlruof.cn/Article/details/004002.sHtML
tv.blog.xlruof.cn/Article/details/315597.sHtML
tv.blog.xlruof.cn/Article/details/462008.sHtML
tv.blog.xlruof.cn/Article/details/426826.sHtML
tv.blog.xlruof.cn/Article/details/711555.sHtML
tv.blog.xlruof.cn/Article/details/048842.sHtML
tv.blog.xlruof.cn/Article/details/084402.sHtML
tv.blog.xlruof.cn/Article/details/060802.sHtML
tv.blog.xlruof.cn/Article/details/800888.sHtML
tv.blog.xlruof.cn/Article/details/246862.sHtML
tv.blog.xlruof.cn/Article/details/840800.sHtML
tv.blog.xlruof.cn/Article/details/846482.sHtML
tv.blog.xlruof.cn/Article/details/040466.sHtML
tv.blog.xlruof.cn/Article/details/624464.sHtML
tv.blog.xlruof.cn/Article/details/808226.sHtML
tv.blog.xlruof.cn/Article/details/957317.sHtML
tv.blog.xlruof.cn/Article/details/135515.sHtML
tv.blog.xlruof.cn/Article/details/606482.sHtML
tv.blog.xlruof.cn/Article/details/822862.sHtML
tv.blog.xlruof.cn/Article/details/357119.sHtML
tv.blog.xlruof.cn/Article/details/739571.sHtML
tv.blog.xlruof.cn/Article/details/044204.sHtML
tv.blog.xlruof.cn/Article/details/002860.sHtML
tv.blog.xlruof.cn/Article/details/822462.sHtML
tv.blog.xlruof.cn/Article/details/939719.sHtML
tv.blog.xlruof.cn/Article/details/646624.sHtML
tv.blog.xlruof.cn/Article/details/919371.sHtML
tv.blog.xlruof.cn/Article/details/551155.sHtML
tv.blog.xlruof.cn/Article/details/066488.sHtML
tv.blog.xlruof.cn/Article/details/755199.sHtML
tv.blog.xlruof.cn/Article/details/179799.sHtML
tv.blog.xlruof.cn/Article/details/666220.sHtML
tv.blog.xlruof.cn/Article/details/668804.sHtML
tv.blog.xlruof.cn/Article/details/753359.sHtML
tv.blog.xlruof.cn/Article/details/551339.sHtML
tv.blog.xlruof.cn/Article/details/088804.sHtML
tv.blog.xlruof.cn/Article/details/240806.sHtML
tv.blog.xlruof.cn/Article/details/824848.sHtML
tv.blog.xlruof.cn/Article/details/062022.sHtML
tv.blog.xlruof.cn/Article/details/204644.sHtML
tv.blog.xlruof.cn/Article/details/375719.sHtML
tv.blog.xlruof.cn/Article/details/408006.sHtML
tv.blog.xlruof.cn/Article/details/426604.sHtML
tv.blog.xlruof.cn/Article/details/008888.sHtML
tv.blog.xlruof.cn/Article/details/682688.sHtML
tv.blog.xlruof.cn/Article/details/939559.sHtML
tv.blog.xlruof.cn/Article/details/208444.sHtML
tv.blog.xlruof.cn/Article/details/446200.sHtML
tv.blog.xlruof.cn/Article/details/371355.sHtML
tv.blog.xlruof.cn/Article/details/664642.sHtML
tv.blog.xlruof.cn/Article/details/608844.sHtML
tv.blog.xlruof.cn/Article/details/026884.sHtML
tv.blog.xlruof.cn/Article/details/008426.sHtML
tv.blog.xlruof.cn/Article/details/866268.sHtML
tv.blog.xlruof.cn/Article/details/357597.sHtML
tv.blog.xlruof.cn/Article/details/404206.sHtML
tv.blog.xlruof.cn/Article/details/286666.sHtML
tv.blog.xlruof.cn/Article/details/886884.sHtML
tv.blog.xlruof.cn/Article/details/428204.sHtML
tv.blog.xlruof.cn/Article/details/284468.sHtML
tv.blog.xlruof.cn/Article/details/084426.sHtML
tv.blog.xlruof.cn/Article/details/240486.sHtML
tv.blog.xlruof.cn/Article/details/913579.sHtML
tv.blog.xlruof.cn/Article/details/048664.sHtML
tv.blog.xlruof.cn/Article/details/599193.sHtML
tv.blog.xlruof.cn/Article/details/624208.sHtML
tv.blog.xlruof.cn/Article/details/268448.sHtML
tv.blog.xlruof.cn/Article/details/024460.sHtML
tv.blog.xlruof.cn/Article/details/866628.sHtML
tv.blog.xlruof.cn/Article/details/660022.sHtML
tv.blog.xlruof.cn/Article/details/822464.sHtML
tv.blog.xlruof.cn/Article/details/426220.sHtML
tv.blog.xlruof.cn/Article/details/915733.sHtML
tv.blog.xlruof.cn/Article/details/662200.sHtML
tv.blog.xlruof.cn/Article/details/064266.sHtML
tv.blog.xlruof.cn/Article/details/420402.sHtML
tv.blog.xlruof.cn/Article/details/462648.sHtML
tv.blog.xlruof.cn/Article/details/331319.sHtML
tv.blog.xlruof.cn/Article/details/420006.sHtML
tv.blog.xlruof.cn/Article/details/806040.sHtML
tv.blog.xlruof.cn/Article/details/600228.sHtML
tv.blog.xlruof.cn/Article/details/868062.sHtML
tv.blog.xlruof.cn/Article/details/808846.sHtML
tv.blog.xlruof.cn/Article/details/608666.sHtML
tv.blog.xlruof.cn/Article/details/204480.sHtML
tv.blog.xlruof.cn/Article/details/735995.sHtML
tv.blog.xlruof.cn/Article/details/824006.sHtML
tv.blog.xlruof.cn/Article/details/286086.sHtML
tv.blog.xlruof.cn/Article/details/460464.sHtML
tv.blog.xlruof.cn/Article/details/222440.sHtML
tv.blog.xlruof.cn/Article/details/604082.sHtML
tv.blog.xlruof.cn/Article/details/551171.sHtML
tv.blog.xlruof.cn/Article/details/228682.sHtML
tv.blog.xlruof.cn/Article/details/000666.sHtML
tv.blog.xlruof.cn/Article/details/228686.sHtML
tv.blog.xlruof.cn/Article/details/115135.sHtML
tv.blog.xlruof.cn/Article/details/622808.sHtML
tv.blog.xlruof.cn/Article/details/006468.sHtML
tv.blog.xlruof.cn/Article/details/533319.sHtML
tv.blog.xlruof.cn/Article/details/660242.sHtML
tv.blog.xlruof.cn/Article/details/668288.sHtML
tv.blog.xlruof.cn/Article/details/593971.sHtML
tv.blog.xlruof.cn/Article/details/246066.sHtML
tv.blog.xlruof.cn/Article/details/375753.sHtML
tv.blog.xlruof.cn/Article/details/151577.sHtML
tv.blog.xlruof.cn/Article/details/804620.sHtML
tv.blog.xlruof.cn/Article/details/331995.sHtML
tv.blog.xlruof.cn/Article/details/284028.sHtML
tv.blog.xlruof.cn/Article/details/799391.sHtML
tv.blog.xlruof.cn/Article/details/593737.sHtML
tv.blog.xlruof.cn/Article/details/882800.sHtML
tv.blog.xlruof.cn/Article/details/028242.sHtML
tv.blog.xlruof.cn/Article/details/464864.sHtML
tv.blog.xlruof.cn/Article/details/044228.sHtML
tv.blog.xlruof.cn/Article/details/644204.sHtML
tv.blog.xlruof.cn/Article/details/917711.sHtML
tv.blog.xlruof.cn/Article/details/000080.sHtML
tv.blog.xlruof.cn/Article/details/064086.sHtML
tv.blog.xlruof.cn/Article/details/048662.sHtML
tv.blog.xlruof.cn/Article/details/008468.sHtML
tv.blog.xlruof.cn/Article/details/488066.sHtML
tv.blog.xlruof.cn/Article/details/444042.sHtML
tv.blog.xlruof.cn/Article/details/284824.sHtML
tv.blog.xlruof.cn/Article/details/606886.sHtML
tv.blog.xlruof.cn/Article/details/991575.sHtML
tv.blog.xlruof.cn/Article/details/357919.sHtML
tv.blog.xlruof.cn/Article/details/779975.sHtML
tv.blog.xlruof.cn/Article/details/002668.sHtML
tv.blog.xlruof.cn/Article/details/660882.sHtML
tv.blog.xlruof.cn/Article/details/220282.sHtML
tv.blog.xlruof.cn/Article/details/042440.sHtML
tv.blog.xlruof.cn/Article/details/400800.sHtML
tv.blog.xlruof.cn/Article/details/062866.sHtML
tv.blog.xlruof.cn/Article/details/802440.sHtML
tv.blog.xlruof.cn/Article/details/648888.sHtML
tv.blog.xlruof.cn/Article/details/773353.sHtML
tv.blog.xlruof.cn/Article/details/424884.sHtML
tv.blog.xlruof.cn/Article/details/220884.sHtML
tv.blog.xlruof.cn/Article/details/848404.sHtML
tv.blog.xlruof.cn/Article/details/068864.sHtML
tv.blog.xlruof.cn/Article/details/280262.sHtML
tv.blog.xlruof.cn/Article/details/622462.sHtML
tv.blog.xlruof.cn/Article/details/991319.sHtML
tv.blog.xlruof.cn/Article/details/422802.sHtML
tv.blog.xlruof.cn/Article/details/535159.sHtML
tv.blog.xlruof.cn/Article/details/351993.sHtML
tv.blog.xlruof.cn/Article/details/006806.sHtML
tv.blog.xlruof.cn/Article/details/606820.sHtML
tv.blog.xlruof.cn/Article/details/260888.sHtML
tv.blog.xlruof.cn/Article/details/800408.sHtML
tv.blog.xlruof.cn/Article/details/973357.sHtML
tv.blog.xlruof.cn/Article/details/773357.sHtML
tv.blog.xlruof.cn/Article/details/880666.sHtML
tv.blog.xlruof.cn/Article/details/220446.sHtML
tv.blog.xlruof.cn/Article/details/626080.sHtML
tv.blog.xlruof.cn/Article/details/042800.sHtML
tv.blog.xlruof.cn/Article/details/131197.sHtML
tv.blog.xlruof.cn/Article/details/911777.sHtML
tv.blog.xlruof.cn/Article/details/220644.sHtML
tv.blog.xlruof.cn/Article/details/284820.sHtML
tv.blog.xlruof.cn/Article/details/759717.sHtML
tv.blog.xlruof.cn/Article/details/082464.sHtML
tv.blog.xlruof.cn/Article/details/591779.sHtML
tv.blog.xlruof.cn/Article/details/464028.sHtML
tv.blog.xlruof.cn/Article/details/248026.sHtML
tv.blog.xlruof.cn/Article/details/602406.sHtML
tv.blog.xlruof.cn/Article/details/420806.sHtML
tv.blog.xlruof.cn/Article/details/555573.sHtML
tv.blog.xlruof.cn/Article/details/864824.sHtML
tv.blog.xlruof.cn/Article/details/064468.sHtML
tv.blog.xlruof.cn/Article/details/533755.sHtML
tv.blog.xlruof.cn/Article/details/599171.sHtML
tv.blog.xlruof.cn/Article/details/886822.sHtML
tv.blog.xlruof.cn/Article/details/840880.sHtML
tv.blog.xlruof.cn/Article/details/264806.sHtML
tv.blog.xlruof.cn/Article/details/913595.sHtML
tv.blog.xlruof.cn/Article/details/444206.sHtML
tv.blog.xlruof.cn/Article/details/088240.sHtML
tv.blog.xlruof.cn/Article/details/177791.sHtML
tv.blog.xlruof.cn/Article/details/022068.sHtML
tv.blog.xlruof.cn/Article/details/060488.sHtML
tv.blog.xlruof.cn/Article/details/402420.sHtML
tv.blog.xlruof.cn/Article/details/622008.sHtML
tv.blog.xlruof.cn/Article/details/220664.sHtML
tv.blog.xlruof.cn/Article/details/082040.sHtML
tv.blog.xlruof.cn/Article/details/820862.sHtML
tv.blog.xlruof.cn/Article/details/226848.sHtML
tv.blog.xlruof.cn/Article/details/082826.sHtML
tv.blog.xlruof.cn/Article/details/040844.sHtML
tv.blog.xlruof.cn/Article/details/828286.sHtML
tv.blog.xlruof.cn/Article/details/022084.sHtML
tv.blog.xlruof.cn/Article/details/319577.sHtML
tv.blog.xlruof.cn/Article/details/664888.sHtML
tv.blog.xlruof.cn/Article/details/228600.sHtML
tv.blog.xlruof.cn/Article/details/820408.sHtML
tv.blog.xlruof.cn/Article/details/371131.sHtML
tv.blog.xlruof.cn/Article/details/404880.sHtML
tv.blog.xlruof.cn/Article/details/446680.sHtML
tv.blog.xlruof.cn/Article/details/668026.sHtML
tv.blog.xlruof.cn/Article/details/404486.sHtML
tv.blog.xlruof.cn/Article/details/579355.sHtML
tv.blog.xlruof.cn/Article/details/557737.sHtML
tv.blog.xlruof.cn/Article/details/519995.sHtML
tv.blog.xlruof.cn/Article/details/086288.sHtML
tv.blog.xlruof.cn/Article/details/640402.sHtML
tv.blog.xlruof.cn/Article/details/020888.sHtML
tv.blog.xlruof.cn/Article/details/480860.sHtML
tv.blog.xlruof.cn/Article/details/195777.sHtML
tv.blog.xlruof.cn/Article/details/682860.sHtML
tv.blog.xlruof.cn/Article/details/353171.sHtML
tv.blog.xlruof.cn/Article/details/428282.sHtML
tv.blog.xlruof.cn/Article/details/995393.sHtML
tv.blog.xlruof.cn/Article/details/577391.sHtML
tv.blog.xlruof.cn/Article/details/888002.sHtML
tv.blog.xlruof.cn/Article/details/719313.sHtML
tv.blog.xlruof.cn/Article/details/248620.sHtML
tv.blog.xlruof.cn/Article/details/915711.sHtML
tv.blog.xlruof.cn/Article/details/402880.sHtML
tv.blog.xlruof.cn/Article/details/517991.sHtML
tv.blog.xlruof.cn/Article/details/082622.sHtML
tv.blog.xlruof.cn/Article/details/624882.sHtML
tv.blog.xlruof.cn/Article/details/917511.sHtML
tv.blog.xlruof.cn/Article/details/208886.sHtML
tv.blog.xlruof.cn/Article/details/644682.sHtML
tv.blog.xlruof.cn/Article/details/242088.sHtML
tv.blog.xlruof.cn/Article/details/426866.sHtML
tv.blog.xlruof.cn/Article/details/242820.sHtML
tv.blog.xlruof.cn/Article/details/004848.sHtML
tv.blog.xlruof.cn/Article/details/084664.sHtML
tv.blog.xlruof.cn/Article/details/040444.sHtML
tv.blog.xlruof.cn/Article/details/808888.sHtML
tv.blog.xlruof.cn/Article/details/266242.sHtML
tv.blog.xlruof.cn/Article/details/426680.sHtML
tv.blog.xlruof.cn/Article/details/262828.sHtML
tv.blog.xlruof.cn/Article/details/711935.sHtML
tv.blog.xlruof.cn/Article/details/268840.sHtML
tv.blog.xlruof.cn/Article/details/864024.sHtML
tv.blog.xlruof.cn/Article/details/804646.sHtML
tv.blog.xlruof.cn/Article/details/224606.sHtML
tv.blog.xlruof.cn/Article/details/066422.sHtML
tv.blog.xlruof.cn/Article/details/886886.sHtML
tv.blog.xlruof.cn/Article/details/713377.sHtML
tv.blog.xlruof.cn/Article/details/179759.sHtML
tv.blog.xlruof.cn/Article/details/331553.sHtML
tv.blog.xlruof.cn/Article/details/040462.sHtML
tv.blog.xlruof.cn/Article/details/608004.sHtML
tv.blog.xlruof.cn/Article/details/440084.sHtML
tv.blog.xlruof.cn/Article/details/624800.sHtML
tv.blog.xlruof.cn/Article/details/488080.sHtML
tv.blog.xlruof.cn/Article/details/642202.sHtML
tv.blog.xlruof.cn/Article/details/195151.sHtML
tv.blog.xlruof.cn/Article/details/886204.sHtML
tv.blog.xlruof.cn/Article/details/240464.sHtML
tv.blog.xlruof.cn/Article/details/862642.sHtML
tv.blog.xlruof.cn/Article/details/331517.sHtML
tv.blog.xlruof.cn/Article/details/117797.sHtML
tv.blog.xlruof.cn/Article/details/460020.sHtML
tv.blog.xlruof.cn/Article/details/377395.sHtML
tv.blog.xlruof.cn/Article/details/604840.sHtML
tv.blog.xlruof.cn/Article/details/000466.sHtML
tv.blog.xlruof.cn/Article/details/711735.sHtML
tv.blog.xlruof.cn/Article/details/484808.sHtML
tv.blog.xlruof.cn/Article/details/195315.sHtML
tv.blog.xlruof.cn/Article/details/680468.sHtML
tv.blog.xlruof.cn/Article/details/719979.sHtML
tv.blog.xlruof.cn/Article/details/268844.sHtML
tv.blog.xlruof.cn/Article/details/664002.sHtML
tv.blog.xlruof.cn/Article/details/608666.sHtML
tv.blog.xlruof.cn/Article/details/599573.sHtML
tv.blog.xlruof.cn/Article/details/686008.sHtML
tv.blog.xlruof.cn/Article/details/002264.sHtML
tv.blog.xlruof.cn/Article/details/919591.sHtML
tv.blog.xlruof.cn/Article/details/200260.sHtML
tv.blog.xlruof.cn/Article/details/680464.sHtML
tv.blog.xlruof.cn/Article/details/442822.sHtML
tv.blog.xlruof.cn/Article/details/224800.sHtML
tv.blog.xlruof.cn/Article/details/024802.sHtML
tv.blog.xlruof.cn/Article/details/804022.sHtML
tv.blog.xlruof.cn/Article/details/644088.sHtML
tv.blog.xlruof.cn/Article/details/593993.sHtML
tv.blog.xlruof.cn/Article/details/602660.sHtML
tv.blog.xlruof.cn/Article/details/246660.sHtML
tv.blog.xlruof.cn/Article/details/284400.sHtML
tv.blog.xlruof.cn/Article/details/400202.sHtML
tv.blog.xlruof.cn/Article/details/208444.sHtML
tv.blog.xlruof.cn/Article/details/206060.sHtML
tv.blog.xlruof.cn/Article/details/115951.sHtML
tv.blog.xlruof.cn/Article/details/862642.sHtML
tv.blog.xlruof.cn/Article/details/428448.sHtML
tv.blog.xlruof.cn/Article/details/262462.sHtML
tv.blog.xlruof.cn/Article/details/155337.sHtML
tv.blog.xlruof.cn/Article/details/860000.sHtML
tv.blog.xlruof.cn/Article/details/460286.sHtML
tv.blog.xlruof.cn/Article/details/462024.sHtML
tv.blog.xlruof.cn/Article/details/044486.sHtML
tv.blog.xlruof.cn/Article/details/802224.sHtML
tv.blog.xlruof.cn/Article/details/042040.sHtML
tv.blog.xlruof.cn/Article/details/575531.sHtML
tv.blog.xlruof.cn/Article/details/202480.sHtML
tv.blog.xlruof.cn/Article/details/802862.sHtML
tv.blog.xlruof.cn/Article/details/664464.sHtML
tv.blog.xlruof.cn/Article/details/460228.sHtML
tv.blog.xlruof.cn/Article/details/680604.sHtML
tv.blog.xlruof.cn/Article/details/042488.sHtML
tv.blog.xlruof.cn/Article/details/713755.sHtML
tv.blog.xlruof.cn/Article/details/917137.sHtML
tv.blog.xlruof.cn/Article/details/173719.sHtML
tv.blog.xlruof.cn/Article/details/806002.sHtML
tv.blog.xlruof.cn/Article/details/066468.sHtML
tv.blog.xlruof.cn/Article/details/666488.sHtML
tv.blog.xlruof.cn/Article/details/808480.sHtML
tv.blog.xlruof.cn/Article/details/646242.sHtML
tv.blog.xlruof.cn/Article/details/668866.sHtML
tv.blog.xlruof.cn/Article/details/759573.sHtML
tv.blog.xlruof.cn/Article/details/551537.sHtML
tv.blog.xlruof.cn/Article/details/335311.sHtML
tv.blog.xlruof.cn/Article/details/840204.sHtML
tv.blog.xlruof.cn/Article/details/608082.sHtML
tv.blog.xlruof.cn/Article/details/404042.sHtML
tv.blog.xlruof.cn/Article/details/595115.sHtML
tv.blog.xlruof.cn/Article/details/284622.sHtML
tv.blog.xlruof.cn/Article/details/133375.sHtML
tv.blog.xlruof.cn/Article/details/468808.sHtML
tv.blog.xlruof.cn/Article/details/242088.sHtML
tv.blog.xlruof.cn/Article/details/268608.sHtML
tv.blog.xlruof.cn/Article/details/824206.sHtML
tv.blog.xlruof.cn/Article/details/111131.sHtML
tv.blog.xlruof.cn/Article/details/682822.sHtML
tv.blog.xlruof.cn/Article/details/806242.sHtML
tv.blog.xlruof.cn/Article/details/757553.sHtML
tv.blog.xlruof.cn/Article/details/486682.sHtML
tv.blog.xlruof.cn/Article/details/315511.sHtML
tv.blog.xlruof.cn/Article/details/648884.sHtML
tv.blog.xlruof.cn/Article/details/464402.sHtML
tv.blog.xlruof.cn/Article/details/648268.sHtML
tv.blog.xlruof.cn/Article/details/111513.sHtML
tv.blog.xlruof.cn/Article/details/828642.sHtML
tv.blog.xlruof.cn/Article/details/973919.sHtML
tv.blog.xlruof.cn/Article/details/002068.sHtML
