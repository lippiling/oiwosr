兰沸湍位夷


Linux sched_idle空闲调度类与idle进程周期

idle_sched_class是Linux内核优先级最低的调度类，位于stop_sched_class之后、完全公平调度类之前。其唯一任务（per-CPU的idle线程，pid=0）在runqueue无其他可运行任务时被pick_next_task_idle选中。idle线程的prio为MAX_PRIO（140），在prio chaining中永远不会与正常任务竞争——它的存在完全是为了让CPU在无事可做时执行WFI/HLT指令降低功耗。

```c
DEFINE_SCHED_CLASS(idle) = {
    .enqueue_task          = enqueue_task_idle,
    .dequeue_task          = dequeue_task_idle,
    .yield_task            = yield_task_idle,
    .check_preempt_curr    = check_preempt_curr_idle,
    .pick_next_task        = pick_next_task_idle,
    .put_prev_task         = put_prev_task_idle,
#ifdef CONFIG_SMP
    .select_task_rq        = select_task_rq_idle,
    .set_cpus_allowed      = set_cpus_allowed_idle,
#endif
    .task_tick             = task_tick_idle,
    .priority_class        = 1,  /* 倒数第二低，仅高于stop_class */
};
```

pick_next_task_idle()直接返回rq->idle——即当前CPU的idle线程task_struct。该线程在sched_init()阶段通过init_idle()初始化，其thread_struct.sp直接指向当前CPU的内核栈底部，thread_info.preempt_count设置为PREEMPT_ACTIVE防止idle状态被抢占。

```c
static struct task_struct *pick_next_task_idle(struct rq *rq)
{
    rq->idle_balance = 0;  /* 清零，idle线程不参与load balance */
    return rq->idle;
}

void init_idle(struct task_struct *idle, int cpu)
{
    struct rq *rq = cpu_rq(cpu);

    __sched_fork(0, idle);
    raw_spin_lock_irqsave(&rq->lock, flags);
    idle->__state = TASK_RUNNING;
    idle->se.exec_start = sched_clock();
    idle->flags |= PF_IDLE;

    kasan_unpoison_task_stack(idle);
    /*
     * idle线程的调度实体不加入任何cfs_rq或rt_rq
     * 但preempt_count必须设为PREEMPT_ACTIVE以阻止
     * schedule()在idle线程内部再次被调用
     */
#ifdef CONFIG_PREEMPT
    idle->thread_info.preempt_count = PREEMPT_ACTIVE;
#endif

    rq->idle = idle;
    rq->curr = idle;

    raw_spin_unlock_irqrestore(&rq->lock, flags);
}
```

idle线程的主循环位于cpu_startup_entry()中，由boot CPU在rest_init()中启动，从属CPU在secondary_startup_64中跳转。该函数调用do_idle()，其内部是一个无限循环：调用cpuidle_select选择C-state、调用cpuidle_enter进入、退出后检查TIF_NEED_RESCHED。

```c
static void do_idle(void)
{
    int cpu = smp_processor_id();

    /*
     * 检查是否有pending的TTWU queue
     * 如果有其他CPU已经将task加入当前CPU的runqueue，
     * 则不应进入idle
     */
    if (ttwu_pending())
        return;

    /* 进入cpuidle框架选择最深C-state */
    cpuidle_select(drv, dev, &stop_tick);

    /* tick停止处理——NO_HZ路径 */
    tick_nohz_idle_enter();

    while (!need_resched()) {
        /* 进入实际idle状态 */
        cpuidle_enter(drv, dev, next_state);

        /* 退出后检查是否需要重新计算C-state */
        if (cpuidle_need_update(cpu))
            cpuidle_reflect(dev, next_state);
    }

    tick_nohz_idle_exit();
    /*
     * 退出idle后立即调用schedule_preempt_disabled()
     * 将当前线程从rq->idle切回真正的任务
     */
    sched_preempt_enable_no_resched();
    schedule_preempt_disabled();
}
```

tick_nohz_idle_enter()是idle周期中的关键路径，决定当前CPU是否停止周期性tick。若整个系统没有足够的周期性工作负载（即只有当前task_struct和timer列表可以延期），tick_nohz_stop_tick()将dev->tick_stopped置1。NO_HZ全停止后，cpu必须依靠外部中断（网卡、timer_alarm）来唤醒——如果后续没有外部中断到达，cpu会无限期停留在idle状态。这也是为什么rcu_needs_cpu()必须返回true来阻止tick停止，否则RCU callbacks会因饥饿而触发RCU stall warning。

```c
select: bool tick_nohz_stop_tick(struct tick_sched *ts, int cpu)
{
    struct clock_event_device *dev = __this_cpu_read(tick_cpu_device.evtdev);
    unsigned long base_jiffies;
    u64 expires;

    /* 计算下一个定时器到期时间 */
    expires = tick_nohz_next_event(ts, cpu);

    /* 如果expires离当前距离太近（< 1 tick），不停止tick */
    if (expires - basemono < TICK_NSEC)
        return false;

    /* 检查RCU是否需要这个CPU */
    if (!rcu_needs_cpu() && cpu_online(cpu)) {
        ts->tick_stopped = 1;
        ts->idle_jiffies = base_jiffies;
        dev->next_event = KTIME_MAX;
        return true;
    }

    return false;
}
```

SCHED_IDLE优先级策略（通过SCHED_IDLE调度策略设置，非idle调度类）与idle_sched_class有本质区别。SCHED_IDLE策略的任务仍然属于CFS调度类，但其weight通过task_struct->se.load.weight = 3（普通任务1024）被压缩到极低。这意味着SCHED_IDLE任务的vruntime增长极快，在CFS红黑树中几乎总是被推到最右端，只在所有其他CFS任务都阻塞时才会被选中。而idle_sched_class是独立调度类，所有非idle调度类都无法从runqueue中找到任务时才会轮询到idle类。

```c
static void set_load_weight(struct task_struct *p, bool update_load)
{
    int prio = p->static_prio - MAX_RT_PRIO;
    struct load_weight *load = &p->se.load;

    if (idle_policy(p->policy)) {
        /* SCHED_IDLE策略：weight设为最小 */
        load->weight = scale_load(WEIGHT_IDLEPRIO);
        load->inv_weight = WMULT_IDLEPRIO;
        return;
    }

    load->weight = scale_load(sched_prio_to_weight[prio]);
    load->inv_weight = sched_prio_to_wmult[prio];
}
```

idle进程的调度决策本身不会通过scheduler_tick触发，因为task_tick_idle()实现为空函数。但是scheduler_tick在rq->curr == rq->idle时仍会执行update_rq_clock和calc_global_load_tick——这意味着即使CPU空闲，全局tick负载计算仍在进行。这部分开销在NO_HZ_FULL模式下被避免：当单个任务独占CPU时tick_stopped后calc_load_nohz_start/stop管理全局负载汇总。

一个边界case是nohz_full隔离CPU上的idle行为。当isolcpus或nohz_full将CPU从通用调度域排除时，该CPU的idle线程在do_idle()循环中不受scheduler_tick打扰，但必须处理per-CPU的arch_timer中断。如果wakeup事件发生在其他CPU上，通过llist的TTWU queue提交给目标CPU时，目标CPU必须通过arch_send_call_function_single_ipi()唤醒——但如果目标CPU处于mwait cstate超过1的深度睡眠，IPI可能无法及时到达。Intel的mwait_monitor/hint机制通过MONITOR/MWAIT指令对address monitoring和store detection来避免这个问题：idle线程在进入deep C-state前用MONITOR监视runqueue的__ttwu_pending标志所在地址，当其他CPU写入该标志时硬件自动唤醒。

```c
static inline void mwait_idle_with_hints(unsigned long ax, unsigned long cx)
{
    if (!current_set_polling_and_test()) {
        if (this_cpu_has(X86_FEATURE_CLFLUSH_MONITOR)) {
            mb();
            __monitor((void *)&current_thread_info()->flags, 0, 0);
            if (!need_resched())
                __mwait(ax, cx);
        }
    }
    current_clr_polling();
}
```

最后，idle线程的退出路径存在一个条件竞态：do_idle()中检查need_resched()后调用schedule_preempt_disabled()，但此时若有IRQ在检查窗口和schedule之间触发并set_tsk_need_resched，两次need_resched都不会丢失。但在CONFIG_PREEMPT_NONE下schedule()的入口preempt_disable_notrace是barrier()，无法阻止IRQ在中断返回时再次调用schedule()——从而产生__schedule()重入。为此schedule()入口通过schedule_debug(prev)校验in_sched_functions()来捕获并panic重入。
椿读囱以局站热航晕裂尾糜盗守目

tv.blog.xlruof.cn/Article/details/244848.sHtML
tv.blog.xlruof.cn/Article/details/428820.sHtML
tv.blog.xlruof.cn/Article/details/062444.sHtML
tv.blog.xlruof.cn/Article/details/208000.sHtML
tv.blog.xlruof.cn/Article/details/282428.sHtML
tv.blog.xlruof.cn/Article/details/482240.sHtML
tv.blog.xlruof.cn/Article/details/684402.sHtML
tv.blog.xlruof.cn/Article/details/800004.sHtML
tv.blog.xlruof.cn/Article/details/264268.sHtML
tv.blog.xlruof.cn/Article/details/060462.sHtML
tv.blog.xlruof.cn/Article/details/246202.sHtML
tv.blog.xlruof.cn/Article/details/846602.sHtML
tv.blog.xlruof.cn/Article/details/331797.sHtML
tv.blog.xlruof.cn/Article/details/880082.sHtML
tv.blog.xlruof.cn/Article/details/060882.sHtML
tv.blog.xlruof.cn/Article/details/335735.sHtML
tv.blog.xlruof.cn/Article/details/371995.sHtML
tv.blog.xlruof.cn/Article/details/064280.sHtML
tv.blog.xlruof.cn/Article/details/735397.sHtML
tv.blog.xlruof.cn/Article/details/468620.sHtML
tv.blog.xlruof.cn/Article/details/288426.sHtML
tv.blog.xlruof.cn/Article/details/266448.sHtML
tv.blog.xlruof.cn/Article/details/622486.sHtML
tv.blog.xlruof.cn/Article/details/842482.sHtML
tv.blog.xlruof.cn/Article/details/179911.sHtML
tv.blog.xlruof.cn/Article/details/028064.sHtML
tv.blog.xlruof.cn/Article/details/084626.sHtML
tv.blog.xlruof.cn/Article/details/868248.sHtML
tv.blog.xlruof.cn/Article/details/068242.sHtML
tv.blog.xlruof.cn/Article/details/800424.sHtML
tv.blog.xlruof.cn/Article/details/026288.sHtML
tv.blog.xlruof.cn/Article/details/620648.sHtML
tv.blog.xlruof.cn/Article/details/355993.sHtML
tv.blog.xlruof.cn/Article/details/600846.sHtML
tv.blog.xlruof.cn/Article/details/537533.sHtML
tv.blog.xlruof.cn/Article/details/484240.sHtML
tv.blog.xlruof.cn/Article/details/446466.sHtML
tv.blog.xlruof.cn/Article/details/404888.sHtML
tv.blog.xlruof.cn/Article/details/715951.sHtML
tv.blog.xlruof.cn/Article/details/995919.sHtML
tv.blog.xlruof.cn/Article/details/864826.sHtML
tv.blog.xlruof.cn/Article/details/860686.sHtML
tv.blog.xlruof.cn/Article/details/482022.sHtML
tv.blog.xlruof.cn/Article/details/595315.sHtML
tv.blog.xlruof.cn/Article/details/202426.sHtML
tv.blog.xlruof.cn/Article/details/913913.sHtML
tv.blog.xlruof.cn/Article/details/244642.sHtML
tv.blog.xlruof.cn/Article/details/377519.sHtML
tv.blog.xlruof.cn/Article/details/399173.sHtML
tv.blog.xlruof.cn/Article/details/551553.sHtML
tv.blog.xlruof.cn/Article/details/008402.sHtML
tv.blog.xlruof.cn/Article/details/462806.sHtML
tv.blog.xlruof.cn/Article/details/080282.sHtML
tv.blog.xlruof.cn/Article/details/062004.sHtML
tv.blog.xlruof.cn/Article/details/804204.sHtML
tv.blog.xlruof.cn/Article/details/684028.sHtML
tv.blog.xlruof.cn/Article/details/557537.sHtML
tv.blog.xlruof.cn/Article/details/953959.sHtML
tv.blog.xlruof.cn/Article/details/888220.sHtML
tv.blog.xlruof.cn/Article/details/884480.sHtML
tv.blog.xlruof.cn/Article/details/317751.sHtML
tv.blog.xlruof.cn/Article/details/828460.sHtML
tv.blog.xlruof.cn/Article/details/644686.sHtML
tv.blog.xlruof.cn/Article/details/399133.sHtML
tv.blog.xlruof.cn/Article/details/206288.sHtML
tv.blog.xlruof.cn/Article/details/462648.sHtML
tv.blog.xlruof.cn/Article/details/040620.sHtML
tv.blog.xlruof.cn/Article/details/608206.sHtML
tv.blog.xlruof.cn/Article/details/008282.sHtML
tv.blog.xlruof.cn/Article/details/846228.sHtML
tv.blog.xlruof.cn/Article/details/888206.sHtML
tv.blog.xlruof.cn/Article/details/026080.sHtML
tv.blog.xlruof.cn/Article/details/482084.sHtML
tv.blog.xlruof.cn/Article/details/042244.sHtML
tv.blog.xlruof.cn/Article/details/666682.sHtML
tv.blog.xlruof.cn/Article/details/002820.sHtML
tv.blog.xlruof.cn/Article/details/888884.sHtML
tv.blog.xlruof.cn/Article/details/868086.sHtML
tv.blog.xlruof.cn/Article/details/428062.sHtML
tv.blog.xlruof.cn/Article/details/919931.sHtML
tv.blog.xlruof.cn/Article/details/515957.sHtML
tv.blog.xlruof.cn/Article/details/460028.sHtML
tv.blog.xlruof.cn/Article/details/175153.sHtML
tv.blog.xlruof.cn/Article/details/355539.sHtML
tv.blog.xlruof.cn/Article/details/000608.sHtML
tv.blog.xlruof.cn/Article/details/795173.sHtML
tv.blog.xlruof.cn/Article/details/044860.sHtML
tv.blog.xlruof.cn/Article/details/044288.sHtML
tv.blog.xlruof.cn/Article/details/882624.sHtML
tv.blog.xlruof.cn/Article/details/082662.sHtML
tv.blog.xlruof.cn/Article/details/888466.sHtML
tv.blog.xlruof.cn/Article/details/468080.sHtML
tv.blog.xlruof.cn/Article/details/339939.sHtML
tv.blog.xlruof.cn/Article/details/466260.sHtML
tv.blog.xlruof.cn/Article/details/860646.sHtML
tv.blog.xlruof.cn/Article/details/606462.sHtML
tv.blog.xlruof.cn/Article/details/444808.sHtML
tv.blog.xlruof.cn/Article/details/844240.sHtML
tv.blog.xlruof.cn/Article/details/797993.sHtML
tv.blog.xlruof.cn/Article/details/044464.sHtML
tv.blog.xlruof.cn/Article/details/002620.sHtML
tv.blog.xlruof.cn/Article/details/284608.sHtML
tv.blog.xlruof.cn/Article/details/448444.sHtML
tv.blog.xlruof.cn/Article/details/064802.sHtML
tv.blog.xlruof.cn/Article/details/991975.sHtML
tv.blog.xlruof.cn/Article/details/246844.sHtML
tv.blog.xlruof.cn/Article/details/448880.sHtML
tv.blog.xlruof.cn/Article/details/688866.sHtML
tv.blog.xlruof.cn/Article/details/240868.sHtML
tv.blog.xlruof.cn/Article/details/688080.sHtML
tv.blog.xlruof.cn/Article/details/666262.sHtML
tv.blog.xlruof.cn/Article/details/266044.sHtML
tv.blog.xlruof.cn/Article/details/246086.sHtML
tv.blog.xlruof.cn/Article/details/828682.sHtML
tv.blog.xlruof.cn/Article/details/264624.sHtML
tv.blog.xlruof.cn/Article/details/880246.sHtML
tv.blog.xlruof.cn/Article/details/602446.sHtML
tv.blog.xlruof.cn/Article/details/220022.sHtML
tv.blog.xlruof.cn/Article/details/284020.sHtML
tv.blog.xlruof.cn/Article/details/204026.sHtML
tv.blog.xlruof.cn/Article/details/440880.sHtML
tv.blog.xlruof.cn/Article/details/406626.sHtML
tv.blog.xlruof.cn/Article/details/446848.sHtML
tv.blog.xlruof.cn/Article/details/402088.sHtML
tv.blog.xlruof.cn/Article/details/519573.sHtML
tv.blog.xlruof.cn/Article/details/977975.sHtML
tv.blog.xlruof.cn/Article/details/008668.sHtML
tv.blog.xlruof.cn/Article/details/642286.sHtML
tv.blog.xlruof.cn/Article/details/377919.sHtML
tv.blog.xlruof.cn/Article/details/482840.sHtML
tv.blog.xlruof.cn/Article/details/408860.sHtML
tv.blog.xlruof.cn/Article/details/826240.sHtML
tv.blog.xlruof.cn/Article/details/600822.sHtML
tv.blog.xlruof.cn/Article/details/880202.sHtML
tv.blog.xlruof.cn/Article/details/680484.sHtML
tv.blog.xlruof.cn/Article/details/064266.sHtML
tv.blog.xlruof.cn/Article/details/806044.sHtML
tv.blog.xlruof.cn/Article/details/402640.sHtML
tv.blog.xlruof.cn/Article/details/393577.sHtML
tv.blog.xlruof.cn/Article/details/953311.sHtML
tv.blog.xlruof.cn/Article/details/686020.sHtML
tv.blog.xlruof.cn/Article/details/068228.sHtML
tv.blog.xlruof.cn/Article/details/353531.sHtML
tv.blog.xlruof.cn/Article/details/886084.sHtML
tv.blog.xlruof.cn/Article/details/595315.sHtML
tv.blog.xlruof.cn/Article/details/626428.sHtML
tv.blog.xlruof.cn/Article/details/264282.sHtML
tv.blog.xlruof.cn/Article/details/884848.sHtML
tv.blog.xlruof.cn/Article/details/791179.sHtML
tv.blog.xlruof.cn/Article/details/151375.sHtML
tv.blog.xlruof.cn/Article/details/575539.sHtML
tv.blog.xlruof.cn/Article/details/246286.sHtML
tv.blog.xlruof.cn/Article/details/622400.sHtML
tv.blog.xlruof.cn/Article/details/266044.sHtML
tv.blog.xlruof.cn/Article/details/464064.sHtML
tv.blog.xlruof.cn/Article/details/886606.sHtML
tv.blog.xlruof.cn/Article/details/199195.sHtML
tv.blog.xlruof.cn/Article/details/808846.sHtML
tv.blog.xlruof.cn/Article/details/224248.sHtML
tv.blog.xlruof.cn/Article/details/880480.sHtML
tv.blog.xlruof.cn/Article/details/331373.sHtML
tv.blog.xlruof.cn/Article/details/688242.sHtML
tv.blog.xlruof.cn/Article/details/711779.sHtML
tv.blog.xlruof.cn/Article/details/086246.sHtML
tv.blog.xlruof.cn/Article/details/246846.sHtML
tv.blog.xlruof.cn/Article/details/408826.sHtML
tv.blog.xlruof.cn/Article/details/155913.sHtML
tv.blog.xlruof.cn/Article/details/222248.sHtML
tv.blog.xlruof.cn/Article/details/648282.sHtML
tv.blog.xlruof.cn/Article/details/480406.sHtML
tv.blog.xlruof.cn/Article/details/717199.sHtML
tv.blog.xlruof.cn/Article/details/420000.sHtML
tv.blog.xlruof.cn/Article/details/486608.sHtML
tv.blog.xlruof.cn/Article/details/404028.sHtML
tv.blog.xlruof.cn/Article/details/664688.sHtML
tv.blog.xlruof.cn/Article/details/802802.sHtML
tv.blog.xlruof.cn/Article/details/420060.sHtML
tv.blog.xlruof.cn/Article/details/319397.sHtML
tv.blog.xlruof.cn/Article/details/664628.sHtML
tv.blog.xlruof.cn/Article/details/848464.sHtML
tv.blog.xlruof.cn/Article/details/842866.sHtML
tv.blog.xlruof.cn/Article/details/442200.sHtML
tv.blog.xlruof.cn/Article/details/420424.sHtML
tv.blog.xlruof.cn/Article/details/202028.sHtML
tv.blog.xlruof.cn/Article/details/313531.sHtML
tv.blog.xlruof.cn/Article/details/442002.sHtML
tv.blog.xlruof.cn/Article/details/864002.sHtML
tv.blog.xlruof.cn/Article/details/064600.sHtML
tv.blog.xlruof.cn/Article/details/682080.sHtML
tv.blog.xlruof.cn/Article/details/006606.sHtML
tv.blog.xlruof.cn/Article/details/662206.sHtML
tv.blog.xlruof.cn/Article/details/371379.sHtML
tv.blog.xlruof.cn/Article/details/866086.sHtML
tv.blog.xlruof.cn/Article/details/022862.sHtML
tv.blog.xlruof.cn/Article/details/424026.sHtML
tv.blog.xlruof.cn/Article/details/244026.sHtML
tv.blog.xlruof.cn/Article/details/800428.sHtML
tv.blog.xlruof.cn/Article/details/466664.sHtML
tv.blog.xlruof.cn/Article/details/135597.sHtML
tv.blog.xlruof.cn/Article/details/624468.sHtML
tv.blog.xlruof.cn/Article/details/004022.sHtML
tv.blog.xlruof.cn/Article/details/260024.sHtML
tv.blog.xlruof.cn/Article/details/424640.sHtML
tv.blog.xlruof.cn/Article/details/682600.sHtML
tv.blog.xlruof.cn/Article/details/404608.sHtML
tv.blog.xlruof.cn/Article/details/286820.sHtML
tv.blog.xlruof.cn/Article/details/628486.sHtML
tv.blog.xlruof.cn/Article/details/044608.sHtML
tv.blog.xlruof.cn/Article/details/999331.sHtML
tv.blog.xlruof.cn/Article/details/484646.sHtML
tv.blog.xlruof.cn/Article/details/686224.sHtML
tv.blog.xlruof.cn/Article/details/004268.sHtML
tv.blog.xlruof.cn/Article/details/775515.sHtML
tv.blog.xlruof.cn/Article/details/882464.sHtML
tv.blog.xlruof.cn/Article/details/200460.sHtML
tv.blog.xlruof.cn/Article/details/800448.sHtML
tv.blog.xlruof.cn/Article/details/955595.sHtML
tv.blog.xlruof.cn/Article/details/466482.sHtML
tv.blog.xlruof.cn/Article/details/046842.sHtML
tv.blog.xlruof.cn/Article/details/739339.sHtML
tv.blog.xlruof.cn/Article/details/466600.sHtML
tv.blog.xlruof.cn/Article/details/119771.sHtML
tv.blog.xlruof.cn/Article/details/468208.sHtML
tv.blog.xlruof.cn/Article/details/466808.sHtML
tv.blog.xlruof.cn/Article/details/626646.sHtML
tv.blog.xlruof.cn/Article/details/422804.sHtML
tv.blog.xlruof.cn/Article/details/240840.sHtML
tv.blog.xlruof.cn/Article/details/206622.sHtML
tv.blog.xlruof.cn/Article/details/460022.sHtML
tv.blog.xlruof.cn/Article/details/220228.sHtML
tv.blog.xlruof.cn/Article/details/713751.sHtML
tv.blog.xlruof.cn/Article/details/826220.sHtML
tv.blog.xlruof.cn/Article/details/862404.sHtML
tv.blog.xlruof.cn/Article/details/064640.sHtML
tv.blog.xlruof.cn/Article/details/408024.sHtML
tv.blog.xlruof.cn/Article/details/622400.sHtML
tv.blog.xlruof.cn/Article/details/000048.sHtML
tv.blog.xlruof.cn/Article/details/575773.sHtML
tv.blog.xlruof.cn/Article/details/359573.sHtML
tv.blog.xlruof.cn/Article/details/913919.sHtML
tv.blog.xlruof.cn/Article/details/688262.sHtML
tv.blog.xlruof.cn/Article/details/406460.sHtML
tv.blog.xlruof.cn/Article/details/422468.sHtML
tv.blog.xlruof.cn/Article/details/737193.sHtML
tv.blog.xlruof.cn/Article/details/999773.sHtML
tv.blog.xlruof.cn/Article/details/735773.sHtML
tv.blog.xlruof.cn/Article/details/648260.sHtML
tv.blog.xlruof.cn/Article/details/288648.sHtML
tv.blog.xlruof.cn/Article/details/000422.sHtML
tv.blog.xlruof.cn/Article/details/840882.sHtML
tv.blog.xlruof.cn/Article/details/533531.sHtML
tv.blog.xlruof.cn/Article/details/404420.sHtML
tv.blog.xlruof.cn/Article/details/197955.sHtML
tv.blog.xlruof.cn/Article/details/646624.sHtML
tv.blog.xlruof.cn/Article/details/622424.sHtML
tv.blog.xlruof.cn/Article/details/684246.sHtML
tv.blog.xlruof.cn/Article/details/539553.sHtML
tv.blog.xlruof.cn/Article/details/460824.sHtML
tv.blog.xlruof.cn/Article/details/775591.sHtML
tv.blog.xlruof.cn/Article/details/868808.sHtML
tv.blog.xlruof.cn/Article/details/002022.sHtML
tv.blog.xlruof.cn/Article/details/660026.sHtML
tv.blog.xlruof.cn/Article/details/159997.sHtML
tv.blog.xlruof.cn/Article/details/444646.sHtML
tv.blog.xlruof.cn/Article/details/660426.sHtML
tv.blog.xlruof.cn/Article/details/248840.sHtML
tv.blog.xlruof.cn/Article/details/646040.sHtML
tv.blog.xlruof.cn/Article/details/173135.sHtML
tv.blog.xlruof.cn/Article/details/779775.sHtML
tv.blog.xlruof.cn/Article/details/159519.sHtML
tv.blog.xlruof.cn/Article/details/244222.sHtML
tv.blog.xlruof.cn/Article/details/002204.sHtML
tv.blog.xlruof.cn/Article/details/826420.sHtML
tv.blog.xlruof.cn/Article/details/757371.sHtML
tv.blog.xlruof.cn/Article/details/486808.sHtML
tv.blog.xlruof.cn/Article/details/002468.sHtML
tv.blog.xlruof.cn/Article/details/595339.sHtML
tv.blog.xlruof.cn/Article/details/848820.sHtML
tv.blog.xlruof.cn/Article/details/737715.sHtML
tv.blog.xlruof.cn/Article/details/800464.sHtML
tv.blog.xlruof.cn/Article/details/224280.sHtML
tv.blog.xlruof.cn/Article/details/395795.sHtML
tv.blog.xlruof.cn/Article/details/206282.sHtML
tv.blog.xlruof.cn/Article/details/133753.sHtML
tv.blog.xlruof.cn/Article/details/486482.sHtML
tv.blog.xlruof.cn/Article/details/286204.sHtML
tv.blog.xlruof.cn/Article/details/620828.sHtML
tv.blog.xlruof.cn/Article/details/266482.sHtML
tv.blog.xlruof.cn/Article/details/200626.sHtML
tv.blog.xlruof.cn/Article/details/686628.sHtML
tv.blog.xlruof.cn/Article/details/139313.sHtML
tv.blog.xlruof.cn/Article/details/393937.sHtML
tv.blog.xlruof.cn/Article/details/971515.sHtML
tv.blog.xlruof.cn/Article/details/608842.sHtML
tv.blog.xlruof.cn/Article/details/468044.sHtML
tv.blog.xlruof.cn/Article/details/595571.sHtML
tv.blog.xlruof.cn/Article/details/088660.sHtML
tv.blog.xlruof.cn/Article/details/593119.sHtML
tv.blog.xlruof.cn/Article/details/884640.sHtML
tv.blog.xlruof.cn/Article/details/462204.sHtML
tv.blog.xlruof.cn/Article/details/468022.sHtML
tv.blog.xlruof.cn/Article/details/648602.sHtML
tv.blog.xlruof.cn/Article/details/351935.sHtML
tv.blog.xlruof.cn/Article/details/602868.sHtML
tv.blog.xlruof.cn/Article/details/044064.sHtML
tv.blog.xlruof.cn/Article/details/159933.sHtML
tv.blog.xlruof.cn/Article/details/088022.sHtML
tv.blog.xlruof.cn/Article/details/684048.sHtML
tv.blog.xlruof.cn/Article/details/088446.sHtML
tv.blog.xlruof.cn/Article/details/424222.sHtML
tv.blog.xlruof.cn/Article/details/137377.sHtML
tv.blog.xlruof.cn/Article/details/680468.sHtML
tv.blog.xlruof.cn/Article/details/028200.sHtML
tv.blog.xlruof.cn/Article/details/682026.sHtML
tv.blog.xlruof.cn/Article/details/066486.sHtML
tv.blog.xlruof.cn/Article/details/088002.sHtML
tv.blog.xlruof.cn/Article/details/440462.sHtML
tv.blog.xlruof.cn/Article/details/224866.sHtML
tv.blog.xlruof.cn/Article/details/062808.sHtML
tv.blog.xlruof.cn/Article/details/288248.sHtML
tv.blog.xlruof.cn/Article/details/400242.sHtML
tv.blog.xlruof.cn/Article/details/482484.sHtML
tv.blog.xlruof.cn/Article/details/666846.sHtML
tv.blog.xlruof.cn/Article/details/824644.sHtML
tv.blog.xlruof.cn/Article/details/048048.sHtML
tv.blog.xlruof.cn/Article/details/973391.sHtML
tv.blog.xlruof.cn/Article/details/139995.sHtML
tv.blog.xlruof.cn/Article/details/400666.sHtML
tv.blog.xlruof.cn/Article/details/119553.sHtML
tv.blog.xlruof.cn/Article/details/606688.sHtML
tv.blog.xlruof.cn/Article/details/044440.sHtML
tv.blog.xlruof.cn/Article/details/006646.sHtML
tv.blog.xlruof.cn/Article/details/315193.sHtML
tv.blog.xlruof.cn/Article/details/600666.sHtML
tv.blog.xlruof.cn/Article/details/848400.sHtML
tv.blog.xlruof.cn/Article/details/155391.sHtML
tv.blog.xlruof.cn/Article/details/315933.sHtML
tv.blog.xlruof.cn/Article/details/042680.sHtML
tv.blog.xlruof.cn/Article/details/024488.sHtML
tv.blog.xlruof.cn/Article/details/448462.sHtML
tv.blog.xlruof.cn/Article/details/682266.sHtML
tv.blog.xlruof.cn/Article/details/688462.sHtML
tv.blog.xlruof.cn/Article/details/864220.sHtML
tv.blog.xlruof.cn/Article/details/404224.sHtML
tv.blog.xlruof.cn/Article/details/020888.sHtML
tv.blog.xlruof.cn/Article/details/917155.sHtML
tv.blog.xlruof.cn/Article/details/735971.sHtML
tv.blog.xlruof.cn/Article/details/959997.sHtML
tv.blog.xlruof.cn/Article/details/713737.sHtML
tv.blog.xlruof.cn/Article/details/640428.sHtML
tv.blog.xlruof.cn/Article/details/224860.sHtML
tv.blog.xlruof.cn/Article/details/666408.sHtML
tv.blog.xlruof.cn/Article/details/135935.sHtML
tv.blog.xlruof.cn/Article/details/046480.sHtML
tv.blog.xlruof.cn/Article/details/084020.sHtML
tv.blog.xlruof.cn/Article/details/020246.sHtML
tv.blog.xlruof.cn/Article/details/311335.sHtML
tv.blog.xlruof.cn/Article/details/155597.sHtML
tv.blog.xlruof.cn/Article/details/284824.sHtML
tv.blog.xlruof.cn/Article/details/066848.sHtML
tv.blog.xlruof.cn/Article/details/260406.sHtML
tv.blog.xlruof.cn/Article/details/680806.sHtML
tv.blog.xlruof.cn/Article/details/428460.sHtML
tv.blog.xlruof.cn/Article/details/262420.sHtML
tv.blog.xlruof.cn/Article/details/715919.sHtML
tv.blog.xlruof.cn/Article/details/646884.sHtML
tv.blog.xlruof.cn/Article/details/973977.sHtML
tv.blog.xlruof.cn/Article/details/208268.sHtML
tv.blog.xlruof.cn/Article/details/422842.sHtML
tv.blog.xlruof.cn/Article/details/157395.sHtML
tv.blog.xlruof.cn/Article/details/842000.sHtML
tv.blog.xlruof.cn/Article/details/208440.sHtML
tv.blog.xlruof.cn/Article/details/600460.sHtML
tv.blog.xlruof.cn/Article/details/646460.sHtML
tv.blog.xlruof.cn/Article/details/440262.sHtML
tv.blog.xlruof.cn/Article/details/666862.sHtML
tv.blog.xlruof.cn/Article/details/602804.sHtML
tv.blog.xlruof.cn/Article/details/844860.sHtML
tv.blog.xlruof.cn/Article/details/662026.sHtML
tv.blog.xlruof.cn/Article/details/397573.sHtML
tv.blog.xlruof.cn/Article/details/402882.sHtML
tv.blog.xlruof.cn/Article/details/648066.sHtML
tv.blog.xlruof.cn/Article/details/608206.sHtML
tv.blog.xlruof.cn/Article/details/462484.sHtML
tv.blog.xlruof.cn/Article/details/731935.sHtML
tv.blog.xlruof.cn/Article/details/600264.sHtML
tv.blog.xlruof.cn/Article/details/284246.sHtML
tv.blog.xlruof.cn/Article/details/040868.sHtML
tv.blog.xlruof.cn/Article/details/060684.sHtML
tv.blog.xlruof.cn/Article/details/373715.sHtML
tv.blog.xlruof.cn/Article/details/868086.sHtML
tv.blog.xlruof.cn/Article/details/975711.sHtML
tv.blog.xlruof.cn/Article/details/820488.sHtML
tv.blog.xlruof.cn/Article/details/020608.sHtML
tv.blog.xlruof.cn/Article/details/264680.sHtML
