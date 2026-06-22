厥悔共仔偷


vhost-net virtio环后端的通知与轮询机制

vhost-net的后端通知模型基于两个核心组件：vhost_work工作队列与vhost_poll轮询引擎。前者负责异步任务调度，后者驱动virtio vring的可用描述符消费。两者在drivers/vhost/vhost.c中实现，vhost_net在drivers/vhost/net.c中组装使用。

vhost_poll的初始化入口是vhost_poll_init，它挂载一个wait_queue_entry到vhost_work的完成回调链上：

```c
void vhost_poll_init(struct vhost_poll *poll, vhost_work_fn_t fn,
                     unsigned long mask, struct vhost_dev *dev)
{
        init_waitqueue_func_entry(&poll->wake_entry, vhost_poll_wake);
        poll->fn = fn;
        poll->mask = mask;
        poll->dev = dev;
        vhost_work_init(&poll->work, fn);
}
```

vhost_poll_wake是唤醒回调的核心。当一个外部事件（例如tap设备的文件操作投递数据）触发时，该函数通过调用vhost_poll_queue将工作项插入vhost_dev的工作链表：

```c
void vhost_poll_queue(struct vhost_poll *poll)
{
        vhost_work_queue(poll->dev, &poll->work);
}
```

注意这里不持有任何锁。vhost_work_queue内部通过spin_lock_irqsave(&dev->work_lock, flags)保护对dev->work_list的并发操作。多个生产者（例如多个tap fd同时唤醒）对work_list的竞争在work_lock下串行化，但每个生产者只会把一个work重复插入——关键在于vhost_work_queue中的重复检测：

```c
void vhost_work_queue(struct vhost_dev *dev, struct vhost_work *work)
{
        unsigned long flags;
        spin_lock_irqsave(&dev->work_lock, flags);
        if (list_empty(&work->node)) {
                list_add_tail(&work->node, &dev->work_list);
                work->done_seq = 0;
        }
        spin_unlock_irqrestore(&dev->work_lock, flags);
}
```

只在work->node链表为空时才入队。这意味着如果同一个poll已经在队列中，多次vhost_poll_wake只产生一次调度，避免了工作项的洪水。但这也引入了一个竞态：vhost_work执行完毕后重置node为空的时间窗口内，如果正好有新的wake到达，该wake可能被丢失。vhost_work_completion中通过work->done_seq的原子递增来处理这个确认问题。

vhost_worker是实际的消费线程。它在vhost_dev_set_owner中被创建为内核线程：

```c
static int vhost_worker(void *data)
{
        struct vhost_dev *dev = data;
        struct vhost_work *work = NULL;

        for (;;) {
                set_current_state(TASK_INTERRUPTIBLE);
                spin_lock_irq(&dev->work_lock);
                if (!list_empty(&dev->work_list))
                        work = list_first_entry(&dev->work_list,
                                                struct vhost_work, node);
                spin_unlock_irq(&dev->work_lock);

                if (work) {
                        __set_current_state(TASK_RUNNING);
                        work->fn(work);
                        spin_lock_irq(&dev->work_lock);
                        list_del_init(&work->node);
                        if (!dev->work_list) {
                                // 不再有pending work，可以进入阻塞
                        }
                        spin_unlock_irq(&dev->work_lock);
                        work = NULL;
                } else {
                        schedule();
                }

                if (kthread_should_stop())
                        break;
        }
        __set_current_state(TASK_RUNNING);
        return 0;
}
```

当work_list为空时，vhost_worker调用schedule()进入睡眠。外部wake通过vhost_poll_wake -> vhost_poll_queue -> list_add_tail + wake_up_process将worker唤醒。这是典型的"等待-工作-等待"模型，零中断路径上的上下文切换。

现在进入virtio通知抑制的核心——event_idx机制。当guest使能VIRTIO_F_EVENT_IDX特性后，vhost后端不再依赖传统的VRING_AVAIL_F_NO_INTERRUPT标记，而是通过avail_event和used_event的索引比较来决定是否发送通知。

vhost_get_vring_buf用于从用户空间映射的vring内存中读取元数据。vhost_enable_notify和vhost_disable_notify控制通知的开关：

```c
/* drivers/vhost/vhost.c */
bool vhost_enable_notify(struct vhost_dev *dev, struct vhost_virtqueue *vq)
{
        struct vring_used *used = vq->used;
        u16 flags;

        if (!(vq->acked_features & (1ULL << VIRTIO_F_EVENT_IDX))) {
                flags = vhost_used_flags(dev, vq);
                vhost_used_flags(dev, vq) = flags & ~VRING_AVAIL_F_NO_INTERRUPT;
                return flags & VRING_AVAIL_F_NO_INTERRUPT;
        }

        vhost_avail_event(vq) = vq->last_avail_idx;
        mb();
        return vhost_need_event(vhost_get_used_event(vq),
                                vq->avail_idx, vq->last_avail_idx);
}
```

非event_idx路径下，仅仅清除used ring的NO_INTERRUPT标记使能中断。event_idx路径则更新avail_event为当前已处理的位置，然后通过vhost_need_event判断guest是否需要实际发送中断：

```c
static bool vhost_need_event(u16 event, u16 new, u16 old)
{
        return (u16)(new - event - 1) < (u16)(new - old);
}
```

这个无符号环绕比较非常微妙。当new >= event时，new - event - 1回绕后是一个大于等于(u16)-1的大数，必然不小于new - old，所以不触发事件——这正是event_idx的精髓：只有当guest将used ring的索引推进到event指定的位置时，才产生通知。相反，如果new - event - 1因为event > new而回绕到小于new - old的值，才触发通知，这对应guest已处理位置超过当前可用描述符的场景（意味着绕过了一个完整的回环，大概率是guest处理过慢）。

vhost_disable_notify的行为与之对称：

```c
void vhost_disable_notify(struct vhost_dev *dev, struct vhost_virtqueue *vq)
{
        if (!(vq->acked_features & (1ULL << VIRTIO_F_EVENT_IDX))) {
                vhost_used_flags(dev, vq) |= VRING_AVAIL_F_NO_INTERRUPT;
                return;
        }

        vhost_avail_event(vq) = vq->avail_idx;
        mb();
}
```

在非event_idx路径，直接设置NO_INTERRUPT位抑制通知。event_idx路径只需要更新avail_event到最新位置就足够了——guest端在vring_interrupt中比较avail_event和last_avail_idx来决定是否注入irq。

现在分析vhost_net的tx/rx轮询路径。handle_tx的核心逻辑在drivers/vhost/net.c中：

```c
static void handle_tx(struct vhost_net *net)
{
        struct vhost_net_virtqueue *nvq = &net->vqs[VHOST_NET_VQ_TX];
        struct vhost_virtqueue *vq = &nvq->vq;
        struct socket *sock = rcu_dereference_check(vq->private_data, 1);
        struct msghdr msg = {
                .msg_name = NULL,
                .msg_namelen = 0,
                .msg_control = NULL,
                .msg_controllen = 0,
                .msg_flags = MSG_DONTWAIT,
        };

        if (!sock)
                return;

        mutex_lock(&vq->mutex);
        vhost_disable_notify(&net->dev, vq);

        for (;;) {
                struct iov_iter iter;
                unsigned out, in;
                size_t headcount;
                struct vhost_net_desc *desc;
                struct vhost_net_ubuf_ref *uninitialized_var(ubufs);
                bool zcopy_used;
                int err;

                headcount = vhost_get_vq_desc(&net->dev, vq, vq->iov,
                                              ARRAY_SIZE(vq->iov),
                                              &out, &in,
                                              NULL, NULL);
                if (headcount < 0)
                        break;            // 内存错误，退出
                if (headcount == out)      // 无可用描述符
                        break;

                // ... 处理描述符链，发送数据 ...
        }

        vhost_zerocopy_signal_used(net, vq);
        mutex_unlock(&vq->mutex);
}
```

边界条件1：headcount == out的判断非常关键。vhost_get_vq_desc返回的是total out（可读）+ in（可写）描述符数。如果out == headcount，说明全部是可读描述符，没有可写描述符来存放tx的元数据——这在半双工或者guest tx路径下不可能发生，因此作为break条件。但如果guest因为bug或恶意构造了全out的描述符链，这个break避免了deadloop。

边界条件2：vhost_get_vq_desc返回0（无可用描述符）时，vq->busyloop_timeout控制是退化为poll还是进入睡眠：

```c
static bool vhost_has_work(struct vhost_dev *dev)
{
        return !list_empty(&dev->work_list);
}

static int vhost_vq_tx_busy_loop(struct vhost_virtqueue *vq,
                                  struct socket *sock,
                                  bool (*sock_can_read)(struct socket *))
{
        unsigned long endtime = busy_clock() + vq->busyloop_timeout;

        while (busy_clock() < endtime) {
                if (vhost_has_work(vq->dev))
                        return 0;
                if (sock_can_read(sock))
                        return 1;
                cpu_relax();
        }
        return 0;
}
```

busy_poll由模块参数vhost_busy_poll控制，单位微秒。设置为0时关闭busy polling，直接vhost_enable_notify + wait。大于0时，worker线程在vhost_get_vq_desc返回0后不立即睡眠，而是自旋等待最多busyloop_timeout微秒，期间轮询sock_can_read。这对低延迟场景至关重要——避免了进入schedule再被tap fd的wake_up重新唤醒的数百微秒延迟。但代价是CPU核被完全占据，吞吐量下降。

竞态场景：vhost_poll_start与vhost_poll_stop在设备生命周期中的同步。vhost_poll_start在tap fd上注册wait queue：

```c
int vhost_poll_start(struct vhost_poll *poll, struct file *file)
{
        __poll_t mask;

        if (poll->wqh)
                return 0;

        mask = vfs_poll(file, &poll->table);
        if (mask)
                vhost_poll_wake(poll);
        return 0;
}
```

竞争点：如果在vhost_poll_start执行到vfs_poll之前，另一个线程调用了vhost_poll_stop：

```c
void vhost_poll_stop(struct vhost_poll *poll)
{
        remove_wait_queue(poll->wqh, &poll->wake_entry);
        poll->wqh = NULL;
}
```

如果vhost_poll_stop先执行了remove_wait_queue并置NULL，然后vhost_poll_start读取到poll->wqh为NULL进入vfs_poll，这个poll就会注册到错误的wait_queue_head上，或者vfs_poll返回的非零mask触发的wake永远不会被处理。vhost core通过dev->mutex串行化设备控制路径来避免这类问题——所有SET_BACKEND、SET_OWNER等ioctl都在持有的mutex下操作。

再来看handle_rx的批量字节流处理。handle_rx负责从tap/sock接收数据并写入guest的可用描述符。它通过mergeable buffer特性支持分割接收缓冲区：

```c
static void handle_rx(struct vhost_net *net)
{
        struct sock *sk = net->sock->sk;
        struct iov_iter *msg_iov = &nvq->msg_iter;

        mutex_lock(&vq->mutex);
        vhost_disable_notify(&net->dev, vq);
        vhost_net_busy_poll(net, vq, true);
        // ... 批量接收循环 ...

        for (;;) {
                headcount = get_rx_bufs(vq, vq->heads + nheads, vhost_hlen,
                                        &in, vq->num * sizeof(struct virtio_net_hdr));
                if (headcount < 0)
                        goto out;

                if (headcount > 0) {
                        // 计算实际数据量
                        seg = vhost_net_build_pskb(net, sock, sk,
                                                    vq->iov, vq->iov_size,
                                                    &len);
                }

                if (seg < 0) {
                        vhost_discard_vq_desc(vq, headcount);
                        break;
                }

                // 批量更新used ring
                vhost_add_used_n(vq, vq->heads, headcount);
        }

        vhost_net_signal_used(net, vq);
        mutex_unlock(&vq->mutex);
}
```

get_rx_bufs从vring的avail ring中批量获取接收缓冲区，最多到num个。它返回时填充vq->heads数组，每个head对应一个描述符链的起始index。vhost_add_used_n利用批量更新减少对used ring的写操作次数——这对性能影响很大，因为used ring的写入通常涉及跨NUMA节点的cache coherence traffic（qemu作为vhost-user进程运行在另一个核上）。

vhost_add_used_n的实现：

```c
int vhost_add_used_n(struct vhost_virtqueue *vq, struct vring_used_elem *heads,
                     unsigned count)
{
        int start, n, r;

        start = vq->last_used_idx & (vq->num - 1);
        if (start + count > vq->num) {
                // 跨回环边界，需要分两次更新
                n = vq->num - start;
                vhost_add_used_indirect(vq, start, heads, n);
                vhost_add_used_indirect(vq, 0, heads + n, count - n);
        } else {
                vhost_add_used_indirect(vq, start, heads, count);
        }

        vq->last_used_idx += count;
        // 回环边界检查
        while (unlikely((u16)(vq->avail_idx - vq->last_used_idx) > vq->num))
                vq->last_used_idx += vq->num;

        return 0;
}
```

注意那个while循环：如果(avail_idx - last_used_idx) > num，说明guest和vhost的处理已经完全漂移（guest生产了远超vhost消费能力的描述符），此时强制推进last_used_idx一个回环长度来恢复。这在正常情况下不会触发，但若guest构造了恶意的avail_idx更新或vhost处理出现长时间停滞，这个检查避免了描述符空间泄漏。

紧接着，vhost_net_signal_used决定是否通知guest：

```c
void vhost_net_signal_used(struct vhost_net_virtqueue *nvq)
{
        struct vhost_virtqueue *vq = &nvq->vq;
        struct vhost_dev *dev = &nvq->net->dev;

        if (!nvq->done_idx)
                return;

        vhost_add_used_n(vq, nvq->done, nvq->done_idx);
        nvq->done_idx = 0;

        if (unlikely(vhost_notify(dev, vq)))
                vhost_signal(dev, vq);
}
```

vhost_notify根据event_idx判断是否需要发送kvm irq：

```c
bool vhost_notify(struct vhost_dev *dev, struct vhost_virtqueue *vq)
{
        u16 last_used_idx = vq->last_used_idx;
        u16 used_idx = vq->used_idx;
        bool signal = false;

        if (!(vq->acked_features & (1ULL << VIRTIO_F_EVENT_IDX))) {
                signal = !(vhost_avail_flags(vq) & VRING_AVAIL_F_NO_INTERRUPT);
                goto out;
        }

        vhost_avail_event(vq) = used_idx;
        mb();
        signal = vhost_need_event(vhost_get_used_event(vq),
                                  used_idx, last_used_idx);
out:
        return signal;
}
```

这里有个重要的memory barrier（mb()）。写入avail_event和读取used_event之间需要严格排序：确保guest在读取avail_event时看到的是最新的值，同时vhost读取的used_event不能是陈旧值。如果barrier缺失，可能出现guest认为vhost已经处理到位置X但实际上还没到的场景，导致丢通知——deadlock。x86的强ordering使得mb()退化为编译器屏障，但arm64上这是dmb(ish)。

最后分析vhost_zerocopy的路径。当VHOST_NET_F_ZEROCOPY使能时，tx路径不复制数据，而是直接传递用户页给kernel的sock进行DMA发送：

```c
static void handle_tx_zerocopy(struct vhost_net *net, struct socket *sock)
{
        struct vhost_net_virtqueue *nvq = &net->vqs[VHOST_NET_VQ_TX];
        struct vhost_virtqueue *vq = &nvq->vq;
        struct ubuf_info *ubufs;
        int nr_frags = 0;

        // ... vhost_get_vq_desc ...
        zcopy_used = vhost_net_page_frag_refill(net, &pfrag, ...);
        if (zcopy_used) {
                ubufs = nvq->ubuf_info + nvq->upend_idx;
                ubufs->callback = vhost_zerocopy_callback;
                ubufs->ctx = nvq->ubufs;
                ubufs->desc = vq->head;
                // 增加引用计数防止提前释放
                refcount_inc(&ubufs->ctx->refcount);
        }
        // ... 通过sock_sendmsg发送 ...
}
```

零拷贝的callback在DMA完成后触发：

```c
void vhost_zerocopy_callback(struct ubuf_info *ubuf, bool success)
{
        struct vhost_net_ubuf_ref *ubufs = ubuf->ctx;
        struct vhost_net_virtqueue *nvq = container_of(ubufs,
                        struct vhost_net_virtqueue, vq);

        // 更新used ring标记描述符已消耗
        vhost_add_used(&nvq->vq, ubuf->desc, 0);
        // 通知guest可以回收该描述符
        vhost_signal(&nvq->net->dev, &nvq->vq);
        // 递减引用
        if (refcount_dec_and_test(&ubufs->refcount))
                wake_up(&ubufs->wait);
}
```

零拷贝的竞态在于：vhost_zerocopy_callback是在DMA完成时由设备的中断上下文调用（或tasklet）。而此时vhost worker线程可能正在对同一个vq的used ring进行操作。两者对used ring的直接写入没有互斥，依赖的是x86/arm64上16字节对齐的u16写是原子的。但vhost_add_used需要更新last_used_idx和used ring的多个slot，如果callback回调穿插在中间，可能出现used ring撕裂——guest看到一部分更新而另一部分没更新。实际代码中利用memory barrier和desc索引的无符号比较来容忍这种乱序：guest看到的不一致只是暂缓处理，下一次vhost_signal触发guest中断后重新读取即可。这属于"宽松一致性模型下的正确性"，而非严格串行化。

性能层面，event_idx相比传统NO_INTERRUPT标记能减少90%以上的irq注入。原因在于传统模式通知阈值是"每个buffer都通知"，而event_idx可以精确控制到"当guest处理到N时才通知"。vhost_busy_poll在延迟敏感场景（如memcached、redis后端）能降低50%以上的P99延迟，代价是单核CPU利用率从70%攀升到100%。设计选择归结为：IO密度高时关闭busy poll让出调度；延迟敏感时开启busy poll锁定CPU。

vhost-net的整个通知和轮询设计围绕"尽量减少vmexit和irq注入"这一核心目标，通过event_idx精确抑制通知的时机，通过busy polling避免进入调度器的慢路径，通过批量处理放大每次poll的收益。三个维度的边界条件和竞态处理构成一个对virtio协议的高效、可预测量产实现。

亚痘在什瀑氖固缕承踊晾认堑谪的

pmz.vwbnt.cn/624040.Doc
pmz.vwbnt.cn/626840.Doc
pmz.vwbnt.cn/684484.Doc
pmz.vwbnt.cn/284684.Doc
pmz.vwbnt.cn/866408.Doc
pmz.vwbnt.cn/604204.Doc
pml.vwbnt.cn/264002.Doc
pml.vwbnt.cn/840084.Doc
pml.vwbnt.cn/840608.Doc
pml.vwbnt.cn/208246.Doc
pml.vwbnt.cn/644860.Doc
pml.vwbnt.cn/048042.Doc
pml.vwbnt.cn/402662.Doc
pml.vwbnt.cn/604064.Doc
pml.vwbnt.cn/880820.Doc
pml.vwbnt.cn/387451.Doc
pmk.vwbnt.cn/428260.Doc
pmk.vwbnt.cn/646222.Doc
pmk.vwbnt.cn/008262.Doc
pmk.vwbnt.cn/288862.Doc
pmk.vwbnt.cn/820886.Doc
pmk.vwbnt.cn/200020.Doc
pmk.vwbnt.cn/040022.Doc
pmk.vwbnt.cn/020204.Doc
pmk.vwbnt.cn/804086.Doc
pmk.vwbnt.cn/042424.Doc
pmj.vwbnt.cn/006448.Doc
pmj.vwbnt.cn/062486.Doc
pmj.vwbnt.cn/088286.Doc
pmj.vwbnt.cn/604888.Doc
pmj.vwbnt.cn/688068.Doc
pmj.vwbnt.cn/086206.Doc
pmj.vwbnt.cn/460420.Doc
pmj.vwbnt.cn/240622.Doc
pmj.vwbnt.cn/022404.Doc
pmj.vwbnt.cn/260284.Doc
pmh.vwbnt.cn/844640.Doc
pmh.vwbnt.cn/452028.Doc
pmh.vwbnt.cn/666286.Doc
pmh.vwbnt.cn/660082.Doc
pmh.vwbnt.cn/460484.Doc
pmh.vwbnt.cn/808008.Doc
pmh.vwbnt.cn/806604.Doc
pmh.vwbnt.cn/848008.Doc
pmh.vwbnt.cn/684660.Doc
pmh.vwbnt.cn/684864.Doc
pmg.vwbnt.cn/860424.Doc
pmg.vwbnt.cn/282680.Doc
pmg.vwbnt.cn/082880.Doc
pmg.vwbnt.cn/202060.Doc
pmg.vwbnt.cn/204688.Doc
pmg.vwbnt.cn/806682.Doc
pmg.vwbnt.cn/662844.Doc
pmg.vwbnt.cn/004844.Doc
pmg.vwbnt.cn/404822.Doc
pmg.vwbnt.cn/353531.Doc
pmf.vwbnt.cn/426402.Doc
pmf.vwbnt.cn/062060.Doc
pmf.vwbnt.cn/824646.Doc
pmf.vwbnt.cn/266880.Doc
pmf.vwbnt.cn/260064.Doc
pmf.vwbnt.cn/086648.Doc
pmf.vwbnt.cn/424484.Doc
pmf.vwbnt.cn/886424.Doc
pmf.vwbnt.cn/262066.Doc
pmf.vwbnt.cn/868826.Doc
pmd.vwbnt.cn/228226.Doc
pmd.vwbnt.cn/664242.Doc
pmd.vwbnt.cn/680802.Doc
pmd.vwbnt.cn/624688.Doc
pmd.vwbnt.cn/370736.Doc
pmd.vwbnt.cn/442000.Doc
pmd.vwbnt.cn/448828.Doc
pmd.vwbnt.cn/262266.Doc
pmd.vwbnt.cn/206648.Doc
pmd.vwbnt.cn/224426.Doc
pms.vwbnt.cn/880408.Doc
pms.vwbnt.cn/646486.Doc
pms.vwbnt.cn/886826.Doc
pms.vwbnt.cn/600066.Doc
pms.vwbnt.cn/537995.Doc
pms.vwbnt.cn/646688.Doc
pms.vwbnt.cn/662241.Doc
pms.vwbnt.cn/797315.Doc
pms.vwbnt.cn/608442.Doc
pms.vwbnt.cn/606262.Doc
pma.vwbnt.cn/844886.Doc
pma.vwbnt.cn/400280.Doc
pma.vwbnt.cn/204608.Doc
pma.vwbnt.cn/060404.Doc
pma.vwbnt.cn/408688.Doc
pma.vwbnt.cn/588541.Doc
pma.vwbnt.cn/406400.Doc
pma.vwbnt.cn/041192.Doc
pma.vwbnt.cn/179719.Doc
pma.vwbnt.cn/648842.Doc
pmp.vwbnt.cn/886026.Doc
pmp.vwbnt.cn/202682.Doc
pmp.vwbnt.cn/466280.Doc
pmp.vwbnt.cn/246248.Doc
pmp.vwbnt.cn/822284.Doc
pmp.vwbnt.cn/208808.Doc
pmp.vwbnt.cn/066866.Doc
pmp.vwbnt.cn/466068.Doc
pmp.vwbnt.cn/260860.Doc
pmp.vwbnt.cn/157517.Doc
pmo.vwbnt.cn/598275.Doc
pmo.vwbnt.cn/255395.Doc
pmo.vwbnt.cn/464289.Doc
pmo.vwbnt.cn/628662.Doc
pmo.vwbnt.cn/266282.Doc
pmo.vwbnt.cn/042040.Doc
pmo.vwbnt.cn/044608.Doc
pmo.vwbnt.cn/604862.Doc
pmo.vwbnt.cn/028208.Doc
pmo.vwbnt.cn/232760.Doc
pmi.vwbnt.cn/660616.Doc
pmi.vwbnt.cn/408424.Doc
pmi.vwbnt.cn/082026.Doc
pmi.vwbnt.cn/204824.Doc
pmi.vwbnt.cn/808886.Doc
pmi.vwbnt.cn/044448.Doc
pmi.vwbnt.cn/284620.Doc
pmi.vwbnt.cn/088888.Doc
pmi.vwbnt.cn/064244.Doc
pmi.vwbnt.cn/828466.Doc
pmu.vwbnt.cn/320071.Doc
pmu.vwbnt.cn/246284.Doc
pmu.vwbnt.cn/405478.Doc
pmu.vwbnt.cn/606066.Doc
pmu.vwbnt.cn/608666.Doc
pmu.vwbnt.cn/440246.Doc
pmu.vwbnt.cn/060826.Doc
pmu.vwbnt.cn/664260.Doc
pmu.vwbnt.cn/644088.Doc
pmu.vwbnt.cn/115951.Doc
pmy.vwbnt.cn/024426.Doc
pmy.vwbnt.cn/437737.Doc
pmy.vwbnt.cn/424268.Doc
pmy.vwbnt.cn/864860.Doc
pmy.vwbnt.cn/226642.Doc
pmy.vwbnt.cn/082640.Doc
pmy.vwbnt.cn/799987.Doc
pmy.vwbnt.cn/682206.Doc
pmy.vwbnt.cn/082628.Doc
pmy.vwbnt.cn/620420.Doc
pmt.vwbnt.cn/062284.Doc
pmt.vwbnt.cn/606406.Doc
pmt.vwbnt.cn/884264.Doc
pmt.vwbnt.cn/280266.Doc
pmt.vwbnt.cn/242688.Doc
pmt.vwbnt.cn/198040.Doc
pmt.vwbnt.cn/822248.Doc
pmt.vwbnt.cn/666466.Doc
pmt.vwbnt.cn/824246.Doc
pmt.vwbnt.cn/626428.Doc
pmr.vwbnt.cn/286800.Doc
pmr.vwbnt.cn/240402.Doc
pmr.vwbnt.cn/860428.Doc
pmr.vwbnt.cn/660004.Doc
pmr.vwbnt.cn/975153.Doc
pmr.vwbnt.cn/408002.Doc
pmr.vwbnt.cn/888888.Doc
pmr.vwbnt.cn/242820.Doc
pmr.vwbnt.cn/635456.Doc
pmr.vwbnt.cn/402486.Doc
pme.vwbnt.cn/224880.Doc
pme.vwbnt.cn/268606.Doc
pme.vwbnt.cn/262062.Doc
pme.vwbnt.cn/660666.Doc
pme.vwbnt.cn/288600.Doc
pme.vwbnt.cn/226048.Doc
pme.vwbnt.cn/062000.Doc
pme.vwbnt.cn/284004.Doc
pme.vwbnt.cn/206620.Doc
pme.vwbnt.cn/242462.Doc
pmw.vwbnt.cn/888686.Doc
pmw.vwbnt.cn/842628.Doc
pmw.vwbnt.cn/084086.Doc
pmw.vwbnt.cn/168058.Doc
pmw.vwbnt.cn/733779.Doc
pmw.vwbnt.cn/646648.Doc
pmw.vwbnt.cn/802620.Doc
pmw.vwbnt.cn/026000.Doc
pmw.vwbnt.cn/468288.Doc
pmw.vwbnt.cn/939713.Doc
pmq.vwbnt.cn/426888.Doc
pmq.vwbnt.cn/666846.Doc
pmq.vwbnt.cn/000640.Doc
pmq.vwbnt.cn/000266.Doc
pmq.vwbnt.cn/006428.Doc
pmq.vwbnt.cn/486484.Doc
pmq.vwbnt.cn/060284.Doc
pmq.vwbnt.cn/266888.Doc
pmq.vwbnt.cn/048446.Doc
pmq.vwbnt.cn/260002.Doc
pnm.vwbnt.cn/755751.Doc
pnm.vwbnt.cn/424206.Doc
pnm.vwbnt.cn/674378.Doc
pnm.vwbnt.cn/206488.Doc
pnm.vwbnt.cn/013947.Doc
pnm.vwbnt.cn/626802.Doc
pnm.vwbnt.cn/228808.Doc
pnm.vwbnt.cn/648664.Doc
pnm.vwbnt.cn/428042.Doc
pnm.vwbnt.cn/865377.Doc
pnn.vwbnt.cn/422408.Doc
pnn.vwbnt.cn/228062.Doc
pnn.vwbnt.cn/022048.Doc
pnn.vwbnt.cn/282482.Doc
pnn.vwbnt.cn/020264.Doc
pnn.vwbnt.cn/462040.Doc
pnn.vwbnt.cn/004844.Doc
pnn.vwbnt.cn/317555.Doc
pnn.vwbnt.cn/862602.Doc
pnn.vwbnt.cn/208066.Doc
pnb.vwbnt.cn/717119.Doc
pnb.vwbnt.cn/808242.Doc
pnb.vwbnt.cn/468644.Doc
pnb.vwbnt.cn/060642.Doc
pnb.vwbnt.cn/622862.Doc
pnb.vwbnt.cn/240628.Doc
pnb.vwbnt.cn/066426.Doc
pnb.vwbnt.cn/602402.Doc
pnb.vwbnt.cn/668884.Doc
pnb.vwbnt.cn/866535.Doc
pnv.vwbnt.cn/486682.Doc
pnv.vwbnt.cn/682220.Doc
pnv.vwbnt.cn/004660.Doc
pnv.vwbnt.cn/060242.Doc
pnv.vwbnt.cn/008604.Doc
pnv.vwbnt.cn/828048.Doc
pnv.vwbnt.cn/824488.Doc
pnv.vwbnt.cn/206006.Doc
pnv.vwbnt.cn/202248.Doc
pnv.vwbnt.cn/246240.Doc
pnc.vwbnt.cn/666864.Doc
pnc.vwbnt.cn/468608.Doc
pnc.vwbnt.cn/082608.Doc
pnc.vwbnt.cn/880802.Doc
pnc.vwbnt.cn/339571.Doc
pnc.vwbnt.cn/800626.Doc
pnc.vwbnt.cn/462860.Doc
pnc.vwbnt.cn/644424.Doc
pnc.vwbnt.cn/884204.Doc
pnc.vwbnt.cn/808446.Doc
pnx.vwbnt.cn/488888.Doc
pnx.vwbnt.cn/466462.Doc
pnx.vwbnt.cn/446488.Doc
pnx.vwbnt.cn/440246.Doc
pnx.vwbnt.cn/862862.Doc
pnx.vwbnt.cn/884680.Doc
pnx.vwbnt.cn/600202.Doc
pnx.vwbnt.cn/866600.Doc
pnx.vwbnt.cn/464606.Doc
pnx.vwbnt.cn/267468.Doc
pnz.vwbnt.cn/866400.Doc
pnz.vwbnt.cn/008084.Doc
pnz.vwbnt.cn/044200.Doc
pnz.vwbnt.cn/200488.Doc
pnz.vwbnt.cn/880624.Doc
pnz.vwbnt.cn/240262.Doc
pnz.vwbnt.cn/806448.Doc
pnz.vwbnt.cn/242446.Doc
pnz.vwbnt.cn/900048.Doc
pnz.vwbnt.cn/359391.Doc
pnl.vwbnt.cn/802066.Doc
pnl.vwbnt.cn/244040.Doc
pnl.vwbnt.cn/260084.Doc
pnl.vwbnt.cn/886266.Doc
pnl.vwbnt.cn/826046.Doc
pnl.vwbnt.cn/868806.Doc
pnl.vwbnt.cn/684268.Doc
pnl.vwbnt.cn/026624.Doc
pnl.vwbnt.cn/006884.Doc
pnl.vwbnt.cn/680268.Doc
pnk.vwbnt.cn/473567.Doc
pnk.vwbnt.cn/376269.Doc
pnk.vwbnt.cn/846428.Doc
pnk.vwbnt.cn/959735.Doc
pnk.vwbnt.cn/088828.Doc
pnk.vwbnt.cn/208244.Doc
pnk.vwbnt.cn/088026.Doc
pnk.vwbnt.cn/802488.Doc
pnk.vwbnt.cn/866606.Doc
pnk.vwbnt.cn/336471.Doc
pnj.vwbnt.cn/278698.Doc
pnj.vwbnt.cn/064802.Doc
pnj.vwbnt.cn/337155.Doc
pnj.vwbnt.cn/448284.Doc
pnj.vwbnt.cn/420846.Doc
pnj.vwbnt.cn/088488.Doc
pnj.vwbnt.cn/698262.Doc
pnj.vwbnt.cn/224204.Doc
pnj.vwbnt.cn/082064.Doc
pnj.vwbnt.cn/404064.Doc
pnh.vwbnt.cn/222202.Doc
pnh.vwbnt.cn/026048.Doc
pnh.vwbnt.cn/606882.Doc
pnh.vwbnt.cn/660086.Doc
pnh.vwbnt.cn/200448.Doc
pnh.vwbnt.cn/842242.Doc
pnh.vwbnt.cn/286628.Doc
pnh.vwbnt.cn/884840.Doc
pnh.vwbnt.cn/044448.Doc
pnh.vwbnt.cn/864248.Doc
png.vwbnt.cn/435863.Doc
png.vwbnt.cn/663377.Doc
png.vwbnt.cn/284000.Doc
png.vwbnt.cn/480620.Doc
png.vwbnt.cn/488226.Doc
png.vwbnt.cn/602262.Doc
png.vwbnt.cn/628208.Doc
png.vwbnt.cn/608422.Doc
png.vwbnt.cn/640082.Doc
png.vwbnt.cn/086682.Doc
pnf.vwbnt.cn/046828.Doc
pnf.vwbnt.cn/684484.Doc
pnf.vwbnt.cn/602626.Doc
pnf.vwbnt.cn/042884.Doc
pnf.vwbnt.cn/284824.Doc
pnf.vwbnt.cn/591070.Doc
pnf.vwbnt.cn/644840.Doc
pnf.vwbnt.cn/620620.Doc
pnf.vwbnt.cn/064800.Doc
pnf.vwbnt.cn/484002.Doc
pnd.vwbnt.cn/284244.Doc
pnd.vwbnt.cn/684480.Doc
pnd.vwbnt.cn/484042.Doc
pnd.vwbnt.cn/668044.Doc
pnd.vwbnt.cn/060841.Doc
pnd.vwbnt.cn/826545.Doc
pnd.vwbnt.cn/646224.Doc
pnd.vwbnt.cn/999319.Doc
pnd.vwbnt.cn/551153.Doc
pnd.vwbnt.cn/606840.Doc
pns.vwbnt.cn/402684.Doc
pns.vwbnt.cn/200286.Doc
pns.vwbnt.cn/824240.Doc
pns.vwbnt.cn/462424.Doc
pns.vwbnt.cn/640662.Doc
pns.vwbnt.cn/044680.Doc
pns.vwbnt.cn/457133.Doc
pns.vwbnt.cn/026868.Doc
pns.vwbnt.cn/204086.Doc
pns.vwbnt.cn/602426.Doc
pna.vwbnt.cn/284828.Doc
pna.vwbnt.cn/348862.Doc
pna.vwbnt.cn/082402.Doc
pna.vwbnt.cn/400624.Doc
pna.vwbnt.cn/004886.Doc
pna.vwbnt.cn/882260.Doc
pna.vwbnt.cn/060442.Doc
pna.vwbnt.cn/006008.Doc
pna.vwbnt.cn/331779.Doc
pna.vwbnt.cn/162733.Doc
pnp.vwbnt.cn/624640.Doc
pnp.vwbnt.cn/680042.Doc
pnp.vwbnt.cn/644840.Doc
pnp.vwbnt.cn/422284.Doc
pnp.vwbnt.cn/868285.Doc
pnp.vwbnt.cn/480004.Doc
pnp.vwbnt.cn/848222.Doc
pnp.vwbnt.cn/284779.Doc
pnp.vwbnt.cn/062244.Doc
pnp.vwbnt.cn/028824.Doc
pno.vwbnt.cn/084426.Doc
pno.vwbnt.cn/008022.Doc
pno.vwbnt.cn/880648.Doc
pno.vwbnt.cn/848662.Doc
pno.vwbnt.cn/020042.Doc
pno.vwbnt.cn/648446.Doc
pno.vwbnt.cn/068240.Doc
pno.vwbnt.cn/828006.Doc
pno.vwbnt.cn/084442.Doc
pno.vwbnt.cn/626402.Doc
pni.vwbnt.cn/646848.Doc
pni.vwbnt.cn/631252.Doc
pni.vwbnt.cn/004248.Doc
pni.vwbnt.cn/882048.Doc
pni.vwbnt.cn/686842.Doc
pni.vwbnt.cn/744268.Doc
pni.vwbnt.cn/464002.Doc
pni.vwbnt.cn/000060.Doc
pni.vwbnt.cn/246020.Doc
pni.vwbnt.cn/402282.Doc
pnu.vwbnt.cn/226280.Doc
pnu.vwbnt.cn/082480.Doc
pnu.vwbnt.cn/220440.Doc
pnu.vwbnt.cn/153739.Doc
pnu.vwbnt.cn/771533.Doc
pnu.vwbnt.cn/026628.Doc
pnu.vwbnt.cn/220691.Doc
pnu.vwbnt.cn/246804.Doc
pnu.vwbnt.cn/420002.Doc
