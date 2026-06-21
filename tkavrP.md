哺欠尉泼现


Linux sched_init调度器初始化与idle线程创建

sched_init是内核调度子系统的初始化入口,定义在kernel/sched/core.c中。它在start_kernel中很早被调用,负责初始化调度器的核心数据结构和为当前CPU创建idle线程。调度器使用sched_class链表管理多个调度类,每个调度类决定不同策略下进程的调度行为。

函数入口如下:

```c
// kernel/sched/core.c
void __init sched_init(void)
{
    int i;
    unsigned long ptr;

    // 初始化调度类优先级链表
    // 调度类按优先级从高到低排列:
    // stop_sched_class -> dl_sched_class -> rt_sched_class
    // -> fair_sched_class -> idle_sched_class
    INIT_LIST_HEAD(&stop_sched_class.sibling);
    // ... 其他调度类链表初始化

    // 等待队列哈希表初始化
    waitq_hash_size = 1UL << wait_table_bits;
    waitq_hash = (struct wait_queue_head *)
        alloc_large_system_hash("Waitqueues",
            sizeof(struct wait_queue_head),
            0, wait_table_bits,
            HASH_EARLY | HASH_ZERO,
            NULL, NULL, 0, 0);

    // 初始化每个CPU的rq(runqueue)
    for_each_possible_cpu(i) {
        struct rq *rq = cpu_rq(i);
        raw_spin_lock_init(&rq->lock);
        rq->nr_running = 0;
        rq->calc_load_active = 0;
        rq->calc_load_update = jiffies + LOAD_FREQ;
        INIT_LIST_HEAD(&rq->cfs_tasks);
        init_cfs_rq(&rq->cfs);
        init_rt_rq(&rq->rt);
        init_dl_rq(&rq->dl);
#ifdef CONFIG_FAIR_GROUP_SCHED
        rq->cfs.root = &rq->cfs;
#endif
    }

    // 设置当前执行上下文为idle任务
    current->sched_class = &idle_sched_class;

    // 初始化idle线程的调度实体
    init_task_rq(&init_task, 0);
}

// rq(per-CPU runqueue)的数据结构
struct rq {
    raw_spinlock_t      lock;          // 运行队列锁
    unsigned int        nr_running;    // 当前运行进程数
    struct cfs_rq       cfs;           // CFS调度子队列
    struct rt_rq        rt;            // 实时调度子队列
    struct dl_rq        dl;            // Deadline调度子队列
    struct task_struct  *curr;         // 当前运行的任务
    struct task_struct  *idle;         // idle任务指针
    struct task_struct  *stop;         // stop任务指针
    u64                 nr_switches;   // 上下文切换计数
    unsigned long       nr_uninterruptible; // D状态进程数
};
```

CFS(完全公平调度器,Completely Fair Scheduler)是大多数进程使用的默认调度类。它的初始化在sched_init中通过init_cfs_rq完成:

```c
// kernel/sched/fair.c
void __init init_cfs_rq(struct cfs_rq *cfs_rq)
{
    // 初始化CFS红黑树
    cfs_rq->tasks_timeline = RB_ROOT_CACHED;
    cfs_rq->min_vruntime = (u64)(-(1LL << 20));
    cfs_rq->nr_running = 0;
    cfs_rq->h_nr_running = 0;

    // 初始化CFS统计信息
    cfs_rq->exec_clock = 0;
    cfs_rq->load.weight = NICE_0_LOAD;
    cfs_rq->load.inv_weight = 0;

#ifdef CONFIG_FAIR_GROUP_SCHED
    // 在组调度场景下设置根CFS RQ
    cfs_rq->tg = &root_task_group;
    root_task_group.cfs_rq[cpu] = cfs_rq;
#endif

    // 初始化PELT(每个实体负载跟踪)
    memset(&cfs_rq->avg, 0, sizeof(cfs_rq->avg));
}

// CFS的就绪队列关键字段
struct cfs_rq {
    struct rb_root_cached   tasks_timeline;
    struct sched_entity     *curr;
    struct sched_entity     *next;
    struct sched_entity     *last;
    unsigned int            nr_running;
    unsigned int            h_nr_running;
    u64                     min_vruntime;
    struct load_weight      load;
    struct sched_statistics statistics;
};
```

idle线程的创建是sched_init中最关键的一步。在x86_64上,idle线程实际上就是起始阶段的init_task。内核通过以下方式建立idle上下文:

```c
// kernel/sched/core.c
void __init sched_init(void)
{
    // 将init_task标记为idle线程
    // init_task是静态定义的task_struct实例
    init_task.__state = 0;
    init_task.flags |= PF_IDLE;
    init_task.prio = MAX_PRIO - 20;
    init_task.static_prio = MAX_PRIO - 20;
    init_task.normal_prio = MAX_PRIO - 20;
    init_task.rt_priority = 0;

    // init_task的调度策略为SCHED_NORMAL
    init_task.policy = SCHED_NORMAL;

    // 初始化调度类的具体实体
    // CFS调度实体
    init_task.se.vruntime = 0;
    init_task.se.sum_exec_runtime = 0;
    init_task.se.prev_sum_exec_runtime = 0;
    init_task.se.nr_migrations = 0;
    init_task.se.exec_start = 0;

    // 实时调度实体(不参与RT调度)
    init_task.rt.timeout = 0;
    init_task.rt.time_slice = 0;

    // 将init_task关联到CPU0的rq
    cpu_rq(0)->idle = &init_task;
    cpu_rq(0)->curr = &init_task;
    cpu_rq(0)->idle->sched_class = &idle_sched_class;

    // 初始化current指针
    this_cpu_write(current_task, &init_task);
}
```

idle_sched_class是一个特殊的调度类,它只在rq中没有其他可运行进程时被选中:

```c
// kernel/sched/idle.c
DEFINE_SCHED_CLASS(idle) = {
    .enqueue_task         = enqueue_task_idle,
    .dequeue_task         = dequeue_task_idle,
    .check_preempt_curr   = check_preempt_curr_idle,
    .pick_next_task       = pick_next_task_idle,
    .put_prev_task        = put_prev_task_idle,
    .set_curr_task        = set_curr_task_idle,
    .task_tick            = task_tick_idle,
    .balance             = balance_idle,
};

// idle调度类的pick_next_task实现
static struct task_struct *
pick_next_task_idle(struct rq *rq)
{
    // 直接返回rq的idle指针
    // idle任务总可以被"选中"
    return rq->idle;
}

// idle线程的主循环
static void do_idle(void)
{
    int cpu = smp_processor_id();

    // 检查是否需要重新调度
    for (;;) {
        tick_nohz_idle_stop_tick();

        while (!need_resched()) {
            // 执行CPU idle指令
            // x86上使用hlt/mwait
            arch_cpu_idle_enter();
            if (!need_resched()) {
                // 真实的idle操作
                // 调用mwait或hlt降低功耗
                default_idle_call();
            }
            arch_cpu_idle_exit();
        }

        // 有进程需要调度,退出idle
        tick_nohz_idle_exit();
        preempt_enable_no_resched();
        schedule_preempt_disabled();
    }
}
```

sched_init中另一个关键组件是负载均衡数据结构的初始化:

```c
// kernel/sched/topology.c
void __init sched_init_domains(const struct cpumask *cpu_map)
{
    int i;

    // 初始化调度域拓扑
    // 调度域分为:
    // SD_LEVEL_SIBLING: 同一物理CPU的兄弟核心
    // SD_LEVEL_MC: 多核心
    // SD_LEVEL_NUMA: NUMA节点间
    // SD_LEVEL_NODE: 跨NUMA节点

    // 当前只有一个CPU(BSP),建立最小的调度域
    for_each_possible_cpu(i) {
        if (cpumask_test_cpu(i, cpu_map)) {
            // 创建该CPU的调度域层级
            struct sched_domain *sd;
            sd = build_sched_domain(NULL, i);
            rcu_assign_pointer(per_cpu(sd_domains, i), sd);
        }
    }
}

// 调度域构建
struct sched_domain *build_sched_domain(struct sched_domain_topology_level *tl,
                                        int cpu)
{
    struct sched_domain *sd = NULL;
    struct sched_domain *parent = NULL;

    // 自底向上构建调度域
    for (; tl; tl = tl->next) {
        sd = sd_init(tl, cpu);
        sd->parent = parent;
        if (parent)
            parent->child = sd;
        parent = sd;
    }

    return sd;
}
```

sched_init完成后,调度器可以支持最基本的进程调度。此时init_task作为idle线程运行在CPU0上。后续rest_init中kernel_init完成用户空间初始化后,idle线程在CPU空闲时由调度器自动选择执行do_idle循环,通过hlt/mwait指令降低CPU功耗。
懊尚握傲谪舶盘导缚善俣裁帘督檬

m.qzk.qtbzn.cn/33719.Doc
m.qzk.qtbzn.cn/86080.Doc
m.qzk.qtbzn.cn/82240.Doc
m.qzk.qtbzn.cn/44440.Doc
m.qzk.qtbzn.cn/24840.Doc
m.qzk.qtbzn.cn/86428.Doc
m.qzk.qtbzn.cn/20424.Doc
m.qzk.qtbzn.cn/55399.Doc
m.qzk.qtbzn.cn/24208.Doc
m.qzk.qtbzn.cn/68488.Doc
m.qzk.qtbzn.cn/13555.Doc
m.qzk.qtbzn.cn/57339.Doc
m.qzj.qtbzn.cn/53337.Doc
m.qzj.qtbzn.cn/44462.Doc
m.qzj.qtbzn.cn/60808.Doc
m.qzj.qtbzn.cn/68268.Doc
m.qzj.qtbzn.cn/55997.Doc
m.qzj.qtbzn.cn/44464.Doc
m.qzj.qtbzn.cn/86840.Doc
m.qzj.qtbzn.cn/24000.Doc
m.qzj.qtbzn.cn/02200.Doc
m.qzj.qtbzn.cn/86800.Doc
m.qzj.qtbzn.cn/93371.Doc
m.qzj.qtbzn.cn/19315.Doc
m.qzj.qtbzn.cn/42280.Doc
m.qzj.qtbzn.cn/66848.Doc
m.qzj.qtbzn.cn/62286.Doc
m.qzj.qtbzn.cn/80842.Doc
m.qzj.qtbzn.cn/53571.Doc
m.qzj.qtbzn.cn/48642.Doc
m.qzj.qtbzn.cn/44686.Doc
m.qzj.qtbzn.cn/62864.Doc
m.qzh.qtbzn.cn/04066.Doc
m.qzh.qtbzn.cn/44022.Doc
m.qzh.qtbzn.cn/08862.Doc
m.qzh.qtbzn.cn/46664.Doc
m.qzh.qtbzn.cn/39715.Doc
m.qzh.qtbzn.cn/08628.Doc
m.qzh.qtbzn.cn/33977.Doc
m.qzh.qtbzn.cn/28020.Doc
m.qzh.qtbzn.cn/93355.Doc
m.qzh.qtbzn.cn/64268.Doc
m.qzh.qtbzn.cn/88626.Doc
m.qzh.qtbzn.cn/84842.Doc
m.qzh.qtbzn.cn/20000.Doc
m.qzh.qtbzn.cn/75375.Doc
m.qzh.qtbzn.cn/68046.Doc
m.qzh.qtbzn.cn/20642.Doc
m.qzh.qtbzn.cn/68802.Doc
m.qzh.qtbzn.cn/20022.Doc
m.qzh.qtbzn.cn/42646.Doc
m.qzh.qtbzn.cn/68042.Doc
m.qzg.qtbzn.cn/42800.Doc
m.qzg.qtbzn.cn/20048.Doc
m.qzg.qtbzn.cn/31979.Doc
m.qzg.qtbzn.cn/71371.Doc
m.qzg.qtbzn.cn/46044.Doc
m.qzg.qtbzn.cn/84868.Doc
m.qzg.qtbzn.cn/79515.Doc
m.qzg.qtbzn.cn/40040.Doc
m.qzg.qtbzn.cn/37939.Doc
m.qzg.qtbzn.cn/08442.Doc
m.qzg.qtbzn.cn/99955.Doc
m.qzg.qtbzn.cn/40604.Doc
m.qzg.qtbzn.cn/64226.Doc
m.qzg.qtbzn.cn/79331.Doc
m.qzg.qtbzn.cn/00060.Doc
m.qzg.qtbzn.cn/46824.Doc
m.qzg.qtbzn.cn/60286.Doc
m.qzg.qtbzn.cn/44488.Doc
m.qzg.qtbzn.cn/26000.Doc
m.qzg.qtbzn.cn/40662.Doc
m.qzf.qtbzn.cn/22008.Doc
m.qzf.qtbzn.cn/84000.Doc
m.qzf.qtbzn.cn/20288.Doc
m.qzf.qtbzn.cn/62228.Doc
m.qzf.qtbzn.cn/35739.Doc
m.qzf.qtbzn.cn/04288.Doc
m.qzf.qtbzn.cn/88862.Doc
m.qzf.qtbzn.cn/24484.Doc
m.qzf.qtbzn.cn/48840.Doc
m.qzf.qtbzn.cn/02042.Doc
m.qzf.qtbzn.cn/55793.Doc
m.qzf.qtbzn.cn/93771.Doc
m.qzf.qtbzn.cn/39339.Doc
m.qzf.qtbzn.cn/40020.Doc
m.qzf.qtbzn.cn/82244.Doc
m.qzf.qtbzn.cn/48608.Doc
m.qzf.qtbzn.cn/19175.Doc
m.qzf.qtbzn.cn/24826.Doc
m.qzf.qtbzn.cn/60640.Doc
m.qzf.qtbzn.cn/80422.Doc
m.qzd.qtbzn.cn/02440.Doc
m.qzd.qtbzn.cn/00046.Doc
m.qzd.qtbzn.cn/20202.Doc
m.qzd.qtbzn.cn/15931.Doc
m.qzd.qtbzn.cn/22664.Doc
m.qzd.qtbzn.cn/42628.Doc
m.qzd.qtbzn.cn/44262.Doc
m.qzd.qtbzn.cn/60482.Doc
m.qzd.qtbzn.cn/84620.Doc
m.qzd.qtbzn.cn/79975.Doc
m.qzd.qtbzn.cn/28806.Doc
m.qzd.qtbzn.cn/28002.Doc
m.qzd.qtbzn.cn/73357.Doc
m.qzd.qtbzn.cn/84608.Doc
m.qzd.qtbzn.cn/60206.Doc
m.qzd.qtbzn.cn/80446.Doc
m.qzd.qtbzn.cn/17791.Doc
m.qzd.qtbzn.cn/22000.Doc
m.qzd.qtbzn.cn/82628.Doc
m.qzd.qtbzn.cn/68066.Doc
m.qzs.qtbzn.cn/13193.Doc
m.qzs.qtbzn.cn/02022.Doc
m.qzs.qtbzn.cn/08882.Doc
m.qzs.qtbzn.cn/22666.Doc
m.qzs.qtbzn.cn/00424.Doc
m.qzs.qtbzn.cn/17351.Doc
m.qzs.qtbzn.cn/60280.Doc
m.qzs.qtbzn.cn/20228.Doc
m.qzs.qtbzn.cn/42082.Doc
m.qzs.qtbzn.cn/02808.Doc
m.qzs.qtbzn.cn/35995.Doc
m.qzs.qtbzn.cn/73759.Doc
m.qzs.qtbzn.cn/24022.Doc
m.qzs.qtbzn.cn/26046.Doc
m.qzs.qtbzn.cn/62202.Doc
m.qzs.qtbzn.cn/66460.Doc
m.qzs.qtbzn.cn/37735.Doc
m.qzs.qtbzn.cn/46440.Doc
m.qzs.qtbzn.cn/64206.Doc
m.qzs.qtbzn.cn/79733.Doc
m.qza.qtbzn.cn/40866.Doc
m.qza.qtbzn.cn/88202.Doc
m.qza.qtbzn.cn/97193.Doc
m.qza.qtbzn.cn/20606.Doc
m.qza.qtbzn.cn/00226.Doc
m.qza.qtbzn.cn/88244.Doc
m.qza.qtbzn.cn/46680.Doc
m.qza.qtbzn.cn/42648.Doc
m.qza.qtbzn.cn/77957.Doc
m.qza.qtbzn.cn/48628.Doc
m.qza.qtbzn.cn/82202.Doc
m.qza.qtbzn.cn/44628.Doc
m.qza.qtbzn.cn/46464.Doc
m.qza.qtbzn.cn/86606.Doc
m.qza.qtbzn.cn/68804.Doc
m.qza.qtbzn.cn/02402.Doc
m.qza.qtbzn.cn/39733.Doc
m.qza.qtbzn.cn/00800.Doc
m.qza.qtbzn.cn/42468.Doc
m.qza.qtbzn.cn/60622.Doc
m.qzp.qtbzn.cn/84260.Doc
m.qzp.qtbzn.cn/77393.Doc
m.qzp.qtbzn.cn/15575.Doc
m.qzp.qtbzn.cn/42024.Doc
m.qzp.qtbzn.cn/22006.Doc
m.qzp.qtbzn.cn/66266.Doc
m.qzp.qtbzn.cn/08400.Doc
m.qzp.qtbzn.cn/40404.Doc
m.qzp.qtbzn.cn/00202.Doc
m.qzp.qtbzn.cn/48004.Doc
m.qzp.qtbzn.cn/95797.Doc
m.qzp.qtbzn.cn/82848.Doc
m.qzp.qtbzn.cn/02680.Doc
m.qzp.qtbzn.cn/82042.Doc
m.qzp.qtbzn.cn/88864.Doc
m.qzp.qtbzn.cn/04644.Doc
m.qzp.qtbzn.cn/33397.Doc
m.qzp.qtbzn.cn/71199.Doc
m.qzp.qtbzn.cn/42488.Doc
m.qzp.qtbzn.cn/88082.Doc
m.qzo.qtbzn.cn/66222.Doc
m.qzo.qtbzn.cn/08208.Doc
m.qzo.qtbzn.cn/71353.Doc
m.qzo.qtbzn.cn/64422.Doc
m.qzo.qtbzn.cn/13317.Doc
m.qzo.qtbzn.cn/40208.Doc
m.qzo.qtbzn.cn/11935.Doc
m.qzo.qtbzn.cn/37917.Doc
m.qzo.qtbzn.cn/04424.Doc
m.qzo.qtbzn.cn/00880.Doc
m.qzo.qtbzn.cn/06228.Doc
m.qzo.qtbzn.cn/40660.Doc
m.qzo.qtbzn.cn/64628.Doc
m.qzo.qtbzn.cn/26602.Doc
m.qzo.qtbzn.cn/06688.Doc
m.qzo.qtbzn.cn/68686.Doc
m.qzo.qtbzn.cn/22682.Doc
m.qzo.qtbzn.cn/46624.Doc
m.qzo.qtbzn.cn/04420.Doc
m.qzo.qtbzn.cn/17133.Doc
m.qzi.qtbzn.cn/44222.Doc
m.qzi.qtbzn.cn/24668.Doc
m.qzi.qtbzn.cn/55173.Doc
m.qzi.qtbzn.cn/24026.Doc
m.qzi.qtbzn.cn/20206.Doc
m.qzi.qtbzn.cn/15155.Doc
m.qzi.qtbzn.cn/40246.Doc
m.qzi.qtbzn.cn/20224.Doc
m.qzi.qtbzn.cn/62080.Doc
m.qzi.qtbzn.cn/28800.Doc
m.qzi.qtbzn.cn/59139.Doc
m.qzi.qtbzn.cn/84682.Doc
m.qzi.qtbzn.cn/82060.Doc
m.qzi.qtbzn.cn/48000.Doc
m.qzi.qtbzn.cn/66624.Doc
m.qzi.qtbzn.cn/11359.Doc
m.qzi.qtbzn.cn/11391.Doc
m.qzi.qtbzn.cn/73553.Doc
m.qzi.qtbzn.cn/59199.Doc
m.qzi.qtbzn.cn/46024.Doc
m.qzu.qtbzn.cn/42028.Doc
m.qzu.qtbzn.cn/51337.Doc
m.qzu.qtbzn.cn/00480.Doc
m.qzu.qtbzn.cn/86482.Doc
m.qzu.qtbzn.cn/73795.Doc
m.qzu.qtbzn.cn/97959.Doc
m.qzu.qtbzn.cn/86648.Doc
m.qzu.qtbzn.cn/66022.Doc
m.qzu.qtbzn.cn/11511.Doc
m.qzu.qtbzn.cn/82622.Doc
m.qzu.qtbzn.cn/88226.Doc
m.qzu.qtbzn.cn/80444.Doc
m.qzu.qtbzn.cn/02022.Doc
m.qzu.qtbzn.cn/64088.Doc
m.qzu.qtbzn.cn/00440.Doc
m.qzu.qtbzn.cn/66246.Doc
m.qzu.qtbzn.cn/57317.Doc
m.qzu.qtbzn.cn/48806.Doc
m.qzu.qtbzn.cn/44002.Doc
m.qzu.qtbzn.cn/66422.Doc
m.qzy.qtbzn.cn/53971.Doc
m.qzy.qtbzn.cn/68088.Doc
m.qzy.qtbzn.cn/97553.Doc
m.qzy.qtbzn.cn/73195.Doc
m.qzy.qtbzn.cn/04880.Doc
m.qzy.qtbzn.cn/77591.Doc
m.qzy.qtbzn.cn/66608.Doc
m.qzy.qtbzn.cn/04402.Doc
m.qzy.qtbzn.cn/35515.Doc
m.qzy.qtbzn.cn/40466.Doc
m.qzy.qtbzn.cn/60684.Doc
m.qzy.qtbzn.cn/22080.Doc
m.qzy.qtbzn.cn/06844.Doc
m.qzy.qtbzn.cn/77115.Doc
m.qzy.qtbzn.cn/84406.Doc
m.qzy.qtbzn.cn/55953.Doc
m.qzy.qtbzn.cn/39375.Doc
m.qzy.qtbzn.cn/88600.Doc
m.qzy.qtbzn.cn/06680.Doc
m.qzy.qtbzn.cn/82626.Doc
m.qzt.qtbzn.cn/26464.Doc
m.qzt.qtbzn.cn/95315.Doc
m.qzt.qtbzn.cn/77193.Doc
m.qzt.qtbzn.cn/02244.Doc
m.qzt.qtbzn.cn/68826.Doc
m.qzt.qtbzn.cn/62808.Doc
m.qzt.qtbzn.cn/24280.Doc
m.qzt.qtbzn.cn/40462.Doc
m.qzt.qtbzn.cn/42646.Doc
m.qzt.qtbzn.cn/86822.Doc
m.qzt.qtbzn.cn/66800.Doc
m.qzt.qtbzn.cn/00424.Doc
m.qzt.qtbzn.cn/88822.Doc
m.qzt.qtbzn.cn/73197.Doc
m.qzt.qtbzn.cn/62424.Doc
m.qzt.qtbzn.cn/42260.Doc
m.qzt.qtbzn.cn/88686.Doc
m.qzt.qtbzn.cn/60008.Doc
m.qzt.qtbzn.cn/00426.Doc
m.qzt.qtbzn.cn/08826.Doc
m.qzr.qtbzn.cn/46286.Doc
m.qzr.qtbzn.cn/00660.Doc
m.qzr.qtbzn.cn/28248.Doc
m.qzr.qtbzn.cn/44868.Doc
m.qzr.qtbzn.cn/66664.Doc
m.qzr.qtbzn.cn/40022.Doc
m.qzr.qtbzn.cn/46024.Doc
m.qzr.qtbzn.cn/20884.Doc
m.qzr.qtbzn.cn/82468.Doc
m.qzr.qtbzn.cn/91951.Doc
m.qzr.qtbzn.cn/22826.Doc
m.qzr.qtbzn.cn/24006.Doc
m.qzr.qtbzn.cn/24880.Doc
m.qzr.qtbzn.cn/35779.Doc
m.qzr.qtbzn.cn/02448.Doc
m.qzr.qtbzn.cn/80802.Doc
m.qzr.qtbzn.cn/55793.Doc
m.qzr.qtbzn.cn/39711.Doc
m.qzr.qtbzn.cn/88206.Doc
m.qzr.qtbzn.cn/42248.Doc
m.qze.qtbzn.cn/60446.Doc
m.qze.qtbzn.cn/00606.Doc
m.qze.qtbzn.cn/84046.Doc
m.qze.qtbzn.cn/57195.Doc
m.qze.qtbzn.cn/44086.Doc
m.qze.qtbzn.cn/26462.Doc
m.qze.qtbzn.cn/60082.Doc
m.qze.qtbzn.cn/62286.Doc
m.qze.qtbzn.cn/08866.Doc
m.qze.qtbzn.cn/82826.Doc
m.qze.qtbzn.cn/20268.Doc
m.qze.qtbzn.cn/66648.Doc
m.qze.qtbzn.cn/15375.Doc
m.qze.qtbzn.cn/51717.Doc
m.qze.qtbzn.cn/80280.Doc
m.qze.qtbzn.cn/08600.Doc
m.qze.qtbzn.cn/08466.Doc
m.qze.qtbzn.cn/35933.Doc
m.qze.qtbzn.cn/26440.Doc
m.qze.qtbzn.cn/35159.Doc
m.qzw.qtbzn.cn/04402.Doc
m.qzw.qtbzn.cn/28806.Doc
m.qzw.qtbzn.cn/59955.Doc
m.qzw.qtbzn.cn/31979.Doc
m.qzw.qtbzn.cn/51915.Doc
m.qzw.qtbzn.cn/60022.Doc
m.qzw.qtbzn.cn/95371.Doc
m.qzw.qtbzn.cn/20684.Doc
m.qzw.qtbzn.cn/24886.Doc
m.qzw.qtbzn.cn/51317.Doc
m.qzw.qtbzn.cn/35351.Doc
m.qzw.qtbzn.cn/24486.Doc
m.qzw.qtbzn.cn/88624.Doc
m.qzw.qtbzn.cn/06442.Doc
m.qzw.qtbzn.cn/82864.Doc
m.qzw.qtbzn.cn/28482.Doc
m.qzw.qtbzn.cn/28044.Doc
m.qzw.qtbzn.cn/37177.Doc
m.qzw.qtbzn.cn/22082.Doc
m.qzw.qtbzn.cn/24620.Doc
m.qzq.qtbzn.cn/22402.Doc
m.qzq.qtbzn.cn/44402.Doc
m.qzq.qtbzn.cn/19773.Doc
m.qzq.qtbzn.cn/79991.Doc
m.qzq.qtbzn.cn/62464.Doc
m.qzq.qtbzn.cn/88080.Doc
m.qzq.qtbzn.cn/22622.Doc
m.qzq.qtbzn.cn/11195.Doc
m.qzq.qtbzn.cn/62206.Doc
m.qzq.qtbzn.cn/42206.Doc
m.qzq.qtbzn.cn/64884.Doc
m.qzq.qtbzn.cn/00648.Doc
m.qzq.qtbzn.cn/64206.Doc
m.qzq.qtbzn.cn/51539.Doc
m.qzq.qtbzn.cn/82620.Doc
m.qzq.qtbzn.cn/88264.Doc
m.qzq.qtbzn.cn/02046.Doc
m.qzq.qtbzn.cn/66622.Doc
m.qzq.qtbzn.cn/55117.Doc
m.qzq.qtbzn.cn/48486.Doc
m.qlm.qtbzn.cn/80024.Doc
m.qlm.qtbzn.cn/40026.Doc
m.qlm.qtbzn.cn/97995.Doc
m.qlm.qtbzn.cn/86462.Doc
m.qlm.qtbzn.cn/57131.Doc
m.qlm.qtbzn.cn/24824.Doc
m.qlm.qtbzn.cn/40408.Doc
m.qlm.qtbzn.cn/93339.Doc
m.qlm.qtbzn.cn/19393.Doc
m.qlm.qtbzn.cn/08846.Doc
m.qlm.qtbzn.cn/17379.Doc
m.qlm.qtbzn.cn/08626.Doc
m.qlm.qtbzn.cn/06824.Doc
m.qlm.qtbzn.cn/26244.Doc
m.qlm.qtbzn.cn/66606.Doc
m.qlm.qtbzn.cn/97717.Doc
m.qlm.qtbzn.cn/40044.Doc
m.qlm.qtbzn.cn/82648.Doc
m.qlm.qtbzn.cn/02606.Doc
m.qlm.qtbzn.cn/80888.Doc
m.qln.qtbzn.cn/46660.Doc
m.qln.qtbzn.cn/40466.Doc
m.qln.qtbzn.cn/42480.Doc
m.qln.qtbzn.cn/17935.Doc
m.qln.qtbzn.cn/97111.Doc
m.qln.qtbzn.cn/66482.Doc
m.qln.qtbzn.cn/93991.Doc
m.qln.qtbzn.cn/86006.Doc
m.qln.qtbzn.cn/68044.Doc
m.qln.qtbzn.cn/31153.Doc
m.qln.qtbzn.cn/48220.Doc
m.qln.qtbzn.cn/53737.Doc
m.qln.qtbzn.cn/84240.Doc
m.qln.qtbzn.cn/46482.Doc
m.qln.qtbzn.cn/86028.Doc
m.qln.qtbzn.cn/22866.Doc
m.qln.qtbzn.cn/00660.Doc
m.qln.qtbzn.cn/42400.Doc
m.qln.qtbzn.cn/42624.Doc
m.qln.qtbzn.cn/06460.Doc
m.qlb.qtbzn.cn/39539.Doc
m.qlb.qtbzn.cn/02442.Doc
m.qlb.qtbzn.cn/79915.Doc
