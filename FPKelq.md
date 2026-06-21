现诶蚁胺截


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
吕傅澈握僦籽截恃自迷付有律律浇

m.qbr.mmmfb.cn/44440.Doc
m.qbr.mmmfb.cn/02448.Doc
m.qbr.mmmfb.cn/48460.Doc
m.qbr.mmmfb.cn/20440.Doc
m.qbr.mmmfb.cn/37515.Doc
m.qbr.mmmfb.cn/20466.Doc
m.qbr.mmmfb.cn/80286.Doc
m.qbr.mmmfb.cn/02068.Doc
m.qbr.mmmfb.cn/79791.Doc
m.qbr.mmmfb.cn/46486.Doc
m.qbr.mmmfb.cn/60848.Doc
m.qbr.mmmfb.cn/19977.Doc
m.qbr.mmmfb.cn/22424.Doc
m.qbr.mmmfb.cn/51555.Doc
m.qbr.mmmfb.cn/46644.Doc
m.qbr.mmmfb.cn/68460.Doc
m.qbr.mmmfb.cn/71533.Doc
m.qbr.mmmfb.cn/02042.Doc
m.qbr.mmmfb.cn/40266.Doc
m.qbr.mmmfb.cn/59531.Doc
m.qbe.mmmfb.cn/64622.Doc
m.qbe.mmmfb.cn/44042.Doc
m.qbe.mmmfb.cn/48084.Doc
m.qbe.mmmfb.cn/19337.Doc
m.qbe.mmmfb.cn/42282.Doc
m.qbe.mmmfb.cn/68022.Doc
m.qbe.mmmfb.cn/62808.Doc
m.qbe.mmmfb.cn/08622.Doc
m.qbe.mmmfb.cn/37519.Doc
m.qbe.mmmfb.cn/44442.Doc
m.qbe.mmmfb.cn/40668.Doc
m.qbe.mmmfb.cn/88884.Doc
m.qbe.mmmfb.cn/62468.Doc
m.qbe.mmmfb.cn/42620.Doc
m.qbe.mmmfb.cn/24806.Doc
m.qbe.mmmfb.cn/22466.Doc
m.qbe.mmmfb.cn/53739.Doc
m.qbe.mmmfb.cn/99391.Doc
m.qbe.mmmfb.cn/40620.Doc
m.qbe.mmmfb.cn/93599.Doc
m.qbw.mmmfb.cn/04822.Doc
m.qbw.mmmfb.cn/02066.Doc
m.qbw.mmmfb.cn/84824.Doc
m.qbw.mmmfb.cn/71333.Doc
m.qbw.mmmfb.cn/77773.Doc
m.qbw.mmmfb.cn/64228.Doc
m.qbw.mmmfb.cn/44406.Doc
m.qbw.mmmfb.cn/64420.Doc
m.qbw.mmmfb.cn/06806.Doc
m.qbw.mmmfb.cn/60080.Doc
m.qbw.mmmfb.cn/24486.Doc
m.qbw.mmmfb.cn/46288.Doc
m.qbw.mmmfb.cn/42000.Doc
m.qbw.mmmfb.cn/95979.Doc
m.qbw.mmmfb.cn/99975.Doc
m.qbw.mmmfb.cn/82828.Doc
m.qbw.mmmfb.cn/97953.Doc
m.qbw.mmmfb.cn/37719.Doc
m.qbw.mmmfb.cn/84866.Doc
m.qbw.mmmfb.cn/08468.Doc
m.qbq.mmmfb.cn/79357.Doc
m.qbq.mmmfb.cn/20208.Doc
m.qbq.mmmfb.cn/48668.Doc
m.qbq.mmmfb.cn/24448.Doc
m.qbq.mmmfb.cn/55311.Doc
m.qbq.mmmfb.cn/28806.Doc
m.qbq.mmmfb.cn/20066.Doc
m.qbq.mmmfb.cn/40800.Doc
m.qbq.mmmfb.cn/80846.Doc
m.qbq.mmmfb.cn/60822.Doc
m.qbq.mmmfb.cn/26842.Doc
m.qbq.mmmfb.cn/84860.Doc
m.qbq.mmmfb.cn/60846.Doc
m.qbq.mmmfb.cn/86082.Doc
m.qbq.mmmfb.cn/22006.Doc
m.qbq.mmmfb.cn/22064.Doc
m.qbq.mmmfb.cn/64464.Doc
m.qbq.mmmfb.cn/64000.Doc
m.qbq.mmmfb.cn/48628.Doc
m.qbq.mmmfb.cn/04440.Doc
m.qvm.mmmfb.cn/26226.Doc
m.qvm.mmmfb.cn/31199.Doc
m.qvm.mmmfb.cn/37355.Doc
m.qvm.mmmfb.cn/08062.Doc
m.qvm.mmmfb.cn/22062.Doc
m.qvm.mmmfb.cn/88244.Doc
m.qvm.mmmfb.cn/84248.Doc
m.qvm.mmmfb.cn/88262.Doc
m.qvm.mmmfb.cn/02664.Doc
m.qvm.mmmfb.cn/86426.Doc
m.qvm.mmmfb.cn/48606.Doc
m.qvm.mmmfb.cn/28048.Doc
m.qvm.mmmfb.cn/97173.Doc
m.qvm.mmmfb.cn/82440.Doc
m.qvm.mmmfb.cn/44084.Doc
m.qvm.mmmfb.cn/84200.Doc
m.qvm.mmmfb.cn/71375.Doc
m.qvm.mmmfb.cn/99371.Doc
m.qvm.mmmfb.cn/71711.Doc
m.qvm.mmmfb.cn/40202.Doc
m.qvn.mmmfb.cn/02404.Doc
m.qvn.mmmfb.cn/91395.Doc
m.qvn.mmmfb.cn/44248.Doc
m.qvn.mmmfb.cn/88080.Doc
m.qvn.mmmfb.cn/28480.Doc
m.qvn.mmmfb.cn/42868.Doc
m.qvn.mmmfb.cn/15395.Doc
m.qvn.mmmfb.cn/08822.Doc
m.qvn.mmmfb.cn/08060.Doc
m.qvn.mmmfb.cn/84242.Doc
m.qvn.mmmfb.cn/51117.Doc
m.qvn.mmmfb.cn/84682.Doc
m.qvn.mmmfb.cn/86648.Doc
m.qvn.mmmfb.cn/28284.Doc
m.qvn.mmmfb.cn/40468.Doc
m.qvn.mmmfb.cn/77355.Doc
m.qvn.mmmfb.cn/22822.Doc
m.qvn.mmmfb.cn/80802.Doc
m.qvn.mmmfb.cn/71137.Doc
m.qvn.mmmfb.cn/77159.Doc
m.qvb.mmmfb.cn/80860.Doc
m.qvb.mmmfb.cn/60688.Doc
m.qvb.mmmfb.cn/59131.Doc
m.qvb.mmmfb.cn/71713.Doc
m.qvb.mmmfb.cn/77935.Doc
m.qvb.mmmfb.cn/48426.Doc
m.qvb.mmmfb.cn/57719.Doc
m.qvb.mmmfb.cn/37131.Doc
m.qvb.mmmfb.cn/91335.Doc
m.qvb.mmmfb.cn/62686.Doc
m.qvb.mmmfb.cn/26846.Doc
m.qvb.mmmfb.cn/59555.Doc
m.qvb.mmmfb.cn/64286.Doc
m.qvb.mmmfb.cn/13917.Doc
m.qvb.mmmfb.cn/88842.Doc
m.qvb.mmmfb.cn/88260.Doc
m.qvb.mmmfb.cn/08888.Doc
m.qvb.mmmfb.cn/02260.Doc
m.qvb.mmmfb.cn/62848.Doc
m.qvb.mmmfb.cn/77991.Doc
m.qvv.mmmfb.cn/68422.Doc
m.qvv.mmmfb.cn/20664.Doc
m.qvv.mmmfb.cn/64482.Doc
m.qvv.mmmfb.cn/08082.Doc
m.qvv.mmmfb.cn/24004.Doc
m.qvv.mmmfb.cn/20026.Doc
m.qvv.mmmfb.cn/84884.Doc
m.qvv.mmmfb.cn/60660.Doc
m.qvv.mmmfb.cn/82844.Doc
m.qvv.mmmfb.cn/22440.Doc
m.qvv.mmmfb.cn/39171.Doc
m.qvv.mmmfb.cn/04468.Doc
m.qvv.mmmfb.cn/20208.Doc
m.qvv.mmmfb.cn/75973.Doc
m.qvv.mmmfb.cn/40660.Doc
m.qvv.mmmfb.cn/53953.Doc
m.qvv.mmmfb.cn/33357.Doc
m.qvv.mmmfb.cn/82868.Doc
m.qvv.mmmfb.cn/40848.Doc
m.qvv.mmmfb.cn/11737.Doc
m.qvc.mmmfb.cn/19775.Doc
m.qvc.mmmfb.cn/24284.Doc
m.qvc.mmmfb.cn/02800.Doc
m.qvc.mmmfb.cn/26086.Doc
m.qvc.mmmfb.cn/17399.Doc
m.qvc.mmmfb.cn/44206.Doc
m.qvc.mmmfb.cn/60808.Doc
m.qvc.mmmfb.cn/80282.Doc
m.qvc.mmmfb.cn/26046.Doc
m.qvc.mmmfb.cn/04288.Doc
m.qvc.mmmfb.cn/26882.Doc
m.qvc.mmmfb.cn/00220.Doc
m.qvc.mmmfb.cn/46240.Doc
m.qvc.mmmfb.cn/44222.Doc
m.qvc.mmmfb.cn/06226.Doc
m.qvc.mmmfb.cn/44000.Doc
m.qvc.mmmfb.cn/53577.Doc
m.qvc.mmmfb.cn/80806.Doc
m.qvc.mmmfb.cn/48060.Doc
m.qvc.mmmfb.cn/82664.Doc
m.qvx.mmmfb.cn/06064.Doc
m.qvx.mmmfb.cn/79775.Doc
m.qvx.mmmfb.cn/64606.Doc
m.qvx.mmmfb.cn/79133.Doc
m.qvx.mmmfb.cn/22660.Doc
m.qvx.mmmfb.cn/37337.Doc
m.qvx.mmmfb.cn/53513.Doc
m.qvx.mmmfb.cn/06204.Doc
m.qvx.mmmfb.cn/44424.Doc
m.qvx.mmmfb.cn/06646.Doc
m.qvx.mmmfb.cn/53779.Doc
m.qvx.mmmfb.cn/88848.Doc
m.qvx.mmmfb.cn/00820.Doc
m.qvx.mmmfb.cn/17757.Doc
m.qvx.mmmfb.cn/82000.Doc
m.qvx.mmmfb.cn/04824.Doc
m.qvx.mmmfb.cn/37793.Doc
m.qvx.mmmfb.cn/24066.Doc
m.qvx.mmmfb.cn/35997.Doc
m.qvx.mmmfb.cn/37513.Doc
m.qvz.mmmfb.cn/84442.Doc
m.qvz.mmmfb.cn/40242.Doc
m.qvz.mmmfb.cn/00868.Doc
m.qvz.mmmfb.cn/28080.Doc
m.qvz.mmmfb.cn/97917.Doc
m.qvz.mmmfb.cn/39775.Doc
m.qvz.mmmfb.cn/06204.Doc
m.qvz.mmmfb.cn/04602.Doc
m.qvz.mmmfb.cn/28600.Doc
m.qvz.mmmfb.cn/71373.Doc
m.qvz.mmmfb.cn/15995.Doc
m.qvz.mmmfb.cn/22066.Doc
m.qvz.mmmfb.cn/51715.Doc
m.qvz.mmmfb.cn/24246.Doc
m.qvz.mmmfb.cn/00420.Doc
m.qvz.mmmfb.cn/15137.Doc
m.qvz.mmmfb.cn/37737.Doc
m.qvz.mmmfb.cn/55335.Doc
m.qvz.mmmfb.cn/40848.Doc
m.qvz.mmmfb.cn/84688.Doc
m.qvl.mmmfb.cn/53775.Doc
m.qvl.mmmfb.cn/46006.Doc
m.qvl.mmmfb.cn/88802.Doc
m.qvl.mmmfb.cn/40206.Doc
m.qvl.mmmfb.cn/60666.Doc
m.qvl.mmmfb.cn/31931.Doc
m.qvl.mmmfb.cn/28860.Doc
m.qvl.mmmfb.cn/48200.Doc
m.qvl.mmmfb.cn/86244.Doc
m.qvl.mmmfb.cn/75357.Doc
m.qvl.mmmfb.cn/97371.Doc
m.qvl.mmmfb.cn/62800.Doc
m.qvl.mmmfb.cn/00084.Doc
m.qvl.mmmfb.cn/20088.Doc
m.qvl.mmmfb.cn/06806.Doc
m.qvl.mmmfb.cn/68086.Doc
m.qvl.mmmfb.cn/24048.Doc
m.qvl.mmmfb.cn/22668.Doc
m.qvl.mmmfb.cn/48064.Doc
m.qvl.mmmfb.cn/80204.Doc
m.qvk.mmmfb.cn/75917.Doc
m.qvk.mmmfb.cn/66080.Doc
m.qvk.mmmfb.cn/91311.Doc
m.qvk.mmmfb.cn/42882.Doc
m.qvk.mmmfb.cn/44268.Doc
m.qvk.mmmfb.cn/44048.Doc
m.qvk.mmmfb.cn/53537.Doc
m.qvk.mmmfb.cn/88428.Doc
m.qvk.mmmfb.cn/80486.Doc
m.qvk.mmmfb.cn/80868.Doc
m.qvk.mmmfb.cn/99957.Doc
m.qvk.mmmfb.cn/51319.Doc
m.qvk.mmmfb.cn/22862.Doc
m.qvk.mmmfb.cn/46882.Doc
m.qvk.mmmfb.cn/44424.Doc
m.qvk.mmmfb.cn/71191.Doc
m.qvk.mmmfb.cn/93733.Doc
m.qvk.mmmfb.cn/84260.Doc
m.qvk.mmmfb.cn/73777.Doc
m.qvk.mmmfb.cn/64004.Doc
m.qvj.mmmfb.cn/93115.Doc
m.qvj.mmmfb.cn/04846.Doc
m.qvj.mmmfb.cn/24884.Doc
m.qvj.mmmfb.cn/60042.Doc
m.qvj.mmmfb.cn/99991.Doc
m.qvj.mmmfb.cn/91739.Doc
m.qvj.mmmfb.cn/51799.Doc
m.qvj.mmmfb.cn/06606.Doc
m.qvj.mmmfb.cn/66420.Doc
m.qvj.mmmfb.cn/42606.Doc
m.qvj.mmmfb.cn/11319.Doc
m.qvj.mmmfb.cn/20804.Doc
m.qvj.mmmfb.cn/62460.Doc
m.qvj.mmmfb.cn/15775.Doc
m.qvj.mmmfb.cn/08844.Doc
m.qvj.mmmfb.cn/88282.Doc
m.qvj.mmmfb.cn/80208.Doc
m.qvj.mmmfb.cn/26608.Doc
m.qvj.mmmfb.cn/28624.Doc
m.qvj.mmmfb.cn/73513.Doc
m.qvh.mmmfb.cn/15931.Doc
m.qvh.mmmfb.cn/24842.Doc
m.qvh.mmmfb.cn/88262.Doc
m.qvh.mmmfb.cn/60246.Doc
m.qvh.mmmfb.cn/28800.Doc
m.qvh.mmmfb.cn/06466.Doc
m.qvh.mmmfb.cn/17331.Doc
m.qvh.mmmfb.cn/93557.Doc
m.qvh.mmmfb.cn/44046.Doc
m.qvh.mmmfb.cn/26262.Doc
m.qvh.mmmfb.cn/26682.Doc
m.qvh.mmmfb.cn/37517.Doc
m.qvh.mmmfb.cn/06246.Doc
m.qvh.mmmfb.cn/40646.Doc
m.qvh.mmmfb.cn/13113.Doc
m.qvh.mmmfb.cn/35917.Doc
m.qvh.mmmfb.cn/86064.Doc
m.qvh.mmmfb.cn/06444.Doc
m.qvh.mmmfb.cn/00484.Doc
m.qvh.mmmfb.cn/26884.Doc
m.qvg.mmmfb.cn/28466.Doc
m.qvg.mmmfb.cn/71915.Doc
m.qvg.mmmfb.cn/51557.Doc
m.qvg.mmmfb.cn/84648.Doc
m.qvg.mmmfb.cn/02088.Doc
m.qvg.mmmfb.cn/64626.Doc
m.qvg.mmmfb.cn/55593.Doc
m.qvg.mmmfb.cn/48600.Doc
m.qvg.mmmfb.cn/88804.Doc
m.qvg.mmmfb.cn/62628.Doc
m.qvg.mmmfb.cn/64266.Doc
m.qvg.mmmfb.cn/40200.Doc
m.qvg.mmmfb.cn/86448.Doc
m.qvg.mmmfb.cn/88682.Doc
m.qvg.mmmfb.cn/62048.Doc
m.qvg.mmmfb.cn/99171.Doc
m.qvg.mmmfb.cn/40462.Doc
m.qvg.mmmfb.cn/22020.Doc
m.qvg.mmmfb.cn/60680.Doc
m.qvg.mmmfb.cn/84444.Doc
m.qvf.mmmfb.cn/22480.Doc
m.qvf.mmmfb.cn/55555.Doc
m.qvf.mmmfb.cn/99537.Doc
m.qvf.mmmfb.cn/06680.Doc
m.qvf.mmmfb.cn/37155.Doc
m.qvf.mmmfb.cn/40082.Doc
m.qvf.mmmfb.cn/04626.Doc
m.qvf.mmmfb.cn/20644.Doc
m.qvf.mmmfb.cn/02200.Doc
m.qvf.mmmfb.cn/88640.Doc
m.qvf.mmmfb.cn/39315.Doc
m.qvf.mmmfb.cn/42244.Doc
m.qvf.mmmfb.cn/20682.Doc
m.qvf.mmmfb.cn/06266.Doc
m.qvf.mmmfb.cn/68260.Doc
m.qvf.mmmfb.cn/82424.Doc
m.qvf.mmmfb.cn/19579.Doc
m.qvf.mmmfb.cn/35777.Doc
m.qvf.mmmfb.cn/48620.Doc
m.qvf.mmmfb.cn/82082.Doc
m.qvd.mmmfb.cn/84028.Doc
m.qvd.mmmfb.cn/93153.Doc
m.qvd.mmmfb.cn/19577.Doc
m.qvd.mmmfb.cn/06224.Doc
m.qvd.mmmfb.cn/06268.Doc
m.qvd.mmmfb.cn/64208.Doc
m.qvd.mmmfb.cn/77555.Doc
m.qvd.mmmfb.cn/04428.Doc
m.qvd.mmmfb.cn/40444.Doc
m.qvd.mmmfb.cn/06480.Doc
m.qvd.mmmfb.cn/62444.Doc
m.qvd.mmmfb.cn/11193.Doc
m.qvd.mmmfb.cn/00446.Doc
m.qvd.mmmfb.cn/06206.Doc
m.qvd.mmmfb.cn/42862.Doc
m.qvd.mmmfb.cn/20604.Doc
m.qvd.mmmfb.cn/62264.Doc
m.qvd.mmmfb.cn/02668.Doc
m.qvd.mmmfb.cn/48640.Doc
m.qvd.mmmfb.cn/86068.Doc
m.qvs.mmmfb.cn/53537.Doc
m.qvs.mmmfb.cn/44828.Doc
m.qvs.mmmfb.cn/06608.Doc
m.qvs.mmmfb.cn/82860.Doc
m.qvs.mmmfb.cn/40682.Doc
m.qvs.mmmfb.cn/44868.Doc
m.qvs.mmmfb.cn/04246.Doc
m.qvs.mmmfb.cn/82842.Doc
m.qvs.mmmfb.cn/00806.Doc
m.qvs.mmmfb.cn/60808.Doc
m.qvs.mmmfb.cn/24484.Doc
m.qvs.mmmfb.cn/08004.Doc
m.qvs.mmmfb.cn/08868.Doc
m.qvs.mmmfb.cn/88008.Doc
m.qvs.mmmfb.cn/15959.Doc
m.qvs.mmmfb.cn/66864.Doc
m.qvs.mmmfb.cn/64684.Doc
m.qvs.mmmfb.cn/68088.Doc
m.qvs.mmmfb.cn/26428.Doc
m.qvs.mmmfb.cn/31793.Doc
m.qva.mmmfb.cn/62600.Doc
m.qva.mmmfb.cn/44606.Doc
m.qva.mmmfb.cn/02484.Doc
m.qva.mmmfb.cn/00280.Doc
m.qva.mmmfb.cn/53199.Doc
m.qva.mmmfb.cn/44828.Doc
m.qva.mmmfb.cn/22828.Doc
m.qva.mmmfb.cn/66202.Doc
m.qva.mmmfb.cn/80608.Doc
m.qva.mmmfb.cn/39953.Doc
m.qva.mmmfb.cn/28222.Doc
m.qva.mmmfb.cn/24088.Doc
m.qva.mmmfb.cn/20208.Doc
m.qva.mmmfb.cn/46404.Doc
m.qva.mmmfb.cn/17915.Doc
