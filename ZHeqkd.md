染巧啬秸侥


Linux wake_up_new_task新进程唤醒与placement优化

wake_up_new_task()是fork/exec路径中激活新进程的唯一入口，由do_fork()内部调用。与wake_up_process()不同，它不需要检查STATE_RUNNING状态——新进程的state在sched_fork()中被设为TASK_RUNNING但尚未入队，因此wake_up_new_task直接调用select_task_rq()做初始placement后激活任务。

```c
void wake_up_new_task(struct task_struct *p)
{
    struct rq_flags rf;
    struct rq *rq;

    raw_spin_lock_irqsave(&p->pi_lock, rf.flags);
    p->state = TASK_RUNNING;

#ifdef CONFIG_SMP
    __set_task_cpu(p, select_task_rq(p, SD_BALANCE_FORK, 0));
#endif

    rq = __task_rq_lock(p, &rf);
    activate_task(rq, p, ENQUEUE_WAKEUP_NEW);
    p->on_rq = TASK_ON_RQ_QUEUED;
    trace_sched_wakeup_new(p);

    if (p->sched_class->task_woken)
        p->sched_class->task_woken(rq, p);

    check_preempt_curr(rq, p, WF_FORK);
    raw_spin_unlock_rq_unlock(rq, &rf);
}
```

select_task_rq()是决定新进程运行在哪个CPU的核心函数。对于SD_BALANCE_FORK类型调用，select_task_rq_fair()优先尝试wake_affine()——检查waker所在的CPU是否适合放置新进程。wake_affine()的核心判断是waker的CPU负载是否低于该CPU的capacity折算阈值，通过wake_affine_weight()计算一个delta值：

```c
static int wake_wide(struct task_struct *p)
{
    unsigned int master = current->nr_cpus_allowed;
    unsigned int slave = p->nr_cpus_allowed;

    if (master < slave)
        swap(master, slave);

    return master > slave * 2;  /* 当waker和wakee的数量级差异超过2倍时，wake_affine失效 */
}

static int wake_affine(struct sched_domain *sd, struct task_struct *p,
                       int this_cpu, int prev_cpu, int sync)
{
    s64 this_eff, prev_eff;
    unsigned long this_weight, prev_weight;
    int idx = sd->wake_idx;

    this_eff = capacity_of(this_cpu);
    prev_eff = capacity_of(prev_cpu);

    this_weight = sd->span_weight;
    prev_weight = sd->span_weight;

    if (sync) {
        /* waker马上要睡眠，可以减少对waker load的计算 */
        this_eff -= task_util_est(current);
    }

    this_eff *= this_weight;
    prev_eff *= prev_weight;

    return this_eff * sd->imbalance_pct < prev_eff * 100;
}
```

find_idest_cpu()在wake_affine()返回false时接管placement决策。该函数在sched domain层次从最低层（SMT）向上遍历，每一层按idle state排序挑选最空闲的CPU。具体的idle_cpu()判断逻辑检查cpu_idle_state == CPU_IDLE（中断开启的空闲）或CPU_NEWLY_IDLE，排除了CPU_IDLE_ENTERED（已关闭中断进入WFI）的候选。

```c
static int find_idest_cpu(struct sched_group *group, struct task_struct *p,
                          int this_cpu)
{
    unsigned int min_exit_latency = UINT_MAX;
    unsigned int compute_capacity = 0;
    int idlest_cpu = -1, idlest_group_cpu = -1;
    struct sched_group *sg;
    int i;

    for_each_sched_group(sg) {
        if (!sched_group_cookie_match(p, sg))
            continue;

        if (local_group && sg->sgc->capacity * imbalance_pct <
                          group->sgc->capacity * 100)
            continue;

        for_each_cpu_and(i, sched_group_span(sg), p->cpus_ptr) {
            if (available_idle_cpu(i) || sched_idle_cpu(i)) {
                if (idle_cpu(i) && !idle_cpu(idlest_cpu)) {
                    idlest_cpu = i;
                }
            }
        }
    }

    if (idlest_cpu == -1)
        idlest_cpu = this_cpu;

    return idlest_cpu;
}
```

activate_task()将新任务通过enqueue_task_fair()插入CFS红黑树。对于WF_TWEAKEUP_NEW标志，enqueue_entity()会跳过slice补偿逻辑并直接将se的vruntime初始化为cfs_rq->min_vruntime减去一个基于任务nice值的唤醒补偿。这个补偿通过sched_vslice()计算：

```c
static void
enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    bool renorm = !(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATED);
    bool inc = true;

    if (renorm) {
        /*
         * 新进程的vruntime不能直接设为0，否则会
         * 抢占所有老进程。初始vruntime = cfs_rq->min_vruntime
         * 减去一个nice相关的补偿量，实现"善意"调度
         */
        se->vruntime = cfs_rq->min_vruntime;

        if (flags & ENQUEUE_WAKEUP_NEW) {
            se->vruntime -= sched_vslice(cfs_rq, se);
        }
        se->prev_sum_exec_runtime = 0;
    }

    update_curr(cfs_rq);
    account_entity_enqueue(cfs_rq, se);

    if (flags & ENQUEUE_WAKEUP_NEW) {
        place_entity(cfs_rq, se, 0);
    }

    if (se != cfs_rq->curr)
        __enqueue_entity(cfs_rq, se);
    se->on_rq = 1;
}
```

place_entity()是新进程vruntime初始化的精确实现。对于fork出来的新进程，它不仅要减去sched_vslice的补偿，还要根据sysctl_sched_child_runs_first来决定是否让新进程排到父进程前面。当child_runs_first置位时，place_entity将新进程的vruntime设为当前任务vruntime减去一个tick粒度，确保新进程在pick_next_task_fair时优于父进程。

```c
static void place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
{
    u64 vruntime = cfs_rq->min_vruntime;

    if (initial && sched_feat(START_DEBIT))
        vruntime += sched_vslice(cfs_rq, se);

    if (!initial) {
        unsigned long thresh = sysctl_sched_wakeup_granularity;

        if (se->vruntime < vruntime)
            thresh += se->vruntime - vruntime;

        if (se->vruntime > vruntime)
            thresh -= se->vruntime - vruntime;

        if (thresh < sysctl_sched_wakeup_granularity) {
            vruntime = max(se->vruntime, vruntime);
        }
    }

    se->vruntime = vruntime;
}
```

check_preempt_curr()在wake_up_new_task末尾执行，比较新进程与当前正在运行进程的vruntime。若新进程的vruntime更小（即优先级更高），则调用resched_curr()触发抢占。这在fork执行batch workload场景下非常关键：父进程fork大量子进程时，每个新进程入队后都会试图抢占父进程，导致父进程被频繁踢出CPU。一种优化手段是在check_preempt_curr()中判断当前任务与新进程的调度类相同且具备RUNNING属性时，使用wakeup_preempt_entity()计算delta是否超过wakeup_granularity阈值而非直接抢占。

SMP迁移路径中的一个竞态：wake_up_new_task中__set_task_cpu()在持有pi_lock时修改了task_struct->cpu，而activate_task内部调用enqueue_task_fair时会通过rq_of(cfs_rq)获取rq指针。如果在__set_task_cpu之后但在activate_task之前被另一个核上的migration thread看到，后者可能对该任务发起无效的pull操作——这就是为什么wake_up_new_task在调用activate_task前没有任何sched_class->dequeue_task的时机，因此migration thread通过task_on_rq_migrating()检测该状态来规避。
等灸了济檬陀独笆慕谄酝涣恃局妹

waz.sthxr.cn/713313.htm
waz.sthxr.cn/711313.htm
waz.sthxr.cn/791113.htm
waz.sthxr.cn/137393.htm
waz.sthxr.cn/644263.htm
wal.sthxr.cn/006823.htm
wal.sthxr.cn/088463.htm
wal.sthxr.cn/157533.htm
wal.sthxr.cn/315533.htm
wal.sthxr.cn/228863.htm
wal.sthxr.cn/802283.htm
wal.sthxr.cn/444003.htm
wal.sthxr.cn/599513.htm
wal.sthxr.cn/202443.htm
wal.sthxr.cn/759993.htm
wak.sthxr.cn/933593.htm
wak.sthxr.cn/602003.htm
wak.sthxr.cn/062023.htm
wak.sthxr.cn/024483.htm
wak.sthxr.cn/377913.htm
wak.sthxr.cn/177173.htm
wak.sthxr.cn/591333.htm
wak.sthxr.cn/462843.htm
wak.sthxr.cn/840423.htm
wak.sthxr.cn/191793.htm
waj.sthxr.cn/248603.htm
waj.sthxr.cn/771113.htm
waj.sthxr.cn/793113.htm
waj.sthxr.cn/866463.htm
waj.sthxr.cn/080823.htm
waj.sthxr.cn/886843.htm
waj.sthxr.cn/919393.htm
waj.sthxr.cn/591393.htm
waj.sthxr.cn/000683.htm
waj.sthxr.cn/020423.htm
wah.sthxr.cn/686043.htm
wah.sthxr.cn/151913.htm
wah.sthxr.cn/399973.htm
wah.sthxr.cn/848643.htm
wah.sthxr.cn/042803.htm
wah.sthxr.cn/884863.htm
wah.sthxr.cn/991733.htm
wah.sthxr.cn/731713.htm
wah.sthxr.cn/799573.htm
wah.sthxr.cn/404243.htm
wag.sthxr.cn/600283.htm
wag.sthxr.cn/488463.htm
wag.sthxr.cn/822243.htm
wag.sthxr.cn/951953.htm
wag.sthxr.cn/913133.htm
wag.sthxr.cn/202203.htm
wag.sthxr.cn/080023.htm
wag.sthxr.cn/602663.htm
wag.sthxr.cn/715773.htm
wag.sthxr.cn/593573.htm
waf.sthxr.cn/640243.htm
waf.sthxr.cn/846883.htm
waf.sthxr.cn/808263.htm
waf.sthxr.cn/195733.htm
waf.sthxr.cn/539193.htm
waf.sthxr.cn/806263.htm
waf.sthxr.cn/977913.htm
waf.sthxr.cn/024843.htm
waf.sthxr.cn/715773.htm
waf.sthxr.cn/046463.htm
wad.sthxr.cn/377933.htm
wad.sthxr.cn/642883.htm
wad.sthxr.cn/486063.htm
wad.sthxr.cn/915513.htm
wad.sthxr.cn/002023.htm
wad.sthxr.cn/951733.htm
wad.sthxr.cn/571333.htm
wad.sthxr.cn/420023.htm
wad.sthxr.cn/480883.htm
wad.sthxr.cn/886843.htm
was.sthxr.cn/933593.htm
was.sthxr.cn/357513.htm
was.sthxr.cn/864403.htm
was.sthxr.cn/511993.htm
was.sthxr.cn/220663.htm
was.sthxr.cn/719133.htm
was.sthxr.cn/715993.htm
was.sthxr.cn/313733.htm
was.sthxr.cn/466203.htm
was.sthxr.cn/042043.htm
waa.sthxr.cn/084283.htm
waa.sthxr.cn/606443.htm
waa.sthxr.cn/915593.htm
waa.sthxr.cn/939713.htm
waa.sthxr.cn/406463.htm
waa.sthxr.cn/200083.htm
waa.sthxr.cn/642223.htm
waa.sthxr.cn/155313.htm
waa.sthxr.cn/975573.htm
waa.sthxr.cn/973733.htm
wap.sthxr.cn/682283.htm
wap.sthxr.cn/442663.htm
wap.sthxr.cn/024023.htm
wap.sthxr.cn/860223.htm
wap.sthxr.cn/971773.htm
wap.sthxr.cn/771593.htm
wap.sthxr.cn/151573.htm
wap.sthxr.cn/882263.htm
wap.sthxr.cn/808863.htm
wap.sthxr.cn/335373.htm
wao.sthxr.cn/595593.htm
wao.sthxr.cn/553553.htm
wao.sthxr.cn/315773.htm
wao.sthxr.cn/973953.htm
wao.sthxr.cn/199913.htm
wao.sthxr.cn/995733.htm
wao.sthxr.cn/753133.htm
wao.sthxr.cn/535353.htm
wao.sthxr.cn/820463.htm
wao.sthxr.cn/177153.htm
wai.sthxr.cn/179353.htm
wai.sthxr.cn/151593.htm
wai.sthxr.cn/268283.htm
wai.sthxr.cn/139713.htm
wai.sthxr.cn/591913.htm
wai.sthxr.cn/288223.htm
wai.sthxr.cn/084803.htm
wai.sthxr.cn/602003.htm
wai.sthxr.cn/919353.htm
wai.sthxr.cn/513993.htm
wau.sthxr.cn/826063.htm
wau.sthxr.cn/779733.htm
wau.sthxr.cn/464203.htm
wau.sthxr.cn/317173.htm
wau.sthxr.cn/860663.htm
wau.sthxr.cn/993733.htm
wau.sthxr.cn/931173.htm
wau.sthxr.cn/488603.htm
wau.sthxr.cn/755953.htm
wau.sthxr.cn/119333.htm
way.sthxr.cn/711973.htm
way.sthxr.cn/939333.htm
way.sthxr.cn/224003.htm
way.sthxr.cn/353773.htm
way.sthxr.cn/400083.htm
way.sthxr.cn/559313.htm
way.sthxr.cn/933593.htm
way.sthxr.cn/064823.htm
way.sthxr.cn/311313.htm
way.sthxr.cn/264663.htm
wat.sthxr.cn/935773.htm
wat.sthxr.cn/735113.htm
wat.sthxr.cn/911393.htm
wat.sthxr.cn/975773.htm
wat.sthxr.cn/840663.htm
wat.sthxr.cn/379393.htm
wat.sthxr.cn/939373.htm
wat.sthxr.cn/515913.htm
wat.sthxr.cn/977513.htm
wat.sthxr.cn/428203.htm
war.sthxr.cn/995733.htm
war.sthxr.cn/422603.htm
war.sthxr.cn/993153.htm
war.sthxr.cn/555393.htm
war.sthxr.cn/048083.htm
war.sthxr.cn/919113.htm
war.sthxr.cn/464403.htm
war.sthxr.cn/931533.htm
war.sthxr.cn/577933.htm
war.sthxr.cn/911373.htm
wae.sthxr.cn/153113.htm
wae.sthxr.cn/668823.htm
wae.sthxr.cn/535933.htm
wae.sthxr.cn/802803.htm
wae.sthxr.cn/711133.htm
wae.sthxr.cn/795133.htm
wae.sthxr.cn/068863.htm
wae.sthxr.cn/939773.htm
wae.sthxr.cn/842443.htm
wae.sthxr.cn/195553.htm
waw.hjiocz.cn/171573.htm
waw.hjiocz.cn/882603.htm
waw.hjiocz.cn/884023.htm
waw.hjiocz.cn/886483.htm
waw.hjiocz.cn/579953.htm
waw.hjiocz.cn/759313.htm
waw.hjiocz.cn/466663.htm
waw.hjiocz.cn/068063.htm
waw.hjiocz.cn/020423.htm
waw.hjiocz.cn/551733.htm
waq.hjiocz.cn/933913.htm
waq.hjiocz.cn/397773.htm
waq.hjiocz.cn/733973.htm
waq.hjiocz.cn/842263.htm
waq.hjiocz.cn/444643.htm
waq.hjiocz.cn/084063.htm
waq.hjiocz.cn/775113.htm
waq.hjiocz.cn/319973.htm
waq.hjiocz.cn/848483.htm
waq.hjiocz.cn/591713.htm
wpm.hjiocz.cn/000023.htm
wpm.hjiocz.cn/791753.htm
wpm.hjiocz.cn/977373.htm
wpm.hjiocz.cn/337133.htm
wpm.hjiocz.cn/771333.htm
wpm.hjiocz.cn/424843.htm
wpm.hjiocz.cn/337913.htm
wpm.hjiocz.cn/373113.htm
wpm.hjiocz.cn/511333.htm
wpm.hjiocz.cn/802483.htm
wpn.hjiocz.cn/808043.htm
wpn.hjiocz.cn/977553.htm
wpn.hjiocz.cn/957513.htm
wpn.hjiocz.cn/977373.htm
wpn.hjiocz.cn/511793.htm
wpn.hjiocz.cn/648423.htm
wpn.hjiocz.cn/519733.htm
wpn.hjiocz.cn/240863.htm
wpn.hjiocz.cn/197553.htm
wpn.hjiocz.cn/975553.htm
wpb.hjiocz.cn/739393.htm
wpb.hjiocz.cn/517353.htm
wpb.hjiocz.cn/228603.htm
wpb.hjiocz.cn/640463.htm
wpb.hjiocz.cn/604883.htm
wpb.hjiocz.cn/395333.htm
wpb.hjiocz.cn/739913.htm
wpb.hjiocz.cn/155733.htm
wpb.hjiocz.cn/808883.htm
wpb.hjiocz.cn/628643.htm
wpv.hjiocz.cn/593793.htm
wpv.hjiocz.cn/539173.htm
wpv.hjiocz.cn/337193.htm
wpv.hjiocz.cn/197113.htm
wpv.hjiocz.cn/682203.htm
wpv.hjiocz.cn/531193.htm
wpv.hjiocz.cn/066843.htm
wpv.hjiocz.cn/373113.htm
wpv.hjiocz.cn/793733.htm
wpv.hjiocz.cn/848643.htm
wpc.hjiocz.cn/066823.htm
wpc.hjiocz.cn/206443.htm
wpc.hjiocz.cn/391173.htm
wpc.hjiocz.cn/539333.htm
wpc.hjiocz.cn/026063.htm
wpc.hjiocz.cn/068283.htm
wpc.hjiocz.cn/886023.htm
wpc.hjiocz.cn/371773.htm
wpc.hjiocz.cn/711373.htm
wpc.hjiocz.cn/975353.htm
wpx.hjiocz.cn/208603.htm
wpx.hjiocz.cn/204223.htm
wpx.hjiocz.cn/604223.htm
wpx.hjiocz.cn/868863.htm
wpx.hjiocz.cn/739353.htm
wpx.hjiocz.cn/599173.htm
wpx.hjiocz.cn/068023.htm
wpx.hjiocz.cn/919113.htm
wpx.hjiocz.cn/973373.htm
wpx.hjiocz.cn/351313.htm
wpz.hjiocz.cn/337993.htm
wpz.hjiocz.cn/000043.htm
wpz.hjiocz.cn/080403.htm
wpz.hjiocz.cn/662203.htm
wpz.hjiocz.cn/335933.htm
wpz.hjiocz.cn/913573.htm
wpz.hjiocz.cn/462083.htm
wpz.hjiocz.cn/397973.htm
wpz.hjiocz.cn/620423.htm
wpz.hjiocz.cn/993393.htm
wpl.hjiocz.cn/208263.htm
wpl.hjiocz.cn/735753.htm
wpl.hjiocz.cn/737133.htm
wpl.hjiocz.cn/686263.htm
wpl.hjiocz.cn/044683.htm
wpl.hjiocz.cn/844843.htm
wpl.hjiocz.cn/177953.htm
wpl.hjiocz.cn/559113.htm
wpl.hjiocz.cn/046423.htm
wpl.hjiocz.cn/173353.htm
wpk.hjiocz.cn/248243.htm
wpk.hjiocz.cn/917973.htm
wpk.hjiocz.cn/317513.htm
wpk.hjiocz.cn/202043.htm
wpk.hjiocz.cn/919113.htm
wpk.hjiocz.cn/626683.htm
wpk.hjiocz.cn/666243.htm
wpk.hjiocz.cn/682283.htm
wpk.hjiocz.cn/155733.htm
wpk.hjiocz.cn/953193.htm
wpj.hjiocz.cn/248483.htm
wpj.hjiocz.cn/008863.htm
wpj.hjiocz.cn/020403.htm
wpj.hjiocz.cn/115333.htm
wpj.hjiocz.cn/911533.htm
wpj.hjiocz.cn/202803.htm
wpj.hjiocz.cn/422403.htm
wpj.hjiocz.cn/200463.htm
wpj.hjiocz.cn/195353.htm
wpj.hjiocz.cn/959793.htm
wph.hjiocz.cn/755593.htm
wph.hjiocz.cn/640423.htm
wph.hjiocz.cn/640043.htm
wph.hjiocz.cn/804243.htm
wph.hjiocz.cn/646683.htm
wph.hjiocz.cn/933533.htm
wph.hjiocz.cn/991713.htm
wph.hjiocz.cn/602663.htm
wph.hjiocz.cn/226263.htm
wph.hjiocz.cn/462883.htm
wpg.hjiocz.cn/379993.htm
wpg.hjiocz.cn/088643.htm
wpg.hjiocz.cn/442423.htm
wpg.hjiocz.cn/593913.htm
wpg.hjiocz.cn/808263.htm
wpg.hjiocz.cn/191553.htm
wpg.hjiocz.cn/355573.htm
wpg.hjiocz.cn/917533.htm
wpg.hjiocz.cn/626063.htm
wpg.hjiocz.cn/206083.htm
wpf.hjiocz.cn/464243.htm
wpf.hjiocz.cn/268403.htm
wpf.hjiocz.cn/511933.htm
wpf.hjiocz.cn/197193.htm
wpf.hjiocz.cn/288203.htm
wpf.hjiocz.cn/222203.htm
wpf.hjiocz.cn/662203.htm
wpf.hjiocz.cn/113953.htm
wpf.hjiocz.cn/933153.htm
wpf.hjiocz.cn/157773.htm
wpd.hjiocz.cn/002043.htm
wpd.hjiocz.cn/680443.htm
wpd.hjiocz.cn/133733.htm
wpd.hjiocz.cn/486243.htm
wpd.hjiocz.cn/337713.htm
wpd.hjiocz.cn/997973.htm
wpd.hjiocz.cn/319713.htm
wpd.hjiocz.cn/933153.htm
wpd.hjiocz.cn/484463.htm
wpd.hjiocz.cn/197373.htm
wps.hjiocz.cn/197133.htm
wps.hjiocz.cn/446443.htm
wps.hjiocz.cn/666843.htm
wps.hjiocz.cn/642063.htm
wps.hjiocz.cn/739713.htm
wps.hjiocz.cn/591153.htm
wps.hjiocz.cn/646443.htm
wps.hjiocz.cn/000803.htm
wps.hjiocz.cn/024043.htm
wps.hjiocz.cn/959333.htm
wpa.hjiocz.cn/955773.htm
wpa.hjiocz.cn/759713.htm
wpa.hjiocz.cn/282263.htm
wpa.hjiocz.cn/680063.htm
wpa.hjiocz.cn/202463.htm
wpa.hjiocz.cn/466683.htm
wpa.hjiocz.cn/159313.htm
wpa.hjiocz.cn/135953.htm
wpa.hjiocz.cn/624603.htm
wpa.hjiocz.cn/175713.htm
wpp.hjiocz.cn/284643.htm
wpp.hjiocz.cn/779793.htm
wpp.hjiocz.cn/755753.htm
wpp.hjiocz.cn/191953.htm
wpp.hjiocz.cn/791753.htm
wpp.hjiocz.cn/240043.htm
wpp.hjiocz.cn/719753.htm
wpp.hjiocz.cn/533933.htm
wpp.hjiocz.cn/484203.htm
wpp.hjiocz.cn/246283.htm
wpo.hjiocz.cn/842243.htm
wpo.hjiocz.cn/080623.htm
wpo.hjiocz.cn/828043.htm
wpo.hjiocz.cn/913773.htm
wpo.hjiocz.cn/551973.htm
wpo.hjiocz.cn/806683.htm
wpo.hjiocz.cn/684283.htm
wpo.hjiocz.cn/286603.htm
wpo.hjiocz.cn/595993.htm
wpo.hjiocz.cn/393593.htm
wpi.hjiocz.cn/822623.htm
wpi.hjiocz.cn/202403.htm
wpi.hjiocz.cn/466403.htm
wpi.hjiocz.cn/739133.htm
wpi.hjiocz.cn/193513.htm
wpi.hjiocz.cn/977733.htm
wpi.hjiocz.cn/240823.htm
wpi.hjiocz.cn/688023.htm
wpi.hjiocz.cn/222283.htm
wpi.hjiocz.cn/264043.htm
wpu.hjiocz.cn/757153.htm
wpu.hjiocz.cn/191193.htm
wpu.hjiocz.cn/266423.htm
wpu.hjiocz.cn/224083.htm
wpu.hjiocz.cn/460683.htm
wpu.hjiocz.cn/353953.htm
wpu.hjiocz.cn/337373.htm
wpu.hjiocz.cn/086283.htm
wpu.hjiocz.cn/484283.htm
wpu.hjiocz.cn/668243.htm
