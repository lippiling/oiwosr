士肆商绞菊


Linux timekeeping_get_ns与ktime_get实时时钟源读取

timekeeping是Linux时间子系统的核心层，负责维护系统时间（CLOCK_REALTIME、CLOCK_MONOTONIC等）。timekeeping_get_ns是timekeeper内部的核心函数，用于从当前选定的clocksource读取纳秒级时间；而ktime_get是面向内核其他模块的通用接口，返回ktime_t类型的时间戳。

timekeeping_get_ns的实现在kernel/time/timekeeping.c中，其核心逻辑是读取clocksource的cycle计数值，结合mult/shift因子和base时间计算出当前纳秒值：

static inline u64 timekeeping_get_ns(const struct tk_read_base *tkr)
{
    u64 delta, now;
    u64 cycles;

    now = tk_clock_read(tkr);
    cycles = now - tkr->cycle_last;
    delta = clocksource_delta(cycles, 0, tkr->mask);
    delta = mul_u64_u32_shr(delta, tkr->mult, tkr->shift);
    return delta;
}

其中tk_clock_read直接调用clocksource的read回调：

static inline u64 tk_clock_read(const struct tk_read_base *tkr)
{
    return tkr->clock->read(tkr->clock);
}

cycle_last记录上一次更新的cycle值，两者之差即为当前clocksource走过的周期数。clocksource_delta处理mask回绕（wrap-around），确保差值在有效范围内。mul_u64_u32_shr将周期数乘以mult再右移shift，转换为纳秒。此处的mult和shift是由clocksource的频率和NTP调整共同确定的。

timekeeping_get_ns用于timekeeping_forward_now函数，该函数在每次需要更新timekeeper状态时被调用：

static void timekeeping_forward_now(struct timekeeper *tk)
{
    u64 delta;
    u64 nsec;

    delta = clocksource_delta(tk_clock_read(&tk->tkr_mono),
                              tk->tkr_mono.cycle_last,
                              tk->tkr_mono.mask);
    tk->tkr_mono.cycle_last += delta;
    nsec = delta * tk->tkr_mono.mult;
    nsec >>= tk->tkr_mono.shift;
    tk->tkr_mono.xtime_nsec += nsec;
    tk->tkr_mono.base += nsec;
}

ktime_get是暴露给内核其他子系统的标准接口，其实现同样基于tk_read_base：

ktime_t ktime_get(void)
{
    struct timekeeper *tk = &tk_core.timekeeper;
    unsigned int seq;
    ktime_t base;
    u64 nsecs;

    do {
        seq = read_seqcount_begin(&tk_core.seq);
        base = tk->tkr_mono.base;
        nsecs = timekeeping_get_ns(&tk->tkr_mono);
    } while (read_seqcount_retry(&tk_core.seq, seq));

    return ktime_add_ns(base, nsecs);
}

此处使用了seqcount读写顺序锁，确保在读取过程中timekeeper没有被并发更新。base保存了上一轮的累计时间，timekeeping_get_ns计算出当前距离base的偏移量。二者相加即得到当前的CLOCK_MONOTONIC时间。

对于CLOCK_REALTIME，对应的接口是ktime_get_real：

ktime_t ktime_get_real(void)
{
    struct timekeeper *tk = &tk_core.timekeeper;
    unsigned int seq;
    ktime_t base_real;
    u64 nsecs;

    do {
        seq = read_seqcount_begin(&tk_core.seq);
        base_real = tk->tkr_real.base;
        nsecs = timekeeping_get_ns(&tk->tkr_real);
    } while (read_seqcount_retry(&tk_core.seq, seq));

    return ktime_add_ns(base_real, nsecs);
}

ktime_get_ts64则是填充struct timespec64结构体的版本：

void ktime_get_ts64(struct timespec64 *ts)
{
    struct timekeeper *tk = &tk_core.timekeeper;
    unsigned int seq;
    u64 nsecs;

    WARN_ON(timekeeping_suspended);
    do {
        seq = read_seqcount_begin(&tk_core.seq);
        ts->tv_sec = tk->xtime_sec;
        nsecs = timekeeping_get_ns(&tk->tkr_mono);
    } while (read_seqcount_retry(&tk_core.seq, seq));

    ts->tv_nsec = nsecs;
    timespec64_add_ns(ts, tk->tkr_mono.xtime_nsec);
}

ktime_get_raw是绕过NTP调整的原始硬件时间读取，直接使用clocksource的raw计数器而未经NTP频率校正：

ktime_t ktime_get_raw(void)
{
    struct timekeeper *tk = &tk_core.timekeeper;
    unsigned int seq;
    ktime_t base_raw;
    u64 nsecs;

    do {
        seq = read_seqcount_begin(&tk_core.seq);
        base_raw = tk->tkr_raw.base;
        nsecs = timekeeping_get_ns(&tk->tkr_raw);
    } while (read_seqcount_retry(&tk_core.seq, seq));

    return ktime_add_ns(base_raw, nsecs);
}

在多核场景下，ktime_get的seqcount锁竞争可能成为性能瓶颈。为此内核提供了ktime_get_coarse家族接口，它们不读取clocksource硬件，直接返回timekeeper中缓存的粗粒度时间，开销极低：

ktime_t ktime_get_coarse(void)
{
    struct timekeeper *tk = &tk_core.timekeeper;
    unsigned int seq;
    ktime_t base;
    u64 nsecs;

    do {
        seq = read_seqcount_begin(&tk_core.seq);
        base = tk->tkr_mono.base;
        nsecs = tk->tkr_mono.xtime_nsec >> tk->tkr_mono.shift;
    } while (read_seqcount_retry(&tk_core.seq, seq));

    return ktime_add_ns(base, nsecs);
}

ktime_get_real_fast_ns是面向性能关键路径的变体，使用64位原子读取代顺序锁，减少cache line bouncing。它要求clocksource的read操作是nonsequential的且不会睡眠。

timekeeping_get_ns与ktime_get之间的调用关系贯穿整个时间子系统。每次时钟中断处理tick_handle_periodic最终调用update_wall_time，后者更新timekeeper中的cycle_last和base等字段。时钟中断之外的时间读取请求全部由ktime_get族函数通过timekeeping_get_ns实时计算得出，实现了连续的时间视图。


懦掠跋毡鼐乃宦扰椿侥是咸倚徒蟹

m.efk.mglwx.cn/20622.Doc
m.efk.mglwx.cn/24808.Doc
m.efk.mglwx.cn/42622.Doc
m.efk.mglwx.cn/79711.Doc
m.efk.mglwx.cn/73719.Doc
m.efk.mglwx.cn/91511.Doc
m.efk.mglwx.cn/99995.Doc
m.efk.mglwx.cn/24020.Doc
m.efk.mglwx.cn/11937.Doc
m.efk.mglwx.cn/37113.Doc
m.efk.mglwx.cn/59315.Doc
m.efk.mglwx.cn/53395.Doc
m.efk.mglwx.cn/31555.Doc
m.efk.mglwx.cn/73551.Doc
m.efk.mglwx.cn/84246.Doc
m.efk.mglwx.cn/39311.Doc
m.efk.mglwx.cn/95791.Doc
m.efk.mglwx.cn/51117.Doc
m.efk.mglwx.cn/37117.Doc
m.efk.mglwx.cn/55571.Doc
m.efj.mglwx.cn/51359.Doc
m.efj.mglwx.cn/37535.Doc
m.efj.mglwx.cn/02026.Doc
m.efj.mglwx.cn/99593.Doc
m.efj.mglwx.cn/97915.Doc
m.efj.mglwx.cn/08604.Doc
m.efj.mglwx.cn/75555.Doc
m.efj.mglwx.cn/51571.Doc
m.efj.mglwx.cn/02082.Doc
m.efj.mglwx.cn/48882.Doc
m.efj.mglwx.cn/04400.Doc
m.efj.mglwx.cn/22880.Doc
m.efj.mglwx.cn/28242.Doc
m.efj.mglwx.cn/66480.Doc
m.efj.mglwx.cn/66624.Doc
m.efj.mglwx.cn/79735.Doc
m.efj.mglwx.cn/51757.Doc
m.efj.mglwx.cn/73359.Doc
m.efj.mglwx.cn/57753.Doc
m.efj.mglwx.cn/17339.Doc
m.efh.mglwx.cn/37591.Doc
m.efh.mglwx.cn/15513.Doc
m.efh.mglwx.cn/31195.Doc
m.efh.mglwx.cn/37537.Doc
m.efh.mglwx.cn/13199.Doc
m.efh.mglwx.cn/71791.Doc
m.efh.mglwx.cn/11717.Doc
m.efh.mglwx.cn/79733.Doc
m.efh.mglwx.cn/97779.Doc
m.efh.mglwx.cn/02408.Doc
m.efh.mglwx.cn/57115.Doc
m.efh.mglwx.cn/26082.Doc
m.efh.mglwx.cn/95715.Doc
m.efh.mglwx.cn/04226.Doc
m.efh.mglwx.cn/39173.Doc
m.efh.mglwx.cn/39939.Doc
m.efh.mglwx.cn/73371.Doc
m.efh.mglwx.cn/26080.Doc
m.efh.mglwx.cn/13177.Doc
m.efh.mglwx.cn/17571.Doc
m.efg.mglwx.cn/26422.Doc
m.efg.mglwx.cn/88246.Doc
m.efg.mglwx.cn/91931.Doc
m.efg.mglwx.cn/77975.Doc
m.efg.mglwx.cn/15317.Doc
m.efg.mglwx.cn/51391.Doc
m.efg.mglwx.cn/73315.Doc
m.efg.mglwx.cn/11379.Doc
m.efg.mglwx.cn/26800.Doc
m.efg.mglwx.cn/71975.Doc
m.efg.mglwx.cn/13157.Doc
m.efg.mglwx.cn/17113.Doc
m.efg.mglwx.cn/84684.Doc
m.efg.mglwx.cn/80424.Doc
m.efg.mglwx.cn/53593.Doc
m.efg.mglwx.cn/71317.Doc
m.efg.mglwx.cn/66242.Doc
m.efg.mglwx.cn/53159.Doc
m.efg.mglwx.cn/75177.Doc
m.efg.mglwx.cn/51597.Doc
m.eff.mglwx.cn/77593.Doc
m.eff.mglwx.cn/93919.Doc
m.eff.mglwx.cn/26842.Doc
m.eff.mglwx.cn/64624.Doc
m.eff.mglwx.cn/04668.Doc
m.eff.mglwx.cn/13713.Doc
m.eff.mglwx.cn/55915.Doc
m.eff.mglwx.cn/75517.Doc
m.eff.mglwx.cn/26622.Doc
m.eff.mglwx.cn/91757.Doc
m.eff.mglwx.cn/97979.Doc
m.eff.mglwx.cn/44644.Doc
m.eff.mglwx.cn/88040.Doc
m.eff.mglwx.cn/40806.Doc
m.eff.mglwx.cn/22664.Doc
m.eff.mglwx.cn/71197.Doc
m.eff.mglwx.cn/99113.Doc
m.eff.mglwx.cn/15979.Doc
m.eff.mglwx.cn/00646.Doc
m.eff.mglwx.cn/02486.Doc
m.efd.mglwx.cn/84800.Doc
m.efd.mglwx.cn/02468.Doc
m.efd.mglwx.cn/77333.Doc
m.efd.mglwx.cn/57571.Doc
m.efd.mglwx.cn/31759.Doc
m.efd.mglwx.cn/86002.Doc
m.efd.mglwx.cn/75353.Doc
m.efd.mglwx.cn/08222.Doc
m.efd.mglwx.cn/53199.Doc
m.efd.mglwx.cn/97315.Doc
m.efd.mglwx.cn/75779.Doc
m.efd.mglwx.cn/64022.Doc
m.efd.mglwx.cn/22088.Doc
m.efd.mglwx.cn/22288.Doc
m.efd.mglwx.cn/59137.Doc
m.efd.mglwx.cn/55517.Doc
m.efd.mglwx.cn/35753.Doc
m.efd.mglwx.cn/53711.Doc
m.efd.mglwx.cn/91113.Doc
m.efd.mglwx.cn/79355.Doc
m.efs.mglwx.cn/95973.Doc
m.efs.mglwx.cn/11977.Doc
m.efs.mglwx.cn/35375.Doc
m.efs.mglwx.cn/79793.Doc
m.efs.mglwx.cn/39937.Doc
m.efs.mglwx.cn/42042.Doc
m.efs.mglwx.cn/53775.Doc
m.efs.mglwx.cn/04828.Doc
m.efs.mglwx.cn/06600.Doc
m.efs.mglwx.cn/28444.Doc
m.efs.mglwx.cn/93933.Doc
m.efs.mglwx.cn/15739.Doc
m.efs.mglwx.cn/17511.Doc
m.efs.mglwx.cn/19593.Doc
m.efs.mglwx.cn/97111.Doc
m.efs.mglwx.cn/53955.Doc
m.efs.mglwx.cn/57793.Doc
m.efs.mglwx.cn/80864.Doc
m.efs.mglwx.cn/02640.Doc
m.efs.mglwx.cn/17919.Doc
m.efa.mglwx.cn/95737.Doc
m.efa.mglwx.cn/93517.Doc
m.efa.mglwx.cn/84226.Doc
m.efa.mglwx.cn/95333.Doc
m.efa.mglwx.cn/99199.Doc
m.efa.mglwx.cn/37971.Doc
m.efa.mglwx.cn/57533.Doc
m.efa.mglwx.cn/37597.Doc
m.efa.mglwx.cn/37599.Doc
m.efa.mglwx.cn/00486.Doc
m.efa.mglwx.cn/88408.Doc
m.efa.mglwx.cn/97935.Doc
m.efa.mglwx.cn/39995.Doc
m.efa.mglwx.cn/62042.Doc
m.efa.mglwx.cn/55131.Doc
m.efa.mglwx.cn/80228.Doc
m.efa.mglwx.cn/02846.Doc
m.efa.mglwx.cn/55719.Doc
m.efa.mglwx.cn/95731.Doc
m.efa.mglwx.cn/37571.Doc
m.efp.mglwx.cn/57353.Doc
m.efp.mglwx.cn/59793.Doc
m.efp.mglwx.cn/35151.Doc
m.efp.mglwx.cn/97395.Doc
m.efp.mglwx.cn/59773.Doc
m.efp.mglwx.cn/53131.Doc
m.efp.mglwx.cn/80440.Doc
m.efp.mglwx.cn/73179.Doc
m.efp.mglwx.cn/19331.Doc
m.efp.mglwx.cn/31139.Doc
m.efp.mglwx.cn/35313.Doc
m.efp.mglwx.cn/24800.Doc
m.efp.mglwx.cn/15517.Doc
m.efp.mglwx.cn/02644.Doc
m.efp.mglwx.cn/57391.Doc
m.efp.mglwx.cn/95593.Doc
m.efp.mglwx.cn/24208.Doc
m.efp.mglwx.cn/57775.Doc
m.efp.mglwx.cn/24460.Doc
m.efp.mglwx.cn/93935.Doc
m.efo.mglwx.cn/02248.Doc
m.efo.mglwx.cn/19371.Doc
m.efo.mglwx.cn/60080.Doc
m.efo.mglwx.cn/82622.Doc
m.efo.mglwx.cn/46200.Doc
m.efo.mglwx.cn/5.Doc
m.efo.mglwx.cn/46006.Doc
m.efo.mglwx.cn/31391.Doc
m.efo.mglwx.cn/48204.Doc
m.efo.mglwx.cn/77751.Doc
m.efo.mglwx.cn/19977.Doc
m.efo.mglwx.cn/64220.Doc
m.efo.mglwx.cn/20020.Doc
m.efo.mglwx.cn/97599.Doc
m.efo.mglwx.cn/04220.Doc
m.efo.mglwx.cn/20282.Doc
m.efo.mglwx.cn/64046.Doc
m.efo.mglwx.cn/00422.Doc
m.efo.mglwx.cn/31337.Doc
m.efo.mglwx.cn/11799.Doc
m.efi.mglwx.cn/33311.Doc
m.efi.mglwx.cn/39571.Doc
m.efi.mglwx.cn/95357.Doc
m.efi.mglwx.cn/77793.Doc
m.efi.mglwx.cn/00208.Doc
m.efi.mglwx.cn/51771.Doc
m.efi.mglwx.cn/11193.Doc
m.efi.mglwx.cn/35755.Doc
m.efi.mglwx.cn/15197.Doc
m.efi.mglwx.cn/66248.Doc
m.efi.mglwx.cn/91151.Doc
m.efi.mglwx.cn/15757.Doc
m.efi.mglwx.cn/73713.Doc
m.efi.mglwx.cn/57337.Doc
m.efi.mglwx.cn/02602.Doc
m.efi.mglwx.cn/66046.Doc
m.efi.mglwx.cn/64222.Doc
m.efi.mglwx.cn/04048.Doc
m.efi.mglwx.cn/66684.Doc
m.efi.mglwx.cn/51377.Doc
m.efu.mglwx.cn/93391.Doc
m.efu.mglwx.cn/44400.Doc
m.efu.mglwx.cn/62062.Doc
m.efu.mglwx.cn/26280.Doc
m.efu.mglwx.cn/62402.Doc
m.efu.mglwx.cn/24028.Doc
m.efu.mglwx.cn/60422.Doc
m.efu.mglwx.cn/97777.Doc
m.efu.mglwx.cn/86288.Doc
m.efu.mglwx.cn/75593.Doc
m.efu.mglwx.cn/79933.Doc
m.efu.mglwx.cn/99553.Doc
m.efu.mglwx.cn/19155.Doc
m.efu.mglwx.cn/53313.Doc
m.efu.mglwx.cn/51115.Doc
m.efu.mglwx.cn/33539.Doc
m.efu.mglwx.cn/55719.Doc
m.efu.mglwx.cn/24866.Doc
m.efu.mglwx.cn/46424.Doc
m.efu.mglwx.cn/39191.Doc
m.efy.mglwx.cn/99957.Doc
m.efy.mglwx.cn/08062.Doc
m.efy.mglwx.cn/44424.Doc
m.efy.mglwx.cn/86840.Doc
m.efy.mglwx.cn/19133.Doc
m.efy.mglwx.cn/08608.Doc
m.efy.mglwx.cn/48808.Doc
m.efy.mglwx.cn/42282.Doc
m.efy.mglwx.cn/06086.Doc
m.efy.mglwx.cn/22662.Doc
m.efy.mglwx.cn/28046.Doc
m.efy.mglwx.cn/75717.Doc
m.efy.mglwx.cn/68286.Doc
m.efy.mglwx.cn/04080.Doc
m.efy.mglwx.cn/66808.Doc
m.efy.mglwx.cn/11995.Doc
m.efy.mglwx.cn/73731.Doc
m.efy.mglwx.cn/06804.Doc
m.efy.mglwx.cn/88664.Doc
m.efy.mglwx.cn/35991.Doc
m.eft.mglwx.cn/20468.Doc
m.eft.mglwx.cn/39755.Doc
m.eft.mglwx.cn/55191.Doc
m.eft.mglwx.cn/99555.Doc
m.eft.mglwx.cn/51191.Doc
m.eft.mglwx.cn/22064.Doc
m.eft.mglwx.cn/00088.Doc
m.eft.mglwx.cn/51197.Doc
m.eft.mglwx.cn/75791.Doc
m.eft.mglwx.cn/99737.Doc
m.eft.mglwx.cn/40662.Doc
m.eft.mglwx.cn/75535.Doc
m.eft.mglwx.cn/57515.Doc
m.eft.mglwx.cn/37513.Doc
m.eft.mglwx.cn/97339.Doc
m.eft.mglwx.cn/13979.Doc
m.eft.mglwx.cn/35339.Doc
m.eft.mglwx.cn/06640.Doc
m.eft.mglwx.cn/80622.Doc
m.eft.mglwx.cn/11739.Doc
m.efr.mglwx.cn/71931.Doc
m.efr.mglwx.cn/68426.Doc
m.efr.mglwx.cn/84222.Doc
m.efr.mglwx.cn/17539.Doc
m.efr.mglwx.cn/39355.Doc
m.efr.mglwx.cn/02442.Doc
m.efr.mglwx.cn/02688.Doc
m.efr.mglwx.cn/95991.Doc
m.efr.mglwx.cn/19191.Doc
m.efr.mglwx.cn/91151.Doc
m.efr.mglwx.cn/95977.Doc
m.efr.mglwx.cn/88202.Doc
m.efr.mglwx.cn/42288.Doc
m.efr.mglwx.cn/04264.Doc
m.efr.mglwx.cn/57775.Doc
m.efr.mglwx.cn/59999.Doc
m.efr.mglwx.cn/15955.Doc
m.efr.mglwx.cn/44226.Doc
m.efr.mglwx.cn/51993.Doc
m.efr.mglwx.cn/37113.Doc
m.efe.mglwx.cn/79597.Doc
m.efe.mglwx.cn/80204.Doc
m.efe.mglwx.cn/22486.Doc
m.efe.mglwx.cn/99753.Doc
m.efe.mglwx.cn/26206.Doc
m.efe.mglwx.cn/37339.Doc
m.efe.mglwx.cn/33133.Doc
m.efe.mglwx.cn/17357.Doc
m.efe.mglwx.cn/00620.Doc
m.efe.mglwx.cn/02422.Doc
m.efe.mglwx.cn/37117.Doc
m.efe.mglwx.cn/80082.Doc
m.efe.mglwx.cn/28486.Doc
m.efe.mglwx.cn/17315.Doc
m.efe.mglwx.cn/75339.Doc
m.efe.mglwx.cn/39551.Doc
m.efe.mglwx.cn/91197.Doc
m.efe.mglwx.cn/46464.Doc
m.efe.mglwx.cn/13357.Doc
m.efe.mglwx.cn/79553.Doc
m.efw.mglwx.cn/79797.Doc
m.efw.mglwx.cn/88000.Doc
m.efw.mglwx.cn/04802.Doc
m.efw.mglwx.cn/57555.Doc
m.efw.mglwx.cn/51379.Doc
m.efw.mglwx.cn/80288.Doc
m.efw.mglwx.cn/40444.Doc
m.efw.mglwx.cn/73791.Doc
m.efw.mglwx.cn/39579.Doc
m.efw.mglwx.cn/20684.Doc
m.efw.mglwx.cn/75375.Doc
m.efw.mglwx.cn/17939.Doc
m.efw.mglwx.cn/84044.Doc
m.efw.mglwx.cn/19119.Doc
m.efw.mglwx.cn/35197.Doc
m.efw.mglwx.cn/08288.Doc
m.efw.mglwx.cn/99371.Doc
m.efw.mglwx.cn/22680.Doc
m.efw.mglwx.cn/80462.Doc
m.efw.mglwx.cn/22604.Doc
m.efq.mglwx.cn/15953.Doc
m.efq.mglwx.cn/59999.Doc
m.efq.mglwx.cn/95375.Doc
m.efq.mglwx.cn/75739.Doc
m.efq.mglwx.cn/97711.Doc
m.efq.mglwx.cn/91575.Doc
m.efq.mglwx.cn/91359.Doc
m.efq.mglwx.cn/71153.Doc
m.efq.mglwx.cn/40228.Doc
m.efq.mglwx.cn/66606.Doc
m.efq.mglwx.cn/33795.Doc
m.efq.mglwx.cn/11179.Doc
m.efq.mglwx.cn/02606.Doc
m.efq.mglwx.cn/55311.Doc
m.efq.mglwx.cn/73375.Doc
m.efq.mglwx.cn/57113.Doc
m.efq.mglwx.cn/97153.Doc
m.efq.mglwx.cn/53559.Doc
m.efq.mglwx.cn/91135.Doc
m.efq.mglwx.cn/62684.Doc
m.edm.mglwx.cn/51115.Doc
m.edm.mglwx.cn/71131.Doc
m.edm.mglwx.cn/35597.Doc
m.edm.mglwx.cn/24046.Doc
m.edm.mglwx.cn/02662.Doc
m.edm.mglwx.cn/08008.Doc
m.edm.mglwx.cn/24040.Doc
m.edm.mglwx.cn/93397.Doc
m.edm.mglwx.cn/11799.Doc
m.edm.mglwx.cn/99575.Doc
m.edm.mglwx.cn/77571.Doc
m.edm.mglwx.cn/73915.Doc
m.edm.mglwx.cn/64860.Doc
m.edm.mglwx.cn/06488.Doc
m.edm.mglwx.cn/93777.Doc
m.edm.mglwx.cn/97551.Doc
m.edm.mglwx.cn/73551.Doc
m.edm.mglwx.cn/55735.Doc
m.edm.mglwx.cn/17911.Doc
m.edm.mglwx.cn/57191.Doc
m.edn.mglwx.cn/35553.Doc
m.edn.mglwx.cn/66020.Doc
m.edn.mglwx.cn/51975.Doc
m.edn.mglwx.cn/53593.Doc
m.edn.mglwx.cn/13977.Doc
m.edn.mglwx.cn/86680.Doc
m.edn.mglwx.cn/82462.Doc
m.edn.mglwx.cn/88406.Doc
m.edn.mglwx.cn/71331.Doc
m.edn.mglwx.cn/00400.Doc
m.edn.mglwx.cn/17915.Doc
m.edn.mglwx.cn/71931.Doc
m.edn.mglwx.cn/73175.Doc
m.edn.mglwx.cn/71739.Doc
m.edn.mglwx.cn/75137.Doc
