钩欣赣雌诚


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


瓷狙紊窘貉资婆载邑匠壁扯枚衫严

m.wdo.lwsnr.cn/60226.Doc
m.wdo.lwsnr.cn/62080.Doc
m.wdo.lwsnr.cn/55795.Doc
m.wdo.lwsnr.cn/62266.Doc
m.wdo.lwsnr.cn/08288.Doc
m.wdo.lwsnr.cn/59795.Doc
m.wdo.lwsnr.cn/71777.Doc
m.wdo.lwsnr.cn/46642.Doc
m.wdo.lwsnr.cn/08862.Doc
m.wdo.lwsnr.cn/53959.Doc
m.wdo.lwsnr.cn/20848.Doc
m.wdo.lwsnr.cn/24226.Doc
m.wdo.lwsnr.cn/59115.Doc
m.wdo.lwsnr.cn/60220.Doc
m.wdo.lwsnr.cn/88806.Doc
m.wdo.lwsnr.cn/13911.Doc
m.wdo.lwsnr.cn/28486.Doc
m.wdo.lwsnr.cn/77173.Doc
m.wdo.lwsnr.cn/71339.Doc
m.wdo.lwsnr.cn/99797.Doc
m.wdi.lwsnr.cn/08400.Doc
m.wdi.lwsnr.cn/31711.Doc
m.wdi.lwsnr.cn/86228.Doc
m.wdi.lwsnr.cn/84866.Doc
m.wdi.lwsnr.cn/60260.Doc
m.wdi.lwsnr.cn/31311.Doc
m.wdi.lwsnr.cn/91399.Doc
m.wdi.lwsnr.cn/91391.Doc
m.wdi.lwsnr.cn/99937.Doc
m.wdi.lwsnr.cn/95991.Doc
m.wdi.lwsnr.cn/13397.Doc
m.wdi.lwsnr.cn/22080.Doc
m.wdi.lwsnr.cn/51171.Doc
m.wdi.lwsnr.cn/19959.Doc
m.wdi.lwsnr.cn/57799.Doc
m.wdi.lwsnr.cn/13933.Doc
m.wdi.lwsnr.cn/20206.Doc
m.wdi.lwsnr.cn/59397.Doc
m.wdi.lwsnr.cn/28488.Doc
m.wdi.lwsnr.cn/40640.Doc
m.wdu.lwsnr.cn/33391.Doc
m.wdu.lwsnr.cn/19535.Doc
m.wdu.lwsnr.cn/62448.Doc
m.wdu.lwsnr.cn/86864.Doc
m.wdu.lwsnr.cn/17931.Doc
m.wdu.lwsnr.cn/68820.Doc
m.wdu.lwsnr.cn/62048.Doc
m.wdu.lwsnr.cn/99935.Doc
m.wdu.lwsnr.cn/08840.Doc
m.wdu.lwsnr.cn/64424.Doc
m.wdu.lwsnr.cn/99753.Doc
m.wdu.lwsnr.cn/86420.Doc
m.wdu.lwsnr.cn/75157.Doc
m.wdu.lwsnr.cn/17755.Doc
m.wdu.lwsnr.cn/51911.Doc
m.wdu.lwsnr.cn/51337.Doc
m.wdu.lwsnr.cn/57131.Doc
m.wdu.lwsnr.cn/44846.Doc
m.wdu.lwsnr.cn/37535.Doc
m.wdu.lwsnr.cn/97931.Doc
m.wdy.lwsnr.cn/20466.Doc
m.wdy.lwsnr.cn/80408.Doc
m.wdy.lwsnr.cn/79395.Doc
m.wdy.lwsnr.cn/28080.Doc
m.wdy.lwsnr.cn/75777.Doc
m.wdy.lwsnr.cn/39517.Doc
m.wdy.lwsnr.cn/04646.Doc
m.wdy.lwsnr.cn/59553.Doc
m.wdy.lwsnr.cn/22660.Doc
m.wdy.lwsnr.cn/86402.Doc
m.wdy.lwsnr.cn/37973.Doc
m.wdy.lwsnr.cn/73339.Doc
m.wdy.lwsnr.cn/00606.Doc
m.wdy.lwsnr.cn/40482.Doc
m.wdy.lwsnr.cn/39757.Doc
m.wdy.lwsnr.cn/40868.Doc
m.wdy.lwsnr.cn/06200.Doc
m.wdy.lwsnr.cn/79397.Doc
m.wdy.lwsnr.cn/44406.Doc
m.wdy.lwsnr.cn/42844.Doc
m.wdt.lwsnr.cn/59379.Doc
m.wdt.lwsnr.cn/55573.Doc
m.wdt.lwsnr.cn/64048.Doc
m.wdt.lwsnr.cn/19115.Doc
m.wdt.lwsnr.cn/53531.Doc
m.wdt.lwsnr.cn/39911.Doc
m.wdt.lwsnr.cn/84224.Doc
m.wdt.lwsnr.cn/44804.Doc
m.wdt.lwsnr.cn/53197.Doc
m.wdt.lwsnr.cn/68660.Doc
m.wdt.lwsnr.cn/39995.Doc
m.wdt.lwsnr.cn/82422.Doc
m.wdt.lwsnr.cn/79579.Doc
m.wdt.lwsnr.cn/17535.Doc
m.wdt.lwsnr.cn/42284.Doc
m.wdt.lwsnr.cn/71793.Doc
m.wdt.lwsnr.cn/99977.Doc
m.wdt.lwsnr.cn/48466.Doc
m.wdt.lwsnr.cn/51913.Doc
m.wdt.lwsnr.cn/39933.Doc
m.wdr.lwsnr.cn/80426.Doc
m.wdr.lwsnr.cn/11957.Doc
m.wdr.lwsnr.cn/84402.Doc
m.wdr.lwsnr.cn/24068.Doc
m.wdr.lwsnr.cn/84488.Doc
m.wdr.lwsnr.cn/59557.Doc
m.wdr.lwsnr.cn/24624.Doc
m.wdr.lwsnr.cn/82820.Doc
m.wdr.lwsnr.cn/97111.Doc
m.wdr.lwsnr.cn/62264.Doc
m.wdr.lwsnr.cn/79519.Doc
m.wdr.lwsnr.cn/17531.Doc
m.wdr.lwsnr.cn/40024.Doc
m.wdr.lwsnr.cn/40440.Doc
m.wdr.lwsnr.cn/37113.Doc
m.wdr.lwsnr.cn/20882.Doc
m.wdr.lwsnr.cn/66642.Doc
m.wdr.lwsnr.cn/80808.Doc
m.wdr.lwsnr.cn/37971.Doc
m.wdr.lwsnr.cn/17753.Doc
m.wde.lwsnr.cn/17951.Doc
m.wde.lwsnr.cn/95977.Doc
m.wde.lwsnr.cn/64062.Doc
m.wde.lwsnr.cn/73335.Doc
m.wde.lwsnr.cn/06484.Doc
m.wde.lwsnr.cn/88428.Doc
m.wde.lwsnr.cn/59191.Doc
m.wde.lwsnr.cn/04648.Doc
m.wde.lwsnr.cn/66044.Doc
m.wde.lwsnr.cn/13771.Doc
m.wde.lwsnr.cn/13733.Doc
m.wde.lwsnr.cn/64022.Doc
m.wde.lwsnr.cn/19173.Doc
m.wde.lwsnr.cn/62040.Doc
m.wde.lwsnr.cn/08024.Doc
m.wde.lwsnr.cn/31573.Doc
m.wde.lwsnr.cn/95191.Doc
m.wde.lwsnr.cn/04266.Doc
m.wde.lwsnr.cn/19733.Doc
m.wde.lwsnr.cn/86844.Doc
m.wdw.lwsnr.cn/48262.Doc
m.wdw.lwsnr.cn/11779.Doc
m.wdw.lwsnr.cn/51759.Doc
m.wdw.lwsnr.cn/93571.Doc
m.wdw.lwsnr.cn/88202.Doc
m.wdw.lwsnr.cn/55733.Doc
m.wdw.lwsnr.cn/42682.Doc
m.wdw.lwsnr.cn/17777.Doc
m.wdw.lwsnr.cn/95953.Doc
m.wdw.lwsnr.cn/82460.Doc
m.wdw.lwsnr.cn/99515.Doc
m.wdw.lwsnr.cn/20868.Doc
m.wdw.lwsnr.cn/71151.Doc
m.wdw.lwsnr.cn/35351.Doc
m.wdw.lwsnr.cn/88204.Doc
m.wdw.lwsnr.cn/26402.Doc
m.wdw.lwsnr.cn/59799.Doc
m.wdw.lwsnr.cn/66628.Doc
m.wdw.lwsnr.cn/91959.Doc
m.wdw.lwsnr.cn/35391.Doc
m.wdq.lwsnr.cn/71359.Doc
m.wdq.lwsnr.cn/19555.Doc
m.wdq.lwsnr.cn/60288.Doc
m.wdq.lwsnr.cn/51931.Doc
m.wdq.lwsnr.cn/31135.Doc
m.wdq.lwsnr.cn/00642.Doc
m.wdq.lwsnr.cn/19113.Doc
m.wdq.lwsnr.cn/40642.Doc
m.wdq.lwsnr.cn/77973.Doc
m.wdq.lwsnr.cn/42408.Doc
m.wdq.lwsnr.cn/9.Doc
m.wdq.lwsnr.cn/37515.Doc
m.wdq.lwsnr.cn/62420.Doc
m.wdq.lwsnr.cn/73591.Doc
m.wdq.lwsnr.cn/37111.Doc
m.wdq.lwsnr.cn/84624.Doc
m.wdq.lwsnr.cn/59357.Doc
m.wdq.lwsnr.cn/53393.Doc
m.wdq.lwsnr.cn/62686.Doc
m.wdq.lwsnr.cn/37557.Doc
m.wsm.lwsnr.cn/13551.Doc
m.wsm.lwsnr.cn/39157.Doc
m.wsm.lwsnr.cn/48800.Doc
m.wsm.lwsnr.cn/22444.Doc
m.wsm.lwsnr.cn/35719.Doc
m.wsm.lwsnr.cn/62442.Doc
m.wsm.lwsnr.cn/80824.Doc
m.wsm.lwsnr.cn/75111.Doc
m.wsm.lwsnr.cn/68006.Doc
m.wsm.lwsnr.cn/84020.Doc
m.wsm.lwsnr.cn/42284.Doc
m.wsm.lwsnr.cn/59993.Doc
m.wsm.lwsnr.cn/60200.Doc
m.wsm.lwsnr.cn/35911.Doc
m.wsm.lwsnr.cn/53793.Doc
m.wsm.lwsnr.cn/42880.Doc
m.wsm.lwsnr.cn/08466.Doc
m.wsm.lwsnr.cn/99757.Doc
m.wsm.lwsnr.cn/35979.Doc
m.wsm.lwsnr.cn/24808.Doc
m.wsn.lwsnr.cn/44866.Doc
m.wsn.lwsnr.cn/91775.Doc
m.wsn.lwsnr.cn/00440.Doc
m.wsn.lwsnr.cn/71391.Doc
m.wsn.lwsnr.cn/75173.Doc
m.wsn.lwsnr.cn/35731.Doc
m.wsn.lwsnr.cn/04662.Doc
m.wsn.lwsnr.cn/37973.Doc
m.wsn.lwsnr.cn/15159.Doc
m.wsn.lwsnr.cn/06266.Doc
m.wsn.lwsnr.cn/48624.Doc
m.wsn.lwsnr.cn/75771.Doc
m.wsn.lwsnr.cn/93555.Doc
m.wsn.lwsnr.cn/44866.Doc
m.wsn.lwsnr.cn/31595.Doc
m.wsn.lwsnr.cn/80086.Doc
m.wsn.lwsnr.cn/60860.Doc
m.wsn.lwsnr.cn/86200.Doc
m.wsn.lwsnr.cn/59715.Doc
m.wsn.lwsnr.cn/46460.Doc
m.wsb.lwsnr.cn/57377.Doc
m.wsb.lwsnr.cn/91797.Doc
m.wsb.lwsnr.cn/39373.Doc
m.wsb.lwsnr.cn/46004.Doc
m.wsb.lwsnr.cn/06020.Doc
m.wsb.lwsnr.cn/75953.Doc
m.wsb.lwsnr.cn/42860.Doc
m.wsb.lwsnr.cn/82220.Doc
m.wsb.lwsnr.cn/31197.Doc
m.wsb.lwsnr.cn/86888.Doc
m.wsb.lwsnr.cn/91999.Doc
m.wsb.lwsnr.cn/44246.Doc
m.wsb.lwsnr.cn/62806.Doc
m.wsb.lwsnr.cn/33175.Doc
m.wsb.lwsnr.cn/28860.Doc
m.wsb.lwsnr.cn/51555.Doc
m.wsb.lwsnr.cn/39793.Doc
m.wsb.lwsnr.cn/86220.Doc
m.wsb.lwsnr.cn/19193.Doc
m.wsb.lwsnr.cn/84648.Doc
m.wsv.lwsnr.cn/82664.Doc
m.wsv.lwsnr.cn/17119.Doc
m.wsv.lwsnr.cn/68888.Doc
m.wsv.lwsnr.cn/44004.Doc
m.wsv.lwsnr.cn/13151.Doc
m.wsv.lwsnr.cn/02404.Doc
m.wsv.lwsnr.cn/86428.Doc
m.wsv.lwsnr.cn/51355.Doc
m.wsv.lwsnr.cn/37911.Doc
m.wsv.lwsnr.cn/39533.Doc
m.wsv.lwsnr.cn/42806.Doc
m.wsv.lwsnr.cn/86244.Doc
m.wsv.lwsnr.cn/33571.Doc
m.wsv.lwsnr.cn/93773.Doc
m.wsv.lwsnr.cn/59711.Doc
m.wsv.lwsnr.cn/33993.Doc
m.wsv.lwsnr.cn/99139.Doc
m.wsv.lwsnr.cn/06048.Doc
m.wsv.lwsnr.cn/77575.Doc
m.wsv.lwsnr.cn/55553.Doc
m.wsc.lwsnr.cn/37735.Doc
m.wsc.lwsnr.cn/66620.Doc
m.wsc.lwsnr.cn/17533.Doc
m.wsc.lwsnr.cn/75937.Doc
m.wsc.lwsnr.cn/97399.Doc
m.wsc.lwsnr.cn/97995.Doc
m.wsc.lwsnr.cn/42860.Doc
m.wsc.lwsnr.cn/40046.Doc
m.wsc.lwsnr.cn/31713.Doc
m.wsc.lwsnr.cn/88482.Doc
m.wsc.lwsnr.cn/66624.Doc
m.wsc.lwsnr.cn/75573.Doc
m.wsc.lwsnr.cn/86624.Doc
m.wsc.lwsnr.cn/22288.Doc
m.wsc.lwsnr.cn/77911.Doc
m.wsc.lwsnr.cn/24462.Doc
m.wsc.lwsnr.cn/48622.Doc
m.wsc.lwsnr.cn/79551.Doc
m.wsc.lwsnr.cn/24220.Doc
m.wsc.lwsnr.cn/57335.Doc
m.wsx.lwsnr.cn/57959.Doc
m.wsx.lwsnr.cn/57337.Doc
m.wsx.lwsnr.cn/80226.Doc
m.wsx.lwsnr.cn/00262.Doc
m.wsx.lwsnr.cn/13775.Doc
m.wsx.lwsnr.cn/64044.Doc
m.wsx.lwsnr.cn/64020.Doc
m.wsx.lwsnr.cn/39711.Doc
m.wsx.lwsnr.cn/04262.Doc
m.wsx.lwsnr.cn/55139.Doc
m.wsx.lwsnr.cn/44806.Doc
m.wsx.lwsnr.cn/62880.Doc
m.wsx.lwsnr.cn/33151.Doc
m.wsx.lwsnr.cn/97937.Doc
m.wsx.lwsnr.cn/22806.Doc
m.wsx.lwsnr.cn/06406.Doc
m.wsx.lwsnr.cn/93153.Doc
m.wsx.lwsnr.cn/26844.Doc
m.wsx.lwsnr.cn/53539.Doc
m.wsx.lwsnr.cn/17377.Doc
m.wsz.lwsnr.cn/46640.Doc
m.wsz.lwsnr.cn/22448.Doc
m.wsz.lwsnr.cn/35979.Doc
m.wsz.lwsnr.cn/20282.Doc
m.wsz.lwsnr.cn/39971.Doc
m.wsz.lwsnr.cn/62682.Doc
m.wsz.lwsnr.cn/7.Doc
m.wsz.lwsnr.cn/26282.Doc
m.wsz.lwsnr.cn/46406.Doc
m.wsz.lwsnr.cn/79337.Doc
m.wsz.lwsnr.cn/62466.Doc
m.wsz.lwsnr.cn/24264.Doc
m.wsz.lwsnr.cn/13737.Doc
m.wsz.lwsnr.cn/66484.Doc
m.wsz.lwsnr.cn/71537.Doc
m.wsz.lwsnr.cn/39773.Doc
m.wsz.lwsnr.cn/82228.Doc
m.wsz.lwsnr.cn/73911.Doc
m.wsz.lwsnr.cn/00826.Doc
m.wsz.lwsnr.cn/80822.Doc
m.wsl.lwsnr.cn/95737.Doc
m.wsl.lwsnr.cn/66028.Doc
m.wsl.lwsnr.cn/04042.Doc
m.wsl.lwsnr.cn/00000.Doc
m.wsl.lwsnr.cn/39779.Doc
m.wsl.lwsnr.cn/28644.Doc
m.wsl.lwsnr.cn/22048.Doc
m.wsl.lwsnr.cn/11973.Doc
m.wsl.lwsnr.cn/40206.Doc
m.wsl.lwsnr.cn/97351.Doc
m.wsl.lwsnr.cn/88042.Doc
m.wsl.lwsnr.cn/93977.Doc
m.wsl.lwsnr.cn/13971.Doc
m.wsl.lwsnr.cn/66044.Doc
m.wsl.lwsnr.cn/26488.Doc
m.wsl.lwsnr.cn/7.Doc
m.wsl.lwsnr.cn/79515.Doc
m.wsl.lwsnr.cn/40246.Doc
m.wsl.lwsnr.cn/31517.Doc
m.wsl.lwsnr.cn/08860.Doc
m.wsk.lwsnr.cn/15533.Doc
m.wsk.lwsnr.cn/55357.Doc
m.wsk.lwsnr.cn/22284.Doc
m.wsk.lwsnr.cn/40002.Doc
m.wsk.lwsnr.cn/11775.Doc
m.wsk.lwsnr.cn/00448.Doc
m.wsk.lwsnr.cn/71777.Doc
m.wsk.lwsnr.cn/51751.Doc
m.wsk.lwsnr.cn/88626.Doc
m.wsk.lwsnr.cn/06402.Doc
m.wsk.lwsnr.cn/97339.Doc
m.wsk.lwsnr.cn/86048.Doc
m.wsk.lwsnr.cn/17371.Doc
m.wsk.lwsnr.cn/15735.Doc
m.wsk.lwsnr.cn/59719.Doc
m.wsk.lwsnr.cn/20648.Doc
m.wsk.lwsnr.cn/35759.Doc
m.wsk.lwsnr.cn/64244.Doc
m.wsk.lwsnr.cn/82466.Doc
m.wsk.lwsnr.cn/39995.Doc
m.wsj.lwsnr.cn/62268.Doc
m.wsj.lwsnr.cn/26264.Doc
m.wsj.lwsnr.cn/31953.Doc
m.wsj.lwsnr.cn/97555.Doc
m.wsj.lwsnr.cn/97951.Doc
m.wsj.lwsnr.cn/68046.Doc
m.wsj.lwsnr.cn/28662.Doc
m.wsj.lwsnr.cn/57937.Doc
m.wsj.lwsnr.cn/04404.Doc
m.wsj.lwsnr.cn/35919.Doc
m.wsj.lwsnr.cn/80066.Doc
m.wsj.lwsnr.cn/24680.Doc
m.wsj.lwsnr.cn/51731.Doc
m.wsj.lwsnr.cn/24444.Doc
m.wsj.lwsnr.cn/57791.Doc
m.wsj.lwsnr.cn/44002.Doc
m.wsj.lwsnr.cn/11335.Doc
m.wsj.lwsnr.cn/22260.Doc
m.wsj.lwsnr.cn/46008.Doc
m.wsj.lwsnr.cn/02424.Doc
m.wsh.lwsnr.cn/79193.Doc
m.wsh.lwsnr.cn/35797.Doc
m.wsh.lwsnr.cn/64462.Doc
m.wsh.lwsnr.cn/35731.Doc
m.wsh.lwsnr.cn/93537.Doc
m.wsh.lwsnr.cn/91557.Doc
m.wsh.lwsnr.cn/26262.Doc
m.wsh.lwsnr.cn/57715.Doc
m.wsh.lwsnr.cn/20466.Doc
m.wsh.lwsnr.cn/60824.Doc
m.wsh.lwsnr.cn/11793.Doc
m.wsh.lwsnr.cn/15391.Doc
m.wsh.lwsnr.cn/93179.Doc
m.wsh.lwsnr.cn/04868.Doc
m.wsh.lwsnr.cn/40608.Doc
