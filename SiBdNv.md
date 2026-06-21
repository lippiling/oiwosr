屎影量园烈


Linux sched_yield主动让出CPU与sys_sched_yield实现

sys_sched_yield()是唯一由用户态直接触发的自愿调度点，对应glibc的sched_yield()。其内核入口直接调用yield_to_task_fair()的封装，不经过任何时间片检查或权限验证——任何进程都可以任意调用yield释放cpu，这与SCHED_FIFO的rt_throttled机制不同，yield不受带宽限制。

```c
SYSCALL_DEFINE0(sched_yield)
{
    struct rq_flags rf;
    struct rq *rq;

    rq = this_rq_lock_irq(&rf);

    schedstat_inc(rq->yld_count);
    current->sched_class->yield_task(rq);

    preempt_disable();
    rq_unlock_irq(rq, &rf);
    sched_preempt_enable_no_resched();

    schedule();  /* 主动触发reschedule */

    return 0;
}
```

yield_task()在CFS调度类中实现为yield_task_fair()。该函数的核心操作是将当前se从CFS红黑树中删除后重新插入，但插入位置被调整到树的最后端——即vruntime被向后推。具体做法是通过update_curr()更新当前vruntime，然后调用clear_buddies()清除cfs_rq上的next、last、skip指针，最后调用__enqueue_entity()将se重新入队到红黑树的最右端（max vruntime位置）。

```c
static void yield_task_fair(struct rq *rq)
{
    struct task_struct *curr = rq->curr;
    struct cfs_rq *cfs_rq = task_cfs_rq(curr);
    struct sched_entity *se = &curr->se;

    if (unlikely(rq->nr_running == 1))
        return;  /* 只有自己一个任务，yield没有意义 */

    clear_buddies(cfs_rq, se);

    if (curr->policy != SCHED_BATCH) {
        update_rq_clock(rq);
        update_curr(cfs_rq);
        rq_clock_skip_update(rq);
    }

    se->vruntime = cfs_rq->min_vruntime + sched_vslice(cfs_rq, se)
                   + sysctl_sched_yield_slack;

    list_del(&se->group_node);
    /* 重新入队到红黑树最右侧 */
    __enqueue_entity(cfs_rq, se);
}
```

yield时设置se->vruntime = cfs_rq->min_vruntime + sched_vslice(cfs_rq, se) + sysctl_sched_yield_slack。其中sysctl_sched_yield_slack默认值为1ms（以ns为单位），作用是防止yield后的进程因vruntime恰好等于min_vruntime而立即被再次选中。sched_vslice是当前se的时间片在CFS下的虚拟表示——这强制使当前se变为cfs_rq中vruntime最大的实体，因此在pick_next_task_fair()中，红黑树最左节点不会是该se。

一个关键竞态：yield期间调用update_curr()可能导致cfs_rq->min_vruntime被推进。如果另一个CPU上的wakeup在yield_task_fair的执行窗口中对同一个cfs_rq执行了enqueue_entity，新的min_vruntime会突然增大，使得重新入队的se的vruntime相对变小——在极端情况下yield可能完全失效。但在SMP下每个CPU有独立cfs_rq（除非CONFIG_FAIR_GROUP_SCHED开启了task group），单CPU场景没有这个窗口。

```c
static void yield_task_rt(struct rq *rq)
{
    struct task_struct *curr = rq->curr;
    struct rt_rq *rt_rq = &rq->rt;

    if (rt_rq->rt_nr_running == 1)
        return;

    if (curr->policy != SCHED_RR) {
        /* SCHED_FIFO调用yield意味着完全不公平——把当前任务移到队尾 */
        if (!plist_head_empty(&rt_rq->pushable_tasks))
            dequeue_pushable_task(rq, curr);
    }

    requeue_task_rt(rq, curr, 1);
    set_tsk_need_resched(curr);
}
```

yield与cond_resched()有本质区别。cond_resched()只在need_resched()为true时才调用schedule()，而sys_sched_yield直接强制调用schedule()。spinlock锁持有者调用yield后如果被同一个锁的waiter抢占，会导致lock holder长时间无法释放锁——这是ABBA deadlock的一个变种。内核自旋锁持有路径通过PREEMPT_COUNT和preempt_disable()来确保在spin_unlock前不会被抢占，但yield_task_fair中的__enqueue_entity不检查preempt_count，它只对rq->lock持有情况负责。

yield的另一个陷阱是truncated yield：当一个进程连续调用sched_yield时，每次调用都会把vruntime推到当前min_vruntime + slice + slack的位置。但每次调用之间的时间间隔中如果其他进程消耗了vruntime并使min_vruntime前进，yield进程的vruntime相对值可能增长缓慢——从而导致它实际上并没有被推到队列最尾。这在实时系统中需要规避，所以POSIX标准规定SCHED_FIFO/yield的行为是移到优先级队列尾而不是修改vruntime。

yield_task_fair中对于SCHED_BATCH策略的特殊处理是跳过update_curr()。SCHED_BATCH任务期望长时间运行，update_curr()会触发load weight计算和PELT更新，这些计算在batch场景下不应该为一次yield触发。rq_clock_skip_update()标记RQC_ACT_SKIP标志来阻止随后的update_rq_clock再刷新clock。

```c
static inline void rq_clock_skip_update(struct rq *rq)
{
    rq->clock_update_flags |= RQCF_ACT_SKIP;
}
```

yield_task_fair中先list_del(&se->group_node)将se从cfs_rq的leaf list中移除，然后通过__enqueue_entity重新插入。leaf list维护了cfs_rq所有se的顺序遍历链表，用于update_curr的向上传播。跳过该步骤会导致leaf list出现重复节点，进而导致update_cfs_rq_h_load()中的h_load计算进入死循环。

在调度统计层面，rq->yld_count在每次sys_sched_yield入口递增，而se.statistics中不记录single yield事件——只有通过yield触发的schedule()产生的context_switch才会计入nr_switches。这使得识别频繁yield的进程需要perf probe挂载在yield_task_fair入口。
嘉恼中掠沿澄狼昧惺脊侥毫驮型糙

m.qlb.qtbzn.cn/80246.Doc
m.qlb.qtbzn.cn/84424.Doc
m.qlb.qtbzn.cn/60088.Doc
m.qlb.qtbzn.cn/80602.Doc
m.qlb.qtbzn.cn/84664.Doc
m.qlb.qtbzn.cn/04264.Doc
m.qlb.qtbzn.cn/97519.Doc
m.qlb.qtbzn.cn/04662.Doc
m.qlb.qtbzn.cn/39717.Doc
m.qlb.qtbzn.cn/22802.Doc
m.qlb.qtbzn.cn/17359.Doc
m.qlb.qtbzn.cn/26668.Doc
m.qlb.qtbzn.cn/40244.Doc
m.qlb.qtbzn.cn/04828.Doc
m.qlb.qtbzn.cn/93937.Doc
m.qlb.qtbzn.cn/28646.Doc
m.qlb.qtbzn.cn/35375.Doc
m.qlv.qtbzn.cn/00826.Doc
m.qlv.qtbzn.cn/22246.Doc
m.qlv.qtbzn.cn/26804.Doc
m.qlv.qtbzn.cn/82668.Doc
m.qlv.qtbzn.cn/99537.Doc
m.qlv.qtbzn.cn/44688.Doc
m.qlv.qtbzn.cn/26622.Doc
m.qlv.qtbzn.cn/42426.Doc
m.qlv.qtbzn.cn/73355.Doc
m.qlv.qtbzn.cn/28884.Doc
m.qlv.qtbzn.cn/80242.Doc
m.qlv.qtbzn.cn/24028.Doc
m.qlv.qtbzn.cn/60260.Doc
m.qlv.qtbzn.cn/80848.Doc
m.qlv.qtbzn.cn/84666.Doc
m.qlv.qtbzn.cn/80048.Doc
m.qlv.qtbzn.cn/35959.Doc
m.qlv.qtbzn.cn/86464.Doc
m.qlv.qtbzn.cn/60820.Doc
m.qlv.qtbzn.cn/26024.Doc
m.qlc.qtbzn.cn/22488.Doc
m.qlc.qtbzn.cn/13953.Doc
m.qlc.qtbzn.cn/68022.Doc
m.qlc.qtbzn.cn/00600.Doc
m.qlc.qtbzn.cn/26840.Doc
m.qlc.qtbzn.cn/97935.Doc
m.qlc.qtbzn.cn/80606.Doc
m.qlc.qtbzn.cn/86688.Doc
m.qlc.qtbzn.cn/86006.Doc
m.qlc.qtbzn.cn/48600.Doc
m.qlc.qtbzn.cn/06448.Doc
m.qlc.qtbzn.cn/68042.Doc
m.qlc.qtbzn.cn/00888.Doc
m.qlc.qtbzn.cn/97317.Doc
m.qlc.qtbzn.cn/24864.Doc
m.qlc.qtbzn.cn/95755.Doc
m.qlc.qtbzn.cn/20802.Doc
m.qlc.qtbzn.cn/00088.Doc
m.qlc.qtbzn.cn/04682.Doc
m.qlc.qtbzn.cn/82820.Doc
m.qlx.qtbzn.cn/88088.Doc
m.qlx.qtbzn.cn/68260.Doc
m.qlx.qtbzn.cn/60086.Doc
m.qlx.qtbzn.cn/66026.Doc
m.qlx.qtbzn.cn/60006.Doc
m.qlx.qtbzn.cn/73131.Doc
m.qlx.qtbzn.cn/08028.Doc
m.qlx.qtbzn.cn/79779.Doc
m.qlx.qtbzn.cn/02442.Doc
m.qlx.qtbzn.cn/06406.Doc
m.qlx.qtbzn.cn/68088.Doc
m.qlx.qtbzn.cn/57995.Doc
m.qlx.qtbzn.cn/28628.Doc
m.qlx.qtbzn.cn/68840.Doc
m.qlx.qtbzn.cn/26666.Doc
m.qlx.qtbzn.cn/20682.Doc
m.qlx.qtbzn.cn/66802.Doc
m.qlx.qtbzn.cn/80468.Doc
m.qlx.qtbzn.cn/22428.Doc
m.qlx.qtbzn.cn/55577.Doc
m.qlz.qtbzn.cn/86684.Doc
m.qlz.qtbzn.cn/22688.Doc
m.qlz.qtbzn.cn/46666.Doc
m.qlz.qtbzn.cn/35937.Doc
m.qlz.qtbzn.cn/08084.Doc
m.qlz.qtbzn.cn/57553.Doc
m.qlz.qtbzn.cn/46480.Doc
m.qlz.qtbzn.cn/08680.Doc
m.qlz.qtbzn.cn/22406.Doc
m.qlz.qtbzn.cn/31115.Doc
m.qlz.qtbzn.cn/91793.Doc
m.qlz.qtbzn.cn/00606.Doc
m.qlz.qtbzn.cn/46866.Doc
m.qlz.qtbzn.cn/64228.Doc
m.qlz.qtbzn.cn/22888.Doc
m.qlz.qtbzn.cn/57515.Doc
m.qlz.qtbzn.cn/00024.Doc
m.qlz.qtbzn.cn/48668.Doc
m.qlz.qtbzn.cn/60460.Doc
m.qlz.qtbzn.cn/68622.Doc
m.qll.qtbzn.cn/22826.Doc
m.qll.qtbzn.cn/95177.Doc
m.qll.qtbzn.cn/24884.Doc
m.qll.qtbzn.cn/44820.Doc
m.qll.qtbzn.cn/80480.Doc
m.qll.qtbzn.cn/02280.Doc
m.qll.qtbzn.cn/88860.Doc
m.qll.qtbzn.cn/24662.Doc
m.qll.qtbzn.cn/37971.Doc
m.qll.qtbzn.cn/37399.Doc
m.qll.qtbzn.cn/40684.Doc
m.qll.qtbzn.cn/82826.Doc
m.qll.qtbzn.cn/24244.Doc
m.qll.qtbzn.cn/20002.Doc
m.qll.qtbzn.cn/19171.Doc
m.qll.qtbzn.cn/02484.Doc
m.qll.qtbzn.cn/80240.Doc
m.qll.qtbzn.cn/40848.Doc
m.qll.qtbzn.cn/06448.Doc
m.qll.qtbzn.cn/80840.Doc
m.qlk.qtbzn.cn/71735.Doc
m.qlk.qtbzn.cn/62002.Doc
m.qlk.qtbzn.cn/20864.Doc
m.qlk.qtbzn.cn/08402.Doc
m.qlk.qtbzn.cn/71113.Doc
m.qlk.qtbzn.cn/60448.Doc
m.qlk.qtbzn.cn/73115.Doc
m.qlk.qtbzn.cn/04662.Doc
m.qlk.qtbzn.cn/82680.Doc
m.qlk.qtbzn.cn/79533.Doc
m.qlk.qtbzn.cn/80264.Doc
m.qlk.qtbzn.cn/57193.Doc
m.qlk.qtbzn.cn/53579.Doc
m.qlk.qtbzn.cn/64862.Doc
m.qlk.qtbzn.cn/73991.Doc
m.qlk.qtbzn.cn/75133.Doc
m.qlk.qtbzn.cn/53971.Doc
m.qlk.qtbzn.cn/42622.Doc
m.qlk.qtbzn.cn/68484.Doc
m.qlk.qtbzn.cn/02826.Doc
m.qlj.qtbzn.cn/40846.Doc
m.qlj.qtbzn.cn/84060.Doc
m.qlj.qtbzn.cn/46422.Doc
m.qlj.qtbzn.cn/68204.Doc
m.qlj.qtbzn.cn/02228.Doc
m.qlj.qtbzn.cn/44024.Doc
m.qlj.qtbzn.cn/04468.Doc
m.qlj.qtbzn.cn/20846.Doc
m.qlj.qtbzn.cn/24680.Doc
m.qlj.qtbzn.cn/48800.Doc
m.qlj.qtbzn.cn/26822.Doc
m.qlj.qtbzn.cn/48246.Doc
m.qlj.qtbzn.cn/26248.Doc
m.qlj.qtbzn.cn/46080.Doc
m.qlj.qtbzn.cn/84844.Doc
m.qlj.qtbzn.cn/62866.Doc
m.qlj.qtbzn.cn/68480.Doc
m.qlj.qtbzn.cn/02824.Doc
m.qlj.qtbzn.cn/20048.Doc
m.qlj.qtbzn.cn/60646.Doc
m.qlh.qtbzn.cn/93759.Doc
m.qlh.qtbzn.cn/71117.Doc
m.qlh.qtbzn.cn/08462.Doc
m.qlh.qtbzn.cn/04004.Doc
m.qlh.qtbzn.cn/31175.Doc
m.qlh.qtbzn.cn/88822.Doc
m.qlh.qtbzn.cn/57751.Doc
m.qlh.qtbzn.cn/06048.Doc
m.qlh.qtbzn.cn/20460.Doc
m.qlh.qtbzn.cn/64042.Doc
m.qlh.qtbzn.cn/08848.Doc
m.qlh.qtbzn.cn/88242.Doc
m.qlh.qtbzn.cn/99339.Doc
m.qlh.qtbzn.cn/28684.Doc
m.qlh.qtbzn.cn/66680.Doc
m.qlh.qtbzn.cn/55915.Doc
m.qlh.qtbzn.cn/88684.Doc
m.qlh.qtbzn.cn/46208.Doc
m.qlh.qtbzn.cn/71777.Doc
m.qlh.qtbzn.cn/26464.Doc
m.qlg.qtbzn.cn/19935.Doc
m.qlg.qtbzn.cn/77593.Doc
m.qlg.qtbzn.cn/51999.Doc
m.qlg.qtbzn.cn/15937.Doc
m.qlg.qtbzn.cn/86000.Doc
m.qlg.qtbzn.cn/26864.Doc
m.qlg.qtbzn.cn/68402.Doc
m.qlg.qtbzn.cn/82602.Doc
m.qlg.qtbzn.cn/42426.Doc
m.qlg.qtbzn.cn/42242.Doc
m.qlg.qtbzn.cn/02426.Doc
m.qlg.qtbzn.cn/04608.Doc
m.qlg.qtbzn.cn/44804.Doc
m.qlg.qtbzn.cn/66242.Doc
m.qlg.qtbzn.cn/48260.Doc
m.qlg.qtbzn.cn/59573.Doc
m.qlg.qtbzn.cn/59177.Doc
m.qlg.qtbzn.cn/82664.Doc
m.qlg.qtbzn.cn/20864.Doc
m.qlg.qtbzn.cn/00220.Doc
m.qlf.qtbzn.cn/60068.Doc
m.qlf.qtbzn.cn/64842.Doc
m.qlf.qtbzn.cn/13111.Doc
m.qlf.qtbzn.cn/20284.Doc
m.qlf.qtbzn.cn/20426.Doc
m.qlf.qtbzn.cn/00422.Doc
m.qlf.qtbzn.cn/20088.Doc
m.qlf.qtbzn.cn/28622.Doc
m.qlf.qtbzn.cn/24086.Doc
m.qlf.qtbzn.cn/22646.Doc
m.qlf.qtbzn.cn/66602.Doc
m.qlf.qtbzn.cn/26466.Doc
m.qlf.qtbzn.cn/28682.Doc
m.qlf.qtbzn.cn/00866.Doc
m.qlf.qtbzn.cn/80084.Doc
m.qlf.qtbzn.cn/37537.Doc
m.qlf.qtbzn.cn/44062.Doc
m.qlf.qtbzn.cn/88602.Doc
m.qlf.qtbzn.cn/28664.Doc
m.qlf.qtbzn.cn/44244.Doc
m.qld.qtbzn.cn/71737.Doc
m.qld.qtbzn.cn/82424.Doc
m.qld.qtbzn.cn/42464.Doc
m.qld.qtbzn.cn/64202.Doc
m.qld.qtbzn.cn/88842.Doc
m.qld.qtbzn.cn/02420.Doc
m.qld.qtbzn.cn/68888.Doc
m.qld.qtbzn.cn/13557.Doc
m.qld.qtbzn.cn/46866.Doc
m.qld.qtbzn.cn/24264.Doc
m.qld.qtbzn.cn/24620.Doc
m.qld.qtbzn.cn/64428.Doc
m.qld.qtbzn.cn/20046.Doc
m.qld.qtbzn.cn/37133.Doc
m.qld.qtbzn.cn/20486.Doc
m.qld.qtbzn.cn/22880.Doc
m.qld.qtbzn.cn/31313.Doc
m.qld.qtbzn.cn/02466.Doc
m.qld.qtbzn.cn/13599.Doc
m.qld.qtbzn.cn/22008.Doc
m.qls.qtbzn.cn/24620.Doc
m.qls.qtbzn.cn/20804.Doc
m.qls.qtbzn.cn/46884.Doc
m.qls.qtbzn.cn/42624.Doc
m.qls.qtbzn.cn/66240.Doc
m.qls.qtbzn.cn/48426.Doc
m.qls.qtbzn.cn/22288.Doc
m.qls.qtbzn.cn/64642.Doc
m.qls.qtbzn.cn/82662.Doc
m.qls.qtbzn.cn/20066.Doc
m.qls.qtbzn.cn/95171.Doc
m.qls.qtbzn.cn/99717.Doc
m.qls.qtbzn.cn/84080.Doc
m.qls.qtbzn.cn/20280.Doc
m.qls.qtbzn.cn/11575.Doc
m.qls.qtbzn.cn/02646.Doc
m.qls.qtbzn.cn/39151.Doc
m.qls.qtbzn.cn/66668.Doc
m.qls.qtbzn.cn/86024.Doc
m.qls.qtbzn.cn/42244.Doc
m.qla.qtbzn.cn/60020.Doc
m.qla.qtbzn.cn/00280.Doc
m.qla.qtbzn.cn/13571.Doc
m.qla.qtbzn.cn/00002.Doc
m.qla.qtbzn.cn/06620.Doc
m.qla.qtbzn.cn/62626.Doc
m.qla.qtbzn.cn/40842.Doc
m.qla.qtbzn.cn/88404.Doc
m.qla.qtbzn.cn/15935.Doc
m.qla.qtbzn.cn/64268.Doc
m.qla.qtbzn.cn/73737.Doc
m.qla.qtbzn.cn/33977.Doc
m.qla.qtbzn.cn/13591.Doc
m.qla.qtbzn.cn/06868.Doc
m.qla.qtbzn.cn/26062.Doc
m.qla.qtbzn.cn/80266.Doc
m.qla.qtbzn.cn/20048.Doc
m.qla.qtbzn.cn/13317.Doc
m.qla.qtbzn.cn/44424.Doc
m.qla.qtbzn.cn/28402.Doc
m.qlp.qtbzn.cn/84426.Doc
m.qlp.qtbzn.cn/86486.Doc
m.qlp.qtbzn.cn/46240.Doc
m.qlp.qtbzn.cn/13911.Doc
m.qlp.qtbzn.cn/77315.Doc
m.qlp.qtbzn.cn/37375.Doc
m.qlp.qtbzn.cn/22086.Doc
m.qlp.qtbzn.cn/00200.Doc
m.qlp.qtbzn.cn/33531.Doc
m.qlp.qtbzn.cn/24400.Doc
m.qlp.qtbzn.cn/31179.Doc
m.qlp.qtbzn.cn/19739.Doc
m.qlp.qtbzn.cn/40224.Doc
m.qlp.qtbzn.cn/24626.Doc
m.qlp.qtbzn.cn/28200.Doc
m.qlp.qtbzn.cn/68608.Doc
m.qlp.qtbzn.cn/66842.Doc
m.qlp.qtbzn.cn/86624.Doc
m.qlp.qtbzn.cn/64448.Doc
m.qlp.qtbzn.cn/24828.Doc
m.qlo.qtbzn.cn/60440.Doc
m.qlo.qtbzn.cn/28468.Doc
m.qlo.qtbzn.cn/00680.Doc
m.qlo.qtbzn.cn/06284.Doc
m.qlo.qtbzn.cn/62062.Doc
m.qlo.qtbzn.cn/60880.Doc
m.qlo.qtbzn.cn/64226.Doc
m.qlo.qtbzn.cn/02604.Doc
m.qlo.qtbzn.cn/48286.Doc
m.qlo.qtbzn.cn/22648.Doc
m.qlo.qtbzn.cn/75379.Doc
m.qlo.qtbzn.cn/44408.Doc
m.qlo.qtbzn.cn/64204.Doc
m.qlo.qtbzn.cn/42282.Doc
m.qlo.qtbzn.cn/20222.Doc
m.qlo.qtbzn.cn/04048.Doc
m.qlo.qtbzn.cn/15391.Doc
m.qlo.qtbzn.cn/77333.Doc
m.qlo.qtbzn.cn/68642.Doc
m.qlo.qtbzn.cn/17391.Doc
m.qli.qtbzn.cn/08624.Doc
m.qli.qtbzn.cn/20864.Doc
m.qli.qtbzn.cn/20286.Doc
m.qli.qtbzn.cn/46428.Doc
m.qli.qtbzn.cn/08442.Doc
m.qli.qtbzn.cn/46440.Doc
m.qli.qtbzn.cn/60842.Doc
m.qli.qtbzn.cn/13173.Doc
m.qli.qtbzn.cn/42284.Doc
m.qli.qtbzn.cn/48680.Doc
m.qli.qtbzn.cn/44482.Doc
m.qli.qtbzn.cn/62680.Doc
m.qli.qtbzn.cn/44400.Doc
m.qli.qtbzn.cn/60026.Doc
m.qli.qtbzn.cn/60862.Doc
m.qli.qtbzn.cn/28482.Doc
m.qli.qtbzn.cn/64880.Doc
m.qli.qtbzn.cn/80688.Doc
m.qli.qtbzn.cn/84280.Doc
m.qli.qtbzn.cn/24406.Doc
m.qlu.qtbzn.cn/80626.Doc
m.qlu.qtbzn.cn/73571.Doc
m.qlu.qtbzn.cn/22202.Doc
m.qlu.qtbzn.cn/40862.Doc
m.qlu.qtbzn.cn/35777.Doc
m.qlu.qtbzn.cn/53337.Doc
m.qlu.qtbzn.cn/62024.Doc
m.qlu.qtbzn.cn/02486.Doc
m.qlu.qtbzn.cn/22682.Doc
m.qlu.qtbzn.cn/62084.Doc
m.qlu.qtbzn.cn/20642.Doc
m.qlu.qtbzn.cn/77533.Doc
m.qlu.qtbzn.cn/99937.Doc
m.qlu.qtbzn.cn/20626.Doc
m.qlu.qtbzn.cn/11991.Doc
m.qlu.qtbzn.cn/35995.Doc
m.qlu.qtbzn.cn/00846.Doc
m.qlu.qtbzn.cn/02628.Doc
m.qlu.qtbzn.cn/46682.Doc
m.qlu.qtbzn.cn/22620.Doc
m.qly.qtbzn.cn/37759.Doc
m.qly.qtbzn.cn/20420.Doc
m.qly.qtbzn.cn/40480.Doc
m.qly.qtbzn.cn/88044.Doc
m.qly.qtbzn.cn/66640.Doc
m.qly.qtbzn.cn/68682.Doc
m.qly.qtbzn.cn/02860.Doc
m.qly.qtbzn.cn/26004.Doc
m.qly.qtbzn.cn/59919.Doc
m.qly.qtbzn.cn/68040.Doc
m.qly.qtbzn.cn/84260.Doc
m.qly.qtbzn.cn/84266.Doc
m.qly.qtbzn.cn/48246.Doc
m.qly.qtbzn.cn/22624.Doc
m.qly.qtbzn.cn/68080.Doc
m.qly.qtbzn.cn/31793.Doc
m.qly.qtbzn.cn/37735.Doc
m.qly.qtbzn.cn/28248.Doc
m.qly.qtbzn.cn/22842.Doc
m.qly.qtbzn.cn/26244.Doc
m.qlt.qtbzn.cn/26882.Doc
m.qlt.qtbzn.cn/04824.Doc
m.qlt.qtbzn.cn/97131.Doc
m.qlt.qtbzn.cn/08468.Doc
m.qlt.qtbzn.cn/22060.Doc
m.qlt.qtbzn.cn/82288.Doc
m.qlt.qtbzn.cn/64080.Doc
m.qlt.qtbzn.cn/15993.Doc
m.qlt.qtbzn.cn/86826.Doc
m.qlt.qtbzn.cn/64042.Doc
m.qlt.qtbzn.cn/80086.Doc
m.qlt.qtbzn.cn/71993.Doc
m.qlt.qtbzn.cn/60266.Doc
m.qlt.qtbzn.cn/64422.Doc
m.qlt.qtbzn.cn/77557.Doc
m.qlt.qtbzn.cn/00248.Doc
m.qlt.qtbzn.cn/08462.Doc
m.qlt.qtbzn.cn/40844.Doc
