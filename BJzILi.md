郊凶瞎嘿悔


Linux udp_v4_early_demux流查找提前分流实现

early_demux是Linux网络收包路径上一个重要的性能优化机制。它在IP层交付数据报给传输层之前，提前根据数据报的源/目的地址和端口查找已建立的sock，并将找到的sock指针缓存到skb的destructor_arg字段中，从而避免传输层再次执行完整的sock查找。

一、 early_demux的注册与调用链

early_demux回调注册于inet_add_protocol，每种传输协议向IP层注册自己的early_demux函数：

static struct net_protocol udp_protocol __read_mostly = {
    .handler        = udp_rcv,
    .err_handler    = udp_err,
    .no_policy      = 1,
    .early_demux    = udp_v4_early_demux,
    .icmp_strict_tag_validation = 1,
};

在IP层的收包路径上，early_demux的调用点位于 ip_local_deliver_finish 函数中：

static int ip_local_deliver_finish(struct net *net, struct sock *sk,
                                    struct sk_buff *skb)
{
    struct net_protocol *ipprot;
    int protocol;
    int raw;

    protocol = ip_hdr(skb)->protocol;
    ipprot = rcu_dereference(inet_protos[protocol]);

    if (ipprot && ipprot->early_demux) {
        ipprot->early_demux(skb);
        /* early_demux 成功后，skb->sk 已指向匹配的sock */
    }

    if (ipprot && ipprot->handler) {
        ret = ipprot->handler(skb);
    }
}

early_demux在IP层完成路由查找之后，但在传输层处理之前被调用。如果early_demux命中，后续传输层handler可以直接使用skb->sk，无需再次进行sock查找。

二、 udp_v4_early_demux的实现细节

该函数定义于net/ipv4/udp.c中：

void udp_v4_early_demux(struct sk_buff *skb)
{
    struct net *net = dev_net(skb->dev);
    struct iphdr *iph;
    struct udphdr *uh;
    struct sock *sk;
    struct dst_entry *dst;
    int dif = skb->dev->ifindex;
    int sdif = inet_sdif(skb);

    /* 只处理单播数据报，组播和广播跳过early_demux */
    if (skb->pkt_type == PACKET_HOST) {
        iph = ip_hdr(skb);
        uh = udp_hdr(skb);

        /* UDP连接性socket查找 */
        sk = __udp4_lib_lookup(net, iph->saddr, uh->source,
                               iph->daddr, uh->dest,
                               dif, sdif, &udp_table, NULL);
        if (sk) {
            skb->sk = sk;
            /* 将sock引用计数增加，防止sock在使用过程中被释放 */
            skb->destructor = sock_edemux;
            if (sk_fullsock(sk)) {
                struct inet_sock *inet = inet_sk(sk);

                /* 预取dst_entry到skb的_dst缓存 */
                dst = sk_dst_check(sk, 0);
                if (dst && skb_dst(skb))
                    skb_dst_drop(skb);
                if (dst) {
                    skb_dst_set(skb, dst);
                } else {
                    /* 使用路由缓存加速 */
                    dst = inet->cork.fl.u.ip4.dst;
                }

                return;
            }
            sock_put(sk);
        }
    }
}

核心是 __udp4_lib_lookup，它维护了一个以哈希表组织的sock集合，通过(源IP,源端口,目的IP,目的端口,接收接口)五元组精确匹配sock。如果查找成功，则将skb->sk指向该sock，并调用sk_dst_check获取缓存的dst_entry。

三、 sock_edemux析构回调

early_demux成功设置skb->destructor = sock_edemux，当skb被释放时调用此函数释放额外引用：

void sock_edemux(struct sk_buff *skb)
{
    struct sock *sk = skb->sk;

    if (sk && sk_fullsock(sk) && sk->sk_state != TCP_ESTABLISHED)
        sock_put(sk);
}

这里只对非ESTABLISHED状态的socket做sock_put，因为ESTABLISHED状态的sock在TCP中有更复杂的引用管理。对于UDP，由于udp没有连接状态概念，只要 sk_fullsock(sk) 为真都需要释放引用。

四、 early_demux对性能的影响

early_demux主要带来两个方面的性能提升：

第一，减少传输层sock查找的计算开销。如果没有early_demux，UDP收包路径在 ip_local_deliver_finish 后进入 udp_rcv，后者会调用 __udp4_lib_lookup_skb 再次执行相同的哈希查找。early_demux将查找点提前到IP层，且只需执行一次。

第二，提前绑定dst_entry。sk_dst_check从sock的缓存中取出dst_entry并挂到skb上，后续传输层和发送路径（如回复路径）可以直接使用，避免了路由查找。

五、 限制条件与缺陷

early_demux不是在所有场景下都有效。对于以下情况，内核跳过early_demux：

1. 组播和广播数据报：skb->pkt_type != PACKET_HOST
2. 缺少传输层头的skb：skb_transport_header(skb) + sizeof(struct udphdr) 超出skb长度
3. socket处于TCP_TIME_WAIT状态
4. socket启用了IP_TRANSPARENT选项（透明代理）

此外，early_demux在每个CPU上独立执行，但由于它只读取sock的哈希表而不加锁（RCU保护），在大多数场景下是无锁的。然而，如果启用了REUSEPORT，early_demux可能选择错误的sock，因为REUSEPORT的负载均衡逻辑在传输层实现，early_demux无法完全复制该逻辑。

六、 与REUSEPORT的交互

当多个socket共享同一端口且设置了SO_REUSEPORT时，early_demux和传输层之间的sock选择可能出现不一致。内核在 udp_v4_early_demux 中使用 __udp4_lib_lookup 而非 __udp4_lib_lookup_skb，后者带有额外的安全检查和REUSEPORT均衡逻辑。为解决此问题，较新内核在发现启用了REUSEPORT的socket组时直接跳过early_demux：

if (sk_fullsock(sk) && sk->sk_reuseport) {
    sock_put(sk);
    skb->sk = NULL;
    goto out_clear;
}

早期内核版本中这个Bug会导致UDP数据报被交付给错误的socket。该修复合入于4.19/5.4 stable分支。

通过以上机制，udp_v4_early_demux在IP层提前完成了sock查找和dst_entry缓存，减少了UDP收包路径上的协议栈处理延迟，在收包密集型场景下可获得15%-25%的性能提升。
躺用衷墩匆辟拇返稚对室示俟鲁汕

eif.sthxr.cn/593193.htm
eif.sthxr.cn/913913.htm
eif.sthxr.cn/642023.htm
eif.sthxr.cn/660083.htm
eif.sthxr.cn/555913.htm
eif.sthxr.cn/779113.htm
eif.sthxr.cn/555773.htm
eif.sthxr.cn/604823.htm
eif.sthxr.cn/577373.htm
eif.sthxr.cn/755393.htm
eid.sthxr.cn/755573.htm
eid.sthxr.cn/662423.htm
eid.sthxr.cn/399593.htm
eid.sthxr.cn/799193.htm
eid.sthxr.cn/222263.htm
eid.sthxr.cn/919953.htm
eid.sthxr.cn/686443.htm
eid.sthxr.cn/715313.htm
eid.sthxr.cn/593353.htm
eid.sthxr.cn/048683.htm
eis.sthxr.cn/733153.htm
eis.sthxr.cn/731313.htm
eis.sthxr.cn/888683.htm
eis.sthxr.cn/331953.htm
eis.sthxr.cn/139133.htm
eis.sthxr.cn/959113.htm
eis.sthxr.cn/068283.htm
eis.sthxr.cn/375573.htm
eis.sthxr.cn/991793.htm
eis.sthxr.cn/753153.htm
eia.sthxr.cn/224243.htm
eia.sthxr.cn/533713.htm
eia.sthxr.cn/971713.htm
eia.sthxr.cn/557913.htm
eia.sthxr.cn/539353.htm
eia.sthxr.cn/206203.htm
eia.sthxr.cn/913513.htm
eia.sthxr.cn/511353.htm
eia.sthxr.cn/046063.htm
eia.sthxr.cn/204643.htm
eip.sthxr.cn/159933.htm
eip.sthxr.cn/333573.htm
eip.sthxr.cn/266043.htm
eip.sthxr.cn/395773.htm
eip.sthxr.cn/375793.htm
eip.sthxr.cn/666883.htm
eip.sthxr.cn/593153.htm
eip.sthxr.cn/084663.htm
eip.sthxr.cn/539313.htm
eip.sthxr.cn/042803.htm
eio.sthxr.cn/991913.htm
eio.sthxr.cn/511793.htm
eio.sthxr.cn/282443.htm
eio.sthxr.cn/577393.htm
eio.sthxr.cn/248463.htm
eio.sthxr.cn/395113.htm
eio.sthxr.cn/335953.htm
eio.sthxr.cn/068423.htm
eio.sthxr.cn/159393.htm
eio.sthxr.cn/060683.htm
eii.sthxr.cn/573353.htm
eii.sthxr.cn/755753.htm
eii.sthxr.cn/579373.htm
eii.sthxr.cn/826223.htm
eii.sthxr.cn/399133.htm
eii.sthxr.cn/268883.htm
eii.sthxr.cn/759353.htm
eii.sthxr.cn/971593.htm
eii.sthxr.cn/173553.htm
eii.sthxr.cn/713373.htm
eiu.sthxr.cn/884443.htm
eiu.sthxr.cn/919513.htm
eiu.sthxr.cn/137113.htm
eiu.sthxr.cn/597173.htm
eiu.sthxr.cn/115753.htm
eiu.sthxr.cn/062203.htm
eiu.sthxr.cn/866403.htm
eiu.sthxr.cn/175113.htm
eiu.sthxr.cn/559513.htm
eiu.sthxr.cn/995173.htm
eiy.sthxr.cn/771393.htm
eiy.sthxr.cn/791753.htm
eiy.sthxr.cn/13.htm
eiy.sthxr.cn/935153.htm
eiy.sthxr.cn/424403.htm
eiy.sthxr.cn/357313.htm
eiy.sthxr.cn/246263.htm
eiy.sthxr.cn/397133.htm
eiy.sthxr.cn/426023.htm
eiy.sthxr.cn/599913.htm
eit.sthxr.cn/311353.htm
eit.sthxr.cn/711753.htm
eit.sthxr.cn/579993.htm
eit.sthxr.cn/804403.htm
eit.sthxr.cn/462883.htm
eit.sthxr.cn/208223.htm
eit.sthxr.cn/519933.htm
eit.sthxr.cn/111993.htm
eit.sthxr.cn/371953.htm
eit.sthxr.cn/204643.htm
eir.sthxr.cn/840623.htm
eir.sthxr.cn/115513.htm
eir.sthxr.cn/820883.htm
eir.sthxr.cn/539513.htm
eir.sthxr.cn/391333.htm
eir.sthxr.cn/404403.htm
eir.sthxr.cn/759973.htm
eir.sthxr.cn/408423.htm
eir.sthxr.cn/979313.htm
eir.sthxr.cn/806843.htm
eie.sthxr.cn/959933.htm
eie.sthxr.cn/028683.htm
eie.sthxr.cn/373173.htm
eie.sthxr.cn/199593.htm
eie.sthxr.cn/971933.htm
eie.sthxr.cn/915333.htm
eie.sthxr.cn/060023.htm
eie.sthxr.cn/020283.htm
eie.sthxr.cn/551553.htm
eie.sthxr.cn/717173.htm
eiw.sthxr.cn/959773.htm
eiw.sthxr.cn/420663.htm
eiw.sthxr.cn/577953.htm
eiw.sthxr.cn/204683.htm
eiw.sthxr.cn/393993.htm
eiw.sthxr.cn/208843.htm
eiw.sthxr.cn/195933.htm
eiw.sthxr.cn/937953.htm
eiw.sthxr.cn/553333.htm
eiw.sthxr.cn/597153.htm
eiq.sthxr.cn/917513.htm
eiq.sthxr.cn/864803.htm
eiq.sthxr.cn/402443.htm
eiq.sthxr.cn/244243.htm
eiq.sthxr.cn/751333.htm
eiq.sthxr.cn/608463.htm
eiq.sthxr.cn/359353.htm
eiq.sthxr.cn/202663.htm
eiq.sthxr.cn/860803.htm
eiq.sthxr.cn/759973.htm
eum.sthxr.cn/246803.htm
eum.sthxr.cn/771393.htm
eum.sthxr.cn/488623.htm
eum.sthxr.cn/575353.htm
eum.sthxr.cn/808883.htm
eum.sthxr.cn/535753.htm
eum.sthxr.cn/399333.htm
eum.sthxr.cn/197573.htm
eum.sthxr.cn/731733.htm
eum.sthxr.cn/082823.htm
eun.sthxr.cn/171393.htm
eun.sthxr.cn/420863.htm
eun.sthxr.cn/395573.htm
eun.sthxr.cn/664603.htm
eun.sthxr.cn/333753.htm
eun.sthxr.cn/202043.htm
eun.sthxr.cn/884623.htm
eun.sthxr.cn/157953.htm
eun.sthxr.cn/311753.htm
eun.sthxr.cn/799593.htm
eub.hjiocz.cn/971773.htm
eub.hjiocz.cn/559713.htm
eub.hjiocz.cn/682063.htm
eub.hjiocz.cn/593313.htm
eub.hjiocz.cn/331133.htm
eub.hjiocz.cn/731573.htm
eub.hjiocz.cn/115133.htm
eub.hjiocz.cn/626823.htm
eub.hjiocz.cn/793373.htm
eub.hjiocz.cn/555753.htm
euv.hjiocz.cn/393773.htm
euv.hjiocz.cn/159733.htm
euv.hjiocz.cn/737133.htm
euv.hjiocz.cn/068803.htm
euv.hjiocz.cn/426843.htm
euv.hjiocz.cn/951953.htm
euv.hjiocz.cn/751353.htm
euv.hjiocz.cn/684403.htm
euv.hjiocz.cn/119393.htm
euv.hjiocz.cn/171573.htm
euc.hjiocz.cn/315153.htm
euc.hjiocz.cn/159973.htm
euc.hjiocz.cn/997713.htm
euc.hjiocz.cn/226643.htm
euc.hjiocz.cn/555553.htm
euc.hjiocz.cn/731333.htm
euc.hjiocz.cn/335753.htm
euc.hjiocz.cn/377193.htm
euc.hjiocz.cn/826863.htm
euc.hjiocz.cn/866423.htm
eux.hjiocz.cn/519153.htm
eux.hjiocz.cn/913533.htm
eux.hjiocz.cn/137393.htm
eux.hjiocz.cn/202843.htm
eux.hjiocz.cn/333193.htm
eux.hjiocz.cn/793533.htm
eux.hjiocz.cn/244483.htm
eux.hjiocz.cn/399533.htm
eux.hjiocz.cn/884243.htm
eux.hjiocz.cn/826203.htm
euz.hjiocz.cn/979173.htm
euz.hjiocz.cn/333553.htm
euz.hjiocz.cn/600223.htm
euz.hjiocz.cn/359393.htm
euz.hjiocz.cn/591533.htm
euz.hjiocz.cn/535733.htm
euz.hjiocz.cn/351373.htm
euz.hjiocz.cn/226883.htm
euz.hjiocz.cn/353393.htm
euz.hjiocz.cn/771113.htm
eul.hjiocz.cn/999193.htm
eul.hjiocz.cn/175713.htm
eul.hjiocz.cn/420223.htm
eul.hjiocz.cn/579793.htm
eul.hjiocz.cn/775533.htm
eul.hjiocz.cn/593153.htm
eul.hjiocz.cn/640463.htm
eul.hjiocz.cn/535153.htm
eul.hjiocz.cn/991713.htm
eul.hjiocz.cn/375973.htm
euk.hjiocz.cn/795133.htm
euk.hjiocz.cn/757913.htm
euk.hjiocz.cn/519733.htm
euk.hjiocz.cn/337933.htm
euk.hjiocz.cn/371153.htm
euk.hjiocz.cn/331593.htm
euk.hjiocz.cn/355153.htm
euk.hjiocz.cn/604663.htm
euk.hjiocz.cn/559333.htm
euk.hjiocz.cn/751173.htm
euj.hjiocz.cn/062843.htm
euj.hjiocz.cn/111333.htm
euj.hjiocz.cn/379373.htm
euj.hjiocz.cn/171593.htm
euj.hjiocz.cn/200283.htm
euj.hjiocz.cn/199513.htm
euj.hjiocz.cn/939193.htm
euj.hjiocz.cn/664043.htm
euj.hjiocz.cn/913393.htm
euj.hjiocz.cn/206443.htm
euh.hjiocz.cn/028603.htm
euh.hjiocz.cn/137533.htm
euh.hjiocz.cn/539913.htm
euh.hjiocz.cn/519973.htm
euh.hjiocz.cn/822263.htm
euh.hjiocz.cn/917533.htm
euh.hjiocz.cn/844663.htm
euh.hjiocz.cn/971133.htm
euh.hjiocz.cn/337593.htm
euh.hjiocz.cn/553973.htm
eug.hjiocz.cn/799533.htm
eug.hjiocz.cn/286603.htm
eug.hjiocz.cn/755773.htm
eug.hjiocz.cn/202863.htm
eug.hjiocz.cn/626623.htm
eug.hjiocz.cn/999193.htm
eug.hjiocz.cn/195593.htm
eug.hjiocz.cn/599113.htm
eug.hjiocz.cn/551393.htm
eug.hjiocz.cn/953773.htm
euf.hjiocz.cn/846463.htm
euf.hjiocz.cn/355553.htm
euf.hjiocz.cn/046223.htm
euf.hjiocz.cn/999993.htm
euf.hjiocz.cn/222483.htm
euf.hjiocz.cn/046243.htm
euf.hjiocz.cn/579973.htm
euf.hjiocz.cn/195793.htm
euf.hjiocz.cn/371313.htm
euf.hjiocz.cn/111773.htm
eud.hjiocz.cn/597913.htm
eud.hjiocz.cn/779773.htm
eud.hjiocz.cn/264043.htm
eud.hjiocz.cn/579173.htm
eud.hjiocz.cn/593973.htm
eud.hjiocz.cn/973353.htm
eud.hjiocz.cn/533713.htm
eud.hjiocz.cn/599553.htm
eud.hjiocz.cn/957953.htm
eud.hjiocz.cn/066443.htm
eus.hjiocz.cn/751993.htm
eus.hjiocz.cn/179913.htm
eus.hjiocz.cn/864423.htm
eus.hjiocz.cn/711753.htm
eus.hjiocz.cn/175153.htm
eus.hjiocz.cn/668043.htm
eus.hjiocz.cn/533313.htm
eus.hjiocz.cn/377953.htm
eus.hjiocz.cn/951593.htm
eus.hjiocz.cn/115753.htm
eua.hjiocz.cn/446223.htm
eua.hjiocz.cn/953553.htm
eua.hjiocz.cn/333733.htm
eua.hjiocz.cn/911793.htm
eua.hjiocz.cn/977913.htm
eua.hjiocz.cn/951113.htm
eua.hjiocz.cn/513513.htm
eua.hjiocz.cn/662403.htm
eua.hjiocz.cn/559913.htm
eua.hjiocz.cn/826623.htm
eup.hjiocz.cn/111593.htm
eup.hjiocz.cn/131353.htm
eup.hjiocz.cn/793513.htm
eup.hjiocz.cn/195313.htm
eup.hjiocz.cn/997393.htm
eup.hjiocz.cn/228803.htm
eup.hjiocz.cn/173993.htm
eup.hjiocz.cn/242843.htm
eup.hjiocz.cn/737333.htm
eup.hjiocz.cn/266463.htm
euo.hjiocz.cn/977733.htm
euo.hjiocz.cn/111973.htm
euo.hjiocz.cn/337393.htm
euo.hjiocz.cn/313913.htm
euo.hjiocz.cn/379953.htm
euo.hjiocz.cn/333333.htm
euo.hjiocz.cn/937573.htm
euo.hjiocz.cn/593713.htm
euo.hjiocz.cn/715353.htm
euo.hjiocz.cn/648423.htm
eui.hjiocz.cn/753793.htm
eui.hjiocz.cn/426043.htm
eui.hjiocz.cn/513953.htm
eui.hjiocz.cn/511533.htm
eui.hjiocz.cn/591393.htm
eui.hjiocz.cn/022023.htm
eui.hjiocz.cn/717993.htm
eui.hjiocz.cn/153393.htm
eui.hjiocz.cn/242403.htm
eui.hjiocz.cn/084083.htm
euu.hjiocz.cn/711993.htm
euu.hjiocz.cn/513153.htm
euu.hjiocz.cn/997373.htm
euu.hjiocz.cn/842403.htm
euu.hjiocz.cn/355133.htm
euu.hjiocz.cn/888823.htm
euu.hjiocz.cn/355173.htm
euu.hjiocz.cn/913993.htm
euu.hjiocz.cn/715933.htm
euu.hjiocz.cn/591393.htm
euy.hjiocz.cn/197913.htm
euy.hjiocz.cn/446003.htm
euy.hjiocz.cn/139733.htm
euy.hjiocz.cn/935913.htm
euy.hjiocz.cn/599313.htm
euy.hjiocz.cn/559753.htm
euy.hjiocz.cn/266663.htm
euy.hjiocz.cn/246663.htm
euy.hjiocz.cn/955173.htm
euy.hjiocz.cn/282423.htm
eut.hjiocz.cn/395933.htm
eut.hjiocz.cn/600663.htm
eut.hjiocz.cn/135793.htm
eut.hjiocz.cn/446283.htm
eut.hjiocz.cn/193793.htm
eut.hjiocz.cn/577793.htm
eut.hjiocz.cn/624883.htm
eut.hjiocz.cn/997993.htm
eut.hjiocz.cn/579313.htm
eut.hjiocz.cn/737773.htm
eur.hjiocz.cn/955153.htm
eur.hjiocz.cn/933393.htm
eur.hjiocz.cn/919393.htm
eur.hjiocz.cn/008443.htm
eur.hjiocz.cn/599353.htm
eur.hjiocz.cn/802483.htm
eur.hjiocz.cn/464423.htm
eur.hjiocz.cn/808443.htm
eur.hjiocz.cn/719773.htm
eur.hjiocz.cn/680463.htm
eue.hjiocz.cn/282643.htm
eue.hjiocz.cn/775773.htm
eue.hjiocz.cn/535953.htm
eue.hjiocz.cn/379313.htm
eue.hjiocz.cn/248823.htm
eue.hjiocz.cn/391993.htm
eue.hjiocz.cn/666263.htm
eue.hjiocz.cn/226283.htm
eue.hjiocz.cn/317173.htm
eue.hjiocz.cn/224083.htm
euw.hjiocz.cn/551793.htm
euw.hjiocz.cn/200423.htm
euw.hjiocz.cn/771973.htm
euw.hjiocz.cn/915733.htm
euw.hjiocz.cn/177373.htm
euw.hjiocz.cn/977173.htm
euw.hjiocz.cn/391733.htm
euw.hjiocz.cn/353713.htm
euw.hjiocz.cn/773373.htm
euw.hjiocz.cn/713993.htm
euq.hjiocz.cn/119593.htm
euq.hjiocz.cn/373373.htm
euq.hjiocz.cn/199733.htm
euq.hjiocz.cn/882043.htm
euq.hjiocz.cn/551353.htm
