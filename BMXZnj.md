桨哨绽磺谂


Linux thread_info与task_struct内核栈布局关联

内核栈与thread_info的联编由THREAD_SIZE和arch-specific的栈底寄存器协作完成。x86_64上内核栈大小为16KB（THREAD_SIZE=16*1024），按页对齐分配。栈顶（高地址端）向下增长存放函数调用帧，栈底（低地址端）存放struct thread_info。通过current_thread_info()宏从栈指针sp的低12位掩码获取栈基地址再强制类型转换为thread_info指针。

```c
/* arch/x86/include/asm/thread_info.h */
#ifndef __ASSEMBLY__
struct thread_info {
    unsigned long flags;          /* TIF_xxx标志位 */
    u32           status;         /* 线程状态位，如TS_COMPAT */
};

/* 低12位清零即得栈基地址，也即thread_info基地址 */
#define current_thread_info()   \
    ((struct thread_info *)(current_top_of_stack() - THREAD_SIZE))

/* 也可以直接通过sp低位掩码获取 */
static inline struct thread_info *current_thread_info(void)
{
    return (struct thread_info *)((unsigned long)current_stack_pointer & ~(THREAD_SIZE - 1));
}
#endif
```

x86_64从4.9内核开始采用了arch_thread_struct_setup的分离布局：thread_info不再是内核栈底部的固定成员，而是通过task_struct中的thread_info字段直接嵌入。CONFIG_THREAD_INFO_IN_TASK=y时，thread_info变成task_struct的第一个成员（flags和status直接内联在task_struct中），内核栈底部不包含thread_info。此时current_thread_info()退化为&current->thread_info。

```c
/* include/linux/sched.h */
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
    struct thread_info      thread_info;
#endif
    /* ... */
    void                    *stack;     /* 指向内核栈的指针 */
    /* ... */
};

/* CONFIG_THREAD_INFO_IN_TASK时current_thread_info的实现变为： */
#define current_thread_info()   (&current->thread_info)
```

内核栈通过alloc_thread_stack_node()分配。该函数优先从thread_info_cache的slab缓存中分配，当缓存耗尽时使用__get_free_pages(GFP_KERNEL_ACCOUNT, THREAD_SIZE_ORDER)。分配的页面物理地址直接作为栈基址，并将stack_vm_area结构链接到task_struct->stack上。释放时检查memcg的kmem CG是否已关联，若是则通过memcg_kmem_uncharge退还计费。

```c
static unsigned long *alloc_thread_stack_node(struct task_struct *tsk, int node)
{
    unsigned long *stack;

    stack = kmem_cache_alloc_node(thread_stack_cache, GFP_KERNEL_ACCOUNT, node);
    if (stack)
        goto done;

    stack = __get_free_pages(GFP_KERNEL_ACCOUNT | __GFP_ZERO,
                             THREAD_SIZE_ORDER, node);
    if (!stack)
        return NULL;

done:
    tsk->stack = stack;
    return stack;
}
```

在CONFIG_THREAD_INFO_IN_TASK=y的架构上，内核栈溢出检测依赖STACK_END_MAGIC。alloc_thread_stack_node在栈顶填入一个0x57AC6E9D魔数作为哨兵。每次内核态entry（如syscall或interrupt）检查current->stack + THREAD_SIZE处的魔数是否被改写。若在栈溢出导致栈空间越界到下一个页面时魔数被覆盖，则触发stack overflow异常。

```c
/* kernel/fork.c */
#ifdef CONFIG_STACK_GROWS_SUP
#define STACK_END_MAGIC    0x57AC6E9D
#else
#define STACK_END_MAGIC    0x57AC6E9D
#endif

static void check_stack_usage(void)
{
    static DEFINE_SPINLOCK(low_water_lock);
    static int lowest_to_date = THREAD_SIZE;
    unsigned long unused;

    /* 计算实际栈使用量——从栈顶向下扫描第一个非零字节 */
    for (unused = THREAD_SIZE - sizeof(struct thread_info); unused > 0; unused--)
        if (unlikely(((unsigned long *)current->stack)[unused / sizeof(unsigned long)]))
            break;

    /* unused更新到lowest_to_date时记录到dmesg */
}
```

内核栈布局直接决定schedule()和__switch_to_asm的行为。在__switch_to_asm中，当前进程的RSP保存到task_struct->thread.sp（属于arch特定的struct thread_struct），然后加载next进程的thread.sp。此时栈完全切换——prev进程后续的调度统计update发生在切换后的栈上，但通过rq指针而非当前栈来访问prev workload。

```c
/* arch/x86/entry/entry_64.S */
__switch_to_asm:
    pushq   %rbp
    pushq   %rbx
    pushq   %r12
    pushq   %r13
    pushq   %r14
    pushq   %r15

    movq    %rsp, TASK_threadsp(%rdi)    /* 保存prev->thread.sp */
    movq    TASK_threadsp(%rsi), %rsp    /* 恢复next->thread.sp */

    popq    %r15
    popq    %r14
    popq    %r13
    popq    %r12
    popq    %rbx
    popq    %rbp
    ret
```

存在一个与内核栈相关的double fault竞态：当进程因page fault进入do_page_fault时，若fault本身发生在内核栈已接近溢出的边缘，do_page_fault的栈帧可能超过剩余栈空间触发#DF双故障。在x86_64上，#DF使用独立的IST栈（Interrupt Stack Table），所以不会递归。但如果CONFIG_THREAD_INFO_IN_TASK已启用，task_struct本身不在栈上从而不受影响。

另一个边界问题是内核栈在kasan和kasan_stack模式下会附加额外的shadow map。stack_depot_save()在fork路径中记录stack trace到slab缓存时，会扫描当前内核栈的call frames——如果进程刚创建但尚未执行任何内核调用（即stack区域全零），stack_trace_save_truncated直接返回0。

preemption与栈关联的特殊性：preempt_disable()递增current_thread_info()->preempt_count（在CONFIG_PREEMPT_COUNT=y时），该字段存放在task_struct->thread_info中而不是内核栈上。这意味着即使栈被完全切换（进程切换后），新进程能读到自己的preempt_count而不会误读旧进程的值。但CONFIG_THREAD_INFO_IN_TASK使preempt_count迁移到了thread_info内部，其行为仍保持一致。

```c
/* include/asm-generic/current.h */
#define get_current()   (current_thread_info()->task)

/* include/linux/sched.h */
#define current          get_current()

/* 在x86_64上：percpu的current_task保存着指向task_struct的指针 */
DECLARE_PER_CPU(struct task_struct *, current_task);

#define current           raw_cpu_read(current_task)
```

x86_64的current实现从percpu的current_task变量读取，而非通过内核栈偏移。与arm64不同——arm64通过sp_el0寄存器保存task_struct指针，比percpu读取少一次内存load。两种架构差异导致switch_to()中更新current的方式不同：x86_64写current_task percpu，arm64写sp_el0。

内核栈耗尽的关键保护机制是double fault IST栈。当exception entry检查到rsp低于栈基址时，硬件自动切换到IST栈执行double fault handler。该handler通过dump_stack()输出内核栈完整信息后调用panic()。但若在fork阶段thread_stack_cache的__GFP_ZERO分配意外返回已污染的页面（slab reuse导致残留值），check_stack_usage可能在first-run时误判栈使用量。
刀掌谡赋滓幽赡孛盅牡耙慕瞧兄号

tv.blog.yqvyet.cn/Article/details/913331.sHtML
tv.blog.yqvyet.cn/Article/details/577591.sHtML
tv.blog.yqvyet.cn/Article/details/931111.sHtML
tv.blog.yqvyet.cn/Article/details/399373.sHtML
tv.blog.yqvyet.cn/Article/details/359391.sHtML
tv.blog.yqvyet.cn/Article/details/353511.sHtML
tv.blog.yqvyet.cn/Article/details/917997.sHtML
tv.blog.yqvyet.cn/Article/details/737331.sHtML
tv.blog.yqvyet.cn/Article/details/153537.sHtML
tv.blog.yqvyet.cn/Article/details/337355.sHtML
tv.blog.yqvyet.cn/Article/details/779931.sHtML
tv.blog.yqvyet.cn/Article/details/751519.sHtML
tv.blog.yqvyet.cn/Article/details/173317.sHtML
tv.blog.yqvyet.cn/Article/details/153391.sHtML
tv.blog.yqvyet.cn/Article/details/999135.sHtML
tv.blog.yqvyet.cn/Article/details/773779.sHtML
tv.blog.yqvyet.cn/Article/details/135979.sHtML
tv.blog.yqvyet.cn/Article/details/155539.sHtML
tv.blog.yqvyet.cn/Article/details/119171.sHtML
tv.blog.yqvyet.cn/Article/details/737773.sHtML
tv.blog.yqvyet.cn/Article/details/773915.sHtML
tv.blog.yqvyet.cn/Article/details/591951.sHtML
tv.blog.yqvyet.cn/Article/details/379397.sHtML
tv.blog.yqvyet.cn/Article/details/371937.sHtML
tv.blog.yqvyet.cn/Article/details/975713.sHtML
tv.blog.yqvyet.cn/Article/details/597379.sHtML
tv.blog.yqvyet.cn/Article/details/351535.sHtML
tv.blog.yqvyet.cn/Article/details/799315.sHtML
tv.blog.yqvyet.cn/Article/details/193733.sHtML
tv.blog.yqvyet.cn/Article/details/557957.sHtML
tv.blog.yqvyet.cn/Article/details/737759.sHtML
tv.blog.yqvyet.cn/Article/details/775359.sHtML
tv.blog.yqvyet.cn/Article/details/599371.sHtML
tv.blog.yqvyet.cn/Article/details/775117.sHtML
tv.blog.yqvyet.cn/Article/details/977917.sHtML
tv.blog.yqvyet.cn/Article/details/757975.sHtML
tv.blog.yqvyet.cn/Article/details/513377.sHtML
tv.blog.yqvyet.cn/Article/details/977333.sHtML
tv.blog.yqvyet.cn/Article/details/733399.sHtML
tv.blog.yqvyet.cn/Article/details/395113.sHtML
tv.blog.yqvyet.cn/Article/details/713777.sHtML
tv.blog.yqvyet.cn/Article/details/979573.sHtML
tv.blog.yqvyet.cn/Article/details/373997.sHtML
tv.blog.yqvyet.cn/Article/details/131917.sHtML
tv.blog.yqvyet.cn/Article/details/999537.sHtML
tv.blog.yqvyet.cn/Article/details/317777.sHtML
tv.blog.yqvyet.cn/Article/details/551515.sHtML
tv.blog.yqvyet.cn/Article/details/311795.sHtML
tv.blog.yqvyet.cn/Article/details/977131.sHtML
tv.blog.yqvyet.cn/Article/details/533579.sHtML
tv.blog.yqvyet.cn/Article/details/993779.sHtML
tv.blog.yqvyet.cn/Article/details/959157.sHtML
tv.blog.yqvyet.cn/Article/details/951935.sHtML
tv.blog.yqvyet.cn/Article/details/795393.sHtML
tv.blog.yqvyet.cn/Article/details/519511.sHtML
tv.blog.yqvyet.cn/Article/details/177953.sHtML
tv.blog.yqvyet.cn/Article/details/951559.sHtML
tv.blog.yqvyet.cn/Article/details/735575.sHtML
tv.blog.yqvyet.cn/Article/details/113313.sHtML
tv.blog.yqvyet.cn/Article/details/515177.sHtML
tv.blog.yqvyet.cn/Article/details/935195.sHtML
tv.blog.yqvyet.cn/Article/details/513373.sHtML
tv.blog.yqvyet.cn/Article/details/555157.sHtML
tv.blog.yqvyet.cn/Article/details/719119.sHtML
tv.blog.yqvyet.cn/Article/details/171335.sHtML
tv.blog.yqvyet.cn/Article/details/953595.sHtML
tv.blog.yqvyet.cn/Article/details/975773.sHtML
tv.blog.yqvyet.cn/Article/details/753771.sHtML
tv.blog.yqvyet.cn/Article/details/357513.sHtML
tv.blog.yqvyet.cn/Article/details/975175.sHtML
tv.blog.yqvyet.cn/Article/details/953151.sHtML
tv.blog.yqvyet.cn/Article/details/733577.sHtML
tv.blog.yqvyet.cn/Article/details/913991.sHtML
tv.blog.yqvyet.cn/Article/details/111719.sHtML
tv.blog.yqvyet.cn/Article/details/353571.sHtML
tv.blog.yqvyet.cn/Article/details/991591.sHtML
tv.blog.yqvyet.cn/Article/details/311773.sHtML
tv.blog.yqvyet.cn/Article/details/375359.sHtML
tv.blog.yqvyet.cn/Article/details/599717.sHtML
tv.blog.yqvyet.cn/Article/details/513553.sHtML
tv.blog.yqvyet.cn/Article/details/999951.sHtML
tv.blog.yqvyet.cn/Article/details/517317.sHtML
tv.blog.yqvyet.cn/Article/details/957391.sHtML
tv.blog.yqvyet.cn/Article/details/571959.sHtML
tv.blog.yqvyet.cn/Article/details/779337.sHtML
tv.blog.yqvyet.cn/Article/details/117971.sHtML
tv.blog.yqvyet.cn/Article/details/971373.sHtML
tv.blog.yqvyet.cn/Article/details/191993.sHtML
tv.blog.yqvyet.cn/Article/details/179195.sHtML
tv.blog.yqvyet.cn/Article/details/113975.sHtML
tv.blog.yqvyet.cn/Article/details/395135.sHtML
tv.blog.yqvyet.cn/Article/details/953997.sHtML
tv.blog.yqvyet.cn/Article/details/975515.sHtML
tv.blog.yqvyet.cn/Article/details/917371.sHtML
tv.blog.yqvyet.cn/Article/details/131759.sHtML
tv.blog.yqvyet.cn/Article/details/939751.sHtML
tv.blog.yqvyet.cn/Article/details/737519.sHtML
tv.blog.yqvyet.cn/Article/details/797177.sHtML
tv.blog.yqvyet.cn/Article/details/775137.sHtML
tv.blog.yqvyet.cn/Article/details/793713.sHtML
tv.blog.yqvyet.cn/Article/details/939171.sHtML
tv.blog.yqvyet.cn/Article/details/791733.sHtML
tv.blog.yqvyet.cn/Article/details/777157.sHtML
tv.blog.yqvyet.cn/Article/details/593391.sHtML
tv.blog.yqvyet.cn/Article/details/351773.sHtML
tv.blog.yqvyet.cn/Article/details/931737.sHtML
tv.blog.yqvyet.cn/Article/details/755317.sHtML
tv.blog.yqvyet.cn/Article/details/553395.sHtML
tv.blog.yqvyet.cn/Article/details/717993.sHtML
tv.blog.yqvyet.cn/Article/details/119193.sHtML
tv.blog.yqvyet.cn/Article/details/757513.sHtML
tv.blog.yqvyet.cn/Article/details/139959.sHtML
tv.blog.yqvyet.cn/Article/details/177119.sHtML
tv.blog.yqvyet.cn/Article/details/919335.sHtML
tv.blog.yqvyet.cn/Article/details/755957.sHtML
tv.blog.yqvyet.cn/Article/details/351773.sHtML
tv.blog.yqvyet.cn/Article/details/753311.sHtML
tv.blog.yqvyet.cn/Article/details/599111.sHtML
tv.blog.yqvyet.cn/Article/details/737193.sHtML
tv.blog.yqvyet.cn/Article/details/755559.sHtML
tv.blog.yqvyet.cn/Article/details/599739.sHtML
tv.blog.yqvyet.cn/Article/details/153997.sHtML
tv.blog.yqvyet.cn/Article/details/977939.sHtML
tv.blog.yqvyet.cn/Article/details/717791.sHtML
tv.blog.yqvyet.cn/Article/details/933155.sHtML
tv.blog.yqvyet.cn/Article/details/153757.sHtML
tv.blog.yqvyet.cn/Article/details/353151.sHtML
tv.blog.yqvyet.cn/Article/details/795571.sHtML
tv.blog.yqvyet.cn/Article/details/357931.sHtML
tv.blog.yqvyet.cn/Article/details/311995.sHtML
tv.blog.yqvyet.cn/Article/details/379531.sHtML
tv.blog.yqvyet.cn/Article/details/793911.sHtML
tv.blog.yqvyet.cn/Article/details/395315.sHtML
tv.blog.yqvyet.cn/Article/details/191195.sHtML
tv.blog.yqvyet.cn/Article/details/715777.sHtML
tv.blog.yqvyet.cn/Article/details/135557.sHtML
tv.blog.yqvyet.cn/Article/details/513573.sHtML
tv.blog.yqvyet.cn/Article/details/971357.sHtML
tv.blog.yqvyet.cn/Article/details/513333.sHtML
tv.blog.yqvyet.cn/Article/details/557773.sHtML
tv.blog.yqvyet.cn/Article/details/597595.sHtML
tv.blog.yqvyet.cn/Article/details/359777.sHtML
tv.blog.yqvyet.cn/Article/details/537335.sHtML
tv.blog.yqvyet.cn/Article/details/733557.sHtML
tv.blog.yqvyet.cn/Article/details/979551.sHtML
tv.blog.yqvyet.cn/Article/details/913533.sHtML
tv.blog.yqvyet.cn/Article/details/997915.sHtML
tv.blog.yqvyet.cn/Article/details/795975.sHtML
tv.blog.yqvyet.cn/Article/details/537771.sHtML
tv.blog.yqvyet.cn/Article/details/793735.sHtML
tv.blog.yqvyet.cn/Article/details/573575.sHtML
tv.blog.yqvyet.cn/Article/details/133311.sHtML
tv.blog.yqvyet.cn/Article/details/151757.sHtML
tv.blog.yqvyet.cn/Article/details/931791.sHtML
tv.blog.yqvyet.cn/Article/details/713937.sHtML
tv.blog.yqvyet.cn/Article/details/317393.sHtML
tv.blog.yqvyet.cn/Article/details/135953.sHtML
tv.blog.yqvyet.cn/Article/details/795115.sHtML
tv.blog.yqvyet.cn/Article/details/151517.sHtML
tv.blog.yqvyet.cn/Article/details/973797.sHtML
tv.blog.yqvyet.cn/Article/details/517939.sHtML
tv.blog.yqvyet.cn/Article/details/595135.sHtML
tv.blog.yqvyet.cn/Article/details/915955.sHtML
tv.blog.yqvyet.cn/Article/details/519711.sHtML
tv.blog.yqvyet.cn/Article/details/793715.sHtML
tv.blog.yqvyet.cn/Article/details/711173.sHtML
tv.blog.yqvyet.cn/Article/details/735977.sHtML
tv.blog.yqvyet.cn/Article/details/531595.sHtML
tv.blog.yqvyet.cn/Article/details/371317.sHtML
tv.blog.yqvyet.cn/Article/details/779537.sHtML
tv.blog.yqvyet.cn/Article/details/195755.sHtML
tv.blog.yqvyet.cn/Article/details/359753.sHtML
tv.blog.yqvyet.cn/Article/details/913151.sHtML
tv.blog.yqvyet.cn/Article/details/995711.sHtML
tv.blog.yqvyet.cn/Article/details/193993.sHtML
tv.blog.yqvyet.cn/Article/details/353579.sHtML
tv.blog.yqvyet.cn/Article/details/357995.sHtML
tv.blog.yqvyet.cn/Article/details/357353.sHtML
tv.blog.yqvyet.cn/Article/details/175975.sHtML
tv.blog.yqvyet.cn/Article/details/799711.sHtML
tv.blog.yqvyet.cn/Article/details/597575.sHtML
tv.blog.yqvyet.cn/Article/details/913593.sHtML
tv.blog.yqvyet.cn/Article/details/373511.sHtML
tv.blog.yqvyet.cn/Article/details/777991.sHtML
tv.blog.yqvyet.cn/Article/details/373111.sHtML
tv.blog.yqvyet.cn/Article/details/195573.sHtML
tv.blog.yqvyet.cn/Article/details/373575.sHtML
tv.blog.yqvyet.cn/Article/details/793519.sHtML
tv.blog.yqvyet.cn/Article/details/317375.sHtML
tv.blog.yqvyet.cn/Article/details/971579.sHtML
tv.blog.yqvyet.cn/Article/details/399773.sHtML
tv.blog.yqvyet.cn/Article/details/531539.sHtML
tv.blog.yqvyet.cn/Article/details/175997.sHtML
tv.blog.yqvyet.cn/Article/details/951119.sHtML
tv.blog.yqvyet.cn/Article/details/555373.sHtML
tv.blog.yqvyet.cn/Article/details/131779.sHtML
tv.blog.yqvyet.cn/Article/details/773537.sHtML
tv.blog.yqvyet.cn/Article/details/151559.sHtML
tv.blog.yqvyet.cn/Article/details/931737.sHtML
tv.blog.yqvyet.cn/Article/details/175115.sHtML
tv.blog.yqvyet.cn/Article/details/199151.sHtML
tv.blog.yqvyet.cn/Article/details/591799.sHtML
tv.blog.yqvyet.cn/Article/details/191971.sHtML
tv.blog.yqvyet.cn/Article/details/339373.sHtML
tv.blog.yqvyet.cn/Article/details/335759.sHtML
tv.blog.yqvyet.cn/Article/details/917191.sHtML
tv.blog.yqvyet.cn/Article/details/191793.sHtML
tv.blog.yqvyet.cn/Article/details/193315.sHtML
tv.blog.yqvyet.cn/Article/details/711935.sHtML
tv.blog.yqvyet.cn/Article/details/731991.sHtML
tv.blog.yqvyet.cn/Article/details/931911.sHtML
tv.blog.yqvyet.cn/Article/details/357793.sHtML
tv.blog.yqvyet.cn/Article/details/399553.sHtML
tv.blog.yqvyet.cn/Article/details/379551.sHtML
tv.blog.yqvyet.cn/Article/details/731713.sHtML
tv.blog.yqvyet.cn/Article/details/131517.sHtML
tv.blog.yqvyet.cn/Article/details/913559.sHtML
tv.blog.yqvyet.cn/Article/details/993159.sHtML
tv.blog.yqvyet.cn/Article/details/391795.sHtML
tv.blog.yqvyet.cn/Article/details/935751.sHtML
tv.blog.yqvyet.cn/Article/details/373191.sHtML
tv.blog.yqvyet.cn/Article/details/957115.sHtML
tv.blog.yqvyet.cn/Article/details/731733.sHtML
tv.blog.yqvyet.cn/Article/details/171779.sHtML
tv.blog.yqvyet.cn/Article/details/555575.sHtML
tv.blog.yqvyet.cn/Article/details/919753.sHtML
tv.blog.yqvyet.cn/Article/details/531719.sHtML
tv.blog.yqvyet.cn/Article/details/595593.sHtML
tv.blog.yqvyet.cn/Article/details/515531.sHtML
tv.blog.yqvyet.cn/Article/details/975779.sHtML
tv.blog.yqvyet.cn/Article/details/559935.sHtML
tv.blog.yqvyet.cn/Article/details/711111.sHtML
tv.blog.yqvyet.cn/Article/details/357995.sHtML
tv.blog.yqvyet.cn/Article/details/399191.sHtML
tv.blog.yqvyet.cn/Article/details/111539.sHtML
tv.blog.yqvyet.cn/Article/details/197737.sHtML
tv.blog.yqvyet.cn/Article/details/197319.sHtML
tv.blog.yqvyet.cn/Article/details/331113.sHtML
tv.blog.yqvyet.cn/Article/details/733553.sHtML
tv.blog.yqvyet.cn/Article/details/553717.sHtML
tv.blog.yqvyet.cn/Article/details/171191.sHtML
tv.blog.yqvyet.cn/Article/details/199793.sHtML
tv.blog.yqvyet.cn/Article/details/315711.sHtML
tv.blog.yqvyet.cn/Article/details/137939.sHtML
tv.blog.yqvyet.cn/Article/details/733371.sHtML
tv.blog.yqvyet.cn/Article/details/995973.sHtML
tv.blog.yqvyet.cn/Article/details/575113.sHtML
tv.blog.yqvyet.cn/Article/details/991795.sHtML
tv.blog.yqvyet.cn/Article/details/533115.sHtML
tv.blog.yqvyet.cn/Article/details/193977.sHtML
tv.blog.yqvyet.cn/Article/details/593559.sHtML
tv.blog.yqvyet.cn/Article/details/331151.sHtML
tv.blog.yqvyet.cn/Article/details/355999.sHtML
tv.blog.yqvyet.cn/Article/details/757771.sHtML
tv.blog.yqvyet.cn/Article/details/355993.sHtML
tv.blog.yqvyet.cn/Article/details/999913.sHtML
tv.blog.yqvyet.cn/Article/details/717515.sHtML
tv.blog.yqvyet.cn/Article/details/937355.sHtML
tv.blog.yqvyet.cn/Article/details/999197.sHtML
tv.blog.yqvyet.cn/Article/details/171337.sHtML
tv.blog.yqvyet.cn/Article/details/531995.sHtML
tv.blog.yqvyet.cn/Article/details/557519.sHtML
tv.blog.yqvyet.cn/Article/details/993397.sHtML
tv.blog.yqvyet.cn/Article/details/515931.sHtML
tv.blog.yqvyet.cn/Article/details/117999.sHtML
tv.blog.yqvyet.cn/Article/details/593711.sHtML
tv.blog.yqvyet.cn/Article/details/551919.sHtML
tv.blog.yqvyet.cn/Article/details/179313.sHtML
tv.blog.yqvyet.cn/Article/details/579155.sHtML
tv.blog.yqvyet.cn/Article/details/153931.sHtML
tv.blog.yqvyet.cn/Article/details/555517.sHtML
tv.blog.yqvyet.cn/Article/details/373557.sHtML
tv.blog.yqvyet.cn/Article/details/399199.sHtML
tv.blog.yqvyet.cn/Article/details/957333.sHtML
tv.blog.yqvyet.cn/Article/details/759511.sHtML
tv.blog.yqvyet.cn/Article/details/131919.sHtML
tv.blog.yqvyet.cn/Article/details/551317.sHtML
tv.blog.yqvyet.cn/Article/details/919759.sHtML
tv.blog.yqvyet.cn/Article/details/793991.sHtML
tv.blog.yqvyet.cn/Article/details/375195.sHtML
tv.blog.yqvyet.cn/Article/details/757197.sHtML
tv.blog.yqvyet.cn/Article/details/753555.sHtML
tv.blog.yqvyet.cn/Article/details/533353.sHtML
tv.blog.yqvyet.cn/Article/details/175771.sHtML
tv.blog.yqvyet.cn/Article/details/995533.sHtML
tv.blog.yqvyet.cn/Article/details/335559.sHtML
tv.blog.yqvyet.cn/Article/details/199375.sHtML
tv.blog.yqvyet.cn/Article/details/557997.sHtML
tv.blog.yqvyet.cn/Article/details/515359.sHtML
tv.blog.yqvyet.cn/Article/details/795537.sHtML
tv.blog.yqvyet.cn/Article/details/799531.sHtML
tv.blog.yqvyet.cn/Article/details/553115.sHtML
tv.blog.yqvyet.cn/Article/details/793919.sHtML
tv.blog.yqvyet.cn/Article/details/159957.sHtML
tv.blog.yqvyet.cn/Article/details/171199.sHtML
tv.blog.yqvyet.cn/Article/details/753715.sHtML
tv.blog.yqvyet.cn/Article/details/119399.sHtML
tv.blog.yqvyet.cn/Article/details/771319.sHtML
tv.blog.yqvyet.cn/Article/details/771359.sHtML
tv.blog.yqvyet.cn/Article/details/553919.sHtML
tv.blog.yqvyet.cn/Article/details/995593.sHtML
tv.blog.yqvyet.cn/Article/details/975371.sHtML
tv.blog.yqvyet.cn/Article/details/355373.sHtML
tv.blog.yqvyet.cn/Article/details/999153.sHtML
tv.blog.yqvyet.cn/Article/details/579335.sHtML
tv.blog.yqvyet.cn/Article/details/397519.sHtML
tv.blog.yqvyet.cn/Article/details/171973.sHtML
tv.blog.yqvyet.cn/Article/details/759179.sHtML
tv.blog.yqvyet.cn/Article/details/317131.sHtML
tv.blog.yqvyet.cn/Article/details/377197.sHtML
tv.blog.yqvyet.cn/Article/details/593319.sHtML
tv.blog.yqvyet.cn/Article/details/991517.sHtML
tv.blog.yqvyet.cn/Article/details/973779.sHtML
tv.blog.yqvyet.cn/Article/details/517571.sHtML
tv.blog.yqvyet.cn/Article/details/731915.sHtML
tv.blog.yqvyet.cn/Article/details/391511.sHtML
tv.blog.yqvyet.cn/Article/details/313919.sHtML
tv.blog.yqvyet.cn/Article/details/733317.sHtML
tv.blog.yqvyet.cn/Article/details/199991.sHtML
tv.blog.yqvyet.cn/Article/details/191575.sHtML
tv.blog.yqvyet.cn/Article/details/791313.sHtML
tv.blog.yqvyet.cn/Article/details/139377.sHtML
tv.blog.yqvyet.cn/Article/details/911319.sHtML
tv.blog.yqvyet.cn/Article/details/513597.sHtML
tv.blog.yqvyet.cn/Article/details/199771.sHtML
tv.blog.yqvyet.cn/Article/details/193371.sHtML
tv.blog.yqvyet.cn/Article/details/426422.sHtML
tv.blog.yqvyet.cn/Article/details/446644.sHtML
tv.blog.yqvyet.cn/Article/details/668822.sHtML
tv.blog.yqvyet.cn/Article/details/864608.sHtML
tv.blog.yqvyet.cn/Article/details/242428.sHtML
tv.blog.yqvyet.cn/Article/details/648648.sHtML
tv.blog.yqvyet.cn/Article/details/260268.sHtML
tv.blog.yqvyet.cn/Article/details/402424.sHtML
tv.blog.yqvyet.cn/Article/details/066002.sHtML
tv.blog.yqvyet.cn/Article/details/822486.sHtML
tv.blog.yqvyet.cn/Article/details/599191.sHtML
tv.blog.yqvyet.cn/Article/details/971971.sHtML
tv.blog.yqvyet.cn/Article/details/191975.sHtML
tv.blog.yqvyet.cn/Article/details/640028.sHtML
tv.blog.yqvyet.cn/Article/details/971399.sHtML
tv.blog.yqvyet.cn/Article/details/397993.sHtML
tv.blog.yqvyet.cn/Article/details/799359.sHtML
tv.blog.yqvyet.cn/Article/details/440648.sHtML
tv.blog.yqvyet.cn/Article/details/468882.sHtML
tv.blog.yqvyet.cn/Article/details/331515.sHtML
tv.blog.yqvyet.cn/Article/details/848280.sHtML
tv.blog.yqvyet.cn/Article/details/046888.sHtML
tv.blog.yqvyet.cn/Article/details/202406.sHtML
tv.blog.yqvyet.cn/Article/details/020044.sHtML
tv.blog.yqvyet.cn/Article/details/599551.sHtML
tv.blog.yqvyet.cn/Article/details/086088.sHtML
tv.blog.yqvyet.cn/Article/details/060826.sHtML
tv.blog.yqvyet.cn/Article/details/951755.sHtML
tv.blog.yqvyet.cn/Article/details/791975.sHtML
tv.blog.yqvyet.cn/Article/details/824240.sHtML
tv.blog.yqvyet.cn/Article/details/553979.sHtML
tv.blog.yqvyet.cn/Article/details/397733.sHtML
tv.blog.yqvyet.cn/Article/details/866620.sHtML
tv.blog.yqvyet.cn/Article/details/828806.sHtML
tv.blog.yqvyet.cn/Article/details/157739.sHtML
tv.blog.yqvyet.cn/Article/details/202868.sHtML
tv.blog.yqvyet.cn/Article/details/755973.sHtML
tv.blog.yqvyet.cn/Article/details/686226.sHtML
tv.blog.yqvyet.cn/Article/details/020488.sHtML
tv.blog.yqvyet.cn/Article/details/606628.sHtML
tv.blog.yqvyet.cn/Article/details/682262.sHtML
tv.blog.yqvyet.cn/Article/details/246008.sHtML
tv.blog.yqvyet.cn/Article/details/820266.sHtML
tv.blog.yqvyet.cn/Article/details/482428.sHtML
tv.blog.yqvyet.cn/Article/details/086842.sHtML
tv.blog.yqvyet.cn/Article/details/240086.sHtML
tv.blog.yqvyet.cn/Article/details/226666.sHtML
tv.blog.yqvyet.cn/Article/details/064868.sHtML
tv.blog.yqvyet.cn/Article/details/282024.sHtML
tv.blog.yqvyet.cn/Article/details/806464.sHtML
tv.blog.yqvyet.cn/Article/details/022862.sHtML
tv.blog.yqvyet.cn/Article/details/199733.sHtML
tv.blog.yqvyet.cn/Article/details/802800.sHtML
tv.blog.yqvyet.cn/Article/details/335953.sHtML
tv.blog.yqvyet.cn/Article/details/066604.sHtML
tv.blog.yqvyet.cn/Article/details/222046.sHtML
tv.blog.yqvyet.cn/Article/details/246864.sHtML
tv.blog.yqvyet.cn/Article/details/759713.sHtML
tv.blog.yqvyet.cn/Article/details/042802.sHtML
tv.blog.yqvyet.cn/Article/details/991137.sHtML
tv.blog.yqvyet.cn/Article/details/280264.sHtML
tv.blog.yqvyet.cn/Article/details/195593.sHtML
tv.blog.yqvyet.cn/Article/details/860022.sHtML
tv.blog.yqvyet.cn/Article/details/399159.sHtML
tv.blog.yqvyet.cn/Article/details/973797.sHtML
tv.blog.yqvyet.cn/Article/details/282444.sHtML
tv.blog.yqvyet.cn/Article/details/426884.sHtML
tv.blog.yqvyet.cn/Article/details/462044.sHtML
tv.blog.yqvyet.cn/Article/details/759135.sHtML
