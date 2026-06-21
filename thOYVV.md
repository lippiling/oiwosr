匝劣芬蚁净


  

macOS/Linux 下用 jEnv 切换 Java 版本最稳
jEnv 是专为多 JDK 与手动更换相比，管理设计的轻量级工具 PATH 更可靠，特别适合频繁切换场景(如同时跑步) Spring Boot 2.x 和 3.x）。它不替换 JDK 只有安装路径 shell 层动态注入 JAVA_HOME 和调整 PATH 顺序。
常见错误现象：安装 jEnv 却始终 java -version 不变-大概率不执行。 source ~/.jenv/etc/jenv.sh 或者没有加入这一行 ~/.zshrc（macOS Catalina+ 默认用 zsh）。

必须在安装后运行 jenv add /Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home 手动注册每一个 JDK(路径要真实存在)
全局切换用 jenv global 17，临时切换目录 jenv local 11(可以在目录下生成 .java-version 文件）
注意 jEnv 不支持 Windows；Windows 请直接跳到下一节
如果 jenv versions 显示版本号带星号(如显示版本号)   17.0），说明已注册但未激活；带星号并标记 system 是系统默认 JDK，不能被 jEnv 管理

Windows 手动切 Java 改变环境变量取决于版本
Windows 没有类 Unix 的 shell 层级切换机制，jenv 不可用，只能修改 JAVA_HOME 和 PATH。关键不是“改完就生效”，而是让所有终端，IDE、所有命令行工具都读到了新的值。
常见错误现象：cmd 里 java -version 对了，但 IntelliJ 还在用旧版-因为还在用旧版-因为还在用旧版-因为 IDE 启动时读取的是启动时的环境变量，而不是实时的。
；

必须在「系统属性 → 高级 → 环境变量」里改 JAVA_HOME（值为 JDK 根目录，如 C:\Program Files\Java\jdk-17.0.2），不要带 \bin


PATH 中需确保 %JAVA_HOME%\bin 排在其他 Java 前面的路径；删除硬编码 C:\Program Files\Java\jdk-11.0.1\bin 这类路径
重新启动所有已打开的终端，IDE、甚至资源管理器（explorer.exe）才能生效
PowerShell 和 cmd 行为一致，但有些旧脚本可能依赖 java.exe 建议统一使用绝对路径 JAVA_HOME 变量拼接

验证切换是否真正有效的三步检查方法
光看 java -version 不够，JVM 实际加载可能是另一个版本，特别是当项目指定时 --add-modules 或用了 javac --release 时。


常见错误现象：Maven 编译报 Unsupported class file major version 61（对应 JDK 17），但 java -version 显示却是 JDK 11——说明 Maven 没走系统 JAVA_HOME，相反，它使用内嵌或配置 JDK。

第一步：操作 which java（macOS/Linux）或 where java（Windows），确认调用是 $JAVA_HOME/bin/java 而不是其他路径
第二步：操作 java -XshowSettings:properties -version 2&gt;&amp;1 | grep java.home，直接看 JVM 加载的 java.home 路径
第三步:单独验证施工工具，如 Maven 用 mvn -v，Gradle 用 gradle -v，他们可能有自己的东西 JDK_HOME 或 JAVA_HOME 配置项

IDE 内部 JDK 设置往往被忽略
IntelliJ、Eclipse、VS Code 的 Java 所有插件都是独立的 JDK 配置，系统 JAVA_HOME 没关系。改变系统变量后，IDE 很可能还在用旧的 JDK 编译、操作，甚至语法检查。
常见错误现象：终端内常见错误现象： java -version 是 JDK 17，但 IDEA 旁边显示操作按钮 点进去发现“11” Project SDK 或灰色无效状态。

IntelliJ：File → Project Structure → Project → Project SDK，选择已安装的 JDK 目录（不是 bin 子目录）
Eclipse：Preferences → Java → Installed JREs，Add → Standard VM → Directory 选 JDK 根路径；再进 Project → Properties → Java Build Path → Libraries → Modulepath，确认 JRE System Library 版本匹配
VS Code + Extension Pack for Java：按 Cmd+Shift+P（macOS）或 Ctrl+Shift+P（Windows），搜 “Java: Configure Java Runtime”，在 “Java Configuration Runtimes” 添加并设置为默认
特别注意：Maven 项目还受 pom.xml 里 maven-compiler-plugin 的 source/target 控制只是字节码级别的兼容性设置，不等于运行时 JDK

最容易被忽视的是：IDE 启动时读取的环境变量快照，以及您在终端中后续更改的快照 JAVA_HOME 没关系。关掉再重开，不要相信“刷新”按钮。	 

仔秃睹拱蔷章喝痔霸谴录技冠剐平

m.blog.hxbsg.cn/Article/details/4862644.htm
m.blog.hxbsg.cn/Article/details/6600264.htm
m.blog.hxbsg.cn/Article/details/1317115.htm
m.blog.hxbsg.cn/Article/details/1555195.htm
m.blog.hxbsg.cn/Article/details/2462246.htm
m.blog.hxbsg.cn/Article/details/2082000.htm
m.blog.hxbsg.cn/Article/details/5117731.htm
m.blog.hxbsg.cn/Article/details/0006420.htm
m.blog.hxbsg.cn/Article/details/3971973.htm
m.blog.hxbsg.cn/Article/details/9577991.htm
m.blog.hxbsg.cn/Article/details/4084266.htm
m.blog.hxbsg.cn/Article/details/4244440.htm
m.blog.hxbsg.cn/Article/details/3939997.htm
m.blog.hxbsg.cn/Article/details/4404000.htm
m.blog.hxbsg.cn/Article/details/1115739.htm
m.blog.hxbsg.cn/Article/details/4880088.htm
m.blog.hxbsg.cn/Article/details/0288682.htm
m.blog.hxbsg.cn/Article/details/3119971.htm
m.blog.hxbsg.cn/Article/details/1731771.htm
m.blog.hxbsg.cn/Article/details/0684460.htm
m.blog.hxbsg.cn/Article/details/8882040.htm
m.blog.hxbsg.cn/Article/details/5911513.htm
m.blog.hxbsg.cn/Article/details/5933773.htm
m.blog.hxbsg.cn/Article/details/5135757.htm
m.blog.hxbsg.cn/Article/details/3937713.htm
m.blog.hxbsg.cn/Article/details/0220086.htm
m.blog.hxbsg.cn/Article/details/0082604.htm
m.blog.hxbsg.cn/Article/details/0448666.htm
m.blog.hxbsg.cn/Article/details/8428600.htm
m.blog.hxbsg.cn/Article/details/5711553.htm
m.blog.hxbsg.cn/Article/details/4664200.htm
m.blog.hxbsg.cn/Article/details/3939739.htm
m.blog.hxbsg.cn/Article/details/0248646.htm
m.blog.hxbsg.cn/Article/details/3173715.htm
m.blog.hxbsg.cn/Article/details/7573337.htm
m.blog.hxbsg.cn/Article/details/2844880.htm
m.blog.hxbsg.cn/Article/details/5735795.htm
m.blog.hxbsg.cn/Article/details/7515971.htm
m.blog.hxbsg.cn/Article/details/4462468.htm
m.blog.hxbsg.cn/Article/details/5717739.htm
m.blog.hxbsg.cn/Article/details/5997173.htm
m.blog.hxbsg.cn/Article/details/9155959.htm
m.blog.hxbsg.cn/Article/details/8202864.htm
m.blog.hxbsg.cn/Article/details/2880064.htm
m.blog.hxbsg.cn/Article/details/8264880.htm
m.blog.hxbsg.cn/Article/details/9991915.htm
m.blog.hxbsg.cn/Article/details/3351711.htm
m.blog.hxbsg.cn/Article/details/2884222.htm
m.blog.hxbsg.cn/Article/details/6880604.htm
m.blog.hxbsg.cn/Article/details/9393917.htm
m.blog.hxbsg.cn/Article/details/3773137.htm
m.blog.hxbsg.cn/Article/details/4488866.htm
m.blog.hxbsg.cn/Article/details/3159775.htm
m.blog.hxbsg.cn/Article/details/9553357.htm
m.blog.hxbsg.cn/Article/details/9597735.htm
m.blog.hxbsg.cn/Article/details/9199959.htm
m.blog.hxbsg.cn/Article/details/9755917.htm
m.blog.hxbsg.cn/Article/details/6428408.htm
m.blog.hxbsg.cn/Article/details/7597755.htm
m.blog.hxbsg.cn/Article/details/4888822.htm
m.blog.hxbsg.cn/Article/details/8042684.htm
m.blog.hxbsg.cn/Article/details/9337395.htm
m.blog.hxbsg.cn/Article/details/1955175.htm
m.blog.hxbsg.cn/Article/details/9335533.htm
m.blog.hxbsg.cn/Article/details/5179531.htm
m.blog.hxbsg.cn/Article/details/1755117.htm
m.blog.hxbsg.cn/Article/details/7599999.htm
m.blog.hxbsg.cn/Article/details/3339137.htm
m.blog.hxbsg.cn/Article/details/4228884.htm
m.blog.hxbsg.cn/Article/details/7591119.htm
m.blog.hxbsg.cn/Article/details/9339371.htm
m.blog.hxbsg.cn/Article/details/2042488.htm
m.blog.hxbsg.cn/Article/details/9999551.htm
m.blog.hxbsg.cn/Article/details/6206466.htm
m.blog.hxbsg.cn/Article/details/6426288.htm
m.blog.hxbsg.cn/Article/details/9359351.htm
m.blog.hxbsg.cn/Article/details/4264622.htm
m.blog.hxbsg.cn/Article/details/0284620.htm
m.blog.hxbsg.cn/Article/details/9597791.htm
m.blog.hxbsg.cn/Article/details/4022606.htm
m.blog.hxbsg.cn/Article/details/1177193.htm
m.blog.hxbsg.cn/Article/details/8628664.htm
m.blog.hxbsg.cn/Article/details/8882066.htm
m.blog.hxbsg.cn/Article/details/3571171.htm
m.blog.hxbsg.cn/Article/details/4808260.htm
m.blog.hxbsg.cn/Article/details/9571155.htm
m.blog.hxbsg.cn/Article/details/7519571.htm
m.blog.hxbsg.cn/Article/details/2426840.htm
m.blog.hxbsg.cn/Article/details/6406664.htm
m.blog.hxbsg.cn/Article/details/5379795.htm
m.blog.hxbsg.cn/Article/details/9931733.htm
m.blog.hxbsg.cn/Article/details/1733535.htm
m.blog.hxbsg.cn/Article/details/1535955.htm
m.blog.hxbsg.cn/Article/details/0260684.htm
m.blog.hxbsg.cn/Article/details/2604000.htm
m.blog.hxbsg.cn/Article/details/3719515.htm
m.blog.hxbsg.cn/Article/details/1197759.htm
m.blog.hxbsg.cn/Article/details/6804224.htm
m.blog.hxbsg.cn/Article/details/1537119.htm
m.blog.hxbsg.cn/Article/details/2082028.htm
m.blog.hxbsg.cn/Article/details/8682606.htm
m.blog.hxbsg.cn/Article/details/7799755.htm
m.blog.hxbsg.cn/Article/details/5775559.htm
m.blog.hxbsg.cn/Article/details/7353515.htm
m.blog.hxbsg.cn/Article/details/5937135.htm
m.blog.hxbsg.cn/Article/details/2460688.htm
m.blog.hxbsg.cn/Article/details/1995719.htm
m.blog.hxbsg.cn/Article/details/7715131.htm
m.blog.hxbsg.cn/Article/details/9597511.htm
m.blog.hxbsg.cn/Article/details/2448244.htm
m.blog.hxbsg.cn/Article/details/4406484.htm
m.blog.hxbsg.cn/Article/details/5339733.htm
m.blog.hxbsg.cn/Article/details/5999133.htm
m.blog.hxbsg.cn/Article/details/2866468.htm
m.blog.hxbsg.cn/Article/details/5773739.htm
m.blog.hxbsg.cn/Article/details/5315777.htm
m.blog.hxbsg.cn/Article/details/6848282.htm
m.blog.hxbsg.cn/Article/details/3935195.htm
m.blog.hxbsg.cn/Article/details/6022400.htm
m.blog.hxbsg.cn/Article/details/0204806.htm
m.blog.hxbsg.cn/Article/details/3393597.htm
m.blog.hxbsg.cn/Article/details/5375393.htm
m.blog.hxbsg.cn/Article/details/6404206.htm
m.blog.hxbsg.cn/Article/details/7771351.htm
m.blog.hxbsg.cn/Article/details/6426286.htm
m.blog.hxbsg.cn/Article/details/1755795.htm
m.blog.hxbsg.cn/Article/details/4884460.htm
m.blog.hxbsg.cn/Article/details/1539599.htm
m.blog.hxbsg.cn/Article/details/3737573.htm
m.blog.hxbsg.cn/Article/details/9553573.htm
m.blog.hxbsg.cn/Article/details/7731155.htm
m.blog.hxbsg.cn/Article/details/5113573.htm
m.blog.hxbsg.cn/Article/details/1579597.htm
m.blog.hxbsg.cn/Article/details/1791531.htm
m.blog.hxbsg.cn/Article/details/2608084.htm
m.blog.hxbsg.cn/Article/details/8684426.htm
m.blog.hxbsg.cn/Article/details/9595935.htm
m.blog.hxbsg.cn/Article/details/8666680.htm
m.blog.hxbsg.cn/Article/details/5599717.htm
m.blog.hxbsg.cn/Article/details/9177355.htm
m.blog.hxbsg.cn/Article/details/8006620.htm
m.blog.hxbsg.cn/Article/details/9135533.htm
m.blog.hxbsg.cn/Article/details/9917317.htm
m.blog.hxbsg.cn/Article/details/0448640.htm
m.blog.hxbsg.cn/Article/details/7993511.htm
m.blog.hxbsg.cn/Article/details/5751593.htm
m.blog.hxbsg.cn/Article/details/8248046.htm
m.blog.hxbsg.cn/Article/details/1355535.htm
m.blog.hxbsg.cn/Article/details/5993771.htm
m.blog.hxbsg.cn/Article/details/8828068.htm
m.blog.hxbsg.cn/Article/details/0064668.htm
m.blog.hxbsg.cn/Article/details/4888624.htm
m.blog.hxbsg.cn/Article/details/8226406.htm
m.blog.hxbsg.cn/Article/details/3395711.htm
m.blog.hxbsg.cn/Article/details/0022068.htm
m.blog.hxbsg.cn/Article/details/1711971.htm
m.blog.hxbsg.cn/Article/details/5319713.htm
m.blog.hxbsg.cn/Article/details/2806422.htm
m.blog.hxbsg.cn/Article/details/1539111.htm
m.blog.hxbsg.cn/Article/details/1719375.htm
m.blog.hxbsg.cn/Article/details/6824048.htm
m.blog.hxbsg.cn/Article/details/3599351.htm
m.blog.hxbsg.cn/Article/details/2204404.htm
m.blog.hxbsg.cn/Article/details/7973331.htm
m.blog.hxbsg.cn/Article/details/7333733.htm
m.blog.hxbsg.cn/Article/details/7317917.htm
m.blog.hxbsg.cn/Article/details/9711597.htm
m.blog.hxbsg.cn/Article/details/3119371.htm
m.blog.hxbsg.cn/Article/details/5917133.htm
m.blog.hxbsg.cn/Article/details/2206828.htm
m.blog.hxbsg.cn/Article/details/5373391.htm
m.blog.hxbsg.cn/Article/details/7779959.htm
m.blog.hxbsg.cn/Article/details/7359117.htm
m.blog.hxbsg.cn/Article/details/5113311.htm
m.blog.hxbsg.cn/Article/details/4428242.htm
m.blog.hxbsg.cn/Article/details/4466660.htm
m.blog.hxbsg.cn/Article/details/9155995.htm
m.blog.hxbsg.cn/Article/details/2422864.htm
m.blog.hxbsg.cn/Article/details/3375317.htm
m.blog.hxbsg.cn/Article/details/3937979.htm
m.blog.hxbsg.cn/Article/details/5317531.htm
m.blog.hxbsg.cn/Article/details/0808468.htm
m.blog.hxbsg.cn/Article/details/8662642.htm
m.blog.hxbsg.cn/Article/details/9175195.htm
m.blog.hxbsg.cn/Article/details/0280800.htm
m.blog.hxbsg.cn/Article/details/9533577.htm
m.blog.hxbsg.cn/Article/details/3957971.htm
m.blog.hxbsg.cn/Article/details/1595733.htm
m.blog.hxbsg.cn/Article/details/1913359.htm
m.blog.hxbsg.cn/Article/details/9959713.htm
m.blog.hxbsg.cn/Article/details/7991391.htm
m.blog.hxbsg.cn/Article/details/8280682.htm
m.blog.hxbsg.cn/Article/details/9113975.htm
m.blog.hxbsg.cn/Article/details/1779135.htm
m.blog.hxbsg.cn/Article/details/2806628.htm
m.blog.hxbsg.cn/Article/details/1713937.htm
m.blog.hxbsg.cn/Article/details/5135999.htm
m.blog.hxbsg.cn/Article/details/4244826.htm
m.blog.hxbsg.cn/Article/details/5993559.htm
m.blog.hxbsg.cn/Article/details/3551919.htm
m.blog.hxbsg.cn/Article/details/1375375.htm
m.blog.hxbsg.cn/Article/details/5579131.htm
m.blog.hxbsg.cn/Article/details/7953337.htm
m.blog.hxbsg.cn/Article/details/6626044.htm
m.blog.hxbsg.cn/Article/details/3573137.htm
m.blog.hxbsg.cn/Article/details/3535551.htm
m.blog.hxbsg.cn/Article/details/3739735.htm
m.blog.hxbsg.cn/Article/details/3593957.htm
m.blog.hxbsg.cn/Article/details/8086408.htm
m.blog.hxbsg.cn/Article/details/5371913.htm
m.blog.hxbsg.cn/Article/details/1531715.htm
m.blog.hxbsg.cn/Article/details/8842406.htm
m.blog.hxbsg.cn/Article/details/4286600.htm
m.blog.hxbsg.cn/Article/details/4024282.htm
m.blog.hxbsg.cn/Article/details/9353113.htm
m.blog.hxbsg.cn/Article/details/3753773.htm
m.blog.hxbsg.cn/Article/details/3915359.htm
m.blog.hxbsg.cn/Article/details/6626282.htm
m.blog.hxbsg.cn/Article/details/8248020.htm
m.blog.hxbsg.cn/Article/details/4604604.htm
m.blog.hxbsg.cn/Article/details/7531579.htm
m.blog.hxbsg.cn/Article/details/5395579.htm
m.blog.hxbsg.cn/Article/details/6662402.htm
m.blog.hxbsg.cn/Article/details/1377911.htm
m.blog.hxbsg.cn/Article/details/1533157.htm
m.blog.hxbsg.cn/Article/details/0820828.htm
m.blog.hxbsg.cn/Article/details/9731137.htm
m.blog.hxbsg.cn/Article/details/7579975.htm
m.blog.hxbsg.cn/Article/details/6202606.htm
m.blog.hxbsg.cn/Article/details/3197111.htm
m.blog.hxbsg.cn/Article/details/9915395.htm
m.blog.hxbsg.cn/Article/details/5391355.htm
m.blog.hxbsg.cn/Article/details/5193517.htm
m.blog.hxbsg.cn/Article/details/7755591.htm
m.blog.hxbsg.cn/Article/details/3153313.htm
m.blog.hxbsg.cn/Article/details/7995977.htm
m.blog.hxbsg.cn/Article/details/7115917.htm
m.blog.hxbsg.cn/Article/details/3555191.htm
m.blog.hxbsg.cn/Article/details/7713739.htm
m.blog.hxbsg.cn/Article/details/1155397.htm
m.blog.hxbsg.cn/Article/details/7377917.htm
m.blog.hxbsg.cn/Article/details/1973197.htm
m.blog.hxbsg.cn/Article/details/8468086.htm
m.blog.hxbsg.cn/Article/details/1771151.htm
m.blog.hxbsg.cn/Article/details/1757537.htm
m.blog.hxbsg.cn/Article/details/8866080.htm
m.blog.hxbsg.cn/Article/details/3159939.htm
m.blog.hxbsg.cn/Article/details/9579973.htm
m.blog.hxbsg.cn/Article/details/3151953.htm
m.blog.hxbsg.cn/Article/details/1111555.htm
m.blog.hxbsg.cn/Article/details/8268422.htm
m.blog.hxbsg.cn/Article/details/1351331.htm
m.blog.hxbsg.cn/Article/details/2468002.htm
m.blog.hxbsg.cn/Article/details/1995995.htm
m.blog.hxbsg.cn/Article/details/7975373.htm
m.blog.hxbsg.cn/Article/details/5119791.htm
m.blog.hxbsg.cn/Article/details/6804284.htm
m.blog.hxbsg.cn/Article/details/8208604.htm
m.blog.hxbsg.cn/Article/details/1531759.htm
m.blog.hxbsg.cn/Article/details/5393733.htm
m.blog.hxbsg.cn/Article/details/8622826.htm
m.blog.hxbsg.cn/Article/details/4882080.htm
m.blog.hxbsg.cn/Article/details/7959335.htm
m.blog.hxbsg.cn/Article/details/5773353.htm
m.blog.hxbsg.cn/Article/details/8664480.htm
m.blog.hxbsg.cn/Article/details/6822480.htm
m.blog.hxbsg.cn/Article/details/7113731.htm
m.blog.hxbsg.cn/Article/details/5391391.htm
m.blog.hxbsg.cn/Article/details/4604428.htm
m.blog.hxbsg.cn/Article/details/8488844.htm
m.blog.hxbsg.cn/Article/details/2822066.htm
m.blog.hxbsg.cn/Article/details/7311775.htm
m.blog.hxbsg.cn/Article/details/7977953.htm
m.blog.hxbsg.cn/Article/details/5357933.htm
m.blog.hxbsg.cn/Article/details/4626468.htm
m.blog.hxbsg.cn/Article/details/0640024.htm
m.blog.hxbsg.cn/Article/details/3319915.htm
m.blog.hxbsg.cn/Article/details/8880088.htm
m.blog.hxbsg.cn/Article/details/5575513.htm
m.blog.hxbsg.cn/Article/details/5911377.htm
m.blog.hxbsg.cn/Article/details/7913155.htm
m.blog.hxbsg.cn/Article/details/0626000.htm
m.blog.hxbsg.cn/Article/details/1735377.htm
m.blog.hxbsg.cn/Article/details/1751733.htm
m.blog.hxbsg.cn/Article/details/1311131.htm
m.blog.hxbsg.cn/Article/details/2664482.htm
m.blog.hxbsg.cn/Article/details/3959975.htm
m.blog.hxbsg.cn/Article/details/8642088.htm
m.blog.hxbsg.cn/Article/details/5591931.htm
m.blog.hxbsg.cn/Article/details/9351993.htm
m.blog.hxbsg.cn/Article/details/4820804.htm
m.blog.hxbsg.cn/Article/details/3155713.htm
m.blog.hxbsg.cn/Article/details/6660264.htm
m.blog.hxbsg.cn/Article/details/5517555.htm
m.blog.hxbsg.cn/Article/details/9915159.htm
m.blog.hxbsg.cn/Article/details/7199179.htm
m.blog.hxbsg.cn/Article/details/3573575.htm
m.blog.hxbsg.cn/Article/details/5793177.htm
m.blog.hxbsg.cn/Article/details/1733151.htm
m.blog.hxbsg.cn/Article/details/0406246.htm
m.blog.hxbsg.cn/Article/details/4826804.htm
m.blog.hxbsg.cn/Article/details/3137139.htm
m.blog.hxbsg.cn/Article/details/6684484.htm
m.blog.hxbsg.cn/Article/details/3717915.htm
m.blog.hxbsg.cn/Article/details/4848226.htm
m.blog.hxbsg.cn/Article/details/3559559.htm
m.blog.hxbsg.cn/Article/details/9571391.htm
m.blog.hxbsg.cn/Article/details/1577395.htm
m.blog.hxbsg.cn/Article/details/7371193.htm
m.blog.hxbsg.cn/Article/details/8882686.htm
m.blog.hxbsg.cn/Article/details/9339393.htm
m.blog.hxbsg.cn/Article/details/1175517.htm
m.blog.hxbsg.cn/Article/details/0002406.htm
m.blog.hxbsg.cn/Article/details/0062268.htm
m.blog.hxbsg.cn/Article/details/5393999.htm
m.blog.hxbsg.cn/Article/details/1391973.htm
m.blog.hxbsg.cn/Article/details/1391511.htm
m.blog.hxbsg.cn/Article/details/0824408.htm
m.blog.hxbsg.cn/Article/details/6822660.htm
m.blog.hxbsg.cn/Article/details/6428460.htm
m.blog.hxbsg.cn/Article/details/7399579.htm
m.blog.hxbsg.cn/Article/details/5953733.htm
m.blog.hxbsg.cn/Article/details/7519937.htm
m.blog.hxbsg.cn/Article/details/0024060.htm
m.blog.hxbsg.cn/Article/details/4420486.htm
m.blog.hxbsg.cn/Article/details/0668288.htm
m.blog.hxbsg.cn/Article/details/8842866.htm
m.blog.hxbsg.cn/Article/details/4408806.htm
m.blog.hxbsg.cn/Article/details/6068860.htm
m.blog.hxbsg.cn/Article/details/3779193.htm
m.blog.hxbsg.cn/Article/details/7331179.htm
m.blog.hxbsg.cn/Article/details/7535357.htm
m.blog.hxbsg.cn/Article/details/9397597.htm
m.blog.hxbsg.cn/Article/details/0624648.htm
m.blog.hxbsg.cn/Article/details/8868420.htm
m.blog.hxbsg.cn/Article/details/5755153.htm
m.blog.hxbsg.cn/Article/details/5739117.htm
m.blog.hxbsg.cn/Article/details/6044028.htm
m.blog.hxbsg.cn/Article/details/3375737.htm
m.blog.hxbsg.cn/Article/details/8866686.htm
m.blog.hxbsg.cn/Article/details/1795375.htm
m.blog.hxbsg.cn/Article/details/2422880.htm
m.blog.hxbsg.cn/Article/details/8644406.htm
m.blog.hxbsg.cn/Article/details/8448622.htm
m.blog.hxbsg.cn/Article/details/3175791.htm
m.blog.hxbsg.cn/Article/details/2224444.htm
m.blog.hxbsg.cn/Article/details/3373535.htm
m.blog.hxbsg.cn/Article/details/9933755.htm
m.blog.hxbsg.cn/Article/details/3797939.htm
m.blog.hxbsg.cn/Article/details/8082882.htm
m.blog.hxbsg.cn/Article/details/8682886.htm
m.blog.hxbsg.cn/Article/details/5999317.htm
m.blog.hxbsg.cn/Article/details/9579513.htm
m.blog.hxbsg.cn/Article/details/0046662.htm
m.blog.hxbsg.cn/Article/details/7139717.htm
m.blog.hxbsg.cn/Article/details/3331917.htm
m.blog.hxbsg.cn/Article/details/2600682.htm
m.blog.hxbsg.cn/Article/details/7917195.htm
m.blog.hxbsg.cn/Article/details/7937531.htm
m.blog.hxbsg.cn/Article/details/0228064.htm
m.blog.hxbsg.cn/Article/details/6228462.htm
m.blog.hxbsg.cn/Article/details/2860884.htm
m.blog.hxbsg.cn/Article/details/9751575.htm
m.blog.hxbsg.cn/Article/details/8200404.htm
m.blog.hxbsg.cn/Article/details/9531933.htm
m.blog.hxbsg.cn/Article/details/7351591.htm
m.blog.hxbsg.cn/Article/details/8860024.htm
m.blog.hxbsg.cn/Article/details/2440882.htm
m.blog.hxbsg.cn/Article/details/0824820.htm
m.blog.hxbsg.cn/Article/details/8022288.htm
m.blog.hxbsg.cn/Article/details/1959375.htm
m.blog.hxbsg.cn/Article/details/9773759.htm
m.blog.hxbsg.cn/Article/details/6240888.htm
m.blog.hxbsg.cn/Article/details/8262066.htm
m.blog.hxbsg.cn/Article/details/8082220.htm
m.blog.hxbsg.cn/Article/details/2820808.htm
m.blog.hxbsg.cn/Article/details/3773959.htm
m.blog.hxbsg.cn/Article/details/4888042.htm
m.blog.hxbsg.cn/Article/details/4060004.htm
m.blog.hxbsg.cn/Article/details/1779973.htm
m.blog.hxbsg.cn/Article/details/6482260.htm
m.blog.hxbsg.cn/Article/details/6080448.htm
m.blog.hxbsg.cn/Article/details/2022464.htm
m.blog.hxbsg.cn/Article/details/4862080.htm
m.blog.hxbsg.cn/Article/details/7757515.htm
m.blog.hxbsg.cn/Article/details/3153757.htm
m.blog.hxbsg.cn/Article/details/8422860.htm
m.blog.hxbsg.cn/Article/details/4442206.htm
m.blog.hxbsg.cn/Article/details/9737111.htm
m.blog.hxbsg.cn/Article/details/8286448.htm
m.blog.hxbsg.cn/Article/details/6264280.htm
m.blog.hxbsg.cn/Article/details/5331797.htm
m.blog.hxbsg.cn/Article/details/6826020.htm
m.blog.hxbsg.cn/Article/details/7757157.htm
m.blog.hxbsg.cn/Article/details/9931379.htm
