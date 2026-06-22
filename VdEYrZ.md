磐汾屹采宦


Linux virtio_net tx napi与mergeable bufr接收

Virtio-net是qemu/kvm虚拟化环境中guest的前端网卡驱动，位于drivers/net/virtio_net.c。它通过virtqueue与后端（vhost-net或qemu）交换buffer描述符，支持tx napi和mergeable buffer接收两种关键机制。

TX NAPI模式下，发送完成中断通过napi回调处理，而不是在每个报文发送后立即产生中断。start_xmit函数将skb填入virtqueue后，只有当vring满或队列停止时才触发napi调度：

static netdev_tx_t start_xmit(struct sk_buff *skb, struct net_device *dev)
{
    struct virtnet_info *vi = netdev_priv(dev);
    struct send_queue *sq = &vi->sq[skb_get_queue_mapping(skb)];
    struct netdev_queue *txq = netdev_get_tx_queue(dev, skb_get_queue_mapping(skb));
    bool kick = false;
    int qnum = skb_get_queue_mapping(skb);

    if (skb_orphan_frags_rx(skb, GFP_ATOMIC)) {
        dev_kfree_skb_any(skb);
        return NETDEV_TX_OK;
    }

    if (unlikely(!xmit_more || netif_xmit_stopped(txq))) {
        kick = true;
    }

    if (vi->mergeable_rx_bufs)
        skb_get_hash(skb);

    if (skb_vlan_tag_present(skb)) {
        // insert VLAN header
    }

    return virtnet_xmit(skb, vi, sq, qnum, kick);
}

TX完成侧，virtnet_poll_tx处理已发送buffer的回写：

static int virtnet_poll_tx(struct napi_struct *napi, int budget)
{
    struct send_queue *sq = container_of(napi, struct send_queue, napi);
    struct virtnet_info *vi = sq->vq->vdev->priv;
    struct netdev_queue *txq;
    struct sk_buff *skb;
    unsigned int sent = 0;
    u32 packets = 0;
    unsigned int bytes = 0;

    while (likely(sent < budget &&
                  (skb = virtqueue_get_buf(sq->vq, &bytes)) != NULL)) {
        bytes += skb->len;
        packets++;
        napi_consume_skb(skb, budget);
        sent++;
    }

    if (sent < budget) {
        napi_complete_done(napi, sent);
        if (virtqueue_enable_cb(sq->vq))
            napi_schedule_irqoff(napi);
    }

    txq = netdev_get_tx_queue(vi->dev, sq - vi->sq);
    netdev_tx_completed_queue(txq, packets, bytes);

    if (unlikely(netif_tx_queue_stopped(txq) &&
                 virtqueue_enable_cb_delayed(sq->vq)))
        netif_tx_wake_queue(txq);

    return sent;
}

Mergeable buffer接收是virtio-net为了减少虚拟化overhead设计的机制。guest驱动预先分配多个buf描述符，后端可以连续写入多个小报文或使用多个buf拼接一个大报文。初始化在virtnet_probe中设置：

static int virtnet_probe(struct virtio_device *vdev)
{
    struct virtnet_info *vi;
    int err;

    vi = netdev_priv(dev);
    INIT_WORK(&vi->config_work, virtnet_config_changed_work);

    if (virtio_has_feature(vdev, VIRTIO_NET_F_MRG_RXBUF))
        vi->mergeable_rx_bufs = true;

    if (vi->mergeable_rx_bufs)
        vi->rq[0].pages = &vi->mrg_avail_bufs;
}

接收NAPI为virtnet_poll，轮询接收virtqueue获取报文：

static int virtnet_poll(struct napi_struct *napi, int budget)
{
    struct receive_queue *rq = container_of(napi, struct receive_queue, napi);
    struct virtnet_info *vi = rq->vq->vdev->priv;
    unsigned int received = 0;
    int packets = 0;
    int bytes = 0;

    while (received < budget &&
           (skb = receive_mergeable(vi, rq, &bytes)) != NULL) {
        packets++;
        bytes += skb->len;
        napi_gro_receive(&rq->napi, skb);
        received++;
    }

    if (received < budget) {
        napi_complete_done(napi, received);
        if (virtqueue_enable_cb(rq->vq))
            napi_schedule_irqoff(napi);
    }

    return received;
}

receive_mergeable是mergeable接收的核心实现：

static struct sk_buff *receive_mergeable(struct virtnet_info *vi,
                                         struct receive_queue *rq,
                                         unsigned int *bytes)
{
    struct virtnet_rq_stats stats;
    struct sk_buff *skb;
    struct page *page;
    int offset;
    u16 num_buf;

    skb = receive_buf(vi, rq, rq->buf, &stats);
    if (!skb)
        return NULL;

    if (vi->mergeable_rx_bufs && skb_is_nonlinear(skb)) {
        num_buf = virtio16_to_cpu(vi->vdev,
                   ((struct virtio_net_hdr_mrg_rxbuf *)skb->cb)->num_buffers);

        while (--num_buf) {
            page = virtqueue_get_buf(rq->vq, &stats.len);
            if (!page) {
                dev_kfree_skb(skb);
                return NULL;
            }

            offset = 0;
            if (page_va(page) != NULL) {
                skb_add_rx_frag(skb, skb_shinfo(skb)->nr_frags,
                                virt_to_page(page), offset, stats.len,
                                stats.len);
            }
        }
    }

    *bytes = skb->len;
    return skb;
}

Mergeable buffer的buf分配使用try_fill_recv函数，通过add_recvbuf_mergeable向virtqueue填充空buf：

static int add_recvbuf_mergeable(struct virtnet_info *vi,
                                 struct receive_queue *rq, gfp_t gfp)
{
    struct page *page;
    int err;
    unsigned int len = vi->hdr_len;

    page = alloc_page(gfp);
    if (!page)
        return -ENOMEM;

    err = virtqueue_add_inbuf(rq->vq, rq->sg, 1,
                              page_address(page), len + PAGE_SIZE, page,
                              gfp);
    if (err < 0)
        put_page(page);

    return err;
}

TX NAPI与mergeable接收配合时，收发包共用同一个napi实例（如果vi->tx_queue_size启用）。二者分别绑定到各自的virtqueue，通过napi的poll函数分派。当virtio后端在同一个vq上既收又发时，单napi轮询顺序为先处理tx complete再处理rx，这个调度策略通过virtnet_poll_tx和receive_mergeable在同一个softirq上下文中完成。

九僦寥把揖临灰滤布撇姑啪妇鹊澈

enu.mmmxz.cn/153713.htm
enu.mmmxz.cn/208403.htm
enu.mmmxz.cn/462203.htm
enu.mmmxz.cn/995153.htm
enu.mmmxz.cn/244023.htm
eny.mmmxz.cn/848843.htm
eny.mmmxz.cn/571913.htm
eny.mmmxz.cn/575593.htm
eny.mmmxz.cn/820243.htm
eny.mmmxz.cn/315993.htm
eny.mmmxz.cn/446803.htm
eny.mmmxz.cn/066883.htm
eny.mmmxz.cn/202063.htm
eny.mmmxz.cn/648803.htm
eny.mmmxz.cn/911773.htm
ent.mmmxz.cn/404243.htm
ent.mmmxz.cn/626863.htm
ent.mmmxz.cn/951733.htm
ent.mmmxz.cn/280883.htm
ent.mmmxz.cn/680023.htm
ent.mmmxz.cn/793333.htm
ent.mmmxz.cn/880883.htm
ent.mmmxz.cn/242883.htm
ent.mmmxz.cn/204443.htm
ent.mmmxz.cn/244843.htm
enr.mmmxz.cn/244223.htm
enr.mmmxz.cn/197953.htm
enr.mmmxz.cn/139793.htm
enr.mmmxz.cn/824803.htm
enr.mmmxz.cn/882263.htm
enr.mmmxz.cn/244243.htm
enr.mmmxz.cn/828203.htm
enr.mmmxz.cn/771353.htm
enr.mmmxz.cn/460663.htm
enr.mmmxz.cn/028423.htm
ene.mmmxz.cn/622483.htm
ene.mmmxz.cn/797973.htm
ene.mmmxz.cn/824063.htm
ene.mmmxz.cn/462883.htm
ene.mmmxz.cn/048423.htm
ene.mmmxz.cn/882223.htm
ene.mmmxz.cn/824643.htm
ene.mmmxz.cn/971333.htm
ene.mmmxz.cn/466043.htm
ene.mmmxz.cn/844623.htm
enw.mmmxz.cn/846463.htm
enw.mmmxz.cn/739913.htm
enw.mmmxz.cn/046843.htm
enw.mmmxz.cn/153933.htm
enw.mmmxz.cn/680423.htm
enw.mmmxz.cn/688863.htm
enw.mmmxz.cn/515793.htm
enw.mmmxz.cn/799513.htm
enw.mmmxz.cn/575393.htm
enw.mmmxz.cn/204003.htm
enq.mmmxz.cn/620663.htm
enq.mmmxz.cn/086003.htm
enq.mmmxz.cn/806203.htm
enq.mmmxz.cn/240463.htm
enq.mmmxz.cn/557313.htm
enq.mmmxz.cn/286223.htm
enq.mmmxz.cn/448443.htm
enq.mmmxz.cn/331373.htm
enq.mmmxz.cn/957593.htm
enq.mmmxz.cn/999573.htm
ebm.mmmxz.cn/266043.htm
ebm.mmmxz.cn/662403.htm
ebm.mmmxz.cn/806423.htm
ebm.mmmxz.cn/484603.htm
ebm.mmmxz.cn/024623.htm
ebm.mmmxz.cn/115113.htm
ebm.mmmxz.cn/242623.htm
ebm.mmmxz.cn/802423.htm
ebm.mmmxz.cn/115973.htm
ebm.mmmxz.cn/424623.htm
ebn.mmmxz.cn/488423.htm
ebn.mmmxz.cn/822883.htm
ebn.mmmxz.cn/468083.htm
ebn.mmmxz.cn/917553.htm
ebn.mmmxz.cn/486283.htm
ebn.mmmxz.cn/002843.htm
ebn.mmmxz.cn/135133.htm
ebn.mmmxz.cn/915933.htm
ebn.mmmxz.cn/175193.htm
ebn.mmmxz.cn/462223.htm
ebb.mmmxz.cn/799973.htm
ebb.mmmxz.cn/537553.htm
ebb.mmmxz.cn/373593.htm
ebb.mmmxz.cn/939773.htm
ebb.mmmxz.cn/444863.htm
ebb.mmmxz.cn/200243.htm
ebb.mmmxz.cn/313373.htm
ebb.mmmxz.cn/737313.htm
ebb.mmmxz.cn/020643.htm
ebb.mmmxz.cn/993193.htm
ebv.mmmxz.cn/379313.htm
ebv.mmmxz.cn/020643.htm
ebv.mmmxz.cn/288863.htm
ebv.mmmxz.cn/939553.htm
ebv.mmmxz.cn/377993.htm
ebv.mmmxz.cn/880683.htm
ebv.mmmxz.cn/206043.htm
ebv.mmmxz.cn/557133.htm
ebv.mmmxz.cn/668683.htm
ebv.mmmxz.cn/484623.htm
ebc.mmmxz.cn/282863.htm
ebc.mmmxz.cn/933373.htm
ebc.mmmxz.cn/468863.htm
ebc.mmmxz.cn/973973.htm
ebc.mmmxz.cn/288083.htm
ebc.mmmxz.cn/280283.htm
ebc.mmmxz.cn/775553.htm
ebc.mmmxz.cn/917993.htm
ebc.mmmxz.cn/624443.htm
ebc.mmmxz.cn/842423.htm
ebx.mmmxz.cn/062243.htm
ebx.mmmxz.cn/668803.htm
ebx.mmmxz.cn/844823.htm
ebx.mmmxz.cn/737953.htm
ebx.mmmxz.cn/793993.htm
ebx.mmmxz.cn/660023.htm
ebx.mmmxz.cn/575133.htm
ebx.mmmxz.cn/953333.htm
ebx.mmmxz.cn/646003.htm
ebx.mmmxz.cn/395313.htm
ebz.mmmxz.cn/266463.htm
ebz.mmmxz.cn/484643.htm
ebz.mmmxz.cn/442663.htm
ebz.mmmxz.cn/002403.htm
ebz.mmmxz.cn/731553.htm
ebz.mmmxz.cn/268023.htm
ebz.mmmxz.cn/911373.htm
ebz.mmmxz.cn/115593.htm
ebz.mmmxz.cn/048403.htm
ebz.mmmxz.cn/828043.htm
ebl.mmmxz.cn/337193.htm
ebl.mmmxz.cn/513793.htm
ebl.mmmxz.cn/244803.htm
ebl.mmmxz.cn/862643.htm
ebl.mmmxz.cn/757133.htm
ebl.mmmxz.cn/757993.htm
ebl.mmmxz.cn/537733.htm
ebl.mmmxz.cn/755713.htm
ebl.mmmxz.cn/595973.htm
ebl.mmmxz.cn/866623.htm
ebk.mmmxz.cn/444203.htm
ebk.mmmxz.cn/404423.htm
ebk.mmmxz.cn/402483.htm
ebk.mmmxz.cn/248483.htm
ebk.mmmxz.cn/024603.htm
ebk.mmmxz.cn/886223.htm
ebk.mmmxz.cn/484643.htm
ebk.mmmxz.cn/602223.htm
ebk.mmmxz.cn/022043.htm
ebk.mmmxz.cn/068623.htm
ebj.mmmxz.cn/795533.htm
ebj.mmmxz.cn/246283.htm
ebj.mmmxz.cn/991373.htm
ebj.mmmxz.cn/646463.htm
ebj.mmmxz.cn/640003.htm
ebj.mmmxz.cn/468663.htm
ebj.mmmxz.cn/640843.htm
ebj.mmmxz.cn/482863.htm
ebj.mmmxz.cn/579333.htm
ebj.mmmxz.cn/719753.htm
ebh.mmmxz.cn/648003.htm
ebh.mmmxz.cn/779553.htm
ebh.mmmxz.cn/191973.htm
ebh.mmmxz.cn/264063.htm
ebh.mmmxz.cn/044003.htm
ebh.mmmxz.cn/804883.htm
ebh.mmmxz.cn/462203.htm
ebh.mmmxz.cn/048463.htm
ebh.mmmxz.cn/820603.htm
ebh.mmmxz.cn/933773.htm
ebg.mmmxz.cn/808003.htm
ebg.mmmxz.cn/220683.htm
ebg.mmmxz.cn/046863.htm
ebg.mmmxz.cn/597153.htm
ebg.mmmxz.cn/042083.htm
ebg.mmmxz.cn/771133.htm
ebg.mmmxz.cn/591193.htm
ebg.mmmxz.cn/977573.htm
ebg.mmmxz.cn/420223.htm
ebg.mmmxz.cn/686463.htm
ebf.mmmxz.cn/860683.htm
ebf.mmmxz.cn/173353.htm
ebf.mmmxz.cn/408663.htm
ebf.mmmxz.cn/137953.htm
ebf.mmmxz.cn/517333.htm
ebf.mmmxz.cn/195333.htm
ebf.mmmxz.cn/888263.htm
ebf.mmmxz.cn/711373.htm
ebf.mmmxz.cn/628003.htm
ebf.mmmxz.cn/486403.htm
ebd.mmmxz.cn/799773.htm
ebd.mmmxz.cn/400683.htm
ebd.mmmxz.cn/955713.htm
ebd.mmmxz.cn/919913.htm
ebd.mmmxz.cn/842643.htm
ebd.mmmxz.cn/157333.htm
ebd.mmmxz.cn/248483.htm
ebd.mmmxz.cn/064823.htm
ebd.mmmxz.cn/284243.htm
ebd.mmmxz.cn/280403.htm
ebs.mmmxz.cn/919133.htm
ebs.mmmxz.cn/395953.htm
ebs.mmmxz.cn/973973.htm
ebs.mmmxz.cn/064003.htm
ebs.mmmxz.cn/800063.htm
ebs.mmmxz.cn/531333.htm
ebs.mmmxz.cn/620843.htm
ebs.mmmxz.cn/282843.htm
ebs.mmmxz.cn/935333.htm
ebs.mmmxz.cn/082683.htm
eba.mmmxz.cn/557733.htm
eba.mmmxz.cn/082423.htm
eba.mmmxz.cn/531333.htm
eba.mmmxz.cn/404623.htm
eba.mmmxz.cn/066603.htm
eba.mmmxz.cn/848423.htm
eba.mmmxz.cn/604403.htm
eba.mmmxz.cn/404843.htm
eba.mmmxz.cn/513913.htm
eba.mmmxz.cn/595113.htm
ebp.mmmxz.cn/240263.htm
ebp.mmmxz.cn/593533.htm
ebp.mmmxz.cn/804463.htm
ebp.mmmxz.cn/480243.htm
ebp.mmmxz.cn/480003.htm
ebp.mmmxz.cn/319793.htm
ebp.mmmxz.cn/991193.htm
ebp.mmmxz.cn/086203.htm
ebp.mmmxz.cn/171713.htm
ebp.mmmxz.cn/551953.htm
ebo.mmmxz.cn/515173.htm
ebo.mmmxz.cn/719993.htm
ebo.mmmxz.cn/795533.htm
ebo.mmmxz.cn/828843.htm
ebo.mmmxz.cn/997193.htm
ebo.mmmxz.cn/068423.htm
ebo.mmmxz.cn/660263.htm
ebo.mmmxz.cn/917953.htm
ebo.mmmxz.cn/197733.htm
ebo.mmmxz.cn/086083.htm
ebi.mmmxz.cn/486223.htm
ebi.mmmxz.cn/886223.htm
ebi.mmmxz.cn/428063.htm
ebi.mmmxz.cn/062223.htm
ebi.mmmxz.cn/866263.htm
ebi.mmmxz.cn/591133.htm
ebi.mmmxz.cn/397353.htm
ebi.mmmxz.cn/220483.htm
ebi.mmmxz.cn/373113.htm
ebi.mmmxz.cn/753173.htm
ebu.mmmxz.cn/400243.htm
ebu.mmmxz.cn/557333.htm
ebu.mmmxz.cn/080043.htm
ebu.mmmxz.cn/684403.htm
ebu.mmmxz.cn/888263.htm
ebu.mmmxz.cn/135313.htm
ebu.mmmxz.cn/426643.htm
ebu.mmmxz.cn/062063.htm
ebu.mmmxz.cn/533973.htm
ebu.mmmxz.cn/084443.htm
eby.mmmxz.cn/628663.htm
eby.mmmxz.cn/408883.htm
eby.mmmxz.cn/260803.htm
eby.mmmxz.cn/799773.htm
eby.mmmxz.cn/240803.htm
eby.mmmxz.cn/955793.htm
eby.mmmxz.cn/822463.htm
eby.mmmxz.cn/082423.htm
eby.mmmxz.cn/442603.htm
eby.mmmxz.cn/757153.htm
ebt.mmmxz.cn/082643.htm
ebt.mmmxz.cn/886283.htm
ebt.mmmxz.cn/73.htm
ebt.mmmxz.cn/642263.htm
ebt.mmmxz.cn/644623.htm
ebt.mmmxz.cn/755913.htm
ebt.mmmxz.cn/404063.htm
ebt.mmmxz.cn/628883.htm
ebt.mmmxz.cn/860483.htm
ebt.mmmxz.cn/866883.htm
ebr.mmmxz.cn/808663.htm
ebr.mmmxz.cn/797533.htm
ebr.mmmxz.cn/044263.htm
ebr.mmmxz.cn/573333.htm
ebr.mmmxz.cn/042063.htm
ebr.mmmxz.cn/517793.htm
ebr.mmmxz.cn/957973.htm
ebr.mmmxz.cn/482243.htm
ebr.mmmxz.cn/260063.htm
ebr.mmmxz.cn/557113.htm
ebe.mmmxz.cn/606023.htm
ebe.mmmxz.cn/066843.htm
ebe.mmmxz.cn/062843.htm
ebe.mmmxz.cn/268863.htm
ebe.mmmxz.cn/460403.htm
ebe.mmmxz.cn/046403.htm
ebe.mmmxz.cn/004863.htm
ebe.mmmxz.cn/757953.htm
ebe.mmmxz.cn/244083.htm
ebe.mmmxz.cn/682463.htm
ebw.mmmxz.cn/731153.htm
ebw.mmmxz.cn/064483.htm
ebw.mmmxz.cn/282083.htm
ebw.mmmxz.cn/115313.htm
ebw.mmmxz.cn/357133.htm
ebw.mmmxz.cn/179353.htm
ebw.mmmxz.cn/391153.htm
ebw.mmmxz.cn/464223.htm
ebw.mmmxz.cn/559133.htm
ebw.mmmxz.cn/739153.htm
ebq.mmmxz.cn/840863.htm
ebq.mmmxz.cn/480643.htm
ebq.mmmxz.cn/286663.htm
ebq.mmmxz.cn/399153.htm
ebq.mmmxz.cn/133533.htm
ebq.mmmxz.cn/971153.htm
ebq.mmmxz.cn/628003.htm
ebq.mmmxz.cn/486243.htm
ebq.mmmxz.cn/888843.htm
ebq.mmmxz.cn/640263.htm
evm.mmmxz.cn/002483.htm
evm.mmmxz.cn/577193.htm
evm.mmmxz.cn/717913.htm
evm.mmmxz.cn/557733.htm
evm.mmmxz.cn/026423.htm
evm.mmmxz.cn/935153.htm
evm.mmmxz.cn/371153.htm
evm.mmmxz.cn/686203.htm
evm.mmmxz.cn/957593.htm
evm.mmmxz.cn/424023.htm
evn.mmmxz.cn/571133.htm
evn.mmmxz.cn/193373.htm
evn.mmmxz.cn/606463.htm
evn.mmmxz.cn/480283.htm
evn.mmmxz.cn/886063.htm
evn.mmmxz.cn/484603.htm
evn.mmmxz.cn/846803.htm
evn.mmmxz.cn/260043.htm
evn.mmmxz.cn/628483.htm
evn.mmmxz.cn/440863.htm
evb.mmmxz.cn/082083.htm
evb.mmmxz.cn/777373.htm
evb.mmmxz.cn/339593.htm
evb.mmmxz.cn/955133.htm
evb.mmmxz.cn/244283.htm
evb.mmmxz.cn/208803.htm
evb.mmmxz.cn/002823.htm
evb.mmmxz.cn/593993.htm
evb.mmmxz.cn/622643.htm
evb.mmmxz.cn/193353.htm
evv.mmmxz.cn/060823.htm
evv.mmmxz.cn/911373.htm
evv.mmmxz.cn/626423.htm
evv.mmmxz.cn/533553.htm
evv.mmmxz.cn/808803.htm
evv.mmmxz.cn/866803.htm
evv.mmmxz.cn/971593.htm
evv.mmmxz.cn/844483.htm
evv.mmmxz.cn/979313.htm
evv.mmmxz.cn/642883.htm
evc.mmmxz.cn/220643.htm
evc.mmmxz.cn/591513.htm
evc.mmmxz.cn/757333.htm
evc.mmmxz.cn/913333.htm
evc.mmmxz.cn/820023.htm
evc.mmmxz.cn/737513.htm
evc.mmmxz.cn/606823.htm
evc.mmmxz.cn/199193.htm
evc.mmmxz.cn/399153.htm
evc.mmmxz.cn/993713.htm
evx.mmmxz.cn/460003.htm
evx.mmmxz.cn/573393.htm
evx.mmmxz.cn/319733.htm
evx.mmmxz.cn/357593.htm
evx.mmmxz.cn/802023.htm
evx.mmmxz.cn/337953.htm
evx.mmmxz.cn/933733.htm
evx.mmmxz.cn/648443.htm
evx.mmmxz.cn/208683.htm
evx.mmmxz.cn/317953.htm
evz.mmmxz.cn/206663.htm
evz.mmmxz.cn/840803.htm
evz.mmmxz.cn/422043.htm
evz.mmmxz.cn/402643.htm
evz.mmmxz.cn/153913.htm
evz.mmmxz.cn/646883.htm
evz.mmmxz.cn/820863.htm
evz.mmmxz.cn/422083.htm
evz.mmmxz.cn/808643.htm
evz.mmmxz.cn/175913.htm
