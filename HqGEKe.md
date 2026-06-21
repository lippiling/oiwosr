晕捉宦反堆


Linux WARN_ON内核警告与__warn冷路径处理

WARN_ON是Linux内核中比BUG_ON轻量级的异常报告机制。它打印警告信息、输出调用栈，但不会杀死当前进程——系统可以继续运行。WARN_ON通常用于检测不应该发生的"不可能"条件，作为防御性编程的断言工具。

WARN_ON的核心宏定义在include/asm-generic/bug.h中：

#define WARN_ON(condition) ({                                           \
        int __ret_warn_on = !!(condition);                              \
        if (unlikely(__ret_warn_on))                                    \
                __WARN();                                               \
        unlikely(__ret_warn_on);                                        \
})

当condition为真时，__WARN被触发。__WARN调用内部的__warn函数：

#define __WARN()                __warn(file, line, func, WARN_ON_BUG, NULL, NULL)

__warn是实际的警告处理函数，位于kernel/panic.c中：

void __warn(const char *file, int line, const char *func, unsigned long bugflags,
            void *caller, void **irqtrace)
{
        disable_trace_on_warning();  /* 防止递归警告导致死循环 */

        if (file)
                pr_warn("WARNING: CPU: %d PID: %d at %s:%d %s()\n",
                        raw_smp_processor_id(), current->pid, file, line, func);
        else
                pr_warn("WARNING: CPU: %d PID: %d at %p\n",
                        raw_smp_processor_id(), current->pid, caller);

        if (irqtrace)
                print_irqtrace_events(current);

        if (panic_on_warn) {
                /* 如果内核配置了panic_on_warn，则升级为panic */
                panic("panic_on_warn set ...\n");
        }

        /* 输出当前进程的调用栈 */
        dump_stack();

        print_modules();

        /* 某些架构可能在此处进入oops流程 */
        if (bugflags & BUGFLAG_WARNING) {
                /* 使用oops_exit来记录oops计数 */
                oops_exit();
        }
}

__warn函数的冷路径特性体现在编译器的优化上。内核使用__cold属性标记warn相关的函数，告知编译器将这些函数放置在.text.unlikely节区，不参与常规的内联和热路径优化：

static __cold __no_instrument_function void __warn(...)

这意味着WARN_ON检查虽然写在内联代码中，但当condition为假时（正常路径），执行流完全不会触碰到__warn的代码。这对于指令缓存（I-Cache）的效率至关重要。

内核还提供了WARN_ON_ONCE变体，使用static_key或简单的原子变量来确保每个WARN_ON_ONCE位置只触发一次警告：

#define WARN_ON_ONCE(condition) ({                                      \
        static bool __section(".data.once") __warned;                   \
        int __ret_warn_once = !!(condition);                            \
                                                                        \
        if (unlikely(__ret_warn_once && !__warned)) {                   \
                __warned = true;                                        \
                __WARN();                                               \
        }                                                               \
        unlikely(__ret_warn_once);                                      \
})

__warned变量使用__section(".data.once")放入独立的节区，该节区在内核启动时被清零，运行时写入一个字节。由于该变量是静态的、按WARN_ON_ONCE调用位置实例化的，每个警告位置有独立的跟踪状态。

WARN_ON系列还包括WARN(condition, fmt, ...)、WARN_RATELIMIT(condition, state)等变体。WARN_RATELIMIT在频繁触发的场景下避免日志泛洪：

#define WARN_RATELIMIT(condition, state)                                \
        ({                                                              \
                int __ret_warn_on = !!(condition);                      \
                int __ratelimit = __ratelimit(state);                   \
                if (unlikely(__ret_warn_on && __ratelimit))             \
                        __WARN();                                       \
                unlikely(__ret_warn_on);                                \
        })

WARN_ON与lockdep工具的交互值得一提。当在持有锁的上下文中触发WARN_ON时，lockdep会记录这个事件并可能产生额外的报告。但__warn中的disable_trace_on_warning()会暂时关闭lockdep追踪，防止在警告输出过程中产生递归锁问题：

void disable_trace_on_warning(void)
{
        tracing_off();
}

对于生产系统，WARN_ON的使用需要权衡。每个WARN_ON在condition为真时会调用printk和dump_stack，这是在原子上下文中可能触发deadlock的风险点。因此，内核社区强调：在中断上下文或持有关键锁时，优先使用WARN_ON_ONCE而非WARN_ON。

痈床裁僦炯僦刃躺侥卮欠帘崖裁口

m.ekd.hxbsg.cn/42620.Doc
m.ekd.hxbsg.cn/00024.Doc
m.ekd.hxbsg.cn/91799.Doc
m.ekd.hxbsg.cn/79771.Doc
m.ekd.hxbsg.cn/22600.Doc
m.ekd.hxbsg.cn/77991.Doc
m.ekd.hxbsg.cn/59777.Doc
m.ekd.hxbsg.cn/77193.Doc
m.ekd.hxbsg.cn/13935.Doc
m.ekd.hxbsg.cn/71159.Doc
m.ekd.hxbsg.cn/22842.Doc
m.ekd.hxbsg.cn/51595.Doc
m.ekd.hxbsg.cn/60266.Doc
m.ekd.hxbsg.cn/17715.Doc
m.ekd.hxbsg.cn/93571.Doc
m.eks.hxbsg.cn/17319.Doc
m.eks.hxbsg.cn/33539.Doc
m.eks.hxbsg.cn/93371.Doc
m.eks.hxbsg.cn/19997.Doc
m.eks.hxbsg.cn/22688.Doc
m.eks.hxbsg.cn/57597.Doc
m.eks.hxbsg.cn/15337.Doc
m.eks.hxbsg.cn/75595.Doc
m.eks.hxbsg.cn/77715.Doc
m.eks.hxbsg.cn/51139.Doc
m.eks.hxbsg.cn/37775.Doc
m.eks.hxbsg.cn/93355.Doc
m.eks.hxbsg.cn/75753.Doc
m.eks.hxbsg.cn/53759.Doc
m.eks.hxbsg.cn/64446.Doc
m.eks.hxbsg.cn/60402.Doc
m.eks.hxbsg.cn/02426.Doc
m.eks.hxbsg.cn/79757.Doc
m.eks.hxbsg.cn/06226.Doc
m.eks.hxbsg.cn/35513.Doc
m.eka.hxbsg.cn/20228.Doc
m.eka.hxbsg.cn/66206.Doc
m.eka.hxbsg.cn/22206.Doc
m.eka.hxbsg.cn/60406.Doc
m.eka.hxbsg.cn/5.Doc
m.eka.hxbsg.cn/91179.Doc
m.eka.hxbsg.cn/80406.Doc
m.eka.hxbsg.cn/71153.Doc
m.eka.hxbsg.cn/35957.Doc
m.eka.hxbsg.cn/24806.Doc
m.eka.hxbsg.cn/20282.Doc
m.eka.hxbsg.cn/42404.Doc
m.eka.hxbsg.cn/17339.Doc
m.eka.hxbsg.cn/60600.Doc
m.eka.hxbsg.cn/79979.Doc
m.eka.hxbsg.cn/17393.Doc
m.eka.hxbsg.cn/20444.Doc
m.eka.hxbsg.cn/42000.Doc
m.eka.hxbsg.cn/46826.Doc
m.eka.hxbsg.cn/77575.Doc
m.ekp.hxbsg.cn/35191.Doc
m.ekp.hxbsg.cn/82802.Doc
m.ekp.hxbsg.cn/80264.Doc
m.ekp.hxbsg.cn/99597.Doc
m.ekp.hxbsg.cn/26462.Doc
m.ekp.hxbsg.cn/04800.Doc
m.ekp.hxbsg.cn/42802.Doc
m.ekp.hxbsg.cn/37973.Doc
m.ekp.hxbsg.cn/82608.Doc
m.ekp.hxbsg.cn/93799.Doc
m.ekp.hxbsg.cn/80608.Doc
m.ekp.hxbsg.cn/17537.Doc
m.ekp.hxbsg.cn/40002.Doc
m.ekp.hxbsg.cn/73999.Doc
m.ekp.hxbsg.cn/95911.Doc
m.ekp.hxbsg.cn/88404.Doc
m.ekp.hxbsg.cn/73591.Doc
m.ekp.hxbsg.cn/55977.Doc
m.ekp.hxbsg.cn/02462.Doc
m.ekp.hxbsg.cn/26080.Doc
m.eko.hxbsg.cn/62086.Doc
m.eko.hxbsg.cn/35517.Doc
m.eko.hxbsg.cn/37595.Doc
m.eko.hxbsg.cn/51331.Doc
m.eko.hxbsg.cn/17935.Doc
m.eko.hxbsg.cn/19731.Doc
m.eko.hxbsg.cn/35795.Doc
m.eko.hxbsg.cn/77339.Doc
m.eko.hxbsg.cn/46086.Doc
m.eko.hxbsg.cn/97793.Doc
m.eko.hxbsg.cn/02226.Doc
m.eko.hxbsg.cn/42042.Doc
m.eko.hxbsg.cn/73133.Doc
m.eko.hxbsg.cn/59791.Doc
m.eko.hxbsg.cn/35117.Doc
m.eko.hxbsg.cn/04206.Doc
m.eko.hxbsg.cn/02626.Doc
m.eko.hxbsg.cn/00604.Doc
m.eko.hxbsg.cn/04820.Doc
m.eko.hxbsg.cn/73771.Doc
m.eki.hxbsg.cn/99319.Doc
m.eki.hxbsg.cn/57195.Doc
m.eki.hxbsg.cn/33179.Doc
m.eki.hxbsg.cn/88848.Doc
m.eki.hxbsg.cn/57955.Doc
m.eki.hxbsg.cn/39713.Doc
m.eki.hxbsg.cn/93135.Doc
m.eki.hxbsg.cn/35173.Doc
m.eki.hxbsg.cn/17399.Doc
m.eki.hxbsg.cn/99353.Doc
m.eki.hxbsg.cn/86628.Doc
m.eki.hxbsg.cn/84462.Doc
m.eki.hxbsg.cn/84424.Doc
m.eki.hxbsg.cn/82400.Doc
m.eki.hxbsg.cn/13773.Doc
m.eki.hxbsg.cn/19757.Doc
m.eki.hxbsg.cn/44068.Doc
m.eki.hxbsg.cn/39995.Doc
m.eki.hxbsg.cn/86608.Doc
m.eki.hxbsg.cn/39911.Doc
m.eku.hxbsg.cn/48022.Doc
m.eku.hxbsg.cn/08802.Doc
m.eku.hxbsg.cn/77355.Doc
m.eku.hxbsg.cn/11359.Doc
m.eku.hxbsg.cn/13597.Doc
m.eku.hxbsg.cn/60482.Doc
m.eku.hxbsg.cn/60424.Doc
m.eku.hxbsg.cn/82400.Doc
m.eku.hxbsg.cn/62668.Doc
m.eku.hxbsg.cn/37175.Doc
m.eku.hxbsg.cn/57397.Doc
m.eku.hxbsg.cn/59535.Doc
m.eku.hxbsg.cn/00446.Doc
m.eku.hxbsg.cn/24862.Doc
m.eku.hxbsg.cn/46464.Doc
m.eku.hxbsg.cn/55977.Doc
m.eku.hxbsg.cn/11171.Doc
m.eku.hxbsg.cn/11599.Doc
m.eku.hxbsg.cn/17755.Doc
m.eku.hxbsg.cn/57539.Doc
m.eky.hxbsg.cn/57791.Doc
m.eky.hxbsg.cn/02644.Doc
m.eky.hxbsg.cn/68868.Doc
m.eky.hxbsg.cn/37379.Doc
m.eky.hxbsg.cn/22020.Doc
m.eky.hxbsg.cn/00000.Doc
m.eky.hxbsg.cn/26462.Doc
m.eky.hxbsg.cn/51937.Doc
m.eky.hxbsg.cn/24622.Doc
m.eky.hxbsg.cn/02264.Doc
m.eky.hxbsg.cn/66802.Doc
m.eky.hxbsg.cn/31911.Doc
m.eky.hxbsg.cn/82824.Doc
m.eky.hxbsg.cn/08646.Doc
m.eky.hxbsg.cn/42086.Doc
m.eky.hxbsg.cn/19759.Doc
m.eky.hxbsg.cn/08044.Doc
m.eky.hxbsg.cn/88848.Doc
m.eky.hxbsg.cn/82202.Doc
m.eky.hxbsg.cn/57513.Doc
m.ekt.hxbsg.cn/73717.Doc
m.ekt.hxbsg.cn/06282.Doc
m.ekt.hxbsg.cn/24604.Doc
m.ekt.hxbsg.cn/20860.Doc
m.ekt.hxbsg.cn/06408.Doc
m.ekt.hxbsg.cn/84482.Doc
m.ekt.hxbsg.cn/71953.Doc
m.ekt.hxbsg.cn/44886.Doc
m.ekt.hxbsg.cn/55197.Doc
m.ekt.hxbsg.cn/86420.Doc
m.ekt.hxbsg.cn/99171.Doc
m.ekt.hxbsg.cn/95113.Doc
m.ekt.hxbsg.cn/02866.Doc
m.ekt.hxbsg.cn/39917.Doc
m.ekt.hxbsg.cn/19357.Doc
m.ekt.hxbsg.cn/51559.Doc
m.ekt.hxbsg.cn/62604.Doc
m.ekt.hxbsg.cn/77135.Doc
m.ekt.hxbsg.cn/77737.Doc
m.ekt.hxbsg.cn/88084.Doc
m.ekr.hxbsg.cn/20260.Doc
m.ekr.hxbsg.cn/91995.Doc
m.ekr.hxbsg.cn/28002.Doc
m.ekr.hxbsg.cn/35579.Doc
m.ekr.hxbsg.cn/15193.Doc
m.ekr.hxbsg.cn/48484.Doc
m.ekr.hxbsg.cn/82086.Doc
m.ekr.hxbsg.cn/42420.Doc
m.ekr.hxbsg.cn/95997.Doc
m.ekr.hxbsg.cn/00022.Doc
m.ekr.hxbsg.cn/64824.Doc
m.ekr.hxbsg.cn/60426.Doc
m.ekr.hxbsg.cn/06060.Doc
m.ekr.hxbsg.cn/22006.Doc
m.ekr.hxbsg.cn/80802.Doc
m.ekr.hxbsg.cn/82082.Doc
m.ekr.hxbsg.cn/55119.Doc
m.ekr.hxbsg.cn/35153.Doc
m.ekr.hxbsg.cn/06626.Doc
m.ekr.hxbsg.cn/82248.Doc
m.eke.hxbsg.cn/26268.Doc
m.eke.hxbsg.cn/22404.Doc
m.eke.hxbsg.cn/73353.Doc
m.eke.hxbsg.cn/37711.Doc
m.eke.hxbsg.cn/60404.Doc
m.eke.hxbsg.cn/97155.Doc
m.eke.hxbsg.cn/13397.Doc
m.eke.hxbsg.cn/39579.Doc
m.eke.hxbsg.cn/24024.Doc
m.eke.hxbsg.cn/62004.Doc
m.eke.hxbsg.cn/77591.Doc
m.eke.hxbsg.cn/57599.Doc
m.eke.hxbsg.cn/04064.Doc
m.eke.hxbsg.cn/55357.Doc
m.eke.hxbsg.cn/68806.Doc
m.eke.hxbsg.cn/24808.Doc
m.eke.hxbsg.cn/37171.Doc
m.eke.hxbsg.cn/93979.Doc
m.eke.hxbsg.cn/71773.Doc
m.eke.hxbsg.cn/31193.Doc
m.ekw.hxbsg.cn/24288.Doc
m.ekw.hxbsg.cn/77955.Doc
m.ekw.hxbsg.cn/11959.Doc
m.ekw.hxbsg.cn/37175.Doc
m.ekw.hxbsg.cn/28600.Doc
m.ekw.hxbsg.cn/24664.Doc
m.ekw.hxbsg.cn/82842.Doc
m.ekw.hxbsg.cn/59391.Doc
m.ekw.hxbsg.cn/51753.Doc
m.ekw.hxbsg.cn/13535.Doc
m.ekw.hxbsg.cn/97111.Doc
m.ekw.hxbsg.cn/15375.Doc
m.ekw.hxbsg.cn/82482.Doc
m.ekw.hxbsg.cn/59935.Doc
m.ekw.hxbsg.cn/79373.Doc
m.ekw.hxbsg.cn/35357.Doc
m.ekw.hxbsg.cn/20006.Doc
m.ekw.hxbsg.cn/06604.Doc
m.ekw.hxbsg.cn/68884.Doc
m.ekw.hxbsg.cn/08268.Doc
m.ekq.hxbsg.cn/22042.Doc
m.ekq.hxbsg.cn/13755.Doc
m.ekq.hxbsg.cn/79713.Doc
m.ekq.hxbsg.cn/51393.Doc
m.ekq.hxbsg.cn/19915.Doc
m.ekq.hxbsg.cn/31573.Doc
m.ekq.hxbsg.cn/79791.Doc
m.ekq.hxbsg.cn/88044.Doc
m.ekq.hxbsg.cn/62204.Doc
m.ekq.hxbsg.cn/46684.Doc
m.ekq.hxbsg.cn/88024.Doc
m.ekq.hxbsg.cn/11771.Doc
m.ekq.hxbsg.cn/06686.Doc
m.ekq.hxbsg.cn/42488.Doc
m.ekq.hxbsg.cn/00406.Doc
m.ekq.hxbsg.cn/55735.Doc
m.ekq.hxbsg.cn/15199.Doc
m.ekq.hxbsg.cn/16153.Doc
m.ekq.hxbsg.cn/95553.Doc
m.ekq.hxbsg.cn/73113.Doc
m.ejm.hxbsg.cn/84408.Doc
m.ejm.hxbsg.cn/75553.Doc
m.ejm.hxbsg.cn/57179.Doc
m.ejm.hxbsg.cn/77311.Doc
m.ejm.hxbsg.cn/51711.Doc
m.ejm.hxbsg.cn/97513.Doc
m.ejm.hxbsg.cn/97951.Doc
m.ejm.hxbsg.cn/55119.Doc
m.ejm.hxbsg.cn/51771.Doc
m.ejm.hxbsg.cn/22860.Doc
m.ejm.hxbsg.cn/84646.Doc
m.ejm.hxbsg.cn/08464.Doc
m.ejm.hxbsg.cn/91595.Doc
m.ejm.hxbsg.cn/66208.Doc
m.ejm.hxbsg.cn/64060.Doc
m.ejm.hxbsg.cn/57937.Doc
m.ejm.hxbsg.cn/82888.Doc
m.ejm.hxbsg.cn/40420.Doc
m.ejm.hxbsg.cn/86664.Doc
m.ejm.hxbsg.cn/60408.Doc
m.ejn.hxbsg.cn/59933.Doc
m.ejn.hxbsg.cn/15739.Doc
m.ejn.hxbsg.cn/79173.Doc
m.ejn.hxbsg.cn/59959.Doc
m.ejn.hxbsg.cn/77117.Doc
m.ejn.hxbsg.cn/55759.Doc
m.ejn.hxbsg.cn/46860.Doc
m.ejn.hxbsg.cn/04046.Doc
m.ejn.hxbsg.cn/06648.Doc
m.ejn.hxbsg.cn/02866.Doc
m.ejn.hxbsg.cn/26000.Doc
m.ejn.hxbsg.cn/04642.Doc
m.ejn.hxbsg.cn/86226.Doc
m.ejn.hxbsg.cn/26444.Doc
m.ejn.hxbsg.cn/35919.Doc
m.ejn.hxbsg.cn/48064.Doc
m.ejn.hxbsg.cn/91799.Doc
m.ejn.hxbsg.cn/99937.Doc
m.ejn.hxbsg.cn/15331.Doc
m.ejn.hxbsg.cn/62048.Doc
m.ejb.hxbsg.cn/39939.Doc
m.ejb.hxbsg.cn/00084.Doc
m.ejb.hxbsg.cn/66000.Doc
m.ejb.hxbsg.cn/99599.Doc
m.ejb.hxbsg.cn/88628.Doc
m.ejb.hxbsg.cn/42604.Doc
m.ejb.hxbsg.cn/64400.Doc
m.ejb.hxbsg.cn/17623.Doc
m.ejb.hxbsg.cn/66628.Doc
m.ejb.hxbsg.cn/93591.Doc
m.ejb.hxbsg.cn/71135.Doc
m.ejb.hxbsg.cn/66242.Doc
m.ejb.hxbsg.cn/22080.Doc
m.ejb.hxbsg.cn/77375.Doc
m.ejb.hxbsg.cn/71395.Doc
m.ejb.hxbsg.cn/13919.Doc
m.ejb.hxbsg.cn/17791.Doc
m.ejb.hxbsg.cn/35959.Doc
m.ejb.hxbsg.cn/86882.Doc
m.ejb.hxbsg.cn/53533.Doc
m.ejv.hxbsg.cn/53537.Doc
m.ejv.hxbsg.cn/86084.Doc
m.ejv.hxbsg.cn/40486.Doc
m.ejv.hxbsg.cn/62462.Doc
m.ejv.hxbsg.cn/35131.Doc
m.ejv.hxbsg.cn/93713.Doc
m.ejv.hxbsg.cn/19719.Doc
m.ejv.hxbsg.cn/91151.Doc
m.ejv.hxbsg.cn/55717.Doc
m.ejv.hxbsg.cn/79191.Doc
m.ejv.hxbsg.cn/59517.Doc
m.ejv.hxbsg.cn/37377.Doc
m.ejv.hxbsg.cn/15519.Doc
m.ejv.hxbsg.cn/31359.Doc
m.ejv.hxbsg.cn/28880.Doc
m.ejv.hxbsg.cn/97395.Doc
m.ejv.hxbsg.cn/11975.Doc
m.ejv.hxbsg.cn/95913.Doc
m.ejv.hxbsg.cn/91717.Doc
m.ejv.hxbsg.cn/64264.Doc
m.ejc.hxbsg.cn/99935.Doc
m.ejc.hxbsg.cn/57913.Doc
m.ejc.hxbsg.cn/11317.Doc
m.ejc.hxbsg.cn/57193.Doc
m.ejc.hxbsg.cn/33111.Doc
m.ejc.hxbsg.cn/59579.Doc
m.ejc.hxbsg.cn/66848.Doc
m.ejc.hxbsg.cn/73939.Doc
m.ejc.hxbsg.cn/95513.Doc
m.ejc.hxbsg.cn/57915.Doc
m.ejc.hxbsg.cn/31737.Doc
m.ejc.hxbsg.cn/59517.Doc
m.ejc.hxbsg.cn/46682.Doc
m.ejc.hxbsg.cn/48620.Doc
m.ejc.hxbsg.cn/17399.Doc
m.ejc.hxbsg.cn/77993.Doc
m.ejc.hxbsg.cn/93115.Doc
m.ejc.hxbsg.cn/77933.Doc
m.ejc.hxbsg.cn/73551.Doc
m.ejc.hxbsg.cn/04000.Doc
m.ejx.hxbsg.cn/73593.Doc
m.ejx.hxbsg.cn/42426.Doc
m.ejx.hxbsg.cn/99759.Doc
m.ejx.hxbsg.cn/06462.Doc
m.ejx.hxbsg.cn/06266.Doc
m.ejx.hxbsg.cn/66448.Doc
m.ejx.hxbsg.cn/95173.Doc
m.ejx.hxbsg.cn/88400.Doc
m.ejx.hxbsg.cn/11395.Doc
m.ejx.hxbsg.cn/97115.Doc
m.ejx.hxbsg.cn/57513.Doc
m.ejx.hxbsg.cn/75115.Doc
m.ejx.hxbsg.cn/33537.Doc
m.ejx.hxbsg.cn/33175.Doc
m.ejx.hxbsg.cn/68028.Doc
m.ejx.hxbsg.cn/91933.Doc
m.ejx.hxbsg.cn/64824.Doc
m.ejx.hxbsg.cn/99939.Doc
m.ejx.hxbsg.cn/68048.Doc
m.ejx.hxbsg.cn/35153.Doc
m.ejz.hxbsg.cn/00600.Doc
m.ejz.hxbsg.cn/22262.Doc
m.ejz.hxbsg.cn/37191.Doc
m.ejz.hxbsg.cn/24482.Doc
m.ejz.hxbsg.cn/79195.Doc
m.ejz.hxbsg.cn/04064.Doc
m.ejz.hxbsg.cn/68666.Doc
m.ejz.hxbsg.cn/88462.Doc
m.ejz.hxbsg.cn/53753.Doc
m.ejz.hxbsg.cn/17737.Doc
m.ejz.hxbsg.cn/37135.Doc
m.ejz.hxbsg.cn/71739.Doc
m.ejz.hxbsg.cn/79779.Doc
m.ejz.hxbsg.cn/55539.Doc
m.ejz.hxbsg.cn/04048.Doc
m.ejz.hxbsg.cn/42880.Doc
m.ejz.hxbsg.cn/11771.Doc
m.ejz.hxbsg.cn/77551.Doc
m.ejz.hxbsg.cn/57731.Doc
m.ejz.hxbsg.cn/97579.Doc
