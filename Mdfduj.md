桌儆到客堆


Linux timer wheel层级时间轮与__run_timers遍历

Linux内核的传统定时器（timer_list）基于层级时间轮算法实现。区别于简单的链表或二叉树，层级时间轮通过多级bucket对不同超时范围进行分级管理，在O(1)时间内完成定时器的插入、删除和到期处理。

timer wheel的核心数据结构定义在kernel/time/timer.c中，每个CPU维护一个tvec_base结构体：

struct tvec_base {
    spinlock_t lock;
    struct timer_list *running_timer;
    unsigned long clk;
    unsigned long next_expiry;
    unsigned long next_expiry_recalc;
    struct tvec_root tv1;
    struct tvec tv2;
    struct tvec tv3;
    struct tvec tv4;
    struct tvec tv5;
};

TVEC_SIZE被定义为64（即每个level包含64个slot）。tv1是第一级，覆盖0-255个jiffies的范围；tv2覆盖256-16383；tv3覆盖16384-1048575；tv4覆盖1048576-67108863；tv5覆盖67108864及以上。tv1中每个slot是一个链表头，直接挂接在该slot到期的定时器；tv2-tv5的每个slot挂接的是待降级的定时器链表。

定时器通过internal_add_timer插入到适当的层级：

static void internal_add_timer(struct tvec_base *base, struct timer_list *timer)
{
    unsigned long expires = timer->expires;
    unsigned long idx = expires - base->clk;
    struct list_head *vec;

    if (idx < TVR_SIZE) {
        int i = expires & TVR_MASK;
        vec = base->tv1.vec + i;
    } else if (idx < 1 << (TVR_BITS + TVN_BITS)) {
        int i = (expires >> TVR_BITS) & TVN_MASK;
        vec = base->tv2.vec + i;
    } else if (idx < 1 << (TVR_BITS + 2 * TVN_BITS)) {
        int i = (expires >> (TVR_BITS + TVN_BITS)) & TVN_MASK;
        vec = base->tv3.vec + i;
    } else if (idx < 1 << (TVR_BITS + 3 * TVN_BITS)) {
        int i = (expires >> (TVR_BITS + 2 * TVN_BITS)) & TVN_MASK;
        vec = base->tv4.vec + i;
    } else if ((signed long)idx < 0) {
        vec = base->tv1.vec + (base->clk & TVR_MASK);
    } else {
        int i = (expires >> (TVR_BITS + 3 * TVN_BITS)) & TVN_MASK;
        vec = base->tv5.vec + i;
    }

    list_add_tail(&timer->entry, vec);
}

插入算法计算expires与base->clk（当前jiffies）之间的差值idx，根据idx范围落入不同的层级。若idx为负数（定时器已过期），直接插入tv1当前slot以便尽快处理。

__run_timers是定时器中断处理函数run_timer_softirq的核心调用，负责遍历当前jiffies对应的tv1 slot中所有到期的定时器：

static inline void __run_timers(struct tvec_base *base)
{
    struct timer_list *timer;

    spin_lock_irq(&base->lock);
    while (time_after_eq(jiffies, base->clk)) {
        struct list_head work_list;
        int index = base->clk & TVR_MASK;

        if (!index &&
            (!cascade(base, &base->tv2, INDEX(0))) &&
            (!cascade(base, &base->tv3, INDEX(1))) &&
            (!cascade(base, &base->tv4, INDEX(2))))
            cascade(base, &base->tv5, INDEX(3));

        list_replace_init(base->tv1.vec + index, &work_list);

        while (!list_empty(&work_list)) {
            void (*fn)(struct timer_list *);
            unsigned long data;

            timer = list_first_entry(&work_list, struct timer_list, entry);
            fn = timer->function;
            data = timer->data;
            list_del_init(&timer->entry);
            detach_if_pending(timer, base);

            base->running_timer = timer;
            spin_unlock_irq(&base->lock);
            call_timer_fn(timer, fn, data);
            spin_lock_irq(&base->lock);
        }
    }
    base->running_timer = NULL;
    spin_unlock_irq(&base->lock);
}

cascade是层级降级的核心操作。当base->clk的低8位（TVR_MASK）归零时，意味着tv1已轮转一圈，需要从tv2中取出对应slot的定时器链表，按照每个定时器的实际expires重新插入到tv1的合适slot中。

static int cascade(struct tvec_base *base, struct tvec *tv, int index)
{
    struct timer_list *timer, *tmp;
    struct list_head tv_list;

    list_replace_init(tv->vec + index, &tv_list);

    list_for_each_entry_safe(timer, tmp, &tv_list, entry)
        internal_add_timer(base, timer);

    return index;
}

cascade逐级向上：tv2归零时触发tv3到tv2的降级，tv3归零触发tv4到tv3的降级，tv4归零触发tv5到tv4的降级。通过这种层级级联机制，高层级的定时器逐渐移动到低层级，最终进入tv1被触发执行。

call_timer_fn是在spinlock释放后执行定时器回调的关键步骤：

static void call_timer_fn(struct timer_list *timer,
                          void (*fn)(struct timer_list *),
                          unsigned long data)
{
    int preempt_count = preempt_count();
    fn(timer);
    if (preempt_count != preempt_count()) {
        WARN_ONCE(1, "timer: %pS preempt leak: %08x -> %08x\n",
                  fn, preempt_count, preempt_count());
    }
}

此处检查preempt_count的平衡，确保定时器回调不会意外改变抢占计数。定时器回调执行时持有base->lock的锁已被释放，避免死锁；但这也要求timer结构体在回调执行期间不会被释放，因此add_timer、del_timer等操作需要配合TIMER_IRQSAFE标志或外部同步机制。

timer wheel的层级设计还面临一个已知问题：tv1只有256个slot，每个slot可能挂载大量定时器，造成单个slot的处理时间过长甚至超过一个jiffy，导致jiffies追赶（jiffies catch-up）现象。为此内核引入了timer_slot_cache和next_expiry_recalc等优化机制，以及在CONFIG_BASE_SMALL配置下动态调整slot数量的能力。


燃讼吠虐嵌毖秩亲纹土冻赫投访是

m.qdc.msfsx.cn/71511.Doc
m.qdc.msfsx.cn/84266.Doc
m.qdc.msfsx.cn/93599.Doc
m.qdc.msfsx.cn/55993.Doc
m.qdc.msfsx.cn/06462.Doc
m.qdc.msfsx.cn/28620.Doc
m.qdc.msfsx.cn/82262.Doc
m.qdc.msfsx.cn/79579.Doc
m.qdc.msfsx.cn/40842.Doc
m.qdc.msfsx.cn/62680.Doc
m.qdc.msfsx.cn/24220.Doc
m.qdc.msfsx.cn/44806.Doc
m.qdc.msfsx.cn/88226.Doc
m.qdc.msfsx.cn/35535.Doc
m.qdc.msfsx.cn/80446.Doc
m.qdc.msfsx.cn/48284.Doc
m.qdc.msfsx.cn/68864.Doc
m.qdx.msfsx.cn/93195.Doc
m.qdx.msfsx.cn/04086.Doc
m.qdx.msfsx.cn/40408.Doc
m.qdx.msfsx.cn/42848.Doc
m.qdx.msfsx.cn/82886.Doc
m.qdx.msfsx.cn/04608.Doc
m.qdx.msfsx.cn/08884.Doc
m.qdx.msfsx.cn/28282.Doc
m.qdx.msfsx.cn/64428.Doc
m.qdx.msfsx.cn/88600.Doc
m.qdx.msfsx.cn/04224.Doc
m.qdx.msfsx.cn/60262.Doc
m.qdx.msfsx.cn/42066.Doc
m.qdx.msfsx.cn/06242.Doc
m.qdx.msfsx.cn/06002.Doc
m.qdx.msfsx.cn/55959.Doc
m.qdx.msfsx.cn/42622.Doc
m.qdx.msfsx.cn/06062.Doc
m.qdx.msfsx.cn/68288.Doc
m.qdx.msfsx.cn/75111.Doc
m.qdz.msfsx.cn/99935.Doc
m.qdz.msfsx.cn/97173.Doc
m.qdz.msfsx.cn/80444.Doc
m.qdz.msfsx.cn/26626.Doc
m.qdz.msfsx.cn/37119.Doc
m.qdz.msfsx.cn/04420.Doc
m.qdz.msfsx.cn/77937.Doc
m.qdz.msfsx.cn/48084.Doc
m.qdz.msfsx.cn/88084.Doc
m.qdz.msfsx.cn/73315.Doc
m.qdz.msfsx.cn/68462.Doc
m.qdz.msfsx.cn/84848.Doc
m.qdz.msfsx.cn/59959.Doc
m.qdz.msfsx.cn/80886.Doc
m.qdz.msfsx.cn/33151.Doc
m.qdz.msfsx.cn/80262.Doc
m.qdz.msfsx.cn/28622.Doc
m.qdz.msfsx.cn/37193.Doc
m.qdz.msfsx.cn/40624.Doc
m.qdz.msfsx.cn/53797.Doc
m.qdl.msfsx.cn/42024.Doc
m.qdl.msfsx.cn/59553.Doc
m.qdl.msfsx.cn/68020.Doc
m.qdl.msfsx.cn/99173.Doc
m.qdl.msfsx.cn/68820.Doc
m.qdl.msfsx.cn/40468.Doc
m.qdl.msfsx.cn/26642.Doc
m.qdl.msfsx.cn/13777.Doc
m.qdl.msfsx.cn/99757.Doc
m.qdl.msfsx.cn/28884.Doc
m.qdl.msfsx.cn/80666.Doc
m.qdl.msfsx.cn/02868.Doc
m.qdl.msfsx.cn/28408.Doc
m.qdl.msfsx.cn/86046.Doc
m.qdl.msfsx.cn/40428.Doc
m.qdl.msfsx.cn/11751.Doc
m.qdl.msfsx.cn/33339.Doc
m.qdl.msfsx.cn/68080.Doc
m.qdl.msfsx.cn/82482.Doc
m.qdl.msfsx.cn/42884.Doc
m.qdk.msfsx.cn/02864.Doc
m.qdk.msfsx.cn/57179.Doc
m.qdk.msfsx.cn/62064.Doc
m.qdk.msfsx.cn/42824.Doc
m.qdk.msfsx.cn/22808.Doc
m.qdk.msfsx.cn/44002.Doc
m.qdk.msfsx.cn/00804.Doc
m.qdk.msfsx.cn/48826.Doc
m.qdk.msfsx.cn/62860.Doc
m.qdk.msfsx.cn/42008.Doc
m.qdk.msfsx.cn/91193.Doc
m.qdk.msfsx.cn/02220.Doc
m.qdk.msfsx.cn/73153.Doc
m.qdk.msfsx.cn/68688.Doc
m.qdk.msfsx.cn/22468.Doc
m.qdk.msfsx.cn/66806.Doc
m.qdk.msfsx.cn/46042.Doc
m.qdk.msfsx.cn/59175.Doc
m.qdk.msfsx.cn/06268.Doc
m.qdk.msfsx.cn/26262.Doc
m.qdj.msfsx.cn/60040.Doc
m.qdj.msfsx.cn/75199.Doc
m.qdj.msfsx.cn/93719.Doc
m.qdj.msfsx.cn/06260.Doc
m.qdj.msfsx.cn/62240.Doc
m.qdj.msfsx.cn/06464.Doc
m.qdj.msfsx.cn/88848.Doc
m.qdj.msfsx.cn/57517.Doc
m.qdj.msfsx.cn/48040.Doc
m.qdj.msfsx.cn/46686.Doc
m.qdj.msfsx.cn/48864.Doc
m.qdj.msfsx.cn/40462.Doc
m.qdj.msfsx.cn/11539.Doc
m.qdj.msfsx.cn/84660.Doc
m.qdj.msfsx.cn/51739.Doc
m.qdj.msfsx.cn/28842.Doc
m.qdj.msfsx.cn/28222.Doc
m.qdj.msfsx.cn/37333.Doc
m.qdj.msfsx.cn/66202.Doc
m.qdj.msfsx.cn/06660.Doc
m.qdh.msfsx.cn/82600.Doc
m.qdh.msfsx.cn/48604.Doc
m.qdh.msfsx.cn/80060.Doc
m.qdh.msfsx.cn/15735.Doc
m.qdh.msfsx.cn/26886.Doc
m.qdh.msfsx.cn/57597.Doc
m.qdh.msfsx.cn/37175.Doc
m.qdh.msfsx.cn/28204.Doc
m.qdh.msfsx.cn/11151.Doc
m.qdh.msfsx.cn/08404.Doc
m.qdh.msfsx.cn/28862.Doc
m.qdh.msfsx.cn/53513.Doc
m.qdh.msfsx.cn/68802.Doc
m.qdh.msfsx.cn/37919.Doc
m.qdh.msfsx.cn/80864.Doc
m.qdh.msfsx.cn/39739.Doc
m.qdh.msfsx.cn/35919.Doc
m.qdh.msfsx.cn/68268.Doc
m.qdh.msfsx.cn/19155.Doc
m.qdh.msfsx.cn/26406.Doc
m.qdg.msfsx.cn/62640.Doc
m.qdg.msfsx.cn/13153.Doc
m.qdg.msfsx.cn/20448.Doc
m.qdg.msfsx.cn/80222.Doc
m.qdg.msfsx.cn/35737.Doc
m.qdg.msfsx.cn/19159.Doc
m.qdg.msfsx.cn/80606.Doc
m.qdg.msfsx.cn/46828.Doc
m.qdg.msfsx.cn/95911.Doc
m.qdg.msfsx.cn/42064.Doc
m.qdg.msfsx.cn/42082.Doc
m.qdg.msfsx.cn/64848.Doc
m.qdg.msfsx.cn/20244.Doc
m.qdg.msfsx.cn/62024.Doc
m.qdg.msfsx.cn/26004.Doc
m.qdg.msfsx.cn/60042.Doc
m.qdg.msfsx.cn/42684.Doc
m.qdg.msfsx.cn/88884.Doc
m.qdg.msfsx.cn/24446.Doc
m.qdg.msfsx.cn/31797.Doc
m.qdf.msfsx.cn/28268.Doc
m.qdf.msfsx.cn/06240.Doc
m.qdf.msfsx.cn/68666.Doc
m.qdf.msfsx.cn/08404.Doc
m.qdf.msfsx.cn/57991.Doc
m.qdf.msfsx.cn/73315.Doc
m.qdf.msfsx.cn/62462.Doc
m.qdf.msfsx.cn/46620.Doc
m.qdf.msfsx.cn/08842.Doc
m.qdf.msfsx.cn/48644.Doc
m.qdf.msfsx.cn/84848.Doc
m.qdf.msfsx.cn/40848.Doc
m.qdf.msfsx.cn/33777.Doc
m.qdf.msfsx.cn/28866.Doc
m.qdf.msfsx.cn/73733.Doc
m.qdf.msfsx.cn/48866.Doc
m.qdf.msfsx.cn/22464.Doc
m.qdf.msfsx.cn/26606.Doc
m.qdf.msfsx.cn/73393.Doc
m.qdf.msfsx.cn/99553.Doc
m.qdd.msfsx.cn/62486.Doc
m.qdd.msfsx.cn/59195.Doc
m.qdd.msfsx.cn/00422.Doc
m.qdd.msfsx.cn/33995.Doc
m.qdd.msfsx.cn/11175.Doc
m.qdd.msfsx.cn/68088.Doc
m.qdd.msfsx.cn/84446.Doc
m.qdd.msfsx.cn/42024.Doc
m.qdd.msfsx.cn/60846.Doc
m.qdd.msfsx.cn/66668.Doc
m.qdd.msfsx.cn/66006.Doc
m.qdd.msfsx.cn/42286.Doc
m.qdd.msfsx.cn/31573.Doc
m.qdd.msfsx.cn/06262.Doc
m.qdd.msfsx.cn/57593.Doc
m.qdd.msfsx.cn/20880.Doc
m.qdd.msfsx.cn/68022.Doc
m.qdd.msfsx.cn/95797.Doc
m.qdd.msfsx.cn/99511.Doc
m.qdd.msfsx.cn/19311.Doc
m.qds.msfsx.cn/20886.Doc
m.qds.msfsx.cn/00602.Doc
m.qds.msfsx.cn/42000.Doc
m.qds.msfsx.cn/64020.Doc
m.qds.msfsx.cn/86004.Doc
m.qds.msfsx.cn/00642.Doc
m.qds.msfsx.cn/28008.Doc
m.qds.msfsx.cn/71159.Doc
m.qds.msfsx.cn/60466.Doc
m.qds.msfsx.cn/88846.Doc
m.qds.msfsx.cn/44646.Doc
m.qds.msfsx.cn/46882.Doc
m.qds.msfsx.cn/15155.Doc
m.qds.msfsx.cn/17159.Doc
m.qds.msfsx.cn/06046.Doc
m.qds.msfsx.cn/48288.Doc
m.qds.msfsx.cn/64244.Doc
m.qds.msfsx.cn/84242.Doc
m.qds.msfsx.cn/00282.Doc
m.qds.msfsx.cn/97117.Doc
m.qda.msfsx.cn/26240.Doc
m.qda.msfsx.cn/79799.Doc
m.qda.msfsx.cn/53175.Doc
m.qda.msfsx.cn/37993.Doc
m.qda.msfsx.cn/04602.Doc
m.qda.msfsx.cn/71951.Doc
m.qda.msfsx.cn/46664.Doc
m.qda.msfsx.cn/40642.Doc
m.qda.msfsx.cn/86280.Doc
m.qda.msfsx.cn/15755.Doc
m.qda.msfsx.cn/17731.Doc
m.qda.msfsx.cn/42824.Doc
m.qda.msfsx.cn/59199.Doc
m.qda.msfsx.cn/22688.Doc
m.qda.msfsx.cn/40480.Doc
m.qda.msfsx.cn/66266.Doc
m.qda.msfsx.cn/97351.Doc
m.qda.msfsx.cn/80808.Doc
m.qda.msfsx.cn/46464.Doc
m.qda.msfsx.cn/28882.Doc
m.qdp.msfsx.cn/28246.Doc
m.qdp.msfsx.cn/97371.Doc
m.qdp.msfsx.cn/00224.Doc
m.qdp.msfsx.cn/24624.Doc
m.qdp.msfsx.cn/00200.Doc
m.qdp.msfsx.cn/60204.Doc
m.qdp.msfsx.cn/08482.Doc
m.qdp.msfsx.cn/82828.Doc
m.qdp.msfsx.cn/60244.Doc
m.qdp.msfsx.cn/95737.Doc
m.qdp.msfsx.cn/20084.Doc
m.qdp.msfsx.cn/22408.Doc
m.qdp.msfsx.cn/75555.Doc
m.qdp.msfsx.cn/75371.Doc
m.qdp.msfsx.cn/82446.Doc
m.qdp.msfsx.cn/02884.Doc
m.qdp.msfsx.cn/7.Doc
m.qdp.msfsx.cn/00806.Doc
m.qdp.msfsx.cn/19575.Doc
m.qdp.msfsx.cn/42802.Doc
m.qdo.msfsx.cn/04080.Doc
m.qdo.msfsx.cn/80424.Doc
m.qdo.msfsx.cn/00008.Doc
m.qdo.msfsx.cn/06280.Doc
m.qdo.msfsx.cn/39971.Doc
m.qdo.msfsx.cn/44444.Doc
m.qdo.msfsx.cn/84602.Doc
m.qdo.msfsx.cn/46668.Doc
m.qdo.msfsx.cn/06486.Doc
m.qdo.msfsx.cn/24260.Doc
m.qdo.msfsx.cn/40286.Doc
m.qdo.msfsx.cn/99137.Doc
m.qdo.msfsx.cn/00604.Doc
m.qdo.msfsx.cn/33111.Doc
m.qdo.msfsx.cn/86840.Doc
m.qdo.msfsx.cn/62262.Doc
m.qdo.msfsx.cn/46240.Doc
m.qdo.msfsx.cn/28888.Doc
m.qdo.msfsx.cn/88208.Doc
m.qdo.msfsx.cn/86888.Doc
m.qdi.msfsx.cn/04628.Doc
m.qdi.msfsx.cn/97517.Doc
m.qdi.msfsx.cn/20662.Doc
m.qdi.msfsx.cn/68640.Doc
m.qdi.msfsx.cn/39719.Doc
m.qdi.msfsx.cn/40620.Doc
m.qdi.msfsx.cn/84824.Doc
m.qdi.msfsx.cn/68680.Doc
m.qdi.msfsx.cn/66622.Doc
m.qdi.msfsx.cn/35519.Doc
m.qdi.msfsx.cn/28846.Doc
m.qdi.msfsx.cn/48828.Doc
m.qdi.msfsx.cn/02002.Doc
m.qdi.msfsx.cn/97591.Doc
m.qdi.msfsx.cn/39731.Doc
m.qdi.msfsx.cn/40402.Doc
m.qdi.msfsx.cn/80088.Doc
m.qdi.msfsx.cn/42266.Doc
m.qdi.msfsx.cn/02044.Doc
m.qdi.msfsx.cn/62060.Doc
m.qdu.msfsx.cn/64220.Doc
m.qdu.msfsx.cn/26846.Doc
m.qdu.msfsx.cn/60044.Doc
m.qdu.msfsx.cn/86606.Doc
m.qdu.msfsx.cn/20466.Doc
m.qdu.msfsx.cn/51793.Doc
m.qdu.msfsx.cn/08640.Doc
m.qdu.msfsx.cn/26488.Doc
m.qdu.msfsx.cn/26080.Doc
m.qdu.msfsx.cn/97393.Doc
m.qdu.msfsx.cn/15131.Doc
m.qdu.msfsx.cn/04048.Doc
m.qdu.msfsx.cn/66224.Doc
m.qdu.msfsx.cn/04202.Doc
m.qdu.msfsx.cn/59199.Doc
m.qdu.msfsx.cn/66602.Doc
m.qdu.msfsx.cn/37519.Doc
m.qdu.msfsx.cn/48060.Doc
m.qdu.msfsx.cn/66484.Doc
m.qdu.msfsx.cn/02468.Doc
m.qdy.msfsx.cn/06006.Doc
m.qdy.msfsx.cn/20662.Doc
m.qdy.msfsx.cn/26660.Doc
m.qdy.msfsx.cn/80806.Doc
m.qdy.msfsx.cn/93131.Doc
m.qdy.msfsx.cn/24640.Doc
m.qdy.msfsx.cn/19973.Doc
m.qdy.msfsx.cn/62802.Doc
m.qdy.msfsx.cn/40268.Doc
m.qdy.msfsx.cn/08020.Doc
m.qdy.msfsx.cn/82442.Doc
m.qdy.msfsx.cn/13173.Doc
m.qdy.msfsx.cn/51799.Doc
m.qdy.msfsx.cn/44644.Doc
m.qdy.msfsx.cn/60086.Doc
m.qdy.msfsx.cn/04284.Doc
m.qdy.msfsx.cn/02286.Doc
m.qdy.msfsx.cn/37733.Doc
m.qdy.msfsx.cn/88082.Doc
m.qdy.msfsx.cn/26246.Doc
m.qdt.msfsx.cn/64826.Doc
m.qdt.msfsx.cn/22608.Doc
m.qdt.msfsx.cn/82822.Doc
m.qdt.msfsx.cn/80642.Doc
m.qdt.msfsx.cn/91793.Doc
m.qdt.msfsx.cn/66624.Doc
m.qdt.msfsx.cn/20862.Doc
m.qdt.msfsx.cn/26446.Doc
m.qdt.msfsx.cn/42642.Doc
m.qdt.msfsx.cn/73733.Doc
m.qdt.msfsx.cn/68882.Doc
m.qdt.msfsx.cn/68442.Doc
m.qdt.msfsx.cn/46006.Doc
m.qdt.msfsx.cn/42222.Doc
m.qdt.msfsx.cn/44060.Doc
m.qdt.msfsx.cn/44082.Doc
m.qdt.msfsx.cn/48426.Doc
m.qdt.msfsx.cn/88626.Doc
m.qdt.msfsx.cn/40642.Doc
m.qdt.msfsx.cn/46442.Doc
m.qdr.msfsx.cn/08622.Doc
m.qdr.msfsx.cn/06028.Doc
m.qdr.msfsx.cn/77393.Doc
m.qdr.msfsx.cn/88204.Doc
m.qdr.msfsx.cn/82020.Doc
m.qdr.msfsx.cn/26482.Doc
m.qdr.msfsx.cn/86444.Doc
m.qdr.msfsx.cn/35519.Doc
m.qdr.msfsx.cn/44648.Doc
m.qdr.msfsx.cn/62226.Doc
m.qdr.msfsx.cn/88864.Doc
m.qdr.msfsx.cn/02088.Doc
m.qdr.msfsx.cn/44200.Doc
m.qdr.msfsx.cn/84660.Doc
m.qdr.msfsx.cn/86006.Doc
m.qdr.msfsx.cn/77995.Doc
m.qdr.msfsx.cn/08064.Doc
m.qdr.msfsx.cn/82262.Doc
m.qdr.msfsx.cn/64462.Doc
m.qdr.msfsx.cn/04882.Doc
m.qde.msfsx.cn/06460.Doc
m.qde.msfsx.cn/84088.Doc
m.qde.msfsx.cn/13991.Doc
m.qde.msfsx.cn/28040.Doc
m.qde.msfsx.cn/82860.Doc
m.qde.msfsx.cn/55317.Doc
m.qde.msfsx.cn/42888.Doc
m.qde.msfsx.cn/08460.Doc
m.qde.msfsx.cn/11555.Doc
m.qde.msfsx.cn/40626.Doc
m.qde.msfsx.cn/66486.Doc
m.qde.msfsx.cn/44442.Doc
m.qde.msfsx.cn/15131.Doc
m.qde.msfsx.cn/44482.Doc
m.qde.msfsx.cn/77371.Doc
m.qde.msfsx.cn/88066.Doc
m.qde.msfsx.cn/57191.Doc
m.qde.msfsx.cn/88240.Doc
