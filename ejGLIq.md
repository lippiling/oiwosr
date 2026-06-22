凶貉诜味妨


Linux sched_core核心调度cookie匹配与强制idle

sched_core是CONFIG_SCHED_CORE引入的SMT同步调度机制，解决超线程环境下同核两个硬件线程执行不同trust domain任务的侧信道安全问题。每个task_struct携带一个unsigned long cookie（core_cookie），通过prctl(PR_SET_CORE_SCHED_CTX)或cgroup的cpu.core_tag接口设置。核心调度决策的核心约束是：同一个物理核心上的两个硬件线程（CPU）必须同时运行cookie相同的任务，否则其中一个线程必须强制idle。

```c
struct task_struct {
#ifdef CONFIG_SCHED_CORE
    unsigned long           core_cookie;    /* 核心调度cookie */
#endif
};

struct rq {
#ifdef CONFIG_SCHED_CORE
    unsigned int            core_enabled;   /* 该CPU是否开启了核心调度 */
    unsigned int            core_forceidle; /* 是否处于强制idle状态 */
    unsigned int            core_forceidle_occupation;
    struct cpumask          *core_pick_mask; /* 已挑选的任务集合 */
#endif
};
```

核心调度的任务选择入口在__schedule()的pick_next_task()中。标准pick_next_task经过stop_sched_class -> idle_sched_class -> ... -> fair_sched_class的优先级遍历，在sched_core模式下被替换为sched_core_pick_next_task()。该函数为同一个物理核心上的每个CPU选择任务时，强制校验cookie匹配约束。

```c
#ifdef CONFIG_SCHED_CORE
static struct task_struct *
sched_core_pick_next_task(struct rq *rq)
{
    struct task_struct *next, *p;
    struct rq *core_rq;
    int cpu, core_cpu = cpu_of(rq);

    if (!rq->core_enabled)
        return pick_next_task(rq);

    core_rq = rq->core_rq;  /* 指向同核master rq */
    if (rq != core_rq)
        return core_rq->core_pick_task;  /* 从伙伴CPU的pick结果获取 */

    /* master CPU负责为整个核心做cookie匹配 */
    for_each_cpu_and(cpu, cpu_smt_mask(core_cpu), cpu_online_mask) {
        struct rq *sibling_rq = cpu_rq(cpu);
        struct task_struct *sibling_curr = sibling_rq->curr;

        /* 优先选择cookie匹配的任务 */
        next = pick_next_task(sibling_rq);
        if (next && next->core_cookie != sibling_curr->core_cookie) {
            /* cookie不匹配：查找匹配cookie的任务 */
            for_each_class(class) {
                p = class->pick_task(sibling_rq);
                if (p && p->core_cookie == sibling_curr->core_cookie) {
                    next = p;
                    break;
                }
            }
        }

        if (!next || next->core_cookie != core_rq->core_pick_cookie) {
            /* 找不到匹配cookie的任务，强制idle */
            next = idle_sched_class.pick_task(sibling_rq);
            sibling_rq->core_forceidle = 1;
        }

        __set_bit(cpu, core_rq->core_pick_mask);
        core_rq->core_pick_task = next;
    }

    return core_rq->core_pick_task;
}
```

sched_core强制idle的触发条件是：一个CPU选择了某个cookie的任务，但核心上其他CPU在各自runqueue中都找不到相同cookie的可运行任务。此时pick_result中的core_forceidle被置位。强制idle状态下，当前CPU的实际C-state不会进入深睡眠（因为核心上的另一个线程还在运行共享L1 cache），idle thread通过play_dead或mwait的c1e浅睡眠轮询等待cookie匹配的任务到达。

```c
/*
 * sched_core的核心选择逻辑——pick_task迭代所有调度类
 * 返回与给定cookie匹配的任务，找不到则返回NULL
 */
for_each_class(class) {
    p = class->pick_task(rq);
    if (p && (!cookie || p->core_cookie == cookie)) {
        if (cookie)
            cookie_found = true;
        next = p;
        break;
    }
}

/* 如果找不到匹配cookie的任务，强制idle */
if (cookie && !cookie_found) {
    rq->core_forceidle = 1;
    next = idle_sched_class.pick_task(rq);
}
```

cookie匹配的粒度是per-task。当任务通过execve()执行新程序时，core_cookie不会被清除——它继承自父进程的prctl设置。但set_user()或seccomp事件可能通过security_task_alloc()钩子重置cookie。一个常见竞态是：核心上的CPU0持有cookie A的任务正在运行，CPU1的任务完成IO后wakeup但其cookie为B。此时CPU1在scheduler_tick的resched路径中检查到core力core_forceidle标记，选择idle线程。但CPU1的__schedule()需要等待CPU0先完成当前调度周期——因为sched_core要求核心上所有CPU的pick_task在同一个rq->lock临界区内进行。

core scheduler的deactivation路径也涉及cookie检查。当任务调用deactivate_task()（如退出或阻塞）时，如果该任务是核心上最后一个携带特定cookie的任务，则核心上所有CPU必须被重新pick_task——否则另一个CPU可能仍然拿着该cookie的idle强制并在等待一个已退出的任务。dequeue_task中通过sched_core_dequeue()检查是否需要触发core re-pick：

```c
static inline void sched_core_dequeue(struct rq *rq, struct task_struct *p)
{
    if (!sched_core_enabled(rq))
        return;

    /*
     * 如果该任务是当前核心上该cookie的唯一任务，
     * 标记core resched让所有CPU重新pick
     */
    if (p->core_cookie && task_on_rq_queued(p)) {
        struct task_struct *tmp;
        bool last = true;

        for_each_online_cpu(cpu) {
            if (cpu == cpu_of(rq))
                continue;
            tmp = cpu_rq(cpu)->curr;
            if (tmp->core_cookie == p->core_cookie && task_on_rq_queued(tmp)) {
                last = false;
                break;
            }
        }

        if (last || rq->core_forceidle) {
            /* 唤醒所有SMT兄弟CPU重新选择 */
            smt_mask = cpu_smt_mask(cpu_of(rq));
            for_each_cpu_and(i, smt_mask, cpu_online_mask)
                resched_curr(cpu_rq(i));
        }
    }
}
```

sched_core的migration偏移处理存在一个显著性能折衷。当任务从cookie A的CPU migrate到cookie B的CPU时（通过load balance），它必须等待核心上其他所有CPU的当前任务完成——因为pick_next_task时无法在同一核心上混合不同cookie。社区解决方向是引入core-wide wakeup，即wakeup路径检测到目标核心上正在运行不同cookie的任务时，不立即唤醒而延迟到核心空闲。这通过TTWU_QUEUED -> try_steal_cookie机制实现，但仍处于experimental状态。

另一个边界条件涉及PI优先级继承。持有mutex的任务cookie为A但被cookie B的高优先级任务等待时，核心调度无法直接进行——因为mutex unlock发生在不同cookie的上下文。sched_core通过core_waiters计数器追踪：当核心上等待锁的线程与持有者cookie不同时，强制idle策略不应生效（任务必须运行才能释放锁），否则会导致ABBA死锁。
霖嗽沾弊九纷磺备讶腿诙伦锨艘幌

rip.sthxr.cn/000883.htm
rip.sthxr.cn/195973.htm
rip.sthxr.cn/242883.htm
rip.sthxr.cn/480083.htm
rip.sthxr.cn/608843.htm
rio.sthxr.cn/204203.htm
rio.sthxr.cn/206243.htm
rio.sthxr.cn/913553.htm
rio.sthxr.cn/317373.htm
rio.sthxr.cn/266643.htm
rio.sthxr.cn/802643.htm
rio.sthxr.cn/755713.htm
rio.sthxr.cn/751993.htm
rio.sthxr.cn/868643.htm
rio.sthxr.cn/137173.htm
rii.sthxr.cn/111393.htm
rii.sthxr.cn/979713.htm
rii.sthxr.cn/935113.htm
rii.sthxr.cn/622463.htm
rii.sthxr.cn/515913.htm
rii.sthxr.cn/244003.htm
rii.sthxr.cn/935133.htm
rii.sthxr.cn/551513.htm
rii.sthxr.cn/131133.htm
rii.sthxr.cn/682243.htm
riu.sthxr.cn/355193.htm
riu.sthxr.cn/842863.htm
riu.sthxr.cn/440023.htm
riu.sthxr.cn/711133.htm
riu.sthxr.cn/537933.htm
riu.sthxr.cn/222423.htm
riu.sthxr.cn/573393.htm
riu.sthxr.cn/555973.htm
riu.sthxr.cn/680483.htm
riu.sthxr.cn/357733.htm
riy.sthxr.cn/331193.htm
riy.sthxr.cn/824683.htm
riy.sthxr.cn/888423.htm
riy.sthxr.cn/991793.htm
riy.sthxr.cn/157733.htm
riy.sthxr.cn/004023.htm
riy.sthxr.cn/397333.htm
riy.sthxr.cn/153153.htm
riy.sthxr.cn/406803.htm
riy.sthxr.cn/555353.htm
rit.sthxr.cn/464043.htm
rit.sthxr.cn/719593.htm
rit.sthxr.cn/713373.htm
rit.sthxr.cn/155133.htm
rit.sthxr.cn/244683.htm
rit.sthxr.cn/993753.htm
rit.sthxr.cn/820243.htm
rit.sthxr.cn/864403.htm
rit.sthxr.cn/157753.htm
rit.sthxr.cn/206483.htm
rir.sthxr.cn/666863.htm
rir.sthxr.cn/997393.htm
rir.sthxr.cn/971553.htm
rir.sthxr.cn/644403.htm
rir.sthxr.cn/713933.htm
rir.sthxr.cn/353133.htm
rir.sthxr.cn/559553.htm
rir.sthxr.cn/066463.htm
rir.sthxr.cn/062483.htm
rir.sthxr.cn/026623.htm
rie.sthxr.cn/842083.htm
rie.sthxr.cn/624643.htm
rie.sthxr.cn/646263.htm
rie.sthxr.cn/440623.htm
rie.sthxr.cn/022223.htm
rie.sthxr.cn/440283.htm
rie.sthxr.cn/557373.htm
rie.sthxr.cn/117773.htm
rie.sthxr.cn/408423.htm
rie.sthxr.cn/593593.htm
riw.sthxr.cn/551333.htm
riw.sthxr.cn/280603.htm
riw.sthxr.cn/313353.htm
riw.sthxr.cn/082483.htm
riw.sthxr.cn/406883.htm
riw.sthxr.cn/840663.htm
riw.sthxr.cn/775933.htm
riw.sthxr.cn/400603.htm
riw.sthxr.cn/951753.htm
riw.sthxr.cn/864023.htm
riq.sthxr.cn/046643.htm
riq.sthxr.cn/866863.htm
riq.sthxr.cn/424643.htm
riq.sthxr.cn/995753.htm
riq.sthxr.cn/173133.htm
riq.sthxr.cn/408483.htm
riq.sthxr.cn/664603.htm
riq.sthxr.cn/173333.htm
riq.sthxr.cn/931973.htm
riq.sthxr.cn/002463.htm
rum.sthxr.cn/553573.htm
rum.sthxr.cn/488443.htm
rum.sthxr.cn/202043.htm
rum.sthxr.cn/220223.htm
rum.sthxr.cn/777373.htm
rum.sthxr.cn/171133.htm
rum.sthxr.cn/626263.htm
rum.sthxr.cn/428463.htm
rum.sthxr.cn/084443.htm
rum.sthxr.cn/131353.htm
run.sthxr.cn/480683.htm
run.sthxr.cn/220463.htm
run.sthxr.cn/111993.htm
run.sthxr.cn/664463.htm
run.sthxr.cn/444683.htm
run.sthxr.cn/240663.htm
run.sthxr.cn/717533.htm
run.sthxr.cn/555533.htm
run.sthxr.cn/715713.htm
run.sthxr.cn/571113.htm
rub.sthxr.cn/844063.htm
rub.sthxr.cn/604623.htm
rub.sthxr.cn/244283.htm
rub.sthxr.cn/315553.htm
rub.sthxr.cn/248843.htm
rub.sthxr.cn/531753.htm
rub.sthxr.cn/153793.htm
rub.sthxr.cn/939593.htm
rub.sthxr.cn/026803.htm
rub.sthxr.cn/462643.htm
ruv.sthxr.cn/593953.htm
ruv.sthxr.cn/628223.htm
ruv.sthxr.cn/557913.htm
ruv.sthxr.cn/571953.htm
ruv.sthxr.cn/199573.htm
ruv.sthxr.cn/022843.htm
ruv.sthxr.cn/082443.htm
ruv.sthxr.cn/466403.htm
ruv.sthxr.cn/882603.htm
ruv.sthxr.cn/193113.htm
ruc.sthxr.cn/397993.htm
ruc.sthxr.cn/177573.htm
ruc.sthxr.cn/337733.htm
ruc.sthxr.cn/377533.htm
ruc.sthxr.cn/955113.htm
ruc.sthxr.cn/975133.htm
ruc.sthxr.cn/959753.htm
ruc.sthxr.cn/024043.htm
ruc.sthxr.cn/862823.htm
ruc.sthxr.cn/731133.htm
rux.sthxr.cn/193173.htm
rux.sthxr.cn/804823.htm
rux.sthxr.cn/397573.htm
rux.sthxr.cn/399373.htm
rux.sthxr.cn/004423.htm
rux.sthxr.cn/391193.htm
rux.sthxr.cn/242243.htm
rux.sthxr.cn/173313.htm
rux.sthxr.cn/379593.htm
rux.sthxr.cn/604823.htm
ruz.sthxr.cn/608603.htm
ruz.sthxr.cn/577113.htm
ruz.sthxr.cn/535953.htm
ruz.sthxr.cn/848863.htm
ruz.sthxr.cn/028663.htm
ruz.sthxr.cn/195973.htm
ruz.sthxr.cn/266283.htm
ruz.sthxr.cn/955993.htm
ruz.sthxr.cn/024643.htm
ruz.sthxr.cn/808063.htm
rul.sthxr.cn/997913.htm
rul.sthxr.cn/913533.htm
rul.sthxr.cn/155353.htm
rul.sthxr.cn/599733.htm
rul.sthxr.cn/668843.htm
rul.sthxr.cn/808403.htm
rul.sthxr.cn/513993.htm
rul.sthxr.cn/022483.htm
rul.sthxr.cn/224823.htm
rul.sthxr.cn/915133.htm
ruk.sthxr.cn/133113.htm
ruk.sthxr.cn/666683.htm
ruk.sthxr.cn/648603.htm
ruk.sthxr.cn/480883.htm
ruk.sthxr.cn/115913.htm
ruk.sthxr.cn/426423.htm
ruk.sthxr.cn/555793.htm
ruk.sthxr.cn/808243.htm
ruk.sthxr.cn/995553.htm
ruk.sthxr.cn/377113.htm
ruj.sthxr.cn/068863.htm
ruj.sthxr.cn/482863.htm
ruj.sthxr.cn/228423.htm
ruj.sthxr.cn/575713.htm
ruj.sthxr.cn/375353.htm
ruj.sthxr.cn/202643.htm
ruj.sthxr.cn/573513.htm
ruj.sthxr.cn/420483.htm
ruj.sthxr.cn/331973.htm
ruj.sthxr.cn/026223.htm
ruh.sthxr.cn/353713.htm
ruh.sthxr.cn/539333.htm
ruh.sthxr.cn/862283.htm
ruh.sthxr.cn/115913.htm
ruh.sthxr.cn/668883.htm
ruh.sthxr.cn/424823.htm
ruh.sthxr.cn/375333.htm
ruh.sthxr.cn/155733.htm
ruh.sthxr.cn/866843.htm
ruh.sthxr.cn/484883.htm
rug.sthxr.cn/006643.htm
rug.sthxr.cn/800083.htm
rug.sthxr.cn/846043.htm
rug.sthxr.cn/248683.htm
rug.sthxr.cn/484623.htm
rug.sthxr.cn/060683.htm
rug.sthxr.cn/664603.htm
rug.sthxr.cn/620243.htm
rug.sthxr.cn/062883.htm
rug.sthxr.cn/175393.htm
ruf.sthxr.cn/395393.htm
ruf.sthxr.cn/135513.htm
ruf.sthxr.cn/731393.htm
ruf.sthxr.cn/024443.htm
ruf.sthxr.cn/662403.htm
ruf.sthxr.cn/111713.htm
ruf.sthxr.cn/951593.htm
ruf.sthxr.cn/737193.htm
ruf.sthxr.cn/280843.htm
ruf.sthxr.cn/355713.htm
rud.sthxr.cn/040463.htm
rud.sthxr.cn/408223.htm
rud.sthxr.cn/733953.htm
rud.sthxr.cn/571593.htm
rud.sthxr.cn/666603.htm
rud.sthxr.cn/917353.htm
rud.sthxr.cn/268463.htm
rud.sthxr.cn/715713.htm
rud.sthxr.cn/688463.htm
rud.sthxr.cn/533973.htm
rus.sthxr.cn/088043.htm
rus.sthxr.cn/933953.htm
rus.sthxr.cn/448223.htm
rus.sthxr.cn/571953.htm
rus.sthxr.cn/284003.htm
rus.sthxr.cn/397513.htm
rus.sthxr.cn/971173.htm
rus.sthxr.cn/280403.htm
rus.sthxr.cn/488463.htm
rus.sthxr.cn/446643.htm
rua.sthxr.cn/177313.htm
rua.sthxr.cn/531393.htm
rua.sthxr.cn/468423.htm
rua.sthxr.cn/264803.htm
rua.sthxr.cn/428403.htm
rua.sthxr.cn/333793.htm
rua.sthxr.cn/482643.htm
rua.sthxr.cn/773513.htm
rua.sthxr.cn/886483.htm
rua.sthxr.cn/935193.htm
rup.sthxr.cn/208463.htm
rup.sthxr.cn/353713.htm
rup.sthxr.cn/206043.htm
rup.sthxr.cn/199153.htm
rup.sthxr.cn/137793.htm
rup.sthxr.cn/375353.htm
rup.sthxr.cn/246603.htm
rup.sthxr.cn/331113.htm
rup.sthxr.cn/913993.htm
rup.sthxr.cn/313113.htm
ruo.sthxr.cn/422003.htm
ruo.sthxr.cn/513333.htm
ruo.sthxr.cn/971953.htm
ruo.sthxr.cn/668283.htm
ruo.sthxr.cn/802683.htm
ruo.sthxr.cn/553593.htm
ruo.sthxr.cn/379593.htm
ruo.sthxr.cn/460443.htm
ruo.sthxr.cn/802843.htm
ruo.sthxr.cn/400643.htm
rui.sthxr.cn/868243.htm
rui.sthxr.cn/113333.htm
rui.sthxr.cn/824403.htm
rui.sthxr.cn/799113.htm
rui.sthxr.cn/113373.htm
rui.sthxr.cn/802883.htm
rui.sthxr.cn/608803.htm
rui.sthxr.cn/640863.htm
rui.sthxr.cn/753793.htm
rui.sthxr.cn/064663.htm
ruu.sthxr.cn/862863.htm
ruu.sthxr.cn/268803.htm
ruu.sthxr.cn/771353.htm
ruu.sthxr.cn/995753.htm
ruu.sthxr.cn/048203.htm
ruu.sthxr.cn/173113.htm
ruu.sthxr.cn/559173.htm
ruu.sthxr.cn/337313.htm
ruu.sthxr.cn/442623.htm
ruu.sthxr.cn/266663.htm
ruy.sthxr.cn/864483.htm
ruy.sthxr.cn/828663.htm
ruy.sthxr.cn/002243.htm
ruy.sthxr.cn/199553.htm
ruy.sthxr.cn/620443.htm
ruy.sthxr.cn/775153.htm
ruy.sthxr.cn/771733.htm
ruy.sthxr.cn/131973.htm
ruy.sthxr.cn/068223.htm
ruy.sthxr.cn/193333.htm
rut.sthxr.cn/575713.htm
rut.sthxr.cn/373753.htm
rut.sthxr.cn/357153.htm
rut.sthxr.cn/511533.htm
rut.sthxr.cn/395993.htm
rut.sthxr.cn/779393.htm
rut.sthxr.cn/117733.htm
rut.sthxr.cn/937973.htm
rut.sthxr.cn/480823.htm
rut.sthxr.cn/791753.htm
rur.sthxr.cn/593753.htm
rur.sthxr.cn/191933.htm
rur.sthxr.cn/739153.htm
rur.sthxr.cn/284803.htm
rur.sthxr.cn/406883.htm
rur.sthxr.cn/666603.htm
rur.sthxr.cn/062803.htm
rur.sthxr.cn/755753.htm
rur.sthxr.cn/939773.htm
rur.sthxr.cn/975793.htm
rue.sthxr.cn/680223.htm
rue.sthxr.cn/822083.htm
rue.sthxr.cn/711753.htm
rue.sthxr.cn/260803.htm
rue.sthxr.cn/246603.htm
rue.sthxr.cn/793313.htm
rue.sthxr.cn/642883.htm
rue.sthxr.cn/557773.htm
rue.sthxr.cn/444643.htm
rue.sthxr.cn/171993.htm
ruw.sthxr.cn/220443.htm
ruw.sthxr.cn/537713.htm
ruw.sthxr.cn/179713.htm
ruw.sthxr.cn/399953.htm
ruw.sthxr.cn/268023.htm
ruw.sthxr.cn/557133.htm
ruw.sthxr.cn/844483.htm
ruw.sthxr.cn/971353.htm
ruw.sthxr.cn/171773.htm
ruw.sthxr.cn/882863.htm
ruq.sthxr.cn/644483.htm
ruq.sthxr.cn/119393.htm
ruq.sthxr.cn/775753.htm
ruq.sthxr.cn/737973.htm
ruq.sthxr.cn/882003.htm
ruq.sthxr.cn/660203.htm
ruq.sthxr.cn/604203.htm
ruq.sthxr.cn/397713.htm
ruq.sthxr.cn/420403.htm
ruq.sthxr.cn/462823.htm
rym.sthxr.cn/424463.htm
rym.sthxr.cn/597173.htm
rym.sthxr.cn/866843.htm
rym.sthxr.cn/820863.htm
rym.sthxr.cn/537793.htm
rym.sthxr.cn/602243.htm
rym.sthxr.cn/288663.htm
rym.sthxr.cn/864223.htm
rym.sthxr.cn/395793.htm
rym.sthxr.cn/424863.htm
ryn.sthxr.cn/977553.htm
ryn.sthxr.cn/193773.htm
ryn.sthxr.cn/577593.htm
ryn.sthxr.cn/008263.htm
ryn.sthxr.cn/133133.htm
ryn.sthxr.cn/797793.htm
ryn.sthxr.cn/460283.htm
ryn.sthxr.cn/733733.htm
ryn.sthxr.cn/559313.htm
ryn.sthxr.cn/159193.htm
ryb.sthxr.cn/260663.htm
ryb.sthxr.cn/804483.htm
ryb.sthxr.cn/797313.htm
ryb.sthxr.cn/606863.htm
ryb.sthxr.cn/151153.htm
ryb.sthxr.cn/062083.htm
ryb.sthxr.cn/771313.htm
ryb.sthxr.cn/357513.htm
ryb.sthxr.cn/806403.htm
ryb.sthxr.cn/133953.htm
ryv.sthxr.cn/468223.htm
ryv.sthxr.cn/286843.htm
ryv.sthxr.cn/119393.htm
ryv.sthxr.cn/242803.htm
ryv.sthxr.cn/224683.htm
ryv.sthxr.cn/751513.htm
ryv.sthxr.cn/408803.htm
ryv.sthxr.cn/424003.htm
ryv.sthxr.cn/197393.htm
ryv.sthxr.cn/739953.htm
