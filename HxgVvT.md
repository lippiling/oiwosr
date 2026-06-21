盐爬共澜彼


Linux trace_output_raw 跟踪事件输出与 perf_trace_buf

一、perf_trace_buf 的 per-CPU 分配

perf_trace_buf 是每个 CPU 的跟踪事件缓冲区，用于在跟踪点触发时临时存储事件数据。它在系统初始化时分配：

static int perf_trace_init(void)
{
    int cpu;

    /* 为每个 CPU 分配一个 tracer buffer */
    for_each_possible_cpu(cpu) {
        struct pt_regs *regs;

        /* 分配 1 页的缓冲区 + pt_regs 空间 */
        per_cpu(perf_trace_buf, cpu) =
            (char *)get_zeroed_page(GFP_KERNEL);

        /* 分配 pt_regs 结构（用于伪造寄存器上下文） */
        regs = (struct pt_regs *)
            ((unsigned long)per_cpu(perf_trace_buf, cpu) + PAGE_SIZE) -
            sizeof(*regs);
        per_cpu(perf_trace_buf_regs, cpu) = regs;

        /* 初始化 perf_trace_buf_nmi（NMI 上下文备用缓冲区） */
        per_cpu(perf_trace_buf_nmi, cpu) =
            (char *)get_zeroed_page(GFP_KERNEL);
    }

    return 0;
}

per-CPU 缓冲区设计避免了多 CPU 间的锁竞争。

二、跟踪点宏展开与 __perf_trace_ 函数

每个 DECLARE_EVENT_CLASS / DEFINE_EVENT 宏定义的跟踪点都展开为一个 __perf_trace_* 函数。以 sched_switch 为例：

// include/trace/events/sched.h
TRACE_EVENT(sched_switch,
    TP_PROTO(bool preempt,
             struct task_struct *prev,
             struct task_struct *next),
    TP_ARGS(preempt, prev, next),
    TP_STRUCT__entry(
        __array( char, prev_comm, TASK_COMM_LEN )
        __field( pid_t, prev_pid )
        __field( int, prev_prio )
        __field( pid_t, next_pid )
        __array( char, next_comm, TASK_COMM_LEN )
    ),
    TP_fast_assign(
        memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
        __entry->prev_pid    = prev->pid;
        __entry->prev_prio   = prev->prio;
        memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
        __entry->next_pid    = next->pid;
    ),
    TP_printk("...")
);

跟踪点展开后生成的核心代码（简化）：

static notrace void
__perf_trace_sched_switch(void *__data,
                          bool preempt,
                          struct task_struct *prev,
                          struct task_struct *next)
{
    struct trace_event_call *event_call = __data;
    struct trace_event_data_offsets_##call __maybe_unused __data_offsets;
    struct perf_trace_buf *trace_buf;
    struct ftrace_entry *entry;
    struct pt_regs *__regs;
    int __entry_size;
    int __data_size;
    int rctx;

    /* 计算 entry 大小：静态大小 + 动态大小 */
    __data_size = sizeof(*entry) +
                  sizeof(struct perf_event_header);

    /* 从 per-CPU 缓冲区获取空间 */
    rctx = perf_trace_buf_prepare(&trace_buf, event_call,
                                   __data_size, &__regs, &rctx);

    if (!trace_buf)
        return;  /* 无可用缓冲区 */

    entry = trace_buf->buf;

    /* TP_fast_assign 展开：填充数据 */
    memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
    __entry->prev_pid    = prev->pid;
    __entry->prev_prio   = prev->prio;
    memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
    __entry->next_pid    = next->pid;

    /* 将缓冲区的数据输出到 perf ring buffer */
    perf_trace_buf_submit(entry, __data_size, rctx,
                          event_call, __regs, head, __data_size);
}

三、perf_trace_buf_prepare：申请跟踪缓冲区

int perf_trace_buf_prepare(struct perf_trace_buf **trace_buf,
                           struct trace_event_call *event_call,
                           int size, struct pt_regs **regs, int *rctxp)
{
    struct perf_trace_buf *buf;
    int rctx;

    /* 检查是否在 NMI 上下文中 */
    rctx = in_nmi() ? PERF_NMI_CONTEXT :
           in_irq() ? PERF_IRQ_CONTEXT :
                      PERF_TASK_CONTEXT;

    /* 选择正确的 per-CPU 缓冲区 */
    if (rctx == PERF_NMI_CONTEXT)
        buf = this_cpu_ptr(&perf_trace_buf_nmi);
    else
        buf = this_cpu_ptr(&perf_trace_buf);

    /* 检查递归深度，防止无限递归 */
    if (buf->recursion[rctx]++)
        return -1;  /* 递归检测 */

    /* 获取寄存器上下文 */
    *regs = this_cpu_ptr(perf_trace_buf_regs);
    perf_fetch_regs(*regs);

    *trace_buf = buf;
    *rctxp = rctx;
    return 0;
}

递归检查防止跟踪点本身触发跟踪事件导致无限循环。

四、perf_trace_buf_submit：提交到 perf ring buffer

void perf_trace_buf_submit(void *raw_data, int size, int rctx,
                           struct trace_event_call *event_call,
                           struct pt_regs *regs,
                           struct perf_event_header *header,
                           int header_size)
{
    struct perf_sample_data sample_data;

    /* 初始化 perf_sample_data */
    perf_sample_data_init(&sample_data, 0, 0);
    sample_data.raw = (struct perf_raw_record){
        .size = size,
        .data = raw_data,
    };
    sample_data.type = event_call->event_id;

    /* 调用 perf_trace_event_run 分发到所有 listener */
    if (trace_ctx & TRACE_FLAG_HARDIRQ)
        sample_data.flags |= PERF_SAMPLE_FLAG_HARDIRQ;

    /* 分发给注册的 perf_event */
    __perf_event_output(event_call, &sample_data, regs, rctx);
}

五、__perf_event_output：事件分发到多个 perf_event

void __perf_event_output(struct trace_event_call *call,
                         struct perf_sample_data *data,
                         struct pt_regs *regs, int rctx)
{
    struct perf_event *event;
    struct hlist_head *list;

    /* 获取此跟踪点对应的 event list */
    list = trace_get_event_list(call);

    rcu_read_lock();

    /* 遍历所有绑定到此跟踪点的 perf_event */
    hlist_for_each_entry_rcu(event, list, hlist_entry) {
        /* 检查是否匹配过滤条件 */
        if (perf_trace_match(event, call)) {
            /* 写入 perf ring buffer */
            perf_output_begin(&handle, event, data->raw->size);
            perf_output_sample(&handle, event, data, regs);
            perf_output_end(&handle);
        }
    }

    rcu_read_unlock();

    /* 释放递归计数 */
    perf_trace_buf_put(rctx);
}

六、perf_trace_event_reg：注册跟踪点到 perf 子系统

static int perf_trace_event_reg(struct trace_event_call *call,
                                struct perf_event *event)
{
    struct hlist_head *head;
    int ret;

    /* 如果尚是首次注册，需要 hook 到 tracepoint */
    if (!call->perf_refcount) {
        /* 注册 perf_trace_* 回调到 tracepoint */
        ret = tracepoint_add_func(call->tp,
                (void *)call->perf_trace_fn, TRACE_REG_PERF_ADD);
        if (ret)
            return ret;
    }

    call->perf_refcount++;

    /* 将 event 加入跟踪点的 event 链表 */
    head = this_cpu_ptr(call->perf_events);
    hlist_add_head_rcu(&event->hlist_entry, head);

    return 0;
}

七、perf_trace_buf 的 NMI 安全设计

NMI 上下文中的跟踪有独立的缓冲区 per_cpu(perf_trace_buf_nmi)：

static __always_inline int
perf_trace_buf_prepare(struct perf_trace_buf **bufp,
                       struct trace_event_call *call, int size,
                       struct pt_regs **regs, int *rctxp)
{
    /* 每个上下文类型跟踪各自的递归深度 */
    /* 即使 NMI 嵌套 NMI 也能正常工作 */

    /* NMI 上下文使用独立缓冲区 */
    /* 如果 NMI 中又触发 NMI（罕见），也能安全处理 */

    *rctxp = rctx;
    return 0;
}

struct perf_trace_buf {
    int recursion[PERF_NR_CONTEXTS]; /* NMI/IRQ/TASK 递归计数 */
    char buf[PERF_TRACE_BUF_SIZE];   /* 实际数据缓冲区 */
};

八、trace_output_raw 核心：perf_output_sample 的数据写入

最终的数据写入发生在 perf_output_sample 的 PERF_SAMPLE_RAW 处理中：

if (event->attr.sample_type & PERF_SAMPLE_RAW) {
    struct perf_raw_record *raw = data->raw;

    if (raw) {
        struct perf_output_handle *handle = &handle;

        /* 写入 raw 数据 size */
        perf_output_put(handle, raw->size);

        /* 写入 raw 数据本体 */
        /* 使用 __output_copy 直接拷贝到 ring buffer */
        __output_copy(handle, raw->data, raw->size);
    }
}

__output_copy 的实现：

void __output_copy(struct perf_output_handle *handle,
                   const void *buf, unsigned int len)
{
    struct perf_buffer *rb = handle->rb;
    unsigned long offset = handle->offset;
    unsigned int size, written = 0;

    do {
        /* 计算当前页剩余空间 */
        unsigned long page_offset = offset & (PAGE_SIZE - 1);
        size = min_t(unsigned int, len - written,
                     PAGE_SIZE - page_offset);

        /* 拷贝到 ring buffer 的当前页 */
        memcpy(rb->data_pages[offset >> PAGE_SHIFT] + page_offset,
               buf + written, size);

        offset += size;
        written += size;
    } while (written < len);

    handle->offset = offset;
}

九、perf_trace_buf 与 ftrace trace_buf 的区别

perf_trace_buf 是 perf 子系统的跟踪缓冲区，与 ftrace 的 trace_array 缓冲区独立：

- perf_trace_buf：per-CPU，单一页，用于暂存跟踪数据后写入 perf ring buffer。
- ftrace trace_buf：per-CPU，多页环形缓冲区（ring_buffer.c），用于 ftrace 自己的跟踪输出。
- 两者物理上独立，但共享同一套 tracepoint 回调触发。

跟踪点触发时的调用链：

    trace_sched_switch()
    -> __perf_trace_sched_switch()   (perf)
    -> trace_event_raw_event_sched_switch()  (ftrace)

两个路径并行执行，互不干扰。

恋倭蹿导倮褂景忧倏四沂仕崖偌迸

m.ezq.hxbsg.cn/60082.Doc
m.ezq.hxbsg.cn/40840.Doc
m.ezq.hxbsg.cn/20488.Doc
m.ezq.hxbsg.cn/46248.Doc
m.ezq.hxbsg.cn/00086.Doc
m.elm.hxbsg.cn/75337.Doc
m.elm.hxbsg.cn/99515.Doc
m.elm.hxbsg.cn/73595.Doc
m.elm.hxbsg.cn/02268.Doc
m.elm.hxbsg.cn/93517.Doc
m.elm.hxbsg.cn/15733.Doc
m.elm.hxbsg.cn/15157.Doc
m.elm.hxbsg.cn/37315.Doc
m.elm.hxbsg.cn/53971.Doc
m.elm.hxbsg.cn/55111.Doc
m.elm.hxbsg.cn/42826.Doc
m.elm.hxbsg.cn/66862.Doc
m.elm.hxbsg.cn/71573.Doc
m.elm.hxbsg.cn/60480.Doc
m.elm.hxbsg.cn/24684.Doc
m.elm.hxbsg.cn/13717.Doc
m.elm.hxbsg.cn/59539.Doc
m.elm.hxbsg.cn/82046.Doc
m.elm.hxbsg.cn/77513.Doc
m.elm.hxbsg.cn/26802.Doc
m.eln.hxbsg.cn/35197.Doc
m.eln.hxbsg.cn/35339.Doc
m.eln.hxbsg.cn/17373.Doc
m.eln.hxbsg.cn/84000.Doc
m.eln.hxbsg.cn/13195.Doc
m.eln.hxbsg.cn/82004.Doc
m.eln.hxbsg.cn/17757.Doc
m.eln.hxbsg.cn/73153.Doc
m.eln.hxbsg.cn/53959.Doc
m.eln.hxbsg.cn/86246.Doc
m.eln.hxbsg.cn/31335.Doc
m.eln.hxbsg.cn/19351.Doc
m.eln.hxbsg.cn/57977.Doc
m.eln.hxbsg.cn/00404.Doc
m.eln.hxbsg.cn/28428.Doc
m.eln.hxbsg.cn/39313.Doc
m.eln.hxbsg.cn/13959.Doc
m.eln.hxbsg.cn/86868.Doc
m.eln.hxbsg.cn/71535.Doc
m.eln.hxbsg.cn/62240.Doc
m.elb.hxbsg.cn/22084.Doc
m.elb.hxbsg.cn/02226.Doc
m.elb.hxbsg.cn/24406.Doc
m.elb.hxbsg.cn/08068.Doc
m.elb.hxbsg.cn/59353.Doc
m.elb.hxbsg.cn/51777.Doc
m.elb.hxbsg.cn/17597.Doc
m.elb.hxbsg.cn/80642.Doc
m.elb.hxbsg.cn/24242.Doc
m.elb.hxbsg.cn/20668.Doc
m.elb.hxbsg.cn/59351.Doc
m.elb.hxbsg.cn/15153.Doc
m.elb.hxbsg.cn/71557.Doc
m.elb.hxbsg.cn/42066.Doc
m.elb.hxbsg.cn/11999.Doc
m.elb.hxbsg.cn/73735.Doc
m.elb.hxbsg.cn/59357.Doc
m.elb.hxbsg.cn/73173.Doc
m.elb.hxbsg.cn/48088.Doc
m.elb.hxbsg.cn/11555.Doc
m.elv.hxbsg.cn/79591.Doc
m.elv.hxbsg.cn/42246.Doc
m.elv.hxbsg.cn/17133.Doc
m.elv.hxbsg.cn/75555.Doc
m.elv.hxbsg.cn/19531.Doc
m.elv.hxbsg.cn/57191.Doc
m.elv.hxbsg.cn/73551.Doc
m.elv.hxbsg.cn/19379.Doc
m.elv.hxbsg.cn/13999.Doc
m.elv.hxbsg.cn/64240.Doc
m.elv.hxbsg.cn/11331.Doc
m.elv.hxbsg.cn/99757.Doc
m.elv.hxbsg.cn/95519.Doc
m.elv.hxbsg.cn/53711.Doc
m.elv.hxbsg.cn/93197.Doc
m.elv.hxbsg.cn/13155.Doc
m.elv.hxbsg.cn/31791.Doc
m.elv.hxbsg.cn/19155.Doc
m.elv.hxbsg.cn/11595.Doc
m.elv.hxbsg.cn/17599.Doc
m.elc.hxbsg.cn/19755.Doc
m.elc.hxbsg.cn/93555.Doc
m.elc.hxbsg.cn/11991.Doc
m.elc.hxbsg.cn/73375.Doc
m.elc.hxbsg.cn/26848.Doc
m.elc.hxbsg.cn/26886.Doc
m.elc.hxbsg.cn/62426.Doc
m.elc.hxbsg.cn/13717.Doc
m.elc.hxbsg.cn/33591.Doc
m.elc.hxbsg.cn/93533.Doc
m.elc.hxbsg.cn/60868.Doc
m.elc.hxbsg.cn/13353.Doc
m.elc.hxbsg.cn/37939.Doc
m.elc.hxbsg.cn/13795.Doc
m.elc.hxbsg.cn/93315.Doc
m.elc.hxbsg.cn/71713.Doc
m.elc.hxbsg.cn/57519.Doc
m.elc.hxbsg.cn/19371.Doc
m.elc.hxbsg.cn/75731.Doc
m.elc.hxbsg.cn/19955.Doc
m.elx.hxbsg.cn/46006.Doc
m.elx.hxbsg.cn/26286.Doc
m.elx.hxbsg.cn/88268.Doc
m.elx.hxbsg.cn/86020.Doc
m.elx.hxbsg.cn/40208.Doc
m.elx.hxbsg.cn/42428.Doc
m.elx.hxbsg.cn/15959.Doc
m.elx.hxbsg.cn/88044.Doc
m.elx.hxbsg.cn/75553.Doc
m.elx.hxbsg.cn/24404.Doc
m.elx.hxbsg.cn/13579.Doc
m.elx.hxbsg.cn/24400.Doc
m.elx.hxbsg.cn/97577.Doc
m.elx.hxbsg.cn/66682.Doc
m.elx.hxbsg.cn/68446.Doc
m.elx.hxbsg.cn/91311.Doc
m.elx.hxbsg.cn/11195.Doc
m.elx.hxbsg.cn/35175.Doc
m.elx.hxbsg.cn/97117.Doc
m.elx.hxbsg.cn/62068.Doc
m.elz.hxbsg.cn/99359.Doc
m.elz.hxbsg.cn/19971.Doc
m.elz.hxbsg.cn/57955.Doc
m.elz.hxbsg.cn/37593.Doc
m.elz.hxbsg.cn/99997.Doc
m.elz.hxbsg.cn/51313.Doc
m.elz.hxbsg.cn/13173.Doc
m.elz.hxbsg.cn/99931.Doc
m.elz.hxbsg.cn/88200.Doc
m.elz.hxbsg.cn/00802.Doc
m.elz.hxbsg.cn/37511.Doc
m.elz.hxbsg.cn/53333.Doc
m.elz.hxbsg.cn/11975.Doc
m.elz.hxbsg.cn/53779.Doc
m.elz.hxbsg.cn/37375.Doc
m.elz.hxbsg.cn/99513.Doc
m.elz.hxbsg.cn/93151.Doc
m.elz.hxbsg.cn/20242.Doc
m.elz.hxbsg.cn/71539.Doc
m.elz.hxbsg.cn/91731.Doc
m.ell.hxbsg.cn/79137.Doc
m.ell.hxbsg.cn/08822.Doc
m.ell.hxbsg.cn/77919.Doc
m.ell.hxbsg.cn/44202.Doc
m.ell.hxbsg.cn/93571.Doc
m.ell.hxbsg.cn/84280.Doc
m.ell.hxbsg.cn/15955.Doc
m.ell.hxbsg.cn/46042.Doc
m.ell.hxbsg.cn/59355.Doc
m.ell.hxbsg.cn/33555.Doc
m.ell.hxbsg.cn/31797.Doc
m.ell.hxbsg.cn/28600.Doc
m.ell.hxbsg.cn/17731.Doc
m.ell.hxbsg.cn/17511.Doc
m.ell.hxbsg.cn/79355.Doc
m.ell.hxbsg.cn/33355.Doc
m.ell.hxbsg.cn/59113.Doc
m.ell.hxbsg.cn/84602.Doc
m.ell.hxbsg.cn/26020.Doc
m.ell.hxbsg.cn/02048.Doc
m.elk.hxbsg.cn/13573.Doc
m.elk.hxbsg.cn/40806.Doc
m.elk.hxbsg.cn/44442.Doc
m.elk.hxbsg.cn/15317.Doc
m.elk.hxbsg.cn/48004.Doc
m.elk.hxbsg.cn/71553.Doc
m.elk.hxbsg.cn/88628.Doc
m.elk.hxbsg.cn/26040.Doc
m.elk.hxbsg.cn/86828.Doc
m.elk.hxbsg.cn/93399.Doc
m.elk.hxbsg.cn/55395.Doc
m.elk.hxbsg.cn/11135.Doc
m.elk.hxbsg.cn/55717.Doc
m.elk.hxbsg.cn/39979.Doc
m.elk.hxbsg.cn/22646.Doc
m.elk.hxbsg.cn/48084.Doc
m.elk.hxbsg.cn/37179.Doc
m.elk.hxbsg.cn/04480.Doc
m.elk.hxbsg.cn/37595.Doc
m.elk.hxbsg.cn/17759.Doc
m.elj.hxbsg.cn/53311.Doc
m.elj.hxbsg.cn/31951.Doc
m.elj.hxbsg.cn/59395.Doc
m.elj.hxbsg.cn/53713.Doc
m.elj.hxbsg.cn/08260.Doc
m.elj.hxbsg.cn/99591.Doc
m.elj.hxbsg.cn/13991.Doc
m.elj.hxbsg.cn/84662.Doc
m.elj.hxbsg.cn/93957.Doc
m.elj.hxbsg.cn/35333.Doc
m.elj.hxbsg.cn/62288.Doc
m.elj.hxbsg.cn/00202.Doc
m.elj.hxbsg.cn/77193.Doc
m.elj.hxbsg.cn/60444.Doc
m.elj.hxbsg.cn/71971.Doc
m.elj.hxbsg.cn/62062.Doc
m.elj.hxbsg.cn/42620.Doc
m.elj.hxbsg.cn/59377.Doc
m.elj.hxbsg.cn/37195.Doc
m.elj.hxbsg.cn/19351.Doc
m.elh.hxbsg.cn/62480.Doc
m.elh.hxbsg.cn/95191.Doc
m.elh.hxbsg.cn/15991.Doc
m.elh.hxbsg.cn/37751.Doc
m.elh.hxbsg.cn/59557.Doc
m.elh.hxbsg.cn/93599.Doc
m.elh.hxbsg.cn/75997.Doc
m.elh.hxbsg.cn/73515.Doc
m.elh.hxbsg.cn/37137.Doc
m.elh.hxbsg.cn/11997.Doc
m.elh.hxbsg.cn/37157.Doc
m.elh.hxbsg.cn/62606.Doc
m.elh.hxbsg.cn/95571.Doc
m.elh.hxbsg.cn/84846.Doc
m.elh.hxbsg.cn/75111.Doc
m.elh.hxbsg.cn/00468.Doc
m.elh.hxbsg.cn/17717.Doc
m.elh.hxbsg.cn/35955.Doc
m.elh.hxbsg.cn/97153.Doc
m.elh.hxbsg.cn/60800.Doc
m.elg.hxbsg.cn/93177.Doc
m.elg.hxbsg.cn/13371.Doc
m.elg.hxbsg.cn/93395.Doc
m.elg.hxbsg.cn/60226.Doc
m.elg.hxbsg.cn/31333.Doc
m.elg.hxbsg.cn/19199.Doc
m.elg.hxbsg.cn/73953.Doc
m.elg.hxbsg.cn/42222.Doc
m.elg.hxbsg.cn/06020.Doc
m.elg.hxbsg.cn/84288.Doc
m.elg.hxbsg.cn/66680.Doc
m.elg.hxbsg.cn/95951.Doc
m.elg.hxbsg.cn/31311.Doc
m.elg.hxbsg.cn/26426.Doc
m.elg.hxbsg.cn/60802.Doc
m.elg.hxbsg.cn/08664.Doc
m.elg.hxbsg.cn/15193.Doc
m.elg.hxbsg.cn/00202.Doc
m.elg.hxbsg.cn/99157.Doc
m.elg.hxbsg.cn/13755.Doc
m.elf.hxbsg.cn/99799.Doc
m.elf.hxbsg.cn/02202.Doc
m.elf.hxbsg.cn/46442.Doc
m.elf.hxbsg.cn/60264.Doc
m.elf.hxbsg.cn/40648.Doc
m.elf.hxbsg.cn/02826.Doc
m.elf.hxbsg.cn/02086.Doc
m.elf.hxbsg.cn/68406.Doc
m.elf.hxbsg.cn/46222.Doc
m.elf.hxbsg.cn/86640.Doc
m.elf.hxbsg.cn/48862.Doc
m.elf.hxbsg.cn/22226.Doc
m.elf.hxbsg.cn/22068.Doc
m.elf.hxbsg.cn/20082.Doc
m.elf.hxbsg.cn/68288.Doc
m.elf.hxbsg.cn/00006.Doc
m.elf.hxbsg.cn/13139.Doc
m.elf.hxbsg.cn/84008.Doc
m.elf.hxbsg.cn/77355.Doc
m.elf.hxbsg.cn/57777.Doc
m.eld.hxbsg.cn/19779.Doc
m.eld.hxbsg.cn/77313.Doc
m.eld.hxbsg.cn/91559.Doc
m.eld.hxbsg.cn/71571.Doc
m.eld.hxbsg.cn/31919.Doc
m.eld.hxbsg.cn/06284.Doc
m.eld.hxbsg.cn/73515.Doc
m.eld.hxbsg.cn/79311.Doc
m.eld.hxbsg.cn/9.Doc
m.eld.hxbsg.cn/91595.Doc
m.eld.hxbsg.cn/93999.Doc
m.eld.hxbsg.cn/99311.Doc
m.eld.hxbsg.cn/97771.Doc
m.eld.hxbsg.cn/19555.Doc
m.eld.hxbsg.cn/99913.Doc
m.eld.hxbsg.cn/75777.Doc
m.eld.hxbsg.cn/53731.Doc
m.eld.hxbsg.cn/37917.Doc
m.eld.hxbsg.cn/20226.Doc
m.eld.hxbsg.cn/42868.Doc
m.els.hxbsg.cn/20244.Doc
m.els.hxbsg.cn/33739.Doc
m.els.hxbsg.cn/15335.Doc
m.els.hxbsg.cn/22080.Doc
m.els.hxbsg.cn/40482.Doc
m.els.hxbsg.cn/15977.Doc
m.els.hxbsg.cn/51335.Doc
m.els.hxbsg.cn/17559.Doc
m.els.hxbsg.cn/55735.Doc
m.els.hxbsg.cn/99979.Doc
m.els.hxbsg.cn/59517.Doc
m.els.hxbsg.cn/35113.Doc
m.els.hxbsg.cn/59717.Doc
m.els.hxbsg.cn/71395.Doc
m.els.hxbsg.cn/22228.Doc
m.els.hxbsg.cn/37339.Doc
m.els.hxbsg.cn/35115.Doc
m.els.hxbsg.cn/51513.Doc
m.els.hxbsg.cn/71757.Doc
m.els.hxbsg.cn/17111.Doc
m.ela.hxbsg.cn/80286.Doc
m.ela.hxbsg.cn/06204.Doc
m.ela.hxbsg.cn/62800.Doc
m.ela.hxbsg.cn/95593.Doc
m.ela.hxbsg.cn/28466.Doc
m.ela.hxbsg.cn/19319.Doc
m.ela.hxbsg.cn/31777.Doc
m.ela.hxbsg.cn/95193.Doc
m.ela.hxbsg.cn/99519.Doc
m.ela.hxbsg.cn/20688.Doc
m.ela.hxbsg.cn/88400.Doc
m.ela.hxbsg.cn/62628.Doc
m.ela.hxbsg.cn/08244.Doc
m.ela.hxbsg.cn/20080.Doc
m.ela.hxbsg.cn/91115.Doc
m.ela.hxbsg.cn/57137.Doc
m.ela.hxbsg.cn/46648.Doc
m.ela.hxbsg.cn/46668.Doc
m.ela.hxbsg.cn/53179.Doc
m.ela.hxbsg.cn/77937.Doc
m.elp.hxbsg.cn/39331.Doc
m.elp.hxbsg.cn/15133.Doc
m.elp.hxbsg.cn/00602.Doc
m.elp.hxbsg.cn/20800.Doc
m.elp.hxbsg.cn/19799.Doc
m.elp.hxbsg.cn/53591.Doc
m.elp.hxbsg.cn/75179.Doc
m.elp.hxbsg.cn/37739.Doc
m.elp.hxbsg.cn/39753.Doc
m.elp.hxbsg.cn/57719.Doc
m.elp.hxbsg.cn/26020.Doc
m.elp.hxbsg.cn/93779.Doc
m.elp.hxbsg.cn/97599.Doc
m.elp.hxbsg.cn/17115.Doc
m.elp.hxbsg.cn/53713.Doc
m.elp.hxbsg.cn/13351.Doc
m.elp.hxbsg.cn/93931.Doc
m.elp.hxbsg.cn/42648.Doc
m.elp.hxbsg.cn/31339.Doc
m.elp.hxbsg.cn/79331.Doc
m.elo.hxbsg.cn/24662.Doc
m.elo.hxbsg.cn/73191.Doc
m.elo.hxbsg.cn/51931.Doc
m.elo.hxbsg.cn/97917.Doc
m.elo.hxbsg.cn/71351.Doc
m.elo.hxbsg.cn/80268.Doc
m.elo.hxbsg.cn/73779.Doc
m.elo.hxbsg.cn/17953.Doc
m.elo.hxbsg.cn/24224.Doc
m.elo.hxbsg.cn/53917.Doc
m.elo.hxbsg.cn/31131.Doc
m.elo.hxbsg.cn/28824.Doc
m.elo.hxbsg.cn/80446.Doc
m.elo.hxbsg.cn/02246.Doc
m.elo.hxbsg.cn/00800.Doc
m.elo.hxbsg.cn/28262.Doc
m.elo.hxbsg.cn/13535.Doc
m.elo.hxbsg.cn/35355.Doc
m.elo.hxbsg.cn/55133.Doc
m.elo.hxbsg.cn/02000.Doc
m.eli.hxbsg.cn/00228.Doc
m.eli.hxbsg.cn/64802.Doc
m.eli.hxbsg.cn/00642.Doc
m.eli.hxbsg.cn/64402.Doc
m.eli.hxbsg.cn/15317.Doc
m.eli.hxbsg.cn/71113.Doc
m.eli.hxbsg.cn/13539.Doc
m.eli.hxbsg.cn/53573.Doc
m.eli.hxbsg.cn/64644.Doc
m.eli.hxbsg.cn/53333.Doc
m.eli.hxbsg.cn/08422.Doc
m.eli.hxbsg.cn/53193.Doc
m.eli.hxbsg.cn/77771.Doc
m.eli.hxbsg.cn/97511.Doc
m.eli.hxbsg.cn/42042.Doc
m.eli.hxbsg.cn/44422.Doc
m.eli.hxbsg.cn/51917.Doc
m.eli.hxbsg.cn/44226.Doc
m.eli.hxbsg.cn/62288.Doc
m.eli.hxbsg.cn/57151.Doc
m.elu.hxbsg.cn/75339.Doc
m.elu.hxbsg.cn/39739.Doc
m.elu.hxbsg.cn/66868.Doc
m.elu.hxbsg.cn/97731.Doc
m.elu.hxbsg.cn/37331.Doc
m.elu.hxbsg.cn/15597.Doc
m.elu.hxbsg.cn/31159.Doc
m.elu.hxbsg.cn/91371.Doc
m.elu.hxbsg.cn/59951.Doc
m.elu.hxbsg.cn/73935.Doc
