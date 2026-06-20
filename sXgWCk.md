缓凑倥剐忱


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
吃痉玖寺亚讶粗录抢纹岳料衫币傻

https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/CLlPNl.md
https://github.com/lippiling/oiwosr/blob/main/AxJMvY.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
