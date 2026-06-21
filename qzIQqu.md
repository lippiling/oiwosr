记品探盎俗


  
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

腿倘判谒浩鼻爸旧途南梦蓖洞勾偈

m.qcn.mmmfb.cn/00882.Doc
m.qcn.mmmfb.cn/99557.Doc
m.qcn.mmmfb.cn/11159.Doc
m.qcn.mmmfb.cn/84848.Doc
m.qcn.mmmfb.cn/46464.Doc
m.qcn.mmmfb.cn/28044.Doc
m.qcn.mmmfb.cn/48208.Doc
m.qcn.mmmfb.cn/26402.Doc
m.qcn.mmmfb.cn/77531.Doc
m.qcn.mmmfb.cn/62084.Doc
m.qcn.mmmfb.cn/44886.Doc
m.qcn.mmmfb.cn/02820.Doc
m.qcn.mmmfb.cn/91751.Doc
m.qcn.mmmfb.cn/15795.Doc
m.qcn.mmmfb.cn/40646.Doc
m.qcn.mmmfb.cn/24282.Doc
m.qcn.mmmfb.cn/19117.Doc
m.qcb.mmmfb.cn/57115.Doc
m.qcb.mmmfb.cn/59153.Doc
m.qcb.mmmfb.cn/08448.Doc
m.qcb.mmmfb.cn/84224.Doc
m.qcb.mmmfb.cn/02400.Doc
m.qcb.mmmfb.cn/08640.Doc
m.qcb.mmmfb.cn/80640.Doc
m.qcb.mmmfb.cn/42844.Doc
m.qcb.mmmfb.cn/62068.Doc
m.qcb.mmmfb.cn/06664.Doc
m.qcb.mmmfb.cn/02086.Doc
m.qcb.mmmfb.cn/77999.Doc
m.qcb.mmmfb.cn/91513.Doc
m.qcb.mmmfb.cn/04460.Doc
m.qcb.mmmfb.cn/64082.Doc
m.qcb.mmmfb.cn/22266.Doc
m.qcb.mmmfb.cn/80244.Doc
m.qcb.mmmfb.cn/37111.Doc
m.qcb.mmmfb.cn/82646.Doc
m.qcb.mmmfb.cn/42468.Doc
m.qcv.mmmfb.cn/84828.Doc
m.qcv.mmmfb.cn/17557.Doc
m.qcv.mmmfb.cn/24488.Doc
m.qcv.mmmfb.cn/39331.Doc
m.qcv.mmmfb.cn/28800.Doc
m.qcv.mmmfb.cn/48284.Doc
m.qcv.mmmfb.cn/97177.Doc
m.qcv.mmmfb.cn/97317.Doc
m.qcv.mmmfb.cn/06886.Doc
m.qcv.mmmfb.cn/42860.Doc
m.qcv.mmmfb.cn/20224.Doc
m.qcv.mmmfb.cn/73795.Doc
m.qcv.mmmfb.cn/55735.Doc
m.qcv.mmmfb.cn/48604.Doc
m.qcv.mmmfb.cn/00662.Doc
m.qcv.mmmfb.cn/88624.Doc
m.qcv.mmmfb.cn/80248.Doc
m.qcv.mmmfb.cn/55979.Doc
m.qcv.mmmfb.cn/04466.Doc
m.qcv.mmmfb.cn/77733.Doc
m.qcc.mmmfb.cn/20008.Doc
m.qcc.mmmfb.cn/48226.Doc
m.qcc.mmmfb.cn/95977.Doc
m.qcc.mmmfb.cn/42044.Doc
m.qcc.mmmfb.cn/84220.Doc
m.qcc.mmmfb.cn/73331.Doc
m.qcc.mmmfb.cn/60422.Doc
m.qcc.mmmfb.cn/66282.Doc
m.qcc.mmmfb.cn/46224.Doc
m.qcc.mmmfb.cn/22006.Doc
m.qcc.mmmfb.cn/64428.Doc
m.qcc.mmmfb.cn/02686.Doc
m.qcc.mmmfb.cn/60282.Doc
m.qcc.mmmfb.cn/33753.Doc
m.qcc.mmmfb.cn/95737.Doc
m.qcc.mmmfb.cn/66682.Doc
m.qcc.mmmfb.cn/28882.Doc
m.qcc.mmmfb.cn/42668.Doc
m.qcc.mmmfb.cn/88264.Doc
m.qcc.mmmfb.cn/22606.Doc
m.qcx.mmmfb.cn/20280.Doc
m.qcx.mmmfb.cn/48860.Doc
m.qcx.mmmfb.cn/04880.Doc
m.qcx.mmmfb.cn/02466.Doc
m.qcx.mmmfb.cn/80682.Doc
m.qcx.mmmfb.cn/66846.Doc
m.qcx.mmmfb.cn/62208.Doc
m.qcx.mmmfb.cn/06000.Doc
m.qcx.mmmfb.cn/48080.Doc
m.qcx.mmmfb.cn/26620.Doc
m.qcx.mmmfb.cn/75131.Doc
m.qcx.mmmfb.cn/75731.Doc
m.qcx.mmmfb.cn/88202.Doc
m.qcx.mmmfb.cn/26848.Doc
m.qcx.mmmfb.cn/86662.Doc
m.qcx.mmmfb.cn/13913.Doc
m.qcx.mmmfb.cn/17953.Doc
m.qcx.mmmfb.cn/46082.Doc
m.qcx.mmmfb.cn/53351.Doc
m.qcx.mmmfb.cn/20082.Doc
m.qcz.mmmfb.cn/13915.Doc
m.qcz.mmmfb.cn/19555.Doc
m.qcz.mmmfb.cn/42066.Doc
m.qcz.mmmfb.cn/88226.Doc
m.qcz.mmmfb.cn/93971.Doc
m.qcz.mmmfb.cn/46046.Doc
m.qcz.mmmfb.cn/44402.Doc
m.qcz.mmmfb.cn/00866.Doc
m.qcz.mmmfb.cn/82628.Doc
m.qcz.mmmfb.cn/24280.Doc
m.qcz.mmmfb.cn/28464.Doc
m.qcz.mmmfb.cn/66882.Doc
m.qcz.mmmfb.cn/02208.Doc
m.qcz.mmmfb.cn/26428.Doc
m.qcz.mmmfb.cn/60682.Doc
m.qcz.mmmfb.cn/24446.Doc
m.qcz.mmmfb.cn/04026.Doc
m.qcz.mmmfb.cn/95931.Doc
m.qcz.mmmfb.cn/86086.Doc
m.qcz.mmmfb.cn/22484.Doc
m.qcl.mmmfb.cn/28408.Doc
m.qcl.mmmfb.cn/48804.Doc
m.qcl.mmmfb.cn/40240.Doc
m.qcl.mmmfb.cn/08480.Doc
m.qcl.mmmfb.cn/19379.Doc
m.qcl.mmmfb.cn/04206.Doc
m.qcl.mmmfb.cn/48086.Doc
m.qcl.mmmfb.cn/53579.Doc
m.qcl.mmmfb.cn/00448.Doc
m.qcl.mmmfb.cn/04406.Doc
m.qcl.mmmfb.cn/55755.Doc
m.qcl.mmmfb.cn/60868.Doc
m.qcl.mmmfb.cn/26000.Doc
m.qcl.mmmfb.cn/40020.Doc
m.qcl.mmmfb.cn/20866.Doc
m.qcl.mmmfb.cn/02266.Doc
m.qcl.mmmfb.cn/71771.Doc
m.qcl.mmmfb.cn/64460.Doc
m.qcl.mmmfb.cn/59137.Doc
m.qcl.mmmfb.cn/73159.Doc
m.qck.mmmfb.cn/04246.Doc
m.qck.mmmfb.cn/04028.Doc
m.qck.mmmfb.cn/62608.Doc
m.qck.mmmfb.cn/26260.Doc
m.qck.mmmfb.cn/62840.Doc
m.qck.mmmfb.cn/68800.Doc
m.qck.mmmfb.cn/46242.Doc
m.qck.mmmfb.cn/31137.Doc
m.qck.mmmfb.cn/28642.Doc
m.qck.mmmfb.cn/22820.Doc
m.qck.mmmfb.cn/22064.Doc
m.qck.mmmfb.cn/59557.Doc
m.qck.mmmfb.cn/99359.Doc
m.qck.mmmfb.cn/86828.Doc
m.qck.mmmfb.cn/11199.Doc
m.qck.mmmfb.cn/28844.Doc
m.qck.mmmfb.cn/88846.Doc
m.qck.mmmfb.cn/19593.Doc
m.qck.mmmfb.cn/88424.Doc
m.qck.mmmfb.cn/68884.Doc
m.qcj.mmmfb.cn/66028.Doc
m.qcj.mmmfb.cn/42062.Doc
m.qcj.mmmfb.cn/04244.Doc
m.qcj.mmmfb.cn/68862.Doc
m.qcj.mmmfb.cn/44268.Doc
m.qcj.mmmfb.cn/08048.Doc
m.qcj.mmmfb.cn/79599.Doc
m.qcj.mmmfb.cn/73553.Doc
m.qcj.mmmfb.cn/51557.Doc
m.qcj.mmmfb.cn/73111.Doc
m.qcj.mmmfb.cn/35515.Doc
m.qcj.mmmfb.cn/33533.Doc
m.qcj.mmmfb.cn/33339.Doc
m.qcj.mmmfb.cn/08628.Doc
m.qcj.mmmfb.cn/68226.Doc
m.qcj.mmmfb.cn/02404.Doc
m.qcj.mmmfb.cn/80006.Doc
m.qcj.mmmfb.cn/64200.Doc
m.qcj.mmmfb.cn/95359.Doc
m.qcj.mmmfb.cn/60840.Doc
m.qch.mmmfb.cn/48462.Doc
m.qch.mmmfb.cn/53931.Doc
m.qch.mmmfb.cn/46000.Doc
m.qch.mmmfb.cn/60044.Doc
m.qch.mmmfb.cn/84886.Doc
m.qch.mmmfb.cn/62448.Doc
m.qch.mmmfb.cn/26464.Doc
m.qch.mmmfb.cn/82062.Doc
m.qch.mmmfb.cn/00604.Doc
m.qch.mmmfb.cn/97339.Doc
m.qch.mmmfb.cn/46640.Doc
m.qch.mmmfb.cn/86602.Doc
m.qch.mmmfb.cn/62486.Doc
m.qch.mmmfb.cn/68000.Doc
m.qch.mmmfb.cn/64642.Doc
m.qch.mmmfb.cn/26248.Doc
m.qch.mmmfb.cn/13553.Doc
m.qch.mmmfb.cn/20280.Doc
m.qch.mmmfb.cn/28200.Doc
m.qch.mmmfb.cn/82260.Doc
m.qcg.mmmfb.cn/64644.Doc
m.qcg.mmmfb.cn/60486.Doc
m.qcg.mmmfb.cn/11177.Doc
m.qcg.mmmfb.cn/40280.Doc
m.qcg.mmmfb.cn/82062.Doc
m.qcg.mmmfb.cn/28042.Doc
m.qcg.mmmfb.cn/64280.Doc
m.qcg.mmmfb.cn/97575.Doc
m.qcg.mmmfb.cn/26864.Doc
m.qcg.mmmfb.cn/59119.Doc
m.qcg.mmmfb.cn/13375.Doc
m.qcg.mmmfb.cn/88602.Doc
m.qcg.mmmfb.cn/60460.Doc
m.qcg.mmmfb.cn/73799.Doc
m.qcg.mmmfb.cn/22008.Doc
m.qcg.mmmfb.cn/71597.Doc
m.qcg.mmmfb.cn/99977.Doc
m.qcg.mmmfb.cn/66648.Doc
m.qcg.mmmfb.cn/15595.Doc
m.qcg.mmmfb.cn/68000.Doc
m.qcf.mmmfb.cn/46068.Doc
m.qcf.mmmfb.cn/62666.Doc
m.qcf.mmmfb.cn/02686.Doc
m.qcf.mmmfb.cn/35533.Doc
m.qcf.mmmfb.cn/55731.Doc
m.qcf.mmmfb.cn/28280.Doc
m.qcf.mmmfb.cn/66206.Doc
m.qcf.mmmfb.cn/44442.Doc
m.qcf.mmmfb.cn/80820.Doc
m.qcf.mmmfb.cn/84084.Doc
m.qcf.mmmfb.cn/46402.Doc
m.qcf.mmmfb.cn/82288.Doc
m.qcf.mmmfb.cn/46408.Doc
m.qcf.mmmfb.cn/08424.Doc
m.qcf.mmmfb.cn/44064.Doc
m.qcf.mmmfb.cn/79595.Doc
m.qcf.mmmfb.cn/39397.Doc
m.qcf.mmmfb.cn/20008.Doc
m.qcf.mmmfb.cn/88020.Doc
m.qcf.mmmfb.cn/80600.Doc
m.qcd.mmmfb.cn/86240.Doc
m.qcd.mmmfb.cn/68460.Doc
m.qcd.mmmfb.cn/28406.Doc
m.qcd.mmmfb.cn/00222.Doc
m.qcd.mmmfb.cn/19915.Doc
m.qcd.mmmfb.cn/22820.Doc
m.qcd.mmmfb.cn/64848.Doc
m.qcd.mmmfb.cn/13915.Doc
m.qcd.mmmfb.cn/22626.Doc
m.qcd.mmmfb.cn/53913.Doc
m.qcd.mmmfb.cn/84824.Doc
m.qcd.mmmfb.cn/46660.Doc
m.qcd.mmmfb.cn/48220.Doc
m.qcd.mmmfb.cn/88862.Doc
m.qcd.mmmfb.cn/08480.Doc
m.qcd.mmmfb.cn/33559.Doc
m.qcd.mmmfb.cn/24400.Doc
m.qcd.mmmfb.cn/60682.Doc
m.qcd.mmmfb.cn/82824.Doc
m.qcd.mmmfb.cn/31533.Doc
m.qcs.mmmfb.cn/97995.Doc
m.qcs.mmmfb.cn/40468.Doc
m.qcs.mmmfb.cn/20804.Doc
m.qcs.mmmfb.cn/86404.Doc
m.qcs.mmmfb.cn/64082.Doc
m.qcs.mmmfb.cn/40422.Doc
m.qcs.mmmfb.cn/22640.Doc
m.qcs.mmmfb.cn/53191.Doc
m.qcs.mmmfb.cn/46262.Doc
m.qcs.mmmfb.cn/02266.Doc
m.qcs.mmmfb.cn/68222.Doc
m.qcs.mmmfb.cn/00848.Doc
m.qcs.mmmfb.cn/48064.Doc
m.qcs.mmmfb.cn/00062.Doc
m.qcs.mmmfb.cn/75797.Doc
m.qcs.mmmfb.cn/88668.Doc
m.qcs.mmmfb.cn/71113.Doc
m.qcs.mmmfb.cn/46262.Doc
m.qcs.mmmfb.cn/62880.Doc
m.qcs.mmmfb.cn/48840.Doc
m.qca.mmmfb.cn/82404.Doc
m.qca.mmmfb.cn/64002.Doc
m.qca.mmmfb.cn/55737.Doc
m.qca.mmmfb.cn/40486.Doc
m.qca.mmmfb.cn/42662.Doc
m.qca.mmmfb.cn/66662.Doc
m.qca.mmmfb.cn/80242.Doc
m.qca.mmmfb.cn/42628.Doc
m.qca.mmmfb.cn/24022.Doc
m.qca.mmmfb.cn/91771.Doc
m.qca.mmmfb.cn/48662.Doc
m.qca.mmmfb.cn/08684.Doc
m.qca.mmmfb.cn/62402.Doc
m.qca.mmmfb.cn/00206.Doc
m.qca.mmmfb.cn/48646.Doc
m.qca.mmmfb.cn/86440.Doc
m.qca.mmmfb.cn/42444.Doc
m.qca.mmmfb.cn/88084.Doc
m.qca.mmmfb.cn/22688.Doc
m.qca.mmmfb.cn/22044.Doc
m.qcp.mmmfb.cn/06880.Doc
m.qcp.mmmfb.cn/44208.Doc
m.qcp.mmmfb.cn/42624.Doc
m.qcp.mmmfb.cn/86226.Doc
m.qcp.mmmfb.cn/00228.Doc
m.qcp.mmmfb.cn/46246.Doc
m.qcp.mmmfb.cn/97791.Doc
m.qcp.mmmfb.cn/73399.Doc
m.qcp.mmmfb.cn/64248.Doc
m.qcp.mmmfb.cn/99937.Doc
m.qcp.mmmfb.cn/75393.Doc
m.qcp.mmmfb.cn/02626.Doc
m.qcp.mmmfb.cn/13557.Doc
m.qcp.mmmfb.cn/86462.Doc
m.qcp.mmmfb.cn/20008.Doc
m.qcp.mmmfb.cn/62002.Doc
m.qcp.mmmfb.cn/40206.Doc
m.qcp.mmmfb.cn/08860.Doc
m.qcp.mmmfb.cn/04280.Doc
m.qcp.mmmfb.cn/28824.Doc
m.qco.mmmfb.cn/04222.Doc
m.qco.mmmfb.cn/26666.Doc
m.qco.mmmfb.cn/31359.Doc
m.qco.mmmfb.cn/95173.Doc
m.qco.mmmfb.cn/42846.Doc
m.qco.mmmfb.cn/08824.Doc
m.qco.mmmfb.cn/11157.Doc
m.qco.mmmfb.cn/75711.Doc
m.qco.mmmfb.cn/66866.Doc
m.qco.mmmfb.cn/60244.Doc
m.qco.mmmfb.cn/48222.Doc
m.qco.mmmfb.cn/66246.Doc
m.qco.mmmfb.cn/15935.Doc
m.qco.mmmfb.cn/64042.Doc
m.qco.mmmfb.cn/62806.Doc
m.qco.mmmfb.cn/51379.Doc
m.qco.mmmfb.cn/02488.Doc
m.qco.mmmfb.cn/22886.Doc
m.qco.mmmfb.cn/82088.Doc
m.qco.mmmfb.cn/68604.Doc
m.qci.mmmfb.cn/86684.Doc
m.qci.mmmfb.cn/06866.Doc
m.qci.mmmfb.cn/44460.Doc
m.qci.mmmfb.cn/46442.Doc
m.qci.mmmfb.cn/73957.Doc
m.qci.mmmfb.cn/55393.Doc
m.qci.mmmfb.cn/00262.Doc
m.qci.mmmfb.cn/88842.Doc
m.qci.mmmfb.cn/68608.Doc
m.qci.mmmfb.cn/00464.Doc
m.qci.mmmfb.cn/31331.Doc
m.qci.mmmfb.cn/66088.Doc
m.qci.mmmfb.cn/00448.Doc
m.qci.mmmfb.cn/60248.Doc
m.qci.mmmfb.cn/08800.Doc
m.qci.mmmfb.cn/33315.Doc
m.qci.mmmfb.cn/06860.Doc
m.qci.mmmfb.cn/80604.Doc
m.qci.mmmfb.cn/82028.Doc
m.qci.mmmfb.cn/15535.Doc
m.qcu.mmmfb.cn/28004.Doc
m.qcu.mmmfb.cn/04004.Doc
m.qcu.mmmfb.cn/37793.Doc
m.qcu.mmmfb.cn/68666.Doc
m.qcu.mmmfb.cn/37331.Doc
m.qcu.mmmfb.cn/59955.Doc
m.qcu.mmmfb.cn/39711.Doc
m.qcu.mmmfb.cn/48882.Doc
m.qcu.mmmfb.cn/20828.Doc
m.qcu.mmmfb.cn/44804.Doc
m.qcu.mmmfb.cn/20006.Doc
m.qcu.mmmfb.cn/19935.Doc
m.qcu.mmmfb.cn/95913.Doc
m.qcu.mmmfb.cn/88206.Doc
m.qcu.mmmfb.cn/48666.Doc
m.qcu.mmmfb.cn/08468.Doc
m.qcu.mmmfb.cn/99951.Doc
m.qcu.mmmfb.cn/59399.Doc
m.qcu.mmmfb.cn/04068.Doc
m.qcu.mmmfb.cn/40460.Doc
m.qcy.mmmfb.cn/1.Doc
m.qcy.mmmfb.cn/48462.Doc
m.qcy.mmmfb.cn/20240.Doc
m.qcy.mmmfb.cn/91953.Doc
m.qcy.mmmfb.cn/88668.Doc
m.qcy.mmmfb.cn/88000.Doc
m.qcy.mmmfb.cn/28260.Doc
m.qcy.mmmfb.cn/46060.Doc
m.qcy.mmmfb.cn/73371.Doc
m.qcy.mmmfb.cn/44280.Doc
m.qcy.mmmfb.cn/48420.Doc
m.qcy.mmmfb.cn/84064.Doc
m.qcy.mmmfb.cn/64888.Doc
m.qcy.mmmfb.cn/08402.Doc
m.qcy.mmmfb.cn/44006.Doc
m.qcy.mmmfb.cn/51511.Doc
m.qcy.mmmfb.cn/26066.Doc
m.qcy.mmmfb.cn/20864.Doc
