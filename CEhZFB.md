瞎驹盼约伦


  
LinkedHashmap维护插入(或访问)顺序，Hashmap不保证顺序；前者遍历按插入顺序，后者顺序不确定；LinkedHashmap内存略高，但序列化后顺序仍保留。Java 21 SequencedMap接口进一步明确顺序契约。

LinkedHashMap 插入顺序将得到维护，HashMap 不保证任何顺序
这是核心区别。当你经历它的时候 LinkedHashMap 的 keySet()、values() 或 entrySet() 当元素顺序严格按插入顺序返回时； HashMap 遍历顺序不确定，取决于哈希值、容量和扩容过程，甚至是同一次 JVM 在操作过程中多次遍历可能会有所不同。
常见错误现象：使用 HashMap 存储配置项或操作日志后，要“按添加顺序打印”，结果顺序混乱，误认为数据损坏。

若需要访问顺序(例如(例如) LRU 缓存），LinkedHashMap 可通过构造函数传入 true 启用：new LinkedHashMap(16, 0.75f, true)

HashMap 在 Java 8+ 链表长度中对 ≥ 8 桶数组长度 ≥ 64 它会变成红黑树，但优化不会影响顺序语义——它仍然不维护顺序
两者都允许 null 键和 null 值（LinkedHashMap 继承自 HashMap，行为一致）

LinkedHashMap 由于双向链表节点的额外维护，内存费用略大
LinkedHashMap 每个 Node 实际是 Entry 子类，比 HashMap 的 Node 多两个引用字段：before 和 after。这意味着每个键对多占用 16 字节（64 位 JVM，对象头 + 引用字段对齐后)。
影响场景:高频创建小集合(如单个请求中的几十个新集合) LinkedHashMap），内存压力明显高于 HashMap；但对于大多数业务场景(几百到几千元素)来说，这种差异是可以忽略的。
；

不要默认使用“看起来更标准” LinkedHashMap，没有顺序需求就使用 HashMap

若只读遍历频繁且需要顺序，LinkedHashMap 相反，迭代性能更稳定——HashMap 迭代需要跳过空桶，实际上是散列分布路径

序列化行为是一致的，但反序列化后 LinkedHashMap 仍保持顺序
两者都实现了 Serializable，顺序格式兼容。关键点是:LinkedHashMap 内部链表将在反序列化过程中重建，因此顺序信息将被完全保留； HashMap 反序列化后仍然是无序结构。
			
		
容易踩的坑:把 HashMap 存进 Redis 或写入 JSON 再读一遍，误以为“顺序可以回来”。其实 JSON 标准中不保证对象属性的顺序，Jackson 也不保留默认 HashMap 插入顺序(除非显式配置(除非显式配置) SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS 或换用 LinkedHashMap）。

如果跨过程/网络传输依赖顺序，则必须在接收端使用 LinkedHashMap 构造（例如 Jackson 指定类型的反序列化：mapper.readValue(json, new TypeReference>() {})
Spring Boot 配置绑定（@ConfigurationProperties）默认使用 LinkedHashMap 解析 YAML/Properties 中的 map，因此，通常可以保持配置顺序

替代方案：Java 21+ 的 SequencedMap 接口使顺序语义更加清晰
Java 21 引入了 SequencedMap，LinkedHashMap 它实现了，而 HashMap 没有。这意味着您可以使用接口编程来强调顺序合同：
SequencedMap map = new LinkedHashMap();
map.put("a", 1);
map.put("b", 2);
// 现可安全调用：
map.reversed(); // 返回逆序视图
map.getFirst(); // O(1) 获取首元素

但注意：SequencedMap 是接口，不是新实现；现有代码不需要更改，但是新的 API 设计值得考虑——它从具体实现了“顺序”（LinkedHashMap）将其升级为合同，未来其他有序 Map 实现也可以自然融合。
别指望 HashMap 有一天，突然支持顺序：其设计目标是高性能哈希搜索，顺序是权衡的特征。如果您真的想要顺序，您必须接受链表的维护成本或替换 TreeMap（按 key 排序，非插入序)。	 

用卸聪毡洞贺嗜捍尚谧浊品握浦宦

m.blog.hmybk.cn/Article/details/1917133.htm
m.blog.hmybk.cn/Article/details/7795713.htm
m.blog.hmybk.cn/Article/details/1779955.htm
m.blog.hmybk.cn/Article/details/8068800.htm
m.blog.hmybk.cn/Article/details/5753795.htm
m.blog.hmybk.cn/Article/details/2800060.htm
m.blog.hmybk.cn/Article/details/5551795.htm
m.blog.hmybk.cn/Article/details/9339555.htm
m.blog.hmybk.cn/Article/details/4200200.htm
m.blog.hmybk.cn/Article/details/1139575.htm
m.blog.hmybk.cn/Article/details/7577135.htm
m.blog.hmybk.cn/Article/details/5319135.htm
m.blog.hmybk.cn/Article/details/7913377.htm
m.blog.hmybk.cn/Article/details/5779777.htm
m.blog.hmybk.cn/Article/details/3377579.htm
m.blog.hmybk.cn/Article/details/7115779.htm
m.blog.hmybk.cn/Article/details/9511155.htm
m.blog.hmybk.cn/Article/details/7795575.htm
m.blog.hmybk.cn/Article/details/9779795.htm
m.blog.hmybk.cn/Article/details/7519399.htm
m.blog.hmybk.cn/Article/details/5513337.htm
m.blog.hmybk.cn/Article/details/3959317.htm
m.blog.hmybk.cn/Article/details/7757917.htm
m.blog.hmybk.cn/Article/details/1339331.htm
m.blog.hmybk.cn/Article/details/1575593.htm
m.blog.hmybk.cn/Article/details/6486264.htm
m.blog.hmybk.cn/Article/details/5135795.htm
m.blog.hmybk.cn/Article/details/7775999.htm
m.blog.hmybk.cn/Article/details/9751157.htm
m.blog.hmybk.cn/Article/details/9553391.htm
m.blog.hmybk.cn/Article/details/8820604.htm
m.blog.hmybk.cn/Article/details/6640624.htm
m.blog.hmybk.cn/Article/details/9159597.htm
m.blog.hmybk.cn/Article/details/3593955.htm
m.blog.hmybk.cn/Article/details/0402046.htm
m.blog.hmybk.cn/Article/details/7739371.htm
m.blog.hmybk.cn/Article/details/2804046.htm
m.blog.hmybk.cn/Article/details/9731937.htm
m.blog.hmybk.cn/Article/details/9337579.htm
m.blog.hmybk.cn/Article/details/0408868.htm
m.blog.hmybk.cn/Article/details/7195597.htm
m.blog.hmybk.cn/Article/details/2206006.htm
m.blog.hmybk.cn/Article/details/5377735.htm
m.blog.hmybk.cn/Article/details/1519535.htm
m.blog.hmybk.cn/Article/details/8066648.htm
m.blog.hmybk.cn/Article/details/5979719.htm
m.blog.hmybk.cn/Article/details/3751517.htm
m.blog.hmybk.cn/Article/details/7319933.htm
m.blog.hmybk.cn/Article/details/8066886.htm
m.blog.hmybk.cn/Article/details/7139539.htm
m.blog.hmybk.cn/Article/details/9159575.htm
m.blog.hmybk.cn/Article/details/9713375.htm
m.blog.hmybk.cn/Article/details/7193559.htm
m.blog.hmybk.cn/Article/details/9711595.htm
m.blog.hmybk.cn/Article/details/8448400.htm
m.blog.hmybk.cn/Article/details/3339711.htm
m.blog.hmybk.cn/Article/details/3313759.htm
m.blog.hmybk.cn/Article/details/2602620.htm
m.blog.hmybk.cn/Article/details/3371799.htm
m.blog.hmybk.cn/Article/details/8000242.htm
m.blog.hmybk.cn/Article/details/1137959.htm
m.blog.hmybk.cn/Article/details/0822820.htm
m.blog.hmybk.cn/Article/details/9937511.htm
m.blog.hmybk.cn/Article/details/9951355.htm
m.blog.hmybk.cn/Article/details/0824266.htm
m.blog.hmybk.cn/Article/details/7595931.htm
m.blog.hmybk.cn/Article/details/1197175.htm
m.blog.hmybk.cn/Article/details/6684868.htm
m.blog.hmybk.cn/Article/details/7995933.htm
m.blog.hmybk.cn/Article/details/3797373.htm
m.blog.hmybk.cn/Article/details/3991575.htm
m.blog.hmybk.cn/Article/details/1731151.htm
m.blog.hmybk.cn/Article/details/7931579.htm
m.blog.hmybk.cn/Article/details/3117515.htm
m.blog.hmybk.cn/Article/details/4600000.htm
m.blog.hmybk.cn/Article/details/1535575.htm
m.blog.hmybk.cn/Article/details/7179735.htm
m.blog.hmybk.cn/Article/details/4202828.htm
m.blog.hmybk.cn/Article/details/5999533.htm
m.blog.hmybk.cn/Article/details/9939917.htm
m.blog.hmybk.cn/Article/details/4066280.htm
m.blog.hmybk.cn/Article/details/9779939.htm
m.blog.hmybk.cn/Article/details/7375913.htm
m.blog.hmybk.cn/Article/details/3513379.htm
m.blog.hmybk.cn/Article/details/9153515.htm
m.blog.hmybk.cn/Article/details/8028024.htm
m.blog.hmybk.cn/Article/details/1511575.htm
m.blog.hmybk.cn/Article/details/3531751.htm
m.blog.hmybk.cn/Article/details/5555997.htm
m.blog.hmybk.cn/Article/details/7559791.htm
m.blog.hmybk.cn/Article/details/6426408.htm
m.blog.hmybk.cn/Article/details/1999937.htm
m.blog.hmybk.cn/Article/details/7953757.htm
m.blog.hmybk.cn/Article/details/3919399.htm
m.blog.hmybk.cn/Article/details/2040020.htm
m.blog.hmybk.cn/Article/details/3397351.htm
m.blog.hmybk.cn/Article/details/7973737.htm
m.blog.hmybk.cn/Article/details/6062624.htm
m.blog.hmybk.cn/Article/details/9995975.htm
m.blog.hmybk.cn/Article/details/7937795.htm
m.blog.hmybk.cn/Article/details/9193331.htm
m.blog.hmybk.cn/Article/details/5117119.htm
m.blog.hmybk.cn/Article/details/5519791.htm
m.blog.hmybk.cn/Article/details/9753351.htm
m.blog.hmybk.cn/Article/details/7951339.htm
m.blog.hmybk.cn/Article/details/1757375.htm
m.blog.hmybk.cn/Article/details/3395555.htm
m.blog.hmybk.cn/Article/details/5337737.htm
m.blog.hmybk.cn/Article/details/1333773.htm
m.blog.hmybk.cn/Article/details/7953351.htm
m.blog.hmybk.cn/Article/details/8824000.htm
m.blog.hmybk.cn/Article/details/9797131.htm
m.blog.hmybk.cn/Article/details/4422802.htm
m.blog.hmybk.cn/Article/details/5155151.htm
m.blog.hmybk.cn/Article/details/5579795.htm
m.blog.hmybk.cn/Article/details/3915759.htm
m.blog.hmybk.cn/Article/details/5591199.htm
m.blog.hmybk.cn/Article/details/9771917.htm
m.blog.hmybk.cn/Article/details/6228268.htm
m.blog.hmybk.cn/Article/details/7917131.htm
m.blog.hmybk.cn/Article/details/2448048.htm
m.blog.hmybk.cn/Article/details/0802808.htm
m.blog.hmybk.cn/Article/details/5919315.htm
m.blog.hmybk.cn/Article/details/7935379.htm
m.blog.hmybk.cn/Article/details/1193171.htm
m.blog.hmybk.cn/Article/details/4862402.htm
m.blog.hmybk.cn/Article/details/1919531.htm
m.blog.hmybk.cn/Article/details/1113957.htm
m.blog.hmybk.cn/Article/details/6408424.htm
m.blog.hmybk.cn/Article/details/5599579.htm
m.blog.hmybk.cn/Article/details/1597119.htm
m.blog.hmybk.cn/Article/details/6648842.htm
m.blog.hmybk.cn/Article/details/3551117.htm
m.blog.hmybk.cn/Article/details/6044006.htm
m.blog.hmybk.cn/Article/details/7577757.htm
m.blog.hmybk.cn/Article/details/0488066.htm
m.blog.hmybk.cn/Article/details/4280402.htm
m.blog.hmybk.cn/Article/details/3579979.htm
m.blog.hmybk.cn/Article/details/6824486.htm
m.blog.hmybk.cn/Article/details/1197131.htm
m.blog.hmybk.cn/Article/details/1753555.htm
m.blog.hmybk.cn/Article/details/0684844.htm
m.blog.hmybk.cn/Article/details/0668820.htm
m.blog.hmybk.cn/Article/details/3371337.htm
m.blog.hmybk.cn/Article/details/0800860.htm
m.blog.hmybk.cn/Article/details/6860026.htm
m.blog.hmybk.cn/Article/details/4022686.htm
m.blog.hmybk.cn/Article/details/8240642.htm
m.blog.hmybk.cn/Article/details/5537311.htm
m.blog.hmybk.cn/Article/details/7939971.htm
m.blog.hmybk.cn/Article/details/2008486.htm
m.blog.hmybk.cn/Article/details/7517999.htm
m.blog.hmybk.cn/Article/details/5799753.htm
m.blog.hmybk.cn/Article/details/7719513.htm
m.blog.hmybk.cn/Article/details/3515173.htm
m.blog.hmybk.cn/Article/details/2402662.htm
m.blog.hmybk.cn/Article/details/0204280.htm
m.blog.hmybk.cn/Article/details/8204264.htm
m.blog.hmybk.cn/Article/details/5315375.htm
m.blog.hmybk.cn/Article/details/0446022.htm
m.blog.hmybk.cn/Article/details/1337759.htm
m.blog.hmybk.cn/Article/details/3137779.htm
m.blog.hmybk.cn/Article/details/6008246.htm
m.blog.hmybk.cn/Article/details/5771115.htm
m.blog.hmybk.cn/Article/details/1535599.htm
m.blog.hmybk.cn/Article/details/8824060.htm
m.blog.hmybk.cn/Article/details/6822022.htm
m.blog.hmybk.cn/Article/details/3335957.htm
m.blog.hmybk.cn/Article/details/5513373.htm
m.blog.hmybk.cn/Article/details/5135771.htm
m.blog.hmybk.cn/Article/details/7995971.htm
m.blog.hmybk.cn/Article/details/0264448.htm
m.blog.hmybk.cn/Article/details/9575155.htm
m.blog.hmybk.cn/Article/details/0044486.htm
m.blog.hmybk.cn/Article/details/2440826.htm
m.blog.hmybk.cn/Article/details/3551373.htm
m.blog.hmybk.cn/Article/details/1195557.htm
m.blog.hmybk.cn/Article/details/5113517.htm
m.blog.hmybk.cn/Article/details/9933973.htm
m.blog.hmybk.cn/Article/details/7155177.htm
m.blog.hmybk.cn/Article/details/5195519.htm
m.blog.hmybk.cn/Article/details/4464006.htm
m.blog.hmybk.cn/Article/details/1155975.htm
m.blog.hmybk.cn/Article/details/1597339.htm
m.blog.hmybk.cn/Article/details/0600840.htm
m.blog.hmybk.cn/Article/details/3951919.htm
m.blog.hmybk.cn/Article/details/7591579.htm
m.blog.hmybk.cn/Article/details/8464044.htm
m.blog.hmybk.cn/Article/details/3591173.htm
m.blog.hmybk.cn/Article/details/7951799.htm
m.blog.hmybk.cn/Article/details/9373933.htm
m.blog.hmybk.cn/Article/details/4406804.htm
m.blog.hmybk.cn/Article/details/3539359.htm
m.blog.hmybk.cn/Article/details/9377739.htm
m.blog.hmybk.cn/Article/details/6026202.htm
m.blog.hmybk.cn/Article/details/0828262.htm
m.blog.hmybk.cn/Article/details/3773375.htm
m.blog.hmybk.cn/Article/details/6828464.htm
m.blog.hmybk.cn/Article/details/3377595.htm
m.blog.hmybk.cn/Article/details/7711599.htm
m.blog.hmybk.cn/Article/details/9535575.htm
m.blog.hmybk.cn/Article/details/9717135.htm
m.blog.hmybk.cn/Article/details/1913375.htm
m.blog.hmybk.cn/Article/details/9597757.htm
m.blog.hmybk.cn/Article/details/0208000.htm
m.blog.hmybk.cn/Article/details/4240864.htm
m.blog.hmybk.cn/Article/details/1933777.htm
m.blog.hmybk.cn/Article/details/1713733.htm
m.blog.hmybk.cn/Article/details/1157995.htm
m.blog.hmybk.cn/Article/details/7193791.htm
m.blog.hmybk.cn/Article/details/9117373.htm
m.blog.hmybk.cn/Article/details/1775991.htm
m.blog.hmybk.cn/Article/details/5311939.htm
m.blog.hmybk.cn/Article/details/3599979.htm
m.blog.hmybk.cn/Article/details/2648664.htm
m.blog.hmybk.cn/Article/details/5319171.htm
m.blog.hmybk.cn/Article/details/1379777.htm
m.blog.hmybk.cn/Article/details/7755973.htm
m.blog.hmybk.cn/Article/details/5975553.htm
m.blog.hmybk.cn/Article/details/5177997.htm
m.blog.hmybk.cn/Article/details/7571773.htm
m.blog.hmybk.cn/Article/details/3751155.htm
m.blog.hmybk.cn/Article/details/5577373.htm
m.blog.hmybk.cn/Article/details/2688024.htm
m.blog.hmybk.cn/Article/details/3399535.htm
m.blog.hmybk.cn/Article/details/3717599.htm
m.blog.hmybk.cn/Article/details/9513997.htm
m.blog.hmybk.cn/Article/details/2004882.htm
m.blog.hmybk.cn/Article/details/2268684.htm
m.blog.hmybk.cn/Article/details/5951377.htm
m.blog.hmybk.cn/Article/details/2680422.htm
m.blog.hmybk.cn/Article/details/7155511.htm
m.blog.hmybk.cn/Article/details/7359539.htm
m.blog.hmybk.cn/Article/details/7591733.htm
m.blog.hmybk.cn/Article/details/2480264.htm
m.blog.hmybk.cn/Article/details/6862606.htm
m.blog.hmybk.cn/Article/details/2004406.htm
m.blog.hmybk.cn/Article/details/1137937.htm
m.blog.hmybk.cn/Article/details/8404460.htm
m.blog.hmybk.cn/Article/details/9335379.htm
m.blog.hmybk.cn/Article/details/8224042.htm
m.blog.hmybk.cn/Article/details/0448260.htm
m.blog.hmybk.cn/Article/details/7573551.htm
m.blog.hmybk.cn/Article/details/7171375.htm
m.blog.hmybk.cn/Article/details/7597931.htm
m.blog.hmybk.cn/Article/details/7959193.htm
m.blog.hmybk.cn/Article/details/7193391.htm
m.blog.hmybk.cn/Article/details/2664662.htm
m.blog.hmybk.cn/Article/details/8024248.htm
m.blog.hmybk.cn/Article/details/4864220.htm
m.blog.hmybk.cn/Article/details/3973719.htm
m.blog.hmybk.cn/Article/details/6888866.htm
m.blog.hmybk.cn/Article/details/7919357.htm
m.blog.hmybk.cn/Article/details/1553713.htm
m.blog.hmybk.cn/Article/details/4868884.htm
m.blog.hmybk.cn/Article/details/9977153.htm
m.blog.hmybk.cn/Article/details/2640220.htm
m.blog.hmybk.cn/Article/details/9971795.htm
m.blog.hmybk.cn/Article/details/7179957.htm
m.blog.hmybk.cn/Article/details/8268088.htm
m.blog.hmybk.cn/Article/details/5579999.htm
m.blog.hmybk.cn/Article/details/1737557.htm
m.blog.hmybk.cn/Article/details/5353193.htm
m.blog.hmybk.cn/Article/details/5313937.htm
m.blog.hmybk.cn/Article/details/2220246.htm
m.blog.hmybk.cn/Article/details/4860444.htm
m.blog.hmybk.cn/Article/details/9719773.htm
m.blog.hmybk.cn/Article/details/0866688.htm
m.blog.hmybk.cn/Article/details/2820424.htm
m.blog.hmybk.cn/Article/details/6220600.htm
m.blog.hmybk.cn/Article/details/5539733.htm
m.blog.hmybk.cn/Article/details/3533977.htm
m.blog.hmybk.cn/Article/details/1319313.htm
m.blog.hmybk.cn/Article/details/9731331.htm
m.blog.hmybk.cn/Article/details/8660422.htm
m.blog.hmybk.cn/Article/details/1319113.htm
m.blog.hmybk.cn/Article/details/9355719.htm
m.blog.hmybk.cn/Article/details/6228666.htm
m.blog.hmybk.cn/Article/details/7995913.htm
m.blog.hmybk.cn/Article/details/8022822.htm
m.blog.hmybk.cn/Article/details/7199997.htm
m.blog.hmybk.cn/Article/details/7951395.htm
m.blog.hmybk.cn/Article/details/0644826.htm
m.blog.hmybk.cn/Article/details/5999151.htm
m.blog.hmybk.cn/Article/details/8622868.htm
m.blog.hmybk.cn/Article/details/9511191.htm
m.blog.hmybk.cn/Article/details/5773173.htm
m.blog.hmybk.cn/Article/details/8848200.htm
m.blog.hmybk.cn/Article/details/6444006.htm
m.blog.hmybk.cn/Article/details/6648888.htm
m.blog.hmybk.cn/Article/details/6828206.htm
m.blog.hmybk.cn/Article/details/6806868.htm
m.blog.hmybk.cn/Article/details/3951113.htm
m.blog.hmybk.cn/Article/details/5311573.htm
m.blog.hmybk.cn/Article/details/6604288.htm
m.blog.hmybk.cn/Article/details/6628480.htm
m.blog.hmybk.cn/Article/details/8022004.htm
m.blog.hmybk.cn/Article/details/7777993.htm
m.blog.hmybk.cn/Article/details/5191197.htm
m.blog.hmybk.cn/Article/details/7113195.htm
m.blog.hmybk.cn/Article/details/3337357.htm
m.blog.hmybk.cn/Article/details/7715139.htm
m.blog.hmybk.cn/Article/details/6260686.htm
m.blog.hmybk.cn/Article/details/6046822.htm
m.blog.hmybk.cn/Article/details/8886282.htm
m.blog.hmybk.cn/Article/details/3351319.htm
m.blog.hmybk.cn/Article/details/6264646.htm
m.blog.hmybk.cn/Article/details/4408468.htm
m.blog.hmybk.cn/Article/details/4428408.htm
m.blog.hmybk.cn/Article/details/2204608.htm
m.blog.hmybk.cn/Article/details/9171511.htm
m.blog.hmybk.cn/Article/details/4002448.htm
m.blog.hmybk.cn/Article/details/3593159.htm
m.blog.hmybk.cn/Article/details/7557379.htm
m.blog.hmybk.cn/Article/details/2206620.htm
m.blog.hmybk.cn/Article/details/3539557.htm
m.blog.hmybk.cn/Article/details/1755797.htm
m.blog.hmybk.cn/Article/details/1595115.htm
m.blog.hmybk.cn/Article/details/2624026.htm
m.blog.hmybk.cn/Article/details/1759113.htm
m.blog.hmybk.cn/Article/details/1199937.htm
m.blog.hmybk.cn/Article/details/9993973.htm
m.blog.hmybk.cn/Article/details/9975955.htm
m.blog.hmybk.cn/Article/details/9111593.htm
m.blog.hmybk.cn/Article/details/4266866.htm
m.blog.hmybk.cn/Article/details/5379117.htm
m.blog.hmybk.cn/Article/details/4806820.htm
m.blog.hmybk.cn/Article/details/3595373.htm
m.blog.hmybk.cn/Article/details/3951993.htm
m.blog.hmybk.cn/Article/details/0046826.htm
m.blog.hmybk.cn/Article/details/0826424.htm
m.blog.hmybk.cn/Article/details/6626826.htm
m.blog.hmybk.cn/Article/details/4600280.htm
m.blog.hmybk.cn/Article/details/7373731.htm
m.blog.hmybk.cn/Article/details/6406202.htm
m.blog.hmybk.cn/Article/details/7737971.htm
m.blog.hmybk.cn/Article/details/1559575.htm
m.blog.hmybk.cn/Article/details/1317733.htm
m.blog.hmybk.cn/Article/details/9911575.htm
m.blog.hmybk.cn/Article/details/2662808.htm
m.blog.hmybk.cn/Article/details/4000220.htm
m.blog.hmybk.cn/Article/details/3333939.htm
m.blog.hmybk.cn/Article/details/7173159.htm
m.blog.hmybk.cn/Article/details/8040064.htm
m.blog.hmybk.cn/Article/details/3737591.htm
m.blog.hmybk.cn/Article/details/9517993.htm
m.blog.hmybk.cn/Article/details/9391153.htm
m.blog.hmybk.cn/Article/details/9771733.htm
m.blog.hmybk.cn/Article/details/7173553.htm
m.blog.hmybk.cn/Article/details/5575115.htm
m.blog.hmybk.cn/Article/details/3955333.htm
m.blog.hmybk.cn/Article/details/2882420.htm
m.blog.hmybk.cn/Article/details/9911395.htm
m.blog.hmybk.cn/Article/details/7373915.htm
m.blog.hmybk.cn/Article/details/1191371.htm
m.blog.hmybk.cn/Article/details/7977751.htm
m.blog.hmybk.cn/Article/details/3113791.htm
m.blog.hmybk.cn/Article/details/4080484.htm
m.blog.hmybk.cn/Article/details/3795375.htm
m.blog.hmybk.cn/Article/details/2462402.htm
m.blog.hmybk.cn/Article/details/7933595.htm
m.blog.hmybk.cn/Article/details/3135939.htm
m.blog.hmybk.cn/Article/details/9377715.htm
m.blog.hmybk.cn/Article/details/7393931.htm
m.blog.hmybk.cn/Article/details/4424848.htm
m.blog.hmybk.cn/Article/details/7397791.htm
m.blog.hmybk.cn/Article/details/7315591.htm
m.blog.hmybk.cn/Article/details/9197533.htm
m.blog.hmybk.cn/Article/details/9553755.htm
m.blog.hmybk.cn/Article/details/3719175.htm
m.blog.hmybk.cn/Article/details/4282262.htm
m.blog.hmybk.cn/Article/details/3511735.htm
m.blog.hmybk.cn/Article/details/0024464.htm
m.blog.hmybk.cn/Article/details/1931737.htm
m.blog.hmybk.cn/Article/details/0080464.htm
m.blog.hmybk.cn/Article/details/5975971.htm
m.blog.hmybk.cn/Article/details/7319397.htm
m.blog.hmybk.cn/Article/details/9139599.htm
m.blog.hmybk.cn/Article/details/3117573.htm
m.blog.hmybk.cn/Article/details/8204882.htm
m.blog.hmybk.cn/Article/details/6642804.htm
m.blog.hmybk.cn/Article/details/8288808.htm
m.blog.hmybk.cn/Article/details/7559377.htm
m.blog.hmybk.cn/Article/details/0860062.htm
m.blog.hmybk.cn/Article/details/9117331.htm
m.blog.hmybk.cn/Article/details/2046608.htm
m.blog.hmybk.cn/Article/details/7399113.htm
m.blog.hmybk.cn/Article/details/8420268.htm
m.blog.hmybk.cn/Article/details/3379313.htm
m.blog.hmybk.cn/Article/details/8808408.htm
m.blog.hmybk.cn/Article/details/3193115.htm
m.blog.hmybk.cn/Article/details/8608040.htm
m.blog.hmybk.cn/Article/details/2224646.htm
m.blog.hmybk.cn/Article/details/7335973.htm
m.blog.hmybk.cn/Article/details/8626280.htm
