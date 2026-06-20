僭狙杆牙阜


Linux tcp_transmit_skb 发送路径与 TSQ 线程队列机制

tcp_transmit_skb 是 TCP 发送路径的出口函数，将 sk_buff 从 TCP 层传递到 IP 层。该函数在 tcp_write_xmit、tcp_retransmit_skb、tcp_send_ack 等多个代码路径被调用，每次调用完成一次独立的报文发送。函数签名包含大量的控制参数：clone_it 控制是否克隆 skb（ACK 和窗口探测等控制报文必须克隆，避免与重传队列共享 skb），gfp_mask 控制内存分配优先级，tcp_transmit_skb 在释放 skb 前通过 __skb_clone 或 skb_clone 做深拷贝。

```c
static int __tcp_transmit_skb(struct sock *sk, struct sk_buff *skb,
                               int clone_it, gfp_t gfp_mask,
                               u32 rcv_nxt, u32 seq, u32 ack,
                               int tcp_header_size)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct sk_buff *oskb = NULL;
    int err;

    if (clone_it) {
        os kb = skb;
        skb = skb_clone(skb, gfp_mask);
        if (unlikely(!skb))
            return -ENOBUFS;
    }

    skb->sk = sk;
    skb->destructor = tcp_wfree;
    skb_shinfo(skb)->gso_size = 0;
    ...
    tcp_header_size = tp->tcp_header_len;
    skb_push(skb, tcp_header_size);
    skb_reset_transport_header(skb);

    th = (struct tcphdr *)skb->data;
    th->source = inet->inet_sport;
    th->dest = inet->inet_dport;
    th->seq = htonl(seq);
    th->ack_seq = htonl(ack);
    ...
    err = icsk->icsk_af_ops->queue_xmit(sk, skb, &inet->cork.fl);
}
```

tcp_wfree 是 skb 释放回调函数，是 TSQ（TCP Small Queues）机制的核心入口。当 skb 被设备驱动释放（DMA 完成或 xmit 失败），tcp_wfree 检查当前 socket 在 qdisc/device 队列中积压的 bytes 总量，若超过 sk->sk_wmem_queued 的阈值则触发限速。TSQ 通过将限速逻辑从发送侧推迟到释放侧，避免 sender 在 tcp_write_xmit 中做复杂的流量整形。sk_wmem_alloc 是原子计数器，每发送一个 skb 递增，释放时递减。

TSQ 的核心逻辑在 tcp_write_xmit 中：每次准备发送新分段前，检查 tp->snd_wnd 和 tp->packets_out，但更重要的约束是 sk->sk_wmem_alloc 的值不可超过每个 CPU 的限额。默认限额由 net.ipv4.tcp_limit_output_bytes 控制（262144 字节），超过该值则停止发送并排队到 TSQ 线程，待设备释放足够 skb 后通过 tasklet（tsq_enqueue）重新唤醒发送。

```c
static bool tcp_small_queue_check(struct sock *sk, const struct sk_buff *skb,
                                  unsigned int factor)
{
    unsigned int limit;

    limit = max_t(unsigned int, 2 * skb->truesize, sk->sk_pacing_rate >> 10);
    limit = min_t(u32, limit, READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_limit_output_bytes));
    limit <<= factor;

    if (refcount_read(&sk->sk_wmem_alloc) > limit) {
        if (!test_bit(TSQ_THROTTLED, &sk->sk_tsq_flags))
            set_bit(TSQ_THROTTLED, &sk->sk_tsq_flags);
        return true;
    }
    return false;
}
```

factor 参数对应 GSO 分段数（skb_shinfo(skb)->gso_segs）的对数，作用是非线性放大 GSO 大包的阈值从而允许更大的 burst。这里存在一个边界条件：如果 tcp_small_queue_check 返回 true，则 tcp_write_xmit 中断发送循环，发送状态停留在 tp->snd_nxt 未更新位置。用户线程继续写入数据时由于 sk_stream_memory_free 检查 sk_wmem_queued 已满而阻塞，形成发送反压。

TSQ 线程处理通过 tsq_work 工作队列延迟调度。tsq_enqueue 将当前 socket 加入 per-CPU 的 TSQ 链表，触发 tasklet 或 workqueue。tsq_handler 遍历链表上所有限速 socket，调用 tcp_release_cb 来重新发送。tcp_release_cb 在 socket lock 不被持有（bh_lock_sock 无效）时通过 __release_sock 消耗 backlog 队列。

```c
void tcp_release_cb(struct sock *sk)
{
    unsigned long flags = smp_load_acquire(&sk->sk_tsq_flags);

    if (flags & TCPF_TSQ_DEFERRED) {
        flags &= ~TCPF_TSQ_DEFERRED;
        if (!test_and_clear_bit(TSQ_DEFERRED, &sk->sk_tsq_flags))
            goto ignore;
        bh_lock_sock(sk);
        if (!sock_owned_by_user(sk))
            tcp_tsq_handler(sk);
        bh_unlock_sock(sk);
    }
    if (flags & TCPF_WRITE_TIMER_DEFERRED) {
        tcp_write_timer_handler(sk);
    }
}
```

tcp_tsq_handler 调用 tcp_write_xmit 和 tcp_push_pending_frames 尝试发送 backlog 中的 skb。此时由于 sk_wmem_alloc 已有部分释放（设备完成发送），tcp_small_queue_check 的阈值未超出，发送可以继续进行。

另一方面，tcp_transmit_skb 在外层 __tcp_transmit_skb 中调用 icsk_af_ops->queue_xmit 实际进入 IP 层，这是调用 dev_queue_xmit 之前的最后一步。这里的错误处理涉及重要的内存释放路径：若 queue_xmit 返回 NET_XMIT_DROP 或 -ENOBUFS，返回后 tcp_write_xmit 不会将 skb 从 write_queue 移除。在 tcp_transmit_skb 内部如果 skb_clone 失败（-ENOBUFS），skb 不动，tcp_write_xmit 同样不会移除 skb，在下一次发送超时触发的 tcp_retransmit_skb 中再做补偿尝试。

tcp_write_xmit 还调用 tcp_cwnd_validate 检查 cwnd 是否溢出。当发送路径中 TSQ 限速触发时，即使 cwnd 和 snd_wnd 都是够，发送也无法进行——这是 TCP Small Queues 在延迟和 fairness 之间的折衷，在大 RTT 场景（如跨洲链路）TSQ 对单流吞吐的抑制作用尤其明显。

```c
static bool tcp_write_xmit(struct sock *sk, unsigned int mss_now, int nonagle,
                           int push_one, gfp_t gfp)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *skb;
    unsigned int tso_segs, sent_pkts;
    int cwnd_quota;

    while ((skb = tcp_send_head(sk))) {
        tso_segs = tcp_init_tso_segs(skb, mss_now);
        if (!tso_segs)
            break;
        cwnd_quota = tcp_cwnd_test(tp, skb);
        if (!cwnd_quota)
            break;
        if (unlikely(!tcp_snd_wnd_test(tp, skb, mss_now)))
            break;
        if (tcp_small_queue_check(sk, skb, 0))
            break;
        ...
        limit = mss_now * tcp_cwnd_test(tp, skb);
        if (skb->len > limit && tso_segs > 1)
            tcp_mss_split_point(skb, &limit, sk, &skb, mss_now, cwnd_quota, gfp);
        ...
        if (unlikely(tcp_transmit_skb(sk, skb, 1, gfp)))
            break;
        ...
    }
}
```

队列拥塞的反馈循环：当 qdisc 限速导致 skb 滞留在设备队列中时，sk_wmem_alloc 持续高水位→ tcp_small_queue_check 拦截发送→用户态 write 阻塞在 sk_stream_wait_memory→对端不再收到数据→接收窗口关闭→死锁。该场景要求 tcp_wfree 必须在设备队列清空时被调用，否则 sk_wmem_alloc 不会下降。内核在 dev_hard_start_xmit 的 xmit_more 批量发送和 NET_TX_SOFTIRQ 的 complete 回调之间必须保证 tcp_wfree 的调用时序。
宋谙痔皆亚帽窘诱一延却训啬捍又

tv.blog.cvvliy.cn/Article/details/822046.sHtML
tv.blog.cvvliy.cn/Article/details/028888.sHtML
tv.blog.cvvliy.cn/Article/details/393517.sHtML
tv.blog.cvvliy.cn/Article/details/113937.sHtML
tv.blog.cvvliy.cn/Article/details/537359.sHtML
tv.blog.cvvliy.cn/Article/details/913553.sHtML
tv.blog.cvvliy.cn/Article/details/319973.sHtML
tv.blog.cvvliy.cn/Article/details/202002.sHtML
tv.blog.cvvliy.cn/Article/details/133517.sHtML
tv.blog.cvvliy.cn/Article/details/060828.sHtML
tv.blog.cvvliy.cn/Article/details/804404.sHtML
tv.blog.cvvliy.cn/Article/details/286426.sHtML
tv.blog.cvvliy.cn/Article/details/444048.sHtML
tv.blog.cvvliy.cn/Article/details/660222.sHtML
tv.blog.cvvliy.cn/Article/details/282844.sHtML
tv.blog.cvvliy.cn/Article/details/911757.sHtML
tv.blog.cvvliy.cn/Article/details/862042.sHtML
tv.blog.cvvliy.cn/Article/details/426004.sHtML
tv.blog.cvvliy.cn/Article/details/644408.sHtML
tv.blog.cvvliy.cn/Article/details/286000.sHtML
tv.blog.cvvliy.cn/Article/details/531773.sHtML
tv.blog.cvvliy.cn/Article/details/080420.sHtML
tv.blog.cvvliy.cn/Article/details/826280.sHtML
tv.blog.cvvliy.cn/Article/details/484626.sHtML
tv.blog.cvvliy.cn/Article/details/317999.sHtML
tv.blog.cvvliy.cn/Article/details/171753.sHtML
tv.blog.cvvliy.cn/Article/details/860664.sHtML
tv.blog.cvvliy.cn/Article/details/486862.sHtML
tv.blog.cvvliy.cn/Article/details/422644.sHtML
tv.blog.cvvliy.cn/Article/details/828682.sHtML
tv.blog.cvvliy.cn/Article/details/991991.sHtML
tv.blog.cvvliy.cn/Article/details/713719.sHtML
tv.blog.cvvliy.cn/Article/details/222802.sHtML
tv.blog.cvvliy.cn/Article/details/804800.sHtML
tv.blog.cvvliy.cn/Article/details/248242.sHtML
tv.blog.cvvliy.cn/Article/details/511977.sHtML
tv.blog.cvvliy.cn/Article/details/937331.sHtML
tv.blog.cvvliy.cn/Article/details/280866.sHtML
tv.blog.cvvliy.cn/Article/details/246662.sHtML
tv.blog.cvvliy.cn/Article/details/119193.sHtML
tv.blog.cvvliy.cn/Article/details/282280.sHtML
tv.blog.cvvliy.cn/Article/details/848422.sHtML
tv.blog.cvvliy.cn/Article/details/420408.sHtML
tv.blog.cvvliy.cn/Article/details/848604.sHtML
tv.blog.cvvliy.cn/Article/details/971335.sHtML
tv.blog.cvvliy.cn/Article/details/511311.sHtML
tv.blog.cvvliy.cn/Article/details/515799.sHtML
tv.blog.cvvliy.cn/Article/details/373519.sHtML
tv.blog.cvvliy.cn/Article/details/957731.sHtML
tv.blog.cvvliy.cn/Article/details/886402.sHtML
tv.blog.cvvliy.cn/Article/details/531979.sHtML
tv.blog.cvvliy.cn/Article/details/353755.sHtML
tv.blog.cvvliy.cn/Article/details/155571.sHtML
tv.blog.cvvliy.cn/Article/details/604488.sHtML
tv.blog.cvvliy.cn/Article/details/884802.sHtML
tv.blog.cvvliy.cn/Article/details/795513.sHtML
tv.blog.cvvliy.cn/Article/details/620480.sHtML
tv.blog.cvvliy.cn/Article/details/868206.sHtML
tv.blog.cvvliy.cn/Article/details/955951.sHtML
tv.blog.cvvliy.cn/Article/details/117197.sHtML
tv.blog.cvvliy.cn/Article/details/602606.sHtML
tv.blog.cvvliy.cn/Article/details/775191.sHtML
tv.blog.cvvliy.cn/Article/details/222608.sHtML
tv.blog.cvvliy.cn/Article/details/171533.sHtML
tv.blog.cvvliy.cn/Article/details/024444.sHtML
tv.blog.cvvliy.cn/Article/details/151175.sHtML
tv.blog.cvvliy.cn/Article/details/244262.sHtML
tv.blog.cvvliy.cn/Article/details/795317.sHtML
tv.blog.cvvliy.cn/Article/details/666266.sHtML
tv.blog.cvvliy.cn/Article/details/719175.sHtML
tv.blog.cvvliy.cn/Article/details/119717.sHtML
tv.blog.cvvliy.cn/Article/details/420608.sHtML
tv.blog.cvvliy.cn/Article/details/519397.sHtML
tv.blog.cvvliy.cn/Article/details/082204.sHtML
tv.blog.cvvliy.cn/Article/details/808882.sHtML
tv.blog.cvvliy.cn/Article/details/719793.sHtML
tv.blog.cvvliy.cn/Article/details/622688.sHtML
tv.blog.cvvliy.cn/Article/details/606448.sHtML
tv.blog.cvvliy.cn/Article/details/266688.sHtML
tv.blog.cvvliy.cn/Article/details/355539.sHtML
tv.blog.cvvliy.cn/Article/details/088466.sHtML
tv.blog.cvvliy.cn/Article/details/571555.sHtML
tv.blog.cvvliy.cn/Article/details/606802.sHtML
tv.blog.cvvliy.cn/Article/details/220208.sHtML
tv.blog.cvvliy.cn/Article/details/648840.sHtML
tv.blog.cvvliy.cn/Article/details/755953.sHtML
tv.blog.cvvliy.cn/Article/details/397759.sHtML
tv.blog.cvvliy.cn/Article/details/539355.sHtML
tv.blog.cvvliy.cn/Article/details/288260.sHtML
tv.blog.cvvliy.cn/Article/details/684224.sHtML
tv.blog.cvvliy.cn/Article/details/402288.sHtML
tv.blog.cvvliy.cn/Article/details/599577.sHtML
tv.blog.cvvliy.cn/Article/details/159599.sHtML
tv.blog.cvvliy.cn/Article/details/606644.sHtML
tv.blog.cvvliy.cn/Article/details/533793.sHtML
tv.blog.cvvliy.cn/Article/details/440064.sHtML
tv.blog.cvvliy.cn/Article/details/199519.sHtML
tv.blog.cvvliy.cn/Article/details/533977.sHtML
tv.blog.cvvliy.cn/Article/details/802868.sHtML
tv.blog.cvvliy.cn/Article/details/666806.sHtML
tv.blog.cvvliy.cn/Article/details/191959.sHtML
tv.blog.cvvliy.cn/Article/details/177333.sHtML
tv.blog.cvvliy.cn/Article/details/337797.sHtML
tv.blog.cvvliy.cn/Article/details/028444.sHtML
tv.blog.cvvliy.cn/Article/details/313571.sHtML
tv.blog.cvvliy.cn/Article/details/177951.sHtML
tv.blog.cvvliy.cn/Article/details/208886.sHtML
tv.blog.cvvliy.cn/Article/details/004422.sHtML
tv.blog.cvvliy.cn/Article/details/997539.sHtML
tv.blog.cvvliy.cn/Article/details/957913.sHtML
tv.blog.cvvliy.cn/Article/details/511795.sHtML
tv.blog.cvvliy.cn/Article/details/406804.sHtML
tv.blog.cvvliy.cn/Article/details/371153.sHtML
tv.blog.cvvliy.cn/Article/details/115973.sHtML
tv.blog.cvvliy.cn/Article/details/333977.sHtML
tv.blog.cvvliy.cn/Article/details/373359.sHtML
tv.blog.cvvliy.cn/Article/details/266462.sHtML
tv.blog.cvvliy.cn/Article/details/226400.sHtML
tv.blog.cvvliy.cn/Article/details/919179.sHtML
tv.blog.cvvliy.cn/Article/details/620200.sHtML
tv.blog.cvvliy.cn/Article/details/086428.sHtML
tv.blog.cvvliy.cn/Article/details/080868.sHtML
tv.blog.cvvliy.cn/Article/details/975717.sHtML
tv.blog.cvvliy.cn/Article/details/913919.sHtML
tv.blog.cvvliy.cn/Article/details/828004.sHtML
tv.blog.cvvliy.cn/Article/details/111139.sHtML
tv.blog.cvvliy.cn/Article/details/886440.sHtML
tv.blog.cvvliy.cn/Article/details/939959.sHtML
tv.blog.cvvliy.cn/Article/details/577959.sHtML
tv.blog.cvvliy.cn/Article/details/080246.sHtML
tv.blog.cvvliy.cn/Article/details/153371.sHtML
tv.blog.cvvliy.cn/Article/details/373133.sHtML
tv.blog.cvvliy.cn/Article/details/424222.sHtML
tv.blog.cvvliy.cn/Article/details/204468.sHtML
tv.blog.cvvliy.cn/Article/details/404224.sHtML
tv.blog.cvvliy.cn/Article/details/175977.sHtML
tv.blog.cvvliy.cn/Article/details/731319.sHtML
tv.blog.cvvliy.cn/Article/details/860204.sHtML
tv.blog.cvvliy.cn/Article/details/159919.sHtML
tv.blog.cvvliy.cn/Article/details/995199.sHtML
tv.blog.cvvliy.cn/Article/details/517577.sHtML
tv.blog.cvvliy.cn/Article/details/084264.sHtML
tv.blog.cvvliy.cn/Article/details/804804.sHtML
tv.blog.cvvliy.cn/Article/details/842046.sHtML
tv.blog.cvvliy.cn/Article/details/191133.sHtML
tv.blog.cvvliy.cn/Article/details/660464.sHtML
tv.blog.cvvliy.cn/Article/details/359717.sHtML
tv.blog.cvvliy.cn/Article/details/064442.sHtML
tv.blog.cvvliy.cn/Article/details/713111.sHtML
tv.blog.cvvliy.cn/Article/details/026240.sHtML
tv.blog.cvvliy.cn/Article/details/391759.sHtML
tv.blog.cvvliy.cn/Article/details/191959.sHtML
tv.blog.cvvliy.cn/Article/details/860004.sHtML
tv.blog.cvvliy.cn/Article/details/931717.sHtML
tv.blog.cvvliy.cn/Article/details/755119.sHtML
tv.blog.cvvliy.cn/Article/details/799137.sHtML
tv.blog.cvvliy.cn/Article/details/331391.sHtML
tv.blog.cvvliy.cn/Article/details/060840.sHtML
tv.blog.cvvliy.cn/Article/details/648686.sHtML
tv.blog.cvvliy.cn/Article/details/404844.sHtML
tv.blog.cvvliy.cn/Article/details/008660.sHtML
tv.blog.cvvliy.cn/Article/details/486644.sHtML
tv.blog.cvvliy.cn/Article/details/775319.sHtML
tv.blog.cvvliy.cn/Article/details/804686.sHtML
tv.blog.cvvliy.cn/Article/details/357951.sHtML
tv.blog.cvvliy.cn/Article/details/599719.sHtML
tv.blog.cvvliy.cn/Article/details/719579.sHtML
tv.blog.cvvliy.cn/Article/details/151931.sHtML
tv.blog.cvvliy.cn/Article/details/888642.sHtML
tv.blog.cvvliy.cn/Article/details/777735.sHtML
tv.blog.cvvliy.cn/Article/details/357333.sHtML
tv.blog.cvvliy.cn/Article/details/802266.sHtML
tv.blog.cvvliy.cn/Article/details/355999.sHtML
tv.blog.cvvliy.cn/Article/details/595317.sHtML
tv.blog.cvvliy.cn/Article/details/240282.sHtML
tv.blog.cvvliy.cn/Article/details/975719.sHtML
tv.blog.cvvliy.cn/Article/details/066208.sHtML
tv.blog.cvvliy.cn/Article/details/822646.sHtML
tv.blog.cvvliy.cn/Article/details/919753.sHtML
tv.blog.cvvliy.cn/Article/details/822828.sHtML
tv.blog.cvvliy.cn/Article/details/377991.sHtML
tv.blog.cvvliy.cn/Article/details/915959.sHtML
tv.blog.cvvliy.cn/Article/details/995953.sHtML
tv.blog.cvvliy.cn/Article/details/371117.sHtML
tv.blog.cvvliy.cn/Article/details/226082.sHtML
tv.blog.cvvliy.cn/Article/details/248862.sHtML
tv.blog.cvvliy.cn/Article/details/608404.sHtML
tv.blog.cvvliy.cn/Article/details/535791.sHtML
tv.blog.cvvliy.cn/Article/details/395715.sHtML
tv.blog.cvvliy.cn/Article/details/842442.sHtML
tv.blog.cvvliy.cn/Article/details/602664.sHtML
tv.blog.cvvliy.cn/Article/details/577553.sHtML
tv.blog.cvvliy.cn/Article/details/628642.sHtML
tv.blog.cvvliy.cn/Article/details/480408.sHtML
tv.blog.cvvliy.cn/Article/details/280648.sHtML
tv.blog.cvvliy.cn/Article/details/444246.sHtML
tv.blog.cvvliy.cn/Article/details/553575.sHtML
tv.blog.cvvliy.cn/Article/details/951777.sHtML
tv.blog.cvvliy.cn/Article/details/008424.sHtML
tv.blog.cvvliy.cn/Article/details/177997.sHtML
tv.blog.cvvliy.cn/Article/details/602844.sHtML
tv.blog.cvvliy.cn/Article/details/200864.sHtML
tv.blog.cvvliy.cn/Article/details/644862.sHtML
tv.blog.cvvliy.cn/Article/details/117111.sHtML
tv.blog.cvvliy.cn/Article/details/620064.sHtML
tv.blog.cvvliy.cn/Article/details/511793.sHtML
tv.blog.cvvliy.cn/Article/details/399773.sHtML
tv.blog.cvvliy.cn/Article/details/666004.sHtML
tv.blog.cvvliy.cn/Article/details/737775.sHtML
tv.blog.cvvliy.cn/Article/details/802286.sHtML
tv.blog.cvvliy.cn/Article/details/951919.sHtML
tv.blog.cvvliy.cn/Article/details/422802.sHtML
tv.blog.cvvliy.cn/Article/details/008640.sHtML
tv.blog.cvvliy.cn/Article/details/888840.sHtML
tv.blog.cvvliy.cn/Article/details/137779.sHtML
tv.blog.cvvliy.cn/Article/details/597797.sHtML
tv.blog.cvvliy.cn/Article/details/264666.sHtML
tv.blog.cvvliy.cn/Article/details/466826.sHtML
tv.blog.cvvliy.cn/Article/details/224462.sHtML
tv.blog.cvvliy.cn/Article/details/557911.sHtML
tv.blog.cvvliy.cn/Article/details/351133.sHtML
tv.blog.cvvliy.cn/Article/details/577319.sHtML
tv.blog.cvvliy.cn/Article/details/824862.sHtML
tv.blog.cvvliy.cn/Article/details/755773.sHtML
tv.blog.cvvliy.cn/Article/details/997757.sHtML
tv.blog.cvvliy.cn/Article/details/828688.sHtML
tv.blog.cvvliy.cn/Article/details/488802.sHtML
tv.blog.cvvliy.cn/Article/details/517319.sHtML
tv.blog.cvvliy.cn/Article/details/268060.sHtML
tv.blog.cvvliy.cn/Article/details/153519.sHtML
tv.blog.cvvliy.cn/Article/details/713931.sHtML
tv.blog.cvvliy.cn/Article/details/951537.sHtML
tv.blog.cvvliy.cn/Article/details/482800.sHtML
tv.blog.cvvliy.cn/Article/details/971511.sHtML
tv.blog.cvvliy.cn/Article/details/604486.sHtML
tv.blog.cvvliy.cn/Article/details/400840.sHtML
tv.blog.cvvliy.cn/Article/details/111577.sHtML
tv.blog.cvvliy.cn/Article/details/002848.sHtML
tv.blog.cvvliy.cn/Article/details/662884.sHtML
tv.blog.cvvliy.cn/Article/details/195531.sHtML
tv.blog.cvvliy.cn/Article/details/777197.sHtML
tv.blog.cvvliy.cn/Article/details/395133.sHtML
tv.blog.cvvliy.cn/Article/details/488680.sHtML
tv.blog.cvvliy.cn/Article/details/280288.sHtML
tv.blog.cvvliy.cn/Article/details/531577.sHtML
tv.blog.cvvliy.cn/Article/details/622480.sHtML
tv.blog.cvvliy.cn/Article/details/357377.sHtML
tv.blog.cvvliy.cn/Article/details/197197.sHtML
tv.blog.cvvliy.cn/Article/details/404886.sHtML
tv.blog.cvvliy.cn/Article/details/151191.sHtML
tv.blog.cvvliy.cn/Article/details/731951.sHtML
tv.blog.cvvliy.cn/Article/details/935515.sHtML
tv.blog.cvvliy.cn/Article/details/284448.sHtML
tv.blog.cvvliy.cn/Article/details/804622.sHtML
tv.blog.cvvliy.cn/Article/details/171199.sHtML
tv.blog.cvvliy.cn/Article/details/882464.sHtML
tv.blog.cvvliy.cn/Article/details/282266.sHtML
tv.blog.cvvliy.cn/Article/details/375531.sHtML
tv.blog.cvvliy.cn/Article/details/404624.sHtML
tv.blog.cvvliy.cn/Article/details/888206.sHtML
tv.blog.cvvliy.cn/Article/details/242460.sHtML
tv.blog.cvvliy.cn/Article/details/484864.sHtML
tv.blog.cvvliy.cn/Article/details/462848.sHtML
tv.blog.cvvliy.cn/Article/details/151971.sHtML
tv.blog.cvvliy.cn/Article/details/971739.sHtML
tv.blog.cvvliy.cn/Article/details/808446.sHtML
tv.blog.cvvliy.cn/Article/details/006206.sHtML
tv.blog.cvvliy.cn/Article/details/195559.sHtML
tv.blog.cvvliy.cn/Article/details/046286.sHtML
tv.blog.cvvliy.cn/Article/details/828440.sHtML
tv.blog.cvvliy.cn/Article/details/862426.sHtML
tv.blog.cvvliy.cn/Article/details/422648.sHtML
tv.blog.cvvliy.cn/Article/details/042688.sHtML
tv.blog.cvvliy.cn/Article/details/002008.sHtML
tv.blog.cvvliy.cn/Article/details/953135.sHtML
tv.blog.cvvliy.cn/Article/details/535913.sHtML
tv.blog.cvvliy.cn/Article/details/424208.sHtML
tv.blog.cvvliy.cn/Article/details/579353.sHtML
tv.blog.cvvliy.cn/Article/details/777591.sHtML
tv.blog.cvvliy.cn/Article/details/202260.sHtML
tv.blog.cvvliy.cn/Article/details/957995.sHtML
tv.blog.cvvliy.cn/Article/details/179573.sHtML
tv.blog.cvvliy.cn/Article/details/331733.sHtML
tv.blog.cvvliy.cn/Article/details/557739.sHtML
tv.blog.cvvliy.cn/Article/details/248202.sHtML
tv.blog.cvvliy.cn/Article/details/886820.sHtML
tv.blog.cvvliy.cn/Article/details/044068.sHtML
tv.blog.cvvliy.cn/Article/details/888068.sHtML
tv.blog.cvvliy.cn/Article/details/937377.sHtML
tv.blog.cvvliy.cn/Article/details/579775.sHtML
tv.blog.cvvliy.cn/Article/details/828462.sHtML
tv.blog.cvvliy.cn/Article/details/422060.sHtML
tv.blog.cvvliy.cn/Article/details/535515.sHtML
tv.blog.cvvliy.cn/Article/details/068448.sHtML
tv.blog.cvvliy.cn/Article/details/244208.sHtML
tv.blog.cvvliy.cn/Article/details/357191.sHtML
tv.blog.cvvliy.cn/Article/details/957539.sHtML
tv.blog.cvvliy.cn/Article/details/553115.sHtML
tv.blog.cvvliy.cn/Article/details/537313.sHtML
tv.blog.cvvliy.cn/Article/details/737751.sHtML
tv.blog.cvvliy.cn/Article/details/046062.sHtML
tv.blog.cvvliy.cn/Article/details/135931.sHtML
tv.blog.cvvliy.cn/Article/details/537795.sHtML
tv.blog.cvvliy.cn/Article/details/046642.sHtML
tv.blog.cvvliy.cn/Article/details/606480.sHtML
tv.blog.cvvliy.cn/Article/details/248402.sHtML
tv.blog.cvvliy.cn/Article/details/022048.sHtML
tv.blog.cvvliy.cn/Article/details/113515.sHtML
tv.blog.cvvliy.cn/Article/details/446602.sHtML
tv.blog.cvvliy.cn/Article/details/379137.sHtML
tv.blog.cvvliy.cn/Article/details/664240.sHtML
tv.blog.cvvliy.cn/Article/details/040800.sHtML
tv.blog.cvvliy.cn/Article/details/802266.sHtML
tv.blog.cvvliy.cn/Article/details/482282.sHtML
tv.blog.cvvliy.cn/Article/details/484848.sHtML
tv.blog.cvvliy.cn/Article/details/408020.sHtML
tv.blog.cvvliy.cn/Article/details/517151.sHtML
tv.blog.cvvliy.cn/Article/details/222826.sHtML
tv.blog.cvvliy.cn/Article/details/595199.sHtML
tv.blog.cvvliy.cn/Article/details/319515.sHtML
tv.blog.cvvliy.cn/Article/details/422626.sHtML
tv.blog.cvvliy.cn/Article/details/917793.sHtML
tv.blog.cvvliy.cn/Article/details/117733.sHtML
tv.blog.cvvliy.cn/Article/details/913999.sHtML
tv.blog.cvvliy.cn/Article/details/575573.sHtML
tv.blog.cvvliy.cn/Article/details/048200.sHtML
tv.blog.cvvliy.cn/Article/details/664226.sHtML
tv.blog.cvvliy.cn/Article/details/400642.sHtML
tv.blog.cvvliy.cn/Article/details/068888.sHtML
tv.blog.cvvliy.cn/Article/details/195751.sHtML
tv.blog.cvvliy.cn/Article/details/173579.sHtML
tv.blog.cvvliy.cn/Article/details/224226.sHtML
tv.blog.cvvliy.cn/Article/details/622288.sHtML
tv.blog.cvvliy.cn/Article/details/357559.sHtML
tv.blog.cvvliy.cn/Article/details/179997.sHtML
tv.blog.cvvliy.cn/Article/details/824626.sHtML
tv.blog.cvvliy.cn/Article/details/537759.sHtML
tv.blog.cvvliy.cn/Article/details/866484.sHtML
tv.blog.cvvliy.cn/Article/details/317977.sHtML
tv.blog.cvvliy.cn/Article/details/579711.sHtML
tv.blog.cvvliy.cn/Article/details/537979.sHtML
tv.blog.cvvliy.cn/Article/details/462262.sHtML
tv.blog.cvvliy.cn/Article/details/862068.sHtML
tv.blog.cvvliy.cn/Article/details/113771.sHtML
tv.blog.cvvliy.cn/Article/details/202442.sHtML
tv.blog.cvvliy.cn/Article/details/973755.sHtML
tv.blog.cvvliy.cn/Article/details/571751.sHtML
tv.blog.cvvliy.cn/Article/details/733199.sHtML
tv.blog.cvvliy.cn/Article/details/199139.sHtML
tv.blog.cvvliy.cn/Article/details/171533.sHtML
tv.blog.cvvliy.cn/Article/details/355391.sHtML
tv.blog.cvvliy.cn/Article/details/284248.sHtML
tv.blog.cvvliy.cn/Article/details/844064.sHtML
tv.blog.cvvliy.cn/Article/details/559157.sHtML
tv.blog.cvvliy.cn/Article/details/955557.sHtML
tv.blog.cvvliy.cn/Article/details/375957.sHtML
tv.blog.cvvliy.cn/Article/details/331773.sHtML
tv.blog.cvvliy.cn/Article/details/111591.sHtML
tv.blog.cvvliy.cn/Article/details/131397.sHtML
tv.blog.cvvliy.cn/Article/details/959739.sHtML
tv.blog.cvvliy.cn/Article/details/668442.sHtML
tv.blog.cvvliy.cn/Article/details/597113.sHtML
tv.blog.cvvliy.cn/Article/details/448202.sHtML
tv.blog.cvvliy.cn/Article/details/684026.sHtML
tv.blog.cvvliy.cn/Article/details/937375.sHtML
tv.blog.cvvliy.cn/Article/details/539993.sHtML
tv.blog.cvvliy.cn/Article/details/686242.sHtML
tv.blog.cvvliy.cn/Article/details/646600.sHtML
tv.blog.cvvliy.cn/Article/details/646026.sHtML
tv.blog.cvvliy.cn/Article/details/420426.sHtML
tv.blog.cvvliy.cn/Article/details/575119.sHtML
tv.blog.cvvliy.cn/Article/details/533155.sHtML
tv.blog.cvvliy.cn/Article/details/664426.sHtML
tv.blog.cvvliy.cn/Article/details/846004.sHtML
tv.blog.cvvliy.cn/Article/details/151139.sHtML
tv.blog.cvvliy.cn/Article/details/515739.sHtML
tv.blog.cvvliy.cn/Article/details/535971.sHtML
tv.blog.cvvliy.cn/Article/details/991117.sHtML
tv.blog.cvvliy.cn/Article/details/571175.sHtML
tv.blog.cvvliy.cn/Article/details/355173.sHtML
tv.blog.cvvliy.cn/Article/details/688440.sHtML
tv.blog.cvvliy.cn/Article/details/200248.sHtML
tv.blog.cvvliy.cn/Article/details/060028.sHtML
tv.blog.cvvliy.cn/Article/details/595737.sHtML
tv.blog.cvvliy.cn/Article/details/246480.sHtML
tv.blog.cvvliy.cn/Article/details/828880.sHtML
tv.blog.cvvliy.cn/Article/details/179759.sHtML
tv.blog.cvvliy.cn/Article/details/115753.sHtML
tv.blog.cvvliy.cn/Article/details/399597.sHtML
tv.blog.cvvliy.cn/Article/details/931795.sHtML
tv.blog.cvvliy.cn/Article/details/080244.sHtML
tv.blog.cvvliy.cn/Article/details/359133.sHtML
tv.blog.cvvliy.cn/Article/details/004866.sHtML
tv.blog.cvvliy.cn/Article/details/602008.sHtML
tv.blog.cvvliy.cn/Article/details/062480.sHtML
