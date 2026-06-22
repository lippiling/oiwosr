肛缚谈沾痉


  
构造方法不是对象初始化的唯一入口。JVM在调用前已经分配了内存，设置了默认值，并实施了家庭结构链；构造前实施了字段初始化和实例块，并绕过构造方法创建了对象。

构造方法并非“初始化的唯一入口”
许多初学者误以为 new MyClass() 之后，代码将从构造方法的第一行开始执行。事实上，JVM 在调用结构方法之前，已经隐约完成了几件事:分配内存，将所有字段放置为默认值（0、null、false）、实施父类结构链(直到 Object）。构造方法只是你“可以插入自定义逻辑”的最早可控节点，而不是对象生命周期的起点。
这意味着：

字段声明中的初始表达(如 private List list = new ArrayList();）结构方法体实施前将完成，但顺序在父类结构调用后和此类结构体之前；
若父类结构抛出异常，您的结构方法体根本不会执行；
使用 Unsafe.allocateInstance() 或反序列化(如 ObjectInputStream）可以绕过构造方法创建对象——此时字段都是默认值，没有初始逻辑操作。

在构造方法链中 this(...) 和 super(...) 的限制
Java 每种结构方法的第一句话必须是显式或隐式的结构调用：this(...)(此类其他结构)或 super(...)(父体结构)。没写？编译器自动补充。 super();。但是两者不能共存，也不能出现在条件分支中。
常见错误包括：
；

在 if 块内写 this(...) → 编译报错：call to this must be first statement；
父类没有参与结构，子类结构没有显式调用 super(...) → 编译失败：constructor Parent() is undefined；
递归调用 this(...)（比如 A 调 B，B 又调 A）→ 拒绝编译期，不等操作。

本质是 JVM 构造路径的拓扑顺序需要明确，以确保字段初始化和继承链的可控性。

实例初始化块比构造方法更早执行
Java 允许用 { ... } 初始化块的定义实例（instance initializer block），它将被编译器“复制”到每个结构方法体的开头(在显式中 super(...) 或 this(...) 之后，在其他代码之前)。
			
		
public class Example {
private int a = 1;// 字段初始化
{ System.out.println("init block"); }  // 实例初始化块
public Example() {
System.out.println("ctor body");
}
}
执行 new Example() 输出顺序如下：


init block(因为字段初始化后，构造体前)
ctor body

该机制常用于避免在多种结构方法中重复同样的初始化逻辑；或匿名内部类/lambda 当捕获外部变量有限时，应进行轻量预处理。但请注意，除非使用，否则不能接收参数或接受异常检测（除非使用） try-catch 包裹）。

静态字段和静态初始化块只执行一次，早于任何结构
静态成员属于类，不属于对象。JVM 第一次主动使用这个类(例如) new、触发类初始化时调用静态方法，访问静态字段)，按源代码顺序执行:

静态字段的初始表达(如 static int x = calc();）
静态初始化块（static { ... }）

即使你完全解耦了这个过程和结构方法。 never new 对象，只要引用静态字段，类就会初始化；相反，new 静态部分只执行10万个对象一次。
容易被忽视的点：

引用父类静态成员的子类不会触发子类初始化(只触发父类)；
数组类型(如 Example[]）不会触发 Example 类初始化；
常量（static final int X = 42;）内联在编译期内，不触发类初始化。

对象初始化真正复杂的不是语法糖，而是在这些不同阶段的交织顺序和可见性规则（类加载、内存分配、字段默认值、静态初始化、实例初始化和结构）。如果你有点粗心，你可能会在多线程下看到未完全初始化的对象状态。	 

赂眉矫硬诠诙督灰屑滓涌诠荚恳雷

ehr.hjiocz.cn/379333.htm
ehr.hjiocz.cn/197713.htm
ehr.hjiocz.cn/208283.htm
ehr.hjiocz.cn/173913.htm
ehr.hjiocz.cn/531733.htm
ehe.hjiocz.cn/335533.htm
ehe.hjiocz.cn/933313.htm
ehe.hjiocz.cn/620663.htm
ehe.hjiocz.cn/264463.htm
ehe.hjiocz.cn/999733.htm
ehe.hjiocz.cn/460683.htm
ehe.hjiocz.cn/115133.htm
ehe.hjiocz.cn/999533.htm
ehe.hjiocz.cn/064663.htm
ehe.hjiocz.cn/791913.htm
ehw.hjiocz.cn/080263.htm
ehw.hjiocz.cn/115513.htm
ehw.hjiocz.cn/264263.htm
ehw.hjiocz.cn/375513.htm
ehw.hjiocz.cn/377993.htm
ehw.hjiocz.cn/646883.htm
ehw.hjiocz.cn/913793.htm
ehw.hjiocz.cn/804803.htm
ehw.hjiocz.cn/620843.htm
ehw.hjiocz.cn/713313.htm
ehq.hjiocz.cn/402463.htm
ehq.hjiocz.cn/680623.htm
ehq.hjiocz.cn/020203.htm
ehq.hjiocz.cn/646003.htm
ehq.hjiocz.cn/088223.htm
ehq.hjiocz.cn/226023.htm
ehq.hjiocz.cn/066083.htm
ehq.hjiocz.cn/044623.htm
ehq.hjiocz.cn/646203.htm
ehq.hjiocz.cn/155773.htm
egm.hjiocz.cn/662003.htm
egm.hjiocz.cn/404243.htm
egm.hjiocz.cn/206643.htm
egm.hjiocz.cn/800423.htm
egm.hjiocz.cn/842823.htm
egm.hjiocz.cn/002463.htm
egm.hjiocz.cn/373553.htm
egm.hjiocz.cn/400263.htm
egm.hjiocz.cn/551153.htm
egm.hjiocz.cn/911733.htm
egn.hjiocz.cn/062683.htm
egn.hjiocz.cn/119533.htm
egn.hjiocz.cn/644223.htm
egn.hjiocz.cn/331593.htm
egn.hjiocz.cn/935113.htm
egn.hjiocz.cn/919713.htm
egn.hjiocz.cn/977733.htm
egn.hjiocz.cn/046483.htm
egn.hjiocz.cn/791353.htm
egn.hjiocz.cn/973533.htm
egb.hjiocz.cn/886843.htm
egb.hjiocz.cn/917133.htm
egb.hjiocz.cn/840443.htm
egb.hjiocz.cn/737173.htm
egb.hjiocz.cn/268023.htm
egb.hjiocz.cn/795713.htm
egb.hjiocz.cn/355393.htm
egb.hjiocz.cn/131113.htm
egb.hjiocz.cn/977953.htm
egb.hjiocz.cn/771153.htm
egv.hjiocz.cn/686843.htm
egv.hjiocz.cn/911373.htm
egv.hjiocz.cn/315173.htm
egv.hjiocz.cn/173553.htm
egv.hjiocz.cn/137333.htm
egv.hjiocz.cn/919173.htm
egv.hjiocz.cn/595773.htm
egv.hjiocz.cn/317793.htm
egv.hjiocz.cn/397553.htm
egv.hjiocz.cn/937913.htm
egc.hjiocz.cn/731133.htm
egc.hjiocz.cn/977373.htm
egc.hjiocz.cn/571353.htm
egc.hjiocz.cn/511573.htm
egc.hjiocz.cn/933313.htm
egc.hjiocz.cn/197733.htm
egc.hjiocz.cn/393913.htm
egc.hjiocz.cn/777713.htm
egc.hjiocz.cn/757793.htm
egc.hjiocz.cn/519173.htm
egx.hjiocz.cn/511573.htm
egx.hjiocz.cn/357913.htm
egx.hjiocz.cn/911153.htm
egx.hjiocz.cn/355173.htm
egx.hjiocz.cn/395713.htm
egx.hjiocz.cn/795153.htm
egx.hjiocz.cn/117553.htm
egx.hjiocz.cn/771933.htm
egx.hjiocz.cn/517533.htm
egx.hjiocz.cn/971953.htm
egz.hjiocz.cn/977173.htm
egz.hjiocz.cn/795953.htm
egz.hjiocz.cn/573753.htm
egz.hjiocz.cn/557353.htm
egz.hjiocz.cn/971773.htm
egz.hjiocz.cn/377393.htm
egz.hjiocz.cn/773313.htm
egz.hjiocz.cn/531793.htm
egz.hjiocz.cn/717333.htm
egz.hjiocz.cn/993793.htm
egl.hjiocz.cn/191593.htm
egl.hjiocz.cn/379733.htm
egl.hjiocz.cn/513953.htm
egl.hjiocz.cn/353373.htm
egl.hjiocz.cn/000443.htm
egl.hjiocz.cn/977513.htm
egl.hjiocz.cn/791373.htm
egl.hjiocz.cn/555793.htm
egl.hjiocz.cn/991133.htm
egl.hjiocz.cn/159513.htm
egk.hjiocz.cn/319933.htm
egk.hjiocz.cn/935153.htm
egk.hjiocz.cn/395553.htm
egk.hjiocz.cn/282223.htm
egk.hjiocz.cn/997773.htm
egk.hjiocz.cn/951573.htm
egk.hjiocz.cn/395113.htm
egk.hjiocz.cn/791933.htm
egk.hjiocz.cn/979713.htm
egk.hjiocz.cn/599173.htm
egj.hjiocz.cn/793973.htm
egj.hjiocz.cn/519733.htm
egj.hjiocz.cn/111333.htm
egj.hjiocz.cn/117573.htm
egj.hjiocz.cn/119373.htm
egj.hjiocz.cn/131353.htm
egj.hjiocz.cn/939333.htm
egj.hjiocz.cn/379953.htm
egj.hjiocz.cn/515993.htm
egj.hjiocz.cn/131193.htm
egh.hjiocz.cn/111373.htm
egh.hjiocz.cn/195113.htm
egh.hjiocz.cn/555913.htm
egh.hjiocz.cn/397533.htm
egh.hjiocz.cn/319173.htm
egh.hjiocz.cn/915353.htm
egh.hjiocz.cn/973133.htm
egh.hjiocz.cn/93.htm
egh.hjiocz.cn/682683.htm
egh.hjiocz.cn/282683.htm
egg.hjiocz.cn/226643.htm
egg.hjiocz.cn/739513.htm
egg.hjiocz.cn/793973.htm
egg.hjiocz.cn/995193.htm
egg.hjiocz.cn/844683.htm
egg.hjiocz.cn/339193.htm
egg.hjiocz.cn/999593.htm
egg.hjiocz.cn/951933.htm
egg.hjiocz.cn/626003.htm
egg.hjiocz.cn/933913.htm
egf.mmmxz.cn/533133.htm
egf.mmmxz.cn/828403.htm
egf.mmmxz.cn/599153.htm
egf.mmmxz.cn/515953.htm
egf.mmmxz.cn/404043.htm
egf.mmmxz.cn/351733.htm
egf.mmmxz.cn/591793.htm
egf.mmmxz.cn/377113.htm
egf.mmmxz.cn/173353.htm
egf.mmmxz.cn/753393.htm
egd.mmmxz.cn/957373.htm
egd.mmmxz.cn/799353.htm
egd.mmmxz.cn/288683.htm
egd.mmmxz.cn/571353.htm
egd.mmmxz.cn/991373.htm
egd.mmmxz.cn/533133.htm
egd.mmmxz.cn/151553.htm
egd.mmmxz.cn/795573.htm
egd.mmmxz.cn/537773.htm
egd.mmmxz.cn/460483.htm
egs.mmmxz.cn/317713.htm
egs.mmmxz.cn/175913.htm
egs.mmmxz.cn/331733.htm
egs.mmmxz.cn/339713.htm
egs.mmmxz.cn/648063.htm
egs.mmmxz.cn/197173.htm
egs.mmmxz.cn/115953.htm
egs.mmmxz.cn/888863.htm
egs.mmmxz.cn/555753.htm
egs.mmmxz.cn/175553.htm
ega.mmmxz.cn/379513.htm
ega.mmmxz.cn/131353.htm
ega.mmmxz.cn/608683.htm
ega.mmmxz.cn/971993.htm
ega.mmmxz.cn/951313.htm
ega.mmmxz.cn/048083.htm
ega.mmmxz.cn/395533.htm
ega.mmmxz.cn/466203.htm
ega.mmmxz.cn/955353.htm
ega.mmmxz.cn/359313.htm
egp.mmmxz.cn/608883.htm
egp.mmmxz.cn/955713.htm
egp.mmmxz.cn/993153.htm
egp.mmmxz.cn/682023.htm
egp.mmmxz.cn/133593.htm
egp.mmmxz.cn/202203.htm
egp.mmmxz.cn/731173.htm
egp.mmmxz.cn/359333.htm
egp.mmmxz.cn/468203.htm
egp.mmmxz.cn/339933.htm
ego.mmmxz.cn/862623.htm
ego.mmmxz.cn/222043.htm
ego.mmmxz.cn/175173.htm
ego.mmmxz.cn/573773.htm
ego.mmmxz.cn/991993.htm
ego.mmmxz.cn/268243.htm
ego.mmmxz.cn/115133.htm
ego.mmmxz.cn/153773.htm
ego.mmmxz.cn/113753.htm
ego.mmmxz.cn/759133.htm
egi.mmmxz.cn/753773.htm
egi.mmmxz.cn/731113.htm
egi.mmmxz.cn/808203.htm
egi.mmmxz.cn/959373.htm
egi.mmmxz.cn/553713.htm
egi.mmmxz.cn/797713.htm
egi.mmmxz.cn/262403.htm
egi.mmmxz.cn/286883.htm
egi.mmmxz.cn/377313.htm
egi.mmmxz.cn/286443.htm
egu.mmmxz.cn/511713.htm
egu.mmmxz.cn/353753.htm
egu.mmmxz.cn/313913.htm
egu.mmmxz.cn/599193.htm
egu.mmmxz.cn/280823.htm
egu.mmmxz.cn/199573.htm
egu.mmmxz.cn/715553.htm
egu.mmmxz.cn/468423.htm
egu.mmmxz.cn/195913.htm
egu.mmmxz.cn/660003.htm
egy.mmmxz.cn/393913.htm
egy.mmmxz.cn/795313.htm
egy.mmmxz.cn/757193.htm
egy.mmmxz.cn/339153.htm
egy.mmmxz.cn/088023.htm
egy.mmmxz.cn/355393.htm
egy.mmmxz.cn/511953.htm
egy.mmmxz.cn/262843.htm
egy.mmmxz.cn/553793.htm
egy.mmmxz.cn/244023.htm
egt.mmmxz.cn/088803.htm
egt.mmmxz.cn/139933.htm
egt.mmmxz.cn/628443.htm
egt.mmmxz.cn/713993.htm
egt.mmmxz.cn/517373.htm
egt.mmmxz.cn/666083.htm
egt.mmmxz.cn/913593.htm
egt.mmmxz.cn/426663.htm
egt.mmmxz.cn/884623.htm
egt.mmmxz.cn/339193.htm
egr.mmmxz.cn/779353.htm
egr.mmmxz.cn/222423.htm
egr.mmmxz.cn/159133.htm
egr.mmmxz.cn/731173.htm
egr.mmmxz.cn/317333.htm
egr.mmmxz.cn/771573.htm
egr.mmmxz.cn/133913.htm
egr.mmmxz.cn/559173.htm
egr.mmmxz.cn/315193.htm
egr.mmmxz.cn/753153.htm
ege.mmmxz.cn/391933.htm
ege.mmmxz.cn/020203.htm
ege.mmmxz.cn/531333.htm
ege.mmmxz.cn/311793.htm
ege.mmmxz.cn/333313.htm
ege.mmmxz.cn/737593.htm
ege.mmmxz.cn/442883.htm
ege.mmmxz.cn/315573.htm
ege.mmmxz.cn/993313.htm
ege.mmmxz.cn/828043.htm
egw.mmmxz.cn/997553.htm
egw.mmmxz.cn/135193.htm
egw.mmmxz.cn/044263.htm
egw.mmmxz.cn/999173.htm
egw.mmmxz.cn/599513.htm
egw.mmmxz.cn/913533.htm
egw.mmmxz.cn/935713.htm
egw.mmmxz.cn/397153.htm
egw.mmmxz.cn/828263.htm
egw.mmmxz.cn/311713.htm
egq.mmmxz.cn/131113.htm
egq.mmmxz.cn/131113.htm
egq.mmmxz.cn/171153.htm
egq.mmmxz.cn/153173.htm
egq.mmmxz.cn/799313.htm
egq.mmmxz.cn/131113.htm
egq.mmmxz.cn/995193.htm
egq.mmmxz.cn/824043.htm
egq.mmmxz.cn/153373.htm
egq.mmmxz.cn/484203.htm
efm.mmmxz.cn/604623.htm
efm.mmmxz.cn/686423.htm
efm.mmmxz.cn/000883.htm
efm.mmmxz.cn/735933.htm
efm.mmmxz.cn/886683.htm
efm.mmmxz.cn/779353.htm
efm.mmmxz.cn/517573.htm
efm.mmmxz.cn/226463.htm
efm.mmmxz.cn/717313.htm
efm.mmmxz.cn/775713.htm
efn.mmmxz.cn/248043.htm
efn.mmmxz.cn/171193.htm
efn.mmmxz.cn/393133.htm
efn.mmmxz.cn/806083.htm
efn.mmmxz.cn/377373.htm
efn.mmmxz.cn/226283.htm
efn.mmmxz.cn/602663.htm
efn.mmmxz.cn/777533.htm
efn.mmmxz.cn/779953.htm
efn.mmmxz.cn/937193.htm
efb.mmmxz.cn/517173.htm
efb.mmmxz.cn/775513.htm
efb.mmmxz.cn/319913.htm
efb.mmmxz.cn/151333.htm
efb.mmmxz.cn/597573.htm
efb.mmmxz.cn/024443.htm
efb.mmmxz.cn/579793.htm
efb.mmmxz.cn/804063.htm
efb.mmmxz.cn/959333.htm
efb.mmmxz.cn/951353.htm
efv.mmmxz.cn/511933.htm
efv.mmmxz.cn/917913.htm
efv.mmmxz.cn/975313.htm
efv.mmmxz.cn/379953.htm
efv.mmmxz.cn/111793.htm
efv.mmmxz.cn/917393.htm
efv.mmmxz.cn/735753.htm
efv.mmmxz.cn/393373.htm
efv.mmmxz.cn/395793.htm
efv.mmmxz.cn/288663.htm
efc.mmmxz.cn/648683.htm
efc.mmmxz.cn/339793.htm
efc.mmmxz.cn/482243.htm
efc.mmmxz.cn/799113.htm
efc.mmmxz.cn/351173.htm
efc.mmmxz.cn/195353.htm
efc.mmmxz.cn/131193.htm
efc.mmmxz.cn/060863.htm
efc.mmmxz.cn/311313.htm
efc.mmmxz.cn/379113.htm
efx.mmmxz.cn/777953.htm
efx.mmmxz.cn/177793.htm
efx.mmmxz.cn/240243.htm
efx.mmmxz.cn/355113.htm
efx.mmmxz.cn/315573.htm
efx.mmmxz.cn/402803.htm
efx.mmmxz.cn/953553.htm
efx.mmmxz.cn/595193.htm
efx.mmmxz.cn/117533.htm
efx.mmmxz.cn/755393.htm
efz.mmmxz.cn/068063.htm
efz.mmmxz.cn/557953.htm
efz.mmmxz.cn/157373.htm
efz.mmmxz.cn/464003.htm
efz.mmmxz.cn/193393.htm
efz.mmmxz.cn/828423.htm
efz.mmmxz.cn/846843.htm
efz.mmmxz.cn/179513.htm
efz.mmmxz.cn/802823.htm
efz.mmmxz.cn/935933.htm
efl.mmmxz.cn/119373.htm
efl.mmmxz.cn/137193.htm
efl.mmmxz.cn/171513.htm
efl.mmmxz.cn/884063.htm
efl.mmmxz.cn/955193.htm
efl.mmmxz.cn/533933.htm
efl.mmmxz.cn/913333.htm
efl.mmmxz.cn/735513.htm
efl.mmmxz.cn/595513.htm
efl.mmmxz.cn/399113.htm
efk.mmmxz.cn/400863.htm
efk.mmmxz.cn/995393.htm
efk.mmmxz.cn/919513.htm
efk.mmmxz.cn/199593.htm
efk.mmmxz.cn/684823.htm
efk.mmmxz.cn/591773.htm
efk.mmmxz.cn/064683.htm
efk.mmmxz.cn/115793.htm
efk.mmmxz.cn/664423.htm
efk.mmmxz.cn/771373.htm
efj.mmmxz.cn/531933.htm
efj.mmmxz.cn/088823.htm
efj.mmmxz.cn/535353.htm
efj.mmmxz.cn/082663.htm
efj.mmmxz.cn/713953.htm
efj.mmmxz.cn/559553.htm
efj.mmmxz.cn/757793.htm
efj.mmmxz.cn/199713.htm
efj.mmmxz.cn/573973.htm
efj.mmmxz.cn/797393.htm
