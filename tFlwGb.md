缺置谠甘疾


  
Iterator 它是用于安全遍历集合的临时一次性访问代理；它剥离了遍历逻辑和集合实现，支持统一 hasNext()/next() 并允许接口通过 remove() 集合安全修改。

Iterator “只读游标”是用来收集安全遍历的
它不是一个容器，也不是数据本身，而是一个临时的、一次性的访问代理**。您可以调用它 list.iterator() 得到的不是收集副本，而是指向当前收集状态的轻量级指针——它知道“下一个应该取谁”，但不负责存储或管理数据。
关键是，它将“如何取元素”从集合实现中剥离出来。不管你面对什么， ArrayList（数组）、LinkedList(链表)还是 HashSet(哈希表)都用同一套。 hasNext()/next() 逻辑，不要写标记，不要担心索引越界，也不要管底层是顺序还是散列。


为什么不能直接？ for 循环删除元素，但可以使用 Iterator.remove()？
因为普通 for 或增强 for 在全历时，集合内部修改计数器（modCount）如果与迭代器预期的版本不一致，会立即抛出 ConcurrentModificationException；而 iterator.remove() 这是唯一允许的“边走边改”操作——它同步更新计数器，使迭代器和集合保持步调一致。

✅ 正确：先 next()，再 remove()(而且只能调一次)
❌ 错误：remove() 前没调 next() → IllegalStateException

❌ 错误：连续两次 remove() → 同样抛 IllegalStateException

❌ 错误：用 list.remove(obj) 替代 iterator.remove() → 必爆并修改异常

ListIterator 和普通 Iterator 实际区别是什么？
ListIterator 是 Iterator 子接口，仅适用于 List 类型，核心价值在「双向 + 可写」：
；
			
		

支持 previous() 和 hasPrevious()，能倒着走
支持 add()：在当前位置插入新元素(不影响当前指针)
支持 set()：替换上一次 next() 或 previous() 返回的元素
不支持 Map 或 Set，强制转型会报 ClassCastException


例如，你需要在每两个元素之间插入一个分隔符，或者从后面找到并删除最后一个匹配项—— ListIterator 它是不可替代的。

迭代器失效，空指针，NoSuchElementException 怎么避坑？
迭代器本身不保存，一旦集合被外部修改（如另一个线程更改，或以非迭代器方式删除），它将立即失效； next() 调用前不检查 hasNext()，就会崩出 NoSuchElementException。

每次 next() 前必须配对 hasNext()，别偷懒写成 while(true) { it.next(); }

迭代器不能重复使用：使用后丢失，重复历必须再次调整 collection.iterator()

即使在多线程场景中使用 Iterator，也不能解决并发修改的问题——必须得到 CopyOnWriteArrayList 或加锁

map.keySet().iterator() 和 map.values().iterator() 返回的迭代器不相互影响，但都依赖于原始迭代器 HashMap 结构稳定性

最常被忽视的一点:remove() 是可选操作。一些集合(如 Arrays.asList() 包装列表)返回的迭代器不支持删除，并在调用时抛出 UnsupportedOperationException——先看文档，或者用 try-catch 容错。	 

酵先杏琳佬瓮杂侗誓彰反按家盗交

tv.blog.cvvliy.cn/Article/details/004466.sHtML
tv.blog.cvvliy.cn/Article/details/004484.sHtML
tv.blog.cvvliy.cn/Article/details/351337.sHtML
tv.blog.cvvliy.cn/Article/details/028882.sHtML
tv.blog.cvvliy.cn/Article/details/660462.sHtML
tv.blog.cvvliy.cn/Article/details/620680.sHtML
tv.blog.cvvliy.cn/Article/details/880288.sHtML
tv.blog.cvvliy.cn/Article/details/462086.sHtML
tv.blog.cvvliy.cn/Article/details/600022.sHtML
tv.blog.cvvliy.cn/Article/details/717999.sHtML
tv.blog.cvvliy.cn/Article/details/379779.sHtML
tv.blog.cvvliy.cn/Article/details/802226.sHtML
tv.blog.cvvliy.cn/Article/details/799377.sHtML
tv.blog.cvvliy.cn/Article/details/119177.sHtML
tv.blog.cvvliy.cn/Article/details/266400.sHtML
tv.blog.cvvliy.cn/Article/details/264282.sHtML
tv.blog.cvvliy.cn/Article/details/624440.sHtML
tv.blog.cvvliy.cn/Article/details/204004.sHtML
tv.blog.cvvliy.cn/Article/details/208606.sHtML
tv.blog.cvvliy.cn/Article/details/371755.sHtML
tv.blog.cvvliy.cn/Article/details/242008.sHtML
tv.blog.cvvliy.cn/Article/details/335959.sHtML
tv.blog.cvvliy.cn/Article/details/717575.sHtML
tv.blog.cvvliy.cn/Article/details/684606.sHtML
tv.blog.cvvliy.cn/Article/details/717513.sHtML
tv.blog.cvvliy.cn/Article/details/513351.sHtML
tv.blog.cvvliy.cn/Article/details/355519.sHtML
tv.blog.cvvliy.cn/Article/details/739915.sHtML
tv.blog.cvvliy.cn/Article/details/202280.sHtML
tv.blog.cvvliy.cn/Article/details/173375.sHtML
tv.blog.cvvliy.cn/Article/details/066602.sHtML
tv.blog.cvvliy.cn/Article/details/171155.sHtML
tv.blog.cvvliy.cn/Article/details/462026.sHtML
tv.blog.cvvliy.cn/Article/details/482862.sHtML
tv.blog.cvvliy.cn/Article/details/888222.sHtML
tv.blog.cvvliy.cn/Article/details/535971.sHtML
tv.blog.cvvliy.cn/Article/details/200804.sHtML
tv.blog.cvvliy.cn/Article/details/848860.sHtML
tv.blog.cvvliy.cn/Article/details/486804.sHtML
tv.blog.cvvliy.cn/Article/details/286046.sHtML
tv.blog.cvvliy.cn/Article/details/086048.sHtML
tv.blog.cvvliy.cn/Article/details/248220.sHtML
tv.blog.cvvliy.cn/Article/details/424244.sHtML
tv.blog.cvvliy.cn/Article/details/468680.sHtML
tv.blog.cvvliy.cn/Article/details/440686.sHtML
tv.blog.cvvliy.cn/Article/details/917571.sHtML
tv.blog.cvvliy.cn/Article/details/377119.sHtML
tv.blog.cvvliy.cn/Article/details/739113.sHtML
tv.blog.cvvliy.cn/Article/details/513593.sHtML
tv.blog.cvvliy.cn/Article/details/711591.sHtML
tv.blog.cvvliy.cn/Article/details/111713.sHtML
tv.blog.cvvliy.cn/Article/details/606468.sHtML
tv.blog.cvvliy.cn/Article/details/577313.sHtML
tv.blog.cvvliy.cn/Article/details/026006.sHtML
tv.blog.cvvliy.cn/Article/details/060424.sHtML
tv.blog.cvvliy.cn/Article/details/684820.sHtML
tv.blog.cvvliy.cn/Article/details/339391.sHtML
tv.blog.cvvliy.cn/Article/details/646820.sHtML
tv.blog.cvvliy.cn/Article/details/171313.sHtML
tv.blog.cvvliy.cn/Article/details/048664.sHtML
tv.blog.cvvliy.cn/Article/details/559377.sHtML
tv.blog.cvvliy.cn/Article/details/264084.sHtML
tv.blog.cvvliy.cn/Article/details/026220.sHtML
tv.blog.cvvliy.cn/Article/details/204880.sHtML
tv.blog.cvvliy.cn/Article/details/113711.sHtML
tv.blog.cvvliy.cn/Article/details/448646.sHtML
tv.blog.cvvliy.cn/Article/details/826200.sHtML
tv.blog.cvvliy.cn/Article/details/155195.sHtML
tv.blog.cvvliy.cn/Article/details/711791.sHtML
tv.blog.cvvliy.cn/Article/details/779157.sHtML
tv.blog.cvvliy.cn/Article/details/246028.sHtML
tv.blog.cvvliy.cn/Article/details/199793.sHtML
tv.blog.cvvliy.cn/Article/details/795113.sHtML
tv.blog.cvvliy.cn/Article/details/006844.sHtML
tv.blog.cvvliy.cn/Article/details/371959.sHtML
tv.blog.cvvliy.cn/Article/details/199959.sHtML
tv.blog.cvvliy.cn/Article/details/266640.sHtML
tv.blog.cvvliy.cn/Article/details/022648.sHtML
tv.blog.cvvliy.cn/Article/details/642262.sHtML
tv.blog.cvvliy.cn/Article/details/042084.sHtML
tv.blog.cvvliy.cn/Article/details/688644.sHtML
tv.blog.cvvliy.cn/Article/details/157179.sHtML
tv.blog.cvvliy.cn/Article/details/220008.sHtML
tv.blog.cvvliy.cn/Article/details/466268.sHtML
tv.blog.cvvliy.cn/Article/details/577959.sHtML
tv.blog.cvvliy.cn/Article/details/937119.sHtML
tv.blog.cvvliy.cn/Article/details/424044.sHtML
tv.blog.cvvliy.cn/Article/details/008088.sHtML
tv.blog.cvvliy.cn/Article/details/460282.sHtML
tv.blog.cvvliy.cn/Article/details/666482.sHtML
tv.blog.cvvliy.cn/Article/details/593955.sHtML
tv.blog.cvvliy.cn/Article/details/773573.sHtML
tv.blog.cvvliy.cn/Article/details/448444.sHtML
tv.blog.cvvliy.cn/Article/details/642220.sHtML
tv.blog.cvvliy.cn/Article/details/519371.sHtML
tv.blog.cvvliy.cn/Article/details/280444.sHtML
tv.blog.cvvliy.cn/Article/details/448606.sHtML
tv.blog.cvvliy.cn/Article/details/135931.sHtML
tv.blog.cvvliy.cn/Article/details/448686.sHtML
tv.blog.cvvliy.cn/Article/details/228004.sHtML
tv.blog.cvvliy.cn/Article/details/048864.sHtML
tv.blog.cvvliy.cn/Article/details/715571.sHtML
tv.blog.cvvliy.cn/Article/details/557777.sHtML
tv.blog.cvvliy.cn/Article/details/460426.sHtML
tv.blog.cvvliy.cn/Article/details/799791.sHtML
tv.blog.cvvliy.cn/Article/details/717933.sHtML
tv.blog.cvvliy.cn/Article/details/317757.sHtML
tv.blog.cvvliy.cn/Article/details/395517.sHtML
tv.blog.cvvliy.cn/Article/details/777191.sHtML
tv.blog.cvvliy.cn/Article/details/795155.sHtML
tv.blog.cvvliy.cn/Article/details/111317.sHtML
tv.blog.cvvliy.cn/Article/details/737119.sHtML
tv.blog.cvvliy.cn/Article/details/482260.sHtML
tv.blog.cvvliy.cn/Article/details/286264.sHtML
tv.blog.cvvliy.cn/Article/details/486480.sHtML
tv.blog.cvvliy.cn/Article/details/644862.sHtML
tv.blog.cvvliy.cn/Article/details/757919.sHtML
tv.blog.cvvliy.cn/Article/details/624226.sHtML
tv.blog.cvvliy.cn/Article/details/575779.sHtML
tv.blog.cvvliy.cn/Article/details/135553.sHtML
tv.blog.cvvliy.cn/Article/details/202026.sHtML
tv.blog.cvvliy.cn/Article/details/846640.sHtML
tv.blog.cvvliy.cn/Article/details/595359.sHtML
tv.blog.cvvliy.cn/Article/details/426460.sHtML
tv.blog.cvvliy.cn/Article/details/828846.sHtML
tv.blog.cvvliy.cn/Article/details/555537.sHtML
tv.blog.cvvliy.cn/Article/details/442444.sHtML
tv.blog.cvvliy.cn/Article/details/911959.sHtML
tv.blog.cvvliy.cn/Article/details/040424.sHtML
tv.blog.cvvliy.cn/Article/details/151579.sHtML
tv.blog.cvvliy.cn/Article/details/802464.sHtML
tv.blog.cvvliy.cn/Article/details/462682.sHtML
tv.blog.cvvliy.cn/Article/details/802020.sHtML
tv.blog.cvvliy.cn/Article/details/440244.sHtML
tv.blog.cvvliy.cn/Article/details/731191.sHtML
tv.blog.cvvliy.cn/Article/details/468046.sHtML
tv.blog.cvvliy.cn/Article/details/006260.sHtML
tv.blog.cvvliy.cn/Article/details/735171.sHtML
tv.blog.cvvliy.cn/Article/details/042808.sHtML
tv.blog.cvvliy.cn/Article/details/713311.sHtML
tv.blog.cvvliy.cn/Article/details/462460.sHtML
tv.blog.cvvliy.cn/Article/details/159915.sHtML
tv.blog.cvvliy.cn/Article/details/826840.sHtML
tv.blog.cvvliy.cn/Article/details/048400.sHtML
tv.blog.cvvliy.cn/Article/details/804264.sHtML
tv.blog.cvvliy.cn/Article/details/608826.sHtML
tv.blog.cvvliy.cn/Article/details/882424.sHtML
tv.blog.cvvliy.cn/Article/details/400084.sHtML
tv.blog.cvvliy.cn/Article/details/646200.sHtML
tv.blog.cvvliy.cn/Article/details/026246.sHtML
tv.blog.cvvliy.cn/Article/details/000020.sHtML
tv.blog.cvvliy.cn/Article/details/179917.sHtML
tv.blog.cvvliy.cn/Article/details/242226.sHtML
tv.blog.cvvliy.cn/Article/details/137355.sHtML
tv.blog.cvvliy.cn/Article/details/979197.sHtML
tv.blog.cvvliy.cn/Article/details/848046.sHtML
tv.blog.cvvliy.cn/Article/details/159717.sHtML
tv.blog.cvvliy.cn/Article/details/157393.sHtML
tv.blog.cvvliy.cn/Article/details/395195.sHtML
tv.blog.cvvliy.cn/Article/details/175777.sHtML
tv.blog.cvvliy.cn/Article/details/002480.sHtML
tv.blog.cvvliy.cn/Article/details/608006.sHtML
tv.blog.cvvliy.cn/Article/details/793951.sHtML
tv.blog.cvvliy.cn/Article/details/539399.sHtML
tv.blog.cvvliy.cn/Article/details/406264.sHtML
tv.blog.cvvliy.cn/Article/details/228844.sHtML
tv.blog.cvvliy.cn/Article/details/084668.sHtML
tv.blog.cvvliy.cn/Article/details/951717.sHtML
tv.blog.cvvliy.cn/Article/details/131195.sHtML
tv.blog.cvvliy.cn/Article/details/997755.sHtML
tv.blog.cvvliy.cn/Article/details/991573.sHtML
tv.blog.cvvliy.cn/Article/details/622402.sHtML
tv.blog.cvvliy.cn/Article/details/288680.sHtML
tv.blog.cvvliy.cn/Article/details/648864.sHtML
tv.blog.cvvliy.cn/Article/details/153199.sHtML
tv.blog.cvvliy.cn/Article/details/428420.sHtML
tv.blog.cvvliy.cn/Article/details/600866.sHtML
tv.blog.cvvliy.cn/Article/details/406220.sHtML
tv.blog.cvvliy.cn/Article/details/953795.sHtML
tv.blog.cvvliy.cn/Article/details/575713.sHtML
tv.blog.cvvliy.cn/Article/details/008284.sHtML
tv.blog.cvvliy.cn/Article/details/222082.sHtML
tv.blog.cvvliy.cn/Article/details/377735.sHtML
tv.blog.cvvliy.cn/Article/details/064622.sHtML
tv.blog.cvvliy.cn/Article/details/513339.sHtML
tv.blog.cvvliy.cn/Article/details/684468.sHtML
tv.blog.cvvliy.cn/Article/details/446660.sHtML
tv.blog.cvvliy.cn/Article/details/713975.sHtML
tv.blog.cvvliy.cn/Article/details/551513.sHtML
tv.blog.cvvliy.cn/Article/details/088004.sHtML
tv.blog.cvvliy.cn/Article/details/571773.sHtML
tv.blog.cvvliy.cn/Article/details/402666.sHtML
tv.blog.cvvliy.cn/Article/details/064220.sHtML
tv.blog.cvvliy.cn/Article/details/606602.sHtML
tv.blog.cvvliy.cn/Article/details/080420.sHtML
tv.blog.cvvliy.cn/Article/details/953995.sHtML
tv.blog.cvvliy.cn/Article/details/402468.sHtML
tv.blog.cvvliy.cn/Article/details/555137.sHtML
tv.blog.cvvliy.cn/Article/details/248448.sHtML
tv.blog.cvvliy.cn/Article/details/977717.sHtML
tv.blog.cvvliy.cn/Article/details/333519.sHtML
tv.blog.cvvliy.cn/Article/details/062484.sHtML
tv.blog.cvvliy.cn/Article/details/844626.sHtML
tv.blog.cvvliy.cn/Article/details/808628.sHtML
tv.blog.cvvliy.cn/Article/details/020628.sHtML
tv.blog.cvvliy.cn/Article/details/511115.sHtML
tv.blog.cvvliy.cn/Article/details/117177.sHtML
tv.blog.cvvliy.cn/Article/details/008480.sHtML
tv.blog.cvvliy.cn/Article/details/060622.sHtML
tv.blog.cvvliy.cn/Article/details/442042.sHtML
tv.blog.cvvliy.cn/Article/details/862846.sHtML
tv.blog.cvvliy.cn/Article/details/682266.sHtML
tv.blog.cvvliy.cn/Article/details/175551.sHtML
tv.blog.cvvliy.cn/Article/details/442480.sHtML
tv.blog.cvvliy.cn/Article/details/155517.sHtML
tv.blog.cvvliy.cn/Article/details/735179.sHtML
tv.blog.cvvliy.cn/Article/details/228028.sHtML
tv.blog.cvvliy.cn/Article/details/137735.sHtML
tv.blog.cvvliy.cn/Article/details/886024.sHtML
tv.blog.cvvliy.cn/Article/details/824288.sHtML
tv.blog.cvvliy.cn/Article/details/975313.sHtML
tv.blog.cvvliy.cn/Article/details/991577.sHtML
tv.blog.cvvliy.cn/Article/details/777399.sHtML
tv.blog.cvvliy.cn/Article/details/513355.sHtML
tv.blog.cvvliy.cn/Article/details/957911.sHtML
tv.blog.cvvliy.cn/Article/details/826088.sHtML
tv.blog.cvvliy.cn/Article/details/759751.sHtML
tv.blog.cvvliy.cn/Article/details/404004.sHtML
tv.blog.cvvliy.cn/Article/details/951311.sHtML
tv.blog.cvvliy.cn/Article/details/115379.sHtML
tv.blog.cvvliy.cn/Article/details/353351.sHtML
tv.blog.cvvliy.cn/Article/details/288406.sHtML
tv.blog.cvvliy.cn/Article/details/664660.sHtML
tv.blog.cvvliy.cn/Article/details/006224.sHtML
tv.blog.cvvliy.cn/Article/details/517313.sHtML
tv.blog.cvvliy.cn/Article/details/464484.sHtML
tv.blog.cvvliy.cn/Article/details/753139.sHtML
tv.blog.cvvliy.cn/Article/details/117799.sHtML
tv.blog.cvvliy.cn/Article/details/666464.sHtML
tv.blog.cvvliy.cn/Article/details/480868.sHtML
tv.blog.cvvliy.cn/Article/details/993755.sHtML
tv.blog.cvvliy.cn/Article/details/888684.sHtML
tv.blog.cvvliy.cn/Article/details/426446.sHtML
tv.blog.cvvliy.cn/Article/details/195171.sHtML
tv.blog.cvvliy.cn/Article/details/919355.sHtML
tv.blog.cvvliy.cn/Article/details/644864.sHtML
tv.blog.cvvliy.cn/Article/details/755555.sHtML
tv.blog.cvvliy.cn/Article/details/826288.sHtML
tv.blog.cvvliy.cn/Article/details/353951.sHtML
tv.blog.cvvliy.cn/Article/details/373111.sHtML
tv.blog.cvvliy.cn/Article/details/511311.sHtML
tv.blog.cvvliy.cn/Article/details/315133.sHtML
tv.blog.cvvliy.cn/Article/details/048426.sHtML
tv.blog.cvvliy.cn/Article/details/791177.sHtML
tv.blog.cvvliy.cn/Article/details/175113.sHtML
tv.blog.cvvliy.cn/Article/details/351395.sHtML
tv.blog.cvvliy.cn/Article/details/262682.sHtML
tv.blog.cvvliy.cn/Article/details/797151.sHtML
tv.blog.cvvliy.cn/Article/details/008426.sHtML
tv.blog.cvvliy.cn/Article/details/882280.sHtML
tv.blog.cvvliy.cn/Article/details/711951.sHtML
tv.blog.cvvliy.cn/Article/details/808200.sHtML
tv.blog.cvvliy.cn/Article/details/606642.sHtML
tv.blog.cvvliy.cn/Article/details/111757.sHtML
tv.blog.cvvliy.cn/Article/details/755735.sHtML
tv.blog.cvvliy.cn/Article/details/066846.sHtML
tv.blog.cvvliy.cn/Article/details/759731.sHtML
tv.blog.cvvliy.cn/Article/details/420006.sHtML
tv.blog.cvvliy.cn/Article/details/204820.sHtML
tv.blog.cvvliy.cn/Article/details/462082.sHtML
tv.blog.cvvliy.cn/Article/details/608428.sHtML
tv.blog.cvvliy.cn/Article/details/266644.sHtML
tv.blog.cvvliy.cn/Article/details/197931.sHtML
tv.blog.cvvliy.cn/Article/details/868664.sHtML
tv.blog.cvvliy.cn/Article/details/680448.sHtML
tv.blog.cvvliy.cn/Article/details/735135.sHtML
tv.blog.cvvliy.cn/Article/details/959773.sHtML
tv.blog.cvvliy.cn/Article/details/151377.sHtML
tv.blog.cvvliy.cn/Article/details/377755.sHtML
tv.blog.cvvliy.cn/Article/details/604806.sHtML
tv.blog.cvvliy.cn/Article/details/864462.sHtML
tv.blog.cvvliy.cn/Article/details/535357.sHtML
tv.blog.cvvliy.cn/Article/details/260466.sHtML
tv.blog.cvvliy.cn/Article/details/753957.sHtML
tv.blog.cvvliy.cn/Article/details/577159.sHtML
tv.blog.cvvliy.cn/Article/details/666608.sHtML
tv.blog.cvvliy.cn/Article/details/591335.sHtML
tv.blog.cvvliy.cn/Article/details/199155.sHtML
tv.blog.cvvliy.cn/Article/details/759537.sHtML
tv.blog.cvvliy.cn/Article/details/713735.sHtML
tv.blog.cvvliy.cn/Article/details/204640.sHtML
tv.blog.cvvliy.cn/Article/details/226266.sHtML
tv.blog.cvvliy.cn/Article/details/591957.sHtML
tv.blog.cvvliy.cn/Article/details/117373.sHtML
tv.blog.cvvliy.cn/Article/details/573353.sHtML
tv.blog.cvvliy.cn/Article/details/408420.sHtML
tv.blog.cvvliy.cn/Article/details/028642.sHtML
tv.blog.cvvliy.cn/Article/details/933751.sHtML
tv.blog.cvvliy.cn/Article/details/113375.sHtML
tv.blog.cvvliy.cn/Article/details/226844.sHtML
tv.blog.cvvliy.cn/Article/details/573315.sHtML
tv.blog.cvvliy.cn/Article/details/804446.sHtML
tv.blog.cvvliy.cn/Article/details/375995.sHtML
tv.blog.cvvliy.cn/Article/details/464466.sHtML
tv.blog.cvvliy.cn/Article/details/020844.sHtML
tv.blog.cvvliy.cn/Article/details/686840.sHtML
tv.blog.cvvliy.cn/Article/details/664080.sHtML
tv.blog.cvvliy.cn/Article/details/084440.sHtML
tv.blog.cvvliy.cn/Article/details/886042.sHtML
tv.blog.cvvliy.cn/Article/details/408042.sHtML
tv.blog.cvvliy.cn/Article/details/777751.sHtML
tv.blog.cvvliy.cn/Article/details/111519.sHtML
tv.blog.cvvliy.cn/Article/details/575173.sHtML
tv.blog.cvvliy.cn/Article/details/519955.sHtML
tv.blog.cvvliy.cn/Article/details/866602.sHtML
tv.blog.cvvliy.cn/Article/details/040820.sHtML
tv.blog.cvvliy.cn/Article/details/008806.sHtML
tv.blog.cvvliy.cn/Article/details/175733.sHtML
tv.blog.cvvliy.cn/Article/details/808626.sHtML
tv.blog.cvvliy.cn/Article/details/068842.sHtML
tv.blog.cvvliy.cn/Article/details/531597.sHtML
tv.blog.cvvliy.cn/Article/details/773919.sHtML
tv.blog.cvvliy.cn/Article/details/260844.sHtML
tv.blog.cvvliy.cn/Article/details/042646.sHtML
tv.blog.cvvliy.cn/Article/details/139915.sHtML
tv.blog.cvvliy.cn/Article/details/448488.sHtML
tv.blog.cvvliy.cn/Article/details/220264.sHtML
tv.blog.cvvliy.cn/Article/details/264802.sHtML
tv.blog.cvvliy.cn/Article/details/800646.sHtML
tv.blog.cvvliy.cn/Article/details/379357.sHtML
tv.blog.cvvliy.cn/Article/details/993793.sHtML
tv.blog.cvvliy.cn/Article/details/060266.sHtML
tv.blog.cvvliy.cn/Article/details/771917.sHtML
tv.blog.cvvliy.cn/Article/details/373797.sHtML
tv.blog.cvvliy.cn/Article/details/224668.sHtML
tv.blog.cvvliy.cn/Article/details/648068.sHtML
tv.blog.cvvliy.cn/Article/details/888202.sHtML
tv.blog.cvvliy.cn/Article/details/464882.sHtML
tv.blog.cvvliy.cn/Article/details/917739.sHtML
tv.blog.cvvliy.cn/Article/details/400224.sHtML
tv.blog.cvvliy.cn/Article/details/882808.sHtML
tv.blog.cvvliy.cn/Article/details/404024.sHtML
tv.blog.cvvliy.cn/Article/details/004860.sHtML
tv.blog.cvvliy.cn/Article/details/446604.sHtML
tv.blog.cvvliy.cn/Article/details/844888.sHtML
tv.blog.cvvliy.cn/Article/details/593317.sHtML
tv.blog.cvvliy.cn/Article/details/026622.sHtML
tv.blog.cvvliy.cn/Article/details/153937.sHtML
tv.blog.cvvliy.cn/Article/details/537571.sHtML
tv.blog.cvvliy.cn/Article/details/202602.sHtML
tv.blog.cvvliy.cn/Article/details/884804.sHtML
tv.blog.cvvliy.cn/Article/details/682644.sHtML
tv.blog.cvvliy.cn/Article/details/600006.sHtML
tv.blog.cvvliy.cn/Article/details/915597.sHtML
tv.blog.cvvliy.cn/Article/details/262206.sHtML
tv.blog.cvvliy.cn/Article/details/824222.sHtML
tv.blog.cvvliy.cn/Article/details/157517.sHtML
tv.blog.cvvliy.cn/Article/details/466200.sHtML
tv.blog.cvvliy.cn/Article/details/000266.sHtML
tv.blog.cvvliy.cn/Article/details/157311.sHtML
tv.blog.cvvliy.cn/Article/details/733937.sHtML
tv.blog.cvvliy.cn/Article/details/137773.sHtML
tv.blog.cvvliy.cn/Article/details/620460.sHtML
tv.blog.cvvliy.cn/Article/details/066244.sHtML
tv.blog.cvvliy.cn/Article/details/719977.sHtML
tv.blog.cvvliy.cn/Article/details/226248.sHtML
tv.blog.cvvliy.cn/Article/details/557193.sHtML
tv.blog.cvvliy.cn/Article/details/913595.sHtML
tv.blog.cvvliy.cn/Article/details/684668.sHtML
tv.blog.cvvliy.cn/Article/details/719535.sHtML
tv.blog.cvvliy.cn/Article/details/624024.sHtML
tv.blog.cvvliy.cn/Article/details/648420.sHtML
tv.blog.cvvliy.cn/Article/details/640668.sHtML
tv.blog.cvvliy.cn/Article/details/846608.sHtML
tv.blog.cvvliy.cn/Article/details/311391.sHtML
tv.blog.cvvliy.cn/Article/details/319955.sHtML
tv.blog.cvvliy.cn/Article/details/846640.sHtML
tv.blog.cvvliy.cn/Article/details/026026.sHtML
tv.blog.cvvliy.cn/Article/details/802204.sHtML
tv.blog.cvvliy.cn/Article/details/020624.sHtML
tv.blog.cvvliy.cn/Article/details/006082.sHtML
tv.blog.cvvliy.cn/Article/details/599593.sHtML
tv.blog.cvvliy.cn/Article/details/626020.sHtML
tv.blog.cvvliy.cn/Article/details/040286.sHtML
tv.blog.cvvliy.cn/Article/details/399157.sHtML
tv.blog.cvvliy.cn/Article/details/864660.sHtML
tv.blog.cvvliy.cn/Article/details/397373.sHtML
tv.blog.cvvliy.cn/Article/details/797357.sHtML
tv.blog.cvvliy.cn/Article/details/193171.sHtML
tv.blog.cvvliy.cn/Article/details/088246.sHtML
tv.blog.cvvliy.cn/Article/details/486668.sHtML
tv.blog.cvvliy.cn/Article/details/466468.sHtML
tv.blog.cvvliy.cn/Article/details/755953.sHtML
tv.blog.cvvliy.cn/Article/details/640820.sHtML
tv.blog.cvvliy.cn/Article/details/779719.sHtML
