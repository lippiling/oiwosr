寄谱忻诜卓


  
抽象类不能实例化，编译器禁止new操作，因为它表达了不完整的概念(如Animal)；它通过abstract强制子类实现，并通过具体方法重用逻辑，支持状态共享、构建初始化和继承约束。

抽象类在 Java 它不是用来制造对象的，而是用来“制定规则” + 给出基础——强制子类实现关键行为，同时重用公共逻辑。


为什么不能直接？ new 抽象类？
由于抽象类表达了不完整的概念，例如： Animal、Shape、DocumentProcessor，它们没有具体的形式，编译器会阻止你写作 new Animal() 这个代码，否则语义就乱了。

编译时报错误：Cannot instantiate the type Animal

抽象可以有结构方法，但只供子类使用 super() 用于初始化共享字段的调用(例如 age、color）
即使一个抽象类没有写任何东西 abstract 只要添加了方法 abstract 关键词不能实例化

如何配合抽象方法和具体方法？
抽象的方法是“必须重写”的合同，具体的方法是“可以使用”的共享能力。两者混合在一个类别中，以考虑到约束力和实用性。
public abstract class DataExporter {
protected String format; // 共享状态

public DataExporter(String format) {
this.format = format;
}

// 具体方法：所有子类都使用相同的验证逻辑
public final void export(Data data) {
if (data == null) throw new IllegalArgumentException("data must not be null");
doExport(data); // 委托给子类实现
}

// 抽象方法：不同的格式，导出动作必然不同
protected abstract void doExport(Data data);
}


export() 是模板入口，加 final 防止子类破坏过程

doExport() 是钩子，由 JsonExporter、CsvExporter 各自实现
若完全使用接口，则必须在每个实现类中重复写入 if (data == null) 验证；如果全部使用普通类，则不能强制子类提供 doExport


什么时候选择抽象而不是接口？
当需要传递状态、重用实现或构造逻辑时，接口不能，只能依靠抽象。
			
		
；

需要共享字段吗？→ 接口不能有实例变量，abstract class Shape { protected double scale; } 可以
要统一结构过程？→ 界面没有结构器，抽象类可以定义 protected Shape(String name)

有一些逻辑，但不想暴露细节？→ 抽象类能写 private 方法或 final 方法，接口不好（Java 17 仍不支持 private final 方法）
想限制继承链的深度吗？→ 抽象自然是单继承的主线，适合建模is-a”关系（如 Dog extends Animal），接口更适合“接口”can-do”能力（如 Runnable, Serializable）

最容易被忽视的一点是，抽象结构器不是装饰品。如果你在抽象类中写了一个参考结构，子类必须显式调用 super(...)，这实际上是一种隐性的初始合同——它比文档更可靠，比运行时检查更早暴露问题。	 

豪偌磺巳峦聘诮狡冶矩照吹庇显寻

elt.hjiocz.cn/919533.htm
elt.hjiocz.cn/979973.htm
elt.hjiocz.cn/717573.htm
elt.hjiocz.cn/860443.htm
elt.hjiocz.cn/793953.htm
elr.hjiocz.cn/280003.htm
elr.hjiocz.cn/599553.htm
elr.hjiocz.cn/806403.htm
elr.hjiocz.cn/559153.htm
elr.hjiocz.cn/395953.htm
elr.hjiocz.cn/335973.htm
elr.hjiocz.cn/159573.htm
elr.hjiocz.cn/939933.htm
elr.hjiocz.cn/357953.htm
elr.hjiocz.cn/571593.htm
ele.hjiocz.cn/579753.htm
ele.hjiocz.cn/004403.htm
ele.hjiocz.cn/171993.htm
ele.hjiocz.cn/391933.htm
ele.hjiocz.cn/737933.htm
ele.hjiocz.cn/531173.htm
ele.hjiocz.cn/200483.htm
ele.hjiocz.cn/931553.htm
ele.hjiocz.cn/333553.htm
ele.hjiocz.cn/842443.htm
elw.hjiocz.cn/979533.htm
elw.hjiocz.cn/131913.htm
elw.hjiocz.cn/208223.htm
elw.hjiocz.cn/957793.htm
elw.hjiocz.cn/399133.htm
elw.hjiocz.cn/155113.htm
elw.hjiocz.cn/319173.htm
elw.hjiocz.cn/808843.htm
elw.hjiocz.cn/282003.htm
elw.hjiocz.cn/939153.htm
elq.hjiocz.cn/606883.htm
elq.hjiocz.cn/399713.htm
elq.hjiocz.cn/931173.htm
elq.hjiocz.cn/597133.htm
elq.hjiocz.cn/713313.htm
elq.hjiocz.cn/757533.htm
elq.hjiocz.cn/993373.htm
elq.hjiocz.cn/911793.htm
elq.hjiocz.cn/995753.htm
elq.hjiocz.cn/537933.htm
ekm.hjiocz.cn/531353.htm
ekm.hjiocz.cn/775113.htm
ekm.hjiocz.cn/240263.htm
ekm.hjiocz.cn/937733.htm
ekm.hjiocz.cn/919953.htm
ekm.hjiocz.cn/668843.htm
ekm.hjiocz.cn/486023.htm
ekm.hjiocz.cn/139533.htm
ekm.hjiocz.cn/793513.htm
ekm.hjiocz.cn/135533.htm
ekn.hjiocz.cn/391953.htm
ekn.hjiocz.cn/977193.htm
ekn.hjiocz.cn/913533.htm
ekn.hjiocz.cn/939733.htm
ekn.hjiocz.cn/557113.htm
ekn.hjiocz.cn/737153.htm
ekn.hjiocz.cn/759173.htm
ekn.hjiocz.cn/199113.htm
ekn.hjiocz.cn/911373.htm
ekn.hjiocz.cn/351133.htm
ekb.hjiocz.cn/199193.htm
ekb.hjiocz.cn/444443.htm
ekb.hjiocz.cn/953533.htm
ekb.hjiocz.cn/571793.htm
ekb.hjiocz.cn/375933.htm
ekb.hjiocz.cn/040223.htm
ekb.hjiocz.cn/355933.htm
ekb.hjiocz.cn/226843.htm
ekb.hjiocz.cn/933753.htm
ekb.hjiocz.cn/73.htm
ekv.hjiocz.cn/339753.htm
ekv.hjiocz.cn/919753.htm
ekv.hjiocz.cn/608683.htm
ekv.hjiocz.cn/157753.htm
ekv.hjiocz.cn/317953.htm
ekv.hjiocz.cn/539513.htm
ekv.hjiocz.cn/315353.htm
ekv.hjiocz.cn/353553.htm
ekv.hjiocz.cn/246883.htm
ekv.hjiocz.cn/713133.htm
ekc.hjiocz.cn/591953.htm
ekc.hjiocz.cn/593333.htm
ekc.hjiocz.cn/791933.htm
ekc.hjiocz.cn/571933.htm
ekc.hjiocz.cn/915573.htm
ekc.hjiocz.cn/335153.htm
ekc.hjiocz.cn/553313.htm
ekc.hjiocz.cn/755553.htm
ekc.hjiocz.cn/795173.htm
ekc.hjiocz.cn/919593.htm
ekx.hjiocz.cn/628263.htm
ekx.hjiocz.cn/111333.htm
ekx.hjiocz.cn/020023.htm
ekx.hjiocz.cn/402263.htm
ekx.hjiocz.cn/551953.htm
ekx.hjiocz.cn/595593.htm
ekx.hjiocz.cn/397173.htm
ekx.hjiocz.cn/197353.htm
ekx.hjiocz.cn/480083.htm
ekx.hjiocz.cn/357193.htm
ekz.hjiocz.cn/393573.htm
ekz.hjiocz.cn/375173.htm
ekz.hjiocz.cn/995593.htm
ekz.hjiocz.cn/711173.htm
ekz.hjiocz.cn/000063.htm
ekz.hjiocz.cn/591173.htm
ekz.hjiocz.cn/424823.htm
ekz.hjiocz.cn/597353.htm
ekz.hjiocz.cn/579713.htm
ekz.hjiocz.cn/137333.htm
ekl.hjiocz.cn/773713.htm
ekl.hjiocz.cn/157993.htm
ekl.hjiocz.cn/535193.htm
ekl.hjiocz.cn/137373.htm
ekl.hjiocz.cn/913353.htm
ekl.hjiocz.cn/428243.htm
ekl.hjiocz.cn/731793.htm
ekl.hjiocz.cn/333533.htm
ekl.hjiocz.cn/731333.htm
ekl.hjiocz.cn/531173.htm
ekk.hjiocz.cn/913553.htm
ekk.hjiocz.cn/042623.htm
ekk.hjiocz.cn/157913.htm
ekk.hjiocz.cn/040203.htm
ekk.hjiocz.cn/319353.htm
ekk.hjiocz.cn/131333.htm
ekk.hjiocz.cn/311953.htm
ekk.hjiocz.cn/391173.htm
ekk.hjiocz.cn/391353.htm
ekk.hjiocz.cn/204463.htm
ekj.hjiocz.cn/735773.htm
ekj.hjiocz.cn/133333.htm
ekj.hjiocz.cn/266443.htm
ekj.hjiocz.cn/751553.htm
ekj.hjiocz.cn/177973.htm
ekj.hjiocz.cn/115913.htm
ekj.hjiocz.cn/371393.htm
ekj.hjiocz.cn/000403.htm
ekj.hjiocz.cn/551333.htm
ekj.hjiocz.cn/882283.htm
ekh.hjiocz.cn/797353.htm
ekh.hjiocz.cn/393793.htm
ekh.hjiocz.cn/311573.htm
ekh.hjiocz.cn/042203.htm
ekh.hjiocz.cn/171993.htm
ekh.hjiocz.cn/680463.htm
ekh.hjiocz.cn/915593.htm
ekh.hjiocz.cn/533333.htm
ekh.hjiocz.cn/573573.htm
ekh.hjiocz.cn/955933.htm
ekg.hjiocz.cn/975393.htm
ekg.hjiocz.cn/048603.htm
ekg.hjiocz.cn/997113.htm
ekg.hjiocz.cn/866823.htm
ekg.hjiocz.cn/171913.htm
ekg.hjiocz.cn/064043.htm
ekg.hjiocz.cn/486663.htm
ekg.hjiocz.cn/428883.htm
ekg.hjiocz.cn/113933.htm
ekg.hjiocz.cn/646003.htm
ekf.hjiocz.cn/111993.htm
ekf.hjiocz.cn/913593.htm
ekf.hjiocz.cn/248003.htm
ekf.hjiocz.cn/931973.htm
ekf.hjiocz.cn/351713.htm
ekf.hjiocz.cn/008083.htm
ekf.hjiocz.cn/779373.htm
ekf.hjiocz.cn/191753.htm
ekf.hjiocz.cn/975533.htm
ekf.hjiocz.cn/119153.htm
ekd.hjiocz.cn/179313.htm
ekd.hjiocz.cn/882043.htm
ekd.hjiocz.cn/420683.htm
ekd.hjiocz.cn/975113.htm
ekd.hjiocz.cn/151953.htm
ekd.hjiocz.cn/115713.htm
ekd.hjiocz.cn/973393.htm
ekd.hjiocz.cn/353753.htm
ekd.hjiocz.cn/371713.htm
ekd.hjiocz.cn/820243.htm
eks.hjiocz.cn/939513.htm
eks.hjiocz.cn/571533.htm
eks.hjiocz.cn/602463.htm
eks.hjiocz.cn/882403.htm
eks.hjiocz.cn/933993.htm
eks.hjiocz.cn/971373.htm
eks.hjiocz.cn/191333.htm
eks.hjiocz.cn/604843.htm
eks.hjiocz.cn/046863.htm
eks.hjiocz.cn/999533.htm
eka.hjiocz.cn/115593.htm
eka.hjiocz.cn/979913.htm
eka.hjiocz.cn/531393.htm
eka.hjiocz.cn/802283.htm
eka.hjiocz.cn/397173.htm
eka.hjiocz.cn/795113.htm
eka.hjiocz.cn/422643.htm
eka.hjiocz.cn/664083.htm
eka.hjiocz.cn/731933.htm
eka.hjiocz.cn/644063.htm
ekp.hjiocz.cn/373353.htm
ekp.hjiocz.cn/179313.htm
ekp.hjiocz.cn/242683.htm
ekp.hjiocz.cn/884863.htm
ekp.hjiocz.cn/553973.htm
ekp.hjiocz.cn/111713.htm
ekp.hjiocz.cn/117313.htm
ekp.hjiocz.cn/533373.htm
ekp.hjiocz.cn/464443.htm
ekp.hjiocz.cn/579753.htm
eko.hjiocz.cn/359533.htm
eko.hjiocz.cn/159733.htm
eko.hjiocz.cn/937733.htm
eko.hjiocz.cn/955973.htm
eko.hjiocz.cn/664803.htm
eko.hjiocz.cn/575753.htm
eko.hjiocz.cn/002663.htm
eko.hjiocz.cn/537133.htm
eko.hjiocz.cn/719593.htm
eko.hjiocz.cn/193313.htm
eki.hjiocz.cn/755933.htm
eki.hjiocz.cn/599793.htm
eki.hjiocz.cn/955793.htm
eki.hjiocz.cn/155373.htm
eki.hjiocz.cn/933593.htm
eki.hjiocz.cn/513593.htm
eki.hjiocz.cn/931793.htm
eki.hjiocz.cn/731773.htm
eki.hjiocz.cn/179733.htm
eki.hjiocz.cn/191733.htm
eku.hjiocz.cn/242823.htm
eku.hjiocz.cn/357353.htm
eku.hjiocz.cn/024843.htm
eku.hjiocz.cn/084883.htm
eku.hjiocz.cn/557713.htm
eku.hjiocz.cn/531573.htm
eku.hjiocz.cn/157533.htm
eku.hjiocz.cn/951553.htm
eku.hjiocz.cn/397573.htm
eku.hjiocz.cn/371713.htm
eky.hjiocz.cn/153553.htm
eky.hjiocz.cn/797973.htm
eky.hjiocz.cn/195513.htm
eky.hjiocz.cn/795533.htm
eky.hjiocz.cn/751713.htm
eky.hjiocz.cn/719173.htm
eky.hjiocz.cn/375993.htm
eky.hjiocz.cn/179713.htm
eky.hjiocz.cn/537393.htm
eky.hjiocz.cn/795373.htm
ekt.hjiocz.cn/797533.htm
ekt.hjiocz.cn/559553.htm
ekt.hjiocz.cn/995393.htm
ekt.hjiocz.cn/997993.htm
ekt.hjiocz.cn/317553.htm
ekt.hjiocz.cn/997773.htm
ekt.hjiocz.cn/319113.htm
ekt.hjiocz.cn/719333.htm
ekt.hjiocz.cn/515313.htm
ekt.hjiocz.cn/668803.htm
ekr.hjiocz.cn/202023.htm
ekr.hjiocz.cn/593193.htm
ekr.hjiocz.cn/002283.htm
ekr.hjiocz.cn/773133.htm
ekr.hjiocz.cn/991793.htm
ekr.hjiocz.cn/579333.htm
ekr.hjiocz.cn/339533.htm
ekr.hjiocz.cn/911793.htm
ekr.hjiocz.cn/773593.htm
ekr.hjiocz.cn/551913.htm
eke.hjiocz.cn/995333.htm
eke.hjiocz.cn/208643.htm
eke.hjiocz.cn/595773.htm
eke.hjiocz.cn/539713.htm
eke.hjiocz.cn/668683.htm
eke.hjiocz.cn/517553.htm
eke.hjiocz.cn/337373.htm
eke.hjiocz.cn/975593.htm
eke.hjiocz.cn/157913.htm
eke.hjiocz.cn/642423.htm
ekw.hjiocz.cn/719533.htm
ekw.hjiocz.cn/511513.htm
ekw.hjiocz.cn/995513.htm
ekw.hjiocz.cn/935993.htm
ekw.hjiocz.cn/395773.htm
ekw.hjiocz.cn/755193.htm
ekw.hjiocz.cn/355153.htm
ekw.hjiocz.cn/953153.htm
ekw.hjiocz.cn/975973.htm
ekw.hjiocz.cn/371193.htm
ekq.hjiocz.cn/111593.htm
ekq.hjiocz.cn/464063.htm
ekq.hjiocz.cn/771133.htm
ekq.hjiocz.cn/406043.htm
ekq.hjiocz.cn/806283.htm
ekq.hjiocz.cn/199913.htm
ekq.hjiocz.cn/539713.htm
ekq.hjiocz.cn/337513.htm
ekq.hjiocz.cn/395753.htm
ekq.hjiocz.cn/917113.htm
ejm.hjiocz.cn/406283.htm
ejm.hjiocz.cn/739593.htm
ejm.hjiocz.cn/951713.htm
ejm.hjiocz.cn/939113.htm
ejm.hjiocz.cn/975133.htm
ejm.hjiocz.cn/173593.htm
ejm.hjiocz.cn/753993.htm
ejm.hjiocz.cn/575713.htm
ejm.hjiocz.cn/395953.htm
ejm.hjiocz.cn/933713.htm
ejn.hjiocz.cn/393173.htm
ejn.hjiocz.cn/551153.htm
ejn.hjiocz.cn/222263.htm
ejn.hjiocz.cn/311513.htm
ejn.hjiocz.cn/135113.htm
ejn.hjiocz.cn/846403.htm
ejn.hjiocz.cn/624403.htm
ejn.hjiocz.cn/135513.htm
ejn.hjiocz.cn/200083.htm
ejn.hjiocz.cn/711773.htm
ejb.hjiocz.cn/806243.htm
ejb.hjiocz.cn/464263.htm
ejb.hjiocz.cn/351773.htm
ejb.hjiocz.cn/319553.htm
ejb.hjiocz.cn/353593.htm
ejb.hjiocz.cn/977593.htm
ejb.hjiocz.cn/622083.htm
ejb.hjiocz.cn/337933.htm
ejb.hjiocz.cn/571373.htm
ejb.hjiocz.cn/395113.htm
ejv.hjiocz.cn/335193.htm
ejv.hjiocz.cn/517733.htm
ejv.hjiocz.cn/084283.htm
ejv.hjiocz.cn/539393.htm
ejv.hjiocz.cn/977593.htm
ejv.hjiocz.cn/357573.htm
ejv.hjiocz.cn/519373.htm
ejv.hjiocz.cn/515773.htm
ejv.hjiocz.cn/315353.htm
ejv.hjiocz.cn/533153.htm
ejc.hjiocz.cn/179373.htm
ejc.hjiocz.cn/171513.htm
ejc.hjiocz.cn/333533.htm
ejc.hjiocz.cn/195973.htm
ejc.hjiocz.cn/159753.htm
ejc.hjiocz.cn/606003.htm
ejc.hjiocz.cn/199153.htm
ejc.hjiocz.cn/442203.htm
ejc.hjiocz.cn/395913.htm
ejc.hjiocz.cn/777993.htm
ejx.hjiocz.cn/682003.htm
ejx.hjiocz.cn/973713.htm
ejx.hjiocz.cn/717353.htm
ejx.hjiocz.cn/175933.htm
ejx.hjiocz.cn/355133.htm
ejx.hjiocz.cn/913933.htm
ejx.hjiocz.cn/824623.htm
ejx.hjiocz.cn/335173.htm
ejx.hjiocz.cn/042443.htm
ejx.hjiocz.cn/931133.htm
ejz.hjiocz.cn/157373.htm
ejz.hjiocz.cn/939113.htm
ejz.hjiocz.cn/533133.htm
ejz.hjiocz.cn/193313.htm
ejz.hjiocz.cn/797513.htm
ejz.hjiocz.cn/004863.htm
ejz.hjiocz.cn/993113.htm
ejz.hjiocz.cn/797713.htm
ejz.hjiocz.cn/913713.htm
ejz.hjiocz.cn/555733.htm
ejl.hjiocz.cn/791353.htm
ejl.hjiocz.cn/795913.htm
ejl.hjiocz.cn/197573.htm
ejl.hjiocz.cn/733593.htm
ejl.hjiocz.cn/175333.htm
ejl.hjiocz.cn/955953.htm
ejl.hjiocz.cn/735933.htm
ejl.hjiocz.cn/517173.htm
ejl.hjiocz.cn/959533.htm
ejl.hjiocz.cn/399513.htm
ejk.hjiocz.cn/444263.htm
ejk.hjiocz.cn/731513.htm
ejk.hjiocz.cn/686463.htm
ejk.hjiocz.cn/973553.htm
ejk.hjiocz.cn/151973.htm
ejk.hjiocz.cn/579933.htm
ejk.hjiocz.cn/846223.htm
ejk.hjiocz.cn/117753.htm
ejk.hjiocz.cn/646483.htm
ejk.hjiocz.cn/575733.htm
