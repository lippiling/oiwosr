淖来督嗣故


Linux tcp_write_queue_tail 发送队列管理与 tsq 处理

tcp_write_queue_tail 是 TCP 发送队列（sk_write_queue）的操作宏，定义在 include/net/tcp.h 中。sk_write_queue 是 struct sock 的 sk_write_queue 字段，类型为 struct sk_buff_head，管理所有已构造但尚未（完全）确认的 TCP 报文段。队列中的每个 skb 使用 tcp_skb_cb->seq、tcp_skb_cb->end_seq 以及 tcp_skb_cb->tcp_flags 标识数据边界和控制信息。

```c
static inline struct sk_buff *tcp_write_queue_tail(const struct sock *sk)
{
    return skb_peek_tail(&sk->sk_write_queue);
}

static inline struct sk_buff *tcp_write_queue_head(const struct sock *sk)
{
    return skb_peek(&sk->sk_write_queue);
}

static inline struct sk_buff *tcp_send_head(const struct sock *sk)
{
    return skb_peek(&sk->sk_write_queue);
}
```

tcp_send_head 返回发送队列头部（即最早未被确认的 skb），而 tcp_write_queue_tail 返回队列尾部——即最新被追加的 skb。两者的差异在 GSO 分段和 Nagle 算法中至关重要：tcp_write_xmit 循环遍历时从 tcp_send_head 开始，每次发送后如果整个 skb 被确认则调用 tcp_advance_send_head 前移队列头部。若 tcp_write_queue_head == tcp_write_queue_tail（仅一个 skb）被完全发送并确认，队列变空，tcp_send_head(NULL) 作为写关闭信号。

skb 追加操作通过 __skb_queue_tail 或 skb_queue_tail 完成。用户进程调用 tcp_sendmsg 经过 tcp_push_one 或 tcp_write_xmit 的分段逻辑后，新 skb 附着到队列尾部。这里的关键约束：skb->truesize 累积计入 sk->sk_wmem_queued 和 sk->sk_forward_alloc，tcp_sendmsg 在 sk_stream_alloc_skb 失败时返回 -ENOMEM，此时 skb 未入队，数据可能部分写入到 cork 的 frags 中。

```c
static int tcp_push_one(struct sock *sk, unsigned int mss_now, unsigned int nonagle)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *skb = tcp_send_head(sk);

    if (!skb)
        return 0;

    tcp_mark_push(tp, skb);
    __skb_queue_tail(&sk->sk_write_queue, skb);
    ...
    return tcp_write_xmit(sk, mss_now, nonagle, 0, GFP_KERNEL);
}
```

write_queue 的管理涉及三个关键尾指针操作：tcp_write_queue_tail(sk) 用于获取最后一个 skb 以在其后追加新数据（tcp_fragment 分割时同样需要尾部插入）；tcp_write_queue_next(sk, skb) 和 tcp_write_queue_prev(sk, skb) 用于遍历。遍历过程在 tcp_sacktag_write_queue 和 tcp_clean_rtx_queue 中大量使用。

当 tcp_write_xmit 循环发送时，每次调用 tcp_transmit_skb 之前检查是否触发了 TSQ 限速。TSQ 在 tcp_write_xmit 内部的判断通过 tcp_small_queue_check 完成。若触发，TSQ_THROTTLED 位置位且发送中断。此时 __tcp_transmit_skb 尚未移除 skb，重传队列头部保持不变。

```c
static void tcp_tsq_handler(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk);
    unsigned long flags = smp_load_acquire(&sk->sk_tsq_flags);

    if (flags & TCPF_TSQ_DEFERRED) {
        bh_lock_sock(sk);
        if (!sock_owned_by_user(sk)) {
            if (tp->lost_out > tp->retrans_out)
                tcp_write_xmit(sk, tcp_current_mss(sk), tp->nonagle,
                               0, GFP_ATOMIC);
            tcp_push_pending_frames(sk);
        }
        bh_unlock_sock(sk);
    }
}
```

tsq 的 deferred 处理通过 tcp_release_cb 被调用。当发送路径因 TSQ 被限速，并且用户进程持有 lock_sock 时，skb 释放后的 tcp_wfree 将 TSQ_DEFERRED 置位。随后用户进程在 release_sock 中调用 tcp_release_cb，触发 tcp_tsq_handler 重新尝试发送。这里是 TSQ 的核心竞争点：tcp_wfree 运行在 NET_TX_SOFTIRQ 上下文，而 tcp_release_cb 运行在进程上下文，两者对 sk_tsq_flags 的 test_and_set_bit 操作通过原子位操作保证可见性。

tcp_push_pending_frames 在发送队列尾部处理 PSH 标志和紧急数据。该函数检查 tcp_send_head(sk) 是否为 NULL，若不为空则调用 tcp_write_xmit 继续发送。在 tp->packets_out 为 0 但 write_queue 非空（即所有数据已发送但未确认）时，__tcp_push_pending_frames 只做极简的 wmem 检查后返回。

超时重传对 write_queue 尾部的影响：tcp_retransmit_skb 不从队列中移除 skb，而是克隆后重新提交给 IP 层。skb->sk 指针维持指向原 socket，但 cloned skb 的 destructor 由 tcp_wfree 处理。原始 skb 保留在 write_queue 中直到被 tcp_clean_rtx_queue 移除。

```c
int tcp_retransmit_skb(struct sock *sk, struct sk_buff *skb, int segs)
{
    struct tcp_sock *tp = tcp_sk(sk);
    int err = __tcp_retransmit_skb(sk, skb);

    if (err == 0) {
        tp->retrans_out += tcp_skb_pcount(skb);
        ...
        if (TCP_SKB_CB(skb)->seq == tp->snd_una) {
            if (tp->lost_out)
                tp->lost_out -= tcp_skb_pcount(skb);
        }
    }
    ...
    return err;
}
```

write_queue 中存在一个边界情况：TCP_SKB_CB(skb)->tcp_flags & TCPHDR_SYN 出现在三次握手阶段的 SYN/ACK 重传。此时 skb 处于 sk_write_queue 中但 writeset（即 tp->write_seq）尚未初始化完全，tcp_retransmit_skb 的检查路径要求验证 sk->sk_state == TCP_SYN_SENT 或 TCP_SYN_RECV。

另一个边界条件是 tcp_write_queue_tail 与 tcp_collapse 的交互。当发送缓冲区积压过多小包，tcp_collapse 将相邻 skb 合并以降低 sk_buff 管理开销。tcp_collapse 从 write_queue 中移除多个 skb 并合并成一个，重新追加到尾部。这要求操作持有 write_queue 的自旋锁，且如果 collapse 发生在 BH 上下文的 tcp_ack 中，不能与用户态的 tcp_sendmsg 并发操作 write_queue——通常通过 sk->sk_lock 的 owned_by_user 标志互斥。
栈耘抡胺雀擞坏聪橇嘉握盐未炭啡

m.whp.hxbsg.cn/33135.Doc
m.whp.hxbsg.cn/66066.Doc
m.whp.hxbsg.cn/48682.Doc
m.whp.hxbsg.cn/37197.Doc
m.whp.hxbsg.cn/17133.Doc
m.whp.hxbsg.cn/35195.Doc
m.whp.hxbsg.cn/40466.Doc
m.whp.hxbsg.cn/39711.Doc
m.whp.hxbsg.cn/97175.Doc
m.whp.hxbsg.cn/59933.Doc
m.whp.hxbsg.cn/95577.Doc
m.whp.hxbsg.cn/26200.Doc
m.whp.hxbsg.cn/66020.Doc
m.whp.hxbsg.cn/39935.Doc
m.whp.hxbsg.cn/51311.Doc
m.whp.hxbsg.cn/22084.Doc
m.whp.hxbsg.cn/08848.Doc
m.whp.hxbsg.cn/84400.Doc
m.whp.hxbsg.cn/99791.Doc
m.whp.hxbsg.cn/06606.Doc
m.who.hxbsg.cn/71773.Doc
m.who.hxbsg.cn/60462.Doc
m.who.hxbsg.cn/95937.Doc
m.who.hxbsg.cn/53711.Doc
m.who.hxbsg.cn/73715.Doc
m.who.hxbsg.cn/15535.Doc
m.who.hxbsg.cn/80042.Doc
m.who.hxbsg.cn/17535.Doc
m.who.hxbsg.cn/59555.Doc
m.who.hxbsg.cn/20084.Doc
m.who.hxbsg.cn/75755.Doc
m.who.hxbsg.cn/31177.Doc
m.who.hxbsg.cn/13991.Doc
m.who.hxbsg.cn/68808.Doc
m.who.hxbsg.cn/60006.Doc
m.who.hxbsg.cn/91395.Doc
m.who.hxbsg.cn/26822.Doc
m.who.hxbsg.cn/79397.Doc
m.who.hxbsg.cn/97357.Doc
m.who.hxbsg.cn/82826.Doc
m.whi.hxbsg.cn/75951.Doc
m.whi.hxbsg.cn/93731.Doc
m.whi.hxbsg.cn/88804.Doc
m.whi.hxbsg.cn/79717.Doc
m.whi.hxbsg.cn/79375.Doc
m.whi.hxbsg.cn/35937.Doc
m.whi.hxbsg.cn/93779.Doc
m.whi.hxbsg.cn/11753.Doc
m.whi.hxbsg.cn/71713.Doc
m.whi.hxbsg.cn/84426.Doc
m.whi.hxbsg.cn/68666.Doc
m.whi.hxbsg.cn/99331.Doc
m.whi.hxbsg.cn/59751.Doc
m.whi.hxbsg.cn/46606.Doc
m.whi.hxbsg.cn/08068.Doc
m.whi.hxbsg.cn/93933.Doc
m.whi.hxbsg.cn/22860.Doc
m.whi.hxbsg.cn/75791.Doc
m.whi.hxbsg.cn/31555.Doc
m.whi.hxbsg.cn/44022.Doc
m.whu.hxbsg.cn/15739.Doc
m.whu.hxbsg.cn/00286.Doc
m.whu.hxbsg.cn/55739.Doc
m.whu.hxbsg.cn/66226.Doc
m.whu.hxbsg.cn/80242.Doc
m.whu.hxbsg.cn/93737.Doc
m.whu.hxbsg.cn/19353.Doc
m.whu.hxbsg.cn/44084.Doc
m.whu.hxbsg.cn/60644.Doc
m.whu.hxbsg.cn/62026.Doc
m.whu.hxbsg.cn/24424.Doc
m.whu.hxbsg.cn/57137.Doc
m.whu.hxbsg.cn/99359.Doc
m.whu.hxbsg.cn/73739.Doc
m.whu.hxbsg.cn/57773.Doc
m.whu.hxbsg.cn/26288.Doc
m.whu.hxbsg.cn/44826.Doc
m.whu.hxbsg.cn/39931.Doc
m.whu.hxbsg.cn/20062.Doc
m.whu.hxbsg.cn/64824.Doc
m.why.hxbsg.cn/55513.Doc
m.why.hxbsg.cn/64280.Doc
m.why.hxbsg.cn/15991.Doc
m.why.hxbsg.cn/13535.Doc
m.why.hxbsg.cn/62844.Doc
m.why.hxbsg.cn/24886.Doc
m.why.hxbsg.cn/20046.Doc
m.why.hxbsg.cn/68006.Doc
m.why.hxbsg.cn/31915.Doc
m.why.hxbsg.cn/59931.Doc
m.why.hxbsg.cn/68228.Doc
m.why.hxbsg.cn/00868.Doc
m.why.hxbsg.cn/15137.Doc
m.why.hxbsg.cn/11339.Doc
m.why.hxbsg.cn/37375.Doc
m.why.hxbsg.cn/66826.Doc
m.why.hxbsg.cn/86886.Doc
m.why.hxbsg.cn/33333.Doc
m.why.hxbsg.cn/02868.Doc
m.why.hxbsg.cn/60206.Doc
m.wht.hxbsg.cn/42462.Doc
m.wht.hxbsg.cn/91937.Doc
m.wht.hxbsg.cn/82860.Doc
m.wht.hxbsg.cn/15995.Doc
m.wht.hxbsg.cn/02000.Doc
m.wht.hxbsg.cn/55711.Doc
m.wht.hxbsg.cn/37159.Doc
m.wht.hxbsg.cn/75977.Doc
m.wht.hxbsg.cn/62820.Doc
m.wht.hxbsg.cn/33533.Doc
m.wht.hxbsg.cn/71737.Doc
m.wht.hxbsg.cn/71571.Doc
m.wht.hxbsg.cn/62268.Doc
m.wht.hxbsg.cn/24282.Doc
m.wht.hxbsg.cn/79179.Doc
m.wht.hxbsg.cn/91537.Doc
m.wht.hxbsg.cn/31355.Doc
m.wht.hxbsg.cn/28266.Doc
m.wht.hxbsg.cn/11393.Doc
m.wht.hxbsg.cn/57937.Doc
m.whr.hxbsg.cn/93375.Doc
m.whr.hxbsg.cn/66202.Doc
m.whr.hxbsg.cn/88860.Doc
m.whr.hxbsg.cn/73393.Doc
m.whr.hxbsg.cn/55555.Doc
m.whr.hxbsg.cn/84640.Doc
m.whr.hxbsg.cn/71759.Doc
m.whr.hxbsg.cn/95557.Doc
m.whr.hxbsg.cn/64862.Doc
m.whr.hxbsg.cn/02022.Doc
m.whr.hxbsg.cn/88660.Doc
m.whr.hxbsg.cn/91337.Doc
m.whr.hxbsg.cn/77393.Doc
m.whr.hxbsg.cn/55155.Doc
m.whr.hxbsg.cn/84006.Doc
m.whr.hxbsg.cn/66466.Doc
m.whr.hxbsg.cn/48428.Doc
m.whr.hxbsg.cn/62204.Doc
m.whr.hxbsg.cn/19395.Doc
m.whr.hxbsg.cn/22682.Doc
m.whe.hxbsg.cn/20488.Doc
m.whe.hxbsg.cn/60660.Doc
m.whe.hxbsg.cn/24688.Doc
m.whe.hxbsg.cn/33937.Doc
m.whe.hxbsg.cn/91535.Doc
m.whe.hxbsg.cn/22206.Doc
m.whe.hxbsg.cn/79173.Doc
m.whe.hxbsg.cn/37511.Doc
m.whe.hxbsg.cn/11599.Doc
m.whe.hxbsg.cn/13519.Doc
m.whe.hxbsg.cn/93579.Doc
m.whe.hxbsg.cn/28604.Doc
m.whe.hxbsg.cn/79713.Doc
m.whe.hxbsg.cn/62220.Doc
m.whe.hxbsg.cn/40864.Doc
m.whe.hxbsg.cn/88806.Doc
m.whe.hxbsg.cn/04004.Doc
m.whe.hxbsg.cn/77591.Doc
m.whe.hxbsg.cn/39573.Doc
m.whe.hxbsg.cn/26662.Doc
m.whw.hxbsg.cn/53173.Doc
m.whw.hxbsg.cn/26000.Doc
m.whw.hxbsg.cn/59533.Doc
m.whw.hxbsg.cn/04600.Doc
m.whw.hxbsg.cn/53791.Doc
m.whw.hxbsg.cn/60280.Doc
m.whw.hxbsg.cn/33395.Doc
m.whw.hxbsg.cn/75939.Doc
m.whw.hxbsg.cn/46480.Doc
m.whw.hxbsg.cn/53951.Doc
m.whw.hxbsg.cn/08668.Doc
m.whw.hxbsg.cn/26028.Doc
m.whw.hxbsg.cn/40046.Doc
m.whw.hxbsg.cn/26842.Doc
m.whw.hxbsg.cn/55711.Doc
m.whw.hxbsg.cn/73795.Doc
m.whw.hxbsg.cn/31159.Doc
m.whw.hxbsg.cn/86624.Doc
m.whw.hxbsg.cn/80466.Doc
m.whw.hxbsg.cn/35155.Doc
m.whq.hxbsg.cn/42826.Doc
m.whq.hxbsg.cn/59119.Doc
m.whq.hxbsg.cn/19331.Doc
m.whq.hxbsg.cn/51933.Doc
m.whq.hxbsg.cn/80866.Doc
m.whq.hxbsg.cn/60464.Doc
m.whq.hxbsg.cn/55555.Doc
m.whq.hxbsg.cn/53197.Doc
m.whq.hxbsg.cn/02466.Doc
m.whq.hxbsg.cn/64482.Doc
m.whq.hxbsg.cn/99117.Doc
m.whq.hxbsg.cn/48824.Doc
m.whq.hxbsg.cn/51991.Doc
m.whq.hxbsg.cn/42028.Doc
m.whq.hxbsg.cn/40440.Doc
m.whq.hxbsg.cn/77317.Doc
m.whq.hxbsg.cn/06884.Doc
m.whq.hxbsg.cn/35911.Doc
m.whq.hxbsg.cn/11399.Doc
m.whq.hxbsg.cn/64004.Doc
m.wgm.hxbsg.cn/77533.Doc
m.wgm.hxbsg.cn/28266.Doc
m.wgm.hxbsg.cn/86442.Doc
m.wgm.hxbsg.cn/53119.Doc
m.wgm.hxbsg.cn/95353.Doc
m.wgm.hxbsg.cn/95391.Doc
m.wgm.hxbsg.cn/62866.Doc
m.wgm.hxbsg.cn/22260.Doc
m.wgm.hxbsg.cn/62040.Doc
m.wgm.hxbsg.cn/28086.Doc
m.wgm.hxbsg.cn/46428.Doc
m.wgm.hxbsg.cn/13939.Doc
m.wgm.hxbsg.cn/73719.Doc
m.wgm.hxbsg.cn/00266.Doc
m.wgm.hxbsg.cn/79713.Doc
m.wgm.hxbsg.cn/08042.Doc
m.wgm.hxbsg.cn/80800.Doc
m.wgm.hxbsg.cn/79939.Doc
m.wgm.hxbsg.cn/20266.Doc
m.wgm.hxbsg.cn/88240.Doc
m.wgn.hxbsg.cn/53737.Doc
m.wgn.hxbsg.cn/71371.Doc
m.wgn.hxbsg.cn/40246.Doc
m.wgn.hxbsg.cn/59393.Doc
m.wgn.hxbsg.cn/22402.Doc
m.wgn.hxbsg.cn/33533.Doc
m.wgn.hxbsg.cn/93193.Doc
m.wgn.hxbsg.cn/26280.Doc
m.wgn.hxbsg.cn/57515.Doc
m.wgn.hxbsg.cn/84408.Doc
m.wgn.hxbsg.cn/88422.Doc
m.wgn.hxbsg.cn/40646.Doc
m.wgn.hxbsg.cn/08082.Doc
m.wgn.hxbsg.cn/48208.Doc
m.wgn.hxbsg.cn/15115.Doc
m.wgn.hxbsg.cn/95771.Doc
m.wgn.hxbsg.cn/26806.Doc
m.wgn.hxbsg.cn/15335.Doc
m.wgn.hxbsg.cn/28422.Doc
m.wgn.hxbsg.cn/42222.Doc
m.wgb.hxbsg.cn/28888.Doc
m.wgb.hxbsg.cn/66088.Doc
m.wgb.hxbsg.cn/84080.Doc
m.wgb.hxbsg.cn/53177.Doc
m.wgb.hxbsg.cn/75915.Doc
m.wgb.hxbsg.cn/57515.Doc
m.wgb.hxbsg.cn/08682.Doc
m.wgb.hxbsg.cn/86462.Doc
m.wgb.hxbsg.cn/28428.Doc
m.wgb.hxbsg.cn/24628.Doc
m.wgb.hxbsg.cn/71579.Doc
m.wgb.hxbsg.cn/17513.Doc
m.wgb.hxbsg.cn/99739.Doc
m.wgb.hxbsg.cn/93735.Doc
m.wgb.hxbsg.cn/53591.Doc
m.wgb.hxbsg.cn/00862.Doc
m.wgb.hxbsg.cn/60640.Doc
m.wgb.hxbsg.cn/28468.Doc
m.wgb.hxbsg.cn/51317.Doc
m.wgb.hxbsg.cn/04260.Doc
m.wgv.hxbsg.cn/80602.Doc
m.wgv.hxbsg.cn/15773.Doc
m.wgv.hxbsg.cn/75551.Doc
m.wgv.hxbsg.cn/39991.Doc
m.wgv.hxbsg.cn/59175.Doc
m.wgv.hxbsg.cn/68622.Doc
m.wgv.hxbsg.cn/66266.Doc
m.wgv.hxbsg.cn/93397.Doc
m.wgv.hxbsg.cn/35973.Doc
m.wgv.hxbsg.cn/15195.Doc
m.wgv.hxbsg.cn/11917.Doc
m.wgv.hxbsg.cn/86440.Doc
m.wgv.hxbsg.cn/99517.Doc
m.wgv.hxbsg.cn/20800.Doc
m.wgv.hxbsg.cn/42666.Doc
m.wgv.hxbsg.cn/59553.Doc
m.wgv.hxbsg.cn/48220.Doc
m.wgv.hxbsg.cn/08620.Doc
m.wgv.hxbsg.cn/77751.Doc
m.wgv.hxbsg.cn/55159.Doc
m.wgc.hxbsg.cn/48288.Doc
m.wgc.hxbsg.cn/13311.Doc
m.wgc.hxbsg.cn/31911.Doc
m.wgc.hxbsg.cn/55757.Doc
m.wgc.hxbsg.cn/97919.Doc
m.wgc.hxbsg.cn/51395.Doc
m.wgc.hxbsg.cn/75735.Doc
m.wgc.hxbsg.cn/33119.Doc
m.wgc.hxbsg.cn/33511.Doc
m.wgc.hxbsg.cn/80624.Doc
m.wgc.hxbsg.cn/80620.Doc
m.wgc.hxbsg.cn/00062.Doc
m.wgc.hxbsg.cn/06442.Doc
m.wgc.hxbsg.cn/66064.Doc
m.wgc.hxbsg.cn/84646.Doc
m.wgc.hxbsg.cn/08822.Doc
m.wgc.hxbsg.cn/97357.Doc
m.wgc.hxbsg.cn/04842.Doc
m.wgc.hxbsg.cn/86280.Doc
m.wgc.hxbsg.cn/66802.Doc
m.wgx.hxbsg.cn/48422.Doc
m.wgx.hxbsg.cn/95537.Doc
m.wgx.hxbsg.cn/15155.Doc
m.wgx.hxbsg.cn/80086.Doc
m.wgx.hxbsg.cn/19951.Doc
m.wgx.hxbsg.cn/35531.Doc
m.wgx.hxbsg.cn/40640.Doc
m.wgx.hxbsg.cn/64442.Doc
m.wgx.hxbsg.cn/68428.Doc
m.wgx.hxbsg.cn/46820.Doc
m.wgx.hxbsg.cn/91575.Doc
m.wgx.hxbsg.cn/37159.Doc
m.wgx.hxbsg.cn/31753.Doc
m.wgx.hxbsg.cn/84808.Doc
m.wgx.hxbsg.cn/64242.Doc
m.wgx.hxbsg.cn/84486.Doc
m.wgx.hxbsg.cn/04066.Doc
m.wgx.hxbsg.cn/24242.Doc
m.wgx.hxbsg.cn/62082.Doc
m.wgx.hxbsg.cn/35193.Doc
m.wgz.hxbsg.cn/08662.Doc
m.wgz.hxbsg.cn/33119.Doc
m.wgz.hxbsg.cn/75135.Doc
m.wgz.hxbsg.cn/84844.Doc
m.wgz.hxbsg.cn/93737.Doc
m.wgz.hxbsg.cn/84886.Doc
m.wgz.hxbsg.cn/59959.Doc
m.wgz.hxbsg.cn/57317.Doc
m.wgz.hxbsg.cn/13577.Doc
m.wgz.hxbsg.cn/82448.Doc
m.wgz.hxbsg.cn/60242.Doc
m.wgz.hxbsg.cn/82442.Doc
m.wgz.hxbsg.cn/22020.Doc
m.wgz.hxbsg.cn/60222.Doc
m.wgz.hxbsg.cn/68648.Doc
m.wgz.hxbsg.cn/15771.Doc
m.wgz.hxbsg.cn/84462.Doc
m.wgz.hxbsg.cn/00644.Doc
m.wgz.hxbsg.cn/60880.Doc
m.wgz.hxbsg.cn/59939.Doc
m.wgl.hxbsg.cn/00442.Doc
m.wgl.hxbsg.cn/33599.Doc
m.wgl.hxbsg.cn/08622.Doc
m.wgl.hxbsg.cn/59797.Doc
m.wgl.hxbsg.cn/59795.Doc
m.wgl.hxbsg.cn/44060.Doc
m.wgl.hxbsg.cn/51797.Doc
m.wgl.hxbsg.cn/24600.Doc
m.wgl.hxbsg.cn/17311.Doc
m.wgl.hxbsg.cn/68228.Doc
m.wgl.hxbsg.cn/73511.Doc
m.wgl.hxbsg.cn/15739.Doc
m.wgl.hxbsg.cn/91195.Doc
m.wgl.hxbsg.cn/44648.Doc
m.wgl.hxbsg.cn/35337.Doc
m.wgl.hxbsg.cn/48488.Doc
m.wgl.hxbsg.cn/15919.Doc
m.wgl.hxbsg.cn/26880.Doc
m.wgl.hxbsg.cn/51377.Doc
m.wgl.hxbsg.cn/57991.Doc
m.wgk.hxbsg.cn/71713.Doc
m.wgk.hxbsg.cn/22046.Doc
m.wgk.hxbsg.cn/60846.Doc
m.wgk.hxbsg.cn/53177.Doc
m.wgk.hxbsg.cn/84466.Doc
m.wgk.hxbsg.cn/64646.Doc
m.wgk.hxbsg.cn/17757.Doc
m.wgk.hxbsg.cn/88042.Doc
m.wgk.hxbsg.cn/13971.Doc
m.wgk.hxbsg.cn/82444.Doc
m.wgk.hxbsg.cn/26400.Doc
m.wgk.hxbsg.cn/62424.Doc
m.wgk.hxbsg.cn/40824.Doc
m.wgk.hxbsg.cn/57557.Doc
m.wgk.hxbsg.cn/22062.Doc
m.wgk.hxbsg.cn/26428.Doc
m.wgk.hxbsg.cn/51377.Doc
m.wgk.hxbsg.cn/22000.Doc
m.wgk.hxbsg.cn/17333.Doc
m.wgk.hxbsg.cn/35599.Doc
m.wgj.hxbsg.cn/04202.Doc
m.wgj.hxbsg.cn/64062.Doc
m.wgj.hxbsg.cn/01731.Doc
m.wgj.hxbsg.cn/77553.Doc
m.wgj.hxbsg.cn/88628.Doc
m.wgj.hxbsg.cn/93511.Doc
m.wgj.hxbsg.cn/02608.Doc
m.wgj.hxbsg.cn/11579.Doc
m.wgj.hxbsg.cn/86006.Doc
m.wgj.hxbsg.cn/33599.Doc
m.wgj.hxbsg.cn/82288.Doc
m.wgj.hxbsg.cn/44860.Doc
m.wgj.hxbsg.cn/11559.Doc
m.wgj.hxbsg.cn/24266.Doc
m.wgj.hxbsg.cn/42862.Doc
