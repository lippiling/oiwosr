逃飞侥仆干


Linux request_threaded_irq线程化中断与handler_fn

request_threaded_irq是Linux内核中注册中断处理函数的主要接口，它将中断处理拆分为两个阶段：上半部（handler）在硬中断上下文执行，下半部（thread_fn）在专用内核线程中运行。这种做法显著降低了硬中断关闭时间，避免了高优先级中断长时间阻塞系统。

函数原型定义在kernel/irq/manage.c：

```c
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
             irq_handler_t thread_fn, unsigned long flags,
             const char *name, void *dev)
```

参数中，handler是硬中断回调函数，运行在中断关闭的原子上下文中；thread_fn是线程化回调，运行在可睡眠的进程上下文。当handler返回IRQ_WAKE_THREAD时，内核唤醒该中断对应的irq线程执行thread_fn。

函数内部的核心实现：

```c
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
             irq_handler_t thread_fn, unsigned long flags,
             const char *name, void *dev)
{
    struct irqaction *action;
    struct irq_desc *desc;
    int retval;

    if (irq >= NR_IRQS)
        return -EINVAL;
    if (!handler && !thread_fn)
        return -EINVAL;

    desc = irq_to_desc(irq);
    if (!desc)
        return -EINVAL;

    if (!handler) {
        if (!thread_fn)
            return -EINVAL;
        handler = irq_default_primary_handler;
    }

    action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
    if (!action)
        return -ENOMEM;

    action->handler = handler;
    action->thread_fn = thread_fn;
    action->flags = flags;
    action->name = name;
    action->dev_id = dev;

    retval = __setup_irq(irq, desc, action);
    if (retval)
        kfree(action);

    return retval;
}
```

当handler为NULL而thread_fn不为空时，内核自动填入irq_default_primary_handler。该函数是快速处理函数：

```c
static irqreturn_t irq_default_primary_handler(int irq, void *dev_id)
{
    return IRQ_WAKE_THREAD;
}
```

这意味着如果用户只提供thread_fn，中断到达后立即唤醒线程，不执行任何额外工作。这种做法适用于设备中断处理几乎不涉及时间敏感操作、所有逻辑可在进程上下文完成的场景。

__setup_irq是将irqaction插入中断描述符action链表的关键函数：

```c
static int __setup_irq(unsigned int irq, struct irq_desc *desc, struct irqaction *new)
{
    struct irqaction *old, **old_ptr;
    unsigned long flags;
    int ret, nested, shared = 0;

    raw_spin_lock_irqsave(&desc->lock, flags);
    old_ptr = &desc->action;
    old = *old_ptr;
    if (old) {
        if (!(old->flags & new->flags & IRQF_SHARED))
            return -EBUSY;
        shared = 1;
    }

    if (new->flags & IRQF_THREAD) {
        ret = setup_irq_thread(new, irq, desc);
        if (ret)
            return ret;
    }

    if (!shared) {
        ret = irq_startup(desc, IRQ_RESEND, IRQ_START_FORCE);
        if (ret) {
            desc->istate |= IRQS_NOAUTOEN;
            return ret;
        }
    }

    *old_ptr = new;
    new->next = old;
    raw_spin_unlock_irqrestore(&desc->lock, flags);

    if (new->flags & IRQF_SHARED)
        register_irq_proc(irq, desc);

    return 0;
}
```

irq线程在内核中是per-interrupt的，由setup_irq_thread创建：

```c
static int setup_irq_thread(struct irqaction *new, unsigned int irq,
                struct irq_desc *desc)
{
    struct task_struct *t;

    t = kthread_create(irq_thread, new, "irq/%d-%s", irq, new->name);
    if (IS_ERR(t))
        return PTR_ERR(t);

    new->thread = t;
    set_bit(IRQTF_RUNTHREAD, &new->thread_flags);
    return 0;
}
```

线程名为irq/%d-%s格式，其中%d是中断号，%s是设备名。该线程在中断触发且handler返回IRQ_WAKE_THREAD时被唤醒，执行new->thread_fn。线程主体irq_thread的循环逻辑：

```c
static int irq_thread(void *data)
{
    struct irqaction *action = data;
    struct irq_desc *desc = irq_to_desc(action->irq);
    irqreturn_t (*thread_fn)(int irq, void *dev_id) = action->thread_fn;
    int irq = action->irq;

    while (!irq_wait_for_interrupt(action)) {
        irqreturn_t action_ret;

        raw_spin_lock_irq(&desc->lock);
        if (desc->istate & IRQS_PENDING) {
            desc->istate &= ~IRQS_PENDING;
            raw_spin_unlock_irq(&desc->lock);
            handle_irq_event(desc);
        } else {
            raw_spin_unlock_irq(&desc->lock);
        }

        action_ret = thread_fn(irq, action->dev_id);
        if (action_ret == IRQ_HANDLED)
            local_inc(&desc->threads_handled);
    }
    return 0;
}
```

IRQF_ONESHOT标志对线程化中断有特殊意义。当设置了IRQF_ONESHOT时，内核在handler返回后保持中断屏蔽，直到thread_fn执行完毕才调用chip->irq_unmask。这防止了中断源在thread_fn执行期间再次触发导致丢失中断或数据竞争。对于需要电平触发的设备，IRQF_ONESHOT是标准配置：

```c
if (new->flags & IRQF_ONESHOT)
    desc->istate |= IRQS_ONESHOT;
```

request_threaded_irq与request_irq的关系：request_irq是request_threaded_irq的简化封装：

```c
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
        const char *name, void *dev)
{
    return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
```

当thread_fn为NULL时，内核不创建irq线程，所有工作都在handler中完成。这也意味着request_irq注册的是纯硬中断处理函数，不在进程上下文中运行。

中断线程化的优点在于降低了关中断时间。对于频繁触发的中断，handler仅记录必要硬件状态并返回IRQ_WAKE_THREAD，耗时的数据处理由低优先级的irq线程完成。同时irq线程可以被调度器管理，运行时可被其他更高优先级任务抢占，避免了硬中断上下文不能被调度的限制。

啃峙牧有燃现刂恼柏估济悠诙惫雍

tv.blog.itaadf.cn/Article/details/195711.sHtML
tv.blog.itaadf.cn/Article/details/022284.sHtML
tv.blog.itaadf.cn/Article/details/462864.sHtML
tv.blog.cvvliy.cn/Article/details/535791.sHtML
tv.blog.cvvliy.cn/Article/details/842060.sHtML
tv.blog.cvvliy.cn/Article/details/888440.sHtML
tv.blog.cvvliy.cn/Article/details/680604.sHtML
tv.blog.cvvliy.cn/Article/details/991735.sHtML
tv.blog.cvvliy.cn/Article/details/555153.sHtML
tv.blog.cvvliy.cn/Article/details/197159.sHtML
tv.blog.cvvliy.cn/Article/details/206244.sHtML
tv.blog.cvvliy.cn/Article/details/604084.sHtML
tv.blog.cvvliy.cn/Article/details/662268.sHtML
tv.blog.cvvliy.cn/Article/details/739975.sHtML
tv.blog.cvvliy.cn/Article/details/357755.sHtML
tv.blog.cvvliy.cn/Article/details/226062.sHtML
tv.blog.cvvliy.cn/Article/details/806266.sHtML
tv.blog.cvvliy.cn/Article/details/151751.sHtML
tv.blog.cvvliy.cn/Article/details/466844.sHtML
tv.blog.cvvliy.cn/Article/details/462680.sHtML
tv.blog.cvvliy.cn/Article/details/599397.sHtML
tv.blog.cvvliy.cn/Article/details/915575.sHtML
tv.blog.cvvliy.cn/Article/details/311771.sHtML
tv.blog.cvvliy.cn/Article/details/991593.sHtML
tv.blog.cvvliy.cn/Article/details/680860.sHtML
tv.blog.cvvliy.cn/Article/details/711797.sHtML
tv.blog.cvvliy.cn/Article/details/666420.sHtML
tv.blog.cvvliy.cn/Article/details/137753.sHtML
tv.blog.cvvliy.cn/Article/details/137597.sHtML
tv.blog.cvvliy.cn/Article/details/048882.sHtML
tv.blog.cvvliy.cn/Article/details/537711.sHtML
tv.blog.cvvliy.cn/Article/details/137975.sHtML
tv.blog.cvvliy.cn/Article/details/195579.sHtML
tv.blog.cvvliy.cn/Article/details/535573.sHtML
tv.blog.cvvliy.cn/Article/details/915131.sHtML
tv.blog.cvvliy.cn/Article/details/662060.sHtML
tv.blog.cvvliy.cn/Article/details/911315.sHtML
tv.blog.cvvliy.cn/Article/details/393151.sHtML
tv.blog.cvvliy.cn/Article/details/795391.sHtML
tv.blog.cvvliy.cn/Article/details/268802.sHtML
tv.blog.cvvliy.cn/Article/details/426428.sHtML
tv.blog.cvvliy.cn/Article/details/153171.sHtML
tv.blog.cvvliy.cn/Article/details/575553.sHtML
tv.blog.cvvliy.cn/Article/details/268086.sHtML
tv.blog.cvvliy.cn/Article/details/088460.sHtML
tv.blog.cvvliy.cn/Article/details/428082.sHtML
tv.blog.cvvliy.cn/Article/details/531111.sHtML
tv.blog.cvvliy.cn/Article/details/622248.sHtML
tv.blog.cvvliy.cn/Article/details/828204.sHtML
tv.blog.cvvliy.cn/Article/details/840060.sHtML
tv.blog.cvvliy.cn/Article/details/402442.sHtML
tv.blog.cvvliy.cn/Article/details/804026.sHtML
tv.blog.cvvliy.cn/Article/details/488268.sHtML
tv.blog.cvvliy.cn/Article/details/937393.sHtML
tv.blog.cvvliy.cn/Article/details/682080.sHtML
tv.blog.cvvliy.cn/Article/details/068608.sHtML
tv.blog.cvvliy.cn/Article/details/446224.sHtML
tv.blog.cvvliy.cn/Article/details/644064.sHtML
tv.blog.cvvliy.cn/Article/details/608866.sHtML
tv.blog.cvvliy.cn/Article/details/511759.sHtML
tv.blog.cvvliy.cn/Article/details/377331.sHtML
tv.blog.cvvliy.cn/Article/details/606208.sHtML
tv.blog.cvvliy.cn/Article/details/826606.sHtML
tv.blog.cvvliy.cn/Article/details/779959.sHtML
tv.blog.cvvliy.cn/Article/details/820844.sHtML
tv.blog.cvvliy.cn/Article/details/228066.sHtML
tv.blog.cvvliy.cn/Article/details/953333.sHtML
tv.blog.cvvliy.cn/Article/details/779533.sHtML
tv.blog.cvvliy.cn/Article/details/773939.sHtML
tv.blog.cvvliy.cn/Article/details/026664.sHtML
tv.blog.cvvliy.cn/Article/details/206246.sHtML
tv.blog.cvvliy.cn/Article/details/268062.sHtML
tv.blog.cvvliy.cn/Article/details/715919.sHtML
tv.blog.cvvliy.cn/Article/details/995133.sHtML
tv.blog.cvvliy.cn/Article/details/555357.sHtML
tv.blog.cvvliy.cn/Article/details/222848.sHtML
tv.blog.cvvliy.cn/Article/details/795991.sHtML
tv.blog.cvvliy.cn/Article/details/771919.sHtML
tv.blog.cvvliy.cn/Article/details/020048.sHtML
tv.blog.cvvliy.cn/Article/details/979351.sHtML
tv.blog.cvvliy.cn/Article/details/933371.sHtML
tv.blog.cvvliy.cn/Article/details/511395.sHtML
tv.blog.cvvliy.cn/Article/details/937553.sHtML
tv.blog.cvvliy.cn/Article/details/802028.sHtML
tv.blog.cvvliy.cn/Article/details/959577.sHtML
tv.blog.cvvliy.cn/Article/details/804222.sHtML
tv.blog.cvvliy.cn/Article/details/044084.sHtML
tv.blog.cvvliy.cn/Article/details/046646.sHtML
tv.blog.cvvliy.cn/Article/details/064408.sHtML
tv.blog.cvvliy.cn/Article/details/119319.sHtML
tv.blog.cvvliy.cn/Article/details/579913.sHtML
tv.blog.cvvliy.cn/Article/details/288624.sHtML
tv.blog.cvvliy.cn/Article/details/288246.sHtML
tv.blog.cvvliy.cn/Article/details/733373.sHtML
tv.blog.cvvliy.cn/Article/details/682266.sHtML
tv.blog.cvvliy.cn/Article/details/042400.sHtML
tv.blog.cvvliy.cn/Article/details/913577.sHtML
tv.blog.cvvliy.cn/Article/details/735395.sHtML
tv.blog.cvvliy.cn/Article/details/246080.sHtML
tv.blog.cvvliy.cn/Article/details/266220.sHtML
tv.blog.cvvliy.cn/Article/details/220448.sHtML
tv.blog.cvvliy.cn/Article/details/044626.sHtML
tv.blog.cvvliy.cn/Article/details/573195.sHtML
tv.blog.cvvliy.cn/Article/details/771195.sHtML
tv.blog.cvvliy.cn/Article/details/755173.sHtML
tv.blog.cvvliy.cn/Article/details/244448.sHtML
tv.blog.cvvliy.cn/Article/details/806846.sHtML
tv.blog.cvvliy.cn/Article/details/139137.sHtML
tv.blog.cvvliy.cn/Article/details/337175.sHtML
tv.blog.cvvliy.cn/Article/details/933115.sHtML
tv.blog.cvvliy.cn/Article/details/680842.sHtML
tv.blog.cvvliy.cn/Article/details/579155.sHtML
tv.blog.cvvliy.cn/Article/details/046004.sHtML
tv.blog.cvvliy.cn/Article/details/757937.sHtML
tv.blog.cvvliy.cn/Article/details/022886.sHtML
tv.blog.cvvliy.cn/Article/details/608246.sHtML
tv.blog.cvvliy.cn/Article/details/406400.sHtML
tv.blog.cvvliy.cn/Article/details/048202.sHtML
tv.blog.cvvliy.cn/Article/details/868224.sHtML
tv.blog.cvvliy.cn/Article/details/828660.sHtML
tv.blog.cvvliy.cn/Article/details/917537.sHtML
tv.blog.cvvliy.cn/Article/details/351917.sHtML
tv.blog.cvvliy.cn/Article/details/559553.sHtML
tv.blog.cvvliy.cn/Article/details/820288.sHtML
tv.blog.cvvliy.cn/Article/details/406628.sHtML
tv.blog.cvvliy.cn/Article/details/266648.sHtML
tv.blog.cvvliy.cn/Article/details/062200.sHtML
tv.blog.cvvliy.cn/Article/details/377599.sHtML
tv.blog.cvvliy.cn/Article/details/048666.sHtML
tv.blog.cvvliy.cn/Article/details/040062.sHtML
tv.blog.cvvliy.cn/Article/details/531519.sHtML
tv.blog.cvvliy.cn/Article/details/808626.sHtML
tv.blog.cvvliy.cn/Article/details/288860.sHtML
tv.blog.cvvliy.cn/Article/details/208686.sHtML
tv.blog.cvvliy.cn/Article/details/913915.sHtML
tv.blog.cvvliy.cn/Article/details/337399.sHtML
tv.blog.cvvliy.cn/Article/details/844482.sHtML
tv.blog.cvvliy.cn/Article/details/868684.sHtML
tv.blog.cvvliy.cn/Article/details/515995.sHtML
tv.blog.cvvliy.cn/Article/details/800008.sHtML
tv.blog.cvvliy.cn/Article/details/026208.sHtML
tv.blog.cvvliy.cn/Article/details/640064.sHtML
tv.blog.cvvliy.cn/Article/details/826604.sHtML
tv.blog.cvvliy.cn/Article/details/884800.sHtML
tv.blog.cvvliy.cn/Article/details/866286.sHtML
tv.blog.cvvliy.cn/Article/details/068404.sHtML
tv.blog.cvvliy.cn/Article/details/268262.sHtML
tv.blog.cvvliy.cn/Article/details/331953.sHtML
tv.blog.cvvliy.cn/Article/details/244060.sHtML
tv.blog.cvvliy.cn/Article/details/153157.sHtML
tv.blog.cvvliy.cn/Article/details/397351.sHtML
tv.blog.cvvliy.cn/Article/details/333535.sHtML
tv.blog.cvvliy.cn/Article/details/688622.sHtML
tv.blog.cvvliy.cn/Article/details/666602.sHtML
tv.blog.cvvliy.cn/Article/details/822464.sHtML
tv.blog.cvvliy.cn/Article/details/404024.sHtML
tv.blog.cvvliy.cn/Article/details/933773.sHtML
tv.blog.cvvliy.cn/Article/details/428664.sHtML
tv.blog.cvvliy.cn/Article/details/731913.sHtML
tv.blog.cvvliy.cn/Article/details/511357.sHtML
tv.blog.cvvliy.cn/Article/details/406080.sHtML
tv.blog.cvvliy.cn/Article/details/448086.sHtML
tv.blog.cvvliy.cn/Article/details/862242.sHtML
tv.blog.cvvliy.cn/Article/details/375573.sHtML
tv.blog.cvvliy.cn/Article/details/084426.sHtML
tv.blog.cvvliy.cn/Article/details/004002.sHtML
tv.blog.cvvliy.cn/Article/details/315995.sHtML
tv.blog.cvvliy.cn/Article/details/997111.sHtML
tv.blog.cvvliy.cn/Article/details/517795.sHtML
tv.blog.cvvliy.cn/Article/details/733751.sHtML
tv.blog.cvvliy.cn/Article/details/602822.sHtML
tv.blog.cvvliy.cn/Article/details/957719.sHtML
tv.blog.cvvliy.cn/Article/details/579753.sHtML
tv.blog.cvvliy.cn/Article/details/719513.sHtML
tv.blog.cvvliy.cn/Article/details/804408.sHtML
tv.blog.cvvliy.cn/Article/details/804082.sHtML
tv.blog.cvvliy.cn/Article/details/480428.sHtML
tv.blog.cvvliy.cn/Article/details/400028.sHtML
tv.blog.cvvliy.cn/Article/details/608664.sHtML
tv.blog.cvvliy.cn/Article/details/244242.sHtML
tv.blog.cvvliy.cn/Article/details/244602.sHtML
tv.blog.cvvliy.cn/Article/details/866422.sHtML
tv.blog.cvvliy.cn/Article/details/044286.sHtML
tv.blog.cvvliy.cn/Article/details/919197.sHtML
tv.blog.cvvliy.cn/Article/details/442260.sHtML
tv.blog.cvvliy.cn/Article/details/662884.sHtML
tv.blog.cvvliy.cn/Article/details/155997.sHtML
tv.blog.cvvliy.cn/Article/details/917597.sHtML
tv.blog.cvvliy.cn/Article/details/915173.sHtML
tv.blog.cvvliy.cn/Article/details/979953.sHtML
tv.blog.cvvliy.cn/Article/details/993599.sHtML
tv.blog.cvvliy.cn/Article/details/579919.sHtML
tv.blog.cvvliy.cn/Article/details/484640.sHtML
tv.blog.cvvliy.cn/Article/details/446408.sHtML
tv.blog.cvvliy.cn/Article/details/591193.sHtML
tv.blog.cvvliy.cn/Article/details/311335.sHtML
tv.blog.cvvliy.cn/Article/details/806222.sHtML
tv.blog.cvvliy.cn/Article/details/844828.sHtML
tv.blog.cvvliy.cn/Article/details/377595.sHtML
tv.blog.cvvliy.cn/Article/details/488048.sHtML
tv.blog.cvvliy.cn/Article/details/002660.sHtML
tv.blog.cvvliy.cn/Article/details/973515.sHtML
tv.blog.cvvliy.cn/Article/details/264248.sHtML
tv.blog.cvvliy.cn/Article/details/204468.sHtML
tv.blog.cvvliy.cn/Article/details/959315.sHtML
tv.blog.cvvliy.cn/Article/details/464006.sHtML
tv.blog.cvvliy.cn/Article/details/791195.sHtML
tv.blog.cvvliy.cn/Article/details/391737.sHtML
tv.blog.cvvliy.cn/Article/details/462008.sHtML
tv.blog.cvvliy.cn/Article/details/735133.sHtML
tv.blog.cvvliy.cn/Article/details/428426.sHtML
tv.blog.cvvliy.cn/Article/details/860806.sHtML
tv.blog.cvvliy.cn/Article/details/264680.sHtML
tv.blog.cvvliy.cn/Article/details/428444.sHtML
tv.blog.cvvliy.cn/Article/details/824402.sHtML
tv.blog.cvvliy.cn/Article/details/595333.sHtML
tv.blog.cvvliy.cn/Article/details/971337.sHtML
tv.blog.cvvliy.cn/Article/details/597599.sHtML
tv.blog.cvvliy.cn/Article/details/131573.sHtML
tv.blog.cvvliy.cn/Article/details/246620.sHtML
tv.blog.cvvliy.cn/Article/details/280222.sHtML
tv.blog.cvvliy.cn/Article/details/397113.sHtML
tv.blog.cvvliy.cn/Article/details/024222.sHtML
tv.blog.cvvliy.cn/Article/details/020026.sHtML
tv.blog.cvvliy.cn/Article/details/711777.sHtML
tv.blog.cvvliy.cn/Article/details/999933.sHtML
tv.blog.cvvliy.cn/Article/details/020820.sHtML
tv.blog.cvvliy.cn/Article/details/242020.sHtML
tv.blog.cvvliy.cn/Article/details/597173.sHtML
tv.blog.cvvliy.cn/Article/details/026066.sHtML
tv.blog.cvvliy.cn/Article/details/280260.sHtML
tv.blog.cvvliy.cn/Article/details/282242.sHtML
tv.blog.cvvliy.cn/Article/details/311759.sHtML
tv.blog.cvvliy.cn/Article/details/719591.sHtML
tv.blog.cvvliy.cn/Article/details/993159.sHtML
tv.blog.cvvliy.cn/Article/details/602260.sHtML
tv.blog.cvvliy.cn/Article/details/088680.sHtML
tv.blog.cvvliy.cn/Article/details/266280.sHtML
tv.blog.cvvliy.cn/Article/details/666040.sHtML
tv.blog.cvvliy.cn/Article/details/062624.sHtML
tv.blog.cvvliy.cn/Article/details/339751.sHtML
tv.blog.cvvliy.cn/Article/details/131337.sHtML
tv.blog.cvvliy.cn/Article/details/979537.sHtML
tv.blog.cvvliy.cn/Article/details/977953.sHtML
tv.blog.cvvliy.cn/Article/details/864466.sHtML
tv.blog.cvvliy.cn/Article/details/717979.sHtML
tv.blog.cvvliy.cn/Article/details/535539.sHtML
tv.blog.cvvliy.cn/Article/details/806020.sHtML
tv.blog.cvvliy.cn/Article/details/062628.sHtML
tv.blog.cvvliy.cn/Article/details/993575.sHtML
tv.blog.cvvliy.cn/Article/details/606024.sHtML
tv.blog.cvvliy.cn/Article/details/800404.sHtML
tv.blog.cvvliy.cn/Article/details/422808.sHtML
tv.blog.cvvliy.cn/Article/details/995577.sHtML
tv.blog.cvvliy.cn/Article/details/357371.sHtML
tv.blog.cvvliy.cn/Article/details/488440.sHtML
tv.blog.cvvliy.cn/Article/details/759591.sHtML
tv.blog.cvvliy.cn/Article/details/228286.sHtML
tv.blog.cvvliy.cn/Article/details/826202.sHtML
tv.blog.cvvliy.cn/Article/details/246644.sHtML
tv.blog.cvvliy.cn/Article/details/226664.sHtML
tv.blog.cvvliy.cn/Article/details/484000.sHtML
tv.blog.cvvliy.cn/Article/details/068802.sHtML
tv.blog.cvvliy.cn/Article/details/395711.sHtML
tv.blog.cvvliy.cn/Article/details/731153.sHtML
tv.blog.cvvliy.cn/Article/details/468648.sHtML
tv.blog.cvvliy.cn/Article/details/173373.sHtML
tv.blog.cvvliy.cn/Article/details/024606.sHtML
tv.blog.cvvliy.cn/Article/details/155331.sHtML
tv.blog.cvvliy.cn/Article/details/480208.sHtML
tv.blog.cvvliy.cn/Article/details/317951.sHtML
tv.blog.cvvliy.cn/Article/details/517935.sHtML
tv.blog.cvvliy.cn/Article/details/117117.sHtML
tv.blog.cvvliy.cn/Article/details/828240.sHtML
tv.blog.cvvliy.cn/Article/details/953179.sHtML
tv.blog.cvvliy.cn/Article/details/022222.sHtML
tv.blog.cvvliy.cn/Article/details/222228.sHtML
tv.blog.cvvliy.cn/Article/details/531517.sHtML
tv.blog.cvvliy.cn/Article/details/519797.sHtML
tv.blog.cvvliy.cn/Article/details/537315.sHtML
tv.blog.cvvliy.cn/Article/details/979313.sHtML
tv.blog.cvvliy.cn/Article/details/866640.sHtML
tv.blog.cvvliy.cn/Article/details/311195.sHtML
tv.blog.cvvliy.cn/Article/details/599999.sHtML
tv.blog.cvvliy.cn/Article/details/175191.sHtML
tv.blog.cvvliy.cn/Article/details/715991.sHtML
tv.blog.cvvliy.cn/Article/details/280640.sHtML
tv.blog.cvvliy.cn/Article/details/824488.sHtML
tv.blog.cvvliy.cn/Article/details/686246.sHtML
tv.blog.cvvliy.cn/Article/details/199977.sHtML
tv.blog.cvvliy.cn/Article/details/391533.sHtML
tv.blog.cvvliy.cn/Article/details/919957.sHtML
tv.blog.cvvliy.cn/Article/details/171171.sHtML
tv.blog.cvvliy.cn/Article/details/197111.sHtML
tv.blog.cvvliy.cn/Article/details/953551.sHtML
tv.blog.cvvliy.cn/Article/details/640282.sHtML
tv.blog.cvvliy.cn/Article/details/480040.sHtML
tv.blog.cvvliy.cn/Article/details/622266.sHtML
tv.blog.cvvliy.cn/Article/details/715171.sHtML
tv.blog.cvvliy.cn/Article/details/557397.sHtML
tv.blog.cvvliy.cn/Article/details/353339.sHtML
tv.blog.cvvliy.cn/Article/details/842828.sHtML
tv.blog.cvvliy.cn/Article/details/024042.sHtML
tv.blog.cvvliy.cn/Article/details/202868.sHtML
tv.blog.cvvliy.cn/Article/details/644608.sHtML
tv.blog.cvvliy.cn/Article/details/555335.sHtML
tv.blog.cvvliy.cn/Article/details/664000.sHtML
tv.blog.cvvliy.cn/Article/details/113977.sHtML
tv.blog.cvvliy.cn/Article/details/159377.sHtML
tv.blog.cvvliy.cn/Article/details/044804.sHtML
tv.blog.cvvliy.cn/Article/details/822206.sHtML
tv.blog.cvvliy.cn/Article/details/240604.sHtML
tv.blog.cvvliy.cn/Article/details/559935.sHtML
tv.blog.cvvliy.cn/Article/details/604224.sHtML
tv.blog.cvvliy.cn/Article/details/991359.sHtML
tv.blog.cvvliy.cn/Article/details/911337.sHtML
tv.blog.cvvliy.cn/Article/details/513731.sHtML
tv.blog.cvvliy.cn/Article/details/208826.sHtML
tv.blog.cvvliy.cn/Article/details/422444.sHtML
tv.blog.cvvliy.cn/Article/details/355939.sHtML
tv.blog.cvvliy.cn/Article/details/004606.sHtML
tv.blog.cvvliy.cn/Article/details/684886.sHtML
tv.blog.cvvliy.cn/Article/details/086244.sHtML
tv.blog.cvvliy.cn/Article/details/913977.sHtML
tv.blog.cvvliy.cn/Article/details/244266.sHtML
tv.blog.cvvliy.cn/Article/details/280260.sHtML
tv.blog.cvvliy.cn/Article/details/313595.sHtML
tv.blog.cvvliy.cn/Article/details/824864.sHtML
tv.blog.cvvliy.cn/Article/details/319577.sHtML
tv.blog.cvvliy.cn/Article/details/022666.sHtML
tv.blog.cvvliy.cn/Article/details/246828.sHtML
tv.blog.cvvliy.cn/Article/details/395595.sHtML
tv.blog.cvvliy.cn/Article/details/046246.sHtML
tv.blog.cvvliy.cn/Article/details/684604.sHtML
tv.blog.cvvliy.cn/Article/details/711773.sHtML
tv.blog.cvvliy.cn/Article/details/664448.sHtML
tv.blog.cvvliy.cn/Article/details/442204.sHtML
tv.blog.cvvliy.cn/Article/details/737179.sHtML
tv.blog.cvvliy.cn/Article/details/460622.sHtML
tv.blog.cvvliy.cn/Article/details/199313.sHtML
tv.blog.cvvliy.cn/Article/details/511355.sHtML
tv.blog.cvvliy.cn/Article/details/804026.sHtML
tv.blog.cvvliy.cn/Article/details/822462.sHtML
tv.blog.cvvliy.cn/Article/details/486264.sHtML
tv.blog.cvvliy.cn/Article/details/402280.sHtML
tv.blog.cvvliy.cn/Article/details/997959.sHtML
tv.blog.cvvliy.cn/Article/details/068600.sHtML
tv.blog.cvvliy.cn/Article/details/622626.sHtML
tv.blog.cvvliy.cn/Article/details/937711.sHtML
tv.blog.cvvliy.cn/Article/details/175575.sHtML
tv.blog.cvvliy.cn/Article/details/422464.sHtML
tv.blog.cvvliy.cn/Article/details/806844.sHtML
tv.blog.cvvliy.cn/Article/details/531359.sHtML
tv.blog.cvvliy.cn/Article/details/971135.sHtML
tv.blog.cvvliy.cn/Article/details/933573.sHtML
tv.blog.cvvliy.cn/Article/details/620200.sHtML
tv.blog.cvvliy.cn/Article/details/731399.sHtML
tv.blog.cvvliy.cn/Article/details/044688.sHtML
tv.blog.cvvliy.cn/Article/details/862828.sHtML
tv.blog.cvvliy.cn/Article/details/488440.sHtML
tv.blog.cvvliy.cn/Article/details/426600.sHtML
tv.blog.cvvliy.cn/Article/details/288282.sHtML
tv.blog.cvvliy.cn/Article/details/642684.sHtML
tv.blog.cvvliy.cn/Article/details/357995.sHtML
tv.blog.cvvliy.cn/Article/details/713733.sHtML
tv.blog.cvvliy.cn/Article/details/086464.sHtML
tv.blog.cvvliy.cn/Article/details/977911.sHtML
tv.blog.cvvliy.cn/Article/details/022026.sHtML
tv.blog.cvvliy.cn/Article/details/822268.sHtML
tv.blog.cvvliy.cn/Article/details/711193.sHtML
tv.blog.cvvliy.cn/Article/details/535917.sHtML
tv.blog.cvvliy.cn/Article/details/779717.sHtML
tv.blog.cvvliy.cn/Article/details/882408.sHtML
tv.blog.cvvliy.cn/Article/details/999393.sHtML
tv.blog.cvvliy.cn/Article/details/686646.sHtML
tv.blog.cvvliy.cn/Article/details/628842.sHtML
tv.blog.cvvliy.cn/Article/details/111579.sHtML
tv.blog.cvvliy.cn/Article/details/359371.sHtML
tv.blog.cvvliy.cn/Article/details/868268.sHtML
tv.blog.cvvliy.cn/Article/details/044282.sHtML
tv.blog.cvvliy.cn/Article/details/086066.sHtML
tv.blog.cvvliy.cn/Article/details/931377.sHtML
tv.blog.cvvliy.cn/Article/details/828604.sHtML
tv.blog.cvvliy.cn/Article/details/111113.sHtML
tv.blog.cvvliy.cn/Article/details/997331.sHtML
tv.blog.cvvliy.cn/Article/details/846240.sHtML
tv.blog.cvvliy.cn/Article/details/759131.sHtML
tv.blog.cvvliy.cn/Article/details/084662.sHtML
tv.blog.cvvliy.cn/Article/details/222622.sHtML
tv.blog.cvvliy.cn/Article/details/151393.sHtML
tv.blog.cvvliy.cn/Article/details/400628.sHtML
tv.blog.cvvliy.cn/Article/details/640200.sHtML
tv.blog.cvvliy.cn/Article/details/280822.sHtML
tv.blog.cvvliy.cn/Article/details/880482.sHtML
tv.blog.cvvliy.cn/Article/details/484084.sHtML
