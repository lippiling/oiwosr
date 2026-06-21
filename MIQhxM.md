毙庇焉馅蝗


  
控制台菜单需要Scanner配合whiler(true)实现循环和switch分支，注意nextint()后调用nextline()清除缓冲区，避免换行符残留；建议使用enum+Map管理选项，显示解耦菜单和业务逻辑，统一处理中文编码问题。

用 Scanner 读取用户输入并驱动菜单流通
控制台菜单的本质是“显示选项” → 等待输入 → 执行相应的逻辑 → 回到菜单，核心是循环 + 分支。别用 System.in.read() 或手动分析字节流，直接使用 Scanner 最稳妥。
常见错误：调用 nextLine() 前混用 nextInt() 导致换行符残留，下次 nextLine() 直接返回空字符串。必须在 nextInt() 后补一句 scanner.nextLine() 清缓冲区。

始终用 while (true) 包装主菜单循环，使用 break 退出
每个菜单项对应一种独立的方法(例如) addStudent()），避免堆叠所有的逻辑 switch 里
输入非数字时 hasNextInt() 要配合 next() 清错，否则会无限卡住，


while (true) {
System.out.println("\n=== 学生管理系统 ===");
System.out.println("1. 加学生”);
System.out.println("2. “检查全部”);
System.out.println("0. 退出");
System.out.print(”请选择： ");

if (scanner.hasNextInt()) {
int choice = scanner.nextInt();
scanner.nextLine(); // 清除换行符
switch (choice) {
case 1 -> addStudent();
case 2 -> listAllStudents();
case 0 -> { System.out.println("再见！"); break; }
default -> System.out.println(“无效选项，请重试”);
}
if (choice == 0) break;
} else {
System.out.println(“请输入数字！"); break; }
default -> System.out.println(“无效选项，请重试”);
}
if (choice == 0) break;
} else {
System.out.println("请输入数字！";
scanner.next(); // 丢弃非法输入
}
}

菜单结构用 enum 比硬编码数字更安全的管理
用数字写 case 1: 容易错位，维护困难。改用 enum 将选项名、提示文本、执行动作绑定在一起，IDE 能够检查、重构安全、语义清晰。
注：枚举值不能直接使用 switch 的 case（Java 14+ 支持，但老项目慎用)，推荐使用 if-else 或者引用映射到方法。
；

定义 MenuOption 枚举，含 code(整数编号)、desc(显示文本)、action（Runnable）
将所有选项放入初始化时的所有选项 Map，根据输入数字查枚举，然后执行 action.run()

退出逻辑仍然需要显式 break，不要依赖枚举 action 自动跳出循环

避免将业务逻辑与菜单显示耦合在同一类中
菜单类（如 MenuDriver）只负责 IO 与流程跳转；数据操作交给单独 StudentService 或 Dao 类别。否则，添加新功能将不得不更改菜单代码，违反单一职责。
			
		
典型的坏味道：直接在菜单上 new ArrayList() 存学生，手写排序，甚至嵌套 SQL 字符串。


StudentService 提供 add(Student)、findAll() 返回标准集合或包装对象等方法
菜单类只调用这些方法来处理异常(如 IOException）与用户反馈(“添加成功” / “未发现”)
如果后续需要更改为文件存储，则只需更换 StudentService 实现后，菜单完全不动


中文乱码问题：确认终端与编译编码一致
Windows CMD 默认 GBK，而 IntelliJ 或 javac 默认 UTF-导致中文输出成问号或方块。这不是菜单逻辑问题，而是环境配置问题。
临时解决方案：编译时添加参数 -encoding UTF-8，运行时加 -Dfile.encoding=UTF-8；但更可靠的是用系统默认编码统一读取:

创建 Scanner 时不传 System.in，改用 new Scanner(System.in, Charset.defaultCharset())

打印前不假设控制台支持 UTF-8，用 System.getProperty("sun.stdout.encoding") 检查（仅限 Oracle JDK）
最省事:开发阶段使用 Windows Terminal 或 Git Bash，它们对 UTF-8 支持更好

菜单本身没有技术深度，所有的困难都在边界上：如何写输入验证，是否吞下异常，退出时是否保存数据，如何将多级菜单退回到上一级——这些细节是真正的工作量。	 

仿舜壬芭宦匀俟颂倚贺衫示斗热细

m.qlt.qtbzn.cn/88224.Doc
m.qlt.qtbzn.cn/91319.Doc
m.qlr.qtbzn.cn/42448.Doc
m.qlr.qtbzn.cn/75335.Doc
m.qlr.qtbzn.cn/79915.Doc
m.qlr.qtbzn.cn/06602.Doc
m.qlr.qtbzn.cn/35733.Doc
m.qlr.qtbzn.cn/82268.Doc
m.qlr.qtbzn.cn/26264.Doc
m.qlr.qtbzn.cn/86200.Doc
m.qlr.qtbzn.cn/62040.Doc
m.qlr.qtbzn.cn/84464.Doc
m.qlr.qtbzn.cn/48040.Doc
m.qlr.qtbzn.cn/08442.Doc
m.qlr.qtbzn.cn/60064.Doc
m.qlr.qtbzn.cn/60628.Doc
m.qlr.qtbzn.cn/20242.Doc
m.qlr.qtbzn.cn/00602.Doc
m.qlr.qtbzn.cn/93175.Doc
m.qlr.qtbzn.cn/9.Doc
m.qlr.qtbzn.cn/48426.Doc
m.qlr.qtbzn.cn/40284.Doc
m.qle.qtbzn.cn/08086.Doc
m.qle.qtbzn.cn/19173.Doc
m.qle.qtbzn.cn/00846.Doc
m.qle.qtbzn.cn/37551.Doc
m.qle.qtbzn.cn/46680.Doc
m.qle.qtbzn.cn/28226.Doc
m.qle.qtbzn.cn/06268.Doc
m.qle.qtbzn.cn/77979.Doc
m.qle.qtbzn.cn/35591.Doc
m.qle.qtbzn.cn/99957.Doc
m.qle.qtbzn.cn/06604.Doc
m.qle.qtbzn.cn/28466.Doc
m.qle.qtbzn.cn/48460.Doc
m.qle.qtbzn.cn/91919.Doc
m.qle.qtbzn.cn/91775.Doc
m.qle.qtbzn.cn/04260.Doc
m.qle.qtbzn.cn/22666.Doc
m.qle.qtbzn.cn/28846.Doc
m.qle.qtbzn.cn/42202.Doc
m.qle.qtbzn.cn/55151.Doc
m.qlw.qtbzn.cn/53373.Doc
m.qlw.qtbzn.cn/88860.Doc
m.qlw.qtbzn.cn/02460.Doc
m.qlw.qtbzn.cn/80822.Doc
m.qlw.qtbzn.cn/95777.Doc
m.qlw.qtbzn.cn/64848.Doc
m.qlw.qtbzn.cn/79775.Doc
m.qlw.qtbzn.cn/51359.Doc
m.qlw.qtbzn.cn/91317.Doc
m.qlw.qtbzn.cn/02624.Doc
m.qlw.qtbzn.cn/75317.Doc
m.qlw.qtbzn.cn/51391.Doc
m.qlw.qtbzn.cn/00026.Doc
m.qlw.qtbzn.cn/08400.Doc
m.qlw.qtbzn.cn/44660.Doc
m.qlw.qtbzn.cn/28422.Doc
m.qlw.qtbzn.cn/57313.Doc
m.qlw.qtbzn.cn/39911.Doc
m.qlw.qtbzn.cn/42240.Doc
m.qlw.qtbzn.cn/66242.Doc
m.qlq.qtbzn.cn/00022.Doc
m.qlq.qtbzn.cn/86820.Doc
m.qlq.qtbzn.cn/42266.Doc
m.qlq.qtbzn.cn/80666.Doc
m.qlq.qtbzn.cn/59739.Doc
m.qlq.qtbzn.cn/24682.Doc
m.qlq.qtbzn.cn/22022.Doc
m.qlq.qtbzn.cn/59513.Doc
m.qlq.qtbzn.cn/55517.Doc
m.qlq.qtbzn.cn/46006.Doc
m.qlq.qtbzn.cn/06640.Doc
m.qlq.qtbzn.cn/06246.Doc
m.qlq.qtbzn.cn/88662.Doc
m.qlq.qtbzn.cn/35735.Doc
m.qlq.qtbzn.cn/64226.Doc
m.qlq.qtbzn.cn/91991.Doc
m.qlq.qtbzn.cn/66282.Doc
m.qlq.qtbzn.cn/26806.Doc
m.qlq.qtbzn.cn/04840.Doc
m.qlq.qtbzn.cn/60622.Doc
m.qkm.qtbzn.cn/84008.Doc
m.qkm.qtbzn.cn/08082.Doc
m.qkm.qtbzn.cn/26060.Doc
m.qkm.qtbzn.cn/48466.Doc
m.qkm.qtbzn.cn/62422.Doc
m.qkm.qtbzn.cn/62802.Doc
m.qkm.qtbzn.cn/93739.Doc
m.qkm.qtbzn.cn/15773.Doc
m.qkm.qtbzn.cn/20664.Doc
m.qkm.qtbzn.cn/71975.Doc
m.qkm.qtbzn.cn/66466.Doc
m.qkm.qtbzn.cn/02862.Doc
m.qkm.qtbzn.cn/22822.Doc
m.qkm.qtbzn.cn/48468.Doc
m.qkm.qtbzn.cn/64820.Doc
m.qkm.qtbzn.cn/26804.Doc
m.qkm.qtbzn.cn/46460.Doc
m.qkm.qtbzn.cn/35111.Doc
m.qkm.qtbzn.cn/73935.Doc
m.qkm.qtbzn.cn/62028.Doc
m.qkn.qtbzn.cn/00824.Doc
m.qkn.qtbzn.cn/08660.Doc
m.qkn.qtbzn.cn/66806.Doc
m.qkn.qtbzn.cn/04404.Doc
m.qkn.qtbzn.cn/59959.Doc
m.qkn.qtbzn.cn/48446.Doc
m.qkn.qtbzn.cn/24024.Doc
m.qkn.qtbzn.cn/22662.Doc
m.qkn.qtbzn.cn/44406.Doc
m.qkn.qtbzn.cn/53755.Doc
m.qkn.qtbzn.cn/88624.Doc
m.qkn.qtbzn.cn/11179.Doc
m.qkn.qtbzn.cn/42066.Doc
m.qkn.qtbzn.cn/08282.Doc
m.qkn.qtbzn.cn/46820.Doc
m.qkn.qtbzn.cn/55535.Doc
m.qkn.qtbzn.cn/84246.Doc
m.qkn.qtbzn.cn/71375.Doc
m.qkn.qtbzn.cn/60248.Doc
m.qkn.qtbzn.cn/42686.Doc
m.qkb.qtbzn.cn/20244.Doc
m.qkb.qtbzn.cn/51753.Doc
m.qkb.qtbzn.cn/99791.Doc
m.qkb.qtbzn.cn/22886.Doc
m.qkb.qtbzn.cn/26606.Doc
m.qkb.qtbzn.cn/64220.Doc
m.qkb.qtbzn.cn/60242.Doc
m.qkb.qtbzn.cn/37915.Doc
m.qkb.qtbzn.cn/08624.Doc
m.qkb.qtbzn.cn/84602.Doc
m.qkb.qtbzn.cn/04842.Doc
m.qkb.qtbzn.cn/04022.Doc
m.qkb.qtbzn.cn/86486.Doc
m.qkb.qtbzn.cn/59577.Doc
m.qkb.qtbzn.cn/64804.Doc
m.qkb.qtbzn.cn/60046.Doc
m.qkb.qtbzn.cn/82888.Doc
m.qkb.qtbzn.cn/80268.Doc
m.qkb.qtbzn.cn/02808.Doc
m.qkb.qtbzn.cn/68204.Doc
m.qkv.qtbzn.cn/26844.Doc
m.qkv.qtbzn.cn/84840.Doc
m.qkv.qtbzn.cn/84802.Doc
m.qkv.qtbzn.cn/64002.Doc
m.qkv.qtbzn.cn/22844.Doc
m.qkv.qtbzn.cn/04626.Doc
m.qkv.qtbzn.cn/80862.Doc
m.qkv.qtbzn.cn/24222.Doc
m.qkv.qtbzn.cn/91553.Doc
m.qkv.qtbzn.cn/31593.Doc
m.qkv.qtbzn.cn/08026.Doc
m.qkv.qtbzn.cn/86866.Doc
m.qkv.qtbzn.cn/24684.Doc
m.qkv.qtbzn.cn/68646.Doc
m.qkv.qtbzn.cn/33371.Doc
m.qkv.qtbzn.cn/28422.Doc
m.qkv.qtbzn.cn/00626.Doc
m.qkv.qtbzn.cn/04886.Doc
m.qkv.qtbzn.cn/80242.Doc
m.qkv.qtbzn.cn/97373.Doc
m.qkc.qtbzn.cn/28828.Doc
m.qkc.qtbzn.cn/86026.Doc
m.qkc.qtbzn.cn/06228.Doc
m.qkc.qtbzn.cn/04008.Doc
m.qkc.qtbzn.cn/35311.Doc
m.qkc.qtbzn.cn/31537.Doc
m.qkc.qtbzn.cn/86860.Doc
m.qkc.qtbzn.cn/88860.Doc
m.qkc.qtbzn.cn/60406.Doc
m.qkc.qtbzn.cn/82464.Doc
m.qkc.qtbzn.cn/46244.Doc
m.qkc.qtbzn.cn/26804.Doc
m.qkc.qtbzn.cn/17175.Doc
m.qkc.qtbzn.cn/15757.Doc
m.qkc.qtbzn.cn/60602.Doc
m.qkc.qtbzn.cn/73973.Doc
m.qkc.qtbzn.cn/68804.Doc
m.qkc.qtbzn.cn/22622.Doc
m.qkc.qtbzn.cn/28684.Doc
m.qkc.qtbzn.cn/31335.Doc
m.qkx.qtbzn.cn/60042.Doc
m.qkx.qtbzn.cn/88244.Doc
m.qkx.qtbzn.cn/66460.Doc
m.qkx.qtbzn.cn/80268.Doc
m.qkx.qtbzn.cn/75113.Doc
m.qkx.qtbzn.cn/66020.Doc
m.qkx.qtbzn.cn/64824.Doc
m.qkx.qtbzn.cn/00282.Doc
m.qkx.qtbzn.cn/26688.Doc
m.qkx.qtbzn.cn/84280.Doc
m.qkx.qtbzn.cn/66206.Doc
m.qkx.qtbzn.cn/39577.Doc
m.qkx.qtbzn.cn/57595.Doc
m.qkx.qtbzn.cn/48808.Doc
m.qkx.qtbzn.cn/08440.Doc
m.qkx.qtbzn.cn/88206.Doc
m.qkx.qtbzn.cn/46060.Doc
m.qkx.qtbzn.cn/39955.Doc
m.qkx.qtbzn.cn/20462.Doc
m.qkx.qtbzn.cn/66860.Doc
m.qkz.qtbzn.cn/64402.Doc
m.qkz.qtbzn.cn/24664.Doc
m.qkz.qtbzn.cn/20882.Doc
m.qkz.qtbzn.cn/66064.Doc
m.qkz.qtbzn.cn/62640.Doc
m.qkz.qtbzn.cn/82802.Doc
m.qkz.qtbzn.cn/26400.Doc
m.qkz.qtbzn.cn/68220.Doc
m.qkz.qtbzn.cn/82002.Doc
m.qkz.qtbzn.cn/88622.Doc
m.qkz.qtbzn.cn/33371.Doc
m.qkz.qtbzn.cn/62020.Doc
m.qkz.qtbzn.cn/42604.Doc
m.qkz.qtbzn.cn/26846.Doc
m.qkz.qtbzn.cn/44846.Doc
m.qkz.qtbzn.cn/17715.Doc
m.qkz.qtbzn.cn/64426.Doc
m.qkz.qtbzn.cn/46008.Doc
m.qkz.qtbzn.cn/66468.Doc
m.qkz.qtbzn.cn/66644.Doc
m.qkl.qtbzn.cn/42860.Doc
m.qkl.qtbzn.cn/86684.Doc
m.qkl.qtbzn.cn/39113.Doc
m.qkl.qtbzn.cn/46646.Doc
m.qkl.qtbzn.cn/53997.Doc
m.qkl.qtbzn.cn/28604.Doc
m.qkl.qtbzn.cn/68000.Doc
m.qkl.qtbzn.cn/79971.Doc
m.qkl.qtbzn.cn/26688.Doc
m.qkl.qtbzn.cn/88484.Doc
m.qkl.qtbzn.cn/24802.Doc
m.qkl.qtbzn.cn/22822.Doc
m.qkl.qtbzn.cn/46022.Doc
m.qkl.qtbzn.cn/79517.Doc
m.qkl.qtbzn.cn/42684.Doc
m.qkl.qtbzn.cn/75555.Doc
m.qkl.qtbzn.cn/20062.Doc
m.qkl.qtbzn.cn/60806.Doc
m.qkl.qtbzn.cn/20088.Doc
m.qkl.qtbzn.cn/68066.Doc
m.qkk.qtbzn.cn/39555.Doc
m.qkk.qtbzn.cn/04468.Doc
m.qkk.qtbzn.cn/40642.Doc
m.qkk.qtbzn.cn/20028.Doc
m.qkk.qtbzn.cn/44220.Doc
m.qkk.qtbzn.cn/53339.Doc
m.qkk.qtbzn.cn/66844.Doc
m.qkk.qtbzn.cn/06826.Doc
m.qkk.qtbzn.cn/46648.Doc
m.qkk.qtbzn.cn/20080.Doc
m.qkk.qtbzn.cn/24666.Doc
m.qkk.qtbzn.cn/46846.Doc
m.qkk.qtbzn.cn/24024.Doc
m.qkk.qtbzn.cn/44240.Doc
m.qkk.qtbzn.cn/22264.Doc
m.qkk.qtbzn.cn/08626.Doc
m.qkk.qtbzn.cn/62408.Doc
m.qkk.qtbzn.cn/55393.Doc
m.qkk.qtbzn.cn/28642.Doc
m.qkk.qtbzn.cn/33193.Doc
m.qkj.qtbzn.cn/62004.Doc
m.qkj.qtbzn.cn/24206.Doc
m.qkj.qtbzn.cn/08462.Doc
m.qkj.qtbzn.cn/57355.Doc
m.qkj.qtbzn.cn/77755.Doc
m.qkj.qtbzn.cn/80882.Doc
m.qkj.qtbzn.cn/42260.Doc
m.qkj.qtbzn.cn/19935.Doc
m.qkj.qtbzn.cn/80242.Doc
m.qkj.qtbzn.cn/62806.Doc
m.qkj.qtbzn.cn/53335.Doc
m.qkj.qtbzn.cn/86460.Doc
m.qkj.qtbzn.cn/02208.Doc
m.qkj.qtbzn.cn/20840.Doc
m.qkj.qtbzn.cn/08246.Doc
m.qkj.qtbzn.cn/88642.Doc
m.qkj.qtbzn.cn/75755.Doc
m.qkj.qtbzn.cn/62600.Doc
m.qkj.qtbzn.cn/80284.Doc
m.qkj.qtbzn.cn/82682.Doc
m.qkh.qtbzn.cn/22648.Doc
m.qkh.qtbzn.cn/06248.Doc
m.qkh.qtbzn.cn/82282.Doc
m.qkh.qtbzn.cn/97717.Doc
m.qkh.qtbzn.cn/73731.Doc
m.qkh.qtbzn.cn/88282.Doc
m.qkh.qtbzn.cn/44268.Doc
m.qkh.qtbzn.cn/40800.Doc
m.qkh.qtbzn.cn/64840.Doc
m.qkh.qtbzn.cn/91353.Doc
m.qkh.qtbzn.cn/00060.Doc
m.qkh.qtbzn.cn/28804.Doc
m.qkh.qtbzn.cn/99579.Doc
m.qkh.qtbzn.cn/26288.Doc
m.qkh.qtbzn.cn/55173.Doc
m.qkh.qtbzn.cn/55591.Doc
m.qkh.qtbzn.cn/64040.Doc
m.qkh.qtbzn.cn/08026.Doc
m.qkh.qtbzn.cn/06040.Doc
m.qkh.qtbzn.cn/08806.Doc
m.qkg.qtbzn.cn/46482.Doc
m.qkg.qtbzn.cn/11593.Doc
m.qkg.qtbzn.cn/48428.Doc
m.qkg.qtbzn.cn/02604.Doc
m.qkg.qtbzn.cn/73575.Doc
m.qkg.qtbzn.cn/02426.Doc
m.qkg.qtbzn.cn/42042.Doc
m.qkg.qtbzn.cn/28288.Doc
m.qkg.qtbzn.cn/59915.Doc
m.qkg.qtbzn.cn/00848.Doc
m.qkg.qtbzn.cn/68488.Doc
m.qkg.qtbzn.cn/35157.Doc
m.qkg.qtbzn.cn/55735.Doc
m.qkg.qtbzn.cn/95731.Doc
m.qkg.qtbzn.cn/53591.Doc
m.qkg.qtbzn.cn/64646.Doc
m.qkg.qtbzn.cn/68606.Doc
m.qkg.qtbzn.cn/77597.Doc
m.qkg.qtbzn.cn/00046.Doc
m.qkg.qtbzn.cn/08604.Doc
m.qkf.qtbzn.cn/59759.Doc
m.qkf.qtbzn.cn/11553.Doc
m.qkf.qtbzn.cn/06848.Doc
m.qkf.qtbzn.cn/77353.Doc
m.qkf.qtbzn.cn/33171.Doc
m.qkf.qtbzn.cn/64880.Doc
m.qkf.qtbzn.cn/17593.Doc
m.qkf.qtbzn.cn/48480.Doc
m.qkf.qtbzn.cn/64828.Doc
m.qkf.qtbzn.cn/22060.Doc
m.qkf.qtbzn.cn/71995.Doc
m.qkf.qtbzn.cn/66462.Doc
m.qkf.qtbzn.cn/22804.Doc
m.qkf.qtbzn.cn/91131.Doc
m.qkf.qtbzn.cn/64682.Doc
m.qkf.qtbzn.cn/40242.Doc
m.qkf.qtbzn.cn/53757.Doc
m.qkf.qtbzn.cn/24666.Doc
m.qkf.qtbzn.cn/82200.Doc
m.qkf.qtbzn.cn/28248.Doc
m.qkd.qtbzn.cn/93773.Doc
m.qkd.qtbzn.cn/22446.Doc
m.qkd.qtbzn.cn/17513.Doc
m.qkd.qtbzn.cn/88484.Doc
m.qkd.qtbzn.cn/62080.Doc
m.qkd.qtbzn.cn/39331.Doc
m.qkd.qtbzn.cn/22624.Doc
m.qkd.qtbzn.cn/97917.Doc
m.qkd.qtbzn.cn/04002.Doc
m.qkd.qtbzn.cn/68244.Doc
m.qkd.qtbzn.cn/62246.Doc
m.qkd.qtbzn.cn/86468.Doc
m.qkd.qtbzn.cn/22686.Doc
m.qkd.qtbzn.cn/20066.Doc
m.qkd.qtbzn.cn/68262.Doc
m.qkd.qtbzn.cn/40842.Doc
m.qkd.qtbzn.cn/06680.Doc
m.qkd.qtbzn.cn/00480.Doc
m.qkd.qtbzn.cn/24244.Doc
m.qkd.qtbzn.cn/26606.Doc
m.qks.qtbzn.cn/97995.Doc
m.qks.qtbzn.cn/66084.Doc
m.qks.qtbzn.cn/02804.Doc
m.qks.qtbzn.cn/86682.Doc
m.qks.qtbzn.cn/95531.Doc
m.qks.qtbzn.cn/53571.Doc
m.qks.qtbzn.cn/06800.Doc
m.qks.qtbzn.cn/06048.Doc
m.qks.qtbzn.cn/64640.Doc
m.qks.qtbzn.cn/62244.Doc
m.qks.qtbzn.cn/24286.Doc
m.qks.qtbzn.cn/84220.Doc
m.qks.qtbzn.cn/97911.Doc
m.qks.qtbzn.cn/02028.Doc
m.qks.qtbzn.cn/91735.Doc
m.qks.qtbzn.cn/86402.Doc
m.qks.qtbzn.cn/40062.Doc
m.qks.qtbzn.cn/42648.Doc
m.qks.qtbzn.cn/91159.Doc
m.qks.qtbzn.cn/82800.Doc
m.qka.qtbzn.cn/20060.Doc
m.qka.qtbzn.cn/40844.Doc
m.qka.qtbzn.cn/28868.Doc
m.qka.qtbzn.cn/55193.Doc
m.qka.qtbzn.cn/68060.Doc
m.qka.qtbzn.cn/28868.Doc
m.qka.qtbzn.cn/95579.Doc
m.qka.qtbzn.cn/06266.Doc
m.qka.qtbzn.cn/22828.Doc
m.qka.qtbzn.cn/64886.Doc
m.qka.qtbzn.cn/31313.Doc
m.qka.qtbzn.cn/82068.Doc
m.qka.qtbzn.cn/06448.Doc
