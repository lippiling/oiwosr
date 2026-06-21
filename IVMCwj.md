握碳哦仙甭


Linux __inet_lookup_skb 接收路径哈希查找 socket 原理

__inet_lookup_skb 是 TCP 接收路径上定位 socket 的核心函数，定义在 include/net/inet_hashtables.h 中。该函数在 tcp_v4_rcv 的入口处被调用，将收到的 skb 映射到对应的 struct sock。当 skb->sk 已经存在（如 RAW socket 或 tcpdump 经 IP 层绑定），函数直接返回已有 sk 而不再查找，这是 pktgen 测试中常见的一次不走查找路径的优化路径。

```c
static inline struct sock *__inet_lookup_skb(struct inet_hashinfo *hashinfo,
                                             struct sk_buff *skb,
                                             int doff,
                                             const __be16 sport,
                                             const __be16 dport,
                                             const int sdif,
                                             bool *refcounted)
{
    struct sock *sk = skb->sk;
    const struct iphdr *iph = ip_hdr(skb);

    if (unlikely(sk && sk->sk_state != TCP_CLOSE))
        return sk;

    return __inet_lookup(dev_net(skb_dst(skb)->dev), hashinfo, skb,
                         doff, iph->saddr, sport, iph->daddr, dport,
                         inet_iif(skb), sdif, refcounted);
}
```

skb->sk 存在的条件是 IP 层已通过 early_demux 将 socket 绑定到 skb。early_demux 是接收路径优化，根据 skb 四元组直接查路由项 rt->rt_sk 并赋值 skb->sk，避免在 tcp_v4_rcv 中再次哈希查找。边界条件：若 early_demux 预绑定的 sk 已经 close（sk_state == TCP_CLOSE），则跳过重用，重新调用 __inet_lookup。这在大量短连接场景下至关重要——TIME_WAIT 或 CLOSE 状态的 socket 不参与 demux 复用，防止已关闭 sock 的非法访问。

__inet_lookup 包含层次化的三阶段查找。首先 __inet_lookup_established 在 ehash（established 哈希表）中做精确匹配。ehash 使用链式哈希 + 读写锁分离（lock_sock_table）设计，每个 bucket 存储 (saddr, sport, daddr, dport, protocol, sk->sk_bound_dev_if) 构成的四元组指纹。

```c
struct sock *__inet_lookup_established(struct net *net,
                                       struct inet_hashinfo *hashinfo,
                                       const __be32 saddr, const __be16 sport,
                                       const __be32 daddr, const __be16 dport,
                                       const int dif, const int sdif)
{
    struct sock *sk;
    const struct hlist_nulls_node *node;
    unsigned int hash = inet_ehashfn(net, daddr, dport, saddr, sport);
    unsigned int slot = hash & hashinfo->ehash_mask;
    struct inet_ehash_bucket *head = &hashinfo->ehash[slot];

begin:
    sk_nulls_for_each_rcu(sk, node, &head->chain) {
        if (sk->sk_hash != hash)
            continue;
        if (likely(inet_match(net, sk, saddr, sport, daddr, dport, dif, sdif)))
            goto found;
    }
    goto nulls_found;
found:
    if (unlikely(!refcount_inc_not_zero(&sk->sk_refcnt)))
        goto begin;
    ...
}
```

nulls 链表在这里起到关键作用：使用 hlist_nulls_for_each_rcu 遍历替代标准 hlist，避免在遍历过程中删除节点时出现 ABA 问题。当 __inet_lookup_established 遍历链尾时若遇到 NULLS 标记（node->next 的低位为 1），说明遍历越过尾节点且未因 resizing 导致误判，此时直接返回 NULL。若因并发 resize 导致无效遍历，则跳回 begin 重新查找。

若查找 ehash 未命中，进入第二阶段 __inet_lookup_listener，在 listening_hash（listener 哈希表，即 ilb）中搜索监听 socket。listener 的查找使用完全四元组匹配（TCP_SEQ_FN_MATCH）或通配符匹配（TCP_SEQ_FN_WILDCARD），优先匹配不带设备的 listener，其次匹配绑定具体设备的 listener。当建立大量监听 socket（如反向代理场景），listener 哈希表退化为链表遍历，此时 O(N) 复杂度会直接拖累接收吞吐。内核通过设置 net.ipv4.tcp_listener_hash_buckets 或启用 SO_REUSEPORT 来分散 listener 负载。

```c
struct sock *__inet_lookup_listener(struct net *net,
                                    struct inet_hashinfo *hashinfo,
                                    struct sk_buff *skb, int doff,
                                    const __be32 saddr, __be16 sport,
                                    const __be32 daddr, const __be16 dport,
                                    const int dif, const int sdif)
{
    struct sock *sk, *result = NULL;
    u32 phash = inet_lhashfn(net, hinfo, dport);
    struct inet_listen_hash_bucket *ilb = &hashinfo->listening_hash[phash];

    sk_for_each_rcu(sk, &ilb->head) {
        const struct inet_sock *inet = inet_sk(sk);
        int score = compute_score(net, sk, saddr, sport, daddr, dport, dif, sdif);

        if (score == -1)
            continue;
        if (score > best_score) {
            result = sk;
            best_score = score;
        }
    }
    return result;
}
```

compute_score 的优先级算法按 score 排序，score 的最低位决定匹配精度。SO_REUSEPORT 场景下还需要通过 inet_reuseport_select_sock 做二次选择，选中与发送端 CPU 编号关联的 listener（通过 BPF 或哈希取模）以最小化监听队列的 cacheline 竞争。不启用 SO_REUSEPORT 时所有 listener 共享同一 ilb 桶，accept 队列锁争用严重，在 80 核以上的机器上是一个明确的扩展性瓶颈。

第三个阶段涉及 timewait 处理。__inet_lookup_skb 不直接处理 TW 套接字，但 __inet_lookup_established 循环中若找到 tw 类型的 sock，调用者需要通过 tcp_timewait_state_process 进行 PAWS 和序列号校验，决定是否允许新 SYN 重用该 TW 槽位。在 TIME_WAIT 套接字存活期间 (2MSL)，如果接收到合法 RST 或新 SYN 且开启了 net.ipv4.tcp_tw_reuse，内核通过 twsk_unique 检查时间戳增量，决定是否提前回收 tw。

tcp_v4_rcv 调用 __inet_lookup_skb 的典型错误路径是收到非 SYN 报文但 lookup 返回 NULL（即 ehash 和 listener 均未命中），此时发送 RST 响应。如果报文是 RST 自身，则直接丢弃，避免 RST 风暴。若 __inet_lookup_skb 返回的 sk 的 sk_state 在 socket 销毁过程中处于 TCP_CLOSE_WAIT 且 sk 已释放，refcount_inc_not_zero 会竞争失败导致进入 begin 重试。
晒科贫油融倮有备詹有蚁现导嗡尚

m.exf.msfsx.cn/39119.Doc
m.exf.msfsx.cn/39755.Doc
m.exf.msfsx.cn/75115.Doc
m.exf.msfsx.cn/68222.Doc
m.exf.msfsx.cn/59711.Doc
m.exf.msfsx.cn/13933.Doc
m.exf.msfsx.cn/66026.Doc
m.exf.msfsx.cn/26628.Doc
m.exf.msfsx.cn/75711.Doc
m.exf.msfsx.cn/35555.Doc
m.exf.msfsx.cn/62280.Doc
m.exf.msfsx.cn/17711.Doc
m.exf.msfsx.cn/68666.Doc
m.exf.msfsx.cn/26644.Doc
m.exf.msfsx.cn/62684.Doc
m.exd.msfsx.cn/51799.Doc
m.exd.msfsx.cn/15973.Doc
m.exd.msfsx.cn/44088.Doc
m.exd.msfsx.cn/51599.Doc
m.exd.msfsx.cn/62424.Doc
m.exd.msfsx.cn/95355.Doc
m.exd.msfsx.cn/15591.Doc
m.exd.msfsx.cn/31775.Doc
m.exd.msfsx.cn/59113.Doc
m.exd.msfsx.cn/20020.Doc
m.exd.msfsx.cn/35151.Doc
m.exd.msfsx.cn/68088.Doc
m.exd.msfsx.cn/88462.Doc
m.exd.msfsx.cn/80020.Doc
m.exd.msfsx.cn/46860.Doc
m.exd.msfsx.cn/11713.Doc
m.exd.msfsx.cn/77597.Doc
m.exd.msfsx.cn/71337.Doc
m.exd.msfsx.cn/26646.Doc
m.exd.msfsx.cn/06086.Doc
m.exs.msfsx.cn/86004.Doc
m.exs.msfsx.cn/06880.Doc
m.exs.msfsx.cn/11713.Doc
m.exs.msfsx.cn/55199.Doc
m.exs.msfsx.cn/19751.Doc
m.exs.msfsx.cn/17959.Doc
m.exs.msfsx.cn/79331.Doc
m.exs.msfsx.cn/15913.Doc
m.exs.msfsx.cn/95519.Doc
m.exs.msfsx.cn/37933.Doc
m.exs.msfsx.cn/93515.Doc
m.exs.msfsx.cn/17771.Doc
m.exs.msfsx.cn/51531.Doc
m.exs.msfsx.cn/17795.Doc
m.exs.msfsx.cn/73993.Doc
m.exs.msfsx.cn/33931.Doc
m.exs.msfsx.cn/15197.Doc
m.exs.msfsx.cn/53597.Doc
m.exs.msfsx.cn/39115.Doc
m.exs.msfsx.cn/97791.Doc
m.exa.msfsx.cn/46842.Doc
m.exa.msfsx.cn/97939.Doc
m.exa.msfsx.cn/71391.Doc
m.exa.msfsx.cn/97111.Doc
m.exa.msfsx.cn/95737.Doc
m.exa.msfsx.cn/26000.Doc
m.exa.msfsx.cn/80482.Doc
m.exa.msfsx.cn/40082.Doc
m.exa.msfsx.cn/71339.Doc
m.exa.msfsx.cn/71793.Doc
m.exa.msfsx.cn/19159.Doc
m.exa.msfsx.cn/00886.Doc
m.exa.msfsx.cn/48804.Doc
m.exa.msfsx.cn/46046.Doc
m.exa.msfsx.cn/53773.Doc
m.exa.msfsx.cn/02600.Doc
m.exa.msfsx.cn/84642.Doc
m.exa.msfsx.cn/06244.Doc
m.exa.msfsx.cn/48062.Doc
m.exa.msfsx.cn/68226.Doc
m.exp.msfsx.cn/60204.Doc
m.exp.msfsx.cn/88462.Doc
m.exp.msfsx.cn/39359.Doc
m.exp.msfsx.cn/75979.Doc
m.exp.msfsx.cn/71739.Doc
m.exp.msfsx.cn/00400.Doc
m.exp.msfsx.cn/64446.Doc
m.exp.msfsx.cn/86266.Doc
m.exp.msfsx.cn/31795.Doc
m.exp.msfsx.cn/66402.Doc
m.exp.msfsx.cn/26202.Doc
m.exp.msfsx.cn/28026.Doc
m.exp.msfsx.cn/04066.Doc
m.exp.msfsx.cn/44448.Doc
m.exp.msfsx.cn/33979.Doc
m.exp.msfsx.cn/68666.Doc
m.exp.msfsx.cn/84020.Doc
m.exp.msfsx.cn/24246.Doc
m.exp.msfsx.cn/86482.Doc
m.exp.msfsx.cn/84466.Doc
m.exo.msfsx.cn/35375.Doc
m.exo.msfsx.cn/86688.Doc
m.exo.msfsx.cn/91933.Doc
m.exo.msfsx.cn/55173.Doc
m.exo.msfsx.cn/17531.Doc
m.exo.msfsx.cn/91359.Doc
m.exo.msfsx.cn/93953.Doc
m.exo.msfsx.cn/44024.Doc
m.exo.msfsx.cn/37751.Doc
m.exo.msfsx.cn/71755.Doc
m.exo.msfsx.cn/73337.Doc
m.exo.msfsx.cn/17917.Doc
m.exo.msfsx.cn/59357.Doc
m.exo.msfsx.cn/11575.Doc
m.exo.msfsx.cn/53511.Doc
m.exo.msfsx.cn/75715.Doc
m.exo.msfsx.cn/77117.Doc
m.exo.msfsx.cn/93195.Doc
m.exo.msfsx.cn/53791.Doc
m.exo.msfsx.cn/44220.Doc
m.exi.msfsx.cn/82662.Doc
m.exi.msfsx.cn/39591.Doc
m.exi.msfsx.cn/51515.Doc
m.exi.msfsx.cn/39135.Doc
m.exi.msfsx.cn/80022.Doc
m.exi.msfsx.cn/79559.Doc
m.exi.msfsx.cn/19115.Doc
m.exi.msfsx.cn/48880.Doc
m.exi.msfsx.cn/77351.Doc
m.exi.msfsx.cn/35793.Doc
m.exi.msfsx.cn/15513.Doc
m.exi.msfsx.cn/57997.Doc
m.exi.msfsx.cn/86240.Doc
m.exi.msfsx.cn/42848.Doc
m.exi.msfsx.cn/15199.Doc
m.exi.msfsx.cn/46028.Doc
m.exi.msfsx.cn/13119.Doc
m.exi.msfsx.cn/15195.Doc
m.exi.msfsx.cn/15195.Doc
m.exi.msfsx.cn/55593.Doc
m.exu.msfsx.cn/06244.Doc
m.exu.msfsx.cn/20022.Doc
m.exu.msfsx.cn/48248.Doc
m.exu.msfsx.cn/71937.Doc
m.exu.msfsx.cn/97593.Doc
m.exu.msfsx.cn/95751.Doc
m.exu.msfsx.cn/55577.Doc
m.exu.msfsx.cn/84280.Doc
m.exu.msfsx.cn/02828.Doc
m.exu.msfsx.cn/80048.Doc
m.exu.msfsx.cn/22826.Doc
m.exu.msfsx.cn/95955.Doc
m.exu.msfsx.cn/73919.Doc
m.exu.msfsx.cn/57759.Doc
m.exu.msfsx.cn/77753.Doc
m.exu.msfsx.cn/40004.Doc
m.exu.msfsx.cn/22068.Doc
m.exu.msfsx.cn/93951.Doc
m.exu.msfsx.cn/93773.Doc
m.exu.msfsx.cn/93395.Doc
m.exy.msfsx.cn/93735.Doc
m.exy.msfsx.cn/97539.Doc
m.exy.msfsx.cn/99951.Doc
m.exy.msfsx.cn/73957.Doc
m.exy.msfsx.cn/97377.Doc
m.exy.msfsx.cn/77393.Doc
m.exy.msfsx.cn/64444.Doc
m.exy.msfsx.cn/02200.Doc
m.exy.msfsx.cn/26624.Doc
m.exy.msfsx.cn/57111.Doc
m.exy.msfsx.cn/71593.Doc
m.exy.msfsx.cn/60086.Doc
m.exy.msfsx.cn/51731.Doc
m.exy.msfsx.cn/11551.Doc
m.exy.msfsx.cn/66286.Doc
m.exy.msfsx.cn/51119.Doc
m.exy.msfsx.cn/73799.Doc
m.exy.msfsx.cn/51739.Doc
m.exy.msfsx.cn/40486.Doc
m.exy.msfsx.cn/19939.Doc
m.ext.msfsx.cn/17177.Doc
m.ext.msfsx.cn/77717.Doc
m.ext.msfsx.cn/15957.Doc
m.ext.msfsx.cn/11335.Doc
m.ext.msfsx.cn/15357.Doc
m.ext.msfsx.cn/13753.Doc
m.ext.msfsx.cn/75799.Doc
m.ext.msfsx.cn/42046.Doc
m.ext.msfsx.cn/46802.Doc
m.ext.msfsx.cn/20044.Doc
m.ext.msfsx.cn/44844.Doc
m.ext.msfsx.cn/22866.Doc
m.ext.msfsx.cn/60626.Doc
m.ext.msfsx.cn/60020.Doc
m.ext.msfsx.cn/57395.Doc
m.ext.msfsx.cn/13577.Doc
m.ext.msfsx.cn/40806.Doc
m.ext.msfsx.cn/75795.Doc
m.ext.msfsx.cn/82280.Doc
m.ext.msfsx.cn/51577.Doc
m.exr.msfsx.cn/93179.Doc
m.exr.msfsx.cn/53517.Doc
m.exr.msfsx.cn/82462.Doc
m.exr.msfsx.cn/97377.Doc
m.exr.msfsx.cn/08804.Doc
m.exr.msfsx.cn/35571.Doc
m.exr.msfsx.cn/62602.Doc
m.exr.msfsx.cn/77977.Doc
m.exr.msfsx.cn/91513.Doc
m.exr.msfsx.cn/48248.Doc
m.exr.msfsx.cn/60266.Doc
m.exr.msfsx.cn/06022.Doc
m.exr.msfsx.cn/33353.Doc
m.exr.msfsx.cn/53539.Doc
m.exr.msfsx.cn/08888.Doc
m.exr.msfsx.cn/11535.Doc
m.exr.msfsx.cn/53951.Doc
m.exr.msfsx.cn/08886.Doc
m.exr.msfsx.cn/60800.Doc
m.exr.msfsx.cn/64220.Doc
m.exe.msfsx.cn/48448.Doc
m.exe.msfsx.cn/40628.Doc
m.exe.msfsx.cn/28064.Doc
m.exe.msfsx.cn/84060.Doc
m.exe.msfsx.cn/84800.Doc
m.exe.msfsx.cn/86026.Doc
m.exe.msfsx.cn/95533.Doc
m.exe.msfsx.cn/11979.Doc
m.exe.msfsx.cn/71979.Doc
m.exe.msfsx.cn/95557.Doc
m.exe.msfsx.cn/15119.Doc
m.exe.msfsx.cn/35739.Doc
m.exe.msfsx.cn/71377.Doc
m.exe.msfsx.cn/37371.Doc
m.exe.msfsx.cn/24044.Doc
m.exe.msfsx.cn/84042.Doc
m.exe.msfsx.cn/60828.Doc
m.exe.msfsx.cn/71775.Doc
m.exe.msfsx.cn/26826.Doc
m.exe.msfsx.cn/59579.Doc
m.exw.msfsx.cn/19975.Doc
m.exw.msfsx.cn/66248.Doc
m.exw.msfsx.cn/51935.Doc
m.exw.msfsx.cn/08448.Doc
m.exw.msfsx.cn/9.Doc
m.exw.msfsx.cn/77757.Doc
m.exw.msfsx.cn/91579.Doc
m.exw.msfsx.cn/73133.Doc
m.exw.msfsx.cn/13973.Doc
m.exw.msfsx.cn/91199.Doc
m.exw.msfsx.cn/97339.Doc
m.exw.msfsx.cn/97577.Doc
m.exw.msfsx.cn/19553.Doc
m.exw.msfsx.cn/55597.Doc
m.exw.msfsx.cn/88640.Doc
m.exw.msfsx.cn/73311.Doc
m.exw.msfsx.cn/51979.Doc
m.exw.msfsx.cn/40820.Doc
m.exw.msfsx.cn/62466.Doc
m.exw.msfsx.cn/93335.Doc
m.exq.msfsx.cn/53311.Doc
m.exq.msfsx.cn/39157.Doc
m.exq.msfsx.cn/17759.Doc
m.exq.msfsx.cn/51191.Doc
m.exq.msfsx.cn/42262.Doc
m.exq.msfsx.cn/22884.Doc
m.exq.msfsx.cn/66482.Doc
m.exq.msfsx.cn/82082.Doc
m.exq.msfsx.cn/08620.Doc
m.exq.msfsx.cn/88288.Doc
m.exq.msfsx.cn/80408.Doc
m.exq.msfsx.cn/55555.Doc
m.exq.msfsx.cn/31157.Doc
m.exq.msfsx.cn/39311.Doc
m.exq.msfsx.cn/55539.Doc
m.exq.msfsx.cn/55153.Doc
m.exq.msfsx.cn/53591.Doc
m.exq.msfsx.cn/84682.Doc
m.exq.msfsx.cn/91155.Doc
m.exq.msfsx.cn/33991.Doc
m.ezm.msfsx.cn/04204.Doc
m.ezm.msfsx.cn/84644.Doc
m.ezm.msfsx.cn/51997.Doc
m.ezm.msfsx.cn/64022.Doc
m.ezm.msfsx.cn/11173.Doc
m.ezm.msfsx.cn/62222.Doc
m.ezm.msfsx.cn/02840.Doc
m.ezm.msfsx.cn/71779.Doc
m.ezm.msfsx.cn/57971.Doc
m.ezm.msfsx.cn/06400.Doc
m.ezm.msfsx.cn/22688.Doc
m.ezm.msfsx.cn/60842.Doc
m.ezm.msfsx.cn/35919.Doc
m.ezm.msfsx.cn/13739.Doc
m.ezm.msfsx.cn/79971.Doc
m.ezm.msfsx.cn/62668.Doc
m.ezm.msfsx.cn/06262.Doc
m.ezm.msfsx.cn/99917.Doc
m.ezm.msfsx.cn/17713.Doc
m.ezm.msfsx.cn/80242.Doc
m.ezn.msfsx.cn/08664.Doc
m.ezn.msfsx.cn/08682.Doc
m.ezn.msfsx.cn/80684.Doc
m.ezn.msfsx.cn/64204.Doc
m.ezn.msfsx.cn/62644.Doc
m.ezn.msfsx.cn/82046.Doc
m.ezn.msfsx.cn/33395.Doc
m.ezn.msfsx.cn/73797.Doc
m.ezn.msfsx.cn/75533.Doc
m.ezn.msfsx.cn/68206.Doc
m.ezn.msfsx.cn/40848.Doc
m.ezn.msfsx.cn/08648.Doc
m.ezn.msfsx.cn/97353.Doc
m.ezn.msfsx.cn/19197.Doc
m.ezn.msfsx.cn/02224.Doc
m.ezn.msfsx.cn/26884.Doc
m.ezn.msfsx.cn/88848.Doc
m.ezn.msfsx.cn/00604.Doc
m.ezn.msfsx.cn/55117.Doc
m.ezn.msfsx.cn/20202.Doc
m.ezb.msfsx.cn/55513.Doc
m.ezb.msfsx.cn/64646.Doc
m.ezb.msfsx.cn/13991.Doc
m.ezb.msfsx.cn/17591.Doc
m.ezb.msfsx.cn/02242.Doc
m.ezb.msfsx.cn/73913.Doc
m.ezb.msfsx.cn/37351.Doc
m.ezb.msfsx.cn/79917.Doc
m.ezb.msfsx.cn/77755.Doc
m.ezb.msfsx.cn/17533.Doc
m.ezb.msfsx.cn/55537.Doc
m.ezb.msfsx.cn/00842.Doc
m.ezb.msfsx.cn/53153.Doc
m.ezb.msfsx.cn/75597.Doc
m.ezb.msfsx.cn/88844.Doc
m.ezb.msfsx.cn/39939.Doc
m.ezb.msfsx.cn/55113.Doc
m.ezb.msfsx.cn/08046.Doc
m.ezb.msfsx.cn/84002.Doc
m.ezb.msfsx.cn/46288.Doc
m.ezv.msfsx.cn/39995.Doc
m.ezv.msfsx.cn/44282.Doc
m.ezv.msfsx.cn/24008.Doc
m.ezv.msfsx.cn/86064.Doc
m.ezv.msfsx.cn/13597.Doc
m.ezv.msfsx.cn/97575.Doc
m.ezv.msfsx.cn/11357.Doc
m.ezv.msfsx.cn/59759.Doc
m.ezv.msfsx.cn/13199.Doc
m.ezv.msfsx.cn/64482.Doc
m.ezv.msfsx.cn/80462.Doc
m.ezv.msfsx.cn/66668.Doc
m.ezv.msfsx.cn/79735.Doc
m.ezv.msfsx.cn/60820.Doc
m.ezv.msfsx.cn/53739.Doc
m.ezv.msfsx.cn/33715.Doc
m.ezv.msfsx.cn/24244.Doc
m.ezv.msfsx.cn/88482.Doc
m.ezv.msfsx.cn/06440.Doc
m.ezv.msfsx.cn/33797.Doc
m.ezc.msfsx.cn/55955.Doc
m.ezc.msfsx.cn/39357.Doc
m.ezc.msfsx.cn/00280.Doc
m.ezc.msfsx.cn/08646.Doc
m.ezc.msfsx.cn/68662.Doc
m.ezc.msfsx.cn/15555.Doc
m.ezc.msfsx.cn/40644.Doc
m.ezc.msfsx.cn/59977.Doc
m.ezc.msfsx.cn/73559.Doc
m.ezc.msfsx.cn/55559.Doc
m.ezc.msfsx.cn/28844.Doc
m.ezc.msfsx.cn/28880.Doc
m.ezc.msfsx.cn/66644.Doc
m.ezc.msfsx.cn/19337.Doc
m.ezc.msfsx.cn/06828.Doc
m.ezc.msfsx.cn/64004.Doc
m.ezc.msfsx.cn/97799.Doc
m.ezc.msfsx.cn/73375.Doc
m.ezc.msfsx.cn/68248.Doc
m.ezc.msfsx.cn/48864.Doc
m.ezx.msfsx.cn/19197.Doc
m.ezx.msfsx.cn/53379.Doc
m.ezx.msfsx.cn/93755.Doc
m.ezx.msfsx.cn/55959.Doc
m.ezx.msfsx.cn/39579.Doc
m.ezx.msfsx.cn/08648.Doc
m.ezx.msfsx.cn/62088.Doc
m.ezx.msfsx.cn/44042.Doc
m.ezx.msfsx.cn/46228.Doc
m.ezx.msfsx.cn/19713.Doc
m.ezx.msfsx.cn/62068.Doc
m.ezx.msfsx.cn/31971.Doc
m.ezx.msfsx.cn/37515.Doc
m.ezx.msfsx.cn/31153.Doc
m.ezx.msfsx.cn/40228.Doc
m.ezx.msfsx.cn/77197.Doc
m.ezx.msfsx.cn/95153.Doc
m.ezx.msfsx.cn/46266.Doc
m.ezx.msfsx.cn/06084.Doc
m.ezx.msfsx.cn/24622.Doc
