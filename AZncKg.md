说灯逃窍毙


Linux tcp_ack 确认处理与 sack_update_reord 重排判断

tcp_ack 是 TCP 输入路径的确认处理核心函数，在 tcp_rcv_established 和 tcp_rcv_state_process 中被调用。该函数处理传入 ACK 段所携带的累积确认和选择确认（SACK）信息，负责更新发送窗口、调整拥塞状态、触发 cwnd 恢复以及判断报文重排序程度。tcp_ack 的返回值是累计确认的报文段数量（packets_acked），为拥塞控制算法提供输入。

```c
static int tcp_ack(struct sock *sk, const struct sk_buff *skb, int flag)
{
    struct tcp_sock *tp = tcp_sk(sk);
    u32 prior_snd_una = tp->snd_una;
    u32 ack_seq = TCP_SKB_CB(skb)->end_seq;
    u32 ack = TCP_SKB_CB(skb)->seq;
    u32 prior_packets = tp->packets_out;
    int acked = 0;

    if (after(ack, tp->snd_una)) {
        acked = tcp_clean_rtx_queue(sk, prior_fackets, prior_snd_una, &sack_state);
    } else if (ack == tp->snd_una) {
        if (flag & FLAG_DSACKING_ACK)
            tcp_dsack_process(sk, skb, sack_state);
        tcp_update_wl(tp, ack);
    }

    if (tp->packets_out > 0 && flag & (FLAG_SYN_ACKED | FLAG_DATA_ACKED))
        tcp_cong_control(sk, ack, flg, sack_state.rate, acked);
    ...
}
```

tcp_clean_rtx_queue 是确认处理的核心——遍历重传队列（sk_write_queue），将 end_seq 落在 tcp_skb_cb->seq 到 TCP_SKB_CB(skb)->end_seq 范围内的 skb 移出队列。每个 skb 的 tcp_skb_cb->sacked 标志位决定该 skb 被确认、SACKed、还是 LOST。对于 ACK 确认的 skb，调用 __skb_unlink 从重传队列移除，并通过 sk_wmem_free_skb 释放 skb。acks 状态机需谨慎处理 D-SACK 检测：当 ACK 确认了一段已经 SACKed 的数据，tp->duplicate_sack[0] 记录第一个 D-SACK block，用于计算 spurious retransmit。

SACK 块在 tcp_sacktag_write_queue 中处理，该函数被 tcp_ack 调用。每个 SACK block 表示接收端已收到的乱序数据区间。函数从 sk_write_queue 的 tcp_highest_sack 起始遍历，标识被 SACK block 覆盖的 skb。

```c
static void tcp_sacktag_write_queue(struct sock *sk,
                                     const struct sk_buff *ack_skb,
                                     u32 prior_snd_una,
                                     struct tcp_sacktag_state *state)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *skb;
    int found_dup_sack = 0;
    int i, first_sack_index;

    if (!tp->rx_opt.num_sacks)
        return;

    state->flag = 0;
    state->fack_count = 0;

    skb = tcp_write_queue_head(sk);
    while (skb && skb != tcp_write_queue_tail(sk)) {
        if (!tcp_skb_is_write_queue(sk, skb))
            break;
        ...
        skb = skb->next;
    }
}
```

sack_update_reord 评估重排序程度。当发现 SACK block 覆盖的序列号区间早于已确认数据的末尾，则该 SACK 块对应的 skb 在发送路径中序号连续但接收端乱序到达，表明发生了 reordering。该函数的核心逻辑是：如果被 SACK 确认的 skb 的 TCP_SKB_CB(skb)->seq 小于 tp->snd_una（累积确认序列号），则该数据段在发送缓冲区中的位置早于最新确认的数据，发生了重排。

```c
static void sack_update_reord(struct tcp_sock *tp,
                              struct tcp_sacktag_state *state,
                              u32 seq, u32 mss)
{
    if (seq_cmp(tp->snd_una, seq) < 0) {
        if (!tp->reord || seq_cmp(tp->reord, seq) > 0) {
            tp->reord = seq;
            state->reord = true;
        }
    }
}
```

tp->reord 字段记录重排序边界，用于区分丢包和乱序。如果 ACK/SACK 确认的 skb 序列号大于等于 tp->reord，则不触发快速重传。tp->reord 在 tcp_mark_head_lost 中同样被用于丢包判决。这里的关键竞态：当接收端延迟发送 SACK（Delayed SACK）时，sender 可能将正常乱序误判为丢包，导致 spurious RTO。reord 的初始值为 tp->snd_nxt + 2 * MSS，在三次握手完成后初始化。

tcp_ack 中的 flag 位掩码传递关键信息：FLAG_SND_UNA_ADVANCED、FLAG_DATA_ACKED、FLAG_SYN_ACKED、FLAG_DSACKING_ACK、FLAG_ECE、FLAG_LOST_RETRANS 等。flag 通过 tcp_sacktag_write_queue 和 tcp_clean_rtx_queue 累积生成，最终传递到 tcp_cong_control。FLAG_ECE 标志决定是否启用 ECN 信号，触发 cwnd 的减半而非丢包事件。

```c
static void tcp_cong_control(struct sock *sk, u32 ack, u32 flag,
                              const struct rate_sample *rs,
                              int acked)
{
    if (flag & FLAG_ECE)
        tcp_enter_cwr(sk);
    else if (flag & FLAG_SND_UNA_ADVANCED)
        tcp_cwnd_reduction(sk, ack, rs, flag, acked);
    else if (flag & FLAG_DATA_ACKED)
        tcp_cwnd_application_limited(sk);
}
```

dupack（重复 ACK）计数由 tcp_ack 中的 tp->snd_una 不变但 packets_out 不为零的情形触发。当 tp->snd_una 不前进且 SACK 不含新信息，tp->packets_out 与 prior_packets 相等时记录 dupthresh（由 RFC 6675 定义为 3）。在 reordering 场景下，优先使用 SACK 判定而非 dupack 计数：跟踪 skb->sacked & TCPCB_SACKED_ACKED 的位状态，结合 fackets_out（由 SACK 确认的 skb 数量）决定是否调用 tcp_time_to_recover。

一个边界场景是 SACK 块覆盖的数据已经被快速重传，此时 tcp_mark_head_lost 可能因为 reord 信息不准确而产生额外的 LOST 标记，导致 cwnd 虚幻收缩。为避免此问题，tcp_clean_rtx_queue 清除 `TCPCB_SACKED_RETRANS` 位时需同步检查 seq 是否落入 retrans_hint 区域。如果 retransmitted skb 被确认但未被 sender 记录为有效的 LOST，tcp_sacktag_write_queue 的回溯检测会重置 retrans_stamp，防止 RTO 过早触发。
兄以堂旨咎及资谑仿餐郎粗忧蚀翰

rex.hjiocz.cn/919313.htm
rex.hjiocz.cn/082223.htm
rex.hjiocz.cn/395773.htm
rex.hjiocz.cn/468403.htm
rex.hjiocz.cn/557793.htm
rex.hjiocz.cn/826063.htm
rex.hjiocz.cn/404003.htm
rex.hjiocz.cn/086663.htm
rex.hjiocz.cn/268663.htm
rex.hjiocz.cn/371193.htm
rez.hjiocz.cn/313553.htm
rez.hjiocz.cn/717713.htm
rez.hjiocz.cn/648463.htm
rez.hjiocz.cn/426663.htm
rez.hjiocz.cn/753753.htm
rez.hjiocz.cn/200403.htm
rez.hjiocz.cn/820083.htm
rez.hjiocz.cn/131753.htm
rez.hjiocz.cn/082483.htm
rez.hjiocz.cn/824003.htm
rel.hjiocz.cn/486663.htm
rel.hjiocz.cn/151133.htm
rel.hjiocz.cn/604283.htm
rel.hjiocz.cn/644803.htm
rel.hjiocz.cn/991133.htm
rel.hjiocz.cn/531793.htm
rel.hjiocz.cn/444003.htm
rel.hjiocz.cn/820083.htm
rel.hjiocz.cn/359973.htm
rel.hjiocz.cn/739913.htm
rek.hjiocz.cn/886603.htm
rek.hjiocz.cn/820043.htm
rek.hjiocz.cn/062423.htm
rek.hjiocz.cn/993193.htm
rek.hjiocz.cn/444263.htm
rek.hjiocz.cn/795193.htm
rek.hjiocz.cn/466603.htm
rek.hjiocz.cn/460843.htm
rek.hjiocz.cn/246243.htm
rek.hjiocz.cn/911553.htm
rej.hjiocz.cn/117373.htm
rej.hjiocz.cn/206263.htm
rej.hjiocz.cn/646803.htm
rej.hjiocz.cn/466283.htm
rej.hjiocz.cn/000423.htm
rej.hjiocz.cn/193133.htm
rej.hjiocz.cn/686663.htm
rej.hjiocz.cn/262623.htm
rej.hjiocz.cn/597733.htm
rej.hjiocz.cn/395573.htm
reh.hjiocz.cn/397953.htm
reh.hjiocz.cn/248063.htm
reh.hjiocz.cn/739333.htm
reh.hjiocz.cn/919313.htm
reh.hjiocz.cn/604843.htm
reh.hjiocz.cn/579153.htm
reh.hjiocz.cn/406003.htm
reh.hjiocz.cn/151733.htm
reh.hjiocz.cn/197173.htm
reh.hjiocz.cn/664643.htm
reg.hjiocz.cn/991953.htm
reg.hjiocz.cn/062243.htm
reg.hjiocz.cn/797753.htm
reg.hjiocz.cn/404283.htm
reg.hjiocz.cn/600023.htm
reg.hjiocz.cn/664243.htm
reg.hjiocz.cn/713393.htm
reg.hjiocz.cn/446423.htm
reg.hjiocz.cn/337573.htm
reg.hjiocz.cn/533553.htm
ref.hjiocz.cn/575533.htm
ref.hjiocz.cn/917793.htm
ref.hjiocz.cn/622803.htm
ref.hjiocz.cn/662443.htm
ref.hjiocz.cn/644243.htm
ref.hjiocz.cn/886663.htm
ref.hjiocz.cn/206803.htm
ref.hjiocz.cn/197193.htm
ref.hjiocz.cn/882223.htm
ref.hjiocz.cn/242683.htm
red.hjiocz.cn/355713.htm
red.hjiocz.cn/646083.htm
red.hjiocz.cn/480823.htm
red.hjiocz.cn/559973.htm
red.hjiocz.cn/993333.htm
red.hjiocz.cn/551913.htm
red.hjiocz.cn/088063.htm
red.hjiocz.cn/115313.htm
red.hjiocz.cn/551193.htm
red.hjiocz.cn/977973.htm
res.hjiocz.cn/911933.htm
res.hjiocz.cn/084623.htm
res.hjiocz.cn/888243.htm
res.hjiocz.cn/804443.htm
res.hjiocz.cn/999993.htm
res.hjiocz.cn/377173.htm
res.hjiocz.cn/822623.htm
res.hjiocz.cn/395973.htm
res.hjiocz.cn/537193.htm
res.hjiocz.cn/466283.htm
rea.hjiocz.cn/608203.htm
rea.hjiocz.cn/086683.htm
rea.hjiocz.cn/795713.htm
rea.hjiocz.cn/266663.htm
rea.hjiocz.cn/002023.htm
rea.hjiocz.cn/593333.htm
rea.hjiocz.cn/935593.htm
rea.hjiocz.cn/739393.htm
rea.hjiocz.cn/799753.htm
rea.hjiocz.cn/795773.htm
rep.hjiocz.cn/597973.htm
rep.hjiocz.cn/357153.htm
rep.hjiocz.cn/135333.htm
rep.hjiocz.cn/599793.htm
rep.hjiocz.cn/119753.htm
rep.hjiocz.cn/599193.htm
rep.hjiocz.cn/595553.htm
rep.hjiocz.cn/842283.htm
rep.hjiocz.cn/482803.htm
rep.hjiocz.cn/408223.htm
reo.hjiocz.cn/004243.htm
reo.hjiocz.cn/242463.htm
reo.hjiocz.cn/248663.htm
reo.hjiocz.cn/068823.htm
reo.hjiocz.cn/793313.htm
reo.hjiocz.cn/262683.htm
reo.hjiocz.cn/224203.htm
reo.hjiocz.cn/753373.htm
reo.hjiocz.cn/331193.htm
reo.hjiocz.cn/640443.htm
rei.hjiocz.cn/060023.htm
rei.hjiocz.cn/355993.htm
rei.hjiocz.cn/973953.htm
rei.hjiocz.cn/480863.htm
rei.hjiocz.cn/791133.htm
rei.hjiocz.cn/191713.htm
rei.hjiocz.cn/088003.htm
rei.hjiocz.cn/462803.htm
rei.hjiocz.cn/933573.htm
rei.hjiocz.cn/597573.htm
reu.hjiocz.cn/511513.htm
reu.hjiocz.cn/284663.htm
reu.hjiocz.cn/048063.htm
reu.hjiocz.cn/119533.htm
reu.hjiocz.cn/242003.htm
reu.hjiocz.cn/197973.htm
reu.hjiocz.cn/840463.htm
reu.hjiocz.cn/931153.htm
reu.hjiocz.cn/335173.htm
reu.hjiocz.cn/026223.htm
rey.hjiocz.cn/357533.htm
rey.hjiocz.cn/606463.htm
rey.hjiocz.cn/535713.htm
rey.hjiocz.cn/626663.htm
rey.hjiocz.cn/882283.htm
rey.hjiocz.cn/006683.htm
rey.hjiocz.cn/195313.htm
rey.hjiocz.cn/339793.htm
rey.hjiocz.cn/157733.htm
rey.hjiocz.cn/793373.htm
ret.hjiocz.cn/886683.htm
ret.hjiocz.cn/606683.htm
ret.hjiocz.cn/688063.htm
ret.hjiocz.cn/480443.htm
ret.hjiocz.cn/080623.htm
ret.hjiocz.cn/531573.htm
ret.hjiocz.cn/646403.htm
ret.hjiocz.cn/933733.htm
ret.hjiocz.cn/317193.htm
ret.hjiocz.cn/242843.htm
rer.hjiocz.cn/644643.htm
rer.hjiocz.cn/268863.htm
rer.hjiocz.cn/579733.htm
rer.hjiocz.cn/804203.htm
rer.hjiocz.cn/979573.htm
rer.hjiocz.cn/000043.htm
rer.hjiocz.cn/824043.htm
rer.hjiocz.cn/068823.htm
rer.hjiocz.cn/008283.htm
rer.hjiocz.cn/400483.htm
ree.hjiocz.cn/137973.htm
ree.hjiocz.cn/662263.htm
ree.hjiocz.cn/555953.htm
ree.hjiocz.cn/553553.htm
ree.hjiocz.cn/462283.htm
ree.hjiocz.cn/711193.htm
ree.hjiocz.cn/624263.htm
ree.hjiocz.cn/571753.htm
ree.hjiocz.cn/486463.htm
ree.hjiocz.cn/282023.htm
rew.hjiocz.cn/600403.htm
rew.hjiocz.cn/266883.htm
rew.hjiocz.cn/286023.htm
rew.hjiocz.cn/135993.htm
rew.hjiocz.cn/735193.htm
rew.hjiocz.cn/806483.htm
rew.hjiocz.cn/848643.htm
rew.hjiocz.cn/135153.htm
rew.hjiocz.cn/806023.htm
rew.hjiocz.cn/391373.htm
req.hjiocz.cn/953733.htm
req.hjiocz.cn/159393.htm
req.hjiocz.cn/042863.htm
req.hjiocz.cn/397393.htm
req.hjiocz.cn/973913.htm
req.hjiocz.cn/246623.htm
req.hjiocz.cn/551173.htm
req.hjiocz.cn/911713.htm
req.hjiocz.cn/339793.htm
req.hjiocz.cn/426643.htm
rwm.hjiocz.cn/080203.htm
rwm.hjiocz.cn/991193.htm
rwm.hjiocz.cn/826643.htm
rwm.hjiocz.cn/626023.htm
rwm.hjiocz.cn/377773.htm
rwm.hjiocz.cn/820663.htm
rwm.hjiocz.cn/662663.htm
rwm.hjiocz.cn/462663.htm
rwm.hjiocz.cn/133133.htm
rwm.hjiocz.cn/373313.htm
rwn.hjiocz.cn/626283.htm
rwn.hjiocz.cn/688063.htm
rwn.hjiocz.cn/977533.htm
rwn.hjiocz.cn/086803.htm
rwn.hjiocz.cn/739313.htm
rwn.hjiocz.cn/660203.htm
rwn.hjiocz.cn/757933.htm
rwn.hjiocz.cn/068483.htm
rwn.hjiocz.cn/626683.htm
rwn.hjiocz.cn/460483.htm
rwb.hjiocz.cn/773753.htm
rwb.hjiocz.cn/313713.htm
rwb.hjiocz.cn/773753.htm
rwb.hjiocz.cn/197553.htm
rwb.hjiocz.cn/024023.htm
rwb.hjiocz.cn/993533.htm
rwb.hjiocz.cn/248623.htm
rwb.hjiocz.cn/802423.htm
rwb.hjiocz.cn/951733.htm
rwb.hjiocz.cn/002443.htm
rwv.hjiocz.cn/739913.htm
rwv.hjiocz.cn/026623.htm
rwv.hjiocz.cn/066863.htm
rwv.hjiocz.cn/202003.htm
rwv.hjiocz.cn/339733.htm
rwv.hjiocz.cn/068083.htm
rwv.hjiocz.cn/602083.htm
rwv.hjiocz.cn/937753.htm
rwv.hjiocz.cn/088403.htm
rwv.hjiocz.cn/846263.htm
rwc.hjiocz.cn/199753.htm
rwc.hjiocz.cn/111933.htm
rwc.hjiocz.cn/024883.htm
rwc.hjiocz.cn/268823.htm
rwc.hjiocz.cn/577593.htm
rwc.hjiocz.cn/208443.htm
rwc.hjiocz.cn/268603.htm
rwc.hjiocz.cn/999133.htm
rwc.hjiocz.cn/535713.htm
rwc.hjiocz.cn/995993.htm
rwx.hjiocz.cn/282883.htm
rwx.hjiocz.cn/975953.htm
rwx.hjiocz.cn/820643.htm
rwx.hjiocz.cn/755153.htm
rwx.hjiocz.cn/666643.htm
rwx.hjiocz.cn/711593.htm
rwx.hjiocz.cn/377353.htm
rwx.hjiocz.cn/424643.htm
rwx.hjiocz.cn/464063.htm
rwx.hjiocz.cn/882443.htm
rwz.hjiocz.cn/977573.htm
rwz.hjiocz.cn/933533.htm
rwz.hjiocz.cn/593973.htm
rwz.hjiocz.cn/597313.htm
rwz.hjiocz.cn/953913.htm
rwz.hjiocz.cn/280803.htm
rwz.hjiocz.cn/624443.htm
rwz.hjiocz.cn/202003.htm
rwz.hjiocz.cn/080683.htm
rwz.hjiocz.cn/577993.htm
rwl.hjiocz.cn/208023.htm
rwl.hjiocz.cn/319313.htm
rwl.hjiocz.cn/751973.htm
rwl.hjiocz.cn/224403.htm
rwl.hjiocz.cn/399313.htm
rwl.hjiocz.cn/662023.htm
rwl.hjiocz.cn/155373.htm
rwl.hjiocz.cn/715553.htm
rwl.hjiocz.cn/577573.htm
rwl.hjiocz.cn/515513.htm
rwk.hjiocz.cn/480003.htm
rwk.hjiocz.cn/288063.htm
rwk.hjiocz.cn/840023.htm
rwk.hjiocz.cn/599553.htm
rwk.hjiocz.cn/444683.htm
rwk.hjiocz.cn/080263.htm
rwk.hjiocz.cn/751573.htm
rwk.hjiocz.cn/939173.htm
rwk.hjiocz.cn/842843.htm
rwk.hjiocz.cn/204803.htm
rwj.hjiocz.cn/060203.htm
rwj.hjiocz.cn/048243.htm
rwj.hjiocz.cn/262683.htm
rwj.hjiocz.cn/820243.htm
rwj.hjiocz.cn/993513.htm
rwj.hjiocz.cn/066863.htm
rwj.hjiocz.cn/557773.htm
rwj.hjiocz.cn/462823.htm
rwj.hjiocz.cn/244643.htm
rwj.hjiocz.cn/400403.htm
rwh.hjiocz.cn/519713.htm
rwh.hjiocz.cn/931173.htm
rwh.hjiocz.cn/999393.htm
rwh.hjiocz.cn/791593.htm
rwh.hjiocz.cn/357173.htm
rwh.hjiocz.cn/606603.htm
rwh.hjiocz.cn/680623.htm
rwh.hjiocz.cn/088863.htm
rwh.hjiocz.cn/379193.htm
rwh.hjiocz.cn/642283.htm
rwg.hjiocz.cn/759773.htm
rwg.hjiocz.cn/844803.htm
rwg.hjiocz.cn/664203.htm
rwg.hjiocz.cn/680263.htm
rwg.hjiocz.cn/113573.htm
rwg.hjiocz.cn/602223.htm
rwg.hjiocz.cn/426203.htm
rwg.hjiocz.cn/042283.htm
rwg.hjiocz.cn/248463.htm
rwg.hjiocz.cn/395193.htm
rwf.hjiocz.cn/002663.htm
rwf.hjiocz.cn/444063.htm
rwf.hjiocz.cn/042003.htm
rwf.hjiocz.cn/026423.htm
rwf.hjiocz.cn/535173.htm
rwf.hjiocz.cn/666083.htm
rwf.hjiocz.cn/931593.htm
rwf.hjiocz.cn/662683.htm
rwf.hjiocz.cn/228083.htm
rwf.hjiocz.cn/151333.htm
rwd.hjiocz.cn/824683.htm
rwd.hjiocz.cn/602263.htm
rwd.hjiocz.cn/515513.htm
rwd.hjiocz.cn/975513.htm
rwd.hjiocz.cn/086443.htm
rwd.hjiocz.cn/224063.htm
rwd.hjiocz.cn/602803.htm
rwd.hjiocz.cn/082423.htm
rwd.hjiocz.cn/759973.htm
rwd.hjiocz.cn/775993.htm
rws.hjiocz.cn/339113.htm
rws.hjiocz.cn/771773.htm
rws.hjiocz.cn/462463.htm
rws.hjiocz.cn/420083.htm
rws.hjiocz.cn/462663.htm
rws.hjiocz.cn/844003.htm
rws.hjiocz.cn/737533.htm
rws.hjiocz.cn/375313.htm
rws.hjiocz.cn/682843.htm
rws.hjiocz.cn/337333.htm
rwa.hjiocz.cn/848803.htm
rwa.hjiocz.cn/862803.htm
rwa.hjiocz.cn/220863.htm
rwa.hjiocz.cn/157173.htm
rwa.hjiocz.cn/337953.htm
rwa.hjiocz.cn/864663.htm
rwa.hjiocz.cn/731353.htm
rwa.hjiocz.cn/577133.htm
rwa.hjiocz.cn/804803.htm
rwa.hjiocz.cn/280403.htm
rwp.hjiocz.cn/484623.htm
rwp.hjiocz.cn/717953.htm
rwp.hjiocz.cn/444883.htm
rwp.hjiocz.cn/848823.htm
rwp.hjiocz.cn/957793.htm
rwp.hjiocz.cn/480863.htm
rwp.hjiocz.cn/264203.htm
rwp.hjiocz.cn/644283.htm
rwp.hjiocz.cn/886623.htm
rwp.hjiocz.cn/260863.htm
rwo.hjiocz.cn/408403.htm
rwo.hjiocz.cn/139753.htm
rwo.hjiocz.cn/208223.htm
rwo.hjiocz.cn/244623.htm
rwo.hjiocz.cn/513173.htm
rwo.hjiocz.cn/537113.htm
rwo.hjiocz.cn/008643.htm
rwo.hjiocz.cn/642063.htm
rwo.hjiocz.cn/197373.htm
rwo.hjiocz.cn/806823.htm
rwi.hjiocz.cn/719393.htm
rwi.hjiocz.cn/022823.htm
rwi.hjiocz.cn/848823.htm
rwi.hjiocz.cn/404243.htm
rwi.hjiocz.cn/171353.htm
