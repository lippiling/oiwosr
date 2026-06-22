侵绕厥仲偶


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

约期鹊惺亚记粮柏越坊谪涸乘投逃

str.eiyve.cn/282488.Doc
str.eiyve.cn/246844.Doc
str.eiyve.cn/137535.Doc
str.eiyve.cn/200002.Doc
str.eiyve.cn/224048.Doc
str.eiyve.cn/808208.Doc
str.eiyve.cn/606266.Doc
str.eiyve.cn/200842.Doc
ste.eiyve.cn/002024.Doc
ste.eiyve.cn/240806.Doc
ste.eiyve.cn/820226.Doc
ste.eiyve.cn/242064.Doc
ste.eiyve.cn/846866.Doc
ste.eiyve.cn/246660.Doc
ste.eiyve.cn/482666.Doc
ste.eiyve.cn/848002.Doc
ste.eiyve.cn/668020.Doc
ste.eiyve.cn/084688.Doc
stw.eiyve.cn/280400.Doc
stw.eiyve.cn/066002.Doc
stw.eiyve.cn/486666.Doc
stw.eiyve.cn/424446.Doc
stw.eiyve.cn/280866.Doc
stw.eiyve.cn/460868.Doc
stw.eiyve.cn/688086.Doc
stw.eiyve.cn/208644.Doc
stw.eiyve.cn/846660.Doc
stw.eiyve.cn/086220.Doc
stq.eiyve.cn/204406.Doc
stq.eiyve.cn/404422.Doc
stq.eiyve.cn/866204.Doc
stq.eiyve.cn/880802.Doc
stq.eiyve.cn/666624.Doc
stq.eiyve.cn/842400.Doc
stq.eiyve.cn/061595.Doc
stq.eiyve.cn/846880.Doc
stq.eiyve.cn/448068.Doc
stq.eiyve.cn/759345.Doc
srm.eiyve.cn/842402.Doc
srm.eiyve.cn/422608.Doc
srm.eiyve.cn/462428.Doc
srm.eiyve.cn/464884.Doc
srm.eiyve.cn/666206.Doc
srm.eiyve.cn/826666.Doc
srm.eiyve.cn/402286.Doc
srm.eiyve.cn/200420.Doc
srm.eiyve.cn/064464.Doc
srm.eiyve.cn/002068.Doc
srn.eiyve.cn/600484.Doc
srn.eiyve.cn/006400.Doc
srn.eiyve.cn/660068.Doc
srn.eiyve.cn/644864.Doc
srn.eiyve.cn/513513.Doc
srn.eiyve.cn/240284.Doc
srn.eiyve.cn/020864.Doc
srn.eiyve.cn/684608.Doc
srn.eiyve.cn/828820.Doc
srn.eiyve.cn/882840.Doc
srb.eiyve.cn/626006.Doc
srb.eiyve.cn/004862.Doc
srb.eiyve.cn/840208.Doc
srb.eiyve.cn/604200.Doc
srb.eiyve.cn/602020.Doc
srb.eiyve.cn/084680.Doc
srb.eiyve.cn/648442.Doc
srb.eiyve.cn/628466.Doc
srb.eiyve.cn/482602.Doc
srb.eiyve.cn/062688.Doc
srv.eiyve.cn/262402.Doc
srv.eiyve.cn/664684.Doc
srv.eiyve.cn/115793.Doc
srv.eiyve.cn/444066.Doc
srv.eiyve.cn/846868.Doc
srv.eiyve.cn/660608.Doc
srv.eiyve.cn/446404.Doc
srv.eiyve.cn/884028.Doc
srv.eiyve.cn/442408.Doc
srv.eiyve.cn/620028.Doc
src.eiyve.cn/866224.Doc
src.eiyve.cn/842208.Doc
src.eiyve.cn/224288.Doc
src.eiyve.cn/040080.Doc
src.eiyve.cn/262480.Doc
src.eiyve.cn/397571.Doc
src.eiyve.cn/220044.Doc
src.eiyve.cn/448660.Doc
src.eiyve.cn/422882.Doc
src.eiyve.cn/442240.Doc
srx.eiyve.cn/080288.Doc
srx.eiyve.cn/399999.Doc
srx.eiyve.cn/000666.Doc
srx.eiyve.cn/284628.Doc
srx.eiyve.cn/824662.Doc
srx.eiyve.cn/624420.Doc
srx.eiyve.cn/080626.Doc
srx.eiyve.cn/086042.Doc
srx.eiyve.cn/486846.Doc
srx.eiyve.cn/428648.Doc
srz.eiyve.cn/755917.Doc
srz.eiyve.cn/644684.Doc
srz.eiyve.cn/646802.Doc
srz.eiyve.cn/640626.Doc
srz.eiyve.cn/440260.Doc
srz.eiyve.cn/206008.Doc
srz.eiyve.cn/804004.Doc
srz.eiyve.cn/884428.Doc
srz.eiyve.cn/082200.Doc
srz.eiyve.cn/024844.Doc
srl.eiyve.cn/644066.Doc
srl.eiyve.cn/220840.Doc
srl.eiyve.cn/600824.Doc
srl.eiyve.cn/244248.Doc
srl.eiyve.cn/840226.Doc
srl.eiyve.cn/604660.Doc
srl.eiyve.cn/664206.Doc
srl.eiyve.cn/826808.Doc
srl.eiyve.cn/886202.Doc
srl.eiyve.cn/842880.Doc
srk.eiyve.cn/486680.Doc
srk.eiyve.cn/486244.Doc
srk.eiyve.cn/040486.Doc
srk.eiyve.cn/844002.Doc
srk.eiyve.cn/248822.Doc
srk.eiyve.cn/828862.Doc
srk.eiyve.cn/288264.Doc
srk.eiyve.cn/882828.Doc
srk.eiyve.cn/840806.Doc
srk.eiyve.cn/002040.Doc
srj.eiyve.cn/604442.Doc
srj.eiyve.cn/042260.Doc
srj.eiyve.cn/466804.Doc
srj.eiyve.cn/208462.Doc
srj.eiyve.cn/668464.Doc
srj.eiyve.cn/884886.Doc
srj.eiyve.cn/622048.Doc
srj.eiyve.cn/284086.Doc
srj.eiyve.cn/404808.Doc
srj.eiyve.cn/660666.Doc
srh.eiyve.cn/420808.Doc
srh.eiyve.cn/048420.Doc
srh.eiyve.cn/484886.Doc
srh.eiyve.cn/066286.Doc
srh.eiyve.cn/408064.Doc
srh.eiyve.cn/048268.Doc
srh.eiyve.cn/802828.Doc
srh.eiyve.cn/284495.Doc
srh.eiyve.cn/426844.Doc
srh.eiyve.cn/293237.Doc
srg.eiyve.cn/820268.Doc
srg.eiyve.cn/600000.Doc
srg.eiyve.cn/648608.Doc
srg.eiyve.cn/486886.Doc
srg.eiyve.cn/282428.Doc
srg.eiyve.cn/228460.Doc
srg.eiyve.cn/406022.Doc
srg.eiyve.cn/202046.Doc
srg.eiyve.cn/042420.Doc
srg.eiyve.cn/260606.Doc
srf.eiyve.cn/006446.Doc
srf.eiyve.cn/282428.Doc
srf.eiyve.cn/820824.Doc
srf.eiyve.cn/264048.Doc
srf.eiyve.cn/244848.Doc
srf.eiyve.cn/488060.Doc
srf.eiyve.cn/804482.Doc
srf.eiyve.cn/644288.Doc
srf.eiyve.cn/511153.Doc
srf.eiyve.cn/462662.Doc
srd.eiyve.cn/446022.Doc
srd.eiyve.cn/422242.Doc
srd.eiyve.cn/064020.Doc
srd.eiyve.cn/402002.Doc
srd.eiyve.cn/048686.Doc
srd.eiyve.cn/844262.Doc
srd.eiyve.cn/220606.Doc
srd.eiyve.cn/664000.Doc
srd.eiyve.cn/602680.Doc
srd.eiyve.cn/486866.Doc
srs.eiyve.cn/080022.Doc
srs.eiyve.cn/408082.Doc
srs.eiyve.cn/000824.Doc
srs.eiyve.cn/248882.Doc
srs.eiyve.cn/204684.Doc
srs.eiyve.cn/808062.Doc
srs.eiyve.cn/848026.Doc
srs.eiyve.cn/426828.Doc
srs.eiyve.cn/684240.Doc
srs.eiyve.cn/068280.Doc
sra.eiyve.cn/864002.Doc
sra.eiyve.cn/628486.Doc
sra.eiyve.cn/860800.Doc
sra.eiyve.cn/626840.Doc
sra.eiyve.cn/820868.Doc
sra.eiyve.cn/975373.Doc
sra.eiyve.cn/084226.Doc
sra.eiyve.cn/402844.Doc
sra.eiyve.cn/313399.Doc
sra.eiyve.cn/200880.Doc
srp.eiyve.cn/268082.Doc
srp.eiyve.cn/559977.Doc
srp.eiyve.cn/284808.Doc
srp.eiyve.cn/182113.Doc
srp.eiyve.cn/208408.Doc
srp.eiyve.cn/420446.Doc
srp.eiyve.cn/048402.Doc
srp.eiyve.cn/484006.Doc
srp.eiyve.cn/682686.Doc
srp.eiyve.cn/135199.Doc
sro.eiyve.cn/042480.Doc
sro.eiyve.cn/951779.Doc
sro.eiyve.cn/806040.Doc
sro.eiyve.cn/622264.Doc
sro.eiyve.cn/260680.Doc
sro.eiyve.cn/535731.Doc
sro.eiyve.cn/826664.Doc
sro.eiyve.cn/860040.Doc
sro.eiyve.cn/024882.Doc
sro.eiyve.cn/842642.Doc
sri.eiyve.cn/664246.Doc
sri.eiyve.cn/040800.Doc
sri.eiyve.cn/624400.Doc
sri.eiyve.cn/339311.Doc
sri.eiyve.cn/268246.Doc
sri.eiyve.cn/862208.Doc
sri.eiyve.cn/866024.Doc
sri.eiyve.cn/286682.Doc
sri.eiyve.cn/044024.Doc
sri.eiyve.cn/084088.Doc
sru.eiyve.cn/460624.Doc
sru.eiyve.cn/820824.Doc
sru.eiyve.cn/644600.Doc
sru.eiyve.cn/977595.Doc
sru.eiyve.cn/660628.Doc
sru.eiyve.cn/800026.Doc
sru.eiyve.cn/028888.Doc
sru.eiyve.cn/006246.Doc
sru.eiyve.cn/939953.Doc
sru.eiyve.cn/802808.Doc
sry.eiyve.cn/202426.Doc
sry.eiyve.cn/200882.Doc
sry.eiyve.cn/608886.Doc
sry.eiyve.cn/282088.Doc
sry.eiyve.cn/797713.Doc
sry.eiyve.cn/808644.Doc
sry.eiyve.cn/666006.Doc
sry.eiyve.cn/260202.Doc
sry.eiyve.cn/886842.Doc
sry.eiyve.cn/428020.Doc
srt.eiyve.cn/379575.Doc
srt.eiyve.cn/264040.Doc
srt.eiyve.cn/882822.Doc
srt.eiyve.cn/800066.Doc
srt.eiyve.cn/311737.Doc
srt.eiyve.cn/208442.Doc
srt.eiyve.cn/808860.Doc
srt.eiyve.cn/462822.Doc
srt.eiyve.cn/406002.Doc
srt.eiyve.cn/888682.Doc
srr.eiyve.cn/862820.Doc
srr.eiyve.cn/246862.Doc
srr.eiyve.cn/064220.Doc
srr.eiyve.cn/608644.Doc
srr.eiyve.cn/406826.Doc
srr.eiyve.cn/600088.Doc
srr.eiyve.cn/840826.Doc
srr.eiyve.cn/462246.Doc
srr.eiyve.cn/086808.Doc
srr.eiyve.cn/060648.Doc
sre.eiyve.cn/242664.Doc
sre.eiyve.cn/646622.Doc
sre.eiyve.cn/822600.Doc
sre.eiyve.cn/868668.Doc
sre.eiyve.cn/848202.Doc
sre.eiyve.cn/002468.Doc
sre.eiyve.cn/442484.Doc
sre.eiyve.cn/446600.Doc
sre.eiyve.cn/408288.Doc
sre.eiyve.cn/408864.Doc
srw.eiyve.cn/684806.Doc
srw.eiyve.cn/240006.Doc
srw.eiyve.cn/662088.Doc
srw.eiyve.cn/204806.Doc
srw.eiyve.cn/666288.Doc
srw.eiyve.cn/040480.Doc
srw.eiyve.cn/406422.Doc
srw.eiyve.cn/420068.Doc
srw.eiyve.cn/517735.Doc
srw.eiyve.cn/220628.Doc
srq.eiyve.cn/682420.Doc
srq.eiyve.cn/044006.Doc
srq.eiyve.cn/260404.Doc
srq.eiyve.cn/751733.Doc
srq.eiyve.cn/571373.Doc
srq.eiyve.cn/844020.Doc
srq.eiyve.cn/668048.Doc
srq.eiyve.cn/460248.Doc
srq.eiyve.cn/042242.Doc
srq.eiyve.cn/486824.Doc
sem.eiyve.cn/622624.Doc
sem.eiyve.cn/200206.Doc
sem.eiyve.cn/808684.Doc
sem.eiyve.cn/400286.Doc
sem.eiyve.cn/242026.Doc
sem.eiyve.cn/888660.Doc
sem.eiyve.cn/931337.Doc
sem.eiyve.cn/915577.Doc
sem.eiyve.cn/802622.Doc
sem.eiyve.cn/664220.Doc
sen.eiyve.cn/664040.Doc
sen.eiyve.cn/844400.Doc
sen.eiyve.cn/062042.Doc
sen.eiyve.cn/002602.Doc
sen.eiyve.cn/086482.Doc
sen.eiyve.cn/862862.Doc
sen.eiyve.cn/424624.Doc
sen.eiyve.cn/824280.Doc
sen.eiyve.cn/808482.Doc
sen.eiyve.cn/462006.Doc
seb.eiyve.cn/773731.Doc
seb.eiyve.cn/802222.Doc
seb.eiyve.cn/466024.Doc
seb.eiyve.cn/648668.Doc
seb.eiyve.cn/440646.Doc
seb.eiyve.cn/646628.Doc
seb.eiyve.cn/595311.Doc
seb.eiyve.cn/200244.Doc
seb.eiyve.cn/882864.Doc
seb.eiyve.cn/628084.Doc
sev.eiyve.cn/262602.Doc
sev.eiyve.cn/424820.Doc
sev.eiyve.cn/880202.Doc
sev.eiyve.cn/026288.Doc
sev.eiyve.cn/682248.Doc
sev.eiyve.cn/486826.Doc
sev.eiyve.cn/044020.Doc
sev.eiyve.cn/468446.Doc
sev.eiyve.cn/662466.Doc
sev.eiyve.cn/000468.Doc
sec.eiyve.cn/022462.Doc
sec.eiyve.cn/860402.Doc
sec.eiyve.cn/828280.Doc
sec.eiyve.cn/391511.Doc
sec.eiyve.cn/688628.Doc
sec.eiyve.cn/066408.Doc
sec.eiyve.cn/086886.Doc
sec.eiyve.cn/022808.Doc
sec.eiyve.cn/668422.Doc
sec.eiyve.cn/024084.Doc
sex.eiyve.cn/800026.Doc
sex.eiyve.cn/462402.Doc
sex.eiyve.cn/084420.Doc
sex.eiyve.cn/228248.Doc
sex.eiyve.cn/868062.Doc
sex.eiyve.cn/620600.Doc
sex.eiyve.cn/482466.Doc
sex.eiyve.cn/868044.Doc
sex.eiyve.cn/426246.Doc
sex.eiyve.cn/664026.Doc
sez.eiyve.cn/806406.Doc
sez.eiyve.cn/282682.Doc
sez.eiyve.cn/886226.Doc
sez.eiyve.cn/064864.Doc
sez.eiyve.cn/402884.Doc
sez.eiyve.cn/620864.Doc
sez.eiyve.cn/848244.Doc
sez.eiyve.cn/202466.Doc
sez.eiyve.cn/280622.Doc
sez.eiyve.cn/666600.Doc
sel.eiyve.cn/179957.Doc
sel.eiyve.cn/248068.Doc
sel.eiyve.cn/004844.Doc
sel.eiyve.cn/864660.Doc
sel.eiyve.cn/282820.Doc
sel.eiyve.cn/597599.Doc
sel.eiyve.cn/028080.Doc
sel.eiyve.cn/866442.Doc
sel.eiyve.cn/284068.Doc
sel.eiyve.cn/866220.Doc
sek.eiyve.cn/228020.Doc
sek.eiyve.cn/004864.Doc
sek.eiyve.cn/220842.Doc
sek.eiyve.cn/222608.Doc
sek.eiyve.cn/682006.Doc
sek.eiyve.cn/062286.Doc
sek.eiyve.cn/686282.Doc
sek.eiyve.cn/420826.Doc
sek.eiyve.cn/044842.Doc
sek.eiyve.cn/828686.Doc
sej.eiyve.cn/323682.Doc
sej.eiyve.cn/244770.Doc
sej.eiyve.cn/728181.Doc
sej.eiyve.cn/342106.Doc
sej.eiyve.cn/852481.Doc
sej.eiyve.cn/663630.Doc
sej.eiyve.cn/182156.Doc
