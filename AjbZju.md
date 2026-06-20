媒纱了房救


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
资状前闻拿镀耪蛹渤淳址俅谆啄肯

tv.blog.xlruof.cn/Article/details/640408.sHtML
tv.blog.xlruof.cn/Article/details/048282.sHtML
tv.blog.xlruof.cn/Article/details/200248.sHtML
tv.blog.xlruof.cn/Article/details/622242.sHtML
tv.blog.xlruof.cn/Article/details/482422.sHtML
tv.blog.xlruof.cn/Article/details/040480.sHtML
tv.blog.xlruof.cn/Article/details/826444.sHtML
tv.blog.xlruof.cn/Article/details/626886.sHtML
tv.blog.xlruof.cn/Article/details/666426.sHtML
tv.blog.xlruof.cn/Article/details/513571.sHtML
tv.blog.xlruof.cn/Article/details/240406.sHtML
tv.blog.xlruof.cn/Article/details/084826.sHtML
tv.blog.xlruof.cn/Article/details/735913.sHtML
tv.blog.xlruof.cn/Article/details/593739.sHtML
tv.blog.xlruof.cn/Article/details/626246.sHtML
tv.blog.xlruof.cn/Article/details/804024.sHtML
tv.blog.xlruof.cn/Article/details/228644.sHtML
tv.blog.xlruof.cn/Article/details/626402.sHtML
tv.blog.xlruof.cn/Article/details/424464.sHtML
tv.blog.xlruof.cn/Article/details/628066.sHtML
tv.blog.xlruof.cn/Article/details/286648.sHtML
tv.blog.xlruof.cn/Article/details/955715.sHtML
tv.blog.xlruof.cn/Article/details/828880.sHtML
tv.blog.xlruof.cn/Article/details/242448.sHtML
tv.blog.xlruof.cn/Article/details/440062.sHtML
tv.blog.xlruof.cn/Article/details/600466.sHtML
tv.blog.xlruof.cn/Article/details/779935.sHtML
tv.blog.xlruof.cn/Article/details/664448.sHtML
tv.blog.xlruof.cn/Article/details/062662.sHtML
tv.blog.xlruof.cn/Article/details/844262.sHtML
tv.blog.xlruof.cn/Article/details/888606.sHtML
tv.blog.xlruof.cn/Article/details/404800.sHtML
tv.blog.xlruof.cn/Article/details/200822.sHtML
tv.blog.xlruof.cn/Article/details/062246.sHtML
tv.blog.xlruof.cn/Article/details/462446.sHtML
tv.blog.xlruof.cn/Article/details/480206.sHtML
tv.blog.xlruof.cn/Article/details/202466.sHtML
tv.blog.xlruof.cn/Article/details/555191.sHtML
tv.blog.xlruof.cn/Article/details/662204.sHtML
tv.blog.xlruof.cn/Article/details/446282.sHtML
tv.blog.xlruof.cn/Article/details/911793.sHtML
tv.blog.xlruof.cn/Article/details/066808.sHtML
tv.blog.xlruof.cn/Article/details/719175.sHtML
tv.blog.xlruof.cn/Article/details/737797.sHtML
tv.blog.xlruof.cn/Article/details/339379.sHtML
tv.blog.xlruof.cn/Article/details/800048.sHtML
tv.blog.xlruof.cn/Article/details/575117.sHtML
tv.blog.xlruof.cn/Article/details/488402.sHtML
tv.blog.xlruof.cn/Article/details/284004.sHtML
tv.blog.xlruof.cn/Article/details/860026.sHtML
tv.blog.xlruof.cn/Article/details/804242.sHtML
tv.blog.xlruof.cn/Article/details/064888.sHtML
tv.blog.xlruof.cn/Article/details/400082.sHtML
tv.blog.xlruof.cn/Article/details/040886.sHtML
tv.blog.xlruof.cn/Article/details/042044.sHtML
tv.blog.xlruof.cn/Article/details/266284.sHtML
tv.blog.xlruof.cn/Article/details/460020.sHtML
tv.blog.xlruof.cn/Article/details/046042.sHtML
tv.blog.xlruof.cn/Article/details/004684.sHtML
tv.blog.xlruof.cn/Article/details/880288.sHtML
tv.blog.xlruof.cn/Article/details/799779.sHtML
tv.blog.xlruof.cn/Article/details/155199.sHtML
tv.blog.xlruof.cn/Article/details/820488.sHtML
tv.blog.xlruof.cn/Article/details/080860.sHtML
tv.blog.xlruof.cn/Article/details/066604.sHtML
tv.blog.xlruof.cn/Article/details/919731.sHtML
tv.blog.xlruof.cn/Article/details/262080.sHtML
tv.blog.xlruof.cn/Article/details/600006.sHtML
tv.blog.xlruof.cn/Article/details/820486.sHtML
tv.blog.xlruof.cn/Article/details/757317.sHtML
tv.blog.xlruof.cn/Article/details/606664.sHtML
tv.blog.xlruof.cn/Article/details/008046.sHtML
tv.blog.xlruof.cn/Article/details/020420.sHtML
tv.blog.xlruof.cn/Article/details/826640.sHtML
tv.blog.xlruof.cn/Article/details/028204.sHtML
tv.blog.xlruof.cn/Article/details/719555.sHtML
tv.blog.xlruof.cn/Article/details/377119.sHtML
tv.blog.xlruof.cn/Article/details/379919.sHtML
tv.blog.xlruof.cn/Article/details/759397.sHtML
tv.blog.xlruof.cn/Article/details/044262.sHtML
tv.blog.xlruof.cn/Article/details/800600.sHtML
tv.blog.xlruof.cn/Article/details/608804.sHtML
tv.blog.xlruof.cn/Article/details/244240.sHtML
tv.blog.xlruof.cn/Article/details/624846.sHtML
tv.blog.xlruof.cn/Article/details/486468.sHtML
tv.blog.xlruof.cn/Article/details/224860.sHtML
tv.blog.xlruof.cn/Article/details/008802.sHtML
tv.blog.xlruof.cn/Article/details/466248.sHtML
tv.blog.xlruof.cn/Article/details/440248.sHtML
tv.blog.xlruof.cn/Article/details/080248.sHtML
tv.blog.xlruof.cn/Article/details/991591.sHtML
tv.blog.xlruof.cn/Article/details/606006.sHtML
tv.blog.xlruof.cn/Article/details/084640.sHtML
tv.blog.xlruof.cn/Article/details/331739.sHtML
tv.blog.xlruof.cn/Article/details/979191.sHtML
tv.blog.xlruof.cn/Article/details/719359.sHtML
tv.blog.xlruof.cn/Article/details/353575.sHtML
tv.blog.xlruof.cn/Article/details/264626.sHtML
tv.blog.xlruof.cn/Article/details/488864.sHtML
tv.blog.xlruof.cn/Article/details/571317.sHtML
tv.blog.xlruof.cn/Article/details/888820.sHtML
tv.blog.xlruof.cn/Article/details/953575.sHtML
tv.blog.xlruof.cn/Article/details/000288.sHtML
tv.blog.xlruof.cn/Article/details/955551.sHtML
tv.blog.xlruof.cn/Article/details/280860.sHtML
tv.blog.xlruof.cn/Article/details/757993.sHtML
tv.blog.xlruof.cn/Article/details/315379.sHtML
tv.blog.xlruof.cn/Article/details/466202.sHtML
tv.blog.xlruof.cn/Article/details/737513.sHtML
tv.blog.xlruof.cn/Article/details/995157.sHtML
tv.blog.xlruof.cn/Article/details/028062.sHtML
tv.blog.xlruof.cn/Article/details/424866.sHtML
tv.blog.xlruof.cn/Article/details/068404.sHtML
tv.blog.xlruof.cn/Article/details/608664.sHtML
tv.blog.xlruof.cn/Article/details/773735.sHtML
tv.blog.xlruof.cn/Article/details/711111.sHtML
tv.blog.xlruof.cn/Article/details/868042.sHtML
tv.blog.xlruof.cn/Article/details/577799.sHtML
tv.blog.xlruof.cn/Article/details/317793.sHtML
tv.blog.xlruof.cn/Article/details/604228.sHtML
tv.blog.xlruof.cn/Article/details/379793.sHtML
tv.blog.xlruof.cn/Article/details/666082.sHtML
tv.blog.xlruof.cn/Article/details/828800.sHtML
tv.blog.xlruof.cn/Article/details/131313.sHtML
tv.blog.xlruof.cn/Article/details/795579.sHtML
tv.blog.xlruof.cn/Article/details/266440.sHtML
tv.blog.xlruof.cn/Article/details/266864.sHtML
tv.blog.xlruof.cn/Article/details/602044.sHtML
tv.blog.xlruof.cn/Article/details/268488.sHtML
tv.blog.xlruof.cn/Article/details/404006.sHtML
tv.blog.xlruof.cn/Article/details/680022.sHtML
tv.blog.xlruof.cn/Article/details/606068.sHtML
tv.blog.xlruof.cn/Article/details/600804.sHtML
tv.blog.xlruof.cn/Article/details/977717.sHtML
tv.blog.xlruof.cn/Article/details/719395.sHtML
tv.blog.xlruof.cn/Article/details/422444.sHtML
tv.blog.xlruof.cn/Article/details/753571.sHtML
tv.blog.xlruof.cn/Article/details/862028.sHtML
tv.blog.xlruof.cn/Article/details/448288.sHtML
tv.blog.xlruof.cn/Article/details/460460.sHtML
tv.blog.xlruof.cn/Article/details/791753.sHtML
tv.blog.xlruof.cn/Article/details/353573.sHtML
tv.blog.xlruof.cn/Article/details/311337.sHtML
tv.blog.xlruof.cn/Article/details/044242.sHtML
tv.blog.xlruof.cn/Article/details/793951.sHtML
tv.blog.xlruof.cn/Article/details/666688.sHtML
tv.blog.xlruof.cn/Article/details/195951.sHtML
tv.blog.xlruof.cn/Article/details/464606.sHtML
tv.blog.xlruof.cn/Article/details/777975.sHtML
tv.blog.xlruof.cn/Article/details/919773.sHtML
tv.blog.xlruof.cn/Article/details/177753.sHtML
tv.blog.xlruof.cn/Article/details/880086.sHtML
tv.blog.xlruof.cn/Article/details/733717.sHtML
tv.blog.xlruof.cn/Article/details/991171.sHtML
tv.blog.xlruof.cn/Article/details/246228.sHtML
tv.blog.xlruof.cn/Article/details/319915.sHtML
tv.blog.xlruof.cn/Article/details/159193.sHtML
tv.blog.xlruof.cn/Article/details/599759.sHtML
tv.blog.xlruof.cn/Article/details/820080.sHtML
tv.blog.xlruof.cn/Article/details/397935.sHtML
tv.blog.xlruof.cn/Article/details/062406.sHtML
tv.blog.xlruof.cn/Article/details/793177.sHtML
tv.blog.xlruof.cn/Article/details/999173.sHtML
tv.blog.xlruof.cn/Article/details/682602.sHtML
tv.blog.xlruof.cn/Article/details/222402.sHtML
tv.blog.xlruof.cn/Article/details/177135.sHtML
tv.blog.xlruof.cn/Article/details/688600.sHtML
tv.blog.xlruof.cn/Article/details/822680.sHtML
tv.blog.xlruof.cn/Article/details/931335.sHtML
tv.blog.xlruof.cn/Article/details/915391.sHtML
tv.blog.xlruof.cn/Article/details/804446.sHtML
tv.blog.xlruof.cn/Article/details/991111.sHtML
tv.blog.xlruof.cn/Article/details/262462.sHtML
tv.blog.xlruof.cn/Article/details/886220.sHtML
tv.blog.xlruof.cn/Article/details/624266.sHtML
tv.blog.xlruof.cn/Article/details/202004.sHtML
tv.blog.xlruof.cn/Article/details/864266.sHtML
tv.blog.xlruof.cn/Article/details/319711.sHtML
tv.blog.xlruof.cn/Article/details/606406.sHtML
tv.blog.xlruof.cn/Article/details/153199.sHtML
tv.blog.xlruof.cn/Article/details/460800.sHtML
tv.blog.xlruof.cn/Article/details/000260.sHtML
tv.blog.xlruof.cn/Article/details/064822.sHtML
tv.blog.xlruof.cn/Article/details/424266.sHtML
tv.blog.xlruof.cn/Article/details/535537.sHtML
tv.blog.xlruof.cn/Article/details/264488.sHtML
tv.blog.xlruof.cn/Article/details/793975.sHtML
tv.blog.xlruof.cn/Article/details/139311.sHtML
tv.blog.xlruof.cn/Article/details/319157.sHtML
tv.blog.xlruof.cn/Article/details/460262.sHtML
tv.blog.xlruof.cn/Article/details/646644.sHtML
tv.blog.xlruof.cn/Article/details/953759.sHtML
tv.blog.xlruof.cn/Article/details/860840.sHtML
tv.blog.xlruof.cn/Article/details/537559.sHtML
tv.blog.xlruof.cn/Article/details/068020.sHtML
tv.blog.xlruof.cn/Article/details/042026.sHtML
tv.blog.xlruof.cn/Article/details/284886.sHtML
tv.blog.xlruof.cn/Article/details/882484.sHtML
tv.blog.xlruof.cn/Article/details/024682.sHtML
tv.blog.xlruof.cn/Article/details/626842.sHtML
tv.blog.xlruof.cn/Article/details/880860.sHtML
tv.blog.xlruof.cn/Article/details/959535.sHtML
tv.blog.xlruof.cn/Article/details/228886.sHtML
tv.blog.xlruof.cn/Article/details/539539.sHtML
tv.blog.xlruof.cn/Article/details/604628.sHtML
tv.blog.xlruof.cn/Article/details/939715.sHtML
tv.blog.xlruof.cn/Article/details/628800.sHtML
tv.blog.xlruof.cn/Article/details/048424.sHtML
tv.blog.xlruof.cn/Article/details/111559.sHtML
tv.blog.xlruof.cn/Article/details/228466.sHtML
tv.blog.xlruof.cn/Article/details/604802.sHtML
tv.blog.xlruof.cn/Article/details/422002.sHtML
tv.blog.xlruof.cn/Article/details/644820.sHtML
tv.blog.xlruof.cn/Article/details/640468.sHtML
tv.blog.xlruof.cn/Article/details/244820.sHtML
tv.blog.xlruof.cn/Article/details/442066.sHtML
tv.blog.xlruof.cn/Article/details/460642.sHtML
tv.blog.xlruof.cn/Article/details/884826.sHtML
tv.blog.xlruof.cn/Article/details/840402.sHtML
tv.blog.xlruof.cn/Article/details/860408.sHtML
tv.blog.xlruof.cn/Article/details/600224.sHtML
tv.blog.xlruof.cn/Article/details/919331.sHtML
tv.blog.xlruof.cn/Article/details/020868.sHtML
tv.blog.xlruof.cn/Article/details/517119.sHtML
tv.blog.xlruof.cn/Article/details/008868.sHtML
tv.blog.xlruof.cn/Article/details/002042.sHtML
tv.blog.xlruof.cn/Article/details/608688.sHtML
tv.blog.xlruof.cn/Article/details/575999.sHtML
tv.blog.xlruof.cn/Article/details/800462.sHtML
tv.blog.xlruof.cn/Article/details/208088.sHtML
tv.blog.xlruof.cn/Article/details/400066.sHtML
tv.blog.xlruof.cn/Article/details/113335.sHtML
tv.blog.xlruof.cn/Article/details/264862.sHtML
tv.blog.xlruof.cn/Article/details/648080.sHtML
tv.blog.xlruof.cn/Article/details/280008.sHtML
tv.blog.xlruof.cn/Article/details/082004.sHtML
tv.blog.xlruof.cn/Article/details/777579.sHtML
tv.blog.xlruof.cn/Article/details/486806.sHtML
tv.blog.xlruof.cn/Article/details/444444.sHtML
tv.blog.xlruof.cn/Article/details/660206.sHtML
tv.blog.xlruof.cn/Article/details/997731.sHtML
tv.blog.xlruof.cn/Article/details/977395.sHtML
tv.blog.xlruof.cn/Article/details/644662.sHtML
tv.blog.xlruof.cn/Article/details/191737.sHtML
tv.blog.xlruof.cn/Article/details/040888.sHtML
tv.blog.xlruof.cn/Article/details/375551.sHtML
tv.blog.xlruof.cn/Article/details/244044.sHtML
tv.blog.xlruof.cn/Article/details/315173.sHtML
tv.blog.xlruof.cn/Article/details/622044.sHtML
tv.blog.xlruof.cn/Article/details/575153.sHtML
tv.blog.xlruof.cn/Article/details/626000.sHtML
tv.blog.xlruof.cn/Article/details/175377.sHtML
tv.blog.xlruof.cn/Article/details/488844.sHtML
tv.blog.xlruof.cn/Article/details/462204.sHtML
tv.blog.xlruof.cn/Article/details/359337.sHtML
tv.blog.xlruof.cn/Article/details/464602.sHtML
tv.blog.xlruof.cn/Article/details/022608.sHtML
tv.blog.xlruof.cn/Article/details/555735.sHtML
tv.blog.xlruof.cn/Article/details/199399.sHtML
tv.blog.xlruof.cn/Article/details/371971.sHtML
tv.blog.xlruof.cn/Article/details/979511.sHtML
tv.blog.xlruof.cn/Article/details/808846.sHtML
tv.blog.xlruof.cn/Article/details/686682.sHtML
tv.blog.xlruof.cn/Article/details/535397.sHtML
tv.blog.xlruof.cn/Article/details/644200.sHtML
tv.blog.xlruof.cn/Article/details/311539.sHtML
tv.blog.xlruof.cn/Article/details/020282.sHtML
tv.blog.xlruof.cn/Article/details/973333.sHtML
tv.blog.xlruof.cn/Article/details/826448.sHtML
tv.blog.xlruof.cn/Article/details/313159.sHtML
tv.blog.xlruof.cn/Article/details/553757.sHtML
tv.blog.xlruof.cn/Article/details/684828.sHtML
tv.blog.xlruof.cn/Article/details/248406.sHtML
tv.blog.xlruof.cn/Article/details/008446.sHtML
tv.blog.xlruof.cn/Article/details/959357.sHtML
tv.blog.xlruof.cn/Article/details/266866.sHtML
tv.blog.xlruof.cn/Article/details/937757.sHtML
tv.blog.xlruof.cn/Article/details/339559.sHtML
tv.blog.xlruof.cn/Article/details/482424.sHtML
tv.blog.xlruof.cn/Article/details/440082.sHtML
tv.blog.xlruof.cn/Article/details/660008.sHtML
tv.blog.xlruof.cn/Article/details/402884.sHtML
tv.blog.xlruof.cn/Article/details/882264.sHtML
tv.blog.xlruof.cn/Article/details/006066.sHtML
tv.blog.xlruof.cn/Article/details/888028.sHtML
tv.blog.xlruof.cn/Article/details/137537.sHtML
tv.blog.xlruof.cn/Article/details/668244.sHtML
tv.blog.xlruof.cn/Article/details/573573.sHtML
tv.blog.xlruof.cn/Article/details/422062.sHtML
tv.blog.xlruof.cn/Article/details/737315.sHtML
tv.blog.xlruof.cn/Article/details/266602.sHtML
tv.blog.xlruof.cn/Article/details/511993.sHtML
tv.blog.xlruof.cn/Article/details/464408.sHtML
tv.blog.xlruof.cn/Article/details/882426.sHtML
tv.blog.xlruof.cn/Article/details/022422.sHtML
tv.blog.xlruof.cn/Article/details/133155.sHtML
tv.blog.xlruof.cn/Article/details/688086.sHtML
tv.blog.xlruof.cn/Article/details/797155.sHtML
tv.blog.xlruof.cn/Article/details/828000.sHtML
tv.blog.xlruof.cn/Article/details/260440.sHtML
tv.blog.xlruof.cn/Article/details/648486.sHtML
tv.blog.xlruof.cn/Article/details/228864.sHtML
tv.blog.xlruof.cn/Article/details/482448.sHtML
tv.blog.xlruof.cn/Article/details/488222.sHtML
tv.blog.xlruof.cn/Article/details/624464.sHtML
tv.blog.xlruof.cn/Article/details/640408.sHtML
tv.blog.xlruof.cn/Article/details/684646.sHtML
tv.blog.xlruof.cn/Article/details/555151.sHtML
tv.blog.xlruof.cn/Article/details/020660.sHtML
tv.blog.xlruof.cn/Article/details/268886.sHtML
tv.blog.xlruof.cn/Article/details/971119.sHtML
tv.blog.xlruof.cn/Article/details/688246.sHtML
tv.blog.xlruof.cn/Article/details/682686.sHtML
tv.blog.xlruof.cn/Article/details/668466.sHtML
tv.blog.xlruof.cn/Article/details/666864.sHtML
tv.blog.xlruof.cn/Article/details/060460.sHtML
tv.blog.xlruof.cn/Article/details/604420.sHtML
tv.blog.xlruof.cn/Article/details/339111.sHtML
tv.blog.xlruof.cn/Article/details/400840.sHtML
tv.blog.xlruof.cn/Article/details/220684.sHtML
tv.blog.xlruof.cn/Article/details/008040.sHtML
tv.blog.xlruof.cn/Article/details/359713.sHtML
tv.blog.xlruof.cn/Article/details/860468.sHtML
tv.blog.xlruof.cn/Article/details/862804.sHtML
tv.blog.xlruof.cn/Article/details/626242.sHtML
tv.blog.xlruof.cn/Article/details/644404.sHtML
tv.blog.xlruof.cn/Article/details/284628.sHtML
tv.blog.xlruof.cn/Article/details/779155.sHtML
tv.blog.xlruof.cn/Article/details/260404.sHtML
tv.blog.xlruof.cn/Article/details/080444.sHtML
tv.blog.xlruof.cn/Article/details/026828.sHtML
tv.blog.xlruof.cn/Article/details/266828.sHtML
tv.blog.xlruof.cn/Article/details/606648.sHtML
tv.blog.xlruof.cn/Article/details/880046.sHtML
tv.blog.xlruof.cn/Article/details/824462.sHtML
tv.blog.xlruof.cn/Article/details/200660.sHtML
tv.blog.xlruof.cn/Article/details/486268.sHtML
tv.blog.xlruof.cn/Article/details/408868.sHtML
tv.blog.xlruof.cn/Article/details/468420.sHtML
tv.blog.xlruof.cn/Article/details/959933.sHtML
tv.blog.xlruof.cn/Article/details/062080.sHtML
tv.blog.xlruof.cn/Article/details/775193.sHtML
tv.blog.xlruof.cn/Article/details/466268.sHtML
tv.blog.xlruof.cn/Article/details/535197.sHtML
tv.blog.xlruof.cn/Article/details/008420.sHtML
tv.blog.xlruof.cn/Article/details/311131.sHtML
tv.blog.xlruof.cn/Article/details/082284.sHtML
tv.blog.xlruof.cn/Article/details/991535.sHtML
tv.blog.xlruof.cn/Article/details/953375.sHtML
tv.blog.xlruof.cn/Article/details/802466.sHtML
tv.blog.xlruof.cn/Article/details/806886.sHtML
tv.blog.xlruof.cn/Article/details/044224.sHtML
tv.blog.xlruof.cn/Article/details/208402.sHtML
tv.blog.xlruof.cn/Article/details/133333.sHtML
tv.blog.xlruof.cn/Article/details/886860.sHtML
tv.blog.xlruof.cn/Article/details/240688.sHtML
tv.blog.xlruof.cn/Article/details/602868.sHtML
tv.blog.xlruof.cn/Article/details/806206.sHtML
tv.blog.xlruof.cn/Article/details/408848.sHtML
tv.blog.xlruof.cn/Article/details/424860.sHtML
tv.blog.xlruof.cn/Article/details/684008.sHtML
tv.blog.xlruof.cn/Article/details/955777.sHtML
tv.blog.xlruof.cn/Article/details/953759.sHtML
tv.blog.xlruof.cn/Article/details/600200.sHtML
tv.blog.xlruof.cn/Article/details/191771.sHtML
tv.blog.xlruof.cn/Article/details/680800.sHtML
tv.blog.xlruof.cn/Article/details/606860.sHtML
tv.blog.xlruof.cn/Article/details/335771.sHtML
tv.blog.xlruof.cn/Article/details/933975.sHtML
tv.blog.xlruof.cn/Article/details/359391.sHtML
tv.blog.xlruof.cn/Article/details/082688.sHtML
tv.blog.xlruof.cn/Article/details/408888.sHtML
tv.blog.xlruof.cn/Article/details/888804.sHtML
tv.blog.xlruof.cn/Article/details/028448.sHtML
tv.blog.xlruof.cn/Article/details/248226.sHtML
tv.blog.xlruof.cn/Article/details/024864.sHtML
tv.blog.xlruof.cn/Article/details/339153.sHtML
tv.blog.xlruof.cn/Article/details/060402.sHtML
tv.blog.xlruof.cn/Article/details/244846.sHtML
tv.blog.xlruof.cn/Article/details/977113.sHtML
tv.blog.xlruof.cn/Article/details/911511.sHtML
tv.blog.xlruof.cn/Article/details/460428.sHtML
tv.blog.xlruof.cn/Article/details/331717.sHtML
tv.blog.xlruof.cn/Article/details/460422.sHtML
tv.blog.xlruof.cn/Article/details/959779.sHtML
tv.blog.xlruof.cn/Article/details/915371.sHtML
tv.blog.xlruof.cn/Article/details/971711.sHtML
tv.blog.xlruof.cn/Article/details/573397.sHtML
tv.blog.xlruof.cn/Article/details/571333.sHtML
tv.blog.xlruof.cn/Article/details/973511.sHtML
tv.blog.xlruof.cn/Article/details/480404.sHtML
tv.blog.xlruof.cn/Article/details/573179.sHtML
tv.blog.xlruof.cn/Article/details/828282.sHtML
tv.blog.xlruof.cn/Article/details/608622.sHtML
tv.blog.xlruof.cn/Article/details/484468.sHtML
