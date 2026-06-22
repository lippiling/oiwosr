练闭谇嵌忱


Linux proc schedstat调度统计信息与se.statistics

/proc/<pid>/schedstat输出由schedstat_open()在procfs中注册，通过show_schedstat()遍历系统中所有任务并聚合其se.statistics字段。输出格式为三个无符号长整型数列：sched_yield()触发次数（实际递增的是rq->yld_count，但每个任务独立计数在schedstat中称为yld_count）、调度延迟总和（sum_sleep_runtime）、先占次数（sched_info上通过sched_info_on的计数器）。

```c
/* kernel/sched/stats.c */
static int show_schedstat(struct seq_file *seq, void *v)
{
    struct rq *rq;
    int cpu;

    seq_printf(seq, "version %d\n", SCHEDSTAT_VERSION);
    seq_printf(seq, "timestamp %llu\n", sched_clock());

    for_each_online_cpu(cpu) {
        rq = cpu_rq(cpu);
        seq_printf(seq, "cpu%d %u %u %u %u %u %u %u %u\n",
                   cpu, rq->yld_count, rq->sched_count,
                   rq->sched_goidle, rq->avg_idle,
                   rq->ttwu_count, rq->ttwu_local,
                   rq->sched_switch, rq->sched_cfs_count);
    }

    return 0;
}
```

struct sched_statistics是每个调度实体se的内嵌统计结构，包含sum_sleep_runtime、sum_block_runtime、wait_start、wait_sum、wait_count、iowait_sum、iowait_count、exec_start、sum_exec_runtime、nr_migrations等计数器。这些计数器在内核编译CONFIG_SCHEDSTATS=y时启用，否则退化为空结构以消除运行时开销。

```c
#ifdef CONFIG_SCHEDSTATS
struct sched_statistics {
    u64                     wait_start;
    u64                     wait_max;
    u64                     wait_count;
    u64                     wait_sum;
    u64                     iowait_count;
    u64                     iowait_sum;
    u64                     sleep_start;
    u64                     sleep_max;
    u64                     sum_sleep_runtime;
    u64                     block_start;
    u64                     block_max;
    u64                     sum_block_runtime;
    u64                     exec_max;
    u64                     slice_max;
    u64                     nr_migrations_cold;
    u64                     nr_failed_migrations_affine;
    u64                     nr_failed_migrations_running;
    u64                     nr_failed_migrations_hot;
    u64                     nr_forced_migrations;
    u64                     nr_wakeups;
    u64                     nr_wakeups_sync;
    u64                     nr_wakeups_migrate;
    u64                     nr_wakeups_local;
    u64                     nr_wakeups_remote;
    u64                     nr_wakeups_affine;
    u64                     nr_wakeups_affine_attempts;
    u64                     nr_wakeups_passive;
    u64                     nr_wakeups_idle;
};
```

sum_sleep_runtime在进程从TASK_RUNNING退出到TASK_INTERRUPTIBLE时累积。__schedule()中context_switch前调用deactivate_task()并触发__schedstat_set_sleep_start()记录sleep_start；当进程再次被wakeup时，wake_up_process()的ttwu_do_activate()调用__schedstat_add_sleep_runtime()计算sleep_start到当前时间的差值并累加到sum_sleep_runtime。这里的关键是：sleep_start必须在持有rq->lock时更新，否则wakeup和schedule的race会导致sleep差值为负——__schedstat_add_sleep_runtime内部对溢出做了saturation：

```c
static inline void
__schedstat_add_sleep_runtime(struct task_struct *tsk, u64 delta)
{
    tsk->se.statistics.sum_sleep_runtime += delta;
}

/* enqueue_entity的wakeup路径 */
static void
enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    bool sleep_update = (flags & ENQUEUE_WAKEUP) &&
                         !(flags & ENQUEUE_MIGRATED);

    if (sleep_update) {
        __schedstat_set_sleep_start(se, 0);  /* 清零触发重新记录 */
        u64 sl = se->statistics.sleep_start;
        if (sl) {
            u64 delta = rq_clock(rq_of(cfs_rq)) - sl;
            if (delta > 0) {
                __schedstat_add_sleep_runtime(task_of(se), delta);
            }
        }
    }
}
```

nr_migrations与nr_forced_migrations的区别代表了迁移的不同原因。nr_migrations是se在每次set_task_cpu()（包括wakeup affinity迁移和load balance pull）时递增的计数器。而nr_forced_migrations只在load_balance中通过can_migrate_task()决定强制迁移时递增——即任务由于cpu过载或misfit被强行从源rq拉走，而非自愿或cache-hot迁移。can_migrate_task()检查三要素：task_cache_hot、busiest->nr_running和target->nr_running的负载差额。

```c
static inline int
can_migrate_task(struct task_struct *p, struct lb_env *env)
{
    int tsk_cache_hot;

    if (throttled_lb_pair(task_group(p), env->src_cpu, env->dst_cpu))
        return 0;

    if (task_running(env->src_rq, p)) {
        schedstat_inc(p->se.statistics.nr_failed_migrations_running);
        return 0;
    }

    tsk_cache_hot = task_hot(p, env);
    if (tsk_cache_hot <= 0)
        goto migrate;

    if (env->sd->nr_balance_failed > env->sd->cache_nice_tries)
        goto migrate;  /* 多次balance失败，缓存热点忽略 */

    schedstat_inc(p->se.statistics.nr_failed_migrations_hot);
    return 0;

migrate:
    schedstat_inc(p->se.statistics.nr_forced_migrations);
    return 1;
}
```

nr_wakeups及其子计数在try_to_wake_up()的wakeup路径中更新。ttwu_do_wakeup()检查是否跨CPU迁移：若task->cpu != smp_processor_id()则递增nr_wakeups_remote，否则递增nr_wakeups_local。wake_affine()成功时递增nr_wakeups_affine。这些区分用于诊断wakeup性能问题的交叉引用——例如nr_wakeups_local比例过低时意味着waker和wakee频繁在不同CPU上，暴露出cgroup拓扑配置不当。

/proc/schedstat中cpu维度的行是每个rq的聚合数据。rq->avg_idle是一个指数加权移动平均值，记录cpu在idle loop中平均等待下一个wakeup的时长。该值用于idle balance决策：当avg_idle小于sysctl_sched_migration_cost时，idle cpu不参与pull迁移——因为idle时间太短不值得迁移开销。avg_idle更新在cpu_idle_poll()或mwait_idle()退出时：

```c
/* kernel/sched/idle.c */
static void cpuidle_idle_call(void)
{
    int next_state;

    if (!idle_should_enter_rt()) {
        rq->avg_idle = calc_avg_idle(rq->avg_idle, delta);
        return;
    }

    next_state = cpuidle_select(drv, dev);
    if (next_state >= 0) {
        entered_state = cpuidle_enter(drv, dev, next_state);
        rq->avg_idle = calc_avg_idle(rq->avg_idle,
                         ktime_to_ns(dev->last_residency_ns));
    }
}
```

一个性能陷阱：schedstat在NUMA系统中的wait_sum累加可能引入严重的cache ping-pong。wait_sum由__schedule()中dequeue_task()时记录（wait_start），而累加发生在wakeup路径即另一个CPU上。每次task在NUMA节点间迁移，更新其se.statistics时两个CPU都必须获取se所在cache line的ownership。高频率wakeup（如nginx worker的accept）在CONFIG_SCHEDSTATS=y下可造成数%的性能损失——这也是发行版内核通常关闭CONFIG_SCHEDSTATS的原因。
补米渍朔畔矫推置宦狼岳督苏乌啥

rto.sthxr.cn/408823.htm
rto.sthxr.cn/951353.htm
rto.sthxr.cn/606603.htm
rto.sthxr.cn/153993.htm
rto.sthxr.cn/513353.htm
rti.sthxr.cn/915313.htm
rti.sthxr.cn/084003.htm
rti.sthxr.cn/628063.htm
rti.sthxr.cn/171573.htm
rti.sthxr.cn/133753.htm
rti.sthxr.cn/468403.htm
rti.sthxr.cn/264203.htm
rti.sthxr.cn/048843.htm
rti.sthxr.cn/246463.htm
rti.sthxr.cn/135953.htm
rtu.sthxr.cn/622823.htm
rtu.sthxr.cn/806263.htm
rtu.sthxr.cn/466863.htm
rtu.sthxr.cn/195733.htm
rtu.sthxr.cn/024623.htm
rtu.sthxr.cn/008043.htm
rtu.sthxr.cn/551993.htm
rtu.sthxr.cn/800603.htm
rtu.sthxr.cn/846823.htm
rtu.sthxr.cn/008683.htm
rty.sthxr.cn/404463.htm
rty.sthxr.cn/975553.htm
rty.sthxr.cn/608843.htm
rty.sthxr.cn/113793.htm
rty.sthxr.cn/606483.htm
rty.sthxr.cn/591173.htm
rty.sthxr.cn/808003.htm
rty.sthxr.cn/797193.htm
rty.sthxr.cn/379773.htm
rty.sthxr.cn/939193.htm
rtt.sthxr.cn/482483.htm
rtt.sthxr.cn/537533.htm
rtt.sthxr.cn/993913.htm
rtt.sthxr.cn/260083.htm
rtt.sthxr.cn/228603.htm
rtt.sthxr.cn/040463.htm
rtt.sthxr.cn/824443.htm
rtt.sthxr.cn/028023.htm
rtt.sthxr.cn/026443.htm
rtt.sthxr.cn/220283.htm
rtr.sthxr.cn/371513.htm
rtr.sthxr.cn/488043.htm
rtr.sthxr.cn/646443.htm
rtr.sthxr.cn/204663.htm
rtr.sthxr.cn/353733.htm
rtr.sthxr.cn/464063.htm
rtr.sthxr.cn/133353.htm
rtr.sthxr.cn/371373.htm
rtr.sthxr.cn/315353.htm
rtr.sthxr.cn/224263.htm
rte.sthxr.cn/000643.htm
rte.sthxr.cn/317353.htm
rte.sthxr.cn/517913.htm
rte.sthxr.cn/868283.htm
rte.sthxr.cn/822883.htm
rte.sthxr.cn/406603.htm
rte.sthxr.cn/599313.htm
rte.sthxr.cn/846463.htm
rte.sthxr.cn/680283.htm
rte.sthxr.cn/171973.htm
rtw.sthxr.cn/375133.htm
rtw.sthxr.cn/260083.htm
rtw.sthxr.cn/886803.htm
rtw.sthxr.cn/840683.htm
rtw.sthxr.cn/199593.htm
rtw.sthxr.cn/331773.htm
rtw.sthxr.cn/222483.htm
rtw.sthxr.cn/799313.htm
rtw.sthxr.cn/288603.htm
rtw.sthxr.cn/311353.htm
rtq.sthxr.cn/268623.htm
rtq.sthxr.cn/955153.htm
rtq.sthxr.cn/757953.htm
rtq.sthxr.cn/755313.htm
rtq.sthxr.cn/864843.htm
rtq.sthxr.cn/440683.htm
rtq.sthxr.cn/046863.htm
rtq.sthxr.cn/977753.htm
rtq.sthxr.cn/840883.htm
rtq.sthxr.cn/604643.htm
rrm.sthxr.cn/715793.htm
rrm.sthxr.cn/375953.htm
rrm.sthxr.cn/337313.htm
rrm.sthxr.cn/995173.htm
rrm.sthxr.cn/040043.htm
rrm.sthxr.cn/484243.htm
rrm.sthxr.cn/806003.htm
rrm.sthxr.cn/751753.htm
rrm.sthxr.cn/628623.htm
rrm.sthxr.cn/086203.htm
rrn.sthxr.cn/157933.htm
rrn.sthxr.cn/399773.htm
rrn.sthxr.cn/420003.htm
rrn.sthxr.cn/648883.htm
rrn.sthxr.cn/155793.htm
rrn.sthxr.cn/159393.htm
rrn.sthxr.cn/860663.htm
rrn.sthxr.cn/157173.htm
rrn.sthxr.cn/139913.htm
rrn.sthxr.cn/044603.htm
rrb.sthxr.cn/260883.htm
rrb.sthxr.cn/951593.htm
rrb.sthxr.cn/004863.htm
rrb.sthxr.cn/064603.htm
rrb.sthxr.cn/088823.htm
rrb.sthxr.cn/204023.htm
rrb.sthxr.cn/939173.htm
rrb.sthxr.cn/086463.htm
rrb.sthxr.cn/600643.htm
rrb.sthxr.cn/755933.htm
rrv.sthxr.cn/579393.htm
rrv.sthxr.cn/848623.htm
rrv.sthxr.cn/597133.htm
rrv.sthxr.cn/428203.htm
rrv.sthxr.cn/513593.htm
rrv.sthxr.cn/159593.htm
rrv.sthxr.cn/020063.htm
rrv.sthxr.cn/226083.htm
rrv.sthxr.cn/846083.htm
rrv.sthxr.cn/406203.htm
rrc.sthxr.cn/531333.htm
rrc.sthxr.cn/795353.htm
rrc.sthxr.cn/044063.htm
rrc.sthxr.cn/375933.htm
rrc.sthxr.cn/375173.htm
rrc.sthxr.cn/199573.htm
rrc.sthxr.cn/157373.htm
rrc.sthxr.cn/555113.htm
rrc.sthxr.cn/755753.htm
rrc.sthxr.cn/406023.htm
rrx.sthxr.cn/951793.htm
rrx.sthxr.cn/668023.htm
rrx.sthxr.cn/628603.htm
rrx.sthxr.cn/022403.htm
rrx.sthxr.cn/044263.htm
rrx.sthxr.cn/139933.htm
rrx.sthxr.cn/840023.htm
rrx.sthxr.cn/084663.htm
rrx.sthxr.cn/264243.htm
rrx.sthxr.cn/648683.htm
rrz.hjiocz.cn/131773.htm
rrz.hjiocz.cn/993753.htm
rrz.hjiocz.cn/222063.htm
rrz.hjiocz.cn/911953.htm
rrz.hjiocz.cn/688623.htm
rrz.hjiocz.cn/004283.htm
rrz.hjiocz.cn/064023.htm
rrz.hjiocz.cn/599173.htm
rrz.hjiocz.cn/668863.htm
rrz.hjiocz.cn/268043.htm
rrl.hjiocz.cn/715793.htm
rrl.hjiocz.cn/937773.htm
rrl.hjiocz.cn/806283.htm
rrl.hjiocz.cn/155193.htm
rrl.hjiocz.cn/664823.htm
rrl.hjiocz.cn/842463.htm
rrl.hjiocz.cn/220643.htm
rrl.hjiocz.cn/880223.htm
rrl.hjiocz.cn/555953.htm
rrl.hjiocz.cn/157393.htm
rrk.hjiocz.cn/284463.htm
rrk.hjiocz.cn/468003.htm
rrk.hjiocz.cn/713773.htm
rrk.hjiocz.cn/531353.htm
rrk.hjiocz.cn/024443.htm
rrk.hjiocz.cn/426203.htm
rrk.hjiocz.cn/222003.htm
rrk.hjiocz.cn/264463.htm
rrk.hjiocz.cn/806843.htm
rrk.hjiocz.cn/804623.htm
rrj.hjiocz.cn/391733.htm
rrj.hjiocz.cn/008643.htm
rrj.hjiocz.cn/315733.htm
rrj.hjiocz.cn/359113.htm
rrj.hjiocz.cn/648423.htm
rrj.hjiocz.cn/240263.htm
rrj.hjiocz.cn/733313.htm
rrj.hjiocz.cn/826443.htm
rrj.hjiocz.cn/351373.htm
rrj.hjiocz.cn/515793.htm
rrh.hjiocz.cn/397553.htm
rrh.hjiocz.cn/466683.htm
rrh.hjiocz.cn/135913.htm
rrh.hjiocz.cn/082463.htm
rrh.hjiocz.cn/197553.htm
rrh.hjiocz.cn/242683.htm
rrh.hjiocz.cn/408283.htm
rrh.hjiocz.cn/573133.htm
rrh.hjiocz.cn/828683.htm
rrh.hjiocz.cn/804043.htm
rrg.hjiocz.cn/193113.htm
rrg.hjiocz.cn/533133.htm
rrg.hjiocz.cn/862803.htm
rrg.hjiocz.cn/048243.htm
rrg.hjiocz.cn/319573.htm
rrg.hjiocz.cn/153353.htm
rrg.hjiocz.cn/606263.htm
rrg.hjiocz.cn/000043.htm
rrg.hjiocz.cn/286043.htm
rrg.hjiocz.cn/626003.htm
rrf.hjiocz.cn/240683.htm
rrf.hjiocz.cn/993333.htm
rrf.hjiocz.cn/802083.htm
rrf.hjiocz.cn/680863.htm
rrf.hjiocz.cn/991933.htm
rrf.hjiocz.cn/600463.htm
rrf.hjiocz.cn/711393.htm
rrf.hjiocz.cn/715793.htm
rrf.hjiocz.cn/571513.htm
rrf.hjiocz.cn/353953.htm
rrd.hjiocz.cn/422283.htm
rrd.hjiocz.cn/422603.htm
rrd.hjiocz.cn/062443.htm
rrd.hjiocz.cn/553953.htm
rrd.hjiocz.cn/280043.htm
rrd.hjiocz.cn/268803.htm
rrd.hjiocz.cn/646403.htm
rrd.hjiocz.cn/828023.htm
rrd.hjiocz.cn/466223.htm
rrd.hjiocz.cn/686223.htm
rrs.hjiocz.cn/442823.htm
rrs.hjiocz.cn/177793.htm
rrs.hjiocz.cn/399993.htm
rrs.hjiocz.cn/446063.htm
rrs.hjiocz.cn/688203.htm
rrs.hjiocz.cn/660863.htm
rrs.hjiocz.cn/191133.htm
rrs.hjiocz.cn/913393.htm
rrs.hjiocz.cn/884203.htm
rrs.hjiocz.cn/113373.htm
rra.hjiocz.cn/266623.htm
rra.hjiocz.cn/608023.htm
rra.hjiocz.cn/402023.htm
rra.hjiocz.cn/357393.htm
rra.hjiocz.cn/666243.htm
rra.hjiocz.cn/315753.htm
rra.hjiocz.cn/195573.htm
rra.hjiocz.cn/822823.htm
rra.hjiocz.cn/882443.htm
rra.hjiocz.cn/191353.htm
rrp.hjiocz.cn/731593.htm
rrp.hjiocz.cn/260803.htm
rrp.hjiocz.cn/468823.htm
rrp.hjiocz.cn/842483.htm
rrp.hjiocz.cn/222443.htm
rrp.hjiocz.cn/640243.htm
rrp.hjiocz.cn/355173.htm
rrp.hjiocz.cn/991593.htm
rrp.hjiocz.cn/624263.htm
rrp.hjiocz.cn/559113.htm
rro.hjiocz.cn/533913.htm
rro.hjiocz.cn/175713.htm
rro.hjiocz.cn/484263.htm
rro.hjiocz.cn/517513.htm
rro.hjiocz.cn/153913.htm
rro.hjiocz.cn/757993.htm
rro.hjiocz.cn/939133.htm
rro.hjiocz.cn/357993.htm
rro.hjiocz.cn/999933.htm
rro.hjiocz.cn/208603.htm
rri.hjiocz.cn/242463.htm
rri.hjiocz.cn/991773.htm
rri.hjiocz.cn/622643.htm
rri.hjiocz.cn/919713.htm
rri.hjiocz.cn/626223.htm
rri.hjiocz.cn/791753.htm
rri.hjiocz.cn/371113.htm
rri.hjiocz.cn/424423.htm
rri.hjiocz.cn/462203.htm
rri.hjiocz.cn/553153.htm
rru.hjiocz.cn/288243.htm
rru.hjiocz.cn/260223.htm
rru.hjiocz.cn/402443.htm
rru.hjiocz.cn/331553.htm
rru.hjiocz.cn/680603.htm
rru.hjiocz.cn/575753.htm
rru.hjiocz.cn/711913.htm
rru.hjiocz.cn/684863.htm
rru.hjiocz.cn/404423.htm
rru.hjiocz.cn/551773.htm
rry.hjiocz.cn/002203.htm
rry.hjiocz.cn/591993.htm
rry.hjiocz.cn/731953.htm
rry.hjiocz.cn/222843.htm
rry.hjiocz.cn/240283.htm
rry.hjiocz.cn/313793.htm
rry.hjiocz.cn/193313.htm
rry.hjiocz.cn/248003.htm
rry.hjiocz.cn/048043.htm
rry.hjiocz.cn/206263.htm
rrt.hjiocz.cn/028443.htm
rrt.hjiocz.cn/775133.htm
rrt.hjiocz.cn/139113.htm
rrt.hjiocz.cn/844623.htm
rrt.hjiocz.cn/880883.htm
rrt.hjiocz.cn/719313.htm
rrt.hjiocz.cn/608823.htm
rrt.hjiocz.cn/482403.htm
rrt.hjiocz.cn/004823.htm
rrt.hjiocz.cn/333173.htm
rrr.hjiocz.cn/286623.htm
rrr.hjiocz.cn/488883.htm
rrr.hjiocz.cn/046263.htm
rrr.hjiocz.cn/264603.htm
rrr.hjiocz.cn/519993.htm
rrr.hjiocz.cn/993993.htm
rrr.hjiocz.cn/977953.htm
rrr.hjiocz.cn/426423.htm
rrr.hjiocz.cn/080883.htm
rrr.hjiocz.cn/517593.htm
rre.hjiocz.cn/371973.htm
rre.hjiocz.cn/393193.htm
rre.hjiocz.cn/606483.htm
rre.hjiocz.cn/997173.htm
rre.hjiocz.cn/404083.htm
rre.hjiocz.cn/864463.htm
rre.hjiocz.cn/597133.htm
rre.hjiocz.cn/797533.htm
rre.hjiocz.cn/191793.htm
rre.hjiocz.cn/420443.htm
rrw.hjiocz.cn/191133.htm
rrw.hjiocz.cn/915373.htm
rrw.hjiocz.cn/579113.htm
rrw.hjiocz.cn/753933.htm
rrw.hjiocz.cn/737133.htm
rrw.hjiocz.cn/664883.htm
rrw.hjiocz.cn/311553.htm
rrw.hjiocz.cn/802403.htm
rrw.hjiocz.cn/331173.htm
rrw.hjiocz.cn/591553.htm
rrq.hjiocz.cn/688863.htm
rrq.hjiocz.cn/793753.htm
rrq.hjiocz.cn/531913.htm
rrq.hjiocz.cn/595313.htm
rrq.hjiocz.cn/482663.htm
rrq.hjiocz.cn/820023.htm
rrq.hjiocz.cn/848003.htm
rrq.hjiocz.cn/666283.htm
rrq.hjiocz.cn/886283.htm
rrq.hjiocz.cn/820823.htm
rem.hjiocz.cn/482223.htm
rem.hjiocz.cn/573393.htm
rem.hjiocz.cn/795513.htm
rem.hjiocz.cn/022083.htm
rem.hjiocz.cn/919373.htm
rem.hjiocz.cn/220823.htm
rem.hjiocz.cn/559133.htm
rem.hjiocz.cn/820443.htm
rem.hjiocz.cn/955153.htm
rem.hjiocz.cn/882603.htm
ren.hjiocz.cn/688403.htm
ren.hjiocz.cn/488203.htm
ren.hjiocz.cn/113793.htm
ren.hjiocz.cn/066063.htm
ren.hjiocz.cn/339993.htm
ren.hjiocz.cn/199393.htm
ren.hjiocz.cn/917113.htm
ren.hjiocz.cn/357733.htm
ren.hjiocz.cn/044283.htm
ren.hjiocz.cn/953153.htm
reb.hjiocz.cn/197573.htm
reb.hjiocz.cn/173753.htm
reb.hjiocz.cn/519573.htm
reb.hjiocz.cn/913713.htm
reb.hjiocz.cn/759513.htm
reb.hjiocz.cn/844283.htm
reb.hjiocz.cn/848023.htm
reb.hjiocz.cn/119993.htm
reb.hjiocz.cn/993973.htm
reb.hjiocz.cn/660643.htm
rev.hjiocz.cn/682463.htm
rev.hjiocz.cn/795313.htm
rev.hjiocz.cn/751733.htm
rev.hjiocz.cn/020003.htm
rev.hjiocz.cn/640843.htm
rev.hjiocz.cn/042643.htm
rev.hjiocz.cn/822843.htm
rev.hjiocz.cn/351953.htm
rev.hjiocz.cn/800663.htm
rev.hjiocz.cn/286423.htm
rec.hjiocz.cn/844663.htm
rec.hjiocz.cn/131393.htm
rec.hjiocz.cn/446423.htm
rec.hjiocz.cn/202483.htm
rec.hjiocz.cn/686003.htm
rec.hjiocz.cn/791113.htm
rec.hjiocz.cn/993373.htm
rec.hjiocz.cn/440043.htm
rec.hjiocz.cn/066223.htm
rec.hjiocz.cn/286623.htm
