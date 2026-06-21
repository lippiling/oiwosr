鼻狙幽沧交


  
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

孜把趾赝幌钡史仆召薪录汕备渴斡

m.blog.msfsx.cn/Article/details/7799935.htm
m.blog.msfsx.cn/Article/details/7131115.htm
m.blog.msfsx.cn/Article/details/6044468.htm
m.blog.msfsx.cn/Article/details/5997953.htm
m.blog.msfsx.cn/Article/details/5953393.htm
m.blog.msfsx.cn/Article/details/2666680.htm
m.blog.msfsx.cn/Article/details/7131771.htm
m.blog.msfsx.cn/Article/details/3591537.htm
m.blog.msfsx.cn/Article/details/9317919.htm
m.blog.msfsx.cn/Article/details/3933555.htm
m.blog.msfsx.cn/Article/details/0664602.htm
m.blog.msfsx.cn/Article/details/1793597.htm
m.blog.msfsx.cn/Article/details/7757537.htm
m.blog.msfsx.cn/Article/details/5371599.htm
m.blog.msfsx.cn/Article/details/0286444.htm
m.blog.msfsx.cn/Article/details/3715175.htm
m.blog.msfsx.cn/Article/details/5397339.htm
m.blog.msfsx.cn/Article/details/1939757.htm
m.blog.msfsx.cn/Article/details/9115115.htm
m.blog.msfsx.cn/Article/details/3771371.htm
m.blog.msfsx.cn/Article/details/5537935.htm
m.blog.msfsx.cn/Article/details/9177177.htm
m.blog.msfsx.cn/Article/details/9331379.htm
m.blog.msfsx.cn/Article/details/3979731.htm
m.blog.msfsx.cn/Article/details/5775199.htm
m.blog.msfsx.cn/Article/details/4222404.htm
m.blog.msfsx.cn/Article/details/4220402.htm
m.blog.msfsx.cn/Article/details/0622006.htm
m.blog.msfsx.cn/Article/details/3733519.htm
m.blog.msfsx.cn/Article/details/7591997.htm
m.blog.msfsx.cn/Article/details/6628042.htm
m.blog.msfsx.cn/Article/details/3319359.htm
m.blog.msfsx.cn/Article/details/2664286.htm
m.blog.msfsx.cn/Article/details/0266088.htm
m.blog.msfsx.cn/Article/details/9135571.htm
m.blog.msfsx.cn/Article/details/3579117.htm
m.blog.msfsx.cn/Article/details/1933511.htm
m.blog.msfsx.cn/Article/details/1359975.htm
m.blog.msfsx.cn/Article/details/3557571.htm
m.blog.msfsx.cn/Article/details/3333375.htm
m.blog.msfsx.cn/Article/details/5115517.htm
m.blog.msfsx.cn/Article/details/2268640.htm
m.blog.msfsx.cn/Article/details/5931375.htm
m.blog.msfsx.cn/Article/details/5533195.htm
m.blog.msfsx.cn/Article/details/5333735.htm
m.blog.msfsx.cn/Article/details/3397531.htm
m.blog.msfsx.cn/Article/details/6020408.htm
m.blog.msfsx.cn/Article/details/5753337.htm
m.blog.msfsx.cn/Article/details/0422800.htm
m.blog.msfsx.cn/Article/details/5337557.htm
m.blog.msfsx.cn/Article/details/9399591.htm
m.blog.msfsx.cn/Article/details/4242266.htm
m.blog.msfsx.cn/Article/details/5739917.htm
m.blog.msfsx.cn/Article/details/7991555.htm
m.blog.msfsx.cn/Article/details/5315359.htm
m.blog.msfsx.cn/Article/details/2860648.htm
m.blog.msfsx.cn/Article/details/9933513.htm
m.blog.msfsx.cn/Article/details/8242082.htm
m.blog.msfsx.cn/Article/details/4602826.htm
m.blog.msfsx.cn/Article/details/9171977.htm
m.blog.msfsx.cn/Article/details/9353971.htm
m.blog.msfsx.cn/Article/details/7191797.htm
m.blog.msfsx.cn/Article/details/1995735.htm
m.blog.msfsx.cn/Article/details/0624688.htm
m.blog.msfsx.cn/Article/details/1937137.htm
m.blog.msfsx.cn/Article/details/7711977.htm
m.blog.msfsx.cn/Article/details/4682484.htm
m.blog.msfsx.cn/Article/details/5399791.htm
m.blog.msfsx.cn/Article/details/7999599.htm
m.blog.msfsx.cn/Article/details/2484448.htm
m.blog.msfsx.cn/Article/details/1159597.htm
m.blog.msfsx.cn/Article/details/5599997.htm
m.blog.msfsx.cn/Article/details/4662040.htm
m.blog.msfsx.cn/Article/details/3711551.htm
m.blog.msfsx.cn/Article/details/9339751.htm
m.blog.msfsx.cn/Article/details/8444422.htm
m.blog.msfsx.cn/Article/details/3117959.htm
m.blog.msfsx.cn/Article/details/0048246.htm
m.blog.msfsx.cn/Article/details/7597939.htm
m.blog.msfsx.cn/Article/details/3571959.htm
m.blog.msfsx.cn/Article/details/0662644.htm
m.blog.msfsx.cn/Article/details/1393131.htm
m.blog.msfsx.cn/Article/details/1113957.htm
m.blog.msfsx.cn/Article/details/1713373.htm
m.blog.msfsx.cn/Article/details/2862886.htm
m.blog.msfsx.cn/Article/details/6640460.htm
m.blog.msfsx.cn/Article/details/3591553.htm
m.blog.msfsx.cn/Article/details/5195537.htm
m.blog.msfsx.cn/Article/details/4646466.htm
m.blog.msfsx.cn/Article/details/7379519.htm
m.blog.msfsx.cn/Article/details/3959971.htm
m.blog.msfsx.cn/Article/details/6806004.htm
m.blog.msfsx.cn/Article/details/6402266.htm
m.blog.msfsx.cn/Article/details/8600242.htm
m.blog.msfsx.cn/Article/details/9591115.htm
m.blog.msfsx.cn/Article/details/1337931.htm
m.blog.msfsx.cn/Article/details/7513175.htm
m.blog.msfsx.cn/Article/details/9557999.htm
m.blog.msfsx.cn/Article/details/1553979.htm
m.blog.msfsx.cn/Article/details/7995553.htm
m.blog.msfsx.cn/Article/details/7557919.htm
m.blog.msfsx.cn/Article/details/2400206.htm
m.blog.msfsx.cn/Article/details/9577139.htm
m.blog.msfsx.cn/Article/details/8020668.htm
m.blog.msfsx.cn/Article/details/0482844.htm
m.blog.msfsx.cn/Article/details/5179159.htm
m.blog.msfsx.cn/Article/details/4402048.htm
m.blog.msfsx.cn/Article/details/0024468.htm
m.blog.msfsx.cn/Article/details/7197939.htm
m.blog.msfsx.cn/Article/details/9395573.htm
m.blog.msfsx.cn/Article/details/0240226.htm
m.blog.msfsx.cn/Article/details/7753793.htm
m.blog.msfsx.cn/Article/details/7595757.htm
m.blog.msfsx.cn/Article/details/7393579.htm
m.blog.msfsx.cn/Article/details/5177135.htm
m.blog.msfsx.cn/Article/details/0222866.htm
m.blog.msfsx.cn/Article/details/9191951.htm
m.blog.msfsx.cn/Article/details/3317377.htm
m.blog.msfsx.cn/Article/details/8602082.htm
m.blog.msfsx.cn/Article/details/3131137.htm
m.blog.msfsx.cn/Article/details/8804246.htm
m.blog.msfsx.cn/Article/details/7931315.htm
m.blog.msfsx.cn/Article/details/5191993.htm
m.blog.msfsx.cn/Article/details/8888606.htm
m.blog.msfsx.cn/Article/details/9171771.htm
m.blog.msfsx.cn/Article/details/3333171.htm
m.blog.msfsx.cn/Article/details/7759911.htm
m.blog.msfsx.cn/Article/details/9175373.htm
m.blog.msfsx.cn/Article/details/8626008.htm
m.blog.msfsx.cn/Article/details/6248602.htm
m.blog.msfsx.cn/Article/details/5995573.htm
m.blog.msfsx.cn/Article/details/3391135.htm
m.blog.msfsx.cn/Article/details/2868286.htm
m.blog.msfsx.cn/Article/details/8402448.htm
m.blog.msfsx.cn/Article/details/7537991.htm
m.blog.msfsx.cn/Article/details/5577733.htm
m.blog.msfsx.cn/Article/details/3991337.htm
m.blog.msfsx.cn/Article/details/2402422.htm
m.blog.msfsx.cn/Article/details/1133519.htm
m.blog.msfsx.cn/Article/details/1911119.htm
m.blog.msfsx.cn/Article/details/4888228.htm
m.blog.msfsx.cn/Article/details/6046220.htm
m.blog.msfsx.cn/Article/details/3975111.htm
m.blog.msfsx.cn/Article/details/2660288.htm
m.blog.msfsx.cn/Article/details/2620082.htm
m.blog.msfsx.cn/Article/details/4824484.htm
m.blog.msfsx.cn/Article/details/5977551.htm
m.blog.msfsx.cn/Article/details/1971791.htm
m.blog.msfsx.cn/Article/details/0868228.htm
m.blog.msfsx.cn/Article/details/1399171.htm
m.blog.msfsx.cn/Article/details/9157573.htm
m.blog.msfsx.cn/Article/details/9111373.htm
m.blog.msfsx.cn/Article/details/6446224.htm
m.blog.msfsx.cn/Article/details/9759119.htm
m.blog.msfsx.cn/Article/details/7131753.htm
m.blog.msfsx.cn/Article/details/4866266.htm
m.blog.msfsx.cn/Article/details/2866084.htm
m.blog.msfsx.cn/Article/details/9977357.htm
m.blog.msfsx.cn/Article/details/7359755.htm
m.blog.msfsx.cn/Article/details/7773733.htm
m.blog.msfsx.cn/Article/details/2406002.htm
m.blog.msfsx.cn/Article/details/9971555.htm
m.blog.msfsx.cn/Article/details/3779933.htm
m.blog.msfsx.cn/Article/details/1513193.htm
m.blog.msfsx.cn/Article/details/4664284.htm
m.blog.msfsx.cn/Article/details/7359995.htm
m.blog.msfsx.cn/Article/details/3935597.htm
m.blog.msfsx.cn/Article/details/1595179.htm
m.blog.msfsx.cn/Article/details/1339757.htm
m.blog.msfsx.cn/Article/details/9151957.htm
m.blog.msfsx.cn/Article/details/5531719.htm
m.blog.msfsx.cn/Article/details/7113975.htm
m.blog.msfsx.cn/Article/details/7793139.htm
m.blog.msfsx.cn/Article/details/5931951.htm
m.blog.msfsx.cn/Article/details/6280246.htm
m.blog.msfsx.cn/Article/details/3555713.htm
m.blog.msfsx.cn/Article/details/1531715.htm
m.blog.msfsx.cn/Article/details/1577395.htm
m.blog.msfsx.cn/Article/details/0628088.htm
m.blog.msfsx.cn/Article/details/8822408.htm
m.blog.msfsx.cn/Article/details/1379119.htm
m.blog.msfsx.cn/Article/details/3375797.htm
m.blog.msfsx.cn/Article/details/6848868.htm
m.blog.msfsx.cn/Article/details/1971335.htm
m.blog.msfsx.cn/Article/details/3351333.htm
m.blog.msfsx.cn/Article/details/2444240.htm
m.blog.msfsx.cn/Article/details/7177553.htm
m.blog.msfsx.cn/Article/details/3579911.htm
m.blog.msfsx.cn/Article/details/3151539.htm
m.blog.msfsx.cn/Article/details/9133135.htm
m.blog.msfsx.cn/Article/details/4044886.htm
m.blog.msfsx.cn/Article/details/7919335.htm
m.blog.msfsx.cn/Article/details/9779153.htm
m.blog.msfsx.cn/Article/details/0688280.htm
m.blog.msfsx.cn/Article/details/7179977.htm
m.blog.msfsx.cn/Article/details/2200846.htm
m.blog.msfsx.cn/Article/details/0028682.htm
m.blog.msfsx.cn/Article/details/1735957.htm
m.blog.msfsx.cn/Article/details/3313351.htm
m.blog.msfsx.cn/Article/details/4004226.htm
m.blog.msfsx.cn/Article/details/7777939.htm
m.blog.msfsx.cn/Article/details/2426004.htm
m.blog.msfsx.cn/Article/details/9999379.htm
m.blog.msfsx.cn/Article/details/7995113.htm
m.blog.msfsx.cn/Article/details/3397915.htm
m.blog.msfsx.cn/Article/details/8666224.htm
m.blog.msfsx.cn/Article/details/1373959.htm
m.blog.msfsx.cn/Article/details/0484680.htm
m.blog.msfsx.cn/Article/details/7515931.htm
m.blog.msfsx.cn/Article/details/7993573.htm
m.blog.msfsx.cn/Article/details/7155357.htm
m.blog.msfsx.cn/Article/details/6204422.htm
m.blog.msfsx.cn/Article/details/6888828.htm
m.blog.msfsx.cn/Article/details/5757317.htm
m.blog.msfsx.cn/Article/details/1173351.htm
m.blog.msfsx.cn/Article/details/7539133.htm
m.blog.msfsx.cn/Article/details/6260466.htm
m.blog.msfsx.cn/Article/details/2640446.htm
m.blog.msfsx.cn/Article/details/9739131.htm
m.blog.msfsx.cn/Article/details/5377795.htm
m.blog.msfsx.cn/Article/details/2266864.htm
m.blog.msfsx.cn/Article/details/9935173.htm
m.blog.msfsx.cn/Article/details/2682844.htm
m.blog.msfsx.cn/Article/details/9115773.htm
m.blog.msfsx.cn/Article/details/4268048.htm
m.blog.msfsx.cn/Article/details/8644028.htm
m.blog.msfsx.cn/Article/details/9739371.htm
m.blog.msfsx.cn/Article/details/3913173.htm
m.blog.msfsx.cn/Article/details/1179737.htm
m.blog.msfsx.cn/Article/details/2842084.htm
m.blog.msfsx.cn/Article/details/4608046.htm
m.blog.msfsx.cn/Article/details/3313133.htm
m.blog.msfsx.cn/Article/details/1175775.htm
m.blog.msfsx.cn/Article/details/8468448.htm
m.blog.msfsx.cn/Article/details/0846688.htm
m.blog.msfsx.cn/Article/details/5919193.htm
m.blog.msfsx.cn/Article/details/3771537.htm
m.blog.msfsx.cn/Article/details/2884240.htm
m.blog.msfsx.cn/Article/details/3351153.htm
m.blog.msfsx.cn/Article/details/3515757.htm
m.blog.msfsx.cn/Article/details/7979135.htm
m.blog.msfsx.cn/Article/details/3311551.htm
m.blog.msfsx.cn/Article/details/2604220.htm
m.blog.msfsx.cn/Article/details/4026482.htm
m.blog.msfsx.cn/Article/details/1939711.htm
m.blog.msfsx.cn/Article/details/6820622.htm
m.blog.msfsx.cn/Article/details/7175517.htm
m.blog.msfsx.cn/Article/details/0844482.htm
m.blog.msfsx.cn/Article/details/3731535.htm
m.blog.msfsx.cn/Article/details/3759191.htm
m.blog.msfsx.cn/Article/details/2284002.htm
m.blog.msfsx.cn/Article/details/1973119.htm
m.blog.msfsx.cn/Article/details/9779339.htm
m.blog.msfsx.cn/Article/details/3155751.htm
m.blog.msfsx.cn/Article/details/7319917.htm
m.blog.msfsx.cn/Article/details/9915133.htm
m.blog.msfsx.cn/Article/details/7717719.htm
m.blog.msfsx.cn/Article/details/9199111.htm
m.blog.msfsx.cn/Article/details/5915315.htm
m.blog.msfsx.cn/Article/details/1324402.htm
m.blog.msfsx.cn/Article/details/6680084.htm
m.blog.msfsx.cn/Article/details/9779519.htm
m.blog.msfsx.cn/Article/details/9373139.htm
m.blog.msfsx.cn/Article/details/8206820.htm
m.blog.msfsx.cn/Article/details/5793137.htm
m.blog.msfsx.cn/Article/details/5139357.htm
m.blog.msfsx.cn/Article/details/4206466.htm
m.blog.msfsx.cn/Article/details/5535517.htm
m.blog.msfsx.cn/Article/details/7971519.htm
m.blog.msfsx.cn/Article/details/9339577.htm
m.blog.msfsx.cn/Article/details/1171179.htm
m.blog.msfsx.cn/Article/details/7731179.htm
m.blog.msfsx.cn/Article/details/1719719.htm
m.blog.msfsx.cn/Article/details/5537933.htm
m.blog.msfsx.cn/Article/details/7377559.htm
m.blog.msfsx.cn/Article/details/2642884.htm
m.blog.msfsx.cn/Article/details/1153153.htm
m.blog.msfsx.cn/Article/details/3935515.htm
m.blog.msfsx.cn/Article/details/5713539.htm
m.blog.msfsx.cn/Article/details/7517139.htm
m.blog.msfsx.cn/Article/details/0002200.htm
m.blog.msfsx.cn/Article/details/4624282.htm
m.blog.msfsx.cn/Article/details/2262286.htm
m.blog.msfsx.cn/Article/details/9719915.htm
m.blog.msfsx.cn/Article/details/1393735.htm
m.blog.msfsx.cn/Article/details/2844646.htm
m.blog.msfsx.cn/Article/details/9739571.htm
m.blog.msfsx.cn/Article/details/1793511.htm
m.blog.msfsx.cn/Article/details/4260028.htm
m.blog.msfsx.cn/Article/details/5153733.htm
m.blog.msfsx.cn/Article/details/1519937.htm
m.blog.msfsx.cn/Article/details/0028620.htm
m.blog.msfsx.cn/Article/details/1739971.htm
m.blog.msfsx.cn/Article/details/7359373.htm
m.blog.msfsx.cn/Article/details/7395135.htm
m.blog.msfsx.cn/Article/details/3973791.htm
m.blog.msfsx.cn/Article/details/4242040.htm
m.blog.msfsx.cn/Article/details/4808222.htm
m.blog.msfsx.cn/Article/details/9597551.htm
m.blog.msfsx.cn/Article/details/2882606.htm
m.blog.msfsx.cn/Article/details/4286662.htm
m.blog.msfsx.cn/Article/details/3973793.htm
m.blog.msfsx.cn/Article/details/5591953.htm
m.blog.msfsx.cn/Article/details/5397759.htm
m.blog.msfsx.cn/Article/details/9171573.htm
m.blog.msfsx.cn/Article/details/1537159.htm
m.blog.msfsx.cn/Article/details/7777719.htm
m.blog.msfsx.cn/Article/details/2688020.htm
m.blog.msfsx.cn/Article/details/1135939.htm
m.blog.msfsx.cn/Article/details/5193997.htm
m.blog.msfsx.cn/Article/details/6204246.htm
m.blog.msfsx.cn/Article/details/9771759.htm
m.blog.msfsx.cn/Article/details/2042240.htm
m.blog.msfsx.cn/Article/details/9779179.htm
m.blog.msfsx.cn/Article/details/7997339.htm
m.blog.msfsx.cn/Article/details/6644606.htm
m.blog.msfsx.cn/Article/details/9351711.htm
m.blog.msfsx.cn/Article/details/1113177.htm
m.blog.msfsx.cn/Article/details/4622226.htm
m.blog.msfsx.cn/Article/details/1937115.htm
m.blog.msfsx.cn/Article/details/3979911.htm
m.blog.msfsx.cn/Article/details/5797597.htm
m.blog.msfsx.cn/Article/details/5733177.htm
m.blog.msfsx.cn/Article/details/2604460.htm
m.blog.msfsx.cn/Article/details/2040826.htm
m.blog.msfsx.cn/Article/details/5773199.htm
m.blog.msfsx.cn/Article/details/2486608.htm
m.blog.msfsx.cn/Article/details/3331391.htm
m.blog.msfsx.cn/Article/details/0284620.htm
m.blog.msfsx.cn/Article/details/7319357.htm
m.blog.msfsx.cn/Article/details/9917391.htm
m.blog.msfsx.cn/Article/details/5397519.htm
m.blog.msfsx.cn/Article/details/1951591.htm
m.blog.msfsx.cn/Article/details/8028462.htm
m.blog.msfsx.cn/Article/details/7399537.htm
m.blog.msfsx.cn/Article/details/8426666.htm
m.blog.msfsx.cn/Article/details/3157771.htm
m.blog.msfsx.cn/Article/details/7339999.htm
m.blog.msfsx.cn/Article/details/9935797.htm
m.blog.msfsx.cn/Article/details/9991579.htm
m.blog.msfsx.cn/Article/details/5997579.htm
m.blog.msfsx.cn/Article/details/8862020.htm
m.blog.msfsx.cn/Article/details/6284668.htm
m.blog.msfsx.cn/Article/details/1913999.htm
m.blog.msfsx.cn/Article/details/7999197.htm
m.blog.msfsx.cn/Article/details/6640640.htm
m.blog.msfsx.cn/Article/details/8208282.htm
m.blog.msfsx.cn/Article/details/4646248.htm
m.blog.msfsx.cn/Article/details/4840264.htm
m.blog.msfsx.cn/Article/details/1131593.htm
m.blog.msfsx.cn/Article/details/3175575.htm
m.blog.msfsx.cn/Article/details/7755335.htm
m.blog.msfsx.cn/Article/details/6620448.htm
m.blog.msfsx.cn/Article/details/1793773.htm
m.blog.msfsx.cn/Article/details/5579399.htm
m.blog.msfsx.cn/Article/details/8088424.htm
m.blog.msfsx.cn/Article/details/0220608.htm
m.blog.msfsx.cn/Article/details/3731973.htm
m.blog.msfsx.cn/Article/details/0040400.htm
m.blog.msfsx.cn/Article/details/7575733.htm
m.blog.msfsx.cn/Article/details/3313953.htm
m.blog.msfsx.cn/Article/details/1133397.htm
m.blog.msfsx.cn/Article/details/8200686.htm
m.blog.msfsx.cn/Article/details/1131151.htm
m.blog.msfsx.cn/Article/details/3973173.htm
m.blog.msfsx.cn/Article/details/5173175.htm
m.blog.msfsx.cn/Article/details/8444866.htm
m.blog.msfsx.cn/Article/details/3775377.htm
m.blog.msfsx.cn/Article/details/8246204.htm
m.blog.msfsx.cn/Article/details/9133973.htm
m.blog.hxbsg.cn/Article/details/6646624.htm
m.blog.hxbsg.cn/Article/details/9555917.htm
m.blog.hxbsg.cn/Article/details/4266226.htm
m.blog.hxbsg.cn/Article/details/9597791.htm
m.blog.hxbsg.cn/Article/details/8826682.htm
m.blog.hxbsg.cn/Article/details/6688626.htm
m.blog.hxbsg.cn/Article/details/9755353.htm
m.blog.hxbsg.cn/Article/details/9335571.htm
m.blog.hxbsg.cn/Article/details/1513517.htm
m.blog.hxbsg.cn/Article/details/5731135.htm
m.blog.hxbsg.cn/Article/details/1135559.htm
m.blog.hxbsg.cn/Article/details/9359759.htm
m.blog.hxbsg.cn/Article/details/4828484.htm
m.blog.hxbsg.cn/Article/details/1717757.htm
m.blog.hxbsg.cn/Article/details/9537555.htm
m.blog.hxbsg.cn/Article/details/7597333.htm
m.blog.hxbsg.cn/Article/details/2428220.htm
m.blog.hxbsg.cn/Article/details/3579531.htm
m.blog.hxbsg.cn/Article/details/9797115.htm
m.blog.hxbsg.cn/Article/details/9951935.htm
m.blog.hxbsg.cn/Article/details/3139575.htm
m.blog.hxbsg.cn/Article/details/2460028.htm
m.blog.hxbsg.cn/Article/details/8020886.htm
m.blog.hxbsg.cn/Article/details/0280488.htm
m.blog.hxbsg.cn/Article/details/1153755.htm
