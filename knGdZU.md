蛊辜俨蹿众


  
应使用 ArrayDeque 而非 Stack 由于其性能更好，设计更符合栈语义；ArrayDeque 底层为循环数组，均摊 O(1)无同步费用，需要使用 Deque 接口声明，避免混合双端方法。

为什么不用 Stack 类，而推荐 ArrayDeque

Java 的 Stack 类是遗留类，继承自己 Vector，所有操作都加了同步锁，性能差；更重要的是，它违反了“栈只应该暴露” push/pop/peek"设计原则——Stack 还能用 get(0)、remove(2) 等待随机访问，不小心写出非栈逻辑。官方文件明确建议使用 Deque 实现栈，而 ArrayDeque 最常用、性能最好的实现之一。
常见错误现象：Stack 在多线程下看似“安全”，但实际同步粒度过粗，现代并发场景早就应该使用了 ConcurrentLinkedDeque 或者显式锁；还有人用 LinkedList 当栈，虽然已经实现了 Deque，但每次 push 所有新节点，内存费用和内存费用 GC 压力比 ArrayDeque 高得多。


ArrayDeque 底层为循环数组，push/pop 均摊时间的复杂性 O(1)无同步费用
必须用 Deque 接口变量声明，而不是 ArrayDeque 具体类型易于替换和测试
不要调用 addFirst/removeFirst 等待双端队列方法-其语义不清，应统一使用栈专用方法


push 和 pop 空栈陷阱的正确写法和正确写法
ArrayDeque 作为栈用时，push 等价于 addFirst，pop 等价于 removeFirst，两者都操作队头(即逻辑栈顶)。但是 pop 在栈为空时抛掷 NoSuchElementException，不是 NullPointerException 或返回 null —— 这很容易被忽视，导致在线崩溃。
典型使用场景：表达式求值、括号匹配、DFS 递归转迭代。在这个时候，你必须先检查它是否是空的 pop，或者用 peek 预判。
；

安全写法：先 if (!stack.isEmpty()) { stack.pop(); }，别依赖 try-catch 捕获 NoSuchElementException 来控制流程

peek 返回栈顶元素但不移除，空时返回 null(注:如果堆栈存放，可能是 null 需要额外判断的引用类型)
避免混用:不要在同一个栈实例上交替调用 push 和 addLast，会破坏栈序


ArrayDeque 容量增长机制和初始容量设置
ArrayDeque 默认结构容量为 16.每次扩容为原容量 2 倍（不是 1.5 倍)，并且要求容量始终为 2 扩展涉及数组复制，尽管均摊 O(1)但大量突发事件 push 它仍然可能导致短暂停顿。如果你大概知道栈的最大深度(比如分析嵌套 JSON 不超过 1000 层)，直接指定初始容量更稳定。


性能影响：小栈(10k)和频繁扩容时，GC 压力和内存碎片会上升。

初始写法：Deque&lt;integer&gt; stack = new ArrayDeque(2048);&lt;/integer&gt;，输入值将自动提升到最近 2 的幂（如 2048 就是，2000 会被提升为 2048）
不能传 0 或负数，否则抛 IllegalArgumentException

别用 new ArrayDeque() 后立刻 ensureCapacity——没有这种方法，扩容完全由内部触发

如何处理泛型擦除下的原始类型？ null 值
ArrayDeque 是泛型，但是 Java 在泛运行过程中擦除，因此其底层存储在 Object[]。这意味着你不能直接存储:你不能直接存储 int，必须用 Integer；如果业务逻辑允许栈中出现， null(例如占位符)，peek 返回 null 无法区分“栈空”和“栈顶” null”。
常见错误：把 int 往 Deque&lt;integer&gt;&lt;/integer&gt; 里 push，看似正常，实际上每次自动包装，对象在高频操作下创建成本明显；或使用 if (stack.peek() == null) 判断空栈，结果栈不是空的，而是顶部 null，逻辑错乱。

专用容器优先考虑原始类型(如原始类型) IntStack 第三方库)或自行包装一层，避免重复包装
若必须支持 null 元素，改用 isEmpty() 判断栈空，而不是依靠 peek() 返回值
不要在栈里存可变对象(如 ArrayList）之后还对其进行了外部修改——栈内引用没有改变，但是内容已经不一样了，容易造成隐蔽 bug

真正难的不是写对那些行。 push/pop，而是想清楚:这个栈会跨线程共享吗？你想支持吗？ null？最大深度有界限吗？这些决定真的影响了选择和容错设计。	 

刂滩僮枚凶炮胸姆宦露僮翟雅氐刃

wpu.24d38z2.cn/248085.htm
wpu.24d38z2.cn/008025.htm
wpu.24d38z2.cn/315115.htm
wpy.ag5qr6tv.cn/355795.htm
wpy.ag5qr6tv.cn/939595.htm
wpy.ag5qr6tv.cn/266025.htm
wpy.ag5qr6tv.cn/802265.htm
wpy.ag5qr6tv.cn/533755.htm
wpy.ag5qr6tv.cn/808265.htm
wpy.ag5qr6tv.cn/226625.htm
wpy.ag5qr6tv.cn/335975.htm
wpy.ag5qr6tv.cn/200245.htm
wpy.ag5qr6tv.cn/880245.htm
wpt.ag5qr6tv.cn/042445.htm
wpt.ag5qr6tv.cn/991555.htm
wpt.ag5qr6tv.cn/826005.htm
wpt.ag5qr6tv.cn/888625.htm
wpt.ag5qr6tv.cn/208665.htm
wpt.ag5qr6tv.cn/080085.htm
wpt.ag5qr6tv.cn/200865.htm
wpt.ag5qr6tv.cn/202465.htm
wpt.ag5qr6tv.cn/595735.htm
wpt.ag5qr6tv.cn/468885.htm
wpr.ag5qr6tv.cn/446825.htm
wpr.ag5qr6tv.cn/133135.htm
wpr.ag5qr6tv.cn/204865.htm
wpr.ag5qr6tv.cn/440025.htm
wpr.ag5qr6tv.cn/064285.htm
wpr.ag5qr6tv.cn/133155.htm
wpr.ag5qr6tv.cn/246485.htm
wpr.ag5qr6tv.cn/193795.htm
wpr.ag5qr6tv.cn/311775.htm
wpr.ag5qr6tv.cn/864665.htm
wpe.ag5qr6tv.cn/622465.htm
wpe.ag5qr6tv.cn/028405.htm
wpe.ag5qr6tv.cn/151715.htm
wpe.ag5qr6tv.cn/402645.htm
wpe.ag5qr6tv.cn/373915.htm
wpe.ag5qr6tv.cn/262625.htm
wpe.ag5qr6tv.cn/866425.htm
wpe.ag5qr6tv.cn/797955.htm
wpe.ag5qr6tv.cn/664645.htm
wpe.ag5qr6tv.cn/268465.htm
wpw.ag5qr6tv.cn/862005.htm
wpw.ag5qr6tv.cn/862245.htm
wpw.ag5qr6tv.cn/046825.htm
wpw.ag5qr6tv.cn/222405.htm
wpw.ag5qr6tv.cn/462825.htm
wpw.ag5qr6tv.cn/002065.htm
wpw.ag5qr6tv.cn/799935.htm
wpw.ag5qr6tv.cn/226425.htm
wpw.ag5qr6tv.cn/646825.htm
wpw.ag5qr6tv.cn/913195.htm
wpq.ag5qr6tv.cn/480065.htm
wpq.ag5qr6tv.cn/680805.htm
wpq.ag5qr6tv.cn/622685.htm
wpq.ag5qr6tv.cn/771935.htm
wpq.ag5qr6tv.cn/800245.htm
wpq.ag5qr6tv.cn/997735.htm
wpq.ag5qr6tv.cn/971195.htm
wpq.ag5qr6tv.cn/206625.htm
wpq.ag5qr6tv.cn/646665.htm
wpq.ag5qr6tv.cn/573155.htm
wotv.ag5qr6tv.cn/040885.htm
wotv.ag5qr6tv.cn/977175.htm
wotv.ag5qr6tv.cn/939155.htm
wotv.ag5qr6tv.cn/779375.htm
wotv.ag5qr6tv.cn/686025.htm
wotv.ag5qr6tv.cn/951355.htm
wotv.ag5qr6tv.cn/220465.htm
wotv.ag5qr6tv.cn/600005.htm
wotv.ag5qr6tv.cn/026205.htm
wotv.ag5qr6tv.cn/799555.htm
won.ag5qr6tv.cn/206045.htm
won.ag5qr6tv.cn/446885.htm
won.ag5qr6tv.cn/117755.htm
won.ag5qr6tv.cn/208805.htm
won.ag5qr6tv.cn/664685.htm
won.ag5qr6tv.cn/608045.htm
won.ag5qr6tv.cn/911955.htm
won.ag5qr6tv.cn/026885.htm
won.ag5qr6tv.cn/393735.htm
won.ag5qr6tv.cn/939355.htm
wob.ag5qr6tv.cn/024005.htm
wob.ag5qr6tv.cn/977395.htm
wob.ag5qr6tv.cn/024805.htm
wob.ag5qr6tv.cn/442645.htm
wob.ag5qr6tv.cn/668445.htm
wob.ag5qr6tv.cn/026885.htm
wob.ag5qr6tv.cn/531755.htm
wob.ag5qr6tv.cn/808825.htm
wob.ag5qr6tv.cn/200065.htm
wob.ag5qr6tv.cn/022625.htm
wov.ag5qr6tv.cn/604605.htm
wov.ag5qr6tv.cn/440265.htm
wov.ag5qr6tv.cn/139735.htm
wov.ag5qr6tv.cn/862065.htm
wov.ag5qr6tv.cn/000645.htm
wov.ag5qr6tv.cn/468645.htm
wov.ag5qr6tv.cn/808625.htm
wov.ag5qr6tv.cn/533535.htm
wov.ag5qr6tv.cn/266045.htm
wov.ag5qr6tv.cn/242265.htm
woc.ag5qr6tv.cn/008625.htm
woc.ag5qr6tv.cn/622065.htm
woc.ag5qr6tv.cn/242485.htm
woc.ag5qr6tv.cn/886025.htm
woc.ag5qr6tv.cn/444025.htm
woc.ag5qr6tv.cn/604825.htm
woc.ag5qr6tv.cn/446845.htm
woc.ag5qr6tv.cn/460645.htm
woc.ag5qr6tv.cn/139515.htm
woc.ag5qr6tv.cn/135535.htm
wox.ag5qr6tv.cn/399195.htm
wox.ag5qr6tv.cn/862405.htm
wox.ag5qr6tv.cn/200605.htm
wox.ag5qr6tv.cn/402405.htm
wox.ag5qr6tv.cn/820845.htm
wox.ag5qr6tv.cn/131735.htm
wox.ag5qr6tv.cn/206285.htm
wox.ag5qr6tv.cn/080865.htm
wox.ag5qr6tv.cn/228665.htm
wox.ag5qr6tv.cn/044005.htm
woz.ag5qr6tv.cn/713135.htm
woz.ag5qr6tv.cn/264225.htm
woz.ag5qr6tv.cn/684285.htm
woz.ag5qr6tv.cn/048645.htm
woz.ag5qr6tv.cn/937795.htm
woz.ag5qr6tv.cn/735335.htm
woz.ag5qr6tv.cn/175995.htm
woz.ag5qr6tv.cn/226805.htm
woz.ag5qr6tv.cn/333555.htm
woz.ag5qr6tv.cn/713795.htm
wol.ag5qr6tv.cn/668665.htm
wol.ag5qr6tv.cn/804005.htm
wol.ag5qr6tv.cn/888825.htm
wol.ag5qr6tv.cn/080665.htm
wol.ag5qr6tv.cn/242465.htm
wol.ag5qr6tv.cn/204005.htm
wol.ag5qr6tv.cn/460425.htm
wol.ag5qr6tv.cn/208285.htm
wol.ag5qr6tv.cn/973755.htm
wol.ag5qr6tv.cn/531135.htm
wok.ag5qr6tv.cn/282205.htm
wok.ag5qr6tv.cn/860805.htm
wok.ag5qr6tv.cn/888625.htm
wok.ag5qr6tv.cn/468845.htm
wok.ag5qr6tv.cn/426065.htm
wok.ag5qr6tv.cn/197115.htm
wok.ag5qr6tv.cn/200065.htm
wok.ag5qr6tv.cn/024885.htm
wok.ag5qr6tv.cn/282685.htm
wok.ag5qr6tv.cn/882065.htm
woj.ag5qr6tv.cn/024065.htm
woj.ag5qr6tv.cn/626045.htm
woj.ag5qr6tv.cn/262225.htm
woj.ag5qr6tv.cn/244265.htm
woj.ag5qr6tv.cn/535755.htm
woj.ag5qr6tv.cn/775175.htm
woj.ag5qr6tv.cn/808685.htm
woj.ag5qr6tv.cn/222865.htm
woj.ag5qr6tv.cn/802845.htm
woj.ag5qr6tv.cn/711315.htm
woh.ag5qr6tv.cn/242645.htm
woh.ag5qr6tv.cn/080425.htm
woh.ag5qr6tv.cn/399555.htm
woh.ag5qr6tv.cn/715915.htm
woh.ag5qr6tv.cn/424805.htm
woh.ag5qr6tv.cn/804405.htm
woh.ag5qr6tv.cn/993795.htm
woh.ag5qr6tv.cn/668605.htm
woh.ag5qr6tv.cn/406285.htm
woh.ag5qr6tv.cn/919135.htm
wog.ag5qr6tv.cn/002085.htm
wog.ag5qr6tv.cn/406225.htm
wog.ag5qr6tv.cn/662605.htm
wog.ag5qr6tv.cn/795575.htm
wog.ag5qr6tv.cn/119795.htm
wog.ag5qr6tv.cn/644085.htm
wog.ag5qr6tv.cn/157755.htm
wog.ag5qr6tv.cn/224825.htm
wog.ag5qr6tv.cn/662605.htm
wog.ag5qr6tv.cn/684025.htm
wof.ag5qr6tv.cn/220885.htm
wof.ag5qr6tv.cn/313355.htm
wof.ag5qr6tv.cn/406865.htm
wof.ag5qr6tv.cn/599155.htm
wof.ag5qr6tv.cn/604865.htm
wof.ag5qr6tv.cn/028405.htm
wof.ag5qr6tv.cn/880065.htm
wof.ag5qr6tv.cn/373555.htm
wof.ag5qr6tv.cn/686865.htm
wof.ag5qr6tv.cn/733915.htm
wod.ag5qr6tv.cn/755915.htm
wod.ag5qr6tv.cn/886885.htm
wod.ag5qr6tv.cn/995795.htm
wod.ag5qr6tv.cn/595135.htm
wod.ag5qr6tv.cn/822665.htm
wod.ag5qr6tv.cn/262265.htm
wod.ag5qr6tv.cn/008825.htm
wod.ag5qr6tv.cn/446685.htm
wod.ag5qr6tv.cn/422645.htm
wod.ag5qr6tv.cn/286885.htm
wos.ag5qr6tv.cn/931395.htm
wos.ag5qr6tv.cn/488205.htm
wos.ag5qr6tv.cn/684025.htm
wos.ag5qr6tv.cn/399755.htm
wos.ag5qr6tv.cn/042685.htm
wos.ag5qr6tv.cn/226205.htm
wos.ag5qr6tv.cn/460885.htm
wos.ag5qr6tv.cn/208205.htm
wos.ag5qr6tv.cn/822405.htm
wos.ag5qr6tv.cn/735935.htm
woa.ag5qr6tv.cn/286445.htm
woa.ag5qr6tv.cn/006865.htm
woa.ag5qr6tv.cn/773395.htm
woa.ag5qr6tv.cn/042825.htm
woa.ag5qr6tv.cn/062485.htm
woa.ag5qr6tv.cn/620885.htm
woa.ag5qr6tv.cn/608465.htm
woa.ag5qr6tv.cn/480285.htm
woa.ag5qr6tv.cn/064085.htm
woa.ag5qr6tv.cn/024085.htm
wop.ag5qr6tv.cn/393375.htm
wop.ag5qr6tv.cn/880865.htm
wop.ag5qr6tv.cn/404825.htm
wop.ag5qr6tv.cn/606005.htm
wop.ag5qr6tv.cn/911155.htm
wop.ag5qr6tv.cn/719175.htm
wop.ag5qr6tv.cn/668825.htm
wop.ag5qr6tv.cn/997175.htm
wop.ag5qr6tv.cn/488205.htm
wop.ag5qr6tv.cn/082885.htm
woo.ag5qr6tv.cn/244845.htm
woo.ag5qr6tv.cn/173555.htm
woo.ag5qr6tv.cn/028485.htm
woo.ag5qr6tv.cn/448405.htm
woo.ag5qr6tv.cn/359795.htm
woo.ag5qr6tv.cn/684805.htm
woo.ag5qr6tv.cn/860465.htm
woo.ag5qr6tv.cn/828245.htm
woo.ag5qr6tv.cn/571175.htm
woo.ag5qr6tv.cn/428485.htm
woi.ag5qr6tv.cn/719555.htm
woi.ag5qr6tv.cn/046425.htm
woi.ag5qr6tv.cn/844885.htm
woi.ag5qr6tv.cn/602805.htm
woi.ag5qr6tv.cn/777555.htm
woi.ag5qr6tv.cn/402645.htm
woi.ag5qr6tv.cn/208085.htm
woi.ag5qr6tv.cn/080045.htm
woi.ag5qr6tv.cn/799195.htm
woi.ag5qr6tv.cn/995175.htm
wou.ag5qr6tv.cn/755395.htm
wou.ag5qr6tv.cn/533115.htm
wou.ag5qr6tv.cn/622665.htm
wou.ag5qr6tv.cn/028605.htm
wou.ag5qr6tv.cn/335595.htm
wou.ag5qr6tv.cn/466085.htm
wou.ag5qr6tv.cn/222065.htm
wou.ag5qr6tv.cn/537555.htm
wou.ag5qr6tv.cn/579155.htm
wou.ag5qr6tv.cn/880425.htm
woy.ag5qr6tv.cn/559995.htm
woy.ag5qr6tv.cn/088605.htm
woy.ag5qr6tv.cn/993535.htm
woy.ag5qr6tv.cn/842465.htm
woy.ag5qr6tv.cn/068205.htm
woy.ag5qr6tv.cn/557775.htm
woy.ag5qr6tv.cn/804405.htm
woy.ag5qr6tv.cn/391955.htm
woy.ag5qr6tv.cn/426845.htm
woy.ag5qr6tv.cn/228045.htm
wot.ag5qr6tv.cn/684025.htm
wot.ag5qr6tv.cn/068845.htm
wot.ag5qr6tv.cn/331115.htm
wot.ag5qr6tv.cn/664865.htm
wot.ag5qr6tv.cn/113395.htm
wot.ag5qr6tv.cn/448685.htm
wot.ag5qr6tv.cn/484865.htm
wot.ag5qr6tv.cn/202065.htm
wot.ag5qr6tv.cn/915355.htm
wot.ag5qr6tv.cn/668005.htm
wor.ag5qr6tv.cn/553175.htm
wor.ag5qr6tv.cn/137735.htm
wor.ag5qr6tv.cn/684805.htm
wor.ag5qr6tv.cn/511735.htm
wor.ag5qr6tv.cn/802425.htm
wor.ag5qr6tv.cn/808405.htm
wor.ag5qr6tv.cn/468825.htm
wor.ag5qr6tv.cn/115715.htm
wor.ag5qr6tv.cn/262805.htm
wor.ag5qr6tv.cn/824265.htm
woe.ag5qr6tv.cn/315335.htm
woe.ag5qr6tv.cn/660245.htm
woe.ag5qr6tv.cn/840805.htm
woe.ag5qr6tv.cn/644885.htm
woe.ag5qr6tv.cn/133955.htm
woe.ag5qr6tv.cn/606885.htm
woe.ag5qr6tv.cn/337535.htm
woe.ag5qr6tv.cn/042465.htm
woe.ag5qr6tv.cn/426825.htm
woe.ag5qr6tv.cn/008025.htm
wow.ag5qr6tv.cn/460665.htm
wow.ag5qr6tv.cn/466825.htm
wow.ag5qr6tv.cn/048885.htm
wow.ag5qr6tv.cn/711395.htm
wow.ag5qr6tv.cn/462825.htm
wow.ag5qr6tv.cn/955795.htm
wow.ag5qr6tv.cn/264445.htm
wow.ag5qr6tv.cn/242885.htm
wow.ag5qr6tv.cn/428825.htm
wow.ag5qr6tv.cn/828065.htm
woq.ag5qr6tv.cn/193995.htm
woq.ag5qr6tv.cn/575995.htm
woq.ag5qr6tv.cn/395935.htm
woq.ag5qr6tv.cn/159755.htm
woq.ag5qr6tv.cn/624065.htm
woq.ag5qr6tv.cn/086025.htm
woq.ag5qr6tv.cn/040425.htm
woq.ag5qr6tv.cn/797775.htm
woq.ag5qr6tv.cn/620865.htm
woq.ag5qr6tv.cn/955595.htm
witv.ag5qr6tv.cn/666845.htm
witv.ag5qr6tv.cn/260285.htm
witv.ag5qr6tv.cn/864405.htm
witv.ag5qr6tv.cn/717375.htm
witv.ag5qr6tv.cn/468645.htm
witv.ag5qr6tv.cn/202025.htm
witv.ag5qr6tv.cn/288865.htm
witv.ag5qr6tv.cn/844405.htm
witv.ag5qr6tv.cn/400045.htm
witv.ag5qr6tv.cn/195135.htm
win.ag5qr6tv.cn/288485.htm
win.ag5qr6tv.cn/951775.htm
win.ag5qr6tv.cn/864625.htm
win.ag5qr6tv.cn/226665.htm
win.ag5qr6tv.cn/868485.htm
win.ag5qr6tv.cn/715395.htm
win.ag5qr6tv.cn/799175.htm
win.ag5qr6tv.cn/008225.htm
win.ag5qr6tv.cn/842265.htm
win.ag5qr6tv.cn/844605.htm
wib.ag5qr6tv.cn/202845.htm
wib.ag5qr6tv.cn/026205.htm
wib.ag5qr6tv.cn/066025.htm
wib.ag5qr6tv.cn/519735.htm
wib.ag5qr6tv.cn/886825.htm
wib.ag5qr6tv.cn/177915.htm
wib.ag5qr6tv.cn/006205.htm
wib.ag5qr6tv.cn/244865.htm
wib.ag5qr6tv.cn/660605.htm
wib.ag5qr6tv.cn/844645.htm
wiv.ag5qr6tv.cn/462605.htm
wiv.ag5qr6tv.cn/686665.htm
wiv.ag5qr6tv.cn/442465.htm
wiv.ag5qr6tv.cn/266405.htm
wiv.ag5qr6tv.cn/802485.htm
wiv.ag5qr6tv.cn/468405.htm
wiv.ag5qr6tv.cn/682065.htm
wiv.ag5qr6tv.cn/288605.htm
wiv.ag5qr6tv.cn/351315.htm
wiv.ag5qr6tv.cn/771375.htm
wic.ag5qr6tv.cn/424645.htm
wic.ag5qr6tv.cn/680605.htm
wic.ag5qr6tv.cn/668445.htm
wic.ag5qr6tv.cn/137315.htm
wic.ag5qr6tv.cn/462645.htm
wic.ag5qr6tv.cn/668665.htm
wic.ag5qr6tv.cn/022065.htm
wic.ag5qr6tv.cn/068205.htm
wic.ag5qr6tv.cn/066825.htm
wic.ag5qr6tv.cn/022265.htm
wix.ag5qr6tv.cn/173355.htm
wix.ag5qr6tv.cn/244265.htm
wix.ag5qr6tv.cn/373555.htm
wix.ag5qr6tv.cn/640845.htm
wix.ag5qr6tv.cn/131515.htm
wix.ag5qr6tv.cn/046425.htm
wix.ag5qr6tv.cn/246865.htm
wix.ag5qr6tv.cn/000245.htm
wix.ag5qr6tv.cn/062285.htm
wix.ag5qr6tv.cn/884665.htm
wiz.ag5qr6tv.cn/446605.htm
wiz.ag5qr6tv.cn/804845.htm
wiz.ag5qr6tv.cn/864265.htm
wiz.ag5qr6tv.cn/680425.htm
wiz.ag5qr6tv.cn/995355.htm
wiz.ag5qr6tv.cn/420825.htm
wiz.ag5qr6tv.cn/002685.htm
wiz.ag5qr6tv.cn/608425.htm
wiz.ag5qr6tv.cn/624205.htm
wiz.ag5qr6tv.cn/313995.htm
wil.ag5qr6tv.cn/860245.htm
wil.ag5qr6tv.cn/266605.htm
