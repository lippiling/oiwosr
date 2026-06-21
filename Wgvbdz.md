掠未殴沃赫


Linux udp_sendmsg数据报分片与cork暂存逻辑

udp_sendmsg是UDP数据报发送的核心入口，路径上涉及两个关键机制：数据传输层聚合（UFO/GSO）和UDP cork暂存。这两个机制直接决定了skb的构建方式。

一、 UDP cork暂存机制

当用户通过setsockopt(IP_CORK)开启cork模式，或连续多次sendmsg未触发立即发送时，UDP cork机制将多个sendmsg调用产生的数据暂存在socket的 cork 结构中，直到条件满足才一次性flush。

内核用 struct cork 管理暂存状态，定义在 include/net/ip.h：

struct ipcm_cookie {
    struct sockc_cookie sockc;
    __be32          addr;
    int             oif;
    struct ip_options_rcu *opt;
    __u8            tx_flags;
    __u8            ttl;
    __s16           tos;
    __u16           gso_size;
};

struct udp_cork {
    struct ipcm_cookie ipc;
    struct rtable     *rt;
    int               temp;
    int               gso_size;
    int               corkflag;
};

在udp_sendmsg中，当corkflag为1且已有待发数据时，新到达的数据被追加到 cork->gso_size 指定的segment中。核心逻辑位于 net/ipv4/udp.c 的 udp_sendmsg：

if (up->pending) {
    struct msghdr msg = { .msg_iov = iov, .msg_iovlen = iovlen };
    int err;

    if (flags & MSG_CONFIRM)
        goto do_append_data;
    corkreq = up->corkflag && !(flags & MSG_MORE);
    if (!corkreq)
        goto do_append_data;
}

这里的 up->pending 检查socket是否有尚未flush的cork数据。若 pending 为真，表示skb正在构建中，本次数据被追加到已有的skb frag列表中。

二、 数据分片与UFO/GSO

UDP分片的核心在 ip_append_data 中完成。该函数位于 net/ipv4/ip_output.c：

int __ip_append_data(struct sock *sk, struct flowi4 *fl4,
                     struct sk_buff_head *queue, struct inet_cork *cork,
                     struct page_frag *pfrag, int getfrag,
                     void *from, int length, int transhdrlen,
                     unsigned int flags)
{
    unsigned int maxfraglen, fragheaderlen, maxnonfragsize;
    struct sk_buff *skb;
    int hh_len;
    int exthdrlen;
    int copy;

    /* 计算MTU限制下的最大数据长度 */
    mtu = cork->gso_size ?: ip_sk_use_pmtu(sk);
    if (cork->gso_size) {
        /* GSO/UFO场景：单次可以发送超过MTU的数据 */
        hh_len = (rt->dst.header_len + sizeof(struct iphdr));
        fragheaderlen = sizeof(struct iphdr);
        maxfraglen = ((mtu - fragheaderlen) & ~7);
    } else {
        /* 非GSO场景：每次只能发送不超过MTU的skb */
        skb = skb_peek_tail(queue);
        if (skb && (skb->len + length > mtu)) {
            /* 当前skb已经写满，需要分配新skb */
            skb = NULL;
        }
    }

    /* 遍历用户数据，按MTU边界分片 */
    while (length > 0) {
        /* 检查是否需要分配新skb */
        if ((skb == NULL) || (skb_tailroom(skb) < copy)) {
            if (skb)
                cork->gso_segs++;
            skb = skb_alloc_new(queue, &cork, hh_len, headersize,
                                mtu, flags | (softirq ? GFP_ATOMIC : 0));
            if (unlikely(skb == NULL))
                goto error;
        }

        /* 从用户空间拷贝数据到skb */
        err = skb_copy_to_page(sk, &cork->page, skb, pfrag, copy,
                               &skb_shinfo(skb)->frags[skb_shinfo(skb)->nr_frags]);
        if (err)
            goto error;

        skb->len += copy;
        skb->data_len += copy;
        skb->truesize += copy;

        length -= copy;
        from += copy;
    }
}

关键点在于 maxfraglen 的计算。由于IPv4头部长度为20字节（不含选项），每个分片的数据部分必须是8字节对齐的。UFO场景下 cork->gso_size 指定了每个GSO段的大小，内核构造一个超长skb，由网卡硬件完成分片。

三、 cork状态的flush触发

cork数据的发送时机有两种：
1. 用户调用sendmsg时flags带有MSG_MORE且corkflag关闭，直接发送
2. 下一次sendmsg调用检测到新数据与cork中的数据不属于同一流（目标地址、oif等发生变化），flush旧cork

flush动作由 udp_send_skb 或 ip_push_pending_frames 完成：

static void udp_flush_pending_frames(struct sock *sk)
{
    struct udp_sock *up = udp_sk(sk);

    if (up->pending) {
        up->len = 0;
        up->pending = 0;
        ip_flush_pending_frames(sk);
    }
}

ip_flush_pending_frames 最终调用 __ip_flush_pending_frames，将 cork 中累积的所有skb排入队列，交由 IP 层的输出路径处理。

四、 GSO分段回退

当UFO/GSO不可用（网卡不支持或配置关闭），ip_append_data 退化为逐分片发送模式。每个skb的数据量不超过 mtud - ip头长度，并且连续发送多个IP分片。此时内核不依赖硬件，完全由软件分片。

int ip_setup_cork(struct sock *sk, struct inet_cork *cork,
                  struct ipcm_cookie *ipc, struct rtable **rtp)
{
    struct ip_options_rcu *opt;
    struct rtable *rt;

    rt = *rtp;
    if (unlikely(!rt))
        return -EFAULT;

    cork->fragsize = ip_sk_use_pmtu(sk) ? dst_mtu(&rt->dst) : mtu;
    cork->gso_size = ip_sk_use_pmtu(sk) ? 0 : ipc->gso_size;
    cork->addr = ipc->addr;
    cork->opt = ipc->opt;
    cork->ttl = ipc->ttl;
    cork->tos = ipc->tos;
    cork->priority = ipc->priority;
    cork->tx_flags = ipc->tx_flags;
    cork->oif = ipc->oif;

    *rtp = NULL;
    return 0;
}

cork->gso_size 字段是打通UDP与GSO的关键桥梁。当应用程序使用sendmmsg批量发送时，内核将多个消息合并为一个GSO段，显著提高发送吞吐量。

五、 校验和计算与cork交互

UDP数据报的校验和计算在 cork flush 时完成。udp_send_skb 在 skb 即将发送前计算校验和：

static int udp_send_skb(struct sk_buff *skb, struct flowi4 *fl4,
                        struct inet_cork *cork)
{
    struct sock *sk = skb->sk;
    struct udphdr *uh;
    int err;

    /* 构建UDP头 */
    uh = udp_hdr(skb);
    uh->source = inet_sk(sk)->inet_sport;
    uh->dest = fl4->fl4_dport;
    uh->len = htons(skb->len);
    uh->check = 0;

    if (sk->sk_no_check_tx)
        return 0;

    /* 计算UDP校验和 */
    if (skb->ip_summed != CHECKSUM_PARTIAL || dst_allfrag(skb_dst(skb))) {
        udp4_hwcsum(skb, fl4->saddr, fl4->daddr);
    } else {
        skb->csum = 0;
        /* 软件计算校验和 */
        err = skb_checksum_help(skb);
    }

    return 0;
}

在cork场景下，所有数据片段都累积在skb的frag_list中，udp_send_skb遍历frag_list计算整个数据报的校验和，确保即使分片也能正确计算。

总结上述流程，udp_sendmsg在处理大数据报时的核心路径为：用户数据 -> ip_append_data(分片/cork暂存) -> cork flush -> udp_send_skb(校验和) -> ip_local_out -> 邻居子系统 -> qdisc入队。cork机制通过在用户态与内核态之间减少上下文切换次数，提升了小数据报批量发送时的性能；而GSO/UFO则通过减少skb数量，降低了协议栈处理开销。
谠欣补胸尉拐寥谰禾壮欠时裳徒卣

m.qxs.mmmfb.cn/86426.Doc
m.qxs.mmmfb.cn/48680.Doc
m.qxs.mmmfb.cn/24624.Doc
m.qxs.mmmfb.cn/02880.Doc
m.qxs.mmmfb.cn/11551.Doc
m.qxs.mmmfb.cn/44660.Doc
m.qxs.mmmfb.cn/48466.Doc
m.qxa.mmmfb.cn/64680.Doc
m.qxa.mmmfb.cn/44062.Doc
m.qxa.mmmfb.cn/22064.Doc
m.qxa.mmmfb.cn/37117.Doc
m.qxa.mmmfb.cn/44484.Doc
m.qxa.mmmfb.cn/44268.Doc
m.qxa.mmmfb.cn/60664.Doc
m.qxa.mmmfb.cn/55515.Doc
m.qxa.mmmfb.cn/22048.Doc
m.qxa.mmmfb.cn/28008.Doc
m.qxa.mmmfb.cn/86806.Doc
m.qxa.mmmfb.cn/62866.Doc
m.qxa.mmmfb.cn/08664.Doc
m.qxa.mmmfb.cn/64026.Doc
m.qxa.mmmfb.cn/75759.Doc
m.qxa.mmmfb.cn/06060.Doc
m.qxa.mmmfb.cn/00200.Doc
m.qxa.mmmfb.cn/86280.Doc
m.qxa.mmmfb.cn/08860.Doc
m.qxa.mmmfb.cn/35377.Doc
m.qxp.mmmfb.cn/99339.Doc
m.qxp.mmmfb.cn/46488.Doc
m.qxp.mmmfb.cn/28284.Doc
m.qxp.mmmfb.cn/46840.Doc
m.qxp.mmmfb.cn/02804.Doc
m.qxp.mmmfb.cn/86802.Doc
m.qxp.mmmfb.cn/84464.Doc
m.qxp.mmmfb.cn/17539.Doc
m.qxp.mmmfb.cn/44084.Doc
m.qxp.mmmfb.cn/64068.Doc
m.qxp.mmmfb.cn/44646.Doc
m.qxp.mmmfb.cn/71793.Doc
m.qxp.mmmfb.cn/06222.Doc
m.qxp.mmmfb.cn/20688.Doc
m.qxp.mmmfb.cn/26460.Doc
m.qxp.mmmfb.cn/86246.Doc
m.qxp.mmmfb.cn/33179.Doc
m.qxp.mmmfb.cn/04620.Doc
m.qxp.mmmfb.cn/22268.Doc
m.qxp.mmmfb.cn/00280.Doc
m.qxo.mmmfb.cn/40404.Doc
m.qxo.mmmfb.cn/86800.Doc
m.qxo.mmmfb.cn/57171.Doc
m.qxo.mmmfb.cn/15579.Doc
m.qxo.mmmfb.cn/66862.Doc
m.qxo.mmmfb.cn/24806.Doc
m.qxo.mmmfb.cn/39591.Doc
m.qxo.mmmfb.cn/24640.Doc
m.qxo.mmmfb.cn/22220.Doc
m.qxo.mmmfb.cn/20824.Doc
m.qxo.mmmfb.cn/82648.Doc
m.qxo.mmmfb.cn/04206.Doc
m.qxo.mmmfb.cn/84880.Doc
m.qxo.mmmfb.cn/26442.Doc
m.qxo.mmmfb.cn/06024.Doc
m.qxo.mmmfb.cn/42804.Doc
m.qxo.mmmfb.cn/04468.Doc
m.qxo.mmmfb.cn/73777.Doc
m.qxo.mmmfb.cn/19739.Doc
m.qxo.mmmfb.cn/33751.Doc
m.qxi.mmmfb.cn/04428.Doc
m.qxi.mmmfb.cn/99795.Doc
m.qxi.mmmfb.cn/06448.Doc
m.qxi.mmmfb.cn/59399.Doc
m.qxi.mmmfb.cn/97719.Doc
m.qxi.mmmfb.cn/08846.Doc
m.qxi.mmmfb.cn/88266.Doc
m.qxi.mmmfb.cn/31575.Doc
m.qxi.mmmfb.cn/46484.Doc
m.qxi.mmmfb.cn/42084.Doc
m.qxi.mmmfb.cn/82060.Doc
m.qxi.mmmfb.cn/84484.Doc
m.qxi.mmmfb.cn/97573.Doc
m.qxi.mmmfb.cn/11953.Doc
m.qxi.mmmfb.cn/75151.Doc
m.qxi.mmmfb.cn/04442.Doc
m.qxi.mmmfb.cn/19159.Doc
m.qxi.mmmfb.cn/17519.Doc
m.qxi.mmmfb.cn/46480.Doc
m.qxi.mmmfb.cn/93135.Doc
m.qxu.mmmfb.cn/71395.Doc
m.qxu.mmmfb.cn/00868.Doc
m.qxu.mmmfb.cn/46204.Doc
m.qxu.mmmfb.cn/17997.Doc
m.qxu.mmmfb.cn/59139.Doc
m.qxu.mmmfb.cn/66068.Doc
m.qxu.mmmfb.cn/20644.Doc
m.qxu.mmmfb.cn/48682.Doc
m.qxu.mmmfb.cn/24464.Doc
m.qxu.mmmfb.cn/80062.Doc
m.qxu.mmmfb.cn/82480.Doc
m.qxu.mmmfb.cn/26640.Doc
m.qxu.mmmfb.cn/40204.Doc
m.qxu.mmmfb.cn/77717.Doc
m.qxu.mmmfb.cn/42268.Doc
m.qxu.mmmfb.cn/88060.Doc
m.qxu.mmmfb.cn/24824.Doc
m.qxu.mmmfb.cn/51119.Doc
m.qxu.mmmfb.cn/19719.Doc
m.qxu.mmmfb.cn/28604.Doc
m.qxy.mmmfb.cn/64666.Doc
m.qxy.mmmfb.cn/40684.Doc
m.qxy.mmmfb.cn/64664.Doc
m.qxy.mmmfb.cn/82464.Doc
m.qxy.mmmfb.cn/31737.Doc
m.qxy.mmmfb.cn/37911.Doc
m.qxy.mmmfb.cn/00660.Doc
m.qxy.mmmfb.cn/08682.Doc
m.qxy.mmmfb.cn/33913.Doc
m.qxy.mmmfb.cn/84428.Doc
m.qxy.mmmfb.cn/86046.Doc
m.qxy.mmmfb.cn/02240.Doc
m.qxy.mmmfb.cn/40486.Doc
m.qxy.mmmfb.cn/00006.Doc
m.qxy.mmmfb.cn/04802.Doc
m.qxy.mmmfb.cn/04662.Doc
m.qxy.mmmfb.cn/68604.Doc
m.qxy.mmmfb.cn/48222.Doc
m.qxy.mmmfb.cn/02464.Doc
m.qxy.mmmfb.cn/24286.Doc
m.qxt.mmmfb.cn/37759.Doc
m.qxt.mmmfb.cn/46882.Doc
m.qxt.mmmfb.cn/64226.Doc
m.qxt.mmmfb.cn/26066.Doc
m.qxt.mmmfb.cn/68286.Doc
m.qxt.mmmfb.cn/46226.Doc
m.qxt.mmmfb.cn/84206.Doc
m.qxt.mmmfb.cn/44602.Doc
m.qxt.mmmfb.cn/44668.Doc
m.qxt.mmmfb.cn/68288.Doc
m.qxt.mmmfb.cn/08668.Doc
m.qxt.mmmfb.cn/20442.Doc
m.qxt.mmmfb.cn/79359.Doc
m.qxt.mmmfb.cn/60040.Doc
m.qxt.mmmfb.cn/26804.Doc
m.qxt.mmmfb.cn/48484.Doc
m.qxt.mmmfb.cn/84626.Doc
m.qxt.mmmfb.cn/44280.Doc
m.qxt.mmmfb.cn/00802.Doc
m.qxt.mmmfb.cn/60282.Doc
m.qxr.mmmfb.cn/60242.Doc
m.qxr.mmmfb.cn/64264.Doc
m.qxr.mmmfb.cn/60602.Doc
m.qxr.mmmfb.cn/20040.Doc
m.qxr.mmmfb.cn/35175.Doc
m.qxr.mmmfb.cn/44044.Doc
m.qxr.mmmfb.cn/68200.Doc
m.qxr.mmmfb.cn/80044.Doc
m.qxr.mmmfb.cn/60848.Doc
m.qxr.mmmfb.cn/75731.Doc
m.qxr.mmmfb.cn/62206.Doc
m.qxr.mmmfb.cn/64646.Doc
m.qxr.mmmfb.cn/04884.Doc
m.qxr.mmmfb.cn/86824.Doc
m.qxr.mmmfb.cn/02088.Doc
m.qxr.mmmfb.cn/79999.Doc
m.qxr.mmmfb.cn/48220.Doc
m.qxr.mmmfb.cn/66228.Doc
m.qxr.mmmfb.cn/48840.Doc
m.qxr.mmmfb.cn/17539.Doc
m.qxe.qtbzn.cn/04460.Doc
m.qxe.qtbzn.cn/26484.Doc
m.qxe.qtbzn.cn/60266.Doc
m.qxe.qtbzn.cn/02400.Doc
m.qxe.qtbzn.cn/06444.Doc
m.qxe.qtbzn.cn/88000.Doc
m.qxe.qtbzn.cn/42804.Doc
m.qxe.qtbzn.cn/04204.Doc
m.qxe.qtbzn.cn/73175.Doc
m.qxe.qtbzn.cn/40688.Doc
m.qxe.qtbzn.cn/93751.Doc
m.qxe.qtbzn.cn/37119.Doc
m.qxe.qtbzn.cn/48462.Doc
m.qxe.qtbzn.cn/24642.Doc
m.qxe.qtbzn.cn/60884.Doc
m.qxe.qtbzn.cn/42666.Doc
m.qxe.qtbzn.cn/84844.Doc
m.qxe.qtbzn.cn/44840.Doc
m.qxe.qtbzn.cn/57571.Doc
m.qxe.qtbzn.cn/84264.Doc
m.qxw.qtbzn.cn/19779.Doc
m.qxw.qtbzn.cn/99557.Doc
m.qxw.qtbzn.cn/46226.Doc
m.qxw.qtbzn.cn/37579.Doc
m.qxw.qtbzn.cn/02864.Doc
m.qxw.qtbzn.cn/02484.Doc
m.qxw.qtbzn.cn/80424.Doc
m.qxw.qtbzn.cn/22002.Doc
m.qxw.qtbzn.cn/00668.Doc
m.qxw.qtbzn.cn/51133.Doc
m.qxw.qtbzn.cn/48424.Doc
m.qxw.qtbzn.cn/26484.Doc
m.qxw.qtbzn.cn/51553.Doc
m.qxw.qtbzn.cn/88082.Doc
m.qxw.qtbzn.cn/42804.Doc
m.qxw.qtbzn.cn/20862.Doc
m.qxw.qtbzn.cn/15979.Doc
m.qxw.qtbzn.cn/00880.Doc
m.qxw.qtbzn.cn/00860.Doc
m.qxw.qtbzn.cn/68662.Doc
m.qxq.qtbzn.cn/06626.Doc
m.qxq.qtbzn.cn/86024.Doc
m.qxq.qtbzn.cn/75533.Doc
m.qxq.qtbzn.cn/84468.Doc
m.qxq.qtbzn.cn/06086.Doc
m.qxq.qtbzn.cn/82480.Doc
m.qxq.qtbzn.cn/55711.Doc
m.qxq.qtbzn.cn/08404.Doc
m.qxq.qtbzn.cn/84860.Doc
m.qxq.qtbzn.cn/42082.Doc
m.qxq.qtbzn.cn/04828.Doc
m.qxq.qtbzn.cn/08006.Doc
m.qxq.qtbzn.cn/44440.Doc
m.qxq.qtbzn.cn/80228.Doc
m.qxq.qtbzn.cn/59991.Doc
m.qxq.qtbzn.cn/68222.Doc
m.qxq.qtbzn.cn/08448.Doc
m.qxq.qtbzn.cn/42802.Doc
m.qxq.qtbzn.cn/66260.Doc
m.qxq.qtbzn.cn/80020.Doc
m.qzm.qtbzn.cn/59117.Doc
m.qzm.qtbzn.cn/28824.Doc
m.qzm.qtbzn.cn/64228.Doc
m.qzm.qtbzn.cn/04262.Doc
m.qzm.qtbzn.cn/46200.Doc
m.qzm.qtbzn.cn/75915.Doc
m.qzm.qtbzn.cn/82644.Doc
m.qzm.qtbzn.cn/20608.Doc
m.qzm.qtbzn.cn/80024.Doc
m.qzm.qtbzn.cn/15733.Doc
m.qzm.qtbzn.cn/42820.Doc
m.qzm.qtbzn.cn/80686.Doc
m.qzm.qtbzn.cn/15773.Doc
m.qzm.qtbzn.cn/24860.Doc
m.qzm.qtbzn.cn/62222.Doc
m.qzm.qtbzn.cn/80406.Doc
m.qzm.qtbzn.cn/68482.Doc
m.qzm.qtbzn.cn/48660.Doc
m.qzm.qtbzn.cn/86288.Doc
m.qzm.qtbzn.cn/04262.Doc
m.qzn.qtbzn.cn/04860.Doc
m.qzn.qtbzn.cn/24868.Doc
m.qzn.qtbzn.cn/06068.Doc
m.qzn.qtbzn.cn/44888.Doc
m.qzn.qtbzn.cn/28086.Doc
m.qzn.qtbzn.cn/44240.Doc
m.qzn.qtbzn.cn/48844.Doc
m.qzn.qtbzn.cn/82446.Doc
m.qzn.qtbzn.cn/20004.Doc
m.qzn.qtbzn.cn/02600.Doc
m.qzn.qtbzn.cn/11957.Doc
m.qzn.qtbzn.cn/42220.Doc
m.qzn.qtbzn.cn/84822.Doc
m.qzn.qtbzn.cn/79537.Doc
m.qzn.qtbzn.cn/26464.Doc
m.qzn.qtbzn.cn/06804.Doc
m.qzn.qtbzn.cn/60626.Doc
m.qzn.qtbzn.cn/68048.Doc
m.qzn.qtbzn.cn/66688.Doc
m.qzn.qtbzn.cn/57531.Doc
m.qzb.qtbzn.cn/20242.Doc
m.qzb.qtbzn.cn/37937.Doc
m.qzb.qtbzn.cn/02848.Doc
m.qzb.qtbzn.cn/62288.Doc
m.qzb.qtbzn.cn/26062.Doc
m.qzb.qtbzn.cn/64408.Doc
m.qzb.qtbzn.cn/40640.Doc
m.qzb.qtbzn.cn/86044.Doc
m.qzb.qtbzn.cn/04648.Doc
m.qzb.qtbzn.cn/66466.Doc
m.qzb.qtbzn.cn/60862.Doc
m.qzb.qtbzn.cn/28602.Doc
m.qzb.qtbzn.cn/22884.Doc
m.qzb.qtbzn.cn/84826.Doc
m.qzb.qtbzn.cn/68624.Doc
m.qzb.qtbzn.cn/40226.Doc
m.qzb.qtbzn.cn/40604.Doc
m.qzb.qtbzn.cn/24646.Doc
m.qzb.qtbzn.cn/62668.Doc
m.qzb.qtbzn.cn/26084.Doc
m.qzv.qtbzn.cn/48066.Doc
m.qzv.qtbzn.cn/04206.Doc
m.qzv.qtbzn.cn/86088.Doc
m.qzv.qtbzn.cn/66220.Doc
m.qzv.qtbzn.cn/82646.Doc
m.qzv.qtbzn.cn/66066.Doc
m.qzv.qtbzn.cn/88242.Doc
m.qzv.qtbzn.cn/06060.Doc
m.qzv.qtbzn.cn/42246.Doc
m.qzv.qtbzn.cn/93575.Doc
m.qzv.qtbzn.cn/08820.Doc
m.qzv.qtbzn.cn/48460.Doc
m.qzv.qtbzn.cn/28828.Doc
m.qzv.qtbzn.cn/22886.Doc
m.qzv.qtbzn.cn/40260.Doc
m.qzv.qtbzn.cn/39153.Doc
m.qzv.qtbzn.cn/46404.Doc
m.qzv.qtbzn.cn/55733.Doc
m.qzv.qtbzn.cn/24248.Doc
m.qzv.qtbzn.cn/17777.Doc
m.qzc.qtbzn.cn/86806.Doc
m.qzc.qtbzn.cn/35739.Doc
m.qzc.qtbzn.cn/44688.Doc
m.qzc.qtbzn.cn/62686.Doc
m.qzc.qtbzn.cn/80022.Doc
m.qzc.qtbzn.cn/06660.Doc
m.qzc.qtbzn.cn/48466.Doc
m.qzc.qtbzn.cn/08866.Doc
m.qzc.qtbzn.cn/84206.Doc
m.qzc.qtbzn.cn/06200.Doc
m.qzc.qtbzn.cn/60064.Doc
m.qzc.qtbzn.cn/24226.Doc
m.qzc.qtbzn.cn/20448.Doc
m.qzc.qtbzn.cn/79393.Doc
m.qzc.qtbzn.cn/86224.Doc
m.qzc.qtbzn.cn/75119.Doc
m.qzc.qtbzn.cn/59593.Doc
m.qzc.qtbzn.cn/64020.Doc
m.qzc.qtbzn.cn/20068.Doc
m.qzc.qtbzn.cn/44022.Doc
m.qzx.qtbzn.cn/46644.Doc
m.qzx.qtbzn.cn/40844.Doc
m.qzx.qtbzn.cn/00224.Doc
m.qzx.qtbzn.cn/66266.Doc
m.qzx.qtbzn.cn/64624.Doc
m.qzx.qtbzn.cn/64408.Doc
m.qzx.qtbzn.cn/06086.Doc
m.qzx.qtbzn.cn/84646.Doc
m.qzx.qtbzn.cn/08282.Doc
m.qzx.qtbzn.cn/86002.Doc
m.qzx.qtbzn.cn/40444.Doc
m.qzx.qtbzn.cn/06624.Doc
m.qzx.qtbzn.cn/60484.Doc
m.qzx.qtbzn.cn/40222.Doc
m.qzx.qtbzn.cn/60264.Doc
m.qzx.qtbzn.cn/02200.Doc
m.qzx.qtbzn.cn/42242.Doc
m.qzx.qtbzn.cn/62280.Doc
m.qzx.qtbzn.cn/04442.Doc
m.qzx.qtbzn.cn/42242.Doc
m.qzz.qtbzn.cn/00244.Doc
m.qzz.qtbzn.cn/60846.Doc
m.qzz.qtbzn.cn/68284.Doc
m.qzz.qtbzn.cn/31757.Doc
m.qzz.qtbzn.cn/40682.Doc
m.qzz.qtbzn.cn/22266.Doc
m.qzz.qtbzn.cn/82648.Doc
m.qzz.qtbzn.cn/80862.Doc
m.qzz.qtbzn.cn/88082.Doc
m.qzz.qtbzn.cn/04006.Doc
m.qzz.qtbzn.cn/40064.Doc
m.qzz.qtbzn.cn/77951.Doc
m.qzz.qtbzn.cn/86682.Doc
m.qzz.qtbzn.cn/55773.Doc
m.qzz.qtbzn.cn/80486.Doc
m.qzz.qtbzn.cn/08844.Doc
m.qzz.qtbzn.cn/46220.Doc
m.qzz.qtbzn.cn/62882.Doc
m.qzz.qtbzn.cn/88420.Doc
m.qzz.qtbzn.cn/31195.Doc
m.qzl.qtbzn.cn/31397.Doc
m.qzl.qtbzn.cn/68828.Doc
m.qzl.qtbzn.cn/24044.Doc
m.qzl.qtbzn.cn/00008.Doc
m.qzl.qtbzn.cn/15913.Doc
m.qzl.qtbzn.cn/15533.Doc
m.qzl.qtbzn.cn/26024.Doc
m.qzl.qtbzn.cn/08082.Doc
m.qzl.qtbzn.cn/40664.Doc
m.qzl.qtbzn.cn/00888.Doc
m.qzl.qtbzn.cn/44444.Doc
m.qzl.qtbzn.cn/13775.Doc
m.qzl.qtbzn.cn/08222.Doc
m.qzl.qtbzn.cn/00442.Doc
m.qzl.qtbzn.cn/22664.Doc
m.qzl.qtbzn.cn/84600.Doc
m.qzl.qtbzn.cn/40640.Doc
m.qzl.qtbzn.cn/24064.Doc
m.qzl.qtbzn.cn/80222.Doc
m.qzl.qtbzn.cn/64682.Doc
m.qzk.qtbzn.cn/60466.Doc
m.qzk.qtbzn.cn/22228.Doc
m.qzk.qtbzn.cn/02446.Doc
m.qzk.qtbzn.cn/55391.Doc
m.qzk.qtbzn.cn/60440.Doc
m.qzk.qtbzn.cn/80808.Doc
m.qzk.qtbzn.cn/04244.Doc
m.qzk.qtbzn.cn/44222.Doc
