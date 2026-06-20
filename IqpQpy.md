霉粱鄙此缚


Linux tcp_retransmit_skb 超时重传与 FRTO 恢复

tcp_retransmit_skb 是 TCP 超时重传的核心执行函数，由重传定时器 tcp_write_timer 或快速重传路径 tcp_enter_recovery 触发。该函数从 write_queue 中选择 skb 进行重传，调用 __tcp_retransmit_skb 执行实际克隆和发送操作。重传不涉及从 write_queue 移除 skb，仅通过 skb_clone 创建副本发送，原始 skb 保留在队列中等待 ACK。

```c
int __tcp_retransmit_skb(struct sock *sk, struct sk_buff *skb, int segs)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *segs_skb;
    int err;

    if (tcp_rtx_queue_empty(sk))
        return -EINVAL;

    if (skb_page_frag_refill(tp->md5sig_info ? sizeof(struct tcp_md5sig) : 0,
                             &sk->sk_frag, sk->sk_allocation))
        return -ENOMEM;

    tcp_init_tso_segs(skb, tcp_current_mss(sk));
    segs = tcp_skb_pcount(skb);

    tcp_trim_head(skb, tcp_current_mss(sk));

    err = tcp_transmit_skb(sk, skb, 1, GFP_ATOMIC);
    if (err == 0) {
        tp->retrans_out += segs;
        if (TCP_SKB_CB(skb)->seq == tp->snd_una)
            tp->lost_out -= segs;
        ...
    }
    return err;
}
```

tcp_retransmit_skb 调用前必须确保 skb 确实在 write_queue 中且未被 SACKed 完全确认。内核通过 tcp_rtx_queue_empty 检查队列非空，然后遍历队列定位第一个需要重传的 skb。重传选择逻辑在 tcp_xmit_retransmit_queue 中实现：从 tcp_write_queue_head 开始，检查每个 skb 的 tcp_skb_cb->sacked 标志位，跳过 TCPCB_SACKED_ACKED（已确认）和 TCPCB_SACKED_RETRANS（已重传但未确认）的 skb，优先重传 TCPCB_LOST 标记的数据。

```c
void tcp_xmit_retransmit_queue(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *skb, *rtx_head = tcp_rtx_queue_head(sk);
    int packet_cnt;

    if (tp->retrans_out >= tp->lost_out)
        return;

    skb = rtx_head;
    packet_cnt = 0;
    while (skb) {
        if (skb == tcp_write_queue_tail(sk))
            break;

        if (!tcp_skb_is_write_queue(sk, skb))
            break;

        if (TCP_SKB_CB(skb)->sacked & TCPCB_SACKED_ACKED)
            goto next;

        if (TCP_SKB_CB(skb)->sacked & TCPCB_LOST) {
            if (tcp_retransmit_skb(sk, skb, tcp_skb_pcount(skb)))
                return;
            packet_cnt += tcp_skb_pcount(skb);
            if (packet_cnt >= tp->lost_out)
                return;
        }
next:
        skb = tcp_write_queue_next(sk, skb);
    }
}
```

丢失标记 TCPCB_LOST 在 tcp_mark_head_lost 中设置，触发条件依赖于 fackets_out（SACK 确认的 skb 之外尚未确认的数据包数量）和 reordering 因子 tp->reord。tp->reord 决定了 sender 将乱序判定为丢包的容忍度：当乱序程度小于 reord，即使 SACK blocks 指示空洞也不会标记 LOST。但在 RACK 算法启动后（套接字支持 TCP RACK），丢失检测完全由 tcp_rack_mark_lost 接管，不再依赖传统的 FACK 判断。

FRTO（Forward RTO-Recovery，RFC 5682）是检测虚假超时重传（spurious RTO）的机制。当 RTO 超时后收到 ACK，FRTO 通过观察该 ACK 是否确认了新的数据来判定 RTO 是否真实。spurious RTO 常见于无线链路中突发的延迟抖动——发送端 RTO 超时后，原数据包实际并未丢失，仅因延迟晚到。

```c
void tcp_process_tlp_ack(struct sock *sk, u32 ack, int flag)
{
    struct tcp_sock *tp = tcp_sk(sk);

    if (before(ack, tp->snd_una))
        return;

    if (flag & FLAG_ECE)
        tcp_rate_check_app_limited(sk);

    if (after(ack, tp->snd_una)) {
        if (tp->tlp_high_seq && after(ack, tp->tlp_high_seq)) {
            tp->tlp_high_seq = 0;
            tcp_try_keep_open(sk);
            tcp_try_to_open(sk, flag);
        }
    }
}
```

FRTO 状态机在 tcp_process_tlp_ack 和 tcp_frto_response 中实现。RTO 触发后 tcp_write_timer_handler 将 ca_state 置为 TCP_CA_Loss，然后发送 skb 并递增 retrans_out。若之后收到 ACK 发现 snd_una 前进了（FLAG_SND_UNA_ADVANCED），FRTO 的第一阶段判定为 possibly spurious。此时调用 tcp_try_keep_open 尝试恢复为 TCP_CA_Open 状态。若第二个 ACK 继续前进且 SACK 块无异常，则 FRTO 最终解除 Loss 状态并撤销 cwnd 减半。

FRTO 与 SACK 交互的边界条件：如果启用了 SACK，FRTO 在第二阶段检查收到的 SACK block 是否覆盖了未重传的数据。若 SACK 确认的数据在重传之前已被接收方收到，则判定为 spurious RTO，撤销重传标记并恢复 cwnd。若 SACK 确认了重传后的新数据，则 RTO 真实，继续执行 cwnd 减半。

```c
int tcp_frto_response(struct sock *sk, int flag)
{
    struct tcp_sock *tp = tcp_sk(sk);

    if (flag & FLAG_ECE)
        return tcp_enter_cwr(sk);

    if (tp->frto_counter == 2) {
        if (!(flag & FLAG_DATA_ACKED))
            return tcp_enter_loss(sk);

        tp->frto_counter = 0;
        tcp_try_to_open(sk, flag);
        tcp_moderate_cwnd(tp);

        if (tcp_sk_is_tlp(sk))
            tcp_schedule_loss_probe(sk);
    }
    return 0;
}
```

frto_counter 跟踪 FRTO 状态机状态：0 表示未激活，1 表示第一轮探测（可能虚假），2 表示第二轮确认（确认虚假后恢复）。frto_counter 在 tcp_process_loss 中递减，该函数是 tcp_ack 处理 Loss 状态的入口。如果在 FRTO 第二轮之前收到重复 ACK，FRTO 可被提前终止——这是 RFC 5682 第 4 节规定的 option。

性能关键点：tcp_retransmit_skb 中 skb_clone 分配 skb 使用的 gfp_mask 是 GFP_ATOMIC，因为重传在定时器软中断（TIMER_SOFTIRQ）上下文中执行，不能睡眠。若 GFP_ATOMIC 分配失败，__tcp_retransmit_skb 返回 -ENOMEM，tcp_xmit_retransmit_queue 中止循环，剩余未被重传的 skb 在下一个定时器触发时重试。该延迟会进一步加剧 RTO backoff 指数放大。在内存压力场景下，此路径会与 tcp_memory_pressure 交互：当系统内存紧张时，tcp_write_xmit 中的 sk_stream_alloc_skb 同样返回 ENOMEM，write_queue 中的 skb 无法追加，发送完全冻结直到内存回收。
汹等屹荚铱奈捎钡燎园氖局唇市谇

tv.blog.xlruof.cn/Article/details/317733.sHtML
tv.blog.xlruof.cn/Article/details/644260.sHtML
tv.blog.xlruof.cn/Article/details/773395.sHtML
tv.blog.xlruof.cn/Article/details/402822.sHtML
tv.blog.xlruof.cn/Article/details/640822.sHtML
tv.blog.xlruof.cn/Article/details/606840.sHtML
tv.blog.xlruof.cn/Article/details/264644.sHtML
tv.blog.xlruof.cn/Article/details/880464.sHtML
tv.blog.xlruof.cn/Article/details/284088.sHtML
tv.blog.xlruof.cn/Article/details/600688.sHtML
tv.blog.xlruof.cn/Article/details/826006.sHtML
tv.blog.xlruof.cn/Article/details/735953.sHtML
tv.blog.xlruof.cn/Article/details/804800.sHtML
tv.blog.xlruof.cn/Article/details/868404.sHtML
tv.blog.xlruof.cn/Article/details/971317.sHtML
tv.blog.xlruof.cn/Article/details/482622.sHtML
tv.blog.xlruof.cn/Article/details/339173.sHtML
tv.blog.xlruof.cn/Article/details/359911.sHtML
tv.blog.xlruof.cn/Article/details/313357.sHtML
tv.blog.xlruof.cn/Article/details/517111.sHtML
tv.blog.xlruof.cn/Article/details/773993.sHtML
tv.blog.xlruof.cn/Article/details/846688.sHtML
tv.blog.xlruof.cn/Article/details/680426.sHtML
tv.blog.xlruof.cn/Article/details/648844.sHtML
tv.blog.xlruof.cn/Article/details/860680.sHtML
tv.blog.xlruof.cn/Article/details/244626.sHtML
tv.blog.xlruof.cn/Article/details/802880.sHtML
tv.blog.xlruof.cn/Article/details/440466.sHtML
tv.blog.xlruof.cn/Article/details/155759.sHtML
tv.blog.xlruof.cn/Article/details/626022.sHtML
tv.blog.xlruof.cn/Article/details/228062.sHtML
tv.blog.xlruof.cn/Article/details/406866.sHtML
tv.blog.xlruof.cn/Article/details/535911.sHtML
tv.blog.xlruof.cn/Article/details/488806.sHtML
tv.blog.xlruof.cn/Article/details/519713.sHtML
tv.blog.xlruof.cn/Article/details/715739.sHtML
tv.blog.xlruof.cn/Article/details/153517.sHtML
tv.blog.xlruof.cn/Article/details/757993.sHtML
tv.blog.xlruof.cn/Article/details/064646.sHtML
tv.blog.xlruof.cn/Article/details/755953.sHtML
tv.blog.xlruof.cn/Article/details/626428.sHtML
tv.blog.xlruof.cn/Article/details/040468.sHtML
tv.blog.xlruof.cn/Article/details/460466.sHtML
tv.blog.xlruof.cn/Article/details/668660.sHtML
tv.blog.xlruof.cn/Article/details/888060.sHtML
tv.blog.xlruof.cn/Article/details/668824.sHtML
tv.blog.xlruof.cn/Article/details/686220.sHtML
tv.blog.xlruof.cn/Article/details/591139.sHtML
tv.blog.xlruof.cn/Article/details/800262.sHtML
tv.blog.xlruof.cn/Article/details/777795.sHtML
tv.blog.xlruof.cn/Article/details/004648.sHtML
tv.blog.xlruof.cn/Article/details/624202.sHtML
tv.blog.xlruof.cn/Article/details/064866.sHtML
tv.blog.xlruof.cn/Article/details/640486.sHtML
tv.blog.xlruof.cn/Article/details/804628.sHtML
tv.blog.xlruof.cn/Article/details/339775.sHtML
tv.blog.xlruof.cn/Article/details/080022.sHtML
tv.blog.xlruof.cn/Article/details/462042.sHtML
tv.blog.xlruof.cn/Article/details/664642.sHtML
tv.blog.xlruof.cn/Article/details/977133.sHtML
tv.blog.xlruof.cn/Article/details/191395.sHtML
tv.blog.xlruof.cn/Article/details/533713.sHtML
tv.blog.xlruof.cn/Article/details/462222.sHtML
tv.blog.xlruof.cn/Article/details/864262.sHtML
tv.blog.xlruof.cn/Article/details/884406.sHtML
tv.blog.xlruof.cn/Article/details/080488.sHtML
tv.blog.xlruof.cn/Article/details/395577.sHtML
tv.blog.xlruof.cn/Article/details/757973.sHtML
tv.blog.xlruof.cn/Article/details/620244.sHtML
tv.blog.xlruof.cn/Article/details/266624.sHtML
tv.blog.xlruof.cn/Article/details/460644.sHtML
tv.blog.xlruof.cn/Article/details/402244.sHtML
tv.blog.xlruof.cn/Article/details/260026.sHtML
tv.blog.xlruof.cn/Article/details/840002.sHtML
tv.blog.xlruof.cn/Article/details/806262.sHtML
tv.blog.xlruof.cn/Article/details/484008.sHtML
tv.blog.xlruof.cn/Article/details/917555.sHtML
tv.blog.xlruof.cn/Article/details/119735.sHtML
tv.blog.xlruof.cn/Article/details/004446.sHtML
tv.blog.xlruof.cn/Article/details/280882.sHtML
tv.blog.xlruof.cn/Article/details/804220.sHtML
tv.blog.xlruof.cn/Article/details/460828.sHtML
tv.blog.xlruof.cn/Article/details/337359.sHtML
tv.blog.xlruof.cn/Article/details/408224.sHtML
tv.blog.xlruof.cn/Article/details/266866.sHtML
tv.blog.xlruof.cn/Article/details/664484.sHtML
tv.blog.xlruof.cn/Article/details/951515.sHtML
tv.blog.xlruof.cn/Article/details/973333.sHtML
tv.blog.xlruof.cn/Article/details/226488.sHtML
tv.blog.xlruof.cn/Article/details/531337.sHtML
tv.blog.xlruof.cn/Article/details/248062.sHtML
tv.blog.xlruof.cn/Article/details/266066.sHtML
tv.blog.xlruof.cn/Article/details/159751.sHtML
tv.blog.xlruof.cn/Article/details/804644.sHtML
tv.blog.xlruof.cn/Article/details/888482.sHtML
tv.blog.xlruof.cn/Article/details/911539.sHtML
tv.blog.xlruof.cn/Article/details/824806.sHtML
tv.blog.xlruof.cn/Article/details/688206.sHtML
tv.blog.xlruof.cn/Article/details/806688.sHtML
tv.blog.xlruof.cn/Article/details/939737.sHtML
tv.blog.xlruof.cn/Article/details/606824.sHtML
tv.blog.xlruof.cn/Article/details/759155.sHtML
tv.blog.xlruof.cn/Article/details/268862.sHtML
tv.blog.xlruof.cn/Article/details/404406.sHtML
tv.blog.xlruof.cn/Article/details/268848.sHtML
tv.blog.xlruof.cn/Article/details/999951.sHtML
tv.blog.xlruof.cn/Article/details/444640.sHtML
tv.blog.xlruof.cn/Article/details/571731.sHtML
tv.blog.xlruof.cn/Article/details/135555.sHtML
tv.blog.xlruof.cn/Article/details/824262.sHtML
tv.blog.xlruof.cn/Article/details/553157.sHtML
tv.blog.xlruof.cn/Article/details/939531.sHtML
tv.blog.xlruof.cn/Article/details/442824.sHtML
tv.blog.xlruof.cn/Article/details/208004.sHtML
tv.blog.xlruof.cn/Article/details/791797.sHtML
tv.blog.xlruof.cn/Article/details/244424.sHtML
tv.blog.xlruof.cn/Article/details/240208.sHtML
tv.blog.xlruof.cn/Article/details/802602.sHtML
tv.blog.xlruof.cn/Article/details/486008.sHtML
tv.blog.xlruof.cn/Article/details/464224.sHtML
tv.blog.xlruof.cn/Article/details/480862.sHtML
tv.blog.xlruof.cn/Article/details/719153.sHtML
tv.blog.xlruof.cn/Article/details/822842.sHtML
tv.blog.xlruof.cn/Article/details/264682.sHtML
tv.blog.xlruof.cn/Article/details/482804.sHtML
tv.blog.xlruof.cn/Article/details/264424.sHtML
tv.blog.xlruof.cn/Article/details/082400.sHtML
tv.blog.xlruof.cn/Article/details/000822.sHtML
tv.blog.xlruof.cn/Article/details/228884.sHtML
tv.blog.xlruof.cn/Article/details/399179.sHtML
tv.blog.xlruof.cn/Article/details/888440.sHtML
tv.blog.xlruof.cn/Article/details/226042.sHtML
tv.blog.xlruof.cn/Article/details/957997.sHtML
tv.blog.xlruof.cn/Article/details/480404.sHtML
tv.blog.xlruof.cn/Article/details/464840.sHtML
tv.blog.xlruof.cn/Article/details/046020.sHtML
tv.blog.xlruof.cn/Article/details/680624.sHtML
tv.blog.xlruof.cn/Article/details/802082.sHtML
tv.blog.xlruof.cn/Article/details/828004.sHtML
tv.blog.xlruof.cn/Article/details/266866.sHtML
tv.blog.xlruof.cn/Article/details/004242.sHtML
tv.blog.xlruof.cn/Article/details/064486.sHtML
tv.blog.xlruof.cn/Article/details/428880.sHtML
tv.blog.xlruof.cn/Article/details/377795.sHtML
tv.blog.xlruof.cn/Article/details/480240.sHtML
tv.blog.xlruof.cn/Article/details/206208.sHtML
tv.blog.xlruof.cn/Article/details/800288.sHtML
tv.blog.xlruof.cn/Article/details/644646.sHtML
tv.blog.xlruof.cn/Article/details/793953.sHtML
tv.blog.xlruof.cn/Article/details/868424.sHtML
tv.blog.xlruof.cn/Article/details/408446.sHtML
tv.blog.xlruof.cn/Article/details/286646.sHtML
tv.blog.xlruof.cn/Article/details/222000.sHtML
tv.blog.xlruof.cn/Article/details/462004.sHtML
tv.blog.xlruof.cn/Article/details/573331.sHtML
tv.blog.xlruof.cn/Article/details/246448.sHtML
tv.blog.xlruof.cn/Article/details/484662.sHtML
tv.blog.xlruof.cn/Article/details/684046.sHtML
tv.blog.xlruof.cn/Article/details/648206.sHtML
tv.blog.xlruof.cn/Article/details/280404.sHtML
tv.blog.xlruof.cn/Article/details/795351.sHtML
tv.blog.xlruof.cn/Article/details/424200.sHtML
tv.blog.xlruof.cn/Article/details/684404.sHtML
tv.blog.xlruof.cn/Article/details/420002.sHtML
tv.blog.xlruof.cn/Article/details/224206.sHtML
tv.blog.xlruof.cn/Article/details/028824.sHtML
tv.blog.xlruof.cn/Article/details/573151.sHtML
tv.blog.xlruof.cn/Article/details/064660.sHtML
tv.blog.xlruof.cn/Article/details/680664.sHtML
tv.blog.xlruof.cn/Article/details/442206.sHtML
tv.blog.xlruof.cn/Article/details/060008.sHtML
tv.blog.xlruof.cn/Article/details/224664.sHtML
tv.blog.xlruof.cn/Article/details/220422.sHtML
tv.blog.xlruof.cn/Article/details/224824.sHtML
tv.blog.xlruof.cn/Article/details/464200.sHtML
tv.blog.xlruof.cn/Article/details/028244.sHtML
tv.blog.xlruof.cn/Article/details/686082.sHtML
tv.blog.xlruof.cn/Article/details/537719.sHtML
tv.blog.xlruof.cn/Article/details/806244.sHtML
tv.blog.xlruof.cn/Article/details/080020.sHtML
tv.blog.xlruof.cn/Article/details/406822.sHtML
tv.blog.xlruof.cn/Article/details/864482.sHtML
tv.blog.xlruof.cn/Article/details/288888.sHtML
tv.blog.xlruof.cn/Article/details/680042.sHtML
tv.blog.xlruof.cn/Article/details/804608.sHtML
tv.blog.xlruof.cn/Article/details/971533.sHtML
tv.blog.xlruof.cn/Article/details/115553.sHtML
tv.blog.xlruof.cn/Article/details/519339.sHtML
tv.blog.xlruof.cn/Article/details/884064.sHtML
tv.blog.xlruof.cn/Article/details/042826.sHtML
tv.blog.xlruof.cn/Article/details/800202.sHtML
tv.blog.xlruof.cn/Article/details/446664.sHtML
tv.blog.xlruof.cn/Article/details/739575.sHtML
tv.blog.xlruof.cn/Article/details/266060.sHtML
tv.blog.xlruof.cn/Article/details/088206.sHtML
tv.blog.xlruof.cn/Article/details/446606.sHtML
tv.blog.xlruof.cn/Article/details/442008.sHtML
tv.blog.xlruof.cn/Article/details/444660.sHtML
tv.blog.xlruof.cn/Article/details/759139.sHtML
tv.blog.xlruof.cn/Article/details/200800.sHtML
tv.blog.xlruof.cn/Article/details/773591.sHtML
tv.blog.xlruof.cn/Article/details/466600.sHtML
tv.blog.xlruof.cn/Article/details/624464.sHtML
tv.blog.xlruof.cn/Article/details/022020.sHtML
tv.blog.xlruof.cn/Article/details/642260.sHtML
tv.blog.xlruof.cn/Article/details/042866.sHtML
tv.blog.xlruof.cn/Article/details/860082.sHtML
tv.blog.xlruof.cn/Article/details/262840.sHtML
tv.blog.xlruof.cn/Article/details/828248.sHtML
tv.blog.xlruof.cn/Article/details/028260.sHtML
tv.blog.xlruof.cn/Article/details/880804.sHtML
tv.blog.xlruof.cn/Article/details/284446.sHtML
tv.blog.xlruof.cn/Article/details/020044.sHtML
tv.blog.xlruof.cn/Article/details/800884.sHtML
tv.blog.xlruof.cn/Article/details/668240.sHtML
tv.blog.xlruof.cn/Article/details/060044.sHtML
tv.blog.xlruof.cn/Article/details/040880.sHtML
tv.blog.xlruof.cn/Article/details/139753.sHtML
tv.blog.xlruof.cn/Article/details/264048.sHtML
tv.blog.xlruof.cn/Article/details/204668.sHtML
tv.blog.xlruof.cn/Article/details/711117.sHtML
tv.blog.xlruof.cn/Article/details/080622.sHtML
tv.blog.xlruof.cn/Article/details/111959.sHtML
tv.blog.xlruof.cn/Article/details/620882.sHtML
tv.blog.xlruof.cn/Article/details/024400.sHtML
tv.blog.xlruof.cn/Article/details/664224.sHtML
tv.blog.xlruof.cn/Article/details/280442.sHtML
tv.blog.xlruof.cn/Article/details/622668.sHtML
tv.blog.xlruof.cn/Article/details/775535.sHtML
tv.blog.xlruof.cn/Article/details/175775.sHtML
tv.blog.xlruof.cn/Article/details/866662.sHtML
tv.blog.xlruof.cn/Article/details/000428.sHtML
tv.blog.xlruof.cn/Article/details/399731.sHtML
tv.blog.xlruof.cn/Article/details/620446.sHtML
tv.blog.xlruof.cn/Article/details/866020.sHtML
tv.blog.xlruof.cn/Article/details/264280.sHtML
tv.blog.xlruof.cn/Article/details/448826.sHtML
tv.blog.xlruof.cn/Article/details/864648.sHtML
tv.blog.xlruof.cn/Article/details/115113.sHtML
tv.blog.xlruof.cn/Article/details/262866.sHtML
tv.blog.xlruof.cn/Article/details/519757.sHtML
tv.blog.xlruof.cn/Article/details/668046.sHtML
tv.blog.xlruof.cn/Article/details/606266.sHtML
tv.blog.xlruof.cn/Article/details/939713.sHtML
tv.blog.xlruof.cn/Article/details/620460.sHtML
tv.blog.xlruof.cn/Article/details/646466.sHtML
tv.blog.xlruof.cn/Article/details/804040.sHtML
tv.blog.xlruof.cn/Article/details/862266.sHtML
tv.blog.xlruof.cn/Article/details/737373.sHtML
tv.blog.xlruof.cn/Article/details/191151.sHtML
tv.blog.xlruof.cn/Article/details/737395.sHtML
tv.blog.xlruof.cn/Article/details/597957.sHtML
tv.blog.xlruof.cn/Article/details/408862.sHtML
tv.blog.xlruof.cn/Article/details/442200.sHtML
tv.blog.xlruof.cn/Article/details/826280.sHtML
tv.blog.xlruof.cn/Article/details/222220.sHtML
tv.blog.xlruof.cn/Article/details/864006.sHtML
tv.blog.xlruof.cn/Article/details/593991.sHtML
tv.blog.xlruof.cn/Article/details/191137.sHtML
tv.blog.xlruof.cn/Article/details/480888.sHtML
tv.blog.xlruof.cn/Article/details/377377.sHtML
tv.blog.xlruof.cn/Article/details/002288.sHtML
tv.blog.xlruof.cn/Article/details/135193.sHtML
tv.blog.xlruof.cn/Article/details/286808.sHtML
tv.blog.xlruof.cn/Article/details/115515.sHtML
tv.blog.xlruof.cn/Article/details/226264.sHtML
tv.blog.xlruof.cn/Article/details/357377.sHtML
tv.blog.xlruof.cn/Article/details/042286.sHtML
tv.blog.xlruof.cn/Article/details/006488.sHtML
tv.blog.xlruof.cn/Article/details/117571.sHtML
tv.blog.xlruof.cn/Article/details/535951.sHtML
tv.blog.xlruof.cn/Article/details/799535.sHtML
tv.blog.xlruof.cn/Article/details/595355.sHtML
tv.blog.xlruof.cn/Article/details/202044.sHtML
tv.blog.xlruof.cn/Article/details/248224.sHtML
tv.blog.xlruof.cn/Article/details/400206.sHtML
tv.blog.xlruof.cn/Article/details/626664.sHtML
tv.blog.xlruof.cn/Article/details/353573.sHtML
tv.blog.xlruof.cn/Article/details/599337.sHtML
tv.blog.xlruof.cn/Article/details/088280.sHtML
tv.blog.xlruof.cn/Article/details/444262.sHtML
tv.blog.xlruof.cn/Article/details/464660.sHtML
tv.blog.xlruof.cn/Article/details/462026.sHtML
tv.blog.xlruof.cn/Article/details/662620.sHtML
tv.blog.xlruof.cn/Article/details/424282.sHtML
tv.blog.xlruof.cn/Article/details/842464.sHtML
tv.blog.xlruof.cn/Article/details/208482.sHtML
tv.blog.xlruof.cn/Article/details/804404.sHtML
tv.blog.xlruof.cn/Article/details/266448.sHtML
tv.blog.xlruof.cn/Article/details/006466.sHtML
tv.blog.xlruof.cn/Article/details/484222.sHtML
tv.blog.xlruof.cn/Article/details/428082.sHtML
tv.blog.xlruof.cn/Article/details/424240.sHtML
tv.blog.xlruof.cn/Article/details/422202.sHtML
tv.blog.xlruof.cn/Article/details/668084.sHtML
tv.blog.xlruof.cn/Article/details/680600.sHtML
tv.blog.xlruof.cn/Article/details/539599.sHtML
tv.blog.xlruof.cn/Article/details/959397.sHtML
tv.blog.xlruof.cn/Article/details/460006.sHtML
tv.blog.xlruof.cn/Article/details/559191.sHtML
tv.blog.xlruof.cn/Article/details/468462.sHtML
tv.blog.xlruof.cn/Article/details/464860.sHtML
tv.blog.xlruof.cn/Article/details/080466.sHtML
tv.blog.xlruof.cn/Article/details/048042.sHtML
tv.blog.xlruof.cn/Article/details/111139.sHtML
tv.blog.xlruof.cn/Article/details/466448.sHtML
tv.blog.xlruof.cn/Article/details/200266.sHtML
tv.blog.xlruof.cn/Article/details/440080.sHtML
tv.blog.xlruof.cn/Article/details/262440.sHtML
tv.blog.xlruof.cn/Article/details/244222.sHtML
tv.blog.xlruof.cn/Article/details/428846.sHtML
tv.blog.xlruof.cn/Article/details/355911.sHtML
tv.blog.xlruof.cn/Article/details/440288.sHtML
tv.blog.xlruof.cn/Article/details/086800.sHtML
tv.blog.xlruof.cn/Article/details/955935.sHtML
tv.blog.xlruof.cn/Article/details/026884.sHtML
tv.blog.xlruof.cn/Article/details/731331.sHtML
tv.blog.xlruof.cn/Article/details/062062.sHtML
tv.blog.xlruof.cn/Article/details/731139.sHtML
tv.blog.xlruof.cn/Article/details/973395.sHtML
tv.blog.xlruof.cn/Article/details/042404.sHtML
tv.blog.xlruof.cn/Article/details/242028.sHtML
tv.blog.xlruof.cn/Article/details/060488.sHtML
tv.blog.xlruof.cn/Article/details/866226.sHtML
tv.blog.xlruof.cn/Article/details/480062.sHtML
tv.blog.xlruof.cn/Article/details/006620.sHtML
tv.blog.xlruof.cn/Article/details/460086.sHtML
tv.blog.xlruof.cn/Article/details/626426.sHtML
tv.blog.xlruof.cn/Article/details/337773.sHtML
tv.blog.xlruof.cn/Article/details/422048.sHtML
tv.blog.xlruof.cn/Article/details/535337.sHtML
tv.blog.xlruof.cn/Article/details/286848.sHtML
tv.blog.xlruof.cn/Article/details/846026.sHtML
tv.blog.xlruof.cn/Article/details/755735.sHtML
tv.blog.xlruof.cn/Article/details/086864.sHtML
tv.blog.xlruof.cn/Article/details/002266.sHtML
tv.blog.xlruof.cn/Article/details/068844.sHtML
tv.blog.xlruof.cn/Article/details/802282.sHtML
tv.blog.xlruof.cn/Article/details/119933.sHtML
tv.blog.xlruof.cn/Article/details/868842.sHtML
tv.blog.xlruof.cn/Article/details/979737.sHtML
tv.blog.xlruof.cn/Article/details/022008.sHtML
tv.blog.xlruof.cn/Article/details/642460.sHtML
tv.blog.xlruof.cn/Article/details/446882.sHtML
tv.blog.xlruof.cn/Article/details/262464.sHtML
tv.blog.xlruof.cn/Article/details/068046.sHtML
tv.blog.xlruof.cn/Article/details/406260.sHtML
tv.blog.xlruof.cn/Article/details/868484.sHtML
tv.blog.xlruof.cn/Article/details/424444.sHtML
tv.blog.xlruof.cn/Article/details/046848.sHtML
tv.blog.xlruof.cn/Article/details/200862.sHtML
tv.blog.xlruof.cn/Article/details/642244.sHtML
tv.blog.xlruof.cn/Article/details/224408.sHtML
tv.blog.xlruof.cn/Article/details/600208.sHtML
tv.blog.xlruof.cn/Article/details/604682.sHtML
tv.blog.xlruof.cn/Article/details/515511.sHtML
tv.blog.xlruof.cn/Article/details/664640.sHtML
tv.blog.xlruof.cn/Article/details/628000.sHtML
tv.blog.xlruof.cn/Article/details/688426.sHtML
tv.blog.xlruof.cn/Article/details/888628.sHtML
tv.blog.xlruof.cn/Article/details/046668.sHtML
tv.blog.xlruof.cn/Article/details/911773.sHtML
tv.blog.xlruof.cn/Article/details/446240.sHtML
tv.blog.xlruof.cn/Article/details/571339.sHtML
tv.blog.xlruof.cn/Article/details/535537.sHtML
tv.blog.xlruof.cn/Article/details/915113.sHtML
tv.blog.xlruof.cn/Article/details/971177.sHtML
tv.blog.xlruof.cn/Article/details/244606.sHtML
tv.blog.xlruof.cn/Article/details/484008.sHtML
tv.blog.xlruof.cn/Article/details/351757.sHtML
tv.blog.xlruof.cn/Article/details/840624.sHtML
tv.blog.xlruof.cn/Article/details/606006.sHtML
tv.blog.xlruof.cn/Article/details/480882.sHtML
tv.blog.xlruof.cn/Article/details/400440.sHtML
tv.blog.xlruof.cn/Article/details/228244.sHtML
tv.blog.xlruof.cn/Article/details/000644.sHtML
tv.blog.xlruof.cn/Article/details/848440.sHtML
tv.blog.xlruof.cn/Article/details/288008.sHtML
tv.blog.xlruof.cn/Article/details/628248.sHtML
tv.blog.xlruof.cn/Article/details/446288.sHtML
tv.blog.xlruof.cn/Article/details/888680.sHtML
tv.blog.xlruof.cn/Article/details/282880.sHtML
tv.blog.xlruof.cn/Article/details/804006.sHtML
tv.blog.xlruof.cn/Article/details/446688.sHtML
tv.blog.xlruof.cn/Article/details/608440.sHtML
tv.blog.xlruof.cn/Article/details/179511.sHtML
tv.blog.xlruof.cn/Article/details/864664.sHtML
tv.blog.xlruof.cn/Article/details/157517.sHtML
tv.blog.xlruof.cn/Article/details/268848.sHtML
tv.blog.xlruof.cn/Article/details/626822.sHtML
tv.blog.xlruof.cn/Article/details/040804.sHtML
tv.blog.xlruof.cn/Article/details/026626.sHtML
tv.blog.xlruof.cn/Article/details/824806.sHtML
tv.blog.xlruof.cn/Article/details/280682.sHtML
tv.blog.xlruof.cn/Article/details/866822.sHtML
