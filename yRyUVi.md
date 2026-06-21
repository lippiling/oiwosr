盘褐督吮颇


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

衬园乜鲁赣恢嚷栈芬乃乃胶澜敌匆

m.wbz.qtbzn.cn/51995.Doc
m.wbz.qtbzn.cn/22442.Doc
m.wbz.qtbzn.cn/57931.Doc
m.wbz.qtbzn.cn/57311.Doc
m.wbz.qtbzn.cn/48620.Doc
m.wbl.qtbzn.cn/51979.Doc
m.wbl.qtbzn.cn/51793.Doc
m.wbl.qtbzn.cn/33731.Doc
m.wbl.qtbzn.cn/9.Doc
m.wbl.qtbzn.cn/46480.Doc
m.wbl.qtbzn.cn/19351.Doc
m.wbl.qtbzn.cn/95533.Doc
m.wbl.qtbzn.cn/22646.Doc
m.wbl.qtbzn.cn/51773.Doc
m.wbl.qtbzn.cn/80848.Doc
m.wbl.qtbzn.cn/40826.Doc
m.wbl.qtbzn.cn/55337.Doc
m.wbl.qtbzn.cn/93997.Doc
m.wbl.qtbzn.cn/80008.Doc
m.wbl.qtbzn.cn/46446.Doc
m.wbl.qtbzn.cn/46040.Doc
m.wbl.qtbzn.cn/48086.Doc
m.wbl.qtbzn.cn/66060.Doc
m.wbl.qtbzn.cn/11955.Doc
m.wbl.qtbzn.cn/42260.Doc
m.wbk.qtbzn.cn/51155.Doc
m.wbk.qtbzn.cn/57379.Doc
m.wbk.qtbzn.cn/57579.Doc
m.wbk.qtbzn.cn/68086.Doc
m.wbk.qtbzn.cn/84008.Doc
m.wbk.qtbzn.cn/75173.Doc
m.wbk.qtbzn.cn/99739.Doc
m.wbk.qtbzn.cn/88024.Doc
m.wbk.qtbzn.cn/57753.Doc
m.wbk.qtbzn.cn/26086.Doc
m.wbk.qtbzn.cn/51715.Doc
m.wbk.qtbzn.cn/06204.Doc
m.wbk.qtbzn.cn/68048.Doc
m.wbk.qtbzn.cn/44402.Doc
m.wbk.qtbzn.cn/59775.Doc
m.wbk.qtbzn.cn/62622.Doc
m.wbk.qtbzn.cn/15515.Doc
m.wbk.qtbzn.cn/91195.Doc
m.wbk.qtbzn.cn/88242.Doc
m.wbk.qtbzn.cn/42268.Doc
m.wbj.qtbzn.cn/95779.Doc
m.wbj.qtbzn.cn/93119.Doc
m.wbj.qtbzn.cn/75337.Doc
m.wbj.qtbzn.cn/91577.Doc
m.wbj.qtbzn.cn/97131.Doc
m.wbj.qtbzn.cn/24448.Doc
m.wbj.qtbzn.cn/13537.Doc
m.wbj.qtbzn.cn/39119.Doc
m.wbj.qtbzn.cn/53979.Doc
m.wbj.qtbzn.cn/75113.Doc
m.wbj.qtbzn.cn/59713.Doc
m.wbj.qtbzn.cn/68204.Doc
m.wbj.qtbzn.cn/73995.Doc
m.wbj.qtbzn.cn/75937.Doc
m.wbj.qtbzn.cn/08420.Doc
m.wbj.qtbzn.cn/84286.Doc
m.wbj.qtbzn.cn/39353.Doc
m.wbj.qtbzn.cn/26842.Doc
m.wbj.qtbzn.cn/86264.Doc
m.wbj.qtbzn.cn/82260.Doc
m.wbh.qtbzn.cn/04208.Doc
m.wbh.qtbzn.cn/77379.Doc
m.wbh.qtbzn.cn/71139.Doc
m.wbh.qtbzn.cn/28268.Doc
m.wbh.qtbzn.cn/31593.Doc
m.wbh.qtbzn.cn/26860.Doc
m.wbh.qtbzn.cn/53173.Doc
m.wbh.qtbzn.cn/28086.Doc
m.wbh.qtbzn.cn/53735.Doc
m.wbh.qtbzn.cn/42006.Doc
m.wbh.qtbzn.cn/19715.Doc
m.wbh.qtbzn.cn/79337.Doc
m.wbh.qtbzn.cn/53559.Doc
m.wbh.qtbzn.cn/33351.Doc
m.wbh.qtbzn.cn/39395.Doc
m.wbh.qtbzn.cn/46682.Doc
m.wbh.qtbzn.cn/71957.Doc
m.wbh.qtbzn.cn/75717.Doc
m.wbh.qtbzn.cn/95135.Doc
m.wbh.qtbzn.cn/71997.Doc
m.wbg.qtbzn.cn/06424.Doc
m.wbg.qtbzn.cn/73335.Doc
m.wbg.qtbzn.cn/40246.Doc
m.wbg.qtbzn.cn/35533.Doc
m.wbg.qtbzn.cn/04444.Doc
m.wbg.qtbzn.cn/26606.Doc
m.wbg.qtbzn.cn/37979.Doc
m.wbg.qtbzn.cn/95117.Doc
m.wbg.qtbzn.cn/15951.Doc
m.wbg.qtbzn.cn/20664.Doc
m.wbg.qtbzn.cn/11717.Doc
m.wbg.qtbzn.cn/60008.Doc
m.wbg.qtbzn.cn/40822.Doc
m.wbg.qtbzn.cn/26204.Doc
m.wbg.qtbzn.cn/53519.Doc
m.wbg.qtbzn.cn/59371.Doc
m.wbg.qtbzn.cn/88028.Doc
m.wbg.qtbzn.cn/26208.Doc
m.wbg.qtbzn.cn/80860.Doc
m.wbg.qtbzn.cn/86682.Doc
m.wbf.qtbzn.cn/17159.Doc
m.wbf.qtbzn.cn/73779.Doc
m.wbf.qtbzn.cn/66022.Doc
m.wbf.qtbzn.cn/40248.Doc
m.wbf.qtbzn.cn/24826.Doc
m.wbf.qtbzn.cn/79113.Doc
m.wbf.qtbzn.cn/55913.Doc
m.wbf.qtbzn.cn/71115.Doc
m.wbf.qtbzn.cn/99331.Doc
m.wbf.qtbzn.cn/00624.Doc
m.wbf.qtbzn.cn/93173.Doc
m.wbf.qtbzn.cn/15339.Doc
m.wbf.qtbzn.cn/75577.Doc
m.wbf.qtbzn.cn/53579.Doc
m.wbf.qtbzn.cn/88202.Doc
m.wbf.qtbzn.cn/20842.Doc
m.wbf.qtbzn.cn/40420.Doc
m.wbf.qtbzn.cn/13177.Doc
m.wbf.qtbzn.cn/46488.Doc
m.wbf.qtbzn.cn/73155.Doc
m.wbd.qtbzn.cn/61193.Doc
m.wbd.qtbzn.cn/28884.Doc
m.wbd.qtbzn.cn/80662.Doc
m.wbd.qtbzn.cn/73777.Doc
m.wbd.qtbzn.cn/71379.Doc
m.wbd.qtbzn.cn/59555.Doc
m.wbd.qtbzn.cn/80646.Doc
m.wbd.qtbzn.cn/46888.Doc
m.wbd.qtbzn.cn/62066.Doc
m.wbd.qtbzn.cn/28820.Doc
m.wbd.qtbzn.cn/86880.Doc
m.wbd.qtbzn.cn/53797.Doc
m.wbd.qtbzn.cn/75993.Doc
m.wbd.qtbzn.cn/51555.Doc
m.wbd.qtbzn.cn/22646.Doc
m.wbd.qtbzn.cn/62624.Doc
m.wbd.qtbzn.cn/51999.Doc
m.wbd.qtbzn.cn/13319.Doc
m.wbd.qtbzn.cn/42264.Doc
m.wbd.qtbzn.cn/82046.Doc
m.wbs.qtbzn.cn/33735.Doc
m.wbs.qtbzn.cn/37131.Doc
m.wbs.qtbzn.cn/68804.Doc
m.wbs.qtbzn.cn/33553.Doc
m.wbs.qtbzn.cn/11339.Doc
m.wbs.qtbzn.cn/31597.Doc
m.wbs.qtbzn.cn/71131.Doc
m.wbs.qtbzn.cn/24664.Doc
m.wbs.qtbzn.cn/59311.Doc
m.wbs.qtbzn.cn/95313.Doc
m.wbs.qtbzn.cn/77393.Doc
m.wbs.qtbzn.cn/04068.Doc
m.wbs.qtbzn.cn/64200.Doc
m.wbs.qtbzn.cn/46082.Doc
m.wbs.qtbzn.cn/59791.Doc
m.wbs.qtbzn.cn/55373.Doc
m.wbs.qtbzn.cn/59373.Doc
m.wbs.qtbzn.cn/35579.Doc
m.wbs.qtbzn.cn/02862.Doc
m.wbs.qtbzn.cn/80448.Doc
m.wba.qtbzn.cn/75351.Doc
m.wba.qtbzn.cn/79575.Doc
m.wba.qtbzn.cn/97971.Doc
m.wba.qtbzn.cn/55931.Doc
m.wba.qtbzn.cn/51195.Doc
m.wba.qtbzn.cn/55191.Doc
m.wba.qtbzn.cn/04484.Doc
m.wba.qtbzn.cn/02082.Doc
m.wba.qtbzn.cn/80042.Doc
m.wba.qtbzn.cn/57359.Doc
m.wba.qtbzn.cn/95317.Doc
m.wba.qtbzn.cn/06044.Doc
m.wba.qtbzn.cn/86868.Doc
m.wba.qtbzn.cn/91113.Doc
m.wba.qtbzn.cn/86226.Doc
m.wba.qtbzn.cn/15799.Doc
m.wba.qtbzn.cn/57757.Doc
m.wba.qtbzn.cn/26882.Doc
m.wba.qtbzn.cn/99133.Doc
m.wba.qtbzn.cn/40222.Doc
m.wbp.qtbzn.cn/17737.Doc
m.wbp.qtbzn.cn/99519.Doc
m.wbp.qtbzn.cn/35379.Doc
m.wbp.qtbzn.cn/28064.Doc
m.wbp.qtbzn.cn/55397.Doc
m.wbp.qtbzn.cn/93177.Doc
m.wbp.qtbzn.cn/60264.Doc
m.wbp.qtbzn.cn/84060.Doc
m.wbp.qtbzn.cn/48462.Doc
m.wbp.qtbzn.cn/77373.Doc
m.wbp.qtbzn.cn/57173.Doc
m.wbp.qtbzn.cn/46462.Doc
m.wbp.qtbzn.cn/84602.Doc
m.wbp.qtbzn.cn/13339.Doc
m.wbp.qtbzn.cn/19571.Doc
m.wbp.qtbzn.cn/11917.Doc
m.wbp.qtbzn.cn/84826.Doc
m.wbp.qtbzn.cn/00064.Doc
m.wbp.qtbzn.cn/57335.Doc
m.wbp.qtbzn.cn/31935.Doc
m.wbo.qtbzn.cn/55797.Doc
m.wbo.qtbzn.cn/39797.Doc
m.wbo.qtbzn.cn/44240.Doc
m.wbo.qtbzn.cn/04060.Doc
m.wbo.qtbzn.cn/75771.Doc
m.wbo.qtbzn.cn/79377.Doc
m.wbo.qtbzn.cn/86048.Doc
m.wbo.qtbzn.cn/19135.Doc
m.wbo.qtbzn.cn/33517.Doc
m.wbo.qtbzn.cn/95351.Doc
m.wbo.qtbzn.cn/28202.Doc
m.wbo.qtbzn.cn/04284.Doc
m.wbo.qtbzn.cn/93397.Doc
m.wbo.qtbzn.cn/86044.Doc
m.wbo.qtbzn.cn/73115.Doc
m.wbo.qtbzn.cn/08646.Doc
m.wbo.qtbzn.cn/64864.Doc
m.wbo.qtbzn.cn/95731.Doc
m.wbo.qtbzn.cn/33971.Doc
m.wbo.qtbzn.cn/64822.Doc
m.wbi.qtbzn.cn/15357.Doc
m.wbi.qtbzn.cn/71331.Doc
m.wbi.qtbzn.cn/11531.Doc
m.wbi.qtbzn.cn/55395.Doc
m.wbi.qtbzn.cn/77111.Doc
m.wbi.qtbzn.cn/97931.Doc
m.wbi.qtbzn.cn/39175.Doc
m.wbi.qtbzn.cn/73155.Doc
m.wbi.qtbzn.cn/53157.Doc
m.wbi.qtbzn.cn/64066.Doc
m.wbi.qtbzn.cn/22664.Doc
m.wbi.qtbzn.cn/13151.Doc
m.wbi.qtbzn.cn/33339.Doc
m.wbi.qtbzn.cn/73177.Doc
m.wbi.qtbzn.cn/28402.Doc
m.wbi.qtbzn.cn/06404.Doc
m.wbi.qtbzn.cn/46262.Doc
m.wbi.qtbzn.cn/04084.Doc
m.wbi.qtbzn.cn/80200.Doc
m.wbi.qtbzn.cn/24246.Doc
m.wbu.qtbzn.cn/75975.Doc
m.wbu.qtbzn.cn/64406.Doc
m.wbu.qtbzn.cn/24668.Doc
m.wbu.qtbzn.cn/66688.Doc
m.wbu.qtbzn.cn/73311.Doc
m.wbu.qtbzn.cn/71957.Doc
m.wbu.qtbzn.cn/91515.Doc
m.wbu.qtbzn.cn/95177.Doc
m.wbu.qtbzn.cn/62460.Doc
m.wbu.qtbzn.cn/24620.Doc
m.wbu.qtbzn.cn/13595.Doc
m.wbu.qtbzn.cn/15931.Doc
m.wbu.qtbzn.cn/04844.Doc
m.wbu.qtbzn.cn/51157.Doc
m.wbu.qtbzn.cn/00664.Doc
m.wbu.qtbzn.cn/31593.Doc
m.wbu.qtbzn.cn/22644.Doc
m.wbu.qtbzn.cn/71159.Doc
m.wbu.qtbzn.cn/84806.Doc
m.wbu.qtbzn.cn/19913.Doc
m.wby.qtbzn.cn/39795.Doc
m.wby.qtbzn.cn/97313.Doc
m.wby.qtbzn.cn/19193.Doc
m.wby.qtbzn.cn/31771.Doc
m.wby.qtbzn.cn/15371.Doc
m.wby.qtbzn.cn/62820.Doc
m.wby.qtbzn.cn/39999.Doc
m.wby.qtbzn.cn/62644.Doc
m.wby.qtbzn.cn/53539.Doc
m.wby.qtbzn.cn/33151.Doc
m.wby.qtbzn.cn/53979.Doc
m.wby.qtbzn.cn/91737.Doc
m.wby.qtbzn.cn/60464.Doc
m.wby.qtbzn.cn/06688.Doc
m.wby.qtbzn.cn/77193.Doc
m.wby.qtbzn.cn/62684.Doc
m.wby.qtbzn.cn/02646.Doc
m.wby.qtbzn.cn/59399.Doc
m.wby.qtbzn.cn/53795.Doc
m.wby.qtbzn.cn/24482.Doc
m.wbt.qtbzn.cn/68064.Doc
m.wbt.qtbzn.cn/26842.Doc
m.wbt.qtbzn.cn/55137.Doc
m.wbt.qtbzn.cn/64406.Doc
m.wbt.qtbzn.cn/28828.Doc
m.wbt.qtbzn.cn/82420.Doc
m.wbt.qtbzn.cn/66022.Doc
m.wbt.qtbzn.cn/00668.Doc
m.wbt.qtbzn.cn/44064.Doc
m.wbt.qtbzn.cn/40804.Doc
m.wbt.qtbzn.cn/40688.Doc
m.wbt.qtbzn.cn/55955.Doc
m.wbt.qtbzn.cn/88206.Doc
m.wbt.qtbzn.cn/17133.Doc
m.wbt.qtbzn.cn/31955.Doc
m.wbt.qtbzn.cn/80060.Doc
m.wbt.qtbzn.cn/62246.Doc
m.wbt.qtbzn.cn/04204.Doc
m.wbt.qtbzn.cn/57939.Doc
m.wbt.qtbzn.cn/99359.Doc
m.wbr.qtbzn.cn/13175.Doc
m.wbr.qtbzn.cn/97993.Doc
m.wbr.qtbzn.cn/28684.Doc
m.wbr.qtbzn.cn/88408.Doc
m.wbr.qtbzn.cn/19337.Doc
m.wbr.qtbzn.cn/53777.Doc
m.wbr.qtbzn.cn/11711.Doc
m.wbr.qtbzn.cn/20408.Doc
m.wbr.qtbzn.cn/20000.Doc
m.wbr.qtbzn.cn/53511.Doc
m.wbr.qtbzn.cn/55375.Doc
m.wbr.qtbzn.cn/06040.Doc
m.wbr.qtbzn.cn/55511.Doc
m.wbr.qtbzn.cn/64042.Doc
m.wbr.qtbzn.cn/13353.Doc
m.wbr.qtbzn.cn/17739.Doc
m.wbr.qtbzn.cn/35955.Doc
m.wbr.qtbzn.cn/20040.Doc
m.wbr.qtbzn.cn/97357.Doc
m.wbr.qtbzn.cn/51375.Doc
m.wbe.qtbzn.cn/62424.Doc
m.wbe.qtbzn.cn/13959.Doc
m.wbe.qtbzn.cn/04440.Doc
m.wbe.qtbzn.cn/62042.Doc
m.wbe.qtbzn.cn/84022.Doc
m.wbe.qtbzn.cn/73159.Doc
m.wbe.qtbzn.cn/53775.Doc
m.wbe.qtbzn.cn/48644.Doc
m.wbe.qtbzn.cn/57397.Doc
m.wbe.qtbzn.cn/88602.Doc
m.wbe.qtbzn.cn/55333.Doc
m.wbe.qtbzn.cn/66842.Doc
m.wbe.qtbzn.cn/97919.Doc
m.wbe.qtbzn.cn/75937.Doc
m.wbe.qtbzn.cn/31557.Doc
m.wbe.qtbzn.cn/28080.Doc
m.wbe.qtbzn.cn/79711.Doc
m.wbe.qtbzn.cn/13971.Doc
m.wbe.qtbzn.cn/51157.Doc
m.wbe.qtbzn.cn/99115.Doc
m.wbw.qtbzn.cn/26622.Doc
m.wbw.qtbzn.cn/53757.Doc
m.wbw.qtbzn.cn/82400.Doc
m.wbw.qtbzn.cn/39999.Doc
m.wbw.qtbzn.cn/53957.Doc
m.wbw.qtbzn.cn/15193.Doc
m.wbw.qtbzn.cn/13397.Doc
m.wbw.qtbzn.cn/44460.Doc
m.wbw.qtbzn.cn/51153.Doc
m.wbw.qtbzn.cn/80626.Doc
m.wbw.qtbzn.cn/22866.Doc
m.wbw.qtbzn.cn/11555.Doc
m.wbw.qtbzn.cn/17959.Doc
m.wbw.qtbzn.cn/37935.Doc
m.wbw.qtbzn.cn/35399.Doc
m.wbw.qtbzn.cn/95591.Doc
m.wbw.qtbzn.cn/82040.Doc
m.wbw.qtbzn.cn/44220.Doc
m.wbw.qtbzn.cn/75375.Doc
m.wbw.qtbzn.cn/55755.Doc
m.wbq.qtbzn.cn/44062.Doc
m.wbq.qtbzn.cn/13955.Doc
m.wbq.qtbzn.cn/19193.Doc
m.wbq.qtbzn.cn/68448.Doc
m.wbq.qtbzn.cn/00468.Doc
m.wbq.qtbzn.cn/04268.Doc
m.wbq.qtbzn.cn/51793.Doc
m.wbq.qtbzn.cn/73759.Doc
m.wbq.qtbzn.cn/33951.Doc
m.wbq.qtbzn.cn/15153.Doc
m.wbq.qtbzn.cn/40242.Doc
m.wbq.qtbzn.cn/93757.Doc
m.wbq.qtbzn.cn/91553.Doc
m.wbq.qtbzn.cn/48680.Doc
m.wbq.qtbzn.cn/51551.Doc
m.wbq.qtbzn.cn/15733.Doc
m.wbq.qtbzn.cn/64084.Doc
m.wbq.qtbzn.cn/48488.Doc
m.wbq.qtbzn.cn/77933.Doc
m.wbq.qtbzn.cn/35333.Doc
m.wvm.qtbzn.cn/71135.Doc
m.wvm.qtbzn.cn/79379.Doc
m.wvm.qtbzn.cn/33735.Doc
m.wvm.qtbzn.cn/39557.Doc
m.wvm.qtbzn.cn/99795.Doc
m.wvm.qtbzn.cn/24880.Doc
m.wvm.qtbzn.cn/59911.Doc
m.wvm.qtbzn.cn/51537.Doc
m.wvm.qtbzn.cn/73711.Doc
m.wvm.qtbzn.cn/68424.Doc
